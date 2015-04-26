# HighTrustSharePointApps
Como criar um High-Trust SharePoint App

Esse exemplo mostra como criar uma High-Trust SharePoint App.

Para rodar esse exemplo você precisará:
- Visual Studio 2013
- SharePoint 2013 configurado para utilizar Apps
- Gerar um certificado
 
## Criar e Registrar o certificado

### Criar o certificado

Para criar umm novo certificado, utilize o script abaixo no PowerShell.

```PowerShell
$makecert = "C:\Program Files\Microsoft Office Servers\15.0\Tools\makecert.exe"
$certmgr = "C:\Program Files\Microsoft Office Servers\15.0\Tools\certmgr.exe"

# Alterar a Senha do certificado
$pfxPassword = "Pass@word1"

# Alterar o nome do certificado
$domain = "intranet"

# Alterar a pasta para armazenar os certificados
$outputDirectory = "c:\certificados\"
New-Item $outputDirectory -ItemType Directory -Force -Confirm:$false | Out-Null


$publicCertificatePath  =  $outputDirectory + $domain + ".cer"
$privateCertificatePath = $outputDirectory + $domain + ".pfx"

Write-Host 
Write-Host "criar o certificado .cer..."
& $makecert -r -pe -n "CN=$domain" -b 01/01/2012 -e 01/01/2022 -eku 1.3.6.1.5.5.7.3.1 -ss my -sr localMachine -sky exchange -sp "Microsoft RSA SChannel Cryptographic Provider" -sy 12 $publicCertificatePath

Write-Host 
Write-Host "Registrar no IIS..."
& $certmgr /add $publicCertificatePath /s /r localMachine root


$publicCertificate = Get-PfxCertificate -FilePath $publicCertificatePath
$publicCertificateThumbprint = $publicCertificate.Thumbprint

Get-ChildItem cert:\\localmachine\my | Where-Object {$_.Thumbprint -eq $publicCertificateThumbprint} | ForEach-Object {
    Write-Host "  Exportar a chave privada do certificado (*.PFK)" -ForegroundColor Gray 
    $privateCertificateByteArray = $_.Export("PFX", $pfxPassword)
    [System.IO.File]::WriteAllBytes($privateCertificatePath, $privateCertificateByteArray)
    Write-Host "  Certificado exportado" -ForegroundColor Gray 
}  
```

### Registrar o Certificado

Utilizar o Script abaixo para registrar o certificado no SharePoint

```PowerShell
Add-PSSnapin "Microsoft.SharePoint.PowerShell"

#alterar o guid abaixo
$issuerID = "11111111-1111-1111-1111-111111111111"

#alterar a URL do Site
$targetSiteUrl = "http://localhost"
$targetSite = Get-SPSite $targetSiteUrl
$realm = Get-SPAuthenticationRealm -ServiceContext $targetSite

$registeredIssuerName = $issuerID + '@' + $realm

Write-Host $registeredIssuerName 

#alterar o caminho do certificado
$publicCertificatePath = "C:\Certificados\intranet.cer"
$publicCertificate = Get-PfxCertificate $publicCertificatePath

Write-Host "Create token issuer"
$secureTokenIssuer = New-SPTrustedSecurityTokenIssuer `
                     -Name $issuerID `
                     -RegisteredIssuerName $registeredIssuerName `
                     -Certificate $publicCertificate `
                     -IsTrustBroker

$secureTokenIssuer | select *
$secureTokenIssuer  | select * | Out-File -FilePath "SecureTokenIssuer.txt"


$serviceConfig = Get-SPSecurityTokenServiceConfig
$serviceConfig.AllowOAuthOverHttp = $true
$serviceConfig.Update()

Write-Host "Pronto..."
```

## Criar uma nova solução

Para criar uma nova solução, vá em **File** e depois em **New Project...**
Escolha **SharePoint App**. 

Nesse exemplo, eu criei uma provider-hosted app.
![URL](https://cloud.githubusercontent.com/assets/12012898/7335406/8e9d8f72-eb91-11e4-9d52-ca1d065add63.png)

Escolha o tipo de projeto
![Project Type](https://cloud.githubusercontent.com/assets/12012898/7335408/8e9e55ba-eb91-11e4-9b43-a9f21ace7c11.png)

Escolha o certificado gerado no passo anterior, digite a senha do certificado e coloque um novo Guid no campo Issuer ID.

![Certificado](https://cloud.githubusercontent.com/assets/12012898/7335423/b6f89712-eb93-11e4-924f-daeee68f86b2.png)


## Rodando esse projeto

Se preferir somente rodar esse projeto para testes, siga os passos abaixo.

### 1 - Clonar ou fazer o download do Repositório

Rode o comando abaixo no Git Shell:

`git clone https://github.com/RARomano/HighTrustSharePointApps.git`

### 2 - Alterar o Web.Config

Você precisará abrir o Web.Config do projeto MVC e editar as linhas abaixo:

Coloque o **ClientID** e o **IssuerID** iguais ao Guid gerado para o IssuerID na etapa anterior. Caso não tenha feito a etapa anterior, só é necessário criar um guid novo e colocar nesses 2 campos.
Alterar o campo **ClientSigningCertificatePath** para o caminho do certificado gerado.
Alterar o campo **ClientSigningCertificatePassword** para a senha do certificado gerado.

```XML
    <add key="ClientId" value="59BC56D1-AA40-466D-B3C6-E920A78A5481" />
    <add key="ClientSigningCertificatePath" value="C:\certificados\intranet.pfx" />
    <add key="ClientSigningCertificatePassword" value="Pass@word1" />
    <add key="IssuerId" value="59BC56D1-AA40-466D-B3C6-E920A78A5481" />
```

#### Referências dos Scripts
Esses scripts foram extraídos do blog do [Andrew Connell](http://www.andrewconnell.com/blog/Fully-Scripted-Solution-for-Creating-and-Registering-Self-Signed-Certs-for-SP2013-High-Trust-Apps-with-S2S-Trust).
