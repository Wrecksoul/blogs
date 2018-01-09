---
title: java并发个人总结——locks API
categories:
- java基础
tags:
- 并发
- J.U.C
---

## 前言

本文对java 6/7 java.util.concurrent.locks的主要接口和抽象类的API进行不完全的翻译，写了一点个人理解，并且对API分了段落。

个人感觉在看那些网上的各种详解教程之前应该先读一下源码中的API注释，这有利于在理解各种同步类源码的时候能纵观全局统筹兼顾，起到窥一斑而知全貌的效果。毕竟Doug Lea才是最懂JUC包的人而他本人写的注释更是最正确的解释，所以看看API肯定没坏处。

本文翻译了三个接口（Lock、Condition、ReadWriteLock）一个抽象类（AQS）。

<!-- more -->


<div class="note danger"><p>敬告：本文在某种意义上有点像中学的阅读理解解析，主要是我记录一些我阅读中理解历程，有些冗长受不了话请火速离开现场。</p></div>

## j.u.c.locks和早期锁对比

java5之前的多线程编程在锁方面就只有 `synchronized` 关键字，在线程互相协调方面有 `Object` 类中的几个方法（`wait, notify and notifyAll`等）使用起来有很多局限性，所以在java5中加入了JUC包，而JUC包中的locks子包中提供了锁的接口及实现。

总的来说：
1. Lock接口是锁的统一接口，它的实现类所要起的作用和synchronized关键字一样，但是提供了更加灵活的用法和**某些条件下**更好的性能；
2. Condition接口是线程之间互相协调这样一个动作的统一接口，它的实现类所要起的作用和`wait, notify and notifyAll`方法起到的作用相同，但是提供了更加灵活的用法。
3. AQS（AbstractQueuedSynchronizer）是一个工具性的抽象类，它的存在不像上面两个接口一样为了定义标准，它提供工具性的方法，一方面方便子类的实现并减少代码冗余，一方面利用这些工具方法它定义了锁等待、锁获取的运行机制（模板方法模式），它以一种特定的方式（AQS的API中会提到）提供服务，剧透一点，它的子类不是锁实现类，而是锁类的一个成员，锁实现类利用此引用完成各种锁操作。

## Lock接口和synchronized

### Lock的作用

Lock接口的实现类主要用于代替synchronized关键字并且提供更多功能，在Lock接口的API中主要说明了synchronized关键字的局限性。

> Lock implementations provide more extensive locking operations than can be obtained using synchronized methods and statements. They allow more flexible structuring, may have quite different properties, and may support multiple associated Condition objects.

Lock接口的实现类提供了相对于synchronized关键字修饰的方法或者代码块更加广泛的锁操作，他们允许更加灵活的结构，允许拥有十分不同的属性，可以支持多个与锁相关联的 Condition 接口的对象。

> A lock is a tool for controlling access to a shared resource by multiple threads. Commonly, a lock provides exclusive access to a shared resource: only one thread at a time can acquire the lock and all access to the shared resource requires that the lock be acquired first. However, some locks may allow concurrent access to a shared resource, such as the read lock of a ReadWriteLock.

一个锁时控制多线程访问共享资源时访问权限的控制工具。一般，一个锁提供一个唯一的访问共享资源的权限：同一时间只有一个线程可以获取锁，并且所有共享资源的访问都需要首先获取到锁才行。当然有些锁也允许共享资源的并发访问，比如 `ReadWriteLock` 的读锁（read lock）。

### 与synchronized对比

> The use of synchronized methods or statements provides access to the implicit monitor lock associated with every object, but forces all lock acquisition and release to occur in a block-structured way: when multiple locks are acquired they must be released in the opposite order, and all locks must be released in the same lexical scope in which they were acquired.
> 
> While the scoping mechanism for synchronized methods and statements makes it much easier to program with monitor locks, and helps avoid many common programming errors involving locks, there are occasions where you need to work with locks in a more flexible way. For example, some algorithms for traversing concurrently accessed data structures require the use of "hand-over-hand" or "chain locking": you acquire the lock of node A, then node B, then release A and acquire C, then release B and acquire D and so on. Implementations of the Lock interface enable the use of such techniques by allowing a lock to be acquired and released in different scopes, and allowing multiple locks to be acquired and released in any order.

