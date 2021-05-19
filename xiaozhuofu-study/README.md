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
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/ant-version-verify.png)
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
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/import-idea-01.png)  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/import-idea-02.png)
## 5.配置启动服务的相关参数
### 5.1 命名zoo.cfg文件
在conf目录下，将zoo.sample.cfg复制一份，为zoo.cfg  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/start-project-01.png)
### 5.2 服务启动类
看zk服务端的启动流程，所以查看bin目录下的zkServer.sh  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/start-project-02.png)  
其中可以发现，服务的执行入口
```java
org.apache.zookeeper.server.quorum.QuorumPeerMain
```
其通过执行main方法来实现服务的启动，需要给这个方法通过参数传递配置文件的路径（zoo.cfg）  
(1)准备zoo.cfg的路径：xxx\zookeeper-release-3.5.4\conf\zoo.cfg  
(2)准备log4j.properties的路径：xxx\zookeeper-release-3.5.4\conf\log4j.properties  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/start-project-03.png) 
### 5.3 配置服务启动相关参数
```jshelllanguage
# 服务主类[Main class]
org.apache.zookeeper.server.quorum.QuorumPeerMain

# 日志文件的路径[VM options]
-Dlog4j.configuration=file:xxx\zookeeper-release-3.5.4\conf\log4j.properties

# 配置文件的路径[Program arguments]
xxx\zookeeper-release-3.5.4\conf\zoo.cfg

# 项目的工作目录[Working directory]
xxx\zookeeper-release-3.5.4
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/build-environment/start-project-04.png) 

# 二、zookeeper服务器启动初始化流程-源码解析
## 1.统一启动入口
不管是单机版还是集群版，这个类都是统一的启动入口：org.apache.zookeeper.server.quorum.QuorumPeerMain  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/server-init/QuorumPeerMain.png)   
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
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/server-init/initializeAndRun.png)   
## 3.DatadirCleanupManager-文件清理器
### 3.1 内部关键属性
1、原由  
zookeeper内部管理的数据分两块，一块是内存中的数据，一块是磁盘中的数据（快照，日志）
随着服务器的运行时间越来越长，那么这些磁盘的历史文件也会越来越多，所以需要采用定时清理  
2、解决方案  
zookeeper内部采用DatadirCleanupManager来实现文件的定期清理  
3、源码关键说明    
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/server-init/DatadirCleanupManager.png)    
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
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/server-init/DatadirCleanupManager-02.png) 

# 三、单机版服务启动流程-源码解析
## 1.启动入口
```java
ZooKeeperServerMain.main(args);
```
## 2.分析核心方法执行流程
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/main.png)  
```java
public static void main(String[] args) {
    ZooKeeperServerMain main = new ZooKeeperServerMain();
    try {
        //才是真正的核心逻辑方法
        main.initializeAndRun(args);
    } catch (IllegalArgumentException e) {
        //忽略非核心逻辑代码
    } 
}


protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    //忽略非核心逻辑代码（JMX监控服务）
    
    //目的：关注的是zookeeper如何对外提供服务的
    //配置文件对象
    ServerConfig config = new ServerConfig();
    if (args.length == 1) {
        //1.解析配置文件
        config.parse(args[0]);
    } else {
        config.parse(args);
    }
    
    //2.根据配置文件来运行zookeeper服务
    //继续发现boss[核心]
    runFromConfig(config);
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-01.png)  
## 3.步骤一：解析配置文件
```java
public void readFrom(QuorumPeerConfig config) {
    //客户端端口
    clientPortAddress = config.getClientPortAddress();
    secureClientPortAddress = config.getSecureClientPortAddress();
    //快照文件对象
    dataDir = config.getDataDir();
    //日志文件对象
    dataLogDir = config.getDataLogDir();
    tickTime = config.getTickTime();
    maxClientCnxns = config.getMaxClientCnxns();
    minSessionTimeout = config.getMinSessionTimeout();
    maxSessionTimeout = config.getMaxSessionTimeout();
}
```
## 4.步骤二：启动zookeeper服务
执行入口：runFromConfig(config)  
```java
public void runFromConfig(ServerConfig config)
            throws IOException, AdminServerException {
	LOG.info("Starting server");
	FileTxnSnapLog txnLog = null;
	try {
		// Note that this thread isn't going to be doing anything else,
		// so rather than spawning another thread, we will just call
		// run() in this thread.
		// create a file logger url from the command line args
        
        //1.创建并初始化快照日志文件操作对象
		txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);
        //2.创建并初始化zookeeper服务对象
		final ZooKeeperServer zkServer = new ZooKeeperServer(txnLog,
		                    config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, null);
		// Registers shutdown handler which will be used to know the
		// server error or shutdown state changes.
       
        //3.监控zookeeper服务的关闭情况（不是重点）
		final CountDownLatch shutdownLatch = new CountDownLatch(1);
		zkServer.registerServerShutdownHandler(
		                    new ZooKeeperServerShutdownHandler(shutdownLatch));
        
		// Start Admin server
        //4.管理服务，用于监控zookeeper服务的运行情况（不是重点）
		adminServer = AdminServerFactory.createAdminServer();
		adminServer.setZooKeeperServer(zkServer);
		adminServer.start();
		Boolean needStartZKServer = true;
		if (config.getClientPortAddress() != null) {
            //5.创建ServerCnxnFactory对象，用于处理客户端和服务端的连接请求
			cnxnFactory = ServerCnxnFactory.createFactory();
			cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), false);
            //6.启动服务
			cnxnFactory.startup(zkServer);
			// zkServer has been started. So we don't need to start it again in secureCnxnFactory.
			needStartZKServer = false;
		}
		if (config.getSecureClientPortAddress() != null) {
			secureCnxnFactory = ServerCnxnFactory.createFactory();
			secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), true);
			secureCnxnFactory.startup(zkServer, needStartZKServer);
		}
		containerManager = new ContainerManager(zkServer.getZKDatabase(), zkServer.firstProcessor,
		                    Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
		                    Integer.getInteger("znode.container.maxPerMinute", 10000)
		            );
		containerManager.start();
		// Watch status of ZooKeeper server. It will do a graceful shutdown
		// if the server is not running or hits an internal error.
		shutdownLatch.await();
		shutdown();
		
        //忽略
	}
	catch (InterruptedException e) {
		//忽略
	}
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-02.png)  
### 4.1 ZooKeeperServer内部细节
```java
//final ZooKeeperServer zkServer = new ZooKeeperServer(txnLog,config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, null);
public ZooKeeperServer(FileTxnSnapLog txnLogFactory, int tickTime,int minSessionTimeout, int maxSessionTimeout, ZKDatabase zkDb) {
    //1.创建serverStats对象，服务状态对象
    serverStats = new ServerStats(this);
    //2.操作日志和快照文件的对象[FileTxnSnapLog]
    this.txnLogFactory = txnLogFactory;
    //3.zookeeper数据库对象[ZKDatabase]
    this.zkDb = zkDb;
    //4.会话超时管理
    this.tickTime = tickTime;
    setMinSessionTimeout(minSessionTimeout);
    setMaxSessionTimeout(maxSessionTimeout);
    listener = new ZooKeeperServerListenerImpl(this);
    
    //忽略非核心代码
}
```
#### （1）ServerStats-封装监控数据的统计信息
```java
public class ServerStats {
    //1.服务端向客户端发送的响应包次数
    private long packetsSent;
    //2.接收到的客户端发送的请求包次数
    private long packetsReceived;
    //3.服务端处理请求的延迟情况
    private long maxLatency;
    private long minLatency = Long.MAX_VALUE;
    private long totalLatency = 0;
    //4.服务端处理客户端的请求次数
    private long count = 0;
}
```
#### （2）FileTxnSnapLog-实现快照及日志文件的持久化
```java
public class FileTxnSnapLog {
    //the direcotry containing the
    //the transaction logs
    //文件目录
    private final File dataDir;
    //the directory containing the
    //the snapshot directory
    //快照
    private final File snapDir;
    //日志
    private TxnLog txnLog;
    private SnapShot snapLog;
}
```
#### （3）ZKDatabase-zookeeper数据库对象
```java
public class ZKDatabase {

