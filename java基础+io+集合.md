

# java基础

## jvm层面的类冲突

​	在 JVM 层面，**类冲突（Class Conflict）** 是指多个版本的相同类（全限定名完全一致）被加载到 JVM 中，导致运行时行为不一致、抛出异常甚至程序崩溃的问题。

**【JVM 是如何加载类的】**

Java 使用的是 **双亲委派机制（Parent Delegation Model）** 的类加载模型：

1. **Bootstrap ClassLoader** ：加载 Java 核心类（如 `rt.jar`）
2. **Extension ClassLoader** ：加载扩展类（如 `$JAVA_HOME/lib/ext` 下的 jar）
3. **Application ClassLoader** ：加载用户类路径（classpath）中的类
4. **自定义 ClassLoader** ：Flink 等框架会使用自己的类加载器来隔离不同 Job 的用户代码

每个类加载器可以加载类，但它们之间有父子关系，优先让父类加载器尝试加载。

**【JVM 中为什么会发生类冲突】**

**1）相同的类由不同的类加载器加载**

JVM 认为：**“同一个类 = 全限定类名 + 加载它的类加载器”**

即使两个类的全限定名一样，只要是由不同的类加载器加载的，就被认为是不同的类。

**2）类的不同版本被加载进 JVM**

- 编译时使用 `Jackson 2.9.0`
- 运行时环境中实际加载了 `Jackson 2.8.0`
- 某些方法在 2.9 有，但在 2.8 没有 → 抛出 `NoSuchMethodError`

【**JVM 层面排查类冲突的方法**】

✅ **查看完整堆栈信息**

从日志中找到异常的完整堆栈，定位是哪个类引发了冲突：

```bash
java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectMapper.enable(Lcom/fasterxml/jackson/core/JsonParser$Feature;)Lcom/fasterxml/jackson/databind/ObjectMapper;
```

【**解决类冲突的常用方法**】

1. 查看异常堆栈 → 是否 NoSuchMethodError / IncompatibleClassChangeError
2. 使用 -verbose:class → 查看类加载路径
3. 使用 mvn dependency:tree / gradle dependencies → 查找重复依赖
4. 使用排除(exclude)、统一版本、Shading 等手段修复
5. 使用自定义 ClassLoader 避免冲突
6. 必要时写 Agent 动态监控类加载过程



## 变量的分类-按声明的位置的不同

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305222449758.png" alt="image-20250305222449758" style="zoom:50%;" />

同：都有生命周期；异：局部变量除形参外，需显式初始化。

## JDK、JRE和JVM之间的关系、Java为什么可以跨平台

==源文件（通过编译）——>字节码文件——>JVM——>OS==

（**<u>1）JVM是一种用于计算设备的规范，</u>**它是一个虚构出来的计算机，是通过在实际的计算机上仿真模拟各种计算机功能来实现的，用于执行byte code字节码。引入Java语言虚拟机后，Java语言在不同平台上运行时不需要重新编译。Java语言使用Java虚拟机屏蔽了与具体平台相关的信息，使得Java语言编译程序只需生成在Java虚拟机上运行的字节码，就可以在多种平台上不加修改地运行。JVM有针对不同系统地特定实现（Windows、Linux和macOS），目的是使用相同的字节码，它们都会给出相同的结果

**<u>（2）JDK和JRE</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305222643043.png" alt="image-20250305222643043" style="zoom:50%;" />

- JDK是Java Development Kit，它是功能齐全的Java SDK（Software Development Kit），拥有JRE所有的一切，还有编译器（javac）和工具（如javadoc和jdb），能够创建和编译程序
- JRE是Java运行时环境，它是运行已编译Java程序所需的所有内容的集合，包括JVM、Java类库、java命令和其他一些基础构件，但不能用于创建新程序。

如果只是为了运行一下Java程序的话，那么只需安装JRE；如果需要进行Java编程方面的工作，就需要安装JDK

## 字节码

​	在Java中，JVM可以理解的代码叫**字节码（即扩展名为**.class**的文件）**，它不面向任何特定处理器，只面向虚拟机。Java语言通过字节码的方式，在一定程度上解决了传统解释型语言执行效率低的问题，同时也保留了解释型语言可移植性的特点。所以Java程序运行时较高效，同时由于字节码不针对一种特定的机器，因此Java程序无需重新编译便可在多种不同操作系统的计算机上运行

- Java程序从源代码到运行一般是3步：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305223450538.png" alt="image-20250305223450538" style="zoom:50%;" />

- 编译完源程序以后，生成一个或多个字节码文件。我们使用JVM中的类加载器和解释器对生成的字节码文件进行解释运行，意味着需要将字节码文件对应的类加载到内存中，涉及到内存解析

## jvm层面的类冲突

​	在 JVM 层面，**类冲突（Class Conflict）** 是指多个版本的相同类（全限定名完全一致）被加载到 JVM 中，导致运行时行为不一致、抛出异常甚至程序崩溃的问题。

**【JVM 是如何加载类的】**

Java 使用的是 **双亲委派机制（Parent Delegation Model）** 的类加载模型：

1. **Bootstrap ClassLoader** ：加载 Java 核心类（如 `rt.jar`）
2. **Extension ClassLoader** ：加载扩展类（如 `$JAVA_HOME/lib/ext` 下的 jar）
3. **Application ClassLoader** ：加载用户类路径（classpath）中的类
4. **自定义 ClassLoader** ：Flink 等框架会使用自己的类加载器来隔离不同 Job 的用户代码

每个类加载器可以加载类，但它们之间有父子关系，优先让父类加载器尝试加载。

**【JVM 中为什么会发生类冲突】**

**1）相同的类由不同的类加载器加载**

JVM 认为：**“同一个类 = 全限定类名 + 加载它的类加载器”**

即使两个类的全限定名一样，只要是由不同的类加载器加载的，就被认为是不同的类。

**2）类的不同版本被加载进 JVM**

- 编译时使用 `Jackson 2.9.0`
- 运行时环境中实际加载了 `Jackson 2.8.0`
- 某些方法在 2.9 有，但在 2.8 没有 → 抛出 `NoSuchMethodError`

【**JVM 层面排查类冲突的方法**】

✅ **查看完整堆栈信息**

从日志中找到异常的完整堆栈，定位是哪个类引发了冲突：

```bash
java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectMapper.enable(Lcom/fasterxml/jackson/core/JsonParser$Feature;)Lcom/fasterxml/jackson/databind/ObjectMapper;
```

【**解决类冲突的常用方法**】

1. 查看异常堆栈 → 是否 NoSuchMethodError / IncompatibleClassChangeError
2. 使用 -verbose:class → 查看类加载路径
3. 使用 mvn dependency:tree / gradle dependencies → 查找重复依赖
4. 使用排除(exclude)、统一版本、Shading 等手段修复
5. 使用自定义 ClassLoader 避免冲突
6. 必要时写 Agent 动态监控类加载过程





## 基本数据类型

对于每一种数据都定义了明确的具体数据类型（强类型语言），在内存中分配了不同大小的内存空间

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305223550402.png" alt="image-20250305223550402" style="zoom:50%;" />

**<u>（1）整数类型：byte、short、int、long</u>**

- Java各整数类型有固定的表数范围和字段长度，不受具体 OS 的影响，以保证java程序的可移植性。
- java的整型常量默认为int型，声明long型常量须后加‘l’或‘L’
- java程序中变量通常声明为int型，除非不足以表示较大的数，才使用long

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305223711413.png" alt="image-20250305223711413" style="zoom:80%;" />

**<font color = '#8D0101'>bit:计算机中的最小存储单位。byte:计算机中基本存储单元</font>**

**int为什么是[2^31-1 ~ -2^31]?**

1）因为最高位是符号位，所以只有31位：2^0~2^30.

正数最大：011111111 11111111 11111111 11111111

**等比数列前n项和公式:2^0 + 2^1 + …+2^30 = 2^31-1**

2）负数：

- 从负数得到其补码：负数的反码+1（符号位不变）
- 从负数补码得到负数：补码-1再取反码。

​	若为负数，最小表示时，首位为1，其余位数全部为1，则为111111111 11111111 11111111 11111111，其补码为10000000 00000000 00000000 00000001转换成十进制就是-2147483647，即-2^31 + 1。那为什么负数最小能表示到-2147483648 即-2^31呢？问题就出在0上

- **0的补码，数0的补码表示是唯一的，例：[+0]补=[+0]反=[+0]原=00000000，[-0]补=11111111+1=0000000**

- 巧合的是**-2147483648的补码表示为1000 0000 0000 0000 0000 0000 0000 0000，与-0的原码是一致的，这样，-0便被利用起来，存储-2147483648**

**<u>（2）浮点类型：float、double</u>**

