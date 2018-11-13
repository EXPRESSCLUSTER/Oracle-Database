# Oracle Database 12c Quick Start Guide for EXPRESSCLUSTER X (Linux Data mirror)

## About This Guide

This guide provides how to integrate Oracle Database 12c with EXPRESSCLUSTER X using mirror disks with 2 nodes. The guide assumes its readers to have EXPRESSCLUSTER X basic knowledge and setup skills.

## System Overview

### System Requirement
- 2 servers
  - IP rechable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
      - Cluster Partition having volume size of 17MB (for X3.3) or 1024MB (for X4.0).
      - Data Partition having volume size depends on Database sizing.
- Oracle Database are installed on local partition on each server.
- Database files are created on Data Partition.


### System Configuration
- CentOS Linux release 7.5.1804 (Core)
  - Kernel version 3.10.0-862.el7.x86_64
- Oracle Database 12c R2
  - 12c and 12c R1 are not confirmed. Please let the authors (see last part of this guide) know if you have confirmed the procedure in this guide can be applied to other than 12c R2.
- EXPRESSCLUSTER X 3.3 / X4.0 for Linux

		Sample configuration
		
		<LAN>
		 |
		 |  +------------------------------+
		 +--| Primary Server               |
		 |  | - CentOS 7.5                 |
		 |  | - Oracle Database 12c R2     |
		 |  | - EXPRESSCLUSTER X 3.3       |
		 |  |                              |
		 |  | RAM              : 4GB       |
		 |  | Cluster Partition: 17MB      |
		 |  |                    /dev/sdb1 |
		 |  | data partitioin  : 40GB      |
		 |  |                    /dev/sdb2 |
		 |  +-----------+------------------+
		 |              |
		 |              | Mirroring
		 |              |
		 |  +-----------+------------------+
		 +--| Secondary Server             |
		 |  | - CentOS 7.5                 |
		 |  | - Oracle DB 12c R2           |
		 |  | - EXPRESSCLUSTER X 3.3       |
		 |  |                              |
		 |  | RAM              : 4GB       |
		 |  | Cluster Partition: 17MB      |
		 |  |                    /dev/sdb1 |
		 |  | data partitioin  : 40GB      |
		 |  |                    /dev/sdb2 |
		 |  +------------------------------+
		 |

#### Cluster configuration
- Network Partition Resolution resource (PING method) : `pingnp1`

- Failover Group: `failover1`
	- Group resource
		- fip1               : floating IP address resource
		- md1                : mirror disk resource
		- exec1              : exec resource

- Monitor resource

	- db1_oraclew       : Oracle monitor resource for database and listener
	- fipw1             : floating IP monitor resource
	- mdnw1             : mirror connect monitor resource
	- mdw1              : mirror disk monitor resource
	- userw             : user-mode monitor resource

#### Database configuration

- SID name               : sid1
- Database name          : sid1
- PDB name               : sid1pdb
- Oracle base            : `/u01/app/oracle`
- Oracle home            : `/u01/app/oracle/product/12.2.0/dbhome_1`
- Database files location: `/oradata/sid1`
- Fast recovery area     : `/oradata/sid1/fast_recovery_area/sid1`

## System setup

### Basic environmental setup

##### On Primary and Secondary servers
1. Login as root
2. Create OS groups with groupadd command

    ```bat
    e.g.
    
    # groupadd -g 1001 oinstall
    # groupadd -g 1002 dba
    # groupadd -g 1003 oper
    # groupadd -g 1004 backupdba
    # groupadd -g 1005 dgdba
    # groupadd -g 1006 kmdba
    ```
    
3. Create an administrative user for Oracle Database with useradd command

    ```bat
    e.g.
    
    # useradd -u 1100 -g oinstall -G dba,oper,backupdba,dgdba,kmdba oracle
    ```
    
4. Create a directory for Oracle Database

    ```bat
    e.g.
    
    # mkdir -p /u01/app/oracle
    # chown -R oracle:oinstall /u01
    # chmod -R 775 /u01
    ```
    
5. Create a directory for a database on a mirror disk

    ```bat
    e.g.
    
    # mkdir -p /oradata/sid1
    # chmod -R 770 /oradata
    # chown -R oracle:oinstall /oradata
    ```
    
6. Change permission and authority of the directory

    ```bat
    e.g.
    
    # mount /dev/sdb2 /oradata/sid1
    # chmod -R 770 /oradata
    # chown -R oracle:oinstall /oradata
    # umount /dev/sdb2
    ```

