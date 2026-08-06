# IronDeploy fork assessment

Evaluation of `lumon-bdip/IronDeploy` (fork of `Syrinoxsis/IronDeploy`) as the
base for an in-house Windows deployment platform, replacing SmartDeploy and a
now-retired MDT workflow.

Assessed at commit `a046661`, 2026-08-06.

---

## 1. Verdict

**Build off the fork. Own it outright — do not treat upstream as a dependency.**

The expensive, high-risk 80% of a bare-metal deployment tool is already built
here, and built with more care than the project's age suggests. Rebuilding
WinPE orchestration, DISM apply, GPT partitioning, offline driver injection,
Offline Domain Join, and a token-scoped control plane from scratch is a
multi-month effort that would rediscover every sharp edge this codebase has
already filed down.

But the upstream project is 12 days old, has one author, 3 stars, and zero
external issues. That is not a knock on the code — it is a statement about
what you can safely bet on. Do not plan a roadmap that depends on upstream
merging your work or still existing in a year.

The practical shape of that:

- Fork it, keep the Apache-2.0 `LICENSE` and `NOTICE` intact, and carry your
  own `main`.
- Pull upstream fixes by cherry-pick while the author is active. Contribute
  back the small, general fixes as a courtesy — not as a delivery mechanism.
- Build USMT, iPXE, and any capture workflow as **new components alongside**
  the existing four, not as edits threaded through them.

The counter-case is worth stating plainly: the three things you most want
(USMT migration, iPXE boot, reference-image capture) are all new components,
and one of them — capture — is *explicitly rejected* by upstream's roadmap.
So a meaningful share of your work is greenfield regardless of this decision.
That argues for extending a good foundation. It does not argue for rewriting
one.

---

## 2. What the fork actually is

| Fact | Value |
| --- | --- |
| Upstream | `Syrinoxsis/IronDeploy` |
| License | Apache 2.0 (fork, modify, keep private, or commercialize — all permitted) |
| Age | First commit 2026-07-26; latest 2026-08-07 |
| Commits | 92 |
| Authors | 1 (`Syrinox`, commits stamped `+0500`) |
| Stars / forks / open issues | 3 / 1 (this one) / 0 |
| Self-declared status | Alpha. "Permanently erases the physical disk." |
| Fork divergence | None. Clean fork, no local commits yet. |

Composition: 45 Python files (IronAPI, FastAPI + SQLAlchemy + SQLite),
21 PowerShell files (WinPE runtime, build tooling, service install),
12 HTML + 13 JS + 10 CSS (web UI), ~10.4k lines of PowerShell.
The WinPE runtime alone is 5,489 lines across the engine, WPF GUI, and config.

Four components with genuinely clean boundaries:

| Component | Responsibility |
| --- | --- |
| SetupWeb | Writes initial server configuration |
| IronAPI | Authorization, deployment plan, state, AD integration |
| WinPE | Operator input, disk erase, offline Windows apply |
| Post-install | Finishes inside the newly installed OS |

---

## 3. Code review

### 3.1 What is genuinely good

**The security model is real, not decorative.** This is the strongest signal
in the codebase and the main reason to build on it rather than start over:

- Browser sessions and WinPE deployment tokens are separate authorization
  mechanisms, with a token phase state machine (`authorized` → `winpe` →
  `postinstall`) enforced server-side in `Core/Api/app/auth.py:361`.
- The SMB payload account is read-only by design, separate from the IronAPI
  service identity, and its credentials are never embedded in the WinPE image
  — they are released per-authorized-deployment
  (`GET /api/deploy/{id}/smb-credentials`).
- Every installer copied into the offline system has its SHA-256 compared
  against the server-approved manifest (`Core/Docs/WINPE.md`, destructive
  phase step 10). A compromised share cannot substitute a payload.
- The target disk is **re-enumerated and re-verified** against the
  operator-confirmed number/model/size snapshot immediately before the erase.
  Losing that race is the classic way a deployment tool eats the wrong disk.
