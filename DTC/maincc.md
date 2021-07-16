##### 1、int main(int argc, char *argv[]) && int main(int argc, char **argv)

```c++
argc：int类型的参数，表示给main函数传递了多少个参数
atgv：一个字符数组（或者二重指针），用来存放字符串。
argv表示传入main函数的参数列表或指针，并且第一个参数argv[0]一定是程序的名称，并且包含了程序所在的完整路径，所以确切的说，需要输入的main函数的参数各位应该是argc-1个。
main函数的参数由由main函数所在的父进程传递，并且由这个父进程接受main函数的返回值。
程序调用本质上都会父进程fork一个子进程，然后子进程和一个程序绑定起来去执行(exec函数族)，我们在exec的时候可以给他同事传参。程序调用时可以被传参(也就是main的传参)是操作系统层面的支持完成的。
```

##### 2、environ

```c++
environ变量是一个char**类型的变量，存放着系统的全局变量。
environ是一个外部的全局变量，在使用时需要用extern声明一下。
https://blog.csdn.net/m0_38121874/article/details/80891428?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control
```

##### 3、getopt

```c++
原型：
	int getopt(int argc, char * const argv[], const char * optstring)
  getopt()用来分析命令行参数。参数optstring则代表预处理的选项字符串。调用一次，返回一个选项。 在命令行选项参数再也检查不到optstring中包含的选项时，返回－1，同时optind储存第一个不包含选项的命令行参数。
 字符串optstring可以下列元素:
 (1)单个字符，表示选项
 (2)单个字符后接一个冒号：表示该选项后必须跟一个参数。参数紧跟在选项后或者以空格隔开。该参数的指针赋给optarg。
 (3)单个字符后跟两个冒号，表示该选项后必须跟一个参数。参数必须紧跟在选项后不能以空格隔开。该参数的指针赋给optarg。（这个特性是GNU的扩张）。
========================================================
optstring = "ab:c::d::"
getopt.exe -a -b host -ckeke -d haha
 -a就是选项元素，去掉'-'，a、b、c就是选项。host是b的参数，keke是c的参数，但haha不是d的参数，因为它们中间有空格隔开。
========================================================
```

##### 4、socketpair

```c++
原型：
     int socketpair(int d, int type, int protocol, int sv[2])
  用于创建一对无名的、相互连接的套接字。成功返回0，创建好的套接字分别为sv[0]和sv[1]；失败返回-1
```

##### 5、snprintf

```c++
原型：
	init snprintf(char *str, size_t size, const char *format,...)
  将可变参数...按照format的格式格式化为字符串，然后再将其拷贝至str中
  如果格式化后的字符串长度<size,则将次字符串全部复制到str中，并给其后添加一个字符串结束符'\0'
  如果格式化后的字符串长度>=size，则只将其中的size-1个字符复制到str中，并添加一个'\0'
  返回值为欲写入的字符串长度。
```

##### 6、setenv

```c++
原型：
	int setenv(const char * name, const char * value, int overwrite)
   setenv()用来改变或增加环境变量的内容。参数name为环境变量名称字符串。参数value为变量内容，参数overwrite用来改变是否要改变已存在的环境变量。如果没有此环境变量则无论overwrite为何值均添加此环境变量。若环境变量存在，当overwrite不为0时，原内容会被改为参数value所指的变量内容，当overwrite为0时，则参数value会被忽略。
   返回值：成功返回0，失败返回-1。
```

##### 7、memset

```c++
原型：
	 void *memset(void *s,  int c, size_t n)
    memset的正规用法是只能用来初始化char类型的数组的，也就是说，它只接受0x00-0xFF的赋值。
    因为char是1字节，memset是按照字节赋值的，相当于把每个字节都设为那个数，所以char型的数组可赋任意值；
    而对于也常用的int类型，int是4个字节，当memset(,1,sizeof());时，1相当于ASSCII码的1，1转为二进制00000001，当做一字节，一字节8位，int为4字节，所以初始化完每个数为00000001000000010000000100000001 = 16843009；
==================================
精巧的最大值设置：
        INF = 0X3F3F3F3F
```

##### 8、access

```c++
原型：
	int _access(const char *pathname, int mode)
    参数：pathname 为文件路径或目录路径 mode 为访问权限（在不同系统中可能用不能的宏定义重新定义）
    返回值：如果文件具有指定的访问权限，则函数返回0；如果文件不存在或者不能访问指定的权限，则返回-1.
```

##### 9、fcntl

```c++
int fcntl(int fd, int cmd)
int fcntl(int fd, int cmd, long arg)
int fcntl(int fd, int cmd, struct flock* lock)
==================================================
fcntl函数功能依据cmd的值不同而不同。
    F_DUPFD			复制文件描述符			   与dup函数功能一样
    F_GETFD			获取文件描述符标志		  读取文件描述符close-on-exec标志
    F_SETFD			设置文件描述符标志		  将文件描述符cldoe-on-exec标志设置为第三个参数arg的最后一位
    F_GETFL			获取文件状态标志		  获取文件打开方式的标志，标志值汉仪与open调用一致
    F_SETFL			设置文件状态标志		  设置文件打开方式为arg指定方式
    F_GETLK			获取文件锁									   
    F_SETLK			设置文件锁				设置或释放锁。short_l_type为F_RDLCK为读锁，F_WDLCK为写锁，F_UNLCK为解
    F_SETLKW		类似F_SETLK，但等待返回
    F_GETOWN		获取当前接受SIGIO和SIGURG信号的进程ID和进程组ID
    F_SETOWN		设置当前接受SIGIO和SIGURG信号的进程ID和进程组ID
```

