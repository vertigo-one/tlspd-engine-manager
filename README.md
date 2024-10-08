# TLSPD Engine Manager

A PowerShell script that enables the performing of core TLSPD engine managment functions via Powershell Remoting, eliminating the need to start a remote desktop session to every engine. The default behavoir of the script if only the engines and credential is provided is to get the status of services. 

Comment based help has been implemented so `get-help .\Manage-TLSPDEngines.ps1` will provide help content in the PowerShell console.

## Requirements
- [WinRM/Powershell Remoting](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements?view=powershell-5.1) must be configured as is required for this script to work.
- Tested with powershell 5.1
- Network access over 5986/5986 from the server/workstation the script is run on to each TLSPD engine.
- A user credential with rights to restart services and manipulate files in the Venafi installation directory.

## Features
- Get status of or restart Venafi windows services. Defaults to `VED`, `VenafiESTService`, and `VenafiWcfHost` services. The below services are supported. 
  - VenafiWcfHost
  - VenafiLogServer
  - VenafiESTService
  - VED
- IIS Management: Authentication must be done by a user who has permissions to do this. 
  - Run `iisreset` on each engine.
- Adaptable Management (All types supported.)
  - Copy adpatable driver from local machine to correct engine folder for adaptable type on each specified engines. 
    - Programatic passing of the filepath to the file on the local system or GUI file picker are supported.
  - List adaptable driver of chosen type on each specified engine. 
    - Defaults to *.ps1 files only but all files in folder supported via the `-AllFileTypes` parameter
  - Delete adaptable driver of chosen type on each specified engine.
    - Programatic passing of the file name via `-AdaptableFileName` or choosing from a menu of files retrieved from the servers is supported.
- All interactions that make changes to TLSPD engines will prompt for comfirmation and display the changes requested. 
  - This will occur unless the `-confirm` switch is provided. Be careful with this. 
- Silencing of the informational output is possible with `-Quiet`

## Specifying Engines to perform work on 
The `-EngineName` parameter accepts either a single string or an array of strings where each string is a TLSDC engine. NetBIOS names, IP addresses, or FQDNs are supported and what is needed or avalaible will depend on your environment.

**Connect to a _single_ engine.**
```powershell
-EngineName 'TLSPD01.corp.net`
```

**Connect to _multiple_ engines at once.**
```powershell
#Comma separated list
-EngineName TLSPD01.corp.net,TLSPD02.corp.net

#Comma separated list of quoted strings.
-EngineName 'TLSPD01.corp.net','TLSPD02'

#Create array of engines and pass to command. 
$engines = ('TLSPD01.corp.net','TLSPD02')
-EngineName $engines
```

## Credentials for connecting to the servers
The `-Credential` parameter accepts a powershell credential object that contains the username and password of the user that will be performing the actions on the TLSPD engines. 
If `-Credential` is not provided when the script is run, a popup will appear where the account credentials must be entered.

If connecting across domains, specifying the username as `DOMAIN\user` may be nessesary. 

The user used must have rights to restart services on the TLSPD engines as well as edit files in the Venafi installation directory.


## Managing Services
This script allows getting the status of services or restarting services via providing the `-Status` or `-Restart` switches.
Using either of these switches without specifying the services to manage will default to targeting the `VenafiLogServer`, `VenafiESTService`, and `VED` services.
The default behavoir of the script if only the engines and credentials is provided is to assume that `-status` is also provided and get the status of the above services. 

To target a specific set of services use the `-VenafiServices` parameter and provide the name of the services you wish to restart or get the status of. TAB autocompletion of these services has been implmented. 

**Supported services**: `VenafiWcfHost`, `VenafiLogServer`, `VenafiESTService`, and `VED` 

**Only restart or get the current status of _a single_ service (VED)**
```powershell
-VenafiServices "VED"
```

**Only Restart or get the current status of _both_ the VED and Logging servies**
```powershell
-VenafiServices "VED","VenafiLogServer"
```


## Managing Adaptable Drivers
This script allows listing existing, uploading to, and deleting adaptables present on TPP engines via the `-ListAdaptables`, `-PushAdaptable`, or `-DeleteAdaptable` switches. Determining the installation directory for TLSPD is done via inspecting the value of the `Base Path` property on the `HKLM:\SOFTWARE\Venafi\Platform\` registry key. 

All of the Adaptable features require the use of the `-AdaptableType` parameter to specify what type of Adaptable to manage. Only a *single* type of adaptable is supported per run of the script. 

### Adaptable management ease of use features.
When pushing adaptables onto TPP servers with `-PushAdaptable`, not providing the `-AdaptableFileLocalPath` parameter will result in a windows file picker dialog GUI launching to allow easy navigation of the windows filesystem. 

Deletion of adaptable Drivers supports using the `-ListFromEngine` to have the script first discover what adapatable drivers of the chosen type are already installed and then present back an in terminal menu where the user can choose the adaptable driver to delete. This allows not knowing what the exact name of the adaptable driver file is. 
```
Connecting to TLSPD Engines
Gathering App adaptables
1. DirectAccess.ps1
2. RemoteDesktopGateway.ps1
Enter the number of the adaptable driver to delete (CTRL+C to cancel):
```

## Examples

### Service Manipulation

```powershell
# Get the Status of all Venafi services on TLSPD01.corp.net and TLSPD02.corp.net
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -Status

# Restart the VED (Platform) and Venafi Log Server sevices on TLSPD01.corp.net and TLSPD02.corp.net. 
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -Restart -VenafiServices "VED","VenafiLogServer"
```

### Reset IIS
```powershell
# Restart IIS on TLSPD01.corp.net and TLSPD02.corp.net
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -IISReset

```

### Adaptable Management
```powershell
# List all Adaptable CAs Drivers installed on TLSPD01.corp.net and TLSPD02.corp.net and any configuration files as well. 
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -ListAdaptables -AdaptableType CA -AllFileTypes

# Delete Adaptable Application Drivers from TLSPD01.corp.net and TLSPD02.corp.net and choose the Driver to delete using a menu
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -DeleteAdaptable -AdaptableType App -ListFromEngine

# Delete the CustomAPPDriver.ps1 Adaptable Application Driver from TLSPD01.corp.net and TLSPD02.corp.net. 
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -DeleteAdaptable -AdaptableType App -AdaptableFileName "CustomAPPDriver.ps1"

# Install the mylogdriver.ps1 Adaptable Log Driver onto TLSPD01.corp.net and TLSPD02.corp.net. 
# If -AdaptableFileLocalPath is not provided, a File picker popup will display to allow choosing the file from the local machine.
.\Manage-TLSPDEngines.ps1 -EngineName TLSPD01.corp.net,TLSPD02.corp.net -Credential $credential -PushAdaptable -AdaptableType Log -AdaptableFileLocalPath "C:\testedDrivers\mylogdriver.ps1"
```
