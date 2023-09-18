# fe-plugins-auditloader
### 插件说明：

StarRocks 中所有 SQL 的审计日志保存在本地 fe/log/fe.audit.log 中，没有入库保存。为方便对业务 SQL 进行分析，社区开发出将审计信息入库 StarRocks 的审计日志插件。在实现上，StarRocks 会在执行 SQL 后调用该插件收集 SQL 的审计信息，审计信息的内容会在内存中攒批后基于 Stream Load 的方式导入至 StarRocks 表中。

**注意事项：**

1、使用插件：随着 StarRocks 的迭代升级，审计日志 fe.audit.log 中的字段个数可能会逐渐增多，因此不同的 StarRocks 版本需要使用对应版本的插件，同时在 StarRocks 中存放审计日志的表的创建语句也要随之调整。当前代码及下文演示所用的建表语句适用于 **StarRocks 2.4 及后续**版本。

2、开发插件：若发现 StarRocks 新版本中的审计日志字段或格式出现了变化，则需要替换 Java 工程中的 `fe-plugins-auditloader\lib\starrocks-fe.jar` 为新版本 StarRocks 包中 `fe/lib/starrocks-fe.jar`，同时修改代码中和字段相关的内容。



### 使用说明：

##### 1、创建内部表

首先需要在 StarRocks 中创建一个动态分区表，来保存审计日志中的数据。为了规范使用，建议为其单独创建一个数据库。

例如，创建存放审计日志的数据库 `starrocks_audit_db__`：

```SQL
create database starrocks_audit_db__;
```

在 `starrocks_audit_db__` 库创建 `starrocks_audit_tbl__` 表：

```SQL
CREATE TABLE starrocks_audit_db__.starrocks_audit_tbl__ (
  `queryId` VARCHAR(48) COMMENT "查询的唯一ID",
  `timestamp` DATETIME NOT NULL COMMENT "查询开始时间",
  `queryType` VARCHAR(12) COMMENT "查询类型（query, slow_query）",
  `clientIp` VARCHAR(32) COMMENT "客户端IP",
  `user` VARCHAR(64) COMMENT "查询用户名",
  `authorizedUser` VARCHAR(64) COMMENT "用户唯一标识，既user_identity",
  `resourceGroup` VARCHAR(64) COMMENT "资源组名",
  `catalog` VARCHAR(32) COMMENT "数据目录名",
  `db` VARCHAR(96) COMMENT "查询所在数据库",
  `state` VARCHAR(8) COMMENT "查询状态（EOF，ERR，OK）",
  `errorCode` VARCHAR(96) COMMENT "错误码",
  `queryTime` BIGINT COMMENT "查询执行时间（毫秒）",
  `scanBytes` BIGINT COMMENT "查询扫描的字节数",
  `scanRows` BIGINT COMMENT "查询扫描的记录行数",
  `returnRows` BIGINT COMMENT "查询返回的结果行数",
  `cpuCostNs` BIGINT COMMENT "查询CPU耗时（纳秒）",
  `memCostBytes` BIGINT COMMENT "查询消耗内存（字节）",
  `stmtId` INT COMMENT "SQL语句增量ID",
  `isQuery` TINYINT COMMENT "SQL是否为查询（1或0）",
  `feIp` VARCHAR(32) COMMENT "执行该语句的FE IP",
  `stmt` STRING COMMENT "SQL原始语句",
  `digest` VARCHAR(32) COMMENT "慢SQL指纹",
  `planCpuCosts` DOUBLE COMMENT "查询规划阶段CPU占用（纳秒）",
  `planMemCosts` DOUBLE COMMENT "查询规划阶段内存占用（字节）"
) ENGINE = OLAP
DUPLICATE KEY (`queryId`, `timestamp`, `queryType`)
COMMENT "审计日志表"
PARTITION BY RANGE (`timestamp`) ()
DISTRIBUTED BY HASH (`queryId`) BUCKETS 3 
PROPERTIES (
  "dynamic_partition.time_unit" = "DAY",
  "dynamic_partition.start" = "-30",  --表示只保留最近30天的审计信息，可视需求调整
  "dynamic_partition.end" = "3",
  "dynamic_partition.prefix" = "p",
  "dynamic_partition.buckets" = "3",
  "dynamic_partition.enable" = "true",
  "replication_num" = "3"  --若集群中BE个数不大于3，可调整副本数为1，生产集群不推荐调整
);
```

`starrocks_audit_tbl__` 是动态分区表，我们在建表时没有直接创建分区，所以建表后需要等待后台动态分区调度线程调度后才会生成当天及后三天的分区。动态分区线程默认每10分钟被调度一次，我们可以先观察该表的分区是否已经被创建，再进行后续操作。分区查看语句为：

```SQL
show partitions from starrocks_audit_db__.starrocks_audit_tbl__;
```



##### 2、修改配置文件

执行安装时，所需的审计插件完整包为 auditloader.zip，使用 unzip 命令解压插件：

```SHELL
[root@node01 ~]# unzip auditloader.zip
Archive:  auditloader.zip
  inflating: auditloader.jar        
  inflating: plugin.conf            
  inflating: plugin.properties
```

说明：该命令会将 zip 内的文件直接解压到当前目录，解压后可以得到插件中的三个文件：

`auditloader.jar`：插件代码打包的核心 jar 包

`plugin.conf`：插件配置文件，需根据集群信息修改

`plugin.properties`：插件属性文件，通常无需修改

根据我们实际的集群信息，修改配置文件 `plugin.conf`：

