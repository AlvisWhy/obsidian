Vnet 
Can not span multiple regions
Scope to single subscription

Can be connected via peering

NSG network security group(firewall)
source, source port 
destination, destination port
protocol
default open RDP SSH to anyone

NSG can be attached to VM and subnet (Can not in App service, app service use constrain)

Use service tag allow access in NSG like using az terminal to ssh in the vm
service tags: internet, loadbalancer, azuremonitor,  virtue network.

AZure subnet: public/ private
GCP subnet: Can not separate private/public

Network peering

Secure connect
Jitconnect, Bastion, VPN, Jump box

Hub and Spoke
Hub VNET -> SPOKE: allow gateway transit, allow forward traffic(data pass the HUB firewall and get into the Spoke network)
SPOKE -> Hub VNET: Use remote gateway.

-------------------------------------------
Service endpoint (Still in public subnet, but optimize route)
connect vm to Azure managed service without allowing other ingress

Private Link(Close public ip, change to private)
(Like private service access in GCP)

VNET Integration(Used for service app to visit VNet)
Let App service **egress** to the vnet to visit private resources.
Should reserve IP in the subnet for the app service.

ASE (Isolated)
Deployed in the vnet
application service environment


App service use VNet integration to get into the Vnet and use private endpoint to visit SQL.

---------------------------------------------------------
Load Balancer
Basic / Standard
      / Redundant
      /Secured by default
   300/ 1000
   Free/Not
LB does not handle http, so used for internal resources
No protection

Application gateway
function as external
WAF in detection or prevention mode
Standard V2/ WAF type V2
Application Gateway in its own vnet

To use cookie for gateway, the host name of the app service must be set to keep same as the gateway, with custom domain setting

-----------------------------------------------
routing

Traffic manager/ Frontdoor

Traffic manager
priority(DR), weighted, performance(low lantency), geographic, multivalue, subnet.

Traffic for DNS, route user to specific ip. Application gateway do the url and request analysis before routiong.

Azure Frontdoor, global application gateway.
route based on url, SSL offloading, WAF
origin group(origin of backend)
priority/ weight