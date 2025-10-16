# 场景题

## 尽可能短的时间得到尽可能多的列表

有一个支付的场景，想要尽可能短的时间得到尽可能多的订单列表

**（1）批量查询**

**接口定义** ：提供支持批量查询的订单接口（如`/order/searchList`），允许通过时间范围、订单状态等条件筛选数据，减少单次请求的数据量。

**分页参数** 

- 结合分页（如`page=1&pageSize=100`）和批量查询条件，避免全表扫描。
- 时间范围（如最近一天的订单）。

```
// 示例：批量查询订单接口
public List<Order> searchOrders(Date startTime, Date endTime, String status, int page, int pageSize) {
    // 构造查询条件
    Map<String, Object> params = new HashMap<>();
    params.put("startTime", startTime);
    params.put("endTime", endTime);
    params.put("status", status);
    params.put("offset", (page - 1) * pageSize);
    params.put("limit", pageSize);
    // 调用数据库查询
    return orderMapper.selectByParams(params);
}
```

**（2）分页与并发处理**

如果订单数据量较大，可以通过分页方式逐步获取订单列表，同时结合多线程或异步请求技术并发拉取多个分页数据

- **分页策略** ：通过`count`查询总条数，计算总页数，再通过多线程并发拉取分页数据。
- **线程池优化** ：使用`CompletableFuture`或自定义线程池提升并发效率。

```
// 分页任务实现（使用Callable封装查询逻辑）
public class OrderPageTask implements Callable<List<Order>> {
    private final OrderMapper orderMapper;
    private final String status;
    private final Date startTime;
    private final int page;
    private final int pageSize;

    public OrderPageTask(OrderMapper orderMapper, String status, Date startTime, int page, int pageSize) {
        this.orderMapper = orderMapper;
        this.status = status;
        this.startTime = startTime;
        this.page = page;
        this.pageSize = pageSize;
    }

    @Override
    public List<Order> call() throws Exception {
        int offset = (page - 1) * pageSize;
        return orderMapper.selectByPage(status, startTime, offset, pageSize);
    }
}
// 示例：多线程并发分页查询
public List<Order> fetchAllOrders(int totalRecords, int pageSize) throws ExecutionException, InterruptedException {
    int totalPages = (totalRecords + pageSize - 1) / pageSize;
  
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
                CORE_POOL_SIZE,
                MAX_POOL_SIZE,
                KEEP_ALIVE_TIME,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(QUEUE_CAPACITY),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy());
    
    List<Future<List<Order>>> futures = new ArrayList<>();
    List<Order> allOrders = new ArrayList<>();
    for (int page = 1; page <= totalPages; page++) {
        OrderPageTask task = new OrderPageTask(orderMapper, status, startTime, page, pageSize);
        Future<List<Order>> future = executor.submit(task);
        futures.add(future);
    }
    
    // 合并结果
    for (Future<List<Order>> future : futures) {
        try {
            List<Order> pageOrders = future.get(); // 阻塞等待结果[[6]]
            allOrders.addAll(pageOrders);
        } catch (InterruptedException | ExecutionException e) {
            log.error("分页任务失败", e);
            Thread.currentThread().interrupt(); // 保留中断状态
            // 可选：跳过失败任务或终止流程
        }
    }
    
    executor.shutdown();
    return allOrders;
}
```

**（3）缓存机制优化**

- **高频数据缓存** ：将近期订单列表缓存至Redis，设置合理过期时间（如5分钟），减少数据库压力。
- **缓存更新策略** ：通过定时任务或订单状态变更时主动更新缓存。

```java
// 示例：Redis缓存订单列表
public List<Order> getCachedOrders(Date startTime, Date endTime, String status) {
    String cacheKey = "orders:" + startTime.getTime() + ":" + endTime.getTime() + ":" + status;
    List<Order> cachedOrders = redisTemplate.opsForValue().get(cacheKey);
    if (cachedOrders != null) {
        return cachedOrders;
    }
    // 缓存未命中则查询数据库并写入缓存
    List<Order> orders = searchOrders(startTime, endTime, status, 1, 1000);
    redisTemplate.opsForValue().set(cacheKey, orders, 5, TimeUnit.MINUTES);
    return orders;
}
```

**线程池配置** ：

- 采用`ThreadPoolExecutor`自定义线程池，避免使用`Executors`默认配置（防止资源耗尽）
- 使用`CallerRunsPolicy`拒绝策略，确保高负载时任务不丢失

**任务封装** ：

- 通过`Callable`实现分页查询任务（`OrderPageTask`），符合知识库中提到的异步任务设计模式

**结果合并** ：

- 直接遍历`Future`列表，通过`future.get()`阻塞获取结果（需处理`InterruptedException`和`ExecutionException`）

**性能权衡** ：

- 虽然`future.get()`是阻塞操作，但通过线程池的并发执行，实际数据库查询是并行的，总耗时取决于最慢的分页任务

**（4）分布式任务与消息队列**

- **消息队列解耦** ：订单生成后通过消息队列（如Kafka）异步推送至下游系统，避免直接查询数据库

**【具体过程】**

订单生成后，**消息队列作为中间件** ，将订单信息的**生产者** （订单服务）与**消费者** （下游系统，如库存、支付、物流等）分离

**生产者（订单服务）** ：

- 订单创建成功后，立即将订单数据（如订单ID、用户ID、商品信息）**异步发送到Kafka的指定Topic** （如`order_created_topic`），而非直接调用下游系统的接口

  ```java
  // 示例：订单服务发送消息到Kafka
  kafkaTemplate.send("order_created_topic", orderEvent);
  ```

**消费者（下游系统）** ：

- 库存系统、支付系统等订阅该Topic，**异步消费订单消息** ，各自执行业务逻辑（如扣减库存、发起支付）

**订单与支付系统解耦** ： 用户下单后，订单服务将订单消息写入Kafka，支付系统订阅该消息并异步处理支付逻辑。即使支付系统短暂不可用，订单仍可成功创建，消息不会丢失

## 系统的吞吐量不高，但是查看机器资源发现占用的也不高

​	系统的吞吐量不高但资源利用率低，通常由 **I/O 瓶颈** 、**外部依赖延迟** 、**锁竞争** 、**任务调度不均** 或 **代码低效** 引起。建议通过以下步骤排查：

1. 使用监控工具（如 Prometheus、Grafana）分析资源使用和线程状态
2. 检查数据库查询效率和索引设计
3. 优化代码逻辑和异步处理流程

【**I/O 瓶颈**】

- **现象** ：**CPU 和内存使用率较低，但磁盘 I/O 或网络 I/O 使用率较高**。
- **原因** ：
  - 系统可能存在大量同步 I/O 操作（如文件读写、数据库查询、外部接口调用），导致任务阻塞 
  - 磁盘性能不足或网络延迟较高，限制了数据传输速度。
- **解决方法** 
  - 使用异步 I/O 或非阻塞操作，减少线程等待时间。
  - 引入缓存机制（如 Redis）以减少对磁盘或外部服务的直接访问。
  - 如果是网络瓶颈，可以优化网络配置或增加带宽。

【**代码效率低下**】

- **现象** ：CPU 和内存使用率低，但任务完成速度慢。
- **原因**
  - 业务逻辑中存在冗余计算或低效算法。
  - 频繁的垃圾回收导致系统暂停时间较长。
- **解决方法**
  - 优化算法，减少不必要的计算。
  - 调整 JVM 参数（如堆内存大小、GC 策略），减少垃圾回收对性能的影响

【**外部依赖响应慢**】

- **现象** ：系统在等待外部服务（如数据库、第三方 API）的响应时处于空闲状态。
- **原因** ：
  - 外部服务性能低下，或者网络延迟较高。
  - 单次请求处理时间过长，导致系统无法快速完成任务。
- **解决方法** ：
  - 对外部调用进行批量处理，减少交互次数。
  - 增加超时重试机制，并设置合理的超时时间。
  - 对外部服务进行性能优化，或引入本地缓存以减少对外部服务的依赖

【**数据库或查询效率低下**】

- **现象** ：系统频繁访问数据库，但数据库性能较低。
- **原因** ：
  - 查询语句未优化，导致数据库扫描大量数据。
  - 缺乏索引或索引设计不合理，导致查询效率低下。
- **解决方法** ：
  - 优化 SQL 查询，避免全表扫描。
  - 添加合适的索引，提升查询性能。
  - 使用数据库连接池，减少连接建立和销毁的开销

【**配置不当或资源限制**】

- **现象** ：系统资源未被充分利用，但任务排队严重。
- 原因：
  - 线程池大小或连接池容量不足，限制了并发能力。
  - 系统参数（如 Kafka 的分区数、数据库连接数等）配置不合理。
- 解决方法 
  - 增加线程池大小或连接池容量，提升并发处理能力。
  - 根据实际需求调整系统参数，确保资源能够满足业务需求

【**任务调度不均衡**】

- **现象** ：**部分 CPU 核心负载较高，而其他核心处于空闲状态**。
- **原因** 
  - 任务分配不均，部分线程或进程承担了大部分工作。
  - 线程池配置不合理，导致线程数量不足。
- **解决方法** 
  - 调整线程池大小，确保任务能够均匀分配到所有线程。
  - 使用负载均衡策略，将任务分散到多个节点上处理



## 请求频繁超时，该怎么排查，怎么做

### 排查思路

**排查优先级** ：网络连通性 > 应用日志 > 数据库慢查询 > 第三方服务。

**（1）快速定位超时类型**

**1）区分超时场景**

- **连接超时** ：客户端无法建立TCP连接（如端口未开放、防火墙拦截）。
- **读取超时** ：连接已建立，但服务端未在预期时间内返回数据。

**（2）网络层排查**

**检查网络连通性**

- **Ping、telnet检测** ：

```bash
# 检查端口连通性
telnet <服务IP> <端口>
```

**检查带宽与负载**

- 服务器带宽是否占满？（通过 `iftop`、`nload` 等工具查看）

- `time` 是最常用的测量命令执行时间的工具，直接在命令前添加 `time` 即可。

  ```
  time <命令>
  ```

**（3）监控与日志分析**

**通过日志定位问题：**

- **服务端日志** ：
  - 检查服务端日志（如 Nginx、Tomcat、Spring Boot 日志），查看是否有错误信息或异常堆栈。
- **客户端日志** ：
  - 检查客户端日志，确认请求的详细信息（如 URL、参数、响应时间等）。

**（3）数据库性能**

- 如果请求涉及数据库操作，是否存在全表扫描、锁表、索引缺失？检查**慢查询日志**，确认是否有耗时较长的 SQL 查询。
- 使用工具（如 `EXPLAIN`）分析查询计划，优化索引或查询语句。

```sql
-- 慢查询日志中发现未命中索引的查询
SELECT * FROM orders WHERE user_id=123 AND status='PAID' ORDER BY create_time DESC;

-- 优化：添加联合索引
ALTER TABLE orders ADD INDEX idx_user_status_time (user_id, status, create_time);
```

**（4）应用层（服务端）排查**

**资源使用率监控**

- **CPU** ：是否持续过高？（通过 `top`、`htop` 查看）
- **内存** ：是否有OOM（内存不足）或频繁交换（`free -h`、`vmstat`）
- **磁盘I/O** ：是否存在高负载（`iostat`）
- **网络连接** ：检查 `netstat` 或 `ss` 的连接数，是否存在大量 `TIME_WAIT` 或 `CLOSE_WAIT`

**线程池和队列**

- 检查服务器的线程池配置是否合理。如果线程池耗尽，新请求可能会被阻塞或直接返回超时。

- **同步调用其他第三方API导致线程阻塞**

  ```java
  // 反例：同步调用第三方API导致线程阻塞
  public Response handleRequest(Request req) {
      ThirdPartyService.syncCall(req);  // 耗时2秒
      return new Response();
  }
  
  // 优化：异步处理非核心逻辑
  public Response handleRequest(Request req) {
      Executor executor = Executors.newCachedThreadPool();
      CompletableFuture.runAsync(() -> ThirdPartyService.syncCall(req), executor);
      return new Response();
  }
  ```

**服务配置优化**

- **超时设置** ：检查HTTP服务器（Nginx/Apache）、数据库连接池、RPC客户端的超时参数。
- **并发限制** ：调整Nginx的 `proxy_read_timeout`、Tomcat的 `maxThreads` 等。

### 解决方案

**优化代码逻辑** ：

- 减少不必要的计算或 I/O 操作。
- 异步处理耗时任务，避免阻塞主线程。

**扩容或负载均衡** ：

- 如果服务端资源不足，可通过增加实例或使用负载均衡分散流量。

**数据库性能差**

- 添加索引、分库分表、读写分离、使用缓存（Redis）。

**缓存策略** ：

- 对高频访问的数据引入缓存（如 Redis、Memcached），减少数据库压力。

**超时重试**

### 长效解决方案

**全链路监控** ：

- 使用Prometheus+Grafana监控各服务响应时间。
- 配置**告警规则**（如超时率>5%触发通知）。

**模拟高并发场景**

工具 ：JMeter、Locust

- 配置测试计划，模拟1000并发用户请求。
- 监控系统指标（CPU、内存、响应时间）。

**容量评估**

- 示例：若平均处理时间50ms，100线程可支撑2000 QPS。

```
最大QPS = (1 / 平均单请求处理时间) × 并发线程数
```

## 使用过程中系统发生卡顿，导致服务重启，怎么分析原因

![image-20250424005332571](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250424005332571.png)

【**1. 初步定位卡顿类型**】

**1.1 确认卡顿表现**

- CPU使用率骤升（`top`/`htop` 查看）。
- 内存泄漏（`free -h` 或 `jstat` 观察堆内存）。
- 磁盘I/O过载（`iostat -x 1` 或 `iotop`）。
- 网络延迟或丢包（`ping`/`netstat`）。

**1.2 服务重启原因**

- 健康检查失败（如超时无响应）。
- 内存溢出（OOM Killer强制终止进程）。
- 人工干预或自动重启策略触发

【**2. 日志分析**】

**2.1 应用日志**

关键点 

- 错误堆栈（如`OutOfMemoryError`、`TimeoutException`）。
- 慢操作日志（如数据库查询超时）。
- 线程死锁或阻塞日志（如`DEADLOCK`关键字）。

**2.3 GC日志**

JVM场景 

- 启用GC日志：`-Xloggc:/path/gc.log -XX:+PrintGCDetails`。
- 分析工具：`jstat -gcutil <pid> 1000` 或 [GCViewer ](https://github.com/chewiebug/GCViewer)。

【**3. 资源监控**】

**3.1 CPU**

- `top`：观察进程CPU占用（按`H`查看线程级消耗）。
- `perf`：分析热点函数（`perf top`）。

**3.2 内存**

- `jmap -heap <pid>`：查看堆内存分布（Java）。

**3.3 磁盘与网络**

- `iostat -x 1`：监控磁盘I/O。
- `iftop`/`nethogs`：定位网络流量异常。

【**4. 线程与进程分析**】

**4.1 线程阻塞（Java示例）**

步骤 

1. 导出线程转储：`jstack <pid> > thread_dump.log`。
2. 分析工具：FastThread 或手动检查：
   - 查找 `BLOCKED` 线程。
   - 检查死锁（`jstack` 会直接提示 `Found one Java-level deadlock`）。

![image-20250424005128107](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250424005128107.png)

【**5. 数据库与外部依赖**】

**5.1 数据库**

慢查询 

- MySQL：`SHOW PROCESSLIST;` 或开启慢查询日志（`long_query_time=1`）。

锁竞争 

- 检查表锁或行锁（如`SHOW ENGINE INNODB STATUS`）





## 如何读取百万数据效率提高

1.使用多线程：建立适当的线程数，对大数据量进行读取。在读取的时候，可以不加锁，在写入的时候，在数据库进行加行级锁；2.建立索引。

## 如何防止url重复提交

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps1-4133580.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps2.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps3-4133580.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps4-4133580.jpg) 

## 如何实现主线程等待子线程完成后执行

**使用 `Thread.join()`**

**原理** ：主线程调用子线程的 `join()` 方法，阻塞自身直到子线程执行完毕

```java
public class JoinExample {
    public static void main(String[] args) {
        Thread childThread = new Thread(() -> {
            System.out.println("子线程开始执行");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            System.out.println("子线程执行完毕");
        });
        childThread.start();
        try {
            childThread.join();  // 主线程等待子线程结束
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程继续执行");
    }
}
```

**使用 `CountDownLatch`**

**原理** ：通过计数器实现线程同步，主线程等待计数器归零

```
import java.util.concurrent.CountDownLatch;

public class CountDownLatchExample {
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(1);  // 初始化计数器为1

        Thread childThread = new Thread(() -> {
            System.out.println("子线程开始执行");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            System.out.println("子线程执行完毕");
            latch.countDown();  // 计数器减1
        });

        childThread.start();
        try {
            latch.await();  // 主线程阻塞，直到计数器为0
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程继续执行");
    }
}
```

**使用 `ExecutorService` 和 `Future`**（需要返回值）

**原理** ：通过线程池提交任务，调用 `Future.get()` 阻塞等待结果。

```
import java.util.concurrent.*;

public class FutureExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<?> future = executor.submit(() -> {
            System.out.println("子线程开始执行");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            System.out.println("子线程执行完毕");
        });
        try {
            future.get();  // 主线程阻塞，直到任务完成
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
        System.out.println("主线程继续执行");
    }
}
```

## 有三个线程a,b,c，一个用synchronized修饰的set方法，和一个普通的get方法，a,b去调用set方法，c调用get方法，会产生什么问题

在这个场景中，`synchronized`修饰的`set`方法和普通的`get`方法之间的交互可能会导致**线程安全问题** 

### 问题

【**1. 场景描述**】

- **线程A、B** ：调用`synchronized`修饰的`set`方法。
- **线程C** ：调用普通的`get`方法（未加锁）。
- **核心问题** ：`set`方法是同步的，而`get`方法是非同步的，这会导致潜在的**数据不一致** 或**可见性问题** 。

【**2. 关键概念**】

**`synchronized`的作用**

- `synchronized`修饰的方法会确保同一时刻只有一个线程可以执行该方法。
- 它提供了**互斥性** （保证只有一个线程修改共享资源）和**可见性** （修改后的值对其他线程可见）。

**普通方法的问题**

- 普通方法没有加锁，因此无法保证与`synchronized`方法之间的**可见性** 。
- 如果一个线程通过`set`方法修改了共享变量，另一个线程通过`get`方法读取时，可能看不到最新的修改结果（由于线程缓存或指令重排序）。

【**3. 可能产生的问题**】

**数据不一致**

- 假设共享变量初始值为0
  1. 线程A调用`set`方法，将值更新为`10`。
  2. 线程C调用`get`方法，但可能仍然读取到旧值`0`（因为`get`方法没有同步，无法感知到线程A的修改）。

**可见性问题**

- 在多核CPU环境下，每个线程可能有自己的缓存副本。如果没有使用同步机制，`set`方法的修改可能只存在于线程A的缓存中，而线程C直接从主内存中读取旧值。

**非原子性操作**

- 如果`set`方法包含多个操作（如先检查再修改），而`get`方法在中间状态读取数据，可能导致读取到部分更新的结果。

```
public class SharedResource {
    private int value;

    // synchronized修饰的set方法
    public synchronized void set(int newValue) {
        this.value = newValue;
        System.out.println("Set value to: " + newValue);
    }

    // 普通的get方法
    public int get() {
        System.out.println("Get value: " + value);
        return value;
    }
}
```

### 解决方案

【**将`get`方法也加锁**】

- 使`get`方法与`set`方法使用相同的锁，确保可见性和一致性。

```
public synchronized int get() {
    System.out.println("Get value: " + value);
    return value;
}
```

- **优点** ：简单易行，确保线程安全。
- **缺点** ：性能开销较大，所有线程都需要竞争锁。

【**使用`volatile`关键字**】

- 为共享变量添加`volatile`修饰符，确保其修改对其他线程立即可见。

```
private volatile int value;
```

- **优点** ：性能优于加锁，适用于简单的读写场景。
- **缺点** ：只能保证可见性，不能保证复合操作的原子性（如先检查再修改）。



**使用 `volatile` 的作用**

如果共享变量被声明为`volatile`：

- 它可以保证变量的**可见性** ：当一个线程（如线程A或B）通过`set`方法修改了共享变量后，线程C通过`get`方法读取时，能够看到最新的值。
- 它可以禁止指令重排序：防止编译器或处理器对代码进行优化而导致数据不一致。

**3.1 如果 `set` 方法是简单的单一写操作**

假设`set`方法只是简单地赋值：

```java
public synchronized void set(int newValue) {
    this.value = newValue;
}
```

在这种情况下，如果共享变量被声明为`volatile`，可以去掉`synchronized`：

```
private volatile int value;

public void set(int newValue) {
    this.value = newValue; // 单一写操作，volatile 足够保证可见性
}

public int get() {
    return value; // 单一读操作，volatile 足够保证可见性
}
```

**3.2 如果 `set` 方法包含复合操作**

如果`set`方法包含复合操作（如先检查再更新）：

```java
public synchronized void set(int newValue) {
    if (this.value < newValue) { // 检查条件
        this.value = newValue;  // 更新值
    }
}
```

此时，即使共享变量被声明为`volatile`，仍然需要保留`synchronized`：

- 原因：复合操作（如`if (value < newValue)`）不是原子操作，`volatile`无法保证整个操作的原子性。
- 解决方案：保留`synchronized`以确保复合操作的线程安全。

```java
public class SharedResource {
    private volatile int value;

    public synchronized void set(int newValue) {
        if (this.value < newValue) { // 先检查
            this.value = newValue;  // 再更新
        }
    }

    public int get() {
        return value; // 单一读操作，volatile 足够
    }
}
```

【**使用`AtomicInteger`**】

- 如果共享变量是简单的整数类型，可以使用`AtomicInteger`等原子类。

```java
import java.util.concurrent.atomic.AtomicInteger;

public class SharedResource {
    private AtomicInteger value = new AtomicInteger(0);

    public void set(int newValue) {
        value.set(newValue);
        System.out.println("Set value to: " + newValue);
    }

    public int get() {
        int currentValue = value.get();
        System.out.println("Get value: " + currentValue);
        return currentValue;
    }
}
```

- **优点** ：性能高，无需显式加锁。
- **缺点** ：仅适用于基本类型或简单的复合操作。

## 两个线程并发写(同时尝试修改共享数据)会出现什么问题

​	并发写的核心问题是 **共享数据的同步** 。如果不加保护，轻则数据错误，重则程序崩溃。解决方案的核心思想是 **通过同步机制保证原子性和可见性** ，从而消除竞态条件。

### 并发写会出现的问题

【 **竞态条件（Race Condition）**】

- **现象** ：线程的执行顺序不可控，导致程序结果依赖于线程调度的“运气”。
- **例子：** 
  - 假设共享变量 `count = 0`，线程A和线程B同时执行 `count++`。
  - 理想结果应为 `count = 2`，但实际可能得到 `1`。
- **原因 ：**
  - count++ 并非原子操作，它分为三步：
    1. 读取 `count` 的值（例如读到 `0`）。
    2. 将值加1（得到 `1`）。
    3. 将结果写回 `count`。
  - 如果两个线程同时读取旧值（都读到 `0`），计算后都会写回 `1`，导致结果错误。

【**原子性破坏**】

- **现象** ：看似简单的操作被拆分成多步，中间状态可能被其他线程干扰。
- **例子** 
  - 假设一个线程写入 `flag = true` 和 `data = "hello"`，另一个线程读取这两个变量。
  - 可能读到 `flag = true` 但 `data` 仍是旧值（如 `null`），因为写操作被重排序或未及时同步。

【**可见性问题**】

- **现象** ：线程的本地缓存未及时刷新，导致其他线程看不到最新值。
- **例子** 
  - 线程A修改了共享变量 `running = false`，但线程B仍在循环中读取旧值 `true`。
- **原因** 
  - 线程可能将变量缓存在寄存器或本地内存中，未立即写回主存。

【**内存一致性错误**】

- **现象** ：不同线程对共享数据的状态认知不一致。
- **例子** 
  - 线程A更新了对象的字段，但线程B看到的对象状态可能部分更新（如构造函数未完成）。

### 如何解决这些问题

**加锁（Synchronized/Lock）** ：

- 用 `synchronized` 或 `ReentrantLock` 保证同一时刻只有一个线程访问共享数据。

  ```java
  synchronized void increment() {
      count++;
  }
  ```

**原子类（Atomic Classes）** ：

- 使用 `AtomicInteger` 等类实现原子操作。

  ```java
  AtomicInteger count = new AtomicInteger(0);
  count.incrementAndGet(); // 原子自增
  ```

**volatile 关键字** ：

- 保证变量的可见性（写操作立即刷新到主存），但无法解决复合操作（如 `i++`）。

**线程安全容器** ：

- 使用 `ConcurrentHashMap`、`CopyOnWriteArrayList` 等并发包提供的工具类。

**避免共享状态** ：

- 尽量使用局部变量或不可变对象（Immutable），减少共享数据。



## 如果有个java进程cpu占有率特别高，如何排查问题

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps5-4133580.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps6-4133580.jpg) 

## SQL查询慢如何解决

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps7-4133580.jpg) 

## 大数据量下慢查询优化

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/wps8-4133580.jpg) 

## 设计一个长连接转短链接的系统(短网址)

**短地址的使用场景**：新浪微博（发的博文有字数限制，长连接上去都编辑不了几个字了）、短网址二维码

**短网址的好处**

（1）节省网址长度，便于社交化传播，一个是让URL更短小，传播更方便，尤其是URL中有中文和特殊字符，短网址解决很长的URL难以记忆不利于传播的问题；

（2）短网址在我们项目里可以很好的对开放以及对URL进行管理。有一部分网址可能会涵盖性、暴力、广告等信息，这样我们可以通过用户的举报，完全管理这个连接将不出现在我们的应用中，对同样的URL通过加密算法之后，得到的地址是一样的；

（3）方便后台跟踪点击量、地域分布等用户统计。我们可以对一系列的网址进行流量，点击等统计，挖掘出大多数用户的关注点，这样有利于我们对项目的后续工作更好的作出决策；

（4）规避关键词、域名屏蔽手段、隐藏真实地址，适合做付费推广链接；

（5）当你看到一个淘宝的宝贝连接后面是200个“e7x8bv7c8bisdj”这样的字符的时候，你还会觉得舒服吗。更何况微博字数只有140字，微博或短信里，字数不够，你用条短网址就能帮你腾出很多空间来；



**【如何生成短地址】**

**通过发号策略，给每一个过来的长地址，发一个号即可，小型系统直接用mysql的自增索引就搞定了。**

如果是大型应用，可以考虑各种分布式key-value系统做发号器。不停的自增就行了。第一个使用这个服务的人得到的短地址是http://xx.xx/0第二个是http://xx.xx/1第11个是http://xx.xx/a第依次往后，相当于实现了一个62进制的自增字段即可。

**1）如何保证同一个长地址每次转出来都是一样的短地址**

- 上面的发号原理中，是不判断长地址是否已经转过的。也就是说用一个地址每次转出来的短地址是不一样的。而一长对多短，属实有些浪费空间。
- **给出方案：用key-value存储，保存“最近”生成的长对短的一个对应关系。也就是说，我并不保存全量的长对短的关系，而只保存最近的。比如采用一小时过期的机制来实现LRU淘汰**。
- **长转短的流程：**
  - 在这个“最近”表中查看一下，看长地址有没有对应的短地址
  - 有就直接返回，并且将这个key-value对的过期时间再延长成一小时
  - 如果没有，就通过发号器生成一个短地址，并且存存放在这个“最近”表中，过期时间设为1小时
- 所以当一个地址被频繁使用，那么它会一直在这个key-value表中，总能返回当初生成那个短地址，不会出现重复的问题。如果它使用并不频繁，那么长对短的key会过期，LRU机制自动就会淘汰掉它。

**2）如何保证发号器的大并发、高可用**

​	从刚才的设计看起来存在一个单点，那就是发号器。如果做成分布式的，那么多节点要保持同步加1，多点同时写入，以CAP理论看，是不可能真正做到的。

- 我们可以实现两个发号器，一个发单号，一个发双号，这样就变单点为多点了
- 依次类推，我们可以实现1000个逻辑发号器，分别发尾号为0到999的号。每发一个号，每个发号器加1000，而不是加1。这些发号器独立工作，互不干扰即可。

​	而且在实现上，也可以先是逻辑的，真的压力变大了，再拆分成独立的物理机器单元。1000个节点，估计对人类来说应该够用了。如果你真的还想更多，理论上也是可以的。

**3）跳转用301还是302**

**301和302的区别：**

- 301：永久重定向，浏览器会缓存DNS解析记录（下次再点击时，直接从本地缓存拿到IP）
- 302：临时重定向，浏览器不会缓存当前域名的解析记录。

短地址一经生成就不会变化，所以用301是符合http语义的。同时对服务器压力也会有一定减少。

但是如果使用了301，我们就无法统计到短地址被点击的次数了（直接从本地缓存获取IP）。

而这个点击次数是一个非常有意思的大数据分析数据源。能够分析出的东西非常非常多。所以选择302虽然会增加服务器压力，但是我想是一个更好的选择

## 数据库调优的维度和步骤

**1.优化表设计**

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/wps27-4133580.jpg) 

**2.优化逻辑查询(sql优化)**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps28-4133580.jpg)

**3.优化物理查询**

​	物理查询优化是在确定了逻辑查询优化之后，采用**物理优化技术（比如索引等）**，通过计算代价模型对各种可能的访问路径进行估算，从而找到执行方式中代价最小的作为执行计划。在这个部分中，我们需要掌握的重点是对索引的创建和使用。

![image-20250319140846737](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319140846737-4133580.png)

**4.使用Redis或Memcached作为缓存**

![image-20250319141009256](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141009256-4133580.png)

![image-20250319141016229](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250319141016229-4133580.png)

**5.库级优化**

**【读写分离】**

![image-20250319141125708](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250319141125708-4133580.png)

![image-20250319141150939](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141150939-4133580.png)



## mysql服务器cpu飙升，如何排查问题并解决

【**确认问题根源**】

**使用`top`命令定位进程**

- 执行`top`命令观察CPU占用情况，确认是否为`mysqld`进程导致的CPU飙升
  - 若`mysqld`占用过高，继续排查MySQL；
  - 若其他进程（如Java应用）异常，则需针对性处理。

【**分析当前MySQL状态**】

**查看活跃查询**

- 登录MySQL执行`SHOW PROCESSLIST;`，观察是否有长时间运行的查询（如`State`为`Sending data`或`Copying to tmp table`）

**检查连接数**

- 执行`SHOW STATUS LIKE 'Threads_connected';`，确认是否因连接数过多（如未释放空闲连接）导致资源耗尽

【**优化慢查询**】

**开启并分析慢查询日志**

- 配置慢查询日志（`slow_query_log=1`，`long_query_time=1`）

- 使用工具（如`mysqldumpslow`或`pt-query-digest`）分析日志，定位高频或耗时SQL

  **工作常用参考：**

  ```bash
  #得到返回记录集最多的10个SQL
  mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log
  
  #得到访问次数最多的10个SQL
  mysqldumpslow -s c -t 10 /var/lib/mysql/atguigu-slow.log
  
  #得到按照时间排序的前10条里面含有左连接的查询语句
  mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/atguigu-slow.log
  
  #另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现爆屏情况
  mysqldumpslow -s r -t 10 /var/lib/mysql/atguigu-slow.log | more
  ```

