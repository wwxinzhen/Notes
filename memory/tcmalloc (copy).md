#### 1、TCMalloc

​	`Thread-Caching Malloc`

​	![img](https://s4.51cto.com/images/blog/202012/21/6116e67b102cdba0e57d12505999e28c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

​	Front-end：负责提供快速分配和重分配内存给应用，有Per-thread cache 和 Per-CPU cache两部分组成

​	Middle-end：负责给Front-end提供缓存。当Front-end缓存内存不够时，从Middle-end申请内存

​	Back-end：负责从操作系统获取内存，并给Middle-end提供缓存使用

​	tcmalloc中每个线程都有独立的线程缓存ThreadCache，线程的内存分配请求会想ThreadCache申请，ThreadCache内存不够用时会向CentralCache申请，CentralCache内存不够用时会向PageHeap申请，PageHeap不够用时会向OS申请。

##### 	1.1 Front-end

​		Front-end处理对特定大小内存的请求，有一个内存缓存用于分配或保存空闲内存。Front-end缓存一次只能由单个线程访问，不需要任何锁，因此大多数分配和释放都很快。

​		Front-end有两种不同的实现模式：

​			per-thread：tcmalloc最初支持对象的per-thread缓存，但会导致内存占用随着线程数增加而增加。现代应用程序可能有大量的线程，会导致每个线程占用累计起来很大，也可能有单个较小线程累积起来占用的内存会很大。

​			per-CPU：tcmalloc近期开始支持per-cpu模式，在这个模式下，系统中的每个cpu逻辑都有自己的缓存，可以从中支配内存。在x86架构，逻辑cpu相当于一个超线程。

##### 	1.2 middle-end

​		middle-end负责向front-end提供内存并将内存返回back-end。middle-end由transfer cache和central free list组成，每个类大小都有一个transfer cache和一个central free list。缓存由互斥锁保护，因此访问缓存会产生串行化成本。

​		transfer cache

​			当front-end请求内存或返回内存时，将访问transfer cache

​			transfer cache保存一个指向空闲内存的指针数组，可以快速地将对象移动到数组中，或者代表front-end从数组中获取对象

​			当一个线程正在分配另一个线程释放的内存时，transfer cache就可以得到内存名称。transfer cache允许内存在两个不同线程之间快速流动

​			如果transfer cache无法满足内存请求，或者没有足够的空间容纳返回的对象，transfer cache将访问central free list

​		central free list

​			central free list使用spans管理内存

##### 	1.3 back-end

​		back-end有三大职责：

​			管理大量未使用的内存块

​			负责在没有合适大小的内存来满足分配请求时从os获取内存

​			负责将不需要的内存返回给操作系统

​		tcmalloc有两种back-end

​			legacy pageheap，管理tcmalloc中page大小的内存

​			支持hugapge的pageheap，以hugepage大小的内存块来管理内存。管理hugepage内存块中内存，使分配器能够通过减少TLB未命中率来提高应用程序性能 

#### 2、动机

​	`TCMalloc`减少了多线程程序中的锁竞争关系。对于小对象，已经基本上达到了零竞争。对于大对象，`TCMalloc`尝试使用恰当粒度和有效的自旋锁。

​	在`ptmalloc2`中，内存不会从一个空间移动到另一个空间。如，在一个Google的应用中，第一阶段可能会为其URL标准化的数据结构分配大约`300MB`内存。当第一阶段结束后，第二阶段将从同样的地址空间开始。如果第二个阶段被安排到了与第一阶段不同的空间内，这个阶段不会复用任何第一阶段留下的的内存，并会给地址空间添加另外一个`300MB`。类似的内存爆炸问题也可以在其他的应用中看到。

​	`TCMalloc`的另一个好处表现在小对象的空间效率，例如，分配N个8字节对象可能要使用大约`8N * 1.01`字节的空间，即多用百分之一的空间。而`ptmalloc2`中每个对象都使用了一个四字节的头，我认为并将最终的尺寸取整为8字节的倍数，最后使用了`16N`字节。

#### 3、使用

##### 3.1 以动态库方式	

​	通过`"-ltcmalloc"`或`-ltcmalloc_minimal`链接器标志接入你的应用即可。

​	你也可以通过使用`LD_PRELOAD`在不是你自己编译的应用中使用`tcmalloc`：

​		`$ LD_PRELOAD = ''usr/lib/libtcmalloc.so"`

​	`TCMalloc`还包含了一个堆检查器以及一个堆测量器，如果你要链接不包含上述两个组件，你应该链接`libtcmalloc_minimal`

##### 3.2 以静态库方式

​	`gperftools`的`README`中说静态库应该使用`libtcmalloc_and_profiler.a`库而不是`libprofiler.a`和`libtcmalloc.a`，但简单测试后者也是OK的，而且在实际项目中也是用的后者，不知道是不是文档太过老旧了。

#### 4、TCMalloc如何生效

##### 4.1 使用glibc（不使用GCC编译器）

​	在glibc中，内存分配相关的函数都是弱符号（weak symbol），因此TCMalloc只需要定义自己的函数将其覆盖即可。

```c++
//libc_override_redefine.h
extern "C" 
{ 
  void* malloc(size_t s)                         { return tc_malloc(s); }
  void  free(void* p)                            { tc_free(p);          }
}  
```

##### 4.2 使用GCC编译器，使用函数属性：alias

```c++
//libc_override_gcc_and_weak.h
#define ALIAS(tc_fn)   __attribute__ ((alias (#tc_fn), used))
extern "C"
{ 
  void* malloc(size_t size) __THROW               ALIAS(tc_malloc);
  void free(void* ptr) __THROW                    ALIAS(tc_free);
}
```

#### 5、TCMalloc初始化

##### 5.1 何时初始化

​	TCMalloc定义了一个类TCMallocGuard，并在文件tcmalloc.cc中定义了该类型的静态变量module_enter_exit_hook，在其构造函数中执行TCMalloc的初始化逻辑，以确保TCMalloc在main()函数之前完成初始化，防止在初始化之前就有多个线程

```c++
//tcmalloc.cc
static TCMallocGuard module_enter_exit_hook;
```

##### 5.2 如何初始化

```c++
TCMallocGuard::TCMallocGuard()
```

##### 5.3 初始化内容

- ​	初始化SizeMap（size class)
- ​	初始化Allocator
- ​	初始化CentralCache
- ​	初始化PageHeap

