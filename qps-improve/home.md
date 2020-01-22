
<h1>在 Java 应用中提升 QPS 实战</h1>

<h3>导读</h3>
作为一名开发人员，你负责的系统可能会出现更高的并发访问量的挑战，
当然这是好事，因为这既反映出您的业务在蒸蒸日上，对你而言也是一个锤炼技术能力的技术。
<br>
本文的主题是记录实战过程中的一次 Java Web 应用性能优化过程，包括：
如何去排查性能问题，如何利用工具，以及如何去解决；
为大家提供一个优化的思路，以供大家参考。


<h3>问题背景</h3>
本次性能优化的起源，是一次线上事故引起的，当时一个系统的访问量突然暴增，系统不堪重负而导致接口响应变慢，
最终App客户端出现大量 socket timeout 异常。
<br>
事后笔者对这个系统进行梳理和排查，发现这个系统的业务逻辑其实并不复杂：
 ![blockchain](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/
 u=702257389,1274025419&fm=27&gp=0.jpg "区块链")



<h3>安装接口压测工具 wrk</h3>

按这个文档安装， https://www.ruoxiaozh.com/blog/article/84， 具体不介绍了。



一、添加 perf4j （用于记录程序各阶段耗时统计，找出性能瓶颈）
为了能了解高并发情况下，程序各阶段的耗时，我们先引入 perf4j 开源项目。

引用 perf4j 依赖

```xml
<dependency>
 <groupId>org.perf4j</groupId>
 <artifactId>perf4j</artifactId>
</dependency>
```

2. 在 Application 类（或其它 Configuration 类中添加 TimingAspect 的 Bean 定义）

@Bean
public TimingAspect timingAspect() {
 return new TimingAspect();
}
注意： 我们的系统的日志库用的是 slf4j，因此要引的是 org.perf4j.slf4j.aop 包里面的 TimingAspect 类，不要引错了！



3. 对想要监测的代码，在方法上添加  @Profiled 注解：

@Profiled
@RequestMapping(name = "请求 Card 化页面数据", value = "/cpage", method = RequestMethod.GET)
public String getCpageResult(
        @RequestParam(value = "cpageCode", required = false) String cpageCode,
 @RequestParam(value = "previewMode", required = false) String previewModeQuery) {


}

或用 org.perf4j.StopWatch 类：

public ColumnDetailV2 queryColumnDetailV2FromCache(Long qipuId) {
    org.perf4j.StopWatch stopWatch = new Slf4JStopWatch("queryColumnDetailV2FromCache");
 stopWatch.start("4-1-genColumnDetailV2CacheKey");
 String cacheKey = genColumnDetailV2CacheKey(qipuId);
 stopWatch.stop();
 stopWatch.start("4-2-getCmsInfoFromCB");
 String result = cmsCouchbaseOp.getCmsInfoFromCB(cacheKey);
 stopWatch.stop();
 if (StringUtils.isBlank(result)) {
        LOGGER.warn("queryColumnDetailV2FromCache: " + cacheKey + " not exist, result:" + result);
 return null;
 } else {
        stopWatch.start("4-3-parseColumnDetailV2");
 ColumnDetailV2 columnDetailV2 = JSON.parseObject(result, ColumnDetailV2.class);
 stopWatch.stop();
 return columnDetailV2;
 }
}


4. logback 日志文件中定义 Perf4j 的 Appender，推荐 Perf4j 的日志单独输出到一个文件：
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

<appender name="Perf4jAppender"
 class="org.perf4j.logback.AsyncCoalescingStatisticsAppender">
 <!-- TimeSlice 用来设置聚集分组输出的时间间隔，默认是 30000 ms，
 在生产环境中可以适当增大以供减少写文件的次数 -->
 <param name="TimeSlice" value="10000"/>
 <appender-ref ref="Perf4jFileAppender"/>
</appender>

<!-- Perf4j 默认用名称为 org.perf4j.TimingLogger 的 Logger -->
<logger name="org.perf4j.TimingLogger" additivity="false">
 <level value="INFO"/>
 <appender-ref ref="Perf4jAppender"/>
</logger>




二、 用 Perf4j 的 API 添加性能监测代码
本地运行程序，找指定的日志文件中查看性能统计数据：



本地（IDEA debug模式）调用几次后的统计数据显示：

总耗时只有 25 ms ，主要耗时在读 cb 缓存（23ms），其中读 cb 为 19 ms ，将 json 解析为对象 3 ms 。

解释下：

1-0-queryColumnDetail 是整个接口的耗时统计；
1-x-xxxx 是  1-0-queryColumnDetail 的分阶段耗时统计；



2-0-queryColumnDetailStatic 等同于 1-4-queryColumnDetailStatic （queryColumnDetail 方法调用 queryColumnDetailStatic 方法）
2-x-xx 是 2-0-queryColumnDetailStatic 的分阶段耗时统计。



三、部署到压测环境，进行压测
部署含性能监控日志的代码到线上机器  10.41.135.74，此机器无任何其它流量，我们将对它进行 N 轮压测。

（注意： 如果是服务启动过，要先压测一轮以进行预热，并且这次的数据不作准）

具体排查及优化过程，见子页面。
具体代码，在分支 cms-ptest，欢迎大家 check out 出来查看。