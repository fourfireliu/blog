---
title: 关于内存屏障的阅读理解
date: 2016-06-13 16:49:16

tags:
 - 内存
categories:
 - 操作系统
---

​       谈到Java并发，就要讲到Java内存模型，而Java内存模型又和内存屏障(Memory Barriers)有着千丝万缕的联系。Java内存模型遵守happens-before的规则，否则，Java多线程程序的执行顺序和执行结果完全无迹可寻。先来回顾一下happens-before规则：

- 程序顺序规则：一个线程中的每个操作，happens- before 于该线程中的任意后续操作。
- 监视器锁规则：对一个监视器锁的解锁，happens- before 于随后对这个监视器锁的加锁。
- volatile变量规则：对一个volatile域的写，happens- before 于任意后续对这个volatile域的读。
- 传递性：如果A happens- before B，且B happens- before C，那么A happens- before C。

happens-before规则的实现很大一部分是靠JVM编译器在生成的指令序列中插入内存屏障指令来实现的。内存屏障能保证该指令前后的代码操作不会被JVM编译器自作聪明地重排序，导致不可预知的后果。内存屏障的实现，则涉及到了操作系统里CPU cache memeory执行指令的原理和优化。最近有幸读到一篇牛人的论文[Memory Barriers: a Hardware View for Software Hackers](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf)，通俗易懂，深有所得，试阐述一二。

指令的重排序极大的优化了cpu的性能，为了保证多核多线程指令的正常顺序，Memory Barries也应运而生。由于CPU的计算速度远远大于CPU从内存中读取写入数据的速度，为了提升性能，引入了Cache，现代CPU的架构最初如下：

{% asset_img cpu1.png cpu1 %}

当cpu加载数据时，首先尝试从cache中获取，这会导致一个cache miss，cpu将会等待很长时间(相对从cache中获取)从memory中获取该数据，之后每次都是从cache中取这个数据了(除非被其他缓存数据替换)。在多核系统中，当某个cpu要写(刷新)数据时(不管是写到它的cache或者写到memory)，由于是多核系统，要保证所有CPU缓存的一致性，首先要发送消息给其他cpu，通知它们这个数据已废弃(invalidated)，当收到其他所有cpu已废弃的回复后，它可以放心写数据了，如果这个缓存数据的状态(read-only)表示该数据缓存在其他cpu上也有至少一份，那么它就得先实现invalidated，然后再写数据，这称之为write-miss。



### Cache Coherence Protocols

正因为不同的cpu有自己的cache，因此cache一致性(cache-coherence)的需求也就随之而生了。为了支持cache一致性，cache定义了多个状态MESI(modified, exclusive, shared, invalid)。

modified: 表示只有当前cpu cache存有该数据的缓存，而且cache中的数据是最新的，还未刷到内存里。如果当前cache不再想要存放该数据缓存，需要将它刷回内存里。

exclusive: 和modified很相似，不同在于cache中的数据和内存中的数据是一致的，数据退出cache不需要刷到内存里。

shared: 内存中数据是最新的，除了当前cache还有别的cpu cache也有数据的缓存。

invalid: 当前cache缓存无效，意味着新data进cache时，可以优先替换状态为invalid的缓存，因为无需做其他任何操作，不需要刷内存或者通知其他cpu cache保存一致性，不会有任何影响，代价是最低的。



### MESI Protocol Messages

在代码执行过程中，各个CPU cache不断的在MESI状态中转换，在转换过程中，多个cpu之间也有着各种通信来保持一致性。记住，每个cpu管理自己的一个cache，数据状态是cache的，消息是cpu发的

read: 包含了要读取的数据的物理地址(memory地址作为cache的key值)，这是一个读取请求。

read response: 包含要读的数据 可能来自于其他cache或者直接来自memory，这是一个读取请求的回复。

invalidate: 包含了要废弃的memory地址，所有其他cache必须执行废弃该地址的的数据的操作并给出回复。

invalidate acknowledge: 尝试废弃缓存数据后，响应invalidate消息。

read invalidate: read和invalidate的合体，应该就是要其它cache废弃缓存，同时从内存里读出最新数据作为回复。

writeback: 消息中包含memory地址和数据，这条命令就是让处于modified状态的cache把数据刷回内存。



cpu之间消息传送导致cache状态转换关系图如下:

