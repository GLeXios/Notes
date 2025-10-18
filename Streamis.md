![image-20250409012745730](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409012745730.png)

# Streamis

## BDP和BDAP的划分

我行当前的Hadoop平台分为BDP集群、BDAP集群。

- BDP集群适用于业务核心跑批作业，如银团报表推送、火鸡收益计算等。
- BDAP集群适用于内部报表计算、探索性数据建模等分析类作业。

集群使用原则：

- 涉及到对外输送结果的任务均为核心任务，应放到BDP集群上。
- 分析探索类、监控类的作业应放到BDAP集群上，避免影响核心任务。

## Streamis环境部署信息

### 开发测试环境

| 环境             | 连接方式                                     | 安装目录                                                     | 访问方式                             | JOB发布接口机地址 |
| :--------------- | :------------------------------------------- | :----------------------------------------------------------- | :----------------------------------- | :---------------- |
| DEV开发环境      | 前端：10.107.98.242<br />后端：10.107.98.242 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://dev.streamis.bdap.weoa.com/   | --                |
| SIT测试环境      | 前端：10.107.119.211<br />后端：10.107.97.36 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://sit.dss.bdp.weoa.com/         | --                |
| BDAP-UAT测试环境 | 前端：10.107.119.46<br />后端：10.107.119.46 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://10.107.119.46:8088/           | --                |
| UAT流式主集群    | 前端：172.21.8.221<br />后端：172.21.8.221   | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://uat.streamis.bdp.weoa.com/    | 172.21.8.221      |
| UAT流式备集群    | 前端：10.108.193.83<br />后端：10.108.193.83 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://uat.streamisbak.bdp.weoa.com/ | 10.108.193.83     |

**推荐使用UAT流式主、备流式集群，主集群广州机房，备集群天津机房，可以用于验证双活或主备高可用应用发布**

### 生产集群

#### STREAMIS东莞流式集群 

| 对应MANAGIS集群名 | DSS入口             | 机器                        | IP          | 接口机地址   |
| :---------------- | :------------------ | :-------------------------- | :---------- | :----------- |
| STM-DG-FLOW-PRD   | 10.134.244.132:8088 | xg.bdz.bdpstms040003.webank | 10.130.1.32 | 10.134.224.5 |
| STM-DG-FLOW-PRD   | 10.134.244.132:8088 | xg.bdz.bdpstms040004.webank | 10.130.1.96 | 10.134.232.5 |

#### STREAMIS龙岗流式集群 

| 对应MANAGIS集群名 | DSS 入口           | 机器                        | IP          | 接口机地址     |
| :---------------- | :----------------- | :-------------------------- | :---------- | :------------- |
| STM-LG-FLOW-DR    | 10.170.16.167:8088 | lg.bdz.bdpstms050003.webank | 10.173.1.86 | 10.170.164.162 |
| STM-LG-FLOW-DR    | 10.170.16.167:8088 | lg.bdz.bdpstms050004.webank | 10.173.2.25 | 10.170.16.139  |

#### STREAMIS BDAP流式数仓集群 

| 对应MANAGIS集群名              | DSS入口             | 机器                           | IP             |
| :----------------------------- | :------------------ | :----------------------------- | :------------- |
| LINKIS-DG-BDAP_LHFX-PRD-MAIN-2 | dss.lhfx.webank.com | xg.bdz.bdaplinkis180009.webank | 10.135.20.8    |
| LINKIS-DG-BDAP_LHFX-PRD-MAIN-2 | dss.lhfx.webank.com | xg.bdz.bdaplinkis180009.webank | 10.134.248.143 |

#### STREAMIS 容灾集群 

| 对应MANAGIS集群名 | DSS入口          | 机器                     | IP           |
| :---------------- | :--------------- | :----------------------- | :----------- |
| STM-SH-FLOW-DR    | 10.142.56.8:8088 | sh.bdz.stms110001.webank | 10.142.1.229 |
| STM-SH-FLOW-DR    | 10.142.56.8:8088 | sh.bdz.stms110002.webank | 10.142.1.230 |

#### STREAMIS 准生产集群 

| 对应MANAGIS集群名 | DSS入口          | 机器                        | IP           |
| :---------------- | :--------------- | :-------------------------- | :----------- |
| STM-SH-FLOW-ZSC   | 10.146.4.11:8088 | sh.bdz.bdpstms060001.webank | 10.146.8.13  |
| STM-SH-FLOW-ZSC   | 10.146.4.11:8088 | sh.bdz.bdpstms060002.webank | 10.146.12.11 |



## 故障

​	这个需要麻烦紧急排期处理一下吧，我扛不住了，昨天3次告警，今天一次告警，每次都是  critical，而且这个不光是把磁盘打满，还会把io 打满，我看还有一个mv flink 作业的日志的动作

​	错误码匹配模块在yarn flink失败后，会自动拉取yarn日志分析，生产上的yarn日志一般比较大，这里就拉了54GB下来，占机器磁盘20% ，这里要优化下，避免磁盘拉爆

**问题：** 问题1：近几天出现几次flink集群linkis ecm节点/data盘被flink日志打满的情况，伴随一次ecm oom的问题。原因是运行多天的flink任务因某些原因挂掉，会触发streamis错误码匹配诊断，将flink很大的聚合日志拉取到本地，并且一次读取多行，可能导致本地磁盘写满（流式集群linkis节点上的虚拟机，最大空间1T），或者ecm oom。

**解决：**

```
   1. streamis添加flink yarn聚合日志拉取开关功能，生产关闭所有场景flink yarn聚合日志的拉取   已完成
```

## 问题排查

### Streamis集群不可用的问题

Streamis集群服务都会设置有告警，统一使用zabbix进行告警，主要分两个部分：

- **Streamis进程服务存活性告警**：包括了stms.process.count进程数量检测，通过服务TCP端口${STMS-PORT}进行检查；

  - 如果告警产生了，运维可以直接参考我们给的SOP对Streamis服务进行重拉；

- **GC告警**：**十分钟内full gc次数大于10次、old区占用大于99%**

  - 查看进程的大对象：使用jps查看进程的pid，然后查看进程的对象信息：`jmap -histo $pid`

    ![image-20251019002139542](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002139542.png)

  - 查看服务进程的堆栈信息，判断线程有无block或者死锁:`jstack $pid`

    ![image-20251019002207114](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002207114.png)

  - 查看垃圾回收相关信息：gc次数、gc时间等。`jstat -gcutil $pid`

    ![image-20251019002222018](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002222018.png)

**直接上集群机器上查相关日志，看有没有相关报错**，可能是由于之前版本发布导致的一些问题。

**还有一些其他的告警：**

- 流式集群磁盘容量监控。之前也有出过问题：错误码匹配模块在yarn flink失败后，会自动拉取yarn日志分析，生产上的yarn日志一般比较大，这里就拉了54GB下来，占机器磁盘20% ，这里要优化下，避免磁盘拉爆

### 用户的flink应用无法启动或者运行失败

主要是查日志：

- 查yarn上的flink日志，一般大部分问题都出在这里，这里可能有：
  - 用户自己的业务代码有问题；
  - yarn队列资源不足，这个一般是最常见的，测试环境需要联系部门其他同事使用相同队列的应用退出，然后再协商进行应用测试；生产环境一般就会进行集群扩容操作。
  - 不合规，队列使用错误
- 查linkis的flink引擎日志，这个跟linkis起ec的过程相关
- linkis集群问题，这个需要去eureka上看机器的地址，然后去对应地址找linkis 的jobmanager日志、bml等日志，进一步排查

### 如何判断streamis的flink应用是否运转良好

【**检查 Flink 应用的运行状态**】

Flink 应用的状态是判断其是否正常运行的核心指标。

**1）查看 JobManager 和 TaskManager 状态**

- **JobManager** 是 Flink 集群的控制中心，负责调度任务。
- **TaskManager** 是实际执行任务的工作节点。

使用 Flink Web UI（默认端口为 8081）查看集群状态。在 Overview 页面中确认：

- JobManager 是否正常运行。
- TaskManager 数量是否达到预期。
- 各个 TaskManager 的资源使用情况（如内存、CPU）。

**2）检查任务状态**

在 Flink Web UI 中查看任务的状态：

- **RUNNING** ：任务正在正常运行。
- **FINISHED** ：任务已完成（批处理任务可能进入此状态）。
- **FAILED** ：任务失败，需要排查错误。
- **CANCELED** ：任务被手动取消

【**监控性能指标**】

Flink 提供了丰富的内置指标，可以通过以下方式获取：

- **Flink Web UI** ：在任务详情页面中查看吞吐量、延迟、背压等指标。
- **Prometheus + Grafana** ：配置 Prometheus 抓取 Flink 指标，并通过 Grafana 可视化。

**关键性能指标**

- **吞吐量（Throughput）** ：
  - 表示每秒处理的数据量。
  - 如果吞吐量突然下降，可能是数据源或下游系统出现问题。
- **延迟（Latency）** ：
  - 表示数据从进入系统到被处理完成的时间。
  - 高延迟可能表明任务执行缓慢或存在背压。
- **背压（Backpressure）** ：
  - 背压表示下游算子无法及时处理上游发送的数据。
  - 在 Web UI 中，如果某个算子显示高背压（如红色），需要优化数据流或增加资源。
- **Checkpoint** ：
  - 检查 Checkpoint 的成功率和耗时。
  - 如果 Checkpoint 失败或耗时过长，可能会影响任务的容错能力。

【**检查日志输出**】

日志是诊断问题的重要来源，能够帮助定位异常和性能瓶颈。

- **JobManager 日志** ：
  - 记录任务调度和全局状态。
  - 检查是否有异常堆栈信息。
- **TaskManager 日志** ：
  - 记录任务执行过程中的详细信息。
  - 关注是否有 `OutOfMemoryError` 或其他异常。

【**常见问题及解决方案**】

**1）任务失败**

- **原因** ：代码错误、资源不足、外部系统不可用。
- **解决** 
  - 检查日志定位问题。
  - 增加 TaskManager 的资源（如内存、CPU）。

**2）数据积压**

- **原因** ：数据源生产速度高于消费速度。
- **解决** 
  - 增加 Flink 任务的并行度。
  - 优化数据处理逻辑，减少单条记录的处理时间。

**3）Checkpoint 失败**

- **原因** ：存储系统（如 HDFS）不可用或网络延迟高。
- 解决 
  - 检查存储系统的状态。
  - 增加 Checkpoint 的超时时间。

### flink节点延迟

​	**节点延迟（Node Latency）**通常指的是数据从一个算子（Operator）到另一个算子所需的时间。这个延迟可能受到多种因素的影响，包括网络传输、处理逻辑复杂度、反压（backpressure）、资源不足等。

**【Flink 节点延迟的常见表现】**

1. Event Time 延迟高
   - 水位线（Watermark）滞后，导致事件时间窗口迟迟不能触发。
2. Processing Time 延迟高
   - 数据到达算子后，处理时间过长，导致整体吞吐下降。
3. Checkpointing 耗时增加
   - Checkpoint 时间变长，可能导致检查点失败或频繁超时。
4. Subtask 处理慢
   - 某些 subtask 明显比其他 subtask 慢，形成瓶颈。

【**常见的延迟原因分析**】

**1）反压（Backpressure）**

- 当下游算子处理不过来数据时，会向上游发送反压信号，导致上游积压。
- 表现为某些 subtask 的输入缓冲区满，处理速度降低。
- 可通过 Web UI 查看背压状态（Backpressure Status）。

**2）热点数据/倾斜（Data Skew）**

- 某个 key 或部分 key 的数据量远高于其他 key，导致对应 subtask 负载过高。
- 表现为部分 subtask 长时间运行，其他 subtask 已完成任务。

**3）资源不足**

- CPU、内存、磁盘 IO 不足会导致处理能力下降。
- 如果 TaskManager 内存不足，可能发生频繁 GC 或 OOM。

**4）外部系统性能瓶颈**

- 如 Kafka 消费慢、数据库写入慢、HDFS 写入慢等。
- Sink 算子成为瓶颈，影响整个流水线。

【**排查工具和方法**】

**1）Flink Web UI**

- 查看每个 subtask 的累加器指标（如处理记录数、buffer使用情况）
- 查看反压状态（Backpressure）
- 查看任务执行时间、GC 情况、CPU 利用率等
- 查看 checkpoint 和 state 相关信息

**2）Metrics System**

启用并集成 Prometheus + Grafana，可以监控更多细粒度指标

**【常见解决策略】**

**✅ 优化数据分区**

- 检查是否 keyby 导致数据倾斜
- 可尝试预分区、打散 key、局部聚合 + 全局聚合（两阶段聚合）

**✅ 提升资源**

- 增加 TaskManager 数量
- 提高 slot 数量
- 增大堆内存或调整垃圾回收参数

**✅ 减少状态操作开销**

- 避免频繁读写状态
- 使用更高效的状态结构（如 MapState / ListState）

**✅ 异步 IO 写外部系统**

- 使用 Async I/O 写 DB 或远程服务

### OOM排查

#### 问题背景

​	在近期测试环境中，应用进程频繁被 `OutOfMemoryError`（OOM）异常触发的 JVM 机制强制终止，而生产环境从未发生类似问题。

![image-20250514151546545](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151546545.png)

初步怀疑可能由以下原因导致：

1. **JVM 参数配置差异** ：生产环境可能分配了更大的堆内存或更合理的 GC 策略。
2. **代码逻辑差异** ：测试环境可能引入了特定的测试组件或配置。
3. **依赖组件行为异常** ：某些仅在测试环境启用的组件存在内存泄漏。

#### 问题排查流程

**（1）定位堆栈文件**

- **JVM 参数配置** ：通过 `-XX:+HeapDumpOnOutOfMemoryError` 和 `-XX:HeapDumpPath` 参数定位堆栈文件（`.hprof`）的生成路径。
  - 或者定位到了虚拟机的日志的输出目录，用简单的命令寻找到堆栈文件：![image-20250514151755128](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151755128.png)
  - ![image-20250514151810247](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151810247.png)
- **容器环境限制** ：因测试环境部署在容器中，需通过 **容器 → 跳板机 → SFTP 服务器 → 本地开发机** 的链路下载堆栈文件

**（2）堆栈文件分析**

![image-20251019002336647](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002336647.png)

​	通过 jvisualvm.exe 应用 文件 -> 装入 的方式导入刚才下载的文件，载入之后会有这样的选项卡

- 在概要的地方就给出了出问题的关键线程：发现 OOM 原因为 **堆内存溢出（Java heap space）** ，将堆栈信息点进去，发现堆栈是由于字符串的复制操作导致发生了 oom。

- 同时通过堆栈信息可以定位到 UnitTestContextHolder.java 类

  ![image-20251019002355642](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002355642.png)

**【类分析】**

​	通过类分析视图，我们可以简单的定位到占据内存最高的类。**这里可以按照 实例数 或者 内存大小进行排序**，查询是否有某个类的实例数量异常，比如远远的高于其他的类，通常这些异常值就是可能发生内存泄露的地方。

- 我们这里由于 char 类实例数比较高，所有我优先往内存泄漏方向思考。

![image-20251019002415059](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002415059.png)

**【实例数分析】**

![image-20251019002426195](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002426195.png)

发现全是这种字符的实例，选中【实例】，右键可以分析【最近的垃圾回收节点】

这里会显示这些对象为何没有被回收，一般是静态变量或者是全局对象引用导致。

![image-20251019002447769](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002447769.png)

通过上面图片可以看出，图中主要是由静态字段导致

**（3）代码分析**

​	代码是发生在项目中引入的企同的试点单元测试组件，刚好我们项目参与了试点，由于是单元测试组件，只在编译或者测试的依赖包中引入，正式的未引入

![image-20250514153410531](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153410531.png)

![image-20251019002503422](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002503422.png)

这里使用到了 ThreadLocal 做一些对象的存储，但是我发现 调用 setAttribute(String key,Object value)的地方有 33 处

![image-20251019002512934](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002512934.png)

但是调用 clear()的地方只有3处。正常来说凡是使用 ThreadLocal 的地方，set 和 clear() 都应该成对出现。

![image-20250514153516478](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153516478.png)

**（4）原因分析**

ThreadLocal 内存泄漏 

- **`ThreadLocal` 的 `set` 方法未配对调用 `clear`，线程池复用时残留数据未释放，导致线程复用时累积大量 `char[]` 实例（如字符串拼接、缓存数据等）。**
- **静态变量引用进一步延长对象生命周期，导致 `char[]` 实例无法被 GC 回收。**

**测试环境特性** 

- **该组件仅在测试环境的依赖中引入（如 `test` 或 `provided` 作用域），生产环境未启用。**
- 测试场景高频触发组件逻辑，加速内存堆积。

**（5）解决方法**

​	联系组件提供方，关闭我们应用该组件的使用，重新观察是否又再次发生 OOM，之后也确实没有再发生 oom，应用不断的挂掉也基本上没有再出现。

#### 问题总结

一般来说排查 OOM一般通过几个方向考虑

- 堆内存不足，一般原因为对象无法被回收(通俗说的内存泄漏)、加载了过大的数据(比如同一时间读取了大量的数据库、加载了大文件)、大量缓存未清理(hashMap做本地缓存，没有过期逻辑)、使用ThreadLocal未及时清理
- 虚拟机参数设置不合理： 堆内存过小、GC 策略不匹配（如 G1 回收器未启用）。
- 线程创建不合理： 无限增长的线程池、线程阻塞未释放资源。

以上是一些场景的原因，最直接的分析方式还是取出堆栈文件，仔细分析



## Streamis与Linkis的通信

Streamis将自身作为微服务注册到Linkis并通过Linkis的SDK调用服务时，底层通信机制主要基于**Feign实现的双向RPC通信方案**

Streamis通过Linkis提供的SDK集成其通信模块，利用Feign客户端将自身服务注册到Linkis的微服务治理体系中，并**通过统一网关（Gateway）实现请求转发和负载均衡**

- 例如，当Streamis需要调用Linkis Flink引擎时，底层通过Feign发起RPC请求，结合Eureka的服务发现完成跨微服务调用

【**发起服务调用流程**】

直接通过注入的Feign客户端调用服务，SDK会自动完成以下操作：

1. **服务发现** ：从Eureka获取目标服务实例地址
2. **请求转发** ：通过Linkis网关（Gateway）路由到对应微服务
3. **双向通信** ：底层基于Feign的RPC机制实现请求-响应交互

【**服务注册与发现** 】

- Linkis的微服务治理架构采用**Eureka** 作为服务注册中心，Streamis作为微服务通过SDK向Eureka注册自身服务，并通过服务发现机制定位其他微服务

【**通信协议与实现**】 Linkis基于**Feign** （是一个http请求调用的轻量级框架，一种声明式REST客户端）实现了一套自定义的RPC通信方案。该方案允许微服务通过SDK以双向RPC的方式发起请求或响应请求，支持服务间的高效通信。具体来说：

- **Sender（发送器）** ：作为客户端接口，负责向其他服务发送请求。
- **Receiver（接收器）** ：作为服务端接口，负责接收并处理请求。
- **Interceptor（拦截器）** ：Sender发送器会将使用者的请求传递给拦截器。拦截器拦截请求，对请求做额外的功能性处理。
  - 分别是广播拦截器用于对请求广播操作
  - 重试拦截器用于对失败请求重试处理
  - 缓存拦截器用于简单不变的请求读取缓存处理
  - 和提供默认实现的默认拦截器



**Linkis RPC模块**

​	基于Feign的微服务之间HTTP接口调用，**开发者可以像调用本地方法一样发起HTTP请求**，**只能满足简单的A微服务实例根据简单的规则随机访问B微服务之中的某个服务实例，而这个B微服务实例如果想异步回传信息给调用方，是根本无法实现的**。 同时，由于Feign只支持简单的服务选取规则，无法做到将请求转发给指定的微服务实例，无法做到将一个请求广播给接收方微服务的所有实例。 **Linkis RPC是基于Spring Cloud + Feign实现的一套微服务间的异步请求和广播通信服务**，可以独立于Linkis而使用。

- Linkis RPC作为底层的通信方案，将提供SDK集成到有需要的微服务之中。 一个微服务既可以作为请求调用方，也可以作为请求接收方。
- 作为请求调用方时，将通过Sender请求目标接收方微服务的Receiver，作为请求接收方时，将提供Receiver用来处理请求接收方Sender发送过来的请求，以便完成同步响应或异步响应。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/RPC-01-140fa51c0efdc3cd71c9fb47dcafd4b1.png)

