# Oracle Database 21c QSG for EXPRESSCLUSTER X (Linux)

(Existing Oracle installation setup -Database Path set to mirror disk after database creation)

## About This Guide

This guide helps you to integrate Oracle Database 21c with 2 mirror nodes using the EXPRESSCLUSTER X (ECX). The guide is only for those who have basic understanding,knowledge and setup skills of EXPRESSCLUSTER X.

For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html).


## Configurations Description

In this document, create 2 nodes (Node1 and Node2 as below) mirror disk type cluster.
Prepare Oracle HA By using EXPRESSCLUSTER X. 


## Software Versions

- Oracle 21c    
- EXPRESSCLUSTER X 5.0 for Linux (internal version：5.0.2-1)
- EXPRESSCLUSTER X license
  - EXPRESSCLUSTER X 5.0 for Linux
  - EXPRESSCLUSTER X Replicator 5.0 for Linux
  - EXPRESSCLUSTER X Database Agent 5.0 for Linux (If you are planning to use Oracle monitor resource)

## Cluster Configurations

- Group resources
  - exec resource
  - floting IP resource
  - mirror disk resource
- Monitor rerources
  - floting IP monitor resource
  - mirror disk connect monitor resource
  - mirror disk monitor resource
  - Oracle monitor resource (Database Agent license is required.))
  
