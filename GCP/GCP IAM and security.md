## 1. 身份认证与 Application Default Credentials（ADC）

### 1.1 Service Account（服务账号）

Service Account 是应用程序、VM、Cloud Run 等工作负载访问 Google Cloud API 时使用的身份，不是普通用户账号。

常见身份类型：

- **User Account**：真实用户身份，例如开发人员通过 Google 登录。
    
- **Service Account**：应用程序或服务使用的身份。
    
- **Google-managed Service Agent**：Google Cloud 服务自动创建和管理的服务账号，用于代表某项服务执行操作。
    

#### Compute Engine Default Service Account

- Compute Engine 默认服务账号通常为：  
    `PROJECT_NUMBER-compute@developer.gserviceaccount.com`
    
- VM 如果没有指定其他 Service Account，通常使用默认服务账号。
    
- 历史上，默认服务账号可能自动获得项目级 `Editor` 角色，但不能认为所有项目都如此。
    
- Google 建议为默认服务账号移除过度权限，改用自定义 Service Account 和最小权限角色。
    

**最佳实践：**

- 不要直接给 VM 或应用程序授予 Owner / Editor。
    
- 不同应用、服务、环境尽量使用独立 Service Account。
    
- 只授予完成任务所需的最小权限。
    
- 避免多个服务共享一个权限过大的 Service Account。
    

### 1.2 Application Default Credentials（ADC）

ADC 是 Google Cloud 客户端库使用的一套自动凭证查找机制。应用程序不需要硬编码凭证，而是由 ADC 自动寻找可用身份。

ADC 查找凭证的顺序：

1. `GOOGLE_APPLICATION_CREDENTIALS` 环境变量指定的凭证文件。
    
2. 本地通过 `gcloud auth application-default login` 创建的 ADC 凭证文件。
    
3. Google Cloud 环境中的 Metadata Server 提供的附加 Service Account 凭证。
    

注意：ADC 的查找顺序不代表安全性优先级。生产环境应根据运行环境选择合适的身份认证方式。

#### GOOGLE_APPLICATION_CREDENTIALS

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/credentials.json"
```

该变量可以指向：

- Service Account Key JSON 文件。
    
- Workload Identity Federation 的凭证配置文件。
    

**注意：**

Service Account Key 是长期有效的凭证，泄漏后可能被攻击者用于访问云资源。因此不建议将 Key 上传到 GitHub、Docker Image 或代码仓库，也不建议将它作为生产环境的首选认证方式。

#### gcloud auth login vs gcloud auth application-default login

|命令|作用|
|---|---|
|`gcloud auth login`|为 Google Cloud CLI 提供用户身份认证|
|`gcloud auth application-default login`|为本地应用程序和 Google Cloud Client Libraries 创建 ADC 凭证|

两者用途不同，执行 `gcloud auth login` 不代表本地 Python、Java 等应用程序已经获得 ADC 凭证。

### 1.3 Metadata Server

运行在 Google Cloud 环境中的工作负载可以通过 Metadata Server 获取附加 Service Account 的短期访问令牌。

典型流程：

```text
Application running on VM
        |
        v
Google Cloud Metadata Server
        |
        v
Attached Service Account
        |
        v
Short-lived Access Token
        |
        v
Google Cloud API
```

优势：

- 不需要在 VM 中保存 Service Account Key。
    
- 自动获取和刷新短期 Token。
    
- 可以通过修改 Service Account 的 IAM Role 控制访问权限。
    

常见适用场景：

- Compute Engine VM
    
- Google Kubernetes Engine（通过相应的 Workload Identity 配置）
    
- Cloud Run
    
- App Engine 等支持附加 Service Account 的运行环境
    

### 1.4 不同运行环境的认证方式

|运行环境|推荐认证方式|
|---|---|
|Compute Engine|Attached Service Account + Metadata Server|
|Cloud Run|User-managed Service Account + ADC|
|GKE|Workload Identity Federation for GKE|
|本地开发环境|`gcloud auth application-default login`|
|On-premises|Workload Identity Federation|
|其他云平台|Workload Identity Federation|

如果 On-premises 或其他云环境无法使用 Google Cloud Metadata Server，不代表必须使用 Service Account Key。

优先考虑 **Workload Identity Federation（WIF）**：

```text
External Workload
      |
      v
External Identity Provider
      |
      v
Workload Identity Federation
      |
      v
Google Cloud STS
      |
      v
Short-lived Google Credentials
      |
      v
Google Cloud Resources
```

WIF 可以让外部身份提供商（例如 AWS、Azure 或企业身份系统）提供身份凭证，再通过联邦机制换取 Google Cloud 的短期凭证，不需要创建长期 Service Account Key。

**面试问题：**

1. 什么是 ADC？ADC 如何查找凭证？
    
2. `gcloud auth login` 和 `gcloud auth application-default login` 有什么区别？
    
3. VM 如何在不保存 Service Account Key 的情况下访问 BigQuery？
    
4. 为什么生产环境不建议使用 Service Account Key？
    
5. On-premises 应用程序如何安全访问 Google Cloud？
    
6. Workload Identity Federation 与 Service Account Key 有什么区别？
    

---

## 2. Cloud Identity、Google Workspace 与身份联邦

### 2.1 Cloud Identity

Cloud Identity 是 Google 提供的企业身份和设备管理服务，用于集中管理组织用户、群组和身份认证。

相关产品：

- **Google Workspace**：企业办公套件，包括 Gmail、Google Drive、Calendar、Meet 等。
    
- **Cloud Identity**：企业身份管理、用户管理、群组管理及相关安全控制。
    
- **Sign in with Google**：允许用户使用 Google 账号登录第三方应用，是面向应用的身份认证能力。
    

Cloud Identity 和 Google Workspace 可以使用统一的 Google 身份体系，但两者提供的产品功能并不相同。

### 2.2 Google Cloud Directory Sync（GCDS）

GCDS 用于将企业现有 LDAP / Active Directory 中的用户和群组信息同步到 Google Identity。

典型架构：

```text
Active Directory / LDAP
          |
          v
         GCDS
          |
          v