```XML
### plugin configuration

# The max size of a batch, default is 50MB.
max_batch_size=52428800

# The max interval of batch loaded, default is 60 seconds.
max_batch_interval_sec=60

# the max stmt length to be loaded in audit table, default is 4096.
max_stmt_length=4096

# StarRocks FE host for loading the audit, default is 127.0.0.1:8030.
# this should be the host port for stream load.
frontend_host_port=127.0.0.1:8030

# If the response time of a query exceed this threshold, it will be recored in audit table as slow_query.
qe_slow_log_ms=5000

# the capacity of audit queue, default is 1000.
max_queue_size=1000

# Database of the audit table.
database=starrocks_audit_db__

# Audit table name, to save the audit data.
table=starrocks_audit_tbl__

# StarRocks user. This user must have LOAD_PRIV to the audit table.
user=root

# StarRocks user's password.
password=
```

修改完成后，可使用 zip 命令将上面的三个文件重新打包为 zip 包：

```SHELL
[root@node01 ~]# zip -q -m -r auditloader.zip auditloader.jar plugin.conf plugin.properties
```

**注意：该命令会将需要打包的文件移动到 auditloader.zip 中，并覆盖该目录下原有的 auditloader.zip 文件。也即执行完打包命令后，该目录下只会保留一个最新的 auditloader.zip 插件包。**



##### 3、分发插件

当使用本地包方式安装时，需将 auditloader.zip 分发至集群所有 FE 节点，且各节点分发路径需要一致。例如我们都分发至 StarRocks 部署目录 `/opt/module/starrocks/` 下，也即 auditloader.zip 文件在集群所有 FE 节点的路径都为：

```
/opt/module/starrocks/auditloader.zip
```



##### 4、安装插件

StarRocks 安装本地插件的语法为：

```sql
INSTALL PLUGIN FROM "/location/plugin_package_name.zip";
```

例如根据上文分发文件的路径修改命令后执行：

```SQL
mysql> INSTALL PLUGIN FROM "/opt/module/starrocks/auditloader.zip";
```

安装完成后，查看当前已安装插件信息：

```SQL
mysql> SHOW PLUGINS\G
*************************** 1. row ***************************
       Name: __builtin_AuditLogBuilder
       Type: AUDIT
Description: builtin audit logger
    Version: 0.12.0
JavaVersion: 1.8.31
  ClassName: com.starrocks.qe.AuditLogBuilder
     SoName: NULL
    Sources: Builtin
     Status: INSTALLED
 Properties: {}
*************************** 2. row ***************************
       Name: AuditLoader
       Type: AUDIT
Description: Available for StarRocks 2.4 and later versions. Load audit information to StarRocks, and user can view the statistic of queries. 
    Version: 3.0.1
JavaVersion: 1.8.0
  ClassName: com.starrocks.plugin.audit.AuditLoaderPlugin
     SoName: NULL
    Sources: /opt/module/starrocks/auditloader.zip
     Status: INSTALLED
 Properties: {}
```

可看到当前有两个插件，其中 Name 为 `AuditLoader` 的插件即为上文安装的审计日志插件，其状态为 INSTALLED 表示已安装成功。Name 为 `__builtin_AuditLogBuilder` 的插件为 StarRocks 内置的审计插件，用来打印审计信息到本地日志目录生成  fe.audit.log，当前不需要关注。需要说明的是，这两个插件的数据来源于同一个方法，若感觉新安装的审计插件入库后的数据不正确，可对比 fe.audit.log 来进行验证。

**说明：fe/plugins 是 StarRocks 外部插件的安装目录，在审计插件安装完成后，会在各个 FE 的 fe/plugins 中生成一个 AuditLoader 文件夹（插件卸载后该目录自动删除）。若我们后续需要修改插件的配置，除卸载重装插件（推荐），也可替换该目录中的 auditloader.jar 或修改 plugin.conf，然后重启 FE 使修改生效。**



##### 5、卸载插件

在需要升级插件或者调整插件配置时，AuditLoader 插件也可视需求进行卸载，卸载命令的语法为：

```SQL
UNINSTALL PLUGIN plugin_name;
--plugin_name 即 SHOW PLUGINS 命令查到的插件 Name 信息，通常应为 AuditLoader。
```



##### 6、查看数据

在 AuditLoader 插件安装完成后，SQL 执行后的审计信息并不是实时入库。StarRocks 后台会按照配置文件 plugin.conf 中配置的参数，攒批 60 秒或 50M 执行一次 Stream Load 导入。测试等待期间，可简单执行几条 SQL 语句，看对应的审计数据是否能够正常入库，例如执行：

```sql
mysql> CREATE DATABASE test;
mysql> CREATE TABLE test.audit_test(c1 int,c2 int,c3 int,c4 date,c5 bigint) Distributed BY hash(c1) properties("replication_num" = "1");
mysql> insert into test.audit_test values(211014001,10001,13,'2021-10-14',1999),(211014002,10002,13,'2021-10-14',6999),(211015098,16573,19,'2021-10-15',3999);
```

等待 1 分钟左右，查看审计表数据：

```sql
mysql> select * from starrocks_audit_db__.starrocks_audit_tbl__;
```

**说明：通常数据都可以正确入库，若表内始终没有数据，可以检查配置文件 plugin.conf 中 IP、端口、用户权限、用户密码等信息是否正确。审计插件的日志会打印在各个 FE 的 fe.log 中，因此也可以在 fe.log 中检索关键字 `audit`，用排查 Stream Load 任务的思路来定位问题。**