`synchronized`修饰的方法和代码块利用的是隐式监视器锁（对象锁，每个对象都关联一个），但是却强制所有锁的获取和释放都需要发生在同一个块结果中：当多个锁被获取时它们被释放的顺序必须跟获取相反，并且且必须在与锁被获取时相同的词法范围内释放所有锁。（注：其实就是说，synchronized是全自动化设计，进入代码块就获取锁，出代码块就释放锁，由于用代码块来控制资源界限，多个锁获取和释放自然是多个代码块嵌套，释放顺序自然就固定了。）

虽然范围机制使得`synchronized`修饰的方法和代码块提供的锁机制使用起来更简单而且还帮助避免了很多涉及到锁的常见编程错误，但是有一些场景下你需要以更加灵活的方式使用锁。例如，某些遍历并发访问数据结构的算法要求使用称为 "hand-over-hand" 或 "chain locking"的获取方式：获取节点 A 的锁，然后再获取节点 B 的锁，然后释放 A 并获取 C，然后释放 B 并获取 D，依此类推。Lock 接口的实现允许锁在不同的作用范围内获取和释放，并允许以任何顺序获取和释放多个锁，从而支持使用"hand-over-hand"这种技术实现。

### 灵活的代价

> With this increased flexibility comes additional responsibility. The absence of block-structured locking removes the automatic release of locks that occurs with synchronized methods and statements. In most cases, the following idiom should be used:

```
Lock l = ...;
l.lock();
try {
 // access the resource protected by this lock
} finally {
 l.unlock();
}
```
> When locking and unlocking occur in different scopes, care must be taken to ensure that all code that is executed while the lock is held is protected by try-finally or try-catch to ensure that the lock is released when necessary.


随着灵活性的增加也带来了更多的责任。`synchronized`修饰的方法和代码块提供的自动释放锁的功能没有了。更多的情况下需要使用如下语句。
(代码略)
当获取锁和解锁发生在不同的作用域时，必须谨慎地确保保持锁定时所执行的所有代码用 try-finally 或 try-catch 加以保护，以确保在必要时释放锁。 

### Lock实现可能提供的额外功能

