

# Overview

Azure API Management (APIM) is a Cloud-based PaaS service that helps organizations publish APIs to external, partner, and internal developers to unlock the potential of their data and services. It includes a built-in gateway service that brokers network API calls to your backend so that you may enforce user-based access, take advantage of quotas and rate limiting, cache responses, and more.  A great overview of APIM on azure.com is here. 
If you are reading this whitepaper, it is probably because you are an IT Pro in need of guidance on how to deploy APIM in Azure to meet your high security data requirements.  Else you are just interested in what is possible with APIM and want to know more. Either way, this white paper is for you. Here, we will discuss exactly what happens when you deploy APIM inside an Azure VNet, and how to take your security footprint to the next level by using firewalls, Network Security Groups (NSGs), User Defined Routes (UDRs), and more. 

# APIM and VNet Injection

For Enterprise, high security is a must for any application foot print that contains sensitive data.  From the networking prospective, a common method to help protect this data is by blocking public Internet access to it, and only allowing access to it over the company’s private, internal network. This can also carry benefits to uptime and availability as well. Azure enables this functionality for many PaaS platforms by means of a technique called VNet Injection. 
VNet Injection uses Azure’s automation and SDN capabilities to deploy a given PaaS service directly into a specific customer VNet that has been configured with a special, delegated subnet for this purpose.  This VNet is typically an extension of the customer’s on-premises private network by way of VPN, ExpressRoute, or both.  The target VNet can be established and does not have to be created at the time of deployment.  The delegated subnet, however, will need to be empty and designated exclusively for the PaaS service prior to deployment. 
It’s good idea to make sure your subnet mask is no bigger than a /27 (e.g. /28 etc.) due to scale requirements in the future. Also, because a VNet is a flat layer 3 switch, you can always add a new network range to the VNet for this subnet, if need be. Remember, by default, any subnet in a VNet can talk to any other subnet in the same VNet.  NSGs and firewalls are used to curtail this. 
VNet injection carries some key benefits that Enterprise requires for high-security APIM deployments:

•	It allows for the creation of an APIM Gateway that listens on a private IP within the VNet. 

•	It supports RFC1918 or custom address space assignment to PaaS nodes and endpoints.  

•	It allows a PaaS service to talk to other VMs inside of the VNet.

•	It supports ExpressRoute and/or VPN connectivity.

•	It supports NSGs, ASGs, and UDRs by surrounding the PaaS service with a subnet. 

•	It allows for inbound and outbound security through devices like firewalls, WAF, IDS/IPS, etc. 

# Benefits of a Firewall with APIM
For any network service that is tasked with transmitting high security data, protecting this service with a firewall is a must. This can be for both inbound and outbound network traffic flows.  Because APIM is typically a web-based service, you can safeguard your inbound flows using a traditional firewall, a WAF, or any other combination of security platforms that are supported as Network Virtual Appliances (NVAs) within your VNet.  The Azure WAF is one example of a supported inbound security device here. 
For outbound flow protection, the common motion is to apply User Defined Routes (UDRS) to the delegated subnet to steer traffic though a firewall for inspection, auditing, and logging. For Internet-based/public destinations, you will need to apply a forced tunnel route (0.0.0.0/0) to your delegated subnet.  This makes your firewall to function as the default gateway for your APIM service. 
Many people will take advantage of “Next-Gen” or application firewalls here, as they provide the ability to filter outbound calls at the fqdn or application-based message level. This functionality is highly recommended to secure your APIM service.  Ideally, your application firewall is situated in your VNet to keep it close to your applications. However, the forced-tunnel route can also be injected via BGP over ExpressRoute or VPN, so your application firewall can live within your corporate network perimeter.  

