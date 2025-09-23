First, make sure there are no remnants of SQL Server or Motorola software on the computer from a previous install. Use Revo uninstaller to remove all instances of 
anything sql server or motorola, delete any folders for SQL Server, Microsoft SQL Server, or Motorola from Program Files, Program Files (x86) and ProgramData. 
Paste the following into an administrator-elevated powershell to force a purge of anything with "SQL Server" or "Motorola" in its name.


```
$removeall = {
for ($i = 1; $i -le $maxAttempts; $i++) {
    Write-Host "Pass $i of $maxAttempts"
    $found = $false
    $x = Get-WmiObject Win32_Product
    $x | ForEach-Object { 
        if($_.Name -like "*$($search)*") {
            $found = $true
            Write-Host "Uninstalling: $($_.Name)"
            Start-Process -Wait -FilePath 'msiexec' -ArgumentList "/qn /x `"$($_.IdentifyingNumber)`""
        }
    }
    if (-not $found) {
        Write-Host "No more $($search) installations found after $i passes."
        break
    }
}
}
$maxAttempts = 5
$search = "SQL Server"
&$removeall
$search = 'Motorola'
&$removeall
```


Make sure all Windows updates are installed and Windows is freshly rebooted before continuing. 

Now run this command as admin:


```
fsutil fsinfo sectorinfo C:
```


If the  PhysicalBytesPerSectorForAtomicity and PhysicalBytesPerSectorForPerformance for the disk you're trying to install on is 4096 or more, SQL Server WILL NOT
WORK without this workaround (requires reboot to take effect):


```
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\stornvme\Parameters\Device" -Name   "ForcedPhysicalSectorSizeInBytes" -PropertyType MultiString        -Force -Value "* 4095"
```


Next issue is the way Motorola allocates privileges to the different Windows service accounts when installing SQL Express 2022. As a result, we want to install it
ourselves. Unzip the RM zip, find the SQL Express installer and invoke it with another admin cli command:


```
SQLEXPRADV_x64_ENU.exe /ACTION=Install /Q /INSTANCENAME=MOTORMSVR2 /FEATURES=SQLEngine `
/ADDCURRENTUSERASSQLADMIN=TRUE /SQLSYSADMINACCOUNTS="NT AUTHORITY\SYSTEM" `
/TCPENABLED=1 /SKIPRULES=RebootRequiredCheck /IACCEPTSQLSERVERLICENSETERMS `
/FILESTREAMLEVEL=3 /FILESTREAMSHARENAME=MOTORMSVR2 /UpdateEnabled=0 `
/SQLCOLLATION=SQL_Latin1_General_CP1_CI_AS
```


here's we've removed Motorola's unnecessary privilege separation and we no longer specify the directory, allowing SQL Server to install to its default location. for
reference, here's the original command line (from the RM Installer Logs):


```
###DO NOT USE, ORIGINAL FOR REFERENCE
###SQLEXPRADV_x64_ENU.exe" /ACTION=Install /Q /INSTANCENAME=MOTORMSVR2 /FEATURES=SQLEngine /ADDCURRENTUSERASSQLADMIN=TRUE /SQLSYSADMINACCOUNTS="NT AUTHORITY\SYSTEM" /INDICATEPROGRESS /TCPENABLED=1 /SKIPRULES=RebootRequiredCheck /IACCEPTSQLSERVERLICENSETERMS /FILESTREAMLEVEL=3 /FILESTREAMSHARENAME=MOTORMSVR2 /UpdateEnabled=0 /SQLCOLLATION=SQL_Latin1_General_CP1_CI_AS
```


Now run the RM installer as admin and install normally, selecting all features. It should complete successfully. 


RM Server should be running and RM Client / etc should be working now. 