Google Identity / Cloud Identity
          |
          v
Google Cloud Resources
```

主要作用：

- 同步用户信息。
    
- 同步群组信息。
    
- 同步组织结构相关信息。
    
- 帮助企业统一管理用户身份。
    

**注意：**

GCDS 本身不负责同步用户密码。企业可以结合 Password Sync 等机制实现密码同步，但这是独立的能力。

### 2.3 SAML Federation

SAML Federation 用于让企业已有的 Identity Provider（IdP）承担用户身份认证工作。

例如企业使用 Microsoft Active Directory Federation Services（AD FS）：

```text
User
 |
 v
Google Cloud Login
 |
 v
Redirect to Enterprise IdP
 |
 v
User Authentication
 |
 v
SAML Assertion
 |
 v
Google Identity
 |
 v
Google Cloud Access
```

主要优势：

- 用户继续使用企业原有登录方式。
    
- 企业可以集中管理用户认证。
    
- 支持 Single Sign-On（SSO）。
    
- 减少多个系统分别管理密码的需求。
    

### 2.4 三种机制的区别

|机制|主要作用|
|---|---|
|GCDS|同步用户和群组信息|
|SAML Federation|将用户认证交给外部 IdP|
|Password Sync|同步用户密码相关信息|
|Workload Identity Federation|让外部工作负载使用联邦身份访问 Google Cloud|

注意：Workforce Identity Federation 面向外部员工身份；Workload Identity Federation 面向应用程序、自动化任务等工作负载身份。

**面试问题：**

1. GCDS 的作用是什么？它会同步密码吗？
    
2. SAML Federation 如何实现企业 Single Sign-On？
    
3. GCDS 和 SAML Federation 有什么区别？
    
4. Workload Identity Federation 和 Workforce Identity Federation 有什么区别？
    
5. 企业已经使用 Active Directory，如何让员工访问 Google Cloud？
    

---

## 3. IAM Policy 与 Organization Policy

### 3.1 IAM Policy

IAM 控制谁（Who）可以对什么资源（Resource）执行什么操作（Permission）。

核心组成：

```text
Principal + Role + Resource
```

例如：

```text
Service Account
      |
      v
BigQuery Data Viewer
      |
      v
BigQuery Dataset
```

IAM Role 包含一组 Permissions。

常见角色：

|Role 类型|说明|
|---|---|
|Basic Roles|Owner、Editor、Viewer|
|Predefined Roles|Google 预定义的服务角色|
|Custom Roles|用户根据需要自定义权限组合|

生产环境应该优先使用 Predefined Roles 或 Custom Roles，而不是直接使用过于宽泛的 Basic Roles。

### 3.2 Organization Policy

Organization Policy 用于限制组织、Folder、Project 层级的资源配置和使用行为。

它解决的是：

**组织允许创建或配置什么资源，而不是某个用户能否访问某个资源。**

常见限制：

- 禁止创建 Service Account Key。
    
- 限制资源部署区域。
    
- 禁止 VM 使用 External IP。
    
- 限制某些资源或服务的创建。
    
- 限制某些 IAM Policy 的配置方式。
    

例如：

```text
Organization
     |
     | Organization Policy:
     | Deny Service Account Key Creation
     |
     v
Folder
     |
     v
Project
     |
     v
Resources
```

### 3.3 Organization Policy 的权限

管理 Organization Policy 通常需要：

`roles/orgpolicy.policyAdmin`

拥有 Project Owner 并不意味着自动拥有 Organization Policy 管理权限。

因此：

- Project Owner 可以管理其项目范围内的很多资源和 IAM 配置。
    
- 但不能仅凭 Project Owner 角色修改组织级 Organization Policy。
    
- 需要具备相应的 Organization Policy 管理权限。
    

### 3.4 Organization Policy 继承

Google Cloud 资源层级：

```text
Organization
     |
     v
Folder
     |
     v
Project
     |
     v