# Understanding Traffic Types in VNet Injected Services
Control Plane
When a PaaS service is deployed into a delegated subnet by way of VNet Injection, it will be still be dependent on Azure management services for health checking, reporting, deployment, etc.  This type of traffic is referred to as control plane. It is somewhat confusing, because some will refer to this as “management traffic”, but in reality, this tier maps neatly into what the industry already understands as control plane traffic.  For any given PaaS deployment in a VNet, there will be both outbound calls to, and inbound calls from, these control plane endpoints.
Extra care needs to be taken here, because the control plane IPs are part of Azure’s own public IP ranges. These IPs will need their own static host routes (i.e. UDRs) to ensure that responses to inbound calls do not follow the forced tunnel route out to the firewall. Else, an asymmetrical response will occur, and the PaaS service will enter a degraded state. This white sheet will supply specific instructions on how to configure these special routes so that a forced tunnel route to a firewall can be supported.

Just how Public is the Control Plane?
It is critical to note that control plane traffic never leaves Azure’s internal network.  It will enter and leave your PaaS service on special public listeners that do not face the Internet. Control plane communication is strictly internal, Azure-to-Azure communication. These IPs need to be public because the Azure management service tier is multi-regional to withstand the failure of a single region and is often built on other Azure PaaS platforms under the hood.
In fact, even though you use the UDR tag “Internet” to point your management host routes away from your firewall, this just tells the Azure Network Stack to use the default gateway of the hypervisor host, which lives deep within an Azure data center. It only leads out to the Internet – and out of Azure’s internal network – if the destination is outside of Azure. Else, traffic pointed to the “Internet” tag stays inside of Azure and follows Microsoft dark fiber to its regional destinations. 

Management Plane
Management plane is a second kind of traffic that will be part of any PaaS service in a VNet. This type of traffic refers to user defined input that is sent into the PaaS service to configure the service according to a specific, desired outcome. This input can come in through the Azure Portal, PowerShell, Azure Cloud Shell, Visual Studio, or any other number of popular implementation tools, like Ansible, Terraform, etc. This traffic is inbound only and is typically RESTful over HTTPS. Management plane traffic needs to be highly secured in all instances, because it represents the “Keys to the Kingdom” for the IT pros who create and code in these PaaS environments.  Two important management plane platforms for APIM are the Azure portal, and the Developer portal.  However, there are also four special fqdns that provide management plane access for administrators. We will review these in another section below. 

Data Plane
The data plane is also referred to end-user traffic, or customer traffic. This is traffic that your end users generate when they access your service for content, to query it, to upload data, and so on. It can also be the traffic that your application generates to talk to other backend services in response to end user input. Thus, the data plane traffic can be both inbound and outbound depending on the application platform and design.

A Word about the Backend
A lot of Azure PaaS platforms use both Azure Storage and Azure SQL services to get the job done across all three planes, or some combination of them.  The classification of this backend traffic will need to be performed independently for each PaaS service that supports VNet Injection. For APIM, we will consider traffic to Azure Storage and Azure SQL as both control plane and data plane traffic, as the APIM reads and writes both types of data here. The bulk of the transfer does belong to data plane, however, so from a compliance or DLP perspective, APIM storage and SQL needs to fall into this same regulatory stance. 

# Internal vs External APIM 
When you deploy APIM into your VNet (https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet), you will have the choice of placing your APIM Gateway behind a Public Azure Load Balancer IP (External), or behind an Internal Azure Load Balancer IP (Internal). The setting that actually determines this outcome is found under the “Virtual Network” settings of your APIM. You will see a choice of “External” or “Internal” for your VNet deployment.  In either case, the actual APIM nodes themselves are build using private IPs from the subnet. 
Because high-security architectures often require moving endpoints to private network space and placing a firewall in front of the service, the “Internal” VNet mode is the ideal choice here.  For hybrid motions, this Internal Gateway IP will be accessible over VPN and or ExpressRoute, such that on-prem clients do not have to use the Internet to contact your APIM Gateway. 

