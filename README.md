# AtlasOS for Windows 10 IoT Enterprise LTSC 2021

An unofficial, community-maintained adaptation of the AtlasOS v0.4.1 playbook for **Windows 10 IoT Enterprise LTSC 2021 (build 19044)**.

I made this fork for my own Windows 10 IoT Enterprise LTSC 2021 installation and decided to publish the source in case it helps someone working with the same setup. The goal is simple: retain the Windows 10-compatible AtlasOS v0.4.1 base while allowing AME Beta to recognize build `19044`.

This is not a new version of Windows, a custom ISO or an official AtlasOS release. It is a modified AME playbook that must be applied to an existing, properly licensed Windows installation.

## Important Warning

This project is **unofficial and unsupported**.

- It is not maintained, tested, approved or supported by the AtlasOS team.
- It is not maintained, tested, approved or supported by Ameliorated.
- I do not provide formal or guaranteed support for this fork.
- This project is published for educational and personal use.
- Applying an AME playbook makes deep changes to Windows and may require a clean reinstall to undo.
- You are responsible for your files, Windows installation, activation, recovery media and hardware.

You are still welcome to open an issue if you find a reproducible bug or get stuck. I or another community member may be able to suggest something but responses and fixes are not guaranteed. Please include useful details such as your Windows edition, full build number, AME version, selected options, error message and relevant log output.

**Do not use this on a production-critical computer without testing it first. Create a full system image before applying the playbook.**

## Target System

This fork is intended for:

- Windows 10 IoT Enterprise LTSC 2021
- Windows build `19044`
- 64-bit systems
- A fresh or recently installed Windows environment

Check your system before continuing:

```powershell
$v = Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
$v | Select-Object ProductName, EditionID, DisplayVersion, CurrentBuild, UBR
```

`CurrentBuild` must report `19044`. Do not assume this fork supports other LTSC releases or Windows builds simply because AME can load the playbook.

## What Was Changed

This project starts from the official AtlasOS `0.4.1` source, the last AtlasOS release designed to support Windows 10.

### Windows 10 LTSC 2021 build support

Build `19044` was added to `SupportedBuilds` in `playbook.conf`:

```xml
<SupportedBuilds>
    <!-- Windows 10 21H2 / IoT Enterprise LTSC 2021 -->
    <string>19044</string>

    <!-- Windows 10 22H2 -->
    <string>19045</string>

    <!-- Windows 11 24H2 -->
    <string>26100</string>
</SupportedBuilds>
```

This allows AME Beta to perform its build check on Windows 10 IoT Enterprise LTSC 2021. It does not make every Atlas action universally compatible with every LTSC configuration.

### Separate local-fork identity

The playbook name, author label, title and `UniqueId` were changed so the modified package no longer presents itself as an untouched official AtlasOS package.

Example metadata:

```xml
<Name>LTSCFork</Name>
<Username>Local</Username>
<Title>Atlas v0.4.1 LTSC Local Fork</Title>
```

The fork must use its own generated UUID. Do not copy the UUID from this README or reuse AtlasOS's official identity. Generate one with:

```powershell
[guid]::NewGuid()
```

### Official verification removed from local builds

Any modification changes the verified AtlasOS artifact. Local packages are therefore built with Atlas's own development option:

```powershell
-Removals Verification
```

AME should show this fork as **Unverified**. That is expected and honest. It should not show the package as verified, because the AtlasOS and Ameliorated teams have not reviewed this modified build.

### Installation-time error handling

Two small reliability changes from the newer Atlas hotfix were applied to the v0.4.1 source:

1. The Reserved Storage DISM action uses `ignoreErrors: true`.
2. The final `w32tm /resync` action uses `ignoreErrors: true`.

These flags do not hide every error and do not repair failed Windows components. They only prevent two noncritical commands from aborting the entire AME run. This is useful on LTSC, where Reserved Storage may be unavailable and immediate time synchronization can fail temporarily while networking or the Windows Time service is still starting.

Relevant files:

```text
src/playbook/Configuration/tweaks/debloat/disable-reserved-storage.yml
src/playbook/Configuration/tweaks/misc/config-time.yml
```

## What Was Not Ported

This is not a Windows 10 conversion of AtlasOS v0.5.0. It remains based on v0.4.1.

The following v0.5 systems have not been copied into this fork:

- Windows 11 component-removal packages
- Windows 11 `sxsc` definitions
- AtlasOS Toolbox
- ISO injection
- Windows 11 upgrade logic
- The complete v0.5 Atlas Folder rewrite
- v0.5 default-user and new-user orchestration

Copying those systems into v0.4.1 without a complete compatibility audit could leave Windows partially configured or unbootable.

## Before Installation

Please do all of the following before opening AME Beta:

1. Back up personal files to another drive or trusted remote location.
2. Create a full disk image and verify that your recovery media can see it.
3. Save your BitLocker recovery key and suspend BitLocker, if enabled.
4. Export important drivers.
5. Install pending Windows updates and restart until no reboot is pending.
6. Confirm the installed build is `19044`.
7. Keep the computer plugged into reliable power.
8. Preferably test the playbook in a Windows 10 IoT Enterprise LTSC 2021 virtual machine first.

