[![Build status](https://ci.appveyor.com/api/projects/status/ptgyc0e6gvnue6db/branch/master?svg=true)](https://ci.appveyor.com/project/Matticusau/mlsqlps/branch/master)

# mlSqlPs

The **mlSqlPs** module contains the **xSqlServerInstall**, **xSqlHAService**, **xSqlHAEndpoint**, **xSqlHAGroup**, **xWaitForSqlHAGroup**, and **xSqlAlias** DSC resources for installing and configuring a SQL Server. 

## Description

The **mlSqlPs** module contains the **xSqlServerInstall**, **xSqlHAService**, **xSqlHAEndpoint**, **xSqlHAGroup**,  **xWaitForSqlHAGroup**, and **xSqlAlias** DSC resources for installing and configuring a SQL Server. 
These DSC Resources allow you to install a SQL Server from software stored on a network or local share, enable SQL high availability service (HA), and configure SQL HA endpoint. 

Note: These resources assume familiarity with certain aspects of the SQL Server install process. 
SQL Server Enterprise installer requires .NET 3.5 to be installed. 
Therefore, DSC resources that install SQL Enterprise require .NET 3.5 sources to be present on the machine.   

This module is a customised version of **xSqlPs**. While I will work to have my changes merged into the main stream, this module will contain latest changes.

## Resources

* **xSqlServerInstall** installs SQL Enterprise on target machine. 
* **xSqlHAService** enables SQL high availability (HA) service on a given SQL instance. 
* **xSqlHAEndpoint** configures the given instance of SQL high availability service to listen port 5022 with given name, and assigns users that are allowed to communicate through the SQL endpoint. 
* **xSqlHAGroup**configures an SQL HA group. 
If the HA group does not exist it will create one with the given name on given SQL instance and add the HA group database(s) to local SQL instance.
* **xWaitForSqlHAGroup** waits for an SQL HA group to be ready by checking the state of the HA group of a given name in a given interval till either the HA group is discoverable or the number of retries reached its maximum.  
* **xSqlAlias** configures Client Aliases in both native and wow6432node paths. Supports both tcp and named pipe protocols.

### xSqlServerInstall

* **InstanceName**: The name of the SQL instance.
* **SourcePath**: The share path of SQL server software.
* **SourcePathCredential**: The credential that the VM could use to access a net share of SQL server software.
* **VersionID**: The numeric value of the SQL version being installed. Default is 120
* **Features**: List of names of SQL Server features to install.
* **SqlAdministratorCredential**: The SQL Server Administrator credential.
* **UpdateEnabled**: Controls if the install media should check for updates
* **SvcAccount**: Specifies the service account for SQL DB Engine
* **SysAdminAccounts**: Specifies the list of accounts to be added as sysadmins during install
* **AgentSvcAccount**: Specifies the service account for SQL Agent
* **SqlCollation**: Specifies the collation for the instance
* **InstallSqlDataDir**: The path to install the system dbs to
* **SqlTempDBDir**: The path for TempDB .mdf and .ldf
* **SqlUserDBDir**: The default path for all User database .mdf/.ndf files
* **SqlUserDBLogDir**: The default path for all User database .ldf files
* **SqlBackupDir**: The default backup path for the instance


### xSqlHAService

* **InstanceName**: The name of the SQL instance.
* **SqlAdministratorCredential**: The SQL Server Administrator credential.
* **ServiceCredential**: Domain credential used to run SQL Service.

### xSqlHAEndpoint

* **InstanceName**: The name of the SQL instance.
* **AllowedUser**: Unique name for HA database mirroring endpoint of the SQL instance.
* **PortNumber**: The single port number (####) on which the SQL HA to listen to.

### xSqlHAGroup

* **Name**: The name of the SQL Availability Group.
* **Database**: Array of databases on the local SQL instance. 
Each database can belong to only one HA group.
* **ClusterName**: The name of the Windows failover cluster for the availability group.
* **DatabaseBackupPath**: The net share for SQL replication initialization,
* **InstanceName**: Name of the SQL Instance. 
* **EndPointName**: Name of endpoint to access High Availability SQL instance. 
* **DomainCredential**: Domain credential to get list of cluster nodes.
* **SqlAdministratorCredential**: SQL Server Administrator credential. 

### xWaitForSqlHAGroup

* **Name**: The name of SQL High Availability Group.
* **ClusterName**: The name of Windows failover cluster for the availability group.
* **RetryIntervalSec**: Interval to check the HA group existency.
* **RetryCount**: Maximum number of retries to check HA group existency.
* **InstanceName**: The name of SQL instance.
* **DomainCredential**: Domain credential could get list of cluster nodes.
* **SqlAdministratorCredential**: SQL Server Administrator credential .

### xSqlAlias

* **SQLServerName**: The name of Alias (e.g. svr01\inst01).
* **ServerName**: The name of real server.
* **Protocol**: The protocol of either tcp or np (named pipes).
* **RetryCount**: Maximum number of retries to check HA group existency.
* **TCPPort**: The tcp port of the instance.

## Versions

### 1.0.0.3

* Minor fix to the samples files

### 1.0.0.2

* Fixed a bug as logged in Issue #1. Paths containing spaces are now correctly handled

### 1.0.0.0

* Initial release based on version 1.1.3.1 of xSqlPs but with customisations pending a pull request back into that main repository. 
This release includes the following resources 
	- **xSqlAlias**
	- **xSqlServerInstall**
    - **xSqlHAService**
    - **xSqlHAEndpoint**
    - **xSqlHAGroup**
    - **xWaitForSqlHAGroup**


## Examples
### Install SQL on a single node

This example installs SQL Server on a single node. 
Note: This examples has prerequisites that must be met before it can be run. 
It assumes that the .NET 3.5 source is present under C:\Software\sxs, that the SQL full enterprise installer is present under C:\Software\sql, and that a local self singed certificate is prepared.

```powershell
# Configuration to install SQL server database engine and management tools.
# 
# A. Prepare a local self signed certificate with the following steps:
# 1\. Install MakeCert.exe (Microsoft SDK 8.1 http://msdn.microsoft.com/en-us/windows/desktop/bg162891.aspx)
# 2\. Open console with Administrator elevation, run the following:
#     makecert -r -pe -n "CN=DSCDemo" -sky exchange -ss my -sr localMachine
# B. Prepare software and run the configuration.
# 1\. On the machine, create a folder as Software
# 2\. On the machine, copy Windows Server 2012 R2 source\sxs to C:\Software\sxs
# 3\. copy sql full enterprise installation software to C:\Software\sql
# 4\. copy xSqlPs to $env:ProgramFiles\WindowsPowershell\Modules
# 5\. Copy this file (sql101.ps1) to c:\DSCDemo
# 6\. in powershell with administrator elevation, go to c:\DSCDemo, run .\sql101.ps1
$certSubject = "CN=DSCDemo"
$keysFolder = Join-Path $env:SystemDrive -ChildPath "Keys"
$cert = dir Cert:\LocalMachine\My | ? { $_.Subject -eq $certSubject }
if (! (Test-Path $keysFolder ))
{
    md $keysFolder | Out-Null
}
$certPath = Export-Certificate -Cert $cert -FilePath (Join-Path $keysFolder -ChildPath "Dscdemo.cer")
$ConfigData=
@{
    AllNodes = @(
       @{
           NodeName = "localhost"
           CertificateFile = $certPath
           Thumbprint = $cert.Thumbprint
        }
    )
 }
Configuration Sql101
{
    param(
        [Parameter(Mandatory=$true)]
        [ValidateNotNullorEmpty()]
        [PsCredential] $credential
        )
    Import-DscResource -Module xSqlPs
   Node $AllNodes.NodeName
   {

    # Install SQL Server
    WindowsFeature installdotNet35
    {            
        Ensure = "Present"
        Name = "Net-Framework-Core"
        Source = "c:\software\sxs"
    }

    xSqlServerInstall installSqlServer
    {
        InstanceName = "PowerPivot"
        SourcePath = "c:\software\sql"
        Features= "SQLEngine,SSMS"
        SqlAdministratorCredential = $credential
        SqlTempDBDir = "E:\mssql\MSSQL.PowerPivot\Data"
        SqlUserDBDir = "D:\mssql\MSSQL.PowerPivot\Data"
        SqlUserDBLogDir = "D:\mssql\MSSQL.PowerPivot\Data"
        SqlBackupDir = "F:\mssql\MSSQL.PowerPivot\Backup"
        DependsOn = "[WindowsFeature]installdotNet35"
    }
	
	xSqlAlias aliasSQLProdTCP
    {
		Ensure = 'Present';
		SQLServerName = 'SQLProd';
		Protocol = 'tcp';
		ServerName = 'sqlnode01.contoso.com';
		TCPPort = 52001;
		DependsOn = "[xSqlServerInstall]installSqlServer"
    }
	
    LocalConfigurationManager 
    { 
        CertificateId = $node.Thumbprint
        RebootNodeIfNeeded = $true
    } 
 }    
}
Sql101 -ConfigurationData $ConfigData -OutputPath .\Mof -credential (Get-Credential -UserName "sa" -Message "Enter password for SqlAdministrator sa")
Set-DscLocalConfigurationManager .\Mof
Start-DscConfiguration -Path .\Mof -ComputerName localhost -Wait -Verbose
