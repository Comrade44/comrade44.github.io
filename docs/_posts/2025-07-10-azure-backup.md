# Overview
In this demo series, we're going to use Azure Backup and built-in data protection services to protect various workloads in Azure. Azure Backup can back up data from many sources, both cloud and on-premises, and here we're going to deploy two common scenarios, Azure blobs and files.

We're going to be using code from the data-lab repository [here](https://github.com/Comrade44/data-lab). You can either clone that repository and follow the instructions to deploy the infrastructure to your Azure subscription, or copy the Terraform config from the "azure-backup" folder and deploy it using your own method.

# Azure Backup
Azure Backup is Microsoft's PaaS backup solution. It allows you to back up data from storage accounts, VMs and other services. It can use built-in Azure native options or an agent to back up data on-prem and in Azure. You can find the Microsoft docs for Azure Backup [here](https://learn.microsoft.com/en-us/azure/backup/backup-overview)

## Vaults
The basis of Azure Backup is the vault. There are two types of vaults:
- Backup vault: Used for file-level backups from Azure blobs and disks.
- Recovery Services Vault: Used to protect a wider range of services, including VMs, SQL DBs on IaaS VMs, Azure Files and data from the Azure Backup agent.

In this demo, we're going to be setting up both types of vault; A backup vault to protect data in a blob storage account, and an RSV to protect Azure Files data.

To deploy the backup vault, copy the file "backup-vault.tf" into the Terraform deployment directory. Let's consider some of the configuration options:
- The Terraform resource "azurerm_data_protection_backup_vault" is used to create a backup vault.
- Although the datastore_type is a required parameter, at time of writing "VaultStore" is the only valid value, and doesn't correspond to anything in the Azure portal. THe other two possible options (ArchiveStore, OperationalStore) are available [through the API](https://learn.microsoft.com/en-us/azure/templates/microsoft.dataprotection/2021-01-01/backupvaults?pivots=deployment-language-terraform), but don't appear to be usable at the moment.
- You can specify the usual redundancy levels (GeoRedundant, LocallyRedundant, ZoneRedundant) using the redundancy parameter, and your choice in production will vary based on your requirements, but as this is a demo I'm just using the lowest level, LocallyRedundant.
- I've set soft_delete to "Off", as we'll have problems cleaning up the infrastructure after the demo if we don't, however the default setting is on and I'd recommend keeping it on in a live environment. Soft delete gives you a grace period after deleting data, during which you can "unDelete" the data in case you made a mistake.
- We're assigning the vault a system assigned managed identity. This allows us to grant the vault permissions to read and copy data from the storage account.

Next to deploy the Recovery Services Vault, we're going to use the config in the "recovery-services-vault.tf" file by copying it into the Terraform deployment folder:
- We use the Terraform resource "azurerm_recovery_services_vault" to create a recovery services vault.
- We're specifying the sku as "Standard", as it's a required parameter, however this is the only valid value.
- As above, we've disabled soft_delete to allow us to clean up the resources more easily after we're done, but I'd recommend not doing this in a live environment.
- And we're also assigning the vault a system assigned managed identity to allow it to be granted roles which allow it to read data from various sources.

In a live environment, you'd ensure that all services have private endpoints attached, and that traffic between the vault and the storage account doesn't leave the Microsoft network. As this is a demo, it doesn't matter here, but it's something to keep in mind.

## Storage Accounts
We're now going to need to add some data to back up. Copy the file "storage-account.tf" into the Terraform deployment directory. We're creating a storage account, a file share container and a blob container, as well as a blob. We're also randomising the name using the random_string block, as storage account names must be globally unique.

