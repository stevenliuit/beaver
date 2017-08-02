

# 基本概念 #
内存模型的概念

重排序

顺序一致性

# 详细分析 #
volatile的内存语义


锁的内存语义


final域的内存语义

# 进阶概念 #
happens-before


双重检查锁定与延迟初始化


内存模型综述





**锁（lock）的代价**
锁是用来做并发最简单的方式，当然其代价也是最高的。内核态的锁的时候需要操作系统进行一次上下文切换，加锁、释放锁会导致比较多的上下文切换和调度延时，等待锁的线程会被挂起直至锁释放。在上下文切换的时候，cpu之前缓存的指令和数据都将失效，对性能有很大的损失。操作系统对多线程的锁进行判断就像两姐妹在为一个玩具在争吵，然后操作系统就是能决定他们谁能拿到玩具的父母，这是很慢的。用户态的锁虽然避免了这些问题，但是其实它们只是在没有真实的竞争时才有效。
Java在JDK1.5之前都是靠synchronized关键字保证同步的，这种通过使用一致的锁定协议来协调对共享状态的访问，可以确保无论哪个线程持有守护变量的锁，都采用独占的方式来访问这些变量，如果出现多个线程同时访问锁，那第一些线线程将被挂起，当线程恢复执行时，必须等待其它线程执行完他们的时间片以后才能被调度执行，在挂起和恢复执行过程中存在着很大的开销。锁还存在着其它一些缺点，当一个线程正在等待锁时，它不能做任何事。如果一个线程在持有锁的情况下被延迟执行，那么所有需要这个锁的线程都无法执行下去。如果被阻塞的线程优先级高，而持有锁的线程优先级低，将会导致优先级反转(Priority Inversion)。
**乐观锁与悲观锁**
独占锁是一种悲观锁，synchronized就是一种独占锁，它假设最坏的情况，并且只有在确保其它线程不会造成干扰的情况下执行，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。
**volatile的问题**
与锁相比，volatile变量是一和更轻量级的同步机制，因为在使用这些变量时不会发生上下文切换和线程调度等操作，但是volatile变量也存在一些局限：不能用于构建原子的复合操作，因此当一个变量依赖旧值时就不能使用volatile变量。（参考：谈谈volatiile）
volatile只能保证变量对各个线程的可见性，但不能保证原子性。为什么？见我的另外一篇文章：《为什么volatile不能保证原子性而Atomic可以？》
[为什么volatile不能保证原子性而Atomic可以？](http://www.cnblogs.com/Mainz/p/3556430.html)
**Java中的原子操作( atomic operations)**
原子操作指的是在一步之内就完成而且不能被中断。原子操作在多线程环境中是线程安全的，无需考虑同步的问题。在java中，下列操作是原子操作：
all assignments of primitive types except for long and double
all assignments of references
all operations of java.concurrent.Atomic* classes
all assignments to volatile longs and doubles
问题来了，为什么long型赋值不是原子操作呢？例如：
long foo = 65465498L;
实时上java会分两步写入这个long变量，先写32位，再写后32位。这样就线程不安全了。如果改成下面的就线程安全了：
private volatile long foo;
因为volatile内部已经做了synchronized.
**CAS无锁算法**
要实现无锁（lock-free）的非阻塞算法有多种实现方法，其中CAS（比较与交换，Compare and swap）是一种有名的无锁算法。CAS, CPU指令，在大多数处理器架构，包括IA32、Space中采用的都是CAS指令，CAS的语义是“我认为V的值应该为A，如果是，那么将V的值更新为B，否则不修改并告诉V的值实际为多少”，CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。CAS无锁算法的C实现如下：

	int compare_and_swap (int* reg, int oldval, int newval) 
	{
	  ATOMIC();
	  int old_reg_val = *reg;
	  if (old_reg_val == oldval) 
	     *reg = newval;
	  END_ATOMIC();
	  return old_reg_val;
	}

**CAS的开销（CPU Cache Miss problem）**
前面说过了，CAS（比较并交换）是CPU指令级的操作，只有一步原子操作，所以非常快。而且CAS避免了请求操作系统来裁定锁的问题，不用麻烦操作系统，直接在CPU内部就搞定了。但CAS就没有开销了吗？不！有cache miss的情况。这个问题比较复杂，首先需要了解CPU的硬件体系结构 Cpu cache

CPU0 检查本地高速缓存，没有找到缓存线。
请求被转发到 CPU0 和 CPU1 的互联模块，检查 CPU1 的本地高速缓存，没有找到缓存线。
请求被转发到系统互联模块，检查其他三个管芯，得知缓存线被 CPU6和 CPU7 所在的管芯持有。
请求被转发到 CPU6 和 CPU7 的互联模块，检查这两个 CPU 的高速缓存，在 CPU7 的高速缓存中找到缓存线。
CPU7 将缓存线发送给所属的互联模块，并且刷新自己高速缓存中的缓存线。
CPU6 和 CPU7 的互联模块将缓存线发送给系统互联模块。
系统互联模块将缓存线发送给 CPU0 和 CPU1 的互联模块。
CPU0 和 CPU1 的互联模块将缓存线发送给 CPU0 的高速缓存。
CPU0 现在可以对高速缓存中的变量执行 CAS 操作了

以上是刷新不同CPU缓存的开销。最好情况下的 CAS 操作消耗大概 40 纳秒，超过 60 个时钟周期。这里的“最好情况”是指对某一个变量执行 CAS 操作的 CPU 正好是最后一个操作该变量的CPU，所以对应的缓存线已经在 CPU 的高速缓存中了，类似地，最好情况下的锁操作（一个“round trip 对”包括获取锁和随后的释放锁）消耗超过 60 纳秒，超过 100 个时钟周期。这里的“最好情况”意味着用于表示锁的数据结构已经在获取和释放锁的 CPU 所属的高速缓存中了。锁操作比 CAS 操作更加耗时，是因深入理解并行编程 
为锁操作的数据结构中需要两个原子操作。缓存未命中消耗大概 140 纳秒，超过 200 个时钟周期。需要在存储新值时查询变量的旧值的 CAS 操作，消耗大概 300 纳秒，超过 500 个时钟周期。想想这个，在执行一次 CAS 操作的时间里，CPU 可以执行 500 条普通指令。这表明了细粒度锁的局限性。

由此可见，AtomicLong.incrementAndGet的实现用了乐观锁技术，调用了sun.misc.Unsafe类库里面的 CAS算法，用CPU指令来实现无锁自增。所以，AtomicLong.incrementAndGet的自增比用synchronized的锁效率倍增。

public final int getAndIncrement() {  
        for (;;) {  
            int current = get();  
            int next = current + 1;  
            if (compareAndSet(current, next))  
                return current;  
        }  
}  
public final boolean compareAndSet(int expect, int update) {  
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  
}


下面是测试代码：可以看到用AtomicLong.incrementAndGet的性能比用synchronized高出几倍。

CAS的例子：非阻塞堆栈

下面是比非阻塞自增稍微复杂一点的CAS的例子：非阻塞堆栈/ConcurrentStack 。ConcurrentStack 中的 push() 和 pop() 操作在结构上与NonblockingCounter 上相似，只是做的工作有些冒险，希望在 “提交” 工作的时候，底层假设没有失效。push() 方法观察当前最顶的节点，构建一个新节点放在堆栈上，然后，如果最顶端的节点在初始观察之后没有变化，那么就安装新节点。如果 CAS 失败，意味着另一个线程已经修改了堆栈，那么过程就会重新开始。

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
public class ConcurrentStack<E> {
    AtomicReference<Node<E>> head = new AtomicReference<Node<E>>();
    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;
        do {
            oldHead = head.get();
            newHead.next = oldHead;
        } while (!head.compareAndSet(oldHead, newHead));
    }
    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) 
                return null;
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead,newHead));
        return oldHead.item;
    }
    static class Node<E> {
        final E item;
        Node<E> next;
        public Node(E item) { this.item = item; }
    }
}
在轻度到中度的争用情况下，非阻塞算法的性能会超越阻塞算法，因为 CAS 的多数时间都在第一次尝试时就成功，而发生争用时的开销也不涉及线程挂起和上下文切换，只多了几个循环迭代。没有争用的 CAS 要比没有争用的锁便宜得多（这句话肯定是真的，因为没有争用的锁涉及 CAS 加上额外的处理），而争用的 CAS 比争用的锁获取涉及更短的延迟。

