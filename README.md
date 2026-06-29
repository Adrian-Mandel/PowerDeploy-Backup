<img width="4167" height="1042" alt="powerdeploy_banner_transparent" src="https://github.com/user-attachments/assets/9771a0f8-0b5a-4c8c-b7c9-e45499cdb9c8" />



# PowerDeploy

**Retire your on-premises print server — and deploy every app and printer from the cloud — without buying new hardware, standing up a new server, or weakening your security.**

PowerDeploy lets a cloud-managed Windows environment (Microsoft Entra ID + Intune) deliver applications and network printers to every device with **no on-premises server, no special hardware, and no new vendor or license.** It runs inside the Microsoft cloud you already pay for.

---

## Why it exists

Moving identity and devices to the cloud is mainstream now, and it mostly works. **Printing is where that path tends to break.**

Microsoft's cloud printing service, **Universal Print**, generally hands you two bad doors:

- **Replace your printers** with a short list of approved models, or
- **Keep the printers you own** by putting a connector *server* back in your building and loosening security settings to register them.

Neither is a trade most organizations can comfortably defend — and the workaround reintroduces the very on-prem infrastructure that going cloud was meant to eliminate. *(In our own fleet of ~150 printers — mostly recent HP hardware — exactly **one** qualified for the clean path.)*

The rest of the industry settled on a **third door**: deliver each printer to a computer the same way you deliver any normal application, through Intune — **no server at all.** That approach is standard and proven. The catch is that doing it by hand is slow to build, opaque to troubleshoot, and doesn't scale. **PowerDeploy is that proven approach, engineered to be fast, reliable, and manageable** — for printers and for ordinary software alike.

### What you get

- **Retire the on-prem print server** without replacing a single printer.
- **No new server, hardware, vendor, or license** — it uses the Intune and Azure storage you already have.
- **Migrate at your own pace.** Keep the existing print server running and move printers over one at a time (about 5–10 minutes each). No big-bang cutover to gamble on.
- **Deployments you can actually troubleshoot**, backed by a documented, version-controlled catalog you own — instead of dozens of hand-built, hard-to-update entries scattered across Intune.

PowerDeploy has run in production with a pilot group for roughly **seven months** and is now in wider rollout.

> The sections below move from the big picture into full technical depth. Skim the top; dig in as far as you need.

---

## Table of contents

