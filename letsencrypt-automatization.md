# Automating SSL Certificates enrollment to the cloud with LetsEncrypt and Azure DevOps

To automate the creation and extension for LetsEncrypt certificates, the "Automated Certificate Management Environment (ACME)" protocol - which is designed to automate the certificate issuance - is used. 
The following steps are based on an (outdated) article [here](https://bit.ly/34qOsil).

We use Azure and Azure DevOps together to automate the certificate issuance and configuration processes. To achieve this, we use 
* Posh-ACME as our PowerShell-based ACME client, 
* Let’s Encrypt as our certificate authority, and 
* we complete DNS challenges to prove control of our domain.

Therefore, we require the following resources:
1. **Azure Storage Account** to store ACME server information, certificate order details, certificates, and private keys on Blob storage.
2. **Azure DNS** to publish our DNS challenges for the corresponding FQDNs. You’ll need to configure your domain registrar to use the DNS zone you’ve created in Azure. As we managing the DNS zone 'example.org' with Azure, we may need a additonal sub zone for the corresponding stage and application in the form of `<stage>.<application>.example.org`.
3. **Azure KeyVault** to store and provide the issued certificates for the requesting endpoint.
4. An **Azure DevOps project and repo** to store a code repository, build pipeline, and service connection.

The overall certificate enrollment (and renewal) process contains two main steps:
1. Issue Certificate with Posh-ACME and
1. Store the issued certificate in Azure KeyVault

## Azure Storage Account

The Blob Storage was created with PowerShell code, but could also be prepared with Azure CLI, Terraform etc.

#### Blob Storage
```powershell
$RGName = "rg-ACME"
$SAName = "storacctacmecerts"
$Location = "westeurope"
New-AzStorageAccount -Name $SAName -ResourceGroupName $RGName -Location $Location -Kind StorageV2 -SkuName Standard_LRS -EnableHttpsTrafficOnly $true
```

#### Create SAS token in the Azure Storage Container
```powershell
New-AzStorageContainerSASToken -Name "poshacme" -Permission rwdl -Context $storageContext -FullUri -ExpiryTime (Get-Date).AddYears(5)
```

## Create Azure KeyVault
```powershell
New-AzKeyVault -Name "kv-acmecerts" -ResourceGroupName $RGName -Location $Location -EnablePurgeProtection
```

## Create the service principal
```powershell
$application = New-AzADApplication -DisplayName "ACME Certificate Automation" -IdentifierUris "http://example.org/acme"

$servicePrincipal = New-AzADServicePrincipal -ApplicationId $application.ApplicationId

$servicePrincipalCredential = New-AzADServicePrincipalCredential -ServicePrincipalObject $servicePrincipal -EndDate (Get-Date).AddYears(5)
```

## Create DNS (sub-)zone
```powershell
New-AzDnsZone -Name "example.org" -ResourceGroupName $RGName
```

## Grant service principal access to the resources

#### Acces to DNS zone
```powershell
New-AzRoleAssignment -ObjectId $servicePrincipal.Id -ResourceGroupName $RGName -ResourceName "example.org" -ResourceType "Microsoft.Network/dnszones" -RoleDefinitionName "DNS Zone Contributor"
```

#### Access to Azure KV  
```powershell
$KVName = "kv-acmecerts"

New-AzRoleAssignment -ObjectId $servicePrincipal.Id -ResourceGroupName $RGName -ResourceName $KVName -ResourceType "Microsoft.KeyVault/vaults" -RoleDefinitionName "Reader"

Set-AzKeyVaultAccessPolicy -ResourceGroupName $RGName -VaultName $KVName -ObjectId $servicePrincipal.Id -PermissionsToCertificates Get, Import
```

#### Generate SAS token to acccess the data in the Azure storage container
```powershell
$storageAccountKey = Get-AzStorageAccountKey -Name $SAName -ResourceGroupName $RGName | Select-Object -First 1 -ExpandProperty Value
$storageContext = New-AzStorageContext -StorageAccountName $SAName -StorageAccountKey $storageAccountKey
New-AzStorageContainer -Name "poshacme" -Context $storageContext
```

#### Retrieve resource Id of Azure KV
```powershell
Get-AzKeyVault -ResourceGroupName $RGName -VaultName $KVName | Select-Object -ExpandProperty ResourceId
```

## Azure DevOps

We use Azure DevOps to orchestrate the certificate issuance process. The build pipeline is scheduled to execute at regular intervals to renew the certificate if required. 
The build pipeline uses a service connection to control Azure resources. The build pipeline is defined in YAML format and stored inside an Azure Repos Git repository `ACME/ACMECertificateAutomation`.

#### The build pipeline (YAML)

The actual build pipeline is referenced as a template file.

Here is an example for a certificate for the FQDN 'www.example.org':

```yaml
name: 0.0.$(Build.BuildId)

# Linux based agent; all except the first step will also work on Windows
pool:
  name: Hosted Ubuntu 1604

# The scheduled trigger will be set in the Azure DevOps portal
trigger: none

# Reference for the repository of the template file 
resources:
  repositories:
    - repository: ACMECertificateAutomation
      type: git
      name: ACME/ACMECertificateAutomation
      ref: 'refs/heads/master'

steps:
- checkout: git://ACME/ACMECertificateAutomation
- template: create_and_store_certificates.yml@ACMECertificateAutomation
  parameters:
    azureSubscription: '<YOUR SERVICE CONNECTION>'
    certs:
    - <FQDN of your cert #1>
    - <FQDN of your cert #2>
    - ...
```

So only the ```parameters``` block (including the service connection to the Azure subscription and the FQDN of the cert(s)) have to be maintained for the environment. With this construct you can use multiple pipelines with dedicated pipeline files.

#### The build pipeline (Variables)

AcmeContact: ```<Mail address of your Let's Encrypt Account>```

AcmeDirectory: either ```LE_PROD``` or ```LE_TEST```

KeyVaultResourceId: see [Retrieve resource Id of Azure KV](#retrieve-resource-id-of-azure-kv)

StorageContainerSASToken: see [Create SAS token in the Azure Storage Container](#create-sas-token-in-the-azure-storage-container)

#### The build pipeline (Triggers)

The pipeline should run at least once a month to extend the validation period of the issued certificates. The LetsEncryot certificates are only valid 90 days.
Running the ACMECertificateAutomation pipelines once a week during the weekend is a good approach.
