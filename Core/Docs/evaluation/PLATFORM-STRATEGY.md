# Platform strategy: build base, competitive position, and product gaps

Companion to [FORK-ASSESSMENT.md](FORK-ASSESSMENT.md). That document evaluated
IronDeploy in isolation. This one evaluates it against the alternatives —
FOG Project, OSDCloud, SmartDeploy, ManageEngine OS Deployer — under a revised
set of requirements:

- wipe and provision devices, install applications;
- capture a configured machine using the Windows tooling (`sysprep /generalize`)
  and restore it to a clean out-of-box experience;
- register devices to Intune / Autopilot on demand;
- usable by an MSP across many customers;
- viable as a product that can be sold.

Assessed 2026-08-07.

---

## 1. The finding that decides the build base

**Both open-source candidates are GPLv3. IronDeploy is the only Apache-2.0
option on the table.**

| Candidate | License | Closed-source commercial derivative? |
| --- | --- | --- |
| **IronDeploy** | **Apache 2.0** | **Yes** |
| FOG Project | GPL-3.0 (`fogproject`, `fog-ipxe`, `fog-client`, `fog-docs`) | No |
| OSDCloud / OSD module | GPL-3.0 | No |
| SmartDeploy | Proprietary (PDQ) | N/A — competitor |
| ManageEngine OS Deployer | Proprietary (Zoho) | N/A — competitor |

GPLv3 permits *selling* software. What it forecloses is shipping a
**closed-source derivative**. Handing a customer a FOG- or OSDCloud-derived
installer, appliance, or bootable image is "conveyance" and triggers full
corresponding-source disclosure — plus, for consumer hardware, GPLv3 §6
Installation Information (anti-tivoization).

Neither project offers an escape hatch:

- **FOG** has no CLA, no DCO, no copyright assignment, and **107 distinct
  contributor identities** across 2008–2026. Nobody — including the lead
  maintainer — is in a position to grant a proprietary license. Relicensing is
  not achievable.
- **OSD** is GPL-3.0 with `CompanyName = 'Recast Software'` in the module
  manifest. It is not merely copyleft; it is copyleft **owned by a commercial
  vendor in this exact market**. Building a competing product on the code of
  the company most likely to commercialize it is a poor strategic position
  regardless of the license.

**License flow is one-way.** Apache-2.0 code can be absorbed *into* a GPLv3
work. No GPLv3 code can enter IronDeploy without relicensing IronDeploy to
GPLv3 and discarding its Apache terms.

### Practical rule for the team

Read FOG and OSDCloud for **architecture and hard-won operational knowledge**.
Do not copy their code. The distinction matters legally and is easy to blur
under deadline pressure:

- ✅ Learning *that* offline driver injection needs a specific DISM sequence.
- ✅ Learning *that* Secure Boot enrollment does not scale without a signed
  shim, from FOG's own architecture decision records.
- ❌ Pasting their functions, or writing a file with their source open beside it.

Both codebases are worth reading closely. FOG's FOS architecture decision
records and OSDCloud's `Set-OSDCloudUnattendAuditModeAutopilot.ps1` are
unusually candid about what does and does not work in practice.

---

## 2. Candidate summaries

### FOG Project

Actively developed right now — 1,563 commits in 90 days, monthly automated
releases, latest `1.5.10.2149` (2026-07-30). But 91% of human commits are one
person, and there are three parked rewrites across ten years.

Disqualifying issues beyond the license:

- **The Windows story is thin.** Zero occurrences of `sysprep` or `unattend` in
  the codebase. No per-model driver injection feature — that is your bash
  script in `postdownloadscripts`. Windows-specific surface is exactly three
  things: hostname change, domain join, product key. It is a block-level
  cloner (partclone + zstd), filesystem-agnostic by design.
- **BitLocker is a hard refusal on capture**, with no re-protection on deploy.
- **Secure Boot needs a human at each machine.** The project's own ADRs state
  this plainly; the signed-shim fix is recorded as proposed, unstarted, and a
  "multi-month, ongoing-cost undertaking."
- **The Windows agent is abandoned** — `fog-client` 0.13.0, March 2023,
  .NET Framework 4.5.2, zero commits in 3.5 years.