#### 6、TCMalloc内存分配算法

按照分配内存大小，TCMalloc将内存分为三类：

- ​	小对象分配：(0, 256KB]
- ​	中对象分配：(256KB, 1MB]
- ​	大对象分配：(1MB, ···)

##### 6.1 小对象分配

###### 6.1.1 size class

​	TCMalloc按大小划分了88个类别，每个类别对应一个大小，比如8字节，16字节·····。TCMalloc会首先将所申请的内存大小向上取整到size class的大小，比如1~8字节之间都会分配8字节。因此这里会造成内部碎片。TCMalloc将这里的内部碎片控制在12.5%以内。

###### 6.1.2 ThreadCache

​	每个ThreadCache中对于每个size class都有一个单独的FreeList，缓存了n个还未被应用程序使用的空闲对象。

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-ThreadCache.svg" alt="img" style="zoom:50%;" />

###### 6.1.3 CentralCache

​	ThreadCache中的空闲对象来自CentralCache，被所有线程公用的缓存。

​	跟ThreadCache类似，CentralCache中对于每个size class也都有一个单独的链表来缓存空闲对象，称之为CentralFreeList。

CentralCache的简化结构如下：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-CentralCache.svg" alt="img" style="zoom:50%;" />

###### 6.1.4 PageHeap

​	CentralCache中的空闲对象来自PageHeap。当CentralCache中的空闲对象不够用时，CentralCache会向PageHeap申请一块内存（可能来自PageHeap的缓存，也可能向系统申请新的内存），并将其拆分成一系列空闲对象，添加到对应size class的CentralFreeList中。

​	PageHeap内部根据内存块（span）的大小采取了两种不同的策略。128个page以内的span，每个大小都用一个链表来缓存，超过128个page的span，存储与一个有序set。PageHeap简化结构如下：

​	<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-PageHeap.svg" alt="img" style="zoom:50%;" />

###### 6.1.5 内存回收

应用程序调用free()或者delete一个小对象时，仅仅是将其插入到ThreadCache中size class对应的FreeList中而已，不需要加锁。

只有当满足一定的条件时，ThreadCache中的空闲对象才会重新放回CentralCache中，以供其他线程取用。当满足一定条件时，CentralCache中的空闲对象也会还给PageHeap，PageHeap再还给系统。

###### 6.1.6 小结

小对象内存分配大致流程：

- 将要分配的内存大小映射到对应的size class。

- 查看ThreadCache中该size class对应的FreeList。

- 如果FreeList非空，则移除FreeList的第一个空闲对象并将其返回，分配结束。

- 如果FreeList是空的：

  - 从CentralCache中size class对应的CentralFreeList获取一堆空闲对象。
    - 如果CentralFreeList也是空的，则：
      - 向PageHeap申请一个span。
      - 拆分成size class对应大小的空闲对象，放入CentralFreeList中。

  - 将这堆对象放置到ThreadCache中size class对应的FreeList中（第一个对象除外）。
  - 返回从CentralCache获取的第一个对象，分配结束。

##### 6.2 中对象分配

采取与小对象不同的分配策略。。

首先，TCMalloc会将应用程序所要申请的内存大小向上取整到**整数个page**（因此，这里会产生1B~8KB的内部碎片）。之后的操作表面上非常简单，向PageHeap申请一个指定page数量的span并返回其起始地址即可：

```c++
Span* span = Static::pageheap()->New(num_pages);
result = (PREDICT_FALSE(span == NULL) ? NULL : SpanToMallocResult(span));
return result;
```

对128个page以内的span和超过128个page的span，PageHeap采取的缓存策略不一样。为了描述方便，以下将128个page以内的span称为小span，大于128个page的span称为大span。

PageHeap中有128个小span的链表，分别对应1~128个page的span：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-SpanList.svg" alt="img" style="zoom:50%;" />