- 与整数类型类似，Java浮点类型也有固定的表数范围和字段长度，不受具体操作系统的影响。
- 浮点型常量有两种表示形式：
  - 十进制数形式：如：5.12 512.0f .512 (必须有小数点）
  - 科学计数法形式:如：5.12e2 512E2 100E-2

float:单精度，尾数可以精确到7位有效数字。很多情况下，精度很难满足需求。

double:双精度，精度是float的两倍。通常采用此类型。

**<font color = '#8D0101'>Java的浮点型常量默认为double型，声明float型常量，须后加‘f’或‘F</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305224439964.png" alt="image-20250305224439964" style="zoom:80%;" />

**<u>（3）字符类型：char</u>**

- char型数据用来表示通常意义上“字符”**<font color = '#8D0101'>(2字节)</font>**
- Java中的所有字符都使用Unicode编码，故一个字符可以存储一个字母，一个汉字，或其他书面语的一个字符。
- 字符型变量的三种表现形式：
  - 字符常量是用单引号(‘ ’)括起来的单个字符。例如：char c1 =‘a’; char c2 = ‘中’; char c3 =‘9’;
  - Java中还允许使用转义字符‘\’来将其后的字符转变为特殊字符型常量。例如：char c3 = ‘\n’;表示换行符

<u>**（4）布尔类型：boolean**</u>

- boolean 类型用来判断逻辑条件，一般用于程序流程控制：
- 不可以使用0或非0的整数替代false和true，这点和C语言不同。
- Java虚拟机中没有任何供boolean值专用的字节码指令，Java语言表达所操作的boolean值，在编译之后都使用java虚拟机中的int数据类型来代替：true用1表示，false用0表示

### 基本数据类型转换/提升

自动类型转换：**<font color = '#8D0101'>容量小的类型自动转换为容量大的数据类型</font>**。数据类型按容量大小排序为

 ![image-20250305225135744](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305225135744.png)

- 有多种类型的数据混合运算时，系统首先自动将所有数据转换成容量最大的那种数据类型，然后再进行计算。
- byte,short,char之间不会相互转换，他们三者在计算时首先转换为int类型。
- **boolean类型不能与其它数据类型运算。**
- 当把任何基本数据类型的值和字符串(String)进行连接运算时(+)，基本数据类型的值将自动转化为字符串(String)类型。

### 强制类型转换

- 自动类型转换的逆过程，将**<font color = '#8D0101'>容量大的数据类型转换为容量小的数据类型</font>**。使用时要加上强制转换符：()，但可能造成精度降低或溢出,格外要注意。
- boolean类型不可以转换为其它的数据类型

## 二进制(反码和补码)

Java整数常量默认是int类型，当用二进制定义整数时，其第32位是符号位；当是long类型时，二进制默认占64位，第64位是符号位

二进制的整数有如下三种形式：

- **原码**：直接将一个数值换成二进制数。最高位是符号位
- **负数的反码**：是对原码按位取反，只是最高位（符号位）确定为1。
- **负数的补码**：其反码加1。计算机以二进制补码的形式保存所有的整数。
  - **从负数得到其补码：负数的反码+1（符号位不变）**
  - **从负数补码得到负数：补码-1再取反码**

**正数的原码、反码、补码都相同**

- **<font color = '#8D0101'>八进制3个为一个单位</font>**

![image-20250305225611891](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305225611891.png)

- **<font color = '#8D0101'>十六进制4个为一个单位</font>**

![image-20250305225706570](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305225706570.png)

## 浅拷贝与深拷贝

- **浅拷贝**：
  - 浅克隆只复制对象的第一层结构，而对象内部的引用类型数据仍然共享原始对象的引用。
  - 换句话说，浅克隆创建了一个新对象，但该对象的属性如果是引用类型，则直接复制引用地址，而不是复制引用内容。

- **深拷贝**：
  - 深克隆会递归地复制对象的所有层级结构，包括内部嵌套的引用类型数据。
  - 换句话说，深克隆创建了一个完全独立的新对象，与原始对象没有任何共享引用。

- **Java中的Object类中提供了 `clone()` 方法来实现浅克隆。**

![image-20250305225902200](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305225902200.png)

![image-20250409184414789](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250409184414789.png)

## java程序初始化顺序

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305225925806.png" alt="image-20250305225925806" style="zoom:80%;" />

## 创建对象的方式(除了new外)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305230100987.png" alt="image-20250305230100987" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305230124932.png" alt="image-20250305230124932" style="zoom:80%;" />

## 封装与隐藏

程序设计追求“高内聚，低耦合”。

- 高内聚：类的内部数据操作细节自己完成，不允许外部干涉；
- 低耦合：仅对外暴露少量的方法用于使用

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305230438726.png" alt="image-20250305230438726" style="zoom:70%;" />

**<u>封装性的体现：</u>**

- 我们将**类的属性私有化(private),同时,提供公共的(public)方法**来获取(getXxx)和设置(setXxx)。
- **不对外暴露的私有方法**；
- **单例模式（将构造器私有化）**。
- 如果不希望类在包外被调用，可以将类设置为缺省的（对类的修饰符，只能为public或者缺省）。

## 继承性extends

**<u>1）继承性的好处</u>**

- 减少了代码的冗余，提高了代码的复用性；
- 便于功能的扩展；
- 为之后多态性的使用，提供了前提

**<u>2）体现</u>**

​	<font color = '#8D0101'>一旦子类A继承父类B以后，子类A中就获取了父类B中声明的所有的属性和方法</font>。特别的：父类中声明为private的属性或方法，子类继承父类后，仍然认为获取了父类中私有的结构，只是因为封装性的影响，使得子类不能之间调用父类的结构而已。（对于<font color = '#8D0101'>private属性，可以利用get，set方法来调用；对于private方法，可以将之放在public方法中，待调用时可一起调用）</font>

<font color = '#8D0101'>	子类继承父类以后，还可以声明自己特有的属性或方法：实现功能的拓展</font>

**<u>3）继承性的规定</u>**

- 一个子类只能有一个父类，一个父类可以有多个子类；
- Java中类的单继承性；
- 子父类是相对的概念。直接继承的叫直接父类，间接继承的叫间接父类。

​	如果我们没有显式的声明一个类的父类的话，则此类继承于java.lang.Object类；所有的java类(除java.long.Object类之外)都直接或间接地继承于java.lang.Object类；意味着，所有的java类具有java.lang.Object类声明的功能

### 子类对象实例化全过程

- **从结果上来看**：子类继承父类以后，就获取了父类中声明的属性和方法，创建子类的对象，在堆空间中，就会加载所有父类中声明的属性。
- **从过程上来看**：当我们通过子类构造器创建子类对象时，一定会直接或间接地调用父类的构造器，进而调用父类的父类的构造器，直到调用了java.lang.Object类中空参的构造器为止（目的是帮助子类做初始化工作），**正因为加载过所有的父类的结构，所以才可以看到内存中有父类中的结构，子类对象才可以考虑从进行调用**。虽然创建子类对象时调用了父类的构造器，但是<font color = '#8D0101'>自始至终就创建过一个对象，即为new的子类对象。</font>

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305230944967.png" alt="image-20250305230944967" style="zoom:80%;" />

### 定义一个不做事且无参的构造方法

​	Java程序在执行子类的构造方法之前，<font color = '#8D0101'>如果没有用super()来调用父类特定的构造方法，则会调用父类中“无参的构造方法”（目的是帮助子类做初始化工作）</font>。因此，如果父类中只定义了有参数的构造方法，而子类的构造方法中又没有用super()来调用父类中特定的构造方法，则编译时将发生错误，因为Java程序在父类中找不到没有参数的构造方法可供执行，所以要在父类里加上一个不做事且无参的构造方法

## 多态性

### 多态的概念

1）对象的多态性：父类的引用指向子类的对象                               

2）多态的使用：虚拟方法调用（父类的方法称为虚拟方法）

有了对象的多态性以后，**<font color = '#8D0101'>我们在编译期，只能调用父类中声明的方法，但在执行期实际执行的是子类重写父类的方法</font>**。

简称：==编译时，看左边；运行时，看右边==

​	多态就是动态绑定，是指在<font color = '#8D0101'>执行期间而不是在编译期间判断引用对象的实际类型调用相关方法</font>。多态是同一个行为具有多个不同表现形式或形态的能力。多态是同一个接口，使用不同的实例而执行不同操作

- **编译时多态：**编译期间决定目标方法、通过**overloading重载**实现、方法名相同，参数不同。
- **运行时多态：**运行期间决定目标方法、同名同参、**overriding重写和继承实现**、JVM决定目标方法

### 重载与多态

- 重载，是指允许存在多个<font color = '#8D0101'>同名方法，而这些方法的参数不同</font>。编译器根据方法不同的参数表，对同名方法的名称做修饰。对于编译器而言，这些同名方法就成了不同的方法。它们的调用地址在编译期就绑定了。Java的重载是可以包括父类和子类的，即<font color = '#8D0101'>子类可以重载父类的同名不同参数的方法</font>。所以：对于重载而言，在方法调用之前，编译器就已经确定了所要调用的方法，这称为“早绑定”或“静态绑定”；
- 而对于多态，只有等到方法调用的那一刻，解释运行器才会确定所要调用的具体方法，这称为“晚绑定”或“动态绑定”

### 向下转型

==向上转型就是多态==

​	有了对象的多态性以后，内存中实际上是加载了子类特有的属性和方法的，但是由于变量声明为父类类型，导致编译时，只能调用父类中声明的属性和方法，子类特有的属性和方法不能调用。如何才能调用子类特有的属性和方法？向下转型：

**对比基本数据类型：**

![image-20250307162001302](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307162001302.png)

​	为了避免在向下转型时出现ClassCastException的异常，**我们在向下转型之前，先进行instanceof判断**（a instanceof A：判断对象a是否是类A的实例，如果类B是类A的父类，那么a instanceof A返回true,则a instanceof B也返回true。**即a的类，需要与A有子类和父类的关系**）

- **编译过不过主要看语法有没有错误，也就是说两边的类是不是子父类关系，运行过不过就看左边new的是不是子类对象然后转成其父类类型**

## 关键字 

### this

this用来修饰、调用：<font color = '#8D0101'>属性、方法、构造器</font>

**<u>（1）this修饰属性和方法</u>**

this理解为：当前对象,或当前正在创建的对象（创建构造器）

- 在类的方法中，我们可以使用"this.属性"或"this.方法"的方式，调用当前对象属性和方法。通常情况下，我们都选择省略“this.”。特殊情况下，<font color = '#8D0101'>如果方法的形参和类的属性同名，我们必须显式的使用"this.变量"的方式，表明此变量是属性，而非形参</font>
- 在类的构造器中，我们可以使用"this.属性"或"this.方法"的方式，调用正在创建的对象属性和方法。但是，通常情况下，我们都选择省略“this.”。特殊情况下，<font color = '#8D0101'>如果构造器的形参和类的属性同名，我们必须显式的使用"this.变量"的方式，表明此变量是属性，而非形参</font>

**<u>（2）this调用构造器</u>**

- 我们可以在类的构造器中，显式的使用"this(形参列表)"的方式，调用本类中重载的其他的构造器
- 构造器中不能通过"this(形参列表)"的方式调用自己
- 规定“this(形参列表)”<font color = '#8D0101'>必须声明在当前构造器的首行</font>
- **不能与 `super()` 同时使用**
- 在类的一个构造器中，最多只能声明一个"this(形参列表)"。
- 如果一个类中声明了n个构造器，则最多有n -1个构造器中使用了"this(形参列表)"

**eg：**

```java
public class Person {
    private String name;
    private int age;

    // 无参构造器
    public Person() {
        this("Unknown"); // 调用带1个参数的构造器
    }

    // 带1个参数的构造器
    public Person(String name) {
        this(name, 0); // 调用带2个参数的构造器
    }

    // 带2个参数的构造器
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

### super

super可以用来调用:<font color = '#8D0101'>属性、方法、构造器</font>

<u>**1）super的使用：调用属性和方法**</u>

- 我们可以在子类的方法或构造器中，通过使用通过"super.属性"或"super.方法"的方式，显式的调用父类中声明的属性或方法。但是，<font color = '#8D0101'>通常情况下，我们习惯去省略这个"super."</font>
- <font color = '#8D0101'>特殊情况：当子类和父类中定义了同名的属性时，我们要想在子类中调用父类中声明的属性，则必须显式的使用"super.属性"的方式</font>，表明调用的是父类中声明的属性
- <font color = '#8D0101'>特殊情况：当子类重写了父类中的方法后，我们想在子类的方法中调用父类中被重写的方法时，必须显式的使用"super.方法"的方式</font>，表明调用的是父类中被重写的方法

<u>**2）super调用构造器**</u>

- 我们可以在子类的构造器中显式的使用"super(形参列表)"的方式,调用父类中声明的指定的构造器
- <font color = '#8D0101'>"super(形参列表)"的使用，必须声明在子类构造器的首行！（跟this一样）</font>
- 我们在类的构造器中，<font color = '#8D0101'>针对于"this(形参列表)"或"super(形参列表)"只能二选一，不能同时出现</font>
- 在构造器的首行，既没有显式的声明"this(形参列表)"或"super(形参列表)",则<font color = '#8D0101'>默认的调用的是父类中的空参构造器。super()</font>
- 在类的多个构造器中，**至少有一个类的构造器使用了"super(形参列表)",调用父类中的构造器**

### static

Static可修饰：<font color = '#8D0101'>属性、方法、代码块、内部类</font>

- 我们有时候希望无论是否产生了对象或无论产生了多少对象的情况下，<font color = '#8D0101'>某些特定的数据在内存空间里只有一份</font>。
- **<font color = '#8D0101'>静态结构与类的生命周期同步，非静态结构与对象生命周期同步</font>**

<u>**1）使用static修饰属性：<font color = '#8D0101'>静态变量（类变量）</font>**</u>

属性，按是否使用static修饰，又分为：静态属性（类变量）vs非静态属性（实例变量）

- **实例变量**:我们创建了类的多个对象，每个对象都独立的拥有了一套类中的非静态属性。当修改其中一个非静态属性时，不会导致其他对象中同样的属性值的修饰。
- **静态变量（类变量）**:我们创建了类的多个对象，多个对象共享同一个静态变量。当通过静态变量去修改某一个变量时，会导致其他对象调用此静态变量时，是修改过的
- **<font color = '#8D0101'>静态变量随着类的加载而加载。可以通过"类.静态变量"的方式进行调用</font>**。
- **静态变量的加载要早于对象的创建（类的加载早于对象的创建）**
- 由于类只会加载一次，则静态变量在内存中也只会存在一次。<font color = '#8D0101'>存在方法区的静态域中</font>

**<u>2）static修饰方法</u>**

- 随着类的加载而加载，可以通过"类.静态方法"的方式调用：常用于比如Arrays.sort()等
- <font color = '#8D0101'>静态方法中，**只能调用静态的方法或属性（因为它们的声明周期不够，静态方法在它们加载之前就已经加载了）**</font>；**非静态的方法中，可以调用所有的方法或属性（包括静态和非静态的方法和属性）**
- **在静态的方法内，不能使用this、super关键字**
- **<font color = '#8D0101'>静态结构与类的生命周期同步，非静态结构与对象生命周期同步</font>**

<u>**3）开发中，如何确定一个属性/方法是否要声明为static？**</u>

**属性：**

- 属性是可以被多个对象所共享的，不会随着对象的不同而不同的。
- 类中的常量也常常声明为static

**方法：**

- 操作静态属性的方法，通常设置为static的
- 工具类中的方法，习惯上声明为static的。比如：Math、Arrays、Collections



### final

final可以用来修饰的结构:<font color = '#8D0101'>类、方法、变量</font>

- **final用来修饰一个类**：此类不能被其他类所继承（功能不用再扩充了）：比如：String类、System类、StringBuffer类
- **final用来修饰方法：**表明此方法不可以被重写
- **final用来修饰变量**：此时的“变量”(成员变量或局部变量)就称为是一个常量。<font color = '#8D0101'>名称大写</font>，且只能被赋值一次
- final修饰属性，可以考虑赋值的位置有:显式初始化、代码块中初始化、构造器中初始化（通过“对象.属性”或“对象.方法”的方式进行赋值太晚了，必须得在初始化时就赋值）
- final修饰局部变量:尤其是使用final修饰形参时，表明此形参是一个常量。当我们调用此方法时，给常量形参赋一个实参。一旦赋值以后，就只能在方法体内使用此形参，但不能进行重新赋值
- **<font color = '#8D0101'>static final用来修饰属性:全局常量</font>**

​	如果一个对象不能够修改其内部状态（属性），那么就是不可变对象。不可变对象是线程安全的，不存在并发修改和可见性问题，是另一种避免竞争的方式

## 方法的重载(overload)

在同一个类中，允许存在一个以上的同名方法，只要它们的参数个数或者参数类型不同即可

“两同一不同”:

- 同一个类、相同方法名
- 参数列表不同：**参数个数不同，参数类型不同**

与方法的**返回值类型、权限修饰符、形参变量名、方法体都无关**

## 方法的重写（override/overwrite）

1）子类继承父类以后，可以对父类中的方法进行覆盖操作

2）重写以后，当创建子类对象以后，通过调用子类对象调用子父类的同名同参数的方法时，实际执行的是子类重写父类的方法。（当调用父类的对象执行的还是，父类的方法，即没重写之前的）

- 子类中的叫重写方法，父类中的叫被重写的方法
- 子类重写的方法的方法名和形参列表与父类被重写的方法的方法名和形参列表相同
- 子类重写的方法的**权限修饰符不小于**父类被重写的方法的权限修饰符。

> **特殊情况：子类不能重写父类中声明为private权限的方法。**

**<u>3）返回值类型：</u>**

- 父类被重写的方法的返回值类型是void，则子类重写的方法的返回值类型只能是void
- **父类被重写的方法的返回值类型是A类型（引用数据类型），则子类重写的方法的返回值类型可以是A类或A类的子类;**（eg：父类为Object类型，子类为String类型）
- **父类被重写的方法的返回值类型是基本数据类型(比如:double)，则子类重写的方法的返回值类型必须是相同的基本数据类型**(必须是:double)。
- **子类重写的方法抛出的异常类型不大于父类被重写的方法抛出的异常类型**（具体在异常处理讲）。

4）**子类和父类中的同名同参数的方法要么都声明为非static的（考虑重写），要么都声明为static的（不是重写）**。因为static方法是属于类的，子类无法覆盖父类的方法。

## 重载和重写的区别

方法的重写Overriding和重载Overloading是Java多态性的不同表现。

**重写Overriding是父类与子类之间多态性的一种表现(运行时多态)，重载Overloading是一个类中多态性的一种表现(编译时多态)**。

- 如果在子类中定义某方法与其父类有相同的名称和参数，我们说该方法被重写(Overriding)。子类的对象使用这个方法时，将调用子类中的定义，对它而言，父类中的定义如同被"屏蔽"了。
- 如果在一个类中定义了多个同名的方法，它们或有不同的参数个数或有不同的参数类型，则称为方法的重载(Overloading)。

## equals()和==的区别

<u>**1）==运算符**</u>

- 可以使用在**基本数据类型变量（数据值）和<font color = '#8D0101'>基本数据类型变量（数据值）和引用数据类型变量（比较地址值）中</font>中**
- 如果比较的是基本数据类型变量，则比较两个变量保存的数据是否相等（**数据类型可以不相同但7种基本数据类型不能与boolean进行运算**）；如果比较的是引用数据类型变量，则比较两个对象的地址值是否相同，即两个引用是否指向同一个对象实体。

<u>**2）equals()方法**</u>

- 只能适用于**引用数据类型**
- Object类中equals()的定义：

```java
public boolean equals(Object obj){
		return (this == obj);
}
```

Object类中定义的equals()和==的作用是相同的，比较两个对象的地址值是否相同，即两个引用是否指向同一个对象实体

- 像**String、Date、File、包装类等都重写了Object类中的equals()方法**.重写之后，对比的不是两个引用的地址是否相同，而是**比较两个对象的“实体内容”是否相同**。

​	通常情况下，我们自定义的类如果使用equals()的话，也通常是比较两个对象的"实体内容"是否相同。那么，我们就需要对Object类中的equals()进行重写。

![image-20250305235052223](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305235052223.png)

## equals()与hashcode()

​	**对象Hash的前提是实现equals()和hashCode()两个方法**，那么HashCode()的作用就是保证对象返回唯一hash值，但当两个对象计算值一样时，这就发生了碰撞冲突

![image-20250305235140743](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305235140743.png)

**3）为什么重写equals时必须重写hashCode()方法**

![image-20250305235223963](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305235223963.png)

**4）为什么两个对象有相同的hashcode值，它们也不一定是相等的**

![image-20250305235322181](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250305235322181.png)

**5）什么情况下可以不重写hashCode()**

当所在类不使用HashSet、Hashtable、HashMap等散列集合进行存储的时候，可以不使用hashcode。

## Hash碰撞

​	哈希表可以定义为是一个根据键（Key）而直接访问在内存中存储位置的值（Value）的数据结构。这里所说的键就是指哈希函数的输入，而这里的值并不是指哈希值，而是指我们想要保存的值

​	**对象Hash的前提是实现equals()和hashCode()两个方法**，那么HashCode()的作用就是保证对象返回唯一hash值，**但当两个对象计算值一样时，这就发生了碰撞冲突**

**1）再hash法(ReHash)**

基本思想：有多个不同的Hash函数，当发生冲突时，使用第二个，第三个，…，等哈希函数计算地址，直到无冲突。虽然不易发生聚集，但是增加了计算时间

**2)链地址法(HashMap采用的方法)**

​	基本思想：每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来。HashMap中hash冲突的解决办法，就是链地址法，如果当前桶上发生hash冲突，根据JDK版本不同采用头插法或尾插法将元素插入链表

## java的toString()方法

​	`toString` 方法是 Java 中 `Object` 类的一个方法，所有类都继承自 `Object` 类，因此所有对象都可以调用 `toString` 方法。该方法的主要作用是返回对象的字符串表示形式，通常用于调试和日志记录。

### 默认实现

在 `Object` 类中，`toString` 方法的默认实现返回一个字符串，包含对象的类名和哈希码（以十六进制表示）。例如：

```java
public class Object {
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
}
```

### 重写 `toString` 方法

​	**像String、Date、File、包装类等都重写了Object类中的toString()方法**。使得在调用toString()时，返回"实体内容"信息；在实际开发中，通常需要重写 `toString` 方法，以便返回更有意义的对象信息。例如：

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }

    public static void main(String[] args) {
        Person person = new Person("Alice", 30);
        System.out.println(person.toString()); // 输出: Person{name='Alice', age=30}
    }
}
```

### 使用场景

1. **调试**：在调试代码时，打印对象的 `toString` 方法可以帮助开发者快速了解对象的状态。
2. **日志记录**：在日志中记录对象信息时，`toString` 方法可以提供清晰的字符串表示。
3. **字符串拼接**：在字符串拼接操作中，`toString` 方法会自动调用，例如 `System.out.println("Person: " + person);`。

## 包装类(wrapper)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306000846032.png" alt="image-20250306000846032" style="zoom:67%;" />

### 包装类与基本数据类型相互转换

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306000957241.png" alt="image-20250306000957241" style="zoom:85%;" />

**<u>1）包装类—>基本数据类型</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306001354882.png" alt="image-20250306001354882" style="zoom: 50%;" />

**<u>2）基本数据类型=》包装类，调用包装类的构造器</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306001425884.png" alt="image-20250306001425884" style="zoom:50%;" />

**<u>3）基本数据类型、包装类=》String类型：调用String重载的==String.valueOf（Xxx xxx==）</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002312509.png" alt="image-20250306002312509" style="zoom:50%;" />

**<u>4）String类型=》基本数据类型、包装类,调用包装类的parseXxx()</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002345314.png" alt="image-20250306002345314" style="zoom:50%;" />

### JDK5.0新特性：自动装箱和自动拆箱

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306001449116.png" alt="image-20250306001449116" style="zoom:50%;" />

- **装箱**就是自动将基本数据类型转换为包装器类型；**`Integer.valueOf();`**
- **拆箱**就是自动将包装器类型转换为基本数据类型。**`intValue();直接返回value`**

总结：**装箱的过程会创建对应的对象，这个会消耗内存**，所以装箱的过程会增加内存的消耗，影响性能

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306001557124.png" alt="image-20250306001557124" style="zoom:80%;" />

**equals()**

![image-20250306001858343](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306001858343.png)

**"=="**

- **当“==”运算符的两个操作数都是包装器类型的引用，则是比较指向的是否是同一个对象，而如果其中有一个操作数是表达式（即包含算术运算）则比较的是数值（即会触发自动拆箱的过程）**。

### Integer的比较问题（缓存IntegerCache）

**new Integer(50);只要是两个new就不管缓冲的问题，说明不是一个对象**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002021810.png" alt="image-20250306002021810" style="zoom:60%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002033489.png" alt="image-20250306002033489" style="zoom:60%;" />

==Integer不能和long用\==比较，Integer和Long都有缓存==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002147444.png" alt="image-20250306002147444" style="zoom:80%;" />

### 8种基本类型的包装类和常量池

- Java基本类型的包装类的大部分都实现了常量池技术
- 即**Byte,Short,Integer,Long,Character,Boolean**；前面4种包装类默认创建了数值[-128，127]的相应类型的缓存数据，
  - Character创建了数值在[0,127]范围的缓存数据，
  - Boolean直接返回True Or False。如果超出对应范围仍然会去创建新的对象。
- 两种浮点数类型的包装类**Float,Double并没有实现常量池技术**。
- 为啥把缓存设置为[-128，127]区间？技术规范以及性能和资源之间的权衡。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306002201501.png" alt="image-20250306002201501" style="zoom:80%;" />

##  抽象类与抽象方法

abstract关键字的使用：<font color = '#8D0101'>抽象的，可修饰类、方法</font>

**1） abstract修饰类**

​	抽象类，此类不能实例化，**<font color = '#8D0101'>抽象类中一定有构造器</font>**，便于子类实例化时调用，开发中都会提供抽象类的子类，让子类对象实例化完成相关操作

**2）abstract修饰方法**

- 抽象方法，只有方法的声明，没有方法体，**包含抽象方法的类一定是一个抽象类，反之，抽象类中可以没有抽象方法**。
- 若子类**重写了父类中的所有的抽象方法后，此子类方可实例化**；若子类没有重写父类中的所有的抽象方法，则此子类也是一个抽象类需使用abstract修饰

**3）abstract使用上的注意点**

- abstract**不能用来修饰变量、代码块、构造器**
- abstract**不能用来修饰私有private方法、静态static方法、final的方法、final的类**
- abstract与**private一起使用，相互矛盾**：abstract修饰的方法是要给子类重写，而private修饰的方法只能本类访问
- abstract与**static一起使用，无意义**：abstract修饰的方法是抽象的，没有实体，而static修饰的方法，类是可以直接调用的，但是抽象方法没有方法体，调用抽象方法是没有意义的。
- abstract与**final一起使用，相互矛盾：**final修饰方法不让子类重写，而abstract修饰的方法就是为了让子类重写

## 接口（interface）

**1）使用接口的原因**

- 一方面，有时必须从几个类中派生出一个子类，继承它们所有的属性和方法，但是，Java不支持多重继承，有了接口就可以得到多重继承的效果。
- 另一方面，有时必须从几个类中抽取一些共同的行为特征，而**它们之间没有is-a的关系，仅仅是具有相同的行为特征而已**。例如：鼠标、键盘、打印机、扫描仪等都支持USB连接

**2）什么是接口**

​	接口就是规范，定义一组规则，体现了现实世界中“如果你是/要…则必须能…”的思想。而继承是一个“是不是”的关系，而接口实现则是“能不能”的关系。接口的本质是契约、标准、规范，就像法律一样，制定好后大家都要遵守。

**3）接口的特点**

- 接口中的所有**成员变量都默认是由<font color = '#8D0101'>public static final修饰</font>的**。
- 接口中的所有**抽象方法都默认是由<font color = '#8D0101'>public abstract修饰</font>的**
- 接口中没有构造器。
- 接口采用多继承机制

**4）接口的使用**

- 接口使用interface来定义
- 接口和类是并列的两个结构

- 定义接口中的成员，
  - Jdk7及之前：**<font color = '#8D0101'>全局常量public static final（书写时修饰符可以省略不写，但它还是常量）、抽象方法public abstract（这里修饰符也可以省略）</font>**
  - Jdk8：除了全局常量和抽象方法之外，还可以定义**<font color = '#8D0101'>静态方法</font>、默认方法**

- 接口中**不能定义构造器！意味着接口不能实例化。**
- 接口通过让类去实现（implements）的方式来使用；如果**实现类覆盖了接口中所有的抽象方法，则此实现类就可以实例化；如果实现类没有覆盖接口中所有的抽象方法，则此实现类仍为一个抽象类**
- Java类可以实现多个接口，弥补了Java单继承性的局限性
- 接口与接口之间是继承,而且可以多继承

**5）应用**

**代理模式和工厂模式**

### JDK8关于接口的改进

除了全局常量和抽象方法之外，还可以定义**静态方法、默认方法**

**1）静态方法**

​	使用static关键字修饰。**可以通过接口直接调用静态方法，并执行其方法体。**我们经常在相互一起使用的类中使用静态方法。你可以在标准库中找到像Collection/Collections或者Path/Paths这样成对的接口和类。

**2）默认方法**

​	默认方法使用default关键字修饰。**可以通过实现类对象来调用**。我们在已有的接口中提供新方法的同时，还保持了与旧版本代码的兼容性。比如：java 8 API中对Collection、List、Comparator等接口提供了丰富的默认方法

## 抽象类与接口的区别

- **抽象类可以实现接口**

  - **代码复用**：抽象类可以提供接口方法的**默认实现**，子类可选择覆盖或直接继承。

  - **部分实现**：抽象类可以仅实现接口中的**部分方法**，**剩余抽象方法由子类强制实现**；

    - **eg：**

      ```java
      interface Worker {
          void work();    // 需要实现的方法
          void rest();    // 需要实现的方法
      }
      
      // 抽象类仅实现部分方法
      abstract class AbstractWorker implements Worker {
          @Override
          public void rest() {
              System.out.println("工人在休息");
          }
          
          // work() 方法未实现，留给子类
      }
      
      // 子类必须实现 work()
      class Engineer extends AbstractWorker {
          @Override
          public void work() {
              System.out.println("工程师在设计");
          }
      }
      ```

  - **扩展性**：通过抽象类实现接口，可以灵活扩展功能，避免子类重复编写相同代码。（抽象类实现接口，并添加通用属性和方法）

    - **eg：**

      ```java
      interface Shape {
          void draw();
          double calculateArea();
      }
      
      // 抽象类实现接口，并添加通用属性和方法
      abstract class AbstractShape implements Shape {
          protected String color;
      
          public AbstractShape(String color) {
              this.color = color;
          }
      
          // 通用方法：获取颜色
          public String getColor() {
              return color;
          }
      }
      
      // 子类只需关注特定逻辑
      class Circle extends AbstractShape {
          private double radius;
      
          public Circle(String color, double radius) {
              super(color);
              this.radius = radius;
          }
      
          @Override
          public void draw() {
              System.out.println("画一个" + color + "的圆形");
          }
      
          @Override
          public double calculateArea() {
              return Math.PI * radius * radius;
          }
      }
      ```

- 接口使用interface修饰；抽象类使用abstract修饰。接口使用implement实现，抽象类使用extends继承。一个类可以实现多个接口，但只能继承一个抽象类，接口自己本身可以通过extends关键字扩展多个接口。

- 关于构造器，接口中没有构造器，抽象类中一定有构造器，方便实现类调用

- 关于属性，**接口中属性只能用static final修饰，抽象类中可以用其他修饰符**

- 关于方法，jdk1.7中接口只有抽象方法，jdk1.8接口加入了静态方法（只能通过接口去调用）和默认方法（通过实现类去调用），jdk1.9接入了私有方法和私有静态方法。抽象类中可以有抽象方法，也可以没有抽象方法。另外实现了接口中所有的抽象方法的类，才能是非抽象类，否则还是抽象类。**接口方法的默认修饰符是public，抽象方法可以有public、protected和default这些修饰符**（抽象方法是为了被重写所以不能用private修饰）。

- 从设计层面来说，**抽象是对类的抽象，是一种模板设计，而接口是对行为的抽象，是一种行为的规范**

- **抽象类体现继承，接口体现多态**

**使用场景**：

1）如果拥有一些方法并且想让它们中的一些有默认实现，那么使用抽象类吧。

2）如果想实现多重继承，那么你必须使用接口。由于Java不支持多继承，子类不能够继承多个类，但可以实现多个接口。

3）如果基本功能在不断改变，那么就需要使用抽象类。如果不断改变基本功能并且使用接口，那么就需要改变所有实现

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306110804426.png" alt="image-20250306110804426" style="zoom:150%;" />

## Java 内部类详解(成员内部类、静态内部/嵌套类、局部内部类、匿名内部类)

### 内部类的种类

- **成员内部类（Member Inner Class）**：
  - 定义在外部类的成员位置，可以具有任意访问修饰符（`public`、`protected`、`private` 或默认访问权限）。
  - 可以直接访问外部类的所有成员（包括私有成员）。
  - 外部类要访问内部类的成员，需要通过内部类的实例来访问。
- **静态内部类（Static Nested Class）**：
  - 使用 `static` 关键字修饰，不依赖于外部类的实例，可以直接使用外部类的静态成员，无需创建外部类对象。
  - 不能直接访问外部类的非静态成员，但可以通过外部类实例来访问。
- **局部内部类（Local Inner Class）**：
  - 定义在外部类的方法或代码块内部。
  - 只能在定义它的方法或代码块中被访问。
  - 可以访问外部类的所有成员，以及定义它的方法或代码块内的局部变量（前提是这些局部变量必须被声明为 `final`）。
- **匿名内部类**（Anonymous Inner Class）：
  - 将内部类的概念进一步简化，没有类名，直接在创建对象时定义并实例化。
  - 由于没有类名，它不能独立存在，只能与创建它的实例关联。

### 成员内部类

​	成员内部类（Member Inner Class）是Java中内部类的一种，它定义在另一个类（外部类）的成员位置，可以具有任意访问修饰符（`public`、`protected`、`private` 或默认访问权限）

**【特点】**

**访问外部类成员**：

- 成员内部类可以直接访问外部类的所有成员，包括私有成员（字段、方法和嵌套类）。这意味着内部类可以访问外部类的私有数据和受保护的功能，这有助于实现更紧密的封装和更复杂的逻辑。
- 外部类访问内部类的成员则需要通过内部类的实例来访问，如同访问其他类的对象一样。

**生命周期依赖**：

- 成员内部类的实例与外部类的实例之间存在依赖关系。**创建一个成员内部类的实例时，必须先有一个外部类的实例存在**。也就是说，**不能独立创建一个成员内部类的对象，除非它被嵌套在外部类的一个方法或代码块中，且该方法或代码块正在被外部类的一个实例调**用。

**访问修饰符**：

- 成员内部类可以有任意访问修饰符，如 `public`、`protected`、`private` 或默认（包访问权限）。访问修饰符决定了其他类能否访问这个内部类。

**不能有静态成员**：

- **成员内部类不能有静态字段、静态方法或静态初始化块，因为它本身依赖于外部类的实例**。然而，它仍然**可以使用外部类的静态成员**。

**编译后的类文件**：

- 编译成员内部类时，**编译器会生成两个独立的类文件：一个是外部类的 `.class` 文件，另一个是内部类的 `.class` 文件，其名称格式通常是 `外部类名$内部类名.class`**。

【**代码示例**】

```java
public class MemberInternalClass {
    private String outerData;

    public MemberInternalClass(String outerData) {
        this.outerData = outerData;
    }

    // 创建一个成员内部类
    class InnerClass {
        void accessOuter() {
            System.out.println("外部成员变量outerData: " + outerData);
        }
    }

    public void demonstrateInnerClass() {
        // 调用成员内部类
        InnerClass inner = new InnerClass();
        inner.accessOuter();
    }
    public static void main(String[] args) {
        MemberInternalClass memberInternalClass = new MemberInternalClass("Hello Java!");
        // 通过成员方法调用成员内部类
        memberInternalClass.demonstrateInnerClass();// 输出: 外部成员变量outerData: Hello Java!
        // 直接访问成员内部类
        MemberInternalClass.InnerClass inner = memberInternalClass.new InnerClass();
        memberInternalClass.new InnerClass().accessOuter();// 输出: 外部成员变量outerData: Hello Java!
    }
}
```

**<u>1）定义</u>**

​	在外部类的成员位置（不在任何方法或代码块内部）使用 `class` 关键字定义内部类，可以加上访问修饰符。

```java
public class OuterClass {
    private String outerData;
	// 定义一个成员内部类
    class InnerClass {
        // ...
    }
}
```

**<u>2）创建实例：</u>**

- 在外部类的方法或代码块中，通过 `new` 关键字创建内部类的实例，需要先有一个外部类的实例。

```java
OuterClass outer = new OuterClass();
OuterClass.InnerClass inner = outer.new InnerClass();
```

<u>**3）访问内部类成员**：</u>

- 外部类通过内部类实例访问其成员，如同访问其他类的对象一样。

```java
inner.someMethod();
```

<u>**4）内部类访问外部类成员**：</u>

- 内部类可以直接访问外部类的所有成员，包括私有成员。

```java
class InnerClass {
    void accessOuter() {
        System.out.println("外部类的成员变量outerData: " + outerData);
    }
}
```

【**应用场景**】

- **封装复杂逻辑**：当某个类的实现细节过于复杂，或者需要对外部类进行深度定制时，可以使用成员内部类来封装这部分逻辑，保持外部类的简洁性。
- **事件监听器**：在处理GUI事件、网络事件等场景中，经常需要创建事件监听器类。将监听器类定义为成员内部类，可以方便地访问外部类的状态和方法。
- **回调函数**：在多线程、异步处理等需要回调的场景中，成员内部类可以作为回调接口的实现类，直接访问外部类的状态，简化代码。
- **模块化设计**：当某个类的功能可以自然地划分为几个逻辑相关的部分时，可以使用成员内部类来组织这些部分，增强代码的模块化和可读性。

### 静态内部/嵌套类

​	`静态内部类`（Static Nested Class），也被称为`静态嵌套类`，是一种特殊的内部类，其声明时使用了 `static` 关键字。

<u>【**特点**】</u>

- **独立于外部类实例**：
  - 静态内部类不依赖于外部类的实例，可以独立创建其对象。**这意味着无需先创建外部类实例就能创建静态内部类的实例**。
- **访问外部类成员**：
  - **静态内部类只能访问外部类的静态成员（包括静态字段、静态方法和嵌套静态类），不能直接访问外部类的实例成员（非静态字段和非静态方法**）。若**需要访问实例成员，通常需要通过传递外部类实例作为参数或在内部类中持有外部类实例引用。**
- **静态成员**：
  - **静态内部类可以拥有自己的静态成员（静态字段、静态方法和静态初始化块）**，这些静态成员遵循普通类中静态成员的规则。
- **编译后的类文件**：
  - **同成员内部类一样，静态内部类也会生成单独的 `.class` 文件，文件名通常为 `外部类名$静态内部类名.class`。**
- **访问修饰符**：
  - 静态内部类可以使用任意访问修饰符（`public`、`protected`、`private` 或默认包访问权限），控制其他类对其的访问。

<u>**【代码示例】**</u>

```java
public class OuterClass {
    private static String staticData = "Static Data";
    // 定义静态内部类
    static class StaticNestedClass {
        void accessStatic() {
            System.out.println("静态成员变量: " + staticData);
        }
    }
}

public static void main(String[] args) {
    // 调用静态内部类
    OuterClass.StaticNestedClass inner = new OuterClass.StaticNestedClass();
    inner.accessStatic();
}
```

```java
import java.util.ArrayList;
import java.util.List;

public class Solution {
    public static class TreeNode {
        int val;
        TreeNode left;
        TreeNode right;
        TreeNode (int val){
            this.val = val;
        }
        TreeNode(int val, TreeNode left, TreeNode right){
            this.val = val;
            this.left = left;
            this.right = right;
        }
    }
    
    private int index = 0;
    public static void main(String[] args) {
        Solution solution = new Solution();
        Integer[] preorder = {1, 3, 4, null, 6, null, null, null, 8};
        TreeNode root = solution.buildTree(preorder);
        // 中序遍历
        List<Integer> inorderResult = solution.inorderTraversal(root);
        System.out.println(inorderResult);
    }

    public TreeNode buildTree(Integer[] preorder) {
        if (index >= preorder.length || preorder[index] == null) {
            index++;
            return null;
        }
        TreeNode root = new TreeNode(preorder[index++]);
        root.left = buildTree(preorder);
        root.right = buildTree(preorder);

        return root;
    }
    // 中序遍历
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        inorderHelper(root, result);
        return result;
    }

    private void inorderHelper(TreeNode node, List<Integer> result) {
        if (node == null) {
            return;
        }
        inorderHelper(node.left, result);  // 遍历左子树
        result.add(node.val);             // 访问根节点
        inorderHelper(node.right, result); // 遍历右子树
    }
}
```

<u>【**使用方式**】</u>

**定义**：

- 在外部类的成员位置，使用 `static class` 关键字定义静态内部类。

```java
public class OuterClass {
    // 创建静态内部类
    static class StaticInnerClass {
        // ...
    }
}
```

**创建实例**：

- 与创建普通类对象类似，直接使用 `new` 关键字创建静态内部类的实例，无需外部类实例。

```java
OuterClass.StaticInnerClass inner = new OuterClass.StaticInnerClass();
```

**访问内部类成员**：

- 通过静态内部类的实例访问其成员。

```java
inner.someMethod();
```

**内部类访问外部类成员**：

- 静态内部类通过外部类的类名访问其静态成员。

```java
class StaticInnerClass {
    void accessOuter() {
        System.out.println("外部静态常量: " + OuterClass.outerStaticData);
    }
}
```

【**应用场景**】

- **工具类或辅助类**：当需要定义一组仅与某个外部类相关的工具方法或常量时，可以使用静态内部类来封装这些功能，避免污染外部类或创建不必要的顶级类。
- **单例模式**：静态内部类可以用于实现线程安全的懒汉式单例模式，利用类加载机制确保单例的唯一性。
- **封闭数据结构**：静态内部类可以用来封装外部类的数据结构，提供安全的访问接口，防止外部直接修改数据。
- **枚举类的替代**：在不支持枚举的早期Java版本中，静态内部类常被用来模拟枚举的行为，每个内部类实例代表一个枚举值。

### 局部内部类

​	`局部内部类`是一种特殊的内部类，它被定义在一个方法、代码块或其他局部作用域内。这种设计允许将类的定义限制在非常特定的范围中，只在需要的地方创建和使用。

【**特点**】

- **访问权限与可见性**： 局部内部类对外部不可见，即不能从外部类的其他方法或外部其他类中直接访问。它只能在其定义的局部作用域内被创建和使用。由于其局限性，局部内部类没有访问修饰符（如 `public`、`private` 等）。
- **访问外部资源**： 尽管作用域有限，局部内部类却可以访问其所在作用域内的所有局部变量，前提是这些变量必须是 `final` 或实际上 `final`（即在局部内部类的生命周期内其值不会改变）。此外，局部内部类还可以访问外部类的所有成员（包括私有成员），这一点与非局部内部类一致。
- **生命周期与垃圾回收**： 局部内部类对象的生命周期与其依赖的局部变量密切相关。如果一个局部内部类对象引用了其外部方法的局部变量，那么只要这个局部内部类对象是可达的，所引用的局部变量就不会被垃圾回收，即使该方法已经执行完毕。这是为了确保内部类对象能够正确访问到这些变量的值。

【**使用方式**】

​	局部内部类定义在外部类的一个方法、循环、条件语句等局部作用域内。它的声明和实现与其他内部类一样，包含类的属性、方法、构造器等组件，**但其作用域仅限于定义它的局部区域**。

```java
public class OuterClass {
    public void someMethod() {
        // 局部内部类定义在此处
        class LocalInnerClass {
            // 属性、方法、构造器等...
        }
        
        // 在此作用域内可以创建和使用 LocalInnerClass 的实例
        LocalInnerClass localInstance = new LocalInnerClass();
    }
}
```

【**代码示例**】

```java
public class Demo03_LocalInnerClass {

    private String sharedField = "外部类定义";
    public void processData() {
        final String importantValue = "外部类方法的变量"; // 必须为 final 或 effectively final

        // 定义局部内部类
        class DataProcessor {
            void doProcessing() {
                System.out.println("外部类方法中的局部变量: " + importantValue);
                // 可以访问外部类成员
                System.out.println("外部类成员变量: " + Demo03_LocalInnerClass.this.sharedField);
            }
        }

        // 创建并使用局部内部类实例
        DataProcessor processor = new DataProcessor();
        processor.doProcessing();
    }
    public static void main(String[] args) {
        // 调用非静态方法，间接调用局部内部类
        Demo03_LocalInnerClass method = new Demo03_LocalInnerClass();
        method.processData();
    }
}
```

【**应用场景**】

局部内部类通常用于解决一些特定的编程问题，如：

- **临时性的类需求**：当某个功能逻辑仅在特定方法中需要，并且该功能需要封装成类，但又无需在整个类层次结构中公开时，可以使用局部内部类。
- **事件处理回调**：在实现事件驱动模型时，有时需要为特定事件创建一个匿名或局部内部类的监听器，这些监听器只在注册事件时创建，并在事件触发时调用其方法。
- **异常处理**：有时为了简化异常处理逻辑，可以定义一个局部内部类来封装特定类型的异常及其处理逻辑。
- **适配器模式**：在需要快速创建一个满足特定接口的适配器对象时，局部内部类可以提供简洁的实现方式。

### 匿名内部类

​	`匿名内部类`（Anonymous Inner Class）是Java中一种特殊的内部类，它没有名字，即在定义时不需要使用 `class` 关键字为其命名。

- 匿名内部类主要用于简化代码，特别是在只需要使用一次某个类的实例，且该类的实现相对简单的情况下。
- 匿名内部类通常用于实现接口或继承抽象类，并在创建该类实例的同时定义其实现。

【**特点**】

- **无名称**：
  - **匿名内部类没有名称，定义时直接省略了类名**。因此，无法在其他地方再次引用或创建该类的实例。
- **继承或实现**：
  - **匿名内部类必须继承一个父类（通常为抽象类）或实现一个接口**。这是匿名内部类存在的主要目的，即在创建对象时同时提供具体的实现。
- **一次性使用**：
  - **由于匿名内部类没有名称，所以它通常伴随着 `new` 关键字一起使用，创建并初始化一个该类的实例**。这种“创建即使用”的特性使得匿名内部类适用于那些只需一次性使用的场景。
- **访问外部资源**：
  - 匿名内部类可以访问其外部类的所有成员（包括私有成员），以及其所在方法的 `final` 或 effectively `final` 局部变量。

【**应用场景**】

- **事件监听器**：在处理GUI事件、网络事件等场景中，匿名内部类常用于快速创建事件监听器的实例，提供事件触发时的处理逻辑。
- **Lambda表达式的前身**：在Java 8之前，匿名内部类是实现函数式编程风格的主要手段，如集合排序、流操作等。现在许多这样的场景已被Lambda表达式取代。
- **一次性使用的策略或算法**：当某个策略或算法只需使用一次，且实现较为简单时，可以用匿名内部类实现，避免创建多余的命名类。

**【代码示例】**

```java
WorkerInterface work = new WorkerInterface() {  // 匿名内部类实现接口
    @Override
    public void doWork() {
        System.out.println("匿名内部类实现");
    }
};
work.doWork();
```

```java
@FunctionalInterface
public interface WorkerInterface {
    void doWork();  // 单抽象方法
}

// Lambda 实现
WorkerInterface work = () -> System.out.println("Lambda 实现");
work.doWork();
```

**==监听器==**

在文件下载场景中，下载完成会触发“下载完成”事件。

- 在这个过程中，你不需要手动调用某个函数（如 `click` 或 `downloadComplete`），而是通过注册监听器的方式，让系统在事件发生时自动调用你的逻辑。
- `FileDownloader` 类负责模拟文件下载的过程。
- `setListener` 方法用于注册监听器。
- 在下载完成后，调用监听器的 `onDownloadComplete` 方法，通知外部下载已完成。

```java
public interface DownloadListener {
    void onDownloadComplete(String fileName);
}

public class FileDownloader {
    private DownloadListener listener;

    // 注册监听器
    public void setListener(DownloadListener listener) {
        this.listener = listener;
    }
    // 模拟文件下载过程
    public void downloadFile(String fileName) {
        System.out.println("Downloading file: " + fileName);

        // 模拟耗时操作（如网络下载）
        try {
            Thread.sleep(3000); // 假设下载需要 3 秒
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 下载完成后触发监听器
        if (listener != null) {
            listener.onDownloadComplete(fileName);
        }
    }
    public static void main(String[] args) {
        // 创建文件下载器
        FileDownloader downloader = new FileDownloader();

        // 使用匿名内部类注册监听器
        downloader.setListener(new DownloadListener() {
            @Override
            public void onDownloadComplete(String fileName) {
                System.out.println("File downloaded: " + fileName);
            }
        });
        //downloader.setListener(fileName -> System.out.println("File downloaded: " + fileName));
        // 开始下载文件
        downloader.downloadFile("example.txt");
    }
}
```

==**匿名内部类作为回调**==

**场景：异步任务完成后的回调**

​	在异步编程中，回调函数用于通知调用者某个任务已完成。以下是一个模拟异步任务完成后触发回调的例子。

- 定义了一个 `Callback` 接口，包含一个 `onComplete` 方法。
- `AsyncTask` 类模拟了一个异步任务，通过 `performTask` 方法接收一个回调函数。
- 在 `main` 方法中，使用匿名内部类实现了 `Callback` 接口，并定义了任务完成时的行为。

```java
public interface Callback {
    void onComplete(String result);
}

public class AsyncTask {
    // 模拟一个异步任务
    public void performTask(Callback callback) {
        new Thread(() -> {
            try {
                // 模拟耗时操作
                Thread.sleep(2000);
                String result = "Task Completed";
                // 调用回调函数
                callback.onComplete(result);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
    public static void main(String[] args) {
        AsyncTask asyncTask = new AsyncTask();

        // 使用匿名内部类实现回调接口
        asyncTask.performTask(new Callback() {
            @Override
            public void onComplete(String result) {
                System.out.println("Callback received: " + result);
            }
        });

        System.out.println("Waiting for task to complete...");
    }
}
```



## 匿名内部类和具体内部类

匿名内部类是没有**显式名称的局部内部类**，通常在创建实例的同时定义并实例化。

eg：`new WorkerInterface() { ... }`实际上是在定义一个**实现了`WorkerInterface`接口的匿名类**，并立即创建其实例。**==虽然`work`变量是该实例的引用，但匿名内部类本身没有类名，因此称为“匿名”==**。

**匿名内部类的特点**

| **特性**       | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| **无显式类名** | 类定义和实例化一步完成，无需单独的类声明。                   |
| **单次使用**   | 通常用于即用即弃的场景，无需复用类定义。                     |
| **内存占用**   | **==每次调用都会生成一个新的类（编译后生成 `.class` 文件，如 `WorkTest$1.class`）==。** |

**1. 匿名性体现在类定义**

```java
WorkerInterface work = new WorkerInterface() {  // 匿名内部类定义
    @Override
    public void doWork() {
        System.out.println("通过匿名内部类调用");
    }
};
```

- **匿名内部类的本质**：

  - 这是一个 **没有显式类名** 的类定义。
  - 你正在创建一个 `WorkerInterface` 接口的 **匿名实现类**，并立即实例化它。
  - **==尽管 `work` 变量持有该实例的引用，但这个实例所属的类 没有名字，因此称为“匿名”==**。

**2.具名内部类（实现类）**

可以使用实现类进行调用

```java
// 具名内部类
class MyWorker implements WorkerInterface {
    @Override
    public void doWork() {
        System.out.println("通过具名内部类调用");
    }
}

// 使用具名内部类
WorkerInterface work = new MyWorker();
```

**3.如果没有实现类**

如果 `WorkerInterface` 是一个 **接口**（且没有被其他类显式实现），直接通过 `new WorkerInterface()` 实例化会编译失败，**因为接口无法直接实例化**。

要使用接口，必须通过以下两种方式之一：

**(1) 匿名内部类**

```java
WorkerInterface work = new WorkerInterface() {  // 匿名内部类实现接口
    @Override
    public void doWork() {
        System.out.println("匿名内部类实现");
    }
};
work.doWork();
```

**(2) Lambda 表达式（仅限函数式接口）**

```java
@FunctionalInterface
public interface WorkerInterface {
    void doWork();  // 单抽象方法
}

// Lambda 实现
WorkerInterface work = () -> System.out.println("Lambda 实现");
work.doWork();
```

若使用 Lambda 表达式（仅适用于函数式接口）：

- **匿名内部类会生成新的 `.class` 文件。**
- ==**Lambda 表达式通过 `invokedynamic` 指令动态生成实现，没有额外的类文件。**==

## 面向接口编程：List\<string> list = new ArrayList<>()

体现了多态和泛型的设计思想

**（1）代码分解**

```java
List<String> list = new ArrayList<>();
```

| **部分**            | **作用**                                                     |
| ------------------- | ------------------------------------------------------------ |
| `List<String>`      | 声明一个 `list` 变量，类型为 `List` 接口，泛型限定元素为 `String`。 |
| `new ArrayList<>()` | 实例化一个 `ArrayList` 对象（`List` 的具体实现类），菱形运算符 `<>` 自动推断泛型类型。 |
| `=`                 | 将 `ArrayList` 实例的引用赋值给 `list` 变量。                |

**（2）面向接口编程**

- List是接口：List定义了列表的基本操作（add, `get`, `remove`），但不提供具体实现。

- **`ArrayList` 是实现类**：`ArrayList` 是 `List` 接口的基于数组的实现。

- 为什么用接口类型声明变量

  提高代码灵活性和可维护性。后续可随时更换实现类，如 LinkedList而无需修改其他代码：

  ```java
  // 更换实现类只需修改右侧
  List<String> list = new LinkedList<>();
  ```

**(3) 泛型（Generics）**

- **`List<String>`**：通过泛型限定列表元素为 `String` 类型，确保类型安全。
- **菱形运算符 `<>`**（Java 7+）：
  编译器自动推断泛型类型，等价于 `new ArrayList<String>()`。

## 异常

### 异常体系结构

在Java中，所有的异常都有一个**共同的祖先java.lang包中的Throwable类**。Throwable有两个重要的子类：

- **<font color = '#8D0101'>Error</font>**：Java虚拟机无法解决的严重问题，大多数错误与代码编写者执行的操作无关，而表示代码运行时Java虚拟机出现的问题。如：**StackOverFlow和OOM**，一般不编写针对性的代码处理，这些异常发生时，Java虚拟机一般会选择线程终止。
- **<font color = '#8D0101'>Exception</font>**：其他因编程错误或偶然的外在因素导致的一般性问题，Exception。如：空指针访问、试图读取不存在的文件、网络连接中断、数组脚标越界

**异常和错误的区别：异常能被程序本身处理，错误是无法处理的**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/1553752019030.png" alt="img" style="zoom:80%;" />

异常分类：**<font color = '#8D0101'>编译时异常和运行时异常</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306111050553.png" alt="image-20250306111050553" style="zoom:120%;" />

<u>**1）运行时异常**</u>

- 是指编译器不要求强制处置的异常。一般是指编程时的逻辑错误，是程序员应该积极避免其出现的异常。**java.lang.RuntimeException类及它的子类都是运行时异常。**
- 对于这类异常，可以不作处理，因为这类异常很普遍，若全处理可能会对程序的可读性和运行效率产生影响。

<u>**2）编译时异常**</u>

- 是指编译器要求必须处置的异常。即程序在运行时由于外界因素造成的一般性异常。编译器要求Java程序必须捕获或声明所有编译时异常
- 对于这类异常，如果程序不处理，可能会带来意想不到的结果

### 异常处理机制

​	“抛”：程序在正常执行的过程中，一旦出现异常，就会在异常代码处生成一个对应异常类的对象，并将此对象抛出。一旦抛出对象以后，其后的代码就不再执行：**throw**

​	“抓”:可以理解为异常的处理方式：**①try-catch-finally ②throws**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306111328610.png" alt="image-20250306111328610" style="zoom:120%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/1553752795010.png" alt="img" style="zoom:90%;" />

#### 自定义异常

**自定义一个异常类，只需要继承 `Exception` 或 `RuntimeException` 即可。**

示例：

```java
public class MyExceptionDemo {
    public static void main(String[] args) {
        throw new MyException("自定义异常");
    }
    static class MyException extends RuntimeException {
        public MyException(String message) {
            super(message);
        }
    }
}
```

输出：

```text
Exception in thread "main" io.github.dunwu.javacore.exception.MyExceptionDemo$MyException: 自定义异常
	at io.github.dunwu.javacore.exception.MyExceptionDemo.main(MyExceptionDemo.java:9)
```

#### 抛出异常

如果想在程序中明确地抛出异常，需要用到 `throw` 和 `throws` 。

==**throw 和 throws 的区别：**==

- **throws 使用在函数上，throw 使用在函数内。**
- **throws 后面跟异常类，可以跟多个，用逗号区别；throw 后面跟的是异常对象。**

**<u>（1）throw</u>**

可以使用 `throw` 关键字抛出一个异常，无论它是新实例化的还是刚捕获到的。

`throw` 示例：

```java
public class ThrowDemo {
    public static void f() {
        try {
            throw new RuntimeException("抛出一个异常");
        } catch (Exception e) {
            System.out.println(e);
        }
    }

    public static void main(String[] args) {
        f();
    }
};
```

输出：

```text
java.lang.RuntimeException: 抛出一个异常
```

**<u>（2）throws</u>**

如果一个方法没有捕获一个检查性异常，那么该方法必须使用 `throws` 关键字来声明。`throws` 关键字放在方法签名的尾部。

`throws` 示例：

```java
public class ThrowsDemo {
    public static void f1() throws NoSuchMethodException, NoSuchFieldException {
        Field field = Integer.class.getDeclaredField("digits");
        if (field != null) {
            System.out.println("反射获取 digits 方法成功");
        }
        Method method = String.class.getMethod("toString", int.class);
        if (method != null) {
            System.out.println("反射获取 toString 方法成功");
        }
    }
    public static void f2() {
        try {
            // 调用 f1 处，如果不用 try catch ，编译时会报错
            f1();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        f2();
    }
};
```

输出：

```text
反射获取 digits 方法成功
java.lang.NoSuchMethodException: java.lang.String.toString(int)
	at java.lang.Class.getMethod(Class.java:1786)
	at io.github.dunwu.javacore.exception.ThrowsDemo.f1(ThrowsDemo.java:12)
	at io.github.dunwu.javacore.exception.ThrowsDemo.f2(ThrowsDemo.java:21)
	at io.github.dunwu.javacore.exception.ThrowsDemo.main(ThrowsDemo.java:30)
```

#### 捕获异常

**使用 try 和 catch 关键字可以捕获异常**。try catch 代码块放在异常可能发生的地方。

它的语法形式如下：

```java
try {
    // 可能会发生异常的代码块
} catch (Exception e1) {
    // 捕获并处理try抛出的异常类型Exception
} catch (Exception2 e2) {
    // 捕获并处理try抛出的异常类型Exception2
} finally {
    // 无论是否发生异常，都将执行的代码块
}
```

此外，JDK7 以后，`catch` 多种异常时，也可以像下面这样简化代码：

```java
try {
    // 可能会发生异常的代码块
} catch (Exception | Exception2 e) {
    // 捕获并处理try抛出的异常类型
} finally {
    // 无论是否发生异常，都将执行的代码块
}
```

- `try` - **`try` 语句用于监听。将要被监听的代码(可能抛出异常的代码)放在 `try` 语句块之内，当 `try` 语句块内发生异常时，异常就被抛出。**
- `catch` - `catch` 语句包含要**捕获异常类型的声明**。当保护代码块中发生一个异常时，`try` 后面的 `catch` 块就会被检查。
- `finally` - **`finally` 语句块总是会被执行，无论是否出现异常。**`try catch` 语句后不一定非要`finally` 语句。`finally` 常用于这样的场景：由于`finally` 语句块总是会被执行，所以那些在 `try` 代码块中打开的，并且必须回收的物理资源(如数据库连接、网络连接和文件)，一般会放在`finally` 语句块中释放资源。
- `try`、`catch`、`finally` 三个代码块中的局部变量不可共享使用。
- `catch` 块尝试捕获异常时，**是按照 `catch` 块的声明顺序从上往下寻找的，一旦匹配，就不会再向下执行**。因此，如果同一个 `try` 块下的多个 `catch` 异常类型有父子关系，应该将子类异常放在前面，父类异常放在后面。

#### 打印异常信息

​	异常类的基类Exception中提供了一组方法用来获取异常的一些信息.所以如果我们获得了一个异常对象,那么我们就可以打印出一些有用的信息。

- **<font color = '#8D0101'>最常用的就是`void printStackTrace()`这个方法</font>**，这个方法将**返回一个由栈轨迹中的元素所构成的数组**,其中每个元素都表示栈中的一帧.**元素0是栈顶元素,并且是调用序列中的<font color = '#8D0101'>最后一个方法调用(这个异常被创建和抛出之处)</font>**;他有几个不同的重载版本,可以将信息输出到不同的流中去.下面的代码显示了如何打印基本的异常信息:

```java
public void f() throws IOException{
    System.out.println("Throws SimpleException from f()"); 
    throw new IOException("Crash");
 }
 public static void main(String[] agrs) {
    try {
    	new B().f();
    } catch (IOException e) {
    	  System.out.println("Caught  Exception");
        System.out.println("getMessage(): "+e.getMessage());
        System.out.println("getLocalizedMessage(): "+e.getLocalizedMessage());
        System.out.println("toString(): "+e.toString());
        System.out.println("printStackTrace(): ");
        e.printStackTrace(System.out);
    }
}
```

输出：

```
Throws SimpleException from f()
Caught  Exception
getMessage(): Crash
getLocalizedMessage(): Crash
toString(): java.io.IOException: Crash
printStackTrace(): 
java.io.IOException: Crash
	at com.learn.example.B.f(RunMain.java:19)
	at com.learn.example.RunMain.main(RunMain.java:26)
```

#### 异常链

异常链是以一个异常对象为参数构造新的异常对象，新的异常对象将包含先前异常的信息。

通过使用异常链，我们可以提高代码的可理解性、系统的可维护性和友好性。

- 我们有两种方式处理异常，一是 `throws` 抛出交给上级处理，二是 `try…catch` 做具体处理。**<font color = '#8D0101'>`try…catch` 的 `catch` 块我们可以不需要做任何处理，仅仅只用 throw 这个关键字将我们封装异常信息主动抛出来。然后在通过关键字 `throws` 继续抛出该方法异常</font>。它的上层也可以做这样的处理，以此类推就会产生一条由异常构成的异常链。**

我们捕获异常以后一般会有两种操作

- 捕获后抛出原来的异常，希望保留最新的异常抛出点－－fillStackTrace
- **捕获后抛出新的异常，希望抛出完整的异常链－－<font color = '#8D0101'>`initCause`</font>**

<u>**（1）捕获异常后重新抛出异常**</u>

在函数中捕获了异常，在catch模块中不做进一步的处理，而是向上一级进行传递 catch(Exception e){ throw e;}，我们通过例子来看一下：

```java
public class ReThrow {
    public static void f()throws Exception{
        throw new Exception("Exception: f()");
    }

    public static void g() throws Exception{
        try{
            f();
        }catch(Exception e){
            System.out.println("inside g()");
            throw e;
        }
    }
    public static void main(String[] args){
        try{
            g();
        }
        catch(Exception e){
            System.out.println("inside main()");
            e.printStackTrace(System.out);
        }
    }
}
```

我们来看输出：

```
 代码解读
复制代码inside g()
inside main()
java.lang.Exception: Exception: f()
        //异常的抛出点还是最初抛出异常的函数f()
	at com.learn.example.ReThrow.f(RunMain.java:5)
	at com.learn.example.ReThrow.g(RunMain.java:10)
	at com.learn.example.RunMain.main(RunMain.java:21)
```

**<u>==（2）捕获异常后抛出新的异常（保留原来的异常信息，区别于捕获异常之后重新抛出）==</u>**

如果我们在抛出异常的时候需要保留原来的异常信息，那么有两种方式

- **方式1：Exception e＝new Exception(); e.initCause(ex);**

- **<font color = '#8D0101'>方式2</font>：Exception e =new Exception("${message}", ex);（**

  - ==可以看出两个参数：message和cause，其中调用了super(message, cause);，实际上就是调用了initCause()方法==

    ```java
    public Exception(String message, Throwable cause) {
        super(message, cause);
    }
    ```

示例：

```java
public class main {
    public void f() throws Exception {
        try {
            g();
        } catch (NullPointerException ex) {
            //方式1
//            Exception e = new Exception();
            //将原始的异常信息保留下来
//            e.initCause(ex);
            //方式2
            Exception e=new Exception("Test Exception",ex);
            throw e;
        }
    }
    public void g() throws NullPointerException {
        System.out.println("inside g()");
        throw new NullPointerException();
    }

    public static void main(String[] agrs) {
        try {
            new main().f();
        } catch (Exception e) {
            System.out.println("inside main()");
            e.printStackTrace();
        }
    }
}

```

结果：

![image-20250306160318752](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306160318752.png)

### Finally不执行？

1）可能程序直接崩了，就是那种非正常退出程序，比如断电了

2）在执行finally中的代码前，程序已经退出了JVM。System.exit(0);

![image-20250306160621420](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250306160621420.png)

3）try catch语句有无限循环的代码

### try-with-resources与try-catch

- `try-with-resources` 的资源必须直接放在 `try()` 的括号内，这是 Java 语法强制要求的
- 使用 `;` 分隔多个资源，无需额外 `{}`

**传统写法（需要手动关闭资源）**

```java
Socket socket = null;
try {
    socket = new Socket(serverAddress, port);
    // 使用资源
} finally {
    if (socket != null) socket.close(); // 需要手动关闭
}
```

**try-with-resources（自动关闭）**

```java
try (Socket socket = new Socket(serverAddress, port)) {
    // 使用资源
} // 自动关闭 socket
```

```java
try (
    Socket socket = new Socket(serverAddress, port);
    PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
    BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
    BufferedReader stdIn = new BufferedReader(new InputStreamReader(System.in))
) {
    // 使用资源的代码
}
```

## 字符串类String,StringBuffer,StringBuilder

### String

#### String的不可变性

我们先来看下 `String` 的定义：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
```

1）String实现了Serializable接口：表示字符串是**支持序列化**的；实现了Comparable接口：表示String可以比较大小；

2）String声明为final的，代表不可变的字符序列也不可被继承；`String` 类的数据存储于 `char[]` 数组，这个数组被 `final` 关键字修饰，表示 **`String` 对象不可被更改**。体现：

- **当对字符串重新赋值时，需要重新指定内存区域赋值**，不能使用原有的value进行赋值；
- 当调用String的replace()方法修改指定字符或字符串时，也需要重新指定内存区域

#### String的内存分配

- **保证 String 对象安全性**。避免 String 被篡改。

- **保证 hash 值不会频繁变更**。

- **可以实现字符串常量池**。通常有两种创建字符串对象的方式，

  - 一种是通过**字符串常量**的方式创建，如 `String str="abc"`，字符串常量池中是不会存储相同内容的字符串的；

  - 另一种是**字符串变量通过 new 形式**的创建，如 `String str = new String("abc")`。

  - 使用**字符串常量**方式创建字符串对象时，JVM 首先会检查该对象是否在字符串常量池中，如果在，就返回该对象引用，否则新的字符串将在常量池中被创建。这种方式可以减少同一个值的字符串对象的重复创建，节约内存。

  - `String str = new String("abc")` 这种方式，首先在编译类文件时，`"abc"` 常量字符串将会放入到常量结构中，在类加载时，`"abc"` 将会在常量池中创建；其次，在调用 new 时，JVM 命令将会调用 `String` 的构造函数，同时引用常量池中的 `"abc"` 字符串，在堆内存中创建一个 `String` 对象；最后，str 将引用 `String` 对象。

    <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307121127278.png" alt="image-20250307121127278" style="zoom:120%;" />

#### 字符串的拼接与分割、intern()方法

**<u>1）拼接</u>**

- 常量与常量的拼接结果在常量池。且常量池中不会存在相同内容的常量。
- 只要其中有一个是变量，结果就在堆中
- 如果拼接的结果调用**intern()方法**，返回值就在**常量池**中：
  - **在每次赋值的时候使用 `String` 的 `intern` 方法，如果常量池中有相同值，就会重复使用该对象，返回对象引用，这样一开始的对象就可以被回收掉**。
  - **<font color = '#8D0101'>在字符串常量中，默认会将对象放入常量池；在字符串变量中，对象是会创建在堆内存中，同时也会在常量池中创建一个字符串对象，复制到堆内存对象中，并返回堆内存对象引用。</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307121733245.png" alt="image-20250307121733245" style="zoom:150%;" />

- **String的+拼接操作，真正实现的原理**：

  - **使用StringBuilder：是通过建立临时的StringBuilder对象，然后调用append方法实现，然后调用StringBuilder的toString()方法将StringBuilder对象转化为String对象，这个对象是新new出来的对象，保存在堆中.**

  - **若拼接符两边都是字符串常量或常量引用，则是编译期优化**

    <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307122044016.png" alt="image-20250307122044016" style="zoom:150%;" />

    final修饰后，s2是一个常量，String s =s2+“a”被JVM优化为String s3 =“hello”+“a”，这个结果是编译期就已知的，指向常量池中的helloa，也就是s1。

**<u>2）字符串分割</u>**

​	**`String` 的 `split()` 方法使用正则表达式实现其强大的分割功能**。而正则表达式的性能是非常不稳定的，使用不恰当会引起回溯问题，很可能导致 CPU 居高不下。

​	所以，应该慎重使用 `split()` 方法，**可以考虑用 `String.indexOf()` 方法代替 `split()` 方法完成字符串的分割**。如果实在无法满足需求，你就在使用 Split() 方法时，对回溯问题加以重视就可以了。

#### new String(“a”) + new String(“b”) 会创建几个对象？

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250313214433780.png" alt="image-20250313214433780" style="zoom:50%;" />

### StringBuffer和StringBuilder

![image-20250307122633118](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307122633118.png)

![image-20250307122840828](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307122840828.png)

![image-20250307122853763](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307122853763.png)

![image-20250307122937224](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307122937224.png)

- **java.lang.StringBuffer**：可变的字符序列（StringBuffer继承自AbstractStringBuilder类，该类中的byte[]没有用final修饰），可以对字符串内容进行增删，此时不会产生新的对象。线程安全的，效率低、底层使用byte[]存储。
- **java.lang.StringBuilder**：可变的字符序列（同StringBuffer），线程不安全的，效率高，底层使用byte[]存储。
- StringBuffer和StringBuilder**初始长度都是16，默认扩容为原来的容量的2倍+2**
- **StringBuffer对方法加了同步锁或者对调用的方法加了同步锁（方法前加了Synchronized关键字），所以是线程安全的。StringBuilder并没有对方法进行加同步锁，所以是非线程安全的**

### String、StringBuffer、StringBuilder 有什么区别

**<u>1）继承：</u>**

- 都是final类，都不允许被继承。

**<u>2）可变：</u>**

- **String类中使用final关键字修饰的字符数组来保存字符串**，private final char[] value，所以String对象是**<font color = '#8D0101'>不可变的</font>**。在Java9之后，String类的实现改用byte数组存储字符串private final byte[] value;
- 而StringBuilder与StringBuffer都继承自AbstractStringBuilder类，在AbstractStringBuilder中也是**使用字符数组保存字符串char[]value，但是没有用final关键字修饰，所以这两种对象都是<font color = '#8D0101'>可变的</font>**。

**<u>3）线程安全</u>**

- String中的对象是不可变的，**也就可以理解为常量，线程安全。**
- AbstractStringBuilder是StringBuilder与StringBuffer的公共父类，定义了一些字符串的基本操作，如expandCapacity、append、insert、indexOf等公共方法。**StringBuffer对方法加了同步锁或者对调用的方法加了同步锁（方法前加了Synchronized关键字）**，**所以是线程安全的。StringBuilder并没有对方法进行加同步锁，所以是非线程安全的**。

**<u>4）性能：</u>**

- 每次对String类型进行改变的时候，都会生成一个新的String对象，然后将指针指向新的String对象。
- StringBuffer每次都会对StringBuffer对象本身进行操作，而不是生成新的对象并改变对象引用。相同情况下使用StringBuilder相比使用StringBuffer仅能获得10%~15%左右的性能提升，但却要冒多线程不安全的风险。
- 对比String、StringBuffer、StringBuilder三者的效率：从高到低排列：StringBuilder>StringBuffer>String

**除非有线程安全的需要，不然一般都使用 `StringBuilder`**。

## Java比较器

![image-20250307145928437](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307145928437.png)

### Comparable接口

​	Comparable**是java.lang包下的接口，是个内比较器**，实现了Comparable接口的类可以和自己比较的，依赖compareTo方法实现，compareTo方法被称为**自然排序方法**。

![image-20250307150136554](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307150136554.png)

**Comparable接口的使用：**

1）String、包装类等实现了Comparable接口，重写了compareTo()方法，给出了比较两个对象大小的方式，进行了从小到大的排列。

2）重写compareTo(obj)的规则：

- 如果当前对象this大于形参对象obj，则返回正整数，
- 如果当前对象this小于形参对象obj，则返回负整数，
- 如果当前对象this等于形参对象obj，则返回零

```java
String a1 = "a";
String a2 = "c";        
System.out.println(a1.compareTo(a2));//结果为-2
```

3）对于自定义类来说，如果需要排序，我们可以让自定义类实现Comparable接口，重写compareTo(obj)方法。

![image-20250307150555647](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307150555647.png)

### 使用Comparator实现定制排序

**Comparator是java.util包的接口，是个外比较器。**

- 当一个对象不支持自己和自己比较（没有实现Comparable接口）又想对两个对象进行比较；或者实现了Comparable接口，但是认为compareTo方法不是自己想要的。在这两种情况下可以实现Comparator接口。
- Comparator接口里有一个compare方法，有T o1和T o2两个参数，o1>o2，返回正整数，小于时候，返回负整数，等于时候，返回0。
  - 可以将Comparator传递给sort方法（如Collections.sort或Arrays.sort）从而允许在排序顺序上实现精确控制。
  - 还可以使用Comparator来控制某些数据结构（如有序set或有序映射）的顺序，或者为那些没有自然顺序的对象集合提供排序


```java
import java.util.Comparator;
 
public class MyComparator implements Comparator<Student> {  //实现比较器
  
	@Override
	public int compare(Student stu1, Student stu2) {
		// TODO Auto-generated method stub
		if(stu1.getAge()>stu2.getAge()){
			return 1;
		}else if(stu1.getAge()<stu2.getAge()){
			return -1;
		}else{
			return 0;
		}
	}
}

public class T {
	public static void main(String[] args) throws Exception{
		Student stu[] = {new Student("张三",23)
						,new Student("李四",26)
						,new Student("王五",22)};
		Arrays.sort(stu,new MyComparator());             //对象数组进行排序操作
		
		List<Student> list = new ArrayList<Student>();
		list.add(new Student("zhangsan",31));
		list.add(new Student("lisi",30));
		list.add(new Student("wangwu",35));
		Collections.sort(list,new MyComparator());      //List集合进行排序操作
		
	}
}
```

## 泛型

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Java泛型.svg" alt="img" style="zoom:75%;" />

### 泛型概述、为什么需要泛型

​	泛型就是允许在定义类、接口时通过一个标识表示类中某个属性的类型或者是某个方法的返回值及参数类型。这个类型参数将在使用时（例如继承或实现这个接口，用这个类型声明变量、创建对象时）确定（即传入实际类型参数）。

**泛型具有以下优点：**

- **编译时的强类型检查**

泛型要求在声明时指定实际数据类型，Java 编译器在编译时会对泛型代码做强类型检查，并在代码违反类型安全时发出告警。早发现，早治理，把隐患扼杀于摇篮，在编译时发现并修复错误所付出的代价远比在运行时小。

- **避免了类型转换**

未使用泛型：

```java
List list = new ArrayList();
list.add("hello");
String s = (String) list.get(0);
```

使用泛型：

```java
List<String> list = new ArrayList<String>();
list.add("hello");
String s = list.get(0);   // no cast
```

- **泛型编程可以实现通用算法**

通过使用泛型，程序员可以实现通用算法，这些算法可以处理不同类型的集合，可以自定义，并且类型安全且易于阅读。

> [!IMPORTANT]
>
> （1）**泛型的类型必须是类，不能是基本数据类型**。需要用到基本数据类型的位置，拿包装类替换
>
> （2）如果实例化时，没有指明泛型的类型。默认类型为java.lang.Object类型

### 泛型类型：泛型类、泛型接口、泛型方法

#### 泛型类

**泛型类的语法形式：**

```java
class name<T1, T2, ..., Tn> { /* ... */ }
```

​	泛型类的声明和非泛型类的声明类似，除了在类名后面添加了类型参数声明部分。由尖括号（`<>`）分隔的类型参数部分跟在类名后面。它指定类型参数（也称为类型变量）T1，T2，...和 Tn。

​	一般将泛型中的类名称为**原型**，而将 `<>` 指定的参数称为**类型参数**

**<u>1）未应用泛型的类</u>**

在泛型出现之前，如果一个类想持有一个可以为任意类型的数据，只能使用 `Object` 做类型转换。示例如下：

```java
public class Info {
	private Object value;
	public Object getValue() {
		return value;
	}
	public void setValue(Object value) {
		this.value = value;
	}
}
```

**<u>2）单类型参数的泛型类</u>**

- 在初始化一个泛型类时，使用 `<>` 指定了内部具体类型，在编译时就会根据这个类型做强类型检查。

```java
public class Info<T> {
    private T value;
    public Info() { }
    public Info(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }
    public void setValue(T value) {
        this.value = value;
    }
    @Override
    public String toString() {
        return "Info{" + "value=" + value + '}';
    }
}

public class GenericsClassDemo01 {
    public static void main(String[] args) {
        Info<Integer> info = new Info<>();
        info.setValue(10);
        System.out.println(info.getValue());

        Info<String> info2 = new Info<>();
        info2.setValue("xyz");
        System.out.println(info2.getValue());
    }
}
// Output:
// 10
// xyz
```

<u>**3）多个类型参数的泛型类**</u>

```java
public class MyMap<K,V> {
    private K key;
    private V value;

    public MyMap(K key, V value) {
        this.key = key;
        this.value = value;
    }
    @Override
    public String toString() {
        return "MyMap{" + "key=" + key + ", value=" + value + '}';
    }
}

public class GenericsClassDemo02 {
    public static void main(String[] args) {
        MyMap<Integer, String> map = new MyMap<>(1, "one");
        System.out.println(map);
    }
}
// Output:
// MyMap{key=1, value=one}
```

#### 泛型接口

**泛型接口语法形式：**

```java
public interface Content<T> {
    T text();
}
```

泛型接口有两种实现方式：

**<u>1）实现接口的子类明确声明泛型类型</u>**

```java
public class GenericsInterfaceDemo01 implements Content<Integer> {
    private int text;

    public GenericsInterfaceDemo01(int text) {
        this.text = text;
    }

    @Override
    public Integer text() { return text; }

    public static void main(String[] args) {
        GenericsInterfaceDemo01 demo = new GenericsInterfaceDemo01(10);
        System.out.print(demo.text());
    }
}
// Output:
// 10
```

**<u>2）实现接口的子类不明确声明泛型类型</u>**

```java
public class GenericsInterfaceDemo02<T> implements Content<T> {
    private T text;

    public GenericsInterfaceDemo02(T text) {
        this.text = text;
    }

    @Override
    public T text() { return text; }

    public static void main(String[] args) {
        GenericsInterfaceDemo02<String> gen = new GenericsInterfaceDemo02<>("ABC");
        System.out.print(gen.text());
    }
}
// Output:
// ABC
```

#### 泛型方法

泛型方法是引入其自己的类型参数的方法。泛型方法可以是普通方法、静态方法以及构造方法。

泛型方法语法形式如下：

```java
public <T> T func(T obj) {}
```

- **`<T>`** ：声明一个类型参数 `T`，表示这是一个泛型方法。==不管返回值是void还是T，必须加上<T>==
- **`T obj`** ：参数类型是 `T`，即方法可以接收**任意类型的对象** 。
- **`T`** ：表示方法返回类型为T

虽然返回值是 `void`，**但参数类型 `T` 依赖于 `<T>` 的声明。如果没有 `<T>`，参数类型只能是 `Object`**，无法实现**编译时类型检查** 和**类型推断** 。

> [!IMPORTANT]
>
> **是否拥有泛型方法，与其所在的类是否是泛型没有关系。**

​	泛型方法的语法包括一个类型参数列表，**在尖括号内，它出现在方法的返回类型之前**。对于静态泛型方法，类型参数部分必须出现在方法的返回类型之前。**类型参数能被用来声明返回值类型，并且能作为泛型方法得到的实际类型参数的占位符**。

- **使用泛型方法的时候，通常不必指明类型参数，因为编译器会为我们找出具体的类型。这称为类型参数推断（type argument inference）。类型推断只对赋值操作有效，其他时候并不起作用**。如果将一个返回类型为 T 的泛型方法调用的结果作为参数，传递给另一个方法，这时编译器并不会执行推断。编译器会认为：调用泛型方法后，其返回值被赋给一个 Object 类型的变量。

```java
public class GenericsMethodDemo01 {
    public static <T> void printClass(T obj) {
        System.out.println(obj.getClass().toString());
    }

    public static void main(String[] args) {
        printClass("abc");
        printClass(10);
    }
}
// Output:
// class java.lang.String
// class java.lang.Integer
```

- 泛型方法中也可以使用可变参数列表

```java
public class GenericVarargsMethodDemo {
    public static <T> List<T> makeList(T... args) {
        List<T> result = new ArrayList<T>();
        Collections.addAll(result, args);
        return result;
    }
    public static void main(String[] args) {
        List<String> ls = makeList("A");
        System.out.println(ls);
        ls = makeList("A", "B", "C");
        System.out.println(ls);
    }
}
// Output:
// [A]
// [A, B, C]
```

### 类型擦除

​	Java 语言引入泛型是为了在编译时提供更严格的类型检查，并支持泛型编程。不同于 C++ 的模板机制，**Java 泛型是使用类型擦除来实现的，使用泛型时，任何具体的类型信息都被擦除了**。

那么，类型擦除做了什么呢？它做了以下工作：

- 把泛型中的所有类型参数替换为 Object，如果指定类型边界，则使用类型边界来替换。因此，生成的字节码仅包含普通的类，接口和方法。
- **擦除出现的类型声明，即去掉 `<>` 的内容**。比如 `T get()` 方法声明就变成了 `Object get()` ；`List<String>` 就变成了 `List`。如有必要，插入类型转换以保持类型安全。
- 生成桥接方法以保留扩展泛型类型中的多态性。类型擦除确保不为参数化类型创建新类；因此，泛型不会产生运行时开销。

让我们来看一个示例：

```java
public class GenericsErasureTypeDemo {
    public static void main(String[] args) {
        List<Object> list1 = new ArrayList<Object>();
        List<String> list2 = new ArrayList<String>();
        System.out.println(list1.getClass());
        System.out.println(list2.getClass());
    }
}
// Output:
// class java.util.ArrayList
// class java.util.ArrayList
```

> 示例说明：
>
> 上面的例子中，虽然指定了不同的类型参数，但是 list1 和 list2 的类信息却是一样的。
>
> 这是因为：**使用泛型时，任何具体的类型信息都被擦除了**。这意味着：`ArrayList<Object>` 和 `ArrayList<String>` 在运行时，JVM 将它们视为同一类型。

###  泛型和继承

​	**泛型不能用于显式地引用运行时类型的操作之中，例如：<font color = '#8D0101'>转型、instanceof 操作和 new 表达式</font>。因为所有关于参数的类型信息都丢失了**。当你在编写泛型代码时，必须时刻提醒自己，你只是看起来好像拥有有关参数的类型信息而已。

正是由于泛型时基于类型擦除实现的，所以，**泛型类型无法向上转型**。

> 向上转型是指用**子类实例去初始化父类**，这是面向对象中多态的重要表现

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/1553147778883.png" alt="img" style="zoom:67%;" />

- `Integer` 继承了 `Object`；`ArrayList` 继承了 `List`；但是 `List<Interger>` 却并非继承了 `List<Object>`，二者是并列关系
  - 这是因为，泛型类并没有自己独有的 `Class` 类对象。比如：并不存在 `List<Object>.class` 或是 `List<Interger>.class`，Java 编译器会将二者都视为 `List.class`。

### 泛型的约束

- **泛型类型的类型参数<font color = '#8D0101'>不能是值类型</font>c(opens new window)**

```java
Pair<int, char> p = new Pair<>(8, 'a');  // 编译错误
```

- **不能创建类型参数的实例(opens new window)**

```java
public static <E> void append(List<E> list) {
    E elem = new E();  // 编译错误
    list.add(elem);
}
```

- **不能声明类型为类型参数的静态成员(opens new window)**

```java
public class MobileDevice<T> {
    private static T os; // error
    // ...
}
```

- **类型参数不能使用类型转换或 instanceof(opens new window)**

```java
public static <E> void rtti(List<E> list) {
    if (list instanceof ArrayList<Integer>) {  // 编译错误
        // ...
    }
}
List<Integer> li = new ArrayList<>();
List<Number>  ln = (List<Number>) li;  // 编译错误
```

- **不能创建类型参数的数组(opens new window)**

```java
List<Integer>[] arrayOfLists = new List<Integer>[2];  // 编译错误
```

- **不能创建、catch 或 throw 参数化类型对象(opens new window)**

```java
// Extends Throwable indirectly
class MathException<T> extends Exception { /* ... */ }    // 编译错误

// Extends Throwable directly
class QueueFullException<T> extends Throwable { /* ... */ // 编译错误
```

## 反射

### 反射概述

​	Reflection（反射）是被视为动态语言的关键，反射机制允许程序在执行期间借助于Reflection API取得任何类的内部信息，并能直接操作任意对象的内部属性及方法

​	加载完类之后，**在堆内存的方法区中就产生了一个Class类型的对象（一个类只有一个Class对象），这个对象就包含了完整的类的结构信息，我们可以通过这个对象看到类的结构**。这个对象就像一面镜子，透过这个镜子看到类的结构，所以称为反射。**通过反射机制，可以在运行时访问 Java 对象的属性，方法，构造方法等。**

![image-20250307164020906](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307164020906.png)

### 反射的应用场景

反射的主要应用场景有：

- **开发通用框架** - 反射最重要的用途就是开发各种通用框架。很多框架（比如 Spring）都是配置化的（比如通过 XML 文件配置 JavaBean、Filter 等），为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射——运行时动态加载需要加载的对象。
- **动态代理** - 在切面编程（AOP）中，需要拦截特定的方法，通常，会选择动态代理方式。这时，就需要反射技术来实现了。
- **注解** - 注解本身仅仅是起到标记作用，它需要利用反射机制，根据注解标记去调用注解解释器，执行行为。如果没有反射机制，注解并不比注释更有用。
- **可扩展性功能** - 应用程序可以通过使用完全限定名称创建可扩展性对象实例来使用外部的用户定义类

### java.lang.Class/Class 对象

**<u>（1）类的加载过程</u>**

- 程序经过javac.exe命令以后会生成一个或多个字节码文件（.class结尾），接着我们使用java.exe命令对某个字节码文件进行解释运行，相当于将某个字节码文件加载到内存中，此过程就称为类的加载，**加载到内存中的类，我们就称为运行时类，Class的实例就对应着一个运行时类**
- **Java 中，无论生成某个类的多少个对象，这些对象都会对应于同一个 Class 对象。这个 Class 对象是由 JVM 生成的，通过它能够获悉整个类的结构**。所以，==`java.lang.Class` 可以视为所有反射 API 的入口点==。

举例来说，假如定义了以下代码：

```java
User user = new User();
```

步骤说明：

1. JVM 加载方法的时候，遇到 `new User()`，JVM 会根据 `User` 的全限定名去加载 `User.class` 。
2. JVM 会去本地磁盘查找 `User.class` 文件并加载 JVM 内存中。
3. JVM 通过调用类加载器自动创建这个**类对应的 `Class` 对象，并且存储在 JVM 的方法区**。注意：**一个类有且只有一个 `Class` 对象**

### 获取 Class 对象的四种方式

![image-20250307170407508](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307170407508.png)

​	如果我们动态获取到这些信息，我们需要依靠 Class 对象。Class 类对象将一个类的方法、变量等信息告诉运行的程序。Java 提供了四种方式获取 Class 对象:

==**1. 知道具体类的情况下可以使用：**==

```
Class alunbarClass = TargetObject.class;
```

但是我们一般是不知道具体类的，基本都是通过遍历包下面的类来获取 Class 对象，通过此方式获取 Class 对象不会进行初始化

==**2. 通过 `Class.forName()`传入类的全路径获取：**==

```
Class alunbarClass1 = Class.forName("cn.javaguide.TargetObject");
```

==**3. 通过对象实例`instance.getClass()`获取：**==

```
TargetObject o = new TargetObject();
Class alunbarClass2 = o.getClass();
```

**4. 通过类加载器`xxxClassLoader.loadClass()`传入类路径获取:**

```
ClassLoader.getSystemClassLoader().loadClass("cn.javaguide.TargetObject");
```

通过类加载器获取 Class 对象不会进行初始化，意味着不进行包括初始化等一系列步骤，静态代码块和静态对象不会得到执行

### 反射的一些基本操作

1. 创建一个我们要使用反射操作的类 `TargetObject`。

```java
package cn.javaguide;

public class TargetObject {
    private String value;

    public TargetObject() {
        value = "JavaGuide";
    }
    public void publicMethod(String s) {
        System.out.println("I love " + s);
    }
    private void privateMethod() {
        System.out.println("value is " + value);
    }
}
```

2.使用反射操作这个类的方法以及属性

```java
package cn.javaguide;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException, NoSuchFieldException {
        /**
         * 获取 TargetObject 类的 Class 对象并且创建 TargetObject 类实例（通过全类名）
         */
        Class<?> targetClass = Class.forName("cn.javaguide.TargetObject");
        TargetObject targetObject = (TargetObject) targetClass.newInstance();
        /**
         * 获取 TargetObject 类中定义的所有方法
         */
        Method[] methods = targetClass.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println(method.getName());
        }
        /**
         * 获取指定方法并调用
         */
        Method publicMethod = targetClass.getDeclaredMethod("publicMethod",
                String.class);
        publicMethod.invoke(targetObject, "JavaGuide");

        /**
         * 获取指定参数并对参数进行修改
         */
        Field field = targetClass.getDeclaredField("value");
        //为了对类中的参数进行修改我们取消安全检查
        field.setAccessible(true);
        field.set(targetObject, "JavaGuide");

        /**
         * 调用 private 方法
         */
        Method privateMethod = targetClass.getDeclaredMethod("privateMethod");
        //为了调用private方法我们取消安全检查
        privateMethod.setAccessible(true);
        privateMethod.invoke(targetObject);
    }
}
```

其中：

- 获取 TargetObject 类的 **==Class 对象==**，然后再通过TargetObject 类的Class对象创建TargetObject 类的实例

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218170930754.png" alt="image-20250218170930754" style="zoom:50%;" />

- **==获取指定方法==**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218171455282.png" alt="image-20250218171455282" style="zoom:50%;" />

### JDK 动态代理

为了解决静态代理的问题，就有了创建动态代理的想法：

​	在运行状态中，需要代理的地方，根据 Subject 和 RealSubject，动态地创建一个 Proxy，用完之后，就会销毁，这样就可以避免了 Proxy 角色的 class 在系统中冗杂的问题了。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/1553614585028.png)

**动态代理步骤：**

1. 获取 RealSubject 上的所有接口列表；
2. 确定要生成的代理类的类名，默认为：`com.sun.proxy.$ProxyXXXX`；
3. 根据需要实现的接口信息，在代码中动态创建该 Proxy 类的字节码；
4. 将对应的字节码转换为对应的 class 对象；
5. 创建 `InvocationHandler` 实例 handler，用来处理 `Proxy` 所有方法调用；
6. Proxy 的 class 对象 以创建的 handler 对象为参数，实例化一个 proxy 对象。

从上面可以看出，JDK 动态代理的实现是基于实现接口的方式，使得 Proxy 和 RealSubject 具有相同的功能。

​	但其实还有一种思路：通过继承。即：让 Proxy 继承 RealSubject，这样二者同样具有相同的功能，Proxy 还可以通过重写 RealSubject 中的方法，来实现多态。CGLIB 就是基于这种思路设计的。

​	在 Java 的动态代理机制中，有两个重要的类（接口），一个是 `InvocationHandler` 接口、另一个则是 `Proxy` 类，这一个类和一个接口是实现我们动态代理所必须用到的



# IO

## 传统 java IO流

### IO流分类

流从概念上来说是一个连续的数据流。当程序需要读数据的时候就需要使用输入流读取数据，当需要往外写数据的时候就需要输出流。

BIO 中操作的流主要有两大类，**字节流（8bit）和字符流（16bit）**，两类根据流的方向都可以分为输入流和输出流。

- **字节流**
  - 输入字节流：`InputStream`
  - 输出字节流：`OutputStream`
- **字符流**
  - 输入字符流：`Reader`
  - 输出字符流：`Writer`

![image-20250307171641610](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307171641610.png)

![image-20250307171651528](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250307171651528.png)

### 字节流

**字节流主要操作字节数据或二进制对象。**

字节流有两个核心抽象类：`InputStream` 和 `OutputStream`。所有的字节流类都继承自这两个抽象类。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200219133627.png" alt="img" style="zoom:67%;" />

文件流操作一般步骤：

1. 使用 `File` 类绑定一个文件。
2. 把 `File` 对象绑定到流对象上。
3. 进行读或写操作。
4. 关闭流

`FileOutputStream` 和 `FileInputStream` 示例：

```java
public class FileStreamDemo {

