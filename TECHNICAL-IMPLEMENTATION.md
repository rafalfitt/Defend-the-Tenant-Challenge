# Technical Implementation Plan (GUI)

**Companion to [HARDENING-PLAN.md](HARDENING-PLAN.md). Scenario: 100 users, two administrators, no SOC, Microsoft 365 E7.**

This document describes how two administrators configure each of the five recommended controls through the admin portals. It is a Phase 2 proposal for a fictional customer. It contains no tenant identifiers, secrets or production data. Portal labels change over time; confirm each path in the tenant before use, and check that the required licenses and roles are present.

## 1. Principles

- Every change follows the same cycle: **check impact, pilot, enforce, monitor**.
- Every new Conditional Access (CA) policy starts in **Report-only**, is reviewed in the sign-in logs, and is switched to **On** only after the gate is met.
- Two administrators never change the same policy at the same time. One implements, the other reviews (four-eyes) and records the change in the change log.
- Emergency access accounts stay outside every blocking policy and are tested before enforcement.
- Rollback for every CA policy: set **Enable policy** to **Off** (or back to **Report-only**). Record the previous state before editing.

## 2. Portals and roles (least privilege)

| Portal | URL | Used for | Minimum role for the task |
|---|---|---|---|
| Microsoft Entra admin center | entra.microsoft.com | Authentication methods, Conditional Access, PIM, access reviews, enterprise apps, ID Protection | Conditional Access Administrator; Authentication Policy Administrator; Privileged Role Administrator; Global Administrator for admin consent workflow settings |
| Microsoft Intune admin center | intune.microsoft.com | Device compliance, Defender for Endpoint integration, Windows Hello for Business policies | Intune Administrator; Endpoint Security Manager for the Defender for Endpoint connection |
| Microsoft Defender portal | security.microsoft.com | Preset security policies, attack disruption prerequisites, incident notifications | Security Administrator (Manage security settings permission) |
| Microsoft 365 admin center | admin.microsoft.com | Integrated apps, agent registry | Role with access to the Agents pages; AI Administrator for agent management actions and the agent registry Graph API |
| Azure portal | portal.azure.com | Azure Monitor alert rule for emergency account sign-ins | Monitoring Contributor |

Use PIM activation (Control 2) for these roles once Control 2 is complete. Until then, sign in with the standard administrative accounts.

## 3. Step 0: Preparation (before any enforcement)

### 3.1 Create security groups

Entra admin center > **Entra ID** > **Groups** > **All groups** > **New group** (type: Security, assigned membership).

| Group | Purpose |
|---|---|
| `sg-emergency-access` | The two emergency accounts; excluded from Conditional Access policies that block or restrict sign-in |
| `sg-pilot-passkeys` | Pilot users for phishing-resistant MFA |
| `sg-priority-accounts` | Executives and finance (Strict preset, stage 2 of passkey rollout) |
| `sg-pilot-compliant-device` | Pilot users for the compliant-device policy |
| `sg-ca-exception-device-code` | Documented exceptions for device code flow (starts empty) |
| `sg-ca-exception-compliant-device` | Documented exceptions for the compliant-device policy (starts empty) |
| `sg-security-alerts` | Membership list of both administrators (for example for Azure Monitor action groups and ownership records). Defender notification rules use explicit mailbox addresses, not this group |

Every exception group member needs a recorded owner, reason and expiry date in the change log.

### 3.2 Create two emergency access accounts

These steps follow Microsoft's guidance on [managing emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access).

1. Entra admin center > **Entra ID** > **Users** > **New user** > **Create new user**.
2. Create two cloud-only accounts on the `*.onmicrosoft.com` domain. They must not be federated or synchronized from on-premises, and must not be associated with an individual user or an employee-supplied device.
3. Assign **Global Administrator** as an **active, permanent** assignment in PIM (**Entra ID** > **Roles & admins** > **Global Administrator** > **Add assignments**). Do not make these accounts PIM-eligible; they must work when activation or approval is not possible.
4. Choose a passwordless method that satisfies the mandatory MFA requirement: **Passkey (FIDO2)** (recommended) or certificate-based authentication if a PKI already exists. Use a method different from the one the normal administrator accounts use (for example, if administrators use Microsoft Authenticator, use FIDO2 security keys for the emergency accounts).
5. Register the credential for each account. The credential or device must not expire and must not be in scope of automated cleanup for lack of use.
6. Add both accounts to `sg-emergency-access`. Exclude this group from every Conditional Access policy that blocks or restricts sign-in. Report-only policies do not block access and do not require an exclusion.
7. Store the credentials in secure, fireproof safes in separate locations. Authorized individuals use a designated secure workstation or Privileged Access Workstation when signing in with these accounts.
8. Test sign-in and an administrative task for both accounts and record the date. Validate at least every 90 days, after IT staff changes, and when subscriptions change. Confirm that no multifactor authentication or self-service password reset method is registered to an individual's device or personal details.