At this stage, we already have some options available for protecting Azure storage account data without using Azure backup. We've deployed the following features in storage-account.tf::
- [Soft delete](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview) provides an extra level of protection for blobs, files and containers. This can be used to protect either blobs (line 18) and their containers, or files (line 29). When a blob, container or file gets deleted, you can un-delete it within the number of days you specify hre as a retention period.
- [Blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-enable?tabs=portal), which we've enabled on line 25, allows you to restore different versions of blobs as they change. When you modify a blob, the older version before modification is saved and can be restored if necessary. This will increase storage costs, and can increase latency when accessing blobs with a high number (1000+) of versions, but this can be mitigated by configuring a backup policy correctly, see below.

You can see these options in action in the portal by doing the following:
  1. In the Azure Portal, navigate to storage browser for the storage account.
  2. Find the blob named "blob-backup-test" which we created in the Terraform config.
  3. Edit the blob to add an extra line with any text you want.
  4. Now navigate to the "versions" tab for the block. You should see an additional version which you can restore by selecting "Make current version". You can also access it directly using the URL provided, as well as generating a SAS token for the version. Effectively, you can treat it as if it's any other blob.
  5. If you delete the blob, you can restore it by navigating to container-1 and selecting "Show active and deleted blobs" from the drop-down menu. You can then restore it, including the versions you created, by clicking "Undelete".

## Types of backup
There are two main methods of backing up blob storage, both of which can be configured using a backup policy:
- Periodic or vaulted backups - These types of backups replicate specified blob data into a backup vault on a schedule defined by a backup policy. Periodic backups provide much more protection than a continuous backup. Since the data exists in the vault, even if the protected storage account or container is deleted, the data can still be restored elsewhere. It does have a higher cost though, as the data needs to be both transferred ot and stored in the vault.
- Continuous or operational backups - Continuous backups use the in-built features of Azure storage (Such as versioning, soft delete and point-in-time restore) to protect the data within a storage account. You can either configure them directly on the storage account, as we did in the section above, or use a backup policy to configure these settings centrally. This will cost less than a periodic backup, as there's no need to transfer or store the data in the vault, but provides less protection as if the container or storage account gets deleted, the protected data is deleted with it.

## Vaulted backups for Blobs
Here' we'll set up a periodic (vaulted) backup for our blob storage account. As we've deployed a backup vault as part of the initial Terraform config, there are a couple of things we're going to add to make this work. Copy the file "blob-backup-config.tf" and "blob-backup-policy-vaulted.tf" into your Terraform deployment directory. It contains the following:
- An RBAC role assignment to grant the backup vault identity the "Storage Account Backup Contributor" RBAC role on the storage account, to allow it to retrieve blob data.
- A blob backup policy named "blob-backup-policy-vaulted".
- A "azurerm_data_protection_backup_instance_blob_storage" to link the policy with the storage account.

You can also see that Azure Backup will automatically create a resource lock on the storage account. This requires you, before deleting the storage account, to go into the portal and remove the lock manually. We'll need to do this as part of our cleanup later in this demo, but for now you can see it by browsing to the storage account and navigating to "Settings > Locks".

The vaulted backup policy configures the following:
- A schedule to back up the blobs every day at 19:00 (backup_repeating_time_intervals). This is in ISO 8601 repeating time format, and for demonstration purposes we're running it every 5 minutes so we can test it (P5M)
- The setting "vault_default_retention_duration" specifies the amount of time to retain the data it backs up. This allows you to managed the number of versions available for each blob, as a higher number of versions can increase storage costs and latency.

We're also applying the policy to our storage account using the "azurerm_data_protection_backup_instance_blob_storage" resource, selecting the storage account, vault and policy. This creates a backup "instance", which you'll be able to see in the backup instances section of the backup vault.  

Now navigate to the "Data Protection" tab of the storage account, as there are a few things to observe here:
- The "Enable Azure Backup for blobs" option should be ticked, with information about the vault and policy that this storage account is associated with.
- Versioning and the blob change feed options are both enabled and locked.
- With a vaulted only backup policy assigned, You can choose to configure point-in-time restore, and soft delete for blobs and containers as required. 