- Secrets written to the WinPE ramdisk are registered and shredded
  (`Register-IronSecretArtifact` / `Clear-IronSecretArtifacts`), including on
  retry after a failed attempt.
- ODJ blobs are short-lived and deleted after acknowledgement.

**Database discipline.** A numbered migration framework with a writer lock and
single-transaction application (`Core/Api/app/database.py`), six migrations
already applied, and a frozen migration-1 schema that is explicitly never
edited. Most tools at this maturity are still doing `CREATE TABLE IF NOT
EXISTS` at startup. This matters directly for USMT, which needs schema
changes.

**Test coverage.** 241 tests across ~6,700 lines, covering the areas that
actually bite: database safety, deployment timeouts, WinPE disk selection,
driver contract, image-apply strategies, ODJ, LDAP naming, access control.

**Operational telemetry beyond MDT.** Per-stage network measurement, negotiated
link speed reported *before* the disk is touched (so a 100 Mbps link caused by
a bad dongle is visible and survives a hard power-off), throughput and byte
counts per stage, and an aggregate final report. MDT never gave you this.

**Staged image apply.** `staged` mode copies the WIM local with unbuffered
robocopy through a `.partial` name, verifies size and SHA-256, then renames and
runs DISM against the local file. The strategy is server-owned and delivered in
the per-deployment manifest, so it is not baked into the boot image. That is
the correct place for that decision.

**The hardware-identity matching is already right.** On the USB-C dongle
problem you described: the inventory logic in
`Core/Api/app/deployments.py:1069-1110` matches on serial number *first* and
only falls back to MAC when no serial is available, and a transient missing
serial explicitly will not overwrite a known-good one. The name history is
surfaced as `known_computer_names` with a `matched_by` discriminator
(`serial_number` | `mac_address`) and presented as a reference, never
auto-applied. That is the behavior you settled on, already implemented.

### 3.2 Findings — things to fix in your fork

**Finding 1 — HTTP and disabled certificate validation are the shipped
defaults. Fix before any pilot.**

`Core/WinPE/Runtime/deploy.config.example.ps1`:

```powershell
$ApiBaseUrl = "http://CHANGE_ME_DEPLOY_SERVER:8000"
$ValidateApiServerCertificate = $false
```

The same API that ships these defaults returns the SMB account password in
cleartext JSON (`Core/Api/app/main.py:1510`). On plain HTTP, anyone on the
deployment VLAN captures the share account. The README calls HTTPS
"preferred"; for a control plane that distributes credentials it is mandatory.
Change the defaults, and have IronAPI refuse to hand out SMB credentials over
a non-TLS connection rather than relying on configuration discipline.

**Finding 2 — ODJ blobs are unencrypted at rest.** Protected only by NTFS
permissions and a short lifetime. Upstream acknowledges this in the roadmap.
An ODJ blob is a domain-join credential; treat it accordingly.

**Finding 3 — the unattend template is localized for the author, not for you.**
`Core/ServerTemplates/Unattend/unattend-win11-template.example.xml` defaults to
`ru-RU` for input/system/UI/user locale, `Central Asia Standard Time`, and
carries Russian comments. The WinPE GUI and web UI are bilingual EN/RU
(`Core/Api/app/static/i18n.js`). Nothing broken — but a localization pass is
day-one work, and it is a fair indicator of whose environment this was built
and tested against.

**Finding 4 — x64 UEFI/GPT only.** No BIOS/MBR path. On upstream's roadmap,
not implemented. Check your remaining fleet before this bites you.

**Finding 5 — stage vocabularies are enum-constrained in the database.**
`deployment_stages.phase` is `CHECK (phase IN ('winpe'))`
(`Core/Api/app/deployments.py:121`) with a matching `Literal["winpe"]` on the
Pydantic model at line 589, and `deployment_network_stages.stage` enumerates
exactly four stage names at line 249. This is good hygiene, but it means every
new deployment stage — including all USMT work — requires a schema migration
plus a model change. Budget for it; the framework to do it cleanly is already
there.

