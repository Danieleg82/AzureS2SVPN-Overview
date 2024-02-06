# Azure S2S VPN: Overview & comparison of most common settings

## INTRO

Does it still make sense to speak about classic VPN solutions in 2024, when it comes to connectivity to cloud?
The short answer is: actually YES!

Despite being true that, when talking about connectivity of big datacenters to the cloud infrastructure, the most reliable and optimized approach relies in the adoption of ExpressRoute-based connectivity, the classic internet-based solution of VPN links still represents the most common way to provide failover capabilities in case of ExpressRoute failure, and the same is valid for connectivity of small branch offices.

SD-WAN technologies are as well playing a big role in this period, but when talking about those we actually talk again about IPSEC links: abstractions built as overlays on top of underlying connectivity layers (Internet / MPLS / ExpressRoute itself…).

So yes, in 2024 VPN is still a real thing, and when it comes to VPN the best option which can be integrated in Azure cloud today is still the good old Azure Virtual Network Gateway / VPN Gateway.

Is that the only option we have?
Of course not, we can build VPN links with any kind of 3rd party appliance (NVA) available in the market place, but the Virtual Network Gateway (VNG):

-	It’s a PaaS service, integrated with Azure control plane
-	It allows to program routes of VPN branches automatically on the VMs of your Azure environment, without need of UDRs configuration
-	It represents an automatic failover mechanism for broken ExpressRoute links
- It supports both Site-to-Site and Point-to-site connectivity methods
-	It’s implicitly highly available
-	It supports built-in monitoring attributes integrated with Azure Monitor 

It still represents a very utilized service, simple and basilar but very common.
Simple…but some times not so simple as it could seem 😊
In this article I want to show you some aspects of the VNG which typically still create confusion in many users: we will talk about Active/Standby and Active/Active modes - what’s the real difference between those, the most common setups and challenges. 


## INTRO OF AZURE VIRTUAL NETWORK GATEWAY

Every VNG in Azure is a logical object composed by 2 underlying **instances**.

Every instance (we can call them **IN0** and **IN1**) is actually a Virtual Machine, injected in customer’s VNET.
The configurations of these IPSEC terminators is managed through 2 specific logical objects:
1.	**Local Network Gateways (LNG)**: these are a logical representations of the remote terminators we want to connect to our VNG. Every LNG contains info about:

-	Remote gateway’s public IP or FQDN
-	Remote network’s IP ranges (if static routing)
-	Remote gateway’s BGP endpoint (if dynamic routing) & ASN

2.	**Connections**: these are logical objects used to link a VNG to a LNG. Connections are programmed with:
-	The shared key used for the IPSEC encryption
-	The IKE protocol to use
-	The decision of using BGP or not
-	Custom IPSEC policies to be applied (if any)
-	Other specific IPSEC parameters
  
When we link a VNG to a _LNG_ through a _Connection_, the Azure control plane programs the 2 VMs “behind” our VNG with the provided parameters, so that they start attempting IPSEC connectivity to the remote side.
You can configure VNG in **Active-Standby** (A/S) or **Active-Active** (A/A) modes by accessing the “Configuration” section of the gateway itself:

 ![](Pics/1%20-%20GUI.png)
 

…or leveraging Powershell/CLI scripting:

https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-activeactive-rm-powershell#part-1---create-and-configure-active-active-vpn-gateways 

Example:

```
New-AzVirtualNetworkGateway -Name $GWName1 -ResourceGroupName $RG1 -Location $Location1 -IpConfigurations $gw1ipconf1,$gw1ipconf2 -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw1 -Asn $VNet1ASN -EnableActiveActiveFeature -Debug 
```

**EnableActiveActiveFeature** = $false --> Active/Standby
**EnableActiveActiveFeature** = $true --> Active/Active

### Azure VPN S2S in Active-Passive mode:

![](Pics/2%20-%20AP.png)
 
**Characteristics**:

When working in Active/Passive mode, only a single VM instance of the VNG is effectively active.
The second instance is ready to take over in case of issues with the first instance.

These issues could be: 
1.	Instance down / not operative due to issues
2.	Planned maintenance events

<ins> The failure in connectivity of an IPSEC tunnel does not represent a reason for failover. </ins>

The instance which is considered active attempts connectivity to the remote endpoints according with the configured Connections 

**A single and unique Gateway public IP exists**, which is associated with the instance considered active.
In case of BGP connectivity, **a single and unique BGP IP is configured on the VNG**, which is associated again with the instance considered active.

**Failover events**:

![](Pics/3%20-%20AP%20failover.png)

In case of failure or planned maintenance events, the VM instance which was standing by takes over from its peer, and becomes the new Active instance.

The Gateway public IP is moved to the new active instance, so the VM is able to build IPSEC connectivity with remote peer.