假设要分配一块内存，其大小经过向上取整之后对应k个page，因此需要从PageHeap取一个大小为k个page的span，过程如下：

- 从k个page的span链表开始，到128个page的span链表，按顺序找到第一个非空链表。
- 取出这个非空链表中的一个span，假设有n个page，将这个span拆分成两个span：
  - 一个span大小为k个page，作为分配结果返回。
  - 另一个span大小为n – k个page，重新插入到n – k个page的span链表中。
- 如果找不到非空链表，则将这次分配看做是大对象分配。

##### 6.3 大对象分配

超过1MB（128个page）的内存分配被认为是大对象分配，与中对象分配类似，也是先将所要分配的内存大小向上取整到整数个page，假设是k个page，然后向PageHeap申请一个k个page大小的span。

大对象分配用到的span都是超过128个page的span，其缓存方式不是链表，而是一个按span大小排序的**有序set**（std::set），以便按大小进行搜索。

假设要分配一块超过1MB的内存，其大小经过向上取整之后对应k个page（k>128），或者是要分配一块1MB以内的内存，但无法由中对象分配逻辑来满足，此时k <= 128。不管哪种情况，都是要从PageHeap的span set中取一个大小为k个page的span，其过程如下：

- 搜索set，找到不小于k个page的最小的span（**best-fit**），假设该span有n个page。
- 将这个span拆分为两个span：
  - 一个span大小为k个page，作为结果返回。
  - 另一个span大小为n – k个page，如果n – k > 128，则将其插入到大span的set中，否则，将其插入到对应的小span链表中。
- 如果找不到合适的span，则使用sbrk或mmap向系统申请新的内存以生成新的span，并重新执行中对象或大对象的分配算法。

##### 6.4 小结

TCMalloc的管理内存策略：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-Overview.svg)

#### 7、TCMalloc的实现细节

##### 7.1 Page

**Page是TCMalloc管理内存的基本单位**（这里的page要区分于操作系统管理虚拟内存的page），page的默认大小为8KB，可在configure时通过选项调整为32KB或64KB。

```shell
./configure <other flags> --with-tcmalloc-pagesize=32 (or 64)
```

page越大，TCMalloc的速度相对越快，但其占用的内存也会越高。简单说，就是空间换时间的道理。

###### 7.1.1PageID

TCMalloc并非只将堆内存看做是一个个的page，而是将整个虚拟内存空间都看做是page的集合。从内存地址0x0开始，每个page对应一个递增的PageID，如下图（以32位系统为例）：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-Page.svg)

对于任意内存地址`ptr`，都可通过简单的移位操作来计算其所在page的PageID：

```c++
static const size_t kPageShift  = 13; // page大小：1 << 13 = 8KB
const PageID p = reinterpret_cast<uintptr_t>(ptr) >> kPageShift;
```

即，`ptr`所属page的PageID为`ptr / page_size`。

##### 7.2 Span

一个或多个连续的Page组成一个Span（a contiguous run of pages）。**TCMalloc以Span为单位向系统申请内存。**

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-Span.svg)

一个span记录了起始page的PageID（`start`），以及所包含page的数量（`length`）。

一个span要么被拆分成多个相同size class的小对象用于小对象分配，要么作为一个整体用于中对象或大对象分配。当作用作小对象分配时，span的`sizeclass`成员变量记录了其对应的size class。

span中还包含两个Span类型的指针（`prev`, `next`），用于将多个span以链表的形式存储。

###### 7.2.1 span的三种状态

一个span处于以下三种状态中的一种：

- IN_USE
- ON_NORMAL_FREELIST
- ON_RETURNED_FREELIST

IN_USE，正在使用中。`N_NORMAL_FREELIST`和`ON_RETURNED_FREELIST`都可以认为是空闲状态，区别在于，ON_RETURNED_FREELIST是指span对应的内存已经被PageHeap释放给系统了。

###### 7.2.2 空闲对象链表

被拆分成多个小对象的span还包含了一个记录空闲对象的链表`objects`，由CentralFreeList来维护。

对于新创建的span，将其对应的内存按size class的大小均分成若干小对象，在每一个小对象的起始位置处存储下一个小对象的地址，首首相连：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-FreshSpan.svg)

可以看到，有些小对象已经不再空闲对象链表objects中了，链表中的元素顺序也已经被打乱。

空闲对象链表中的元素乱序没什么影响，因为只有当一个span的所有小对象都被释放之后，CentralFreeList才会将其还给PageHeap。

##### 7.3 PageMap

PageMap缓存了PageID到Span的对应关系

32位系统、x86-64、arm64使用两级PageMap，以32位系统为例：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-PageMap.svg" alt="img" style="zoom:50%;" />

在root_数组中包含512个指向Leaf的指针，每个Leaf又是1024个void*的数组，数组索引为PageID，数组元素为page所属Span的指针。一共$2^{19}$个数组元素，对应32位系统的$2^{19}$个page。

使用两级map可以减少TCMalloc元数据的内存占用，因为初始只会给第一层（即root_数组）分配内存（2KB），第二层只有在实际用到时才会实际分配内存。而如果初始就给2^19^个page都分配内存的话，则会占用2^19^ * 4 bytes = 2MB的内存。

