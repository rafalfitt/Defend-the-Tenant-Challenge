# Defend the Tenant, Step by Step

**Risk-Based M365 E7 Hardening Roadmap for a 100-User, Two-Admin Team**

*Fix the gaps, reduce privilege, enforce safely, then keep watching, with an impact check before each change.*

**Five controls beyond the Phase 1 baseline, each mapped to an E7 license, a risk and a measurable outcome, sized so two admins can run it. Today, 1 of 10 checked accounts is MFA-capable; the target is 100%.**

---

## 1. Customer scenario

A services firm with about 100 users who work from anywhere and meet clients and partners in Teams. Two administrators also run day-to-day IT, and there is no SOC. The customer is licensed for Microsoft 365 E7 with Microsoft Entra ID P2 / Entra Suite capabilities and Microsoft 365 Copilot and agent governance features available in the tenant, so the gap is configuration and operations, not budget.

Main risks:

- Phishing-led account takeover.
- Over-privileged admin access.
- Stolen sessions and tokens that bypass MFA.
- Apps and AI agents holding silent, long-lived access to company data.

Design constraint: every control must be small enough for two people to deploy and keep running.

## 1.5 Operational guardrails for a two-admin environment

Because there is no SOC, every control must be simple enough to validate, log, and reverse quickly. The plan assumes that no policy is enforced unless the following are in place:

- A recorded impact review before the change is enabled.
- A pilot or testing group, with no critical business service left unassessed.
- A documented rollback procedure and a test of that rollback path.
- A list of exceptions with an owner, reason, expiry date, and review cadence.
- A shared change log that records what changed, who approved it, and when it was reviewed.

Every Conditional Access policy starts in report-only mode, and every exception stays temporary. The default assumption is to remove access when the business need expires, not to leave permanent carve-outs. This is the operational discipline needed to keep a no-SOC environment sustainable.

### Exception and rollback process

For every control that restricts access, the admins should maintain a short exception register with:

- Exception name and scope
- Business owner
- Risk justification
- Requested start and end date
- Approval by the second administrator
- Review date and final decision

Rollback must be explicit:

- for Conditional Access, return the policy to report-only or disable it
- for device compliance, revert the policy assignment or tighten the group scope
- for app or agent restrictions, revoke the consent or remove the assignment
- for role changes, restore the prior assignment and revalidate access before closing the ticket

This reduces operational drift and ensures the team can recover quickly if a control blocks an essential workflow.

## 2. Already in place (Phase 1 baseline)

The following 10 controls are already in place, as verified by the Phase 1 assessment script:

1. MFA for all users in administrative roles
2. MFA for all users
3. Legacy authentication blocked through Conditional Access
4. Entra ID Protection sign-in risk policy
5. Entra ID Protection user risk policy
6. User consent to applications not allowed
7. Teams: only invited users auto-admitted
8. Teams: presenter role restricted
9. Teams: anonymous join restricted
10. Self-service password reset set to All

This plan does not repeat these controls. It builds on them and targets what the Microsoft Digital Defense Report 2026 identifies as the next priorities: identity as the primary control plane, phishing-resistant authentication, tiered and least-privilege administration, session and token theft, and ungoverned AI agents.

### Why these controls were chosen

- **The report:** the Microsoft Digital Defense Report 2026 calls identity the primary control plane and names phishing-resistant MFA, passkeys and strong privileged-access enforcement among the best safeguards (controls 1 and 2).
- **Microsoft's recommendations:** Entra security guidance calls for PIM, emergency accounts, restricted consent, and alerts on privileged and consent changes (controls 2, 4 and 5).
- **Midnight Blizzard (2024):** the intrusion started with a password spray against a legacy test account without MFA and escalated through an over-privileged legacy OAuth app (controls 1, 2 and 4).
- **Storm-2372 (device-code phishing):** it stole tokens that bypass MFA, which is why Microsoft advises blocking device-code flow and limiting it with Conditional Access (control 3).
- **Salesloft Drift breach (2025):** stolen OAuth tokens from a trusted, over-permissioned integration exposed hundreds of organizations (control 4).
- **Lesson learned:** a baseline MFA policy is not enough when sessions, apps and forgotten identities stay ungoverned, and a team of two needs detection that works without a SOC (control 5).

## 3. Recommended controls (ranked)

**Ranking method (risk-based):** each control was scored on (1) the likelihood and impact of the attack path it closes, (2) fit with this customer's context, and (3) the effort required of two admins using licenses already owned. Controls scoring high on risk and low on effort came first.

