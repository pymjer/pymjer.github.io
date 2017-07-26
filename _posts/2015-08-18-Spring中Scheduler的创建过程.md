---
layout:     post
title:      "Spring中Quartz的Scheduler创建过程"
subtitle:   "跟踪Spring中Scheduler创建过程"
date:       2015-08-18
author:     "Pymjer"
tags: ["spring", "quartz"]
---
## Scheduler配置
在Spring中，如果我们有定时的调度任务，通常配置的Scheduler是这个样子

    <bean id="scheduler" autowire="no" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="schedulerName" value="凭证中间库数据加载" />
        <property name="triggers">
          <list>
            <ref bean="fetchOfflinePaymentsJobTrigger" />
          </list>
        </property>
        <property name="autoStartup" value="${job.autoStartup}" />
    </bean>

 * FactoryBean 是Spring集成第三方框架常用的招数，Spring在集成JPA,Mybatis使用的是相同的手段
 * dataSource 是可选的，当你不需要持久化任务时可以不用指定

## Scheduler 的创建

下面来看一下SchedulerFactoryBean 类是如何创建Scheduler对象的

SchedulerFactoryBean 没有构造函数，初始化改对象后，spring会调用对象的 afterPropertiesSet 方法，该方法中除了一些基本的数据初始化外，最重要的是下面几行代码

    this.scheduler = createScheduler(schedulerFactory, this.schedulerName);
    populateSchedulerContext();

    if (!this.jobFactorySet && !(this.scheduler instanceof RemoteScheduler)) {
        // Use AdaptableJobFactory as default for a local Scheduler, unless when
        // explicitly given a null value through the "jobFactory" bean property.
        this.jobFactory = new AdaptableJobFactory();
    }
    if (this.jobFactory != null) {
        if (this.jobFactory instanceof SchedulerContextAware) {
            ((SchedulerContextAware) this.jobFactory).setSchedulerContext(this.scheduler.getContext());
        }
        this.scheduler.setJobFactory(this.jobFactory);
    }

通过调用createScheduler方法创建Scheduler对象，createScheduler 需要传入一个SchedulerFactory，该方法的核心代码如下

    SchedulerRepository repository = SchedulerRepository.getInstance();
    synchronized (repository) {
        Scheduler existingScheduler = (schedulerName != null ? repository.lookup(schedulerName) : null);
        Scheduler newScheduler = schedulerFactory.getScheduler();
        if (newScheduler == existingScheduler) {
            throw new IllegalStateException("Active Scheduler of name '" + schedulerName + "' already registered " +
            "in Quartz SchedulerRepository. Cannot create a new Spring-managed Scheduler of the same name!");
        }
        if (!this.exposeSchedulerInRepository) {
            // Need to remove it in this case, since Quartz shares the Scheduler instance by default!
            SchedulerRepository.getInstance().remove(newScheduler.getSchedulerName());
        }
        return newScheduler;
    }

在创建调度器时，需要加上repository的互斥锁来防止创建相同的调度器

在调度器创建完成后，通过调用 registerJobsAndTriggers() 方法来注册具体的任务

调度器的具体创建和初始化过程参见方法：`org.quartz.impl.StdSchedulerFactory#instantiate()`，顺便说一句，这个方法有800行！

## Scheduler 初始化
Scheduler 是一个接口，在quartz中有很多实现，继承关系如下

    Scheduler (org.quartz)
    ——|RemoteMBeanScheduler (org.quartz.impl)
        ——|JBoss4RMIRemoteMBeanScheduler (org.quartz.ee.jmx.jboss)
    ——|RemoteScheduler (org.quartz.impl)
    ——|StdScheduler (org.quartz.impl)

以StdScheduler为例，StdScheduler 本质是一个包装类，真正执行调度任务的是其内部持有一个 org.quartz.core.QuartzScheduler sched对象，下面重点分析一下QuartzScheduler 的初始化过程，factory在初始化QuartzScheduler时，会传入一个QuartzSchedulerResources rsrcs对象，这个对象负责管理调度的资源分配，参见如下代码：

    // 路径：org.quartz.impl.StdSchedulerFactory#instantiate()
    QuartzSchedulerResources rsrcs = new QuartzSchedulerResources();
    rsrcs.setName(schedName);
    rsrcs.setThreadName(threadName);
    rsrcs.setInstanceId(schedInstId);
    rsrcs.setJobRunShellFactory(jrsf);
    rsrcs.setMakeSchedulerThreadDaemon(makeSchedulerThreadDaemon);
    rsrcs.setThreadsInheritInitializersClassLoadContext(threadsInheritInitalizersClassLoader);
    rsrcs.setRunUpdateCheck(!skipUpdateCheck);
    rsrcs.setBatchTimeWindow(batchTimeWindow);
    rsrcs.setMaxBatchSize(maxBatchSize);
    rsrcs.setInterruptJobsOnShutdown(interruptJobsOnShutdown);
    rsrcs.setInterruptJobsOnShutdownWithWait(interruptJobsOnShutdownWithWait);
    rsrcs.setJMXExport(jmxExport);
    rsrcs.setJMXObjectName(jmxObjectName);
    if (rmiExport) {
        rsrcs.setRMIRegistryHost(rmiHost);
        rsrcs.setRMIRegistryPort(rmiPort);
        rsrcs.setRMIServerPort(rmiServerPort);
        rsrcs.setRMICreateRegistryStrategy(rmiCreateRegistry);
        rsrcs.setRMIBindName(rmiBindName);
    }

    .....
    qs = new QuartzScheduler(rsrcs, idleWaitTime, dbFailureRetry);

