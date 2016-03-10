
createDB.db2

    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> cat createDB.db2
    force applications all;
    drop database db2pt;
    create database db2pt automatic storage yes on '/home/db2inst1' dbpath on '/home/db2inst1' using codeset utf-8 territory us collate using system pagesize 4096;
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 

dftmonOff.db2 

    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> cat ~/Documents/LabScripts/db2pt/essential/db2scripts/dftmonOff.db2 
    update dbm cfg using dft_mon_bufpool off;
    update dbm cfg using dft_mon_sort off;
    update dbm cfg using dft_mon_table off;
    update dbm cfg using dft_mon_lock off;
    update dbm cfg using dft_mon_stmt off;
    update dbm cfg using dft_mon_uow off;
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 

Turn off system monitor switches and restart the instance.
  
    db2 -tvf createDB.db2
    db2 -tvf dftmonOff.db2
    db2stop force
    db2start
    db2 terminate


DB2® Version 10 provides new monitor elements that enable you to perform more granular monitoring, without using the
monitor switches or snapshot interfaces. Database-wide monitoring control is provided by database configuration parameters.
With the new monitor elements and infrastructure, you can use SQL statements to efficiently collect monitor data from memory
to determine whether specific aspects of the system are working correctly and to help you diagnose performance problems.
With the new access methods, you can get the data you need without using the snapshot interfaces. The increased monitoring
granularity gives you more control over the data collection process; collect the data you want from the source you want. The
monitoring framework also reduces performance monitoring overhead to a minimum. Though the monitoring framework was
designed to work with the Performance Optimization feature pack, it can still be used with the default workloads that
are predefined for each database created.

### Getting the Monitor Parameters

    db2start
    db2 get db cfg for db2pt | grep -i MON
    
The request (REQ), activity (ACT) and Object Metrics (OBJ) are all set on (BASE) by default, which means all metrics are
collected for all requests executed on the data server.

    db2inst1@cocbase:~> 
       Monitor Collect Settings
     Request metrics                       (MON_REQ_METRICS) = BASE
     Activity metrics                      (MON_ACT_METRICS) = BASE
     Object metrics                        (MON_OBJ_METRICS) = BASE
     Unit of work events                      (MON_UOW_DATA) = NONE
       UOW events with package list        (MON_UOW_PKGLIST) = OFF
       UOW events with executable list    (MON_UOW_EXECLIST) = OFF
     Lock timeout events                   (MON_LOCKTIMEOUT) = NONE
     Deadlock events                          (MON_DEADLOCK) = WITHOUT_HIST
     Lock wait events                         (MON_LOCKWAIT) = NONE
     Lock wait event threshold               (MON_LW_THRESH) = 5000000
     Number of package list entries         (MON_PKGLIST_SZ) = 32
     Lock event notification level         (MON_LCK_MSG_LVL) = 1
    db2inst1@cocbase:~> 

turn on maximum collection for lock wait and deadlock monitoring and set the threshold for lockwaits to 3 seconds.

    db2 update db cfg for db2pt using mon_lockwait hist_and_values
    db2 update db cfg for db2pt using mon_lw_thresh 3000000
    db2 update db cfg for db2pt using mon_deadlock hist_and_values

