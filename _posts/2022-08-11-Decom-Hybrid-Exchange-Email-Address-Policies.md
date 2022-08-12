---
layout: article
title:  "Decomission Hybrid Exchange & Email Address Policies"
tags: [PowerShell, Microsoft Exchange, Hybrid Exchange]
---

Working on a project to decomission Hybrid Exchange this week, I have quickly found that certain functionality is simply not present in Exchange Online. Before I start this post, note that Microsoft do not recommend that you *uninstall* the last server from your environment due to the Exchange attributes being removed from the Active Directory schema. For those that are interested in removing Hybrid Exchange, check this [MS docs article](https://docs.microsoft.com/en-us/exchange/decommission-on-premises-exchange) out.

The functionality that was important to us are the onpremise email address policies for both contractors and our vendors that have accounts inside our tenant. Contractors are identified with an "X-" prefix on the email address and vendors, "V-" is added.

This came to be due to IR35, we make it abundantly clear that the email address someone is in contact with, is a contractor.

Email address policies were based on a few things:
* Based on the "Company" field in the AD user object.
* Based on the user's job title containing defining characters.
* Primary SMTP address is automatically updated and the old address added as an alias not to disrupt mailflow.

I have created a quick script that can be quickly adapted for use in other scenarios, I've used an array to define the company names - it's not the cleanest script in the world but it's functional and can be added to a scheduled task.  
Comments throughout the script should explain what it does.

*Contractor-Email-Address-Policies-V1.0.ps1:*

```powershell
<#    
    .SYNOPSIS  
    Automatically updates the Primary SMTP Address & Email Address fields for Company Name Contractors

    .DESCRIPTION
    This script has been designed to update two main attributes for an Active Directory User Object:
        - ProxyAddresses
        - EmailAddress
    
    This version of the script filters for an (X) in the Title field of a user profile and the company matching Company Name 1 & Company Name 2.

    .EXAMPLE
    Run the script as a standalone, there are no command line parameters available - ".\Contractor-Email-Address-Policies-V1.0.ps1"
    Note that the script is not signed so you will need to allow unsigned scripts to run on the local machine.

    .NOTES
        Author  - Tom Healy
        Version - 1.0
        Release Notes:
            - V1.0 - 09/08/2022 - Initial Creation
#>

# -- | Setup the array of company names | -- #
$CompanyList = @('Company Name 1',
    'Company Name 2')

# -- | Get all Contractors filtering them based on the tile | -- #
$Contractors = Get-ADUser -Filter 'Title -Like "*(X)*"' -Properties * 

# -- | Create blank variables for use in the Functions | -- #
$Forename = ""
$Surname = ""
$CurrentEmail = ""
$DesiredEmail = ""

# -- | Declare the functions | -- #
function Update-EmailAddress {
    if ($CurrentEmail -eq $DesiredEmail) {
        Write-Host "Email address matches for user:"  $Contractor.UserPrincipalName
    }
    else {
        Set-ADUser -Identity $Contractor -EmailAddress $DesiredEmail # Correct the Email Address
        $CurrentEmail = $DesiredEmail # Set the CurrentEmail variable to the newly set DesiredEmail 
    }
}

function Update-ProxyAddresses {
    Write-Host "Updating proxy addresses for user:" $Contractor.UserPrincipalName
    
    $ProxyArray = New-Object System.Collections.ArrayList # Create empty array for proxy addresses

    $PrimaryAddress = "SMTP:" + $DesiredEmail # Creates the desired SMTP address variable based on the DesiredEmail variable
    $ProxyArray.Add($PrimaryAddress) # Adds the primary SMTP address to the array as the first item

    # Gets the current uppercase SMTP ProxyAddress and places it into a variable
    $PrimarySMTPAddr = Get-ADUser -Identity $Contractor -Properties * | 
    Select-object EmailAddress, ProxyAddresses, @{name = 'PrimarySMTPAddress'; Expression = { $_.ProxyAddresses -cmatch '^SMTP:' } }

    # Add the current address to the array as lowercase
    $ProxyArray.Add($PrimarySMTPAddr.PrimarySMTPAddress.ToLower()) # Adds the old primary SMTP address as a lowercase string

    # Remove the current PrimarySMTP Address
    Set-ADUser -Identity $Contractor -Remove @{ProxyAddresses = $PrimarySMTPAddr.PrimarySMTPAddress } # Removes the Primary SMTP Proxy Address
    
    [string[]]$ProxyAddressString = ($ProxyArray) # Convert the array to a string to be able to add it using the Set-ADUser command
    Set-ADUser -Identity $Contractor -add @{ProxyAddresses = $ProxyAddressString } # Add the correct primary SMTP & alias SMTP addresses back to the user (does not touch x500, SIP etc.)
}

# -- | Loops through all Contractors in the list and performs the corrections in order | -- #
foreach ($Contractor in $Contractors) { 
    # -- | Check if the company attribute matches any in the array | -- #
    if ($Contractor.Company -in $CompanyList) {
        $Forename = $Contractor.GivenName
        $Surname = $Contractor.Surname
        
        $CurrentEmail = $Contractor.EmailAddress
        $DesiredEmail = "X-" + "$Forename" + "." + "$Surname" + "@example.com"
        
        # -- | Calls functions to update the Email Address & Proxy Addresses | -- #
        Update-EmailAddress
        Update-ProxyAddresses
    }
    else {
        Write-Host "Company name is missing from array for user with UPN:" $Contractor.UserPrincipalName
    }
}
```
GitHub Gist version [available here](https://gist.github.com/tomhhealy/45d0f5eaa7fa52ec61e1f98b60d20dea).

An honorable mention for Brad Wyatt over at The Lazy Administrator for a great article for an Exchange Online scenario where users have migrated to the cloud completely - [The Lazy Administrator post here](https://www.thelazyadministrator.com/2019/11/20/office-365-email-address-policies-with-azure-automation/#:~:text=Unfortunately%2C%20in%20Office%20365%20Exchange,or%20your%20company%20has%20implemented). Brad shows you how to use automation accounts and runbooks in Azure.