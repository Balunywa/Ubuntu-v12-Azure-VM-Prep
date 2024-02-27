# Ubuntu-v12-Azure-VM-Prep Scripts for Migration

## Disclaimer

**IMPORTANT:** 

This repository and its contents are intended solely for testing purposes. It has been tested in 2 or 3 Ubuntu-v12 running in lab vSphere environments and is not recommended for production use. 

Upgrading to the latest operating systems is always advised for better security, features, and support. In scenarios where an upgrade is not feasible, these scripts may be considered as a workaround. However, users must proceed with caution, acknowledging that this approach is experimental and should be tested thoroughly before any attempt to use it in a critical environment. The scripts are specifically designed for preparing Ubuntu-v12 VMs for migration to Azure and involve configurations that may not be suitable for all users or environments.

### Serious Disclaimer

The scripts provided have been tested in a limited number of Ubuntu v12 virtual machine (VM) instances and are to be used **exclusively for testing purposes**. While the ultimate recommendation is to always migrate towards the latest operating systems for enhanced performance and security, there might be instances where upgrading is not a viable option. In such cases, and only after careful consideration, you might find these scripts useful. However, it's crucial to understand that these scripts are intended for testing environments only, and deploying them in production without thorough validation might lead to unforeseen issues.

## Introduction

This GitHub repository contains a collection of Bash scripts for preparing Ubuntu v12 VMs for migration to Azure. These scripts are designed to configure the VMs to ensure they function correctly post-migration. The preparation involves mounting file systems, rebuilding the initial ramdisk (initrd) for Hyper-V compatibility, enabling Azure Serial Console, configuring network settings, and installing the Azure Linux Guest Agent. Each script segment addresses a specific preparation task and is meant for execution in an Ubuntu v12 VM that is slated for migration to Azure.

## Pre-Migration Configuration Notice

Before migrating your VMs to Azure, it's essential to adjust the VMs' configurations to ensure their operability within Azure. Azure Migrate handles these configuration changes through the hydration process for supported versions of operating systems. For operating systems not listed, manual adjustments are required before migration. Failure to make these necessary changes could result in the VM not booting or issues with connectivity post-migration.

## Scripts Breakdown

### 1. Mount Root Partition

#This script segment defines the mount point and log file location, ensures the script is executed with root privileges, creates the mount point if it doesn't exist, and attempts to mount the root partition using its UUID.

```bash
#!/bin/bash

# Define log file location
LOG_FILE="/var/log/mount_root_partition.log"

# Function to log messages
function log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

# Ensure the script is run as root
if [ "$(id -u)" -ne 0 ]; then
    log_message "This script must be run as root."
    exit 1
fi

# Define the mount point
MOUNT_POINT="/mnt/azure_sms_root"

# Dynamically identify the root partition
ROOT_PARTITION=$(findmnt -n -o SOURCE /)

if [ -z "$ROOT_PARTITION" ]; then
    log_message "Unable to identify the root partition."
    exit 1
fi

log_message "Identified root partition: $ROOT_PARTITION"

# Check if the root partition is already correctly mounted at the mount point
if findmnt -rn -S "$ROOT_PARTITION" -T "$MOUNT_POINT"; then
    log_message "$ROOT_PARTITION is already correctly mounted on $MOUNT_POINT."
    exit 0
fi

# If the mount point directory does not exist, create it
if [ ! -d "$MOUNT_POINT" ]; then
    log_message "Creating mount point at $MOUNT_POINT."
    mkdir -p "$MOUNT_POINT"
else
    log_message "Mount point $MOUNT_POINT already exists."
fi

# Attempt to mount the root partition
log_message "Attempting to mount root partition ($ROOT_PARTITION) at $MOUNT_POINT."
mount "$ROOT_PARTITION" "$MOUNT_POINT" 2>&1 | tee -a "$LOG_FILE"

# Verify the mount was successful
if findmnt -rn -S "$ROOT_PARTITION" -T "$MOUNT_POINT"; then
    log_message "Successfully mounted root partition to $MOUNT_POINT."
else
    log_message "Failed to mount root partition. Please check logs and system status."
    exit 1
fi


```

