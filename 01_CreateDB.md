### 01.Create customized database

    db2 "create database INTRO ON /data1, /data2 DBPATH ON /home/db2inst1 RESTRICTIVE"

This command uses the ON clause which indicates that the database will use automatic storage and all automatic tablespace
containers will be created under specified locations. The “DBPATH ON” clause indicates that database metadata along with log
control and data files will be created under: /home/db2inst1. If DBPATH is not specified these things will be created on the first
path found in the ON clause.

### 02.Change the path to a completely separate location.

    db2 get db cfg for intro |grep -i "path to log"
    
The home directory is acceptable for database metadata but leaving the log data and control files in the same location
is not the best for transactional performance.

    db2 "update db cfg for intro using NEWLOGPATH /logs"

    db2inst1@cocbase:~> db2 "update db cfg for intro using NEWLOGPATH /logs"
    DB20000I  The UPDATE DATABASE CONFIGURATION command completed successfully.
    db2inst1@cocbase:~> db2 get db cfg for intro |grep -i "path to log"
     Changed path to log files                  (NEWLOGPATH) = /logs/NODE0000/LOGSTREAM0000/
     Path to log files                                       = /home/db2inst1/db2inst1/NODE0000/SQL00001/LOGSTREAM0000/
    db2inst1@cocbase:~> 

### 03.RESTRICTIVE key word

This can be checked by displaying the
database configuration file with the “get db cfg” command under Restrict Access. Without it, db2 catalog information is readable by PUBLIC, i.e. anyone connected to the database. This will revoke the authorization to PUBLIC and make the database more secure. See DB2 Information center for other restrictions.

### 04.Registry Variables

#### DB2_MEM_TUNING_RANGE

This registry variable is related to self tuning memory manager (STMM). If STMM is being used, it will take more memory, as
needed, on the server as long as there is free memory available. This may cause problems on systems where you need to
leave some memory for another application or for the system.
db2set DB2_MEM_TUNING_RANGE=MINFREE, MAXFREE
MINFREE and MAXFREE are percentage numbers. 

    db2set DB2_MEM_TUNING_RANGE=40,80

#### DB2MEMDISCLAIM

DB2 agents may have some associated paging space. This paging space may remain allocated even after the associated
memory has been freed depending on the platform and virtual memory tunings. DB2MEMDISCLAIM, when set, indicates that
explicit requests should be made by DB2 to free associated paging space as well. This leads to smaller paging space
requirements and possibly less disk activity from paging. On systems where memory is tight and paging space is being used,
this may provide a performance improvement.

    db2set DB2MEMDISCLAIM=YES
    db2stop force
    db2start

#### DB2_SMS_TRUNC_TMPTABLE_THRESH

By default, when a temporary table is not needed it is truncated to zero pages. If there is a lot of temp table activity, this can degrade performance. Using this registry variable, one can specify the number of extents that should be preserved upon
truncation so that when the table is reused there are pre allocated extents available for use.

http://www.ibm.com/developerworks/cn/data/library/techarticle/dm-1208hanrb/index.html
在系统中设置 DB2_SMS_TRUNC_TMPTABLE_THRESH 注册变量。默认情况下，当系统临时表空间中的对象被删除时，先前为其分配的所有文件都会被回收。当下次需要分配新空间时，会重新进行分配。不断重复的分配 / 释放也会带来很多性能方面的开销。我们可以通过设置这个注册变量，来保证临时表空间中已经分配的空间不会在对应的对象被删除时完全回收。这样可以保留一部分可用空间（保留的大小由该变量的值决定）供我们下次使用，因此减少了重新分配这些空间带来的种种开销。 

10 extents should be preserved when truncating any temp table

    db2set DB2_SMS_TRUNC_TMPTABLE_THRESH=10
    db2stop force
    db2start
    
### 05.Buffer pools and Tablespaces


    db2 connect to intro
    db2 list tablespaces show detail

db2inst1@cocbase:~/sqllib> db2 connect to intro

   Database Connection Information

 Database server        = DB2/LINUXX8664 10.1.0
 SQL authorization ID   = DB2INST1
 Local database alias   = INTRO

