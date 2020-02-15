---
title: "Java多线程基本原理"
date: 2020-02-13T15:22:52+08:00
show_in_homepage: true
description: "Java多线程基本原理"
tags: ["Java","多线程"]
categories: ["Java多线程"]
toc: true
auto_collapse_toc: true
draft: false
---
&emsp;&emsp;Java多线程基本原理学习的笔记。其中包括**使用多线程的原因**，**Thread类简介**，**多线程问题的来源、适用场景**。

<!--more-->
## 为什么需要多线程
&emsp;&emsp;Java的执行模型是**同步**（synchronize）/**阻塞**（block）的。    
&emsp;&emsp;Java程序执行时，在默认情况下，只有一个主线程。如果主线程中有一个非常耗时的操作，线程的执行流会等待耗时操作执行完成，才会执行下一个操作。
这种执行方是虽然会使流程非常清晰、易懂，但是同时具有严重的性能问题。   
&emsp;&emsp;这种执行方式在一些场景中会大大降低CPU的利用效率，降低程序的性能，所以在一些适用场景中，我们需要使用多线程。

## Java线程--Thread线程简介
&emsp;&emsp;Thread是在Java中**最简单**，**效率略低**的开启线程的方法。  
### 创建一个Thread的两种方式
&emsp;&emsp;继承Thread类，重写run()方法。
```
public class ThreadDemo {
 public static void main(String[] args) {
        PrimeThread primeThread = new PrimeThread(143);
        primeThread.start();
    }
 static class PrimeThread extends Thread {
         long minPrime;
 
         PrimeThread(long minPrime) {
             this.minPrime = minPrime;
         }
 
         public void run() {
             System.out.println("compute primes larger than minPrime" + minPrime);
         }
     }
}
```
&emsp;&emsp;使用Thread中参数为Runnable的构造器，传入一个Runnable接口的实现类。
```
public class ThreadDemo {
 public static void main(String[] args) {
         PrimeRun primeRun = new PrimeRun(143);
         new Thread(primeRun).start();
    }
 static class PrimeRun implements Runnable {
         long minPrime;
 
         PrimeRun(long minPrime) {
             this.minPrime = minPrime;
         }
 
         public void run() {
             System.out.println("compute primes larger than minPrime" + minPrime);
         }
     }
}
```
### start方法才能并发执行
&emsp;&emsp;start()方法才能并发执行线程！！！run()方法不能！！！  
&emsp;&emsp;run()方法并不会创建一个新的线程执行run方法，执行后run中代码仍在主线程中执行。
```
public class ThreadTest {
    private static long startTime = System.currentTimeMillis();
    public static void main(String[] args) {
        new timeConsumingThread().run();
        new timeConsumingThread().run();
        new timeConsumingThread().run();
        new timeConsumingThread().run();
        long endTime = System.currentTimeMillis();
        System.out.println("调用耗时:"+(endTime-startTime));
    }
    static class timeConsumingThread extends Thread {

        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                System.out.println("线程完成耗时"+(System.currentTimeMillis()-startTime));
            }
        }
    }
}
```
![调用run()方法执行结果](/images/Java/thread/run.jpg)
&emsp;&emsp;从输出结果可以看出，调用run()方法后，代码并没有并发执行，仍然是同步/阻塞的。
```
public class TwoMethodToCreateThread {
    private static long startTime = System.currentTimeMillis();
    public static void main(String[] args) {
        new timeConsumingThread().start();
        new timeConsumingThread().start();
        new timeConsumingThread().start();
        new timeConsumingThread().start();
        long endTime = System.currentTimeMillis();
        System.out.println("调用耗时:"+(endTime-startTime));
    }
    static class timeConsumingThread extends Thread {

        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                System.out.println("线程完成耗时"+(System.currentTimeMillis()-startTime));
            }
        }
    }
}
```
![调用star()方法执行结果](/images/Java/thread/start.jpg)
&emsp;&emsp;从输出结果可以看出，调用start()方法后，代码运行是异步/非阻塞的。主线程2毫秒就执行完成，剩余四个线程几乎同时并发执行完，此时运行效率为之前的四倍。
### 方法栈是线程私有的,每多开一个线程,就多一个执行流
&emsp;&emsp;如下是相应代码对应的方法栈图解
```
public class ThreadDemo {
    public static void main(String[] args) {
        Thread thread0 = new Thread(new runnableDemo());
        Thread thread1 = new Thread(new runnableDemo());
        Thread thread2 = new Thread(new runnableDemo());
        thread0.start();
        thread1.start();
        thread2.start();
    }

    static class runnableDemo implements Runnable {
        @Override
        public void run() {
            System.out.println("do some thing.");
        }
    }
}
```
![](/images/Java/thread/method_stack.png)
&emsp;&emsp;方法栈的**局部变量**是方法栈私有的。       
&emsp;&emsp;**静态变量**/**类变量**是被所有线程共享的。
## 多线程问题的来源
&emsp;&emsp;当需要收集不同线程的执行结果时，就会用到共享变量。而共享变量就是多线程几乎所有问题坑的来源。  
&emsp;&emsp;**线程难的本质原因是，你要看着同一份代码，想象不同的人在疯狂的乱序执行它。**  
![一个进程（process）含有两个线程（threads）的运行](/images/Java/thread/220px-Multithreaded_process.svg.png)
&emsp;&emsp;线程工作时可能随时被CPU打断。如图，thread1在执行时，由于时间片耗尽进入等待状态。而thread2在thread1等待时间片轮转的过程中，已经执行完成。
如果thread1，thread2在执行过程中操作了对一个共享变量进行了非原子操作，则很可能出现线程不安全的问题。  
&emsp;&emsp;**原子操作**：一件事情在某一个时刻只能被一个线程去做。  
&emsp;&emsp;接下来，我们写一个让i作为共享变量，在Thread中自增20次的操作。
```
public class ThreadDemo {
    private static int i;

    public static void main(String[] args) {
        for (int j = 0; j < 20; j++) {
            new Thread(ThreadDemo::modifySharedVariable).start();
        }
    }
    static void modifySharedVariable() {
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        i++;
        System.out.println("i:" + i);
    }
}
```
&emsp;&emsp;这时候可怕的事情发生了，我们会发现代码每次执行的结果都不同，并且有时会不正确。
![上图代码的某次执行结果](/images/Java/thread/modifySharedVaribaleResult.png)
此时的```i++```就是一个非原子操作，分为三步完成。
- 取 i 的值
- 把 i 的值加1
- 把修改后的值写回 i<br/>
![](/images/Java/thread/ThreadModifySharedVariable.png)
&emsp;&emsp;如上图所示，虽然执行了两次对i的自增操作，但是i的值还是1。  
&emsp;&emsp;这就是多线程问题的来源。享受多线程带来的运算速率便利时，也会承担相应风险。
## 多线程的适用场景
&emsp;&emsp;一般认为一个抽象的任务有两种类型，**CPU密集型(intense)**和**IO密集型**。多线程对CPU密集型带来的提升有限，对于IO密集型及其有用。
IO密集型包括**网络IO(通常包括数据库)**、**文件IO**。  
<br/>多线程性能提升的上限 
- 单核CPU 100%  
- 多核CPU N*100%





