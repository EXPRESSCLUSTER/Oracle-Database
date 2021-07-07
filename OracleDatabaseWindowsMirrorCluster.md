# Oracle Database 19c / 12c Quick Start Guide for EXPRESSCLUSTER X (Windows data mirror)

## About This Guide

This guide provides how to integrate Oracle Database 19c / 12c with EXPRESSCLUSTER X (ECX) using mirror disks with 2 nodes. The guide assumes its readers to have EXPRESSCLUSTER X basic knowledge and setup skills.

## System Overview

### System Requirement
- 2 servers
  - IP rechable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
      - Cluster partition size depends on ECX version.
        - X 4.0 or later: 1024MB
        - X 3.3: 17MB
      - Data partition size depends on Database sizing.
- Oracle Database are installed on local partition on each server.
- Database files are created on data partition.


### System Configuration
- Windows Server 2019 Datacenter / 2016 Datacenter
- Oracle Database 19c / 12c R2
  - 12c and 12c R1 are not confirmed. Please let the authors (see last part of this guide) know if you have confirmed the procedure in this guide can be applied to other than 12c R2.
- EXPRESSCLUSTER X 3.3 (11.35) or later

        Sample configuration

		<LAN>
		 |
		 |  +--------------------------+
		 +--| Primary Server           |
		 |  | - Windows Server 2019    |
		 |  | - Oracle Database 19c    |
		 |  | - EXPRESSCLUSTER X 4.3   |
		 |  |                          |
		 |  | RAM   : 8GB              |
		 |  | Disk 0: 30GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 40GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      F: mirror partition |
		 |  +-----------+--------------+
		 |              |
		 |              | Mirroring
		 |              |
		 |  +-----------+--------------+
		 +--| Secondary Server         |
		 |  | - Windows Server 2019    |
		 |  | - Oracle DB 19c          |
		 |  | - EXPRESSCLUSTER X 4.3   |
		 |  |                          |
		 |  | RAM   : 8GB              |
		 |  | Disk 0: 30GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 40GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      F: mirror partition |
		 |  +--------------------------+
		 |

#### Cluster configuration
- Network Partition Resolution resource (PING method) : `pingnp1`

- Failover Group: `failover1`
	- Group resource
		- fip1               : floating IP address resource
		- md1                : mirror disk resource
		- db1_service        : service resource for OracleServiceSID1
		- listener1_service  : service resource for OracleOraDB12Home1TNSListener

- Monitor resource

	- db1_oraclew       : Oracle monitor resource for database
	- listener1_oraclew : Oracle monitor resource for listener
	- fipw1             : floating IP monitor resource
	- mdnw1             : mirror connect monitor resource
	- mdw1              : mirror disk monitor resource
	- servicew1         : service monitor resource for db1_service
	- servicew2         : service monitor resource for listener1_service
	- userw             : user-mode monitor resource

#### Database configuration

- SID name               : sid1
- Database name          : sid1
- PDB name               : sid1pdb
- Oracle base            : `C:\app\oradb`
- Software location      : `c:\app\oradb\product\19.0.0\dbhome_1`
- Database files location: `F:\oradata`
- Fast recovery area     : `F:\fast_recovery_area`

## System setup

### Prerequisite

##### On Primary and Secondary servers
1. Change a virtual memory (swap) size
    1. Open Control Panel
    1. Go to **System and Security** -> **System**
    1. Click **Advanced system settings**
    1. Click **Settings** of **Performance** in **Advanced** tab
    1. Click **Change** in **Advanced** tab
    1. Uncheck **Automatically manage paging file size for all drivers**
    1. Select **Custome size** and input the size to **Initial size** and **Maximize size**
    1. Click **Set** and **OK**
        - If physical memory is between 2 GB and 16 GB, then set virtual memory to 1 times the size of the RAM
        - If physical memory is more than 16 GB, then set virtual memory to 16 GB
    1. Reboot the machine after applying the settings

### Basic cluster setup

##### On Primary and Secondary servers
1. Install EXPRESSCLUSTER X (ECX)
2. Register ECX licenses
    - EXPRESSCLUSTER X 3.3 for Windows
    - EXPRESSCLUSTER X Replicator 3.3 for Windows
    - EXPRESSCLUSTER X Database Agent 3.3 for Windows

##### On Primary server
3. Create a cluster and a failover group
    - Network partition: 
    
        pingnp1: PING method
        
    - Group:
    
        failover1
        - fip1: floating IP resource
        - md1 : mirror disk resource
        
4. Start group on Primary server

### Create OS user

##### On Primary and Secondary servers
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

##### On Primary and Secondary servers
1. Logon as orasys
1. **(In case of 19c)** Create the folder for **%ORACLE_HOME%**
    - e.g. C:\app\oradb\product\19.0.0\dbhome_1
1. **(In case of 19c)** Unzip Oracle installation file to the **%ORACLE_HOME%**
1. Start **setup.exe** of Oracle Database
1. **(In case of 12c)** Configure Security Updates:
    - Uncheck **I wish to receive security updates via My Oracle Support.**
