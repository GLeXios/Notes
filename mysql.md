

# MySQL数据库基础

## 相关概念

### 驱动表和被驱动表

跟表的连接有关

驱动表在SQL语句执行的过程中，总是先读取。而被驱动表在SQL语句执行的过程中，总是后读取。

1.当使用left join时，左表是驱动表，右表是被驱动表

2.当使用right join时，右表时驱动表，左表是驱动表

3.当使用inner join时，mysql会选择数据量比较小的表做为驱动表，大表做为被驱动表

### 池化设计思想、数据库连接池

**【池化】**	

​	**池化（Pooling）** 是一种通过**预先创建并复用资源** 来减少资源创建/销毁开销的设计模式，核心目标是**提高性能** 和**控制资源消耗** 。

​	这种设计会初始化预设资源，解决的问题就是抵消每次获取资源的消耗，如创建线程的开销，获取远程连接的开销等。除了初始化资源，池化设计还包括如下这些特征：池子的初始值、池子的活跃值、池子的最大值等，这些特征可以直接映射到java线程池和数据库连接池的成员属性中。

**典型应用场景**

- **线程池** ：复用线程，减少线程创建开销（如 `ExecutorService`）。
- **数据库连接池** ：复用数据库连接，避免频繁建立/断开 TCP 连接。
- **对象池** ：复用大对象（如数据库连接、网络套接字）。
- **内存池** ：预分配内存块，减少碎片化（如 Redis 的内存管理）。

​	数据库连接本质就是一个Socket的连接。数据库服务端还要维护一些缓存和用户权限信息之类的所以占用了一些内存。我们可以把数据库连接池是看做是维护数据库连接的缓存，以便将来需要对数据库的请求时可以重用这些连接，为每个用户打开和维护数据库连接，尤其是对动态数据库驱动的网站应用程序的请求，既昂贵又浪费资源。在连接池中，创建连接后，将其放置在池中，并再次使用它，因此不必建立新的连接，如果使用了所有连接，则会建立一个新连接并将其添加到池中。连接池还减少了用户必须等待建立与数据库的连接的时间

**【数据库连接池】**

​	数据库连接池是池化思想在数据库访问中的具体实现，**解决频繁创建/关闭连接导致的性能问题** 。

**主流数据库连接池对比**

| **连接池**      | **特点**                                                     | **适用场景**             |
| --------------- | ------------------------------------------------------------ | ------------------------ |
| **HikariCP**    | 轻量级、高性能（号称最快），通过优化代码和并发结构实现低延迟。 | 高并发、低延迟的微服务   |
| **Druid**       | 阿里开源，提供监控、SQL 防注入、日志分析等高级功能。         | 需要监控和安全防护的系统 |
| **C3P0**        | 早期流行，配置复杂，性能较低。                               | 传统遗留系统             |
| **Tomcat JDBC** | 与 Tomcat 集成度高，性能中等。                               | Tomcat 容器内应用        |

### 关系性和非关系性数据库

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316151700141.png" alt="image-20250316151700141" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316151708561.png" alt="image-20250316151708561" style="zoom:80%;" />

键值型数据库典型的使用场景是作为内存缓存。**Redis是最流行的键值型数据库**

### SQL语句分类及规范

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316151932977.png" alt="image-20250316151932977" style="zoom:80%;" />

**【分类】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316151956247.png" alt="image-20250316151956247" style="zoom:80%;" />

**【SQL语言的规则与规范】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316152246408.png" alt="image-20250316152246408" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316152216723.png" alt="image-20250316152216723" style="zoom:80%;" />

**【数据导入指令】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316152402732.png" alt="image-20250316152402732" style="zoom:80%;" />

## DML

### SELECT的执行过程

#### SELECT完整结构

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151944599.png" alt="image-20250317151944599" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317152004364.png" alt="image-20250317152004364" style="zoom:80%;" />

#### SELECT执行顺序

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317152024532.png" alt="image-20250317152024532" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317152120873.png" alt="image-20250317152120873" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317152130221.png" alt="image-20250317152130221" style="zoom:80%;" />

**【eg】**

```sql
SELECT DISTINCT player id, player name, count（*）as num # 顺序 5
FROM player JOIN team ON player. team id = team. team id # 顺序 1
WHERE height > 1.80# 顺序 2
GROUP BY player. team id # 顺序 3
HAVING num > 2# 顺序 4
ORDER BY num DESC # 顺序 6
LIMIT 2# 顺序 7
```



### 基本SELECT语句(SELECT,AS,DISTINCT,WHERE,FROM)

- **着重号**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316154220517.png" alt="image-20250316154220517" style="zoom:80%;" />

- **AS关键字，别名**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316153258826.png" alt="image-20250316153258826" style="zoom:80%;" />

- **去除重复行：DISTINCT关键字**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316153406827.png" alt="image-20250316153406827" style="zoom:80%;" />

- **空值参与运算**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316153426279.png" alt="image-20250316153426279" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316153434746.png" alt="image-20250316153434746" style="zoom:80%;" />

- **过滤数据：WHERE关键字**，跟在FROM后面

![image-20250316153610375](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316153610375.png)

### 排序ORDER BY、分页LIMIT

**【ORDER BY】**

- **排序规则：使用ORDER BY对查询到的数据进行排序操作。升序（ASC，ascend）：默认按照升序排列；降序（DESC，descend）**

​	可以使用不在SELECT列表中的列排序。在对多列进行排序的时候，首先排序的第一列必须有相同的列值，才会对第二列进行排序，如果第一列数据中所有值都是唯一的，将不再对第二列进行排序

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316154022290.png" alt="image-20250316154022290" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316154247962.png" alt="image-20250316154247962" style="zoom:67%;" />

**【LIMIT】**

==LIMIT子句必须放在整个SELECT语句的最后==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250316154349567.png" alt="image-20250316154349567" style="zoom:80%;" />

所谓分页显示，就是将数据库中的结果集，一段一段显示出来需要的条件。

- MySQL中使用**LIMIT**实现分页。格式：LIMIT 位置偏移量,行数

```sql
LIMIT [位置偏移量],行数
```

​	第一个“位置偏移量”参数指示MySQL从哪一行开始显示，是一个可选参数，如果不指定“位置偏移量”，将会从表中的第一条记录开始（**第一条记录的位置偏移量是0，第二条记录的位置偏移量是1，以此类推**）；第二个参数“行数”指示返回的记录条数

- **每页显示pageSize条记录，此时显示第pageNo页：<font color = '#8D0101'>公式：`LIMIT (pageNo-1) * pageSize，pageSize`</font>**；

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317000259952.png" alt="image-20250317000259952" style="zoom:67%;" />

> [!IMPORTANT]
>
> 约束返回结果的数量可以**减少数据表的网络传输量，也可以提升查询效率**。如果我们知道返回结果只有1条，就可以使用LIMIT 1，告诉SELECT语句只需要返回一条记录即可。这样的好处就是SELECT不需要扫描完整的表，只需要检索到一条符合条件的记录即可返回。

### 多表查询/连接查询/JOIN

- **WHERE**：适用于所有的关联查询；
- **ON**：只能和JOIN一起使用，只能写关联条件，虽然关联条件可以并到WHERE中和其他条件一起写，但分开写可读性更好；
- **USING**：只能和JOIN一起使用，而且要求两个关联字段在关联表中名称一致，而且只能表示关联字段值相等。

#### 驱动表和被驱动表

**跟表的连接有关**

驱动表在SQL语句执行的过程中，总是先读取。而被驱动表在SQL语句执行的过程中，总是后读取。

1.当使用left join时，左表是驱动表，右表是被驱动表

2.当使用right join时，右表时驱动表，左表是驱动表

3.当使用inner join时，mysql会选择数据量比较小的表做为驱动表，大表做为被驱动表

#### 等值连接 非等值连接 BETWEEN...AND

**【等值连接】**

- **多个表中有相同列时，必须在列名之前加上表名前缀**。
- 在不同表中具有相同列名的列可以用表名加以区分。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317002413475.png" alt="image-20250317002413475" style="zoom:67%;" />

**【非等值连接】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317002518095.png" alt="image-20250317002518095" style="zoom:67%;" />

#### 内连接JOIN 外连接LEFT JOIN...

**<u>（1）内连接</u>**

- **其实上面的都是内连接**：合并具有同一列的两个以上的表的行，<font color = '#8D0101'>结果集中不包含一个表与另一个表不匹配的行</font>

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317002413475.png" alt="image-20250317002413475" style="zoom:67%;" />

- **INNER可以省略**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003148372.png" alt="image-20250317003148372" style="zoom:67%;" />

**<u>（2）外连接</u>**

- **外连接**：合并具有同一列的两个以上的表的行,<font color = '#8D0101'>结果集中除了包含一个表与另一个表匹配的行之外，还查询到了左表或右表中不匹配的行</font>

- **外连接的分类**：左外连接、右外连接

【**左外连接**】

- 两个表在连接过程中**除了返回满足连接条件的行以外还返回左表中不满足条件的行，这种连接称为左外连接**。如果是左外连接，则**连接条件中左边的表也称为主表，右边的表称为从表**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003421634.png" alt="image-20250317003421634" style="zoom:67%;" />

**【右外连接】**

- 两个表在连接过程中除了返回满足连接条件的行以外还返回右表中不满足条件的行，这种连接称为右外连接。如果是右外连接，则连接条件中右边的表也称为主表，左边的表称为从表

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003524916.png" alt="image-20250317003524916" style="zoom:67%;" />

#### 7种SQL JOINS的实现

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003927293.png" alt="image-20250317003927293" style="zoom:80%;" /> 

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003946436.png" alt="image-20250317003946436" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317003959212.png" alt="image-20250317003959212" style="zoom:67%;" />

#### UNION合并查询

​	合并查询结果利用UNION关键字，可以给出多条SELECT语句，并将它们的结果组合成单个结果集。**合并时，两个表对应的列数和数据类型必须相同**，并且相互对应。各个SELECT语句之间使用UNION或UNIONALL关键字分隔。

- **UNION操作符返回两个查询的结果集的并集，去除重复记录。**
- **UNION ALL操作符返回两个查询的结果集的并集，对于两个结果集的重复部分，不去重。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317004135028.png" alt="image-20250317004135028" style="zoom:80%;" />

### 单行函数

#### 流程控制函数CASE、IF

​	流程处理函数可以根据不同的条件，执行不同的处理流程，可以在SQL语句中实现不同的条件选择。MySQL中的流程处理函数主要包括**IF()、IFNULL()和CASE()函数**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317145921600.png" alt="image-20250317145921600" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317145931757.png" alt="image-20250317145931757" style="zoom:67%;" />

#### 日期和时间函数

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150129768.png" alt="image-20250317150129768" style="zoom:67%;" />





#### 数值函数

**【基本函数】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317145459107.png" alt="image-20250317145459107" style="zoom:50%;" />

#### 字符串函数

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317145753099.png" alt="image-20250317145753099" style="zoom:50%;" />

### 聚合函数：AVG(),SUM(),MAX(),MIN(),COUNT()

- **AVG/SUM**：只适用于数值类型的字段（或变量）。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150244914.png" alt="image-20250317150244914" style="zoom:67%;" />

- **MAX/MIN**：适用于数值类型、字符串类型、日期时间类型的字段（或变量）。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150304470.png" alt="image-20250317150304470" style="zoom:67%;" />

- **COUNT**：计算指定字段在查询结构中出现的个数（不包含NULL值的）。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150328118.png" alt="image-20250317150328118" style="zoom:67%;" />

### GROUP BY分组、HAVING过滤数据

#### GROUP BY分组

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150535089.png" alt="image-20250317150535089" style="zoom:67%;" />

**【eg1】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150623648.png" alt="image-20250317150623648" style="zoom:67%;" />

**【eg2】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150636973.png" alt="image-20250317150636973" style="zoom:67%;" />

==注意==

- **<font color = '#8D0101'>SELECT中出现的未包含在组函数的字段必须声明在GROUP BY中</font>**，反之，GROUP BY中声明的字段可以不出现在SELECT中。 
- **GROUP BY声明在FROM后面、WHERE后面，ORDER BY前面、LIMIT前面。**

**【错误eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317150824460.png" alt="image-20250317150824460" style="zoom:67%;" />

#### HAVING过滤数据

- **如果过滤条件中使用了聚合函数，则必须使用HAVING来替换WHERE，否则，报错**

**【eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151313942.png" alt="image-20250317151313942" style="zoom:67%;" />

- **HAVING必须声明在GROUP BY的后面,HAVING不能单独使用**
- 当过滤条件中**没有聚合函数时**，则此过滤条件声明在WHERE中或HAVING中都可以。但是，**建议大家将过滤条件声明在WHERE中，这样效率更高**

**【eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151423279.png" alt="image-20250317151423279" style="zoom:67%;" />

#### WHERE与HAVING的对比

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151738900.png" alt="image-20250317151738900" style="zoom:80%;" />

- **WHERE可以直接使用表中的字段作为筛选条件，但不能使用分组中的计算函数作为筛选条件；HAVING必须要与GROUP BY配合使用，可以把分组计算的函数和分组字段作为筛选条件**。从适用范围上来讲，HAVING的适用范围更广

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151559283.png" alt="image-20250317151559283" style="zoom:80%;" />

- **如果需要通过连接从关联表中获取需要的数据**，**<font color = '#8D0101'>WHERE是先筛选后连接，而HAVING是先连接后筛选</font>**。如果过滤条件中没有聚合函数：这种情况下，WHERE的执行效率要高于HAVING

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317151714176.png" alt="image-20250317151714176" style="zoom:80%;" />

- **开发选择**：WHERE和HAVING也不是互相排斥的，我们可以在一个查询里面同时使用WHERE和HAVING。

  - **包含分组统计函数的条件用HAVING，普通条件用WHERE。**
  - 这样，我们就既利用了WHERE条件的高效快速，又发挥了HAVING可以使用包含分组统计函数的查询条件的优点，当数据量特别大的时候，运行效率会有很大的差别

### 子查询

==在SELECT中，除了GROUP BY和LIMIT之外，其他位置都可以声明子查询==

#### 单行子查询

**【单行比较操作符】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317154909107.png" alt="image-20250317154909107" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317154917335.png" alt="image-20250317154917335" style="zoom:67%;" />

**【HAVING中的子查询】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317155004674.png" alt="image-20250317155004674" style="zoom:67%;" />

**【CASE中的子查询】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317155026965.png" alt="image-20250317155026965" style="zoom:67%;" />

#### 多行子查询

**多行子查询**：1、也称为集合比较子查询；2、内查询返回多行；3、使用多行比较操作符

【**多行比较操作符**】

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317155803701.png" alt="image-20250317155803701" style="zoom:67%;" />

**【eg1】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317155852638.png" alt="image-20250317155852638" style="zoom:67%;" />

**【eg2】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317160752249.png" alt="image-20250317160752249" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317160829158.png" alt="image-20250317160829158" style="zoom:45%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317160848576.png" alt="image-20250317160848576" style="zoom:67%;" />



#### 相关子查询/关联子查询

​	**相关子查询**：如果**子查询的执行依赖于外部查询**，通常情况下都是因为子查询中的表用到了外部的表，并进行了条件关联，**因此每执行一次外部查询，子查询都要重新计算一次，这样的子查询就称之为关联子查询**。相关子查询按照一行接一行的顺序执行，主查询的每一行都执行一次子查询。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317161359363.png" alt="image-20250317161359363" style="zoom:67%;" />

```sql
#案例：查询员工中工资大于本部门平均工资的员工的last name, salary和其department id
#方式一：相关子查询
SELECT last_name, salary, department_id
FROM employees e #outer table
WHERE salary > (
	SELECT AVG(salary)
	FROM departments #inner table
	WHERE department_id = e. department_id)
```

```sql
#方式2：在FROM中使用子查询
SELECT last name, salary,e1. department id 
FROM employees e1,(
	SELECT department id,AVG(salary) dept avg sal
	FROM employees
	GROUP BY department id) e2#把子查询的结果作为一张虚拟的新表
WHERE e1. department id = e2. department id
AND e2. dept avg sal < e1. salary;
```



### 数据CRUD

#### 插入数据INSERT

##### VALUES的方式一条一条添加

- 为表的所有字段按默认顺序插入数据

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162634621.png" alt="image-20250317162634621" style="zoom:67%;" />

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162643413.png" alt="image-20250317162643413" style="zoom:67%;" />

- **为表的指定字段插入数据**      

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162722938.png" alt="image-20250317162722938" style="zoom:67%;" />

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162731819.png" alt="image-20250317162731819" style="zoom:67%;" />

- **同时插入多条记录**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162754850.png" alt="image-20250317162754850" style="zoom:67%;" />

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162817715.png" alt="image-20250317162817715" style="zoom:67%;" />

  - **【eg】**

    <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317162859799.png" alt="image-20250317162859799" style="zoom:67%;" />

##### 将查询结果插入到表中

​	INSERT还可以将SELECT语句查询的结果插入到表中，此时不需要把每一条记录的值一个一个输入，只需要使用一条INSERT语句和一条SELECT语句组成的组合语句即可快速地从一个或多个表中向一个表中插入多行。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317163048980.png" alt="image-20250317163048980" style="zoom:80%;" />

- 在INSERT语句中加入子查询。
- **不必书写VALUES子句。**
- 子查询中的值列表应与INSERT子句中的列名对应。

**【eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317163156732.png" alt="image-20250317163156732" style="zoom:80%;" />

#### 更新数据UPDATE

使用UPDATE语句更新数据。语法如下：

- 可以**一次更新多条数据**。
- 如果需要回滚数据，需要保证在DML前，**进行设置：`SETAUTOCOMMIT=FALSE;`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317163339118.png" alt="image-20250317163339118" style="zoom:80%;" />

- **使用WHERE子句指定需要更新的数据。**

```sql
UPDATE employees
SET	department id =70
WHERE	employee id =113;
```

- 如果省略WHERE子句，则表中的所有数据都将被更新。

```sql
UPDATE	copy emp
SET	department id =110;
```

#### 删除数据DELETE

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164418939.png" alt="image-20250317164418939" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164426175.png" alt="image-20250317164426175" style="zoom:80%;" />

### DML相关面试问题

#### delete、drop和truncate区别

- **drop**：删除内容和定义、释放空间。简单来说就是**把整个表去掉**，包括表的所有结构，以后要是新增数据除非新增一个表
- **truncate**：删除内容、释放空间，但是不删除定义。与drop不同的是，它只是清空表数据而已。（**truncate不能删除行数据，要删就把表清空**）。
- **delete**：虽然也是删除整个表的数据，但是是一行一行的删除，效率比truncate低。并在事务日志中为所删除的每行记录一项。所以可对delete进行roll back

​	**<font color = '#8D0101'>truncate和drop属于DDL(数据定义语言)语句，操作立即生效，原数据不放到rollback segment中，不能回滚，操作不触发trigger</font>。**

​	**<font color = '#8D0101'>而delete语句是DML(数据库操作语言)语句，这个操作会放到rollback segement中，事务提交之后才生效，能够回滚</font>**

**效率：drop>truncate>delete**

#### Count、limit、distinct

**【Count函数】**

**1） COUNT(\*)**:统计所有的行数，包括为null的行,不单会进行全表扫描，也会对表的每个字段进行扫描。而COUNT('x')或者COUNT(COLUMN)或者COUNT(0)等则只进行一个字段的全表扫描

**2） COUNT(1):**计算一共有多少符合条件的行，不会忽略null值（其实就可以想成表中有这么一个字段,这个字段就是固定值1,count(1),就是计算一共有多少个1..同理,count(2),也可以,得到的值完全一样,count('x'),count('y')都是可以的。count(),执行时会把星号翻译成字段的具体名字,效果也是一样的,不过多了一个翻译的动作,比固定值的方式效率稍微低一些。）

**按照效率排序的话，count(字段)<count(主键 id)<count(1)≈count()**，所以建议尽量使用count()，因为这个是SQL92定义的标准统计行数的语法

**【limit】**

- 分页。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps23.jpg) 

**【Distinct】**

- select distinct去除重复的记录。

#### varchar(n)、char(n)、int(n)的区别

**1）varchar与char：**

- **varchar与char的区别，char是一种固定长度的类型，varchar则是一种可变长度的类型**,varchar还要占个byte用于存储信息长度
- varchar(30)中30的涵义最多存放30个字符。varchar(30)和(130)存储hello所占空间一样，但后者在排序时会消耗更多内存，因为ORDER BY col采用fixed_length计算col长度( memory引擎也一样)。
- 对效率要求高用char，对空间使用要求高用varchar。

**2）int：**

- int(11)中的11，不影响字段存储的范围，只影响展示效果
- 表示显示宽度，M的取值范围是(0, 255)。例如，int(5):当数据宽度小于5位的时候在数字前面需要用字符填满宽度。该项功能需要配合”ZEROFILL"使用，表示用"0"填满宽度，否则指定显示宽度无效。
- **从MySQL 8.0.17开始，整数数据类型不推荐使用显示宽度属性。**

