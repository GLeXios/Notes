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

![image-20250424005332571](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424005332571.png)

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

![image-20250424005128107](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424005128107.png)

【**5. 数据库与外部依赖**】

**5.1 数据库**

慢查询 

- MySQL：`SHOW PROCESSLIST;` 或开启慢查询日志（`long_query_time=1`）。

锁竞争 

- 检查表锁或行锁（如`SHOW ENGINE INNODB STATUS`）





## 如何读取百万数据效率提高

1.使用多线程：建立适当的线程数，对大数据量进行读取。在读取的时候，可以不加锁，在写入的时候，在数据库进行加行级锁；2.建立索引。

## 如何防止url重复提交

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps1-4133580.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps2.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps3-4133580.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps4-4133580.jpg) 

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

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps5-4133580.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps6-4133580.jpg) 

## SQL查询慢如何解决

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps7-4133580.jpg) 

## 大数据量下慢查询优化

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps8-4133580.jpg) 

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

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps27-4133580.jpg) 

**2.优化逻辑查询(sql优化)**

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps28-4133580.jpg)

**3.优化物理查询**

​	物理查询优化是在确定了逻辑查询优化之后，采用**物理优化技术（比如索引等）**，通过计算代价模型对各种可能的访问路径进行估算，从而找到执行方式中代价最小的作为执行计划。在这个部分中，我们需要掌握的重点是对索引的创建和使用。

![image-20250319140846737](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319140846737-4133580.png)

**4.使用Redis或Memcached作为缓存**

![image-20250319141009256](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319141009256-4133580.png)

![image-20250319141016229](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319141016229-4133580.png)

**5.库级优化**

**【读写分离】**

![image-20250319141125708](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319141125708-4133580.png)

![image-20250319141150939](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319141150939-4133580.png)



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

  ![image-20220628212029574](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20220628212029574-5386793.png)

  ![image-20250319001708346](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250319001708346-5386793.png)

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

![image-20250424011630042](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424011630042.png)

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

![image-20250410004224899](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250410004224899.png)

![image-20250410004538016](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 952 708"%3E%3C/svg%3E)

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

![image-20250410004933662](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250410004933662.png)

![image-20250410005028940](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250410005028940.png)

![image-20250410005125989](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250410005125989.png)

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

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps9-4554369.jpg)

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

![image-20250424000200309](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424000200309.png)

**【步骤3：合并所有桶中的 Top 100词】**

具体方法是：

1. 创建一个大小为 100 的**小顶堆（Min-Heap**）。
2. 遍历这 500个列表/哈希表中所有的（词，频率）对。
3. 对于每一个（词，频率）对：
   1. 如果堆的大小 ＜100，直接将该对（基于频率）加入堆中。
   2. 如果堆的大小 ==100，将当前词的频率与**堆顶元素（堆中频率最小的词）**的频率比较。
   3. 如果当前词的频率 ＞堆顶词的频率，则移除堆顶元素，并将当前（词，频率）对插入堆中。 • 当遍历完所有500 个小文件的所有词频信息后，小顶堆中剩下的100个元素就是全局频率最高的 100个词。

![image-20250424000428406](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424000428406.png)

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

![image-20250424000810643](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424000810643.png)

### 最热门的10个查询串

