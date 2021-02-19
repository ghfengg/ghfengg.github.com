# presto学习笔记

# 基本概念
**
## **Presto Server**


- Coordinator
担当 Master 角色，负责解析 SQL，生成查询计划，提交查询任务给 Worker 执行，管理 Worker 节点。
- Worker
执行任务和处理数据
## **Data Source**


- **Connector**
Connector 是一个适配连接器，Presto 使用 Connector 去连接不同的数据源，比如 Hive 、关系型数据库等。可以通过实现自己的 Connector 去扩展数据源。

- **Catalog**
Catalog 多个 schema 的集合,表示通过 connector 获取的一种数据源，你可以使用 hive connector 的多个 catalog 来代表不同的 hive 集群数据源。常见的 catalog 为：mysql catalog,hive catalog 等

- **Schema**
表的集合，类似于 Hive、MySQL 中的 database。

- **Table**
类似于Hive中的table


查询 catalog 为 hive，数据库为 test，表为 table1 的语句为
`select count(*) from hive.test.table1`
## **Query Model**


- **Statement**
表示一个 SQL 查询语句

- **Query**
表示 Statement 经过解析，生成的执行计划，查询计划。在 Presto 集群中运行的查询， 一个 Query 由多个 Stage 组成、Task、Driver、Split、Operator 和 Datasource 组成。

- **Stage**查询执行阶段，一个 Query 会被拆分成具有层级关系的多个 Stage 执行,一个 Stage 就是查询执行计划的一部分。 四种stage:
   - Coordinator_Only:一般表示 DDL,DML 的 Stage。
   - Single：用于聚合子 stages 数据，并最终将数据输出给终端用户。比如每个查询中的 Root Stage。
   - Fixed：用于接收子 Stage 产生的数据，并在集群中对这些数据进行聚合或分组计算。
   - Source：连接数据源，从数据源读取数据。
- **Exchange**连接不同的 Stage，用于不同 Stage 之间的数据交互
   - Output Buffer:向下游提供数据，数据提供者
   - Exchange Client：从上游读取数据，数据消费者
- **Task**
Stage 有多个 Task 组成。Stage 并不会运行，只是负责管理 Task 和封装建模。Stage 实际运行的是 Task。每个Task 处理一个或者多个 Split。每个 Task 都有对应的输入和输出。

- **Driver**
Task 被分解成一个或者多个 Driver，并行执行多个 Driver 的方式来实现 Task 的并发执行。Driver 是作用于一个 Split 的一系列 Operator 的集合。一个 Driver 处理一个 Split，产生输出由 Task 收集并传递给下游的 Stage 中的一个 Task。一个 Driver 拥有一个输入和输出。
- **Operator**
Operator 表示对一个 Split 的一种操作。比如过滤、转换等。 一个 Operator 一次读取一个 Split 的数据，将 Operator 所表示的计算、操作作用于 Split 的数据上，产生输出。每个 Operator 会以 Page 为最小处理单位分别读取输入数据和产生输出数据。Operator 每次只读取一个 Page,输出产生一个 Page

- **Split**
一个分片表示大的数据集合中的一个小子集,与 MapReduce 中的 Split 概念类似。

- **Page**
Presto 中处理的最小数据单元。一个 Page 对象包括多个 Block 对象，而每个 Block 对象是一个字节数组，存储一个字段的若干行。多个 Block 的横切的一行表示真实的一行数据。一个 Page 最大1MB，最多1 6x1024 行数据。



# SQL编译过程
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/16292/1606892520402-08c0a080-63fb-4b0c-8c2a-85e093b1a90e.png#align=left&display=inline&height=147&margin=%5Bobject%20Object%5D&name=image.png&originHeight=226&originWidth=1145&size=79839&status=done&style=none&width=746)
LocalExecutionPlan是在每个worker节点上执行



# 基本概念


![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/16292/1606892390402-72735261-0b0d-4606-ad90-9b3242d394fc.png#align=left&display=inline&height=307&margin=%5Bobject%20Object%5D&name=image.png&originHeight=307&originWidth=636&size=23050&status=done&style=none&width=636)





# 表模型
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/16292/1606880514978-1630b0cd-8c4a-4056-8c17-562db74ea3ea.png#align=left&display=inline&height=371&margin=%5Bobject%20Object%5D&name=image.png&originHeight=371&originWidth=570&size=27604&status=done&style=none&width=570)

# MapReduce VS MPP
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/16292/1606892587235-bf8b166c-634b-4ce2-a177-a1c0f28adcaa.png#align=left&display=inline&height=844&margin=%5Bobject%20Object%5D&name=image.png&originHeight=844&originWidth=1224&size=687107&status=done&style=none&width=1224)




# 知识点
1、Presto Task 是由Stage来调度的，属于执行前调度。执行前调度表示整个查询在真正执行前都已经分配完毕，不会在执行过程中更改Task所在的计算节点
2、Presto的查询调度本质就是Split分配到各个节点的过程,每个阶段(Stage)依据本身所承担的职责，调度方式有所区别
3、Split配置([https://prestodb.io/docs/current/admin/properties.html](https://prestodb.io/docs/current/admin/properties.html)):
### `node-scheduler.max-splits-per-node`
> - **Type:** `integer`
> - **Default value:** `100`
> 
The target value for the total number of splits that can be running for **each worker node**.
> 

> Using a higher value is recommended if queries are submitted in large batches (e.g., running a large group of reports periodically) or for connectors that produce many splits that complete quickly. Increasing this value may improve query latency by ensuring that the workers have enough splits to keep them fully utilized.
> 

> Setting this too high will waste memory and may result in lower performance due to splits not being balanced across workers. Ideally, it should be set such that there is always at least one split waiting to be processed, but not higher.

待确认？ 每个split的默认大小是多少？ 32MB？
4、每个Operator均会以Page为最小处理单位分别读取输入数据和产生输出数据
5、Source Stage 选择节点的个数是根据Table的Split个数来决定的(SplitManager)
6、Presto内部有4种类型的计算节点

- source节点，读源数据的节点，负责读取数据、Map阶段的计算任务。分配的个数由SplitManager根据数据决定
- fixed节点，shuffle节点，用于处理reduce任务。比如group by计算，source阶段的数据按照hash发送到fixed节点。分配的个数由hash_partition_count这个session参数决定，由于是固定个数，所以对于不同的数据规模采用同样的个数也不是很合适，这也是需要改进的点
- single节点，单节点计算任务，某些计算需要在单一节点计算，比如MergeReduce，output等，需要分配一个节点
- coordinator only，只在coordinator节点计算的任务，一般是meta类操作

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/16292/1607497973528-0809bf70-7904-45f0-b99a-2a3501060352.png#align=left&display=inline&height=570&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1140&originWidth=2038&size=197495&status=done&style=none&width=1019)
7、Split加载过程see:[https://blog.csdn.net/zhanyuanlin/article/details/109215177](https://blog.csdn.net/zhanyuanlin/article/details/109215177)



