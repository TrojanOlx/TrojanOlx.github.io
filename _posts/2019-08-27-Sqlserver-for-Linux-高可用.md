---
layout:     post
title:      Sqlserver for Linux 高可用
subtitle:   Sqlserver for Linux 高可用部署
date:       2019-08-27
author:     Trojan
header-img: img/post-bg-7.jpg
catalog: true
tags:
    - SqlServer
---
### Sqlserver for Linux 高可用

### **环境**   
主机名|IP|类型   
:--:|:--:|:--:   
node1|10.0.0.101|centos   
node2|10.0.0.102|centos   
node3|10.0.0.103|centos   

> [安装 sqlserver](https://docs.microsoft.com/zh-cn/sql/linux/quickstart-install-connect-red-hat?view=sql-server-2017)   

### **准备工作**   

1. 更新每个节点的机器名，必须满足：

    15个字符或更少。

    在网络中是唯一的。

    可以用以下语句修改机器名：   

    ```
    sudo vi /etc/hostname
    ```   

2. 配置主机名和IP地址的解析

    通常在DNS服务器注册主机名和IP地址，为了进一步保证同一个AG中多个节点可以互相通信，我们在每个节点使用如下命令修改Hosts文件：   

    ```
    sudo vi /etc/hosts

    10.0.0.101 node1
    10.0.0.102 node2
    10.0.0.103 node3
    ```   

### **创建AG**   

#### **启用AG并重启mssql-server**

在所有节点的SQL Server上启用AlwaysOn AG，然后重启mssql-server服务：   
```
sudo /opt/mssql/bin/mssql-conf set hadr.hadrenabled 1
sudo systemctl restart mssql-server
```   

#### **启用AlwaysOn_health扩展事件会话**  (T-SQL)

在每个节点上开启该会话，以便在对某一可用性组进行故障排除时帮助诊断根本原因：   
```
ALTER EVENT SESSION AlwaysOn_health ON SERVER WITH (STARTUP_STATE=ON);
GO
```   

#### 创建数据库镜像端点访问使用的用户   
```
CREATE LOGIN dbm_login WITH PASSWORD = '<Password>';
CREATE USER dbm_user FOR LOGIN dbm_login;
```   

#### **创建证书** (T-SQL)

Linux 上的 SQL Server 服务使用证书验证镜像终结点之间的通信。连接到主 SQL Server 实例。以下 Transact-SQL 脚本创建主密钥和证书。 然后备份证书，并使用私钥保护文件。 使用强密码更新脚本。   
```
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<Master_Key_Password>';
CREATE CERTIFICATE dbm_certificate WITH SUBJECT = 'dbm';
BACKUP CERTIFICATE dbm_certificate
TO FILE = '/var/opt/mssql/data/dbm_certificate.cer'
WITH PRIVATE KEY (
FILE = '/var/opt/mssql/data/dbm_certificate.pvk',
ENCRYPTION BY PASSWORD = '<Private_Key_Password>'
);
```   

#### 将证书和私钥拷贝到所有可用副本的服务器上的相同位置。(bash)   
```
cd /var/opt/mssql/data
scp dbm_certificate.* root@<node2>:/var/opt/mssql/data/
```   

#### 在每个目标服务器上，授予mssql用户访问这些文件的权限。（bash）   
```
cd /var/opt/mssql/data
chown mssql:mssql dbm_certificate.*
```   

#### 在辅助服务器上创建证书

以下 Transact-SQL 脚本根据在主 SQL Server 副本上创建的备份创建主密钥和证书。 使用强密码更新脚本。 解密密码与在此前的步骤中创建 .pvk 文件使用的密码相同。   
```
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<Master_Key_Password>';
CREATE CERTIFICATE dbm_certificate
FROM FILE = '/var/opt/mssql/data/dbm_certificate.cer'
WITH PRIVATE KEY (
FILE = '/var/opt/mssql/data/dbm_certificate.pvk',
DECRYPTION BY PASSWORD = '<Private_Key_Password>'
);
```   

#### 在所有节点上创建数据库镜像端点

（可选）可以包含 IP 地址 LISTENER_IP = (0.0.0.0)。 侦听器 IP 地址必须是 IPv4 地址。 还可以使用 0.0.0.0。

如果配置的节点是仅配置副本，唯一有效的值为ROLE = WITNESS。

对于 SQL Server 2017 版本中，支持数据库镜像终结点的唯一身份验证方法是CERTIFICATE。   

```
CREATE ENDPOINT [Hadr_endpoint]
AS TCP (LISTENER_IP = (0.0.0.0), LISTENER_PORT = <5022>)
FOR DATA_MIRRORING (
ROLE = ALL,
AUTHENTICATION = CERTIFICATE dbm_certificate,
ENCRYPTION = REQUIRED ALGORITHM AES
);
ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
GRANT CONNECT ON ENDPOINT::[Hadr_endpoint] TO [dbm_login];
```   


### 在主节点上创建AG   
- 三个同步副本   
  > 此配置包含三个同步副本。 默认情况下，它提供高可用性和数据保护。 它还可以提供读取缩放。

    | |读取缩放|高可用性(& a)数据保护|数据保护   
    :--:|:--:|:--:|:--:
    主要副本中断|手动故障转移。 可能导致数据丢失。 新的主副本是 R / w|自动故障转移。 新的主副本是 R / w|自动故障转移。 新的主数据库不可用的用户事务，直到在先前的主要副本恢复并加入可用性组作为辅助  
    一个次要副本中断|主要是 R / w。 如果主没有自动故障转移会失败。|主要是 R / w。 如果主没有自动故障转移也会失败|主数据库不可用的用户事务。 

  ```
    CREATE AVAILABILITY GROUP [ag1]
    WITH (DB_FAILOVER = ON, CLUSTER_TYPE = EXTERNAL)
    FOR REPLICA ON
    N'<node1>'
    WITH (
    ENDPOINT_URL = N'tcp://<node1>:<5022>',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    ),
    N'<node2>'
    WITH (
    ENDPOINT_URL = N'tcp://<node2>:<5022>',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    ),
    N'<node3>'
    WITH(
    ENDPOINT_URL = N'tcp://<node3>:<5022>',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    );
    ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
  ```   

- 两个同步副本   
    > 此配置启用数据保护。像其他可用性组配置，它可以实现读取缩放。 两个同步副本配置不提供自动高可用性。
  | |读取缩放|数据保护   
    :--:|:--:|:--:
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT=|0|1*|2  
    主要副本中断|0*|@shouldalert  
    主要副本中断|手动故障转移。 可能导致数据丢失。 新的主副本是 R / w。|自动故障转移。 新的主数据库不可用的用户事务，直到在先前的主要副本恢复并加入可用性组作为辅助。   
    一个次要副本中断|主要是 R/W，运行可能导致数据丢失。| 主次要副本恢复之前不可用的用户事务。   

    ```
    CREATE AVAILABILITY GROUP [ag1]
    WITH (CLUSTER_TYPE = EXTERNAL)
    FOR REPLICA ON
    N'node1' WITH (
    ENDPOINT_URL = N'tcp://node1:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    ),
    N'node2' WITH (
    ENDPOINT_URL = N'tcp://node2:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    );
    ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
    ```   
- 两个同步副本和仅配置副本   
  | |读取缩放|数据保护   
    :--:|:--:|:--:
    REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT=|0 *|@shouldalert  
    主要副本中断|自动故障转移。 新的主副本是 R / w。|自动故障转移。 新的主数据库不可用的用户事务。  
    次要副本中断|主要副本是 R/W，运行可能导致数据丢失 （如果主数据库将失败并且无法恢复）。 如果主没有自动故障转移也会失败。|主数据库不可用的用户事务。 如果主故障转移到没有副本也会失败。   
    配置仅副本中断|主要是 R / w。 如果主没有自动故障转移也会失败。| 主要是 R / w。 如果主没有自动故障转移也会失败。  
    同步辅助 + 配置仅副本中断|主数据库不可用的用户事务。 无自动故障转移。| 主数据库不可用的用户事务。 故障转移到如果没有副本以及主服务器失败。   

    ```
    CREATE AVAILABILITY GROUP [ag1]
    WITH (CLUSTER_TYPE = EXTERNAL)
    FOR REPLICA ON
    N'<node1>' WITH (
    ENDPOINT_URL = N'tcp://<node1>:<5022>',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    ),
    N'<node2>' WITH (
    ENDPOINT_URL = N'tcp://<node2>:<5022>',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE = EXTERNAL,
    SEEDING_MODE = AUTOMATIC
    ),
    N'<node3>' WITH (
    ENDPOINT_URL = N'tcp://<node3>:<5022>',
    AVAILABILITY_MODE = CONFIGURATION_ONLY
    );
    ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
    ```

#### 将辅助副本加入到AG
在所有辅助副本上执行：   

```
ALTER AVAILABILITY GROUP [ag1] JOIN WITH (CLUSTER_TYPE = EXTERNAL);
ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;
```   

#### 将数据库添加到可用性组
确保添加到可用性组的数据库处于完全恢复模式，并具有有效的日志备份。   

```
CREATE DATABASE [dbTest];
ALTER DATABASE [dbTest] SET RECOVERY FULL;
BACKUP DATABASE [dbTest]
TO DISK = N'/var/opt/mssql/data/dbTest.bak';
ALTER AVAILABILITY GROUP [ag1] ADD DATABASE [dbTest];
```   

#### 验证是否已在辅助服务器上创建了数据库

在每个辅助服务器上执行：   
```
SELECT * FROM sys.databases WHERE name = 'dbTest';
GO
SELECT DB_NAME(database_id) AS 'database', synchronization_state_desc FROM sys.dm_hadr_database_replica_states;
```   

---

转载整理至： [UltraSQL -- SQL Server 2017 on Linux](https://blog.51cto.com/ultrasql/2152717)   
查阅资料： [为 Linux 上的 SQL Server 创建和配置可用性组](https://docs.microsoft.com/zh-cn/sql/linux/sql-server-linux-create-availability-group?view=sql-server-2017)






