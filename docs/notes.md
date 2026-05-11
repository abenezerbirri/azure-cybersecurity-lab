# Cyber Lab — Portfolio Notes

A six-phase Microsoft Azure cybersecurity lab covering Active Directory,
hybrid identity, endpoint management, SIEM detection engineering, and
adversary emulation. This document captures the technical narrative,
design decisions, engineering findings, and skills demonstrated across
all phases.

---

## Lab Architecture (end state)

**Cloud infrastructure:**
- 1 Azure Virtual Network (`vnet-cyberlab`) with one subnet (`snet-servers`)
- 2 Network Security Groups separating Windows and Linux exposure profiles
- 3 VMs:
  - `vm-dc01` — Windows Server 2022 Domain Controller (B2s_v2)
  - `vm-client01` — Windows 11 Enterprise managed endpoint (D2as_v7)
  - `vm-kali01` — Kali Linux attack/tooling box (D2als_v7)
- $50/month budget alerts; auto-shutdown 23:59 PT on all VMs

**On-prem identity:**
- Active Directory domain `cyberlab.local`
- 3 Organizational Units: `Employees`, `IT Admins`, `Service Accounts`
- 5 user accounts: jsmith, sjohnson, mdavis, admin01, svc-sql
- 3 security groups, 2 baseline GPOs (password policy, USB block)

**Cloud identity:**
- Microsoft Entra ID tenant with hybrid sync via Microsoft Entra Connect
- Password Hash Synchronization (PHS) with password writeback
- Selective sync filtering (Employees + IT Admins OUs only)
- Conditional Access policy enforcing MFA on privileged accounts with
  break-glass exclusions
- M365 E5 trial (cancellation reminder set well before auto-renewal)

**Endpoint management:**
- vm-client01 domain-joined and Workplace-Joined
- Microsoft Defender for Endpoint onboarded and reporting
- Microsoft Intune managing the device with one Settings catalog policy

**Security operations:**
- Microsoft Sentinel deployed on Log Analytics Workspace `law-cyberlab-sentinel`
- 1 GB/day ingestion cap as cost guardrail
- 3 data connectors: Entra ID, Windows Security Events (via AMA + DCR),
  Defender for Endpoint
- 5 custom KQL detection rules with annotated logic, MITRE ATT&CK mapping,
  and false-positive controls
- Sentinel Workbook with 4 widgets visualizing authentication telemetry

---

# Phase 1 — Foundation

## What I built

Azure resource group, virtual network, two NSGs, and two VMs forming
the foundation for the entire lab. Set up cost controls before
provisioning anything else (budget alerts, auto-shutdown) — discipline
that paid off across the whole project.

Two NSGs instead of one because Windows and Linux have different
exposure profiles (RDP vs SSH from a single trusted IP). Cleaner than
mixing rules on a single NSG and aligned with the principle of least
privilege.

## Design decisions

**Cloud-native substitution for an on-prem hypervisor build.** Local
hardware constraints (8 GB RAM Mac) couldn't run multi-VM workloads
under Hyper-V. Substituted Azure VMs + NSGs + custom DNS for the
recommended Hyper-V + Windows Server + Linux + pfSense stack. Same
architectural concepts at cloud scale: VM lifecycle management, network
segmentation, multi-machine domain authentication, hybrid identity
extension.

**West US 2 region selected for AMA connector compatibility.** Microsoft
has documented regional restrictions on Azure Monitor Agent
data-flow-to-workspace pairings. Standardizing on one region eliminated
a class of debugging that wouldn't have taught me anything I didn't
already know.

**Standard security type, not Trusted Launch.** Trusted Launch interferes
with Active Directory Domain Services promotion. Lesson learned by hitting
it the hard way and rebuilding.

## Skills demonstrated

- Azure resource provisioning (RG, VNet, subnet, NSG, VM, NIC)
- Network segmentation via NSG inbound rule design
- Cost discipline: budgets, alerts, auto-shutdown, deallocation habit
- SSH key-pair generation and lockdown (chmod 600)
- Linux package management (apt update/upgrade, kali-linux-default)
- RDP from macOS via Windows App
- Public IP rotation (My IP address NSG rule refresh on home IP changes)

---

# Phase 2 — Active Directory

## What I built

Promoted vm-dc01 to a Domain Controller, established `cyberlab.local`
domain, built the OU/user/group structure, and applied baseline group
policies. The classic AD foundation that hybrid identity in Phase 3
extends from.