- explain具体慢查询语句

  ![image-20220628212029574](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20220628212029574-5386793.png)

  ![image-20250319001708346](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319001708346-5386793.png)

  **type**

  ​	执行计划的一条记录就代表着MySQL对某个表的 **执行查询时的访问方法,** 又称“访问类型”，其中的 `type` 列就表明了这个访问方法是啥，是较为重要的一个指标。比如，看到`type`列的值是`ref`，表明`MySQL`即将使用`ref`访问方法来执行对`s1`表的查询。

  ​	完整的访问方法如下： **`system ， const ， eq_ref ， ref ， fulltext ， ref_or_null ， index_merge ， unique_subquery ， index_subquery ， range ， index ， ALL`** 。

**优化SQL语句**

- **添加索引** ：对WHERE、JOIN、ORDER BY涉及的字段建立索引，避免全表扫描
- **减少全表扫描** ：优化查询逻辑（如避免SELECT *，拆分大查询）

【**检查锁与事务**】

**查看锁竞争**

- 执行`SHOW ENGINE INNODB STATUS;`，检查是否存在行锁或表锁等待
  - 优化事务逻辑（如减少事务粒度，避免长事务）。
  - 对高频更新的表拆分或使用乐观锁

【**硬件与架构优化**】

**升级硬件资源**

- 增加CPU核心数或使用更高性能的磁盘（如SSD）

**分库分表或读写分离**

- 对大数据量表进行水平分表，或通过主从复制分流读请求

## 索引失效怎么解决

**避免模糊查询前导通配符**

- **问题** ：`LIKE '%abc'`会导致全表扫描，索引失效
- **解决** ：
  - 改用`LIKE 'abc%'`（使用前缀匹配）；
  - 或使用全文索引（`FULLTEXT`）替代模糊查询

**确保数据类型匹配**

- **问题** ：字段为`VARCHAR`，但查询值未加引号（如`WHERE id=1002`），导致隐式类型转换
- **解决** ：
  - 改写为范围查询（如`WHERE create_time BETWEEN '2023-01-01' AND '2023-12-31'`）；
  - 或使用生成列（Generated Column）并为其创建索引

**禁止对索引列使用函数或表达式**

- **问题** ：`WHERE YEAR(create_time)=2023`导致索引失效
- **解决** ：
  - 改写为范围查询（如`WHERE create_time BETWEEN '2023-01-01' AND '2023-12-31'`）；
  - 或使用生成列（Generated Column）并为其创建索引

**遵循最左前缀法则**

- **问题** ：复合索引`(a,b,c)`未使用最左列（如`WHERE b=1`）
- **解决** ：
  - 调整查询条件顺序（如`WHERE a=1 AND b=2`）;
  - 或创建符合查询习惯的新索

**优化OR条件**

- **问题** ：`WHERE a=1 OR b=2`可能导致索引失效
- **解决** ：
  - 为`a`和`b`创建复合索引；
  - 拆分为`UNION`查询（如`SELECT * FROM t WHERE a=1 UNION SELECT * FROM t WHERE b=2`）

**使用EXPLAIN分析** ：

- 通过`EXPLAIN`查看执行计划，确认是否命中索引

## 在一个大数据量的场景下对一个很长的url创建索引

【**1. URL 索引的核心挑战**】

- **长度限制** ：URL 长度可能超出数据库索引的最大长度（如 MySQL 的 `VARCHAR` 最大索引长度为 767 字节）。
- **存储效率** ：直接存储原始 URL 浪费空间，尤其是在重复 URL 较多的情况下。
- **查询性能** ：长字符串的索引会增加磁盘 I/O 和内存消耗。

【**2. 解决方案：哈希化与分片策略**】

**2.1 哈希化（推荐）**

通过将 URL 转换为固定长度的哈希值（如 MD5 或 SHA-256），可以显著减少存储开销并提高索引效率。

**实现步骤**

1. **计算哈希值** ：

   - 使用一致性哈希算法（如 MD5、SHA-256）生成固定长度的哈希值。
   - 哈希值长度固定（如 MD5 为 32 字节），适合存储和索引。

2. **存储结构** ：

   - 数据库表设计：

   - 使用 `url_hash` 字段存储哈希值，并为其创建唯一索引。

   - 使用 `original_url` 字段存储原始 URL。

     ```
     CREATE TABLE url_index (
         id BIGINT AUTO_INCREMENT PRIMARY KEY,
         url_hash CHAR(32) NOT NULL,       -- MD5 哈希值
         original_url TEXT NOT NULL,      -- 原始 URL
         created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
         UNIQUE INDEX idx_url_hash (url_hash) -- 对哈希值创建唯一索引
     );
     ```

3. **查询逻辑** ：

   - 插入前检查哈希值是否存在：

   - ```
     SELECT id FROM url_index WHERE url_hash = ?;
     ```

   - 如果不存在，则插入新记录。

4. **优点** ：

   - 减少存储开销（哈希值比原始 URL 更短）。
   - 提高查询性能（固定长度字段更适合索引）。
   - 支持去重（通过唯一索引）。

**2.2 分片策略**

如果单表无法承载海量数据，可通过分片技术将数据分散到多个表或数据库中。

**实现步骤**

1. **按哈希值分片** ：
   - 根据哈希值取模分配到不同表中：shard_id = hash_url(url) % NUM_SHARDS;
   - 每个分片独立存储和索引。
2. **分布式存储** ：
   - 使用分布式数据库（如 HBase、Cassandra）或搜索引擎（如 Elasticsearch）存储和索引 URL。

**【3.前缀索引】**

**什么是前缀索引？**

前缀索引是一种对字符串字段的部分内容（即前 N 个字符）创建索引的技术。它通过仅索引字段的前缀部分，而不是整个字段的内容，来减少索引的存储开销和提高查询性能。

- 可使用前缀索引（仅索引 URL 的前 N 个字符）。语句表示对 `url` 字段的前 100 个字符创建索引，而不是对整个 URL 创建索引。

  ```sql
  CREATE INDEX idx_url_prefix ON url_table (url(100)); -- 索引前100字符
  ```

  由于前缀索引可能导致索引不唯一，可以通过以下方法解决：

  **结合其他字段创建复合索引** ：

  - 将前缀索引与其他字段（如 `id` 或时间戳）组合，确保唯一性。

    ```sql
    CREATE UNIQUE INDEX idx_url_prefix_id ON url_table (LEFT(url, 100), id);
    ```

  **使用哈希化替代前缀索引** ：

  - 对 URL 计算固定长度的哈希值（如 MD5 或 SHA-256），并基于哈希值创建索引。

    ```sql
    ALTER TABLE url_table ADD COLUMN url_hash CHAR(32) AS (MD5(url));
    CREATE UNIQUE INDEX idx_url_hash ON url_table (url_hash);
    ```

    

**为什么使用前缀索引？**

在处理长字符串字段（如 URL、文章内容等）时，直接对整个字段创建索引会导致以下问题：

- **存储开销大** ：索引需要额外的存储空间，而长字符串会显著增加索引大小。
- **性能问题** ：索引越长，磁盘 I/O 和内存消耗越高，查询效率降低。
- **索引限制** ：某些数据库（如 MySQL 的 InnoDB 引擎）对单列索引的长度有限制（如最大 767 字节或 3072 字节，取决于配置）。

前缀索引通过仅索引字段的前 N 个字符，可以在一定程度上缓解这些问题。

**前缀索引的工作原理**

![image-20250424011630042](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250424011630042.png)

**前缀索引的缺点**

1. **可能导致索引不唯一** ：
   - 如果前缀部分不足以区分不同的记录，可能会导致索引冲突。
2. **无法支持全字段匹配** ：
   - 前缀索引只能加速前缀匹配的查询（如 `LIKE 'prefix%'`），但无法加速全字段匹配（如 `=` 或 `LIKE '%substring%'`）。
3. **需要选择合适的前缀长度** ：
   - 前缀长度过短可能导致索引区分度不足（即多个记录共享相同的前缀）。
   - 前缀长度过长则无法显著节省存储空间。

**解决索引不唯一的问题**

由于前缀索引可能导致索引不唯一，可以通过以下方法解决：

1. **结合其他字段创建复合索引** ：

   - 将前缀索引与其他字段（如 `id` 或时间戳）组合，确保唯一性。

     ```sql
     CREATE UNIQUE INDEX idx_url_prefix_id ON url_table (LEFT(url, 100), id);
     ```

2. **使用哈希化替代前缀索引** ：

   - 对 URL 计算固定长度的哈希值（如 MD5 或 SHA-256），并基于哈希值创建索引。

     ```sql
     ALTER TABLE url_table ADD COLUMN url_hash CHAR(32) AS (MD5(url));
     CREATE UNIQUE INDEX idx_url_hash ON url_table (url_hash);
     ```

     



## 怎么保证一个库存不会同时出库入库？超扣？

在分布式系统中，库存的并发操作（如同时出库和入库）可能导致**超扣** （即库存数量变为负数）或**数据不一致** 。以下是保证库存操作线程安全的解决方案，结合数据库事务、锁机制和分布式设计：

【**总结**】

防止库存超扣的核心是**控制并发操作的原子性** ，可通过以下组合策略实现：

- **单机事务** ：数据库锁 + 乐观锁。
- **分布式场景** ：Redis 缓存预扣 + 分布式锁。
- **兜底机制** ：数据库约束 + 对账任务

【**1. 数据库事务与锁**】

**悲观锁（Pessimistic Locking）**

- **原理** ：在查询库存时显式加锁，确保其他事务无法修改。

- **实现** 

  ```sql
  BEGIN TRANSACTION;
  SELECT stock FROM products WHERE id = 1 FOR UPDATE;
  -- 执行库存更新（如扣减或增加）
  UPDATE products SET stock = stock - 1 WHERE id = 1;
  COMMIT;
  ```

- **适用场景** ：**高并发写操作（如秒杀场景）。**

- **缺点** ：可能引发锁竞争，降低吞吐量。

【**2. 分布式锁**】

**当库存操作跨多个服务或实例时，需使用分布式锁保证全局一致性。**

**1）Redis 分布式锁**

```java
// 使用 Redis 的 SETNX 命令获取锁
public boolean lock(String lockKey, String requestId, long expireTime) {
    return redisTemplate.opsForValue().setIfAbsent(lockKey, requestId, expireTime, TimeUnit.MILLISECONDS);
}

// 释放锁（需校验锁持有者）
public void unlock(String lockKey, String requestId) {
    if (requestId.equals(redisTemplate.opsForValue().get(lockKey))) {
        redisTemplate.delete(lockKey);
    }
}
```

**2）ZooKeeper 分布式锁**

- **原理** ：通过临时有序节点实现锁竞争。
- **优点** ：天然支持锁的自动释放（Session 超时）

【**3. 预扣库存与最终一致性**】

**消息队列串行化**

- 将所有库存操作请求发送到消息队列（如 Kafka）。
- 消费者单线程处理请求，保证顺序性。

**适用场景** ：高并发但允许短暂延迟的场景。



## 海量数据的排序

### 估算内存占用量

**数据类型与大小**：

- 32位整数：4字节/条
- 64位整数或双精度浮点数：8字节/条
- 字符串或结构体：取决于长度，例如100字节/条

200万数量级：假设每个元素为整数（4字节），总内存为： `2,000,000 × 4B = 8,000,000B ≈ 7.63MB`

**对象内存占用估算**

```java
class Person {
    String name;  // 假设平均长度10字符（Unicode，2B/字符 → 20B）
    int age;      // 4B
}
```

- **对象头** ：Java中每个对象头占12B（32位JVM）或16B（64位JVM）。
- **字段对齐** ：JVM按8字节对齐，总大小可能为： `对象头(16B) + name引用(4B) + age(4B) + 填充(4B) ≈ 28B` `String`对象本身额外占用约40B（字符数组+其他字段）。
- **总内存** ： `200万 × (28B + 40B) ≈ 136,000,000B ≈ 130MB` 现代JVM可轻松处理（堆内存建议至少512MB）。

### 选择排序算法

**数据完全在内存中**

- **快速排序** （推荐）：
  - **优点** ：平均时间复杂度`O(n log n)`，常数因子小，原地排序（空间复杂度`O(log n)`）。
  - **缺点** ：最坏情况`O(n²)`（可通过随机化pivot避免）。
  - **适用** ：大多数编程语言默认排序算法（如Java的`Arrays.sort()`）。

**`Comparator.comparingInt`**

- `comparingInt` 是 `Comparator` 接口提供的一个静态方法，用于根据某个整数属性生成一个比较器。
- 它接受一个函数式接口（如 Lambda 表达式或方法引用）作为参数，该接口的作用是从对象中提取出一个整数值（这里是 `p.age`）。

![image-20250410004224899](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250410004224899.png)

![image-20250410004538016](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 952 708"></svg>)

**内存受限时的外部排序**

​	当数据量过大（例如10亿个整数，约40GB）无法一次性加载到内存时，需要使用外部排序。外部排序分为两个阶段：**分块排序和归并**。我们假设内存限制为4GB，分块处理数据

**分块排序**：

- 数据从输入文件`large_data.txt`读取，每块1亿个整数（约400MB）。
- 使用`Arrays.sort()`对每块排序后，存为临时文件（如`block_0.txt`）。

**归并阶段**：

- 使用`PriorityQueue`（最小堆）实现k路归并，从所有临时文件中读取数据。
- 每次取出最小值写入输出文件`sorted_data.txt`，并从对应块读取下一行。

**适用场景**：数据量远超内存时（例如10亿条数据占40GB，内存只有4GB）。

**输入文件准备**：需要预先生成`large_data.txt`，每行一个整数（示例中未提供生成代码，可参考`InMemorySort`中的随机数生成逻辑）。

![image-20250410004933662](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250410004933662.png?token=ARFL34DECLEFLAWNOYNBTLTI563LU)

![image-20250410005028940](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250410005028940.png?token=ARFL34A6UFFBAWKYF6B5L3DI563LU)

![image-20250410005125989](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250410005125989.png)

## Topk

### 解法

**1）排序**

​	最容易想到的肯定是排序算法，然后取其排序的最大/小的k个数就完事了。其时间复杂度是O(nlogn)，但是问题来了，如果前提是以亿为单位的数据，你还敢用排序算法吗？明明只需要k个数，为啥要对所有数都排序呢？并且对这种海量数据，计算机内存不一定能扛得住

**2）局部淘汰法**

​	既然只需要k个数，那么我们可以再优化一下，先用一个容器装这个数组的前k个数，然后找到这个容器中最小的那个数，再依次遍历后面的数，如果后面的数比这个最小的数要大，那么两者交换。一直到剩余的所有数都比这个容器中的数要小，那么这个容器中的数就是最大的k个数。

这种算法的时间复杂度为O(n*m)，其中m为容器的长度。

**3）如果有很多重复，可以使用Hash去重**

​	如果这1亿个书里面有很多重复的数，先通过Hash法，把这1亿个数字去重复，这样如果重复率很高的话，会减少很大的内存用量，从而缩小运算空间，然后通过分治法或最小堆法查找最大的10000个数

**4）小根堆**

我们可以先用前k个元素生成一个小顶堆，这个小顶堆用于存储当前k个元素，例子同上，可以构造小顶堆如下：

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps9-4554369.jpg?token=ARFL34FKR3S3KCZXY7KXPBTI563LW)

1. 初始化一个小顶堆，其堆顶元素最小。
2. 先将数组的前 k 个元素依次入堆。
3. 从第 k+1 个元素开始，若当前元素大于堆顶元素，则将堆顶元素出堆，并将当前元素入堆。
4. 遍历完成后，堆中保存的就是最大的 k 个元素。

​	整个过程直至10亿个数全部遍历完为止。这个方法使用的内存是可控的，只有10个数字所需的内存即可。这种方法Java中有现成的数据结构优先级队列可以使用:java.util.PriorityQueue

这种堆解法的时间复杂度为O(N*logk)，并且堆解法也是求解TopK问题的经典解法，用代码实现如下：

```java
/* 基于堆查找数组中最大的 k 个元素 */
Queue<Integer> topKHeap(int[] nums, int k) {
    // 初始化小顶堆
    Queue<Integer> heap = new PriorityQueue<Integer>();
    // 将数组的前 k 个元素入堆
    for (int i = 0; i < k; i++) {
        heap.offer(nums[i]);
    }
    // 从第 k+1 个元素开始，保持堆的长度为 k
    for (int i = k; i < nums.length; i++) {
        // 若当前元素大于堆顶元素，则将堆顶元素出堆、当前元素入堆
        if (nums[i] > heap.peek()) {
            heap.poll();
            heap.offer(nums[i]);
        }
    }
    return heap;
}
```

### TopK的海量数据

海量数据前提下，肯定不可能放在单机上。

- 拆分，可以按照哈希取模或者其它方法拆分到多台机器上，并在每个机器上维护最小堆
- 整合，将每台机器上得到的最小堆合并成最终的最小堆

（1）单机+单核+足够大内存

如果需要查找10亿个查询次（每个占8B）中出现频率最高的10个，考虑到每个查询词占8B，则10亿个查询次所需的内存大约是10^9 * 8B=8GB内存。如果有这么大内存，直接在内存中对查询次进行排序，顺序遍历找出10个出现频率最大的即可。这种方法简单快速，使用。然后，也可以先用HashMap求出每个词出现的频率，然后求出频率最大的10个词。

（2）单机+多核+足够大内存

这时可以直接在内存总使用Hash方法将数据划分成n个partition，每个partition交给一个线程处理，线程的处理逻辑同（1）类似，最后一个线程将结果归并。

该方法存在一个瓶颈会明显影响效率，即数据倾斜。每个线程的处理速度可能不同，快的线程需要等待慢的线程，最终的处理速度取决于慢的线程。而针对此问题，解决的方法是，将数据划分成c×n个partition（c>1），每个线程处理完当前partition后主动取下一个partition继续处理，知道所有数据处理完毕，最后由一个线程进行归并。

（3）单机+单核+受限内存

这种情况下，需要将原数据文件切割成一个一个小文件，如用hash(x)%M，将原文件中的数据切割成M小文件，如果小文件仍大于内存大小，继续采用Hash的方法对数据文件进行分割，知道每个小文件小于内存大小，这样每个文件可放到内存中处理。采用（1）的方法依次处理每个小文件。

（4）多机+受限内存

​	这种情况，为了合理利用多台机器的资源，可将数据分发到多台机器上，每台机器采用（3）中的策略解决本地的数据。可采用hash+socket方法进行数据分发。

### 两个各有50亿URL 的文件，找出共同的URL

​	在处理两个各含50亿URL的文件（假设每个URL占64字节，内存限制4GB）时，找出共同URL的核心挑战在于数据量远超内存容量。

【**1. 进行hash分片**】

**步骤** ：

1. **选择哈希函数** ：使用均匀分布的哈希算法（如MD5或CRC32）。
2. **分片操作** 
   - 对文件A和文件B分别遍历所有URL，计算哈希值后取模（如`hash(url) % 1000`）。
   - 将URL按哈希值分配到1000个小文件中（如`A_0, A_1, ..., A_999`和`B_0, B_1, ..., B_999`）。
3. **原理** ：相同URL必落在同一个小文件对中（如`A_i`和`B_i`），仅需逐对处理。

**优势** ：

- 每个小文件约320MB（50亿URL / 1000 = 500万URL，每个64字节 ≈ 320MB），可载入内存。
- 避免全量比对，减少计算量。

【**2. 处理小文件对**】

**步骤** 

1. 对每对小文件A_i和B_i：
   - 将`A_i`的URL存入内存哈希集合hashset。
   - 遍历`B_i`的URL，检查是否存在于集合中，存在则为共同URL。
2. 优化 
   - 若小文件仍较大，可递归分片（如再分100片）。
   - 使用位图（Bitmap）或布隆过滤器（Bloom Filter）进一步节省内存。



**关键优化技术**

**1. 磁盘I/O优化**

- **缓冲读取** ：逐行读取大文件，避免一次性加载。
- **并行处理** ：多线程同时处理多个小文件对，提升吞吐量。

**2. 数据结构优化**

- **哈希集合** ：使用`HashSet`存储URL，时间复杂度O(1)。
- **位图** ：若URL可映射为整数，用位图标记存在性（节省内存）。

**3. 容错与验证**

- **校验分片** ：确保分片后文件无数据丢失。
- **结果去重** ：合并所有小文件的结果时，去重输出。



### 出现频率最高的100个词

​	题目描述：假如有一个 1G 大小的文件，文件里每一行是一个词，每个词的大小不超过16 bytes，要求返回出现频率最高的 100个词。内存限制是 10M。

#### 解法1：分治法

​	由于内存限制（10MB），无法一次性将整个1GB文件加载到内存中进行处理。因此，我们采用分治的策略，将大文件分解成多个小文件，确保每个小文件的大小不超过10MB，从而在内存中进行高效处理。

**【思路如下】**

- 采用分治的思想，进行哈希取余
- 使用 HashMap 统计每个小文件单词出现的次数
- 使用小顶堆，遍历步骤2中的小文件，找到词频 Top 100的单词。

**【步骤1：分桶处理】**

- 遍历大文件，读取每一行的词，并通过哈希函数将每个词分配到不同的桶（小文件）中。具体来说，使用 hash(x)%500（相同的词一定落在同一个小文件中），将词 ×分配到编号为i（0 ≤i< 500）的桶文件f（i）中。
- 通过哈希分桶，将大文件分割成大约500个小文件，每个小文件的大小约为 2MB（1GB / 500 ~ 2MB），保证每个小文件能够在内存限制内完成词频统计。

**【步骤2：统计每个桶中的词频】**

​	统计每个小文件中出现频率最高的100个词。可以用 HashMap 来实现，其中key 为词，value 为该次出现的频率（伪代码如下）。

- 对于遍历到的词 x，如果在 map 中不存在，则执行 map.put（x，1）。若存在，则执行 map.put （x，map.get（x）+ 1），将该词出现的次数＋1。

![image-20250424000200309](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250424000200309.png)

**【步骤3：合并所有桶中的 Top 100词】**

具体方法是：

1. 创建一个大小为 100 的**小顶堆（Min-Heap**）。
2. 遍历这 500个列表/哈希表中所有的（词，频率）对。
3. 对于每一个（词，频率）对：
   1. 如果堆的大小 ＜100，直接将该对（基于频率）加入堆中。
   2. 如果堆的大小 ==100，将当前词的频率与**堆顶元素（堆中频率最小的词）**的频率比较。
   3. 如果当前词的频率 ＞堆顶词的频率，则移除堆顶元素，并将当前（词，频率）对插入堆中。 • 当遍历完所有500 个小文件的所有词频信息后，小顶堆中剩下的100个元素就是全局频率最高的 100个词。

![image-20250424000428406](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250424000428406.png?token=ARFL34EH2VU2LLXAYEPB2QDI563LY)

#### 解法 2：多路归并排序方法

**【步骤1：多路归井排序对大文件进行排序】**

多路归并排序对大文件排序的步骤如下：

- 将1GB 的文件按顺序切分成多个小文件，每个文件大小不超过2MB，总共500个小文件。这样可以确保每个小文件在后续处理时能够被完全加载到内存中，满足 10MB 的内存限制。
- 使用10MB 的内存分别对每个小文件中的单词进行排序。这样可以确保每个小文件内部是有序的，这为后续的多路归并排序打下基础。
- 使用一个大小为 500的最小堆，将所有500个已排序的小文件进行合并，生成一个完全有序的文件。
  - 初始化一个最小堆，大小就是有序小文件的个数 500。**堆中的每个节点存放每个有序小文件对应的输入流**。
  - **按照每个有序文件中的下一行数据对所有文件输入流进行排序，单词小的输入文件流放在堆顶**。
  - 拿出堆顶的输入流，并且将下一行数据写入到最终排序的文件中，如果拿出来的输入流还有数据的话，那么就将这个输入流再次添加到堆中。否则说明该文件输入流中没有数据了，那么可以关闭这个流。
  - 循环这个过程，直到所有文件输入流中没有数据为止。

**【步骤2：统计出现频率最高的100个词】**

1. 初始化一个 100个节点的小顶堆，用于保存 100 个出现频率最高的单词。
2. 遍历整个文件，一个单词一个单词地从文件中读取出来，并且进行计数。
3. 等到遍历的单词和上一个单词不同的话，那么上一个单词及其频率如果大于堆顶的词的频率，那么放在堆中。否则不放

![image-20250424000810643](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250424000810643.png?token=ARFL34DMED52XUOAHB247ZLI563LW)

### 最热门的10个查询串

