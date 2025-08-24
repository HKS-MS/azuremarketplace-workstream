
# Azure Application Offer

This repository provides the necessary files and instructions to create an Azure application offer.

## Required Files

To successfully deploy an Azure application, include the following two files in the root folder of a `.zip` archive:

### 1. `mainTemplate.json`
This is a Resource Manager template file that defines the resources to be deployed into the customer's Azure subscription.

- It contains the infrastructure and configuration settings.
- For examples, refer to:
  - [Azure Quickstart Templates Gallery](https://azure.microsoft.com/en-us/resources/templates/)
  - [Azure Resource Manager Quickstart Templates GitHub Repo](https://github.com/Azure/azure-quickstart-templates)

### 2. `createUiDefinition.json`
This file defines the user interface for the Azure application creation experience.

- It specifies UI elements that allow consumers to input parameter values.
- Helps guide users through the deployment process with a customized interface.

## Notes
- Ensure both files are placed in the root of the `.zip` archive.
- Validate the templates using Azure tools before publishing.



## Deployment Instructions

To deploy your Azure application package, follow these steps:

### 1. Create a Storage Account
Open a new PowerShell terminal in Visual Studio Code and sign in to your Azure subscription:

```powershell
Connect-AzAccount
```

Create a resource group:

```powershell
New-AzResourceGroup -Name packageStorageGroup -Location westus
```

Define parameters and create the storage account using PowerShell splatting:

```powershell
$pkgstorageparms = @{
  ResourceGroupName = "packageStorageGroup"
  Name = "<pkgstorageaccountname>"
  Location = "westus"
  SkuName = "Standard_LRS"
  Kind = "StorageV2"
  MinimumTlsVersion = "TLS1_2"
  AllowBlobPublicAccess = $true
  AllowSharedKeyAccess = $false
}

$pkgstorageaccount = New-AzStorageAccount @pkgstorageparms
```

### 2. Assign Role to Storage Account
Assign the role **Storage Blob Data Contributor** to your Microsoft Entra user account at the storage account scope. This may require additional permissions from your administrator.

Refer to:
- [Assign an Azure role for access to blob data](https://learn.microsoft.com/en-us/azure/role-based-access-control/blob-access)
- [Assign Azure roles using the Azure portal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal)

### 3. Upload the Package
After the role assignment becomes active, create the context and upload your `.zip` package:

```powershell
$pkgstoragecontext = New-AzStorageContext -StorageAccountName $pkgstorageaccount.StorageAccountName -UseConnectedAccount

New-AzStorageContainer -Name appcontainer -Context $pkgstoragecontext -Permission blob

$blobparms = @{
  File = "app.zip"
  Container = "appcontainer"
  Blob = "app.zip"
  Context = $pkgstoragecontext
}

Set-AzStorageBlobContent @blobparms
```


## Retrieve Group ID and Role Definition ID

To manage resources for the customer, you need to assign an identity such as a user, security group, or application. This identity will have permissions on the managed resource group based on the assigned role (e.g., Owner or Contributor).

### 1. Get Group Object ID
This example uses a security group. Ensure your Microsoft Entra account is a member of the group.
Replace `<managedAppDemo>` with your group's name:

```powershell
$principalid = (Get-AzADGroup -DisplayName <managedAppDemo>).Id
```

To create a new Microsoft Entra group, visit:
- [Manage Microsoft Entra groups and group membership](https://learn.microsoft.com/en-us/entra/identity/groups/groups-overview)

### 2. Get Role Definition ID
Retrieve the role definition ID for the Azure built-in role you want to assign:

```powershell
$roleid = (Get-AzRoleDefinition -Name Owner).Id
```

You will use these values when deploying the managed application definition.


## Publish the Managed Application Definition

To publish your managed application definition, follow these steps using Azure PowerShell:

### 1. Create a Resource Group
Create a resource group for your managed application definition:

```powershell
New-AzResourceGroup -Name appDefinitionGroup -Location westus
```

### 2. Retrieve the Blob URL
Create a variable to store the URL for the package `.zip` file:

```powershell
$blob = Get-AzStorageBlob -Container appcontainer -Blob app.zip -Context $pkgstoragecontext
```

### 3. Define and Publish the Application
Use the following parameters to define and publish the managed application:

```powershell
$publishparms = @{
  Name = "sampleManagedApplication"
  Location = "westus"
  ResourceGroupName = "appDefinitionGroup"
  LockLevel = "ReadOnly"
  DisplayName = "Sample managed application"
  Description = "Sample managed application that deploys web resources"
  Authorization = "${principalid}:$roleid"
  PackageFileUri = $blob.ICloudBlob.StorageUri.PrimaryUri.AbsoluteUri
}

New-AzManagedApplicationDefinition @publishparms
```

### Parameter Details
- **ResourceGroupName**: Name of the resource group for the application definition.
- **LockLevel**: Prevents customers from performing undesirable operations. `ReadOnly` is the only supported value.
- **Authorization**: Specifies the principal ID and role definition ID for access control. Format: `${principalid}:${roleid}`. Use commas to separate multiple values.
- **PackageFileUri**: URL of the `.zip` package containing the required files.

Once the command completes, the managed application definition will be available in your resource group.