**Identity content:**
- 5 users with realistic role separation (employees, IT admin, service
  account)
- 3 security groups for role-based access control
- Password policy GPO tightening domain defaults (8+ char minimum,
  complexity required, lockout after 5 failed attempts)
- USB block GPO targeted at Employees OU — common DoD/DISA STIG pattern

**Static private IP for the DC** (10.0.0.4). Domain-joined machines
look up the DC by IP for DNS and Kerberos; dynamic assignment
catastrophically breaks domain authentication on every DC reboot.

## Design decisions

**Selective OU structure.** Service accounts isolated from human users
because they have different access patterns (no interactive logon,
elevated rights, machine-to-machine auth). IT admins separate from
employees because privileged accounts need stricter policy targeting.

**Lockout policy at 5 attempts, not 3.** Three is too aggressive for
real users and creates helpdesk noise. Five is the threshold that
catches automated attacks while tolerating human typos. Validated by
Phase 5 where a real Hydra brute force triggered lockout exactly at
attempt 5.

**Password policy enforced via GPO, not Default Domain Policy
modification.** Editing Default Domain Policy is brittle and hard to
roll back. Linking a custom Password Policy GPO to the domain root
gives the same effect with cleaner operational ownership.

## What surprised me

**Domain Controller promotion requires multiple reboots.** First reboot
after AD DS role install; second after dcpromo. The DC briefly
disconnects from RDP both times. Patience required.

**The Employees OU population step matters for GPO scoping.** GPOs
linked to an OU only apply to objects in that OU. Default user creation
lands users in the Users container, not in any OU. New users need to be
explicitly moved to the right OU before their group policies apply.
Validated by gpresult /r showing "Block USB Storage" only on jsmith
after the move.

## Skills demonstrated

- Active Directory Domain Services installation and dcpromo
- AD DNS configuration (auto-installed during dcpromo)
- OU structure design with role-based separation
- User and security group lifecycle management (PowerShell + GUI)
- Group Policy authoring, linking, and enforcement
- gpupdate / gpresult for GPO validation
- Static private IP configuration in Azure (NIC IP configuration → Static)

---

# Phase 3 — Hybrid Identity, Endpoint Management, and Conditional Access

## What I built

Built a hybrid identity stack on top of the Phase 2 AD environment.
Added a Windows 11 Enterprise client, joined it to `cyberlab.local`,
then extended identity into the cloud by syncing on-prem users to
Microsoft Entra ID via Microsoft Entra Connect Sync (Password Hash
Synchronization). Layered in Conditional Access for privileged identity
protection, onboarded the client to Microsoft Defender for Endpoint,
and enrolled it in Microsoft Intune for cloud-based device management.

**End state:** a single user identity (jsmith) created in PowerShell on
the on-prem DC now authenticates across on-prem domain logon, M365
cloud services, and managed device enrollment. A single endpoint
(CLIENT01) is simultaneously domain-joined, Workplace-Joined to Entra,
EDR-managed by Defender, and MDM-managed by Intune — the architecture
pattern most enterprises operate today.

## Design decisions

**Tenant selection — pivot rationale.** Microsoft restricts new Entra
tenant creation on freshly-upgraded Pay-As-You-Go subscriptions for
~24-48 hours as an anti-fraud measure. Rather than wait, pivoted to
using the existing Default Directory and operating cloudadmin as a
tenant-scoped admin within it. Functionally identical for hybrid
identity work; cosmetic difference only.

**Cloudadmin as separate identity from primary account.** Dedicated lab
admin account isolates blast radius. If lab work locks out cloudadmin
(CA policy misconfiguration, password loss, etc.), the primary account
retains tenant-level access. Mirrors the tier-0 admin separation
pattern most enterprises implement for break-glass emergency access.

**Password Hash Sync over Pass-through Authentication.** PHS sends a
one-way hash of the on-prem password hash to Entra. Cloud
authentication continues working even if the on-prem DC is offline. PTA
requires the on-prem DC to be online for every authentication, which
is fragile for a lab. Production environments often choose PTA for
stronger guarantees about password material location, but the
resilience trade-off is real.

