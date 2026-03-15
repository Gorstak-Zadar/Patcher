# CVE Mitigation Patcher

Script that **fetches CISA Known Exploited Vulnerabilities (KEV)** and **applies scriptable mitigations** for new CVEs without waiting for Windows Update.

## How it works

1. **Fetches** the [CISA KEV catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) (JSON).
2. **Tracks** which CVEs were already seen in `CVE-PatcherState.json` so only **new** entries trigger actions.
3. For each **new** CVE:
   - If a scriptable mitigation exists in `CVE-MitigationActions.json`, it runs it (registry, services, etc.).
   - Otherwise it **reports** the CVE and advisory link so you can apply workarounds manually.

**Important:** These are **mitigations/workarounds**, not full patches. Always install Windows Update when the official fix is available.

## Files

| File | Purpose |
|------|--------|
| `CVE-MitigationPatcher.ps1` | Main script (usable as **single file** – see below) |
| `CVE-MitigationActions.json` | Optional. CVE → actions; if missing, embedded config in script is used |
| `CVE-PatcherState.json` | Created by script – tracks “last seen” CVEs |
| `Mitigations\` | Folder for external `.ps1` scripts (optional) |
| `CVE-MitigationPatcher.log` | Log output |

**Single-file distribution:** You can ship only `CVE-MitigationPatcher.ps1`. The script embeds the same CVE/action mappings and vendor defaults; when `CVE-MitigationActions.json` is not found, it uses this embedded config. To customize, add a JSON file in the same folder (it overrides embedded) or edit the `$EmbeddedConfigJson` here-string in the script.

## Usage

**Report only (no changes, see new CVEs and links):**
```powershell
.\CVE-MitigationPatcher.ps1 -ReportOnly
```

**Only Microsoft / Windows CVEs:**
```powershell
.\CVE-MitigationPatcher.ps1 -ReportOnly -FilterVendor "Microsoft"
.\CVE-MitigationPatcher.ps1 -ReportOnly -FilterProduct "Windows"
```

**WhatIf (default) – show what *would* be applied:**
```powershell
.\CVE-MitigationPatcher.ps1
.\CVE-MitigationPatcher.ps1 -FilterVendor "Microsoft"
```

**Actually apply mitigations (run as Administrator):**
```powershell
.\CVE-MitigationPatcher.ps1 -Apply
```

## Fire-and-forget (recommended)

Run once as Administrator; the script copies itself to `C:\ProgramData\CVE-MitigationPatcher\` and registers an hourly task that runs as **SYSTEM**. No further action needed.

**Log file (read this to see what it did):**  
`C:\ProgramData\CVE-MitigationPatcher\CVE-MitigationPatcher.log`

```powershell
# Report-only every hour (Microsoft CVEs) – fire and forget
.\CVE-MitigationPatcher.ps1 -RegisterSchedule -FilterVendor "Microsoft"

# Apply mitigations every hour – fire and forget (run as Administrator)
.\CVE-MitigationPatcher.ps1 -RegisterSchedule -ScheduleApply -FilterVendor "Microsoft"

