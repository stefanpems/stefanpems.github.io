---
title: "How to call Microsoft 365 Defender API from a Logic App"
last_modified_at: 2022-04-25T18:15:02-05:00
tags:
  - M365 Defender
  - M365D API
  - Logic Apps
---

Recently I needed to create a Logic App to execute periodically a specific Kusto query against the Advanced Hunting tables of Microsoft Defender for Identity (MDI).

Logic App Standard and Power Automate Premium have a native connector for Microsoft Defender for Endpoint (MDE): [https://docs.microsoft.com/en-us/connectors/wdatp/](https://docs.microsoft.com/en-us/connectors/wdatp/). It includes an Advanced Hunting action, allowing to run a Kusto query against the tables related to MDE. Adding that action to a Logic App workflow is strightforward: it simply asks to sign-in once, so that it creates the connection to the MDE service API, and the KQL query to execute.
Unfortunately, it seems that you can't use this connector for querying tables not related to MDE: in my experience, you get an error. For example, this is what I get when trying to query tables related to MDI:

![error-failed-to-resolve-table](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/f7186a6c1fc3314362cd8aae60e5a31317fdac82/assets/2022-04-25-Logic%20App%20and%20M365DAPI/error-mdi-table-in-mde-query.png)

In order to query, from a Logic App, the tables related to any Microsoft 365 Defender service included MDI, it is possible to use the Microsoft 365 Defender API. You can do a first test of these APIs on the Microsoft 365 Defender portal: under the "Endpoint" section, open "Partners and APIs" and then "API explorer". Here you can open any sample query, change the URL of ths service (from "https://api-eu.securitycenter.windows.com/api/advancedhunting/run" to "https://api.security.microsoft.com/api/advancedhunting/run") and change the KQL to query the desired tables.

-img




