(以下docker相关的命令，需要在root用户环境下或通过sudo提升权限来进行操作。)

## 1.rabbitmq安装
常用的 Vim 操作命令整理
https://juejin.cn/post/7070699702732783623



```bash
sta@sta-virtual-machine:~$ sudo mkdir ./data/rabbitmq -p


sudo docker run -d --hostname rabbit-svr --name rabbit -p 5672:5672 -p 15672:15672 -p 25672:25672 -v ./data/rabbitmq:/var/lib/rabbitmq rabbitmq:management

sta@sta-virtual-machine:~$ sudo docker ps

sudo docker exec -it rabbit rabbitmq-plugins enable rabbitmq_tracing
rabbitmq启用日志功能记录消息队列收发情况：https://blog.csdn.net/fuck487/article/details/78677108



```
## 1.拉取MySQL5.7镜像到本地
```bash
docker pull mysql

//启动存在的dockers
sudo docker container start mysql-master;

sudo docker container start mysql-slave;

sudo docker ps;

sudo netstat -antp | grep docker

//终止已经存在的dockers
docker container stop mysql-master

docker container stop mysql-slave;

//查看所有已经创建的包括终止状态的容器
docker container ls -a

https://yeasy.gitbook.io/docker_practice/container/rm


# 如果你只需要跑一个mysql实例，不做主从，那么执行以下命令即可，不用再做后面的参考步骤:
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
#然后用shell或客户端软件通过配置( 用户名:root 密码:root IP:你的本机ip 端口:3306)来登录即可
```

## 2. 准备MySQL配置文件

mysql5.7安装后的默认配置文件在`/etc/mysql/my.cnf`, 而自定义的配置文件一般放在`/etc/mysql/conf.d`这个路径下。
现在我们在本地host主机上自定义的某个目录(如`/data/mysql/conf/`)，先创建两个文件master.conf和slave.conf，分别用于配置主从两个节点。
- /data/mysql/conf/master.conf
```
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
log_bin = log  #开启二进制日志，用于从节点的历史复制回放
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
server_id = 1  #需保证主库和从库的server_id不同， 假设主库设为1
replicate-do-db=fileserver  #需要复制的数据库名，需复制多个数据库的话则重复设置这个选项
```
- /data/mysql/conf/slave.conf
```
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
log_bin = log  #开启二进制日志，用于从节点的历史复制回放
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
server_id = 2  #需保证主库和从库的server_id不同， 假设从库设为2
replicate-do-db=fileserver  #需要复制的数据库名，需复制多个数据库的话则重复设置这个选项
```
## 3. Docker分别运行MySQL主/从两个容器

- 将mysql主节点运行起来
```bash
mkdir -p /data/mysql/datam
docker run -d --name mysql-master -p 13306:3306 -v ./data/mysql/conf/master.conf:/etc/mysql/mysql.conf.d/mysqld.cnf -v ./data/mysql/datam:/var/lib/mysql  -e MYSQL_ROOT_PASSWORD=root mysql
```
运行参数说明:
>--name mysql-master: 容器的名称设为mysql-master
>
>-p 13306:3306: 将host的13306端口映射到容器的3306端口
>
>-v ./data/mysql/conf/master.conf:/etc/mysql/mysql.conf.d/mysqld.cnf ： master.conf配置文件挂载
>
>-v /data/mysql/datam:/var/lib/mysql ： mysql容器内数据挂载到host的/data/mysql/datam， 用于持久化
>
>-e MYSQL_ROOT_PASSWORD=root : mysql的root登录密码为root

- 将mysql从节点运行起来
```bash
mkdir -p /data/mysql/datas
docker run -d --name mysql-slave -p 13307:3306 -v ./data/mysql/conf/slave.conf:/etc/mysql/mysql.conf.d/mysqld.cnf -v ./data/mysql/datas:/var/lib/mysql  -e MYSQL_ROOT_PASSWORD=root mysql
```
运行参数说明:
>--name mysql-slave: 容器的名称设为mysql-slave
>
>-p 13307:3306: 将host的13307端口映射到容器的3306端口
>
>-v /data/mysql/conf/slave.conf:/etc/mysql/mysql.conf.d/mysqld.cnf ： slave.conf配置文件挂载
>
>-v /data/mysql/datas:/var/lib/mysql ： mysql容器内数据挂载到host的/data/mysql/datas， 用于持久化
>
>-e MYSQL_ROOT_PASSWORD=root : mysql的root登录密码为root

## 4.登录MySQL主节点配置同步信息

- 登录mysql
```
# 192.168.1.xx 是你本机的内网ip
mysql -u root -h 192.168.1.xx -P13306 -p123456
mysql -uroot -h192.168.140.132 -P13306 -p;

```
- 在mysql client中执行

```
//mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'slave';

mysql> create user 'slave'@'%' identified by 'slave';


mysql> grant replication slave on *.* to 'slave'@'%'  with grant option;

mysql> flush privileges;
mysql> create database fileserver default character set utf8mb4;
```
再获取status, 得到类似如下的输出:
```
mysql> show master status\G;
*************************** 1. row ***************************
             File: log.000025
         Position: 155
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```
## 5.登录MySQL从节点配置同步信息

- 另开一个tab登录mysql

```shell
# 192.168.1.xx 是你本机的内网ip
mysql -u root -h 192.168.1.xx -P13307 -p123456
mysql -uroot -h192.168.140.132 -P13307 -p;

```
- 在mysql client中操作:

```
mysql> stop slave;

mysql> reset slave;


#注意其中的日志文件和数值要和上面show master status的值对应
mysql> CHANGE MASTER TO MASTER_HOST='你的本地ip地址如192.168.1.x',master_port=13306,MASTER_USER='slave',MASTER_PASSWORD='slave',MASTER_LOG_FILE='log.000025',MASTER_LOG_POS=155;

mysql> CHANGE MASTER TO MASTER_HOST='192.168.140.132',master_port=13306,MASTER_USER='slave',MASTER_PASSWORD='slave',MASTER_LOG_FILE='binlog.000002',MASTER_LOG_POS=0;

mysql> start slave;

mysql> show slave status\G;

```
## 出错改正办法：


```
（2）报错：Fatal error: The slave I/O thread stops because master and slave have equal MySQL server ids; these ids must be different for replication to work (or the --replicate-same-server-id option must be used on slave but this does not always make sense; please check the manual before using it)


mysql> show variables like 'server_id';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | server_id     | 1     |
    +---------------+-------+
    1 row in set (0.22 sec)
    ###在数据库里看居然是一样的

mysql>set global server_id=2; #此处的数值和my.cnf里设置的一样就行

http://www.hebinghua.com/studydetail/13/38.html

（1）报错：Last_IO_Error: error connecting to master 'replica@192.168.78.102:3306' - retry-time: 30 retries: 6 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.

- 在mysql client中执行：
ALTER USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'slave';

https://www.cnblogs.com/wtx106/p/16307911.html

```


再获取status, 正常应该得到类似如下的输出:
```
mysql> show slave status\G;
// ...
Slave_IO_Running: Yes 
Slave_SQL_Running: Yes 
// ...
```
到这时说明主从配置已经完成，可以尝试在主mysql的`fileserver`数据库里建表操作下，然后在从节点上检查数据是否已经同步过来。
