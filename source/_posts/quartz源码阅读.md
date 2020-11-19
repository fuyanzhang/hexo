---
title: quartz源码阅读
date: 2020-07-17 18:38:23
tags:
  - java
  - 源码阅读
  - 定时任务
categories:
  - 源码阅读
  - 技术
---

## 前面的话

这里只对 quartz 的源码做一个整体的梳理，关于 quartz 的整体结构，百度 Google 之，一堆一堆的。

## 具体阅读

quartz 中主要围绕 3 个东东搞各种逻辑。分别是调度器（Scheduler），触发器（trigger）和任务（job）。调度器去获取触发器，触发器指定任务的调度时间，调度策略，调度状态，优先级，开始时间，结束时间等信息。任务就是具体的业务逻辑实现。

<!--more-->

### 一个栗子进入代码

```
        SchedulerFactory factory = new StdSchedulerFactory("test_quartz.properties");
        Scheduler scheduler = factory.getScheduler();
        scheduler.start();
        Trigger t = newTrigger().withIdentity("t1","g1").startAt(new Date(1466746025000l)).withSchedule(simpleSchedule().withMisfireHandlingInstructionNextWithRemainingCount().withRepeatCount(0)).build();
        JobDetail job = newJob(TestJob.class).withIdentity("myJob1", "g1").build();
        scheduler.scheduleJob(job,t);
    }
```

