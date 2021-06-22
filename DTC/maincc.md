##### 1、int main(int argc, char *argv[]) && int main(int argc, char **argv)

​	argc：int类型的参数，表示给main函数传递了多少个参数

​	atgv：一个字符数组（或者二重指针），用来存放字符串。

​	argv表示传入main函数的参数列表或指针，并且第一个参数argv[0]一定是程序的名称，并且包含了程序所在的完整路径，所以确切的说，需要输入的main函数的参数各位应该是argc-1个。

​	main函数的参数由由main函数所在的父进程传递，并且由这个父进程接受main函数的返回值。

​	程序调用本质上都会父进程fork一个子进程，然后子进程和一个程序绑定起来去执行(exec函数族)，我们在exec的时候可以给他同事传参。程序调用时可以被传参(也就是main的传参)是操作系统层面的支持完成的。

##### 2、environ

​	environ变量是一个char**类型的变量，存放着系统的全局变量。

​	environ是一个外部的全局变量，在使用时需要用extern声明一下。

https://blog.csdn.net/m0_38121874/article/details/80891428?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control

##### 3、getopt

​	原型：

​		int getopt(int argc, char * const argv[], const char * optstring)

​	getopt()用来分析命令行参数。参数optstring则代表预处理的选项字符串。

##### 4、socketpair

​	原型：

​		int socketpair(int d, int type, int protocol, int sv[2])

​	用于创建一对无名的、相互连接的套接字。成功返回0，创建好的套接字分别为sv[0]和sv[1]；失败返回-1

##### 5、snprintf

​	原型：

​		init snprintf(char *str, size_t size, const char *format,...)

​	将可变参数...按照format的格式格式化为字符串，然后再将其拷贝至str中

​	如果格式化后的字符串长度<size,则将次字符串全部复制到str中，并给其后添加一个字符串结束符'\0'

​	如果格式化后的字符串长度>=size，则只将其中的size-1个字符复制到str中，并添加一个'\0'

​	返回值为欲写入的字符串长度。

##### 6、setenv

​	原型：

​		int setenv(const char * name, const char * value, int overwrite)

​	setenv()用来改变或增加环境变量的内容。参数name为环境变量名称字符串。参数value为变量内容，参数overwrite用来改变是否要改变已存在的环境变量。如果没有此环境变量则无论overwrite为何值均添加此环境变量。若环境变量存在，当overwrite不为0时，原内容会被改为参数value所指的变量内容，当overwrite为0时，则参数value会被忽略。

​	返回值：成功返回0，失败返回-1。