### Kernel parameter setup

##### On Primary and Secondary servers
1. Login as root
2. Confirm recommendation values

    - semmsl, semmns, semopm, semmni
    
        ```bat
        # /sbin/sysctl -a | grep sem
        ```
    
        - **semmsl** should be more than **250**.
        - **semmns** should be more than **32000**.
        - **semopm** should be more than **100**.
        - **semmni** should be more than **128**.
    
    - shmmax, shmmni, shmall
    
        ```bat
        # /sbin/sysctl -a | grep shm
        ```
    
        - **shmmax** should be more than **half of RAM size**
        - **shmmni** should be more than **4096**.
        - **shmmax** should be more than **40% of RAM size**
    
    - file-max
    
        ```bat
        # /sbin/sysctl -a | grep file-max
        ```
    
        - **file-max** should be more than **6815744**
    
    - ip_local_port_range
    
        ```bat
        # /sbin/sysctl -a | grep ip_local_port_range
        ```
    
        - **ip_local_port_range** should be more than **9000** and less than **65500**.
    
    - rmem_default, rmem_max
    
        ```bat
        # /sbin/sysctl -a | grep net.core.rmem
        ```
    
        - **rmem_default** should be more than **262144**.
        - **rmem_max** should be more than **4194304**.
    
    - wmem_default, wmem_max
    
        ```bat
        # /sbin/sysctl -a | grep net.core.wmem
        ```
    
        - **wmem_default** should be more than **262144**.
        - **wmem_max** should be more than **1048576**.
    
    - aio-max-nr
    
        ```bat
        # /sbin/sysctl -a | grep aio-max-nr
        ```
    
        - **aio-max-nr** should be more than **1048576**.
    
    - panic_on_oops
    
        ```bat
        # /sbin/sysctl -a | grep panic_on_oops
        ```
    
        - **panic_on_oops** should be more than **1**.

3. Edit `/etc/sysctl.conf` to change kernel parameter

    If kernel parameters are less than recommendation values, change `/etc/sysctl.conf`.
    - Set **ipv4.ip_local_reserved_ports** because the ports that EXPRESSCLUSTER uses should be reserved
    
    ```bat
    e.g.
    
    # vi /etc/sysctl.conf
    
      .
      .
      .
      
    fs.aio-max-nr = 1048576
    fs.file-max = 6815744
    kernel.sem = 250 32000 100 128
    net.ipv4.ip_local_port_range = 30000 65500
    net.ipv4.ip_local_reserved_ports = 29001,29002,29003,29004,29006,29031,29032,29033,29034,29051,29052,29053,29054,29071,29072,29073,29074
    net.core.rmem_default = 262144
    net.core.wmem_default = 262144
    net.core.rmem_max = 4194304
    net.core.wmem_max = 1048576
    kernel.panic_on_oops = 1
    ```
    
    Reflect changes
    
    ```bat
    # /sbin/sysctl -p
    ```
    
### Shell setup

##### On Primary and Secondary servers
1. Login as root
2. Add the below lines to `/etc/security/limits.conf`

    ```bat
    # vi /etc/security/limits.conf
    
      .
      .
      .
    
    oracle		soft	nproc	2047
    oracle		hard	nproc	16384
    oracle		soft	nofile	1024
    oracle		hard	nofile	65536
    oracle		soft	stack	10240
    oracle		hard	stack	32768
    # End of file
    ```
    
3. Add the below line to `/etc/pam.d/system-auth`

    ```bat
    # vi /etc/pam.d/system-auth
    
      .
      .
      .
      
    session		required	pam_limits.so
    ```
    
4. Add the below lines to `/etc/profile`

    ```bat
    # vi /etc/profile
    
      .
      .
      .
      
    if [ $USER = "oracle" ]; then
    	if [ $SHELL = "/bin/ksh" ]; then
    		ulimit -u 16384
    		ulimit -n 65536
    		ulimit -s 32768
    	fi
    	umask 022
    fi
    ```
    
5. Restart server

    ```bat
    # shutdown -r now
    ```

### Install EXPRESSCLUSTER X

##### On Primary and Secondary servers
1. Login as root
2. Install EXPRESSCLUSTER X (ECX)
3. Register ECX licenses
    - EXPRESSCLUSTER X 3.3 for Linux
    - EXPRESSCLUSTER X Replicator 3.3 for Linux
    - EXPRESSCLUSTER X Database Agent 3.3 for Linux