    private static final String FILEPATH = "temp.log";
    public static void main(String[] args) throws Exception {
        write(FILEPATH);
        read(FILEPATH);
    }

    public static void write(String filepath) throws IOException {
        // 第1步、使用File类找到一个文件
        File f = new File(filepath);
        // 第2步、通过子类实例化父类对象
        OutputStream out = new FileOutputStream(f);
        // 实例化时，默认为覆盖原文件内容方式；如果添加true参数，则变为对原文件追加内容的方式。
        // OutputStream out = new FileOutputStream(f, true);
        // 第3步、进行写操作
        String str = "Hello World\n";
        byte[] bytes = str.getBytes();
        out.write(bytes);
        // 第4步、关闭输出流
        out.close();
    }

    public static void read(String filepath) throws IOException {
        // 第1步、使用File类找到一个文件
        File f = new File(filepath);
        // 第2步、通过子类实例化父类对象
        InputStream input = new FileInputStream(f);
        // 第3步、进行读操作
        // 有三种读取方式，体会其差异
        byte[] bytes = new byte[(int) f.length()];
        int len = input.read(bytes); // 读取内容
        System.out.println("读入数据的长度：" + len);
        // 第4步、关闭输入流
        input.close();
        System.out.println("内容为：\n" + new String(bytes));
    }

}
```

不过，一般我们是不会直接单独使用 `FileInputStream` ，通常会配合 `BufferedInputStream`（字节缓冲输入流，后文会讲到）来使用。

像下面这段代码在我们的项目中就比较常见，我们通过 `readAllBytes()` 读取输入流所有字节并将其直接赋值给一个 `String` 对象。

```java
// 新建一个 BufferedInputStream 对象
BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream("input.txt"));
// 读取文件的内容并复制到 String 对象中
String result = new String(bufferedInputStream.readAllBytes());
System.out.println(result);
```

类似于 `FileInputStream`，`FileOutputStream` 通常也会配合 `BufferedOutputStream`（字节缓冲输出流，后文会讲到）来使用。

```java
FileOutputStream fileOutputStream = new FileOutputStream("output.txt");
BufferedOutputStream bos = new BufferedOutputStream(fileOutputStream)
```

> [!IMPORTANT]
>
> - `BufferedInputStream` 从源头（通常是文件）读取数据（字节信息）到内存的过程中不会一个字节一个字节的读取，而是会先将读取到的字节存放在缓存区，并从内部缓冲区中单独读取字节。这样大幅减少了 IO 次数，提高了读取效率。
> - `BufferedOutputStream` 将数据（字节信息）写入到目的地（通常是文件）的过程中不会一个字节一个字节的写入，而是会先将要写入的字节存放在缓存区，并从内部缓冲区中单独写入字节。这样大幅减少了 IO 次数，提高了读取效率
> - 类似于 `BufferedInputStream` ，`BufferedOutputStream` 内部也维护了一个缓冲区，并且，这个缓存区的大小也是 **8192** 字节。
> - `BufferedReader` （字符缓冲输入流）和 `BufferedWriter`（字符缓冲输出流）类似于 `BufferedInputStream`（字节缓冲输入流）和`BufferedOutputStream`（字节缓冲输入流），内部都维护了一个字节数组作为缓冲区。不过，前者主要是用来操作字符信息。

### 字符流

字符流主要操作字符，一个字符等于两个字节。

字符流有两个核心类：`Reader` 类和 `Writer` 。所有的字符流类都继承自这两个抽象类。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200219133648.png" alt="img" style="zoom:67%;" />

文件字符流 `FileReader` 和 `FileWriter` 可以向文件读写文本数据。

`FileReader` 和 `FileWriter` 读写文件示例：

```java
public class FileReadWriteDemo {

