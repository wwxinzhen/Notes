#### 1、TCMalloc

​	`Thread-Caching Malloc`

#### 2、动机

​	`TCMalloc`减少了多线程程序中的锁竞争关系。对于小对象，已经基本上达到了零竞争。对于大对象，`TCMalloc`尝试使用恰当粒度和有效的自旋锁。

​	在`ptmalloc2`中，内存不会从一个空间移动到另一个空间。。例如，在一个Google的应用中，第一阶段可能会为其URL标准化的数据结构分配大约`300MB`内存。当第一阶段结束后，第二阶段将从同样的地址空间开始。如果第二个阶段被安排到了与第一阶段不同的空间内，这个阶段不会复用任何第一阶段留下的的内存，并会给地址空间添加另外一个`300MB`。类似的内存爆炸问题也可以在其他的应用中看到。

​	`TCMalloc`的另一个好处表现在小对象的空间效率，例如，分配N个8字节对象可能要使用大约`8N * 1.01`字节的空间，即多用百分之一的空间。而`ptmalloc2`中每个对象都使用了一个四字节的头，我认为并将最终的尺寸取整为8字节的倍数，最后使用了`16N`字节。

#### 3、使用

​	要使用`TCMalloc`，只要将`tcmalloc`通过`"-ltcmalloc"`链接器标志接入你的应用即可。

​	你也可以通过使用`LD_PRELOAD`在不是你自己编译的应用中使用`tcmalloc`：

​		`$ LD_PRELOAD = ''usr/lib/libtcmalloc.so"`

​	`TCMalloc`还包含了一个堆检查器以及一个堆测量器，如果你要链接不包含上述两个组件，你应该链接`libtcmalloc_minimal`

#### 4、综述

​	`tcmalloc`给每个线程分配了一个线程局部缓存。小对象的分配是直接由线程局部缓存来完成的。如果需要的话，会将对象从中央数据结构移动到线程局部缓存，同时定期的用垃圾收集器把内存从线程局部缓存迁移回中央数据结构中。

​	`tcmalloc`将尺寸小于等于`32K`的对象（小对象）和大对象区分开。大对象直接使用页级分配器（一个页是一个`4K`的对齐内存区域）从中央堆直接分配。即，一个大对象总是页对齐的并占据了整数个数的页。

​	连续的一些页面可以被分割为一系列相等大小的小对象。例如，一个连续的页面（`4K`）可以被划分为32个128字节的对象。

#### 5、小对象分配

​	每个小对象的大小都会被映射到与之接近的60个可分配的尺寸类别的一个。例如，所有大小在833和1024字节之间的小对象，都会被规整到1024字节。60个可分配的尺寸类别按照以下方式隔开：较小尺寸相差8字节，较大尺寸相差16字节，再大一点的相差32字节······。

​	一个线程缓存包含了由多个尺寸内存的对象组成的单链表，如下图所示：