### 2. Mount File System Partition

#Processes `/etc/fstab` entries to mount additional file systems, ensuring they are correctly bound within the VM's file system hierarchy.

```bash
#!/bin/bash

# Define a log file location
LOG_FILE="/tmp/mount_system_partitions.log"

# Function to log messages
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

mount_system_partitions() {
    # Ensure the script is run as root
    if [ "$(id -u)" -ne 0 ]; then
        log_message "This script must be run as root."
        exit 1
    fi

    fstab_file="/mnt/azure_sms_root/etc/fstab"
    if [[ -f "$fstab_file" ]]; then
        log_message "Processing /etc/fstab entries."

        while IFS= read -r line || [[ -n "$line" ]]; do
            # Skip comments and empty lines
            [[ "$line" =~ ^#.*$ ]] || [[ -z "$line" ]] && continue

            # Extract the device, mount point, and file system type
            read -r device mount_point fs_type <<< $(echo $line | awk '{print $1, $2, $3}')

            # Skip swap entries
            [[ "$fs_type" == "swap" ]] && continue

            # Determine the full path for the new mount point
            new_mount_point="/mnt/azure_sms_root${mount_point}"

            # Check if the partition is already mounted
            if findmnt -rn "$new_mount_point" > /dev/null 2>&1; then
                log_message "$device is already mounted on $new_mount_point."
                continue # Skip to the next iteration without attempting to remount
            fi

            # Create the mount point directory if it doesn't exist
            if [ ! -d "$new_mount_point" ]; then
                mkdir -p "$new_mount_point" 2>>"$LOG_FILE" && log_message "Created mount point $new_mount_point."
            fi

            # Mount the partition
            if mount --bind "$device" "$new_mount_point" 2>>"$LOG_FILE"; then
                log_message "Successfully mounted $device to $new_mount_point."
            # Removed the else block that logs the failure to mount
            fi
        done < "$fstab_file"
    else
        log_message "/etc/fstab not found in root partition."
        exit 1
    fi
}

# Execute the function
mount_system_partitions


```

### 3. Rebuild initrd

Backs up and rebuilds the initial ramdisk image for the current kernel version to ensure Hyper-V compatibility, crucial for Azure VMs.

```bash
#!/bin/bash

# Define log file location
LOG_FILE="/tmp/initrd_rebuild.log"

# Function to log messages
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

rebuild_initrd() {
    # Ensure the script is run as root
    if [ "$(id -u)" -ne 0 ]; then
        log_message "This script must be run as root."
        exit 1
    fi

    # Optionally accept a kernel version as an argument
    kernel_version="${1:-$(uname -r)}"
    log_message "Kernel version identified as $kernel_version."

    # Ensure running in the correct directory
    cd /boot

    # Check if the initrd image exists before attempting to back it up
    if [ ! -f "initrd.img-${kernel_version}" ]; then
        log_message "initrd image for kernel $kernel_version does not exist, cannot proceed."
        exit 1
    fi

    # Backup the existing initrd image
    log_message "Backing up the current initrd image for kernel $kernel_version..."
    if cp "initrd.img-${kernel_version}" "initrd.img-${kernel_version}.bak" 2>>"$LOG_FILE"; then
        log_message "Backup created: initrd.img-${kernel_version}.bak"
    else
        log_message "Failed to create backup for initrd.img-${kernel_version}."
        exit 1
    fi

    # Rebuild the initrd image with necessary modules for Hyper-V compatibility
    log_message "Rebuilding initrd image for Hyper-V compatibility..."
    if update-initramfs -u -k "${kernel_version}" 2>>"$LOG_FILE"; then
        log_message "initrd image rebuilt successfully for kernel $kernel_version."
    else
        log_message "Failed to rebuild initrd image for kernel $kernel_version."
        exit 1
    fi
}

# Execute the function with an optional kernel version argument
rebuild_initrd "$@"

```