【**Linkis如何通过Feign实现RPC**】

1. **接口定义与代理** ： Linkis通过Feign的`@FeignClient`定义服务接口，生成动态代理对象，将方法调用转换为HTTP请求
2. **服务注册与发现** ： 服务启动时向Eureka注册，调用方通过服务名（如`linkis-engineconn-service`）动态发现目标服务地址
3. **双向通信封装** ： Linkis在Feign基础上增加了`Sender`和`Receiver`体系，允许服务主动接收并处理请求，形成双向交互能力
4. **拦截器与治理** ： Linkis在Feign的调用链中插入拦截器，实现权限校验、日志记录等治理功能

【 **Linkis RPC与Feign的核心区别**】

| **特性**     | **原生FEIGN**                    | **LINKIS RPC**                           |
| :----------- | :------------------------------- | :--------------------------------------- |
| **通信协议** | 纯HTTP协议（RESTful API）        | 基于HTTP的封装，但对外暴露为RPC接口      |
| **服务角色** | 单向调用（客户端→服务端）        | 双向调用（服务可同时作为调用方和提供方） |
| **扩展功能** | 依赖Spring Cloud生态（如Ribbon） | 内置拦截器、状态管理等治理功能           |



## Linkis Flink 引擎使用

### 引擎验证

在执行 `Flink` 任务之前，检查下执行用户的这些环境变量。具体方式是

```
sudo su - ${username}
echo ${JAVA_HOME}
echo ${FLINK_HOME
```

| 环境变量名      | 环境变量内容   | 备注                                                         |
| :-------------- | :------------- | :----------------------------------------------------------- |
| JAVA_HOME       | JDK安装路径    | 必须                                                         |
| HADOOP_HOME     | Hadoop安装路径 | 必须                                                         |
| HADOOP_CONF_DIR | Hadoop配置路径 | Linkis启动Flink引擎采用的Flink on yarn的模式,所以需要yarn的支持。 |
| FLINK_HOME      | Flink安装路径  | 必须                                                         |
| FLINK_CONF_DIR  | Flink配置路径  | 必须,如 ${FLINK_HOME}/conf                                   |
| FLINK_LIB_DIR   | Flink包路径    | 必须,${FLINK_HOME}/lib                                       |

### Flink引擎的使用

`Linkis` 的 `Flink` 引擎是通过 `flink on yarn` 的方式进行启动的,所以需要指定用户使用的队列，如下图所示

![image-20251019002643513](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002643513.png)

#### 通过 `Linkis-cli` 提交任务（Shell方式）

```
sh ./bin/linkis-cli -engineType flink-1.12.2 \
-codeType sql -code "show tables"  \
-submitUser hadoop -proxyUser hadoop
```

#### 通过 `OnceEngineConn` 提交任务

​	`OnceEngineConn` 的使用方式是用于正式启动 `Flink` 的流式应用,具体的是**通过 `LinkisManagerClient` 调用 `LinkisManager` `的createEngineConn` 的接口，并将代码发给创建的 `Flink` 引擎，然后 `Flink` 引擎就开始执行，此方式可以被其他系统进行调用，比如 `Streamis`** 。 `Client` 的使用方式也很简单，首先新建一个 `maven` 项目，或者在您的项目中引入以下的依赖。

```
<dependency>
    <groupId>org.apache.linkis</groupId>
    <artifactId>linkis-computation-client</artifactId>
    <version>${linkis.version}</version>
</dependency>
```

然后新建 `scala` 测试文件,点击执行，就完成了从一个 `binlog` 数据进行解析并插入到另一个 `mysql` 数据库的表中。但是需要注意的是，您必须要在 `maven` 项目中新建一个 `resources` 目录，放置一个 `linkis.properties` 文件，并指定 `linkis` 的 `gateway` 地址和 `api` 版本，如

```
wds.linkis.server.version=v1
wds.linkis.gateway.url=http://ip:9001/
object OnceJobTest {
  def main(args: Array[String]): Unit = {
    val sql = """CREATE TABLE mysql_binlog (
                | id INT NOT NULL,
                | name STRING,
                | age INT
                |) WITH (
                | 'connector' = 'mysql-cdc',
                | 'hostname' = 'ip',
                | 'port' = 'port',
                | 'username' = '${username}',
                | 'password' = '${password}',
                | 'database-name' = '${database}',
                | 'table-name' = '${tablename}',
                | 'debezium.snapshot.locking.mode' = 'none'
                |);
                |CREATE TABLE sink_table (
                | id INT NOT NULL,
                | name STRING,
                | age INT,
                | primary key(id) not enforced
                |) WITH (
                |  'connector' = 'jdbc',
                |  'url' = 'jdbc:mysql://${ip}:port/${database}',
                |  'table-name' = '${tablename}',
                |  'driver' = 'com.mysql.jdbc.Driver',
                |  'username' = '${username}',
                |  'password' = '${password}'
                |);
                |INSERT INTO sink_table SELECT id, name, age FROM mysql_binlog;
                |""".stripMargin
    val onceJob = SimpleOnceJob.builder().setCreateService("Flink-Test").addLabel(LabelKeyUtils.ENGINE_TYPE_LABEL_KEY, "flink-1.12.2")
      .addLabel(LabelKeyUtils.USER_CREATOR_LABEL_KEY, "hadoop-Streamis").addLabel(LabelKeyUtils.ENGINE_CONN_MODE_LABEL_KEY, "once")
      .addStartupParam(Configuration.IS_TEST_MODE.key, true)
      //    .addStartupParam("label." + LabelKeyConstant.CODE_TYPE_KEY, "sql")
      .setMaxSubmitTime(300000)
      .addExecuteUser("hadoop").addJobContent("runType", "sql").addJobContent("code", sql).addSource("jobName", "OnceJobTest")
      .build()
    onceJob.submit()
    println(onceJob.getId)
    onceJob.waitForCompleted()
    System.exit(0)
  }
}
```

## 安全认证（Token）

### 实现逻辑介绍

通过统一的认证处理filter:`org.apache.linkis.server.security.SecurityFilter` 来控制