| # | Control | E7 capability used |
|---|---|---|
| 1 | Phishing-resistant MFA and passkeys: admins first, then executives and finance, then everyone | Entra ID P2 (Authentication Strengths), Temporary Access Pass |
| 2 | Privileged access: eligible-only admin roles, two monitored emergency accounts, recurring access reviews | Entra ID P2 (PIM), Entra ID Governance (Entra Suite) |
| 3 | Session and token protection: compliant-device access for sensitive apps, block device-code flow where not needed | Conditional Access, Intune |
| 4 | Governed apps and AI agents: named owners, admin consent workflow, least privilege, quarterly review | Microsoft Entra admin consent workflow, app governance, Microsoft 365 Copilot/agent governance |
| 5 | Detection and response for a team of two: Safe Links and QR-phishing protection, automatic attack disruption, alerts to both admins | Defender XDR, Defender for Office 365 |

## 4. Control detail

### Control 1: Phishing-resistant MFA and passkeys

- **Risk addressed:** credential theft and MFA relay. One-time codes and push prompts can be phished or replayed through attacker-in-the-middle pages.
- **Why it matters:** the baseline MFA policy is the floor. Phishing-resistant methods (passkeys/FIDO2, Windows Hello for Business) cannot be replayed by a fake sign-in page, and the Digital Defense Report names them among the best safeguards.
- **Rollout:** verify MFA registration for every account, then roll out in stages: admins, executives and finance, everyone. Use Authentication Strengths in Conditional Access, and Temporary Access Pass for onboarding and recovery.
- **Tradeoffs:** enrollment effort and lost-device recovery. Mitigated by staged groups and a documented recovery process.

### Control 2: Privileged access with PIM and access reviews

- **Risk addressed:** a permanently privileged account is a permanent full-takeover target.
- **Why it matters:** just-in-time access means admin power exists only while it is used, and every activation is logged with a reason.
- **Rollout:** convert admin roles to eligible, require MFA and justification on activation, keep the number of admins minimal, create two cloud-only emergency accounts with alerts on any use, and run a recurring access review (Entra ID Governance).
- **Tradeoffs:** one extra step per privileged task. Mitigated by clear activation rules.

### Control 3: Session and token protection

- **Risk addressed:** a stolen session or token bypasses MFA after sign-in, and device-code phishing is a growing technique.
- **Why it matters:** tying access to a trusted device and removing unneeded sign-in flows makes stolen sessions far less useful.
- **Rollout:** require compliant devices for sensitive apps and block device-code flow where it is not needed. Start both in report-only mode, review results, then enforce.
- **Tradeoffs:** personal devices may be blocked or asked to enroll. Mitigated by a pilot group and a documented exception path.

### Control 4: Governed apps and AI agents

- **Risk addressed:** over-privileged apps and agents hold long-lived access that is rarely reviewed. Copilot extensions and other agent experiences can request broad Microsoft Graph permissions if not governed carefully.
- **Why it matters:** user consent is already blocked in the baseline, but existing grants are not revoked by that setting. Governance closes the remaining gap.
- **Rollout:** run the Microsoft Entra admin consent workflow, assign a named owner and purpose to every app and agent, review existing grants, and repeat the review quarterly.
- **Tradeoffs:** approval delays. Mitigated by a short approver SLA and a pre-approved list.

### Control 5: Detection and response for a team of two

- **Risk addressed:** with no SOC, phishing and compromise go unnoticed.
- **Why it matters:** earlier detection and automatic containment shorten attacker dwell time without adding staff.
- **Rollout:** turn on Safe Links and Safe Attachments (including QR-code phishing protection), enable automatic attack disruption, and route high-severity alerts to both admins.
- **Tradeoffs:** early alert noise. Mitigated by tuning during the first weeks.

## 5. Rollout plan (security is a journey, not a switch)

Every change follows the same cycle: **check impact, pilot, enforce, monitor.** This follows the Baseline Security Mode approach of reviewing impact reports and enabling settings with no impact first. New Conditional Access policies start in report-only mode.

| Step | Focus | Gate before moving on |
|---|---|---|
| 1. Fix the gaps | Verify MFA registration and role coverage for every account. Create and monitor two cloud-only emergency accounts. Passkeys for admins. | 100% of accounts MFA-capable |
| 2. Reduce privilege | Convert admin roles to PIM eligible. Run the first access review. | No standing admin access except emergency accounts |
| 3. Enforce safely | Compliant-device and device-code policies in report-only, then enforced. Turn on Safe Links and attack disruption. | Report-only results clean, no support spike |
| 4. Keep watching | Alerts, monthly review, quarterly review of roles, apps and agents. | Review log kept, actions closed |