    private static final String FILEPATH = "temp.log";

    public static void main(String[] args) throws IOException {
        write(FILEPATH);
        System.out.println("内容为：" + new String(read(FILEPATH)));
    }

    public static void write(String filepath) throws IOException {
        // 1.使用 File 类绑定一个文件
        File f = new File(filepath);
        // 2.把 File 对象绑定到流对象上
        Writer out = new FileWriter(f);
        // Writer out = new FileWriter(f, true); // 追加内容方式
        // 3.进行读或写操作
        String str = "Hello World!!!\r\n";
        out.write(str);
        // 4.关闭流
        // 字符流操作时使用了缓冲区，并在关闭字符流时会强制将缓冲区内容输出
        // 如果不关闭流，则缓冲区的内容是无法输出的
        // 如果想在不关闭流时，将缓冲区内容输出，可以使用 flush 强制清空缓冲区
        out.flush();
        out.close();
    }

    public static char[] read(String filepath) throws IOException {
        // 1.使用 File 类绑定一个文件
        File f = new File(filepath);
        // 2.把 File 对象绑定到流对象上
        Reader input = new FileReader(f);
        // 3.进行读或写操作
        int temp = 0; // 接收每一个内容
        int len = 0; // 读取内容
        char[] c = new char[1024];
        while ((temp = input.read()) != -1) {
            // 如果不是-1就表示还有内容，可以继续读取
            c[len] = (char) temp;
            len++;
        }
        System.out.println("文件字符数为：" + len);
        // 4.关闭流
        input.close();
        return c;
    }

}
```

### IO工具类

#### File

​	`File` 类是 `java.io` 包中唯一对文件本身进行操作的类。它可以对文件、目录进行增删查操作。

**<u>（1）createNewFille</u>**

**可以使用 `createNewFille()` 方法创建一个新文件**。

注：

> [!NOTE]
>
> Windows 中使用反斜杠表示目录的分隔符 `\`。
>
> Linux 中使用正斜杠表示目录的分隔符 `/`。
>
> 最好的做法是使用 `File.separator` 静态常量，可以根据所在操作系统选取对应的分隔符。

【示例】创建文件

```java
File f = new File(filename);
boolean flag = f.createNewFile();
```

**<u>（2）mkdir</u>**

​	**可以使用 `mkdir()` 来创建文件夹**，但是如果要创建的目录的父路径不存在，则无法创建成功。

​	如果要解决这个问题，可以使用 `mkdirs()`，当父路径不存在时，会连同上级目录都一并创建。

【示例】创建目录

```java
File f = new File(filename);
boolean flag = f.mkdir();
```

**<u>（3）delete</u>**

​	**可以使用 `delete()` 来删除文件或目录**。

​	需要注意的是，如果删除的是目录，且目录不为空，直接用 `delete()` 删除会失败。

【示例】删除文件或目录

```java
File f = new File(filename);
boolean flag = f.delete();
```

**<u>（4）list 和 listFiles</u>**

`File` 中给出了两种列出文件夹内容的方法：

- **`list()`: 列出全部名称，返回一个字符串数组**。
- **`listFiles()`: 列出完整的路径，返回一个 `File` 对象数组**。

`list()` 示例：

```java
File f = new File(filename);
String str[] = f.list();
```

`listFiles()` 示例：

```java
File f = new File(filename);
File files[] = f.listFiles();
```

#### Scanner

**`Scanner` 可以获取用户的输入，并对数据进行校验**。

【示例】校验输入数据是否格式正确

```java
import java.io.*;
public class ScannerDemo {

