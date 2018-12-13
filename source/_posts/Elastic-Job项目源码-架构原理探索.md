---
title: Elastic-Job项目源码-架构原理探索
date: 2018-12-08 20:49:50
tags:  
    - elastic-job
---

- 在之前的文章中介绍了elatic-job的出现背景，和能解决的问题，下面我们通过源码的方式来分析一下elatic-job的架构。

# 1、环境准备

##  Zookeeper
使用Zookeeper为注册中心。所以需要在本地启动zookeeper，作为Dubbo的注册中心。
启动Zookeeper服务：用于elastic-job的调度中心。
启动zooweb:这种看zk客户端有很多，我推荐zooweb，基于淘宝的zkWeb客户端改造成springboot的启动，很方便。
github:
https://github.com/zhuhongyu345/zooweb
<img src="/assets/2018/elstic-job/zooweb.png" width = "600" div align=center />


##  elastic-job-lite 源码
gitHub:
https://github.com/elasticjob/elastic-job-lite

<!-- more -->
# 2、elastic-job角色
再看一次架构：
<img src="/assets/2018/elstic-job/elastic-job-lite-jiagou.png" width = "800" div align=center />

- 作业job
Elastic-job提供三种作业类型：Simple类型作业、Dataflow类型作业、Script类型作业。
- 注册中心
Elastic-Job依赖Zookeeper作为注册中心，利用zk的功能完成节点选举、分片和配置变更等相关的功能
- 任务调度框架
Elastic-Job底层依赖quartz框架进行任务的调度

# 3、elastic-job源码大致解读

<img src="/assets/2018/elstic-job/elastic-job-code.png" width = "800" div align=center />
## elastic-job-example
elastic-job源码虽说注释很少，但是测试用例，测试demo可以说覆盖率极高，点个大赞，程序猿阅读足矣。
- elastic-job-example-cloud
- elastic-job-example-embed-zk
- elastic-job-example-jobs
- elastic-job-example-lite-java
- elastic-job-example-lite-spring
- elastic-job-example-lite-springboot

## elastic-job-lite-console
elastic-job的控制台，方便查看各个任务的执行情况和手工调度，禁用等等功能。
## elastic-job-lite-core
elastic-job的核心库，负责任务创建，状态变更，选举，路由等等核心功能。
## elastic-job-lite-lifecycle
elastic-job的任务的操作封装类，和作业相关的 API 集合服务。
## elastic-job-lite-spring
elastic-job的任务支持Spring容器加载。

# 4、elastic-job 例子
 
 ## 官方的例子
 ### 这里简单的先使用elastic-job-example-lite-springboot的例子
  - pom.xml：依赖
  
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    ...
    <dependencies>
        ...
        <dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-core</artifactId>
            <version>2.1.5</version>
        </dependency>
        <dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-spring</artifactId>
            <version>2.1.5</version>
        </dependency>
        ...
    </dependencies>
</project>
``` 
 
- application.yml:zk配置，以及任务配置

``` yml
regCenter:
  serverList: localhost:2181
  namespace: elastic-job-lite-springboot
  
simpleJob:
  cron: 0/5 * * * * ?
  shardingTotalCount: 3
  shardingItemParameters: 0=Beijing,1=Shanghai,2=Guangzhou
  
``` 

- SpringSimpleJob.java: Job类

``` java
public class SpringSimpleJob implements SimpleJob {
    
    @Resource
    private FooRepository fooRepository;
    
    @Override
    public void execute(final ShardingContext shardingContext) {
        System.out.println(String.format("Item: %s | Time: %s | Thread: %s | %s",
                shardingContext.getShardingItem(), new SimpleDateFormat("HH:mm:ss").format(new Date()), Thread.currentThread().getId(), "SIMPLE"));
        List<Foo> data = fooRepository.findTodoData(shardingContext.getShardingParameter(), 10);
        for (Foo each : data) {
            fooRepository.setCompleted(each.getId());
        }
    }
}
``` 
- 运行结果
<img src="/assets/2018/elstic-job/elastic-job-eg-1.png" width = "800" div align=center />

- zk节点分布
<img src="/assets/2018/elstic-job/elastic-job-zk.jpg" width = "800" div align=center />

 ### 基于elastic-job-lite-spring-boot-starter构架，更简单
 
  - pom.xml：依赖
  
``` xml
    <dependencies>
        ...
        <dependency>
            <groupId>com.github.kuhn-he</groupId>
            <artifactId>elastic-job-lite-spring-boot-starter</artifactId>
            <version>2.1.52</version>
        </dependency>
        ...
    </dependencies>
``` 
 
- application.yml:zk配置，以及任务配置

``` yml
elaticjob:
  zookeeper:
    server-lists: 127.0.0.1:2181
    namespace: elastic-job-lite-demo
  
``` 

- MyJob.java: Job类

``` java
@ElasticSimpleJob(cron = "*/10 * * * * ?",jobName = "MyJob",jobParameter = "1~2~")
@Component
public class MyJob implements SimpleJob {

    @Override
    public void execute(ShardingContext arg0) {
        System.out.println("hello,ElasticSimpleJob-->" + System.currentTimeMillis());
    }
}
``` 

启动SpringBootMain，一样的效果。

后面会针对源码，进行深入了解。