## Java线程同步基础

线程同步意味着多个线程同时运行，可能会访问同一套资源，这时候会出现竞争条件。

当多个线程同时对同一个资源进行没有限制的访问时，就可能出现数据错误的情况。如，Bank类有若干个account，transfer方法用于在各account之间随机转账：

```java
class bank {
    // ...
    private double[] accounts = new double[100];
    
    public bank() {
        // 构造器将初始化accounts数组
    }
    public void transfer(int from, int to, double amount) {
        accounts[from] -= amount;
        accounts[to] += amount;
        System.out.println(getAllBalance());
    }
    public static double getAllBalance() {
        double sum;
        for(double i : accounts) {
            sum += i;
        }
        return sum;
    }
}
```

当单线程运行transfer方法时，任何时候调用getAllBalance方法输出的银行总余额是不会改变的。当使用多线程运行时，结果可能不同：

```java
calss TransferRunnable implements Runnable {
    // ...
    public void run() {
        try {
            int to = (int)(bank.size() * Math.random());
            int from = (int)(bank.size() * Math.random());
            double amount = maxAmount * Math.random();
            bank.transfer(from, to, amount);
            Thread.sleep(...);
        }
        catch (InterruptedException e) {}
    }
}
```

当使用多线程运行bank的transfer方法时，输出的银行总余额就可能发生变化。原因是转账操作并不是原子操作，倘若两个线程同时执行： ```accounts[to] += amount; ``` ，假设该指令被处理如下：

1. 加载accounts[to] 到寄存器。
2. 增加amount。
3. 结果写回accounts[to]。

现第一个线程执行了步骤1、2后被剥夺了运行权，此时第2个线程修改了数组中该项。然后第1个线程被唤醒完成了第3步。可见，第2个线程的更新被1个线程的第3步覆盖了，此时总余额就不正确了。

---

### 锁

锁机制可防止并发访问出现的竞争问题。

#### ReentrantLock

ReentrantLock 是一个实现Lock接口的类，其保证了只有获得该锁对象的线程才能进入临界区。一般结构如下:

```java
mylock.lock(); // mylock是一个ReentrantLock对象
try {
    // critical section 
}
finally {
    mylock.unlock();
    // 确保锁任何情况都释放
}
```

上述结构下，一旦一个线程获得了锁对象，其他任何线程都无法通过lock语句直到第一个线程释放锁对象。

注意，一定要确保锁对象在任何情况下都能够被释放，因此将unlock语句放在finally语句块中是比较合适的。示例如下：

```java
class lockTest {
    private Lock myLock = new ReentrantLock();

    public void lockMethod() {
        myLock.lock();
        try {
            System.out.println("Get an lock");
            Thread.currentThread().sleep(1000);
        }
        catch (InterruptedException e) {}
        finally {
            myLock.unlock();
        }
    }

    public void noLock() {
        System.out.println("No lock");
    }
}
```

锁是***可重入的***，即线程可以重复地获得已经持有的锁。例如，将上述代码中的noLock方法也是用锁机制限制，当一个线程通过lockMethd()方法获得该对象锁后，其还可以调用noLock方法。锁维持了一个持有计数器，每获得一次锁计数+1，释放一次-1。



``` java
// java.util.concurrent.locks.Lock 5.0
void lock();
// 获取该锁，若锁已被另一个线程拥有则阻塞当前线程。

void unlock();

// java.util.concurrent.locks.ReentrantLock 5.0
ReentrantLock(boolean fair = false);
// 构建一个可用来保护临界区的可重入锁。 参数为true时，带有公平策略，偏向等待时间最长的线程。
```

---

### 条件对象

线程进入临界区后，有时发现要在某一条件满足后它才能继续执行，但这个条件暂时无法满足。这种情况下就会出现死锁。使用***Condition***类条件对象机制可消除死锁。**条件对象是与对象锁相联系的，往往是锁对象调用newCondition方法生成的**。

```java
class Bank {
    private Lock bankLock = new ReentrantLock();
    private Condition sufficientBalance;
    
    public Bank() {
        // ...
        sufficientBalance = bankLock.newCondition();
    }
}
```

在银行示例中，当一个线程获得锁对象后发现自己的余额不足以进行转账即不满足条件，则对应条件对象会调用***await***方法，当前线程**放弃锁**并进入该条件的等待集阻塞，直到另一个线程调用同一条件上的**signalAll**方法时。另一个线程调用signalAll方法的时机是，其刚完成转账意味着前一个阻塞的线程的余额有**可能**满足转账条件了。

调用了await方法的线程只能被其他线程调用signalAll方法唤醒。被唤醒后，该线程试图重新进入临界区，一旦锁此时是可用的，线程将从await调用返回获得该锁并从被阻塞的地方继续执行。 注意，signalAll方法仅通知正在等待的线程此时可能满足条件但不能确保，因此最好再度检验条件。await调用推荐形式如下：

```java
while(!(ok to proceed)) {
    condition.await();
}
```



```java
// java.util.concurrent.locks.Lock 5.0
Condition newCondition();
// 返回一个与当前锁相关的条件对象

//java.util.concurrent.locks.Condition 5.0
void await();
// 将该线程放到条件的等待集中，并放弃锁进入阻塞。

void signalAll();
// 接触该条件的等待集的所有阻塞状态。

void signal();
// 从该条件的等待集中随机选择一个线程接触其阻塞状态
```

---

### synchronized关键字

事实上，Java Object对象自带一个内部锁即每个Java对象都有一个内部锁。因此锁机制可以简化为，使用***synchronized***关键字声明的方法就受对象锁保护。

```java
public synchronized void method(){}

// 等价于

public void method() {
    lock.lock();
    // ...
    lock.unlock();
}
```

内部对象锁有且只有**一个相关条件**。Java对象的***wait/notifyAll/notify***方法作用等价于条件对象的await/signalAll/signal方法。

现使用synchronized关键字重写银行类的transfer算法：

```java
class Bank{
    public sychronized void transfer(int from, int to, int amount) throws InterruptedException {
        while(accounts[frmo]<amount) {
            await();
        }
        // 完成转账
        notifyAll();
    }
}
```

另外，也可以使用synchronized声明**静态方法**；调用这种方法会获得相关的类的内部锁，如Bank.class的对象锁。



目前来看，内部锁和条件存在一些局限。包括：

* 不能中断一个正在试图获得锁的线程。
* 试图获得锁时不能设定超时时限。
* 每个锁仅有单一的条件，可能不够。



```java
// java.lang.Object 1.0
void notifyAll();
// 解除所有在该对象上调用wait方法的线程的阻塞状态。若当前线程不是对象锁的持有者，抛出IllegalMonitorStateException.

void notify();
// 随机选择一个。

void wait();
// 使线程进入等待状态直到它被通知。异常情况如notifyAll
void wait(long millis, int nanos);
```





