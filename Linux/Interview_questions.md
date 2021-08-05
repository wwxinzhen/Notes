#### 一、Linux 多线程

1、同一进程中的多条线程将共享该进程中的全部系统资源，如虚拟地址空间，文件描述符和信号处理等。但同一进程中的多个线程有各自的调用栈、自己的寄存器环境，自己的线程本地存储。

2、基本`API`

```c++
创建线程 pthread_create
    用于创建一个线程，成功返回0，否则返回Exxx（为正数）。
	原型：int pthread_create(pthread_t *tid, const pthread_attr_t *attr, void *(*func) (void *), void *arg)
		pthread_t *tid:线程id的类型为pthread_t，通常为无符号整型，当调用pthread_create成功时，通过*tid指针返回。
    	const pthread_attr_t *attr：指定创建线程的属性，如线程优先级、初始栈大小、是否为守护进程等。可以使用NULL来使用默认值，通常情况下我们都是使用默认值。
    	void *(*func) (void *)：函数指针func，指定当新的线程创建之后，将执行的函数。
    	void *arg：线程将执行的函数的参数。如果想传递多个参数，请将它们封装在一个结构体中。
================================================================================
线程等待 pthread_join
    用于等待某个线程退出，成功返回0，否则返回Exxx（为正数）。    
    原型：int pthread_join (pthread_t tid, void ** status);
		pthread_t tid：指定要等待的线程ID
    	void ** status：如果不为NULL，那么线程的返回值存储在status指向的空间中（这就是为什么status是二级指针的原因！这种才参数也称为“值-结果”参数）。
================================================================================
结束线程 pthread_exit
    用于终止线程，可以指定返回值，以便其他线程通过pthread_join函数获取该线程的返回值。
    原型：void pthread_exit (void *status);
		void *status：指针线程终止的返回值。
================================================================================
pthread_self 用于返回当前线程的ID。            
pthread_detach用于是指定线程变为分离状态，就像进程脱离终端而变为后台进程类似。成功返回0，否则返回Exxx（为正数）。变为分离状态的线程，如果线程退出，它的所有资源将全部释放。而如果不是分离状态，线程必须保留它的线程ID，退出状态直到其它线程对它调用了pthread_join。
```

3、`mutex` & `cond`

```c++
#include <pthread.h> 
=================================================================================
int pthread_mutex_lock(pthread_mutex_t * mptr); 
int pthread_mutex_unlock(pthread_mutex_t * mptr);
在对临界资源进行操作之前需要pthread_mutex_lock先加锁，操作完之后pthread_mutex_unlock再解锁。而且在这之前需要声明一个pthread_mutex_t类型的变量，用作前面两个函数的参数。
    
int pthread_cond_wait(pthread_cond_t *cptr, pthread_mutex_t *mptr); 
int pthread_cond_signal(pthread_cond_t *cptr);
pthread_cond_wait用于等待某个特定的条件为真，pthread_cond_signal用于通知阻塞的线程某个特定的条件为真了。在调用者两个函数之前需要声明一个pthread_cond_t类型的变量，用于这两个函数的参数。

```

4、`semaphore（信号量）`

```c++
#include <semaphore.h>
=================================================================================
初始化信号量 int sem_init(sem_t *sem, int pshared, unsigned int value);
	成功返回0，失败返回-1
    参数
        sem:指向信号量结构的一个指针
        pshared:不是0的时候，该信号量在进程间共享，否则只能为当前进程的所有线程们共享
        value:信号量的初始值
```

#### 二、mysql主从基本原理

1、mysql主从区别

​	（1）主从分工：master负责写操作，slave负责读操作。读写比例大概在10:1，所以有多个slave。

​	（2）用途

​			实在灾备，用于故障切换

​			读写分离，提供查询服务

​			备份，避免影响业务

​	（3）条件

​			master设置log-bin参数，开启binlog日志

​			主从server-id不同

​			slave能联通master

2、主从同步的粒度、原理和形式

​	（1）三种主要实现粒度：

​			statement：会将对数据操作的sql语句写到binlog中

​			row：会将每一条数据的变化写到binlog中

​			mixed：statement与row的混合。mysql决定何时写statement格式的binlog，何时写row格式的binlog。

​	（2）实现原理

​			master：当master上的数据发生变化时，该事件变化会按照顺序写入binlog中。当slave连接到master时，master机器会为slave开启binlog dump线程。当master的binlog发生变化的时候，binlog dump线程会通知slave，并将相应的binlog内容发送给slave。

​			slave：当主从同步开启时，slave会创建两个线程：I/O线程，sql线程。对于I/O线程，该线程会连接到master，master上的binlog dump线程会将binlog的内容发送给I/O线程。该I/O线程接收到binlog内容后，再将内容写入到本地的relay log；对于sql线程，该线程读取I/O线程写入的relay log，并且根据relay log的内容，做出对slave数据库相应的操作。

![image-20210705140407550](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210705140407550.png)