##### 7.4 Size Class

###### 7.4.1 划分跨度

- 16字节以内，每8字节划分一个size class。
  - 满足这种情况的size class只有两个：8字节、16字节。
- 16~128字节，每16字节划分一个size class。
  - 满足这种情况的size class有7个：32, 48, 64, 80, 96, 112, 128字节。
- 128B~256KB，按照每次步进`(size / 8)`字节的长度划分，并且步长需要向下对齐到2的整数次幂，比如：
  - 144字节：128 + 128 / 8 = 128 + 16 = 144
  - 160字节：144 + 144 / 8 = 144 + 18 = 144 + 16 = 160
  - 176字节：160 + 160 / 8 = 160 + 20 = 160 + 16 = 176
  - 以此类推7.4.2

###### 7.4.2 一次移动多个空闲对象

hreadCache会从CentralCache中获取空闲对象，也会将超出限制的空闲对象放回CentralCache。ThreadCache和CentralCache之间的对象移动是**批量**进行的，即一次移动多个空闲对象。CentralCache由于是所有线程公用，因此对其进行操作时需要加锁，一次移动多个对象可以**均摊锁操作的开销**，提升效率。

那么一次批量移动多少呢？每次移动64KB大小的内存，即因size class而异，但至少2个，至多32个（可通过环境变量`TCMALLOC_TRANSFER_NUM_OBJ`调整）。

###### 7.4.3 一次申请多个page

对于每个size class，TCMalloc向系统申请内存时一次性申请n个page（一个span），然后均分成多个小对象进行缓存，以此来**均摊系统调用的开销**。

不同的size class对应的page数量是不同的，如何决定n的大小呢？从1个page开始递增，一直到均分成若干小对象后所剩的空间小于span总大小的1/8为止，因此，浪费的内存被控制在12.5%以内。这是TCMalloc**减少内部碎片**的一种措施。

另外，所分配的page数量还需满足一次移动多个空闲对象的数量要求（源码中的注释是这样说的，不过实际代码是满足1/4即可，原因不明）。

###### 7.4.4 合并操作

TCMalloc会将**相同page数量，相同对象数量**的相邻的size class合并为一个size class。比如：

<img src="https://wallenwang.com/wp-content/uploads/size_merge.svg" alt="img" style="zoom:50%;" />

第30个size class的对象大小是832字节，page数量为1个，因此包含8192 / 832 = 9个小对象。

第31个size class对应的page数量（1个）和对象数量（9个）跟第30个size class完全一样，因此第30个size class和第31个size class合并，所以第30个size class对应的对象大小为896字节。

下一个size class对应的对象大小为1024字节，page数量为1个，因此对象数量是8个，跟第30个size class的对象数量不一样，无法合并。

最终，第30个size class对应的对象大小为896字节。

###### 7.4.5 记录映射关系

一个size class对应：

- 一个对象大小
- 一个申请page的数量
- 一个批量移动对象的数量

TCMalloc将size class与这些信息的映射关系分别记录在三个以size class的编号为索引的数组中（`class_to_size_`，`num_objects_to_move_`， `class_to_pages_`）。

还有一项非常重要的映射关系：小对象大小到size class编号的映射。TCMalloc将其记录在一个一维数组`class_array_`中。

256KB以内都是小对象，而size class的编号用一个字节就可以表示。但由于size class之间是有间隔的，因此可以通过简单的计算对class_array_的索引进行压缩，以减少内存占用。

```c++
// s <= 1024
static inline size_t SmallSizeClass(size_t s) {
  return (static_cast<uint32_t>(s) + 7) >> 3;
}
// s > 1024
static inline size_t LargeSizeClass(size_t s) {
  return (static_cast<uint32_t>(s) + 127 + (120 << 7)) >> 7;
}
```

当s = 256KB时，计算结果即为`class_array_`的最大索引2169，因此数组的大小为2170字节。

###### 7.4.6 计算任意内存地址对应的对象大小

​	当应用程序调用`free()`或`delete`释放内存时，需要有一种方式获取所要释放的内存地址对应的内存大小。结合前文所述的各种映射关系，在TCMalloc中按照以下顺序计算任意内存地址对应的对象大小：

- 计算给定地址计所在的PageID（ptr >> 13）
- 从PageMap中查询该page所在的span
- span中记录了size class编号
- 根据size class编号从`class_to_size_`数组中查询对应的对象大小

这样做的好处是：**不需要在内存块的头部记录内存大小，减少内存的浪费**。

###### 7.4.7 小结

size class的实现有很多省空间省时间的做法：

- 省空间
- 控制划分跨度的最大值（8KB），减少内部碎片
- 控制一次申请page的数量，减少内部碎片
- 通过计算和一系列元数据记录内存地址到内存大小的映射关系，避免在实际分配的内存块中记录内存大小，减少内存浪费
- 两级PageMap或三级PageMap
- 压缩class_array_
- 省时间
- 一次申请多个page
- 一次移动多个空闲对象

##### 7.5 PageHeap

###### 7.5.1 空闲Span管理器

