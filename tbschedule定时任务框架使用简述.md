---
title: tbschedule定时任务框架使用简述 
date: 2016-11-30 21:32:21
categories: 框架
tags: [tbschedule,定时调度]
---

tbschedule是淘宝开源的定时任务框架，提供强大的任务切分支持，支撑起分布式定时任务执行。其特点是轻量级、任务切分简单、配置灵活。现简述下框架的安装过程以及定时任务的配置执行。

### 配置单节点的zookeeper，用于注册服务
OS：centos
- 下载
wget http://apache.fayea.com/zookeeper/stable/zookeeper-3.4.6.tar.gz
- 设置配置文件，在conf目录下拷贝zoo_sample.cfg为zoo.cfg。
cp zoo_sample.cfg zoo.cfg
- 进入bin目录，启动zk
./zkServer.sh start
- 客户端连接，测试服务成功开启
./zkCli.sh -server localhost:2181
若出现如下日志，说明启动成功
![](/images/image001.png)

### 安装TBSchedule控制台
- 下载TBSchedule源码包，将console目录下war包放入tomcat容器
- 启动tomcat，访问http://yourhost:8080/ScheduleConsole/
出现如下表格
![](/images/image002.png)
Zk没设账户的后两项随意填写
完事后点保存，错误信息毋需理会
- 访问http://yourhost:8080/ScheduleConsole/schedule/?manager=true，进入控制台
![](/images/image003.png)

### 客户端任务配置
- 首先写一个我们要执行的任务，实现IScheduleTaskDealSingle接口
```java
public class Task implements IScheduleTaskDealSingle<String> {
 
    public List<String> selectTasks(String taskParameter, String ownSign, int taskItemNum, List<TaskItemDefine> taskItemList, int eachFetchDataNum, int pageNum) throws Exception {
       List<String> list = new ArrayList<String>();
       System.out.println("taskParameter: " + taskParameter);
       System.out.println("ownSign: " + ownSign);
       for(TaskItemDefine taskItemDefine: taskItemList) {
           System.out.println("task item id: " + taskItemDefine.getTaskItemId());
           list.add(taskItemDefine.getTaskItemId() + "00");
           list.add(taskItemDefine.getTaskItemId() + "25");
           list.add(taskItemDefine.getTaskItemId() + "50");
           list.add(taskItemDefine.getTaskItemId() + "75");
       }
       System.out.println("eachFetchDataNum: " + eachFetchDataNum);
       return list;
    }
 
    public Comparator<String> getComparator() {
       // TODO Auto-generated method stub
       return null;
    }
 
    public boolean execute(String task, String ownSign) throws Exception {
       System.out.println("task offset: " + Integer.parseInt(task));
       return false;
    }
}
```
selectTasks方法负责从控制台获取配置的任务参数，其中最主要的参数taskItemList指该线程组所分得的任务项

Execute方法用于执行具体的任务，由该线程组中的线程调用。

IScheduleTaskDealSingle接口仅支持该方法每次执行一个任务，若实现IScheduleTaskDealMulti则可进行任务的批处理

上述Task类实例由Spring加载

### 控制台配置说明
- 策略配置
![](/images/image005.png)
任务类型默认选择Schedule，经分析源码，后两项目前没有作用
任务名称与后面将配置的具体任务名一致
任务参数由用户根据业务需要自拟
单JVM最大线程组数目：指单台机器所能持有的最大线程组数目
最大线程数目：总共的线程组数目，将由各台机器分担

- 任务配置
![](/images/image007.png)
任务名称自拟，与策略配置中任务名称一致

任务处理的SpingBean：与代码中sping注入的任务bean的ID一致

线程数：设置线程组内线程的数目

处理模式：默认SLEEP模式，该模式下某一线程处理完任务后，若发现还存在其他的活动线程，则自身休眠，否则将调用selectTasks方法获取新的任务，并唤醒其余线程开始执行；而NOTSLEEP模式则体现在线程执行完任务后若发现任务队列为空，则将直接调用selectTasks获取任务放入任务队列

每次执行数量：该值在实现IScheduleTaskDealSingle接口时只能为1，IScheduleTaskDealMulti接口时可任意设置，此时execute方法可传入task数组进行批量处理

执行开始时间与结束时间遵循Crontab时间格式

单线程组最大任务数：单个线程能够承载的最大任务数，可有效防止当机器删除时造成剩余机器任务数的无限制上涨


以上内容介绍了tbschedule框架的简单搭建和应用，并简要介绍了策略与任务配置的参数。tbschedule以其对代码较低的侵入性和简洁的编码，实现复杂的分布式定时任务控制流程。但其不足在于任务控制功能相对简陋，如无法提供任务执行时间的监控，无法中止任务，无法立即执行任务，无法获知任务异常等。