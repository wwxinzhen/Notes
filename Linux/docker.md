##### 1、介绍

​	Docker是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中。容器是完全使用沙箱机制，相互之间不会有任何接口。

##### 2、Docker架构

​	（1）Docker包含三个概念

​			Image：相当于一个root文件系统。

​			Container： Image和Container的关系，就像类和实例的关系。

​			Repository：可以看成一个代码控制在中心，用来保存镜像。

​	（2）Docker使用C/S架构模式，使用远程API来管理和创建Docker容器

​	（3）Docker容器通过Docker镜像来创建。

##### 3、结构图

![image-20210615151140451](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210615151140451.png)

![image-20210615151209246](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210615151209246.png)

4、命令

```shell
启动docker
	systemctl start docker
重启docker
	systemctl restart docker
关闭docker
	systemctl stop docker
查看是否启动成功
	docker ps -a
```

5、docker安装Ubuntu

```shell
拉取最新Ubuntu镜像
	docker pull ubuntu
	或者 
	docker pull ubuntu:latest
查看本地镜像
	docker images
运行容器，并且可以通过exec命令进入ubuntu容器
	docker run -itd --name ubuntu-test ubuntu
查看容器信息
	docker ps
进入容器
	docker exec -it 容器id /bin/bash
列出所有容器
	docker container ls -a
启动容器
	docker container start ID
```

