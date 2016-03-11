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
