- 可用的token以及对应可使用的ip相关信息数据存储在表`linkis_mg_gateway_auth_token`中， 详细见[表解析说明](https://linkis.apache.org/zh-CN/docs/latest/development/table/all#16-linkis_mg_gateway_auth_token)，非实时更新， 会定期`wds.linkis.token.cache.expire.hour`(默认间隔12小时)刷新到服务内存中

```
val TOKEN_KEY = "Token-Code"
val TOKEN_USER_KEY = "Token-User"

/* TokenAuthentication.isTokenRequest 通过判断请求request中：
     1.请求头是否包含TOKEN_KEY和TOKEN_USER_KEY :getHeaders.containsKey(TOKEN_KEY) && getHeaders.containsKey(TOKEN_USER_KEY)
     2.或则请求cookies中是否包含TOKEN_KEY和TOKEN_USER_KEY:getCookies.containsKey(TOKEN_KEY) &&getCookies.containsKey(TOKEN_USER_KEY)
*/

if (TokenAuthentication.isTokenRequest(gatewayContext)) {
      /* 进行token认证 
        1. 确认是否开启token认证 配置项 `wds.linkis.gateway.conf.enable.token.auth`
        2. 提取token tokenUser host信息进行认证，校验合法性
      */
      TokenAuthentication.tokenAuth(gatewayContext)
    } else {
    //普通的用户名密码认证    
}
```

### 使用方式

#### 原生的方式

构建的http请求方式，需要在请求头中添加`Token-Code`,`Token-User`参数,

请求地址: `http://127.0.0.1:9001/api/rest_j/v1/entrance/submit`

body参数:

```
{
    "executionContent": {"code": "sleep 5s;echo pwd", "runType":  "shell"},
    "params": {"variable": {}, "configuration": {}},
    "source":  {"scriptPath": "file:///mnt/bdp/hadoop/1.hql"},
    "labels": {
        "engineType": "shell-1",
        "userCreator": "hadoop-IDE",
        "executeOnce":"false "
    }
}
```

请求头header:

```
Content-Type:application/json
Token-Code:BML-AUTH
Token-User:hadoop
```

#### 客户端使用token认证

linkis 提供的客户端认证方式都支持Token策略模式`new TokenAuthenticationStrategy()`

详细可以参考[SDK 方式](https://linkis.apache.org/zh-CN/docs/latest/user-guide/sdk-manual)

**示例**

```
// 1. build config: linkis gateway url
 DWSClientConfig clientConfig = ((DWSClientConfigBuilder) (DWSClientConfigBuilder.newBuilder()
        .addServerUrl("http://127.0.0.1:9001/")   //set linkis-mg-gateway url: http://{ip}:{port}
        .connectionTimeout(30000)   //connectionTimeOut
        .discoveryEnabled(false) //disable discovery
        .discoveryFrequency(1, TimeUnit.MINUTES)  // discovery frequency
        .loadbalancerEnabled(true)  // enable loadbalance
        .maxConnectionSize(5)   // set max Connection
        .retryEnabled(false) // set retry
        .readTimeout(30000)  //set read timeout
        .setAuthenticationStrategy(new TokenAuthenticationStrategy()) // AuthenticationStrategy Linkis auth Token
        .setAuthTokenKey("Token-Code")  // set token key
        .setAuthTokenValue("DSM-AUTH") // set token value
        .setDWSVersion("v1") //linkis rest version v1
        .build();
```

### token的配置

​	支持的token，对应的可用的用户/可使用请求方ip 是通过表`linkis_mg_gateway_auth_token`来控制，**加载是非实时更新，使用了缓存机制**

## Flink流式计算业务现状

![image-20251019002733227](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002733227.png)

![image-20250409163819159](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163819159.png)

![image-20250409163900622](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163900622.png)

![image-20250409163918570](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163918570.png)



## Steamis已有能力现状

![image-20251019002753035](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002753035.png)

## Streamis简介



**Streamis 是流式应用运维管理系统**

​	基于 DataSphere Studio 的框架化能力，以及底层对接 Linkis 的 Flink 引擎，让用户低成本完成流式应用的开发、发布和生产管理。

**【核心特点】**

**（1）基于 DSS 和 Linkis的流式应用运维管理系统。**       以 Flink 为底层计算引擎，基于开发中心和生产中心隔离的架构设计模式，完全隔离开发权限与发布权限，隔离开发环境与生产环境，保证业务应用的高稳定性和高安全性。

​	应用执行层集成 Linkis 计算中间件，打造金融级具备高并发、高可用、多租户隔离和资源管控等能力的流式应用管理能力。

**（2）流式应用生产中心能力。**       支持流式作业的多版本管理、全生命周期管理、监控告警、checkpoint 和 savepoint 管理能力。

**【依赖的生态组件】**

![image-20251019002820894](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002820894.png)

**【Streamis功能介绍】**

![image-20251019002830102](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002830102.png)

**【Streamis架构】** 

![image-20251019002934097](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002934097.png)

## 痛点、解决措施

### 痛点一：开发成本高且难以复用

​	针对这个痛点，我们引入了meta.json模板，进行模板式开发，减少重复造轮子的动作。

**【meta.json模版】**

​	用户在本地开发Flink程序后打包成Flink jar形式，作为flink的主jar包，Meta.json是任务的元数据信息，是一个多层级的json文件，记录了该job需要用到的一系列信息：比如项目名projectName、作业名jobName、并行度parallelism和其他job生产参数等信息，刚刚提到的flink的主jar包就可以填入meta.json文件的对应字段下：

- 用户通过标准模板填入必需的参数，而不需要的参数就可以直接删去，因此可以在保证规范的同时拥有一定的定制自由，能够大大降低开发的成本。
- 用户通过把meta.json和相关资源文件打入一个zip包上传至Streamis的服务器，Streamis就可以自动将相关信息拆解后形成一个job，相关资源文件则通过对接linkis到bml物料管理系统（实际使用HDFS文件进行数据存储）进行持久化。

![flinkjar_metajson](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar_metajson.png)

meta.json是StreamisJob的元数据信息，其格式为：

- 主要包括flink.resource：flink作业的资源配置，比如Parallelism并行度、JobManager和TaskManager Memory、yarn队列等
- flink.custom：flink应用的配置，包括kerberos权限keytab，log的过滤配置等；
- flink.produce：streamis对于flink作业管理等配置，包括Checkpoint开关、作业失败自动重拉机制开关、业务产品名称、高可用策略等

```
{
  "projectName": "", // 项目名
  "jobName": "",  // 作业名
  "jobType": "flink.sql",   // 目前只支持flink.sql、flink.jar
  "tags": "",   // 应用标签
  "description": ""    // 作业描述,
  "jobContent": {
    // 不同的jobType，其内容各不相同，具体请往下看
  },
  "jobConfig": {
      "wds.linkis.flink.resource": {
            "wds.linkis.flink.app.parallelism":"",  // Parallelism并行度
            "wds.linkis.flink.jobmanager.memory":"",  // JobManager Memory(M)
            "wds.linkis.flink.taskmanager.memory":"", // TaskManager Memory(M)
            "wds.linkis.flink.taskmanager.numberOfTaskSlots":"",  // TaskManager Slot数量
            "wds.linkis.flink.taskmanager.cpus":"",  // TaskManager CPUs
            "wds.linkis.rm.yarnqueue":""  // Yarn队列
            "flink.client.memory":""  // 客户端内存
      },
      "wds.linkis.flink.custom": {
          // Flink作业相关参数
            "stream.log.filter.keywords":"ERROR,WARN,INFO",
            "security.kerberos.krb5-conf.path":"",
            "demo.log.tps.rate":"20000",
            "classloader.resolve-order":"parent-first",
            "stream.log.debug":"true",
            "security.kerberos.login.contexts":"",
            "security.kerberos.login.keytab":"",
            "security.kerberos.login.principal":""
      },
      "wds.linkis.flink.produce": {
            "wds.linkis.flink.checkpoint.switch":"OFF",  // Checkpoint开关
            "wds.linkis.flink.app.fail-restart.switch":"OFF",  // 作业失败自动拉起开关
            "wds.linkis.flink.app.start-auto-restore.switch":"OFF"  // 作业启动状态自恢复
            "wds.linkis.flink.product": "testRestart"   // 业务产品名称，用于回调日志路径生成，可以不填，默认为空
            "wds.streamis.app.highavailable.policy": "single"   // 高可用策略，如single，double，managerSlave等，默认为single
            "wds.linkis.flink.alert.failure.user": "gelxiogong,alexyang"  //  任务失败告警人，必填
            "linkis.ec.app.manage.mode": "detach"  // ec管理模式，分为detach和attach
      }
    }
}
```

### 痛点二：维护成本高

第二个痛点就是flink作业的维护成本高，无法对flink作业的生命周期做一个统一的维护管理

**Streamis针对这个问题**：

**【首先是对整个项目的管理】**

- streamis能够对资源的上限进行管理配置，比如yarn队列的名称、个数、队列内存CPU等。
- 通过DSS的代理用户实现对项目权限的管控：
  - 实名用户必须通过代理用户才能登录系统。
  - 只有代理用户比如hduser01拥有对应的集群操作权限，系统通过通过代理用户来实现权限的隔离与分配，实名用户是没有相关权限的。
- 通过审计日志查看入口来观察最近的每一个接口调用信息，包括其入参出参和操作类型等。
- 对项目物料的管理：物料分为job级别的物料和项目工程级别的物料
  - job级别的物料随着job的导入持久化在物料服务，且与其他job的物料隔离；
  - 项目工程级别的物料则可以通过工程物料导入接口进行上传，整个项目的job作业都能共享这个物料
  - 加载物料的顺序为：先加载job级别的物料，若没有则加载项目工程级别的物料

![企业微信截图_17425468956360](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17425468956360.png)

![image-20251019002959912](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019002959912.png)

**【维护一个flink应用的完整生命周期】**

​	streamis维护了一个作业job的完整生命周期，包括其通过meta.json导入任务、启停、批量启停、启用禁用、查找、标签管理、快照、作业查看和配置查看编辑等功能。

- 每新上传一个作业就会用新作业覆盖旧作业，同时后面的版本号+1，用户点击版本号就能看到历史所有版本作业的信息，包括这个作业的执行历史、任务详情、依赖的jar包等信息；
- job启动前会有一个inspect界面，能够观察到job版本、快照和一致性检查结果等信息，如果有某项有问题会通过反色和文字提醒可能的异常。
- 每一次job启动都会记录到运行记录里进行查看，可以观察停止失败的原因；
- 每一次告警也会记录在告警页面里供查阅
- 这里的重点在于作业的启停其中相关资源管理的难题，这里放在痛点三里讲述。

![image-20250321163422704](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321163422704.png)

![image-20251019003021454](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003021454.png)

**【对flink作业的配置】**

​	这个flink配置基本上跟之前提到过的meta.json是一一对应的，即用户在meta.json中填写的相关参数通过配置界面进行可视化，用户可以通过配置界面对应用进行参数配置。

- 主要包括flink.resource：flink作业的资源配置，比如Parallelism并行度、JobManager和TaskManager Memory、yarn队列等
- flink.custom：flink应用的配置，包括kerberos权限keytab，log的过滤配置等；
- flink.produce：streamis对于flink作业管理等配置，包括Checkpoint开关、作业失败自动重拉机制开关、业务产品名称、高可用策略等

​	然后在设置了关键字日志回调和关键字告警之后，就能够通过ims对比如WARN、ERROR关键字进行告警。除了应用的配置，还能够收集应用的指标和执行日志提供给用户。

### 痛点三：资源管理难

#### ec介绍、一个ec占用多少资源、释放了什么资源

**【ec介绍】**

- **EngineConn微服务**：**EngineConn 是一个独立的 JVM 进程** ，其核心职责是作为 **"连接器"** 桥接 Linkis 上层服务与底层计算引擎（如 Spark、Flink、Hive 等）。
  - **微服务架构** ：指包含了一个EngineConn、一个或多个Executor，用于对计算任务提供计算能力的实际微服务，拥有专属的 JVM 内存空间和线程池。我们说的新增一个EngineConn，其实指的就是新增一个EngineConn微服务。
  - **生命周期** ：由 ECM（EngineConnManager）通过启动脚本触发，通过 `java -jar` 命令启动，进程结束时由 ECM 或 LinkisManager 统一回收资源
- **EngineConn**：引擎连接器，是与底层计算存储引擎的实际连接单元，包含了与实际引擎的会话信息。它与Executor的差别，是EngineConn只是起到一个连接、一个客户端的作用，并不真正的去执行计算。如SparkEngineConn，其会话信息为SparkSession。
- **Executor**：执行器，作为真正的计算存储场景执行器，是实际的计算存储逻辑执行单元，对EngineConn各种能力的具体抽象，提供交互式执行、订阅式执行、响应式执行等多种不同的架构能力。

**【EngineConn微服务的初始化一般分为三个阶段】**

**阶段1：EngineConn 启动**

- **入口方法** ：通过 `main` 方法启动，解析命令行参数生成 `EngineCreationContext`，包含引擎类型、标签、资源需求等元数据。
- **引擎连接建立** ：通过EngineCreationContext初始化EngineConn，完成EngineConn与底层Engine的连接建立。
  - **示例（Spark）** ：初始化 `SparkSession`，与底层 Spark 集群建立会话；
  - **示例（Flink）** ：创建 Flink Client，连接到 YARN/K8s 集群并启动 JobManage

**阶段2：Executor 初始化**

- EngineConn初始化之后，接下来会根据实际的使用场景，初始化对应的Executor，为接下来的用户使用，提供服务能力。
  - 比如：交互式计算场景的SparkEngineConn，会初始化一系列可以用于提交执行SQL、PySpark、Scala代码能力的Executor，支持Client往该SparkEngineConn提交执行SQL、PySpark、Scala等代码
- **Executor 类型** ：根据场景初始化不同执行器，如：
  - `ComputationExecutor`：交互式任务（SQL、PySpark）。
  - `OnceExecutor`：一次性任务（批处理作业）。
  - `FlinkStreamExecutor`：流式任务

**阶段3：心跳与退出**

**心跳机制** ：

- 定时向 LinkisManager 汇报心跳上报状态（如负载、健康度），维持微服务注册信息，并等待EngineConn结束退出。

**退出条件** ：

- 底层引擎异常（如 Spark Application 失败）；
- 超过最大空闲时间（默认 10 分钟）。
- Executor执行完成、或是用户手动kill时

**【释放了什么资源，JVM 进程的资源特征】**

​	EC 是一个 Java 进程，主要用于作业提交和管理，作为 **"连接器"** 桥接 Linkis 上层服务与底层计算引擎（如 Spark、Flink、Hive 等）。

**内存模型** ：

- **堆内存** ：用于存储会话元数据（如 Spark 的 `SparkSession` 配置）和任务上下文。
- **堆外内存** ：部分引擎（如 Flink）可能直接分配堆外内存以减少 GC 压力

**线程模型** ：

- **主线程**：负责心跳上报和资源监控。
- **工作线程池**：由 Executor 使用，执行具体任务（如 SQL 解析、数据计算）

**释放的计算机资源包括：**

- **内存资源**：EC 进程在 JVM 中运行，占用堆内存和非堆内存，用于存储作业状态、配置信息和通信缓存。退出后，这些内存资源被释放。
- **CPU 资源**：EC 进程需要定期执行心跳、状态更新和任务查询等操作，占用 CPU 时间。进程退出后，CPU 资源得以释放。
- **网络资源**：EC 维持与 Flink 集群、Linkis Manager 和 ECM 之间的网络连接，用于状态同步和任务调度。退出后，网络连接关闭，释放网络带宽。

**将 EC 的资源占用从每个作业独占（O(n)）转变为通过管理节点共享（O(1)），显著降低了系统资源压力**

**【一个ec占用多少资源】**

- 一个 EC 的内存需求通常在 512MB 到 1GB 之间。
- EC 进程通常占用 1-2 个 CPU 核心，用于执行心跳和状态更新等操作。

​	这里是可以通过linkis管理台看见的，ec引擎占用的资源较小，但是由于生产上的流式Flink应用有差不多1200个，所以总的下来能够节省不少内存和cpu的开销。

​	**flink作业启动的时候还是复用之前的flink ec，只是在flink应用启动后就会直接释放资源，然后在启动的时候会起一个特殊的flink ec：flink manager ec，用来管理flink应用的专属操作比如说stop，savepoint，checkpoint之类的**

**【ecm的资源】**

ecm资源也跟ec差不多，可以从linkis管理台进行管理，决定起多少ecm

![image-20250410001904370](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250410001904370-4602709.png)



#### 分离式引擎简介

​	目前生产的流式作业数量众多，其中Flink作业的驱动执行交由Linkis Flink EngineConn处理，在Flink引擎中，常驻维护了一个任务client，占用了机器内存资源；为了能支持更多的流式作业执行，需要将client占用的资源释放，即使用cluster管理模式去维护执行中的Flink作业。

​	就是说我们可以采用一种分离式架构detach，使flinks EC在flink应用初始化成功后，EC进程退出，同时保证flink应用能够正常运行。因此分离式EC架构可以释放本地客户端资源，而且可以减少流式Linkis集群ECM节点资源：O(n) -> O(1)，达到降本增效的结果。

1）**EC** - linkis的引擎连接器EngineConn，一般为封装各底层集群的client，可以向底层集群提交、查询任务，同时向RM汇报负载信息和自身的健康状况

2）**EngineConnManager（ECM）**：Linkis 的EC进程管理服务，负责管控EngineConn的生命周期（启动、停止）

3）**ATTACH模式** - linkis flink EngineConn引擎管理模式，指作业通过streamis Launcher驱动器提交至Linkis后，对于作业任务的所有操作都连接至linkis的EngineConn引擎处理，EngineConn会常驻一个作业任务客户端，这种管理模式下streamis的客户端与EngineConn点对点连接，类型归为ATTACH（不可以分离/强绑定），EngineConn存活性影响整个作业。

4）**DETACH模式** - 对应linkis两种flink引擎的executor：分离式DETACH executor和管理manager executor。作业同样是通过Launcher驱动器提交至Linkis，但对于DETACH executor linkis EngineConn在作业成功提交后即刻关闭释放；关于flink应用的操作，由管理节点响应。客户端提交类型为DETACH（可分离）。

5）**MANAGER模式** - linkis flink manager EngineConn，属于flink EngineConn中交互式executor的实现。该executor接受flink应用的applicationId，可以附加client，对yarn集群上应用进行操作。

6）**LinkisManager**作为Linkis的一个独立微服务，对外提供了AppManager（应用管理）、ResourceManager（资源管理）、LabelManager（标签管理）的能力，能够支持多活部署，具备高可用、易扩展的特性。

类似IO多路复用的思想，通过一个ec manager维护一批flink作业

#### flink分离式引擎设计细节

![image-20251019003126283](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003126283.png)

**【整体流程】**

- **启动分离式和非分离式引擎，以及管理引擎的流程图**

![image-20251019003201486](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003201486.png)

- **创建Flink Manager引擎、查询status和save详细过程**

![image-20251019003228731](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003228731.png)

​	status、kill操作均为通用操作，可以走ECM。save、doCheckPoint为flink专有操作，需要走flink管理引擎。



**启动流程**：

**streamis -->linkis manager --> ECM --> 判断是否有flink manager ec，没有则初始化一个并创建一个flink 普通ec，否则只创建一个flink 普通ec --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作 --> attach:EC常驻，detach:EC退出。** 

**通用操作**：

**streamis --> linkis manager --> ECM --> query status/kill等操作 --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作**

**flink特有操作：**

**streamis --> linkis manager -->  ECM --> flink manager ec --> savepoint/checkpoint/stop --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作**

【**streamis相关功能点**】

- JobLaunchManager内置**FlinkManagerFactory，支持创建或复用现有的flink管理引擎（flink manager ec）。**flink特有操作均需要通过flink manager ec执行。
- 新增定时任务，**每10min请求jobmanager ask flink manager ec，保证有可用的flink管理ec 存活**

**【Linkis相关功能点】**

①linkis flink EC新增detach模式

- 提交任务 starupMap linkis.flink.client.type=detach （默认为attach）
  - FlinkJarOnceExecutor 重写execute逻辑，如果参数linkis.flink.client.type=detach  ，则启动定时任务watiToExit方法，每一分钟循环一次，检查applicationId不为null，则上报心跳将applicationId上报至streamis，上报成功后，就执行trySucceed，使FlinkEC JVM 退出。

②新增linkis manager ec的manager executer

- 接收到Streamis的停止应用、快照生成、metrics查询（定义接口，不实现）请求后，**起一个flink yarn client通过applicationId attach到yarn集群的flink应用实例，进行相关操作并返回**。
- executor内置支持并发操作的**并行任务计数器**，如果计数器超过阈值（默认100），则flink管理ec会转变状态为busy，linkismanager会启动新的flink 管理ec，并且本次请求仍然提交到现有busy的flink管理ec。为了便于加锁处理， flink manager executor依然需要继承并发executor ConcurrentComputationExecutor的能力。

③linkismanager新增operation选择flink manager ec策略

- flink manager ec是并发引擎，先采用默认node打分逻辑，选取标签匹配的所有ec，并且加入缓存concurrentHashMap。10s未写入则缓存失效。从缓存引擎中拿取管理ec，如果有空闲引擎，选择创建最早的。如果都是busy引擎，选择创建最晚的。

④linkismanager新增operation类型定义：操作增加类型： 基本、专有

- 基本操作则会提交给ECM、专有操作则会提交给按照operation内的engineType、codeType标签匹配的flink manager ec。



#### flink client包与scala包的关系

【**flink.client 包的作用**】

​	flink.client 是 Flink 引擎的核心模块之一，主要用于客户端与 Flink 集群的交互。它提供了以下功能：

- **集群部署**：负责将 Flink 应用程序提交到不同的集群（如 Yarn、Kubernetes 或 Standalone 集群）。
- **任务管理**：通过客户端 API 与集群通信，支持任务提交、取消、状态查询等操作。
- **配置管理**：提供对 Flink 配置项的加载和解析功能。

在代码中，flink.client 包通常包含以下内容：

- ClusterDescriptor：定义了如何描述和部署 Flink 集群。
- ClusterClient：用于与运行中的 Flink 集群进行交互。
- RestClusterClient：基于 REST 协议的客户端实现，用于与远程 Flink 集群通信。
- YarnClusterDescriptor 和 YarnClusterClient：专门用于与 Yarn 集群交互的实现。
  - 例如，在片段8中可以看到 LinkisYarnClusterClientFactory 继承自 YarnClusterClientFactory，这是 flink.client 包的一部分，用于创建 Yarn 集群客户端。

**【scala 包的作用】**

​	scala 包是 Linkis 的 Flink 插件模块，主要实现了 Flink 引擎与 Linkis 平台的集成。它的核心职责包括：

- 执行器实现：定义了如何在 Linkis 中运行 Flink 任务（如 FlinkJarOnceExecutor、FlinkSQLComputationExecutor 等）。
- 上下文管理：维护 Flink 任务的运行时上下文（如 EnvironmentContext）。
- 监听器机制：通过监听器（如 FlinkListenerGroup）捕获任务状态变化并通知 Linkis 平台。
- 工具类封装：提供了一些工具类（如 YarnUtil），用于简化 Flink 客户端的操作。
  - 例如，在片段4中可以看到 YarnUtil 提供了与 Yarn 集群交互的工具方法，而在片段5中可以看到 FlinkSQLComputationExecutor 实现了 SQL 执行逻辑。

**【flink.client 包与 scala 包的关系】**

flink.client 包与 scala 包之间的关系可以总结为以下几点：

**(1) 功能层次上的依赖**

- scala 包依赖于 flink.client 包提供的底层功能，例如：
- 在片段6中，FlinkEngineConnFactory 使用了 flink.client 包中的 Configuration 类来设置 Flink 的运行时参数。
- 在片段7中，FlinkRestClientManager 使用了 flink.client.program.rest.RestClusterClient 来与 Flink 集群通信。

**(2) 抽象与实现的分离**

- **flink.client 包提供了通用的客户端抽象（如 ClusterDescriptor 和 ClusterClient）**，而 scala 包则实现了这些抽象的具体逻辑。
- 例如，片段8中的 LinkisYarnClusterClientFactory 是对 flink.client 包中 YarnClusterClientFactory 的扩展，专门用于 Linkis 平台的 Yarn 集群部署。

**(3) 语言层面的桥接**

- flink.client 包是基于 Java 编写的，而 scala 包是基于 Scala 编写的。两者通过 Scala 的互操作性（Scala 与 Java 的无缝调用）实现协作。
- 例如，在片段11中，FlinkCodeOnceExecutor 使用了 flink.client.deployment.ClusterClientJobClientAdapter，这是一个 Java 类，但在 Scala 中可以直接使用。

**(4) 功能扩展**

- scala 包在 flink.client 包的基础上进行了功能扩展，以满足 Linkis 平台的需求。例如：
- 在片段10中，EnvironmentContext 封装了 Flink 和 Yarn 的配置，并添加了额外的参数（如 extraParams）。
- 在片段17中，FlinkSQLComputationExecutor 基于 flink.client 提供的 Operation 接口实现了 SQL 执行逻辑。

**【总结】**

- flink.client 包：提供了 Flink 客户端的核心功能，负责与 Flink 集群交互。
- scala 包：基于 flink.client 包的功能，实现了 Linkis 平台与 Flink 引擎的集成。

**关系：**

- scala 包依赖于 flink.client 包提供的底层功能。
- scala 包对 flink.client 包进行了功能扩展和定制化。
- 通过 Scala 与 Java 的互操作性，两者无缝协作。
- 这种设计使得 Linkis 能够灵活地利用 Flink 的原生功能，同时扩展出适合平台需求的特性。

## Streamis高可用

### 高可用审计

平台会根据前期的用于登记架构，对生产的应用巡检，发送日报和提问题单。包括：

1）监控同一个集群中是否有同名的应用；

2）双集群架构的检查两个IDC中的物料和实际存活性；

3）三集群架构的检查容灾的物料和实际存活性。

### 单集群高可用

#### 作业自动重拉

- 对于**单集群策略**的作业：根据用户配置的作业失败自动拉起开关确定是否重拉
- 对于**多集群策略**的作业（非single和singleWithBak的其它策略）：强制开启，同时在配置里加一个开关，可以配置是否开启这个功能

​	对于单集群高可用，实际上它是通过一个**作业自动重拉逻辑**来实现的。我们通过一个自动重拉开关**fail_restart_switch**来判断是否对失败任务重拉，如果配置为ON，就会在yarn应用失败后进行重拉和告警。另外如果用户在streamis侧手动停止应用、后手动启动，应用没有进入Running状态就失败，均不会触发自动重拉，这个是针对单集群策略single的重拉策略；对于多集群策略，如双活double和热主冷备managerSlave策略，我们通过会强制该开关开启，并额外设置一个外部开关来控制该功能的开关。

![image-20251019003253347](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003253347.png)

#### 核心逻辑

​	**`TaskMonitorService` 是一个任务监控服务类**，主要用于**管理和监控流式任务（StreamTasks）的运行状态，并在任务异常或失败时触发告警、自动恢复等操作**。以下是对其核心功能和设计的详细解析：

**(1) 任务状态监控**

- 定期检查所有流式任务的状态（通过 doMonitor() 方法）。
- 更新任务状态到数据库中（如 streamTaskMapper.updateTask()）。
- 如果任务长时间未更新状态，则触发告警。

**(2) 任务异常处理**

- 当任务失败时（如 Flink 任务状态为 FAILED），分析失败原因并记录错误信息。
- 根据配置决定是否自动重启失败的任务（如 restartJob() 方法）。
- 支持高可用策略（如 HIGHAVAILABLE_POLICY 配置）。

**(3) 告警机制**

- 根据任务状态和配置，向相关用户发送告警信息（通过 alert() 方法）。
- 告警内容包括任务失败、Linkis 集群异常等。

**(4) 任务清理与重置**

- 在服务启动时，清理未完成的任务（如 STARTING 状态的任务），避免因服务重启导致的任务残留。
- 提供任务重置功能，确保服务重启后任务状态一致性。

#### 核心方法解析

##### 初始化与销毁

**（1）【服务启动时初始化定时任务（如 doMonitor() 的周期性执行）和清理未完成的任务】**

- 启动定时器，定期调用 doMonitor() 方法。
- 清理服务重启后遗留的未完成任务。

```
@PostConstruct
def init(): Unit
```

**1）【使用 Utils.defaultScheduler.scheduleAtFixedRate 实现周期性任务监控】**

![image-20250321222103386](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321222103386.png)

**`Utils.defaultScheduler.scheduleAtFixedRate`：**创建一个固定频率的定时任务。



实际上用的是**线程池中的Executors类的ScheduledThreadPoolExecutor**

ScheduledThreadPoolExecutor使用的任务队列**DelayQueue封装了一个PriorityQueue**，PriorityQueue会对队列中的任务进行排序，**执行所需时间短的放在前面先被执行(ScheduledFutureTask的time变量小的先执行)，如果执行所需时间相同则先提交的任务将被先执行**

- 第一个参数：任务执行逻辑：Runnable 接口实现，执行doMonitor()方法。
- 第二个参数：初始延迟时间（单位：毫秒）。
  - 这里设置为 JobConf.TASK_MONITOR_INTERVAL.getValue.toLong，表示首次执行的时间间隔。
- 第三个参数：任务执行间隔时间（单位：毫秒）。
  - 同样由 JobConf.TASK_MONITOR_INTERVAL 配置决定。
- 第四个参数：时间单位（TimeUnit.MILLISECONDS）。

**将定时任务的返回值赋值给 future对象，表示该定时任务的执行状态。**

- 这个 Future 对象可以用来检查任务是否完成、取消任务等。

- 在服务关闭时，可以通过 future.cancel(true) 取消定时任务，释放资源

  ```
  @PreDestroy
  def close(): Unit = {
      Option(future).foreach(_.cancel(true))
  }
  ```



Future 提供了以下方法来检查任务状态：

- isDone()：判断任务是否已完成。
- isCancelled()：判断任务是否已被取消。

**2）【在服务启动时清理未完成的任务（处于STARTING状态）】**

![image-20251019003327577](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003327577.png)

通过**`wds.streamis.job.reset_on_restart.enable`配置**是否在服务启动时清理未完成的任务

- 将任务清理逻辑提交到线程池中异步执行。

```
Utils.defaultScheduler.submit(new Runnable {
    override def run(): Unit = Utils.tryAndError {
        ...
    }
})
```

- 创建一个状态列表 statusList，包含需要清理的任务状态（这里是 STARTING）。
- 调用 streamTaskMapper.getTasksByStatus(statusList) 获取所有符合状态的任务。
- 使用 **filter 方法筛选出属于当前服务实例的任务**（通过 thisServerInstance ：{hostname}:{port}比较）。

![image-20250321225644211](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321225644211.png)

- 遍历所有未完成的任务，逐一清理并更新状态
  - 通过JobLaunchManage的connect() 连接 Linkis 集群得到jobClient，并调用 jobClient.stop() 停止任务。
  - 更新任务状态为 FAILED，并记录错误描述。
  - 调用 streamTaskMapper.updateTask(tmpTask) 将更新后的任务状态写入数据库

**（2）【服务关闭时取消定时任务，释放资源】**

```
@PreDestroy
def close(): Unit
```

##### 任务监控

定期扫描所有流式任务，检查其状态并执行相应操作。

```
def doMonitor(): Unit
```

【**关键逻辑**】

- 获取所有未完成的任务（getTasksByStatus）。
- 检查任务是否需要监控（shouldMonitor）。
- 更新任务状态（refresh 方法）。
- 如果任务失败，触发告警（alert方法）并尝试自动重启（restart方法）

**【更新任务状态】**

**重试机制：**

- 使用 RetryHandler 实现重试逻辑，最多重试 3 次，每次间隔不超过 2 秒。
- 捕获指定异常（如 ErrorException）并重试。

**刷新任务状态：**

- 调用 refresh() 方法，通过 jobLaunchManager 获取任务的最新状态。

**异常处理：**

- 如果刷新失败，记录错误日志。
- 如果错误原因是 EngineConn 不存在，则将任务状态标记为 FAILED。
- 如果其他原因导致失败，触发告警

**【处理失败任务】**

如果更新任务状态为失败，则对失败任务进行处理

- 日志记录：记录任务失败的日志。
- 构造额外信息：如果任务信息包含 EngineConn，**提取其 ApplicationId。**
- 更新错误描述：设置任务的错误描述，并保存到数据库。
- **错误码匹配：调用 errorCodeMatching() 方法分析失败原因**。（之前有个日志拉取把磁盘拉爆的生产问题）
- **自动重启**：
  - 根据配置决定是否自动重启任务（如 FAIL_RESTART_SWITCH 和高可用策略）。
  - 如果需要重启，调用 restartJob() 方法。
- **告警通知**：向相关用户发送告警信息。

##### 重启任务

![image-20250321235316245](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321235316245.png)

- 调用 restartJob 方法，尝试重新启动任务。
- 检查自动恢复开关配置**fail_restart_switch**，发现为 "ON"。
- 调用 `streamTaskService.asyncExecute()` 方法，异步提交任务。
  - `job.getId`：任务所属的作业 ID。
  - `0L`：任务版本号（通常为 0 表示最新版本）。
  - `job.getSubmitUser`：任务的提交用户。
  - `startAutoRestoreSwitch`：是否启用自动恢复功能
- 如果提交成功，任务进入运行状态。
- 如果提交失败，捕获异常并执行以下操作：
  - 记录日志：Fail to reLaunch the StreamisJob [StreamingJob-001]。
  - 获取告警用户列表并发送告警信息：Fail to reLaunch the StreamisJob [StreamingJob-001], please be noticed!。

### 多集群高可用

分为两个部分：**多集群高可用发布**和**多集群一致性启动**

整个发布启动流程分为两个过程，概括一下就是

- **多集群高可用发布**：就是将物料包推送到streamis服务上，包括单活、双活、双活灾等高可用策略
- **多集群一致性启动**：检查集群物料的一致性，根据高可用策略启动不同集群上的作业

![image-20251019003403098](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003403098.png)

![image-20251019003417190](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003417190.png)

#### 多集群高可用发布

通过指定的打包层级和差异化替换模板，实现streamis flink作业的多集群高可用发布

**【包括以下几个部分】**

- 物料包打包格式，发布前会校验；
- meta.json文件模版，对变量进行分级，发布前会校验；
- 校验发布集群与填写的高可用策略是否一致；
- 在cmdb平台上登记flink应用高可用架构以及信息，发布前会校验；
- 正式发布流程，通过高可用策略把物料发步到不同集群，带上source的jsaon字段，包括了物料包md5、发布人、一致性标志位等信息，为后续一致性启动做准备

![image-20251019003440220](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003440220.png)

##### 物料包打包格式

​	校验物料包的打包层级，用户必须严格按照提供的物料包模版进行打包，否则无法通过校验流程。我们规定了相应打包格式在这里。主要分为三个部分：

- 1）自动化执行脚本，主要包括物料发布、参数校验的核心逻辑；
- 2）差异化变量，主要包括需要用到的环境变量如集群地址等；
- 3）资源文件，主要包括用户本次发布启动需要用到的物料资源包，如jar包等。

![image-20250323223443053](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250323223443053.png)

##### meta.json作业模版定义

通过作业模版的差异化配置来确保job的一致性



**如果每个job的meta.json都不一样，那么无法实现一个物料包同时发送多个集群**

因此我们将json文件的key进行了分类，

- 1）Job级别1：job级别相同，必须校验物料一致性，使用固定值，定义为级别①
- 2）Job级别2：job级别不相同，不必须物料校验一致性，须保证参数数量一致，使用cmdb idc变量，定义为级别②
- 3）idc级别不同的，要用cmdb icd变量，定义为级别③



Meta.json文件是StreamisJob的元数据信息，记录了该job需要用到的一系列信息：比如项目名、作业名、并行度和其他job生产参数等信息，某些参数可能会由于job不同集群不同等原因而产生变化。因此将meta.json的参数划分为不同的等级，并按照等级来确认参数的value值需要使用固定值或者差异化变量。通过将meta.json的参数划分为不同等级后，考虑到用户失误或是其他意外情况，需要额外对用户上传的meta.json文件的参数进行校验。

```
 {
  "jobType": "flink.jar",  1
  "jobName": "0725_1",   1
  "tags": "test_0.2.5_jar_12",  2[@IDC_STREAMISJOBCONF..p1.j1..c1]
  "description": "测试普罗米修斯",   2[@IDC_STREAMISJOBCONF..p1.j1..c2]
  "jobContent": {
    "main.class.jar": "demo-stream-app-0.1.3-SNAPSHOT.jar",  1
    "main.class": "com.webank.wedatasphere.stream.apps.core.entrance.StreamAppEntrance",  1
    "args": [   
      "--kafka.bootstrap.servers",
      "10.108.192.49:9092,10.108.192.59:9092,10.108.192.75:9092",
      "--user",
      "hadoop",
      "--kafka.topic",
      "cfpd_rcs_ef_risk_bdap"
    ],3[@IDC_STREAMISJOBCONF..p1.j1..c3]
    "hdfs.jars": [],1
    "resources": [] 1
  },
  "jobConfig": {
  "wds.linkis.flink.resource": {
    "wds.linkis.flink.app.parallelism":"1", 1
    "wds.linkis.flink.jobmanager.memory":"1024",  1
    "wds.linkis.flink.taskmanager.memory":"2048",    1
    "wds.linkis.flink.taskmanager.numberOfTaskSlots":"2",    1
    "wds.linkis.flink.taskmanager.cpus":"2",    1
    "wds.linkis.rm.yarnqueue":"dws"    3[@IDC_STREAMISJOBCONF..p1.j1..c4]
  },
  "wds.linkis.flink.custom": {
        "classloader.resolve-order": "parent-first",    1
    "demo.log.tps.rate": "600",   2[@IDC_STREAMISJOBCONF..p1.j1..c5]
    "security.kerberos.krb5-conf.path": "/appcom/config/keytab/krb5.conf",   2[@IDC_STREAMISJOBCONF..p1.j1..c6]
    "security.kerberos.login.contexts": "Client,KafkaClient",      2[@IDC_STREAMISJOBCONF..p1.j1..c7]
    "security.kerberos.login.keytab": "/appcom/Install/streamis/hadoop.keytab",    2[@IDC_STREAMISJOBCONF..p1.j1..c8]
    "security.kerberos.login.principal": "hadoop/inddev010004@WEBANK.COM",   2[@IDC_STREAMISJOBCONF..p1.j1..c9]
    "stream.log.filter.keywords": "INFO,ERROR",      1
    "stream.log.filter.level-match.level": "INFO",    1
    "env.java.opts": " -DjobName=0607_1",        3[@IDC_STREAMISJOBCONF..p1.j1..c10]
  },
  "wds.linkis.flink.produce": {
    "wds.linkis.flink.checkpoint.switch":"ON",     1
    "wds.linkis.flink.alert.failure.user":"alexyang",   1
    "wds.linkis.flink.app.fail-restart.switch":"OFF",   1
    "wds.linkis.flink.app.start-auto-restore.switch":"OFF",    1
    "linkis.ec.app.manage.mode": "detach"     1
  }
}
}
```

##### 校验发布集群与填写的高可用策略是否一致

​	目前我们定义了六种高可用策略：**single单活，singleWithBak单活灾，double双活，doublewithbak双活灾，managerslave热主冷备，managerslavewithbak主备灾**。每一个集群都支持多种高可用策略，但是也有不支持的策略。

- 比如对于容灾集群只能发布策略中带BAK的job，比如singleWithBak、doublewithbak、managerslavewithbak。因此我们会在发布前进行 批量检查，检查当前任务的“wds.streamis.app.highavailable.policy”参数值是否符合当前发布集群的高可用策略，如果不符合条件，则检查失败。

![image-20250326170920888](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326170920888.png)

##### 在cmdb平台上登记flink应用高可用架构以及信息

​	出于对流式应用的规范化管理，需要用户在发布作业前在cmdb录入对应的job信息，包括其所属子系统、应用架构等。所以在发布前的最后一步还需要检查用户是否按照规范将job录入到了cmdb中。

​	具体流程是：先通过itsm进行流式应用信息登记，cmdb成功录入流式应用的信息，然后在发布前校验会调用cmdb的接口去判断当前流式应用是否在cmdb存在并且符合记录，还有meta.json里的高可用参数“wds.streamis.app.highavailable.policy”是否跟cmdb登录的一样，如果不同的话也是会校验失败无法发布。

​	同时使用定时脚本巡检高可用策略与cmdb登记的是否一致

![image-20250326171314170](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326171314170.png)

![image-20250326171339510](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326171339510.png)

##### 正式发布流程

​	主要过程就是通过高可用策略把物料发步到不同集群，针对不同的高可用策略的job发布策略也有调整，比如单活任务只会发布到选定集群上，双活任务则会同时发布到双集群上。

​	在将物料上传至streamis集群的同时会带上一个名为source的json字段，包括了物料包md5、发布人、一致性标志位等信息，为后续一致性启动做准备。

![image-20251019003508639](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003508639.png)

#### 多集群一致性启动

**主要包括物料一致性检查和任务启动两个部分**

​	用户可以在页面上填写两个需要启动的集群名，其中单集群策略比如single则两个集群名填写一个即可；若是多集群策略如double，则需要填写两个集群名不同且相关的集群，比如BDAP_UAT和BDAP_SIT这两个互为主备的集群

```
job-list
 |--job-list.sh
 |--joblist.txt ----（格式：proj,job ，每行只有一个job）
 |--streamis-env.sh
 |--user-env.sh
```

![image-20251019003530526](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003530526.png)

![image-20251019003549365](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003549365.png)

![image-20251019003605018](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003605018.png)

##### 一致性检查

**脚本一致性检查流程：**

- 读取当前job的高可用策略，看他是否是主备或者是双活策略，如果是的话就会进入一致性校验的主流程，否则就会跳过校验。
- 具体的校验主流程主要是通过第一次发布传入的source参数来校验一致性，检查主备集群中source字段的物料包md5和发布人是否一致，若一致则将标志位置为true并将成功信息塞入message字段中；若不一致则将标志位置为false，并将具体不一样的物料信息传入message，更新source字段的这两个key到数据库，最后汇总检查结果。

**source字段结构如下：**

```
{
 "pkgName": "$pkgName",
 "deployUser": "$deployUser",
 "createTime": "$createTime",
 "source":"$source",
 "isHighAvailable":"$isHighAvailable",
 "highAvailableMessage":"首次上传",
 "token":"streamis0.3.8"
}
```

![image-20250326172455273](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326172455273.png)

##### 启动任务

​	一致性检查后则开始对主备集群的job发起启动命令，通过**后台服务代码**读取source字段中的一致性标志位和来源，从而判断job是否能够启动，判断一致性通过后才会最终启动job，否则最终无法启动job。

- 首先判断高可用执行开关 是否开启，若开启则会进行下一步；
- 判断双活策略key（wds.streamis.app.highavailable.policy）是否为"双活、主备"，是double则会继续校验一致性；否则会直接检查通过并提示当前任务是单活
- 继续判断source中的source字段是否为aomp，若来自aomp则会继续校验一致性；否则会直接检查通过并提示当前任务不是来自aomp
- 最后判断isHighAvailable参数是否为true，为true则允许job启动，否则不能启动。

**通过最终校验后，开始启动任务：**

- **若发布时的"wds.streamis.app.highavailable.policy"参数为double，则启动时会同时启动主备两个集群的job；若参数为single，则启动时会启动主集群的job，主集群为启动时设定的地址（可在配置差异化变量方案的步骤查看）；若为managerSlave则也只会启动主集群的job。**

![image-20251019003640463](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003640463.png)

#### Streamis双活应用的数据一致性问题

​	**流式应用方需确保幂等问题，即就算作业重启，不会存在数据重复计算、漏算等问题**

​	**BDP集群每天会定时进行一次数据备份，数据时效为T-1。Hadoop平台无法做到数据实时同步，只能通过定时同步的方式对数据进行备份，对于Flink实时处理的数据如有更高的数据时效需求，应自行设计外部存储的数据备份方案。**

​	**双活通常指两个数据中心同时运行，互为备份。当某个数据中心故障时，流量会切换到另一个数据中心，由存活的应用继续处理数据。**若数据同步是双向的，每个机房可能都有全量数据，但需处理同步冲突

##### 确定数据来源

在双活应用中，两个Flink作业可能从**同一数据源（如Kafka主题）消费数据**

【**Kafka消费者组与分区分配**】

- **原理**：通过配置**两个Flink应用使用同一Kafka消费者组**，Kafka会**根据负载均衡策略将主题分区分配给不同应用的消费者**。例如，若主题有10个分区，Kafka可能分配5个分区给应用A，5个分区给应用B。
- **实现**：在Flink的KafkaSource中设置相同的消费者组ID（"group.id"），通过Kafka监控工具（如Kafka Manager）查看分区分配情况，确认哪个应用消费了哪些分区。
- **挑战**：需确保作业配置一致，避免分区分配冲突。分区倾斜可能导致性能不均，需优化分区策略。

【**元数据标记**】

- **原理**：在Flink作业的处理管道中为输出数据添加标识，如通过map函数添加“application_id”字段，标明是哪个应用处理的。

- **实现**：在代码中为每个应用配置唯一标识比如ID（通过环境变量或配置文件），在DataStream API中添加，并添加到输出。例如：

  ```
  DataStream<Output> output = input.map(event -> new Output(event, appId));
  ```

- **优势**：下游系统可直接根据标识识别数据来源，灵活性高。

【**水印与事件时间管理**】

- **原理**：Flink使用水印（watermark）管理事件时间，两个应用可通过共享水印生成逻辑或外部时间服务，确保处理顺序一致。
- **实现**：配置相同的watermark策略（如基于事件时间的固定延迟），通过日志或监控工具验证水印进度。
- **作用**：间接验证数据来源是否符合预期时间窗口，但需结合分区分配使用。

##### 保证数据一致性

**流式应用方需确保幂等问题，即就算作业重启，不会存在数据重复计算、漏算等问题**

**Flink不支持跨作业的直接状态共享，因此确保两个应用的数据一致性需要依赖外部机制或调整架构。**

【**Exactly-Once语义**】

- **原理**：**Flink通过检查点（checkpointing）和状态管理实现Exactly-Once处理**，确保数据不重复、不丢失。检查点定期将operator状态保存到持久存储（如HDFS），故障恢复时从最新检查点恢复。
- **实现**：启用checkpointing，配置存储后端（如RocksDBStateBackend），并确保sink支持事务（如Kafka的Exactly-Once producer）。
- **优势**：每个应用内部保持一致性，适合独立处理的分区数据。
- **参考**：Flink文档强调Exactly-Once是高可用性基础 ([Apache Flink Documentation](https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/config/))。

【**事务性Sink**】

- **原理**：在将处理结果写入外部系统时，使用支持事务的sink，确保写入一致性。例如，Flink支持Kafka的Exactly-Once producer，通过两阶段提交（Two-Phase Commit）避免重复数据。
- **实现**：配置Kafka sink启用事务，结合检查点确保写入与状态一致。
- **优势**：即使两个应用同时写入，Kafka能通过事务机制避免重复。
- **挑战**：需确保sink支持Exactly-Once，否则可能导致数据不一致。

【**共享状态存储**】

- **原理**：使用共享外部存储（如HDFS、S3）保存检查点和保存点（savepoint），理论上两个集群可以访问相同的状态快照。
- **实现**：配置Flink的checkpoint存储为共享路径，例如fs.state.backend: hdfs://shared-path，并确保权限一致。
- **优势**：共享存储避免状态分散，是双活集群一致性的基础。
- **挑战**：实际操作中，Flink的州管理通常绑定到作业，跨作业共享状态需自定义实现，可能涉及外部数据库或键值存储（如Redis），增加复杂性。

【**外部协调服务**】

- **原理**：使用ZooKeeper等服务监控集群状态，进行leader选举或故障转移。如果一个集群失败，另一个集群可以接管处理，同时同步状态。
- **实现**：配置Flink的高可用性服务（如HighAvailabilityServices）使用ZooKeeper，监控集群健康状态。
- **优势**：提高系统可用性，辅助数据一致性维护。
- **挑战**：需额外配置和维护协调服务，增加运维成本。

#### 热主冷备高可用：flink跨集群快照同步

##### flink跨集群快照同步设计

​	为满足配置热主冷备高可用策略的流式应用，在进行主备切换的时候，能够从备集群成功继承原来的流式应用状态，并基于该状态进行应用恢复的需求，要求流式应用能够做到跨集群的快照同步，当主集群异常时，可以从备集群根据同步过来的最新快照进行流式应用恢复。

**【详细设计过程】**

![image-20251019003719423](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003719423.png)

**【包含的函数】**

1. **check_cluster_status** : 检查当前集群是否为主集群，通过调用Streamis的API，决定是否执行同步。
2. **kerberos_certification** : 执行Kerberos认证，确保后续HDFS操作的权限。
3. **get_active_nn** : 从NameNode列表中找到当前Active的节点，确保HDFS操作的可用性。
4. **pre_distcp** : 预检查，包括检查YARN任务状态、运行任务数是否超过限制、目标路径是否合法。
5. **distcp_snapshot** : 使用hadoop distcp命令同步数据，包含重试机制。
6. **execute_cmd** : 封装subprocess另起一个子进程执行命令，处理异常和输出。

**【整个流程】**

**（1）前置检查、权限认证**

**步骤1：集群状态检查**

- 发送HTTP GET请求到Streamis接口 `/streamis/streamJobManager/highAvailable/getHAMsg`，检查当前集群是否为主集群，是主集群则进行下个步骤，否则退出并告警

**步骤2：Kerberos认证**

- 执行 `kinit -kt <keytab> <user>` 命令，确保后续HDFS操作的权限，认证成功则进行下个步骤，否则发送告警并退出

**步骤3：获取Active NameNode**

- 遍历 `nn_str` 中的所有NameNode地址，对每个地址执行 `hdfs dfs -ls /flink`，若输出中无 `state standby`，标记为Active节点。

**（2）Checkpoint同步流程**

**步骤4：预检查 (`pre_distcp`)**

**检查YARN任务状态** 

- 执行 `yarn application -list` 过滤出 `distcp_streaming_apps` 相关任务。
- 若存在 `ACCEPTED` 状态任务：判断为队列资源不足，告警退出。
- 若 `RUNNING` 任务数超过 `running_job_num_limit`：告警退出。

**步骤5：执行distcp同步**

使用Hadoop `distcp`工具同步Checkpoint/Savepoint数据。构造distcp命令如下：

```
hadoop distcp \
  -Dmapreduce.job.queuename=<queue> \
  -Dmapred.job.name=distcp_streaming_apps_checkpoint \
  -update \
  -delete \
  -pbcugp \
  hdfs://<active_nn>/flink/checkpoints \
  hdfs://<active_nn>/backup/checkpoints
```

通过 `-update` 和 `-delete` 实现**目录级增量同步** ，但同步范围是整个Checkpoint/Savepoint目录。



**【注意】**

**1)checkpoint路径规范，按照streamis目录规范**

**streamis上开启checkpoint后hdfs生成的checkpoint路径:**

- hdfs:///flink/flink-checkpoints/${应用架构名}/RCS_F1/ccpd_risk2_flsql_prehdl_zzj_dw_06/11532/0f4cae573abdc50e4fc11818c9f3fb1f/chk-170/_meatadata
- 双活架构：
  - hdfs:///flink/flink-checkpoints/${双活}/RCS_F1/ccpd_risk2_flsql_prehdl_zzj_dw_06/11532/0f4cae573abdc50e4fc11818c9f3fb1f/chk-170/_meatadata
- 热主冷备架构：
  - hdfs:///flink/flink-checkpoints/${热主冷备}/RCS_F1/ccpd_risk2_flsql_prehdl_zzj_dw_06/11532/0f4cae573abdc50e4fc11818c9f3fb1f/chk-170/_meatadata

**2)主备切换应用恢复过程：**

