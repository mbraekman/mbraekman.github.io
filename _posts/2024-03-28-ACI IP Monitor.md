---
layout: post
author: Maxim Braekman
title: Detect changing IP addresses of Azure Container Groups running in a VNET
description: Provide service continuity by avoiding downtime due to undetected configuration changes.
image: ./img/aci-ip-monitor.jpg
tags: [Containers, Architecture, Bicep]
---

Dive into the hidden challenges of running secure applications in Azure Container Instances, where the sun shines on seamless connections until unexpected clouds bring server reboots and IP address changes. Imagine a scenario where your VNET-enabled container instance becomes a crucial piece, accessible only by its elusive IP address. Discover the thrill of finding connections being dropped between your App Services and/or Functions, leaving you to ponder the 'when' and 'how often' of this enigma. While quick fixes may bring temporary relief, our journey takes a turn towards a foolproof solution — the intriguing 'sidecar pattern'.  
Join us as we unravel the mysteries of maintaining a clear sky in your Azure Container Instances journey.  

## Steps
- [Getting Started](#getting-started)
- [Sidecar Pattern](#sidecar-pattern)
- [Checking the IP address](#checking-the-ip-address)
- [Creating the Container image](#creating-the-container-image)
- [Good to know](#good-to-know)
- [Conclusion](#conclusion)

## Getting Started

When you have small workloads that are to be hosted in Azure but are not supported by Azure App Services or Functions, Azure Container Instances could offer a good light-weight solution to have these applications up and running in no time without requiring you to run a full-blown Kubernetes cluster, which, in 95% of the cases, is just pure overkill. A container instance, which is hosting a single container image, is grouped into a container group (what's in a name), which, by default, is publicly accessible and can be addressed by either its IP address or a custom DNS label. However, if you need to have a more secure setup and thus require this container group to become a member of a virtual network, then this option to make use of a custom DNS label is not (yet) supported, and the only way to communicate with the instance is by its IP address.  
In most cases, for weeks or even months in a row, this can run without any hiccups. App Services or Functions connect with the application running in the container; the sun is shining and birds are singing; in short, life is good.  
But sometimes, clouds are inbound, bringing along all kinds of bugs (or features).  
Whatever the reason, it could be a power outage in the Microsoft data center running your services or simply a scheduled maintenance operation. Servers can be rebooted, running images can be moved around, and from an SLA perspective, there's not a single reason why this would impact the SLA of your services.  
Well, it doesn't, unless you have a VNET-enabled Azure Container Instance running, which is only accessible by calling it by its IP address, and all of a sudden this IP address changes.  

In this situation, you will find out — probably a bit too late — that all connections have been dropped between your App Services, Functions,... and the container instance.  
A short-term fix for this would be to simply adjust the configuration in the calling services to have them point to the new IP address. And this works; everything will be up and running again and functioning as well as before.  
However, the sky will always remain cloudy because it's solved for now, but will this issue happen again? When will this happen again? How often will this happen? Should I check this every day, hour or minute?  
The first thing on your mind could, of course, be, to make use of Azure Alerts, which could trigger based on the 'Restart Container Group'-action. That could work, yes, but it's not 100% fool-proof, because if the services are moved without restarting, they can also get a new IP address assigned, messing everything up again without you ever getting the alert.  

So, how can we make sure we've got this thing covered?  
We could embed some kind of functionality inside our application, which runs in the container, to periodically check the IP address and trigger an event if it has changed. But what should we do with those applications where we don't have any control over the container images, and what exactly is running? What if you have 10 different applications running? Do we need to implement the same functionality in all of these applications?  
Let's make it simple and make use of the sidecar pattern to offload this IP-checking functionality from the main container.  

## Sidecar Pattern
But what exactly is this sidecar pattern?  

This pattern is a certain design approach where an additional service — or container, in our case — runs alongside the main application to perform additional tasks that extend or enhance the functionality of this main application without directly affecting the core logic. This also means that our sidecar shouldn't even be running the same language or framework version and could be seen as a completely separate piece of software.  
![Sidecar Pattern](../../../../img/posts/aci-ip-monitor/sidecar-pattern.png)  
_Image source: [Microsoft Learn - Sidecar pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar)_

In our specific situation, we do not require any communication between the main application and our sidecar, since its main purpose is to verify the IP address of our container group.  

## Checking the IP address

There are a couple of ways you could do this, for example by using bash to fetch the IP address by using the ifconfig command:  
```bash
IPaddresses=$(ifconfig | grep 'inet addr:10')
echo "${IPaddresses/'inet addr:'/}" | sed -e 's/^[[:space:]]*//' | cut -d' ' -f1
> 10.0.3.4
```

Another way to do this, and this is the route we took, is to use AZ CLI to fetch the IP address, followed by updating the value stored in KeyVault, which was used as the configuration feed for multiple app services already. Another benefit of using AZ CLI is that we can use a role assignment to grant access for this container instance to upsert the secrets in KeyVault, so we don't need to store or provide any credentials to the container configuration.  
Within our App Services or Functions, we made sure that we specified a ReloadInterval that was small enough to quickly catch any changes in the secret values upon registering KeyVault.  

Which resulted in the following setup:  
![IPMonitor Sidecar](../../../../img/posts/aci-ip-monitor/ContainerInstance-IpMonitor-Sidecar.png)

As mentioned before, we chose to go for the bash-script approach in combination with AZ CLI, as this offers us the possibility to keep it rather simple to maintain yet provide all required functionality, and it allows us to use MSI to authenticate against KeyVault to update any secrets. Of course, all of its information, like the names of the KeyVault, the secret, the container instance, and the resource group, is fed to this script via the environment variables.  
```bash
#!/bin/bash
update_ip()
{
    ACI_IP=$(az container show --name $ACI_INSTANCE_NAME --resource-group $RESOURCE_GROUP --query ipAddress.ip --output tsv)
    timestamp=$(date)
    old_ip=$(az keyvault secret show --vault-name $KEYVAULT_NAME --name $SECRET_NAME --query value --output tsv)
    echo 'Checking IP on' $timestamp
    echo 'OLD IP: ' $old_ip
    echo 'CURRENT IP: ' $ACI_IP
    if [ "$ACI_IP" != "" ]; then
        if [ "$old_ip" != "$ACI_IP" ]; then
            echo 'Updating KeyVault'
            az keyvault secret set --vault-name $KEYVAULT_NAME --name $SECRET_NAME --value $ACI_IP --tags createdBy=ACI createdOn=$timestamp --output none
        else
            echo 'IP has not changed - skipping update'
        fi
    fi
}
az --version
az login --identity --output none

killed=0
while [ $killed -eq 0 ];
do
    update_ip &
    sleep 60
done
```

So, what exactly does this script do? A short recap:  
- Authentication against AZ CLI, using managed identity
- Call the check_ip function every minute
- Check the current IP address of the container instance
- Compare this with the currently stored IP address in KeyVault
- Update the secret if the IP address has been changed

## Creating the Container Image

Now that we have this script, we still need to get it running on an Azure container instance.  
Below is the _dockerfile_ which can be used to create the image that runs the above script on startup.  

```dockerfile
FROM mcr.microsoft.com/azure-cli AS base

WORKDIR /app

COPY ContainerInstanceIpMonitor.sh .

# Give execute permissions to the file
RUN chmod u+x ContainerInstanceIpMonitor.sh

CMD ["bash", "./ContainerInstanceIpMonitor.sh"]
```

Want to run this locally? Then use the next docker-compose.yml file:  
```yml
version: "3"
services:
  ipmonitor:
    build:
      context: .
      dockerfile: Dockerfile
    image: aci.ipmonitor/latest
    environment:
      RESOURCE_GROUP: "rg-awe-sandbox-maxim"
      ACI_INSTANCE_NAME: "ci-awe-sandbox-maxim-dev"
      KEYVAULT_NAME: "kvawesandmaximdev"
      SECRET_NAME: "containerInstance--IP"
```


The downside of this option, of course, is that we have now built our container image and need to host it somewhere. In our case, we've chosen to go for a single Azure Container Registry, which has been made part of a dedicated Azure DevOps pipeline to get this ACR up and running and to build and push the image, since this would be a requirement to have this available before the actual services are being created and also doesn't have to be updated every single time.  

```yaml
  - task: AzurePowerShell@5
    displayName: "Create Azure Container Registry; if it does not exist, build and push the image"
    inputs:
      azureSubscription: '$(azureServiceConnectionName)'
      ScriptType: InlineScript
      Inline: |
        az login --service-principal -u '$(azureServiceConnectionId)' --password="$(devopsAdSPClientSecret)" --tenant "$(tenantId)" --output none
        az account set --subscription '$(azureSubscriptionId)'
        $acr = (az acr show --resource-group '$(resourceGroupName)' --name '$(containerRegistryName)')
        $acr_name = '$(containerRegistryName)'
        if($acr -eq $null)
        {
            Write-Host 'ACR does not exist yet! Creating...'
            az acr create --resource-group '$(resourceGroupName)' --name '$(containerRegistryName)' --sku 'Basic' --output none
            # Enable admin before you can retrieve the password
            az acr update -n $acr_name --admin-enabled true
            Write-Host 'ACR Created!'
        }
        else
        {
            Write-Host 'ACR exists already.'
        }

        # Deploy the image to ACR
        az acr build --resource-group '$(resourceGroupName)' --registry $acr_name --image aci.ipmonitor:$(containerVersion) $(rootDirectory)$(projectName).Deployment/Containers/ContainerInstanceIpMonitor
      FailOnStandardError: false
      azurePowerShellVersion: "LatestVersion"
      pwsh: true
```

If you want to refer to this container image from Bicep when creating the actual sidecar container, this would result in the following container section in Bicep:  

```bicep
{
        name: '${naming.container_instance_av}-ipmonitor'
        properties: {
          command: [
          ]
          environmentVariables: [
            {
              name: 'RESOURCE_GROUP'
              value: resourceGroup().name
            }
            {
              name: 'ACI_INSTANCE_NAME'
              value: naming.container_instance_av
            }
            {
              name: 'KEYVAULT_NAME'
              value: naming.keyVaultShared
            }
            {
              name: 'SECRET_NAME'
              value: 'containerInstance--IP'
            }
          ]
          resources: {
            requests: {
                cpu: 1
                memoryInGB: 1
            }
          }
          image: '${naming.container_registry}.azurecr.io/aci.ipmonitor:${imageVersion}'
        }
      }
```

What does this require when running it inside an Azure Container Instance?  
- The subnet — yes we're still in a VNET — should be granted access to the KeyVault instance.
- The container instance should be granted the "Key Vault Secrets Officer" role on the shared KeyVault instance for it to be able to read and update secrets.
- The container instance should be granted the "Reader" role on itself. It seems like this needs to be explicitly set; otherwise, it is not allowed to request its information using the AZ CLI.

## Good to know
Another, a little less lightweight, solution could be to make use of Azure Container Apps set to an internal accessibility level.  
In this setup, all container apps hosted within the Azure Container Apps Environment are only accessible to services added to any other subnet within the same virtual network. However, this approach offers the advantage of using the Fully Qualified Domain Name (FQDN) instead of an IP address for communication, eliminating the need for setting up a sidecar, as described above.  
This can simplify the configuration and management of your containerized applications while providing enhanced accessibility within the virtual network environment.  
Depending on your overall needs, you could stick to the simple, lightweight, cheaper option of Azure Container Instances, including a sidecar to monitor any IP changes or opt for the more advanced Azure Container Apps in case you have more complex requirements.  

## Conclusion
Want to be sure you're catching the change in IP addresses of a container instance running within a virtual network?  
The easiest way to achieve this is to leverage the sidecar pattern and use a bash script to fetch the IP address. This is easy to reuse on any other container instance you'll be setting up within your subscription without requiring any additional development.  