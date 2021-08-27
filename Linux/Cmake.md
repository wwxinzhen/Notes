##### 1、

![img](D:\\wuxinzhen1\\Documents\\JD\\office_dongdong\\wuxinzhen1\\Temp\\JdOnline20210819102803.png)

```cmake
cmake_minimum_required(VERSION 2.8.9)						//用于指定cmake最低版本

project (hello)												//指定项目名称

include_directories(include)								//包含头目录文件

aux_sourcee_directory(. SRC_LIST)							//把当前目录下的源文件列表存放到变量SRC_LIST里

set(SRC_LIST 
	./mainapp.cpp 
	./Student.cpp)											/*使用set(SOURCES ...) 或GLOB(or GLOB_SOURCES)设置															   源文件SOURCES*/
file(GLOB SRC_LIST "src/*.cpp")

add_executable(testStudent ${SRC_LIST})						/*指定编译一个可执行文件，hello是第一个参数，表示生成可执															  行文件的文件名，第二个从参数用于指定源文件*/

```

##### 2、

![image-20210819102834769](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210819102834769.png)

```cmake
cmake_minimum_required (VERSION 2.8)
project (demo)
include_directories (test_func test_func1)					//向工程添加多个指定头文件的搜索路径，路径之间用空格分隔

aux_source_directory (test_func SRC_LIST)
aux_source_directory (test_func1 SRC_LIST1)

add_executable (main main.c ${SRC_LIST} ${SRC_LIST1})
```

##### 3、

![image-20210819103651544](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210819103651544.png)

最外层`CMakeLists.txt`

```
cmake_minimum_required (VERSION 2.8)

project (demo)

add_subdirectory (src)									//可以向当前工程添加存放源文件的子目录
```

这里指定`src`目录下存放了源文件，当执行`cmake`时，就会进入`src`目录下去找`src`目录下的`CMakeLists.txt`，所以在`src`目录下也建立一个`CMakeLists.txt`

```cmake
aux_source_directory (. SRC_LIST)

include_directories (../include)

add_executable (main ${SRC_LIST})

set (EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

#EXECUTABLE_OUTPUT_PATH ：目标二进制可执行文件的存放位置
#PROJECT_SOURCE_DIR：工程的根目录
#这里set的意思是把存放elf文件的位置设置为工程根目录下的bin目录
```

##### 4、动态库和静态库的编译控制

![image-20210819104349671](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210819104349671.png)

```cmake
cmake_minimum_required (VERSION 3.5)
project (demo)
set (SRC_LIST ${PROJECT_SOURCE_DIR}/testFunc/testFunc.c)

add_library (testFunc_shared SHARED ${SRC_LIST})
add_library (testFunc_static STATIC ${SRC_LIST})

set_target_properties (testFunc_shared PROPERTIES OUTPUT_NAME "testFunc")
set_target_properties (testFunc_static PROPERITES OUTPUT_NAME "testFunc")

set (LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

#add_lirbary:生成动态库或静态库（第一个参数指定库的名字，第二个参数决定动态还是静态，默认为静态；第三个参数指定生成库的源文件
#set_target_properties:设置最终生成的库的名称，还有其他功能，如设置库的版本号等
#LIBRARY_OUTPUT_PATH:库文件的默认输出路径
```

##### 5、对库进行链接

![image-20210819105049186](C:\Users\wuxinzhen1\AppData\Roaming\Typora\typora-user-images\image-20210819105049186.png)

```cmake
cmake_minimum_required (VERSION 3.5)

project (demo)

set (EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

set (SRC_LIST ${PROJECT_SOURCE_DIR}/src/main.c}

include_directories (${PROJECT_SOURCE_DIR}/testFunc/inc)

find_library(TESTFUNC_LIB testFunc HINTS ${PROJECT_SOURCE_DIR}/testFunc/lib)

add_executable (main ${SRC_LIST})

target_link_libraries (main ${TESTFUNC_LIB})

#find_library:在指定目录下查找库，并把库的绝对路径存放在变量里，其第1个参数是变量名称，第2个参数是库名称，第3个参数是HINTS，第4个参数是路径 默认是查找动态库，如想指定类型，可以写成find_library(TESTFUNC_LIB libtestFunc.so ...)
#target_link_libraries:把目标文件与库文件进行链接
```

##### 6、添加编译选项

```cmake
cmake_minimum_required (VERSION 2.8)

project (demo)

set (EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

add_compile_options (-std=c++11 -Wall)

add_exectable (main main.cpp)

#add_compile_options 添加编译选项
```

##### 7、include

​	为了避免重复编写，可以把一些语句写在`.cmake`文件中，在文件中进行include操作就行

```cmake
include <(file | module)> [OPTIONAL] [RESULT_VARIBLE <bar>]
						  [NO_POLICY_SCOPE]
一般情况下，直接写
include(file | module)
为了使CMakeList.txt能够找到该文件，需要指定文件完整路径（绝对或者相对），如果制定了CMAKE_MODULE_PATH,就可以直接include该目录下的.cmake文件了
```

##### 8、MACRO & function

```cmake
macro(<name> [arg1 [arg2 [arg3 ...]]])
  COMMAND1(ARGS ...)            # 命令语句
  COMMAND2(ARGS ...)
  ...
endmacro()

function(<name> [arg1 [arg2 [arg3 ...]]])
  COMMAND1(ARGS ...)            # 命令语句
  COMMAND2(ARGS ...)
  ...
function()

#当宏和函数调用的时候，如果传递的是经set设置的变量，必须通过${}取出内容
#在宏的定义中，对变量的操作必须通过${}取出内容
```