1. Installation Option:
    - **(In case of 12c)** Select **Install database software only**
    - **(In case of 19c)** Select **Set Up Software Only**
1. Database Installation Options:
    - Select **Single instance database instalation**
1. Database Edition:
    - Select **Enterprise Edition (6.0GB)**
1. Oracle Home User:
    - Select **Use Existing Windows User**
    - Input Oracle home user name
        - e.g. oradb
1. Installation Location:
    - Input **Oracle base** and **Software location** in a local partition
      - e.g. Oracle base -> C:\app\oradb
      - e.g. Software location -> C:\app\oradb\product\19.0.0\dbhome_1
    - In this document, **%ORACLE_HOME%** means **Software location**
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
    - e.g. Global database name -> sid1
    - e.g. SID -> sid1
    - Check **Create as Container database**
    - Check **Use Local Undo tablespace for PDBs**
    - Select **Create a Container database with one or more PDBs**
    - e.g. PDB name -> sid1pdb
7. Storage Option:
    - Select **Use following for the database storage attributes**
    - Select **File System** as **Database files storage type**
    - Input a location in the *data partition* as **Database files location**
      - e.g. Database files location -> F:\oradata
8. Fast Recovery Option:
    - Check **Specify Fast Recovery Area**
    - Select **File System** as **Recovery files storage type**
    - Input a location in a data partition as **Fast Recovery Area**
      - e.g. Fast Recovery Area -> F:\fast_recovery_area
      - e.g. Fast Recovery Area size -> 17271MB
    - Check **Enable archiving**
9. Network Configuration:
    - Click **Next**
10. Database Options:
    - Check all components
    - If you need, check **Include in PDBs** checkboxes
11. Data Vault Option:
    - Click **Next**
12. Configuration Options:
    - Click **Next**
        - Block size and Character sets cannot be changed after creating database
            - If you want to change Block size or Character sets, you must reconstruct database
13. Management Options:
    - Uncheck **Configure Enterprise Manager (EM) database express**
14. User Crredentials:
    - Input each passwords
15. Creation Option:
    - Check **Create database**
16. Summary:
    - Click **Finish**
17. After creating the database, run Command Prompt as administrator
    - Right click Command Prompt icon and select **Run as administrator**
18. Change database startmode

    ```bat
    > oradim -edit -sid <SID name> -startmode auto
    
    e.g.
    > oradim -edit -sid sid1 -startmode auto
    ```
    
    Note: The database is started automatically after starting OracleServiceSID1

##### On Secondary server
19. Logon as orasys, and run Command Prompt as administrator
    - Right click Command Prompt icon and select **Run as administrator**

20. Create database service

    ```bat
    > oradim -new -sid <SID name> -startmode auto
    
    e.g.
    > oradim -new -sid sid1 -startmode auto
    ```
    
21. Create a password file

    The password is recommended to be same with the SYS password defined in **14. User Crredentials:**
    
    - If Oracle Database version is earlier than 12.2
    
    ```bat
    > orapwd file=%ORACLE_HOME%\database\PWD<SID name>.ora password=<password>
    
    e.g.
    > orapwd file=C:\app\oradb\product\12.0.0\dbhome_1\database\PWDSID1.ora password=oracle
    ```
    
    - If Oracle Database version is 12.2 or later
    
    ```bat
    > orapwd file=%ORACLE_HOME%\database\PWD<SID name>.ora password=<password> format=12
    
    e.g.
    > orapwd file=C:\app\oradb\product\19.0.0\dbhome_1\database\PWDSID1.ora password=oracle format=12
    ```
    
### Create listener

##### On Primary and Secondary servers
1. Create listener.ora

    Sample of listener.ora
    
    (%ORACLE_HOME%\network\admin\listener.ora)
    ```bat
    LISTENER =
    	(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    ```
    
2. Create tnsnames.ora

    Sample of tnsnames.ora
    
    (%ORACLE_HOME%\network\admin\tnsnames.ora)
    ```bat
    LISTENER =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    	)
    
    SID1 =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = sid1)
    		)
    	)
    
    SID1PDB =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = sid1pdb)
    		)
    	)
    ```
    
##### On Primary server
3. Set LOCAL_LISTENER

    ```bat
    > sqlplus / as sysdba
    SQL> startup
    SQL> alter system set LOCAL_LISTENER='listener';
    ```
    
### Put a parameter file in a data partition on the mirror disk and link it to a database

##### On Primary server
1. Create a text initialization parameter file (PFILE)
    ```bat
    SQL> create pfile from spfile;
    ```
    
2. Copy the PFILE to a data partition
    ```bat
    e.g.
    > copy C:\app\oradb\product\19.0.0\dbhome_1\database\INITSID1.ORA F:
    ```
    
##### On Primary and Secondary server
3. Link the PFILE to the database
    ```bat
    oradim -edit -sid sid1 -pfile <PFILE>
    
    e.g.
    oradim -edit -sid sid1 -pfile F:\INITSID1.ORA
    ```
    
