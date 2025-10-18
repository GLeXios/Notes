# Flink行内

## 集群部署、备份、数据恢复

### 测试环境

#### 集群部署

​	大数据平台当前在测试环境提供多套集群以供测试。

​	1）BDP集群

​	BDP集群有两套测试集群：SIT、UAT。

​	SIT，UAT为大数据平台提供的两套独立的集群，与业务IT方划分的测试环境并不完全一致，其对应关系，由业务IT方自行维护，包括但不限于机器归属、CMDB变量等。大数据平台不参与业务IT的测试环境信息维护。

​	2）BDAP集群

​	BDAP集群只有一套：SIT

#### 集群备份

测试环境的集群均无备份集群，相关数据、应用均由使用方自行备份。

### 生产环境

#### 集群部署

  **主机环境变量检查，检查主机的/appcom/config/env.sh中有flink 运行的环境的变量；**

```bash
 export PATH=/appcom/Install/mask_utils/bin:$PATH
 
 export FLINK_HOME=/appcom/Install/flink
 
 export FLINK_CONF_DIR=/appcom/config/flink-config
 
 export FLINK_LOG_DIR=/appcom/logs/flink
 
 export PATH=$FLINK_HOME/bin:$PATH
 
 export HADOOP_CLASSPATH=`hadoop classpath`
```

​	1）BDP集群

​	BDP集群分主、备集群，当前主集群在坪山，备集群在福田。

​	2）BDAP集群

​	BDAP集群只有在坪山有一套集群。

#### 集群备份

​	**数据完整性时效：**

​	Checkpoint、Savepoint等数据应使用HDFS作高可用存储，用户数据及作业产生的数据如不使用HDFS存储，应由用户自行处理完整性时效。依赖于HDFS存储的数据完整性时效，参考《Hadoop基本法》集群备份。

​	**备份说明：**

​	依赖于HDFS存储的数据备份，参考《Hadoop基本法》集群备份。

#### 数据恢复

​	**功能说明：**

​	当前生产环境的Hadoop集群都已启用回收站功能， Checkpoint、Savepoint、hive表、实时数据的删除操作，数据会进入回收站：/user/{username}/.Trash。最新回收数据在/user/{username}/.Trash/Current。

​	**恢复时效：**

​	回收站数据恢复时效，由Hadoop平台决定，参考《Hadoop基本法》

### 高可用部署规范

**单个Flink Job的高可用由Flink HA机制+Yarn Attemp机制来保障**。

- 当Flink TaskManager出现故障时，会触发Task Failover，由Flink JobManager重启相应的Task。
- 当Flink JobManager故障时，会由Yarn自动重启Flink JobManager。所默认配置为30min内可自动恢复1次，如JobManager在短时间内频繁重启，则会导致作业失败退出。以需要用户严格按照1.1.3配置Flink Job HA相关参数。

**Flink集群的高可用，需要用户采用跨IDC双活部署的方案。**

![image-20250406150821990](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406150821990-3929427.png)



## Flink应用开发规范

### 重要配置项

​	请注意以下配置关系到任务运行的稳定性，标记为必填的配置项，属于用户必须要按照建议值配置的，后续会在Streamis中提交任务时进行校验，和定期进行存量任务的巡检。还有一些特殊配置注明了统一在平台添加的，不允许用户显示配置。

#### 一个flink应用占用的资源

**`总内存 ≈ JobManager内存 + TaskManager数量 × TaskManager内存`**

#### 资源相关

【**jobmanager.memory.process.size**】

**是否必填：**是

**建议值：[2g,8g]** 

**说明：对应streamis物料包meta.json文件中的wds.linkis.flink.jobmanager.memory参数**。因为streamis限制了该参数必填，一般情况只要设置该参数值即可，Flink会根据进程总内存大小自动计算分配其他各区域的内存大小，如果需要更精细化的设置其他区域内存大小，注意与配置的进程总内存大小相匹配，否则可能出现IllegalConfigurationException异常。**内存设置太小，可能造成内存溢出或者频繁full gc问题。内存设置过大，可能到造成gc时间较长**。所以建议jobmanager内存设置范围为[2g,8g]。



