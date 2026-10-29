# Defend the Tenant, Step by Step

**Risk-Based M365 E7 Hardening Roadmap for a 100-User, Two-Admin Team**

*Fix the gaps, reduce privilege, enforce safely, then keep watching, with an impact check before each change.*

---

## 1. Customer scenario

A services firm with about 100 users who work from anywhere and meet clients and partners in Teams. Two administrators also run day-to-day IT, and there is no SOC. The customer is licensed for Microsoft 365 E7 (Microsoft 365 E5, Microsoft 365 Copilot, Agent 365 and the Microsoft Entra Suite), so the gap is configuration and operations, not budget.

Main risks:

- Phishing-led account takeover.
- Over-privileged admin access.
- Stolen sessions and tokens that bypass MFA.
- Apps and AI agents holding silent, long-lived access to company data.

Design constraint: every control must be small enough for two people to deploy and keep running.

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

The selection rests on three converging sources. The Microsoft Digital Defense Report 2026 calls identity the primary control plane and names phishing-resistant MFA, passkeys and strong privileged-access enforcement among the best safeguards, and Microsoft's Entra security guidance calls for PIM, emergency accounts, restricted consent, and alerts on privileged and consent changes. Recent real-world incidents confirm the priorities: the 2024 Midnight Blizzard intrusion started with a password spray against a legacy test account without MFA and escalated through an over-privileged legacy OAuth app (controls 1, 2 and 4), and Storm-2372's device-code phishing stole tokens that bypass MFA, which is why Microsoft advises blocking device-code flow and limiting it with Conditional Access (control 3). The 2025 Salesloft Drift breach showed how stolen OAuth tokens from a trusted, over-permissioned integration exposed hundreds of organizations (control 4). The lesson in each case is that a baseline MFA policy is not enough when sessions, apps and forgotten identities stay ungoverned, and a team of two needs detection that works without a SOC (control 5).

## 3. Recommended controls (ranked)

**Ranking method (risk-based):** each control was scored on (1) the likelihood and impact of the attack path it closes, (2) fit with this customer's context, and (3) the effort required of two admins using licenses already owned. Controls scoring high on risk and low on effort came first.

| # | Control | E7 capability used |
|---|---|---|
| 1 | Phishing-resistant MFA and passkeys: admins first, then executives and finance, then everyone | Entra ID P2 (Authentication Strengths), Temporary Access Pass |
| 2 | Privileged access: eligible-only admin roles, two monitored emergency accounts, recurring access reviews | Entra ID P2 (PIM), Entra ID Governance (Entra Suite) |
| 3 | Session and token protection: compliant-device access for sensitive apps, block device-code flow where not needed | Conditional Access, Intune |
| 4 | Governed apps and AI agents: named owners, admin consent workflow, least privilege, quarterly review | Entra admin consent workflow, Agent 365 |
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

- **Risk addressed:** over-privileged apps and agents hold long-lived access that is rarely reviewed. Copilot extensions and agents increasingly request broad Microsoft Graph permissions.
- **Why it matters:** user consent is already blocked in the baseline, but existing grants are not revoked by that setting. Governance closes the remaining gap.
- **Rollout:** run the admin consent workflow, assign a named owner and purpose to every app and agent, review existing grants, and repeat the review quarterly.
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
- **Monthly (about 2 hours):** review PIM activations, Conditional Access changes, consent and agent requests, and the success measures below.
- **Quarterly:** review admin roles, apps and agents.

Each item has an owner and a log entry.

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

### Success measures (targets, reviewed monthly)

| Measure | Target |
|---|---|
| Accounts that are MFA-capable | 100% |
| Permanent admin role assignments | Emergency accounts only |
| Unreviewed app and agent grants | 0 |
| Time to triage a high-severity alert | Same working day |

Targets are goals, not measured results.

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
