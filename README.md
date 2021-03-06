---
page_type: sample
languages:
- javascript
products:
- azure
description: "This sample explains how you can create a VM in Nodejs, with certificates installed automatically 
from a Key Vault account."
urlFragment: key-vault-node-deploy-certificates-to-vm
---

# Deploy Certificates to VMs from customer-managed Key Vault in Node

This sample explains how you can create a VM in Nodejs, with certificates installed automatically 
from a Key Vault account.

## Getting Started

### Prerequisites

- An Azure subscription

- A Service Principal. Create an Azure service principal either through
[Azure CLI](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal-cli/),
[PowerShell](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal/)
or [the portal](https://azure.microsoft.com/documentation/articles/resource-group-create-service-principal-portal/).

### Installation

1.  If you don't already have it, [get node.js](https://nodejs.org).

1.  Clone the repository.

    ```
    git clone https://github.com/Azure-Samples/key-vault-node-deploy-certificates-to-vm.git
    ```

2.  Install the dependencies using pip.

    ```
    cd key-vault-node-deploy-certificates-to-vm
    npm install
    ```

1. Export these environment variables into your current shell or update the credentials in the example file.

    ```
    export AZURE_TENANT_ID={your tenant id}
    export AZURE_CLIENT_ID={your client id}
    export AZURE_CLIENT_SECRET={your client secret}
    export AZURE_SUBSCRIPTION_ID={your subscription id}
    ```

    Optional, but recommended; this sample will attempt to obtain through graph if not provided:
    ```
    export AZURE_OBJECT_ID={your object id},
    ```
    you can access the object_id through portal by navigating to your service principal.
    ![Object ID through Portal screenshot](portal_obj_id.JPG)

    > [AZURE.NOTE] On Windows, use `set` instead of `export`.

1. Run the sample.

    ```
    node index.js
    ```

## Demo

### Preliminary operations

This example setup some preliminary components that are no the topic of this sample and do not differ
from regular scenarios:

- A Resource Group
- A Virtual Network
- A Subnet
- A Public IP
- A Network Interface

For details about creation of these components, you can refer to the generic sample:

- [Managing VMs](https://github.com/Azure-Samples/compute-node-manage-vm)

### Creating a KeyVault account enabled for deployment

```javascript
    let keyVaultParameters = {
      location: location,
      tenantId: domain,
      properties: {
        sku: {
          name: 'standard',
        },
        accessPolicies: [
          {
            tenantId: domain,
            objectId: objectId,
            permissions: {
              certificates: ['all'],
              secrets: ['all']
            }
          }
        ],
        # Critical to allow the VM to download certificates later
        enabledForDeployment: true
      },
      tags: {}
    };
    return keyVaultManagementClient.vaults.createOrUpdate(resourceGroupName, keyVaultName, keyVaultParameters);
```

You can also found different example on how to create a Key Vault account:

  - From Node SDK: https://github.com/Azure-Samples/key-vault-node-getting-started

> In order to execute this sample, your Key Vault account MUST have the "enabled-for-deployment" special permission.
  The EnabledForDeployment flag explicitly gives Azure (Microsoft.Compute resource provider) permission to use the certificates stored as secrets for this deployment. 

> Note that access policy takes an *object_id*, not a client_id as parameter. This sample provides a quick way to convert a Service Principal client_id to an object_id using the `azure-graphrbac` client, provided the service principal used has the permission to view information on Microsoft Graph; this can be done by navigating to the service-principal => Settings => Required Permissions => Add (Microsoft Graph). Alternatively, simply set the `AZURE_OBJECT_ID` environment variable ([see above](https://github.com/Azure-Samples/key-vault-node-deploy-certificates-to-vm#installation)).


### Ask Key Vault to create a certificate for you

```javascript
    keyVaultClient.createCertificate(vaultObj.properties.vaultUri, certificateName, { certificatePolicy: certificatePolicy });
```

A default `certificatePolicy` is described in the sample file:
```javascript
    let certificatePolicy = {
      keyProperties: {
        exportable: true,
        keyType: 'RSA',
        keySize: 2048,
        reuseKey: true
      },
      secretProperties: {
        contentType: 'application/x-pkcs12'
      },
      issuerParameters: {
        name: 'Self'
      },
      x509CertificateProperties: {
        subject: 'CN=CLIGetDefaultPolicy',
        validity_in_months: 12,
        key_usage: [
          "cRLSign",
          "dataEncipherment",
          "digitalSignature",
          "keyEncipherment",
          "keyAgreement",
          "keyCertSign"
        ]
      },
      lifetimeActions: [
        {
          action: { actionType: "AutoRenew" },
          trigger: { daysBeforeExpiry: 90 }
        }
      ]
    };
```

This is the same policy that:

- Is pre-configured in the Portal when you choose "Generate" in the Certificates tab
- You get when you use the CLI 2.0: `az keyvault certificate get-default-policy`

> Create certificate is an async operation. This sample provides a simple polling mechanism example.

### Create a VM with Certificates from Key Vault

First, get your certificate as a Secret object:

```javascript
    keyVaultClient.getSecret(vaultObj.properties.vaultUri, certificateName, '')
```

During the creation of the VM, use the `secrets` atribute to assign your certificate:.

```javascript
  let vmParameters = {
    location: location,
    osProfile: {
      computerName: vmName,
      adminUsername: adminUsername,
      adminPassword: adminPassword,
      // Key Vault Critical part
      secrets: [{
        sourceVault: {
            id: vaultObj.id,
        },
        vaultCertificates: [{
            certificateUrl: certificateSecretObj.id
        }]
    }]
    },
    hardwareProfile: {
      vmSize: 'Basic_A0'
    },
    storageProfile: {
      imageReference: {
        publisher: publisher,
        offer: offer,
        sku: sku,
        version: "latest"
      }
    },
    networkProfile: {
      networkInterfaces: [
        {
          id: nicId,
          primary: true
        }
      ]
    }
  };

  return computeClient.virtualMachines.createOrUpdate(resourceGroupName, vmName, vmParameters);
```
> [AZURE.NOTE] By default, this sample deletes the resources created in the sample. To retain them, comment out the line: `return resourceClient.resourceGroups.deleteMethod(resourceGroupName);`

## Resources

- https://azure.microsoft.com/services/key-vault/
- https://github.com/Azure/azure-sdk-for-node
- https://docs.microsoft.com/node/api/overview/azure/key-vault
