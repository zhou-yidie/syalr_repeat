# sylar-repeat
从零开始重写sylar C++高性能分布式服务器框架
模仿大佬重写sylar，原项目地址：https://github.com/zhongluqiang/sylar-from-scratch

## 模块概述

### 日志模块
支持流式日志风格写日志和格式化风格写日志，支持日志格式自定义，日志级别，多日志分离等等功能。
流式日志使用：
```cpp
SYLAR_LOG_INFO(g_logger) << "this is a log";
```
格式化日志使用：
```cpp
SYLAR_LOG_FMT_INFO(g_logger, "%s", "this is a log"); 
```
日志支持自由配置日期时间，累计运行毫秒数，线程id，线程名称，协程id，日志线别，日志名称，文件名，行号。
与日志模块相关的类：
`LogEvent`: 日志事件，用于保存日志现场，比如所属日志器的名称，时期时间，线程/协程id，文件名/行号等，以及日志消息内容。
`LogFormatter`: 日志格式器，构造时可指定一个格式模板字符串，根据该模板字符串定义的模板项对一个日志事件进行格式化，提供format方法对LogEvent对象进行格式化并返回对应的字符串或流。
`LogAppender`：日志输出器，用于将日志输出到不同的目的地，比如终端和文件等。LogAppender内部包含一个LogFormatter成员，提供log方法对LogEvent对象进行输出到不同的目的地。这是一个虚类，通过继承的方式可派生出不同的Appender，目前默认提供了StdoutAppender和FileAppender两个类，用于输出到终端和文件。
`Logger`: 日志器，用于写日志，包含名称，日志级别两个属性，以及数个LogAppender成员对象，一个日志事件经过判断高于日志器自身的日志级别时即会启动Appender进行输出。日志器默认不带Appender，需要用户进行手动添加。
`LoggerManager`：日志器管理类，单例模式，包含全部的日志器集合，并且提供工厂方法，用于创建或获取日志器。LoggerManager初始化时自带一个root日志器，这为日志模块提供一个初始时可用的日志器。
本项目的日志模块基于sylar的进行了简化，同时参考了log4cpp的一些设计，在保证功能可用的情况下，简化了几个类的设计，降低了耦合度。目前来看，在LogAppender上仍然需要丰富，以下几种类型的Appender在实际项目中都非常有必要实现：
1. Rolling File Appender，循环覆盖写文件
2. 内存Rolling Appender
3. 支持Log日志按大小分片
4. 支持网络日志服务器，比如syslog


### Util与Marco模块

工具接口与工具类，功能宏定义。包括获取时间，日期时间格式转换，栈回溯，文件系统操作接口，类型转换接口，以及SYLAR_ASSERT宏。详细接口参考util.h，macro.h。

### 环境变量模块

提供管理环境变量管理功能，包括系统环境变量，自定义环境变量，命令行参数，帮助信息，exe名称与程序路径相关的信息。环境变量全部以key-value的形式进行存储，key和value都是字符串格式。提供add/get/has/del接口用于操作自定义环境变量和命令行选项与参数，提供setEnv/getEnv用于操作系统环境变量，提供addHelp/removeHelp/printHelp用于操作帮忙信息，提供getExe/getCwd用于获取程序名称及程序路径，提供getAbsolutePath/getAbsoluteWorkPath/getConfigPath用于获取路径相关的信息。

待改进：
使用getopt系列接口解析命令行选项与参数，这样可以支持解析选项合并和长选项，比如像"ps -auf"和"ps --help"。


### 配置模块

采用约定优于配置的思想。定义即可使用。支持变更通知功能。使用YAML文件做为配置内容，配置名称大小写不敏感。支持级别格式的数据类型，支持STL容器(vector,list,set,map等等),支持自定义类型的支持（需要实现序列化和反序列化方法)。

使用方式如下：

```cpp
static sylar::ConfigVar<int>::ptr g_tcp_connect_timeout = 
    sylar::Config::Lookup("tcp.connect.timeout", 5000, "tcp connect timeout");
```

定义了一个tcp连接超时参数，可以直接使用`g_tcp_connect_timeout->getValue()`获取参数的值，当配置修改重新加载，该值自动更新（并触发对应的值更新回调函数），上述配置格式如下：

```yaml
tcp:
    connect:
            timeout: 10000
```

与配置模块相关的类：

`ConfigVarBase`: 配置参数基类，定义了一个配置参数的基本属性和方法，比如名称，描述，以及toString/fromString两个纯虚函数，配置参数的类型和值由继承类实现。

`ConfigVar`: 配置参数类，这是一个模板类，有三个模板参数，一个是类型T，另外两个是FromStr和ToStr，这两个模板参数是仿函数，共中FromStr用于将字符串转类型T，ToStr用于将T转字符串。这两个仿函数通过一个类型转换模板类和不同的片特化类来实现不同类型的序列化与反序列化。ConfigVar类包含了一个T类型的配置参数值成员和一个变更回调函数数组，以及toString/fromString函数实现。toString/fromString用到了模板参数toStr/fromStr。此外，ConfigVar还提供了setValue/getValue方法用于设置/修改值，并触发回调函数数组，以及addListener/delListener方法用于添加或删除回调函数。

