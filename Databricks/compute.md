Serverless compute: in databricks subscription


Classic compute: in customer subscription

All purpose cluster: created manually, persistent, shared, expensive
/
Job Cluster: created by jobs, automated workload, terminated once job ends, isolated

single/ multi - node

Access Mode: Dedicated(for single user)/ Standard(shared)/ No isolation shared

Auto Termination, Auto scale

Billing= DBU cost  + VM infra cost

# Spark Submit

spark-submit --class <main-class> --master --deployment <mode> <application-jar> [application-args]
master      yarn, local[3] (run locally with 3 threads)
deployment-mode   client or cluster
conf       spark.ececutor.memoryOverhead = 0.2
driver-cores    2
driver-memory   8G
num-executors     4
executor-cores
executor-memory

cluster mode
Driver is running in the ApplicationMaster（AM ）container on a worker node in the cluster
No dependency for client machine, better performance

client mode
start the driver on the client machine, no driver in the cluster.
For interactive mode, notebook ...


Spark give 2 interface to wokr
spark SQL and Dataframe API, Dataset API(for scala and Java)

