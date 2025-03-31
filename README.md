# PXE Boot Automation with Puppet

This directory contains documentation and code samples for implementing PXE boot automation using Puppet.

## Table of Contents
- [Overview](#overview)
- [How PXE Boot Works](#how-pxe-boot-works)
- [Components and Requirements](#components-and-requirements)
- [Implementation Steps](#implementation-steps)
- [Configuration Files](#configuration-files)
- [Network Configuration](#network-configuration)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Contents](#contents)
- [Related Documentation](#related-documentation)

## Overview

The PXE (Preboot Execution Environment) boot system provides a mechanism for automatically deploying operating systems over the network. This implementation uses Puppet to manage all the components involved in the PXE boot process, enabling consistent, repeatable server deployments with minimal manual intervention.

## How PXE Boot Works

The PXE boot process follows these steps:

1. **Boot Initiation**: When a server is powered on, it broadcasts a DHCP request.

2. **DHCP Response**: The DHCP server responds with:
   - IP address, subnet mask, and default gateway
   - Next-server option (TFTP server address)
   - Filename option (boot loader file name)

3. **TFTP Boot File Download**: The client downloads the boot file (GRUB2 or PXELINUX) from the TFTP server.

4. **Boot Menu**: The boot file presents a menu of installation options.

5. **Kernel and Initrd Loading**: The client downloads the Linux kernel and initial RAM disk.

6. **Kickstart Location**: The installer reads the kickstart file location from the kernel parameters.

7. **OS Installation**: The automated installation proceeds according to the kickstart file.

8. **Post-Installation**: After installation, Puppet runs to configure the system.

## Components and Requirements

### Hardware Requirements
- **Server for PXE Infrastructure**: 
  - Minimum: 4 CPU cores, 8GB RAM, 100GB storage
  - Recommended: 8 CPU cores, 16GB RAM, 500GB+ storage
- **Network**: 
  - Gigabit Ethernet recommended
  - VLAN capability if using separate networks for PXE and regular traffic

### Software Components
1. **DHCP Server**:
   - ISC DHCP Server
   - DHCP options properly configured (next-server, filename)
   - Reserved IP addresses for servers

2. **TFTP Server**:
   - tftp-server package
   - Properly configured file permissions
   - Boot files (GRUB2, kernel, initrd)

3. **HTTP Server (Apache)**:
   - httpd package
   - Configuration for kickstart files and OS repository
   - Sufficient bandwidth for multiple concurrent installations

4. **Puppet Master**:
   - Puppet 6.x or newer
   - Required modules (stdlib, concat, etc.)
   - Properly configured CA certificates

5. **OS Installation Source**:
   - Complete OS repositories mirror
   - Sufficient disk space for multiple OS versions (50GB+ per OS)

6. **Network Configuration**:
   - UDEV rules for predictable interface naming
   - Support for bonding and VLAN configurations

### Puppet Modules
- **kickstart**: Generates kickstart files based on host configurations
- **tftp**: Manages TFTP server configuration
- **network**: Handles post-installation network setup
- **udev**: Ensures consistent network interface naming
- **dns**: Manages DNS entries for the hosts (optional)
- **dhcp**: Manages DHCP server configuration (optional)

## Implementation Steps

### 1. Set Up Base Infrastructure
1. Install the operating system on the PXE server.
2. Install required packages:
   ```bash
   dnf install httpd tftp-server dhcp-server bind puppet-agent git
   ```
3. Configure firewall to allow DHCP, TFTP, HTTP, and DNS traffic:
   ```bash
   firewall-cmd --permanent --add-service=dhcp --add-service=tftp --add-service=http --add-service=dns
   firewall-cmd --reload
   ```

### 2. Configure Puppet
1. Install Puppet Master or use an existing one.
2. Create/deploy the required Puppet modules:
   - Clone this repository or use the provided modules
   - Ensure module dependencies are satisfied
3. Configure the Puppet environment:
   ```bash
   mkdir -p /etc/puppetlabs/code/environments/production/modules
   cp -r puppet-pxe /etc/puppetlabs/code/environments/production/modules/kickstart
   ```

### 3. Configure DHCP Server
1. Set up the DHCP server with PXE options:
   ```
   option space pxelinux;
   option pxelinux.magic code 208 = string;
   option pxelinux.configfile code 209 = text;
   option pxelinux.pathprefix code 210 = text;
   option pxelinux.reboottime code 211 = unsigned integer 32;
   
   subnet 10.0.1.0 netmask 255.255.255.0 {
     range 10.0.1.100 10.0.1.200;
     option routers 10.0.1.1;
     option domain-name-servers 10.0.1.1;
     
     class "pxeclients" {
       match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
       next-server 10.0.1.10;
       filename "EFI/BOOT/grubx64.efi";
     }
   }
   ```

### 4. Configure TFTP Server
1. Create TFTP directory structure:
   ```bash
   mkdir -p /tftpboot/EFI/BOOT
   mkdir -p /tftpboot/linux/images/pxeboot
   mkdir -p /tftpboot/linux/kickstarts
   ```
2. Copy GRUB files and configure them:
   ```bash
   cp /boot/efi/EFI/BOOT/grubx64.efi /tftpboot/EFI/BOOT/
   ```

### 5. Set Up HTTP Server
1. Configure Apache for serving kickstart files:
   ```bash
   cat > /etc/httpd/conf.d/kickstart.conf << EOF
   Alias /linux/kickstarts /tftpboot/linux/kickstarts
   <Directory /tftpboot/linux/kickstarts>
     Options Indexes FollowSymLinks
     Require all granted
   </Directory>
   EOF
   ```
2. Configure Apache for serving OS repository:
   ```bash
   cat > /etc/httpd/conf.d/osrepo.conf << EOF
   Alias /linux /tftpboot/linux
   <Directory /tftpboot/linux>
     Options Indexes FollowSymLinks
     Require all granted
   </Directory>
   EOF
   ```

### 6. Prepare OS Installation Source
1. Mount installation media:
   ```bash
   mount -o loop linux.iso /mnt
   ```
2. Copy files to the HTTP server:
   ```bash
   cp -r /mnt/* /tftpboot/linux/
   ```

### 7. Configure Host Definitions
1. Define hosts in Hiera YAML file as shown in the `hosts.yaml` example.
2. Ensure all hosts have properly defined network configurations.

### 8. Run Puppet to Generate Configurations
1. Apply the Puppet configuration:
   ```bash
   puppet apply -e 'include kickstart include tftp include network include udev'
   ```
2. Verify that kickstart files have been generated.

## Configuration Files

### Host Definition (hosts.yaml)
The `hosts.yaml` file defines all servers and their network configurations. This file is the central configuration point for the PXE boot system. Example:

```yaml
hosts:
  - hostname: "webserver01"
    type: "web"
    ip_address: "10.0.1.10"
    swap_size: "2000"
    aliases: []
    network:
      primary_interface:
        device: "eth0"
        mac_address: "aa:bb:cc:dd:ee:10"
        bootproto: "dhcp"
        onboot: "yes"
        type: "Ethernet"
      second_interface:
        device: "eth1"
        mac_address: "aa:bb:cc:dd:ee:11"
        bootproto: "static"
        ipaddr: "10.0.2.10"
        netmask: "255.255.255.0"
        type: "Ethernet"
        onboot: "yes"
```

### Kickstart Template (ks.cfg.epp)
The kickstart template defines the automated installation process. Key sections include:

- **Language, keyboard, timezone**: Basic system settings
- **Network configuration**: Primary interface setup
- **Disk partitioning**: Storage layout based on host configuration
- **Package selection**: Core packages needed
- **Post-installation scripts**: Setup for Puppet and initial configuration

### GRUB Configuration (grub.cfg.epp)
This file defines the boot menu and kernel parameters for the PXE boot process. It uses the MAC address to determine the appropriate kickstart file.

### Network Configuration Script (configure_network.sh.erb)
This template generates a shell script for configuring complex network setups post-installation, including:

- Multiple interfaces
- Bond configurations
- VLAN setups

### UDEV Rules (70-persistent-net.rules.epp)
Ensures consistent network interface naming based on MAC addresses.

## Network Configuration

The network configuration system supports several network setups:

### 1. Simple Network (Single Interface)
- Primary interface only, typically configured via DHCP
- Minimal configuration needed in hosts.yaml

### 2. Multiple Interfaces
- Support for up to four separate network interfaces
- Each interface can have its own IP configuration
- Interface naming is consistent across reboots

### 3. Network Bonding
- Supports various bonding modes (active-backup, 802.3ad, etc.)
- Multiple slave interfaces per bond
- VLAN support on top of bonds

### 4. VLANs
- 802.1q VLAN tagging
- Multiple VLANs per physical interface or bond
- Proper kernel module loading for VLAN support

## Troubleshooting

### DHCP Issues
- **Problem**: Client not receiving DHCP address
  - **Check**: Verify DHCP server is running (`systemctl status dhcpd`)
  - **Check**: Confirm network connectivity between client and DHCP server
  - **Check**: Examine DHCP server logs (`journalctl -u dhcpd`)

### TFTP Issues
- **Problem**: Unable to download boot files
  - **Check**: Verify TFTP server is running (`systemctl status tftp`)
  - **Check**: Confirm file permissions in TFTP directory
  - **Check**: Test TFTP manually (`tftp <server> -c get EFI/BOOT/grubx64.efi`)

### Kickstart Issues
- **Problem**: Kickstart file not found
  - **Check**: Verify HTTP server is running (`systemctl status httpd`)
  - **Check**: Confirm kickstart file exists in correct location
  - **Check**: Test HTTP access to kickstart file

### Network Configuration Issues
- **Problem**: Interfaces not configured correctly after installation
  - **Check**: Verify UDEV rules are applied (`cat /etc/udev/rules.d/70-persistent-net.rules`)
  - **Check**: Examine Puppet logs (`tail -n 100 /var/log/puppetlabs/puppet/puppet.log`)
  - **Check**: Test network configuration script manually (`/root/configure_network.sh`)

## Security Considerations

- **Network Isolation**: Consider using a separate network for PXE boot traffic
- **Kickstart Authentication**: Implement secure methods for retrieving kickstart files
- **Post-Installation**: Remove or disable unnecessary services after installation
- **Firewall Configuration**: Enable firewall after installation completes
- **SSH Hardening**: Configure SSH properly to prevent unauthorized access

## Contents

- `/puppet-pxe`: Contains the Puppet module for PXE boot automation
- `/code`: Contains example code snippets and sample configurations

## Prerequisites

To implement this PXE boot solution, you need:

- A Puppet master server
- DHCP server (see the [dhcp] https://github.com/enemy100/puppet-dhcp module)
- HTTP server (Apache)
- TFTP server
- Access to OS installation media 
