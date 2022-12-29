# CDH(Cloudera)离线安装和踩坑

## 背景
需要CDH去学习Flink, 但是现在开源cdh都无法直接下载, 只能找个blog尝试

## 目标
在centos7(docker)下成功运行cdh

## 业界方案
[Cloudera Manager CDH离线安装](https://gujincheng.github.io/2022/03/11/Cloudera-Manager-CDH%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85/)
### 安装准备

准备3台虚拟机
golden-01
golden-02
golden-03
并设置免秘钥登录，这里忽略

下载cdh6.2.0，在百度网盘下载：
https://pan.baidu.com/s/1QlWyMRSRJNcaStsZMSdmLg 提取码：ietj

### 关闭防火墙与关闭 SELINUX

1. 关闭防火墙

   ```shell
   systemctl stop firewalld  ## 关闭防火墙
   systemctl disable firewalld ###禁止防火墙开机自启
   ```

2. 关闭SELINUX，只需要关闭一个需要配置http文件服务的虚拟机就可以

   vim /etc/selinux/config —> SELINUX=disabled (修改)

   这里改完了需要重启虚拟机

   这里如果不关闭SELINUX，下面视同httpd访问文件的时候，会报错：

   ```shell
   You don't have permission to access upload/ on this server
   ```

### 配置NTP服务（所有节点）

修改时区（改为中国标准时区）

```shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
## 安装ntp
yum -y install ntp 
```

ntp主机配置 vi /etc/ntp.conf
在文件里新增server ntp.aliyun.com，并把原始的server给注释掉
重启`service ntpd restart`

### 配置本地服务器（选定任意一台主机即可）

配置的本地文件服务器的目的是为了，让之后其他节点可以从这直接下载

```shell
yum install -y httpd  ##安装httpd
service httpd start 启用htttpd
cd /var/www/html/  && mkdir cdh6  && mkdir cm6
## 把下载的rpm放到文件服务cm6里
cp /home/golden/cdh6.2.0/cm6/* /var/www/html/cm6/
cp /home/golden/cdh6.2.0/oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm /var/www/html/cm6/
```

这里注意，启用httpd服务以后，防火墙和selinux必须都关闭，否则，就会报如下错误：

```shell
You don't have permission to access upload/ on this server
```

生成yum源的描述的目录信息 可以让其他节点知道到这里下载

```shell
yum install -y createrepo  ##下载createrepo命令
## 进入到cm6安装包的httpd资源位置
cd /var/www/html/cm6
##创建yum源的描述meta
createrepo .
```

在**所有节点上**添加yum源的配置文件

```shell
cat >> /etc/yum.repos.d/cm6.repo << EOF
[cm6-local]
name=cm6-local
baseurl=http://192.168.233.133/cm6/
enabled=1
gpgcheck=0
EOF
```

查看yum配置源是否生效

```shell
yum clean all
yum repolist
yum makecache
```

### JDK安装

前面安装了2次，都是使用自己配置JAVA_HOME的方式，都没问题，但是，最后一次使用了CentOS7精简版的，还是使用自己配置JAVA_HOME的方式，发现cloudera-manager-server启动不成功
通过`journalctl -u cloudera-manager-server`以及`journalctl -xe`来查看启动日志，发现，报找不到java。但是java安装没有问题
最后使用了yum安装了jdk问题解决了

```shell
yum install -y oracle-j2sdk1.8-1.8.0+update181-1.x86_64
```

### clouder server 与agent安装

#### 安装cm6相关依赖（所有节点）

```shell
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse fuse-libs redhat-lsb httpd mod_ssl
```

#### 安装Cloudera Manager Server

这一步只需要在CM Server节点上操作。
执行下面的命令：

```shell
# 安装openjdk8
yum install oracle-j2sdk1.8   ##这里不知道需不需要安装，因为我本机已经安装了java了。这里先不安装，如果报错再重来
# 安装 cm manager(只需在server节点安装)
yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```

### 配置本地Parcel存储库

Cloudera Manager Server安装完成后，进入到本地Parcel存储库目录：

```shell
cd /opt/cloudera/parcel-repo  ## Cloudera Manager Server完成以后，该目录就已经生成了
```

将第一部分下载的CDH parcels文件上传至该目录下，然后执行修改sha文件：

```shell
mv /data6/upload/parcels/* /opt/cloudera/parcel-repo/
mv CDH-6.3.0-1.cdh6.3.0.p0.1279813-el7.parcel.sha1 CDH-6.3.0-1.cdh6.3.0.p0.1279813-el7.parcel.sha
```

### 配置mysql jdbc驱动

从前面下载好的mysql-connector-java-5.1.47.tar.gz包中解压出mysql-connector-java-5.1.47-bin.jar文件，将mysql-connector-java-5.1.47-bin.jar文件上传至CM Server节点上的/usr/share/java/目录下并重命名为mysql-connector-java.jar（如果/usr/share/java/目录不存在，需要手动创建）：

```shell
cd /usr/share/java  && mv /home/golden/mysql-connector-java-5.1.47.jar .
```

### 创建CDH所需要的数据库

根据所需要安装的服务参照下表创建对应的数据库以及数据库用户，数据库必须使用utf8编码，创建数据库时要记录好用户名及对应密码：

```sql
-- scm
CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'scm';

-- amon
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';

-- rman
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'rman';

-- hue
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci; 
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'hue';

-- hive
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive';

-- sentry
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;   
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry';

-- nav
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;      
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'nav';

-- navms
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'navms';

-- oozie
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';

-- hive
CREATE DATABASE hive DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
-- flush
FLUSH PRIVILEGES;
```

### 设置Cloudera Manager 数据库

Cloudera Manager Server包含一个配置数据库的脚本。

- mysql数据库与CM Server是同一台主机
  执行命令：`/opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm`

- mysql数据库与CM Server不在同一台主机上

  执行命令：

  ```
  /opt/cloudera/cm/schema/scm_prepare_database.sh mysql -h <mysql-host-ip> --scm-host <cm-server-ip> scm scm
  ```

  执行如下命令：

  ```shell
  /opt/cloudera/cm/schema/scm_prepare_database.sh mysql -uroot -h golden-02 -p'Gjc123!@#' --scm-host golden-01 scm scm scm
  ```

  这里卡了很久，报了如下错误：

  ```shell
  ERROR Exception when creating/dropping database with user 'root' and jdbc url 'jdbc:mysql://golden-02/?useUnicode=true&characterEncoding=UTF-8'
  com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
  
  The last packet successfully received from the server was 202 milliseconds ago.  The last packet sent successfully to the server was 197 milliseconds ago.
      at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)[:1.8.0_321]
      at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)[:1.8.0_321]
      at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)[:1.8.0_321]
      at java.lang.reflect.Constructor.newInstance(Constructor.java:423)[:1.8.0_321]
  ```

  找了很久的问题，首先是安装的mysql不支持远程连接，需要做如下配置：

  ```sql
  grant all privileges on *.* to root@'%' identified by 'Gjc123!@#';
  flush privileges;
  ```

  配置了以后，还是报错，找了很久，最终找到原因，是因为下载的

  ```
  mysql-connector-java-5.1.47-bin.jar
  ```

  有问题，重新下载了一个，问题解决

### 安装CDH节点

#### 启动Cloudera Manager Server服务

```shell
systemctl start cloudera-scm-server
```

然后等待Cloudera Manager Server启动，可能需要稍等一会儿，可以通过命令`tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log`去监控服务启动状态。
当看到`INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.`日志打印出来后，说明服务启动成功，可以通过浏览器访问`Cloudera Manager WEB`界面了。

#### 访问Cloudera Manager WEB界面

- 打开浏览器，访问地址：http://:7180，默认账号和密码都为admin：
- 首先是Cloudera Manager的欢迎页面，点击页面右下角的【继续】按钮进行下一步
- 勾选接受条款，点击【继续】进行下一步：
- 版本选择,这里我就选择免费版了：
- 选择版本以后会出现第二个欢迎界面，不过这个是安装集群的欢迎页：
- 选择主机,这一步是要搜索并选择用于安装CDH集群的主机，在主机名称后面的输入框中输入各个节点的hostname，中间使用英文逗号分隔开，然后点击搜索，在结果列表中勾选要安装CDH的节点即可：

![CM离线安装_配置主机](https://fastly.jsdelivr.net/gh/matrixcall/bed001@master/picx/2022/12/111.4krzalhyl080.jpg)

- 指定存储库`Cloudera Manager Agent`这里选择自定义，填写上面使用httpd搭建好的Cloudera Manager YUM 库URL：
- CDH and other software 如果我们之前的【配置本地Parcel存储库】步骤操作无误的话，这里会自动选择【使用Parcel】，并加载出CDH版本，但是这里一直没有识别出来，还报了如下错误：

![CM离线安装_识别不到parcel-repo](https://fastly.jsdelivr.net/gh/matrixcall/bed001@master/picx/2022/12/xxx.2ifj59dr41k0.jpg)

找到问题原因：是因为parcel文件的`.sha1`需要改成`.sha`，修改完以后，就能识别出来了
这里一开始还跟着教程先用yum安装了`cloudera-manager-agent`，安装完了更报错，这里不要安装

![CM离线安装_正常识别到parcel-repo](https://fastly.jsdelivr.net/gh/matrixcall/bed001@master/picx/2022/12/xxx.3phcsf5flu80.jpg)



- JDK安装选项，这里jdk已经安装了，不要勾选
- SSH登录配置，用于配置集群主机之间的SSH登录，填写root用户的密码，根据集群配置填写合适的【同时安装数量】值即可：

  ![CM离线安装_提供SSH登录凭据](https://fastly.jsdelivr.net/gh/matrixcall/bed001@master/picx/2022/12/xxx.3fvnbztgzie0.jpg)



- 安装Agent

- 安装Parcels

- 主机检查，`Inspect Network Performance `和`Inspect Network Performance`需要点击的，一开始以为是自动检查，一直等着
  然后标黄了几个选项：
  Cloudera 建议将 /proc/sys/vm/swappiness 设置为最大值 10。当前设置为 30。使用 sysctl 命令在运行时更改该设置并编辑 /etc/sysctl.conf，以在重启后保存该设置。您可以继续进行安装，但 Cloudera Manager 可能会报告您的主机由于交换而运行状况不良。
  已启用透明大页面压缩，可能会导致重大性能问题。请运行echo never > /sys/kernel/mm/transparent_hugepage/defrag和echo never > /sys/kernel/mm/transparent_hugepage/enabled以禁用此设置，然后将同一命令添加到 /etc/rc.local 等初始化脚本中，以便在系统重启时予以设置。
  安装上面的提示执行即可；
  但是有一点，报找不到java，这个有点不知道怎么解决了，这里先跳过了，点击`I understand this risks`

### 安装CDH集群

#### 选择服务类型

这里我选择自定义服务，然后选择很多组件，hdfs、zk、yarn、hbase、kafka等等

#### 角色分配

这里会有默认设置，然后把需要手动设置的手动设置一下，点击继续

#### 数据库设置

这里根据要求设置即可
后面的步骤就没有什么问题了，可以参考下面的两篇文章。

这里参考了
[CentOS7 Cloudera Manager6 完全离线安装 CDH6 集群](https://segmentfault.com/a/1190000021809645)
[cdh6.2离线安装（傻瓜式安装教程）](https://blog.csdn.net/weixin_45682234/article/details/105844209)

### CDH日志清理

看原文(此处略)



## 个人方案
参考业界方案即可, 这里重点记录余下的坑
> docker模拟虚拟机运行, 指定host, 用centos7可以成功, ubuntu尝试多次不行
```
docker run  --net=bridge-to-router --ip=192.168.0.243 --name node243 --hostname node243 -d  --restart always  --privileged=true  centos:7 /usr/sbin/init && sy@tty1.service && systemctl mask getty@tty1.service
```
> /etc/hosts 不要随便乱加, 不能加自定义的 , 如node240.mxcall.top, 就直接加
```
192.168.0.241   node241   node241.bridge-to-router
192.168.0.242   node242   node242.bridge-to-router
192.168.0.243   node243   node243.bridge-to-router
```
后面的 node243.bridge-to-router 是根据`/etc/cloudera-scm-agent/config.ini`中内容
和报错信息来加的 ;

> /etc/cloudera-scm-agent/config.ini 重点修改 , 不修改Agent安装不成功
```
listening_hostname=node242
reported_hostname=node242
max_collection_wait_seconds=60.0
metrics_url_timeout_seconds=60.0
task_metrics_timeout_seconds=30.0
monitored_nodev_filesystem_types=tmpfs
```

> 基本hive插入查询语句, 验证成功性
```sql
CREATE DATABASE if not exists ccbft_ods LOCATION '/user/hive/warehouse/ccbft_ods.db';

CREATE TABLE if not exists ccbft_ods.ods_test_mongohive
( 
id string comment 'mongo的id' 
,flinkBson string comment 'mongo的JSON'
) 
comment 'mongo导入hive表'
partitioned by (dt string comment 'yyyyMMdd' ) 
row format delimited fields terminated by '\003' 
stored as parquet
tblproperties ('parquet.compression'='SNAPPY');

insert into ccbft_ods.ods_test_mongohive partition(dt='20221229') values('1','{"aa":1}'); 
select * from ods_test_mongohive where dt='20221229';
```