## Oracle Prerequisites and Installation Procedure

  - System Requirements for oracle-21c
  - Please refer and check to [this site](https://docs.oracle.com/en/database/oracle/oracle-database/21/ladbi/oracle-database-installation-checklist.html#GUID-E847221C-1406-4B6D-8666-479DB6BDB046) 
 
 OR
  - Please refer [this site](https://oracle-base.com/articles/21c/oracle-db-21c-installation-on-oracle-linux-8)

## Pre-checks before Installation :-

  - Hostname should configure on the both servers.
  - Also, set the fully qualified hostname is configured into /etc/hostname and /etc/hosts.
  - To completely disable SELinux on RedHat, open `/etc/selinux/config` file with a text editor and set the line SELINUX to disabled.
  - Firewall should be in closed state.
  - Make sure before installing a swap is created on the both machines.   

## EXPRESSCLUSTER setup  

  - Let us consider the following 2 node cluster and try to understand it.

### Cluster Information:-
  ---
  ||Node1(Active)|Node2(Standby)|
  |---|---|---|
  |Server Name|Server1|Server2|
  |IPaddress|10.0.7.174|10.0.7.175|  
  |cluster partition|/dev/sdb1|/dev/sdb1|
  |data partition|/dev/sdc2|/dev/sdc2|
    
### Failover Group Information:-
  ---
  |Parameter|Value|
  |---|---|
  |Name|Failover1|
  |Startup Server| Server1 -> Server2 |
  |Floating ip address|10.0.7.176|
  |Mirror disk resource (mount point))|/oradata/ORCL|

  - In Config mode of the Cluster WebUI, add failover group to use Db_oracle.  
  - You need to add the following resources.
      - Floating ip resource  
      - Mirror disk resource  
  - If you want to know how to add resource, please refer to [this site](https://github.com/EXPRESSCLUSTER/BasicCluster/blob/master/X41/Lin/2nodesMirror_Lin.md#how-to-setup-basic-2-nodes-mirror-cluster-on-linux) 
                                                          
 OR
 
  - please refer to how to setup basic 2 nodes mirror cluster on Linux [this site](https://github.com/EXPRESSCLUSTER/BasicCluster/blob/master/X41/Lin/2nodesMirror_Lin.md)


### Database configuration

- SID name               : orcl
- Database name          : ORCL
- Oracle home            : `/home/oracle`
- Database files location: `/oradata/ORCL`


## Moving Oracle Database Files to the Mirror Disk Partition

### Changing the location of the Database files (*.DBF) on the Primary Server

- All the default  (*.DBF) file path is /u01/app/oracle/oradata/ORCL

- Switch to oracle user and export the ORACLE_SID environment variable to target SID (example: orcl):
           
      export ORACLE_SID= orcl

- Start SQLPlus in command console:
         
      sqlplus as sysdba

- Connect to target database instance:
    
      Enter user-name: /as sysdba

- You should see the status at the SQLPlus prompt as:

    --> Connected

- Stop target database instance:
         
      SQL> shutdown immediate;

- Check the status at the SQL prompt as below:
   - Database closed.
   - Database dismounted.
   - ORACLE instance shut down.

 - Copy the database (*.DBF) files from default directory to mirrored disk directory. Some typical database files include the following:
  
   - SYSAUX01.DBF
   - SYSTEM01.DBF
   - UNDOTBS01.DBF
   - USERS01.DBF
           
          scp -r  /u01/app/oracle/oradata/ORCL/sysaux01.dbf   /oradata/ORCL/

          scp -r  /u01/app/oracle/oradata/ORCL/system01.dbf   /oradata/ORCL/

          scp -r  /u01/app/oracle/oradata/ORCL/users01.dbf    /oradata/ORCL/

          scp -r  /u01/app/oracle/oradata/ORCL/undotbs01.dbf  /oradata/ORCL/


- Start and mount database instance:
       
      SQL> startup mount

- Alter database (*.DBF) files by executing the following command for all moved database files:

  ## SYNTAX 

  SQL> alter database move datafile 'original file path' to 'new file path';

  Example: 
    ```
    SQL> alter database move datafile '/u01/app/oracle/oradata/ORCL/users01.dbf' to '/oradata/ORCL/users01.dbf';
    ```
  --> Database altered.
    ```
    SQL> alter database move datafile '/u01/app/oracle/oradata/ORCL/sysaux01.dbf' to '/oradata/ORCL/sysaux01.dbf';
    ```
  --> Database altered.
    ```
    SQL> alter database move datafile '/u01/app/oracle/oradata/ORCL/system01.dbf' to '/oradata/ORCL/system01.dbf';
    ``` 
  --> Database altered.
    ```
    SQL> alter database move datafile '/u01/app/oracle/oradata/ORCL/undotbs01.dbf' to '/oradata/ORCL/undotbs01.dbf';
    ```
  --> Database altered.

- Open alter database instance:
      
      SQL> alter database open;

  --> Database altered.

- Verify new database file locations:
    
      SQL> select name from v$datafile;

  - The command output should show new location of moved database files.


- Create new temporary database file by executing the following command at the prompt:

      SQL> create temporary tablespace temp2 tempfile '/oradata/ORCL/temp02.dbf' size 50M extent management local;

   --> Tablespace created.

  - Adjust the command options as per need.


- Configure new default temporary database file by executing the following command at the prompt:

      SQL> alter database default temporary tablespace temp2;

   - See the status at the SQL prompt as:

     --> Database altered.


- Remove original temporary tablespace and data files by executing the following command at the prompt: 

      SQL> drop tablespace temp including contents and datafiles;

    - You should see the status at the SQL prompt as:

      --> Tablespace altered.


## Changing the location of the log (*.log) database files

- Shutdown target database instance:
     
      SQL> shutdown immediate;

- Copy log (*.LOG) files from default directory (example: /u01/app/oracle/oradata/orcl) to mirrored disk directory (example: /oradata/orcl). 

  - Some typical database log files include the following:
    - REDO01.LOG
    - REDO02.LOG
    - REDO03.LOG        
   
  EXAMPLE :-

      mv /u01/app/oracle/oradata/ORCL/redo01.log    /oradata/ORCL/

      chown oracle:oinstall redo01.log
      
      mv /u01/app/oracle/oradata/ORCL/redo02.log    /oradata/ORCL/
       
      chown oracle:oinstall redo02.log

      mv /u01/app/oracle/oradata/ORCL/redo01.log    /oradata/ORCL/
      
      chown oracle:oinstall redo03.log

- Restart and mount target database instance:
       
      SQL> startup mount        

  - Alter log database files by executing the following command pattern for all moved log files:

    EXAMPLE:- 

        SQL>  alter database rename file '/u01/app/oracle/oradata/ORCL/redo01.log' to '/oradata/ORCL/redo01.log';

        SQL>  alter database rename file '/u01/app/oracle/oradata/ORCL/redo02.log' to '/oradata/ORCL/redo02.log';  

        SQL>  alter database rename file '/u01/app/oracle/oradata/ORCL/redo03.log' to '/oradata/ORCL/redo03.log';    


-  Reopen database instance:
        
        SQL> alter database open;

-  Check log file location 

        SQL>  select * from v$logfile;       

  

## Changing the location of the Control (*.ctl) database files

- Shutdown target database instance:
     
      SQL> shutdown immediate;

- Modify target Oracle instance SPFILE

- Execute the following SQLPlus command pattern to create PFILE for modification

      SQL> create pfile from spfile;

- see the status at the SQL prompt similar to:

     --> File created.

 - Switch to root user and Copy control (*.CTL) files from default directories to Oracle instance data directory on the mirrored disk (example: /oradata/orcl). Typically, the database control files include the following:

    - CONTROL01.CTL
    - CONTROL02.CTL  

    EXAMPLE :-

        [root@db1 ORCL]# scp -r  /u01/app/oracle/oradata/ORCL/control*    /oradata/ORCL/

        [root@db1 ORCL]# chown oracle:oinstall control01.ctl

        [root@db1 ORCL]# chown oracle:oinstall control02.ctl

- After that go in the path /home/oracle/dbs/initorcl.ora and edit control files location as MD
                      
      [root@db1 ~]# find /home -name initorcl.ora
      

      [oracle@db1 dbs]$ vi initorcl.ora
      ----------------------------------------------------
      *.control_files='/oradata/ORCL/control01.ctl','/oradata/ORCL/control02.ctl'
      ----------------------------------------------------
              
  - Edit the newly created PFILE and change directory paths in for the parameter “control_files” to the mirrored disk path.

- Start database instance with new parameter file:

      SQL> startup

- Verify control file location changes:
 
      SQL> show parameter control_files;

### All the below commands should be run by the oracle user 

- Edit/etc/oratab, change like follows

      db01:/usr/oracle/database:Y

- Edit .bash_profile

      export ORACLE_HOME=/home/oracle

      export ORACLE_SID=orcl

      sqlplus /nolog

      conn / as sysdba

## Changing the location of the Database files (*.DBF) on the secondary Server

- All the default  (*.DBF) file path is /u01/app/oracle/oradata/ORCL2

      SQL>Shutdown immediate;
 
- Copy pfile from primary server and replace in secondary server then create spfile from pfile. E.g. (/home/oracle/dbs/initorcl.ora)
 
      SQL> create spfile from pfile;
      SQL> Startup;
      
- Verify control file location changes:
 
      SQL> show parameter control_files;

- Verify new database file locations:

      SQL> select name from v$datafile;
      
- Verify new database file locations:
  
      SQL> select *  from v$logfile;

## Create table in oracle on primary server
```bat
SQL>   create table test as select * From dual;
```
The output is as Follows: **Table created.**

- checking created tables

```bat
SQL>select * From test;
```
The output is as Follows:

```bat

DUM
---
X

```

- Quit SQLPLus
```bat
SQL> quit;
```

## Move failover from **Primary server** to **Secondary Server**

- checking created tables

```bat
SQL>select * From test;
```
The output is as Follows:

```bat

DUM
---
X

```

## Set Listner on both the servers

- Replace the Hostname with the Virtual Computer Name in the “Listener.ora” "tnsnames.ora" file (/home/oracle/network/admin) to allow make the connection remotely on all the nodes in the cluster.

## Create Systemd Setting file for Oracle Database services.

- Login as root user and create Systemd setting files.


   ### [root@dlp ~]# vi /etc/sysconfig/orcl.oracledb  

       
          # create new : define environment variables
          ORACLE_BASE=/u01/app/oracle
          ORACLE_HOME=/home/oracle
          ORACLE_SID=orcl
        
- configure listener service

  
   ### [root@dlp ~]# vi /usr/lib/systemd/system/orcl@lsnrctl.service    
           
        # this is an example, modify for free
          [Unit]
          Description=Oracle Net Listener
          After=network.target

          [Service]
          Type=forking
          EnvironmentFile=/etc/sysconfig/orcl.oracledb
          ExecStart=/home/oracle/bin/lsnrctl start
          ExecStop=/home/oracle/bin/lsnrctl stop
          User=oracle

          [Install]
          WantedBy=multi-user.target

- Configure database service
       
    ### [root@dlp ~]# vi /usr/lib/systemd/system/orcl@oracledb.service   
   
             
        # this is an example, modify for free
        [Unit]
        Description=Oracle Database service
        After=network.target lsnrctl.service

        [Service]
        Type=forking
        EnvironmentFile=/etc/sysconfig/db01.oracledb
        ExecStart=/home/oracle/bin/dbstart $ORACLE_HOME
        ExecStop=/home/oracle/bin/dbshut $ORACLE_HOME
        User=oracle

        [Install]
        WantedBy=multi-user.target

- Reload configuration files
    ### [root@dlp ~]#systemctl daemon-reload  

- Change a startup type of the services
    ### [root@dlp ~]# systemctl enable orcl@lsnrctl orcl@oracledb  


## Create Exec resource for oracle service

### On Primary Server
1. Add one exec resource to the failover group
	- failover group
		- exec
			- start.sh: Refer a [sample script](#startsh)
			- stop.sh:  Refer a [sample script](#stopsh)
1. Apply the configuration
1. Start the exec resource

## Sample scripts
### start.sh
<a id="startsh"></a>
```bat
#! /bin/sh
#***************************************
#*              start.sh               *
#***************************************

#ulimit -s unlimited

systemctl start orcl@oracledb.service 

exit $?
```
### stop.sh
<a id="stopsh"></a>
```bat
#! /bin/sh
#***************************************
#*              start.sh               *
#***************************************

#ulimit -s unlimited

systemctl stop orcl@oracledb.service 

exit $?
```

## Create Oracle monitor resource (If you have Database Agent license)

### On Primary Server
1. Create an Oracle monitor resource
    - Select a exec resource as **Target Resource** of **Monitor Timing**
    - Select **listener and instance monitor** as **Monitor Type**
    - Input a database name as **Connect Command**
        - e.g. Connect String -> orcl
    - Input **system** as **User Name**
    - Input password of **system**
    - Select **DEFAULT** as **Authority Method**
    - Input **ORACLE_HOME**
        - e.g. ORACLE_HOME -> `/home/oracle`
    - Select **AMERICAN_AMERICA.UTF8** as **Character Set**
    - Input **Library Path**
        - e.g. Library Path -> `/home/oracle/lib/libclntsh.so.21.1`
    - Select a failover group as **Recovery Target**
        - e.g. Recovery Target -> failover group

1. Apply the configuration file
1. Start a failover group

## Change PASSWORD_LIFE_TIME unlimited (If you use Oracle monitor)

### On Primary server
The default password expiration date for user authentication is 180 days. If you use Oracle monitor resource, you should change the password regularly or change the password expiration date to an indefinite time in order to avoid unnecessary monitor failure.

```bat
> sqlplus system/<password>@SID1
SQL> alter profile DEFAULT limit PASSWORD_LIFE_TIME unlimited;
SQL> alter profile ORA_STIG_PROFILE limit PASSWORD_LIFE_TIME unlimited;
```