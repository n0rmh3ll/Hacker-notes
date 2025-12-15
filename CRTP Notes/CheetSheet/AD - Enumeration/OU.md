
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

