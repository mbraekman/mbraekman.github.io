---
layout: post
author: Maxim Braekman
title: Scan for malicious file-content
description: Ensure the files uploaded to an API are safe to accept within your environment.
image: ./img/api-file-scanning.jpg
tags: [API Development, Security, Bicep]
---

Security should always be a top priority when designing and developing applications that interact with the internet.
One important security measure is ensuring that files that are being uploaded to your application do not contain viruses or malware that could harm your system or other users.
In this article, we'll explore how to use ClamAV to validate the content of uploaded files, including how to set up and run a ClamAV instance in a Docker container and how to customize it for your needs.

## Steps
- [Getting Started](#getting-started)
- [Automated Deployment](#automated-deployment)
- [Custom Configuration](#custom-configuration)
- [Conclusion](#conclusion)

## Getting Started

When exposing an endpoint that is being used to upload files, you might want to include some sort of validation/security mechanism to ensure that the file content is safe.
Instead of relying solely on file extension validation, there is a way to fully validate the payload by using an anti-virus, without having to run it on a virtual machine.

The [nClam NuGet Package](https://www.nuget.org/packages/nClam) allows you to access ClamAV's file-scanning capabilities, with an instance of ClamAV running on an Azure Container Instance using the _mkodockx/docker-clamav:alpine_ image.

This container image hosts an instance of the open-source anti-virus scanner and comes with an automated process to maintain an up-to-date database of known threats.

If you want to run an instance of the Docker container locally, you can use the provided docker-compose.yaml file.

```yml
version: '3'

services:
  clamav-mko:
    image: mkodockx/docker-clamav:alpine
    ports:
      - '3310:3310'
```

But how do you communicate with the container instance version of the anti-virus scanner? The nClam NuGet Package handles most of the work for you. All you need to do is specify the hostname (or private IP address when using a VNET) and port number, which is why the docker-compose file includes port mapping.


The code snippet below demonstrates how to use the ClamAV client to scan uploaded files for viruses. Simply create a new _ClamClient_ object by providing the hostname and port number, use the _SendAndScanFileAsync_-method to scan the file content, and wait for the result. If a virus is detected, you can handle it accordingly, in this case, I'm merely returning a custom object including the results of the scan.

```csharp
var clamClient = new ClamClient(_fileScannerSettings.HostName, _fileScannerSettings.PortNumber);

using var memoryStream = new MemoryStream();
await fileStream.CopyToAsync(memoryStream);
var result = await clamClient.SendAndScanFileAsync(memoryStream.ToArray());

var isSafeFile = result.InfectedFiles == null || result.InfectedFiles.Count == 0;
if (isSafeFile)
{
     _logger.LogInformation("The file with name '{fileName}' has been scanned and was considered to be safe", fileName);
}
else
{
     _logger.LogWarning("The file with name '{fileName}' has been scanned and was considered to be malicious: {description}", fileName, result.Result.ToString());
}

return new ScanResult()
{
     IsSafe = isSafeFile,
     Description = result.Result.ToString()
};
```


## Automated Deployment
Once you are ready to have everything running in Azure, you will want to include the setup of the Azure Container Instance as part of your automated deployments.
To achieve this, you can utilize the provided Bicep file to create the container instance and have it linked to the container image mentioned earlier.

```bicep
param global object
param naming object

resource containerInstanceAv 'Microsoft.ContainerInstance/containerGroups@2021-09-01' = {
  name: naming.container_instance_av
  location: global.location
  properties: {
    sku: 'Standard'
    containers: [
      {
        name: naming.container_instance_av
        properties: {
          resources: {
            requests: {
              cpu: 1
              memoryInGB: 2
            }
          }
          image: 'mkodockx/docker-clamav:alpine'
          ports: [
            {
              port: 3310
              protocol: 'TCP'
            }
          ]
        }
      }
    ]
    osType: 'Linux'
    restartPolicy: 'OnFailure'
    ipAddress: {
      type: 'Public'
      ports: [
        {
          port: 3310
          protocol: 'TCP'
        }
      ]
      dnsNameLabel: naming.container_instance_av
    }
  }
}
```


## Custom Configuration
One thing to keep in mind though is that the ClamAV container has a default file size limit of 25MB. However, you can increase this limit by creating a custom clamd.conf file and increasing the StreamMaxLength-value. For example, if you want to set the file size limit to 1GB, you will have to set the value to 1000M.

To import the custom configuration file into the container, you can mount a custom file share (Storage Account) that requires a Shared Key for access. Once this volume has been mounted, set the CLAMD_CONF_FILE environment variable to point to the custom configuration file on the mounted file share.

During development, when running the container locally, you won't have to point to an online file share of course.
You can simply modify your docker-compose.yml file as shown below:

```yml
version: '3'

services:
  clamav-mko:
    image: mkodockx/docker-clamav:alpine
    ports:
      - '3310:3310'
    volumes:
      - ./clamd.conf:/mnt/clamd.conf
    environment:
      - CLAMD_CONF_FILE=/mnt/clamd.conf
```

Of course, adding a custom configuration file and mounting a file share should also be part of your automated deployment process.
To achieve this you can use the deploymentScript-resource type to upload the file onto the Storage Account, before triggering the creation of the container instance.

```bicep
param global object
param naming object

resource deployScript 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: '${naming.container_instance_av}-deploy-custom-config'
  location: global.location
  dependsOn: [
    fileshare
  ]
  kind: 'AzureCLI'
  properties: {
    azCliVersion: '2.26.1'
    retentionInterval: 'PT1H'
    timeout: 'PT5M'
    environmentVariables: [
      {
        name: 'STORAGE_ACCOUNT_NAME'
        value: storageAccount.name
      }
      {
        name: 'STORAGE_ACCOUNT_KEY'
        value: listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value
      }
      {
        name: 'FILE_SHARE_NAME'
        value: naming.storage_documents_fileshare_ci
      }
      {
        name: 'CONTENT'
        value: replace(loadTextContent('../Docker/clamd.conf'), '\r\n', '\n')
      }
    ]
    scriptContent: 'echo "$CONTENT" > "clamd.conf" && az storage file upload --source "clamd.conf" --share-name "$FILE_SHARE_NAME" --account-name $STORAGE_ACCOUNT_NAME --account-key $STORAGE_ACCOUNT_KEY'
  }
}

resource containerInstanceAv 'Microsoft.ContainerInstance/containerGroups@2021-09-01' = {
  name: naming.container_instance_av
  location: global.location
  properties: {
    sku: 'Standard'
    containers: [
      {
        name: naming.container_instance_av
        properties: {
          environmentVariables: [
            {
              name: 'CLAMD_CONF_FILE'
              value: '/mnt/share/clamd.conf'
            }
          ]
          resources: {
            requests: {
              cpu: 1
              memoryInGB: 2
            }
          }
          image: 'mkodockx/docker-clamav:alpine'
          ports: [
            {
              port: 3310
              protocol: 'TCP'
            }
          ]
          volumeMounts: [
            {
              name: 'filesharevolume'
              mountPath: '/mnt/share/'
            }
          ]
        }
      }
    ]
    osType: 'Linux'
    restartPolicy: 'OnFailure'
    ipAddress: {
      type: 'Public'
      ports: [
        {
          port: 3310
          protocol: 'TCP'
        }
      ]
      dnsNameLabel: naming.container_instance_av
    }
    volumes: [
      {
        name: 'filesharevolume'
        azureFile: {
          storageAccountName: storageAccount.name
          shareName: naming.storage_documents_fileshare_ci
          storageAccountKey: listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value
        }
      }
    ]
  }
}
```

## Conclusion
With ClamAV, you can add an extra layer of security to your application by validating uploaded files' content. This can help prevent malware and viruses from entering your system, protecting your data and your users. Setting up and running ClamAV in a Docker container is straightforward, and customizing it to your needs is easy.
By following the steps outlined in this article, you can ensure your application is secure against file-based threats.