- 当发生集群切换时，需要手动从备集群上拉起作业，该作业会先读取当前作业成功同步过去的最新快照数据，然后基于该状态进行恢复

**3）从备集群拉起的作业，所继承的快照数据不保证是最新的chk**

- 因为在进行快照从主集群同步到备集群的过程中，为保证同步过去的每个chk为完整快照数据，每次只会同步较新的快照数据并且也会存在快照同步之前主集群异常使得最新快照无法顺利同步，会有以下情况：
  - 最新快照数据没开始同步，集群异常，导致最新快照数据无法同步，从备集群上同步的最新快照启动应用；
  - 快照数据同步一半，主集群异常，导致剩余的快照无法同步，尝试从备集群上同步的快照启动应用，如果失败，则从上一个快照进行应用恢复；
- **因此，备集群启动的作业是从较新的chk中恢复的作业。**

 

**【原过程】**

1.用户在itsm进行应用信息登记；

2.在主备集群的streamis部署机器，启动一个定时脚本，定期遍历yarn上的所有在运行的流式应用，并结合cmdb登记的应用架构信息过滤出使用热主冷备策略的流式应用；

3.根据流式应用的appId，获取到对应的流式应用jobId以及找出该流式应用的checkpoint地址，checkpoint周期以及checkpoint超时时间；

4.根据当前checkpoint目录下快照文件的更新时间过滤出可同步的快照文件，，并将满足条件的快照文件通过hdfs distcp命令同步到备集群

5.定期删除hdfs上过期的文件，以保留固定数量的快照文件。

![image-20251019003756689](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003756689.png)

**【原详细设计】**

![image-20251019003826447](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003826447.png)

1.通过定时脚本，定期**调用yarn restful接口获取yarn集群上所有在运行的流式应用列表**；

2.遍历流式应用，然后根据**应用名+逻辑集群作为唯一键**，调用cmdb接口获取当前流式应用的应用架构，并且只有当该流式应用是热主冷备高可用架构才继续往下执行，否则跳过；

3.根据当前应用信息找到应用的appId，并通过该**appId作为查询条件**，调用**flink restful接口找到对应流式应用的id：flink job id**，并找出当前应用的checkpoint地址，checkpoint周期以及checkpoint超时时间（ckp_timeout）；

4.**执行hdfs查找命令，过滤出当前流式应用的checkpoint列表**，先找出最新快照当前文件的更改时间（ckp_lastest_update_time），并将ckp_latest_update_time减去ckp_timeout得到文件未更新最晚时间（ckp_latest_unupdate_time），然后将该时间以前的快照文件通过hdfs distcp命令将此快照文件同步到备集群；

5.**通过hdfs删除命令，定时清理每个应用上的checkpoint数量**，如每个作业只保留5个checkpoint，先统计当前作业的checkpoint数量，当数量超过5时，会根据快照文件的生成时间先后，从早到晚依次删除过期的文件。

##### flink跨集群快照同步脚本代码解读

**【包含的函数】**

1. **check_cluster_status** : 检查当前集群是否为主集群，通过调用Streamis的API，决定是否执行同步。
2. **kerberos_certification** : 执行Kerberos认证，确保后续HDFS操作的权限。
3. **get_active_nn** : 从NameNode列表中找到当前Active的节点，确保HDFS操作的可用性。
4. **pre_distcp** : 预检查，包括检查YARN任务状态、运行任务数是否超过限制、目标路径是否合法。
5. **distcp_snapshot** : 使用hadoop distcp命令同步数据，包含重试机制。
6. **execute_cmd** : 封装subprocess执行命令，处理异常和输出。

