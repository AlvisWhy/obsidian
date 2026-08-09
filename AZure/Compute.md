Auto close
Disktype:
premium SSD(default)
Standard SSD
HDD

# 1. Azure VM 命名规则

以：

```
Standard_D8s_v5
```

为例：

| 部分       | 含义             |
| -------- | -------------- |
| Standard | 价格/服务级别        |
| D        | VM 系列（Series）  |
| 8        | vCPU 数量        |
| s        | 支持 Premium SSD |
| v5       | 第五代硬件平台        |

## 简单记忆 Azure VM 系列：

| 系列  | 定位    | Databricks用途   |
| --- | ----- | -------------- |
| D   | 通用    | 普通 ETL         |
| E   | 内存优化  | Spark 大数据（最常见） |
| F   | CPU优化 | 计算密集           |
| L   | 存储优化  | 高 IO           |
| M   | 超大内存  | 极端场景           |
| N   | GPU   | AI/ML          |

Availability
Fault Domain(a rack)
Update Domain
Availability Set(Across multi fault and update domain)
Availability Zone

Scale set
orchestration mode
flexible (AZure choose type for the set)
uniform
scale-in policy: newest vm/oldest vm

resource provider (register)

metadata service
using to detect azure operation to a vm from the vm. Like scale in or delete.

app service
should plan a machine type,  not serverless
Tiers: Free basic(can not autoscale), standard, premier, dedicated
Deployment slot
Charge can only be stopped after deletion

We should define the machine in azure but ==not GCP app engine standard==

AKS 
container registry

update :
enable with patch(only update small version)
enable with stable(Only update to stable version)
with rapid(update to latest)

###### **Azure Functions**

triggers
blob storage cosmos DB, event grid, dapr, event hubs, http, iot hub, queue storage, service bus, timer
/

bindings(connection between others)
mobile apps, send grid.... 

consumption plan
1.5 gb limit ram ==cold start==
pay per s/gb, times

premium plan choose a host
no cold start , no memory limit, vnet integration
vcpu duration(CPU * s)
mem duration(GB * s)

dedicated (Define a machine type, like app service)
run in app service
no addition pay
rely on the app service always on

durable functions( stateful function)

```
[Function("Function Name")]
[Output bindings]
public string Run([Trigger], ...other params){
	
}
```