从另一个维度来看，PageHeap是分开管理`ON_NORMAL_FREELIST`和`ON_RETURNED_FREELIST`状态的span的。因此，每个小span对应两个链表，所有大span对应两个set。

```c++
//Page_Heap.h  
//265~275行
```

实际的PageHeap：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-PageHeap2.svg" alt="img" style="zoom:50%;" />

###### 7.5.2 创建Span

```c++
Span* New(Length n);
```

创建span的过程其实就是分配中对象和大对象的过程，假设要创建k个page大小的span（以下简称大小为k的span），过程如下：

- **搜索空闲span：**

```c++
Span* SearchFreeAndLargeLists(Length n);
```

 （1）搜索空闲span链表，按照以下顺序，找出第一个不小于k的span：	

​		从大小为k的span的链表开始依次搜索

​		对于某个大小的span，先搜normal链表，再搜returned链表

（2）如果span链表中没找到合适的span，则搜索存储大span的set：

​		从大小为k的span开始搜索

​		同样先搜normal再搜returned

​		优先使用长度最小并且起始地址最小的span（**best-fit**）

（3）如果通过以上两步找到了一个大小为m的span，则将其拆成两个span：

​		大小为m – k的span重新根据大小和状态放回链表或set中

​		大小为k的span作为结果返回，创建span结束

（4）如果没搜到合适的span，则继续后面的步骤：向系统申请内存。

- **释放所有空闲内存**

```c++
ReleaseAtLeastNPages(static_cast<Length>(0x7fffffff));
```

当没有可用的空闲span时，需要向系统申请新的内存，但在此之前，还有一次避免向系统申请新内存的机会：释放所有空闲内存。向系统申请的内存每达到128MB，且空闲内存超过从系统申请的总内存的1/4，就需要将所有的空闲内存释放。

因为TCMalloc将normal和returned的内存分开管理，而这两种内存不会合并在一起。因此，可能有一段连续的空闲内存符合要求（k个page大小），但因为它既有normal的部分，又有returned的部分，因此前面的搜索规则搜不到它。而释放所有内存可以将normal的内存也变为returned的，然后就可以合并了（合并规则详细后文合并span）。

释放完毕后，会按照前面的搜索规则再次尝试搜索空闲span，如果还搜不到，才继续后续的步骤。

- **向系统申请内存**

```c++
bool GrowHeap(Length n);
```

找不到合适的空闲span，就只能向系统申请新的内存了。

TCMalloc以`sbrk()`和`mmap()`两种方式向系统申请内存，所申请内存的大小和位置均按page对齐，优先使用`sbrk()`，申请失败则会尝试使用`mmap()`（64位系统Debug模式优先使用mmap，原因详见`InitSystemAllocators()`注释）。

- **sbrk()**

下图展示了`SbrkSysAllocator::Alloc()`的执行流程，为了说明外部碎片的产生，并覆盖到`SbrkSysAllocator::Alloc()`的大部分流程，假设page大小为8KB，所申请的内存大小为16KB：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-SbrkAlloc.svg" alt="img" style="zoom:50%;" />

​	（1）假设在申请内存之前，pb（program break，可以认为是堆内存的上边界）指向红色箭头所示位置，即没有在page的边界处。	  	（2）第一次sbrk申请16KB内存，因此pb移至绿色箭头所示位置。

​	（3）由于需要对申请的内存按page对齐，因此需要第二次sbrk，pb指向蓝色箭头所示位置，page的边界处。

​	（4）最终，返回的内存地址为黑色箭头所示位置，黑色和蓝色箭头之间刚好16KB。

可以看出，红色箭头和黑色箭头之间的内存就无法被使用了，产生了**外部碎片**。

- **mmap()**	

如果使用sbrk()申请内存失败，TCMalloc会尝试使用mmap()来分配。同样，为了覆盖`MmapSysAllocator::Alloc()`的大部分情况，下图假设系统的page为4KB，TCMalloc的page为16KB，申请的内存大小为32KB：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-MmapAlloc.svg)

​	（1）假设在申请内存之前，mmap段的边界位于红色箭头所示位置。

​	（2）第一次mmap，会在32KB的基础上，多申请(对齐大小 – 系统page大小) = 16 -4 = 12KB的内存。此时mmap的边界位于绿色箭头所示位置。

​	（3）然后通过两次munmap将所申请内存的两侧边界分别对齐到TCMalloc的page边界。

​	（4）最终申请到的内存为两个蓝色箭头之间的部分，返回左侧蓝色箭头所指示的内存地址。

如果申请内存成功，则创建一个新的span并立即删除，可将其放入空闲span的链表或set中，然后继续后面的步骤。

- **最后的搜索**

最后，重新搜索一次空闲span，如果还找不到合适的空闲span，那就认为是创建失败了。

至此，创建span的操作结束。

###### 7.5.3 删除span

```c++
void Delete(Span* span);
```

当span所拆分成的小对象全部被应用程序释放变为空闲对象，或者作为中对象或大对象使用的span被应用程序释放时，需要将span删除。不过并不是真正的删除，而是放到空闲span的链表或set中。

删除的操作非常简单，但可能会触发合并span的操作，以及释放内存到系统的操作。

