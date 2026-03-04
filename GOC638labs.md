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
## DC
- new gMSA pro SQLAG
- computer account pro cluster **WSFC-SQL-AG-01**, full control pro db-admin
