# Mysql8.0 数据库生产环境Docker部署(Replication方式实现)  #


## Part 0 OverView

### 0.0 WHY? ###
如果你是资深运维不必多言，容器化就对了:)

MySQL8.0官方部署方法应该有14步，按照文档做一次一天就过去了，要不要加班？下一次部署是不是还要看文档配置再一天？
[MySQL8.0 Deployment Guide](https://dev.mysql.com/doc/mysql-secure-deployment-guide/8.0/en/)
<table><tr><td bgcolor=orange>早用Docker早下班，剩下时间看B站！</td></tr></table>

### 0.1 Scope ###
常见mysql集群方案有replication和PXC。本版本文档用只介绍**在单一宿主上**Replication部署的方法，PXC部署可以参考文档列表。
+ Replication采用异步复制，无法保证数据的一致。
+ PXC采用同步复制，事务在集群的所有节点要么同时提交，要么不提交，PXC使用的是percona，percona是mysql改进版，性能挺升很大。
---  
### 0.2 Index ###
+ Docker Compose部署单一数据库
    + 环境变量&.env文件
    + MySQL配置文件
    + 数据持久化储存&数据库文件挂载
    + 日志
+ 单一宿主主备双库数据库读写分离
    + MySQL配置文件
+ 多宿主主备数据库集群（预留,暂不涉及）    
+ 后续完善的内容
    + MySQL管理工具Adminer（在官方docker compose中出现）
    + 在docker启动时运行sql文件完成部署（Initializing a fresh instance）
    + 生产环境下的数据库备份
        + 使用MySQLdump进行数据库冷备
        + 使用xtrabackup数据库热备
---   
### 0.3 Folder Structure ###
```
.
│  .env
│  docker-compose.yml
│
└─mysql
    ├─config
    │      my.cnf
    │
    ├─mysqldump
    │      db.sql
    │
    └─data
```
## Part 1 Docker Compose部署 ##

### 1.1.0 环境变量&.env文件 ###

#### a.镜像启动必(需)要的环境变量 ####
[docker-entrypoint.sh](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/docker-entrypoint.sh)
定义了MySQL镜像启动时可以定义的若干个环境变量,我们分析下面几个环境变量其他的请阅读[MySql@Docker Hub](https://hub.docker.com/_/mysql).
+ `MYSQL_ROOT_PASSWORD` 定义root账号的密码，这是一种不安全的设置方法，请不要在生产环境下使用。（√测试环境推荐√）
+ `MYSQL_RANDOM_ROOT_PASSWORD` root账号的密码安全的设置方法，镜像第一次启动时会随机生成并显示在命令行中
可以在宿主机上使用命令`sudo docker logs d74d58bc576f`来查看生成的密码，记得要请除带有密码的文件。
+ `MYSQL_USER`,`MYSQL_PASSWORD` 创建一个**超级用户**权限的用户，这个账号拥有所有数据库权限，此环境变量是可选的 `GRANT ALL`
+ `MYSQL_DATABASE` 创建一个管理员具有权限的数据库`GRANT ALL`
+ `MYSQL_INITDB_SKIP_TZINFO` 初始化数据库时[docker-entrypoint.sh]会自动设置数据库时区，有时这个过程会很慢（5min）我们可以跳过这个步骤，我们在.cnf文件中设置就好了

compose.yml:
```
environment:
  - MYSQL_INITDB_SKIP_TZINFO=1
```
mysql.cnf
```
default-time-zone="+8:00"
```

#### b.设置文件挂载的环境变量 <参考/建议>####
为保证镜像文件持久化保存，我们需要额外几个环境变量用来定义相关文件在宿主上的储存路径.
+ 数据库配置文件:{MYSQL_SQL_CNF}
+ 数据库文件{MYSQL_DATA}
+ 数据库初始化脚本{MYSQL_SQL_SCRIPT}
+ 数据库日志{MYSQL_SQL_LOG}

#### c .env文件 ####
+ 每次部署的配置都各不相同，我们尽量不去修改`docker-compose.yml`文件
+ 由于环境变量中可能包含数据库管理密码，强烈建议在镜像初始化后删除.env文件

### 1.1.1 单一数据库的mysql.cnf（后续更新会另写文档） ###
暂时只写如下几点单一部署还是很简单的
+ 时区（1.1.0.a）
+ 编码格式格式
+ `skip-name-resolve` docker部署的mysql一定不要禁用dns解析，docker内部都是以镜像名作为主机名进行解析的，如果关闭数据库将无法访问。

### 数据持久化储存&数据库文件挂载 ###
镜像内的数据库文件若不设置挂载或挂载错误，在镜像重启后数据库中的所有信息（配置&数据）都会消失，务必确保数据库文件挂载到宿主文件系统中。

compose.yml:
```
volumes:
  - ${MYSQL_SQL_CNF}:/etc/mysql/conf.d
  - ${MYSQL_DATA}:/var/lib/mysql
  - ${MYSQL_SQL_SCRIPT}:/docker-entrypoint-initdb.d
```
**注意**宿主路径在前，容器路径在后



--- 
## Part n-1 后续完善的内容

### Adminer —— 数据库管理工具 ###
Adminer是一个类似于phpMyAdmin的MySQL管理客户端。整个程序只有一个PHP文件，易于使用和安装。因此官方文档中也出现了Adminer。
yml参考:

```
version: '3.1'

services:

  db_master:
     (...master db configuration)

  db_slave0:
     (...slave db configuration)

  adminer:
    image: adminer
    restart: always
    ports:
      - 8080:8080
```

### 在docker启动时运行sql文件完成部署-Initializing a fresh instance ###

#### WHY? ####
当数据迁移时我们也需要在数据库部署的同时完成数据库数据的初始化。
另外，在使用docker-compose部署数据库时虽然可以通过环境变量完成创建表和用户，但是环境变量不能满足多数据库多用户部署的要求。(参见docker hub文档)
```
*MYSQL_DATABASE*
This variable is optional and allows you to specify the name of a database to be created on image startup. 
If a user/password was supplied (see below) then that user will be granted superuser access (corresponding to GRANT ALL) to this database.
```
Docker hub中给出了相应的解决方案:
```
*Initializing a fresh instance*
When a container is started for the first time, a new database with the specified name will be created and initialized with the provided configuration variables. 
Furthermore, it will execute files with extensions .sh, .sql and .sql.gz that are found in /docker-entrypoint-initdb.d. 
Files will be executed in alphabetical order. You can easily populate your mysql services by mounting a SQL dump into that directory and provide custom images with contributed data. 
SQL files will be imported by default to the database specified by the MYSQL_DATABASE variable.
```
#### 代码分析 ####
先分析一下：Dockerfile & docker-entrypoint.sh 文件

[Dockerfile](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/Dockerfile)
```
COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]
```

[docker-entrypoint.sh](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/docker-entrypoint.sh)
```
for f in /docker-entrypoint-initdb.d/*; do
    case "$f" in
        *.sh)     echo "$0: running $f"; . "$f" ;;
        *.sql)    echo "$0: running $f"; "${mysql[@]}" < "$f"; echo ;;
        *.sql.gz) echo "$0: running $f"; gunzip -c "$f" | "${mysql[@]}"; echo ;;
        *)        echo "$0: ignoring $f" ;;
    esac
    echo
done
```

#### yml代码实现方式 ####
由上可知docker镜像启动时可以会运行**镜像文件夹**`/docker-entrypoint-initdb.d/`中所有 .sql .sq .sh 的脚本。
所以我们只需要将数据库脚本或者shell脚本挂载到**镜像文件夹**`/docker-entrypoint-initdb.d/`中，在镜像初次启动中即可自动完成数据库的数据初始化。
在yml文件中，修改在volumes配置项中加入如下信息即可实现数据初始化。
+ Example:
```
volumes:
  - ${MYSQL_SQL_SCRIPT}:/docker-entrypoint-initdb.d
  - other configuration ............... :)
```
+ **问题1**主备两个镜像都要绑定脚本文件吗？---->>>>上测试结果
+ **问题2**挂载的脚本文件每次docker-compose up 都要运行一次吗？
  No，同[docker-entrypoint.sh](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/docker-entrypoint.sh)
  设置环境变量的逻辑相同:
  当镜像文件系统中存在`$DATADIR/mysql`文件夹时，脚本就不会设置环境变量/运行初始化脚本。
```
	if [ ! -d "$DATADIR/mysql" ]; then
        some shell script...
fi
```


#### 创建数据库&账号的方式 ####
需要注意的是，`docker-entrypoint.sh`会在镜像初次启动时优先处理环境变量定义的内容：
+ 创建root账号&主机：使用环境变量定义
+ 创建创建所有数据库普通账号&主机(GRANT ALL)：推荐使用环境变量定义
+ 创建单一数据库权限的用户: 在创建数据库后再创建用户并授权
+ 数据库:参照创建用户账号的方法

### 生产环境下的数据库备份 ###
#### 冷备 ####
待完善
#### 热备 ####
待完善


---
## Part n 参考信息&感谢
+ MySql官方Docker Hub文档
[MySql@Docker Hub](https://hub.docker.com/_/mysql)
+ 单一主数据库：
[浮云Cloud](https://www.cnblogs.com/cloudfloating/p/11541020.html)
+ 主备数据库：
[chanjarster](https://github.com/chanjarster/mysql-master-slave-docker-example), 
[诗圣_杜甫](https://juejin.im/post/6844903811920707592), 
[digitalocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql)
+ sql部署启动
[程序员欣宸](https://blog.csdn.net/boling_cavalry/article/details/71055159)
+ 数据库集群
[一个大西瓜](https://www.cnblogs.com/wyt007/p/10828879.html)
[祝亚森](https://zhuyasen.com/post/mysql-cluster.html)