**Finding 6 — SQLite, single writer.** Correct for your scale. Know that it is
the ceiling if you ever run concurrent mass deployments from one server.

**Finding 7 — bus factor 1, twelve days of history.** The most important
finding, and not a code problem. There is no track record, no external issue
history, and no second maintainer. Whoever on your team owns PowerShell and
Python needs to actually own this, because upstream may not be there.

---

## 4. Gap analysis against what you need

| Capability | Status in fork |
| --- | --- |
| Bare-metal Windows 10/11 install from WIM/ESD | Complete |
| Driver package per model | Complete |
| App install post-deployment | Complete |
| Computer naming + AD Offline Domain Join | Complete |
| Name history by MAC/serial | Complete, and handled correctly |
| Operator authorization / RBAC | Complete |
| **USMT user-state migration** | **Absent — zero code** |
| **iPXE network boot** | **Effectively absent** |
| **Reference-image capture** | **Absent, and explicitly out of upstream scope** |

That third row is the one to think hardest about. You are coming from
SmartDeploy, whose entire model is golden-image capture plus platform packs.
IronDeploy deploys from an unmodified Microsoft installation image and
composes the result from drivers, unattend, and app installers — and upstream's
roadmap lists capturing a configured reference machine under **"Outside the
current scope."** That is a deliberate design position, not a backlog item.

If your workflow depends on capturing a tuned reference machine, that is either
a fourth new component you build, or a change in how you produce a standard
build. Worth deciding early — it shapes everything else.

Context worth having: Microsoft declared MDT **immediately retired on
2026-01-06**, with no updates, security fixes, or support. Your current MDT
workflow is running on a dead tool, so the pressure to move is real. USMT
itself is *not* retired — it still ships in the Windows 11 ADK and remains the
supported path for user-state migration.

---

## 5. USMT integration design

### 5.1 The blocking design tension

**IronDeploy's SMB account is read-only by design, and USMT capture requires
write access.** This is stated in the README, the architecture doc, and the API
doc, and it is the single most important constraint on this feature.

You cannot reuse the existing share credential for a migration store. Adding
USMT means introducing a second, write-scoped credential path — which means
deciding how it is scoped, issued, and revoked. Get this decision right first;
everything else in the feature follows from it.

Recommended: a separate `MigStore` share with its own account, and a
**per-migration write grant** issued by IronAPI against the deployment token,
scoped to a single store directory. Do not create a broadly writable share that
every machine in the fleet can reach — a migration store is a full copy of a
user's documents and credentials-adjacent data.

### 5.2 The missing component

Everything in IronDeploy today runs either in WinPE or in the *newly installed*
OS. **Nothing runs in a live source machine's OS before deployment.** Online
capture therefore requires a fifth component:

```
IronCapture (new)  — runs in full Windows on the source machine, elevated
       |
       +--> authenticates to IronAPI, requests a migration store + write grant
       +--> runs scanstate.exe against the MigStore share
       +--> reports store ID, size, user list, and hash back to IronAPI
```

Restore is much easier and needs no new component. `postinstall.ps1` is invoked
by `SetupComplete.cmd` **before first user logon** — which is precisely where
`loadstate.exe` belongs. Extend the existing post-install script rather than
inventing a new phase.

### 5.3 Work breakdown

**Schema (migration 7):**

- New `migration_stores` table: id, source serial/MAC, source computer name,
  captured user list, store path, size, SHA-256, encryption key reference,
  created/expires timestamps, state, consuming deployment id.
- Relax `deployment_stages.phase` from `CHECK (phase IN ('winpe'))` to include
  `'capture'` and `'postinstall'`, and widen the matching Pydantic `Literal`.
- Add `usmt_capture` / `usmt_restore` to the `deployment_network_stages` stage
  enum so migration transfers get the same per-stage throughput telemetry the
  image apply already gets. This is worth doing — a 40 GB profile over a
  100 Mbps dongle is exactly the failure you want visible.

**IronAPI:**