​		![image](https://img-blog.csdn.net/20141012210533671)

​	当分配一个小对象时：

​		（1）我们将其大小映射到对应的尺寸

​		（2）查找当前线程的线程缓存中相应的尺寸的内存链表

​		（3）如果当前尺寸链表非空，那么就从链表中移除第一个对象，并返回它

​		当按照这种方式分配时，`tcmalloc`不需要任何锁。可以极大提高分配的速度

​	如果当前尺寸内存链表为空：

​		（1）从central heap（被所有线程共享）中取得一系列这种尺寸的对象

​		（2）将他们放入该线程的缓冲区

​		（3）返回一个新获取的对象给应用程序

​	如果central heap也为空：

​		（1）从中央页分配器分配了一系列页面

​		（2）将他们分割成该尺寸的一系列对象

​		（3）将新分配的对象放入central heap的链表上

​		（4）像前面一样，将部分对象放入线程局部的链表中

#### 6、线程缓冲的大小的确定

​	如果缓冲区太小，需要经常去central heap中分配；如果太大，又会导致大量对象闲置，浪费内存。

​	确定缓冲区大小，采用"慢开始"算法来确定每一个尺寸内存链表的最大长度。当某个链表使用频繁，就扩大长度。如果某个链表释放操作比分配操作更多，它的最大长度将被增长到整个链表可以一次性有效的移动到central heap的长度。

```c++
//“慢开始”算法伪代码
Start each freelist max_length at 1.
 
Allocation
if (freelist is empty) 
{
     fetch min(max_length, num_objects_to_move) from central list;
     if (max_length < num_objects_to_move) 
     {  // slow-start
       max_length++;
     } 
    else 
    {
       max_length += num_objects_to_move;
     }
}
  
Deallocation
if (length > max_length) 
{
     // Don't try to release num_objects_to_move if we don't have that many.
	release min(max_length, num_objects_to_move) objects to central list
	if (max_length < num_objects_to_move) 
    {
       // Slow-start up to num_objects_to_move.
       max_length++;
    } 
	else if (max_length > num_objects_to_move) 
    {
		// If we consistently go over max_length, shrink max_length.
		overages++;
        if (overages > kMaxOverages) 
        {
			max_length -= num_objects_to_move;
         	overages = 0;
        }
     }
}
```

#### 7、大对象的分配

​	一个大对象的尺寸`(32K)`会被中央页堆处理，被归整到一个页面尺寸`(4K)`。中央页堆是由空闲内存列表组成的数组。对于i<256而言，数组的第K个元素是一个由一些单元(由K个页面组成)组成的空闲内存链表。第256个条目则是一个包含了长度≥256个页面的空闲内存链表。

​	k个页面的一次分配通过在第k个空闲内存链表中查找来完成。如果该空闲内存链表为空，那么我们则在下一个空闲内存链表中查找，如此继续。最终，如果必要的话，我们将在最后空闲内存链表中查找。如果这个动作也失败了，我们将向系统获取内存`(sbrk(),mmap()或者/deb/mem/ 映射)`。

​	如果k个页面的分配是由连续的＞k个页面的空闲内存链表完成，剩下的连续页面将被重新插入到与之页面大小接近的空闲内存链表中去。

#### 8、跨度

​	`tcmalloc`管理的堆由一系列页面组成。一系列的连续的页面由一个"跨度(span)"对象来表示。一个跨度可以是已被分配或者是空闲的。如果是空闲的，跨度则会是一个页面堆链表中的一个条目。如果已被分配，它会或者是一个已经被传递给应用程序的大对象，或者是一个已经被分割成一系列小对象的一个页面。如果是被分割成小对象，对象的尺寸类别会被记录在跨度中。

​	由页面号索引的中央数组可以用于找到某个页面所属的跨度对象。例如，下面的跨度a占据了2个页面，跨度b占据了1个页面，跨度c占据了5个页面，跨度d占据了3个页面。

​	![image](https://img-blog.csdn.net/20141012210536609)

​	在一个32位的地址空间中，中央数组由一个2层的基数树来表示，其中根包含了32个条目，每个页包含了2<sup>15</sup>个条目(1个32位地址空间包含了2<sup>20</sup>个`4k`页面(2<sup>32</sup>/`4k`)，一层则是用2<sup>5</sup>整除2<sup>20</sup>个页面)。这就导致了中央阵列的初始内存使用需要`128KB`(2<sup>15</sup>*4字节)的空间。

​	在64位机器上，我们将使用一个3层的基数树。

#### 9、释放

​	当一个对象被释放时，我们先计算它的页面号并在中央数组中查找对应的跨度对象。该跨度会告诉我们该对象是大是小。

​	如果它是小对象的话，尺寸类别是多少，我们将其插入到当前线程的线程缓存中对应的空闲内存链表中。如果线程缓存现在超过了某个预定的大小(默认`2MB`)，我们便运行垃圾收集器将未使用的对象从线程缓存中移入中央自由列表。

​	如果该对象是大对象，跨度对象会告诉我们该对象包含的页面的范围。假设该范围是[p,q]，我们还会查找页面p-1和q+1对应的跨度对象。如果这两个相邻的跨度中有任何一个是空闲的，我们将他们和[p,q]跨度结合起来，最后跨度会被插入到页面堆中合适的空闲链表中。

#### 10、小对象的重要空闲内存链表

​	我们为每一个尺寸类别设置了一个中央空闲列表。每个中央空闲列表由两层数据结构来组成：一系列跨度和每个跨度对象的一个空闲内存的链表。

​	一个对象是通过从某个跨度对象的空闲列表中取出第一个条目来分配的。如果所有的跨度只有空链表，那么首先从中央页面堆中分配一个尺寸合适的跨度。

​	一个对象通过将其添加到它包含的跨度对象的空闲内存链表中来将还回中央空闲列表。如果链表长度现在等于跨度对象中所有小对象的数量，那么该跨度就是完全自由，就会被返回到页面堆中。跨度对象中所有的小对象都回收完了，整个跨度对象就空闲了。

#### 11、线程缓冲区的垃圾回收

​	垃圾回收对象保证线程缓冲区的大小可控制并将未使用的对象交还给中央空闲列表。有的线程需要大量的缓冲来保证工作有很好的性能，而有的线程只需要很少甚至不需要缓冲就能工作，当一个线程的缓冲区超过它的max_size，垃圾回收对象介入，之后这个线程就要和其他线程竞争获取更大的缓冲。

​	垃圾回收仅仅会在内存释放的时候才会允许，我们检查所有的内存空闲链表并把一些数量的对象从空闲列表移动到中央链表。

​	从某个空闲链表移除的对象的数量是通过使用一个空闲链表的低水位线L来确定的。L记录了上一次垃圾收集以来最短的长度。在上一次的垃圾收集中，我们可能只是将列表缩短了L个对象而没有对中央列表进行任何额外访问。我们利用这个过去的历史作为对未来访问的预测器并将L/2个对象从线程缓存空闲了列表中移到相应的中央空闲链表中。这个算法有个很好的特性，如果某个线程不再使用某个特定的尺寸，该尺寸的所有对象都会很快从线程缓存被移到中央空闲链表，然后被其他缓存使用。

​	如果在线程中，某个大小的内存对象持续释放比分配操作多，这种L/2行为会引起至少有L/2的对象长期处于空闲链表中，为了避免这种内存浪费，我们减少每个链表的最大长度`num_objects_to_move`个。

```
Garbage Collection
if (L != 0 && max_length > num_objects_to_move) 
{
	max_length = max(max_length - num_objects_to_move, num_objects_to_move)
}
```

​	线程的缓冲区超过`size_max`的事实表明如果提高线程的缓冲区，线程将运行得更加有效率。简单的提高`max_size`的值将用掉过度的内存。对于一个有很多线程的应用程序。我们需要通过`-tcmalloc_max_total_thread_cache_bytes`标志来限制内存的用量。

​	`max_size`每个线程缓冲区从一个最小的`max_size`(例如`64K`)开始，这样空闲的线程就不会再当它们不许的时候预分配内存。每次允许垃圾回收器，如果线程的缓冲区大小小于`tcmalloc_max_total_thread_cache_bytes`，也会尝试增加`max_size`，`max_size`增长非常容易。否则，线程1将通过减少线程2的`max_size`来尝试从线程2偷取内存。采用这种方式，更加活跃的线程将从其他线程偷取内存。这样大多数空闲线程将保持很小的缓冲区而活跃的线程将保持较大的缓冲区。注意这种偷取可能引起线程缓冲区的总和比`tcmalloc_max_total_thread_cache_bytes`大，直到线程2释放内存到垃圾回收器。

#### 12、性能

​	`tcmalloc`要比`ptmalloc2`更具有一致的伸缩性

​	`tcmalloc`在绝大多数情况下要比`ptmalloc2`快，特别是小分配上。

​	`tcmalloc`的性能随着分配尺寸的增加而降低。这是因为当线程缓存达到了阈值(默认`2MB`)的时候会被垃圾手机。对于更大的分配尺寸，在垃圾收集之前只能在缓存中存储更少的对象。

#### 13、修改运行行为

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

#### 14、在代码中修改行为

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

