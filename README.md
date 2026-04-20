# Device Code Flow Attack — Microsoft Entra ID Lab

**Organization:** Eag(xxx)cureIT  
**Tenant:** eagle(xxx)ureit.com  
**Lab Date:** April 2026  
**Type:** Red Team / Blue Team Simulation

---

## What This Is

This project documents a hands-on security lab where I simulated a Device Code Flow phishing attack against a real Microsoft Entra ID tenant — first with no defenses, then with a full security stack deployed. The goal was to understand how this attack works at a technical level, what the attacker sees, what the defender sees, and how to stop it.

This is not a theoretical exercise. Every screenshot, log entry, and alert in this repo came from a real simulation run against a live test environment.

---

## Why Device Code Flow Matters

Most people think phishing attacks need a fake login page. Device code flow phishing is different — it uses Microsoft's own legitimate login page. The victim sees no red flags. No fake site. No suspicious URL. Just a normal Microsoft sign-in prompt.

The attacker never touches the victim's password. They just wait while the victim unknowingly hands them a fully valid access token. Once they have it, they have everything — email, files, Teams messages, and the ability to move laterally inside the organization.

What makes this particularly dangerous is that the token comes with a refresh token attached. Even if the victim changes their password the next day, the attacker's session can remain active for up to 90 days.

---

## Business Impact — What Happens Without These Controls

Before getting into the technical details, it is worth understanding what a successful device code flow attack actually costs a business.

### Direct Financial Impact

| Scenario | Estimated Cost |
|---|---|
| Incident response and forensics | $50,000 — $200,000 |
| Legal and regulatory notifications | $20,000 — $100,000 |
| Regulatory fines (PIPEDA, GDPR) | $25,000 — $20,000,000+ |
| Business downtime and lost productivity | $10,000 — $50,000 per day |
| Ransomware deployment (common follow-on) | $500,000 — $5,000,000+ |
| Cyber insurance premium increase | 20% — 40% annual increase |
| Reputational damage and lost clients | Unquantifiable long-term |

### What an Attacker Can Do With One Stolen Token

If the controls in this lab were not deployed, here is exactly what the attacker could do from the moment they receive the token:

**Hour 1 — Establish foothold**
- Read all emails in the victim's inbox going back years
- Create inbox rules to silently delete Microsoft security alert emails so the victim never notices the breach
- Register their own phone number as an MFA method so they keep access even after a password reset

**Hour 2 — Lateral movement**
- Send internal phishing emails to colleagues, executives, and finance teams — appearing to come from a trusted internal sender with no suspicious indicators
- Access SharePoint and Teams channels for internal documents, stored credentials, network diagrams, and financial records

**Hour 3 — Escalation**
- Search the victim's email history for passwords, VPN credentials, and banking information
- If the victim has admin rights — access other user accounts, disable security controls, and create persistent backdoor accounts
- Deploy ransomware across the environment or sell verified access to other threat actors

The average cost of a data breach in Canada in 2024 was **$6.32 million CAD**. A single compromised account left undetected for 24 hours can lead to full domain compromise. Device code flow attacks have been directly attributed to real-world breaches by threat actor groups including **Storm-2372**, which Microsoft Threat Intelligence documented actively targeting enterprises and government organizations using this exact technique.

---

## Lab Results at a Glance

| | Phase 1 — No Defenses | Phase 2 — Defended |
|---|---|---|
| Token issued to attacker | ✅ Yes - full access obtained | ❌ No — blocked at issuance |
| Device code status | SUCCESS | EXPIRED |
| Sign-in detection | Nothing fired | Defender for Cloud Apps alerted |
| Attacker IP logged | Not captured | 17X.81.2XX.XXX — Arizona, US |
| Sentinel alert | No rule matched | FAILURE logged — gap identified and fixed |
| Account status | Active and accessible | Auto-disabled by Identity Protection |
| Post-compromise actions | All possible | All prevented — no token, no access |

---

## Attack Flow