During planned maintenance events, a process of Security Associations migration takes place between the instance which is going to go standby and the new active instance: this way there will be no need to rebuild IPSEC tunnels, the move from one instance to the other will be transparent from the point of view of the remote endpoint.

During unplanned events the rebuild of IPSEC tunnel may take up to some minutes.
In case of BGP routing, the new active instance keeps using the same BGP IP, so once again remote side will not be aware of any change.

**Pros**

Simpler configuration, compared with Active/Active --> we have to build a single link.
No possibility of mistakes in terms of configuration: you can connect to a single VNG Public IP, and the BGP IP of the GW is persistent across instances.

**Cons**

Performances are capped to the limits of a single VM instance.
In case of unplanned failure, even with BGP, the rebuild of IPSEC tunnels may take up to minutes, hence possibility of longer outages.


### Azure VPN S2S in Active-Active mode:

![](Pics/4%20-%20GUI%20enable%20A-A.png)
 
![](Pics/5%20-%20A-A%20generic.png)

**Characteristics**

When working in Active/Active mode, both the VM instances of a VNG are active, hence attempting to build IPSEC connectivity with the remote endpoints, again according with the Connections configured on the GW.

Every VM Instance is associated with its own Public IP (we’ll hence have 2 IPs).
Every VM Instance is associated with its own BGP IP (we’ll hence have 2 BGP IPs).

Remote endpoints are supposed to create 2 links:
-	One link between remote IP and VIP_0
-	One link between remote IP and VIP_1
  
… and 2 BGP peerings:

-	One peering between remote side’s BGP peer IP and the BGP IP of IN_0 (BGP_IP_0), on top of the IPSEC tunnel built with VIP_0
-	One peering between remote side’s BGP peer IP and the BGP IP of IN_1 (BGP_IP_1), on top of the IPSEC tunnel built with VIP_1

We can here split between different scenarios.


#### Single link enabled, Static routing

![](/Pics/6%20-%20AA%20single%20link.png)
 
In this scenario customer decides to connect a single remote endpoint to a single VM instance of the VNG, in a scenario of Static routing
You may ask: WHY?

Some times, customer do not really have necessity to implement Active/Active solution for the additional complexity it introduces, but there are scenarios where a VNG has to be configured in Active/Active mode as mandatory step, like for example when leveraging Azure Route Server in the same HUB VNET to enable Branch2Branch transitivity in VPN + ExpressRoute coexistence.

Some other times, we simply forget about configuring a second tunnel on our local terminator 😊


**Failover events**:

![](/Pics/7%20-%20AA%20single%20link%20failover.png)
 
In case of failure, the “surviving” instance will start leveraging the Public IP of the “dead” instance to build IPSEC.
The IPSEC will come up.

During planned maintenance events, the Security Associations Migration takes place.
During unplanned events the rebuild of IPSEC tunnel may take up to some minutes.

**Pros**

No added value, same as Active/Standby 

**Cons**

As per Active/Standby scenario.
 


#### Single link enabled with BGP: DO NOT DO THIS!!

![](/Pics/8%20-%20AA%20single%20link%20BGP.png)
 
This represents the most dangerous condition for the reliability of your VPN connectivity with a VNG.

The scenario is similar to previous one, with the difference that we enable BGP on top of the connection.

Now, remember that in Active/Active scenario, any VM instance of the VNG has its own BGP IP.

You connect your local terminator to a single instance of the GW and proceed with the BGP peering.

You will soon notice that your VPN link will suffer of long downtimes, occurring any time planned maintenance is performed on the GW instances.

Let’s see in detail why you should never do this.


**Failover events:**

![](/Pics/9%20-%20AA%20single%20link%20BGP%20failover.png)
 
During failover, the connectivity of to the “surviving” instance is destined to fail.
For basically 2 reasons:

1.	Your local firewall is not programmed to build a tunnel with the public IP of the surviving instance, as per initial setup
2.	Even if the public IP of the “dead” instance was swapped to “surviving” one (as it happens for the static routing scenarios) - and the IPSEC re-established - the BGP peering between your firewall and the “surviving” instance will fail, since your firewall just knows the BGP IP of the primary instance it was connected to, so it will try to reach a no-longer-existing BGP IP over an active IPSEC tunnel
Due to that, every time Azure performed maintenance on VM instances, your connectivity would suffer downtime until the maintenance event is completed (and we’re talking potentially of 30mins or more).

**Pros**
None

**Cons**
Totally unreliable connectivity.

2.3	Double link with static routing

![](/Pics/10%20-%20AA%20double%20link%20static.png)

In this scenario we connect our local VPN terminator with both instances of the VNG, but we use Static routing (no BGP).
This scenario is allowing to reach potentially doubled throughput performances in normal conditions and avoiding complexity of BGP setup, but there are some things to keep in consideration for failover events.