### Operating rhythm for two admins

- **Weekly (about 15 minutes):** triage Defender and risky-user alerts.
- **Monthly (about 2 hours):** review PIM activations, Conditional Access changes, consent and agent requests, emergency-account use, and the success measures below.
- **Quarterly:** review admin roles, apps and agents, and expire stale exceptions.

Each item has an owner and a log entry.

### Monthly review checklist

The two admins should use the same checklist every month to keep the plan manageable and defensible:

- PIM activations and approvals
- Emergency account use and alerts
- Conditional Access changes and exception register
- Risky sign-ins and user-risk remediation outcomes
- New app consent or AI agent requests
- Unreviewed or stale app and agent grants
- Defender alerts and containment actions
- Passkey or phishing-resistant MFA registration status
- Any support issue caused by policy enforcement

This keeps the review from becoming a vague “check the portal” exercise.

## 6. Security risk to customer impact

| Security risk | Recommended control | Why it matters | Customer impact |
|---|---|---|---|
| Credential theft, MFA relay | Passkeys / phishing-resistant MFA | The report names these among the best safeguards | Account takeover much harder |
| Standing admin access | PIM, access reviews, emergency accounts | Admin power exists only while used and is logged | Far smaller blast radius |
| Session and token theft | Compliant-device Conditional Access, block device-code flow | MFA does not stop a stolen session | Stolen tokens far less useful |
| Over-privileged apps and AI agents | Consent workflow, owners, Agent 365, quarterly review | Ungoverned agents are a named risk | Only reviewed access to company data |
| Undetected phishing, no SOC | Defender XDR, Safe Links, alert routing | Faster containment without extra staff | Same-day triage |

## 7. How the overall security posture improves

The Phase 1 baseline covers the basics. This plan moves the customer from "controls configured" to "controls that hold under attack": credentials cannot be replayed, a stolen admin account or session does limited damage, apps and agents cannot hold silent access, and threats are contained quickly without a SOC. The result is a smaller attack surface, a shorter attacker dwell time, and a plan that stays maintainable by two people. The approach follows Zero Trust principles: verify explicitly, use least privilege, and assume breach.

### Success measures (reviewed monthly)

| Measure | Baseline (measured) | Target |
|---|---|---|
| Accounts that are MFA-capable | 1 of 10 checked accounts (assessment script, October 29, 2026) | 100% |
| Accounts with phishing-resistant MFA enabled and tested | Not yet measured | 100% |
| Users with verified MFA registration | Not yet measured | 100% in scope |
| Permanent admin role assignments | Not yet measured | Emergency accounts only |
| PIM role assignments with reviewed activation and reason | Not yet measured | 100% |
| Unreviewed app and agent grants | Not yet measured | 0 |
| Exceptions with owner, expiry and review date | Not yet measured | 100% |
| Time to triage a high-severity alert | Not yet measured | Same working day |
| Emergency account use reviewed | Not yet measured | 100% within 1 business day |

Only the MFA-capable baseline is a measured result; it comes from the Phase 1 assessment script. All targets are goals, not measured results. The team should not treat a target as met by policy presence alone; the review must confirm actual user registration, real sign-in testing, and completed exception reviews.

## 8. Looking ahead

- Move toward passwordless by default as passkeys mature.
- Keep the consent workflow as the control point for AI agents.
- Extend into device compliance baselines and data protection (Intune, Purview) using the same gated cycle.
- Re-check against Microsoft's evolving Entra security guidance.

## 9. References

1. [Microsoft Digital Defense Report 2026](https://www.microsoft.com/en-us/corporate-responsibility/topics/cybersecurity/reports/microsoft-digital-defense-report/)
2. [Configure Microsoft Entra for increased security (preview)](https://learn.microsoft.com/en-us/entra/fundamentals/configure-security)
3. [Best practices to secure with Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/architecture/secure-best-practices)
4. [Baseline Security Mode settings](https://learn.microsoft.com/en-us/microsoft-365/baseline-security-mode/baseline-security-mode-settings?view=o365-worldwide)
5. [Phishing-resistant MFA, Secure Future Initiative guidance](https://learn.microsoft.com/en-us/security/zero-trust/sfi/phishing-resistant-mfa)
6. [Midnight Blizzard: Guidance for responders on nation-state attack (Microsoft Security Blog)](https://www.microsoft.com/en-us/security/blog/2024/01/25/midnight-blizzard-guidance-for-responders-on-nation-state-attack/)

---

Phase 2 is a proposal for a fictional customer. Phase 1 results come from the challenge lab tenant.