```
Attacker                          Victim                        Microsoft
   │                                 │                               │
   │── POST /devicecode ────────────────────────────────────────────▶│
   │◀─ user_code: F2GGN3XXX ────────────────────────────────────────│
   │                                 │                               │
   │── "Enter this code" ───────────▶│                               │
   │                                 │── goes to devicelogin ───────▶│
   │                                 │── enters F2GGN3YVQ ──────────▶│
   │                                 │── completes MFA ─────────────▶│
   │                                 │                               │
   │── POST /token (polling) ───────────────────────────────────────▶│
   │◀─ access_token + refresh_token ────────────────────────────────│
   │                                                                  │
   │── GET /me, /messages, /drive (as victim) ──────────────────────▶│
```

The victim authenticates on Microsoft's real infrastructure. The attacker collects the token from a completely separate machine. The two never interact directly.


<img width="1000" height="470" alt="image" src="https://github.com/user-attachments/assets/ba9be557-840b-4f5b-b96b-a2777daf941c" />

---

## Controls Deployed in Phase 2

### Conditional Access Policies — 8 Policies, All Enforced

| Policy | Purpose |
|---|---|
| All Users — Device Code Flow — Block Access | Blocks the device code protocol entirely at the CA layer |
| All Users — Require Token Protection | Binds tokens to the enrolled device — prevents token replay on attacker machine |
| All Users — Sign In Risk — Block Access | Integrates with Identity Protection to block risky sign-ins |
| All Users — All Apps — Require Strong MFA | Enforces phishing-resistant MFA across all applications |
| All Users — Device Registrations — Require MFA | Blocks attacker from registering their own MFA method post-compromise |
| All Users — Legacy Auth Clients — Block Access | Blocks older protocols that bypass MFA |
| Block Login From External Corporate Network | Restricts sign-ins from outside trusted IP ranges |
| Corporate Network — MFA Required To Login | Requires MFA even inside the corporate network |

### Named Locations
Trusted IP ranges defined as "Corporate Network - Trusted" and referenced by CA policies. Sign-ins from outside these ranges are treated as higher risk and feed into Identity Protection scoring.

<img width="900" height="700" alt="image" src="https://github.com/user-attachments/assets/4b236adc-1723-48f4-83d3-504c9ca2fe5a" />

### Microsoft Defender for Cloud Apps
- **Device Code Auth - Anomalous Location** — Custom activity policy alerting on native client logons from outside Canada
- **Impossible Travel** — Built-in anomaly detection, sensitivity set to High
- **Mass Download by Single User** — Built-in anomaly detection for bulk file exfiltration, sensitivity set to High

### Microsoft Sentinel — KQL Detection Rules
- Rule 1: Alert on successful device code authentication (ResultType == 0)
- Rule 2: Alert on blocked device code attempts — gap fix identified during Phase 2 testing
- Rule 3: Alert on inbox rule creation matching security alert keywords
- Rule 4: Alert on new MFA method registration

### Microsoft Entra Identity Protection
Sign-in risk policy set to block on Medium and High risk. Automatically disabled the target account after the Phase 2 attack attempt.

### Privileged Identity Management
All privileged roles converted from permanent active assignment to eligible — requiring just-in-time activation with MFA and justification.

---

## Post-Compromise Actions Documented

These are the actions an attacker performs after obtaining a token. They are documented here to show the full blast radius of a successful compromise and to validate that detection rules for each action are in place.

**Inbox rule creation** — Attacker creates rules to silently delete or move Microsoft security alert emails before the victim sees them. Detected by Sentinel OfficeActivity rule monitoring New-InboxRule operations.

**Sensitive file download** — Attacker uses Graph API to enumerate and bulk download OneDrive and SharePoint files. Detected by Defender for Cloud Apps Mass Download policy.

**Internal phishing — lateral movement** — Attacker sends phishing emails from the victim's mailbox to other employees. Trusted internal sender bypasses most email filters. Detected by Defender for Cloud Apps mail anomaly detection.

