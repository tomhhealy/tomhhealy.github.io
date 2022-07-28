---
layout: article
title:  "Migrating from WordPress to Jekyll"
tags: [WordPress, Jekyll, Microsoft Azure, Blogging]
---

I have recently gone through a bit of a change in my homelab. Namely shutting down both the QNAP NAS and my Dell R720 servers.  
The UK hit 40°C last week, almost unbearable in the UK and this old kit was working overtime and I've since shut it all down as I found I wasn't really using it.

One of the bigger changes from my homelab was how I hosted my previous blog. I was using WordPress, but found that there were things that I didn't like and wanted to change.  
I have since moved over to Jekyll hosted in GitHub Pages and have found that it's quit good at what it does. Having a static site generated each time I commit changes to a repo is quite nice - I know WordPress has versioning when it comes to the pages but catastrophic damage can be done to a site very quickly.

I thought I'd write a quick post on my reasons for moving to Jekyll over WordPress and what my setup was vs what it is now.  

## WordPress Setup
I used to run WordPress as an App Instance in Azure, this has since changed. It was relatively cost efficient, but got pricey quite quickly if you started doing more than just hosting static content.  

Primarily I used the blog as a learning experience. It allowed me to learn a number of different things during the time it was in Azure:
- App Services (App Service Plans, App Services etc.) - I'll generalise for this point.
- Networking with App Serivces, Private Endpoints and SQL - this one was interesting.
- Basics of MySQL, migrating databases from MySQL Server to a [Flexible MySQL Server](https://docs.microsoft.com/en-us/azure/mysql/flexible-server/) (primarily to save cost)

I'll touch on some sections I found interesting below;

### VNET Integration with Web App Services
This was an integral part of 