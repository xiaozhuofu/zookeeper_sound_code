# 研究zookeeper源码
【核心目的】：研究服务端的工作原理
# 一、搭建zookeeper的源码环境
【zookeeper版本】3.5.4  
采用ant的方式来构建项目，需要先安装ant，然后再借助ant工具构建zookeeper项目
## 1.安装Ant
网址：http://ant.apache.org/bindownload.cgi
### 1.1 下载
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/ant-version.png)
### 1.2 解压到指定文件夹
### 1.3 配置path环境变量
ant的解压目录\bin
### 1.4 验证
验证ant是否安装成功，cmd窗口，输入ant-version  
![](/images/build-environment/ant-version-verify.png)
## 2.下载zookeeper源码
网址：https://github.com/apache/zookeeper/tree/release-3.5.4
## 3.构建zookeeper源码
下载源码后，解压到指定目录，借助ant构建完整的zookeeper项目，过程需要些时间
```jshelllanguage
# 构建成功关键词：BUILD SUCCESS
ant eclipse  
```
## 4.将zookeeper源码导入IDEA
操作路径如下，之后一路往下点，本次JDK版本选择1.8  
![](/images/build-environment/import-idea-01.png)  
![](/images/build-environment/import-idea-02.png)
## 5.配置启动服务的相关参数
### 5.1 命名zoo.cfg文件
在conf目录下，将zoo.sample.cfg复制一份，为zoo.cfg  
![](/images/build-environment/start-project-01.png)
### 5.2 服务启动类
看zk服务端的启动流程，所以查看bin目录下的zkServer.sh  
![](/images/build-environment/start-project-02.png)  
其中可以发现，服务的执行入口
```java
org.apache.zookeeper.server.quorum.QuorumPeerMain
```
其通过执行main方法来实现服务的启动，需要给这个方法通过参数传递配置文件的路径（zoo.cfg）  
(1)准备zoo.cfg的路径：xxx\zookeeper-release-3.5.4\conf\zoo.cfg  
(2)准备log4j.properties的路径：xxx\zookeeper-release-3.5.4\conf\log4j.properties  
![](/images/build-environment/start-project-03.png) 
### 5.3 配置服务启动相关参数
```jshelllanguage
# 服务主类[Main class]
org.apache.zookeeper.server.quorum.QuorumPeerMain

# 日志文件的路径[VM options]
-Dlog4j.configuration=file:xxx\zookeeper-release-3.5.4\conf\log4j.properties

# 配置文件的路径[Program arguments]
xxx\zookeeper-release-3.5.4\conf\zoo.cfg

# 项目的工作目录[Working directory]
xxx\huangguizhao\zookeeper-release-3.5.4
```
![](/images/build-environment/start-project-04.png) 

# 二、zookeeper服务器启动初始化流程-源码解析
## 1.同意启动入口
不管是单机版还是集群版，这个类都是统一的启动入口：org.apache.zookeeper.server.quorum.QuorumPeerMain  
![](/images/server-init/QuorumPeerMain.png)   
内部都是采用main方法来执行
```java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        //真正的核心执行方法
        //初始化并启动服务
        main.initializeAndRun(args);
    } catch (IllegalArgumentException e) {
        //忽略非核心代码
    } 
}
```
## 2.核心执行方法解析-确定流程主脉络
```java
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        //1.解析配置文件zoo.cfg
        config.parse(args[0]);
    }
    // Start and schedule the the purge task[启动并定时安排清除任务]
    //2.启动定时的清理任务，由DatadirCleanupManager对象来实现
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                    .getDataDir(), config.getDataLogDir(), config
                    .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();
    //3.通过读取配置文件，判断是单机模式还是集群模式
    //config.isDistributed()：集群模式返回true
    if (args.length == 1 && config.isDistributed()) {
        //3.1 走集群模式的路径
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running "
                            + " in standalone mode");
        // there is only server in the quorum -- run as standalone
        //3.2 走单机模式的路径
        ZooKeeperServerMain.main(args);
    }
}
```
![](/images/server-init/initializeAndRun.png)   
## 3.DatadirCleanupManager-文件清理器
### 3.1 内部关键属性
1、原由  
zookeeper内部管理的数据分两块，一块是内存中的数据，一块是磁盘中的数据（快照，日志）
随着服务器的运行时间越来越长，那么这些磁盘的历史文件也会越来越多，所以需要采用定时清理
2、解决方案
zookeeper内部采用DatadirCleanupManager来实现文件的定期清理
3、源码关键说明  
![](/images/server-init/DatadirCleanupManager.png)    
```java
public class DatadirCleanupManager {
    //忽略
    
    //快照的地址
    private final File snapDir;

    //日志的地址
    private final File dataLogDir;

    //指定需要保留的文件的个数
    private final int snapRetainCount;

    //指定清除周期，以小时为单位
    //默认是0 , 表示不开启启动清理功能
    //需要开启，则在zoo.cfg配置即可，如下：【需要放开就将autopurge.purgeInterval=1注释去掉】
    //# Purge task interval in hours
    //# Set to "0" to disable auto purge feature
    //# autopurge.purgeInterval=1
    private final int purgeInterval;

    //定时器对象
    private Timer timer;
    
    //忽略
}
```
### 3.2 实现定期清理
```java
public void start() {
    //忽略非核心代码
    
    //1.初始化定时器对象，取名并且以后台线程的方式存在
    timer = new Timer("PurgeTask", true);
    //2.创建清理任务对象[PurgeTask间接实现Runnable接口]
    TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
    //3.设置清理任务的定时执行周期[后台线程执行task任务]
    timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));
    //4.设置任务的状态
    purgeTaskStatus = PurgeTaskStatus.STARTED;
}

//PurgeTask间接实现Runnable接口
static class PurgeTask extends TimerTask {
    //忽略非核心代码
    @Override
    public void run() {
        LOG.info("Purge task started.");
        try {
            //PurgeTxnLog完成真正的日志清理
            PurgeTxnLog.purge(logsDir, snapsDir, snapRetainCount);
        } catch (Exception e) {
            LOG.error("Error occurred while purging.", e);
        }
        LOG.info("Purge task completed.");
    }
}

public abstract class TimerTask implements Runnable {}
```
![](/images/server-init/DatadirCleanupManager-02.png) 