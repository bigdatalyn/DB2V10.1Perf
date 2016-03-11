### 1.用表监控有：
系统级别/活动级别/数据级别

The request (REQ), activity (ACT) and Object Metrics (OBJ) are all set on (BASE) by default, which means all metrics are
collected for all requests executed on the data server.

db2 get db cfg for XXXX | grep -i mon

### 2.system monitor switches 

打开才能收集snapshot.

开关有实例级别/session

实例：

db2 get dbm cfg | grep -i DFT_MON

db2 update dbm cfg using dft_mon_bufpool on 设置完后需要重启才能有效

会话(session):

db2 get monitor switches

db2 update monitor switches using bufferpool off

db2 get monitor switches

db2 activate database XXXXX
-->数据库必须激活之后才会开始收集


### 3.数据库的监控

步骤：
创建event monitor---打开收集开关---访问收集到的数据（最后关闭开光）

event monitor的出力可以输出到表也可以输出到文件

    db2 "CREATE EVENT MONITOR EM_CHANGE_HIST
    FOR CHANGE HISTORY WHERE EVENT IN (DBMCFG, DDLALL, DDLDATA, DDLSQL)
    WRITE TO TABLE
    CHANGESUMMARY (TABLE CHG_SUMMARY_HISTORY),
    DDLSTMTEXEC (TABLE DDLSTM_HISTORY)
    AUTOSTART"

上面将创建下面的两个表保存收集到的信息（记录了DBM/DDL变换的结果）

    db2 "describe table chg_summary_history"
    db2 "describe table ddlstm_history"

下面语句查看什么时候那个app修改的

    db2 "SELECT SUBSTR(appl_name,1,20)
    appl_name,application_handle,event_timestamp, event_type,
    substr(system_authid,1,20) system_authid FROM chg_summary_history"

如果需要查看创建表/视图/type等操作：

    db2 "SELECT event_id, event_timestamp, SUBSTR(event_type,1,20) event_type,
    ddl_classification, SUBSTR(local_transaction_id,1,20) local_transaction_id,
    SUBSTR(stmt_text,1,50) stmt_text
    FROM ddlstm_history"

如果需要收集表的操作等，需要打开MON_OBJ_METRICS 的 EXTENDED（默认是based）

    db2 update DATABASE CONFIGURATION using MON_OBJ_METRICS EXTENDED

    db2 "CREATE USAGE LIST DOCUMENTSUL FOR TABLE   xxxxxx"


需要激活
    
    db2 "SET USAGE LIST DOCUMENTSUL STATE = ACTIVE"

查看是否激活了

    db2 "SELECT MEMBER, STATE, LIST_SIZE, USED_ENTRIES, WRAPPED
    FROM TABLE(MON_GET_USAGE_LIST_STATUS('DOCUMENTS', 'DOCUMENTSUL', -2))"

DOCUMENTS:表名
DOCUMENTSUL： usage listm名

关闭：
    
    db2 "SET USAGE LIST DOCUMENTSUL STATE = INACTIVE"

如何查看刚才收集的数据变化操作过程：

    db2 "SELECT MEMBER, EXECUTABLE_ID, NUM_REFERENCES, NUM_REF_WITH_METRICS,
    ROWS_READ, ROWS_INSERTED, ROWS_UPDATED, ROWS_DELETED
    FROM TABLE(MON_GET_TABLE_USAGE_LIST('DB2INST1', 'DOCUMENTSUL', -2))
    ORDER BY ROWS_READ DESC
    FETCH FIRST 10 ROWS ONLY"

再通过：executable_id确认具体的操作语句（01000000000000004F0000000000000000000000020020120402174639156732）

    db2 "SELECT SUBSTR(STMT_TEXT,1,40) STMT_TEXT
    FROM TABLE
    (MON_GET_PKG_CACHE_STMT(NULL,
    x'01000000000000004F0000000000000000000000020020120402174639156732', NULL, -
    2))"

### 4.event monitor收集信息：

收集到的信息输出到表，表必须是非分区表

收集的event type如下项目：

ACTIVITIES
BUFFERPOOLS
CHANGE HISTORY
CONNECTIONS
DATABASE
DEADLOCKS
LOCKING
PACKAGE CACHE
STATEMENTS
STATISTICS
TABLE
TABLESPACE
THRESHOLD VIOLATIONS
TRANSACTIONS
UNIT OF WORK

例子：

db2 "CREATE EVENT MONITOR myevmon FOR UNIT OF WORK WRITE TO TABLE"

db2 "CREATE EVENT MONITOR evm_test FOR STATEMENTS WRITE TO TABLE"

创建完之后会自动创建如下表：

connheader_evm_test
stmt_ test
subsection_ test
control_ evm_test

    db2 "describe table connheader_evm_test"
    db2 "describe table stmt_evm_test"
    db2 "describe table control_evm_test"
    db2 "set event monitor evm_test state 1"

