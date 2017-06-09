---
title: 生产者-消费者模式 系列 之二 ReentrantLock, Condition 篇
date: 2016-06-06
categories: javaSE
tags: JavaConcurrent
---

这篇文章将主要集中在JDK 1.5中引入的 lock 机制和其对应的condition 条件变量.  将通过ReentrantLock 来实现系列一中同样功能的生产者-消费者模式.
Lock :  Lock概念的出现,乍一看是把Java中一直使用的隐式对象锁(可参考系列一文章synchronized中锁类型的实验例子)以一种独立明确的方式进行了定义. 使得在多线程环境中对共享资源的访问控制,除synchronized之外,又多了一种选择. 但仔细琢磨你会发现, Lock还有很多synchronized无法比拟的优势.

* Lock 提供了tryLock方法,这样一旦锁被其他线程占用, 调用该方法不至于导致线程被阻塞. 而synchronized 会导致线程阻塞.
*  Lock 有不同的实现. 比如我们即将用到的reentrantLock ,它实现的是一个排它锁; ReadWriteLock实现的是读写锁,适用于大量读而少量写的场景.
* 通过调用Lock.newCondition() 方法, 可以获得绑定在该Lock上的条件变量. 从而对线程的阻塞和唤醒有更灵活的控制. 在这个例子中会有展现.
当然,事物都要两面性,Lock同样也不例外. 它的引入也会带来一些负作用. 比如
* 必须用户显示地调用Lock 去获得锁,然后还要确保unlock被调用从而释放锁. 于是就有了下面比较繁琐的代码段. 感觉Java 在走C++的路,想想垃圾回收.
```
try{   
lock.lock();   
//do something  
} finally{  
lock.unlock();  
} 
```
* Lock 同样也是对象,这样它本身也有notify 和 wait 方法. 看看如下代码是否很费解?  
```
private Lock lock = new ReentrantLock();  
synchronized(lock){  
  //do something  
  lock.notify();  
} 
```