Resource
```

默认情况下，子资源继承父级 Organization Policy。

但子级可以根据约束类型和策略配置，对父级策略进行继承、合并或覆盖。

**注意：**

不能简单地认为子级 Policy 永远覆盖父级 Policy。具体行为取决于 Constraint 类型、继承配置以及是否允许 Override。

此外，Organization Policy 通常不是自动追溯执行的：新策略可能会阻止后续操作，但不一定自动改变已经存在的资源状态。

### 3.5 IAM Policy vs Organization Policy

|对比|IAM Policy|Organization Policy|
|---|---|---|
|核心问题|谁可以做什么|哪些资源配置或行为被允许|
|控制对象|Principal、Role、Permission|资源配置和使用约束|
|典型用途|授权用户查询 BigQuery|禁止创建外部 IP|
|管理层级|Organization、Folder、Project、资源等|Organization、Folder、Project|
|典型角色|Project IAM Admin 等|Organization Policy Admin|

**面试问题：**

1. IAM Policy 和 Organization Policy 有什么区别？
    
2. Project Owner 能否修改 Organization Policy？为什么？
    
3. 如何禁止组织内所有项目创建 Service Account Key？
    
4. Organization Policy 如何在 Organization、Folder、Project 之间继承？
    
5. 子级 Project 能否覆盖 Organization 层级的限制？
    
6. 如何限制 Google Cloud 资源只能部署在指定区域？
    

---

## 4. Cloud SQL 网络连接：Public IP、PSA、PSC 与 Auth Proxy

### 4.1 Public IP

Cloud SQL 可以通过 Public IP 接收外部网络连接。

如果应用程序通过 Public IP 直接连接，需要考虑：

- Cloud SQL Authorized Networks。
    
- 防火墙和网络访问控制。
    
- 数据库身份认证。
    
- TLS 加密。
    

Authorized Networks 用于限制哪些客户端 IP 可以直接连接 Cloud SQL 的 Public IP。

### 4.2 Cloud SQL Auth Proxy

Cloud SQL Auth Proxy 提供安全的 Cloud SQL 连接方式。

主要作用：

- 使用 IAM 身份验证应用程序是否有权连接实例。
    
- 自动处理连接所需的 TLS。
    
- 建立本地或应用程序侧的连接代理。
    
- 可以避免直接管理 Cloud SQL 的 Public IP Authorized Networks。
    

连接流程：

```text
Application
    |
    v
Cloud SQL Auth Proxy
    |
    | IAM Authentication
    | Secure Connection
    v
Cloud SQL Instance
```

注意：

1. Proxy 不等于数据库账号认证。
    
2. 如果没有启用 IAM Database Authentication，仍然需要数据库用户名和密码。
    
3. 使用 Private IP 时，运行 Proxy 的机器仍然需要具备到 Cloud SQL 私有网络的连接能力。
    
4. Proxy 不能替代 VPC 网络连通性配置。
    

### 4.3 Private Services Access（PSA）

PSA 是一种基于 VPC Network Peering 的私有服务连接方式。

Cloud SQL 使用 Google 管理的 Service Producer Network，用户 VPC 通过 VPC Peering 与该网络建立连接。

```text
Customer VPC
     |
     | VPC Peering
     |
     v
Google Service Producer Network
     |
     v
Cloud SQL
```

特点：

- Cloud SQL 使用 Private IP。
    
- 基于 VPC Peering。
    
- 一个 Cloud SQL PSA 配置通常关联一个 Consumer VPC。
    
- 如果多个项目使用 Shared VPC，它们可以通过同一个 VPC 访问。
    
- 如果需要多个不同 VPC 网络访问同一个 Cloud SQL 实例，通常需要考虑 PSC 或额外网络架构。
    

### 4.4 Private Service Connect（PSC）

PSC 提供更灵活的私有服务连接方式。

它允许 Consumer VPC 通过内部 IP Endpoint 访问 Service Producer 提供的服务。

```text
Consumer VPC A
      |
      v
PSC Endpoint A
      |
      |
      v
Cloud SQL Service Attachment
      |
      ^
      |
PSC Endpoint B
      ^
      |
Consumer VPC B
```

特点：

- 支持多个不同 VPC 网络访问同一个 Cloud SQL 实例。
    
- 每个 Consumer VPC 通常有自己的 PSC Endpoint。
    
- Endpoint 使用 Consumer VPC 内部 IP。
    
- 可以跨项目、跨组织以及支持的混合云网络访问。
    
- Cloud SQL 需要配置允许连接的 Consumer Projects。
    

### 4.5 PSA vs PSC

|对比|PSA|PSC|
|---|---|---|
|网络技术|VPC Network Peering|Private Service Connect|
|多个 Consumer VPC|不直接支持，需要额外网络配置|支持|
|Endpoint|通过 Peering 访问|每个 Consumer VPC 建立内部 IP Endpoint|
|网络耦合|相对较强|相对较低|
|使用场景|单一 VPC 访问 Cloud SQL|多个项目或 VPC 访问 Cloud SQL|

对于新建 Cloud SQL 私有连接架构，PSC 通常是更灵活的选择。

### 4.6 PSA / PSC 与 Cloud SQL Auth Proxy 的关系

它们解决的问题不同：

- PSA / PSC：解决网络层如何访问 Cloud SQL。
    
- Cloud SQL Auth Proxy：解决连接认证和安全连接建立。
    

两者可以结合使用，而不是互相替代。

**面试问题：**

1. Cloud SQL Public IP 和 Private IP 有什么区别？
    
2. Cloud SQL Auth Proxy 解决什么问题？是否还需要数据库密码？
    
3. PSA 和 PSC 有什么区别？
    
4. 为什么多个项目需要访问同一个 Cloud SQL 时可以考虑 PSC？
    
5. 使用 Private IP 的 Cloud SQL，是否可以直接从没有 VPC 连通性的本地机器访问？
    
6. PSC Endpoint 和 Cloud SQL 的 Private IP 是什么关系？
    

---

## 5. Serverless VPC Access 与 Direct VPC Egress

### 5.1 Serverless VPC Access

Serverless 服务通常运行在 Google 管理的基础设施中，不一定直接位于用户的 VPC 网络内部。

Serverless VPC Access Connector 可以让 Serverless 应用访问 VPC 内部资源。

支持的典型场景：

- Cloud Run
    
- Cloud Run Functions
    
- App Engine Standard 等支持的环境
    

架构：

```text
Cloud Run
    |
    v
