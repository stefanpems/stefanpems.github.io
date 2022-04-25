---
title: "How to call Microsoft 365 Defender API from a Logic App"
last_modified_at: 2022-04-25T18:15:02-05:00
tags:
  - M365 Defender
  - M365D API
  - Logic Apps
---

Recently I needed to create a Logic App to execute periodically a specific Kusto query against the Advanced Hunting Tables of Microsoft Defender for Identity.

Logic App Standard and Power Automate Premium have a native connector for Microsoft Defender for Endpoint (MDE): https://docs.microsoft.com/en-us/connectors/wdatp/. It includes an Advanced Hunting action, allowing to run a Kusto query against the tables related to MDE. Adding that action to a Logic App workflow is strightforward: it simply asks to sign-in once, so that it creates the connection to the MDE service, and the KQL query to execute.
Unfortunately, it seems that you can't use this connector for querying tables not related to MDE. In my experience, you get an error:

-image-

