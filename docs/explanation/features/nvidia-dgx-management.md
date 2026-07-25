---
myst:
  html_meta:
    description: "Learn how Landscape manages NVIDIA DGX systems as endpoint appliances. Understand the Landscape Client remote script execution model, in-band and out-of-band interfaces, and the two-tier evidence strategy."
---

(explanation-nvidia-dgx-management)=
# NVIDIA DGX systems management

NVIDIA DGX systems (such as DGX Spark) are high-performance AI infrastructure platforms. Managing a fleet of DGX devices at scale requires treating them as managed endpoint appliances operating on a standardized OS baseline (DGX OS).

Here is how the standard DGX OS releases map to their underlying Ubuntu versions:

- DGX OS 7 (required for DGX Spark, DGX B300, and DGX GB300) is based on Ubuntu 24.04 LTS
- DGX OS 6 (used for DGX H100, H200, and A100) is based on Ubuntu 22.04 LTS
- DGX OS 5 (used for earlier systems like DGX-1, DGX-2, and A100) is based on Ubuntu 20.04 LTS
- DGX OS 4 is based on Ubuntu 18.04 LTS

Landscape provides a centralized control plane for DGX fleets, enabling operators to automate provisioning, track hardware and software inventory, run remote diagnostics, and manage system updates.

## Management interfaces

NVIDIA DGX systems expose both **out-of-band** (hardware-level) and **in-band** (operating system-level) management interfaces. This split allows enterprise management tools, orchestration platforms, and administrators to automate operations and monitor telemetry effectively.

### Out-of-band interfaces
Hosted by the system's Baseboard Management Controller (BMC) and accessed via a dedicated, secure management network, these interfaces remain functional regardless of the operating system's state:

* **Redfish API:** A RESTful API utilizing standard HTTP methods to perform power cycles, query sensor telemetry (thermal, power, cooling), view system health logs, and orchestrate firmware updates programmatically.
* **IPMI:** A legacy UDP-based protocol (port 623) used by tools like `ipmitool` for basic sensor queries, initial BMC network configuration, and chassis power controls.
* **BMC Web Dashboard and Remote KVM:** A secure HTTPS web interface (port 443) providing interactive graphical monitoring, user access management, and a virtual Keyboard/Video/Mouse console for low-level BIOS or OS installation troubleshooting.

### In-band interfaces
In-band interfaces require the DGX Operating System to be running and are accessed over the primary compute or management network. Landscape operates primarily as an in-band management controller:

* **Landscape Client remote script execution:** Landscape Client runs on each DGX system as a resident agent and can be configured with the native `ScriptExecution` plugin. This allows administrators to securely trigger administrative actions and run the DGX manageability script suite.
* **Centralized audit trail:** SSH-based systems management is often avoided in enterprise environments because raw SSH access does not produce a centralized, tamper-proof audit trail of the administrative activities performed. Landscape closes this security gap by logging all remote script executions centrally in the Landscape Server Activity Log, including the user who initiated the action, the execution timestamp, the full parameters, and the exact stdout/stderr outputs and exit codes.
* **NVIDIA System Management (NVSM) and DCGM:** Local administrative command-line tools—NVIDIA Data Center GPU Manager (DCGM) and NVSM—that provide granular GPU telemetry, topology mapping, and system health checks. These are queried locally by Landscape scripts or exposed via Prometheus endpoints.
* **Evidence artifacts:** Standardized diagnostic tarballs generated locally on-demand for deep troubleshooting.

---

## Landscape Client script execution and JSON integration model

To secure fleet operations and guarantee a complete audit trail, Landscape interacts with the DGX fleet using the resident Landscape Client configured for remote script execution. This native Landscape feature allows operators to orchestrate the standard NVIDIA DGX enterprise manageability script suite locally on the device.

```{mermaid}
sequenceDiagram
    autonumber
    participant LS as Landscape Server
    participant LC as Landscape Client
    participant DGX as DGX OS (In-band)
    participant LD as Local Disk (/var/lib/dgx_spark_management)

    LS->>LC: Deploy & Execute Script Activity
    LC->>DGX: Run tool (e.g. hardware_config)
    DGX->>LD: Write full evidence logs & diagnostic files
    DGX->>LC: Return single-line JSON to stdout
    LC->>LS: Send stdout JSON + exit code (0/1/2)
    Note over LS: Parse JSON to update CMDB/SIEM<br/>Retrieve heavy artifacts via SCP if needed
```

