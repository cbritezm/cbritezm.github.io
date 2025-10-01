---
title: "Oracle RMAN Recover Table"
date: 2018-08-11 13:00:00 -0300
categories: [Database, Oracle ]
tags: [Rman, sql]
author: Cristhian Britez
description: "Use new RMAN feature. Learn how to recover from oracle databas backup using rman."
pinned: true # Optional: to pin the post to the homepage
---

Today I would like to show you how to recover tables with RMAN, this is a new feature on Oracle 12c database. Check the oficial documentation here: Recovering Tables and Table Partitions from RMAN Backups To start let me explain you the current environment where I working on: A virtual machine with Oracle 12c and ASM, one CDB and One PDB, as backup solution I have installed another virtual machine with Oracle Secure Backup and MHVTL as virtual media manager. We will recover a table from PDB.

# How To?
This procedure is as easy as telling RMAN to recover a table from database, it will create a temporary instance, restore & recover the database and then export the table, you can choose to import it directly to the database or to keep the dumpfile. All of these, orchestrated by RMAN.

To recover table from a media manager you have to configure your sbt channel with the default environment, allocating channel inside a run block is not supported when recovering tables. My current RMAN configuration is:

```RMAN

[oracle@grid12c backups]$ rman target /

Recovery Manager: Release 12.1.0.2.0 - Production on Fri Aug 10 23:03:31 2018

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.

connected to target database: ORACDB (DBID=2649666862)

RMAN> show all;

using target database control file instead of recovery catalog
RMAN configuration parameters for database with db_unique_name ORACDB are:
CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
CONFIGURE BACKUP OPTIMIZATION OFF; # default
CONFIGURE DEFAULT DEVICE TYPE TO DISK; # default
CONFIGURE CONTROLFILE AUTOBACKUP ON; # default
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F'; # default
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE SBT_TAPE TO '%F'; # default
CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO BACKUPSET; # default
CONFIGURE DEVICE TYPE SBT_TAPE PARALLELISM 1 BACKUP TYPE TO BACKUPSET; # default
CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE SBT_TAPE TO 1; # default
CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE SBT_TAPE TO 1; # default
CONFIGURE CHANNEL DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(OB_MEDIA_FAMILY=RMAN-DEFAULT)';
CONFIGURE MAXSETSIZE TO UNLIMITED; # default
CONFIGURE ENCRYPTION FOR DATABASE OFF; # default
CONFIGURE ENCRYPTION ALGORITHM 'AES128'; # default
CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE ; # default
CONFIGURE RMAN OUTPUT TO KEEP FOR 7 DAYS; # default
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE; # default
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/u01/app/oracle/product/12.1.0/dbhome_1/dbs/snapcf_oracdb.f'; # default

RMAN> exit

```

If you don't have configured sbt channel you will get this error from RMAN:
RMAN-06034: at least 1 channel must be allocated to execute this command. Check docid: 2105325.1 Now, I will create and delete a table on hr schema:

```SQL

SQL> create table employees_2 as select * from employees;
Table created.
SQL> insert into employees_2 select * from employees;
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> /
107 rows created.
SQL> insert into employees_2 select * from employees_2;
963 rows created.
SQL> /
1926 rows created.
SQL> /
3852 rows created.
SQL> /
7704 rows created.
SQL> commit;
```

After creating the table I have executed a full backup of the database and archivelogs. Now I am going to delete the table from hr schema.

```sql

SQL> drop table employees_2;
Table dropped.
SQL> commit;
Commit complete.
SQL>
SQL>
SQL>
SQL> exit    select count(*) frm   om employees_2;
select count(*) from employees_2
ERROR at line 1:
ORA-00942: table or view does not exist
```	    

Now it is time to recover the table renaming it and using an auxiliary location. The new table name will be emp_recvr. The script to be used is:

```RMAN

connect target /
RECOVER TABLE HR.EMPLOYEES_2 OF PLUGGABLE DATABASE HR_PDB
UNTIL TIME "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')"
AUXILIARY DESTINATION '/u02/recover'
DATAPUMP DESTINATION '/u02/recover'
REMAP TABLE 'HR'.'EMPLOYEES_2':'EMP_RECVR';
```

The RMAN session output is:

