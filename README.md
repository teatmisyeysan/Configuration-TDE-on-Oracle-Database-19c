## Using Grid User use asmcmd to create asm wallet folder:
```bash
asmcmd mkdir +DATADISK/wallet
asmcmd mkdir +DATADISK/wallet/PROD
asmcmd mkdir +DATADISK/wallet/PROD/tde
asmcmd ls -l +DATADISK/wallet/PROD

asmcmd mkdir +DATA/wallet
asmcmd mkdir +DATA/wallet/PROD
asmcmd mkdir +DATA/wallet/PROD/tde
asmcmd ls -l +DATA/wallet/PROD
```

## Using Oracle user use sqlplus to configure wallet:
```bash
show parameter wallet_root
alter system set WALLET_ROOT="+DATADISK/wallet/PROD" scope=spfile sid='*';
alter system set WALLET_ROOT="+DATA/wallet/PROD" scope=spfile sid='*';
```

### Using oracle user restart Database:
```bash
srvctl status database -db prod
srvctl stop database -db prod
srvctl start database -db prod
-- using sqlplus 
show parameter wallet_root
```

## Using Oracle user use sqlplus to configure the TDE software Keystore Option:
```bash
show parameter tde_configuration
alter system set TDE_CONFIGURATION="KEYSTORE_CONFIGURATION=FILE" scope=both sid='*';
show parameter tde_configuration

-- check the PDBs if they are in mount (Not Opened)
show pdbs 
alter pluggable database all open;

-- check the status of the wallet 
set lines 999
col wrl_type for a10
col WRL_PARAMETER for a50
col wallet_type for a15
col STATUS for a10
select * from GV$ENCRYPTION_WALLET;


-- below command will create a software keystore, check asm you can find a file ewallet.p12, 
ADMINISTER KEY MANAGEMENT CREATE KEYSTORE IDENTIFIED BY welcome1;

-- command to change the wallet password 

ADMINISTER KEY MANAGEMENT ALTER KEYSTORE PASSWORD IDENTIFIED BY welcome1 SET password1 WITH BACKUP ;


-- check the stats of the wallet
select * from GV$ENCRYPTION_WALLET;

-- create a Auto-LOGIN wallet type 
ADMINISTER KEY MANAGEMENT CREATE  AUTO_LOGIN KEYSTORE FROM KEYSTORE IDENTIFIED BY welcome1;

-- check the stats of the wallet (No Master key yet)
select * from GV$ENCRYPTION_WALLET;

-- force open the software keystore 
administer key management set keystore open force keystore identified by welcome1 container=all;

/* Note 
To switch over to opening the password-protected software keystore when an auto-login keystore is configured and is currently open, specify the FORCE KEYSTORE clause as follows.
*/

 -- Set the Keystore TDE Encryption Master Key
administer key management set key FORCE KEYSTORE identified by welcome1 with backup Container=all; 

-- check the stats of the wallet
select WRL_TYPE, WRL_PARAMETER, STATUS, CON_ID from gv$encryption_wallet;

-- To close the Wallet 
administer key management set keystore close identified by welcome1 container=all;

++ open key store
administer key management set keystore open force keystore identified by welcome1;

++ close keystore 
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE CONTAINER=ALL;
```

## Remove the Auto Login and back to password wallet type:
===========================================================

# using grid user use the asmcmd

cd  +DATADISK/wallet/PROD/tde/
cp cwallet.sso ../
rm cwallet.sso

-- Using oracle sqlplus close wallet 
select * from GV$ENCRYPTION_WALLET;
alter system set wallet close;
select * from GV$ENCRYPTION_WALLET;
administer key management set keystore open identified by welcome1 container=all;
select * from GV$ENCRYPTION_WALLET;


## Re-Enable the Auto_Login wallet:
====================================

# using grid user use the asmcmd

cd  +DATA/wallet/PROD/tde/
cp  ../cwallet.sso .

-- Using oracle sqlplus close wallet :

select * from GV$ENCRYPTION_WALLET;
administer key management set keystore close identified by welcome1 container=all;
select * from GV$ENCRYPTION_WALLET;


## TDE Table-space Encryption :-
===============================
# Offline Table space Encyption:
----------------------------------

-- Create tablespace in PDB database 
	
ALTER SESSION SET CONTAINER = PRODPDB1;