    //1.内存的数据节点（节点树）
    protected DataTree dataTree;
    //2.快照及日志文件
    protected FileTxnSnapLog snapLog;
    //3.管理会话
    protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
}
```
#### （4）总结：内部数据管理结构
zookeeper内部需要管理的数据分为三块，其中业务处理分为内存的节点树+快照日志文件（ZKDatabase），还有一块是我们监控数据统计（ServerStats）  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-03.png) 
### 4.2 ServerCnxnFactory-处理网络连接
#### （1）内部的灵活扩展机制
```java
static public ServerCnxnFactory createFactory() throws IOException {
    //1.读取系统的参数-zookeeper.serverCnxnFactory    [ZOOKEEPER_SERVER_CNXN_FACTORY = "zookeeper.serverCnxnFactory"]
    String serverCnxnFactoryName =
        System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
    if (serverCnxnFactoryName == null) {
        //2.如果没有配置，则获取到NIOServerCnxnFactory的类名  [没有配置的话，zookeeper默认用的是NIOServerCnxnFactory，但是zk内部也实现了可以用netty来实现网络连接]
        serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
    }
    try {
        //3.根据这个类名通过反射来具体的实例化ServerCnxnFactory对象
        //网络连接的处理方式支持通过参数的设置的方式来扩展
        ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
            .getDeclaredConstructor().newInstance();
        LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
        return serverCnxnFactory;
    } catch (Exception e) {
        //忽略非核心代码
    }
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-04.png)   
目前内部也支持了Netty来实现网络连接处理  
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-05.png) 
#### （2）启动服务主流程
```java
//cnxnFactory.startup(zkServer)
public void startup(ZooKeeperServer zkServer) throws IOException, InterruptedException {
    startup(zkServer, true);
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-06.png)   
```java
@Override
public void startup(ZooKeeperServer zks, boolean startServer)
    throws IOException, InterruptedException {
    //1.启动相关的处理线程
    start();
    setZooKeeperServer(zks);
    
    if (startServer) {
        //2.数据相关-恢复本地数据
        zks.startdata();
        //3.启动服务-建立处理请求责任链
        zks.startup();
    }
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-07.png)  
#### （3）步骤一：start()
```java
@Override
public void start() {
    stopped = false;
    //线程池
    if (workerPool == null) {
        workerPool = new WorkerService(
            "NIOWorker", numWorkerThreads, false);
    }
    //选择器
    for(SelectorThread thread : selectorThreads) {
        if (thread.getState() == Thread.State.NEW) {
            thread.start();
        }
    }
    // ensure thread is started once and only once
    //accept线程
    if (acceptThread.getState() == Thread.State.NEW) {
        acceptThread.start();
    }
    //expirer超时线程
    if (expirerThread.getState() == Thread.State.NEW) {
        expirerThread.start();
    }
}
```
#### （4）步骤二：zks.startdata()
```java
public void startdata()
    throws IOException, InterruptedException {
    //check to see if zkDb is not null
    //初始化zk的数据库对象
    if (zkDb == null) {
        zkDb = new ZKDatabase(this.txnLogFactory);
    }
    if (!zkDb.isInitialized()) {
        //加载本地数据-做数据恢复 [zkDb不是初始化了，证明已经在运行了，需要将内存中的数据加载到磁盘，避免宕机的数据丢失，可以做数据恢复]
        loadData();
    }
}
```
#### （5）步骤三：zks.startup()
```java
public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    //1.启动session的跟踪器
    startSessionTracker();
    //2.重点：建立请求处理链路(责任链)
    setupRequestProcessors();
	
    //3.注册JMX
    registerJMX();

    setState(State.RUNNING);
    notifyAll();
}
```
### 4.3 责任链模式的请求处理链
责任链模式，每个处理器处理请求的不同逻辑部分【可以联想 Netty的handler处理链】  
```java
// setupRequestProcessors()
protected void setupRequestProcessors() {
    //RequestProcessor-请求处理器
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor syncProcessor = new SyncRequestProcessor(this,finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}

// RequestProcessor finalProcessor = new FinalRequestProcessor(this);
public FinalRequestProcessor(ZooKeeperServer zks) {
    this.zks = zks;
}

//RequestProcessor syncProcessor = new SyncRequestProcessor(this,finalProcessor);
// syncProcessor的下一个处理器是finalProcessor
public SyncRequestProcessor(ZooKeeperServer zks,RequestProcessor nextProcessor) {
    super("SyncThread:" + zks.getServerId(), zks.getZooKeeperServerListener());
    this.zks = zks;
    this.nextProcessor = nextProcessor;
    running = true;
}

//firstProcessor = new PrepRequestProcessor(this, syncProcessor);
// firstProcessor的下一个处理器是syncProcessor
public PrepRequestProcessor(ZooKeeperServer zks,RequestProcessor nextProcessor) {
    super("ProcessThread(sid:" + zks.getServerId() + " cport:" + zks.getClientPort() + "):", zks.getZooKeeperServerListener());
    this.nextProcessor = nextProcessor;
    this.zks = zks;
}

public interface RequestProcessor {
	//下面是各种实现的子类
}
```
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/start-08.png)  
## 5.汇总图
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/single-server/summary.png)  
1、解析配置文件zoo.cfg  
  
