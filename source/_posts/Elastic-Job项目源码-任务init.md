---
title: Elastic-Job项目源码-任务init
date: 2018-12-12 21:46:47
tags:  
    - elastic-job
---


# 涉及的类图

<img src="/assets/2018/elstic-job/elastic-job-init-class.png"  div align=center />

# 入口

``` java
     
    private static void setUpSimpleJob(final CoordinatorRegistryCenter regCenter, final JobEventConfiguration jobEventConfig) {
        //定义任务
        JobCoreConfiguration coreConfig = JobCoreConfiguration.newBuilder("javaSimpleJob", "0/5 * * * * ?", 3).shardingItemParameters("0=Beijing,1=Shanghai,2=Guangzhou").build();
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(coreConfig, JavaSimpleJob.class.getCanonicalName());
        //任务初始化
        new JobScheduler(regCenter, LiteJobConfiguration.newBuilder(simpleJobConfig).build(), jobEventConfig).init();
    }
```

<!-- more -->

# 初始化
## JobScheduler-init
``` java
//JobScheduler.java
/**
* 初始化作业.
*/
public void init() {
   // 更新 作业配置
   LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
   // 设置 当前作业分片总数
   JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
   // 创建 作业调度控制器
   JobScheduleController jobScheduleController = new JobScheduleController(
           createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
   // 添加 作业调度控制器
   JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
   // 注册 作业启动信息
   schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
   // 调度作业
   jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
}

```
 
## 更新作业配置

``` java
// SchedulerFacade.java
/**
* 更新作业配置.
*
* @param liteJobConfig 作业配置
* @return 更新后的作业配置
*/
public LiteJobConfiguration updateJobConfiguration(final LiteJobConfiguration liteJobConfig) {
   // 更新 作业配置
   configService.persist(liteJobConfig);
   // 读取 作业配置
   return configService.load(false);
}

```

## 设置作业分片总数

``` java
// JobRegistry.java
private Map<String, Integer> currentShardingTotalCountMap = new ConcurrentHashMap<>();
/**
* 设置当前分片总数.
*
* @param jobName 作业名称
* @param currentShardingTotalCount 当前分片总数
*/
public void setCurrentShardingTotalCount(final String jobName, final int currentShardingTotalCount) {
   currentShardingTotalCountMap.put(jobName, currentShardingTotalCount);
}

```

## 创建作业调度控制器

``` java
//JobScheduler.java
// 创建 作业调度控制器
   JobScheduleController jobScheduleController = new JobScheduleController(
           createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
 
```

- JobScheduleController,作业调度控制器，提供对 Quartz 方法的封装：

``` java
public final class JobScheduleController {

    /**
     * Quartz 调度器
     */
    private final Scheduler scheduler;
    /**
     * 作业信息
     */
    private final JobDetail jobDetail;
    /**
     * 触发器编号
     * 目前使用工作名字( jobName )
     */
    private final String triggerIdentity;
    
    public void scheduleJob(final String cron) {} // 调度作业
    public synchronized void rescheduleJob(final String cron) {} // 重新调度作业
    private CronTrigger createTrigger(final String cron) {} // 创建触发器
    public synchronized boolean isPaused() {} // 判断作业是否暂停
    public synchronized void pauseJob() {} // 暂停作业
    public synchronized void resumeJob() {} // 恢复作业
    public synchronized void triggerJob() {} // 立刻启动作业
    public synchronized void shutdown() {} // 关闭调度器
}

```

- 调用 #createScheduler() 方法创建 Quartz 调度器：

``` java
// JobScheduler.java
private Scheduler createScheduler() {
   Scheduler result;
   try {
       StdSchedulerFactory factory = new StdSchedulerFactory();
       factory.initialize(getBaseQuartzProperties());
       result = factory.getScheduler();
       result.getListenerManager().addTriggerListener(schedulerFacade.newJobTriggerListener());
   } catch (final SchedulerException ex) {
       throw new JobSystemException(ex);
   }
   return result;
}
    
private Properties getBaseQuartzProperties() {
   Properties result = new Properties();
   result.put("org.quartz.threadPool.class", org.quartz.simpl.SimpleThreadPool.class.getName());
   result.put("org.quartz.threadPool.threadCount", "1"); // Quartz 线程数：1
   result.put("org.quartz.scheduler.instanceName", liteJobConfig.getJobName());
   result.put("org.quartz.jobStore.misfireThreshold", "1");
   result.put("org.quartz.plugin.shutdownhook.class", JobShutdownHookPlugin.class.getName()); // 作业关闭钩子
   result.put("org.quartz.plugin.shutdownhook.cleanShutdown", Boolean.TRUE.toString()); // 关闭时，清理所有资源
   return result;
}

```

