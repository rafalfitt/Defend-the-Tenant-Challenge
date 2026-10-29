# Technical Implementation Plan (GUI)

**Companion to [HARDENING-PLAN.md](HARDENING-PLAN.md). Scenario: 100 users, two administrators, no SOC, Microsoft 365 E7.**

This document describes how two administrators configure each of the five recommended controls through the admin portals. It is a Phase 2 proposal for a fictional customer. It contains no tenant identifiers, secrets or production data. Portal labels change over time; where a menu path was not verified against current documentation, it is marked **(verify)**.

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
| Microsoft Intune admin center | intune.microsoft.com | Device compliance, Windows Hello for Business policies | Intune Administrator |
| Microsoft Defender portal | security.microsoft.com | Preset security policies, attack disruption prerequisites, alert notifications | Security Administrator |
| Microsoft 365 admin center | admin.microsoft.com | Integrated apps, agent inventory **(verify)** | Global Reader for review; Global Administrator only when changing settings |

Use PIM activation (Control 2) for these roles once Control 2 is complete. Until then, sign in with the standard administrative accounts.

## 3. Step 0: Preparation (before any enforcement)

### 3.1 Create security groups

Entra admin center > **Entra ID** > **Groups** > **All groups** > **New group** (type: Security, assigned membership).

| Group | Purpose |
|---|---|
| `sg-emergency-access` | The two emergency accounts; excluded from blocking CA policies |
| `sg-pilot-passkeys` | Pilot users for phishing-resistant MFA |
| `sg-priority-accounts` | Executives and finance (Strict preset, stage 2 of passkey rollout) |
| `sg-pilot-compliant-device` | Pilot users for the compliant-device policy |
| `sg-ca-exception-device-code` | Documented exceptions for device code flow (starts empty) |
| `sg-ca-exception-compliant-device` | Documented exceptions for the compliant-device policy (starts empty) |
| `sg-security-alerts` | Both administrators; alert recipients |

Every exception group member needs a recorded owner, reason and expiry date in the change log.

### 3.2 Create two emergency access accounts

1. Entra admin center > **Entra ID** > **Users** > **New user** > **Create new user**.
2. Create two cloud-only accounts on the `*.onmicrosoft.com` domain, with non-descriptive names, not tied to a person.
3. Set a long random password for each; store the credentials in two separate physical locations.
4. Assign **Global Administrator** as an **active, permanent** assignment (**Entra ID** > **Roles & admins** > **Global Administrator** > **Add assignments**). Do not make these accounts PIM-eligible; they must work when activation or approval is not possible.
5. Register a FIDO2 security key for each account (phishing-resistant authentication). Store keys separately.
6. Add both accounts to `sg-emergency-access`.
7. Test sign-in for both accounts and record the date. Retest at each quarterly review.

### 3.3 Alert on emergency account use

Defender portal > **Hunting** > **Advanced hunting** > create a query on sign-in events for the two account UPNs (table `EntraIdSignInEvents` **(verify table availability)**) > **Create detection rule** > frequency: every hour > actions: generate alert, severity High. Add both administrators as recipients through the notification rule in Control 5.

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
| Users > Include | **Select users and groups** > **Directory roles** > the administrative roles in use (at minimum Global Administrator, Privileged Role Administrator, Security Administrator, Conditional Access Administrator, Intune Administrator, Exchange Administrator, SharePoint Administrator, User Administrator) |
| Users > Exclude | `sg-emergency-access` |
| Target resources | **All resources** |
| Grant | **Grant access** > **Require authentication strength** > **Phishing-resistant MFA strength** |
| Enable policy | **Report-only** |

Review in **Entra ID** > **Monitoring & health** > **Sign-in logs** > open an event > **Report-only** tab. When no legitimate administrator sign-in would fail, switch to **On**.

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
| Require approval to activate | Global Administrator only; approver is the other administrator |
| Notifications | Send to both administrators |
| Allow permanent eligible assignment | No; set expiration (for example 12 months) |
| Allow permanent active assignment | No (emergency accounts are assigned directly, outside PIM settings) |