**3）char**： 

如果char初始设定为5个字符，然后只存了3个字符，那么剩下的字符会进行**补空格操作。**

#### 哪些情况使用CHAR，哪些情况使用VARCHAR

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps24.jpg) 

1）存储很短的信息。比如门牌号码101，20.1......这样很短的信息应该用char，因为varchar还要占个byte用于存储信息长度，本来打算节约存储的，结果得不偿失。

2）十分频繁改变的column。因为varchar每次存储都要有额外的计算，得到长度等工作，如果column改变得非常频繁，那就要有很多的精力用于计算，而这些对于char来说是不需要的。

3）InnoDB存储引擎，建议使用VARCHAR类型。因为对于InnoDB数据表，内部的行存储格式并没有区分固定长度和可变长度列（所有数据行都使用指向数据列值的头指针)，而且主要影响性能的因素是数据行使用的存储总量，由于char平均占用的空间多于varchar，所以除了简短并且固定长度的，其他考虑varchar。这样节省空间，对磁盘I/O和数据存储总量比较好

#### 浮点精度损失，定点数DECIMAL

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319140100748.png" alt="image-20250319140100748" style="zoom:80%;" />

- 浮点数相对于定点数的优点是在长度一定的情况下，浮点类型取值范围大，但是不精准，适用于需要取值范围大，又可以容忍微小误差的科学计算场景（比如计算化学、分子建模、流体动力学等)
- 定点数类型取值范围相对小，但是精准，没有误差，适合于对精度要求极高的场景（比如涉及金额计算的场景)















## DDL

### 数据库操作

#### 创建数据库CREATE

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164543738.png" alt="image-20250317164543738" style="zoom:80%;" />

#### 使用数据库

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164705133.png" alt="image-20250317164705133" style="zoom:80%;" />

#### 修改数据库ALTER

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164853584.png" alt="image-20250317164853584" style="zoom:80%;" />

#### 删除数据库DROP

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317164908233.png" alt="image-20250317164908233" style="zoom:80%;" />

### 表操作

#### 创建表CREATE

<u>**1）方式1：白手起家**</u>

- **加上了IF NOT EXISTS关键字**，则表示：如果当前数据库中不存在要创建的数据表，则创建数据表；如果当前数据库中已经存在要创建的数据表，则忽略建表语句，不再创建数据表。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317165318893.png" alt="image-20250317165318893" style="zoom:80%;" />

**【eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317165456528.png" alt="image-20250317165456528" style="zoom:80%;" />

**【eg】**

```sql
CREATE TABLE students (
    -- 主键（自增）
    student_id INT AUTO_INCREMENT PRIMARY KEY,
    
    -- 基本信息
    student_no VARCHAR(20) NOT NULL UNIQUE COMMENT '学号，唯一标识',
    name VARCHAR(50) NOT NULL COMMENT '学生姓名',
    gender CHAR(1) NOT NULL CHECK (gender IN ('M', 'F')) COMMENT '性别：M-男，F-女',
    birth_date DATE NOT NULL COMMENT '出生日期',
    email VARCHAR(100) UNIQUE COMMENT '邮箱，唯一',
    phone VARCHAR(20) COMMENT '联系电话',
    
    -- 时间戳
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '最后更新时间',
    
    -- 检查约束（年龄必须 ≥ 15 岁）
    CONSTRAINT chk_age CHECK (TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) >= 15)
);
```

**<u>2）方式2：基于现有的表创建新表</u>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317165909744.png" alt="image-20250317165909744" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317165915001.png" alt="image-20250317165915001" style="zoom:80%;" />

#### 修改表ALTER

修改表指的是修改数据库中已经存在的数据表的结构。使用ALTERTABLE语句可以实现：

- 向已有的表中添加列
- 修改现有表中的列
- 删除现有表中的列
- 重命名现有表中的列

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317170255761.png" alt="image-20250317170255761" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317170329224.png" alt="image-20250317170329224" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317170425225.png" alt="image-20250317170425225" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317170431941.png" alt="image-20250317170431941" style="zoom:67%;" />

#### 重命名表RENAME

RENAME关键字

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps1.jpg" alt="img" style="zoom:80%;" />

#### 删除表DROP

==DROP TABLE语句不能回滚==

<img src="Users/glexios/Library/Containers/com.kingsoft.wpsoffice.mac/Data/tmp/wps-glexios/ksohtml//wps2.jpg" alt="img" style="zoom:80%;" /> 

IF EXISTS的含义为：如果当前数据库中存在相应的数据表，则删除数据表；如果当前数据库中不存在相应的数据表，则忽略删除语句，不再执行删除数据表的操作。

## 约束

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317170940683.png" alt="image-20250317170940683" style="zoom:80%;" />

==**如何添加删除约束**==

​	约束是表级的强制规定。**可以在创建表时规定约束（通过CREATE TABLE语句），或者在表创建之后通过ALTER TABLE语句规定约束。**

### 约束的分类

**1）根据约束数据列的限制**

- 单列约束：每个约束只约束一列
- 多列约束：每个约束可约束多列数据；

**2）根据约束的作用范围：**

- 列级约束：只能作用在一个列上，声明在对应字段的后面；
- 表级约束：可以作用在多个列上，不与列一起，而是单独定义（表中所有字段都是声明完后，在所有字段的后面声明的约束）。

**3）约束的作用(或功能)**：

​	NOT NULL非空约束，UNIQUE唯一约束、PRIMARY KEY主键（非空且唯一）、FOREIGN KEY外键约束、CHECK检查约束、DEFAULT默认值约束。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171129424.png" alt="image-20250317171129424" style="zoom:80%;" />

### 非空约束NOT NULL

**特点**：

- 默认，所有的类型的值都可以是NULL，包括INT、FLOAT等数据类型；
- 非空约束只能出现在表对象的列上，只能某个列单独限定非空，不能组合非空；
- 一个表可以有很多列都分别限定了非空；
- **<font color = '#8D0101'>“空字符串''（''）不等于NULL，0也不等于NULL</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171619447.png" alt="image-20250317171619447" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171633561.png" alt="image-20250317171633561" style="zoom:80%;" />

### 唯一性约束UNIQUE

用来限制某个字段/某列的值不能重复

- 同一个表可以有多个唯一约束；
- 唯一约束可以是某一个列的值唯一，也**可以多个列组合的值唯一；**
- **唯一性约束允许列值为空；**
- 在创建唯一约束的时候，如果不**给唯一约束命名，就默认和列名相同；**
- **MySQL会给唯一约束的列上默认创建一个唯一索引**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171800655.png" alt="image-20250317171800655" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171853087.png" alt="image-20250317171853087" style="zoom:80%;" />

- 建表后

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171915482.png" alt="image-20250317171915482" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317171956281.png" alt="image-20250317171956281" style="zoom:80%;" />



**【复合唯一约束】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps3.jpg" alt="img" style="zoom:80%;" /> 

Eg：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps4.jpg" alt="img" style="zoom:80%;" /> 

**<font color = '#8D0101'>即不能同时相同，or的逻辑</font>**

**【删除唯一约束】**

1）**添加唯一性约束的列上也会自动创建唯一索引**；

2）**删除唯一约束只能通过删除唯一索引的方式删除**；

3）删除时需要指定唯一索引名，唯一索引名就和唯一约束名一样；

4）如果创建唯一约束时未指定名称，如果是单列，就默认和列名相同；如果是组合列，那么默认和()中排在第一个的列名相同，也可以自定义唯一性约束名。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps5.jpg" alt="img" style="zoom:80%;" /> 



### 主键约束PIMARY KEY

用来唯一标识表中的一行记录

- **<font color = '#8D0101'>主键约束相当于唯一约束+非空约束的组合</font>，主键约束列不允许重复，也不允许出现空值**；
- 一个表最多只能有一个主键约束，建立主键约束可以在列级别创建，也可以在表级别上创建；
- 如果是多列组合的复合主键约束，那么这些列都不允许为空值，并且组合的值不允许重复；
- 当创建主键约束时，**系统默认会在所在的列或列组合上建立对应的主键索引（能够根据主键查询的，就根据主键查询，效率更高）**，如果删除主键约束了，主键约束对应的索引就自动删除了

### 自增列：AUTO_INCREMENT

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317174801153.png" alt="image-20250317174801153" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250317175000681.png" alt="image-20250317175000681" style="zoom:67%;" />

### 为什么建表时，加not null default ''或default 0，为什么不想要null的值？

不想让表中出现null值。

- **不好比较。null是一种特殊值，比较时只能用专门的is null和is not null来比较。碰到运算符，通常返回null。**
- **效率不高。影响提高索引效果。因此，往往在建表时使用not null default ''或default 0。**

### 带AUTO_INCREMENT约束的字段值是从1开始的吗？

​	在MySQL中，**默认AUTO_INCREMENT的初始值是1**，每新增一条记录，字段值自动加1。设置自增属性（AUTO_INCREMENT）的时候，还**可以指定第一条插入记录的自增字段的值**，这样新插入的记录的自增字段值从初始值开始递增，如在表中插入第一条记录，同时指定id值为5，则以后插入的记录的id值就会从6开始往上增加。**添加主键约束时，往往需要设置字段自动增加属性**。

## 视图

视图的本质，就可以看做是**<font color = '#8D0101'>存储起来的SELECT语句</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318133435697.png" alt="image-20250318133435697" style="zoom:80%;" />

### 创建视图

【**单表视图】**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps6.jpg)

Eg：

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps7.jpg)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps8.jpg" alt="img" style="zoom:67%;" /> 

**【多表视图】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318134507897.png" alt="image-20250318134507897" style="zoom:70%;" />

### 查看视图

 <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318134748532.png" alt="image-20250318134748532" style="zoom:73%;" />

### 总结

​	视图一般只做查询不做增删改的操作。视图的目的通常是从优点考虑的，一个是安全（敏感字段），第二是高效，第三是可以定制化数据（如多表查询）

- 视图，可以看做是一个虚拟表，本身是不存储数据的。视图的本质，就可以看做是存储起来的SELECT语句
- 视图中SELECT语句中涉及到的表，称为基表
- **针对视图做DML操作，会影响到对应的基表中的数据。反之亦然。**
- **视图本身的删除，不会导致基表中数据的删除。



# MySQL原理

## MySQL总体架构与执行流程

### 执行流程

==执行一条 SQL 查询语句，期间发生了什么？==

- **连接器**：建立连接，管理连接、校验用户身份；
- **查询缓存**：查询语句如果命中查询缓存则直接返回，否则继续往下执行。MySQL 8.0已删除该模块；
- **解析SQL**：通过解析器对SQL查询语句**进行词法分析、语法分析，然后构建语法树**，方便后续模块读取表名、字段、语句类型；
- **执行SQL**：执行SQL共有三个阶段：
  - **预处理阶段**：检查表或字段是否存在；将select *中的 *符号扩展为表上的所有列。
  - **优化阶段**：基于查询成本的考虑，  选择查询成本最小的执行计划；
  - **执行阶段**：根据执行计划执行SQL查询语句，从存储引擎读取记录，返回给客户端；

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/mysql查询流程.png" alt="查询语句执行流程" style="zoom:73%;" />

可以看到，  MySQL的架构共分为两层：**Server层和存储引擎层，**

- **Server层负责建立连接、分析和执行SQL**。MySQL大多数的**核心功能模块**都在这实现，主要**包括连接器，查询缓存、解析器、预处理器、优化器、执行器等**。另外，所有的内置函数（如日期、时间、数学和加密函数等）和所有跨存储引擎的功能（如存储过程、触发器、视图等。）都在Server层实现。
- **存储引擎层负责数据的存储和提取**。支持InnoDB、MyISAM、Memory等多个存储引擎，不同的存储引擎共用一个Server层。现在最常用的存储引擎是InnoDB, 从MySQL 5.5版本开始, InnoDB成为了MySQL的默认存储引擎.我们常说的索引数据结构，就是由存储引擎层实现的，不同的存储引擎支持的索引类型也不相同，比如InnoDB支持索引类型是B+树，且是默认使用，也就是说在数据表中创建的主键索引和二级索引默认使用的是B+树索引。

**<u>【一条 SQL 查询语句的执行流程，依次看看每一个功能模块的作用】</u>**

#### 第一步：连接器

**【主要步骤】**

- **与客户端进行TCP三次握手建立连接；**
- **校验客户端的用户名和密码，如果用户名或密码不对，则会报错；**
- **如果用户名和密码都对了，会读取该用户的权限，然后后面的权限逻辑判断都基于此时读取到的权限；**

​	

​	如果在Linux操作系统里要使用MySQL，那第一步肯定是要先连接MySQL服务，然后才能执行SQL语句，普遍使用下面这条命令进行连接：

```bash
#-h指定 MySQL 服务得 IP 地址，如果是连接本地的 MySQL服务，可以不用这个参数；
#-u指定用户名，管理员角色名为 root;
#-p指定密码，如果命令行中不填写密码（为了密码安全，建议不要在命令行写密码），就需要在交互对话里面输入密码

mysql -h$ip -u$user -p
```

- **连接的过程需要先经过 TCP 三次握手，因为<font color = '#8D0101'> MySQL 是基于 TCP 协议进行传输的</font>**
- 如果 MySQL 服务正常运行，完成 TCP 连接的建立后，**连接器就要开始验证你的用户名和密码**
- 如果用户密码都没有问题，连接器就会获取该用户的权限，然后保存起来，后续该用户在此连接里的任何操作，都会基于连接开始时读到的权限进行权限逻辑的判断。

**【如何查看 MySQL 服务被多少个客户端连接了？】**

- 可以执行 `show processlist` 命令进行查看。
  - 共有两个用户名为 root 的用户连接了 MySQL 服务，其中 id 为 6 的用户的Command 列的状态为 `Sleep` ，这意味着该用户连接完 MySQL 服务就没有再执行过任何命令，也就是说这是一个空闲的连接，并且空闲的时长是 736 秒（ Time 列）。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E6%9F%A5%E7%9C%8B%E8%BF%9E%E6%8E%A5.png)

**【空闲连接会一直占用着吗？】**

- MySQL定义了**空闲连接的最大空闲时长，由 wait timeout 参数控制的，默认值是8小时 (28880秒)**，如果空闲连接超过了这个时间，连接器就会自动将它断开。

![image-20250318143151860](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318143151860.png)

**【MySQL 的连接数有限制吗？】**

- MySQL 服务支持的最大连接数由 `max_connections` 参数控制，比如我的 MySQL 服务默认是 151 个,超过这个值，系统就会拒绝接下来的连接请求，并报错提示“Too many connections”。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318141548778.png" alt="image-20250318141548778" style="zoom:70%;" />

#### 第二步：查询缓存

- 连接器的工作完成后，客户端就可以向MySQL服务发送SQL语句了，MySQL服务收到SQL语句后，就会解析出SQL语句的第一个字段，看看是什么类型的语句。
- 如果SQL是查询语句 (select语句), MySQL就会先去查询缓存 (Query Cache) 里查找缓存数据，看看之前有没有执行过这一条命令，这个查询缓存是以key-value形式保存在内存中的，key为SQL查询语句, value为SQL语句查询的结果。
- 如果查询的语句命中查询缓存，那么就会直接返回value给客户端。如果查询的语句没有命中查询缓存中，那么就要往下继续执行，等执行完后，查询的结果就会被存入查询缓存中。

这么看，查询缓存还挺有用，但是其实**查询缓存挺鸡肋**的。

- 对于更新比较频繁的表，查询缓存的命中率很低的，因为只要一个表有更新操作，那么这个表的查询缓存就会被清空。如果刚缓存了一个查询结果很大的数据，还没被使用的时候，刚好这个表有更新操作，查询缓冲就被清空了，相当于缓存了个寂寞。


> [!WARNING]
>
> 这里说的查询缓存是 server 层的，也就是 **<font color = '#8D0101'>MySQL 8.0版本移除的是 server层的查询缓存，并不是Innodb 存储引擎中的buffer pool</font>**。

#### 第三步：解析 SQL（解析器）

**<font color = '#8D0101'>解析器只负责检查语法和构建语法树，但是不会去查表或者字段存不存在。</font>**

解析器会做如下两件事情。

- **第一件事情，词法分析**。MySQL 会根据你输入的字符串识别出关键字出来，例如，SQL语句 select username from userinfo，在分析之后，会得到4个Token，其中有2个Keyword，分别为select和from：

| 关键字 | 非关键字 | 关键字 | 非关键字 |
| ------ | -------- | ------ | :------- |
| select | username | from   | userinfo |

- **第二件事情，语法分析**。根据词法分析的结果，语法解析器会根据语法规则判断你输入的这个 SQL 语句是否满足 MySQL 语法，如果没问题就会构建出 SQL 语法树，这样方便后面模块获取 SQL 类型、表名、字段名、 where 条件等等。

​	如果我们输入的 SQL 语句语法不对，就会在解析器这个阶段报错。比如，我下面这条查询语句，把 from 写成了 form，这时 MySQL 解析器就会给报错。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E8%AF%AD%E6%B3%95%E9%94%99%E8%AF%AF.png" alt="img" style="zoom:50%;" />

#### 第四步：执行 SQL

经过解析器后，接着就要进入执行 SQL 查询语句的流程了，每条`SELECT` 查询语句流程主要可以分为下面这三个阶段：

- **prepare 阶段，也就是预处理阶段，<font color = '#8D0101'>预处理器</font>；**
- **optimize 阶段，也就是优化阶段，<font color = '#8D0101'>优化器</font>；**
- **execute 阶段，也就是执行阶段，<font color = '#8D0101'>执行器</font>；**

##### 预处理器

**预处理阶段：**

- **检查 SQL 查询语句中的表或者字段是否存在；**
- **将 `select *` 中的 `*` 符号，扩展为表上的所有列；**

下面这条查询语句，test 这张表是不存在的，这时 MySQL就会在执行 SQL查询语句的 prepare 阶段中报错。

```bash
mysql> select * from test;
ERROR 1146 (42S02): Table "mysql. test' doesn't exist
```

##### 优化器

​	经过预处理阶段后，还需要为 SQL 查询语句先制定一个执行计划，这个工作交由「优化器」来完成的。

​	**<font color = '#8D0101'>优化器主要负责将 SQL 查询语句的执行方案确定下来</font>**，比如在表里面有多个索引的时候，优化器会基于查询成本的考虑，来决定选择使用哪个索引。

- 要想知道优化器选择了哪个索引，我们可以在查询语句最前面加个 `explain` 命令这样就会输出这条 SQL 语句的执行计划，然后执行计划中的 key 就表示执行过程中使用了哪个索引，比如下图的 key 为 `PRIMARY` 就是使用了主键索引

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92.png" alt="img" style="zoom:50%;" />

##### 执行器

​	经历完优化器后，就确定了执行方案，接下来 MySQL 就真正开始执行语句了，这个工作是由**「执行器」**完成的。在执行的过程中，执行器就会和存储引擎交互了，交互是以记录为单位的。

【**三种方式执行过程，来说明执行器和存储引擎的交互过程**】

- **主键索引查询**
- **全表扫描**
- **索引下推**

**【主键索引查询】**

```sql
select * from product where id = 1;
```

​	这条查询语句的查询条件用到了**主键索引**，而且是**等值查询**，同时主键id是唯一，不会有id 相同的记录，所以优化器决定选用**访问类型为 const 进行查询，也就是使用主键索引查询一条记录**，那么执行器与存储引擎的执行流程是这样的：

- **执行器第一次查询**，会调用read_first_record 函数指针指向的函数，因为优化器选择的访问类型为 const，这个函数指针被指向为InnoDB 引擎索引查询的接口，把条件 id = 1 交给存储引擎，**让存储引擎定位符合条件的第一条记录**。
- **存储引擎通过主键索引的B+树结构定位到id=1的第一条记录**，如果记录是不存在的，就会向执行器上报记录找不到的错误，然后查询结束。如果记录是存在的，就会将**记录返回给执行器**；
- **执行器从存储引擎读到记录**后，接着判断记录是否符合查询条件，如果**符合则发送给客户端，如果不符合则跳过该记录**。
- 执行器查询的过程是一个 while 循环，所以还会再查一次，但是这次因为不是第一次查询了，所以会调用read_record 函数指针指向的函数，因为优化器选择的访问类型为 const，这个函数指针被指向为一个永远返回-1的函数，所以当调用该函数的时候，执行器就退出循环，也就是结束查询了。

### 数据库缓冲池（buffer pool）

​	==缓冲池和查询缓存不是一个东西==

​	`InnoDB` 存储引擎是以页为单位来管理存储空间的，我们进行的**增删改查操作其实本质上都是在访问页面（包括读页面、写页面、创建新页面等操作）**。而**磁盘 I/O 需要消耗的时间很多，而在内存中进行操作，效率则会高很多**，为了能让数据表或者索引中的数据随时被我们所用，DBMS 会申请**占用内存来作为数据缓冲池**，在真正访问页面之前，需要**把在磁盘上的页缓存到内存中的 Buffer Pool 之后才可以访问**。