Serverless VPC Access Connector
    |
    v
Customer VPC
    |
    +------ Compute Engine
    |
    +------ Memorystore
    |
    +------ Private IP Cloud SQL
```

适用场景：

- 访问 VPC 内部 VM。
    
- 访问 Memorystore。
    
- 访问具有私有 IP 的数据库。
    
- 访问通过 VPN / Interconnect 接入的内部网络。
    

### 5.2 Direct VPC Egress

对于 Cloud Run 等支持的产品，Google 现在推荐优先考虑 Direct VPC Egress。

它可以让 Serverless 工作负载直接向 VPC 发送出站流量，不需要创建 Serverless VPC Access Connector。

|对比|Serverless VPC Access|Direct VPC Egress|
|---|---|---|
|是否需要 Connector|需要|不需要|
|连接方式|通过 Connector|直接连接 VPC|
|典型场景|兼容旧架构或需要 Connector 的环境|支持 Direct VPC 的 Cloud Run 等服务|

注意：Serverless VPC Access 主要解决 Serverless 到 VPC 的网络访问问题，不负责授予数据库或其他云服务的 IAM 权限。

**面试问题：**

1. Cloud Run 如何访问 VPC 内部的 Memorystore？
    
2. Serverless VPC Access 的作用是什么？
    
3. Direct VPC Egress 和 Serverless VPC Access 有什么区别？
    
4. Serverless VPC Access 是否会自动授予 Cloud SQL 的访问权限？
    

---

## 6. Access Transparency、Access Approval 与 Assured Workloads

### 6.1 Access Transparency

Access Transparency 用于记录 Google 员工访问客户数据的行为。

日志可能包含：

- 访问时间。
    
- 被访问的资源。
    
- 执行的操作。
    
- Google 员工相关信息。
    
- 访问的业务理由。
    

日志可以在 Cloud Logging 中查看。

主要用途：

- 审计 Google Cloud 服务提供方人员对客户数据的访问。
    
- 满足企业合规要求。
    
- 增加云平台管理操作的透明度。
    

### 6.2 Access Approval

Access Approval 与 Access Transparency 不同。

- Access Transparency：记录 Google 员工访问客户数据的行为。
    
- Access Approval：允许客户对符合条件的 Google 员工访问请求进行批准或拒绝。
    

两者结合可以实现：

```text
Google Employee Requests Access
             |
             v
       Access Approval
             |
             v
      Customer Approval
             |
             v
       Access Granted
             |
             v
      Access Transparency
             |
             v
       Access Audit Logs
```

### 6.3 Assured Workloads

Assured Workloads 用于帮助企业在 Google Cloud 上构建满足特定合规和监管要求的工作负载环境。

例如：

- 数据驻留要求。
    
- 特定地区的数据处理限制。
    
- 特定监管框架的合规要求。
    
- 人员访问和管理控制要求。
    

Assured Workloads 通常通过专门的 Folder 组织受控工作负载，并应用相应的合规控制。

注意：Assured Workloads 不等于单纯的 IAM Role，也不是只要创建 Folder 就自动满足所有合规要求。

**面试问题：**

1. Access Transparency 和 Cloud Audit Logs 有什么区别？
    
2. Access Transparency 和 Access Approval 有什么区别？
    
3. 如何审计 Google 员工对企业客户数据的访问？
    
4. Assured Workloads 主要解决什么问题？
    
5. 企业有数据驻留和监管合规要求时，可以使用什么 Google Cloud 服务？
    

---

## 7. VPC Service Controls（VPC SC）

### 7.1 核心概念

VPC Service Controls 用于为支持的 Google Cloud 服务建立安全边界，降低数据泄漏和数据外传（Data Exfiltration）的风险。

它不是传统 VPC Firewall，而是围绕 Google Cloud API 和受保护服务建立服务访问边界。

例如：

- Cloud Storage
    
- BigQuery
    
- Secret Manager
    
- Vertex AI 等支持的服务
    

### 7.2 Service Perimeter

Service Perimeter 定义一个受保护的服务边界。

```text
             Service Perimeter
    +--------------------------------+
    |                                |
    |  BigQuery                      |
    |  Cloud Storage                 |
    |  Secret Manager                |
    |                                |
    +--------------------------------+
                  |
                  v
         Restricted API Access
```

可以通过 Service Perimeter 限制：

- 哪些项目可以访问受保护资源。
    
- 哪些 Google Cloud API 可以被访问。
    
- 哪些跨边界访问可以被允许。
    
- 哪些外部访问需要满足特定条件。
    

### 7.3 Access Level

Access Level 用于描述访问请求必须满足的上下文条件。

例如：

- 来源 IP 地址范围。
    
- 设备是否满足安全要求。
    
- 用户设备的操作系统或设备属性。
    
- 地理位置等条件（取决于可用的访问上下文和配置）。
    

可以把 Access Level 与 Service Perimeter 结合，控制哪些访问请求能够进入受保护边界。

### 7.4 VPC SC vs IAM

|对比|IAM|VPC Service Controls|
|---|---|---|
|主要作用|身份和权限控制|服务边界和数据外传防护|
|关注点|谁可以执行什么操作|请求是否可以跨越服务边界|
|控制对象|Principal、Permission、Resource|支持的 Google Cloud API 和服务|
|是否可以替代对方|不能|不能|

例如：

一个 Service Account 即使具有 BigQuery Data Viewer 权限，如果请求违反 VPC Service Controls 的边界规则，仍可能无法访问 BigQuery 数据。

**面试问题：**

1. VPC Service Controls 解决什么安全问题？
    
2. VPC SC 和 IAM 有什么区别？
    
3. 什么是 Service Perimeter？
    
4. Access Level 可以限制哪些访问条件？
    
5. 一个 Service Account 拥有 BigQuery 权限，但仍然无法查询数据，可能是什么原因？
    
6. VPC SC 能否替代 VPC Firewall？
    

---

## 8. Identity-Aware Proxy（IAP）与安全远程访问

### 8.1 传统访问方式

传统方式中，管理员可能通过：

```text
Internet
   |
   v