如果只收集部分逻辑组（LOCK and PARTICIPANT logical groups）到表，例如：

    db2 "CREATE EVENT MONITOR mylocks FOR LOCKING WRITE TO TABLE   LOCK,LOCK_PARTICIPANTS"

测试：插入一些数据到表之后查看：Logical Data Grouping connheader

    db2 "select * from connheader_evm_test"

db2 "select substr(EVENT_MONITOR_NAME,1,20) EVENT_MONITOR_NAME,
substr(MESSAGE,1,20) MESSAGE, substr( MESSAGE_TIME,1,20) MESSAGE_TIME from
control_evm_test"


如何查看当前数据库有那些event monitor呢？

    db2 "SELECT SUBSTR(EVMONNAME,1,20) AS EVMON_NAME, TARGET_TYPE, SUBSTR(OWNER,1,20) OWNER FROM SYSCAT.EVENTMONITORS"

或者：

    db2 "SELECT SUBSTR(TYPE,1,20) AS EVENT_TYPE, SUBSTR(EVMONNAME,1,20) AS EVENT_MONITOR_NAME FROM SYSCAT.EVENTS ORDER BY TYPE"

eventmonitor表会随着数据库版本升级而有所改变，所以涉及到eventmonitor的表也需要升级

    db2 "call evmon_upgrade_tables(null, null, null, ?, ?, ?)"

部分表升级：

    db2 "call evmon_upgrade_tables(null,'ACTIVITIES', null, ?, ?, ?)"
    db2 "call evmon_upgrade_tables(null,'STATISTICS', null, ?, ?, ?)"

### 5.Applications

application的快照

    db2 list applications for database db2pt
    
关注：Appl. Handle 和 Application Id

查看有多少连接

    db2 list applications for database db2pt | grep DB2INST1 | wc

查看application’s status (monitoring element = appl_status).

    db2 list applications for database db2pt show detail | more

how many agents are “Executing” or in a “Lockwait” status.

具体application的信息：

    db2 list applications
    db2 list applications for database db2pt show detail | more
    db2 get snapshot for application applid <your_applid> | more

上面是通过snapshot来查看，现在通过SYSIBMADM.APPLICATIONS和db2pt工具视图来查看：

    db2 "select agent_id, substr(appl_name,1,10) as appl_name,
    substr(authid,1,20) as authid, appl_status from sysibmadm.applications where
    db_name = 'DB2PT'"

用db2pd

    db2pd -db db2pt -applications | more
    db2pd -db db2pt -edus | more
    db2pd -edus |more

显示每个连接返回的行数

    db2 connect to db2pt
    db2 "SELECT application_handle, rows_returned, tcpip_send_volume,
    evmon_wait_time, total_peas, total_connect_request_time
    FROM TABLE(MON_GET_CONNECTION(cast(NULL as bigint), -2)) AS t
    ORDER BY rows_returned DESC"

显示每个container的使用率

    db2 "SELECT varchar(container_name, 65) as container_name, SUBSTR(fs_id,1,10)
    fs_id, fs_used_size, fs_total_size, CASE WHEN fs_total_size > 0
    THEN DEC(100*(FLOAT(fs_used_size)/FLOAT(fs_total_size)),5,2)
    ELSE DEC(-1,5,2) END as utilization
    FROM TABLE(MON_GET_CONTAINER('',-1)) AS t ORDER BY utilization DESC"

从数据库激活来之后，读取表中数据多少行的统计（从高到低total_rows_read）

    db2 "SELECT varchar(tabschema,20) as tabschema, varchar(tabname,20) as
    tabname, sum(rows_read) as total_rows_read, sum(rows_inserted) as
    total_rows_inserted, sum(rows_updated) as total_rows_updated,
    sum(rows_deleted) as total_rows_deleted
    FROM TABLE(SYSPROC.MON_GET_TABLE('','',-2)) AS t
    GROUP BY tabschema, tabname
    ORDER BY total_rows_read DESC"

从数据库字典package缓存中获取SQL，耗费平均CPU时间的排序

    db2 "SELECT MEMBER, SECTION_TYPE, TOTAL_CPU_TIME/NUM_EXEC_WITH_METRICS as
    AVG_CPU_TIME, EXECUTABLE_ID FROM TABLE(SYSPROC.MON_GET_PKG_CACHE_STMT('D',
    NULL, NULL, -2)) as T WHERE T.NUM_EXEC_WITH_METRICS <> 0 ORDER BY
    AVG_CPU_TIME"

"D" stands for dynamic; "S" for static.

### 06.系统瓶颈

#### I/O Wait

通过vmstat的CPU wait time来观察disk瓶颈，一般wait times超过25%以上可以有机会减少I/O的整体负荷 来提高整体吞吐量

收集每个表空间的的physical reads来统计container的负荷

