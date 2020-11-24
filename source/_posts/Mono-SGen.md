---
title: Mono内存管理之SGen
date: 2020-10-2 14:04:10
tags:
- GC
- Mono
---
&emsp; 在使用c#进行Unity开发时, 构建一个对象只需要New对象构造函数()，我们并不知道如何回收该对象。那么就有一个问题: **该对象在创建时，其内存是如何分配的；以及何时对该对象进行回收，如何回收其对应内存？** 由于c#并没有提供直接可操作内存的方法（unsafe除外，一般不会使用），所以上面的问题都交给了一个叫做**垃圾回收器**的东西，它既为创建对象分配内存，又回收对象内存。在Mono中使用的垃圾回收器为**SGen**, 这是一款专门为Mono设计的垃圾回收器，下面就来初步认识一下SGen。
<!-- more -->
&emsp; SGen的内存托管堆由三个部分构成：**Nursery, Major Heap, Large Object Space**。
![](/images/Mono-SGen/1.png)

&emsp; **Nursery**：又称为Young Generation, 主要用来创建内存较小并且生命周期较短的一些对象，该托管堆是使用的是比较常见的**Semi-Copying Collector**进行内存回收。Nursery托管堆默认大小为4M，当Nursery占用比例达到一定阈值后， SGen会将Nursery中的对象移动到**Major Heap**，并清空Nursery。
![](/images/Mono-SGen/2.png)
&emsp; **Major Heap**：又称为Old Generation, 顾名思义存放的都是一些‘老顽固’，SGen提供了两种垃圾回收器给Major Heap，分别是**Copying Collector**和**Mark and Sweep Collector**默认使用Mark and Sweep Collector 。
&emsp; **large Object Space**： SGen默认大于8000bytes的对象称之为Large Object，它在创建时直接的向操作系统申请内存，回收时也是及时将内存还给系统。
# Nursery
&emsp; Nursery作为生命周期较短的对象产生和回收地，随着对象的回收内存中会产生大量的大小不确定内存碎片，分散在Nursery中，导致内存不能合理的利用。
&emsp; 我们来看一个产生内存碎片的过程：
1）创建一个长度为4096字节的对象a;
2) 创建一个长度为1024字节的对象b;
3) 创建一个长度为1024字节的对象c;
4) 回收对象a;
5) 创建一个长度为1024字节的对象d;
6) 回收c;
7) 创建一个长度为4096的对象e;
![](/images/Mono-SGen/3.png)
&emsp; 基于Nursery的特性，SGen为它提供了一种名为Copying Collector的垃圾回收算法解决上述问题。
## Semi-Space/Copying Collector
&emsp; Copying  Collector又称为Semi-Space Collector是因为：它将内存对半分，回收的时候遍历root表查找到还存在引用的对象，然后将他们复制到内存另外一半空间，并整齐的排列，同时清除内存的另外一半空间 。这样我们发现经过复制操作后，成功的消除了内存中的碎片。下面通过一个例子来理解一下复制这个过程：
&emsp; 首先创建了Obj1-5，共5个对象；
![](/images/Mono-SGen/4.png)
&emsp; GC时Obj2和Obj4还存在引用，将他们复制到内存的另外一半，并整齐排列；
![](/images/Mono-SGen/5.png)
&emsp; 清除内存另外一半，故 Obj1,Obj3和Obj5所占用的内存被回收；
![](/images/Mono-SGen/6.png)
&emsp; 内存中又产生了两个新的对象Obj7和Obj8;
![](/images/Mono-SGen/7.png)
&emsp; 再次GC将Obj4,Obj6和Obj8复制，并回收Obj2和Obj7；
![](/images/Mono-SGen/8.png)
&emsp; 通过上述过程我们发现，内存碎片在复制的过程中就消失了。

## 重定向forwarding
&emsp; 我们知道当Nursery中的对象所占用内存到一定程度后，SGen回将Nursery中的存在引用的对象copy到Major Heap当中。所以copy不仅在copying collector中要用到，在对象copy到Major Heap时也需要，并且完成copy后的对象需要进行地址的重定向。
&emsp; 通过广度优先遍历root表，将存在引用的对象存放的栈中（存在重复对象），然后依次将进行copy操作，那么如何知道一个对象是否已经完成copy？针对上述问题SGen将对象地址最后一位用来标记对象是否已经完成重定向。
&emsp; 下面的伪代码用来描述copy到Major Heap的过程：
````c++
while (!gray_stack_is_empty ()) {
    object = gray_stack_pop ();
    foreach (refp in object) {
        old = *refp;
        if (!ptr_in_nursery (old))
            continue;
        if (object_is_forwarded (old)) {
            new = forwarding_destination (old);
        } else {
             new = major_alloc (object_size (old));
             copy_object (new, old);
             forwarding_set (old, new);
             gray_stack_push (new);
        }
        *refp = new;
    }
 }