### 5.2 Optional: stronger activation

1. **Conditional Access** > **Authentication contexts** > **New authentication context** (name: `PIM-Activation`).
2. Create a CA policy targeting that authentication context, include the administrators, grant **Require authentication strength** > **Phishing-resistant MFA strength**.
3. In the PIM role settings, set **On activation, require** to **Microsoft Entra Conditional Access authentication context** and select `PIM-Activation`.

### 5.3 Convert permanent assignments to eligible

1. PIM > **Microsoft Entra roles** > **Assignments** > **Active assignments**: list every permanent assignment.
2. For each administrator: **Add assignments** > role > member > **Assignment type: Eligible** > set start and end dates.
3. Test activation with each administrator account in a new browser profile.
4. Only after successful activation, remove the permanent active assignment (**Active assignments** > **Remove**). Keep the emergency accounts as permanent active.
5. Verify that at most two to four people can activate Global Administrator, and that each role has the fewest assignees possible.

### 5.4 Recurring access review

PIM > **Microsoft Entra roles** > **Access reviews** > **New** (or Entra ID > **Identity Governance** > **Access reviews** > **New access review** **(verify path)**).

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

### 6.1 Intune compliance policy (prerequisite)

Intune admin center > **Devices** > **Compliance** > **Create policy** > **Windows 10 and later**.

| Area | Recommended setting |
|---|---|
| Device Health | Require BitLocker; require Secure Boot; require code integrity |
| Device Properties | Minimum OS version aligned to the supported Windows 11 release |
| System Security | Require firewall; require antivirus and anti-spyware; require password to unlock |
| Microsoft Defender for Endpoint | Maximum allowed machine risk score: Medium **(verify requires Defender for Endpoint connection)** |
| Actions for noncompliance | Mark noncompliant immediately; send email at day 1; retire after a documented period |

Assign to the device or user groups. Create equivalent policies for iOS/iPadOS and Android only if those platforms access sensitive apps.

### 6.2 Conditional Access: compliant device (pilot first)

| Setting | Value |
|---|---|
| Name | `CA-SENSITIVE-CompliantDevice` |
| Users > Include | `sg-pilot-compliant-device` (later all users) |
| Users > Exclude | `sg-emergency-access`, `sg-ca-exception-compliant-device` |
| Target resources | Select resources: Office 365 (Exchange Online, SharePoint Online, Teams) and any application classified as sensitive |
| Grant | **Require device to be marked as compliant** |
| Enable policy | **Report-only** |

Plan for personal devices: either enroll them, give access through a browser-only policy with session controls, or place the user in the documented exception group with an expiry date.

### 6.3 Conditional Access: block device code flow

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

### 7.4 AI agents and integrated apps

- Microsoft 365 admin center > **Settings** > **Integrated apps**: review deployed Copilot extensions and agents; remove unused ones and restrict deployment to named users.
- Review agents in the agent inventory provided by the licensed Agent 365 / Microsoft Entra Agent ID capabilities **(verify current portal path and capabilities in the tenant; this document does not assume a menu path)**.
- Apply the same register fields: owner, purpose, permissions, last review date.
- Approve a short pre-approved list of vendors to reduce approval delay.

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

Verify Safe Links behavior (Threat policies > **Safe Links** > view the policy applied by the preset): URL scanning of email, time-of-click checks, and protection in Teams and Office apps. QR-code phishing detection is part of Defender for Office 365 protection; no separate toggle was verified for this document **(verify)**.

### 8.2 Automatic attack disruption prerequisites

Attack disruption is built in when the licensing and Defender products are deployed; there is no single master switch.

1. Defender portal > **System** > **Settings** > **Endpoints** > **Device groups**: set the **Remediation level** to **Full - remediate threats automatically**.
2. **Settings** > **Endpoints** > **Device discovery**: set to **Standard discovery** (required for the automatic Contain device action).
3. Confirm that Defender for Endpoint, Defender for Office 365, Defender for Identity and Defender for Cloud Apps are deployed to the extent the customer uses them; coverage is limited to the deployed workloads.
4. Review the **Action center** (**Actions & submissions**) weekly for pending and completed actions.

