# Oracle Database 21c Quick Start Guide for EXPRESSCLUSTER X 

## About This Guide

This guide provides how to integrate Oracle Database 21c with EXPRESSCLUSTER X (ECX) using mirror disks with 2 nodes. The guide assumes its readers to have EXPRESSCLUSTER X basic knowledge and setup skills.

## System Overview

### System Requirement
- 2 servers
  - IP reachable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
      - Cluster partition size depends on ECX version.
        - X 5.0 or later: 1024MB
      - Data partition size depends on Database sizing.
- Oracle Database are installed on local partition on each server.
- Database files are created on data partition.


### System Configuration
- Windows Server 2022 Standard
- Oracle Database 21c
- EXPRESSCLUSTER X 5.0.2

        Sample configuration

		<LAN>
		 |
		 |  +--------------------------+
		 +--| Primary Server           |
		 |  | - Windows Server 2022    |
		 |  | - Oracle Database 21c    |
		 |  | - EXPRESSCLUSTER X 5.0.2 |
		 |  |                          |
		 |  | RAM   : 8GB              |
		 |  | Disk 0: 50GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 40GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +-----------+--------------+
		 |              |
		 |              | Mirroring
		 |              |
		 |  +-----------+--------------+
		 +--| Secondary Server         |
		 |  | - Windows Server 2022    |
		 |  | - Oracle DB 21c          |
		 |  | - EXPRESSCLUSTER X 5.0.2 |
		 |  |                          |
		 |  | RAM   : 8GB              |
		 |  | Disk 0: 50GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 40GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +--------------------------+
		 |

#### Cluster Configuration
- Network Partition Resolution resource (PING method) : `pingnp1`

- failover Group: `Oraclefailover`
	- Group resource
		- Fip                : Floating IP address resource
		- md                 : mirror disk resource
		- db1_service        : service resource for OracleServiceorcl
		- listener1_service  : service resource for OracleOraDB12Home1TNSListener

- Monitor resource

	- Fipw1             : Floating IP monitor resource
	- mdnw1             : mirror connect monitor resource
	- mdw1              : mirror disk monitor resource
	- servicew1         : service monitor resource for db1_service
	- servicew2         : service monitor resource for listener1_service
	- userw             : user-mode monitor resource

#### Database Configuration

- SID name               : orcl
- Database name          : orcl
- PDB name               : orclpdb
- Oracle base            : `C:\app\oradb`
- software location      : `c:\app\oradb\product\21c\dbhome_1`
- Database Files location: `D:\oradata`
- fast recovery area     : `D:\fast_recovery_area`

## System Setup

### Prerequisite

##### On Primary and Secondary Servers
1. Change a virtual memory (swap) size
    1. Open Control Panel
    1. Go to **System and Security** -> **System**
    1. Click **Advanced system settings**
    1. Click **Settings** of **Performance** in **Advanced** tab
    1. Click **Change** in **Advanced** tab
    1. Uncheck **Automatically manage paging file size for all drivers**
    1. Select **Custom size** and input the size to **Initial size** and **Maximize size**
    1. Click **Set** and **OK**
        - IF physical memory is between 2 GB and 16 GB, then set virtual memory to 1 times the size of the RAM
        - IF physical memory is more than 16 GB, then set virtual memory to 16 GB
    1. Reboot the machine after applying the settings

### Basic cluster setup

##### On Primary and Secondary Servers
1. Install EXPRESSCLUSTER X (ECX)
2. Register ECX licenses
    - EXPRESSCLUSTER X 5.0.2 for Windows
    - EXPRESSCLUSTER X Replicator 5.0.2 for Windows

##### On Primary Server
3. Create a cluster and a failover group
    - Network partition: 
    
        pingnp1: PING method
        
    - Group:
    
        Oraclefailover
        - Fip: Floating IP resource
        - md : mirror disk resource
        
4. Start group on Primary server

### Create OS user

##### On Primary and Secondary Servers
1. Create Oracle install user

    1. Open **Computer Management**
    
        ```bat
       > compmgmt.msc
       ```

    1. Select **Local Users and Groups** in  the left pane and click **Users**
    1. Right-click in the center pane and click **New user...**
    1. Input proper information
        - e.g. Username is `orasys`
        - Uncheck **User must change password at next logon**
    1. Click **Create**
    
