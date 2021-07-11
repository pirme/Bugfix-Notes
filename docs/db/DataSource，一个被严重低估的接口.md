## 文章产出背景

代码不多，文章不长，简要描述了下 DataSource 演进过程的故事

最近这段时间一直忙着集团内部安全等保加密相关事项，初步决定使用 shardingsphere 来进行
 
因为项目众多，需要兼容的需求也随之而来，加密数据源和动态数据源的互相兼容，以及分库分表和加密数据源的兼容等等，反正一言不合就兼容

![](https://images-machen.oss-cn-beijing.aliyuncs.com/datasource-bq-委屈.jpg)

无疑，上面这些或多或少都和数据源有关系，所以在处理不同兼容性等问题时，也让我再次对 DataSource 产生了了解的兴趣，这个曾经被很多人遗忘的重要概念


## 老时代的数据查询

在很久很久以前（反正忘了多久），那个时候程序员连接数据库还是这么个操作

![](https://images-machen.oss-cn-beijing.aliyuncs.com/datasource-2.png)

可以清楚的看到，曾经获取数据库连接的代码还需要使用 DriverManager，大家都清楚，`DriverManager#getConnection` 是通过数据库驱动直接与数据库建立连接

<font color="#FF0000">而建立数据库连接属于耗费时间的事情</font>，如果业务层每次进行 SQL 查询都使用此方式，将会产生较大的系统开销

一般系统的性能要求，单次请求需要稳定在 200 ms。经过在本地环境测试，DriverManager 形式创建数据库连接需要 **300~500 ms** 左右，**健身五分钟，拍照两小时？**

![](https://images-machen.oss-cn-beijing.aliyuncs.com/datasource-老天爷.jpg)

随着系统的发展迭代，出于 **系统的复杂度以及对性能的要求**，这种获取连接的方式是难以接受的

## 连接池的出现

相信有些读者能够联想出来，数据库连接创建消耗资源这个场景好像在哪听过，没错，和线程创建的情况基本类似。既然线程可以用线程池关联，**那数据库连接是不是可以放到一个池子中？**

是的，还真有存放数据库连接的池化技术，叫做 **连接池**。应用程序从连接池中获取数据库连接，**使用过再放到池子里**，是不是感觉很 Nice？完美的解决了重复创建消耗资源的情况

看似完美的背后，其实还存在一个致命的问题，那就是最开始去连接池中获取连接时，连接池中是没有连接的，还需要走创建流程

连接池是怎么解决这一问题呢？**通过创建连接池时初始化其中的连接**，一般连接池都有这样一个参数 `initialSize`，代表池子中初始化的连接数量

下图是 Druid 在执行 init 方法时数据库连接初始化的流程，当初始化连接小于池内连接时，进行循环创建，直到池内连接满足初始化数量

![](https://images-machen.oss-cn-beijing.aliyuncs.com/datasource-4.png)

连接池、线程池...这些池化技术的核心思想就是 **空间换取时间**。因为在绝大数情况下，空间并没有那么稀缺，我们更关心的是系统的性能

## 数据源登场

连接池虽然 🐂 🍺 ，但是独木难支，**连接池没有产生连接的能力**，所以还需要配合类似 DriverManager 组件与数据库驱动配合创建连接。如果这样放到业务代码里，那岂不是还得封装一层？

这个时候，不约而同的想到一个公司，sun 公司是干啥的？制定规范的对不对，他们在 jdbc 2.0 版本推出一个 DataSource 的东东，**用来进行规范约束**

相当于把 DriverManager 和连接池概念揉合在一起，**如果你想获取数据库连接，你通过我 DataSource 获取**，你不用关心连接池和数据库连接怎么创建的，用就完了。其实，DataSource 获取的连接来自于连接池，而连接池的连接其实还是从 DriverManager或类似组件中创建的

---

**DriverManager 只是 jdbc 1.0 版本用来调用数据库驱动的的工具包**，jdbc 2.0 版本推出 DataSource 之后，典型的像 **DruidDataSource 就没有依赖 DriverManager**，而是在自己实现类中调用了数据库驱动。这里只是重点强调，**数据库连接不一定是使用 DriverManager 创建**

---

总结下，数据源（DataSource）是 sun 公司指定用于获取数据库连接的规范接口，应用程序于数据库连接抽象的中间层，它存在于 `javax.sql` 包，用来代替 DriverManager 的方式获取数据库连接

#### 使用 DataSource 比 DriverManager 到底有什么好处呢

**DriverManager**

- 在应用程序里创建/关闭连接时会妨碍应用程序性能

- 不支持连接池，重复创建/关闭连接，浪费系统性能

**DataSource**

- 由于不在应用程序中创建/关闭连接，可以很好的提高应用程序性能

- 提供了连接池的功能，避免重复创建

这里画一张图来描述下，针对应用程序使用 DataSource 和 DriverManager 获取连接的不同

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210516123742062.png)

有了 DataSource 之后，数据库连接、用户名、密码都进行了统一的管理，作为 DataSource 属性的一部分，并且将数据库驱动名称填充，底层自动加载



## DataSource 技术解析

我们先来看下 DataSource 的接口描述以及对应的方法，先初步进行了解

DataSource 中只有两个接口，是一个重载的关系，用于建立 DataSource 所代表数据源的数据库连接

![](https://images-machen.oss-cn-beijing.aliyuncs.com/datasource-3.png)

这里应该注意的是 `CommonDataSource` 接口，公共数据源接口用来定义以下三个数据源接口的公共方法

- javax.sql.DataSource：定义基础获取数据库连接的接口

- javax.sql.ConnectionPoolDataSource：定义从数据库连接池中获取连接的接口

- javax.sql.XADataSource：定义获取分布式事务连接的接口。一般少有直接使用 XA 分布式事务，具体原因参考 [分布式 2PC、3PC 事务模型](https://mp.weixin.qq.com/s/_zhntxv07GEz9ktAKuj70Q)

第一、二种就比较容易理解，sun 公司定义规范时，就是希望你 **普通获取数据库连接使用 DataSource**，**数据源底层如果是连接池那么使用 ConnectionPoolDataSource**

后面发展逐渐脱离了原本的轨道预期，比如 DruidDataSource 就同时实现了两者，类图如下

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210516130105185.png)

其实这样也没啥事，DruidDataSource 一个类包装两种 DataSource 接口实现，这种方式对于使用者是无感知的



## 完事总结

文章使用循序渐进的方式，帮助大家梳理了一遍 DataSource 产出背景

讲述了 jdbc 1.0 版本获取数据库连接的方式 DriverManager，发展到 2.0 后的 DataSource，以及其中引入的数据库连接池技术

相信读者看完对 DataSource 应该有了更深入的了解，感兴趣的读者可以去研读 Hikari 和 Druid 实现的数据源，通过阅读源码的方式能够更好的理解 DataSource 设计思路
