---
title: "Security Monitoring Azure Key Vault Managed HSM with Microsoft Sentinel"
last_modified_at: 2022-05-22T23:15:02-05:00
tags:
  - BYOK
  - Customer Managed Key
  - Key Vault
  - Managed HSM
  - Azure Monitor
  - Sentinel
---

[Azure Key Vault Managed HSM (Hardware Security Module)](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/overview) - in the rest of this post abbreviated as MHSM - is a fully managed, highly available, single-tenant, standards-compliant cloud service that enables customers to safeguard cryptographic keys for their cloud applications, using FIPS 140-2 Level 3 validated HSMs and with a customer-controlled [security domain](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/security-domain).

The deployment of MHSMs is ideal for highly regulated "[Bring Your Own Key](https://docs.microsoft.com/en-us/azure/key-vault/keys/byok-specification)" (BYOK) scenarios or when customers want to fully control the ["Root of Trust"](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/mhsm-control-data) of their cloud-based key management solution. Several types of Microsoft Azure services already allow to encrypt "at rest" customers' data by using a "[Customer Managed Key](https://docs.microsoft.com/en-us/azure/security/fundamentals/encryption-models#supporting-services)" (CMK) stored in a MHSM; this capability will be soon available for the wide majority of Microsoft cloud services.

MHSM is clearly a critical component in a customer's cloud deployment, requiring the higher level of protection both in terms of security posture ([configuration best practices](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/best-practices)) and security monitoring. 

Today it's mandatory to protect IT deployments guided by a "[Zero Trust](https://docs.microsoft.com/en-us/azure/security/fundamentals/zero-trust)" mentality; its "Assume Breach" principle imposes to continuosly monitor the infrastructure, use analytics to get visibility, drive threat detection, and improve defenses. [Microsoft Sentinel](https://docs.microsoft.com/en-us/azure/sentinel/overview) is a cloud-based [SIEM](https://www.gartner.com/en/information-technology/glossary/security-information-and-event-management-siem) and [SOAR](https://www.gartner.com/en/information-technology/glossary/security-orchestration-automation-response-soar) allowing to effectively accomplish these objectives.    

