##### 1、makefile

​	一个工程文件的编译规则，描述了整个工程的编译和链接等规则。其中包含了哪些文件需要编译，哪些文件不需要编译，哪些文件需要先编译，哪些文件需要后编译，哪些文件需要重建等等。

##### 2、规则

​	两部分组成：依赖的关系和执行的命令，其结构如下：

```shell
targets : prerequisites
	command
```

​		或者是

```shell
targets : prerequisites; command
	command
```

​		targets:规则的目标，可以是Object file，也可以是执行文件，也可以是一个标签；

​		prerequisites：是我们的依赖文件，要生成targets需要的文件或者是目标。可以是多个，也可以没有；

​		command：make需要执行的命令。可以多条，每一条命令占一行。

​		**！！！目标和依赖文件之间要使用冒号分隔开，命令的开始一定要使用Tab键。**

```shell
test : test.cpp
	g++ test.cpp -o test
test是目标文件，也就是最后生成的可执行文件。依赖文件是test.c源文件。重建目标文件需要执行的操作是 g++ test.cpp -o test。
```

​	makefile一共包含五部分：

​		显式规则

​		隐晦规则

​		变量的定义

​		文件的指示

​		注释：makefile只有行注释，注释符号是用“#”

##### 3、makefile工作流程

```shell
main:main.o test1.o test2.o
	g++ main.o test1.o test2.o -o main
main.o:main.c test.h
	g++ -c main.c -o main.o
test1.o:test1.c test.h
	g++ -c test1.c -o test1.o
test2.o:test2.c test.h
	g++ -c test2.c -o test2.o
```

​	当在shell提示符下输入make命令后，make读取当前目录下的Makefile文件，并将Makefile文件中的第一目标作为其执行的“终极目标”，开始处理第一个规则。在上面的例子中，第一个规则就是目标“main”所在的规则。规则描述了“main"的依赖关系，并定义了链接”.o"文件生成目标"main"命令，make在执行这个规则所定义的命令之前，首先处理目标"main"的所有依赖文件

​	清除工作目录中的过程文件，在结尾加上：

```shell
.PHONY:clean
clean:
	rm -rf *.o test
```

##### 4、变量

​	Makefile文件中定义变量的基本语法：

```
	变量的名称=值列表
```

​	变量的名称可以由**大小写字母、阿拉伯数字和下划线**构成。值列表，可以是零项，一项或者多项。

```
VALUE_LIST = one two three
```

调用变量的时候可以用"$(VALUE_LIST)"或者是"${VALUE_LIST}"来替换

```shell
OBJ = test.cpp
test : $(OBJ)
	g++ $(OBJ) -o test 
```

##### 5、变量的赋值

​	简单赋值（:=） ：编程语言中常规理解的赋值方式，只对当前语句的变量有效

```shell
x:=foo
y:=$(x)b
x:=new
test:
	@echo "y=>$(y)"
	@echo "x=>$(x)"
=========================
make test
=========================
y=>foob
x=>new
```

​	递归赋值(=) 赋值语句可能影响多个变量，所有目标变量相关的其他变量都受影响

```shell
x=foo
y=$(x)b
x=new
test:
	@echo "y=>$(y)"
	@echo "x=>$(x)"
========================
make test
========================
y=>newb
x=>new
```

​	条件赋值(?=) 如果变量未定义，则使用符号中的值定义变量。如果该变量已经赋值，则该赋值语句无效。

```shell
x:=foo
y:=$(x)b
x?=new
test:
	@echo "y=>$(y)"
	@echo "x=>$(x)"
========================
make test
========================
y=>foob
x=>foo
```

​	追加赋值(+=) 原变量用空格隔开的方式追加一个新值

```shell
x:=foo
y:=$(x)b
x+=$(y)S
test:
	@echo "y=>$(y)"
	@echo "x=>$(x)"
========================
make test
========================
y=>foob
x=>foo foob
```

##### 6、自动化变量

