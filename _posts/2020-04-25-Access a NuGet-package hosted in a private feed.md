---
layout: post
author: Maxim Braekman
title: "Access a NuGet-package, hosted in a private feed"
description: "Learn how to import a NuGet-package, which is being hosted in a private feed."
image: ./img/devops-private-nuget.jpg
tags: [Azure DevOps, NuGet]
---
In some cases a NuGet-package has been published inside a private feed (Azure Artifacts), within a different project. In which case you might hit some difficulties in running build pipelines for projects referring those packages. 

## Requirements
- [Personal Access Token](#personal-access-token)
- [Service Connection to the private feed](#create-a-service-connection)
- [Configure the build pipeline](#configure-the-build-pipeline)


### Personal Access Token
Ensure you obtain a personal access token linked to the organization in which the package has been published and make sure to enable "Read"-access for packages.

![Create-a-PAT](../../../../img/posts/azure-devops-private-nuget-feed/AzureDevOps_PAT.png)


### Create a Service Connection
In order to identify the location of the private packages and to be able to point towards it from the pipeline, you need to create a service connection.  
When creating a new service connection, make sure to select the "NuGet" connection type.  

![Create-a-ServiceConnection-NuGet](../../../../img/posts/azure-devops-private-nuget-feed/AzureDevOps_ServiceConnection_NuGet.png)

Next to that, make sure to select the 'External Azure DevOps Server'-option, provide the url to the feed you require and fill in the newly created PAT, before saving.  

![Create-a-ServiceConnection-NuGet-ExternalDevOps](../../../../img/posts/azure-devops-private-nuget-feed/AzureDevOps_ServiceConnection_NuGet_ExternalDevOpsServer.png)

### Configure the build pipeline
In order to obtain access to the private NuGet package, ensure to add the following tasks:
#### NuGet Authenticate  
Used to verify if the provided feed is accessible by making use of the newly created service connection.   

{% highlight yml %}
    steps:
    - task: NuGetAuthenticate@0
      displayName: 'NuGet Authenticate'
      inputs:
        nuGetServiceConnections: 'PrivateFeed_NuGetPackage'
{% endhighlight %}
    
> *Note: You might notice that in some cases this task is not required.*

#### NuGet restore  
Used to _restore_ the required NuGet-packages, by downloading those required for the MSBuild action to succeed.  

{% highlight yml %}
    steps:
    - task: NuGetCommand@2
      displayName: 'NuGet restore'
      inputs:
        restoreSolution: src/PrivateFeedUsage/PrivateFeedUsage.sln
        feedsToUse: config
        nugetConfigPath: src/PrivateFeedUsage/PrivateFeedUsage/nuget.config
        externalFeedCredentials: 'PrivateFeed_NuGetPackage'
{% endhighlight %}

	
In this task, again, you are making use of the same service connection, while pointing towards the solution for which the packages are to be restored.  
But you also need to point towards a custom **nuget.config**-file added to the source code. The purpose of this config-file is to provide the location of the private NuGet feed, since it's not publicly available/known.

Below is an example of how such a nuget.config-file might look like:

	
{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<configuration>
	<packageSources>
		<clear />
		<add key="nuget.org" value="https://www.nuget.org/api/v2/" />
		<add key="PrivateFeed.NuGet.Package" value="https://mbraekman.pkgs.visualstudio.com/_packaging/my-private-feed/nuget/v3/index.json" />
	</packageSources>
</configuration>
{% endhighlight %}
