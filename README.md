# Ubuntu-v12-Azure-VM-Prep


# Ubuntu v12 Preparation Scripts for Migration

## Disclaimer

**IMPORTANT:** This repository and its contents are intended solely for testing and educational purposes. It has been tested in 2 or 3 Ubuntu-v12 running on lab vSphere environments and is not recommended for production use. Upgrading to the latest operating systems is always advised for better security, features, and support. In scenarios where an upgrade is not feasible, these scripts may be considered as a workaround. However, users must proceed with caution, acknowledging that this approach is experimental and should be tested thoroughly before any attempt to use it in a critical environment. The scripts are specifically designed for preparing Ubuntu-v12 VMs for migration to Azure and involve configurations that may not be suitable for all users or environments.

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
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

# Ensure the script is run as root
if [ "$(id -u)" -ne 0 ]; then
    log_message "This script must be run as root."
    exit 1
fi

# Define the mount point
MOUNT_POINT="/mnt/azure_sms_root"

# Create the mount point if it doesn't exist
if [ ! -d "$MOUNT_POINT" ]; then
    log_message "Creating mount point at $MOUNT_POINT."
    mkdir -p "$MOUNT_POINT"
else
    log_message "Mount point $MOUNT_POINT already exists."
fi

# Attempt to find the root partition by UUID and mount it
ROOT_PARTITION_UUID=$(blkid -o value -s UUID /dev/sda1)
if [ -n "$ROOT_PARTITION_UUID" ]; then
    log_message "Attempting to mount root partition with UUID $ROOT_PARTITION_UUID at $MOUNT_POINT."
    mount UUID="$ROOT_PARTITION_UUID" "$MOUNT_POINT" 2>&1 | tee -a "$LOG_FILE"
    
    if mountpoint -q "$MOUNT_POINT"; then
        log_message "Successfully mounted root partition to $MOUNT_POINT."
    else
        log_message "Failed to mount root partition."
        exit 1
    fi
else
    log_message "Unable to find UUID for /dev/sda1."
    exit 1
fi
```

### 2. Mount File System Partition

#Processes `/etc/fstab` entries to mount additional file systems, ensuring they are correctly bound within the VM's file system hierarchy.

```bash
#!/bin/bash
# Define a log file location and function to mount system partitions
# Function details and implementation omitted for brevity
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
            
            # Create the mount point directory if it doesn't exist
            mkdir -p "$new_mount_point" 2>>"$LOG_FILE" && log_message "Created mount point $new_mount_point."
            
            # Mount the partition
            log_message "Mounting $device to $new_mount_point"
            mount --bind "$device" "$new_mount_point" 2>>"$LOG_FILE" && log_message "Successfully mounted $device to $new_mount_point." || {
                log_message "Failed to mount $device on $new_mount_point"
                continue
            }
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
# Define a log file location and rebuild initrd function
# Function details and implementation omitted for brevity
#initrd rebuild function
# Define a log file location
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

    # Dynamically determine the kernel version
    kernel_version=$(uname -r)
    log_message "Kernel version identified as $kernel_version."

    # Ensure running in the correct directory
    cd /boot

    # Backup the existing initrd image
    log_message "Backing up the current initrd image for kernel $kernel_version..."
    cp initrd.img-${kernel_version} initrd.img-${kernel_version}.bak 2>>"$LOG_FILE" && \
    log_message "Backup created: initrd.img-${kernel_version}.bak"

    # Rebuild the initrd image with necessary modules for Hyper-V compatibility
    log_message "Rebuilding initrd image for Hyper-V compatibility..."
    update-initramfs -u -k ${kernel_version} 2>>"$LOG_FILE" && \
    log_message "initrd image rebuilt successfully for kernel $kernel_version."
}

# Call the function
rebuild_initrd
```

### 4. Enable Azure Serial Console

Modifies the GRUB configuration to enable Azure Serial Console, allowing for troubleshooting and management of VMs directly from the Azure portal.

```bash
#!/bin/bash
# Define a log file location and function to enable Azure Serial Console
# Function details and implementation omitted for brevity
#seriall logging

