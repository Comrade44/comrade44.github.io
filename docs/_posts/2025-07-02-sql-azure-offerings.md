# Overview
There are three common ways that SQL can be deployed in Azure, and in this series we're going to deep-dive into all three. This post is going to be much more text-based, as there are a lot of features to explain that can't particularly be demonstrated (e.g. vCore vs. DTU models). The core infrastructure is based on the simple web app we set up [here](https://comrade44.github.io/azure/compute/2025/06/19/basic-web-app.html), so I'd recommend checking out that post first, or you can skip over that and copy the following files from the "basic-web-app" folder [here](https://github.com/Comrade44/app-lab):

- versions.tf
- sql.tf
- network.tf

As always, you can clone the repo and follow the instructions, or just copy the Terraform code and deploy it yourself.

## Azure SQL
The first and simplest method of deployment, and the one I'd generally recommend for greenfield deployments and as a default, is [Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview?view=azuresql). If you've deployed the Terraform config as-is, this will already be in place, and the config for it can be found in the "sql.tf" file. Azure SQL is essentially a PaaS SQL server, which abstracts away most of the management that comes with a traditional SQL server instance hosted on a VM, such as patching, backups, performance monitoring etc. You'll see it in the Terraform config as "azurerm_mssql_server". You can have multiple databases associated with the server, "azurerm_mssql_database".

### Authentication
Access to SQL is configured at the server level, and can either use SQL username & password authentication, Entra ID, or both. In the initial deployment, the web app uses a connection string with the SQL username and admin password embedded (see "AZURE_SQL_CONNECTIONSTRING" in "web-app.tf"). This is a less secure practice, and in production you'd normally use Entra ID authentication (Which we'll cover in another lab).
[Microsoft documentation on Azure SQL authentication](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-overview?view=azuresql)

### vCores & DTU
There are two pricing options when it comes to provisioning SQL databases - vCore and DTU. Here are the key differences:
- When choosing DTU, compute and storage scale together. There is a max data size per number of DTUs.You also can't use the serverless option with the DTU model, or select your hardware configuration. Overall it's simpler but much less flexible.
- vCore allows you to choose the hardware configuration and number of vCores independently of the storage size. It also allows serverless and hyperscale which are explained below.

