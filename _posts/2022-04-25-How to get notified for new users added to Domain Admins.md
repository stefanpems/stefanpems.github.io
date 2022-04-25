---
title: "How to get notified for new users added as Domain Admins"
last_modified_at: 2022-04-25T22:00:02-05:00
tags:
  - M365 Defender
  - MDI
---

Recently, I read this interesting official blog post: [Track changes to sensitive groups with Advanced Hunting in Microsoft 365 Defender](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/track-changes-to-sensitive-groups-with-advanced-hunting-in/ba-p/3275198): it clearly explains how to query the Advanced Hunting table "IdentityDirectoryEvents" populated by the Microsoft Defender for Identity (MDI) sensor running on the Domain Controllers (DCs) of the on premises Active Directory (AD) infrastructure. 

I though that the proposed KQL code could be very simply used to get the security supervisors notified by email, for example, in the event of new users added to the Domain Admins group or to any other sensitive group in AD. Quite a useful monitoring functionality that could be achieved just as a side-capability of MDI. 

Just to be clear, MDI is an extremely important technology, allowing prevention, detection, investigation and response for protecting the on premises AD and the ADFS servers, the so called "Tier 0" of the IT security in most organizations. The capability described here is just a consequence of the availability of the data collected on the DCs.

My first thought was to simply create a Custom Detection Rule in Microoft 365 (M365) Defender, to be executed periodically (e.g., every 1 hour), with a KQL query almost identical to the one described in the above mentioned article. I did only 3 minimal modifications:
1. In order to simplify the first implementation, I limited the query to the Domain Admins group
2. I added the initial statement "where ingestion_time() > ago(1h)", to get only the data ingested in the last hour
3. I added the AccountSid and ReportId field as required by the Custom Detection Rules (see [here](https://docs.microsoft.com/en-us/microsoft-365/security/defender/custom-detection-rules))

The first execution was successful: as expected, it created an Incident and an Alert. 

[new-incident]()

I order to get notified by email, I had only to configure the "Notification" feature in the M365 Defender settings. 

I though that this could be enough but I was wrong: while doing a second test, I noticed that the new detections of that same Custom Detection Rules were added in the timeline of the already existing Alert. Because of that, no new Incident was created and, then, no new email notificationw was sent.