![image-20250424000942870](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1026 814"></svg>)

![image-20250424001007706](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250424001007706.png)

![image-20250424001027053](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1040 1124"></svg>)

### 每天热门 100词（分流＋ 哈希）

**题目描述**：某搜索公司一天的用户搜索词汇量达到百亿级别，请设计一种方法在内存和计算资源允许的情况下，求出每天热门的 Top 100词汇。

**分流＋ 哈希**

![image-20250424001251324](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1010 694"></svg>)

![image-20250424001318082](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1014 916"></svg>)

![image-20250424001344534](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1030 798"></svg>)





### 要求100亿个数里出现次数最多的10,000个数

【**核心思路**】

由于数据量巨大（100 亿个数），我们无法一次性将所有数据加载到内存中。因此，需要分步骤处理：

1. **分片处理** ：将数据分成多个小文件，每个文件包含一部分数据。
2. **分片统计频率** ：对每个分片单独统计频率，并将结果写入磁盘。
3. **合并频率** ：将所有分片的频率统计结果合并到一个全局哈希表或按需加载。
4. **提取前 10,000 个高频数** ：使用最小堆（Min-Heap）从全局频率中提取前 10,000 个高频数。

### 外部排序的核心思想

外部排序的基本思路是：

1. **分片** ：将大数据集分割成多个小块（chunk），每个小块可以单独加载到内存中。
2. **内部排序** ：对每个小块在内存中进行排序（如使用快速排序或归并排序）。
3. **多路归并** ：将所有已排序的小块合并成一个有序的整体。

## 如何提高数据库并发能力(大量访问)

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps11-4133580.jpg?token=ARFL34DQCUVIIZMLYUNVFSDI563L2) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps12-4133580.jpg?token=ARFL34HQ5B7NT3DI5XEJSY3I563LY) 

此外，一般应用对数据库而言都是“**读多写少**”，也就说对数据库读取数据的压力比较大，有一个思路就是采用数据库集群的方案，做主从架构、进行读写分离，这样同样可以提升数据库的并发处理能力。但并不是所有的应用都需要对数据库进行主从架构的设置，毕竟设置架构本身是有成本的。 

如果我们的目的在于提升数据库高并发访问的效率，那么：

- 首先考虑的是如何优化SQL和索引，这种方式简单有效；
- 其次才是采用缓存的策略，比如使用Redis将热点数据保存在内存数据库中，提升读取的效率；
- 最后才是对数据库采用主从架构，进行读写分离(主从复制)。

## select a,b,c from table where d=" order by e的索引设置

【**核心目标**】

1. **快速定位符合条件的数据行（WHERE 条件）**
2. **避免排序开销（ORDER BY）**
3. **尽量减少回表操作（减少 I/O）**

【**推荐：联合索引 `(d, e)`**】

**作用** 

- `d` 是等值条件，用于快速定位符合条件的记录。
- `e` 是排序字段，在 `d` 的范围内保持有序，使得数据库可以直接利用索引完成排序，避免 `filesort`。

**适用场景** 

- `d` 的选择性较高（即不同值较多）。
- 查询中 `d = 'value'` 的结果集较小。
- `e` 字段用于排序，且排序方向固定（如升序或降序）。

```sql
CREATE INDEX idx_d_e ON table (d, e);
```

【**索引生效原理**】

**等值条件 `WHERE d = 'value'` ：**

- 数据库会直接跳转到索引中 `d = 'value'` 的部分。

**排序 `ORDER BY e` ：**

- 在 `d = 'value'` 的子集中，`e` 字段在索引中已经是有序的。
- 因此，数据库可以**顺序读取索引条目** ，无需额外排序。



【**是否需要覆盖索引？**】

- **如果查询字段 `a, b, c` 都在索引中（即 `INDEX(d, e, a, b, c)`），可以避免回表操作，进一步提升性能。**
- 但代价是索引体积更大，维护成本更高。
- 建议在以下情况考虑使用覆盖索引：
  - 表数据量大，查询频繁；
  - `a, b, c` 字段较短或数量较少；
  - 对性能要求极高。

【**索引字段顺序重要**】

- 联合索引 `(d, e)` 与 `(e, d)` 效果完全不同。
- 若改为 `(e, d)`，`WHERE d = 'value'` 将无法高效利用索引。

【**避免使用两个单列索引**】

- 如 `INDEX(d)` 和 `INDEX(e)`，数据库可能只使用其中一个索引。
- 即使两者都被使用，也可能导致性能下降（如索引合并成本高）。

【**避免索引失效**】

- 不要在 `d` 上使用函数或表达式（如 `WHERE LOWER(d) = 'value'`），会导致索引失效。
- 确保 `d` 的值类型与字段定义一致（如不要将数字类型误传为字符串）。

【 **验证索引效果**】

- `key` 是否使用了预期的索引（如 `idx_d_e`）。
- `Extra` 字段是否出现 `Using filesort`（表示排序未使用索引）。
- `rows` 是否合理（越小越好）。

```sql
EXPLAIN SELECT a, b, c FROM table WHERE d = 'value' ORDER BY e;
```

## 插入数据对主键索引和普通索引有啥影响

### 对主键索引与普通索引的影响

【**主键索引与普通索引的特性**】

**主键索引**：

- **数据与索引一体** ：主键索引的叶子节点就是数据行本身（即数据行存储在主键索引的 B+ 树中）。
- **物理存储顺序** ：数据行的物理存储顺序与主键索引的逻辑顺序一致。

**普通索引**：

- **叶子节点为聚簇索引键值** ：每个叶子节点存储的是主键值（而非数据行地址）。
- **独立 B+ 树结构** ：普通索引是一个独立于主键索引的 B+ 树。

【 **插入数据时的影响**】

**1）主键索引**

**✅ 顺序插入（如自增 ID）**

- **优点** 
  - 新数据直接追加到索引末尾，无需查找插入位置。
  - 减少页分裂（Page Split），提高写入效率。
  - 提高 I/O 利用率，提升整体性能。

**❌ 随机插入（如无序字段）**

- **缺点** 
  - 每次插入需要找到合适的位置，导致页分裂。
  - 索引页碎片增加，降低缓存命中率。
  - 写入性能下降，尤其在高并发或大数据量场景下。

**2）普通索引**

**✅ 顺序插入（如时间戳字段）**

- **优点** 
  - 插入位置连续，B+ 树可以高效地追加新条目。
  - 减少页分裂，提高索引维护效率。
  - 提升范围查询性能（如 `WHERE create_time BETWEEN ...`）。

**❌ 随机插入（如无序字段）**

- **缺点** 
  - 插入位置不固定，容易引发页分裂。
  - 索引碎片增加，影响查询效率。
  - 回表操作（通过主键查找完整数据行）也可能变慢。

【**性能优化建议**】

**1）主键设计建议**

- **优先使用自增 ID** （`AUTO_INCREMENT`），避免使用 UUID。
- 如果必须使用 UUID，建议结合批量插入 + 索引重建优化。
- 对于分布式系统，考虑使用 Snowflake、NanoID 等有序唯一 ID 生成器。

**2）普通索引设计建议**

- 对于**频繁进行范围查询的字段（如时间戳、状态码），建立普通索引，并尽量保证插入顺序有序**。
- **避免对频繁更新的字段建立索引，减少索引维护成本**。
- 考虑使用覆盖索引（Covering Index）减少回表操作。

**3）控制索引数量**

- **避免冗余索引** ：仅创建对查询有显著加速效果的索引。
- **优先使用联合索引** ：减少索引个数，降低维护成本。



【**执行计划验证**】

使用 `EXPLAIN` 分析查询是否走索引，观察以下关键点：

| `TYPE`  | 是否为`RANGE`或`REF`（表示使用了索引）      |
| :------ | :------------------------------------------ |
| `Extra` | 是否出现`Using filesort`或`Using temporary` |
| `rows`  | 扫描行数是否合理（越小越好）                |

### select a,b,c from table where d=" order by e对于这个场景

**1. 联合索引 `(d, e)`**

**作用** 

- 加速 `WHERE d = "xxx"` 的过滤。
- 利用 `e` 的有序性避免 `filesort`。

**插入影响** 

- 每次插入新数据时，**数据库需要更新该索引的 B+ 树结构。**
- **如果 `d` 和 `e` 的值分布较散乱（如随机值），可能导致频繁的页分裂（Page Split），降低写入性能。**
- 如果 `d` 或 `e` 是单调递增（如时间戳、自增 ID），插入效率更高。

**2. 覆盖索引 `(d, e, a, b, c)`**

**作用** 

- 查询完全命中索引，无需回表访问主键索引。

**插入影响** 

- 索引体积更大，插入时需额外维护 `a, b, c` 字段的值。
- 写入成本更高，但查询性能最优。

**举例说明**

假设表结构如下：

```sql
CREATE TABLE example (
    id INT AUTO_INCREMENT PRIMARY KEY,
    d VARCHAR(50),
    e INT,
    a VARCHAR(100),
    b VARCHAR(100),
    c VARCHAR(100)
);
CREATE INDEX idx_d_e ON example (d, e);
```

![image-20250430125202773](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250430125202773.png?token=ARFL34D6NOYULHQNPLNTCFTI563L2)

![image-20250430125213285](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250430125213285.png?token=ARFL34E53M7E4F3W3GPM7P3I563L2)



## 如何实现幂等性

**1．幂等性含义**

意思是多次执行相同的操作，结果都是「相同」的

**Eg：计网中get操作就是幂等的，post则不是。**

 

**2．为什么要实现幂等性？**

在接口调用时一般情况下都能正常返回信息不会重复提交，不过在遇见以下情况时可以就会出现问题，如：

- 前端重复提交表单：在填写一些表格时候，用户填写完成提交，很多时候会因网络波动没有及时对用户做出提交成功响应，致使用户认为没有成功提交，然后一直点提交按钮，这时就会发生重复提交表单请求。
- 用户恶意进行刷单：例如在实现用户投票这种功能时，如果用户针对一个用户进行重复提交投票，这样会导致接口接收到用户重复提交的投票信息，这样会使投票结果与事实严重不符。
- 接口超时重复提交：很多时候HTTP客户端工具都默认开启超时重试的机制，尤其是第三方调用接口时候，为了防止网络波动超时等造成的请求失败，都会添加重试机制，导致一个请求提交多次。
- 消息进行重复消费：当使用MQ消息中间件时候，如果发生消息中间件出现错误未及时提交消费信息，导致发生重复消费。

**3．如何实现幂等性**

**1）数据库唯一主键实现幂等性**

- 数据库唯一主键的实现主要是利用数据库中主键唯一约束的特性，一般来说唯一主键比较适用于“插入”时的幂等性，其能保证一张表中只能存在一条带该唯一主键的记录。
- 使用数据库唯一主键完成幂等性时需要注意的是，该主键一般来说并不是使用数据库中自增主键，而是使用**分布式ID**充当主键，这样才能能保证在分布式环境下ID的全局唯一性。（就比如我们项目中用到的sharding jdbc的分布式主键生成方案，雪花算法，UUID等）

**2）防重Token令牌实现幂等性（基于Redis）**

- 针对客户端连续点击或者调用方的超时重试等情况，例如提交订单，此种操作就可以用Token的机制实现防止重复提交。
  - 简单的说就是调用方在调用接口的时候先向后端请求一个全局ID（Token），请求的时候携带这个全局ID一起请求（Token最好将其放到Headers中），后端需要对这个Token作为Key，用户信息作为Value到Redis中进行键值内容校验，如果Key存在且Value匹配就执行删除命令，然后正常执行后面的业务逻辑。如果不存在对应的Key或Value不匹配就返回重复执行的错误信息，这样来保证幂等操作

**这个就有点类似计网中防止CSRF攻击，也是利用了一个token令牌**

**CSRF(Cross Site Request Forgery)是指跨站请求伪造**。

- 进行Session认证的时候，我们一般使用Cookie来存储SessionId,当我们登陆后后端生成一个SessionId放在Cookie中返回给客户端，**服务端通过Redis或者其他存储工具记录保存着这个SessionId**，客户端登录以后每次请求都会带上这个SessionId，服务端通过这个SessionId来标示你这个人。**如果别人通过Cookie拿到了SessionId后就可以代替你的身份访问系统了**。
- Session认证中Cookie中的SessionId是由浏览器发送到服务端的，借助这个特性，攻击者就可以通过让用户误点攻击链接，达到攻击效果
- 但是，我们使用**Token**的话就不会存在这个问题，在我们登录成功获得Token之后，一般会选择存放在localStorage（浏览器本地存储）中。然后我们在前端通过某些方式会给每个发到后端的请求加上这个Token,这样就不会出现CSRF漏洞的问题。因为，即使有个你点击了非法链接发送了请求到服务端，这个非法请求是不会携带Token的，所以这个请求将是非法的。

## 如何保证消息队列的高可用（Kafka为例子）

**Kafka 的高可用性**

- Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点；你创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。
- 这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

​	实际上 RabbitMQ 之类的，并不是分布式消息队列，它就是传统的消息队列，只不过提供了一些集群、HA(High Availability, 高可用性) 的机制而已，因为无论怎么玩儿，RabbitMQ 一个 queue 的数据都是放在一个节点里的，镜像集群模式下，也是每个节点都放这个 queue 的完整数据。

- Kafka 0.8 以前，是没有 HA 机制的，就是任何一个 broker 宕机了，那个 broker 上的 partition 就废了，没法写也没法读，没有什么高可用性可言。比如说，我们假设创建了一个 topic，指定其 partition 数量是 3 个，分别在三台机器上。但是，如果第二台机器宕机了，会导致这个 topic 的 1/3 的数据就丢了，因此这个是做不到高可用的。

  ![kafka-before](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/kafka-before.png?token=ARFL34CSWKIVUEJQBRRSDVTI563L4)

**Kafka 的高可用性的体现**

- Kafka 0.8 以后，提供了 HA 机制，就是  **replica（复制品） 副本机制**。每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可。

- **为何只能读写 leader？**

  - 很简单，**要是你可以随意读写每个 follower，那么就要 care 数据一致性的问题**，系统复杂度太高，很容易出问题。Kafka 会均匀地将一个 partition 的所有 replica 分布在不同的机器上，这样才可以提高容错性。

    ![kafka-after](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/kafka-after.png)

- 这样就有**高可用性**了，因为如果某个 broker 宕机了，那个 broker 上面的 partition 在其他机器上都有副本的。如果这个宕机的 broker 上面有某个 partition 的 leader，那么此时会从 follower 中**重新选举**一个新的 leader 出来，大家继续读写那个新的 leader 即可。这就有所谓的高可用性了。

- **写数据**的时候，生产者就写 leader，然后 leader 将数据落地写本地磁盘，接着其他 follower 自己主动从 leader 来 pull 数据。一旦所有 follower 同步好数据了，就会发送 ack 给 leader，leader 收到所有 follower 的 ack 之后，就会返回写成功的消息给生产者。（当然，这只是其中一种模式，还可以适当调整这个行为）

- **消费**的时候，只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到。

## 如何保证消息不被重复消费、如何保证消息消费的幂等性？

**（1）为何会产生重复消费**

​	首先，比如 RabbitMQ、RocketMQ、Kafka，都有可能会出现消息重复消费的问题，正常。因为这问题通常不是 MQ 自己保证的，是由我们开发来保证的。

- Kafka 有 offset 的概念，就是每个消息写进去，都有一个 offset，代表消息的序号，然后 consumer 消费了数据之后，**每隔一段时间**（定时定期），会把自己消费过的消息的 offset 提交一下，表示“我已经消费过了，下次我要是重启啥的，你就让我继续从上次消费到的 offset 来继续消费吧”。
- 但是凡事总有意外，比如我们之前生产经常遇到的，就是你有时候重启系统，看你怎么重启了，如果碰到点着急的，直接 kill 进程了，再重启。这会导致 consumer 有些消息处理了，但是没来得及提交 offset，尴尬了。重启之后，少数消息会再次消费一次。（产生**分区再均衡**）

**eg：**

- 有这么个场景。数据 1/2/3 依次进入 Kafka，Kafka 会给这三条数据每条分配一个 offset，代表这条数据的序号，我们就假设分配的 offset 依次是 152/153/154。消费者从 Kafka 去消费的时候，也是按照这个顺序去消费。假如当消费者消费了 `offset=153` 的这条数据，刚准备去提交 offset 到 Zookeeper，此时消费者进程被重启了。那么此时消费过的数据 1/2 的 offset 并没有提交，Kafka 也就不知道你已经消费了 `offset=153` 这条数据。那么重启之后，消费者会找 Kafka 说，嘿，哥儿们，你给我接着把上次我消费到的那个地方后面的数据继续给我传递过来。由于之前的 offset 没有提交成功，那么数据 1/2 会再次传过来，如果此时消费者没有去重的话，那么就会导致重复消费。

- 如果消费者干的事儿是拿一条数据就往数据库里写一条，会导致说，你可能就把数据 1/2 在数据库里插入了 2 次，那么数据就错啦。

  ![mq-10](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/mq-10.png?token=ARFL34HRYGVYSIYF3D3J6Z3I563L4)

**（2）保证幂等性**

其实重复消费不可怕，可怕的是你没考虑到重复消费之后，**怎么保证幂等性**。

幂等性，通俗点说，就一个数据，或者一个请求，给你重复来多次，你得确保对应的数据是不会改变的，**不能出错**。

其实还是得结合业务来思考，我这里给几个思路：

- 比如你拿个数据要写库，你先根据主键查一下，如果这数据都有了，你就别插入了，update 一下好吧。
- 比如基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复数据插入只会报错，不会导致数据库中出现脏数据。
- 比如你是写 Redis，那没问题了，反正每次都是 set，天然幂等性。
- 比如你不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据的时候，里面加一个全局唯一的 id，类似订单 id 之类的东西，然后你这里消费到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个 id 写 Redis。如果消费过了，那你就别处理了，保证别重复处理相同的消息即可。
- 将 `enable.auto.commit`参数设置为 false，关闭自动提交，开发者在代码中手动提交 offset。那么这里会有个问题：什么时候提交 offset 合适？
  - 处理完消息再提交：依旧有消息重复消费的风险，和自动提交一样
  - 拉取到消息即提交：会有消息丢失的风险。允许消息延时的场景，一般会采用这种方式。然后，通过定时任务在业务不繁忙（比如凌晨）的时候做数据兜底。

## 如何设计一个高可用系统

**1）注重代码质量，测试严格把关**

避免代码中出现比较常见的内存泄漏、循环依赖的问题，这些问题对系统的可用性有极大的损害。

提升代码质量的工具:

- **sonarqube:**保证你写出更安全更干净的代码(ps:目前所在的项目基本都会用到这个插件)。
- **Alibaba开源的Java诊断工具Arthas**也是很不错的选择。

**2）使用缓存**

如果我们的系统属于并发量比较高的话，如果我们单纯使用数据库的话，当大量请求直接落到数据库可能数据库就会直接挂掉。使用缓存缓存热点数据，因为缓存存储在内存中，所以速度相当地快!

 

**3）使用多级缓存**

​	热点数据一定要放在缓存中，并且最好可以写入到jvm内存一份(多级缓存)，并设置个过期时间。需要注意写入到jvm的热点数据不宜过多，避免内存占用过大，一定要设置到淘汰策略。

为什么还要放在jvm内存一份?

- 因为放在jvm内存中的数据访问速度是最快的，不存在什么网络开销。

 

**4）使用集群,减少单点故障**

​	以Redis举个例子，我们需要使用resdis集群来保证Redis缓存高可用，避免单点故障。当我们使用一个Redis实例作为缓存的时候，这个Redis实例挂了之后，整个缓存服务可能就挂了。使用了集群之后，即使一台Redis实例挂掉，不到一秒就会有另外一台Redis实例顶上。

**5）限流**

​	**流量控制(flow control)**，其原理是监控应用**流量的QPS或并发线程数**等指标，当达到指定的阈值时对流量进行控制，以避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

**6）异步调用**

​	异步调用的话我们不需要关心最后的结果，这样我们就可以用户请求完成之后就立即返回结果，具体处理我们可以后续再做，秒杀场景用这个还是蛮多的。但是，使用异步之后我们可能需要适当修改业务流程进行配合，比如用户在提交订单之后，不能立即返回用户订单提交成功，需要在消息队列的订单消费者进程真正处理完该订单之后，甚至出库后，再通过电子邮件或短信通知用户订单成功。除了可以在程序中实现异步之外，我们常常还使用消息队列，消息队列可以通过异步处理提高系统性能（削峰、减少响应所需时间)并且可以降低系统耦合性。

**7）超时和重试机制设置(类比TCP通信等)**

​	一旦用户请求超过某个时间的得不到响应，就抛出异常。这个是非常重要的，很多线上系统故障都是因为没有进行超时设置或者超时设置的方式不对导致的。我们在读取第三方服务的时候，尤其适合设置超时和重试机制。一般我们使用一些RPC(远程调用)框架的时候，这些框架都自带的超时重试的配置。如果不进行超时设置可能会导致请求响应速度慢，甚至导致请求堆积进而让系统无法在处理请求。重试的次数一般设为3次，再多次的重试没有好处，反而会加重服务器压力〈部分场景使用失败重试机制会不太适合)。

**8）熔断机制**

​	超时和重试机制设置之外，熔断机制也是很重要的。熔断机制说的是系统自动收集所依赖服务的资源使用情况和性能指标，当所依赖的服务恶化或者调用失败次数达到某个阈值的时候就迅速失败，让当前系统立即切换依赖其他备用服务。比较常用的是流量控制和熔断降级框架是Netflix的Hystrix和alibaba的Sentinel。

## 如何设计一个高并发系统

### 需求分析与容量规划

1. **明确高并发场景特征**
   - 通过业务场景分析确定高并发类型（读多写少/突发流量/混合型）
   - 例如：电商秒杀（瞬时写高峰）、社交Feed流（读多写少）、物流轨迹查询（高频读+复杂查询）
2. **量化性能指标**
   - QPS（每秒查询量）/TPS（每秒事务数）目标
   - 响应时间（如核心接口需控制在50ms内）
   - 系统吞吐量（如日订单量百万级）
3. **容量评估与扩容策略**
   - 单机极限测试（如单机MySQL QPS约5k，Redis QPS 10w+）
   - 根据业务增长预测，设计水平扩展方案（如分库分表的分片数需预留3年扩展空间）

### 架构分层设计原则

**用户层 → 接入层 → 服务层 → 数据层 → 存储层**

- **接入层** ：Nginx/OpenResty实现负载均衡、动态限流（如令牌桶算法）
- **服务层** ：基于Dubbo/Spring Cloud的微服务化，实现服务治理（熔断、降级）
- **数据层** ：读写分离、分库分表、冷热分离
- **存储层** ：混合存储（MySQL+Redis+ES+HBase）

### 实施路径建议

【**阶段一：单体优化**】

**（1）代码级优化（减少锁粒度、批量操作）**

**批量操作**

- 批量接口设计 
  - 合并多次请求为一次（如批量下单接口`/order/batchCreate`）
  - 限制单次批量操作数据量（如单批最多处理200条）

**线程池优化**

可以通过分页方式逐步获取列表，同时结合多线程或异步请求技术并发拉取多个分页数据

- 使用`CompletableFuture`或自定义线程池提升并发效率。

**（2）数据库索引优化+慢SQL治理**

**索引优化**

- **最左前缀原则** 
  - 联合索引`(a,b,c)`可优化`WHERE a=1 AND b=2`，但无法优化`WHERE b=2`
- **覆盖索引** 
  - 索引包含查询字段（如`INDEX idx_user(age, name)`可直接返回`age`和`name`）

**慢SQL治理**

- **慢查询抓取** ：
  - 开启MySQL慢查询日志（`long_query_time=1`秒），explain具体的sql语句

**【阶段二：缓存机制与性能提升】**

- **使用场景**：**缓存频繁读取但更新不频繁的数据**，如用户资料、商品信息。使用 Redis 或 Memcached 存储，减少数据库查询时间。
- **实现**：缓存策略包括写穿（write-through）、写回（write-back），设置 TTL（生存时间）或使用 LRU 驱逐策略。
- **优势**：Redis 单机可支持数万并发，显著降低数据库压力。研究强调，缓存需平衡新鲜度和性能，实时性要求高的场景可能需短 TTL 或异步刷新 ([IGotAnOffer: System Design Interview Questions](https://igotanoffer.com/blogs/tech/system-design-interviews))。
- **挑战**：缓存一致性问题需注意，例如缓存失效后如何同步更新，需结合业务需求设计。

**【阶段三：消息队列（MQ）与异步处理】**

- **可能还是会出现高并发写的场景**，比如说一个业务操作里要频繁搞数据库几十次，增删改增删改。用 redis 来承载写那肯定不行，人家是缓存，数据随时就被 LRU 了，数据格式还无比简单，没有事务支持。
- **功能**：MQ 用于流量控制、削峰填谷和解耦。例如，大促期间通过 MQ 延迟处理非核心功能（如退款），集中资源处理订单和支付。
- **实现**：大量的写请求放入 MQ 里，然后慢慢排队，**后边系统消费后慢慢写**，控制在 mysql 承载范围之内，即通过 MQ 异步更新数据库，避免数据库即时写入压力。工具如 RabbitMQ、Kafka 可支持数万并发。
- **优势**：异步处理帮助系统应对突发流量，研究指出 MQ 可作为临时数据库，异步写入 DB ([Medium: Designing A High Concurrency, Low Latency System Architecture](https://medium.com/@markyangjw/designing-a-high-concurrency-low-latency-system-architecture-part-1-f5f3a5f32e36))。
- **挑战**：MQ 本身可能成为瓶颈，需优化设计，如增加节点或使用分布式 MQ。

【**阶段四：拆分服务**】

**按业务拆分微服务**

- **原理**：通过微服务架构，将单体应用拆分为多个独立服务，每个服务负责特定功能，**使用 Dubbo 作为 RPC 框架实现服务间通信。**然后每个系统连一个数据库，这样本来就一个库，现在多个数据库，不也可以扛高并发么。
- **实现**：例如，将用户管理、订单处理、支付等功能拆分为独立服务，每个服务可独立部署和扩展。每个服务连接自己的数据库，分散数据库负载。
- **优势**：分布式架构支持水平扩展，降低单点故障风险，提升并发能力。
- **挑战**：服务间通信需优化，需处理分布式事务和数据一致性问题，通常通过消息队列（MQ）实现最终一致性。

**【阶段五：分库分表与 Sharding】**

- **原理**：将数据库按用户 ID 或订单 ID 分片，分布到多个库和表。例如，按 UID 范围分片，查询时直接定位到对应分片。
- **实现**：分片规则可使用客户端分片框架，如 Mango，CSDN 文章提到分 8 个数据库，每个数据库 10 个表，规则为 (uid / 10) % 8 + 1 确定数据库，uid % 10 确定表。
- **优势**：分片后，每个查询只访问部分数据，减少单点压力，支持高并发读写。
- **挑战**：跨库查询复杂，需通过 MQ 实现最终一致性，实时监控确保数据同步。

【**阶段六：读写分离与负载均衡**】

- **架构**：主库处理写操作，从库处理读请求，通过 LVS（Linux Virtual Server）将读请求负载均衡到多个从库。
- **实现**：主库故障时，使用 KeepAlive 技术实现虚拟 IP 切换，秒级切换到备份数据库，确保高可用。
- **优势**：适合读多写少的场景，读流量可分布到多个从库，减轻主库压力。
- **挑战**：主库故障需快速切换，需确保数据复制延迟最小化。

**【阶段七：Elasticsearch 与搜索优化】**

- **功能**：Elasticsearch 是分布式搜索和分析引擎，适合复杂查询、全文搜索或实时分析。
- **实现**：将搜索和分析任务从主数据库卸载到 Elasticsearch，分布式架构可轻松扩展，支持高并发。
- **优势**：研究表明，Elasticsearch 适合处理大规模数据查询，性能优于传统数据库 ([Medium: Designing A High Concurrency, Low Latency System Architecture](https://medium.com/@markyangjw/designing-a-high-concurrency-low-latency-system-architecture-part-1-f5f3a5f32e36))。
- **挑战**：需优化索引和分片策略，避免查询性能下降

## 如何不通过压测预估项目的 QPS？

常见的性能测试工具，我们应该都不陌生了：

1. Jmeter:Apache JMeter 是 JAVA 开发的性能测试工具。
2. LoadRunner：一款商业的性能测试工具。
3. Galtling：一款基于 Scala 开发的高性能服务器性能测试工具。
4. ab：全称为 Apache Bench。Apache 旗下的一款测试工具，非常实用。

这个问题问的考察点在于，我们如何不利用这些工具来预估项目的 QPS。

一种常用的办法是，根据你的项目的日活跃用户数（DAU）来估算 QPS。这种方式比较简单，但需要项目有一定的用户数据和行为分析经验作为支撑。



- 假设我们的系统有100万的日活跃用户，每个用户日均发送10次请求。这样的话，总请求量为1000万，均值 QPS为1000万/（24 60 60） ≥116。但用户的访问也符合局部性原理，通常我们可以认为 20% 的时间集中了80% 的活跃用户访问，也就是说峰值时间占总时间的 20%。那么，峰值时间的QPS 为1000万 0.8/ （24 60 60 0.2） ~= 463。
- 这里预估的QPS 如果遇到像秒杀活动、限时抢购这种特殊的场景，还需要在峰值时间的 QPS 的基础上再乘以5 或者其他合适的倍数。

## 一个系统用户登录信息保存在服务器 A 上，服务器 B 如何获取到 Session 信息？

这道问题的本质是在问**分布式 Session 共享的解决方案。**



**SSO解决的是多个系统之间的统一登录，而Session共享解决的是同一个应用的不同实例之间的状态同步。**

- **基于Cookie的SSO**
- **基于Token的SSO**

​	正如题目描述的那样，假设一个系统用户登录信息保存在服务器 A上，该系统用户通过服务器 A 登录之后，需要访问服务器B的某个登录的用户才能访问的接口。假设 Session 信息只保存在服务器 A上，就会导致服务器 B 认为该用户并未登录。因此，我们需要让 Session 信息被所有的服务器都能访问到，也就是分布式 Session 共享。

【**Session 复制（实际项目中不会采用这种方案）**】

用户第一次访问某个服务器时，该服务器创建一个新的 Session，并将该 Session 的数据复制到其他服务器上。

这样，在用户的后续请求中，无论访问哪个服务器，都可以访问到相同的 Session 数据。

- 优点：数据安全（即使某些服务器宕机，也不会导致 Session 数据的丢失）
- 缺点：实现相对比较复杂、效率很低（尤其是服务器太多的情况下，Session 复制的成本太高）、内存占用大 （每个服务器都需要保存相同的 Session 数据）、数据不一致性问题（由于数据同步存在时间延迟，服务器之间的 Session 数据可能存在短暂的不一致）

【**分布式缓存保存（推荐）**】

将 Session 数据存储到分布式缓存比如 Redis 中，所有的服务器都可以访问。

这是目前最广泛使用的方案。

- 优点：性能优秀、支持横向扩展（如 Redis 集群）
- 缺点：存在数据丢失风险（虽然 Redis 支持多种数据持久化方式，但仍然可能会丢失小部分数据）

## 作为一个技术人员，保障技术、运维、服务的稳定，需要做什么

​	需要从多个维度进行系统性设计与落地。下面我将从 **技术架构、开发规范、部署运维、监控告警、故障响应** 等方面，详细说明一个技术人员在实际工作中应做的工作。

### 技术架构层面：架构分层设计

- **接入层** ：Nginx + Keepalived 实现负载均衡。

  - 接收外部请求（如用户访问、API 调用）
  - 实现流量分发（负载均衡）
  - 提供反向代理、SSL 终止等功能
  - 高可用部署（防止单点故障）

- **应用层** ：微服务化（Spring Cloud/Dubbo）+ 容器化（Docker/Kubernetes）。

  - 将业务逻辑拆分为多个独立服务

  - 每个服务可独立部署、升级、伸缩

  - 利用容器化技术实现快速发布和资源隔离

    ![image-20250512115708357](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250512115708357.png?token=ARFL34B333RCVD33ZFGT63DI563L6)

- **数据层** 

  - 数据库主从复制 + 分库分表（如 ShardingSphere）。
  - Redis 集群（Codis/Redis Cluster）。

- **消息队列层** ：Kafka/RabbitMQ 解耦服务。

- **缓存层** ：本地缓存（Caffeine）+ 分布式缓存（Redis）。

- **服务治理层** ：服务注册发现（Nacos/Eureka）、配置中心（Nacos/Spring Cloud Config）、链路追踪（SkyWalking/Jaeger）。

【**性能优化**】

- **代码优化** ：
  - 减少不必要的数据库查询，使用缓存（Redis、Memcached）加速热点数据访问。
  - 优化算法复杂度，减少 CPU 和内存消耗。
- **资源管理** ：
  - 合理配置线程池（`ThreadPoolExecutor`），避免线程数过多导致资源耗尽。
  - 定期分析 JVM GC 日志，优化堆内存分配。

【**自动化测试**】

- **单元测试** ：覆盖核心业务逻辑，确保代码质量。
- **集成测试** ：模拟外部依赖（如数据库、消息队列）进行端到端测试。
- **压力测试** ：使用工具（如 JMeter、Gatling）模拟高并发场景，评估系统性能瓶颈。

### 运维层面：保障基础设施的可靠性

【**基础设施监控**】

- **监控工具** ：
  - 使用 Prometheus + Grafana 监控服务器资源（CPU、内存、磁盘、网络）。
  - 配置日志收集系统（如 ELK Stack 或 Loki）分析异常日志。
- **关键指标** ：
  - 系统负载：CPU 使用率、内存占用、磁盘 I/O。
  - 应用性能：接口响应时间、吞吐量、错误率。
  - 数据库状态：连接数、慢查询、锁等待。

【**自动化运维**】

- **CI/CD 流程** ：
  - 使用 Jenkins、GitLab CI 实现自动化构建和部署。
  - 配置回滚机制，快速恢复到上一稳定版本。
- **容器化与编排** ：
  - 使用 Docker 容器化应用，确保环境一致性。
  - 使用 Kubernetes 实现自动化扩缩容和故障恢复。

【**备份与恢复**】

- **数据备份** ：
  - 定期备份数据库（如 MySQL、PostgreSQL）和文件存储。
  - 测试恢复流程，确保备份数据可用。
- **灾难恢复计划** ：
  - 制定详细的应急预案（如机房断电、网络中断）。
  - 模拟故障场景，定期演练恢复流程。

### 应急响应：快速处理突发问题

【**故障定位**】

- **日志分析** ：
  - 使用日志分析工具（如 ELK、Loki）快速定位问题。
  - 关注异常堆栈信息、错误码和请求上下文。
- **链路追踪** ：
  - 使用分布式追踪工具（如 Jaeger、Zipkin）分析服务调用链路，找出性能瓶颈。

【**快速恢复**】

- **回滚机制** ：
  - 如果新版本出现问题，快速回滚到上一稳定版本。
  - 使用蓝绿部署或金丝雀发布降低风险。
- **手动干预** ：
  - 在紧急情况下，临时扩容资源（如增加服务器实例）。
  - 手动重启服务或切换流量到备用节点。

【**事后复盘**】

- **问题总结** ：
  - 记录故障原因、影响范围、解决过程。
  - 分析根本原因（Root Cause Analysis, RCA）。
- **改进措施** ：
  - 优化系统设计或运维流程，避免类似问题再次发生。
  - 更新应急预案文档，完善监控和告警规则。

## 幂等和加锁、先加锁后判断幂等

**【幂等性】**

**1）幂等性** 指一个操作无论执行一次还是多次，其结果始终一致。例如：

- **HTTP 方法** ：`GET`（获取数据）、`PUT`（更新数据）、`DELETE`（删除数据）是幂等的，而 `POST`（创建数据）通常不是。
- **业务场景** ：支付接口中，同一笔订单的多次支付请求应只扣款一次。

**2）分布式系统中的幂等问题**

**常见场景**：

HTTP 请求重试 

- 客户端或网关在超时或失败后重发请求。

消息队列消费 

- 消息可能被重复投递（如 Kafka 的至少一次交付语义）。

分布式事务 

- 在分布式事务中，某些分支操作可能被重复提交。

接口调用

- 跨服务调用时，网络抖动可能导致重复调用。

**可能的问题**

- **数据重复** ：例如，支付系统中同一笔订单被扣款两次。
- **状态不一致** ：例如，订单状态被错误更新为“已支付”两次。
- **资源浪费** ：重复操作可能导致额外的计算、存储或网络开销。

**3）幂等的实现方法**

**1️⃣唯一请求 ID（Token 机制）**

- 客户端为每个请求生成唯一 ID（如 UUID、请求指纹）。
- 服务端通过缓存（如 Redis）或数据库记录已处理的请求 ID。
- 后续相同 ID 的请求直接返回结果，避免重复执行。

**适用场景**

- 高并发场景（如支付、下单）。
- 无状态服务（如 REST API）。

**2️⃣数据库约束（唯一索引）**

- 利用数据库唯一索引防止重复操作（如插入重复记录）。
- 适用于**创建型操作** （如用户注册、订单创建）。

**适用场景**

- 需要防止重复插入的场景（如唯一用户名、唯一订单号）。

**3️⃣幂等 Token（结合缓存）**

- 客户端在发起请求前先获取 Token，服务端缓存该 Token。
- 请求处理完成后删除 Token 或设置过期时间。

**适用场景**

- 防止表单重复提交、重复下单。



【**加锁** 】

- 目标：确保同一资源在同一时间只能被一个操作修改，防止并发问题（如超卖）。
- 实现方式：使用分布式锁（如 Redisson、ZooKeeper）或数据库锁。
- 特点：加锁通常在需要修改共享资源时使用，作用范围更细粒度。

**【先加锁后幂等】**

**1）为什么必须先加锁？**

**幂等校验与加锁之间存在“时间窗口”**

1. 请求A 和 请求B 同时到达 
   - 它们**同时检查状态** （如订单是否已支付）。
   - 此时状态为 `未支付`，**两者均通过幂等校验** （认为可以执行支付）。
2. **请求A 获取锁** ，执行支付逻辑，将状态更新为 `已支付`。
3. **请求B 等待锁释放后获取锁** ，此时状态已经是 `已支付`，但它**已经通过了幂等校验** ，仍会执行支付逻辑（如扣款）。

**问题本质**

- **幂等校验和加锁之间存在间隙** （时间窗口），导致多个请求通过校验后依次加锁并执行业务逻辑。
- **锁的作用范围仅保护业务逻辑，未保护幂等校验** ，无法阻止并发请求通过校验。

**2）正确顺序：加锁 → 幂等校验 → 业务逻辑**

1. **请求A 获取锁** ，进入临界区。
2. **请求B 等待锁释放** ，无法进入临界区。
3. 请求A 执行幂等校验 （检查订单状态）：
   - 若未支付，执行支付逻辑并更新状态。
4. 请求B 获取锁后，再次执行幂等校验：
   - 此时状态已变为 `已支付`，直接返回结果，避免重复操作。

**关键点**

- **锁保护整个临界区** （幂等校验 + 业务逻辑），确保同一时间只有一个请求能执行检查和操作。
- **幂等校验在锁内执行** ，避免并发请求通过校验后重复操作。

**3）无锁幂等设计（Token 机制）**

如果使用 **Token 机制** （如唯一请求 ID），可以在无锁情况下实现幂等：

1. **客户端生成唯一 Token** （如 `request_id`）。
2. **服务端记录 Token** （如写入 Redis 或数据库）。
3. **重复请求携带相同 Token** 时直接返回结果。

**适用场景** ：

- 高并发场景（如秒杀），避免锁竞争导致性能瓶颈。
- 业务逻辑本身无副作用（如查询）。

**缺点** ：

- 需维护 Token 状态，增加存储压力。
- 无法完全替代锁（如涉及库存扣减时仍需锁）。

**4️⃣分布式锁 + 幂等校验**

- 先加分布式锁（如 Redis 锁），再执行幂等校验。
- 确保校验和操作的原子性。

**适用场景**

- 涉及共享资源的分布式操作（如支付、库存扣减）。

# 项目排查问题的套路和工具

## 排查问题的套路

<u>**（1）在不同环境排查问题，有不同的方式**</u>

要说排查问题的思路，我们首先得明白是在什么环境排错。

- 如果是在自己的开发环境排查问题，那你几乎可以使用任何自己熟悉的工具来排查，甚至可以进行单步调试。只要问题能重现，排查就不会太困难，最多就是把程序调试到 JDK 或三方类库内部进行分析。
- 如果是在测试环境排查问题，相比开发环境少的是调试，不过你可以使用 JDK 自带的 jvisualvm 或阿里的Arthas，附加到远程的 JVM 进程排查问题。另外，测试环境允许造数据、造压力模拟我们需要的场景，因此遇到偶发问题时，我们可以尝试去造一些场景让问题更容易出现，方便测试。
- 如果是在生产环境排查问题，往往比较难：一方面，生产环境权限管控严格，一般不允许调试工具从远程附加进程；另一方面，生产环境出现问题要求以恢复为先，难以留出充足的时间去慢慢排查问题。但，因为生产环境的流量真实、访问量大、网络权限管控严格、环境复杂，因此更容易出问题，也是出问题最多的环境。

<u>**（2）生产问题的排查很大程度依赖监控**</u>

**==主要依赖日志和监控==**

<u>**1）日志主要注意两点：**</u>

-  确保错误、异常信息可以被完整地记录到文件日志中；
-  确保生产上程序的日志级别是 INFO 以上。记录日志要使用合理的日志优先级，**==DEBUG 用于开发调试、INFO 用于重要流程信息、WARN 用于需要关注的问题、ERROR 用于阻断流程的错误。==**

<u>**2）对于监控，在生产环境排查问题时，首先就需要开发和运维团队做好充足的监控，而且是多个层次的监控。**</u>

- 主机层面，对 CPU、内存、磁盘、网络等资源做监控。如果应用部署在虚拟机或 Kubernetes 集群中，那么除了对物理机做基础资源监控外，还要对虚拟机或 Pod 做同样的监控。监控层数取决于应用的部署方案，有一层 OS 就要做一层监控。
- 网络层面，需要监控专线带宽、交换机基本情况、网络延迟。
- 所有的中间件和存储都要做好监控，不仅仅是监控进程对 CPU、内存、磁盘 IO、网络使用的基本指标，更重要的是监控组件内部的一些重要指标。比如，著名的监控工具 Prometheus，就提供了大量的exporter来对接各种中间件和存储系统。
- 应用层面，需要监控 JVM 进程的类加载、内存、GC、线程等常见指标（比如使用Micrometer来做应用监控），此外还要确保能够收集、保存应用日志、GC 日志

<u>**3）快照**</u>

- 这里的“快照”是指，应用进程在某一时刻的快照。
- 通常情况下，我们会为生产环境的 Java 应用设置 <font color = '#8D0101'>-XX:+HeapDumpOnOutOfMemoryError 和 -XX:HeapDumpPath=…这 2 个 JVM 参数</font>，用于在出现 OOM 时保留堆快照。这个课程中，我们也多次使用 MAT 工具来分析堆快照。

**<u>（3）分析定位问题的套路</u>**

定位问题，首先要定位问题出在哪个层次上。

- 比如，是 Java 应用程序自身的问题还是外部因素导致的问题。我们可以先查看程序是否有异常，异常信息一般比较具体，可以马上定位到大概的问题方向；
- 如果是一些资源消耗型的问题可能不会有异常，我们可以通过指标监控配合显性问题点来定位。

**一般情况下，程序的问题来自以下三个方面。**

**<u>1）程序发布后的 Bug</u>**，回滚后可以立即解决。这类问题的排查，可以回滚后再慢慢分析版本差异。

<u>**2）外部因素**</u>，比如主机、中间件或数据库的问题。这类问题的排查方式，按照主机层面的问题、中间件或存储（统称组件）的问题分为两类。

1️⃣主机层面的问题，可以使用工具排查：

- **CPU 相关问题，可以使用 <font color = '#8D0101'>top、vmstat、pidstat、ps </font>等工具排查；**
- **内存相关问题，可以使用 <font color = '#8D0101'>free、top、ps、vmstat、cachestat、sar</font> 等工具排查；**
- **IO 相关问题，可以使用 <font color = '#8D0101'>lsof、iostat、pidstat、sar、iotop、df、du </font>等工具排查；**
- **网络相关问题，可以使用 <font color = '#8D0101'>ifconfig、ip、nslookup、dig、ping、tcpdump、iptables</font> 等工具排查**

2️⃣组件的问题，可以从以下几个方面排查：

- 排查组件所在主机是否有问题；
- 排查组件进程基本情况，观察各种监控指标；查看组件的日志输出，特别是错误日志；
- 进入组件控制台，使用一些命令查看其运作情况

**<u>3）第三，因为系统资源不够造成系统假死的问题</u>**，通常需要先通过重启和扩容解决问题，之后再进行分析，不过最好能留一个节点作为现场。系统资源不够，一般体现在 CPU 使用高、内存泄漏或 OOM 的问题、IO 问题、网络相关问题这四个方面。

1️⃣对于 CPU 使用高的问题，如果现场还在，具体的分析流程是：

- **首先，在 Linux 服务器上运行 `top -Hp pid` 命令**查看线程级 CPU 占用**，来查看进程中哪个线程 CPU 使用高；**

  - `top` 默认按进程显示资源占用，**<font color = '#8D0101'>`-H` 选项会显示进程内的所有线程（Thread），`-p` 指定目标进程 PID。</font>**
  - 关键输出：`PID`：线程 ID（十进制）、`%CPU`：线程的 CPU 占用百分比。

- **然后，<font color = '#8D0101'>输入大写的 P 将线程按照 CPU 使用率排序，并把明显占用 CPU 的线程 ID 转换为 16 进制</font>（`jstack` 输出的线程 ID 是十六进制格式，而 `top` 显示的是十进制。）；**

- **最后，<font color = '#8D0101'>在 jstack \<pid>命令输出的线程栈中搜索这个线程 ID，定位出问题的线程当时的调用栈</font>**

  - **eg:`jstack <pid> ｜ grep -C 10 "nid=0x3039"`**

- ==举例如下：==

  - 找到高 CPU 线程的十进制 ID 

    ```bash
    top -Hp 5678  # 假设进程 PID 是 5678
    \# 输出中发现线程 ID 12345 占用 CPU 过高
    ```

  - 转换为十六进制 

    ```bash
    printf "%x\n" 12345  # 输出 3039
    ```

  - 在 `jstack` 输出中搜索 

    ```bash
    jstack 5678 | grep -C 10 "nid=0x3039"
    #匹配到的堆栈信息会显示该线程的代码执行路径。
    ```

2️⃣如果现场没有了，我们可以通过排除法来分析。CPU 使用高，一般是由下面的因素引起的：

- <u>突发压力</u>。这类问题，我们可以通过应用之前的负载均衡的流量或日志量来确认，诸如 Nginx 等反向代理都会记录 URL，可以依靠代理的 Access Log 进行细化定位，也可以通过监控观察 JVM 线程数的情况。压力问题导致 CPU 使用高的情况下，如果程序的各资源使用没有明显不正常，之后可以通过压测 +Profiler（jvisualvm 就有这个功能）进一步定位热点方法；如果资源使用不正常，比如产生了几千个线程，就需要考虑调参。
- <u>GC</u>。这种情况，我们可以通过 JVM 监控 GC 相关指标、GC Log 进行确认。如果确认是 GC 的压力，那么内存使用也很可能会不正常，需要按照内存问题分析流程做进一步分析。
- <u>**程序中死循环逻辑或不正常的处理流程。**</u>这类问题，我们可以结合应用日志分析。一般情况下，应用执行过程中都会产生一些日志，可以重点关注日志量异常部分。

3️⃣注意：

- 需要注意的是，Java 进程对内存的使用不仅仅是<font color = '#8D0101'>堆区</font>，还包括<font color = '#8D0101'>线程使用的内存（线程个数 * 每一个线程的线程栈）和元数据区</font>。每一个内存区都可能产生 OOM，可以结合监控观察线程数、已加载类数量等指标分析。另外，我们需要注意看一下，JVM 参数的设置是否有明显不合理的地方，限制了资源使用。
- IO 相关的问题，除非是代码问题引起的资源不释放等问题，否则通常都不是由 Java 进程内部因素引发的
- 网络相关的问题，一般也是由外部因素引起的。对于连通性问题，结合异常信息通常比较容易定位；对于性能或瞬断问题，可以先尝试使用 ping 等工具简单判断，如果不行再使用 tcpdump 或 Wireshark 来分析。

## 分析和定位问题需要注意的九个点

<u>**（1）考虑“鸡”和“蛋”的问题**</u>。比如，发现业务逻辑执行很慢且线程数增多的情况时，我们需要考虑两种可能性：

- 一是，程序逻辑有问题或外部依赖慢，使得业务逻辑执行慢，在访问量不变的情况下需要更多的线程数来应对。比如，10TPS 的并发原先一次请求 1s 可以执行完成，10 个线程可以支撑；现在执行完成需要 10s，那就需要 100 个线程。
- 二是，有可能是请求量增大了，使得线程数增多，应用本身的 CPU 资源不足，再加上上下文切换问题导致处理变慢了。

出现问题的时候，我们需要结合内部表现和入口流量一起看，确认这里的“慢”到底是根因还是结果。

<u>**（2）考虑通过分类寻找规律**</u>。在定位问题没有头绪的时候，我们可以尝试总结规律。

- 比如，我们有 10 台应用服务器做负载均衡，出问题时可以通过日志分析是否是均匀分布的，还是问题都出现在 1 台机器。
- 又比如，应用日志一般会记录线程名称，出问题时我们可以分析日志是否集中在某一类线程上。
- 再比如，如果发现应用开启了大量 TCP 连接，通过 netstat 我们可以分析出主要集中连接到哪个服务。如果能总结出规律，很可能就找到了突破点。

<u>**（3）分析问题需要根据调用拓扑来，不能想当然**</u>。

- 比如，我们看到 <font color = '#8D0101'>Nginx 返回 502 错误，一般可以认为是下游服务的问题导致网关无法完成请求转发</font>。对于下游服务，不能想当然就认为是我们的 Java 程序，比如在拓扑上可能 Nginx 代理的是 Kubernetes 的 Traefik Ingress，链路是 Nginx->Traefik-> 应用，如果一味排查 Java 程序的健康情况，那么始终不会找到根因。
- 又比如，我们虽然使用了 Spring Cloud Feign 来进行服务调用，出现连接超时也不一定就是服务端的问题，有可能是客户端通过 URL 来调用服务端，并不是通过 Eureka 的服务发现实现的客户端负载均衡。换句话说，客户端连接的是 Nginx 代理而不是直接连接应用，客户端连接服务出现的超时，其实是 Nginx 代理宕机所致。

<u>**（4）考虑资源限制类问题。**</u>

观察各种曲线指标，如果发现曲线慢慢上升然后稳定在一个水平线上，那么一般就是资源达到了限制或瓶颈。比如，在观察网络带宽曲线的时候，如果发现带宽上升到 120MB 左右不动了，那么很可能就是打满了 1GB 的网卡或传输带宽。又比如，观察到数据库活跃连接数上升到 10 个就不动了，那么很可能是连接池打满了。观察监控一旦看到任何这样的曲线，都要引起重视。

**<u>（5）考虑资源相互影响。</u>**

- CPU、内存、IO 和网络，这四类资源就像人的五脏六腑，是相辅相成的，一个资源出现了明显的瓶颈，很可能会引起其他资源的连锁反应。
- 比如，内存泄露后对象无法回收会造成大量 Full GC，此时 CPU 会大量消耗在 GC 上从而引起 CPU 使用增加。
- 又比如，我们经常会把数据缓存在内存队列中进行异步 IO 处理，网络或磁盘出现问题时，就很可能会引起内存的暴涨。因此，出问题的时候，我们要考虑到这一点，以避免误判。

**<u>（6）排查网络问题要考虑三个方面，到底是客户端问题，还是服务端问题，还是传输问题</u>**

- 比如，出现数据库访问慢的现象，可能是客户端的原因，连接池不够导致连接获取慢、GC 停顿、CPU 占满等；
- 也可能是传输环节的问题，包括光纤、防火墙、路由表设置等问题；
- 也可能是真正的服务端问题，需要逐一排查来进行区分。

​	<font color = '#8D0101'>服务端慢一般可以看到 MySQL 出慢日志，传输慢一般可以通过 ping 来简单定位</font>，排除了这两个可能，并且仅仅是部分客户端出现访问慢的情况，就需要怀疑是客户端本身的问题。对于第三方系统、服务或存储访问出现慢的情况，不能完全假设是服务端的问题。

**<u>（7）快照类工具和趋势类工具需要结合使用。</u>**

- 比如，**<font color = '#8D0101'>jstat、top、各种监控曲线是趋势类工具，可以让我们观察各个指标的变化情况，定位大概的问题点；而 jstack 和分析堆快照的 MAT 是快照类工具，用于详细分析某一时刻应用程序某一个点的细节。</font>**
- 一般情况下，我们会先使用趋势类工具来总结规律，再使用快照类工具来分析问题。如果反过来可能就会误判，因为快照类工具反映的只是一个瞬间程序的情况，不能仅仅通过分析单一快照得出结论，如果缺少趋势类工具的帮助，那至少也要提取多个快照来对比。

**<u>（8）不要轻易怀疑监控。</u>**

- 如果你真的怀疑是监控系统有问题，可以看一下这套监控系统对于不出问题的应用显示是否正常，如果正常那就应该相信监控而不是自己的经验。

**<u>（9）如果因为监控缺失等原因无法定位到根因的话，相同问题就有再出现的风险</u>**

需要做好三项工作：

- 做好日志、监控和快照补漏工作，下次遇到问题时可以定位根因；
- 针对问题的症状做好实时报警，确保出现问题后可以第一时间发现；
- 考虑做一套热备的方案，出现问题后可以第一时间切换到热备系统快速解决问题，同时又可以保留老系统的现场。

## JVM 命令行工具

以下是较常用的 JDK 命令行工具：

| 名称     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| `jps`    | JVM 进程状态工具。显示系统内的所有 JVM 进程。                |
| `jstat`  | JVM 统计监控工具。监控虚拟机运行时状态信息，它可以显示出 JVM 进程中的类装载、内存、GC、JIT 编译等运行数据。 |
| `jmap`   | JVM 堆内存分析工具。用于打印 JVM 进程对象直方图、类加载统计。并且可以生成堆转储快照（一般称为 heapdump 或 dump 文件）。 |
| `jstack` | JVM 栈查看工具。用于打印 JVM 进程的线程和锁的情况。并且可以生成线程快照（一般称为 threaddump 或 javacore 文件）。 |
| `jhat`   | 用来分析 jmap 生成的 dump 文件。                             |
| `jinfo`  | JVM 信息查看工具。用于实时查看和调整 JVM 进程参数。          |
| `jcmd`   | JVM 命令行调试 工具。用于向 JVM 进程发送调试命令。           |

## 栈溢出StackOverflowError

<font color = '#8D0101'>**对于 HotSpot 虚拟机来说，栈容量只由 `-Xss` 参数来决定**。**如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 `StackOverflowError` 异常。**</font>

从实战来说，栈溢出的常见原因：

- **递归函数调用层数太深**
- **大量循环或死循环**

【示例】递归函数调用层数太深导致 `StackOverflowError`

```java
public class StackOverflowDemo {

    private int stackLength = 1;

    public void recursion() {
        stackLength++;
        recursion();
    }

    public static void main(String[] args) {
        StackOverflowDemo obj = new StackOverflowDemo();
        try {
            obj.recursion();
        } catch (Throwable e) {
            System.out.println("栈深度：" + obj.stackLength);
            e.printStackTrace();
        }
    }

}
```

## 内存溢出 OutOfMemoryError

​	`OutOfMemoryError` 简称为 OOM。Java 中对 OOM 的解释是，没有空闲内存，并且垃圾收集器也无法提供更多内存。通俗的解释是：JVM 内存不足了。

​	在 JVM 规范中，**除了程序计数器区域外，其他运行时区域都可能发生 `OutOfMemoryError` 异常（简称 OOM）**。

### Java heap space 分析步骤

<u>**（1）使用 `jmap` 或 `-XX:+HeapDumpOnOutOfMemoryError` 获取堆快照。**</u>

**【jmap命令】**

- `-dump:live`：触发 Full GC，只导出存活对象（推荐使用，避免快照中包含垃圾对象）。
- `format=b`：生成二进制格式的快照文件（`.hprof`）。
- `file=<输出路径>`：指定快照文件的保存路径（如 `/tmp/heap.hprof`）。
- `<进程ID>`：目标 Java 进程的 PID（通过 `jps` 或 `ps` 命令获取）。

**注意：应用暂停 ：**

- `jmap -dump:live` 会触发 Full GC，导致应用暂停（STW，Stop-The-World），时间可能从几毫秒到数秒不等。
- **生产环境慎用** ，建议在低峰期操作。

```bash
jmap -dump:live,format=b,file=<输出路径> <进程ID>
```

【 **`-XX:+HeapDumpOnOutOfMemoryError` 和 `-XX:HeapDumpPath` 参数**】

通过 `-XX:+HeapDumpOnOutOfMemoryError` 和 `-XX:HeapDumpPath` 参数定位堆栈文件（`.hprof`）的生成路径。

![image-20250514144511130](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514144511130.png?token=ARFL34DPLU6AQ7JVSJU6UUTI563L6)

**<u>2）使用内存分析工具（visualvm、mat、jProfile 等）对堆快照文件进行分析。</u>**

**步骤** 

1. 打开 `jvisualvm`（JDK 自带）。
2. 文件 → 装入 `.hprof` 文件。
3. 查看 **类视图** ，按实例数或内存占用排序，定位异常类。
4. 右键类 → **最近的垃圾回收根节点** ，分析引用链。

**<u>3）根据分析图，重点是确认内存中的对象是否是必要的，分清究竟是是内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）。</u>**



### OOM排查例子

#### 问题背景

​	在近期测试环境中，应用进程频繁被 `OutOfMemoryError`（OOM）异常触发的 JVM 机制强制终止，而生产环境从未发生类似问题。

![image-20250514151546545](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250514151546545-7208511.png)

初步怀疑可能由以下原因导致：

1. **JVM 参数配置差异** ：生产环境可能分配了更大的堆内存或更合理的 GC 策略。
2. **代码逻辑差异** ：测试环境可能引入了特定的测试组件或配置。
3. **依赖组件行为异常** ：某些仅在测试环境启用的组件存在内存泄漏。

#### 问题排查流程

**（1）定位堆栈文件**

- **JVM 参数配置** ：通过 `-XX:+HeapDumpOnOutOfMemoryError` 和 `-XX:HeapDumpPath` 参数定位堆栈文件（`.hprof`）的生成路径。
  - 或者：![image-20250514151733172](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151733172-7208511.png?token=ARFL34E2NSXFAYMH6H6MUHTI563MA)，定位到了虚拟机的日志的输出目录，用简单的命令寻找到堆栈文件：![image-20250514151755128](/Users/glexios/Library/Mobile%2520Documents/com~apple~CloudDocs/Notes/%25E9%259D%25A2%25E8%25AF%2595%25E8%25A1%25A5%25E5%2585%2585%25E7%25AC%2594%25E8%25AE%25B0.assets/image-20250514151755128.png)
  - ![image-20250514151810247](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151810247-7208511.png?token=ARFL34A7WVYPYGUNCPIJHULI563ME)

- **容器环境限制** ：因测试环境部署在容器中，需通过 **容器 → 跳板机 → SFTP 服务器 → 本地开发机** 的链路下载堆栈文件

**（2）堆栈文件分析**

![image-20250514151936510](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151936510-7208511.png?token=ARFL34AUHVICP57D2QQJYYTI563MC)

​	通过 jvisualvm.exe 应用 文件 -> 装入 的方式导入刚才下载的文件，载入之后会有这样的选项卡

- 在概要的地方就给出了出问题的关键线程：发现 OOM 原因为 **堆内存溢出（Java heap space）** ，将堆栈信息点进去，发现堆栈是由于字符串的复制操作导致发生了 oom。
- 同时通过堆栈信息可以定位到 UnitTestContextHolder.java 类
- ![image-20250514152614559](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250514152614559-7208511.png)

**【类分析】**

​	通过类分析视图，我们可以简单的定位到占据内存最高的类。**这里可以按照 实例数 或者 内存大小进行排序**，查询是否有某个类的实例数量异常，比如远远的高于其他的类，通常这些异常值就是可能发生内存泄露的地方。

- 我们这里由于 char 类实例数比较高，所有我优先往内存泄漏方向思考。

![image-20250514152522535](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514152522535-7208511.png?token=ARFL34ATLDG2GGELU7TSGWLI563ME)

**【实例数分析】**

![image-20250514152658794](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514152658794-7208511.png?token=ARFL34HUC2F4OY6B5YBI56TI563ME)

发现全是这种字符的实例，选中【实例】，右键可以分析【最近的垃圾回收节点】

这里会显示这些对象为何没有被回收，一般是静态变量或者是全局对象引用导致。

![image-20250514153311959](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153311959-7208511.png?token=ARFL34GRXI2FUTCJIFPSPTLI563MG)

通过上面图片可以看出，图中主要是由静态字段导致

**（3）代码分析**

​	代码是发生在项目中引入的企同的试点单元测试组件，刚好我们项目参与了试点，由于是单元测试组件，只在编译或者测试的依赖包中引入，正式的未引入

![image-20250514153410531](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250514153410531-7208511.png)

![image-20250514153435758](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250514153435758-7208511.png)

这里使用到了 ThreadLocal 做一些对象的存储，但是我发现 调用 setAttribute(String key,Object value)的地方有 33 处

![image-20250514153502559](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250514153502559-7208511.png)

但是调用 clear()的地方只有3处。正常来说凡是使用 ThreadLocal 的地方，set 和 clear() 都应该成对出现。

![image-20250514153516478](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153516478-7208511.png?token=ARFL34HDNQBLJM2NOG5YRJ3I563MI)

**<u>（4）原因分析</u>**

==ThreadLocal 内存泄漏== 

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

### 堆空间溢出

`java.lang.OutOfMemoryError: Java heap space` 这个错误意味着：**堆空间溢出**。

- 更细致的说法是：Java 堆内存已经达到 `-Xmx` 设置的最大值。Java 堆用于存储对象实例，只要不断地创建对象，并且保证 GC Roots 到对象之间有可达路径来避免垃圾收集器回收这些对象，那么当堆空间到达最大容量限制后就会产生 OOM。

​	堆空间溢出有可能是<font color = '#8D0101'>**`内存泄漏（Memory Leak）`** 或 **`内存溢出（Memory Overflow）`**</font> 。需要使用 jstack 和 jmap 生成 threaddump 和 heapdump，然后用内存分析工具（如：MAT）进行分析。

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250313221216979.png" alt="image-20250313221216979" style="zoom:67%;" />



#### 内存泄漏

**严格来说只有<font color = '#8D0101'>对象不会再被程序用到了，但是GC又不能回收他们的情况</font>，才叫内存泄漏。**

- 但实际情况很多时候一些不太好的实践（或疏忽）会导致**对象的生命周期变得很长甚至导致OOM，也可以叫做宽泛意义上的“内存泄漏”** 。
- 尽管内存泄漏并不会立刻引起程序崩溃，但是一旦发生内存泄漏，程序中的可用内存就会被逐步蚕食，直至耗尽所有内存，最终出现OutOfMemory异常， 导致程序崩溃。

<u>**内存泄漏常见场景：**</u>

- **单例模式**
  - 单例的生命周期和应用程序是一样长的，所以单例程序中，如果持有对外部对象的引用的话，那么这个外部对象是不能被回收的，则会导致内存泄漏的产生

- **静态容器**
  - 声明为静态（`static`）的 `HashMap`、`Vector` 等集合
  - 通俗来讲 A 中有 B，当前只把 B 设置为空，A 没有设置为空，回收时 B 无法回收。因为被 A 引用。
- **监听器**
  - 监听器被注册后释放对象时没有删除监听器
- **物理连接**
  - 各种连接池建立了连接，必须通过 `close()` 关闭链接
- **内部类和外部模块等的引用**
  - 发现它的方式同内存溢出，可再加个实时观察
  - `jstat -gcutil 7362 2500 70`
- **导致内存泄漏的常见原因是使用容器，且不断向容器中添加元素，但没有清理，导致容器内存不断膨胀。**

**内存泄露eg：**

```java
/**
 * 内存泄漏示例
 * 错误现象：java.lang.OutOfMemoryError: Java heap space
 * VM Args：-verbose:gc -Xms10M -Xmx10M -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOutOfMemoryDemo {

    public static void main(String[] args) {
        List<OomObject> list = new ArrayList<>();
        while (true) {
            list.add(new OomObject());
        }
    }
    static class OomObject {}
}
```

####  内存溢出

​	如果不存在内存泄漏，即内存中的对象确实都必须存活着，**则应当检查虚拟机的堆参数（`-Xmx` 和 `-Xms`），与机器物理内存进行对比，看看是否可以调大**。并从代码上检查是否存在某些对象生命周期过长、持有时间过长的情况，尝试减少程序运行期的内存消耗。

**eg：**

- 下面的例子是一个极端的例子，试图创建一个维度很大的数组，堆内存无法分配这么大的内存，从而报错：`Java heap space`。
- 但如果在现实中，代码并没有问题，**仅仅是因为堆内存不足，可以通过 `-Xms` 和 `-Xmx` 适当调整堆内存大小**。

```java
/**
 * 堆溢出示例
 * <p>
 * 错误现象：java.lang.OutOfMemoryError: Java heap space
 * <p>
 * VM Args：-verbose:gc -Xms10M -Xmx10M
 *
 * @author <a href="mailto:forbreak@163.com">Zhang Peng</a>
 * @since 2019-06-25
 */
public class HeapOutOfMemoryDemo {

    public static void main(String[] args) {
        Double[] array = new Double[999999999];
        System.out.println("array length = [" + array.length + "]");
    }

}
```

### GC 开销超过限制

​	`java.lang.OutOfMemoryError: GC overhead limit exceeded` 这个错误，官方给出的定义是：**超过 `98%` 的时间用来做 GC 并且回收了不到 `2%` 的堆内存时会抛出此异常**。这意味着，发生在 GC 占用大量时间为释放很小空间的时候发生的，是一种保护机制。

- **导致异常的原因：一般是因为堆太小，没有足够的内存。**

**【示例】**

- 与 **Java heap space** 错误处理方法类似，先判断是否存在内存泄漏。如果有，则修正代码；如果没有，则通过 `-Xms` 和 `-Xmx` 适当调整堆内存大小。

```java
/**
 * GC overhead limit exceeded 示例
 * 错误现象：java.lang.OutOfMemoryError: GC overhead limit exceeded
 * 发生在GC占用大量时间为释放很小空间的时候发生的，是一种保护机制。导致异常的原因：一般是因为堆太小，没有足够的内存。
 * 官方对此的定义：超过98%的时间用来做GC并且回收了不到2%的堆内存时会抛出此异常。
 * VM Args: -Xms10M -Xmx10M
 */
public class GcOverheadLimitExceededDemo {

    public static void main(String[] args) {
        List<Double> list = new ArrayList<>();
        double d = 0.0;
        while (true) {
            list.add(d++);
        }
    }

}
```

### 永久代（方法区）空间不足

【错误】

```text
java.lang.OutOfMemoryError: PermGen space
```

【原因】

​	Perm （永久代）空间主要用于存放 `Class` 和 Meta 信息，包括类的名称和字段，带有方法字节码的方法，常量池信息，与类关联的对象数组和类型数组以及即时编译器优化。GC 在主程序运行期间不会对永久代空间进行清理，默认是 64M 大小。

- 根据上面的定义，可以得出 **PermGen 大小要求取决于加载的类的数量以及此类声明的大小**。因此，**可以说造成该错误的主要原因是永久代中装入了太多的类或太大的类。**
- **在 JDK8 之前的版本中，可以通过 `-XX:PermSize` 和 `-XX:MaxPermSize` 设置永久代空间大小，从而限制方法区大小，并间接限制其中常量池的容量。**

【解决方案】

​	在应用程序启动期间触发由于 PermGen 耗尽导致的 `OutOfMemoryError` 时，解决方案很简单。该应用程序仅需要更多空间才能将所有类加载到 PermGen 区域，因此我们只需要增加其大小即可。为此，更改你的应用程序启动配置并添加（或增加，如果存在）`-XX:MaxPermSize` 参数，类似于以下示例：上面的配置将告诉 JVM，PermGen 可以增长到 512MB。

```text
java -XX:MaxPermSize=512m com.yourcompany.YourClass
```

### 元数据区空间不足

【错误】

```text
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```

【原因】

Java8 以后，JVM 内存空间发生了很大的变化。取消了永久代，转而变为元数据区。

**元数据区的内存不足，即方法区和运行时常量池的空间不足**。

- 一个类要被垃圾收集器回收，判定条件是比较苛刻的。在经常动态生成大量 Class 的应用中，需要特别注意类的回收状况，比如 **CGLib 字节码增强和动态语言**

【解决】

当由于元空间而面临 `OutOfMemoryError` 时，第一个解决方案应该是显而易见的。如果应用程序耗尽了内存中的 Metaspace 区域，则应增加 Metaspace 的大小。更改应用程序启动配置并增加以下内容：

```text
-XX:MaxMetaspaceSize=512m
```

上面的配置示例告诉 JVM，允许 Metaspace 增长到 512 MB。

### 无法新建本地线程

`java.lang.OutOfMemoryError: Unable to create new native thread` 这个错误意味着：**Java 应用程序已达到其可以启动线程数的限制**。

【原因】

当发起一个线程的创建时，虚拟机会在 JVM 内存创建一个 `Thread` 对象同时创建一个操作系统线程，而这个系统线程的内存用的不是 JVM 内存，而是系统中剩下的内存。

那么，究竟能创建多少线程呢？这里有一个公式：

```text
线程数 = (MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize)
```

【参数】

- **`MaxProcessMemory` - 一个进程的最大内存**
- **`JVMMemory` - JVM 内存**
- **`ReservedOsMemory` - 保留的操作系统内存**
- **`ThreadStackSize` - 线程栈的大小**

​	**给 JVM 分配的内存越多，那么能用来创建系统线程的内存就会越少，越容易发生 `unable to create new native thread`**。所以，JVM 内存不是分配的越大越好。

但是，通常导致 `java.lang.OutOfMemoryError` 的情况：无法创建新的本机线程需要经历以下阶段：

1. JVM 内部运行的应用程序请求新的 Java 线程
2. JVM 本机代码代理为操作系统创建新本机线程的请求
3. 操作系统尝试创建一个新的本机线程，该线程需要将内存分配给该线程
4. 操作系统将拒绝本机内存分配，原因是 32 位 Java 进程大小已耗尽其内存地址空间（例如，已达到（2-4）GB 进程大小限制）或操作系统的虚拟内存已完全耗尽
5. 引发 `java.lang.OutOfMemoryError: Unable to create new native thread` 错误。

【处理】

​	通常，`OutOfMemoryError` 对新的本机线程的限制表示编程错误。当应用程序产生数千个线程时，很可能出了一些问题—很少有应用程序可以从如此大量的线程中受益。

## 排查GC问题

### GC 问题常见表现

GC 问题通常表现为以下现象：

- **频繁 Full GC** ：老年代对象不断晋升，触发 Full GC，导致应用暂停。
- **GC 时间过长** ：单次 GC 耗时超过预期（如 >1s），影响响应时间。
- **内存抖动** ：Eden 区频繁分配和回收对象，Young GC 频繁。
- **STW（Stop-The-World）时间过长** ：GC 导致应用暂停，影响用户体验。

### 排查 GC 问题的详细步骤

**1）收集 GC 日志**

GC 日志是分析 GC 问题的核心依据。通过以下 JVM 参数启用 GC 日志：

```bash
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/to/gc.log
```

**日志示例** ：

```bash
[GC (Allocation Failure)  123456K->12345K(512000K), 0.1234567 secs]
[Full GC (System.gc())  123456K->12345K(512000K), 1.2345678 secs]
#GC 类型 ：Young GC（YGC）、Full GC（FGC）。
#内存变化 ：123456K->12345K(512000K) 表示 GC 前内存 → GC 后内存 / 总堆内存。
#耗时 ：0.1234567 secs 表示 GC 持续时间。
```

**2）使用 `jstat` 实时监控 GC 状态**

`jstat` 是 JDK 自带的命令行工具，用于实时查看 GC 统计信息。

```bash
jstat -gc <PID> 1000 10
# <PID>：目标 Java 进程的 PID（通过 jps 获取）。
# 1000：每秒刷新一次。
# 10：共刷新 10 次。
```

**输出示例** ：

- **S0C/S1C** ：Survivor 区容量（KB）。
- **S0U/S1U** ：Survivor 区已使用（KB）。
- **EC/EU** ：Eden 区容量/已使用（KB）。
- **OC/OU** ：老年代容量/已使用（KB）。
- **YGC/YGCT** ：Young GC 次数和总耗时（秒）。
- **FGC/FGCT** ：Full GC 次数和总耗时（秒）。
- **GCT** ：GC 总耗时（秒）。

```bash
S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
4096.0 4096.0  0.0    4096.0 32768.0 16384.0   131072.0   65536.0   51200  25600  5120   2560    100    12.345   10    12.345   24.690
```

**3）分析 GC 日志**

**使用 `GCViewer`**

- **功能** ：本地可视化工具，支持上传 GC 日志文件。
- **分析重点** 
  - **GC 频率** ：查看 GC 次数是否过高（如每秒多次 YGC）。
  - **GC 时间** ：单次 GC 是否耗时过长（如 >1s）。
  - **内存分配趋势** ：Eden 区是否频繁满溢，老年代是否持续增长。

 **4）使用 `jmap` 生成堆快照**

当发现频繁 Full GC 或内存泄漏时，生成堆快照进一步分析。

```bash
jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>
```

**参数说明** ：

- `live`：仅导出存活对象（触发 Full GC）。
- `format=b`：生成二进制格式的 `.hprof` 文件。
- `<PID>`：目标 Java 进程的 PID。

**分析工具** ：

- **jvisualvm** ：JDK 自带，支持查看类实例数、内存占用。

**5）使用 `jstack` 分析线程状态**

如果 GC 停顿时间过长，检查是否存在线程阻塞或死锁。

```bash
jstack <PID> > /tmp/thread_dump.log
```

**分析重点** ：

- **线程状态** ：查找 `BLOCKED`、`WAITING`、`TIMED_WAITING` 状态的线程。
- **死锁检测** ：工具会提示是否存在死锁（如 `Deadlock` 关键字）。

### 常见 GC 问题及解决方案

**1. 频繁 Young GC**

- **表现** ：`YGC` 次数高，`YGCT` 高。
- **原因** 
  - Eden 区过小，频繁分配临时对象。
  - 应用存在大量短生命周期对象（如字符串拼接、集合创建）。
- **解决方案** 
  - 增大 Eden 区（`-XX:NewSize`、`-XX:MaxNewSize`）。
  - 优化代码减少临时对象创建（如复用对象、使用池化技术）。

**2.频繁 Full GC**

- **表现** ：`FGC` 次数高，`FGCT` 高。
- **原因** 
  - 老年代内存不足，对象频繁晋升。
  - 内存泄漏导致老年代对象无法回收。
- **解决方案** 
  - 调整堆内存参数（`-Xmx`、`-Xms`）。
  - 分析堆快照，修复内存泄漏（如静态引用、缓存未清理）。
  - 避免频繁调用 `System.gc()`（可通过 `-XX:+DisableExplicitGC` 禁用）。

**3.GC 停顿时间过长**

- **表现** ：单次 GC 耗时 >1s。
- **原因** 
  - 堆内存过大（如 >10GB），GC 算法不适合。
  - 大对象频繁分配（如大数组、大集合）。
- **解决方案** 
  - 切换为低延迟 GC（如 G1、ZGC）。
  - 分片处理大对象（如拆分集合）。
  - 调整 GC 参数（如 G1 的 `-XX:MaxGCPauseMillis`）。

![image-20250514150555467](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514150555467.png?token=ARFL34CAXA55GXJGDM47DOTI563MK)



## Java故障诊断

### 故障定位思路

- **CPU 相关问题，可以使用 <font color = '#8D0101'>top、vmstat、pidstat、ps </font>等工具排查；**
- **内存相关问题，可以使用 <font color = '#8D0101'>free、top、ps、vmstat、cachestat、sar</font> 等工具排查；**
- **IO 相关问题，可以使用 <font color = '#8D0101'>lsof、iostat、pidstat、sar、iotop、df、du </font>等工具排查；**
- **网络相关问题，可以使用 <font color = '#8D0101'>ifconfig、ip、nslookup、dig、ping、tcpdump、iptables</font> 等工具排查**

【**一般来说，服务器故障诊断的整体思路如下**】

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200309181645.png?token=ARFL34GZGC7TNRPKYXOH5W3I563MK" alt="img" style="zoom: 50%;" /><img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/20200309181831.png" alt="img" style="zoom: 67%;" />

### CPU问题

- **CPU 使用率过高**：往往是由于程序逻辑问题导致的。常见导致 CPU 飙升的问题场景如：死循环，无限递归、频繁 GC、线程上下文切换过多。
- **CPU 始终升不上去**：往往是由于程序中存在大量 IO 操作并且时间很长（数据库读写、日志等）。

#### 查找 CPU 占用率较高的进程、线程

线上环境的 Java 应用可能有多个进程、线程，所以，要先找到 CPU 占用率较高的进程、线程。

（1）使用 `ps` 命令查看 xxx 应用的进程 ID（PID）

- 也可以使用 `jps` 命令来查看。

```shell
ps -ef | grep xxx
```

（2）如果应用有多个进程，可以用 `top` 命令查看哪个占用 CPU 较高。

（3）用 `top -Hp pid` 来找到 CPU 使用率比较高的一些线程。

（4）将占用 CPU 最高的 PID 转换为 16 进制，使用 `printf '%x\n' pid` 得到 `nid`

- **<font color = '#8D0101'>PID是进程的唯一标识，而NID是线程在操作系统中的唯一标识</font>**

（5）**使用 `jstack <pid> | grep '<nid>' -C5` 命令**，查看线程堆栈信息：

```bash
$ jstack 7129 | grep '0x1c23' -C5
        at java.lang.Object.wait(Object.java:502)
        at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
        - locked <0x00000000b5383ff0> (a java.lang.ref.Reference$Lock)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"main" #1 prio=5 os_prio=0 tid=0x00007f4df400a800 nid=0x1c23 in Object.wait() [0x00007f4dfdec8000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000b5384018> (a org.apache.felix.framework.util.ThreadGate)
        at org.apache.felix.framework.util.ThreadGate.await(ThreadGate.java:79)
        - locked <0x00000000b5384018> (a org.apache.felix.framework.util.ThreadGate)

```

（6）更常见的操作是用 `jstack` 生成堆栈快照，然后基于快照文件进行分析。生成快照命令：

```shell
jstack -F -l pid >> threaddump.log
```

（7）分析堆栈信息

一般来说，状态为 `WAITING`、`TIMED_WAITING` 、`BLOCKED` 的线程更可能出现问题。可以执行以下命令查看线程状态统计：

```bash
cat threaddump.log | grep "java.lang.Thread.State" | sort -nr | uniq -c
```

如果存在大量 `WAITING`、`TIMED_WAITING` 、`BLOCKED` ，那么多半是有问题。

#### 是否存在频繁 GC

如果应用频繁 GC，也可能导致 CPU 飙升。**为何频繁 GC 可以使用 `jstat` 来分析问题**（分析和解决频繁 GC 问题，在后续讲解）。

那么，如何判断 Java 进程 GC 是否频繁？

**可以使用 `jstat -gc <pid> 1000` 命令来观察 GC 状态**。

```bash
$ jstat -gc 29527 200 5
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
22528.0 22016.0  0.0   21388.2 4106752.0 921244.7 5592576.0  2086826.5  110716.0 103441.1 12416.0 11167.7   3189   90.057  10      2.140   92.197
22528.0 22016.0  0.0   21388.2 4106752.0 921244.7 5592576.0  2086826.5  110716.0 103441.1 12416.0 11167.7   3189   90.057  10      2.140   92.197
22528.0 22016.0  0.0   21388.2 4106752.0 921244.7 5592576.0  2086826.5  110716.0 103441.1 12416.0 11167.7   3189   90.057  10      2.140   92.197
22528.0 22016.0  0.0   21388.2 4106752.0 921244.7 5592576.0  2086826.5  110716.0 103441.1 12416.0 11167.7   3189   90.057  10      2.140   92.197
22528.0 22016.0  0.0   21388.2 4106752.0 921244.7 5592576.0  2086826.5  110716.0 103441.1 12416.0 11167.7   3189   90.057  10      2.140   92.197
```

#### 是否存在频繁上下文切换

针对频繁上下文切换问题，可以**使用 `vmstat pid` 命令来进行查看**。

```shell
$ vmstat 7129
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0   6836 737532   1588 3504956    0    0     1     4    5    3  0  0 100  0  0
```

其中，`cs` 一列代表了上下文切换的次数。

**【解决方法】**

如果，线程上下文切换很频繁，可以考虑在应用中针对线程进行优化，方法有：

- **无锁并发**：多线程竞争时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的 ID 按照 Hash 取模分段，不同的线程处理不同段的数据；
- **CAS 算法**：Java 的 Atomic 包使用 CAS 算法来更新数据，而不需要加锁；
- **最少线程**：避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态；
- **使用协程**：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换；

### 内存问题

​	内存问题诊断起来相对比 CPU 麻烦一些，场景也比较多。主要包括 OOM、GC 问题和堆外内存。一般来讲，我们会先用 `free` 命令先来检查一发内存的各种情况。

诊断内存问题**，一般首先会用 `free` 命令查看一下机器的物理内存使用情况**。

```shell
$ free
              total        used        free      shared  buff/cache   available
Mem:        8011164     3767900      735364        8804     3507900     3898568
Swap:       5242876        6836     5236040
```

### 磁盘问题

#### 查看磁盘空间使用率

**可以使用 `df -hl` 命令查看磁盘空间使用率**。

```bash
$ df -hl
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        494M     0  494M   0% /dev
tmpfs           504M     0  504M   0% /dev/shm
tmpfs           504M   58M  447M  12% /run
tmpfs           504M     0  504M   0% /sys/fs/cgroup
/dev/sda2        20G  5.7G   13G  31% /
/dev/sda1       380M  142M  218M  40% /boot
tmpfs           101M     0  101M   0% /run/user/0
```

#### 查看磁盘读写性能

可以使用 `iostat` 命令查看磁盘读写性能。

```bash
iostat -d -k -x
Linux 3.10.0-327.el7.x86_64 (elk-server)        03/07/2020      _x86_64_        (4 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.14    0.01    1.63     0.42   157.56   193.02     0.00    2.52   11.43    2.48   0.60   0.10
scd0              0.00     0.00    0.00    0.00     0.00     0.00     8.00     0.00    0.27    0.27    0.00   0.27   0.00
dm-0              0.00     0.00    0.01    1.78     0.41   157.56   177.19     0.00    2.46   12.09    2.42   0.59   0.10
dm-1              0.00     0.00    0.00    0.00     0.00     0.00    16.95     0.00    1.04    1.04    0.00   1.02   0.00
```

### 网络问题

#### 无法连接

**可以通过 `ping` 命令，查看是否能连通。**

**通过 `netstat -nlp | grep <port>` 命令，查看服务端口是否在工作。**

#### TIME_WAIT 和 CLOSE_WAIT

**对比总结**

- **CLOSE_WAIT** 是代码缺陷的信号，需检查资源释放逻辑。

如果发现大量 `CLOSE_WAIT`，优先检查应用程序代码！

```bash
# 查看所有连接状态
netstat -anp | grep -E 'TIME_WAIT|CLOSE_WAIT'

# 或使用 ss 命令（更快）
ss -tanp | grep -E 'TIME-WAIT|CLOSE-WAIT'
```

| **状态**       | **角色**   | **原因**                       | **常见问题**           |
| -------------- | ---------- | ------------------------------ | ---------------------- |
| **TIME_WAIT**  | 主动关闭方 | 正常关闭流程的中间状态         | 端口耗尽（短连接场景） |
| **CLOSE_WAIT** | 被动关闭方 | 未正确关闭连接（代码逻辑问题） | 连接泄漏、资源耗尽     |

![image-20250411234839988](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250411234839988.png?token=ARFL34DWD5LWSCA3MFPJNLDI563MO)

![image-20250411234853716](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250411234853716.png)

​	TIME_WAIT 和 CLOSE_WAIT 是啥意思相信大家都知道。 在线上时，我们可以直接用命令`netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`来查看 time-wait 和 close_wait 的数量

用 ss 命令会更快`ss -ant | awk '{++S[$1]} END {for(a in S) print a, S[a]}'`

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/2019-11-04-083830.png?token=ARFL34BQIYXEGYKIINYFDLTI563MO)

##### TIME_WAIT

time_wait 的存在一是为了丢失的数据包被后面连接复用，二是为了在 2MSL 的时间范围内正常关闭连接。它的存在其实会大大减少 RST 包的出现。

过多的 time_wait 在短连接频繁的场景比较容易出现。这种情况可以在服务端做一些内核参数调优:

```java
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭
net.ipv4.tcp_tw_reuse = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭
net.ipv4.tcp_tw_recycle = 1
```

##### CLOSE_WAIT

​	close_wait 往往都是因为应用程序写的有问题，没有在 ACK 后再次发起 FIN 报文。close_wait 出现的概率甚至比 time_wait 要更高，后果也更严重。往往是由于某个地方阻塞住了，没有正常关闭连接，从而渐渐地消耗完所有的线程。

​	想要定位这类问题，最好是**通过 jstack 来分析线程堆栈来诊断问题**，具体可参考上述章节。这里仅举一个例子。

​	开发同学说应用上线后 CLOSE_WAIT 就一直增多，直到挂掉为止，jstack 后找到比较可疑的堆栈是大部分线程都卡在了`countdownlatch.await`方法，找开发同学了解后得知使用了多线程但是确没有 catch 异常，修改后发现异常仅仅是最简单的升级 sdk 后常出现的`class not found`。



# 设计模式

## 创建型模式

创建型模式的主要关注点是**“怎样创建对象？”**，它的主要特点是“将对象的创建与使用分离”。这样可以**降低系统的耦合度，使用者不需要关注对象的创建细节**。

### 工厂方法模式（Factory Method）

**<font color = '#8D0101'>定义一个用于创建对象的接口，让子类决定实例化哪个产品类对象。工厂方法使一个产品类的实例化延迟到其工厂的子类。</font>**

**问题**：

​	在软件设计中，我们经常遇到需要创建不同类型对象的情况。但是，如果直接在代码中实例化对象，会使代码紧密耦合在一起，难以维护和扩展。此外，如果对象的创建方式需要变化，那么就需要在整个代码中进行大量的修改。工厂方法模式旨在解决这个问题。

**解决方案**：

​	**工厂方法模式提供了一个创建对象的接口，但是将具体的对象创建延迟到子类中。这样，客户端代码不需要知道要创建的具体对象的类，只需要通过工厂方法来创建对象。这使得客户端代码与具体对象的创建解耦，提高了代码的灵活性和可维护性。**

​	**工厂方法模式的主要角色：**

- **抽象工厂（Abstract Factory）**：提供了创建产品的接口，调用者通过它访问具体工厂的工厂方法来创建产品。
- **具体工厂（ConcreteFactory）**：主要是实现抽象工厂中的抽象方法，完成具体产品的创建。
- **抽象产品（Product）**：定义了产品的规范，描述了产品的主要特性和功能。
- **具体产品（ConcreteProduct）**：实现了抽象产品角色所定义的接口，由具体工厂来创建，它同具体工厂之间一一对应。

**优点：**

- **松耦合**：客户端代码与具体对象的创建解耦，使得系统更具弹性和可维护性。
- **扩展性**：通过添加新的具体工厂和产品子类，可以很容易地扩展系统以支持新的对象类型。
- **封装性**：将对象的创建集中在工厂类中，封装了对象的创建细节，使得客户端代码更简洁。

**缺点：**

每增加一个产品就要增加一个具体产品类和一个对应的具体工厂类，这增加了系统的复杂度

![image-20250413110811534](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413110811534.png?token=ARFL34DFW2MBC6Y2WV2IK73I56266)

![image-20250413110825654](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413110825654.png)

### 抽象工厂模式

​	前面介绍的工厂方法模式中考虑的是一类产品的生产，如畜牧场只养动物、电视机厂只生产电视机、传智播客只培养计算机软件专业的学生等。

​	这些工厂只生产同种类产品，同种类产品称为同等级产品，也就是说：**工厂方法模式只考虑生产同等级的产品**，但是**在现实生活中许多工厂是综合型的工厂**，能生产多等级（种类） 的产品，如电器厂既生产电视机又生产洗衣机或空调，大学既有软件专业又有生物专业等。

**<font color = '#8D0101'>抽象工厂模式是工厂方法模式的升级版本，工厂方法模式只生产一个等级的产品，而抽象工厂模式可生产多个等级的产品。</font>**

**抽象工厂模式的主要角色如下：**

- **抽象工厂（Abstract Factory）**：提供了创建产品的接口，它包含**多个创建产品的方法**，可以**创建多个不同等级的产品**。
- **具体工厂（Concrete Factory）**：主要是实现抽象工厂中的多个抽象方法，完成具体产品的创建。
- **抽象产品（Product）**：定义了产品的规范，描述了产品的主要特性和功能，抽象工厂模式有多个抽象产品。
- **具体产品（ConcreteProduct）**：实现了抽象产品角色所定义的接口，由具体工厂来创建，它 同具体工厂之间是多对一的关系。

![image-20250413121158977](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413121158977.png?token=ARFL34ABPWEC6T3VWW4CMCTI56266)

![image-20250413121219845](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413121219845.png?token=ARFL34GM4NFBDBHUU2CH7NLI56264)

![image-20250413121235402](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413121235402.png?token=ARFL34AKTPEQJTLZRYZJ76TI56264)

![image-20250413121319600](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413121319600.png)

### 手写单例（singleton）设计模式

![image-20250409002955025](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409002955025-4541422.png?token=ARFL34F3KXOZNISJJMYQEALI5627C)

**单例模式的优点：**

​	由于单例模式只生成一个实例，减少了系统性能开销，当一个对象的产生需要比较多的资源时，如读取配置、产生其他依赖对象时，则可以通过在应用启动时直接产生一个单例对象，然后永久驻留内存的方式来解决

**饿汉式**：坏处:对象加载时间过长;好处:饿汉式是线程安全的。

**懒汉式**：好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![image-20250409002716615](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409002716615-4541422.png?token=ARFL34C53Q3KGKMVE6MZLCTI5627E)

#### 饿汉式

坏处:对象加载时间过长;好处:饿汉式是线程安全的。

![image-20250409002741419](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409002741419-4541422.png)

#### 懒汉式

好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/wps15-4133580-4541422.jpg)

#### 单例模式之懒汉式（双重校验锁）

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps16-4541422.jpg?token=ARFL34D425P6O6QN6CYSZHLI5627E) 

假设有两个线程a和b调用getInstance()方法，假设a先走，一路走到4这一步，执行instance = new Singleton()这句代码

这里如果变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序：

![image-20250409003110512](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409003110512-4541422.png)

​	此时，线程b刚刚进来执行到1（看上面的代码块），就有可能会看到instance不为null，然后线程b也就不会等待监视器锁，而是直接返回instance。问题是这个instance可能还没执行完构造方法（线程a此时还在4这一步），所以**线程b拿到的instance是不完整的**，**它里面的属性值可能是初始化的零值(0/false/null)，而不是线程a在构造方法中指定的值**。

#### 反射破解单例模式

以上五种单例模式的实现方式中，前四种方式都是不太安全的，饿汉、懒汉、双重检查锁、静态内部类这四种方式都可以使用反射进行破解

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/wps17-4133580-4541422.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps18-4133580-4541422.jpg?token=ARFL34AWJX3YY5CF6SLYXD3I5627G) 

源码中我们可以看见这么一句话，如果你的这个类型是枚举类型，想要通过反射去创建对象时会抛出一个异常（不能通过反射创建枚举对象）

### Spring中单例模式和设计模式中单例模式区别

**区别在于它们关联的环境不一样**：设计模式中的单例模式是指在一个JVM进程中仅有一个实例，无论在程序中何处获取该实例，始终都返回同一个对象；而Spring中的单例模式是与其容器密切相关的，如果一个JVM有多个Spring容器，即使是单例Bean，也一定会创建多个实例。

## 结构型模式

​	**结构型模式描述如何将类或对象按某种布局组成更大的结构**。它分为**类结构型模式和对象结构型模式**，前者采用继承机制来组织接口和类，后者釆用组合或聚合来组合对象。由于组合关系或聚合关系比继承关系耦合度低，满足“合成复用原则”，所以对象结构型模式比类结构型模式具有更大的灵活性。

### 代理模式

​	需要给某对象提供一个代理以控制对该对象的访问。这时，访问对象不适合或者不能直接引用目标对象，代理对象作为访问对象和目标对象之间的中介。

​	Java中的代理按照代理类生成时机不同又分为

- `静态代理`和`动态代理`。静态代理代理类在编译期就生成，而动态代理代理类则是在Java运行时动态生成。
  - 动态代理又有`JDK代理`和`CGLib代理`两种。

**【代理（Proxy）模式分为三种角色】**

- **抽象主题（Subject）类：** 通过接口或抽象类声明真实主题和代理对象实现的业务方法。
- **真实主题（Real Subject）类：** 实现了抽象主题中的具体业务，是代理对象所代表的真实对象，是最终要引用的对象。
- **代理（Proxy）类 ：** 提供了与真实主题相同的接口，其内部含有对真实主题的引用，它可以访问、控制或扩展真实主题的功能。

**【代理模式优缺点】**

**优点：**

- 代理模式在客户端与目标对象之间起到一个中介作用和保护目标对象的作用；
- 代理对象可以扩展目标对象的功能；
- 代理模式能将客户端与目标对象分离，在一定程度上降低了系统的耦合度；

**缺点：**

- 增加了系统的复杂度；

#### JDK静态代理

火车站在多个地方都有代售点，我们去代售点买票就方便很多了。这个例子其实就是典型的代理模式，火车站是目标对象，代售点是代理对象。

- 测试类直接访问的是ProxyPoint类对象，也就是说ProxyPoint作为访问对象和目标对象的中介。同时也对sell方法进行了增强（代理点收取一些服务费用）。

![image-20250413162618013](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413162618013.png?token=ARFL34AEAVKG7KZ6YZ7XMPTI5627I)

#### JDK动态代理

​	Java中提供了一个动态代理类Proxy，Proxy并不是我们上述所说的代理对象的类，而是提供了一个**创建代理对象的静态方法（newProxyInstance方法）来获取代理对象**。

**<font color = '#8D0101'>注意：ProxyFactory不是代理模式中所说的代理类，而代理类是程序在运行过程中动态的在内存中生成的类。</font>**

**执行流程如下：**

1）在测试类中通过代理对象调用sell()方法
2）根据多态的特性，执行的是代理类（$Proxy0）中的sell()方法
3）代理类（$Proxy0）中的sell()方法中又调用了`InvocationHandler`接口的子实现类对象的`invoke`方法
4）`invoke`方法通过反射执行了真实对象所属类(TrainStation)中的`sell()`方法

![image-20250413170951871](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413170951871.png)

![image-20250413170929023](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413170929023.png?token=ARFL34GAEY5VVORHBTBGDMLI5627I)

![image-20250413171006093](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413171006093.png)

#### CGLIB动态代理

​	如果没有定义SellTickets接口，只定义了TrainStation(火车站类)。很显然JDK代理是无法使用了，因为JDK动态代理要求必须定义接口，对接口进行代理。

CGLIB是一个功能强大，高性能的代码生成包。它为没有实现接口的类提供代理，为JDK的动态代理提供了很好的补充。

CGLIB是第三方提供的包，所以需要引入jar包的坐标：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>2.2.2</version>
</dependency>
```

![image-20250413174723864](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413174723864.png?token=ARFL34EB64DDZXSBRVY5OB3I5627K)

![image-20250413174734589](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413174734589.png?token=ARFL34FJ6TMOTE6ZYM3JNW3I5627M)

#### jdk代理和CGLIB代理

​	使用CGLib实现动态代理，CGLib底层采用ASM字节码生成框架，使用字节码技术生成代理类，在JDK1.6之前比使用Java反射效率要高。唯一需要注意的是，CGLib不能对声明为final的类或者方法进行代理，因为CGLib原理是动态生成被代理类的子类。

​	在JDK1.6、JDK1.7、JDK1.8逐步对JDK动态代理优化之后，在调用次数较少的情况下，JDK代理效率高于CGLib代理效率，只有当进行大量调用的时候，JDK1.6和JDK1.7比CGLib代理效率低一点，但是到JDK1.8的时候，JDK代理效率高于CGLib代理。**所以如果有接口使用JDK动态代理，如果没有接口使用CGLIB代理。**

#### 动态代理和静态代理

​	动态代理与静态代理相比较，最大的好处是接口中声明的所有方法都被转移到调用处理器一个集中的方法中处理（InvocationHandler.invoke）。这样，在接口方法数量比较多的时候，我们可以进行灵活处理，而不需要像静态代理那样每一个方法进行中转。

​	如果接口增加一个方法，静态代理模式除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度。而动态代理不会出现该问题

### 适配器模式

生活中这样的例子很多，手机充电器（将220v转换为5v的电压），读卡器等，其实就是使用到了适配器模式。

**将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类能一起工作。**

- 适配器模式分为**类适配器模式**和**对象适配器模式**，前者类之间的耦合度比后者高，且要求程序员了解现有组件库中的相关组件的内部结构，所以应用相对较少些。

**适配器模式（Adapter）包含以下主要角色：**

- **目标（Target）接口：**当前系统业务所期待的接口，它可以是抽象类或接口。
- **适配者（Adaptee）类：**它是被访问和适配的现存组件库中的组件接口。
- **适配器（Adapter）类：**它是一个转换器，通过继承或引用适配者的对象，把适配者接口转换成目标接口，让客户按目标接口的格式访问适配者。

**适配器模式应用场景**

- 以前开发的系统存在满足新系统功能需求的类，但其接口同新系统的接口不一致。
- 使用第三方提供的组件，但组件接口定义和自己要求的接口定义不同。

**eg：现有一台电脑只能读取SD卡，而要读取TF卡中的内容的话就需要使用到适配器模式。创建一个读卡器，将TF卡中的内容读取出来。**

![image-20250413175617523](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413175617523.png?token=ARFL34CYA5FCH3W5HSTZVHLI5627K)

![image-20250413175635827](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413175635827.png?token=ARFL34G4ITOVHDFZ76T4ZK3I5627M)

![image-20250413175657625](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413175657625.png?token=ARFL34BAXJATTJWH4PSEF63I5627M)

![image-20250413175710818](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413175710818.png)



## 行为型模式

### 责任链模式（呼叫中心，有接线员、经理、主管三种角色）

**责任链模式，又名职责链模式，为了避免请求发送者与多个请求处理者耦合在一起，将所有请求的处理者通过前一对象记住其下一个对象的引用而连成一条链；当有请求发生时，可将请求沿着这条链传递，直到有对象处理它为止。**

**职责链模式主要包含以下角色:**

- **抽象处理者（Handler）角色：**定义一个处理请求的接口，包含抽象处理方法和一个后继连接。
- **具体处理者（Concrete Handler）角色：**实现抽象处理者的处理方法，判断能否处理本次请求，如果可以处理请求则处理，否则将该请求转给它的后继者。
- **客户类（Client）角色：**创建处理链，并向链头的具体处理者对象提交请求，它不关心处理细节和请求的传递过程。

**优点：**

- **降低了对象之间的耦合度：**该模式降低了请求发送者和接收者的耦合度。
- **增强了系统的可扩展性：**可以根据需要增加新的请求处理类，满足开闭原则。
- **增强了给对象指派职责的灵活性：**当工作流程发生变化，可以动态地改变链内的成员或者修改它们的次序，也可动态地新增或者删除责任。
- **责任链简化了对象之间的连接：**一个对象只需保持一个指向其后继者的引用，不需保持其他所有处理者的引用，这避免了使用众多的 if 或者 if···else 语句。
- **责任分担：**每个类只需要处理自己该处理的工作，不能处理的传递给下一个对象完成，明确各类的责任范围，符合类的单一职责原则。

**题目**：假设有个呼叫中心，有接线员、经理、主管三种角色。如果接线员无法处理呼叫，就上传给经理；如果仍无法处理，则上传给主管。请用代码描述这一过程

​	设计一个**责任链模式**来实现。在责任链模式中，每个角色（接线员、经理、主管）充当处理者，每个处理者都可以选择处理问题，或者将问题传递给下一个处理者。

- **动态责任传递** ：通过 `setNextHandler` 构建处理链，符合开闭原则
- 每个处理者通过 `canHandle()` 决策是否继续传递请求（通过随机数模拟不同角色的处理成功率，体现能力差异）

![image-20250411225326371](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250411225326371-4541422.png?token=ARFL34BQSY2Q2TTBUZLX3V3I5627O)

![image-20250411225341196](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250411225341196-4541422.png?token=ARFL34HHNVK6KLPA4NQUYQDI5627Q)

![image-20250411225459145](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250411225459145-4541422.png?token=ARFL34CPCRIICOZTHIKV6XTI5627Q)

### 模板方法模式

​	定义一个操作中的算法骨架，而将算法的一些步骤延迟到子类中，使得子类可以不改变该算法结构的情况下重定义该算法的某些特定步骤。

【**模板方法（Template Method）模式包含以下主要角色：**】

**1、抽象类（Abstract Class）：**负责给出一个算法的轮廓和骨架。它由一个模板方法和若干个基本方法构成。

- **模板方法：**定义了算法的骨架，按某种顺序调用其包含的基本方法。
- **基本方法：**是实现算法各个步骤的方法，是模板方法的组成部分。基本方法又可以分为三种：
  - **1）抽象方法(Abstract Method) ：**一个抽象方法由抽象类声明、由其具体子类实现。
  - **2）具体方法(Concrete Method) ：**一个具体方法由一个抽象类或具体类声明并实现，其子类可以进行覆盖也可以直接继承。
  - **3）钩子方法(Hook Method) ：**在抽象类中已经实现，包括用于判断的逻辑方法和需要子类重写的空方法两种。

一般钩子方法是用于判断的逻辑方法，这类方法名一般为isXxx，返回值类型为boolean类型。
**2、具体子类（Concrete Class）：**实现抽象类中所定义的抽象方法和钩子方法，它们是一个顶级逻辑的组成步骤。

【**优点：**】

- 提高代码复用性：将相同部分的代码放在抽象的父类中，而将不同的代码放入不同的子类中。
- 实现了反向控制：通过一个父类调用其子类的操作，通过对子类的具体实现扩展不同的行为，实现了反向控制 ，并符合“开闭原则”。

【**缺点：**】

- 对每个不同的实现都需要定义一个子类，这会导致类的个数增加，系统更加庞大，设计也更加抽象。
- 父类中的抽象方法由子类实现，子类执行的结果会影响父类的结果，这导致一种反向的控制结构，它提高了代码阅读的难度。

**eg：炒菜的步骤是固定的，分为倒油、热油、倒蔬菜、倒调料品、翻炒等步骤。现通过模板方法模式来用代码模拟。**

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413181633986.png" alt="image-20250413181633986" style="zoom: 48%;" />

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250413181741405.png" alt="image-20250413181741405" style="zoom:40%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250413181812398.png?token=ARFL34DK4HXNEAJ3CST3K4DI5627S" alt="image-20250413181812398" style="zoom:40%;" />

### 观察者模式（发布-订阅（Publish/Subscribe）模式）

它定义了一种一对多的依赖关系，让**多个观察者对象同时监听某一个主题对象**。这个主题对象在状态变化时，会通知所有的观察者对象，使他们能够自动更新自己。

**【在观察者模式中有如下角色】**

- **Subject**：抽象主题（抽象被观察者），抽象主题角色把所有观察者对象保存在一个集合里，每个主题都可以有任意数量的观察者，抽象主题提供一个接口，可以增加和删除观察者对象。
- **ConcreteSubject：**具体主题（具体被观察者），该角色将有关状态存入具体观察者对象，在具体主题的内部状态发生改变时，给所有注册过的观察者发送通知。
- **Observer：**抽象观察者，是观察者的抽象类，它定义了一个更新接口，使得在得到主题更改通知时更新自己。
- **ConcrereObserver：**具体观察者，实现抽象观察者定义的更新接口，以便在得到主题更改通知时更新自身的状态。

**【优点】**

- 降低了目标与观察者之间的耦合关系，两者之间是抽象耦合关系。
- 被观察者发送通知，所有注册的观察者都会收到信息【可以实现广播机制】

**【缺点】**

- 如果观察者非常多的话，那么所有的观察者收到被观察者发送的通知会耗时
- 如果被观察者有循环依赖的话，那么被观察者发送通知会使观察者循环调用，会导致系统崩溃

【**观察者模式应用场景**】

- 对象间存在一对多关系，一个对象的状态发生改变会影响其他对象。
- 当一个抽象模型有两个方面，其中一个方面依赖于另一方面时。

**eg：在使用微信公众号时，大家都会有这样的体验，当你关注的公众号中有新内容更新的话，它就会推送给关注公众号的微信用户端。我们使用观察者模式来模拟这样的场景，微信用户就是观察者，微信公众号是被观察者，有多个的微信用户关注了Java潘大师这个公众号。**

```java
//1.定义抽象观察者类，里面定义一个更新的方法
public interface Observer {
    void update(String message);
}
//2.定义具体观察者类，微信用户是观察者，里面实现了更新的方法
public class WeixinUser implements Observer {
    // 微信用户名
    private String name;
    public WeixinUser(String name) {
        this.name = name;
    }
    @Override
    public void update(String message) {
        System.out.println(name + "-" + message);
    }
}
//3.定义抽象主题类，提供了attach、detach、notify三个方法
public interface Subject {
    //增加订阅者
    public void attach(Observer observer);
    //删除订阅者
    public void detach(Observer observer);
    //通知订阅者更新消息
    public void notify(String message);
}
//4.微信公众号是具体主题（具体被观察者），里面存储了订阅该公众号的微信用户，并实现了抽象主题中的方法
public class SubscriptionSubject implements Subject {
    //储存订阅公众号的微信用户
    private List<Observer> weixinUserlist = new ArrayList<Observer>();
    @Override
    public void attach(Observer observer) {
        weixinUserlist.add(observer);
    }
    @Override
    public void detach(Observer observer) {
        weixinUserlist.remove(observer);
    }
    @Override
    public void notify(String message) {
        for (Observer observer : weixinUserlist) {
            observer.update(message);
        }
    }
}
//5.客户端程序
public class Client {
    public static void main(String[] args) {
        SubscriptionSubject mSubscriptionSubject=new SubscriptionSubject();
        //创建微信用户
        WeixinUser user1=new WeixinUser("孙悟空");
        WeixinUser user2=new WeixinUser("猪悟能");
        WeixinUser user3=new WeixinUser("沙悟净");
        //订阅公众号
        mSubscriptionSubject.attach(user1);
        mSubscriptionSubject.attach(user2);
        mSubscriptionSubject.attach(user3);
        //公众号更新发出消息给订阅的微信用户（观察者对象或订阅者）
        mSubscriptionSubject.notify("Java潘老师博客官网更新了");
    }
}

```

**【JDK中提供的观察者模式实现】**

在 Java 中，通过 `java.util.Observable` 类（抽象主题类）和 `java.util.Observer` （抽象观察者类）接口定义了观察者模式，只要实现它们的子类就可以编写观察者模式实例。

【**Observable类**】

​	Observable 类是抽象目标类（被观察者），它有一个 Vector 集合成员变量，用于保存所有要通知的观察者对象，下面来介绍它最重要的 3 个方法。

- `void addObserver(Observer o)` 方法：用于将新的观察者对象添加到集合中。
- `void notifyObservers(Object arg)` 方法：调用集合中的所有观察者对象的 update方法，通知它们数据发生改变。通常越晚加入集合的观察者越先得到通知。
- `void setChange() `方法：用来设置一个 boolean 类型的内部标志，注明目标对象发生了变化。当它为true时，notifyObservers() 才会通知观察者。

**【Observer 接口】**

​	Observer 接口是抽象观察者，它监视目标对象的变化，当目标对象发生变化时，观察者得到通知，并调用 update 方法，进行相应的工作。



### 迭代器模式

提供一个对象来顺序访问聚合对象中的一系列数据，而不暴露聚合对象的内部表示。

**【迭代器模式主要包含以下角色】**

- **抽象聚合（Aggregate）角色：**定义存储、添加、删除聚合元素以及创建迭代器对象的接口。
- **具体聚合（ConcreteAggregate）角色：**实现抽象聚合类，返回一个具体迭代器的实例。
- **抽象迭代器（Iterator）角色：**定义访问和遍历聚合元素的接口，通常包含 hasNext()、next() 等方法。
- **具体迭代器（Concretelterator）角色：**实现抽象迭代器接口中所定义的方法，完成对聚合对象的遍历，记录遍历的当前位置。

**eg：迭代器模式JAVA的很多集合类中被广泛应用，接下来看看JAVA源码中是如何使用迭代器模式的。**

```java
List<String> list = new ArrayList<>();
Iterator<String> iterator = list.iterator(); //list.iterator()方法返回的肯定是Iterator接口的子实现类对象
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

- List：抽象聚合类
- ArrayList：具体的聚合类
- Iterator：抽象迭代器
- list.iterator()：返回的是实现了 Iterator 接口的具体迭代器对象

### 策略模式

​	策略模式定义了一系列[算法](https://www.panziye.com/tags/sf)，并将每个算法封装起来，使它们可以相互替换，且算法的变化不会影响使用算法的客户。策略模式属于对象行为模式，它通过对算法进行封装，把使用算法的责任和算法的实现分割开来，并委派给不同的对象对这些算法进行管理。

【**策略模式的主要角色如下**】

- **抽象策略（Strategy）类：**这是一个抽象角色，通常由一个接口或抽象类实现。此角色给出所有的具体策略类所需的接口。
- **具体策略（Concrete Strategy）类：**实现了抽象策略定义的接口，提供具体的算法实现或行为。
- **环境（Context）类：**持有一个策略类的引用，最终给客户端调用。

**【优点】**

- 策略类之间可以自由切换：由于策略类都实现同一个接口，所以使它们之间可以自由切换。
- 易于扩展：增加一个新的策略只需要添加一个具体的策略类即可，基本不需要改变原有的代码，符合“开闭原则“
- 避免使用多重条件选择语句（if else），充分体现面向对象设计思想。

**【缺点】**

- 客户端必须知道所有的策略类，并自行决定使用哪一个策略类。
- 策略模式将造成产生很多策略类，可以通过使用享元模式在一定程度上减少对象的数量。

```java
// 抽象策略类
public interface Strategy {
    void show();
}
//定义具体策略角色（Concrete Strategy）：每个节日具体的促销活动
//为春节准备的促销活动A
public class StrategyA implements Strategy {
    public void show() {
        System.out.println("买一送一");
    }
}
//为中秋准备的促销活动B
public class StrategyB implements Strategy {
    public void show() {
        System.out.println("满200元减50元");
    }
}
//定义环境角色（Context）：用于连接上下文，即把促销活动推销给客户，这里可以理解为销售员
public class SalesMan {                        
    //持有抽象策略角色的引用                              
    private Strategy strategy;                                                   
    public SalesMan(Strategy strategy) {       
        this.strategy = strategy;              
    }                                                                          
    //向客户展示促销活动                                
    public void salesManShow(){                
        strategy.show();                       
    }                                          
}       
//定义测试类，测试策略模式效果：
public class Client {
    public static void main(String[] args) {
        //春节来了，使用春节促销活动
        SalesMan salesMan = new SalesMan(new StrategyA());
        //展示促销活动
        salesMan.salesManShow();
        System.out.println("==============");
        //中秋节到了，使用中秋节的促销活动
        salesMan.setStrategy(new StrategyB());
        //展示促销活动
        salesMan.salesManShow();
    }
}

```

【**JDK源码应用策略模式解析**】

`Comparator` 中的策略模式。在`Arrays`类中有一个` sort()` 方法，如下：

```java
public class Arrays{
    public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) {
            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c);
            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);
        }
    }
}
```

Arrays就是一个环境角色类，这个sort方法可以传一个新策略让Arrays根据这个策略来进行排序。就比如下面的测试类。

```java
public class demo {
    public static void main(String[] args) {
        Integer[] data = {12, 2, 3, 2, 4, 5, 1};
        // 实现降序排序
        Arrays.sort(data, new Comparator<Integer>() {
            public int compare(Integer o1, Integer o2) {
                return o2 - o1;
            }
        });
        System.out.println(Arrays.toString(data)); //[12, 5, 4, 3, 2, 2, 1]
    }
}
```

这里我们在调用Arrays的sort方法时，第二个参数传递的是Comparator接口的子实现类对象。所以Comparator充当的是抽象策略角色，而具体的子实现类充当的是具体策略角色。环境角色类（Arrays）应该持有抽象策略的引用来调用。

# 手写代码篇

## 生产者消费者模式

在线程世界里，**生产者就是生产数据的线程，消费者就是消费数据的线程**。 

在多线程开发当中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps13.png?token=ARFL34FWKT3Z6LMTOYQLHFLI562HK) 

当缓冲区满的时候，生产者停止执行，让其他线程进行

当缓冲区空的时候，消费者停止执行，让其他线程执行

当生产者向缓冲区放入一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态；

当消费者从缓冲区取出一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态

**等待通知机制wait/notify/notifyAll**

![image-20250310233338783](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310233338783-4133580.png?token=ARFL34F5EVRJJGGGSOMZWMDI562HK)

- `wait` - `wait` 会自动**释放当前线程占有的对象锁**，并请求操作系统挂起当前线程，**让线程从 `Running` 状态转入 `Waiting` 状态**，等待 `notify` / `notifyAll` 来唤醒。如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 `notify` 或者 `notifyAll` 来唤醒挂起的线程，造成死锁。
- `notify` - 唤醒一个正在 `Waiting` 状态的线程，并让它拿到对象锁，具体唤醒哪一个线程由 JVM 控制 
- `notifyAll` - 唤醒所有正在 `Waiting` 状态的线程，接下来它们需要竞争对象锁。



- **`wait`、`notify`、`notifyAll` 都是 `Object` 类中的方法**，而非 `Thread`。
- **`wait`、`notify`、`notifyAll` 只能用在 `synchronized` 方法或者 `synchronized` 代码块中使用，否则会在运行时抛出 `IllegalMonitorStateException`**。
- **wait()，notify()，notifyAll()三个方法的调用者必须是同步代码块或同步方法中的同步监视器。**

**为什么 `wait`、`notify`、`notifyAll` 不定义在 `Thread` 中？为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？**

首先，需要了解几个基本知识点：

- 每一个 Java 对象都有一个与之对应的 **监视器（monitor）**
- 每一个监视器里面都有一个 **对象锁** 、一个 **等待队列**、一个 **同步队列**

了解了以上概念，我们回过头来理解前面两个问题。

**为什么这几个方法不定义在 `Thread` 中？**

- 由于**每个对象都拥有对象锁**，让当前线程等待某个对象锁，自然应该基于这个对象（`Object`）来操作，而非使用当前线程（`Thread`）来操作。因为当前线程可能会等待多个线程的锁，如果基于线程（`Thread`）来操作，就非常复杂了。

**为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？**

- 如果调用某个对象的 `wait` 方法，当前线程必须拥有这个对象的对象锁，因此调用 `wait` 方法必须在 `synchronized` 方法和 `synchronized` 代码块中，如果没有获取到锁前调用wait()方法，会抛出java.lang.IllegalMonitorStateException异常。

![image-20250409013043263](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409013043263.png?token=ARFL34FIFAQDUSTAKK7FLSDI562HI)

![image-20250409013058767](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409013058767.png?token=ARFL34BM4T5QHZFFM4LQFP3I562HK)

![image-20250409013131548](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409013131548.png)

```java
 public class ThreadWaitNotifyDemo02 {
 
     private static final int QUEUE_SIZE = 10;
     private static final PriorityQueue<Integer> queue = new PriorityQueue<>(QUEUE_SIZE);
 
     public static void main(String[] args) {
         new Producer("生产者A").start();
         new Producer("生产者B").start();
         new Consumer("消费者A").start();
         new Consumer("消费者B").start();
     }
 
     static class Consumer extends Thread {
         Consumer(String name) {
             super(name);
         }
         @Override
         public void run() {
             while (true) {
                 synchronized (queue) {
                     while (queue.size() == 0) {
                         try {
                             System.out.println("队列空，等待数据");
                             queue.wait();
                         } catch (InterruptedException e) {
                             e.printStackTrace();
                             queue.notifyAll();
                         }
                     }
                     queue.poll(); // 每次移走队首元素
                     queue.notifyAll();
                     try {
                         Thread.sleep(500);
                     } catch (InterruptedException e) {
                         e.printStackTrace();
                     }
                     System.out.println(Thread.currentThread().getName() + " 从队列取走一个元素，队列当前有：" + queue.size() + "个元素");
                 }
             }
         }
     }
 
     static class Producer extends Thread {
 
         Producer(String name) {
             super(name);
         }
 
         @Override
         public void run() {
             while (true) {
                 synchronized (queue) {
                     while (queue.size() == QUEUE_SIZE) {
                         try {
                             System.out.println("队列满，等待有空余空间");
                             queue.wait();
                         } catch (InterruptedException e) {
                             e.printStackTrace();
                             queue.notifyAll();
                         }
                     }
                     queue.offer(1); // 每次插入一个元素
                     queue.notifyAll();
                     try {
                         Thread.sleep(500);
                     } catch (InterruptedException e) {
                         e.printStackTrace();
                     }
                     System.out.println(Thread.currentThread().getName() + " 向队列取中插入一个元素，队列当前有：" + queue.size() + "个元素");
                 }
             }
         }
     }
 }
```

## 手写单例（singleton）设计模式

![image-20250409002955025](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409002955025.png?token=ARFL34D547RFRN5ZB5ADE2TI562HM)

**单例模式的优点：**

​	由于单例模式只生成一个实例，减少了系统性能开销，当一个对象的产生需要比较多的资源时，如读取配置、产生其他依赖对象时，则可以通过在应用启动时直接产生一个单例对象，然后永久驻留内存的方式来解决

**饿汉式**：坏处:对象加载时间过长;好处:饿汉式是线程安全的。

**懒汉式**：好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![image-20250409002716615](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409002716615.png)

### 饿汉式

坏处:对象加载时间过长;好处:饿汉式是线程安全的。

![image-20250409002741419](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409002741419.png?token=ARFL34BPBCI7L4O62QZD7CLI562HO)

### 懒汉式

好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps15-4133580.jpg?token=ARFL34GJAUTA2U5PQ5PKV7TI562HO)

### 单例模式之懒汉式（双重校验锁）

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps16.jpg?token=ARFL34HW3IMG3VB323XIOT3I562HO) 

假设有两个线程a和b调用getInstance()方法，假设a先走，一路走到4这一步，执行instance = new Singleton()这句代码

这里如果变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序：

![image-20251015225606695](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251015225606695.png?token=ARFL34ADTA47ISCOBETVSJLI563EI)

​	此时，线程b刚刚进来执行到1（看上面的代码块），就有可能会看到instance不为null，然后线程b也就不会等待监视器锁，而是直接返回instance。问题是这个instance可能还没执行完构造方法（线程a此时还在4这一步），所以**线程b拿到的instance是不完整的**，**它里面的属性值可能是初始化的零值(0/false/null)，而不是线程a在构造方法中指定的值**。

### 反射破解单例模式

以上五种单例模式的实现方式中，前四种方式都是不太安全的，饿汉、懒汉、双重检查锁、静态内部类这四种方式都可以使用反射进行破解

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/wps17-4133580.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps18-4133580.jpg?token=ARFL34CLGE4CBGALNVYHAYLI562HS) 