* ReentrantLock : Lock 接口的一个实现. 一个互斥锁. 基本上就是synchronized 的替代品. 但作为Lock接口的实现也继承了其优势. 在对阻塞进程进行唤醒时比object.notify 多了一些自主权.  Object.notify方法不能指定[算法](http://lib.csdn.net/base/datastructure)说哪个线程将被唤醒. 但 ReentrantLock 由于在构造函数中有 fairness 参数,当为true时,可以将等待时间最长的线程优先唤醒.
*  Condition :  此概念的出现,将Object的Notify,wait方法进行了独立. 并且可以和Lock对象通过Lock.newCondition() 方法自由绑定. 控制更加灵活. 另外 Condition.await 和Condition.signal 方法都是原子操作.  Condition 概念理解起来不太容易. 尤其是看到JDK自带的Javadoc 关于BoundedBuffer的示例之后,里面的notFull 和notEmpty相当拗口. 但仔细琢磨你会发现它原来就是一个条件变量 相当于说 

```
if ( condition variable == true )   
  signal;   
else  
  await;  
end;  
```

所以, 在命名Condition 变量时, 要根据你的上下文,给它赋予一个有意义,好辩识的名字.
 
 生产者消费者场景假定
 * 同时只能有一个生产者进行生产.生产的同时,不能有消费者在消费.
 * 同时只有一个消费者在消费.消费的同时,不能有生产者生产.
 * 生产者最多能生产10个产品. 如果当前产品数超过10个,生产者将等待直到产品数小于10才开始生产.
 * 当前如果没有可消费产品时,消费者将等待直到有产品可消费为止.

```
import java.util.Stack;  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.Lock;  
import java.util.concurrent.locks.ReentrantLock;  
  
public class ReentrantLockCPTest {  
  
  
  
    public static void main(String[] args) {  
          
        Repository s = new Repository();  
        Maker f1 = new Maker(s,"P001");  
        Maker f2 = new Maker(s,"P002");  
          
        f1.start();  
        f2.start();  
  
        Taker c1 = new Taker(s,"C001");  
        Taker c2 = new Taker(s,"C002");  
        Taker c3 = new Taker(s,"C003");  
        c1.start();  
        c2.start();  
        c3.start();  
          
    }  
  
}  
class Repository{  
      
    private  final static int MAX_ELEMENT = 10;  
      
    private Stack<String> _store = new Stack<String>();  
      
    private Lock lock = new ReentrantLock();  
    private final Condition notEmpty = lock.newCondition();  
    private final Condition notFull = lock.newCondition();  
  
      
      
    public  void add(String in) throws InterruptedException{  
          
        lock.lock();  
          
        try{  
            while (this._store.size() >= MAX_ELEMENT) {//这里一定是Loop 循环,因为被阻塞的生产者线程被唤醒后要继续执行,但之前必须判断产品库是否已满.  
                System.out.println(Thread.currentThread().getName() + " is waiting on add.");  
                notFull.await(); //如果参数中数量达到10个的最大值.生产者线程等待.  
                System.out.println(Thread.currentThread().getName() + " is after waiting on add.");  
            }  
            this._store.push(in);  
            System.out.println(Thread.currentThread().getName() + " is adding product "+ in+". Remaining size is "+ this._store.size());  
            notEmpty.signal(); //唤醒那些因为产品库为零时等待的消费者进程.  
  
        }finally{  
            lock.unlock();  
        }  
  
  
  
  
    }  
      
    public   String  get() throws InterruptedException{  
        lock.lock();  
        String rtn = "";  
          
        try{  
              
            while (this._store.isEmpty()) {//这里一定是Loop 循环,因为被阻塞的消费者线程被唤醒后要继续执行,但之前必须判断产品库是否为空.  
                System.out.println(Thread.currentThread().getName() + " is waiting on get.");  
                notEmpty.await();//如果产品库为空,消费者线程等待.直到产品库中有产品时被唤醒.  
                System.out.println(Thread.currentThread().getName() + " is after waiting on get.");  
                  
            }  
              
            rtn = this._store.pop();  
            System.out.println(Thread.currentThread().getName() + " is getting product "+ rtn+". Remaining size is "+ this._store.size());  
            notFull.signal();//唤醒因产品库为空可能导致等待的生产者线程.  
              
        }finally{  
            lock.unlock();  
        }  
      
  
  
        return rtn;  
    }  
  
}  
  
class Maker implements Runnable {  
         
    private Repository _store;  
    private String _name;  
    private Thread _thread;  
    public Maker(Repository s,String name) {  
    super();  
    this._store = s;  
    this._name = name;  
    this._thread = new Thread(this,name);  
    }  
      
    public void start(){  
        this._thread.start();  
    }  
      
    @Override  
    public void run() {  
        int i = 0;  
        while(true){  
            try {  
                this._store.add(this._name + " Product "+ ++i);  
                Thread.sleep(1000);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            } catch (Exception e){  
                e.printStackTrace();  
            }  
        }  
  
    }  
  
}  
  
  
class Taker implements Runnable {  
        
    private Repository _store;  
    private Thread _thread;  
    public Taker(Repository s,String name) {  
        super();  
        this._store =s;  
        this._thread = new Thread(this,name);  
    }  
    public void start(){  
        this._thread.start();  
    }  
      
    @Override  
    public void run() {  
        while(true){  
            try {  
                this._store.get();  
                Thread.sleep(5000);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }catch (Exception e) {  
                e.printStackTrace();  
            }  
        }  
  
    }  
  
}  
```

运行结果：
```
P002 is adding product P002 Product 1. Remaining size is 1  
P001 is adding product P001 Product 1. Remaining size is 2  
C002 is getting product P001 Product 1. Remaining size is 1  
C001 is getting product P002 Product 1. Remaining size is 0  
C003 is waiting on get.  
P001 is adding product P001 Product 2. Remaining size is 1  
P002 is adding product P002 Product 2. Remaining size is 2  
C003 is after waiting on get.  
C003 is getting product P002 Product 2. Remaining size is 1  
P002 is adding product P002 Product 3. Remaining size is 2  
P001 is adding product P001 Product 3. Remaining size is 3  
P001 is adding product P001 Product 4. Remaining size is 4  
P002 is adding product P002 Product 4. Remaining size is 5  
P002 is adding product P002 Product 5. Remaining size is 6  
P001 is adding product P001 Product 5. Remaining size is 7  
C001 is getting product P001 Product 5. Remaining size is 6  
P002 is adding product P002 Product 6. Remaining size is 7  
C002 is getting product P002 Product 6. Remaining size is 6  
P001 is adding product P001 Product 6. Remaining size is 7  
C003 is getting product P001 Product 6. Remaining size is 6  
P002 is adding product P002 Product 7. Remaining size is 7  
P001 is adding product P001 Product 7. Remaining size is 8  
P002 is adding product P002 Product 8. Remaining size is 9  
P001 is adding product P001 Product 8. Remaining size is 10  
P002 is waiting on add.  
P001 is waiting on add.  
C001 is getting product P001 Product 8. Remaining size is 9  
C002 is getting product P002 Product 8. Remaining size is 8  
P002 is after waiting on add.  
````