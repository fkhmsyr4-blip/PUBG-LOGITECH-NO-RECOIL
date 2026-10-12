# Windows Services Gaming Mode (GameBooster Guide)

Disabling Windows services at random can cause system instability, broken audio or networking, and anti-cheat errors. In addition, simply lowering the total number of running processes or services **does not guarantee higher average FPS**—its primary benefit on lower- or mid-range systems is reducing background disk I/O, freeing standby/active RAM, and minimizing periodic CPU wake-ups (frametime spikes / 1% low drops).

This document defines the **Windows Services Gaming Mode** specification and manual workflow for **GameBooster**, keeping service management safe, reversible, and strictly opt-in.

---

## Core Design Principles

**Windows Services Gaming Mode** is a dedicated, **opt-in** section inside GameBooster with two explicit actions (**Game Mode** and **Restore All**). It never runs automatically when the application launches because hardware behavior varies across systems (for example, 8 GB vs. 32 GB RAM, or SATA HDD vs. NVMe SSD).

| Feature | Behavior |
| --- | --- |
| 🟢 **Game Mode** | Temporarily stops selected optional, non-essential background services (`SysMain`, `WSearch`, etc.) for the current gaming session without permanently changing their Windows startup type (`StartupType`). |
| 💾 **Save Original State** | Snapshots only the target services that are actively `Running` before optimization (`services-state.json`), ensuring services that were already stopped are never started later or reconfigured. |
| 🔄 **Restore All** | Automatically restarts only the services stopped by Game Mode as soon as **GameLoop** (`AndroidEmulatorEx.exe` / `aow_exe.exe`) or **PUBG** (`TslGame.exe`) exits, with a manual **Restore All** button/command available at any time. |
| 🛡️ **System Protection** | Enforces a strict protection list so **Microsoft Defender**, **Windows Update**, **Windows Audio**, **Core Networking**, and **Logitech G Hub / GPU drivers** are never stopped. |

---

## Candidate Optional Services (Stopped in Game Mode)

Only services that are currently `Running` at the time you activate **Game Mode** are stopped and logged for restoration:

| Service Name | Display Name | Why Stop Temporarily During Gaming | Trade-off While Active |
| --- | --- | --- | --- |
| `SysMain` | SysMain (Superfetch) | Preloads apps into RAM and performs background disk reads/writes. Stopping it frees 500 MB–1.5 GB of RAM pressure on 8–16 GB systems and prevents background disk thrashing. | Non-game desktop apps may take 1–2 seconds longer to open until restored. |
| `WSearch` | Windows Search | Continuously indexes files in the background, competing with game asset streaming and shader caching. | Start menu and File Explorer searches fall back to non-indexed search until restored. |
| `DiagTrack` | Connected User Experiences and Telemetry | Periodically aggregates and uploads diagnostic telemetry, causing occasional CPU and network spikes. | Windows diagnostic reporting is paused during the session. |
| `DoSvc` | Delivery Optimization | Uses background bandwidth and disk I/O for peer-to-peer Windows/Store update transfers. | Peer-assisted background downloads pause until restored. |
| `WerSvc` | Windows Error Reporting Service | Generates and queues crash dumps in the background. | Application crash reports are not queued while Game Mode is active. |
| `Spooler` | Print Spooler | Polls printer drivers and print queues in the background. | Local and network printing is unavailable until restored. |
| `MapsBroker` | Downloaded Maps Manager | Background maintenance for offline Windows Maps data. | Offline map updates are paused. |
| `Fax` | Fax | Legacy fax background service. | None on gaming PCs. |
| `stisvc` | Windows Image Acquisition (WIA) | Polls scanners and legacy imaging devices. | Scanner acquisition is paused until restored. |

> **Hardware Tip:** If your PC has **32 GB+ RAM** and a fast **NVMe SSD**, stopping `SysMain` rarely changes FPS. Keep service toggling optional so each user can test whether their specific hardware benefits.

---

## 🛡️ Protected Services (Never Stopped)

To prevent system lockups, audio loss, disconnects, security warnings, or G Hub macro failures, **Windows Services Gaming Mode** never stops any of the following categories:

| Category | Protected Service Names (`Name`) | Why Protected |
| --- | --- | --- |
| **Security & Defender** | `WinDefend`, `WdNisSvc`, `SecurityHealthService`, `mpssvc`, `BFE`, `wscsvc` | Stopping Windows Defender or Windows Firewall leaves the system unprotected and can trigger anti-cheat integrity checks. |
| **Windows Update & Crypto** | `wuauserv`, `UsoSvc`, `WaaSMedicSvc`, `BITS`, `CryptSvc`, `TrustedInstaller` | `CryptSvc` is required for digital signature verification (used by games and anti-cheat); forcibly breaking update services can corrupt pending servicing states. |
| **Audio** | `Audiosrv`, `AudioEndpointBuilder`, `MMCSS` | Required for all game audio, voice chat, and Multimedia Class Scheduler thread prioritization. |
| **Networking** | `Dhcp`, `Dnscache`, `NlaSvc`, `netprofm`, `LanmanWorkstation`, `WlanSvc`, `nsi` | Required for IP assignment, DNS resolution, Wi-Fi/Ethernet connectivity, and low-latency packet routing. |
| **Input, Peripherals & GPU** | `LGHUBUpdaterService`, `HidServ`, `DeviceAssociationService`, `bthserv`, NVIDIA/AMD display services | Required for Logitech G Hub scripts, mouse/keyboard HID input, wireless headsets/controllers, and GPU driver features. |

