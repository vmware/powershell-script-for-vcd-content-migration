[![PowerShell](https://img.shields.io/badge/PowerShell-7.4%2B-blue.svg)](https://github.com/PowerShell/PowerShell)
[![License](https://img.shields.io/badge/License-Broadcom-green.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-1.0.0.0-orange.svg)](CHANGELOG.md)
[![GitHub Clones](https://img.shields.io/badge/dynamic/json?color=success&label=Clone&query=count&url=https://gist.githubusercontent.com/nathanthaler/dbee1477f49946216c3c99c39b5ece46/raw/clone.json&logo=github)](https://gist.githubusercontent.com/nathanthaler/dbee1477f49946216c3c99c39b5ece46/raw/clone.json)
![Downloads](https://img.shields.io/github/downloads/vmware/powershell-script-for-vcd-content-migration/total?label=Release%20Downloads)

# VcdToVcfaMigrator.ps1

Downloads media files from VMware Cloud Director (VCD) catalogs, uploads media files to vCenter Content Libraries, and synchronizes VCF Automation catalog content sources.

## Requirements

- PowerShell 7.2+ (PowerShell Core)
- VCF PowerCLI 9 or later (`Install-Module -Name VCF.PowerCLI -Scope CurrentUser`)
- For Download: active VCD connection credentials and a populated catalog
- For Upload: active vCenter connection and Content Library permissions
- For Sync: active VCF Automation provider credentials

## Installation

```powershell
Install-Module -Name VCF.PowerCLI -Scope CurrentUser
```

Place `VcdToVcfaMigrator.ps1` and `infrastructure.json` in the same working directory.

## Configuration

The script reads connection and catalog details from a JSON file (default: `infrastructure.json`).

### Download workflow

```json
{
    "common": {
        "localContentPath": "C:\\ISO"
    },
    "vcd": {
        "vcdFqdn": "vcd01a.example.com",
        "vcdUser": "administrator",
        "defaultOrg": "System",
        "orgName": "org1-for-media",
        "catalog": { "name": "catalog-1" }
    }
}
```

### Upload workflow

```json
{
    "common": {
        "localContentPath": "C:\\ISO"
    },
    "vcfa": {
        "vcfaFqdn": "auto01.example.com",
        "vcfaUser": "administrator@vsphere.local",
        "vCenterFqdn": "vcenter02.example.com",
        "vCenterUser": "administrator@vsphere.local",
        "contentLibrary": { "name": "provider-cl" }
    }
}
```

### Sync workflow

```json
{
    "vcfa": {
        "vcfaFqdn": "auto01.example.com",
        "vcfaUser": "administrator@vsphere.local"
    }
}
```

## Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `-Download` | Switch | — | Explicitly select the Download workflow. |
| `-Upload` | Switch | — | Explicitly select the Upload workflow. |
| `-Sync` | Switch | — | Explicitly select the Sync workflow. |
| `-LogLevel` | String | `INFO` | Minimum log level for console output. Values: `DEBUG`, `INFO`, `ADVISORY`, `WARNING`, `EXCEPTION`, `ERROR`. All levels are always written to the log file. |
| `-InsecureTls` | Bool | *(derived)* | Override TLS certificate validation. When omitted, the script reads `Get-PowerCLIConfiguration` (`InvalidCertificateAction`): `Ignore`/`Warn`/`Prompt` → `$true`; `Fail` → `$false`; Unset/unavailable → `$true` (lab-safe fallback). Pass `-InsecureTls $false` to enforce strict validation. |
| `-JsonFilePath` | String | `infrastructure.json` | Path to the JSON configuration file. |
| `-Version` | Switch | — | Display the script version and exit. |

If none of `-Download`, `-Upload`, or `-Sync` is specified, the script presents an interactive workflow selection menu.

## Usage

```powershell
# Interactive workflow selection
.\VcdToVcfaMigrator.ps1

# Explicit download workflow
.\VcdToVcfaMigrator.ps1 -Download

# Explicit upload workflow
.\VcdToVcfaMigrator.ps1 -Upload

# Explicit sync workflow
.\VcdToVcfaMigrator.ps1 -Sync

# Debug logging with a custom config file
.\VcdToVcfaMigrator.ps1 -Download -LogLevel DEBUG -JsonFilePath "C:\Config\infrastructure.json"

# Force strict TLS validation
.\VcdToVcfaMigrator.ps1 -Upload -InsecureTls $false

# Display version
.\VcdToVcfaMigrator.ps1 -Version
```

## Workflows

### Download

Downloads media files from a VCD catalog to local storage.

1. Authenticates to VCD using credentials from the configuration file.
2. Lists available media files in the configured catalog.
3. Prompts to download all files or select specific files by ID.
4. Performs an idempotent size check — files that already exist locally with a matching size are skipped automatically.
5. Prompts for overwrite confirmation when a local file exists but the source size differs.
6. Downloads files with a live progress bar and computes a SHA-256 checksum on completion.
7. Validates available disk space (including a safety buffer) before starting each download.

### Upload

Uploads media files from local storage to vCenter Content Libraries.

1. Authenticates to vCenter using credentials from the configuration file.
2. Establishes a PowerCLI connection to vCenter.
3. Lists available files in the configured local content path.
4. Prompts to upload all files or select specific files by ID.
5. Performs an idempotent size check — library items that already exist with a matching size are skipped automatically.
6. Uploads files with progress monitoring and automatic retry with exponential backoff.
7. Falls back from simple upload to streamed HttpClient upload for large files or PS 5.1 compatibility.

### Sync

Synchronizes VCF Automation catalog content sources.

1. Authenticates to the VCF Automation Provider API.
2. Queries all catalog content sources.
3. Presents an interactive selection menu to sync specific sources or all sources.
4. Monitors each sync task to completion and reports results.

## Interactive Selection Format

When prompted to select specific files or media items by ID, the following formats are accepted:

| Format | Example | Description |
|---|---|---|
| Individual IDs | `1,3,5` | Select files 1, 3, and 5 |
| Range | `1-5` | Select files 1 through 5 |
| Mixed | `1,3-5,7` | Select files 1, 3, 4, 5, and 7 |
| All | `A` or `ALL` | Select all files |

## Security

- Passwords are never stored in configuration files or logs.
- Passwords are entered interactively via masked prompts.
- SecureStrings are zeroed from memory immediately after use.
- TLS certificate validation behavior is derived from PowerCLI configuration (see `-InsecureTls`).
- SSL bypass generates a `[WARNING]` log entry on every run for operator visibility.

## Logging

Log files are written to the `logs/` subdirectory alongside the script. Each run creates a new timestamped file (e.g., `VCDToVCFAMigrator-2026-04-24.log`). All log levels are always written to the file; the `-LogLevel` parameter controls the minimum level displayed on the console.

Log level hierarchy (lowest to highest): `DEBUG` → `INFO` → `ADVISORY` → `WARNING` → `EXCEPTION` → `ERROR`

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Completed successfully or cancelled by user. |
| `1` | Script encountered an error and could not complete. |

---

## Example: Download — idempotent run (files already exist)

```powershell
PS C:\VCD.To.VCFA.Migrator> .\VcdToVcfaMigrator.ps1
How would you like to proceed?
  [D] Download - Download media files from vCD catalog (default)
  [U] Upload   - Upload media files to vCenter content library
  [S] Sync     - Sync VCF Automation catalog content sources

Enter your choice (press Enter for default): D

[INFO] Downloading media files from catalog 'catalog-1'...

Enter the password for user "administrator" on vCD server "vcd01a.example.com" (or press 'c' to cancel): ****************

[INFO] Retrieving available media files from catalog 'catalog-1'...

ID File Name                        Description
-- ---------                        -----------
 1 photon-5.0-dde71ec57.aarch64.iso (No description)
 2 Rocky-9.4-x86_64-dvd.iso         (No description)
 3 Rocky-9.4-x86_64-minimal.iso     (No description)

How would you like to proceed?
  [A] All    - Download all media files (default)
  [S] Select - Select specific media files by ID

Enter your choice (press Enter for default): A
[INFO] Selected all 3 media file(s) for download.
[INFO] [1/3] Processing: photon-5.0-dde71ec57.aarch64.iso
[WARNING] File 'photon-5.0-dde71ec57.aarch64.iso' already exists with matching size (3.81 GB). Skipping download (idempotent).
[INFO] SHA256 Checksum: 06F4B20D3097FCEBC3EA067E41E4FB64FFE41828BDB9FA96CEBC7A49F290C0D9
[INFO] [2/3] Processing: Rocky-9.4-x86_64-dvd.iso
[WARNING] File 'Rocky-9.4-x86_64-dvd.iso' already exists (8.98 GB), but source size (10.17 GB) differs. File may be incomplete.

File 'Rocky-9.4-x86_64-dvd.iso' already exists but source size differs. Overwrite?
  [Y] Yes - Overwrite this file
  [N] No  - Skip this file (default)
  [A] All - Overwrite this and all remaining files with different sizes

Enter your choice (press Enter for default): A
[INFO] Overwriting this and all remaining existing files with different sizes.
[INFO] Starting Download: Rocky-9.4-x86_64-dvd.iso (10.17 GB)
[ERROR] Cannot download file: Insufficient disk space. Available: 4.26 GB, Required: 11.18 GB (including safety buffer)
[INFO] [3/3] Processing: Rocky-9.4-x86_64-minimal.iso
[WARNING] File 'Rocky-9.4-x86_64-minimal.iso' already exists with matching size (1.7 GB). Skipping download (idempotent).
[INFO] ========================================
[INFO] Download Summary
[INFO] ========================================
[INFO] Files Downloaded: 0
[INFO] Files Skipped: 2
[ERROR] Errors: 1
[INFO] ========================================
```

---

## Example: Upload — idempotent run (files already in content library)

```powershell
PS C:\VCD.To.VCFA.Migrator> .\VcdToVcfaMigrator.ps1 -Upload
[INFO] Uploading media files to content library...

Enter the password for the user "administrator@w01.local" on vCenter "vcenter02.example.com" (or press 'c' to cancel): ****************
[INFO] Successfully authenticated to vCenter. Session token obtained.
[INFO] Connecting to vCenter 'vcenter02.example.com'...
[INFO] Verifying content library 'provider-cl' exists in vCenter...

ID File Name                        Size (MB)
-- ---------                        ---------
 1 photon-5.0-dde71ec57.aarch64.iso 3,902.54
 2 Rocky-9.4-x86_64-dvd.iso         10,410.69
 3 Rocky-9.4-x86_64-minimal.iso     1,744.88

How would you like to proceed?
  [A] All    - Upload all files (default)
  [S] Select - Select specific files by ID

Enter your choice (press Enter for default): A
[INFO] Selected all 3 file(s) for upload.

[INFO] [1/3] Processing file: photon-5.0-dde71ec57.aarch64.iso
[WARNING] Library item 'photon-5.0-dde71ec57.aarch64.iso' already exists with matching size (3.81 GB). Skipping upload (idempotent).

[INFO] [2/3] Processing file: Rocky-9.4-x86_64-dvd.iso
[WARNING] Library item 'Rocky-9.4-x86_64-dvd.iso' already exists with matching size (10.17 GB). Skipping upload (idempotent).

[INFO] [3/3] Processing file: Rocky-9.4-x86_64-minimal.iso
[WARNING] Library item 'Rocky-9.4-x86_64-minimal.iso' already exists with matching size (1.7 GB). Skipping upload (idempotent).

[INFO] Upload Summary:
[INFO]   Total files processed: 3
[INFO]   Successful uploads:    0
[INFO]   Failed uploads:        0
[INFO]   Skipped files:         3
[INFO] Upload process completed successfully.
```

---

## Example: Download — single file by selection

```powershell
PS C:\VCD.To.VCFA.Migrator> .\VcdToVcfaMigrator.ps1 -Download

Enter the password for user "administrator" on vCD server "vcd01a.example.com" (or press 'c' to cancel): ****************

ID File Name                                Description
-- ---------                                -----------
 1 sample-1.2.1-24977245.iso (No description)
 2 nsx-edge-9.0.1.0.24952115.iso            4.7GB, big file

How would you like to proceed?
  [A] All    - Download all media files (default)
  [S] Select - Select specific media files by ID

Enter your choice (press Enter for default): S

Enter your selection: 1
[INFO] Selected 1 media file(s) for download: sample-1.2.1-24977245.iso
[INFO] [1/1] Processing: sample-1.2.1-24977245.iso
[INFO] Starting Download: sample-1.2.1-24977245.iso (41.23 MB)
[INFO] Download completed: 43,231,232 bytes downloaded
[INFO] Status: File size matches expected size - OK
[INFO] SHA256 Checksum: 596F7CA0BC44F3732C4EB15F612CDCFAA2E50ACC2C9CCCF7FEB3D24A84E658D5
[INFO] ========================================
[INFO] Download Summary: Files Downloaded: 1, Skipped: 0, Errors: 0
[INFO] ========================================
```

---

## Example: Cancelling a prompt

```powershell
PS C:\VCD.To.VCFA.Migrator> .\VcdToVcfaMigrator.ps1

Enter the password for user "administrator" on vCD server "vcd01a.example.com" (or press 'c' to cancel): *
[INFO] Password entry cancelled by user.
[INFO] Operation cancelled by user.
```