源码中我们可以看见这么一句话，如果你的这个类型是枚举类型，想要通过反射去创建对象时会抛出一个异常（不能通过反射创建枚举对象）

## 多个线程顺序执行问题

### 三个线程分别打印A，B，C

要求这三个线程一起运行，打印 n 次，输出形如“ABCABCABC....”的字符串

**【利用synchronized+wait()/notify()】**

​	**思路**：用对象监视器来实现，通过wait和notify()方法来实现等待、通知的逻辑，A执行后，唤醒B，B执行后唤醒C，C执行后再唤醒A，这样循环的等待、唤醒来达到目的。使用一个取模的判断逻辑C%M ==N，题为3个线程，所以可以按取模结果编号：0、1、2，他们与3取模结果仍为本身，则执行打印逻辑。

**state是volatile注意！！**

- **`Object.wait()` 方法可能因系统中断或其他原因提前返回（虚假唤醒），此时线程需要重新检查条件是否满足。**

![image-20250409003447207](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409003447207.png?token=ARFL34DDM43AXMYK6SRNJFTI562HS)

![image-20250409003515747](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409003515747.png?token=ARFL34BFOQKVG2HECUY2EEDI562HU)

![image-20250409003519564](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409003519564.png)

### 两个线程交替打印奇数和偶数