- Store lifecycle endpoints: create, claim, release, expire, delete.
- Write-grant issuance scoped to one store, tied to the deployment token.
- Encryption key custody. USMT supports `/encrypt /key:`; generate per-store,
  store server-side, release only to the authorized restoring deployment.
  A migration store left unencrypted on a share is a data-breach waiting for
  an audit.
- Retention policy with automatic expiry. These stores are large and sensitive;
  they must not accumulate.
- Store-to-hardware association. Reuse the existing serial-first matching from
  `find_known_computer_names` — it already has the precedence right, and it is
  what makes "this new laptop is replacing that old one" work reliably even
  through a shared dongle.

**IronCapture (new agent):**

- Elevated PowerShell, same certificate-trust and API-timing patterns already
  established in `IronDeploy.Engine.ps1` — reuse them, do not reinvent.
- Pre-flight: free space on store, profile size estimate, users to migrate,
  BitLocker state, whether the target share is reachable at adequate speed.
- Wrap `scanstate.exe` with progress reporting into IronAPI stages.

**WinPE GUI:**

- Optional fourth wizard step. The wizard is currently a fixed 3-step flow
  (`IronDeploy.Gui.ps1:1207`, `[ValidateSet(1,2,3)]`) with three hardcoded
  step indicators — adding a step touches the validate-set, the indicators,
  the navigation closure, and both EN and RU string tables.
- Offer any migration store matched to this hardware, showing source machine,
  capture date, size, and user list. Given your dongle experience: **show it,
  let the operator confirm it, never auto-apply it.** Restoring the wrong
  user's profile onto a machine is worse than restoring nothing.

**Post-install:**

- `loadstate.exe` before first logon, with `/mu` `/md` support for local→domain
  account remapping.
- Report per-user restore results into the existing stage/telemetry model.

**Deployment/ops:**

- USMT binaries come from the ADK. Do not commit them — follow the existing
  pattern of referencing a local ADK install and staging at prepare time.
  `1. Prepare-IronDeploy.ps1` already resolves ADK paths.
- **Pin your ADK version.** ADK `10.1.28000.1` ships a `loadstate` bug
  (USMT 10.0.26100.1) that fails profile restore with `ERROR_PATH_NOT_FOUND`
  (0x3) on Windows 11. `10.1.26100.2454` is the known-good version. Verify
  against your own hardware before standardizing.

### 5.4 Scope note

Because IronDeploy fully erases the target disk, hardlink migration
(`/hardlink /nocompress`) — MDT's fast in-place refresh — is not available.
Network-store migration is the only viable mode here. That is a real speed
difference from an MDT refresh scenario, and it makes the link-speed telemetry
above more valuable, not less.

---

## 6. iPXE integration design

This is the easy one, and IronDeploy is unusually well-positioned for it.

### 6.1 Current state

Essentially nothing. The only PXE-aware code is a `-UpdatePxeBundle` switch on
`Core/Tools/Build-IronDeployWinPE.ps1` that copies the published WIM to
`Core/dist/PXE/irondeploy/boot.wim` after verification. There is no iPXE
script, no TFTP integration, no HTTP boot endpoint. Documentation says only
"publish the WIM through WDS/PXE."

### 6.2 Why it fits well

IronAPI is already an HTTP server with per-MAC hardware knowledge and an
authorization model. Serving the iPXE boot script from IronAPI turns network
boot into a control-plane decision rather than a static file drop:

```
firmware PXE ROM --> chainload ipxe.efi
      |
      +--> GET https://irondeploy/api/boot/ipxe?mac=${net0/mac}
              |
              +--> IronAPI decides: deploy / capture-first / local boot / refuse
              |
              +--> wimboot + BCD + boot.sdi + boot.wim over HTTP
```

Per-MAC boot policy composes directly with the features you already have. A
machine with a pending migration store can be routed straight to the capture or
restore path. A machine not authorized for deployment can be told to boot
local, which is a meaningful safety property for a tool that erases disks.