【**1. 集群状态检查 (`check_cluster_status`)**】

**函数调用** ：`check_cluster_status(cluster_ipport, token_code, token_user)`

**功能** ：通过调用Streamis API判断当前集群是否为主集群（Active），仅主集群执行同步操作。

**实现细节** 

- **输入** ：`cluster_ipport`（集群地址）、`token_code`、`token_user`。
- **API调用** ：发送HTTP GET请求到`/streamis/streamJobManager/highAvailable/getHAMsg`接口。
- **响应解析** ：检查返回的JSON数据中的`whetherManager`字段（是否为主集群）。
- **容错机制** ：最多重试2次，失败则发送告警并退出。

**意义** ：避免在备用集群（Standby）上重复执行同步任务，确保数据一致性

【**2. Kerberos认证 (`kerberos_certification`)**】

**函数调用** ：`kerberos_certification(kerberos_certification_cmd)`

**功能** ：通过`kinit`命令进行Kerberos安全认证，确保后续HDFS操作权限。

**实现细节** 

- **输入**：`kerberos_certification_cmd`（如 `kinit -kt keytab user`）
- **命令执行** ：使用`subprocess.run`运行`kinit -kt keytab user`命令。
- **结果校验** ：检查命令返回码（0表示成功）。
- **异常处理** ：认证失败时发送告警并终止脚本。

**意义** ：满足Hadoop集群的安全访问要求，防止未授权操作。

【**3. 获取Active NameNode (`get_active_nn`)**】

**功能** ：从NameNode列表中找出当前Active的节点。

**实现细节** ：

- **输入** ：`nn_str`（逗号分隔的NameNode地址列表）。
- **遍历检测** ：依次对每个NameNode执行`hdfs dfs -ls`命令。
- **状态判断** ：通过命令输出是否包含`state standby`判断节点状态，跳过标记为 `state standby` 的节点
- **容错机制** ：最多重试2次，失败则告警退出。

**意义** ：确保后续HDFS操作（如distcp）连接到可用的NameNode。

【**4. 资源预检查 (`pre_distcp`)**】

**函数调用**：`pre_distcp(src_distcp_path_str, des_distcp_path_str):`

**功能** ：在同步前检查YARN资源和目标路径合法性。

**输入** ：源路径、目标路径。

**检查项** 

- **任务状态** 
  - 检查是否有处于`ACCEPTED`状态的任务（队列资源不足）。
  - 检查运行中的任务数是否超过阈值（`running_job_num_limit`）。
- **路径校验** ：确保目标路径与源路径不同（防止覆盖生产数据）。

**异常处理** ：任一检查失败则发送告警并终止脚本。

**意义** ：避免因资源不足或配置错误导致同步失败。

【**5. 分布式拷贝 (`distcp_snapshot`)**】

```
hadoop distcp \
  -Dmapreduce.job.queuename=streaming_queue \  # 指定YARN队列
  -Dmapred.job.name=distcp_streaming_apps_checkpoint \  # 任务名称
  -update \  # 增量同步（仅复制修改过的文件）
  -delete \  # 删除目标端多余的文件（删除目标路径中源路径不存在的文件，保持与源路径一致）。
  -pbcugp \  # 保留文件属性（块大小、校验和、权限等）
  hdfs://namenode1:8020/flink/checkpoints \  # 源路径
  hdfs://namenode2:8020/backup/checkpoints  # 目标路径
```

**函数调用：**`distcp_snapshot(active_nn_url,exec_queue_str,src_distcp_path_str,des_distcp_path_str,snapshot_str)`

**功能** ：使用Hadoop `distcp`工具同步Checkpoint/Savepoint数据。

通过 `-update` 和 `-delete` 实现**目录级增量同步** ，但同步范围是整个Checkpoint/Savepoint目录。

**输入** ：

- `active_nn`：Active NameNode地址。
- `src_path`：源HDFS路径（如 `/flink/checkpoints`）。
- `des_path`：目标HDFS路径（如 `/backup/checkpoints`）

**核心参数** 

- `-update`：增量同步：仅同步修改过的文件（**仅同步源路径中修改时间晚于目标路径 的文件**）。
- `-delete`：删除目标端多余的文件**（删除目标路径中源路径不存在的文件，保持与源路径一致）**
- `-pbcugp`：保留文件属性（块大小、校验和等）。

**执行流程** 

1. 调用`pre_distcp`进行预检查。
2. 构造distcp命令并执行。
3. 失败时重试2次，最终告警退出。

**意义** ：实现高效、可靠的HDFS数据增量同步。



**全量同步风险** ： 如果主集群Checkpoint/Savepoint目录包含历史数据 （如旧版本作业的Checkpoint），这些数据也会被同步到备集群。 

**解决方案 ：可通过时间戳过滤（如只同步最近N天的Checkpoint）。**

通过 **筛选出最近N天内修改的文件** ，仅同步这些文件到备集群。 **核心步骤** ：

1. **获取文件列表** ：列出主集群中所有Checkpoint/Savepoint文件。
2. **时间过滤** ：筛选出修改时间在最近N天内的文件。（使用 `hdfs dfs -ls` 结合 `awk` 和 `find` 过滤文件）
3. **增量同步** ：将筛选后的文件通过 `distcp` 同步到备集群。

【**6. 告警机制 (`alert`)**】

**功能** ：在任务失败时通过IMS系统发送告警。

**告警场景** ：

- 集群状态检查失败。
- Kerberos认证失败。
- YARN资源不足或任务数超限。
- distcp同步失败。

**实现细节**

- **参数构造** ：包含子系统ID、告警等级、接收人等信息。
- **HTTP请求** ：通过POST方法发送到IMS接口。

**意义** ：及时通知运维人员处理异常，保障任务可靠性。

【**7. 命令执行封装 (`execute_cmd`)**】

**功能** ：**封装`subprocess`模块**（**创建子进程并执行系统命令**），统一处理命令执行。

**特性** 

- 捕获标准输出和错误输出。
- 返回命令执行结果（返回码、输出内容）。
- 异常时发送告警。

**意义** ：简化代码逻辑，提高命令执行的健壮性。

```python
#!/usr/bin/env python
#coding:utf-8

import subprocess  # 执行shell命令
import os
import requests  # 发送HTTP请求
import json
import time

#ims alert args
# 告警子系统id
sub_system_id='5632'
# 告警标题
ims_alert_title='流式应用快照同步异常'
# 告警等级,1:critical,2:major,3:minor
ims_alert_level='3'
# 告警方式
ims_alert_way='1,2,3'
# 告警接收人
ims_alert_receiver='[&ims_alert_receiver]'
# 告警地址
ims_alert_url='[&ims_alert_url]'

# HDFS路径和集群配置
nn_str = '[&nn_str]'  # NameNode地址列表（逗号分隔）
src_checkpoint_path_str = '/flink/flink-checkpoints/managerSlave'  # 源Checkpoint路径
src_savepoint_path_str = '/flink/flink-savepoints/managerSlave'  # 源Savepoint路径
des_checkpoint_path_str = '[&checkpoint_path_str]'  # 目标Checkpoint路径（占位符）
des_savepoint_path_str = '[&savepoint_path_str]'  # 目标Savepoint路径（占位符）
exec_queue_str = '[&exec_queue_str]'  # YARN队列名（占位符）
job_name = '-Dmapred.job.name=distcp_streaming_apps'  # YARN任务名
cluster_ipport = '[&cluster_ipport]'  # Streamis集群地址（占位符）
token_code = '[&token_code]'  # 接口认证Token
token_user = '[&token_user]'  # 接口认证用户
running_job_num_limit = '[&running_job_num_limit]'  # 最大并发任务数限制

# kerberos认证参数（需替换为实际keytab和用户）
kerberos_certification_cmd=['kinit', '-kt', '[&kerberos_keytab]', '[&kerberos_user]']

# 发送ims告警
def alert(alert_info):
    print("Send alert...")
    try:
       ims_common_params={'sub_system_id':sub_system_id,'alert_title':ims_alert_title,'alert_level':ims_alert_level,'alert_way':ims_alert_way,'alert_reciver':ims_alert_receiver,'alert_info':alert_info}# 构造告警参数并发送POST请求
       print(ims_alert_url)
       ims_resp=requests.post(ims_alert_url,data=ims_common_params)
       print("Alert resp:"+str(ims_resp.text))
    except Exception as e:
       print("Failed to send alert and the exception as follow: "+str(e))

# 集群状态检查：当前集群状态是主集群才进行快照同步操作，否则不进行快照同步操作
def check_cluster_status(cluster_ipport, token_code, token_user):
   print("Check cluster status...")
   tries=0
   while tries < 2:
      try:
        # 调用Streamis API获取集群状态
cluster_url='http://'+str(cluster_ipport)+'/api/rest_j/v1/streamis/streamJobManager/highAvailable/getHAMsg'
         headers_data = {'Token-Code': token_code,'Token-User': token_user}
         resp_info=requests.get(cluster_url, headers=headers_data)
         cluster_resp=json.loads(resp_info.content)
         if cluster_resp['message'] == 'success':
             cluster_info=cluster_resp['data']
             cluster_name=cluster_info['clusterName']
             is_master_cluster=cluster_info['whetherManager']
             if not is_master_cluster:
                print("Current cluster is standby,skipping distcp snapshot data...")
                return False
             else:
                print("Current cluster is active,starting distcp snapshot data...")
                return True
         else:
             raise Exception("Cluster Service Unavailable, the resp info"+str(resp_info.content))
      except Exception as e:
         print("Failed to check cluster status and the exception as follow: " + str(e))
         print("retrying after 2s...")
         tries += 1
         time.sleep(2)
   else:
      exception_msg="Failed to distcp streaming apps snapshot data after 2 tries and the exception as follow: " + str(e)
      print(exception_msg)
      alert(exception_msg)
      exit(-1)  
   return False

# 进行kerberos认证
def kerberos_certification(krb_cert_cmd):
    print("Kerberos certification...")
    try:
      # 运行kinit命令进行认证
       cert_res=execute_cmd(krb_cert_cmd)
       #print(cert_res)
       if len(cert_res) !=0:
         res_code=cert_res[0]
         if res_code == 0:
            print("Kerberos certification successfully!")
            return True
         else:
            err_msg="Kerberos certification failed!"
            alert(err_msg)
            print(err_msg)
            return False
    except Exception as e:
        exception_msg="Kerberos certification failed and the exception as follow: " + str(e)
        print(exception_msg)
        alert(exception_msg)
    return False

# 获取目标集群active namenode
def get_active_nn(nn_str):
    print("Get active namenode...")
    active_nn_str=''
    tries=0
    while tries < 2:
       try:
          # 遍历所有NameNode，检查状态
          for nn in nn_str.split(","):
              path='hdfs://' + str(nn) + '/flink'
              cmd_str=['hdfs', 'dfs', '-ls']
              cmd_str.append(path)
              res=execute_cmd(cmd_str)
              # print(res)
              if len(res) != 0:
                # 如果返回结果中不包含'state standby'，则为Active节点
                 if 'state standby' not in res[1][0]:
                    active_nn_str=nn
          
          # jump out while loop
          break
       except Exception as e:
           print("Failed to get active namenode, retrying after 2s...")
           tries += 1
           time.sleep(2)
    else:
        exception_msg="Failed to get active namenode after 2 tries and the exception as follow: " + str(e)
        print(exception_msg)
        alert(exception_msg)
    print(active_nn_str)
    return active_nn_str


# 获取处于ACCEPTED或者RUNNING状态的快照同步任务
def get_apps(get_apps_cmd):
    tries=0
    while tries < 2:
       try:
          # 执行yarn命令过滤相关任务
           process=subprocess.Popen(get_apps_cmd,shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
           output,error=process.communicate()
           apps=output.strip().split("\n")
           apps=[x for x in apps if x != ""]
           return apps
       except Exception  as e:
           print("failed to get apps and the exception as follow: %s" % (str(e)))
           print("retrying after 2s...")
           tries += 1
           time.sleep(2)
    else:
       exception_msg="Failed to get apps after 2 tries and the exception as follow: " + str(e)
       print(exception_msg)
       alert(exception_msg)
       exit(-1)
    return ['']

'''
快照同步前预处理:
1.检查当前是否存在ACCEPTED状态的快照同步任务
2.检查当前快照同步任务数是否达到阈值
3.检查目标快照路径是否合法
'''
def pre_distcp(src_distcp_path_str, des_distcp_path_str):
    # 检查现有任务状态和数量
    get_apps_cmd="yarn application -list|grep \"distcp: distcp_streaming_apps\"|grep -E \"RUNNING|ACCEPTED\"|awk -F\" \" '{print $7,$1}'"
    apps=get_apps(get_apps_cmd)
    result_dict={}
    for item in apps:
        status,app=item.split(" ", 1)
        if status in result_dict:
           result_dict[status].add(app)
        else:
           result_dict[status]={app}
    # 检查是否有等待中的任务（队列资源不足）
    if 'ACCEPTED' in result_dict.keys():
        accept_apps=result_dict['ACCEPTED']
        if len(accept_apps) > 0:
           accept_msg="Queue resources is not enough and do not start the new distcp snapshot operator."
           print(accept_msg)
           alert(accept_msg)
           exit(-1)
    # 检查运行中任务是否超过限制
    if 'RUNNING' in result_dict.keys():
        running_apps=result_dict['RUNNING']
        if len(running_apps) > running_job_num_limit:
            running_msg="The number of running tasks has reach the limit: %s and do not start the new distcp snapshot operator." % str(running_job_num_limit)
            print(running_msg)
            alert(running_msg)
            exit(-1)
    # 路径校验,如果目标路径为生产正式路径，则不进行同步
    if src_distcp_path_str == des_distcp_path_str:
       msg="distcp destination path should be a tmp path"
       print(msg)
       alert(msg)
       exit(-1)
       
'''
快照同步操作
1.同步前先进行任务运行状态和路径校验
2.进行快照同步
'''
def distcp_snapshot(active_nn_url,exec_queue_str,src_distcp_path_str,des_distcp_path_str,snapshot_str):
   pre_distcp(src_distcp_path_str, des_distcp_path_str)

   print("distcp streaming apps %s data..." % snapshot_str)
   tries=0
   while tries < 2:
      try:
         # 构造distcp命令
         cmd_str=['hadoop', 'distcp'] # 使用Hadoop的distcp工具
         cmd_str.append(exec_queue_str)# 指定YARN队列
         cmd_str.append(job_name+"_"+snapshot_str)# 设置任务名称（便于识别）
         # 添加distcp核心参数
            exec_args = [
                '-update',  # 仅同步源路径中修改过的文件（增量同步）
                '-delete',  # 删除目标路径中源路径不存在的文件（保持一致性）
                '-pbcugp'   # 保留文件属性：
                            # -p: 权限  -b: 块大小  -c: 校验和类型
                            # -u: 用户  -g: 组     -p: 权限
            ]
         cmd=cmd_str + exec_args # 合并基础参数和核心参数
         des_path='hdfs://' + str(active_nn_url) + des_distcp_path_str
         cmd.append(src_distcp_path_str)# 源路径
         cmd.append(des_path) # 目标路径
         print(cmd)
         dist_res=execute_cmd(cmd) # 执行命令并获取结果
         if dist_res[0] != 0:# 返回码非0表示失败
            raise Exception("distcp snapshot data failed...")
         print("end distcp...")
         break
      except Exception as e:
         print("Failed to distcp snapshot data and the exception as follow: " + str(e))
         print("retrying after 2s...")
         tries += 1
         time.sleep(2)
   else:
    # 重试次数耗尽，发送告警并退出
      exception_msg="Failed to distcp snapshot data after 2 tries and the exception as follow: " + str(e)
      print(exception_msg)
      alert(exception_msg)
      exit(-1)

def execute_cmd(cmd):
    try:
      # 使用subprocess.Popen执行命令
       res=subprocess.Popen(cmd, 
                            stdout=subprocess.PIPE, # 捕获标准输出
                            stderr=subprocess.STDOUT # 合并标准错误到标准输出)
       output=res.communicate()
       return_code=res.returncode
       print("returncode=%d and output=%s" % (return_code, str(output)))
       return return_code,output
    except Exception as e:
       exception_msg="Failed to exec %s and the exception as follow: %s" % (str(cmd),str(e))
       print(exception_msg)
       alert(exception_msg)
    return ()
      
if __name__ == '__main__':
   # 检查当前集群状态，只有当前集群是主集群才进行快照同步；否则不进行快照同步；
   if check_cluster_status(cluster_ipport, token_code, token_user):
      # 进行kerberos认证
      if kerberos_certification(kerberos_certification_cmd):
         # 获取到active namenode 
         active_nn=get_active_nn(nn_str)
         print("active_nn="+active_nn) 
         if len(active_nn) != 0:
            # 进行checkpoint和savepoint快照同步
            distcp_snapshot(active_nn, exec_queue_str, src_checkpoint_path_str, des_checkpoint_path_str, "checkpoint")
            distcp_snapshot(active_nn, exec_queue_str, src_savepoint_path_str, des_savepoint_path_str, "savepoint")
         else:
            print("Failed to get active namenode.")
            exit(-1)
      else:
         print("Kerberos certification failed.")
         exit(-1)
```

### Streamis ⾼可⽤管理规范

**⼀，⽬标**

确定流式应⽤的单集群和多集群⾼可⽤管理规范

**⼆，单集群⾼可⽤规范**

- ⽤户作业配置⾃动失败重拉参数如果开启，Streamis在每分钟的定时任务⾥检测到Flink应⽤状态变为结束，则会⾃动重拉应⽤，并发出告警。⽤户应评估是否需要开启⾃动失败重拉，并配置合适的告警级别。

**三，多集群⾼可⽤规范**

- ⽤户应⽤发布前，应该在ITSM 提单 流式应⽤登记，登记名称、⾼可⽤架构、开发属主等信息，会登记到CMDB流式应⽤信息。
- ⽤户作业⽣产配置的⾼可⽤策略，应该跟CMDB⾥登记的⾼可⽤策略⼀致。
- 推荐流式应⽤使⽤双活和双活灾的⾼可⽤策略，其它⾼可⽤策略均要申请架构例外。其中主备、主备灾、双活、双活灾都涉及⽣产双集群部署作业，主备只在主集群启动应⽤，双活需要在主备集群都启动应⽤。有应⽤部署和启动不符合⾼可⽤策略的，会发出审计邮件限期整改。

## Streamis使用手册

### 1. 前言

本文是Streamis的快速入门文档，涵盖了Stremis的基本使用流程，更多的操作使用细节，将会在用户使用文档中提供。  

### 2. Streamis整合至DSS

为了方便用户使用，**Streamis系统以DSS组件的形式嵌入DSS系统中**

#### 2.1 如何接入？

按照 [StreamisAppConn安装文档](../development/StreamisAppConn安装文档.md) 安装部署StreamisAppConn成功后，Streamis系统会自动嵌入DSS系统中。

#### 2.2 如何验证 DSS 已经成功集成了 Streamis？

请进入 DSS 的工程首页，创建一个工程

进入到工程里面，点击左上角按钮切换到”流式生产中心“，如果出现streamis的首页，则表示 DSS 已经成功集成了 Streamis。如下图

### 3. 核心指标 

进入到streamis首页，上半部显示的是核心指标。

核心指标显示当前用户可查看到的上传到该项目执行的Flink任务的状态汇总，状态暂时有9种，显示状态名称和处于该状态的任务数量，具体内容如下图。

![image-20251019003917453](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019003917453.png)

### 4. 任务示例

主要演示案例从Script FlinkSQL开发，调试到Streamis发布的整个流程。

#### 4.1. FlinkSQL作业示例

**4.1.1. Script开发SQL**

顶部Scriptis菜单创建一个脚本文件，脚本类型选择Flink,如下图所示：

![进入FlinkSQL](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/enter_flinksql-2476903.png)

![create_script_file.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/create_script_file-2476903.png)

编写FlinkSQL，source,sink,transform等。

![image-20251019004013250](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004013250.png)

点击运行后，即可调试该脚本

**4.1.2. 文件打包层级**

流式应用物料包是指的按照Streamis打包规范，将元数据信息(流式应用描述信息),流式应用代码，流式应用使用到的物料等内容打包成zip包。zip具体格式如下：