- 使用对象监视器实现

​	两个线程A、B竞争同一把锁，只要其中一个线程获取锁成功，就打印 ++i，并通知另一线程从等待集合中释放，然后自身线程加入等待集合并释放锁即可

**`private static volatile Object lock = new Object();`**

![image-20251015225639301](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251015225639301.png?token=ARFL34C6FLF672XVP42OSYDI563GK)

![image-20250409003948350](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409003948350.png)

![image-20250409003952621](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409003952621.png)

### 用两个线程，交替输出1A2B3C4D...26Z

- 跟两个线程交替打印奇数和偶数的对象监视器一样实现

![image-20250409004226701](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409004226701.png?token=ARFL34DL2GLEUCEDCORZGHTI562HY)

![image-20250409004230498](file:///Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409004230498.png?lastModify=1760538619)

![image-20250409004235464](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/场景题+项目排查问题+设计模式+手写代码+Streamis.assets/image-20250409004235464.png)

### 通过N个线程顺序循环打印从0至100

​	用等待唤醒机制，当一个线程执行完当前任务后再唤醒其他线程来执行任务，自己去休眠，避免在线程执行任务的时候其他线程处于忙等状态，浪费cpu资源。

![image-20250409012732358](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409012732358.png?token=ARFL34BDGFNULOVQ6JYHZK3I562H2)

![image-20250409012740339](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409012740339.png?token=ARFL34BXRQ6UP4SKYDOZ43LI562H4)

![image-20250409012745730](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409012745730.png?token=ARFL34CUJD33THZZES4LGFTI562HY)















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

| 环境             | 连接方式                                     | 安装目录                                                     | 访问方式                             | job发布接口机地址 |
| ---------------- | -------------------------------------------- | ------------------------------------------------------------ | ------------------------------------ | ----------------- |
| DEV开发环境      | 前端：10.107.98.242<br />后端：10.107.98.242 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://dev.streamis.bdap.weoa.com/   | --                |
| SIT测试环境      | 前端：10.107.119.211<br />后端：10.107.97.36 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://sit.dss.bdp.weoa.com/         | --                |
| BDAP-UAT测试环境 | 前端：10.107.119.46<br />后端：10.107.119.46 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://10.107.119.46:8088/           | --                |
| UAT流式主集群    | 前端：172.21.8.221<br />后端：172.21.8.221   | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://uat.streamis.bdp.weoa.com/    | 172.21.8.221      |
| UAT流式备集群    | 前端：10.108.193.83<br />后端：10.108.193.83 | 前端：/appcom/Install/streamis/streamis-web/dist<br />后端： /appcom/Install/streamis-server | http://uat.streamisbak.bdp.weoa.com/ | 10.108.193.83     |

**推荐使用UAT流式主、备流式集群，主集群广州机房，备集群天津机房，可以用于验证双活或主备高可用应用发布**

###  生产集群

####  STREAMIS东莞流式集群 

| 对应Managis集群名 | dss入口             | 机器                        | IP          | 接口机地址   |
| ----------------- | ------------------- | --------------------------- | ----------- | ------------ |
| STM-DG-FLOW-PRD   | 10.134.244.132:8088 | xg.bdz.bdpstms040003.webank | 10.130.1.32 | 10.134.224.5 |
| STM-DG-FLOW-PRD   | 10.134.244.132:8088 | xg.bdz.bdpstms040004.webank | 10.130.1.96 | 10.134.232.5 |

####  STREAMIS龙岗流式集群 

| 对应Managis集群名 | dss 入口           | 机器                        | IP          | 接口机地址     |
| ----------------- | ------------------ | --------------------------- | ----------- | -------------- |
| STM-LG-FLOW-DR    | 10.170.16.167:8088 | lg.bdz.bdpstms050003.webank | 10.173.1.86 | 10.170.164.162 |
| STM-LG-FLOW-DR    | 10.170.16.167:8088 | lg.bdz.bdpstms050004.webank | 10.173.2.25 | 10.170.16.139  |

####  STREAMIS BDAP流式数仓集群 

| 对应Managis集群名              | dss入口             | 机器                           | IP             |
| ------------------------------ | ------------------- | ------------------------------ | -------------- |
| LINKIS-DG-BDAP_LHFX-PRD-MAIN-2 | dss.lhfx.webank.com | xg.bdz.bdaplinkis180009.webank | 10.135.20.8    |
| LINKIS-DG-BDAP_LHFX-PRD-MAIN-2 | dss.lhfx.webank.com | xg.bdz.bdaplinkis180009.webank | 10.134.248.143 |

####  STREAMIS 容灾集群 

| 对应Managis集群名 | dss入口          | 机器                     | IP           |
| ----------------- | ---------------- | ------------------------ | ------------ |
| STM-SH-FLOW-DR    | 10.142.56.8:8088 | sh.bdz.stms110001.webank | 10.142.1.229 |
| STM-SH-FLOW-DR    | 10.142.56.8:8088 | sh.bdz.stms110002.webank | 10.142.1.230 |

####  STREAMIS 准生产集群 

| 对应Managis集群名 | dss入口          | 机器                        | IP           |
| ----------------- | ---------------- | --------------------------- | ------------ |
| STM-SH-FLOW-ZSC   | 10.146.4.11:8088 | sh.bdz.bdpstms060001.webank | 10.146.8.13  |
| STM-SH-FLOW-ZSC   | 10.146.4.11:8088 | sh.bdz.bdpstms060002.webank | 10.146.12.11 |



## 故障

​	这个需要麻烦紧急排期处理一下吧，我扛不住了，昨天3次告警，今天一次告警，每次都是  critical，而且这个不光是把磁盘打满，还会把io 打满，我看还有一个mv flink 作业的日志的动作

​	错误码匹配模块在yarn flink失败后，会自动拉取yarn日志分析，生产上的yarn日志一般比较大，这里就拉了54GB下来，占机器磁盘20% ，这里要优化下，避免磁盘拉爆

**问题：**
 问题1：近几天出现几次flink集群linkis ecm节点/data盘被flink日志打满的情况，伴随一次ecm oom的问题。原因是运行多天的flink任务因某些原因挂掉，会触发streamis错误码匹配诊断，将flink很大的聚合日志拉取到本地，并且一次读取多行，可能导致本地磁盘写满（流式集群linkis节点上的虚拟机，最大空间1T），或者ecm oom。

**解决：**

       1. streamis添加flink yarn聚合日志拉取开关功能，生产关闭所有场景flink yarn聚合日志的拉取   已完成

## 问题排查

### Streamis集群不可用的问题

Streamis集群服务都会设置有告警，统一使用zabbix进行告警，主要分两个部分：

- **Streamis进程服务存活性告警**：包括了stms.process.count进程数量检测，通过服务TCP端口${STMS-PORT}进行检查；

  - 如果告警产生了，运维可以直接参考我们给的SOP对Streamis服务进行重拉；

- **GC告警**：==**十分钟内full gc次数大于10次、old区占用大于99%**==

  - 查看进程的大对象：使用jps查看进程的pid，然后查看进程的对象信息：`jmap -histo $pid`

    <img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250414095417659.png" alt="image-20250414095417659" style="zoom:30%;" />

  - 查看服务进程的堆栈信息，判断线程有无block或者死锁:`jstack $pid`

    <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250414095454090.png" alt="image-20250414095454090" style="zoom:40%;" />

  - 查看垃圾回收相关信息：gc次数、gc时间等。`jstat -gcutil $pid`

    ![image-20250414095550868](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250414095550868.png)

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
  - 或者：![image-20250514151733172](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试补充笔记.assets/image-20250514151733172.png)，定位到了虚拟机的日志的输出目录，用简单的命令寻找到堆栈文件：![image-20250514151755128](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151755128.png)
  - ![image-20250514151810247](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514151810247.png)

- **容器环境限制** ：因测试环境部署在容器中，需通过 **容器 → 跳板机 → SFTP 服务器 → 本地开发机** 的链路下载堆栈文件

**（2）堆栈文件分析**

![image-20250514151936510](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250514151936510.png)

​	通过 jvisualvm.exe 应用 文件 -> 装入 的方式导入刚才下载的文件，载入之后会有这样的选项卡

- 在概要的地方就给出了出问题的关键线程：发现 OOM 原因为 **堆内存溢出（Java heap space）** ，将堆栈信息点进去，发现堆栈是由于字符串的复制操作导致发生了 oom。
- 同时通过堆栈信息可以定位到 UnitTestContextHolder.java 类
- ![image-20250514152614559](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514152614559.png)

**【类分析】**

​	通过类分析视图，我们可以简单的定位到占据内存最高的类。**这里可以按照 实例数 或者 内存大小进行排序**，查询是否有某个类的实例数量异常，比如远远的高于其他的类，通常这些异常值就是可能发生内存泄露的地方。

- 我们这里由于 char 类实例数比较高，所有我优先往内存泄漏方向思考。

![image-20250514152522535](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250514152522535.png)

**【实例数分析】**

![image-20250514152658794](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514152658794.png)

发现全是这种字符的实例，选中【实例】，右键可以分析【最近的垃圾回收节点】

这里会显示这些对象为何没有被回收，一般是静态变量或者是全局对象引用导致。

![image-20250514153311959](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153311959.png)

通过上面图片可以看出，图中主要是由静态字段导致

**（3）代码分析**

​	代码是发生在项目中引入的企同的试点单元测试组件，刚好我们项目参与了试点，由于是单元测试组件，只在编译或者测试的依赖包中引入，正式的未引入

![image-20250514153410531](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250514153410531.png)

![image-20250514153435758](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153435758.png)

这里使用到了 ThreadLocal 做一些对象的存储，但是我发现 调用 setAttribute(String key,Object value)的地方有 33 处

![image-20250514153502559](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250514153502559.png)

但是调用 clear()的地方只有3处。正常来说凡是使用 ThreadLocal 的地方，set 和 clear() 都应该成对出现。

![image-20250514153516478](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250514153516478.png)

**<u>（4）原因分析</u>**

==ThreadLocal 内存泄漏== 

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

【**通信协议与实现**】
	Linkis基于**Feign** （是一个http请求调用的轻量级框架，一种声明式REST客户端）实现了一套自定义的RPC通信方案。该方案允许微服务通过SDK以双向RPC的方式发起请求或响应请求，支持服务间的高效通信。具体来说：

- **Sender（发送器）** ：作为客户端接口，负责向其他服务发送请求。
- **Receiver（接收器）** ：作为服务端接口，负责接收并处理请求。
- **Interceptor（拦截器）** ：Sender发送器会将使用者的请求传递给拦截器。拦截器拦截请求，对请求做额外的功能性处理。
  - 分别是广播拦截器用于对请求广播操作
  - 重试拦截器用于对失败请求重试处理
  - 缓存拦截器用于简单不变的请求读取缓存处理
  - 和提供默认实现的默认拦截器

> [!CAUTION]
>
> **==Linkis RPC模块==**
>
> ​	基于Feign的微服务之间HTTP接口调用，**开发者可以像调用本地方法一样发起HTTP请求**，**只能满足简单的A微服务实例根据简单的规则随机访问B微服务之中的某个服务实例，而这个B微服务实例如果想异步回传信息给调用方，是根本无法实现的**。 同时，由于Feign只支持简单的服务选取规则，无法做到将请求转发给指定的微服务实例，无法做到将一个请求广播给接收方微服务的所有实例。 **Linkis RPC是基于Spring Cloud + Feign实现的一套微服务间的异步请求和广播通信服务**，可以独立于Linkis而使用。
>
> - Linkis RPC作为底层的通信方案，将提供SDK集成到有需要的微服务之中。 一个微服务既可以作为请求调用方，也可以作为请求接收方。
> - 作为请求调用方时，将通过Sender请求目标接收方微服务的Receiver，作为请求接收方时，将提供Receiver用来处理请求接收方Sender发送过来的请求，以便完成同步响应或异步响应。
>
> ![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/RPC-01-140fa51c0efdc3cd71c9fb47dcafd4b1.png)
>
> 【**Linkis如何通过Feign实现RPC**】
>
> 1. **接口定义与代理** ：
>    Linkis通过Feign的`@FeignClient`定义服务接口，生成动态代理对象，将方法调用转换为HTTP请求
> 2. **服务注册与发现** ：
>    服务启动时向Eureka注册，调用方通过服务名（如`linkis-engineconn-service`）动态发现目标服务地址
> 3. **双向通信封装** ：
>    Linkis在Feign基础上增加了`Sender`和`Receiver`体系，允许服务主动接收并处理请求，形成双向交互能力
> 4. **拦截器与治理** ：
>    Linkis在Feign的调用链中插入拦截器，实现权限校验、日志记录等治理功能
>
> 【 **Linkis RPC与Feign的核心区别**】
>
> | **特性**     | **原生Feign**                    | **Linkis RPC**                           |
> | ------------ | -------------------------------- | ---------------------------------------- |
> | **通信协议** | 纯HTTP协议（RESTful API）        | 基于HTTP的封装，但对外暴露为RPC接口      |
> | **服务角色** | 单向调用（客户端→服务端）        | 双向调用（服务可同时作为调用方和提供方） |
> | **扩展功能** | 依赖Spring Cloud生态（如Ribbon） | 内置拦截器、状态管理等治理功能           |



## Linkis Flink 引擎使用

### 引擎验证

在执行 `Flink` 任务之前，检查下执行用户的这些环境变量。具体方式是

```text
sudo su - ${username}
echo ${JAVA_HOME}
echo ${FLINK_HOME
```

| 环境变量名      | 环境变量内容   | 备注                                                         |
| --------------- | -------------- | ------------------------------------------------------------ |
| JAVA_HOME       | JDK安装路径    | 必须                                                         |
| HADOOP_HOME     | Hadoop安装路径 | 必须                                                         |
| HADOOP_CONF_DIR | Hadoop配置路径 | Linkis启动Flink引擎采用的Flink on yarn的模式,所以需要yarn的支持。 |
| FLINK_HOME      | Flink安装路径  | 必须                                                         |
| FLINK_CONF_DIR  | Flink配置路径  | 必须,如 ${FLINK_HOME}/conf                                   |
| FLINK_LIB_DIR   | Flink包路径    | 必须,${FLINK_HOME}/lib                                       |

### Flink引擎的使用

`Linkis` 的 `Flink` 引擎是通过 `flink on yarn` 的方式进行启动的,所以需要指定用户使用的队列，如下图所示

![yarn](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/yarn-conf-395feb3695fdbf71df62544d5df21ad3.png)

#### 通过 `Linkis-cli` 提交任务（Shell方式）

```bash
sh ./bin/linkis-cli -engineType flink-1.12.2 \
-codeType sql -code "show tables"  \
-submitUser hadoop -proxyUser hadoop
```

#### 通过 `OnceEngineConn` 提交任务

​	`OnceEngineConn` 的使用方式是用于正式启动 `Flink` 的流式应用,具体的是**通过 `LinkisManagerClient` 调用 `LinkisManager` `的createEngineConn` 的接口，并将代码发给创建的 `Flink` 引擎，然后 `Flink` 引擎就开始执行，此方式可以被其他系统进行调用，比如 `Streamis`** 。 `Client` 的使用方式也很简单，首先新建一个 `maven` 项目，或者在您的项目中引入以下的依赖。

```xml
<dependency>
    <groupId>org.apache.linkis</groupId>
    <artifactId>linkis-computation-client</artifactId>
    <version>${linkis.version}</version>
</dependency>
```

然后新建 `scala` 测试文件,点击执行，就完成了从一个 `binlog` 数据进行解析并插入到另一个 `mysql` 数据库的表中。但是需要注意的是，您必须要在 `maven` 项目中新建一个 `resources` 目录，放置一个 `linkis.properties` 文件，并指定 `linkis` 的 `gateway` 地址和 `api` 版本，如

```properties
wds.linkis.server.version=v1
wds.linkis.gateway.url=http://ip:9001/
```

```scala
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

```scala
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

```json
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

```text
Content-Type:application/json
Token-Code:BML-AUTH
Token-User:hadoop
```

#### 客户端使用token认证

linkis 提供的客户端认证方式都支持Token策略模式`new TokenAuthenticationStrategy()`

详细可以参考[SDK 方式](https://linkis.apache.org/zh-CN/docs/latest/user-guide/sdk-manual)

**示例**

```java
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

![image-20250409163643310](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250409163643310.png)

![image-20250409163819159](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163819159.png)

![image-20250409163900622](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163900622.png)

![image-20250409163918570](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409163918570.png)



## Steamis已有能力现状

![image-20250409164209968](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409164209968.png)

## Streamis简介



**Streamis 是流式应用运维管理系统**

​	基于 DataSphere Studio 的框架化能力，以及底层对接 Linkis 的 Flink 引擎，让用户低成本完成流式应用的开发、发布和生产管理。

**【核心特点】**

**（1）基于 DSS 和 Linkis的流式应用运维管理系统。**
       以 Flink 为底层计算引擎，基于开发中心和生产中心隔离的架构设计模式，完全隔离开发权限与发布权限，隔离开发环境与生产环境，保证业务应用的高稳定性和高安全性。

​	应用执行层集成 Linkis 计算中间件，打造金融级具备高并发、高可用、多租户隔离和资源管控等能力的流式应用管理能力。

**（2）流式应用生产中心能力。**
       支持流式作业的多版本管理、全生命周期管理、监控告警、checkpoint 和 savepoint 管理能力。

**【依赖的生态组件】**

![企业微信截图_17424615822243](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424615822243.png)

**【Streamis功能介绍】**

![企业微信截图_17424616182701](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424616182701.png)

**【Streamis架构】** 

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_1742461728912.png" alt="企业微信截图_1742461728912" style="zoom:20%;" />

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

```json
{
	"projectName": "", // 项目名
	"jobName": "",  // 作业名
	"jobType": "flink.sql",		// 目前只支持flink.sql、flink.jar
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

![企业微信截图_17425468956360](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17425468956360.png)

![企业微信截图_17424613143934](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424613143934.png)

**【维护一个flink应用的完整生命周期】**

​	streamis维护了一个作业job的完整生命周期，包括其通过meta.json导入任务、启停、批量启停、启用禁用、查找、标签管理、快照、作业查看和配置查看编辑等功能。

- 每新上传一个作业就会用新作业覆盖旧作业，同时后面的版本号+1，用户点击版本号就能看到历史所有版本作业的信息，包括这个作业的执行历史、任务详情、依赖的jar包等信息；
- job启动前会有一个inspect界面，能够观察到job版本、快照和一致性检查结果等信息，如果有某项有问题会通过反色和文字提醒可能的异常。
- 每一次job启动都会记录到运行记录里进行查看，可以观察停止失败的原因；
- 每一次告警也会记录在告警页面里供查阅
- 这里的重点在于作业的启停其中相关资源管理的难题，这里放在痛点三里讲述。

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250321163422704.png" alt="image-20250321163422704" style="zoom:100%;" />

![企业微信截图_17424614571701](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17424614571701.png)

**【对flink作业的配置】**

​	这个flink配置基本上跟之前提到过的meta.json是一一对应的，即用户在meta.json中填写的相关参数通过配置界面进行可视化，用户可以通过配置界面对应用进行参数配置。

- 主要包括flink.resource：flink作业的资源配置，比如Parallelism并行度、JobManager和TaskManager Memory、yarn队列等
- flink.custom：flink应用的配置，包括kerberos权限keytab，log的过滤配置等；
- flink.produce：streamis对于flink作业管理等配置，包括Checkpoint开关、作业失败自动重拉机制开关、业务产品名称、高可用策略等

​	然后在设置了关键字日志回调和关键字告警之后，就能够通过ims对比如WARN、ERROR关键字进行告警。除了应用的配置，还能够收集应用的指标和执行日志提供给用户。

### 痛点三：资源管理难

#### ==ec介绍、一个ec占用多少资源、释放了什么资源==

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

​	==**flink作业启动的时候还是复用之前的flink ec，只是在flink应用启动后就会直接释放资源，然后在启动的时候会起一个特殊的flink ec：flink manager ec，用来管理flink应用的专属操作比如说stop，savepoint，checkpoint之类的**==

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

==类似IO多路复用的思想，通过一个ec manager维护一批flink作业==

#### flink分离式引擎设计细节

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250324174150782.png" alt="image-20250324174150782" style="zoom:30%;" />

**【整体流程】**

- **启动分离式和非分离式引擎，以及管理引擎的流程图**

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250324175601011.png" alt="image-20250324175601011" style="zoom:70%;" />

- **创建Flink Manager引擎、查询status和save详细过程**

![image-20250324175646677](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250324175646677.png)

​	status、kill操作均为通用操作，可以走ECM。save、doCheckPoint为flink专有操作，需要走flink管理引擎。

> [!IMPORTANT]
>
> **启动流程**：
>
> **streamis -->linkis manager --> ECM --> 判断是否有flink manager ec，没有则初始化一个并创建一个flink 普通ec，否则只创建一个flink 普通ec --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作 --> attach:EC常驻，detach:EC退出。** 
>
> **通用操作**：
>
> **streamis --> linkis manager --> ECM --> query status/kill等操作 --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作**
>
> **flink特有操作：**
>
> **streamis --> linkis manager -->  ECM --> flink manager ec --> savepoint/checkpoint/stop --> attach flink yarn client，通过applicationId找到对应的实例来进行相关操作**

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

![image-20250321175529316](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250321175529316.png)

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

<u>**（1）【服务启动时初始化定时任务（如 doMonitor() 的周期性执行）和清理未完成的任务】**</u>

- 启动定时器，定期调用 doMonitor() 方法。
- 清理服务重启后遗留的未完成任务。

```scala
@PostConstruct
def init(): Unit
```

**<u>1）【使用 Utils.defaultScheduler.scheduleAtFixedRate 实现周期性任务监控】</u>**

![image-20250321222103386](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321222103386.png)

**`Utils.defaultScheduler.scheduleAtFixedRate`：**创建一个固定频率的定时任务。

> [!WARNING]
>
> 实际上用的是**线程池中的Executors类的ScheduledThreadPoolExecutor**
>
> ScheduledThreadPoolExecutor使用的任务队列**DelayQueue封装了一个PriorityQueue**，PriorityQueue会对队列中的任务进行排序，**执行所需时间短的放在前面先被执行(ScheduledFutureTask的time变量小的先执行)，如果执行所需时间相同则先提交的任务将被先执行**

- 第一个参数：任务执行逻辑：Runnable 接口实现，执行doMonitor()方法。
- 第二个参数：初始延迟时间（单位：毫秒）。
  - 这里设置为 JobConf.TASK_MONITOR_INTERVAL.getValue.toLong，表示首次执行的时间间隔。
- 第三个参数：任务执行间隔时间（单位：毫秒）。
  - 同样由 JobConf.TASK_MONITOR_INTERVAL 配置决定。
- 第四个参数：时间单位（TimeUnit.MILLISECONDS）。

**将定时任务的返回值赋值给 future对象，表示该定时任务的执行状态。**

- 这个 Future 对象可以用来检查任务是否完成、取消任务等。

- 在服务关闭时，可以通过 future.cancel(true) 取消定时任务，释放资源

  ```scala
  @PreDestroy
  def close(): Unit = {
      Option(future).foreach(_.cancel(true))
  }
  ```

> [!NOTE]
>
> Future 提供了以下方法来检查任务状态：
>
> - isDone()：判断任务是否已完成。
> - isCancelled()：判断任务是否已被取消。

<u>**2）【在服务启动时清理未完成的任务（处于STARTING状态）】**</u>

![image-20250321222808369](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250321222808369.png)

通过**`wds.streamis.job.reset_on_restart.enable`配置**是否在服务启动时清理未完成的任务

- 将任务清理逻辑提交到线程池中异步执行。

```scala
Utils.defaultScheduler.submit(new Runnable {
    override def run(): Unit = Utils.tryAndError {
        ...
    }
})
```

- 创建一个状态列表 statusList，包含需要清理的任务状态（这里是 STARTING）。
- 调用 streamTaskMapper.getTasksByStatus(statusList) 获取所有符合状态的任务。
- 使用 **filter 方法筛选出属于当前服务实例的任务**（通过 thisServerInstance ：{hostname}:{port}比较）。

![image-20250321225644211](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250321225644211.png)

- 遍历所有未完成的任务，逐一清理并更新状态
  -  通过JobLaunchManage的connect() 连接 Linkis 集群得到jobClient，并调用 jobClient.stop() 停止任务。
  -  更新任务状态为 FAILED，并记录错误描述。
  -  调用 streamTaskMapper.updateTask(tmpTask) 将更新后的任务状态写入数据库

<u>**（2）【服务关闭时取消定时任务，释放资源】**</u>

```scala
@PreDestroy
def close(): Unit
```

##### 任务监控

定期扫描所有流式任务，检查其状态并执行相应操作。

```scala
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

![image-20250323223747645](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250323223747645.png)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250323223709881.png" alt="image-20250323223709881" style="zoom:45%;" />

#### 多集群高可用发布

通过指定的打包层级和差异化替换模板，实现streamis flink作业的多集群高可用发布

**【包括以下几个部分】**

- 物料包打包格式，发布前会校验；
- meta.json文件模版，对变量进行分级，发布前会校验；
- 校验发布集群与填写的高可用策略是否一致；
- 在cmdb平台上登记flink应用高可用架构以及信息，发布前会校验；
- 正式发布流程，通过高可用策略把物料发步到不同集群，带上source的jsaon字段，包括了物料包md5、发布人、一致性标志位等信息，为后续一致性启动做准备

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250323223324926.png" alt="image-20250323223324926" style="zoom:30%;" />

##### 物料包打包格式

​	校验物料包的打包层级，用户必须严格按照提供的物料包模版进行打包，否则无法通过校验流程。我们规定了相应打包格式在这里。主要分为三个部分：

- 1）自动化执行脚本，主要包括物料发布、参数校验的核心逻辑；
- 2）差异化变量，主要包括需要用到的环境变量如集群地址等；
- 3）资源文件，主要包括用户本次发布启动需要用到的物料资源包，如jar包等。

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250323223443053.png" alt="image-20250323223443053" style="zoom:100%;" />

##### meta.json作业模版定义

==通过作业模版的差异化配置来确保job的一致性==

> [!IMPORTANT]
>
> **如果每个job的meta.json都不一样，那么无法实现一个物料包同时发送多个集群**

因此我们将json文件的key进行了分类，

- 1）Job级别1：job级别相同，必须校验物料一致性，使用固定值，定义为级别①
- 2）Job级别2：job级别不相同，不必须物料校验一致性，须保证参数数量一致，使用cmdb idc变量，定义为级别②
- 3）idc级别不同的，要用cmdb icd变量，定义为级别③

> [!WARNING]
>
> Meta.json文件是StreamisJob的元数据信息，记录了该job需要用到的一系列信息：比如项目名、作业名、并行度和其他job生产参数等信息，某些参数可能会由于job不同集群不同等原因而产生变化。因此将meta.json的参数划分为不同的等级，并按照等级来确认参数的value值需要使用固定值或者差异化变量。通过将meta.json的参数划分为不同等级后，考虑到用户失误或是其他意外情况，需要额外对用户上传的meta.json文件的参数进行校验。

```json
 {
  "jobType": "flink.jar",  1
  "jobName": "0725_1",   1
  "tags": "test_0.2.5_jar_12",  2[@IDC_STREAMISJOBCONF..p1.j1..c1]
  "description": "测试普罗米修斯",   2[@IDC_STREAMISJOBCONF..p1.j1..c2]
  "jobContent": {
    "main.class.jar": "demo-stream-app-0.1.3-SNAPSHOT.jar",  1
    "main.class": "com.webank.wedatasphere.stream.apps.core.entrance.StreamAppEntrance",  1
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
		"wds.linkis.flink.jobmanager.memory":"1024",  1
		"wds.linkis.flink.taskmanager.memory":"2048",    1
		"wds.linkis.flink.taskmanager.numberOfTaskSlots":"2",    1
		"wds.linkis.flink.taskmanager.cpus":"2",    1
		"wds.linkis.rm.yarnqueue":"dws"    3[@IDC_STREAMISJOBCONF..p1.j1..c4]
	},
	"wds.linkis.flink.custom": {
        "classloader.resolve-order": "parent-first",    1
		"demo.log.tps.rate": "600",   2[@IDC_STREAMISJOBCONF..p1.j1..c5]
		"security.kerberos.krb5-conf.path": "/appcom/config/keytab/krb5.conf",   2[@IDC_STREAMISJOBCONF..p1.j1..c6]
		"security.kerberos.login.contexts": "Client,KafkaClient",      2[@IDC_STREAMISJOBCONF..p1.j1..c7]
		"security.kerberos.login.keytab": "/appcom/Install/streamis/hadoop.keytab",    2[@IDC_STREAMISJOBCONF..p1.j1..c8]
		"security.kerberos.login.principal": "hadoop/inddev010004@WEBANK.COM",   2[@IDC_STREAMISJOBCONF..p1.j1..c9]
		"stream.log.filter.keywords": "INFO,ERROR",      1
		"stream.log.filter.level-match.level": "INFO",    1
		"env.java.opts": " -DjobName=0607_1",        3[@IDC_STREAMISJOBCONF..p1.j1..c10]
	},
	"wds.linkis.flink.produce": {
		"wds.linkis.flink.checkpoint.switch":"ON",     1
		"wds.linkis.flink.alert.failure.user":"alexyang",   1
		"wds.linkis.flink.app.fail-restart.switch":"OFF",   1
		"wds.linkis.flink.app.start-auto-restore.switch":"OFF",    1
		"linkis.ec.app.manage.mode": "detach"     1
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

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326171314170.png" alt="image-20250326171314170" style="zoom:90%;" />

![image-20250326171339510](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326171339510.png)



##### 正式发布流程

​	主要过程就是通过高可用策略把物料发步到不同集群，针对不同的高可用策略的job发布策略也有调整，比如单活任务只会发布到选定集群上，双活任务则会同时发布到双集群上。

​	在将物料上传至streamis集群的同时会带上一个名为source的json字段，包括了物料包md5、发布人、一致性标志位等信息，为后续一致性启动做准备。



#### 多集群一致性启动

**主要包括<font color = '#8D0101'>物料一致性检查</font>和<font color = '#8D0101'>任务启动</font>两个部分**

​	用户可以在页面上填写两个需要启动的集群名，其中单集群策略比如single则两个集群名填写一个即可；若是多集群策略如double，则需要填写两个集群名不同且相关的集群，比如BDAP_UAT和BDAP_SIT这两个互为主备的集群

```
job-list
 |--job-list.sh
 |--joblist.txt ----（格式：proj,job ，每行只有一个job）
 |--streamis-env.sh
 |--user-env.sh
```

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250326172332530.png" alt="image-20250326172332530" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326172342828.png?token=ARFL34BE67E43UD6L43AQH3I565KG" alt="image-20250326172342828" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326171906844.png?token=ARFL34GBNB7ZR4DXMFJ3WUDI565KG" alt="image-20250326171906844" style="zoom:33%;" />

##### 一致性检查

**脚本一致性检查流程：**

- 读取当前job的高可用策略，看他是否是主备或者是双活策略，如果是的话就会进入一致性校验的主流程，否则就会跳过校验。
- 具体的校验主流程主要是通过第一次发布传入的source参数来校验一致性，检查主备集群中source字段的物料包md5和发布人是否一致，若一致则将标志位置为true并将成功信息塞入message字段中；若不一致则将标志位置为false，并将具体不一样的物料信息传入message，更新source字段的这两个key到数据库，最后汇总检查结果。

**source字段结构如下：**

```json
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

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250326172455273.png" alt="image-20250326172455273" style="zoom:80%;" />

##### 启动任务

​	一致性检查后则开始对主备集群的job发起启动命令，通过**后台服务代码**读取source字段中的一致性标志位和来源，从而判断job是否能够启动，判断一致性通过后才会最终启动job，否则最终无法启动job。

- 首先判断高可用执行开关 是否开启，若开启则会进行下一步；
- 判断双活策略key（wds.streamis.app.highavailable.policy）是否为"双活、主备"，是double则会继续校验一致性；否则会直接检查通过并提示当前任务是单活
- 继续判断source中的source字段是否为aomp，若来自aomp则会继续校验一致性；否则会直接检查通过并提示当前任务不是来自aomp
- 最后判断isHighAvailable参数是否为true，为true则允许job启动，否则不能启动。

**通过最终校验后，开始启动任务：**

- **若发布时的"wds.streamis.app.highavailable.policy"参数为double，则启动时会同时启动主备两个集群的job；若参数为single，则启动时会启动主集群的job，主集群为启动时设定的地址（可在配置差异化变量方案的步骤查看）；若为managerSlave则也只会启动主集群的job。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326173309369.png?token=ARFL34DHJTL2L2G5S2I3AU3I565KI" alt="image-20250326173309369" style="zoom: 35%;" />

#### Streamis双活应用的数据一致性问题

​	**==流式应用方需确保幂等问题，即就算作业重启，不会存在数据重复计算、漏算等问题==**

​	**<font color = '#8D0101'>BDP集群每天会定时进行一次数据备份，数据时效为T-1。Hadoop平台无法做到数据实时同步，只能通过定时同步的方式对数据进行备份，对于Flink实时处理的数据如有更高的数据时效需求，应自行设计外部存储的数据备份方案。</font>**

​	**<font color = '#8D0101'>双活通常指两个数据中心同时运行，互为备份。当某个数据中心故障时，流量会切换到另一个数据中心，由存活的应用继续处理数据。</font>**若数据同步是双向的，每个机房可能都有全量数据，但需处理同步冲突

##### 确定数据来源

在双活应用中，两个Flink作业可能从**同一数据源（如Kafka主题）消费数据**

【**Kafka消费者组与分区分配**】

- **原理**：通过配置**两个Flink应用使用同一Kafka消费者组**，Kafka会**根据负载均衡策略将主题分区分配给不同应用的消费者**。例如，若主题有10个分区，Kafka可能分配5个分区给应用A，5个分区给应用B。
- **实现**：在Flink的KafkaSource中设置相同的消费者组ID（"group.id"），通过Kafka监控工具（如Kafka Manager）查看分区分配情况，确认哪个应用消费了哪些分区。
- **挑战**：需确保作业配置一致，避免分区分配冲突。分区倾斜可能导致性能不均，需优化分区策略。

【**元数据标记**】

- **原理**：在Flink作业的处理管道中为输出数据添加标识，如通过map函数添加“application_id”字段，标明是哪个应用处理的。

- **实现**：在代码中为每个应用配置唯一标识比如ID（通过环境变量或配置文件），在DataStream API中添加，并添加到输出。例如：

  ```java
  DataStream<Output> output = input.map(event -> new Output(event, appId));
  ```

- **优势**：下游系统可直接根据标识识别数据来源，灵活性高。

【**水印与事件时间管理**】

- **原理**：Flink使用水印（watermark）管理事件时间，两个应用可通过共享水印生成逻辑或外部时间服务，确保处理顺序一致。
- **实现**：配置相同的watermark策略（如基于事件时间的固定延迟），通过日志或监控工具验证水印进度。
- **作用**：间接验证数据来源是否符合预期时间窗口，但需结合分区分配使用。

##### 保证数据一致性

**==流式应用方需确保幂等问题，即就算作业重启，不会存在数据重复计算、漏算等问题==**

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

==**【详细设计过程】**==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250327105630356.png?token=ARFL34AU7EYEPH5RFXUARMTI565KI" alt="image-20250327105630356" style="zoom:70%;" />

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

```bash
hadoop distcp \
  -Dmapreduce.job.queuename=<queue> \
  -Dmapred.job.name=distcp_streaming_apps_checkpoint \
  -update \
  -delete \
  -pbcugp \
  hdfs://<active_nn>/flink/checkpoints \
  hdfs://<active_nn>/backup/checkpoints
```

==通过 `-update` 和 `-delete` 实现**目录级增量同步** ，但同步范围是整个Checkpoint/Savepoint目录。==



**==【注意】==**

<u>**1)checkpoint路径规范，按照streamis目录规范**</u>

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
- **<font color = '#8D0101'>因此，备集群启动的作业是从较新的chk中恢复的作业。</font>**

 

