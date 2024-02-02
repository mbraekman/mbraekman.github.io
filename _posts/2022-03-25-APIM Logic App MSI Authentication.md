---
layout: post
author: Maxim Braekman
title: Use MSI-authentication between APIM and Logic Apps
description: Restrict access to an Azure Logic App using Azure API Management with MSI-authentication.
image: ./img/azure-sql-threat-detection.jpg
tags: [Azure Logic Apps, API Management, Security]
---

When creating an Azure Logic App to perform a specific part of your application logic and is being triggered by an HTTP-trigger, by default this is publicly accesible.
Agreed, you do need to be obtain the generated SAS-token before you will be able to succesfully deliver your payload and actually trigger a Logic App run, but still it is not very secure.
By making use of the combination of MSI-authentication and Azure Active Directory Authorization Policies on top of Azure Logic Apps, you can highly increase the level of security.

## Steps
- [Getting Started](#getting-started)
- [Securing the Logic App](#securing-the-logic-app)
- [Set the APIM Policy](#set-the-apim-policy)
- [Conclusion](#conclusion)

## Getting Started



## Securing the Logic App
Part of the solution to make your newly developed Azure Logic App fully secure, is limit access to it by:
- Assigned an Azure AD Authorization Policy, which will validate whether the request is containing an authorization token, which contains specific claims/values.
- Limited the range of IP addresses allowed to make a request to the Azure Logic App HTTP Trigger endpoint.

### Create an Azure AD Authorization Policy
Creating an Azure AD Authorization Policy implies that every request being transmitted towards your Azure Logic App must contain an `Authorization`-header with a valid `Bearer`-token and allows you to specify additional rules to validate the issuer, audience and/or any claims.
In our case, this allows us to validate that the issuer is our own Azure Active Directory tenant and the object-id matches the system-assigned id of our API Management instance.
```json
"accessControl": {
	"triggers": {
		"openAuthenticationPolicies": {
			"policies": {
				"APIM-only": {
					"type": "AAD",
					"claims": [
						{
							"name": "iss",
							"value": "[concat('https://sts.windows.net/', subscription().tenantId, '/')]"
						},
						{
							"name": "oid",
							"value": "[reference(resourceId(parameters('ApiManagement.ResourceGroup'), 'Microsoft.ApiManagement/service', parameters('ApiManagement.Name')) ,'2019-01-01', 'Full').identity.principalId]"
						}
					]
				}
			}
		},
		"allowedCallerIpAddresses": []
	}
}
```

### Limit access to the public IP op API Management
Within the same `accesscontrol`-section, you can limit access to a specific range of IP addresses. In this case, you can add the public IP address of the Azure API Management instance.

*Note: Keep in mind that this will not be possible in case of the consumption tier version of APIM, since this does not have a static IP and thus might change over time.*

The property to be assigned does require a valid IP **range** as input, so in this case you can resolve this by using the following approach:
```json
"allowedCallerIpAddresses": [
    {
        "addressRange": "[concat(parameters('ApiManagement.PublicIP'), '-', parameters('ApiManagement.PublicIP'))]"
    }
]
```

## Set the APIM Policy
Within the Azure API Management, adjust the policy of the API forwarding the call towards your Azure Logic App, by providing the following section:

```xml
<policies>
    <inbound>
        <base />
            <!-- Set the base-url of the http-trigger endpoint of the Logic App to call -->
		    <set-backend-service base-url="{{backend-logicapp-http-trigger-baseurl}}" />

            <!-- Make sure to refer to the correct name of the trigger -->
            <rewrite-uri id="apim-generated-policy" template="/?api-version=2016-06-01&amp;sp=/triggers/manual/run&amp;sv=1.0&amp;" />

            <!-- Set the HTTP operation, in case it would be different from the operation activating the APIM operation -->
            <set-method id="apim-generated-policy">POST</set-method>

            <!-- Ensure to remove the Subscription-key from the headers -->
            <set-header id="apim-generated-policy" name="x-api-key" exists-action="delete" />

            <!-- Have a MSI-authentication token generated and stored in the Authorization header -->
            <authentication-managed-identity resource="https://management.azure.com/" />
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

## Conclusion