The schedule we've configured won't provide an immediate backup, so go into the backup instance and select "Backup Now" to start a backup. Once the backup has completed, you can restore items by selecting the backup instance, selecting "restore" and choosing the restore point that you want to pull data from. This will spin up a snapshot as a new read-only container, that you can connect to using the web browser or SMB, to copy files out of.

### Operational backups for Blobs
The main difference with a continuous, or operational, backup is that the data is transferred into the backup vault rather than being stored within the storage account. This will cost more overall, but means that you'll have access to the protected data regardless of the status of the storage account. It can be configured either with a backup policy, or each of the settings can be enabled separately directly on the storage account. 

Before we proceed with creating an operational backup policy, it's worth highlighting a difference between configuring an operational backup in Terraform vs. in the portal. If you create the policy in the portal, you'll notice that on the "Schedule and retention" page, Vaulted Backup is always selected and you can't de-select it, but you can choose whether to configure an operational backup alongside it. Whereas when using the Terraform resource to create the operational backup as in our config, no scheduled backup is created (see backup instances) and only the operational settings are enabled.

, you can't configure a continuous backup policy alone, you have to enable it in addition to a vaulted backup policy. You can see this if you go into the Azure portal, in the backup vault and navigate to "Manage > Backup Policies", then click add. You'll notice that on the "Schedule and retention" page, Vaulted Backup is always selected and you can't de-select it, but you can choose whether to configure an operational backup. If you want to deploy only an operational backup to a storage account, you can enable all of it's features individually (blob versioning, soft delete, point-in-time restore) through the portal under the data protection tab (or in Terraform) without enabling Azure backup, as we did in the storage account section above.

Now we're going to replace the vaulted backup policy we created above with an operational policy. Remove the config file "blob-backup-policy-vaulted.tf" from your Terraform config directory, run an apply (to disassociate the policy from the storage account), then copy the file "blob-backup-policy-operational.tf" in and run another Terraform apply.

You can see the effects of this in the portal. You'll see a backup policy and a backup instance in the vault, and if you browse to the data protection tab of the storage account you'll notice a few things:
- The operational backup policy controls the settings for point-in-time restore and soft delete for blobs (but not for containers), as well as versioning and blob change feed. These options cannot be disabled with an operational backup policy assigned to the storage account.
- These settings can also be enabled on an individual basis even without Azure backup, as we saw in the storage account section. An operational backup policy mainly gives you the ability to centrally enforce these settings at scale without having to configure each storage account separately.

Next, let's have a look at backing up Azure Files shares.

## Backing up Azure Files
Azure File shares can be protected using file share snapshots. A file share snapshot is a point-in-time backup of the file share, from which you can retrieve files and data at the time the snapshot was taken. When you create a file share snapshot for SMB shares, it will appear in the "Previous versions" tab of the drive in Windows. You can also create file share snapshots for NFS shares in specific configurations (Premium SSD Provisioned v1). Snapshots can be created and managed without any additional infrastructure, but the functionality can be extended using Azure Backup.

To create a file share backup, browse to the storage account we created, in the Azure portal. From there, under "Data Storage > File Shares", select the share container-2 and go to "Operations > Snapshots". From here you can manually create a snapshot of the share. You can connect directly to snapshots as an SMB share, or as mentioned above, see all snapshots associated with a share in the previous versions tab.

To automate snapshots, we need to use a recovery services vault (NOT a backup vault). Copy the "recovery-services-vault.tf" file into your Terraform config directory and run apply, and as with the backup vault we're assigning it an identity and the "Storage Account Backup Contributor" role so that it can back up the data. 

There are two types of backup you can perform using Azure Backup:
- Snapshot - This creates the snapshot automatically, but doesn't transfer any data to the backup vault. Therefore, if you delete the storage account, or if there's an outage, the backup data might not be available.
- Vaulted - This creates a snapshot, then transfers the data to the backup vault, providing an extra level of protection. If the vault gets deleted, or if it doesn't have the right level of geo-redundancy and there's an outage, you'll still have the data and can restore it elsewhere (e.g. another storage account or file share). This is more expensive but provides better data protection.