![image-20250424000942870](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1026 814"%3E%3C/svg%3E)

![image-20250424001007706](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250424001007706.png)

![image-20250424001027053](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1040 1124"%3E%3C/svg%3E)

### 每天热门 100词（分流＋ 哈希）

**题目描述**：某搜索公司一天的用户搜索词汇量达到百亿级别，请设计一种方法在内存和计算资源允许的情况下，求出每天热门的 Top 100词汇。

**分流＋ 哈希**

![image-20250424001251324](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1010 694"%3E%3C/svg%3E)

![image-20250424001318082](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1014 916"%3E%3C/svg%3E)

![image-20250424001344534](data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1030 798"%3E%3C/svg%3E)





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

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps11-4133580.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps12-4133580.jpg) 

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

![image-20250430125202773](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250430125202773.png)

![image-20250430125213285](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250430125213285.png)



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

  ![kafka-before](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/kafka-before.png)

**Kafka 的高可用性的体现**

- Kafka 0.8 以后，提供了 HA 机制，就是  **replica（复制品） 副本机制**。每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可。

- **为何只能读写 leader？**

  - 很简单，**要是你可以随意读写每个 follower，那么就要 care 数据一致性的问题**，系统复杂度太高，很容易出问题。Kafka 会均匀地将一个 partition 的所有 replica 分布在不同的机器上，这样才可以提高容错性。

    ![kafka-after](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/kafka-after.png)

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

  ![mq-10](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/mq-10.png)

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

    ![image-20250512115708357](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250512115708357.png)

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

![image-20250514144511130](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514144511130.png)

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

![image-20250514151546545](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514151546545-7208511.png)

初步怀疑可能由以下原因导致：

1. **JVM 参数配置差异** ：生产环境可能分配了更大的堆内存或更合理的 GC 策略。
2. **代码逻辑差异** ：测试环境可能引入了特定的测试组件或配置。
3. **依赖组件行为异常** ：某些仅在测试环境启用的组件存在内存泄漏。

#### 问题排查流程

**（1）定位堆栈文件**

- **JVM 参数配置** ：通过 `-XX:+HeapDumpOnOutOfMemoryError` 和 `-XX:HeapDumpPath` 参数定位堆栈文件（`.hprof`）的生成路径。
  - 或者：![image-20250514151733172](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514151733172-7208511.png)，定位到了虚拟机的日志的输出目录，用简单的命令寻找到堆栈文件：![image-20250514151755128](/Users/glexios/Library/Mobile%252520Documents/com~apple~CloudDocs/Notes/%2525E9%25259D%2525A2%2525E8%2525AF%252595%2525E8%2525A1%2525A5%2525E5%252585%252585%2525E7%2525AC%252594%2525E8%2525AE%2525B0.assets/image-20250514151755128.png)
  - ![image-20250514151810247](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514151810247-7208511.png)

- **容器环境限制** ：因测试环境部署在容器中，需通过 **容器 → 跳板机 → SFTP 服务器 → 本地开发机** 的链路下载堆栈文件

**（2）堆栈文件分析**

![image-20250514151936510](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514151936510-7208511.png)

​	通过 jvisualvm.exe 应用 文件 -> 装入 的方式导入刚才下载的文件，载入之后会有这样的选项卡

- 在概要的地方就给出了出问题的关键线程：发现 OOM 原因为 **堆内存溢出（Java heap space）** ，将堆栈信息点进去，发现堆栈是由于字符串的复制操作导致发生了 oom。
- 同时通过堆栈信息可以定位到 UnitTestContextHolder.java 类
- ![image-20250514152614559](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514152614559-7208511.png)

**【类分析】**

​	通过类分析视图，我们可以简单的定位到占据内存最高的类。**这里可以按照 实例数 或者 内存大小进行排序**，查询是否有某个类的实例数量异常，比如远远的高于其他的类，通常这些异常值就是可能发生内存泄露的地方。

- 我们这里由于 char 类实例数比较高，所有我优先往内存泄漏方向思考。

![image-20250514152522535](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514152522535-7208511.png)

**【实例数分析】**

![image-20250514152658794](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514152658794-7208511.png)

发现全是这种字符的实例，选中【实例】，右键可以分析【最近的垃圾回收节点】

这里会显示这些对象为何没有被回收，一般是静态变量或者是全局对象引用导致。

![image-20250514153311959](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514153311959-7208511.png)

通过上面图片可以看出，图中主要是由静态字段导致

**（3）代码分析**

​	代码是发生在项目中引入的企同的试点单元测试组件，刚好我们项目参与了试点，由于是单元测试组件，只在编译或者测试的依赖包中引入，正式的未引入

![image-20250514153410531](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514153410531-7208511.png)

![image-20250514153435758](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514153435758-7208511.png)

这里使用到了 ThreadLocal 做一些对象的存储，但是我发现 调用 setAttribute(String key,Object value)的地方有 33 处

![image-20250514153502559](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514153502559-7208511.png)

但是调用 clear()的地方只有3处。正常来说凡是使用 ThreadLocal 的地方，set 和 clear() 都应该成对出现。

![image-20250514153516478](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514153516478-7208511.png)

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

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250313221216979.png" alt="image-20250313221216979" style="zoom:67%;" />



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

![image-20250514150555467](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250514150555467.png)



## Java故障诊断

### 故障定位思路

- **CPU 相关问题，可以使用 <font color = '#8D0101'>top、vmstat、pidstat、ps </font>等工具排查；**
- **内存相关问题，可以使用 <font color = '#8D0101'>free、top、ps、vmstat、cachestat、sar</font> 等工具排查；**
- **IO 相关问题，可以使用 <font color = '#8D0101'>lsof、iostat、pidstat、sar、iotop、df、du </font>等工具排查；**
- **网络相关问题，可以使用 <font color = '#8D0101'>ifconfig、ip、nslookup、dig、ping、tcpdump、iptables</font> 等工具排查**

【**一般来说，服务器故障诊断的整体思路如下**】

<img src="/Users/glexios/Library/Mobile%2520Documents/com~apple~CloudDocs/Notes/%25E9%259D%25A2%25E8%25AF%2595%25E7%25AC%2594%25E8%25AE%25B0.assets/20200309181645.png" alt="img" style="zoom: 50%;" /><img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/20200309181831.png" alt="img" style="zoom: 67%;" />

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

![image-20250411234839988](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250411234839988.png)

![image-20250411234853716](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250411234853716.png)

​	TIME_WAIT 和 CLOSE_WAIT 是啥意思相信大家都知道。 在线上时，我们可以直接用命令`netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`来查看 time-wait 和 close_wait 的数量

用 ss 命令会更快`ss -ant | awk '{++S[$1]} END {for(a in S) print a, S[a]}'`

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/2019-11-04-083830.png)

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

![image-20250413110811534](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413110811534.png)

![image-20250413110825654](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413110825654.png)

### 抽象工厂模式

​	前面介绍的工厂方法模式中考虑的是一类产品的生产，如畜牧场只养动物、电视机厂只生产电视机、传智播客只培养计算机软件专业的学生等。

​	这些工厂只生产同种类产品，同种类产品称为同等级产品，也就是说：**工厂方法模式只考虑生产同等级的产品**，但是**在现实生活中许多工厂是综合型的工厂**，能生产多等级（种类） 的产品，如电器厂既生产电视机又生产洗衣机或空调，大学既有软件专业又有生物专业等。

**<font color = '#8D0101'>抽象工厂模式是工厂方法模式的升级版本，工厂方法模式只生产一个等级的产品，而抽象工厂模式可生产多个等级的产品。</font>**

**抽象工厂模式的主要角色如下：**

- **抽象工厂（Abstract Factory）**：提供了创建产品的接口，它包含**多个创建产品的方法**，可以**创建多个不同等级的产品**。
- **具体工厂（Concrete Factory）**：主要是实现抽象工厂中的多个抽象方法，完成具体产品的创建。
- **抽象产品（Product）**：定义了产品的规范，描述了产品的主要特性和功能，抽象工厂模式有多个抽象产品。
- **具体产品（ConcreteProduct）**：实现了抽象产品角色所定义的接口，由具体工厂来创建，它 同具体工厂之间是多对一的关系。

![image-20250413121158977](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413121158977.png)

![image-20250413121219845](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413121219845.png)

![image-20250413121235402](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413121235402.png)

![image-20250413121319600](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413121319600.png)

### 手写单例（singleton）设计模式

![image-20250409002955025](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002955025-4541422.png)

**单例模式的优点：**

​	由于单例模式只生成一个实例，减少了系统性能开销，当一个对象的产生需要比较多的资源时，如读取配置、产生其他依赖对象时，则可以通过在应用启动时直接产生一个单例对象，然后永久驻留内存的方式来解决

**饿汉式**：坏处:对象加载时间过长;好处:饿汉式是线程安全的。

**懒汉式**：好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![image-20250409002716615](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002716615-4541422.png)

#### 饿汉式

坏处:对象加载时间过长;好处:饿汉式是线程安全的。

![image-20250409002741419](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002741419-4541422.png)

#### 懒汉式

好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps15-4133580-4541422.jpg)

#### 单例模式之懒汉式（双重校验锁）

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps16-4541422.jpg) 

假设有两个线程a和b调用getInstance()方法，假设a先走，一路走到4这一步，执行instance = new Singleton()这句代码

这里如果变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序：

![image-20250409003110512](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003110512-4541422.png)

​	此时，线程b刚刚进来执行到1（看上面的代码块），就有可能会看到instance不为null，然后线程b也就不会等待监视器锁，而是直接返回instance。问题是这个instance可能还没执行完构造方法（线程a此时还在4这一步），所以**线程b拿到的instance是不完整的**，**它里面的属性值可能是初始化的零值(0/false/null)，而不是线程a在构造方法中指定的值**。

#### 反射破解单例模式

以上五种单例模式的实现方式中，前四种方式都是不太安全的，饿汉、懒汉、双重检查锁、静态内部类这四种方式都可以使用反射进行破解

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps17-4133580-4541422.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps18-4133580-4541422.jpg) 

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