Public IP
   |
   v
Bastion Host
   |
   v
Private VM
```

或者通过 VPN 进入企业网络，再访问内部 VM。

这些方式可能需要管理：

- Public IP。
    
- Bastion Host。
    
- VPN。
    
- SSH Key。
    
- 防火墙规则。
    

### 8.2 IAP TCP Forwarding

IAP TCP Forwarding 可以通过 IAM 身份和加密隧道访问 VM，不需要为目标 VM 配置 External IP。

架构：

```text
Administrator
      |
      v
Google Cloud IAP
      |
      | IAM Authorization
      |
      v
Encrypted TCP Tunnel
      |
      v
Private VM
```

主要用途：

- SSH 到 VM。
    
- RDP 到 VM。
    
- 访问其他 TCP 服务。
    
- 对访问用户和目标 VM 进行细粒度授权。
    

通常需要：

- 配置 IAP 相关 IAM 权限。
    
- 配置 VM 的网络防火墙，允许 IAP TCP Forwarding 使用的来源范围。
    
- 目标 VM 不需要 External IP。
    

**注意：**

IAP 并不意味着所有网络防火墙配置都不需要，也不意味着所有情况下都可以完全替代 VPN。它主要提供基于身份的安全访问通道。

### 8.3 IAP vs VPN vs Bastion Host

|方案|核心机制|典型用途|
|---|---|---|
|VPN|建立网络层连接|让用户或网络接入 VPC|
|Bastion Host|通过跳板机访问内部机器|传统运维管理|
|IAP TCP Forwarding|IAM + 加密 TCP Tunnel|无 Public IP 的 VM 管理访问|

**面试问题：**

1. 如何在不配置 VM Public IP 的情况下 SSH 访问 Compute Engine？
    
2. IAP TCP Forwarding 如何工作？
    
3. IAP 和 VPN 有什么区别？
    
4. 使用 IAP 是否还需要配置 Firewall？
    
5. 为什么 IAP 可以降低 Bastion Host 的使用需求？
    

---

## 9. BigQuery IAM Roles

BigQuery 的权限需要区分 Job 权限和 Data 权限。

### 9.1 BigQuery Job User

`roles/bigquery.jobUser`

主要权限：

- 创建和运行 BigQuery Jobs。
    
- 执行 Query。
    
- 执行其他需要创建 Job 的操作。
    

通常授予在 Project 层级。

### 9.2 BigQuery Data Viewer

`roles/bigquery.dataViewer`

主要权限：

- 查看 Dataset 和 Table 元数据。
    
- 查询和读取表数据。
    
- 导出数据。
    
- 查看符合授权范围的表和 View。
    

通常授予在 Dataset、Table 或 View 层级。

### 9.3 BigQuery Data Editor

`roles/bigquery.dataEditor`

主要权限：

- 创建和修改 Table。
    
- 修改表数据。
    
- 删除表。
    
- 更新表结构和元数据。
    
- 创建和恢复 Snapshot 等。
    

通常授予在 Dataset、Table 或 View 层级。

### 9.4 查询 BigQuery 所需的组合权限

一个用户需要查询其他 Dataset 中的表，通常需要：

```text
Project A:
    BigQuery Job User
          |
          v
    Execute Query Job

Dataset B:
    BigQuery Data Viewer
          |
          v
    Read Table Data
```

注意：

- Job User 不代表可以读取 Dataset 数据。
    
- Data Viewer 不代表可以在 Project 中创建 Query Job。
    
- 两者经常需要组合使用。
    
- 具体权限取决于角色授予的资源层级和查询涉及的资源。
    

### 9.5 常见角色对比

|Role|主要作用|
|---|---|
|BigQuery Job User|创建和执行 Jobs|
|BigQuery Data Viewer|读取数据|
|BigQuery Data Editor|创建、修改和删除数据资源|
|BigQuery Data Owner|管理数据资源及其权限，权限比 Data Editor 更广|

**面试问题：**

1. BigQuery Job User 和 Data Viewer 有什么区别？
    
2. 为什么用户拥有 Data Viewer 仍然无法运行 Query？
    
3. 如何让用户只能查询 Dataset A，而不能读取 Dataset B？
    
4. Data Editor 和 Data Owner 有什么区别？
    
5. BigQuery IAM 权限应该授予 Project 还是 Dataset？
    

---

## 10. Firestore / Datastore Batch Get

当已经知道 Entity 或 Document 的 Key 时，可以直接通过 Key 批量读取数据，而不必执行查询扫描。

### 10.1 Batch Get

例如：

```text
Known Keys:
    Key A
    Key B
    Key C
       |
       v
Batch Get
       |
       v