#### 缓冲池介绍

​	在MySQL启动的时候，**InnoDB会为Buffer Pool申请一片连续的内存空间，然后按照默认的16KB的大小划分出一个个的页，Buffer Pool中的页就叫做缓存页。**此时这些缓存页都是空闲的，之后随着程序的运行，才会有磁盘上的页被缓存到Buffer Pool中。

-  InnoDB 缓冲池包括了数据页、索引页、插入缓冲、锁信息、自适应 Hash 和数据字典信息等。

![image-20220615175309751](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220615175309751.png)

**【缓存原则】**

**“位置 * 频次 ”**这个原则，可以帮我们对 I/O 访问效率进行优化。

- 首先，**位置决定效率**，提供缓冲池就是为了**在内存中可以直接访问数据**。
- 其次，**频次决定优先级顺序**。因为**缓冲池的大小是有限的**，比如磁盘有 200G，但是内存只有 16G，缓冲池大小只有 1G，就无法将所有数据都加载到缓冲池里，这时就涉及到优先级顺序，会**优先对使用频次高的热数据进行加载**。

**【缓冲池的预读特性】**

- 缓冲池的作用就是**提升 I/O 效率**，而我们进行读取数据的时候存在一个“局部性原理”，也就是说我们使用了一些数据，**大概率还会使用它周围的一些数据**，因此采用**“预读”的机制提前加载，可以减少未来可能的磁盘 I/O 操作**。

#### 缓冲池如何读取数据

​	缓冲池管理器会尽量将经常使用的数据保存起来，在数据库进行页面读操作的时候，首先会**判断该页面是否在缓冲池中，如果存在就直接读取，如果不存在，就会通过内存或磁盘<font color = '#8D0101'>将页面存放到缓冲池中再进行读取</font>**。

![image-20220615222455867](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220615222455867.png)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220615193131719.png" alt="image-20220615193131719" style="zoom:80%;" />

**如果我们执行 SQL 语句的时候更新了缓存池中的数据，那么这些数据会马上同步到磁盘上吗？**

- 实际上，当我们对数据库中的记录进行修改的时候，首先会修改缓冲池中页里面的记录信息，然后数据库会以一定的频率刷新到磁盘中。注意并不是每次发生更新操作，都会立即进行磁盘回写。**缓冲池会采用一种叫做 checkpoint 的机制将数据回写到磁盘上，这样做的好处就是提升了数据库的整体性能**。
- 比如，当缓冲池不够用时，需要释放掉一些不常用的页，此时就可以强行采用checkpoint的方式，将不常用的脏页回写到磁盘上，然后再从缓存池中将这些页释放掉。这里的**脏页 (dirty page) 指的是缓冲池中被修改过的页，与磁盘上的数据页不一致**。





### 存储引擎

**【InnoDB】**

- MySQL从3.23.34a开始就包含InnoDB存储引擎。 **大于等于5.5之后，默认采用InnoDB引擎**。
- **InnoDB是MySQL的默认事务型引擎** ，它被设计用来处理大量的短期(short-lived)事务。可以确保事务的完整提交(Commit)和回滚(Rollback)。
- **对比MyISAM的存储引擎， InnoDB写的处理效率差一些 ，并且会占用更多的磁盘空间以保存数据和索引。**
- **MyISAM只缓存索引，不缓存真实数据；InnoDB不仅缓存索引还要缓存真实数据， 对内存要求较高** ，而且内存大小对性能有决定性的影响。

**【MyISAM引擎】**

- MyISAM提供了大量的特性，包括全文索引、压缩、空间函数(GIS)等，但MyISAM 不支持事务、行级锁、外键 ，有一个毫无疑问的缺陷就是 崩溃后无法安全恢复 。
- 5.5之前默认的存储引擎
- 优势是访问的 速度快 ，对事务完整性没有要求或者以SELECT、INSERT为主的应用
- 应用场景：只读应用或者以读为主的业务

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318160151712.png" alt="image-20250318160151712" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318160304565.png" alt="image-20250318160304565" style="zoom:95%;" />

## 池化设计思想、数据库连接池

**【池化】**	

​	**池化（Pooling）** 是一种通过**预先创建并复用资源** 来减少资源创建/销毁开销的设计模式，核心目标是**提高性能** 和**控制资源消耗** 。

​	这种设计会初始化预设资源，解决的问题就是抵消每次获取资源的消耗，如创建线程的开销，获取远程连接的开销等。除了初始化资源，池化设计还包括如下这些特征：池子的初始值、池子的活跃值、池子的最大值等，这些特征可以直接映射到java线程池和数据库连接池的成员属性中。

**典型应用场景**

- **线程池** ：复用线程，减少线程创建开销（如 `ExecutorService`）。
- **数据库连接池** ：复用数据库连接，避免频繁建立/断开 TCP 连接。
- **对象池** ：复用大对象（如数据库连接、网络套接字）。
- **内存池** ：预分配内存块，减少碎片化（如 Redis 的内存管理）。

​	数据库连接本质就是一个Socket的连接。数据库服务端还要维护一些缓存和用户权限信息之类的所以占用了一些内存。我们可以把数据库连接池是看做是维护数据库连接的缓存，以便将来需要对数据库的请求时可以重用这些连接，为每个用户打开和维护数据库连接，尤其是对动态数据库驱动的网站应用程序的请求，既昂贵又浪费资源。在连接池中，创建连接后，将其放置在池中，并再次使用它，因此不必建立新的连接，如果使用了所有连接，则会建立一个新连接并将其添加到池中。连接池还减少了用户必须等待建立与数据库的连接的时间

**【数据库连接池】**

​	数据库连接池是池化思想在数据库访问中的具体实现，**解决频繁创建/关闭连接导致的性能问题** 。

**主流数据库连接池对比**

| **连接池**      | **特点**                                                     | **适用场景**             |
| --------------- | ------------------------------------------------------------ | ------------------------ |
| **HikariCP**    | 轻量级、高性能（号称最快），通过优化代码和并发结构实现低延迟。 | 高并发、低延迟的微服务   |
| **Druid**       | 阿里开源，提供监控、SQL 防注入、日志分析等高级功能。         | 需要监控和安全防护的系统 |
| **C3P0**        | 早期流行，配置复杂，性能较低。                               | 传统遗留系统             |
| **Tomcat JDBC** | 与 Tomcat 集成度高，性能中等。                               | Tomcat 容器内应用        |



![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318160151712.png)

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318160151712-20251018231130399.png)



## MySQL索引

​	为什么要建索引，目的就是为了 **减少磁盘I/O的次数，加快查询速率。**

​	索引是一种用于**快速查询和检索数据的数据结构。常见的索引结构有:B树，B+树，Hash。**

### 建立索引要考虑的东西，索引的优点、缺点

**【优点】**

（1）类似大学图书馆建书目索引，提高数据检索的效率，**<font color = '#8D0101'>降低数据库的IO成本</font>**，这也是创建索引最主要的原因。

（2）通过创建唯一索引，可以保**证数据库表中每一行数据的唯一性**。

（3）在实现数据的参考完整性方面，可以**加速表和表之间的连接。换句话说，对于有依赖关系的子表和父表联合查询时，可以提高查询速度**。

（4）在使用分组和排序子句进行数据查询时，可以**显著减少查询中分组和排序的时间，降低了CPU的消耗**。

**【缺点】**

（1）**创建索引和维护索引要耗费时间**，并且随着数据量的增加，所耗费的时间也会增加。

（2）**索引需要占磁盘空间**，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，存储在磁盘上，如果有大量的索引，索引文件就可能比数据文件更快达到最大文件尺寸。

（3）**虽然索引大大提高了查询速度，同时却会降低更新表的速度**。当对表中的数据进行增加、删除和修改的时候，索引也要动态地维护，这样就降低了数据的维护速度。

### 索引基本操作CRUD

<u>**【创建索引】**</u>

**1）创建表的时候创建索引**

- 使用CREATE TABLE创建表时，除了可以定义列的数据类型外，还可以定义主键约束、外键约束或者唯一性约束，而不论创建哪种约束，在定义约束的同时相当于在指定列上创建了一个索引

```sql
CREATE TABLE dept(
dept_id INT PRIMARY KEY AUTO_INCREMENT,
dept_name VARCHAR(20)
);

CREATE TABLE emp(
emp_id INT PRIMARY KEY AUTO_INCREMENT,
emp_name VARCHAR(20) UNIQUE,
dept_id INT,
CONSTRAINT emp_dept_id_fk FOREIGN KEY(dept_id) REFERENCES dept(dept_id)
)
```

![image-20250318200249748](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318200249748.png)

![image-20250318200259865](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318200259865.png)

- **创建普通索引**

```sql
CREATE TABLE book(
book_id INT ,
book_name VARCHAR(100),
authors VARCHAR(100),
info VARCHAR(100) ,
comment VARCHAR(100),
year_publication YEAR,
INDEX(year_publication)
);
```

- **创建唯一索引**

```sql
CREATE TABLE test1(
id INT NOT NULL,
name varchar(30) NOT NULL,
UNIQUE INDEX uk_idx_id(id)
);
```

- **主键索引**

```sql
CREATE TABLE student (
id INT(10) UNSIGNED AUTO_INCREMENT ,
student_no VARCHAR(200),
student_name VARCHAR(200),
PRIMARY KEY(id)
);
```

- **创建组合索引**

```sql
CREATE TABLE test3(
id INT(11) NOT NULL,
name CHAR(30) NOT NULL,
age INT(11) NOT NULL,
info VARCHAR(255),
INDEX multi_idx(id,name,age)
);
```

**2）在已经存在的表上创建索引**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps11.jpg)

**<u>【删除索引】</u>**

- 删除表中的列时，如果要删除的列为索引的组成部分，则该列也会从索引中删除。如果组成索引的所有列都被删除，则整个索引将被删除。

![image-20250318200533065](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318200533065.png)

### 限制索引的数目

在实际工作中，我们也需要注意平衡；索引的数组不是越多越好。我们需要限制每张表上的索引数量，建议**单张索引数量不超过6个**。原因：

- **每个索引都需要占用磁盘空间**，索引越多，需要的磁盘空间越大
- 索引会**影响insert,delete,update等语句的性能**，因为表中数据更改的同时，索引也会进行调整和更新，会造成负担
- 优化器在选择如何优化查询是，会根据统一的信息，对每一个可以用到的索引进行评估，以生成出一个最好的执行计划，如果同时有多个索引都可以用于查询，会增加MySQL优化器生成执行计划时间，降低查询性能。

### 索引下推

​	启用索引下推后，把where条件由MySQL服务层放到了存储引擎层去执行，带来的好处就是**存储引擎根据id到表格中读取数据的次数变少了。从而减少回表查询次数，提升检索效率。**

**索引下推**

- 索引条件下推（Index Condition Pushdown），简称ICP。MySQL5.6新添加，用于优化数据的查询。
  - 当你不使用ICP，通过使用非主键索引（普通索引or二级索引）进行查询，存储引擎通过索引检索数据，然后返回给MySQL服务器，服务噐再判断是否符合条件。
  - 使用ICP，当存在索引的列做为判断条件时，MySQL服务噐将这一部分判断条件传递给存储引擎，然后存储引擎通过判断索引是否符合MySQL服务品传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器。

**适用场景**

- 当需要整表扫描，e.g range,ref,eq_ref.
- 适用InnoDB引擎和MyISAM引擎查询（5.6版本不适用分区查询，5.7版本可以用于分区表查询）。
- InnoDB引擎仅仅适用二级索引。（原因InnoDB聚簇索引将整行数据读到InnoDB缓冲区）。
- 子查询条件不能下推。触发条件不能下推，调用存储过程条件不能下推。

### 索引的设计原则

#### 哪些情况适合创建索引

 **1. 字段的数值有唯一性的限制**

![image-20220623154615702](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623154615702.png)

 **2. 频繁作为 WHERE 查询条件的字段**

- 某个字段在SELECT语句的 WHERE 条件中经常被使用到，那么就需要给这个字段创建索引了。尤其是在 数据量大的情况下，创建普通索引就可以大幅提升数据查询的效率。

  比如student_info数据表（含100万条数据），假设我们想要查询 student_id=123110 的用户信息。

**3. 经常 GROUP BY 和 ORDER BY 的列**

- 索引就是让数据按照某种顺序进行存储或检索，因此当我们使用 GROUP BY 对数据进行分组查询，或者使用 ORDER BY 对数据进行排序的时候，就需要对分组或者排序的字段进行索引 。**如果待排序的列有多个，那么可以在这些列上建立组合索引** 。

**4. UPDATE、DELETE 的 WHERE 条件列**

- 对数据按照某个条件进行查询后再进行 UPDATE 或 DELETE 的操作，如果对 WHERE 字段创建了索引，就能大幅提升效率。原理是因为我们需要先根据 WHERE 条件列检索出来这条记录，然后再对它进行更新或删除。**如果进行更新的时候，更新的字段是非索引字段，提升的效率会更明显，这是因为非索引字段更新不需要对索引进行维护**

 **5.DISTINCT 字段需要创建索引**

- 有时候我们需要对某个字段进行去重，使用 DISTINCT，那么对这个字段创建索引，也会提升查询效率。

- 比如，我们想要查询课程表中不同的 student_id 都有哪些，如果我们没有对 student_id 创建索引，执行 SQL 语句：

  - ```sql
    SELECT DISTINCT(student_id) FROM `student_info`;
    ```

    运行结果（600637 条记录，运行时间 0.683s ）

    如果我们对 student_id 创建索引，再执行 SQL 语句：

    ```sql
    SELECT DISTINCT(student_id) FROM `student_info`;
    ```

    运行结果（600637 条记录，运行时间 0.010s ）

 **6. 多表 JOIN 连接操作时，创建索引注意事项**

- 首先， `连接表的数量尽量不要超过 3 张` ，因为每增加一张表就相当于增加了一次嵌套的循环，数量级增 长会非常快，严重影响查询的效率。
- 其次， `对 WHERE 条件创建索引` ，因为 WHERE 才是对数据条件的过滤。如果在数据量非常大的情况下， 没有 WHERE 条件过滤是非常可怕的。
- 最后， `对用于连接的字段创建索引` ，并且该字段在多张表中的 类型必须一致 。比如 course_id 在 student_info 表和 course 表中都为 int(11) 类型，而不能一个为 int 另一个为 varchar 类型。

**7. 使用列的类型小的创建索引**

![image-20220623175306282](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623175306282.png)

 **8. 使用字符串前缀创建索引**

![image-20220623175513439](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623175513439.png)

```sql
create table shop(address varchar(120) not null);
alter table shop add index(address(12));
```

**拓展：Alibaba《Java开发手册》**

【 强制 】**在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引**，根据实际文本 区分度决定索引长度。

 **9. 使用最频繁的列放到联合索引的左侧**

- 这样也可以较少的建立一些索引。同时，由于"最左前缀原则"，可以增加联合索引的使用率。

**10.在多个字段都要创建索引的情况下，联合索引优于单值索引**

#### 不适合创建索引的情况

**1. 在where中使用不到的字段，不要设置索引**

- WHERE条件 (包括 GROUP BY、ORDER BY) 里用不到的字段不需要创建索引，**索引的价值是快速定位，如果起不到定位的字段通常是不需要创建索引的**。举个例子：
- 因为我们是按照 student_id 来进行检索的，所以不需要对其他字段创建索引，即使这些字段出现在SELECT字段中。

```sql
SELECT course_id, student_id, create_time
FROM student_info
WHERE student_id = 41251;
```

**2. 数据量小的表最好不要使用索引**

- 如果表记录太少，比如少于1000个，那么是不需要创建索引的。表记录太少，是否创建索引 `对查询效率的影响并不大`。甚至说，查询花费的时间可能比遍历索引的时间还要短，索引可能不会产生优化效果。

**3. 有大量重复数据的列上不要建立索引**

- 在条件表达式中经常用到的不同值较多的列上建立索引，**但字段中如果有大量重复数据，也不用创建索引**。比如在学生表的"性别"字段上只有“男”与“女”两个不同值，因此无须建立索引。如果建立索引，不但不会提高查询效率，反而会**严重降低数据更新速度**。

**4.避免对经常更新的表创建过多的索引**

- 第一层含义：频繁更新的字段不一定要创建索引。因为更新数据的时候，也需要更新索引，如果索引太多，在更新索引的时候也会造成负担，从而影响效率。
- 第二层含义：避免对经常更新的表创建过多的索引，并且索引中的列尽可能少。此时，虽然提高了查询速度，同时却降低更新表的速度。

**5.不建议用无序的值作为索引**

- 例如身份证、UUID(在索引比较时需要转为ASCII，并且插入时可能造成页分裂)、MD5、HASH、无序长字 符串等。

**6. 删除不再使用或者很少使用的索引**

- 表中的数据被大量更新，或者数据的使用方式被改变后，原有的一些索引可能不再需要。数据库管理员应当定期找出这些索引，将它们删除，从而减少索引对更新操作的影响。

### ==索引的分类==

MySQL的索引包括**普通索引、唯一性索引、全文索引、单列索引、多列索引和空间索引等**。

- 从功能逻辑上说，索引主要有4种，分别是**普通索引、唯一索引、主键索引、全文索引**。
- 按照物理实现方式，索引可以分为2种：**聚簇索引和非聚簇索引**。
- 按照作用字段个数进行划分，分成**单列索引和联合索引**。

#### 每一种索引的简介

**1）普通索引(非唯一索引)**

- 在创建普通索引时，不附加任何限制条件**，只是用于提高查询效率**。这类索引可以创建在任何数据类型中，其值是否唯一和非空，要由字段本身的完整性约束条件决定。建立索引以后，可以通过索引进行查询。

**2）唯一性索引**

- 使用**UNIQUE参数**可以设置索引为唯一性索引，在创建唯一性索引时，限制该**索引的值必须是唯一的，但允许有空值**。在—张数据表里可以有多个唯一索引。例如，在表student的字段email中创建唯一性索引，那么字段email的值就必须是唯一的。通过唯一性索引，可以更快速地确定某条记录。 

**3）主键索引**

- **主键索引就是一种特殊的唯一性索引，在唯一索引的基础上增加了不为空的约束，也就是NOT NULL+UNIQLUE**，一张表里最多只有一个主键索引。

**4）全文索引**

- 全文索引（也称全文检索)是目前搜索引擎使用的一种关键技术。它能够利用【分词技术】等多种算法智能分析出文本文字中关键词的频率和重要性，然后按照一定的算法规则智能地筛选出我们想要的搜索结果。全文索引非常适合大型数据集，对于小的数据集，它的用处比较小。 

**5）单列索引**

- 在表中的单个字段上创建索引。单列索引只根据该字段进行索引。单列索引可以是普通索引，也可以是唯一性索引，还可以是全文索引。只要保证该索引只对应一个字段即可。一个表可以有多个单列索引。

**6）多列、组合、联合索引**

- 多列索引是在表的多个字段组合上创建一个索引。该索引指向创建时对应的多个字段，可以通过这几个字段进行查询，但是**只有查询条件中使用了这些字段中的第一个字段时才会被使用**。例如，在表中的字段id、name和gender上建立一个多列索引idx_id_name_gender，只有在查询条件中使用了字段id时该索引才会被使用。**使用组合索引时遵循最左前缀集合**。

**7）聚簇和非聚簇索引**

- 索引按照物理实现方式，索引可以分为2种：**聚簇（聚集）索引(根据主键构建的索引)和非聚簇（非聚集）索引（根据非主键构建的索引）**。我们也把**非聚集索引称为二级索引或者辅助索引**。
- 聚簇索引的叶子节点就是数据节点，而非聚簇索引的叶子节点仍然是索引节点，只不过有指向对应数据块的指针。

#### 聚簇索引

​	**<font color = '#8D0101'>根据主键id，从小到大排列这些行数据，将这些数据页用双向链表的形式组织起来，再将这些页里的部分信息(比如主键值、页编号)提取出来放到一个新的16kb的数据页里，再加入层级的概念。于是，一个个数据页就被组织起来了，成为了一棵B+树索引。</font>**

- **<font color = '#8D0101'>索引结构和数据一起存放的索引，主键索引属于聚集索引</font>**。对于InnoDB引擎表来说，<font color = '#8D0101'>**该表的索引（B+树）的每个非叶子节点存储索引，叶子节点存储索引和索引对应的行数据**</font>

**Eg：建一个表：**

- 这个新建的 index_demo 表中有2个INT类型的列，1个CHAR（1）类型的列，而且我们规定了**c1列为主键**， 这个表使用Compact 行格式来实际存储记录的。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps12.jpg) 

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318203820488.png" alt="image-20250318203820488" style="zoom:90%;" />

==**【特点】**==

1）**使用记录主键值的大小进行记录和页的排序**，这包括三个方面的含义：

