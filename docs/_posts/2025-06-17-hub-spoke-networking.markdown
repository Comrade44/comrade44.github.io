---
layout: post
title:  "Azure hub-and-spoke networking and Virtual WAN"
date:   2025-06-17 11:04:00 +0100
categories: azure networking
---

# Overview & Setup
This post is based on the hub-and-spoke networking demo repository here: [GitHub Repo networking-lab](https://github.com/Comrade44/networking-lab). To use and follow along with the instructions in this article, you can either clone the repo and follow the instructions for setting it up in your own GitHub organisation, then use the pipeline to deploy the infrastructure, or use the Terraform files as-is and deploy using another method.

To begin, the following Terraform config should be deployed by copying the files from the "hub-spoke-demo" folder into the "terraform" folder. All numbered config files should be left outside of the terraform directory, or out of the deployment code path:

- uks-spokea-net.tf - Deploys the networking components for the spoke A vnet
- uks-spokea-vm.tf - Deploys a VM into the spoke A vnet for testing purposes
- uks-spokeb-net.tf - Deploys the networking components for the spoke B vnet
- uks-spokeb-vm.tf - Deploys a VM into the spoke B vnet for testing purposes
- bastion.tf - Deploys a Bastion instance and connects it into the spoke A vnet, to allow testing from vm-a

Where an instruction is given to use the config in a certain file, copy the file or it's contents into the Terraform folder.

## Basic networking
As standard, outbound internet access for resources connected into an Azure Virtual Network is directly out to the internet, using one of Azure's random outbound public IPs. This is due to be deprecated in September 2025 [Microsoft Learn article](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access).
Communication between vnets requires a bi-directional peering.

![Default outbound internet access from Virtual Machines](/assets/hub-spoke-networking-azure/1.png)

1. Log into the spoke A VM via Bastion. 
2. Run "curl https://ipinfo.io" to retrieve the IP address of the VM as seen from the internet. Notice that it will be one of the randomly assigned Azure public IP addresses.

This can be controlled further by adding NAT gateways to each subnet where VMs will access the internet. NAT Gateway allows resources from within the network to connect to the internet whilst remaining fully private, using Source NAT and an assigned public IP. A NAT gateway needs to be added for each VNet, as using a NAT gateway in a peered vnet does not work without an NVA.

![Virtual Machines with outbound internet access via NAT gateway](/assets/hub-spoke-networking-azure/2.png)

1. Deploy the config in the file "2.tf". This will deploy a NAT gateway and connect it to each of the spoke subnets.
2. Log into the spoke A VM again via Bastion.
3. Run "curl https://ipinfo.io" again. Notice that the IP should now be the public IP address of the NAT gateway.

## Hub-and-spoke networking
For larger networks, it is recommended to configure a hub-and-spoke topology per-region. Hub-and-spoke means having a central "hub" Virtual Network, where core services and network gateways are hosted, and "spoke" vnets connected into it for each application or workload.

![Basic hub-and-spoke topology](/assets/hub-spoke-networking-azure/3.png)

Without Azure Virtual WAN, peerings, routing and outbound internet connectivity must be managed by code or config. 

For outbound internet access via the hub, some requirements must be met:
- A Firewall must be deployed and connected to the hub vnet
- Peerings must exist between the spokes and the hub
- A default route must be created on each subnet pointing to the firewall

1. Remove the config deployed by "2.tf".
2. Add the config in "3.tf". This will deploy a hub net and bi-directional peerings between hub and both spokes.
3. To send outbound internet access from the VMs via the hub firewall, add the config in "4.tf" into your Terraform deployment and "4-routing.tf" to configure a default User-defined route to the firewall and assign it to both spoke subnets.
4. Once deployed, connect to the spoke A VM and run "curl https://ipinfo.io" again. You should see that the IP address has changed to the public IP address of the firewall.
5. You should also be able to connect to the spoke B VM at this stage, and can verify this by running "ssh {spoke B VM IP}". This requires that traffic forwarding is enabled on each of the vnet peering connections. This can be seen in the "3.tf" code as the parameter "allow_forwarded_traffic".

![Outbound internet access via hub network](/assets/hub-spoke-networking-azure/4.png)

For communication between the spokes, an additional step needs to be taken. The peerings need to allow forwarded traffic, and for testing purposes, port 22 ssh needs to be allowed through the firewall.

## Hub NAT Gateway
Adding a NAT gateway in front of a hub firewall, whilst adding cost, brings some advantages. Azure Firewall allows 2496 SNAT ports per public IP address, whereas NAT gateway provides 64,512. This has the advantage of allowing more outbound connections from distinct services (e.g. microservices), as well as having fewer public IPs to manage.

![NAT gateway hub firewall](/assets/hub-spoke-networking-azure/5.png)

1. Add the config in "5.tf" to your Terraform deployment code location. This will deploy a NAT gateway into the hub subnet, and associate it. This will automatically send all traffic from the Azure Firewall to the internet via the NAT gateway.
2. Connect to the spoke A VM and run "curl https://ipinfo.io" again. You should see that the IP address is now the NAT gateway public IP rather than the Azure Firewall's.

## Virtual WAN
Azure Virtual WAN simplifies the management of vnets and routing in a hub-and-spoke topology (and other topologies):
- Routes are created and propagated automatically, including for default internet-bound traffic
- Peerings are created and managed automatically as spokes are added to the hub
- Scales better for multi-region WANs, as it removes the need for multiple peerings and UDRs in code

![Virtual WAN](/assets/hub-spoke-networking-azure/6.png)

1. To deploy the hub & spoke network using vWAN, remove all numbered .tf files from your Terraform deployment directory, and add in "6.tf". This will deploy a vWAN, a hub in UK South, and hub connections to both of the spokes.
2. You will notice if you connect to the spoke A VM and run "curl https://ipinfo.io", the IP address will be one of Azure's random outbound IPs. Access to the internet from a subnet without a hub-connected Firewall or NVA is direct. Additional steps below are needed to enable central internet routing.

## vWAN with Firewall

Adding a firewall to a VWAN hub provides outbound connectivity. In order to specify a default route outbound via the firewall for connected spokes, a "routing intent" must be configured.

![Central internet routing](/assets/hub-spoke-networking-azure/7.png)

1. Add the code in "7.tf" into the configuration. This will configure an Azure FIrewall in the hub, and add a "routing intent" which propagates a default route to all connected vnets via this firewall.
2. Connect to the spoke A VM and run "curl https://ipinfo.io". The outbound IP address should be the public IP of the Azure firewall.

## Further concepts
This article showcases the basics of hub-and-spoke networking with and without Azure Virtual WAN. There are many other considerations which aren't covered here but which can be used to extend this topology, including but not limited to:

- Adding site-to-site connectivity using VPN, expressroute etc.
- Adding a NAT gateway in front of the Azure Firewall when deploying a Virtual WAN
- Extending into multi-regions. This particularly showcases the advantages of vWAN, as managing multi-region network deployments requires much less code when deployed through vWan.