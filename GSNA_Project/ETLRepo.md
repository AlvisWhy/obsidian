负责全球fincrime多平台（txn customer cdm ifai gtrf orbis icij）的etl 构建客户账户交易情报关系网 
面向百亿级的交易处理能力 落地分区子分区 智能子集 去重校验机制 提升稳定性可回溯性
产出直接服务告警评分关系调查的标准化实体和风险特征数据



Working in global Finicial crime monitor data platform, responsible for mutisource 异构 ETL(tracscation/ Customer/ CDM customer/ IFAI/ Orbis/ ICIJ/ ExportGenius), 全球上百个国家持续变化的数据源，make 亿级到百亿级别交易主体数据汇聚成 可检索，可关联，alertable， scrabble， unified entity and relationship network. 
Using partition/sub partition and index shards to control the stability and cost for ETl process
subpartition(For huge data, load and process with chunks)

### 大表治理
Using editable configs for large dataset: use run_duration to control the input data range, use maxdeduplicationcount per time(2亿) to control max operation

1 Use minloadbysubpartitioncount = 1000000000，used for huge subpartition load 
CB交易索引，多在千万级到亿级
Orbis partitioncount = 50000 按实体类型（bussiniess. individual, address, phone, email, search）配置recordspershard
using withAdditionalRepartitional and withModuloHashSubpartition to make large txn into multi small subpartitions

2 Using chunk in deduplication operations , 使用去重键构造hash，通过harh分chunk， 每个chunk分别去重最后union。降低单词shuffle的OOM和长时间GC
判断需要分块计算：单次去重出现明显spill，executor OOM，starge retry，去充key skew，

3 to maintain the stability of system, for customer whose account > 50000 They will not be calculated in the ETL. Using boradcast to spread the large account name list to all the partitions to filter

4 Using cache persist and checkpoint to avoid recalculation .

cache: low expense, fast reuse, suitable for multiple consuming in same jobs
persist: using disk, avoid OOM
checkpoint: cut long lineage suitable for failure rerun
### 业务提升

4 交易域过滤
from domain， extract noncbtxn and cbtxn， then combine accoriding to countries and rules
在保留业务有效交易的前提下， 剔除口径外交易 确保后续归户可解释可审计
cintolookup，交易的account映射成customer number 让交易信息变成用户的行为

5 identifier 体系优化
在business层面需求是几个数据源体统要合并，体系原有的identifier的基础上，各系统（如 CAMP/TMIS/GCDU）等客户号语义 国家口径字段存在差异，导致同一客户在不同系统可能被拆成多个实体，下游出现疑似重复客户，影响告警聚合，关系分析和人工调查效率
(后三个是boolean)identiferschema: idval, idtype, idsource, idcountry, idtypeclobaluniquefield, common customeridfield, gtrflinkingcustomeridfield.

创建多个identifier builder，按不同国家返回不同的builder 数组，

在createdataclass的时候加入identifier字段

下游根据这个字段筛选match 根据不同国家定义是否match，并叠加parsed_full_name/alias 作为软特征辅助召回

在识别为同一人后做join 把交易账户客户信息连接成一个人 进行去重

利用builder schema + 条件化注入，低风险， 同时处理工程可维护，规则正确，下游可解释的identifier，每个identifer带type source country flag可追溯
降低了跨系统拆分的疑似用户，提升客户归户一致性，同时保留归并逻辑让下游可以判断。

### 智能uat测试子集构建机制实现
通过随机采样抽取documentid（包括supplier和buyer）先构建可追溯的id锚点 再对大表做半链接/命中过滤 把全量裁剪成结构完整关系不丢失的可验证子集 用于UAT和迭代

1 IFAI intelligence subset(ID 驱动 + 半链接裁剪)
broadcast + leftsemi
通过统一的config对多表配置join key，根据表名字筛选key去做join
对待裁剪大表dftobesetsub先做persist（memory and disk）并计数，避免重复action重算，
前后会触发两次action，需要persist，内存放不下落盘避免OOM，做可会退缓存

对document id拆分提取supplier和buyer， 去重后broadcast成小表做裁剪依据

（leftsemi只保留左表命中行 比inner join更省shuffle 内存，避免有表重复键导致膨胀）

2 QFD: 一致性样本采集（客户样本驱动 + 账户传播）
先sample（false，sampleperstance）
手工补充manualsubset与客户inner join做union生成主样本
对subset进行cache + broadcast
最后操作完进行unpersist

不会导致只抽到客户没有抽到交易（反之）

customerid去重 + cache再broadcast到executor 

Construct relationship network

1 IFAI: 贸易关系网络
 Use supplier and buyer identification create relationship key 
 聚合形成供应商集中度 敞口 近一年融资等风险特征
 计算concentration_pct 1 year/ finaned amount 1year 作为风险系数
供应商集中度 = 关系融资额/供应商总融资额