Failover events:

![](/Pics/11%20-%20AA%20double%20link%20static%20failover.png)
 
In case of a link/gateway failure on Azure side, Azure will reprogram routes to automatically point to the healthy link even if the routing is static, but the same may not happen on the other side.
The remote device has 2 static routes in its route table.
If a link fails, the remote device may not remove the faulty route when IPSEC fails, so the system could still be sending traffic through a dead-end.
It’s possible that appliance supports link probing and could implement mechanisms to remove the faulty route, but this should be carefully validated before implementing a similar setup.

Pros
Potentially double link capacity and no BGP complexity

Cons
Remote VPN terminator could experience issues in case of failure of one of the IPSEC links to Azure, due to usage of static routes.


2.4	Double link with BGP

![](/Pics/12%20-%20AA%20double%20link%20BGP.png)
 
This represents the ideal and best connectivity model for an active-active VNG.
The Gateway is connected to remote side through 2x IPSEC links, and BGP is enabled on both links in order to unblock potential double capacity or to allow to use one link as primary and the other as secondary in case of failures.
Every GW instance is programmed with its own BGP-IP.
Remote VPN device must be configured properly in order to bind the peering of a specific remote BGP peer IP with the correct IPSEC interface as nexthop.

Failover events:

![](/Pics/13%20-%20AA%20double%20link%20BGP%20failover.png)
 
In case of failure of one Gateway instance, the IPSEC tunnel with the surviving instance is already in place.
No public IP swapping is performed.
After BGP timers, the traffic is automatically redirected to the surviving instance only.

Pros
Potentially double link capacity and all the benefits of BGP dynamic routing. 
Possibility to tune links as active or standby via BGP configuration (useful in case of remote stateful terminator)

Cons
None, a part for the complexity of double IPSEC link and BGP peering setup.


2.5	Double remote endpoint – 4x links – Static/BGP

![](/Pics/14%20-%20AA%20double%20link%202x%20remote.png)

The scenarios we described so far do not take into account possible failure of remote VPN endpoint.
With this last option – Active/Active instances on both ends – we reach the maximum level of reliability for our VPN solution to Azure.
We configure 2 local appliances for building 2x IPSEC tunnels with both the instances of the Azure VNG.
The scenario could technically be built in static-routing configuration on Azure side, but such configuration may bring to challenges for the routing from Onprem side.
BGP based setup is recommended.
With this configuration you can theoretically reach a 4x per-tunnel bandwidth if ECMP traffic balancing is supported on the Onprem side, or alternatively use link pairs in Active-Passive mode (i.e. Links 1-2 active +  Links 3-4 passive) by leveraging BGP attributes.
From the point of view of Azure, you will configure:

•	2x Local Network Gateways 
o	LNG0: Representing remote gateway 0
o	LNG1: Representing remote gateway 1
•	2x Connections
o	Connection0 to LNG0
o	Connection1 to LNG1
With this configuration, both VNG instances will start establishing connectivity attempts to the remote endpoints.

From the point of view of Onpremise side, of course you will have to configure 4x links.

Failover events:

![](/Pics/15%20-%20AA%20double%20link%202x%20remote%20failover.png)
 
In case of failure of one Gateway instance, the IPSEC tunnels with the surviving instance are already in place.
No public IP swapping is performed.
After BGP timers, the traffic is automatically redirected to the surviving instances only.

Since you have still 2x links available, you can still reach a theoretical value of 2x capacity if traffic is properly balanced

Pros
Potentially 4x link capacity and all the benefits of BGP dynamic routing. 
Possibility to tune links as active or standby via BGP configuration (useful in case of remote stateful terminator)

Cons
None, a part for the complexity of 4x IPSEC links and BGP peering setup.