​	Makefile自动产生的变量，自动化变量的取值根据执行的规则来决定，取决于执行规则的目标文件和依赖文件。以下为自动化变量说明：

![image-20210601140118631](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210601140118631.png)

##### 7、目标文件搜索（VPATH & vpath）

​	VPATH是变量，更具体的说是环境变量，Makefile中的一种特殊变量，使用时需要指定文件的路径；vpath是关键字，按照模式搜索，搜索的时候不仅需要加上文件的路径，还需要加上相应限制的条件。

​	VPATH的使用

```shell
VPATH := src
把src的值赋值给变量VPATH，所以在执行make的时候会从src目录下找我们需要的文件
===========================
当存在多个路径：
VPATH := src:car
多个路径之间要使用空格或者冒号隔开
```

##### ！！！无论是否定义了路径，makefile执行的时候都会先搜索当前路径下的文件，没找到文件才回去VPATH中寻找。

​	vpath的使用

```shell
1）vpath PATTERN DIRECTORIES
2）vpath PATTERN
3）vpath
#PATTERN:可以理解为要寻找的条件；DIRECTORIES:寻找的路径
=========================================================
vpath test.c src
#在src路径下搜索文件test.c，多路径书写规则如下：
vpath test.c stc car | vpath test.c src:car
=========================================================
vpath tets.c
#清除符合文件test.c的搜索目录
=========================================================
vpath
#清除所有已被设置的文件搜索路径
=========================================================
可以使用模式字符"%"，这个符号的作用是匹配一个或者多个字符，例如"%.c"，表示搜索路径下所有.c结尾的文件。
```

​	使用情况

​		使用VPATH的情况是当前路径下文件较少，或者搜索的文件不能使用通配符表示。如果存在某个路径的文件特别的多或者是可以使用通配符表示的时候，就不建议使用VPATH这种方法，速度慢，效率低。

##### 8、伪目标

​	使用伪目标的两点原因：

​		（1）避免我们的Makefile中定义的只执行命令的目标和工作目录下的实际文件出现名字冲突

```shell
clean:
	rm -rf *.o test
#规则中rm命令不是创建文件clean的命令，而是执行删除任务，删除当前目录下的所有.o结尾和文件名为test的文件。当工作目录下不存在以clean命令的文件时，在shell中输入make clean命令，rm -rf *.o test总会被执行。如果当前目录存在文件名为clean的文件时情况就会不一样。当我们在shell中执行make clean，由于这个规则没有依赖文件，所以目标被认为是最新的为不去执行规则所定义的命令。为了解决这个问题，删除clean文件或者是在Makefile中将目标clean声明为伪目标。将一个目标声明为伪目标的方式是将它作为特殊的目标.PHONY的依赖
.PHONY:clean
================================
完整格式：
.PHONY:clean
clean:
	rm -rf *.o test
```

​		（2）提高执行make时的效率。在make的并行和递归执行的过程中，此情况下一般会存在一个变量，定义为所有需要make的子目录。对多个目录make的实现，可以在一个规则的命令行中使用shell循环来完成。

```shell
SUBDIRS=foo bar baz
subdirs:
	for dir in $(SUBDIRS);do $(MAKE) -C $$dir;done
#代码表达的意思是当前目录下存在三个子文件目录，每个子目录都有相对应的Makefile文件，代码中实现的部分是用当前目录下的Makefile控制其他子模块的Makefile的运行，但是这种方法存在以下几个问题：
#当子目录执行make出现错误时，make不会退出。在最终执行失败的情况下，我们很难根据错误提示定位出具体是在哪个目录下执行make发生的错误
#使用shell循环方式时，没有用到make对目录的并行处理功能，由于规则的命令是一条完整的shell命令，不能被并行处理。
===============================
#解决方法：
SUBDIRS = foo bar baz
.PHONY : subdirs $(SUBDIRS)
subdirs:$(SUBDIRS)
$(SUBDIRS):
	$(MAKE) -C $@
foo:baz
#foo:baz，这个规则是用来规定三个子目录的编译顺序。因为在规则中"baz"的子目录被当做"foo"的依赖文件，所以baz比foo更先执行，bar最后执行。
#一般情况下，一个伪目标不作为另外一个目标的依赖。这是因为当一个目标文件的依赖包含伪目标时，每一次在执行这个规则伪目标所定义的命令都会被执行。当一个伪目标没有任何目标的依赖时，我们只能通过make的命令来明确的指定它的终极目标，执行它所在规则所定义的命令。例如make clean
```