2 GTRF 票据/贷款/信用证贸易链
整合多源表 统一到trade item语义层
支持基于trade_document_id parent_document_id trade_account_id 的上下游关联

3 ICIJ 离岸图谱实体关系
输入原始关系网的relationship，整理成以公司 entity为中心的人物关系卡片，先拿relationship的边 再去entiy/officer/intermediaries/address去找这些点都是什么类型，最后每一个实体记录都会有这个实体关联了哪些人，都有哪些地址，每个实体关联哪些地址，关系角色信息。

4 orbis 企业所有权和控制关系
吧orbis分散在多个表的关系信息 按一个企业主键汇总成一条公司档案，包括公司基础信息 股东 董事 子公司 顾问
 股东 董事 子公司 顾问四类数据先各自清洗，然后通过企业主键进行join
 先把明细关系表按公司进行关系整理成一个数组 先在四类主表按公司内部聚合 再和公司主表进行聚合，避免join的时候产生大量的行导致行数爆炸
 
整体：客户 账户 交易对手 企业 离岸实体化为caseclass
账户-客户 买方-卖方 公司-股东 实体-地址/人员关系可查询
风险特征化：
金额归一：对多币种交易保留原有金额 对比USD口径 解决跨国交易问题
集中度：不止看金额 还看关系交易主体/融资占比 用于识别高依赖的
时间窗口标志：近12月和历史口径对比 分析风险
跨源标签：内部交易数据集合外部情报 形成复合风险标签
offshore relation: 实体到董事到地址到收益人的多级连跳 上金融犯罪的重点监控对象
通过icij的图谱关系图吧entities officer itermediater address关联回实体 再和客户交易叠加 生成离岸加关系行为的复合视角
创建in-domain label：风险等级 历史报警次数 国家交易和对手标记 关系标签（买卖双方，收益所有人，子母公司）

降低多系统不可分析数据的门槛 给analyser统一口径的数据


如何处理异构表的schema不稳定问题：
在ifai读区缺失字段自动补齐null 并统一字段类型再trim可以兼容上游schema的变差

参与构建 ETL Datarecon 数据对账框架 建立 Input/Output/Filter/Deduplicate/Orphan（关联数据存在 但是对应不到其中一个节点的关系断裂）全链路计数闭环，通过报告级娇艳欲失败明细落盘（parquet + json summary）实现可追溯数据统计审计 并自动失败时抛出异常阻断发布，降低无感数据丢失和人工核对成本

1 原来ETL过滤去重孤儿很难快速回到哪里少了 为什么少了，我们把处理动作抽象成Dataaction，按rawdatapath和reason聚合 形成全链路计数账本
DataMetric/ DatareconReport 定义每一类rawdata的输入输出 过滤去充孤儿计数的原因字段
```
case class DataMetric(
	rawDataFilePath: String,
	role: String,
	testResult:String,
	reason: String,
	inputCount:Long,
	outputCount:Long,
	filterTotal:Long,
	FilterStatics:Seq[MetricStatics],
	DuplicateTotal:Long,
	DuplicateStatics:Seq[MetricStatics],
	OrphanTotal:Long,
	..
)
```
DataAction
定义orphanrecorddataaction, dfdataaction, dfadditionaldataaction,dfaggregationdataaction, becoreafterdataaction, countdataaction.几类继承basedataaction 可以同时记录对数据的操作
包含dataactiontype rawdatapath dataframe和reasonkey

其中
Dfdataaction: using countV, reson记录raw_datainput
							或代表join后的表
```join + orphan split
val joinDF = dedupDF.join(msterdf, Seq("customer_id"), "left")
val orphanDf = joinedDF.filter("master_key is null")


```
becoreafterdataaction: using before.count - after.count, reason 记录过滤掉的数量或去充的id
dfadditionaldataaction：-count， reason代表backfill的数量回填
aggregationdataaction: 取聚合列值： 避免重复count全表
orphanrecorddataaction: union后统一count, reason 代表missin_master_key


2 计数层采集生成报告
在开启datarecon之后，每个计算步骤在操作后会以对应类型的action把操作，dataframe什么的信息发到总的容器中等待后面统计
计算过程中的表部分缓存起来，部分大表落盘避免重算。

3 校验层 检验指标
input= output + filter +deduplicate + orphan
不等直接返回fail + 差值原因
先计算subetid的 再计算总的

4 统计报警

总体：
每个spark job在自己的进程内用datareoncrunner收集action 生成datarecpnreport
job结束前用saveDatareconreport把报告写到输入出目录下
如果有孤儿数据再写一个路径 
到汇总任务的时候用reconsummarygenerator生成汇总记录

为什么会出现input= output + filter +deduplicate + orphan不成立情况出现
1 数据没有对齐 选partition的内容不对等
2 重跑重试错误 idompotency error
3 跨天跑数据导致job前后数据集选取日期不同
4 去重性能不稳定，不能按代码逻辑完成实际去重