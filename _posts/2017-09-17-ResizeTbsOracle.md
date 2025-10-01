---
title: "Oracle Automatically Resize Tablespace"
date: 2017-09-17 13:00:00 -0300
categories: [Database, Oracle ]
tags: [Tablespace, automation, sql]
author: Cristhian Britez
description: "How to resize Oracle tablespace automatically using plsql"
pinned: true # Optional: to pin the post to the homepage
---
As oracle DBA sometimes we have a really hard work with high used databases because these increase so fast. Today I want to show you a pl/sql code that may help you resizing datafiles automatically without your intervention.

The first thing we need to think is in oracle autoextend feature, autoextend help us a lot increasing the datafiles when the available space is compromissed, but what happen when the last datafile of a tablespace reach its maxsize?. Oracle (until today) isn't able to create a new datafile and continue adding space to the database. For this scenario, this code could help you a lot, and if you have some database monitoring system that is able to run script against the database you won't receive anymore service request for doing this task.

Let's start,
 First we need to get some information for the datafiles (size,path,bigfile,maxsize) from dba_tablespaces and dba_data_files and load into cursor.

```sql

SQL>  select df.file_name AS df_file_name,tb.bigfile AS isbigfile,df.autoextensible AS df_file_autoextend,bytes AS df_file_size,df.maxbytes maxbytes FROM dba_data_files df,dba_tablespaces tb WHERE df.tablespace_name=:tb_name AND df.tablespace_name=tb.tablespace_name;

DF_FILE_NAME                                    ISB DF_    DF_FILE_SIZE        MAXBYTES
---------------------------------------------------------------------- --- --- --------------- ---------------
/u02/oradata/orcl/oetbs_01.dbf                                         NO  NO        524288000               0
```

Then, we need to get the maxbytes allowable by a datafile (the max size is determined by the block_size and a constant value depending if it is a big or normal datafile.

```sql

SQL> SELECT CASE tb.isbigfile when 'YES' THEN 4294967295*vp.db_block_size ELSE 4194303*vp.db_block_size END AS max_df_size  FROM (SELECT value db_block_size FROM v$parameter WHERE name='db_block_size')vp,(SELECT bigfile AS isbigfile FROM dba_tablespaces WHERE tablespace_name=:tb_name) tb;
MAX_DF_SIZE
---------------
34359730176
```

After the maxsize, we also are able to determine the minsize required to get x% free (for monitoring system an alert maybe launched for datafiles with less than 10% free

```sql

	SQL>  SELECT ((total*0.1)-free_space) k_needed  FROM (SELECT nvl(sum(bytes),0) total FROM dba_data_files WHERE tablespace_name=:tb_name GROUP BY tablespace_name)df,(SELECT CASE count(dba_fs.file_id)  WHEN 0 THEN (SELECT nvl(dba_df.user_bytes-dba_s.bytes,0) from (select sum(user_bytes) user_bytes from dba_data_files where tablespace_name=:tb_name group by tablespace_name)dba_df,(select sum(bytes) bytes from dba_segments where tablespace_name=:tb_name group by tablespace_name) dba_s) ELSE (SELECT nvl(sum(bytes),0) FROM dba_free_space WHERE tablespace_name=:tb_name GROUP BY tablespace_name) END AS free_space from dba_data_files dba_df full outer join dba_free_space dba_fs using(tablespace_name) where tablespace_name=:tb_name group by tablespace_name) ds;
	K_NEEDED
---------------
275251.2
```


Now, we can loop into the cursor and execute autoextend on, resize and create new datafile if needed. The complete code is this:

```sql

SET SERVEROUT on;
set verify off;
set lines 300;
set feedback off;
declare
	tb_name varchar2(40):= upper('&1');
	new_data_file varchar2(1000);
	v_query_autoextend varchar2(1000);
	v_query_size varchar2(1000);
	v_message varchar2(500);
	oracle_home varchar2(500);
	v_isbig_dbf varchar(3):='NO';
	min_req_size number :=0;
	max_datafile_size number :=0;
	v_new_df_size number :=0;
	type t_record is record ( code varchar2(10),msg varchar2(500));
  type v_record is table of t_record;
  v_array v_record := v_record();
	v_array_index number :=0;

	-- Get datafile path, size, bigfile, autoextensible from dba_data_Files
	CURSOR c1_df_info
  IS select df.file_name AS df_file_name,tb.bigfile AS isbigfile,df.autoextensible AS df_file_autoextend,bytes AS df_file_size,df.maxbytes maxbytes
  FROM dba_data_files df,dba_tablespaces tb
  WHERE df.tablespace_name=tb_name AND df.tablespace_name=tb.tablespace_name;

begin

	-- Get max datafile size with formula= db_block_size, when bigfile 4294967295*vp.db_block_size else 4194303*vp.db_block_size

	SELECT CASE tb.isbigfile when 'YES' THEN 4294967295*vp.db_block_size ELSE 4194303*vp.db_block_size END AS max_df_size INTO max_datafile_size FROM (SELECT value db_block_size FROM v$parameter WHERE name='db_block_size')vp,(SELECT bigfile AS isbigfile FROM dba_tablespaces WHERE tablespace_name=tb_name) tb;

	-- Get minimun size required to come to 10% free, the formula is: 10% from total size (dba_data_Files) - free_space (dba_free_space)

	SELECT ((total*0.1)-free_space) k_needed INTO min_req_size FROM (SELECT nvl(sum(bytes),0) total FROM dba_data_files WHERE tablespace_name=tb_name GROUP BY tablespace_name)df,(SELECT CASE count(dba_fs.file_id)  WHEN 0 THEN (SELECT nvl(dba_df.user_bytes-dba_s.bytes,0) from (select sum(user_bytes) user_bytes from dba_data_files where tablespace_name=tb_name group by tablespace_name)dba_df,(select sum(bytes) bytes from dba_segments where tablespace_name=tb_name group by tablespace_name) dba_s) ELSE (SELECT nvl(sum(bytes),0) FROM dba_free_space WHERE tablespace_name=tb_name GROUP BY tablespace_name) END AS free_space from dba_data_files dba_df full outer join dba_free_space dba_fs using(tablespace_name) where tablespace_name=tb_name group by tablespace_name) ds;

	for cursor_stm in c1_df_info
	loop
		v_isbig_dbf := cursor_stm.isbigfile;
		v_query_autoextend := NULL;
		v_query_size := NULL;
		v_message := NULL;

		-- If autoextend if NO or autoextend if NO and maxbytes is less than the maximum size expressed in megabytes, autoextend to unlimited.

		if ( cursor_stm.df_file_autoextend = 'NO' OR (cursor_stm.df_file_autoextend = 'YES' AND (trunc(cursor_stm.maxbytes/1048576) < trunc(max_datafile_size/1048576))))
		then
			v_query_autoextend := 'ALTER DATABASE DATAFILE '''||cursor_stm.df_file_name||''' AUTOEXTEND ON NEXT 16777216 MAXSIZE UNLIMITED';

		end if;

		-- Resize datafile: if minimun required size is higher than max datafile size, then increment the datafile to its max size else increment the required to get 12% free.

		if ( min_req_size > 0 )
		then
			v_new_df_size := round(cursor_stm.df_file_size + min_req_size + (cursor_stm.df_file_size*0.03));
			if ( v_new_df_size > max_datafile_size )
			then
				v_query_size:= 'ALTER DATABASE DATAFILE '''||cursor_stm.df_file_name||''' RESIZE '||to_char(max_datafile_size);
			else
				v_query_size := 'ALTER DATABASE DATAFILE '''||cursor_stm.df_file_name||''' RESIZE '||to_char(v_new_df_size);
			end if;
		end if;

		-- Execute generated statements for autoextend and resize.

			begin
				if ( v_query_autoextend IS NOT NULL )
				then
					execute immediate v_query_autoextend;
					v_array_index := v_array_index+1;
					v_array.extend(1);
					v_array(v_array_index).code:='INFO: ';
					v_array(v_array_index).msg:='Command: '||v_query_autoextend||'. Executed successfully';
				end if;
			exception
				when others then
					v_array_index := v_array_index+1;
          v_array.extend(1);
          v_array(v_array_index).code:='ERROR: ';
          v_array(v_array_index).msg:= SQLERRM||chr(10)||'Failed sql: '||v_query_autoextend;
			end;

			begin
				if (v_query_size IS NOT NULL)
				then
					execute immediate v_query_size;
					v_array_index := v_array_index+1;
          v_array.extend(1);
          v_array(v_array_index).code:='INFO: ';
          v_array(v_array_index).msg:='Command: '||v_query_size||'. Executed successfully';
				end if;
			exception
				when others then
          v_array_index := v_array_index+1;
          v_array.extend(1);
					v_array(v_array_index).code:='ERROR: ';
          v_array(v_array_index).msg:= SQLERRM||chr(10)||'Failed sql: '||v_query_size;
					continue;
			end;

		--end if;

		SELECT ((total*0.1)-free_space) k_needed INTO min_req_size FROM (SELECT nvl(sum(bytes),0) total FROM dba_data_files WHERE tablespace_name=tb_name GROUP BY tablespace_name)df,(SELECT CASE count(dba_fs.file_id)  WHEN 0 THEN (SELECT nvl(dba_df.user_bytes-dba_s.bytes,0) from (select sum(user_bytes) user_bytes from dba_data_files where tablespace_name=tb_name group by tablespace_name)dba_df,(select sum(bytes) bytes from dba_segments where tablespace_name=tb_name group by tablespace_name) dba_s) ELSE (SELECT nvl(sum(bytes),0) FROM dba_free_space WHERE tablespace_name=tb_name GROUP BY tablespace_name) END AS free_space from dba_data_files dba_df full outer join dba_free_space dba_fs using(tablespace_name) where tablespace_name=tb_name group by tablespace_name) ds;

	end loop;


	-- If min required size is higher than 0, no enought datafiles exits and a new one is created.

	if (min_req_size>0 and v_isbig_dbf='NO') then
		v_new_df_size := round(min_req_size + (max_datafile_size*0.03));
		-- Get Oracle home to avoid creating new datafile in oracle home
	  begin
		  sys.dbms_system.get_env('ORACLE_HOME',oracle_home);

		    SELECT 'ALTER TABLESPACE '|| tb_name||' ADD DATAFILE '''||last_data_file||CASE INSTR(first_data_file,'_',-1,1) WHEN 0 THEN first_data_file ELSE  SUBSTR(first_data_file,1,INSTR(first_data_file,'_',-1,1)-1) END||'_'||TRIM(new_qty.new_qty)||'.dbf'' SIZE '|| v_new_df_size ||' AUTOEXTEND ON NEXT 4194304 MAXSIZE UNLIMITED'as new_df_name into new_data_file
		    FROM
		        (SELECT SUBSTR(file_name,INSTR(file_name,'/',-1,1)+1,INSTR(file_name,'.dbf',-1,1) - INSTR(file_name,'/',-1,1)-1) first_data_file,tablespace_name FROM dba_data_files d WHERE tablespace_name=tb_name AND file_id=(SELECT MIN(file_id) FROM dba_Data_files WHERE tablespace_name=tb_name)) first_file,
		          (SELECT CASE  WHEN LENGTH(COUNT(*)+1) <= 2 THEN TO_CHAR(COUNT(*)+1,'00') ELSE TO_CHAR(COUNT(*)+1,'000') END as new_qty,tablespace_name FROM dba_data_files WHERE tablespace_name=tb_name group by tablespace_name) new_qty,
		            (SELECT SUBSTR(file_name,1,INSTR(file_name,'/',-1,1)) last_data_file,tablespace_name FROM dba_data_files WHERE tablespace_name=tb_name AND file_id=(SELECT MAX(file_id) FROM dba_data_files WHERE tablespace_name=tb_name))last_file
		     where first_file.tablespace_name=new_qty.tablespace_name and first_file.tablespace_name=last_file.tablespace_name;
		-- If datafile path is equal to oracle home, en error is printed, else a new datafile is created.

		if ( oracle_home = substr(new_data_file,instr(new_data_file,'/',1),instr(new_data_file,'/',-1)-instr(new_data_file,'/',1)))
    then
			v_array_index := v_array_index+1;
			v_array.extend(1);
			v_array(v_array_index).code:= 'ERROR: ';
			v_array(v_array_index).msg:= 'Unable to create datafile in ORACLE_HOME:'||oracle_home||chr(10)||'Failed sql: '||new_data_file;
		else

			execute immediate new_data_file;
			v_array_index := v_array_index+1;
                        v_array.extend(1);
                        v_array(v_array_index).code:= 'INFO: ';
                        v_array(v_array_index).msg:= 'Command: '||new_data_file||'. Executed successfully';

		end if;
	    exception
			when others then
          v_array_index := v_array_index+1;
          v_array.extend(1);
          v_array(v_array_index).code:= 'ERROR: ';
          v_array(v_array_index).msg:= SQLERRM||chr(10)||'Failed sql: '||new_data_file;
      end;
	end if;
	if ( v_array_index > 0) then

		for i in 1..v_array_index
		loop
			dbms_output.put_line(v_array(i).code||v_array(i).msg);
		end loop;
	end if;

end;
/
```

[Download this code or improve it](https://github.com/cbritezm/cbritezm/blob/master/resize_tablespace_plsql.sql)

# How to implement?

For implementing, you just have to call this script from you monitoring system or schedule an execution with crontab, for that, you just need to call the script and pass the tablespace name as argument You could also modify the script for extend all the datafiles in the database, just you need to do little modifications to the code. The default free space for this script is 12% free.

If something is wrong, the script will print a error message with the failed sentence and the error code.
```sql
SQL> @resize_tbs_plsql.sql oetbs

INFO: Command: ALTER DATABASE DATAFILE '/u02/oradata/orcl/oetbs_01.dbf' AUTOEXTEND ON NEXT 16777216 MAXSIZE UNLIMITED. Executed successfully

INFO: Command: ALTER DATABASE DATAFILE '/u02/oradata/orcl/oetbs_01.dbf' RESIZE 13235651. Executed successfully
```