---

## How Restoration Works (Auto-Watch + Manual Restore All)

Restoration is designed around two layers so services never remain accidentally stopped after you finish playing:

1. **💾 State Snapshot (`services-state.json`):**
   - When **🟢 Game Mode** is clicked, the tool checks each candidate service in the optional list.
   - Only services whose status is currently `Running` are added to `services-state.json` and then stopped.
   - Services that were already `Stopped` or `Disabled` by the user prior to launching GameBooster are ignored.
2. **🔄 Automatic Restore on GameLoop / Game Exit:**
   - A lightweight background watcher monitors the emulator and game processes:
     - **GameLoop:** `AndroidEmulatorEx.exe`, `AndroidEmulatorEn.exe`, `AndroidEmulator.exe`, `aow_exe.exe`
     - **PUBG (Steam):** `TslGame.exe`
   - Once the monitored game/emulator process closes, every service listed in `services-state.json` is automatically started again (`Start-Service`), and the state snapshot is cleared.
3. **🔄 Manual `Restore All` Button (Fallback):**
   - You can click **Restore All** (or run the restore command manually) at any time—even while the game is still open or if the emulator exited abnormally—to immediately restart all recorded services from `services-state.json`.

---

## Reference PowerShell Workflow (Manual Use)

If you want to run the exact **Windows Services Gaming Mode** workflow manually from an **Administrator PowerShell** prompt without installing third-party tweaking tools, use the commands below.

### 1. 🟢 Enable Game Mode (Save Original State & Stop Running Optional Services)

```powershell
# Run in an elevated (Administrator) PowerShell window
$StateFile = "$env:LOCALAPPDATA\GameBooster\services-state.json"
New-Item -ItemType Directory -Path (Split-Path $StateFile) -Force | Out-Null

$CandidateServices = @(
    "SysMain",
    "WSearch",
    "DiagTrack",
    "DoSvc",
    "WerSvc",
    "Spooler",
    "MapsBroker",
    "Fax",
    "stisvc"
)

$StoppedByGameMode = @()

foreach ($Name in $CandidateServices) {
    $Svc = Get-Service -Name $Name -ErrorAction SilentlyContinue
    if ($null -ne $Svc -and $Svc.Status -eq 'Running') {
        try {
            Stop-Service -Name $Name -Force -ErrorAction Stop
            $StoppedByGameMode += $Name
            Write-Host "[Game Mode] Stopped: $Name ($($Svc.DisplayName))" -ForegroundColor Green
        } catch {
            Write-Warning "[Game Mode] Could not stop $Name : $_"
        }
    }
}

$StoppedByGameMode | ConvertTo-Json | Set-Content -Path $StateFile -Encoding UTF8
Write-Host "[Game Mode] Saved $($StoppedByGameMode.Count) service(s) to $StateFile" -ForegroundColor Cyan
```

### 2. 🔄 Watch GameLoop / PUBG and Auto-Restore on Exit

```powershell
# Waits for GameLoop or PUBG to close, then restores only the services stopped above
$StateFile = "$env:LOCALAPPDATA\GameBooster\services-state.json"
$WatchProcesses = @("AndroidEmulatorEx", "AndroidEmulatorEn", "AndroidEmulator", "aow_exe", "TslGame")

Write-Host "[Watcher] Waiting for GameLoop / PUBG process to start..." -ForegroundColor Yellow
while (-not (Get-Process -Name $WatchProcesses -ErrorAction SilentlyContinue)) {
    Start-Sleep -Seconds 3
}

Write-Host "[Watcher] Game detected. Monitoring until exit..." -ForegroundColor Cyan
while (Get-Process -Name $WatchProcesses -ErrorAction SilentlyContinue) {
    Start-Sleep -Seconds 3
}

Write-Host "[Watcher] Game closed. Running Restore All..." -ForegroundColor Green
if (Test-Path $StateFile) {
    $ServicesToRestore = Get-Content $StateFile | ConvertFrom-Json
    foreach ($Name in $ServicesToRestore) {
        Start-Service -Name $Name -ErrorAction SilentlyContinue
        Write-Host "[Restore All] Restarted: $Name" -ForegroundColor Green
    }
    Remove-Item $StateFile -Force
}
```

### 3. 🔄 Manual Restore All (Immediate Fallback)

```powershell
# Run at any time to immediately restore all services recorded by Game Mode
$StateFile = "$env:LOCALAPPDATA\GameBooster\services-state.json"

if (Test-Path $StateFile) {
    $ServicesToRestore = Get-Content $StateFile | ConvertFrom-Json
    foreach ($Name in $ServicesToRestore) {
        try {
            Start-Service -Name $Name -ErrorAction Stop
            Write-Host "[Restore All] Restarted: $Name" -ForegroundColor Green
        } catch {
            Write-Warning "[Restore All] Could not restart $Name : $_"
        }
    }
    Remove-Item $StateFile -Force
    Write-Host "[Restore All] Original service state restored." -ForegroundColor Cyan
} else {
    Write-Host "[Restore All] No saved Game Mode state found." -ForegroundColor Yellow
}
```
