---
layout: article
title:  "Intune HWID Upload to Azure Storage Account"
tags: [Autopilot, Intune, Microsoft Azure, PowerShell]
---

I have found myself in a situation where I have no record of HWIDs for existing Autopilot machines. The challenge I now face is that these devices are going to be moved to another tenant and there’s a lot of them.

To save the administrative overhead of having to install the [Get-WindowsAutoPilotInfo.ps1](https://www.powershellgallery.com/packages/Get-WindowsAutoPilotInfo/3.5) script, running it, gathering the HWID and then uploading it to Intune, I have put together a script that can just sit running for a week or so gathering the HWID’s of existing devices managed by MDM, ready for me to move them.

A copy of the script can be found below or on my [GitHub Gist page](https://gist.github.com/tomhhealy/a6e9a087baa566486fe6c3829725dc05):


```powershell
using namespace System.Net
# ------ Export CSV with HWID information ------ #
$Session = New-CimSession
$DeviceDetails = (Get-CimInstance -CimSession $Session -Namespace root/cimv2/mdm/dmmap -Class MDM_DevDetail_Ext01 -Filter "InstanceID='Ext' AND ParentID='./DevDetail'") # Get the HWID from the local machine
 
# Get the device information into variables
$Serial = (Get-CimInstance -CimSession $Session -Class Win32_BIOS).SerialNumber
$Product = ""
$HWID = $DeviceDetails.DeviceHardwareData
 
# Creates the array then creates a custom object to export in the required format
$DeviceDetails = @()
$DeviceDetails += [pscustomobject]@{
    "Device Serial Number" = $Serial
    "Windows Product ID" = $Product
    "Hardware Hash" = $HWID
}
 
# Create the C:\HWID directory if it does not exist
$ExportPath = "C:\HWID"
if(Test-Path $ExportPath) { Write-Verbose "Folder C:\HWID exists..." }
else { New-Item $ExportPath -ItemType Directory }
 
$ExportFile = $Serial+".csv"
$DeviceDetails | Export-CSV -Path "C:\HWID\$ExportFile" -NoClobber -NoTypeInformation
Remove-CimSession $Session
 
 
# ------ Upload HWID CSV to Azure Storage Account via REST API ------ #
$headers = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers.Add("x-ms-blob-type", "BlockBlob")
$headers.Add("Content-Type", "text/csv")
$uri = "https://yourstorageaccount.blob.core.windows.net/hwids/"+$ExportFile+"?sv=2020-08-04&ss=bf&srt=o&sp=wacitfx&se=2022-12-31T00:00:00Z&st=2022-05-09T18:15:35Z&spr=https&sig=kQyeTBqv3oX34J8yHdLiWuzZEoBZFV9z"
$response = Invoke-RestMethod -Uri $uri -Method 'PUT' -Headers $headers -InFile "C:\HWID\$ExportFile"
$response | ConvertTo-Json
```

Note that the above omits my SAS key and container name that I have created. If you want to try this out, create a storage account by clicking the below button & then deploy the script via Intune to the required device group. Note that you will also be required to setup a SAS key – see [MS Docs article](https://docs.microsoft.com/en-us/azure/cognitive-services/translator/document-translation/create-sas-tokens?tabs=Containers)


[![Deploy to Azure](/assets/images/Deploy-To-Azure.svg)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2Fquickstarts%2Fmicrosoft.storage%2Fstorage-account-create%2Fazuredeploy.json)