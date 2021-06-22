##### x重启网卡

```shell
systemctl restart network
```

##### 防火墙

```shell
查看状态：systemctl status firewalld.service
关闭：systemctl stop firewalld
开启：systemctl start firewalld
```

##### 比较文本差异

```shell
diff ttpapi.h ../delete/ttpapi.h
```

##### 解压文件

```shell
rar：
	rar x demo.rar   	 	//解压demo.rar到当前目录
	rar demo.rar ./demo  	//将demo目录打包为demo.rar
unzip:
	unzip demo.zip		 	//解压demo.rar到当前目录
	zip demo.zip ./demo		//将demo目录打包为demo.rar
.tar.gz
	tar -zxf demo.tar.gz -C //解答demo.tar.gz到当前目录
```

##### 文件与目录操作

```shell
cp file1 file2  			//将file1复制为file2
cp -a dir1 dir2 			//复制一个目录
cp -a /tmp/dir1 .			//复制一个目录到当前目录（.代表当前目录）in
pwd							//显示工作路径
mkdir -p /tmp/dir1/dir2		//创建一个目录树
mv dir1 dir2				//移动/重命名一个目录
```

##### 文件内容查看

```shell
head -2 file1				//查看文件前两行
more file1					//查看一个长文件内容
tac file1					//从最后一行反向查看文件
tail -3 file1				//查看文件最后三行
```

##### 文件内容处理

```shell
grep str /tmp/test			//在文件/tmp/test中查找"str"
grep ^str /tmp/test			//在文件/tmp/test中查找"str"开始的行
```

##### 查询操作

```shell
find / -name file1			//从'/'开始进入根文件系统查找文件和目录
find / -user user1			//查找属于用户'user1'的文件和目录
find /home/user1 -name *.bin//在目录'/home/usr1'中查找'.bin'结尾的文件
```

##### RPM命令

```shell
rpm -ivh package 			//直接安装
rpmrpm --force -ivh package //忽略报错，强制安装
rpm -ql						//查询出所有安装过的包
rpm -q package				//获得某个软件包全名
rpm -ql package				//获得rpm包中文件安装的位置
rpm -e package				//卸载
```

##### YUM命令

```shell
yum -y install package		//下载并安装一个rpm包
yum localinstall package	//安装一个rpm包，使用自己的软件仓库解决依赖关系
yum -y update				//更新当前系统中安装的rpm包
yum update package			//更新一个rpm包
yum remove package			//卸载一个rpm包
yum list					//列出当前系统中安装的所有包
yum search package			//在rpm仓库中搜索软件包
yum clean all 				//删除所有缓存的包和头文件
```

##### 网络相关

```
ifconfig eth0				//显示一个以太网卡的配置
ifdown eth0					//禁用"eth0"网络设备
ifup eth0					//启用"eth0"网络设备
iwconfig eth1				//显示一个无线网卡的配置
iwlist scan					//显示无线网络
ip addr show				//显示网卡的Ip
```

##### 窗体快捷键

```shell
Ctrl + u					//删除光标之前到行首的字符
Ctrl + k					//删除光标位置到行尾的字符
Ctrl + a					//光标移动到行首
Ctrl + e					//光标移动到行尾
Ctrl + l					//清屏
Ctrl + w					//删除从光标位置前到当前所处单词的开头
```

##### 其他

```shell
ldd insert（文件名）			 //查看文件链接
```