![image-20250413162618013](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413162618013.png)

#### JDK动态代理

​	Java中提供了一个动态代理类Proxy，Proxy并不是我们上述所说的代理对象的类，而是提供了一个**创建代理对象的静态方法（newProxyInstance方法）来获取代理对象**。

**<font color = '#8D0101'>注意：ProxyFactory不是代理模式中所说的代理类，而代理类是程序在运行过程中动态的在内存中生成的类。</font>**

**执行流程如下：**

1）在测试类中通过代理对象调用sell()方法
2）根据多态的特性，执行的是代理类（$Proxy0）中的sell()方法
3）代理类（$Proxy0）中的sell()方法中又调用了`InvocationHandler`接口的子实现类对象的`invoke`方法
4）`invoke`方法通过反射执行了真实对象所属类(TrainStation)中的`sell()`方法

![image-20250413170951871](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413170951871.png)

![image-20250413170929023](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413170929023.png)

![image-20250413171006093](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413171006093.png)

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

![image-20250413174723864](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413174723864.png)

![image-20250413174734589](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413174734589.png)

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

![image-20250413175617523](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413175617523.png)

![image-20250413175635827](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413175635827.png)

![image-20250413175657625](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413175657625.png)

![image-20250413175710818](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413175710818.png)



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

![image-20250411225326371](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250411225326371-4541422.png)