在高度争用的情况下（即有多个线程不断争用一个内存位置的时候），基于锁的算法开始提供比非阻塞算法更好的吞吐率，因为当线程阻塞时，它就会停止争用，耐心地等候轮到自己，从而避免了进一步争用。但是，这么高的争用程度并不常见，因为多数时候，线程会把线程本地的计算与争用共享数据的操作分开，从而给其他线程使用共享数据的机会。

CAS的例子3：非阻塞链表

以上的示例（自增计数器和堆栈）都是非常简单的非阻塞算法，一旦掌握了在循环中使用 CAS，就可以容易地模仿它们。对于更复杂的数据结构，非阻塞算法要比这些简单示例复杂得多，因为修改链表、树或哈希表可能涉及对多个指针的更新。CAS 支持对单一指针的原子性条件更新，但是不支持两个以上的指针。所以，要构建一个非阻塞的链表、树或哈希表，需要找到一种方式，可以用 CAS 更新多个指针，同时不会让数据结构处于不一致的状态。

在链表的尾部插入元素，通常涉及对两个指针的更新：“尾” 指针总是指向列表中的最后一个元素，“下一个” 指针从过去的最后一个元素指向新插入的元素。因为需要更新两个指针，所以需要两个 CAS。在独立的 CAS 中更新两个指针带来了两个需要考虑的潜在问题：如果第一个 CAS 成功，而第二个 CAS 失败，会发生什么？如果其他线程在第一个和第二个 CAS 之间企图访问链表，会发生什么？