db2inst1@cocbase:~/sqllib> db2 list tablespaces show detail

           Tablespaces for Current Database

 Tablespace ID                        = 0
 Name                                 = SYSCATSPACE
 Type                                 = Database managed space
 Contents                             = All permanent data. Regular table space.
 State                                = 0x0000
   Detailed explanation:
     Normal
 Total pages                          = 32768
 Useable pages                        = 32760
 Used pages                           = 25088
 Free pages                           = 7672
 High water mark (pages)              = 25088
 Page size (bytes)                    = 4096
 Extent size (pages)                  = 4
 Prefetch size (pages)                = 8
 Number of containers                 = 2

 Tablespace ID                        = 1
 Name                                 = TEMPSPACE1
 Type                                 = System managed space
 Contents                             = System Temporary data
 State                                = 0x0000
   Detailed explanation:
     Normal
 Total pages                          = 2
 Useable pages                        = 2
 Used pages                           = 2
 Free pages                           = Not applicable
 High water mark (pages)              = Not applicable
 Page size (bytes)                    = 4096
 Extent size (pages)                  = 32
 Prefetch size (pages)                = 64
 Number of containers                 = 2

 Tablespace ID                        = 2
 Name                                 = USERSPACE1
 Type                                 = Database managed space
 Contents                             = All permanent data. Large table space.
 State                                = 0x0000
   Detailed explanation:
     Normal
 Total pages                          = 8192
 Useable pages                        = 8128
 Used pages                           = 96
 Free pages                           = 8032
 High water mark (pages)              = 96
 Page size (bytes)                    = 4096
 Extent size (pages)                  = 32
 Prefetch size (pages)                = 64
 Number of containers                 = 2

db2inst1@cocbase:~/sqllib> 

• SYSCATSPACE (catalog data)
• TEMPSPACE1 (system temp)
• USERSPACE1 (user data)
USERSPACE1 is the default tablespace created and it has a page size of 4k. DB2 supports four page sizes: 4k, 8k, 16k and
32k. You may have tables with large rows that do not fit on a 4k page, so tablespaces with larger page sizes should also be
created. However, larger page size tablespaces need buffer pools with matching page size.


    db2 "select substr(BPNAME, 1,25) as BPNAME, NPAGES, PAGESIZE FROM SYSCAT.BUFFERPOOLS"

NPAGES value of -2 indicates that IBMDEFAULTBP buffer pool has been enabled for automatic tuning.

#### Create customized Buffer pools and Tablespaces

    db2 create bufferpool bp8k size automatic pagesize 8k
    db2 create bufferpool bp16k size automatic pagesize 16k
    db2 create bufferpool bp32k size automatic pagesize 32k
    db2 create bufferpool tempbp size automatic pagesize 4k

verify the results

    db2 "select substr(BPNAME, 1,25) as BPNAME, NPAGES, PAGESIZE FROM SYSCAT.BUFFERPOOLS"

PNAME                    NPAGES      PAGESIZE   
------------------------- ----------- -----------
IBMDEFAULTBP                       -2        4096
BP8K                               -2        8192
BP16K                              -2       16384
BP32K                              -2       32768
TEMPBP                             -2        4096

5 record(s) selected.

    db2 create tablespace tbsp8k pagesize 8k bufferpool bp8k
    db2 create tablespace tbsp16k pagesize 16k bufferpool bp16k
    db2 create tablespace tbsp32k pagesize 32k bufferpool bp32k
    db2 "select substr(bpname,1,25) as BPNAME, substr(tbspace,1,25) as TBSPACE  from syscat.bufferpools B ,
    syscat.tablespaces T where B.BUFFERPOOLID=T.BUFFERPOOLID"


BPNAME                    TBSPACE                  
------------------------- -------------------------
IBMDEFAULTBP              SYSCATSPACE              
IBMDEFAULTBP              SYSTOOLSPACE             
IBMDEFAULTBP              USERSPACE1               
IBMDEFAULTBP              TEMPSPACE1               

4 record(s) selected.

EMPBP buffer pool exists but is not beingused by TEMPSPACE1

    db2 alter tablespace TEMPSPACE1 bufferpool TEMPBP
    db2 "select substr(bpname,1,25) as BPNAME, substr(tbspace,1,25) as TBSPACE  from syscat.bufferpools B ,
    syscat.tablespaces T where B.BUFFERPOOLID=T.BUFFERPOOLID"

BPNAME                    TBSPACE                  
------------------------- -------------------------
IBMDEFAULTBP              SYSCATSPACE              
IBMDEFAULTBP              SYSTOOLSPACE             
IBMDEFAULTBP              USERSPACE1               
TEMPBP                    TEMPSPACE1               

4 record(s) selected.


    



    
