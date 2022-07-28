---
layout: article
title:  "SCCM Audit Bloat in Co-Management Configuration"
tags: [Microsoft Intune, SCCM, Hybrid Management, Modern Workplace]
---

I stumbled upon an interesting issue that we had to confer with Microsoft on this week.  
The SCCM_Audit table inside of the ConfigMgr database, is repeatedly logging entries for the co-management configuration with Microsoft Intune. As you can imagine, this is not ideal and causes a massive bloat in the database. 

## Setting the scene...
The environment this was found on and relates to is as follows:
- PRIMARY filegroup consists of 4 x 6144 MB files.
- SCCM is running in a co-management configuration.
- Laptop builds, software deployment, driver deployment and management is all done via SCCM.
- SCCM is responsible for the management of 1400-1500 Windows 10 PCs.

## The issue...
We were repeatedly receiving alerts from our monitoring platform that the PRIMARY filegroup was full, SCCM was also not allowing us to deploy new applications or driver packages and as such, people were starting to complain.

I started to dig into it more. I was able to shrink the database files and regain about 100MB in space but this would then be instantly eaten up by the time I got round to shrinking the next file. I started to look at what table was the largest and found that the SCCM_Audit table was almost 2GB in size with no signs of slowing down so adding another .mdf file to the filegroup or expanding the current files in size, was not going to be a resolution.

Running a simple query and looking at the records, it was clear something wasn't right. The records were duplicate XML entries containing nothing exciting. While I won't post a record here in the event it links to something larger, nearly all entries were created by the Azure_Service.

## The resolution...

Working with Microsoft, the recommendation was to delete logs in this table manually with a delete statement in SQL. I have gone a step further here and created a SQL job that automatically clears the logs  every two weeks just to keep on top of it. Microsoft verbally recognised that this was a bug in SCCM and are looking to report this internally, however at this stage I am yet to find anything online about the bug being recognised or reported by anyone else.

A quick SQL query, just to delete logs from the SCCM_Audit table:
```sql
/* Change this out for your SQL database name */
USE SCCM_DB;

/* Setup a date variable and deduct 30 days from the current date/time */
DECLARE @RetentionDate datetime
DECLARE @CurrentDate datetime
SET @CurrentDate = getdate()
SET @RetentionDate = dateadd(day,-30,@CurrentDate)

/* Delete the data from the SCCM_Audit table */
DELETE FROM SCCM_Audit
WHERE ChangeTime <= @RetentionDate
```

I also found that when trying to run the SQL query above, I'd often find that the transaction log would fill before I was able to delete any data - as such, I had to delete records on a month-by-month basis:

```sql
USE SCCM_DB;

DELETE FROM SCCM_Audit
WHERE ChangeTime between '01-01-2022' and '01-02-2022'
```