- **页内的记录**是按照主键的大小顺序排成一个**单向链表**
- 各个存放用户记录的**页**也是**根据页中用户记录的主键大小顺序排成一个双向链表**
- 存放目录项记录的页分为不同的层次，在**同一层次**中的页也是根据页中目录项记录的主键大小顺序排成一个**双向链表**；

2）**B+树的叶子节点存储的是完整的用户记录**

- 所谓完整的用户记录，就是指这个记录中存储了所有列的值（包括隐藏列）。

**【优点】**

- 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快；
- 聚簇索引对于主键的排序查找和范围查找速度非常快；
- 按照聚簇索引排列顺序，查询显示一定范围数据的时候，由于数据都是紧密相连，数据库不用从多个数据块中提取数据，所以节省了大量的IO操作。

**【缺点】**

- **插入速度严重依赖于有序的插入顺序**，按照主键的顺序插入是最快的方式，否则将会出现页分裂，严重影响性能。因此，对于InnoDB表，我们一般都会定义一个自增ID列为主键
- **更新主键的代价很高**，因为将会导致被更新的行移动，因此，对于InnoDB表，我们一般定义主键为不可更新；
- **二级索引访问需要两次索引查找**，第一次找到主键值，第二次根据主键值找到行数据

#### 非聚簇索引(二级索引或者辅助索引)（回表操作）

​	索引结构和数据分开存放的索引，二级索引属于非聚集索引。对于InnoDB引擎来说，**叶子节点存储索引和相应行数据的聚集索引键（即主键）。会先遍历辅助索引并通过叶子节点的指针获得指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录**。

eg：上边介绍的聚簇索引只能在搜索条件是主键值时才能发挥作用，因为B+树中的数据都是按照主键进行排序的。那如果我们想以别的列作为搜索条件该怎么办呢？肯定不能是从头到尾沿着链表依次遍历记录一遍。

- 我们可以多建几棵B+树，不同的B+树中的数据采用不同的排序规则。比方说**我们用c2列的大小作为数据页、页中记录的排序规则**，再建一棵B+树，效果如下图所示：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318210047277.png" alt="image-20250318210047277" style="zoom:90%;" />

==**【特点】**==

**使用记录c2列的大小进行记录和页的排序**，这包括三个方面的含义：

- **页内的记录是按照c2列的大小顺序**排成一个**单向链表**。
- 各个**存放用户记录的页也是根据页中记录的c2列大小顺序排成一个双向链表**。
- 存放目录项记录的页分为不同的层次，**在同一层次中的页也是根据页中目录项记录的c2列大小顺序排成一个双向链表**。

**B+树的<font color = '#8D0101'>叶子节点</font>存储的并不是完整的用户记录，而只是 <font color = '#8D0101'>c2列+主键这两个列的值</font>。**

**<font color = '#8D0101'>目录项记录</font>中不再是主键+页号的搭配，而变成了<font color = '#8D0101'>c2列+页号的搭配</font>。**

**【优缺点】**

![image-20250318210112077](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318210112077.png)

**==【回表】==**

​	我们根据这个以c2列大小排序并建立的B+树**只能确定我们要查找记录的主键值**，所以如果我们想根据c2列的值查找到完整的用户记录的话，仍然需要到聚簇索引中再查一遍，这个过程称为回表。也就是<font color = '#8D0101'>**根据c2列的值查询一条完整的用户记录需要使用到2棵B+树**</font>

> [!IMPORTANT]
>
> - 如果把完整的用户记录放到叶子节点是可以不用回表。但是太占地方了，相当于每建立一棵B+树都需要把所有的用户记录再都拷贝一遍，这就有点太浪费存储空间了。
> - 因为这种**按照非主键列建立的B+树需要一次回表操作才可以定位到完整的用户记录**，所以这种B+树也被称为**二级索引**（英文名 secondary index），或者辅助索引。由于我们使用的是c2列的大小作为B+树的排序规则，所以我们也称这个B+树是为c2列建立的索引。
> - 非聚簇索引的存在不影响数据在聚簇索引中的组织，所以**一张表可以有多个非聚簇索引**。

==**【非聚集索引一定回表查询吗(覆盖索引)】**==

如果直接构成了覆盖索引，那么就不会回表

![image-20250318210420066](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318210420066.png)

#### 联合索引

本质上也是一个==非聚簇索引==

​	我们也可以同时以多个列的大小作为排序规则，也就是同时为多个列建立索引，比方说我们想让B+树按照c2和c3列的大小进行排序，这个包含两层含义：

- 先把各个记录和页按照c2列进行排序。
- 在记录的c2列相同的情况下，采用c3列进行排序

​	注意一点，以c2和c3列的大小为排序规则建立的B+树称为联合索引，本质上也是一个二级索引。它的意思与分别为c2和c3列分别建立索引的表述是不同的，不同点如下：

- 建立联合索引只会建立如上图一样的1棵B+树。
- 为c2和c3列分别建立索引会分别以c2和c3列的大小为排序规则建立2棵B+树。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220616215251172.png" alt="image-20220616215251172" style="zoom:67%;" />



#### 覆盖索引

==一个索引包含了所有需要查询的字段的值，就称为覆盖索引。==

eg：表如下：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318211118475.png" alt="image-20250318211118475" style="zoom:40%;" />

```sql
CREATE INDEX multi_idx ON table(a,b) #建立a,b的联合索引

SELECT a,b FROM table #查询的a,b正好有索引，所以走索引覆盖，不用回表
SELECT a FROM table #查询的a正好有索引，所以走索引覆盖，不用回表
SELECT a,c FROM table #回表操作，需要通过主键id再次查出c

CREATE INDEX single_idx ON table(a) #建立a的普通索引
SELECT a FROM table #查询的a正好有索引，所以走索引覆盖，不用回表

```



### InnoDB中的索引方案、B+tree

​	**根据主键id，从小到大排列这些行数据，将这些数据页用双向链表的形式组织起来，再将这些页里的部分信息(比如主键值、页编号)提取出来放到一个新的16kb的数据页里，这些行数据通过单向链表连接，再加入层级的概念。于是，一个个数据页就被组织起来了，成为了一棵B+树索引。**

==树的层数越少，IO的次数越少：每到达一层IO一次==

**【建一个表】**

- 这个新建的 index_demo 表中有2个INT类型的列，1个CHAR（1）类型的列，而且我们规定了**c1列为主键**， 这个表使用Compact 行格式来实际存储记录的。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps12.jpg)

```sql
INSERT INTO index_demo VALUES(1, 4, 'u'), (3, 9, 'd'), (5, 3, 'y');
```

 **【index_demo 表的行格式示意图】**

![image-20250318211531485](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318211531485.png)

我们只在示意图里展示记录的这几个部分：

- **record_type** ：记录头信息的一项属性，表示记录的类型， **0 表示普通用户记录、1表示目录项纪录、2 表示最小记录、 3 表示最大记录**
- **next_record** ：记录头信息的一项属性，**表示下一条地址相对于本条记录的地址偏移量**，我们用 箭头来表明下一条记录是谁。
- **各个列的值** ：这里只记录在 index_demo 表中的三个列，分别是 c1 、 c2 和 c3 。
- **其他信息** ：除了上述3种信息以外的所有信息，包括其他隐藏列的值以及记录的额外信息。

![image-20250318213811024](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318213811024.png)

![image-20250318213830752](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318213830752.png)

**==【B+树结构示意】==**

现在以查找主键为 20 的记录为例，根据某个主键值去查找记录的步骤就可以大致拆分成下边两步：

- 我们生成了一个存储更高级目录项的页33 ，这个页中的两条记录分别代表页30和页32，如果用 户记录的主键值在 [1, 320) 之间，则到页30中查找更详细的目录项记录，如果主键值不小于320 的 话，就到页32中查找更详细的目录项记录，这里是去页30

- 到页30中通过 **二分法** 快速定位到对应目录项，因为 12 < 20 < 209 ，所以定位到对应的记录所在的页就是页9。
- 再到存储用户记录的页9中根据 二分法 快速定位到主键值为 20 的用户记录。

**1）目录项记录和普通的用户记录的不同点**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps15.jpg) 

**2）相同点**：两者用的是一样的数据页，都会为主键值生成Page Directory（页目录），从而在页内部按照主键值进行查找时可以使用二分法来加快查询速度

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318213847586.png" alt="image-20250318213847586" style="zoom:80%;" />

**可以用下边这个图来描述它**：这个数据结构的名称是B+树。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220616173717538.png" alt="image-20220616173717538" style="zoom:50%;" />

- 一个B+树的节点其实可以分成好多层，规定最下边的那层，也就是存放我们用户记录的那层为第 0 层， 之后依次往上加。真实环境中一个页存放的记录数量是非常大的
- 一般情况下，我们用到的 **B+树都不会超过4层** ，那我们通过主键值去查找某条记录最多只需要做4个页面内的查找（查找3个目录项页和一个用户记录页），又因为在每个页面内有所谓的 **Page Directory （页目录）**，所以在页面内也可以通过 **二分法** 实现快速 定位记录。

### InnoDB的B+树索引的注意事项

<u>**1.根页面位置万年不动（先复制，再分裂）**</u>

​	一个B+树索引的根节点自诞生之日起，便不会再移动。这样**只要我们对某个表建立一个索引，那么它的根节点的页号便会被记录到某个地方，然后凡是InnoDB存储引擎需要用到这个索引的时候，都会从那个固定的地方取出根节点的页号，从而来访问这个索引**。

**B+树的形成过程：**

- 每当为某个表创建一个**B+树索引（聚簇索引不是人为创建的，默认就有）的时候**，都会为这个索引创建一个**根节点页面**。最开始表中没有数据的时候，每个B+树索引对应的根节点中既没有用户记录，也没有目录项记录。
- 随后向表中插入用户记录时，**先把用户记录存储到这个根节点中**。
- 当根节点中的可用空间用完时继续插入记录，此时**会将根节点中的所有记录复制到一个新分配的页，比如页a中**，然后对这个新页进行页分裂(后一个数据页中的所有行主键值比前一个数据页中主键值大)的操作，得到另一个新页，比如页b。这时新插入的记录根据键值（也就是聚簇索引中的主键值，二级索引中对应的索引列的值）的大小就会被分配到页a或者页b中，而根节点便升级为存储目录项记录的页。

**<u>2.内节点中目录项记录的唯一性(索引列的值+主键值+页号)</u>**

​	为了让新插入记录能找到自己在哪个页里，我们需要保证在B+树的同一层内节点的目录项记录除页号这个字段以外是唯一的。所以对于二级索引的内节点的目录项记录的内容实际上是由三个部分构成的：**索引列的值+主键值+页号**

​	也就是我们**把主键值也添加到二级索引内节点中的目录项记录了**，这样就能**保证B+树每一层节点中各条目录项记录除页号这个字段外是唯一的**

![image-20250318220856550](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318220856550.png)

### 索引优化

![image-20250318221851296](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318221851296.png)

#### 索引失效

![image-20250318223356898](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318223356898.png)

![image-20250318225959187](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318225959187.png)

![image-20250318230006122](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318230006122.png)

![image-20250318230011554](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318230011554.png)

![image-20250318230016937](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318230016937.png)

![image-20250318230026376](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318230026376.png)

#### 最左前缀匹配原则（联合索引）

​	MySQL建立联合索引的规则是这样的，它首先会**根据联合索引中最左边的，也就是第一个字段进行排序，在第一个字段的基础上，在对联合索引的第二个字段进行排序，以此类推**。

​	综上，**第一个字段是绝对有序的，从第二个字段开始是无序的**，这就解释了为什么直接使用第二个字段进行条件判断用不到索引了（从第二个字段开始，无序，无法走B+Tree索引）。这就是MySQL在联合索引中强调最左匹配原则的原因。

![image-20250318230730106](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318230730106.png)

**两个条件：**

- **需要查询的列和组合索引的列顺序一致**
- **查询不要跨列**

**eg：**

![image-20250318231328216](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318231328216.png)

![image-20250318231344555](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318231344555.png)

![image-20250318231359347](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318231359347.png)

![image-20250318231417304](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318231417304.png)

#### 联合索引B+树是如何建立的？是如何查询的？

![image-20250318231857872](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318231857872.png)

![image-20250318232239555](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318232239555.png)

![image-20250318232251523](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318232251523.png)

### MyISAM与InnoDB对比

​	**MyISAM的索引方式都是“非聚簇”的，与InnoDB包含1个聚簇索引是不同的。小结两种引擎中索引的区别：从查找+data域去讲解**

① 在InnoDB存储引擎中，我们只需要根据主键值对 聚簇索引 进行一次查找就能找到对应的记录，而在 MyISAM 中却需要进行一次 回表 操作，意味着**MyISAM中建立的索引相当于全部都是 二级索引** 。

② I**nnoDB的数据文件本身就是索引文件，而MyISAM索引文件和数据文件是 分离的 ，索引文件仅保存数 据记录的地址**。

③ InnoDB的非聚簇索引data域存储相应记录主键的值 ，而MyISAM索引记录的是 地址 。换句话说， **InnoDB的所有非聚簇索引都引用主键作为data域**。

④ **MyISAM的回表操作是十分 快速 的，因为是拿着地址偏移量直接到文件中取数据的**，反观InnoDB是通 过获取主键之后再去聚簇索引里找记录，虽然说也不慢，但还是比不上直接用地址去访问。

⑤ **InnoDB要求表 必须有主键** （ MyISAM可以没有 ）。如果没有显式指定，则MySQL系统会自动选择一个 可以非空且唯一标识数据记录的列作为主键。如果不存在这种列，则MySQL自动为InnoDB表生成一个隐 含字段作为主键，这个字段长度为6个字节，类型为长整型。

​	为何**推荐使用整型自增主键**：整型是因为存储开销小并且比较索引大小的开销也更小；自增是因为方便插入一个数据，因为每层节点都是按照从左往右递增排列的

### MySQL索引数据结构选择

#### 二叉搜索树

如果我们利用二叉树作为索引结构，那么**磁盘的IO次数和索引树的高度是相关的**。

【**二叉搜索树的特点**】

- 一个节点只能有两个子节点，也就是一个节点度不能超过2
- 左子节点 < 本节点; 右子节点 >= 本节点，比我大的向右，比我小的向左

【**查找规则**】

![image-20220617163952166](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220617163952166.png)

![image-20220617164022728](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220617164022728.png)

![image-20250318221641002](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318221641002.png)

​	此时性能上已经退化成一个链表，所以时间复杂度为O(N)，性能很差。

​	为了提高查询效率，就需要**减少磁盘IO数。为了减少磁盘IO的次数，就需要尽量降低树的高度**，需要把原来“瘦高”的树结构变的“矮胖”，树的每层的分叉越多越好。

#### AVL树(平衡二叉树)和红黑树

**<u>（1）AVL树</u>**

![image-20250318232612578](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318232612578.png)

![image-20250318232636640](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318232636640.png)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318232654332.png" alt="image-20250318232654332" style="zoom:33%;" />

- AVL树：平衡二叉查找树，一般是用平衡因子差值判断是否平衡并通过旋转来实现平衡，左右子树树高不超过1，和红黑树相比，AVL树是严格的平衡二叉树，平衡条件必须满足（所有节点的左右子树高度差不超过1）。不管我们是执行插入还是删除操作，只要不满足上面的条件，就要通过旋转来保持平衡，而旋转非常耗时的，由此我们可以知道AVL树适合用于插入与删除次数比较少，但查找多的情况

**局限性：**

- 由于维护这种高度平衡所付出的代价比从中获得的效率收益还大，故而实际的应用不多，更多的地方是用追求局部而不是非常严格整体平衡的红黑树。当然，如果应用场景中对插入删除不频繁，只是对查找要求较高，那么AVL还是较优于红黑树。

**<u>（2）红黑树</u>**

​	**红黑树是一种二叉搜索树**，但在**每个节点增加一个存储位表示节点的颜色，可以是红或黑**（非红即黑）。通过对任何一条从根到叶子的路径上各个节点着色的方式的限制，**红黑树确保没有一条路径会比其它路径长出两倍**，因此，红黑树是一种**弱平衡二叉树**（由于是弱平衡，可以看到，在相同的节点情况下，AVL树的高度低于红黑树），相对于要求严格的AVL树来说，它的**旋转次数少**，所以**对于搜索，插入，删除操作较多的情况下，我们就用红黑树**。

- 每一个结点都有一个颜色，要么为红色，要么为黑色；
- **树的根结点为黑色**；
- 树中不存在两个相邻的红色结点（即红色结点的父结点和孩子结点均不能是红色）；
- 从任意一个结点（包括根结点）到其任何后代NULL结点（默认是黑色的）的每条路径都具有相同数量的黑色结点。
- **叶子节点（Null节点）都是黑色的；**
- **所有新插入节点都为红色**

![image-20250318233025700](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233025700.png)

**<u>（3）两者对比</u>**

![image-20250318233223557](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233223557.png)

![image-20250318233248524](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233248524.png)

#### 红黑树和B+树

**【范围查询效率】**

**B+树的优势**：

- 所有数据都存储在叶子节点，且叶子节点通过指针相连形成链表
- 范围查询非常高效（找到起始点后顺序遍历叶子节点即可）
- 适合数据库常见的`BETWEEN`、`>`、`<`等范围查询操作

**红黑树的劣势**：

- 数据分布在整个树中，没有这种顺序链接
- 范围查询需要多次遍历，效率低下

【**缓存友好性**】

**B+树的优势**：

- 非叶子节点只存储键值不存储数据，可以缓存更多索引信息
- 一次磁盘读取可以加载更多键值到内存
- 更高的缓存命中率

**红黑树的劣势**：

- 每个节点都存储数据，缓存效率较低

【**实际案例分析**】

假设一个包含1亿条记录的表：

- 使用B+树（假设每个节点500个键值）：
  - 高度约为3（500^3=125,000,000 > 100,000,000）
  - 范围查询只需3次I/O找到起始点，然后顺序读取
- 使用红黑树：
  - 高度约为27（log₂(100,000,000)≈26.57）
  - 范围查询需要约27次I/O找到起始点，然后复杂遍历

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233323666.png" alt="image-20250318233323666" style="zoom:80%;" />

**一页为16kb**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233331766.png" alt="image-20250318233331766" style="zoom:80%;" />

**【查询时间】**

一次典型的磁盘读取操作（如B+树查询需要读取一个页）包括以下时间开销：

```
总时间 = 寻道时间 + 旋转延迟 + 传输时间 + 系统开销
```

**2.1 B+树查询示例**

假设一个3层的B+树索引：

- **HDD环境**：
  - 最坏情况：3次随机I/O × 10ms = 30ms
  - 最佳情况：预读可能减少到10-15ms
- **SSD环境**：
  - 通常：3次I/O × 0.2ms = 0.6ms
  - 并行处理可能更快

**2.2 与红黑树对比**

假设红黑树高度为27（1亿条记录）：

- **HDD**：27 × 10ms = 270ms（比B+树慢约9倍）
- **SSD**：27 × 0.2ms = 5.4ms（仍比B+树慢约9倍）

#### B+树存储结构和B树的区别

**【B树的结构】**

**B-树是多路平衡搜索树**

​	B树结构图中可以看到**每个节点中不仅包含数据key值，还有data值**。而**每个页的存储空间是有限的，如果data数据较大时将会导致每个页存储的key的数量很小**，**当存储的数据量很大时同样会导致B树的深度很大，增大查询时磁盘的IO次数进而影响查询效率**。

![image-20250318233828008](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318233828008.png)

**【B+树的结构】**

​	**B+树：加强版多路平衡搜索树**，**所有的data信息都在叶子节点中，而且叶子节点之间会有个双向指针指向，这个也是B+树的核心点**，这样可以大大提升范围查询效率，也方便遍历整个树。

**【二者的区别】**

二者有着一个根本差别：**B+树的中间节点不直接存储数据**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318234438283.png" alt="image-20250318234438283" style="zoom:80%;" />

### 索引相关面试问题

#### B+树的存储能力如何？为何说一般查找行记录，最多只需1~3次磁盘IO

![image-20250318234542020](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318234542020.png)

#### Hash索引与B+树索引的区别

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps17.jpg) 

#### 为什么要用b+树

![image-20250318234726066](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250318234726066.png)

#### InnoDB表必须有主键，并且推荐使用整型的自增主键

1）必须要有主键列，因为表数据文件本身就是按B+Tree组织的一个索引结构文件。如果没有设置，会自动生成一个。

2）整型是因为，存储开销小和比较索引大小的开销也更小；

3）自增：方便插入一个数据，因为每层节点都是按照从左往右递增排列的。

#### 自增主键和UUID

​	MySQL在使用默认存储引擎InnoDB的情况下，绝大多数情况下建议使用自增主键。需要注意的是，InnoDB有一个特性，默认会设置主键为聚簇索引。

**1）性能**

