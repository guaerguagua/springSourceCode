spring dao层单例模式如何保证每个线程获取的数据库连接不一样
## 问题
我们知道spring的bean加载的时候默认是单例模式，这在controller层很好理解，因为controller层没有成员变量，但是service层和dao如果是单例的话，就很奇诡。
因为dao层是需要从连接池获取一个数据库连接的，如果单例那么如何保证每个访问线程获取的数据库连接是不同的连接，如果是相同的显然是线程不安全的。
其实这里是使用了ThreadLocal类。
要弄明白这一切，又得明白事务管理在Spring中是怎么工作的，所以本文就对Spring中多线程、事务的问题进行解析。
## ThreadLocal类
ThreadLocal简单类说内部有一个map结构，存储了不同线程对应的value值，那么每个线程访问该变量的时候就是获取这个变量对应这个线程的一个副本。
API方法如下：
|方法|说明|
|----|----|
|public T get()|返回当前线程的这个thread-local变量的一个副本，如果这个值为空，将被初始化函数initialValue的实现初始化，该函数默认初始化为空|
|void set(T value) |对当前线程的这个thread-locl变量副本设定一个指定的值。|
|public void remove()|移除当前线程的这个thread-local变量的副本。|
|public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier)|创建一个thread local 变量，初始化值将由supplier的实现决定|

## 使用ThreadLocal解决dao层并发安全问题
对于每个dao对象单例持有的数据库连接，为了保证并发安全，我们必须保证每个线程获取的是不同的连接。spring其实是将数据库连接对象持有在threadlocal变量中，然后每个线程都获取到该线程对应的那个数据库连接副本。
```java
package com.yzwuestc.spring.dao;


import java.util.concurrent.atomic.AtomicInteger;

public class ThreadLocalDemo {
    //原子变量，检验是不是每个currentSession都获取了不同的session值
    private AtomicInteger sum=new AtomicInteger(0);
    private static final ThreadLocal<String> session = new ThreadLocal();
    private static AtomicInteger i =new AtomicInteger(0);
    private static String getNewSession(){
         return String.valueOf(i.getAndIncrement());
    }
    private static final String currentSession(){
        String s = session.get();
        if(s==null){
            //通过ThreadLocal让每个线程都获取不同变量副本，可以类比spring dao层获得不同的数据库连接
            s = ThreadLocalDemo.getNewSession();
            session.set(s);
        }
        return s;
    }
    //测试方法
    public  void  doSomething(){
        String a = Thread.currentThread().getName().split("-")[1];
        String b = ThreadLocalDemo.currentSession();
        //一加一减，期望sum最后为0
        sum.addAndGet(Integer.valueOf(a));
        sum.addAndGet(-Integer.valueOf(b));
        //当前线程id：获取的session id
        System.out.println(a+":"+b);
    }
    public  int getSum(){
        return sum.get();
    }
    //私有静态内部类实现单例，懒加载
    private ThreadLocalDemo(){}
    private static class SingletonInstance{
        private static final ThreadLocalDemo INSTANCE = new ThreadLocalDemo();
    }
    public static ThreadLocalDemo getInstance(){
        return SingletonInstance.INSTANCE;
    }
    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
               ThreadLocalDemo.getInstance().doSomething();
            }
        };
        for(int i = 0;i<30000;i++) {
            new Thread(runnable).start();
        }
        Thread.sleep(3000);
        System.out.println(ThreadLocalDemo.getInstance().getSum());

    }
}
```
以上例子中启动一个线程，在run方法中调用ThreadLocalDemo.getInstance()获取单例对象（使用私有静态内部类的方式），再调用doSomething方法。
这里打印出当前线程名称和持有的session（可以认为就是数据库的连接），ThreadLocalDemo.currentSession()里调用一个ThreadLocal变量session的get方法，在第一获取的时候为空，则ThreadLocalDemo.getNewSession()调用新建一个session，getNewSession内部用一个原子变量自增（保证线程安全），每调用一次就获取一个新的值，表示一个连接的id，在调用session.set方法设置当前线程的thread-local变量副本。在main函数里启动多个线程，可以看到每个线程打印出来的当前线程id和session id都是一一对应的，数据量太多不好判断，在doSomethin内部使用一加一减的技巧，最后等待3秒，看总和为0则说明当前线程id和session id都是一一对应的。

注意：如果getNewSession不是线程安全的，那么不同线程可能获取到同样的session id。
