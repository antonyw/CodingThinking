# Spring 事务机制剖析
作为Java开发者，不免要接触到两种事务管理机制——全局事务和本地事务。

如果你不熟悉什么是全局事务或本地事务，让我们先看几个概念：
1. Resource Manager 简称RM，它负责存储并管理系统数据资源的状态，比如数据库服务器、JMS消息服务器都是RM。
2. Transaction Processing Monitor 简称TPM或TP monitor，它的职责是在分布式事务中协调包含多个RM的事务处理。TPM通常对应特定的中间件。
3. Transaction Manager 简称TM，它可以认为是TPM中的核心模块，直接负责多RM之间事务处理的协调工作，并且提供事务界定、事务上下文传播。
4. Application 以独立行使存在的或运行于容器中的应用程序，可以认为是事务边界的触发点。

理解了上面几个概念之后，其实全局事务和本地事务的区别就是在整个事务的过程中涉及到的RM数量多少。

### 全局事务
如果真个事务过程中涉及多个RM参与，那么就需要引入TPM来协调多个RM之间的事务处理。TPM通过一定的机制保证整个事务在多个RM之间的ACID属性。其实也就是我们常说的“分布式事务”。

### 局部事务
如果事务只涉及到一个RM参与，那么这就是局部事务，它并不需要TPM的介入。例如我们在订单生成后写一张表，或向一个消息队列发送一条消息。

### Java 中的事务
在Java原生环境中谈，对于局部事务来说，它对代码有很大的侵入性，而且对访问方式的抽象做的并不好。例如使用JDBC访问数据，那么就需要通过java.sql.Connection来处理事务；如果使用Hibernate访问数据，则需要通过Hibernate自己的类。这就容易导致事务管理代码与数据访问代码高度融合。

对于分布式事务来说，Spring之前使用全局事务的首选方法是通过EJB CMT（容器管理事务）。CMT是一种声明式事务管理（与程序化事务管理不同）。EJB CMT消除了与事务相关的JNDI查找的需要，尽管EJB本身的使用需要使用JNDI。它消除了编写Java代码以控制事务的大部分但不是全部的需要。但它有个限制，那就是必须借助EJB容器才能得以使用这种功能。

## Spring 事务管理突破
看过了Spring时代之前的局限性，接下来我们看一下**Spring为我们提供的一个重大革新——一致性编程模型（Consistent Programming Model）。**

它允许应用程序开发人员在任何环境中使用一致的编程模型。你只需编写一次代码，就可以从不同环境中的不同事务管理策略中受益。

与此同时，Spring 提供了声明式和编程式事务管理。

### Spring 事务抽象层
PlatformTransactionManager 是事务抽象架构的核心接口，既然是接口那么它就是为应用程序提供统一的界定方式。在接口内部很简洁的定义了三个函数。
```java
TransactionStatus getTransaction(TransactionDefinition def);
void commit(TransactionStatus status);
void rollback(TransactionStatus status);
```
该接口有一个默认的抽象实现类AbstractPlatformTransactionManager（**以下简称APTM**），打开这个抽象实现类就能看到它的子类是各个数据访问框架的支持类，其中包括jdbc、hibernate、jta等等。

**APTM 以模版方法的形式封装了固定的事务处理逻辑，而只将与事务资源相关的操作暴露给子类实现。** APTM 替子类实现了以下逻辑：
1. 判定是否存在当前事务，然后根据判断结果执行不同的处理逻辑；
2. 根据TransactionDefinition中定义的事务传播级别执行后续逻辑；
3. 根据情况挂起或者恢复事务；
4. 提交事务之前检查rollbackOnly字段是否被设置，如果是的话，以事务的回滚代替事务提交；
5. 在事务回滚的情况下，清理并恢复事务状态；
6. 如果事务的Synchonization处于active状态，在事务处理的规定时点触发注册的Synchonization回调接口。

### Spring 事务多子类实现
上述逻辑基本分布于这几个方法中：getTransaction / rollback / commit / suspend / resume

从getTransaction开始看，它的主要目的是开启一个事务，但需要在此之前判断一下是否已经存在了一个事务。如果存在，则需要根据定义的传播级别决定是挂起当前事务还是抛出异常。接下来我们看一下该方法的具体逻辑。

