---
title: "How to call Microsoft 365 Defender API from a Logic App"
last_modified_at: 2022-04-25T18:15:02-05:00
tags:
  - M365 Defender
  - M365D API
  - Logic Apps
---

Recently I needed to create an automation for executing periodically a specific Kusto query against the Advanced Hunting tables of Microsoft Defender for Identity (MDI). My objective was to identify the events of new users added to the Domain Admins group in Active Directory; more on that in this other blo post: [How to get notified for new users added as Domain Admins](https://stefanpems.github.io/How-to-get-notified-for-new-users-added-to-Domain-Admins/)

I realized the desired automation as an Azure Logic App that queries the Microsoft 365 Defender API. 

![full-logic-app](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/full-logic-app.png)

The JSON template of the full Logic App can be downloaded [here](https://github.com/stefanpems/m365defender/tree/main/Logic%20App). In the rest of this blog post, I explain the details of this implementation.

Logic App Standard and Power Automate Premium have a native connector for Microsoft Defender for Endpoint (MDE): [https://docs.microsoft.com/en-us/connectors/wdatp/](https://docs.microsoft.com/en-us/connectors/wdatp/). It includes an Advanced Hunting action, allowing to run a Kusto query against the tables related to MDE. Adding that action to a Logic App workflow is strightforward: it simply asks to sign-in once, so that it creates the connection to the MDE service API; then it is only necessary to specify the KQL query that should be executed.
Unfortunately, in my experience this connector doesn't allow to query tables not related to MDE. For example, this is the error that I got when I tried to use that action to query tables related to MDI:

![error-wdatp-connector](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/f7186a6c1fc3314362cd8aae60e5a31317fdac82/assets/2022-04-25-Logic%20App%20and%20M365DAPI/error-wdatp-connector.png)

The Microsoft 365 Defender APIs allow to query remotely the tables related to any Microsoft 365 Defender service included MDI. You can do a first test of these APIs on the same Microsoft 365 Defender portal: under the "Endpoint" section (...yes, this section is related to MDE but we are going to change the query), open "Partners and APIs" and then "API explorer". Here you can open the "Run Advanced Hunting query", change the URL of ths service (from "https://api-eu.securitycenter.windows.com/api/advancedhunting/run" to "https://api.security.microsoft.com/api/advancedhunting/run") and change the KQL to query the desired tables with the desired filters.

![query-in-m365d-api-explorer](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/ac148fbc909417253a863718df2b2efedbd06f01/assets/2022-04-25-Logic%20App%20and%20M365DAPI/query-in-m365d-api-explorer.png)

In order to execute the same KQL query from a Logic App, in the absence of a native connector to the M365 Defender API, it is necessary to separately manage the retrieval of a valid authorization token for the API service from Azure Active Directory (step 1) and the execution of the query by passing that token (step 2). In my lab environment, I created the steps 1 and 2 by using a standard HTTP connector.

Before creating the Logic App, it is necessary to register an App in Azure AD by following the steps documented in the paragraph "Register an app in Azure Active Directory" on this page: [https://docs.microsoft.com/en-us/microsoft-365/security/defender/api-hello-world?view=o365-worldwide#register-an-app-in-azure-active-directory](https://docs.microsoft.com/en-us/microsoft-365/security/defender/api-hello-world?view=o365-worldwide). Please note that, to ensure the right to call advanced hunting queries, it is necessary to assign also the permission (and admin consent) for "AdvancedHunting.Read.All" 

![permissions](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/permissions.png)

Before continuing, you need to ensure that you have:
* The Tenant ID
* The Application ID (a.k.a. as Client ID)
* The Secret ID

In the body sent to get the token, you needed also to specify these two strings:
* resource=https://api.security.microsoft.com
* grant_type=client_credentials

The first HTTP request is sent to get the token. It should look like this:

![get-token-request](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/get-token-req.png)

After its successful execution, you can copy the token and decode it in [https://jwt.ms/](https://jwt.ms/), to ensure that it includes the reference to "https://api.security.microsoft.com" (in "aud", the resource) and the correct permissions (in "roles"):

The token can be found here:
![get-token-response](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/get-token.res.png)

This is how jwt.ms decode its content:
![token-decoded](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/token-decoded.png)

The token just retrieved from the call to Azure Active Directory can be read in the Logic App by using the Parse JSON action and, then, can be used to call the Advanced Hunting API as shown below. Please note that the body of the HTTP call action contains the KQL used to query the MDI tables.

![call-advancedhunting](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/call-advanced-hunting-api.png)

The results of this call is finally parsed, again by using the Parse JSON action, so that the workflow can manage them with the desired logic.

My sample Logic App has been configured to start periodically, every hour. The Kusto exectued in the HTTP call to the M365 Defender API is a simplified version of the query described in this official blog post: [Track changes to sensitive groups with Advanced Hunting in Microsoft 365 Defender](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/track-changes-to-sensitive-groups-with-advanced-hunting-in/ba-p/3275198). The target is the MDI table "IdentityDirectoryEvents". As already clarified, the aim of the Logic App is to identify the users that have been added to the Domain Admins group since last execution of the query. 

If the query returns one or more users, the list of their names is sent by email to a specified set of recipients. In my sample Logic App, from the results of the query I extract only the name of the users and send that list of concatenated names in the body of the notification email. In your implementation you can add more details; for example, you can extract the time when the change to the Domain Admins membership has happened, the author of the change, the possible names of the groups added to the Domain Admins, etc.... Of course, you can also add some nice formatting.

![simple-email](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-04-25-Logic%20App%20and%20M365DAPI/simple-email.png)

As already mentioned, the JSON template of the full Logic App can be downloaded from [here](https://github.com/stefanpems/m365defender/tree/main/Logic%20App). You can then import it in your own Logic App, but you need first to replace these placeholders with their correct values in your environment:
* "insert-your-tenant-id"
* "insert-your-application-ID"
* "insert-your-application-secret" 
* "insert-your-desired-email-recipients"
* "insert-your-subscription-ID" (It's the Azure Subscription ID where you have your Logic App)
* "insert-your-resource-group-name" (It's the Azure Resource Group where you have your Logic App)

I hope that what I shared here can be useful for your needs of automated security controls.