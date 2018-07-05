# Chapter 11. Concurrency

THREADS allow multiple activities to proceed concurrently. Concurrent programming is harder than single-threaded 
programming, because more things can go wrong, and failures can be hard to reproduce. You can’t avoid concurrency. 
It is inherent in the platform and a requirement if you are to obtain good performance from multicore processors, 
which are now ubiquitous. This chapter contains advice to help you write clear, correct, well-documented 
concurrent programs.

## ITEM 78: SYNCHRONIZE ACCESS TO SHARED MUTABLE DATA

The synchronized keyword ensures that only a single thread can execute a method or block at one time. Many 
programmers think of synchronization solely as a means of mutual exclusion, to prevent an object from being seen 
in an inconsistent state by one thread while it’s being modified by another. In this view, an object is created in 
a consistent state (Item 17) and locked by the methods that access it. These methods observe the state and 
optionally cause a state transition, transforming the object from one consistent state to another. Proper use of 
synchronization guarantees that no method will ever observe the object in an inconsistent state.

This view is correct, but it’s only half the story. Without synchronization, one thread’s changes might not be 
visible to other threads. Not only does synchronization prevent threads from observing an object in an inconsistent 
state, but it ensures that each thread entering a synchronized method or block sees the effects of all previous 
modifications that were guarded by the same lock.
{Aaron notes: Above is an important design.}

The language specification guarantees that reading or writing a variable is atomic unless the variable is of type 
long or double [JLS, 17.4, 17.7]. In other words, reading a variable other than a long or double is guaranteed to 
return a value that was stored into that variable by some thread, even if multiple threads modify the variable 
concurrently and without synchronization.
{Aaron notes: Above is an important design.}

You may hear it said that to improve performance, you should dispense with synchronization when reading or writing 
atomic data. This advice is dangerously wrong. While the language specification guarantees that a thread will not 
see an arbitrary value when reading a field, it does not guarantee that a value written by one thread will be 
visible to another. <b>Synchronization is required for reliable communication between threads as well as for mutual 
exclusion.</b> This is due to a part of the language specification known as the memory model, which specifies when 
and how changes made by one thread become visible to others [JLS, 17.4; Goetz06, 16].
{Aaron notes: Above is an important design.}

The consequences of failing to synchronize access to shared mutable data can be dire even if the data is atomically 
readable and writable. Consider the task of stopping one thread from another. The libraries provide the Thread.stop
method, but this method was deprecated long ago because it is inherently unsafe—its use can result in data corruption.
<b>Do not use Thread.stop. </b>A recommended way to stop one thread from another is to have the first thread poll 
a boolean field that is initially false but can be set to true by the second thread to indicate that the first thread 
is to stop itself. Because reading and writing a boolean field is atomic, some programmers dispense with 
synchronization when accessing the field:
{Aaron notes: Above is an important design.}
```aidl
// Broken! - How long would you expect this program to run?
public class StopThread {

    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```
You might expect this program to run for about a second, after which the main thread sets stopRequested to true, 
causing the background thread’s loop to terminate. On my machine, however, the program never terminates: the 
background thread loops forever!

The problem is that in the absence of synchronization, there is no guarantee as to when, if ever, the background 
thread will see the change in the value of stopRequested made by the main thread. In the absence of synchronization, 
it’s quite acceptable for the virtual machine to transform this code:
```aidl
    while (!stopRequested)
        i++;
```
into this code:
```aidl
if (!stopRequested)
    while (true)
        i++;
```
This optimization is known as hoisting, and it is precisely what the OpenJDK Server VM does. The result is a liveness
failure: the program fails to make progress. One way to fix the problem is to synchronize access to the stopRequested
field. This program terminates in about one second, as expected:
{Aaron notes: Above is an important design.}
```aidl
//Properly synchronized cooperative thread termination
public class StopThread {

   private static boolean stopRequested;

   private static synchronized void requestStop() {
       stopRequested = true;
   }

   private static synchronized boolean stopRequested() {
       return stopRequested;
   }

   public static void main(String[] args) throws InterruptedException {
       Thread backgroundThread = new Thread(() -> {
           int i = 0;
           while (!stopRequested())
               i++;
       });
       backgroundThread.start();
       TimeUnit.SECONDS.sleep(1);
       requestStop();
   }
}
```
Note that both the write method (requestStop) and the read method (stop-Requested) are synchronized. It is not
sufficient to synchronize only the write method! <b>Synchronization is not guaranteed to work unless both read 
and write operations are synchronized.</b> Occasionally a program that synchronizes only writes (or reads) may appear 
to work on some machines, but in this case, appearances are deceiving.
{Aaron notes: Above is an important design.}