![image-20250411225341196](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250411225341196-4541422.png)

![image-20250411225459145](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250411225459145-4541422.png)

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

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413181633986.png" alt="image-20250413181633986" style="zoom: 48%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413181741405.png" alt="image-20250413181741405" style="zoom:40%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250413181812398.png" alt="image-20250413181812398" style="zoom:40%;" />

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

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps13.png) 

当缓冲区满的时候，生产者停止执行，让其他线程进行

当缓冲区空的时候，消费者停止执行，让其他线程执行

当生产者向缓冲区放入一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态；

当消费者从缓冲区取出一个产品时，向其他等待的线程发出可执行的通知，同时放弃锁，使自己处于等待状态

**<font color = '#8D0101'>等待通知机制wait/notify/notifyAll</font>**

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250310233338783-4133580.png" alt="image-20250310233338783" style="zoom:80%;" />

- `wait` - `wait` 会自动**释放当前线程占有的对象锁**，并请求操作系统挂起当前线程，**让线程从 `Running` 状态转入 `Waiting` 状态**，等待 `notify` / `notifyAll` 来唤醒。如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 `notify` 或者 `notifyAll` 来唤醒挂起的线程，造成死锁。
- `notify` - 唤醒一个正在 `Waiting` 状态的线程，并让它拿到对象锁，具体唤醒哪一个线程由 JVM 控制 
- `notifyAll` - 唤醒所有正在 `Waiting` 状态的线程，接下来它们需要竞争对象锁。

> [!WARNING]
>
> - **`wait`、`notify`、`notifyAll` 都是 `Object` 类中的方法**，而非 `Thread`。
> - **`wait`、`notify`、`notifyAll` 只能用在 `synchronized` 方法或者 `synchronized` 代码块中使用，否则会在运行时抛出 `IllegalMonitorStateException`**。
> - **wait()，notify()，notifyAll()三个方法的调用者必须是同步代码块或同步方法中的同步监视器。**

**为什么 `wait`、`notify`、`notifyAll` 不定义在 `Thread` 中？为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？**

首先，需要了解几个基本知识点：

- 每一个 Java 对象都有一个与之对应的 **监视器（monitor）**
- 每一个监视器里面都有一个 **对象锁** 、一个 **等待队列**、一个 **同步队列**

了解了以上概念，我们回过头来理解前面两个问题。

**为什么这几个方法不定义在 `Thread` 中？**

- 由于**每个对象都拥有对象锁**，让当前线程等待某个对象锁，自然应该基于这个对象（`Object`）来操作，而非使用当前线程（`Thread`）来操作。因为当前线程可能会等待多个线程的锁，如果基于线程（`Thread`）来操作，就非常复杂了。

**为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？**

- 如果调用某个对象的 `wait` 方法，当前线程必须拥有这个对象的对象锁，因此调用 `wait` 方法必须在 `synchronized` 方法和 `synchronized` 代码块中，如果没有获取到锁前调用wait()方法，会抛出java.lang.IllegalMonitorStateException异常。

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409013043263.png" alt="image-20250409013043263" style="zoom:60%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409013058767.png" alt="image-20250409013058767" style="zoom:55%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409013131548.png" alt="image-20250409013131548" style="zoom:55%;" />

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

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002955025.png" alt="image-20250409002955025" style="zoom:50%;" />

**单例模式的优点：**

​	由于单例模式只生成一个实例，减少了系统性能开销，当一个对象的产生需要比较多的资源时，如读取配置、产生其他依赖对象时，则可以通过在应用启动时直接产生一个单例对象，然后永久驻留内存的方式来解决

