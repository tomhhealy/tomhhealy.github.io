---
layout: article
title:  "Windows 10 ODJ Devices - Stuck at User ESP"
tags: [Windows Autopilot, Microsoft Intune, Microsoft 365]
---

I’ve found that Windows 10 Offline Domain Join devices that do not have direct line of sight to the Domain Controller during the enrolment stage often get stuck at the “Account setup” stage when trying to “Join your organisation’s network” like below:

![Mail Enabled Guests](/assets/images/media/ODJ-User-ESP-Stuck.png)

Obviously this poses a challenge when trying to deploy hundreds of devices across an estate to remote workers or in Regus-type office where they do not have line of sight to a domain controller. While we have deployed pre-logon VPNs with Autopilot that allow users to sign in so the domain is available at the point of login, this does not allow you to sign in while the device is configuring. Microsoft have provided [a list](https://docs.microsoft.com/en-us/mem/autopilot/windows-autopilot-hybrid#supported-byo-vpns) of BYO VPN clients:

- In-box Windows VPN client
- Cisco AnyConnect
- Pulse Secure
- GlobalProtect
- Checkpoint
- Citrix NetScaler
- SonicWall

The setting that you need to deploy to skip this stage can be found below:
- **OMA URI Name** – SkipUserStatusPage
- **Description** – Skips the Account setup ESP step for Autopilot devices
- **OMA URI:**
```
./Vendor/MSFT/DMClient/Provider/MS DM Server/FirstSyncStatus/SkipUserStatusPage
```
- **Data Type** - Boolean
- **Value** - True

While the user will not be forced to wait for user-targeted configuration to complete, it still deploys once the user signs into the device such as apps targeted at users and so on. This wasn’t a concern for the project I was working on as configuration was targeted at devices and not users or user groups.