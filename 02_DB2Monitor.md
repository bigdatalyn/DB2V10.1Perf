
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
