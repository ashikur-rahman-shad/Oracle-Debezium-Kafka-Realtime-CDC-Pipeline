#### Install Dependencies
```bash
dnf install -y oracle-database-preinstall-19c
```

#### Download and install the Oracle 19c RPM
https://www.oracle.com/apac/database/technologies/oracle19c-linux-downloads.html
Go to the download location and run the following command
```bash
yum localinstall -y oracle-database-ee-19c-1.0-1.x86_64.rpm
```

#### Configure Database
```bash
/etc/init.d/oracledb_ORCLCDB-19c configure
```
Defaults:
- CDB: ORCLCDB
- PDB: ORCLPDB1
- Listener port: 1521

#### Add Oracle to Path
Switch to user `oracle` 
```bash
su - oracle
```

Edit this file: `~/.bash_profile` and add the following lines:
```bash
export ORACLE_BASE=/opt/oracle
export ORACLE_HOME=/opt/oracle/product/19c/dbhome_1
export ORACLE_SID=ORCLCDB
export PATH=$PATH:$ORACLE_HOME/bin
```

Then apply it
```bash
source ~/.bash_profile
```

If firewall is enabled, allow port 1521 for TCP
```bash
firewall-cmd --permanent --add-port=1521/tcp
firewall-cmcd --reload
```

Check Container:
```Oracle
SHOW CON_NAME;
```

To Switch to PDB Session:
```SQL Oracle
SELECT name, open_mode FROM v$pdbs;
```
```SQL
ALTER SESSION SET CONTAINER=ORCLPDB1;
```

To switch to CDB Session:
```SQL Oracle
ALTER SESSION SET CONTAINER=CDB$ROOT;
```

Change system password:
```sql
ALTER USER system IDENTIFIED BY "YourNewPassword123";
```
