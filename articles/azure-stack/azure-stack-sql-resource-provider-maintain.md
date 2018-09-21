---
title: Using SQL databases on Azure Stack | Microsoft Docs
description: Learn how you can deploy SQL databases as a service on Azure Stack and the quick steps to deploy the SQL Server resource provider adapter.
services: azure-stack
documentationCenter: ''
author: jeffgilb
manager: femila
editor: ''

ms.service: azure-stack
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 05/24/2018
ms.author: jeffgilb
ms.reviewer: jeffgo
---

# Maintenance operations 
The SQL resource provider is a locked down virtual machine. Updating the resource provider virtual machine's security can be done through the PowerShell Just Enough Administration (JEA) endpoint _DBAdapterMaintenance_. A script is provided with the RP's installation package to facilitate these operations.

## Patching and updating
The SQL resource provider is not serviced as part of Azure Stack as it is an add-on component. Microsoft will be providing updates to the SQL resource provider as necessary. The SQL resource provider is instantiated on a _user_ virtual machine under the Default Provider Subscription. Therefore, it is necessary to provide Windows patches, anti-virus signatures, etc. The Windows update packages that are provided as part of the patch-and-update cycle can be used to apply updates to the Windows VM. When an updated adapter is released, a script is provided to apply the update. This script creates a new RP VM and migrate any state that you already have.

 ## Backup/Restore/Disaster Recovery
 The SQL resource provider is not backed up as part of Azure Stack BC-DR process, as it is an add-on component. Scripts will be provided to facilitate:
- Backing up of necessary state information (stored in an Azure Stack storage account)
- Restoring the RP in the event a complete stack recovery becomes necessary.
Database servers must be recovered first (if necessary), before the resource provider is restored.

## Updating SQL credentials
You are responsible for creating and maintaining system admin accounts on your SQL servers. The resource provider needs an account with these privileges to manage databases on behalf of users - it does not need access to the data in those databases. If you need to update the sa passwords on your SQL servers, you can use the update capability of the resource provider's administrator interface to change the stored password used by the resource provider. These passwords are stored in a Key Vault on your Azure Stack instance.

To modify the settings, click **Browse** &gt; **ADMINISTRATIVE RESOURCES** &gt; **SQL Hosting Servers** &gt; **SQL Logins** and select a login name. The change must be made on the SQL instance first (and any replicas, if necessary). In the **Settings** panel, click on **Password**.

![Update the admin password](./media/azure-stack-sql-rp-deploy/sqlrp-update-password.PNG)

## Secrets rotation 
*These instructions apply only to Azure Stack Integrated Systems Version 1804 and Later. Do not attempt secret rotation on pre-1804 Azure Stack Versions.*

When using the SQL and MySQL resource providers with Azure Stack integrated systems, you can rotate the following infrastructure (deployment) secrets:
- External SSL Certificate [provided during deployment](azure-stack-pki-certs.md).
- The resource provider VM local administrator account password provided during deployment.
- Resource provider diagnostic user (dbadapterdiag) password.

### PowerShell examples for rotating secrets

**Change all secrets at the same time**
```powershell
.\SecretRotationSQLProvider.ps1 `
    -Privilegedendpoint $Privilegedendpoint `
    -CloudAdminCredential $cloudCreds `
    -AzCredential $adminCreds `
    –DiagnosticsUserPassword $passwd `
    -DependencyFilesLocalPath $certPath `
    -DefaultSSLCertificatePassword $certPasswd  `
    -VMLocalCredential $localCreds
```

**Change diagnostic user password only**
```powershell
.\SecretRotationSQLProvider.ps1 `
    -Privilegedendpoint $Privilegedendpoint `
    -CloudAdminCredential $cloudCreds `
    -AzCredential $adminCreds `
    –DiagnosticsUserPassword  $passwd 
```

**Change VM local administrator account password**
```powershell
.\SecretRotationSQLProvider.ps1 `
    -Privilegedendpoint $Privilegedendpoint `
    -CloudAdminCredential $cloudCreds `
    -AzCredential $adminCreds `
    -VMLocalCredential $localCreds
```

**Change SSL Certificate**
```powershell
.\SecretRotationSQLProvider.ps1 `
    -Privilegedendpoint $Privilegedendpoint `
    -CloudAdminCredential $cloudCreds `
    -AzCredential $adminCreds `
    -DependencyFilesLocalPath $certPath `
    -DefaultSSLCertificatePassword $certPasswd 
```

### SecretRotationSQLProvider.ps1 parameters

|Parameter|Description|
|-----|-----|
|AzCredential|Azure Stack Service Admin account credential.|
|CloudAdminCredential|Azure Stack cloud admin domain account credential.|
|PrivilegedEndpoint|Privileged Endpoint to access Get-AzureStackStampInformation.|
|DiagnosticsUserPassword|Diagnostics User password.|
|VMLocalCredential|The local administrator account of the MySQLAdapter VM.|
|DefaultSSLCertificatePassword|Default SSL Certificate (*pfx) Password.|
|DependencyFilesLocalPath|Dependency Files Local Path.|
|     |     |

### Known issues
**Issue**: The logs for secrets rotation are not automatically collected if the secret rotation custom script fails when it is run.

**Workaround**: Use the Get-AzsDBAdapterLogs cmdlet to collect all resource provider logs, including AzureStack.DatabaseAdapter.SecretRotation.ps1_*.log, under C:\Logs.