This model is built on three pillars:

### 1. Secure outbound HTTPS control plane (no inbound SSH required)
Unlike raw SSH-based systems management which requires opening inbound network ports and exposes endpoints to credential/key-compromise risks, Landscape Client uses a secure outbound pull model. The client communicates with the Landscape Server over standard outbound HTTPS (port 443). This eliminates the requirement to expose inbound SSH ports on the DGX compute or management networks, significantly reducing the network attack surface of your AI infrastructure.

### 2. Standardized stdout JSON API
Tools on the device return strictly bounded, single-line JSON documents on `stdout` per invocation. This JSON-first output contract allows Landscape to cleanly ingest inventory, monitoring, and status data into its Activity Log and pass it to central CMDB, SIEM, or ITSM pipelines.

### 3. Tool classification
Operational scripts and commands on the DGX device are divided into two functional classes:
* **Collectors:** Read-only tools safe to execute frequently at high concurrency to gather status (such as `device_identity.py`, `hardware_config.py`, or `firmware_reporter.py`).
* **Controllers:** State-changing tools that modify system configuration, apply patches, or trigger reboots. These are gated by change windows, rollout waves, and strict pre/post-execution checks (such as `spark_updatectl.py`).

---

## Evidence minimization and artifact strategy

Because full diagnostic collections and system logs can exceed several gigabytes, transmitting them directly over standard status API payloads would saturate network bandwidth and overwhelm Landscape Server database storage. 

To prevent this, Landscape and the DGX management scripts utilize a **two-tier evidence strategy**:

1. **Bounded `stdout` JSON:** Used for routine monitoring, health signals, and inventory summaries. This is sent back to Landscape Server immediately upon execution.
2. **On-demand evidence artifacts:** Large log files, deep system reports, or diagnostic tarballs are written directly to local storage on the DGX device under `/var/lib/dgx_spark_management/` or `/var/log/dgx_spark/`. 

The bounded `stdout` JSON returned to Landscape contains pointers (such as file paths, cryptographic hashes, and file sizes) to these local artifacts. This allows orchestration systems or administrators to retrieve the full evidence packages—either directly via Landscape Client log-streaming scripts or using secure file transfer (SCP/SFTP) if configured—only when deep triage is required.

---

## Fleet lifecycle management backbone

Scale operations across the DGX fleet are organized around a standardized lifecycle backbone mapping directly to standard enterprise IT workflows:

| Lifecycle phase | Objective | Primary mechanism | Landscape role |
| :--- | :--- | :--- | :--- |
| **Procurement and receiving** | Record stable asset identifiers and establish "as-received" system snapshots. | `device_identity.py`, `os_build_identity.py` | Query identities to enroll devices and register baseline records. |
| **Initial provisioning** | Enumerate hardware, firmware, software, and driver baselines; record UEFI asset tags. | `hardware_config.py`, `firmware_reporter.py`, `driver_inventory_reporter.py`, `NVAIAwrite`/`NVAIAread` | Execute on-boot scripts to write and verify hardware and firmware baselines. |
| **Ongoing monitoring** | Collect health signals, detect configuration drift, and analyze reset reasons. | `spark_diagctl.py`, `reset_reason_reporter.py` | Run scheduled checks to catch L1 hardware faults and reboot anomalies. |
| **Maintenance windows** | Perform controlled, staged operating system and firmware updates. | `spark_updatectl.py` | Execute updates in waves, coordinating pre/post-checks and reboots. |
| **Incident response** | Triage L1 issues or generate full L2 diagnostic evidence bundles. | `spark_diagctl.py` (bundle modes) | Trigger deep diagnostics on-demand when hardware issues arise. |
| **End-of-life / Cascade** | Execute factory resets, issue retirement proof certificates, and redeploy. | Factory reset and reprovisioning scripts | Run gated, secure factory reset probes to prepare devices for decommissioning or reuse. |