{% asset_img cpu2.png cpu2 %}

- Transition a: modified-exclusive，主要是cpu发出一个writeback消息。
- Transition b: exclusive-modified，cpu更新了exclusive状态的cache数据，这个转换cpu没有发出接收任何消息。
- Transition c: modified-invalidate，cpu收到了针对它的一个modified状态的数据的read-invalidate消息，cpu将会使该数据失效。(看来这里read-invalidate是读cache上的数据 什么时候需要这样的场景呢？为啥需要invalidate它呢？)
- Transition d: invalidate-modified, cpu在一个废弃cache上执行一个read-modify-write操作，它发出一个read-invalidate消息，读取了某个数据的最新缓存值到自己的缓存上，并要求其他cpu invalidate它们可能的缓存，当它收到所有的invalidate response后，该转换完成。(和transition c的刚好一对)
- Transition e: shared-modified, cpu在此cache中执行了一个read-modify-write的操作，意思就是场景就是(a=1) 这么一个操作，但是初始状态是shared 表示其他cpu cache里还有老的缓存，所以要同时给其他所有cpu发一个invalidate消息，等到收到所有的invalidate response这个转换才算完成。
- Transition f: modified-shared, cpu cache这个数据状态是modified 收到一个read消息 然后返回data作为read response(可能还同时把它刷到memory了)(问题来了 shared到底有没有保证多个cache和memory数据一致呢)
- Transition g: exclusive-shared, 这个和transition f很相像 区别就是f的数据来自于cache 有可能会刷回memory，这里则是数据来自于cache或memory，肯定不需要刷回memory。
- Transition h: shared-exclusive, 使用场景是(cpu意识到它马上就要对这个cache的数据进行更新操作，那么它给其他所有的cpu发了一个这个数据的invalidate命令，导致其他cpu将这个缓存都写回到memory里 invalidate掉缓存)，因此这个转换要等收到所有的invalidate acknowledge才完成。这样这个cache是唯一拥有这个数据的缓存了。
- Transition i: exclusive-invalidate, 使用场景(其他某个cpu对一个不在它缓存上的数据要执行一个read-modify-write的原子操作，因此它群发了一个read-invalidate消息，当前cpu收到这个消息，返回了read-response和invalidate acknowledge，这样看来 对exclusive read-invalidate是读缓存的即可)
- Transition j: invalidate-exclusive, 使用场景(该cpu将要执行的代码是给某个在其他cache中有缓存的数据赋新值，因此它群发了一个read-invalidate消息，当它收集齐全read-response和invalidate-acknowledge时，转换完成，这意味着马上它就要做一个exclusive-modified的转换...那就是一行代码花了几个状态切换才完成。)
- Transition k: invalidate-shared, 使用场景(该cpu要加载一个在别的cache的data，所以发了个read，当收到read response时，转换完成)
- Transition l: shared-invalidate, 遭遇了transition j。

<!-- more -->


由上可以发现一点，在transition j中，如果cpu0要给某个在别的cache中有缓存的memory地址赋新值，它必须发出一个invalidate 然后一直等到收集齐invalidate acknowledge才能进行操作。但反过来想想cpu0为什么一定要等cpu1的响应结果呢？无论cpu1那边有还是没有data缓存，cpu0都是要做赋新值的操作的，cpu1的原缓存值是多少然后过了多久做invalidate并不会影响cpu0的赋值结果(我们这里当然是假定cpu间的通信是永远不会丢失的，处于一个理想状态)。

考虑到这点，一个叫做store buffer的东东就出现了，cpu架构图也变成这样：

{% asset_img cpu3.png cpu3 %}

每个cpu和cache间都加了一个store buffer，它上连cpu下连cache，当cpu想写一个变量(给某个memory不管在不在它的cache里 赋新值)时，它直接把这个写操作(memory地址为key:新值为value)发到store buffer里，然后接着做别的操作。当最终其他cpu都invalidate好了(invalidate acknowledge都到齐了)，操作会从store buffer清除，而值会存放到cache里(所以store buffer是进去一个store操作，出来执行一个cache store操作)。

来看点代码:

```java
a = 1;
b = a + 1;
assert(b == 2);
```

看起来这简直是不可能错的。但如果用的是上图的那种架构的cpu，就未必了，一起来看看，假设cpu0的cache缓存了a，cpu1的cache缓存了b，初始值都是0：