**【原过程】**

1.用户在itsm进行应用信息登记；

2.在主备集群的streamis部署机器，启动一个定时脚本，定期遍历yarn上的所有在运行的流式应用，并结合cmdb登记的应用架构信息过滤出使用热主冷备策略的流式应用；

3.根据流式应用的appId，获取到对应的流式应用jobId以及找出该流式应用的checkpoint地址，checkpoint周期以及checkpoint超时时间；

4.根据当前checkpoint目录下快照文件的更新时间过滤出可同步的快照文件，，并将满足条件的快照文件通过hdfs distcp命令同步到备集群

5.定期删除hdfs上过期的文件，以保留固定数量的快照文件。

![image-20250326233822972](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326233822972.png?token=ARFL34EAEKEJJ7SIICRJ7I3I565KM)

**【原详细设计】**

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250327002451789.png" alt="image-20250327002451789" style="zoom:30%;" />

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

==【**5. 分布式拷贝 (`distcp_snapshot`)**】==

```bash
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

==通过 `-update` 和 `-delete` 实现**目录级增量同步** ，但同步范围是整个Checkpoint/Savepoint目录。==

**输入** ：

- `active_nn`：Active NameNode地址。
- `src_path`：源HDFS路径（如 `/flink/checkpoints`）。
- `des_path`：目标HDFS路径（如 `/backup/checkpoints`）

==**核心参数**== 

- `-update`：增量同步：仅同步修改过的文件（**仅同步源路径中修改时间晚于目标路径 的文件**）。
- `-delete`：删除目标端多余的文件**（删除目标路径中源路径不存在的文件，保持与源路径一致）**
- `-pbcugp`：保留文件属性（块大小、校验和等）。

**执行流程** 

1. 调用`pre_distcp`进行预检查。
2. 构造distcp命令并执行。
3. 失败时重试2次，最终告警退出。

**意义** ：实现高效、可靠的HDFS数据增量同步。

> [!CAUTION]
>
> **全量同步风险** ： 如果主集群Checkpoint/Savepoint目录包含历史数据 （如旧版本作业的Checkpoint），这些数据也会被同步到备集群。 
>
> **解决方案 ：可通过==时间戳过滤==（如只同步最近N天的Checkpoint）。**
>
> 通过 **筛选出最近N天内修改的文件** ，仅同步这些文件到备集群。
> **核心步骤** ：
>
> 1. **获取文件列表** ：列出主集群中所有Checkpoint/Savepoint文件。
> 2. **时间过滤** ：筛选出修改时间在最近N天内的文件。（使用 `hdfs dfs -ls` 结合 `awk` 和 `find` 过滤文件）
> 3. **增量同步** ：将筛选后的文件通过 `distcp` 同步到备集群。

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

**<u>⼀，⽬标</u>**

确定流式应⽤的单集群和多集群⾼可⽤管理规范

**<u>⼆，单集群⾼可⽤规范</u>**

- ⽤户作业配置⾃动失败重拉参数如果开启，Streamis在每分钟的定时任务⾥检测到Flink应⽤状态变为结束，则会⾃动重拉应⽤，并发出告警。⽤户应评估是否需要开启⾃动失败重拉，并配置合适的告警级别。

**<u>三，多集群⾼可⽤规范</u>**

- ⽤户应⽤发布前，应该在ITSM 提单 流式应⽤登记，登记名称、⾼可⽤架构、开发属主等信息，会登记到CMDB流式应⽤信息。
- ⽤户作业⽣产配置的⾼可⽤策略，应该跟CMDB⾥登记的⾼可⽤策略⼀致。
- 推荐流式应⽤使⽤双活和双活灾的⾼可⽤策略，其它⾼可⽤策略均要申请架构例外。其中主备、主备灾、双活、双活灾都涉及⽣产双集群部署作业，主备只在主集群启动应⽤，双活需要在主备集群都启动应⽤。有应⽤部署和启动不符合⾼可⽤策略的，会发出审计邮件限期整改。

## Streamis使用手册

### 1. 前言

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本文是Streamis的快速入门文档，涵盖了Stremis的基本使用流程，更多的操作使用细节，将会在用户使用文档中提供。  


### 2. Streamis整合至DSS

为了方便用户使用，**Streamis系统以DSS组件的形式嵌入DSS系统中**

#### 		2.1 如何接入？

按照 [StreamisAppConn安装文档](../development/StreamisAppConn安装文档.md) 安装部署StreamisAppConn成功后，Streamis系统会自动嵌入DSS系统中。

####       2.2 如何验证 DSS 已经成功集成了 Streamis？

请进入 DSS 的工程首页，创建一个工程

进入到工程里面，点击左上角按钮切换到”流式生产中心“，如果出现streamis的首页，则表示 DSS 已经成功集成了 Streamis。如下图：

![image-20211230173839138](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_product_center.png?token=ARFL34BCFYKZTGVD4DEGPVLI565KK)


### 3. 核心指标 

进入到streamis首页，上半部显示的是核心指标。

核心指标显示当前用户可查看到的上传到该项目执行的Flink任务的状态汇总，状态暂时有9种，显示状态名称和处于该状态的任务数量，具体内容如下图。

![核心指标](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/home_page.png?token=ARFL34CBPYHNJC3DPUAVGN3I565KM)

### 4. 任务示例

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;主要演示案例从Script FlinkSQL开发，调试到Streamis发布的整个流程。

#### 4.1. FlinkSQL作业示例

**4.1.1. Script开发SQL**

顶部Scriptis菜单创建一个脚本文件，脚本类型选择Flink,如下图所示：

![进入FlinkSQL](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/enter_flinksql-2476903.png?token=ARFL34HZCAOJ7FOW5NO52OLI564UK)

![create_script_file.png](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/create_script_file-2476903.png)

编写FlinkSQL，source,sink,transform等。

![flinksql_script_file](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/flinksql_script_file-2476903.png)

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

![flinksql_job_use_demo](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/flinksql_job_use_demo-2476903.png)

![flinksql_job_use_demo3](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinksql_job_use_demo3-2476903.png?token=ARFL34G6R7YTCF7QIO7XS63I564UK)

将SQL文件和meta.json文件打包成一个zip文件，注意：只能打包成zip文件，其他格式如rar、7z等格式无法识别

#### 4.2. FlinkJar作业示例

##### 4.2.1.  本地开发Flink应用程序

本地开发Flink应用程序，并打包成jar包形式

![flinkJar](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar.png?token=ARFL34G3UV43GS7GZUGM24DI564UI)

##### 4.2.2.  文件打包层级

流式应用物料包是指的按照Streamis打包规范，将元数据信息(流式应用描述信息),流式应用代码，流式应用使用到的物料等内容打包成zip包。zip具体格式如下：

```
xxx.zip
    ├── meta.json
    ├── resource.txt