### 4. Enable Azure Serial Console

Modifies the GRUB configuration to enable Azure Serial Console, allowing for troubleshooting and management of VMs directly from the Azure portal.

```bash
#!/bin/bash

# Set the GRUB command line parameters for a cloud environment and ensure console output
GRUB_CMDLINE_LINUX_DEFAULT="rootdelay=300 console=ttyS0 earlyprintk=ttyS0 net.ifnames=0"
GRUB_CMDLINE_LINUX="console=ttyS0 earlyprintk=ttyS0"

# Path to the GRUB configuration
GRUB_CONFIG="/etc/default/grub"

# Backup the original GRUB configuration file
sudo cp "$GRUB_CONFIG" "${GRUB_CONFIG}.bak"

# Update GRUB configuration file with the default parameters
sudo sed -i "s/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=\"$GRUB_CMDLINE_LINUX_DEFAULT\"/" "$GRUB_CONFIG"

# Update GRUB configuration file to ensure console output
sudo sed -i "s/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX=\"$GRUB_CMDLINE_LINUX\"/" "$GRUB_CONFIG"

# Remove the quiet and splash options to enable verbose boot messages
sudo sed -i '/GRUB_CMDLINE_LINUX_DEFAULT=/ s/" quiet splash"/""/' "$GRUB_CONFIG"

# Update the GRUB configuration
sudo update-grub

# Add the Hyper-V modules to initramfs
KERNEL_VERSION=$(uname -r)
INITRAMFS_PATH="/boot/initramfs-$KERNEL_VERSION.img"
BACKUP_INITRAMFS_PATH="/boot/initramfs-$KERNEL_VERSION.img.bak"

# Backup the current initramfs image
sudo cp "$INITRAMFS_PATH" "$BACKUP_INITRAMFS_PATH"

# Add the required modules
sudo dracut -f -v "$INITRAMFS_PATH" "$KERNEL_VERSION" --add-drivers "hv_vmbus hv_netvsc hv_storvsc"

# Update GRUB once more to ensure changes take effect
sudo update-grub

# Setup Upstart job for ttyS0 to ensure getty starts at boot
TTY_CONFIG="/etc/init/ttyS0.conf"

# Check if ttyS0.conf already exists to avoid duplicate entries
if [ ! -f "$TTY_CONFIG" ]; then
    echo "Creating ttyS0 Upstart job..."

    # Create the ttyS0 Upstart job
    echo "start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]

respawn
exec /sbin/getty -L 115200 ttyS0 vt102" | sudo tee "$TTY_CONFIG"

    echo "ttyS0 Upstart job created."
else
    echo "ttyS0 Upstart job already exists."
fi

echo "GRUB and initramfs configuration completed successfully."

# Restart the VM to apply changes
echo "A reboot is required to apply the changes. Would you like to reboot now? (y/n)"
read -r answer
if [ "$answer" != "${answer#[Yy]}" ]; then
    sudo reboot
else
    echo "Please remember to reboot the system manually to apply changes."
fi



```

### 5. Configure Network

##Disables udev's persistent network device naming and configures eth0 to use DHCP, ensuring network connectivity in Azure.