【**taskmanager.memory.process.size**】

**是否必填：**是

**建议值：[2g,4g]**

**说明：** **对应streamis物料包meta.json文件中的wds.linkis.flink.taskmanager.memory参数**。因为streamis限制了该参数必填，一般情况只要设置该参数值即可，Flink会根据进程总内存大小自动计算分配其他各区域的内存大小，如果需要更精细化的设置其他区域内存大小，注意与配置的进程总内存大小相匹配，否则可能出现IllegalConfigurationException异常。内存设置太小，可能造成内存溢出或者频繁full gc问题。内存设置过大，可能到造成gc时间较长。所以建议taskmanager内存设置范围为[2g,4g]，**如果单个taskmanager设置为4g内存不够，可以通过扩并行度解决。**



【**pipeline.max-parallelism**】

**是否必填：**是

**建议值：**大于算子并并行度，预留后续扩容的空间，设置为算子并行度的倍数值。

**说明：** **flink job最大并行度**，默认值为-1。不允许使用默认值。原因是如果没有由用户指定算子的最大并行度，后续一旦用户调整了算子的并行度，Flink会按照算子并行度重新计算一个最大并行度，而如果最大并行度发生变化，就有可能导致Flink流处理作业无法从之前的快照恢复，从而导致流处理作业产出错误的结果。**算子最大并行度和算子并行度保持倍数关系，可以保证每一个SubTask中分配的KeyGroup个数是相同的，也不建议最大并行度配置得特别大，因为最大并行度就是KeyGroup个数，会加大内存消耗，也会导致Flink作业从快照恢复时键值状态的恢复压力也会比较大。**

Flink默认计算最大并行度逻辑：默认情况下，Flink会给每一个算子自动计算出一个最大并行度，计算步骤如下。第一步，计算n=算子并行度+（算子并行度/2）。第二步，计算m=n四舍五入后大于n的一个整数值，并且要求m是2的幂次方。举例来说，当n为300时，m的取值为512。第三步，当m≤128时，取128作为算子的最大并行度；当128<m≤32768时，则取m作为算子的最大并行度。注意，m不能超过32768。

#### Checkpoint相关

【**execution.checkpointing.interval**】

**是否必填：**是

**建议值：**[5min,10min]

**说明：** **checkpoint 时间间隔**。请按照建议范围进行设置，不宜过短和过长，**设置过短会造成系统性能开销增加，影响Job运行稳定性和吞吐。设置过长会导致从checkpoint恢复时间变长，单次checkpoint大小会变大，也会影响Job运行稳定性，任务恢复时需要重放更多数据，导致恢复时间变长。**如果因为sink端依赖checkpoint进行数据flush,并且下游对数据时效要求是秒级别， 需要降低checkpoint间隔，请提前向BDP-FLINK研发提出，综合评估后进行调整。