1. Create Oracle home user for database

    Follow the same steps as orasys to create Oracle home user.
    - e.g. User name is `oradb`
    - Uncheck **User must change password at next logon**

1.  Add orasys to Administrators group
    1. Open **Computer Management**
    
        ```bat
       > compmgmt.msc
       ```

    1. Select **Local Users and Groups** in  the left pane and click **Groups**
    1. Select **Administrators** and right-click and click **Add to Group...**
    1. Add orasys to **Members:**
    1. Click **OK**
    
### Install Oracle Database

##### On Primary and Secondary Servers
1. Logon as orasys
1. **(In case of 21c)** Create the folder for **%ORACLE_HOME%**
    - e.g. C:\app\oradb\product\21c\dbhome_1
1. **(In case of 21c)** Unzip Oracle installation file to the **%ORACLE_HOME%**
1. Start **setup.exe** of Oracle Database
1. **(In case of 21c)** Configure Security Updates:
    - Uncheck **I wish to receive security updates via My Oracle Support.**
1. Installation Option:
    - **(In case of 21c)** Select **Install database software only**
    - **(In case of 21c)** Select **Set Up software Only**
1. Database Installation Options:
    - Select **Single instance database installation**
1. Database Edition:
    - Select **Enterprise Edition (6.0GB)**
1. Oracle Home User:
    - Select **Use Existing Windows User**
    - Input Oracle home user name
        - e.g. oradb
1. Installation Location:
    - Input **Oracle base** and **software location** in a local partition
      - e.g. Oracle base -> C:\app\oradb
      - e.g. software location -> C:\app\oradb\product\21c\dbhome_1
    - In this document, **%ORACLE_HOME%** means **software location**
1. Prerequisite Checks:
    - Wait for checking
1. Summary:
    - Click **Install**

### Create Database

##### On Primary Server
1. Logon as orasys
2. Start **dbca.bat** with Administrator authority
    - Right click the bat file and click **Run as administrator**
    - The path is `%ORACLE_HOME%\bin\dbca.bat`
3. Database Operation:
    - Select **Create a database**
4. Creation Mode:
    - Select **Advanced configuration**
5. Deployment Type:
    - Select **Oracle Single Instance database** as **Database type**
    - Select **Custom Database**
6. Database Identification:
    - e.g. Global database name -> orcl
    - e.g. SID -> orcl
    - Check **Create as Container database**
    - Check **Use Local Undo tablespace for PDBs**
    - Select **Create a Container database with one or more PDBs**
    - e.g. PDB name -> orclpdb
7. Storage Option:
    - Select **Use Following for the database storage attributes**
    - Select **file System** as **Database Files storage type**
    - Input a location in the *data partition* as **Database Files location**
      - e.g. Database Files location -> C:\oradata
8. fast Recovery Option:
    - Check **Specify fast Recovery Area**
    - Select **file System** as **Recovery Files storage type**
    - Input a location in a data partition as **fast Recovery Area**
      - e.g. fast Recovery Area -> C:\fast_recovery_area
      - e.g. fast Recovery Area size -> 17271MB
    - Check **Enable archiving**
9. Network configuration:
    - Click **Next**
10. Database Options:
    - Check all components
    - IF you need, check **Include in PDBs** checkboxes
11. Data Vault Option:
    - Click **Next**
12. configuration Options:
    - Click **Next**
        - Block size and Character sets cannot be changed after creating database
            - IF you want to change Block size or Character sets, you must reconstruct database
13. Management Options:
    - Uncheck **Configure Enterprise Manager (EM) database express**
14. User Credentials:
    - Input each passwords
15. Creation Option:
    - Check **Create database**
16. Summary:
    - Click **Finish**
17. after creating the database, run Command Prompt as administrator
    - Right click Command Prompt icon and select **Run as administrator**
18. Change database startmode

    ```bat
    > oradim -edit -sid <SID name> -startmode auto
    
    e.g.
    > oradim -edit -sid orcl -startmode auto
    ```
    
    Note: The database is started automatically after starting OracleServiceorcl

##### On Secondary Server
19. Logon as orasys, and run Command Prompt as administrator
    - Right click Command Prompt icon and select **Run as administrator**

20. Create database service

    ```bat
    > oradim -new -sid <SID name> -startmode auto
    
    e.g.
    > oradim -new -sid orcl -startmode auto
    ```
    