- 自增主键能保证数据行是顺序写入的，批量插入快，关联操作性能好。方便插入一个数据，因为每层节点都是按照从左往右递增排列的。
- UUID是不连续的，且值分布范围非常大。UUID作为聚簇索引会使索引的插入完全随机，导致大量的随机IO；并产生大量的页分裂操作，导致大量数据移动，产生很多额外开销

**2）存储成本**

- 自增：存储开销小和比较索引大小的开销也更小，由于mysql使用B+树索引，叶子节点是从小到大排序的，如果使用自增id做主键，这样**每次数据都加在B+树的最后，比起每次加在B+树中间的方式，加在最后可以有效减少页分裂的问题，如果此时最末尾的数据页满了，那创建个新的页就好**。如果主键不是自增的，比方说上次分配了id=7，这次分配了id=3，为了让新加入数据后B+树的叶子节点还能保持有序，它就需要往叶子结点的中间找，查找过程的时间复杂度是O(Ign)，如果这个页正好也满了，这时候就需要进行页分裂了。并且**页分裂操作本身是需要加悲观锁的**。总体看下来，**自增的主键遇到页分裂的可能性更少，因此性能也会更高**。
- UUID主键不连续、随机写入的特点造成的频繁的**页分裂**，会导致页变得稀疏，产生不规则的数据填充问题，最终导致**碎片化的存储，增加存储成本**

**3）并发争用**

- 自增主键也有不太好的方面。高并发场景下，**自增主键的下一个值可能会被不同线程争用**。这种情况下可以通过设置参数innodb_autoinc_lock_mode来适配不同场景

#### MySQL自增主键在数据库重启时，是否可以按照重启之前继续自增。

- 在**MySQL 5.7**系统中，对于自增主键的分配规则，是由InnoDB数据字典内部一个计数器来决定的，而该**计数器只在内存中维护，并不会持久化到磁盘中。当数据库重启时，该计数器会被初始化**。
- **MySQL 8.0将自增主键的计数器持久化到redo日志中**。每次计数器发生改变，都会将其写入redo日志中。如果数据库重启，**InnoDB会根据redo日志中的信息来初始化计数器的内存值**

#### 具有重复值的字段能否创建索引

**可以创建索引，只是不能创建唯一索引，可以创建普通索引**

**1）普通索引**

在创建普通索引时，不附加任何限制条件，只是用于提高查询效率。这类索引可以创建在任何数据类型中，其值是否唯一和非空，要由字段本身的完整性约束条件决定。建立索引以后，可以通过索引进行查询。

**2）唯一性索引**

使用UNIQUE参数可以设置索引为唯一性索引，在创建唯一性索引时，限制该索引的值必须是唯一的，但允许有空值。在—张数据表里可以有多个唯一索引。例如，在表student的字段email中创建唯一性索引，那么字段email的值就必须是唯一的。通过唯一性索引，可以更快速地确定某条记录。

#### Where a=1,b=2,c=3建立什么索引，where b=2，c=3是否走索引，where b= 5,a = 6是否走索引。如果a>1,b=2怎么设计索引

1） 建立联合索引

2） 不走索引，因为有个最左匹配原则

3） 走索引

4） 设计(b，a)联合索引。==最左匹配原则，直到遇到范围查询(>，<, between, like)就停止==，比如where a = 1 and b = 2 and c >3and d = 4。如果建立(a,b,c,d)顺序的索引，**d是用不到索引的**，如果建立(a,b,d,c)的索引则都可以用到，abd的顺序可以任意调整

## SQL优化

### 统计SQL的查询成本：last_query_cost

​	一条SQL查询语句在执行前需要查询执行计划，如果存在多种执行计划的话，MySQL会计算每个执行计划所需要的成本，从中选择**成本最小**的一个作为最终执行的执行计划。

- 如果我们想要查看某条SQL语句的查询成本，可以在执行完这条SQL语句之后，通过查看当前会话中的`last_query_cost`变量值来得到当前查询的成本。它通常也是我们**评价一个查询的执行效率的一个常用指标**。这个查询成本对应的是**SQL 语句所需要读取的读页的数量**。

> [!WARNING]
>
> SQL查询时一个动态的过程，从页加载的角度来看，我们可以得到以下两点结论：
>
> 1. **位置决定效率**。如果**页就在数据库 `缓冲池` 中，那么效率是最高的，否则还需要从 `内存` 或者 `磁盘` 中进行读取**，当然针对单个页的读取来说，如果页存在于内存中，会比在磁盘中读取效率高很多。
> 2. **批量决定效率**。如果我们从磁盘中对单一页进行随机读，那么效率是很低的(差不多10ms)，而采用**顺序读取的方式，批量对页进行读取**，平均一页的读取效率就会提升很多，甚至要快于单个页面在内存中的随机读取。
>
> 所以说，遇到I/O并不用担心，方法找对了，效率还是很高的。我们首先要考虑数据存放的位置，如果是进程使用的数据就要尽量放到`缓冲池`中，其次我们可以充分利用磁盘的吞吐能力，一次性批量读取数据，这样单个页的读取效率也就得到了提升。

### 定位执行慢的 SQL：慢查询日志

![image-20250319000502846](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319000502846.png)

**【开启 slow_query_log】**

- 在使用前，我们需要先查下慢查询是否已经开启，使用下面这条命令即可：

```bash
mysql > show variables like '%slow_query_log';
```

![image-20220628173525966](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628173525966.png)

- 我们可以看到 `slow_query_log=OFF`，我们可以把慢查询日志打开，注意设置变量值的时候需要使用 global，否则会报错：

```bash
mysql > set global slow_query_log='ON';
```

- 然后我们再来查看下慢查询日志是否开启，以及慢查询日志文件的位置：

  - 你能看到这时慢查询分析已经开启，同时文件保存在 `/var/lib/mysql/atguigu02-slow.log` 文件 中。

    ![image-20220628175226812](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628175226812.png)

【**修改 long_query_time 阈值**】

```bash
#测试发现：设置global的方式对当前session的long_query_time失效。对新连接的客户端有效。所以可以一并
执行下述语句
mysql > set global long_query_time = 1;
mysql> show global variables like '%long_query_time%';

mysql> set long_query_time=1;
mysql> show variables like '%long_query_time%';
```

![image-20220628175425922](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628175425922.png)

### 慢查询日志分析工具：mysqldumpslow

​	在生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活，MySQL提供了日志分析工具 `mysqldumpslow` 。

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

### 查看 SQL 执行成本：SHOW PROFILE

执行相关的查询语句。接着看下当前会话都有哪些 profiles，使用下面这条命令：

```bash
mysql > show profiles;
```

![image-20220628205243769](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628205243769.png)

**日常开发需注意的结论：**

① `converting HEAP to MyISAM`: 查询结果太大，内存不够，数据往磁盘上搬了。

② `Creating tmp table`：创建临时表。先拷贝数据到临时表，用完后再删除临时表。

③ `Copying to tmp table on disk`：把内存中临时表复制到磁盘上，警惕！

④ `locked`。

如果在show profile诊断结果中出现了以上4条结果中的任何一条，则sql语句需要优化。

### ==分析查询语句：EXPLAIN==

![image-20250319001410323](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319001410323.png)

#### 基本语法

EXPLAIN 或 DESCRIBE语句的语法形式如下：

```sql
EXPLAIN SELECT select_options
或者
DESCRIBE SELECT select_options
```

如果我们想看看某个查询的执行计划的话，可以在具体的查询语句前边加一个 EXPLAIN ，就像这样：

```sql
mysql> EXPLAIN SELECT 1;
```

![image-20220628212029574](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628212029574.png)

#### ==EXPLAIN 语句输出==

![image-20250319001708346](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319001708346.png)

![image-20250319001738742](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319001738742.png)

**<u>（1）table：表名</u>**

​	不论我们的查询语句有多复杂，包含了多少个表，到最后也是需要对每个表进行单表访问的，所以MySQL规定**EXPLAIN语句输出的每条记录都对应着某个单表的访问方法**，该条记录的**table列代表着该表的表名**（有时不是真实的表名字，可能是简称）。

