## Java线程基础

Java中新建一个线程有如下两种基本方法：

* 使用实现 ***Runnable*** 接口的类构造 ***Thread*** 类对象，调用Thread对象的 ***start()*** 方法运行新线程：

```java
public interface Runnable {
    void run();
}

// 线程建立示例
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("This is a new Thread");
    }    
}

public class RunnableTest {
    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable());
        thread.start();
    }
}
```

* 继承Thread类，调用Thread对象的start方法运行新线程。

前一种方式更适用于目前的情况，原因如下：

1. 大部分类只想新建立一个线程，继承Thread类会拥有诸多无用的成员。
2. Java不能多继承，但可以实现多个接口。



```java
// java.lang.Thread 1.0

Thread(Runnable target);

void start();
// 启动该线程，引发调用Thread类对象的run()方法(会调用关联Runnable的run方法)。该方法立即返回。
```

---

### 线程中断

除了正常运行结束之外，线程运行过程中出现的异常也会中断线程。

线程中断的机制的基础是 ***InterruptedException*** ，在线程处于阻塞、等待、计时等待等状态时调用 ***interrupt*** 方法会抛出此异常导致线程中断，但不能中断阻塞的I/O调用和synchronized锁阻塞。

```java
class MyRunnable implements Runnabel {
    public void run() {
        try {
        	// ...work to do
        	Thread.sleep(2000); // 线程进入阻塞状态
        }
        catch (InterruptedException e) {
            // 异常是不能跨线程传递的，因此需要在当前线程解决
            e.printStack();
        }
        finally {
            // clean work to do
        }
    }
}

public class InterruptedTest {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start();
        // 线程开始运行后start方法会立即返回，执行interrupt方法，此时线程还在阻塞状态因此会抛出InterruptedException异常。
        t.interrupt();
    }
}
```



若线程没有处于上述三种状态之一，此时调用Interrupt方法不会抛出InterruptedException因此不会直接中断线程。但是，每个线程都有一个 ***boolean*** 类型的中断位，Interrupt方法会将其置位。***isInterrupted方法*** 可以获取此标志位。因此虽然不能直接中断线程，但是能使用此状态提示线程结束，如：

```java
class MyRunnable implements Runnabel {
    public void run() {
        try {
            while(!Thread.currentThread().isInterrupted()){
                // work to do
            }
        }
        finally {
            // clean work to do
        }
    }
}

public class InterruptedTest {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start();
        t.interrupt();
    }
}
```

在实现功能上，interrupted( )方法与isInterrupted( )方法 有相似之处。interrupted方法返回true当中断位被置位，然后将中断置为false。



```java
// java.lang.Thread 1.0

void interrupt();
// 向线程发送中断请求。当线程正处于阻塞、等待、计时等待状态时调用会抛出InterruptedException异常。

static boolean interrupted();
// 测试当前线程是否被中断。调用后将当前线程的状态置为false。

boolean isInterrupted();
// 测试当前线程是否被中断。

static Thread currentThread();
// 返回代表当前执行线程的Thread对象。
// 此方法在实现Runnable接口的类中使用Thread对象方法十分有用。
```

