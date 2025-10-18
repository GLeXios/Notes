# 并发

## 并发概念

### 并发和并行

并发和并行是最容易让新手费解的概念，那么如何理解二者呢？其最关键的差异在于：是否是**同时**发生：

- **并发**：是指具备处理多个任务的能力，但不一定要同时。
- **并行**：是指具备同时处理多个任务的能力。

下面是我见过最生动的说明，摘自 [并发与并行的区别是什么？——知乎的高票答案 (opens new window)](https://www.zhihu.com/question/33515481/answer/58849148)：

- 你吃饭吃到一半，电话来了，你一直到吃完了以后才去接，这就说明你不支持并发也不支持并行。
- 你吃饭吃到一半，电话来了，你停了下来接了电话，接完后继续吃饭，这说明你支持并发。
- 你吃饭吃到一半，电话来了，你一边打电话一边吃饭，这说明你支持并行。

### 同步和异步

- **同步**：是指在发出一个调用时，在没有得到结果之前，该调用就不返回。但是一旦调用返回，就得到返回值了。
- **异步**：则是相反，调用在发出之后，这个调用就直接返回了，所以没有返回结果。换句话说，当一个异步过程调用发出后，调用者不会立刻得到结果。而是在调用发出后，被调用者通过状态、通知来通知调用者，或通过回调函数处理这个调用。

举例来说明：

- 同步就像是打电话：不挂电话，通话不会结束。
- 异步就像是发短信：发完短信后，就可以做其他事；当收到回复短信时，手机会通过铃声或振动来提醒。

### 阻塞和非阻塞

阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态：

- **阻塞**：是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。
- **非阻塞**：是指在不能立刻得到结果之前，该调用不会阻塞当前线程。

举例来说明：

- 阻塞调用就像是打电话，通话不结束，不能放下。
- 非阻塞调用就像是发短信，发完短信后，就可以做其他事，短信来了，手机会提醒

### 进程(process)、线程和协程

**<u>（1）进程</u>**

​	**<font color = '#8D0101'>进程是系统进行资源分配的基本单位。</font>**

​	每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。进程是一个动态的过程：有它自身的产生、存在和消亡的过程（生命周期）。如：运行中的QQ，运行中的MP3播放器。程序是静态的，进程是动态的

- 我们编写的代码只是一个存储在硬盘的静态文件，通过编译后就会生成二进制可执行文件，当我们运行这个可执行文件后，它会被装载到内存中，接着CPU会执行程序中的每一条指令，那么这个**运行中的程序，就被称为「进程」（Process）**。

**<u>（2）线程</u>**

​	**<font color = '#8D0101'>线程是操作系统进行调度的基本单位</font>**

​	如果一个进程有多个子任务时，只能逐个得执行这些子任务，很影响效率，所以那么能不能让这些子任务同时执行呢？于是人们又提出了线程的概念，**让一个线程执行一个子任务，这样一个进程就包含了多个线程，每个线程负责一个单独的子任务。**

​	线程是进程的一个实体，**是CPU调度和分派的基本单位**，它是比进程更小的能独立运行的单位。线程拥有自己的**程序计数器、本地方法栈、虚拟机栈**，同一个进程中的多个线程可以共享进程所拥有的资源（**堆与方法区**）。**线程间通信主要通过共享内存**，上下文切换很快，资源开销少，但相比进程不够稳定，容易丢失数据。**当一个进程中的线程挂掉之后（非正常退出），就会导致该线程占有的资源永远无法释放**，从而影响其他线程的正常工作。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310163558225.png" alt="image-20250310163558225" style="zoom:60%;" />

**<u>（3）协程</u>**

​	是一种用户态的轻量级线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁地访问全局变量，所以上下文切换非常快

### 进程与线程对比

**进程属于操作系统的并发，而线程是属于进程的内部并发**

- **java中，进程是操作系统进行资源分配的基本单位，而线程是操作系统进行调度的基本单位**。
- **地址空间**：线程是进程内的一个执行单元，进程内至少有一个线程，它们共享进程的地址空间，而进程有自己独立的地址空间。
- **资源拥有**：进程是资源分配和拥有的单位，同一个进程内的线程共享进程资源。
- 二者均可并发执行。
- 每个独立的进程有一个程序运行入口、顺序执行序列和程序的出口，但是线程不能够独立执行，必须依存在应用。
- 线程更轻量，**线程上下文切换成本一般要比进程上下文切换低**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310164011782.png" alt="image-20250310164011782" style="zoom:50%;" />

### Java中的线程与操作系统的线程有什么区别

**<font color = '#8D0101'>java线程与操作系统线程绑定</font>**

在多核操作系统中，jvm也会允许在一个进程内同时并发执行多个线程。java中的线程和操作系统中的线程分别存在于虚拟机和操作系统中。 

线程在创建时，其实是先创建一个java线程，等到本地存储、程序计数器、缓冲区等都分配好以后，**JVM会调用操作系统的方法，创建一个与java线程绑定的原生线程**。线程的调度是由操作系统负责的，当操作系统为线程分配好时间片以后，就会调用java线程的run方法执行线程。当线程结束后，会释放java线程和原生线程所占用的资源。

### 并发、多线程的优点

**<u>（1）提升资源利用率</u>**

- 在一个程序中，有很多操作是非常耗时的，如数据库读写操作，IO操作等，如果使用单线程，那么程序就必须等待这些操作执行完成之后才能执行其他操作，使用多线程，可以在将耗时任务放在后台继续执行的同时，同时执行其他操作。
- 多核时代：多核时代多线程主要是为了**提高CPU利用率**。举个例子：假如我们要计算一个复杂的任务，我们只用一个线程的话，只会有一个CPU核被利用到，而创建多个线程就可以让多个CPU核被利用到，这样就提高了CPU的利用率。

**<u>（2）程序响应更快</u>**

- 将一个单线程应用程序变成多线程应用程序的另一个常见的目的是**实现一个响应更快的应用程序**

eg：设想一个服务器应用，**它在某一个端口监听进来的请求。当一个请求到来时，它去处理这个请求，然后再返回去监听**

- 如果一个请求需要占用大量的时间来处理，在这段时间内新的客户端就无法发送请求给服务端。只有服务器在监听的时候，请求才能被接收。

  ```java
  while(server is active) {
      listen for request
      process request
  }
  ```

- 另一种设计是，**监听线程把请求传递给工作者线程(worker thread)，然后立刻返回去监听。而工作者线程则能够处理这个请求并发送一个回复给客户端**。这种设计如下所述：

  - 这种方式，服务端线程迅速地返回去监听。因此，更多的客户端能够发送请求给服务端。这个服务也变得响应更快

  ```java
  while(server is active) {
      listen for request
      hand request to worker thread
  }
  ```

**<u>（3）降低切换成本</u>**

- 从计算机底层来说：线程可以比作是轻量级的进程，是程序执行的最小单位，线程间的切换和调度的成本远远小于进程。另外，多核CPU时代意味着多个线程可以同时运行，这减少了线程上下文切换的开销（CPU切换也需要时间）。

### 临界区和竞态条件

- **竞态条件（Race Condition）**：多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件。
- **临界区（Critical Sections）**：导致竞态条件发生的代码区称作临界区（一段代码块内如果存在对共享资源的多线程读写操作）

​	为避免临界区的竞态条件发生（解决线程安全问题）：**①阻塞式的解决方案：synchronized、lock；②非阻塞式的解决方案：原子变量。**

### 并发的问题（线程安全问题等）

我们知道了并发带来的好处：提升资源利用率、程序响应更快，同时也要认识到并发带来的问题，主要有：

- 安全性问题
- 活跃性问题
- 性能问题

#### 安全性问题

**并发安全**：是指保证程序的正确性，使得并发处理结果符合预期。

并发安全需要保证几个基本特性：

- **可见性** - 是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上，`volatile` 就是负责保证可见性的。
- **原子性** - 简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制（加锁：`sychronized`、`Lock`）实现。
- **有序性** - 是保证线程内串行语义，避免指令重排等。

**<u>（1）缓存导致的可见性问题</u>**

- 多核时代，每颗 CPU 都有自己的缓存，这时 CPU 缓存与内存的数据一致性就没那么容易解决了，当多个线程在不同的 CPU 上执行时，这些线程操作的是不同的 CPU 缓存。比如下图中，线程 A 操作的是 CPU-1 上的缓存，而线程 B 操作的是 CPU-2 上的缓存，很明显，这个时候线程 A 对变量 V 的操作对于线程 B 而言就不具备可见性了。

**<u>（2）线程切换带来的原子性问题</u>**

==**<u>（3）编译优化带来的有序性问题（使用volatile）</u>**==

- 顾名思义，有序性指的是程序按照代码的先后顺序执行。编译器为了优化性能，有时候会改变程序中语句的先后顺序，例如程序中：“a=6；b=7；”编译器优化后可能变成“b=7；a=6；”，在这个例子中，编译器调整了语句的顺序，但是不影响程序的最终结果。不过有时候编译器及解释器的优化可能导致意想不到的 Bug。

**eg：双重检查创建单例对象，**例如下面的代码：在获取实例 getInstance() 的方法中，我们首先判断 instance 是否为空，如果为空，则锁定 Singleton.class 并再次检查 instance 是否为空，如果还为空则创建 Singleton 的一个实例。

```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
```

- 假设有两个线程 A、B 同时调用 getInstance() 方法，他们会同时发现 `instance == null` ，于是同时对 Singleton.class 加锁，此时 JVM 保证只有一个线程能够加锁成功（假设是线程 A），另外一个线程则会处于等待状态（假设是线程 B）；线程 A 会创建一个 Singleton 实例，之后释放锁，锁释放后，线程 B 被唤醒，线程 B 再次尝试加锁，此时是可以加锁成功的，加锁成功后，线程 B 检查 `instance == null` 时会发现，已经创建过 Singleton 实例了，所以线程 B 不会再创建一个 Singleton 实例。

但实际上这个 getInstance() 方法并不完美。问题出在哪里呢？出在 new 操作上，我们以为的 new 操作应该是：

1. 分配一块内存 M；
2. 在内存 M 上初始化 Singleton 对象；
3. 然后 M 的地址赋值给 instance 变量。

但是实际上优化后的执行路径却是这样的：

1. 分配一块内存 M；
2. 将 M 的地址赋值给 instance 变量；
3. 最后在内存 M 上初始化 Singleton 对象。

- 我们假设线程 A 先执行 getInstance() 方法，当执行完指令 2 时恰好发生了线程切换，切换到了线程 B 上；如果此时线程 B 也执行 getInstance() 方法，那么线程 B 在执行第一个判断时会发现 `instance != null` ，所以直接返回 instance，而此时的 instance 是没有初始化过的，如果我们这个时候访问 instance 的成员变量就可能触发空指针异常。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200701111050.png" alt="img" style="zoom:50%;" />

使用volatile关键字解决：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310170930279.png" alt="image-20250310170930279" style="zoom:50%;" />

#### 保证并发安全的思路：阻塞同步、非阻塞同步、无同步

==**Java中使用线程同步的方式来解决线程安全问题**==

**<u>（1）互斥同步（阻塞同步）</u>**

​	互斥同步是最常见的并发正确性保障手段。**同步是指在多线程并发访问共享数据时，保证共享数据在同一时刻只能被一个线程访问**。

- **互斥同步最主要的问题是线程阻塞和唤醒所带来的性能问题**，互斥同步属于一种悲观的并发策略，总是认为只要不去做正确的同步措施，那就肯定会出现问题。无论共享数据是否真的会出现竞争，它都要进行加锁

==**典型案例**==

- **同步方法：**Synchronized修饰的同步方法；Synchronized也可以修饰静态方法，相当于锁住当前类;
- **同步代码块**：Synchronized修饰的同步代码块；
- **volatile关键字实现线程同步**，相当于告诉虚拟机，该变量可能被修改，因此每次使用该域都需要重新计算，而不是从寄存器中取出数据。从而实现线程的同步。
- **使用可重入锁**。使用Reentrantlock类定义锁，其中lock()开启锁，unlock()关闭锁，要注意及时关闭锁，不然会进入死锁状态

**<u>（2）非阻塞同步</u>**

​	随着硬件指令集的发展，我们可以使用基于冲突检测的乐观并发策略：**先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地自旋重试，直到成功为止）**。这种乐观的并发策略的许多实现都不需要将线程阻塞，因此这种同步操作称为非阻塞同步。

这类乐观锁指令常见的有：

- 测试并设置（Test-and-Set）
- 获取并增加（Fetch-and-Increment）
- 交换（Swap）
- **比较并交换（CAS）**
- 加载链接、条件存储（Load-linked / Store-Conditional）

<font color = '#8D0101'>**Java 典型应用场景：J.U.C 包中的原子类（基于 `Unsafe` 类的 CAS 操作）**</font>

**<u>（3）无同步</u>**

​	要保证线程安全，不一定非要进行同步。同步只是保证共享数据争用时的正确性，如果一个方法本来就不涉及共享数据，那么自然无须同步。

Java 中的 **无同步方案** 有：

- **可重入代码** - 也叫纯代码。如果一个方法，它的 **返回结果是可以预测的**，即只要输入了相同的数据，就能返回相同的结果，那它就满足可重入性，当然也是线程安全的。
- **线程本地存储** - 使用 **<font color = '#8D0101'>`ThreadLocal` </font>为共享变量在每个线程中都创建了一个本地副本**，这个副本只能被当前线程访问，其他线程无法访问，那么自然是线程安全的

#### 活跃性问题（死锁、活锁、饥饿锁）

==见锁内容总结==

#### 性能问题（上下文切换）

**<font color = '#8D0101'>并发不一定比串行快</font>**。<font color = '#8D0101'>因为有创建线程和线程上下文切换的开销</font>

#### CPU上下文切换

<u>**（1）CPU上下文：**</u>

任务是交给CPU运行的，那么在每个任务运行前，CPU需要知道任务从哪里加载，又从哪里开始运行。

所以，操作系统需要事先帮**CPU设置好<font color = '#8D0101'>CPU寄存器和程序计数器</font>**。

- **CPU寄存器是CPU内部一个容量小，但是速度极快的内存（缓存）**。eg：寄存器像是你的口袋，内存像你的书包，硬盘则是你家里的柜子，如果你的东西存放到口袋，那肯定是比你从书包或家里柜子取出来要快的多。
- **程序计数器则是用来存储CPU正在执行的指令位置、或者即将执行的下一条指令位置**。

CPU寄存器和程序计数是CPU在运行任何任务前，所必须依赖的环境，这些环境就叫做CPU上下文。

<u>**（2）CPU上下文切换：**</u>

- CPU上下文切换就是先把前一个任务的CPU上下文（CPU寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。
- 系统内核会存储保持下来的上下文信息，当此任务再次被分配给CPU运行时，CPU会重新加载这些上下文，这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。

​	上面说到所谓的「任务」，主要包含**进程、线程和中断**。所以，可以根据任务的不同，==把CPU上下文切换分成：进程上下文切换、线程上下文切换==。

#### 进程的上下文切换

进程是由内核管理和调度的，**所以进程的切换只能发生在内核态**。

**进程的上下文切换不仅包含了<font color = '#8D0101'>虚拟内存、栈、全局变量等用户空间</font>的资源，还包括了<font color = '#8D0101'>内核堆栈、寄存器等内核空间</font>的资源**。

**发生的场景：**

- 为了保证所有进程可以得到公平调度，CPU时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当**某个进程的时间片耗尽了**，进程就从运行状态变为就绪状态，系统从就绪队列选择另外一个进程运行；
- **进程在系统资源不足（比如内存不足）时**，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行；
- 当进程通过**睡眠函数sleep**这样的方法将自己主动挂起时，自然也会重新调度；
- 当有**优先级更高的进程**运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行；

#### 线程的上下文切换

**线程是调度的基本单位，而进程则是资源拥有的基本单位。**

所谓操作系统的任务调度，实际上的**调度对象是线程，而进程只是给线程提供了虚拟内存、全局变量等资源**。

**线程上下文切换时，还得看线程是不是属于同一个进程：**

- 当两个线程不是属于同一个进程，则切换的过程就跟进程上下文切换一样；
- 当两个线程是属于同一个进程，因为虚拟内存是共享的，所以在切换时，**虚拟内存、全局变量这些资源就保持不动，只需要切换线程的私有数据比如栈和寄存器等不共享的数据**；

**线程发生上下文切换的场景：**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310173332306.png" alt="image-20250310173332306" style="zoom:67%;" />

#### 减少上下文切换的方法

- 无锁并发编程 - 多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的 ID 按照 Hash 算法取模分段，不同的线程处理不同段的数据。
- CAS 算法 - Java 的 Atomic 包使用 CAS 算法来更新数据，而不需要加锁。
- 使用最少线程 - 避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
- 使用协程 - 在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换

## 创建线程（Thread，Runnable，Callable，线程池）

### 继承Thread类

**Thread类本身也实现了Runnable接口，Thread类中持有Runnable的属性，执行线程run方法底层是调用Runnable的run方法**

通过继承 `Thread` 类创建线程的步骤：

1. 定义 `Thread` 类的子类，并覆写该类的 `run` 方法。`run` 方法的方法体就代表了线程要完成的任务，因此把 `run` 方法称为执行体。
2. 创建 `Thread` 子类的实例，即创建了线程对象。
3. 调用线程对象的 `start` 方法来启动该线程（<font color = '#8D0101'>该语句是主线程main做的</font>）
   1. 启动当前线程
   2. 调用当前线程的run()

> [!IMPORTANT]
>
> - start()方法底层其实是给CPU注册当前线程，并且触发run()方法执行；
> - 线程的启动**必须调用start()方法**，如果线程直接调用run()方法，相当于变成了普通类的执行，此时主线程将只执行该线程；
> - 建议线程先创建子线程，主线程的任务放在之后，否则主线程（main）永远是先执行完；
> - 不可以让已经start()的线程再次执行，如果重复执行，会报IllegalThreadStateException异常。

```java
public class ThreadDemo {

    public static void main(String[] args) {
        // 实例化对象
        MyThread tA = new MyThread("Thread 线程-A");
        MyThread tB = new MyThread("Thread 线程-B");
        // 调用线程主体
        tA.start();
        tB.start();
    }

    static class MyThread extends Thread {

        private int ticket = 5;
        MyThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                ticket--;
            }
        }
    }
}
```

<u>**继承Thread类的优缺点**：</u>

- **优点**：编码简单；
- **缺点**：线程类已经继承Thread类无法继承其他类了，功能不能通过继承拓展（单继承的局限性）

### 实现Runnable接口

**实现 `Runnable` 接口优于继承 `Thread` 类**，因为：

- Java 不支持多重继承，所有的类都只允许继承一个父类，但可以实现多个接口。如果继承了 `Thread` 类就无法继承其它类，这不利于扩展。
- 类可能只要求可执行就行，继承整个 `Thread` 类开销过大。
- **线程池可以放入Runnable或Callable线程任务对象**。
- 实现解耦操作，线程任务代码可以被多个线程共享，线程任务代码和线程独立

通过实现 `Runnable` 接口创建线程的步骤：

1. 定义 `Runnable` 接口的实现类，并覆写该接口的 `run` 方法。该 `run` 方法的方法体同样是该线程的线程执行体。
2. 创建 `Runnable` 实现类的实例，并以此实例作为 `Thread` 的 target 来创建 `Thread` 对象，该 `Thread` 对象才是真正的线程对象。
3. 调用线程对象的 `start` 方法来启动该线程。**（可以使用匿名内部类）**

```java
public class RunnableDemo {

    public static void main(String[] args) {
        // 实例化对象
        Thread tA = new Thread(new MyThread(), "Runnable 线程-A");
        Thread tB = new Thread(new MyThread(), "Runnable 线程-B");
        // 调用线程主体
        tA.start();
        tB.start();
    }

    static class MyThread implements Runnable {

        private int ticket = 5;

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                ticket--;
            }
        }
    }
}
```

### 实现Callable接口(异步模型)，Future接口、FutureTask 类

**<u>（1）Callable接口</u>**

​	**继承 Thread 类和实现 Runnable 接口这两种创建线程的方式都没有返回值**。所以，线程执行完后，无法得到执行结果。Java 1.5 后，提供了 **`Callable` 接口和 `Future` 接口，通过它们，可以在线程执行结束后，返回执行结果**。

- Callable与Runnable类似，同样是只有一个抽象方法的函数式接口。不同的是，Callable提供的方法是有返回值的，而且支持泛型

Callable 接口只声明了一个方法，这个方法叫做 call()：

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

- `Callable`一般是配合**线程池工具`ExecutorService`**来使用的。`ExecutorService`可以使用`submit`方法来让一个`Callable`接口执行，**submit()方法用于提交需要返回值的任务，它会返回一个`Future的对象`，我们后续的程序可以通过这个`Future`的`get`方法来获取返回值**，get()方法会阻塞当前线程直到任务完成，在 ExecutorService 接口中声明了若干个 submit 方法的重载版本：

```java
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

**<u>（2）Future接口</u>**

​	**Future<V>接口是用来获取异步计算结果的**，Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过**get方法获取执行结果，该方法会阻塞直到任务返回结果**。

- **Future接口的方法**

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

**<u>（3）FutureTask类--Future接口的实现类</u>**

FutureTask 类**实现了 RunnableFuture 接口**，RunnableFuture **继承了 Runnable 接口和 Future 接口。**

所以，FutureTask 既可以作为 Runnable 被线程执行，又可以作为 Future 得到 Callable 的返回值。

```java
public class FutureTask<V> implements RunnableFuture<V> {
    // ...
    public FutureTask(Callable<V> callable) {}
    public FutureTask(Runnable runnable, V result) {}
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

### Callable + Future + FutureTask 示例

通过实现 `Callable` 接口创建线程的步骤：

1. 创建 `Callable` 接口的实现类，并实现 `call` 方法。该 `call` 方法将作为线程执行体，并且有返回值。
2. 创建 `Callable` 实现类的实例，使用 `FutureTask` 类来包装 `Callable` 对象，该 `FutureTask` 对象封装了该 `Callable` 对象的 `call` 方法的返回值。
3. 使用 `FutureTask` 对象作为 `Thread` 构造器的参数创建并启动新线程。
4. 调用 `FutureTask` 对象的 `get` 方法来获得线程执行结束后的返回值。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310220521340.png" alt="image-20250310220521340" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310220533874.png" alt="image-20250310220533874" style="zoom:67%;" />

### 线程池创建

==见线程池章节==

## 线程生命周期（线程状态）

`java.lang.Thread.State` 中定义了 **6** 种不同的线程状态，在给定的一个时刻，线程只能处于其中的一个状态。

以下是各状态的说明，以及状态间的联系：

- **新建（New）** - 尚未调用 `start` 方法的线程处于此状态。此状态意味着：**创建的线程尚未启动**。

- **就绪（Runnable）** - 指该线程已经被创建（与操作系统线程关联），被start()后，将进入线程队列等待CPU时间片，此时它已具备了运行的条件，只是没分配到CPU资源。

- **运行状态（Running）**- 当就绪的线程被调度并获得CPU资源时,便进入运行状态，run()方法定义了线程的操作和功能。当CPU时间片用完，会从【运行状态】转换至【可运行状态】，会导致线程的上下文切换

- **阻塞（Blocked）** - 此状态意味着：**线程处于被阻塞状态**。表示线程在等待 `synchronized` 的隐式锁（Monitor lock）。`synchronized` 修饰的方法、代码块同一时刻只允许一个线程执行，其他线程只能等待，即处于阻塞状态。当占用 `synchronized` 隐式锁的线程释放锁，并且等待的线程获得 `synchronized` 隐式锁时，就又会从 `BLOCKED` 转换到 `RUNNABLE` 状态。

- **等待（Waiting）** - 此状态意味着：**线程无限期等待，直到被其他线程显式地唤醒**。 阻塞和等待的区别在于，阻塞是被动的，它是在等待获取 `synchronized` 的隐式锁。而等待是主动的，通过调用 `Object.wait` 等方法进入。

  | 进入方法                                                     | 退出方法                             |
  | ------------------------------------------------------------ | ------------------------------------ |
  | 没有设置 Timeout 参数的 `Object.wait` 方法                   | `Object.notify` / `Object.notifyAll` |
  | 没有设置 Timeout 参数的 `Thread.join` 方法                   | 被调用的线程执行完毕                 |
  | `LockSupport.park` 方法（Java 并发包中的锁，都是基于它实现的） | `LockSupport.unpark`                 |

- **定时等待（Timed waiting）** - 此状态意味着：**无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒**。

  | 进入方法                                                     | 退出方法                                        |
  | ------------------------------------------------------------ | ----------------------------------------------- |
  | `Thread.sleep` 方法                                          | 时间结束                                        |
  | 获得 `synchronized` 隐式锁的线程，调用设置了 Timeout 参数的 `Object.wait` 方法 | 时间结束 / `Object.notify` / `Object.notifyAll` |
  | 设置了 Timeout 参数的 `Thread.join` 方法                     | 时间结束 / 被调用的线程执行完毕                 |
  | `LockSupport.parkNanos` 方法                                 | `LockSupport.unpark`                            |
  | `LockSupport.parkUntil` 方法                                 | `LockSupport.unpark`                            |

- **终止(Terminated)** - 线程执行完 `run` 方法，或者因异常退出了 `run` 方法。此状态意味着：线程结束了生命周期。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20210102103928.png" alt="img" style="zoom: 60%;" />

## 线程的基本方法

### 基本方法清单

| 方法            | 描述                                                         |
| --------------- | :----------------------------------------------------------- |
| `run`           | 线程的执行实体。                                             |
| `start`         | 线程的启动方法。                                             |
| `currentThread` | 返回对当前正在执行的线程对象的引用。                         |
| `setName`       | 设置线程名称。                                               |
| `getName`       | 获取线程名称。                                               |
| `setPriority`   | 设置线程优先级。Java 中的线程优先级的范围是 [1,10]，一般来说，高优先级的线程在运行时会具有优先权。可以通过 `thread.setPriority(Thread.MAX_PRIORITY)` 的方式设置，默认优先级为 5。 |
| `getPriority`   | 获取线程优先级。                                             |
| `setDaemon`     | 设置线程为守护线程。                                         |
| `isDaemon`      | 判断线程是否为守护线程。                                     |
| `isAlive`       | 判断线程是否启动。                                           |
| `interrupt`     | 中断另一个线程的运行状态。                                   |
| `interrupted`   | 测试当前线程是否已被中断。通过此方法可以清除线程的中断状态。换句话说，如果要连续调用此方法两次，则第二次调用将返回 false（除非当前线程在第一次调用清除其中断状态之后且在第二次调用检查其状态之前再次中断）。 |
| `join`          | 可以使一个线程强制运行，线程强制运行期间，其他线程无法运行，必须等待此线程完成之后才可以继续执行。 |
| `Thread.sleep`  | 静态方法。将当前正在执行的线程休眠。                         |
| `Thread.yield`  | 静态方法。将当前正在执行的线程暂停，让其他线程执行。         |

### 线程启动：start()、run()

- **start()**：使用start启动新的线程，此线程处于就绪（可运行）状态，通过新的线程间接执行run()中的代码。
- **run()**：称为线程体，包含了要执行的这个线程的内容，方法运行结束，此线程随即终止。直接调用run()就是在主线程中执行了run()，没有启动新线程，是顺序执行

​	**run()方法中的异常不能抛出，只能try/catch**：因为父类中没有抛出任何异常，子类不能比父类抛出更多的异常；异常不能跨线程传播回main()中，因此必须在本地进行处理。

**<u>为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法</u>**

- new一个Thread，线程进入新建状态；
- 调用start()方法，会启动一个线程并使线程进入就绪状态，当分配到时间片后就可以开始运行了。
  - start()会执行线程的相应准备工作，然后自动执行run()方法的内容，这是真正的多线程工作。
  - 直接执行run()方法，会把run()方法当成一个main()线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。
- 不可以让已经start()的线程再次执行，如果重复执行，会报IllegalThreadStateException异常。
- start()方法底层其实是给CPU注册当前线程，并且触发run()方法执行；

**总结：调用start()方法才可以启动线程并使线程进入就绪状态，而run()方法只是thread的一个普通方法调用，还是在主线程里执行**

### 线程休眠sleep()、线程礼让yield()

**<u>（1）sleep()</u>**

**使用 `Thread.sleep` 方法可以使得当前正在执行的线程进入休眠状态。**

- 使用 `Thread.sleep` 需要向其传入一个整数值，这个值表示线程将要休眠的毫秒数。
- `Thread.sleep` 方法可能会抛出 `InterruptedException`，因为异常不能跨线程传播回 `main` 中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。

**<u>（2）yield</u>**

**调用yield会让当前线程从运行态进入就绪态，提示线程调度器让出当前线程对CPU的使用；**

- `Thread.yield` 方法的调用声明了当前线程已经完成了生命周期中最重要的部分，可以切换给其它线程来执行 。
- 该方法只是对线程调度器的一个建议，而且也只是建议具有相同优先级的其它线程可以运行。
- **会放弃CPU资源，但锁资源不会释放**

### interrupt()，中断线程

​	在某些情况下，我们在线程启动后发现并不需要它继续执行下去时，需要中断线程。目前在Java里还没有安全直接的方法来停止线程，但是Java提供了线程中断机制来处理需要中断线程的情况。

​	线程中断机制是一种协作机制。需要注意，通过中断操作并不能直接终止一个线程，而是通知需要被中断的线程自行处理。

- **public void interrupt()**：中断这个线程，异常处理机制，这里的中断线程并不会立即停止线程，而是设置线程的中断状态为true（默认是flase）
- **public static boolean interrupted()**：判断当前线程是否被打断，打断返回true，清除打断标记，连续调用两次一定返回false。
- **public boolean isInterrupted()**：判断当前线程是否被打断，不清除打断标记。

​	打断的线程会发生上下文切换，操作系统会保存线程信息，抢占到CPU后会从中断的地方接着运行（打断不是停止）。在线程中断机制里，当其他线程通知需要被中断的线程后，线程中断的状态被设置为true，但是具体被要求中断的线程要怎么处理，完全由被中断线程自己而定，可以在合适的实际处理中断请求，也可以完全不处理继续执行下去

### Thread.stop()，终止线程

![image-20250310231414020](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310231414020.png)

### 守护线程（Daemon）、用户线程

**<u>常见的守护线程</u>**

- 垃圾回收器线程就是一种守护线程，Finalizer线程也是守护线程；
- Tomcat中的Acceptor和poller线程都是守护线程，所以Tomcat接收到shutdown命令后，不会等待它们处理完当前请求。

**<u>什么是守护线程？</u>**

- **守护线程（Daemon Thread）是在后台执行并且不会阻止 JVM 终止的线程**。**当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程**。
- <font color = '#8D0101'>与守护线程（Daemon Thread）相反的，叫用户线程（User Thread），也就是非守护线程</font>。

**<u>为什么需要守护线程？</u>**

- 守护线程的优先级比较低，用于为系统中的其它对象和线程提供服务。典型的应用就是垃圾回收器。

**<u>如何使用守护线程？</u>**

- 可以使用 `isDaemon` 方法判断线程是否为守护线程。
- 可以使用`setDaemon`方法设置线程为守护线程。
  - 正在运行的用户线程无法设置为守护线程，**所以 `setDaemon` 必须在 `thread.start` 方法之前设置，否则会抛出 `llegalThreadStateException` 异常**；
  - 一个守护线程创建的子线程依然是守护线程。
  - 不要认为所有的应用都可以分配给守护线程来进行服务，比如读写操作或者计算逻辑。

```java
public class ThreadDaemonDemo {

    public static void main(String[] args) {
        Thread t = new Thread(new MyThread(), "线程");
        t.setDaemon(true); // 此线程在后台运行
        System.out.println("线程 t 是否是守护进程：" + t.isDaemon());
        t.start(); // 启动线程
    }

    static class MyThread implements Runnable {

        @Override
        public void run() {
            while (true) {
                System.out.println(Thread.currentThread().getName() + "在运行。");
            }
        }
    }
}
```

## 进程通信方式(复习线程间的通信方式)

- **管道/匿名管道(Pipes)**：用于具有亲缘关系的父子进程间或者兄弟进程之间的通信。

- **有名管道(Names Pipes)** :匿名管道由于没有名字，只能用于亲缘关系的进程间通信。为了克服这个缺点，提出了有名管道。有名管道严格遵循先进先出(first in first out)。有名管道以磁盘文件的方式存在，可以实现本机任意两个进程通信。
- **信号(Signal)**：信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生
- **消息队列(Message Queuing)**：消息队列是消息的链表,具有特定的格式,存放在内存中并由消息队列标识符标识。管道和消息队列的通信数据都是先进先出的原则。与管道（无名管道：只存在于内存中的文件；命名管道：存在于实际的磁盘介质或者文件系统）不同的是消息队列存放在内核中，只有在内核重启(即，操作系统重启)或者显式地删除一个消息队列时，该消息队列才会被真正的删除。消息队列可以实现消息的随机查询,消息不一定要以先进先出的次序读取,也可以按消息的类型读取.比 FIFO 更有优势。消息队列克服了信号承载信息量少，管道只能承载无格式字节流以及缓冲区大小受限等缺点。
- **信号量(Semaphores)**：信号量是一个计数器，用于多进程对共享数据的访问，信号量的意图在于进程间同步。这种通信方式主要用于解决与同步相关的问题并避免竞争条件
- **共享内存(Shared memory)**：使得多个进程可以访问同一块内存空间，不同进程可以及时看到对方进程中对共享内存中数据的更新。这种方式需要依靠某种同步操作，如互斥锁和信号量等。可以说这是最有用的进程间通信方式。
- **套接字(Sockets)**:此方法主要用于在客户端和服务器之间通过网络进行通信。套接字是支持 TCP/IP 的网络通信的基本操作单元，可以看做是不同主机之间的进程进行双向通信的端点，简单的说就是通信的两方的一种约定，用套接字中的相关函数来完成通信过程

## 线程通信方式、调度方式

### volatile关键字

​	基于volatile关键字来实现线程间相互通信是使用共享内存的思想，大致意思就是多个线程同时监听一个变量，当这个变量发生变化的时候，线程能够感知并执行相应的业务

==详细参考后续volatile==

### 等待通知机制wait/notify/notifyAll

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310233338783.png" alt="image-20250310233338783" style="zoom:80%;" />

- `wait` - `wait` 会自动**释放当前线程占有的对象锁**，并请求操作系统挂起当前线程，**让线程从 `Running` 状态转入 `Waiting` 状态**，等待 `notify` / `notifyAll` 来唤醒。如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 `notify` 或者 `notifyAll` 来唤醒挂起的线程，造成死锁。
- `notify` - 唤醒一个正在 `Waiting` 状态的线程，并让它拿到对象锁，具体唤醒哪一个线程由 JVM 控制 
- `notifyAll` - 唤醒所有正在 `Waiting` 状态的线程，接下来它们需要竞争对象锁。

> [!CAUTION]
>
> - **`wait`、`notify`、`notifyAll` 都是 `Object` 类中的方法**，而非 `Thread`。
> - **`wait`、`notify`、`notifyAll` 只能用在 `synchronized` 方法或者 `synchronized` 代码块中使用，否则会在运行时抛出 `IllegalMonitorStateException`**。
> - **wait()，notify()，notifyAll()三个方法的调用者必须是同步代码块或同步方法中的<font color = '#8D0101'>同步监视器</font>。**

**<u>为什么 `wait`、`notify`、`notifyAll` 不定义在 `Thread` 中？为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？</u>**

首先，需要了解几个基本知识点：

- 每一个 Java 对象都有一个与之对应的 **监视器（monitor）**
- 每一个监视器里面都有一个 **对象锁** 、一个 **等待队列**、一个 **同步队列**

了解了以上概念，我们回过头来理解前面两个问题。

**<u>为什么这几个方法不定义在 `Thread` 中？</u>**

- 由于**每个对象都拥有对象锁**，让当前线程等待某个对象锁，自然应该基于这个对象（`Object`）来操作，而非使用当前线程（`Thread`）来操作。因为当前线程可能会等待多个线程的锁，如果基于线程（`Thread`）来操作，就非常复杂了。

**<u>为什么 `wait`、`notify`、`notifyAll` 要配合 `synchronized` 使用？</u>**

- 如果调用某个对象的 `wait` 方法，当前线程必须拥有这个对象的对象锁，因此调用 `wait` 方法必须在 `synchronized` 方法和 `synchronized` 代码块中，如果没有获取到锁前调用wait()方法，会抛出java.lang.IllegalMonitorStateException异常。

**<u>典型案例：生产者消费者模式</u>**

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

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310234443010.png" alt="image-20250310234443010" style="zoom:60%;" />

### join

​	**`public final void join()`**：在线程操作中，可以使用 `join` 方法让一个线程强制运行，线程强制运行期间，其他线程无法运行，必须等待此线程完成之后才可以继续执行。

- `join` 方法会 **让线程从 `Running` 状态转入 `Waiting` 状态**。
- **join方法是被synchronized修饰的，本质上是一个对象锁**

**线程同步**：

- join实现线程同步，因为会阻塞等待另一个线程的结束，才能继续向下运行（需要外部共享变量，不符合面向对象封装的思想；必须等待线程结束，不能配合线程池使用）；
- Future实现（同步）：get()方法阻塞等待执行结果（main线程接收结果；get方法是让调用线程同步等待）

```java
public class ThreadJoinDemo {

    public static void main(String[] args) {
        MyThread mt = new MyThread(); // 实例化Runnable子类对象
        Thread t = new Thread(mt, "mythread"); // 实例化Thread对象
        t.start(); // 启动线程
        for (int i = 0; i < 50; i++) {
            if (i > 10) {
                try {
                    t.join(); // 线程强制运行
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("Main 线程运行 --> " + i);
        }
    }
    static class MyThread implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < 50; i++) {
                System.out.println(Thread.currentThread().getName() + " 运行，i = " + i); // 取得当前线程的名字
            }
        }
    }
}
```



### sleep()方法和wait()方法对比

**<u>（1）相同点：</u>**

- wait()、wait(long)和sleep(long)的效果都是让当前线程暂时放弃CPU的使用权，进入阻塞状态。

<u>**（2）不同点：**</u>

- **两个方法声明的位置不同**：Thread类中声明sleep(),是线程用来控制自身流程的，使此线程暂停执行一段时间而把执行机会让给其他线程；Object类中声明wait()，用于线程间通信。
- **对锁(同步监视器)的处理机制不同：**
  - 调用**sleep()**方法的过程中，线程**<font color = '#8D0101'>不会释放对象锁</font>**；
  - 调用**wait()**方法的时候，线程会**<font color = '#8D0101'>放弃对象锁，进入等待此对象的等待锁定池</font>**（如果不释放锁的话其他线程怎么抢占到锁从而执行唤醒操作呢？）

- **调用的要求不同**：<font color = '#8D0101'>sleep()可以在任何需要的场景下调用。wait()必须使用在同步代码块或同步方法中</font>
- **醒来时机不同**：
  - 执行sleep(long)和wait(long)的线程都会在等待相应毫秒后醒来；
  - wait(long)和wait()还可以被notify、notifyAll()唤醒，wait()如果不唤醒就一直等下去；
  - sleep()方法执行完成后，线程会自动苏醒
- **wait()通常被用于线程间交互/通信，sleep()通常被用于暂停执行**

​	**虚假唤醒**：notify只能随机唤醒一个WaitSet中的线程，这时如果有其他线程也在等待，那么就可能唤醒不了正确的线程。解决方法：采用notifyAll。

### sleep、yield、join 方法有什么区别

- **yield方法**
  - `yield` 方法会 **让线程从 `Running` 状态转入 `Runnable` 状态**。
  - 当调用了 `yield` 方法后，只有**与当前线程相同或更高优先级的`Runnable` 状态线程才会获得执行的机会**。
- **sleep方法**
  - `sleep` 方法会 **让线程从 `Running` 状态转入 `Waiting` 状态**。
  - `sleep` 方法需要指定等待的时间，**超过等待时间后，JVM 会将线程从 `Waiting` 状态转入 `Runnable` 状态**。
  - 当调用了 `sleep` 方法后，**无论什么优先级的线程都可以得到执行机会**。
  - `sleep` 方法不会释放“锁标志”，也就是说如果有 `synchronized` 同步块，其他线程仍然不能访问共享数据。
- **join方法**
  - `join` 方法会 **让线程从 `Running` 状态转入 `Waiting` 状态**。
  - 当调用了 `join` 方法后，**当前线程必须等待调用 `join` 方法的线程结束后才能继续执行**。

## Java并发核心机制

==Java 对于并发的支持主要汇聚在 `java.util.concurrent`，即 J.U.C。而 J.U.C 的核心是 `AQS`。==

### J.U.C 简介

​	Java 的 `java.util.concurrent` 包（简称 J.U.C）中提供了大量并发工具类，是 Java 并发能力的主要体现（注意，不是全部，有部分并发能力的支持在其他包中）。从功能上，大致可以分为：

- 原子类 - 如：`AtomicInteger`、`AtomicIntegerArray`、`AtomicReference`、`AtomicStampedReference` 等。
- 锁 - 如：`ReentrantLock`、`ReentrantReadWriteLock` 等。
- 并发容器 - 如：`ConcurrentHashMap`、`CopyOnWriteArrayList`、`CopyOnWriteArraySet` 等。
- 阻塞队列 - 如：`ArrayBlockingQueue`、`LinkedBlockingQueue` 等。
- 非阻塞队列 - 如： `ConcurrentLinkedQueue` 、`LinkedTransferQueue` 等。
- `Executor` 框架（线程池）- 如：`ThreadPoolExecutor`、`Executors` 等。

**<u>java并发类架构</u>**

- J.U.C 包中的工具类是基于 `synchronized`、`volatile`、`CAS`、`ThreadLocal` 这样的并发核心机制打造的。所以，要想深入理解 J.U.C 工具类的特性、为什么具有这样那样的特性，就必须先理解这些核心机制

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/java-concurrent-basic-mechanism.png)

### synchronized关键字

#### synchronized概述

- `synchronized` 是 Java 中的关键字，是 **利用锁的机制来实现互斥同步的**，解决的是多个线程之间访问资源的同步性。
- **`synchronized` 可以保证在同一个时刻，只有一个线程可以执行某个方法或者某个代码块**。
  - 即采用互斥的方式让**同一时刻至多只有一个线程能持有对象锁**，其他线程获取这个对象锁时会阻塞，保证拥有锁的线程可以安全地执行临界区内的代码，不用担心线程上下文切换。

> [!NOTE]
>
> 另外，在Java早期版本中，synchronized属于重量级锁，效率低下。因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，**Java的线程是映射到操作系统的原生线程之上的**。如果需要挂起或唤醒一个线程，都需要操作系统帮忙完成，而**操作系统实现线程之间的切换时需要从用户态转换到内核态**，这个状态之间的转换需要相对比较长的时间，这也是为什么早期的synchronized效率低的原因

#### `synchronized` 的3 种应用方式

- **同步实例方法** - 对于普通同步方法，锁是当前实例对象
- **同步静态方法** - 对于静态同步方法，锁是当前类的 `Class` 对象
- **同步代码块** - 对于同步方法块，锁是 `synchonized` 括号里配置的对象

**<u>（1）修饰实例方法</u>**

作用于**当前对象实例加锁**，进入同步代码前要获得当前对象实例的锁。

- **synchronized修饰的方法不具备继承性**，所以子类是线程不安全的，如果子类的方法也被synchronized修饰，两个锁对象其实是一把锁，而且是子类作为锁对象

同步方法**默认用this作为锁的对象**：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311000127565.png" alt="image-20250311000127565" style="zoom:60%;" />

**<u>（2）修饰静态方法</u>**:

​	也就是**给当前类加锁**，会作用于类的所有对象实例，进入同步代码前要获得当前class的锁。因为静态成员不属于任何一个实例对象，是类成员（static表明这是该类的一个静态资源，不管new了多少个对象，只有一份）。

> [!IMPORTANT]
>
> - 如果一个线程A调用一个实例对象的非静态synchronized方法，而线程B需要调用这个实例对象所属类的静态synchronized方法，是允许的，不会发生互斥现象，因为访问静态synchronized方法占用的锁是当前类的锁，而访问非静态synchronized方法占用的锁是当前实例对象锁。

**同步方法默认用“类名.class”作为锁的对象**：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311003407989.png" alt="image-20250311003407989" style="zoom:60%;" />                               

**<u>（3）修饰代码块</u>**：

**指定加锁对象，对给定对象/类加锁**。

- **锁对象：理论上可以是任意的唯一对象；**
- synchronized是可重入、不公平的重量级锁；

> [!NOTE]
>
> 原则上：锁对象建议使用共享资源（即多个线程必须要共用同一把锁）、在非静态方法中使用this作为锁对象，锁住的this正好是共享资源、在静态方法中使用“类名.class”字节码作为锁对象，因为静态成员属于类，被所有实例对象共享，所以需要锁住类

- synchronized(this|object)表示进入同步代码库前要获得给定对象的锁。
- synchronized(类.class)表示进入同步代码前要获得当前class的锁

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311003622888.png" alt="image-20250311003622888" style="zoom:67%;" />

**==注意：==**

- ==一般可以使用一个普通的object实例来作为同步锁的监视器（Monitor）==
  - 明确性： 专为同步而创建的 Object 实例（如 REGISTER_LOCK）的唯一作用就是管理线程同步，代码意图更清晰，避免误用其他对象（如业务对象）作为锁的风险。
  - 安全性： 若使用其他对象（例如 this 或某个业务对象）作为锁，可能因外部代码持有该锁而导致死锁或意外行为。专用锁对象完全由当前类控制，无副作用。
  - 轻量化： Object 实例是最轻量的锁载体，无需额外资源开销。

```java
final private Object REGISTER_LOCK = new Object();

synchronized (REGISTER_LOCK) {
                    StreamRegister info = streamisRegisterService.getInfoByApplicationName(applicationName);
                    if (info != null) {
                        streamisRegisterService.deleteRegister(applicationName);
                    }
                    streamisRegisterService.addStreamRegister(streamRegister);
                }
```

**<u>==synchronized(this) vs synchronized(object) 的区别==</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311004343268.png" alt="image-20250311004343268" style="zoom:80%;" />

#### synchronized底层原理

synchronized关键字底层原理属于JVM层面，两者的本质都是对**对象监视器`monitor`的获取**，**`Monitor` 对象是同步的基本实现单元**

**<u>（1）同步代码块</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311113518475.png" alt="image-20250311113518475" style="zoom:60%;" />

- **synchronized同步语句块**使用的是**<font color = '#8D0101'>monitorenter和monitorexit指令</font>**，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置。
- 当执行monitorenter指令时，线程试图获取锁也就是获取monitor（<font color = '#8D0101'>monitor对象存在于每个Java对象的**对象头**中</font>，synchronized锁便是通过这种方式获取锁的，这也是为什么Java中任意对象可以作为锁的原因）的持有权。当计数器为0则可以成功获取，获取后将锁计数器设为1即加1。相应地，在执行monitorexit指令后，将锁计数器设为0，表明锁被释放，如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

<u>**（2）synchronized修饰方法的情况**</u>

- synchronized修饰的方法并没有monitorenter指令和monitorexit指令，**而是用ACC_SYNCHRONIZED标识，该标识指明该方法是一个同步方法**
- JVM通过该标识来辨别一个方法是否声明为同步方法，当方法调用时，调用指令将会检查该方法是否被设置 `ACC_SYNCHRONIZED` 访问标志。如果设置了该标志，执行线程将先持有 `Monitor` 对象，然后再执行方法。
- 在该方法运行期间，其它线程将无法获取到该 `Mointor` 对象，当方法执行完成后，再释放该 `Monitor` 对象。

#### 构造方法可以使用synchronized关键字修饰么

构造方法不能使用synchronized关键字修饰。

<font color = '#8D0101'>构造方法本身就属于线程安全的</font>，不存在同步的构造方法一说

#### jdk1.6之后的synchronized优化（锁升级）

Java 1.6 引入了偏向锁和轻量级锁，从而让 `synchronized` 拥有了四个状态：

- **无锁状态（unlocked）**
- **偏向锁状态（biasble）**
- **轻量级锁状态（lightweight locked）**
- **重量级锁状态（inflated）**

​	当 JVM 检测到不同的竞争状况时，会自动进行锁升级切换到适合的锁实现。**<font color = '#8D0101'>注意锁可以升级不可降级</font>**，这种策略是为了提高获得锁和释放锁的效率。

**<u>锁升级的基本流程</u>**，==具体看锁章节==

- **当没有竞争出现时，默认会使用偏向锁**。JVM 会利用 CAS 操作（compare and swap），在对象头上的 Mark Word 部分设置线程 ID，以表示这个对象偏向于当前线程，所以并不涉及真正的互斥锁。这样做的假设是基于在很多应用场景中，大部分对象生命周期中最多会被一个线程锁定，使用偏斜锁可以降低无竞争开销。
- 如果有另外的线程试图锁定某个已经被偏斜过的对象，JVM 就需要撤销（revoke）偏向锁，并切换到轻量级锁实现。轻量级锁依赖 CAS 操作 Mark Word 来试图获取锁，如果重试成功，就使用普通的轻量级锁；否则，进一步升级为重量级锁。

### volatile

#### volatile 的要点

**`volatile` 是轻量级的 `synchronized`，它在多处理器开发中保证了共享变量的“可见性”。**

被 `volatile` 修饰的变量，具备以下特性：

- **线程可见性** - 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个共享变量，另外一个线程能读到这个修改的值。
- **禁止指令重排序**
- **不保证原子性**

我们知道，**==线程安全需要具备：可见性、原子性、顺序性==**。`volatile` 不保证原子性，所以决定了它不能彻底地保证线程安全。

#### volatile的内存语义(内存可见，禁止重排)（读写屏障）

​	Java内存模型抽象了线程和主内存之间的关系，就比如说线程之间的共享变量必须存储在主内存中。Java内存模型主要目的是为了屏蔽系统和硬件的差异，避免一套代码在不同的平台下产生的效果不一致。

​	在JDK1.2之前，Java的内存模型实现总是从主存(即共享内存）读取变量，是不需要进行特别的注意的。而在当前的Java内存模型下，**线程可以把变量保存本地内存（比如机器的寄存器)中，而不是直接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值的拷贝，造成数据的不一致。**

- 要解决这个问题，就**需要把变量声明为volatile**，这就指示JVM，**这个变量是共享的，每次使用它都到主存中进行读取**。这就是保证变量的可见性
- volatile关键字还有一个重要作用就是**防止JVM的指令重排**

**而如果变量是全局不变的，就不需要每次去主存中读取**

**<u>volatile的两个功能、一个不能</u>**

- 保证变量的内存可见性
- 禁止volatile**变量与普通变量重排序**（JSR133提出，Java 5开始才有这个“增强的volatile内存语义”）
- 不保证原子性

==**<u>eg：</u>**==

```java
public class VolatileExample 
		int a = 0;
		volatile boolean flag = false;

		public void writer() {
				a = 1; // step 1
				flag = true; // step 2
		}

		public void reader) i
				if (flag) { // step 3
				System.out.println(a); // step 4
		}
}
```

**<u>（1）保证变量的内存可见性</u>**

在这段代码里，我们使用volatile关键字修饰了一个boolean类型的变量flag。

所谓**内存可见性**，指的是

- 当一个线程对volatile修饰的变量进行写操作（比如step2）时，JMM会立即把该线程对应的本地内存中的共享变量的值刷新到主内存；
- 当一个线程对volatile修饰的变量进行读操作（比如step3）时，JMM会把立即该线程对应的本地内存置为无效，从主内存中读取共享变量的值。
- <font color = '#8D0101'>在这一点上，volatile与锁具有相同的内存效果，volatile变量的写和锁的释放具有相同的内存语义，volatile变量的读和锁的获取具有相同的内存语义</font>。

​	如果flag变量没有用volatile修饰，在step2，线程A的本地内存里面的变量就不会立即更新到主内存，那随后线程B也同样不会去主内存拿最新的值，仍然使用线程B本地内存缓存的变量的值a = 0，flag = false。

**<u>（2）禁止volatile变量与普通变量重排序</u>**

在JSR-133之前的旧的Java内存模型中，是允许volatile变量与普通变量重排序的。那上面的案例中，可能就会被重排序成下列时序来执行

- 线程A写volatile变量，step 2，设置flag为true；
- 线程B读同一个volatile，step 3，读取到flag为true；
- 线程B读普通变量，step 4，读取到a = 0；
- 线程A修改普通变量，step 1，设置a = 1

可见，如果volatile变量与普通变量发生了重排序，也可能导致普通变量读取错误

<u>**（3）JVM是怎么还能限制处理器的重排序的？内存屏障**</u>

==**内存屏障：**==

**编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。**硬件层面，内存屏障分两种：<font color = '#8D0101'>读屏障（Load Barrier）和写屏障（Store Barrier）</font>。内存屏障有两个作用：（包含了内存可见性和指令重排序）

**<u>1） 强制把写缓冲区/高速缓存中的脏数据等写回主内存，或者让缓存中相应的数据失效</u>**

- 写屏障（sfence，Store Barrier）保证在**<font color = '#8D0101'>写屏障之前</font>**的对共享变量的改动，**都同步到主存当中**（**将volatile修饰的变量ready及其之前的变量修改都会同步到主存当中**）

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311123219754.png" alt="image-20250311123219754" style="zoom:67%;" />

- 读屏障（lfence，Load Barrier）保证在**<font color = '#8D0101'>读屏障之后的对共享变量的读取，从主存刷新变量值</font>**，加载的是主存中最新数据（**将volatile修饰的变量ready及其之后的变量读取操作读取的是主存中最新的数据**）

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311124448837.png" alt="image-20250311124448837" style="zoom: 67%;" />

**<u>2） 阻止屏障两侧的指令重排序；</u>**

- 写屏障会确保指令重排序时，不会将写屏障之前的代码排在写屏障之后（num = 2与ready = true）
- 读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前

####  volatile 的应用（单例双端检验锁、双重校验锁）

**<u>（1）未使用volatile关键字</u>**

例如下面的代码：在获取实例 getInstance() 的方法中，我们首先判断 instance 是否为空，如果为空，则锁定 Singleton.class 并再次检查 instance 是否为空，如果还为空则创建 Singleton 的一个实例。

```java
public class Singleton {
  private static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) { //1.第一次检查
      synchronized(Singleton.class) {//2
        if (instance == null)//3.第二次检查
          instance = new Singleton();//4
        }
    }
    return instance;
  }
}
```

- 假设有两个线程 A、B 同时调用 getInstance() 方法，他们会同时发现 `instance == null` ，于是同时对 Singleton.class 加锁，此时 JVM 保证只有一个线程能够加锁成功（假设是线程 A），另外一个线程则会处于等待状态（假设是线程 B）；线程 A 会创建一个 Singleton 实例，之后释放锁，锁释放后，线程 B 被唤醒，线程 B 再次尝试加锁，此时是可以加锁成功的，加锁成功后，线程 B 检查 `instance == null` 时会发现，已经创建过 Singleton 实例了，所以线程 B 不会再创建一个 Singleton 实例。

但实际上这个 getInstance() 方法并不完美。问题出在哪里呢？出在 new 操作上，我们以为的 new 操作应该是：

1. 分配一块内存 M；
2. 在内存 M 上初始化 Singleton 对象；
3. 然后 M 的地址赋值给 instance 变量。

但是实际上优化后的执行路径却是这样的：

1. 分配一块内存 M；
2. 将 M 的地址赋值给 instance 变量；
3. 最后在内存 M 上初始化 Singleton 对象。

- 我们假设线程 A 先执行 getInstance() 方法，当执行完指令 2 时恰好发生了线程切换，切换到了线程 B 上；如果此时线程 B 也执行 getInstance() 方法，那么线程 B 在执行第一个判断时会发现 `instance != null` ，所以直接返回 instance，而此时的 instance 是没有初始化过的，如果我们这个时候访问 instance 的成员变量就可能触发空指针异常。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200701111050.png" alt="img" style="zoom:50%;" />

**<u>（2）使用volatile关键字</u>**

双重校验锁实现线程安全的单例模式

```java
class Singleton {
    private volatile static Singleton instance = null;

    private Singleton() {}

    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

#### synchronized无法禁止指令重排和处理器优化，为什么可以保证有序性可见性

- 加了锁之后，只能有一个线程获得到了锁，获得不到锁的线程就要阻塞，所以同一时间只有一个线程执行，**相当于单线程，由于数据与数据间有依赖性的存在，单线程的指令重排是没有问题的**；
- **线程加锁前，将清空工作内存中共享变量的值，使用共享变量时需要从主内存中重新读取最新的值；线程解锁前，必须把共享变量的最新值刷新到主内存中**。

#### synchronized和volatile的区别

在保证内存可见性这一点上，volatile有着与锁相同的内存语义，所以可以作为一个“轻量级”的锁来使用。在功能上，锁比volatile更强大；在性能上，volatile更有优势。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311125949123.png" alt="image-20250311125949123" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311130008324.png" alt="image-20250311130008324" style="zoom:67%;" />

### CAS（Compare And Swap）与原子类

#### CAS简介、要点

- **互斥同步最主要的问题是线程阻塞和唤醒所带来的性能问题**，因此互斥同步也被称为阻塞同步
- 我们可以使用基于冲突检测的乐观并发策略：先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地重试，直到成功为止）。这种乐观的并发策略的许多实现都不需要将线程阻塞，因此这种同步操作称为非阻塞同步。

​	**CAS（Compare and Swap），字面意思为比较并交换。CAS 有 3 个操作数，分别是：内存值 M，期望值 E，更新值 U。当且仅当内存值 M 和期望值 E 相等时，将内存值 M 修改为 U，否则什么都不做**。

**<u>（1）是CPU并发原语：</u>**

- **CAS并发原语体现在Java语言中就是sun.misc.Unsafe类的各个方法**，调用UnSafe类中的CAS方法，JVM会实现出CAS汇编指令，这是一种完全依赖于硬件的功能，实现了原子操作；
- **CAS是一种系统原语**，原语属于操作系统范畴，是由若干条指令组成，用于完成某个功能的一个过程，并且**原语的执行必须是连续的，执行过程中不允许被中断**，所以CAS是一条CPU的原子指令，不会造成数据不一致的问题，是线程安全的

**<u>（2）有三个值V、E、N</u>**

​	**当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作**。

- V：要更新的变量值(var)
- E：预期值(expected)
- N：新值(new)

​	判断V是否等于E，如果等于，将V的值设置为N；如果不等，说明已经有其它线程更新了V，则当前线程放弃更新，什么都不做。所以这里的**预期值E本质上指的是“旧值”**。

(即：**比较当前工作内存中的值和主物理内存中的值，如果相同则执行规定操作，否则继续比较直到主内存和工作内存的值一致为止**)

**eg：**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311145748100.png" alt="image-20250311145748100" style="zoom:60%;" />

<u>**（3）CAS特点**</u>

- 结合CAS和volatile可以实现无锁并发，适用于线程数少、多核CPU的场景；
- CAS体现的是无锁并发、无阻塞并发，线程**不会**陷入阻塞，线程不需要**频繁**切换状态（上下文切换，系统调用）(意思是还是会切换状态的)；
- CAS是基于乐观锁的思想

#### Java实现CAS的原理-Unsafe类

在Java中，如果一个方法是native的，那Java就不负责具体实现它，而是交给底层的JVM使用c或者c++去实现

在Java中，有一个Unsafe类，它在sun.misc包中。它里面是一些native方法，其中就有几个关于CAS的，他们都是public native的：

- Unsafe中对CAS的实现是C++写的，它的具体实现和操作系统、CPU都有关系

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311150117838.png" alt="image-20250311150117838" style="zoom:67%;" />

​	Unsafe类里面还有其它方法用于不同的用途。比如支持线程挂起和恢复的**park和unpark**，**LockSupport类底层就是调用了这两个方法**。还有支持反射操作的allocateInstance()方法。

#### CAS的缺点、问题（ABA问题）

- 循环时间长开销大
- 只能保证一个共享变量的原子性
- ABA 问题

**<u>（1）循环时间长，开销大：</u>**

因为执行的是循环操作，如果比较不成功一直在循环，最差的情况某个线程一直取到的值和预期值都不一样，就会无限循环导致饥饿。

**解决思路**：

- **使用CAS线程数不要超过CPU的核心数；**
- JVM支持处理器提供的pause指令：pause指令能让自旋失败时cpu睡眠一小段时间再继续自旋，从而使得读操作的频率低很多,为解决内存顺序冲突而导致的CPU流水线重排的代价也会小很多。

**<u>（2）只能保证一个共享变量的原子操作：</u>**

- 对于一个共享变量执行操作时，可以通过循环CAS的方式来保证原子操作；
- 对于多个共享变量操作时，循环CAS就无法保证操作的原子性

这个时候可以**用锁来保证原子性**、或者使用AtomicReference类保证对象之间的原子性，**把多个变量放到一个对象里面进行CAS操作**



**<u>==（3）ABA问题==</u>**

​	ABA问题核心原因：**<font color = '#8D0101'>值相等并不等价于数据没有被修改过</font>**

​	当进行获取主内存值时，该内存值在写入主内存时已经被修改了N次，但是最终又改成原来的值：**如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过**。

**<u>可利用<font color = '#8D0101'>版本号机制</font>解决：</u>**

- ABA问题的产生是因为变量的状态值产生了环形转换，就是从A到B，在由B到A。如果变量的值只能朝着一个方向转换，比如A到B，B到C，不构成环形，就不会存在问题。

- J.U.C 包提供了一个带有标记的**<font color = '#8D0101'>原子引用类 `AtomicStampedReference` </font>来解决这个问题**，它可以通过控制变量值的版本来保证 CAS 的正确性。

  - 它给每个变量的状态值都配备了一个**时间戳**，从而避免ABA问题的产生

  - **构造方法**：`public AtomicStampedReference(V initialRef, int initialStamp)`：初始值和初始版本号

    **compareAndSet方法**的作用是首先检查**当前引用是否等于预期引用，并且检查当前标志是否等于预期标志**，如果二者都相等，才使用CAS设置为新的值和标志

#### CAS应用（Atomic原子类）

​	Atomic原子操作类：**<font color = '#8D0101'>底层使用CAS机制</font>**，能保证操作真正的原子性,指一个操作时不可中断的，即使是多个线程一起执行的时候，一个操作一旦开始就不会被其他线程干扰

并发包java.util.concurrent的原子类都存放在java.util.concurrent.atomic。

##### JUC包中的原子类的类型

<u>**（1）使用原子的方式更新基本类型**</u>

- **<font color = '#8D0101'>AtomicInteger（整型原子类）</font>：**Atomiclnteger类主要利用CAS(compare and swap)+volatile和native方法来保证原子操作，从而避免synchronized的高开销,执行效率大为提升；
- AtomicLong（长整型原子类）；
- AtomicBoolean：布尔型原子类

**AtomicInteger类常用方法**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311153436247.png" alt="image-20250311153436247" style="zoom:50%;" />

**<u>（2）原子数组：使用原子的方式更新数组里的某个元素</u>**

- AtomicIntegerArray：整型数组原子类；
- AtomicLongArray：长整型数组原子类；
- AtomicReferenceArray：引用类型数组原子类

**AtomicIntegerArray类方法**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311153621775.png" alt="image-20250311153621775" style="zoom:50%;" />                               

**<u>（3）原子引用：对Object进行原子操作，提供一种读和写都是原子性的对象引用变量：</u>**

- AtomicReference：引用类型原子类；
- AtomicMarkableReference：原子更新带有标记的引用类型，该类将boolean标记与引用关联起来；
- **<font color = '#8D0101'>AtomicStampedReference</font>**：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以**解决使用CAS进行原子更新时可能出现的ABA问题**

##### AtomicInteger类（解决a++问题）

AtomicUnteger类主要利用**CAS+volatile和native方法来保证原子操作**，从而避免synchronized的高开销，执行效率大为提升。

==**<u>（1）a++问题</u>**==

- 如果 `i++` 执行前， i 的值被修改了怎么办？还能得到预期值吗？出现该问题的原因是在并发环境下，以上代码片段不是原子操作，随时可能被其他线程所篡改。

```java
if(i==b) {
    i++;
}
```

解决这种问题的最经典方式是应用原子类的 `incrementAndGet` 方法。

==**<u>（2）incrementAndGet()方法</u>**==

**以原子方式将当前值加一，<font color = '#8D0101'>返回自增后的值，相当于i++</font>**

> [!WARNING]
>
> (注意跟**getAndIncrement()**区分：以原子方式将当前值加一，**返回自增前的值**)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311155255900.png" alt="image-20250311155255900" style="zoom:60%;" />

我们来分析下incrementAndGet的逻辑：

- 先获取当前的value值
- 对value+1
- 第三步是关键步骤，**调用compareAndSet方法来来进行原子更新操作**，这个方法的语义是： 先检查当前value是否等于current，如果相等，则意味着value没被其他线程修改过，更新并返回true。如果不相等，compareAndSet则会返回false，然后循环继续尝试更新。

**compareAndSet调用了<font color = '#8D0101'>Unsafe类的compareAndSwaplnt</font>方法**

- **compareAndSet()：**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311155707505.png" alt="image-20250311155707505" style="zoom:67%;" />

- **Unsafe类的compareAndSwapInt()**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311155714657.png" alt="image-20250311155714657" style="zoom:67%;" />

#### 为什么无锁效率高

- 无锁情况下，即使重试失败，线程始终在高速运行，没有停歇，而synchronized会让线程在没有获得锁的时候，发生上下文切换，进入阻塞。打个比喻，线程就好像高速跑道上的赛车，高速运行时，速度超快，一旦发生上下文切换，就好比赛车要减速、熄火，等被唤醒又得重新打火、启动、加速恢复到高速运行，代价比较大。
- 但无锁情况下，**因为线程要保持运行，需要额外CPU支持**，CPU在这里就好比高速跑道，没有额外的跑道，线程想要高速运行也无从谈起，**虽然不会进入阻塞，但由于没有分到时间片，仍然会进入可运行状态，还是会导致上下文切换**。

#### CAS和synchronized(乐观锁和悲观锁对比)

- **synchronized是从悲观的角度出发**：总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程），因此synchronized也称之为悲观锁，ReentrantLock也是一种悲观锁，性能较差
- **CAS是从乐观的角度出发**：总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据。如果别人修改过，则获取现在最新的值，如果别人没修改过，直接修改共享数据的值，CAS这种机制也称之为乐观锁，综合性能较好。

​	CAS体现了无锁并发、无阻塞并发的思想：①没有使用synchronized，所以线程不会陷入阻塞，这也是效率提升的因素之一；②如果竞争激烈，失败重试必然频繁发生，效率会受到影响，所以适用于线程数少、多核CPU的场景

### ThreadLocal

> [!IMPORTANT]
>
> **`ThreadLocal` 是一个存储线程本地副本的工具类**。
>
> 要保证线程安全，不一定非要进行同步。同步只是保证共享数据争用时的正确性，如果一个方法本来就不涉及共享数据，那么自然无须同步。
>
> Java 中的 **无同步方案** 有：
>
> - **可重入代码** - 也叫纯代码。如果一个方法，它的 **返回结果是可以预测的**，即只要输入了相同的数据，就能返回相同的结果，那它就满足可重入性，当然也是线程安全的。
> - **线程本地存储** - 使用 **`ThreadLocal` 为共享变量在每个线程中都创建了一个本地副本**，这个副本只能被当前线程访问，其他线程无法访问，那么自然是线程安全的。

​	通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。如果想实现每一个线程都有自己的专属本地变量该如何解决呢?JDK中提供的ThreadLocal类正是为了解决这样的问题。**ThreadLocal类主要解决的就是让每个线程绑定自己的值**，可以将ThreadLocal类形象的比喻成存放数据的盒子，盒子中可以**存储每个线程的私有数据**。

​	如果创建了一个ThreadLocal变量，那么**访问这个变量的每个线程都会有这个变量的本地副本**，这也是ThreadLocal变量名的由来。**他们可以使用get()和set()方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题**。

​	ThreadLocal对象可以提供**线程局部变量**，每个线程Thread拥有一份自己的副本变量，多个线程互不干扰。

​	**作用**：1、ThreadLocal可以实现资源对象的线程隔离，让每个线程各用各的资源对象，避免争用引发的线程安全问题；2、ThreadLocal同时实现了线程内的资源共享。

#### ThreadLocal应用

`ThreadLocal` 的方法：

```java
public class ThreadLocal<T> {
    public T get() {}
    public void set(T value) {}
    public void remove() {}
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {}
}
```

> 说明：
>
> - `get` - 用于获取 `ThreadLocal` 在当前线程中保存的变量副本。
> - `set` - 用于设置当前线程中变量的副本。
> - `remove` - 用于删除当前线程中变量的副本。如果此线程局部变量随后被当前线程读取，则其值将通过调用其 `initialValue` 方法重新初始化，除非其值由中间线程中的当前线程设置。 这可能会导致当前线程中多次调用 `initialValue` 方法。
> - `initialValue` - 为 ThreadLocal 设置默认的 `get` 初始值，需要重写 `initialValue` 方法 。

`ThreadLocal` 常用于防止对可变的单例（Singleton）变量或全局变量进行共享。典型应用场景有：管理数据库连接、Session。

<u>**【示例】数据库连接**</u>

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    @Override
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};

public static Connection getConnection() {
    return connectionHolder.get();
}
```

**<u>【示例】Session 管理</u>**

```java
private static final ThreadLocal<Session> sessionHolder = new ThreadLocal<>();

public static Session getSession() {
    Session session = (Session) sessionHolder.get();
    try {
        if (session == null) {
            session = createSession();
            sessionHolder.set(session);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return session;
}
```

**<u>【示例】完整使用 `ThreadLocal` 示例</u>**

```java
//全部输出 count = 10
public class ThreadLocalDemo {

    private static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(new MyThread());
        }
        executorService.shutdown();
    }

    static class MyThread implements Runnable {

        @Override
        public void run() {
            int count = threadLocal.get();
            for (int i = 0; i < 10; i++) {
                try {
                    count++;
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            threadLocal.set(count);
            threadLocal.remove();
            System.out.println(Thread.currentThread().getName() + " : " + count);
        }
    }
}
```

#### ThreadLocal原理

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311164736667.png" alt="image-20250311164736667" style="zoom:67%;" />

**<u>（1）Thread类中的`ThreadLocal.ThreadLocalMap`</u>**

```java
public class Thread implements Runnable { 
  	//......
		//与此线程有关的ThreadLocal值。由ThreadLocal类维护
  	ThreadLocal. ThreadLocalMap threadLocals = null;

		//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocaL类维护
  	ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
  	//....
}
```

- **`Thread` 类中维护着一个 `ThreadLocal.ThreadLocalMap` 类型的成员** `threadLocals`。这个成员就是用来存储当前线程独占的变量副本。
  - 默认情况下这个变量是 null，**只有当前线程调用 ThreadLocal 类的 set 或 get 方法时才创建它们，实际上调用这两个方法的时候，我们调用的是 ThreadLocalMap 类对应的 get（）、set（）方法**。

**<u>（2）ThreadLocalMap</u>**

```java
static class ThreadLocalMap {
    // ...
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    // ...
}
```

​	最终的变量是放在了当前线程ThreadLocalMap中，并不是存在ThreadLocal上，**ThreadLocal可以理解为只是ThreadLocalMap的封装，传递了变量值**。ThrealLocal类中可以通过Thread.currentThread()获取到当前线程对象后，直接通过getMap(Thread t)可以访问到该线程的ThreadLocalMap对象。

​	`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类，它维护着一个 `Entry` 数组，**`Entry` 继承了 `WeakReference`** ，所以是弱引用。 `Entry` 用于保存键值对，其中：

- **`key` 是 `ThreadLocal` 对象（实际上不是ThreadLocal对象本身而是它的一个弱引用）；**
- **`value` 是传递进来的对象（变量副本）。**

> [!IMPORTANT]
>
> - 调用set方法，就是以ThreadLocal自己作为key，资源对象作为value，放入当前线程的ThreadLocalMap集合中；
> - 调用get方法，就是以ThreadLocal自己作为key，到当前线程中查找关联的资源值；
> - 调用remove方法，就是以ThreadLocal自己作为key，移除当前线程关联的资源值

#### ThreadLocal的数据结构

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311170728515.png" alt="image-20250311170728515" style="zoom:65%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311170738982.png" alt="image-20250311170738982" style="zoom:67%;" />

#### ThreadLocal内存泄露问题

```java
static class ThreadLocalMap {
    // ...
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    // ...
}
```

- `ThreadLocalMap` 的 `Entry` 继承了 `WeakReference`，所以它的<font color = '#8D0101'> **key （`ThreadLocal` 对象）是弱引用，而 value （变量副本）是强引用**</font>。
  - 如果 `ThreadLocal` 对象没有外部强引用来引用它，那么 `ThreadLocal` 对象会在下次 GC 时被回收。
  - 此时，`Entry` 中的 key 已经被回收，但是 value 由于是强引用不会被垃圾收集器回收。如果创建 `ThreadLocal` 的线程一直持续运行，那么 value 就会一直得不到回收，产生**内存泄露**。

​	**解决方案**：ThreadLocalMap实现中已经考虑了这种情况，在调用set()、get()、remove()方法的时候，会清理掉key为null的记录。**使用 `ThreadLocal` 的 `set` 方法后，显示的调用 `remove` 方法** 。

```java
ThreadLocal<String> threadLocal = new ThreadLocal();
try {
    threadLocal.set("xxx");
    // ...
} finally {
    threadLocal.remove();
}
```

#### ThreadLocalMap Hash算法

​	`ThreadLocalMap` 虽然是类似 `Map` 结构的数据结构，但它并没有实现 `Map` 接口。它不支持 `Map` 接口中的 `next` 方法，这意味着 `ThreadLocalMap` 中解决 Hash 冲突的方式并非 **拉链表** 方式。

- 实际上，**`ThreadLocalMap` 采用线性探测的方式来解决 Hash 冲突**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311171416948.png" alt="image-20250311171416948" style="zoom:80%;" />

每当创建一个ThreadLocal对象，ThreadLocal.nextHashCode这个值就会增长 0x61c88647。

这个值很特殊，它是斐波那契数也叫黄金分割数。hash增量为这个数字，带来的好处就是hash分布非常均匀。

#### ThreadLocalRandom

**<u>Java.util.Random的做法</u>**

​	在多线程下，使用java.util.Random产生的实例来产生随机数是线程安全的，但深挖Random的实现过程，会发现多个线程会竞争同一seed而造成性能降低

Random生成新的随机数需要两步：

- 根据老的seed生成新的seed
- 由新的seed计算出新的随机数

​	其中，第二步的算法是固定的，如果每个线程并发地获取同样的seed，那么得到的随机数也是一样的。为了避免这种情况，Random使用CAS操作保证每次只有一个线程可以获取并更新seed，失败的线程则需要自旋重试。

**<u>ThreadLocalRandom的做法</u>**

​	在多线程下用Random不太合适，为了解决这个问题，出现了ThreadLocalRandom，**在多线程下，它为每个线程维护一个seed变量，**这样就不用竞争了。但ThreadLocalRandom在多线程下产生了相同的随机数，因此需要给每个线程初始化一个seed，那就需要调用ThreadLocalRandom.current()方法。**起始ThreadLocalRandom是对每个线程都设置了单独的随机数种子，这样就不会发生多线程同时更新一个数时产生的资源争抢了，用空间换时间**。

ThreadLocalRandom在高并发下的性能远远超过Random。

## Java锁

> [!TIP]
>
> 本文先阐述 Java 中各种锁的概念。
>
> 然后，介绍锁的核心实现 AQS。
>
> 然后，重点介绍 Lock 和 Condition 两个接口及其实现。并发编程有两个核心问题：同步和互斥。
>
> **互斥**，即同一时刻只允许一个线程访问共享资源；
>
> **同步**，即线程之间如何通信、协作。
>
> 这两大问题，管程（`sychronized`）都是能够解决的。**J.U.C 包还提供了 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题**。

### 锁的各种概念

#### 重入锁和非重入锁

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311214107154.png" alt="image-20250311214107154" style="zoom:80%;" />

**可重入锁可以在一定程度上避免死锁**。

- **`ReentrantLock` 、`ReentrantReadWriteLock` 是可重入锁**。这点，从其命名也不难看出。
- **`synchronized` 也是一个可重入锁**。

【示例】`synchronized` 的可重入示例

- 如果使用的锁不是可重入锁的话，`setB` 可能不会被当前线程执行，从而造成死锁。

```java
synchronized void setA() throws Exception{
    Thread.sleep(1000);
    setB();
}
synchronized void setB() throws Exception{
    Thread.sleep(1000);
}
```

#### 公平锁与非公平锁

- **公平锁** - 公平锁是指 **多线程按照申请锁的顺序来获取锁**。
- **非公平锁** - 非公平锁是指 **多线程不按照申请锁的顺序来获取锁** 。这就可能会出现优先级反转（后来者居上）或者饥饿现象（某线程总是抢不过别的线程，导致始终无法执行）。

> [!CAUTION]
>
> - ==已经处在阻塞队列中的线程（不考虑超时）始终都是公平的，先进先出==；
> - 公平锁是指未处于阻塞队列中的线程来争抢锁，如果队列不为空，则老实到队尾等待；
> - 非公平锁是指未处于阻塞队列中的线程来争抢锁，与队列头唤醒的线程去竞争，谁抢到算谁的。

**公平锁为了保证线程申请顺序，势必要付出一定的性能代价，因此其吞吐量一般低于非公平锁。**

公平锁与非公平锁 在 Java 中的典型实现：

- **`synchronized` 只支持非公平锁**。
- **`ReentrantLock` 、`ReentrantReadWriteLock`，默认是非公平锁，但支持公平锁**。

![image-20250311214246480](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311214246480.png)

#### 独享锁与共享锁

独享锁与共享锁是一种广义上的说法，从实际用途上来看，也常被称为互斥锁与读写锁。

- **独享锁** - 独享锁是指 **锁一次只能被一个线程所持有**。
- **共享锁** - 共享锁是指 **锁可被多个线程所持有**。

独享锁与共享锁在 Java 中的典型实现：

- **`synchronized` 、`ReentrantLock` 只支持独享锁**。
- **`ReentrantReadWriteLock` 其写锁是独享锁，其读锁是共享锁**。读锁是共享锁使得并发读是非常高效的，读写，写读 ，写写的过程是互斥的。

​	**独占锁是一种悲观锁**，由于每次访问资源都先加上互斥锁，这限制了并发性，因为读操作并不会影响数据的一致性，而独占锁只允许在同一时间由一个线程读取数据，其他线程必须等待当前线程释放锁才能进行读取。

​	**共享锁则是一种乐观锁**，它放宽了加锁的条件，允许多个线程同时进行读操作

#### 悲观锁与乐观锁

乐观锁与悲观锁不是指具体的什么类型的锁，而是**处理并发同步的策略**。

- **悲观锁** - 悲观锁对于并发采取悲观的态度，认为：**不加锁的并发操作一定会出问题**。**悲观锁适合写操作频繁的场景**。
- **乐观锁** - 乐观锁对于并发采取乐观的态度，认为：**不加锁的并发操作也没什么问题。对于同一个数据的并发操作，是不会发生修改的**。在更新数据的时候，会采用不断尝试更新的方式更新数据。**乐观锁适合读多写少的场景**。

**悲观锁与乐观锁在 Java 中的典型实现：**

- **synchronized是从悲观的角度出发：**总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程），因此synchronized也称之为悲观锁，ReentrantLock也是一种悲观锁，性能较差
- **CAS是从乐观的角度出发：**总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据。如果别人修改过，则获取现在最新的值，如果别人没修改过，直接修改共享数据的值，CAS这种机制也称之为乐观锁，综合性能较好。（**`CAS` 操作通过 `Unsafe` 类提供，但这个类不直接暴露为 API，所以都是间接使用，如各种原子类）**
  - CAS体现了无锁并发、无阻塞并发的思想：①没有使用synchronized，所以线程不会陷入阻塞，这也是效率提升的因素之一；②如果竞争激烈，失败重试必然频繁发生，效率会受到影响，所以适用于线程数少、多核CPU的场景

#### 显式锁和内置锁（可中断锁/不可中断锁）

- Java 1.5 之前，协调对共享对象的访问时可以使用的机制只有 **`synchronized` 和 `volatile`。这两个都属于内置锁**，即锁的申请和释放都是由 JVM 所控制。
- Java 1.5 之后，增加了新的机制：**`ReentrantLock`、`ReentrantReadWriteLock`** ，这类锁的申请和释放都可以由程序所控制，所以常被称为**显式锁**。

**<u>以下对比一下显式锁和内置锁的差异：</u>**

- 主动获取锁和释放锁
  - `synchronized` 不能主动获取锁和释放锁。获取锁和释放锁都是 JVM 控制的。
  - `ReentrantLock` 可以主动获取锁和释放锁。（如果忘记释放锁，就可能产生死锁）。
- 响应中断
  - `synchronized` 不能响应中断。
  - `ReentrantLock` 可以响应中断。
- 超时机制
  - `synchronized` 没有超时机制。
  - `ReentrantLock` 有超时机制。`ReentrantLock` 可以设置超时时间，超时后自动释放锁，避免一直等待。
- 支持公平锁
  - `synchronized` 只支持非公平锁。
  - `ReentrantLock` 支持非公平锁和公平锁。
- 是否支持共享
  - 被 `synchronized` 修饰的方法或代码块，只能被一个线程访问（独享）。如果这个线程被阻塞，其他线程也只能等待
  - `ReentrantLock` 可以基于 `Condition` 灵活的控制同步条件。
- 是否支持读写分离
  - `synchronized` 不支持读写锁分离；
  - `ReentrantReadWriteLock` 支持读写锁，从而使阻塞读写的操作分开，有效提高并发性。

#### 释放/不释放锁的操作

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311221728123.png" alt="image-20250311221728123" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311221734775.png" alt="image-20250311221734775" style="zoom:67%;" />

### 锁原理（对象头中的MarkWord、Monitor对象）

​	**Monitor**：**监视器或管程**。每个Java对象都可以关联一个Monitor对象，**Monitor也是class，实例存储在堆中**，使用synchronized给对象上锁（重量级）之后，该对象头的Mark Word中就被设置为指向Monitor对象的指针，这就是重量级锁。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311174657789.png" alt="image-20250311174657789" style="zoom:50%;" />

- synchronized必须是进入同一个对象的Monitor才有上述效果；
- 不加synchronized的对象不会关联监视器，不遵从以下规则。

**<u>工作流程（通过object锁住代码块）</u>**

- 开始时Monitor中的Owner为null。
- 当Thread-2执行synchronized(obj)就会**<font color = '#8D0101'>将Monitor的所有者Owner设置为Thread-2</font>**（图中箭头应该指向Thread-2），**Monitor中只能有一个Owner**，**==obj对象的Mark Word指向Monitor（此时的MarkWord就是执行Monitor的指针）==**，把对象原有的Mark Word存入线程栈中的锁记录中。
- 如果在Thread-2上锁的过程中，Thread-1、Thread-3、Thread-4也执行synchronized(obj)，就会**进入EntryList BLOCKED（双向链表）**
- Thread-2执行完同步代码块的内容后，根据obj对象头中的Monitor地址，设置Owner为null，把线程栈的锁记录中的对象头的值设置为原来的Mark Word。
- **唤醒EntryList中的等待的线程来竞争锁，竞争是非公平的**，如果这时有新的线程想要获取锁，可能就直接就抢占到了，阻塞队列的线程就会继续阻塞
- WaitSet中的Thread-0，**是以前获得过锁，但条件不满足进入WAITING状态的线程（<font color = '#8D0101'>wait-notify机制</font>）**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311175856724.png" alt="image-20250311175856724" style="zoom:67%;" />

### ==synchronized锁升级机制==

**synchronized是可重入、不公平的重量级锁**，所以可以对其进行优化

随着竞争的增加，只能**锁升级，不能锁降级**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311180300496.png" alt="image-20250311180300496" style="zoom:50%;" />

#### 对象头与其中的Mark Word 

​	锁升级功能主要依赖于 Mark Word 中的锁标志位和是否偏向锁标志位，`synchronized` 同步锁就是从偏向锁开始的，随着竞争越来越激烈，偏向锁升级到轻量级锁，最终升级到重量级锁。

​	Mark Word 记录了对象和锁有关的信息。Mark Word 在 64 位 JVM 中的长度是 64bit，我们可以一起看下 64 位 JVM 的存储结构是怎么样的。如下图所示：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200629191250.png" alt="img" style="zoom:50%;" />

Java 1.6 引入了偏向锁和轻量级锁，从而让 `synchronized` 拥有了四个状态：

- **无锁状态（unlocked）**
- **偏向锁状态（biasble）**
- **轻量级锁状态（lightweight locked）**
- **重量级锁状态（inflated）**

#### 锁的四种状态(无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁)

- **偏向锁通过对比Mark Word解决加锁问题，避免执行CAS操作**
- **轻量级锁是通过用CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒而影响性能**
- **重量级锁是将除了拥有锁的线程以外的线程都阻塞**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311180717611.png" alt="image-20250311180717611" style="zoom:67%;" />

**<u>（2）偏向锁</u>**

​	轻量级锁在没有竞争时（只有自己这个线程），每次重入仍然需要执行CAS操作，Java 6中引入了偏向锁来做进一步优化。**<font color = '#8D0101'>其目标就是在只有一个线程执行同步代码块时能够提高性能</font>**。

​	**偏向锁的思想**是偏向于第一个获取锁对象的线程，这个线程之后重新获取该锁不再需要同步操作：

- 当锁对象第一次被线程获得的时候进入偏向状态，同时**将线程ID记录到Mark Word**。当下次该线程进入这个同步块时，会去检查锁的Mark Word里面是不是放的自己的线程ID。
  - **如果是**，表明该线程已经获得了锁，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁；
  - **如果不是**，就代表有另一个线程来竞争这个偏向锁。这个时候会尝试使用CAS来替换Mark Word里面的线程ID为新线程的ID，这个时候要分两种情况：
    - **成功**，表示之前的线程不存在了，Mark Word里面的线程ID为新线程的ID，**锁不会升级，仍然为偏向锁**；
    - **失败**，表示之前的线程仍然存在，那么暂停之前的线程，设置偏向锁标识为0，并设置锁标志位为00，**升级为轻量级锁，会按照轻量级锁的方式进行竞争锁**。

**撤销偏向锁的状态：**

- 调用对象的hashCode()方法：偏向锁的对象Mark Word中存储的是线程id，调用hashCode()导致偏向锁被撤销
- 当有其他线程使用偏向锁对象时，会将偏向锁升级为轻量级锁
- 调用wait/notify（会使当前线程放弃当前锁），需要申请Monitor，进入waitSet。

**<u>（3）轻量级锁</u>**

​	如果一个对象虽然有多个线程要加锁，但是**加锁时间是错开的（也就是没有竞争），那么可以使用轻量级锁进行优化**。轻量级锁对使用者是透明的（不可见），语法仍是synchronized(锁对象)。

- JVM会为每个线程在当前线程的栈帧中创建用于存储锁记录的空间，我们称为Displaced Mark Word。如果一个线程获得锁的时候发现是轻量级锁，会把锁的Mark Word复制到自己的Displaced Mark Word里面。
- 然后线程尝试**用CAS将锁的Mark Word替换为指向锁记录的指针**，如果成功，当前线程获得锁；
- 如果失败有两种情况
  - 表示Mark Word已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，当前线程就尝试使用自旋来获取锁；
  - 如果是线程自己执行了synchronized的锁重入，那么再添加一条Lock Record作为重入的计数器

==**关于自旋**==

- 自旋是需要消耗CPU的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费CPU资源。但是JDK采用了更聪明的方式——适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。
- 自旋也不是一直进行下去的，如果自旋到一定程度（和JVM、操作系统相关），依然没有获取到锁，称为自旋失败，那么这个线程会阻塞。同时这个锁就会升级成重量级锁。

**<u>（4）重量级锁</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311182433879.png" alt="image-20250311182433879" style="zoom:80%;" />

#### 锁的升级流程

每一个线程在准备获取共享资源时：

- 第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于“偏向锁” 。
- 第二步，如果MarkWord不是自己的ThreadId，锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。
- 两个线程都把锁对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作，把锁对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord。
- 第三步中成功执行CAS的获得资源，失败的则进入自旋。
- 自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于轻量级锁的状态，如果自旋失败则进入重量级锁状态
- 第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己。

### 锁优化（自旋锁、锁消除、锁粗化）

**<u>（1）自旋锁</u>：**重量级锁竞争时，尝试获取锁的线程不会立即阻塞，可以使用自旋（默认10次）来进行优化，采用循环的方式去尝试获取锁。

- 自旋占用CPU时间，单核CPU自旋就是浪费时间，因为同一时刻只能运行一个线程，多核CPU自旋才能发挥优势
- 自旋失败的线程会进入阻塞状态

**优点**：不会进入阻塞状态，减少线程上下文切换的消耗；**缺点**：当自旋的线程越来越多时，会不断地消耗CPU资源

**说明**：在Java 6之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性较高，就多自旋几次，反之就少自旋甚至不自旋，比较智能；②Java 7之后不能控制是否开启自旋功能，由JVM控制。

<u>**（2）锁消除**：</u>指对于检测出不可能存在竞争的共享数据的锁进行消除，这是JVM即时编译器的优化。锁消除主要是通过逃逸分析来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除（同步消除：JVM逃逸分析）。

<u>**（3）锁粗化**：</u>对相同对象多次加锁，导致线程发生多次重入，频繁的加锁操作会导致性能损耗，可以使用锁粗化方式优化。如果虚拟机探测到一串的操作都对同一个对象加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部。

**Eg：一些看起来没有加锁的代码，其实隐式地加了很多锁：**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311183505294.png" alt="image-20250311183505294" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311183510354.png" alt="image-20250311183510354" style="zoom:67%;" />

​	String是一个不可变的类，编译器会对String的拼接自动优化。在JDK1.5之前，转化为StringBuffer对象的连续append()操作，每个append()方法中都有一个同步块。现在有三个append()追加，如果对每一个append()都加锁操作，频繁地进行互斥同步操作也会导致不必要的性能损耗，这是JVM不乐意看到的，为了提高JVM并发性能，此时，JVM会进行一个优化，**就是扩展到第一个append()操作之前直至最后一个append()操作之后，这样只需要加锁一次就可以了**，这就是**<font color = '#8D0101'>锁粗化</font>**。

### 锁的活跃性（死锁、活锁、饥饿）

#### 死锁

​	多个线程同时被阻塞，它们中的一个或者全部都在等待某个资源被释放，由于线程被无限期地阻塞，因此程序不可能正常终止。如下图所示，线程A持有资源2，线程B持有资源1，它们同时都想申请对方地资源，所以这两个线程就会相互等待而进入死锁状态。

出现死锁后，不会出现异常，不会出现提示，只是所有的线程都处于阻塞状态，无法继续

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172017339.png" alt="image-20250311172017339" style="zoom:80%;" />

#### 死锁四个必要条件

<font color = '#8D0101'>**四个条件都成立的时候，便形成死锁。死锁情况下打破任何一个条件，便可让死锁消失。**</font>

**<u>（1）互斥条件</u>**，即当资源被一个线程使用（占有）时，别的线程不能使用；

如果线程A已经持有的资源，不能再同时被线程B持有，如果线程B请求获取线程A已经占用的资源，那线程B只能等待，直到线程A释放了资源

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172100471.png" alt="image-20250311172100471" style="zoom:60%;" />

**<u>（2）不可剥夺条件</u>**，当线程已经持有了资源，在自己使用完之前不能被其他线程获取，线程B如果也想使用此资源，则只能在线程A使用完并释放后才能获取；

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172128097.png" alt="image-20250311172128097" style="zoom:60%;" />                               

**<u>（3）请求和保持条件</u>**，即当资源请求者在请求其他的资源的同时保持对原有资源的占有；如：当线程A已经持有了资源1，又想申请资源2，而资源2已经被线程C持有了，所以线程A就会处于等待状态，但是线程A在等待资源2的同时并不会释放自己已经持有的资源1

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172242349.png" alt="image-20250311172242349" style="zoom:60%;" />

**<u>（4）循环等待条件</u>**，即存在一个等待循环队列：p1要p2的资源，p2要p1的资源，形成了一个等待环路。如：线程A已经持有资源2，而想请求资源1，线程B已经获取了资源1，而想请求资源2，这就形成资源请求等待的环形图

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172313894.png" alt="image-20250311172313894" style="zoom:60%;" />

#### 如何定位检测死锁

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172528371.png" alt="image-20250311172528371" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311172538526.png" alt="image-20250311172538526" style="zoom:67%;" />

#### 如何预防和避免线程死锁

（1）避免一个线程同时获取多个锁。避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源。

（2）尝试使用定时锁 `lock.tryLock(timeout)`，避免锁一直不能释放。

（3）对于数据库锁，加锁和解锁必须在一个数据库连接中里，否则会出现解锁失败的情况

（4）破坏请求与保持条件：一次性申请所有资源

（5）破坏不剥夺条件：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

**避免死锁**

==银行家算法==

​	使用银行家算法破坏循环等待条件，在线程/进程进入系统时候，必须声明需要的每种资源的最大数目，当进程/线程请求一组资源时，系统必须首先确定是否有足够的资源换分配给该进程，如有，判断如果同意该请求后系统是否处于安全状态，如果处于安全状态，则统一分配，否则该线程继续等待。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311174319022.png" alt="image-20250311174319022" style="zoom:80%;" />

**上述三个矩阵间存在下述关系:Need[i,j] = Max[i,j] - allocation[i, j]**

#### 活锁

任务或执行者没有被阻塞，由于某些条件没有满足，导致一直重复尝试-失败-尝试-失败的过程。两个线程互相改变对方的结束条件，最后谁也无法结束。

#### 饥饿

一个线程由于优先级太低，始终得不到CPU调度执行，也不能够结束。

## ReentrantLock可重入锁

**`ReentrantLock` 类是 `Lock` 接口的具体实现**，与内置锁 `synchronized` 相同的是，它是一个**可重入锁**

**`Lock` 接口提供了一组无条件的、可轮询的、定时的以及可中断的锁操作**，所有获取锁、释放锁的操作都是显式的操作。

- **能够响应中断**。synchronized 的问题是，持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦发生死锁，就没有任何机会来唤醒阻塞的线程。但如果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占条件了。
- **支持超时**。如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。
- **非阻塞地获取锁**。如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件

### ReentrantLock 的特性

`ReentrantLock` 的特性如下：

- **`ReentrantLock` 提供了与 `synchronized` 相同的互斥性、内存可见性和可重入性**。
- `ReentrantLock` **支持公平锁和非公平锁**（默认）两种模式。
- ReentrantLock实现了Lock接口，支持了synchronized所不具备的灵活性。
  - 可中断；
  - 可以设置超时时间

### ReentrantLock 的方法

#### ReentrantLock 的构造方法(初始化公平、非公平锁)

`ReentrantLock` 有两个构造方法：

```java
public ReentrantLock() {}
public ReentrantLock(boolean fair) {}
```

- `ReentrantLock()` - 默认构造方法会初始化一个**非公平锁（NonfairSync）**；
- `ReentrantLock(boolean)` - `new ReentrantLock(true)` 会初始化一个**公平锁（FairSync）**。**公平锁会降低吞吐量，一般不用。**
  - ==已经处在阻塞队列中的线程（不考虑超时）始终都是公平的，先进先出==；
  - 公平锁是指未处于阻塞队列中的线程来争抢锁，如果队列不为空，则老实到队尾等待；
  - 非公平锁是指未处于阻塞队列中的线程来争抢锁，与队列头唤醒的线程去竞争，谁抢到算谁的。

**lock 方法在公平锁和非公平锁中的实现：**

​	二者的区别仅在于申请非公平锁时，如果同步状态为 0，尝试将其设为 1，如果成功，直接将当前线程置为排它线程；否则和公平锁一样，调用 AQS 获取独占锁方法 `acquire`。

```java
// 非公平锁实现
final void lock() {
    if (compareAndSetState(0, 1))
    // 如果同步状态为0，将其设为1，并设置当前线程为排它线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
    // 调用 AQS 获取独占锁方法 acquire
        acquire(1);
}
// 公平锁实现
final void lock() {
    // 调用 AQS 获取独占锁方法 acquire
    acquire(1);
}
```

#### lock 和 unlock 方法

- `lock()` - **无条件获取锁**。**如果当前线程无法获取锁，则当前线程进入休眠状态不可用**，直至当前线程获取到锁。如果该锁没有被另一个线程持有，则获取该锁并立即返回，将锁的持有计数设置为 1。

- `unlock()` - 用于**释放锁**。

  > [!WARNING]
  >
  > 注意：请务必牢记，获取锁操作 **`lock()` 必须在 `try catch` 块中进行，并且将释放锁操作 `unlock()` 放在 `finally` 块中进行，以保证锁一定被被释放，防止死锁的发生**

示例：`ReentrantLock` 的基本操作

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311224908603.png" alt="image-20250311224908603" style="zoom:67%;" />

#### tryLock 方法（可轮询、定时获取锁）

与无条件获取锁相比，tryLock 有更完善的容错机制。

- `tryLock()` - **可轮询获取锁**。如果成功，则返回 true；如果失败，则返回 false。也就是说，这个方法**无论成败都会立即返回**，获取不到锁（锁已被其他线程获取）时不会一直等待。

  ```java
  public void execute() {
      if (lock.tryLock()) {
          try {
              for (int i = 0; i < 3; i++) {
                 // 略...
              }
          } finally {
              lock.unlock();
          }
      } else {
          System.out.println(Thread.currentThread().getName() + " 获取锁失败");
      }
  }
  ```

- `tryLock(long, TimeUnit)` - **可定时获取锁**。和 `tryLock()` 类似，区别仅在于这个方法在**获取不到锁时会等待一定的时间**，在时间期限之内如果还获取不到锁，就返回 false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回 true。

  ```java
  public void execute() {
      try {
          if (lock.tryLock(2, TimeUnit.SECONDS)) {
              try {
                  for (int i = 0; i < 3; i++) {
                      // 略...
                  }
              } finally {
                  lock.unlock();
              }
          } else {
              System.out.println(Thread.currentThread().getName() + " 获取锁失败");
          }
      } catch (InterruptedException e) {
          System.out.println(Thread.currentThread().getName() + " 获取锁超时");
          e.printStackTrace();
      }
  }
  ```

#### lockInterruptibly 方法（可中断获取锁）

`lockInterruptibly()` - **可中断获取锁**。可中断获取锁可以在获得锁的同时保持对中断的响应。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311225510561.png" alt="image-20250311225510561" style="zoom:67%;" />



### ReentrantReadWriteLock

​	`ReadWriteLock` 适用于**读多写少的场景**。

​	**`ReentrantReadWriteLock` 类是 `ReadWriteLock` 接口的具体实现**，它是一个可重入的读写锁。`ReentrantReadWriteLock` 维护了一对读写锁，将读写锁分开，有利于提高并发效率。

读写锁，并不是 Java 语言特有的，而是一个广为使用的通用技术，所有的读写锁都遵守以下三条基本原则：

- **允许多个线程同时读共享变量；**
- **只允许一个线程写共享变量；**
- **如果一个写线程正在执行写操作，此时禁止读线程读共享变量。**

​	读写锁与互斥锁的一个重要区别就是**读写锁允许多个线程同时读共享变量**，而互斥锁是不允许的，这是读写锁在读多写少场景下性能优于互斥锁的关键。但**读写锁的写操作是互斥的**，当一个线程在写共享变量的时候，是不允许其他线程执行写操作和读操作。

## AQS

> `AbstractQueuedSynchronizer`（简称 **AQS**）是**队列同步器**，顾名思义，其主要作用是处理同步。它是并发锁和很多同步工具类的实现基石（如 `ReentrantLock`、`ReentrantReadWriteLock`、`CountDownLatch`、`Semaphore`、`FutureTask` 等）

### 概述

​	AQS是AbstractQueuedSynchronizer的简称，即抽**象队列同步器**，是JDK提供的一个同步框架:

- 抽象：抽象类，只实现一些主要逻辑，有些方法由子类实现；
- 队列：使用**先进先出（FIFO）队列，即CLH同步队列**存储数据；
- 同步：实现了同步的功能

​	AQS是阻塞式锁和相关的同步器工具的框架，使用AQS能简单且高效地构造出应用广泛的同步器，比如我们提到的ReentrantLock，Semaphore，ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。

### AQS 的应用

**AQS 提供了对独享锁与共享锁的支持**

#### 独享锁 API

获取、释放独享锁的主要 API 如下：

```java
public final void acquire(int arg)
public final void acquireInterruptibly(int arg)
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
public final boolean release(int arg)
```

- `acquire` - 获取独占锁。
- `acquireInterruptibly` - 获取可中断的独占锁。
- `tryAcquireNanos`\- 尝试在指定时间内获取可中断的独占锁。在以下三种情况下回返回：
  - 在超时时间内，当前线程成功获取了锁；
  - 当前线程在超时时间内被中断；
  - 超时时间结束，仍未获得锁返回 false。
- `release` - 释放独占锁。

#### 共享锁 API

获取、释放共享锁的主要 API 如下：

```java
public final void acquireShared(int arg)
public final void acquireSharedInterruptibly(int arg)
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
public final boolean releaseShared(int arg)
```

- `acquireShared` - 获取共享锁。
- `acquireSharedInterruptibly` - 获取可中断的共享锁。
- `tryAcquireSharedNanos` - 尝试在指定时间内获取可中断的共享锁。
- `release` - 释放共享锁。

### AQS 的原理

> ASQ 原理要点：
>
> - AQS 使用一个整型的 `volatile` 变量`state`来 **维护同步状态**。状态的意义由子类赋予。
> - AQS 维护了一个 FIFO 的双链表，用来存储获取锁失败的线程。
>
> AQS 围绕同步状态提供两种基本操作“获取”和“释放”，并提供一系列判断和处理方法，简单说几点：
>
> - state 是独占的，还是共享的；
> - state 被获取后，其他线程需要等待；
> - state 被释放后，唤醒等待线程；
> - 线程等不及时，如何退出等待。
>
> 至于线程是否可以获得 state，如何释放 state，就不是 AQS 关心的了，要由子类具体实现。

#### AQS核心思想

- **如果被请求的共享资源空闲**，则将当前请求资源的线程设置为有效的工作线程，并将共享资源设置锁定状态；
- **如果请求的共享资源被占用**，AQS用CLH队列锁实现线程阻塞等待以及被唤醒时锁分配的机制，将暂时获取不到锁的线程加入到队列中

#### state属性和FIFO队列

- AQS使用一个<font color = '#8D0101'>**voliate修饰的state变量**</font>，来 **维护同步状态**（**分独占模式和共享模式）**

  - 状态信息通过protected类型的getState()，setState()，compareAndSetState()进行操作，子类可以覆盖这些方法来实现自己的逻辑，控制如何获取锁和释放锁.

- AQS **维护了一个 `Node` 类型（AQS 的内部类）的双链表来完成同步状态的管理**。这个双链表是一个双向的 FIFO 队列，通过 `head` 和 `tail` 指针进行访问。

  - 如果获取同步状态state失败时，会将当前线程及等待信息等构建成一个Node，将Node放到FIFO队列队尾，同时阻塞当前线程，当线程将同步状态state释放时，会把FIFO队列中的首节点的唤醒，使其获取同步状态state。

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/aqs_1-20250311234121590.png" alt="img" style="zoom:75%;" />

**这三种操作均是原子操作，其中`compareAndSetState`的实现依赖于Unsafe的`compareAndSwapInt()`方法**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311233303096.png" alt="image-20250311233303096" style="zoom:60%;" />

**具体线程等待队列的维护(如线程获取资源失败入队/唤醒出队等)，AQS在顶层已经实现好了**：

- 提供了基于FIFO的等待队列，类似于Monitor的**EntryList**；
- 用条件变量来实现等待、唤醒机制，支持多个条件变量，类似于Monitor的WaitSet；
- 独占模式是只有一个线程能够访问资源，如**ReentrantLock**；
- 共享模式允许多个线程访问资源，**如Semaphore，ReentrantReadWriteLock**。

**<u>AQS的原理图</u>**

​	**<font color = '#8D0101'>一个锁对应一个AQS阻塞队列</font>**

​	AQS内部使用了一个先进先出（FIFO）的双端队列，并使用了两个指针head和tail用于标识队列的头部和尾部。FIFO(CLH)是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)，AQS是将每条请求共享资源的线程封装成一个CLH队列的一个结点（Node）来实现锁的分配。

​	它并不是直接储存线程，而是**储存拥有线程的Node节点。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250311234455347.png" alt="image-20250311234455347" style="zoom:67%;" />

**Node数据结构**

- `waitStatus` - `Node` 使用一个整型的 `volatile` 变量来 维护 AQS 同步队列中线程节点的状态。
- 等待锁的线程也被封装进去了,同时是volatile类型的

```java
static final class Node {
    /** 该等待同步的节点处于共享模式 */
    static final Node SHARED = new Node();
    /** 该等待同步的节点处于独占模式 */
    static final Node EXCLUSIVE = null;

    /** 线程等待状态，状态值有: 0、1、-1、-2、-3 */
    volatile int waitStatus;
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    /** 前驱节点 */
    volatile Node prev;
    /** 后继节点 */
    volatile Node next;
    /** 等待锁的线程 */
    volatile Thread thread;

  	/** 和节点是否共享有关 */
    Node nextWaiter;
}
```

#### 资源共享模式(独占和共享)

​	**(1）Exclusive（独占）**：**资源是独占的，一次只能一个线程获取**，如ReentrantLock。又可分为公平锁和非公平锁：公平锁:按照线程在队列中的排队顺序，先到者先拿到锁；非公平锁:当线程要获取锁时，无视队列顺序直接抢锁，谁抢到就是谁的。

​	**(2）share（共享）**：**资源是共享的，同时可以被多个线程获取**，具体的资源个数可以通过参数指定，如Semaphore、CountDownLatch、CyclicBarrier、ReadWriterLock。

​	不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在上层已经帮我们实现好了。

## Java线程池

**线程池**：一个容纳多个线程的容器，容器中的线程可以重复使用，省去了频繁创建和销毁线程对象的操作

### 线程池的好处

​	如果并发请求数量很多，但每个线程执行的时间很短，就会出现频繁的创建和销毁线程。如此一来，会大大降低系统的效率，可能频繁创建和销毁线程的时间、资源开销要大于实际工作所需.

- **降低资源消耗**：减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。
- **提高响应速度**：当任务到达时，如果有线程可以直接用，不会出现系统卡顿。
- **提高线程的可管理性**：如果无限制地创建线程，不仅消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控

**线程池的核心思想**：线程复用，同一个线程可以被重复使用，来处理多个任务。

**池化技术**（pool）：一种编程技巧，核心思想是资源复用，在请求量大时能优化应用性能，降低系统频繁建立销毁的资源开销。**线程池、连接池**

### Executor 框架

- `Executor` - 运行任务的简单接口。
- **`ExecutorService`** - 扩展了 `Executor` 接口。扩展能力：
  - **支持有返回值的线程；**
  - **支持管理线程的生命周期**。

- `ScheduledExecutorService` - 扩展了 `ExecutorService` 接口。扩展能力：支持**定期执行任务**。
- `AbstractExecutorService` - `ExecutorService` 接口的默认实现。
- **<font color = '#8D0101'>`ThreadPoolExecutor` - Executor 框架最核心的类，它继承了 `AbstractExecutorService` 类</font>**。
- `ScheduledThreadPoolExecutor` - `ScheduledExecutorService` 接口的实现，一个可定时调度任务的线程池。
- `Executors` - 可以通过调用 `Executors` 的静态工厂方法来创建线程池并返回一个 `ExecutorService` 对象

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/exexctor-uml.png" alt="img" style="zoom:75%;" />

`Executor` 框架不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略等，`Executor` 框架让并发编程变得更加简单。

**<u>`Executor` 框架结构主要由三大部分组成：</u>**

<u>**1、任务(`Runnable` /`Callable`)**</u>

​	执行任务需要实现的 **`Runnable` 接口** 或 **`Callable`接口**。**`Runnable` 接口**或 **`Callable` 接口** 实现类都可以被 **`ThreadPoolExecutor`** 或 **`ScheduledThreadPoolExecutor`** 执行。

<u>**2、任务的执行(`Executor`)**</u>

​	如下图所示，包括任务执行机制的核心接口 **`Executor`** ，以及继承自 `Executor` 接口的 **`ExecutorService` 接口。`ThreadPoolExecutor`** 和 **`ScheduledThreadPoolExecutor`** 这两个关键类实现了 **`ExecutorService`** 接口。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/executor-class-diagram.png" alt="img" style="zoom:50%;" />

<u>**3、异步计算的结果(`Future`)**</u>

​	**`Future`** 接口以及 `Future` 接口的实现类 **`FutureTask`** 类都可以代表异步计算的结果。

​	当我们把 **`Runnable`接口** 或 **`Callable` 接口** 的实现类提交给 **`ThreadPoolExecutor`** 或 **`ScheduledThreadPoolExecutor`** 执行。（调用 `submit()` 方法时会返回一个 **`FutureTask`** 对象）

**`Executor` 框架的使用示意图**：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Executor%E6%A1%86%E6%9E%B6%E7%9A%84%E4%BD%BF%E7%94%A8%E7%A4%BA%E6%84%8F%E5%9B%BE-8GKgMC9g.png" alt="Executor 框架的使用示意图" style="zoom:65%;" />

1. 主线程首先要创建实现 `Runnable` 或者 `Callable` 接口的任务对象。
2. 把创建完成的实现 `Runnable`/`Callable`接口的 对象直接交给 `ExecutorService` 执行: `ExecutorService.execute（Runnable command）`）或者也可以把 `Runnable` 对象或`Callable` 对象提交给 `ExecutorService` 执行（`ExecutorService.submit（Runnable task）`或 `ExecutorService.submit（Callable <T> task）`）。
3. 如果执行 `ExecutorService.submit（…）`，`ExecutorService` 将返回一个实现`Future`接口的对象（我们刚刚也提到过了执行 `execute()`方法和 `submit()`方法的区别，`submit()`会返回一个 `FutureTask 对象）。由于 FutureTask` 实现了 `Runnable`，我们也可以创建 `FutureTask`，然后直接交给 `ExecutorService` 执行。
4. 最后，主线程可以执行 `FutureTask.get()`方法来等待任务执行完成。主线程也可以执行 `FutureTask.cancel（boolean mayInterruptIfRunning）`来取消此任务的执行。

### ThreadPoolExecutor 类介绍

#### ThreadPoolExecutor构造函数重要参数

`ThreadPoolExecutor` 有四个构造方法，前三个都是基于第四个实现。第四个构造方法定义如下：

```java
    /**
     * 用给定的初始参数创建一个新的ThreadPoolExecutor。
     */
    public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                              int maximumPoolSize,//线程池的最大线程数
                              long keepAliveTime,//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                              TimeUnit unit,//时间单位
                              BlockingQueue<Runnable> workQueue,//任务队列，用来储存等待执行任务的队列
                              ThreadFactory threadFactory,//线程工厂，用来创建线程，一般默认即可
                              RejectedExecutionHandler handler//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                               ) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

**<u>(1)`ThreadPoolExecutor` 3 个最重要的参数：</u>**

1. <font color = '#8D0101'>**`corePoolSize`** </font>: 任务队列未达到队列容量时，最大可以同时运行的线程数量。**核心线程会一直存活，即使没有任务需要执行，也不会被销毁**。
2. **<font color = '#8D0101'>`maximumPoolSize`</font>** : 线程池中创建的最大线程数，当线程数量达到corePoolSize，且workQueue队列塞满任务之后，会继续创建线程，直到达到maximumPoolSize.
   1. 值得注意的是：如果使用了无界的任务队列这个参数就没什么效果。
3. <font color = '#8D0101'>**`workQueue`**</font>: 新任务来的时候会先判断当前运行的线程数量是否达到核心线程数`corePoolSize`，如果达到的话，新任务就会被存放在队列`workQueue`中。

> [!WARNING]
>
> - 如果运行的线程数少于 `corePoolSize`，则创建新线程来处理任务，即使线程池中的其他线程是空闲的。
> - 如果线程池中的线程数量大于等于 `corePoolSize` 且小于 `maximumPoolSize`，则只有当 `workQueue` 满时才创建新的线程去处理任务；
> - 如果设置的 `corePoolSize` 和 `maximumPoolSize` 相同，则创建的线程池的大小是固定的。这时如果有新任务提交，若 `workQueue` 未满，则将请求放入 `workQueue` 中，等待有空闲的线程去从 `workQueue` 中取任务并处理；
> - 如果运行的线程数量大于等于 `maximumPoolSize`，这时如果 `workQueue` 已经满了，则使用 `handler` 所指定的策略来处理任务；
> - 所以，任务提交时，**判断的顺序为 `corePoolSize` => `workQueue` => `maximumPoolSize`**。

==常见的workQueue有(具体介绍可以看并发容器章节):==

- **基于链表的无界阻塞队列**（**LinkedBlockingQueue**）（最大容量为Integer.MAX）

  - 此队列是**基于链表的先进先出队列（FIFO）**。
  - 如果创建时没有指定此队列大小，则默认为 `Integer.MAX_VALUE`。
  - 吞吐量通常要高于 `ArrayBlockingQueue`。
  - 使用 `LinkedBlockingQueue` 意味着： `maximumPoolSize` 将不起作用，线程池能创建的最大线程数为 `corePoolSize`，因为任务等待队列是无界队列。
  - `Executors.newFixedThreadPool` 使用了这个队列

- **基于数组的有界阻塞队列ArrayBlockingQueue**

  - 此队列是**基于数组的先进先出队列（FIFO）**。
  - 此队列创建时必须指定大小。

- **具有优先级的无界阻塞队列PriorityBlockingQueue**（优先级通过参数Comparator实现）

  - **无界队列**：PriorityBlockingQueue 是无界的，不会因容量限制而抛出异常，大小只受限于 JVM 内存。
  - **基于优先级排序**：队列中的元素是按照优先级排序的，优先级高的元素会排在前面。默认使用自然排序，**也可以自定义 Comparator。**
  - **不保证元素的顺序一致性**：同一优先级的元素的顺序不能保证。

- **不存储元素的阻塞队列SynchronousQueue.**如果不希望任务在队列

  - 每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态。

  - 吞吐量通常要高于 `LinkedBlockingQueue`。

  - `Executors.newCachedThreadPool` 使用了这个队列

    > [!NOTE]
    >
    > 如果不希望任务在队列中等待而是希望将任务直接移交给工作线程，可使用SynchronousQueue作为等待队列。SynchronousQueue不是一个真正的队列，而是一种线程之间移交的机制。要将一个元素放入SynchronousQueue中，必须有另一个线程正在等待接收这个元素。**只有在使用无界线程池或者有饱和策略时才建议使用该队列**。

**<u>(2)`ThreadPoolExecutor`其他常见参数：</u>**

1. `handler` - **饱和策略(拒绝策略)**。它是 `RejectedExecutionHandler` 类型的变量。当线数量达到maximumPoolSize大小，并且workQueue也已经塞满了任务，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。线程池支持以下策略：

   1. **`ThreadPoolExecutor.AbortPolicy`**(默认策略):丢弃任务并抛出RejectedExecutionException异常,拒绝新任务的处理

   2. **`ThreadPoolExecutor.DiscardPolicy`**：丢弃任务，但是不抛出异常。

   3. **`ThreadPoolExecutor.DiscardOldestPolicy`**：丢弃队列最前面的任务(最早的未处理的任务请求)，然后重新提交被拒绝的任务 

   4. **`ThreadPoolExecutor.CallerRunsPolicy`**：调用执行自己的线程运行任务，也就是**直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务(其实就是在主线程中运行被拒绝的任务)**，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。

      1. 我们直接通过 `ThreadPoolExecutor` 的构造函数创建线程池的时候，当我们不指定 `RejectedExecutionHandler` 拒绝策略来配置线程池的时候，默认使用的是 `AbortPolicy`。在这种拒绝策略下，如果队列满了，`ThreadPoolExecutor` 将抛出 `RejectedExecutionException` 异常来拒绝新来的任务 ，这代表你将丢失对这个任务的处理。如果不想丢弃任务的话，可以使用`CallerRunsPolicy`。`CallerRunsPolicy` 和其他的几个策略不同，它既不会抛弃任务，也不会抛出异常，而是将任务回退给调用者，使用调用者的线程来执行任务

         ```java
         public static class CallerRunsPolicy implements RejectedExecutionHandler {
         
                 public CallerRunsPolicy() { }
         
                 public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                     if (!e.isShutdown()) {
                         // 直接主线程执行，而不是线程池中的线程执行
                         r.run();
                     }
                 }
             }
         ```

2. **`keepAliveTime:`**-**非核心线程保持活动的时间**: 线程没有任务时最多保持多长时间会被销毁，该时间可以通过setKeepAliveTime设置：

   1. 当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，**核心线程外的线程不会立即销毁**，而是会等待，直到等待的时间超过了keepAliveTime才会被回收销毁；
   2. 如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。

3. `unit` - **`keepAliveTime` 的时间单位**。有 7 种取值。可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和毫微秒(NANOSECONDS, 千分之一微秒)。

4. `threadFactory` - **线程工厂**。创建一个新线程时使用的工厂，可以用来设置线程名、是否为daemon线程等等；executor创建新线程的时候会用到

#### execute()-线程池的执行流程(工作原理)

(1)默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。

- 提交任务可以使用 `execute` 方法，它是 `ThreadPoolExecutor` 的核心方法，通过这个方法可以**向线程池提交一个任务，交由线程池去执行**。

<u>(2)当调用**<font color = '#8D0101'>execute()方法添加一个请求任务</font>**时，线程池会做如下判断：</u>

1. 如果 `workerCount < corePoolSize`，则创建并启动一个线程来执行新提交的任务；
2. 如果 `workerCount >= corePoolSize`，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
3. 如果 `workerCount >= corePoolSize && workerCount < maximumPoolSize`，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
   1. 此时会创建**<font color = '#8D0101'>非核心线程（救济线程）</font>**立刻运行这个任务，对于阻塞队列中的任务不公平。这是因为创建每个Worker（线程）对象会绑定一个初始任务，启动Worker时会优先执行（救急线程和有界队列配合使用，如果任务超过了队列大小时，会创建**maximumPoolSize – corePoolSize数目的线程来救急**）
4. 如果`workerCount >= maximumPoolSize`，并且线程池内的阻塞队列已满，则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

(3）当一个线程完成任务时，会从队列中取下一个任务来执行；

(4）<font color = '#8D0101'>(**线程的销毁)**</font>当一个线程空闲超过一定的时间（keepAliveTime）时，线程池会判断：如果当前运行的线程数大于corePoolSize，那么这个线程就被停掉，所以线程池的所有任务完成后最终会收缩到corePoolSize大小。

> [!TIP]
>
> 在 `execute` 方法中，多次调用 **`addWorker`** 方法。`addWorker` 这个方法主要用来创建新的工作线程，如果返回 true 说明创建和启动工作线程成功，否则的话返回的就是 false。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312140627416.png" alt="image-20250312140627416" style="zoom:67%;" />

#### 其他方法submit、shutdown、shutdownNow

- `submit`类似于 `execute`，但是针对的是有返回值的线程。**`submit` 方法是在 `ExecutorService` 中声明的方法，在 `AbstractExecutorService` 就已经有了具体的实现**。**<font color = '#8D0101'>`ThreadPoolExecutor` 直接复用 `AbstractExecutorService` 的 `submit` 方法</font>**

- `shutdown` - 不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务。
  - 将线程池切换到 `SHUTDOWN` 状态；
  - 并调用 `interruptIdleWorkers` 方法请求中断所有空闲的 worker；
  - 最后调用 `tryTerminate` 尝试结束线程池。

- `shutdownNow` - 立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务。与 `shutdown` 方法类似，不同的地方在于:
  - 设置状态为 `STOP`；
  - 中断所有工作线程，无论是否是空闲的；
  - 取出阻塞队列中没有被执行的任务并返回。

#### 执行execute()方法和submit()方法的区别

- **<font color = '#8D0101'>execute()方法用于提交不需要返回值的任务</font>**，所以无法判断任务是否被线程池执行成功与否；
- **<font color = '#8D0101'>submit()方法用于提交需要返回值的任务</font>**。线程池会返回一个Future类型的对象，通过这个Future对象可以判断任务是否执行成功，并且可以通过Future的get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用get(long timeout，TimeUnit unit)方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

**处理异常**：<font color = '#8D0101'>execute会直接抛出任务执行时的异常，submit会吞掉异常</font>，有两种处理方法

- 方法一：主动捕捉异常

  ![image-20250312141418238](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312141418238.png)

- 方法二：使用Future对象的get()方法

  ![image-20250312141429276](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312141429276.png)

### Executors类实现的四种常见的线程池类型

JDK 的 `Executors` 类中提供了几种具有代表性的线程池，这些线程池 **都是基于 `ThreadPoolExecutor` 的定制化实现**。

- **`FixedThreadPool`**：**使用的是无界的 `LinkedBlockingQueue`,**固定线程数量的线程池。该线程池中的线程数量始终不变。当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。
- **`SingleThreadExecutor`**： **使用的是无界的 `LinkedBlockingQueue`,**只有一个线程的线程池。若多余一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出的顺序执行队列中的任务。
- **`CachedThreadPool`**： **使用的是同步队列 `SynchronousQueue`**,可根据实际情况调整线程数量的线程池。线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。若所有线程均在工作，又有新的任务提交，则会创建新的线程处理任务。所有线程在当前任务执行完毕后，将返回线程池进行复用。
- **`ScheduledThreadPool`**：**使用的无界的延迟阻塞队列`DelayedWorkQueue`**,给定的延迟后运行任务或者定期执行任务的线程池。

#### FixedThreadPool

**<font color = '#8D0101'>创建一个拥有n个线程的线程池，无界队列LinkedBlockingQueue</font>**

​	`FixedThreadPool` 被称为可重用固定线程数的线程池。通过 `Executors` 类中的相关源代码来看一下相关实现：

- 新创建的 `FixedThreadPool` 的 `corePoolSize` 和 `maximumPoolSize` 都被设置为 `nThreads`，这个 `nThreads` 参数是我们使用的时候自己传递的。
- 即使 `maximumPoolSize` 的值比 `corePoolSize` 大，也至多只会创建 `corePoolSize` 个线程。这是因为`FixedThreadPool` 使用的是容量为 `Integer.MAX_VALUE` 的 `LinkedBlockingQueue`（无界队列），队列永远不会被放满。

```java
   /**
     * 创建一个可重用固定数量线程的线程池
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

**<u>`FixedThreadPool` 的 `execute()` 方法运行示意图</u>**

![image-20250312142459960](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312142459960.png)

<u>**为什么不推荐使用FixedThreadPool？**</u>

`FixedThreadPool` 使用无界队列 `LinkedBlockingQueue`（队列的容量为 Integer.MAX_VALUE）作为线程池的工作队列会对线程池带来如下影响：

1. 当线程池中的线程数达到 `corePoolSize` 后，新任务将在无界队列中等待，因此线程池中的线程数不会超过 `corePoolSize`；
2. 由于使用无界队列时 `maximumPoolSize` 将是一个无效参数，因为不可能存在任务队列满的情况。所以，通过创建 `FixedThreadPool`的源码可以看出创建的 `FixedThreadPool` 的 `corePoolSize` 和 `maximumPoolSize` 被设置为同一个值。
3. 由于 1 和 2，使用无界队列时 `keepAliveTime` 将是一个无效参数；
4. 运行中的 `FixedThreadPool`（未执行 `shutdown()`或 `shutdownNow()`）不会拒绝任务，**<font color = '#8D0101'>在任务比较多的时候会导致 OOM（内存溢出）</font>**。

#### SingleThreadPoolExecutor

**<font color = '#8D0101'>创建一个只有1个线程的单线程池，无界队列LinkedBlockingQueue</font>**

**SingleThreadExecutor 的实现：**

- 新创建的SingleThreadExecutor的**corePoolSize和maximumPoolSize都被设置为1.**其他参数和FixedThreadPool相同。

```java
   /**
     *返回只有一个线程的线程池
     */
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

**SingleThreadExecutor 运行示意图：**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312143002609.png" alt="image-20250312143002609" style="zoom:80%;" />

**<u>为什么不推荐使用SingleThreadExecutor？</u>**

- `SingleThreadExecutor` 和 `FixedThreadPool` 一样，使用的都是容量为 `Integer.MAX_VALUE` 的 `LinkedBlockingQueue`（无界队列）作为线程池的工作队列。`SingleThreadExecutor` 使用无界队列作为线程池的工作队列会对线程池带来的影响与 `FixedThreadPool` 相同。说简单点，就是可能会导致 OOM。

#### CachedThreadPool

**<font color = '#8D0101'>创建一个可扩容的线程池，同步队列SynchronousQueue作为阻塞队列</font>**

CachedThreadPool是一个会根据需要创建新线程的线程池。

- CachedThreadPool的corePoolSize被设置为空（0），maximumPoolSize被设置为Integer.MAX.VALUE，即它是无界的，这也就意味着如果**主线程提交任务的速度高于maximumPool中线程处理任务的速度时**，CachedThreadPool会不断创建新的线程。极端情况下，这样会**导致耗尽cpu和内存资源**。

```java
    /**
     * 创建一个线程池，根据需要创建新线程，但会在先前构建的线程可用时重用它。
     */
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

**<u>`CachedThreadPool` 的 `execute()` 方法的执行示意图</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/CachedThreadPool-execute-CmSVV1Ww.jpeg" alt="CachedThreadPool的execute()方法的执行示意图" style="zoom:50%;" />

1. 首先执行 `SynchronousQueue.offer(Runnable task)` 提交任务到任务队列。如果当前 `maximumPool` 中有闲线程正在执行 `SynchronousQueue.poll(keepAliveTime,TimeUnit.NANOSECONDS)`，那么主线程执行 offer 操作与空闲线程执行的 `poll` 操作配对成功，主线程把任务交给空闲线程执行，`execute()`方法执行完成，否则执行下面的步骤 2；
2. 当初始 `maximumPool` 为空，或者 `maximumPool` 中没有空闲线程时，将没有线程执行 `SynchronousQueue.poll(keepAliveTime,TimeUnit.NANOSECONDS)`。这种情况下，步骤 1 将失败，此时 `CachedThreadPool` 会创建新线程执行任务，execute 方法执行完成；

**<u>为什么不推荐使用CachedThreadPool？</u>**

- `CachedThreadPool` 使用的是同步队列 `SynchronousQueue`, 允许创建的线程数量为 `Integer.MAX_VALUE` ，可能会创建大量线程，从而导致 OOM。

#### SheduledThreadPool

**<font color = '#8D0101'>ScheduledThreadPool：任务调度线程池,DelayedWorkQueue作为线程池队列。</font>**

- **ScheduledThreadPoolExecutor继承ThreadPoolExecutor。主要用在给定的延迟后运行任务，或者定期执行任务**。
  - ScheduledThreadPoolExecutor使用的任务队列**DelayQueue封装了一个PriorityQueue**，PriorityQueue会对队列中的任务进行排序，**执行所需时间短的放在前面先被执行(ScheduledFutureTask的time变量小的先执行)，如果执行所需时间相同则先提交的任务将被先执行**

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

<u>**为什么不推荐使用ScheduledThreadPool**：</u>

**ScheduledThreadPool**允许创建的线程数量为Integer.MAX_VALUE，可能会创建大量线程，从而导致OOM。

#### 阿里为何不让使用

​	线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 构造函数的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

- **线程资源必须通过线程池提供，不允许在应用中自行显式地创建线程**：
  - 使用线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题；
  - 如果不使用线程池，有可能造成系统创建大量同类线程而消耗完内存或者过度切换的问题。

- **线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式**，这样的处理方式更加明确线程池的运行规则，避免资源耗尽的风险。而Executors返回的线程池对象弊端如下：
  - FixedThreadPool和SingleThreadPool：**<font color = '#8D0101'>请求队列长度为Integer.MAX_VALUE</font>**，可能会堆积大量请求，从而导致OOM；
  - CacheThreadPool和ScheduledThreadPool：**<font color = '#8D0101'>允许创建线程数量为Integer.MAX_VALUE</font>**,可能会创建大量的线程，导致OOM。

### 线程池创建的两种方式

**方式一：通过`ThreadPoolExecutor`构造函数来创建（推荐）。**

![通过构造方法实现](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/threadpoolexecutor构造函数-BR-2Ub-c.jpeg)

**方式二：通过 `Executor` 框架的工具类 `Executors` 来创建。**

`Executors`工具类提供的创建线程池的方法如下图所示：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312141622257.png" alt="image-20250312141622257" style="zoom:80%;" />

### 线程池示例代码

**Eg1:**

首先创建一个 `Runnable` 接口的实现类（当然也可以是 `Callable` 接口，我们后面会介绍两者的区别。）

```java
public class ThreadPoolExecutorDemo {

    private static final int CORE_POOL_SIZE = 5;
    private static final int MAX_POOL_SIZE = 10;
    private static final int QUEUE_CAPACITY = 100;
    private static final Long KEEP_ALIVE_TIME = 1L;
    public static void main(String[] args) {

        //使用阿里巴巴推荐的创建线程池的方式
        //通过ThreadPoolExecutor构造函数自定义参数创建
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                CORE_POOL_SIZE,
                MAX_POOL_SIZE,
                KEEP_ALIVE_TIME,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(QUEUE_CAPACITY),
                new ThreadPoolExecutor.CallerRunsPolicy());
				for (int i = 0; i < 10; i++) {
            //创建WorkerThread对象（WorkerThread类实现了Runnable 接口）
            Runnable worker = new MyRunnable("" + i);
            //执行Runnable
            executor.execute(worker);
        }
        //终止线程池
        executor.shutdown();
        while (!executor.isTerminated()) {
        }
        System.out.println("Finished all threads");
    }
  	static class MyThread implements Runnable {
        private String command;

    		public MyRunnable(String s) {
        		this.command = s;
    		}
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " Start. Time = " + new Date());
            processCommand();
            System.out.println(Thread.currentThread().getName() + " End. Time = " + new Date());
        }
        private void processCommand() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        @Override
        public String toString() {
            return this.command;
        }
    }
}
```

**输出结构**：**线程池首先会先执行 5 个任务，然后这些任务有任务被执行完的话，就会去拿新的任务执行。**

```java
pool-1-thread-3 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-5 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-2 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-4 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-3 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-4 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-5 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-2 End. Time = Sun Apr 12 11:14:42 CST 2020
输出结构：pool-1-thread-3 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-5 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-2 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-4 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-3 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-4 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-5 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-2 End. Time = Sun Apr 12 11:14:42 CST 2020
Finished all threads  // 任务全部执行完了才会跳出来，因为executor.isTerminated()判断为true了才会跳出while循环，当且仅当调用 shutdown() 方法后，并且所有提交的任务完成后返回为 true
```

**Eg2:**

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 10, 500, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

        for (int i = 0; i < 100; i++) {
            threadPoolExecutor.execute(new MyThread());
            String info = String.format("线程池中线程数目：%s，队列中等待执行的任务数目：%s，已执行玩别的任务数目：%s",
                threadPoolExecutor.getPoolSize(),
                threadPoolExecutor.getQueue().size(),
                threadPoolExecutor.getCompletedTaskCount());
            System.out.println(info);
        }
        threadPoolExecutor.shutdown();
    }

    static class MyThread implements Runnable {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " 执行");
        }
    }
}
```

