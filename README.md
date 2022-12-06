# fe-plugins-auditloader
### 插件说明：

StarRocks中所有SQL的审计日志保存在本地fe/log/fe.audit.log里，没有入库保存。为了方便对业务SQL进行分析，这里参考Apache Doris的审计插件代码，简单改造了一版适用于StarRocks的审计日志插件。逻辑上，StarRocks会在执行SQL后调用插件方法收集SQL的审计信息，审计信息的内容会在内存中攒批后基于Stream Load的方式导入至StarRocks表中。

##### 注意：

1、在使用插件时，随着StarRocks的迭代升级，审计日志fe.audit.log中的字段个数也在逐渐增多，所以不同的StarRocks版本需要使用对应版本的插件，同时在StarRocks中存放审计日志的表的创建语句也要随之调整，下面演示所用的建表语句适用于**StarRocks 2.4**版本。

2、在开发插件时，若发现StarRocks新迭代版本中的审计日志格式出现了变化，则需要替换工程中的fe-plugins-auditloader\lib\starrocks-fe.jar，同时修改代码中和字段相关的内容。



### 使用说明：

##### 1、创建内部表

我们首先在StarRocks中创建一个动态分区表，来保存审计日志中的数据。为了规范使用，建议为其单独创建一个数据库。

创建存放审计日志的数据库`starrocks_audit_db__`：

```SQL
create database starrocks_audit_db__;
```

在`starrocks_audit_db__`库创建`starrocks_audit_tbl__`表：

```SQL
CREATE TABLE starrocks_audit_db__.starrocks_audit_tbl__ (
  `queryId` VARCHAR(48) COMMENT "查询的唯一ID",
  `timestamp` DATETIME NOT NULL COMMENT "查询开始时间",
  `clientIp` VARCHAR(32) COMMENT "客户端IP",
  `user` VARCHAR(64) COMMENT "查询用户名",
  `authorizedUser` VARCHAR(64) COMMENT "用户唯一标识，即user_identity",
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
DUPLICATE KEY (`queryId`, `timestamp`, `clientIp`)
COMMENT "审计日志表"
PARTITION BY RANGE (`timestamp`) ()
DISTRIBUTED BY HASH (`queryId`) BUCKETS 3 
PROPERTIES (
  "dynamic_partition.time_unit" = "DAY",
  "dynamic_partition.start" = "-30",
  "dynamic_partition.end" = "3",
  "dynamic_partition.prefix" = "p",
  "dynamic_partition.buckets" = "3",
  "dynamic_partition.enable" = "true",
  "replication_num" = "3"
);
```

`starrocks_audit_tbl__`是动态分区表，我们在建表时没有直接创建分区，所以建表后需要等待后台动态分区调度进程调度后才会生成当天及后三天的分区。动态分区调度进程默认10分钟调度一次，我们可以先观察分区是否已经被创建，再进行后续操作，分区查看语句为：

```SQL
show partitions from starrocks_audit_db__.starrocks_audit_tbl__;
```



##### 2、修改配置文件

度盘中的审计插件名称为auditloader.zip，解压插件：

```SHELL
[root@node01 ~]# unzip auditloader.zip
Archive:  auditloader.zip
  inflating: auditloader.jar        
  inflating: plugin.conf            
  inflating: plugin.properties
```

这句命令会把zip里面的文件直接解压到当前目录，解压后可以得到插件中的三个文件：

`auditloader.jar`：插件代码打包的核心jar包

`plugin.conf`：插件配置文件，需根据集群信息修改

`plugin.properties`：插件属性文件，无需修改

根据我们实际的集群信息，修改配置文件`plugin.conf`：

