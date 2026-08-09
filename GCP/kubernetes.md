Use Fleet, manage multiple clusters
cluster namespace, we can run multiple container based applications on the same cluster. 
loader balance for kubernetes-> node port of the cluster -> target port of the container

**workload identity f**ederation allow us to assign a google SA to a workspace

Image pull policy
Always: For dev, every time start will pull new image
IfNotPresent: If local exists that version, will not pull. Used in prod, because image will not changed for the same version.
Never

Control dev->qa->prod strictly in GKE
Using binary authorization to require attestation in GKE policy

Single cluster entrance: Gateway
Multicluster entrance: Multicluster gateway
center control cluster: fleet
Fleet +--cluster A
     +--cluster B
     +-- ...
Fleet will create central management entrance to control policy and enable interconnect services
service-to-service: service mesh

Config controller:
Change google cloud resources into kubernetes object(like vm types)
Config sync:
control pod and others in the cluster

Anthos Ingress -> multi cluster gateway

Logs management:
kubectl logs pod-name --previous/
GCP cloud logging -> pod -> container