##### On Primary server
4. Create a cluster and a failover group
    - Network partition: 
    
        pingnp1: PING method
        
    - Group:
    
        failover1
        - fip1: floating IP resource
        - md1 : mirror disk resource
        
5. Start group on Primary server


### Install Oracle Database

##### On Primary and Secondary servers
1. Logon as oracle
2. Start **runInstaller** of Oracle Database
3. Configure Security Updates:
    - Uncheck **I wish to receive security updates via My Oracle Support.**
4. Installation Option:
    - Select **Install database software only**
5. Database Installation Options:
    - Select **Single instance database instalation**
6. Database Edition:
    - Select **Enterprise Edition (7.5GB)**
7. Installation Location:
    - Input **Oracle base** and **Software location** in a local partition
      - e.g. Oracle base -> `/u01/app/oracle`
      - e.g. Software location -> `/u01/app/oracle/product/12.2.0/dbhome_1`
8. Create Inventory:
    - Input **Inventory Directory**
      - e.g. Inventory Directory -> `/u01/app/oraInventory`
    - Select **oinstall** as **oraInventory Group Name**
9. Operating System Groups:
    - Select **dba** as **Database Administrator (OSDBA) group**
    - Select **oper** as **Database Operator (OSOPER) group (Optional)**
    - Select **backupdba** as **Database Backup and Recovery (OSBACKUPDBA) group**
    - Select **dgdba** as **Data Guard administrative (OSDGDBA) group**
    - Select **kmdba** as **Encryption Key Management administrative (OSKMDBA) group**
    - Select **dba** as **Real Application Cluster administrative (OSRACDBA) group**
10. Prerequisite Checks:
    - Wait for checking
11. Summary:
    - Click **Install**
12. Install Product:
    - Wait for installation
13. Execute Configuration Scripts
    - Execute the scripts according to messages
14. Set environmental values
    
    ```bat
    $ vi $HOME/.bash_profile
    
    export ORACLE_BASE=/u01/app/oracle
    export ORACLE_HOME=/u01/app/oracle/product/12.2.0/dbhome_1
    export PATH=$ORACLE_HOME/bin:$PATH
    export NLS_LANG=American_America.UTF8
    export ORACLE_SID=sid1
    
    $ source ~/.bash_profile
    ```

### Create Database

##### On Primary Server
1. Logon as oracle
2. Start **dbca**
    
    ```bat
    $ dbca
    ```
    
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
    - Input a location in the Data Partition as **Database files location**
      - e.g. Database files location -> `/oradata/sid1`