example:

    db2inst1@cocbase:~> db2 update db cfg for db2pt using mon_lockwait hist_and_values
    DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
    db2inst1@cocbase:~> db2 update db cfg for db2pt using mon_lw_thresh 3000000
    DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
    db2inst1@cocbase:~> db2 update db cfg for db2pt using mon_deadlock hist_and_values
    DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
    db2inst1@cocbase:~> db2 get db cfg for db2pt | grep -i mon
     Monitor Collect Settings
     Request metrics                       (MON_REQ_METRICS) = BASE
     Activity metrics                      (MON_ACT_METRICS) = BASE
     Object metrics                        (MON_OBJ_METRICS) = BASE
     Unit of work events                      (MON_UOW_DATA) = NONE
       UOW events with package list        (MON_UOW_PKGLIST) = OFF
       UOW events with executable list    (MON_UOW_EXECLIST) = OFF
     Lock timeout events                   (MON_LOCKTIMEOUT) = NONE
     Deadlock events                          (MON_DEADLOCK) = HIST_AND_VALUES
     Lock wait events                         (MON_LOCKWAIT) = HIST_AND_VALUES
     Lock wait event threshold               (MON_LW_THRESH) = 3000000
     Number of package list entries         (MON_PKGLIST_SZ) = 32
     Lock event notification level         (MON_LCK_MSG_LVL) = 1
    db2inst1@cocbase:~> 
























    
        
    db2 connect to db2pt
    db2 "CREATE EVENT MONITOR EM_CHANGE_HIST
    FOR CHANGE HISTORY WHERE EVENT IN (DBMCFG, DDLALL, DDLDATA, DDLSQL)
    WRITE TO TABLE
    CHANGESUMMARY (TABLE CHG_SUMMARY_HISTORY),
    DDLSTMTEXEC (TABLE DDLSTM_HISTORY)
    AUTOSTART"
    
    db2 "describe table chg_summary_history"
    db2 "describe table ddlstm_history"
    
    
    db2 update dbm cfg using dft_mon_bufpool off
    db2 update dbm cfg using dft_mon_lock off
    db2 update dbm cfg using dft_mon_stmt off
    
    
    db2stop force
    db2start
    db2 connect to db2pt
    db2 "SELECT SUBSTR(appl_name,1,20)
    appl_name,application_handle,event_timestamp, event_type,
    substr(system_authid,1,20) system_authid FROM chg_summary_history"
    
    
    db2 update dbm cfg using dft_mon_bufpool on
    db2 update dbm cfg using dft_mon_lock on
    db2 update dbm cfg using dft_mon_stmt on
    db2stop force
    db2start
    db2 connect to db2pt
    db2 "SELECT SUBSTR(appl_name,1,20)
    appl_name,application_handle,event_timestamp, event_type,
    SUBSTR(system_authid,1,20) system_authid FROM chg_summary_history"
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> cat create_objects_evm.sql
    CREATE TABLE documents (summary VARCHAR(1000), repor VARCHAR(2000));
    
    CREATE TYPE image AS BLOB (10M);
    
    CREATE VIEW v_documents AS SELECT * FROM documents;
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> db2 -tvf create_objects_evm.sql
    CREATE TABLE documents (summary VARCHAR(1000), repor VARCHAR(2000))
    DB20000I  The SQL command completed successfully.
    
    CREATE TYPE image AS BLOB (10M)
    DB20000I  The SQL command completed successfully.
    
    CREATE VIEW v_documents AS SELECT * FROM documents
    DB20000I  The SQL command completed successfully.
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 
    
    
    db2 "SELECT event_id, event_timestamp, SUBSTR(event_type,1,20) event_type,
    ddl_classification, SUBSTR(local_transaction_id,1,20) local_transaction_id,
    SUBSTR(stmt_text,1,50) stmt_text
    FROM ddlstm_history"
    
    
    > ddl_classification, SUBSTR(local_transaction_id,1,20) local_transaction_id,
    > SUBSTR(stmt_text,1,50) stmt_text
    > FROM ddlstm_history"
    
    EVENT_ID             EVENT_TIMESTAMP            EVENT_TYPE           DDL_CLASSIFICATION             LOCAL_TRANSACTION_ID STMT_TEXT                                         
    -------------------- -------------------------- -------------------- ------------------------------ -------------------- --------------------------------------------------
                       2 2016-03-10-11.28.16.077172 DDLSTMTEXEC          DATA                           0000000000000162     CREATE TABLE documents (summary VARCHAR(1000), rep
                       4 2016-03-10-11.28.16.399917 DDLSTMTEXEC          SQL                            0000000000000163     CREATE TYPE image AS BLOB (10M)                   
                       6 2016-03-10-11.28.16.530462 DDLSTMTEXEC          SQL                            0000000000000165     CREATE VIEW v_documents AS SELECT * FROM documents
    
      3 record(s) selected.
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 
    
    =================================
    
    
    
    
    
    db2 update DATABASE CONFIGURATION using MON_OBJ_METRICS EXTENDED
    
    db2 "CREATE USAGE LIST DOCUMENTSUL FOR TABLE DOCUMENTS"
    
    db2 "SET USAGE LIST DOCUMENTSUL STATE = ACTIVE"
    
    db2 "SELECT MEMBER, STATE, LIST_SIZE, USED_ENTRIES, WRAPPED
    FROM TABLE(MON_GET_USAGE_LIST_STATUS('DOCUMENTS', 'DOCUMENTSUL', -2))"
    
    db2 -tvf create_data.db2

    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> db2 "SET USAGE LIST DOCUMENTSUL STATE = ACTIVE"
    DB20000I  The SQL command completed successfully.
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> cat create_data.db2
    INSERT INTO documents (summary, repor) VALUES ('10','1');
    INSERT INTO documents (summary, repor) VALUES ('20','2');
    INSERT INTO documents (summary, repor) VALUES ('30','3');
    INSERT INTO documents (summary, repor) VALUES ('40','4');
    INSERT INTO documents (summary, repor) VALUES ('50','5');
    INSERT INTO documents (summary, repor) VALUES ('60','6');
    INSERT INTO documents (summary, repor) VALUES ('70','7');
    INSERT INTO documents (summary, repor) VALUES ('80','8');
    INSERT INTO documents (summary, repor) VALUES ('90','9');
    INSERT INTO documents (summary, repor) VALUES ('100','10');
    INSERT INTO documents (summary, repor) VALUES ('110','101');
    INSERT INTO documents (summary, repor) VALUES ('120','102');
    INSERT INTO documents (summary, repor) VALUES ('130','103');
    INSERT INTO documents (summary, repor) VALUES ('140','104');
    INSERT INTO documents (summary, repor) VALUES ('150','105');
    UPDATE documents SET summary = '100' WHERE repor = '1';
    UPDATE documents SET summary = '300' WHERE repor = '3';
    UPDATE documents SET summary = '500' WHERE repor = '5';
    UPDATE documents SET summary = '700' WHERE repor = '7';
    UPDATE documents SET summary = '900' WHERE repor = '9';
    DELETE documents WHERE repor = '101';
    DELETE documents WHERE repor = '102';
    DELETE documents WHERE repor = '103';
    DELETE documents WHERE repor = '104';
    DELETE documents WHERE repor = '105';db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> db2 -tvf create_data.db2
    INSERT INTO documents (summary, repor) VALUES ('10','1')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('20','2')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('30','3')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('40','4')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('50','5')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('60','6')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('70','7')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('80','8')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('90','9')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('100','10')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('110','101')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('120','102')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('130','103')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('140','104')
    DB20000I  The SQL command completed successfully.
    
    INSERT INTO documents (summary, repor) VALUES ('150','105')
    DB20000I  The SQL command completed successfully.
    
    UPDATE documents SET summary = '100' WHERE repor = '1'
    DB20000I  The SQL command completed successfully.
    
    UPDATE documents SET summary = '300' WHERE repor = '3'
    DB20000I  The SQL command completed successfully.
    
    UPDATE documents SET summary = '500' WHERE repor = '5'
    DB20000I  The SQL command completed successfully.
    
    UPDATE documents SET summary = '700' WHERE repor = '7'
    DB20000I  The SQL command completed successfully.
    
    UPDATE documents SET summary = '900' WHERE repor = '9'
    DB20000I  The SQL command completed successfully.
    
    DELETE documents WHERE repor = '101'
    DB20000I  The SQL command completed successfully.
    
    DELETE documents WHERE repor = '102'
    DB20000I  The SQL command completed successfully.
    
    DELETE documents WHERE repor = '103'
    DB20000I  The SQL command completed successfully.
    
    DELETE documents WHERE repor = '104'
    DB20000I  The SQL command completed successfully.
    
    DELETE documents WHERE repor = '105'
    DB20000I  The SQL command completed successfully.
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> db2 "SET USAGE LIST DOCUMENTSUL STATE = INACTIVE"
    DB20000I  The SQL command completed successfully.
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> db2 "SELECT MEMBER, EXECUTABLE_ID, NUM_REFERENCES, NUM_REF_WITH_METRICS,
    ROWS_READ, ROWS_INSERTED, ROWS_UPDATED, ROWS_DELETED
    FROM TABLE(MON_GET_TABLE_USAGE_LIST('DB2INST1', 'DOCUMENTSUL', -2))
    ORDER BY ROWS_READ DESC
    FETCH FIRST 10 ROWS ONLY"
    
    MEMBER EXECUTABLE_ID                                                       NUM_REFERENCES       NUM_REF_WITH_METRICS ROWS_READ            ROWS_INSERTED        ROWS_UPDATED         ROWS_DELETED        
    ------ ------------------------------------------------------------------- -------------------- -------------------- -------------------- -------------------- -------------------- --------------------
         0 x'0100000000000000320000000000000000000000020020160310113417021566'                    1                    1                   15                    0                    1                    0
         0 x'0100000000000000330000000000000000000000020020160310113417022984'                    1                    1                   15                    0                    1                    0
         0 x'0100000000000000340000000000000000000000020020160310113417025763'                    1                    1                   15                    0                    1                    0
         0 x'0100000000000000350000000000000000000000020020160310113417028779'                    1                    1                   15                    0                    1                    0
         0 x'0100000000000000360000000000000000000000020020160310113417032999'                    1                    1                   15                    0                    1                    0
         0 x'0100000000000000370000000000000000000000020020160310113417036493'                    1                    1                   15                    0                    0                    1
         0 x'0100000000000000380000000000000000000000020020160310113417040695'                    1                    1                   14                    0                    0                    1
         0 x'0100000000000000390000000000000000000000020020160310113417044082'                    1                    1                   13                    0                    0                    1
         0 x'01000000000000003A0000000000000000000000020020160310113417047094'                    1                    1                   12                    0                    0                    1
         0 x'01000000000000003B0000000000000000000000020020160310113417049794'                    1                    1                   11                    0                    0                    1
    
      10 record(s) selected.
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 

    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> d
    FROM TABLE
    (MON_GET_PKG_CACHE_STMT(NULL,
    x'01000000000000003A0000000000000000000000020020160310113417047094', 
    2))"
    
    STMT_TEXT                               
    ----------------------------------------
    DELETE documents WHERE repor = '104'    
    
      1 record(s) selected.
    
    db2inst1@cocbase:~/Documents/LabScripts/db2pt/essential/db2scripts> 
    
    
    
    
    

    
    

    
    
    
    
    
    
    
    
    

  