##### 10、lseek

```c++
原型：
	off_t lseek(int filder, off_t offset, int whence)
    lssek()用来控制该文件的读写位置，参数filder为已打开的文件描述符，参数offset为根据参数whence来移动读写位置的位移数。
    参数whence为下列其中一种：
    	SEEK_SET 参数offset即为当前读写位置
    	SEEL_CUR 以当前的读写位置往后增加offset个位移量
    	SEEK_END 将读写位置指向文件尾后再增加offset个位移量，当whence值为SEEK_CUR或SEEK_END时，参数offset允许负值的出现。
    返回值：当调用成功时则返回目前的读写位置, 也就是距离文件开头多少个字节. 若有错误则返回-1, errno 会存放错误代码.
==================================================
methods：
lseek(int fildes, 0 , SEEK_SET)  	//将读写位置移到文件开头
lseek(int fildes, 0, ) 				//将读写位置移到文件尾
lseek(int fildes, 0, SEEK_CUR)      //想要取得目前文件位置
```

##### 11、pthread_getspecific & pthread_setspecific



##### 12、mmap

~~~c++
原型：
	void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset)
    mmap将一个文件或者其它对象映射进内存。文件被映射到多个页上，如果文件的大小不是所有页的大小之和，最后一个页不被使用的空间将会清零。mmap在用户空间映射调用系统中作用很大。
    参数说明：
    	start：映射区的开始地址，设置为0时表示由系统决定映射区的起始地址。
    	length：映射区的长度。//长度单位是 以字节为单位，不足一内存页按一内存页处理
    	prot：期望的内存保护标志，不能与文件的打开模式冲突。是以下的某个值，可以通过or运算合理地组合在一起
    		PROT_EXEC //页内容可以被执行
    		PROT_READ //页内容可以被读取
    		PROT_WRITE //页可以被写入
    		PROT_NONE //页不可访问
    	flags:
    		MAP_FIXED
            MAP_SHARED
            ``````
		fd:文件描述符
        offset：被映射对象内容的起点
    返回值：
        https://baike.baidu.com/item/mmap/1322217?fr=aladdin
~~~

##### 13、sigaction

```c++
原型：
	int sigaction(int signum,const struct sigaction *act ,struct sigaction *oldact)
    sigaction()会依参数signum指定的信号编号来设置该信号的处理函数。参数signum可以指定SIGKILL和SIGSTOP以外的所有信号。
结构sigaction定义如下:
    struct sigaction
    {
        void (*sa_handler) (int);
        sigset_t sa_mask;
        int sa_flags;
        void (*sa_restorer) (void);
    }
	sa_handler：是一个函数指针，指向一个信号处理函数。
    sa_mask：用来指定信号处理函数执行期间需要被屏蔽的信号，特别是当某个信号被处理时，它自身会自动放入进程的信号掩码，因此在信号处理函数执行期间，这个信号不会再度发生。
    sa_restorer：此参数没有使用。
    sa_flags：用来设置信号处理的其他相关操作。下列的数值可用OR 运算（|）组合：
        SA_NOCLDSTOP:使父进程在它的子进程暂停或继续运行时不会收到SIGCHLD信号。
		SA_RESTART:被信号中断的系统调用会自行重启
		SA_NODEFER:在运行信号的处理函数时再次收到当前的信号，也会触发新处理函数调用。
返回值 
	执行成功则返回0，如果有错误则返回-1。
错误代码 
	EINVAL：参数signum 不合法， 或是企图拦截SIGKILL/SIGSTOPSIGKILL信号
	EFAULT：参数act，oldact指针地址无法存取。
	EINTR：此调用被中断
```

##### 14、socketpair

```c++
原型：
	int socketpair(int domain, int type, int protocol, int sv[2])
    函数用于创建一对无名的、相互连接的套接子。
参数：
    domain：Unix本地协议族AF_UNIX
    type：可以被指定为
    	SOCK_DGRAM：无保障的面向消息的socket，主要用于在网络上发广播信息；数据包,是udp协议网络编程
    	SOCK_STREAM：相当于创建一个双向管道，每个socket都可以用来读取和写入；数据流,一般是tcp/ip协议的编程
    protocol：必须为0
    sv[2]：返回了引用这两个相互连接的socket的文件描述符
返回值：
    如果创检测成功，返回0，创建好的套接字为sv[0]、sv[1]；否则返回0
========================================================================
int main(int argc, char* argv[])
{
	char buf[128] = {0};
	int socket_pair[2];
    pid_t pid;
    if(socketpair(AF_UNIX, SOCK_STREAM, 0, socket_pair) == -1 ) 
    {
        printf("Error, socketpair create failed, errno(%d): %s\n", errno, strerror(errno));
        return EXIT_FAILURE;
    }
    int size = write(socket_pair[0], str, strlen(str));
    //可以读取成功；
    read(socket_pair[1], buf, size);
    printf("Read result: %s\n",buf);
    return EXIT_SUCCESS;
}
```