> [!CAUTION]
>
> **重要注意项：**
>
> **<font color = '#8D0101'>（1）Flink作业的算子有状态，并且业务不能容忍数据丢失或计算结果错误的场景，不允许将execution.checkpointing.interval设置为小于等于0来禁止checkpoint行为，这会导致task异常failover后无法从checkpoint恢复state数据，无法保证端到端AT_LEAST_ONCE和EXACTLY_ONCE语义，影响数据准确性。比如常见的问题有，Kafka Source关闭checkpoint会造成数据重复消费或者消费丢失（由auto.offset.reset决定）。窗口聚合算子关闭checkpoint会造成task重启后聚合结果不准确。使用两阶段提交的Sink端会依赖checkpoint完成进行数据commit，如果关闭checkpoint会导致数据无法正常commit。</font>**
>
> （2）如果sink端使用了两阶段事务提交，比如kafka，一定要将事务超时时间设置成大于(execution.checkpointing.interval+checkpoint duration),否则会导致事务频繁abort。以kafka sink为例，应该设置transaction.timeout.ms，如果使用的Flink SQL，应该设置，并且要注意kafka broker端的配置transaction.max.timeout.ms（默认15min）需要大于transaction.timeout.ms的值，不然不生效。
>
> **两阶段事务提交**
>
> **第一阶段：准备提交（Prepare Commit）**
> 生产者调用 commitTransaction API 后，协调者将事务状态更新为 “prepare_commit” ，并将该状态写入事务日志（持久化）。
> 协调者向所有涉及事务的分区（生产者写入的分区和消费者位移提交的分区）发送 事务控制消息 ，标记事务进入准备阶段 。
> **第二阶段：正式提交（Commit）**
> 协调者等待所有分区确认准备完成后，向分区发送 提交指令 ，确保数据对消费者可见 。
> 若任一分区失败，协调者会触发回滚（abortTransaction），保证原子性 。
>
> 在 Flink 中，Kafka 生产者通过继承 `TwoPhaseCommitSinkFunction` 实现两阶段提交：
>
> 1. 预提交（Pre-Commit） ：算子完成处理后，预提交数据到临时存储 。
> 2. **确认提交（Commit）** ：所有算子成功后，正式提交事务



【**execution.checkpointing.externalized-checkpoint-retention**】

**是否必填：**是

**建议值：**RETAIN_ON_CANCELLATION

**说明： checkpoint保留策略**。默认值为NEVER_RETAIN_AFTER_TERMINATION，表示checkpoint不会保留，在Job finished、cancelled、failed时都会删除checkpoint。RETAIN_ON_FAILURE，表示只有Job failed时才会保留checkpoint，其他状态时会删除checkpoint。**RETAIN_ON_CANCELLATION，表示Job cancelled、failed都会保留checkpoint，只有finished状态会删除checkpoint。如果采用默认值，会导致job异常失败后无法从checkpoint恢复，所以禁止使用默认值，统一配置为RETAIN_ON_CANCELLATION。**



【**state.checkpoints.num-retained**】

**是否必填：**是

**建议值：3**

**说明：checkpoint最大保留数量**，默认值为1。Flink JobManager每次在checkpoint完成时，对历史checkpoint进行清理，只保留最新的参数值数量的checkpoint，**为了防止异常情况下造成的checkpoint文件损坏，导致flink job无法从checkpoint恢复，所以要求该参数值设置为3。**



【**execution.checkpointing.min-pause**】

**是否必填：**是

**建议值：**保持和execution.checkpointing.interval一样

**说明：checkpoint最短时间间隔**，默认值为0，如果checkpoint耗时长，会导致两个checkpoint之间的实际间隔很短，造成频繁触发checkpoint。**要求将该参数值和execution.checkpointing.interval保持一样，这样即使checkpoint耗时长，也会保证两个checkpoint之间最短时间间隔**。



【**execution.checkpointing.timeout**】

**是否必填：**是

**建议值：20**min 

**说明：checkpoint超时时间**。至少设置为20min，如果你的状态比较大，可以视情况增大该参数值，也可以通过开启增量checkpoint和扩并行度来降低checkpoint时间。



【**execution.checkpointing.tolerable-failed-checkpoints**】

**是否必填：**是

**建议值：3**

**说明： 最大容忍checkpoint 连续失败次数**，超过该阈值会触发 flink job restart，默认值为0，意味着只要出现checkpoint失败就会触发job restart。**该参数不宜设置过小和过大，设置过小，作业可能会因为临时的网络和资源抖动而频繁restart，影响系统的稳定性和数据的实时性。如果该值设置过大，会增加作业恢复的时间，并且可能会掩盖一些潜在的问题，使得问题在早期阶段不容易被发现。**要求用户将该参数值设置为3，并**配合checkpoint失败监控告警来及时感知到checkpoint失败情况**。



