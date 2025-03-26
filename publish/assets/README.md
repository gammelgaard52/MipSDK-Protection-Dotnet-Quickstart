# Important

## Required files

publish/AppxManifest.xml
publish/assets/*

## Signing package

```powershell
# Create certificate
New-SelfSignedCertificate `
  -Subject "CN=Your name" `
  -Type Custom `
  -KeyUsage DigitalSignature `
  -FriendlyName "MipProtectionCert" `
  -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3") `
  -CertStoreLocation "Cert:\CurrentUser\My"

# Ensure validity
$cert = Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -eq "CN=Your name" }
$cert.EnhancedKeyUsageList

# Export private key for safekeeping and for signing
Export-PfxCertificate -Cert $cert -FilePath "MipProtection.pfx" -Password (ConvertTo-SecureString "YourPassword123!" -AsPlainText -Force)

# Sign the package
signtool sign /fd SHA256 /a /f MipProtection.pfx /p YourPassword123! MipProtection.msix
```

## Export public cert for distribution with the package

```powershell
# Export public key to be distributed with the package
$cert = Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -eq "CN=Your name" }
Export-Certificate -Cert $cert -FilePath "MipProtection.cer"
```

## Install certificate before running package install

```powershell
# Point to the certificate
$certPath = "MipProtection.cer"

# Setup paths for install certificate to
$stores = @(
    @{ Location = "LocalMachine"; StoreName = "Root" },              # Root CA (trust issuer)
    @{ Location = "LocalMachine"; StoreName = "TrustedPublisher" }  # Publisher trust
)

# Install certificate to paths
foreach ($storeInfo in $stores) {
    $store = New-Object System.Security.Cryptography.X509Certificates.X509Store $storeInfo.StoreName, $storeInfo.Location
    $store.Open("ReadWrite")
    try {
        Write-Host "Importing to $($storeInfo.Location)\$($storeInfo.StoreName)..."
        Import-Certificate -FilePath $certPath -CertStoreLocation "Cert:\$($storeInfo.Location)\$($storeInfo.StoreName)" | Out-Null
    } catch {
        Write-Warning "Failed to import into $($storeInfo.Location)\$($storeInfo.StoreName): $_"
    } finally {
        $store.Close()
    }
}
```

Now the package can be installed
