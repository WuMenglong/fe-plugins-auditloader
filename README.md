# fe-plugins-auditloader
本文档介绍如何通过插件 AuditLoader 在 StarRocks 内部管理审计信息。

### 插件说明：

在 StarRocks 中，所有的审计信息仅存储在日志文件 **fe/log/fe.audit.log** 中，无法直接通过 StarRocks 进行访问。AuditLoader 插件可实现审计信息的入库，让您在 StarRocks 内方便的通过 SQL 进行集群审计信息的查看和管理。安装 AuditLoader 插件后，StarRocks 在执行 SQL 后会自动调用 AuditLoader 插件收集 SQL 的审计信息，然后将审计信息在内存中攒批，最后基于 Stream Load 的方式导入至 StarRocks 表中。

**注意**：StarRocks 各个大版本的审计日志字段个数存在差异，为保证版本通用性，新版本的审计插件选取了各大版本中通用的日志字段进行入库。若业务中需要更完整的字段，可替换工程中的 `fe-plugins-auditloader\lib\starrocks-fe.jar` ，同时修改代码中与字段相关的内容后重新编译打包。



### 使用说明：

##### 1、创建内部表

首先在 StarRocks 中创建一个动态分区表，来保存审计日志中的数据。为了规范使用，建议为其单独创建一个数据库。

例如，创建存放审计日志的数据库 `starrocks_audit_db__`：

```SQL
create database starrocks_audit_db__;
```

在 `starrocks_audit_db__` 库创建 `starrocks_audit_tbl__` 表，表的属性部分可以视实际业务进行修改：

```SQL
CREATE TABLE starrocks_audit_db__.starrocks_audit_tbl__ (
  `queryId` VARCHAR(64) COMMENT "查询的唯一ID",
  `timestamp` DATETIME NOT NULL COMMENT "查询开始时间",
  `queryType` VARCHAR(12) COMMENT "查询类型（query, slow_query, connection）",
  `clientIp` VARCHAR(32) COMMENT "客户端IP",
  `user` VARCHAR(64) COMMENT "查询用户名",
  `authorizedUser` VARCHAR(64) COMMENT "用户唯一标识，既user_identity",
  `resourceGroup` VARCHAR(64) COMMENT "资源组名",
  `catalog` VARCHAR(32) COMMENT "数据目录名",
  `db` VARCHAR(96) COMMENT "查询所在数据库",
  `state` VARCHAR(8) COMMENT "查询状态（EOF，ERR，OK）",
  `errorCode` VARCHAR(512) COMMENT "错误码",
  `queryTime` BIGINT COMMENT "查询执行时间（毫秒）",
  `scanBytes` BIGINT COMMENT "查询扫描的字节数",
  `scanRows` BIGINT COMMENT "查询扫描的记录行数",
  `returnRows` BIGINT COMMENT "查询返回的结果行数",
  `cpuCostNs` BIGINT COMMENT "查询CPU耗时（纳秒）",
  `memCostBytes` BIGINT COMMENT "查询消耗内存（字节）",
  `stmtId` INT COMMENT "SQL语句增量ID",
  `isQuery` TINYINT COMMENT "SQL是否为查询（1或0）",
  `feIp` VARCHAR(128) COMMENT "执行该语句的FE IP",
  `stmt` VARCHAR(1048576) COMMENT "SQL原始语句",
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

**注意**：为便于对审计信息进行TTL（Time to Live）生命周期管理，通常推荐将表 `starrocks_audit_tbl__` 创建为动态分区表，上文的建表语句中没有显示的创建分区，所以建表后需要等待后台动态分区调度线程调度后才会生成当天及后三天的分区。动态分区调度进程默认 10 分钟调度一次（fe.conf  dynamic_partition_check_interval_seconds），您可以先观察分区是否已经被创建，再进行后续操作，分区查看命令为：

```SQL
show partitions from starrocks_audit_db__.starrocks_audit_tbl__;
```



##### 2、修改配置文件

审计插件包名称为 auditloader.zip，解压插件：

```SHELL
[root@node01 ~]# unzip auditloader.zip
Archive:  auditloader.zip
  inflating: auditloader.jar        
  inflating: plugin.conf            
  inflating: plugin.properties
```

该命令会把 zip 里面的文件直接解压到当前目录，解压后可以得到插件中的三个文件：

`auditloader.jar`：审计插件代码编译后得到的程序 jar 包。

`plugin.conf`：插件配置文件，用于提供插件底层进行 Stream Load 写入时的配置参数，需根据集群信息修改。通常只建议修改其中的 `user` 和 `password` 信息。

`plugin.properties`：插件属性文件，用于提供审计插件在 StarRocks 集群内的描述信息，无需修改。

根据实际的集群信息，修改配置文件 `plugin.conf`：

```XML
### plugin configuration

# The max size of a batch, default is 50MB.
max_batch_size=52428800

# The max interval of batch loaded, default is 60 seconds.
max_batch_interval_sec=60

# the max stmt length to be loaded in audit table, default is 1048576.
max_stmt_length=1048576

# StarRocks FE host for loading the audit, default is 127.0.0.1:8030.
# this should be the host port for stream load.
frontend_host_port=127.0.0.1:8030

# If the response time of a query exceed this threshold, it will be recored in audit table as slow_query.
qe_slow_log_ms=5000

# Database of the audit table.
database=starrocks_audit_db__

# Audit table name, to save the audit data.
table=starrocks_audit_tbl__

