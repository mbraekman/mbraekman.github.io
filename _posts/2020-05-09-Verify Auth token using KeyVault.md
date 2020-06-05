---
layout: post
author: Maxim Braekman
title: Verify auth-token by using Key Vault
description: How can you use Key Vault to validate auth-tokens within API Management?
image: ./img/apim-verify-keyvault-token.jpg
tags: [API Management, Azure Key Vault]
---


The verification-process of API-key's sent along in every client-request in order to authenticate themselves against your API does require APIM to know what values to expect, hence these should be stored somewhere in your Azure subscription as well.  
In order to prevent having to add these values as plain-text *(argh.. my eyes!!)* in your APIM-policy, you can make use of KeyVault instead. 

## Requirements
- [KeyVault secret](#create-a-keyvault-secret)
- [APIM Policy: keyVault lookup](#apim-policy-keyvault-lookup)
- [APIM Policy: implement caching](#apim-policy-implement-caching)
- [APIM Policy: comparing values](#apim-policy-comparing-values)

### Create a KeyVault-secret
The first requirement, of course, is the creation of a secret within KeyVault to store the actual value/API-key.

![Create KeyVault Secret](../../../../img/posts/azure-apim-keyvault-secrets/create-a-keyvault-secret.png)

When creating a secret, think about whether you want to set an expiration date in order to force a new key to be used every x amount of time. By doing so, you can prevent the api-key to remain useable for too long, if it falls into the wrong hands.  

> ***Hint**: when setting an expiration date, make sure to use event grid to subscribe to the 'Microsoft.KeyVault.SecretNearExpiry'-event, which will be used to notify you whenever a secret is about to expire.*

### APIM Policy: KeyVault lookup
Since, not storing the actual API-key in plain-text within the policy-definition should not only be best practice but also simple common sense, you could either make use of APIM Named Values - to store secrets on APIM-instance level - or store the secrets within KeyVault.  
In order to be able to keep track of all secrets, it would be best to store them in a single repository, being KeyVault - *although even when using KeyVaults it is advised to create multiple instances.*.  

As many Azure services, each KeyVault-instance is accessible through an API, allowing you to manage keys, secrets or certificates, which is exactly what will be leveraged from within an APIM-policy to retrieve the value of a secret stored in KeyVault.  
Luckily, managed identity can be used in this case to ensure our APIM-instance can authenticate against KeyVault if the following requirements are met:  
- Enable managed identity on the APIM-instance.  
No need to create the required AD-artifacts yourself, just go for the '*System assigned*'-option.   
![APIM Managed Identity](../../../../img/posts/azure-apim-keyvault-secrets/apim-managed-identity.png)  

- Create an access policy in KeyVault to allow APIM to read *(List|Get)* secrets.  
![KeyVault access policy](../../../../img/posts/azure-apim-keyvault-secrets/keyvault-access-policy-apim.png)  

Once APIM is allowed to use managed identity to access KeyVault-secrets, it is time to start building the policy of which the first step would be to retrieve the actual secret-value.  
This can be done using the '**[send-request](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#SendRequest)**'-policy, in which a GET-operation will be performed against the URL pointing to the *latest version* of the secret.

```xml
<!-- Use Managed Identity to authenticate against KeyVault and retrieve the secret-value -->
<send-request mode="new" response-variable-name="auth-api" timeout="30" ignore-error="true">
	<set-url>https://mbr-apim-kv.vault.azure.net/secrets/api-key-secret/?api-version=7.0</set-url>
	<set-method>GET</set-method>
	<authentication-managed-identity resource="https://vault.azure.net" />
</send-request>
```

The retrieved value is then going to be stored inside a variable named 'auth-api'.

### APIM Policy: Implement caching
Since KeyVault is [limited](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#key-vault-limits) to 2000 operations per 10 seconds - *or 200 per second* - on secrets, managed storage account keys, and vault transactions per vault per region you don't want every API-call to actually perform a lookup against KeyVault for you secret, especially not when your API should be build to easily scale.

In order to prevent hitting the KeyVault-throttling limitations, you can make use of the **[cache-store-value](https://docs.microsoft.com/en-us/azure/api-management/api-management-caching-policies)**-policy to actually store a copy of the secret in-memory for a given period of time, before contacting KeyVault again to retriev the latest version of the secret.

Before performing the GET-operation and storing any data, you need to check the current cached data to ensure whether or not the value is already known and still valid. This can be done by using the **[cache-lookup-value](https://docs.microsoft.com/en-us/azure/api-management/api-management-caching-policies)**-policy.  

Appending these caching-policies to the aforementioned lookup-policy, would result into the following section:

{% highlight xml %}
<!-- Get secret from cache -->
<cache-lookup-value key="authentication-token" default-value="noToken" variable-name="authentication-token" />
<choose>
	<!-- Check if secret was found -->
	<when condition="@((string)context.Variables["authentication-token"] == "noToken")">
		<!-- Secret was not found in cache, retrieve secret from Key Vault -->
		<send-request mode="new" response-variable-name="auth-api" timeout="30" ignore-error="true">
			<set-url>https://mbr-apim-kv.vault.azure.net/secrets/api-key-secret/?api-version=7.0</set-url>
			<set-method>GET</set-method>
			<authentication-managed-identity resource="https://vault.azure.net" />
		</send-request>
		<set-variable name="authentication-token" value="@((string)((IResponse)context.Variables["auth-api"]).Body.As<JObject>()["value"])" />
		<cache-store-value key="authentication-token" value="@((string)context.Variables["authentication-token"])" duration="300" />
	</when>
</choose>
{% endhighlight %}

Notice how the **[cache-store-value](https://docs.microsoft.com/en-us/azure/api-management/api-management-caching-policies)**-policy has a **duration**-property set to 300. In this case, this means that the cached value will remain there for 300 *seconds* and only 1 GET-operation will be executed for this secret every 5 minutes.  
Depending on you needs this can either be increased or decreased, keeping in mind the throttling limitations of KeyVault of course.

### APIM Policy: comparing values
In the aforementioned steps, we retrieved the actual/cached secret from KeyVault, but this still needs to be compared against the value provided byt the client calling our API.  
In order to do this, simply use the '**[check-header](https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies#CheckHTTPHeader)**'-policy which will compare the value present in the given HTTP-header and any given value - *in this case, the value retrieved from KeyVault*.

{% highlight xml %}
<!-- Compare the value of the HTTP-header 'api-key' with the variable 'authentication-token'
    If these don't match, return a 401-Unauthorized error. -->
<check-header name="api-key" failed-check-httpcode="401" failed-check-error-message="Unauthorized" ignore-case="true">
    <value>@((string)context.Variables["authentication-token"])</value>
</check-header>
{% endhighlight %}
By adding this after the caching/lookup-policy, you can ensure that any HTTP-header is being compared with a - cached version of a - KeyVault-secret.


> ***Hint:** In order to prevent having to repeat this section, you can add this as part of the base-policy.*

For an advanced version of the caching-policy, which incorporates a retry-mechanism in case a KeyVault-secret has expired, have a look at the following blog: [https://blog.eldert.net/implementing-smart-caching-of-secrets-in-azure-api-management-policies/](https://blog.eldert.net/implementing-smart-caching-of-secrets-in-azure-api-management-policies/)