```
xxx.zip
    ├── meta.json
    ├── streamisjobtest.sql
    ├── test.jar
    ├── file3
```

meta.json是该任务的元数据信息，streamisjobtest为flinksql文件

![flinksql_job_use_demo](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinksql_job_use_demo-2476903.png)

![flinksql_job_use_demo3](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinksql_job_use_demo3-2476903.png)

将SQL文件和meta.json文件打包成一个zip文件，注意：只能打包成zip文件，其他格式如rar、7z等格式无法识别

#### 4.2. FlinkJar作业示例

##### 4.2.1.  本地开发Flink应用程序

本地开发Flink应用程序，并打包成jar包形式

![flinkJar](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar.png)

##### 4.2.2.  文件打包层级

流式应用物料包是指的按照Streamis打包规范，将元数据信息(流式应用描述信息),流式应用代码，流式应用使用到的物料等内容打包成zip包。zip具体格式如下：

```
xxx.zip
    ├── meta.json
    ├── resource.txt
```

meta.json是该任务的元数据信息，resource.txt为flink应用使用到的资源文件

![flinkjar_zip](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar_zip.png)

![flinkjar_metajson](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar_metajson.png)

#### 4.3. meta.json文件格式详解

meta.json是StreamisJob的元数据信息，其格式为：

```
{
  "projectName": "", // 项目名
  "jobName": "",  // 作业名
  "jobType": "flink.sql",   // 目前只支持flink.sql、flink.jar
  "tags": "",   // 应用标签
  "description": ""    // 作业描述,
  "jobContent": {
    // 不同的jobType，其内容各不相同，具体请往下看
  },
  "jobConfig": {
        "wds.linkis.flink.resource": {
        "wds.linkis.flink.app.parallelism":"",  // Parallelism并行度
        "wds.linkis.flink.jobmanager.memory":"",  // JobManager Memory(M)
        "wds.linkis.flink.taskmanager.memory":"", // TaskManager Memory(M)
        "wds.linkis.flink.taskmanager.numberOfTaskSlots":"",  // TaskManager Slot数量
        "wds.linkis.flink.taskmanager.cpus":"",  // TaskManager CPUs
        "wds.linkis.rm.yarnqueue":""  // Yarn队列
        "flink.client.memory":""  // 客户端内存
      },
      "wds.linkis.flink.custom": {
            // Flink作业相关参数
      },
      "wds.linkis.flink.produce": {
          "wds.linkis.flink.checkpoint.switch":"OFF",  // Checkpoint开关
        "wds.linkis.flink.app.fail-restart.switch":"OFF",  // 作业失败自动拉起开关
        "wds.linkis.flink.app.start-auto-restore.switch":"OFF"  // 作业启动状态自恢复
      "wds.linkis.flink.product": "testRestart"   // 业务产品名称，用于回调日志路径生成，可以不填，默认为空
      "wds.streamis.app.highavailable.policy": "single"   // 高可用策略，如single，double，managerSlave等，默认为single
      "wds.linkis.flink.alert.failure.user": "gelxiogong,alexyang"  //  任务失败告警人，必填
      "linkis.ec.app.manage.mode": "detach"  // ec管理模式，分为detach和attach
      
      }
    }
}
```

其中，jobConfig为从0.2.5版本开始支持的配置项，该配置项可以不填，有默认的配置项，具体如下：

```
"jobConfig": {
        "wds.linkis.flink.resource": {
        "wds.linkis.flink.app.parallelism":"4",
        "wds.linkis.flink.jobmanager.memory":"2048",
        "wds.linkis.flink.taskmanager.memory":"4096",
        "wds.linkis.flink.taskmanager.numberOfTaskSlots":"2",
        "wds.linkis.flink.taskmanager.cpus":"2"
      },
      "wds.linkis.flink.custom": {
            "stream.log.filter.keywords":"ERROR,WARN,INFO",
            "security.kerberos.krb5-conf.path":"",
            "demo.log.tps.rate":"20000",
             "classloader.resolve-order":"parent-first",
             "stream.log.debug":"true",
             "security.kerberos.login.contexts":"",
             "security.kerberos.login.keytab":"",
             "security.kerberos.login.principal":""
      },
      "wds.linkis.flink.produce": {
          "wds.linkis.flink.checkpoint.switch":"OFF",
        "wds.linkis.flink.app.fail-restart.switch":"OFF",
        "wds.linkis.flink.app.start-auto-restore.switch":"OFF"
      }
    }
```

 ![image-20251019004047846](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004047846.png)

 ![image-20251019004057952](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004057952.png)

#### 4.4. 作业发布至Streamis

测试环境允许直接通过页面上传发布作业，生产推荐使用基于aomp的一致性物料发布与启动方案，用户使用手册：http://docs.weoa.com/docs/vVAXVnRQ4WsQj8qm

目前生产上暂时保留直接通过页面上传发布作业，后续会逐渐切换至aomp标准发布流程

发布模板：

![image-20251019004122378](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004122378.png)

 启动模板： ![aomp_based_start](/Users/glexios/Library/Mobile%2520Documents/com~apple~CloudDocs/Notes/%25E9%25A1%25B9%25E7%259B%25AE%25E6%2595%25B4%25E7%2590%2586.assets/aomp_based_start.png)

##### 4.4.1. 上传工程资源文件

进入到DSS页面，新建工程后，选择“流式生产中心”进入，点击“工程资源文件”

![image-20251019004139954](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004139954.png)

点击“导入”进行资源包的导入，选择文件系统的资源包，并设置版本号

![image-20251019004149760](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004149760.png)

导入完成

![image-20251019004204084](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004204084.png)

##### 4.4.2. 上传作业的zip包

进入到DSS页面，新建工程后，选择“流式生产中心”进入，点击“导入”，选择文件系统中打包好的zip文件进行上传

![job_import](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/job_import.png)

如果上传zip文件出现下面错误，请调整下nginx的配置`vi /etc/nginx/conf.d/streamis.conf`，添加属性`client_max_body_size`，如下图所示。 

![image-20251019004222513](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004222513.png)

![upload_jobtask_error_solve](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/upload_jobtask_error_solve.png)

在streamis中将该zip包导入，导入任务后，任务的运行状态变成"未启动"，导入的新作业版本从v00001开始，最新发布时间会更新至最新时间。

### 5. 工程资源文件

Streamis首页-核心指标右上角-工程资源文件。 工程资源文件提供了上传和管理项目所需资源文件的功能，如下图所示：

![image-20251019004300897](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004300897.png)

上传项目文件

![project_source_file_import](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/project_source_file_import.png)

### 6. Streamis作业介绍

点击”作业名称“,可查看任务的详情，包括，运行情况、执行历史、配置、任务详情、告警等。

#### 6.1 运行情况

![image-20251019004331563](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004331563.png)

#### 6.2 执行历史

打开执行历史可以查看该任务的历史运行情况，

历史日志：只有正在运行的任务才能查看历史日志。

![image-20251019004341921](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004341921.png)

历史日志中可以查看当前任务启动的flink引擎的日志，也可以查看当前任务启动的Yarn日志，可以根据关键字等查看关键日志，点击查看最新日志，可以查看当前引擎的最新日志。

![job_log1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_log1.png)

![image-20251019004359559](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004359559.png)

#### 6.3 配置

给Streamis任务配置一些flink资源参数以及checkpoint的参数

![image-20251019004426944](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004426944.png)

![image-20251019004439369](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004439369.png)

#### 6.4任务详情

任务详情根据任务类型Flink Jar 和 Flink SQL分为两种显示界面。

- **Flink Jar任务详情**

![image-20251019004500580](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004500580.png)

Flink Jar任务详情展示了任务Jar包的内容和参数， 同时提供下载该Jar包的功能。

- **Flink SQL任务详情**

![任务详情](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_job_flinksql_jobcontent.png)

Flink SQL任务详情展示了该任务的SQL语句。

#### 6.5 进入Yarn页面

正在运行的Streamis任务可以通过该按钮进入到yarn管理界面上的查看flink任务运行情况。

![image-20251019004517005](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004517005.png)

### 7. Streamis作业生命周期管理

#### 7.1. 作业启动

导入成功的任务初始状态为“未启动”，点击任务启动，启动完成后会刷新该作业在页面的状态为“运行中”

![image-20251019004532973](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004532973.png)

![job_start2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_start2.png)

作业启动会进行前置信息检查，会检查作业的版本信息和快照信息，当作业有检查信息需要用户确认时，则会弹出弹窗；对于批量重启的场景，可以勾选“确认所以批量作业”进行一次性确认

![image-20251019004548330](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004548330.png)

#### 7.2. 作业停止

点击“停止”，可以选择“直接停止”和“快照并停止”，“快照并停止”会生成快照信息，并展示出来

![image-20251019004613431](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004613431.png)

#### 7.3. 保存快照功能

点击相应的作业名称、配置或左边3个竖点中（参数配置/告警配置/运行历史/运行日志）可进入job任务详情，点击 启动 可执行作业

点击作业左边的3个竖点中的“快照【savepoint】 ”可保存快照

![savepoint1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/savepoint1.png)

![image-20251019004628570](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004628570.png)

#### 7.4. 批量操作功能

点击“批量操作”，可选中多个作业任务重启，快照重启会先生成快照再重新启动，直接重启不会生成快照

![image-20251019004642609](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004642609.png)

#### 7.5. 作业启用禁用功能

支持对单个作业启用/禁用或多个作业批量启用/禁用；

用户可以在页面对任务进行禁用启用：

![image-20251019004655374](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004655374.png)

![image-20251019004706268](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004706268.png)

可以筛选已启用、已禁用和全部job：

![image-20251019004717366](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004717366.png)

已禁用的job无法进行启动等操作：

![image-20251019004726948](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004726948.png)

### 8. Streamis版本新增特性

#### 8.1.支持修改args参数

- 1、进入任务详情页 

![image-20251019004746957](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004746957.png)

- 2、点击编辑 

![image-20251019004804553](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004804553.png)

- 3、点击确认，保存   注意事项：   1、编辑arguement超长需要小于400长度   2、编辑格式需要注意，下面是示例：

```
1、编辑arguement为[ "" ]确认
2、编辑arguement为[ ] 确认
3、编辑arguement为空，确认
4、编辑arguement为"" 确认
5、编辑arguement为[ "--kafka.bootstrap.servers": ""; "--user": "test", "--kafka.topic", "" ]，确认
6、编辑arguement为 "--kafka.bootstrap.servers": ""; "--user": "", "--kafka.topic", ""，确认 
【预期结果】
1-2-5、成功
3-4-6、失败，页面参数保持不变 
```

#### 8.2. Streamis支持按照任务自定义log4j文件

​	0.3.2版本对配置文件加载顺序进行改造，优化flink启动脚本，在linkis默认的管理体系下，flink启动首先加载hadoop-config目录下Log4j配置文件，由平台统一管理。

##### 使用方式

**①首先在工程资源文件上传log4j.properties**

![image-20251019004838456](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004838456.png)

**②在meta.json 配置文件中添加参数 "resources": [log4j.properties]**

```
  "jobContent": {
    "main.class.jar": "demo-stream-app-0.1.3-SNAPSHOT.jar",
    "main.class": "com.webank.wedatasphere.stream.apps.core.entrance.StreamAppEntrance",
    "args": [
      "--kafka.bootstrap.servers",
      "10.108.192.49:9092,10.108.192.59:9092,10.108.192.75:9092",
      "--user",
      "hadoop",
      "--kafka.topic",
      "cfpd_rcs_ef_risk_bdap"
    ],
    "hdfs.jars": [],
    "resources": [log4j.properties]
  },
```

#### 8.3. 标签筛选

支持继续标签筛选任务

![image-20251019004853995](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004853995.png)

#### 8.4. 标签批量修改

先点批量修改，然后选中多个任务，点 修改标签。在弹出窗口输入新标签内容，支持大小写字母、数字、逗号、下划线。

![image-20251019004912196](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004912196.png)

 ![3.2](/Users/glexios/Library/Mobile%2520Documents/com~apple~CloudDocs/Notes/%25E9%25A1%25B9%25E7%259B%25AE%25E6%2595%25B4%25E7%2590%2586.assets/3.2.png)

#### 8.5. 日志回调使用说明

1、flink应用在启动会先向streamis发起注册，注册数据如下图所示 ![logs_call_back_01.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/logs_call_back_01.png)

2、定时任务：flink应用使用 Java 中的定时任务工具来定期发送心跳消息。上图中的heartbeat_time为记录的心跳时间

3、streamis新增定时任务检测应用，streamis告警信息如下图所示 

![image-20251019004933811](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019004933811.png)

## Steamis产品截图

**【生产中心】**

![image-20251019005000233](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005000233.png)

**【运行情况】**

![image-20251019005011935](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005011935.png)

**【运行历史】**

![image-20251019005023437](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005023437.png)

 **【流式应用参数配置】**

![image-20251019005034838](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005034838.png)

![image-20251019005049470](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005049470.png)

![image-20251019005107209](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005107209.png)

![企业微信截图_17424604676047](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424604676047.png)

![企业微信截图_17424604797635](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424604797635.png)

![企业微信截图_17424605021312](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424605021312.png)

**【任务详情】**

![image-20251019005130073](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005130073.png)

![企业微信截图_17424606762072](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424606762072.png)

**【告警】**

![企业微信截图_17424607188533](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424607188533.png)

**【物料】**

![image-20251019005146011](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005146011.png)

**【审计日志】**

![image-20251019005158200](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005158200.png)

## 流式数仓任务-配置化数据集成

**（1）背景**

- 策略分析平台策略模型数据目前使用离线批量同步，T+1时效
- 现有的BINLOG流式数据同步方案时效小时级，存在较多的可用性问题，另外每个新需求也需要较大版本人力开发，响应慢。每个规则一个应有，无法资源复用
- 基于以上背景，希望实现一套低延时、可用性高、资源消耗低、人力成本低的配置化数据集成降本增效方案，降低数据同步需求需求的开发人力和资源小号。

![image-20251019005216754](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005216754.png)

- 绿色线条为数据流动方向。
- 红色框为现存链路存在问题：
  - Binlog采集大数据记录丢失
  - Binlog采集元数据丢失。元数据变更不够自动化。
  - DM数据提单，过多手动补充的内容
  - 需要定时任务，根据binlog类型，来执行相应数据操作。

**（2）现有方案存在的问题**

①新表要提新单。提单过程复杂，每个需要科技排期开发上线。且每个新表一个任务，资源占用大。

②小时级延迟

③数据链路长，可用性存在问题

- kafka抖动存在丢数可能
- 仅支持单活，无高可用支持

④kafka限制topic大小为1MB，超过的消息都丢弃

## application.properties配置

```
server.port=9400
spring.application.name=streamis-server
spring.mvc.servlet.path=/api/rest_j/v1
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=1024MB
spring.servlet.multipart.max-request-size=1024MB
spring.main.allow-bean-definition-overriding=true

eureka.client.serviceUrl.defaultZone=[@GLOBAL_LINKIS_GZ_FLOWMAIN_UAT_EUREKA_URL]
eureka.instance.metadata-map.test=wedatasphere

management.endpoints.web.exposure.include=refresh,info
logging.config.classpath=log4j2.xml

feature.enabled=true

# after linkis 1.6.0
#spring.main.allow-circular-references=true
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
spring.cloud.loadbalancer.cache.enabled=false
```

**1. 基础服务配置**

- **`server.port=9400`**
  - **作用** ：指定应用的HTTP服务端口为 `9400`，即Spring Boot服务监听的端口号为 `9400`，客户端通过此端口访问服务。。
  - **场景** ：避免端口冲突，明确服务暴露的端口。
- **`spring.application.name=streamis-server`**
  - **作用** ：定义应用名称为 `streamis-server`。
  - **场景** ：服务注册到Eureka时，该名称作为服务唯一标识（Service ID）；日志标识

**2.请求路径与文件上传配置**

- **`spring.mvc.servlet.path=/api/rest_j/v1`**
  - **作用** ：设置Spring MVC的Servlet路径前缀为 `/api/rest_j/v1`。
  - **框架** ：Spring MVC。
  - **场景** ：统一API版本控制，所有接口需以 `/api/rest_j/v1` 开头（如 `/api/rest_j/v1/users`）。
- **`spring.servlet.multipart.enabled=true`**
  - **作用** ：启用对 `multipart/form-data` 请求的支持（如文件上传）。
  - **框架** ：Spring Boot。
  - **场景** ：允许客户端通过HTTP上传文件。
- **`spring.servlet.multipart.max-file-size=1024MB`**
  - **作用** ：限制单个上传文件的最大大小为 `1GB`。
  - **框架** ：Spring Boot。
  - **场景** ：防止大文件上传耗尽服务器资源。
- **`spring.servlet.multipart.max-request-size=1024MB`**
  - **作用** ：限制整个HTTP请求的最大大小为 `1GB`（包括文件和其他表单数据）。
  - **框架** ：Spring Boot。
- **场景** ：防止恶意请求或意外的大请求导致内存溢出。

**3. Bean管理与依赖注入**

- **`spring.main.allow-bean-definition-overriding=true`**
  - **作用** ：允许覆盖已存在的Bean定义。
  - **框架** ：Spring Boot。
  - **场景** ：在多模块项目或自动配置中，显式覆盖默认Bean（如自定义配置类）。

**解释**：当Spring容器中存在**同名Bean** 时，此配置允许后加载的Bean定义**覆盖**先前的定义。

Eg：若两个配置类分别定义了名为 `userService` 的Bean，后加载的Bean会替换已存在的定义

- **默认行为** ： Spring Boot 2.1.0+ 默认禁止覆盖（`spring.main.allow-bean-definition-overriding=false`），以避免意外覆盖导致的不可预测行为

**4. 服务发现（Eureka）配置**

- **`eureka.client.serviceUrl.defaultZone=[@GLOBAL_LINKIS_GZ_FLOWMAIN_UAT_EUREKA_URL]`**
  - **作用** ：指定Eureka服务注册中心的地址。
  - **框架** ：Spring Cloud Netflix Eureka。
  - **场景** ：服务启动时向该地址注册自身，并发现其他服务。`[@...]` 是占位符，需在部署时替换为实际URL（如 `http://eureka-server:8761/eureka`）。
- **`eureka.instance.metadata-map.test=wedatasphere`**
  - **作用** ：为当前服务实例添加元数据 `test=wedatasphere`。
  - **框架** ：Spring Cloud Netflix Eureka。
  - **场景** ：用于服务分类或自定义路由策略（如灰度发布）。

**5. 监控与管理端点**

- **`management.endpoints.web.exposure.include=refresh,info`**
  - **作用** ：暴露Spring Boot Actuator的 `refresh` 和 `info` 端点。
  - **框架** ：Spring Boot Actuator。
  - 场景 ，便于运维管理：
    - `/actuator/refresh`：动态刷新配置（需配合 `@RefreshScope`）。
    - `/actuator/info`：返回应用自定义信息（如版本号）。
- **`/actuator/refresh`**
  - 作用 ：**动态刷新**应用的配置（如从配置中心拉取最新配置），无需重启服务
- 使用条件 ：
  - 需配合 `@RefreshScope` 注解（通常标注在需要动态刷新配置的Bean上）。
  - 适用于集成Spring Cloud Config等配置中心的场景。

**6. 日志配置**

- **`logging.config.classpath=log4j2.xml`**
  - **作用** ：指定Log4j2的配置文件路径为类路径下的 `log4j2.xml`。
  - **框架** ：Log4j2。
  - **场景** ：自定义日志输出格式、级别和目标（如文件、控制台）。

**7. 自定义功能开关**

- **`feature.enabled=true`**
  - **作用** ：自定义配置项，控制特定功能的启用/禁用。
  - **框架** ：无（应用自定义）。
  - **场景** ：通过 `@Value("${feature.enabled}")` 注入到代码中，动态开启/关闭功能（如实验性特性）。

**8. 兼容性配置**

- **`#spring.main.allow-circular-references=true`**
  - **作用** ：允许Spring容器中的循环依赖（被注释，未生效）。
  - **框架** ：Spring Boot（2.6+ 默认禁止循环引用）。
  - **场景** ：若代码中存在循环依赖（如A依赖B，B依赖A），需取消注释以允许。
- **`spring.mvc.pathmatch.matching-strategy=ant_path_matcher`**
  - **作用** ：使用Ant风格路径匹配（而非默认的PathPatternParser）。
  - **框架** ：Spring MVC。
  - **场景** ：解决路径匹配兼容性问题（如旧版API的 `**` 通配符行为）。