1. cpu0执行a=1
2. cpu0发现a不在缓存里，因此cpu0群发一个read-invalidate消息，这就是transition j的场景，从invalidate-exclusive。
3. cpu0将所有的store操作都丢到store buffer里，a=1这个操作也不例外。
4. cpu1收到read-invalidate消息，最终清了cache, 返回read response和invalidate-acknowledge消息
5. cpu0开始执行下一条 b=a+1。
6. cpu0因为异步的store buffer流程没走完，自己的cache里没有，所以只能从cpu1那取，取的还是memory中老的a=0，并存在自己cache这。
7. cpu0于是把自己cache里的b=a+1 得到b=1
8. cpu0的store buffer终于等到了cpu1的invalidate-acknowledge，这样就可以把a=1写到cache里了，注意，这时的cpu0 cache里 a=1。
9. cpu0执行assert(b==2) 返回false。



为什么会这样呢？原因就在cpu0里出现了两份变量a的copy，一份在cache里为0，一份在store buffer里为1。为了解决这个问题，硬件工程师们改变cpu策略，让cpu加载数据时先尝试读store buffer，然后再cache，那么架构图变为:

{% asset_img cpu4.png cpu4 %}

这样，上面第7步，cpu0先从store buffer中取到a为1，因此b=2，然后得到正常结果。

这样就没问题了吗？不，让我们看下面的代码：

```java
var foo(void) {
  a = 1;
  b = 1;
}

var bar(void) {
  while (b == 0) {
  	continue;
  }
  assert(a == 1);
}
```

假设cpu0执行foo()而cpu1执行bar()，同时a存在于cpu1的cache里而b存在于cpu0的cache里，我们来看看操作执行流程：

1. cpu0执行a=1，因为a不在cpu0的cache里，因此cpu0发出一个read invalidate然后把a=1的操作放到store buffer里。 
2. cpu1执行 while(b==0) continue; 这里它不是赋值b，只是尝试获取b，b不在它的cache里，因此它发了个read消息出去。
3. cpu0执行b=1 因此b在它的cache里，它直接把cache里的值改为1就好，注意到在这里cpu0不会把b=1的操作放到store buffer里，虽然它肯定也是先在本地store buffer-cache这样的顺序load一遍b。改好以后b在cache中肯定处于modified状态。
4. cpu0收到read消息，于是将1返回出去，同时改状态为shared(我们先假设modified-shared同时是刷了memory的)，cpu1也收到了read response，于是把b=1存到cache里，它也是shared的。
5. cpu1于是跳出while循环，执行下一步，但这时cpu0的read invalidate还没到，因此cpu1取了自己的老缓存a=0，因此返回false。
6. cpu0的read invalidate紧赶慢赶总算到了，废弃了cpu1的老cache a，但已经太晚了。cpu0收到invalidate acknowledge然后store buffer把a=1 存到cache里 modified。



这里为什么会和想要的结果不一样呢？因为cpu0的对一个变量的修改没有及时通知到cpu1上，但硬件工程师也没辙，因为不可能知道一个cpu的哪些操作会影响到另一个cpu的哪些操作，cpu1只是自顾自的做它的事，不知道有别的cpu已经废弃了它的缓存。因此，内存屏障(memory barrier)出现了：

```java
var foo(void) {
  a = 1;
  smp_mb();
  b = 1;
}

var bar(void) {
  while (b == 0) {
  	continue;
  }
  assert(a == 1);
}
```

smp_mb()的作用是: 导致在下一次store指令之前，cpu必须处理完现有的store buffer，它可以有两个实现，要么在下一次执行store指令，把store buffer的指令执行完；要么把下一次store命令也放到store buffer里，不管这个变量是不是已经在自己的cache里了。这样的话，指令执行流程就变成了这样：

