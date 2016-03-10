### 01.Create customized database

    db2 "create database INTRO ON /data1, /data2 DBPATH ON /home/db2inst1 RESTRICTIVE"

This command uses the ON clause which indicates that the database will use automatic storage and all automatic tablespace
containers will be created under specified locations. The “DBPATH ON” clause indicates that database metadata along with log
control and data files will be created under: /home/db2inst1. If DBPATH is not specified these things will be created on the first
path found in the ON clause.
