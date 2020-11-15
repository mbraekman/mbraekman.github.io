---
layout: post
author: Maxim Braekman
title: Generate a SAS-token from within an APIM policy
description: Allow users to request access to a specific file on an Azure Storage Account, while staying in full control.
image: ./img/apim-generate-sas-token.jpg
tags: [API Management, Security]
---

When you want to expose files stored on an Azure Storage Account, you typically wouldn't want to open up your entire container to the public, while granting access to specific users.  

One of the options is to generate a _Secure Access Signature_ (SAS) which grants access to a specific blob for a certain amount of time. But, you don't want to be generating these yourselves. So, why not use Azure API Management to host - _and secure_ - an API which can generate these tokens upon request.   

## Steps
- [Overview](#overview)
- [Create a new API](#create-a-new-api)
- [Create a new operation](#create-a-new-operation)
- [Set the policy](#set-the-policy)
- [Conclusion](#conclusion)
- [The full policy](#full-policy)


### Overview
Let's start off with a high-level overview of how such a setup would look like and how this affects the flow of requests/responses.

[![High level overview](../../../../img/posts/azure-apim-generate-sas-token/high-level-overview.png)  ](../../../../img/posts/azure-apim-generate-sas-token/high-level-overview.png) 

As you can see, by using API Management you will be in full control of how clients have to authenticate and/or authorize, to ensure SAS-tokens are being granted to those who absolutely deserve it. While this requires some additional setup from your end, no-one will need to bother you anymore for generating a new SAS-token to access one of your files.  
_Automate all the things!_

### Create a new API
As you can see in the overview, the API that is going to be used in this setup does not have any _real_ backend. This means all of the magic will be done from within the policies, without requiring you to manage an additional app service/container/...

In order to give you a complete overview of how this type of API can be created, we'll go step-by-step through the entire process.

Because of this, we've created a new *blank API* within our API Management instance, specifically to host this expose/storage-endpoint.

![Blank API](../../../../img/posts/azure-apim-generate-sas-token/apim-blank-api-no-operations.png)

### Create a new operation
Now we have an API, it's time to add the _/generate/sas/{containerName}/{blobName}_-endpoint, as this specific endpoint will grant you access to a specific file/blob.

![Create a new endpoint](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint.png)

Since the client will need to be able to provide the name of the blob and the name of the container in which the blob is located, the following template-parameters have to be added.  

![Endpoint query parameters](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint-template-parameters.png)  

While we’re at it, let’s also set the possible response-codes along with a sample response:
- *200 OK*  
![Return SAS token OK](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint-response-200.png)
- *401 Unauthorized*  
![Return SAS token unauthorized](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint-response-401.png)

Once saved, the new endpoint has been created and is ready to be consumed, however as long as we haven't specified any policies, nothing usefull will be returned.  
[![Set new endpoint request info](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint-completed.png)](../../../../img/posts/azure-apim-generate-sas-token/create-new-generate-blob-sas-endpoint-completed.png)  

### Set the policy

Now that the _infrastructure_ has been set, it's time to dive into the policy.  
As you will notice, we're using some inline C# here and there to create the signatures, which make up part of the SAS-token, as well as to build up the SAS-token and URL to be used by the client.

Before you go to the policy-editor, we will need to store the following values in a _Named Value_ within API Management:
- _KB-Storage-Account:_  
    The name of the storage account where the blobs are located.
- _KB-Storage-Account-AccessKey:_  
    An accesskey to be used within the creation of the signature.

Let's go through each of the steps that are going to be performed by the policy, in order to make it clear what everything is doing.
- Extract the parameters from the request:
    {% raw %}
    ```xml
    <!-- extract parameters from URL -->
    <set-variable name="containerName" value="@(context.Request.MatchedParameters["containerName"])" />
    <set-variable name="blobName" value="@(context.Request.MatchedParameters["blobName"])" />
    ```  
    {% endraw %}

- Next, we need to initialize a set of variables that will be used for the creation of the SAS-token.  
    You will notice that next to the expiration time (in seconds) we will also set the protocol to be used, the version of the storage account API, as well as the full name of the blob.  
    Some properties, such as signedIp, signedIdentifier, rscc _(Cache-Control)_, rscd _(Content-Disposition)_, rsce _(Content-Encoding)_, rscl _(Content-Language)_ and rsct _(Content-Type)_ will remain empty in our example but could easily be provided for different scenarios.  
    More details on these properties can be found [here](https://docs.microsoft.com/en-us/rest/api/storageservices/create-service-sas#construct-a-service-sas).
    {% raw %}
    ```xml
    <!-- Initialize context variables with property values. -->
    <set-variable name="accessKey" value="{{KB-Storage-Account-AccessKey}}" />
    <set-variable name="expiryInSeconds" value="3600" />
    <set-variable name="storageAccount" value="{{KB-Storage-Account}}" />
    <set-variable name="x-ms-date" value="@(DateTime.UtcNow)" />
    <set-variable name="signedPermissions" value="r" />
    <set-variable name="signedService" value="b" />
    <set-variable name="signedStart" value="@(((DateTime)context.Variables["x-ms-date"]).ToString("yyyy-MM-ddTHH:mm:ssZ"))" />
    <set-variable name="signedExpiry" value="@(((DateTime)context.Variables["x-ms-date"]).AddSeconds(Convert.ToInt32((string)context.Variables["expiryInSeconds"])).ToString("yyyy-MM-ddTHH:mm:ssZ"))" />
    <set-variable name="signedProtocol" value="https" />
    <set-variable name="signedVersion" value="2017-07-29" />
    <set-variable name="canonicalizedResource" value="@{
        return string.Format("/blob/{0}/{1}/{2}",
            (string)context.Variables["storageAccount"],
            (string)context.Variables["containerName"],
            (string)context.Variables["blobName"]
        );
    }" />
    <set-variable name="signedIP" value="" />
    <set-variable name="signedIdentifier" value="" />
    <set-variable name="rscc" value="" />
    <set-variable name="rscd" value="" />
    <set-variable name="rsce" value="" />
    <set-variable name="rscl" value="" />
    <set-variable name="rsct" value="" />
    ```  
    {% endraw %}

- Combine all of the earlier defined properties into a string that will be used to create the signature.  
    {% raw %}
    ```xml
    <!-- Build the string to form signature. -->
    <set-variable name="StringToSign" value="@{
        return string.Format("{0}\n{1}\n{2}\n{3}\n{4}\n{5}\n{6}\n{7}\n{8}\n{9}\n{10}\n{11}\n{12}",
            (string)context.Variables["signedPermissions"],
            (string)context.Variables["signedStart"],
            (string)context.Variables["signedExpiry"],
            (string)context.Variables["canonicalizedResource"],
            (string)context.Variables["signedIdentifier"],
            (string)context.Variables["signedIP"],
            (string)context.Variables["signedProtocol"],
            (string)context.Variables["signedVersion"],
            (string)context.Variables["rscc"],
            (string)context.Variables["rscd"],
            (string)context.Variables["rsce"],
            (string)context.Variables["rscl"],
            (string)context.Variables["rsct"]
        );
    }" />
    ```  
    {% endraw %}

- Now, we need to hash this string to obtain the actual signature.
    {% raw %}
    ```xml
    <!-- Build/Hash the signature. -->
    <set-variable name="Signature" value="@{
        byte[] SignatureBytes = System.Text.Encoding.UTF8.GetBytes((string)context.Variables["StringToSign"]);
        System.Security.Cryptography.HMACSHA256 hasher = new System.Security.Cryptography.HMACSHA256(Convert.FromBase64String((string)context.Variables["accessKey"]));
        return Convert.ToBase64String(hasher.ComputeHash(SignatureBytes));
    }" />
    ```  
    {% endraw %}

- The signature has been created, so now we need to build up the entire SAS-token.
    {% raw %}
    ```xml
    <!-- Form the sasToken. -->
    <set-variable name="SasToken" value="@{
        return string.Format("sv={0}&sr={1}&sp={2}&st={3}&se={4}&spr={5}&sig={6}",
            (string)context.Variables["signedVersion"],
            (string)context.Variables["signedService"],
            (string)context.Variables["signedPermissions"],
            System.Net.WebUtility.HtmlEncode((string)context.Variables["signedStart"]),
            System.Net.WebUtility.HtmlEncode((string)context.Variables["signedExpiry"]),
            (string)context.Variables["signedProtocol"],
            System.Net.WebUtility.HtmlEncode((string)context.Variables["Signature"])
        );
    }" />
    ```  
    {% endraw %}

- Before returning the SAS-token in itself, let's build up the full URL so the client can immediately use this to download the file.  
    {% raw %}
    ```xml
    <!-- Form the complete Url. -->
    <set-variable name="FullUrl" value="@{
        return string.Format("https://{0}.blob.core.windows.net/{1}/{2}?{3}",
            (string)context.Variables["storageAccount"],
            (string)context.Variables["containerName"],
            (string)context.Variables["blobName"],
            (string)context.Variables["SasToken"]
        ).Replace("+", "%2B");
    }" />
    ```  
    {% endraw %}

- But wait!  
    We've just generated a SAS-token, without verifying the existence of the file.  
    Let's attempt to retrieve the metadata of the file, to verify it really exists.

    {% raw %}
    ```xml
    <set-variable name="metadataUrl" value="@{
        return string.Format("https://{0}.blob.core.windows.net/{1}/{2}?comp=metadata&{3}",
            (string)context.Variables["storageAccount"],
            (string)context.Variables["containerName"],
            (string)context.Variables["blobName"],
            (string)context.Variables["SasToken"]
        ).Replace("+", "%2B");
    }" />
    
    <!-- Check existance of the blob before returning 200 OK with sas token. -->
    <send-request mode="new" response-variable-name="blobMetadata" timeout="30" ignore-error="false">
        <set-url>@((string)context.Variables["metadataUrl"])</set-url>
        <set-method>GET</set-method>
    </send-request>
    ```  
    {% endraw %}

- Based on the response of the metadata-call, return the SAS-token or return a "NotFound"-exception.  
 
    {% raw %}
    ```xml
    <choose>
        <!-- Check active property in response -->
        <when condition="@(((IResponse)context.Variables["blobMetadata"]).StatusCode == 404)">
            <!-- Return 401 Unauthorized with http-problem payload -->
            <return-response>
                <set-status code="404" reason="Not Found" />
                <set-header name="Content-Type" exists-action="override">
                    <value>application/json</value>
                </set-header>
                <set-body>@{
                    return new JObject(
                        new JProperty("error", "not_found"),
                        new JProperty("error_description", "No data could be found for the given parameters."),
                        new JProperty("timestamp", DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ssZ"))
                    ).ToString();
                }</set-body>
            </return-response>
        </when>
        <otherwise>
            <!-- return response -->
            <return-response>
                <set-status code="200" reason="OK" />
                <set-header name="Content-Type" exists-action="override">
                    <value>application/json</value>
                </set-header>
                <set-body>@{
                    var fullUrl = ((string)context.Variables["FullUrl"]);
                    var remainingValidDuration = ((Convert.ToDateTime((string)context.Variables["signedExpiry"])) - DateTime.Now).TotalSeconds;
                    
                    return new JObject(
                        new JProperty("url", fullUrl),
                        new JProperty("expiresIn", Math.Floor(remainingValidDuration).ToString()),
                        new JProperty("timestamp", DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ssZ"))
                    ).ToString();
                }</set-body>
            </return-response>
        </otherwise>
    </choose>
    ```  
    {% endraw %}

If you want to see the entire policy, instead of the snippets, scroll down or click [here](#full-policy).

### Try it out!
Once everything has been saved, we can use Postman to build the request and retrieve a token.  
If we first provide a wrong file-name, we should now receive a "404 NotFound"-exception.  

[![Postman - Not Found](../../../../img/posts/azure-apim-generate-sas-token/postman-unexisting-file.png)](../../../../img/posts/azure-apim-generate-sas-token/postman-unexisting-file.png)  

If we now provide a valid file-name, we should receive a "200 OK"-response, along with the URL grant us access to the file.  

[![Postman - Not Found](../../../../img/posts/azure-apim-generate-sas-token/postman-existing-blob.png)](../../../../img/posts/azure-apim-generate-sas-token/postman-existing-blob.png)  


### Conclusion
By making use of API Management and the policy-capabilities it is possible to create an API that allows clients to request SAS-tokens on the fly.  
Next to saving you some time as you don't have to generate these tokens yourself, you will also get additional logging within Application Insights, allowing you to see whom is accessing what. 



#### Full Policy
The full policy will look like this:  

{% raw %}
```xml
<policies>
    <inbound>
        <base />
        <!-- extract parameters from URL -->
        <set-variable name="containerName" value="@(context.Request.MatchedParameters["containerName"])" />
        <set-variable name="blobName" value="@(context.Request.MatchedParameters["blobName"])" />
        <!-- create SAS-token to access data on blob storage -->
        <!-- Initialize context variables with property values. -->
        <set-variable name="accessKey" value="{{KB-Storage-Account-AccessKey}}" />
        <set-variable name="expiryInSeconds" value="3600" />
        <set-variable name="storageAccount" value="{{KB-Storage-Account}}" />
        <set-variable name="x-ms-date" value="@(DateTime.UtcNow)" />
        <set-variable name="signedPermissions" value="r" />
        <set-variable name="signedService" value="b" />
        <set-variable name="signedStart" value="@(((DateTime)context.Variables["x-ms-date"]).ToString("yyyy-MM-ddTHH:mm:ssZ"))" />
        <set-variable name="signedExpiry" value="@(((DateTime)context.Variables["x-ms-date"]).AddSeconds(Convert.ToInt32((string)context.Variables["expiryInSeconds"])).ToString("yyyy-MM-ddTHH:mm:ssZ"))" />
        <set-variable name="signedProtocol" value="https" />
        <set-variable name="signedVersion" value="2017-07-29" />
        <set-variable name="canonicalizedResource" value="@{
            return string.Format("/blob/{0}/{1}/{2}",
                (string)context.Variables["storageAccount"],
                (string)context.Variables["containerName"],
                (string)context.Variables["blobName"]
            );
        }" />
        <set-variable name="signedIP" value="" />
        <set-variable name="signedIdentifier" value="" />
        <set-variable name="rscc" value="" />
        <set-variable name="rscd" value="" />
        <set-variable name="rsce" value="" />
        <set-variable name="rscl" value="" />
        <set-variable name="rsct" value="" />
        
        <!-- Build the string to form signature. -->
        <set-variable name="StringToSign" value="@{
            return string.Format("{0}\n{1}\n{2}\n{3}\n{4}\n{5}\n{6}\n{7}\n{8}\n{9}\n{10}\n{11}\n{12}",
                (string)context.Variables["signedPermissions"],
                (string)context.Variables["signedStart"],
                (string)context.Variables["signedExpiry"],
                (string)context.Variables["canonicalizedResource"],
                (string)context.Variables["signedIdentifier"],
                (string)context.Variables["signedIP"],
                (string)context.Variables["signedProtocol"],
                (string)context.Variables["signedVersion"],
                (string)context.Variables["rscc"],
                (string)context.Variables["rscd"],
                (string)context.Variables["rsce"],
                (string)context.Variables["rscl"],
                (string)context.Variables["rsct"]
            );
        }" />
        
        <!-- Build/Hash the signature. -->
        <set-variable name="Signature" value="@{
            byte[] SignatureBytes = System.Text.Encoding.UTF8.GetBytes((string)context.Variables["StringToSign"]);
            System.Security.Cryptography.HMACSHA256 hasher = new System.Security.Cryptography.HMACSHA256(Convert.FromBase64String((string)context.Variables["accessKey"]));
            return Convert.ToBase64String(hasher.ComputeHash(SignatureBytes));
        }" />

        <!-- Form the sasToken. -->
        <set-variable name="SasToken" value="@{
            return string.Format("sv={0}&sr={1}&sp={2}&st={3}&se={4}&spr={5}&sig={6}",
                (string)context.Variables["signedVersion"],
                (string)context.Variables["signedService"],
                (string)context.Variables["signedPermissions"],
                System.Net.WebUtility.HtmlEncode((string)context.Variables["signedStart"]),
                System.Net.WebUtility.HtmlEncode((string)context.Variables["signedExpiry"]),
                (string)context.Variables["signedProtocol"],
                System.Net.WebUtility.HtmlEncode((string)context.Variables["Signature"])
            );
        }" />

        <!-- Form the complete Url. -->
        <set-variable name="FullUrl" value="@{
            return string.Format("https://{0}.blob.core.windows.net/{1}/{2}?{3}",
                (string)context.Variables["storageAccount"],
                (string)context.Variables["containerName"],
                (string)context.Variables["blobName"],
                (string)context.Variables["SasToken"]
            ).Replace("+", "%2B");
        }" />
        <set-variable name="metadataUrl" value="@{
            return string.Format("https://{0}.blob.core.windows.net/{1}/{2}?comp=metadata&{3}",
                (string)context.Variables["storageAccount"],
                (string)context.Variables["containerName"],
                (string)context.Variables["blobName"],
                (string)context.Variables["SasToken"]
            ).Replace("+", "%2B");
        }" />
        
        <!-- Check existance of the blob before returning 200 OK with sas token. -->
        <send-request mode="new" response-variable-name="blobMetadata" timeout="30" ignore-error="false">
            <set-url>@((string)context.Variables["metadataUrl"])</set-url>
            <set-method>GET</set-method>
        </send-request>
        
        <choose>
            <!-- Check active property in response -->
            <when condition="@(((IResponse)context.Variables["blobMetadata"]).StatusCode == 404)">
                <!-- Return 401 Unauthorized with http-problem payload -->
                <return-response>
                    <set-status code="404" reason="Not Found" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>@{
                        return new JObject(
                                new JProperty("error", "not_found"),
                                new JProperty("error_description", "No data could be found for the given parameters."),
                                new JProperty("timestamp", DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ssZ"))
                            ).ToString();
                    }</set-body>
                </return-response>
            </when>
            <otherwise>
                <!-- return response -->
                <return-response>
                    <set-status code="200" reason="OK" />
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/json</value>
                    </set-header>
                    <set-body>@{
                        var fullUrl = ((string)context.Variables["FullUrl"]);
                        var remainingValidDuration = ((Convert.ToDateTime((string)context.Variables["signedExpiry"])) - DateTime.Now).TotalSeconds;
                        
                        return new JObject(
                                new JProperty("url", fullUrl),
                                new JProperty("expiresIn", Math.Floor(remainingValidDuration).ToString()),
                                new JProperty("timestamp", DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ssZ"))
                            ).ToString();
                    }</set-body>
                </return-response>
            </otherwise>
        </choose>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```  
{% endraw %}
 