````

# Major Heap
&emsp; 通过对Nursery的学习可以发现，Nursery中回频繁触发copy的操作，对象生命周期较短的对象可以达到回收的目的。但是对象生命周期较长的对象来说，copy操作就显得无用，而且浪费性能，所以SGen将他们移动到了Major Heap。
&emsp; Mark-Sweep过程伪代码
````c++
//BFS（广度优先遍历）
```
mark():
    while !isEmpty(worklist):
        ref = remove(worklist)
        for each field in Pointers(ref)  
            child = *field
            if child != null && !isMarked(child)
                setMarked(child)
                add(worklist, child)
                
sweep(start, end):
    scan = start
    while scan < end
        if isMarked(scan)
            unsetMarked(scan)
        else
            free(scan)
        scan = nextObject(scan)
````
## sweep-mark
&emsp; Major Heap新推出的垃圾回收算法叫作Mark-Sweep(标记清除），和Copying Collector不同的是此算法不会将对象在内存中移动。简单来说就是：先扫描Heap，标记出有引用对象，然后执行Sweep回收没有引用的对象。接下来我们来看一下它是如何标记内存对象，以Mark-Sweep是如何避免内存碎片的产生？
## block
&emsp; Mark-Sweep将内存划分为固定大小的Block，并且每一个Block内放的对象的大小都是相同的。这样一来就算对象被回收留下了空位，空位的大小也正好可以分配给一个新的对象。当然对象有各种各种各样的大小，所以SGen需要提前针对不同大小的对象生成Block。
&emsp; Block内存维护了一个freelist，这样的在分配内存的时候就快速从freelist中获取，而省去了搜索的过程。
&emsp; 由于Block内对象都是以内存对齐的方式排列，因此可以使用bitmap用来标记对象。例如一个大小位65535字节的block，对象以16字节对齐，那么block中就有4096个内存空间，对应4096个bit=512个字节。通过block开始地址和bitmap可以很方便的计算出对象的内存地址，并且bitmap结构紧凑，可以一次性加载到内存，提升了内存的局域性。
&emsp; 下面给出使用bitmap进行mark过程的伪代码：
````c++
mark():
    cur = nextInBitmap()
    while cur < heapEnd
        add(worklist, cur)
        markStep(cur)
        cur = nextInBitmap()
      
markStep(start):
    while !isEmpty():
        ref = remove(worklist)
        for each field in Pointers(ref):
            child = *field
            if child != null && !isMarked(child)
                setMarked(child)
                if child > start 
                    add(worklist, child)
````
&emsp; 同样的用来描述Block的BlockInfo数据结构也是顺序的排列，这样就不需要加载对应的Block，而只需要通过block的index去BlockInfo的队伍中去获取。
#### fixed heap
&emsp; 当我们在向系统申请内存时，系统会block当前线程，然后去内核获取，然后完成内存分配恢复对应线程。这是一个很复制的过程，对于游戏引擎来说，需要时再获取会相当耗费性能，可能会导致卡顿。因此SGen会在程序启动时提前申请一块较大的堆内存用域存放Block。
&emsp; 并且当堆内存不够用是，SGen又会向系统再申请一块固定大小的堆内存，这样的设计有一个不好的点就是可能会导致其内存使用率不高。
# 增量GC
&emsp; 传统的GC过程都会导致一个问题：stop-the-world，意思就是在引用分析和清除阶段需要挂起程序，直到这一次GC结束然后恢复程序。STW对于游戏引擎来说最致命就是会导致卡顿。随之Unity2019.1推出了最新的增量式GC，大大的缓解了上述问题。
![](/images/Mono-SGen/10.png)
&emsp; 从上图可以看出，增量式(incremental)GC实际上是将单次MS过程分成了多个小批次，从而导致程序停顿很小，达到近似实时的效果。
&emsp; 实现增量式GC的核心是在之前的算法基础上引入了**三色标记**算法，有兴趣的同学可以研究一下。h(Θ) t(Θ) α
&emsp; 以上就是对Mono内存管理SGen的一个大概描述。也是自己学习过程中对资料的整理加上一些自己的理解，如果不对或不清楚的地方还希望大家能够指出，感谢!
  
    
      
       
       
       
[Mono SGen Doc](https://www.mono-project.com/docs/advanced/garbage-collector/sgen/)
[Copying Carbage Collection](http://www.cs.cornell.edu/courses/cs312/2003fa/lectures/sec24.htm)
[深入浅出垃圾回收（二）Mark-Sweep 详析及其优化](https://liujiacai.net/blog/2018/07/08/mark-sweep/)
[从零开始手敲次世代游戏引擎（十八）](https://zhuanlan.zhihu.com/p/29023579)