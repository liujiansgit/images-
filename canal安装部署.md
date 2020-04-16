# 环境要求

### 1. 操作系统

​    a.  纯java开发，windows/linux均可支持

​    b.  jdk建议使用1.6.25以上的版本，稳定可靠，目前阿里巴巴使用基本为此版本. 

 

### 2. mysql要求

   a. 当前的canal开源版本支持5.7及以下的版本(阿里内部mysql 5.7.13, 5.6.10, mysql 5.5.18和5.1.40/48)，ps. mysql4.x版本没有经过严格测试，理论上是可以兼容

   b. canal的原理是基于mysql binlog技术，所以这里一定需要开启mysql的binlog写入功能，并且配置binlog模式为row.

```
[mysqld]  
log-bin=mysql-bin #添加这一行就ok  
binlog-format=ROW #选择row模式  
server_id=1 #配置mysql replaction需要定义，不能和canal的slaveId重复  
```

​    数据库重启后, 简单测试 `my.cnf` 配置是否生效:

```
mysql> show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

​    如果 my.cnf 设置不起作用,请参考:
​    <https://stackoverflow.com/questions/38288646/changes-to-my-cnf-dont-take-effect-ubuntu-16-04-mysql-5-6>
​    <https://stackoverflow.com/questions/52736162/set-binlog-for-mysql-5-6-ubuntu16-4>

   c.  canal的原理是模拟自己为mysql slave，所以这里一定需要做为mysql slave的相关权限 

```
CREATE USER canal IDENTIFIED BY 'canal';    
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';  
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;  
FLUSH PRIVILEGES; 
```

​     针对已有的账户可通过grants查询权限：

```
show grants for 'canal' 
```

###  

# 部署

### 1. 获取发布包

方法1： (直接下载)

访问：<https://github.com/alibaba/canal/tree/gh-pages/download> ，会列出所有历史的发布版本包

下载方式，比如以1.0.4版本为例子： 

```
wget https://raw.github.com/alibaba/canal/gh-pages/download/canal.deployer-1.0.4.tar.gz
```

方法2:  (自己编译)

```
git clone git@github.com:alibaba/canal.git
git co canal-$version #切换到对应的版本上
mvn clean install -Denv=release
```

执行完成后，会在canal工程根目录下生成一个target目录，里面会包含一个 canal.deployer-$verion.tar.gz

 

### 2. 目录结构

解压缩发布包后，可得如下目录结构：

```
drwxr-xr-x 2 jianghang jianghang  136 2013-03-19 15:03 bin
drwxr-xr-x 4 jianghang jianghang  160 2013-03-19 15:03 conf
drwxr-xr-x 2 jianghang jianghang 1352 2013-03-19 15:03 lib
drwxr-xr-x 2 jianghang jianghang   48 2013-03-19 15:03 logs
```

 

### 3. 启动/停止

   linux启动 :   

```
sh startup.sh 
```

   linux带debug方式启动：(默认使用suspend=y，阻塞等待你remote debug链接成功)

```
sh startup.sh debug 9099
```

   linux停止：

```
sh stop.sh
```

​       

  几点注意： 

1. linux启动完成后，会在bin目录下生成canal.pid，stop.sh会读取canal.pid进行进程关闭
2. startup.sh默认读取系统环境变量中的which java获得JAVA执行路径，需要设置PATH=$JAVA_HOME/bin环境变量

\-------------   

​    windows启动：(windows支持相对比较弱)

```
startup.bat
```

​    windows停止：直接关闭终端即可