可以用Range partitioning范围分区来扩展disk的并行度

没2秒收集一次vmstat

    vmstat 2

关注cpu/wa 列, 表示 %cpu time 在等待

没2秒收集iostat

    iostat 2

关注：%CPU Waiting iowaiting

表空间快照获取 Bufferpool data physical reads的信息：

    db2 get snapshot for tablespaces on db2pt | more

每个container的（MON_GET_CONTAINER)h获取pool read  time的排序（从高到底）


Note: Since actual disk resources for both containers are the same, you will not see a big difference in CPU Wait time (maybe a little bit, attributed to multiple prefetchers working). The emphasis here is that you will see that the I/O is indeed being spread between the containers with a snapshot of the tablespaces and the MON_GET_CONTAINERS table function.

通过修改为范围分区表前后的数据

### 07.数据库IO的监控

I/O活动主要来自于：

表和索引的扫描
排序溢出
batch执行
大量使用临时表空间
不够系统资源（系统瓶颈）
数据库设计不合理等其他原因

怎样收集潜在或者当前：IO的热点

关注项目：
Rows_Read ： 为了返回request读取了多少行（评估是否需要增加额外的索引）

Rows_Returned ：返回给应用的行数，跟rows_read可以评估读的有效因子，也可以用来评估网络流量（在层次架构中）

Overflow_Accesses ：读写表时候，有多少的溢出行，表示有数据碎片，可以通过 reorg来重组表，一般发生在更新表中有varchar列的数据或者是alter table修改表结构导致的

Rows_Written :在表中的 I U D的行数，如果太高值的话需要runstat收集最新统计信息

Direct_Reads ：没用使用buffer的读次数，如：LOBS，LONG VARCHARS， Backups等，，可以使打开filesytem cachingk开关来减少直接读的开销

Direct_Read_Time ：没事用buffer直接读所消耗的时间，单位：miliseconds

Direct_Read_Requests ：直接读一个或多个sector数据的的次数

Pool_Data_P_Reads ：从磁盘container中读取多少data page到bufferpool

Pool_Index_P_Reads :从磁盘container中读取多少index page到bufferpool

Pool_Temp_Data_P_Reads ： 临时表空间

Pool_Temp_Index_P_Reads ：临时表空间

获取IO想关的信息

    db2 get snapshot for database on db2pt | egrep -i 'read|rows|timestamp' | more

从第一次数据连接或者reset timestamp以来计数的信息
可以快速的评估数据库的读写活动，逻辑和物理读，直接读写

#### 读的有效性

每个事务的读：Reads per Transaction

you see a result of 110 reads/trx, a fairly good rate for an OLTP system. Under 10 would be very good, over 50 worth investigating.

每个app读取了多少行返回了多少行 ：the rows read for every row returned to the applications.

每个container读取的时间（那个container在高负荷中）

#### 检索数据库状态

通过下面语句获得：

    db2 "SELECT SUBSTR(DB_NAME, 1, 20) AS DB_NAME, DB_STATUS, SERVER_PLATFORM,
    DB_LOCATION, DB_CONN_TIME, DBPARTITIONNUM FROM SYSIBMADM.SNAPDB ORDER BY
    DBPARTITIONNUM"

或者：

    db2 "SELECT SUBSTR(DB_NAME, 1, 20) AS DB_NAME, DB_STATUS, SERVER_PLATFORM,
    DB_LOCATION, DB_CONN_TIME FROM TABLE(SNAP_GET_DB(CAST (NULL AS VARCHAR(128)),
    -2)) AS T"


### 08.Bufferpool的监控

通过收集查看buffer的命中率来评估

    db2 alter bufferpool bpmontune size 5000
    db2 -tvf get_bphits_all.db2

修改查看命中率

或者改为automatic

    db2 alter bufferpool bpmontune size automatic

### 09.db2top

db2top -d XXXX

操作：

O is used to look at current settings for db2top, 
k is used to toggle between actual and delta values and 
C enables the capturing of the session into a file that can be replayed and thus reanalyzed.

Tables (T), Bufferpools (b), and finally Dynamic SQL (D).


• Use the Enter Key to exit Help.
• T to display the most active tables, you should see the Delta view which is representative of the activity
   during a tuning interval.
• k to toggle to Actual view where cumulative results are seen.

active tables and what kind of I/O is occurring.

• R to reset the snapshot counters then y, enter to confirm
The Actual counters will be reset to zero.

• b to see Bufferpool activity including Bufferpool Hit Ratio.

• D to see recent dynamic SQL activity against the database.
• f to freeze the screen.
• Pick out a query to “tune”. Use your mouse to highlight the SQL_Statement Hash Value and copy the
       value on to the clipboard.

收集到的sql可以就行基准测试：

    db2batch –d db2pt –f mySQL.out ( to invoke db2batch get benchmark results).





