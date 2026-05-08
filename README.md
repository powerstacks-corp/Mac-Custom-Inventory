# macOS Custom Inventory for Azure Log Analytics

This shell script collects detailed hardware and software inventory from macOS devices managed by Microsoft Intune and sends the data to an Azure Log Analytics workspace. It was designed for use by **PowerStacks BI for Intune** customers to extend inventory visibility beyond what Intune natively provides on macOS.

## Overview

The script uses native macOS tools (`sysctl`, `system_profiler`, `diskutil`, `ioreg`, `security`, `scutil`) to gather hardware specifications, disk health, battery health, and the installed application list. The collected payload is JSON-encoded, gzip-compressed, base64-encoded, and chunked to fit Log Analytics column-size limits before upload via either the modern Log Ingestion API or the legacy Data Collector API.

For implementation guidance and integration with the BI for Intune reporting solution, refer to the documentation below.

🔗 [macOS Inventory Collection Script – PowerStacks BI for Intune Documentation](https://docs.powerstacks.com/bi-for-intune/installation/custom-inventory/macos-inventory-collection-script/)

## Features

- Device hardware inventory (CPU, memory, disks, battery, model)
- macOS application inventory via `system_profiler SPApplicationsDataType`
- Disk SMART status and temperature where reported
- Battery design vs. full-charged capacity for battery-health reporting
- GZip-compressed and Base64-encoded payloads sized for Azure ingestion
- Compatible with Intune **Shell scripts** deployment (root context)
- Supports both the **Log Ingestion API** (recommended) and the legacy **Data Collector API**

## Parameters

The script is configured by editing variables at the top of the file. There are no command-line parameters — Intune Shell scripts run the script as-is, so all settings live in-script.

### Log Ingestion API (recommended)

| Variable | Value |
|----------|-------|
| `LogAPIMode` | `LogIngestionAPI` |
| `TenantId` | Your Microsoft Entra Tenant ID |
| `ClientId` | The Application (Client) ID of the app registration created for inventory |
| `ClientSecret` | The client secret value for that app registration |
| `DceURI` | The Log Ingestion URL of your Data Collection Endpoint |
| `DcrImmutableId` | The Immutable ID of your Data Collection Rule |

### Data Collector API (legacy)

| Variable | Value |
|----------|-------|
| `LogAPIMode` | `DataCollectorAPI` |
| `CustomerId` | Log Analytics Workspace ID |
| `SharedKey` | Log Analytics Workspace Primary Key |

### Collection toggles

| Variable | Description |
|----------|-------------|
| `CollectDeviceInventory` | Set to `true` to collect device hardware inventory. Default: `true`. |
| `CollectAppInventory` | Set to `true` to collect installed-application inventory. Default: `true`. |
| `InventoryDateFormat` | `date` format string for the final status timestamp. Default: `"%m-%d %H:%M"`. |

## Usage

1. Open `Mac_Custom_Inventory.sh` in a text editor.
2. Set `LogAPIMode` to `LogIngestionAPI` (or `DataCollectorAPI` if you haven't migrated).
3. Fill in the API credentials for your chosen mode.
4. Save the script.
5. Upload it to Intune as a **macOS shell script** (Devices → macOS → Shell scripts) and assign it to your target group on a daily schedule.

The script runs as root in the Intune Shell scripts context. It does not require user interaction.

## Output

Data is posted to two custom tables in Log Analytics:

- `PowerStacksDeviceInventory_CL` — device hardware, OS, disk, and battery details
- `PowerStacksAppInventory_CL` — installed applications

For each device, the script collects:

### Device inventory (`PowerStacksDeviceInventory_CL`)

| Field | Source | Notes |
|-------|--------|-------|
| `ComputerName` | `scutil --get ComputerName` | |
| `ManagedDeviceID` | Intune MDM device CA certificate (via `security find-certificate`) | Matches Intune's reported device ID |
| `Memory` | `sysctl hw.memsize` | Bytes |
| `CPUManufacturer` | `sysctl machdep.cpu.vendor` (or `Apple` on Apple Silicon) | |
| `CPUName` | `sysctl machdep.cpu.brand_string` | |
| `CPUMaxClockSpeed` | `sysctl hw.cpufrequency_max` ÷ 1000 | MHz; Intel Macs only |
| `CPUPhysical` | `sysctl hw.packages` | Physical CPU package count |
| `CPUCores` | `sysctl hw.physicalcpu` | |
| `CPULogical` | `sysctl hw.logicalcpu` | |
| `LastBootTime` | `sysctl kern.boottime` | ISO 8601 UTC |
| `BatteryHealthPercent` | `(MaxCapacity ÷ DesignCapacity) × 100` | Calculated; differs by Intel vs. Apple Silicon |
| `BatteryFullChargedCapacity` | `ioreg AppleSmartBattery` (`AppleRawMaxCapacity` on Apple Silicon, `MaxCapacity` on Intel) | |
| `BatteryDesignedCapacity` | `ioreg AppleSmartBattery` (`DesignCapacity`) | |
| `DeviceManufacturer` | Hardcoded `Apple Inc.` | |
| `DeviceModel` | `system_profiler SPHardwareDataType` (`Model Identifier`) | |
| `PhysicalDisks` | Array of objects (one per detected disk) | See below |

**`PhysicalDisks[]` per-disk fields:**

| Field | Source |
|-------|--------|
| `BusType` | `diskutil info` (`Protocol`) |
| `HealthStatus` | `diskutil info` (`SMART Status`); `Verified` is normalized to `Healthy` |
| `Manufacturer` | Hardcoded `Apple` |
| `Model` | `diskutil info` (`Device / Media Name`) |
| `Size` | `diskutil info` (`Disk Size`, in bytes) |
| `Type` | `SSD` or `HDD`, derived from `diskutil` (`Solid State`) |
| `Temperature` | `ioreg` temperature sensor reading where available |

### App inventory (`PowerStacksAppInventory_CL`)

| Field | Source |
|-------|--------|
| `AppName` | `system_profiler SPApplicationsDataType` (`_name`) |
| `AppVersion` | `system_profiler SPApplicationsDataType` (`version`) |
| `AppInstallDate` | `system_profiler SPApplicationsDataType` (`lastModified`) |
| `AppInstallPath` | `system_profiler SPApplicationsDataType` (`path`) |

The envelope of each upload also includes `ComputerName` and `ManagedDeviceID` so reports can join inventory rows back to the Intune device record.

Payloads are compressed, encoded, and split into safe chunks to meet Azure ingestion limits (1 MB per request for the Log Ingestion API; 32 MB per request for the legacy Data Collector API).

## Requirements

- macOS device managed by Microsoft Intune
- Bash, `curl`, `openssl`, `jq`, `bc`, `xxd`, and `iconv` (all present by default on macOS)
- Outbound network access to `login.microsoftonline.com` and either your Data Collection Endpoint (`*.ingest.monitor.azure.com`) or the legacy Log Analytics ingestion endpoint
- An Azure Log Analytics workspace and (for Log Ingestion API mode) a Data Collection Endpoint and a Data Collection Rule
- Run via Intune **Shell scripts** (executes as `root`)

## License

MIT License

This script is provided as-is without warranty. Test thoroughly before deploying in production.

---

This script is maintained by the PowerStacks team and intended for integration with [BI for Intune](https://powerstacks.com/bi-for-intune/), a reporting solution built for Microsoft Intune environments.
