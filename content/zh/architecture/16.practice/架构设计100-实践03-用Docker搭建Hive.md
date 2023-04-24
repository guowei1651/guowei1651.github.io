[架构设计](https://www.jianshu.com/c/753debf1423d)系列文章，请参见连接。

# 背景
因为现在很多云都使用PAAS的方式提供大数据存储与计算的功能。作者就在思考他们是怎样实现的，并且提供出动态部署能力的大数据存储与计算过程的。所以，使用Docker进行Hadoop环境的搭建。并且为之后的K8s环境搭建提供一种思路。

在环境构建与使用的过程可以分为几个阶段：准备过程，搭建过程，配置过程，启动过程、测试过程。

# 准备过程
主要是准备软件包，并为Docker镜像与配置项分离做准备。
- **准备软件包**
> 下载以下软件包：
>  - [hadoop-2.7.7](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-2.7.7/hadoop-2.7.7.tar.gz)
>  - [hive-2.3.6](https://mirrors.tuna.tsinghua.edu.cn/apache/hive/hive-2.3.6/apache-hive-2.3.6-bin.tar.gz)
>  - [mysql-connector-java-5.1.48](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.48.tar.gz)
>  - [jdk-8u231-linux-x64.rpm](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

- **创建目录**
```bash
mkdir -p /home/docker/{centos_ssh,hadoop,hive,mysql}
```
- **上传文件**
```bash
cd /home/docker/hadoop
rz 【hadoop-2.7.7.tar.gz，jdk-8u231-linux-x64.rpm】
cd /home/docker/hive
rz 【apache-hive-2.3.6-bin.tar.gz，jdk-8u231-linux-x64.rpm，mysql-connector-java-5.1.48.tar.gz】
```
- **解压并Copy配置**
```bash
cd /home/docker/hadoop
tar -xzvf hadoop-2.7.7.tar.gz
cp -rf hadoop-2.7.7/etc/hadoop/* ./conf
cd /home/docker/hive
tar -xzvf apache-hive-2.3.6-bin.tar.gz
tar -xzvf mysql-connector-java-5.1.48.tar.gz
cp -rf apache-hive-2.3.6-bin/conf/ ./conf
```
- **创建my.cnf**
为Mysql配置my.cfg配置。这个配置是从CDH6.3的配置中拉过来的。
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
symbolic-links = 0

key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space.
#Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
#system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

#In later versions of MySQL, if you enable the binary log and do not set
#a server_id, MySQL will not start. The server_id must be unique within
#the replicating group.
server_id=1

binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
```
修改my.cfg权限
```
chmod 644 my.cnf
```

# 搭建过程
在这个过程中将上面准备好的包，构建为Docker镜像。

- **搭建Docker+CentOS+SSH环境**

```
FROM centos:7.2.1511
MAINTAINER wales.kuo

RUN yum install -y wget && mkdir -p /etc/yum.repos.d/repo_bak \
	&& mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/repo_bak/ \
	&& wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo \
	&& wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo \
	&& yum clean all && yum makecache \
	&& yum install -y openssh-server sudo openssh-clients \
	&& ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key \
	&& ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key  \
	&& ssh-keygen -t rsa -f /etc/ssh/ssh_host_ecdsa_key \
	&& ssh-keygen -t rsa -f /etc/ssh/ssh_host_ed25519_key \
	&& echo "root:XXX" | chpasswd  && echo "root   ALL=(ALL)       ALL" >> /etc/sudoers \
	&& sed -i 's/.*session.*required.*pam_loginuid.so.*/session optional pam_loginuid.so/g' /etc/pam.d/sshd \
	&& sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config \
	&& sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config \
	&& mkdir /var/run/sshd

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```
使用命令构建：
```
docker build -t centos-ssh .
```
- **搭建Hadoop环境**
```
FROM centos-ssh
MAINTAINER wales.kuo

ADD jdk-8u231-linux-x64.rpm /usr/local/
ADD hadoop-2.7.7.tar.gz /usr/local/
RUN yum install -y which gcc && rpm -ivh /usr/local/jdk-8u231-linux-x64.rpm \
	&& rm -rf /usr/local/jdk-8u231-linux-x64.rpm
ENV HADOOP_HOME /usr/local/hadoop-2.7.7
ENV JAVA_HOME /usr/java/jdk1.8.0_231-amd64/jre/
ENV PATH $HADOOP_HOME/bin:$PATH
```
使用命令构建：
```
docker build -t hadoop:2.7.7 -t centos-hadoop .
```
- **搭建MetaData的Mysql**
```
docker pull mysql:5.7.27
```


- **搭建Hive环境**
```
FROM centos-hadoop
MAINTAINER wales.kuo

ADD apache-hive-2.3.6-bin.tar.gz /usr/local/
RUN mv /usr/local/apache-hive-2.3.6-bin /usr/local/hive && mkdir /home/hadoop/hive/tmp
ADD mysql-connector-java-5.1.48/mysql-connector-java-5.1.48-bin.jar /usr/local/hive/lib/
ENV HIVE_HOME /usr/local/hive
ENV PATH $HIVE_HOME/bin:$PATH
```
使用命令构建：
```
docker build -t hive:2.3.6 -t centos-hive .
```

# 配置过程
修改Hadoop，Hive的默认参数，准备启动使用。并且因为Hadoop需要免密登录的配置，所以需要先将容器启动，然后再进入容器中免密登录部分。

#### 修改Hadoop参数
hadoop配置修改/home/docker/hadoop/conf/下的配置文件core-site.xml、hdfs-site.xml、yarn-site.xml、mapred-site.xml、slaves。

(1)hadoop-env.sh
```
export JAVA_HOME=/usr/java/jdk1.8.0_231-amd64/jre/
```
(2)core-site.xml

``` xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop0:9000</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/usr/local/hadoop/tmp</value>
        </property>
         <property>
                 <name>fs.trash.interval</name>
                 <value>1440</value>
        </property>
</configuration>
```
(3)hdfs-site.xml
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
```
(4)yarn-site.xml
```
<configuration>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property> 
                <name>yarn.log-aggregation-enable</name> 
                <value>true</value> 
        </property>
        <property>
                <description>The hostname of the RM.</description>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop0</value>
        </property>
</configuration>
```
(5)修改文件名：mv mapred-site.xml.template mapred-site.xml 
vi mapred-site.xml
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
(10)slaves
删除原来的所有内容，修改为如下
```
hadoop1
hadoop2
```
#### 修改Hive参数
使用hive的默认配置，创建配置文件。并修改配置。
(1)Copy创建配置文件
```
cp hive-env.sh.template hive-env.sh
cp hive-default.xml.template hive-site.xml
cp hive-log4j2.properties.template hive-log4j2.properties
cp hive-exec-log4j2.properties.template hive-exec-log4j2.properties
```
(2)修改hive-env.sh
```
export JAVA_HOME=/usr/java/jdk1.8.0_231-amd64/jre/    ##Java路径
export HADOOP_HOME=/usr/local/hadoop-2.7.7   ##Hadoop安装路径
export HIVE_HOME=/usr/local/hive    ##Hive安装路径
export HIVE_CONF_DIR=/usr/local/hive/conf    ##Hive配置文件路径
```
(3)修改hive-site.xml
>把{system:java.io.tmpdir} 改成 /usr/local/hive/tmp
>把 {system:user.name} 改成 {user.name}

并修改以下内容
```xml
<property>
    <name>hive.exec.scratchdir</name>
    <value>/user/hive/tmp</value>
</property>
<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
</property>
<property>
    <name>hive.querylog.location</name>
    <value>/user/hive/log</value>
</property>

<!--配置 MySQL 数据库连接信息-->
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://mysql:3306/metastore?createDatabaseIfNotExist=true&characterEncoding=UTF-8&useSSL=false</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root123</value>
  </property>
```

#### 启动容器
```
version: '3.1'

services:
  db:
    image: mysql:5.7.27
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    ports:
      - 3306:3306
    volumes:
      - /home/docker/mysql/log:/var/log/
      - /home/docker/mysql/my.cnf:/etc/my.cnf
      - /home/docker/mysql/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: dls123456



docker network create --subnet=172.19.0.0/16 hadoop_default
docker run --name hadoop0 --hostname hadoop0 --net hadoop_default --ip 172.19.0.2 -v /home/docker/hadoop/conf:/usr/local/hadoop-2.7.7/etc/hadoop/ -v /home/docker/hadoop/hadoop0:/usr/local/hadoop/tmp -v /home/docker/hadoop/hadoop0/logs:/usr/local/hadoop-2.7.7/logs/ -d -P -p 50070:50070 -p 8088:8088  centos-hadoop
docker run --name hadoop1 --hostname hadoop1 --net hadoop_default --ip 172.19.0.3 -v /home/docker/hadoop/conf:/usr/local/hadoop-2.7.7/etc/hadoop/ -v /home/docker/hadoop/hadoop1:/usr/local/hadoop/tmp  -v /home/docker/hadoop/hadoop0/logs:/usr/local/hadoop-2.7.7/logs/ -d -P centos-hadoop
docker run --name hadoop2 --hostname hadoop2 --net hadoop_default --ip 172.19.0.4 -v /home/docker/hadoop/conf:/usr/local/hadoop-2.7.7/etc/hadoop/ -v /home/docker/hadoop/hadoop2:/usr/local/hadoop/tmp  -v /home/docker/hadoop/hadoop0/logs:/usr/local/hadoop-2.7.7/logs/ -d -P centos-hadoop
docker run -d --name mysql --hostname mysql --restart=always -p 3306:3306 -v /home/docker/mysql/log:/var/log/ -v /home/docker/mysql/my.cnf:/etc/my.cnf -v /home/docker/mysql/data:/var/lib/mysql --net hadoop_default -e MYSQL_ROOT_PASSWORD=dls123456 mysql:5.7.27 --default-authentication-plugin=mysql_native_password
docker run --name hive --hostname hive --net hadoop_default --ip 172.19.0.6 -p9083:9083 -p 10000:10000 -p 10002:10002 -v /home/docker/hive/conf:/usr/local/hive/conf -v /home/docker/hive/tmp:/usr/local/hive/tmp -v /home/docker/hive/data:/user/hive/ -v /home/docker/hadoop/conf:/usr/local/hadoop-2.7.7/etc/hadoop/ -v /home/docker/hadoop/hadoop0:/usr/local/hadoop/tmp -v /home/docker/hadoop/hadoop0/logs:/usr/local/hadoop-2.7.7/logs/ -d -P centos-hive
```
#### 创建数据库
```sql
create database metastore;
```

#### 容器内Hadoop配置
(1)设置ssh免密码登录 
在hadoop0上执行下面操作
```
ssh-keygen -t rsa(一直按回车即可)
ssh-copy-id -i localhost
ssh-copy-id -i hadoop0
ssh-copy-id -i hadoop1
ssh-copy-id -i hadoop2
```
在hadoop1上执行下面操作
```
ssh-keygen -t rsa(一直按回车即可)
ssh-copy-id -i localhost
ssh-copy-id -i hadoop1
```
在hadoop2上执行下面操作
```
ssh-keygen -t rsa(一直按回车即可)
ssh-copy-id -i localhost
ssh-copy-id -i hadoop2
```
#### 容器内Hive配置
(1)初始化Hive mysql数据库
```
schematool -dbType mysql -initSchema
```
# 启动服务
(1)启动Hadoop服务
```
cd /usr/local/hadoop-2.7.7/
sbin/start-all.sh
```
(2)启动Hive服务
```
hive --service metastore &
hiveserver2
```
(3)在hdfs 中创建下面的目录 ，并且授权
```
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -mkdir -p /user/hive/tmp
hdfs dfs -mkdir -p /user/hive/log
hdfs dfs -chmod -R 777 /user/hive/warehouse
hdfs dfs -chmod -R 777 /user/hive/tmp
hdfs dfs -chmod -R 777 /user/hive/log
```

# 测试过程

### hadoop测试
使用程序验证集群服务 
(1)创建一个本地文件a.txt
```
hello you
hello me
```
(2)上传a.txt到hdfs上
```
hdfs dfs -put a.txt /
```
(3)执行wordcount程序
```
cd /usr/local/hadoop-2.7.7/share/hadoop/mapreduce
hadoop jar hadoop-mapreduce-examples-2.7.7.jar wordcount /a.txt /out
```
(4)查看结果
```
hdfs dfs -text /out/part-r-00000
```

# 参考
[Kubernetes-在Kubernetes集群上搭建Hadoop集群](https://www.jianshu.com/p/e768ab139842)
[Docker+Hadoop+Hive+Presto 使用Docker部署Hadoop环境和Presto](https://www.cnblogs.com/liujinhong/p/8795387.html)
[Install and Configure MySQL for Cloudera Software](https://docs.cloudera.com/documentation/enterprise/5-16-x/topics/cli_install_mysql.html)