The Internal APIM deployment (https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet#--routing) guide discusses in detail what will happen to your APIM management plane when you move in behind a private IP in your VNet.  The next section will dive into this in more detail. It is crucial that you read this solution carefully so you know how to design and access your APIM management plane. 

Management endpoints with Internal VNet mode.
As discussed above, the management plane is how admins and developers come in to configure and view your service.  In addition to the Azure and Developer portals, the following special management fqdns will need to resolve to the APIM private gateway IP, which will be hosting these services:

•	<ServiceName>.azure-api.net
 
•	<ServiceName>.portal.azure-api.net
 
•	<ServiceName>.management.azure-api.net
 
•	<ServiceName>.scm.azure-api.net 
 
You will need to set up a custom DNS solution for these four fdqns such that the A record for each of these hostnames will resolve to the private IP address of the APIM Gateway. The hostnames above are required in the HTTP request, such that the client will pass the correct hostname value to the APIM Gateway to forward to the right backend service. This solution does a great job of outlining how to set up custom DNS in Azure.  Note you should set up the custom DNS service prior to deploying APIM into the VNet, so your management calls work correctly post-deployment.

Per the linked Internal APIM deployment guide, there are two other strategies you can use to simplify your DNS configuration:
1.	Change the DNS fqdns of your APIM service to a domain name of your choice, e.g., change “contoso.azure-api.net” to “contoso.contoso-api.net”.
2.	Set up a VM jumpbox in your VNet and modify the /etc/hosts record for these fqdns such that they resolve to the private IP of your APIM service.

# The Essential Architecture
The best-practice implementation of the high security APIM architecture is provided below. It is provided as a framework to discuss the full inbound and outbound requirements in the following sections. 

1. An APIM service deployed in a delegated subnet:
 
    - Custom DNS will need to be configured prior to APIM VNet Injection 
 
    - The APIM Gateway should be configured in Internal mode.

2. A Routing Table applied to the APIM delegated subnet with these configurations:

    - A set of User-defined routes (UDRs) that steer egress control plane traffic to the Azure backbone using the “Internet” tag. 

    - A set of User-defined routes (UDRs) that steer egress management plane traffic into a private firewall IP using the “Virtual Network Appliance” tag.  

    - A set of User-defined routes that steer egress data plane management traffic into the firewall using the “Virtual Network Appliance” tag. 

    - A User-defined route that points the default gateway of the subnet (0.0.0.0/0) to the firewall using the “Virtual Network Appliance” tag, if your firewall resides in your VNet.  Else, this route will arrive via BGP into the APIM subnet over ExpressRoute or VPN. 

3. A Network Security Group (NSG) which will curtail inbound access. The best approach is to deny all inbound traffic, from both the Internet and the VNet, then white list the following groups:

    - Control plane, management plane, and data plane channels (discussed below in detail)

    - The APIM subnet itself for internal communication

    - Any private network ranges and ports that will be opening inbound connections to APIM. 

4. This NSG will also need to restrict outbound access. The best approach here is to deny all outbound traffic, to both the Internet and to the VNet, then white list the following groups:

    - Control plane, management plane, and data plane channels (discussed below in detail)

    - The APIM subnet itself for internal communication.

    - Any private network ranges and ports to which APIM will open outbound connections.
   
5. An application firewall in an adjacent subnet or VNet to provide inbound and outbound protection. 

# Symmetric Routing for Inbound APIM Control Plane Traffic

As we learned, the APIM control plane will be making inbound calls to your APIM delegated subnet, targeting a special management IP hosted by the fabric. This public endpoint will then forward control plane connection requests to the private IPs of your APIM role instances. In order to ensure that responses symmetrically map back to these inbound source IPs, you will need to create a route table and build a set of UDRs to steer traffic back to Azure by setting the destination of these host routes to “Internet”.  The host routes you will need to create are as follows:

1.	Destination: 13.84.189.17/32 => Next Hop: Internet

2.	Destination 13.85.22.63/32 => Next Hop: Internet

3.	Destination 23.96.224.175/32 => Next Hop: Internet

4.	Destination 23.101.166.38/32 => Next Hop: Internet

5.	Destination 52.162.110.80/32 => Next Hop: Internet

6.	Destination 104.214.19.224/32 => Next Hop: Internet

7.	Destination 13.64.39.16/32 => Next Hop: Internet 

8.	Destination 40.81.47.216/32 => Next Hop: Internet 

9.	Destination 40.90.185.46/32 => Next Hop: Internet

10.	Destination 20.40.125.155/32 => Next Hop: Internet

11.	Destination 52.142.95.35/32 => Next Hop: Internet

12.	Destination 51.145.179.78/32 => Next Hop: Internet

 
A Word About the Future of APIM Control Plane

Today the APIM control plane uses the four endpoints above for high availability. This range will grow in the future as Azure continues to grow. In the future, is possible that some control plane functions for your APIM deployment will dynamically shift to a new IP which is not listed above. If this happens, your first motion will be to open a support ticket and add the new host routes in to your Route Table.  Any disturbance to your control plane will not impact your other planes, and your end-user traffic will not be affected. 
To address this issue, Microsoft Azure is developing a special route tag that will dynamically support all control plane targets.  At the time of this writing, this is currently in development, but this white sheet will be updated when this feature is in production. 

# Security for Inbound APIM Traffic
When planning your high security deployment, it is essential to understand all the inbound sources and channels so that you can implement your routes, and NSGs, and firewall rules for maximum security. Here are the inbound services that will be calling to your APIM subnet:
 
Let’s map the “Purpose” column to the three traffic types we have discussed for a better understanding of how to fine tune your NSG:

1.	Client communication to API Management

    - This traffic will be both data plane and management plane calls and will hit the private IP address of your APIM Gateway (Internal VNet mode). 
    
    - The source IP can be restricted to the private NAT IP of your WAF or firewall.  The key idea here is that your firewall sits in front of APIM, and will forward calls from its trusted interface to your APIM Gateway. 
    
    - The destination IP range can be restricted to subnet mask of your APIM Gateway.
    
    - The source port range needs to be “*”
    
    - The destination ports need to be 80 and 443

2.	Management endpoint for Azure portal and Powershell

    - This traffic will be control plane only. Do not be confused by the term “management” here. 
    
    - The source IP range can be restricted to the “ApiManagement” NSG service tag
    
    - The destination IP range can be restricted to the subnet mask of your APIM Gateway.
    
    - The source port range will need to be “*”
    
    - The destination port range will need to the 3443

3.	Access Azure Cache for Redis Instances between RoleInstances 

    - This traffic will be internal data plane calls that will stay within your APIM subnet. 
    
    - The source IP range will need to be the APIM subnet mask
    
    - The source port range will need to be “*”
    
    - The destination IP range will need to be the APIM subnet mask
    
    - The destination port range will need to be 6381-6383

4.	Azure Infrastructure Load Balancer

    - This traffic will be internal control plane calls that will hit the Azure Internal Load Balancer that provides the private IP endpoint for your APIM Gateway.
    
    - The source IP range will need to be the “AzureLoadBalancer” NSG service tag
    
    - The source port range will need to be “*”
    
    - The destination IP range will need to be the APIM subnet mask
    
    - The destination IP port will need to be “*”
 


Routing for Outbound APIM Traffic
For steering outbound connection requests from your APIM deployment though an NVA firewall, you may need the following UDRs with the Route Table which is applied to the delegated subnet of APIM: 
1.	 Destination: 0.0.0.0/0 => Next Hop: Virtual Network IP
a.	This UDR will not be required if you are injecting 0.0.0.0/0 into the VNet over BGP. In this case, your firewall is on-prem. 
b.	This route will carry all public traffic permitted by your NSG to your firewall for inspection. This will include outbound traffic for all three types: control plane, management plane, and data plane.  
2.	Destination: [Azure subnet range] => Net Hop: Virtual Network IP
a.	These routes will differ per deployment and are optional.  They will forward traffic destined to other subnets or hosts in your subscription into your NVA firewall for inspection.
b.	These routes will technically service both requests and responses between this other subnet and your APIM deployment. 
c.	This is typically due to data plane calls to and from a backend service, but can also be for management or admin traffic from a jumpbox etc.
d.	It is important to note that the subnet or host in question will also need a corresponding UDR to the firewall for the APIM subnet, to ensure symmetrical traffic through your firewall. This would be configured in a separate Routing Table and applied to the remote subnet in question. 
 
Security for Outbound APIM Traffic
Here are the outbound destinations that APIM will need to access through your NSG. We will discuss each one in more detail:
 
Let’s map the “Purpose” column to the three traffic types we have discussed for a better understanding of how to build the NSG on your APIM delegated subnet for outbound security:
1.	Dependency on Azure Storage
a.	This will allow outbound data plane and control plane traffic to the dedicated Azure Storage tier for your APIM deployment
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “80” and “443”
e.	The destination IP range will need to be the NSG service tag “Storage.” It is best if you use the global tag in the event APIM needs to connect to back up storage in another region.
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level. 

2.	  Azure Active Directory
a.	This will allow outbound management plane traffic for admin and user authentication.
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “80” and “443”
e.	The destination IP range will need to be the NSG service tag “AzureActiveDiretory”
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level.
 
3.	Access to Azure SQL endpoints
a.	This will allow outbound data plane and control plane traffic to Azure SQL.
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “1433”
e.	The destination IP range will need to be the NSG service tag “SQL.” It is best if you use the global tag in the event APIM needs to connect to back up SQL in another region.
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level. 

4.	Dependency for Log to Event Hub policy and monitoring agent.
a.	Logging and monitoring traffic is part of parcel of the APIC management plane.
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “80” and “443”
e.	The destination IP range will need to be the NSG service tag “EventHub”. 
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level. 



5.	Dependency on Azure File Share for GIT
a.	The handling of code updates to/from GIT is part of the management plane for APIM
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “445”.
e.	The destination IP range will need to be the NSG service tag “Storage.” It is best if you use the global tag in the event APIM needs to connect to back up storage in another region.
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level.
 
6.	Health status to Resource Health
a.	Health status reporting will be initiated by each role instance inside of your APIM subnet and will be part of the APIM control plane
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “1886”.
e.	The destination IP range will need to be set to the NSG service tag “Internet”
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level.
 
7.	Publish Diagnostics Logs and Metrics
a.	APIM Logs and Metrics for consumption by admins and your IT team are all part of the management plane. 
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “443”.
e.	The destination IP range will need to be set to the NSG service tag “AzureMonitor”
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level. 

8.	Connect to SMTP Relay for sending e-mails (25, 587, 25028)
a.	APIM features the ability to generate email traffic as part of the data plane and the management plane. 
b.	The source port range will need to be “*”
c.	The source IP range will need to be the subnet range of your APIM deployment
d.	The destination port range will need to be “25”, “587”, and “25028”
e.	The destination IP range will need to be set to the NSG service tag “Internet”
f.	This traffic will follow the forced tunnel route out to your firewall and can be further protected at the application/fqdn level. 
 

Benefits of an Outbound Application Firewall
It is important to note that outbound IP destinations listed above are public IPs, and all of them belong to Microsoft. Thus, while you will need to open ACLs to “Internet” for things like SMTP and Diagnostics, the endpoints themselves are trusted Microsoft Public IPs.  Still, many high security deployments benefit from further inspection of outbound traffic at the application level – fqdns and hostnames, for example – to ensure that the destination of outbound connection requests are trusted endpoints, and not some unknown URL that could be due to fault in code, or worse. 
Fortunately, all of the outbound services above carry known fqdns that we can use for just such a purpose.  Most are printed in this solution and will need to be added to your application firewall’s outbound security settings for application level (layer7) whitelisting.  I add them here for your convenience: 
Outbound control plane and management plane fqdns for Azure Consumer Cloud
	prod.warmpath.msftcloudes.com
	shoebox2.metrics.nsatc.net
	prod3.metrics.nsatc.net
	prod3-black.prod3.metrics.nsatc.net
	prod3-red.prod3.metrics.nsatc.net
	prod.warm.ingestion.msftcloudes.com
	[azure region].prod.warm.ingestion.msftcloudes.com
o	where [East US 2] is eastus2.prod.warm.ingestion.msftcloudes.com
	SMTP Relay: ies.global.microsoft.com (25, 587, 25028)
	Diagnostic Log output: dc.services.visualstudio.com (443)
Outbound control plane and management plane fqdns for Azure Gov Cloud
	fairfax.warmpath.usgovcloudapi.net
	shoebox2.metrics.nsatc.net
	prod3.metrics.nsatc.net
	SMTP Relay: ies.global.microsoft.com (25, 587, 25028)
	Diagnostic Log output: dc.services.visualstudio.com (443)
Outbound control plane and management plane fqdns for Azure China Cloud
	mooncake.warmpath.chinacloudapi.cn
	shoebox2.metrics.nsatc.net
	prod3.metrics.nsatc.net
	SMTP Relay: ies.global.microsoft.com (25, 587, 25028)
	Diagnostic Log output: dc.services.visualstudio.com (443)
 
 
 
 
 
 
Per-Service FQDNs for APIM
The remainder of the fqdns that your APIM service will use, like for Azure Storage, SQL, EventHub, etc, will be customized per your deployment. To learn them, you will need to call a “Network Status” API post deployment, then add these fqnds into the security policy of your application firewall along with the others listed above. The Network Status API is listed in this solution and also here. 
 
IMPORTANT: Using ServiceEndpoints with APIM when your Force Tunnel
Currently, as of this writing, Virtual Network ServiceEndpoints are not supported directly by APIM. However, you can still make good use of them alongside your NVA firewall, and here is why: Without ServiceEndpoints, critical backend traffic between your APIM deployment and Azure will follow your forced tunnel route, which may lead to the subnet just next door, or all the way to your on-premise firewall. 
Technically, your on-premise firewall can in turn forward traffic to backend services like Azure Storage, SQL, and Eventhub back over the Internet and to Azure again, where these PaaS services are listening. However, this can introduce a fair amount of latency and slow down your APIM deployment’s data plane activity.  To avoid this scenario, you will need to move your firewall footprint into an adjacent subnet or VNet, next to your APIM subnet. This design is called a VDMZ or virtual DMZ.
When you have a NVA firewall living right next door to APIM and then force tunnel to it, a couple of really good things happen:
1.	You can enable ServiceEndpoints on the firewall’s egress subnet so that network traffic from APIM to the supported backend services (Storage, SQL, and EventHub) will flow directly to your firewall, then directly out of the VNet though Azure’s internal network. This keeps the traffic very secure and makes your round-trip times to your backend very fast! 
 

![alt text](https://github.com/jgmitter/images/blob/master/16.png) 
 
2.	For those control plane and management plane IPs are not supported by Service Endpoints, you can still use Azure’s “Internet” path on the egress NIC of your firewall to keep all that traffic on the Azure backbone. This will have the same outcome – your critical control plane and management plane traffic stays in Azure and reaps the benefit of security and speed. 
For these reasons, the high security APIM deployment really benefits from a VDMZ, as opposed to a forced tunnel route to an on-prem firewall. 

# Tips for Deploying your High Security APIM

1.	Deploy your VDMZ tier if you are going to use an NVA firewall

2.	Next, create the delegated subnet for your APIM in your target VNet. /29 is the smallest supported size but /27 is highly recommended.

3.	Next, set up custom DNS in the APIM VNet as discussed in the Internal APIM deployment guide (https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-internal-vnet). 

4.	Deploy your APIM into the subnet and set your VNet mode to “Internal”

5.	Prepare your application firewall with the right outbound and inbound policy

6.	Set up your route table correctly and apply it your APIM subnet

7.	Set up your NSG for inbound and outbound correctly and apply it to our APIM subnet. 




































# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all other rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.
