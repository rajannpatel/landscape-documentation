---
myst:
  html_meta:
    description: "Step-by-step guide to provisioning NVIDIA DGX devices with zero-touch automation. Learn to repack BaseOS images, seed configuration via USB, and manage offline or air-gapped installations."
---

(how-to-provision-dgx)=
# How to provision NVIDIA DGX systems

NVIDIA DGX systems can be provisioned using automated, unattended "zero-touch" workflows. By leveraging **cloud-init** with a local configuration seed, administrators can bootstrap bare-metal DGX devices, inject credentials, establish connectivity, and configure package mirrors without manual post-installation steps.

This guide explains how to customize installation media, configure USB-based seeding, set up offline or air-gapped package installations, and verify provisioning outcomes.

---

## Prerequisites

Before beginning, ensure you have:
* The official NVIDIA DGX BaseOS installation ISO.
* A standard administrative Linux host (desktop or server) for repackaging ISOs and formatting installation USB media.
* A USB flash drive (minimum 16 GB) to act as the installation/provisioning medium.

---

## Step 1: Customize the BaseOS image

To automate the initial installation, you can integrate your cloud-init configuration directly into the BaseOS ISO. This is achieved using the `repack_baseos.sh` utility script.

On your administrative host:

1. Extract the `repack_baseos.sh` tool from the DGX management scripts repository.
2. Prepare your custom `user-data` and `meta-data` files (see {ref}`step-2-prepare-cloud-init-seed` below).
3. Execute the repackaging script to generate a customized, bootable installer:

   ```bash
   ./repack_baseos.sh \
     --input-iso dgx-baseos-noble-arm64.iso \
     --output-iso dgx-custom-install-noble-arm64.iso \
     --user-data user-data \
     --meta-data meta-data
   ```

The script extracts the ISO, injects your custom configuration files into the initial ramdisk (initrd) under `/etc/cloud/cloud.cfg.d/` and the installer seed directory, and reconstructs the UEFI-bootable ISO.

---

(step-2-prepare-cloud-init-seed)=
## Step 2: Prepare the cloud-init seed (CIDATA)

If you prefer to keep your installation media generic, you can seed configuration to an unmodified DGX BaseOS installer on first boot using a separate volume labeled `CIDATA`. cloud-init detects this volume on boot via its **NoCloud** data source.

1. Format a USB drive (or a secondary partition on the installation drive) as **FAT32**.
2. Set the volume label of this partition strictly to `CIDATA`.
3. In the root of the `CIDATA` partition, create a file named `meta-data` specifying the system's instance ID and hostname:

   ```yaml
   instance-id: dgx-spark-prod-01
   local-hostname: dgx-spark-prod-01
   ```

4. Create a file named `user-data` to define administrative accounts, SSH keys, network configurations, and initial setup steps:

   ```yaml
   #cloud-config
   users:
     - name: nvidia
       gecos: NVIDIA Administrator
       sudo: ALL=(ALL) NOPASSWD:ALL
       shell: /bin/bash
       lock_passwd: true
       ssh_authorized_keys:
         - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... admin@company.com

   landscape:
     client:
       account_name: "standalone"
       computer_title: "my-machine"
       url: "https://landscape.example.com/message-system"
       ping_url: "http://landscape.example.com/ping"

   runcmd:
     - landscape-config --silent --include-manager-plugins=ScriptExecution --script-users=root,landscape,nobody
     - systemctl restart landscape-client
   ```

When the installer boots, cloud-init applies this configuration once during the first-boot cycle and remains inactive on subsequent normal boots.

---

## Step 3: Configure offline and air-gapped provisioning (OEMDATA)

For secure, disconnected environments, you can configure the installation media to deploy proprietary drivers, firmware updates, and local APT package mirrors completely offline.

This is done by partitioning your bootable USB drive to include a second data partition labeled `OEMDATA`.

### Partition layout
Write the customized BaseOS ISO to the USB drive, and then create a secondary ext4 or FAT32 partition on the remaining unallocated space. Format this partition and label it `OEMDATA`.

The partition must adhere to the following directory layout:
```text
OEMDATA/
├── hook.sh
├── apt-repo.url
├── lvfs-mirror.url
├── packages/
│   └── *.deb
└── firmware/
    └── *.cab
```

