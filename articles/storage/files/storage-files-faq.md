---
title: Frequently asked questions (FAQ) for Azure Files | Microsoft Docs
description: Get answers to Azure Files frequently asked questions. You can mount Azure file shares concurrently on cloud or on-premises Windows, Linux, or macOS deployments.
author: khdownie
ms.service: storage
ms.date: 06/06/2022
ms.author: kendownie
ms.subservice: files
ms.topic: conceptual
---

# Frequently asked questions (FAQ) about Azure Files
[Azure Files](storage-files-introduction.md) offers fully managed file shares in the cloud that are accessible via the industry-standard [Server Message Block (SMB) protocol](/windows/win32/fileio/microsoft-smb-protocol-and-cifs-protocol-overview) and the [Network File System (NFS) protocol](https://en.wikipedia.org/wiki/Network_File_System). You can mount Azure file shares concurrently on cloud or on-premises deployments of Windows, Linux, and macOS. You also can cache Azure file shares on Windows Server machines by using Azure File Sync for fast access close to where the data is used.

## Azure File Sync

* <a id="cross-domain-sync"></a>
  **Can I have domain-joined and non-domain-joined servers in the same sync group?**  
    Yes. A sync group can contain server endpoints that have different Active Directory memberships, even if they are not domain-joined. Although this configuration technically works, we do not recommend this as a typical configuration because access control lists (ACLs) that are defined for files and folders on one server might not be able to be enforced by other servers in the sync group. For best results, we recommend syncing between servers that are in the same Active Directory forest, between servers that are in different Active Directory forests but have established trust relationships, or between servers that aren't in a domain. We recommend that you avoid using a mix of these configurations.

* <a id="afs-change-detection"></a>
  **I created a file directly in my Azure file share by using SMB or in the portal. How long does it take for the file to sync to the servers in the sync group?**  
    [!INCLUDE [storage-sync-files-change-detection](../../../includes/storage-sync-files-change-detection.md)]


* <a id="afs-conflict-resolution"></a>
  **If the same file is changed on two servers at approximately the same time, what happens?**  
    Azure File Sync uses a simple conflict-resolution strategy: we keep both changes to files that are changed in two endpoints at the same time. The most recently written change keeps the original file name. The older file (determined by LastWriteTime) has the endpoint name and the conflict number appended to the filename. For server endpoints, the endpoint name is the name of the server. For cloud endpoints, the endpoint name is **Cloud**. The name follows this taxonomy:
   
    \<FileNameWithoutExtension\>-\<endpointName\>\[-#\].\<ext\>  

    For example, the first conflict of CompanyReport.docx would become CompanyReport-CentralServer.docx if CentralServer is where the older write occurred. The second conflict would be named CompanyReport-CentralServer-1.docx. Azure File Sync supports 100 conflict files per file. Once the maximum number of conflict files has been reached, the file will fail to sync until the number of conflict files is less than 100.
  
* <a id="afs-tiered-files-tiering-disabled"></a>
  **I have cloud tiering disabled, why are there tiered files in the server endpoint location?**  
    There are two reasons why tiered files may exist in the server endpoint location:

    - When adding a new server endpoint to an existing sync group, if you choose either the recall namespace first option or recall namespace only option for initial download mode, files will show up as tiered until they're downloaded locally. To avoid this, select the avoid tiered files option for initial download mode. To manually recall files, use the [Invoke-StorageSyncFileRecall](../file-sync/file-sync-how-to-manage-tiered-files.md#how-to-recall-a-tiered-file-to-disk) cmdlet.

    - If cloud tiering was enabled on the server endpoint and then disabled, files will remain tiered until they're accessed.

* <a id="afs-tiered-files-not-showing-thumbnails"></a>
  **Why are my tiered files not showing thumbnails or previews in Windows Explorer?**  
    For tiered files, thumbnails and previews won't be visible at your server endpoint. This behavior is expected since the thumbnail cache feature in Windows intentionally skips reading files with the offline attribute. With Cloud Tiering enabled, reading through tiered files would cause them to be downloaded (recalled).

    This behavior isn't specific to Azure File Sync. Windows Explorer displays a "grey X" for any files that have the offline attribute set. You'll see the X icon when accessing files over SMB. For a detailed explanation of this behavior, refer to [Why don’t I get thumbnails for files that are marked offline?](https://devblogs.microsoft.com/oldnewthing/20170503-00/?p=96105)

    For questions on how to manage tiered files, see [How to manage tiered files](../file-sync/file-sync-how-to-manage-tiered-files.md).

* <a id="afs-tiered-files-out-of-endpoint"></a>
  **Why do tiered files exist outside of the server endpoint namespace?**  
    Prior to Azure File Sync agent version 3, Azure File Sync blocked the move of tiered files outside the server endpoint but on the same volume as the server endpoint. Copy operations, moves of non-tiered files, and moves of tiered to other volumes were unaffected. The reason for this behavior was the implicit assumption that File Explorer and other Windows APIs have that move operations on the same volume are (nearly) instantaneous rename operations. This means moves will make File Explorer or other move methods (such as command line or PowerShell) appear unresponsive while Azure File Sync recalls the data from the cloud. Starting with [Azure File Sync agent version 3.0.12.0](../file-sync/file-sync-release-notes.md#supported-versions), Azure File Sync will allow you to move a tiered file outside of the server endpoint. We avoid the negative effects previously mentioned by allowing the tiered file to exist as a tiered file outside of the server endpoint and then recalling the file in the background. This means that moves on the same volume are instantaneous, and we do all the work to recall the file to disk after the move has completed.

* <a id="afs-do-not-delete-server-endpoint"></a>
  **I'm having an issue with Azure File Sync on my server (sync, cloud tiering, etc.). Should I remove and recreate my server endpoint?**  
    [!INCLUDE [storage-sync-files-remove-server-endpoint](../../../includes/storage-sync-files-remove-server-endpoint.md)]
    
* <a id="afs-resource-move"></a>
  **Can I move the storage sync service and/or storage account to a different resource group, subscription, or Azure AD tenant?**  
   Yes, the storage sync service and/or storage account can be moved to a different resource group, subscription, or Azure AD tenant. After the  storage sync service or storage account is moved, you need to give the Microsoft.StorageSync application access to the storage account (see [Ensure Azure File Sync has access to the storage account](../file-sync/file-sync-troubleshoot.md?tabs=portal1%252cportal#troubleshoot-rbac)).

    > [!Note]  
    > When creating the cloud endpoint, the storage sync service and storage account must be in the same Azure AD tenant. Once the cloud endpoint is created, the storage sync service and storage account can be moved to different Azure AD tenants.
    
* <a id="afs-ntfs-acls"></a>
  **Does Azure File Sync preserve directory/file level NTFS ACLs along with data stored in Azure Files?**

    As of February 24, 2020, new and existing ACLs tiered by Azure file sync will be persisted in NTFS format, and ACL modifications made directly to the Azure file share will sync to all servers in the sync group. Any changes on ACLs made to Azure Files will sync down via Azure file sync. When copying data to Azure Files, make sure you use a copy tool that supports the necessary "fidelity" to copy attributes, timestamps and ACLs into an Azure file share - either via SMB or REST. When using Azure copy tools, such as AzCopy, it's important to use the latest version. Check the [file copy tools table](storage-files-migration-overview.md#file-copy-tools) to get an overview of Azure copy tools to ensure you can copy all of the important metadata of a file.

    If you have enabled Azure Backup on your file sync managed file shares, file ACLs can continue to be restored as part of the backup restore workflow. This works either for the entire share or individual files/directories.

    If you're using snapshots as part of the self-managed backup solution for file shares managed by file sync, your ACLs may not be restored properly to NTFS ACLs if the snapshots were taken before February 24, 2020. If this occurs, consider contacting Azure Support.

* <a id="afs-lastwritetime"></a>
  **Does Azure File Sync sync the LastWriteTime for directories?**  
    No, Azure File Sync doesn't sync the LastWriteTime for directories.
    
## Security, authentication, and access control

* <a id="file-auditing"></a>
**How can I audit file access and changes in Azure Files?**

  There are two options that provide auditing functionality for Azure Files:
  - If users are accessing the Azure file share directly, [Azure Storage logs](../blobs/monitor-blob-storage.md?tabs=azure-powershell#analyzing-logs) can be used to track file changes and user access. These logs can be used for troubleshooting purposes and the requests are logged on a best-effort basis.
  - If users are accessing the Azure file share via a Windows Server that has the Azure File Sync agent installed, use an [audit policy](/windows/security/threat-protection/auditing/apply-a-basic-audit-policy-on-a-file-or-folder) or third-party product to track file changes and user access on the Windows Server. 

* <a id="access-based-enumeration"></a>
**Does Azure Files support using Access-Based Enumeration (ABE) to control the visibility of the files and folders in SMB Azure file shares?**

  No, this scenario isn't supported.

   
### AD DS & Azure AD DS Authentication
* <a id="ad-support-devices"></a>
**Does Azure Active Directory Domain Services (Azure AD DS) support SMB access using Azure AD credentials from devices joined to or registered with Azure AD?**

    No, this scenario isn't supported.

* <a id="ad-vm-subscription"></a>
**Can I access Azure file shares with Azure AD credentials from a VM under a different subscription?**

    If the subscription under which the file share is deployed is associated with the same Azure AD tenant as the Azure AD DS deployment to which the VM is domain-joined, you can then access Azure file shares using the same Azure AD credentials. The limitation is imposed not on the subscription but on the associated Azure AD tenant.
    
* <a id="ad-support-subscription"></a>
**Can I enable either Azure AD DS or on-premises AD DS authentication for Azure file shares using an Azure AD tenant that is different from the Azure file share's primary tenant?**

    No, Azure Files only supports Azure AD DS or on-premises AD DS integration with an Azure AD tenant that resides in the same subscription as the file share. Only one subscription can be associated with an Azure AD tenant. This limitation applies to both Azure AD DS and on-premises AD DS authentication methods. When using on-premises AD DS for authentication, [the AD DS credential must be synced to the Azure AD](../../active-directory/hybrid/how-to-connect-install-roadmap.md) that the storage account is associated with.

* <a id="ad-multiple-forest"></a>
**Does on-premises AD DS authentication for Azure file shares support integration with an AD DS environment using multiple forests?**

    Azure Files on-premises AD DS authentication only integrates with the forest of the domain service that the storage account is registered to. To support authentication from another forest, your environment must have a forest trust configured correctly. The way Azure Files register in AD DS almost the same as a regular file server, where it creates an identity (computer or service logon account) in AD DS for authentication. The only difference is that the registered SPN of the storage account ends with "file.core.windows.net" which does not match with the domain suffix. Consult your domain administrator to see if any update to your suffix routing policy is required to enable multiple forest authentication due to the different domain suffix. We provide an example below to configure suffix routing policy.
    
    Example: When users in forest A domain want to reach a file share with the storage account registered against a domain in forest B, this won't automatically work because the service principal of the storage account doesn't have a suffix matching the suffix of any domain in forest A. We can address this issue by manually configuring a suffix routing rule from forest A to forest B for a custom suffix of "file.core.windows.net".

    First, you must add a new custom suffix on forest B. Make sure you have the appropriate administrative permissions to change the configuration, then follow these steps:

    1. Logon to a machine domain joined to forest B.
    2. Open up the **Active Directory Domains and Trusts** console.
    3. Right-click on **Active Directory Domains and Trusts**.
    4. Select **Properties**.
    5. Select **Add**.
    6. Add "file.core.windows.net" as the UPN Suffixes.
    7. Select **Apply**, then **OK** to close the wizard.
    
    Next, add the suffix routing rule on forest A, so that it redirects to forest B.

    1. Logon to a machine domain joined to forest A.
    2. Open up "Active Directory Domains and Trusts" console.
    3. Right-click on the domain that you want to access the file share, then select the **Trusts** tab and select forest B domain from outgoing trusts. If you haven't configured trust between the two forests, you need to set up the trust first.
    4. Select **Properties** and then **Name Suffix Routing**
    5. Check if the "*.file.core.windows.net" suffix shows up. If not, click **Refresh**.
    6. Select "*.file.core.windows.net", then select **Enable** and **Apply**.

* <a id="ad-aad-smb-files"></a>
**Is there any difference in creating a computer account or service logon account to represent my storage account in AD?**

    Creating either a [computer account](/windows/security/identity-protection/access-control/active-directory-accounts#manage-default-local-accounts-in-active-directory) (default) or a [service logon account](/windows/win32/ad/about-service-logon-accounts) has no difference on how the authentication would work with Azure Files. You can make your own choice on how to represent a storage account as an identity in your AD environment. The default DomainAccountType set in `Join-AzStorageAccountForAuth` cmdlet is computer account. However, the password expiration age configured in your AD environment can be different for computer or service logon account and you need to take that into consideration for [Update the password of your storage account identity in AD](./storage-files-identity-ad-ds-update-password.md).

* <a id="ad-support-rest-apis"></a>
**How to remove cached credentials with storage account key and delete existing SMB connections before initializing new connection with Azure AD or AD credentials?**

    You can follow the two step process below to remove the saved credential associated with the storage account key and remove the SMB connection:

    1. Run the cmdlet below in Windows Cmd.exe to remove the credential. If you cannot find one, it means that you have not persisted the credential and can skip this step.
    
       cmdkey /delete:Domain:target=storage-account-name.file.core.windows.net
    
    2. Delete the existing connection to the file share. You can specify the mount path as either the mounted drive letter or the storage-account-name.file.core.windows.net path.
    
       net use <drive-letter/share-path> /delete

* <a id="ad-sid-to-upn"></a>
**Is it possible to view the userPrincipalName (UPN) of a file/directory owner in Windows Explorer instead of the security identifier (SID)?**

    In Windows Explorer, the SID of a file/directory owner is displayed instead of the UPN for files and directories hosted on Azure Files. However, you can use the following PowerShell command to view all items in a directory and their owner, including UPN:

    ```PowerShell
    Get-ChildItem <Path> | Get-ACL | Select Path, Owner
    ```

## Network File System (NFS v4.1)

* <a id="when-to-use-nfs"></a>
**When should I use Azure Files NFS?**

    See [NFS shares](files-nfs-protocol.md).

* <a id="backup-nfs-data"></a>
**How do I backup data stored in NFS shares?**

    Backing up your data on NFS shares can either be orchestrated using familiar tooling like rsync or products from one of our third-party backup partners. Multiple backup partners including [Commvault](https://documentation.commvault.com/commvault/v11/article?p=92634.htm), [Veeam](https://www.veeam.com/blog/?p=123438), and [Veritas](https://players.brightcove.net/4396107486001/default_default/index.html?videoId=6189967101001) and have extended their solutions to work with both SMB 3.x and NFS 4.1 for Azure Files.

* <a id="migrate-nfs-data"></a>
**Can I migrate existing data to an NFS share?**

    Within a region, you can use standard tools like scp, rsync, or SSHFS to move data. Because Azure Files NFS can be accessed from multiple compute instances concurrently, you can improve copying speeds with parallel uploads. If you want to bring data from outside of a region, use a VPN or a ExpressRoute to mount to your file system from your on-premises data center.
    
* <a id=nfs-ibm-mq-support></a>
**Can you run IBM MQ (including multi-instance) on Azure Files NFS?**
    * Azure Files NFS v4.1 file shares meets the three requirements set by IBM MQ:
       - https://www.ibm.com/docs/en/ibm-mq/9.2?topic=multiplatforms-requirements-shared-file-systems
          + Data write integrity
          + Guaranteed exclusive access to files
          + Release locks on failure
    * The following test cases run successfully:
        1. https://www.ibm.com/docs/en/ibm-mq/9.2?topic=multiplatforms-verifying-shared-file-system-behavior
        2. https://www.ibm.com/docs/en/ibm-mq/9.2?topic=multiplatforms-running-amqsfhac-test-message-integrity


## Share snapshots

### Create share snapshots

* <a id="geo-redundant-snaphsots"></a>
**Are my share snapshots geo-redundant?**  
    Share snapshots have the same redundancy as the Azure file share for which they were taken. If you've selected geo-redundant storage for your account, your share snapshot also is stored redundantly in the paired region.
  
### Clean up share snapshots
* <a id="delete-share-keep-snapshots"></a>
**Can I delete my share but not delete my share snapshots?**  
    If you have active share snapshots on your share, you can't delete your share. You can use an API to delete share snapshots, along with the share. You also can delete both the share snapshots and the share in the Azure portal.

## Billing and pricing

* <a id="share-snapshot-price"></a>
**How much do share snapshots cost?**  
    Share snapshots are incremental in nature. The base share snapshot is the share itself. All subsequent share snapshots are incremental and store only the difference from the preceding share snapshot. You're billed only for the changed content. If you have a share with 100 GiB of data but only 5 GiB has changed since your last share snapshot, the share snapshot consumes only 5 additional GiB, and you're billed for 105 GiB. For more information about transaction and standard egress charges, see the [Pricing page](https://azure.microsoft.com/pricing/details/storage/files/).

## See also
* [Troubleshoot Azure Files in Windows](storage-troubleshoot-windows-file-connection-problems.md)
* [Troubleshoot Azure Files in Linux](storage-troubleshoot-linux-file-connection-problems.md)
* [Troubleshoot Azure File Sync](../file-sync/file-sync-troubleshoot.md)