### 3.3 Alert on emergency account use (Azure Monitor)

This procedure follows the [Monitor sign-in and audit logs](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access#monitor-sign-in-and-audit-logs) section of Microsoft's guidance. Microsoft Sentinel or another tool can be used instead.

**Prerequisite:** send Microsoft Entra sign-in logs to a Log Analytics workspace (Entra admin center > **Monitoring & health** > **Diagnostic settings** > **Add diagnostic setting** > select **SignInLogs** (and **AuditLogs**) > **Send to Log Analytics workspace**). Workspace ingestion and alert rules can incur Azure charges; estimate before enabling.

1. Entra admin center > **Entra ID** > **Users** > open each emergency account > copy the **Object ID**.
2. Azure portal (at least Monitoring Contributor) > **Monitor** > **Alerts** > **+ Create** > **Alert rule**.
3. **Scope** tab: select the Log Analytics workspace that receives the sign-in logs > **Apply**.
4. **Condition** tab: **Signal name** > **Custom log search**; **Query type**: **Aggregated logs**; **Search query**:

   ```kusto
   SigninLogs
   | where UserId == "<object-id-account-1>" or UserId == "<object-id-account-2>"
   | project TimeGenerated, UserPrincipalName, UserId, IPAddress, ResultType, ResultDescription
   ```

5. **Measurement**: choose the measure, aggregation type and granularity. Under **Split by dimensions**, select the **Resource ID column**. **Alert logic**: threshold type **Static**, operator **Greater than**, threshold value **0**, frequency of evaluation as often as the administrators want the query to run.
6. **Actions** tab: select or create an action group (**Notification type**: **Email/SMS message/Push/Voice**; add both administrators; use a display name of at most 12 characters).
7. **Details** tab: **Severity**: **0 - Critical**; name the rule; select the region and identity used to run the query; under **Advanced options** select **Enable upon creation**.
8. **Review + create** > **Create**.
9. Run a drill: sign in with an emergency account during a planned test, confirm both administrators receive the alert, and record the date.

**Post-mortem:** if the alert fires, preserve the Entra and workload logs and review whether the use was a planned drill, a real emergency, or misuse. Examine the actions taken by the person who used the account.

The object IDs are tenant identifiers. Do not publish them in this repository.

### 3.4 Check MFA registration coverage

Entra admin center > **Entra ID** > **Authentication methods** > **User registration details**. Filter for users that are not MFA-capable. The Phase 1 script verifies configuration, not every account, so this report is the evidence for gate 1.

## 4. Control 1: Phishing-resistant MFA and passkeys

**Gate:** 100% of in-scope accounts MFA-capable and registered with an approved phishing-resistant method before the policy for that group is switched on.

### 4.1 Enable the methods

1. Entra admin center > **Entra ID** > **Authentication methods** > **Policies**.
2. Select **Passkey (FIDO2)** > **Enable** > target: **Select groups** > `sg-pilot-passkeys` (later add `sg-priority-accounts`, then all users). Save.
3. Select **Temporary Access Pass** > **Enable** > target the same groups. Set the lifetime to the shortest value that supports onboarding (for example 1 hour, one-time use). Save.
4. Windows Hello for Business: configure through Intune (Intune admin center > **Devices** > **Enroll devices** > **Windows enrollment** > **Windows Hello for Business**).
5. Optional: **Authentication methods** > **Registration campaign** to prompt users to register a passkey.

### 4.2 Register methods for the pilot

Administrators issue a Temporary Access Pass (Entra ID > **Users** > select user > **Authentication methods** > **Add authentication method** > **Temporary Access Pass**). The user then registers a passkey at `https://aka.ms/mysecurityinfo`.

### 4.3 Conditional Access policy: admins first

Entra admin center > **Entra ID** > **Conditional Access** > **Policies** > **New policy**.

| Setting | Value |
|---|---|
| Name | `CA-ADMIN-PhishResistantMFA` |
| Users > Include | **Select users and groups** > **Directory roles** > the administrative roles in use (at minimum Global Administrator, Application Administrator, Authentication Administrator, Billing Administrator, Cloud Application Administrator, Conditional Access Administrator, Exchange Administrator, Helpdesk Administrator, Intune Administrator, Password Administrator, Privileged Authentication Administrator, Privileged Role Administrator, Security Administrator, SharePoint Administrator, User Administrator) |
| Users > Exclude | `sg-emergency-access` |
| Target resources | **All resources** |
| Grant | **Grant access** > **Require authentication strength** > **Phishing-resistant MFA strength** |
| Enable policy | **Report-only** |

Review in **Entra ID** > **Monitoring & health** > **Sign-in logs** > open an event > **Report-only** tab. When no legitimate administrator sign-in would fail, switch to **On**.

Conditional Access directory-role targeting applies to built-in roles, not custom roles or administrative-unit-scoped role assignments. Use an appropriately scoped user or group assignment for administrators holding those roles.

### 4.4 Stage 2 and 3 policies

Clone the policy twice:

- `CA-PRIORITY-PhishResistantMFA`: include `sg-priority-accounts`.
- `CA-ALLUSERS-PhishResistantMFA`: include **All users**; exclude `sg-emergency-access`; start in Report-only only after at least 95% of users hold a registered passkey or Windows Hello credential (the remainder must be explained).

**Tradeoff and mitigation:** enrollment effort and lost-device recovery. Mitigation: staged groups, Temporary Access Pass, and a written recovery procedure (helpdesk identity verification before issuing a pass).

## 5. Control 2: Privileged access with PIM and access reviews

**Gate:** no standing admin role assignments except the two emergency accounts.

### 5.1 Configure role settings

Entra admin center > **Entra ID** > **Identity Governance** > **Privileged Identity Management** > **Microsoft Entra roles** > **Roles** (or **Settings**) > select a role > **Edit**.

| Setting | Recommended value |
|---|---|
| Activation maximum duration | 1 hour for Global Administrator; 4 hours for other roles |
| On activation, require | **Azure MFA**, or **Microsoft Entra Conditional Access authentication context** (see 5.2) |
| Require justification on activation | Yes |
| Require ticket information | Optional; use the change log reference |
| Require approval to activate | Global Administrator only; approver is the other administrator. Microsoft recommends at least two approvers. With only two administrators, document the single-approver availability risk and how activation will be handled if that approver is unavailable. Never leave the approver list empty, and never make every Global Administrator and Privileged Role Administrator assignment eligible with approval required without an active emergency account (see 3.2) |
| Notifications | Send to both administrators |
| Allow permanent eligible assignment | No; set expiration (for example 12 months) |
| Allow permanent active assignment | No (emergency accounts are assigned directly, outside PIM settings) |

Keep the two emergency accounts as permanent active Global Administrators through direct assignment. The permanent-active restriction is for routine assignments; verify that both emergency assignments remain active after configuring PIM and during each access review. Emergency accounts are the recovery path, not routine PIM approvers.

### 5.2 Optional: stronger activation

1. **Conditional Access** > **Authentication contexts** > **New authentication context** (name: `PIM-Activation`).
2. Create a CA policy targeting that authentication context, include the administrators, grant **Require authentication strength** > **Phishing-resistant MFA strength**.
3. In the PIM role settings, set **On activation, require** to **Microsoft Entra Conditional Access authentication context** and select `PIM-Activation`.

### 5.3 Convert permanent assignments to eligible

1. PIM > **Microsoft Entra roles** > **Assignments** > **Active assignments**: list every permanent assignment.
2. For each non-emergency administrator: **Add assignments** > role > member > **Assignment type: Eligible** > set start and end dates.
3. Test activation with each administrator account in a new browser profile.
4. Only after successful activation, remove the permanent active assignment (**Active assignments** > **Remove**). Keep the emergency accounts as permanent active.
5. Verify that at most two to four people can activate Global Administrator, and that each role has the fewest assignees possible.

### 5.4 Recurring access review

- **Privileged roles:** Entra admin center > **ID Governance** > **Privileged Identity Management** > **Microsoft Entra roles** > **Access reviews** > **New**.
- **Groups and applications:** Entra admin center > **ID Governance** > **Access reviews** > **New access review** (used in Control 4 for application assignments and in 3.1 for exception groups).
- **Reviewers complete assigned reviews** at [myaccess.microsoft.com](https://myaccess.microsoft.com/) > **Access reviews**.

| Setting | Value |
|---|---|
| Scope | Privileged roles (all, or Global Administrator and Privileged Role Administrator first) |
| Reviewers | The other administrator (administrators do not approve their own access) |
| Frequency | Quarterly |
| Duration | 14 days |
| If reviewers do not respond | **No change** for the first two cycles; evaluate stricter handling after that |
| Require reason | Yes |

**Tradeoff:** one extra activation step per privileged task. Mitigation: documented activation rules and pre-planned maintenance windows.

## 6. Control 3: Session and token protection

**Gate:** report-only results show no unexplained impact; no support spike during the pilot.

Enforcement chain: Defender for Endpoint machine risk, then Intune compliance status, then the Conditional Access policy that requires a compliant device. The compliance policy sets the status; only the enabled Conditional Access policy restricts resource access.

### 6.1 Defender for Endpoint and Intune integration (prerequisites)

Roles: Security Administrator (or the Manage security settings permission) in the Defender portal, and Endpoint Security Manager (or equivalent Mobile Threat Defense rights) in Intune. See [Configure Defender for Endpoint with Intune](https://learn.microsoft.com/en-us/intune/device-security/microsoft-defender/configure-integration).

1. Defender portal > **System** > **Settings** > **Endpoints** > **General** > **Advanced features**: set **Intune connection** to **On** > **Save preferences**.
2. Intune admin center > **Endpoint security** > **Defender for Endpoint**: confirm **Connection status** is **Enabled** (it can take up to 15 minutes to update).
3. Under **Compliance policy evaluation**, set **Connect Windows devices to Defender for Endpoint** to **On** > **Save**. Enable the iOS/iPadOS and Android switches only if those platforms access sensitive apps.
4. Onboard the in-scope Windows devices: Intune admin center > **Endpoint security** > **Endpoint detection and response** > **EDR Onboarding Status** > **Deploy preconfigured policy** (all Windows devices), or **Create policy** for a custom scope with **Auto from connector** as the client configuration package type.
5. Verify onboarding: devices appear in the Defender portal under **Endpoints** > **Device inventory**, and **EDR Onboarding Status** shows successful onboarding. Verify risk reporting before enforcing access.

The service-to-service connection, the platform compliance-evaluation switch and device onboarding are separate steps; enabling the connector alone does not complete the deployment.

### 6.2 Intune compliance policy

Intune admin center > **Devices** > **Compliance** > **Policies** > **Create policy** > **Windows 10 and later**.

| Area | Recommended setting |
|---|---|
| Device Health | Require BitLocker; require Secure Boot; require code integrity |
| Device Properties | Minimum OS version aligned to the supported Windows 11 release |
| System Security | Require firewall; require antivirus and anti-spyware; require password to unlock |
| Microsoft Defender for Endpoint | **Require the device to be at or under the machine risk score**: **Medium** (pilot value; see the threshold table below) |
| Actions for noncompliance | Mark noncompliant immediately; send email at day 1; retire after a documented period |

Assign to the device or user groups. Create equivalent policies for iOS/iPadOS and Android only if those platforms access sensitive apps.

**Machine risk threshold.** The threshold is inclusive: the device may be at the selected level or below it. It evaluates Defender for Endpoint machine risk, not Entra user risk or sign-in risk. Passing this setting does not override other compliance requirements.

| Threshold | Machine risk levels that pass | Levels that fail |
|---|---|---|
| Clear | Clear | Low, Medium, High |
| Low | Clear, Low | Medium, High |
| Medium | Clear, Low, Medium | High |
| High | All levels | None based on risk alone |

Microsoft's integration guide recommends **Low** as the best balance of security and productivity for most organizations. This plan starts the pilot at **Medium**, which blocks only high-risk devices, and tightens to **Low** after the report-only review shows acceptable impact. Record the chosen threshold and the reason in the change log; moving to Low is a security-policy change, not a wording change.

### 6.3 Conditional Access: compliant device (pilot first)

| Setting | Value |
|---|---|
| Name | `CA-SENSITIVE-CompliantDevice` |
| Users > Include | `sg-pilot-compliant-device` (later all users) |
| Users > Exclude | `sg-emergency-access`, `sg-ca-exception-compliant-device` |
| Target resources | Select resources: Office 365 (Exchange Online, SharePoint Online, Teams) and any application classified as sensitive |
| Grant | **Require device to be marked as compliant** |
| Enable policy | **Report-only** |

Wait at least 24 hours for sign-in data to accumulate, then review the **Report-only** results in the sign-in logs. Plan for personal devices: either enroll them, give access through a browser-only policy with session controls, or place the user in the documented exception group with an expiry date.

### 6.4 Conditional Access: block device code flow

1. **Conditional Access** > **Policies** > **New policy**; name `CA-ALL-BlockDeviceCodeFlow`.
2. **Users**: include **All users**; exclude `sg-emergency-access` and `sg-ca-exception-device-code`.
3. **Target resources** > **Include**: **All resources**.
4. **Conditions** > **Authentication flows** > **Configure: Yes** > select **Device code flow** > **Done**.
5. **Grant** > **Block access**.
6. **Enable policy**: **Report-only** > **Create**.

Before enforcement, run **Sign-in logs** filtered by **Authentication protocol: Device code** over at least 14 days. Typical legitimate users are Azure CLI or PowerShell device logins, conference-room devices and some line-of-business tools. Move each legitimate case to the exception group with an owner and expiry date, or constrain it with a separate allow-style policy (for example by network location). Then set the policy to **On**.

**Limits:** these policies reduce exposure; they do not invalidate every token already stolen. Revoke sessions for a suspected compromised user with **Users** > select user > **Revoke sessions**.

## 7. Control 4: Governed apps and AI agents

**Gate:** every enterprise application and agent has a named owner and recorded purpose; zero unreviewed grants.

### 7.1 Verify user consent

Entra admin center > **Entra ID** > **Enterprise apps** > **Consent and permissions** > **User consent settings**: confirm **Do not allow user consent**. This setting does not remove existing grants.

### 7.2 Enable the admin consent workflow

1. Sign in as Global Administrator (activate through PIM).
2. **Entra ID** > **Enterprise apps** > **Consent and permissions** > **Admin consent settings**.
3. **Users can request admin consent to apps they are unable to consent to**: **Yes**.
4. **Who can review admin consent requests**: add both administrators (a reviewer group is easier to maintain).
5. Enable email notifications and expiration reminders; set **Consent request expires after (days)** to 7.
6. **Save**. It can take up to an hour to apply.

Note: only Global Administrators can approve requests for Microsoft Graph application permissions; reviewers can view, block or deny.

### 7.3 Inventory existing applications and grants

1. **Enterprise apps** > **All applications**; export the list (**Download (Export)**).
2. For each application: **Permissions** (admin and user consent grants), **Users and groups**, **Sign-in logs**, **Owners**.
3. Remove unused applications and revoke excessive grants (**Permissions** > **Review permissions**).
4. Assign at least one owner to every retained application (**Owners** > **Add**).
5. Record in a register (SharePoint list): application, owner, business purpose, permissions, date reviewed.
6. Optional: create a quarterly access review of application assignments (**ID Governance** > **Access reviews** > **New access review** > applications).

### 7.4 AI agents and integrated apps

- Microsoft 365 admin center > **Settings** > **Integrated apps**: review deployed Copilot extensions and agents; remove unused ones and restrict deployment to named users.
- Microsoft 365 admin center > **Agents** > **All Agents** > **Registry**: review the agent inventory, identify agents without owners (the **Agents without owners** card filters the list), and export the list (**Export**, CSV) for the governance register. Review each retained agent's owner, business purpose, permissions and last review date. **Agents** > **Overview** is the summary dashboard, not the inventory.
- Agent risk signals in the registry require an E7 or Agent 365 license (**Billing** > **Licenses** > **Subscriptions** shows the subscriptions). The updated risk-details pane is rolling out gradually, so the interface may differ by region. Counts can lag the security portals by up to an hour.
- Entries can appear more than once with different publisher types (for example **Shared by creator** and **Published by your org**). This is by design; do not treat it as an inventory error. Removing one entry does not remove the other.
- Approve a short pre-approved list of vendors to reduce approval delay.

See [Manage agent registry in Microsoft 365 admin center](https://learn.microsoft.com/en-us/microsoft-agent-365/admin/agent-registry-overview).

### 7.5 Operating the workflow

Target approver service level: 2 working days. Reviewer checks: publisher verification, requested delegated and application permissions, offline access, redirect URIs, business owner and purpose, and whether a narrower permission exists.

## 8. Control 5: Detection and response for two administrators

**Gate:** both administrators receive a test alert; weekly triage is recorded.

### 8.1 Preset security policies

Defender portal > **Email & collaboration** > **Policies & rules** > **Threat policies** > **Preset security policies**.

1. **Standard protection** > **Manage protection settings**; for **Exchange Online Protection** and **Defender for Office 365 protection**, select **All recipients**. Keep the preset values.
2. **Strict protection** > scope to **Specific recipients**: `sg-priority-accounts`.
3. On the impersonation step, add executives and finance approvers as protected users.
4. Order of precedence is Strict, then Standard, then custom policies; custom policies do not override a preset.

Verify Safe Links behavior (Threat policies > **Safe Links** > view the policy applied by the preset): URL scanning of email, time-of-click checks, and protection in Teams and Office apps.

**QR-code phishing:** there is no separate administrator-facing toggle. Microsoft extracts URLs from QR-code images and evaluates them with the other message signals (sender intelligence, headers, content, reputation) in the existing threat-detection pipeline. Defender for Office 365 Plan 1 and Plan 2 can also detonate extracted URLs. The presence of a QR code alone does not determine the verdict. Administrative work is limited to reviewing the protection policies and mail-flow exceptions (for example transport rules or allow entries that bypass filtering), and submitting missed detections and false positives for analysis. See the [Microsoft Tech Community article on QR-code phishing](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/protect-your-organizations-against-qr-code-phishing-with-defender-for-office-365/4007041).

### 8.2 Automatic attack disruption prerequisites

Attack disruption is built in when the licensing and Defender products are deployed; there is no single master switch.

1. Defender portal > **System** > **Settings** > **Endpoints** > **Device groups**: set the **Remediation level** to **Full - remediate threats automatically**.
2. **Settings** > **Endpoints** > **Device discovery**: set to **Standard discovery** (required for the automatic Contain device action).
3. Confirm that Defender for Endpoint, Defender for Office 365, Defender for Identity and Defender for Cloud Apps are deployed to the extent the customer uses them; coverage is limited to the deployed workloads.
4. Review the **Action center** (**Actions & submissions**) weekly for pending and completed actions.

### 8.3 Alert routing to both administrators

**Defender XDR incident notifications.** Defender portal > **Settings** > **Microsoft Defender XDR** > **Email notifications** (under **General**) > **Incidents** tab > **Add incident notification rule**.

1. **Basics**: enter a rule name and description > **Next**.
2. **Notification settings**: set **Alert severity** to **High** (optionally also **Medium**); set **Device group scope** to all device groups unless a narrower scope is needed. Optionally select **Send only one notification per incident**.
3. **Recipients**: type the email address of each administrator and select **Add** after each address. Do not enter `sg-security-alerts`; the wizard requires email addresses. Select **Send test email** and confirm delivery to both administrators.
4. **Review rule** > **Create rule**.

Required permission: **Manage security settings**. With basic permissions management, the Security Administrator role is sufficient; Global Administrator is not needed. This is the cross-workload incident notification; do not confuse it with the endpoint alert notifications under **Settings** > **Endpoints** > **General** > **Email notifications**, which use **Add item**. See [Get incident notifications by email](https://learn.microsoft.com/en-us/defender-xdr/m365d-notifications-incidents).

**Entra ID Protection.** Entra admin center > **Entra ID** > **ID Protection** > **Notifications**: enable users-at-risk email alerts and the weekly digest, recipients both administrators.

Record the date of the successful test email and of the Azure Monitor drill (3.3).

## 9. Rollout schedule

| Week | Activities | Gate |
|---|---|---|
| 1 | Section 3 (groups, emergency accounts, Azure Monitor alert, registration report); enable FIDO2 and TAP for admins; admin passkeys | All administrators hold a passkey; emergency accounts tested; alert drill passed |
| 2 | `CA-ADMIN-PhishResistantMFA` report-only then On; PIM role settings; convert roles to eligible | No standing admin access except emergency accounts |
| 3 | Defender-Intune integration and device onboarding; Intune compliance policy; device-code policy in report-only; `CA-SENSITIVE-CompliantDevice` report-only for pilot | Devices onboarded and reporting risk; report-only reviewed; exceptions documented |
| 4 | Preset security policies; attack disruption prerequisites; incident notification rule; admin consent workflow; app and agent inventory | Test email received by both administrators |
| 5-6 | Enforce device code policy and compliant-device pilot; extend passkeys to priority accounts | No unexplained support spike |
| 7-12 | Passkeys for all users in stages; first access review; first monthly review | Review log kept; actions closed |

Adjust the schedule to the customer's change windows. Do not run two enforcement changes on the same day.

## 10. Operating rhythm

| Frequency | Task | Where | Time |
|---|---|---|---|
| Weekly | Triage incidents and risky users; check Action center | Defender portal; Entra ID Protection | About 15 minutes |
| Monthly | Review PIM activations, CA changes (audit logs), consent and agent requests, success measures, exception group expiry | Entra admin center | About 2 hours |
| Quarterly (at most every 90 days) | Access review of admin roles; app and agent register review against the Agent Registry; emergency account drill and alert test; recovery procedure test | Entra admin center; M365 admin center; Azure Monitor | Half day |

Times are planning estimates, not measured results.

## 11. Validation checklist

- [ ] Both emergency accounts tested, use a passwordless method different from the administrator accounts, and are excluded from blocking policies.
- [ ] Azure Monitor alert for emergency account sign-ins fired during a drill and reached both administrators.
- [ ] Registration report shows 100% MFA-capable for in-scope accounts.
- [ ] Admins can sign in only with phishing-resistant methods.
- [ ] Every admin role is eligible; PIM activation tested by both administrators.
- [ ] Defender for Endpoint connection enabled, Windows devices onboarded, machine risk threshold recorded.
- [ ] Report-only results reviewed and documented before each enforcement.
- [ ] Device-code exceptions have an owner and expiry date.
- [ ] Every enterprise app and agent has an owner and recorded purpose; no agents without owners remain in the registry.
- [ ] Preset security policies assigned; incident notification test email delivered to both administrators.
- [ ] Change log records who changed what, when, and the rollback decision.

## 12. Rollback reference

| Change | Rollback |
|---|---|
| Any CA policy | Set **Enable policy** to **Off** or **Report-only**; sign in with an emergency account if both administrators are locked out |
| PIM conversion | Re-add a permanent active assignment through an emergency account |
| Passkey method | Remove the group from the method target; issue a Temporary Access Pass for recovery |
| Preset policy | Remove the group scope under Preset security policies |
| Machine risk threshold | Raise the threshold in the compliance policy (for example Low back to Medium) and record the reason |

## 13. Sources

- [Manage emergency access accounts in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
- [Manage agent registry in Microsoft 365 admin center](https://learn.microsoft.com/en-us/microsoft-agent-365/admin/agent-registry-overview)
- [Get incident notifications by email (Defender XDR)](https://learn.microsoft.com/en-us/defender-xdr/m365d-notifications-incidents)
- [Configure Microsoft Defender for Endpoint with Intune and onboard devices](https://learn.microsoft.com/en-us/intune/device-security/microsoft-defender/configure-integration)
- [Protect your organization against QR code phishing with Defender for Office 365](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/protect-your-organizations-against-qr-code-phishing-with-defender-for-office-365/4007041)
- [Perform an access review (myaccess.microsoft.com)](https://learn.microsoft.com/en-us/entra/id-governance/perform-access-review)
- [Require phishing-resistant MFA for administrators](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-admin-phish-resistant-mfa)
- [Block authentication flows with Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-authentication-flows)
- [Conditional Access: authentication flows](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows)
- [Assign Microsoft Entra roles in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user)
- [Plan a PIM deployment](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-deployment-plan)
- [Configure the admin consent workflow](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow)
- [Configure how users consent to applications](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)
- [Safe Links overview](https://learn.microsoft.com/en-us/defender-office-365/safe-links-about)
- [Configure automatic attack disruption](https://learn.microsoft.com/en-us/defender-xdr/configure-attack-disruption)
