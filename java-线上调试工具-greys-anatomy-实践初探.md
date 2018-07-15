---
title: java 线上调试工具 greys-anatomy 实践初探
date: 2017-07-22 23:12:26
tags:
---

有这样一种场景，跑在线上环境的jvm进程发生了错误，错误日志无法清晰的定位问题，或因为日志不够详尽或因为异常被吃掉没有完整的堆栈信息，一般做法是修改代码重新发布实验性的去打印更多的信息，又要重新发布程序。或者在本地试图去重现这个问题通过ide工具进行debug调试。总之这是一个比较尴尬的情况，添加过日志后还面临着两个问题：
- 日志添加的不全还要添加再次添加日志
- 查到了问题，还要去掉日志代码重新发布程序


下面介绍一个线上jvm进程调试的工具，可以说是一个神器，[greys-anatomy](https://github.com/oldmanpushcart/greys-anatomy)

细节（安装、使用、原理）可以阅读github上面的文档，本文旨在根据一个简单的场景让大家快速上手这个工具，至于更多更详细的功能，还需要去官文学习。


### 虚拟场景

周末老板要紧急上线一个系统，经过周六一天的紧急加班（写bug），终于按时提交测试。
源码提交在github，[codereview](https://github.com/HongkaiWen/web-play)

当然，没有经过各位大神review，我已经把系统发布到我的测试环境。

测试人员花了1分钟对所有功能功能进行了详细的测试，系统仅有的两个功能全部测试不通过：


bug num| 功能 | 功能入口 | 描述
---|---|---|---|---|
1 | 查看系统版本| /system/version | 页面提示 system error
2 | 校验余额是否不足 | /low/balance/check?userId=399 | 页面提示系统500错误 

自己写的bug跪着也要解决啊...

### bug 1
去线上系统重现问题，果然提示system error，看日志吧，日志输出如下：
```log
2017-07-22 10:36:47.564 ERROR 2988 --- [nio-8080-exec-2] c.g.h.w.controller.IndexController       : system error
```
没有任何堆栈信息，查看代码：
```java
public Object systemVersion(){
    try{
        return playService.systemVersion();
    }catch(Exception e){
        logger.error("system error");
        return "system error";
    }
}
```
这段代码写的很有水平，吃异常并不一定是不正确的做法，但是日志中直接把堆栈信息丢掉一般一定是在给自己挖坑。日志正常的打印方法应该是这样的：
```java
logger.error("system error", e);
```

现在service层发生的异常堆栈信息全部丢掉了，常规做法一般是这样的：
修改日志代码 -> 重新发布 -> 重现错误 -> 定位问题 -> 解决问题

现在借助[greys-anatomy](https://github.com/oldmanpushcart/greys-anatomy)就可以直接在线上定位问题，查看堆栈信息。

进入greys工具：
```
[root@dev greys]# jps
3049 Jps
2988 Bootstrap
[root@dev greys]# ./greys.sh 2988
                                                        _                    
  ____  ____ _____ _   _  ___ _____ _____ ____  _____ _| |_ ___  ____  _   _ 
 / _  |/ ___) ___ | | | |/___|_____|____ |  _ \(____ (_   _) _ \|    \| | | |
( (_| | |   | ____| |_| |___ |     / ___ | | | / ___ | | || |_| | | | | |_| |
 \___ |_|   |_____)\__  (___/      \_____|_| |_\_____|  \__)___/|_|_|_|\__  |
(_____|           (____/                                              (____/ 
                                              +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                              |v|e|r|s|i|o|n|:|1|.|7|.|6|.|4|
                                              +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
ga?>
ga?>
```

这里使用watch （Display the details of specified class and method）工具来分析问题，可以通过help watch来查看具体使用方法：

```
ga?>help watch
+---------+----------------------------------------------------------------------------------+
|   USAGE | -[bfesx:En:] class-pattern method-pattern express condition-express              |
|         | Display the details of specified class and method                                |
+---------+----------------------------------------------------------------------------------+
| OPTIONS |              [b] | Watch before invocation                                       |
|         | -----------------+-------------------------------------------------------------- |
|         |              [f] | Watch after invocation                                        |
|         | -----------------+-------------------------------------------------------------- |
|         |              [e] | Watch after throw exception                                   |
|         | -----------------+-------------------------------------------------------------- |
|         |              [s] | Watch after successful invocation                             |
|         | -----------------+-------------------------------------------------------------- |
|         |             [x:] | Expand level of object (0 by default)                         |
|         | -----------------+-------------------------------------------------------------- |
|         |              [E] | Enable regular expression to match (wildcard matching by def  |
|         |                  | ault)                                                         |
|         | -----------------+-------------------------------------------------------------- |
|         |             [n:] | Threshold of execution times                                  |
……
```
这里只截取了片段，此处异常被吃掉，我需要查下抛出异常的方法所抛的异常信息是什么，最好是能把堆栈打出来。
此处异常是service层跑出来的，就watch下service （playService.systemVersion()）
```
watch -e *PlayService* systemVersion throwExp.printStackTrace()
```
简单解释下：
- -e表示：Watch after throw exception
- \*PlayService\* 是通配匹配类名，也可以写具体类名
- systemVersion 是方法名
- throwExp 是工具定义的参数名，表示抛出的异常对象的

执行完上面这句之后，再次重现问题，查看应用日志中输出了详细的堆栈信息
```log
java.lang.RuntimeException: ssss
	at com.github.hongkaiwen.webplay.service.PlayService.systemVersion(PlayService.java:12)
	at com.github.hongkaiwen.webplay.controller.IndexController.systemVersion(IndexController.java:28)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205)
……
```
### bug 2
这个接口调用第三方接口（假设我写的service层方法就是一个第三方的http接口）校验余额是否不足，预期的第三方接口返回的是一个json字符串。
在线上系统重现问题，系统日志中内容输出：
```
com.alibaba.fastjson.JSONException: syntax error, pos 1
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1295) ~[fastjson-1.2.7.jar:na]
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1205) ~[fastjson-1.2.7.jar:na]
	……
```
很明显，第三方接口返回的不是一个json，但和第三方扯比较麻烦，我们需要知道第三方到底返回了什么东西。
使用watch工具，查看第三方接口方法的返回值：
```
watch -f *PlayService* balanceQuery  returnObj
```
重现问题，然后查看控制台输出：
```
<html><body>500 Internal Error: xxxxx</body></html>
```
很明显第三方系统除了问题了。


### 总结
上面这个例子是我故意写的俩bug，但类似场景我相信总有人遇到过，没有这个工具也能解决，但有了这个工具，我们可以更快捷的定位问题，这种好东西一定要放到自的工具箱。

### 延伸
这个工具不只是我介绍的这一点功能，大家可以花点时间去简单研究下（半个小时就够了），看看有哪些功能，怎么安装，遇到问题的时候也许用得到。

help 查看所有工具：
 ```
 ga?>help
+----------+----------------------------------------------------------------------------------+
|       sc | Search all the classes loaded by JVM                                             |
+----------+----------------------------------------------------------------------------------+
|       sm | Search the method of classes loaded by JVM                                       |
+----------+----------------------------------------------------------------------------------+
|  monitor | Monitor the execution of specified Class and its method                          |
+----------+----------------------------------------------------------------------------------+
|    watch | Display the details of specified class and method                                |
+----------+----------------------------------------------------------------------------------+
|       tt | Time Tunnel                                                                      |
+----------+----------------------------------------------------------------------------------+
|    stack | Display the stack trace of specified class and method                            |
+----------+----------------------------------------------------------------------------------+
|   ptrace | Display the detailed thread path stack of specified class and method             |
+----------+----------------------------------------------------------------------------------+
|       js | Enhanced JavaScript                                                              |
+----------+----------------------------------------------------------------------------------+
|    trace | Display the detailed thread stack of specified class and method                  |
+----------+----------------------------------------------------------------------------------+
|  session | Display current session information                                              |
+----------+----------------------------------------------------------------------------------+
|     quit | Quit Greys console                                                               |
+----------+----------------------------------------------------------------------------------+
|  version | Display Greys version                                                            |
+----------+----------------------------------------------------------------------------------+
|      jvm | Display the target JVM information                                               |
+----------+----------------------------------------------------------------------------------+
|    reset | Reset all the enhanced classes                                                   |
+----------+----------------------------------------------------------------------------------+
|      asm | Display class bytecode by asm format                                             |
+----------+----------------------------------------------------------------------------------+
| shutdown | Shut down Greys server and exit the console                                      |
+----------+----------------------------------------------------------------------------------+
|     help | Display Greys Help                                                               |
+----------+----------------------------------------------------------------------------------+
|      top | Display The Threads Of Top CPU TIME                                              |
+----------+----------------------------------------------------------------------------------+

 ```
 
 可以help 具体的工具名查看用法：
 
 ```
 ga?>help top
+---------+----------------------------------------------------------------------------------+
|   USAGE | -[i:t:d]                                                                         |
|         | Display The Threads Of Top CPU TIME                                              |
+---------+----------------------------------------------------------------------------------+
| OPTIONS |             [i:] | The thread id info                                            |
|         | -----------------+-------------------------------------------------------------- |
|         |             [t:] | The top NUM of thread cost CPU times                          |
|         | -----------------+-------------------------------------------------------------- |
|         |              [d] | Display the thread stack detail                               |
+---------+----------------------------------------------------------------------------------+
| EXAMPLE | top                                                                              |
|         | top -t 5                                                                         |
|         | top -d                                                                           |
+---------+----------------------------------------------------------------------------------+

 ```
 
 希望可以帮到大家，
 向作者致敬！