​	伪目标实现多文件编辑

```shell
.PHONY:all
all : test1 test2 test3
test1 : test1.o
	g++ -o $@ $^
test2 : test2.o
	g++ -o $@ $^
test3 test3.o
	g++ -o $@ $^
```

##### 9、Makefile常用字符串处理函数

​	函数调用的格式如下：

```shell
$(<function> <arguments>) 或者是 ${<function> <arguments>}
#function是函数名，arguments是参数，参数之间用","隔开。参数和函数名之间用空格隔开。
```

​	1）模式字符串替换函数

```shell
$(patsubst <pattern>,<replacement>,<text>)
#函数功能是查找text中的单词是否符合模式pattern，如果匹配的话，则用replacement替换。返回值为替换后的新字符串。
=========================================
OBJ = $(patsubst %.c,%.o,1.c 2.c 3.c)
all:
	@echo $(OBJ)
=========================================
执行make：
1.o 2.o 3.o
```

​	2）字符串替换函数

```shell
$(subst <from>,<to>,<text>)
#把字符串中的from替换成to，返回值为替换后的新字符串。
=========================================
OBJ = $(subst ee,EE,feet on the street)
all:
	@echo $(OBJ)
=========================================
执行make：
fEEt on the strEEt
```

​	3）去空格函数

```shell
$(strip<string>)
#去掉字符串的开头和结尾的空格，并且将其中的多个连续的空格合并成为一个空格。
```

​	4）查找字符串函数

```shell
$(findstring<find>,<in>)
#查找in中的find，如果我们查找的目标字符串存在，返回值为目标字符串，不存在返回空。
```

​	5）过滤函数

```shell
$(filter<pattern>,<text>)
#过滤出text中符合模式pattern的字符串，可以后多个pattern。返回值为过滤后的字符串
```

​	6）反过滤字符串

```shell
$(filter-out <pattern>,<text>)
#与filter函数相反，返回值是保留的字符串
```

​	7）排序函数

```shell
$(sort<list>)
#将list中的单词升序排序。
#sort会去除重复的单词
```

​	8）取单词函数

```shell
$(word<n>;<text>)
#取出函数text中低n个单词。
```

​	9）call函数

​	可以将多行变量展开，并将传入的参数替换掉多行变量中的$(1)$(2)等。

​	10）eval函数

​	eval函数可以将call函数返回的Makefile语句，或者直接将多行变量进行Make解析，使其生效。

##### 10、常用文件名操作函数

​	1）取目录函数

```shell
$(dir <names>)
#从文件名序列names中取出目录部分，如果names中没有"/"，取出的值为"./"；返回值为目录部分，指的是最后一个反斜杠之前的部分。
===================================
OBJ = $(dir src/foo.c hacks)
all:
	@echo $(OBJ)
===================================
执行make：
	src/ ./
```

​	2）取文件函数

```shell
$(notdir <names>)
#从文件名序列中names中去除非目录的部分。非目录的部分是最后一个反斜杠之后的部分。
```

##### 11、-l -L -I(i)区别

​	-l  -->  指定连接时期望连接的库的名字

​	-L -->  指定连接库的搜索路径

​	-I  -->  指定第一个寻找头文件的目录

进行DTC(distributed table cache)项目重构。DTC是一种DB代理，并能够提供热点数据高速访问服务的通用Cache Server，可以提高数据访问效率减少对后端DB压力。在 研发方面：提供高可用、易接入的缓存接入服务，缩短开发周期；运营方面：提供开发、运维合理使用缓存的指导。 （8月下旬开源）