对于非复杂数据结构，构建非阻塞算法的 “技巧” 是确保数据结构总处于一致的状态（甚至包括在线程开始修改数据结构和它完成修改之间），还要确保其他线程不仅能够判断出第一个线程已经完成了更新还是处在更新的中途，还能够判断出如果第一个线程走向 AWOL，完成更新还需要什么操作。如果线程发现了处在更新中途的数据结构，它就可以 “帮助” 正在执行更新的线程完成更新，然后再进行自己的操作。当第一个线程回来试图完成自己的更新时，会发现不再需要了，返回即可，因为 CAS 会检测到帮助线程的干预（在这种情况下，是建设性的干预）。

这种 “帮助邻居” 的要求，对于让数据结构免受单个线程失败的影响，是必需的。如果线程发现数据结构正处在被其他线程更新的中途，然后就等候其他线程完成更新，那么如果其他线程在操作中途失败，这个线程就可能永远等候下去。即使不出现故障，这种方式也会提供糟糕的性能，因为新到达的线程必须放弃处理器，导致上下文切换，或者等到自己的时间片过期（而这更糟）。
	
	public class LinkedQueue <E> {
	    private static class Node <E> {
	        final E item;
	        final AtomicReference<Node<E>> next;
	        Node(E item, Node<E> next) {
	            this.item = item;
	            this.next = new AtomicReference<Node<E>>(next);
	        }
	    }
	    private AtomicReference<Node<E>> head
	        = new AtomicReference<Node<E>>(new Node<E>(null, null));
	    private AtomicReference<Node<E>> tail = head;
	    public boolean put(E item) {
	        Node<E> newNode = new Node<E>(item, null);
	        while (true) {
	            Node<E> curTail = tail.get();
	            Node<E> residue = curTail.next.get();
	            if (curTail == tail.get()) {
	                if (residue == null) /* A */ {
	                    if (curTail.next.compareAndSet(null, newNode)) /* C */ {
	                        tail.compareAndSet(curTail, newNode) /* D */ ;
	                        return true;
	                    }
	                } else {
	                    tail.compareAndSet(curTail, residue) /* B */;
	                }
	            }
	        }
	    }
	}

高并发环境下优化锁或无锁（lock-free）的设计思路

服务端编程的3大性能杀手：1、大量线程导致的线程切换开销。2、锁。3、非必要的内存拷贝。在高并发下,对于纯内存操作来说,单线程是要比多线程快的, 可以比较一下多线程程序在压力测试下cpu的sy和ni百分比。高并发环境下要实现高吞吐量和线程安全，两个思路：一个是用优化的锁实现，一个是lock-free的无锁结构。但非阻塞算法要比基于锁的算法复杂得多。开发非阻塞算法是相当专业的训练，而且要证明算法的正确也极为困难，不仅和具体的目标机器平台和编译器相关，而且需要复杂的技巧和严格的测试。虽然Lock-Free编程非常困难，但是它通常可以带来比基于锁编程更高的吞吐量。所以Lock-Free编程是大有前途的技术。它在线程中止、优先级倒置以及信号安全等方面都有着良好的表现。

优化锁实现的例子：Java中的ConcurrentHashMap，设计巧妙，用桶粒度的锁和锁分离机制，避免了put和get中对整个map的锁定，尤其在get中，只对一个HashEntry做锁定操作，性能提升是显而易见的（详细分析见《探索 ConcurrentHashMap 高并发性的实现机制》）。
Lock-free无锁的例子：CAS（CPU的Compare-And-Swap指令）的利用和LMAX的disruptor无锁消息队列数据结构等。有兴趣了解LMAX的disruptor无锁消息队列数据结构的可以移步slideshare。
disruptor无锁消息队列数据结构的类图和技术文档下载
http://lmax-exchange.github.io/disruptor/