### 8.3 Alert routing to both administrators

- Defender portal > **Settings** > **Microsoft Defender XDR** > **Email notifications** > **Incidents** > **Add item**: severity **High** (optionally Medium), recipients `sg-security-alerts` or both mailboxes **(verify menu label)**.
- Entra admin center > **Entra ID** > **ID Protection** > **Notifications**: enable users-at-risk email alerts and the weekly digest, recipients both administrators.
- Send a test alert or review a recent incident to prove delivery. Record the date.

## 9. Rollout schedule

| Week | Activities | Gate |
|---|---|---|
| 1 | Section 3 (groups, emergency accounts, registration report); enable FIDO2 and TAP for admins; admin passkeys | All administrators hold a passkey; emergency accounts tested |
| 2 | `CA-ADMIN-PhishResistantMFA` report-only then On; PIM role settings; convert roles to eligible | No standing admin access except emergency accounts |
| 3 | Intune compliance policy; device-code policy in report-only; `CA-SENSITIVE-CompliantDevice` report-only for pilot | Report-only reviewed; exceptions documented |
| 4 | Preset security policies; attack disruption prerequisites; alert routing; admin consent workflow; app inventory | Test alert received by both administrators |
| 5-6 | Enforce device code policy and compliant-device pilot; extend passkeys to priority accounts | No unexplained support spike |
| 7-12 | Passkeys for all users in stages; first access review; first monthly review | Review log kept; actions closed |

Adjust the schedule to the customer's change windows. Do not run two enforcement changes on the same day.

## 10. Operating rhythm

| Frequency | Task | Where | Time |
|---|---|---|---|
| Weekly | Triage incidents and risky users; check Action center | Defender portal; Entra ID Protection | About 15 minutes |
| Monthly | Review PIM activations, CA changes (audit logs), consent and agent requests, success measures, exception group expiry | Entra admin center | About 2 hours |
| Quarterly | Access review of admin roles; app and agent register review; emergency account test; recovery procedure test | Entra admin center; M365 admin center | Half day |

Times are planning estimates, not measured results.

## 11. Validation checklist

- [ ] Both emergency accounts tested and excluded from blocking policies.
- [ ] Registration report shows 100% MFA-capable for in-scope accounts.
- [ ] Admins can sign in only with phishing-resistant methods.
- [ ] Every admin role is eligible; PIM activation tested by both administrators.
- [ ] Report-only results reviewed and documented before each enforcement.
- [ ] Device-code exceptions have an owner and expiry date.
- [ ] Every enterprise app and agent has an owner and recorded purpose.
- [ ] Preset security policies assigned; test alert delivered to both administrators.
- [ ] Change log records who changed what, when, and the rollback decision.

## 12. Rollback reference

| Change | Rollback |
|---|---|
| Any CA policy | Set **Enable policy** to **Off** or **Report-only**; sign in with an emergency account if both administrators are locked out |
| PIM conversion | Re-add a permanent active assignment through an emergency account |
| Passkey method | Remove the group from the method target; issue a Temporary Access Pass for recovery |
| Preset policy | Remove the group scope under Preset security policies |

## 13. Sources

- [Require phishing-resistant MFA for administrators](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-admin-phish-resistant-mfa)
- [Block authentication flows with Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-authentication-flows)
- [Conditional Access: authentication flows](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows)
- [Assign Microsoft Entra roles in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user)
- [Plan a PIM deployment](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-deployment-plan)
- [Configure the admin consent workflow](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow)
- [Configure how users consent to applications](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)
- [Safe Links overview](https://learn.microsoft.com/en-us/defender-office-365/safe-links-about)
- [Configure automatic attack disruption](https://learn.microsoft.com/en-us/defender-xdr/configure-attack-disruption)
