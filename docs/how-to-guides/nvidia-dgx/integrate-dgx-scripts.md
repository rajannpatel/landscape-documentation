---
myst:
  html_meta:
    description: "Learn how to register, configure, and execute NVIDIA DGX enterprise management scripts in Canonical Landscape. Detailed guide on JWT/HMAC API authentication, batch script uploads, and output retrieval."
---

(how-to-integrate-dgx-scripts)=
# How to integrate and execute DGX scripts in Landscape

NVIDIA DGX systems run as managed endpoint appliances. Secure systems management is achieved using the resident Landscape Client configured for remote script execution. This native Landscape feature provides a complete, centralized audit trail of all executed tasks, closing the security gaps of raw SSH-based management. You can register, authorize, and orchestrate the DGX enterprise manageability script suite (containing 11 production tools and 8 reference scripts) inside Landscape.

This guide walks you through enabling the Landscape script plugin, setting up API authentication, uploading scripts to the Landscape library, executing jobs across a fleet of devices, and retrieving execution results and large diagnostic packages without requiring direct, inbound SSH connections.

---

## Configure the Landscape Client on DGX devices

By default, the Landscape Client does not run arbitrary remote script payloads. To manage your DGX appliances, you must explicitly enable the `ScriptExecution` plugin on each DGX client node and authorize the appropriate system users.

On each managed DGX system:

1. Configure the Landscape Client to load the script execution manager and allow execution under the `root` user context:

   ```bash
   sudo landscape-config \
     --include-manager-plugins=ScriptExecution \
     --script-users=root,landscape,nobody
   ```

2. Restart the Landscape Client to apply the changes:

   ```bash
   sudo systemctl restart landscape-client
   ```

3. Ensure that the DGX system is tagged with a consistent identifier (such as `dgx-spark`) in the Landscape Server web portal. This is useful for fleet-wide querying in mixed estates, where DGX machines are not the only Ubuntu instances enrolled in Landscape.

---

## Prepare and upload scripts to Landscape

In Landscape, **Scripts v2** is the modern scripting framework that supports script versioning, attachments, and scheduling via script profiles. To add a script to the Landscape Script Library, use the new Landscape web portal.

### Add a script via the Web Portal (Scripts v2)

1. In the Landscape web portal, select **Scripts** from the left-side navigation menu.
2. Click **Add script**.
3. Complete the form:
   - **Title**: Enter a descriptive name, such as `DGX - Device Identity` or `DGX - Hardware Configuration`.
   - **Access group**: Choose the appropriate access group (such as `global` or `dgx-group`).
   - **Code**: Paste or load the script source directly into the editor (for example, the contents of `src/device_identity.py` or `src/hardware_config.py`).
   
   ```{note}
   The script code must begin with a valid interpreter directive (shebang) such as `#!/usr/bin/env python3` or `#!/bin/bash`. Landscape uses this to execute the script in the correct context on the client system.
   ```

4. **Add attachments** (Optional):
   For scripts that rely on external configuration files, such as `hardware_config.py` referencing `default.json`, you can attach these auxiliary files directly:
   - Under the **Attachments** section, upload or attach files (e.g., `default.json`).
   - During execution, Landscape downloads these attachments to a temporary directory on the client node and exposes its path through the `LANDSCAPE_ATTACHMENTS` environment variable.
   - Reference the attachments within your script using this variable:
     - **In Bash**: `"$LANDSCAPE_ATTACHMENTS/default.json"`
     - **In Python**:
       ```python
       import os
       attachments_dir = os.environ.get("LANDSCAPE_ATTACHMENTS")
       config_path = os.path.join(attachments_dir, "default.json")
       ```
5. Click **Add script** to save.

---

## Execute scripts at scale

You can execute scripts on target devices using either the Landscape web portal or the REST API v2. In Scripts v2, script executions are managed through **Script Profiles**, allowing for immediate (one-time), event-driven, or recurring execution.

### Target selection query syntax
Landscape allows grouping and filtering target nodes based on tags or properties:
* `tag:dgx-spark` — Targets all nodes with the `dgx-spark` tag.
* `hostname:dgx-prod-*` — Targets machines with matching hostnames.
* `tag:datacenter-01 AND tag:production` — Selects production machines in a specific facility.

### Execute immediately via the Web Portal
To execute a script immediately on target systems using the web portal:
1. Go to **Instances** from the sidebar.
2. Select the target DGX instance(s) using checkboxes or search.
3. Click **Operations** > **Run script**.
4. In the side panel, select your registered DGX script (e.g., `DGX - Device Identity`), configure execution parameters (such as the system user as `root` and time limit), and click **Run**.

### Execute immediately via the REST API v2
To programmatically trigger an execution on-demand, fetch the script ID and create a one-time **Script Profile** scheduled for the current UTC time.

1. **Retrieve the script ID** using `GET /scripts`:

   ```bash
   # Retrieve the script ID for "DGX - Device Identity" using REST API v2
   SCRIPT_ID=$(curl -s -X GET "https://landscape.canonical.com/api/v2/scripts" \
     -H "Authorization: Bearer $JWT" \
     | jq '.results[] | select(.title=="DGX - Device Identity") | .id')
   ```

2. **Trigger the execution** by creating a script profile with a `one_time` trigger:

   ```bash
   # Create a one-time script profile to run immediately
   curl -X POST "https://landscape.canonical.com/api/v2/script-profiles" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $JWT" \
     -d "{
       \"title\": \"Run Device Identity on DGX Spark\",
       \"username\": \"root\",
       \"time_limit\": 300,
       \"script_id\": $SCRIPT_ID,
       \"tags\": [\"dgx-spark\"],
       \"all_computers\": false,
       \"trigger\": {
         \"trigger_type\": \"one_time\",
         \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
       }
     }"
   ```

The API returns details of the newly created script profile, and Landscape schedules immediate execution on all nodes tagged with `dgx-spark`.

---

## Retrieve and parse results

Script results are returned to the Landscape database, while full execution logs and large evidence directories are retained locally on the DGX node to avoid saturating network bandwidth.

### 1. View summary status (stdout API)
For routine monitoring and inventory reports, check the status JSON output returned by the script directly into Landscape's Activity Log:

```bash
# Query the activity details by ID using the REST API v2
curl -s -X GET "https://landscape.canonical.com/api/v2/activities/54321" \
  -H "Authorization: Bearer $JWT"