The following figure shows some sample [incidents](https://docs.microsoft.com/en-us/azure/sentinel/investigate-cases) created by Sentinel because of some suspicious activities simulated on a MHSM and automatically detected by Sentinel.

**Recommendation:** To better visualize the images shown in this blog post, I recommend to right-click on them and open it in a new tab or window
{: .notice--warning}

![sentinelalerts](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/sentinelalerts.png)

In order to protect an existing MHSM with Sentinel, it is necessary to enable its audit logging in the [Azure Log Analytics](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview) workspace where Sentinel is enabled (Sentinel is basically a "solution" deployed in a Log Analytic workspace). This configuration can be done by using the following Azure CLI command. The execution of this command can be done in the [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) or in a [local Azure CLI shell installation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (latest version recommended).

```html
{% raw %}
hsmresource=$(az keyvault show --hsm-name "<MHSM-NAME>" --query id -o tsv)

az monitor diagnostic-settings create --name "<NEW-DIAGNOSTIC-SETTINGS-NAME>" --resource $hsmresource --logs '[{"category": "AuditEvent","enabled": true}]' --workspace "/subscriptions/<SUBSCRIPTION-ID>/resourcegroups/<SENTINEL-RESOURCE-GROUP-ID>/providers/microsoft.operationalinsights/workspaces/<SENTINEL-WORKSPACE-NAME>"
{% endraw %}
```

![enable audit loggin in Log Analytics](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/diag-audit.png)

NOTE: at the time of this writing (June 18, 2022), the official MHSM documentation describes how to [enable MHSM logging](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/logging) but only to a Azure Blob Storage. My expectation is that the documentation will be completed soon with the just described command, required to enable the logging on Azure Log Analytics and, then, in Sentinel.
{: .notice}

Just a few minutes after having executed the command described above, it is possible to verify that log data of type "AuditEvent" generated by the MHSM is being logged in the AzureDiagnostic table of the selected Log Analytics workspace. The content of this table can be visualized by accessing the "Logs" page either in the Azure Log Analytics blade or, equivalently, in the Microsoft Sentinel blade in the Azure Portal. The table rows related to the "AuditEvent" generated by the MHSM can be identified by filtering by *ResourceProvider* == "*MICROSOFT.KEYVAULT*" and *ResourceType* == "*MANAGEDHSMS*". 

```html
{% raw %}
AzureDiagnostics 
| where ResourceProvider == "MICROSOFT.KEYVAULT" and ResourceType == "MANAGEDHSMS"
{% endraw %}
```

To ensure that at least an "AuditEvent" entry is logged, it is enough to visualize the properties of the MHSM object by executing this simple Az command in the CLI:

```html
{% raw %}
az keyvault show --hsm-name "<MHSM-NAME>"
{% endraw %}
```

![sample-query](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/sample-query.png)

At this point, we have all the "AuditEvent" log data produced by the MHSM flowing into the Log Analytics workspace. To enable the Sentinel's threat detection capabilities we have to configure and enable adequate [Analytic Rules](https://docs.microsoft.com/en-us/azure/sentinel/work-with-anomaly-rules). These rules are based on [Kusto Query Language (KQL)](https://docs.microsoft.com/en-us/azure/sentinel/kusto-overview) queries written to identify anomalies or suspected events within the mass of logs ingested by Sentinel. Any security analyst can create custom queries at any point in time. It is convenient, however, to quickly deploy the initial threat detection capability by leveraging the queries made available by Microsoft and/or by the large world wide community of security analysts using Sentinel (the main repository of this community is [https://github.com/azure/azure-sentinel](https://github.com/azure/azure-sentinel)).

Microsoft has released in the Sentinel's [Content Hub](https://docs.microsoft.com/en-us/azure/sentinel/sentinel-solutions-catalog) a specific "Solution Package" for protecting Azure Key Vaults. At the time of this writing, this solution contains 1 Data Connector and 4 Analytic Rules: 

![solution](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/solution.png)

If not already installed, by selecting the solution it is possible to see an "Install" button. After its installation, the solution appears as "Installed" and it shows the "Manage" and "Actions" buttons as in the image above. 

After having installed the solution, the Data Connector will be automatically activated (status = "*Connected*") by the enablement of the MHSM's audit logging in the Sentinel's Log Analytic workspace, as described above.

![data-connector](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/data-connector.png)

The 4 Analytic Rules included in the solution must be enabled one by one. They realize their threat detection logic through KQL queries written for simple Azure Key Vault instances. It is possible to modify these KQL queries - indifferently at the solution's deployment time or post-deployment - to ensure that they analyze the log data written by MHSM instances instead of by Azure Key Vaults instances. The modification consists in a simple "Replace All" text editing action of the string "*VAULTS*" with the string "*MANAGEDHSMS*" (the recommendation is to include the double quotes as shown in the figure below and, possibly, to select the options "case sensitive" and "whole words" offered by the "Replace All" command).

![kql-editing](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/kql-editing.png)

The "Replace All" text editing action shown in the figure above must be repeated for each of the 4 Analytic Rules included in the solution.

The following figure shows all the 4 Analytic Rules in the status "Enabled".

![enabled-rules](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/enabled-rules.png)

At this point, Sentinel has the audit data coming from MHSM and some initial threat detection logic enabled to create incidents when the "suspicious" or "sensitive" events searched by the 4 Analytic Rules are found within the set of data ingested in the Log Analytic workspace. 

To test this threat detection capability I recommend to:
* [Test 1]. Create and delete a key. The "KeyDelete" operation is one of the sensitive operations searched by the Analytic Rule named "Sensitive Azure Key Vault operations". 
* [Test 2]. Create a key and access its properties repeatedly for more than 25 times. The "KeyGet" is one of the operation monitored by the Analytic Rule named "Mass secret retrieval from Azure Key Vault" and the number 25 is the default threshold value considered by this rule to trigger an incident. 
* [Test 3]. Temporarly add your PC's public IP (the public IP that is seen by Microsoft when you access the Azure Portal) in the Sentinel's Threat Intelligence page. This will trigger the Alert Rule named "TI map IP entity to Azure Key Vault logs"

NOTE: the first two of the three mentioned Analytic Rules have a frequency of 1 day: this means that you should expect their triggered incidents appearing in Sentinel after that amount of time. The "TI map IP entity to Azure Key Vault logs" rule, instead, has a frequency of 1 hour: you should expect its incident appearing much faster.  
{: .notice}

For the sake of completeness, I present here below the commands that can be used for creating a key (of type RSA-HSM 2048 bits), accessing its  properties and, then, deleting it as required by the tests 1, 2 and 3.

```html
{% raw %}
az keyvault key create --hsm-name "<MHSM-NAME>" --kty RSA-HSM --size 2048 --name "<NEW-KEY-NAME>"
az keyvault key show --hsm-name "<MHSM-NAME>" --name "<KEY-NAME>"
az keyvault key delete --hsm-name "<MHSM-NAME>" --name "<KEY-NAME>"
{% endraw %}
```

(To execute the test 2, the "az keyvault key show" command must be executed repeatedly for a number of times above the threshold of the Analytic Rule that by default is 25) 

The key management operations require adequate "data plane" local RBAC permissions for the logged on user. More on that here:
* [MHSM Built-in Roles](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/built-in-roles)
* [MHSM Role Management](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/role-management)

The following figure shows the execution of the key creation command, after the verification of the correct CLI session context (correct tenant and subscription):

![key-creation](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/key-creation.png)

Please be aware that if your Active Directory tenant includes Control Access Policies requiring MFA and Device Compliance (which, by the way, is the optimal, recommended configuration!), you won't be able to execute key management operations by using the Azure Cloud Shell, neither when invoked by a the local [Windows Terminal MS Store App](ms-windows-store://pdp/?ProductId=9n0dx20hk701): you'll be blocked by the Conditional Access Policy. To go through these operations you must use the [local Azure CLI shell installation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (latest version recommended).

The 3 incidents generated by the 3 tests described above are shown in the first figure of this blog post. In the figure here below you can see the details of one of these incidents. 

![incident-details](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-06-18-Security%20monitoring%20Managed%20HSM%20with%20Sentinel/incident-details.png)

The Analytic Rules described and tested in this blog post enable the threat detection capability from the audit logs generated by MHSM. Security analysts can further improve this capability by writing their own queries with additional detection logic for proactive hunting or reactive investigations. They can also leverage the SOAR functinalities available in Sentinel to add automations for managing the suspicious events related to MHSM. For example: 
* incidents can be automatically enriched with additional evidences read from Open Source Threat Intelligence (OSINT) providers or paid TI feeds;
* incidents can be automatically notified to relevant teams through different channels like email, Teams, etc...
* incidents can trigger the automatic creation and assignment of tickets in IT Service Management tools like ServiceNow, etc...
* threat containement actions can be executed automatically (e.g., IP blocked at firewall level, ...)

A final note: at the time of this writing (June 18, 2022), the "Key Vault" protection plan in Microsoft Defender for Cloud does not cover MHSM instances. I believe that this feature will be added quite soon in the future. For the time being, the Sentinel's threat detection capability is the only one available to protect MHSM instances. It's always important to monitor IT resources with a SIEM like Sentinel; for MHSM is even more critical. 
{: .notice}

As always, I hope that the information provided in this blog post may be useful. 

PS. Feedback on this blog post can be added in this related [LinkedIn post](https://www.linkedin.com/posts/stefanopescosolido_security-monitoring-azure-key-vault-managed-activity-6943979665271214080-4mOO?utm_source=linkedin_share&utm_medium=member_desktop_web)