【**execution.checkpointing.max-concurrent-checkpoints**】

**是否必填：**否

**建议值：**1

**说明： 最大并发执行的 checkpoint 数量**，默认值为1。默认情况下，在上一个 checkpoint未完成时，系统不会触发另一个 checkpoint。如果并发执行多个checkpoint，会造成dfs 写 IO压力，从而影响job 稳定性，所以禁止将该参数值设置为大于1的值。



【**state.backend**】

**是否必填：**是

**建议值：**filesystem或者rocksdb

**说明：状态后端**，决定了**运行时状态数据和状态快照存储位置**。

- Filesystem状态后端，运行时状态数据存放在堆内存中，状态快照存储在分布式存储上（hdfs）,所以filesystem状态后端读写性能更好，但是会占据堆内存，可能导致gc问题。
- RocksDb状态后端，运行时状态数据存放在RocksDB中，状态快照一样需要存放在存储上（hdfs）,但是它支持增量快照，所以状态快照异步写的时间会更短。
- 当状态总大小小于1G时，如果想要性能更好，允许使用filesystem状态后端。如果状态大小超过1G，则应使用rocksdb状态后端，并且禁止用户设置为jobmanager。



【**state.backend.incremental**】

**是否必填：**是

**建议值：**如果使用rocksdb状态后端，应设置为true

**说明：**是否开启增量checkpoint，默认值为false。如果使用了rocksdb状态后端，要求该参数值设置为true，已降低异步快照写时的IO消耗，加快checkpoint完成速度。



【**state.checkpoints.dir**】

**是否必填：**否

**建议值：**无

**说明：**checkpoint保存路径，禁止用户显示设置，因为在开启了checkpoint后，streamis平台会根据一定规则动态设置改路径。

#### Job HA相关

```properties
high-availability: zookeeper

high-availability.storageDir: hdfs:///flink/recovery

high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181

high-availability.zookeeper.path.root: /flink

high-availability.cluster-id: /default_ns

zookeeper.sasl.service-name: hadoop

zookeeper.sasl.login-context-name: Client

yarn.application-attempts：3

yarn.application-attempt-failures-validity-interval:600000
```

<font color = '#8D0101'>**以上参数会统一在Flink平台设置，禁止用户显示设置。**</font>

【**restart-strategy**】

**是否必填：**是

**建议值：**fixeddelay或者fixed-delay

**说明：**flink task异常时的重启策略，默认值为none, off, disable。禁止使用默认值，推荐使用逻辑简单的固定次数延时重启策略。

 

【**restart-strategy.fixed-delay.attempts**】

**是否必填：**是

**建议值：**5

**说明：**固定次数延时重启策略最大重试次数，默认值为1,要求至少设置为5次。

 

【**restart-strategy.fixed-delay.delay**】

**是否必填：**是

**建议值：**10 s

**说明：**固定次数延时重启策略两次重试之间的时间间隔，默认值为1s，要求设置至少为10 s。



#### 安全相关 kerberos.login.keytab

【**security.kerberos.login.keytab**】

**是否必填：**是

**建议值：**/appcom/keytab/${username}.keytab

**说明：**kerberos keytab文件，用于kerberos认证，指定错误会导致Flink Job提交失败。（发布时在streamis中自动按应用启动用户进行生成）

 

【**security.kerberos.login.principal**】

**是否必填：**是

**建议值：**${username}@WEBANK.COM

**说明：**需要在keytab文件中存在，否则向KDC认证不通过。（发布时在streamis中自动按应用启动用户进行生成）

 

【**security.kerberos.login.contexts**】

**是否必填：**是

**建议值：**Client,KafkaClient

**说明：**用于Zookeeper和Kafka的kerberos认证。（发布时在streamis中自动按应用启动用户进行生成）