    public static void main(String args[]) {
        Scanner scan = new Scanner(System.in);    // 从键盘接收数据
        int i = 0;
        float f = 0.0f;
        System.out.print("输入整数：");
        if (scan.hasNextInt()) {    // 判断输入的是否是整数
            i = scan.nextInt();    // 接收整数
            System.out.println("整数数据：" + i);
        } else {
            System.out.println("输入的不是整数！");
        }

        System.out.print("输入小数：");
        if (scan.hasNextFloat()) {    // 判断输入的是否是小数
            f = scan.nextFloat();    // 接收小数
            System.out.println("小数数据：" + f);
        } else {
            System.out.println("输入的不是小数！");
        }

        Date date = null;
        String str = null;
        System.out.print("输入日期（yyyy-MM-dd）：");
        if (scan.hasNext("^\\d{4}-\\d{2}-\\d{2}$")) {    // 判断
            str = scan.next("^\\d{4}-\\d{2}-\\d{2}$");    // 接收
            try {
                date = new SimpleDateFormat("yyyy-MM-dd").parse(str);
            } catch (Exception e) {}
        } else {
            System.out.println("输入的日期格式错误！");
        }
        System.out.println(date);
    }
}
```

输出：

```text
输入整数：20
整数数据：20
输入小数：3.2
小数数据：3.2
输入日期（yyyy-MM-dd）：1988-13-1
输入的日期格式错误！
null
```

## java序列化（JDK序列化、Json序列化、二进制序列化）

### 什么是序列化和反序列化?

如果我们需要持久化 Java 对象比如将 Java 对象保存在文件中，或者在网络传输 Java 对象，这些场景都需要用到序列化。

- **序列化**：将数据结构或对象转换成可以存储或传输的形式，通常是**二进制字节流**，也可以是 **JSON, XML 等文本格式**；
- **反序列化**：将在序列化过程中所生成的数据转换为原始数据结构或者对象的过程

下面是序列化和反序列化常见应用场景：

- 对象在进行**网络传输**（比如远程方法调用 RPC 的时候）之前需要先被序列化，接收到序列化的对象之后需要再进行反序列化；
- 将**对象存储到文件**之前需要进行序列化，将对象从文件中读取出来需要进行反序列化；
- 将对象**存储到数据库（如 Redis）**之前需要用到序列化，将对象从缓存数据库中读取出来需要反序列化；
- 将对象**存储到内存**之前需要进行序列化，从内存中读取出来之后需要进行反序列化。

**序列化的主要目的是通过网络传输对象或者说是将对象存储到文件系统、数据库、内存中。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/a478c74d-2c48-40ae-9374-87aacf05188c.png" alt="img" style="zoom:80%;" />

### JDK 序列化

Java 通过对象输入输出流来实现序列化和反序列化：

- **`java.io.ObjectOutputStream` 类的 `writeObject()` 方法可以实现序列化；**
- **`java.io.ObjectInputStream` 类的 `readObject()` 方法用于实现反序列化。**

JDK 自带的序列化，**被序列化的类必须属于 `Enum`、`Array` 和 `Serializable` 类型其中的任何一种，否则将抛出 `NotSerializableException` 异常**

```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Builder
@ToString
public class RpcRequest implements Serializable {
    private static final long serialVersionUID = 1905122041950251207L;
    private String requestId;
    private String interfaceName;
    private String methodName;
    private Object[] parameters;
    private Class<?>[] paramTypes;
    private RpcMessageTypeEnum rpcMessageTypeEnum;
}
```

**<u>serialVersionUID 有什么作用？</u>**

- 序列化号 `serialVersionUID` 属于**版本控制**的作用。反序列化时，会检查 `serialVersionUID` 是否和当前类的 `serialVersionUID` 一致。如果 `serialVersionUID` 不一致则会抛出 `InvalidClassException` 异常。强烈推荐每个序列化类都手动指定其 `serialVersionUID`，如果不手动指定，那么编译器会动态生成默认的 `serialVersionUID`。
- <font color = '#8D0101'>**`serialVersionUID` 字段必须是 `static final long` 类型**。</font>

<u>**serialVersionUID 不是被 static 变量修饰了吗？为什么还会被“序列化”？**</u>

- `static` 修饰的变量是静态变量，属于类而非类的实例，本身是不会被序列化的。
- 然而，`serialVersionUID` 是一个特例，`serialVersionUID` 的序列化做了特殊处理。当一个对象被序列化时，`serialVersionUID` 会被写入到序列化的二进制流中；
- 在反序列化时，也会解析它并做一致性判断，以此来验证序列化对象的版本一致性。如果两者不匹配，反序列化过程将抛出 `InvalidClassException`，因为这通常意味着序列化的类的定义已经发生了更改，可能不再兼容。
- 也就是说，**`serialVersionUID` 只是用来被 JVM 识别，实际并没有被序列化**

<u>**如果有些字段不想进行序列化怎么办（transient）**</u>

对于不想进行序列化的变量，可以使用 `transient` 关键字修饰。

- `transient` 关键字的作用是：阻止实例中那些用此关键字修饰的的变量序列化；当对象被反序列化时，被 `transient` 修饰的变量值不会被持久化和恢复。
- `transient` 只能修饰变量，不能修饰类和方法。
- `transient` 修饰的变量，在反序列化后变量值将会被置成类型的默认值。例如，如果是修饰 `int` 类型，那么反序列后结果就是 `0`。
- `static` 变量因为不属于任何对象(Object)，所以无论有没有 `transient` 关键字修饰，均不会被序列化。

**<u>序列化和反序列化示例：</u>**

```java
public class SerializeDemo01 {
    enum Sex {
        MALE,
        FEMALE
    }

    static class Person implements Serializable {
        private static final long serialVersionUID = 1L;
        private String name = null;
        private Integer age = null;
        private Sex sex;

        public Person() { }
        public Person(String name, Integer age, Sex sex) {
            this.name = name;
            this.age = age;
            this.sex = sex;
        }
        @Override
        public String toString() {
            return "Person{" + "name='" + name + '\'' + ", age=" + age + ", sex=" + sex + '}';
        }
    }

    /**
     * 序列化
     */
    private static void serialize(String filename) throws IOException {
        File f = new File(filename); // 定义保存路径
        OutputStream out = new FileOutputStream(f); // 文件输出流
        ObjectOutputStream oos = new ObjectOutputStream(out); // 对象输出流
        oos.writeObject(new Person("Jack", 30, Sex.MALE)); // 保存对象
        oos.close();
        out.close();
    }

    /**
     * 反序列化
     */
    private static void deserialize(String filename) throws IOException, ClassNotFoundException {
        File f = new File(filename); // 定义保存路径
        InputStream in = new FileInputStream(f); // 文件输入流
        ObjectInputStream ois = new ObjectInputStream(in); // 对象输入流
        Object obj = ois.readObject(); // 读取对象
        ois.close();
        in.close();
        System.out.println(obj);
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        final String filename = "d:/text.dat";
        serialize(filename);
        deserialize(filename);
    }
}
// Output:
// Person{name='Jack', age=30, sex=MALE}
```

### json序列化

#### JSON 标准

这估计是最简单标准规范之一：

- 只有两种结构：对象内的键值对集合结构和数组，对象用 `{}` 表示、内部是 `"key":"value"`，数组用 `[]` 表示，不同值用逗号分开
- 基本数值有 7 个： `false` / `null` / `true` / `object` / `array` / `number` / `string`
- 再加上结构可以嵌套，进而可以用来表达复杂的数据
- 一个简单实例：

```json
{
  "Image": {
    "Width": 800,
    "Height": 600,
    "Title": "View from 15th Floor",
    "Thumbnail": {
      "Url": "http://www.example.com/image/481989943",
      "Height": 125,
      "Width": "100"
    },
    "IDs": [116, 943, 234, 38793]
  }
}
```

#### JSON 优缺点

优点：

- 基于纯文本，所以对于人类阅读是很友好的。
- 规范简单，所以容易处理，开箱即用，特别是 JS 类的 ECMA 脚本里是内建支持的，可以直接作为对象使用。
- 平台无关性，因为类型和结构都是平台无关的，而且好处理，容易实现不同语言的处理类库，可以作为多个不同异构系统之间的数据传输格式协议，特别是在 HTTP/REST 下的数据格式。

缺点：

- 性能一般，文本表示的数据一般来说比二进制大得多，在数据传输上和解析处理上都要更影响性能。

#### Java JSON 库

Java 中比较流行的 JSON 库有：

- [Fastjson (opens new window)](https://github.com/alibaba/fastjson)- 阿里巴巴开发的 JSON 库，性能十分优秀。
- [Jackson (opens new window)](http://wiki.fasterxml.com/JacksonHome)- 社区十分活跃且更新速度很快。Spring 框架默认 JSON 库。
- [Gson (opens new window)](https://github.com/google/gson)- 谷歌开发的 JSON 库，目前功能最全的 JSON 库 。

从性能上来看，一般情况下：Fastjson > Jackson > Gson

#### Jackson 应用

> 扩展阅读：更多 API 使用细节可以参考 [jackson-databind 官方说明(opens new window)](https://github.com/FasterXML/jackson-databind)

**<u>（1）添加 maven 依赖</u>**

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.8</version>
</dependency>
```

**<u>（2）序列化</u>**

```java
ObjectMapper mapper = new ObjectMapper();

mapper.writeValue(new File("result.json"), myResultObject);
// or:
byte[] jsonBytes = mapper.writeValueAsBytes(myResultObject);
// or:
String jsonString = mapper.writeValueAsString(myResultObject);
```

**<u>（3）反序列化</u>**

```java
ObjectMapper mapper = new ObjectMapper();

MyValue value = mapper.readValue(new File("data.json"), MyValue.class);
// or:
value = mapper.readValue(new URL("http://some.com/api/entry.json"), MyValue.class);
// or:
value = mapper.readValue("{\"name\":\"Bob\", \"age\":13}", MyValue.class);
```

**<u>（4）容器的序列化和反序列化</u>**

- json对象是一个map，List对应json中的数组

```java
Person p = new Person("Tom", 20);
Person p2 = new Person("Jack", 22);
Person p3 = new Person("Mary", 18);

List<Person> persons = new LinkedList<>();
persons.add(p);
persons.add(p2);
persons.add(p3);

Map<String, List> map = new HashMap<>();
map.put("persons", persons);

String json = null;
try {
 json = mapper.writeValueAsString(map);
} catch (JsonProcessingException e) {
 e.printStackTrace();
}
```

#### Gson应用

**<u>（1）添加 maven 依赖</u>**

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.6</version>
</dependency>
```

**<u>（2）序列化</u>**

```java
Gson gson = new Gson();
gson.toJson(1);            // ==> 1
gson.toJson("abcd");       // ==> "abcd"
gson.toJson(10L); // ==> 10
int[] values = { 1 };
gson.toJson(values);       // ==> [1]
```

**<u>（3）反序列化</u>**

```java
int i1 = gson.fromJson("1", int.class);
Integer i2 = gson.fromJson("1", Integer.class);
Long l1 = gson.fromJson("1", Long.class);
Boolean b1 = gson.fromJson("false", Boolean.class);
String str = gson.fromJson("\"abc\"", String.class);
String[] anotherStr = gson.fromJson("[\"abc\"]", String[].class);
```

**<u>（4）GsonBuilder</u>**

`Gson` 实例可以通过 `GsonBuilder` 来定制实例化，以控制其序列化、反序列化行为。

```java
Gson gson = new GsonBuilder()
  .setPrettyPrinting()
  .setDateFormat("yyyy-MM-dd HH:mm:ss")
  .excludeFieldsWithModifiers(Modifier.STATIC, Modifier.TRANSIENT, Modifier.VOLATILE)
  .create();
```

### 二进制序列化

**<u>（1）为什么需要二进制序列化库</u>**

​	原因很简单，就是 Java 默认的序列化机制（`ObjectInputStream` 和 `ObjectOutputStream`）具有很多缺点。

> 不了解 Java 默认的序列化机制，可以参考：[Java 序列化(opens new window)](https://dunwu.github.io/waterdrop/pages/2b2f0f/)

Java 自身的序列化方式具有以下缺点：

- **无法跨语言使用**。这点最为致命，对于很多需要跨语言通信的异构系统来说，不能跨语言序列化，即意味着完全无法通信（彼此数据不能识别，当然无法交互了）。
- **序列化的性能不高**。序列化后的数据体积较大，这大大影响存储和传输的效率。
- 序列化一定需要实现 `Serializable` 接口。
- 需要关注 `serialVersionUID`。

引入二进制序列化库就是为了解决这些问题，这在 RPC 应用中尤为常见。

**<u>（2）Protobuf</u>**

​	**rpc协议的传输过程就可以用Protobuf**

​	Protobuf 出自于 Google，性能还比较优秀，也支持多种语言，同时还是跨平台的。就是在使用中过于繁琐，因为你需要自己定义 IDL 文件和生成对应的序列化代码。这样虽然不灵活，但是，另一方面导致 protobuf 没有序列化漏洞的风险。

> Protobuf 包含序列化格式的定义、各种语言的库以及一个 IDL 编译器。正常情况下你需要定义 proto 文件，然后使用 IDL 编译器编译成你需要的语言

一个简单的 proto 文件如下：

```protobuf
// protobuf的版本
syntax = "proto3";
// SearchRequest会被编译成不同的编程语言的相应对象，比如Java中的class、Go中的struct
message Person {
  //string类型字段
  string name = 1;
  // int 类型字段
  int32 age = 2;
}
```

**<u>（3）Kryo</u>**

​	Kryo 是一个高性能的序列化/反序列化工具，由于其变长存储特性并使用了字节码生成机制，拥有较高的运行速度和较小的字节码体积。

​	另外，Kryo 已经是一种非常成熟的序列化实现了，已经在 Twitter、Groupon、Yahoo 以及多个著名开源项目（如 Hive、Storm）中广泛的使用。

```java
/**
 * Kryo serialization class, Kryo serialization efficiency is very high, but only compatible with Java language
 *
 * @author shuang.kou
 * @createTime 2020年05月13日 19:29:00
 */
@Slf4j
public class KryoSerializer implements Serializer {

