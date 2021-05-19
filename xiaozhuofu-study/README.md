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