`Config`: ConfigVar的管理类，负责托管全部的ConfigVar对象。提供Lookup方法，根据参数名称查询配置参数。如果调用Lookup查询时同时提供了默认值和配置描述信息，那么在未找到对应的配置时，会创建一个新的配置项，这样就保证了配置模块定义即可用的特性。除此外，Config类还提供了LoadFromYaml和LoadFromConfDir两个方法，用于从YAML结点或从命令行-c选项指定的配置文件路径中加载配置。Config的全部成员变量和方法都是static类型，保证了全局只有一个实例。

### 线程模块

线程模块，封装了pthread里面的一些常用功能，Thread,Semaphore,Mutex,RWMutex,Spinlock等对象，可以方便开发中对线程日常使用。为什么不适用c++11里面的thread 本框架是使用C++11开发，不使用thread，是因为thread其实也是基于pthread实现的。并且C++11里面没有提供读写互斥量，RWMutex，Spinlock等，在高并发场景，这些对象是经常需要用到的。所以选择了自己封装pthread。

线程模块相关的类：

`Thread`：线程类，构造函数传入线程入口函数和线程名称，线程入口函数类型为void()，如果带参数，则需要用std::bind进行绑定。线程类构造之后线程即开始运行，构造函数在线程真正开始运行之后返回。

线程同步类(这部分被拆分到mutex.h)中：

`Semaphore`: 计数信号量，基于sem_t实现
`Mutex`: 互斥锁，基于pthread_mutex_t实现
`RWMutex`: 读写锁，基于pthread_rwlock_t实现
`Spinlock`: 自旋锁，基于pthread_spinlock_t实现
`CASLock`: 原子锁，基于std::atomic_flag实现

待改进：
线程取消及线程清理

### 协程模块

协程：用户态的线程，相当于线程中的线程，更轻量级。后续配置socket hook，可以把复杂的异步调用，封装成同步操作。降低业务逻辑的编写复杂度。 目前该协程是基于ucontext_t来实现的，后续将支持采用boost.context里面的fcontext_t的方式实现。

协程原语：

`resume`：恢复，使协程进入执行状态  
`yield`: 让出，协程让出执行权

yield和resume是同步的，也就是，一个协程的resume必然对应另一个协程的yield，反之亦然，并且，一条线程同一时间只能有一个协程是执行状态。

### 协程调度模块
协程调度器，管理协程的调度，内部实现为一个线程池，支持协程在多线程中切换，也可以指定协程在固定的线程中执行。是一个N-M的协程调度模型，N个线程，M个协程。重复利用每一个线程。
限制：
一个线程只能有一个协程调度器
潜在问题：  
调度器在idle情况下会疯狂占用CPU，所以，创建了几个线程，就一定要有几个类似while(1)这样的协程参与调度。
### IO协程调度模块
继承自协程调度器，封装了epoll（Linux），支持注册socket fd事件回调。只支持读写事件。IO协程调度器解决了协程调度器在idle情况下CPU占用率高的问题，当调度器idle时，调度器会阻塞在epoll_wait上，当IO事件发生或添加了新调度任务时再返回。通过一对pipe fd来实现通知调度协程有新任务。

### 定时器模块
在IO协程调度器之上再增加定时器调度功能，也就是在指定超时时间结束之后执行回调函数。定时的实现机制是idle协程的epoll_wait超时，大体思路是创建定时器时指定超时时间和回调函数，然后以当前时间加上超时时间计算出超时的绝对时间点，然后所有的定时器按这个超时时间点排序，从最早的时间点开始取出超时时间作为idle协程的epoll_wait超时时间，epoll_wait超时结束时把所有已超时的定时器收集起来，执行它们的回调函数。
sylar的定时器以gettimeofday()来获取绝对时间点并判断超时，所以依赖系统时间，如果系统进行了校时，比如NTP时间同步，那这套定时机制就失效了。sylar的解决办法是设置一个较小的超时步长，比如3秒钟，也就是epoll_wait最多3秒超时，如果最近一个定时器的超时时间是10秒以后，那epoll_wait需要超时3次才会触发。每次超时之后除了要检查有没有要触发的定时器，还顺便检查一下系统时间有没有被往回调。如果系统时间往回调了1个小时以上，那就触发全部定时器。个人感觉这个办法有些粗糙，其实只需要换个时间源就可以解决校时问题，换成clock_gettime(CLOCK_MONOTONIC_RAW)的方式获取系统的单调时间，就可以解决这个问题了。

## 当前进度

| 日期       | 进度       |
| ---------- | ---------- |
| 2024.12.14 | 项目初始化 |
| 2024.12.18 | 实现日志模块 |
| 2024.12.19 | Util与Macro模块 |
| 2024.12.21 | 环境变量模块 |
| 2024.12.22 | 配置模块 |
| 2024.1.4   | 期末考试 |
| 2025.1.6   | 线程模块 |
| 2025.1.11  | 协程模块 |
| 2025.1.17  | 协程调度模块 |
| 2025.1.22  | IO调度模块 |
