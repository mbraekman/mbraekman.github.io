---
layout: post
author: Maxim Braekman
title: "Azure API Management: Consumption Tier"
description: "For those of you looking for a less costly, entry-level version of Azure API Management, the time has come to meet: Azure's APIM Consumption Tier!"
image: ./img/azure-apim-consumption-tier.jpg
tags: [API Management]
---
Surely, whatever type of development you’re involved with, no matter the language or application type, you’re bound to get your hands dirty with some sort of API sooner or later.  
The last couple of years, the API ecosystem started booming, requiring every type of application to expose an API which has been opened to the general public or just to those happy few who have been granted permission to access it. While there is already enough of complexity when just having to manage the entire life cycle of a single API, one can imagine that doing this for multiple services exponentially increases complexity.

Since early 2014 Microsoft offers a cloud-based solution to reduce all this overhead by providing a solution to manage the overall life-cycle, from documentation and publishing tools to an API Gateway that allows access to any backend services, enforcing throttling, security policies, … Through the years, more and more functionalities got added to **Azure API Management** ending up in what we have today(-1): 4 [fixed-price](https://azure.microsoft.com/en-us/pricing/details/api-management/) tiers, offering everything from a limited to a full range of features.

While these tiers contain everything one could possibly need, for smaller applications or services, the fixed price contract appeared to be daunting to some as they would never reach the given consumption-limits, nor would they need all the available features. Because of this, you’ll probably never run into any project team stating they’ll start off by using API Management, just because it would simplify the life of their developers. Let’s face it, most project managers are all about cutting costs, definitely for those small-budget projects or within smaller companies.

Starting today, that mindset will be changed, because for those looking for an entry-level model of API Management, say hello to Azure API Management: Consumption Tier.

## Introducing Azure APIM Consumption Tier
As of today, the consumption tier of Azure API Management is in **public preview** ([see the official announcement here](https://azure.microsoft.com/en-us/blog/announcing-azure-api-management-for-serverless-architectures/)), ready to address the needs of customers looking into publishing microservices-based applications, implemented on Logic Apps, API Apps or various other offerings or to expose facades for serverless Azure services such as Service Bus, Storage, Event Hub, Cosmos DB and more.

This tier is offering a full set of properties commonly associated with serverless computing, such as:

- **No infrastructure** – don’t even think about the provisioning or management of any servers, Microsoft will take care of your gateway and the required capacity without having to spin up a separate server just for you. While you didn’t need to think about any of this for the other tiers either, they did require the creation of a specific set of infra-related artifacts, considerably increasing the amount of time needed to create/upscale an APIM service.
- **Auto-scaling** – Based on the load of your services, capacity will automatically be increased or decreased depending on the needs. No more worries about having to properly monitor the load on your services to ensure it can be scaled up in time.
- **Micro-billing** – Due to the auto-scaling, you’re no longer paying for any unused capacity, instead you will only be paying-per-call. Of course, when you have millions of requests coming in each month, the previous statement about this tier being ‘less costly’ can seem to be quite wrong, however it remains a fair pricing model, since at the time of writing, the estimated pricing (*) would be $0.035 per 10K requests, with the first 1 million free of charge.
- **Built-in Availability** – Even though you might be paying less (*depending on the load*), there’s no excuse for your services to go offline for a long period of time, therefore this tier will be offering a triple 9 SLA-level.

	![API Tier](../../../../img/posts/azure-apim-consumption-tier/API-Tier-blog.jpg)

	_(*) For an accurate pricing-overview, we advise you to have a look at the [pricing-page](https://aka.ms/apimpricing)._

The overall reason of existence for this tier is really to be used closely together with, and be an extension of, Azure Functions & Logic Apps. Taking Azure Function Proxies for example, while this is a decent way of manipulating the URI and exposed operations of a service or Logic App, it doesn’t really offer any of the APIM-capabilities, such as throttling, caching, versioning, … As the pricing threshold was a big reason for people to make do with proxies, this consumption tier will remove that threshold, exposing an entire APIM-tool at a low cost.

## Consumption Tier Limitations
Of course, this tier does have some limitations compared to the other tiers, as it’s considered to be a *lighter*-weight version of API Management, but don’t let that scare you off, as you’ll notice you can perfectly make do without several of the below mentioned features. If this is not the case, then you’ll be better off looking at 1 of the full-blown dedicated tiers.

Below you can find an initial overview of these limitations, however a full comparison can be found [here](https://docs.microsoft.com/en-us/azure/api-management/api-management-features).

_The information provided below was available to us at the time of writing, but these lists of feature and usage limits are not exhaustive and are subject to change._

## Functional limitations
- **No developer portal** – while this is included in the other tiers, developers or publishers will not be able to use this portal to provide documentation about the exposed API’s. This also excludes the usage of users, groups, issues, email templates and notifications.
- **No built-in cache** – as you don’t have any specifically assigned infrastructure, there is no place to store any caching, however this tier **does support** the usage of **external Redis cache** (**). This will require the creation of a Redis cache database, however it does overcome some of the limitations of the built-in cache:
  - Your cache is not being cleared periodically because of APIM updates
  - You have more control over cache configuration
  - Allows you to cache more data than any of the built-in cache-features in the other tiers

	<img src="../../../../img/posts/azure-apim-consumption-tier/redis.png" width="295">

	_(**) will also come to the other tiers soon._

 

- **No built-in analytics** – you will not be able to get a detailed overview of all requests, based on the geographical origins, what users have been making calls through which product/subscription or what specific operations have been called. If this is a requirement for your services, you will have to look at the other tiers. Even though you will not have this type of analytics, you’re still able to consult Application Insights for monitoring your gateway.
- **No VNET support** – no, this tier does not provide support for Virtual Networks, if this would be something required for your project, you’d be obliged to go for the Premium Tier.

	![No VNet support](../../../../img/posts/azure-apim-consumption-tier/no-Vnet.png)

- **Serverless Cold-start** – as capacity is only getting assigned if the gateway is being used, you can experience cold-start delays if none of your APIs within a specific gateway have been called.
- **No Git configuration** – the possibility to store the APIM service configuration into Git, providing configuration versioning, bulk configuration changes… will not be available for this tier.

	![No Git Configuration](../../../../img/posts/azure-apim-consumption-tier/no-git.png)

- **No backup/restore** – this tier does not support disaster recovery strategies, meaning if the region that hosts your gateway service, goes offline, so does your gateway.
- **No direct management API** – there is no management service on top of this APIM-tier. Because of this, you are not able to perform operations on products and subscriptions, … However, you are still able to perform any type of configuration change, by using the Azure portal or by making use of the ARM REST-API’s.
- **Client certificate authentication** – while this is, along with basic authentication, one of the most commonly-used security implementations, this is not available yet in the public preview version of this tier. But rest assured, it will be added before it’s GA – with 1 caveat: every API within the same gateway, will need to have client certificate authentication enabled, meaning you’ll have to create a separate gateway if you want other API’s to have a different security-implementation. But since the pay-per-call model, this definitely is not a show-stopper.
- **SSL Settings** – even though most applications should’ve been upgraded to make use of the latest version of SSL/TLS, there is still a wide range of companies out there, using an old less-secure version. The existing tiers offer the capability to specify what versions of these protocols are allowed to access the gateway. Within the consumption tier however, this is not possible.
- **Subscribing to an API** – before, a user needed to subscribe to a specific product in order to get access to all API’s that got assigned to this product. Within this tier, it will become possible to create subscriptions on service-, API- or product-level, providing a more fine-grained access level.


## Usage limitations
As is the case with any technology, you’ll need to properly look into the requirements of your specific project and that’s no different with the APIM Consumption tier. Next to the mentioned functional limitations, this tier will also have some quotas which you will need to consider:

- Max number of gateways per subscription: 5
- Max number of subscriptions per gateway: 500
- Max number of APIs per gateway: 50
- Max number of API operations per gateway: 1000
- Total request duration: max 30 seconds
- Max buffered payload size: 2MB
- Max policy length: 4KB
- Max total policy execution time per request: 3 seconds

Also, keep in mind that this public preview might only be available in a select number of regions for the time being. At the time of writing, this would be limited to: West US, North Central US, West Europe, North Europe, South East Asia, Australia East.

## Conclusion
For those of you who have always been put off by the price of the previously available tiers, the time has come to embrace the idea of using Azure API Management within your projects.

With the creation of this new Consumption Tier, Microsoft opened the door for scores of people or companies who are getting involved in Logic App/Azure Function/API App development and want to expose their services, without requiring a full-fledged APIM service that offers the above mentioned features.

Of course, this tier will not be suitable to everyone, but based on your specific needs, at least now you have the possibility to get started with this entry-level/less-pricey solution. So, give this one a try and you’ll notice how relieved your developers are going to be, by no longer having to implement and support several security protocols within their services.

Thanks for reading and happy consuming!