【内部数据管理】  
2、zookeeper内部的数据管理结构（DataTree+FileTxnSnapLog）  
3、FileTxnSnapLog实现快照日志文件的持久化  
4、DatadirCleanupManager定期清理快照日志文件（文件清理采用的是PurgetTxnLog）  
  
【对外提供服务】  
前提条件：加载本地数据  
1、ServerCnxnFactory（NIO，Netty等），负责处理客户端连接请求  
2、基于责任链模式的处理链  

# 三、集群服务器的启动流程-源码解析
## 1.执行入口
```java
//if (args.length == 1 && config.isDistributed()) {
//    runFromConfig(config);
//}

public void runFromConfig(QuorumPeerConfig config)throws IOException, AdminServerException{
     
   	//忽略非核心代码
    
    try {
        ServerCnxnFactory cnxnFactory = null;
        ServerCnxnFactory secureCnxnFactory = null;

        if (config.getClientPortAddress() != null) {
            //1.创建ServerCnxnFactory对象，处理网络连接
            cnxnFactory = ServerCnxnFactory.createFactory();
            cnxnFactory.configure(config.getClientPortAddress(),
                                  config.getMaxClientCnxns(),
                                  false);
        }

        //忽略非核心代码

        //2.创建QuorumPeer对象，表示一个服务实例
        quorumPeer = getQuorumPeer();
        //3.为其设置FileTxnSnapLog属性
        quorumPeer.setTxnFactory(new FileTxnSnapLog(
            config.getDataLogDir(),
            config.getDataDir()));
        quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
        quorumPeer.enableLocalSessionsUpgrading(
            config.isLocalSessionsUpgradingEnabled());
        //quorumPeer.setQuorumPeers(config.getAllMembers());
        //4.设置选举类型
        quorumPeer.setElectionType(config.getElectionAlg());
        //server Id
        quorumPeer.setMyid(config.getServerId());
        quorumPeer.setTickTime(config.getTickTime());
        quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
        quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
        quorumPeer.setInitLimit(config.getInitLimit());
        quorumPeer.setSyncLimit(config.getSyncLimit());
        quorumPeer.setConfigFileName(config.getConfigFilename());
        //5.设置ZKDatabase
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
        quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
        if (config.getLastSeenQuorumVerifier()!=null) {
            quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
        }
        quorumPeer.initConfigInZKDatabase();
        quorumPeer.setCnxnFactory(cnxnFactory);
        quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
        quorumPeer.setLearnerType(config.getPeerType());
        quorumPeer.setSyncEnabled(config.getSyncEnabled());
        quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());

        // sets quorum sasl authentication configurations
        quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
        if(quorumPeer.isQuorumSaslAuthEnabled()){
            quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
            quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
            quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
            quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
            quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
        }
        quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
        //初始化服务节点的相关信息
        quorumPeer.initialize();
        
        //启动服务
        quorumPeer.start();
        quorumPeer.join();
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Quorum Peer interrupted", e);
    }
}
```
## 2.start方法继续解读
```java
quorumPeer.start();

//QuorumPeer
@Override
public synchronized void start() {
    //忽略非核心代码
    
    //装载数据
    loadDataBase();
    
    //启动网络服务
    startServerCnxnFactory();
    
    //忽略非核心代码
    
    //选举？？【其实是初始化选举算法】
    startLeaderElection();
    
    //开启选举算法
    super.start();
}
```
### 2.1 startLeaderElection()-解读
```java
synchronized public void startLeaderElection() {
    try {
        if (getPeerState() == ServerState.LOOKING) {
            //构建选票
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch(IOException e) {
        //忽略
    }

    //忽略
    
    //初始化选举算法
    //electionType默认是3 [因为zk集群leader选举采用过半数follower机制，肯定奇数，无设置的话那默认就是3]
    this.electionAlg = createElectionAlgorithm(electionType);
}

//this.electionAlg = createElectionAlgorithm(electionType);
protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;

    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
        //忽略无关代码
        case 3:
            qcm = createCnxnManager();
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
                listener.start();
                //标识为3，采用的选举算法为FastLeaderElection
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
    }
    return le;
}
```
### 2.2 super.start()-开启选举算法
```java
//QuorumPeer
super.start();

//进入父类Thread模板方法, QuorumPeer继承Thread，会执行QuorumPeer内部的run方法
public synchronized void start() {
    /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
        }
    }
}

//QuorumPeer内部run方法
@Override
public void run() {
    updateThreadName();

    //忽略非核心代码

    try {
        /*
             * Main loop
             */
        while (running) {
            switch (getPeerState()) {
                case LOOKING:
                    //判断当前服务器的状态是否为Looking
                    LOG.info("LOOKING");

                    if (Boolean.getBoolean("readonlymode.enabled")) {
                        LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                        // Create read-only server but don't start it immediately
                        final ReadOnlyZooKeeperServer roZk =
                            new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);

                         //忽略非核心代码
                        
                        try {
                            roZkMgr.start();
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                            //调用lookForLeader()方法
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        } finally {
                            // If the thread is in the the grace period, interrupt
                            // to come out of waiting.
                            roZkMgr.interrupt();
                            roZk.shutdown();
                        }
                    } else {
                        try {
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        }                        
                    }
                    break;
                case OBSERVING:
                    try {
                        LOG.info("OBSERVING");
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e );
                    } finally {
                        observer.shutdown();
                        setObserver(null);  
                        updateServerState();
                    }
                    break;
                case FOLLOWING:
                    try {
                        LOG.info("FOLLOWING");
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        follower.shutdown();
                        setFollower(null);
                        updateServerState();
                    }
                    break;
                case LEADING:
                    LOG.info("LEADING");
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        if (leader != null) {
                            leader.shutdown("Forcing shutdown");
                            setLeader(null);
                        }
                        updateServerState();
                    }
                    break;
            }
            start_fle = Time.currentElapsedTime();
        }
    } finally {
        //忽略非核心代码
    }
}
```
## 3.汇总图
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images/cluster/summary.png) 

# 四、单机版服务、集群服务启动流程汇总图
![](https://github.com/xiaozhuofu/zookeeper_sound_code/blob/master/xiaozhuofu-study/images//all.png) 