rsrcs 有三个核心组件需要初始化设置：

- ThreadPool
- JobStore
- ThreadExecutor

下面分析进行这三个组件的初始化流程

ThreadPool的初始化流程如下：

    // Get ThreadPool Properties
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    String tpClass = cfg.getStringProperty(PROP_THREAD_POOL_CLASS, SimpleThreadPool.class.getName());

    if (tpClass == null) {
        initException = new SchedulerException(
                "ThreadPool class not specified. ");
        throw initException;
    }

    try {
        tp = (ThreadPool) loadHelper.loadClass(tpClass).newInstance();
    } catch (Exception e) {
        initException = new SchedulerException("ThreadPool class '"
                + tpClass + "' could not be instantiated.", e);
        throw initException;
    }
    tProps = cfg.getPropertyGroup(PROP_THREAD_POOL_PREFIX, true);
    try {
        setBeanProps(tp, tProps);
    } catch (Exception e) {
        initException = new SchedulerException("ThreadPool class '"
                + tpClass + "' props could not be configured.", e);
        throw initException;
    }

> 从上面的代码可知，ThreadPool的初始化过程，读取参数org.quartz.threadPool.class 定义的线程池对象，如果没有，就用默认的SimpleThreadPool
> 上面的代码在获取到tp后，还给tp设置了所有`org.quartz.threadPool`前缀的属性

JobStore的初始化过程如下：

    // Get JobStore Properties
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    String jsClass = cfg.getStringProperty(PROP_JOB_STORE_CLASS,
            RAMJobStore.class.getName());

    if (jsClass == null) {
        initException = new SchedulerException(
                "JobStore class not specified. ");
        throw initException;
    }

    try {
        js = (JobStore) loadHelper.loadClass(jsClass).newInstance();
    } catch (Exception e) {
        initException = new SchedulerException("JobStore class '" + jsClass
                + "' could not be instantiated.", e);
        throw initException;
    }

    SchedulerDetailsSetter.setDetails(js, schedName, schedInstId);

    tProps = cfg.getPropertyGroup(PROP_JOB_STORE_PREFIX, true, new String[] {PROP_JOB_STORE_LOCK_HANDLER_PREFIX});
    try {
        setBeanProps(js, tProps);
    } catch (Exception e) {
        initException = new SchedulerException("JobStore class '" + jsClass
                + "' props could not be configured.", e);
        throw initException;
    }


> 读取参数org.quartz.jobStore.class 定义的任务库，如果没有，就用默认的RAMJobStore

ThreadExecutor的初始化过程如下：

    // Get ThreadExecutor Properties
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    String threadExecutorClass = cfg.getStringProperty(PROP_THREAD_EXECUTOR_CLASS);
    if (threadExecutorClass != null) {
        tProps = cfg.getPropertyGroup(PROP_THREAD_EXECUTOR, true);
        try {
            threadExecutor = (ThreadExecutor) loadHelper.loadClass(threadExecutorClass).newInstance();
            log.info("Using custom implementation for ThreadExecutor: " + threadExecutorClass);

            setBeanProps(threadExecutor, tProps);
        } catch (Exception e) {
            initException = new SchedulerException(
                    "ThreadExecutor class '" + threadExecutorClass + "' could not be instantiated.", e);
            throw initException;
        }
    } else {
        log.info("Using default implementation for ThreadExecutor");
        threadExecutor = new DefaultThreadExecutor();
    }

> 读取参数`org.quartz.threadExecutor.class`,如果没有则默认是用DefaultThreadExecutor

## scheduler 的启动

在调用scheduler.start() 方法启动调度时，实际是调用sched.start();

    public void start() throws SchedulerException {
        sched.start();
    }

在sched start方法中，执行如下代码，启动任务

    if (initialStart == null) {
        initialStart = new Date();
        this.resources.getJobStore().schedulerStarted();
        startPlugins();
    }