```

meta.json是该任务的元数据信息，resource.txt为flink应用使用到的资源文件

![flinkjar_zip](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/flinkjar_zip.png)

![flinkjar_metajson](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/flinkjar_metajson.png?token=ARFL34CXDUYPFOAIAI6DMPTI564UO)

#### 4.3. meta.json文件格式详解

meta.json是StreamisJob的元数据信息，其格式为：

```json
{
	"projectName": "", // 项目名
	"jobName": "",  // 作业名
	"jobType": "flink.sql",		// 目前只支持flink.sql、flink.jar
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

```yaml
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

 ![default_config1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/default_config1.png?token=ARFL34E32O4TBTVPWLLXPADI564UO) 

 ![default_config2](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/default_config2.png) 

#### 4.4. 作业发布至Streamis

测试环境允许直接通过页面上传发布作业，生产推荐使用基于aomp的一致性物料发布与启动方案，用户使用手册：http://docs.weoa.com/docs/vVAXVnRQ4WsQj8qm

目前生产上暂时保留直接通过页面上传发布作业，后续会逐渐切换至aomp标准发布流程

发布模板：
 ![aomp_based_release](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/aomp_based_release.png)
启动模板：
 ![aomp_based_start](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/aomp_based_start.png?token=ARFL34GJIAMNDCYVKYPGHHLI564UO)

##### 4.4.1. 上传工程资源文件

进入到DSS页面，新建工程后，选择“流式生产中心”进入，点击“工程资源文件”

![job_resource1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_resource1.png?token=ARFL34AJ5K3BYR23SVTHC33I564UQ)

点击“导入”进行资源包的导入，选择文件系统的资源包，并设置版本号

![job_resource2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_resource2.png?token=ARFL34EPWYVLL6AYERMF323I564US)

导入完成

![job_resource3](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_resource3.png?token=ARFL34DWQT5JLCP4QJUHYZTI564UU)

##### 4.4.2. 上传作业的zip包

进入到DSS页面，新建工程后，选择“流式生产中心”进入，点击“导入”，选择文件系统中打包好的zip文件进行上传

![job_import](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/job_import.png)

如果上传zip文件出现下面错误，请调整下nginx的配置`vi /etc/nginx/conf.d/streamis.conf`，添加属性`client_max_body_size`，如下图所示。
![upload_jobtask_error](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/upload_jobtask_error.png)

![upload_jobtask_error_solve](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/upload_jobtask_error_solve.png)

在streamis中将该zip包导入，导入任务后，任务的运行状态变成"未启动"，导入的新作业版本从v00001开始，最新发布时间会更新至最新时间。

### 5. 工程资源文件

Streamis首页-核心指标右上角-工程资源文件。
工程资源文件提供了上传和管理项目所需资源文件的功能，如下图所示：

![project_source_file_list](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/project_source_file_list.png)

上传项目文件

![project_source_file_import](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/project_source_file_import.png?token=ARFL34BQGTL4EY5GWSAVYO3I564UY)

### 6. Streamis作业介绍

点击”作业名称“,可查看任务的详情，包括，运行情况、执行历史、配置、任务详情、告警等。

#### 6.1 运行情况

![stream_job_detail](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_job_detail.png?token=ARFL34EZ56SYV6TL2PE737TI564UW)

#### 6.2 执行历史

打开执行历史可以查看该任务的历史运行情况，

历史日志：只有正在运行的任务才能查看历史日志。

![stream_job_history](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/stream_job_history.png)

历史日志中可以查看当前任务启动的flink引擎的日志，也可以查看当前任务启动的Yarn日志，可以根据关键字等查看关键日志，点击查看最新日志，可以查看当前引擎的最新日志。

![job_log1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/job_log1.png)

![job_log2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_log2.png?token=ARFL34FQG4GY45XI4MO3GQLI564UY)

#### 6.3 配置

给Streamis任务配置一些flink资源参数以及checkpoint的参数

![image-20211231101503678](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_job_config_1.png?token=ARFL34EITWOEUK2YE2MDUPDI564U2)

![image-20211231101503678](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_job_config_2.png?token=ARFL34BTVITNSUARRVFHTHDI564U4)

#### 6.4任务详情

任务详情根据任务类型Flink Jar 和 Flink SQL分为两种显示界面。

- **Flink Jar任务详情**

![任务详情](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/stream_job_flinkjar_jobcontent.png)

&nbsp;&nbsp;Flink Jar任务详情展示了任务Jar包的内容和参数， 同时提供下载该Jar包的功能。

- **Flink SQL任务详情**

![任务详情](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stream_job_flinksql_jobcontent.png?token=ARFL34AOVDELLP2LDJNBNSLI564U4)

&nbsp;&nbsp;Flink SQL任务详情展示了该任务的SQL语句。

#### 6.5 进入Yarn页面

正在运行的Streamis任务可以通过该按钮进入到yarn管理界面上的查看flink任务运行情况。

![image-20211231102020703](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20211231102020703.png?token=ARFL34AWN6CZ44NM5QLTJ73I564U6)

### 7. Streamis作业生命周期管理

#### 7.1. 作业启动

导入成功的任务初始状态为“未启动”，点击任务启动，启动完成后会刷新该作业在页面的状态为“运行中”

![job_start1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/job_start1.png)

![job_start2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/job_start2.png?token=ARFL34FT7THAELURLM5RZ7LI564U6)

作业启动会进行前置信息检查，会检查作业的版本信息和快照信息，当作业有检查信息需要用户确认时，则会弹出弹窗；对于批量重启的场景，可以勾选“确认所以批量作业”进行一次性确认

![inspect1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/inspect1.png?token=ARFL34DXWLSAIAYZMCNE7KTI564VA)

#### 7.2. 作业停止

点击“停止”，可以选择“直接停止”和“快照并停止”，“快照并停止”会生成快照信息，并展示出来

![job_stop1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/job_stop1.png)

#### 7.3. 保存快照功能

点击相应的作业名称、配置或左边3个竖点中（参数配置/告警配置/运行历史/运行日志）可进入job任务详情，点击 启动 可执行作业

点击作业左边的3个竖点中的“快照【savepoint】 ”可保存快照

![savepoint1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/savepoint1.png?token=ARFL34B7PU6OAULSDHIX5C3I564VA)

![savepoint2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/savepoint2.png?token=ARFL34G2PEDJFM34KIA777LI564VC)

#### 7.4. 批量操作功能

点击“批量操作”，可选中多个作业任务重启，快照重启会先生成快照再重新启动，直接重启不会生成快照

![jobbulk_operate](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/jobbulk_operate.png)

#### 7.5. 作业启用禁用功能

支持对单个作业启用/禁用或多个作业批量启用/禁用；

用户可以在页面对任务进行禁用启用：

![enable_and_disable_01](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/enable_and_disable_01.png?token=ARFL34CYVJMHB2R6BSTEOTLI564VE)

![enable_and_disable_02](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/enable_and_disable_02.png?token=ARFL34HDVIVCKIFDNUJUXWLI564VE)

可以筛选已启用、已禁用和全部job：

![enable_and_disable_03](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/enable_and_disable_03.png)

已禁用的job无法进行启动等操作：

![enable_and_disable_04](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/enable_and_disable_04.png?token=ARFL34DQVKMZKKBS4DACLYLI564VG)

### 8. Streamis版本新增特性

#### 8.1.支持修改args参数

* 1、进入任务详情页
  ![1.1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/18.png)
* 2、点击编辑
  ![1.1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/19.png)
* 3、点击确认，保存  
  注意事项：  
  1、编辑arguement超长需要小于400长度  
  2、编辑格式需要注意，下面是示例：

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

![1.1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/0.3.2.1.png?token=ARFL34CQYJKPXBJXMK5N663I564VI)

**②在meta.json 配置文件中添加参数 "resources": [log4j.properties]**

```json
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

![2.1](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/2.1.png?token=ARFL34AXLFWY5NJVDB7M42TI564VI)

#### 8.4. 标签批量修改

先点批量修改，然后选中多个任务，点 修改标签。在弹出窗口输入新标签内容，支持大小写字母、数字、逗号、下划线。

![3.1](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/3.1.png)
![3.2](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/3.2.png?token=ARFL34GEE5UQ36QQFIRCSF3I564VK)

#### 8.5. 日志回调使用说明

1、flink应用在启动会先向streamis发起注册，注册数据如下图所示 ![logs_call_back_01.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/logs_call_back_01.png?token=ARFL34GGUQJ47GNGR3IL2YLI564VK)

2、定时任务：flink应用使用 Java 中的定时任务工具来定期发送心跳消息。上图中的heartbeat_time为记录的心跳时间

3、streamis新增定时任务检测应用，streamis告警信息如下图所示 ![logs_call_back_02.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/logs_call_back_02.png?token=ARFL34AXYKB3HG65NJ2SNEDI564VQ)
 ![logs_call_back_03.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/logs_call_back_03.png?token=ARFL34GTPP5MLUNJJR4XE73I564VM)

## Steamis产品截图

**【生产中心】**

![企业微信截图_17424614571701](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424614571701.png?token=ARFL34AZGUM23YALZA2SPE3I564VO)

**【运行情况】**

![企业微信截图_17424601679599](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424601679599.png)

**【运行历史】**

![企业微信截图_17424601774034](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424601774034.png)

 **【流式应用参数配置】**

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424601941298.png" alt="企业微信截图_17424601941298" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424604422726.png?token=ARFL34EPNO7OD6BYXVW27O3I564VQ" alt="企业微信截图_17424604422726" style="zoom:67%;" />

![企业微信截图_17424604586542](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424604586542.png)

![企业微信截图_17424604676047](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424604676047.png?token=ARFL34CCMPRTT2OEJD445FLI564VS)

![企业微信截图_17424604797635](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424604797635.png)

![企业微信截图_17424605021312](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424605021312.png?token=ARFL34GYNAKIKN6NN5XUBTLI564VS)

**【任务详情】**

![企业微信截图_17424606654455](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424606654455.png?token=ARFL34HP4DWTKMT3OENR5GLI564VU)

![企业微信截图_17424606762072](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424606762072.png)

**【告警】**

![企业微信截图_17424607188533](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424607188533.png?token=ARFL34DBFPDB54NKL5OFKDDI564VW)

**【物料】**

![企业微信截图_17424613265001](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424613265001.png)

**【审计日志】**

![企业微信截图_17424613143934](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424613143934.png)

## 流式数仓任务-配置化数据集成

**<u>（1）背景</u>**

- 策略分析平台策略模型数据目前使用离线批量同步，T+1时效
- 现有的BINLOG流式数据同步方案时效小时级，存在较多的可用性问题，另外每个新需求也需要较大版本人力开发，响应慢。每个规则一个应有，无法资源复用
- 基于以上背景，希望实现一套低延时、可用性高、资源消耗低、人力成本低的配置化数据集成降本增效方案，降低数据同步需求需求的开发人力和资源小号。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250227154728743.png?token=ARFL34CZX77IGIZHTEWKL5DI564VY" alt="image-20250227154728743" style="zoom:85%;" />

- 绿色线条为数据流动方向。
- 红色框为现存链路存在问题：
  - Binlog采集大数据记录丢失
  - Binlog采集元数据丢失。元数据变更不够自动化。
  - DM数据提单，过多手动补充的内容
  - 需要定时任务，根据binlog类型，来执行相应数据操作。

**<u>（2）现有方案存在的问题</u>**

①新表要提新单。提单过程复杂，每个需要科技排期开发上线。且每个新表一个任务，资源占用大。

②小时级延迟

③数据链路长，可用性存在问题

- kafka抖动存在丢数可能

- 仅支持单活，无高可用支持

④kafka限制topic大小为1MB，超过的消息都丢弃

## application.properties配置

```properties
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

- **默认行为** ：
  Spring Boot 2.1.0+ 默认禁止覆盖（`spring.main.allow-bean-definition-overriding=false`），以避免意外覆盖导致的不可预测行为

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
  - 作用 ：**<font color = '#8D0101'>动态刷新</font>**应用的配置（如从配置中心拉取最新配置），无需重启服务

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

```xml
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

<u>**1. 应用基本信息**</u>

**`<name>STREAMIS-SERVER</name>`**

- **含义** ：应用名称，对应 `spring.application.name=streamis-server` 的配置。
- **注意** ：Eureka 默认将应用名转为大写（如 streamis-server → STREAMIS-SERVER）。
- **用途** ：服务发现时通过此名称调用（如 Ribbon 客户端负载均衡）

<u>**2. 实例详情**</u>

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144527764.png?token=ARFL34CQJPNRVG4YUPKFIGTI564V2" alt="image-20250313144527764" style="zoom:50%;" />

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

<u>**3. 端口与安全配置**</u>

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313144544987.png?token=ARFL34CHBDEUNO6CFVTEWSDI564V2" alt="image-20250313144544987" style="zoom:50%;" />

**（1）`<port enabled="true">9400</port>`**

- **含义** ：非SSL端口（HTTP）为 `9400`，与 `server.port=9400` 配置一致。
- **作用** ：服务间通信默认使用此端口

**（2）`<securePort enabled="false">443</securePort>`**

- **含义** ：SSL端口（HTTPS）为 `443`，但未启用（`enabled="false"`）。
- **启用方法** ：需配置SSL证书并设置 `server.ssl.*` 参数

<u>**4. 租约与健康检查**</u> **`<leaseInfo>`**

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250313144601667.png" alt="image-20250313144601667" style="zoom:50%;" />

**（1）续约机制**

- `renewalIntervalInSecs=30` ：客户端每30秒向Eureka Server发送一次心跳
- `durationInSecs=90` ：若Eureka Server在90秒内未收到心跳，将剔除该实例

**（2）时间戳**

- `registrationTimestamp`：实例注册时间（Unix毫秒时间戳）。
- `lastRenewalTimestamp`：最后一次心跳时间。

<u>**5. 元数据（Metadata）**</u>

<img src="/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250313144752621.png" alt="image-20250313144752621" style="zoom:50%;" />

**（1）`<metadata><test>wedatasphere</test></metadata>`**

- **含义** ：自定义元数据 `test=wedatasphere`，通过 `eureka.instance.metadata-map.test=wedatasphere` 配置。
- **用途** ：可用于灰度发布、路由策略或监控分类

**（2）`<management.port>9400</management.port>`**

- **含义** ：Actuator管理端点的端口（与主应用端口一致）。
- **作用** ：通过 `/actuator/info` 和 `/actuator/health` 提供元数据和健康状态

**<u>6. 服务发现与调用</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313145153771.png?token=ARFL34BZPZDZRRCNZH2SBMLI564V6" alt="image-20250313145153771" style="zoom:50%;" />

**（1）`<vipAddress>streamis-server</vipAddress>`**

- **含义** ：虚拟IP地址（VIP），通常与应用名一致。
- 用途 ：客户端通过VIP（如 `http://streamis-server/api/...`）调用服务，Ribbon会自动解析为具体实例

**（2）`<homePageUrl>` 和 `<statusPageUrl>`**

- **`homePageUrl`** ：应用根路径（`http://bdpujes110002:9400/`）。
- **`statusPageUrl`** ：健康状态页面（`/actuator/info`），返回自定义信息（如版本号）

<u>**7. 其他关键字段**</u>

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313145314173.png?token=ARFL34BMIQAVKAEINTMHO53I564V6" alt="image-20250313145314173" style="zoom:50%;" />

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
```

```bash
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

==**<u>（1）第三步检查服务是否已运行</u>**==

```bash
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

==**<u>（2）第四步调试与安全代理配置</u>**==

```bash
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

**<u>1）调试配置（DEBUG_PORT）</u>**

- **`export DEBUG_CMD="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=$DEBUG_PORT"`** ：
  - **内容** ：设置 Java 远程调试参数。
  - **参数详解** ：
    - `transport=dt_socket`：使用套接字传输。
    - `server=y`：作为调试服务器（等待客户端连接）。
    - `suspend=n`：启动时不暂停 JVM（若设为 `y`，则等待调试器连接后再启动）。
    - `address=$DEBUG_PORT`：指定调试端口（如 `5005`）

> [!CAUTION]
>
> **Java 调试参数`-agentlib:jdwp`** 
>
> - 启用 JDWP（Java Debug Wire Protocol），允许 IDE（如 IntelliJ、Eclipse）远程调试 JVM

<u>**2）IAST 安全代理配置**</u>

- **`export JAVA_AGENT_OPTS="-javaagent:$IAST_AGENT_JAR=app.full.name=BDP-STREAMIS,flag=iast,vul.log.level=INFO"`** ：
  - **内容** ：设置 Java 代理参数。
  - **参数详解** 
    - `-javaagent:$IAST_AGENT_JAR`：指定代理 JAR 文件。
    - `app.full.name=BDP-STREAMIS`：应用名称（用于漏洞报告）。
    - `flag=iast`：启用 IAST 模式（交互式应用安全测试）。
    - `vul.log.level=INFO`：设置漏洞日志级别为 INFO

==**<u>（3）JVM关于内存溢出的参数</u>**==

```bash
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
- **生成文件** ：**<font color = '#8D0101'>文件名为 `streamis-server-2024010112-dump.log.hprof`（JVM 会自动添加 `.hprof` 后缀）</font>**

**3）`-XX:ErrorFile=路径`**

- 当 JVM 发生致命错误（如崩溃、`OutOfMemoryError`）时，生成 错误日志文件（hs_err_pid\* .log），记录详细错误信息

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

  - ```bash
    jvisualvm # 启动 VisualVM，直接加载 `.hprof` 文件
    ```

- **JVM 日志分析器** ：解析 `hs_err` 文件，检查 Native 层问题

==**<u>（4）第五步 JVM 参数与启动命令</u>**==

```bash
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

```bash
-cp $STREAMIS_CONF_PATH:$HOME/lib/*
```

- 作用 
  - **<font color = '#8D0101'>`-cp`（或 `-classpath`）</font>** ：**指定 JVM 查找类文件（`.class`）和依赖库（`.jar`）的路径**。
- 路径解析 
  - **`$STREAMIS_CONF_PATH`** ：指向配置文件目录（如 `conf/`），包含 `application.properties` 等配置。
  - **`$HOME/lib/\*`** ：包含所有依赖的 JAR 文件（如 Spring Boot、Linkis 客户端等）。
  - ==在 **Linux/Unix 系统** 中，`:` 是 **类路径分隔符** ，用于分隔多个路径条目==，**意思是conf和lib是两个不同的目录**

**4）`com.webank.wedatasphere.streamis.server.boot.StreamisServerApplication`：主类**

- 作用 
  - Spring Boot 应用的入口类，包含 `main` 方法启动应用。
- 执行流程：
  1. 加载 `SpringApplication`，读取 `@SpringBootApplication` 注解配置。
  2. 扫描组件（如 `@Controller`、`@Service`），初始化 Spring 容器。
  3. 启动嵌入式 Web 服务器（如 Tomcat），监听 `server.port=9400`。

**5）`2>&1 > $STREAMIS_LOG_PATH/streamis.out &`：输出重定向与后台运行**

```bash
2>&1 > $STREAMIS_LOG_PATH/streamis.out &
```

- **分解说明** 
  - **`>`** ：重定向标准输出（`stdout`）到文件 `streamis.out`。
  - <font color = '#8D0101'>**`2>&1`** ：将标准错误（`stderr`）重定向到标准输出（即**合并到 `streamis.out`**）。</font>
    - 在 Linux/Unix 系统中，每个进程默认有三个标准文件描述符：==**`0`** ：标准输入（stdin）**`1`** ：标准输出（stdout）**`2`** ：标准错误（stderr）==
    - **`2>`** ：表示将文件描述符 `2`（标准错误）的输出重定向。
    - **`&1`** ：表示目标是文件描述符 `1`（标准输出）的位置。
  - ==**`&`** ：将进程放入后台运行。==

- 日志文件命名 
  - `streamis.out` 包含应用日志（如启动日志、未捕获的异常）。
  - GC 日志、堆转储等通过 `STREAMIS_JAVA_OPTS` 单独指定路径。

## Streamis的Nginx配置

```nginx
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

<u>**Linkis 网关路由规则**</u>

- Linkis 网关通过 `api` 模块的 `rest_j` 路径统一暴露 REST 接口。所有微服务（包括 Streamis）的接口均通过该前缀路由。

<u>**Streamis 接口定义**</u>

- Streamis 的 Controller 通常使用 `@RequestMapping("/streamis")` 作为基础路径，例如：

  ```java
  @RestController
  @RequestMapping("/streamis/streamJobManager")
  public class StreamJobController {
      @PostMapping("/job/upload")
      public Response uploadJob(...) { ... }
  }
  ```

<u>**调用流程验证**</u>

- **服务发现** ：Streamis 注册到 Eureka 后，Linkis 网关通过服务名（如 `STREAMIS-SERVER`）发现其地址
- **路由转发** ：网关将 `/api/rest_j/v1/streamis/` 的请求转发到 Streamis 服务
- **接口执行** ：Streamis 处理请求并返回结果（如作业上传状态）

## Streamis的日志配置

### Streamis日志规范

【**约定**】Streamis 选择引用 Linkis Commons 通用模块，其中已包含了日志框架，主要**以 slf4j 和 Log4j2 作为日志打印框架**，去除了Spring-Cloud包中自带的logback。
由于Slf4j会随机选择一个日志框架进行绑定，所以以后在引入新maven包的时候，需要将诸如slf4j-log4j等桥接包exclude掉，不然日志打印会出现问题。但是如果新引入的maven包依赖log4j等包，不要进行exclude，不然代码运行可能会报错。

【**配置**】log4j2的配置文件默认为 log4j2.xml ，需要放置在 classpath 中。如果需要和 springcloud 结合，可以在 application.yml 中加上 logging:config:classpath:log4j2-spring.xml (配置文件的位置)。

【**强制**】类中不可直接使用日志系统（log4j2、Log4j、Logback）中的API。

* 如果是Scala代码，强制继承Logging trait
* java采用 LoggerFactory.getLogger(getClass)。

【**强制**】严格区分日志级别。其中：

* Fatal级别的日志，在初始化的时候，就应该抛出来，并使用System.out(-1)退出。
* ERROR级别的异常为开发人员必须关注和处理的异常，不要随便用ERROR级别的。
* Warn级别是用户操作异常日志和一些方便日后排除BUG的日志。
* INFO为关键的流程日志。
* DEBUG为调式日志，非必要尽量少写。

【**强制**】要求：INFO级别的日志，每个小模块都必须有，关键的流程、跨模块级的调用，都至少有INFO级别的日志。守护线程清理资源等必须有WARN级别的日志。

【**强制**】异常信息应该包括两类信息：案发现场信息和异常堆栈信息。如果不处理，那么通过关键字throws往上抛出。 正例：logger.error(各类参数或者对象toString + "_" + e.getMessage(), e);

### SLF4J 的角色

**门面模式（Facade）** ：

- SLF4J 是日志接口标准，不提供具体实现，需绑定 Log4j2、Logback 等后端

**示例代码** ：

```java
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
| -------- | ------------ | --------------------------- |
| SLF4J    | 接口（门面） | 需绑定具体实现（如 Log4j2） |
| Log4j2   | 具体实现     | 可直接使用或通过 SLF4J 调用 |

**优先使用 Log4j2 （性能优势） + SLF4J 门面 （解耦代码与实现）**

```xml
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

```xml
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

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="error" monitorInterval="30">
```

- **`status="error"`** 
  - 设置 Log4j2 内部日志的输出级别为 `ERROR`，仅记录框架自身的错误信息（避免调试信息干扰）
- **`monitorInterval="30"`**
  - 每 30 秒自动重新加载配置文件（热更新），无需重启服务即可生效新配置

#### 输出方式定义（Appenders）

<u>【**Console（控制台输出）**】</u>

```xml
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

<u>【**RollingFile（滚动文件输出）**】</u>

```xml
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

```xml
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
| ------------------ | ------------------------------------------------------------ |
| **Console**        | 输出`INFO`及以上日志到控制台，格式包含时间、级别、线程、类名、行号等。 |
| **RollingFile**    | 按`100MB`大小滚动日志，每月归档，最多保留`20`个文件。        |
| **Root Logger**    | 全局日志级别为`INFO`，输出到文件和控制台。                   |
| **SecurityFilter** | 覆盖特定类的日志级别为`WARN`，减少冗余日志。                 |

## Streamis告警

### 用户态告警

目前STREAMIS生产环境上的用户态告警主要有两个，都属于任务状态告警，分别如下:

Flink应用失败重拉告警

- 告警样例：

![企业微信截图_17424574118687](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/企业微信截图_17424574118687.png?token=ARFL34F66HOMOV52BZAEKALI564QY)

- 告警描述：该告警表明因为一些外部因素导致了Flink流应用的失败，通常会附带自动重拉策略。
- 处理SOP：
  - 如果用户选择了自动重拉策略，则不需要平台运维介入处理，否则平台运维应起到通知业务的作用。



### 服务进程存活性告警

​	统一使用zabbix进行服务进程存活性监控，zabbix管理地址为zabbix.bdp.webank.com，创建一个流式平台监控模板，如下图：

![企业微信截图_17424576323859](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/企业微信截图_17424576323859.png)

![image-20250326202732913](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326202732913.png?token=ARFL34HHTMSW64LVNYNTAK3I564MS)

​	其中包括了stms.process.count进程数量检测，通过服务TCP端口${STMS-PORT}进行。 然后再进入生产环境managis(ops.bdp.webank.com)中，在监控管理台-监控模板上绑定监控模板和具体服务：

![stms_monitor_ops[1]](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/stms_monitor_ops%5B1%5D.png?token=ARFL34EHLWRQK4EUBRLO3WTI564MS)

 	以上的进程存活性告警一旦产生，平台运维则应该参照服务进程启停指南对进程进行恢复。

### GC告警

​	统一使用zabbix进行GC监控，zabbix管理地址为zabbix.bdp.webank.com，创建一个流式平台监控模板。

![image-20250326202754383](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326202754383.png?token=ARFL34CFVV5CVKVHXXADBJDI564MU)

==**十分钟内full gc次数大于10次、old区占用大于99%**==

- **查看进程的大对象**：使用jps查看进程的pid，然后查看进程的对象信息：

  ```bash
  jmap -histo $pid
  ```

![image-20250326202903106](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326202903106.png?token=ARFL34A4FYPABWVDM2J54Y3I564MS)

- 查看服务进程的堆栈信息，判断线程有无block或者死锁:

  ```bash
  jstack $pid
  ```

![image-20250326203036607](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326203036607.png?token=ARFL34FONX5OHQTEYDQBBZ3I564MW)

- 查看垃圾回收相关信息：gc次数、gc时间等。

  ```bash
  jstat -gcutil $pid
  ```

![image-20250326203121296](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250326203121296.png)

## Streamis日志查看

### 查看客户端引擎日志

（1）点击job名，查看详情

![image-20250326222428722](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326222428722.png?token=ARFL34BFFG6IRFJCA5VBBNDI564MW)

（2）点击执行历史页面，选择要查看的历史记录，点历史日志

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps107.jpg?token=ARFL34F43FFATFO6XY7EUI3I564MY) 