另外，在设计思路上除了尽量减少资源争用以外，还可以借鉴nginx/node.js等单线程大循环的机制，用单线程或CPU数相同的线程开辟大的队列，并发的时候任务压入队列，线程轮询然后一个个顺序执行。由于每个都采用异步I/O，没有阻塞线程。这个大队列可以使用RabbitMQueue，或是JDK的同步队列（性能稍差），或是使用Disruptor无锁队列(Java)。任务处理可以全部放在内存（多级缓存、读写分离、ConcurrentHashMap、甚至分布式缓存Redis）中进行增删改查。最后用Quarz维护定时把缓存数据同步到DB中。当然，这只是中小型系统的思路，如果是大型分布式系统会非常复杂，需要分而治理，用SOA的思路，参考这篇文章的图。（注：Redis是单线程的纯内存数据库，单线程无需锁，而Memcache是多线程的带CAS算法，两者都使用epoll，no-blocking io）

深入JVM的OS的无锁非阻塞算法

如果深入 JVM 和操作系统，会发现非阻塞算法无处不在。垃圾收集器使用非阻塞算法加快并发和平行的垃圾搜集；调度器使用非阻塞算法有效地调度线程和进程，实现内在锁。在 Mustang（Java 6.0）中，基于锁的 SynchronousQueue 算法被新的非阻塞版本代替。很少有开发人员会直接使用 SynchronousQueue，但是通过 Executors.newCachedThreadPool() 工厂构建的线程池用它作为工作队列。比较缓存线程池性能的对比测试显示，新的非阻塞同步队列实现提供了几乎是当前实现 3 倍的速度。在 Mustang 的后续版本（代码名称为 Dolphin）中，已经规划了进一步的改进。



在上篇《非阻塞同步算法与CAS(Compare and Swap)无锁算法》中讲到在Java中long赋值不是原子操作，因为先写32位，再写后32位，分两步操作，而AtomicLong赋值是原子操作，为什么？为什么volatile能替代简单的锁，却不能保证原子性？这里面涉及volatile，是java中的一个我觉得这个词在Java规范中从未被解释清楚的神奇关键词，在Sun的JDK官方文档是这样形容volatile的：
意思就是说，如果一个变量加了volatile关键字，就会告诉编译器和JVM的内存模型：这个变量是对所有线程共享的、可见的，每次jvm都会读取最新写入的值并使其最新值在所有CPU可见。volatile似乎是有时候可以代替简单的锁，似乎加了volatile关键字就省掉了锁。但又说volatile不能保证原子性（java程序员很熟悉这句话：volatile仅仅用来保证该变量对所有线程的可见性，但不保证原子性）。这不是互相矛盾吗？

不要将volatile用在getAndOperate场合，仅仅set或者get的场景是适合volatile的
不要将volatile用在getAndOperate场合（这种场合不原子，需要再加锁），仅仅set或者get的场景是适合volatile的。
volatile没有原子性举例：AtomicInteger自增
例如你让一个volatile的integer自增（i++），其实要分成3步：1）读取volatile变量值到local； 2）增加变量的值；3）把local的值写回，让其它的线程可见。这3步的jvm指令为：

mov    0xc(%r10),%r8d ; Load
inc    %r8d           ; Increment
mov    %r8d,0xc(%r10) ; Store
lock addl $0x0,(%rsp) ; StoreLoad Barrier
注意最后一步是内存屏障。

内存屏障（memory barrier）是一个CPU指令。基本上，它是这样一条指令： a) 确保一些特定操作执行的顺序； b) 影响一些数据的可见性(可能是某些指令执行后的结果)。编译器和CPU可以在保证输出结果一样的情况下对指令重排序，使性能得到优化。插入一个内存屏障，相当于告诉CPU和编译器先于这个命令的必须先执行，后于这个命令的必须后执行。内存屏障另一个作用是强制更新一次不同CPU的缓存。例如，一个写屏障会把这个屏障前写入的数据刷新到缓存，这样任何试图读取该数据的线程将得到最新值，而不用考虑到底是被哪个cpu核心或者哪颗CPU执行的。

内存屏障（memory barrier）和volatile什么关系？上面的虚拟机指令里面有提到，如果你的字段是volatile，Java内存模型将在写操作后插入一个写屏障指令，在读操作前插入一个读屏障指令。这意味着如果你对一个volatile字段进行写操作，你必须知道：1、一旦你完成写入，任何访问这个字段的线程将会得到最新的值。2、在你写入前，会保证所有之前发生的事已经发生，并且任何更新过的数据值也是可见的，因为内存屏障会把之前的写入值都刷新到缓存。

