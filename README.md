# MySQL 部署手册 #
整合Mysql部署所需的知识，用以指导简单业务场景下容器化的数据库部署、维护、备份等工作。

**项目代码包含以下几个部分：**
+ 单宿主单数据库docker compose 的部署方案
+ 单宿主一主一（多）备数据库docker compose 的部署方案（replication）
+ PXC数据库集群方案
+ mycat分库分表读写分离容器化部署方案


**项目Wiki中的几个部分:**
* 基础篇
    + MySQL8.0 知识整理
    + Docker 知识整理
* 初级篇
    + 启动一个MySQL8.0容器
    + 单宿主机利用docker compose启动MySQL8.0容器
    + 单宿主机利用docker compose启动MySQL8.0一主一（多）容器
    + 简单场景下数据库配置最佳实践/my.cnf
  