```sql
mysql > EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

- 可以看出这个连接查询的执行计划中有两条记录，这两条记录的table列分别是s1和s2，这两条记录用来分别说明对s1表和s2表的访问方法是什么

![image-20220628221414097](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628221414097.png)

**<u>（2）id：在一个大的查询语句中<font color = '#8D0101'>每个SELECT关键字都对应一个唯一的id</font></u>**

- 对于连接查询来说，一个SELECT关键字后边的FROM字句中可以跟随多个表，所以在连接查询的执行计划中，每个表都会对应一条记录，但是这些记录的id值都是相同的，比如：

```sql
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2;
```

![image-20220628222251309](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220628222251309.png)

- 对于包含子查询的查询语句来说，就可能涉及多个`SELECT`关键字，所以在包含子查询的查询语句的执行计划中，每个`SELECT`关键字都会对应一个唯一的id值，比如这样：

```sql
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
```

![image-20220629165122837](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220629165122837.png)

<u>**（3） select_type**</u>

SELECT关键字对应的那个查询的类型，确定小查询在整个大查询中扮演了一个什么角色

**常见类型**

![image-20250319002645002](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319002645002.png)

**<u>（4）partitions</u>**

==**<u>（5）type</u>**==

​	执行计划的一条记录就代表着MySQL对某个表的 **执行查询时的访问方法,** 又称“访问类型”，其中的 `type` 列就表明了这个访问方法是啥，是较为重要的一个指标。比如，看到`type`列的值是`ref`，表明`MySQL`即将使用`ref`访问方法来执行对`s1`表的查询。

​	完整的访问方法如下： **`system ， const ， eq_ref ， ref ， fulltext ， ref_or_null ， index_merge ， unique_subquery ， index_subquery ， range ， index ， ALL`** 。

结果值从最好到最坏依次是：

**1.system**【表中只有一条记录，且表中使用的存储引擎的统计数据是精确的，MyISAM、MEMORY】

**2.<font color = '#8D0101'>const</font>【根据主键或唯一二级索引与常数等值匹配】**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps18.jpg) 

**3.<font color = '#8D0101'>eq_ref</font>** 【连接查询时，如果被驱动表根据主键或唯一二级索引与常数等值匹配】

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps19.jpg) 

**4.<font color = '#8D0101'>ref</font>**【普通二级索引列与常量等值匹配】

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps20.jpg) 

**5.fulltext**【使用全文索引的时候是这个类型】

**6.ref_or_null** 【普通二级索引列于常量等值匹配，该索引值可以是NULL】

**7.<font color = '#8D0101'>index_merge</font>**【单表访问方法时在某些场景下可以使用Intersection '、‘Union '、# Sort-Union这三种索引合并的方式来执行查询】

**8.unique_subquery**

**9.index_subquery** 

**10.<font color = '#8D0101'>range</font>**【如果使用索引获取某些范围区间的记录，那么就可能用到range访问方法】

**11.<font color = '#8D0101'>Index</font>**【使用索引覆盖,【不用回表就查到数据，换句话说查询列要被所使用的索引覆盖】，需要扫描全部的索引记录时，该表的访问方法就是index】

**12.<font color = '#8D0101'>ALL</font>**【全表扫描】



**<u>（6）key</u>**

- **查询真正使用到的索引**，select_type为index_merge时，这里可能出现两个以上的索引，其他的select_type这里只会出现一个。

<u>**（7）key_len**</u>

- **用于处理查询的索引长度，主要针对联合索引**

​	如果是单列索引，那就整个索引长度算进去，如果是多列索引，那么查询不一定都能使用到所有的列，具体使用到了多少个列的索引，这里就会计算进去，没有使用到的列，这里不会计算进去。留意下这个列的值，算一下你的多列索引总长度就知道有没有使用到所有的列了。

**<u>（8）ref：</u>**

- 如果是使用的**常数等值查询，这里会显示const**，如果是连接查询，被驱动表的执行计划这里会显示驱动表的关联字段，如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里可能显示为func

**<u>（9）rows：</u>**

- 这里是执行计划中估算的扫描行数，不是精确值。这个值非常直观显示SQL的效率好坏,原则上rows越少越好.

**<u>（10）filtered：</u>**

- 越大越好

**<u>（11）Extra：</u>**

- Extra列是用来说明一些额外信息的，包含不适合在其他列中显示但十分重要的额外信息。我们可以通过这些额外信息来更准确的理解MySQL到底将如何执行给定的查询语句。

### SQL查询很慢怎么办

1、**开启慢查询日志**，**根据日志找到哪条SQL语句执行的慢**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps21.jpg) 

2、**用Explain关键字分析这条SQL语句**，查看是否建立了索引。如果没有，就可以考虑建立索引。如果建立了索引,但是并没有使用到，就分析它为什么没有使用到索引。

- **字段没有索引**

- **字段有索引，但却没有用索引**

  - 如果我们在**字段的左边做了运算**，那么在查询的时候，就不会用上索引了。**字段上有索引，但由于自己的疏忽，导致系统没有使用索引**的情况。

    ![image-20250319133847375](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319133847375.png)

  - **函数操作导致没有用上索引**

    ![image-20250319133921158](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319133921158.png)

  - **范围查询没有用上索引**

    ```sql
    select * from t where a < 10;
    ```

- SQL语句本身是没有什么问题的，**可能是数据库在刷新脏页**，由于Redo log日志写满了，数据库需要暂停其他操作，同步数据到磁盘，就有可能导致我们的SQL语句执行的很慢了。执行的这条语句，刚好这条语句涉及到的表或行，别人在用，并且加锁了，我们拿不到锁，只能慢慢等待别人释放锁了。如果要**判断是否真的在等待锁，我们可以用<font color = '#8D0101'>`show processlist`</font>c这个命令来查看当前的状态**

### 大数据量下慢查询优化

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319134224094.png" alt="image-20250319134224094" style="zoom:80%;" />

1.优化SQL语句和索引；

2.加缓存，memcached，redis

3.以上都做了后，还是慢，尝试**主从复制或者主主复制，读写分离，可以在应用层做，效率高。**

4.如果以上都做了还是慢，不要想着去做切分，mysql自带分区表，先试试这个，对你的应用是透明的，无需更改代码，但是sql语句是需要针对分区表做优化的，sql条件中要带上分区条件的列，从而使查询定位到少量的分区上，否则就会扫描全部分区，另外分区表还有一些坑，在这里就不多说了

5.如果以上都做了，那就先做垂直拆分，其实就是根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统; 

6.再做水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key[分表键]，为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改,sql中尽量带sharding key分片键，将数据定位到限定的表上去查，而不是扫描全部的表，mysql数据库一般都是按照这个步骤去演化的，成本也是由低到高

## ==数据库调优的维度和步骤==

**1.优化表设计**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps27.jpg" alt="img" style="zoom:80%;" /> 

**2.优化逻辑查询(sql优化)**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps28.jpg" alt="img" style="zoom:80%;" />

**==3.优化物理查询==**

​	物理查询优化是在确定了逻辑查询优化之后，采用**物理优化技术（比如索引等）**，通过计算代价模型对各种可能的访问路径进行估算，从而找到执行方式中代价最小的作为执行计划。在这个部分中，我们需要掌握的重点是对索引的创建和使用。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319140846737.png" alt="image-20250319140846737" style="zoom:80%;" />

**==4.使用Redis或Memcached作为缓存==**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141009256.png" alt="image-20250319141009256" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141016229.png" alt="image-20250319141016229" style="zoom:80%;" />

**==5.库级优化==**

**【读写分离】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141125708.png" alt="image-20250319141125708" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319141150939.png" alt="image-20250319141150939" style="zoom:80%;" />

## 大表优化

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319143827965.png" alt="image-20250319143827965" style="zoom:80%;" />

当MySQL单表记录数过大时，数据库的CRUD性能会明显下降，一些常见的优化措施如下

### 限定数据的范围

务必禁止不带任何限制数据范围条件的查询语句。比如∶我们当用户在查询订单历史的时候，我们可以控制在一个月的范围内;

### 读/写分离

经典的数据库拆分方案，**主库负责写，从库负责读**;

![image-20250319144032424](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144032424.png)

![image-20250319144042044](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144042044.png)

### 垂直拆分

当数据量级达到千万级以上时，有时候我们需要把一个数据库切成多份，放到不同的数据库服务器上，减少对单一数据库服务器的访问压力。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps31.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps32.jpg) 

- 垂直拆分的优点：可以使得列数据变小，在查询时减少读取的Block数，减少I/O次数。此外，垂直分区可以简化表的结构，易于维护。
- 垂直拆分的缺点：主键会出现冗余，需要管理冗余列，并会引起JOIN操作。此外，垂直拆分会让事务变得更加复杂。

### 水平拆分

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144308995.png" alt="image-20250319144308995" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144326193.png" alt="image-20250319144326193" style="zoom:80%;" />

下面补充一下数据库分片的两种常见方案：

- 客户端代理：分片逻辑在应用端，封装在jar包中，通过修改或者封装JDBC层来实现。当当网的Sharding-JDBC、阿里的TDDL是两种比较常用的实现。
- 中间件代理：在应用和数据中间加了一个代理层。分片逻辑统一维护在中间件服务中。我们现在谈的Mycat、360的Atlas、网易的DDB等等都是这种架构的实现。

## 主从复制

### 如何提高数据库并发能力

如果我们的目的在于提升数据库高并发访问的效率，那么：

- **首先考虑的是如何优化SQL和索引，这种方式简单有效；**
- **其次才是采用缓存的策略，比如使用Redis将热点数据保存在内存数据库中，提升读取的效率；**
- **最后才是对数据库采用主从架构，进行读写分离。** 

![image-20251018231632416](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018231632416.png)

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps34.jpg) 

- 此外，一般应用对数据库而言都是“**读多写少**”，也就说对数据库读取数据的压力比较大，有一个思路就是采用**数据库集群的方案，做主从架构、进行读写分离，这样同样可以提升数据库的并发处理能力**。但并不是所有的应用都需要对数据库进行主从架构的设置，毕竟设置架构本身是有成本的。

### 主从复制的作用

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144613895.png" alt="image-20250319144613895" style="zoom:80%;" />

**【读写分离】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144641544.png" alt="image-20250319144641544" style="zoom:80%;" />

**【数据备份】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144708607.png" alt="image-20250319144708607" style="zoom:80%;" />

**【具有高可用性】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144730468.png" alt="image-20250319144730468" style="zoom:80%;" />

### 主从复制的原理

==**Slave会从Master读取binlog来进行数据同步。**==

实际上**主从同步的原理就是基于binlog进行数据同步的**。在主从复制过程中，会**基于3个线程来操作，一个主库线程，两个从库线程**。

**【涉及三个线程】**

- **二进制日志转储线程（Binlog dump thread）主库线程**：负责将主服务器上的数据更改写入二进制日志（Binary log）中，当从库线程连接的时候，主库可以将二进制日志发送给从库
- **从库I/O线程**：连接到主库，向主库发送请求更新Binlog。这时从库的I/O线程就可以读取到主库的二进制日志转储线程发送的Binlog更新部分，并且拷贝到本地的中继日志（Relay log）。
- **从库SQL线程**：读取从库中的中继日志，并且执行日志中的事件，将从库中的数据与主库保持同步。

![image-20250319144941240](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319144941240.png)

**【复制三步骤】**

- Master将写操作记录到二进制日志（binlog），这些记录过程叫做二进制日志事件，binary log events;
- Slave通过I/O线程将Master的binary log events拷贝到它的中继日志（relay log）；
- Slave重做中继日志中的事件，将改变应用到自己的数据库中。MySQL复制是异步的且串行化的，而且重启后从接入点开始复制，而Redis则是会复制主库的所有操作

**复制的最大问题：<font color = '#8D0101'>延时</font>**

**【复制的基本原则】**

- 每个Slave只有一个Master
- 每个Slave只能有一个唯一的服务器ID
- 每个Master可以有多个Slave

### 主从复制架构

**一主一从架构搭建**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps35.jpg) 

**双主双从架构搭建**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps36.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps37.jpg) 

### 解决同步数据一致性问题

数据一致性问题：**主库更新后，从库还没有完成更新，这时的读从库可能会读到的是旧数据**。

**【主从同步的要求】**

- 读库和写库的数据一致(最终一致)；
- 写数据必须写到写库；
- 读数据必须到读库；

**【主从延迟原因】**

​	进行主从同步的内容是二进制日志，它是一个文件，在进行网络传输的过程中就一定会存在主从延迟（比如500ms），这样就可能造成**用户在从库上读取的数据不是最新的数据**，也就是主从同步中的数据不一致性问题。

**【解决方法】**

- 如果操作的数据存储在同一个数据库中，那么对数据进行更新的时候，可以对记录加写锁，这样在读取的时候就不会发生数据不一致的情况。但这时从库的作用就是备份，并没有起到读写分离，分担主库读压力的作用。

  ![image-20250319145421959](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319145421959.png)

- 读写分离情况下，<font color = '#8D0101'>**解决主从同步中数据不一致的问题，就是解决主从之间数据复制方式的问题**</font>，如果**<font color = '#8D0101'>按照数据一致性从弱到强来进行划分，有以下3种复制方式</font>。**

#### 异步复制（数据一致性最弱）

​	异步模式就是**客户端提交COMMIT之后不需要等从库返回任何结果，而是直接将结果返回给客户端**，这样做的好处是不会影响主库写的效率，但**可能会存在主库宕机，而Binlog还没有同步到从库的情况，也就是此时的主库和从库数据不一致**。这时候从从库中选择一个作为新主，那么新主则可能缺少原来主服务器中已提交的事务。所以，这种复制模式下的数据一致性是最弱的。

![image-20250319145635656](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319145635656.png)

#### 半同步复制

​	MySQL5.5版本之后开始支持半同步复制的方式。原理是在**客户端提交COMMIT之后不直接将结果返回给客户端，而是等待至少有一个从库接收到了Binlog，并且写入到中继日志中，再返回给客户端**。这样做的好处就是提高了数据的一致性，当然相比于异步复制来说，至少多增加了一个网络连接的延迟，降低了主库写的效率。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319145731681.png" alt="image-20250319145731681" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319150014890.png" alt="image-20250319150014890" style="zoom:80%;" />

#### 全同步复制

当主库提交事务之后，所有的从库节点必须收到、APPLY并且提交这些事务，然后主库线程才能继续做后续操作。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的**性能必然会收到严重的影响**。



## 数据库的设计规范/范式

### 建表要考虑哪些因素

**<u>1）确定所储存的值(类型及是否定长)</u>**

- varchar可变长度和char长度固定(char费空间,速度快;varchar省空间,速度慢)
- 数字类型用数组类型存储,时间类型用时间类型存储,不要用字符类型存储：字符串占用的空间更大；字符串存储的日期比较效率比较低（逐个字符进行比对），无法用日期相关的API进行计算和比较。
- 尽可能的定义“NOT NULL”
- 如果要储存的数据为字符串,且可能值已知且有限,优先使用enum或set
- 尽量不要使用定义外键

**<u>2）遵循3大范式</u>**

- 第一范式是最基本的范式。如果数据库中的所有字段都是不可分解的原子值，就说明该数据库表满足了第一范式。属性不可分。
- 第二范式需要确保数据库表中的每一列都和主键相关，而不能只与主键的某一部分相关（主要针对联合主键而言，对于联合主键必须跟两个主键都相关才行）。**如果知道主键的所有属性的值，就可以检索到任何元组（行）的任何属性的任何值**
- 第三范式是在第二范式的基础上，确保数据表中的每一个非主键字段都和主键字段直接相关，也就是说，要求数据表中的所有非主键字段不能依赖于其他非主键字段。（即，不能存在非主属性A依赖于非主属性B，非主属性B依赖于主键C的情况，即存在"A-->B-->C"的决定关系）通俗地讲，该规则的意思是所有非主键属性之间不能有依赖关系，必须**相互独立**

**<u>3）索引</u>**

==哪些情况适合创建索引==

 **1. 字段的数值有唯一性的限制**

![image-20220623154615702](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623154615702.png)

 **2. 频繁作为 WHERE 查询条件的字段**

- 某个字段在SELECT语句的 WHERE 条件中经常被使用到，那么就需要给这个字段创建索引了。尤其是在 数据量大的情况下，创建普通索引就可以大幅提升数据查询的效率。

  比如student_info数据表（含100万条数据），假设我们想要查询 student_id=123110 的用户信息。

**3. 经常 GROUP BY 和 ORDER BY 的列**

- 索引就是让数据按照某种顺序进行存储或检索，因此当我们使用 GROUP BY 对数据进行分组查询，或者使用 ORDER BY 对数据进行排序的时候，就需要对分组或者排序的字段进行索引 。**如果待排序的列有多个，那么可以在这些列上建立组合索引** 。

**4. UPDATE、DELETE 的 WHERE 条件列**

- 对数据按照某个条件进行查询后再进行 UPDATE 或 DELETE 的操作，如果对 WHERE 字段创建了索引，就能大幅提升效率。原理是因为我们需要先根据 WHERE 条件列检索出来这条记录，然后再对它进行更新或删除。**如果进行更新的时候，更新的字段是非索引字段，提升的效率会更明显，这是因为非索引字段更新不需要对索引进行维护**

 **5.DISTINCT 字段需要创建索引**

- 有时候我们需要对某个字段进行去重，使用 DISTINCT，那么对这个字段创建索引，也会提升查询效率。

- 比如，我们想要查询课程表中不同的 student_id 都有哪些，如果我们没有对 student_id 创建索引，执行 SQL 语句：

  - ```sql
    SELECT DISTINCT(student_id) FROM `student_info`;
    ```

    运行结果（600637 条记录，运行时间 0.683s ）

    如果我们对 student_id 创建索引，再执行 SQL 语句：

    ```sql
    SELECT DISTINCT(student_id) FROM `student_info`;
    ```

    运行结果（600637 条记录，运行时间 0.010s ）

 **6. 多表 JOIN 连接操作时，创建索引注意事项**

- 首先， `连接表的数量尽量不要超过 3 张` ，因为每增加一张表就相当于增加了一次嵌套的循环，数量级增 长会非常快，严重影响查询的效率。
- 其次， `对 WHERE 条件创建索引` ，因为 WHERE 才是对数据条件的过滤。如果在数据量非常大的情况下， 没有 WHERE 条件过滤是非常可怕的。
- 最后， `对用于连接的字段创建索引` ，并且该字段在多张表中的 类型必须一致 。比如 course_id 在 student_info 表和 course 表中都为 int(11) 类型，而不能一个为 int 另一个为 varchar 类型。

**7. 使用列的类型小的创建索引**

![image-20220623175306282](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623175306282.png)

 **8. 使用字符串前缀创建索引**

![image-20220623175513439](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20220623175513439.png)

```sql
create table shop(address varchar(120) not null);
alter table shop add index(address(12));
```

**拓展：Alibaba《Java开发手册》**

【 强制 】**在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引**，根据实际文本 区分度决定索引长度。

 **9. 使用最频繁的列放到联合索引的左侧**

- 这样也可以较少的建立一些索引。同时，由于"最左前缀原则"，可以增加联合索引的使用率。

**10.在多个字段都要创建索引的情况下，联合索引优于单值索引**

==不适合创建索引的情况==

**1. 在where中使用不到的字段，不要设置索引**

- WHERE条件 (包括 GROUP BY、ORDER BY) 里用不到的字段不需要创建索引，**索引的价值是快速定位，如果起不到定位的字段通常是不需要创建索引的**。举个例子：
- 因为我们是按照 student_id 来进行检索的，所以不需要对其他字段创建索引，即使这些字段出现在SELECT字段中。

```sql
SELECT course_id, student_id, create_time
FROM student_info
WHERE student_id = 41251;
```

**2. 数据量小的表最好不要使用索引**

- 如果表记录太少，比如少于1000个，那么是不需要创建索引的。表记录太少，是否创建索引 `对查询效率的影响并不大`。甚至说，查询花费的时间可能比遍历索引的时间还要短，索引可能不会产生优化效果。

**3. 有大量重复数据的列上不要建立索引**

- 在条件表达式中经常用到的不同值较多的列上建立索引，**但字段中如果有大量重复数据，也不用创建索引**。比如在学生表的"性别"字段上只有“男”与“女”两个不同值，因此无须建立索引。如果建立索引，不但不会提高查询效率，反而会**严重降低数据更新速度**。

**4.避免对经常更新的表创建过多的索引**

- 第一层含义：频繁更新的字段不一定要创建索引。因为更新数据的时候，也需要更新索引，如果索引太多，在更新索引的时候也会造成负担，从而影响效率。
- 第二层含义：避免对经常更新的表创建过多的索引，并且索引中的列尽可能少。此时，虽然提高了查询速度，同时却降低更新表的速度。

**5.不建议用无序的值作为索引**

- 例如身份证、UUID(在索引比较时需要转为ASCII，并且插入时可能造成页分裂)、MD5、HASH、无序长字 符串等。

**6. 删除不再使用或者很少使用的索引**

- 表中的数据被大量更新，或者数据的使用方式被改变后，原有的一些索引可能不再需要。数据库管理员应当定期找出这些索引，将它们删除，从而减少索引对更新操作的影响。

### 范式

在关系型数据库中，关于数据表设计的基本原则、规则就称为范式。可以理解为，一张数据表的设计结构需要满足的某种设计标准的级别。要想设计一个结构合理的关系型数据库，必须满足一定的范式。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319152345158.png" alt="image-20250319152345158" style="zoom:80%;" />

#### 第一范式(1st NF)（列不可再分，确保每列保持原子性）

第一范式是最基本的范式。**如果数据库中的所有字段都是不可分解的原子值，就说明该数据库表满足了第一范式**。属性不可分。

第一范式的合理遵循需要根据系统的**实际需求**来定。比如使用“地址”就可以设计成数据库的字段，但是经常访问“城市”，因此将这个属性拆分成“省份”“城市”等多个部分，这样访问某一部分非常方便，这样设计才算满足了数据库的第一范式。

![image-20250319152515361](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319152515361.png)

#### 第二范式(2nd NF)（行可以唯一区分，确保表中的每列都和主键相关）

​	**第二范式需要确保数据库表中的每一列都和主键相关，而不能只与主键的某一部分相关**（主要针对联合主键而言，**对于联合主键必须跟两个主键都相关才行**）。

​	**如果知道主键的所有属性的值，就可以检索到任何元组（行）的任何属性的任何值**。

1NF告诉我们字段属性是原子性的，而2NF告诉我们一张表就是一个独立的对象，一张表只表达一个意思

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319152615732.png" alt="image-20250319152615732" style="zoom:80%;" />

#### 第三范式(3rd NF)

​	第三范式是在第二范式的基础上，确**保数据表中的每一个非主键字段都和主键字段直接相关**，也就是说，要求**数据表中的所有非主键字段不能依赖于其他非主键字段**。（即，不能存在非主属性A依赖于非主属性B，非主属性B依赖于主键C的情况，即存在"A-->B-->C"的决定关系）通俗地讲，该规则的意思是**所有非主键属性之间不能有依赖关系，必须相互独立**。

![image-20250319152803637](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319152803637.png)

![image-20250319152821858](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319152821858.png)

#### 范式优点与缺点

- 范式的优点：**数据的标准化有助于消除数据库中的数据冗余**，第三范式（3NF）通常被认为在性能、拓展性和数据完整性方面达到了最好的平衡。
- 范式的缺点：**范式的使用，可能降低查询的效率**。因为范式等级越高，设计出来的数据表就越多、越精细，数据的冗余度就越低，进行数据查询的时候就可能需要关联多张表，这不但代价昂贵，也可能使一些索引策略无效。

​	范式只是提出了设计的标准，实际上设计数据表时，未必一定要符合这些标准。开发中，我们会出现为了性能和读取效率违反范式化的原则，通过增加少量的冗余或重复的数据来提高数据库的读性能，减少关联查询，join表的次数，实现空间换取时间的目的。因此在实际的设计过程中要理论结合实际，灵活运用。

### ER模型

ER模型中有三个要素，分别是**实体、属性和关系**

- **实体**，可以看做是数据对象，往往对应于现实生活中的真实存在的个体。在ER模型中，用矩形来表示。实体分为两类，分别是强实体和弱实体。强实体是指不依赖于其他实体的实体；弱实体是指对另一个实体有很强的依赖关系的实体。
- **属性**，则是指实体的特性。比如超市的地址、联系电话、员工数等。在 ER 模型中用椭圆形来表示。
- **关系**，则是指实体之间的联系。比如超市把商品卖给顾客，就是一种超市与顾客之间的联系。在ER模型中用菱形来表示。 

​	注意：实体和属性不容易区分。这里提供一个原则：我们要从系统整体的角度出发去看，可以独立存在的是实体，不可再分的是属性。也就是说，属性不能包含其他属性。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/90fc257700d13d4e910d311664b335fe.png" alt="在这里插入图片描述" style="zoom:60%;" />

## MySQL事务基础

### 事务四大特性ACID

- ACID是事务的四大特性，在这四个特性中，**==一致性是目的，原子性、隔离性、持久性是手段==**。
- 数据库事务，其实就是数据库设计者为了方便起见，把==**需要保证原子性、隔离性、一致性和持久性的一个或多个数据库操作称为一个事务**==

#### 原子性(atomicity)

原子性是指**事务是一个不可分割的工作单位，要么全部提交，要么全部失败回滚**。eg：要么转账成功，要么转账失败，不存在中间状态

#### 一致性(consistency)

​	一致性是指**事务执行前后，数据从一个合法性状态变换到另外一个合法性状态**。那什么是合法的数据状态呢?满足预定的约束的状态就叫做合法的状态。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319155713674.png" alt="image-20250319155713674" style="zoom:80%;" />

#### 隔离型（isolation）

​	事务的隔离性是指一**个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰**。

【eg】如果无法保证隔离性会怎么样？假设A账户有200元，B账户0元。A账户往B账户转账两次，每次金额为50元，分别在两个事务中执行。如果无法保证隔离性，会出现下面的情形：B少了50块

```sql
#案例: AA用户给BB用户转账100
UPDATE accounts SET money = money - 50 WHERE NAME = 'AA';
UPDATE accounts SET money = money + 50 WHERE NAME = 'BB';
```

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/ad641141d9069b9948208dd461adbd17.png" alt="在这里插入图片描述" style="zoom:50%;" />

#### 持久性（durability）

​	持久性是指**一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来的其他操作和数据库故障不应该对其有任何影响**。

​	持久性是通过<font color = '#8D0101'>**redo事务日志**</font>来保证的。日志包括了**重做日志redo log和回滚日志undo log**。当我们**通过事务对数据进行修改的时候，首先会将数据库的变化信息记录到重做日志redo log中，然后再对数据库中对应的行进行修改**。这样做的好处是，即使数据库系统崩溃，数据库重启后也能找到没有更新到数据库系统中的重做日志，重新执行，从而使事务具有持久性。

#### 一致性和原子性的区别

原子性：一个事务内的操作，要么同时成功，要么同时失败

一致性：一个事务必须使数据库从一个一致性状态变换到另一个一致性状态

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps39.jpg) 

从这段话来看，所谓一致性，即，从实际的业务逻辑上来说，最终结果是对的、是跟程序员的所期望的结果完全符合

### 如何保证四大特性实现

**<u>（1）原子性</u>**

- **<font color = '#8D0101'>利用undo log(回滚日志)</font>**，是实现原子性的关键，当**事务回滚时能够撤销所有已经成功执行的sql语句，需要记录你要回滚的相应日志信息**。
  - 例如：当你delete一条数据的时候，就需要记录这条数据的信息，回滚的时候，insert这条旧数据；当你update一条数据的时候，就需要记录之前的旧值，回滚的时候，根据旧值执行update操作；当insert一条数据的时候，就需要这条记录的主键，回滚的时候，根据主键执行delete操作。

- **undo log记录了这些回滚需要的信息，当事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子**。

**<u>（2）一致性</u>**

分为两个层面。

- **从数据库层面，数据库通过原子性、隔离性、持久性来保证一致性**。也就是说ACID四大特性之中，C(一致性)是目的，A(原子性)、I(隔离性)、D(持久性)是手段，（跟shk讲的不太一样）是为了保证一致性，数据库提供的手段。数据库必须要实现AID三大特性，才有可能实现一致性。例如，原子性无法保证，显然一致性也无法保证。但是，如果你在事务里故意写出违反约束的代码，一致性还是无法保证的。
- 还必须从应用层角度考虑。**从应用层面，通过代码判断数据库数据是否有效，然后决定回滚还是提交数据！**

**<u>（3）隔离性</u>**

- **<font color = '#8D0101'>利用的是锁或MVCC机制(Multi Version Concurrency Control,多版本并发控制)。</font>**
- 一个行记录数据有多个版本的快照数据，这些快照数据在undo log中。如果一个事务读取的行正在做DELELE或者UPDATE操作，读取操作不会等行上的锁释放，而是读取该行的快照版本。由于**MVCC机制在可重复读(Repeateable Read)和读已提交(ReadCommited)的MVCC表现形式不同**。
  - 但是有一点说明一下，**在事务隔离级别为读已提交(Read Commited)时，一个事务能够读到另一个事务已经提交的数据，是不满足隔离性的。但是当事务隔离级别为可重复读(Repeateable Read)中，是满足隔离性的**。

**<u>（4）持久性</u>**

- **<font color = '#8D0101'>利用redo log（重做日志）</font>**
- 当做数据修改的时候，不仅在内存中操作，还会在redo log中记录这次操作。当事务提交的时候，会将redo log日志进行刷盘(redo log一部分在内存中，一部分在磁盘上)。当数据库宕机重启的时候，会将redo log中的内容恢复到数据库中，再根据undo log和binlog内容决定回滚数据还是提交数据。
- **使用redolog的好处**：redo log体积小，毕竟只记录了哪一页修改了什么，因此体积小，刷盘快。redo log是一直往末尾进行追加，属于顺序IO。效率显然比随机IO来的快。(日志一般都是顺序IO)

### 并发事务带来的四个问题

​	在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务(多个用户对同一数据进行操作)。并发虽然是必须的，但可能会导致以下的问题。

#### 脏写(Dirty Write)

对于两个事务SessionA、SessionB，如果<font color = '#8D0101'>**事务SessionA修改了另一个未提交事务SessionB修改过的数据**</font>，那就意味着发生了脏写。

![image-20250319162803136](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319162803136.png)

![image-20250319162807086](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319162807086.png)

#### 脏读（Dirty Read）

​	**<font color = '#8D0101'>如果一个事务「读到」了另一个「未提交事务修改过的数据」，就意味着发生了「脏读」现象。</font>**

​	当SessionA正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时SessionB也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么SessionB读到的这个数据是“脏数据”，依据“脏数据"所做的操作可能是不正确的：之后若SessionA回滚，SessionB读取的内容就是临时且无效的。

#### 不可重复读（Non-Repeatable Read）

​	**<font color = '#8D0101'>在一个事务内多次读取同一个数据，如果出现前后两次读到的数据不一样的情况，就意味着发生了「不可重复读」现象。</font>**

​	指在一个事务内多次读同一数据。对于两个事务SessionA、SessionB。SessionA读取了一个字段，然后SessionB更新了该字段。之后Session A再次读取同一个字段，值就不同了。那就意味着发生了不可重复读。

#### 幻读（Phantom read）

​	**<font color = '#8D0101'>幻读指一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行</font>**

​	幻读与不可重复读类似。对于两个事务Session A、Session B,Session A从一个表中读取了一个字段,然后Session B在该表中插入了一些新的行。之后,如果SessionA再次读取同一个表,就会多出几行。那就意味着发生了幻读。

![在这里插入图片描述](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/e05a735bb05630cecb26ae7a62fc4ed2.png)

> [!CAUTION]
>
> 注意1:
>
> - 有的同学会有疑问，那如果Session B中删除了一些符合studentno > 的记录而不是插入新记录，那SessionA之后再根据studentno > 0的条件读取的记录变少了，这种现象算不算幻读呢?这种现象不属于幻读，幻读强调的是一个事务按照某个相同条件多次读取记录时，后读取时读到了之前没有读到的记录。
>
> 注意2:
>
> - 那对于先前已经读到的记录，之后又读取不到这种情况，算啥呢?这**<font color = '#8D0101'>相当于对每一条记录都发生了不可重复读的现象。幻读只是重点强调了读取到了之前读取没有获取到的记录</font>**

### 事务隔离级别

#### 读未提交、读已提交、可重复读、可串行化

![image-20250319163642820](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319163642820.png)

我们愿意舍弃一部分隔离性来换取一部分性能在这里就体现在：设立一些隔离级别，**隔离级别越低，并发问题发生的越多**。SQL标准中设立了4个隔离级别：

- **<font color = '#8D0101'>READ UNCOMMITTED：读未提交</font>，**在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。**不能避免脏读、不可重复读、幻读**。

- **<font color = '#8D0101'>READ COMMITTED：读已提交</font>**，它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。**可以避免脏读，但不可重复读、幻读问题仍然存在**。**读提交隔离级别是在每次读取数据时，都会生成一个新的Read View**

- **<font color = '#8D0101'>REPEATABLE READ：可重复读</font>**，事务A在读到一条数据之后，此时事务B对该数据进行了修改并提交，那么事务A再读该数据，读到的还是原来的内容。**可以避免脏读、不可重复读，但幻读问题仍然存在（innodb可以避免幻读）**。这是==MySQL的默认隔离级别==。**可重复读隔离级别是启动事务时生成一个Read View，然后整个事务期间都在用这个Read View**。

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319163930568.png" alt="image-20250319163930568" style="zoom:80%;" />

- **<font color = '#8D0101'>SERIALIZABLE：可串行化</font>**，确保事务可以从一个表中读取相同的行。在这个事务持续期间，禁止其他事务对该表执行插入、更新和删除操作。所有的并发问题都可以避免，但性能十分低下。能避免脏读、不可重复读和幻读。

> [!CAUTION]
>
> - 与SQL标准不同的地方在于**InnoDB存储引擎**在**REPEATABLE-READ(可重复读)事务隔离级别下使用的是Next-Key Lock算法实现了行锁，因此可以避免幻读的产生**，这与其他数据库系统(如SQLServer)是不同的。
> - 所以说**InnoDB存储引擎的默认支持的隔离级别是REPEATABLE-READ(可重读)已经可以完全保证事务的隔离性要求，即达到了SQL标准的SERIALIZABLE(可串行化)隔离级别。**因为隔离级别越低，事务请求的锁越少，所以大部分数据库系统的隔离级别都是READ-COMMITTED(读取提交内容)，但是你要知道的是InnoDB存储引擎默认使用REPEATABLE-READ(可重读)并不会有任何性能损失。
> - InnoDB存储引擎在分布式事务的情况下一般会用到SERIALIZABLE(可串行化)隔离级别。

#### 幻读有什么问题，MySQL如何解决幻读（快照读和当前读）

**【快照读和当前读】**

**快照读**

- MVCC 的 SELECT 操作是快照中的数据，不需要进行加锁操作。


**当前读**

- 所谓当前读就是，**读取的是最新版本的数据,并且对读取的记录行加锁，阻塞其他事务同时改动相同记录，避免出现安全问题**。
  - MVCC 会对数据库进行修改的操作（INSERT、UPDATE、DELETE） 需要进行加锁操作，从而读取最新的数据。
  - 可以看到 **MVCC 并不是完全不用加锁，而只是避免了 SELECT 的加锁操作**。

**【幻读】**

- 幻读就是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。
- 幻读的后果就是**数据库中的数据和binlog的执行结果会不一致**，其原因就在于，我们无法阻止新插入的数据。就是说，我们在给扫描到的行加锁的时候，你等会要插入的行还不存在，也就没法对他进行加锁，那么这个新插入的数据，可能在主库中是这个样子，从库执行完binlog后其实是会被修改的

**==【MySQL是如何解决的】==**

​	==MVCC+next-key lock：在快照读时使用MVCC，在当前读时使用next-key lock，保证并发安全==

- **幻读问题在"当前读"下才会出现**。所谓当前读就是，读取的是最新版本的数据,并且对读取的记录加锁，阻塞其他事务同时改动相同记录，避免出现安全问题。与之对应的，快照读，读取的是快照中的数据，不需要进行加锁。
- **读取已提交和可重复读**这俩隔离级别下的**普通select操作就是快照读**。其实就是MVCC机制，或者说，**<font color = '#8D0101'>在快照读下，采用MVCC机制解决幻读</font>**。
- **对于当前读这种情况**，如果不加锁的话(其实当前读是会加锁的)，由于无法阻止新插入的数据，所以无法解决幻读问题，所以，我们考虑，**不仅对扫描到的行进行加锁，还对行之间的间隙进行加锁，这样就能杜绝新数据的插入和更新**。这个其实就是记录锁Record Lock和间隙锁Gap Lock，也被称为**<font color = '#8D0101'>临键锁Next-Key Lock</font>**。
- **临键锁只在可重复读也就是InnoDB的默认隔离级别下生效**。也可以采用更高的可串行化隔离级别，所有的操作都是串行执行的，可以直接杜绝幻读问题。

​	**在可重复读（RR）的隔离级别下，InnoDB存储引擎使用Next-Lock Lock算法，避免幻读的产生**。但是如Oracle数据库，可能需要在SERIALIZABLE的事务隔离级别下才能解决幻读问题

### 事务的状态

**活动的（active）**
事务对应的数据库操作正在执行过程中时，就说该事务处在 活动的 状态。

**部分提交的（partially committed）**
当事务中的最后一个操作执行完成，但由于操作都在内存中执行，所造成的影响并 没有刷新到磁盘时，我们就说该事务处在 部分提交的 状态。

**失败的（failed）**
当事务处在 活动的 或者 部分提交的 状态时，可能遇到了某些错误（数据库自身的错误、操作系统错误或者直接断电等）而无法继续执行，或者人为的停止当前事务的执行，就说该事务处在失败的状态

**中止的（aborted）**
如果事务执行了一部分而变为 失败的 状态，那么就需要把已经修改的事务中的操作还原到事务执行前的状态。换句话说，就是要撤销失败事务对当前数据库造成的影响。把这个撤销的过程称之为 回滚 。当 回滚 操作执行完毕时，也就是数据库恢复到了执行事务之前的状态，就说该事务处在了 中止的 状态。
举例：

```sql
UPDATE accounts SET money = money - 50 WHERE NAME = 'AA';
UPDATE accounts SET money = money + 50 WHERE NAME = 'BB';
```

**提交的（committed）**
当一个处在 部分提交的 状态的事务将修改过的数据都 同步到磁盘 上之后，就可以说该事务处在了 提交的 状态。

![image-20250319161432604](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319161432604.png)



## 多版本并发控制 MVCC(Multiversion Concurrency Control)

​	**<font color = '#8D0101'>这种通过「版本链」来控制并发事务访问同一个记录时的行为就叫MVCC（多版本并发控制）</font>**

​	**<font color = '#8D0101'>MVCC的实现依赖于：隐藏字段、Undo Log、Read View</font>**

​	MVCC是通过数据行的多个版本管理来实现数据库的并发控制，是MySQL的InnoDB存储引擎实现隔离级别的一种具体方式，用于**实现提交读和可重复读这两种隔离级别**。而未提交读隔离级别总是读取最新的数据行，要求很低，无需使用MVCC。可串行化隔离级别需要对所有读取的行都加锁，单纯使用MVCC无法实现。

- 加锁能解决多个事务同时执行时出现的并发一致性问题。在实际场景中读操作往往多于写操作，因此又引入了**读写锁来避免不必要的加锁操作，例如读和读没有互斥关系**。
- **读写锁中读和写操作仍然是互斥的**，而**<font color = '#8D0101'>MVCC利用了多版本的思想，写操作更新最新的版本快照，而读操作去读旧版本快照，没有互斥关系</font>**，这一点和CopyOnWrite(写入时复制思想)类似，**在MVCC中事务的修改操作（DELETE、INSERT、UPDATE）会为数据行新增一个版本快照**。脏读和不可重复读最根本的原因是**事务读取到其它事务未提交的修改**。在事务进行读取操作时，为了解决脏读和不可重复读问题，**MVCC规定只能读取已经提交的快照。当然一个事务可以读取自身未提交的快照，这不算是脏读**。

### MVCC**整体操作流程**

1.首先获取事务自己的版本号，也就是事务ID(Transaction ID)；

2.获取ReadView；

3.查询得到的数据，然后与ReadView中的事务版本号进行比较；

4.如果不符合ReadView规则，就需要从Undo Log中获取历史快照；

5.最后返回符合规则的数据。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps40.jpg) 

### 快照读(一致性读)与当前读

​	MVCC在MySQL InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到**即使有读写冲突时，也能做到不加锁，非阻塞并发读**，而这个读指的就是快照读，而非当前读。 **当前读实际上是一种加锁的操作，是悲观锁的实现。而MVCC本质是采用乐观锁思想的一种方式**。

#### 快照读(一致性读)

**快照读又叫一致性读，读取的是快照数据**，**不需要进行加锁操作。**不加锁的简单的SELECT都属于快照读，即不加锁的非阻塞读

- 之所以出现快照读的情况，是基于提高并发性能的考虑，**快照读的实现是基于MVCC，它在很多情况下，避免了加锁操作，降低了开销**。
- 既然是基于多版本，那么快照读**一般读到的并一定是数据的最新版本，而是之前的历史版本**。
- **快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读。**

#### 当前读

​	**当前读读取的是记录的最新版本（最新数据，而不是历史版本的数据）**，读取时还要保证其他并发事务不能修改当前记录，**会对读到的记录加上next-key lock锁**。加锁的SELECT，或者对数据进行增删改都会进行当前读。

**【eg】**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319170406602.png" alt="image-20250319170406602" style="zoom:80%;" />

### 隐藏字段、Undo log版本链

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319170647559.png" alt="image-20250319170647559" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319170658562.png" alt="image-20250319170658562" style="zoom:80%;" />

- **每次对记录进行改动，都会记录一条undo日志**，每条undo日志也都有一个roll_pointer属性（INSERT操作对应的undo日志没有该属性，因为该记录并没有更早的版本），可以将这些undo日志都连起来，**串成一个链表**：

  ![image-20250319171408851](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319171408851.png)

- 对该记录每次更新后，都会将旧值放到一条undo日志中，就算是该记录的一个旧版本，随着更新次数的增多，所有的版本都会被**roll_pointe**r属性连接成一个链表，**我们把这个链表称之为版本链，版本链的头节点就是当前记录最新的值**。

- 每个版本中还包含生成该版本时对应的事务id。

### MVCC实现原理之ReadView

#### 概述

​	在MVCC机制中，多个事务对同一个行记录进行更新会产生多个历史快照，这些历史快照保存在Undo Log里。如果一个事务想要查询这个行记录，需要读取哪个版本的行记录呢?这时就需要用到ReadView了，它解决了行的可见性问题

​	**ReadView就是事务在使用MVCC机制进行快照读操作时产生的读视图**。当事务启动时，会生成数据库系统当前的一个快照，**InnoDB为每个事务构造了一个数组，用来记录并维护系统当前活跃事务的ID（“活跃"指的就是，启动了但还没提交)**

#### 设计思路（READ COMMITTD和REPEATABLE READ的read view不同点）

- 使用READ COMMITTED和REPEATABLE READ隔离级别的事务，都必须保证**读到已经提交了的事务修改过的记录**。假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，核心问题就是需要**<font color = '#8D0101'>判断一下版本链中的哪个版本是当前事务可见的，这是ReadView要解决的主要问题</font>**。

> [!CAUTION]
>
> - **READ COMMITTD**在**每一次**进行普通SELECT操作前都会生成一个ReadView，意味着，**事务期间的多次读取同一条数据，前后两次读的数据可能会出现不一致，因为可能这期间另外一个事务修改了该记录，并提交了事务**
> - **REPEATABLE READ**只在**第一次**进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复使用这个ReadView就好了，这样就**保证了在事务期间读到的数据都是事务启动前的记录**。

#### ReadView包含的4个内容

- creator_trx_id ，创建这个 Read View 的事务 ID。

  - 说明：只有在对表中的记录做改动时（执行INSERT、DELETE、UPDATE这些语句时）才会为事务分配事务id，否则在一个只读事务中的事务id值都默认为0。

- trx_ids ，表示在生成ReadView时当前系统中活跃的读写事务的 事务id列表 。
- up_limit_id ，活跃的事务中最小的事务 ID。
- low_limit_id ，表示生成ReadView时系统中应该分配给下一个事务的 id 值。low_limit_id 是系统最大的事务id值，这里要注意是系统中的事务id，需要区别于正在活跃的事
  - low_limit_id并不是trx_ids中的最大值，事务id是递增分配的。比如，现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，trx_ids就包括1和2，up_limit_id的值就是1，low_limit_id的值就是4。

#### ReadView的规则

对当前事务来说，按照以下规则**从最新的版本开始遍历，获取对应的版本记录。**

1、被访问的trx_id与readview中的creator_trx_id相同，表示当前事务在访问自己修改的记录，可见，返回;

2、被访问的trx_id小于min_trx_id，表示这个版本的记录是在**创建Read View前已经提交的事务生成的**，所以该版本的记录对当前事务可见;

3、被访问的trx_id大于等于max_trx_id，表示这个版本的记录是在**创建Read View后才启动的事务生成的**，所以该版本的记录对当前事务不可见；

4、被访问的trx_id在min_trx_id和max_trx_id之间，判断是否在trx_ids中

- 如果在，则说明生成readview时，该版本事务未提交，该版本不可见，那么**就沿着undo log链条往下找旧版本的记录，直到找到trx_id「小于」事务B的Read View中的min_trx_id值的第一条记录**;
- 如果不在，表示生成该版本read view记录的活跃事务已经被提交，所以该版本的记录对当前事务可见

## MySQL锁

###  MySQL并发事务访问相同记录

#### 三种情况：读-读、写-写、读-写

<u>**1）读-读情况**</u>

- 读-读情况，即并发事务相继读取相同的记录。读取操作本身不会对记录有任何影响，并不会引起什么问题，所以允许这种情况的发生。

<u>**2）写-写情况**</u>

- 写-写 情况，即并发事务相继对相同的记录做出改动。

- 在这种情况下会发生 **脏写** 的问题，**任何一种隔离级别都不允许这种问题的发生**。所以**在多个未提交事务相继对一条记录做改动时，需要让它们 排队执行 ，这个排队的过程其实是通过 锁 来实现的**。这个所谓的锁其实是一个 内存中的结构 ，在事务执行前本来是没有锁的，也就是说一开始是没有 锁结构 和记录进行关联的,如图所示：

  ![image-20250319172858901](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319172858901.png)

**不加锁**：意思就是不需要在内存中生成对应的锁结构，可以直接执行操作。

**获取锁成功**，或者加锁成功：意思就是在内存中生成了对应的锁结构，而且锁结构的is_waiting属性为false，也就是事务可以继续执行操作。

**获取锁失败**，或者加锁失败，或者没有获取到锁：意思就是在内存中生成了对应的锁结构，不过锁结构的is_waiting属性为true，也就是事务需要等待，不可以继续执行操作。

<u>**3）读-写或写-读情况**</u>

读-写 或 写-读 ，即一个事务进行读取操作，另一个进行改动操作。这种情况下可能发生 **脏读 、 不可重复读 、 幻读** 的问题。
各个数据库厂商对 SQL标准 的支持都可能不一样。比如**MySQL在 REPEATABLE READ 隔离级别上就已经解决了 幻读 问题**。

#### 并发问题的解决方案

**<u>【方案一：读操作利用多版本并发控制，写操作进行 加锁 】</u>**
	所谓的MVCC，就是生成一个ReadView，通过ReadView找到符合条件的记录版本（(历史版本由undo日志构建)。

- **查询语句只能读到在生成ReadView之前已提交事务所做的更改**，在生成ReadView之前未提交的事务或者之后才开启的事务所做的更改是看不到的。
- 而**写操作肯定针对的是最新版本的记录**，读记录的历史版本和改动记录的最新版本本身并不冲突，也就是采用MVCC时，读-写操作并不冲突

> [!CAUTION]
>
> - **READ COMMITTD**在**每一次**进行普通SELECT操作前都会生成一个ReadView，意味着，**事务期间的多次读取同一条数据，前后两次读的数据可能会出现不一致，因为可能这期间另外一个事务修改了该记录，并提交了事务**
> - **REPEATABLE READ**只在**第一次**进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复使用这个ReadView就好了，这样就**保证了在事务期间读到的数据都是事务启动前的记录**。

<u>**【方案二:读、写操作都采用加锁的方式】**</u>

​	**脏读**的产生是因为当前事务读取了另一个未提交事务写的一条记录，如果**另一个事务在写记录的时候就给这条记录加锁，那么当前事务就无法继续读取该记录了，所以也就不会有脏读问题的产生了**。

​	**不可重复读**的产生是因为当前事务先读取一条记录，另外一个事务对该记录做了改动之后并提交之后，当前事务再次读取时会获得不同的值，如果在**当前事务读取记录时就给该记录加锁，那么另一个事务就无法修改该记录，自然也不会发生不可重复读了**

​	**幻读**问题的产生是因为当前事务读取了一个范围的记录，然后另外的事务向该范围内插入了新记录，当前事务再次读取该范围的记录时发现了新插入的新记录。采用加锁的方式解决幻读问题就有一些麻烦，因为当前事务在第一次读取记录时幻影记录并不存在，所以读取的时候加锁就有点尴尬（因为你并不知道给谁加锁)


**<u>【小结对比发现】</u>**

- 采用 MVCC 方式的话， 读-写 操作彼此并不冲突， 性能更高 。
- 采用 加锁 方式的话， 读-写 操作彼此需要 排队执行 ，影响性能。

​	一般情况下当然愿意采用 MVCC 来解决 读-写 操作并发执行的问题，但是业务在某些特殊情况下，要求必须采用 加锁 的方式执行。下面就讲解下MySQL中不同类别的

### 锁的分类

MyISAM和InnoDB存储引擎使用的锁:

- **MyISAM采用表级锁(table-level locking)。**
- **InnoDB支持行级锁(row-level locking)和表级锁,默认为行级锁**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/94e4294c6808e333bf1238f0b6d23aa7.png" alt="请添加图片描述" style="zoom:67%;" />

### 读锁(共享锁/S锁)、写锁(排他锁/X锁)：从数据操作的类型划分

​	对于数据库中并发事务的读-读情况并不会引起什么问题。对于写-写、读-写或写-读这些情况可能会引起一些问题，需要使用MVCC或者加锁的方式来解决它们。在使用加锁的方式解决问题时，由于既要允许读-读情况不受影响，又要使写-写、读-写或写-读情况中的操作相互阻塞，所以MySQL实现一个由两种类型的锁组成的锁系统来解决。**这两种类型的锁通常被称为共享锁(Shared Lock，S Lock)和排他锁(Exclusive Lock，X Lock)，也叫读锁(readlock)和写锁(write lock)**

- **读锁**：也称为**共享锁、英文用S表示**。针对同一份数据，多个事务的读操作可以同时进行而不会互相影响，相互不阻塞的。
- **写锁**：也称为**排他锁、英文用X表示**。当前写操作没有完成前，它会阻断其他写锁和读锁。这样就能确保在给定的时间里，只有一个事务能执行写入，并防止其他用户读取正在写入的同一资源。

**需要注意的是对于InnoDB引擎来说，读锁和写锁可以加在表上，也可以加在行上。**

![image-20250319174123418](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319174123418.png)

#### 锁定读

​	在采用加锁方式解决脏读、不可重复读、幻读这些问题时，读取一条记录时需要获取该记录的S锁，其实是不严谨的，有时候需要在读取记录时就获取记录的X锁，来禁止别的事务读写该记录，为此MySQL提出了两种比较特殊的SELECT语句格式:

**【对读取的记录加S锁(IN SHARE MODE)】**

在普通的SELECT语句后边加LOCK IN SHARE MODE

- 如果当前事务执行了该语句，那么它会**为读取到的记录加S锁，**这样**允许别的事务继续获取这些记录的S锁**（比方说别的事务也使用SELECT …LOCK IN SHARE MODE语句来读取这些记录)，
- 但是**不能获取这些记录的X锁**(比如使用SELECT … FOR UPDATE语句来读取这些记录，或者直接修改这些记录)。
  - 如果别的事务**想要获取这些记录的X锁，那么它们会阻塞，直到当前事务提交之后将这些记录上的S锁释放掉**

```sql
SELECT ... LOCK IN SHARE MODE; 
#或
SELECT ... FOR SHARE;#(8.0新增语法)
```

【**对读取的记录加X锁(FOR UPDATE)**】

在普通的SELECT语句后边加FOR UPDATE，

- 如果当前事务执行了该语句，那么它会**为读取到的记录加X锁**，这样既**不允许别的事务获取这些记录的S锁**(比方说别的事务使用SELECT … LOCK IN SHARE MODE语句来读取这些记录)，
- 也**不允许获取这些记录的X锁**(比如使用SELECT … FOR UPDATE语句来读取这些记录，或者直接修改这些记录)。如果别的事务想要获取这些记录的S锁或者X锁，那么它们会阻塞，直到当前事务提交之后将这些记录上的X锁释放掉

```sql
SELECT ... FOR UPDATE;
```

#### 写操作

==包括DELETE、UPDATE、INSERT三种==

【**DELETE**】

- 对一条记录做DELETE操作的过程其实是**先在B+树中定位到这条记录的位置**，然后**获取这条记录的X锁**，再执行delete mark.操作。也可以把这个定位待删除记录在B+树中位置的过程看成是一个获取X锁的锁定读。

**【UPDATE】**

- **情况1:** **未修改该记录的键值，并且被更新的列占用的存储空间在修改前后未发生变化**。 则先在B+树中定位到这条记录的位置，然后再获取一下记录的X锁，最后在原记录的位置进行修改操作。也可以把这个定位待修改记录在B+树中位置的过程看成是一个获取X锁的锁定读。
- **情况2∶** **未修改该记录的键值，并且至少有一个被更新的列占用的存储空间在修改前后发生变化**。 则先在B+树中定位到这条记录的位置，然后获取一下记录的X锁，将该记录彻底删除掉（就是把记录彻底移入垃圾链表)，最后再插入一条新记录。这个定位待修改记录在B+树中位置的过程看成是一个获取X锁的锁定读，新插入的记录由INSERT操作提供的隐式锁进行保护。
- **情况3∶** **修改了该记录的键值，则相当于在原记录上做DELETE操作之后再来一次INSERT操作**，加锁操作就需要按照DELETE和INSERT的规则进行了。

**【INSERT】**

- 一般情况下，新插入一条记录的操作并不加锁,通过一种称之为隐式锁的结构来保护这条新插入的记录在本事务提交前不被别的事务访问。

### 表级锁、页级锁、行锁：从数据操作的粒度划分

#### 表锁（Table Lock）（重点：意向锁）

​	表级锁:MySQL中锁定粒度最大的一种锁，对当前操作的整张表加锁，**实现简单，资源消耗也比较少，加锁快，不会出现死锁。其锁定度最大，触发锁冲突的概率最高，并发度最低，MyISAM和InnoDB引擎都支持表级锁。**

**InnoDB存储引擎的表级锁有四种：**

##### X、S锁

一般情况下，不会使用InnoDB存储引擎提供的表级别的S锁和X锁。只会在一些特殊情况下，比方说崩溃恢复过程中用到。 

MyISAM在执行查询语句（SELECT）前，会给涉及的所有表加读锁，在执行增删改操作前，会给涉及的表加写锁。**InnoDB存储引擎是不会为这个表添加表级别的读锁或者写锁的。**

##### ==意向锁==

InnoDB支持多粒度锁（multiple granularity locking），它**<font color = '#8D0101'>允许行级锁与表级锁共存</font>**，而意向锁就是其中的一种表锁。

**意向锁的目的是为了<font color = '#8D0101'>快速判断表里是否有记录被加锁</font>**

- 意向锁的存在是为了**协调行锁和表锁的关系**，支持多粒度（表锁与行锁）的锁并存。**意向锁是一种不与行级锁冲突的表级锁**，这一点非常重要。
- **意向锁之间也不会发生冲突**，只会和**表级的S锁（lock tables ... read）和表级的X锁（lock tables ... write）发生冲突**。(但IS与S兼容)
- 表明“某个事务正在某些行持有了锁或该事务准备去持有锁”
- 意向锁在**保证并发性的前提下，实现了行锁和表锁共存且满足事务隔离性的要求**。

**<u>1）意向锁分为两种：</u>**

- **意向共享锁**（intention shared lock,IS）：事务有意向对表中的某些行加共享锁（S锁）

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps44.jpg) 

- **意向排他锁**（intention exclusive lock, IX）：事务有意向对表中的某些行加排他锁（X锁）

![image-20251018231919114](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018231919114.png)

<u>**2）意向锁要解决的问题**</u>

​	**如果没有「意向锁」，那么加「独占表锁」时，就需要遍历表里所有记录，查看是否有记录存在独占锁，这样效率会很慢。**

​	**那么有了「意向锁」，由于在对记录加独占锁前，先会加上表级别的意向独占锁，那么在加「独占表锁」时，直接查该表是否有意向独占锁，如果有就意味着表里已经有记录被加了独占锁，这样就不用去遍历表里的记录,大大提高了效率。**

- **<font color = '#8D0101'>如果事务想要获取数据表中某些记录的共享锁，就需要在数据表上添加意向共享锁</font>**
- **<font color = '#8D0101'>如果事务想要获取数据表中某些记录的排它锁，就需要在数据表上添加意向排他锁</font>**

**eg：**

![image-20250319204552623](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319204552623.png)

<u>**3）意向锁的兼容互斥性：**</u>

- **意向锁不会与行级的共享/排他锁互斥**
- 意向锁是标记锁，任意IX/IS锁都是兼容的，它们只是**想要对表加锁**，而**不是真正加锁**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319204617310.png" alt="image-20250319204617310" style="zoom:73%;" />

注意这里的X和S锁是表锁，意向锁是不会跟行级锁互斥的

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps46.jpg) 

#### 行锁（记录锁、间隙锁、临键锁、插入意向锁）

​	行级锁:MySQL中锁定**粒度最小的一种锁**，只针对当前操作的行进行加锁。**行级锁能大大减少数据库操作的冲突。其加锁粒度最小，并发度高，但加锁的开销也最大，加锁慢，会出现死锁**。加了索引之后默认的就是加了行锁

- **<font color = '#8D0101'>==行锁在InnoDB中是基于索引实现的==</font>**，所以一旦某个加锁操作没有使用索引，那么该锁就会退化为表锁
- 除了直接在主键索引加锁，我们还可以通过辅助索引找到相应主键索引后再加锁

##### Record lock 记录锁：单个行记录上的锁

记录锁也就是仅仅把一条记录锁上，官方的类型名称为：LOCK_REC_NOT_GAP。

![image-20250319205431351](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319205431351.png)

记录锁是有S锁和X锁之分的，称之为**S型记录锁和X型记录锁**。

- 当一个事务获取了一条记录的S型记录锁后，其他事务也可以继续获取该记录的S型记录锁，但不可以继续获取X型记录锁；

- 当一个事务获取了一条记录的X型记录锁后，其他事务既不可以继续获取该记录的S型记录锁，也不可以继续获取X型记录锁。

##### Gap lock间隙锁：锁定一个范围，针对insert操作，不包括记录本身

​	**间隙锁本质上是用于阻止其他事务在该间隙内插入新记录**，而自身事务是允许在该间隙内插入数据的，也就是说**间隙锁的应用场景包括并发读取、并发更新、并发删除和并发插入**

​	这间隙锁在本质上是不区分共享间隙锁或互斥间隙锁的，而且**<font color = '#8D0101'>间隙锁是不互斥的</font>**，即**两个事务可以同时持有包含共同间隙的间隙锁：**这里的共同间隙包括两种场景：其一是两个间隙锁的间隙区间完全一样；二是一个间隙锁包含的间隙区间是另一个间隙锁包含间隙区间的子集。

- MySQL在REPEATABLE READ隔离级别下是可以解决幻读问题的，解决方案有两种，可以使用**MVCC方案(快照读情况)解决，也可以采用加锁方案(当前读情况)解决**。
- 但是在使用加锁方案解决时有个大问题，就是事务在第一次执行读取操作时，那些幻影记录尚不存在，我们无法给这些幻影记录加上记录锁。
- InnoDB提出了一种称之为Gap Locks的锁，官方的类型名称为：LOCK_GAP，我们可以简称为gap锁。

![image-20250319205824394](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319205824394.png)

==不能在(3,8)区间内插入数据，两边都是开区间==

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319205912867.png" alt="image-20250319205912867" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319205925654.png" alt="image-20250319205925654" style="zoom:80%;" />

##### 临键锁（Next-key Locks）：record+gap锁定一个范围，包含记录本身

​	有时候**既想 锁住某条记录 ，又想 阻止 其他事务在该记录前边的 间隙插入新记录** ，所以InnoDB就提出了一种称之为 Next-Key Locks 的锁，官方的类型名称为： LOCK_ORDINARY ，我们也可以简称为next-key锁 。**Next-Key Locks是在存储引擎 innodb 、事务级别在 可重复读 的情况下使用的数据库锁，innodb默认的锁就是Next-Key locks**。
![image-20250319210403271](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319210403271.png)

![image-20250319210418106](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319210418106.png)

- 它是**Record Locks和Gap Locks的结合**，不仅锁定一个记录上的索引，也锁定索引之间的间隙。它锁定一个<font color = '#8D0101'>**前开后闭区间**</font>，例如一个索引包含以下值:10，11，13 and 20，那么就需要锁定以下区间：( -00, 10]( 10，11]( 11，13]( 13，20](20，+00)
- Next-Key Locks是在**存储引擎innodb、事务级别在可重复读(RR)**的情况下使用的数据库锁，**innodb默认的锁就是Next-Key lock,即对记录加锁时，加锁的基本单位是next-key lock**

> [!WARNING]
>
> **注意**:
>
> - 对于**唯一键值的锁定**，Next-Key Lock降级为Record Lock仅存在于查询所有的唯一索引列。若唯一索引列由多个列组成，而查询仅是查找多个唯一索引列中的其中一个，那么查询其实是range类型查询，不是point类型的查询,故InnoDB依然使用Next-Key Lock进行锁定。

##### 插入意向锁（Insert Intention Locks）(一定是有gap锁或者临键锁存在才有这个插入意向锁)

**插入意向锁是一种特殊的间隙锁，不是意向锁，但不同于间隙锁的是，该锁<font color = '#8D0101'>只用于并发插入操作</font>**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319211513365.png" alt="image-20250319211513365" style="zoom:80%;" />

- **插入意向锁是在插入一条记录行前，由 INSERT 操作产生的一种间隙锁**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319211557961.png" alt="image-20250319211557961" style="zoom:80%;" />

- **事实上插入意向锁并不会阻止别的事务继续获取该记录上任何类型的锁。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319211658164.png" alt="image-20250319211658164" style="zoom:80%;" />

### 乐观锁、悲观锁：从对待锁的态度划分

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps47.jpg) 

​	从对待锁的态度来看锁的话，可以将锁分成乐观锁和悲观锁，从名字中也可以看出这两种锁是两种看待数据并发的思维方式。需要注意的是，**乐观锁和悲观锁并不是锁，而是锁的设计思想**。

<u>**【悲观锁】**</u>

​	悲观锁适合写操作多的场景，因为写的操作具有排它性。采用悲观锁的方式，可以在数据库层面阻止其他事务对该数据的操作权限，防止读-写和写-写的冲突

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319211906752.png" alt="image-20250319211906752" style="zoom:80%;" />

**eg：**商品秒杀过程中，库存数量的减少，避免出现超卖的情况。比如，商品表中有一个字段为quantity表示当前该商品的库存量。假设商品为华为mate40，id为1001，quantity=100个。如果不使用锁的情况下，操作方法如下所示:

- 不使用锁的情况：

  ```sql
  #第1步:查出商品库存
  select quantity from items where id = 1001 ;
  #第2步:如果库存大于0，则根据商品信息生产订单
  insert into orders (item_id）values ( 1001 ) ;
  #第3步:修改商品的库存，num表示购买数量
  update items set quantity = quantity-num where id = 1001 ;
  ```

- 使用锁的情况：**需要将要执行的SQL语句放在同一个事务中**

  ```sql
  #第1步:查出商品库存
  select quantity from items where id = 1001 for update;
  #第2步:如果库存大于0，则根据商品信息生产订单
  insert into orders (item_id)values(1001);
  #第3步:修改商品的库存，num表示购买数量
  update items set quantity = quantity-num where id = 1001 ;
  ```

  - select … for update是MySQL中悲观锁。 此时在items表中，id为1001的那条数据就被锁定了，其他的要执行select quantity from items where id = 1001 for update;语句的事务必须等本次事务提交之后才能执行。这样可以保证当前的数据不会被其它事务修改。
  - 注意，**当执行select quantity from items where id = 1001 for update;语句之后，如果在其他事务中执行select quantity from items where id = 1001;语句，并不会受第一个事务的影响，仍然可以正常查询出数据（因为此时是快照读）。**

**<u>【乐观锁】</u>**

- 乐观锁认为对同一数据的并发操作不会总发生，属于小概率事件，不用每次都对数据上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，也就是**不采用数据库自身的锁机制，而是通过程序来实现**。
- 我们可以采用**<font color = '#8D0101'>版本号机制或者CAS机制</font>**实现。在Java中java.util.concurrent.atomic包下的原子变量类就是使用了乐观锁的一种实现方式：CAS实现的。
- 乐观锁适合**读操作多的场景，可以提高吞吐量**，相对来说写的操作比较少。它的优点在于程序实现，不存在死锁

**1） 版本号机制**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps48.jpg) 

**2）时间戳机制**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps49.jpg) 

### 显式锁、隐式锁：按加锁的方式划分

**【隐式锁】**

- 一个事务对新插入的记录可以**不显示的加锁（生成一个锁结构**），但是由于事务id的存在，相当于加了一个隐式锁。别的事务在对这条记录加S锁或者X锁时，由于隐式锁的存在，会**先帮助当前事务生成一个锁结构，然后自己再生成一个锁结构后进入等待状态**。**隐式锁是一种延迟加锁的机制，从而来减少加锁的数量。**

**【显式锁】**

- 通过特定的语句进行加锁，我们一般称之为显示加锁。

### 数据库死锁

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。若无外力作用，事务将无法推进下去。

- 两个事务都持有对方需要的锁，并且都在等待对方释放，并且双方都不会释放自己的锁

【**死锁的示例**】

假设这时有两事务，一个事务A要插入订单1007，另外一个事务要插入订单1008，两个事务先要查询该订单是否存在，不存在才插入记录，过程如下：

![image-20250319212932085](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319212932085.png)

​	可以看到，两个事务都陷入了等待状态（前提没有打开死锁检测），也就是发生了死锁，因为都在相互等待对方释放锁。这里在**查询记录是否存在的时候，使用了select ... for update语句，目的为了防止事务执行的过程中，有其他事务插入了记录，而出现幻读的问题**。

【**解决方式**】

**1）等待，直到超时（innodb_lock_wait_timeout=50s）**

即当两个事务互相等待时，当一个事务等待时间超过设置的阈值时，就将其回滚，另外事务继续进行。

- InnoDB中，参数innodb_lock_wait_timeout可以设置超时的时间。该机制虽然简单，但是其仅通过超时后对事务进行回滚的方式来处理，或只说根据FIFO的顺序选择回滚对象，但是若超时的事务所占权重比较大，如事务操作更新了很多行，占用了较多的undo log，这时采用FIFO的方式，就显得不合适了，因为回滚这个事务的时间相对另外一个事务所占用的时间可能更多。

**2）使用死锁检测进行死锁处理**

- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务（将持有最少行级排他锁的事务进行回滚），让其他事务得以继续执行。
- **InnoDB采用Wait-for graph(等待图)方式**，该方式是一种更为主动的死锁检测方式。等待图要求数据库保存一下两种信息：锁的信息链表，事务等待链表。通过这俩表可以构造出一张图，而在这个图中若存在回路，就代表存在死锁，InnoDB选择回滚undo量最小的事务。

## 事务日志

### 概述

事务有4种特性：**原子性、一致性、隔离性和持久性**。那么事务的四种特性到底是基于什么机制实现呢？

- **事务的隔离性由锁机制实现**。 

- 而事务的**原子性、一致性和持久性**由事务的**redo日志和undo日志**来保证。
  - Redo log称为重做日志，提供再写入操作，恢复提交事务修改的页操作，用来保证事务的持久性。
  - Undo log称为回滚日志，回滚行记录到某个特定版本，用来保证事务的原子性、一致性。

**redo log是物理日志，记录的是数据页的物理变化**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps50.jpg) 

**undo log是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子。**![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps51.jpg)

**undo log不是redo log的逆过程**

### redo日志

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319213751588.png" alt="image-20250319213751588" style="zoom:80%;" />

​	InnoDB引擎的事务采用了**WAL技术（Write-Ahead Logging)**，这种技术的思想就是<font color = '#8D0101'>**先写日志，再写磁盘**</font>，只有日志写入成功，才算事务提交成功，这里的日志就是redo log。**当发生宕机且数据未刷到磁盘的时候，可以通过redo log来恢复，保证ACID中的D持久性，这就是redo log的作用**。

#### Redo日志的好处、特点

**【好处】**

- redo日志降低了刷盘频率
- redo日志占用的空间非常小

存储表空间ID、页号、偏移量以及需要更新的值，所需存储空间小，刷盘快

**【特点】**

- redo日志是顺序写入磁盘的

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps52.jpg) 

- 事务执行过程中，redo log不断记录

#### redo的整体流程

![image-20250319214606344](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319214606344.png)

1）先将原始数据从磁盘中读入内存中的buffer pool来，修改数据的内存拷贝

2）生成一条redo log并写入redo log buffer，记录的是数据被修改后的值

3）当事务commit时，将redo log buffer中的内容刷新到redo log file，对redo log file采用追加写的方式

4）定期将内存中修改的数据刷新到磁盘中

### Undo日志

#### 基本概念

**redo log是事务持久性的保证，undo log是事务原子性的保证。在事务中更新数据的前置操作其实是要先写入一个undo log**。

事务需要保证原子性，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半会出现一些情况，比如： 

- 情况一：事务执行过程中可能遇到各种错误，比如服务器本身的错误，操作系统错误，甚至是突然断电导致的错误。
- 情况二：程序员可以在事务执行过程中手动输入ROLLBACK语句结束当前事务的执行

​	以上情况出现，我们需要把数据**改回原先的样子**，这个过程称之为**回滚**，这样就可以造成一个假象：这个事务看起来什么都没做，所以符合原子性要求

#### Undo日志的作用

- 作用1：回滚数据(覆水难收)

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps53.jpg) 

- 作用2：MVCC

#### Undo的类型

在lnnoDB存储引擎中,undo log分为:

**1）insert undo log**

- insert undo log是指在**insert操作中产生的undo log**。因为insert操作的记录，只对事务本身可见，对其他事务不可见(这是事务隔离性的要求)，故**该undo log可以在事务提交后直接删除**，不需要进行删除操作

**2）update undo log**

- update undo log记录的是对**delete和update操作产生的undo log**。**该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除**。

#### undo log的生命周期（包括redo log和buffer pool）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319215423290.png" alt="image-20250319215423290" style="zoom:80%;" />

### redo log和undo log的对比

![image-20250319215522828](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319215522828.png)

### Bin log

​	binlog即binary log，二进制日志文件，也叫作变更日志（update log）。它记录了**数据库所有执行的DDL和DML等数据库更新事件的语句，但是不包含没有修改任何数据的语句（如数据查询语句select、show等）**。

主要应用场景：

- **一是用于数据恢复**，如果MySQL数据库意外停止，可以通过二进制日志文件来查看用户执行了哪些操作，对数据库服务器文件做了哪些修改，然后根据二进制日志文件中的记录来恢复数据库服务器。
- **二是用于数据复制**，由于日志的延续性和时效性，master把它的二进制日志传递给slaves来达到master-slave数据—致的目的。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319215623367.png" alt="image-20250319215623367" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250319215638983.png" alt="image-20250319215638983" style="zoom:80%;" />

### ==redo log和bin log的两阶段提交==

​	**两阶段提交（Two-Phase Commit, 2PC）** 主要用于保证 **事务的原子性和持久性** ，它解决了事务在提交过程中，如何确保 **Redo Log（重做日志）** 和 **Binlog** 的一致性问题，避免数据不一致或丢失

【**为何需要两阶段提交**】

MySQL 的事务提交涉及两个关键日志：

- **Redo Log** （InnoDB 层）：记录事务对数据页的物理修改，用于崩溃恢复。
- **Binlog** （Server 层）：记录逻辑 SQL 变更，用于主从复制和数据恢复
- **redo log与binlog的写入时机不一样**。在执行更新语句过程，会记录redo log与binlog两块日志，以基本的事务为单位，**redo log在事务执行过程中可以不断写入，而binlog只有在提交事务时才写入**。

如果直接提交事务（先写 Redo Log 再写 Binlog，或反之），可能因崩溃导致两者不一致。

eg：**事务 T1 修改了数据，Redo Log 写入成功，但 Binlog 写入前崩溃。**

- InnoDB 通过 Redo Log 恢复数据，但 Binlog 缺失该事务，导致主从数据不一致。

【**redo log的两阶段提交流程**】

1. **Prepare 阶段** （InnoDB 层）：
   - 写入 Redo Log，状态标记为 `Prepare`（未提交）。
   - 将事务的 XID（事务唯一标识）写入 Redo Log。
   - **不提交事务** ，仅保证事务的持久性。
2. **Commit 阶段** （Server 层）：
   - 写入 Binlog。
   - 将 Redo Log 的状态从 `Prepare` 改为 `Commit`，标记事务完成。

![请添加图片描述](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/ae8d7ee35d16e863a95e499f6cece05e.png)

【**崩溃恢复的规则**】

如果系统在两阶段提交过程中崩溃，重启时会根据 Redo Log 和 Binlog 的状态决定事务的最终状态：

1. **Redo Log 状态为 `Prepare`** ：
   - 检查 Binlog 是否完整（是否存在对应的事务）。
     - **Binlog 完整** ：提交事务（将 Redo Log 状态改为 `Commit`）。
     - **Binlog 不完整** ：回滚事务（删除 Redo Log）。
2. **Redo Log 状态为 `Commit`** ：
   - 事务已提交，无需额外操作。

### binlog与redolog对比

- **redo log 是物理日志** ，记录内容是“在某个数据页上做了什么修改”，属于 **InnoDB 存储引擎层**产生的。
- **而 binlog 是逻辑日志** ，记录内容是语句的原始逻辑，类似于“给 ID=2 这一行的 c 字段加 1”，属于 **MySQL Server层**。
- 虽然它们都属于持久化的保证，但是侧重点不同。
  - redo log让InnoDB存储引擎拥有了崩溃恢复能力
  - binlog 保证了MySQL集群架构的数据一致性。
- **redo log与binlog的写入时机不一样**。在执行更新语句过程，会记录redo log与binlog两块日志，以基本的事务为单位，**redo log在事务执行过程中可以不断写入，而binlog只有在提交事务时才写入**。



