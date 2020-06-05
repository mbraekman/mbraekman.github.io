---
layout: post
author: Maxim Braekman
title: How to deploy SSIS packages using Azure DevOps
description: Find out how you can expose a WSDL through API Management.
image: ./img/azure-devops-ssis-deploy.jpg
tags: [Azure DevOps, SSIS, SQL]
---
Since a lot of our code-bases are moving to Azure DevOps nowadays, so are the SSIS-projects. But how can we *easily* deploy these onto our on-prem SQL Servers?

## Requirements
- [SSIS DevOps Tools](#ssis-devops-tools)
- [Configure the build pipeline](#configure-the-build-pipeline)
- [Configure the release pipeline](#configure-the-release-pipeline)

### SSIS DevOps Tools
In the past, whenever you wanted to get an SSIS-package released onto your SQL-Server by using Azure DevOps, you needed to make use of the PowerShell-script provided in [the official Microsoft guidelines](https://docs.microsoft.com/en-us/sql/integration-services/lift-shift/ssis-azure-deploy-run-monitor-tutorial?view=sql-server-2017#deploy-a-project-with-powershell).  
While this allowed you obtain your objective - getting the SSIS-package deployed - it was quite the hassle to get it configured and running succesfully.

Luckily, in December 2019, Microsoft decided to release a new DevOps-task to the VisualStudio marketplace, specifically designed to build/deploy SSIS-packages from Azure DevOps.  
This package is called the *SSIS DevOps Tools* and can be found [here](https://marketplace.visualstudio.com/items?itemName=SSIS.ssis-devops-tools). So, before you can start setting up any pipelines in DevOps, make sure to install these tools in you Azure DevOps Organization. Or, lets face it, have them installed by the admin/owner of the DevOps Organization, because we rarely our allowed to do this ourselves.

![SSIS-DevOps-Tools](../../../../img/posts/azure-devops-deploying-ssis-packages/devops-marketplace-ssis-devops-tools.png)

An overview of the details and release notes can be found on [this page](https://docs.microsoft.com/nl-nl/sql/integration-services/devops/ssis-devops-overview?view=sql-server-ver15#ssis-catalog-configuration-task).

### Configure the build pipeline
As is the case for any type of project, any release starts off with creating a release package, containing all required artifacts.  
In order to create such a package a build pipeline has to be created and will, in this case, make use of the '**Build SSIS**'-task, which is part of the earlier installed tools.  
The configuration of this task is quite simple, as you only need to specify the location of the project containing the actual SSIS-package(s). 

{% highlight yml %}
- task: SSIS.ssis-devops-tools.ssis-build-task.SSISBuild@0
  displayName: 'Build SSIS'
  inputs:
    projectPath: src/SSIS/SSIS.Package/SSIS.Package.Sample.dtproj
    outputPath: '$(Build.ArtifactStagingDirectory)\SSISBuild'
{% endhighlight %}
Note: no special agent is required here - you can simply use the '*vs2017-win2016*'-agent located in the '*Azure Pipelines*'-pool.

Besides that, just add an additional task to publish the output of the build and you'll be good to go start on the release pipeline.

{% highlight yml %}
- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact: SSIS'
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)\SSISBuild'
    ArtifactName: SSIS
{% endhighlight %}

### Configure the release pipeline

One the required artifacts have been published by the build pipeline, the next step is to create the release pipeline to actually deploy the SSIS-package(s). As part of the SSIS DevOps Tools a '**Deploy SSIS**'-task has been made available to help you deploy the packages onto the SQL-Server.  
In order to make sure Azure DevOps is able to connect to the actual database server, an agent installed needs to be installed on that server. Depending on which type of pipeline you will be making use of, you'll either need to:
- create an **agent pool** and register a new agent when making use of a **YAML**-pipeline, since at the time of writing deployment groups are *not supported* yet in YAML-pipelines.

- creating a **deployment group** and use of the provided PowerShell-command to get the agent installed and configured, when making use of a **classic release pipeline** (UI)

Once the agent has been installed, make sure the release pipeline is configured to make use of this agent and add the **Deploy SSIS**-task with the following configuration.

{% highlight yml %}
- task: SSIS.ssis-devops-tools.ssis-deploy-task.SSISDeploy@0
  displayName: 'Deploy SSIS'
  inputs:
    sourcePath: '$(System.DefaultWorkingDirectory)/_SSIS/SSISBuild/SSIS.Package.Sample.ispac'
    destinationType: ssisdb
    destinationServer: 'MB-SQL-01'
    destinationPath: '/SSISDB/DevOps'
    authType: winAuth
{% endhighlight %}
