---
title: 生产者-消费者模式 系列 之一 Sychronized,Notify,Wait 篇
date: 2016-05-06
categories: javaSE
tags: JavaConcurrent
---

生产者,消费者模式可谓是Java多线程中比较经典的例子.该系列文章希望以该模式的实现为起点,将Java中对于多线程同步和通讯技术做一个总结.这第一个坑肯定要留给包括 Sychronized , Notify,和Wait 方法.关于这几个技术点的介绍,网上一搜一大把,这里不多说.只是把我自己学习过程中的一些理解难点做个说明,希望对看到这篇文章的人有所帮助.
     Sychronized :  知道锁一般包括类级锁和对象级锁.锁类型不同,结果也是不同的.  实验一下下面的代码就知道了.看注释.
```
public class SynchronziedTest {  
  
    public static void main(String[] args){  
        C c1 = new C(new Share(),"Consumer01");//注意,这里每个线程有不同的Share实例.  
        C c2 = new C(new Share(),"Consumer02");  
        C c3 = new C(new Share(),"Consumer03");  
        P p1 = new P(new Share(),"Producer01");  
        c1.start();  
        c2.start();  
        c3.start();  
        p1.start();  
          
    }  
}  
class Share{  
      
    private static int i;  
    private final static Object _lock = new Object();  
      
    public void add(){  
        synchronized(this._lock){  //将此处改为this.getClass()效果相同,因为他们获得的是类级锁.更改为this就错了,它获得是对象锁.   
          ++i;  
          System.out.println(Thread.currentThread().getName() + " Remaining Size = " + i);  
        }  
          
    }  
    public void get(){  
        synchronized(this._lock){ //将此处改为this.getClass()效果相同,因为他们获得的是类级锁.更改为this就错了,它获得是对象锁.            if (i > 0){  
                try {  
                    Thread.sleep(1000); //让线程睡会,更改为synchronized(this)后,增大别的线程(i>0)为true 的机会.  
                } catch (InterruptedException e) {  
                    // TODO Auto-generated catch block  
                    e.printStackTrace();  
                }  
                --i;  
                System.out.println(Thread.currentThread().getName() + " Remaining Size = " + i);  
            }  
              
        }  
    }  
}  
class C extends Thread{  
    private Share _repository;  
    public C(Share s,String threadName){  
        super(threadName);  
        this._repository = s;  
    }  
    public void run() {  
         while(true){  
             this._repository.get();  
             try {  
                Thread.sleep(1000);  
            } catch (InterruptedException e) {  
                // TODO Auto-generated catch block  
                e.printStackTrace();  
            }  
         }  
    }  
}  
  
class P extends Thread{  
    private Share _repository;  
    public P(Share s,String threadName){  
        super(threadName);  
        this._repository = s;  
          
    }  
    public void run() {  
        while(true){  
            this._repository.add();  
            try {  
                Thread.sleep(1000);  
            } catch (InterruptedException e) {  
                // TODO Auto-generated catch block  
                e.printStackTrace();  
            }  
        }  
    }  
}  
```
Wait :  阻塞当前持有调用该方法的对象的锁的线程,并释放锁.当被同一个对象的notify方法唤醒时,只有获得CPU资源后,该线程将从调用wait方法执行点继续执行.
Notify : 通知被同一个对象的wait方法所阻塞的一个线程.使其在获得CPU资源后从上次的执行点(即wait方法处)继续执行.
 
* 生产者消费者场景假定
* 同时只能有一个生产者进行生产.生产的同时,不能有消费者在消费.
* 同时只有一个消费者在消费.消费的同时,不能有生产者生产.
* 生产者最多能生产10个产品. 如果当前产品数超过10个,生产者将等待直到产品数小于10才开始生产.
* 当前如果没有可消费产品时,消费者将等待直到有产品可消费为止.

```
import java.util.Stack;  
  
public class TypicalCPTest {  
  
  
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
      
    public  void add(String in) throws InterruptedException{  
          
        synchronized(this){  
            while (this._store.size() >= MAX_ELEMENT) {  
                System.out.println(Thread.currentThread().getName() + " is waiting on add.");  
                this.wait(); //如果参数中数量达到10个的最大值.生产者线程等待.  
                System.out.println(Thread.currentThread().getName() + " is after waiting on add.");  
            }  
            this._store.push(in);  
            System.out.println(Thread.currentThread().getName() + " is adding product "+ in+". Remaining size is "+ this._store.size());  
            this.notify(); //唤醒那些因为产品库为零时等待的消费者进程.  
        }  
  
  
    }  
      
    public   String  get() throws InterruptedException{  
        String rtn = "";  
        synchronized(this){  
            while (this._store.isEmpty()) {  
                System.out.println(Thread.currentThread().getName() + " is waiting on get.");  
                this.wait();//如果产品库为空,消费者线程等待.直到产品库中有产品时被唤醒.  
                System.out.println(Thread.currentThread().getName() + " is after waiting on get.");  
                  
            }  
              
            rtn = this._store.pop();  
            System.out.println(Thread.currentThread().getName() + " is getting product "+ rtn+". Remaining size is "+ this._store.size());  
            this.notify();//唤醒因产品库为空可能导致等待的生产者线程.  
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
运行结果:
```
P001 is adding product P001 Product 1. Remaining size is 1  
C001 is getting product P001 Product 1. Remaining size is 0  
P002 is adding product P002 Product 1. Remaining size is 1  
C003 is getting product P002 Product 1. Remaining size is 0  
C002 is waiting on get.  
P002 is adding product P002 Product 2. Remaining size is 1  
C002 is after waiting on get.  
C002 is getting product P002 Product 2. Remaining size is 0  
P001 is adding product P001 Product 2. Remaining size is 1  
P001 is adding product P001 Product 3. Remaining size is 2  
P002 is adding product P002 Product 3. Remaining size is 3  
P002 is adding product P002 Product 4. Remaining size is 4  
P001 is adding product P001 Product 4. Remaining size is 5  
P001 is adding product P001 Product 5. Remaining size is 6  
P002 is adding product P002 Product 5. Remaining size is 7  
P001 is adding product P001 Product 6. Remaining size is 8  
P002 is adding product P002 Product 6. Remaining size is 9  
C003 is getting product P002 Product 6. Remaining size is 8  
C001 is getting product P001 Product 6. Remaining size is 7  
C002 is getting product P002 Product 5. Remaining size is 6  
P002 is adding product P002 Product 7. Remaining size is 7  
P001 is adding product P001 Product 7. Remaining size is 8  
P002 is adding product P002 Product 8. Remaining size is 9  
P001 is adding product P001 Product 8. Remaining size is 10  
P001 is waiting on add.  
P002 is waiting on add.  
C001 is getting product P001 Product 8. Remaining size is 9  
P001 is after waiting on add.  
P001 is adding product P001 Product 9. Remaining size is 10  
P002 is after waiting on add.  
P002 is waiting on add.  
```