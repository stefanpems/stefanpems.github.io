---
title: "How to get notified for new users added as Domain Admins"
last_modified_at: 2022-04-27T00:00:00-00:00
tags:
  - M365 Defender
  - MDI
---

Recently, I read this interesting official blog post: [Track changes to sensitive groups with Advanced Hunting in Microsoft 365 Defender](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/track-changes-to-sensitive-groups-with-advanced-hunting-in/ba-p/3275198): it clearly explains how to query the Advanced Hunting table "IdentityDirectoryEvents" populated by the Microsoft Defender for Identity (MDI) sensor running on the Domain Controllers (DCs) of the on premises Active Directory (AD) infrastructure. 

I though that the proposed KQL code could be very simply used to get the security supervisors notified by email, for example, in the event of new users added to the Domain Admins group or to any other sensitive group in AD. Quite a useful monitoring functionality that could be achieved just as a side-capability of MDI. 

Just to be clear, [MDI](https://docs.microsoft.com/en-us/defender-for-identity/what-is) is an extremely important technology, allowing prevention, detection, investigation and response for protecting the on premises AD infrastructue and the ADFS servers, the so called "Tier 0" of the IT security in most organizations. The capability described here is just a consequence of the availability of the data collected on the DCs.

My first thought was to simply create a Custom Detection Rule in Microsoft 365 (M365) Defender, to be executed periodically (e.g., every 1 hour), with a KQL query almost identical to the one described in the above mentioned article. I did only 3 minimal modifications:
1. In order to simplify the first implementation, I limited the query to the Domain Admins group
2. I added the initial statement "where ingestion_time() > ago(1h)", to get only the data ingested in the last hour
3. I added the AccountSid and ReportId field as required by the Custom Detection Rules (see [here](https://docs.microsoft.com/en-us/microsoft-365/security/defender/custom-detection-rules))

The first execution was successful: as expected, it created an Incident and an Alert. 

![new-incident](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-How%20to%20get%20notified%20for%20new%20users%20added%20to%20Domain%20Admins/new-incident.png)

In order to get notified by email, I though I had only to configure the "Notification" feature in the M365 Defender settings and my objective was achieved. 

Well, I was wrong: while doing a second test, I noticed that the new detections of that same Custom Detection Rules were added in the timeline of the already existing Alert instead of creating a new Incident, as I was expecting. Because of that, no new email notification was sent.

![alert-timeline](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-How%20to%20get%20notified%20for%20new%20users%20added%20to%20Domain%20Admins/alert-timeline.png)

In my experience, new Incidents are created only if the Custom Dection Rule identifies new results after some days from the last detection.

I'm still investigating if the Custom Detection Rules can be configured to ensure that any new execution returning one or more results creates a new Incidents. If I'll find a way, I'll update this blog post. Apparently this is not possible, and it seems to be "by design": one of the important objectives of M365 Defender is to reduce the "alert fatigue" for the security analysts and this objective is pursuit also by minimizing the number of Incidents related to similar events. 

I also noticed a second, very important evidence: the events collected by the MDI sensor arrive on the Advanced Hunting tables in M365 Defender with a variable and possibly substantial delay. In my experience, the detection of a change in a group membership (event with "ActionType"="Group Membership changed") arrives on the Advanced Hunting table "IdentityDirectoryEvents" with a delay varying from a few minutes to even more than a couple of hours. Because of that, I understood that the notification logic that I was trying to implement through MDI should not be considered a "near real-time detection"; it is, instead, a periodical supervision/revision on what happened on the membership of this sensitive group. Note: the situation is different when implementing different soltions; more on that at the end of this post.

With these two evidences in mind, I started investigating how to obtain the desired behavior. 

In my lab, like many of my customers, I have also [Microsoft Sentinel](https://docs.microsoft.com/en-us/azure/sentinel/overview) and its [M365 Defender connector](https://docs.microsoft.com/en-us/azure/sentinel/connect-microsoft-365-defender?tabs=MDI) already configured to import MDI related tables. 

![data-connector](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-How%20to%20get%20notified%20for%20new%20users%20added%20to%20Domain%20Admins/data-connector.png)

In these conditions, it is quite easy to create a Scheduled Analytic Rules in Sentinel, doing periodically the same KQL query that I was trying to implement through a Custom Detection Rule. It is also very easy to add a Playbook to send a notification email with the evidences of the query results; it's possible to start from the existing "Send basic email" [Playbook Template](https://docs.microsoft.com/en-us/azure/sentinel/automate-responses-with-playbooks#playbook-templates).

![analytic-rule-query](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-How%20to%20get%20notified%20for%20new%20users%20added%20to%20Domain%20Admins/analytic-rule-query.png)

![analytic-rule-playbook](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-How%20to%20get%20notified%20for%20new%20users%20added%20to%20Domain%20Admins/analytic-rule-playbook.png)

This solution could be adopted only by the customers who have already decided to use the SIEM and SOAR capabilities available in Microsoft Sentinel and who are importing the data from MDI. Specifically, those customers will benefit from the many built-in Analytic Rules and Hunting queries available in Sentinel for the detection of suspected identity abuse. Note: a much better solution based on Sentinel/Azure Monitor has been added at the end of this post.

But, how to obtain the desired email notifications for those customers not using Sentinel or not importing MDI data in Sentinel? 

This is possible by using Logic Apps and the M365 Defender API. The solution is fully described in this other dedicated post in my blog: [How to call Microsoft 365 Defender API from a Logic App](https://stefanpems.github.io/Logic-App-and-M365DAPI/). The source code is [here](https://github.com/stefanpems/m365defender/tree/main/Logic%20App). Basically, the Logic App makes two HTTP calls: the first one is a POST to Azure AD to obtain a JWT token, the second one is a POST to the M365 Advanced Hunting API to execute the desired KQL query. The possible results are sent by email. The execution of the Logic App is scheduled periodically. 

![full-logic-app](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/full-logic-app.png)

I have not yet tested but I'm quite confident that the solution can be easily adapted to Power Automate for those customers who do not have an Azure subscription where to run a Logic App.

**Update - Apr. 27, 2022** - For those customers who are already ingesting in Log Analytics and Microsoft Sentinel the Security events from the Active Directory domain controllers, the easiest and most effective way to be notified about the occurrence of new users added to the Domain Admins group is by periodically searching for the event ID 4728 in the SecurityEvent table in Log Analytics. This search can be easily automated through the creation of a simple Alert in Azure Monitor, but in this case there is no flexibility in how the email can be formatted. An alternative (my favorite one) is to create a simple Logic App configured with a periodic execution, using the native Azure Monitor connection and the related "Run query and list results" action followed by the "Create HTML table" action and, finally, by the "Send an email" action. A third alternative is by creating and Automation Rule for having an alert in Sentinel and a Playbook triggered by this alert. A last but very important point: in my tests, this solution, based on the monitoring of the event 4728 on the SecurityEvent table has the very welcome charactetistic to be extremely fast and reliable in terms of delays: the collection latency for the Azure Monitor Agent [is minimal](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/data-ingestion-time#agent-collection-latency), so the event 4728 appears in the SecurityEvent table almost immediately after the change done on the group membership for the Domain Admins group. In these conditions, it is also possible to build a near-real time execution logic based on playbooks.

I hope this helps.