（3）默认客户端日志如下

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps108.jpg?token=ARFL34GGCB5D6AMC54QY5BDI564MY) 

如果看不到客户端日志，如下图，可尝试复制引擎实例信息

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps109.jpg?token=ARFL34CTHMLXQEEIVAHLSQDI564MY) 

（4）在运行情况页面，复制客户端引擎实例 如 gz.bdz.bdplinkisrcs05.webank:36616

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps110.jpg?token=ARFL34EQH6BCW7GTEKAU4RTI564M2) 

（5）点击顶部管理，进入管理台--资源管理--历史引擎管理，输入复制的实例，点搜索

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps111.jpg?token=ARFL34FR2JLJ6J7YUNQRXQ3I564M2) 

（6）点查看日志，可查看客户端日志

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps112.jpg)  

如果这里看不到日志，可能是日志被清理或其它原因，请联系管理员协助。

### 查看yarn应用日志（仅应用停止才能看）

在linkis集群上执行，先切换到对应用户：

```bash
sudo su - hduser03
```

通过applicationId查看对应yarn日志:

```bash
yarn logs -applicationId application_1666667549351_102687  | less
```

**或者在streamis查看**

类型选yarn日志。注意任务结束后才能查看。如果job执行时间很长，日志可能很大，要等候一段时间才能查看。

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps114.jpg)

### Yarn Flink JobHistory

**<u>1）通过yarn web ui查看</u>**

a. 打开Web UI界面

**`http://{resourcemanager}:8088`**

b. 点击RUNNING、FINISHED

RUNNING：在执行的时候可以看到作业，执行完之后只能在FINISHED里面看到。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps115.jpg?token=ARFL34EJ2FUEOGMUPXT6BH3I564NA) 

**查看YARN作业的日志**

a. 当作业跑完之后，进入FINISHED，点击作业的History，跳转到historyserver 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps116.jpg?token=ARFL34GJNM27ALZFXE6HHATI564M6) 

 类似如下:

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps117.jpg?token=ARFL34GXOOLTK5F7NM6STRTI564M6) 

**2）通过flink页面查看**

打开流式集群yarn web地址

**`http://{resourcemanager}:8088`**

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps121.jpg) 

点击application，输入applicationld，即可查询到对应appld。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps122.jpg?token=ARFL34HD2P5C4RJESPSW3GDI564NC) 

查询到之后，点击Application Master，即可进入flink页面。分别点击jobManager和TaskManager的loglist，可查看运行中日志

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps123.jpg) 

### ==Yarn Flink JobHistory 与 Yarn 的 Flink 日志的区别==

- Yarn Flink JobHistory应该指的是Flink在Yarn上运行时生成的作业历史记录，通常由Flink的JobManager生成，记录作业的执行状态、任务信息等。
- 而Yarn的Flink日志可能指的是Yarn管理的各个容器（比如TaskManager）产生的日志，这些日志包括标准输出、错误输出以及用户日志。

**【定义与来源】**

| **类别** | **Yarn Flink JobHistory**                      | **Yarn 的 Flink 日志**                                |
| -------- | ---------------------------------------------- | ----------------------------------------------------- |
| **定义** | Flink 作业的执行历史记录，由 JobManager 生成。 | Yarn 容器（如 TaskManager）的运行日志，由 Yarn 管理。 |
| **来源** | Flink 框架自身（JobManager/TaskManager）       | Yarn 容器（NodeManager）的标准输出/错误输出           |

**【内容与用途】**

| **类别** | **Yarn Flink JobHistory**                                    | **Yarn 的 Flink 日志**                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **内容** | 作业元数据（如 DAG、任务状态、Checkpoint 信息、任务执行时间等）。 | 容器运行时的标准输出（stdout）、错误日志（stderr）、用户日志（如打印的日志信息）。 |
| **用途** | 作业监控、性能分析、故障诊断（如任务失败原因）。             | 调试 TaskManager 启动失败、OOM 错误、用户代码异常（如空指针）。。 |

------

**【存储位置与生命周期】**

| **类别**     | **Yarn Flink JobHistory**                                    | **Yarn 的 Flink 日志**                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
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

2. 选择LevelMatch-日志等级过滤：

 日志等级：stream.log.filter.level-match.level

3. 选择ThresholdMatch-日志等级阈值过滤

 日志等级阈值： stream.log.filter.threshold.level

4. 选择RegexFilter-日志正则过滤：

 日志正则: stream.log.filter.regex.value

> [!WARNING]
>
> 配置好日志过滤，回调地址会自动到streamis服务器上，格式
>
> - /data/stream/log/代理用户名称/项目名称/业务产品/streamis应用名称/yarn应用名称.log
>
> 其中yarn应用名= 项目名称.streamis应用名称
>
> - 业务产品为生产配置参数：wds.linkis.flink.product，默认空值。

## Streamis集群容量评估

【 **组件依赖关系**】

- **DSS** ：固定双节点部署，不参与扩容（负责工作流调度）
- **Linkis** ：固定4节点部署，采用分离式提交模式 （提交任务到 YARN 后释放本地资源），无需扩容
- **Streamis** ：根据流式应用并发数动态扩容，是集群资源评估的核心对象

**【标准机型配置】**

| **组件**     | **机型**     | **节点数** | **CPU** | **内存** | **硬盘** | **说明**                       |
| ------------ | ------------ | ---------- | ------- | -------- | -------- | ------------------------------ |
| **DSS**      | 虚拟机       | 固定2      | 8核     | 16GB     | 500GB    | 工作流调度服务，不直接处理数据 |
| **Linkis**   | 虚拟机       | 固定4      | 32核    | 64GB     | 1024GB   | 任务提交网关，资源随 YARN 释放 |
| **Streamis** | 物理机 WB25X | ≥2         | 90核    | 192GB    | 7TB      | 流式应用核心计算节点，按需扩容 |

==**【Streamis扩容公式】**==

**TPS 的定义与场景**

- **<font color = '#8D0101'>TPS（Transactions Per Second）</font>** ：表示**<font color = '#8D0101'>每秒处理的日志事件数</font>**，是衡量流处理系统**吞吐量**的核心指标
  - 评估 Streamis 集群处理 **日志回调** 的能力（如实时监控、审计日志上报）。

**流式应用日志 TPS 计算** 

- **单应用 TPS** ：100~750/s（经验值取 **500 TPS/应用** ）。
- **总峰值 TPS** ：600应用×500(TPS/应用)=300,000TPS
- **单机容量** ：每台物理机WB25X支持 **150,000 TPS** （受 Nginx 转发限制）。
- **扩容公式** ：所需机器数=总TPS/单机容量=150,000/300,000=2台

**成本分摊逻辑**

- **单机支撑应用数** = 150,000 TPS / 500 TPS/应用 = **300 应用/机器** 。

  ![image-20250326220032114](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/image-20250326220032114.png)

  ![image-20250326220106603](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250326220106603.png)