## Update the virtual machine operating system
There are several ways to update the Windows Server VM:
- Install the latest resource provider package using a currently patched Windows Server 2016 Core image
- Install a Windows Update package during the installation or update of the RP

## Update the virtual machine Windows Defender definitions
Follow these steps to update the Defender definitions:

1. Download the Windows Defender definitions update from [Windows Defender Definition](https://www.microsoft.com/en-us/wdsi/definitions).

    On that page, under “Manually download and install the definitions” download “Windows Defender Antivirus for Windows 10 and Windows 8.1” 64-bit file. 
    
    Direct link: https://go.microsoft.com/fwlink/?LinkID=121721&arch=x64.

2. Create a PowerShell session to the SQL RP adapter virtual machine’s maintenance endpoint
3. Copy the definitions update file to the DB adapter machine using the maintenance endpoint session
4. On the maintenance PowerShell session invoke the _Update-DBAdapterWindowsDefenderDefinitions_ command
5. After install, it is recommended to remove the used definitions update file. It can be removed on the maintenance session using the _Remove-ItemOnUserDrive)_ command.


Here is a sample script to update the Defender definitions (substitute the address or name of the virtual machine with the actual value):

```powershell
# Set credentials for the RP VM local admin user
$vmLocalAdminPass = ConvertTo-SecureString "<local admin user password>" -AsPlainText -Force
$vmLocalAdminUser = "<local admin user name>"
$vmLocalAdminCreds = New-Object System.Management.Automation.PSCredential `
    ($vmLocalAdminUser, $vmLocalAdminPass)

# Public IP Address of the DB adapter machine
$databaseRPMachine  = "<RP VM IP address>"
$localPathToDefenderUpdate = "C:\DefenderUpdates\mpam-fe.exe"

# Download Windows Defender update definitions file from https://www.microsoft.com/en-us/wdsi/definitions. 
Invoke-WebRequest -Uri 'https://go.microsoft.com/fwlink/?LinkID=121721&arch=x64' `
    -Outfile $localPathToDefenderUpdate 

# Create session to the maintenance endpoint
$session = New-PSSession -ComputerName $databaseRPMachine `
    -Credential $vmLocalAdminCreds -ConfigurationName DBAdapterMaintenance
# Copy defender update file to the db adapter machine
Copy-Item -ToSession $session -Path $localPathToDefenderUpdate `
     -Destination "User:\"
# Install the update file
Invoke-Command -Session $session -ScriptBlock `
    {Update-AzSDBAdapterWindowsDefenderDefinition -DefinitionsUpdatePackageFile "User:\mpam-fe.exe"}
# Cleanup the definitions package file and session
Invoke-Command -Session $session -ScriptBlock `
    {Remove-AzSItemOnUserDrive -ItemPath "User:\mpam-fe.exe"}
$session | Remove-PSSession 
```


## Collect diagnostic logs
The SQL resource provider is a locked down virtual machine. If it becomes necessary to collect logs from the virtual machine, a PowerShell Just Enough Administration (JEA) endpoint _DBAdapterDiagnostics_ is provided for that purpose. There are two commands available through this endpoint:

- **Get-AzsDBAdapterLog**. Prepares a zip package containing RP diagnostics logs and puts it on the session user drive. The command can be called with no parameters and will collect the last four hours of logs.
- **Remove-AzsDBAdapterLog**. Cleans up existing log packages on the resource provider VM

A user account called **dbadapterdiag** is created during RP deployment or update for connecting to the diagnostics endpoint for extracting RP logs. The password of this account is the same as the password provided for the local administrator account during deployment/update.

To use these commands, you will need to create a remote PowerShell session to the resource provider virtual machine and invoke the command. You can optionally provide FromDate and ToDate parameters. If you don't specify one or both of these, the FromDate will be four hours before the current time, and the ToDate will be the current time.

This sample script demonstrates the use of these commands:

```powershell
# Create a new diagnostics endpoint session.
$databaseRPMachineIP = '<RP VM IP address>'
$diagnosticsUserName = 'dbadapterdiag'
$diagnosticsUserPassword = '<Enter Diagnostic password>'

$diagCreds = New-Object System.Management.Automation.PSCredential `
        ($diagnosticsUserName, (ConvertTo-SecureString -String $diagnosticsUserPassword -AsPlainText -Force))
$session = New-PSSession -ComputerName $databaseRPMachineIP -Credential $diagCreds `
        -ConfigurationName DBAdapterDiagnostics

# Sample captures logs from the previous one hour
$fromDate = (Get-Date).AddHours(-1)
$dateNow = Get-Date
$sb = {param($d1,$d2) Get-AzSDBAdapterLog -FromDate $d1 -ToDate $d2}
$logs = Invoke-Command -Session $session -ScriptBlock $sb -ArgumentList $fromDate,$dateNow

# Copy the logs
$sourcePath = "User:\{0}" -f $logs
$destinationPackage = Join-Path -Path (Convert-Path '.') -ChildPath $logs
Copy-Item -FromSession $session -Path $sourcePath -Destination $destinationPackage

# Cleanup logs
$cleanup = Invoke-Command -Session $session -ScriptBlock {Remove- AzsDBAdapterLog }
# Close the session
$session | Remove-PSSession
```

## Next steps
[Add SQL Server hosting servers](azure-stack-sql-resource-provider-hosting-servers.md)