**9. 负载均衡配置**

- **`spring.cloud.loadbalancer.cache.enabled=false`**
  - **作用** ：禁用Spring Cloud LoadBalancer的服务实例缓存。
  - **框架** ：Spring Cloud LoadBalancer。
  - **场景** ：强制每次请求都从Eureka获取最新服务实例列表（适用于动态扩缩容场景）。

## Streamis在linkis Eureka上注册的信息

```
<application>
  <name>STREAMIS-SERVER</name>
  <instance>
    <instanceId>bdpujes110002:streamis-server:9400</instanceId>
    <hostName>bdpujes110002</hostName>
    <app>STREAMIS-SERVER</app>
    <ipAddr>10.107.97.36</ipAddr>
    <status>UP</status>
    <overriddenstatus>UNKNOWN</overriddenstatus>
    <port enabled="true">9400</port>
    <securePort enabled="false">443</securePort>
    <countryId>1</countryId>
    <dataCenterInfo class="com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo">
      <name>MyOwn</name>
    </dataCenterInfo>
    <leaseInfo>
      <renewalIntervalInSecs>30</renewalIntervalInSecs>
      <durationInSecs>90</durationInSecs>
      <registrationTimestamp>1740533480169</registrationTimestamp>
      <lastRenewalTimestamp>1740735379800</lastRenewalTimestamp>
      <evictionTimestamp>0</evictionTimestamp>
      <serviceUpTimestamp>1740533480169</serviceUpTimestamp>
    </leaseInfo>
    <metadata>
      <test>wedatasphere</test>
      <management.port>9400</management.port>
    </metadata>
    <homePageUrl>http://bdpujes110002:9400/</homePageUrl>
    <statusPageUrl>http://bdpujes110002:9400/actuator/info</statusPageUrl>
    <healthCheckUrl>http://bdpujes110002:9400/actuator/health</healthCheckUrl>
    <vipAddress>streamis-server</vipAddress>
    <secureVipAddress>streamis-server</secureVipAddress>
    <isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>
    <lastUpdatedTimestamp>1740533480169</lastUpdatedTimestamp>
    <lastDirtyTimestamp>1739173375007</lastDirtyTimestamp>
    <actionType>ADDED</actionType>
  </instance>
</application>
```

**1. 应用基本信息**

**`<name>STREAMIS-SERVER</name>`**

- **含义** ：应用名称，对应 `spring.application.name=streamis-server` 的配置。
- **注意** ：Eureka 默认将应用名转为大写（如 streamis-server → STREAMIS-SERVER）。
- **用途** ：服务发现时通过此名称调用（如 Ribbon 客户端负载均衡）

**2. 实例详情**

![image-20250313144527764](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144527764.png)

**（1）`<instanceId>bdpujes110002:streamis-server:9400</instanceId>`**

- **含义** ：实例唯一标识，格式为 `主机名:应用名:端口`。

- **自定义方式** ：可通过 `eureka.instance.instance-id` 修改，例如：

  `eureka.instance.instance-id=${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}`

**（2）`<hostName>bdpujes110002</hostName>`**

- **含义** ：实例所在主机的名称（通过 `hostname` 命令获取）。
- **问题排查** ：若服务调用失败，需检查主机名是否可被其他服务解析（DNS或本地hosts配置

**（3）`<status>UP</status>`**

- **含义** ：实例状态为“运行中”。其他可能状态包括 `DOWN`（健康检查失败）、`OUT_OF_SERVICE`（手动下线）等。
- **状态更新** ：通过 `healthCheckUrl`（`/actuator/health`）的返回值决定

**3. 端口与安全配置**

![image-20250313144544987](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144544987.png)

**（1）`<port enabled="true">9400</port>`**

- **含义** ：非SSL端口（HTTP）为 `9400`，与 `server.port=9400` 配置一致。
- **作用** ：服务间通信默认使用此端口

**（2）`<securePort enabled="false">443</securePort>`**

- **含义** ：SSL端口（HTTPS）为 `443`，但未启用（`enabled="false"`）。
- **启用方法** ：需配置SSL证书并设置 `server.ssl.*` 参数

**4. 租约与健康检查** **`<leaseInfo>`**

![image-20250313144601667](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144601667.png)

**（1）续约机制**

- `renewalIntervalInSecs=30` ：客户端每30秒向Eureka Server发送一次心跳
- `durationInSecs=90` ：若Eureka Server在90秒内未收到心跳，将剔除该实例

**（2）时间戳**

- `registrationTimestamp`：实例注册时间（Unix毫秒时间戳）。
- `lastRenewalTimestamp`：最后一次心跳时间。

**5. 元数据（Metadata）**

![image-20250313144752621](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144752621.png)

**（1）`<metadata><test>wedatasphere</test></metadata>`**

- **含义** ：自定义元数据 `test=wedatasphere`，通过 `eureka.instance.metadata-map.test=wedatasphere` 配置。
- **用途** ：可用于灰度发布、路由策略或监控分类

**（2）`<management.port>9400</management.port>`**

- **含义** ：Actuator管理端点的端口（与主应用端口一致）。
- **作用** ：通过 `/actuator/info` 和 `/actuator/health` 提供元数据和健康状态

**6. 服务发现与调用**

![image-20250313145153771](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313145153771.png)

**（1）`<vipAddress>streamis-server</vipAddress>`**

- **含义** ：虚拟IP地址（VIP），通常与应用名一致。
- 用途 ：客户端通过VIP（如 `http://streamis-server/api/...`）调用服务，Ribbon会自动解析为具体实例

**（2）`<homePageUrl>` 和 `<statusPageUrl>`**

- **`homePageUrl`** ：应用根路径（`http://bdpujes110002:9400/`）。
- **`statusPageUrl`** ：健康状态页面（`/actuator/info`），返回自定义信息（如版本号）

**7. 其他关键字段**

![image-20250313145314173](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313145314173.png)

- **`<isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>`** ：标识该实例是否为Eureka Server节点（`false` 表示是普通服务实例）。
- **`<actionType>ADDED</actionType>`** ：实例注册到Eureka Server的动作类型（`ADDED` 表示首次注册）。

## Streamis服务启动脚本（包括jvm参数和启动命令）

### 注释版脚本

```shell
#!/bin/bash

##################
#1.路径与环境准备
##################
cd `dirname $0`  # 进入脚本所在目录
cd ..            # 返回上一级目录（项目根目录）
export HOME=`pwd` # 设置 HOME 环境变量为当前路径（项目根目录）
source $HOME/conf/stms-env.sh # 加载环境配置文件（定义 STREAMIS_HOME 等变量）

#####################
#2. 配置路径与日志目录
#####################
export STREAMIS_CONF_PATH=${STREAMIS_CONF_DIR:-$STREAMIS_HOME/conf} # 配置文件路径，默认为 $STREAMIS_HOME/conf
export STREAMIS_LOG_PATH=${STREAMIS_LOG_DIR:-$STREAMIS_HOME/logs}  # 日志文件路径，默认为 $STREAMIS_HOME/logs
mkdir -p $STREAMIS_LOG_PATH # 创建日志目录（如果不存在）

#####################
#3. 检查服务是否已运行
#####################
if [[ -f "${STREAMIS_PID}" ]]; then # 检查 PID 文件是否存在（记录进程ID）
    pid=$(cat ${STREAMIS_PID})      # 读取 PID 文件中的进程ID
    if kill -0 ${pid} >/dev/null 2>&1; then # 验证进程是否存活
      echo "Streamis Server is already running." # 若存活，提示服务已运行
      return 0; # 退出脚本
    fi
fi

#####################
#4. 调试与安全代理配置
#####################
if [ "$DEBUG_PORT" ]; then # 若定义了 DEBUG_PORT 环境变量
   export DEBUG_CMD="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=$DEBUG_PORT" # 配置远程调试参数
fi
# IAST 安全代理配置（用于漏洞检测）
export IAST_AGENT_JAR=/appcom/Install/IastInstall/vulhunter.jar 
if [[ -e "$IAST_AGENT_JAR" && -e $IAST_AGENT_JAR ]]; then # 检查代理 JAR 是否存在
    export JAVA_AGENT_OPTS="-javaagent:$IAST_AGENT_JAR=app.full.name=BDP-STREAMIS,flag=iast,vul.log.level=INFO" # 添加 Java 代理参数
fi

#####################
#5. JVM 参数与启动命令
#####################
startTime=$(date "+%Y%m%d%H") # 生成时间戳（用于日志文件名）

# JVM 参数配置
export STREAMIS_HEAP_SIZE="3G" # 堆内存大小
export SERVER_NAME=streamis-server # 服务名称
export STREAMIS_JAVA_OPTS="
-Xms$STREAMIS_HEAP_SIZE -Xmx$STREAMIS_HEAP_SIZE # 初始与最大堆内存
-XX:+UseG1GC # 使用 G1 垃圾回收器
-XX:MaxPermSize=500m # 永久代大小（仅适用于 Java 8）
-Xloggc:$STREAMIS_LOG_PATH/${SERVER_NAME}-${startTime}-gc.log # GC 日志路径
-XX:+PrintGCDateStamps # 打印 GC 时间戳
-XX:+HeapDumpOnOutOfMemoryError # 内存溢出时生成堆转储
-XX:HeapDumpPath=$STREAMIS_LOG_PATH/${SERVER_NAME}-${startTime}-dump.log # 堆转储路径
-XX:ErrorFile=$STREAMIS_LOG_PATH/${SERVER_NAME}-${startTime}-hs_err.log # JVM 错误日志路径
$JAVA_AGENT_OPTS $DEBUG_CMD # 安全代理与调试参数
"
# 启动命令
nohup java $STREAMIS_JAVA_OPTS \ # 启动 Java 进程，忽略挂断信号
-cp $STREAMIS_CONF_PATH:$HOME/lib/* \ # 类路径：配置文件与依赖库
com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication \ # 主类
2>&1 > $STREAMIS_LOG_PATH/streamis.out & # 重定向标准输出与错误到日志文件

pid=$! # 获取后台进程的 PID

#####################
#6. 启动状态检查
#####################
if [[ -z "${pid}" ]]; then # 如果 PID 为空
    echo "Streamis Server start failed!" # 启动失败
    sleep 1
    exit 1
else
    echo "Streamis Server start succeeded!" # 启动成功
    echo $pid > $STREAMIS_PID # 将 PID 写入文件
    sleep 1
fi
exit 0 # 正常退出
export STMS_VERSION=[@GLOBAL_STREAMIS_VERSION]

# streamis-server jvm heap
export SERVER_HEAP_SIZE="20G"

# uncomment the following line to enable debugging
#export DEBUG_PORT=15005

export STREAMIS_HOME=$HOME

export STREAMIS_PID=$HOME/bin/linkis.pid

# conf path
export STREAMIS_CONF_DIR=${STREAMIS_CONF_DIR:-"$STREAMIS_HOME/conf"}

# log path
export STREAMIS_LOG_DIR=${STREAMIS_LOG_DIR:-/data/bdp/logs/streamis}
```

### 部分脚本详细解释

**（1）第三步检查服务是否已运行**

```
#####################
#3. 检查服务是否已运行
#####################
if [[ -f "${STREAMIS_PID}" ]]; then # 检查 PID 文件是否存在（记录进程ID）
    pid=$(cat ${STREAMIS_PID})      # 读取 PID 文件中的进程ID
    if kill -0 ${pid} >/dev/null 2>&1; then # 验证进程是否存活
      echo "Streamis Server is already running." # 若存活，提示服务已运行
      return 0; # 退出脚本
    fi
fi
```

**1）`kill -0 ${pid}`**

- **作用** ：检查进程是否存在
  - **`kill -0`** 不会真正终止进程，而是向进程发送一个空信号（Signal 0），仅**用于检测进程是否存活**
  - 如果进程存在且当前用户有权限访问，则返回 **0** （成功）；否则返回非零值（如 `1`）
- **变量 `${pid}`** ：通过 `pid=$(cat ${STREAMIS_PID})` 从 PID 文件中读取的进程 ID。

**2） `>/dev/null 2>&1`**

- **作用**：抑制命令的输出。
  - **`>/dev/null`：将标准输出（stdout）重定向到空设备（丢弃输出）。**
  - **`2>&1`：将标准错误（stderr）重定向到标准输出的位置（即同样丢弃错误信息）**
- **目的** ：避免脚本执行时输出无关信息（如 `kill: No such process`）。

**（2）第四步调试与安全代理配置**

```
#####################
#4. 调试与安全代理配置
#####################
if [ "$DEBUG_PORT" ]; then # 若定义了 DEBUG_PORT 环境变量
   export DEBUG_CMD="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=$DEBUG_PORT" # 配置远程调试参数
fi
# IAST 安全代理配置（用于漏洞检测）
export IAST_AGENT_JAR=/appcom/Install/IastInstall/vulhunter.jar 
if [[ -e "$IAST_AGENT_JAR" && -e $IAST_AGENT_JAR ]]; then # 检查代理 JAR 是否存在
    export JAVA_AGENT_OPTS="-javaagent:$IAST_AGENT_JAR=app.full.name=BDP-STREAMIS,flag=iast,vul.log.level=INFO" # 添加 Java 代理参数
fi
```

**1）调试配置（DEBUG_PORT）**

- **`export DEBUG_CMD="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=$DEBUG_PORT"`** ：
  - **内容** ：设置 Java 远程调试参数。
  - **参数详解** ：
    - `transport=dt_socket`：使用套接字传输。
    - `server=y`：作为调试服务器（等待客户端连接）。
    - `suspend=n`：启动时不暂停 JVM（若设为 `y`，则等待调试器连接后再启动）。
    - `address=$DEBUG_PORT`：指定调试端口（如 `5005`）



**Java 调试参数`-agentlib:jdwp`** 

- 启用 JDWP（Java Debug Wire Protocol），允许 IDE（如 IntelliJ、Eclipse）远程调试 JVM

**2）IAST 安全代理配置**

- **`export JAVA_AGENT_OPTS="-javaagent:$IAST_AGENT_JAR=app.full.name=BDP-STREAMIS,flag=iast,vul.log.level=INFO"`** ：
  - **内容** ：设置 Java 代理参数。
  - **参数详解** 
    - `-javaagent:$IAST_AGENT_JAR`：指定代理 JAR 文件。
    - `app.full.name=BDP-STREAMIS`：应用名称（用于漏洞报告）。
    - `flag=iast`：启用 IAST 模式（交互式应用安全测试）。
    - `vul.log.level=INFO`：设置漏洞日志级别为 INFO

**（3）JVM关于内存溢出的参数**

```
XX:+HeapDumpOnOutOfMemoryError # 内存溢出时生成堆转储
-XX:HeapDumpPath=$STREAMIS_LOG_PATH/${SERVER_NAME}-${startTime}-dump.log # 堆转储路径
-XX:ErrorFile=$STREAMIS_LOG_PATH/${SERVER_NAME}-${startTime}-hs_err.log # JVM 错误日志路径
```

**1）`-XX:+HeapDumpOnOutOfMemoryError`**

- 当 JVM 发生 `OutOfMemoryError`（内存溢出）时，自动生成 **堆转储文件（Heap Dump）** ，记录内存快照

**2）`-XX:HeapDumpPath=路径`**：指定堆转储文件的保存路径和文件名

- 路径变量 
  - `$STREAMIS_LOG_PATH`：日志目录（如 `/opt/streamis/logs`）。
  - `${SERVER_NAME}`：服务名称（如 `streamis-server`）。
  - `${startTime}`：时间戳（如 `2024010112`）。
- **生成文件** ：**文件名为 `streamis-server-2024010112-dump.log.hprof`（JVM 会自动添加 `.hprof` 后缀）**

**3）`-XX:ErrorFile=路径`**

- 当 JVM 发生致命错误（如崩溃、`OutOfMemoryError`）时，生成 错误日志文件（hs_err_pid* .log），记录详细错误信息
- **生成文件** ：文件名为 `streamis-server-2024010112-hs_err.log`，包含以下内容：
  - JVM 版本、系统环境。
  - 内存使用状态、线程堆栈跟踪。
  - 寄存器状态（Native 层错误）

**【例子】**

**触发条件** 

- 应用持续占用内存，GC 无法回收，最终抛出 `OutOfMemoryError`。

**生成文件** 

- **堆转储文件** ：`streamis-server-2024010112-dump.log.hprof`（分析内存对象）。
- **错误日志** ：`streamis-server-2024010112-hs_err.log`（查看崩溃时的线程和内存状态）。

**分析工具** ：

- **使用jvisualvm分析堆转储**，定位内存泄漏点

  - ```
    jvisualvm # 启动 VisualVM，直接加载 `.hprof` 文件
    ```

- **JVM 日志分析器** ：解析 `hs_err` 文件，检查 Native 层问题

**（4）第五步 JVM 参数与启动命令**

```
# 启动命令
nohup java $STREAMIS_JAVA_OPTS \ # 启动 Java 进程，忽略挂断信号
-cp $STREAMIS_CONF_PATH:$HOME/lib/* \ # 类路径：配置文件与依赖库
com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication \ # 主类
2>&1 > $STREAMIS_LOG_PATH/streamis.out & # 重定向标准输出与错误到日志文件
```

**1）`nohup`：后台运行与进程持久化**

- **忽略挂断信号** ：即使终端（SSH会话）关闭，进程仍继续运行。
- **输出重定向** ：默认将标准输出和错误输出到 `nohup.out` 文件（此处被显式重定向到自定义日志文件）。

**2）`java $STREAMIS_JAVA_OPTS`：JVM 参数配置**

**3）`-cp $STREAMIS_CONF_PATH:$HOME/lib/\*`：类路径配置**

```
-cp $STREAMIS_CONF_PATH:$HOME/lib/*
```

- 作用 
  - **`-cp`（或 `-classpath`）** ：**指定 JVM 查找类文件（`.class`）和依赖库（`.jar`）的路径**。
- 路径解析 
  - **`$STREAMIS_CONF_PATH`** ：指向配置文件目录（如 `conf/`），包含 `application.properties` 等配置。
  - **`$HOME/lib/\*`** ：包含所有依赖的 JAR 文件（如 Spring Boot、Linkis 客户端等）。
  - 在 **Linux/Unix 系统** 中，`:` 是 **类路径分隔符** ，用于分隔多个路径条目，**意思是conf和lib是两个不同的目录**

**4）`com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication`：主类**

- 作用 
  - Spring Boot 应用的入口类，包含 `main` 方法启动应用。
- 执行流程：
  1. 加载 `SpringApplication`，读取 `@SpringBootApplication` 注解配置。
  2. 扫描组件（如 `@Controller`、`@Service`），初始化 Spring 容器。
  3. 启动嵌入式 Web 服务器（如 Tomcat），监听 `server.port=9400`。

**5）`2>&1 > $STREAMIS_LOG_PATH/streamis.out &`：输出重定向与后台运行**

```
2>&1 > $STREAMIS_LOG_PATH/streamis.out &
```

- **分解说明** 
  - **`>`** ：重定向标准输出（`stdout`）到文件 `streamis.out`。
  - **`2>&1`** ：将标准错误（`stderr`）重定向到标准输出（即**合并到 `streamis.out`**）。
    - 在 Linux/Unix 系统中，每个进程默认有三个标准文件描述符：**`0`** ：标准输入（stdin）**`1`** ：标准输出（stdout）**`2`** ：标准错误（stderr）
    - **`2>`** ：表示将文件描述符 `2`（标准错误）的输出重定向。
    - **`&1`** ：表示目标是文件描述符 `1`（标准输出）的位置。
  - **`&`** ：将进程放入后台运行。
- 日志文件命名 
  - `streamis.out` 包含应用日志（如启动日志、未捕获的异常）。
  - GC 日志、堆转储等通过 `STREAMIS_JAVA_OPTS` 单独指定路径。

## Streamis的Nginx配置

