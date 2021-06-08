---
layout: post
author: Maxim Braekman
title: Azure AD lookups security loophole
description: In some cases, Azure AD offers you a loophole to lookup specific app registration info.
image: ./img/azure-ad-lookup-loophole.jpg
tags: [Azure AD, Security]
---

When you are lacking sufficient access rights to read Azure AD, there are workarounds you can use (*Azure CLI/PowerShell/..*) to make modifications to users/service principals/.., considering you can provide the correct objectId's.

Obtaining these IDs can be difficult without read-access to Azure AD. However, using the proper PowerShell commands you will be able to obtain these values even when you are lacking said access. 

## Steps
- [Getting Started](#getting-started)
- [Retrieve App Registration Info](#retrieve-app-registration-info)
- [Retrieve Service Principal info](#retrieve-service-principal-info)
- [Conclusion](#conclusion)

### Getting Started
Before you can use any of the PowerShell commands mentioned below, you will have to make sure you are connected to Azure AD.  
This can be done by using the following:  
```powershell
Connect-AzureAD
```  
  
  <br />
### Retrieve App Registration Info
The following command would normally allow you to retrieve the objectId/applicationId for an app registration:  

```powershell
Get-AzureADApplication -SearchString 'app-registration-name'
```

However, when executing this command without having sufficient access to Azure AD, you will be getting the following exception:  
```diff
- Get-AzureADApplication: Error occurred while executing GetApplications
- Code: Authorization_RequestDenied
- Message: Insufficient privileges to complete the operation.
- RequestId: 37a6d760-ddf5-4629-a11f-1b50a37f5975
- DateTimeStamp: Thu, 18 Mar 2021 10:16:30 GMT
- HttpStatusCode: Forbidden
- HttpStatusDescription: Forbidden
- HttpResponseStatus: Completed
```  

**But**, when you would use the same command with the `Filter`-parameter , you will see that instead of getting an exception you will get the requested objectId/applicationId as a response instead.

```powershell
Get-AzureADApplication -Filter "DisplayName eq 'app-registration-name'"
```  

![Lookup App Registration Info](../../../../img/posts/azure-ad-lookup-loophole/lookup-app-registration-info.png)  


### Retrieve Service Principal info
The Service Principal can normally be found underneath the Enterprise Applications in Azure AD, which is linked to an app registration.
To retrieve this information, you can also use 2 versions of the same command:

```powershell
Get-AzureADServicePrincipal -SearchString 'app-registration-name'
```  

This will, as is the case for the app registration command, an exception indicating you are lacking sufficient access rights:  
```diff
- Get-AzureADServicePrincipal: Error occurred while executing GetServicePrincipals
- Code: Authorization_RequestDenied
- Message: Insufficient privileges to complete the operation.
- RequestId: f4c9243b-7b1f-4672-833d-633f074ccf1b
- DateTimeStamp: Thu, 18 Mar 2021 09:24:02 GMT
- HttpStatusCode: Forbidden
- HttpStatusDescription: Forbidden
- HttpResponseStatus: Completed
```  
  <br />


But also in this situation, you can use the same command with a different parameter, which will get you the requested objectId/applicationId as a response instead.  

```powershell
Get-AzureADServicePrincipal -Filter "DisplayName eq 'app-registration-name'"
```  

![Lookup Service Principal Info](../../../../img/posts/azure-ad-lookup-loophole/lookup-service-principal-info.png)  

Now you might be wondering, what is the difference between these 2 commands.  
It's quite straightforward, by specifying the exact field you wish to query in the `Filter`-parameter, you're narrowing down the amount of information you need to have access to. Since the DisplayName is normally something that you will always be able to read, this will work as that is the only field that is being queried.  
If you're using the `SearchString`-parameter, the command is not limiting its search-action to the DisplayName, but also looks at all other properties in the app/enterprise application. When trying to read some of the properties you don't have access to, it simply throws the mentioned exception, preventing you even from reading those values that are accessible to you.  

### Conclusion
If you think there is no way for you to continue without waiting for a response from the Infra-team, providing the objectId/applicationId of the Service Principal before you can set up the proper roles, it is always good to have a look at the other parameters which have been made available.