- **合并span**

当span被delete时，会尝试向前向后合并一个span。

合并规则如下：

​	（1）只有在虚拟内存空间上连续的span才可以被合并。	

​	（2）只有同是normal状态的span或同是returned状态的span才可以被合并。

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-SpanMerge1.svg)

上图中，被删除的span的前一个span是normal状态，因此可以与之合并，而后一个span是returned状态，无法与之合并。合并操作之后如下图：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-SpanMerge2.svg)

- **释放span**

```c++
Length ReleaseAtLeastNPages(Length num_pages);
```

释放规则：

- 从小到大循环，按顺序释放空闲span，直到释放的page数量满足需求。
- 多次调用会从上一次循环结束的位置继续循环，而不会重新从头（1 page）开始。
- 释放的过程中能合并span就合并span
- 可能释放少于num_pages，没那么多free的span了。
- 可能释放多于num_pages，还差一点就够num_pages了，但刚好碰到一个很大的span。

##### 7.6 ThreadCache

###### 7.6.1 每个线程Cache

每个线程一个ThreadCache是通过这两种技术：Thread Local Storage（TLS)，和Thread Specific Data（TSD）实现。两者的功能基本是一样的，都是提供每线程存储。TLS用起来更方便，读取数据更快，但在线程销毁时TLS无法执行清理操作，而TSD可以，因此TCMalloc使用TSD为每个线程提供一个ThreadCache，如果TLS可用，则同时使用TLS保存一份拷贝以加速数据的访问。

详细可参考源码中`ThreadCache::CreateCacheIfNecessary()`函数和`threadlocal_data_`变量相关代码。

###### 7.6.2 何时创建ThreadCache

当某线程第一次申请分配内存时，TCMalloc为该线程创建其专属的ThreadCache。

```c++
ThreadCache::GetCache() -> ThreadCache::CreateCacheIfNecessary()
```

###### 7.6.3 何时销毁ThreadCache

当一个线程销毁时，其对应的ThreadCache会由该函数销毁。

###### 7.6.4 ThreadCache的大小

所有线程的ThreadCache的总大小限制（`overall_thread_cache_size_`）默认为32MB（`kDefaultOverallThreadCacheSize`），取值范围512KB~1GB，可以通过环境变量`TCMalloc_MAX_TOTAL_THREAD_CACHE_BYTES`或以下方式来进行调整：

```c++
MallocExtension::instance()->SetNumericProperty("TCMalloc.max_total_thread_cache_bytes", value);
```

每个线程的ThreadCache的大小限制默认为4MB（`kMaxThreadCacheSize`）。调整ThreadCache总大小时，会修改每个ThreadCache的大小限制到512KB~4MB之间的相应值。

###### 7.6.5 慢启动算法：FreeList的长度控制

类似TCP的拥塞控制算法，TCMalloc采用了慢启动（slow start）的方式来控制FreeList的长度，其效果如下：

- FreeList被使用的越频繁，最大长度就越大。
- 如果FreeList更多的用于释放而不是分配，则其最大长度将仅会增长到某一个点，以有效的将整个空闲对象链表一次性移动到CentralCache中。

分配内存时的慢启动代码如下（`FetchFromCentralCache`）：

```c++
//Thread_cache.cc
   const int batch_size = Static::sizemap()->num_objects_to_move(cl);
   ReleaseToCentralCache(list, cl, batch_size);

  // If the list is too long, we need to transfer some number of
  // objects to the central cache.  Ideally, we would transfer
  // num_objects_to_move, so the code below tries to make max_length
  // converge on num_objects_to_move.

   if (list->max_length() < batch_size) 
   {
    // Slow start the max_length so we don't overreserve.
    list->set_max_length(list->max_length() + 1);
   } 
   else if (list->max_length() > batch_size) 
   {
    // If we consistently go over max_length, shrink max_length.  If we don't
    // shrink it, some amount of memory will always stay in this freelist.
    list->set_length_overages(list->length_overages() + 1);
    if (list->length_overages() > kMaxOverages) 
    {
      ASSERT(list->max_length() > batch_size);
      list->set_max_length(list->max_length() - batch_size);
      list->set_length_overages(0);
    }
```

max_length即为FreeList的最大长度

可以看到，只要max_length没有超过batch_size，每当FreeList中没有元素需要从CentralCache获取空闲对象时（即`FetchFromCentralCache`），max_length就加1。

一旦max_length达到batch_size，接下来每次`FetchFromCentralCache`就会导致max_length增加batch_size。

再来看内存回收时的情况，每次释放小对象，都会检查FreeList的当前长度是否超过max_length：

```c++
if (PREDICT_FALSE(length > list->max_length())) 
{
  ListTooLong(list, cl);
  return;
}
```

如果超长，则执行以下逻辑：

