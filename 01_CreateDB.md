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






    