```
    server {
            listen       9188;# 访问端口
            server_name  uat.stms.bdap.weoa.com; # 域名绑定
            client_max_body_size 1024M; # 允许客户端上传最大 1GB 文件
            #charset koi8-r;
            #access_log  /var/log/nginx/host.access.log  main;
            location / {
             root   /appcom/Install/streamis-web; # 静态文件目录
             index  index.html index.html; # 默认首页文件
            }
            location /streamis-next {
             root   /appcom/Install; # 静态文件目录
             index  index.html index.html;
            }

            location /ws {
#              proxy_pass http://10.107.119.46:9001;#后端Linkis的地址
             proxy_pass [@GLOBAL_LINKIS_GZ_BDAP_UAT_GATEWAY_URL];  # 反向代理到 Linkis 网关
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade; # 支持 WebSocket 升级
             proxy_set_header Connection upgrade;
            }

            location /api {
#              proxy_pass http://10.107.119.46:9001; #后端Linkis的地址
             proxy_pass [@GLOBAL_LINKIS_GZ_BDAP_UAT_GATEWAY_URL]; # 反向代理到 Linkis 网关
             proxy_set_header Host $host; # 传递原始 Host 头
             proxy_set_header X-Real-IP $remote_addr; # 客户端真实 IP
             proxy_set_header remote_addr $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;# 代理链记录
             proxy_http_version 1.1;
             proxy_connect_timeout 4s;# 连接超时时间
             proxy_read_timeout 600s;# 读取超时（长任务需较大值
             proxy_send_timeout 12s; # 发送超时
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection upgrade;
            }

            #error_page  404              /404.html;
            # redirect server error pages to the static page /50x.html
            #
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
              root   /usr/share/nginx/html; # 默认错误页面路径
            }
        }
    server {
          listen 9400;
          server_name localhost;
          location / {
            proxy_pass http://10.107.119.46:9321;# 反向代理到本地服务
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection upgrade;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header x_real_ipP $remote_addr;
            proxy_set_header remote_addr $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }
    }
```

当 **Streamis 服务注册到 Linkis 的 Eureka** 后，调用其接口的 URL 通常遵循以下格式：

- `http://<Linkis-Gateway-Address>/api/rest_j/v1/streamis/<具体接口路径>`

**eg：`http://linkis-gateway:9001/api/rest_j/v1/streamis/streamJobManager/job/upload`**

- **`linkis-gateway:9001`** ：Linkis 网关地址（通过 Eureka 发现服务）。
- **`/api/rest_j/v1`** ：Linkis REST API 的统一前缀
- **`/streamis`** ：Streamis 服务的专属路径前缀。
- **`/streamJobManager/job/upload`** ：具体接口路径（如上传作业）

**Linkis 网关路由规则**

- Linkis 网关通过 `api` 模块的 `rest_j` 路径统一暴露 REST 接口。所有微服务（包括 Streamis）的接口均通过该前缀路由。

**Streamis 接口定义**

- Streamis 的 Controller 通常使用 `@RequestMapping("/streamis")` 作为基础路径，例如：

  ```java
  @RestController
  @RequestMapping("/streamis/streamJobManager")
  public class StreamJobController {
      @PostMapping("/job/upload")
      public Response uploadJob(...) { ... }
  }
  ```

**调用流程验证**

- **服务发现** ：Streamis 注册到 Eureka 后，Linkis 网关通过服务名（如 `STREAMIS-SERVER`）发现其地址
- **路由转发** ：网关将 `/api/rest_j/v1/streamis/` 的请求转发到 Streamis 服务
- **接口执行** ：Streamis 处理请求并返回结果（如作业上传状态）

## Streamis的日志配置

### Streamis日志规范

【**约定**】Streamis 选择引用 Linkis Commons 通用模块，其中已包含了日志框架，主要**以 slf4j 和 Log4j2 作为日志打印框架**，去除了Spring-Cloud包中自带的logback。 由于Slf4j会随机选择一个日志框架进行绑定，所以以后在引入新maven包的时候，需要将诸如slf4j-log4j等桥接包exclude掉，不然日志打印会出现问题。但是如果新引入的maven包依赖log4j等包，不要进行exclude，不然代码运行可能会报错。

【**配置**】log4j2的配置文件默认为 log4j2.xml ，需要放置在 classpath 中。如果需要和 springcloud 结合，可以在 application.yml 中加上 logging:config:classpath:log4j2-spring.xml (配置文件的位置)。

【**强制**】类中不可直接使用日志系统（log4j2、Log4j、Logback）中的API。

- 如果是Scala代码，强制继承Logging trait
- java采用 LoggerFactory.getLogger(getClass)。

【**强制**】严格区分日志级别。其中：

- Fatal级别的日志，在初始化的时候，就应该抛出来，并使用System.out(-1)退出。
- ERROR级别的异常为开发人员必须关注和处理的异常，不要随便用ERROR级别的。
- Warn级别是用户操作异常日志和一些方便日后排除BUG的日志。
- INFO为关键的流程日志。
- DEBUG为调式日志，非必要尽量少写。

【**强制**】要求：INFO级别的日志，每个小模块都必须有，关键的流程、跨模块级的调用，都至少有INFO级别的日志。守护线程清理资源等必须有WARN级别的日志。

【**强制**】异常信息应该包括两类信息：案发现场信息和异常堆栈信息。如果不处理，那么通过关键字throws往上抛出。 正例：logger.error(各类参数或者对象toString + "_" + e.getMessage(), e);

### SLF4J 的角色

**门面模式（Facade）** ：

- SLF4J 是日志接口标准，不提供具体实现，需绑定 Log4j2、Logback 等后端

**示例代码** ：

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass {
    private static final Logger logger = LoggerFactory.getLogger(MyClass.class);
    logger.info("This is an SLF4J log message");
}
```

**与 Log4j2 的关系**

**桥接使用** ：

- SLF4J 可通过 `slf4j-log4j12` 绑定到 Log4j 1.x，或通过 `log4j-slf4j-impl` 绑定到 Log4j2

**对比总结** ：

| **框架** | **角色**     | **依赖关系**                |
| :------- | :----------- | :-------------------------- |
| SLF4J    | 接口（门面） | 需绑定具体实现（如 Log4j2） |
| Log4j2   | 具体实现     | 可直接使用或通过 SLF4J 调用 |

**优先使用 Log4j2 （性能优势） + SLF4J 门面 （解耦代码与实现）**

```
<!-- SLF4J API -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.36</version>
</dependency>
<!-- Log4j2 绑定 SLF4J -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.17.1</version>
</dependency>
```

### Streamis的log4j2.xml配置

```
<?xml version="1.0" encoding="UTF-8"?>
<!--
  ~ Copyright 2021 WeBank
  ~ Licensed under the Apache License, Version 2.0 (the "License");
  ~ you may not use this file except in compliance with the License.
  ~ You may obtain a copy of the License at
  ~
  ~ http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing, software
  ~ distributed under the License is distributed on an "AS IS" BASIS,
  ~ WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  ~ See the License for the specific language governing permissions and
  ~ limitations under the License.
  -->

<configuration status="error" monitorInterval="30">
    <appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%t] %logger{36} %L %M - %msg%xEx%n"/>
        </Console>
        <RollingFile name="RollingFile" fileName="logs/streamis-server.log"
                     filePattern="logs/$${date:yyyy-MM}/streamis-server-log-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%-40t] %c{1.} (%L) [%M] - %msg%xEx%n"/>
            <SizeBasedTriggeringPolicy size="100MB"/>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>
    </appenders>
    <loggers>
        <root level="INFO">
            <appender-ref ref="RollingFile"/>
            <appender-ref ref="Console"/>
        </root>

        <logger name="org.apache.linkis.server.security.SecurityFilter" level="WARN" />
    </loggers>
</configuration>
```

#### 配置文件基础信息

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="error" monitorInterval="30">
```

- **`status="error"`** 
  - 设置 Log4j2 内部日志的输出级别为 `ERROR`，仅记录框架自身的错误信息（避免调试信息干扰）
- **`monitorInterval="30"`**
  - 每 30 秒自动重新加载配置文件（热更新），无需重启服务即可生效新配置

#### 输出方式定义（Appenders）

【**Console（控制台输出）**】

```
<appenders>
  <Console name="Console" target="SYSTEM_OUT">
    <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
    <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%t] %logger{36} %L %M - %msg%xEx%n"/>
  </Console>
```

**作用** ：将日志输出到控制台（标准输出）。

**关键配置** ：

- **`ThresholdFilter`** ：仅允许 `INFO` 及以上级别（WARN、ERROR）的日志输出到控制台
- **`PatternLayout`** ：定义日志格式：
  - **`%d{...}`**：日期时间（如 `2024-01-01 12:34:56.789`）。
  - **`%-5level`**：左对齐的日志级别（如 `INFO `）。
  - **`[%t]`**：线程名（如 `[main]`）。
  - **`%logger{36}`**：日志记录器名称（缩短为36字符）。
  - **`%L`**：代码行号，`%M`方法名（可能影响性能，生产环境慎用）
  - **`%msg%xEx`**：日志消息及异常堆栈

【**RollingFile（滚动文件输出）**】

```
<RollingFile name="RollingFile" fileName="logs/streamis-server.log"
             filePattern="logs/$${date:yyyy-MM}/streamis-server-log-%d{yyyy-MM-dd}-%i.log">
  <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%-40t] %c{1.} (%L) [%M] - %msg%xEx%n"/>
  <SizeBasedTriggeringPolicy size="100MB"/>
  <DefaultRolloverStrategy max="20"/>
</RollingFile>
```

**作用** ：将日志写入文件，并按大小和时间滚动。

**关键配置** ：

- **`fileName`** ：当前日志文件路径（如 `logs/streamis-server.log`）。
- **`filePattern`** ：滚动后的文件名格式：
  - **`$${date:yyyy-MM}`**：按月分目录（如 `logs/2024-01/`）。
  - **`%d{yyyy-MM-dd}`**：文件名包含日期（如 `2024-01-01`）。
  - **`%i`**：当日多次滚动时的索引（如 001）
- **`SizeBasedTriggeringPolicy`** ：单个文件达到 `100MB` 时触发滚动
- **`DefaultRolloverStrategy`** ：最多保留 `20` 个滚动文件（旧文件会被删除）

#### 日志记录器配置（Loggers）

【**根日志记录器（Root Logger）**】

```
<root level="INFO">
  <appender-ref ref="RollingFile"/>
  <appender-ref ref="Console"/>
</root>
```

**作用** ：定义全局日志行为

**配置** ：

- **`level="INFO"`** ：全局日志级别为 `INFO`（输出 INFO、WARN、ERROR 级别的日志）
- **`appender-ref`** ：关联 `RollingFile` 和 `Console`，**日志同时输出到文件和控制台**

【**特定包的日志级别覆盖**】

```xml
<logger name="org.apache.linkis.server.security.SecurityFilter" level="WARN" />
```

**作用** ：针对 `SecurityFilter` 类，将日志级别提升为 `WARN`。

**场景** ：抑制该类的 `INFO` 日志（如安全审计信息），仅记录警告及以上级别

#### 配置总结

| 组件               | 功能                                                         |
| :----------------- | :----------------------------------------------------------- |
| **Console**        | 输出`INFO`及以上日志到控制台，格式包含时间、级别、线程、类名、行号等。 |
| **RollingFile**    | 按`100MB`大小滚动日志，每月归档，最多保留`20`个文件。        |
| **Root Logger**    | 全局日志级别为`INFO`，输出到文件和控制台。                   |
| **SecurityFilter** | 覆盖特定类的日志级别为`WARN`，减少冗余日志。                 |

## Streamis告警

### 用户态告警

目前STREAMIS生产环境上的用户态告警主要有两个，都属于任务状态告警，分别如下:

Flink应用失败重拉告警

- 告警样例：

![企业微信截图_17424574118687](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424574118687.png)

- 告警描述：该告警表明因为一些外部因素导致了Flink流应用的失败，通常会附带自动重拉策略。
- 处理SOP：
  - 如果用户选择了自动重拉策略，则不需要平台运维介入处理，否则平台运维应起到通知业务的作用。

### 服务进程存活性告警

​	统一使用zabbix进行服务进程存活性监控，zabbix管理地址为zabbix.bdp.webank.com，创建一个流式平台监控模板，如下图：

![image-20251019005358838](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005358838.png)

![image-20251019005407754](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005407754.png)

​	其中包括了stms.process.count进程数量检测，通过服务TCP端口${STMS-PORT}进行。 然后再进入生产环境managis(ops.bdp.webank.com)中，在监控管理台-监控模板上绑定监控模板和具体服务：

![image-20251019005419872](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005419872.png)

```
以上的进程存活性告警一旦产生，平台运维则应该参照服务进程启停指南对进程进行恢复。
```

### GC告警

​	统一使用zabbix进行GC监控，zabbix管理地址为zabbix.bdp.webank.com，创建一个流式平台监控模板。

![image-20251019005313859](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005313859.png)

**十分钟内full gc次数大于10次、old区占用大于99%**

- **查看进程的大对象**：使用jps查看进程的pid，然后查看进程的对象信息：

  ```
  jmap -histo $pid
  ```

![image-20251019005436234](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005436234.png)

- 查看服务进程的堆栈信息，判断线程有无block或者死锁:

  ```
  jstack $pid
  ```

![image-20251019005449981](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005449981.png)

- 查看垃圾回收相关信息：gc次数、gc时间等。

  ```
  jstat -gcutil $pid
  ```

![image-20251019005524705](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005524705.png)

## Streamis日志查看

### 查看客户端引擎日志

（1）点击job名，查看详情

![image-20251019005536924](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005536924.png)

（2）点击执行历史页面，选择要查看的历史记录，点历史日志

![image-20251019005605450](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005605450.png) 

（3）默认客户端日志如下

![image-20251019005622265](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005622265.png) 

如果看不到客户端日志，如下图，可尝试复制引擎实例信息

![image-20251019005632030](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005632030.png) 

（4）在运行情况页面，复制客户端引擎实例 如 gz.bdz.bdplinkisrcs05.webank:36616

![image-20251019005650144](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005650144.png) 

（5）点击顶部管理，进入管理台--资源管理--历史引擎管理，输入复制的实例，点搜索

![image-20251019005703647](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005703647.png) 

（6）点查看日志，可查看客户端日志

![image-20251019005716113](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005716113.png)  

如果这里看不到日志，可能是日志被清理或其它原因，请联系管理员协助。

### 查看yarn应用日志（仅应用停止才能看）

在linkis集群上执行，先切换到对应用户：

```
sudo su - hduser03
```

通过applicationId查看对应yarn日志:

```
yarn logs -applicationId application_1666667549351_102687  | less
```

**或者在streamis查看**

类型选yarn日志。注意任务结束后才能查看。如果job执行时间很长，日志可能很大，要等候一段时间才能查看。

![image-20251019005751827](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005751827.png)

### Yarn Flink JobHistory

**1）通过yarn web ui查看**

a. 打开Web UI界面

**`http://{resourcemanager}:8088`**

b. 点击RUNNING、FINISHED

RUNNING：在执行的时候可以看到作业，执行完之后只能在FINISHED里面看到。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps115.jpg) 

**查看YARN作业的日志**

a. 当作业跑完之后，进入FINISHED，点击作业的History，跳转到historyserver 

![image-20251019005814019](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005814019.png) 

 类似如下:

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps117.jpg) 

**2）通过flink页面查看**

打开流式集群yarn web地址

**`http://{resourcemanager}:8088`**

![image-20251019005829781](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005829781.png) 

点击application，输入applicationld，即可查询到对应appld。

![image-20251019005841431](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005841431.png) 

查询到之后，点击Application Master，即可进入flink页面。分别点击jobManager和TaskManager的loglist，可查看运行中日志

![image-20251019005851554](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251019005851554.png) 

### Yarn Flink JobHistory 与 Yarn 的 Flink 日志的区别

- Yarn Flink JobHistory应该指的是Flink在Yarn上运行时生成的作业历史记录，通常由Flink的JobManager生成，记录作业的执行状态、任务信息等。
- 而Yarn的Flink日志可能指的是Yarn管理的各个容器（比如TaskManager）产生的日志，这些日志包括标准输出、错误输出以及用户日志。

**【定义与来源】**

| **类别** | **YARN FLINK JOBHISTORY**                      | **YARN 的 FLINK 日志**                                |
| :------- | :--------------------------------------------- | :---------------------------------------------------- |
| **定义** | Flink 作业的执行历史记录，由 JobManager 生成。 | Yarn 容器（如 TaskManager）的运行日志，由 Yarn 管理。 |
| **来源** | Flink 框架自身（JobManager/TaskManager）       | Yarn 容器（NodeManager）的标准输出/错误输出           |

**【内容与用途】**

| **类别** | **YARN FLINK JOBHISTORY**                                    | **YARN 的 FLINK 日志**                                       |
| :------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **内容** | 作业元数据（如 DAG、任务状态、Checkpoint 信息、任务执行时间等）。 | 容器运行时的标准输出（stdout）、错误日志（stderr）、用户日志（如打印的日志信息）。 |
| **用途** | 作业监控、性能分析、故障诊断（如任务失败原因）。             | 调试 TaskManager 启动失败、OOM 错误、用户代码异常（如空指针）。。 |

**【存储位置与生命周期】**

| **类别**     | **YARN FLINK JOBHISTORY**                                    | **YARN 的 FLINK 日志**                                       |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **存储位置** | 默认存储在 JobManager 的文件系统（如 HDFS）或 Yarn 的日志聚合路径。 | Yarn 容器日志默认存储在 NodeManager 节点本地，可通过 Yarn 日志聚合（Log Aggregation）功能集中到 HDFS。 |
| **生命周期** | 长期保留（取决于配置），用于历史作业分析。                   | 默认短期保留（随容器销毁而删除），需启用 Yarn 日志聚合持久化。 |

## 运行时flink日志回调和关键字告警

**【flink日志回调过滤和告警级别设置】**

采集端增加关键字过滤策略：

增加关键字过滤策略作为默认过滤策略，默认匹配ERROR关键字。

**1）Flink应用，通过streamis管理，任务配置Flink参数：**

日志过滤策略：**stream.log.filter.strategies** （默认值Keyword-关键字, 可选值：LevelMatch-日志等级过滤，ThresholdMatch-日志等级阈值过滤，RegexFilter-日志正则过滤）

1. 选择Keyword-关键字策略：

 匹配关键字：stream.log.filter.keywords（多个关键字用逗号分隔）

 排除关键字：stream.log.filter.keywords.exclude （多个关键字用逗号分隔）

1. 选择LevelMatch-日志等级过滤：

 日志等级：stream.log.filter.level-match.level

1. 选择ThresholdMatch-日志等级阈值过滤

 日志等级阈值： stream.log.filter.threshold.level

1. 选择RegexFilter-日志正则过滤：

 日志正则: stream.log.filter.regex.value



配置好日志过滤，回调地址会自动到streamis服务器上，格式

- /data/stream/log/代理用户名称/项目名称/业务产品/streamis应用名称/yarn应用名称.log

其中yarn应用名= 项目名称.streamis应用名称

- 业务产品为生产配置参数：wds.linkis.flink.product，默认空值。

## Streamis集群容量评估

【 **组件依赖关系**】

- **DSS** ：固定双节点部署，不参与扩容（负责工作流调度）
- **Linkis** ：固定4节点部署，采用分离式提交模式 （提交任务到 YARN 后释放本地资源），无需扩容
- **Streamis** ：根据流式应用并发数动态扩容，是集群资源评估的核心对象

**【标准机型配置】**

| **组件**     | **机型**     | **节点数** | **CPU** | **内存** | **硬盘** | **说明**                       |
| :----------- | :----------- | :--------- | :------ | :------- | :------- | :----------------------------- |
| **DSS**      | 虚拟机       | 固定2      | 8核     | 16GB     | 500GB    | 工作流调度服务，不直接处理数据 |
| **Linkis**   | 虚拟机       | 固定4      | 32核    | 64GB     | 1024GB   | 任务提交网关，资源随 YARN 释放 |
| **Streamis** | 物理机 WB25X | ≥2         | 90核    | 192GB    | 7TB      | 流式应用核心计算节点，按需扩容 |

**【Streamis扩容公式】**

**TPS 的定义与场景**

- **TPS（Transactions Per Second）** ：表示**每秒处理的日志事件数**，是衡量流处理系统**吞吐量**的核心指标
  - 评估 Streamis 集群处理 **日志回调** 的能力（如实时监控、审计日志上报）。

**流式应用日志 TPS 计算** 

- **单应用 TPS** ：100~750/s（经验值取 **500 TPS/应用** ）。
- **总峰值 TPS** ：600应用×500(TPS/应用)=300,000TPS
- **单机容量** ：每台物理机WB25X支持 **150,000 TPS** （受 Nginx 转发限制）。
- **扩容公式** ：所需机器数=总TPS/单机容量=150,000/300,000=2台

**成本分摊逻辑**

- **单机支撑应用数** = 150,000 TPS / 500 TPS/应用 = **300 应用/机器** 。

  ![image-20250326220032114](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326220032114.png)

  ![image-20250326220106603](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326220106603.png)