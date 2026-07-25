---
myst:
  html_meta:
    description: "Technical reference for NVIDIA DGX enterprise manageability scripts and Landscape reference scripts. View runtime directories, timeouts, exit codes, and execution pathways."
---

(reference-nvidia-dgx-scripts)=
# NVIDIA DGX scripts and tools reference

This page contains a comprehensive technical reference for the NVIDIA DGX (such as DGX Spark) enterprise manageability script suite. These tools are split into **production tools** (which gather system configurations, perform diagnostic checks, and manage updates) and **Landscape reference scripts** (which act as simple, single-capability conformance checks).

---

## Script suite classifications

The manageability code package is organized into two distinct implementation types:

### 1. Production tools
These are robust Python-based or compiled tools designed for operational environments. They feature extensive error handling, CLI argument parsing, and structured logging.
* **Location on DGX appliance:** `/usr/local/sbin/` (deployed copy in `bin/` of the management repository).
* **API contract:** They always return standard JSON envelopes on `stdout` and log detailed tracebacks to local files.

### 2. Landscape reference scripts
These are lightweight Bash shell scripts that wrap production tools or standard system utilities to perform automated, gated compliance probes.
* **Location in repository:** `{functional_area}/landscape_{script_name}/` (prefixed with `landscape_`).
* **Purpose:** Built as simple remote script templates for registration within the Landscape Script Library.

---

## Master script and tool directory

The following table catalogs every command, its configuration defaults, and where it stores evidence on the local DGX device:

| Deployed command / script | Category / Functional area | Recommended timeout | Default user | Primary purpose | Local output/evidence path |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **`hardware_config`** | Clear Asset Information | 300s | `root` | Enumerates CPU, GPU (via `nvidia-smi`), memory topology, SSD layout, and network interfaces. | `/var/lib/dgx_spark_management/clear_asset_information/hardware_inventory_collector/hardware_config.json` |
| **`device_identity`** | Clear Asset Information | 300s | `root` | Generates a stable system asset identifier using fallback checks (SMBIOS serial or UUID). | `/var/lib/dgx_spark_management/clear_asset_information/hardware_inventory_collector/device_identity.json` |
| **`firmware_reporter`** | Clear Asset Information | 300s | `root` | Queries system and component firmware versions (BIOS, EC, NIC, SSD, GPU) via `fwupd`. | `/var/lib/dgx_spark_management/clear_asset_information/firmware_version_reporter/firmware_versions.json` |
| **`os_build_identity`** | Clear Asset Information | 120s | `root` | Reports OS release details, kernel parameters, and custom DGX build properties. | `/var/lib/dgx_spark_management/clear_asset_information/os_build_identity_reporter/os_build_identity.json` |
| **`driver_inventory_reporter`**| Clear Asset Information | 300s | `root` | Inventories loaded kernel modules and active hardware drivers. | `/var/lib/dgx_spark_management/clear_asset_information/driver_inventory_reporter/driver_inventory.json` |
| **`software_inventory_reporter`**| Clear Asset Information | 600s | `root` | Compiles comprehensive software lists (installed `dpkg` packages, `pip` packages, active snaps, Docker containers). | `/var/lib/dgx_spark_management/clear_asset_information/software_inventory_reporter/software_inventory.json` |
| **`NVAIAread` / `NVAIAwrite`** | Clear Asset Information | 60s | `root` | Reads/writes custom enterprise metadata and asset tags from non-volatile UEFI variables. | `/sys/firmware/efi/efivars/` (SMBIOS space) |
| **`spark_updatectl`** | Controlled SW/FW Updates | 120s | `root` | Handles kernel rollbacks, grub configuration, and coordinates reboots. | `/var/lib/dgx_spark_management/controlled_sw_fw_updates/update_control_plane/status.json` |
| **`spark_diagctl`** | Remote Ops & Remediation | 600s | `root` | Monitors GPU temperatures, thermal limits, and collects crash/error diagnostics. | `/var/lib/dgx_spark_management/remote_ops_remediation/diagnostic_collector/diagnostics_full.json` |
| **`reset_reason_reporter`** | Remote Ops & Remediation | 120s | `root` | Analyzes previous-boot system state to diagnose sudden reboots or hardware crashes. | `/var/lib/dgx_spark_management/remote_ops_remediation/reset_reason_reporter/reset_reason_report.json` |
| **`signing_verification.sh`** | Attestable Conformance | 120s | `root` | Compliance probe ensuring all active APT sources are cryptographically signed. | `/var/lib/dgx_spark_management/attestable_conformance_regulatory/landscape_signing_verification/` |
| **`verified_boot_integrity.sh`**| Resilience & Recovery | 120s | `root` | Validates active UEFI Secure Boot state, kernel lockdown status, and TPM sealing evidence. | `/var/lib/dgx_spark_management/resilience_recovery_rollback/landscape_verified_boot_integrity/run_<UTC>/` |
| **`recovery_backup_levels.sh`**| Resilience & Recovery | 180s | `root` | Verifies and snapshots backup profiles across three system levels (probes, configuration files, rebuilding manifests). | `/var/lib/dgx_spark_management/resilience_recovery_rollback/landscape_recovery_backup_levels/run_<UTC>/` |
| **`factory_reset_reprovision.sh`**| Resilience & Recovery | 120s | `root` | Performs gated dry-run testing of the 4-level factory reset framework before redeployment. | `/var/lib/dgx_spark_management/resilience_recovery_rollback/landscape_factory_reset_reprovision/run_<UTC>/` |
| **`health_watchdogs.sh`** | Resilience & Recovery | 120s | `root` | Sets up and tests localized system health watchdogs and systemd transient monitoring units. | `/var/lib/dgx_spark_management/resilience_recovery_rollback/landscape_health_watchdogs/run_<UTC>/` |
| **`collect_package.sh`** | Network & Connectivity | 300s | `root` | Packages system logs and diagnostic files into a compressed, timestamped support bundle. | `/var/lib/dgx_spark_management/network_enterprise_connectivity/landscape_collect_package/run_<UTC>/dgx_support_bundle_<timestamp>.tar.gz` |
| **`retrieve_logs_stdout.sh`** | Network & Connectivity | 180s | `root` | Streams requested text logs as a compressed, base64-encoded text payload directly to Landscape's Activity Log. | (Temporary buffer only; streams directly to stdout) |
| **`encryption_at_rest.sh`** | Security Posture | 120s | `root` | Validates state of LUKS/dm-crypt encryption for all mounted non-root block storage volumes. | `/var/lib/dgx_spark_management/security_posture_vuln_response/landscape_encryption_at_rest/run_<UTC>/` |

