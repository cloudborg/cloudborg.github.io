+++
title = 'Aws Transit Gateway'
date = 2024-08-24T00:44:23+05:30
draft = false
+++



# Overview
![tgw](images/aws-transit-gw.png)


#### Why might I want to use Transit VPCs instead of Transit Gateways?

[AWS Transit Gateways](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html) provide no capability to examine the content of transitive traffic. This would need to be done using EC2-hosted software in a transit VPC. 

- Transit VPCs are an un-managed option, requiring more effort from the customer to manage resources. 

- Applying security controls to traffic entering a single, shared-services VPC, or traffic leaving your network for the internet can be accomplished by routing traffic via a Transit Gateway to a spoke VPC, and does not require use of a Transit VPC.


#### Which Direct Connect components cannot directly interact with Transit VPC architecture?

- Transit VIF

- Direct Connect Gateway

- Public VIF

For Direct Connect components to interact with a Transit VPC architecture, they must be able to associate with a detached Virtual Private Gateway. Public and Transit VIFs cannot associate with VGWs, while Direct Connect Gateways may only associate with VGWs that are attached to a VPC.


#### After creating a Transit Gateway infrastructure with default settings, you realize that, while some networks must be accessible from all networks, other networks should be isolated from each other. Which steps must be performed to configure new route tables for restricted networks while continue using the default route table for shared networks?

- Disassociate the restricted networks from their current route table and associate them with a new route table containing routes to the shared networks.

- Create additional route tables as needed for your restricted networks.

- Configure the shared networks to propagate routes to a new route table. This will allow traffic to be routed from your shared networks to your restricted networks.


#### Which of the following statements about VPC attachments to Transit Gateways are true?

VPC attachments to TGWs must have at least one subnet associated. 

- AWS customers are charged per TGW attachment. 

- VPCs propagate their CIDR to appropriate TGW route tables, not their local or static routes. 
- Routes from the associated TGW route table are never propagated to VPC route tables. Routes from the VPC to the Transit Gateway must be added manually.

#### Transit VPCs support transitive networking by:

Transit VPCs support transitive networking by hosting software VPN systems that connect to other VPCs using site-to-site VPN connections. 

- Allowing other VPCs to establish site-to-site VPN connections with the transit VPC's VGW or establishing VPC peering relationships to other VPCs will not create transitive networks.

- While using dynamic routing to distribute VPC prefixes throughout the network is recommended, it alone does not enable transitive networking.

#### A Transit Gateway receives traffic destined for 192.168.1.34. Which of the following route table entries will be used?

The TGW will use the 192.168.1.0/24 static route to a DX Gateway. 

- Routes with longer prefixes are preferred, then static over propagated routes. Finally, within propagated routes, VPC over Direct Connect over VPN. In this scenario, all routes have the same prefix length, but only one route is static.


#### Which of the following statements about AWS Site-to-Site VPN attachments to Transit Gateways are true?

TGW VPN attachments create a VPN connection to a new or existing Customer Gateway in the same region, not to Virtual Private Gateways. 

- VPN attachments also incur standard AWS Site-to-Site VPN charges in addition to TGW attachment charges. 

- Both static and dynamic routing are supported for VPN attachments.


#### By default, each Transit Gateway attachment may propagate to how many TGW route tables?

- TGW attachments may propagate to as many route tables as the TGW can support, which by default is 20.


#### An organization using a Transit Gateway architecture wants to have all internet-bound traffic processed by customer-managed security appliances in a DMZ network. Traffic to known bad-actor networks should be prevented while minimizing traffic to the DMZ network. Which of the following features should be used to enable proper routing?

Default routes should be used in applicable TGW route tables to send internet-bound traffic to the designated network attachment. Blackhole routes to the bad-actor networks should be created in TGW route tables to cause matching traffic to be dropped by the TGW before it reaches the DMZ network.

- Additional TGW route tables and TGW peering might be required depending on other network needs, but the question does not specify any situations requiring them. 

- If the DMZ network is hosted in AWS, then an Internet Gateway would be needed, but the question does not specify where the DMZ network is located. 

- While Network ACLs in a VPC could prevent traffic to the bad-actor networks, the traffic would only be blocked as it is leaving the DMZ network, which does not satisfy the scenario requirement to minimize traffic to the DMZ network.

#### The simplest way for an on-prem network to interconnect to a transit VPC architecture using VPN is to:

The simplest way for an on-prem network to interconnect to a transit VPC architecture using VPN is to connect to the EC2-hosted VPN systems in the transit VPC. If dynamic routing is used for all connections, the on-prem network will be able to reach all of the spoke VPC networks. 

- Connecting to a VGW attached to the transit VPC would prevent transitive communication to the spoke VPCs. 

- Connect to the VGWs attached to each spoke VPC could allow full connectivity but is not the simplest option. 

- Connect to a Client VPN endpoint in the transit VPC could allow full connectivity but is not the simplest option.


#### **True or false**: spoke VPCs connected to a transit VPC can have direct communication by provisioning a fully operational peering connection between the two spoke VPCs.

**TRUE**.Spoke VPCs may be directly peered together, but static routes to the peered VPCs must still be added to their route tables, otherwise traffic will still flow through the Transit VPC, as long as appropriate routes are in place.