#### JVM相关 env.java.opts

**是否必填：**是

**建议值：**

```bash
-Djava.security.krb5.conf=<your-config-directory>/krb5.conf -Djava.security.auth.login.config=<your-config-directory>/kafka_client_jaas.conf -Dfile.encoding=UTF-8 -Xloggc:<LOG_DIR>/gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=50M -XX:+PrintPromotionFailure -XX:+PrintGCCause  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<LOG_DIR>/dump.hprof
```

**垃圾回收器选择建议：**`-XX:+UseG1GC或者-XX:+UseConcMarkSweepGC` 

**说明：该参数用于设置JobManager和TaskManager JVM参数**，允许在建议值基础上添加自己的JVM参数，但注意建议值是必须添加的参数。除了可以通过env.java.opts设置JobManager和TaskManager JVM参数之外，还可以通过env.java.opts.jobmanager和env.java.opts.taskmanager分别设置JobManager和TasManager的JVM参数。JDK8默认垃圾回收器组合是Parallel Scavenge+Parallel Old,但因为Flink流式应用对低延迟和可预测的暂停时间有较高要求，所以建议**使用G1（-XX:+UseG1GC）或者ParNew+CMS（-XX:+UseConcMarkSweepGC）替代Parallel**。因为G1更占内存，如果堆内存本身比较小（小于4g），可以使用ParNew+CMS。

#### 日志相关

Flink平台目前支持的日志框架是log4j-2.17.0,允许用户配置应用自己的log4j.properties配置文件，但需要符合以下规范：

（1）生产环境日志级别只允许设置INFO（含）级别以上，禁止设置为DEBUG、TRACE、ALL级别。

（2）避免过多的无效日志打印，和在日志里面打印数据明细信息，导致覆盖有效日志。

（3）**本地日志输出appender，必须使用RollingFile，日志文件数量限制为10个，单个日志文件大小限制为100M，避免日志文件过大，将磁盘打满**。如果有需要保留更多的历史日志，建议接入统一将日志上报MSS。log4j appender配置参考如下：

```properties
appender.main.type=RollingFile

appender.main.append=true

appender.main.fileName=${sys:log.file}

appender.main.filePattern=${sys:log.file}.%i

appender.main.layout.type=PatternLayout

appender.main.layout.pattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n

appender.main.policies.type=Policies

appender.main.policies.size.type=SizeBasedTriggeringPolicy

appender.main.policies.size.size=100MB

appender.main.policies.startup.type=OnStartupTriggeringPolicy

appender.main.strategy.type=DefaultRolloverStrategy

appender.main.strategy.max=${env:MAX_LOG_FILE_NUMBER:-10}
```

##  Connector使用规范（Kafka Source）

### DataStream程序

```java
Properties properties = new Properties();
// 指定kafka brokers地址
properties.setProperty("bootstrap.servers", "brokerhosts:9092");
// 指定消费组名称
properties.setProperty("group.id", "替换成自己的消费组名称");
// 指定一个topic进行消费
DataStream<String> stream = env
    .addSource(new FlinkKafkaConsumer<>("topic", new SimpleStringSchema(), properties));
// 指定多个topic进行消费， 可以将多个topic添加到list中
DataStream<String> stream = env
    .addSource(new FlinkKafkaConsumer<>(list, new SimpleStringSchema(), properties));
```

#### 1、设置Kafka Consumer参数 