### Change permissions of files on the mirror disk

##### On Primary server
1. Stop OracleService
    - Open Services
        ```bat
        > services.msc
        ```
    - Stop OracleServiceSID1
2. Move a failover group to stand-by server

##### On Secondary server
3. Add fullcontrol permissions to access all files and folders in a data partition to orasys and oradb
    - If the server OS is Windows Server 2016
        ```bat
        > cacls <Data partition> /t /e /c /g <username>:f

        e.g. 
        > cacls F: /t /e /c /g orasys:f
        > cacls F: /t /e /c /g oradb:f
        ```
    - If the server OS is Windows Server 2019
        ```bat
        > icacls <Data partition> /t /c /grant <username>:F

        e.g. 
        > icacls F: /t /c /grant orasys:F
        > icacls F: /t /c /grant oradb:F
        ```
4. Stop OracleService
    - Open Services
        ```bat
        > services.msc
        ```
    - Stop OracleServiceSID1
    
### Configure a client

##### On Client
1. Create tnsnames.ora

    Sample of tnsnames.ora
    
    (%ORACLE_HOME%\network\admin\tnsnames.ora)
    ```bat
    SID1 =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ENABLE = BROKEN)
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = sid1)
    		)
    	)
    
    SID1PDB =
    	(DESCRIPTION =
    		(ADDRESS_LIST =
    			(ENABLE = BROKEN)
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = sid1pdb)
    		)
    	)
    ```

### Create listener service

##### On Primary server
1. Move a failover group to a primary server and start OracleService

2. Set environment variables
    ```bat
    > set ORACLE_SID=sid1
    > setx ORACLE_SID "sid1" /m 
    ```

3. Start Comand Prompt as an Administrator and start LSNRCTL
    ```bat
    > lsnrctl
    ```
    
4. By **start** command, a listener service is created and started
    ```bat
    LSNRCTL> start listener
    ```
    
5. Connection test (CDB)
    ```bat
    > sqlplus system/<password>@SID1
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
    > sqlplus system/<password>@SID1PDB
    ```
    
    Note: Normally **PDB**(SID1PDB) is used as database. **CDB**(SID1) is used to manage multiple PDBs.
    
8. Connection test **on Client**
    ```bat
    > sqlplus system/<password>@SID1
    > sqlplus system/<password>@SID1PDB
    ```
    
9. Change the Startup Type of Oracle services and listener services to **Manual**
    ```bat
    > services.msc
    ```
    - e.g. `OracleServiceSID1` and `OracleOraDB12Home1TNSListener`
    
10. Stop an OracleService and a listener service
    
11. Move a failover group to another server

##### On Secondary server
12. Create a listener service in the same way as a primary server, but **please skip the step 6**.

### Create resources for database on Cluster WebUI

##### On Primary server
1. Reboot both servers on Cluster WebUI.

1. After rebooting, confirm that an OracleService and a listerner service are stopped on both servers.

1. Create a service resource for database
    - Follow the default dependency
    - **Service Name** is OracleServiceSID1
    
1. Create a service resource for listener
    - **Dependent Resources** is a service resource for database
    - **Service Name** is OracleOraDB\<version\>Home1TNSListener
    - **Retry Count at Deactivation Failure** should be larger than 0 to avoid a known Oracle Listener stop issue.
    
### Create monitor resources on Builder

##### On Primary server
1. Create an Oracle monitor resource
    - Select a service resource for listener as **Target Resource** of **Monitor Timing**
    - Select **Listener and Instance Monitor** as **Monitor Type**
    - Input a PDB name as **Connect Command**
        - e.g. Connect Command -> sid1pdb
    - Input **system** as **User Name**
    - Input password of **system**
    - Select **DEFAULT** as **Authority Method**
    - Input **ORACLE_HOME**
        - e.g. ORACLE_HOME -> C:\app\oradb\product\19.0.0\dbhome_1
    - Check **Set error during Oracle initialization or shutdown**
    - Select a failover group as **Recovery Target**
        - e.g. Recovery Target -> failover1

1. Apply the configuration file
1. Start a failover group

### Change the password expiration date

##### On Primary server
The default password expiration date for user authentication is 180 days. If you use Oracle monitor resource, you should change the password regularly or change the password expiration date to an indefinite time.

```bat
> sqlplus / as sysdba
SQL> alter profile DEFAULT limit PASSWORD_LIFE_TIME unlimited;
SQL> alter profile ORA_STIG_PROFILE limit PASSWORD_LIFE_TIME unlimited;
```
##### On Primary and Secondary server
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
2018.09.28	rev.1.2	Ogata Yosuke <ys-ogata@nec.com>	1st issue

2018.09.28	rev.1.3	Miyamoto Kazuyuki <kazuyuki@nec.com> Minor update

2020.09.02  rev.1.4	Ogata Yosuke <ys-ogata@nec.com>	Oracle Database 19c and ECX 4.x is covered

2021.07.07  rev.1.5	Ogata Yosuke <ys-ogata@nec.com>	Minor update