Return Multiple Entities
```

适用场景：

- 已经知道多个 Entity Key。
    
- 需要批量读取多个实体。
    
- 希望避免不必要的 Query 操作。
    

对于 Datastore，常见概念是 Lookup / Batch Lookup；对于 Firestore，常见 API 是 BatchGetDocuments。

### 10.2 Batch Get vs Query

|对比|Batch Get|Query|
|---|---|---|
|输入|已知 Key|查询条件|
|目标|读取指定实体|查找满足条件的实体|
|使用场景|已经知道要读取哪些记录|不知道 Key，需要筛选记录|
|典型操作|Get by Key|Filter / Order / Query|

**面试问题：**

1. 已经知道多个 Firestore Document ID，如何批量读取？
    
2. Batch Get 和 Query 有什么区别？
    
3. 为什么已知 Key 时通常不需要执行 Query？
    
4. Datastore 和 Firestore 的批量读取 API 有什么区别？
    

---

## 11. JAR Signature 与数字签名

### 11.1 JAR 文件签名

Java JAR 文件可以通过数字签名机制验证文件是否被篡改，以及签名者身份是否可信。

常见签名文件：

```text
META-INF/
    MANIFEST.MF
    MYKEY.SF
    MYKEY.RSA
```

其中：

|文件|作用|
|---|---|
|MANIFEST.MF|保存 JAR 文件条目及其摘要信息|
|MYKEY.SF|保存 Manifest 或其各个 Section 的摘要|
|MYKEY.RSA|保存数字签名以及签名证书等信息|

### 11.2 签名和验证流程

签名阶段：

```text
JAR Class Files
      |
      v
Calculate SHA Digest
      |
      v
MANIFEST.MF
      |
      v
Generate .SF File
      |
      v
Private Key Signs .SF
      |
      v
.RSA Signature Block
```

验证阶段：

```text
.RSA Signature Block
      |
      v
Public Key from Certificate
      |
      v
Verify Signature of .SF
      |
      v
Verify Digest in MANIFEST.MF
      |
      v
Recalculate Class File Digest
      |
      v
Compare Digests
```

如果验证成功，说明：

- `.SF` 文件的签名与对应公钥匹配。
    
- JAR 文件内容的摘要与 Manifest 中保存的摘要一致。
    

如果文件内容发生变化，重新计算的 Digest 就可能与原来的 Digest 不一致。

### 11.3 数字签名 vs 加密

需要纠正一个容易混淆的概念：

数字签名不是简单地使用私钥加密内容，再用公钥解密内容。

|操作|使用的密钥|作用|
|---|---|---|
|数字签名|私钥|证明签名由对应私钥持有者生成|
|验证签名|公钥|验证签名是否有效|
|非对称加密|公钥|加密数据|
|非对称解密|私钥|解密数据|

数字签名主要保证：

- Integrity：完整性。
    
- Authenticity：真实性。
    
- Non-repudiation：不可否认性（依赖密钥管理、身份绑定及法律环境）。
    

**面试问题：**

1. JAR Signature 如何检测文件是否被篡改？
    
2. `.SF`、`.RSA` 和 `MANIFEST.MF` 分别有什么作用？
    
3. 为什么数字签名使用私钥签名、公钥验证？
    
4. Hashing、Encryption 和 Digital Signature 有什么区别？
    
5. 如果 JAR 中一个 Class 文件被修改，签名验证会发生什么？
    

---

## 12. PCI DSS 与数据安全合规

### 12.1 PCI DSS

PCI DSS（Payment Card Industry Data Security Standard）是支付卡行业数据安全标准。

适用于存储、处理或传输支付卡持卡人数据的相关组织和系统。

主要关注：

- 保护持卡人数据。
    
- 限制数据访问权限。
    
- 网络安全。
    
- 加密和密钥管理。
    
- 安全日志与监控。
    
- 漏洞管理。
    
- 定期安全测试。
    

对于 Data Engineer，常见相关工作包括：

- 防止未经授权访问支付数据。
    
- 对敏感数据进行加密。
    
- 使用 IAM 最小权限。
    
- 使用审计日志记录数据访问。
    
- 对数据进行脱敏或 Tokenization。
    
- 控制数据在不同环境之间的流转。
    

**面试问题：**

1. PCI DSS 是什么？主要保护什么数据？
    
2. 如何使用 IAM 限制支付数据的访问？
    
3. 如何在数据仓库中保护信用卡信息？
    
4. 如何审计敏感数据的访问行为？
    
5. Encryption 和 Tokenization 有什么区别？
    

---

## 13. Artifact Analysis 与软件供应链安全

### 13.1 Artifact Analysis

Artifact Analysis 用于分析软件制品中的安全漏洞和相关安全信息。

常见使用场景：

- Container Image Vulnerability Scanning。
    
- 检测容器镜像中的已知漏洞。
    
- 分析软件包依赖中的漏洞。
    
- 配合 Artifact Registry 管理容器制品。
    

典型流程：

```text
Developer
    |
    v
Build Container Image
    |
    v
Artifact Registry
    |
    v
Artifact Analysis
    |
    v
Vulnerability Scanning
    |
    v