​	可以通过构建Properties对象实现Kafka Consumer配置，指定properties的key对应Kafka Consumer的配置项名称，Kafka Cosumer常用配置项参考[《Kafka使用规范（新）》](http://docs.weoa.com/docs/9030Md1j2XI0odqw)4.3消费者重要参数和[社区Ka](https://kafka.apache.org/10/documentation.html#consumerconfigs)[f](https://kafka.apache.org/10/documentation.html#consumerconfigs)[ka使用文档](https://kafka.apache.org/10/documentation.html#consumerconfigs)。

```java
Properties properties = new Properties();
// 指定kafka brokers地址
properties.setProperty("bootstrap.servers", "brokerhosts:9092");
// 指定消费组名称
properties.setProperty("group.id", "替换成自己的消费组名称");
// 指定一个topic进行消费
DataStream<String> stream = env
    .addSource(new FlinkKafkaConsumer<>("topic", new SimpleStringSchema(), properties));
```

#### 2、指定起始消费位置

​	Flink DataStreamAPI允许用户自己通过以下方法指定起始消费位置。但请注意如果Job是从checkpoint/savepoint恢复运行的，以下设置不会生效，起始消费offset只会采用从checkpoint/savepoint文件中记录的offset，在checkpoint/savepoint文件不存在的partition（比如新增的partition），默认从该partition当前记录的最早的offset位置开始消费。

```java
FlinkKafkaConsumer<String> myConsumer = new FlinkKafkaConsumer<>(...);

myConsumer.setStartFromEarliest();   
myConsumer.setStartFromLatest();    
myConsumer.setStartFromTimestamp(...); 
// 默认方法
myConsumer.setStartFromGroupOffsets(); 
//细粒度的指定每个分区的起始消费位置
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);
myConsumer.setStartFromSpecificOffsets(specificStartOffsets);

DataStream<String> stream = env.addSource(myConsumer);
...
```

setStartFromGroupOffsets()：从Kafka broker消费者组中记录的offset开始消费，如果记录的offset已经不存在，将由auto.offset.reset配置决定起始消费位置。使用该方法，必须指定group.id，否则也将由auto.offset.reset配置决定起始消费位置。

setStartFromEarliest()： 从每个partition当前记录的最早的offset位置开始消费。

setStartFromLatest()：从每个partition最新offset位置开始消费

setStartFromTimestamp()：每个partition会从时间戳大于或等于指定时间戳的第一条消息开始消费，如果没有找到默认从最新offset位置开始消费 

setStartFromSpecificOffsets：细粒度的指定每个分区的起始消费位置

#### 3、offset提交方式

Flink支持三种Kafka消费位移提交方式：ON_CHECKPOINTS、KAFKA_PERIODIC、DISABLED。

ON_CHECKPOINTS：每次checkpoint完成时，自动Kafka Server提交消费位移。这是开启了checkpoint时的默认提交方式。

```java
FlinkKafkaConsumer<String> myConsumer = new FlinkKafkaConsumer<>(...);

myConsumer.setCommitOffsetsOnCheckpoints(true);
```

KAFKA_PERIODIC：定时向Kafka Server提交消费位移。这是关闭checkpoint时的默认提交方式。

```java
FlinkKafkaConsumer<String> myConsumer = new FlinkKafkaConsumer<>(...);

Properties properties = new Properties();

properties.setProperty("enable.auto.commit", "true");

properties.setProperty("auto.commit.interval.ms", "500");
```

DISABLE：禁止向Kafka Server提交消费位移。这是在将enable.auto.commit设置为false，或则auto.commit.interval.ms设置为小于或等于0时的默认方式。

```java
FlinkKafkaConsumer<String> myConsumer = new FlinkKafkaConsumer<>(...);

Properties properties = new Properties();

properties.setProperty("enable.auto.commit", "false");
```

### Table & SQL程序

```sql
CREATE TABLE KafkaTable (

 `user_id` BIGINT,

 `item_id` BIGINT,

 `behavior` STRING,

 `ts` TIMESTAMP(3) METADATA FROM 'timestamp'

) WITH (

 'connector' = 'kafka',

 'topic' = 'user_behavior',

 'properties.bootstrap.servers' = 'xxxx:9092',

 'properties.group.id' = '指定消费组名称',

 'scan.startup.mode' = 'earliest-offset',

 'format' = 'csv'

)
```