### 1. Staging local packages and firmware
Place required proprietary `.deb` packages in the `packages/` directory, and vendor firmware files (such as `.cab` or `.cap` capsules) in the `firmware/` directory.

### 2. Configuring local mirror links
Instead of bundling large files, you can point devices to local enterprise mirror servers on your internal network:
* **`apt-repo.url`:** A single-line text file containing the URL to your local Ubuntu Ports package mirror (for example, `http://mirror.internal/ubuntu-ports`).
* **`lvfs-mirror.url`:** A single-line text file containing the URL to your local Linux Vendor Firmware Service (LVFS) mirror (for example, `http://fwupd.internal/`).

### 3. Implementing `hook.sh` automation
Create a shell script named `hook.sh` at the root of the `OEMDATA` partition. When cloud-init boots, it mounts the `OEMDATA` partition and executes this script as `root`. 

An example `hook.sh` script to install staged packages, register local mirrors, and record an audit log:

```bash
#!/bin/bash
# oemdata/hook.sh - Automated offline package and mirror configuration
set -e

MOUNT_DIR="/mnt/oemdata"

# 1. Install local Debian packages
if [ -d "${MOUNT_DIR}/packages" ]; then
    echo "Installing staged local packages..."
    dpkg -i ${MOUNT_DIR}/packages/*.deb || apt-get install -f -y
fi

# 2. Reconfigure APT sources using local mirror
if [ -f "${MOUNT_DIR}/apt-repo.url" ]; then
    MIRROR_URL=$(cat ${MOUNT_DIR}/apt-repo.url | tr -d '\r\n')
    echo "Reconfiguring APT to use local mirror: ${MIRROR_URL}"
    
    # Write DEB822 formatted sources
    cat > /etc/apt/sources.list.d/ubuntu.sources <<EOF
Types: deb
URIs: ${MIRROR_URL}
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
fi

# 3. Reconfigure firmware (fwupd) remotes
if [ -f "${MOUNT_DIR}/lvfs-mirror.url" ]; then
    FW_URL=$(cat ${MOUNT_DIR}/lvfs-mirror.url | tr -d '\r\n')
    echo "Configuring local LVFS remote: ${FW_URL}"
    
    # Disable public remote and create local fwupd configuration
    sed -i 's/Enabled=true/Enabled=false/g' /etc/fwupd/remotes.d/lvfs.conf
    
    cat > /etc/fwupd/remotes.d/lvfs-local.conf <<EOF
[fwupd Remote]
Enabled=true
Title=Local Enterprise Mirror
Keyring=gpg
MetadataURI=${FW_URL}/metadata.xml.gz
ReportURI=${FW_URL}/report
EOF
fi

# 4. Record provisioning audit summary
mkdir -p /var/log/provisioning
cat > /var/log/provisioning/provisioning_audit.txt <<EOF
PROVISIONING STATUS: SUCCESS
TIMESTAMP: $(date -Iseconds)
HOSTNAME: $(hostname)
IMAGE_BASE: DGX-BaseOS-Noble-ARM64
MIRRORS_CONFIGURED: LOCAL
EOF
```

---

## Step 4: Verify provisioning outcomes

Upon successful build and system bootstrap, cloud-init can execute audit actions to confirm fleet consistency. 

1. Ensure your cloud-init configuration or `hook.sh` script is set to write an audit summary on completion. You can also configure cloud-init to reboot the system automatically on completion using a drop-in file (for example, `50-dgx-base-audit.cfg`):

   ```yaml
   #cloud-config
   bootcmd:
     - mkdir -p /var/log/provisioning/
   power_state:
     delay: "+1"
     mode: reboot
     message: "First-boot provisioning complete. Rebooting system."
     condition: true
   ```

2. After the automated post-installation reboot, log in to the DGX device (or query it via Landscape) and verify that the provisioning audit file exists and is populated correctly:

   ```bash
   cat /var/log/provisioning/provisioning_audit.txt
   ```

   Expected output contains the completion timestamp, hostname, and deployment details:

   ```text
   PROVISIONING STATUS: SUCCESS
   TIMESTAMP: 2026-07-21T21:38:19-04:00
   HOSTNAME: dgx-spark-prod-01
   IMAGE_BASE: DGX-BaseOS-Noble-ARM64
   MIRRORS_CONFIGURED: LOCAL
   ```

Once verified, the DGX appliance is ready to communicate with your central Landscape Server and execute managed activities.

