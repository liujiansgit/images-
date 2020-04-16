## 基于canal的Mysql数据同步测试准备

### 1、主从数据库准备

1个主库，2个从库

要求3个库均为utf8字符集编码，主库开启日志同步，并设置日志模式为row模式。

my.inf文件配置参考：

```
log_bin = mysql-bin    # 打开日志（自定义名称）
binlog_format = ROW  # 设置row模式的日志格式（必须使用row模式）
server-id = 2 #id不能重复
```

主库中配置canal数据库用户

```
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```



完成后提供3个库的数据库名称、数据库账号、数据库密码、数据库端口信息，两个从库需提供约等于root用户权限的用户（较大权限的用户）。

### 2、rocketmq安装部署

参考：http://rocketmq.apache.org/docs/quick-start/  或者： https://blog.csdn.net/markfengfeng/article/details/80445529

jvm启动参数建议保持原样或者调大，以保证mq的性能，默认配置为

```
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx8g -Xmn8g"
```



###### 1、安装jdk

1. 查看是否已经安装JDK：rpm -qa | grep -i java
2. 若有则删除：rpm -e --nodeps java-xxx，删除所有相关的java
3. 下载[jdk8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)安装包，将gz压缩文件放到指定目录如/usr/local，解压：tar -zxvf      jdk-8u191-linux-x64.tar.gz
4. 设置全局变量：vim /etc/profile，追加
   ​      JAVA_HOME=/usr/local/jdk1.8.0_191
   ​      CLASSPATH=$JAVA_HOME/lib/
   ​      PATH=$PATH:$JAVA_HOME/bin
   ​      export PATH JAVA_HOME CLASSPATH
5. source /etc/profile
6. java -version

###### 2、安装maven

1.下载maven包 
首先从官网上 <http://maven.apache.org/> 下载最新版Maven。

2、执行`tar -zxvf apache-maven-3.6.0-bin.tar.gz`

3、vim /etc/profile，追加

export M2_HOME=/usr/local/apache-maven-3.6.0
export PATH=$PATH:$M2_HOME/bin

4、source /etc/profile

5、mvn –v

###### 3、安装rocketmq

1、git clone -b develop https://github.com/apache/incubator-rocketmq.git

2、cd incubator-rocketmq

3、打包   mvn -Prelease-all -DskipTests clean install -U 

4、cd distribution/target/apache-rocketmq/bin

5、启动服务 nohup sh mqnamesrv &

查看日志 tail -f ~/logs/rocketmqlogs/namesrv.log

6、启动nohup sh mqbroker -n localhost:9876 -c ../conf/broker.conf &

tail -f ~/logs/rocketmqlogs/broker.log



###### 注意

1、启动mqbroker前需修改conf文件夹下，broker.conf,添加如下两行，否则可能会启动不了broker

```
namesrvAddr = 127.0.0.1:9876

#实际ip地址，不能使用localhost/127.0.0.1等

brokerIP1 = 192.168.0.22  
```

2、修改runbroker.sh和runserver.sh的jvm启动参数，否则可能启动失败

```
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx8g -Xmn8g"
```

改成(根据实际情况调整)：

```
JAVA_OPT="${JAVA_OPT} -server -Xms128m -Xmx256m -Xmn256m"
```



完成后提供mqserver的访问地址（ip+端口）

### 3、rocketmq-console控制台安装

参考：https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console

可使用docker方式，namesrv.addr需设置为第3步中mq地址，参考配置	

```
server.contextPath=
#控制台端口
server.port=8777
#spring.application.index=true
spring.application.name=rocketmq-console
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
logging.config=classpath:logback.xml
#if this value is empty,use env value rocketmq.config.namesrvAddr  NAMESRV_ADDR | now, you can set it in ops page.default localhost:9876
#name节点配置，重要
rocketmq.config.namesrvAddr=192.168.174.129:9876
#if you use rocketmq version < 3.5.8, rocketmq.config.isVIPChannel should be false.default true
rocketmq.config.isVIPChannel=false
#rocketmq-console's data path:dashboard/monitor
rocketmq.config.dataPath=/tmp/rocketmq-console/data
#set it false if you don't want use dashboard.default true
rocketmq.config.enableDashBoardCollect=true
```

安装方式：下载git上面的zip文件，上传至服务器解压，将console模块打包成jar包，直接java -jar启动（默认端口8080）

```
mvn clean package -Dmaven.test.skip=true

nohup java -jar target/rocketmq-console-ng-1.0.0.jar &

--后台运行
```





完成后提供访问地址

### 4、canal的server端部署

参考：https://github.com/alibaba/canal/wiki/AdminGuide 中的“部署”

推荐使用发布包的部署方式，安装位置建议在离主库最近的位置。

canal配置文件(`canal.properties`)关键配置

```
#canal端口
canal.port = 11111
#消息模式
canal.serverMode = RocketMQ
#rocketmq地址(第2步的安装中涉及)
canal.mq.servers = 192.168.174.129:9876
#使用false（非json格式）
canal.mq.flatMessage = false
canal.mq.compressionType = none
#消息确认
canal.mq.acks = all
#修改为主库的数据库名和密码
canal.instance.tsdb.dbUsername = liujian
canal.instance.tsdb.dbPassword = liujian@123
```

instance.properties关键配置：

```
#主库数据库配置
canal.instance.master.address=192.168.0.22:3306
canal.instance.dbUsername=liujian
canal.instance.dbPassword=liujian@123
canal.instance.connectionCharset = UTF-8
canal.instance.defaultDatabaseName =test
canal.instance.enableDruid=false

#白名单配置
canal.instance.filter.regex=
#黑名单配置
canal.instance.filter.black.regex=mysql\\..*,sys\\..*,information_schema\\..*,performance_schema\\..*
#mq config，配置单独的group和topic
canal.mq.topic=slavetest2
canal.mq.partition=0
canal.mq.producerGroup = Canal-Producer2
```

