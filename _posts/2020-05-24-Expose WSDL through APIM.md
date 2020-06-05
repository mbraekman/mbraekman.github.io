---
layout: post
author: Maxim Braekman
title: Expose a WSDL through APIM
description: Want to find out how you can expose a WSDL through API Mangement?
image: ./img/apim-expose-wsdl.jpg
tags: [API Management, SOAP]
caption: "API Management"
---
## Expose WSDL through SOAP APIM endpoint

Creating a new SOAP-based API within API Management is easy, just import the WSDL and you're done. But this API lacks an endpoint by which the actual WSDL is being exposed, as would be the case on a traditional SOAP-service.  
Luckily, there is an easy way to achieve this, so you don't have to share the WSDL in any other way.

## Steps
- [Overview](#overview)
- [Provision the required storage account](#provision-the-required-storage-account)
- [Create a new operation](#create-a-new-operation)
- [Alternative setup](#alternative-setup)
- [Set the policy](#set-the-policy)

### Overview
Before we get started, let's give an overview of how we can expose the WSDL through an API-endpoint and how everything will be connected.

Since a diagram gives a better overview than a whole bunch of text, have a look at the diagram added underneath and consider the fact that in this case we are going to expose a WSDL, stored as a blob inside a storage account, without requiring any custom code on our backend.

[![High level overview](../../../../img/posts/azure-apim-wsdl-endpoint/high-level-overview.png)  ](../../../../img/posts/azure-apim-wsdl-endpoint/high-level-overview.png) 

### Provision the required storage account
First, you need some place to store the actual WSDL-file and while you could use many different services to do so, a Blob container within a Storage Account does seem like to cheapest/best option.  

For this purpose, an existing Storage Account can be re-used, by adding a new Blob container, specifically for this purpose.  As this container will only contain WSDL-files *(hopefully as less as possible)*, which should be made publicly available to whoever needs it, we can set the **public access level** of this container to '*Public read access for blobs only*'.  
While this does allow anonymous requests to access the blobs within the container - which isn't very secure - for this purpose that is OK, since you want the WSDL to be publicly available any way.

![Set public access level](../../../../img/posts/azure-apim-wsdl-endpoint/set-public-access.png) 

### Create a new operation
Assuming you've already created an API Management instance and added an API based on the prepared WSDL, it is time to extend the SOAP API with a WSDL-endpoint.  

Add a new **GET**-operation to the API and set the display name to something such as 'Expose WSDL'.  
However, since this is a SOAP API, we are forced to set a specific soapAction as part of the URL.  

![Set soapAction](../../../../img/posts/azure-apim-wsdl-endpoint/apim-specify-soapaction.png)

In this case, we can simply set the URL to **/?soapAction=wsdl**.  
While, this doesn't match the exact structure of a regular SOAP API, in which you can access the WSDL via '*?wsdl*' or '*?singleWsdl*', it's coming close.

![Create new operation](../../../../img/posts/azure-apim-wsdl-endpoint/apim-create-new-operation.png)

As part of the communication with your client, you can simply state the WSDL can be accessed through the following URL:  
https://mbraekman-dev-we-apim.azure-api.net/api/soap/?soapAction=wsdl 

### Alternative setup
Don't want to tell people they have to browse to a specific URL containing '*/?soapAction=wsdl*'?  
No problem, there is an alternative way of achieving the default SOAP-behavior regarding exposing a WSDL-file.  

Instead of adding a new operation to the existing SOAP API within API Management, you can also choose a add a new blank API instead.   

*Note: This API can be re-used to expose all WSDL's for any of the SOAP API's you created.*   

When creating this WSDL API, make sure to leave the *API URL Suffix empty*, so can you specify this suffix from within every single operation to match the URL of the API for which the WSDL is to be exposed.  


[![Create WSDL API](../../../../img/posts/azure-apim-wsdl-endpoint/wsdl-api.png)  ](../../../../img/posts/azure-apim-wsdl-endpoint/wsdl-api.png) 

Inside this new API, add a **GET**-operation and make sure to use the display name property to refer to the corresponding API, but also pay attention to set the URL of this operation to match the base-URL of you SOAP API, followed by the suffix '*?wsdl*'.  

![Create new WSDL Operation](../../../../img/posts/azure-apim-wsdl-endpoint/wsdl-api-new-operation.png)

In turn, this means your WSDL will be exposed through an URL, matching the traditional way of exposing a WSDL on your SOAP API, even though this means having an additional API set up in API Management.  
When finished, the URL of this operation will look like this:  
https://mbraekman-dev-we-apim.azure-api.net/api/soap?wsdl 

### Set the policy
Based on how you would like to have the WSDL exposed you could have gone for either of the 2 options explained above and while this creates the required endpoint, this doesn't expose the WSDL itself.  

To return the WSDL, which has been stored in a blob storage container, we can make use of the APIM policies to do the work for us - *no need to create a specific function in your backend for this*.  

What exactly this policy should be doing, is actually quite simple:
- use the **send-request** policy the retrieve the publicly accessible WSDL from our blob storage and store it inside of a variable named '*wsdlContent*'.
- use the **return-response** policy to build the response and insert the content of the WSDL from the previous policy as the body.  
Something which you need to think about as well, unless you adjust this in the WSDL itself, is to update the service-URL stored inside the WSDL itself to point towards your API Management endpoint instead of the backend API.

Below you can see how such a policy would look like.  

{% highlight xml %}
<policies>
    <inbound>
        <!-- retrieve the content from the publicly accessible blob -->
        <send-request mode="new" response-variable-name="wsdlContent" timeout="30" ignore-error="true">
            <set-url>https://mbraekman-dev-we-kbstorage.blob.core.windows.net/wsdl/kb-soap-service-wsdl.wsdl</set-url>
            <set-method>GET</set-method>
        </send-request>
    </inbound>
    <backend>
        <!-- instead of forwarding the call to any back-end API, return the content of the WSDL instead. -->
        <return-response>
            <set-status code="200" reason="OK" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/xml</value>
            </set-header>
            <!-- make sure to replace the default service URL present in the WSDL to point towards the correct/environment-specific APIM endpoint. -->
            <set-body>@{
                string wsdlContent = ((IResponse)context.Variables["wsdlContent"]).Body.As<string>();
                string backendUrl = "https://mbr-api-app.azurewebsites.net/api/v1/soap";
                backendUrl = backendUrl.Replace("&amp;","&amp;amp;");
                wsdlContent = wsdlContent.Replace(backendUrl,"https://mbraekman-dev-we-apim.azure-api.net/soap/endpoint/");
                return wsdlContent.ToString();
            }</set-body>
        </return-response>
    </backend>
    <outbound />
    <on-error>
        <base />
    </on-error>
</policies>
{% endhighlight %}

### Conclusion
While API Management is more focused on exposing REST-based API's, you can still expose SOAP-services but you will need to add an additional endpoint - *or a separate API* - specifically for exposing the WSDL to anyone who would be interested.