-- encrypt the tablespace by default need to check the system parameter encrypt_new_tablespaces
-- the encrypt_new_tablespaces parameter have 3 values “CLOUD_ONLY / ALWAYS / DDL.”
-- default value is CLOUD_ONLY if you changed it to ALWAYS will encrypt tablespace with AES128 even if there is no “ENCRYPTION” clause specified when creating the tablespace.
	
CREATE TABLESPACE test_ts DATAFILE SIZE 1M;
CREATE TABLE EMPLOYEE (ID NUMBER(5),NAME VARCHAR(42),SALARY NUMBER(10)) TABLESPACE test_ts;

INSERT INTO EMPLOYEE VALUES (001,'JOHN SMITH',15000);
INSERT INTO EMPLOYEE VALUES (002,'SCOTT TIGER',25000);
INSERT INTO EMPLOYEE VALUES (003,'DIANA HAYDEN',35000);

set lines 999
col df_name for a80
col ts_name for a10
select df.name df_name ,ts.name ts_name  from v$datafile df join v$tablespace ts on (df.ts# = ts.ts#);

SELECT tablespace_name, encrypted, status FROM dba_tablespaces;
ALTER TABLESPACE test_ts ENCRYPTION ONLINE USING 'AES256' ENCRYPT;
SELECT tablespace_name, encrypted, status FROM dba_tablespaces;

ALTER TABLESPACE test_ts ENCRYPTION ONLINE DECRYPT;

-- give you more details 
select ts.name , ENCRYPTIONALG, status, ENCRYPTEDTS  from v$encrypted_tablespaces ets join v$tablespace ts on (ets.ts# = ts.ts#);



# Offline Table space Encyption:
----------------------------------
-- Create tablespace in PDB database 
alter session set container = PRODPDB1;
-- encrypt the tablespace by default need to check the system parameter encrypt_new_tablespaces
-- the encrypt_new_tablespaces parameter have 3 values “CLOUD_ONLY / ALWAYS / DDL.”
-- default value is CLOUD_ONLY if you changed it to ALWAYS will encrypt tablespace with AES128 even if there is no “ENCRYPTION” clause specified when creating the tablespace.

CREATE TABLESPACE test_ts DATAFILE SIZE 1M;

col df_name for a80
col ts_name for a10
select df.name df_name ,ts.name ts_name  from v$datafile df join v$tablespace ts on (df.ts# = ts.ts#);

ALTER TABLESPACE test_ts OFFLINE NORMAL;

SELECT tablespace_name, encrypted, status FROM dba_tablespaces;
ALTER TABLESPACE test_ts ENCRYPTION OFFLINE USING 'AES256' ENCRYPT;
SELECT tablespace_name, encrypted, status FROM dba_tablespaces;

ALTER TABLESPACE test_ts ONLINE;

SELECT tablespace_name, encrypted, status FROM dba_tablespaces;
alter tablespace test_ts offline normal;
SELECT tablespace_name, encrypted, status FROM dba_tablespaces;
ALTER TABLESPACE test_ts ENCRYPTION ONLINE DECRYPT;
SELECT tablespace_name, encrypted, status FROM dba_tablespaces;

-- give you more details 
select ts.name , ENCRYPTIONALG, status, ENCRYPTEDTS  from v$encrypted_tablespaces ets join v$tablespace ts on (ets.ts# = ts.ts#);


# Rekey the Tablespace :
==========================

-- Check the Encryption algo 

select ts.name , ENCRYPTIONALG, status, ENCRYPTEDTS  from v$encrypted_tablespaces ets join v$tablespace ts on (ets.ts# = ts.ts#);

ALTER TABLESPACE test_ts ENCRYPTION USING 'AES192' REKEY;

-- we can add below to preform the file name conversion 
-- FILE_NAME_CONVERT = ('SECURE01.DBF', 'SECURE02.DBF');

-- Check the Encryption algo 

select ts.name , ENCRYPTIONALG, status, ENCRYPTEDTS  from v$encrypted_tablespaces ets join v$tablespace ts on (ets.ts# = ts.ts#);



+++ TDE Column Encryption:
==========================
++++ Create user and connection service name:
---------------------------------------------

# using oracle user 
vi $ORACLE_HOME/network/admin/tnsnames.ora

# paste this inside the tnsnames.ora
PRODPDB1 =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.30.111)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = PRODPDB1.oradomain) # oradomain if the domain added 
    )
  )
# print out the file contant
cat $ORACLE_HOME/network/admin/tnsnames.ora 


# using oracle sqlplus : login / as sysdba
---------------------------

CREATE USER tst_user IDENTIFIED BY password DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
grant connect, resource to tst_user;
grant select any dictionary to tst_user; 

# this to show the is for seek of demo don't grant it for any normal user this is DBA role 


# TDE column encrypt by default add salt to the encrypted to make it tough 
for the stealer to perform  Brute forcing attack :
-----------------------------------------------------------------------------
sqlplus tst_user/password@prodpdb1

 CREATE TABLE customer (
    cust_id      NUMBER,
    cust_name    VARCHAR2(100),
    cust_email   VARCHAR2(50) encrypt,
    cust_phone   NUMBER encrypt,
    cust_address VARCHAR2(3000) encrypt
  );
  
  
  INSERT INTO customer VALUES (001,'JOHN SMITH','fc1@gmail.com',010115000,'phnom penh');
  INSERT INTO customer VALUES (002,'SCOTT TIGER','fc2@gmail.com',012125000,'kratie');
  INSERT INTO customer VALUES (003,'DIANA HAYDEN','fc3@gmail.com',090135000,'kompot');
  
  INSERT INTO customer VALUES (004,'DIANA VA','fc3@gmail.com',090135000,'PP');
  
  
  set lines 999
  col cust_name for a15
  col cust_email for a15
  col cust_phone for a15
  col cust_address for a30
  
  select * from customer;
  
  --- if login sys :
  set lines 999
  col cust_name for a15
  col cust_email for a15
  col cust_phone for a15
  col cust_address for a30
  
  select * from tst_user.customer;
  
  -- check the encrypted columns
  set lines 999
  col owner for a15
  col table_name for a15
  col column_name for a15
  col ENCRYPTION_ALG for a30
  SELECT * FROM DBA_ENCRYPTED_COLUMNS;
  
  -- To remove the Salt from the and try different Encryption algorithm 
  
  /* Note: All the encrypted columns in a table must use the same encryption algorithm. If we try to use 	 different encryption algorithms for multiple columns in the same table, we may encounter 				ORA-28340: a different encryption algorithm has been chosen for the table exception.
  */ 
  
  Drop table customer;
  
   CREATE TABLE customer (
    cust_id      NUMBER,
    cust_name    VARCHAR2(100),
    cust_email   VARCHAR2(50) encrypt no salt,
    cust_phone   NUMBER encrypt,
    cust_address VARCHAR2(3000) encrypt using 'AES256'
  );
  
  /* The ALTER TABLE command can be used for encrypting columns in an existing table by either adding an encrypted column or by encrypting an already existing column.
  */
  -- To add an encrypted column to an existing table in the database
  ALTER TABLE customer ADD (cust_ssn VARCHAR2(11) ENCRYPT USING 'AES256' salt);
  
  -- To encrypt an existing column in a table in the database,
  ALTER TABLE customer MODIFY (cust_name encrypt);
  
  -- To decrypt an existing column in a table in the database,
  ALTER TABLE customer MODIFY (cust_name decrypt);
  
  -- To add SALT to an encrypted column in a table in the database,
  ALTER TABLE customer MODIFY (cust_email encrypt salt);
  
  -- To remove SALT from an encrypted column in a table in the database,
  ALTER TABLE customer MODIFY (cust_email encrypt no salt);
  
  -- To change the encrypted key for the table containing one or more encrypted column,
  ALTER TABLE customer rekey;
  
  -- To change the encryption algorithm for the table containing one or more encrypted column,
  ALTER TABLE customer rekey USING '3DES168';
   
  -- We can also use the parameter NOMAC for bypassing the integrity check, thus saving up to 20bytes of        disk space per encrypted value.
  ALTER TABLE customer rekey USING '3DES168' 'NOMAC';
  
  -- The TDE also adds a Message Authentication Code (MAC) to the data for integrity checking. The default      integrity algorithm is SHA-1.
  ALTER TABLE customer rekey USING '3DES168' 'SHA-1';
 
  /* 	Note: If the encrypted column is being indexed, it must be specified without SALT. If not, we may 		  encounter ORA-28338: cannot encrypt indexed column(s) with salt exception.
  */
  
  
 