**饿汉式**：坏处:对象加载时间过长;好处:饿汉式是线程安全的。

**懒汉式**：好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![image-20250409002716615](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002716615.png)

### 饿汉式

坏处:对象加载时间过长;好处:饿汉式是线程安全的。

![image-20250409002741419](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409002741419.png)

### 懒汉式

好处:延迟对象的创建;坏处:目前的写法，会线程不安全。---》到多线程内容时，再修改

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps15-4133580.jpg)

### 单例模式之懒汉式（双重校验锁）

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps16.jpg) 

假设有两个线程a和b调用getInstance()方法，假设a先走，一路走到4这一步，执行instance = new Singleton()这句代码

这里如果变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序：

![image-20250409003110512](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003110512.png)

​	此时，线程b刚刚进来执行到1（看上面的代码块），就有可能会看到instance不为null，然后线程b也就不会等待监视器锁，而是直接返回instance。问题是这个instance可能还没执行完构造方法（线程a此时还在4这一步），所以**线程b拿到的instance是不完整的**，**它里面的属性值可能是初始化的零值(0/false/null)，而不是线程a在构造方法中指定的值**。

### 反射破解单例模式

以上五种单例模式的实现方式中，前四种方式都是不太安全的，饿汉、懒汉、双重检查锁、静态内部类这四种方式都可以使用反射进行破解

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps17-4133580.jpg) 

![img](/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/wps18-4133580.jpg) 

源码中我们可以看见这么一句话，如果你的这个类型是枚举类型，想要通过反射去创建对象时会抛出一个异常（不能通过反射创建枚举对象）

## 多个线程顺序执行问题

### 三个线程分别打印A，B，C

要求这三个线程一起运行，打印 n 次，输出形如“ABCABCABC....”的字符串

**【利用synchronized+wait()/notify()】**

​	**思路**：用对象监视器来实现，通过wait和notify()方法来实现等待、通知的逻辑，A执行后，唤醒B，B执行后唤醒C，C执行后再唤醒A，这样循环的等待、唤醒来达到目的。使用一个取模的判断逻辑C%M ==N，题为3个线程，所以可以按取模结果编号：0、1、2，他们与3取模结果仍为本身，则执行打印逻辑。

**<font color = '#8D0101'>state是volatile注意！！</font>**

- **<font color = '#8D0101'>`Object.wait()` 方法可能因系统中断或其他原因提前返回（虚假唤醒），此时线程需要重新检查条件是否满足。</font>**

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003447207.png" alt="image-20250409003447207" style="zoom: 50%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003515747.png" alt="image-20250409003515747" style="zoom: 50%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003519564.png" alt="image-20250409003519564" style="zoom: 50%;" />

### 两个线程交替打印奇数和偶数

- 使用对象监视器实现

​	两个线程A、B竞争同一把锁，只要其中一个线程获取锁成功，就打印 ++i，并通知另一线程从等待集合中释放，然后自身线程加入等待集合并释放锁即可

**`private static volatile Object lock = new Object();`**

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003933768.png" alt="image-20250409003933768" style="zoom: 67%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003948350.png" alt="image-20250409003948350" style="zoom: 67%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409003952621.png" alt="image-20250409003952621" style="zoom: 67%;" />

### 用两个线程，交替输出1A2B3C4D...26Z

- 跟两个线程交替打印奇数和偶数的对象监视器一样实现

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409004226701.png" alt="image-20250409004226701" style="zoom:50%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409004230498.png" alt="image-20250409004230498" style="zoom:50%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409004235464.png" alt="image-20250409004235464" style="zoom:50%;" />

### 通过N个线程顺序循环打印从0至100

​	用等待唤醒机制，当一个线程执行完当前任务后再唤醒其他线程来执行任务，自己去休眠，避免在线程执行任务的时候其他线程处于忙等状态，浪费cpu资源。

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409012732358.png" alt="image-20250409012732358" style="zoom:60%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409012740339.png" alt="image-20250409012740339" style="zoom:60%;" />

<img src="/Users/glexios/Library/Mobile%20Documents/com~apple~CloudDocs/Notes/%E5%9C%BA%E6%99%AF%E9%A2%98+%E9%A1%B9%E7%9B%AE%E6%8E%92%E6%9F%A5%E9%97%AE%E9%A2%98+%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F+%E6%89%8B%E5%86%99%E4%BB%A3%E7%A0%81+Streamis.assets/image-20250409012745730.png" alt="image-20250409012745730" style="zoom:60%;" />

# 