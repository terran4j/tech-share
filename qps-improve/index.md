
# 提升 Java 应用 QPS 实战

### 导读

作为一名开发人员，你负责的系统可能会出现高并发的挑战。
当然这是好事，因为这既反映出您的业务在蒸蒸日上，对你而言也是一个锤炼技术能力的机会。
<br>
本文的主题是记录笔者在工作过程中的一次 Java Web 应用性能优化过程，包括：如何去排查性能问题，如何利用工具，以及如何去解决等，以供大家参考。


### 问题背景

本次性能优化的起源，是一次线上事故引起的，当时一个接口的访问量突然暴增，系统不堪重负而导致接口响应变慢，
最终 App 端的接口调用出现大量 socket timeout 异常。
<br>
事后笔者任务就是对这个接口排查性能瓶颈并解决，以提升 QPS 指标。<br>
经过梳理和排查，总结这个模块的架构如下：

![arch-old](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/arch-old.jpg "系统架构简图")

当然这只是一个简图，真实的业务逻辑还是要比这个复杂一点，但本质上它的业务逻辑并不复杂，主要的流程还是图上反映的那样，
即：从缓存中读数据，读到后返回给端上（没读到就抛无数据异常了）。
<br>

Web 服务器是 CentOS 系统， CPU 是 4 核的，JVM 堆内存大小为 4 G。
程序是基于 SpringBoot 1.5.2 构建的，内嵌的 Web 容器 Tomcat 。
<br>
分布式缓存用的是 CoachBase（后续简称 cb），对 CoachBase 不了解的读者可以参考
[这里](https://xiaoxiami.gitbook.io/couchbase/chapter1)
<br>

按理说这个接口的 QPS 应该很高才对，但实际测下来只有 1400，所以笔者的任务就是排查性能瓶颈并解决，以提升 QPS 指标。


### 安装接口压测工具 wrk

工欲善其事，必先利其器。
要进行性能优化，对接口进行压测是经常要做的事，所以我们希望有一款简单好用的压测工具，最终我们选择了 wrk 。
我们找了一台 CentOS 的机器作为压测机安装 wrk ，安装方法如下：

```jshelllanguage
sudo yum groupinstall 'Development Tools'
sudo yum install openssl-devel
sudo yum install git
git clone https://github.com/wg/wrk.git wrk
cd wrk
make
```

安装好后，我们对一台 web 服务器直接发起 HTTP 请求进行压测：
```
./wrk/wrk -t300 -c400 -d30s "http://10.41.135.74:8982/api/v3/dynamic/product/column/detail?productId=204918601"
```
执行的结果如下：

![try-wrk](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/try-wrk.png "wrk试用")

参数说明：<br>
* -t300 指起 300 个线程来执行请求，每个线程在执行完一个HTTP请求后会立即执行下一个HTTP请求。<br>
* -c400 指建立400个连接，用于 HTTP 请求，而不是每次请求都重新建立连接。<br>
* -d30s 指本轮压测持续时间为 30 秒。<br>

结果说明：
* Requests/sec 指每秒处理的请求数量，也就是我们说的 QPS （Query Per Second）


### 引入 Perf4j 

perf4j 是一款用于诊断 Java 应用程序性能的开源工具，它可以统计各阶段耗时统计，帮助我们找出性能瓶颈。
为了知晓在高并发情况下，程序各阶段的耗时，我们先引入 perf4j 开源项目，步骤如下：


* 在项目 pom.xml 中添加 perf4j 依赖:

```xml
<dependency>
 <groupId>org.perf4j</groupId>
 <artifactId>perf4j</artifactId>
</dependency>
```


* 在 Application 类（或其它 Configuration 类）中添加 TimingAspect 的 Bean 定义：

```java
@Bean
public TimingAspect timingAspect() {
 return new TimingAspect();
}
```

注意： 我们的系统的日志库用的是 slf4j，因此要引的是 org.perf4j.slf4j.aop 包里面的 TimingAspect 类，不要引错了哦。


* 对想要监测的代码，在方法上添加  @Profiled 注解，如：

```java
@Profiled
@RequestMapping(name = "请求 Card 化页面数据", value = "/cpage", method = RequestMethod.GET)
public String getCpageResult(
        @RequestParam(value = "cpageCode", required = false) String cpageCode,
        @RequestParam(value = "previewMode", required = false) String previewModeQuery) {
  // ...
}
```

或用 org.perf4j.StopWatch 类：

```java
public ColumnDetailV2 queryColumnDetailV2FromCache(Long qipuId) {
    org.perf4j.StopWatch stopWatch = new Slf4JStopWatch("queryColumnDetailV2FromCache");
    stopWatch.start("4-1-genColumnDetailV2CacheKey");
    String cacheKey = genColumnDetailV2CacheKey(qipuId);
    stopWatch.stop();
    stopWatch.start("4-2-getCmsInfoFromCB");
    String result = cmsCouchbaseOp.getCmsInfoFromCB(cacheKey);
    stopWatch.stop();
    if (StringUtils.isBlank(result)) {
        LOGGER.warn("queryColumnDetailV2FromCache: {} not exist, result: {}", cacheKey, result);
        return null;
    } else {
        stopWatch.start("4-3-parseColumnDetailV2");
        ColumnDetailV2 columnDetailV2 = JSON.parseObject(result, ColumnDetailV2.class);
        stopWatch.stop();
        return columnDetailV2;
    }
}
```

注意 SpringBoot 框架自身也提供了一个 StopWatch 类（在包 `org.springframework.util` 中），
它没有 `org.perf4j` 提供的 StopWatch 类的功能强大。

* 在 logback 日志文件中定义 Perf4j 的 Appender（推荐 Perf4j 的日志单独输出到一个文件），如：

```xml
<property name="LOG_HOME" value="/data/logs/kpp-lighthouse" />

<appender name="Perf4jFileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
   <File>${LOG_HOME}/lighthouse-cms-web-perf4.log</File>
   <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <!--文件的路径与名称,{yyyy-MM-dd.HH}精确到小时,则按小时分割保存-->
      <FileNamePattern>${LOG_HOME}/lighthouse-cms-web-perf4.%d{yyyy-MM-dd.HH}.log.gz</FileNamePattern>
      <!-- 如果当前是按小时保存，则保存48小时(=2天)内的日志（建议生产环境改成7天） -->
      <MaxHistory>48</MaxHistory>
   </rollingPolicy>
   <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>%m%n</pattern>
   </encoder>
</appender>

<appender name="Perf4jAppender" class="org.perf4j.logback.AsyncCoalescingStatisticsAppender">
    <!-- 
       TimeSlice 用来设置聚集分组输出的时间间隔，默认是 30000 ms，
       在生产环境中可以适当增大以供减少写文件的次数 
    -->
    <param name="TimeSlice" value="10000"/>
    <appender-ref ref="Perf4jFileAppender"/>
</appender>

<!-- Perf4j 默认用名称为 org.perf4j.TimingLogger 的 Logger -->
<logger name="org.perf4j.TimingLogger" level="INFO" additivity="false">
  <appender-ref ref="Perf4jAppender"/>
</logger>
```


引入完了后，笔者用 `org.perf4j.StopWatch` 的方式在程序代码中添加性能监测代码。
然后在本地（IDEA debug模式）运行程序，调用几次后 per4j 输出的统计数据如下：

![per4j-log](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/per4j-log.png "per4j-log")

上图中的 Tag 列是笔者添加性能监测代码时，给每段代码起的名字，这里解释它们的意义：<br>
* 1-0-queryColumnDetail 是整个接口（Controller层的方法）的耗时统计；<br>
* 1-x-xxxx 是在 1-0-queryColumnDetail 中的各段代码的耗时统计；<br>
* 2-0-queryColumnDetailStatic 等同于 1-4-queryColumnDetailStatic （queryColumnDetail 方法调用 queryColumnDetailStatic 方法）<br>
* 2-x-xx 是 2-0-queryColumnDetailStatic 中的各段代码耗时统计。<br> 
* 以此类推......<br>

我们可以看到：接口的总耗时只有 25 ms，主要耗时在读 cb 缓存（23ms），其中读 cb 为 19 ms ，将 json 解析为对象 3 ms 。



### 准备压测

为了进行压测，我们将线上一台机器（10.41.135.74）切走流量，把含性能监控日志的代码部署到这台机器上，
后续我们将对它进行 N 轮压测：压测 -> 优化 -> 再压测 -> 再优化......
直到达到一个让人接受的 QPS 指标为止。
<br>
*（注意： 如果是服务启动过，要先压测一轮以进行预热，并且这次的数据不作准）*


### 第 0 轮：无压力测试

我们先了解一下在无压力的情况程序性能表现如何，如果发现哪段代码耗时较长，就排查这段代码好了。
<br>
不压测，只访问接口一次，查看性能统计数据如下：

![ptest-0](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-0.png "ptest-0")

可以看出来，没有压力时，整个接口的总时长就 3 ms，基本上就不耗性能。
也就是说，程序没有哪段代码有明显的性能损耗，只有压力上上去了才有。

###　第 1 轮： 诊断是否是日志造成的性能损耗

笔者首先猜想可能是日志输出较多造成的性能损耗，因为日志导致的性能问题的案例比较常见（笔者在以前的项目中就遇到过类似的问题）。
<br>
但是猜归猜，还需要实际测试来确认这个猜想，所以笔者打算在有日志输出时压测一次，在无日志输出时再压测一次，比较下它们的 QPS 差异。

这里先交待一下原来的 logback.xml 日志配置如下：

```xml
<property name="LOG_HOME" value="/data/logs/kpp-lighthouse" />
<!-- 正常 info 级别的程序日志输出 -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>${LOG_HOME}/lighthouse-cms-dynamic.log</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <!--日志文件输出的文件名-->
    <FileNamePattern>${LOG_HOME}/lighthouse-cms-dynamic.%d{yyyy-MM-dd}.log.gz</FileNamePattern>
    <!--日志文件保留天数-->
    <MaxHistory>60</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
    <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的[如配置80,则80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志] -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <includeCallerData>true</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="FILE"/>
</appender>
<logger name="o.s">
    <level value="DEBUG" />
</logger>
<logger name="c.i.input.dao">
    <level value="DEBUG" />
</logger>
<root level="INFO">
    <appender-ref ref="ASYNC_DAILY_ROLLING_FILE"/>
</root>
```

日志配置用的是 `RollingFileAppender`，是一种很常见的日志配置方法（细心的读者们发现什么端倪了么？）
<br>

在以上日志配置下，笔者起 300 并发进行压测：
```
./wrk -t300 -c600 -d60s http://10.41.135.74:8982/api/v3/dynamic/product/column/detail?productId=204918601
```

结果如下：

![ptest-log-1-1](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-log-1-1.png "有日志下的压测结果")

QPS 只有 1400 左右。
<br>

然后我们将所有日志级别都调成 ERROR 级别，再压测：

![ptest-log-1-2](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-log-1-1.png "无日志下的压测结果")

发现 QPS 从 1400 提升到 2400，因此确认了日志输出对程序性能有较大损耗。


### 第 2 轮：优化日志配置

虽然从上一轮我们知道是日志输出的问题，但笔者也不敢真将日志输出都关闭，万一哪天出线上 BUG 了，没有日志岂不是得抓狂？？<br>

经过笔者对日志配置仔细分析，发现有几个可优化项：<br>
* 一、将 ImmediateFlush 改为 false 。为 false 的意思是去掉写日志时立即刷磁盘的操作，理论上这样可以提升日志写入的性能。<br>
* 二、将 queueSize 值改大。queueSize 值是指定日志缓存队列（一个 BlockingQueue）的最大容量，改大点可以提升性能（但也不要太大，以免占用太多内存）。
 笔者计算了一下： 一条日志按 500B 算（根据当前业务场景），10K条日志也就5M，因此这个值从 512 改到 1W 。<br>
* 三、将 includeCallerData 从 true 改为 false 。includeCallerData 表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，
 为了提升性能，改为 false，即：当 event 被加入 BlockingQueue 时，event 关联的调用者数据不会被提取，只有线程名这些比较简单的数据。<br>


调整参数后的日志配置如下：

```xml
<!-- 正常 info 级别的程序日志输出 -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>${LOG_HOME}/lighthouse-cms-dynamic.log</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--日志文件输出的文件名-->
        <FileNamePattern>${LOG_HOME}/lighthouse-cms-dynamic.%d{yyyy-MM-dd}.log.gz</FileNamePattern>
        <!--日志文件保留天数-->
        <MaxHistory>60</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!-- 优化项1：去掉写日志时立即刷磁盘的操作，以提升日志写入的性能。 -->
        <ImmediateFlush>false</ImmediateFlush>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的[如配置80,则80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志] -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 优化项2：本项是暂存日志记录的 BlockingQueue 的最大容量，改大点可以提升性能（但也不要太大，会占用内存），
        一条日志500B算，10K条日志也就5M，因此这个值从 512 改到 1W 。 -->
    <queueSize>10000</queueSize>
    <!-- 优化项3：includeCallerData表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，
        为了提升性能，默认为false，即：当event被加入BlockingQueue时，event关联的调用者数据不会被提取，只有线程名这些比较简单的数据
        因此这个值从 true 改为 false 。
    -->
    <includeCallerData>false</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="FILE"/>
</appender>
<root level="INFO">
    <appender-ref ref="ASYNC_DAILY_ROLLING_FILE"/>
</root>
```

改后再进行压测，结果如下：

![ptest-log-2](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-log-2.png "优化日志后的压测结果")

优化日志配置后，QPS提升 到 2300 ，虽不如完全关闭日志后的 2400 ，但也相差不多了。


### 第 3 轮： 分析系统性能瓶颈

QPS 到 2300 就是极限了么？ 笔者觉得应该还远没到程序的`最佳状态`，
这一轮我们尝试分析下压测状态下的机器性能，如 CPU 内存 网络IO 等指标，看能不能找出性能瓶颈。
<br>

我们先查看 java 进程的性能：<br>
* 用 `java -ef | grep java` 查看 java 程序的`进程id`(这步开发者应该都会吧，这里就省略截图了)；<br>
* 用 `pidstat -p [进程id] [打印的间隔时间] [打印次数]` 命令来打印出 java 进程的 CPU 使用情况：

![ptest-cpu-1](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-1.png "进程CPU使用情况")

笔者发现，一旦压测开始， CPU 直接涨到 100% （上图红框所示）。（这里显示的 CPU 为每个核的平均值，而不是加和值，因此虽然是 4 核，100% 指 CPU 已打满了）。<br>
* 为了知道是具体哪个线程占 CPU 高，用 `top -Hp [进程id]` 来查看线程的 CPU 占用情况：

![ptest-cpu-2](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-2.png "线程CPU使用情况")

发现是 id 为 63099 的线程占 CPU 最高，为 46% 。<br>
* 用 `printf "%x\n" [进程id]` 打印出此线程 id 的十六进制值： f67b 。<br>
* 用 `jstack [进程id] |grep -B 1 -C 20 [线程id十六进制值]` 打印出此线程的堆栈信息（建议多打印几次），如下：

![ptest-cpu-3](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-3.png "线程堆栈1")
![ptest-cpu-4](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-4.png "线程堆栈2")

<br>

通过以上堆栈信息，发现是下面这个日志输出的 Appender 线程占用了大量的 CPU 资源：
```
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
```

具体来说，有两个问题：<br>
* 一、 logback 用 ArrayBlockingQueue 缓存日志对象，似乎在 take 端 和 poll 端存在资源竞争（ArrayBlockingQueue 存取元素用的是一把锁）。<br>
* 二、 仍有获取 caller 数据的情况（还记得第 2 轮的优化项 3 么？）<br>

对第 1 个问题，网上查了下，logback 的低版本在创建 ArrayBlockingQueue 时，用的是公平锁，的确会有性能问题
（它在高版本解决了，用非公平锁），在我们项目中升级 logback jar 的改动较大，笔者简单起见直接 hack 改它的源码，将类 AsyncAppenderBase 的源码单独拷贝出来：
![ptest-cpu-5](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-5.png "修改AsyncAppenderBase源码")

然后进行修改：

```java
public class AsyncAppenderBase<E> extends UnsynchronizedAppenderBase<E> implements AppenderAttachable<E> {
    
    // 省略其它代码...

    public void start() {
        if (!this.isStarted()) {
            if (this.appenderCount == 0) {
                this.addError("No attached appenders found.");
            } else if (this.queueSize < 1) {
                this.addError("Invalid queue size [" + this.queueSize + "]");
            } else {
                this.blockingQueue = new ArrayBlockingQueue(this.queueSize, false);
                if (this.discardingThreshold == -1) {
                    this.discardingThreshold = this.queueSize / 5;
                }

                this.addInfo("Setting discardingThreshold to " + this.discardingThreshold);
                this.worker.setDaemon(true);
                this.worker.setName("AsyncAppender-Worker-" + this.getName());
                super.start();
                this.worker.start();
            }
        }
    }
    
    // 省略其它代码...
```

其实只改了一行源代码，就是将：
`this.blockingQueue = new ArrayBlockingQueue(this.queueSize);`
改成：
`this.blockingQueue = new ArrayBlockingQueue(this.queueSize, false);`
（SpringBoot 的类加载机制，是优先加载我们项目中的代码，因此这样就替换掉了在 logback JAR包中的同名类）。
<br>

对第 2 个问题，终于在 logback.xml 中找到了问题所在，原来日志配置中有提取 method 信息（<pattern>元素中有 %method），还是会触发读取调用者信息：

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <!-- 省略其它代码 ... -->
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!-- 优化项1：去掉写日志时立即刷磁盘的操作，以提升日志写入的性能。 -->
        <ImmediateFlush>false</ImmediateFlush>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
```

所以把 %method 去掉。


改了这两个地方后，部署代码再压测：
![ptest-cpu-6](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cpu-6.png "优化日志后的压测结果")

发现 QPS 提升到 2500，虽然不是很明显，但也算前进了一步。


### 第 4 轮： 优化 JVM 参数


经过上一轮优化后，我们再看一下 CPU 情况。通过 `top -Hp [进程id]`，我们查看下压测状态下占 CPU 高的前几个线程：

```
13753 root 20 0 7905m 4.3g 6584 R 24.0 56.0 3:48.70 java
13713 root 20 0 7905m 4.3g 6584 S 12.3 56.0 0:20.75 java
13716 root 20 0 7905m 4.3g 6584 S 12.3 56.0 0:20.71 java
13715 root 20 0 7905m 4.3g 6584 S 12.0 56.0 0:20.57 java
13714 root 20 0 7905m 4.3g 6584 S 11.6 56.0 0:20.85 java
20154 root 20 0 7905m 4.3g 6584 R 8.7 56.0 0:10.30 java
20137 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.28 java
20141 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.24 java
20159 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.23 java
14080 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:11.44 java
20129 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:10.92 java
20138 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.27 java
20140 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.32 java
20144 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:10.25 java
20151 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.27 java
......
```

* 排名第 1 位的线程，占 CPU 24% ，经排查它仍是日志线程；<br>
* 排名第 2 ~ 5 位的线程，每个占 CPU 12%（4个加起来就是 48%），经排查它们都是 GC 线程（用于 YGC）：

![ptest-jvm-1](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-1.png "YGC线程信息")

* 排名第 6 位及以后线程，平均每个占比 7~8%，经排查大多都是 Web 容器的线程，阻塞在等待 CB 调用的返回上：

![ptest-jvm-2](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-1.png "Web容器线程信息")


虽然这三个都可能是问题，但我们每次都聚焦在一个问题是解决，我们选择先解决第 2 个问题，即 GC 线程的问题。<br>

我们用 `jstat -gcutil [进程id] [打印的间隔时间（毫秒）] [打印的次数]` 命令来查一下 GC 的情况：

![ptest-jvm-3](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-3.png "GC信息")

每列解释如下：<br>
* S0：幸存1区当前使用比例<br>
* S1：幸存2区当前使用比例<br>
* E：伊甸园区使用比例<br>
* O：老年代使用比例<br>
* M：元数据区使用比例<br>
* CCS：压缩使用比例<br>
* YGC：年轻代垃圾回收次数<br>
* FGC：老年代垃圾回收次数<br>
* FGCT：老年代垃圾回收消耗时间<br>
* GCT：垃圾回收消耗总时间<br>

上图的红框部分，是压测开始时的 GC 信息，从图上 S0 区、S1 区在压测开始后就不停的捣腾，并且 YGC 这列数字增加得很快，说明 YGC 比较严重。<br>
那 YGC 具体有多严重，以及 YGC 的具体情况是怎样的，
我们用 `jstat -gcnewcapacity [进程id] [打印的间隔时间（毫秒）] [打印的次数]` 命令输出 JVM 新生代的具体信息

![ptest-jvm-4](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-4.png "YGC信息")

每列解释如下：<br>
* NGCMN：新生代最小容量<br>
* NGCMX：新生代最大容量<br>
* NGC：当前新生代容量<br>
* S0CMX：最大幸存0区大小<br>
* S0C：当前幸存0区大小<br>
* S1CMX：最大幸存1区大小<br>
* S1C：当前幸存1区大小<br>
* ECMX：最大伊甸园区大小<br>
* EC：当前伊甸园区大小<br>
* YGC：年轻代垃圾回收次数<br>
* FGC：老年代回收次数<br>
上图的红框部分，是压测开始时的 YGC 总次数，我们发现竞然每 1 秒要执行 6 次 YGC。
另外，NGCMX（新生代最大容量）只有 340 M，S0CMX（最大幸存 0 区大小）、S1CMX（最大幸存 1 区大小）都只有 34 M，的确是有点小。<br>


因此我们在 JVM 启动命令中设置 JVM 参数， 添加 -Xmn1536m 及 -XX:SurvivorRatio=6 两个参数：<br>
* -Xmn1536m 是指新生代最大容易为 1.5G（总堆内存为 4 G）；<br>
* -XX:SurvivorRatio=6 是指 S0(幸存0区) : S1(幸存1区) : EC(伊甸园区) 的比例为 2:2:6。<br>

设置好 JVM 参数后，我们再压测：

![ptest-jvm-5](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-5.png "优化JVM后的压测结果")

发现 QPS 提升到 2680（接近 2700）。
同时GC 线程的 CPU 占用率降下来了，而YGC 的次数，也由 1 秒 6 次，变成了 2 秒 1 次。

![ptest-jvm-6](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-jvm-6.png "优化JVM后的YGC信息")


### 第 5 轮： 将 Web 容器从 Tomcat 改为 Undertow

经过上一轮优化后，发现不再有明显 CPU 高的线程(最高还是日志线程 14%)，
但 Web 容器中的 Worker 线程平均为 7%，因此可以考虑下对 Web 容器进行优化。<br>

在网上查资料，有的文章分析说 Undertow 的性能比 Tomcat 好，
因此我们将 SpringBoot 中默认的 Tomcat 改为 Undertow 试试看。<br>

首先，在 pom.xml 配置如下：

```xml
<!-- 网上说 Undertow 的性能比 Tomcat 好，试试看？ -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <!-- 移除掉默认支持的 Tomcat -->
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<!-- 添加 Undertow 容器 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

然后，在 application.yml 文件中配置 Undertow 的相关参数：

```yaml
server:
    port: 8982
    undertow:
        io-threads: 4          # 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置每个CPU核心一个线程
        worker-threads: 256    # 阻塞任务线程池, 当执行类似servlet请求阻塞操作, undertow会从这个线程池中取得线程,它的值设置取决于系统的负载
                               # 这个值要根据阻塞系数来设置，并不是所谓的越大越好，太大反而会因竞争资源导致性能下降！！
        # 以下的配置会影响buffer,这些buffer会用于服务器连接的IO操作,有点类似netty的池化内存管理
        # 每块buffer的空间大小,越小的空间被利用越充分，不要设置太大，以免影响其他应用，合适即可
        # 每个区分配的buffer数量 , 所以 pool 的大小是buffer-size * buffers-per-region
        buffer-size: 1048
        buffers-per-region: 1048
        direct-buffers: true   #  是否分配的直接内存(NIO直接分配的堆外内存)，建议开启可以使用堆外内存减少byte数组的回收开销。
        accesslog:
            enabled: false
```

再压测:

![ptest-undertow-1](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-undertow-1.png "优化Web容器后的压测结果")

发现 QPS 提升到 3800，果然 Undertow 的性能比 Tomcat 好不少。


### 第 6 轮： 添加加上内存缓存

之前发现 Web 容器的 Worker 线程大多阻塞在调用 cb 上，考虑到当前场景下 cb 中的数据是静态（每 5 分钟变化一次），
可以进行内存缓存，也就是如果从 cb 中读到了数据，就放在内存缓存中（当然要将内存缓存设置一个比较短的过期时间）。<br>

经过比较，我们选择了 Guava 作为内存缓存的实现方案，代码如下：

```java
@Configuration
@EnableCaching
public class GuavaCacheConfig {
 
    @Bean
    public CacheManager cacheManager() {
        GuavaCacheManager cacheManager = new GuavaCacheManager();
        cacheManager.setCacheBuilder(
                CacheBuilder.newBuilder().
                        expireAfterWrite(10, TimeUnit.SECONDS).
                        maximumSize(10000));
        return cacheManager;
    }
}
```

经过计算，本场景下一条缓存平均大小为 10KB，1W 条就是 100M，而 JVM 堆内存有 4G，理论上是完全撑得住的。
在 cb 中的数据，是每 5 分钟刷新一次，因此内存缓存设置为 10 秒过期，也完全没有问题。<br>


然后在我们在读 cb 的地方加上内存缓存，代码如下：

```java
@Cacheable(value = "column-detail", key = "#qipuId")
public ColumnDetailV2 queryColumnDetailV2FromCache(Long qipuId) {
    LOGGER.info("queryColumnDetailV2FromCache, qipuId = {}", qipuId);
    String cacheKey = genColumnDetailV2CacheKey(qipuId);
    String result = cmsCouchbaseOp.getCmsInfoFromCB(cacheKey);
    if (StringUtils.isBlank(result)) {
        LOGGER.warn("queryColumnDetailV2FromCache: {} not exist, result: {}", cacheKey, result);
        return null;
    } else {
        return JSON.parseObject(result, ColumnDetailV2.class);
    }
}
```

关键是第 1 行代码：`@Cacheable(value = "column-detail", key = "#qipuId")`
即缓存块名为 column-detail，其中每条缓存的 key 为参数 qipuId 的值。

考虑到加上内存缓存后，处理请求的 Worker 线程 Blocking 时间会大大减少，这里改小了 undertow 的线程数：

```yaml
server:
    port: 8982
    undertow:
        ioThreads: 4
        workerThreads: 32
```

io 线程数改为 4 （因为被测机器的 CPU 只有 4 核）， worker 线程数改为 32 （即 io 线程数的 8 倍）。
注意：并不是线程数越多越小，因为太多线程数，资源竞争加剧反而对性能有损耗。


将修改的代码部署到压测环境，再压测：

![ptest-cache-1](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/ptest-cache-1.png "添加内存缓存后的压测结果")

QPS 进一步提升到 5000，说明在“恰当”的场景下，在分布式缓存的基础上加上内存缓存，能更进一步提升性能。


### 总结与展望

由于 5000 的 QPS 对当前业务场景已经足够用了，因此笔者也不打算花时间继续优化了。<br>
最后总结下做性能优化的一些思路和原则：<br>
* 数据驱动：通过“压测 + 观察数据”的方式，来分析程序的性能瓶颈，而不要想当然（即使是猜想，也要先想办法验证你的猜想再动手优化）；<br>
* 聚焦最大瓶颈：如果有多个优化方向，则先评估哪一个方向的问题是最大瓶颈，然后集中精力去优化这个最大瓶颈，优化完后再评估下一个最大瓶颈并解决，如此迭代前进；<br>
* 利用工具： 现在业界性能优化的工具还是比较多的，用好这些工具可以事半功倍，有些特殊场景即使没有好用的工具，也可以花点精力制造些小工具来使用。<br>

那实际上还有没有优化空间呢？笔者判断应该还是有不少优化空间的，感兴趣读可以继续尝试更多的方案哦！