    /**
     * Because Kryo is not thread safe. So, use ThreadLocal to store Kryo objects
     */
    private final ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(() -> {
        Kryo kryo = new Kryo();
        kryo.register(RpcResponse.class);
        kryo.register(RpcRequest.class);
        return kryo;
    });
    @Override
    public byte[] serialize(Object obj) {
        try (ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
             Output output = new Output(byteArrayOutputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            // Object->byte:将对象序列化为byte数组
            kryo.writeObject(output, obj);
            kryoThreadLocal.remove();
            return output.toBytes();
        } catch (Exception e) {
            throw new SerializeException("Serialization failed");
        }
    }
    @Override
    public <T> T deserialize(byte[] bytes, Class<T> clazz) {
        try (ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
             Input input = new Input(byteArrayInputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            // byte->Object:从byte数组中反序列化出对象
            Object o = kryo.readObject(input, clazz);
            kryoThreadLocal.remove();
            return clazz.cast(o);
        } catch (Exception e) {
            throw new SerializeException("Deserialization failed");
        }
    }
}

```

## Unix IO模型

### 什么是IO

​	为了保证操作系统的稳定性和安全性，一个进程的地址空间划分为**用户空间（User space）和内核空间（Kernel space）**。像我们平常运行的应用程序都是运行在用户空间，**只有内核空间才能进行系统态级别的资源有关的操作**，比如文件管理、进程通信、内存管理等等。**想要进行IO操作，一定是要依赖内核空间的能力**。

​	用户进程想要执行IO操作的话，因为其没有权限，必须**通过系统调用来间接访问内核空间**。即用户进程(程序)对操作系统的内核发起IO调用（系统调用），操作系统负责的内核执行具体的IO操作。也就是说，**我们的应用程序实际上只是发起了IO操作的调用而已，具体IO的执行是由操作系统的内核来完成的**

当应用程序发起I/O调用后，会经历两个步骤：

- **内核等待I/O设备准备好数据**
- **内核将数据从内核空间拷贝到用户空间**。



**<u>UNIX 系统下的 I/O 模型有 5 种：</u>**

- **同步阻塞 I/O（blocking I/O）**
- **同步非阻塞 I/O（nonblocking I/O）**
- **I/O 多路复用（I/O multiplexing）**
- **信号驱动 I/O（signal driven I/O (SIGIO)）**
- **异步 I/O（asynchronous I/O (the POSIX aio_functions)）**

<font color = '#8D0101'>前面四种都是同步IO、第五种是异步IO；</font>

### 同步与异步、阻塞与非阻塞

<u>**1）同步与异步**</u>

- **同步**：两个同步任务相互依赖，并且一个任务必须以依赖于另一任务的某种方式执行。比如在A->B事件模型中，你需要先完成A才能执行B。再换句话说，同步调用中被调用者未处理完请求之前，调用不返回，调用者会一直等待结果的返回。比如要读数据，等结果返回后再继续读
- **异步**：两个异步的任务完全独立的，一方的执行不需要等待另外一方的执行。再换句话说，异步调用中一调用就返回。结果不需要等待结果返回，**当结果返回的时候通过<font color = '#8D0101'>回调函数</font>或者其他方式拿着结果再做相关事情**。比如要读数据，由后台直接完成读操作，然后线程继续做接下来的事

**<u>2）阻塞与非阻塞</u>**

- **阻塞**：阻塞就是发起一个请求，调用者一直等待请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。
- **非阻塞**：非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

**<u>3）同步阻塞IO、同步非阻塞IO、异步非阻塞IO</u>**

​	我们通常会说到同步阻塞IO、同步非阻塞IO，异步IO几种术语，通过上面的内容，那么我想你现在肯定已经理解了什么是阻塞什么是非阻塞了，所谓阻塞就是发起读取数据请求时，当数据还没准备就绪的时候，这时请求是即刻返回，还是在这里等待数据的就绪，如果需要等待的话就是阻塞，反之如果即刻返回就是非阻塞。

- **我们再看<font color = '#8D0101'>同步阻塞</font>、<font color = '#8D0101'>同步非阻塞</font>，他们不同的只是发起读取请求的时候一个请求阻塞，一个请求不阻塞，但是相同的是，<font color = '#8D0101'>他们都需要应用自己监控整个数据完成的过程</font>。**
- 而为什么只有异步非阻塞而没有异步阻塞呢，**因为异步模型下请求指定发送完后就即刻返回了，没有任何后续流程了，所以它注定不会阻塞**，所以也就只会有异步非阻塞模型了。

不能一概而论认为同步或阻塞就是低效，具体还要看应用和系统特征。

<u>**4）如何区分“同步/异步”和“阻塞/非阻塞”**</u>

- 同步/异步是从行为角度描述事物的
- 阻塞和非阻塞描述的当前事物的状态（等待调用结果时的状态）

###  同步阻塞 IO（BIO）

​	阻塞IO就是当应用B发起读取数据申请时，在内核数据没有准备好之前，应用B会一直处于等待数据状态，直到内核把数据准备好了交给应用B才结束。	

**术语描述**：

​	在应用**调用recvfrom读取数据**时，其系统调用直到**数据包到达且被复制到应用缓冲区中或者发送错误时才返回**，在此期间一直会等待，进程从调用到返回这段时间内都是被阻塞的称为阻塞IO；

**流程：**

- 应用进程向内核发起recfrom读取数据。
- 准备数据报（应用进程阻塞）。
- 将数据从内核复制到应用空间（因为我们的用户程序只能获取用户空间的内存，无法直接获取内核空间的内存）。
- 复制完成后，返回成功提示。

==阻塞两个地方：1. OS等待数据报准备好。2.将数据从内核空间拷贝到用户空间。==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308182735775.png" alt="image-20250308182735775" style="zoom:50%;" />

### 同步非阻塞 NIO（Nonblocking I/O）

​	所谓非阻塞IO就是当应用B发起读取数据申请时，如果内核数据没有准备好会即刻告诉应用B，不会让B在这里等待。

**术语：**

​	非阻塞IO是在应用调用recvfrom读取数据时，如果该缓冲区没有数据的话，就会直接返回一个EWOULDBLOCK错误，不会让应用一直等待中。在没有数据的时候会即刻返回错误标识，那也意味着如果**应用要读取数据就需要不断的调用recvfrom请求，直到读取到它数据要的数据为止。**

**流程：**

- 应用进程向内核发起recvfrom读取数据。
- 没有数据报准备好，即刻返回EWOULDBLOCK错误码。
- 应用进程向内核发起recvfrom读取数据。
- 已有数据包准备好就进行步骤，否则还是返回错误码。
- 将数据从内核拷贝到用户空间。
- 完成后，返回成功提示。

​	==NIO不会在recvfrom也就是socket.read()时候阻塞，但是还是会在将数据从内核空间拷贝到用户空间阻塞。一定要注意这个地方，Non-Blockin还是会阻塞的。==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20201121163344.jpg" alt="img" style="zoom:50%;" />

### I/O 多路复用（IO Multiplex）（select、poll、epoll的区别）

<u>**应用场景：**</u>

​	==IO实际指的就是网络的IO、多路也就是多个不同的tcp连接；复用也就是指使用同一个线程合并处理多个不同的IO操作，这样的话可以减少CPU资源==。（单个线程可以同时处理多个不同的IO操作，应用场景非常广泛：redis原理。Mysql连接原理）

- 为什么**Nginx、redis**能够支持非常高的并发最终都是靠的linux版本的**IO多路复用机制epoll**
- Redi的底层是采用NIO多路IO复用机制实现对多个不同的连接（tcp）实现IO的复用；能够非常好的支持高并发，同时能够先天性支持线程安全的问题。

**<u>注意：</u>**

​	==IO复用模型里面的select虽然可以监控多个fd了，但select其实现的本质上还是通过不断的轮询fd来监控数据状态，只是应用程序不需要自己主动去轮询了==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20201121163408.jpg" alt="img" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308192756248.png" alt="image-20250308192756248" style="zoom:50%;" />

**<u>（1）并发环境下的IO</u>**

​	如果在并发的环境下，可能会N个人向应用B发送消息，这种情况下我们的应用就必须创建多个线程去读取数据，每个线程都会自己调用recvfrom去读取数据。那么此时情况可能如下图：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308183904620.png" alt="image-20250308183904620" style="zoom:50%;" />

​	并发情况下服务器很可能一瞬间会收到几十上百万的请求，这种情况下应用B就需要创建几十上百万的线程去读取数据，同时又因为应用线程是不知道什么时候会有数据读取，为了保证消息能及时读取到，**那么这些线程自己必须不断的向内核发送recvfrom请求来读取数据；显然为每个请求分配一个进程/线程的方式不合适**。

**<u>（2）解决方案</u>**

​	IO复用模型的思路就是**系统提供了一种系统调用函数可以同时监控多个连接的操作**，这个调用函数就是我们常说到的**<font color = '#8D0101'>select、poll、epoll函数</font>**。

**<u>1）select和poll函数</u>**

**<u>select实现多路复用的方式：</u>**

- 将已连接的Socket都放到一个集合中（这个集合是使用固定长度的**<font color = '#8D0101'>BitMap（位图）</font>**来实现的，而且**所支持的文件描述符的个数是有限制的）**
- **调用select函数将文件描述符集合拷贝到内核里，通过遍历集合的方式让内核来检查是否有网络事件产生**，当检查到有事件产生后，将此Socket标记为可读或可写，接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的Socket，然后再对其处理。
- 总结一下，就是**Select函数需要2次「遍历」文件描述符集合**，一次是在内核态里，一次是在用户态里；而且还会发生**2次「拷贝」文件描述符集合**，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中

**<u>poll函数</u>**

​	poll不再用BitMap来存储所关注的文件描述符，取而代之用**<font color = '#8D0101'>动态数组，以链表形式来组织</font>，突破了select的文件描述符个数限制**

**<u>二者的区别</u>**

- 但是poll和select并没有太大的本质区别，**都是使用「线性结构」存储进程关注的Socket集合，因此都需要遍历文件描述符集合来找到可读或可写的Socket，时间复杂度为O(n)**
- 而且也**需要在用户态与内核态之间拷贝文件描述符集合**，这种方式随着并发数上来，性能的损耗会呈指数级增长。

**<u>2）epoll函数</u>**

<u>**过程**：</u>

- 在内核创建一个epoll对象
- 将**待检测的socket拷贝到epoll的红黑树中**
- **通过回调函数将有网络事件发生的socket移动到链表中**
- **将链表拷贝到用户态**

**<u>epoll通过两个方面，很好解决了select/poll的问题：</u>**

- epoll在内核里使用**红黑树来跟踪进程所有待检测的文件描述字**
  - 把需要监控的socket通过epoll_ctl()函数加入内核中的红黑树里。红黑树是个高效的数据结构，增删改一般时间复杂度是O(logn)。
  - 而select/poll内核里没有类似epoll红黑树这种保存所有待检测的socket的数据结构，所以select/poll每次操作时都传入整个socket集合给内核，而epoll因为在内核维护了红黑树，可以保存所有待检测的socket，所以只需要传入一个待检测的socket，**减少了内核和用户空间大量的数据拷贝和内存分配**。
- **epoll使用事件驱动的机制**，内核里**维护了一个链表来记录就绪事件**
  - 当**某个socket有事件发生**时，**通过回调函数，内核会将其加入到这个就绪事件列表中**，当用户调用epoll_wait()函数时，只会返回有事件发生的文件描述符的个数，不需要像select/poll那样轮询扫描整个socket集合，大大提高了检测的效率

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308190841585.png" alt="image-20250308190841585" style="zoom:50%;" />

**<u>3）边缘触发和水平触发：</u>**

​	epoll支持两种事件触发模式，分别是边缘触发（edge-triggered，ET）和水平触发（level-triggered，LT）。

- 水平触发的意思是只要满足事件的条件，比如内核中有数据需要读，就一直不断地把这个事件传递给用户；
- 而边缘触发的意思是只有第一次满足条件的时候才触发，之后就不会再传递同样的事件了

**<u>Eg：</u>**

​	你的快递被放到了一个快递箱里，如果快递箱只会通过短信通知你一次，即使你一直没有去取，它也不会再发送第二条短信提醒你，这个方式就是边缘触发；如果快递箱发现你的快递没有被取出，它就会不停地发短信通知你，直到你取出了快递，它才消停，这个就是水平触发的方式

### 信号驱动 I/O

​	IO复用模型里面的select虽然可以监控多个fd了，但select其实现的本质上还是通过不断的轮询fd来监控数据状态。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308192456064.png" alt="image-20250308192456064" style="zoom:100%;" />

​	信号驱动IO不是用循环请求询问的方式去监控数据就绪状态，而是在**调用sigaction时候建立一个SIGIO的信号联系，当内核数据准备好之后再通过SIGIO信号通知线程数据准备好后的可读状态，当线程收到可读状态的信号后，此时再向内核发起recvfrom读取数据的请求**，因为信号驱动IO的模型下应用线程在发出信号监控后即可返回，不会阻塞，所以这样的方式下，一个应用线程也可以同时监控多个fd

### 异步AIO(Asynchronous IO)

​	其实经过了上面两个模型的优化，我们的效率有了很大的提升，但是我们当然不会就这样满足了，有没有更好的办法，通过观察我们发现：

- 不管是IO复用还是信号驱动，我们要读取一个数据总是要发起两阶段的请求，**第一次发送select请求，询问数据状态是否准备好，第二次发送recevform请求读取数据**
- 应用告知内核启动某个操作，并让内核在整个操作完成之后，通知应用，这种模型与信号驱动模型的主要区别在于，信号驱动IO只是由内核通知我们合适可以开始下一个IO操作，而异步IO模型是由内核通知我们操作什么时候完成。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308192207544.png" alt="image-20250308192207544" style="zoom:80%;" />

​	异步IO的优化思路是解决了应用程序需要先后发送询问请求、发送接收数据请求两个阶段的模式，在异步IO的模式下，只需要向内核发送一次请求就可以完成状态询问和数拷贝的所有操作

## Java IO模型

### java的BIO

> BIO（blocking IO） 即阻塞 IO。指的主要是传统的 `java.io` 包，它基于流模型实现。

**<u>（1） BIO 简介</u>**

​	`java.io` 包提供了我们最熟知的一些 IO 功能，比如 **File 抽象、输入输出流等**。交互方式是同步、阻塞的方式，也就是说，在读取输入流或者写入输出流时，在读、写动作完成之前，线程会一直阻塞在那里，它们之间的调用是可靠的线性顺序。

​	很多时候，人们也把 java.net 下面提供的部分网络 API，比如 `Socket`、`ServerSocket`、`HttpURLConnection` 也归类到同步阻塞 IO 类库，因为网络通信同样是 IO 行为。

​	BIO 的优点是代码比较简单、直观；缺点则是 IO 效率和扩展性存在局限性，容易成为应用性能的瓶颈。

**<u>（2）BIO 的性能缺陷</u>**

**BIO 会阻塞进程，不适合高并发场景**。

​	采用 BIO 的服务端，通常由一个独立的 Acceptor 线程负责监听客户端连接。服务端一般在`while(true)` 循环中调用 `accept()` 方法等待客户端的连接请求，一旦接收到一个连接请求，就可以建立 Socket，并基于这个 Socket 进行读写操作。此时，不能再接收其他客户端连接请求，只能等待当前连接的操作执行完成。

- 如果要让 **BIO 通信模型 能够同时处理多个客户端请求，就必须使用多线程**（主要原因是`socket.accept()`、`socket.read()`、`socket.write()` 涉及的三个主要函数都是同步阻塞的），但会造成不必要的线程开销。不过可以通过 **线程池机制** 改善，线程池还可以让线程的创建和回收成本相对较低。
- **即使可以用线程池略微优化，但是会消耗宝贵的线程资源，并且在百万级并发场景下也撑不住**。如果并发访问量增加会导致线程数急剧膨胀可能会导致线程堆栈溢出、创建新线程失败等问题，最终导致进程宕机或者僵死，不能对外提供服务。

###  java的NIO（NIO与BIO的区别）

> NIO（non-blocking IO） 即非阻塞 IO。指的是 Java 1.4 中引入的 `java.nio` 包。

​	为了解决 BIO 的性能问题， Java 1.4 中引入的 `java.nio` 包。NIO 优化了内存复制以及阻塞导致的严重性能问题。

​	`java.nio` 包提供了 `Channel`、`Selector`、`Buffer` 等新的抽象，可以构建多路复用的、同步非阻塞 IO 程序，同时提供了更接近操作系统底层的高性能数据操作方式。

**<u>（1）Non-blocking IO（非阻塞IO）</u>**

​	**IO流是阻塞的，NIO流是不阻塞的。**

​	==NIO不会在recvfrom也就是socket.read()时候阻塞，但是还是会在将数据从内核空间拷贝到用户空间阻塞。一定要注意这个地方，Non-Blockin还是会阻塞的。==

- **Java NIO使我们可以进行非阻塞IO操作**，比如说：
  - **非阻塞读：**单线程中从通道读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。
  - **非阻塞写：**一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情
- Java BIO的各种流是阻塞的。
  - 这意味着，当一个线程调用read()或write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了

**<u>（2）Buffer（缓冲区）</u>**

<font color = '#8D0101'>**IO面向流(Stream oriented)，而NIO面向缓冲区(Buffer oriented)**</font>

- `Buffer` 是一块连续的内存块，是 NIO 读写数据的缓冲。**`Buffer` 可以将文件一次性读入内存再做后续处理，而传统的方式是边读文件边处理数据。**
- 在NIO厍中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的;在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。最常用的缓冲区是ByteBuffer,一个ByteBuffer提供了一组功能用于操作byte数组。除了ByteBuffer,还有其他的一些缓冲区，事实上，每一种Java基本类型（除了Boolean类型）都对应有一种缓冲区。

<u>**（3）Channel(通道)**</u>

- NIO需要利用通道和缓存类（ByteBuffer）配合完成读写等操作，**表示缓冲数据的源头或者目的地，它用于读取缓冲或者写入数据，是访问缓冲的接口**。
- 通道是双向的，可读也可写，而流的读写是单向的。无论读写，通道只能与Buffer交互。因为Buffer，通道可以异步地读写，即读写是可以同时进行的

<u>**（4）Selector(选择器)**</u>

- NIO有选择器，而IO没有。NIO利用这个可以实现IO多路复用。
- **选择器用于使用单个线程处理多个通道**。因此，它需要较少的线程来处理这些通道。线程之间的切换对于操作系统来说是昂贵的。因此，为了提高系统效率选择器是有用的。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308221627750.png" alt="image-20250308221627750" style="zoom:50%;" />

**<u>（4）NIO读数据和写数据方式</u>**

通常来说NIO中的所有IO都是从Channel（通道）开始的。

- 从通道进行数据读取：创建一个缓冲区，然后请求通道读取数据。
- 从通道进行数据写入：创建一个缓冲区，填充数据，并要求通道写入数据。

数据读取和写入操作图示

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308221718698.png" alt="image-20250308221718698" style="zoom:50%;" />

### java的AIO

> AIO（Asynchronous IO） 即异步非阻塞 IO，指的是 Java 7 中，对 NIO 有了进一步的改进，也称为 NIO2，引入了异步非阻塞 IO 方式。

​	在 Java 7 中，NIO 有了进一步的改进，也就是 NIO 2，引入了异步非阻塞 IO 方式，也有很多人叫它 AIO（Asynchronous IO）。异步 IO 操作基于事件和回调机制，可以简单理解为，应用操作直接返回，而不会阻塞在那里，当后台处理完成，操作系统会通知相应线程进行后续工作



# java集合、容器

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200221175550.png" alt="img" style="zoom:50%;" />

## 集合，容器框架

集合、数组都是对多个数据进行存储操作的结构，简称Java容器

此时的存储，主要是指**内存层面的存储，不涉及到持久化的存储**（.txt,.jpg,.avi,数据库中）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308222330282.png" alt="image-20250308222330282" style="zoom:80%;" />

Java 容器框架主要分为 `Collection` 和 `Map` 两种。其中，`Collection` 又分为 `List`、`Set` 以及 `Queue`。

- **`Collection`：**一个独立元素的序列，这些元素都服从一条或者多条规则。
  - `List` - 必须按照插入的顺序保存元素。
  - `Set` - 不能有重复的元素。
  - `Queue` - 按照排队规则来确定对象产生的顺序（通常与它们被插入的顺序相同）。
- **`Map`** - 一组成对的“键值对”对象，允许你使用键来查找值。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308222441308.png" alt="image-20250308222441308" style="zoom:60%;" />

### 集合框架底层数据结构总结

先来看一下 `Collection` 接口下面的集合。

**<u>（1）List</u>**

- `ArrayList`：`Object[]` 数组。详细可以查看：[ArrayList 源码分析]()。
- `Vector`：`Object[]` 数组。
- `LinkedList`：双向链表(JDK1.6 之前为循环链表，JDK1.7 取消了循环)。详细可以查看：[LinkedList 源码分析]()。

**<u>（2）Set</u>**

- `HashSet`(无序，唯一): 基于 `HashMap` 实现的，底层采用 `HashMap` 来保存元素。
- `LinkedHashSet`: `LinkedHashSet` 是 `HashSet` 的子类，并且其内部是通过 `LinkedHashMap` 来实现的。
- `TreeSet`(有序，唯一): 红黑树(自平衡的排序二叉树)。

**<u>（3）Queue</u>**

- `PriorityQueue`: `Object[]` 数组来实现小顶堆。详细可以查看：[PriorityQueue 源码分析]()。
- `DelayQueue`:`PriorityQueue`。详细可以查看：[DelayQueue 源码分析]()。
- `ArrayDeque`: 可扩容动态双向数组。

再来看看 `Map` 接口下面的集合。

**<u>Map</u>**

- `HashMap`：JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。详细可以查看：[HashMap 源码分析]()。

- `LinkedHashMap`：`LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。详细可以查看：[LinkedHashMap 源码分析]()

- `Hashtable`：数组+链表组成的，数组是 `Hashtable` 的主体，链表则是主要为了解决哈希冲突而存在的。

- `TreeMap`：红黑树（自平衡的排序二叉树）。

### List，set，map的区别

- List可以存储重复的元素，多个元素指向相同的引用。
- Set不允许重复的元素，不会有多个元素引用相同的对象。
- Map使用键值对来存储，**两个key可以引用相同的对象，但是key不能重复**

### 数组与容器

Java 中常用的存储容器就是数组和容器，二者有以下区别：

- 存储大小是否固定
  - 数组的**长度固定**；
  - 容器的**长度可变**。
- 数据类型
  - **数组可以存储基本数据类型，也可以存储引用数据类型**；
  - **容器只能存储引用数据类型**，**基本数据类型的变量要转换成对应的包装类才能放入容器类**中。

## 容器的基本机制

### 泛型

Java 1.5 引入了泛型技术。

​	Java **容器通过泛型技术来保证其数据的类型安全**。什么是类型安全呢？

​	举例来说：如果有一个 `List<Object>` 容器，Java **编译器在编译时不会对原始类型进行类型安全检查**，却会对带参数的类型进行检查，通过使用 Object 作为类型，可以告知编译器该方法可以接受任何类型的对象，比如 String 或 Integer。

```java
List<Object> list = new ArrayList<Object>();
list.add("123");
list.add(123);
```

如果没有泛型技术，如示例中的代码那样，容器中就可能存储任意数据类型，这是很危险的行为。

```text
List<String> list = new ArrayList<String>();
list.add("123");
list.add(123);
```

### Iterable、Iterator迭代器

**<font color = '#8D0101'>迭代器模式</font>** - **提供一种方法顺序访问一个聚合对象中各个元素，而又无须暴露该对象的内部表示**

**<u>（1）为什么有Iterator还需要Iterable</u>**

- `Iterator`接口的核心方法next()或者hashNext()，previous()等，都是**严重依赖于指针**的，也就是迭代的目前的位置。如果Collection直接实现`Iterator`接口，那么集合对象就拥有了指针的能力，内部不同方法传递，就会让next()方法互相受到阻挠。只有一个迭代位置，互相干扰。  
- **`Iterable` 每次获取迭代器，就会返回一个从头开始的，不会和其他的迭代器相互影响。**  
- 这样子也是解耦合的一种，有些集合不止有一个`Iterator`内部类，可能有两个，比如`ArrayList`，`LinkedList`，可以获取不同的`Iterator`执行不一样的操作。

**<u>（2）Iterator</u>**

​	Iterator对象称为迭代器（设计模式的一种），主要用于遍历Collection集合中的元素，**提供一种方法访问一个容器对象中各个元素，而又不需暴露该对象的内部细节**。

```java
public interface Iterator<E> {
    boolean hasNext(); // 是否有下一个元素
		E next();   // 获取下一个元素

// 移除元素
		default void remove() {
        throw new UnsupportedOperationException("remove");
    }
// 对剩下的所有元素进行处理，action则为处理的动作，意为要怎么处理
		default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

- 集合对象每次调用iterator()方法都得到一个全新的迭代器对象，默认游标都在集合的第一个元素之前

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308225326063.png" alt="image-20250308225326063" style="zoom:66%;" />

**<u>Iterator的remove方法（fail fast）</u>**	

remove方法：内部定义了remove(),可以在遍历的时候，删除集合中的元素。**此方法不同于集合直接调用remove()**

**<u>注意：</u>**

- 在迭代集合元素的过程中，不能调用集合对象的remove方法，删除元素
- 获取迭代器对象，迭代器用来遍历集合，此时相当于对当前集合的状态拍了一个快照。迭代器迭代的时候会参照这个快照进行迭代。在迭代元素的过程当中，一定要使用迭代器iterator的remove方法删除元素

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308225518660.png" alt="image-20250308225518660" style="zoom:55%;" />

**（3）iterable接口**

​	`iterable`接口其实是java集合大家庭的最顶级的接口之一了，实现这个接口，可以视为拥有了获取迭代器的能力。`Iterable`接口出现在JDK1.5，那个时候只有`iterator()`方法，主要是定义了迭代集合内元素的规范。从字面的意思看，是指可以迭代的接口。

```java
public interface Iterable<T> {

// 返回一个内部元素为T类型的迭代器（JDK1.5只有这个接口）
		Iterator<T> iterator();

// 遍历内部元素，action意思为动作，指可以对每个元素进行操作（JDK1.8添加）
		default void forEach(Consumer<? super T> action) {}

// 创建并返回一个可分割迭代器（JDK1.8添加），分割的迭代器主要是提供可以并行遍历元素的迭代器，可以适应现在cpu多核的能力，加快速度。
		default Spliterator<T> spliterator() {
    		return Spliterators.spliteratorUnknownSize(iterator(), 0);
		}
}
```

### Comparable 和 Comparator

​	`Comparable` 是排序接口。若一个类实现了 `Comparable` 接口，表示该类的实例可以比较，也就意味着支持排序。实现了 `Comparable` 接口的类的对象的列表或数组可以通过 `Collections.sort` 或 `Arrays.sort` 进行自动排序。

<u>**`Comparable` 接口定义：**</u>

```java
public interface Comparable<T> {
    public int compareTo(T o);
}
```

`Comparator` 是比较接口，我们如果需要控制某个类的次序，而该类本身不支持排序(即没有实现 `Comparable` 接口)，那么我们就可以建立一个“该类的比较器”来进行排序，这个“比较器”只需要实现 `Comparator` 接口即可。也就是说，我们可以通过实现 `Comparator` 来新建一个比较器，然后通过这个比较器对类进行排序。

<u>**`Comparator` 接口定义：</u>**

```java
@FunctionalInterface
public interface Comparator<T> {

    int compare(T o1, T o2);

    boolean equals(Object obj);

    // 反转
    default Comparator<T> reversed() {
        return Collections.reverseOrder(this);
    }

    default Comparator<T> thenComparing(Comparator<? super T> other) {
        Objects.requireNonNull(other);
        return (Comparator<T> & Serializable) (c1, c2) -> {
            int res = compare(c1, c2);
            return (res != 0) ? res : other.compare(c1, c2);
        };
    }

    // thenComparingXXX 方法略

    // 静态方法略
}
```

在 Java 容器中，一些可以排序的容器，如 `TreeMap`、`TreeSet`，都可以通过传入 `Comparator`，来定义内部元素的排序规则。

## List接口

`List` 是 `Collection` 的子接口，其中可以保存各个重复的内容。

​	鉴于Java中数组用来存储数据的局限性，通常使用List替代数组。List集合类中元素**有序、可重复**，集合中的每个元素都有其对应的顺序索引。List容器中的元素都对应一个整数型的序号记载其在容器中的位置，可以根据序号存取容器中的元素。List接口的实现类常用的有：ArrayList、LinkedList和Vector

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308230259493.png" alt="image-20250308230259493" style="zoom:70%;" />

### ArrayList

#### ArrayList概述和源码

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20220529190340.png" alt="img" style="zoom: 55%;" />

​	ArrayList是List接口的典型实现类、主要实现类。本质上，**<font color = '#8D0101'>ArrayList是对象引用的一个”变长”数组</font>**

- **<u>jdk7情况下</u>**：
  - 建议开发中使用带参的构造器：ArrayList list=new ArrayList(int capacity)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308230644891.png" alt="image-20250308230644891" style="zoom:67%;" />

- <u>**jdk8及以后源码分析**</u>
  - **jdk7**中的ArrayList的对象的创建类似于**单例的饿汉式**，而**jdk8**中的ArrayList的对象的创建类似于**单例的懒汉式**，延迟了数组的创建，节省内存

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308230800750.png" alt="image-20250308230800750" style="zoom:67%;" />

#### ArrayList扩容机制

​	**初始为10**，每次newcapacity变为oldcapacity的1.5倍。每次扩容都**新建一个数组，将旧数组的元素都复制到新数组里边**

![image-20250308231143322](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308231143322.png)

#### ArrayList的add（index，Element）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308231325106.png" alt="image-20250308231325106" style="zoom:66%;" />

#### ArrayList循环遍历并删除元素的常见错误

不能在ArrayList的For-Each循环中删除元素

**（1）普通for循环正序删除**，删除过程中元素向左移动，会漏删相邻的重复元素，因此不能删除重复的元素

- 针对这种情况可以倒序删除的方式来避免.

  因为数组倒序遍历时即使发生元素删除也不影响后序元素遍历。

**（2）增强for循环删除**，使用ArrayList的remove()方法删除，产生**并发修改异常ConcurrentModificationException**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308231613621.png" alt="image-20250308231613621" style="zoom:67%;" />

- ArrayList中创建了一个内部迭代器Itr，并实现了Iterator接口，而For-Each遍历正是基于这个迭代器的hasNext()和next()方法来实现的；
- ArrayList里还保存了一个变量modCount，用来记录List修改的次数，而iterator保存了一个expectedModCount来表示期望的修改次数，在每个操作前都会判断两者值是否一样，不一样则会抛出异常：ConcurrentModificationException异常

**（3）一定要使用迭代器iterator的remove方法删除元素**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308231842681.png" alt="image-20250308231842681" style="zoom:67%;" />

#### Arrays.asList 问题点

​	在业务开发中，我们常常会把原始的数组转换为 `List` 类数据结构，来继续展开各种 `Stream` 操作。通常，我们会使用 `Arrays.asList` 方法可以把数组一键转换为 `List`。

<u>**(1) 固定大小**</u>

返回的 `List` 是底层数组的**视图**，因此长度固定：

- **支持修改元素值**（通过 `set` 方法）。
- **不支持增删操作**（`add` 或 `remove` 会抛出 `UnsupportedOperationException`）。

```
List<String> list = Arrays.asList("A", "B", "C");
list.set(0, "X");    // ✅ 允许修改
list.add("D");       // ❌ 抛出异常
list.remove(0);      // ❌ 抛出异常
```

<u>**(2) 与原数组关联**</u>

修改 `List` 的元素值会直接影响原数组：

```
String[] array = {"A", "B", "C"};
List<String> list = Arrays.asList(array);
list.set(0, "X");
System.out.println(array[0]); // 输出 "X"
```

<u>**(3) 不支持基本类型数组**</u>

如果传入基本类型数组（如 `int[]`），`List` 会包含单个数组对象，而非数组元素：

```
int[] intArray = {1, 2, 3};
List<int[]> list = Arrays.asList(intArray); // List<int[]>，而非 List<Integer>
```

要解决此问题，**<font color = '#8D0101'>需使用包装类数组</font>**：

```
Integer[] integerArray = {1, 2, 3};
List<Integer> list = Arrays.asList(integerArray); // 正确
```

<u>**（4）`Arrays.asList` 返回的 `List` 不支持增删操作。**</u>

- `Arrays.asList()`将数组转换为集合后,底层其实还是数组，《阿里巴巴》Java 开发使用手册对于这个方法有如下描述：
- `Arrays.asList()`返回的是一个固定大小的列表，底层确实基于原始数组。这意味着如果用户尝试对这个列表进行`add()`或`remove()`操作，会抛出`UnsupportedOperationException`。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/640" alt="图片" style="zoom:67%;" />

==**解决办法**==

- **直接创建新 `ArrayList`**

将数组转换为可修改的 `ArrayList`：

```java
String[] array = {"A", "B", "C"};
List<String> list = new ArrayList<>(Arrays.asList(array));
```

​	此时 `list` 是一个全新的 `ArrayList` 对象，支持 `add()`、`remove()` 等操作，且与原数组**无关联**。

- **使用 Java 8 Stream（复杂场景推荐）**

通过 `Stream` 转换数组为 `List`：

```java
String[] array = {"A", "B", "C"};
List<String> list = Arrays.stream(array)
                          .collect(Collectors.toList());
```

==注意：`Collectors.toList()` 返回的 `List` 实现类（如 `ArrayList`）是 **可修改** 的==

**何时用 `Arrays.asList()`？**

- 需要 **快速创建固定大小的列表**（只读或仅修改元素值）。
- 希望列表与原数组 **保持同步修改**（例如通过 `list.set(0, "X")` 会同步修改原数组）。

**何时用 `new ArrayList<>()`？**

- 需要 **动态增删元素**。
- 希望列表与原数组 **完全解耦**。

### LinkedList

#### LinkedList概述和源码

**`LinkedList` 内部维护了一个双链表**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308232534025.png" alt="image-20250308232534025" style="zoom:70%;" />

`LinkedList` 通过 `Node` 类型的头尾指针（`first` 和 `last`）来访问数据。

```java
// 链表长度
transient int size = 0;
// 链表头节点
transient Node<E> first;
// 链表尾节点
transient Node<E> last;
```

- `size` - **表示双链表中节点的个数，初始为 0**。
- `first` 和 `last` - **分别是双链表的头节点和尾节点**。

`Node` 是 `LinkedList` 的静态内部类，它表示链表中的元素实例，作为LinkedList中保存数据的基本结构。Node 中包含三个元素：

- `prev` 是该节点的上一个节点；
- `next` 是该节点的下一个节点；
- `item` 是该节点所包含的值。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    ...
}
```

**<u>源码分析</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308232607643.png" alt="image-20250308232607643" style="zoom:67%;" />

#### linkedlist是双向链表，对比单向链表的优势

​	LinkedList的查找底层是通过**node(int index)方法**实现的，这个方法会先判读index是否大于链表的长度的一半，如果大于就从尾部开始遍历，如果不大于，就从头部开始遍历。

### ArrayList与LinkedList区别？

**<u>（1）是否线程安全？</u>**

- 都不同步，不保证线程安全。

**<u>（2）底层数据结构？</u>**

- ArrayList底层使用object数组实现。LinkedList底层使用的是双向链表（1.6前为循环链表，1.7取消了循环）

**<u>（3）插入和删除是否受元素位置的影响？</u>**

- ArrayList插入到末尾，时间复杂度o(1),插入到i位置或者从i位置删除，元素要向前或者向后移动一位，时间复杂度是O(n-i)；

- LinkedList，直接删除或插入O(1)，在指定位置插入或者删除O(n)，因为它要移动到指定位置再删除

**<u>（4）是否支持随机访问？</u>**

- ArrayList支持，通过元素的序号快速访问元素（get（int index）方法）；
- 而LinkedList是不支持随机访问的。

**<u>（5）内存空间占用？</u>**

- ArrayList空间浪费主要体现在list列表的结尾会有一定的容量空间。
- LinkedList的空间花费体现在它每一个元素都需要消耗比ArrayList更多的空间（不仅存储数据，还有前驱和后驱）

**（6）list的遍历方式选择**

- ArrayList实现了RandomAccess接口，LinkedList没有实现，它只是一个标识。
- 实际上RandomAccess接口中什么都没有定义，所以RandomAccess只是用来标识实现这个接口的类具有随机访问功能（由于底层数据结构的影响，ArrayList实现了RandomAccess接口，而LinkedList没有实现，表明ArrayList具有快速随机访问功能，但是并不是说ArrayList实现RandomAccess接口才具有快速随机访问功能的）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250308233012235.png" alt="image-20250308233012235" style="zoom: 67%;" />

### Vector概述

​	Vector是一个古老的集合，JDK1.0就有了。大多数操作与ArrayList相同，区别之处在于Vector是线程安全的。

​	在各种list中，最好把ArrayList作为缺省选择。当插入、删除频繁时，使用LinkedList；Vector总是比ArrayList慢，所以尽量避免使用

​	Vector的源码分析：jdk7和jdk8中通过Vector()构造器创建对象时，底层都创建了长度为10的数组。

在扩容方面，默认扩容为原来的数组长度的2倍

### ArrayList与Vector的区别

​	Vector类的所有方法都使用了关键字synchronized使它们同步，可以使多个线程安全地访问一个对象，但是一个线程访问的时候要在同步操作耗费大量的时间。而ArrayList是不同步的，所以在不需要保证线程安全的时候应该使用ArrayList。

## Set接口

### Set简介

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Set-diagrams.png" alt="img" style="zoom:60%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309000717555.png" alt="image-20250309000717555" style="zoom:75%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309000721800.png" alt="image-20250309000721800" style="zoom:75%;" />

- `HashSet` 类依赖于 `HashMap`，它实际上是通过 `HashMap` 实现的。`HashSet` 中的元素是无序的、散列的。

- `TreeSet` 类依赖于 `TreeMap`，它实际上是通过 `TreeMap` 实现的。`TreeSet` 中的元素是有序的，它是按自然排序或者用户指定比较器排序的 Set。
- `LinkedHashSet` 是按插入顺序排序的 Set

### Set的无序性与不可重复性

​	**Set接口中没有定义额外的方法，使用的都是Collection中声明过的方法。**

Set:存储无序的、不可重复的数据

- **无序性**：不等于随机性。存储的数据在底层数组中并非按照数组索引的顺序添加，而是根据数据的哈希值决定的。
- **不可重复性**：保证添加的元素按照equals()判断时，不能返回true.即：相同的元素只能添加一个。

### HashSet 类

#### HashSet简介

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309001228029.png" alt="image-20250309001228029" style="zoom:70%;" />

​	`HashSet` 类依赖于 `HashMap`，它实际上是通过 `HashMap` 实现的。`HashSet` 中的元素是无序的、散列的。

`HashSet` 类定义如下：

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {}
```

#### `HashSet` 是基于 `HashMap` 实现的

```java
// HashSet 的核心，通过维护一个 HashMap 实体来实现 HashSet 方法
private transient HashMap<E,Object> map;

// PRESENT 是用于关联 map 中当前操作元素的一个虚拟值
private static final Object PRESENT = new Object();
}
```

- HashSet 中维护了一个 HashMap 对象 map，HashSet 的重要方法，如 add、remove、iterator、clear、size 等都是围绕 map 实现的。
  - HashSet 类中通过定义 writeObject() 和 readObject() 方法确定了其序列化和反序列化的机制。
- PRESENT 是用于关联 map 中当前操作元素的一个虚拟值。

#### HashSet添加元素、扩容

<u>**（1）添加元素的过程**</u>

​	我们向HashSet中添加元素a，首先调用元素a所在类的hashCode()方法，计算元素a的哈希值。**此哈希值接着通过(n-1)&hash计算出在HashSet底层数组中的存放位置（即为：索引位置）**，判断数组此位置上是否已经有元素：

- 如果此位置上没有其他元素，则元素a添加成功。--->情况1 

- 如果此位置上有其他元素b(或以链表形式存在的多个元素），则比较元素a与元素b的hash值：
  - 如果hash值不相同，则元素a添加成功。--->情况2；
  - 如果hash值相同，进而需要调用元素a所在类的equals()方法：
    - equals()返回true,元素a添加失败
    - equals()返回false,则元素a添加成功。--->情况3

​	对于添加成功的情况2和情况3而言：**元素a与已经存在指定索引位置上数据以链表的方式存储**。

- **jdk7:**元素a放到数组中，指向原来的元素(头插法)

- **jdk8:**原来的元素在数组中，指向元素a(尾插法)

==HashSet底层：数组+链表的结构==

<u>**（2）扩容问题**</u>

​	底层也是数组，**<font color = '#8D0101'>初始容量为16（若超过16也必须为2的次幂），当使用率超过0.75时，就会扩大容量为原来的2倍</font>**

### LinkedHashSet类

`LinkedHashSet` 是按插入顺序排序的 Set。

`LinkedHashSet` 类定义如下：

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {}
```

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309115023039.png" alt="image-20250309115023039" style="zoom:55%;" />

- LinkedHashSet作为HashSet的子类，在添加数据的同时，每个数据还维护了两个引用，记录此数据前一个数据和后一个数据。

优点：对于频繁的遍历操作，LinkedHashSet效率高于HashSet

### TreeSet类

​	`TreeSet` 类依赖于 `TreeMap`，它实际上是通过 `TreeMap` 实现的。`TreeSet` 中的元素是有序的，它是按自然排序或者用户指定比较器排序的 Set。

`TreeSet` 类定义如下：

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {}
```

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309115320183.png" alt="image-20250309115320183" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309115336089.png" alt="image-20250309115336089" style="zoom:67%;" />

- 向TreeSet中添加的数据，要求是相同类的对象。
- 两种排序方式：自然排序（实现Comparable接口）和定制排序（Comparator）
  - 自然排序中，比较两个对象是否相同的标准为：compareTo()返回0.不再是equals().
  - 定制排序中，比较两个对象是否相同的标准为：compare()返回0.不再是equals()。

## HashSet与HashMap常用方法和遍历方法

**HashSet 常用方法**：

- `add(E e)`: 添加元素，成功返回 `true`，重复元素返回 `false`。
- `remove(Object o)`: 删除指定对象，存在则返回 `true`，否则 `false`。
- `contains(Object o)`: 检查是否包含指定对象。
- `size()`: 返回集合大小。
- `isEmpty()`: 判断是否为空。
- `clear()`: 清空集合。

**HashSet 遍历方法**：

**增强 for 循环**：

```java
HashSet<String> set = new HashSet<>();
set.add("A");
set.add("B");
for (String item : set) {
    System.out.println(item);
}
```

**HashMap 常用方法**：

- `put(K key, V value)`: 插入键值对，返回旧值（无则返回 `null`）。
- `get(Object key)`: 通过键获取值，不存在返回 `null`。
- `remove(Object key)`: 删除键对应的键值对，返回被删除的值。
- `containsKey(Object key)`: 检查是否存在键。
- `containsValue(Object value)`: 检查是否存在值。
- `size()`: 返回键值对数量。
- `clear()`: 清空所有映射。
- `keySet()`: 返回所有键的集合（`Set<K>`）。
- `values()`: 返回所有值的集合（`Collection<V>`）。
- `entrySet()`: 返回所有键值对的集合（Set<Map.Entry<K, V>>）。

**HashMap 遍历方法**：

1. **遍历键（Key）**：

   ```java
   HashMap<Integer, String> map = new HashMap<>();
   map.put(1, "A");
   map.put(2, "B");
   
