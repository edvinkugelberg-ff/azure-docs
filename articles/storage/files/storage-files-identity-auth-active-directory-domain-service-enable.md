---
title: Use Azure AD Domain Services to authorize access to file data over SMB
description: Learn how to enable identity-based authentication over Server Message Block (SMB) for Azure Files through Azure Active Directory Domain Services. Your domain-joined Windows virtual machines (VMs) can then access Azure file shares by using Azure AD credentials.
author: khdownie

ms.service: storage
ms.topic: how-to
ms.date: 01/14/2022
ms.author: kendownie
ms.subservice: files
ms.custom: contperf-fy21q1, devx-track-azurecli, devx-track-azurepowershell
---

# Enable Azure Active Directory Domain Services authentication on Azure Files

[Azure Files](storage-files-introduction.md) supports identity-based authentication over Server Message Block (SMB) through two types of Domain Services: on-premises Active Directory Domain Services (AD DS) and Azure Active Directory Domain Services (Azure AD DS). We strongly recommend you to review the [How it works section](./storage-files-active-directory-overview.md#how-it-works) to select the right domain service for authentication. The setup is different depending on the domain service you choose. This article focuses on enabling and configuring Azure AD DS for authentication with Azure file shares.

If you are new to Azure file shares, we recommend reading our [planning guide](storage-files-planning.md) before reading the following series of articles.

> [!NOTE]
> Azure Files supports Kerberos authentication with Azure AD DS with RC4-HMAC and AES-256 encryption.
>
> Azure Files supports authentication for Azure AD DS with full synchronization with Azure AD. If you have enabled scoped synchronization in Azure AD DS which only sync a limited set of identities from Azure AD, authentication and authorization is not supported.

## Applies to
| File share type | SMB | NFS |
|-|:-:|:-:|
| Standard file shares (GPv2), LRS/ZRS | ![Yes](../media/icons/yes-icon.png) | ![No](../media/icons/no-icon.png) |
| Standard file shares (GPv2), GRS/GZRS | ![Yes](../media/icons/yes-icon.png) | ![No](../media/icons/no-icon.png) |
| Premium file shares (FileStorage), LRS/ZRS | ![Yes](../media/icons/yes-icon.png) | ![No](../media/icons/no-icon.png) |

## Prerequisites

Before you enable Azure AD over SMB for Azure file shares, make sure you have completed the following prerequisites:

1.  **Select or create an Azure AD tenant.**

    You can use a new or existing tenant for Azure AD authentication over SMB. The tenant and the file share that you want to access must be associated with the same subscription.

    To create a new Azure AD tenant, you can [Add an Azure AD tenant and an Azure AD subscription](/windows/client-management/mdm/add-an-azure-ad-tenant-and-azure-ad-subscription). If you have an existing Azure AD tenant but want to create a new tenant for use with Azure file shares, see [Create an Azure Active Directory tenant](/rest/api/datacatalog/create-an-azure-active-directory-tenant).

1.  **Enable Azure AD Domain Services on the Azure AD tenant.**

    To support authentication with Azure AD credentials, you must enable Azure AD Domain Services for your Azure AD tenant. If you aren't the administrator of the Azure AD tenant, contact the administrator and follow the step-by-step guidance to [Enable Azure Active Directory Domain Services using the Azure portal](../../active-directory-domain-services/tutorial-create-instance.md).

    It typically takes about 15 minutes for an Azure AD DS deployment to complete. Verify that the health status of Azure AD DS shows **Running**, with password hash synchronization enabled, before proceeding to the next step.

1.  **Domain-join an Azure VM with Azure AD DS.**

    To access a file share by using Azure AD credentials from a VM, your VM must be domain-joined to Azure AD DS. For more information about how to domain-join a VM, see [Join a Windows Server virtual machine to a managed domain](../../active-directory-domain-services/join-windows-vm.md).

    > [!NOTE]
    > Azure AD DS authentication over SMB with Azure file shares is supported only on Azure VMs running on OS versions above Windows 7 or Windows Server 2008 R2.

1.  **Select or create an Azure file share.**

    Select a new or existing file share that's associated with the same subscription as your Azure AD tenant. For information about creating a new file share, see [Create a file share in Azure Files](storage-how-to-create-file-share.md).
    For optimal performance, we recommend that your file share be in the same region as the VM from which you plan to access the share.

1.  **Verify Azure Files connectivity by mounting Azure file shares using your storage account key.**

    To verify that your VM and file share are properly configured, try mounting the file share using your storage account key. For more information, see [Mount an Azure file share and access the share in Windows](storage-how-to-use-files-windows.md).

## Regional availability

Azure Files authentication with Azure AD DS is available in [all Azure Public, Gov, and China regions](https://azure.microsoft.com/global-infrastructure/locations/).

## Overview of the workflow

Before you enable Azure AD DS Authentication over SMB for Azure file shares, verify that your Azure AD and Azure Storage environments are properly configured. We recommend that you walk through the [prerequisites](#prerequisites) to make sure you've completed all the required steps.

Next, do the following things to grant access to Azure Files resources with Azure AD credentials:

1. Enable Azure AD DS authentication over SMB for your storage account to register the storage account with the associated Azure AD DS deployment.
2. Assign access permissions for a share to an Azure AD identity (a user, group, or service principal).
3. Configure NTFS permissions over SMB for directories and files.
4. Mount an Azure file share from a domain-joined VM.

The following diagram illustrates the end-to-end workflow for enabling Azure AD DS authentication over SMB for Azure Files.

![Diagram showing Azure AD over SMB for Azure Files workflow](media/storage-files-active-directory-enable/azure-active-directory-over-smb-workflow.png)

## (Optional) Use AES 256 encryption

By default, Azure AD DS authentication uses Kerberos RC4 encryption. To use Kerberos AES256 instead, follow these steps:

As an Azure AD DS user with the required permissions (typically, members of the **AAD DC Administrators** group will have the necessary permissions), open the Azure cloud shell.

Execute the following commands:

```azurepowershell
# 1. Find the service account in your managed domain that represents the storage account.

$storageAccountName= “<InsertStorageAccountNameHere>”
$searchFilter = "Name -like '*{0}*'" -f $storageAccountName
$userObject = Get-ADUser -filter $searchFilter

if ($userObject -eq $null)
{
   Write-Error "Cannot find AD object for storage account:$storageAccountName" -ErrorAction Stop
}

# 2. Set the KerberosEncryptionType of the object

Set-ADUser $userObject -KerberosEncryptionType AES256

# 3. Validate that the object now has the expected (AES256) encryption type.

Get-ADUser $userObject -properties KerberosEncryptionType
```

## Enable Azure AD DS authentication for your account

To enable Azure AD DS authentication over SMB for Azure Files, you can set a property on storage accounts by using the Azure portal, Azure PowerShell, or Azure CLI. Setting this property implicitly "domain joins" the storage account with the associated Azure AD DS deployment. Azure AD DS authentication over SMB is then enabled for all new and existing file shares in the storage account.

Keep in mind that you can enable Azure AD DS authentication over SMB only after you have successfully deployed Azure AD DS to your Azure AD tenant. For more information, see the [prerequisites](#prerequisites).

# [Portal](#tab/azure-portal)

To enable Azure AD DS authentication over SMB with the [Azure portal](https://portal.azure.com), follow these steps:

1. In the Azure portal, go to your existing storage account, or [create a storage account](../common/storage-account-create.md).
1. In the **File shares** section, select **Active directory: Not Configured**.

    :::image type="content" source="media/storage-files-active-directory-enable/files-azure-ad-enable-storage-account-identity.png" alt-text="Screenshot of the File shares pane in your storage account, Active directory is highlighted." lightbox="media/storage-files-active-directory-enable/files-azure-ad-enable-storage-account-identity.png":::

1. Select **Azure Active Directory Domain Services** then switch the toggle to **Enabled**.
1. Select **Save**.

    :::image type="content" source="media/storage-files-active-directory-enable/files-azure-ad-highlight.png" alt-text="Screenshot of the Active Directory pane, Azure Active Directory Domain Services is enabled." lightbox="media/storage-files-active-directory-enable/files-azure-ad-highlight.png":::

# [PowerShell](#tab/azure-powershell)

To enable Azure AD DS authentication over SMB with Azure PowerShell, install the latest Az module (2.4 or newer) or the Az.Storage module (1.5 or newer). For more information about installing PowerShell, see [Install Azure PowerShell on Windows with PowerShellGet](/powershell/azure/install-Az-ps).

To create a new storage account, call [New-AzStorageAccount](/powershell/module/az.storage/New-azStorageAccount), and then set the **EnableAzureActiveDirectoryDomainServicesForFile** parameter to **true**. In the following example, remember to replace the placeholder values with your own values. (If you were using the previous preview module, the parameter for enabling the feature is **EnableAzureFilesAadIntegrationForSMB**.)

```powershell
# Create a new storage account
New-AzStorageAccount -ResourceGroupName "<resource-group-name>" `
    -Name "<storage-account-name>" `
    -Location "<azure-region>" `
    -SkuName Standard_LRS `
    -Kind StorageV2 `
    -EnableAzureActiveDirectoryDomainServicesForFile $true
```

To enable this feature on existing storage accounts, use the following command:

```powershell
# Update a storage account
Set-AzStorageAccount -ResourceGroupName "<resource-group-name>" `
    -Name "<storage-account-name>" `
    -EnableAzureActiveDirectoryDomainServicesForFile $true
```


# [Azure CLI](#tab/azure-cli)

To enable Azure AD authentication over SMB with Azure CLI, install the latest CLI version (Version 2.0.70 or newer). For more information about installing Azure CLI, see [Install the Azure CLI](/cli/azure/install-azure-cli).

To create a new storage account, call [az storage account create](/cli/azure/storage/account#az-storage-account-create), and set the `--enable-files-aadds` argument. In the following example, remember to replace the placeholder values with your own values. (If you were using the previous preview module, the parameter for feature enablement is **file-aad**.)

```azurecli-interactive
# Create a new storage account
az storage account create -n <storage-account-name> -g <resource-group-name> --enable-files-aadds
```

To enable this feature on existing storage accounts, use the following command:

```azurecli-interactive
# Update a new storage account
az storage account update -n <storage-account-name> -g <resource-group-name> --enable-files-aadds
```
---

[!INCLUDE [storage-files-aad-permissions-and-mounting](../../../includes/storage-files-aad-permissions-and-mounting.md)]

You have now successfully enabled Azure AD DS authentication over SMB and assigned a custom role that provides access to an Azure file share with an Azure AD identity. To grant additional users access to your file share, follow the instructions in the [Assign access permissions](#assign-access-permissions-to-an-identity) to use an identity and [Configure NTFS permissions over SMB sections](#configure-ntfs-permissions-over-smb).

## Next steps

For more information about Azure Files and how to use Azure AD over SMB, see these resources:

- [Overview of Azure Files identity-based authentication support for SMB access](storage-files-active-directory-overview.md)
- [FAQ](storage-files-faq.md)
