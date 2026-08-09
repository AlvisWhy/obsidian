Azure Container Instance
Container App service/  Container App service function
AKS, Azure Container Apps
Red Hat OpenShift

Container Apps
Environment
VNET: existing(final control, peering NSG), generated VNET(can not configure , can only reach Internet accessible endpoints)

single(same VNET, same log destination)/ multiple(not share same compute resource) environment

consumption only: auto scaling, pay for what you use, scale to 0(no payment)
workload profile: hardware isolation, predictable cost.

Container APP
where container run

Revision
manage version the container app
immutable, automatically provisioned, historical record, traffic splitting

Replica
AutoScaling
scaling limit: min/max per replica revision
http/tcp/custom triggers/ KEDA scaler-base

Change scaling ---- new revision

KEDA Kubernetes Event-Driven Autoscaler

Azure storage queue, Active MQ, Memory....
Scaler

----------------------------------------------------------------
Service connectors
App service/spring App/container app/azure functions connect to DB or other services

A connection link secret created and saved in the secret. A env variable refer to that secret for connection.

DAPR(a kind of service mesh)
service mesh for auth, communication, timeout retry, health check,
monitoring, service discovery, testing, load balancing.
circuit breaker. prevent cascade failure when a service fails.

when a service timeout, the service breaker will return error instead of calling it again and again.

Data Plane connect with service, and control plane control the data plane

In process(data mesh is in the service process)/ side car(more popular)
performance, / platform agnostic , code agnostic

Service Invocation
Dapr Components 
Defined on Environment scope.
Attached to revelent container apps, it is control by the scope. Onnly container app with app id defined can use the component.

Add the depr and app id in the request header to use depr to connect to other services

Depr runs in localhost:3500 (to call another container apps service must be in the same environment)

container has its residence policy. time out/ http retry/ circuit breaker.
But with DEPR, it has its own circuit breaker, can not be changed.

------------------------------------------------
Container job triggers
manual, schedule, event

CRON
minute, hour, day of month, month, day of week
```
  */5  * * *  Triggered every 5 minutes
  0 * 4 * * Every hour of 4th for each month
  30 5 * * 1 every monday 5:30am
```