### 线程池的运行状态

![image-20250312150247045](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312150247045.png)

### 线程池最佳实践

#### 线程池大小如何确定、线程创建过多怎么办

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312153334634.png" alt="image-20250312153334634" style="zoom:75%;" />

一般多线程执行的任务类型可以分为 CPU 密集型和 I/O 密集型，根据不同的任务类型，我们计算线程数的方法也不一样。

- **CPU 密集型任务：**这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。

- **I/O 密集型任务：**这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

<u>**(1）如何判断是CPU密集型任务还是I/O密集型任务**</u>

- CPU 密集型简单理解就是利用 CPU 计算能力的任务比如你**在内存中对大量数据进行排序**。但凡涉及到**网络读取，文件读取这类都是I0密集型**，这类任务的特点是CPU 计算耗费时间相比于等待1O 操作完成的时间来说很少，大部分时间都花在了等待IO 操作完成上。


**(2）最佳线程数目**

- CPU密集型：线程数=N+1（N表示cpu个数）；
- IO密集型：线程数=2*N
- 混合型
  - 要考虑CPU空闲的时间，在IO期间CPU可以继续执行
  - ==最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间）* CPU数目==

**<u>(3）高并发、任务执行时间短的业务怎样使用线程池？并发不高、任务执行时间长的业务怎样使用线程池？并发高、业务执行时间长的业务怎样使用线程池？</u>**

- **高并发、任务执行时间短的业务**，线程池中的线程数可以设置为**CPU核心数N+1，减少线程上下文的切换次数**。
- **并发不高、任务执行时间长**的业务要区分来看：
  - ①假如业务长时间集中在IO操作上，也就是IO密集型的任务，因为IO操作不占用CPU，所以不要让所有的CPU闲下来，可以适当加大线程池中的线程数目，让CPU处理更多的业务；
  - ②假如业务长时间集中在计算操作上，也就是计算密集型任务，只能将线程池中的线程数设置得少一些，减少线程上下文得切换次数。
- **解决并发高、业务执行时间长**的业务关键不在于线程池而在于整体架构的设计，看看这些业务里面某些数据是否能做缓存是第一步，增加服务器是第二步，至于线程池的设置，设置参考并发不高、任务执行时间长的业务。最后，业务执行时间长的问题，也需要分析一下，看看能不能使用中间件对任务进行拆解和解耦

> [!CAUTION]
>
> **注意**：上面提到的公示也只是参考，实际项目不太可能直接按照公式来设置线程池参数，毕竟不同的业务场景对应的需求不同，具体还是要根据项目实际线上运行情况来动态调整。接下来介绍的美团的线程池参数动态配置这种方案就非常不错，很实用！

#### 正确声明线程池

**线程池必须手动通过 `ThreadPoolExecutor` 的构造函数来声明，避免使用`Executors` 类创建线程池，会有 OOM 风险。**

`Executors` 返回线程池对象的弊端如下(后文会详细介绍到)：

- **`FixedThreadPool` 和 `SingleThreadExecutor`**：使用的是有界阻塞队列 `LinkedBlockingQueue`，任务队列的默认长度和最大长度为 `Integer.MAX_VALUE`，可能堆积大量的请求，从而导致 OOM。
- **`CachedThreadPool`**：使用的是同步队列 `SynchronousQueue`，允许创建的线程数量为 `Integer.MAX_VALUE` ，可能会创建大量线程，从而导致 OOM。
- **`ScheduledThreadPool` 和 `SingleThreadScheduledExecutor`** : 使用的无界的延迟阻塞队列`DelayedWorkQueue`，任务队列最大长度为 `Integer.MAX_VALUE`，可能堆积大量的请求，从而导致 OOM。

说白了就是：**使用有界队列，控制线程创建数量。**

除了避免 OOM 的原因之外，不推荐使用 `Executors`提供的两种快捷的线程池的原因还有：

- 实际使用中需要根据自己机器的性能、业务场景来手动配置线程池的参数比如核心线程数、使用的任务队列、饱和策略等等。
- 我们应该显示地给我们的线程池命名，这样有助于我们定位问题。

#### 建议不同类别的业务用不同的线程池

很多人在实际项目中都会有类似这样的问题：**我的项目中多个业务需要用到线程池，是为每个线程池都定义一个还是说定义一个公共的线程池呢？**

- 一般建议是**不同的业务使用不同的线程池**，配置线程池的时候根据当前业务的情况对当前线程池进行配置，因为不同的业务的并发以及对资源的使用情况都不同，重心优化系统性能瓶颈相关的业务。

#### 别忘记给线程池命名

初始化线程池的时候需要显示命名（设置线程池名称前缀），有利于定位问题。

默认情况下创建的线程名字类似 `pool-1-thread-n` 这样的，没有业务含义，不利于我们定位问题。

给线程池里的线程命名通常有下面两种方式：

**1、利用 guava 的 `ThreadFactoryBuilder`**

```java
ThreadFactory threadFactory = new ThreadFactoryBuilder()
                        .setNameFormat(threadNamePrefix + "-%d")
                        .setDaemon(true).build();
ExecutorService threadPool = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory)
```

**2、自己实现 `ThreadFactory`。**

```java
/**
 * 线程工厂，它设置线程名称，有利于我们定位问题。
 */
public final class NamingThreadFactory implements ThreadFactory {

    private final AtomicInteger threadNum = new AtomicInteger();
    private final String name;

    /**
     * 创建一个带名字的线程池生产工厂
     */
    public NamingThreadFactory(String name) {
        this.name = name;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName(name + " [#" + threadNum.incrementAndGet() + "]");
        return t;
    }
}
```

#### 别忘记关闭线程池

当线程池不再需要使用时，应该显式地关闭线程池，释放线程资源。

线程池提供了两个关闭方法：

- **`shutdown（）`** :关闭线程池，线程池的状态变为 `SHUTDOWN`。线程池不再接受新任务了，但是队列里的任务得执行完毕。
- **`shutdownNow（）`** :关闭线程池，线程池的状态变为 `STOP`。线程池会终止当前正在运行的任务，停止处理排队的任务并返回正在等待执行的 List。

调用完 `shutdownNow` 和 `shuwdown` 方法后，并不代表线程池已经完成关闭操作，它只是异步的通知线程池进行关闭处理。如果要同步等待线程池彻底关闭后才继续往下执行，需要调用`awaitTermination`方法进行同步等待。

在调用 `awaitTermination()` 方法时，应该设置合理的超时时间，以避免程序长时间阻塞而导致性能问题。另外。由于线程池中的任务可能会被取消或抛出异常，因此在使用 `awaitTermination()` 方法时还需要进行异常处理。`awaitTermination()` 方法会抛出 `InterruptedException` 异常，需要捕获并处理该异常，以避免程序崩溃或者无法正常退出。

```java
// ...
// 关闭线程池
executor.shutdown();
try {
    // 等待线程池关闭，最多等待5分钟
    if (!executor.awaitTermination(5, TimeUnit.MINUTES)) {
        // 如果等待超时，则打印日志
        System.err.println("线程池未能在5分钟内完全关闭");
    }
} catch (InterruptedException e) {
    // 异常处理
}
```

#### 重复创建线程池的坑

线程池是可以复用的，一定不要频繁创建线程池比如一个用户请求到了就单独创建一个线程池。

```java
@GetMapping("wrong")
public String wrong() throws InterruptedException {
    // 自定义线程池
    ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,1L,TimeUnit.SECONDS,new ArrayBlockingQueue<>(100),new ThreadPoolExecutor.CallerRunsPolicy());

    //  处理任务
    executor.execute(() -> {
      // ......
    }
    return "OK";
}
```

出现这种问题的原因还是对于线程池认识不够，需要加强线程池的基础知识。

#### Spring 内部线程池的坑

使用 Spring 内部线程池时，一定要手动自定义线程池，配置合理的参数，不然会出现生产问题（一个请求创建一个线程）。

```java
@Configuration
@EnableAsync
public class ThreadPoolExecutorConfig {

    @Bean(name="threadPoolExecutor")
    public Executor threadPoolExecutor(){
        ThreadPoolTaskExecutor threadPoolExecutor = new ThreadPoolTaskExecutor();
        int processNum = Runtime.getRuntime().availableProcessors(); // 返回可用处理器的Java虚拟机的数量
        int corePoolSize = (int) (processNum / (1 - 0.2));
        int maxPoolSize = (int) (processNum / (1 - 0.5));
        threadPoolExecutor.setCorePoolSize(corePoolSize); // 核心池大小
        threadPoolExecutor.setMaxPoolSize(maxPoolSize); // 最大线程数
        threadPoolExecutor.setQueueCapacity(maxPoolSize * 1000); // 队列程度
        threadPoolExecutor.setThreadPriority(Thread.MAX_PRIORITY);
        threadPoolExecutor.setDaemon(false);
        threadPoolExecutor.setKeepAliveSeconds(300);// 线程空闲时间
        threadPoolExecutor.setThreadNamePrefix("test-Executor-"); // 线程名字前缀
        return threadPoolExecutor;
    }
}
```

#### 线程池和 ThreadLocal 共用的坑

​	**线程池和 `ThreadLocal`共用，可能会导致线程从`ThreadLocal`获取到的是旧值/脏数据**。**这是因为线程池会复用线程对象，与线程对象绑定的类的静态属性 `ThreadLocal` 变量也会被重用，这就导致一个线程可能获取到其他线程的`ThreadLocal` 值。**

不要以为代码中没有显示使用线程池就不存在线程池了，像常用的 Web 服务器 Tomcat 处理任务为了提高并发量，就使用到了线程池，并且使用的是基于原生 Java 线程池改进完善得到的自定义线程池。

当然了，你可以将 Tomcat 设置为单线程处理任务。不过，这并不合适，会严重影响其处理任务的速度。

```properties
server.tomcat.max-threads=1
```

解决上述问题比较建议的办法是使用阿里巴巴开源的 `TransmittableThreadLocal`(`TTL`)。`TransmittableThreadLocal`类继承并加强了 JDK 内置的`InheritableThreadLocal`类，在使用线程池等会池化复用线程的执行组件情况下，提供`ThreadLocal`值的传递功能，解决异步执行时上下文传递的问题。

## Java内存模型JMM

**并发安全需要满足可见性、有序性、原子性**

- Java内存区域和内存模型是不一样的概念，**<font color = '#8D0101'>内存区域</font>是指JVM运行时将数据分区域存储，强调对内存空间的划分。而<font color = '#8D0101'>内存模型</font>（Java Memory Model）是定义了线程和主内存之间的抽象关系**，即JVM描述的是一组规则或规范，通过这组规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式。
- Java内存模型（Java Memory Model，JMM）是Java虚拟机规范定义的，<font color = '#8D0101'>用来屏蔽掉java程序在各种不同的硬件和操作系统对内存的访问的差异，实现java程序在各种不同的平台上都能达到内存访问的一致性</font>

### 主内存与工作内存

​	Java内存模型的主要目标是**定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细**节。此处的变量与Java编程中的变量不是完全等同的，<font color = '#8D0101'>**这里的变量指的是实例字段、静态字段、构成数组对象的元素**</font>，但<font color = '#8D0101'>**不包括局部变量与方法参数**</font>，因为后者是线程私有的，不会被共享，自然就不会存在竞争的问题

- **主内存**：Java内存模型规定所有的变量都存储在主内存中（此处的主内存与物理硬件时的主内存名字一样，两者也可以相互类比，但此处仅是虚拟机内存的一部分）。
- **工作内存**：每条线程有自己的工作内存（可与物理硬件中处理器的高速缓存类比）。线程的工作内存中保存了该线程所使用到的变量的主内存拷贝副本，线程对变量的所有操作都必须在工作内存中进行、而不能直接读写主内存中的变量。不同线程之间也无法直接访问对方工作内存中的变量，线程间的变量传递均需要通过主内存来完成

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20210102225839.png" alt="img" style="zoom:50%;" />

​	线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量。不同的线程间也无法直接访问对方工作内存中的变量，**线程间变量值的传递均需要通过主内存来完成**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20210102225657.png" alt="img" style="zoom:50%;" />

### 工作内存与主内存交互操作

​	Java内存模型定义了8种方法来完成主内存与工作内存之间的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存。虚拟机实现的时候，必须**每一种操作都是原子的、不可再分**的

- `lock` (锁定) - 作用于**主内存**的变量，它把一个变量标识为一条线程独占的状态,一个变量在同一时间只能被一个线程锁定，
- `unlock` (解锁) - 作用于**主内存**的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- `read` (读取) - 作用于**主内存**的变量，它把一个变量的值从主内存**传输**到线程的工作内存中，以便随后的 `load` 动作使用。
- `write` (写入) - 作用于**主内存**的变量，它把 store 操作从工作内存中得到的变量的值放入主内存的变量中。
- `load` (载入) - 作用于**工作内存**的变量，它把 read 操作从主内存中得到的变量值放入工作内存的变量副本中。
- `use` (使用) - 作用于**工作内存**的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值得字节码指令时就会执行这个操作。
- `assign` (赋值) - 作用于**工作内存**的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- `store` (存储) \- 作用于**工作内存**的变量，它把工作内存中一个变量的值传送到主内存中，以便随后 `write` 操作使用。

> [!WARNING]
>
> - 如果要把一个变量从主内存中复制到工作内存，就**需要按序执行 `read` 和 `load` 操作**；如果把变量从工作内存中同步回主内存中，就**需要按序执行 `store` 和 `write` 操作**。但 Java 内存模型只要求上述操作必须按顺序执行，而没有保证必须是连续执行。

### JMM三大特性/并发安全特性：原子性、可见性、有序性

<u>**（1）原子性**</u>

​	**原子性即一个操作或者多个操作，要么全部执行（执行的过程不会被任何因素打断），要么就都不执行**。即使在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程所干扰。

​	在 Java 中，为了保证原子性，提供了两个高级的字节码指令 `monitorenter` 和 `monitorexit`。这两个字节码，在 Java 中对应的关键字就是 `synchronized`。

**<u>Java中实现原子性的方式：</u>**

- 通过**synchronized**关键字定义同步代码块或者同步方法保障原子性；
- 通过**Lock**接口保障原子性；
- 通过**Atomic**类型保障原子性。

**<u>（2）可见性</u>**

​	**可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值**。

​	**<font color = '#8D0101'>存在不可见问题的根本原因是由于缓存的存在</font>**，线程持有的是共享变量的副本，无法感知其他线程对于共享变量的更改，导致读取的值不是最新的。但是final修饰的变量是不可变的，就算有缓存，也不会存在不可见的问题。

- JMM 是通过 **"变量修改后将新值同步回主内存**， **变量读取前从主内存刷新变量值"** 这种依赖主内存作为传递媒介的方式来实现的。

**<u>实现可见性的方式：</u>**

- **volatile**：volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新；
- **synchronized**，对一个变量执行unlock操作之前，必须把变量值同步回主内存；
- **final**修饰的变量，一旦完成初始化，就不能改变。

**（3）有序性**

代码在执行的过程中的先后顺序，Java在编译器以及运行期间的优化，代码的执行顺序未必就是编写代码时候的顺序。

**重排序由以下几种机制引起：**

- **编译器优化**：对于没有数据依赖关系的操作，编译器在编译的过程中会进行一定程度的重排。
- **指令重排序**：CPU优化行为，也是会对不存在数据依赖关系的指令进行一定程度的重排。
- **内存系统重排序**：内存系统没有重排序，但是由于有缓存的存在，使得程序整体上会表现出乱序的行为

<u>**保证有序性**：</u>

- volatile：通过添加内存屏障的方式来禁止指令重排；
- synchronized：过互斥保证同一时刻只允许一条线程操作，相当于是让线程顺序执行同步代码

#### Happens-Before

JMM 为程序中所有的操作定义了一个偏序关系，称之为 **`先行发生原则（Happens-Before）`**。

<font color = '#8D0101'>**Happens-Before** 是指 **前面一个操作的结果对后续操作是可见的**。</font>

**Happens-Before** 非常重要，它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们可以通过几条规则一揽子地解决并发环境下两个操作间是否可能存在冲突的所有问题。

#### CPU缓存模型

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312161115341.png" alt="image-20250312161115341" style="zoom:80%;" />

==CPU—>CPU缓存—>内存—>硬盘==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250312161129128.png" alt="image-20250312161129128" style="zoom:80%;" />

## Java并发工具类CountDownLatch、 CyclicBarrier、Semaphore

- `CountDownLatch` 和 `CyclicBarrier` 都能够实现线程之间的等待，只不过它们侧重点不同：

  - `CountDownLatch` 一般用于某个线程 A 等待若干个其他线程执行完任务之后，它才执行；

  - `CyclicBarrier` 一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；

  - 另外，`CountDownLatch` 是不可以重用的，而 `CyclicBarrier` 是可以重用的。

- `Semaphore` 其实和锁有点类似，它一般用于控制对某组资源的访问权限。

### CountDownLatch

> 字面意思为 **递减计数锁**。用于**控制一个线程等待多个线程**。
>
> `CountDownLatch` 维护一个计数器 count，表示需要等待的事件数量。`countDown` 方法递减计数器，表示有一个事件已经发生。调用 `await` 方法的线程会一直阻塞直到计数器为零，或者等待中的线程中断，或者等待超时。

`CountDownLatch` 是基于 AQS(`AbstractQueuedSynchronizer`) 实现的。

`CountDownLatch` 唯一的构造方法：

```java
// 初始化计数器
public CountDownLatch(int count) {};
```

`CountDownLatch` 的重要方法：

```java
public void await() throws InterruptedException { };
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };
public void countDown() { };
```

说明：

- `await()` - 调用 `await()` 方法的线程会被挂起，它会等待直到 count 值为 0 才继续执行。
- `await(long timeout, TimeUnit unit)` - 和 `await()` 类似，只不过等待一定的时间后 count 值还没变为 0 的话就会继续执行
- `countDown()` - 将统计值 count 减 1

**示例：**

```java
public class CountDownLatchDemo {

    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(2);

        new Thread(new MyThread(latch)).start();
        new Thread(new MyThread(latch)).start();

        try {
            System.out.println("等待2个子线程执行完毕...");
            latch.await();
            System.out.println("2个子线程已经执行完毕");
            System.out.println("继续执行主线程");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class MyThread implements Runnable {

        private CountDownLatch latch;
        public MyThread(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void run() {
            System.out.println("子线程" + Thread.currentThread().getName() + "正在执行");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("子线程" + Thread.currentThread().getName() + "执行完毕");
            latch.countDown();
        }
    }
}
```

### CyclicBarrier

> 字面意思是 **循环栅栏**。**`CyclicBarrier` 可以让一组线程等待至某个状态（遵循字面意思，不妨称这个状态为栅栏）之后再全部同时执行**。之所以叫循环栅栏是因为：**当所有等待线程都被释放以后，`CyclicBarrier` 可以被重用**。
>
> `CyclicBarrier` 维护一个计数器 count。每次执行 `await` 方法之后，count 加 1，直到计数器的值和设置的值相等，等待的所有线程才会继续执行。

`CyclicBarrier` 是基于 `ReentrantLock` 和 `Condition` 实现的。

`CyclicBarrier` 应用场景：`CyclicBarrier` 在并行迭代算法中非常有用。

`CyclicBarrier` 提供了 2 个构造方法

```java
public CyclicBarrier(int parties) {}
public CyclicBarrier(int parties, Runnable barrierAction) {}
```

> 说明：
>
> - `parties` - `parties` 数相当于一个阈值，当有 `parties` 数量的线程在等待时， `CyclicBarrier` 处于栅栏状态。
> - `barrierAction` - 当 `CyclicBarrier` 处于栅栏状态时执行的动作。

`CyclicBarrier` 的重要方法：

```java
public int await() throws InterruptedException, BrokenBarrierException {}
public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {}
// 将屏障重置为初始状态
public void reset() {}
```

> 说明：
>
> - `await()` - 等待调用 `await()` 的线程数达到屏障数。如果当前线程是最后一个到达的线程，并且在构造函数中提供了非空屏障操作，则当前线程在允许其他线程继续之前运行该操作。如果在屏障动作期间发生异常，那么该异常将在当前线程中传播并且屏障被置于断开状态。
> - `await(long timeout, TimeUnit unit)` - 相比于 `await()` 方法，这个方法让这些线程等待至一定的时间，如果还有线程没有到达栅栏状态就直接让到达栅栏状态的线程执行后续任务。
> - `reset()` - 将屏障重置为初始状态。

示例：

```java
public class CyclicBarrierDemo {

    final static int N = 4;

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(N,
            new Runnable() {
                @Override
                public void run() {
                    System.out.println("当前线程" + Thread.currentThread().getName());
                }
            });
        for (int i = 0; i < N; i++) {
            MyThread myThread = new MyThread(barrier);
            new Thread(myThread).start();
        }
    }

    static class MyThread implements Runnable {

        private CyclicBarrier cyclicBarrier;
        MyThread(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            System.out.println("线程" + Thread.currentThread().getName() + "正在写入数据...");
            try {
                Thread.sleep(3000); // 以睡眠来模拟写入数据操作
                System.out.println("线程" + Thread.currentThread().getName() + "写入数据完毕，等待其他线程写入完毕");
                cyclicBarrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### Semaphore

> 字面意思为 **信号量**。`Semaphore` 用来控制某段代码块的并发数。
>
> `Semaphore` 管理着一组虚拟的许可（permit），permit 的初始数量可通过构造方法来指定。每次执行 `acquire` 方法可以获取一个 permit，如果没有就等待；而 `release` 方法可以释放一个 permit。

`Semaphore` 应用场景：

- `Semaphore` 可以用于实现资源池，如数据库连接池。
- `Semaphore` 可以用于将任何一种容器变成有界阻塞容器。

`Semaphore` 提供了 2 个构造方法：

```java
// 参数 permits 表示许可数目，即同时可以允许多少线程进行访问
public Semaphore(int permits) {}
// 参数 fair 表示是否是公平的，即等待时间越久的越先获取许可
public Semaphore(int permits, boolean fair) {}
```

> 说明：
>
> - `permits` - 初始化固定数量的 permit，并且默认为非公平模式。
> - `fair` - 设置是否为公平模式。所谓公平，是指等待久的优先获取 permit。

`Semaphore`的重要方法：

```java
// 获取 1 个许可
public void acquire() throws InterruptedException {}
//获取 permits 个许可
public void acquire(int permits) throws InterruptedException {}
// 释放 1 个许可
public void release() {}
//释放 permits 个许可
public void release(int permits) {}
```

示例：

```java
public class SemaphoreDemo {

    private static final int THREAD_COUNT = 30;

    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore semaphore = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        semaphore.acquire();
                        System.out.println("save data");
                        semaphore.release();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```

## Fork/Join

​	`Fork/Join` 是Fork/Join框架是一个实现了ExecutorService接口的多线程处理器，主要用于高效地处理可以分解为子任务的工作负载。**是JDK1.7加入的新的线程池实现，位于 `java.util.concurrent` 包中**，体现的是**<font color = '#8D0101'>分治思想</font>**，适用于能够进行任务拆分的**<font color = '#8D0101'>CPU密集型运算，用于并行计算</font>**

​	与其他ExecutorService相关的实现相同的是，**Fork/Join框架会将任务分配给线程池中的线程**。而与之不同的是，Fork/Join框架在执行任务时使用了**工作窃取算法**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250427104318751.png" alt="image-20250427104318751" style="zoom: 33%;" />

【**核心组件**】

1. **`ForkJoinPool`**
   - 线程池实现类，管理工作者线程（Worker Thread）和任务队列。
   - 默认使用公共池（`ForkJoinPool.commonPool()`），也可自定义池。
   - 支持动态调整并行级别（默认为 CPU 核心数）。
2. **`ForkJoinTask`**
   - 抽象基类，表示可被 `ForkJoinPool` 执行的任务。
   - 主要子类：
     - **`RecursiveAction`** ：无返回值任务（如打印操作）。
     - **`RecursiveTask<T>`** ：有返回值任务（如数值计算）。
3. **`work-stealing（工作窃取）`**
   - 线程间动态平衡负载的核心机制：空闲线程从其他线程的任务队列尾部“窃取”任务。

【 **Fork/Join 的核心思想**】

`Fork/Join` 的工作原理可以用以下步骤概括：

1. **Fork（分解）** ：将大任务递归拆分为更小的子任务，直到达到预设的 **"阈值（Threshold）"** （如任务规模小于 1000）。
2. **并行执行** ：将这些小任务分配给线程池中的线程并行执行。
3. **Join（合并）** ：等待所有小任务完成，并将它们的结果合并成最终结果。

这种模式非常适合解决递归问题或可以并行化的工作负载，例如排序、搜索、矩阵计算等。

【**核心机制：Work-Stealing**】

- **问题** ：传统线程池中，线程竞争同一队列可能导致锁争用。
- **解决方案** ：每个线程维护自己的双端队列（Deque）。
  - **push/pop** ：线程从自己队列的头部处理任务。
  - **steal** ：空闲线程从其他线程队列的尾部窃取任务。
- **优势** 
  - 减少线程阻塞。
  - 动态平衡负载，避免某些线程闲置。

【**使用步骤**】

1. **定义任务类**
   继承 `RecursiveTask` 或 `RecursiveAction`，重写 `compute()` 方法。
2. **创建任务实例**
   初始任务传入数据范围（如数组、起始/结束索引）。
3. **提交到 ForkJoinPool**
   使用 `pool.invoke(task)` 同步执行，或 `pool.submit(task)` 异步执行。
4. **获取结果**
   调用 `task.get()` 获取最终结果。

【**适用场景与限制**】

**适用场景：**

- **可分治的计算密集型任务（如排序、矩阵乘法、蒙特卡洛模拟）**。
- 并行流（Parallel Stream）底层依赖 Fork/Join 框架。

**限制：**

- **不适合 IO 密集型任务（如文件读写、网络请求），因其会阻塞线程**。
- 任务拆分过细会导致额外开销（需合理设置阈值）。
- 若任务存在依赖关系，需手动管理顺序

【**代码示例：数组求和**】

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

class SumTask extends RecursiveTask<Long> {
    private final int[] array;
    private final int start, end;
    private static final int THRESHOLD = 1000; // 任务拆分阈值

    public SumTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // 直接计算
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // 拆分子任务
            int mid = (start + end) / 2;
            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);
            left.fork();  // 异步执行左任务
            right.fork(); // 异步执行右任务
            return left.join() + right.join(); // 合并结果
        }
    }
}

public class Main {
    public static void main(String[] args) {
        int[] array = new int[10000];
        // 初始化数组...
        ForkJoinPool pool = new ForkJoinPool();
        SumTask task = new SumTask(array, 0, array.length);
        long result = pool.invoke(task); // 提交任务
        System.out.println("Sum: " + result);
    }
}
```



