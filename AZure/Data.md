Azure SQL
Full backup every week
differential backup every 12-24h
transaction log backup every 5-10 min

Retention
regular: 7day

computer tiers
1 provisioned
2 serverless(autoscale), used when flow unstable

Elastic Pool(Predefine a certain cpu resource, the dbs will be arranged the cpu)
one server --- many database

Managed instance
Like on-prem, no active geo-replication
Can support built in function, can run CLR code
No AZ, No serverless, No autoscaling


Azure SQL
Single Database/ Elastic Pool
DTU/Vcores
Basic/Standard/Premium.   ///   general purpose/business critical/hyperscale
Compute
Serverless/provisioned

set private endpoint for DB

read scale out(same region, for replica, for high throughput, no auto failover, manually)
Geo-replication, have secondary in other regions(for DR and across region read, but async)
Create Auto-failover group, choose 2 DB in the group(DR, auto failover)

No load balance in AZure SQL

------------------------------
Cosmos DB

Multi-region 
API auto choose closest one.
Use partitions

5 consistency level, can be set by account level and request level:
strong, bounded staleness, session, consistent prefix, eventual

STRONG
region X always get same data as Y

Bounded Staleness(order guaranteed)
Y lag behind X by K versions or T time, keep order of version

Session(default)
consistent in one session

Consistent Prefix(order guaranteed)
No guarantee for lag size,others like bounded staleness


1RU = Read 1 item of 1kb in 1 s

3 pricing levels
Provision predefined num of RU/s
Auto scale: set maxmium RU/s
Serverless


AZure MySQL(No support hot DR)
Basic, General purpose, Memory optimized


Blob Storage
Redundancy
LRS: 3copies in the same zone
ZRS: 3copies in the region
GRS: 3copies in same zone and also in paired regions. can not read from secondary region unless failover.
GZRS: 3 copies in same region and also copied in paired region. can not read from secondary region unless failover.
RA-GRS: 3 copies in same zone and also copied in paired region, also the data in the paired region can be read
RA-GZRS: 3 copies in same region and also copied in paired region, also the data in the paired region can be read

retrieval time for cool and hot tiers are the same 

Allow Blob anonymous access

IP  confine+ Access key for secure
Access Key
Shared access signature(SAS Token)


