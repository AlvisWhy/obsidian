VPN 3GB/s, using for backup
Dedicated Interconnect 10GB/100GB/s, using for main connect
direct peering: For Using google services, not connecting the VPC

VPN gateway is regional, should deploy in each region

Firewall insight feature:
analyze the conflict, unused, duplicate firewall rules

Firewall rule log: which firewall rule hit
VPC flow log: src, dest, traffic bytes

NEG network endpoint group
user -> LB -> NEG -> backend

Google Edge Network
User -- Google Edge --  Google backbone -- Google private network
Decrease the connection in public network
For primer network, could quickly enter google backbone then to the target region, with low latency and global optimizer.
Standard tier will first in public network to get close to target region and finally enter google network.

The protection before LB is cloud armor, not firewall. So should set security policy in cloud armor

VPN is regional, if need to connect resource in different regions in a VPC, should set up VPN in each region.

429 Too Many requests