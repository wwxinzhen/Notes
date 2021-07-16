##### 一、mysql主从基本原理

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

