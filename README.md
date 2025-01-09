# Azure Functions Flex Consumption - Migration Guide

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]
![GitHub commit activity](https://img.shields.io/github/commit-activity/m/erwinkramer/azure-functions-flex-consumption-migration-guide)

A guide to migrate to Azure Functions Flex Consumption plan hosting. Contains specific steps for secure, identity based, C# Functions with VNet integration. With continuous deployment via Azure DevOps.

A fully working Bicep sample is available at [Azure Functions Flex Consumption Samples](https://github.com/Azure-Samples/azure-functions-flex-consumption-samples/).

## Generic settings

Many app settings and properties are moved to a different place in an Flex Consumption plan or simply depreciated entirely, for a complete overview, see [Flex Consumption plan deprecations](https://learn.microsoft.com/en-us/azure/azure-functions/functions-app-settings#flex-consumption-plan-deprecations).

A big change is the introduction of the `properties.functionAppConfig` element in `Microsoft.Web/sites`, available from versions `@2023-12-01` and up.

Of course, the most obvious change is the SKU, on `Microsoft.Web/serverfarms@2023-12-01`, the `kind` and `sku` will look like this:

```bicep
kind: 'functionapp'
sku: {
    tier: 'FlexConsumption'
    name: 'FC1'
}
```

If coming from a premium plan, make sure to remove most properties from the `serverfarms` resource, you will end up with just this simple configuration:

```bicep
properties: { 
    reserved: true
}
```

## Scaling settings

Scaling now is set on the `sites` resource, instead of the `serverfarms` resource, it might look like this inside the `properties` element:

```bicep
@maxValue(1000)
param maximumInstanceCount int = 100
@allowed([2048, 4096])
param instanceMemoryMB int = 2048

...

functionAppConfig: {
    scaleAndConcurrency: {
        maximumInstanceCount: maximumInstanceCount
        instanceMemoryMB: instanceMemoryMB
    }
}
```

## C# app properties

To specify C# properties;

- change app setting `FUNCTIONS_WORKER_RUNTIME_VERSION`=`8.0` to resource setting `properties.functionAppConfig.runtime.version`=`8.0`.
- change app setting `FUNCTIONS_WORKER_RUNTIME`=`dotnet-isolated` to resource setting `properties.functionAppConfig.runtime.name`=`dotnet-isolated`.
- delete `properties.siteConfig.netFrameworkVersion`=`v8.0`
- if on an old linux plan; delete `properties.siteConfig.LinuxFxVersion`=`DOTNET-ISOLATED|8.0`.
- if on an old windows plan; delete `properties.siteConfig.windowsFxVersion`=`DOTNET-ISOLATED|8.0`.

## Identity-based backend connections

If coming from an [identity-based connection](https://learn.microsoft.com/en-us/azure/azure-functions/functions-identity-based-connections-tutorial) for `AzureWebJobsStorage` (host), this still works with identities, but requires a different setup:

- delete app setting `AzureWebJobsStorage__blobServiceUri`
- delete app setting `AzureWebJobsStorage__queueServiceUri`
- delete app setting `AzureWebJobsStorage__tableServiceUri`
- add app setting `AzureWebJobsStorage__accountName` and point to the storage account name of the backend storage.
- add `authentication.type` property on the deployment element, as specified under [Deployment](#deployment), this will make sure the connection to the backend uses a managed identity.

## Network behavior

If using VNet integration:

- remove the `WEBSITE_VNET_ROUTE_ALL` app setting
- remove `properties.vnetImagePullEnabled`
- remove `properties.vnetRouteAllEnabled`
- remove `properties.vnetContentShareEnabled`
- remove `properties.vnetBackupRestoreEnabled`
- remove the  `Microsoft.Web/sites/networkConfig` resource entirely, it might have looked like this:

```bicep
resource networkConfig 'Microsoft.Web/sites/networkConfig@2023-01-01' = {
  parent: functionApp
  name: 'virtualNetwork'
  properties: {
    subnetResourceId: appServicePlanSubnetId
    swiftSupported: true
  }
}
```

- add `properties.virtualNetworkSubnetId`=`appServicePlanSubnetId` to `Microsoft.Web/sites`
- change delegation of the subnet, so for the resource `Microsoft.Network/virtualNetworks/subnets`; `properties.delegations.properties.serviceName` used to be `Microsoft.Web/serverFarms`, now it is `Microsoft.App/environments`.
- register the `Microsoft.App` [resource provider](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-services-resource-providers#container-resource-providers) on subscription level.

### VNet-based backend connections

If coming from a [secure VNet integrated connection](https://learn.microsoft.com/en-us/azure/azure-functions/configure-networking-how-to?tabs=portal#restrict-your-storage-account-to-a-virtual-network) for `AzureWebJobsStorage` (host), this still works with VNet integration, but requires a different setup:

- delete app setting `WEBSITE_CONTENTOVERVNET`

That's all there is to it! You can isolate your storage account completely with private endpoints, and block all public access.

## Deployment

To deploy a C# app, or any other language app, it's completely different to what you are used to, zip deploy isn't what it used to be anymore. You will still zip, but the internal logic will unzip itself. The app will manage all deployments in a storage account using a container. In a Premium plan, managing deployments used to be within a file share:

- delete app setting `WEBSITE_RUN_FROM_PACKAGE`.
- if coming from a premium plan: delete app setting `WEBSITE_CONTENTSHARE`.
- add a `deployment` element to your `properties.functionAppConfig` element, it might look like this:

```bicep
deployment: {
    storage: {
        type: 'blobContainer'
        value: '${storage.properties.primaryEndpoints.blob}${deploymentStorageContainerName}'
        authentication: {
            type: 'SystemAssignedIdentity'
        }
    }
}
```

> This setup, with `authentication.type`=`SystemAssignedIdentity`, assumes the Function App has data permissions on the `deploymentStorageContainerName`.

- add a deployment container to your backend storage account, and use as specified in the `deploymentStorageContainerName` variable, it might look like this on a `Microsoft.Storage/storageAccounts/blobServices@2023-04-01` resource:

```bicep
resource container 'containers' = [
    for container in containers: {
        name: container.name
        properties: {
            publicAccess: 'None'
        }
    }
]
```

## C# Deployment

To deploy your actual code inside an Azure DevOps environment, deploy steps will be very similar.

### Deploy steps

Deployment makes use of the [Azure Functions Deploy task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-function-app-v2?view=azure-pipelines), with some small changes:

- add the `isFlexConsumption` input and set to `true`.
- leave the `deploymentMethod` input at default value `auto` (or omit this setting), instead of `zipDeploy` or `runFromPackage`.
- leave the `deployToSlotOrASE` input at default value `false` (or omit this setting), instead of `true`.
- set the `appType` to `functionAppLinux`.

## License

This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg