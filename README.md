# Use Azure KeyVault as configuration provider in .net core for on-premise
This is to explain how to use Azure Key Vault for keeping secret configuration in .net core for on-premise.
As the .net core app will be on-premise, we cannot use Managed Identities, so we should use certificates and Azure AD Applications.

## Prepare the infrastructure
We use PowerShell 7.0 and az module.
### Create the key vault
Let's connect to azure first using az PowerShell module:
``` powershell
Import-Module az
Connect-AzAccount
```
We create the resource group:
``` powershell
New-AzResourceGroup -Name 'rg-cace-dev-kvtest-01' -Location 'Canada Central'
```
Now we can create a key vault in that resource group:
``` powershell
New-AzKeyVault -VaultName 'my-kv-01' -ResourceGroupName 'rg-cace-dev-kvtest-01' -Location 'Canada Central'
```
And finally let's add some secrets to our key vault:
``` powershell
$secretvalue = ConvertTo-SecureString "my very secret value" -AsPlainText -Force
$secret = Set-AzKeyVaultSecret -VaultName "my-kv-01" -Name "MySecret" -SecretValue $secretvalue
```

### Generate Self-Signed Certificate
Create a certificate for localhost and store it current user:
``` powershell
$cert = New-SelfSignedCertificate -Subject "localhost" `
 -TextExtension @("2.5.29.17={text}DNS=localhost&IPAddress=127.0.0.1&IPAddress=::1") `
 -CertStoreLocation "Cert:\CurrentUser\My"
```
Export the certificate as a DER-encoded certificate (.cer):
``` powershell
Export-Certificate -Cert $cert -FilePath c:\temp\mycert.cer
```
Or export it as pfx:
``` powershell
$pwd = ConvertTo-SecureString -String "password1234" -Force -AsPlainText
$path = "Cert:\CurrentUser\My\" + $cert.thumbprint
Export-PfxCertificate -cert $path -FilePath c:\temp\mycert.pfx -Password $pwd
```
Store the certificate thumbprint somewhere, we're going to use it later:
``` powershell
$cert.Thumbprint
```
### Add Azure AD application
Create a new Azure AD application, usaing the certificate:
``` powershell
$binCert = $cert.GetRawCertData() 
$credValue = [System.Convert]::ToBase64String($binCert)
$adApplication = New-AzADApplication -DisplayName "my-kv-test-01" `
    -IdentifierUris "https://localhost:5001" `
    -CertValue $credValue `
    -EndDate $cert.NotAfter `
    -StartDate $cert.NotBefore
```
Add permissions to Azure AD application so it can access your Key Vault by using a service princiapl and key vault access policies:
``` powershell
$servicePrincipal = New-AzADServicePrincipal `
    -ApplicationId $adApplication.ApplicationId `
    -CertValue $credValue `
    -EndDate $cert.NotAfter `
    -StartDate $cert.NotBefore

Set-AzKeyVaultAccessPolicy -VaultName 'my-kv-01' `
    -ServicePrincipalName $servicePrincipal.ServicePrincipalNames[0] `
    -PermissionsToSecrets all -PermissionsToKeys all
```

## Sample asp.net core application
Add package references for the following packages:
* Azure.Extensions.AspNetCore.Configuration.Secrets
* Azure.Identity

The X.509 certificate is managed by the Windows. The application calls AddAzureKeyVault with values from appsettings.json. The program.cs file should be like this:
``` csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((context, config) =>
    {
        var builtConfig = config.Build();

        using var store = new X509Store(StoreLocation.CurrentUser);
        store.Open(OpenFlags.ReadOnly);
        var certs = store.Certificates.Find(
            X509FindType.FindByThumbprint,
            builtConfig["AzureADCertThumbprint"], false);

        config.AddAzureKeyVault(new Uri($"https://{builtConfig["KeyVaultName"]}.vault.azure.net/"),
                                new ClientCertificateCredential(builtConfig["AzureADDirectoryId"], builtConfig["AzureADApplicationId"], certs.OfType<X509Certificate2>().Single()),
                                new KeyVaultSecretManager());

        store.Close();
    })
    .ConfigureWebHostDefaults(webBuilder =>
    {
        webBuilder.UseStartup<Startup>();
    });
```
Update your appsettings.json file by adding these key/values :
``` json
{
  "KeyVaultName": "Key Vault Name",
  "AzureADApplicationId": "Azure AD Application ID",
  "AzureADCertThumbprint": "Azure AD Certificate Thumbprint",
  "AzureADDirectoryId": "Azure AD Directory ID"
}
```
