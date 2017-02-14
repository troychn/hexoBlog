---
title: java中并发工具包-同步器（一）
toc: true
date: 2017-02-15 19:18:23 Sunday
updated: 2017-02-15 19:49:34 Sunday
categories: [java]
tags: [java,Thread]

---
传统的多线程中并没有提供高级特性，比如：信号量、线程池、执行管理器等。而这些特殊恰恰是有助于创建更加强大的并发程序。java中并发工具处于java.util.concurrent包中，其中包括的内容包括：同步器、执行器、并发集合、Fork/join框架、atomic包和locks包。
下面介绍一下线程开发中的工具包内容：
- 同步器：是为每种特定的同步问题提供的解决方案；
- 执行器：用来管理线程执行，如：线程池
- 并发集合：提供了集合框架中集合的并发版本
- Fork/join框架：提供了对并行编程的支持
- atomic包：提供了不需要锁即可完成并发环境变量使用的原子性操作。
- locks包：使用locks接口为并发编程提供了同步的另一种替代方案，也就相当于Synchronize关键字的替代方案。

# 同步器
同步器：是为每种特定的同步问题提供的解决方案；

**信号量同步器** ：通过计数器控制对共享资源的访问  
步骤：  
semaphore(int count):创建拥有count个许可证的信号量  
acquire()/acquire(int num):获取1/num个许可证  
release()/release(int num):释放1/num个许可证  
如：

```java
public class SemaphoreDemo {
    public static void main(String [] args) {
        //编写两个柜员的信号量许可证，相当于最大允许多少线程来并发。
        Semaphore semaphore = new Semaphore(2);

        Person p1 = new Person(semaphore, "A");
        p1.start();
        Person p2 = new Person(semaphore, "B");
        p2.start();
        Person p3 = new Person(semaphore, "C");
        p3.start();
    }
}
class Person extends Thread {
    //定义一个信号量
    private Semaphore semaphore;
    public Person(Semaphore semaphore, String name) {
        setName(name);
        this.semaphore = semaphore;
    }
    public void run() {
        //等待空闲柜台来服务
        System.out.println(getName() + " is waiting ....");
        try {
            //获取信号量同步器的许可证来为我服务，是否能够获取得到柜台服务，
            // 是要看银行有多少个柜台，这里相当于定义的信号最同步器最大允许的同步数。
            semaphore.acquire();
            System.out.println(getName() + " is servicing...");
            //休眠一秒表示服务的时长
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(getName() + " is done!");
        //服务完成后，释放信号量许可给下一个客记服务。
        semaphore.release();
    }
}

```
**计数栓同步器** :必须发生指定数量的事件后才可以继续运行  
 CountDownLatch(int count):必须发生count个数量才可以打开锁存器  
 await():等待锁存器：线程可以阻塞等待这一数量到达零  
 countDown()解发事件：每被调用一次，这一数量就减一  

```java
public class CountDownLatchDemo {
    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(3);

        new Racer(countDownLatch, "A").start();
        new Racer(countDownLatch, "B").start();
        new Racer(countDownLatch, "C").start();

        //开始计数
        for (int i = 0; i < 3; i++) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(3 - i);
            countDownLatch.countDown();
            if (i == 2)
                System.out.println("Start");
        }
    }
}
/**
 * 如运动员赛跑比赛，在裁判员喊一二三，开如跑的时候，一起跑。
 */
class Racer extends Thread {
    private CountDownLatch countDownLatch;
    public Racer(CountDownLatch countDownLatch, String name) {
        setName(name);
        this.countDownLatch = countDownLatch;
    }
    public void run() {
        try {
            System.out.println(getName()+ "线程阻塞等待");
            countDownLatch.await();
            for (int i = 0; i < 3; i++)
                System.out.println(getName() + " :" + i);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}

```
**循环屏障同步器：**主要作用适用于只有多个线程都到达预定点时才可以继续执行  
 CyclicBarrier(int num):等待线程的数量  
 CyclicBarrier(int num,Runnable action):等待线程的数量以及所有线程到达后的操作  
 await():到达临界点后的暂停线程。
 
```java
public class cyclicBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, new Runnable() {
            @Override
            public void run() {
                System.out.println("Game start");
            }
        });

        new Player(cyclicBarrier, "A").start();
        new Player(cyclicBarrier, "B").start();
        new Player(cyclicBarrier, "C").start();
    }
}
/**
 * 如斗地方游戏的场景，一个人开始，等待另两个玩家开始
 */
class Player extends Thread {
    private CyclicBarrier cyclicBarrier;
    public Player(CyclicBarrier cyclicBarrier, String name) {
        setName(name);
        this.cyclicBarrier = cyclicBarrier;
    }
    public void run() {
        System.out.println(getName() + " is waiting other players....");
        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }

    }
}

```
**交换器同步器：** 简化两个线程间数据的交换  
 Exchanger<V>：指定进行交换的数据类型  
 V exchange(V object):等待线程到达，交换数据
 如：
 
```java
public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> ex = new Exchanger<String>();
        new A(ex).start();
        new B(ex).start();
    }
}

class A extends Thread {
    private Exchanger<String> ex;
    public A(Exchanger<String> ex) {
        this.ex = ex;
    }
    public void run() {
        String str = null;
        try {
            str = ex.exchange("Hello?");
            System.out.println(str);
            str = ex.exchange("A");
            System.out.println(str);
            str = ex.exchange("B");
            System.out.println(str);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
class B extends Thread {
    private Exchanger<String> ex;
    public B(Exchanger<String> ex) {
        this.ex = ex;
    }
    public void run() {
        String str = null;
        try {
            str = ex.exchange("Hi!");
            System.out.println(str);
            str = ex.exchange("1");
            System.out.println(str);
            str = ex.exchange("2");
            System.out.println(str);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```
**Phaser同步器：** 工作方式与CyclicBarrier类似，但是可以定义多个阶段  
 Phaser()/Phaser(int num):使用指定0/num个party创建phaser  
 register():注册party  
 arriveAndAwaitAdvance():到达时等待到所有party到达  
 arriveAndDeregister():到达时注销线程自己
 
 如：
 
```java
public class PhaserDemo {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(1);
        System.out.println(" starting ....");

        new Worker(phaser, "Fuwuyuan").start();
        new Worker(phaser, "Chushi").start();
        new Worker(phaser, "Shangcaiyuan").start();

        for (int i = 1; i <= 3; i++) {
            phaser.arriveAndAwaitAdvance();
            System.out.println("Order " + i + " finished!");
        }
        phaser.arriveAndDeregister();
        System.out.println("All done!");
    }
}

class Worker extends Thread {
    private Phaser phaser;
    public Worker (Phaser phaser, String name) {
        this.setName(name);
        this.phaser = phaser;
        phaser.register();
    }
    public void run() {
        for (int i = 1; i <= 3; i++) {
            System.out.println("current order is: " + i + " : " + getName());
            if (i == 3) {
                phaser.arriveAndDeregister();
            } else {
                phaser.arriveAndAwaitAdvance();
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
