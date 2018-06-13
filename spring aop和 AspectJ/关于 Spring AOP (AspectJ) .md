>【版权申明】未经博主同意，谢绝转载！（请尊重原创，博主保留追究权） http://blog.csdn.net/javazejian/article/details/54629058 出自【zejian的博客】

# 关于 Spring AOP (AspectJ) 你该知晓的一切
## OOP的新生机
### OOP新生机前夕
OOP即面向对象的程序设计，谈起了OOP，我们就不得不了解一下POP即面向过程程序设计，它是以功能为中心来进行思考和组织的一种编程方式，强调的是系统的数据被加工和处理的过程，说白了就是注重功能性的实现，效果达到就好了，而OOP则注重封装，强调整体性的概念，以对象为中心，将对象的内部组织与外部环境区分开来。之前看到过一个很贴切的解释，博主把它们画成一幅图如下：
![](./1.png)
在这里我们暂且把程序设计比喻为房子的布置，一间房子的布局中，需要各种功能的家具和洁具（类似方法），如马桶、浴缸、天然气灶，床、桌子等，对于面向过程的程序设计更注重的是功能的实现(即功能方法的实现)，效果符合预期就好，因此面向过程的程序设计会更倾向图1设置结构，各种功能都已实现，房子也就可以正常居住了。但对于面向对象的程序设计则是无法忍受的，这样的设置使房子内的各种家具和洁具间摆放散乱并且相互暴露的机率大大增加，各种气味相互参杂，显然是很糟糕的，于是为了更优雅地设置房屋的布局，面向对象的程序设计便采用了图2的布局，对于面向对象程序设计来说这样设置好处是显而易见的，房子中的每个房间都有各自的名称和相应功能(在java程序设计中一般把类似这样的房间称为类，每个类代表着一种房间的抽象体)，如卫生间是大小解和洗澡梳妆用的，卧室是休息用的，厨房则是做饭用的，每个小房间都各司其职并且无需时刻向外界暴露内部的结构，整个房间结构清晰，外界只需要知道这个房间并使用房间内提供的各项功能即可(方法调用)，同时也更有利于后期的拓展了，毕竟哪个房间需要添加那些功能，其范围也有了限制，也就使职责更加明确了(单一责任原则)。OOP的出现对POP确实存在很多颠覆性的，但并不能说POP已没有价值了，毕竟只是不同时代的产物，从方法论来讲，更喜欢将面向过程与面向对象看做是事物的两个方面–局部与整体（你必须要注意到局部与整体是相对的），因此在实际应用中，两者方法都同样重要。了解完OOP和POP各自的特点，接着看java程序设计过程中OOP应用，在java程序设计过程中，我们几乎享尽了OOP设计思想带来的甜头，以至于在这个一切皆对象，众生平等的世界里，狂欢不已，而OOP确实也遵循自身的宗旨即将数据及对数据的操作行为放在一起，作为一个相互依存、不可分割的整体，这个整体美其名曰：对象，利用该定义对于相同类型的对象进行分类、抽象后，得出共同的特征，从而形成了类，在java程序设计中这些类就是class，由于类(对象)基本都是现实世界存在的事物概念（如前面的不同的小房间）因此更接近人们对客观事物的认识，同时把数据和方法(算法)封装在一个类(对象)中，这样更有利于数据的安全，一般情况下属性和算法只单独属于某个类，从而使程序设计更简单，也更易于维护。基于这套理论思想，在实际的软件开发中，整个软件系统事实也是由系列相互依赖的对象所组成，而这些对象也是被抽象出来的类。相信大家在实际开发中是有所体验的(本篇文件假定读者已具备面向对象的开发思想包括封装、继承、多态的知识点)。但随着软件规模的增大，应用的逐渐升级，慢慢地，OOP也开始暴露出一些问题，现在不需要急于知道它们，通过案例，我们慢慢感受：
A类：
```java
public class A {
    public void executeA(){
        //其他业务操作省略......
        recordLog();
    }

    public void recordLog(){
        //....记录日志并上报日志系统
    }
}
```
B类：
```java
public class B {
    public void executeB(){
        //其他业务操作省略......
        recordLog();
    }

    public void recordLog(){
        //....记录日志并上报日志系统
    }
}
```
C类：
```java
public class C {
    public void executeC(){
        //其他业务操作省略......
        recordLog();
    }

    public void recordLog(){
        //....记录日志并上报日志系统
    }
}
```
假设存在A、B、C三个类，需要对它们的方法访问进行日志记录，在代码中各种存在recordLog方法进行日志记录并上报，或许对现在的工程师来说几乎不可能写出如此糟糕的代码，但在OOP这样的写法是允许的，而且在OOP开始阶段这样的代码确实并大量存在着，直到工程师实在忍受不了一次修改，到处挖坟时（修改recordLog内容），才下定决心解决该问题，为了解决程序间过多冗余代码的问题，工程师便开始使用下面的编码方式
```java
//A类
public class A {
    public void executeA(){
        //其他业务操作省略...... args 参数，一般会传递类名，方法名称 或信息（这样的信息一般不轻易改动）
        Report.recordLog(args ...);
    }
}

//B类
public class B {
    public void executeB(){
        //其他业务操作省略......
        Report.recordLog(args ...);
    }
}

//C类
public class C {
    public void executeC(){
        //其他业务操作省略......
        Report.recordLog(args ...);
    }
}

//record
public class Report {
    public static void recordLog(args ...){
        //....记录日志并上报日志系统
    }
}
```
这样操作后，我们欣喜地发现问题似乎得到了解决，当上报信息内部方法需要调整时，只需调整Report类中recordLog方法体，也就避免了随处挖坟的问题，大大降低了软件后期维护的复杂度。确实如此，而且除了上述的解决方案，还存在一种通过继承来解决的方式，采用这种方式，只需把相通的代码放到一个类(一般是其他类的父类)中，其他的类（子类）通过继承父类获取相通的代码，如下：
```java
//通用父类
public class Dparent {
    public void commond(){
        //通用代码
    }
}
//A 继承 Dparent 
public class A extends Dparent {
    public void executeA(){
        //其他业务操作省略......
        commond();
    }
}
//B 继承 Dparent 
public class B extends Dparent{
    public void executeB(){
        //其他业务操作省略......
        commond();
    }
}
//C 继承 Dparent 
public class C extends Dparent{
    public void executeC(){
        //其他业务操作省略......
        commond();
    }
}
```
显然代码冗余也得到了解决，这种通过继承抽取通用代码的方式也称为纵向拓展，与之对应的还有横向拓展(现在不需急于明白，后面的分析中它将随处可见)。事实上有了上述两种解决方案后，在大部分业务场景的代码冗余问题也得到了实实在在的解决，原理如下图
![](./2.png)
但是随着软件开发的系统越来越复杂，工程师认识到，传统的OOP程序经常表现出一些不自然的现象，核心业务中总掺杂着一些不相关联的特殊业务，如日志记录，权限验证，事务控制，性能检测，错误信息检测等等，这些特殊业务可以说和核心业务没有根本上的关联而且核心业务也不关心它们，比如在用户管理模块中，该模块本身只关心与用户相关的业务信息处理，至于其他的业务完全可以不理会，我们看一个简单例子协助理解这个问题
```java
/**
 * Created by zejian on 2017/2/15.
 * Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 */
public interface IUserService {

    void saveUser();

    void deleteUser();

    void findAllUser();
}
//实现类
public class UserServiceImpl implements IUserService {

    //核心数据成员

    //日志操作对象

    //权限管理对象

    //事务控制对象

    @Override
    public void saveUser() {

        //权限验证(假设权限验证丢在这里)

        //事务控制

        //日志操作

        //进行Dao层操作
        userDao.saveUser();

    }

    @Override
    public void deleteUser() {

    }

    @Override
    public void findAllUser() {

    }
}
```
上述代码中我们注意到一些问题，权限，日志，事务都不是用户管理的核心业务，也就是说用户管理模块除了要处理自身的核心业务外，还需要处理权限，日志，事务等待这些杂七杂八的不相干业务的外围操作，而且这些外围操作同样会在其他业务模块中出现，这样就会造成如下问题