```c++
void ThreadCache::ListTooLong(FreeList* list, uint32 cl) 
{
  size_ += list->object_size();
  const int batch_size = Static::sizemap()->num_objects_to_move(cl);
  ReleaseToCentralCache(list, cl, batch_size);
  // If the list is too long, we need to transfer some number of
  // objects to the central cache.  Ideally, we would transfer
  // num_objects_to_move, so the code below tries to make max_length
  // converge on num_objects_to_move.
  if (list->max_length() < batch_size) 
  {
    // Slow start the max_length so we don't overreserve.
    list->set_max_length(list->max_length() + 1);
  } else if (list->max_length() > batch_size) {
    // If we consistently go over max_length, shrink max_length.  If we don't
    // shrink it, some amount of memory will always stay in this freelist.
    list->set_length_overages(list->length_overages() + 1);
    if (list->length_overages() > kMaxOverages) {
      ASSERT(list->max_length() > batch_size);
      list->set_max_length(list->max_length() - batch_size);
      list->set_length_overages(0);
    }
  }
  if (PREDICT_FALSE(size_ > max_size_)) {
    Scavenge();
  }
}
```

与内存分配的情况类似，只要max_length还没有达到batch_size，每当FreeList的长度超过max_length，max_length的值就加1。

当max_length达到或超过batch_size后，并不会立即调整max_length，而是累计超过3次（`kMaxOverages`）后，才会将max_length减少batch_size。

##### 7.7 CentralCache

CentralCache是逻辑上的概念，其本质是CentralFreeListPadded类型（CentralFreeList的子类，用于64字节对齐）的数组，每个size class对应数组中的一个元素。

```c++
ATTRIBUTE_HIDDEN static CentralFreeListPadded central_cache_[kClassSizesMax];
```

CentralFreeList中缓存了一系列小对象，供各线程的ThreadCache取用，各线程也会将多余的空闲小对象还给CentralFreeList，另外CentralFreeList还负责从PageHeap申请span以分割成小对象，以及将不再使用的span还给PageHeap。

###### 7.7.1 管理span

CentralFreeList真正管理的是span，而小对象是包含在span中的空闲对象链表中的。CentralFreeList的`empty_`链表保存了已经没有空闲对象可用的span，`nonempty_`链表保存了还有空闲对象可用的span：

<img src="https://wallenwang.com/wp-content/uploads/tcmalloc-CentralFreeList.svg" alt="img" style="zoom:50%;" />

###### 7.7.2 CentralFreeList↔ PageHeap

- **从PageHeap获取span**

当ThreadCache从CentralFreeList取用空闲对象（`RemoveRange`），但CentralFreeList的空闲对象数量不够时，CentralFreeList调用`Populate()`从PageHeap申请一个span拆分成若干小对象，首首连接记录在span的`objects`指针中，即每个小对象的起始位置处，记录了下一个小对象的地址。此时的span如下图：