- **Multi-tenancy is actively hostile.** The installer grants every storage
  node `SELECT` on the entire database from any host (`'%'`), with the password
  in plaintext in `/opt/fog/.fogsettings`. A node at Customer A's site holds
  credentials to read every other customer's data. The imaging plane is
  unauthenticated — hosts are identified by client-POSTed MAC.
- **Zero Autopilot/Intune.** One user issue asking about provisioning packages,
  open and unanswered for two years.

⚠️ Also note: the `fos` repository — the actual imaging engine — has **no
license file at all**. Not permissive, not copyleft: undetermined. Would
require written clarification before any use.

### OSDCloud / OSD PowerShell module

- GPL-3.0. Author David Segura; `CompanyName = 'Recast Software'`.
- Version `26.8.5.2`, last commit 2026-08-05 — very actively developed.
- 244 stars, 101 forks. Bus factor 1, now with commercial backing.

**What it is:** a PowerShell module that builds WinPE media and deploys Windows
pulled directly from Microsoft over the internet. `Start-OSDCloud`,
`Start-OSDCloudGUI`, `New-OSDCloudTemplate`, `New-OSDCloudUSB`.

**What it is not:** a platform. No server, no database, no web console, no
inventory, no reporting, no RBAC, no multi-tenancy. It is a very good toolkit,
not a product.

**Autopilot handling** is via `AutopilotConfigurationFile.json` injection into
the offline image plus an audit-mode reseal unattend
(`Set-OSDCloudUnattendAuditModeAutopilot.ps1`). That is *offline profile
assignment* — it presumes the device is already registered to a tenant. It is
not tenant-aware hardware-hash harvesting.

**Its real value to you is as a reference.** The thin-image model — pull stock
Windows, add drivers, add apps, never maintain a golden image — is the
direction the market is moving, and IronDeploy already works this way.

### SmartDeploy (now PDQ)

Three decoupled artifacts merged at deploy time: a hardware-clean golden image
captured from a **powered-off VM's virtual disk file**, a Platform Pack of
per-model drivers, and Application Packs of silent installers.

Two claims worth deflating, since you use this product:

- **"Application layering" is not layering.** Their own documentation describes
  a zip containing an installer plus a command line, executed silently after
  deployment. That is package-and-run — the same thing Intune Win32 apps,
  winget, and PDQ Deploy do. Real layering is a mountable filesystem overlay.
  Their review corpus confirms the difference: users report that packages must
  be manually re-edited when a bundled component updates, which a real layer
  would not require.
- **"Hardware-independent imaging" is commodity `DISM /Add-Driver`.** The moat
  is the 1,500-model curated pack library, not the technique — a content and
  logistics business. It is also their softest point: reviews repeatedly cite
  packs that are stale or "don't always apply."

One design decision worth copying outright: **sysprep runs at deploy time on
the target, not at capture time on the reference machine.** The golden VM stays
permanently patchable instead of being consumed by every capture. See §4.

3.0 was a control-plane modernization, not an imaging rewrite: SOAP→REST,
LocalDB→MongoDB, desktop→web console, multi-user concurrent access, cloud
distribution points. They reached **multi-user** and stopped short of
**multi-tenant**.

Pricing: roughly $1,100/yr floor at a 50-device minimum (≈$20–22/device/yr),
quote-only above that. No MSP or multi-tenant licensing program found.

### ManageEngine OS Deployer

Two-tier on-premises: server plus a Distribution Server per remote office.
Supports **both online capture (imaging a live running machine) and offline
WinPE capture** — a genuine architectural advantage over SmartDeploy, which can
only capture from a powered-off VM's disk file. Hardware Independent Deployment
injects drivers at deploy time from a central repository. x64 WinPE only.

Published pricing: $345/yr for 50 devices, $645/100, **$2,995/yr unlimited**.
Roughly 3× cheaper per device than SmartDeploy, and transparent.

---

## 3. The competitive gap — narrower than it looks

**MDT was retired outright on 2026-01-06** — immediate, with downloads pulled
and no in-place upgrade path. That created the opening. But the opening is not
empty.

**ManageEngine shipped OS deployment into their multi-tenant Endpoint Central
MSP cloud suite in March 2025**: per-customer dashboards, tenant segregation,
per-client deployment templates. This is the one combination a new entrant
would be counting on being unserved, and it is not.

