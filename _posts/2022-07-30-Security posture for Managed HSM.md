---
title: "Security Posture for Azure Key Vault Managed HSM with Azure Policies"
last_modified_at: 2022-07-30T15:15:02-05:00
tags:
  - BYOK
  - Customer Managed Key
  - Key Vault
  - Managed HSM
  - Azure Monitor
  - Azure Policy
---

In my [previous blog post](https://stefanpems.github.io/Security-monitoring-Managed-HSM-with-Sentinel/) I described how to monitor, from a security perspective, the events occurring on an [Azure Key Vault Managed HSM (Hardware Security Module)](https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/overview) - in the rest of this post abbreviated as "MHSM" - by using [Microsoft Sentinel](https://docs.microsoft.com/en-us/azure/sentinel/overview). This security monitoring allows to identify suspicious activities and to react accordingly.

In this blog post I'll describe how to proactively monitor, always from a security perspective, the configuration of a MHSM in order to identify and possibly correct any deviation from the Microsoft recommendations and known security best practices.

Typically, in the Microsoft Azure cloud environment, this kind of "security posture" monitoring is done by [Microsoft Defender for Cloud](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction) - in the rest of this post abbreviated as "MDC": the [Cloud Security Posture Management](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/organize/cloud-security-posture-management), also here abbreviated as "CSPM", is a key MDC's feature. To do this continuous security assessment, MDC relies on the [Azure Policies](https://docs.microsoft.com/en-us/azure/governance/policy/overview); specifically, it uses the definition of the Azure "initiative" (group of policies) named [Azure Security Benchmark (ASB)](https://docs.microsoft.com/en-us/security/benchmark/azure/), which is "assigned" (with assignment name "ASC Default") at subscription or  management group level and then which covers any Azure resource. CSPM is a "free of charge" capability of MDC. MDC has also [Cloud Platform Workload Protection](https://docs.microsoft.com/en-us/azure/defender-for-cloud/workload-protections-dashboard) (CWPP) capabilities to protect against security threats different workloads running in hybrid and multi-cloud environment.  

Unfortunately, at the time of this writing (July 30, 2022), the MDC protection capabilities (CSPM and CWPP) do not cover yet MHSM instances, though they fully cover simple Azure Key Vault instances. I'm sure that the coverage extension to MHSM will be added very soon in the future. 

Today we can easily assess the known deviations of the MHSM configuration from the security best practices by directly relying on the Azure Policies. Different polices already exists with the words "Managed HSM" in their title, some of them also with the "Preview" tag in the title. 

**Recommendation:** To better visualize the images shown in this blog post, I recommend to right-click on them and open it in a new tab or window
{: .notice--warning}

![built-in-policies](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-07-30-Security%20posture%20for%20Managed%20HSM/built-in-policies.png)

In the above screenshot I'm highlighting the evidence that these polices are "built-in" in Azure. 

**Info Notice:** At the time of this writing, I could not find the existence of these policies mentioned in the official Microsoft documentation for the MHSM. Nevertheless, each of these policy has a very clear name and description.
{: .notice--info}

As already clarified, at the time of this writing these individual policies are not included in the Azure Security Benchmark initiative so they are not part of its default assignment: they must be assigned individually at the desired scope (subscription, resource group)

![policy-assignment](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-07-30-Security%20posture%20for%20Managed%20HSM/policy-assignment.png)

Additional "custom policies" can be created to further protect MHSMs. 

When all policies are assigned at the desired scope, it is possible to have a view of their effect by using the "Compliance" page of the Azure Policy blade.

![compliance-status](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-07-30-Security%20posture%20for%20Managed%20HSM/compliance-status.png)

For example, this the detailed view of a policy with a non-compliance...:

![non-compliant-policy](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-07-30-Security%20posture%20for%20Managed%20HSM/non-compliant-policy.png)

...while this the detailed view of a policy with full compliance:

![compliant-policy](https://raw.githubusercontent.com/stefanpems/stefanpems.github.io/master/assets/2022-07-30-Security%20posture%20for%20Managed%20HSM/compliant-policy.png)

As always, I hope that the information provided in this blog post may be useful. 

PS. Feedback on this blog post can be added in this related [LinkedIn post](https://www.linkedin.com/posts/stefanopescosolido_security-posture-for-azure-key-vault-managed-activity-6959154673442283520--2qe?utm_source=linkedin_share&utm_medium=member_desktop_web)

A final note. When I started investigating on how to assess the security configuration of MHSMs I was not aware of the existence of the (yet undocumented) built-in policies in Azure. Because of that, I started by cloning the  policies existing for simple Key Vault instances. For the sake of completeness, this is how I did it: open the "Azure Security Benchmark" initiative, search for the included polices with the word "Key Vault" in their title, duplicate them, replacing the word "vaults" with the word "managedhsms" on their JSON definitions, and assign them individually to the desired scope including the MHSM(s) to be protected. Only a few of the individual policies created for simple Key Vault instances could not be applied to MHSM (e.g., those referring to the protection of "secrets", which do not exist in MHSMs). 
{: .notice--info}