The actions of the synchronized methods in StopThread would be atomic even without synchronization. In other words, 
the synchronization on these methods is used solely for its communication effects, not for mutual exclusion. While 
the cost of synchronizing on each iteration of the loop is small, there is a correct alternative that is less verbose 
and whose performance is likely to be better. The locking in the second version of StopThread can be omitted if 
stopRequested is declared volatile. While the volatile modifier performs no mutual exclusion, it guarantees that any 
thread that reads the field will see the most recently written value:
```aidl
// Cooperative thread termination with a volatile field
public class StopThread {

    private static volatile boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```
You do have to be careful when using volatile. Consider the following method, which is supposed to generate serial 
numbers:
```aidl
// Broken - requires synchronization!
private static volatile int nextSerialNumber = 0;

public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```
The intent of the method is to guarantee that every invocation returns a unique value 
(as long as there are no more than 2^32 invocations). The method’s state consists of a single atomically accessible field, 
nextSerialNumber, and all possible values of this field are legal. Therefore, no synchronization is necessary to protect 
its invariants. Still, the method won’t work properly without synchronization.

The problem is that the increment operator (++) is not atomic. It performs two operations on the nextSerialNumber field: 
first it reads the value, and then it writes back a new value, equal to the old value plus one. If a second thread reads 
the field between the time a thread reads the old value and writes back a new one, the second thread will see the same 
value as the first and return the same serial number. This is a safety failure: the program computes the wrong results.

One way to fix generateSerialNumber is to add the synchronized modifier to its declaration. This ensures that multiple 
invocations won’t be interleaved and that each invocation of the method will see the effects of all previous invocations. 
Once you’ve done that, you can and should remove the volatile modifier from nextSerialNumber. To bulletproof the method, 
use long instead of int, or throw an exception if nextSerialNumber is about to wrap.

Better still, follow the advice in Item 59 and use the class AtomicLong, which is part of java.util.concurrent.atomic. 
This package provides primitives for lock-free, thread-safe programming on single variables. While volatile provides 
only the communication effects of synchronization, this package also provides atomicity. This is exactly what we want 
for generateSerialNumber, and it is likely to outperform the synchronized version:
```aidl
// Lock-free synchronization with java.util.concurrent.atomic
private static final AtomicLong nextSerialNum = new AtomicLong();

public static long generateSerialNumber() {
    return nextSerialNum.getAndIncrement();
}
```

The best way to avoid the problems discussed in this item is not to share mutable data. <b>Either share immutable data 
(Item 17) or don’t share at all.</b> In other words, <b>confine mutable data to a single thread.</b>
If you adopt this policy, it is important to document it so that the policy is maintained as your program evolves.
It is also important to have a deep understanding of the frameworks and libraries you’re using because they may 
introduce threads that you are unaware of.
{Aaron notes: Above is an important design.}

It is acceptable for one thread to modify a data object for a while and then to share it with other threads, 
synchronizing only the act of sharing the object reference. Other threads can then read the object without further 
synchronization, so long as it isn’t modified again. Such objects are said to be effectively immutable [Goetz06, 3.5.4]. 
Transferring such an object reference from one thread to others is called safe publication [Goetz06, 3.5.3]. There are 
many ways to safely publish an object reference: you can store it in a static field as part of class initialization; 
you can store it in a volatile field, a final field, or a field that is accessed with normal locking; or you can put 
it into a concurrent collection (Item 81).

### In summary, <b>when multiple threads share mutable data, each thread that reads or writes the data must perform synchronization.</b> In the absence of synchronization, there is no guarantee that one thread’s changes will be visible to another thread. The penalties for failing to synchronize shared mutable data are liveness and safety failures. These failures are among the most difficult to debug. They can be intermittent and timing-dependent, and program behavior can vary radically from one VM to another. If you need only inter-thread communication, and not mutual exclusion, the volatile modifier is an acceptable form of synchronization, but it can be tricky to use correctly.

## ITEM 79: AVOID EXCESSIVE SYNCHRONIZATION
Item 78 warns of the dangers of insufficient synchronization. This item concerns the opposite problem. Depending on 
the situation, excessive synchronization can cause reduced performance, deadlock, or even nondeterministic behavior.

To avoid liveness and safety failures, never cede control to the client within a synchronized method or block. In other
words, inside a synchronized region, do not invoke a method that is designed to be overridden, or one provided by a 
client in the form of a function object (Item 24). From the perspective of the class with the synchronized region, 
such methods are alien. The class has no knowledge of what the method does and has no control over it. Depending on 
what an alien method does, calling it from a synchronized region can cause exceptions, deadlocks, or data corruption.
{Aaron notes: Above is an important design.}

