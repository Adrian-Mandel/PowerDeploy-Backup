
<img width="4167" height="1042" alt="powerdeploy_banner_transparent" src="https://github.com/user-attachments/assets/9771a0f8-0b5a-4c8c-b7c9-e45499cdb9c8" />



# PowerDeploy

This writeup was written by AI and is just for proof of concept purposes.

**Architecture:** A modular PowerShell-based enterprise deployment framework (~22K LOC across ~40 files) that overcomes native Intune limitations by decoupling payload hosting from orchestration.

**Core flow:** Setup.ps1 (admin wizard) → generates Intune commands with Base64-encoded params → Git-Runner_TEMPLATE.ps1 (lightweight runner deployed to endpoints) → clones repo → calls target installer script → logs everything.

**Key components:**

- 5 general installers (WinGet, MSI, EXE, URL Download, JSON-App orchestrator)
- 2 custom installers (Dell Command Update, MS Office - both multi-step with full clean)
- Multi-method uninstaller (WinGet, MSI, Registry, WMI, AppX)
- Detection scripts for Intune compliance (WinGet registry, MSI registry, AppX)
- Printer management via Azure Blob-hosted JSON config + driver ZIPs
- Azure Blob Storage integration (SAS token + AAD auth methods)
- Security Manager with ACL enforcement and path validation
- Comprehensive logging with severity levels, timestamp formatting, and filename sanitization


## Overview

PowerDeploy is specifically architected for deployment through enterprise tools like **Microsoft Intune**, **RMM agents**, and **SCCM**, with scripts designed to run in SYSTEM context on managed endpoints. The framework emphasizes reliability, consistency, and enterprise-scale deployment capabilities.


### Key Features

- **Software Management** — WinGet-based installers, MSI support, and comprehensive uninstallers with multiple removal methods
- **Printer Management** — Automated IP printer installation with driver management via Azure Blob Storage
- **Azure Integration** — Blob storage access for centralized configuration and driver distribution
- **Windows Features** — Enable/disable Windows Optional Features programmatically
- **Registry Management** — Secure registry operations with ACL management and 32/64-bit redirection handling
- **Git-Based Deployment** — Pull latest scripts from repositories and execute on target machines


## Why PowerDeploy?

Native Microsoft Intune has significant limitations that create headaches for IT administrators. PowerDeploy was built to solve these real-world problems:


### 🖨️ Printer Management Done Right

**The Problem:** Intune only offers Universal Print by default, which is unreliable and doesn't cover many enterprise printing scenarios.

**Our Solution:** PowerDeploy includes complete infrastructure to replace your print server. Easily add and manage printer availability across your organization using Azure Blob Storage for centralized driver and configuration management.


### 📦 Reliable Application Deployment

**The Problem:** Intune has notoriously inconsistent installation success rates. Even custom installers under 50MB fail regularly. Larger applications like Microsoft Office or Adobe Acrobat have extremely high failure rates when deployed through native Intune.

**Our Solution:** Only a lightweight runner script is hosted in Intune. The actual applications are pulled from the Microsoft Store via WinGet or your private Azure Blob storage — dramatically improving reliability and eliminating Intune's size limitations.


### ⚡ Rapid Development & Testing

**The Problem:** Intune apps have a painfully long development cycle. Modifying installers requires re-uploading, re-configuring, and waiting for sync cycles. Testing is slow and centralized changes are difficult.

**Our Solution:** The installer script never needs modification for troubleshooting. Make changes in real-time via GitHub or Azure Blob — your endpoints pull the latest version automatically. Test iterations in minutes, not hours.


### 📋 Comprehensive Logging

**The Problem:** Native Intune installation logging is inflexible, vague, and scattered across multiple locations. Troubleshooting failed deployments often feels like detective work.

**Our Solution:** Every script includes detailed, color-coded logging with timestamps, severity levels, and centralized log storage. Know exactly what happened, when, and why.


### 🏪 Full Microsoft Store Access

**The Problem:** The app selection available through Intune's Microsoft Store integration is limited, and many packages are outdated versions.

**Our Solution:** Full, real-time access to nearly everything in the Microsoft Store through WinGet. Install the latest versions of applications the moment they're available.


### 🔄 Streamlined Update Management

**The Problem:** Intune is extremely clunky at handling application updates. Keeping apps current requires creating new app entries, managing supersedence, and hoping detection rules work correctly.

**Our Solution:** For WinGet-based installs, pair with [WinGet Updater](https://github.com/Romanitho/Winget-AutoUpdate) to automatically keep applications current. For custom MSI deployments, simply update your JSON configuration and Azure Blob source — no need to touch the Intune app entry. *(Custom app auto-updater coming soon)*


### 🗑️ Simplified Uninstallation

**The Problem:** Finding the correct uninstaller strings for applications is often a frustrating treasure hunt through registry keys and installer logs.

**Our Solution:** The General Uninstaller script handles multiple removal methods automatically — WinGet, registry-based, WMI, and more. Just provide the app name and let the script figure out the rest.


### ✅ WinGet That Actually Works

**The Problem:** WinGet in SYSTEM context (how Intune runs scripts) has numerous quirks and often fails silently.

**Our Solution:** Our WinGet installer script handles all the edge cases — bootstrapping WinGet if missing, resetting sources, handling 32/64-bit contexts, and providing detailed error reporting when things go wrong.


### 🚀 On-Demand Execution

**The Problem:** Intune automatic app deployment can take anywhere from minutes to 48+ hours with no option to trigger execution manually for testing or urgent deployments.

**Our Solution:** Scripts are stored locally in ProgramData with manual launch options. Need an app installed right now? Run it directly. No waiting for Intune sync cycles.


## License

PENDING

## Support

For issues and feature requests, please use the [GitHub Issues](https://github.com/Santa-Cruz-COE/PowerDeploy/issues) page.

---

**Source:** [https://github.com/Santa-Cruz-COE/PowerDeploy](https://github.com/Santa-Cruz-COE/PowerDeploy)

---


**NOTE**

This writeup was written by AI and is just for proof of concept purposes.






 

#

<p align="center">
  <img src="https://github.com/user-attachments/assets/38b2e30d-dd82-4681-a18a-4e7c96e23d9b" />
</p>