# Define a log file location
LOG_FILE="/tmp/enable_azure_serial_console.log"

# Function to log messages
log_message() {
    echo "$(date +"%Y-%m-%d %H:%M:%S") - $1" | tee -a "$LOG_FILE"
}

enable_azure_serial_console() {
    grub_cfg="/mnt/azure_sms_root/boot/grub/grub.cfg"
    backup_grub_cfg="${grub_cfg}.bak"

    # Ensure the script is run as root
    if [ "$(id -u)" -ne 0 ]; then
        log_message "This script must be run as root."
        exit 1
    fi

    if [[ -f "$grub_cfg" ]]; then
        # Back up the existing GRUB configuration file
        cp "$grub_cfg" "$backup_grub_cfg" && \
        log_message "Backed up the GRUB config to $backup_grub_cfg."

        # Add console parameters to GRUB_CMDLINE_LINUX for Azure Serial Console
        log_message "Modifying $grub_cfg for Azure Serial Console support."
        
        # Safely add serial console settings to GRUB_CMDLINE_LINUX, avoiding duplicate entries
        sed -i '/GRUB_CMDLINE_LINUX=/ s/"$/ console=ttyS0,115200n8 earlyprintk=ttyS0,115200 rootdelay=300"/' "$grub_cfg"

        # Update GRUB
        if chroot /mnt/azure_sms_root update-grub; then
            log_message "Successfully updated GRUB configuration for Azure Serial Console support."
        else
            log_message "Failed to update GRUB. Check the log for details."
            exit 1
        fi
    else
        log_message "GRUB configuration file not found."
        exit 1
    fi
}

# Execute the function
enable_azure_serial_console
```

### 5. Configure Network

##Disables udev's persistent network device naming and configures eth0 to use DHCP, ensuring network connectivity in Azure.

```bash
#!/bin/bash
# Define a log file location and network configuration function
# Function details and implementation omitted for brevity
#Configure Network

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

    # Disable udev's persistent network device naming rules
    log_message "Disabling udev persistent net rules."
    ln -s /dev/null /etc/udev/rules.d/75-persistent-net-generator.rules 2>>"$LOG_FILE"
    rm -f /etc/udev/rules.d/70-persistent-net.rules 2>>"$LOG_FILE"

    # Check if the network interface configuration for eth0 exists
    eth0_cfg="/etc/network/interfaces.d/eth0.cfg"
    if [ ! -f "$eth0_cfg" ]; then
        # If not, configure eth0 to use DHCP
        log_message "Configuring eth0 to use DHCP."
        mkdir -p /etc/network/interfaces.d
        echo -e "auto eth0\niface eth0 inet dhcp" > "$eth0_cfg"
    else
        log_message "eth0 configuration already exists. Skipping DHCP configuration."
    fi

    # Additional network configuration changes can be added here

    # Restart networking to apply changes (Use with caution; may disrupt SSH sessions)
    # service networking restart || {
    #    log_message "Failed to restart networking service."
    #    exit 1
    # }
    log_message "Network configuration updated. Please restart the networking service manually."
}

# Execute the function
configure_network
```

# 6. Install the Azure Linux Guest Agent

Updates package sources, installs required packages, and installs the Azure Linux Guest Agent, which is necessary for managing VMs within Azure. Due to the employment of an obsolete operating system, manual downloading of the zip file followed by its transfer to the Ubuntu VM via SFTP is required. You can obtain the most recent version from the walinuxagent package : Ubuntu (launchpad.net). Adjust the agent's name and path as per your selection prior to executing the respective function

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

The scripts provided in this repository are designed to assist in the preparation of Ubuntu v12 VMs for migration to Azure. They address common configuration requirements and compatibility issues. However, given the potential risks associated with deploying these scripts without proper testing, it's imperative to use them judiciously, ensuring thorough validation in a controlled environment before any production use.
