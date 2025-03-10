---
title: Configure encryption with customer-managed keys stored in Azure Key Vault
titleSuffix: Azure Storage
description: Learn how to configure Azure Storage encryption with customer-managed keys stored in Azure Key Vault by using the Azure portal, PowerShell, or Azure CLI.
services: storage
author: tamram

ms.service: storage
ms.topic: how-to
ms.date: 05/05/2022
ms.author: tamram
ms.reviewer: ozgun
ms.subservice: common 
ms.custom: devx-track-azurepowershell, devx-track-azurecli
---

# Configure encryption with customer-managed keys stored in Azure Key Vault

Azure Storage encrypts all data in a storage account at rest. By default, data is encrypted with Microsoft-managed keys. For additional control over encryption keys, you can manage your own keys. Customer-managed keys must be stored in Azure Key Vault or Key Vault Managed Hardware Security Model (HSM).

This article shows how to configure encryption with customer-managed keys stored in a key vault by using the Azure portal, PowerShell, or Azure CLI. To learn how to configure encryption with customer-managed keys stored in a managed HSM, see [Configure encryption with customer-managed keys stored in Azure Key Vault Managed HSM](customer-managed-keys-configure-key-vault-hsm.md).

> [!NOTE]
> Azure Key Vault and Azure Key Vault Managed HSM support the same APIs and management interfaces for configuration.

## Configure a key vault

You can use a new or existing key vault to store customer-managed keys. The storage account and key vault may be in different regions or subscriptions in the same tenant. To learn more about Azure Key Vault, see [Azure Key Vault Overview](../../key-vault/general/overview.md) and [What is Azure Key Vault?](../../key-vault/general/basic-concepts.md).

Using customer-managed keys with Azure Storage encryption requires that both soft delete and purge protection be enabled for the key vault. Soft delete is enabled by default when you create a new key vault and cannot be disabled. You can enable purge protection either when you create the key vault or after it is created.

# [Azure portal](#tab/portal)

To learn how to create a key vault with the Azure portal, see [Quickstart: Create a key vault using the Azure portal](../../key-vault/general/quick-create-portal.md). When you create the key vault, select **Enable purge protection**, as shown in the following image.

:::image type="content" source="media/customer-managed-keys-configure-key-vault/configure-key-vault-portal.png" alt-text="Screenshot showing how to enable purge protection when creating a key vault":::

To enable purge protection on an existing key vault, follow these steps:

1. Navigate to your key vault in the Azure portal.
1. Under **Settings**, choose **Properties**.
1. In the **Purge protection** section, choose **Enable purge protection**.

# [PowerShell](#tab/powershell)

