##### 1、主要类说明

| 类名      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Filter    | 过滤器，过滤输出消息                                         |
| Layout    | 布局器，控制输出消息的格式                                   |
| Appender  | 挂载器，与布局器和过滤器紧密配合，将特定格式消息过滤后输出到所挂载的设备终端（屏幕、文件等） |
| Logger    | 记录器，保存并跟踪对象日志信息变更的实体，当你需要一个对象进行记录时，就需要生成一个logger |
| Hierarchy | 分类器，层次化的树形结构，用于对被记录信息的分类，层次中每个节点维护一个logger的所有信息 |
| LogLevel  | 优先权，包括TRACE、DEBUG、INFO、WARNING、ERROR、FATAL        |

##### 2、基本步骤

```c++
/*
六个步骤：
	实例化一个封装了输出介质的appender对象；
	实例化一个封装了输出格式的layout对象；
	将layout对象绑定(attach)到appender对象；如省略此步骤，简单布局器SimpleLayout(参见5.1小节)对象会绑定到logger。
	实例化一个封装了日志输出logger对象,并调用其静态函数getInstance()获得实 
		例，log4cplus::Logger::getInstance("logger_name")；
	将appender对象绑定(attach)到logger对象；
	设置logger的优先级，如省略此步骤，各种有限级的日志都将被输出。
*/
int main()
{
    /* step 1: Instantiate an appender object */
	SharedAppenderPtr _append (new ConsoleAppender());
	_append->setName("append for test");
　　　
    /* step 2: Instantiate a layout object */
	std::string pattern = "%d{%m/%d/%y %H:%M:%S}  - %m [%l]%n";
	std::auto_ptr<Layout> _layout(new PatternLayout(pattern));
　　　
    /* step 3: Attach the layout object to the appender */
	_append->setLayout( _layout );
　　　
    /* step 4: Instantiate a logger object */
	Logger _logger = Logger::getInstance("test");
　　　
    /* step 5: Attach the appender object to the logger  */
	_logger.addAppender(_append);
　　　
    /* step 6: Set a priority for the logger  */
	_logger.setLogLevel(ALL_LOG_LEVEL);
　　　
    /* log activity */
    LOG4CPLUS_DEBUG(_logger, "This is the FIRST log message...")
    sleep(1);
    LOG4CPLUS_WARN(_logger, "This is the SECOND log message...")
    return 0;
}
//简洁使用可以只使用1、4、5步骤
```