```
public class TestJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        try {
            Thread.sleep(20000);
            System.out.println(context.getTrigger().getKey()+"执行成功！！！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

上面两段代码就是一个简单的任务的写法。
主要过程如下：
1、首先通过调度器工厂获取一个调度器。启动调度器。
2、定义触发器。
3、定义任务。
4、通过调度器将触发器和任务关联起来。
首先来看下调度器的初始化。
调度器工厂初始化主要是读取配置信息。通过 getScheduler 方法才是真正的初始化 scheduler，里边主要是通过配置信息组装 scheduler。这里不是重点，一笔带过。【注：在调度器组装的时候，顺便启动了任务的执行线程
` qs = new QuartzScheduler(rsrcs, idleWaitTime, dbFailureRetry);`

```
public QuartzScheduler(QuartzSchedulerResources resources, long idleWaitTime, @Deprecated long dbRetryInterval)
        throws SchedulerException {
        this.resources = resources;
        if (resources.getJobStore() instanceof JobListener) {
            addInternalJobListener((JobListener)resources.getJobStore());
        }

        this.schedThread = new QuartzSchedulerThread(this, resources);
        ThreadExecutor schedThreadExecutor = resources.getThreadExecutor();
        schedThreadExecutor.execute(this.schedThread);
```

，只是在线程启动后一直等待，知道调度器调用 start 方法

```
while (paused && !halted.get()) {
                      try {
                          // wait until togglePause(false) is called...
                          sigLock.wait(1000L);
                      } catch (InterruptedException ignore) {
                      }
```

】

下面试调度器的启动。

### 调度器启动过程

quartz 支持集群模式下的任务调度。任务持久化采用 DB 的方式。
这里主要涉及集群模式下的任务执行过程。
启动过程代码如下：

```
 public void start() throws SchedulerException {
        if (shuttingDown|| closed) {
            throw new SchedulerException(
                    "The Scheduler cannot be restarted after shutdown() has been called.");
        }
        notifySchedulerListenersStarting();
        if (initialStart == null) {
            initialStart = new Date();
    //调度器第一次启动
            this.resources.getJobStore().schedulerStarted();
            startPlugins();
        } else {

            resources.getJobStore().schedulerResumed();
        }
    //将执行线程唤醒。用于获取触发器，触发任务
        schedThread.togglePause(false);
        getLog().info(
                "Scheduler " + resources.getUniqueIdentifier() + " started.");
        notifySchedulerListenersStarted();
    }
```

```
public void schedulerStarted() throws SchedulerException {

//如果是集群，启动集群的管理线程，定时检查集群的健康性。
        if (isClustered()) {
            clusterManagementThread = new ClusterManager();
            if(initializersLoader != null)
                clusterManagementThread.setContextClassLoader(initializersLoader);
            clusterManagementThread.initialize();
        } else {
            try {
  //非集群，则恢复调度器宕机前的任务
                recoverJobs();
            } catch (SchedulerException se) {
                throw new SchedulerConfigException(
                        "Failure occured during job recovery.", se);
            }
        }
//起线程，用于检查是否有任务错过执行时间，若有，则根据不同的策略修改不同的nextfiretime值，以便于工作线程去选择trigger。
        misfireHandler = new MisfireHandler();
        if(initializersLoader != null)
            misfireHandler.setContextClassLoader(initializersLoader);
        misfireHandler.initialize();
        schedulerRunning = true;

        getLog().debug("JobStore background threads started (as scheduler was started).");
    }
```

下面深入对三大线程做讲解。QuartzSchedulerThread，ClusterManager 和 MisfireHandler。

#### QuartzSchedulerThread

这个线程是 quartz 的主要线程，负责调度的。看下代码：
run 方法很长，这里选取主要的代码。

```
  public void run() {
        int acquiresFailed = 0;
        while (!halted.get()) {
            try {
                synchronized (sigLock) {
                    while (paused && !halted.get()) {
                        try {
                           //scheduler调用start方法前，在此处循环，直到start方法里调用了togglePause方法。
                            sigLock.wait(1000L);
                        } catch (InterruptedException ignore) {
                        }
。。。
                int availThreadCount = qsRsrcs.getThreadPool().blockForAvailableThreads();
                if(availThreadCount > 0) { // will always be true, due to semantics of blockForAvailableThreads...
                    try {
                      //获取当前可以被触发的trigger
                        triggers = qsRsrcs.getJobStore().acquireNextTriggers(
                                now + idleWaitTime, Math.min(availThreadCount, qsRsrcs.getMaxBatchSize()), qsRsrcs.getBatchTimeWindow());
                        acquiresFailed = 0;
                     。。。

                        List<TriggerFiredResult> bndles = new ArrayList<TriggerFiredResult>();
                        if(goAhead) {
                            try {
                              //触发trigger，主要是修改qrtz_trigger表中trigger的状态从acquired状态变成WATTING或者complete状态。并计算下一次执行时间，等待下一次被选中。同时修改QRTZ_FIRED_TRIGGERS中trigger状态为executing状态。
                                List<TriggerFiredResult> res = qsRsrcs.getJobStore().triggersFired(triggers);
                                if(res != null)
                                    bndles = res;
。。。
                            JobRunShell shell = null;
                            try {
                                shell = qsRsrcs.getJobRunShellFactory().createJobRunShell(bndle);
                                shell.initialize(qs);
                            } catch (SchedulerException se) {
                                qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
                                continue;
                            }
                            //真正的到了调用job的execute的地方了，该方法执行完成之后，本次调度就真正完成了。
                            if (qsRsrcs.getThreadPool().runInThread(shell) == false) {
                                // this case should never happen, as it is indicative of the
                                // scheduler being shutdown or a bug in the thread pool or
                                // a thread pool being used concurrently - which the docs
                                // say not to do...
                                getLog().error("ThreadPool.runInThread() return false!");
                                qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
                            }

                        }

                        continue; // while (!halted)
 //后面就是该线程等待一段时间，用于其他节点来调度任务。
    }
```

总结起来，上面代码的逻辑如下：
1、启动时，该线程一直都在等待，知道有调用 scheudler 的 start 方法，开始唤醒该线程。
2、查看当前任务的处理线程池里空闲线程的个数，然后去 qrtz_triggers 表中获取可以处理的 trigger，并将 trigger 的状态改为 acquired，同时插入表 qrtz_fired_triggers,此时 qrtz_fired_triggers 表中 trigger 的状态也为 acquired。
3、获取到待执行的 trigger，由于取的是时间窗里的 trigger，所以，从待执行的 trigger 列表中取第一个 trigger（trigger 列表是按照 next_fire_time 升序排列），与当前时间比较，如果大于 2s，则等待。
4、等到第一个 trigger 的任务到了，则去 qrtz_triggers 表中再次确认获取到的 trigger 的状态是否为 aquired，若是，则修改 qrtz_fired_triggers 状态为 executing。同时，qrtz_triggers 中的状态在本次调度时已经走到尽头，可以等待下一次的调度了。即，计算下一次的调度时间，将并将任务状态改为 watting 状态。若计算得到的下一次调度时间为 null，则表明该任务已经执行完成。将任务改为 complete 状态。返回待本次调度的 trigger。
5、循环 trigger，获取任务执行线程，执行任务的 execute 方法。
6、改调度线程 wait 一段时间，等待下一次获取 trigger，调度。
接下来看下真正调度的线程 JobRunShell，同样，很长的 run 方法，这里只摘取部分代码：

```
 public void run() {
        qs.addInternalSchedulerListener(this);
        try {
            OperableTrigger trigger = (OperableTrigger) jec.getTrigger();
            JobDetail jobDetail = jec.getJobDetail();
            do {
                JobExecutionException jobExEx = null;
                Job job = jec.getJobInstance();
......
                long startTime = System.currentTimeMillis();
                long endTime = startTime;
                try {
                    job.execute(jec);
                    endTime = System.currentTimeMillis();
                } catch (JobExecutionException jee) {
                    endTime = System.currentTimeMillis();
                    jobExEx = jee;
                    getLog().info("Job " + jobDetail.getKey() +
                            " threw a JobExecutionException: ", jobExEx);
                } catch (Throwable e) {
                    endTime = System.currentTimeMillis();
                    getLog().error("Job " + jobDetail.getKey() +
                            " threw an unhandled Exception: ", e);
                    SchedulerException se = new SchedulerException(
                            "Job threw an unhandled exception.", e);
                    qs.notifySchedulerListenersError("Job ("
                            + jec.getJobDetail().getKey()
                            + " threw an exception.", se);
                    jobExEx = new JobExecutionException(se, false);
                }

                jec.setJobRunTime(endTime - startTime);
......
                CompletedExecutionInstruction instCode = CompletedExecutionInstruction.NOOP;
                try {
                    instCode = trigger.executionComplete(jec, jobExEx);
                } catch (Exception e) {
                    // If this happens, there's a bug in the trigger...
                    SchedulerException se = new SchedulerException(
                            "Trigger threw an unhandled exception.", e);
                    qs.notifySchedulerListenersError(
                            "Please report this error to the Quartz developers.",
                            se);
                }
                if (instCode == CompletedExecutionInstruction.RE_EXECUTE_JOB) {
                    jec.incrementRefireCount();
                    try {
                        complete(false);
                    } catch (SchedulerException se) {
                        qs.notifySchedulerListenersError("Error executing Job ("
                                + jec.getJobDetail().getKey()
                                + ": couldn't finalize execution.", se);
                    }
                    continue;
                }

                try {
                    complete(true);
                } catch (SchedulerException se) {
                    qs.notifySchedulerListenersError("Error executing Job ("
                            + jec.getJobDetail().getKey()
                            + ": couldn't finalize execution.", se);
                    continue;
                }

                qs.notifyJobStoreJobComplete(trigger, jobDetail, instCode);
                break;
            } while (true);

        } finally {
            qs.removeInternalSchedulerListener(this);
        }
    }
```

上述代码的逻辑很简单，就是获取 job 并执行 job 的 execute 方法。执行完成之后，通过不同的返回码，进行不同的数据库操作。 `qs.notifyJobStoreJobComplete(trigger, jobDetail, instCode);`这句话就是通过不同的返回值做不同的数据库操作。主要是修改 qrtz_triggers 里的 trigger 状态及某些场景下删除 trigger。然后是删除 qrtz_fired_triggers 里的当前 trigger。
到此，正常的任务调度完成了。当然其中很多步骤里都调用了 SchedulerListener，TriggerListener 中的一些方法，这些是 quartz 开放出来的定制接口，方便每步操作时，我们对任务的监控。
![状态转换](https://upload-images.jianshu.io/upload_images/11942209-865598eb7cbf6f1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 接下来看 misfired 的进程 MisfireHandler。

Misfirehandler 是一个内部类。run 接口 代码如下：

```
  public void run() {

            while (!shutdown) {

                long sTime = System.currentTimeMillis();

//获取misfired的job
                RecoverMisfiredJobsResult recoverMisfiredJobsResult = manage();

// 如果任务处理线程在等待下一次的扫描满足的trigger，则唤醒线程，来处理misfired的任务
                if (recoverMisfiredJobsResult.getProcessedMisfiredTriggerCount() > 0) {
                    signalSchedulingChangeImmediately(recoverMisfiredJobsResult.getEarliestNewTime());
                }

                if (!shutdown) {
                    long timeToSleep = 50l;  // At least a short pause to help balance threads
                    if (!recoverMisfiredJobsResult.hasMoreMisfiredTriggers()) {
                        timeToSleep = getMisfireThreshold() - (System.currentTimeMillis() - sTime);
                        if (timeToSleep <= 0) {
                            timeToSleep = 50l;
                        }

                        if(numFails > 0) {
                            timeToSleep = Math.max(getDbRetryInterval(), timeToSleep);
                        }
                    }

                    try {
                        Thread.sleep(timeToSleep);
                    } catch (Exception ignore) {
                    }
                }//while !shutdown
            }
        }
```

主要逻辑在 manager 方法里。

```
 private RecoverMisfiredJobsResult manage() {
            try {
                getLog().debug("MisfireHandler: scanning for misfires...");

                RecoverMisfiredJobsResult res = doRecoverMisfires();
                numFails = 0;
                return res;
            } catch (Exception e) {
                if(numFails % 4 == 0) {
                    getLog().error(
                        "MisfireHandler: Error handling misfires: "
                                + e.getMessage(), e);
                }
                numFails++;
            }
            return RecoverMisfiredJobsResult.NO_OP;
        }
```

主要的方法是 doRecoverMisfires()。

```
protected RecoverMisfiredJobsResult doRecoverMisfires() throws JobPersistenceException {
        boolean transOwner = false;
        Connection conn = getNonManagedTXConnection();
        try {
            RecoverMisfiredJobsResult result = RecoverMisfiredJobsResult.NO_OP;

            // Before we make the potentially expensive call to acquire the
            // trigger lock, peek ahead to see if it is likely we would find
            // misfired triggers requiring recovery.
            int misfireCount = (getDoubleCheckLockMisfireHandler()) ?
                getDelegate().countMisfiredTriggersInState(
                    conn, STATE_WAITING, getMisfireTime()) :
                Integer.MAX_VALUE;

            if (misfireCount == 0) {
                getLog().debug(
                    "Found 0 triggers that missed their scheduled fire-time.");
            } else {
                transOwner = getLockHandler().obtainLock(conn, LOCK_TRIGGER_ACCESS);
                //修改misfired的next fired time ，等待任务选取线程去调度
                result = recoverMisfiredJobs(conn, false);
            }

            commitConnection(conn);
            return result;
```

` int misfireCount = (getDoubleCheckLockMisfireHandler()) ? getDelegate().countMisfiredTriggersInState( conn,STATE_WAITING,getMisfireTime()) :`
获取 misfired 的 trigger，执行的查询为`select count(TRIGGER_NAME) from QRTZ_TRIGGERS where SCHED_NAME=xxx and not (MISFIRE_INSTR = -1 ) and NEXT_FIRE_TIME < 当前时间 and TRIGGER_STATE='STATE_WAITING'`即选取当前调度器的 misfired 的策略不为-1 的，且下一次执行时间小于当前时间的且状态为 waiting 的 trigger。
` result = recoverMisfiredJobs(conn, false);`真正获取 misfired 的 job 的时候了。

```
protected RecoverMisfiredJobsResult recoverMisfiredJobs(
        Connection conn, boolean recovering)
        throws JobPersistenceException, SQLException {

        // If recovering, we want to handle all of the misfired
        // triggers right away.
        int maxMisfiresToHandleAtATime =
            (recovering) ? -1 : getMaxMisfiresToHandleAtATime();

        List<TriggerKey> misfiredTriggers = new LinkedList<TriggerKey>();
        long earliestNewTime = Long.MAX_VALUE;
        // We must still look for the MISFIRED state in case triggers were left
        // in this state when upgrading to this version that does not support it.
        boolean hasMoreMisfiredTriggers =
            getDelegate().hasMisfiredTriggersInState(
                conn, STATE_WAITING, getMisfireTime(),
                maxMisfiresToHandleAtATime, misfiredTriggers);

        if (hasMoreMisfiredTriggers) {
            getLog().info(
                "Handling the first " + misfiredTriggers.size() +
                " triggers that missed their scheduled fire-time.  " +
                "More misfired triggers remain to be processed.");
        } else if (misfiredTriggers.size() > 0) {
            getLog().info(
                "Handling " + misfiredTriggers.size() +
                " trigger(s) that missed their scheduled fire-time.");
        } else {
            getLog().debug(
                "Found 0 triggers that missed their scheduled fire-time.");
            return RecoverMisfiredJobsResult.NO_OP;
        }

        for (TriggerKey triggerKey: misfiredTriggers) {

            OperableTrigger trig =
                retrieveTrigger(conn, triggerKey);

            if (trig == null) {
                continue;
            }

            doUpdateOfMisfiredTrigger(conn, trig, false, STATE_WAITING, recovering);

            if(trig.getNextFireTime() != null && trig.getNextFireTime().getTime() < earliestNewTime)
                earliestNewTime = trig.getNextFireTime().getTime();
        }

        return new RecoverMisfiredJobsResult(
                hasMoreMisfiredTriggers, misfiredTriggers.size(), earliestNewTime);
    }
```

每次获取 misfired 的 trigger 有一定的数量，默认 20 个，超过 20 个，则会在下一次去获取。
处理 misfired 的 trigger` doUpdateOfMisfiredTrigger(conn, trig, false, STATE_WAITING, recovering);`

```
 private void doUpdateOfMisfiredTrigger(Connection conn, OperableTrigger trig, boolean forceState, String newStateIfNotComplete, boolean recovering) throws JobPersistenceException {
        Calendar cal = null;
        if (trig.getCalendarName() != null) {
            cal = retrieveCalendar(conn, trig.getCalendarName());
        }

//触发triggerlistener的misfired方法。
        schedSignaler.notifyTriggerListenersMisfired(trig);

//根据不同的misfired的策略计算next_fired_time。【比较的绕，下次详细介绍】
        trig.updateAfterMisfire(cal);

        if (trig.getNextFireTime() == null) {
//如果下一次执行的时间为空，则认为执行的任务已经完成了。直接修改状态的state_complete
            storeTrigger(conn, trig,
                null, true, STATE_COMPLETE, forceState, recovering);
            schedSignaler.notifySchedulerListenersFinalized(trig);
        } else {
//修改任务的下一次执行时间和任务状态，等待调度线程去调度
            storeTrigger(conn, trig, null, true, newStateIfNotComplete,
                    forceState, recovering);
        }
    }
```

执行完上面的代码，提交了数据库事物后，任务就可以被正常调度了。到这里，misfired 的任务大体的也完成。
然后就是回到 manager 方法了。唤醒调度线程，至此，misfired 的本次扫描全部完成，接下来的事情就交给`QuartzSchedulerThread`来处理了。
写了好几天，终于写完了两个线程的处理过程。接下来的一篇主要介绍任务节点在 down 机的时候的处理及不同情况下 trigger 的 next_fire_time 的计算。
