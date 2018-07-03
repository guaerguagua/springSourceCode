# spring事务机制
## 初步理解
理解事务之前，先通过一个例子来说明：充值。
比如你给一个游戏账户充钱，在你从微信、支付宝、银行卡扣款以后，游戏账户的后台大概应该做如下的一些事情：比如增加一条充值的交易明细，在你的账户上加上一笔钱，同时可能还有一个汇总的金额表示公司账户上一共充值了多了钱，这三个修改记录是分布在不同的数据库表里，如果其中任何一个操作失败了，都应该回滚到最初的状态。事务就是用来解决同时操作多个表的一致性问题的，事务是一系列操作，这些操作必须全部完成，如果有一个失败了，那么就应该回滚到最初的状态。在企业级应用中，事务管理是必不可少的技术，用来确保数据的一致性和完成性。
事务有一下的四个特征：ACID
>*原子性（Atomicity）:事务是一个原子操作，由一系列动作组成，事务的原子性确保动作要么全部不做，要么完全不起作用
*一致性（Consistency）：一旦事务完成，不管成功还是失败，系统确保它所建模的业务处于一致的状态，而不会是部分成功或失败。在现实中的数据不应该被破坏。
*隔离性（Isolation）：可能会有很多事务同时处理数据相同的数据，因此每个事务都应该与其他事务隔离开来，防止数据损坏。
*持久性（Durability）：一旦事务完成，无论发生什么系统错误，它的结果都不应该受影响，这样就能从任何系统奔溃中恢复过来。通常情况下，事务的结果被写到持久化存储中。

## 核心接口
spring事务设计的接口的联系如下：
![](./1.png)
### 事务管理器
Spring并不直接管理事务，而是提供多种事务管理器，他们将事务管理器的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。
Spring事务管理的接口是org.springframework.transaction.PlatformTransactionManager,通过这个接口，Spring为各个平台如JDBC，Hibernate,JPA,JTA等都提供了对应的事务管理器，但是具体的实现就是平台自己的事情了。此接口的内容如下：
```java
public interface PlatformTransactionManager()...{
	//由TransactionDefinition得到TransactionStatus对象
	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
	//提交
	void commit(TransactionStatus status) throws TransactionException;
	//回滚
	void rollback(TransactionStatus status) throws TransactionException;
}
```
#### JDBC事务
如果应用程序直接使用JDBC来进行持久化，DataSourceTransactionManager会为你处理事务边界。为了使用DataSourceTransactionManager，你需要使用如下的xml配置：
```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref = "dataSource"/>
</bean>
```
实际上，DataSourceTransactionManager是通过调用java.sql.Connection来管理事务，而后者是通过DataSource获取的。通过调用连接的commit()方法来提交事务，通过调用其中的rollback()方法来回滚事务。
#### Hibernate事务
如果应用程序的持久化是通过Hibernate实现的，那么你需要使用HibernateTransactionManager。对于hibernate3，需要在spring上下文定义中添加如下bean：
```xml
<bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
	<property name = "sessionFactory" ref = "sessionFactory"/>
</bean>
```
sessionFactory属性需要装配一个Hibernate的session工厂。HibernateTransactionManager的实现细节是它将事务管理的职责委托给org.hibernate.Transaction对象，而后者是从session工程中获取的。当事务完成时候，调用Transaction对象的commit()方法，失败则调用回滚rollback()。
#### Java持久化API事务
Hibernate多年以来一直是事实上的Java持久化标准，但是现在Java持久化API作为真正的持久化标准进入大家视野。使用JPA，需要在spring上下文定义中添加如下bean：
```xml
<bean id ="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
	<property name="sessionFactory" ref="sessionFactory"/>
</bean>
```
JpaTransactionManager只需要装配一个JPA实体管理工厂（javax.persistence.EntityManagerFactory接口的任意实现）。JpaTransactionManager将与由工厂所产生的JPA EntityManager合作来构建事务。
#### Java原生API事务
如果你没有使用以上所述的事务管理，或者是跨越了多个事务管理源（比如两个或者是多个不同的数据源），你就需要使用JtaTransactionManager：
```xml
 <bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
        <property name="transactionManagerName" value="java:/TransactionManager" />
 </bean>
```
JtaTransactionManager将事务管理的责任委托给javax.transaction.UserTransaction和javax.transaction.TransactionManager对象，其中事务成功完成通过UserTransaction.commit()方法提交，事务失败通过UserTransaction.rollback()方法回滚。
### 基本事务属性的定义
上面讲到事务管理接口通过PlatformTransactionManager通过getTransaction(TransactionDefinition definition)方法得到事务，这个方法里面的参数是TrasactionDefinition类，这个类就定义了一些基本的事务属性：
事务属性可以理解为一些事务的基本配置，描述了事务策略如何应用到方法上。事务属性包含5个方面，如下：
1[](./2.png)
TransactionDefinition接口内容如下：
```java
public interface TransactionDefinition{
	int getPropagationBehavior();//返回事务的传播行为
	int getIsolationLevel();//返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务的哪些数据
	int getTimeout();//返回事务的超时时间
	boolean inReadOnly();//事务是否只读，事务管理器能顾根据这个返回值进行优化，确保事务是只读的
}
```
#### 传播行为
事务的第一个方面是传播行为（propagation behavior）。当事务方法被另一个事务方法调用的时候，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。Spring定义了7种传播行为：
|传播行为|含义|
|PROPAGATION_REQUIRED|表示当前方法必须运行在事务中。如果当前事务存在，方法将在该事务中运行。否则会启动一个新的事务。|
|PROPAGATION_SUPPORTS|表示当前方法不需要事务上下文，如果存在当前事务的话，那么该方法会在事务中运行。|
|PROPAGATION_MANDATORY|表示该方法必须在事务中运行，如果当前事务不存在则会抛出一个异常。|
|PROPAGATION_REQUIRED_NEW|表示当前方法必须运行在它自己的事务中。一个新的事务将被开启。如果存在当前事务，则该方法执行期间，当前事务会被挂起。|
|PROPAGATION_NOT_SUPPORTED|表示该方法不应该运行在事务中，如果当前事务存在的话，那么当前事务会被挂起。|
|PROPAGATION_NEVER|表示当前方法不应该运行在事务上下文当中。如果当前事务存在，那么会抛出异常。|
|PROPAGATION_NESTED|表示如果当前已经存在一个事务，那么该方法将会在嵌套事务中运行。嵌套事务可以独立与当前的事务进行单独的回滚和提交。如果当前事务不存在，会启动一个新的事务。注意各个厂商对于这种传播行为的支持有差异，可以参考资源管理器的文档来确认它们是否支持嵌套事务。|

嵌套事务和在当前事务中新开一个事务的区别？
#### 隔离级别
#### 只读
#### 事务超时
#### 回滚规则
### 事务状态
## 编程式事务
### 编程式事务和声明式事务的区别
### 如何实现编程式事务
#### 使用TransactionTemplate
#### 使用PlatformTransactionManager
## 声明式事务
### 配置方式
### 一个声明式事务的实例