org.quartz.threadPool.threadCount = 1，即 Quartz 执行作业线程数量为 1。原因：一个作业( ElasticJob )的调度，需要配置独有的一个作业调度器( JobScheduler )，两者是 1 : 1 的关系。
org.quartz.plugin.shutdownhook.class 设置作业优雅关闭钩子：JobShutdownHookPlugin,删除主节点，关闭运行状态操作。

- 调用 #createJobDetail() 方法创建 Quartz 作业：

``` java
// JobScheduler.java
private JobDetail createJobDetail(final String jobClass) {
   // 创建 Quartz 作业
   JobDetail result = JobBuilder.newJob(LiteJob.class).withIdentity(liteJobConfig.getJobName()).build();
   //
   result.getJobDataMap().put(JOB_FACADE_DATA_MAP_KEY, jobFacade);
   // 创建 Elastic-Job 对象
   Optional<ElasticJob> elasticJobInstance = createElasticJobInstance();
   if (elasticJobInstance.isPresent()) {
       result.getJobDataMap().put(ELASTIC_JOB_DATA_MAP_KEY, elasticJobInstance.get());
   } else if (!jobClass.equals(ScriptJob.class.getCanonicalName())) {
       try {
           result.getJobDataMap().put(ELASTIC_JOB_DATA_MAP_KEY, Class.forName(jobClass).newInstance());
       } catch (final ReflectiveOperationException ex) {
           throw new JobConfigurationException("Elastic-Job: Job class '%s' can not initialize.", jobClass);
       }
   }
   return result;
}
    
protected Optional<ElasticJob> createElasticJobInstance() {
   return Optional.absent();
}
    
// SpringJobScheduler.java
@Override
protected Optional<ElasticJob> createElasticJobInstance() {
   return Optional.fromNullable(elasticJob);
}

```

## 注册作业启动信息
``` java
    /**
     * 注册作业启动信息.
     * 
     * @param enabled 作业是否启用
     */
    public void registerStartUpInfo(final boolean enabled) {
         // 开启 所有监听器
           listenerManager.startAllListeners();
           // 选举 主节点
           leaderService.electLeader();
           // 持久化 作业服务器上线信息
           serverService.persistOnline(enabled);
           // 持久化 作业运行实例上线相关信息
           instanceService.persistOnline();
           // 设置 需要重新分片的标记
           shardingService.setReshardingFlag();
           // 初始化 作业监听服务
           monitorService.listen();
           // 初始化 调解作业不一致状态服务
           if (!reconcileService.isRunning()) {
               reconcileService.startAsync();
           }
    }
```

## 调度作业

``` java
    // JobScheduler.java
    public void init() {
       // .... 省略部分代码
       // 调度作业
       jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
    }
    
    // JobScheduleController.java
    /**
    * 调度作业.
    *
    * @param cron CRON表达式
    */
    public void scheduleJob(final String cron) {
       try {
           if (!scheduler.checkExists(jobDetail.getKey())) {
               scheduler.scheduleJob(jobDetail, createTrigger(cron));
           }
           scheduler.start();
       } catch (final SchedulerException ex) {
           throw new JobSystemException(ex);
       }
    }
```

# 总结
- 至此，一个任务的初始化任务完成。

- 优点，任务流程清晰，为后面的任务调度，选主，分片，zk监听等功能提供了很好的支持。

- 疑问点，看第一个导图，在一个任务创建了太多的重复类，都是内部new出对象。我理解框架不想耦合当前的ioc的框架做出的妥协，但是可否自实现一些简单的功能。

- 建议，其实在阅读dubbo源码时，dubbo当时也存在相同的问题。dubbo的SPI个人觉得值得学习，可否改造出一版elastic-job版本的SPI，CoordinatorRegistryCenter regCenter或许就不用全框架内传递了。