```RMAN

[oracle@grid12c backups]$ rman @recover_table.rman

Recovery Manager: Release 12.1.0.2.0 - Production on Fri Aug 10 23:28:10 2018

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
2> #run{
3> #ALLOCATE CHANNEL c1 DEVICE TYPE sbt_tape PARMS 'ENV=(OB_MEDIA_FAMILY=RMAN-DEFAULT)';
4> #ALLOCATE CHANNEL c2 DEVICE TYPE sbt_tape PARMS 'ENV=(OB_MEDIA_FAMILY=RMAN-DEFAULT)';
5> #ALLOCATE CHANNEL c3 DEVICE TYPE DISK ;
6> #ALLOCATE AUXILIARY CHANNEL c3 DEVICE TYPE DISK ;
7> RECOVER TABLE HR.EMPLOYEES_2 OF PLUGGABLE DATABASE HR_PDB
8> #UNTIL SCN 4410552
9> UNTIL TIME "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')"
10> AUXILIARY DESTINATION '/u02/recover'
11> DATAPUMP DESTINATION '/u02/recover'
12> REMAP TABLE 'HR'.'EMPLOYEES_2':'EMP_RECVR';
13> #RELEASE CHANNEL c1;
14> #RELEASE CHANNEL c2;
15> #}
16>
connected to target database: ORACDB (DBID=2649666862)

Starting recover at 10-AUG-18
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=33 device type=DISK
allocated channel: ORA_SBT_TAPE_1
channel ORA_SBT_TAPE_1: SID=278 device type=SBT_TAPE
channel ORA_SBT_TAPE_1: Oracle Secure Backup
RMAN-05026: WARNING: presuming following set of tablespaces applies to specified Point-in-Time

List of tablespaces expected to have UNDO segments
Tablespace SYSTEM
Tablespace UNDOTBS1

Creating automatic instance, with SID='Enea'

initialization parameters used for automatic instance:
db_name=ORACDB
db_unique_name=Enea_pitr_HR_PDB_ORACDB
compatible=12.1.0.2.0
db_block_size=8192
db_files=200
diagnostic_dest=/u01/app/oracle
_system_trig_enabled=FALSE
sga_target=1184M
processes=200
db_create_file_dest=/u02/recover
log_archive_dest_1='location=/u02/recover'
enable_pluggable_database=true
_clone_one_pdb_recovery=true
#No auxiliary parameter file used


starting up automatic instance ORACDB

Oracle instance started

Total System Global Area    1241513984 bytes

Fixed Size                     2923872 bytes
Variable Size                335544992 bytes
Database Buffers             889192448 bytes
Redo Buffers                  13852672 bytes
Automatic instance created

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')";
# restore the controlfile
restore clone controlfile;

# mount the controlfile
sql clone 'alter database mount clone database';

# archive current online log
sql 'alter system archive log current';
}
executing Memory Script

executing command: SET until clause

Starting restore at 10-AUG-18
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=12 device type=DISK
allocated channel: ORA_AUX_SBT_TAPE_1
channel ORA_AUX_SBT_TAPE_1: SID=173 device type=SBT_TAPE
channel ORA_AUX_SBT_TAPE_1: Oracle Secure Backup

channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: restoring control file
channel ORA_AUX_SBT_TAPE_1: reading from backup piece c-2649666862-20180810-05
channel ORA_AUX_SBT_TAPE_1: piece handle=c-2649666862-20180810-05 tag=TAG20180810T222446
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
output file name=/u02/recover/ORACDB/controlfile/o1_mf_fpw0wpk7_.ctl
Finished restore at 10-AUG-18

sql statement: alter database mount clone database

sql statement: alter system archive log current

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')";
# set destinations for recovery set and auxiliary set datafiles
set newname for clone datafile  1 to new;
set newname for clone datafile  4 to new;
set newname for clone datafile  3 to new;
set newname for clone datafile  8 to new;
set newname for clone datafile  9 to new;
set newname for clone tempfile  1 to new;
set newname for clone tempfile  3 to new;
# switch all tempfiles
switch clone tempfile all;
# restore the tablespaces in the recovery set and the auxiliary set
restore clone datafile  1, 4, 3, 8, 9;

switch clone datafile all;
}
executing Memory Script

executing command: SET until clause

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

renamed tempfile 1 to /u02/recover/ORACDB/datafile/o1_mf_temp_%u_.tmp in control file
renamed tempfile 3 to /u02/recover/ORACDB/datafile/o1_mf_temp_%u_.tmp in control file

Starting restore at 10-AUG-18
using channel ORA_AUX_DISK_1
using channel ORA_AUX_SBT_TAPE_1

channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00001 to /u02/recover/ORACDB/datafile/o1_mf_system_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00004 to /u02/recover/ORACDB/datafile/o1_mf_undotbs1_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 48t9oqdn_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=48t9oqdn_1_1 tag=TAG20180805T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:01:35
channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00003 to /u02/recover/ORACDB/datafile/o1_mf_sysaux_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 47t9oqdn_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=47t9oqdn_1_1 tag=TAG20180805T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:01:15
channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00009 to /u02/recover/ORACDB/datafile/o1_mf_sysaux_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 49t9oqh0_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=49t9oqh0_1_1 tag=TAG20180805T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:55
channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00008 to /u02/recover/ORACDB/datafile/o1_mf_system_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4bt9oqio_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4bt9oqio_1_1 tag=TAG20180805T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
Finished restore at 10-AUG-18

datafile 1 switched to datafile copy
input datafile copy RECID=9 STAMP=983835220 file name=/u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
datafile 4 switched to datafile copy
input datafile copy RECID=10 STAMP=983835220 file name=/u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
datafile 3 switched to datafile copy
input datafile copy RECID=11 STAMP=983835220 file name=/u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
datafile 8 switched to datafile copy
input datafile copy RECID=12 STAMP=983835220 file name=/u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf
datafile 9 switched to datafile copy
input datafile copy RECID=13 STAMP=983835220 file name=/u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')";
# online the datafiles restored or switched
sql clone "alter database datafile  1 online";
sql clone "alter database datafile  4 online";
sql clone "alter database datafile  3 online";
sql clone 'HR_PDB' "alter database datafile
8 online";
sql clone 'HR_PDB' "alter database datafile
9 online";
# recover and open database read only
recover clone database tablespace  "SYSTEM", "UNDOTBS1", "SYSAUX", "HR_PDB":"SYSTEM", "HR_PDB":"SYSAUX";
sql clone 'alter database open read only';
}
executing Memory Script

executing command: SET until clause

sql statement: alter database datafile  1 online

sql statement: alter database datafile  4 online

sql statement: alter database datafile  3 online

sql statement: alter database datafile  8 online

sql statement: alter database datafile  9 online

Starting recover at 10-AUG-18
using channel ORA_AUX_DISK_1
using channel ORA_AUX_SBT_TAPE_1
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4ft9rer7_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4ft9rer7_1_1 tag=TAG20180806T030049
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4et9rer5_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4et9rer5_1_1 tag=TAG20180806T030049
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4mt9u364_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4mt9u364_1_1 tag=TAG20180807T030017
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4lt9u363_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4lt9u363_1_1 tag=TAG20180807T030017
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4tta0nhq_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4tta0nhq_1_1 tag=TAG20180808T030008
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4sta0nhp_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4sta0nhp_1_1 tag=TAG20180808T030008
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 54ta3btn_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=54ta3btn_1_1 tag=TAG20180809T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 53ta3btn_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=53ta3btn_1_1 tag=TAG20180809T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5bta609q_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5bta609q_1_1 tag=TAG20180810T030007
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5ata609o_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5ata609o_1_1 tag=TAG20180810T030007
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:56
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5ita7va0_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5ita7va0_1_1 tag=TAG20180810T205528
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5hta7va0_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5hta7va0_1_1 tag=TAG20180810T205528
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00008: /u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5lta7vc4_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5lta7vc4_1_1 tag=TAG20180810T205528
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00009: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5jta7vc3_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5jta7vc3_1_1 tag=TAG20180810T205528
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5pta7vs3_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5pta7vs3_1_1 tag=TAG20180810T210506
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5ota7vs3_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5ota7vs3_1_1 tag=TAG20180810T210506
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00009: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5qta7vu7_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5qta7vu7_1_1 tag=TAG20180810T210506
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00008: /u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5sta7vuf_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5sta7vuf_1_1 tag=TAG20180810T210506
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 65ta82lb_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=65ta82lb_1_1 tag=TAG20180810T215242
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 64ta82lb_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=64ta82lb_1_1 tag=TAG20180810T215242
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00008: /u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 68ta82n3_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=68ta82n3_1_1 tag=TAG20180810T215242
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00009: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 66ta82n2_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=66ta82n2_1_1 tag=TAG20180810T215242
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:26
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00001: /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf
destination for restore of datafile 00004: /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 6eta84em_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=6eta84em_1_1 tag=TAG20180810T222317
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00003: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 6dta84em_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=6dta84em_1_1 tag=TAG20180810T222317
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:45
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00009: /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 6fta84gd_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=6fta84gd_1_1 tag=TAG20180810T222317
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00008: /u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 6hta84gl_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=6hta84gl_1_1 tag=TAG20180810T222317
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25

starting media recovery

archived log for thread 1 with sequence 203 is already on disk as file +FRA/ORACDB/ARCHIVELOG/2018_08_10/thread_1_seq_203.290.983831425
archived log file name=+FRA/ORACDB/ARCHIVELOG/2018_08_10/thread_1_seq_203.290.983831425 thread=1 sequence=203
media recovery complete, elapsed time: 00:00:02
Finished recover at 10-AUG-18

sql statement: alter database open read only

contents of Memory Script:
{
sql clone 'alter pluggable database  HR_PDB open read only';
}
executing Memory Script

sql statement: alter pluggable database  HR_PDB open read only

contents of Memory Script:
{
sql clone "create spfile from memory";
shutdown clone immediate;
startup clone nomount;
sql clone "alter system set  control_files =
''/u02/recover/ORACDB/controlfile/o1_mf_fpw0wpk7_.ctl'' comment=
''RMAN set'' scope=spfile";
shutdown clone immediate;
startup clone nomount;
# mount database
sql clone 'alter database mount clone database';
}
executing Memory Script

sql statement: create spfile from memory

database closed
database dismounted
Oracle instance shut down

connected to auxiliary database (not started)
Oracle instance started

Total System Global Area    1241513984 bytes

Fixed Size                     2923872 bytes
Variable Size                352322208 bytes
Database Buffers             872415232 bytes
Redo Buffers                  13852672 bytes

sql statement: alter system set  control_files =   ''/u02/recover/ORACDB/controlfile/o1_mf_fpw0wpk7_.ctl'' comment= ''RMAN set'' scope=spfile

Oracle instance shut down

connected to auxiliary database (not started)
Oracle instance started

Total System Global Area    1241513984 bytes

Fixed Size                     2923872 bytes
Variable Size                352322208 bytes
Database Buffers             872415232 bytes
Redo Buffers                  13852672 bytes

sql statement: alter database mount clone database

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')";
# set destinations for recovery set and auxiliary set datafiles
set newname for datafile  10 to new;
# restore the tablespaces in the recovery set and the auxiliary set
restore clone datafile  10;

switch clone datafile all;
}
executing Memory Script

executing command: SET until clause

executing command: SET NEWNAME

Starting restore at 10-AUG-18
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=16 device type=DISK
allocated channel: ORA_AUX_SBT_TAPE_1
channel ORA_AUX_SBT_TAPE_1: SID=177 device type=SBT_TAPE
channel ORA_AUX_SBT_TAPE_1: Oracle Secure Backup

channel ORA_AUX_SBT_TAPE_1: starting datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_SBT_TAPE_1: restoring datafile 00010 to /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_%u_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 4bt9oqio_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=4bt9oqio_1_1 tag=TAG20180805T030006
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:35
Finished restore at 10-AUG-18

datafile 10 switched to datafile copy
input datafile copy RECID=50 STAMP=983836228 file name=/u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE('10/08/2018 22:27:00','dd/mm/yyyy hh24:mi:ss')";
# online the datafiles restored or switched
sql clone 'HR_PDB' "alter database datafile
10 online";
# recover and open resetlogs
recover clone database tablespace  "HR_PDB":"HR_TBS", "SYSTEM", "UNDOTBS1", "SYSAUX", "HR_PDB":"SYSTEM", "HR_PDB":"SYSAUX" delete archivelog;
alter clone database open resetlogs;
}
executing Memory Script

executing command: SET until clause

sql statement: alter database datafile  10 online

Starting recover at 10-AUG-18
using channel ORA_AUX_DISK_1
using channel ORA_AUX_SBT_TAPE_1
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00010: /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5lta7vc4_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5lta7vc4_1_1 tag=TAG20180810T205528
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:26
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00010: /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 5sta7vuf_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=5sta7vuf_1_1 tag=TAG20180810T210506
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00010: /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 68ta82n3_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=68ta82n3_1_1 tag=TAG20180810T215242
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25
channel ORA_AUX_SBT_TAPE_1: starting incremental datafile backup set restore
channel ORA_AUX_SBT_TAPE_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00010: /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf
channel ORA_AUX_SBT_TAPE_1: reading from backup piece 6hta84gl_1_1
channel ORA_AUX_SBT_TAPE_1: piece handle=6hta84gl_1_1 tag=TAG20180810T222317
channel ORA_AUX_SBT_TAPE_1: restored backup piece 1
channel ORA_AUX_SBT_TAPE_1: restore complete, elapsed time: 00:00:25

starting media recovery

archived log for thread 1 with sequence 203 is already on disk as file +FRA/ORACDB/ARCHIVELOG/2018_08_10/thread_1_seq_203.290.983831425
archived log file name=+FRA/ORACDB/ARCHIVELOG/2018_08_10/thread_1_seq_203.290.983831425 thread=1 sequence=203
media recovery complete, elapsed time: 00:00:00
Finished recover at 10-AUG-18

database opened

contents of Memory Script:
{
sql clone 'alter pluggable database  HR_PDB open';
}
executing Memory Script

sql statement: alter pluggable database  HR_PDB open

contents of Memory Script:
{
# create directory for datapump import
sql 'HR_PDB' "create or replace directory
TSPITR_DIROBJ_DPDIR as ''
/u02/recover''";
# create directory for datapump export
sql clone 'HR_PDB' "create or replace directory
TSPITR_DIROBJ_DPDIR as ''
/u02/recover''";
}
executing Memory Script

sql statement: create or replace directory TSPITR_DIROBJ_DPDIR as ''/u02/recover''

sql statement: create or replace directory TSPITR_DIROBJ_DPDIR as ''/u02/recover''

Performing export of tables...
EXPDP> Starting "SYS"."TSPITR_EXP_Enea_Ebhe":  
EXPDP> Estimate in progress using BLOCKS method...
EXPDP> Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
EXPDP> Total estimation using BLOCKS method: 2 MB
EXPDP> Processing object type TABLE_EXPORT/TABLE/TABLE
EXPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
EXPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/MARKER
EXPDP> . . exported "HR"."EMPLOYEES_2"                          1.070 MB   15408 rows
EXPDP> Master table "SYS"."TSPITR_EXP_Enea_Ebhe" successfully loaded/unloaded
EXPDP> ******************************************************************************
EXPDP> Dump file set for SYS.TSPITR_EXP_Enea_Ebhe is:
EXPDP>   /u02/recover/tspitr_Enea_86769.dmp
EXPDP> Job "SYS"."TSPITR_EXP_Enea_Ebhe" successfully completed at Fri Aug 10 23:54:18 2018 elapsed 0 00:00:53
Export completed


contents of Memory Script:
{
# shutdown clone before import
shutdown clone abort
}
executing Memory Script

Oracle instance shut down

Performing import of tables...
IMPDP> Master table "SYS"."TSPITR_IMP_Enea_ieCF" successfully loaded/unloaded
IMPDP> Starting "SYS"."TSPITR_IMP_Enea_ieCF":  
IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE
IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
IMPDP> . . imported "HR"."EMP_RECVR"                            1.070 MB   15408 rows
IMPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
IMPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/MARKER
IMPDP> Job "SYS"."TSPITR_IMP_Enea_ieCF" successfully completed at Fri Aug 10 23:55:31 2018 elapsed 0 00:00:46
Import completed


Removing automatic instance
Automatic instance removed
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_temp_fpw2211k_.tmp deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_temp_fpw21q9h_.tmp deleted
auxiliary instance file /u02/recover/ENEA_PITR_HR_PDB_ORACDB/onlinelog/o1_mf_3_fpw28jqt_.log deleted
auxiliary instance file /u02/recover/ENEA_PITR_HR_PDB_ORACDB/onlinelog/o1_mf_2_fpw28h9d_.log deleted
auxiliary instance file /u02/recover/ENEA_PITR_HR_PDB_ORACDB/onlinelog/o1_mf_1_fpw28g23_.log deleted
auxiliary instance file /u02/recover/ENEA_PITR_HR_PDB_ORACDB/datafile/o1_mf_hr_tbs_fpw24pbf_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw133yp_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_system_fpw14v1m_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_sysaux_fpw10s20_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_undotbs1_fpw0xss7_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/datafile/o1_mf_system_fpw0xsr6_.dbf deleted
auxiliary instance file /u02/recover/ORACDB/controlfile/o1_mf_fpw0wpk7_.ctl deleted
auxiliary instance file tspitr_Enea_86769.dmp deleted
Finished recover at 10-AUG-18

Recovery Manager complete.
```

Now let's check the new table emp_recvr

```SQL

SQL> select count(*) from emp_recvr;
COUNT(*)
----------
 15408
SQL> selet ct count(*) from employees_2;
select count(*) from employees_2
ERROR at line 1:
ORA-00942: table or view does not exist
table or view does not exist
```