![img](https://wallenwang.com/wp-content/uploads/tcmalloc-FreshSpan.svg)

可以看到，此时span包含的对象按顺序连接在一起。

新申请的span被放入CentralFreeList的`nonempty_`链表头部。

- **将span还给PageHeap**

CentralFreeList维护span的成员变量`refcount`，用来记录ThreadCache从中获取了多少对象。

当ThreadCache将不再使用的对象归还给CentralCache以致refcount减为0，即span中所有对象都空闲时，则CentralCache将这个span还给PageHeap。截取`CentralFreeList::ReleaseToSpans()`部分代码如下：

```C++
span->refcount--;
if (span->refcount == 0) {
  Event(span, '#', 0);
  counter_ -= ((span->length<<kPageShift) /
               Static::sizemap()->ByteSizeForClass(span->sizeclass));
  tcmalloc::DLL_Remove(span);
  --num_spans_;
  // Release central list lock while operating on pageheap
  lock_.Unlock();
  {
    SpinLockHolder h(Static::pageheap_lock());
    Static::pageheap()->Delete(span);
  }
  lock_.Lock();
}
```

###### 7.7.3 PageHeap↔CentralFreeList 

CentralFreeList和ThreadCache之间的对象移动是批量进行的：

```C++
void InsertRange(void *start, void *end, int N);
// Returns the actual number of fetched elements and sets *start and *end.
int RemoveRange(void **start, void **end, int N);
```

start和end指定小对象链表的范围，N指定小对象的数量。批量移动小对象可以**均摊锁操作的开销**。

- **ThreadCache取用小对象**

当ThreadCache中某个size class没有空闲对象可用时，需要从CentralFreeList获取N个对象，那么N的值是多少呢？从`ThreadCache::FetchFromCentralCache()`中可以找到答案：

```c++
const int batch_size = Static::sizemap()->num_objects_to_move(cl);
const int num_to_move = min<int>(list->max_length(), batch_size);
void *start, *end;
int fetch_count = Static::central_cache()[cl].RemoveRange(&start, &end, num_to_move);
```

移动数量N为max_length和batch_size的最小值（两者的具体涵义参见ThreadCache慢启动一节）。

假设只考虑内存分配的情况，一开始移动1个，然后是2个、3个，以此类推，同时max_length每次也加1，直到达到batch_size后，每次移动batch_size个对象。

CentralFreeList和ThreadCache之间的对象移动有个优化措施，因为大部分情况都是每次移动batch_size个对象，为了减少链表操作，提升效率，CentralFreeList将移动的batch_size个对象的链表的首尾指针缓存在了TCEntry中。因此后续只要需要移动batch_size个对象，只需要操作链表的首尾即可。

```c++
TCEntry tc_slots_[kMaxNumTransferEntries];
```

- **ThreadCache归还小对象**

当ThreadCache中的空闲对象过多时（`ThreadCache::ListTooLong()`），会将一部分空闲对象放回CentralFreeList（`ThreadCache::ReleaseToCentralCache()`）。如何判断空闲对象过多请参考ThreadCache慢启动一节。

线程销毁也会将其ThreadCache中所有的空闲对象都放回CentralFreeList。

如果ThreadCache缓存的内存大小超过其允许的最大值，会触发GC操作（`ThreadCache::Scavenge()`），在其中也会将部分小对象归还给CentralFreeList，具体请参考ThreadCache垃圾回收一节。

#### 8、性能

​	`tcmalloc`要比`ptmalloc2`更具有一致的伸缩性

​	`tcmalloc`在绝大多数情况下要比`ptmalloc2`快，特别是小分配上。

​	`tcmalloc`的性能随着分配尺寸的增加而降低。这是因为当线程缓存达到了阈值(默认`2MB`)的时候会被垃圾手机。对于更大的分配尺寸，在垃圾收集之前只能在缓存中存储更少的对象。

#### 9、修改运行行为

​	可以通过环境变量来控制`tcmalloc`的行为，以下是常见的有用标志。

|                  标志                   |   默认值   |            作用            |
| :-------------------------------------: | :--------: | :------------------------: |
|       `TCMALLOC_SAMPLE_PARAMETER`       |     0      |        采样时间间隔        |
|         `TCMALLOC_RELEASE_RATE`         |    1.0     |    释放未使用内存的概率    |
| `TCMALLOC_LARGE_ALLOC_REPORT_THRESHOLD` | 1073741824 |      内存最大分配阈值      |
| `TCMALLOC_MAX_TOTAL_THREAD_CACHE_BYTES` |  16777216  | 分配给线程缓冲最大内存上限 |

​	微调参数：

| `TCMALLOC_SKIP_MMAP`              | default: false    | If true, do not try to use `mmap` to obtain memory from the kernel. |
| --------------------------------- | ----------------- | ------------------------------------------------------------ |
| `TCMALLOC_SKIP_SBRK`              | default: false    | If true, do not try to use `sbrk` to obtain memory from the kernel. |
| `TCMALLOC_DEVMEM_START`           | default: 0        | Physical memory starting location in MB for `/dev/mem` allocation. Setting this to 0 disables`/dev/mem` allocation. |
| `TCMALLOC_DEVMEM_LIMIT`           | default: 0        | Physical memory limit location in MB for `/dev/mem` allocation. Setting this to 0 means no limit. |
| `TCMALLOC_DEVMEM_DEVICE`          | default: /dev/mem | Device to use for allocating unmanaged memory.               |
| `TCMALLOC_MEMFS_MALLOC_PATH`      | default: ""       | If set, specify a path where hugetlbfs or tmpfs is mounted. This may allow for speedier allocations. |
| `TCMALLOC_MEMFS_LIMIT_MB`         | default: 0        | Limit total memfs allocation size to specified number of MB. 0 means "no limit". |
| `TCMALLOC_MEMFS_ABORT_ON_FAIL`    | default: false    | If true, abort() whenever memfs_malloc fails to satisfy an allocation. |
| `TCMALLOC_MEMFS_IGNORE_MMAP_FAIL` | default: false    | If true, ignore failures from mmap.                          |
| `TCMALLOC_MEMFS_MAP_PRVIATE`      | default: false    | If true, use MAP_PRIVATE when mapping via memfs, not MAP_SHARED. |

#### 10、在代码中修改行为

​	在`malloc_extension.h`中的`MallocExtension`类提供了一些微调的接口来修改`tcmalloc`的行为来使得你的程序达到更高的效率

##### 	（1）归还内存给操作系统

​		默认情况下，`tcmalloc`将逐渐的释放长时间未使用的内存给内核。`tcmalloc_release_rate`标志控制归还给操作系统内存的速度，你也可以强制释放内存通过执行如下操作：

​		`MallocExtension::instance() -> ReleaseFreeMemory();`

​		你同样可以调用`SetMemoryReleaseRate()`来运行时修改`tcmalloc_release_rate`的值，或者调用`GetMemoryRate()`来查看当前释放的概率值。

##### 	（2）内存诊断

​		有几种操作可以获取可读的当前内存的使用情况： 

​			`MallocExtension::instance() -> GetStats(buffer, buffer_length);`

​			`MallocExtension::instance() -> GetHeapSample(&string);`

​			`MallocExtension::instance() -> GetHeapGrowthStacks(&string);`

​		后面两个方法创建如同`heap-profiler`一样的文件格式，可以直接传递给`pprof`。第一个方法主要用于调试

##### 	（3）一般的tcmalloc的状态

​		`tcmalloc`支持设置和获取状态属性

​			`MallocExtension::instance() -> SetNumericProperty(property_name, value);`

​			`MallocExtension::instance() -> GetNumericProperty(property_name, &value);`