Further information can be found [here](https://learn.microsoft.com/en-us/azure/azure-sql/database/purchasing-models?view=azuresql).

### Serverless
- [Serverless compute](https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview?view=azuresql&tabs=general-purpose) is available on the vCore standard and hyperscale pricing models.
- Serverless compute allows auto-scaling of compute resources, and is better for smaller unpredictable workloads as it doesn't scale up as well as other tiers.
- It also enables auto-pause, where the server pauses if no activity occurs for a specified time. Because of this, workloads which infrequently access the database may suffer from slower performance, as the DB needs to warm up again after being paused.

To deploy a serverless database, copy the file "serverless-db.tf" from the folder "azure-sql" into the Terraform deployment directory. This will deploy a general-purpose serverless database to the SQL server.
- The parameter auto_pause_delay_in_minutes specifies how long after no activity is received by the database it will wait until pausing the compute.
- The min_capacity parameter is the number of vCores that will remain allocated to the DB when there is no activity and the DB is not paused.

### Hyperscale
- [Hyperscale](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale?view=azuresql) is available on the vCore pricing model. It allows for an effectively unlimited amount of storage, and is good for workloads that require a DB size of over 4TB.
- It also allows for provisioning replicas in the same or a different region.
  
To deploy a Hyperscale DB, copy the file "hyperscale-db.tf" from the "azure-sql" folder into the Terraform deployment directory.
- Hyperscale is determined by the sku_name parameter, but allows additional options not available to a general purpose DB.
- Specifically, you can configure one or more read replicas when provisioning a hyperscale DB. We're not doing this here as we want to view this process in the portal for demonstration purposes.
- The SKU we're using is a serverless hyperscale SKU, in order to save costs. The same points above apply, specifically auto-pause will be enabled by default.
- The parameter "storage_account_type" corresponds to the "Backup storage redundancy" option in the portal. We're using LRS here to save cost, but in production you'd likely want a higher level of redundancy.

Let's take a look at the [replication options](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-replicas?view=azuresql) for hyperscale tier. If you navigate to the hyperscale database in the portal, and look at "Data management > Replicas", you'll see some options here:
- You can create two types of replica: named and Geo-scale.
- Named replicas are designed to provide a read-only copy of your database in the same region, and allow other modifications to be made such as authentication methods. They exist as a separate database object in the portal.
- Geo-replicas provide a readable replica of the primary database. It will always have the same name as the primary, you can only have one, and they must be created on a different SQL server to the primary.
- Notice also how the backup storage redundancy and elastic pools become unavailable when you select a named replica. You cannot use an elastic pool with a [named replica](https://learn.microsoft.com/en-us/azure/azure-sql/database/hyperscale-elastic-pool-overview?view=azuresql#limitations).

### Business-critical
- [Business critical](https://docs.azure.cn/en-us/azure-sql/database/service-tiers-sql-database-vcore#business-critical) is another vCore model service tier. We're not going to configure it in this demo, but I will provide a few observations.
- It is similar to general purpose but with better SLAs. Business critical adds three additional DB replicas to provide fast failover.
- It works in a similar way to an always-on availability group (covered later). A single writeable copy of the database is replicated three times.
- It can also provide a read-only database copy, to allow reporting and other read operations to be done without consuming resources on the primary repliace.

### Elastic Pools
- An elastic pool is a shared set of resources to which multiple SQL DBs can be associated.
- It is useful because autoscaling outside of the serverless tier isn't simple. This isn't a problem for databases with consistent utilization patterns, but for databases where the average utilisation is low, but with spikes of usage, selecting the right amount of compute is tricky. Do you provision a database which has enough compute to handle spikes in usage, but then end up paying for compute you aren't using? Or do you under-provision, saving cost, but then spikes in usage will mean that your database can't handle the load?
- Elastic pools allow you to share the resources of databases like these, so that when the compute you provision isn't in use by one DB, it can be used by another, therefore utilising resources to their fullest extent.
- This pattern means that the more of these types of database you add to a pool, the more efficient it becomes, as the provisioned resources will always be in use.
- It's also unsuitable for databases where the usage patterns are consistent, as we can just provision the DB on the correct pricing model. It's also not good for single databases of this type, as you still will be under-utilising resources in slower utilisation periods, as there aren't any other databases to use the un-used resources. Consider the serverless tier in this case.
- You need to look at average usage across all the databases you're going to add to the pool, to determine how to size your elastic pool.

We can provision an elastic pool and add some databases to it using the config found in "elastic-pool.tf" in the "azure-sql" folder. Copy the config into your Terraform deployment directory and apply it.
- An elastic pool is associated with a SQL server, and databases are then associated with it. Multiple pools can be added to a SQL server.
- The SKU is selected in the same way as you would select it for a DB (e.g. DTU, vCores, general-purpose, hyperscale etc.). See earlier in this post for information. The main exception being that you can't use the "serverless" pricing model.
- The "per_database_settings" block specifies the minimum and maximum number of vCores/DTU that a database linked to the pool can consume. This is to prevent one database from spiking and preventing other DBs from utilising the compute.
- The databases we've added, db-1 and db-2, have the "elastic_pool_id" parameter specified, linking them to the pool we've created. Notice that we haven't specified a SKU or min_capacity. This is handled by the elastic pool, and if you check the compute and storage tab in the portal, you'll notice that you can't change the compute without removing it from the pool.

## Managed Instance
Azure SQL Managed Instance provides an additional level of control over your SQL service, when compared to Azure SQL. It is effectively a dedicated VM or set of VMs where you can run SQL databases and connect to the SQL server instance using management tools, but with the OS abstracted away and managed completely by Microsoft. If Azure SQL is "SQL database as a service", then SQL MI can be considered "SQL instance as a service". It is much more expensive than SQL server, even using basic settings, so I'd advise caution when deploying it in your demo environment, or even not deploying it at all and just reading through the notes and config.

To deploy a SQL MI instance, remove all .tf files from your Terraform deployment directory apart from versions.tf and network.tf, then add the file "sql-mi.tf" from the "azure-sql" folder into it. This should destroy the existing SQL instance and databases, and create a new SQL MI instances. As always, some notes on this deployment:
- Notice with SQL MI you specify the SKU when deploying the server rather than the DB. With SQL MI, you're spinning up an actual server cluster and choosing the resources to assign to it, whereas in Azure SQL, the server you create is just a representation. Conversely, you don't choose the SKU/compute resources at database level, it will just use whatever is available on the MI server.
- The pricing model when choosing compute resources in SQL MI is also very different. Here, there's no hyperscale available, and no serverless compute option. There's also no DTU option, you specify the number of vCores and the amount of storage, then the service tier (General Purpose, Business Critical).
- SQL MI, similarly to a VM (as that's effectively what it is) requires you to specify the ID of a subnet to connect to. In fact, to enable any public access at all to the MI instance, public access must be explicitly enabled through a [series of steps](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/public-endpoint-configure?view=azuresql&tabs=azure-portal) which we're not going to demonstrate here. 
- The subnet for SQL MI will require a [delegation](https://learn.microsoft.com/en-us/azure/virtual-network/subnet-delegation-overview). To make this clearer, we're going to create a new subnet with a delegation for the SQL MI service, although you can use an existing subnet and just modify the settings. You can see the configuration for this in the azurerm_subnet resource. Specifically, the delegation block.
- Again as with the Azure SQL server, we're going to configure SQL authentication with a username and password hard-coded into the Terraform config. This is a terrible idea in a production or even a Dev environment, you should always inject any sensitive data such as usernames and passwords using variables or other methods. I'm simply doing it here to make the demo easier to use.
- One of the big differences with SQL MI is the way you manage databases. The standard azurerm_sql_database resource won't work here. You'll need to create and manage databases as you would with a traditional SQL server, by connecting to the instance using SQL server management studio or other tools. This presents a challenge from an IaC/idempotency standpoint, as the database won't exist in code, and you'll need to manage things such as transaction log shipping to back up the data.

## Azure VM
Deploying SQL to an Azure VM is the closest we can get to a more traditional on-premises SQL architecture. You'll have one or more VMs running SQL server, perhaps with shared storage and other supporting infrastructure connecting them depending on your setup (for example load balancers, which may be required when setting up multiple SQL instances in a highly available scenario. More on that in a future demo). It gives you the highest level of control and flexibility of all the solutions, but also requires the highest amount of administrative overhead due to the need to manage, update and configure the whole VM, operating system and software, and is likely to be more expensive for most scenarios.

I'd recommend using this type of solution as a quick solution when performing a lift-and-shift migration into the cloud, where refactoring or rebuilding your application would not be possible due to time and/or effort constraints. Using Azure migrate to pick up an existing VM or set, and migrate it into Azure, would achieve this. There are some scenarios where you'd use Azure VMs to host SQL in a greenfield scenario, but the more straightforward option if you're deploying a brand new solution is to use a PaaS SQL offering.

For this demo, we're going to deploy a VM using an image from the marketplace that includes SQL server. The code isn't particularly complicated but there are a few important things to note about this type of setup. To begin, remove all .tf files from your Terraform deployment directory apart from versions.tf and network.tf, then add the file "sql-vm.tf" from the "azure-sql" folder into it. This should deploy only the SQL server VM and remove any other PaaS instances.

- In production, you'd commonly have at least two VMs set up with high-availability configured between them (e.g. failover clustering, availability groups). This is an in depth topic that also covers PaaS offerings, which is best addressed in a separate demo.
- As before, we're being extremely dangerous and using a plaintext admin password again. Just wanted to point it out again and I'll continue to do so, as it's critical that plaintext passwords are never used for anything serious. The only real exception here is environments where we can assume that the system is compromised already and don't care about it, such as sandboxed/demo environments isolated from everything else where the infrastructure won't be live for very long.
- The VM image that we're using (See line 33 source_image_reference) installs a Windows Server 2022 VM with a developer licensed (free) edition of SQL server 2022. Requirements on versions will vary in production, so always use the correct image either from the Marketplace or an image gallery.
- Similarly, we're using a dev SKU for the VM (line 28 "size"), but in production you'll want to choose the correct one for your requirements. Will traffic to the server be consistently stable, or will there be spikes of usage with long periods of inactivity in between? VM sizing, especially when it comes to SQL server, needs careful consideration.
- Even after deploying the VM, you'll still need to account for a lot of administrative overhead. You'll need to configure the SQL server and database, configure backups and HA, Operating System updates, secure the server, among other things. This really is the most flexible but highest administrative effort offering.
- I also want to highlight the networking that we've deployed with the VM here. For demo purposes, we've added a public IP address that you can use to connect via RDP and manage the server. In production this is very insecure. Ordinarily you'd restrict access from the outside world using a combination of firewalls, NSGs and other connectivity tools such as Bastion. You can find out more in the previous demo I did on [hub and spoke networking](https://comrade44.github.io/azure/networking/2025/06/17/hub-spoke-networking.html). Needless to say, this configuration wouldn't be suitable for a production environment.

## Final Thoughts
Hopefully this demo has provided a useful brief on the main types of SQL services available in Azure. The final decision on which one you choose comes down to many different factors, including some of those that we've talked about here, as well as considerations in the [Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/pillars) and [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/).