Bandwidth considerations (per-tunnel / aggregate)
Often, users are confused about the capacity values for Azure VNG that we provide in our documentation.
It makes sense to try clarifying…
The throughput values provided in our documentation (https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways#gwsku) are defined as aggregate throughput:

![](/Pics/16%20-%20Aggregate%20table.png)

The definition of aggregate throughput is the maximum throughput achievable by the VNG instance as sum of all the connections configured, including P2S traffic.
This simply means that, if you select a VNG SKU with 5 or 10Gbps aggregate throughput, this doesn’t mean you will be able to reach such capacity over a single IPSEC link.
Some benchmark tests’ results regarding the performances of single links depending on the VNG’s SKU is available here:
https://learn.microsoft.com/en-us/azure/vpn-gateway/about-gateway-skus#performance

![](/Pics/17%20-%20pertunnel%20table.png)

As you can see, the performances of the single link are strictly correlated with the SKU type, but also with the kind of encryption algorithms used for the IPSEC tunnel, GCMAES256 being the one generally speaking offering best performances.
The maximum throughput you could expect today on a single IPSEC link of Azure VNG – using GCMAES256 – is around 2.3Gbps, with a GW offering a maximum of 10Gbps aggregate capacity.

Monitoring GW performances
The metrics available for the Azure VNG are available here:
https://learn.microsoft.com/en-us/azure/vpn-gateway/monitor-vpn-gateway-reference#metrics 
Many customers often ask: “Which of these metrics should we use to understand if our GW is well-dimensioned, or potentially experiencing issues?”
Today, the Azure VNG is not exposing info about underlying CPU usage, and this makes the monitoring phases a bit more difficult.
What I usually recommend in order to assess health status of a VNG from the point of view of performances, is to set alerts based on the following parameters:
•	Gateway S2S Bandwidth + Gateway P2S Bandwidth < aggregate SKU capacity
•	Tunnel Bandwidth < max benchmarked per-tunnel throughput depending on algorithms used [assessment to be replicated per every S2S tunnel configured]
•	Tunnel Peak PPS < max benchmarked per-tunnel pps rates depending on algorithms used
•	Tunnel Ingress/Egress Packet Drop Count < 1

Last but not least, it’s fundamental to implement a periodical assessments of the amount of VMs deployed in your environment: in fact, every VNG can support a limited number of VMs deployed in the HUB, or in spoke VNETs connected to the HUB and leveraging GW-transit.
The max amount of backend VMs is again reported here:
https://learn.microsoft.com/en-us/azure/vpn-gateway/about-gateway-skus#benchmark

![](/Pics/18%20-%20SupportedVMs.png)

If an excessive amount of VMs is polling routing info from the VNG, the VNG can suffer severe performance issues, resulting in recurrent failovers, connectivity flaps, drops, etc… 
Similar alerts, based on different metrics, can be used instead to assess connectivity, i.e.:

•	BGP Peer status
•	Tunnel MMSA count
•	Tunnel QMSA count


Considerations about flows symmetry:
Can I use stateful devices to terminate IPSEC links with Azure VNG?
Once more, the answer is different depending on the configuration we consider…
Scenario	Is stateful remote VPN endpoint supported?
A/P VNG 	YES
A/A VNG + single link – Static routing	YES
A/A VNG + single link - BGP	Unsupported scenario (see above)
A/A VNG + double link – Static routing	NO
A/A VNG + double link – BGP	YES only if leveraging BGP-path prepending to use links in active/standby approach
A/A VNG + 4x link – BGP (2x remote endpoints)	YES only if leveraging BGP-path prepending to use links in active/standby approach

The reason why stateful appliances are not always supportable in any VNG scenario is due to the fact that flow symmetry cannot be granted in Azure.
Let’s see a couple of examples:
FLOW SYMMETRY 

In these cases we analyze what happens in a scenarios of:
•	A/A VNG
•	2x links
•	BGP (Same routes advertised on both links, same AS Path length)  Active-active links
•	Single remote endpoint, stateful

TRAFFIC GENERATED FROM ONPREM TO AZURE :


GO PATH:
 
![](/Pics/19%20-%20symmetry1.png)

RETURN PATH:

![](/Pics/20%20-%20symmetry2.png)
 
When the traffic is originated from Onpremise, the return traffic generated by the VM in Azure does not grant any symmetry.
If packets have landed on IN0 in one direction (Onprem  Azure) it may be forwarded to IN1 in the opposite direction (Azure  Onprem).
Any further packet relevant to such flow/connection, from Azure VM to Onprem, will always be redirected through the same VNG instance, which – again – may not be the same where traffic landed initially.

TRAFFIC GENERATED FROM AZURE TO ONPREM :

GO PATH:

![](/Pics/21%20-%20simmetry3.png)
 
RETURN PATH:

![](/Pics/22%20-%20simmetry4.png)

One flow generated from Azure side, will always land on the same VNG instance, and that instance will always encapsulate it through the same logical tunnel.
The return path will depend on the logics applied the remote endpoint.
If the device routes back according with 5-tuple mapping, and ignoring the source link through which the ingress traffic has been received, the asymmetry could still happen

According with these considerations, it’s easy to understand why it’s commonly not recommended to use stateful appliances building IPSEC with Azure VNG in Active-Active mode, due to possible symmetry issues.
If a stateful device is mandatory, you can consider BGP and configure ASPath-prepending to make links passive and maintain the symmetry, scarifying a portion of the potential connection’s capacity.

CONCLUSIONS
Site-2-Site VPN solutions are still a very largely utilized instrument adopted today for connectivity with hybrid cloud environments.
The Azure Virtual Network Gateway represents the 1st party, integrated, recommended solution to build IPSEC tunnels to Azure infrastructure.
Azure VNG in Active/Active mode, represents the most reliable and best-performing setup available, with just some special care to be taken when assessing the usage of stateful remote terminators


