1 default VM SA(Editor)

2 ADC (default credential)

3 $GOOGLE_APPLICATION_CREDENTIALS cloud point a key file(not secure)
auth-login result
Using metadata server to search for service account token

cloud run (SA, SA for each service for least privilege)
for onprem, no metadata server, should use SA key for service access
workload identity federation

Cloud Identity
Google Workplace, Sign in with Google, Google Cloud Identity

AD users ---> (Google Cloud Directory Sync) --> Google Identity

Policy
SA key creation prohibition/ VM region/ public ip forbidden
org policy admin role could(not included in the owner of the project), so project owner can not change the org policy.
Current node policy can overwrite the ancestor policy

PSA/ PSC
Cloud SQL has its own firewall, we should allow its connections IP to connect.
or use the cloud SQL proxy to connect(user should log in) , create the proxy tunnel, do not need the IP

PSA, a bridge, one to one, can not have many project talking to the CloudSQL
using VPC peering, cloudsql is in a different project, owned by google admin team(hidden). Peering from user VPC to CloudSQL,

PSC, make an endpoint, can be one to many, other project can also use a proxy to talk to the Cloud SQL gives an internal IP on the VPC, and create a proxy to the real IP. 

Can use a IP address to connect (Google API/ Googel Service/ Third-party services outside google) .

serverless vpc access
Help appengine/cloud run/ cloud function get into VPC

Access transparency: Record what data the Google engineer visit. Enable it in the IAM can see the log in the cloud logging

Assured folder


SAML federation/ GCDS
using original sign in method/ sync user info to google, no password sync, can work with password sync to save password to Google.

**Service Control**

Use service perimeters to set the boundary.
Like: aiplatform.googleapis.com, notebooks.googleapis.com
Another level of limitation, can limit on gcs, bq, sercret manager,
control access level like ip range/ region/ country/device/ operating system to these resources.

IAP Tunnel login
publicIP+Bastion host+VPN. --> Iam+iap(do not use public IP, use IAM to create network tunnel)

Bigquery jobuser: Can query
Bigquery dataViewer: Can view data
Bigquery data editor: Can modify data

Datastore/ Firestore:
Key known: batch get -> Return multiple entities