To make this concrete, consider the following class, which implements an observable set wrapper. It allows clients to 
subscribe to notifications when elements are added to the set. This is the Observer pattern [Gamma95]. For brevity’s 
sake, the class does not provide notifications when elements are removed from the set, but it would be a simple matter 
to provide them. This class is implemented atop the reusable ForwardingSet from Item 18 (page 90):
```aidl
// Broken - invokes alien method from synchronized block!
public class ObservableSet<E> extends ForwardingSet<E> {

    public ObservableSet(Set<E> set) { super(set); }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized(observers) {
            observers.add(observer);
        }
    }

    public boolean removeObserver(SetObserver<E> observer) {
        synchronized(observers) {
            return observers.remove(observer);
        }
    }
    
    private void notifyElementAdded(E element) {
        synchronized(observers) {
            for (SetObserver<E> observer : observers)
                observer.added(this, element);
        }
    }

    @Override 
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
            notifyElementAdded(element);
        return added;
    }

    @Override 
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
            result |= add(element);  // Calls notifyElementAdded
        return result;
    }
}
```
Observers subscribe to notifications by invoking the addObserver method and unsubscribe by invoking the removeObserver 
method. In both cases, an instance of this callback interface is passed to the method.
```aidl
@FunctionalInterface 
public interface SetObserver<E> {
    // Invoked when an element is added to the observable set
    void added(ObservableSet<E> set, E element);
}
```

This interface is structurally identical to BiConsumer<ObservableSet<E>,E>. We chose to define a custom functional 
interface because the interface and method names make the code more readable and because the interface could evolve to 
incorporate multiple callbacks. That said, a reasonable argument could also be made for using BiConsumer (Item 44).

On cursory inspection, ObservableSet appears to work fine. For example, the following program prints the numbers from 
0 through 99:

```aidl
public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

    set.addObserver((s, e) -> System.out.println(e));

    for (int i = 0; i < 100; i++)
        set.add(i);
}
```

Now let’s try something a bit fancier. Suppose we replace the addObserver call with one that passes an observer that 
prints the Integer value that was added to the set and removes itself if the value is 23:

```aidl
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23)
            s.removeObserver(this);
    }
});
```
Note that this call uses an anonymous class instance in place of the lambda used in the previous call. That is 
because the function object needs to pass itself to s.removeObserver, and lambdas cannot access themselves (Item 42).

You might expect the program to print the numbers 0 through 23, after which the observer would unsubscribe and the 
program would terminate silently. In fact, it prints these numbers and then throws a ConcurrentModificationException. 
The problem is that notifyElementAdded is in the process of iterating over the observers list when it invokes the 
observer’s added method. The added method calls the observable set’s removeObserver method, which in turn calls the 
method observers.remove. Now we’re in trouble. We are trying to remove an element from a list in the midst of 
iterating over it, which is illegal. The iteration in the notifyElementAdded method is in a synchronized block to 
prevent concurrent modification, but it doesn’t prevent the iterating thread itself from calling back into the 
observable set and modifying its observers list.

Now let’s try something odd: let’s write an observer that tries to unsubscribe, but instead of calling removeObserver 
directly, it engages the services of another thread to do the deed. This observer uses an executor service (Item 80):

```aidl
// Observer that uses a background thread needlessly
set.addObserver(new SetObserver<>() {
   public void added(ObservableSet<Integer> s, Integer e) {
      System.out.println(e);
      if (e == 23) {
         ExecutorService exec = Executors.newSingleThreadExecutor();
         try {
            exec.submit(() -> s.removeObserver(this)).get();
         } catch (ExecutionException | InterruptedException ex) {
            throw new AssertionError(ex);
         } finally {
            exec.shutdown();
         }
      }
   }
});
```
Incidentally, note that this program catches two different exception types in one catch clause. This facility, 
informally known as multi-catch, was added in Java 7. It can greatly increase the clarity and reduce the size of 
programs that behave the same way in response to multiple exception types.

When we run this program, we don’t get an exception; we get a deadlock. The background thread calls s.removeObserver, 
which attempts to lock observers, but it can’t acquire the lock, because the main thread already has the lock. All the 
while, the main thread is waiting for the background thread to finish removing the observer, which explains the deadlock.

This example is contrived because there is no reason for the observer to use a background thread to unsubscribe itself, 
but the problem is real. Invoking alien methods from within synchronized regions has caused many deadlocks in real 
systems, such as GUI toolkits.

