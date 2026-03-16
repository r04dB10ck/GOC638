# LAB - AlwaysOn Failover Cluster Instances

**Heslo**: Pa$$w0rd

## DC
- skupina Global/Security pro SQL Servery SQLNodes, cleny jsou SQLFCI1 a SQLFCI2
```
new-adgroup -groupcategory security -groupscope global -name SQLNodes
```
- ucet pro beh SQL Serveru gMSA
```
new-adserviceaccount -name gMSASQL01 -samaccountname gMSASQL01 -dnshostname gMSASQL01.gopas.virual -managedpasswordintervalindays 30 -PrincipalsAllowedToRetrieveManagedPassword SQLNodes
```
- computer account pro WSFC, opravneni pro **db-admin** nastavit na **full control**, disabled
- computer account pro SQLcluster, opravneni pro vyse vytvoreny ucet WSFC, disabled

[Prestage cluster Objects](https://learn.microsoft.com/en-us/windows-server/failover-clustering/prestage-cluster-adds)

[SQL Account Overview](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-windows-service-accounts-and-permissions?view=sql-server-ver16)


## DC (emulace SAN / SDS)
- server manager - iSCSI, new virtual disk, nastavit kdo se muze pripojit (fci1 a fci2)

## SQLFCI1
- install WSFC
- install AD PowerShell (RSAT)
```
$servers = ('SQLFCI1','SQLFCI2')
foreach ($server in $servers) {install-windowsFeature -name RSAT-AD-PowerShell, Failover-Clustering, RSAT-Clustering -computername $server}
```
- vytvorit cluster pomoci cluster konzole, validovat pozdeji (u validacnich vynechat storage)
- spustit iSCSI Initiator, pripojit na server **DATA**
- disk na online, disk na initialize, vytvorit volume, naformatovat, pripojit do WSFC
- Instalacni medium sql je na disku E FCI1 a FCI2, lepe zkopirovat na disk C, vytvorit slozku updates a do ni nahrat CU29
- install gMSA
```
Install-AdServiceAccount -identity gMSASQL01
Test-AdServiceAccount -identity gMSASQL01
```
- instalace SQL Serveru z prikazove radky (vytvoreni cluster instance)
```
setup.exe /action=InstallFailoverCluster /updateenabled=true /updatesource="[path]"
```
- upravit local tempDB
```
--zjisteni kde TempDB lezi
SELECT name,
       physical_name AS CurrentLocation
FROM sys.master_files
WHERE database_id = 2;

--presun jednotlivych souboru (fyzicky se nic nepresouva, pouze zmena metadat v systemovem catalogu) filename je nova cesta, kde bude po restartu TempDB databaze a jeji soubory
ALTER DATABASE tempdb
    MODIFY FILE (NAME = tempdev, FILENAME = 'C:\SQLTemp\tempdb.mdf');
```
[Presun SQL Databazi](https://learn.microsoft.com/en-us/sql/relational-databases/databases/move-system-databases?view=sql-server-ver16)

## SQLFCI2
- install WSFC
- install AD PowerShell (RSAT)
- spustit iSCSI Initiator, pripojit na server **DATA**
- install gMSA
```
Install-AdServiceAccount -identity gMSASQL01
Test-AdServiceAccount -identity gMSASQL01
```
- instalace SQL Serveru z prikazove radky (pridani nodu do existujiciho clusteru)
```
setup.exe /action=AddNode /updateenabled=true /updatesource="[path]"
```
- nachystat slozku pro TempDB se stejnou filepath jako na **SQLFCI1**

## SQL CLUSTER
- install CU30 (pasivni, failover, pasivni)
- vypnout SharedMemory v ramci Network Protokolu (SQL SERVER CONFIG MANAGER)
- nastavit Kerberos pro pripojeni k SQL clusteru `setspn -l gmsaSQL01`
-- setspn -s 
-- Kerberos Configuration Tool
-- automatic registration v Active Directory
```
dsacls (Get-ADServiceAccount identity gMSASQL01).DistinguishedName /G "SELF:RPWP;servicePrincipalName"
```

# LAB - Always On Availability Groups

**Heslo**: Pa$$w0rd

## DC
- skupina Global/Security pro SQL Servery SQLAGNodes, cleny jsou SQLAG1 a SQLAG2
```
new-adgroup -groupcategory security -groupscope global -name SQLAGNodes
```
- ucet pro beh SQL Serveru gMSA
```
new-adserviceaccount -name gMSASQL02 -samaccountname gMSASQL02 -dnshostname gMSASQL02.gopas.virtual -managedpasswordintervalindays 30 -PrincipalsAllowedToRetrieveManagedPassword SQLAGNodes
```
- computer account pro WSFC cluster **WSFC-SQL-AG-01**, opravneni pro **db-admin** nastavit na **full control**, disabled
- computer account pro AG listener **SQLAG-LIS-01**, opravneni pro vyse vytvoreny ucet WSFC, disabled

[Prestage cluster Objects](https://learn.microsoft.com/en-us/windows-server/failover-clustering/prestage-cluster-adds)

## SQLAG1
- install WSFC a RSAT
```
$servers = ('SQLAG1','SQLAG2')
foreach ($server in $servers) {install-windowsFeature -name RSAT-AD-PowerShell, Failover-Clustering, RSAT-Clustering -computername $server}
```
- vytvorit cluster pomoci cluster konzole (bez sdileneho uloziste, pouze quorum disk nebo file share witness)
- install gMSA
```
Install-AdServiceAccount -identity gMSASQL02
Test-AdServiceAccount -identity gMSASQL02
```
- instalace SQL Serveru jako standalone instance (zadna sdilena storage)
```
setup.exe /action=Install /updateenabled=true /updatesource="[path]"
```
- povolit funkci Always On Availability Groups v SQL Server Configuration Manager nebo PowerShell
```
Enable-SqlAlwaysOn -ServerInstance SQLAG1 -Force
```
- vytvorit endpoint pro Database Mirroring
```sql
CREATE ENDPOINT [Hadr_endpoint]
    AS TCP (LISTENER_PORT = 5022)
    FOR DATA_MIRRORING (ROLE = ALL, ENCRYPTION = REQUIRED ALGORITHM AES);

ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
```
- vytvorit testovaci databazi a zazalohovat ji (plny backup + log backup)
```sql
CREATE DATABASE AGTestDB;
ALTER DATABASE AGTestDB SET RECOVERY FULL;
BACKUP DATABASE AGTestDB TO DISK = 'C:\Backup\AGTestDB.bak' WITH FORMAT;
BACKUP LOG AGTestDB TO DISK = 'C:\Backup\AGTestDB_log.bak';
```

## SQLAG2
- install WSFC a RSAT (viz SQLAG1)
- pripojit node do existujiciho clusteru
- install gMSA
```
Install-AdServiceAccount -identity gMSASQL02
Test-AdServiceAccount -identity gMSASQL02
```
- instalace SQL Serveru jako standalone instance
```
setup.exe /action=Install /updateenabled=true /updatesource="[path]"
```
- povolit funkci Always On Availability Groups
```
Enable-SqlAlwaysOn -ServerInstance SQLAG2 -Force
```
- vytvorit endpoint pro Database Mirroring
```sql
CREATE ENDPOINT [Hadr_endpoint]
    AS TCP (LISTENER_PORT = 5022)
    FOR DATA_MIRRORING (ROLE = ALL, ENCRYPTION = REQUIRED ALGORITHM AES);

ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
```
- obnovit databazi ze zalohy (NORECOVERY)
```sql
RESTORE DATABASE AGTestDB FROM DISK = 'C:\Backup\AGTestDB.bak' WITH NORECOVERY;
RESTORE LOG AGTestDB FROM DISK = 'C:\Backup\AGTestDB_log.bak' WITH NORECOVERY;
```

## SQL AG konfigurace (na SQLAG1)
- vytvorit Availability Group
```sql
CREATE AVAILABILITY GROUP [AG01]
    WITH (AUTOMATED_BACKUP_PREFERENCE = SECONDARY)
    FOR DATABASE [AGTestDB]
    REPLICA ON
        N'SQLAG1' WITH (
            ENDPOINT_URL = N'TCP://SQLAG1.gopas.virtual:5022',
            FAILOVER_MODE = AUTOMATIC,
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            BACKUP_PRIORITY = 50,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)),
        N'SQLAG2' WITH (
            ENDPOINT_URL = N'TCP://SQLAG2.gopas.virtual:5022',
            FAILOVER_MODE = AUTOMATIC,
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            BACKUP_PRIORITY = 50,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = NO));
```
- pripojit sekundarni repliku na SQLAG2
```sql
ALTER AVAILABILITY GROUP [AG01] JOIN;
ALTER DATABASE [AGTestDB] SET HADR AVAILABILITY GROUP = [AG01];
```
- pridat listener
```sql
ALTER AVAILABILITY GROUP [AG01]
    ADD LISTENER N'SQLAG-LIS-01' (
        WITH IP ((N'[IP adresa]', N'[Maska]')),
        PORT = 1433);
```
- otestovat failover
```sql
-- na primarni replike
ALTER AVAILABILITY GROUP [AG01] FAILOVER;
```

[Always On Availability Groups Overview](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver16)

[Create Availability Group](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/creation-and-configuration-of-availability-groups-sql-server?view=sql-server-ver16)