Export installed third-party drivers with:

```cmd
pnputil /export-driver * C:\Drivers
```

## Building the Playbook

Install 7-Zip, open PowerShell and run the build script from the playbook source directory:

```powershell
Set-Location .\src\playbook

powershell -ExecutionPolicy Bypass `
  -File ..\dependencies\local-build.ps1 `
  -Removals Verification `
  -FileName "LTSCFork-UNVERIFIED"
```

The generated file will be created in the current playbook directory:

```text
LTSCFork-UNVERIFIED.apbx
```

Load that APBX into the latest AME Beta. The expected trust label is **Unverified**.

If AME labels it **Malicious**, do not continue. Confirm that the official `ProductCode` was removed from the built package and that your fork uses a new `UniqueId`, short custom `Name`, custom `Username` and custom `Title`.

## Installation Notes

Follow the prompts in AME Beta and read every option before selecting it. In particular:

- Keep Windows Defender enabled unless you understand and accept the security impact of disabling it.
- Keep CPU mitigations enabled unless you have a measured reason to change them.
- Keep Windows Update available so LTSC can continue receiving security updates.
- Avoid selecting removals merely because they sound performance-related.

The exact AME requirements can change between Beta releases. Use AME only from the official Ameliorated website and do not download repacked playbooks or Windows ISOs from unknown sources.

## Post-Install Process

Restart Windows after AME finishes. Then complete the following checks from **Command Prompt as Administrator**.

### 1. Check the Windows Time startup configuration

```cmd
sc qc w32time
```

Look at `START_TYPE`.

- `DEMAND_START` is acceptable.
- `AUTO_START` is acceptable.
- If it says `DISABLED`, change it to manual:

```cmd
sc config w32time start= demand
```

The space after `start=` is required by `sc.exe`.

### 2. Start the Windows Time service

```cmd
net start w32time
```

Confirm it is running:

```cmd
sc query w32time
```

The result should include:

```text
STATE : 4 RUNNING
```

If Windows says the service is already running, continue normally.

### 3. Check the current time source and status

```cmd
w32tm /query /status
w32tm /query /source
```

A valid NTP source, domain controller or Windows time source indicates that the service is configured. `Local CMOS Clock` may appear briefly before the first successful synchronization.

### 4. Request synchronization

```cmd
w32tm /resync /rediscover
```

The expected result is:

```text
The command completed successfully.
```

If it reports that no time data is available, make sure networking and DNS work, wait a minute and try once more. Do not repeatedly modify the service if the failure is only temporary.

### Check Reserved Storage

```cmd
DISM /Online /Get-ReservedStorageState
```

If it reports that Reserved Storage is disabled, no action is required.

If it is enabled and you want the Atlas default, run:

```cmd
DISM /Online /Set-ReservedStorageState /State:Disabled
```

An unsupported-command error can be harmless on LTSC because the feature may not exist in the expected state. Do not run unrelated DISM cleanup commands in response.

### Check Windows Recovery Environment

```cmd
reagentc /info
```

If it reports:

```text
Windows RE status: Enabled
```

you are done.

If it reports `Disabled`, enable it and check again:

```cmd
reagentc /enable
reagentc /info
```

If `reagentc /enable` says the recovery image cannot be found, stop. Do not create, delete or resize recovery partitions without diagnosing the existing recovery configuration first.

## Reporting a Problem

Issues and bug reports are welcome but please understand that this is a personal project and support is best-effort only.

Include the following when opening an issue:

- Full Windows product name and edition
- Output of `winver`
- Full build, including UBR
- AME Beta version
- The fork commit or release used
- Options selected during installation
- Exact error text or screenshot
- Relevant AME log output
- Whether the problem also occurs on clean Windows or only after applying the playbook

You can collect basic system details with:

```powershell
$v = Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
$v | Select-Object ProductName, EditionID, DisplayVersion, CurrentBuild, UBR
```

Please remove usernames, license information, IP addresses, recovery keys, tokens and other private data before posting logs.

## Recovery and Removal

There is no guaranteed one-click rollback for every change made by an AME playbook. If the system becomes unstable, the dependable recovery choices are:

1. Restore the full disk image created before installation.
2. Restore a VM snapshot if testing virtually.
3. Clean-install Windows from trusted installation media.

`Reset this PC` should not be treated as a guaranteed way to restore every removed or modified component.

## Credits

- [AtlasOS](https://github.com/Atlas-OS/Atlas) for the original open-source v0.4.1 playbook
- [Ameliorated](https://amelabs.net/) for AME Beta and the playbook platform
- The AtlasOS and AME contributors whose work made local, auditable Windows playbooks possible

This repository is an independent fork. References to AtlasOS and AME are for attribution and compatibility information only and do not imply endorsement.

## License

The original AtlasOS playbook is licensed under the GNU General Public License v3.0. This fork must preserve the original license and applicable copyright notices. See [`LICENSE`](LICENSE) for the complete terms.