1. cpu0执行a=1，因为a不在cpu0的cache里，因此cpu0发出一个read invalidate然后把a=1的操作放到store buffer里。 
2. cpu1执行 while(b==0) continue; 这里它不是赋值b，只是尝试获取b，b不在它的cache里，因此它发了个read消息出去。
3. cpu0执行smp_mb()，标记出所有目前的store buffer指令(目前a=1)
4. cpu0执行b=1 因为有了smp_mb()标记出store buffer里有值，它只能把b=1也放进store buffer。
5. cpu0收到read消息，于是将初始值0返回出去，同时改状态为shared(我们先假设modified-shared同时是刷了memory的)，cpu1也收到了read response，于是把b=0存到cache里，它也是shared的。
6. cpu1收到关于a的read invalidate消息，返回a=0然后invalidate了自己cache的a。
7. cpu0收到了a=0，存入cache，它现在是modified状态了。它现在开始处理store buffer，在cache中有a，直接改成a=1，接下来是b了，但b现在是share状态，因此发了一个invalidate消息(如果自己没有 那就是发read invalidate消息了).
8. cpu1收到invalidate消息，清掉自己的b=0的cache，在下一次循环中，又要用到b，只能发一个read消息，cpu0先收到之前的invalidate acknowledge消息，把自己的b改成1并刷入memory，现在它的b是exclusive状态，接下来它又收到read消息，将b=1回复给cpu1，同时它的b状态变为shared.
9. cpu1收到b=1后，终于可以跳出while了，接着，发现自己cache没有a，发消息从cpu0获得了正确的a=1，也返回了正确结果。

解决了这个问题，新问题又来了，如果smp_mb经常用的话，那么每次store都需要等sotre buffer里的所有已有store处理完才能执行，而每个store必然伴随着一次read invalidate/invalidate消息的完全执行，那就相当于还是要完全等待每个acknowledge的返回。

怎样才能让invalidate acknowledge快速返回而不是要等cpu们真正把invalidate执行完毕才返回呢？硬件工程师给每个cpu增加了一个invalidate queues，架构图变为：

{% asset_img cpu5.jpg cpu5 %}

cpu收到一个invalidate消息时，立马把它放到invalidate queue里然后返回一个invalidate acknowledge，而不等待执行invalidate.而cpu要发出invalidate消息时，必须确保invalidate queue里没有相关变量或者已经处理完成。但这个新增机制也可能影响memory barrier的正确性，我们接下来看，假设a b 初始都为0,a是shared，b只在cpu0 cache里存在，cpu0执行foo()，cpu1执行bar():

```java
void foo(void) {
    a = 1;
    smp_mb();
    b = 1;
}

void bar(void) {
    while (b == 0) continue;
    assert(a == 1);
}
```

1. cpu0执行a=1，因为是shared，因此存此操作到store buffer里，并且发了一个invalidate出去，cpu1正在执行b == 0，但是b不在它的cache中，因此发出一个read。
2. cpu1收到invalidate消息，把它丢到invalidate queue里，然后立马回复了它。cpu0很快收到了回复，开心地把a刷到缓存里，通过了smp_mb()，此时a在它这是modified状态。
3. cpu0开始接着执行b=1，就它自己有，因此它直接在cache里改了值就行。它又收到read消息，因此刷到内存，返回最新的b=1,同时状态就是shared了。
4. cpu1收到了read response，b=1进缓存shared。终于跳出了while循环，下一步判断a==1,但此时它cache中还有a=0,之前cpu0给它的invalidate呢，还放在invalidate queue里睡大觉呢，苦逼地出错了。最终才慢悠悠地把cache a给invalidate掉。

我们在bar中也加一个memory barrier，保证读cache的时候先确保invalidate queue里清理干净了。
```java
void foo(void) {
    a = 1;
    smp_mb();
    b = 1;
}

void bar(void) {
    while (b == 0) continue;
    smp_mb();
    assert(a == 1);
}
```

其他都一样，当cpu1跳出while循环碰到memory barrier时，它必须要先清理完invalidate queue,把它的cache里的a invalidate掉。现在它的cache没有a了，发出一个read消息被cpu0接收到，cpu0返回了最新的a=1给它，它因此也获得了正确结果。

好了，我们来回顾一下，最终在cpu0和cpu1的流程中都用到了memory barrier来保证正确性。但是cpu0中和invalidate queue没啥关系，cpu1中和store buffer没啥关系。因此，工程师们灵机一动，把它们区分开来，就有了read memory barrier，write memory barrier和all memory barrier。这样就演化出了c中的两个指令smp_wmb()和smp_rmb()。总的看来smp_wmb就是保证当前线程之前store的所有值能被其他线程准确读到，smp_rmb保证了当前线程能读到其他线程改动的各种变量的最新值。