   for (Integer key : map.keySet()) {
       System.out.println("Key: " + key);
   
   ```

2. **遍历值（Value）**：

   ```java
   for (String value : map.values()) {
       System.out.println("Value: " + value);
   }
   ```

3. **遍历键值对（Entry）**：**增强 for 循环 + entrySet()**：

   ```java
   for (Map.Entry<Integer, String> entry : map.entrySet()) {
       System.out.println("Key=" + entry.getKey() + ", Value=" + entry.getValue());
   }
   ```

## HashMap和HashSet对比

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309140528344.png" alt="image-20250309140528344" style="zoom:80%;" />

## Map接口

### Map接口简介

#### Map架构

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309130820764.png" alt="image-20250309130820764" style="zoom:50%;" />

|----Map:双列数据，存储key-value对的数据  ---类似于高中的函数：y = f(x)

- |----HashMap:作为Map的主要实现类；线程不安全的，效率高；存储null的key和value
  - |----LinkedHashMap:保证在遍历map元素时，可以按照添加的顺序实现遍历。
  - 原因：在原有的HashMap底层结构基础上，添加了一对指针，指向前一个和后一个元素。对于频繁的遍历操作，此类执行效率高于HashMap。

- ----TreeMap:保证按照添加的key-value对进行排序，实现排序遍历。此时考虑key的自然排序或定制排序
  - 底层使用红黑树

- |----Hashtable:作为古老的实现类；线程安全的，效率低；不能存储null的key和value
  - |----Properties:常用来处理配置文件。key和value都是String类型

#### Map中存储的key-value的特点:

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309131151455.png" alt="image-20250309131151455" style="zoom:50%;" />

- Map中的key:无序的、不可重复的，使用**Set存储所有的key--->key所在的类要重写equals()和hashCode()**（以HashMap为例）
- Map中的value:无序的、可重复的，使用**Collection存储所有的value--->value所在的类要重写equals()**

**<font color = '#8D0101'>一个键值对：key-value构成了一个Entry对象。</font>**

- Map中的entry:无序的、不可重复的，使用**Set存储所有的entry**

#### Map.Entry 接口

​	`Map.Entry` 一般用于通过迭代器（`Iterator`）访问 `Map`。

​	**`Map.Entry` 是 Map 中内部的一个接口**，此接口为泛型Entry<K,V>，`Map.Entry` 代表了 **键值对** 实体，Map 通过 `entrySet()` 获取 `Map.Entry` 集合，从而通过该集合实现对键值对的操作，接口中有getKey(),getValue方法

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309142238449.png" alt="image-20250309142238449" style="zoom:60%;" />

### HashMap

#### HashMap的概述与其常量

- JDK7及以前版本：==HashMap是数组+链表结构(即为链地址法)==
- JDK8版本发布以后：==HashMap是数组+链表+红黑树实现==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309131845768.png" alt="image-20250309131845768" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309131903140.png" alt="image-20250309131903140" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309131942382.png" alt="image-20250309131942382" style="zoom:70%;" />

#### HashMap 数据结构

**<u>`HashMap` 的核心字段：</u>**

- `table` - `HashMap` 使用一个 `Node<K,V>[]` 类型的数组 `table` 来储存元素。
- `size` - 初始容量。 初始为 16，每次容量不够自动扩容
- `loadFactor` - 负载因子。自动扩容之前被允许的最大饱和量，默认 0.75。

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    // 该表在初次使用时初始化，并根据需要调整大小。分配时，长度总是2的幂。
    transient Node<K,V>[] table;
    // 保存缓存的 entrySet()。请注意，AbstractMap 字段用于 keySet() 和 values()。
    transient Set<Map.Entry<K,V>> entrySet;
    // map 中的键值对数
    transient int size;
    // 这个HashMap被结构修改的次数结构修改是那些改变HashMap中的映射数量或者修改其内部结构（例如，重新散列）的修改。
    transient int modCount;
    // 下一个调整大小的值（容量*加载因子）。
    int threshold;
    // 散列表的加载因子
    final float loadFactor;
}
```

**<u>HashMap 构造方法</u>**

```java
public HashMap(); // 默认加载因子0.75
public HashMap(int initialCapacity); // 默认加载因子0.75；以 initialCapacity 初始化容量
public HashMap(int initialCapacity, float loadFactor); // 以 initialCapacity 初始化容量；以 loadFactor 初始化加载因子
public HashMap(Map<? extends K, ? extends V> m) // 默认加载因子0.75
```

#### Hashmap会一直扩容下去吗，hashmap容量

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309133053940.png" alt="image-20250309133053940" style="zoom: 67%;" />

因为map底层也有一个数组，**数组的最大值就是INT的最大值**

#### Hash()函数

**<font color = '#8D0101'>先进行hashcode()，再进行Hash()</font>**

​	==Map中的hash()函数(扰动函数)是对hashCode()方法获取的hash值进行了一些改动，增加随机性，让数据元素更加均衡地散列，减少碰撞==

​	所谓扰动函数指的就是HashMap的hash方法。使用hash方法也就是扰动函数是为了防止一些实现比较差的hashCode()方法，换句话说使用扰动函数之后可以减少碰撞。

1.7的hash方法的性能会稍差一点点，因为毕竟扰动了4次

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309134513388.png" alt="image-20250309134513388" style="zoom:67%;" />

#### JDK7的map:头插法

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/7ae0a7bd64d74b5d9a149f7849e38f93.gif" alt="在这里插入图片描述" style="zoom:50%;" />

在1.7中，多线程会造成死循环

- HashMap map = new HashMap();在实例化以后，**底层创建了长度是16的一维数组Entry[] tabale。**
- jdk7底层时数组和链表结合在一起使用也就是**链表散列**。**<font color = '#8D0101'>HashMap通过key的hashCode()经过扰动函数处理</font>过后得到hash值**，然后通过**<font color = '#8D0101'>(n-1)&hash</font>**判断当前元素存放的位置（n指数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的hash值以及key是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。
- 所谓**扰动函数指的就是HashMap的hash()方法**，使用hash()方法也就是扰动函数是为了防止一些实现比较差的hashCode()方法，即使用**扰动函数后可以减少哈希碰撞。**

​	**Map中的hash()函数(扰动函数)是对hashCode()方法获取的hash值进行了一些改动，增加随机性，让数据元素更加均衡地散列，减少碰撞**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309134711165.png" alt="image-20250309134711165" style="zoom:80%;" />

#### JDK7 put的过程

**map.put(key1,value1)的步骤（拉链法）**

​	首先，调用key1所在类的hashCode()经过扰动函数处理之后计算key1哈希值，此哈希值经过(n-1)&hash计算以后，得到Entry数组中的存放位置。

- 如果此位置上的数据为空，此时的entry1(key1-value1)添加成功。---情况1
- 如果此位置上的数据不为空，（意味着此位置上存在一个或多个数据（以链表形式存在），比较key1和已经存在的一个或多个数据的哈希值：
  - 1）如果key1的哈希值与已经存在的数据的哈希值都不相同，此时entry1(key1-value1)添加成功---情况2
  - 2）如果key1的哈希值与已经存在的某一个数据的哈希值相同，继续比较：
    - ①如果equals()返回false：此时entry1(key1-value1)添加成功。---情况3
    - ②如果equals()返回true：使用value1替换value2.（修改功能）		

> [!IMPORTANT]
>
> - 关于情况2和情况3：此时entry1(key1-value1)和原来的数据以链表的方式存储。
> - 当数组的某一个索引位置上的元素**<font color = '#8D0101'>以链表形式存在的数据个数>8、且当前数组的长度>64</font>(<64时直接扩容就好)时，此时此索引位置上的所数据改为使用红黑树存储**

#### JDK7扩容（resize）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309135130020.png" alt="image-20250309135130020" style="zoom:67%;" />

- 首先判断**`OldCap`有没有超过最大值**。
- 当**hashmap中的元素个数size**超过”**数组大小(capacity)\*`loadFactor`**”时，就会进行数组扩容，**`default_load_Factor`的默认值为0.75**，也就是说默认情况下数组大小为16，那么当hashmap中元素个数超过16\*0.75=12的时候，就把数组的大小扩展为2*16=32，即扩大一倍。
- 然后重新计算每个元素在数组中的位置，而这是一个非常消耗性能的操作，**所以如果我们已经预知hashmap中元素的个数，那么预设元素的个数能够有效的提高hashmap的性能**。

​	比如说，我们有1000个元素newHashMap(1000),但是理论上来讲new HashMap(1024)更合适，不过上面已经说过，即使是1000，hashmap也自动会将其设置为1024。但是new HashMap(1024)还不是更合适的，因0.75\*1000<1000,也就是说为了让0.75*size>1000,我们必须这样new HashMap(2048)才最合适，既考虑了&的问题，也避免了resize的问题。

#### JDK8：尾插法

jdk8相较于jdk7在底层实现方面的不同：

- new HashMap():底层没有创建一个长度为16的数组，类似饿汉式
- jdk8底层的数组是：**Node[]**,而非Entry[]
- **首次调用put()方法时，底层创建长度为16的数组**
- **jdk7底层结构只有：数组+链表。jdk8中底层结构：数组+链表+红黑树。**
  - 形成链表时，<font color = '#8D0101'>七上八下（jdk7:新的元素指向旧的元素（头插）。jdk8：旧的元素指向新的元素（尾插））</font>
  - 当数组的某一个索引位置上的元素**<font color = '#8D0101'>以链表形式存在的数据个数>8、且当前数组的长度>64</font>(<64时直接扩容就好)时，此时此索引位置上的所数据改为使用红黑树存储**。

> [!IMPORTANT]
>
> DEFAULT_INITIAL_CAPACITY:HashMap的默认容量，16
>
> DEFAULT_LOAD_FACTOR：HashMap的默认(负载)加载因子：0.75
>
> Threshold:扩容的临界值，=容量*填充因子 =>12
>
> TREEIFY_THRESHOLD:Bucket中链表长度大于该默认值，转化为红黑树：8
>
> MIN_TREEIFY_CAPACITY:桶中的Node被树化时最小的hash表容量：64

#### JDK8 扩容和树形化

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309140233941.png" alt="image-20250309140233941" style="zoom:65%;" />

#### 为什么使用红黑树，不使用其他的树

- 因为**树结点占用的存储空间是普通结点的两倍**。因此红黑树比链表更加耗费空间。结点较少的时候，时间复杂度上链表不会比红黑树高太多，但是能大大减少空间。
- 因为红黑树不追求完美的平衡，只要求达到部分平衡，可以减少增删结点时的左旋转和右旋转的次数。完美平衡树AVL树  

**<u>为什么JDK8中HashMap底层使用红黑树</u>**

- **链表查询的时间复杂度是O（n）,红黑树的时间复杂度为O（logn）。可以提升效率。**

#### 为什么链表长度大于阈值8时才将链表转为红黑树？

- 因为树结点占用的存储空间是普通结点的两倍(因为)。因此红黑树比链表更加耗费空间。结点较少的时候，时间复杂度上链表不会比红黑树高太多，但是能大大减少空间。
- 当链表元素个数大于等于8时，链表换成树结构；若桶中链表元素个数小于等于6时，树结构还原成链表。**因为红黑树的平均查找长度是log（n）,长度为8的时候，平均查找长度为3，如果继续使用链表，查找长度为8/2=4，这才有转换为树的必要。**

#### HashMap的长度为什么是2的幂次方

为了提高哈希计算效率。

- **要计算key插入哈希表中的位置，需要key%length(数组的长度)得到余数**，而如果余数是2的幂次方，则等价于与（&）操作，**即hash%length=hash&（length-1）**，把取模操作等价于与操作的前提是length是2的幂次方。**采用二进制操作&，能够提升%操作运算效率**，这就解释了hashmap的长度为什么是2的幂次方。

#### loadfactor是0.75的原因

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309142755347.png" alt="image-20250309142755347" style="zoom:80%;" />

#### HashMap在高并发情况的问题（jdk7）

- Hashmap在插入元素过多的时候需要进行Resize，Resize的条件是**HashMap.Size>=Capacity*LoadFactor。**
- Hashmap的Resize包含**扩容和ReHash**两个步骤，<font color = '#8D0101'>**ReHash在并发的情况下可能会形成链表环**</font>。==一旦链表成环，后续的 `get()` 操作会陷入无限循环，导致 CPU 占用率飙升。==

**简介：**

​	链表头插法的会颠倒原来一个散列桶里面链表的顺序。

​	在并发的时候原来的顺序被另外一个线程a颠倒了，而被挂起线程b恢复后拿扩容前的节点和顺序继续完成第一次循环后，又遵循a线程扩容后的链表顺序重新排列链表中的顺序，最终形成了环。

### LinkedHashMap

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309142111108.png" alt="image-20250309142111108" style="zoom:60%;" />

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V> {

    // 双链表的头指针
    transient LinkedHashMap.Entry<K,V> head;
    // 双链表的尾指针
    transient LinkedHashMap.Entry<K,V> tail;
    // 迭代排序方法：true 表示访问顺序；false 表示插入顺序
    final boolean accessOrder;
}
```

`LinkedHashMap` 继承了 `HashMap` 的 `put` 方法，本身没有实现 `put` 方法。

**`LinkedHashMap` 通过维护一个保存所有条目（Entry）的双向链表，保证了元素迭代的顺序（即插入顺序）**。

| 关注点                | 结论                           |
| --------------------- | ------------------------------ |
| 是否允许键值对为 null | Key 和 Value 都允许 null       |
| 是否允许重复数据      | Key 重复会覆盖、Value 允许重复 |
| 是否有序              | 按照元素插入顺序存储           |
| 是否线程安全          | 非线程安全                     |

### TreeMap

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309142405976.png" alt="image-20250309142405976" style="zoom:67%;" />

### HashMap、HashTable、TreeMap区别

- **线程是否安全**：HashMap不安全，HashTable内部方法通过synchronized修饰，线程安全（并发环境下推荐使用ConcurrentHashMap）。<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250309142933296.png" alt="image-20250309142933296" style="zoom:80%;" />
- **效率**：因为线程安全问题，HashMap比HashTable效率高
- **null key与null value的区别**：**<font color = '#8D0101'>HashMap中可以有一个key为null，但是value可以有多个null</font>**。HashTable中只要键值有一个为null，则直接抛出nullPointerException。
- **初始容量大小与每次扩容大小**：HashMap中的元素个数超过“数组大小loadFactor”时候，就会扩容，默认是0.75，即16\*0.75 = 12，当HashMap元素达到12就会扩容为16\*2=32.使用resize函数（数组扩容函数）创建扩容后的数组，然后使用transerfer函数将旧数组中的元素迁移到新的数组
- **底层数据结构**：jdk8后的HashMap为解决哈希冲突，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，减少搜索时间。HashTable没有这样的限制。
- **HashMap与TreeMap的区别**：TreeMap和HashMap都继承自AbstractMap，但是TreeMap实现了NavigableMap接口和SortedMap接口。实现NavigableMap接口让TreeMap有对集合内元素搜索的能力，实现SortedMap接口让TreeMap有了对集合中的元素根据键排序的能力，默认是按key的升序排列

### Queue 接口

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/Queue-diagrams.png)

`Queue` 接口定义如下：

```java
public interface Queue<E> extends Collection<E> {}
```

#### AbstractQueue 抽象类

**`AbstractQueue` 类提供 `Queue` 接口的核心实现**，以最大限度地减少实现 `Queue` 接口所需的工作。

`AbstractQueue` 抽象类定义如下：

```java
public abstract class AbstractQueue<E>
    extends AbstractCollection<E>
    implements Queue<E> {}
```

#### Deque 接口

​	Deque 接口是 double ended queue 的缩写，即**双端队列**。Deque 继承 Queue 接口，并扩展支持**在队列的两端插入和删除元素**。

所以提供了特定的方法，如:

- 尾部插入时需要的 [addLast(e) (opens new window)](https://docs.oracle.com/javase/9/docs/api/java/util/Deque.html#addLast-E-)、[offerLast(e) (opens new window)](https://docs.oracle.com/javase/9/docs/api/java/util/Deque.html#offerLast-E-)。
- 尾部删除所需要的 [removeLast() (opens new window)](https://docs.oracle.com/javase/9/docs/api/java/util/Deque.html#removeLast--)、[pollLast() (opens new window)](https://docs.oracle.com/javase/9/docs/api/java/util/Deque.html#pollLast--)。

大多数的实现对元素的数量没有限制，但这个接口既支持有容量限制的 deque，也支持没有固定大小限制的。

#### PriorityQueue

`PriorityQueue` 类定义如下：

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {}
```

`PriorityQueue` 要点：

- `PriorityQueue` 实现了 `Serializable`，支持序列化。
- `PriorityQueue` 类是无界优先级队列。
- `PriorityQueue` 中的元素根据自然顺序或 `Comparator` 提供的顺序排序。
- `PriorityQueue` 不接受 null 值元素。
- `PriorityQueue` 不是线程安全的。



## 并发容器(线程安全的集合)

### 并发容器简介

Java 1.5 后提供了多种并发容器，**使用并发容器来替代同步容器，可以极大地提高伸缩性并降低风险**。

**J.U.C （java.util.concurrent.*）**包中提供了几个非常有用的并发容器作为线程安全的容器：

- **`Concurrent`**
  - 这类型的锁竞争相对于 `CopyOnWrite` 要高一些，但写操作代价要小一些。
  - 此外，`Concurrent` 往往提供了较低的遍历一致性，即：当利用迭代器遍历时，如果容器发生修改，迭代器仍然可以继续进行遍历。代价就是，在获取容器大小 `size()` ，容器是否为空等方法，不一定完全精确，但这是为了获取并发吞吐量的设计取舍，可以理解。与之相比，如果是使用同步容器，就会出现 `fail-fast` 问题，即：检测到容器在遍历过程中发生了修改，则抛出 `ConcurrentModificationException`，不再继续遍历。
- **`CopyOnWrite`** - 一个线程写，多个线程读。读操作时不加锁，写操作时通过在副本上加锁保证并发安全，空间开销较大。
- **`Blocking`** - 内部实现一般是基于锁，提供阻塞队列的能力。

| 并发容器                | 对应的普通容器 | 描述                                                         |
| ----------------------- | -------------- | ------------------------------------------------------------ |
| `ConcurrentHashMap`     | `HashMap`      | Java 1.8 之前采用分段锁机制细化锁粒度，降低阻塞，从而提高并发性；Java 1.8 之后基于 CAS 实现。 |
| `ConcurrentSkipListMap` | `SortedMap`    | 基于跳表实现的                                               |
| `CopyOnWriteArrayList`  | `ArrayList`    |                                                              |
| `CopyOnWriteArraySet`   | `Set`          | 基于 `CopyOnWriteArrayList` 实现。                           |
| `ConcurrentSkipListSet` | `SortedSet`    | 基于 `ConcurrentSkipListMap` 实现。                          |
| `ConcurrentLinkedQueue` | `Queue`        | 线程安全的无界队列。底层采用单链表。支持 FIFO。              |
| `ConcurrentLinkedDeque` | `Deque`        | 线程安全的无界双端队列。底层采用双向链表。支持 FIFO 和 FILO。 |
| `ArrayBlockingQueue`    | `Queue`        | 数组实现的阻塞队列。                                         |
| `LinkedBlockingQueue`   | `Queue`        | 链表实现的阻塞队列。                                         |
| `LinkedBlockingDeque`   | `Deque`        | 双向链表实现的双端阻塞队列                                   |

**<u>并发场景下的 Map</u>**

- 如果对数据有强一致要求，则需使用 `Hashtable`；
- 在大部分场景通常都是弱一致性的情况下，使用 `ConcurrentHashMap` 即可；
- 如果数据量在千万级别，且存在大量增删改操作，则可以考虑使用 `ConcurrentSkipListMap`。

**<u>并发场景下的 List</u>**

- 读多写少用 `CopyOnWriteArrayList`。
- 写多读少用 `ConcurrentLinkedQueue` ，但由于是无界的，要有容量限制，避免无限膨胀，导致内存溢出。

### ConcurrentHashMap

#### ConcurrentHashMap简介和特性

`ConcurrentHashMap` 实现了 `ConcurrentMap` 接口，而 `ConcurrentMap` 接口扩展了 `Map` 接口。

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    // ...
}
```

​	`ConcurrentHashMap` 的实现包含了 `HashMap` 所有的基本特性，如：数据结构、读写策略等。

​	`ConcurrentHashMap` 没有实现对 `Map` 加锁以提供独占访问。因此无法通过在客户端加锁的方式来创建新的原子操作。但是，一些常见的复合操作，如：“若没有则添加”、“若相等则移除”、“若相等则替换”，都已经实现为原子操作，并且是围绕 `ConcurrentMap` 的扩展接口而实现。

```java
public interface ConcurrentMap<K, V> extends Map<K, V> {

