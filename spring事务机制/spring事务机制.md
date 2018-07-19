# spring事务机制
## 初步理解
理解事务之前，先通过一个例子来说明：充值。
比如你给一个游戏账户充钱，在你从微信、支付宝、银行卡扣款以后，游戏账户的后台大概应该做如下的一些事情：比如增加一条充值的交易明细，在你的账户上加上一笔钱，同时可能还有一个汇总的金额表示公司账户上一共充值了多了钱，这三个修改记录是分布在不同的数据库表里，如果其中任何一个操作失败了，都应该回滚到最初的状态。事务就是用来解决同时操作多个表的一致性问题的，事务是一系列操作，这些操作必须全部完成，如果有一个失败了，那么就应该回滚到最初的状态。在企业级应用中，事务管理是必不可少的技术，用来确保数据的一致性和完成性。
事务有一下的四个特征：ACID
>*原子性（Atomicity）:事务是一个原子操作，由一系列动作组成，事务的原子性确保动作要么全部不做，要么完全不起作用
*一致性（Consistency）：一旦事务完成，不管成功还是失败，系统确保它所建模的业务处于一致的状态，而不会是部分成功或失败。在现实中的数据不应该被破坏。
事务的运行并不改变数据库中数据的一致性.例如,完整性约束了a+b=10,一个事务改变了a,那么b也应该随之改变.
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
注：以下具体讲解传播行为的内容参考自[Spring事务机制详解](http://www.open-open.com/lib/view/open1350865116821.html)
(1)PROPAGATION_REQUIRED 如果存在一个事务，则支持当前事务。如果没有事务存开启一个新的事务。
```java
//事务属性 PROPAGATION_REQUIRED
methodA{
	...
	methodB();
	...
}
//事务属性 PROPAGATION_REQUIRED
methodB{
	...
}
```
单独调用methodB方法:
```java
main{
	methodB();
}
```
相当于：
```java
Main{
	Connection con= null;
	try{
		con = getConnection();
		con.setAutoCommit(false);
		methodB();
		con.commit();
	}Catch(){
		con.rollback();
	}finally{
		con.close();
	}
}
```
Spring保证在methodB方法中所有的调用都获得一个相同的连接。在调用methodB方法的时候，没有一个存在的事务，所以获得一个新的连接开启一个新的事务。单独调用methodA的时候，在methodA内部又会调用methodB，但是只会开启一个事务。
（2）PROPAGATION_SUPPORTS 如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。
```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
  methodB();
}

//事务属性 PROPAGATION_SUPPORTS
methodB(){
  ……
}
```
单纯的调用methodB时，methodB方法是非事务的执行的。当调用methdA时,methodB则加入了methodA的事务中,事务地执行。
(3）PROPAGATION_MANDATORY 如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。
```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    methodB();
}

//事务属性 PROPAGATION_MANDATORY
    methodB(){
    ……
}
```
当单独调用methodB时，因为当前没有一个活动的事务，则会抛出异常throw new IllegalTransactionStateException(“Transaction propagation ‘mandatory’ but no existing transaction found”);当调用methodA时，methodB则加入到methodA的事务中，事务地执行。
(4）PROPAGATION_REQUIRES_NEW 总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。
```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    doSomeThingA();
    methodB();
    doSomeThingB();
}

//事务属性 PROPAGATION_REQUIRES_NEW
methodB(){
    ……
}
```
调用A方法：
```java
main(){
    methodA();
}
```
相当于：
```java
main{
	TransactionManager tm= null;
	try{
		//获得一个jta事务管理器
		tm = getTransactionManager();
		//开启一个新事务
		tm.begin();
		Transaction tr1 = tm.getTransaction();
		doSomethingA();
		//挂起当前事务
		tm.suspend();
		try{
			//开启第二个新事务
			tm.begin();
			Transaction tr2 = tm.getTransaction();
			methodB();
			tr2.commit();
		}catch(){
			tr2.rollback();
		}finally{
			//释放资源
		}
		//恢复第一个事务
		tm.resume(tr1);
		doSomeThingB();
		tr1.commit();
	}catch(){
		tr1.rollback();
	}finally{
		//释放资源
	}
}
```
在这里，我把ts1称为外层事务，ts2称为内层事务。从上面的代码可以看出，ts2与ts1是两个独立的事务，互不相干。Ts2是否成功并不依赖于 ts1。如果methodA方法在调用methodB方法后的doSomeThingB方法失败了，而methodB方法所做的结果依然被提交。而除了 methodB之外的其它代码导致的结果却被回滚了。使用PROPAGATION_REQUIRES_NEW,需要使用 JtaTransactionManager作为事务管理器。
（5）PROPAGATION_NOT_SUPPORTED 总是非事务地执行，并挂起任何存在的事务。使用PROPAGATION_NOT_SUPPORTED,也需要使用JtaTransactionManager作为事务管理器。（代码示例同上，可同理推出）
（6）PROPAGATION_NEVER 总是非事务地执行，如果存在一个活动事务，则抛出异常。
（7）PROPAGATION_NESTED如果一个活动的事务存在，则运行在一个嵌套的事务中. 如果没有活动事务, 则按TransactionDefinition.PROPAGATION_REQUIRED 属性执行。这是一个嵌套事务,使用JDBC 3.0驱动时,仅仅支持DataSourceTransactionManager作为事务管理器。需要JDBC 驱动的java.sql.Savepoint类。有一些JTA的事务管理器实现可能也提供了同样的功能。使用PROPAGATION_NESTED，还需要把PlatformTransactionManager的nestedTransactionAllowed属性设为true;而 nestedTransactionAllowed属性值默认为false。
```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    doSomeThingA();
    methodB();
    doSomeThingB();
}

//事务属性 PROPAGATION_NESTED
methodB(){
    ……
}
```
如果单独调用methodB方法，则按REQUIRED属性执行。如果调用methodA方法，相当于下面的效果：
```java
main(){
    Connection con = null;
    Savepoint savepoint = null;
    try{
        con = getConnection();
        con.setAutoCommit(false);
        doSomeThingA();
        savepoint = con2.setSavepoint();
        try{
            methodB();
        } catch(RuntimeException ex) {
            con.rollback(savepoint);
        } finally {
            //释放资源
        }
        doSomeThingB();
        con.commit();
    } catch(RuntimeException ex) {
        con.rollback();
    } finally {
        //释放资源
    }
}
```
当methodB方法调用之前，调用setSavepoint方法，保存当前的状态到savepoint。如果methodB方法调用失败，则恢复到之前保存的状态。但是需要注意的是，这时的事务并没有进行提交，如果后续的代码(doSomeThingB()方法)调用失败，则回滚包括methodB方法的所有操作。

嵌套事务一个非常重要的概念就是内层事务依赖于外层事务。外层事务失败时，会回滚内层事务所做的动作。而内层事务操作失败并不会引起外层事务的回滚。

PROPAGATION_NESTED 与PROPAGATION_REQUIRES_NEW的区别:它们非常类似,都像一个嵌套事务，如果不存在一个活动的事务，都会开启一个新的事务。使用 PROPAGATION_REQUIRES_NEW时，内层事务与外层事务就像两个独立的事务一样，一旦内层事务进行了提交后，外层事务不能对其进行回滚。两个事务互不影响。两个事务不是一个真正的嵌套事务。同时它需要JTA事务管理器的支持。

使用PROPAGATION_NESTED时，外层事务的回滚可以引起内层事务的回滚。而内层事务的异常并不会导致外层事务的回滚，它是一个真正的嵌套事务。DataSourceTransactionManager使用savepoint支持PROPAGATION_NESTED时，需要JDBC 3.0以上驱动及1.4以上的JDK版本支持。其它的JTA TrasactionManager实现可能有不同的支持方式。

PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 “内部” 事务. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行。

另一方面, PROPAGATION_NESTED 开始一个 “嵌套的” 事务, 它是已经存在事务的一个真正的子事务. 潜套事务开始执行时, 它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交。

由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于, PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 嵌套事务也会被 commit, 这个规则同样适用于 roll back.

PROPAGATION_REQUIRED应该是我们首先的事务传播行为。它能够满足我们大多数的事务需求。
#### 隔离级别
事务的第二个维度就是隔离级别（isolation level）。隔离级别定义了一个事务可能受其他并发事务影响的程度。 
（1）并发事务引起的问题
在典型的应用中，多个事务并发的运行，经常会操作相同的数据来完成各自的任务。并发虽然是必须的，但是会引起以下的问题：
```
脏读（dirty reads）:脏读发生在一个事务读取了另一个事务改写但是未提交的数据，但是改写稍后回滚了，那个第一个事务获取的数据就是无效的。
不可重复读（Nonrepeatable reads）：不可重复读发生在一个事务读取了两次或者以上次数的数据，但是读到的数据不一致，通常发生在并发中另一个事务更新了数据。
幻读（Phantom reads）：幻读与不可重复读类似。幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录。
```
不可重复读与幻读的区别
不可重复读是修改，幻读是增加了或者删除了些数据。
（2）隔离级别
|隔离级别|含义|
|---|---|
|ISOLATION_DEFAULT|使用后端数据库默认的隔离级别|
|ISOLATION_READ_UNCOMMITTED|最低的隔离级别，允许读取未提交的数据变更，可能会导致脏读，不可重复读，幻读|
|ISOLATION_READ_COMMITTED|允许读取并发事务已经提交的数据，可以阻止脏读，但是不可重复读和幻读有可能发生|
|ISOLATIN_REPETABLE_READ|对同一个字段的多次读取结果都是一致的，除非数据是被本身的事务所修改，可以阻止不可重复读和脏读，但是幻读仍然可能发生|
|ISOLATION_SERIALIZABLE|最高的隔离级别，完全服从ACID的隔离级别，确保阻止脏读、不可重复读和幻读，也是最慢的事务隔离级别，因为它通常是通过锁定事务相关的数据库表来实现的|

#### 只读
是指事务的运行并不改变数据库中数据的一致性.例如,完整性约束了a+b=10,一个事务改变了a,那么b也应该随之改变.
#### 事务超时
为了使应用程序很好地运行，事务不能运行太长的时间。因为事务可能涉及对后端数据库的锁定，所以长时间的事务会不必要的占用数据库资源。事务超时就是事务的一个定时器，在特定时间内事务如果没有执行完毕，那么就会自动回滚，而不是一直等待其结束。
#### 回滚规则
事务五边形的最后一个方面是一组规则，这些规则定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有遇到运行期异常时才会回滚，而在遇到检查型异常时不会回滚（这一行为与EJB的回滚行为是一致的） 
但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。
### 事务状态
上面讲到的调用PlatformTransactionManager接口的getTransaction()的方法得到的是TransactionStatus接口的一个实现，这个接口的内容如下：
```java
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事务
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
} 
```
可以发现这个接口描述的是一些处理事务提供简单的控制事务执行和查询事务状态的方法，在回滚或提交的时候需要应用对应的事务状态。
## 编程式事务
### 编程式事务和声明式事务的区别
Spring提供了对编程式事务和声明式事务的支持，编程式事务允许用户在代码中精确定义事务的边界，而声明式事务（基于AOP）有助于用户将操作与事务规则进行解耦。 
简单地说，编程式事务侵入到了业务代码里面，但是提供了更加详细的事务管理；而声明式事务由于基于AOP，所以既能起到事务管理的作用，又可以不影响业务代码的具体实现。
### 如何实现编程式事务
Spring提供两种方式的编程式事务管理，分别是：使用TransactionTemplate和直接使用PlatformTransactionManager。
#### 使用TransactionTemplate
采用TransactionTemplate和采用其他Spring模板，如JdbcTempalte和HibernateTemplate是一样的方法。它使用回调方法，把应用程序从处理取得和释放资源中解脱出来。如同其他模板，TransactionTemplate是线程安全的。代码片段：
```java
TransactionTemplate tt = new TransactionTemplate();
Object result= tt.execute(
	new TransactionCallback(){
		public Object doTransaction(TransactionStatus status){
			updateOperation();
			return returnOfUpdateOperation();
		}
	}
	);//执行execute方法进行事务管理
```
使用TransactionCallback()可以返回一个值。如果使用TransactionCallbackWithoutResult则没有返回值。
#### 使用PlatformTransactionManager
示例：
```java
DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
dataSourceTransactionManager.setDataSource(this.getJdbcTemplate().getDataSource());//设置数据源
DefaultTransactionDefinition transDef = new DefaultTransactionDefinition();//定义事务属性
transDef.setPropagationBehavior(DefaultTransactionDefiniton.PROPAGATION_REQUIRED);//设置传播行为属性
TransactionStatus status = dataSourceTransactionManager.getTransaction(transDef);
try{
	dataSourceTransactionManager.commit(status);
}catch（Exception e){
	dataSourceTransactionManager.roolback(status);
}
```
## 声明式事务
注：以下配置代码参考自[Spring事务配置的五种方式](http://www.blogjava.net/robbie/archive/2009/04/05/264003.html)
根据代理机制的不同，总结了五种Spring事务的配置方式，配置文件如下：
### 配置方式
（1）每个Bean都有一个代理
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">
    <bean id = "sessionFactory" class="org.springframework.orm.hibernate3.LocalSessionFactoryBean">
    	<property name="configLocation" value="classpath:hibernate.cfg.xml" />
    	<property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
    </bean>
    <!-- 定义事务管理器 （声明式的事务）-->
    <bean id = "transactionManager" class = "org.springframework.orm.hibernate3.HibernateTransactionManager">
    	<property name = "sessionFactory" ref ="sessionFactory"/>
    </bean>
     <!-- 配置DAO -->
    <bean id="userDaoTarget" class="com.bluesky.spring.dao.UserDaoImpl">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

    <bean id="userDao" 
        class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"> 
           <!-- 配置事务管理器 --> 
           <property name="transactionManager" ref="transactionManager" />    
        <property name="target" ref="userDaoTarget" /> 
         <property name="proxyInterfaces" value="com.bluesky.spring.dao.GeneratorDao" />
        <!-- 配置事务属性 --> 
        <property name="transactionAttributes"> 
            <props> 
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props> 
        </property> 
    </bean> 
</beans>
```
（2）所有Bean共享一个代理基类
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">

    <bean id="sessionFactory" 
            class="org.springframework.orm.hibernate3.LocalSessionFactoryBean"> 
        <property name="configLocation" value="classpath:hibernate.cfg.xml" /> 
        <property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
    </bean> 

    <!-- 定义事务管理器（声明式的事务） --> 
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

    <bean id="transactionBase" 
            class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean" 
            lazy-init="true" abstract="true"> 
        <!-- 配置事务管理器 --> 
        <property name="transactionManager" ref="transactionManager" /> 
        <!-- 配置事务属性 --> 
        <property name="transactionAttributes"> 
            <props> 
                <prop key="*">PROPAGATION_REQUIRED</prop> 
            </props> 
        </property> 
    </bean>   

    <!-- 配置DAO -->
    <bean id="userDaoTarget" class="com.bluesky.spring.dao.UserDaoImpl">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

    <bean id="userDao" parent="transactionBase" > 
        <property name="target" ref="userDaoTarget" />  
    </bean>
</beans>
```
(3）使用拦截器
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">

    <bean id="sessionFactory" 
            class="org.springframework.orm.hibernate3.LocalSessionFactoryBean"> 
        <property name="configLocation" value="classpath:hibernate.cfg.xml" /> 
        <property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
    </bean> 

    <!-- 定义事务管理器（声明式的事务） --> 
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean> 

    <bean id="transactionInterceptor" 
        class="org.springframework.transaction.interceptor.TransactionInterceptor"> 
        <property name="transactionManager" ref="transactionManager" /> 
        <!-- 配置事务属性 --> 
        <property name="transactionAttributes"> 
            <props> 
                <prop key="*">PROPAGATION_REQUIRED</prop> 
            </props> 
        </property> 
    </bean>

    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator"> 
        <property name="beanNames"> 
            <list> 
                <value>*Dao</value>
            </list> 
        </property> 
        <property name="interceptorNames"> 
            <list> 
                <value>transactionInterceptor</value> 
            </list> 
        </property> 
    </bean> 

    <!-- 配置DAO -->
    <bean id="userDao" class="com.bluesky.spring.dao.UserDaoImpl">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>
</beans>
```
（4）使用tx标签配置的拦截器
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
           http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd">

    <context:annotation-config />
    <context:component-scan base-package="com.bluesky" />

    <bean id="sessionFactory" 
            class="org.springframework.orm.hibernate3.LocalSessionFactoryBean"> 
        <property name="configLocation" value="classpath:hibernate.cfg.xml" /> 
        <property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
    </bean> 

    <!-- 定义事务管理器（声明式的事务） --> 
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" />
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="interceptorPointCuts"
            expression="execution(* com.bluesky.spring.dao.*.*(..))" />
        <aop:advisor advice-ref="txAdvice"
            pointcut-ref="interceptorPointCuts" />       
    </aop:config>     
</beans>
```
（5）全注解
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-2.5.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
           http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd">

    <context:annotation-config />
    <context:component-scan base-package="com.bluesky" />

    <tx:annotation-driven transaction-manager="transactionManager"/>

    <bean id="sessionFactory" 
            class="org.springframework.orm.hibernate3.LocalSessionFactoryBean"> 
        <property name="configLocation" value="classpath:hibernate.cfg.xml" /> 
        <property name="configurationClass" value="org.hibernate.cfg.AnnotationConfiguration" />
    </bean> 

    <!-- 定义事务管理器（声明式的事务） --> 
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

</beans>
```
此时在DAO上需加上@Transactional注解，如下：
```java
package com.bluesky.spring.dao;

import java.util.List;

import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Component;

import com.bluesky.spring.domain.User;

@Transactional
@Component("userDao")
public class UserDaoImpl extends HibernateDaoSupport implements UserDao {

    public List<User> listUsers() {
        return this.getSession().createQuery("from User").list();
    }  
}
```
### 一个声明式事务的实例
注：以下配置代码参考自(Spring事务配置的五种方式)[http://www.blogjava.net/robbie/archive/2009/04/05/264003.html]
首先是数据库表 
book(isbn, book_name, price) 
account(username, balance) 
book_stock(isbn, stock)

然后是XML配置
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:aop="http://www.springframework.org/schema/aop"
xmlns:tx="http://www.springframework.org/schema/tx"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-3.0.xsd
http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd
http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd">

    <import resource="applicationContext-db.xml" />

    <context:component-scan
        base-package="com.springinaction.transaction">
    </context:component-scan>

    <tx:annotation-driven transaction-manager="txManager"/>

    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

</beans>
```
使用的类 
BookShopDao
```java
package com.springinaction.transaction;

public interface BookShopDao {
    // 根据书号获取书的单价
    public int findBookPriceByIsbn(String isbn);
    // 更新书的库存，使书号对应的库存-1
    public void updateBookStock(String isbn);
    // 更新用户的账户余额：account的balance-price
    public void updateUserAccount(String username, int price);
}
```