```XML
### plugin configuration

# The max size of a batch, default is 50MB
max_batch_size=52428800

# The max interval of batch loaded, default is 60 seconds
max_batch_interval_sec=60

# the max stmt length to be loaded in audit table, default is 4096
max_stmt_length=4096

# StarRocks FE host for loading the audit, default is 127.0.0.1:8030.
# this should be the host port for stream load
frontend_host_port=127.0.0.1:8030

# Database of the audit table
database=starrocks_audit_db__

# Audit table name, to save the audit data.
table=starrocks_audit_tbl__

# StarRocks user. This user must have LOAD_PRIV to the audit table.
user=root

# StarRocks user's password
password=
```

修改完成后，再将上面的三个文件重新打包为zip包备用：

```SHELL
[root@node01 ~]# zip -q -m -r auditloader.zip auditloader.jar plugin.conf plugin.properties
```

**注意：这句命令会将需要打包的文件移动到auditloader.zip中，并覆盖该目录下原有的auditloader.zip文件。也即执行完打包命令后，该目录下只会保留一个最新的auditloader.zip插件包文件。**



##### 3、分发插件

将auditloader.zip分发至集群所有FE节点，各节点分发路径需要一致。例如我们都分发至StarRocks部署目录/opt/module/starrocks/下，也即auditloader.zip文件在集群所有FE节点的路径都为：

```
/opt/module/starrocks/auditloader.zip
```



##### 4、安装插件

StarRocks安装本地插件的语法为：

```sql
INSTALL PLUGIN FROM "/location/plugindemo.zip";
```

根据我们分发文件的路径修改命令后执行：

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
Description: load audit log to olap load, and user can view the statistic of queries
    Version: 1.0.4
JavaVersion: 1.8.0
  ClassName: com.starrocks.plugin.audit.AuditLoaderPlugin
     SoName: NULL
    Sources: /opt/module/starrocks/auditloader.zip
     Status: INSTALLED
 Properties: {}
```

我们可以看到当前有两个插件，其中Name属性为`AuditLoader`的插件即为我们刚才安装的审计日志插件，其状态为INSTALLED，表示已安装成功。

Name为`__builtin_AuditLogBuilder`的插件为StarRocks自己的日志插件，是用来打印审计日志到本地日志目录生成fe.audit.log的。

StarRocks自带的这个`__builtin_AuditLogBuilder`插件我们千万不要动它，咱们自己安装的插件则可以视需求随意卸载，卸载命令语法为：

```SQL
UNINSTALL PLUGIN plugin_name;
--plugin_name即SHOW PLUGINS命令查到的插件Name信息。
```

**注意：AuditLoader日志审计插件安装完成后，我们可以观察到fe/plugins/目录下生成了一个AuditLoader文件夹，这个文件夹相当于是StarRocks加载auditloader.zip插件后存放的临时目录。每次重启FE，StarRocks都会重新从我们安装插件时指定的路径中重新加载auditloader.zip文件。也是因此，前面分发至各FE节点的auditloader.zip文件我们千万不要移动或删除。**



##### 5、查看数据

AuditLoader插件安装完成后，审计日志文件中的数据并不是实时的入库。StarRocks后台会按照我们配置文件plugin.conf中配置的参数，攒批60秒或50M执行一次Stream Load导入。等待期间，我们可以简单执行几条SQL语句，看对应的审计数据是否能够正常入库，例如执行：

```sql
mysql> CREATE TABLE starrocks.audit_test(c1 int,c2 int,c3 int,c4 date,c5 bigint) Distributed BY hash(c1) properties("replication_num" = "1");
mysql> insert into starrocks.audit_test values(211014001,10001,13,'2021-10-14',1999),(211014002,10002,13,'2021-10-14',6999),(211015098,16573,19,'2021-10-15',3999);
```

等待1分钟左右，查看审计表数据：

```sql
mysql> select * from starrocks_audit_db__.starrocks_audit_tbl__;
```

正常来说数据都可以成功入库，如果表内始终没有数据，可以检查配置文件plugin.conf中信息配置是否正确，如果有误，可以参考前面的步骤修改配置文件，卸载插件后重新安装。或者我们也可以查看fe.log，用排查Stream Load任务的方式来定位问题。