**MFA method registration** — Attacker registers their own phone or authenticator app under the victim's account, surviving a password reset. Prevented by the Device Registrations CA policy and detected by Sentinel AuditLogs rule.

---

## Detection Gap Found and Fixed

During Phase 2 testing, a gap was identified in the Sentinel detection rules.

The original rule used `where ResultType == 0` which only fires on successful authentications. Because the CA policy blocked the Phase 2 attack before a token was issued, the sign-in logged as a failure — and no Sentinel alert fired.

This means the security team would have had zero visibility into the blocked attack attempt. Blocked attempts matter — they are early warning signals that an attacker is actively targeting accounts.

**The fix — updated rule covering both outcomes:**

```kql
SigninLogs
| where AuthenticationProtocol == "deviceCode"
    or ResultDescription has "AADSTS70016"
| where ResultType != 50053
| extend Outcome = case(
    ResultType == 0,     "SUCCESS - Token Issued",
    ResultType == 50097, "BLOCKED - Device Compliance",
    ResultType == 53003, "BLOCKED - Conditional Access",
    "BLOCKED - Other Failure")
| project TimeGenerated, UserPrincipalName, IPAddress,
          Location, AppDisplayName, Outcome, ResultType
| order by TimeGenerated desc
```

---

## Evidence

### Phase 1 — No Defenses

| Screenshot | What It Shows |
|---|---|
| 01-devicelogin-prompt.png | microsoft.com/devicelogin — victim-side code entry page |
| 02-device-code-success.png | Attacker tool — User code F2GGN3YVQ — Status: **SUCCESS** |

### Phase 2 — Defended

| Screenshot | What It Shows |
|---|---|
| 01-browser-blocked.png | "Help us keep your device secure" — token refused, device not managed |
| 02-device-code-expired.png | Attacker tool — Status: **EXPIRED** — no token issued |
| 03-sentinel-kql-failure.png | Sentinel log — IP 172.81.60.237, Location US, ResultSignature: FAILURE |
| 04-sentinel-enhanced.png | Enhanced query — AuthenticationProtocol: none — early CA block behavior |
| 05-defender-alert.png | Defender for Cloud Apps — failed logon, attacker device fingerprinted |
| 06-ca-policies-list.png | All 8 CA policies — State: On |
| 07-named-locations.png | Corporate Network - Trusted — IP ranges, Trusted: Yes |
| 08-account-disabled.png | RDP Error 0xb07 — account auto-disabled by Identity Protection |

---

## Repo Structure

```
device-code-lab/
├── README.md
├── docs/
│   ├── attack-explained.md      How device code flow works
│   ├── phase1-attack.md         Phase 1 full walkthrough
│   ├── phase2-defense.md        Every control deployed and why
│   ├── phase2-results.md        What happened when defenses were active
│   ├── gap-analysis.md          Detection gap found and fixed
│   └── recommendations.md       Priority-ordered production guidance
├── detections/
│   ├── sentinel-rules.kql       All KQL analytics rules
│   └── defender-policies.md     Defender for Cloud Apps configurations
├── evidence/
│   ├── phase1/                  Phase 1 screenshots
│   └── phase2/                  Phase 2 screenshots
└── scripts/
    └── simulate-attack.md       Attack commands reference
```

---

## Key Takeaway

The same attack that took under 3 minutes to succeed in Phase 1 was completely stopped in Phase 2 — with the attacker's IP, device, location, and ISP all logged as evidence. The security stack did not just block the attack. It documented it.

The gap analysis added additional value beyond the initial lab objective by identifying that detection and prevention need to be tested together. A control that blocks an attack silently, with no corresponding alert, leaves the security team blind to the fact that they are being targeted.

---

> **Disclaimer:** This lab was conducted against an authorized test environment. All simulation activity was performed on accounts and systems owned by the lab operator. Do not replicate this against any environment you do not have explicit written authorization to test.
