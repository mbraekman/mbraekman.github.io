---
layout: post
author: Maxim Braekman
title: Azure Key Vault - Empty Secrets
description: You might require the use of an empty secret in Azure Key Vault. But that isn't possible, or is it?!
image: ./img/azure-key-vault-empty-secret.jpg
tags: [Azure Key Vault, Security]
---

Sometimes you would like to create an empty secret in Azure Key Vault, just so you would be able to optimize your ARM/Bicep files to support multiple authentication types, for example.  
However, since the Azure Portal does not allow you to create a Key Vault secret without specifying any value, this seems to be quite impossible. Or is it?  

## Steps
- [Getting Started](#getting-started)
- [Creating the secret](#creating-the-secret)
- [Conclusion](#conclusion)

### Getting Started
Before you can use any of the PowerShell commands mentioned below, you will have to make sure you are connected to the correct Azure Tenant/Subscription.  
This can be done by using the following:  
```powershell
Connect-AzAccount

Set-AzContext -Subscription $subscriptionId -Tenant $tenantId
```  
  
  <br />

### Creating the secret
To create a new secret within an existing Azure Key Vault instance, you need to provide a `SecureString`-object containing the actual secret value.  
Using PowerShell, you would be able to generate this SecureString by using the `ConvertTo-SecureString`-command. Unfortunately, this does not accept providing an empty string either.  

A workaround to this approach is to simply initialize a new SecureString-object which will - since the constructor does not require you to provide any actual value - generate a _blank_ secret for you.  
```powershell
$emptySecret = (new-object System.Security.SecureString)
```

<br />

Once this new object has been instantiated, it is just a matter of passing it along to the proper Az-command to create the _empty_ secret within any given Azure Key Vault.
```powershell
Set-AzKeyVaultSecret -VaultName $keyVaultName -Name $secretName -SecretValue $emptySecret
```  

<br />

### Conclusion
As long as the constructor of this SecureString-object does not get updated to require you to provide the secret value, you will always be able to use this little workaround to create a blank secret in Azure Key Vault.