volatile为什么没有原子性?
明白了内存屏障（memory barrier）这个CPU指令，回到前面的JVM指令：从Load到store到内存屏障，一共4步，其中最后一步jvm让这个最新的变量的值在所有线程可见，也就是最后一步让所有的CPU内核都获得了最新的值，但中间的几步（从Load到Store）是不安全的，中间如果其他的CPU修改了值将会丢失。下面的测试代码可以实际测试voaltile的自增没有原子性：

volatile没有原子性举例：singleton单例模式实现

这是一段线程不安全的singleton（单例模式）实现，尽管使用了volatile：

public class wrongsingleton {
    private static volatile wrongsingleton _instance = null; 
 
    private wrongsingleton() {}
 
    public static wrongsingleton getInstance() {
 
        if (_instance == null) {
            _instance = new wrongsingleton();
        }
 
        return _instance;
    }
}
下面的测试代码可以测试出是线程不安全的：

public class wrongsingleton {
    private static volatile wrongsingleton _instance = null; 
 
    private wrongsingleton() {}
 
    public static wrongsingleton getInstance() {
 
        if (_instance == null) {
            _instance = new wrongsingleton();
            System.out.println("--initialized once.");
        }
 
        return _instance;
    }
}
 
private static void testInit(){
         
        Thread t1 = new Thread(new LoopInit());
        Thread t2 = new Thread(new LoopInit2());
        Thread t3 = new Thread(new LoopInit());
        Thread t4 = new Thread(new LoopInit2());
        t1.start();
        t2.start();
        t3.start();
        t4.start();
         
        while (t1.isAlive() || t2.isAlive() || t3.isAlive()|| t4.isAlive()) {
             
        }
 
    }
输出：有时输出"--initialized once."一次，有时输出好几次
原因自然和上面的例子是一样的。因为volatile保证变量对线程的可见性，但不保证原子性。

附：正确线程安全的单例模式写法：
@ThreadSafe
public class SafeLazyInitialization { 
   private static Resource resource; 
   public synchronized static Resource getInstance() { 
      if (resource == null) 
          resource = new Resource(); 
      return resource; 
    } 
}
另外一种写法：
@ThreadSafe
public class EagerInitialization { 
  private static Resource resource = new Resource(); 
  public static Resource getResource() { return resource; } 
}
延迟初始化的写法：

@ThreadSafe
public class ResourceFactory { 
    private static class ResourceHolder { 
        public static Resource resource = new Resource(); 
    } 
    public static Resource getResource() { 
        return ResourceHolder.resource ; 
    } 
}
二次检查锁定/Double Checked Locking的写法（反模式）

public class SingletonDemo {
    private static volatile SingletonDemo instance = null;//注意需要volatile
  
    private SingletonDemo() {   }
  
    public static SingletonDemo getInstance() {
        if (instance == null) { //二次检查，比直接用独占锁效率高
               synchronized (SingletonDemo .class){
                    if (instance == null) {
                               instance = new SingletonDemo (); 
                    }
             }
        }
        return instance;
    }
}

为什么AtomicXXX具有原子性和可见性？

就拿AtomicLong来说，它既解决了上述的volatile的原子性没有保证的问题，又具有可见性。它是如何做到的？当然就是上文《非阻塞同步算法与CAS(Compare and Swap)无锁算法》提到的CAS（比较并交换）指令。 其实AtomicLong的源码里也用到了volatile，但只是用来读取或写入，见源码：

public class AtomicLong extends Number implements java.io.Serializable {
    private volatile long value;
 
    /**
     * Creates a new AtomicLong with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicLong(long initialValue) {
        value = initialValue;
    }
 
    /**
     * Creates a new AtomicLong with initial value {@code 0}.
     */
    public AtomicLong() {
    }
其CAS源码核心代码为：

int compare_and_swap (int* reg, int oldval, int newval) 
{
  ATOMIC();
  int old_reg_val = *reg;
  if (old_reg_val == oldval) 
     *reg = newval;
  END_ATOMIC();
  return old_reg_val;
}
虚拟机指令为：

mov    0xc(%r11),%eax       ; Load
mov    %eax,%r8d            
inc    %r8d                 ; Increment
lock cmpxchg %r8d,0xc(%r11) ; Compare and exchange
因为CAS是基于乐观锁的，也就是说当写入的时候，如果寄存器旧值已经不等于现值，说明有其他CPU在修改，那就继续尝试。所以这就保证了操作的原子性。

**参考资料**
1. 方腾飞. Java并发编程的艺术[M]. 上海:机械工业出版社, 2017.