    // 仅当 K 没有相应的映射值才插入
    V putIfAbsent(K key, V value);

    // 仅当 K 被映射到 V 时才移除
    boolean remove(Object key, Object value);

    // 仅当 K 被映射到 oldValue 时才替换为 newValue
    boolean replace(K key, V oldValue, V newValue);

    // 仅当 K 被映射到某个值时才替换为 newValue
    V replace(K key, V value);
}
```

不同于 `Hashtable`，`ConcurrentHashMap` 提供的迭代器不会抛出 `ConcurrentModificationException`，因此不需要在迭代过程中对容器加锁。

> [!CAUTION]
>
> 一些需要**对整个 `Map` 进行计算的方法，如 `size` 和 `isEmpty`** ，由于返回的结果在计算时可能已经过期，所以**并非实时的精确值**。这是一种策略上的权衡，在并发环境下，这类方法由于总在不断变化，所以获取其实时精确值的意义不大。`ConcurrentHashMap` 弱化这类方法，以换取更重要操作（如：`get`、`put`、`containesKey`、`remove` 等）的性能。

#### ConcurrentHashMap 的原理（JDK7和JDK8）

> `ConcurrentHashMap` 一直在演进，尤其在 Java 1.7 和 Java 1.8，其数据结构和并发机制有很大的差异。

- Java 1.7
  - 数据结构：**Segment数组＋HashEntry数组+单链表**
  - 并发机制：采用**分段锁机制**细化锁粒度，降低阻塞，从而提高并发性。
- Java 1.8
  - 数据结构：**Node数组＋单链表＋红黑树**
  - 并发机制：取消分段锁，之后基于 **CAS + synchronized** 实现。

<u>**（1）Java 1.7 的实现**</u>

​	分段锁，是将内部进行分段（Segment），里面是 `HashEntry` 数组，和 `HashMap` 类似，哈希相同的条目也是以链表形式存放。 `HashEntry` 内部使用 `volatile` 的 `value` 字段来保证可见性，也利用了不可变对象的机制。

- 可以说ConcurrentHashMap是一个**二级哈希表**，将整张表也就是 一个Segments的数组分成了多个数组（Segment），每个Segment数组本身相当于一个HashMap对象，Segment包含一个hashEntry数组，数组中的每一个hashEntry即为一个键值对，也是一个链表的头结点。
- 集合中共有2的N次方个Segment，共同保存在一个名为Segments的数组中。
- **其中segments不可扩容，而HashEntry可以扩容**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200605214405.png" alt="img" style="zoom:50%;" />

- 在进行并发写操作时，**`ConcurrentHashMap` 会获取可重入锁（`ReentrantLock`），以保证数据一致性**。所以，在并发修改期间，**相应 `Segment` 是被锁定的**，而其他Segment还是可以操作的，这样不同Segment之间就可以实现并发，大大提高效率

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {

    // 将整个hashmap分成几个小的map，每个segment都是一个锁；与hashtable相比，这么设计的目的是对于put, remove等操作，可以减少并发冲突，对
    // 不属于同一个片段的节点可以并发操作，大大提高了性能
    final Segment<K,V>[] segments;

    // 本质上Segment类就是一个小的hashmap，里面table数组存储了各个节点的数据，继承了ReentrantLock, 可以作为互拆锁使用
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        transient volatile HashEntry<K,V>[] table;
        transient int count;
    }

    // 基本节点，存储Key， Value值
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
    }
}
```

**<u>（2）Java 1.8 的实现</u>**

- **数据结构改进**：与 HashMap 一样，将原先 **数组＋单链表** 的数据结构，变更为 **数组＋单链表＋红黑树** 的结构。当出现哈希冲突时，数据会存入数组指定桶的单链表，当链表长度达到 8，则将其转换为红黑树结构，这样其查询的时间复杂度可以降低到 `O(logN)`，以改进性能。

- **并发机制改进**：
  - 取消 segments 分段锁，**直接采用 `transient volatile Node<K,V>[] table` 保存数据，采用 table 数组元素作为锁，从而实现了对每一行数据进行加锁，进一步减少并发冲突的概率**。
  - **使用 CAS + `sychronized` 操作**，在特定场景进行无锁并发操作。现代 JDK 中，synchronized 已经被不断优化，可以不再过分担心性能差异，另外，相比于 ReentrantLock，它可以减少内存消耗，这是个非常大的优势。
  - Java7中使用Entry来代表每个HashMap中的数据节点，Java8中使用Node，基本没有区别，都是key，value，hash和next这四个属性，不过，Node只能用于链表的情况，红黑树的情况需要使用TreeNode。JDK1.8的Node节点中value和next都用**volatile修饰**，保证并发的可见性。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310123315292.png" alt="image-20250310123315292" style="zoom:65%;" />

#### ConcurentHashMap的put过程

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310123505338.png" alt="image-20250310123505338" style="zoom:75%;" />

### CopyOnWriteArrayList(写时复制)

#### CopyOnWriteArrayList简介

​	`CopyOnWriteArrayList` 是线程安全的 ArrayList，CopyOnWriteArrayList采用了**写时复制**的思想，**增删改操作会将底层数组拷贝一份，在新数组上执行操作，不影响其它线程的并发读，<font color = '#8D0101'>读写分离</font>**。修改完成之后，将指向原来容器的引用指向新的容器(副本容器)

- CopyOnWriteArrayList **仅适用于写操作非常少的场景**，而且能够容忍读写的短暂不一致。如果读写比例均衡或者有大量写操作的话，使用 CopyOnWriteArrayList 的性能会非常糟糕。

<u>**写时复制带来的影响**</u>

- 由于不会修改原始容器，只修改副本容器。因此，可以对原始容器进行并发地读。其次，实现了读操作与写操作的分离，**读操作发生在原始容器上，写操作发生在副本容器上**；
- **数据一致性问题**：读操作的线程可能不会立即读取到新修改的数据，因为修改操作发生在副本上。但最终修改操作会完成并更新容器，因此这是最终一致性。**CopyOnWrite容器有数据一致性的问题，它只能保证最终数据一致性。所以如果我们希望写入的数据马上能准确地读取，请不要使用CopyOnWrite容器**

#### CopyOnWriteArrayList 原理

​	CopyOnWriteArrayList 内部维护了一个数组，成员变量 array 就指向这个内部数组，所有的读操作都是基于 array 进行的，如下图所示，迭代器 Iterator 遍历的就是 array 数组。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/20200702204541.png" alt="img" style="zoom:50%;" />

- lock - 执行写时复制操作，需要使用可重入锁加锁
- array - 对象数组，用于存放元素

```java
    /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();

    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

**<u>（1）读操作</u>**

- 在 `CopyOnWriteAarrayList` 中，**读操作不同步**，因为它们在内部数组的快照上工作，所以多个迭代器可以同时遍历而不会相互阻塞
- CopyOnWriteArrayList 的读操作是不用加锁的，性能很高。

```java
public E get(int index) {
    return get(getArray(), index);
}
private E get(Object[] a, int index) {
    return (E) a[index];
}
```

**<u>（2）写操作</u>**

​	所有的写操作都是同步的。他们在数组的副本上工作。添加的逻辑很简单，先将原容器 copy 一份，然后在新副本上执行写操作，之后再切换引用。当然此过程是要加锁的

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250310152607849.png" alt="image-20250310152607849" style="zoom:70%;" />

**<u>（3）删除操作</u>**

**删除操作** - 删除操作同理，将除要删除元素之外的其他元素拷贝到新副本中，然后切换引用，将原容器引用指向新副本。同属写操作，需要加锁。

#### CopyOnWriteArrayList优缺点

<u>**（1）优点**</u>

- CopyOnWriteArrayList经常被**用于“读多写少”的并发场景**，是因为CopyOnWriteArrayList无需任何同步措施，大大增强了读的性能。在Java中遍历线程非安全的List(如：ArrayList和LinkedList)的时候，若中途有别的线程对List容器进行修改，那么会抛出ConcurrentModificationException异常。CopyOnWriteArrayList由于其"读写分离"，遍历和修改操作分别作用在不同的List容器，所以在使用迭代器遍历的时候，则不会抛出异常。

**<u>（2）缺点</u>**

- 第一个缺点是CopyOnWriteArrayList每次执行写操作都会将原容器进行拷贝一份，数据量大的时候，内存会存在较大的压力，可能会引起频繁Full GC（ZGC因为没有使用Full GC）。比如这些对象占用的内存200M左右，那么再写入100M数据进去，内存就会多占用300M。
- 第二个缺点是CopyOnWriteArrayList由于实现的原因，写和读分别作用在不同新老容器上，在写操作执行过程中，读不会阻塞，但读取到的却是老容器的数据

### Queue

Java 并发包里面 Queue 这类并发容器可以从以下两个维度来分类。

- 一个维度是**阻塞与非阻塞**，所谓阻塞指的是：**当队列已满时，入队操作阻塞；当队列已空时，出队操作阻塞**。
- 另一个维度是**单端与双端**，单端指的是只能队尾入队，队首出队；而双端指的是队首队尾皆可入队出队.

​	Java 并发包里**阻塞队列都用 Blocking 关键字标识，单端队列使用 Queue 标识，双端队列使用 Deque 标识**。

#### BlockingQueue(线程池会用到这些队列)

​	`BlockingQueue` 顾名思义，是一个**阻塞队列**。**`BlockingQueue` 基本都是基于锁实现**。在 `BlockingQueue` 中，**当队列已满时，入队操作阻塞；当队列已空时，出队操作阻塞**。

`BlockingQueue` 接口定义如下：

```java
public interface BlockingQueue<E> extends Queue<E> {}
```

核心 API：

```java
// 获取并移除队列头结点，如果必要，其会等待直到队列出现元素
E take() throws InterruptedException;
// 插入元素，如果队列已满，则等待直到队列出现空闲空间
void put(E e) throws InterruptedException;
```

JDK 提供了以下阻塞队列：

- `ArrayBlockingQueue` - 一个由**数组结构组成的有界阻塞队列**。
- `LinkedBlockingQueue` - 一个由**链表结构组成的有界阻塞队列**。
- `PriorityBlockingQueue` - 一个**支持优先级排序的无界阻塞队列**。
- `SynchronousQueue` - 一个**不存储元素的阻塞队列**。
- `DelayQueue` - 一个使用优先级队列实现的无界阻塞队列。
- `LinkedTransferQueue` - 一个**由链表结构组成的无界阻塞队列**。

`BlockingQueue` 基本都是基于锁实现。

#### PriorityBlockingQueue 类

==**支持优先级排序的阻塞队列**==,基于 **二叉堆（Binary Heap） 实现，底层是一个数组**，用于存储元素

`PriorityBlockingQueue` 类定义如下：

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {}
```

**特点：**

- **无界队列**：PriorityBlockingQueue 是无界的，不会因容量限制而抛出异常，大小只受限于 JVM 内存。
- **线程安全**：PriorityBlockingQueue 是线程安全的，支持多线程并发访问。
- **基于优先级排序**：队列中的元素是按照优先级排序的，优先级高的元素会排在前面。默认使用自然排序，**也可以自定义 Comparator。**
- **不保证元素的顺序一致性**：同一优先级的元素的顺序不能保证。

#### ArrayBlockingQueue 类

`ArrayBlockingQueue` :**==基于数组的有界阻塞队列==**。

`ArrayBlockingQueue` 类定义如下：

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    // 数组的大小就决定了队列的边界，所以初始化时必须指定容量
    public ArrayBlockingQueue(int capacity) { //... }
    public ArrayBlockingQueue(int capacity, boolean fair) { //... }
    public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) { //... }
}
```

**<u>ArrayBlockingQueue 原理</u>**

`ArrayBlockingQueue` 的重要成员如下：

```java
// 用于存放元素的数组
final Object[] items;
// 下一次读取操作的位置
int takeIndex;
// 下一次写入操作的位置
int putIndex;
// 队列中的元素数量
int count;
// 以下几个就是控制并发用的同步器
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;
```

`ArrayBlockingQueue` 内部以 `final` 的数组保存数据，**数组的大小就决定了队列的边界**。

`ArrayBlockingQueue` 实现并发同步的原理就是，**读操作和写操作都需要获取到 AQS 独占锁才能进行操作**。

对于 `ArrayBlockingQueue`，我们可以在构造的时候指定以下三个参数：

- **队列容量**，其限制了队列中最多允许的元素个数；
- **指定独占锁是公平锁还是非公平锁**。非公平锁的吞吐量比较高，公平锁可以保证每次都是等待最久的线程获取到锁；
- **可以指定用一个集合来初始化，将此集合中的元素在构造方法期间就先添加到队列中**

#### LinkedBlockingQueue 类

​	`LinkedBlockingQueue` 是==**由链表结构组成的有界阻塞队列**==。

​	容易被误解为无边界，但其实其行为和内部代码都是基于有界的逻辑实现的，只不过如果我们没有在创建队列时就指定容量，那么其容量限制就自动被设置为 `Integer.MAX_VALUE`，成为了无界队列.

`LinkedBlockingQueue` 类定义如下：

- `LinkedBlockingQueue` 实现了 `BlockingQueue`，也是一个阻塞队列。
- `LinkedBlockingQueue` 实现了 `Serializable`，支持序列化。
- `LinkedBlockingQueue` 是基于单链表实现的阻塞队列，可以当做无界队列也可以当做有界队列来使用。
- `LinkedBlockingQueue` 中元素按照插入顺序保存（FIFO）。

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {}
```

**<u>LinkedBlockingQueue 原理</u>**

`LinkedBlockingQueue` 中的重要数据结构：

```java
// 队列容量
private final int capacity;
// 队列中的元素数量
private final AtomicInteger count = new AtomicInteger(0);
// 队头
private transient Node<E> head;
// 队尾
private transient Node<E> last;

// take, poll, peek 等读操作的方法需要获取到这个锁
private final ReentrantLock takeLock = new ReentrantLock();
// 如果读操作的时候队列是空的，那么等待 notEmpty 条件
private final Condition notEmpty = takeLock.newCondition();
// put, offer 等写操作的方法需要获取到这个锁
private final ReentrantLock putLock = new ReentrantLock();
// 如果写操作的时候队列是满的，那么等待 notFull 条件
private final Condition notFull = putLock.newCondition();
```

这里用了两对 `Lock` 和 `Condition`，简单介绍如下：

- `takeLock` 和 `notEmpty` 搭配：如果要获取（take）一个元素，需要获取 `takeLock` 锁，但是获取了锁还不够，如果队列此时为空，还需要队列不为空（`notEmpty`）这个条件（`Condition`）。
- `putLock` 需要和 `notFull` 搭配：如果要插入（put）一个元素，需要获取 `putLock` 锁，但是获取了锁还不够，如果队列此时已满，还需要队列不是满的（notFull）这个条件（`Condition`）。

#### SynchronousQueue 类

​	SynchronousQueue 是**==不存储元素的阻塞队列==**。

​	每个删除操作都要等待插入操作，反之每个插入操作也都要等待删除动作。那么这个队列的容量是多少呢？是 1 吗？其实不是的，其内部容量是 0。

`SynchronousQueue` 定义如下：

```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {}
```

`SynchronousQueue` 这个类，在线程池的实现类 `ScheduledThreadPoolExecutor` 中得到了应用。

`SynchronousQueue` 的队列其实是虚的，即队列容量为 0。数据必须从某个写线程交给某个读线程，而不是写到某个队列中等待被消费。

- **所以说这是一个很有意思的阻塞队列，其中每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都等待另一个线程的插入操作。因此此队列内部其实没有任何一个元素，因此不能调用peek操作，因为只有移除元素时才有元素。**
  - `SynchronousQueue` 中不能使用 peek 方法（在这里这个方法直接返回 null），peek 方法的语义是只读取不移除，显然，这个方法的语义是不符合 SynchronousQueue 的特征
  - `SynchronousQueue` 也不能被迭代，因为根本就没有元素可以拿来迭代的。
  - 虽然 `SynchronousQueue` 间接地实现了 Collection 接口，但是如果你将其当做 Collection 来用的话，那么集合是空的。

# 