- [What you can do with it](#what-you-can-do-with-it)
- [Printing: solving what Universal Print can't](#printing-solving-what-universal-print-cant)
- [Packaging and managing your assets](#packaging-and-managing-your-assets)
- [How it works (under the hood)](#how-it-works-under-the-hood)
  - [The runner pattern](#the-runner-pattern)
  - [End-to-end flow](#end-to-end-flow)
  - [Where things live: scripts vs. payloads vs. config](#where-things-live-scripts-vs-payloads-vs-config)
- [With and without the Company Portal](#with-and-without-the-company-portal)
- [Components](#components)
- [Configuration model](#configuration-model)
- [Deployment modes (public vs. private fork)](#deployment-modes-public-vs-private-fork)
- [Logging](#logging)
- [Security](#security)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [License](#license)
- [Support](#support)

---

## What you can do with it

- **Deploy applications** via WinGet, MSI (from Azure Blob or a direct URL), EXE, direct URL download, or fully custom installer scripts.
- **Deploy network (IP) printers** with centrally managed driver packs and per-printer JSON configuration.
- **Uninstall almost anything** through a single multi-method uninstaller.
- **Generate the Intune `.intunewin` package, install/uninstall commands, and detection script** automatically from a guided wizard.
- **Push organization configuration** (storage account, container keys, repo settings) to endpoints via Intune remediation scripts.
- **Manage the Windows registry and optional features** with safe, ACL-aware operations.
- **Run everything locally, on demand** for testing and urgent fixes — independent of the management tool's sync schedule.

Each deployment is **available both with and without the Company Portal**: the same definition can be assigned as a self-service or required app in Intune *and* run on demand by a technician directly on the device.

---

## Printing: solving what Universal Print can't

Microsoft's cloud answer for printing is **Universal Print**, and if you've evaluated it for a real environment you've likely hit the same walls we did:

- **Hardware support is the exception, not the rule.** Universal Print requires printers with native support. In our fleet of ~150 printers — most modern and recently in production — only about **10% qualify**, and of the HP printers we standardize on, exactly **one** supports it natively. Going all-in would mean buying from a narrow approved list and replacing hardware that works perfectly well.
- **The workaround defeats the purpose.** Microsoft's bridge for non-UP printers requires an **on-premises connector server** — reintroducing the on-prem print infrastructure that going cloud was supposed to retire.
- **It asks you to weaken your security posture.** Registering printers requires loosening settings many organizations (us included) are unwilling to loosen.
- **Deployment is unreliable and hard to troubleshoot**, even where the hardware is supported.

**PowerDeploy replaces the print server, not the printers.** It deploys directly to standard IP printers with no on-prem server, no hardware allow-list, and no change to your security posture:

- Driver packs live as **version-managed files in your Azure Blob Storage** — a central driver library, not drivers embedded in dozens of packages.
- Each printer is **one entry** in a `PrinterData.json` manifest: name, IP/port, and which driver to use.
- At deploy time the endpoint pulls the right driver pack, stages it with `pnputil`, and creates the port and print queue — fully unattended.
- Define a printer once and it's deployable everywhere: assignable through the **Company Portal** *and* installable **on demand** by a technician.

The result is the cloud printer management Universal Print promised — that actually works with the printers you already own.

---

## Packaging and managing your assets

Most of the recurring cost of fleet deployment isn't the install itself — it's the **packaging** and the **ongoing upkeep**. PowerDeploy is built to make both cheap.

**Packaging is a guided, minutes-long task.** Run [`Setup.ps1`](Setup.ps1), pick (or define) an app or printer, and the wizard:

- optionally **test-installs it on the local machine first**, so you validate the configuration before you ship it;
- **builds the `.intunewin` package for you** — [`Make-InTuneWin`](Setup.ps1) auto-downloads Microsoft's `IntuneWinAppUtil.exe` and runs it silently, so you never touch the prep tool by hand;
- generates the **install command, uninstall command, and detection script** — with parameters Base64-encoded for clean Intune compatibility, written to text files and copied to your clipboard;
- **prints exactly what to paste** into each field of the Intune Win32 app form.

> **What's automated vs. manual:** PowerDeploy automates the *packaging and artifact generation*. Creating and uploading the Win32 app in the Intune portal remains a deliberate, **guided manual step** — the wizard walks you through it click by click.

**Management is editing a catalog, not maintaining packages.** Your deployable assets live in JSON manifests — a shared public catalog in this repo and your organization's private catalog in Azure Blob:

- **Add or change an app/printer** → edit one JSON entry. The Intune app entry never has to be rebuilt.
- **Change how an installer behaves** → edit a script and commit. Endpoints pick it up on their next run.
- **Ship a new payload** (updated MSI, newer driver pack) → replace the file in Azure Blob. Nothing in Intune changes.
- **Reuse across the fleet** → define an asset once and deploy it everywhere, via Company Portal or on demand.

This is the difference between maintaining *dozens of brittle, individually-built Intune packages* and maintaining *one catalog behind a runner that always pulls the current version.*

---

## How it works (under the hood)

### The runner pattern

The central design decision is **decoupling orchestration from payload hosting**.

- **Orchestration** (the *logic* — what to install and how) lives in this Git repository.
- **Payloads** (the *bits* — installers, driver ZIPs) live in WinGet or your Azure Blob Storage.
- **The management tool** (Intune / RMM / SCCM) only ever holds [`Git-Runner_TEMPLATE.ps1`](Templates/Git-Runner_TEMPLATE.ps1) — a small, rarely-changing launcher — wrapped in a `.intunewin` package.

When an endpoint runs the package (intended to run under **SYSTEM**, e.g. when triggered by Intune), the runner:

1. **Ensures Git is present** — installs Git for Windows if missing, fetching the latest 64-bit release from the git-for-windows API and installing it silently. The install is guarded by a named mutex (`Global\PowerDeploy_GitInstall`, 300s timeout, with abandoned-mutex recovery) so parallel deployments don't collide.
2. **Clones or pulls** the target repo into its working directory (`<WorkingDirectory>\<RepoNickName>` — both supplied as parameters; for real deployments the working directory is the mode-specific folder under `C:\ProgramData`, e.g. `C:\ProgramData\PowerDeploy--PRODUCTION`). Local drift is stashed first (optional, on by default), and a missing `.git` is **self-healed** via init/fetch/reset rather than failing.
3. **Decodes its parameters** — Intune-friendly Base64-encoded JSON is decoded into a normal PowerShell parameter string.
4. **Locks down permissions** — runs [`Security_Manager.ps1`](Other_Tools/Security_Manager.ps1) to enforce strict ACLs (SYSTEM + Administrators only) on a fixed set of folders (the Intune log folder, the repo root, and the working directory's `TEMP` and `LOGS` folders), and delegates registry lock-down of `HKLM\SOFTWARE\PowerDeploy` to [`Configure-Registry.ps1`](Configurators/Configure-Registry.ps1). This runs **before and after** the target script as a tamper check.
5. **Invokes the target script** (an installer, uninstaller, printer install, etc.) with the decoded parameters.
6. **Logs execution and propagates a meaningful exit code** that Intune can act on.

Because the runner *pulls the latest commit each time it runs*, iterating on a deployment is just editing a script and committing — the Intune app entry never changes.

### End-to-end flow

```
TECHNICIAN (Setup.ps1, run elevated)
   │
   ├─ Reads org config via OrganizationCustomRegistryValues-Reader (HKLM\SOFTWARE\PowerDeploy)
   ├─ Picks a deployment mode (public/private × dev/main → which repo & branch)
   ├─ Selects an app/printer from JSON (or adds a new one)
   ├─ (optional) Test-installs locally on this machine
   │
   ├─ Make-InTuneWin  ── wraps Git-Runner_TEMPLATE.ps1 ──► .intunewin
   └─ Generate_Install-Command.ps1 ──► install/uninstall commands
                                       + detection script
                                       (params Base64-encoded, copied to clipboard)
                 │
                 ▼
        Technician creates the Win32 app in Intune (guided, manual)
                 │
                 ▼
ENDPOINT (SYSTEM context, triggered by Intune or run on demand)
   │
   └─ Git-Runner_TEMPLATE.ps1
        ├─ install Git (mutex-guarded) → clone/pull repo
        ├─ decode Base64 params
        ├─ Security_Manager → lock down ACLs (before & after)
        └─ run target script, e.g. General_JSON-App_Installer.ps1
                 │
                 ├─ WinGet            → WinGet catalog
                 ├─ MSI / EXE / URL   → Azure Blob (SAS or AAD) or direct URL
                 └─ Custom_Script     → Office, Dell Command Update, .NET, …
```

### Where things live: scripts vs. payloads vs. config

| Concern | Source of truth | Delivered to endpoint by |
|---|---|---|
| **Deployment logic / scripts** | This Git repo (public or your private fork) | `git clone` / `git pull` at run time |
| **App payloads** | WinGet catalog, Azure Blob Storage, or a direct URL | WinGet, or `DownloadFrom-AzureBlob-*` at run time |
| **Printer drivers** | Driver ZIPs in Azure Blob Storage | Azure Blob download → extract → `pnputil` |
| **What to install (manifest)** | `ApplicationData.json` / `PrinterData.json` (public copy in repo, private copy in Blob) | Read at run time |
| **Org configuration** | `HKLM\SOFTWARE\PowerDeploy` registry | Set once via Intune remediation scripts |

---

## With and without the Company Portal

The same deployment serves two execution paths from one definition:

- **With Company Portal (managed):** The Win32 app generated by the wizard is assigned in Intune. End users install it self-service from the Company Portal, or it's pushed as required — with a detection script reporting compliance.
- **Without Company Portal (on demand):** Because the repo and scripts are cloned locally under `C:\ProgramData`, a technician can run `Setup.ps1` and install the same app or printer immediately — useful for testing a new package or fixing a machine right now, with no sync-cycle wait.

---

## Components

**Application installers** (general-purpose, parameter-driven):

- [`General_WinGet_Installer.ps1`](Installers/General_WinGet_Installer.ps1) — robust WinGet installs: bootstraps WinGet via `Install-WinGet.ps1` if missing, validates the App ID, re-detects after install, and on failure **clears the WinGet source cache and retries**, with a final multi-attempt re-check loop (default 15).
- [`General_MSI_Installer.ps1`](Installers/General_MSI_Installer.ps1) — silent MSI with verbose logging, configurable timeout (process-kill on hang), MSI exit-code interpretation (0/3010 success; 1602/1603/1618/1619/1639 mapped), and pre/post-install registry verification.
- [`General_EXE_Installer.ps1`](Installers/General_EXE_Installer.ps1) — EXE installs with **installer-framework auto-detection** (InnoSetup, NSIS, InstallShield, WiX Burn, Setup Factory, Advanced Installer, with a raw-byte fallback) to infer silent switches; supports custom expected exit codes and optional wait-for-process.
- [`General_URL_DL_Installer.ps1`](Installers/General_URL_DL_Installer.ps1) — downloads from a URL (file or ZIP, filename derived from the `Content-Disposition` header), extracts, and hands off to the MSI/EXE installer.
- [`General_JSON-App_Installer.ps1`](Installers/General_JSON-App_Installer.ps1) — **the orchestrator.** Looks up an app in the JSON manifest (public template, falling back to a private Azure Blob copy), installs the app's declared prerequisites first, and dispatches by `InstallMethod` (`WinGet`, `MSI-Private-AzureBlob`, `EXE-Private-AzureBlob`, `MSI-Online`, `URL_Download`, `Custom_Script`).

**Custom multi-step installers:**

- [`InstallApp-MS_Office-FullClean.ps1`](Installers/InstallApp-MS_Office-FullClean.ps1) — Microsoft 365 Apps with a full clean: scrapes the current Office Deployment Tool link, builds dynamic ODT XML (channel, architecture, language, per-app exclusions, NoTeams variant), removes any existing install, then installs silently.
- [`InstallApp-DellCommandUpdate-FullClean.ps1`](Installers/InstallApp-DellCommandUpdate-FullClean.ps1) — an ordered 4-step flow: remove any prior DCU (across WinGet/Appx/CIM identities), uninstall .NET 8, install a pinned .NET 8 Desktop Runtime, then install DCU via WinGet.
- [`Install-DotNET.ps1`](Installers/Install-DotNET.ps1) — routes by version: .NET Framework 3.5 via Windows Optional Features; .NET 5+ (Runtime/Desktop/SDK/AspNet, generic or pinned) via WinGet.
- [`Install-WinGet.ps1`](Installers/Install-WinGet.ps1) — resilient WinGet bootstrap that detects SYSTEM vs. user context and tries multiple install methods in sequence until WinGet works.

**Printers:**

- [`General_IP-Printer_Installer.ps1`](Installers/General_IP-Printer_Installer.ps1) — reads `PrinterData.json`, downloads the driver ZIP from Azure Blob, stages the driver with `pnputil /add-driver`, and creates the port and print queue. Auto-relaunches in 64-bit PowerShell when needed, supports JSON-defined driver overrides, and is idempotent (skips/replaces existing drivers, ports, and printers).
- [`Uninstall-Printer.ps1`](Uninstallers/Uninstall-Printer.ps1) — removes a printer by name (`Remove-Printer` with a WMI fallback), deletes the associated port, and re-checks to confirm removal.

**Uninstallers:**

- [`General_Uninstaller.ps1`](Uninstallers/General_Uninstaller.ps1) — one tool, many methods: WinGet, MSI uninstall strings, registry, CIM/WMI (`Win32_Product`), and AppX/provisioned packages. `UninstallType` selects a method or `All` (runs every method, then re-verifies removal). An optional hardened mode runs each method as a monitored, timeout-bounded process.
- [`Adobe_Uninstaller_Suite/`](Uninstallers/Adobe_Uninstaller_Suite) — full Adobe cleanup that stops Adobe processes, runs CIM uninstalls, and bundles Adobe's official uninstaller, Genuine Cleaner, and Creative Cloud Uninstaller.

**Downloaders (Azure Blob):**

- [`DownloadFrom-AzureBlob-SAS.ps1`](Downloaders/DownloadFrom-AzureBlob-SAS.ps1) — SAS-token auth requiring no interactive login, so it can run unattended / in SYSTEM context. Accepts a full SAS URL or storage-account + container + token.
- [`DownloadFrom-AzureBlob-AADauth.ps1`](Downloaders/DownloadFrom-AzureBlob-AADauth.ps1) — Azure AD / connected-account auth (runs in user context, uses the `Az` modules).

**Configurators:**

- [`Configure-Registry.ps1`](Configurators/Configure-Registry.ps1) — read / read-all (breadth-first subtree walk) / backup / modify / ACL lock-down (SYSTEM + Administrators only). Backup auto-selects the 32- or 64-bit `reg.exe` view by process architecture; lock-down operates on the 64-bit view.
- [`Configure-WindowsOptionalFeatures.ps1`](Configurators/Configure-WindowsOptionalFeatures.ps1) — enable/disable Windows optional features, with a post-action re-check and optional auto-restart when a feature requires it.

**Templates** (cloned to endpoints and/or used to generate artifacts):

- [`Git-Runner_TEMPLATE.ps1`](Templates/Git-Runner_TEMPLATE.ps1) — the endpoint runner described above.
- [`Detection-Script-Application_TEMPLATE.ps1`](Templates/Detection-Script-Application_TEMPLATE.ps1) / [`Detection-Script-Printer_TEMPLATE.ps1`](Templates/Detection-Script-Printer_TEMPLATE.ps1) — Intune detection scripts.
- [`General_RemediationScript-Registry_TEMPLATE.ps1`](Templates/General_RemediationScript-Registry_TEMPLATE.ps1) / [`OrganizationCustomRegistryValues-Reader_TEMPLATE.ps1`](Templates/OrganizationCustomRegistryValues-Reader_TEMPLATE.ps1) — push and read org config in the registry.
- [`ApplicationData_TEMPLATE.json`](Templates/ApplicationData_TEMPLATE.json) / [`PrinterData_TEMPLATE.json`](Templates/PrinterData_TEMPLATE.json) — manifest formats.

**Tooling:**

- [`Setup.ps1`](Setup.ps1) — the technician's main entry point (guided wizards + local install/uninstall + config-remediation generation).
- [`Generate_Install-Command.ps1`](Other_Tools/Generate_Install-Command.ps1) — builds the Base64-encoded Intune install/uninstall commands and detection scripts.
- [`Security_Manager.ps1`](Other_Tools/Security_Manager.ps1) — enforces strict ACLs on PowerDeploy folders and registry (with VM-aware graceful skipping where ACL protection isn't supported).

---

## Configuration model

Per-organization settings live in the registry under **`HKLM\SOFTWARE\PowerDeploy`**, organized into three subkeys. They're typically deployed fleet-wide using the **Intune remediation scripts** that `Setup.ps1` can generate, and read at run time by [`OrganizationCustomRegistryValues-Reader_TEMPLATE.ps1`](Templates/OrganizationCustomRegistryValues-Reader_TEMPLATE.ps1).

| Subkey | Value | Purpose |
|---|---|---|
| `\General` | `StorageAccountName` | Azure Storage account hosting payloads & private JSON |
| `\General` | `CustomRepoURL` | Your private fork's Git URL (for production) |
| `\General` | `CustomRepoToken` | OAuth token for the private repo (optional) |
| `\Printers` | `PrinterDataJSONpath` | Blob path to `PrinterData.json` |
| `\Printers` | `PrinterContainerSASkey` | SAS token for the printers container |
| `\Applications` | `ApplicationDataJSONpath` | Blob path to `ApplicationData.json` |
| `\Applications` | `ApplicationContainerSASkey` | SAS token for the applications container |

The hive is ACL-locked to SYSTEM + Administrators by the Security Manager (via `Configure-Registry.ps1`). The registry holds *where to find things* (storage account, SAS keys, JSON paths, repo URL/token); the actual per-asset printer and app definitions live in the Azure Blob JSON manifests.

**JSON manifests** describe *what* is available to deploy. Apps come from two manifests merged at run time: a **public** `ApplicationData.json` in this repo (community-maintained) and a **private** copy in your Azure Blob (your org's custom/proprietary apps). If the private copy can't be reached, lookup gracefully proceeds with the public catalog alone. Each entry declares an `InstallMethod` (`WinGet`, `MSI-Private-AzureBlob`, `EXE-Private-AzureBlob`, `MSI-Online`, `URL_Download`, `Custom_Script`) plus the fields that method needs, and optional `PreRequisites`.

---

## Deployment modes (public vs. private fork)

PowerDeploy is meant to be **forked into a private organization repo**. The public repo carries shared logic and the community app catalog; your private fork carries your org's customizations and is what production endpoints pull from.

`Setup.ps1` asks which **deployment mode** an artifact should target — a **public/private × dev/main matrix** — which selects the repo + branch the generated package will pull from at run time:

| Mode | Repo | Branch | Use for |
|---|---|---|---|
| **Public – Development** | Official public repo | `dev` | Trying in-progress shared code |
| **Public – Testing** | Official public repo | `main` | Validating released shared code |
| **Private – Development** | Your org fork | `dev` | Testing your own changes |
| **Production** | Your org fork | `main` | Real deployments |

Each mode installs into its own `C:\ProgramData\PowerDeploy--<MODE>` working directory, so test and production payloads stay isolated on the same machine.

> Detailed setup of the private fork, Azure Blob containers, and SAS/AAD configuration is intended to be documented separately as the project matures.

---

## Logging

Every script writes structured logs under its working directory, e.g. `C:\ProgramData\PowerDeploy--<MODE>\Logs\`, split by area. The named areas below are examples — scripts also write to `Download_Logs`, `Uninstaller_Logs`, `Security_Logs`, `Generator_Logs`, and others:

`Git_Logs`, `Installer_Logs`, `Detection_Logs`, `Config_Logs`, `Setup_Logs`, …

Each entry is timestamped and tagged with a severity level (`INFO`, `WARNING`, `ERROR`, `SUCCESS`, plus a `DRYRUN` level) and color-coded in the console. Because everything lands in a predictable, per-area location, troubleshooting a failed deployment is reading one log rather than correlating across the Intune Management Extension logs.

---

## Security

- **Designed to run in SYSTEM context** on managed endpoints, with the working directory under `C:\ProgramData` (not visible to standard users by default). *(SYSTEM is a deployment assumption — e.g. Intune-invoked — not something the scripts self-enforce.)*
- **ACL enforcement** via the Security Manager: a fixed set of working folders is restricted to SYSTEM + Administrators with inheritance broken, and the `HKLM\SOFTWARE\PowerDeploy` hive is locked down via `Configure-Registry.ps1`. Enforcement runs before *and* after the target script as a tamper check, and verifies state before mutating ACLs.
- **Malformed-path validation** guards against illegal characters, reserved device names, over-length paths, and bad formats before file/registry operations run.
- **Azure Blob access** uses scoped SAS tokens (or AAD for user-context scenarios) rather than embedded account keys.

> **Note for reviewers:** generated commands pass parameters to the runner as Base64-encoded JSON, which is decoded and invoked via `Invoke-Expression`. Path validation checks *syntax*, not injection; treat the repo and the generated parameters as trusted inputs.

---

## Repository layout

```
PowerDeploy/
├─ Setup.ps1                  # Technician entry point (run elevated)
├─ Setup_RUNNER.bat
├─ Installers/                # WinGet, MSI, EXE, URL, JSON orchestrator, custom installers, IP printer
├─ Uninstallers/              # General multi-method uninstaller, printer, Adobe suite
├─ Downloaders/               # Azure Blob (SAS + AAD)
├─ Configurators/             # Registry + Windows optional features
├─ Templates/                 # Git runner, detection/remediation scripts, JSON manifests
├─ Other_Tools/               # Install-command generator, Security Manager, utilities
├─ Tests/
├─ LICENSE.md  /  NOTICE.md
└─ README.md
```

---

## Getting started

> High-level only — assumes Intune + an Azure Storage account.

1. **Fork** this repo into your organization's private repo (for production use).
2. **Stand up Azure Blob Storage**: containers for application payloads and printer drivers, plus your private `ApplicationData.json` / `PrinterData.json`.
3. **Generate and deploy the config remediation** from `Setup.ps1` to populate `HKLM\SOFTWARE\PowerDeploy` on your fleet (storage account, SAS keys, JSON paths, repo URL).
4. **Run `Setup.ps1` elevated** and follow a wizard to add an app or printer: it can test the install locally, then produce the `.intunewin`, the install/uninstall commands, and the detection script.
5. **Create the Win32 app in Intune** using those generated artifacts, and assign it.

---

## License

Licensed under the **Apache License 2.0** — see [LICENSE.md](LICENSE.md). Trademark and attribution terms are in [NOTICE.md](NOTICE.md).

Copyright © Santa Cruz County Office of Education.

## Support

For issues and feature requests, please use the [GitHub Issues](https://github.com/Santa-Cruz-COE/PowerDeploy/issues) page.

---

**Source:** <https://github.com/Santa-Cruz-COE/PowerDeploy>

<sub>Portions of this README were drafted with AI assistance and verified against the source scripts; it describes an evolving project, so confirm specifics against the code before relying on them in production.</sub>

<p align="center">
  <img src="https://github.com/user-attachments/assets/38b2e30d-dd82-4681-a18a-4e7c96e23d9b" />
</p>