# Remove the task (optional: delete folder C:\ProgramData\CVE-MitigationPatcher to remove files)
.\CVE-MitigationPatcher.ps1 -UnregisterSchedule
```

### Manual setup (Task Scheduler)

1. Open **Task Scheduler**.
2. Create Task → Trigger: **Daily** (or every few hours); repeat every 1–2 hours if desired.
3. Action: **Start a program**  
   Program: `powershell.exe`  
   Arguments: `-NoProfile -ExecutionPolicy Bypass -File "C:\Users\Admin\Documents\CVE-MitigationPatcher.ps1" -FilterVendor "Microsoft"`
4. For automatic mitigation: add `-Apply` and set “Run with highest privileges”.

## Adding mitigations for new CVEs

1. When a new CVE appears, check the **MSRC** or CISA link in the script output for **workarounds** (e.g. “Disable X”, “Block port Y”).
2. If you can do it via registry/service/firewall, add a **built-in action** in the script (see `$MitigationActions`) and/or add a **mapping** in `CVE-MitigationActions.json`:

```json
"CVE-YYYY-NNNNN": {
  "description": "Short description",
  "run": ["Disable-SomeFeature"]
}
```

3. Or use generic primitives (registry, service, firewall, script) or templates—no script edits needed.

## Built-in actions (out of the box)

| Action | Effect | Example CVEs |
|--------|--------|--------------|
| `Disable-SMBv1` | Sets SMB1=0 in registry; disables SMB1 optional feature if present | CVE-2017-0143, 0144, 0145, 0146, 0147, 0148 |
| `Disable-SMBv3Compression` | Disables SMBv3 compression on server (registry) | CVE-2020-0796 (SMBGhost) |
| `Disable-PrintSpooler` | Stops and disables the Print Spooler service | CVE-2021-34527, CVE-2021-1675 (PrintNightmare) |
| `Block-MSDTProtocol` | Removes ms-msdt URL protocol handler from registry | CVE-2022-30190 (Follina), CVE-2022-34713 |
| `Mitigate-SigRed` | Sets TcpReceivePacketSize for DNS server (domain controllers only) | CVE-2020-1350 |
| `Enable-RDPNLA` | Enables Network Level Authentication for RDP | CVE-2019-0708 (BlueKeep) |
| `Disable-WebClient` | Stops and disables WebClient service | NTLM relay, WebDAV, PetitPotam (CVE-2022-26923) |
| `Disable-IPv6` | Disables IPv6 via registry | CVE-2024-38063 |
| `Disable-LLMNR` | Disables multicast name resolution (LLMNR poisoning mitigation) | NTLM relay attacks |
| `Disable-NBT-NS` | Disables NetBIOS over TCP/IP on all interfaces | NTLM relay attacks |
| `Disable-RemoteRegistry` | Stops and disables Remote Registry service | Lateral movement reduction |
| `Disable-NFS` | Stops and disables NFS client/server/redirector services | CVE-2022-30136, CVE-2022-26937 |

Requires **Administrator** for service and registry changes.

### Vendor/product defaults

When a CVE has **no explicit config entry**, the script can apply **default actions** based on vendor/product. Configure `defaultActionsForVendorProduct` in `CVE-MitigationActions.json`:

```json
"defaultActionsForVendorProduct": [
  {
    "vendor": "Microsoft",
    "productPattern": "Windows",
    "description": "Generic Windows hardening",
    "run": ["Disable-SMBv1", "Block-MSDTProtocol", "Disable-WebClient"]
  }
]
```

New Microsoft Windows CVEs without per-CVE mappings will then get these defense-in-depth mitigations automatically.

## Generic action types (config-only, no script edit)

| Type | Config keys | Example |
|------|-------------|---------|
| `registry` | path, name, value, valueType (DWord/QWord/String) | Set a registry value |
| `deleteRegistryKey` | path | Remove a registry key (e.g. block protocol) |
| `service` | name, startupType (Disabled/Manual) | Stop and set service startup |
| `firewall` | displayName, port, protocol, direction, action | Add firewall rule |
| `script` | path, arguments (optional) | Run external `.ps1` file |

---

## Disclaimer

**NO WARRANTY.** THERE IS NO WARRANTY FOR THE PROGRAM, TO THE EXTENT PERMITTED BY APPLICABLE LAW. EXCEPT WHEN OTHERWISE STATED IN WRITING THE COPYRIGHT HOLDERS AND/OR OTHER PARTIES PROVIDE THE PROGRAM "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE. THE ENTIRE RISK AS TO THE QUALITY AND PERFORMANCE OF THE PROGRAM IS WITH YOU. SHOULD THE PROGRAM PROVE DEFECTIVE, YOU ASSUME THE COST OF ALL NECESSARY SERVICING, REPAIR OR CORRECTION.

**Limitation of Liability.** IN NO EVENT UNLESS REQUIRED BY APPLICABLE LAW OR AGREED TO IN WRITING WILL ANY COPYRIGHT HOLDER, OR ANY OTHER PARTY WHO MODIFIES AND/OR CONVEYS THE PROGRAM AS PERMITTED ABOVE, BE LIABLE TO YOU FOR DAMAGES, INCLUDING ANY GENERAL, SPECIAL, INCIDENTAL OR CONSEQUENTIAL DAMAGES ARISING OUT OF THE USE OR INABILITY TO USE THE PROGRAM (INCLUDING BUT NOT LIMITED TO LOSS OF DATA OR DATA BEING RENDERED INACCURATE OR LOSSES SUSTAINED BY YOU OR THIRD PARTIES OR A FAILURE OF THE PROGRAM TO OPERATE WITH ANY OTHER PROGRAMS), EVEN IF SUCH HOLDER OR OTHER PARTY HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
