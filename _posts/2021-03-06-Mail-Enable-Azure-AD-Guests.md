---
layout: article
title:  "Mail Enable Azure AD Guests"
tags: [Microsoft Azure, Microsoft Exchange, Microsoft 365, PowerShell, Automation]
---

## Background
Mail-enabling a guest is as simple as logging into Exchange Online PowerShell, running a single command and voila, they appear in the GAL and allow you to email them like you would a regular user in the tenant.

However this being said, the customer has 700+ guests in the tenant from other companies and organizations. Slowly running an individual command to mail-enable guests and stay on-top of things is a longer task than first thought. When discussing this with the customer, I initially thought about having the users provide us with a CSV file that I could then import into a simple foreach loop and set each guest recursively.

## Requirements
Allow non-admin internal users to manage guests & corresponding attributes (job title, manager, location etc.)
Allow non-admin internal users to add/remove guests when they see fit.
Allow non-admin internal users to mail-enable guests when they see fit.

## Approach
The approach:
I proceeded to create an Administrative Unit inside of Azure AD to restrict the access I was giving to the non-admin users. Initially, an administrator would need to assign the users to this AU but in the long run this is more manageable than constantly importing CSV files.

Once this was done, I proceeded to create a group that would be used to mail enable any guests that are added to this AU – for this example I’ve setup a group called “Mail Enabled Guests”.

![Mail Enabled Guests](/assets/images/media/Mail-Enabled-Guests-Group.jpeg)

I like to try and automate things as much as possible but in this case, my thoughts were that I wanted to try and save as much time . The way I see it is if I have to constantly login to repeat a task over and over again, there has to be a way to automate it (or at least try!) – this is where Azure Automation & Runbooks come into the picture.

I set out by creating an Automation Account – in this example I use “AADGuest-Automation” as the name. From there I entered an administrator username/password that the run-book would run from – for this example only I use my own admin credentials as this is a lab tenant and it doesn’t concern me what account is used to run the script at this point.

![Mail Enabled Guests](/assets/images/media/AADGuests-Automation-Account.jpeg)

Now, onto the runbook!
This part is relatively simple as it’s only a few lines of code:

```powershell
# Sets the runbook to use the default 'Run As' connection in the Azure Automation Account (would be changed if live!)
$ConnectionName = "AzureRunAsConnection"
$SvcConnection = Get-AutomationConnection –Name $ConnectionName
 
# Specifies the credentials to use that I have stored in the automation account
$UserName = "Tom Healy - Admin"
$SvcUser = Get-AutomationPSCredential -Name $UserName
 
"INFO: Logging into Azure Active Directory..."Connect-AzureAD –TenantId $SvcConnection.TenantId `
    –ApplicationId $SvcConnection.ApplicationId `
    –CertificateThumbprint $SvcConnection.CertificateThumbprint
 
"INFO: Logging into Exchange Online..."
Connect-ExchangeOnline -Credential $SvcUser
 
# Pulls the group object ID into a variable
$GroupObjectId = (Get-AzureADGroup -SearchString 'Mail Enabled Guests').ObjectID.ToString()
 
#Pulls all members into a variable
$GroupMembers = Get-AzureADGroupMember -ObjectId $GroupObjectID | ? { $_.UserPrincipalName -match "\#EXT\#" }
 
# Sets each individual group member to be mail-enabled
$GroupMembers | % { Set-MailUser $_.Mail -HiddenFromAddressListsEnabled $false -wa SilentlyContinue }
 
# Reports how many group members there are
"Total guests mail enabled: " + $GroupMembers.Count
```