HTTP transport also beats TFTP substantially on WIM delivery, and the existing
link-speed telemetry gives you the data to prove it.

### 6.3 The real constraint: Secure Boot

Be clear-eyed about this before committing:

- **`wimboot` is signed for UEFI Secure Boot** (Microsoft-signed as of 2.7.2).
  That part is fine.
- **`ipxe.efi` generally is not.** Standard builds are unsigned, which means
  either disabling Secure Boot on target hardware or sourcing a signed build
  (Broadcom publishes one signed with a Microsoft certificate).

Disabling Secure Boot fleet-wide to enable a deployment tool is a poor trade —
especially where Windows 11 hardware requirements and BitLocker policy are in
play. Resolve the signed-binary question during the pilot, not after.

### 6.4 Work breakdown

- `GET /api/boot/ipxe` on IronAPI returning a generated iPXE script, keyed on
  MAC, with policy decisions from the existing computer inventory.
- Serve `wimboot`, BCD, `boot.sdi`, and `boot.wim` over HTTP from IronAPI or a
  co-located web root; keep the existing SHA-256 verification discipline.
- Extend `-UpdatePxeBundle` to publish the full boot set, not just `boot.wim`.
- DHCP configuration guidance for the standard "chainload iPXE once, then serve
  the script" conditional. IronDeploy should not manage DHCP — but it should
  document the exact options.
- Decide and document the Secure Boot position.

---

## 7. Risks

| Risk | Severity | Mitigation |
| --- | --- | --- |
| Alpha software that erases disks | High | VM and disposable-hardware validation before any production pilot. Non-negotiable. |
| Upstream abandonment (1 author, 12 days) | High | Own the fork. Assign an internal maintainer. Never block on upstream. |
| Credentials over plain HTTP by default | High | Fix defaults; enforce TLS server-side before pilot. |
| No reference-image capture | Medium | Decide early whether you need it; it is a fourth new component. |
| ADK/USMT version bugs | Medium | Pin ADK 10.1.26100.2454; validate restore on your hardware. |
| Secure Boot vs unsigned iPXE | Medium | Resolve signed-binary sourcing during pilot. |
| x64 UEFI/GPT only | Low–Medium | Audit remaining BIOS/MBR fleet. |
| Migration stores as sensitive data | Medium | Per-store encryption, scoped write grants, enforced retention. |

---

## 8. Recommended sequence

1. **Validate before building.** Run the existing tool end-to-end on a VM and
   two or three representative models. Confirm it deploys cleanly on your
   hardware before investing engineering time in extending it. If it does not,
   the entire decision changes.
2. **Harden the fork.** TLS defaults and enforcement, localization pass on the
   unattend template, ODJ blob encryption. Small, contained, and prerequisite
   to any pilot.
3. **iPXE.** Fastest payoff, lowest architectural risk, no schema changes, and
   it immediately improves the daily workflow. Resolve Secure Boot here.
4. **USMT restore path first, then capture.** Restore extends existing
   post-install code; capture requires the new agent and the write-grant
   design. Build the easier half first and prove `loadstate` works against
   stores you produce by hand, before automating capture.
5. **Then decide on reference-image capture**, informed by how much the
   drivers-plus-apps-plus-unattend composition model actually covers in
   practice. It may cover more than the SmartDeploy workflow led you to expect.

Steps 1 and 2 are worth doing regardless of the final build-versus-buy call —
they are cheap, and they produce the evidence that makes the call defensible.

---

## Sources

- [MDT immediate retirement notice — Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/mem/configmgr/mdt/mdt-retirement)
- [USMT overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows/deployment/usmt/usmt-overview)
- [wimboot — iPXE](https://ipxe.org/wimboot)
- [iPXE Secure Boot discussion](https://github.com/ipxe/ipxe/discussions/358)
- [USMT loadstate ERROR_PATH_NOT_FOUND on ADK 10.1.28000.1](https://industrialmonitordirect.com/blogs/knowledgebase/resolving-usmt-loadstate-error-71-windows-11-migration)
