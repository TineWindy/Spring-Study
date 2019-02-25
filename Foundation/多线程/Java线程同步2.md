## Java线程同步2

### volatile关键字

volatile关键字为**实例域的同步访问**提供了免锁机制，对实例域使用此关键字修饰后就能对其进行同步访问。

volatile关键字的原理如下：

* 保证内存一致性。即对volatile关键字修饰的实例域，读操作都会在Java 堆内存中读取，写操作会立即写回Java堆内存，因此保证了内存数据一致性。
* 涉及到此变量的语句，编译器不会将指令代码进行重排序。

综上，volatile关键字在实例域的同步访问方面有轻便的优势。但volatile变量并不能提供原子性。

---

### 死锁、锁测试

死锁相关内容操作系统原理已了解。发生死锁后，Java是没有机制可以解决的，因此只能在设计的时候仔细。

由于线程在调用lock方法获取不到锁时会进入无法中断的阻塞，因此申请锁时应谨慎。***tryLock***方法试图申请锁，成功后返回true，否则立即返回false让线程可离开去做其他事情。一般格式如下：

```java
if(myLock.tryLock())
    try {...}
	finally {myLock.unlock();}
else // do extra thing
```

tryLock方法也可以带有超时参数，即在一定时间内阻塞以申请锁，申请不到就返回false。在这种情况下，阻塞是计时等待状态，可以中断。



```java
// java.util.concurrent.locks.Lock 5.0
boolean tryLock();
boolean tryLock(long time, TimeUnit unit);
// 获取锁，否则立即返回false。
// 计时参数为等待获取锁的时间，此期间线程进入计时等待状态，中断时抛出InterruptedException。
```

---

### 读写锁

常用锁类除前文中的ReentrantLock类，还有个读写锁ReentrantReadWriteLock类。

读写锁主要作用是让读写分开获取锁，允许读者共享访问，写者互斥访问。更确切是，读时只排斥写操作，写时排斥所有操作。示例如下：

```java
class MyLockClass {
    private ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private Lock readLock = rwl.readLock();
    private Lock writeLock = rwl.writeLock();
    
    public String readBooks() {
        readLock.lock();
        // ...
        readLock.unlock();
    }
    public void writeBook() {
        writeLock.lock();
        // ...
        writeLock.unlock();
    }
    /** 该类中，可多个线程同时调用readBooks方法，此时无法调用writeBook
      * 调用writeBook方法获取到写锁后，其他线程均无法获取读写锁 */ 
}
```



```java
// java.util.concurrent.locks.ReentrantReadWriteLock 5.0
Lock readLock();
// 得到一个可以被多个读操作共用的读锁，但排除所有写操作

Lock writeLock();
// 得到一个写锁，排除所有的读操作和写操作
```