Review Security Findings
```

需要注意：

- 漏洞扫描用于发现已知漏洞。
    
- 扫描结果不代表镜像没有所有安全问题。
    
- Artifact Analysis 发现漏洞不代表自动完成漏洞修复。
    
- 还需要结合 CI/CD Pipeline 设置安全门禁。
    

### 13.2 面试问题

1. Artifact Analysis 的作用是什么？
    
2. 如何在 CI/CD Pipeline 中加入 Container Image Vulnerability Scanning？
    
3. 为什么容器镜像需要定期扫描？
    
4. Artifact Registry 和 Artifact Analysis 有什么区别？
    
5. 扫描发现 Critical Vulnerability 后，应该如何处理？
    

---

## 14. Migration Center

Migration Center 是 Google Cloud 提供的迁移评估工具，用于帮助企业评估从本地数据中心或其他云平台迁移到 Google Cloud 的成本和技术方案。

主要功能：

- 收集现有基础设施信息。
    
- 评估 VM、Server、数据库等资源。
    
- 估算迁移后的云资源成本。
    
- 支持迁移规划和成本分析。
    
- 帮助制定迁移策略。
    

典型流程：

```text
On-premises Infrastructure
          |
          v
   Collect Inventory
          |
          v
     Migration Center
          |
          v
   Cost Estimation
          |
          v
   Migration Planning
          |
          v
      Google Cloud
```

**面试问题：**

1. Migration Center 解决什么问题？
    
2. 企业迁移到 Google Cloud 前，如何估算基础设施成本？
    
3. Migration Center 与 Cloud Monitoring 有什么区别？
    
4. 迁移前需要收集哪些基础设施信息？
    

---

## 15. Apigee API Management

### 15.1 Apigee

Apigee 是 Google Cloud 的 API Management 平台，用于管理、保护、监控和分析 API。

它可以作为 API Facade，将后端服务封装在统一 API 接口之后。

```text
Client Application
       |
       v
     Apigee
       |
       +------ Authentication
       |
       +------ Authorization
       |
       +------ Rate Limiting
       |
       +------ Threat Protection
       |
       +------ Caching
       |
       v
   Backend Services
```

主要优势：

- 客户端不需要直接访问后端服务。
    
- 统一管理 API 安全策略。
    
- 隐藏后端服务实现。
    
- 支持 API Versioning。
    
- 方便监控和管理 API 流量。
    

### 15.2 API Key Validation

API Key 用于识别调用 API 的应用或客户端。

Apigee 可以通过 API Key 验证调用方身份或应用信息。

注意：API Key 不等同于完整的用户身份认证，也不能单独替代 OAuth 2.0 等机制。

### 15.3 OAuth 2.0

OAuth 2.0 是授权框架，允许客户端在获得授权后访问受保护资源。

Apigee 可以通过 OAuth 2.0 Policy 管理 Token 的生成、验证和相关授权流程。

### 15.4 Quota Policy

Quota Policy 用于限制一段时间内允许的 API 请求数量。

例如：

```text
Application A:
    Maximum 10,000 requests / day
```

超过 Quota 后，请求可能被拒绝。

适用场景：

- 按应用限制调用次数。
    
- 按开发者或客户限制 API 使用量。
    
- 控制 API 资源消耗。
    

### 15.5 SpikeArrest Policy

SpikeArrest 用于限制突发请求流量，平滑请求速率，避免后端服务被瞬时流量冲击。

例如：

```text
Incoming Requests:
1000 requests in a short burst
          |
          v
     SpikeArrest
          |
          v
Control Request Rate
          |
          v
    Backend Service
```

### 15.6 Quota vs SpikeArrest

|对比|Quota|SpikeArrest|
|---|---|---|
|主要目的|限制一段时间内的总请求量|平滑突发流量|
|控制重点|总量|请求速率|
|典型场景|每天最多 10,000 次请求|限制短时间内的请求速率|

### 15.7 Rate Limiting

Rate Limiting 是请求速率控制的统称，可以通过不同策略实现。

例如：

- 每秒最多 100 次请求。
    
- 每分钟最多 1,000 次请求。
    
- 每天最多 10,000 次请求。
    

Quota 和 SpikeArrest 都属于 API 流量控制的重要策略，但作用侧重点不同。

### 15.8 Threat Protection

Apigee 可以使用 Threat Protection Policies 对 API 请求进行安全检查。

例如：

- JSON Threat Protection。
    
- XML Threat Protection。
    
- 检查异常或恶意请求内容。
    
- 限制不符合预期格式的请求。
    

### 15.9 API Facade

API Facade 在客户端和后端服务之间增加一层统一 API 接口。

例如：

```text
Client
  |
  v
Apigee API: /v1/orders
  |
  v