> Lock implementations provide additional functionality over the use of synchronized methods and statements by providing a non-blocking attempt to acquire a lock (tryLock()), an attempt to acquire the lock that can be interrupted (lockInterruptibly(), and an attempt to acquire the lock that can timeout (tryLock(long, TimeUnit)).
> 
> A Lock class can also provide behavior and semantics that is quite different from that of the implicit monitor lock, such as guaranteed ordering, non-reentrant usage, or deadlock detection. If an implementation provides such specialized semantics then the implementation must document those semantics.

Lock 实现类提供了使用 synchronized 方法和语句所没有的其他功能，包括：提供了一个**非阻塞**（其实是只尝试一次）的获取锁方法`tryLock()`、一个**可中断**的获取锁方法 `lockInterruptibly()` 和一个**超时失效**的获取锁的方法`tryLock(long, TimeUnit)`。

一个Lock实现类还可以提供与隐式监视器锁完全不同的行为和语义，比如保证排序、不可重入性或死锁检测，如果一个实现类提供了这样的特殊行为和语义的话必须在文档中记录。

### 切勿这样使用

> Note that Lock instances are just normal objects and can themselves be used as the target in a synchronized statement. Acquiring the monitor lock of a Lock instance has no specified relationship with invoking any of the lock() methods of that instance. It is recommended that to avoid confusion you never use Lock instances in this way, except within their own implementation.
> 
> Except where noted, passing a null value for any parameter will result in a NullPointerException being thrown.

<div class="note warning"><p>注意Lock 实例只是普通的对象，其本身可以在 synchronized 语句中作为目标使用。获取 Lock 实例的监视器锁与调用该实例的任何 lock() 方法没有特别的关系。为了避免混淆，建议除了在其自身的实现中之外，决不要以这种方式使用 Lock 实例。</p></div>

除非另有说明，否则为任何参数传递 null 值都将导致抛出 NullPointerException。

*(这段话和Condition API最后的话非常相似)*

### 内存同步

> *Memory Synchronization*
> All Lock implementations must enforce the same memory synchronization semantics as provided by the built-in monitor lock, as described in section 17.4 of The Java™ Language Specification:
> 
> 1. A successful lock operation has the same memory synchronization effects as a successful Lock action.
> 2. A successful unlock operation has the same memory synchronization effects as a successful Unlock action.
> 
> Unsuccessful locking and unlocking operations, and reentrant locking/unlocking operations, do not require any memory synchronization effects.

*内存同步*
所有 Lock 实现都**必须**实施与**内置监视器锁**提供的相同内存同步语义，如 The Java Language Specification, Third Edition (17.4 Memory Model) 中所描述的: 

1. 成功的 lock 操作与成功的 *Lock* 操作具有同样的内存同步效应。
2. 成功的 unlock 操作与成功的 *Unlock* 操作具有同样的内存同步效应。

不成功的锁定与取消锁定操作以及重入锁定/取消锁定操作都不需要任何内存同步效果。

（斜体的 *Lock* 和 *Unlock* 指的是内置监视器锁的锁动作）

### 实现Lock接口时的注意事项

> *Implementation Considerations*
> The three forms of lock acquisition (interruptible, non-interruptible, and timed) may differ in their performance characteristics, ordering guarantees, or other implementation qualities. Further, the ability to interrupt the ongoing acquisition of a lock may not be available in a given Lock class. Consequently, an implementation is not required to define exactly the same guarantees or semantics for all three forms of lock acquisition, nor is it required to support interruption of an ongoing lock acquisition. An implementation is required to clearly document the semantics and guarantees provided by each of the locking methods. It must also obey the interruption semantics as defined in this interface, to the extent that interruption of lock acquisition is supported: which is either totally, or only on method entry.
> 
> As interruption generally implies cancellation, and checks for interruption are often infrequent, an implementation can favor responding to an interrupt over normal method return. This is true even if it can be shown that the interrupt occurred after another action may have unblocked the thread. An implementation should document this behavior.

*实现注意事项*
三种形式的锁获取（可中断、不可中断和定时）在其性能特征、排序保证或其他实现质量上可能会有所不同。而且，对于给定的 `Lock` 类，可能没有中断正在进行的锁获取的能力。因此，并不要求实现为所有三种形式的锁获取定义相同的保证或语义，也不要求其支持中断正在进行的锁获取。实现类必须清楚地对每个锁定方法所提供的语义和保证提供清晰的文档说明，还必须遵守此接口中定义的中断语义到这样的程度：完全支持中断，或仅在进入方法时支持中断。

由于中断通常意味着取消，而通常又很少进行中断检查，因此，相对于普通方法返回而言，实现可能更喜欢响应某个中断。即使出现在另一个操作**之后**的中断可能会释放线程锁时也是这样。实现对此行为提供详细文档说明。 (这里是说由于中断检查不是很常见，所以一旦出现中断它的优先级会比较高，“即使出现在另一个操作之后的中断可能是否线程锁时也是这样”，如果这样的话应该是非常危险的吧，因为锁被释放可能会造成线程安全问题。)

**其实如果看过具体实现你会发现，所谓的中断，并不是在中间断开，而是在最后执行结束后检查是否有中断。**，这个其实我没有验证过啊。。。。。

## Condition接口

AQS有两个内部类，一个是Node一个是ConditionObject。前者是static final修饰的，不可继承，对于子类来说不需要关心，后者ConditionObject是Condition接口的实现类。

java6/7中，此Condition接口只有两个实现类，分别是 `AbstractQueuedLongSynchronizer` 和 `AbstractQueuedSynchronizer` 的内部类都叫做 `ConditionObject`，其实原理一模一样只是一个使用int一个使用long做核心整型而已。

### Condition的作用

> Condition factors out the Object monitor methods (wait, notify and notifyAll) into distinct objects to give the effect of having multiple wait-sets per object, by combining them with the use of arbitrary Lock implementations. Where a Lock replaces the use of synchronized methods and statements, a Condition replaces the use of the Object monitor methods. 
> 
> Conditions (also known as condition queues or condition variables) provide a means for one thread to suspend execution (to "wait") until notified by another thread that some state condition may now be true. Because access to this shared state information occurs in different threads, it must be protected, so a lock of some form is associated with the condition. The key property that waiting for a condition provides is that it atomically releases the associated lock and suspends the current thread, just like Object.wait. 

这句话从全局的角度看Condition接口以及它的实现类，可以简单的理解为 `Condition` 接口的作用就是替代 `wait, notify and notifyAll` 这些方法，就好似 `ReetrantLock` 替代 `synchronized` 关键字一样。上面英文虽然很长但是意思就是这些，如果不了解`wait, notify and notifyAll` 这些方法的含义当然就没办法理解Condition接口了。

### Condition的使用

> A Condition instance is intrinsically bound to a lock. To obtain a Condition instance for a particular Lock instance use its newCondition() method. 

这句话非常重点，它指明了此接口的子类不是独立存在的，这有两个方面的限制，一必须是一个内部类，二不能是静态内部类，再看 `AbstractQueuedSynchronizer.ConditionObject`的声明： `public class ConditionObject implements Condition, java.io.Serializable` 。当需要获取一个锁实例的条件时调用 `newCondition` 方法，注意此方法是接口 `Lock` 定义的。由于是内部类，`ConditionObject` 的实例一定会绑定在一个 `Lock` 接口的实例上，即lock可以获取到跟它关联的condition，其实我感觉使用内部类并不是必须的，不过由于内部类天然的可以访问外部类所有成员，所以少了很多传递参数的必要更加方便了。

> As an example, suppose we have a bounded buffer which supports put and take methods. If a take is attempted on an empty buffer, then the thread will block until an item becomes available; if a put is attempted on a full buffer, then the thread will block until a space becomes available. We would like to keep waiting put threads and take threads in separate wait-sets so that we can use the optimization of only notifying a single thread at a time when items or spaces become available in the buffer. This can be achieved using two Condition instances.

下面举个例子，假定有一个绑定的缓冲区，它支持 put 和 take 方法。如果试图在空的缓冲区上执行 take 操作，则在某一个项变得可用之前，线程将一直阻塞；如果试图在满的缓冲区上执行 put 操作，则在有空间变得可用之前，线程将一直阻塞。我们喜欢在单独的等待 set 中保存 put 线程和 take 线程，这样就可以在缓冲区中的项或空间变得可用时利用最佳规划，一次只通知一个线程。可以使用两个 Condition 实例来做到这一点。

```
 class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   }
 }
```

> (The ArrayBlockingQueue class provides this functionality, so there is no reason to implement this sample usage class.)
> A Condition implementation can provide behavior and semantics that is different from that of the Object monitor methods, such as guaranteed ordering for notifications, or not requiring a lock to be held when performing notifications. If an implementation provides such specialized semantics then the implementation must document those semantics.

（ArrayBlockingQueue 类提供了这项功能，因此没有理由去实现这个示例类。） 
Condition 实现可以提供不同于 Object 监视器方法的行为和语义，比如**受保证的通知排序**，或者在**执行通知时不需要拥有一个锁**。如果某个实现提供了这样特殊的语义，则该实现必须在文档中详细说明这些语义。（即这种额外的功能性不是每个condition都一定实现需要实现的，具体要看文档。）

### 切勿这种方式使用

> Note that Condition instances are just normal objects and can themselves be used as the target in a synchronized statement, and can have their own monitor wait and notification methods invoked. Acquiring the monitor lock of a Condition instance, or using its monitor methods, has no specified relationship with acquiring the Lock associated with that Condition or the use of its waiting and signalling methods. It is recommended that to avoid confusion you never use Condition instances in this way, except perhaps within their own implementation.
> 
> Except where noted, passing a null value for any parameter will result in a NullPointerException being thrown.

<div class="note warning"><p>注意，Condition 实例只是一些普通的对象，它们自身可以用作 synchronized 语句中的目标，并且可以调用自己的 wait 和 notification 监视器方法。获取 Condition 实例的监视器锁或者使用其监视器方法，与获取和该 Condition 相关的 Lock 或使用其 waiting 和 signalling 方法没有什么特定的关系。为了避免混淆，建议除了在其自身的实现中之外，切勿以这种方式使用 Condition 实例。</p></div>

除非另行说明，否则为任何参数传递 null 值将导致抛出 NullPointerException。 

## ReadWriteLock接口

### 总览
> A ReadWriteLock maintains a pair of associated locks, one for read-only operations and one for writing. The read lock may be held simultaneously by multiple reader threads, so long as there are no writers. The write lock is exclusive.

一个读写锁维护这一对相关的锁，一个用于只读操作一个用于写操作。读锁可以被多个读线程同时保持，但写锁是独占的。（写锁不止独占写锁，独占时读锁也不能被获取）

> All ReadWriteLock implementations must guarantee that the memory synchronization effects of writeLock operations (as specified in the Lock interface) also hold with respect to the associated readLock. That is, a thread successfully acquiring the read lock will see all updates made upon previous release of the write lock.

所有ReadWriteLock的实现类必须保证写锁操作的影响到获取到读锁的线程。换句话说，一个成功获取到读锁的线程可以看到所有写锁的更新内容。

> A read-write lock allows for a greater level of concurrency in accessing shared data than that permitted by a mutual exclusion lock. It exploits the fact that while only a single thread at a time (a writer thread) can modify the shared data, in many cases any number of threads can concurrently read the data (hence reader threads). In theory, the increase in concurrency permitted by the use of a read-write lock will lead to performance improvements over the use of a mutual exclusion lock. In practice this increase in concurrency will only be fully realized on a multi-processor, and then only if the access patterns for the shared data are suitable.

相比互斥锁，读写锁允许更高级别的并发访问。要满足线程安全，虽然同一时间只能由一个进程修改数据，但是任何数量的进程可以同时读取，读写锁正是利用了这一点。理论上讲，与互斥锁相比，使用读-写锁所允许的并发性增强将带来更大的性能提高。在实际中，只有在多处理器上并且只在访问模式适用于共享数据时，才能完全实现并发性增强。 
<div class="note info"><p>
1. 为什么必须多处理器呢，因为单处理器实现的多线程读共享数据不是真正的**并行访问**;
2. 所谓的合适的访问模式，应该是指读锁不会占用时间太长导致写锁被饿死的情况。
</p></div>

> Whether or not a read-write lock will improve performance over the use of a mutual exclusion lock depends on the frequency that the data is read compared to being modified, the duration of the read and write operations, and the contention for the data - that is, the number of threads that will try to read or write the data at the same time. For example, a collection that is initially populated with data and thereafter infrequently modified, while being frequently searched (such as a directory of some kind) is an ideal candidate for the use of a read-write lock. However, if updates become frequent then the data spends most of its time being exclusively locked and there is little, if any increase in concurrency. Further, if the read operations are too short the overhead of the read-write lock implementation (which is inherently more complex than a mutual exclusion lock) can dominate the execution cost, particularly as many read-write lock implementations still serialize all threads through a small section of code. Ultimately, only profiling and measurement will establish whether the use of a read-write lock is suitable for your application.

使用读写锁时的性能能不能比使用互斥锁高，取决于读写频度的对比、读写操作的耗时以及数据的争用程度（即同一时间内想要读或写数据的线程数有多少）。例如，一个数据初始化并且之后不经常对其进行修改的集合，但是经常对其进行搜索（比如搜索某种目录），那么这样的集合是使用读-写锁的理想对象。但是如果数据更新很频繁，共享数据大部分时间被写锁独占，就算这时候有并发性能上的增强，也是微不足道的。进一步说，如果**读操作所用的时间太短**，则读写锁本身的实现（读写锁本身的实现比互斥锁复杂）开销将成为一个显著的性能影响因素，特别是许多读写锁的实现仍然通过一点段代码将所有线程序列化（**这里不知所云**）。最终，只有通过分析和测量，才能确定应用程序是否适合使用读-写锁。 

这里对读写锁的使用场景进行了简单说明

> Although the basic operation of a read-write lock is straight-forward, there are many policy decisions that an implementation must make, which may affect the effectiveness of the read-write lock in a given application. Examples of these policies include:
> 1. Determining whether to grant the read lock or the write lock, when both readers and writers are waiting, at the time that a writer releases the write lock. Writer preference is common, as writes are expected to be short and infrequent. Reader preference is less common as it can lead to lengthy delays for a write if the readers are frequent and long-lived as expected. Fair, or "in-order" implementations are also possible.
2. Determining whether readers that request the read lock while a reader is active and a writer is waiting, are granted the read lock. Preference to the reader can delay the writer indefinitely, while preference to the writer can reduce the potential for concurrency.
3. Determining whether the locks are reentrant: can a thread with the write lock reacquire it? Can it acquire a read lock while holding the write lock? Is the read lock itself reentrant?
4. Can the write lock be downgraded to a read lock without allowing an intervening writer? Can a read lock be upgraded to a write lock, in preference to other waiting readers or writers?

> You should consider all of these things when evaluating the suitability of a given implementation for your application.

在实现读写锁时，尽管基本操作可以实现的非常直接，但是仍然有很多抉择需要做，它们可能影响到读写锁在应用中的使用效果。这些需要考虑的抉择：
1. 抉择：一个写锁被释放时，如果读线程和写线程都在等待的，要授予读锁还是写锁呢。一般情况下回优先授予写锁，因为写操作往往耗时短并且不频繁；优先读锁不上很常见，因为如果读操作很频繁而且耗时的话会导致写操作迟迟得不到执行。当然，公平 或者 按次序 的实现也是可以的。
2. 抉择：当读操作进行时而有一个写操作在等待时，一个新的读锁请求是否可以获取读锁呢。可以获取，会导致写操作执行后延，不可以获取会导致并发下降。
3. 抉择：是否可重入。读锁可以重入吗？写锁可以重入吗？一个拥有写锁的线程可以获取读锁吗？
4. 抉择：一个写锁可以在不允许其他写线程进入的前提下降级为读锁吗？一个读锁的持有线程可以优先于其他正在等待读锁或者写锁的线程升级成写锁吗？

为你的应用找到一个合适的读写锁实现，应该考虑上述所有情况。

## AQS类

### 总览

> Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues. This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state.Subclasses must define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released. Given these, the other methods in this class carry out all queuing and blocking mechanics. Subclasses can maintain other state fields, but only the atomically updated int value manipulated using methods getState(), setState(int) and compareAndSetState(int, int) is tracked with respect to synchronization. 

AQS类是一个框架，它只提供了一些同步的阻塞、排队机制，并且排队时是使用先进先出的队列方式。最重要的一句话 “This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state.” 这个类是被设计来作为一个基类的，言外之意这个类只是基类，功能并不完全，这个基类可以帮助构建很多同步器，而这个基类的特点就是**使用一个原子的int型整数来作为标志位**。

从上面的最后一句可以简单理解为，所有以AQS作为基类的同步器的最终核心都是一个int类型的整数，获取锁、释放锁、排队等待等行为其实都是根据当前此整数的数值进行判断的。而这个整型值各种数值代表什么意义却由子类来定义。

既然这个整型数字如此重要，所以最后提到一句“Subclasses can maintain other state fields, but only the atomically updated int value manipulated using methods getState(), setState(int) and compareAndSetState(int, int) is tracked with respect to synchronization.” 子类在具体实现的时候可以自行维护状态，但是由于AQS里面提供了排队、阻塞的逻辑代码，而这些代码的依据就是那个核心整型数字，而这个核心整型数字的get和set只能使用`getState(), setState(int) and compareAndSetState(int, int)`这三个方法来操作。所以，只有这三个方法设定和获取的状态值才是同步的。

<div class="note warning"><p>简单点说就是如果你自行定义一个同步类，而且你使用AQS作为基类，你可以自行维护一个自定义的状态值，但是这个状态值由于AQS不知道，所有请你自己去实现它的同步吧。再简单点说就是别整那些幺蛾子，要是整的话自己写代码去吧AQS不是万能的。</p></div>

### 子类如何利用AQS

> Subclasses should be defined as non-public internal helper classes that are used to implement the synchronization properties of their enclosing class. Class AbstractQueuedSynchronizer does not implement any synchronization interface. Instead it defines methods such as acquireInterruptibly(int) that can be invoked as appropriate by concrete locks and related synchronizers to implement their public methods. 

这是说子类如何去写，其实大部分时候我们不会自己去写。**子类以非公共的内部类形式定义**。还有就是AQS没有实现任何同步相关的接口，比如Lock接口，而是定义了一些方法，这些方法可以在你定义子类的时候去调用他们。

<div class="note info"><p>看出来了吧，AQS其实就是个工具包，它不但提供了 **机制** 还提供了 **方法** 供子类使用，一方面减少代码重复性，一方面方便扩展新的同步器。而且AQS的核心是一个被原子操作的整型值。而从设计的角度看，Doug Lea选择设计一个基类而不是一堆工具类来实现，换句话说AQS没有兄弟姐妹我们不需要在一大堆互相关联的类里面跳来跳去。</p></div>

从上面我们就看到了AQS的作用和整体JUC包中的总体思路。

### AQS特性

> This class supports either or both a default exclusive mode and a shared mode. When acquired in exclusive mode, attempted acquires by other threads cannot succeed. Shared mode acquires by multiple threads may (but need not) succeed. This class does not "understand" these differences except in the mechanical sense that when a shared mode acquire succeeds, the next waiting thread (if one exists) must also determine whether it can acquire as well. Threads waiting in the different modes share the same FIFO queue. Usually, implementation subclasses support only one of these modes, but both can come into play for example in a ReadWriteLock. Subclasses that support only exclusive or only shared modes need not define the methods supporting the unused mode. 

说明了两种锁模式——独占和共享。独占锁只能一个线程占用，而共享锁可能被多个线程同时获得。

*AQS类不懂这两种锁模式的区别，除了机械的意识到当共享锁被一个线程占用时，另一个想获取它的线程也需要决定是不是也可以获取。* 斜体字部分是直接翻译，用人话说就是：“AQS锁在处理两种锁模式的时候几乎没有什么区别，唯一的区别在于获取锁的逻辑，对于独占锁来说，如果一个线程已经占有了锁，那么其他线程肯定需要等待，但是对于共享锁来说，就需要多一个逻辑判断是否可以获取了。”

这句非常重要：“Threads waiting in the different modes share the same FIFO queue.” 就是说AQS只维持了一个队列。关于这一点需要在 `ReadWriteLock` 里面仔细看看才好，没准是要用两个锁呢。

最后一句话说明了在使用AQS时，子类只需要重写自己需要的那些方法，不需要全部重写。因为AQS类没有抽象方法，但是有几个方法的实现只是简单的抛出 `UnsupportedOperationException` 异常，如果用到的话就重写，用不到就拉倒呗。

### 内部类

> This class defines a nested AbstractQueuedSynchronizer.ConditionObject class that can be used as a Condition implementation by subclasses supporting exclusive mode for which method isHeldExclusively() reports whether synchronization is exclusively held with respect to the current thread, method release(int) invoked with the current getState() value fully releases this object, and acquire(int), given this saved state value, eventually restores this object to its previous acquired state. No AbstractQueuedSynchronizer method otherwise creates such a condition, so if this constraint cannot be met, do not use it. The behavior of AbstractQueuedSynchronizer.ConditionObject depends of course on the semantics of its synchronizer implementation. 

这一段详细解释一下，但是需要结合 `ConditionObject` 部分的注释才能说清楚。第一句话非常的长 `for which` 后面跟了一个特别长的从句，翻译成中文的话大概要翻译成3句话，第一句，AQS定义了一个嵌套的内部类 `ConditionObject` ，它可以作为 `Condition` 的一个现成的实现被子类使用；第二句， `ConditionObject` 可以帮助实现**独占模式**；第三句，方法 `isHeldExclusively()` 可以报告当前线程是否持有独占锁，方法 `release(getState())` 可以完全释放此对象（指ConditionObject），而如果你把刚刚 `getState()` 的值保持下来然后传入 `acquire(int)` 里面去，就可以恢复占有的状态。第三句话是对独占模式的一个解释，忽略掉它可以更清楚的了解一整句的真正意思。

> This class provides inspection, instrumentation, and monitoring methods for the internal queue, as well as similar methods for condition objects. These can be exported as desired into classes using an AbstractQueuedSynchronizer for their synchronization mechanics. 

这段话几乎没什么用，`AbstractQueuedSynchronizer` 提供了对队列和 `condition objects` 的一些维护方法，你可以根据需要使用AQS来提供这些同步机制。

### 序列化

> Serialization of this class stores only the underlying atomic integer maintaining state, so deserialized objects have empty thread queues. Typical subclasses requiring serializability will define a readObject method that restores this to a known initial state upon deserialization.

序列化方面的问题（一般JAVA官方API的注释最后一段都会提到这个），序列化时只会存储维护状态的基础原子整数，而线程队列不会被存储（其实存储也没用啊，因为线程的信息是不同的OS无法相同的，就算同一个OS不同时间也无法相同，就算相同了也没有实际意义）。如果子类有序列化的需求，需要重写 `readObject(ObjectInputStream)` 方法，该方法在反序列化时将此对象恢复到某个已知初始状态。这里我有一点不懂，序列化一个锁有什么用呢，是为了存储当前系统的运行状态，重启时恢复？

## 总结

<div class="note default"><p>锁是控制多线程对共享资源的访问权限的一种机制。</p></div>

我们平时对锁的需求除了**临界区**执行线程互相排斥（互斥锁）、控制信号量等需求之外，还需要线程间进行协调工作（注1），也就是说，**线程间有竞争关系时需要锁（Lock），线程间需要协调时不仅需要获得锁还需要满足条件（Condition）**,条件（Condition）是锁（Lock）的附属品，一个锁可以有多个条件，具体的需求可能只需要锁或者只需要条件也可能都需要用到。各种锁实现从不同程度上支持Lock和Condition。那具体它们是怎么组织实现这两个接口的呢？

答案是AQS，AQS内部定义了一个内部类ConditionObject实现了Condition接口，还提供了一些与锁相关的方法，理论上讲，我们只要继承AQS实现一个子类，并且让这个类实现Lock接口，那么这个类就同时具有了Lock和Condition的特性。不过为了更好的灵活性API中并不建议我们这样做，而是利用代理模式。

继承AQS定义一个具有了Condition特性的类Sync，然后定义另外一个实现Lock接口的锁实现类，在锁实现类中持有一个Sync实例的引用，以代理模式的写法利用一个内部引用完成各种功能。

<div class="note warning"><p>为什么如此实现：锁往往有公平锁和非公平锁之分，往往一个锁实现类（比如ReetrantLock）其实内部存在两个锁实现，这两个实现分别是公平锁和非公平锁，同一时间只有一个有效，而控制这个的只是构造函数的一个boolean参数，这样使得API更加简洁易用。</p></div>

*注1：什么是线程间的协调工作呢，线程之间虽然大多数情况下可以是并行的，但是某些情况下应业务需要必须保证有序执行，比如分库分表的查询最后的汇总线程必须等待所有查询线程都结束才能执行，类似于这种的业务需求导致我们需要线程之间互相协调。这就不同于普通的互斥锁*