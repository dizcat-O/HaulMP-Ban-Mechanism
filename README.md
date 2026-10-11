# HaulMP Ban Mechanism

Technical analysis of the ban detection and HWID fingerprinting mechanism used by the HaulMP launcher, updater, and game stub.

---

## Overview

When HaulMP rejects a client, the ban is enforced at two levels: an HTTP-layer rejection during the update check, and a hardware fingerprint collected by a DLL injected into the game process. The hardware fingerprint persists across IP changes and Steam account changes because it is tied to the host machine, not the network identity.

| Component | Type | Role |
|-----------|------|------|
| `HaulMP.exe` | .NET updater | Fetches update manifest; receives ban as HTTP reason phrase; writes JSON status file |
| `HaulMP.Launcher.exe` | .NET launcher | Reads JSON status; raises `IOException`; displays ban message to user |
| `HaulMP.LaunchCore.dll` | Native x64 DLL | SHA-256 verifies ETS2 exe; spawns ETS2; injects `haulmp_stub.dll` into ETS2 via `CreateRemoteThread(LoadLibraryW)` |
| `haulmp_stub.dll` | Native x64 DLL (injected into ETS2) | Reads machine fingerprint (MachineGuid + disk serial); authenticates with HaulMP server; triggers ban-evasion rejection if fingerprint is blacklisted |

---

## Ban Detection Flow

```
HaulMP Server
     |
     |  HTTP 4xx  Reason: "This PC or Steam library is banned from HaulMP."
     |            -- OR --  "Banned: ban evasion."
     v
HaulMP.exe  -  <Get>d__26::MoveNext
     |  EnsureSuccessStatusCode() -> throws HttpRequestException
     |  Catch: status.Text = ex.Message
     |  Updater::Report(done=true, error=true, message=status.Text)
     v
%LocalAppData%\HaulMP\Client\status\<guid>.json
     { "error": true, "message": "<reason phrase>" }
     |
     v
HaulMP.Launcher.exe  -  <RunUpdate>d__7a::MoveNext
     |  UpdateReport.Error == true
     |  throw new IOException(UpdateReport.Message)
     v
Launcher UI  ->  displays ban message
```

Two distinct rejection messages observed:
- `"This PC or Steam library is banned from HaulMP."` - standard account ban
- `"Banned: ban evasion."` - triggered when the server recognises a previously banned hardware fingerprint connecting under a new IP or Steam account

---

## HTTP Request Analysis

### Updater (HaulMP.exe)

The updater creates a plain HttpClient with no authentication headers and no hardware identifier in the request:

```csharp
new HttpClient(new HttpClientHandler { AllowAutoRedirect = false })
```

Fetches `stable.json?check=<UtcNow.Ticks>` and `stable.sig?check=<UtcNow.Ticks>`. The `check` parameter is a cache-buster. No HWID, Steam token, or session credential observed in these requests.

### Launcher (HaulMP.Launcher.exe)

Sets one additional header: `User-Agent: HaulMP-Launcher/<version>`. No hardware identifier or Steam credential observed at this layer.

The ban message originates entirely on the server. The string "banned" has zero matches in any examined client binary. The server returns the rejection text as an HTTP reason phrase; the client propagates it unchanged to the UI.

---

## Hardware Fingerprinting

### Injection chain

`HaulMP.LaunchCore.dll` is a native (non-.NET) DLL loaded by the launcher via `LoadLibraryExW`. Exports two functions: `HaulMP_CheckGame` and `HaulMP_LaunchGame`.

`HaulMP_LaunchGame` executes the following sequence:

1. SHA-256 hash the ETS2 executable and compare against a hardcoded expected digest (anti-tamper). Failure throws `haulmp_launch::Failure`.
2. Spawn ETS2 with flags `-pure -nointro` via `CreateProcessW`.
3. Create named Win32 events for IPC: `Local\HaulMP-SkinStartup-<PID>-Ready` and `Local\HaulMP-SkinStartup-<PID>-Failed`.
4. Inject `haulmp_stub.dll` into the ETS2 process:
   `VirtualAllocEx` -> `WriteProcessMemory(DLL path)` -> `CreateRemoteThread(LoadLibraryW, path)`
5. Wait up to 10 s for the ready event, then exchange authentication data (30 s timeout).

`HaulMP.LaunchCore.dll` import table confirms: no registry, network, or hardware enumeration APIs. All HWID collection is delegated to `haulmp_stub.dll`.

### What haulmp_stub.dll reads

Static analysis of `haulmp_stub.dll` (6.3 MB, compiled with LLVM/libc++) confirms:

| Identifier | API | Location |
|------------|-----|----------|
| Windows Machine GUID | `RegOpenKeyExW`, `RegGetValueW` | `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` |
| Physical disk serial | `DeviceIoControl` | `\\.\PhysicalDrive0` via `IOCTL_STORAGE_QUERY_PROPERTY` |

`BCryptOpenAlgorithmProvider` is also present, consistent with combining these values into a fingerprint digest before transmission.

---

## Why IP Change and New Steam Account Fail

The server-side ban is keyed to the hardware fingerprint, not the network address or Steam identity. When a previously banned fingerprint reconnects under a new IP or Steam account, the server responds with the "ban evasion" reason phrase.

Verified empirically: Cloudflare WARP (new IP) and a brand-new Steam account both triggered the "ban evasion" response on the same physical machine.

---

## Bypass Implications

The fingerprint is derived from MachineGuid and physical disk serial:

1. MachineGuid: changeable by writing a new GUID to `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` with administrator privileges.
2. Disk serial: requires a kernel-level driver or registry override for virtual/software-enumerated devices.

Changing MachineGuid alone may be sufficient if the server weights it as the primary identifier. Testing required.

---

## Component Import Summary

| Component | Registry | Network | Hardware I/O | Process injection |
|-----------|----------|---------|-------------|------------------|
| `HaulMP.exe` | no | HttpClient (no HWID headers) | no | no |
| `HaulMP.Launcher.exe` | no | HttpClient (User-Agent only) | no | no |
| `HaulMP.LaunchCore.dll` | no | no | File SHA-256 only | VirtualAllocEx + CreateRemoteThread |
| `haulmp_stub.dll` | RegGetValueW (MachineGuid) | yes (auth exchange) | DeviceIoControl (disk serial) | no |


---

## Disclaimer

Intended for software analysis, documentation, and educational reverse engineering. Does not claim access to HaulMP's internal server implementation or private infrastructure.