---

## Standardized exit code conventions

When executing these scripts programmatically via the Landscape API or the command line, scripts report execution success or failure using standard UNIX exit codes.

### Production tools
All Python commands and drivers return standard codes:
* **`0` (Success):** The script completed without fatal exceptions. Outputs are stored locally, and a valid JSON structure was written to `stdout`.
* **`1` (Partial or full failure):** A fatal error occurred, or hardware details were missing. Review the list of error messages returned in the root `errors` field of the stdout JSON envelope.

### Landscape reference scripts
Bash compliance probes conform to Landscape's three-state monitoring pattern:
* **`0` (PASS):** The audited system property passed the verification checks (e.g., Secure Boot is enabled, block devices are encrypted).
* **`1` (FAIL):** The check was executed successfully, but the system property failed the audit (e.g., Secure Boot is disabled).
* **`2` (UNKNOWN):** The check could not determine compliance status because of external issues (e.g., missing dependencies like `fwupdmgr` or lacking `sudo` rights).

---

## Local directory structure mapping

Scripts run locally and write operational data, state flags, logs, and evidence tarballs to the device file system:

### 1. Raw JSON state and inventories (`/var/lib/dgx_spark_management/`)
Stores the most recent operational snapshots in standard JSON format:
```text
/var/lib/dgx_spark_management/
├── clear_asset_information/
│   ├── hardware_inventory_collector/
│   │   ├── device_identity.json
│   │   └── hardware_config.json
│   ├── firmware_version_reporter/
│   │   └── firmware_versions.json
│   ├── os_build_identity_reporter/
│   │   └── os_build_identity.json
│   ├── driver_inventory_reporter/
│   │   └── driver_inventory.json
│   └── software_inventory_reporter/
│       └── software_inventory.json
├── controlled_sw_fw_updates/
│   └── update_control_plane/
│       └── status.json
└── remote_ops_remediation/
    ├── diagnostic_collector/
    │   └── diagnostics_full.json
    └── reset_reason_reporter/
        └── reset_reason_report.json
```

### 2. Operational and execution logs (`/var/log/dgx_spark/`)
Retains rolling historical logs from each background execution of the core tools:
```text
/var/log/dgx_spark/
├── clear_asset_information/
│   └── manageability_suite_runs.log
├── controlled_sw_fw_updates/
│   └── firmware_updates.log
└── remote_ops_remediation/
    └── system_diag_runs.log
```
