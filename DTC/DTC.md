##### 1、DTC简介

​	Distributed Table Cache,DTC是一种DB代理，并能够提供热点数据高速访问服务的通用Cache Server,可以提高数据访问效率减少对后端DB压力。

##### 2、DTC应用

​	研发方面：提供高可用、易接入的缓存接入服务，缩短开发周期。

​	运营方面：提供开发、运维合理使用缓存的指导。

##### 3、Redis & DTC

​	Redis：

​		支持结构化数据有限，本质上是key-value（string、hash、list、set、zset）

​		集群支持不够完善

​		相同硬件条件下，相同包大小，read吞吐110000次/s

​		相同硬件条件下，相同包大小，时延<100us

​	DTC：

​		支持结构化数据，提供相对复杂的结构化查询

​		集群支持专业化，自主开发

​		相同硬件条件下，相同包大小，read吞吐160000次/s

​		相同硬件条件下，相同包大小，时延<300us

​		专业自主管理平台，申请简单，自动部署

​		完善的监控系统支持

​		提供专业化的数据分析引擎，帮助业务开发人员做决策

##### 4、SOA服务化--业务指标

​	DTC响应耗时

​	命中率

​	请求响应数

​	请求流量

​	客户端响应耗时

##### 5、SOA服务化--分析引擎

​	存储数据大小区间分布

​	内存数据行分布

​	最近访问时间分布

​	生存时间分布

​	操作类型分布

​	脏数据比率

​	脏数据时间分布

##### 6、DTC适用场景

​	大数据量存储

​	读请求远大于写请求

​	适合简单统计场景

​	单表操作，不支持多表连接

##### 7、DTC有源逻辑架构

![image-20210528140601952](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210528140601952.png)

##### 8、DTC架构原理

​	处理路径的抽象允许随时attach新的处理单元，拓展程序功能

​	Cache系统和数据访问helper分离，使得系统支持多种数据源

​	datasource分布允许后端数据离散分布

##### 9、内存管理

​	设计思路：

​		不固化存储结构，允许内存块大小可变，位置可移动

​		不固定索引节点属性，随时允许动态增加

​	特性：

​		Hash Bucket

​		Node Index

​		Node Group

​		Virtual Node

​		LRU list

​	特性抽象：

​		众多的实现特性如何去管理？

​		Feature-descriptot对外提供统一接口

​	属性聚合：

![image-20210528141443036](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210528141443036.png)

​		属性聚合使得紧密存储成为可能，能大幅提高内存利用率

​		属性聚合方便动态增加新属性

​	多级索引：

![image-20210528141809656](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210528141809656.png)

​		变成分配器：

​			摒弃老的定长数据chunk概念，不再采用定长存储结构，转而采用变长分配机制

​			变长分配采用类似ptmalloc的bins分配策略，使得内存分配、释放非常高效。

##### 10、模块介绍和规划

![image-20210528143932145](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210528143932145.png)