It remains attackable for one specific reason: **it requires adopting their
entire UEM suite** — RMM, patch management, inventory, remote control. An MSP
already running NinjaOne, Datto, or Syncro cannot take ManageEngine's OS
deployment without a full platform migration.

> **The wedge is multi-tenant deployment that does not require replacing the
> RMM.** That is narrower and more defensible than "there is no multi-tenant
> imaging tool," which is no longer true.

### Why the Microsoft-native answers fail an MSP

- **Autopilot v1** requires a hardware hash. Hashes are generated at OEM
  manufacturing and are not normally provided to customers or CSPs. Microsoft's
  own documentation calls manual registration "primarily for testing and
  evaluation" because it requires booting into Windows first — roughly
  30–45 minutes per device. MSPs buy retail, mixed, and refurbished hardware
  with no OEM registration relationship. **Disqualifying at the small-client
  end.**
- **Autopilot v2 (device preparation)** removes the hash requirement, but is
  **Entra-join only — no Hybrid Join** — and Windows 11 only. That eliminates
  every client with on-prem AD, which is most SMB clients running a
  line-of-business app or a file server.
- **Neither variant installs an operating system.** Autopilot configures the
  Windows already on the disk. Bare metal, a corrupted recovery partition, or a
  bloated retail laptop all still need an imaging tool.
- **ConfigMgr OSD** requires site servers, SQL, and distribution points.
  Economically absurd for a 25-seat client.

### The highest-leverage unbuilt feature

The correct 2026 MSP workflow is not "image *or* Autopilot" — it is
**image *then* Autopilot**:

```
bare metal / broken machine / OEM-bloated retail laptop
   -> WinPE: wipe, clean Windows, drivers, baseline apps
   -> harvest the hardware hash during deployment
   -> upload to the CORRECT client's Entra tenant
   -> hand off to Autopilot/Intune for identity, policy, apps
```

No incumbent appears to do tenant-aware hash harvesting. OSDCloud injects an
Autopilot profile JSON but presumes prior registration. SmartDeploy has a
*strategic* reason not to build it — their marketing positions Autopilot as the
competitor, so integrating with it undercuts their own pitch.

For a 50-laptop rollout, automating hash capture and tenant-correct upload
recovers roughly 25–37 technician-hours. That is the feature that wins a
bake-off, and it is unclaimed.

⚠️ Verify before betting the roadmap: this is absence of evidence, not proof of
absence. Confirm neither incumbent has shipped it.

### The most defensible gap is the pricing model

Every incumbent charges **per-device, per-year, in perpetuity** for what is, to
an MSP, a **one-time billable event**. The device may never be touched by the
deployment tool again, and the model penalizes the MSP for growing its client
base.

PDQ already sells Deploy & Inventory **per-admin at $1,650/yr** — the
per-technician model is proven in this exact market by the company that now
owns SmartDeploy. They simply have not applied it to imaging.

**Price per technician, per deployment, or per MSP — never per managed device.**

Constraint: ManageEngine's **$2,995/yr unlimited** SKU sets a hard ceiling on
what any deployment-only product can charge an MSP. A competent buyer will find
it. Exceeding that requires being more than a deployment tool — which
multi-tenancy provides.

---

## 4. The capture requirement, and why deploy-time sysprep is the right answer

The requirement is to clone a configured machine using the Windows tooling,
strip the SID, and restore to a clean OOBE. That is `sysprep /generalize /oobe`
plus a DISM capture. IronDeploy does not do this today, and upstream lists it
under "outside the current scope."

Before building the obvious version, note that **the naive workflow is the one
with the known failure modes**:

- **`sysprep /generalize` fails on Appx/MSIX provisioning mismatches**, error
  `0x80073cf2`. If an all-user provisioned package was updated by any user on
  the reference machine, or deprovisioned from the image but not removed for a
  user, generalize fails. Windows 11 24H2 has a well-documented case where
  `Microsoft.SecHealthUI` blocks generalize and resists removal. Validation is
  markedly stricter on current Windows 11 and Server 2025 builds because
  Microsoft leans heavily on Appx/MSIX provisioning and servicing integrity.