21. Create a password file

    The password is recommended to be same with the SYS password defined in **14. User Credentials:**
    
    - IF Oracle Database version is earlier than 12.2
    
    ```bat
    > orapwd file=%ORACLE_HOME%\database\PWD<SID name>.ora password=<password>
    
    e.g.
    > orapwd file=C:\app\oradb\product\12.0.0\dbhome_1\database\PWforcl.ora password=oracle
    ```
    
    - IF Oracle Database version is 12.2 or later
    
    ```bat
    > orapwd file=%ORACLE_HOME%\database\PWD<SID name>.ora password=<password> format=12
    
    e.g.
    > orapwd file=C:\app\oradb\database\PWforcl.ora password=oracle format=12
    ```
    
### Create listener

##### On Primary and Secondary Servers
1. Create listener.ora

    ($ORACLE_HOME/bin/netca)

    - Perform the Following steps to create a listener.​
    - Select Listener configuration, click Next.
    - Select Add, click Next.
    - Choose a listener name, click Next.
    - Default listener name is 'LISTENER'.
    - for selected protocols, select TCP, click Next.
    - Enter in the port used to connect to the oracle database, click Next.
    - Default port is 1521.
    - Select No to Configure another listener, click Next.
    - Click Next to complete the listener configuration.
    - Click Finish on the Welcome screen.

    Sample of listener.ora
    ```bat
    LISTENER =
    	(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    ```
    
2. Create tnsnames.ora

    Sample of tnsnames.ora
    
    (%ORACLE_HOME%\network\admin\tnsnames.ora)
    ```bat
    LISTENER =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    		)
    	)
    
    orcl =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = orcl)
    		)
    	)
    
    orclPDB =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = orclpdb)
    		)
    	)
    ```

### Oracle Database Configuration on Mirror Disk

##### On Primary Server


Now create directory in Data Mirror drive.
Eg:-  “D:\oradata\orcl”

- Moving Oracle Database file to the Data Partition Follow given below steps.
                                                                     
1. Start the command console using **Start->Run->cmd**
 
2. Set the ORACLE_SID environment variable to target SID (example: ORCL):

```bat
C:\> set ORACLE_SID= ORCL
```

3. Start SQLPlus in command console:
```bat
C:\>sqlplus /nolog
```

4. Connect  to target database instance:

```bat
SQL> conn /as sysdba
```

The output would be as Follows:
**Connected**

5. Stop target database instance:
```bat
SQL> shutdown immediate;
```
The output would be as Follows:

---
- Database closed.
- Database dismounted.
- ORACLE instance shut down.
---

6. Copy the database (*.dbF) Files From default directory 

(example: C:\app\oradata\orcl\) to mirrored disk directory (example: D:\oradata\orcl\). 

```bat
Some typical database Files include the Following:

EXAMPLE01.dbF
SYSAUX01.dbF
SYSTEM01.dbF
UNDOTBS01.dbF
USERS01.dbF
```

7. Restart and mount target database instance:
```bat
SQL> startup mount
```
The output would be as Follows:
**ORACLE instance started.**

```bat
Total System Global Area    131144544 bytes
Fixed Size                   453472 bytes
Variable Size               96468992 bytes
Database buffers           33554432 bytes
Redo buffers                 667648 bytes
Database mounted.
```


8. Set new location of the database (*.DBF) files by executing the following command for all moved database files:

```bat
SQL> alter database rename file <original file path> to <new file path>;
```

Example: 

```bat
alter database rename file 'C:\app\oradata\orcl\system01.dbf’  to 'D:\oradata\orcl\system01.dbf’;

alter database rename file 'C:\app\oradata\orcl\sysaux01.dbf’  to 'D:\oradata\orcl\sysaux01.dbf’;

alter database rename file 'C:\app\oradata\orcl\example01.dbf’  to 'D:\oradata\orcl\example01.dbf’;

alter database rename file 'C:\app\oradata\orcl\undotbs01.dbf’  to 'D:\oradata\orcl\undotbs01.dbf’;

alter database rename file 'C:\app\oradata\orcl\users01.dbf’  to 'D:\oradata\orcl\users01.dbf’;
```

The output would be as follows:
**Database altered.**

9. Open target database instance:

```bat
SQL> alter database open;
```

The output would be as Follows:
**Database altered.**

