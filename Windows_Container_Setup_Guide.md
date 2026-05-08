# Windows Container Environment — Step-by-Step Setup Guide - Draft 0.1

Technical guide for setting up a container development and testing environment

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisite Tooling (Both Paths)](#2-prerequisite-tooling-both-paths)
3. [Path A: Developer Workstation (Windows 11)](#3-path-a-developer-workstation-windows-11)
4. [Path B: Windows Server 2025](#4-path-b-windows-server-2025)
5. [Developer Tooling (Both Paths)](#5-developer-tooling-both-paths)
6. [Common Steps (Both Paths)](#6-common-steps-both-paths)
7. [Linux Containers via WSL Ubuntu 24.04](#7-linux-containers-via-wsl-ubuntu-2404)
8. [Remote Development with VS Code and Visual Studio 2026](#8-remote-development-with-vs-code-and-visual-studio-2026)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Overview

There are **two practical paths** depending on the available hardware:

| Path | Target | Docker Engine | License |
|------|--------|---------------|---------|
| **A** | Windows 11 Pro/Enterprise workstation | Docker Desktop | Docker Desktop license required for enterprises >250 employees |
| **B** | Windows Server 2025 Datacenter (Azure VM) | Docker Engine (Moby) | Included with Windows Server |

Both paths converge at Step 6 (developer tooling) and share the same Dockerfile and container workflow.

**_Remark:_** It is assumed, that either a Windows 11 or Windows Server 2005 has been initialy set-up, either as a pysical or virtual machine.

---

## 2. Prerequisite Tooling (Both Paths)

Before starting either path, set up the baseline shell environment. Everything that follows assumes you are working inside **Windows Terminal** with the **PowerShell 7 (pwsh)** profile selected and the session **“Run as Administrator”**.

### Step 1 — Install Windows Terminal

Windows Terminal is the modern multi-tab terminal host used throughout this guide.

- **Windows 11**: Windows Terminal is preinstalled. Update it from the Microsoft Store, or via winget (Step 2).
- **Windows Server 2025**: install it via winget once winget is available (Step 2):

  ```powershell
  winget install --id Microsoft.WindowsTerminal -e --accept-source-agreements --accept-package-agreements
  ```

To check whether _Windows Terminal_ is already installed, the following statement could be used to check from a Command Line Prompt:

```cmd
winget list
```

Alternatively try to search for it from the Windows Start Menu <kbd>⊞</kbd>.

---

### Step 2 — Verify Winget and Upgrade All Packages

[Winget](https://learn.microsoft.com/windows/package-manager/winget/) is the Windows Package Manager and ships with the **App Installer** package on Windows 11 and Windows Server 2025.

```powershell
# Run in an elevated PowerShell window

# Confirm winget is available
winget --version

# Accept source agreements once
winget source update

# Upgrade every installed package that has a newer version available
winget upgrade --all --include-unknown --accept-source-agreements --accept-package-agreements
```

> If `winget` is not recognised on Windows Server 2025, install the latest **App Installer** (`Microsoft.DesktopAppInstaller`) bundle from [GitHub releases](https://github.com/microsoft/winget-cli/releases) and reopen the terminal.

---

### Step 3 — Install PowerShell 7 (pwsh) via Winget

Windows ships with Windows PowerShell 5.1. The remainder of this guide uses **PowerShell 7+ (pwsh)**.

```powershell
winget install --id Microsoft.PowerShell -e --accept-source-agreements --accept-package-agreements

# Verify
pwsh -NoLogo -Command '$PSVersionTable.PSVersion'
```

Then open **Windows Terminal**, click the tab dropdown (∨) → **Settings** → **Startup** and set the **Default profile** to **PowerShell** (the pwsh entry, not “Windows PowerShell”). Open a new tab — from this point on, every command in this guide is executed in that **PowerShell (pwsh) profile, elevated**.

---

## 3. Path A: Developer Workstation (Windows 11)

### Step 1 — Verify Prerequisites

```powershell
# Check Windows edition (must be Pro, Enterprise, or Education)
(Get-ComputerInfo).WindowsProductName

# Check OS build
[System.Environment]::OSVersion.Version
```

**Requirements:**

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Windows Edition | Pro / Enterprise / Education | Enterprise |
| Processor | 64-bit, VT-x / AMD-V enabled in BIOS | — |
| RAM | 16 GB | 32 GB |
| Free Disk | 128 GB | 512 GB |
| OS Version | Windows 11 | Latest |

### Step 2 — Enable Windows Features

Enable **all** of the features below in a single pass before rebooting. WSL itself will fail to install on a host that is missing `VirtualMachinePlatform` or `Microsoft-Windows-Subsystem-Linux`; enabling them up-front and rebooting first lets Windows finish staging the optional components (the “auto-repair” on next boot) so that `wsl --install` can succeed.

```powershell
# Run PowerShell as Administrator

# Containers (Windows containers)
Enable-WindowsOptionalFeature -Online -FeatureName Containers -All -NoRestart

# Hyper-V (needed for Hyper-V isolation mode and as a WSL2 dependency)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart

# Virtual Machine Platform (REQUIRED for WSL2 — must be enabled manually)
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All -NoRestart

# Windows Subsystem for Linux (REQUIRED for WSL — must be enabled manually)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux -All -NoRestart

# Reboot so Windows finalises ("auto-repairs") the staged optional components
Restart-Computer
```

### Step 3 — Install WSL and Ubuntu 24.04

After the reboot from Step 2 has completed, install WSL and the Ubuntu 24.04 distribution.

```powershell
# Run PowerShell as Administrator

# Make sure WSL is set to v2 by default and update the kernel
wsl --set-default-version 2
wsl --update

# List available distributions (optional — confirms Ubuntu-24.04 is offered)
wsl --list --online

# Install Ubuntu 24.04 (will prompt for a UNIX username and password on first launch)
wsl --install -d Ubuntu-24.04

# Verify
wsl --list --verbose
```

Expected output of `wsl --list --verbose`:

```
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2
```

### Step 4 — Install Docker Desktop

1. Download from [https://docs.docker.com/desktop/install/windows-install/](https://docs.docker.com/desktop/install/windows-install/)
2. Run the installer
3. **Important**: After first launch, switch to **Windows containers**:
   - Right-click Docker icon in system tray → **"Switch to Windows containers..."**
   - Or from PowerShell:
     ```powershell
     & "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchWindowsEngine
     ```

### Step 5 — Verify Docker is Running in Windows Mode

```powershell
# Should show OS/Arch: windows/amd64
docker version

# Should show "OSType: windows"
docker info | Select-String "OSType"
```

---

## 4. Path B: Windows Server 2025

### Step 1 — Provision the Azure VM

If not yet done, provision an **Azure VM** running `Windows Server 2025 Datacenter`.

**Minimum specs:**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 16 GB | 32 GB |
| Disk | 128 GB | 512 GB |
| Edition | Standard or Datacenter | Datacenter |

### Step 2 — Install Required Windows Features

Install **all** required features in a single pass before rebooting. WSL on Windows Server 2025 depends on `VirtualMachinePlatform` and the `Microsoft-Windows-Subsystem-Linux` optional component; enabling them manually first and rebooting lets Windows finish staging them (the “auto-repair” on next boot) so `wsl --install` can succeed.

```powershell
# Run PowerShell as Administrator

# Containers (Windows containers)
Install-WindowsFeature -Name Containers

# Hyper-V (Hyper-V isolation and WSL2 dependency)
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools

# Virtual Machine Platform (REQUIRED for WSL2 — must be enabled manually)
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Windows Subsystem for Linux (REQUIRED for WSL — must be enabled manually)
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux  /all /norestart

# Reboot so Windows finalises ("auto-repairs") the staged optional components
Restart-Computer
```

### Step 3 — Install WSL and Ubuntu 24.04

After the reboot from Step 2 has completed, install WSL and the Ubuntu 24.04 distribution.

```powershell
# Run PowerShell as Administrator

# Make sure WSL is set to v2 by default and update the kernel
wsl --set-default-version 2
wsl --update

# List available distributions (optional — confirms Ubuntu-24.04 is offered)
wsl --list --online

# Install Ubuntu 24.04 (will prompt for a UNIX username and password on first launch)
wsl --install -d Ubuntu-24.04
```

The distribution will be started, asking for an account and password. After having entered the data, exit from the Ubunto prompt by entering

```bash
exit
```

Verify that the just installed Ubuntu is now available in WSL.

```powershell
# Verify
wsl --list --verbose
```

Expected output of `wsl --list --verbose`:

```powershell
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2
```

### Step 4 — Install Docker Engine

**Docker CE (Community Engine) manual install**

Docker Community Edition (Docker CE) provides a standard runtime environment for containers. The environment offers a common API and command-line interface.

To get started with Docker on Windows Server, use the following command to run the 'install-docker-ce.ps1' PowerShell script. This script configures your environment to enable container-related OS features. The script also installs the Docker runtime.

```powershell
Invoke-WebRequest -UseBasicParsing "https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-DockerCE/install-docker-ce.ps1" -outfile install-docker-ce.ps1

.\install-docker-ce.ps1
```

### Step 5 — Verify Installation

```powershell
docker version
docker info
```

---

## 5. Developer Tooling (Both Paths)

With Docker Engine/Desktop and WSL in place, install the development tooling via winget. Run these from the **elevated Windows Terminal → PowerShell (pwsh) profile** established in [Section 2](#2-prerequisite-tooling-both-paths).

### Step 6 — Install VS Code, Visual Studio, and Python

- If only Command-line tooling is desired/needed - i.e. not using any GUI for build - only Steps 6a & 6c need to be done.
- If only GUI usage is desired/needed, only Steps 6b & 6c need to be done.
- If all is desired/needed, do Steps 6a to &c.

#### Step 6a - Install VS Code + required Toolsets

```powershell
# Visual Studio Code
winget install --id Microsoft.VisualStudioCode -e --accept-source-agreements --accept-package-agreements

# VS Code extensions (close and reopen the terminal first if `code` is not yet on PATH).
# Extension IDs (Microsoft-published, extension packs preferred):
#   ms-vscode.cpptools-extension-pack            -> "C/C++ Extension Pack"
#   ms-dotnettools.csdevkit                      -> ".NET C# Development Kit"
#   ms-azuretools.vscode-docker                  -> "Docker" -> "Container Tools"
#   ms-vscode-remote.vscode-remote-extensionpack -> "Remote Development"
# The github.copilot extension is not needed, as it is already a built-in extension.
# The env setting tells NODE to use the Windows Certificate Store for a trusted Root CA.
$env:NODE_OPTIONS="--use-system-ca"
$vscodeExtensions = @(
    'ms-vscode.cpptools-extension-pack',
    'ms-dotnettools.csdevkit',
    'ms-azuretools.vscode-docker',
    'ms-vscode-remote.vscode-remote-extensionpack'
)
foreach ($ext in $vscodeExtensions) { code --install-extension $ext --force }

# When working only with VS Code, the following additional Tools are required as well
# Git for Windows
winget install --id Git.Git -e --source winget --silent --accept-package-agreements --accept-source-agreements

# CMake
winget install --id Kitware.CMake -e --source winget --silent --accept-package-agreements --accept-source-agreements

# MSVC Build Toolsets 143 (VS 2022) to be able to build from Command Line
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --source winget --override "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.Windows11SDK.22621 --includeRecommended"
```

For checking the Toolset installations, the fowwing can be executed:

```powershell
# 1) Locate vswhere (works for both VS 2022 and VS 2026 installs)
$vswhere = @("${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe","$env:ProgramFiles\Microsoft Visual Studio\Installer\vswhere.exe") | Where-Object { Test-Path $_ } | Select-Object -First 1; if (-not $vswhere) { Write-Error "vswhere.exe not found — VS Installer not present"; return }

# 2) List every installed VS product with version + path
& $vswhere -products * -prerelease -format json | ConvertFrom-Json | Select-Object displayName, installationVersion, installationPath

# 3) Enumerate the MSVC toolsets inside each install
& $vswhere -products * -prerelease -property installationPath | ForEach-Object {
    $msvc = Join-Path $_ 'VC\Tools\MSVC'
    if (Test-Path $msvc) {
        Write-Host "`n== $_ ==" -ForegroundColor Cyan
        Get-ChildItem $msvc -Directory | Select-Object Name
    }
} 
```

#### Step 6b - Instal Visual Studio incl. required Workloads

```powershell
# Visual Studio 2026 Enterprise
# Workloads:
#   Microsoft.VisualStudio.Workload.NativeDesktop   -> "Desktop development with C++"
#   Microsoft.VisualStudio.Workload.ManagedDesktop  -> ".NET desktop development"
#   Microsoft.VisualStudio.Workload.NativeCrossPlat -> "Linux and embedded development with C++"
# Individual components:
#   Microsoft.VisualStudio.Component.Git            -> "Git for Windows"
#   Microsoft.VisualStudio.Component.VC.CMake.Project -> "C++ CMake tools for Windows"
winget install --id Microsoft.VisualStudio.2026.Enterprise -e `
    --override "--quiet --wait --norestart --nocache --add Microsoft.VisualStudio.Workload.NativeDesktop --add Microsoft.VisualStudio.Workload.ManagedDesktop --add Microsoft.VisualStudio.Workload.NativeCrossPlat --add Microsoft.VisualStudio.Component.Git --add Microsoft.VisualStudio.Component.VC.CMake.Project --includeRecommended" `
    --accept-source-agreements --accept-package-agreements
```

#### Step 6c - Install Python including required modules

```powershell
# Python 3.14 (latest stable as of 2026-04 — Python 3.4 is end-of-life and not
# available via winget; use 3.14 unless a specific legacy version is required).
winget install --id Python.Python.3.14 -e --accept-source-agreements --accept-package-agreements

# Updating PIP
python -m pip install --upgrade pip
# Additional Python Packages regarding C++ Development
pip install conan coloredlogs cmake_format
```

### Step 7 — Verify the Installations

Close and reopen the terminal so the updated `PATH` is picked up, then:

```powershell
git --version
cmake --version
code --version
python --version
pwsh -NoLogo -Command '$PSVersionTable.PSVersion'
```

For Visual Studio, launch **Visual Studio Installer** and confirm the **Desktop development with C++** workload is present (add it from the installer if not).

---

## 6. Common Steps (Both Paths)

### Step 8 — Pull Windows Base Images

```powershell
# Windows Server Core 2025 (LTSC)
docker pull mcr.microsoft.com/windows/servercore:ltsc2025

# Nano Server (minimal, but lacks many Win32 APIs — NOT suitable for most enterprise workloads)
docker pull mcr.microsoft.com/windows/nanoserver:ltsc2025

# Check pulled images
docker image ls
```

> ⚠️ **Critical rule**: The container OS version must **match** (or be ≤) the host OS version when using **process isolation**. A version mismatch requires **Hyper-V isolation**.

### Step 9 — Run Your First Windows Container

```powershell
# Interactive PowerShell session inside a Windows container
docker run -it mcr.microsoft.com/windows/servercore:ltsc2025 powershell

# Inside the container, verify:
[System.Environment]::OSVersion
Get-Process
hostname
exit
```

### Step 10 — Understand Isolation Modes

```powershell
# Process isolation (default on Server, faster, shares host kernel)
docker run --isolation=process mcr.microsoft.com/windows/servercore:ltsc2025 cmd /c ver

# Hyper-V isolation (each container gets its own kernel, required for version mismatch)
docker run --isolation=hyperv mcr.microsoft.com/windows/servercore:ltsc2025 cmd /c ver
```

---

## 7. Linux Containers via WSL Ubuntu 24.04

The Docker engine on Windows can only run **Windows** containers. To also run **Linux** containers from the same `docker` CLI on the Windows host, install Docker Engine inside the Ubuntu 24.04 WSL distribution, expose its socket, and register it as a named **Docker context** called `linux` on the host. Switching between the `default` (Windows) and `linux` (WSL) contexts then becomes a one-liner — and the same contexts are picked up automatically by VS Code and Visual Studio 2026.

> All steps below run on the **Windows Server 2025** host that you set up in [Section 4](#4-path-b-windows-server-2025). The same procedure works on a Windows 11 workstation when Docker Desktop's WSL2 backend is **disabled** (so it doesn't fight for the WSL distro).

### Step 11 — Install Docker Engine inside Ubuntu 24.04 (WSL)

Open Windows Terminal, switch to the **Ubuntu-24.04** profile (or run `wsl -d Ubuntu-24.04`) and install Docker Engine using the official Docker apt repository:

```bash
# Inside the Ubuntu-24.04 WSL shell

# Remove any distro-shipped docker bits
sudo apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

# Prerequisites + Docker apt repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine, CLI, containerd, Buildx and Compose plugins
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Run docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```

WSL does not run `systemd` units automatically until you opt in. Enable systemd so the Docker service starts with the distro:

```bash
# /etc/wsl.conf inside Ubuntu
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true
EOF
```

From an elevated **PowerShell (pwsh)** window on the Windows host, restart the distro so systemd takes effect:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Back in Ubuntu, enable and verify the service:

```bash
sudo systemctl enable --now docker
systemctl is-active docker      # -> active
docker run --rm hello-world     # smoke test, pulls a Linux image
```

### Step 12 — Expose the Docker Socket from WSL to the Windows Host

The simplest, most stable bridge is to expose the Docker daemon over a **TCP socket bound to the WSL loopback** and reach it from Windows via `localhost`. WSL2 forwards `localhost` between Windows and the distro automatically.

```bash
# Inside Ubuntu-24.04

# Add a TCP listener (in addition to the default unix socket) via a systemd drop-in
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/override.conf > /dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://127.0.0.1:2375
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker

# Sanity check from inside WSL
curl -s http://127.0.0.1:2375/_ping && echo
# -> OK
```

> ⚠️ Port `2375` is **unencrypted**. Binding to `127.0.0.1` keeps it local to the host (and the WSL guest). Do **not** bind to `0.0.0.0` on a multi-user server. For production, switch to a TLS-secured `tcp://...:2376` listener and distribute client certificates.

### Step 13 — Create the `linux` Docker Context on the Windows Host

Back in **Windows Terminal → PowerShell (pwsh)** on the host:

```powershell
# Create a named context that points at the WSL daemon
docker context create linux `
    --description "Docker Engine inside WSL Ubuntu 24.04" `
    --docker "host=tcp://127.0.0.1:2375"

# List contexts (the asterisk shows which one is active)
docker context ls
# NAME      DESCRIPTION                               DOCKER ENDPOINT
# default * Current DOCKER_HOST based configuration   npipe:////./pipe/docker_engine
# linux     Docker Engine inside WSL Ubuntu 24.04     tcp://127.0.0.1:2375
```

### Step 14 — Switch Between Windows and Linux Contexts

```powershell
# Use the WSL/Linux engine
docker context use linux
docker version           # Server OS/Arch should now read linux/amd64
docker run -it --rm alpine uname -a

# Back to the native Windows engine
docker context use default
docker version           # Server OS/Arch should read windows/amd64
docker run -it --rm mcr.microsoft.com/windows/servercore:ltsc2025 cmd /c ver
```

A one-shot override (without changing the active context) is also useful in scripts:

```powershell
docker --context linux ps
docker --context default ps
```

### Step 15 — Verify End-to-End

```powershell
# Build a tiny multi-arch sanity check
docker --context linux run -it --rm python:3.14-slim python -c "print('hello from linux')"
docker --context default run --rm mcr.microsoft.com/windows/servercore:ltsc2025 cmd /c "echo hello from windows"
```

Both commands should succeed from the same elevated pwsh session on the Windows host.

---

## 8. Remote Development with VS Code and Visual Studio 2026

With the `default` (Windows) and `linux` (WSL) Docker contexts in place, both IDEs can target either engine without leaving the Windows host.

### Step 16 — VS Code: Remote — WSL

The **Remote Development** extension pack from [Section 5](#5-developer-tooling-both-paths) provides three relevant components: *Remote — WSL*, *Dev Containers*, and *Remote — SSH*.

1. From Windows, launch VS Code: `code .`
2. Press `F1` → **WSL: Connect to WSL using Distro…** → select **Ubuntu-24.04**.
3. VS Code installs its server inside the distro automatically. The status bar in the lower-left turns green and reads **WSL: Ubuntu-24.04**.
4. Open the Linux project folder (e.g. `~/project`). All build/debug commands now execute inside Ubuntu against the Linux toolchain.

### Step 17 — VS Code: Dev Containers Against the `linux` Context

The **Docker** extension reads the same context list as the CLI.

1. Open the **Docker** (or **Containers**) view in the activity bar.
2. Click the context selector at the top of the view and choose **linux**. Containers, images, and volumes from the WSL daemon now appear. Or with the **Containers** view, expand "DOCKER CONTEXTS" and select **linux**.
3. To develop *inside* a Linux container, add a `.devcontainer/devcontainer.json` to the repository, then run `F1` → **Dev Containers: Reopen in Container**. VS Code will:
   - build the image on the active context (`linux`),
   - start the container,
   - install the VS Code server inside it,
   - and reattach the editor.

Switch back to Windows containers by selecting the **default** context in the same picker before reopening.

### Step 18 — Visual Studio 2026: WSL Toolchain for C++

The **Linux and embedded development with C++** workload installed in [Step 6](#step-6--install-vs-code-visual-studio-and-python) ships a built-in WSL connection.

1. Open Visual Studio 2026 → **Tools → Options → Cross Platform → Connection Manager**.
2. The **WSL** node should already list **Ubuntu-24.04** (auto-discovered). If not, click **Add** → **WSL** and select the distro.
3. Inside Ubuntu, install the build prerequisites once:

   ```bash
   sudo apt-get install -y build-essential gdb gdbserver cmake ninja-build rsync zip
   ```

4. Create or open a **CMake Project** → in the configuration dropdown, pick **WSL: Ubuntu-24.04 — GCC** (or *Clang*). Build, run, and debug all happen inside the WSL distro using the toolchain you just installed.

### Step 19 — Visual Studio 2026: Container Tools and the `linux` Context

Visual Studio 2026 reuses the active Docker context from the host CLI.

1. Verify the desired context on the host: `docker context use linux` (or `default`).
2. In Visual Studio, open or create a project, then **Add → Docker Support…** (or **Container Orchestrator Support…** for compose). The generated `Dockerfile` / `docker-compose.yml` builds against the active context, so a `linux` context produces Linux images and a `default` context produces Windows images.
3. To debug an already-running Linux container, use **Debug → Attach to Process…** → **Connection type: Docker (Linux Container)** → pick the container from the dropdown.

### Step 20 - Visual Studio Code: Using Toolchain for C++

The Build Tools require the Developer environment vars (INCLUDE, LIB, PATH for cl.exe, link.exe, msbuild.exe). Easiest option is to launch VS Code from the corresponding Developer Prompt in Terminal.

Alternatively add a VS Code Terminal Profile in **settings.json**:

```json
"VS 2022 Dev": {
      "path": "cmd.exe",
      "args": ["/k", "C:\\Program Files (x86)\\Microsoft Visual Studio\\2020\\BuildTools\\Common7\\Tools\\VsDevCmd.bat"]
    },
"VS 2026 Dev": {
      "path": "cmd.exe",
      "args": ["/k", "C:\\Program Files (x86)\\Microsoft Visual Studio\\18\\BuildTools\\Common7\\Tools\\VsDevCmd.bat"]
    }    
```
  
### Step 21 — Daily Workflow Cheat Sheet

| Task | Command / Action |
|------|------------------|
| Switch to Linux engine | `docker context use linux` |
| Switch to Windows engine | `docker context use default` |
| One-off Linux command | `docker --context linux <args>` |
| Open repo inside WSL in VS Code | `F1` → *WSL: Connect to WSL using Distro…* |
| Open repo inside a Linux container | `F1` → *Dev Containers: Reopen in Container* |
| Build C++ on WSL from VS 2026 | CMake config dropdown → *WSL: Ubuntu-24.04* |
| Attach VS 2026 debugger to Linux container | *Debug → Attach to Process…* → *Docker (Linux Container)* |

---

## 9. Troubleshooting

### Useful Diagnostic Commands

```powershell
# Check Docker service status
Get-Service docker

# View Docker daemon logs (Windows Server)
Get-EventLog -LogName Application -Source Docker -Newest 20

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View container logs
docker logs <container-id>

# Inspect container details
docker inspect <container-id>

# Check disk usage
docker system df

# Clean up unused images and containers
docker system prune -f
```

---