In both of the previous examples (the exception and the deadlock) we were lucky. The resource that was guarded by the 
synchronized region (observers) was in a consistent state when the alien method (added) was invoked. Suppose you were 
to invoke an alien method from a synchronized region while the invariant protected by the synchronized region was 
temporarily invalid. Because locks in the Java programming language are reentrant, such calls won’t deadlock. As in 
the first example, which resulted in an exception, the calling thread already holds the lock, so the thread will 
succeed when it tries to reacquire the lock, even though another conceptually unrelated operation is in progress on 
the data guarded by the lock. The consequences of such a failure can be catastrophic. In essence, the lock has 
failed to do its job. Reentrant locks simplify the construction of multithreaded object-oriented programs, but they 
can turn liveness failures into safety failures.
{Aaron notes: Above is an important design.}

Luckily, it is usually not too hard to fix this sort of problem by moving alien method invocations out of synchronized 
blocks. For the notifyElementAdded method, this involves taking a “snapshot” of the observers list that can then be 
safely traversed without a lock. With this change, both of the previous examples run without exception or deadlock:
```aidl
// Alien method moved outside of synchronized block - open calls
private void notifyElementAdded(E element) {

    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```
In fact, there’s a better way to move the alien method invocations out of the synchronized block. The libraries provide
a concurrent collection (Item 81) known as CopyOnWriteArrayList that is tailor-made for this purpose. This List 
implementation is a variant of ArrayList in which all modification operations are implemented by making a fresh copy 
of the entire underlying array. Because the internal array is never modified, iteration requires no locking and is 
very fast. For most uses, the performance of CopyOnWriteArrayList would be atrocious, but it’s perfect for observer 
lists, which are rarely modified and often traversed.
{Aaron notes: Above is an important design.}

The add and addAll methods of ObservableSet need not be changed if the list is modified to use CopyOnWriteArrayList. 
Here is how the remainder of the class looks. Notice that there is no explicit synchronization whatsoever:
```aidl
// Thread-safe observable set with CopyOnWriteArrayList
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```

An alien method invoked outside of a synchronized region is known as an open call [Goetz06, 10.1.4]. Besides preventing 
failures, open calls can greatly increase concurrency. An alien method might run for an arbitrarily long period. If 
the alien method were invoked from a synchronized region, other threads would be denied access to the protected 
resource unnecessarily.

<b>As a rule, you should do as little work as possible inside synchronized regions. </b>
1. Obtain the lock, 
2. examine the shared data, 
3. transform it as necessary, 
4. and drop the lock. 
If you must perform some time-consuming activity, find a way to move it out of the synchronized region without 
violating the guidelines in Item 78.
{Aaron notes: Above is an important design.}

The first part of this item was about correctness. Now let’s take a brief look at performance. While the cost of 
synchronization has plummeted since the early days of Java, it is more important than ever not to oversynchronize. 
In a multicore world, the real cost of excessive synchronization is not the CPU time spent getting locks; it is 
contention: the lost opportunities for parallelism and the delays imposed by the need to ensure that every core has a 
consistent view of memory. Another hidden cost of oversynchronization is that it can limit the VM’s ability to 
optimize code execution.
{Aaron notes: Above is an important design.}

If you are writing a mutable class, you have two options: you can omit all synchronization and allow the client to 
synchronize externally if concurrent use is desired, or you can synchronize internally, making the class 
thread-safe (Item 82). You should choose the latter option only if you can achieve significantly higher concurrency 
with internal synchronization than you could by having the client lock the entire object externally. The collections 
in java.util (with the exception of the obsolete Vector and Hashtable) take the former approach, while those in 
java.util.concurrent take the latter (Item 81).
{Aaron notes: Above is an important design.}

In the early days of Java, many classes violated these guidelines. For example, StringBuffer instances are almost 
always used by a single thread, yet they perform internal synchronization. It is for this reason that StringBuffer 
was supplanted by StringBuilder, which is just an unsynchronized StringBuffer. Similarly, it’s a large part of the 
reason that the thread-safe pseudorandom number generator in java.util.Random was supplanted by the unsynchronized 
implementation in java.util.concurrent.ThreadLocalRandom. When in doubt, do not synchronize your class, but document 
that it is not thread-safe.
{Aaron notes: Above is an important design.}

If you do synchronize your class internally, you can use various techniques to achieve high concurrency, such as lock 
splitting, lock striping, and nonblocking concurrency control. These techniques are beyond the scope of this book, 
but they are discussed elsewhere [Goetz06, Herlihy08].
{Aaron notes: Above is an important design.}

## ITEM 80: PREFER EXECUTORS, TASKS, AND STREAMS TO THREADS

