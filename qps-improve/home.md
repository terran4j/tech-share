
# 在 Java 应用中提升 QPS 实战

### 导读
作为一名开发人员，你负责的系统可能会出现更高的并发访问量的挑战，
当然这是好事，因为这既反映出您的业务在蒸蒸日上，对你而言也是一个锤炼技术能力的技术。
<br>
本文的主题是记录实战过程中的一次 Java Web 应用性能优化过程，包括：
如何去排查性能问题，如何利用工具，以及如何去解决；
为大家提供一个优化的思路，以供大家参考。


### 问题背景
本次性能优化的起源，是一次线上事故引起的，当时一个系统的访问量突然暴增，系统不堪重负而导致接口响应变慢，
最终App客户端出现大量 socket timeout 异常。
<br>
事后笔者对这个系统进行梳理和排查，发现这个系统的业务逻辑其实并不复杂：

![arch-old](https://raw.githubusercontent.com/terran4j/tech-share/master/qps-improve/arch-old.jpg "系统架构简图")

当然这只是一个简图，真实的业务逻辑还是要比这个复杂一些，但主要的流程还是图上反映的那样：
从缓存中读数据，读到后返回给端（没读到就抛无数据异常）。
<br>
Web 服务器是 CentOS 系统， CPU 是 4 核，JVM堆内存大小为 4 G。
程序是基于 SpringBoot 1.5.2 构建的，内嵌的 Tomcat 。
<br>
分布式缓存用的是 CoachBase（简称 cb），对 CoachBase 不了解的读者可以参考
[这里](https://xiaoxiami.gitbook.io/couchbase/chapter1)
<br>

按理说这个接口的 QPS 应该很高才对，但实际测下来只有 1400，所以笔者的任务就是排查性能瓶颈并解决，以提升 QPS 指标。




### 安装接口压测工具 wrk

工欲善其事，必先利其器。
要进行性能优化，对接口进行压测是经常要做的事，所以我们希望有一款简单好用的压测工具，最终我们选择了 wrk 。
我们找了一台 CentOS 的机器作为压测机安装 wrk ，安装方法如下：

```shell
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

参数说明：
* -t300 指起 300 个线程来执行请求，每个线程在执行完一个HTTP请求后会立即执行下一个HTTP请求。
* -c400 指建立400个连接，用于 HTTP 请求，而不是每次请求都重新建立连接。
* -d30s 指本轮压测持续时间为 30 秒。

结果说明：
* Requests/sec 指每秒处理的请求数量，也就是我们说的 QPS （Query Per Second）


### 引入 Perf4j 

perf4j 是一款用于诊断 Java 应用程序性能的开源工具，它可以统计各阶段耗时统计，帮助我们找出性能瓶颈。
为了知晓在高并发情况下，程序各阶段的耗时，我们先引入 perf4j 开源项目，步骤如下：


1. 在项目 pom.xml 中添加 perf4j 依赖:

```xml
<dependency>
 <groupId>org.perf4j</groupId>
 <artifactId>perf4j</artifactId>
</dependency>
```


2. 在 Application 类（或其它 Configuration 类中添加 TimingAspect 的 Bean 定义）

```java
@Bean
public TimingAspect timingAspect() {
 return new TimingAspect();
}
```

注意： 我们的系统的日志库用的是 slf4j，因此要引的是 org.perf4j.slf4j.aop 包里面的 TimingAspect 类，不要引错了哦。


3. 对想要监测的代码，在方法上添加  @Profiled 注解，如：
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

4. 在 logback 日志文件中定义 Perf4j 的 Appender（推荐 Perf4j 的日志单独输出到一个文件），如：

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

解释笔者添加的性能监测代码的意义：
* 1-0-queryColumnDetail 是整个接口（Controller层的方法）的耗时统计；
* 1-x-xxxx 是在 1-0-queryColumnDetail 中的各段代码的耗时统计；
* 2-0-queryColumnDetailStatic 等同于 1-4-queryColumnDetailStatic （queryColumnDetail 方法调用 queryColumnDetailStatic 方法）
* 2-x-xx 是 2-0-queryColumnDetailStatic 中的各段代码耗时统计。
* 以此类推......

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

经过笔者对日志配置仔细分析，发现有几个可优化项：
1：将 ImmediateFlush 改为 false 。为 false 的意思是去掉写日志时立即刷磁盘的操作，理论上这样可以提升日志写入的性能。
2：将 queueSize 值改大。queueSize 值是指定日志缓存队列（一个 BlockingQueue）的最大容量，改大点可以提升性能（但也不要太大，以免占用太多内存）。
 笔者计算了一下： 一条日志按 500B 算（根据当前业务场景），10K条日志也就5M，因此这个值从 512 改到 1W 。
3：将 includeCallerData 从 true 改为 false 。includeCallerData 表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，
 为了提升性能，改为 false，即：当 event 被加入 BlockingQueue 时，event 关联的调用者数据不会被提取，只有线程名这些比较简单的数据。
<br>


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

结论： 优化日志配置后，QPS提升 到 2300 ，虽不如完全关闭日志后的 2400 ，但也相差不多了。