10. Verify new database file locations:
```bat
SQL> select name From v$datafile;

The command output should show new location of moved database Files.

D:\ORADATA\ORCL\SYSTEM01.DBF
C:\ORADATA\ORCL\PDBSEED\SYSTEM01.DBF
D:\ORADATA\ORCL\SYSAUX01.DBF
C:\ORADATA\ORCL\PDBSEED\SYSAUX01.DBF
D:\ORADATA\ORCL\UNDOTBS01.DBF
C:\ORADATA\ORCL\PDBSEED\UNDOTBS01.DBF
D:\ORADATA\ORCL\USERS01.DBF
C:\ORADATA\ORCL\ORCLPDB1\SYSTEM01.DBF
C:\ORADATA\ORCL\ORCLPDB1\SYSAUX01.DBF
C:\ORADATA\ORCL\ORCLPDB1\UNDOTBS01.DBF
C:\ORADATA\ORCL\ORCLPDB1\USERS01.DBF
```

11. Move the original database Files EXCEPT for TEMP01.dbF to a different directory as backup.

12. Create new temporary database file by executing the Following command at the prompt:

```bat
SQL> create temporary tablespace temp2 tempfile '<mirrored disk Oracle instance data directory>\temp02.dbF' <options>;
```

Example:

```bat
SQL> create temporary tablespace temp2 tempfile 'D:\oradata\orcl\temp02.dbF' size 50M extent management local;
```

The command options may be adjusted to suit one’s needs:

The output would be as Follows:
**Tablespace created.**

13. Configure new default temporary database file by executing the Following command at the prompt:

```bat
SQL> alter database default temporary tablespace temp2;
```

The output would be as Follows:
**Database altered.**


14. Remove original temporary tablespace and data Files by executing the Following command at the prompt:

```bat
SQL> drop tablespace temp including contents and dataFiles;
```

 The output would be as Follows:
**Tablespace altered.**

This completes the re-configuration of the database (*.dbF) Files to the mirror partition D:\.

**Do not quit From the SQLPlus utility at this time. Please proceed to Changing the location of the log (*.log) Files on the Primary Server (Machine 1).** 


15. Changing the location of the log Files (*.log) on the Primary Server (Machine 1) 
    Shutdown target database instance:

```bat
SQL> shutdown immediate;
```

The output would be as Follows:

---
- Database closed.
- Database dismounted.
- ORACLE instance shut down.
---


16. Copy log (*.LOG) Files From default directory (example: C:\oradata\ORCL\) to mirrored disk directory (example: D:\oradata\orcl\).  Some typical database log Files include the Following:

- REDO01.LOG
- REDO02.LOG
- REDO03.LOG

17. Restart and mount target database instance:

```bat
SQL>startup mount;
```
 The output would be as Follows:

```bat
ORACLE instance started.
Total System Global Area          131144544 bytes
Fixed Size                          453472 bytes
Variable Size                      96468992 bytes
Database buffers                  33554432 bytes
Redo buffers                      667648 bytes
Database mounted.
```

18. Set new location of the log Files by executing the Following command pattern for all moved log Files:

```bat
SQL> alter database rename file <original file path> to <new file path>;
```
Example: alter database rename file 

'C:\oradata\orcl\redo01.log' to 'D:\oradata\orcl\redo01.log';

```bat
alter database rename file 'C:\oradata\ORCL\redo02.log' to 'D:\oradata\orcl\redo02.log';
alter database rename file 'C:\oradata\ORCL\redo03.log' to 'D:\oradata\orcl\redo03.log'; 
```

The output would be as Follows:
 **Database altered.**

19. Reopen target database instance:
```bat
SQL> alter database open;
```
   The output would be as Follows:
**Database altered.**

20. Changing the location of the Control Files (*.ctl) on the Primary Server (Machine 1)    
        
 Shutdown target database instance:
```bat
SQL> shutdown immediate;	
```
The output would be as Follows:

---
- Database closed.
- Database dismounted.
- ORACLE instance shut down.
---

21. Modify target Oracle instance SPfile
Execute the Following SQLPlus command pattern to create 
Pfile for modification:
```bat
SQL> create pfile From spfile;
```
The output would be as Follows:
**file created.**

The sample Pfile may also want to be reviewed which is automatically created by the Oracle DB installer to see If there is any parameter setting that may want to be added to the SPfile that was created:

 The sample Pfile can typical be found (example:C:\app\oradb\database\)