Backend Service
```

即使后端服务从 V1 迁移到 V2，也可以通过 Apigee 维持对客户端的接口兼容，降低客户端修改成本。

### 15.10 Caching

Apigee 可以缓存 API 响应，减少重复请求对后端服务的压力。

适用场景：

- 查询频率高。
    
- 数据变化不频繁。
    
- 可以接受一定程度缓存延迟的 API。
    

**面试问题：**

1. Apigee 解决什么问题？
    
2. API Key 和 OAuth 2.0 有什么区别？
    
3. Quota Policy 和 SpikeArrest 有什么区别？
    
4. 如何防止 API 被大量突发请求压垮？
    
5. API Facade 有什么优势？
    
6. Apigee Caching 可以改善什么问题？
    
7. 如何使用 Apigee 对 API 进行 Threat Protection？
    

---

## 16. Google Cloud 成本优化：SUD 与 CUD

这部分属于 Cloud Cost Optimization，而不是 IAM Security，但属于 Google Cloud Engineer 常见考点。

### 16.1 Sustained Use Discounts（SUD）

SUD 是持续使用折扣。

主要特点：

- 适用于符合条件的资源和服务。
    
- 根据资源在计费周期内的持续使用情况提供折扣。
    
- 通常不需要提前承诺。
    
- 适合使用时间较长，但未来使用量不确定的工作负载。
    

### 16.2 Committed Use Discounts（CUD）

CUD 是承诺使用折扣。

主要特点：

- 用户承诺在一定期限内使用指定资源或达到一定消费金额。
    
- 根据承诺类型获得折扣。
    
- 通常有 1 年或 3 年等承诺期限。
    
- 适合长期稳定、使用量可预测的工作负载。
    

### 16.3 SUD vs CUD

|对比|SUD|CUD|
|---|---|---|
|是否需要承诺|不需要|需要|
|折扣依据|持续使用情况|预先承诺的资源或消费|
|灵活性|较高|相对较低|
|适用场景|使用时间长但未来不确定|长期稳定、可预测的工作负载|

注意：SUD 和 CUD 的适用产品、折扣计算方式和兼容规则并不完全相同，不能假设所有 Google Cloud 服务都支持两者。

**面试问题：**

1. SUD 和 CUD 有什么区别？
    
2. 对于长期稳定运行的 Compute Engine，如何优化成本？
    
3. 如果工作负载未来使用量不确定，应该优先考虑什么？
    
4. CUD 为什么可能带来成本风险？
    

---

## 17. SRE：SLO、SLA、SLI 与 Error Budget

### 17.1 SLI（Service Level Indicator）

SLI 是服务质量的实际测量指标。

例如：

- Availability：服务可用性。
    
- Latency：请求延迟。
    
- Error Rate：错误率。
    
- Throughput：吞吐量。
    

例如：

```text
SLI = Successful Requests / Total Requests
```

### 17.2 SLO（Service Level Objective）

SLO 是服务质量目标。

例如：

```text
Monthly Availability SLO = 99.9%
```

代表希望在指定时间窗口内，服务可用性达到 99.9%。

### 17.3 SLA（Service Level Agreement）

SLA 是服务提供方与客户之间的服务等级协议，通常包含服务承诺及未达到目标时的补偿条款。

|概念|含义|
|---|---|
|SLI|实际测量的服务质量指标|
|SLO|内部或对外设定的服务质量目标|
|SLA|面向客户的服务等级协议|

### 17.4 Error Budget

Error Budget 是在指定时间窗口内允许服务出现的不可靠程度。

计算方式：

```text
Error Budget = 1 - SLO
```

例如：

```text
SLO = 99.9%

Error Budget = 0.1%
```

如果一个月有 30 天：

```text
30 days × 24 hours × 60 minutes
= 43,200 minutes
```

允许的不可用时间：

```text
43,200 × 0.1%
= 43.2 minutes
```

也就是说，在 30 天的统计窗口内，最多允许约 43.2 分钟的不可靠时间。

### 17.5 Error Budget 的用途

Error Budget 用于平衡：

- 服务稳定性。
    
- 新功能发布速度。
    
- 软件迭代速度。
    
- 系统可靠性投入。
    

当 Error Budget 消耗过快时：

- 增加告警和监控。
    
- 调查故障原因。
    
- 减少高风险发布。
    
- 优先进行稳定性改进。
    

当 Error Budget 充足时，可以适当增加新功能发布和系统迭代。

### 17.6 Error Budget Burn Rate

Burn Rate 表示 Error Budget 消耗的速度。

例如：

- Burn Rate = 1：按照预期速度消耗 Error Budget。
    
- Burn Rate = 2：消耗速度是预期的 2 倍。
    
- Burn Rate = 10：消耗速度是预期的 10 倍。
    

可以通过 Burn Rate Alert 及时发现服务可靠性正在恶化的情况。

**注意：**

Error Budget 不是简单预测未来是否违反 SLA，而是根据 SLO 和实际服务表现计算允许的不可靠预算，并监控预算消耗速度。

**面试问题：**

1. SLI、SLO 和 SLA 有什么区别？
    
2. 什么是 Error Budget？如何计算？
    
3. 如果服务 SLO 是 99.9%，一个月允许多少不可用时间？
    
4. Error Budget 消耗过快时，工程团队应该怎么做？
    
5. 什么是 Error Budget Burn Rate？
    
6. 为什么 SRE 团队需要在开发速度和系统稳定性之间进行权衡？
    

---

# 核心考点总结

|模块|核心关键词|
|---|---|
|身份认证|Service Account、ADC、Metadata Server、WIF|
|企业身份管理|Cloud Identity、GCDS、SAML、SSO|
|组织安全策略|IAM Policy、Organization Policy、Policy Inheritance|
|Cloud SQL 网络|Public IP、Auth Proxy、PSA、PSC|
|Serverless 网络|Serverless VPC Access、Direct VPC Egress|
|数据访问审计|Access Transparency、Access Approval、Assured Workloads|
|数据外传防护|VPC Service Controls、Service Perimeter、Access Level|
|远程访问|IAP TCP Forwarding、VPN、Bastion Host|
|数据仓库权限|BigQuery Job User、Data Viewer、Data Editor|
|NoSQL 读取|Firestore Batch Get、Datastore Lookup|
|软件完整性|JAR Signature、Hash、Public Key、Private Key|
|数据合规|PCI DSS、Encryption、Audit Logs|
|软件供应链|Artifact Analysis、Vulnerability Scanning|
|迁移评估|Migration Center、Cost Estimation|
|API 安全|Apigee、OAuth 2.0、API Key、Quota、SpikeArrest|
|成本优化|SUD、CUD|
|服务可靠性|SLI、SLO、SLA、Error Budget、Burn Rate|