- **The rearm counter** historically limits generalize to 3 runs. This is
  largely mitigated on modern Windows 10/11 — and further mitigable via
  `SkipRearm` in the `Microsoft-Windows-Security-SPP` unattend component — so
  treat SmartDeploy's framing of it as an inevitable wall as **overstated but
  directionally real**.

The traditional sysprep-then-capture model also destroys the reference
machine's reusability: you must snapshot and roll back, or rebuild, to patch it
next month.

**SmartDeploy's inversion solves this and is worth copying: capture the
reference machine *without* sysprep, and run sysprep at deploy time on the
target using a generated answer file.** The golden image stays a live,
patchable asset, and generalize runs against a freshly applied image rather
than a machine that has accumulated months of user and Appx state — which is
exactly where `0x80073cf2` comes from.

Two capture modes are worth supporting, and they serve different jobs:

1. **VM-based capture** (SmartDeploy's model) — read the `.vhdx`/`.vmdk` of a
   powered-off VM. Produces a hardware-clean image because a hypervisor guest
   has a tiny synthetic device set.
2. **Online capture of a live physical machine** (ManageEngine's model) — the
   "clone this client's one perfect PC" case. SmartDeploy structurally cannot
   do this. For an MSP inheriting an un-reproducible machine, it is the
   difference between "we can clone it tonight" and "rebuild it in a VM first."

Also worth weighing honestly: the market is moving toward **thin images** —
stock Windows plus drivers plus apps, no golden image to rot. OSDCloud and
Autopilot both work this way, and IronDeploy already does. Golden-image capture
is a migration path and a sales feature, not necessarily the long-term model.

---

## 5. Where IronDeploy stands against the required feature set

Ranked by whether absence kills the sale.

### Tier 0 — cannot ship without

| # | Feature | IronDeploy |
| --- | --- | --- |
| 1 | WinPE boot, GPT/UEFI partition, apply image, boot | ✅ Complete |
| 2 | Automatic driver handling on arbitrary hardware | ✅ Per-model packages |
| 3 | Sysprep/generalize + unattend/OOBE automation | ⚠️ Unattend yes; **no generalize/capture** |
| 4 | Signed, Secure-Boot-compatible media that works with no server | ⚠️ ISO/USB yes; **Secure Boot unresolved** |
| 5 | Disk wipe / erase | ✅ With disk re-verification before erase |

### Tier 1 — required to be competitive

| # | Feature | IronDeploy |
| --- | --- | --- |
| 6 | Post-deploy app install | ✅ Complete |
| 7 | Naming + AD domain join **and** Entra join | ⚠️ AD/ODJ yes; **no Entra join** |
| 8 | Per-deployment reporting / audit trail | ✅ Strong — better than most |
| 9 | **Multi-tenancy / per-client isolation** | ❌ **None. Zero tenant concept anywhere.** |
| 10 | **Tenant-aware Autopilot/Intune handoff** | ❌ **None** |

### Tier 2 — differentiators

| # | Feature | IronDeploy |
| --- | --- | --- |
| 11 | Cloud content delivery, no VPN | ❌ SMB only |
| 12 | Technician-oriented licensing | N/A — business model |
| 13 | Config as versioned, diffable text | ❌ Not today; nobody does this |

### Tier 3 — widely over-weighted, build later

- **USMT** — both incumbents just wrap Microsoft's `ScanState`/`LoadState`.
  Still supported and in the Win11 ADK. But for clients with data in
  OneDrive/M365 its value is declining. Wrap it when a deal demands it; do not
  lead with it. *(This revises the priority given in FORK-ASSESSMENT.md §8,
  which was written before the MSP and Intune requirements were known.)*
- **PXE server** — SmartDeploy ships none at all and defers to WDS. The market
  leader proves you can sell without it. USB is more practical on networks you
  do not control. iPXE remains worth building for your own bench efficiency,
  just not as a competitive requirement.
- **Multicast** — SmartDeploy's own documentation says do not use it below 25
  simultaneous machines. Almost no MSP job reaches that. RFP checkbox, near-zero
  real utility. Deprioritize hard.

---

## 6. Revised recommendation

**Build on IronDeploy.** The license is now the deciding factor, not the code
quality — it is the only candidate that can become a product you sell. The code
quality assessed in FORK-ASSESSMENT.md makes it a viable base; the licensing
makes it the *only* viable base.

Two decisions must be made before writing feature code, because both are
architectural and neither retrofits cheaply:

1. **Multi-tenancy.** There is no tenant, org, customer, or site concept
   anywhere in the codebase, and permissions are a flat global set. Retrofitting
   this touches the data model, auth, the SMB payload layout, and the WinPE
   token model simultaneously. If the product is for MSPs, this is Tier 0, and
   it must be architectural from the first commit.
2. **Business model.** Per-technician versus per-device determines what you
   build and how you price against ManageEngine's $2,995 unlimited ceiling.

Then, in order:

1. **Validate the existing tool** on a VM and two or three representative
   models. Nothing else matters if it does not deploy cleanly on your hardware.
2. **Harden** — TLS defaults and enforcement, unattend localization, ODJ blob
   encryption. Cheap, contained, prerequisite to any pilot.
3. **Multi-tenancy**, if the MSP product is the goal.
4. **Tenant-aware Autopilot handoff.** Highest-ROI unclaimed feature.
5. **Entra join** alongside the existing AD/ODJ path — covers the clients
   Autopilot v2 cannot.
6. **Deploy-time sysprep + capture**, both VM-based and online.
7. **OEM driver-pack automation** — pull Dell/HP/Lenovo enterprise packs
   automatically. Structurally beats SmartDeploy's curated library and fixes
   their top complaint by construction.
8. **Secure Boot resolution** for boot media.
9. **iPXE** for bench efficiency.
10. **USMT** when a deal requires it.

---

## 7. Open verification items

Before committing roadmap or pricing:

| # | Item | Why it matters |
| --- | --- | --- |
| 1 | How deep ManageEngine's Endpoint Central MSP OS deployment actually goes | Determines whether the core opportunity is open or contested. Highest-value single check. |
| 2 | Whether SmartDeploy or ManageEngine already does tenant-aware Autopilot hash upload | Collapses the §3 differentiator if already shipped. |
| 3 | Real SmartDeploy quote at 50 / 200 / 1,000 devices | The per-device figure underpins the pricing-model argument. |
| 4 | ManageEngine "unlimited computers" EULA — does it permit MSP multi-customer use? | Sets the price ceiling. |
| 5 | Current Microsoft guidance on sysprep rearm limits and `SkipRearm` | Underpins §4. |
| 6 | Practitioner sentiment on r/msp specifically | Least-verified material in this analysis. |

⚠️ Sourcing note: the competitive and pricing material in §2–§3 was gathered
through search summarization rather than direct page reads, because the fetch
path was blocked during research. Treat vendor mechanics and pricing as
directionally reliable but verify the items above before acting on them.
The license findings in §1 are **direct** — read from the repositories
themselves — and are the most reliable claims in this document.

---

## Sources

- [MDT retirement notice](https://learn.microsoft.com/en-us/troubleshoot/mem/configmgr/mdt/mdt-retirement)
- [Autopilot manual registration](https://learn.microsoft.com/en-us/autopilot/manual-registration) · [registration overview](https://learn.microsoft.com/en-us/autopilot/registration-overview)
- [Sysprep fails with Microsoft Store apps](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/sysprep-fails-remove-or-update-store-apps)
- [sysprep /generalize fails on Microsoft.SecHealthUI (24H2)](https://learn.microsoft.com/en-us/answers/questions/2265263/sysprep-generalize-fails-due-to-appx-package-micro)
- [OSD PowerShell module](https://github.com/OSDeploy/OSD) · [OSDCloud](https://www.osdcloud.com/)
- [FOG Project](https://github.com/FOGProject/fogproject) · [fog-ipxe](https://github.com/FOGProject/fog-ipxe)
- [SmartDeploy 3.0 under the hood](https://www.smartdeploy.com/blog/smartdeploy-3-0-under-the-hood/) · [Platform Packs](https://smartdeploy.pdq.com/hc/en-us/articles/12982110948251-What-is-a-Platform-Pack) · [Application Packs](https://support.smartdeploy.com/support/solutions/articles/48000960610-what-is-an-application-pack-)
- [ManageEngine OS Deployer](https://www.manageengine.com/products/os-deployer/) · [Endpoint Central MSP OS deployment](https://www.manageengine.com/desktop-management-msp/os-deployment.html)