* 代码混乱：核心业务模块可能需要兼顾处理其他不相干的业务外围操作，这些外围操作可能会混乱核心操作的代码，而且当外围模块有重大修改时也会影响到核心模块，这显然是不合理的。

* 代码分散和冗余：同样的功能代码，在其他的模块几乎随处可见，导致代码分散并且冗余度高。

* 代码质量低扩展难：由于不太相关的业务代码混杂在一起，无法专注核心业务代码，当进行类似无关业务扩展时又会直接涉及到核心业务的代码，导致拓展性低。

显然前面分析的两种解决方案已束手无策了，那么该如何解决呢？事实上我们知道诸如日志，权限，事务，性能监测等业务几乎涉及到了所有的核心模块，如果把这些特殊的业务代码直接到核心业务模块的代码中就会造成上述的问题，而工程师更希望的是这些模块可以实现热插拔特性而且无需把外围的代码入侵到核心模块中，这样在日后的维护和扩展也将会有更佳的表现，假设现在我们把日志、权限、事务、性能监测等外围业务看作单独的关注点(也可以理解为单独的模块)，每个关注点都可以在需要它们的时刻及时被运用而且无需提前整合到核心模块中，这种形式相当下图：
![](./3.png)
从图可以看出，每个关注点与核心业务模块分离，作为单独的功能，横切几个核心业务模块，这样的做的好处是显而易见的，每份功能代码不再单独入侵到核心业务类的代码中，即核心模块只需关注自己相关的业务，当需要外围业务(日志，权限，性能监测、事务控制)时，这些外围业务会通过一种特殊的技术自动应用到核心模块中，这些关注点有个特殊的名称，叫做“横切关注点”，上图也很好的表现出这个概念，另外这种抽象级别的技术也叫AOP（面向切面编程），正如上图所展示的横切核心模块的整面，因此AOP的概念就出现了，而所谓的特殊技术也就面向切面编程的实现技术，AOP的实现技术有多种，其中与Java无缝对接的是一种称为AspectJ的技术。那么这种切面技术（AspectJ）是如何在Java中的应用呢？不必担心，也不必全面了解AspectJ，本篇博文也不会这样进行，对于AspectJ，我们只会进行简单的了解，从而为理解Spring中的AOP打下良好的基础(Spring AOP 与AspectJ 实现原理上并不完全一致，但功能上是相似的，这点后面会分析)，毕竟Spring中已实现AOP主要功能，开发中直接使用Spring中提供的AOP功能即可，除非我们想单独使用AspectJ的其他功能。这里还需要注意的是，AOP的出现确实解决外围业务代码与核心业务代码分离的问题，但它并不会替代OOP，如果说OOP的出现是把编码问题进行模块化，那么AOP就是把涉及到众多模块的某一类问题进行统一管理，因此在实际开发中AOP和OOP同时存在并不奇怪，后面将会慢慢体会带这点，好的，已迫不及待了，让我们开始了解神一样的AspectJ吧。
### 神一样的AspectJ-AOP的领跑者
这里先进行一个简单案例的演示，然后引出AOP中一些晦涩难懂的抽象概念，放心，通过本篇博客，我们将会非常轻松地理解并掌握它们。编写一个HelloWord的类，然后利用AspectJ技术切入该类的执行过程。
```java
/**
 * Created by zejian on 2017/2/15.
 * Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 */
public class HelloWord {

    public void sayHello(){
        System.out.println("hello world !");
    }
    public static void main(String args[]){
        HelloWord helloWord =new HelloWord();
        helloWord.sayHello();
    }
}
```
编写AspectJ类，注意关键字为aspect(MyAspectJDemo.aj,其中aj为AspectJ的后缀)，含义与class相同，即定义一个AspectJ的类
```java
/**
 * Created by zejian on 2017/2/15.
 * Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 * 切面类
 */
public aspect MyAspectJDemo {
    /**
     * 定义切点,日志记录切点
     */
    pointcut recordLog():call(* HelloWord.sayHello(..));

    /**
     * 定义切点,权限验证(实际开发中日志和权限一般会放在不同的切面中,这里仅为方便演示)
     */
    pointcut authCheck():call(* HelloWord.sayHello(..));

    /**
     * 定义前置通知!
     */
    before():authCheck(){
        System.out.println("sayHello方法执行前验证权限");
    }

    /**
     * 定义后置通知
     */
    after():recordLog(){
        System.out.println("sayHello方法执行后记录日志");
    }
}
```
ok~，运行helloworld的main函数：
![](./4.png)
>我这里报了一些错误，可能是编译器配置的问题。
>![](./5.png)
对于结果不必太惊讶，完全是意料之中。我们发现，明明只运行了main函数，却在sayHello函数运行前后分别进行了权限验证和日志记录，事实上这就是AspectJ的功劳了。对aspectJ有了感性的认识后，再来聊聊aspectJ到底是什么？AspectJ是一个java实现的AOP框架，它能够对java代码进行AOP编译（一般在编译期进行），让java代码具有AspectJ的AOP功能（当然需要特殊的编译器），可以这样说AspectJ是目前实现AOP框架中最成熟，功能最丰富的语言，更幸运的是，AspectJ与java程序完全兼容，几乎是无缝关联，因此对于有java编程基础的工程师，上手和使用都非常容易。在案例中，我们使用aspect关键字定义了一个类，这个类就是一个切面，它可以是单独的日志切面(功能)，也可以是权限切面或者其他，在切面内部使用了pointcut定义了两个切点，一个用于权限验证，一个用于日志记录，而所谓的切点就是那些需要应用切面的方法，如需要在sayHello方法执行前后进行权限验证和日志记录，那么就需要捕捉该方法，而pointcut就是定义这些需要捕捉的方法（常常是不止一个方法的），这些方法也称为目标方法，最后还定义了两个通知，通知就是那些需要在目标方法前后执行的函数，如before()即前置通知在目标方法之前执行，即在sayHello()方法执行前进行权限验证，另一个是after()即后置通知，在sayHello()之后执行，如进行日志记录。到这里也就可以确定，切面就是切点和通知的组合体，组成一个单独的结构供后续使用。
这里简单说明一下切点的定义语法：关键字为pointcut，定义切点，后面跟着函数名称，最后编写匹配表达式，此时函数一般使用call()或者execution()进行匹配，这里我们统一使用call()
>pointcut 函数名 : 匹配表达式
案例：recordLog()是函数名称，自定义的，* 表示任意返回值，接着就是需要拦截的目标函数，sayHello(..)的..，表示任意参数类型。这里理解即可，后面Spring AOP会有关于切点表达式的分析，整行代码的意思是使用pointcut定义一个名为recordLog的切点函数，其需要拦截的(切入)的目标方法是HelloWord类下的sayHello方法，参数不限。
关于定义通知的语法：首先通知有5种类型分别如下：

