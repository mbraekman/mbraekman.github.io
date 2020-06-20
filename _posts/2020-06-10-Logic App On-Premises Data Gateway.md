---
layout: post
author: Maxim Braekman
title: Logic Apps and the On-Premises Data Gateway
description: Some things to keep in mind when setting up the On-Premises Data Gateway.
image: ./img/logic-app-on-prem-data-gateway.jpg
tags: [Azure Logic Apps]
---

These days more and more interfaces are moving to the cloud to enable integrations between all kinds of systems, but in some cases we still require access to on-premises resources, such as file-shares, SQL Servers, Oracle databases, SAP or even BizTalk Server to achieve a full hybrid scenario.  
In this post, I will not go into details about how to install the on-premises data gateway, but instead I'll list up some of the errors you might run into and settings to take into account.   

## Topics
- [Official documentation](#official-documentation)
- [Installation prerequisites](#installation-prerequisites)
- [Where to find the logs](#where-to-find-the-logs)
- [How to use a ODPG-instance across multiple subscriptions](#how-to-use-a-odpg-instance-across-multiple-subscriptions)

#### Official documentation
Below are some useful links towards official Microsoft documentation regarding the On-Premises Data Gateway in combination with Logic Apps:
- [Install the data gateway](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-gateway-install)
- [Connect to on-premises data](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-gateway-connection)

#### Installation prerequisites


#### Where to find the logs


#### How to use an ODPG-instance across multiple subscriptions

Officially since last summer - *and fully functional since a couple of months* - the on-premises data gateway can be shared across multiple subscriptions.  
While at first glance this seems to be something trivial, it is in fact a very useful enhancement.  
Before this feature was made available whenever you wanted to separate your resources into different subscriptions based on the environment and if these resources required access to on-premises resources, you were required to install 1 on-premises data gateway per environment, as you can only register each on-prem gateway once in Azure.  
This also meant that, since you can only install a single gateway per machine, you were required to have at least 1 server, whether this was physical or virtual didn't matter, per environment, even if all of those were pointing towards the same resource, e.g. a file-share.  

With this feature this is no longer required. Simply install an on-premises data gateway onto a server within your network, register it within 1 of your Azure subscriptions and you will be able to connect from all your subscriptions.  
Or, at least, you should be able to connect.  

If you want to create a new ApiConnection from the designer view, you will be able to select your registered gateway from any of the subscriptions available to you.  
![Support on multiple subscriptions](../../../../img/posts/azure-logic-app-on-premises-data-gateway/new-api-connection-multiple-subscriptions.png)  

However, if you've registered the on-premises data gateway within a subscription that hasn't been used for anything else yet, you might run into the following error-message when attempting to create an ApiConnection:
```error
'/subscriptions/{SubscriptionId}/resourceGroups/{ResourceGroup}/providers/Microsoft.Web/connections/filesystem'.   
Encountered internal server error.   
The tracking Id is '67bec0fd-a8d5-49a2-9d84-48481c3c68bf'.
```

The infamous *internal server error*, without any other trace or indication as to what might be causing it.  
This could, of course, have several causes, of which the last one - *it's always the last thing you check* - solved it for me:
- Your on-premises data gateway instance is unable to connect to the internet.  
Even though the install-prerequisites documentation doesn't mention this, some ports need to be opened up, either directly within your server's firewall or further up the chain with your ISP.  
To get an overview of these ports, you can check the following [documentation](https://docs.microsoft.com/en-us/data-integration/gateway/service-gateway-communication#:~:text=Ports).  
Next to the network port tests included in the on-premises data gateway UI, you can also execute below command to verify whether a connection can be made to ServiceBus across the given port:
```powershell
Test-NetConnection -ComputerName {namespace}.servicebus.windows.net -Port 9350
```
*More information about this command can be found [here](https://docs.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection?view=win10-ps).*

- The Azure subscription, into which the on-prem gateway has been registered, is so brand new that the *resource provider* '**Microsoft.Logic**' hasn't been registered yet.  
Seems odd that this is required? Well, not really!  
When creating a new ApiConnection, what happens in the back is that the runtime is going to look for the connector-specific managed API within the subscription where the gateway has been registered - *ok, this might be a bit odd* - in order to link the new ApiConnection to the selected on-premises data gateway.  
If, at that time, the Microsoft.Logic resource provider hasn't been registered yet, you won't be able to use any of the Logic Apps-related services, which includes the connectors and their API's.