8. Fast Recovery Option:
    - Check **Specify Fast Recovery Area**
    - Select **File System** as **Recovery files storage type**
    - Input a location in a Data Partition as **Fast Recovery Area**
      - e.g. Fast Recovery Area -> `/oradata/sid1/fast_recovery_area
      - e.g. Fast Recovery Area size -> 17271MB
    - Check **Enable archiving**
9. Network Configuration:
    - Click **Next**
10. Database Options:
    - Check all components
    - Uncheck all **Include in PDBs** checkboxes
11. Data Vault Option:
    - Click **Next**
12. Configuration Options:
    - Click **Next**
        - Block size and Character sets cannot be changed after creating database
            - If you want to change Block size or Character sets, you must reconstruct database
13. Management Options:
    - Uncheck **Configure Enterprise Manager (EM) database express**
14. User Crredentials:
    - Select **Use different administrative passwords**
    - Input **Password** and **Confirm password**
15. Creation Option:
    - Check **Create database**
16. Summary:
    - Click **Finish**
17. Copy parameter files to a secondary server
    
    ```bat
    e.g.
    
    $ scp $ORACLE_HOME/dbs/hc_sid1.dat oracle@192.168.1.1:$ORACLE_HOME/dbs
    $ scp $ORACLE_HOME/dbs/lkSID1 oracle@192.168.1.1:$ORACLE_HOME/dbs
    $ scp $ORACLE_HOME/dbs/orapwsid1 oracle@192.168.1.1:$ORACLE_HOME/dbs
    $ scp $ORACLE_HOME/dbs/spfilesid1.ora oracle@192.168.1.1:$ORACLE_HOME/dbs
    $ scp -r $ORACLE_BASE/admin/sid1 oracle@192.168.1.1:$ORACLE_BASE/admin/
    $ scp -r $ORACLE_BASE/diag/rdbms/sid1 oracle@192.168.1.1:$ORACLE_BASE/diag/rdbms/
    ```
    
##### On Primary and Secondary servers
17. Login as root
18. Change /dev/shm size
    
    ```bat
    # mount -t tmpfs shmfs -o size=7g /dev/shm
    ```
    
    Add below line to `/etc/fstab`
    
    ```bat
    shmfs /dev/shm tmpfs size=7g 0
    ```
    
### Create listener

##### On Primary and Secondary servers
1. Login as oracle
2. Create listener.ora

    Sample of listener.ora
    
    (%ORACLE_HOME%\network\admin\listener.ora)
    ```bat
    LISTENER =
    	(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    ```
    
3. Create tnsnames.ora

    Sample of tnsnames.ora
    
    (%ORACLE_HOME%\network\admin\tnsnames.ora)
    ```bat
    listener =
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
4. Set LOCAL_LISTENER

    ```bat
    > sqlplus / as sysdba
    SQL> startup
    SQL> alter system set LOCAL_LISTENER='listener';
    ```
    
### Create a parameter file in a Data Partition on the mirror disk

##### On Primary server
1. Copy a spfile to a Data Partition
    
    ```bat
    $ cp $ORACLE_HOME/dbs/spfilesid1.ora /oradata/sid1
    ```
    
2. Create init.ora

    ```bat
    $ vi /oradata/sid1/init.ora
    
    spfile=/oradata/sid1/spfilesid1.ora
    ```
    
### Create scripts for startup and shutdown database

##### On Primary server
1. Create startup.sql

    Sample of startup.sql
    
    (/oradata/sid1/startup.sql)
    ```bat
    whenever sqlerror exit 1
    connect sys/<password> as sysdba
    startup pfile=/oradata/sid1/init.ora
    alter pluggable database all open;
    exit;
    ```
    
2. Create shutdown.sql
    
    Sample of shutdown.sql
    
    (/oradata/sid1/shutdown.sql)
    ```bat
    whenever sqlerror exit 1
    connect sys/<password> as sysdba
    shutdown immediate
    exit;
    ```
    
3. Create start.sh

    Sample of start.sh
    
    This script is used when you create exec resource on EXPRESSCLUSTER.
    ```sh
	#! /bin/sh
	#********************************
	#*           start.sh           *
	#********************************

	#ulimit -s unlimited

	if [ "$CLP_EVENT" = "START" ]
	then
		if [ "$CLP_DISK" = "SUCCESS" ]
		then
			echo "NORMAL1"
			su -l oracle -c 'lsnrctl start listener'
			if [ $? -ne 0 ]
			then
				echo "listener start error"
				exit 1
			fi

			su -l oracle -c 'export ORACLE_SID=sid1pdb;sqlplus /nolog @/oradata/sid1/startup.sql'
			if [ $? -ne 0 ]
			then
				echo "db start error"
				su -l oracle -c 'lsnrctl stop listener'
				exit 1
			fi

			if [ "$CLP_SERVER" = "HOME" ]
			then
				echo "NORMAL2"
			else
				echo "ON_OTHER1"
			fi
		else
			echo "ERROR_DISK from START"
			exit 1
		fi
	elif [ "$CLP_EVENT" = "FAILOVER" ]
	then
		if [ "$CLP_DISK" = "SUCCESS" ]
		then
			echo "FAILOVER1"
			su -l oracle -c 'lsnrctl start listener'
			if [ $? -ne 0 ]
			then
				echo "listener start error"
				exit 1
			fi

			su -l oracle -c 'export ORACLE_SID=sid1pdb;sqlplus /nolog @/oradata/sid1/startup.sql'
			if [ $? -ne 0 ]
			then
				echo "db start error"
				su -l oracle -c 'lsnrctl stop listener'
				exit 1
			fi

			if [ "$CLP_SERVER" = "HOME" ]
			then
				echo "FAILOVER2"
			else
				echo "ON_OTHER2"
			fi
		else
			echo "ERROR_DISK from FAILOVER"
			exit 1
		fi
	else
		echo "NO_CLP"
		exit 1
	fi
	echo "EXIT"
	exit 0
    ```

4. Create stop.sh

    Sample of stop.sh
    
    This script is used when you create exec resource on EXPRESSCLUSTER.
    ```sh
	#! /bin/sh
	#********************************
	#*           stop.sh            *
	#********************************

	#ulimit -s unlimited

	if [ "$CLP_EVENT" = "START" ]
	then
		if [ "$CLP_DISK" = "SUCCESS" ]
		then
			echo "NORMAL1"
			su -l oracle -c 'export ORACLE_SID=sid1pdb;sqlplus /nolog @/oradata/sid1/shutdown.sql'
			if [ $? -ne 0 ]
			then
				echo "db shutdown error"
				exit 1
			fi

			su -l oracle -c 'lsnrctl stop listener'
			if [ $? -ne 0 ]
			then
				echo "listener stop error"
				exit 1
			fi

			if [ "$CLP_SERVER" = "HOME" ]
			then
				echo "NORMAL2"
			else
				echo "ON_OTHER1"
			fi
		else
			echo "ERROR_DISK from START"
			exit 1
		fi
	elif [ "$CLP_EVENT" = "FAILOVER" ]
	then
		if [ "$CLP_DISK" = "SUCCESS" ]
		then
			echo "FAILOVER1"
			su -l oracle -c 'export ORACLE_SID=sid1pdb;sqlplus /nolog @/oradata/sid1/shutdown.sql'
			if [ $? -ne 0 ]
			then
				echo "db shutdown error"
				exit 1
			fi

			su -l oracle -c 'lsnrctl stop listener'
			if [ $? -ne 0 ]
			then
				echo "listener stopt error"
				su -l oracle -c 'lsnrctl stop listener'
				exit 1
			fi

			if [ "$CLP_SERVER" = "HOME" ]
			then
				echo "FAILOVER2"
			else
				echo "ON_OTHER2"
			fi
		else
			echo "ERROR_DISK from FAILOVER"
			exit 1
		fi
	else
		echo "NO_CLP"
		exit 1
	fi
	echo "EXIT"
	exit 0
    ```

### Create resources for database on Builder

##### On Primary server
1. Create a execute resource
    - **Dependent Resources** are a mirror disk resource and a floating IP resource
    - Replace **Start script** and **Stop script** on the scripts created earlier
    - Tune **Timeout** of scripts

### Create monitor resources on Builder

##### On Primary server
1. Create an Oracle monitor resource
    - Select a exec resource as **Target Resource** of **Monitor Timing**
    - Select **listener and instance monitor** as **Monitor Type**
    - Input a database name as **Connect String**
        - e.g. Connect String -> sid1
    - Input **system** as **User Name**
    - Input password of **system**
    - Select **DEFAULT** as **Authority Method**
    - Input **ORACLE_HOME**
        - e.g. ORACLE_HOME -> `/u01/app/oracle/product/12.2.0/dbhome_1/`
    - Select **AMERICAN_AMERICA.UTF8** as **Character Set**
    - Input **Library Path**
        - e.g. Library Path -> `/u01/app/oracle/product/12.2.0/dbhome_1/lib/libclntsh.so.12.1`
    - Select a failover group as **Recovery Target**
        - e.g. Recovery Target -> failover1

3. Apply the configuration file
4. Start a failover group

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
    			(ADDRESS = (PROTOCOL = TCP) (HOST = <floating IP>) (PORT = 1521))
    		)
    		(CONNECT_DATA =
    			(SERVICE_NAME = sid1pdb)
    		)
    	)
    ```

### Connection test

##### On Primary and Secondary server and Client

```bat
> sqlplus system/<password>@SID1
> sqlplus system/<password>@SID1PDB
```
    
Note: Normally **PDB**(SID1PDB) is used as database. **CDB**(SID1) is used to manage multiple PDBs. 

### Change PASSWORD_LIFE_TIME unlimited

##### On Primary server
The default password expiration date for user authentication is 180 days. If you use Oracle monitor resource, you should change the password regularly or change the password expiration date to an indefinite time.

```bat
> sqlplus system/<password>@SID1
SQL> alter profile DEFAULT limit PASSWORD_LIFE_TIME unlimited;
SQL> alter profile ORA_STIG_PROFILE limit PASSWORD_LIFE_TIME unlimited;
```

### Appendix: Sample commands for a database
- Create table

  ```bat
  SQL> create table test (id number(10), name varchar(10));
  ```
  
- Insert a row

  ```bat
  SQL> insert into test values (1, 'ogata');
  ```
  
- Show a table

  ```bat
  SQL> select * from test;
  ```
  
- Remove a table

  ```bat
  SQL> drop table test
  ```

----
2019.11.13	Ogata Yosuke <y-ogata@hg.jp.nec.com>	1st issue
