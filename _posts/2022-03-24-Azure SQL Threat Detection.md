---
layout: post
author: Maxim Braekman
title: Azure SQL - Enable Threat Detection using Bicep
description: Enabling Threat Detection on Azure SQL is easy using the Azure Portal, but how can you do this using Bicep?
image: ./img/azure-sql-threat-detection.jpg
tags: [SQL, Security, Bicep]
---

If you are following the Azure Advisor recommendations as much as possible, you might have already noticed that enabling Microsoft Defender for SQL and configuring the vulnerability assessment is always marked as a resolution that will have a high impact on the security level of your environment, but also has a *quick fix* available.  
While this quick fix is indeed a easily set up using the Azure Portal, this does not make it part of your automated deployment procedure. Below you will be able to find all steps required to enable these security features.

## Steps
- [Getting Started](#getting-started)
- [Creating the SQL Server](#creating-the-sql-server)
- [Enabling Threat Detection](#enabling-threat-detection)
- [Setting vulnerability rules baseline](#setting-vulnerability-rules-baseline)
- [Conclusion](#conclusion)

## Getting started
As mentioned, the Azure Advisor section might point out to you that there is a *quick fix* to simply enable Threat Detection on SQL Server, by activating Microsoft Defender for SQL including configuring a vulnerability scan on your server and databases.  
![Advisor recommendations](../../../../img/posts/azure-sql-threat-detection/azure-advisor-recommendations.png)  

But what exactly does this do?  
Well, Microsoft Defender for SQL is an additional service that can be activated within your subscription and will take care of the following things:  
- It helps in __identifying and mitigating potential database vulnerabilities__.  
  Running a vulnerability scan offers an overview of potential vulnerabilities and how to remediate these.
- It helps in __detecting anomalous activities__ which could indicate a threat to your database.  
  This means that it will continuously monitor your SQL server and database to scan for threats such as SQL Injection, brute-force attacks and privilege abuse.  

As you might have guessed by now, all of this doesn't come free of charge.  
At the time of writing, activating Microsoft Defender for SQL on Azure would add a runtime cost of approximately *13,5€/month*, which, in most cases, will be an acceptable cost considering the level of security you will gain.  
For an up-to-date overview of the prices, you can have a look at the pricing page over [here](https://azure.microsoft.com/en-us/pricing/details/defender-for-cloud/).  

## Creating the SQL Server
Before we can enable Microsoft Defender for SQL, we need to create the SQL Server and database themselves of course. Since jumping into the middle of any file is never a good idea, let's start at the beginning of our bicep file and have a look at how we can create this server and database.  

```bicep
param location string = 'westeurope'
param sqlServerName string
param sqlAdministratorLogin string
@secure()
param sqlAdministratorLoginPassword
param sqlDatabaseName string = 'master'

// SQL Server 
resource sqlServer 'Microsoft.Sql/servers@2021-08-01-preview' = {
  name: sqlServerName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    administratorLogin: sqlAdministratorLogin
    administratorLoginPassword: sqlAdministratorLoginPassword
  }
}

// SQL Database: master
resource sqlServer_masterDb 'Microsoft.Sql/servers/databases@2021-08-01-preview' = {
  name: '${sqlServer.name}/${sqlDatabaseName}'
  location: location
  properties: {}
}
```

The above section will create a new Azure SQL Database Server, located in the West Europe data center, in which we are going to create a single database called *master* just as an example.   

## Enabling Threat Detection
Now that we have our SQL server and database created, it is time to enable the additional security measures, as suggested in the advisor recommendations.  
The below step does require us to create an additional Storage Account (or you can use an already existing one) which will be used to store the vulnerability assessment reports. Since we want to use Managed Identities as much as possible, this will require assigning the Storage Blob Contributor role to our SQL Server.  
Notice how, in the current bicep file the actual creation of the Storage Account is not included, instead we are referring to it, by making use of the *existing* keyword, which will simply look up the service and allow us to use it throughout the rest of our current bicep file.  

*Note:*  
*The below section is located in the same bicep file however, the parameters and/or variables have only been included within those sections where they are relevant.*  

```bicep
param storageAccountName string
param loggingRetentionInDays int = '90'

var storageBlobDataContributorRoleId = '${subscription().id}/providers/Microsoft.Authorization/roleDefinitions/ba92f5b4-2d11-453d-a403-e96b0029c9fe'

// Link towards the existing Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2020-08-01-preview' existing = {
  name: storageAccountName
}

// Assign the Blob Contributor role for the SQL Server on the target Storage Account
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2020-10-01-preview' = {
  name: guid(sqlServer.name, storageBlobDataContributorRoleId, storageAccount.name)
  scope: storageAccount
  properties: {
    roleDefinitionId: storageBlobDataContributorRoleId
    principalId: sqlServer.identity.principalId
  }
}

// Enable the Advanced Data Security feature on SQL Server
resource securityAlertPolicy 'Microsoft.Sql/servers/securityAlertPolicies@2020-08-01-preview' = {
  name: '${sqlServer.name}/Default'
  properties: {
    state: 'Enabled'
    retentionDays: loggingRetentionInDays
    emailAddresses: []
    disabledAlerts: []
    emailAccountAdmins: true
  }
}

// Enable the vulnerability assessment
resource vulnerabilityAssessment 'Microsoft.Sql/servers/vulnerabilityAssessments@2020-08-01-preview' = {
  name: '${sqlServer.name}/Default'
  dependsOn: [
    roleAssignment
    securityAlertPolicy
  ]
  properties: {
    recurringScans: {
      emails: []
      emailSubscriptionAdmins: true
      isEnabled: true
    }
    storageContainerPath: '${storageAccount.properties.primaryEndpoints.blob}sql-vulnerability-assessment'
  }
}
```

Ensure that the Advanced Data Security feature has been enabled before triggering the creation of a Vulnerability Assessment resource within Bicep, using the *dependsOn*-property.  
If you do not include this, you will most likely run into the following exception while running the deployment for this resource type:  
```json
{
    "status": "Failed",
    "error": {
        "code": "VulnerabilityAssessmentADSIsDisabled",
        "message": "Advanced Data Security should be enabled in order to use Vulnerability Assessment."
    }
}
```

This means that even though Bicep is good at recognizing dependencies it does require you to manually include it if you don't have an explicit link from within the resource definition.  
And also: <u>order of execution is important</u>!  

## Setting vulnerability rules baseline
Running the above sections will get you a brand new SQL Server and database, including the activation of Microsoft Defender for SQL and the activation of the vulnerability scans however, it will not take into account any of your specific application requirements.  
Meaning that, if you don't want the vulnerability scan to indicate threats on each execution, you need to specify some sort of baseline.  
For example, you might want to specify that the server-level firewall rules should allow other Azure Services to access the server, but not accept any other client IP addresses, you should use the following baseline:  

```bicep
// Baseline for rule VA2065: Server-level firewall rules should be tracked and maintained at a strict minimum
resource vulnerability_baseline_va2065 'Microsoft.Sql/servers/databases/vulnerabilityAssessments/rules/baselines@2020-08-01-preview' = {
  name: '${sqlServer.name}/${sqlDatabaseName}/Default/VA2065/Default'
  properties: {
    baselineResults: [
      {
        result: [
          'AllowAllWindowsAzureIps'
          '0.0.0.0'
          '0.0.0.0'
        ]
      }
    ]
  }
}
```

Or if you want to specify that the vulnerability scan should not give you any failures when a specific user (sqlAdmin in this case) and the AD group for the developers have access to the database, you should use the following baseline:
```bicep
param developerADGroupName string

// Baseline for rule VA2130: Track all users with access to the database
resource vulnerability_baseline_va2130 'Microsoft.Sql/servers/databases/vulnerabilityAssessments/rules/baselines@2020-08-01-preview' = {
  name: '${sqlServer.name}/${sqlDatabaseName}/Default/VA2130/Default'
  properties: {
    baselineResults: [
      {
        result: [
          sqlAdministratorLogin
        ]
      }
      {
        result: [
          developerADGroupName
        ]
      }
    ]
  }
}
```

While at first, it might not be very clear how these resources specify which rules should receive a specific baseline, it is quite straightforward once you see it. Have a look at the structure of the *name*-property:  
  *sqlServerName/${sqlDatabaseName}/Default/{ruleId}/Default*

A full overview of all possible vulnerability rules, including the ruleId and a description, can be found here:
[https://docs.microsoft.com/en-us/azure/azure-sql/database/sql-database-vulnerability-assessment-rules](https://docs.microsoft.com/en-us/azure/azure-sql/database/sql-database-vulnerability-assessment-rules)


## Conclusion
While these features are very easy to activate from within the Azure Portal, it does require some additional steps from within your deployment templates, but at least this ensures you will always have everything deployed properly, validation in accordance to your baseline.  