* before 目标方法执行前执行，前置通知
* after 目标方法执行后执行，后置通知
* after returning 目标方法返回时执行 ，后置返回通知
* after throwing 目标方法抛出异常时执行 异常通知
* around 在目标函数执行中执行，可控制目标函数是否执行，环绕通知
语法：
>[返回值类型] 通知函数名称(参数) [returning/throwing 表达式]：连接点函数(切点函数){ 函数体 }
案例如下，其中要注意around通知即环绕通知，可以通过proceed()方法控制目标函数是否执行。
```java
/**
  * 定义前置通知
  *
  * before(参数):连接点函数{
  *     函数体
  * }
  */
 before():authCheck(){
     System.out.println("sayHello方法执行前验证权限");
 }

 /**
  * 定义后置通知
  * after(参数):连接点函数{
  *     函数体
  * }
  */
 after():recordLog(){
     System.out.println("sayHello方法执行后记录日志");
 }


 /**
  * 定义后置通知带返回值
  * after(参数)returning(返回值类型):连接点函数{
  *     函数体
  * }
  */
 after()returning(int x): get(){
     System.out.println("返回值为:"+x);
 }

 /**
  * 异常通知
  * after(参数) throwing(返回值类型):连接点函数{
  *     函数体
  * }
  */
 after() throwing(Exception e):sayHello2(){
     System.out.println("抛出异常:"+e.toString());
 }



 /**
  * 环绕通知 可通过proceed()控制目标函数是否执行
  * Object around(参数):连接点函数{
  *     函数体
  *     Object result=proceed();//执行目标函数
  *     return result;
  * }
  */
 Object around():aroundAdvice(){
     System.out.println("sayAround 执行前执行");
     Object result=proceed();//执行目标函数
     System.out.println("sayAround 执行后执行");
     return result;
 }
```
切入点(pointcut)和通知(advice)的概念已比较清晰，而切面则是定义切入点和通知的组合如上述使用aspect关键字定义的MyAspectJDemo，把切面应用到目标函数的过程称为织入(weaving)。在前面定义的HelloWord类中除了sayHello函数外，还有main函数，以后可能还会定义其他函数，而这些函数都可以称为目标函数，也就是说这些函数执行前后也都可以切入通知的代码，这些目标函数统称为连接点，切入点(pointcut)的定义正是从这些连接点中过滤出来的，下图协助理解。
![](./6.png)
### AspectJ的织入方式及其原理概要
经过前面的简单介绍，我们已初步掌握了AspectJ的一些语法和概念，但这样仍然是不够的，我们仍需要了解AspectJ应用到java代码的过程（这个过程称为织入），对于织入这个概念，可以简单理解为aspect(切面)应用到目标函数(类)的过程。对于这个过程，一般分为动态织入和静态织入，动态织入的方式是在运行时动态将要增强的代码织入到目标类中，这样往往是通过动态代理技术完成的，如Java JDK的动态代理(Proxy，底层通过反射实现)或者CGLIB的动态代理(底层通过继承实现)，Spring AOP采用的就是基于运行时增强的代理技术，这点后面会分析，这里主要重点分析一下静态织入，ApectJ采用的就是静态织入的方式。ApectJ主要采用的是编译期织入，在这个期间使用AspectJ的acj编译器(类似javac)把aspect类编译成class字节码后，在java目标类编译时织入，即先编译aspect类再编译目标类。
![](./7.png)
关于ajc编译器，是一种能够识别aspect语法的编译器，它是采用java语言编写的，由于javac并不能识别aspect语法，便有了ajc编译器，注意ajc编译器也可编译java文件。为了更直观了解aspect的织入方式，我们打开前面案例中已编译完成的HelloWord.class文件，反编译后的java代码如下：
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.zejian.demo;

import com.zejian.demo.MyAspectJDemo;
//编译后织入aspect类的HelloWord字节码反编译类
public class HelloWord {
    public HelloWord() {
    }

    public void sayHello() {
        System.out.println("hello world !");
    }

    public static void main(String[] args) {
        HelloWord helloWord = new HelloWord();
        HelloWord var10000 = helloWord;

   try {
        //MyAspectJDemo 切面类的前置通知织入
        MyAspectJDemo.aspectOf().ajc$before$com_zejian_demo_MyAspectJDemo$1$22c5541();
        //目标类函数的调用
           var10000.sayHello();
        } catch (Throwable var3) {
        MyAspectJDemo.aspectOf().ajc$after$com_zejian_demo_MyAspectJDemo$2$4d789574();
            throw var3;
        }

        //MyAspectJDemo 切面类的后置通知织入 
        MyAspectJDemo.aspectOf().ajc$after$com_zejian_demo_MyAspectJDemo$2$4d789574();
    }
}
```