# StarRocks user. This user must have import permissions for the audit table.
user=root

# StarRocks user's password.
password=
```

**说明**：推荐使用参数 `frontend_host_port` 的默认配置，即 `127.0.0.1:8030` 。StarRocks 中各个 FE 是独立管理各自的审计信息的，在安装审计插件后，各个 FE 分别会启动各自的后台线程进行审计信息的获取攒批和 Stream load 写入。 `frontend_host_port` 配置项用于为插件后台 Stream Load 任务提供 http 协议的 IP 和端口，该参数不支持配置为多个值。其中，参数 IP 部分可以使用集群内任意某个 FE 的 IP，但并不推荐这样配置，因为若对应的 FE 出现异常，其他 FE 后台的审计信息写入任务也会因无法通信导致写入失败。推荐配置为默认的 `127.0.0.1:8030`，让各个 FE 均使用自身的 http 端口进行通信，以此规避其他 FE 异常时对通信的影响（当然，所有的写入任务最终都会被自动转发到 FE Leader 节点执行）。



修改完成后，再将上面的三个文件重新打包为 zip 包：

```SHELL
[root@node01 ~]# zip -q -m -r auditloader.zip auditloader.jar plugin.conf plugin.properties
```

**注意**：该命令会将需要打包的文件移动到 auditloader.zip 中，并覆盖该目录下原有的 auditloader.zip 文件。也即执行完打包命令后，该目录下只会保留一个最新的 auditloader.zip 插件包文件。



##### 3、分发插件

将 auditloader.zip 分发至集群所有 FE 节点，各节点分发路径需要一致。例如我们都分发至StarRocks部署目录 `/opt/module/starrocks/` 下，也即 auditloader.zip 文件在集群所有FE节点的路径都为：

```
/opt/module/starrocks/auditloader.zip
```

**说明**：也可将 auditloader.zip 分发至所有 FE 都可访问到的 http 服务中（例如 httpd 或 nginx），然后使用网络路径安装。



##### 4、安装插件

StarRocks安装本地插件的语法为：

```sql
INSTALL PLUGIN FROM "/location/plugindemo.zip";
```

若通过网络路径安装，还需要在安装命令的属性中提供插件压缩包的 md5 信息，语法示例：

```sql
INSTALL PLUGIN FROM "http://192.168.141.203/extra/auditloader.zip" PROPERTIES("md5sum" = "3975F7B880C9490FE95F42E2B2A28E2D");
```

以安装本地插件包为例，根据上文分发文件的路径修改命令后执行：

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
Description: Available for versions 2.3+. Load audit log to starrocks, and user can view the statistic of queries.
    Version: 4.0.0
JavaVersion: 1.8.0
  ClassName: com.starrocks.plugin.audit.AuditLoaderPlugin
     SoName: NULL
    Sources: /opt/module/starrocks/auditloader.zip
     Status: INSTALLED
 Properties: {}
```

可看到当前集群有两个插件，其中 Name 为 `AuditLoader` 的插件即为上文安装的审计日志插件，插件状态为 INSTALLED 表示已安装成功。Name 为 `__builtin_AuditLogBuilder` 的插件为 StarRocks 内置的审计插件，用来打印审计信息到本地日志目录生成  fe.audit.log。这两个插件的数据来源于 FE 的同一个方法，正常情况下审计日志表中的数据应与本地审计日志文件中的内容保持一致 。



##### 5、卸载插件

在需要升级插件或者调整插件配置时，已安装的 AuditLoader 插件也可视需求进行卸载，卸载命令的语法为：

```SQL
UNINSTALL PLUGIN plugin_name;
--plugin_name 即 SHOW PLUGINS 命令查到的插件 Name 信息，通常应为 AuditLoader。
```

**注意**：fe/plugins 是 StarRocks 外部插件的安装目录，在审计插件安装完成后，会在各个 FE 的 fe/plugins 中生成一个 AuditLoader 文件夹（插件卸载后该目录自动删除）。若我们后续需要修改插件的配置，除卸载重装插件（推荐），也可替换该目录中的 auditloader.jar 或修改 plugin.conf，然后重启 FE 使修改生效。



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



##### 7、异常排查

正常数据都可以正确入库，如果插件安装成功，同时动态分区创建成功后审计信息仍然长时间没导入至表内，您可以检查配置文件 plugin.conf 中 IP、端口、用户权限、用户密码等信息是否正确。AuditLoader 的日志会打印在各个 FE 的 fe.log 中，您也可以在 fe.log 中检索关键字 `audit`，用排查 Stream Load 任务的思路来定位问题。



##### 8、拓展用法

StarRocks审计表中支持的 `queryType` 类型包括：query、slow_query 和 connection。对于 query 和 slow_query，AuditLoader 插件使用 `plugin.conf` 中配置的 `qe_slow_log_ms` 时间来进行对比判断，SQL 执行时长大于 `qe_slow_log_ms` 的即为 slow_query，您可以以此进行集群慢 SQL 的统计。

对于 connection，StarRocks 3.0.6+ 版本支持在 fe.audit.log 中打印客户端连接时成功/失败的 connection 信息，您可以在 `fe.conf` 里配置 `audit_log_modules=slow_query, query, connection`，然后重启 FE 来进行启用。在启用 connection 信息后，AuditLoader 插件同样能采集到这类客户端连接信息并入库到表 `starrocks_audit_tbl__` 中，入库后该类信息对应的审计表的 `queryType` 字段即为 connection，您可以以此进行用户登录信息的审计。

