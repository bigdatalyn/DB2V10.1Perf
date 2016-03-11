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

Display connections that return the highest volume of data to clients, ordered by rows returned.

    db2 connect to db2pt
    db2 "SELECT application_handle, rows_returned, tcpip_send_volume,
    evmon_wait_time, total_peas, total_connect_request_time
    FROM TABLE(MON_GET_CONNECTION(cast(NULL as bigint), -2)) AS t
    ORDER BY rows_returned DESC"

List utilization of container file systems, ordered by highest utilization

    db2 "SELECT varchar(container_name, 65) as container_name, SUBSTR(fs_id,1,10)
    fs_id, fs_used_size, fs_total_size, CASE WHEN fs_total_size > 0
    THEN DEC(100*(FLOAT(fs_used_size)/FLOAT(fs_total_size)),5,2)
    ELSE DEC(-1,5,2) END as utilization
    FROM TABLE(MON_GET_CONTAINER('',-1)) AS t ORDER BY utilization DESC"

lists the activity on all tables accessed since the database was activated, aggregated across all database members, ordered by highest number of reads.

    db2 "SELECT varchar(tabschema,20) as tabschema, varchar(tabname,20) as
    tabname, sum(rows_read) as total_rows_read, sum(rows_inserted) as
    total_rows_inserted, sum(rows_updated) as total_rows_updated,
    sum(rows_deleted) as total_rows_deleted
    FROM TABLE(SYSPROC.MON_GET_TABLE('','',-2)) AS t
    GROUP BY tabschema, tabname
    ORDER BY total_rows_read DESC"

list a point-in-time view of both static and dynamic SQL statements in the database package cache.
List all the dynamic SQL statements from the database package cache ordered by the average CPU time.

    db2 "SELECT MEMBER, SECTION_TYPE, TOTAL_CPU_TIME/NUM_EXEC_WITH_METRICS as
    AVG_CPU_TIME, EXECUTABLE_ID FROM TABLE(SYSPROC.MON_GET_PKG_CACHE_STMT('D',
    NULL, NULL, -2)) as T WHERE T.NUM_EXEC_WITH_METRICS <> 0 ORDER BY
    AVG_CPU_TIME"

"D" stands for dynamic; "S" for static.

### 06.系统瓶颈

#### I/O Wait

use vmstat to detect the disk bottleneck by looking at the CPU wait time. Generally, if you see wait times here in excess of 25% you have an opportunity to improve overall throughput by reducing the overall I/O burden on the containers being taxed.

collect information like how many physical reads are occurring on the tablespace and how much read time is accumulating against the underlying container. 

Range partitioning is only one technique you can use to spread your data out across disk resources. 

Start the vmstat tool with a 2 second interval.

    vmstat 2

Take note of the cpu/wa column, this represents %cpu time in wait

Start the “iostat tool” with a 2 second interval

    iostat 2

%CPU Waiting iowaiting

take a snapshot of the tablespaces. Note how many physical reads are being done against the TS_RANGE1 tablespace. Then run the
script that will get the read times on a per container basis.

    db2 get snapshot for tablespaces on db2pt | more
    
Excessive I/O activity results from many different database operations; table and index scans, mismanaged or lack of memory to relieve I/O sort overflows, batch activity, excessive logging, temporary tables, lack of sufficient resources, and poor database designs, are some typical causes.