```bash
#!/bin/bash

# Define a log file location
LOG_FILE="/tmp/configure_network.log"

# Function to log messages
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

configure_network() {
    # Ensure the script is run as root
    if [ "$(id -u)" -ne 0 ]; then
        log_message "This script must be run as root."
        exit 1
    fi

    # Dynamically identify the primary network interface
    #primary_iface=$(ip route | grep default | sed -e "s/^.*dev.//" -e "s/.proto.*//")
    primary_iface=$(ip route | grep default | awk '{ print $5 }')
    log_message "Identified primary network interface: $primary_iface"

    # Backup existing network configuration
    datetime=$(date +"%Y%m%d-%H%M%S")
    backup_dir="/etc/network/interfaces.d/backup-$datetime"
    mkdir -p "$backup_dir"
    cp /etc/network/interfaces.d/* "$backup_dir" 2>/dev/null || true
    log_message "Backup of existing network configurations created in $backup_dir."

    # Disable udev's persistent network device naming rules
    log_message "Disabling udev persistent net rules."
    ln -s /dev/null /etc/udev/rules.d/75-persistent-net-generator.rules 2>>"$LOG_FILE"
    rm -f /etc/udev/rules.d/70-persistent-net.rules 2>>"$LOG_FILE"

    # Configure primary interface to use DHCP if not already configured
    iface_cfg="/etc/network/interfaces.d/${primary_iface}.cfg"
    if [ ! -f "$iface_cfg" ]; then
        log_message "Configuring $primary_iface to use DHCP."
        echo -e "auto $primary_iface\niface $primary_iface inet dhcp" > "$iface_cfg"
    else
        log_message "$primary_iface configuration already exists. Skipping DHCP configuration."
    fi

    # Note: Directly restarting networking services in a script can be risky, especially over SSH connections.
    log_message "Network configuration updated. Please manually restart the networking service if required."
}

# Execute the function
configure_network

```

# 6. Install the Azure Linux Guest Agent

Updates package sources, installs required packages, and installs the Azure Linux Guest Agent, which is necessary for managing VMs within Azure. Due to the employment of an unsupported and outdated operating system, manual downloading of the zip file followed by its transfer to the Ubuntu VM via SFTP is required. In my testing am using version 2.9.1.1 purely for testing purposes from this repo - https://github.com/Azure/WALinuxAgent/releases. You can obtain the most recent version from https://launchpad.net/ubuntu/+source/walinuxagent. Adjust the agent's name and path as per your selection prior to executing the respective function

```bash
#!/bin/bash
# Define log file and installation steps
# Steps to install Azure Linux Guest Agent omitted for brevity
# Install the Azure Linux Guest Agent

# Exit on any error
set -e

# Define log file
LOG_FILE="/var/log/azure_agent_install.log"

# Function to log messages
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" >> $LOG_FILE
}

log_message "Starting Azure Linux Agent installation script."

# Update sources to old-releases for Ubuntu 12.04
log_message "Updating sources.list to old-releases."
sed -i 's|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list
sed -i 's|http://.*.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list

# Update and install dependencies
log_message "Updating package lists."
apt-get update >> $LOG_FILE

log_message "Installing zip and other required packages."
apt-get install -y zip unzip python python-setuptools >> $LOG_FILE

# Assuming walinuxagent.zip is in /home/test
WALINUXAGENT_ZIP="/home/test/walinuxagent.zip"
WALINUXAGENT_DIR="/home/test/walinuxagent"

log_message "Unzipping walinuxagent.zip."
unzip -o $WALINUXAGENT_ZIP -d $WALINUXAGENT_DIR >> $LOG_FILE

# Navigate to the walinuxagent directory and install
cd $WALINUXAGENT_DIR/WALinuxAgent-*

log_message "Installing the Azure Linux Agent."
python setup.py install >> $LOG_FILE

# Start the walinuxagent service
log_message "Starting walinuxagent service."
service walinuxagent start

# Check if walinuxagent service is running
if service walinuxagent status | grep -q "running"; then
    log_message "walinuxagent service is running."
else
    log_message "Error: walinuxagent service is not running."
    exit 1
fi

log_message "Azure Linux Agent installation completed successfully."

# Return to the original directory
cd -

```

# Conclusion

The scripts provided in this repository are designed to assist in the preparation of Ubuntu v12 VMs for migration to Azure. They address common configuration requirements and compatibility issues. However, given the potential risks associated with deploying these scripts without proper testing, it's imperative to use them judiciously, ensuring thorough validation in a controlled test environment. The scripts are based off the steps outlined in this document - https://learn.microsoft.com/en-us/azure/migrate/prepare-for-agentless-migration