```

The output contains standard execution states and a bounded, single-line JSON string returned to `stdout` by the tool:

```json
{
  "ok": true,
  "data": {
    "product_serial": "1983925017704",
    "sys_vendor": "NVIDIA",
    "product_name": "NVIDIA_DGX_Spark",
    "product_uuid": "12345678-1234-1234-1234-123456789012"
  },
  "errors": [],
  "meta": {
    "tool": "device_identity",
    "version": "0.1.0",
    "collected_at_utc": "2026-07-21T21:38:19Z"
  }
}
```

### 2. Extract logs to stdout (Compressed payload)
If a script exceeds the stdout size limit (~1 MB), or to adhere to secure-by-default policies that avoid direct SSH/SCP access to the appliances, use the native **`DGX - Retrieve Logs`** script (`retrieve_logs_stdout.sh`):

1. Execute the `DGX - Retrieve Logs` script on the target node via Landscape.
2. The script compresses the target logs, base64-encodes the resulting tarball, and streams it to stdout surrounded by boundary markers.
3. Extract the base64 block from the Landscape activity log output and decode it locally on your administration host:

   ```bash
   # Extract block, decode and gunzip
   base64 -d < stdout_payload.b64 | gunzip > extracted_support_log.log
   ```

### 3. Retrieve deep diagnostic packages (SCP transfer)
For complex incident response, run the **`DGX - Collect Support Package`** script (`collect_package.sh`) to assemble a comprehensive, encrypted support bundle containing system profiles, GPU core dumps, and kernel telemetry.

Because these bundles can be several gigabytes, they are stored locally on the DGX appliance:

1. Run the `DGX - Collect Support Package` script via Landscape. The script returns a pointer to the generated local tarball in its `stdout` JSON payload:

   ```json
   {
     "status": "PASS",
     "bundle_path": "/var/lib/dgx_spark_management/network_enterprise_connectivity/landscape_collect_package/run_20260721/dgx_support_bundle_213819.tar.gz",
     "sha256": "8f43981881cf42...a90f30c69"
   }
   ```

2. Retrieve the bundle directly from the DGX node using secure copy:

   ```bash
   scp root@dgx-prod-01:/var/lib/dgx_spark_management/network_enterprise_connectivity/landscape_collect_package/run_20260721/dgx_support_bundle_213819.tar.gz .
   ```

> [!NOTE]
> Retrieving bundles via SCP requires direct SSH access to the endpoint. If direct SSH is disabled on your DGX networks for security compliance, use the native log-streaming script method described in **Extract logs to stdout** to retrieve log archives directly through Landscape's secure, audited outbound channel.
