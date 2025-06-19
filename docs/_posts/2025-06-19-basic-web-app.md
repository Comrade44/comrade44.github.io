---
layout: post
title:  "Azure basic web app"
date:   2025-06-19 14:00:00 +0100
categories: azure compute
---

# Initial Deployment
In this lab we'll be deploying a basic web app in Azure, based on [Microsoft's reference architecture for a proof-of-concept web app](https://learn.microsoft.com/en-us/azure/architecture/web-apps/app-service/architectures/basic-web-app). I'm going to be providing a follow-along guide and exploring some of the features in this solution, as well as some notes on how the solution relates to the pillars of the Well-Architected Framework.

To get started, the code being used is hosted in [this](https://github.com/Comrade44/app-lab) repository, specifically the config in the "basic-web-app" folder. Clone the repo and follow the instructions in the README.md to deploy using GitHub Actions, or just copy the Terraform config from the "basic-web-app" folder and deploy it directly.

## Deploying the network
We're going to be using a single network and subnet for this solution, with all services connected via private endpoint and public access disabled. This is deployed using "network.tf". Copy this file into your Terraform deployment directory and apply.

## Deploying the app service
In this section we'll be deploying the app service. We'll be using the [Azure App Service sample workload](https://github.com/Azure-Samples/app-service-sample-workload), which is a two-tier .Net application that uses a SQL database. The config for this can be found in "web-app.tf". Copy this file into your Terraform deployment directory and apply. Some points about this configuration:

- Deploying a web app consists of two resources - The app service plan, which defines the set of compute resources that are available to you, and the app that runs on it (in this case a Windows web app, but it could also be a linux app or function app).
- I've selected the "Shared D1" tier, which in Terraform is referred to as "sku_name=D1".
- As we're using the "shared" tier for the app service, "always_on" must be disabled. When "Always On" is disabled, the app is unloaded after 20 minutes of no access and takes some time to warm up on subsequent requests. [App Service General Settings](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#configure-general-settings)
- The [Terraform "random" provider](https://registry.terraform.io/providers/hashicorp/random/latest/docs) is used to generate the web app name, as it must be unique.

## Deploying the example app
In production environments, apps would normally be deployed to app service using a pipeline or a Git webhook. This may be covered in a future demo, but for this demonstration we'll upload the app as a .zip file on the Kudu dashboard. This can be done by:

1. Open the app service in the Azure portal.
2. Browse to "Development tools" > "Advanced tools"
3. Click "Go" to open the Kudu dashboard
4. On the dashboard, select "Tools" > "Zip Push Deploy"
5. To upload the zip file, drag and drop the file "web-app-code/SimpleWebApp.zip" into the page to deploy
6. Browse to the website and you should see the SimpleWebApp front page.

## Deploying the SQL server & database
We now need to deploy a SQL server for the back end. The file "sql.tf" deploys an Azure SQL server and database instance. Copy this file into your Terraform deployment directory and apply. Some notes on these resources.

- When deploying a SQL server and database, the resource "azurerm_mssql_server" represents the sql server, and "azurerm_mssql_database" is a database instance.
- For demonstration purposes, we're using a hard-coded password to make things easier (see line 12). This is a terrible idea in production, and you would normally insert it using an environmental variable or key vault secret. More on that later.
- Again we're using the random provider for the SQL server name, as it must be globally unique.
- We're also using the "sample_name" parameter in the azurerm_sql_database to populate it with sample SQL data.

## Connecting the app to SQL
The app requires that the environmental variable "AZURE_SQL_CONNECTIONSTRING" is populated with the connection string of the SQL database. There are several ways to do this (using appsettings.json, connection strings in the app config, etc.), but for clarity and to prevent having to modify the sample code, we need to add this as an environmental variable in the App Service settings. 

We also need to allow trusted Microsoft services to connect to the SQL server, to allow the app service to connect to the database through the SQL firewall. In later steps we're going to be enabling private networking, which will require further changes, but for now we're going to be connecting from the app service to SQL over the public internet. 

This is achieved in code in two places:

- Adding an azurerm_mssql_firewall_rule resource in sql.tf. See [this](https://learn.microsoft.com/en-us/azure/azure-sql/database/firewall-configure?view=azuresql#connections-from-inside-azure) article for details.
- Constructing a connection string in the app_settings block of the app service, using outputs from azurerm_mssql_server and azurerm_mssql_database.

Browsing to the web app URL and clicking "Database test" should show sample data from the database. 

At this stage, the infrastructure looks like this. The arrow shows the traffic flow of a typical request when the "Database test" button is clicked on the website:

![Initial configuration after deployment](/assets/basic-web-app/basic-web-app.png)

# Improvements
Whilst this app isn't intended for production use, there are still some things we could improve from the perspective of Microsoft's [Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/). In this section we're going to discuss potential improvements and even implement some of them.

- The App Service isn't very resilient. Regional, datacenter or even hardware failures could mean down time for the app. As the App Service is on the "Shared" tier, Availability Zones aren't available. Scaling up the App Service Plan to at least "Premium v2" will allow use of availability zones to protect against datacenter-level outages. For even higher resilience, a multi-region deployment could be considered, and may be covered in a future post.
- The app can't scale to accommodate increased demand. Upgrading the app service plan to at least the "Premium v2" tier will allow for automatic scaling, although for an app of this size that would be overkill.
- When deploying code to the app, it's expected that there will be some down time. This can be mitigated using the deployment slots feature, with a secondary spot warmed up and ready to switch into production. This isn't available in this solution, as it requires the app service plan to be on at least the standard tier.
- The SQL server is also very basic when it comes to redundancy, resilience and scaling. A future post will investigate options around SQL server availability, of which there are many.
- There isn't any monitoring set up at this time. This prevents observability of the system, increasing the possibility of an unexpected outage. There are two complementary resources that can mitigate this, Azure Monitor and Application Insights, and in the next section we'll look at how to implement them.
- All traffic between components is via the public internet. This is very insecure, as anyone is able to intercept traffic between e.g. the app and the SQL database. This can be mitigated by enabling private networking, which we'll look at in the next sector.
- Traffic to the app from the internet should also have some sort of intermediary such as Azure Firewall or Web Application Firewall. Again there are many different ways to do this, but we're not going to cover any of them in this demo.
- The password for the SQL database is in plain text in the Terraform config. This is simply for ease of use, but should never be used in production. Valid methods of password configuration could be either reading the password in from a key vault, or using Entra ID authentication on the SQL database and setting up a service principal for the app. Both of these are quite in-depth discussions which would be best left to their own demonstrations.
- We deployed the code to the app using .zip file upload. This is rarely done in production. More commonly you'd use either a pipeline or webhook based deployment. Again these are in-depth topics that won't be covered here.

## Monitoring
Logs and metrics need to be sent to a Log Analytics Workspace in order to use them for things such as sending alerts and tracking application performance. The config in the file "monitor.tf" will deploy a Log Analytics Workspace and configure a diagnostic setting to send all logs and metrics from the app service and the SQL database to it. Copy the file into your Terraform deployment directory and apply it. Some things to note here:

- When enabling logs, as seen in the enabled-log block of the azurerm_monitor_diagnostic_setting blocks, the easiest way to add all available logs is to do a data lookup as seen in azurerm_monitor_diagnostic_categories, then use for_each to loop through each log category. The longer way is to have a enabled_log block for each log category you want to enable, which is time consuming and error-prone.
- It takes some time to ingest the logs into the workspace. If you wait 5-10 minutes after deployment, then browse to the Log Analytics Workspace, then select "Logs", you will see the tables containing the logs forwarded from the app and sql database.
- Similar with Metrics, If you wait 5-10 minutes after deployment, then browse to the Log Analytics Workspace, then select "Metrics", you will see the metrics that the app and sql database send to the Workspace.

## Application Insights
Application Insights sends logs from directly within the application to allow you to troubleshoot things such as request failures. In this solution, an azurerm_app_insights resource is provisioned, and you can enable application insights on the demo app by adding the "APPLICATIONINSIGHTS_CONNECTION_STRING" key to the application config. To do this, uncomment line 40 and 41 (to enable the agent) in the "web-app.tf" code and re-apply, after deploying the config in "Monitor.tf".

- This procedure configures Application Insights for a .Net application, using the most up-to-date method of [connection strings](https://learn.microsoft.com/en-us/azure/azure-monitor/app/connection-strings). There are other methods to deploy App Insights for your application, depending on the type of code you're deploying (e.g. Python, Node). This is beyond the scope of this demo.
- To verify that App Insights is working, browse to the web app's URL, hit "Database test" a few times, then browse to the Application Insights resource deployed by "Monitor.tf". Under "Investigate > Failures", you should see a graph showing the successful request count increasing, indicating that the app is sending metrics to Application Insights.

## Private Networking
Finally we're going to enable private networking, to stop traffic between the app and the SQL server from leaving the vnet. Copy the config "network.tf" into your Terraform deployment directory. This will create a few different things:

- A Virtual Network and subnet
- A private DNS zone for SQL service. More on that [here](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration#virtual-network-workloads-without-azure-private-resolver)
- VNet links from the DNS zones to the vnet
- A private endpoint for the SQL service, connected into the subnet
- I've omitted a private endpoint for the app service, as it isn't completely necessary for this demonstration and requires the plan to be on at least the "Basic" tier, however in production you would almost certainly deploy one.

After the improvements added above, this is how a typical request should now flow between client, app service and SQL:

![Final deployment](/assets/basic-web-app/final.png)

Notice that:
- The client still connects to the app service over the internet. This is necessary as the private endpoint only provides an internal IP. As above, this connectivity would normally be protected by a Web App Firewall, App Gateway or other configuration in production.
- To talk to the SQL database, the web app uses it's private endpoint to communicate with the private endpoint of the SQL service. It uses the private DNS zone privatelink.database.windows.net to resolve an internal, non-routable IP address for the SQL server, so the traffic can't leave the Microsoft network. This would also apply if the SQL service needed to initiate a connection to the app service.
- Normally you'd put NSGs on all of the subnets and have an explicit deny rule to prevent unspecified traffic from entering or leaving the subnet, however for demonstration purposes we're leaving this off for now. In production you should always have NSGs as the most basic form of network protection.

# Final notes
Although this is really more of a proof-of-concept design, the core principles for an N-tier application are the same:

- Internet-facing app service connecting to a database on the back end
- Traffic configured to stay within the Microsoft backbone network
- Azure Monitor configured to provide observability

There's a lot more about this type of architecture to explore. We're going to use this as a basis for looking at other concepts in more detail. Next we're going to focus on some of the availability options for Azure MS SQL in this solution.