22. Copy control (*.CTL) Files From default directories (example: C:\oradata\orcl and C:\fast recovery area\ORCL) to Oracle instance data directory on the mirrored disk (example: D:\oradata\orcl ):

 Typically, the database control Files include the Following:
- CONTROL01.CTL
- CONTROL02.CTL

23. Edit the newly created Pfile and change directory paths in for the parameter “control_Files” to the mirrored disk paths used in the previous step.

24. Recreate system parameter file:
```bat
SQL> create spfile From pfile;
```
The output is as Follows:
**file created.**

25. Restart target database instance with new parameter file:
```bat
SQL> startup
```
The output is as Follows:

```bat
ORACLE instance started.
Total System Global Area 131144544 bytes
Fixed Size                   453472 bytes
Variable Size              96468992 bytes
Database buffers           33554432 bytes
Redo buffers                 667648 bytes
Database mounted.
```
26. Verify control file location changes:
```bat
SQL> show parameter control_Files;
```

The new control Files location on the mirrored disk should be checked.

```bat
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string       D:\ORADATA\ORCL\CONTROL01.CTL
                                                  D:\ORADATA\ORCL\CONTROL02.CTL
```

27. Verify new database file locations:

```bat
SQL> select name From v$datafile;
```

The command output should show new location of moved database Files.


28. Verify new database file locations:

```bat
SQL> select *  From v$logfile;
```
The output is as Follows:

```bat
GROUP# STATUS  TYPE
---------- ------- -------
MEMBER
--------------------------------------------------------------------------------
IS_     CON_ID
--- ----------
         1         ONLINE
D:\ORADATA\ORCL\REDO01.LOG
NO           0          2         ONLINE
D:\ORADATA\ORCL\REDO02.LOG
NO           0     GROUP# STATUS  TYPE
---------- ------- -------
MEMBER
--------------------------------------------------------------------------------
IS_     CON_ID
--- ----------          3         ONLINE
D:\ORADATA\ORCL\REDO03.LOG
NO           0
```

29. Create table in oracle
```bat
SQL>   create table test as select * From dual;
```
The output is as Follows: **Table created.**

30. checking created tables
```bat
SQL>select * From test;
```

The output is as Follows:

```bat
D
-
X
```

31. Quit SQLPLus
```bat
SQL> quit;
```

32. Move failover from **Primary server** to **Secondary Server**

##### On Secondary Server

### Change permissions of Files on the Mirror Disk

1. Add fullcontrol permissions to access all Files and Folders in a data partition to orasys and oradb

    - Windows Server 2022
        ```bat
        > icacls <Data partition> /t /c /grant <username>:D

        e.g. 
        > icacls D: /t /c /grant orasys:D
        > icacls D: /t /c /grant oradb:D
        ```

    
2. C:\Windows\system32>sqlplus /nolog

```bat
SQL> conn /as sysdba
SQL>Shutdown immediate;
```

3. Copy pfile From primary server and replace in standby server then create spfile From pfile:
E.g. (C:\app\oradb\product\21c\dbhome_1\database\initorcl.ora)

```bat
SQL> create spfile From pfile;
SQL> Startup;
```

4. Verify control file location changes:

```bat
SQL> show parameter control_Files;
```
The output is as Follows:

```bat
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      D:\ORADATA\ORCL\CONTROL01.CTL,
                                                  D:\ORADATA\ORCL\CONTROL02.CTL
```

5. Verify new database file locations:

```bat
SQL> select name From v$datafile;
```
The command output should show new location of moved database Files

6. Verify new database file locations:
```bat
SQL> select *  From v$logfile;
```
The output is as Follows:

```bat
 GROUP# STATUS  TYPE
---------- ------- -------
MEMBER
--------------------------------------------------------------------------------
IS_     CON_ID
--- ----------
         1         ONLINE
D:\ORADATA\ORCL\REDO01.LOG
NO           0          2         ONLINE
D:\ORADATA\ORCL\REDO02.LOG
NO           0     GROUP# STATUS  TYPE
---------- ------- -------
MEMBER
--------------------------------------------------------------------------------
IS_     CON_ID
--- ----------          3         ONLINE
D:\ORADATA\ORCL\REDO03.LOG
NO           0
```

7. checking created tables
```bat
SQL>select * From test;
```
The output is as Follows:

```bat
D
-
X
```

8. Create table in oracle

```bat
SQL>   create table test1 as select * From dual;
```
The output is as Follows: **Table created.**