To create a new key vault with PowerShell, install version 2.0.0 or later of the [Az.KeyVault](https://www.powershellgallery.com/packages/Az.KeyVault/2.0.0) PowerShell module. Then call [New-AzKeyVault](/powershell/module/az.keyvault/new-azkeyvault) to create a new key vault. With version 2.0.0 and later of the Az.KeyVault module, soft delete is enabled by default when you create a new key vault.

The following example creates a new key vault with both soft delete and purge protection enabled. Remember to replace the placeholder values in brackets with your own values.

```azurepowershell
$keyVault = New-AzKeyVault -Name <key-vault> `
    -ResourceGroupName <resource_group> `
    -Location <location> `
    -EnablePurgeProtection
```

To learn how to enable purge protection on an existing key vault with PowerShell, see [Azure Key Vault recovery overview](../../key-vault/general/key-vault-recovery.md?tabs=azure-powershell).

# [Azure CLI](#tab/azure-cli)

To create a new key vault using Azure CLI, call [az keyvault create](/cli/azure/keyvault#az-keyvault-create). Remember to replace the placeholder values in brackets with your own values:

```azurecli
az keyvault create \
    --name <key-vault> \
    --resource-group <resource_group> \
    --location <region> \
    --enable-purge-protection
```

To learn how to enable purge protection on an existing key vault with Azure CLI, see [Azure Key Vault recovery overview](../../key-vault/general/key-vault-recovery.md?tabs=azure-cli).

---

## Add a key

Next, add a key to the key vault.

Azure Storage encryption supports RSA and RSA-HSM keys of sizes 2048, 3072 and 4096. For more information about supported key types, see [About keys](../../key-vault/keys/about-keys.md).

# [Azure portal](#tab/portal)

To learn how to add a key with the Azure portal, see [Quickstart: Set and retrieve a key from Azure Key Vault using the Azure portal](../../key-vault/keys/quick-create-portal.md).

# [PowerShell](#tab/powershell)

To add a key with PowerShell, call [Add-AzKeyVaultKey](/powershell/module/az.keyvault/add-azkeyvaultkey). Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurepowershell
$key = Add-AzKeyVaultKey -VaultName $keyVault.VaultName `
    -Name <key> `
    -Destination 'Software'
```

# [Azure CLI](#tab/azure-cli)

To add a key with Azure CLI, call [az keyvault key create](/cli/azure/keyvault/key#az-keyvault-key-create). Remember to replace the placeholder values in brackets with your own values.

```azurecli
az keyvault key create \
    --name <key> \
    --vault-name <key-vault>
```

---

## Choose a managed identity to authorize access to the key vault

When you enable customer-managed keys for a storage account, you must specify a managed identity that will be used to authorize access to the key vault that contains the key. The managed identity must have permissions to access the key in the key vault.

The managed identity that authorizes access to the key vault may be either a user-assigned or system-assigned managed identity, depending on your scenario:

- When you configure customer-managed keys at the time that you create a storage account, you must specify a user-assigned managed identity.
- When you configure customer-managed keys on an existing storage account, you can specify either a user-assigned managed identity or a system-assigned managed identity.

To learn more about system-assigned versus user-assigned managed identities, see [Managed identity types](../../active-directory/managed-identities-azure-resources/overview.md#managed-identity-types).

### Use a user-assigned managed identity to authorize access

A user-assigned is a standalone Azure resource. To learn how to create and manage a user-assigned managed identity, see [Manage user-assigned managed identities](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md).

Both new and existing storage accounts can use a user-assigned identity to authorize access to the key vault. You must create the user-assigned identity before you configure customer-managed keys.

#### [Azure portal](#tab/portal)

When you configure customer-managed keys with the Azure portal, you can select an existing user-assigned identity through the portal user interface. For details, see one of the following sections:

- [Configure customer-managed keys for a new account](#configure-customer-managed-keys-for-a-new-account)
- [Configure customer-managed keys for an existing account](#configure-customer-managed-keys-for-an-existing-account)

#### [PowerShell](#tab/powershell)

To authorize access to the key vault with a user-assigned managed identity, you will need the resource ID and principal ID of the user-assigned managed identity. Call [Get-AzUserAssignedIdentity](/powershell/module/az.managedserviceidentity/get-azuserassignedidentity) to get the user-assigned managed identity, then save the resource ID and principal ID to variables. You will need these values in subsequent steps:

```azurepowershell
$userIdentityId = Get-AzUserAssignedIdentity -Name <user-assigned-identity> -ResourceGroupName <resource-group>
$principalId = $userIdentity.PrincipalId
```

#### [Azure CLI](#tab/azure-cli)

To authorize access to the key vault with a user-assigned managed identity, you will need the resource ID and principal ID of the user-assigned managed identity. Call [az identity show](/cli/azure/identity#az-identity-show) command to get the user-assigned managed identity, then save the resource ID and principal ID to variables. You will need these values in subsequent steps:

```azurecli
userIdentityId=$(az identity show --name sample-user-assigned-identity --resource-group storagesamples-rg --query id)
principalId=$(az identity show --name sample-user-assigned-identity --resource-group storagesamples-rg --query principalId)
```

---

### Use a system-assigned managed identity to authorize access

A system-assigned managed identity is associated with an instance of an Azure service, in this case an Azure Storage account. You must explicitly assign a system-assigned managed identity to a storage account before you can use the system-assigned managed identity to authorize access to the key vault that contains your customer-managed key.

Only existing storage accounts can use a system-assigned identity to authorize access to the key vault. New storage accounts must use a user-assigned identity, if customer-managed keys are configured on account creation.

#### [Azure portal](#tab/portal)

When you configure customer-managed keys with the Azure portal with a system-assigned managed identity, the system-assigned managed identity is assigned to the storage account for you under the covers. For details, see [Configure customer-managed keys for an existing account](#configure-customer-managed-keys-for-an-existing-account).

#### [PowerShell](#tab/powershell)

To assign a system-assigned managed identity to your storage account, call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount):

```azurepowershell
$storageAccount = Set-AzStorageAccount -ResourceGroupName <resource_group> `
    -Name <storage-account> `
    -AssignIdentity
```

Next, get the principal ID for the system-assigned managed identity, and save it to a variable. You will need this value in the next step to create the key vault access policy:

```azurepowershell
$principalId = $storageAccount.Identity.PrincipalId
```

#### [Azure CLI](#tab/azure-cli)

To authenticate access to the key vault with a system-assigned managed identity, assign the system-assigned managed identity to the storage account by calling [az storage account update](/cli/azure/storage/account#az-storage-account-update):

```azurecli
az storage account update \
    --name <storage-account> \
    --resource-group <resource_group> \
    --assign-identity
```

Next, get the principal ID for the system-assigned managed identity, and save it to a variable. You will need this value in the next step to create the key vault access policy:

```azurecli
principalId = $(az storage account show --name <storage-account> --resource-group <resource_group> --query identity.principalId)
```

---

## Configure the key vault access policy

The next step is to configure the key vault access policy. The key vault access policy grants permissions to the managed identity that will be used to authorize access to the key vault. To learn more about key vault access policies, see [Azure Key Vault Overview](../../key-vault/general/overview.md#securely-store-secrets-and-keys) and [Azure Key Vault security overview](../../key-vault/general/security-features.md#key-vault-authentication-options).

### [Azure portal](#tab/portal)

To learn how to configure the key vault access policy with the Azure portal, see [Assign an Azure Key Vault access policy](../../key-vault/general/assign-access-policy.md).

### [PowerShell](#tab/powershell)

To configure the key vault access policy with PowerShell, call [Set-AzKeyVaultAccessPolicy](/powershell/module/az.keyvault/set-azkeyvaultaccesspolicy), providing the variable for the principal ID that you previously retrieved for the managed identity.

```azurepowershell
Set-AzKeyVaultAccessPolicy `
    -VaultName $keyVault.VaultName `
    -ObjectId $principalId `
    -PermissionsToKeys wrapkey,unwrapkey,get
```

To learn more about assigning the key vault access policy with PowerShell, see [Assign an Azure Key Vault access policy](../../key-vault/general/assign-access-policy.md).

### [Azure CLI](#tab/azure-cli)

To configure the key vault access policy with PowerShell, call [az keyvault set-policy](/cli/azure/keyvault#az-keyvault-set-policy), providing the variable for the principal ID that you previously retrieved for the managed identity.

```azurecli
az keyvault set-policy \
    --name <key-vault> \
    --resource-group <resource_group>
    --object-id $principalId \
    --key-permissions get unwrapKey wrapKey
```

To learn more about assigning the key vault access policy with Azure CLI, see [Assign an Azure Key Vault access policy](../../key-vault/general/assign-access-policy.md).

---

## Configure customer-managed keys for a new account

When you configure encryption with customer-managed keys for a new storage account, you can choose to automatically update the key version used for Azure Storage encryption whenever a new version is available in the associated key vault. Alternately, you can explicitly specify a key version to be used for encryption until the key version is manually updated.

You must use an existing user-assigned managed identity to authorize access to the key vault when you configure customer-managed keys while creating the storage account. The user-assigned managed identity must have appropriate permissions to access the key vault.

### [Azure portal](#tab/portal)

To configure customer-managed keys for a new storage account with automatic updating of the key version, follow these steps:

1. In the Azure portal, navigate to the **Storage accounts** page, and select the **Create** button to create a new account.
1. Follow the steps outlined in [Create a storage account](storage-account-create.md) to fill out the fields on the **Basics**, **Advanced**, **Networking**, and **Data Protection** tabs.
1. On the **Encryption** tab, indicate for which services you want to enable support for customer-managed keys in the **Enable support for customer-managed keys** field.
1. In the **Encryption type** field, select **Customer-managed keys (CMK)**.
1. In the **Encryption key** field, choose **Select a key vault and key**, and specify the key vault and key.
1. For the **User-assigned identity** field, select an existing user-assigned managed identity.

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-new-account-configure-cmk.png" alt-text="Screenshot showing how to configure customer-managed keys for a new storage account in Azure portal":::

1. Select **Review + create** to validate and create the new account.

You can also configure customer-managed keys with manual updating of the key version when you create a new storage account. Follow the steps described in [Configure encryption for manual updating of key versions](#configure-encryption-for-manual-updating-of-key-versions).

### [PowerShell](#tab/powershell)

To configure customer-managed keys for a new storage account with automatic updating of the key version, call [New-AzStorageAccount](/powershell/module/az.storage/new-azstorageaccount), as shown in the following example. Use the variable you created previously for the resource ID for the user-assigned managed identity. You will also need the key vault URI and key name:

```azurepowershell
New-AzStorageAccount -ResourceGroupName <resource-group> `
    -Name <storage-account> `
    -Kind StorageV2 `
    -SkuName Standard_LRS `
    -Location $location `
    -IdentityType SystemAssignedUserAssigned `
    -UserIdentityId $userIdentityId `
    -KeyVaultUri $keyVault.VaultUri `
    -KeyName $key.Name `
    -KeyVaultUserAssignedIdentityId $userIdentityId
```

### [Azure CLI](#tab/azure-cli)

To configure customer-managed keys for a new storage account with automatic updating of the key version, call [az storage account create](/cli/azure/storage/account#az-storage-account-create), as shown in the following example. Use the variable you created previously for the resource ID for the user-assigned managed identity. You will also need the key vault URI and key name:

```azurecli
az storage account create \
    --name <storage-account> \
    --resource-group <resource-group> \
    --location <location> \
    --sku Standard_LRS \
    --kind StorageV2 \
    --identity-type SystemAssigned,UserAssigned \
    --user-identity-id <user-assigned-managed-identity> \
    --encryption-key-vault <key-vault-uri> \
    --encryption-key-name <key-name> \
    --encryption-key-source Microsoft.Keyvault \
    --key-vault-user-identity-id <user-assigned-managed-identity>
```

---

## Configure customer-managed keys for an existing account

When you configure encryption with customer-managed keys for an existing storage account, you can choose to automatically update the key version used for Azure Storage encryption whenever a new version is available in the associated key vault. Alternately, you can explicitly specify a key version to be used for encryption until the key version is manually updated.

You can use either a system-assigned or user-assigned managed identity to authorize access to the key vault when you configure customer-managed keys for an existing storage account.

> [!NOTE]
> To rotate a key, create a new version of the key in Azure Key Vault. Azure Storage does not handle key rotation, so you will need to manage rotation of the key in the key vault. You can [configure key auto-rotation in Azure Key Vault](../../key-vault/keys/how-to-configure-key-rotation.md) or rotate your key manually.

### Configure encryption for automatic updating of key versions

Azure Storage can automatically update the customer-managed key that is used for encryption to use the latest key version from the key vault. Azure Storage checks the key vault daily for a new version of the key. When a new version becomes available, then Azure Storage automatically begins using the latest version of the key for encryption.

> [!IMPORTANT]
> Azure Storage checks the key vault for a new key version only once daily. When you rotate a key, be sure to wait 24 hours before disabling the older version.

### [Azure portal](#tab/portal)

To configure customer-managed keys for an existing account with automatic updating of the key version in the Azure portal, follow these steps:

1. Navigate to your storage account.
1. On the **Settings** blade for the storage account, click **Encryption**. By default, key management is set to **Microsoft Managed Keys**, as shown in the following image.

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-configure-encryption-keys.png" alt-text="Screenshot showing encryption options in Azure portal" lightbox="media/customer-managed-keys-configure-key-vault/portal-configure-encryption-keys.png":::

1. Select the **Customer Managed Keys** option.
1. Choose the **Select from Key Vault** option.
1. Select **Select a key vault and key**.
1. Select the key vault containing the key you want to use. You can also create a new key vault.
1. Select the key from the key vault. You can also create a new key.

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-select-key-from-key-vault.png" alt-text="Screenshot showing how to select key vault and key in Azure portal.":::

1. Select the type of identity to use to authenticate access to the key vault. The options include **System-assigned** (the default) or **User-assigned**. To learn more about each type of managed identity, see [Managed identity types](../../active-directory/managed-identities-azure-resources/overview.md#managed-identity-types).

    1. If you select **System-assigned**, the system-assigned managed identity for the storage account is created under the covers, if it does not already exist.
    1. If you select **User-assigned**, then you must select an existing user-assigned identity that has permissions to access the key vault. To learn how to create a user-assigned identity, see [Manage user-assigned managed identities](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md).

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/select-user-assigned-managed-identity-portal.png" alt-text="Screenshot showing how to select a user-assigned managed identity for key vault authentication":::

1. Save your changes.

After you've specified the key, the Azure portal indicates that automatic updating of the key version is enabled and displays the key version currently in use for encryption. The portal also displays the type of managed identity used to authorize access to the key vault and the principal ID for the managed identity.

:::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-auto-rotation-enabled.png" alt-text="Screenshot showing automatic updating of the key version enabled":::

### [PowerShell](#tab/powershell)

To configure customer-managed keys for an existing account with automatic updating of the key version with PowerShell, install the [Az.Storage](https://www.powershellgallery.com/packages/Az.Storage) module, version 2.0.0 or later.

Next, call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount) to update the storage account's encryption settings, omitting the key version. Include the **-KeyvaultEncryption** option to enable customer-managed keys for the storage account.

```azurepowershell
Set-AzStorageAccount -ResourceGroupName $storageAccount.ResourceGroupName `
    -AccountName $storageAccount.StorageAccountName `
    -KeyvaultEncryption `
    -KeyName $key.Name `
    -KeyVaultUri $keyVault.VaultUri
```

### [Azure CLI](#tab/azure-cli)

To configure customer-managed keys for an existing account with automatic updating of the key version with Azure CLI, install [Azure CLI version 2.4.0](/cli/azure/release-notes-azure-cli#april-21-2020) or later. For more information, see [Install the Azure CLI](/cli/azure/install-azure-cli).

Next, call [az storage account update](/cli/azure/storage/account#az-storage-account-update) to update the storage account's encryption settings, omitting the key version. Include the `--encryption-key-source` parameter and set it to `Microsoft.Keyvault` to enable customer-managed keys for the account.

```azurecli
key_vault_uri=$(az keyvault show \
    --name <key-vault> \
    --resource-group <resource_group> \
    --query properties.vaultUri \
    --output tsv)
az storage account update
    --name <storage-account> \
    --resource-group <resource_group> \
    --encryption-key-name <key> \
    --encryption-key-source Microsoft.Keyvault \
    --encryption-key-vault $key_vault_uri
```

---

### Configure encryption for manual updating of key versions

If you prefer to manually update the key version, then explicitly specify the version at the time that you configure encryption with customer-managed keys. In this case, Azure Storage will not automatically update the key version when a new version is created in the key vault. To use a new key version, you must manually update the version used for Azure Storage encryption.

# [Azure portal](#tab/portal)

To configure customer-managed keys with manual updating of the key version in the Azure portal, specify the key URI, including the version. To specify a key as a URI, follow these steps:

1. To locate the key URI in the Azure portal, navigate to your key vault, and select the **Keys** setting. Select the desired key, then click the key to view its versions. Select a key version to view the settings for that version.
1. Copy the value of the **Key Identifier** field, which provides the URI.

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-copy-key-identifier.png" alt-text="Screenshot showing key vault key URI in Azure portal":::

1. In the **Encryption key** settings for your storage account, choose the **Enter key URI** option.
1. Paste the URI that you copied into the **Key URI** field. Omit the key version from the URI to enable automatic updating of the key version.

    :::image type="content" source="media/customer-managed-keys-configure-key-vault/portal-specify-key-uri.png" alt-text="Screenshot showing how to enter key URI in Azure portal":::

1. Specify the subscription that contains the key vault.
1. Specify either a system-assigned or user-assigned managed identity.
1. Save your changes.

# [PowerShell](#tab/powershell)

To configure customer-managed keys with manual updating of the key version, explicitly provide the key version when you configure encryption for the storage account. Call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount) to update the storage account's encryption settings, as shown in the following example, and include the **-KeyvaultEncryption** option to enable customer-managed keys for the storage account.

Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurepowershell
Set-AzStorageAccount -ResourceGroupName $storageAccount.ResourceGroupName `
    -AccountName $storageAccount.StorageAccountName `
    -KeyvaultEncryption `
    -KeyName $key.Name `
    -KeyVersion $key.Version `
    -KeyVaultUri $keyVault.VaultUri
```

When you manually update the key version, you will need to update the storage account's encryption settings to use the new version. First, call [Get-AzKeyVaultKey](/powershell/module/az.keyvault/get-azkeyvaultkey) to get the latest version of the key. Then call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount) to update the storage account's encryption settings to use the new version of the key, as shown in the previous example.

# [Azure CLI](#tab/azure-cli)

To configure customer-managed keys with manual updating of the key version, explicitly provide the key version when you configure encryption for the storage account. Call [az storage account update](/cli/azure/storage/account#az-storage-account-update) to update the storage account's encryption settings, as shown in the following example. Include the `--encryption-key-source` parameter and set it to `Microsoft.Keyvault` to enable customer-managed keys for the account.

Remember to replace the placeholder values in brackets with your own values.

```azurecli
key_vault_uri=$(az keyvault show \
    --name <key-vault> \
    --resource-group <resource_group> \
    --query properties.vaultUri \
    --output tsv)
key_version=$(az keyvault key list-versions \
    --name <key> \
    --vault-name <key-vault> \
    --query [-1].kid \
    --output tsv | cut -d '/' -f 6)
az storage account update
    --name <storage-account> \
    --resource-group <resource_group> \
    --encryption-key-name <key> \
    --encryption-key-version $key_version \
    --encryption-key-source Microsoft.Keyvault \
    --encryption-key-vault $key_vault_uri
```

When you manually update the key version, you will need to update the storage account's encryption settings to use the new version. First, query for the key vault URI by calling [az keyvault show](/cli/azure/keyvault#az-keyvault-show), and for the key version by calling [az keyvault key list-versions](/cli/azure/keyvault/key#az-keyvault-key-list-versions). Then call [az storage account update](/cli/azure/storage/account#az-storage-account-update) to update the storage account's encryption settings to use the new version of the key, as shown in the previous example.

---

## Change the key

You can change the key that you are using for Azure Storage encryption at any time.

# [Azure portal](#tab/portal)

To change the key with the Azure portal, follow these steps:

1. Navigate to your storage account and display the **Encryption** settings.
1. Select the key vault and choose a new key.
1. Save your changes.

# [PowerShell](#tab/powershell)

To change the key with PowerShell, call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount) as shown in [Configure customer-managed keys for an existing account](#configure-customer-managed-keys-for-an-existing-account) and provide the new key name and version. If the new key is in a different key vault, then you must also update the key vault URI.

# [Azure CLI](#tab/azure-cli)

To change the key with Azure CLI, call [az storage account update](/cli/azure/storage/account#az-storage-account-update) as shown in [Configure customer-managed keys for an existing account](#configure-customer-managed-keys-for-an-existing-account) and provide the new key name and version. If the new key is in a different key vault, then you must also update the key vault URI.

---

## Revoke customer-managed keys

Revoking a customer-managed key removes the association between the storage account and the key vault.

# [Azure portal](#tab/portal)

To revoke customer-managed keys with the Azure portal, disable the key as described in [Disable customer-managed keys](#disable-customer-managed-keys).

# [PowerShell](#tab/powershell)

You can revoke customer-managed keys by removing the key vault access policy. To revoke a customer-managed key with PowerShell, call the [Remove-AzKeyVaultAccessPolicy](/powershell/module/az.keyvault/remove-azkeyvaultaccesspolicy) command, as shown in the following example. Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurepowershell
Remove-AzKeyVaultAccessPolicy -VaultName $keyVault.VaultName `
    -ObjectId $storageAccount.Identity.PrincipalId `
```

# [Azure CLI](#tab/azure-cli)

You can revoke customer-managed keys by removing the key vault access policy. To revoke a customer-managed key with Azure CLI, call the [az keyvault delete-policy](/cli/azure/keyvault#az-keyvault-delete-policy) command, as shown in the following example. Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurecli
az keyvault delete-policy \
    --name <key-vault> \
    --object-id $storage_account_principal
```

---

## Disable customer-managed keys

When you disable customer-managed keys, your storage account is once again encrypted with Microsoft-managed keys.

# [Azure portal](#tab/portal)

To disable customer-managed keys in the Azure portal, follow these steps:

1. Navigate to your storage account and display the **Encryption** settings.
1. Deselect the checkbox next to the **Use your own key** setting.

# [PowerShell](#tab/powershell)

To disable customer-managed keys with PowerShell, call [Set-AzStorageAccount](/powershell/module/az.storage/set-azstorageaccount) with the `-StorageEncryption` option, as shown in the following example. Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurepowershell
Set-AzStorageAccount -ResourceGroupName $storageAccount.ResourceGroupName `
    -AccountName $storageAccount.StorageAccountName `
    -StorageEncryption  
```

# [Azure CLI](#tab/azure-cli)

To disable customer-managed keys with Azure CLI, call [az storage account update](/cli/azure/storage/account#az-storage-account-update) and set the `--encryption-key-source parameter` to `Microsoft.Storage`, as shown in the following example. Remember to replace the placeholder values in brackets with your own values and to use the variables defined in the previous examples.

```azurecli
az storage account update
    --name <storage-account> \
    --resource-group <resource_group> \
    --encryption-key-source Microsoft.Storage
```

---

## Next steps

- [Azure Storage encryption for data at rest](storage-service-encryption.md)
- [Customer-managed keys for Azure Storage encryption](customer-managed-keys-overview.md)
- [Configure encryption with customer-managed keys stored in Azure Key Vault Managed HSM](customer-managed-keys-configure-key-vault-hsm.md)