#### 1.该方法的第一行代码
```java
Object transaction = doGetTransaction();
```
doGetTransaction 是一个抽象方法，需要子类来实现，也就是说获取的transactionObject会因子类的不同而的不同。但APTM不需要知道具体什么类型，因为在模板方法模式中，后续的调用都通过参数传递，只要后续的操作中子类自己知道是什么类型即可。

以DataSourceTransactionManager（**以下简称DSTM**）为例，它的doGetTransaction实现中有一个很重要的逻辑，从TransactionSynchronizationManager中获取绑定的资源，然后添加到DSTM之后返回。之后我们会讲为什么要做资源绑定。

#### 2.definition检查
如果definition参数为空，则创建一个DefaultTransactionDefinition实例已提供默认事务定义数据。

#### 3.事务处理
首先根据先前获得的transaction object判断是否存在当前事务，根据判定结果采取不同的处理方式
```java
if (isExistingTransaction(transaction)) {
    // Existing transaction found -> check propagation behavior to find out how to behave.
	return handleExistingTransaction(definition, transaction, debugEnabled);
}
```
isExistingTransaction默认返回false，该方法的具体实现由子类覆写。其实不管isExistingTransaction的返回值如何，下面的代码都是基于definition中定义的事务传播级别的不同，处理当前事务是应该加入之前的事务，还是新开启一个事务。

这个过程中有一个doBegin方法，上面讲到它也是抽象方法，需要由子类来实现。在DSTM中，该方法的逻辑会首先检查传入的transaction object是否存在绑定的connection。如果没有，则从datasource中获取新的connection，然后将其AutoCommit设为false，并绑定到TransactionSynchronizationManager。之后newTransactionStatus会创建一个包含definition、transaction object以及挂起的事务信息和其他状态信息的DataTransactionStatus并返回。

#### 4.事务处理完成
事务处理完成有两种情况，回滚事务或者提交事务。rollback和commit两个方法对应了这两种情况。

**Rollback**

rollback的逻辑大致有以下3点：回滚事务、触发Synchronization事件、清理事务资源。

**回滚事务：**

1. 如果是嵌套事务，则通过TransactionStatus释放SavaPoint；
2. 如果TransactionStatus表示当前事务是一个新事务，则调用子类doRollback方法。对于DSTM这个实现类来说，其实就是调用了connection.rollback；
3. 如果要参与一个已经存在的事务，则调用子类doSetRollbackOnly方法，子类的实现会保证transaction object的状态设置为rollbackOnly。

**触发Synchronization事件：**
triggerBeforeCompletion和triggerAfterCompletion会被触发。

**清理事务资源：**
1. 设置TransactionStatus中的completed为完成状态；
2. 清理与当前事务相关的Synchronization;
3. 调用doCleanupAfterCompletion清理资源，并解除资源绑定；对于DSTM来说，就是关闭数据库，并解除对datasource对应资源的绑定。

**Commit**

commit操作的核心逻辑也是通过doCommit方法交由子类实现，对于DSTM来说，就是connection.commit。同时，commit也会触发trigger事件，commit的结尾也会涉及清理资源。

## 核心思想
前面说到了很多资源绑定解绑的操作，究竟为什么需要绑定资源？

其实不难理解，在传统的JDBC代码中，JDBC的局部事务控制是通过一个java.sql.Connection来完成的。Spring要通过声明式或编程式的事务将多个数据库操作融合进一个事务里，那么只需要将多个操作共享一个connection，并把autocommit属性设为false即可。

讲下来是不是发现这里的connection资源使用方式似曾相识，就像我们常用的线程，那或许可以存在一个类似线程池的组件，把所有connection都放到一个地方，无论是谁要使用该资源，都从这里获取。通俗点讲，我们事务开始之前取得一个connection，然后将connection绑定到当前调用线程。之后，数据访问都使用这同一个connection。当所有数据访问操作结束之后，使用这个connection进行事务提交或回滚。最后再接触connection和线程的绑定关系。

文章开头就说到Spring为我们提供了一个一致性的编程模型，这个模型的核心思想就是**让事务管理的关注点与数据访问的关注点相分离。** 而connection的复用正是实现这种模型的关键一步，因为我们在编写业务代码的时候不再需要显式的注入connection，只需要在关键位置从统一位置获取同一个connection进行复用即可，实现了高效率解耦。

### 参考资料
- [Spring官方文档](https://docs.spring.io/spring/docs/5.1.8.RELEASE/spring-framework-reference/)
- 《Spring 揭秘》作者：王福强