The first edition of this book contained code for a simple work queue [Bloch01, Item 49]. This class allowed clients to 
enqueue work for asynchronous processing by a background thread. When the work queue was no longer needed, the client 
could invoke a method to ask the background thread to terminate itself gracefully after completing any work that was 
already on the queue. The implementation was little more than a toy, but even so, it required a full page of subtle, 
delicate code, of the sort that is prone to safety and liveness failures if you don’t get it just right. Luckily, 
there is no reason to write this sort of code anymore.

By the time the second edition of this book came out, java.util.concurrent had been added to Java. This package contains 
an Executor Framework, which is a flexible interface-based task execution facility. Creating a work queue that is better 
in every way than the one in the first edition of this book requires but a single line of code:
```aidl
ExecutorService exec = Executors.newSingleThreadExecutor();
```
Here is how to submit a runnable for execution:
```aidl
exec.execute(runnable);
```
And here is how to tell the executor to terminate gracefully 
(if you fail to do this, it is likely that your VM will not exit):
```aidl
exec.shutdown();
```

You can do many more things with an executor service. For example, you can wait for a particular task to complete 
(with the get method, as shown in Item 79, page 319), you can wait for any or all of a collection of tasks to complete 
(using the invokeAny or invokeAll methods), you can wait for the executor service to terminate 
(using the awaitTermination method), you can retrieve the results of tasks one by one as they complete 
(using an ExecutorCompletionService), you can schedule tasks to run at a particular time or to run periodically 
(using a ScheduledThreadPoolExecutor), and so on.
{Aaron notes: Above is an important design.}

If you want more than one thread to process requests from the queue, simply call a different static factory that 
creates a different kind of executor service called a thread pool. You can create a thread pool with a fixed or 
variable number of threads. The java.util.concurrent.Executors class contains static factories that provide most of 
the executors you’ll ever need. If, however, you want something out of the ordinary, you can use the ThreadPoolExecutor 
class directly. This class lets you configure nearly every aspect of a thread pool’s operation.

Choosing the executor service for a particular application can be tricky. For a small program, or a lightly loaded 
server, Executors.newCachedThreadPool is generally a good choice because it demands no configuration and generally 
“does the right thing.” <b>But a cached thread pool is not a good choice for a heavily loaded production server!</b> In a 
cached thread pool, submitted tasks are not queued but immediately handed off to a thread for execution. If no 
threads are available, a new one is created. If a server is so heavily loaded that all of its CPUs are fully utilized 
and more tasks arrive, more threads will be created, which will only make matters worse. Therefore, in a heavily 
loaded production server, you are much better off using Executors.newFixedThreadPool, which gives you a pool with a 
fixed number of threads, or using the ThreadPoolExecutor class directly, for maximum control.
{Aaron notes: Above is an important design.}

Not only should you refrain from writing your own work queues, but you should generally refrain from working directly 
with threads. When you work directly with threads, a Thread serves as both a unit of work and the mechanism for 
executing it. In the executor framework, the unit of work and the execution mechanism are separate. The key 
abstraction is the unit of work, which is the task. There are two kinds of tasks: Runnable and its close cousin, 
Callable (which is like Runnable, except that it returns a value and can throw arbitrary exceptions). The general 
mechanism for executing tasks is the executor service. If you think in terms of tasks and let an executor service 
execute them for you, you gain the flexibility to select an appropriate execution policy to meet your needs and to 
change the policy if your needs change. In essence, the Executor Framework does for execution what the Collections 
Framework did for aggregation.

In Java 7, the Executor Framework was extended to support fork-join tasks, which are run by a special kind of 
executor service known as a fork-join pool. A fork-join task, represented by a ForkJoinTask instance, may be split up 
into smaller subtasks, and the threads comprising a ForkJoinPool not only process these tasks but “steal” tasks from 
one another to ensure that all threads remain busy, resulting in higher CPU utilization, higher throughput, and lower 
latency. Writing and tuning fork-join tasks is tricky. Parallel streams (Item 48) are written atop fork join pools 
and allow you to take advantage of their performance benefits with little effort, assuming they are appropriate for 
the task at hand.
{Aaron notes: Above is an important design.}

### A complete treatment of the Executor Framework is beyond the scope of this book, but the interested reader is directed to Java Concurrency in Practice [Goetz06].

## ITEM 81: PREFER CONCURRENCY UTILITIES TO WAIT AND NOTIFY

## ITEM 82: DOCUMENT THREAD SAFETY

## ITEM 83: USE LAZY INITIALIZATION JUDICIOUSLY

## ITEM 84: DON’T DEPEND ON THE THREAD SCHEDULER