You can see an example of a snapshot backup policy in the "backup-config-file.tf" file. Copy this file into your Terraform deployment directory and apply it.
- The resource "azurerm_backup_policy_file_share" sets the schedule. The backup block configures the snapshot frequency, and the retention_daily block configures how many copies of the snapshot get retained.
- We need to register the storage account with the recovery services vault (azurerm_backup_container_storage_account). This can be seen in the vault in the portal under "Manage > Backup infrastructure > Azure Storage Accounts".
- We're then linking that policy to the file share using the "azurerm_backup_protected_file_share" resource. Because this resource doesn't reference the azurerm_backup_container_storage_account resource directly, but still relies on it as a pre-requisite, we've added an explicit "depends_on" parameter.

Once the config is deployed, you can see the backup in action by browsing to the recovery services vault in the portal, selecting backup items, Then selecting "Azure Storage (Azure Files)". It will likely have a warning status of "Initial backup pending" as we've set the schedule to run at 23:00, but we can start a manual backup by clicking "Backup now" and accepting the default retention for the snapshot. This backup should finish pretty quickly as there's nothing in the container, but you can still see how we could restore a file if necessary:
- The "Restore Share" option allows you to select a restore point and either restore the files to the original location, or to a different file share.
- The "File Recovery" option allows you to select a restore point and recover a single file, either to the existing share or to a new share.

Note that with this type of backup policy (Snapshot), the data doesn't exist in the vault, instead it is held in the storage account. We can see this by browsing to the file share container and selecting "snapshots". As noted, this type of backup doesn't protect against the container being deleted. For that, you'll need a vaulted backup.

At the time of writing, [the Terraform azurerm provider doesn't support creating vaulted backups for file shares](https://github.com/hashicorp/terraform-provider-azurerm/issues/29100). We can created a vaulted backup policy manually with the following steps:
1. Go to "Manage > Backup Policies"
2. Click "Add"
3. Select "Azure File Share"
4. Fill in the details on the next page. Give it a name, select "Vault-Standard" as the policy tier, and configure the schedule as required (default settings are fine for this demo)
5. Next, go to "Protected Items > Backup items" and select "Azure Storage"
6. Click "Add", then select "Azure file share" in the drop-down and click "Backup"
7. Select the storage account we have provisioned and container-3 as the file share to back up
8. Select the vault-standard backup policy in the drop-down list and click "Enable Backup"
9. (optional, as this can take a long time even with no data in the container) Create a backup of container-3 as we did for container-2 using the "Backup now" option

Once the backup has been enabled, it will look and act almost the same as the snapshot policy backup we created. You still create restore points in the same way and use the same options for restoration. The key difference is the behavior on the back-end. If you look in the backup jobs tab of the vault, you'll see the backups we ran on container-2 and container-3. Notice that container-3 contains an extra step, "Transfer data to vault". The backed up data exists on the backup vault rather than in the storage account, meaning that for a vaulted backup, you can restore the data even after the container or storage account goes down or gets deleted.

# Cleaning Up & Final Notes
There are some things in this config that won't be cleaned up by Terraform when running a destroy operation. There is good reason for this, mainly to protect data from automatic deletion during IaC operations, but it does mean that we'll have to perform some manual steps when we're cleaning up the environment after this demo. Before running Terraform destroy:
- Azure backup creates resource locks on storage accounts when you register them with a vault for backup. To remove this, browse to the storage account in the portal, navigate to "Settings > Locks" and delete all the locks that exist here.
- In the recovery services vault, delete the backup of container-3 if you created one in the above section.

The policies we've deployed here are very basic, but there are many more options for scheduling backups within them. Read the [blob](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/data_protection_backup_policy_blob_storage) and [file](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/backup_policy_file_share) policy resource documentation for more detail. Try experimenting with more complex scheduling to find configurations that fit different requirements.