**Selective OU sync filtering.** Every synced object is an attack
surface in the cloud. Service accounts should never sync (they have
privileged on-prem rights and don't need cloud access). Domain
controllers should never sync (catastrophic if compromised). Default
Users and Computers containers contain legacy and default objects that
pollute the cloud directory. Filtering the sync scope to only Employees
+ IT Admins OUs keeps the cloud directory clean and aligned with
operational reality.

**CA policy scoping at user level vs group level.** Policy targets the
admin01 user directly rather than an "IT Administrators" security
group. In production this would target the group at the role level. Lab
has only one admin user, so user-level targeting is acceptable and
simpler to reason about. Break-glass exclusions (cloudadmin, primary
account) are non-negotiable in any restrictive CA policy.

**Defender local-script onboarding vs Intune-managed onboarding.** Local
script is the most direct onboarding path and works without
dependencies on Intune. Intune-driven onboarding via Defender for
Endpoint configuration profile is the production pattern, but requires
additional license configuration. Local script demonstrates the same
end state — agent installed, telemetry flowing, device visible in
portal — without the licensing overhead.

**WorkplaceJoin vs full Hybrid Azure AD Join.** The client ended up in
WorkplaceJoined state rather than fully Hybrid Azure AD Joined because
hybrid AAD join requires AAD Connect to enable the device sync feature,
which I deliberately skipped to keep Phase 3 manageable. WorkplaceJoined
is sufficient for Intune management and Defender onboarding; only
device-based Conditional Access policies require full hybrid join.
Brownfield domain-joined enterprises often run in this same state
during cloud-migration phases.

## What surprised me

**Microsoft has deprecated Download Center distribution for Entra
Connect.** The legacy Azure AD Connect URL still serves a deprecation
PDF. Current versions are only available through the Microsoft Entra
Admin Center under Hybrid Management. The portal also pushes Cloud Sync
as "recommended" — for enterprise environments and broader industry
relevance, Connect Sync remains the right choice.

**IE Enhanced Security Configuration blocks the AAD Connect OAuth flow.**
Default Windows Server hardening blocks the embedded Internet Explorer
engine that Connect Sync uses for Microsoft authentication. Resolved by
adding the required URLs to Trusted Sites (msauth.net, msftauth.net,
login.microsoftonline.com, etc.). This is a common Windows Server admin
gotcha — IE ESC is generally too restrictive for interactive admin work.

**Intune enrollment requires user-level Intune license, not just
tenant-level.** First enrollment attempt for jsmith failed with error
-895156188 ("Error response came from MDM terms of use page"). Initial
assumption was MDM scope misconfiguration. Actual root cause: jsmith
was synced from on-prem AD via AAD Connect but had no M365 E5 license
assigned. Tenant-level license pool existed (25 seats), but jsmith
specifically had none allocated. Fix: assigned license via M365 admin
center → Users → Licenses and apps. Re-attempted enrollment — succeeded
immediately.

Takeaway: cloud service entitlements are user-scoped, not
tenant-scoped. Auto-enrollment scope controls "who is allowed to
enroll." User license controls "who can actually enroll." Both
required. This is a common real-world helpdesk ticket.

**Number-matching MFA is now mandatory.** The MFA challenge for admin01
displayed a number ("77") that I had to type into Microsoft
Authenticator on my phone. Microsoft made this mandatory in 2023 after
attackers started spamming push notifications hoping users would tap
"Approve" out of annoyance. Number matching forces active engagement,
which kills push fatigue attacks.

## Skills demonstrated

- Microsoft Entra ID administration (tenant configuration, user
  provisioning, role assignment)
- Microsoft Entra Connect Sync installation and PHS configuration
- Hybrid identity architecture (on-prem AD ↔ Entra ID via PHS)
- UPN suffix management on AD forests for routable cloud identities
- Conditional Access policy design with break-glass exclusions
- MFA enforcement and number-matching authentication
- Microsoft Defender for Endpoint local-script onboarding
- Sense service verification and EDR telemetry validation
- Microsoft Intune MDM auto-enrollment configuration
- Settings catalog policy authoring and assignment
- WorkplaceJoin vs Hybrid Azure AD Join trade-offs
- Multi-portal navigation (Azure, Entra, M365 admin, Security, Intune,
  M365 portal) — operationally what real cloud security work looks like
- Real-world troubleshooting: license-scope errors, IE ESC issues,
  Marketplace gating, quota constraints

---

# Phase 4 — Microsoft Sentinel + KQL Detection Engineering

## What I built

A working SIEM on top of the Phase 3 hybrid identity environment.
Deployed Microsoft Sentinel against a Log Analytics Workspace, wired
three data sources (Microsoft Entra ID sign-in/audit logs, Windows
Security Events from on-prem DC and client via Azure Monitor Agent, and
Microsoft Defender for Endpoint alerts), and authored five custom KQL
detection rules covering brute force, privilege escalation, anomalous
admin behavior, on-prem account lockout, and EDR alert correlation.

A Sentinel Workbook visualizes live sign-in activity, top-authenticating
users, failure codes, and the active detection rule catalog.

## The 5 detection rules

| # | Name | MITRE | Severity | What it catches |
|---|------|-------|----------|-----------------|
| 01 | Failed Sign-In Burst | T1110 | Medium | 5+ Entra failures from 1 user in 5 min |
| 02 | New Privileged Role Assignment | T1098 | High | New admin role granted to any user (incl. PIM) |
| 03 | Off-Hours Admin Sign-In | T1078 | Low | Admin authentication outside business hours |
| 04 | Windows Account Lockout | T1110 | Medium | Event ID 4740 on the DC |
| 05 | Defender Alert + Sign-In Correlation | Multi | Medium | Defender alert + recent sign-in for same user |

All five committed as annotated `.kql` files with rationale, false
positive controls, and analyst response playbooks.

## Design decisions

**1 GB/day ingestion cap as cost guardrail.** Sentinel pricing is
per-GB ingested. A misconfigured connector could trivially generate
10+ GB/day in a noisy environment. Setting the cap before connecting
any data sources is the safe order of operations: data starts
dropping at midnight UTC if you hit the cap, but no billing surprise.
Lab realistic ingestion is 1-50 MB/day.

**AMA + DCR over the deprecated Microsoft Monitoring Agent.** AMA is
Microsoft's strategic agent and the only one that supports Data
Collection Rules. DCRs scope what events flow to which workspace, which
is essential for cost control. The deprecated MMA path is
"connect and ingest everything," which doesn't scale.

**Common-tier event collection, not All Events.** "Common" includes the
authentication-relevant security events (4624, 4625, 4634, 4672, 4720,
4724, 4738, 4740, 4767) without the noise of every successful object
access. For a focused detection ruleset this dramatically reduces
ingestion volume while preserving analytical value.

**Severity calibration grounded in real analyst experience.** High =
wake someone up. Medium = investigate within an hour. Low = look at
during normal operations. Rule 2 (privileged role grant) is High
because it's both rare and high-impact. Rule 3 (off-hours admin) is Low
because legitimate off-hours work is common and false positives are
expected. Rule 4 (lockout) is Medium even though some lockouts are
legitimate (forgotten passwords) because automated brute force is the
more important signal.

## Engineering findings

**Sentinel workspace name collision.** First attempt to deploy the
workspace failed with a name-already-exists error. Workspace names are
globally unique within a region. Resolved by appending a discriminator.

**AMA on Windows runs as MonAgent processes, not as a single
"AzureMonitorAgent" service.** Searching Services.msc for the name
shows nothing; ps via Task Manager shows MonAgentCore, MonAgentManager,
MonAgentLauncher running. Took about 20 minutes to figure out the
agent was actually running.

**SecurityAlert table uses `AlertSeverity` not `Severity`.** Initial KQL
rule version filtered on `Severity == "High"` and returned empty. Quick
inspection of the table schema via `take 5` revealed the actual column
name. Reading raw schema before writing detection logic is the lesson.

**Microsoft is migrating Sentinel into the unified Defender XDR
portal.** Some Sentinel Workbook export features still live in the
classic Azure portal, others only work in the new portal, and some are
broken in both during the migration period. Workbook PDF export from
the unified portal is currently broken; classic portal works. Small
papercut for now, consequential for documentation/sharing workflows.

## Skills demonstrated

- Microsoft Sentinel workspace deployment and lifecycle
- Log Analytics Workspace cost protection (ingestion caps)
- Data connector configuration (Entra, AMA-via-DCR, Defender)
- KQL query authoring across multiple tables (SigninLogs, AuditLogs,
  SecurityEvent, SecurityAlert)
- Detection rule design (logic, severity, MITRE mapping, FP controls)
- Multi-table joins via let statements and inner joins
- Sentinel Workbook authoring (visualizations, drill-down)
- Reading Defender XDR alert schema and entity formats
- Detection-rule lifecycle: deploy → run → suppress → tune

---

# Phase 5 — Attack Simulation and Detection Validation

## What I built

Validated all 5 Phase 4 detection rules against real adversary
emulation from vm-kali01. Attack scenarios spanned reconnaissance,
brute force (both on-prem RDP and cloud sign-in), privilege escalation
via PIM eligible role assignment, behavioral analysis of off-hours
admin activity, and EDR alert generation via EICAR test file with full
3-table correlation across Defender alerts, Windows logon events, and
cloud sign-ins.

End state: every detection rule has been exercised against real attack
data. Two rules required substantial re-engineering during validation
(Rule 2 PIM expansion, Rule 5 host-bridged identity normalization) —
findings that would have caused real production gaps if discovered
during a live incident instead of a controlled exercise.

## Session A — Reconnaissance and brute force

**Reconnaissance:** nmap basic port scan and service version detection
against vm-dc01. Discovered open ports 53, 88, 135, 139, 389, 445, 464,
636, 3268, 3389, 5985 — fingerprint of a domain controller. Service
banners leaked Windows version, AD site name, domain name
(cyberlab.local), and server time.

**Hydra brute force against admin01 via RDP:** 10-password list
including the real password. Hydra reported 0 of 10 valid passwords —
ATTACK FULLY CONTAINED. Phase 2 GPO lockout policy triggered after ~5
failed attempts. All subsequent attempts (including the real password
attempt #7) rejected at the locked-account layer.

**Detection in Sentinel:**
- Rule 4 (Windows Account Lockout) fired on Event ID 4740
- Failed logon timeline: 20 events sourced from 10.0.0.5 with two
  distinct substatus codes:
  - 0xc000006a (STATUS_WRONG_PASSWORD) — first ~5 attempts
  - 0x0 (locked-account rejection) — remaining attempts post-lockout

**Cloud-side brute force:** 6 sequential failed sign-ins via
portal.office.com from InPrivate browser. Microsoft Smart Lockout did
NOT engage at 6 attempts (default threshold ~10). Rule 1 fired at
5-attempt threshold within 5-minute window — 70-second total attack
window.

## Session B — Privilege escalation and behavioral detection

**Privileged role assignment attack:** granted Privileged Role
Administrator to mdavis via Entra portal. No justification provided —
modeling attacker behavior. Removed assignment immediately after
detection validation.

**Rule 2 detection — initial failure and root cause analysis:** initial
KQL filter `OperationName == "Add member to role"` returned ZERO results
despite confirmed assignment. Diagnostic queries revealed actual
OperationName values:
- "Add eligible member to role"
- "Add eligible member to role in PIM completed"
- "Add eligible member to role in PIM requested"

Root cause: Microsoft 365 E5 includes Entra ID P2 which routes
privileged role assignments through Privileged Identity Management
(PIM) by default.

**Rule 2 KQL rebuild:** expanded OperationName filter from `==` to
`has_any(...)` covering direct active AND PIM eligible operations.
Modified RoleName extraction with `case()` fallback because PIM events
store role name at a different JSON path. Added AssignmentType column
(PIM Eligible / Direct Active / Other) for analyst triage clarity.

**Off-hours admin sign-in behavioral analysis:** ran Rule 3's KQL
against 7 days of historical data. Returned only 2 events. Diagnostic
union query against AADNonInteractiveUserSignInLogs revealed actual
admin activity volume (hundreds of token refreshes vs 2 interactive
sign-ins). Documented as a coverage finding rather than a rule failure.

## Session C — EDR alert generation and 3-table correlation

**EICAR test file generation:** RDP'd into vm-client01 as
cyberlab\jsmith. Wrote EICAR string to `$env:TEMP\eicar_test.txt` via
PowerShell Out-File. Defender immediately detected, quarantined, and
generated alert.

**Process tree captured complete attack chain:**
```
userinit.exe → explorer.exe → powershell.exe → eicar_test.txt write
```
Timestamps to-the-second from logon (7:47:28) through detection
(7:48:30) — 14 seconds from PowerShell open to file write.

**Rule 5 detection — initial failure and root cause analysis:** initial
KQL returned ZERO rows after 30 minutes wait. Diagnostic queries
revealed the Entities field for the EICAR alert contained 16 rows but
ZERO of EntityType = "account". Defender's EICAR alert tagged file
(×2) and host (×2) entities only, not user.

**Rule 5 KQL rebuild as 3-table correlation:** pivoted from 2-table
account-based join to 3-table host-bridged correlation:
1. SecurityAlert filtered on EntityType = host
2. SecurityEvent (EventID 4624 logons) as bridge: which user was on that host?
3. SigninLogs joined via short username for cloud attribution

Implemented identity normalization across three different formats:
- Defender host: `client01`
- SecurityEvent Computer: `CLIENT01.cyberlab.local` → split → `client01`
- SigninLogs UPN: `jsmith@<tenant>.onmicrosoft.com` → split → `jsmith`
- SecurityEvent TargetUserName: `jsmith`

Rule 5 fired successfully with full attribution chain: Defender alert →
host CLIENT01 → jsmith logon → jsmith cloud sign-in → US/home IP.

## Engineering findings catalog

1. **AD banner leakage on standard ports** — recon mostly unavoidable;
   network segmentation is primary control
2. **Network-layer detection gap** — no rule covers port scans / nmap
   activity
3. **Alert suppression critical** — single attack generated 6 incidents
   without suppression; fixed across all 5 rules with tiered durations
4. **Defense-in-depth victory** — Phase 2 lockout GPO contained Hydra
   even with correct password in wordlist
5. **Microsoft Smart Lockout > custom rule threshold** — Rule 1
   catches attacks earlier than platform default
6. **PIM operations require explicit handling** — modern E5 tenants
   route privileged roles through PIM; original Rule 2 missed all
   PIM-eligible assignments
7. **Interactive vs non-interactive sign-in coverage** — Rule 3 only
   sees interactive auth; token-based attacks bypass it entirely
8. **EDR Process Tree forensics** — Defender reconstructs full attack
   chain, distinguishing file evidence from prior process activity
9. **Defender entity schema for file alerts** — file/process/host
   alerts don't carry account entities; account-based correlation
   fails for these
10. **Hybrid identity normalization required** — Defender,
    SecurityEvent, and SigninLogs use three different identity
    formats; cross-table correlation requires explicit normalization

## Skills demonstrated

**Offensive operations (purple-team perspective):**
- Internal nmap reconnaissance with service version detection
- Hydra brute force against RDP via NLA
- Cloud-side credential burst against Entra portal
- Privilege escalation via PIM eligible role assignment
- EICAR-based EDR alert generation

**Defensive operations:**
- KQL detection rule authoring across SigninLogs, AuditLogs,
  SecurityEvent, and SecurityAlert tables
- Multi-table joins via let statements and inner joins
- Time-windowed correlation with `between (X-Yh .. X+Zh)`
- Identity normalization across hybrid telemetry sources
- Sentinel Analytics Rule lifecycle: create, deploy, validate, suppress, tune
- Defender XDR incident management: triage, classify, resolve

**Engineering judgment:**
- Iterative rule development (first pass = does it detect, second
  pass = does it detect ONCE per attack)
- Honest false-positive rate documentation as portfolio output
- Detection coverage gap identification and mitigation roadmap
- Real-world rule debugging: 3-step diagnostic pattern (source data,
  target data, join field)

---

# Cumulative reflection

The phase that taught the most was the phase that broke the most.

Rule 2 didn't fire because PIM operations use different OperationName
values. Rule 5 didn't fire because Defender file alerts don't carry
account entities. Both failures were learning opportunities the
original Phase 4 detection engineering work would have shipped to
production unnoticed.

The portfolio value of "I built 5 rules that fired cleanly" pales next
to "I built 5 rules, validated them against real attacks, discovered
two had production-relevant gaps, debugged the root causes, and
rewrote them to handle the actual data shape my tenant produces."

The lab progressed from "I built a SIEM" to "I purple-teamed my own
SIEM, broke it intentionally, fixed what I found, and documented every
lesson."

---

## What I'd do differently

- **Set Public IPs to Static from day 1** instead of fighting NSG drift
  three separate times when home IP rotated.
- **Configure alert suppression on analytics rules at deployment time**,
  not after the first attack generated 6 duplicate incidents.
- **Plan the Defender alert correlation rule for host-bridged
  architecture from the start**, not as a Phase 5 rebuild.
- **Read the schema before writing the KQL.** AlertSeverity vs
  Severity, Add member to role vs Add eligible member to role —
  diagnostic queries against the live data table would have surfaced
  these in 2 minutes instead of 20.

---

## Repository structure

```
.
├── README.md           — this overview
├── docs/
│   └── notes.md        — this document
├── kql-queries/
│   ├── 01-failed-login-burst.kql
│   ├── 02-new-privileged-role.kql
│   ├── 03-off-hours-admin-signin.kql
│   ├── 04-account-lockout.kql
│   └── 05-defender-alert-correlation.kql
└── screenshots/        — visual evidence per phase
```