9. Move Group failover From **Secondry server** to **Primary server**

10. checking created tables

```bat
SQL>select * From test1;
```
The output is as Follows:

```bat
D
-
X
```

### Configure a Client

##### On Client
1. Create tnsnames.ora

    Sample of tnsnames.ora
    
    (%ORACLE_HOME%\network\admin\tnsnames.ora)
    ```bat
    orcl =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ENABLE = BROKEN)
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = orcl)
    		)
    	)
    
    orclPDB =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ENABLE = BROKEN)
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <Floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = orclpdb)
    		)
    	)
    ```

### Create Listener Service

##### On Primary Server
1. Move a failover group to a primary server and start OracleService

2. Set environment variables
    ```bat
    > set ORACLE_SID=orcl
    > setx ORACLE_SID "orcl" /m 
    ```

3. Start Command Prompt as an Administrator and start LSNRCTL
    ```bat
    > lsnrctl
    ```
    
4. By **start** command, a listener service is created and started
    ```bat
    LSNRCTL> start listener
    ```
    
5. Connection test (CDB)
    ```bat
    > sqlplus system/<password>@orcl
    ```
    Note: It takes about 60 seconds for the listener to be registered.
    
6. Start PDB and set a startmode of PDB auto
    ```bat
    > sqlplus / as sysdba
    SQL> alter pluggable database all open;
    SQL> alter pluggable database all save state;
    ```
    
7. Connection test (PDB)
    ```bat
    > sqlplus system/<password>@orclPDB
    ```
    
    Note: Normally **PDB**(orclPDB) is used as database. **CDB**(orcl) is used to manage multiple PDBs.
    
8. Connection test **on Client**
    ```bat
    > sqlplus system/<password>@orcl
    > sqlplus system/<password>@orclPDB
    ```
    
9. Change the Startup Type of Oracle services and listener services to **Manual**
    ```bat
    > services.msc
    ```
    - e.g. `OracleServiceorcl` and `OracleOraDB21Home1TNSListener`
    
10. Stop an OracleService and a listener service
    
11. Move a failover group to another server

##### On Secondary Server
12. Create a listener service in the same way as a primary server, but **please skip the step 6**.

### Create Resources for Database on Cluster WebUI

##### On Primary Server
1. Reboot both servers on Cluster WebUI.

1. after rebooting, confirm that an OracleService and a listerner service are stopped on both servers.

1. Create a service resource for database
    - Follow the default dependency
    - **Service Name** is OracleServiceorcl
    
1. Create a service resource for listener
    - **Dependent Resources** is a service resource for database
    - **Service Name** is OracleOraDB21Home1TNSListener
    - **Retry Count at Deactivation failure** should be larger than 0 to avoid a known Oracle Listener stop issue.
    
### Create Monitor Resources on Webui

##### On Primary Server
1. Create an Oracle monitor resource
    - Select a service resource for listener as **Target Resource** of **Monitor Timing**
    - Select **Listener and Instance Monitor** as **Monitor Type**
    - Input a PDB name as **Connect Command**
        - e.g. Connect Command -> orclpdb
    - Input **system** as **User Name**
    - Input password of **system**
    - Select **Default** as **Authority Method**
    - Input **ORACLE_HOME**
        - e.g. ORACLE_HOME -> C:\app\oradb\product\21c\dbhome_1
    - Check **Set error during Oracle initialization or shutdown**
    - Select a failover group as **Recovery Target**
        - e.g. Recovery Target -> Oraclefailover

1. Apply the configuration file
1. Start a failover group

### Change the Password Expiration Date

##### On Primary Server
The default password expiration date for user authentication is 180 days. IF you use Oracle monitor resource, you should change the password regularly or change the password expiration date to an indefinite time.

```bat
> sqlplus / as sysdba
SQL> alter profile Default limit PASSWORD_LIFE_TIME unlimited;
SQL> alter profile ORA_STIG_PROfile limit PASSWORD_LIFE_TIME unlimited;
```
##### On Primary and Secondary Server
The default password expiration data for Windows OS user is 41 days. You should change the password regularly or change the password expiration date to an indefinite time.
- How to disable the password expiration
    - Logon to **Administrator** account
    - Open **Computer Management**
    - Click **Local Users and Groups**
    - Open **Users**
    - Open the property of **orasys**
    - In **General** tab, check **Password never expires**
    - Do the same with **oradb**

----

