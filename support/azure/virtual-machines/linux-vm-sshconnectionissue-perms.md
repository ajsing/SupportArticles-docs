---
title: Troubleshoot SSH connection issues in Linux VM due to permission and ownership issues 
description: Resolves an issue in which SSH fails because the /var/empty/sshd file isn't owned by the root directory and isn't group or world-writable.
ms.date: 05/04/2023
author: saimsh-msft
ms.reviewer: vilibert, saimsh-msft
ms.service: virtual-machines
ms.subservice: vm-cannot-connect
ms.collection: linux
---
# Troubleshoot SSH connection issues in Linux VM due to permission and ownership issues 

This article provides a solution to an issue in which SSH service fails because the /var/empty/sshd file isn't owned by the root directory and isn't group or world-writable.

## Symptoms

You can't connect a Linux virtual machine (VM) by using a secure shell (SSH) connection due to permission and ownership issues. When this problem occurs, you may receive the following error message about the /var/empty/sshd file, depending on your Linux distribution.

**SuSE**

> /var/empty must be owned by root and not group or world-writable.  
startproc: exit status of parent of /usr/sbin/sshd: 255  
Failed

**CentOS/RHEL**  

> Starting sshd: /var/empty/sshd must be owned by root and not group or world-writable.  
[FAILED]

## Cause

This problem may occur if the /var/empty/sshd file is not owned by the root directory and is not group-writable or world-writable.

## Resolution

There are two ways to resolve the issue:
* Repair the VM online
    * [Serial Console](#serial-console)
    * [Run Command Extension](#run-command-extension)
* Repair the VM offline
    * [Use Azure Linux Auto Repair (ALAR)](#use-azure-linux-auto-repair-alar)
    * [Use Manual Method](#use-manual-method)

### Repair the VM online

#### Serial Console

1. Connect to [the serial console](./serial-console-linux.md) of the VM from Azure portal.
2. Login to the VM using local administrative account and its corresponding credential/password.
3. Gain Root privilege using the `sudo su -` command.
4. Run the following commands to resolve the permission and ownership issue:
```bash
chmod 755 /var/empty/sshd
chown root:root /var/empty/sshd
systemctl restart sshd
```
### Run Command Extension

This method relies on the Azure Linux agent (WAagent). It's required that the VM has the agent installed and service running. 

Open the **Properties** window of the VM in the Azure portal to check the agent status. If the agent is enabled and has "Ready" status, follow these steps to change the permission:

1. Go to the Azure portal, locate your VM settings, and then select **Run Command**  under **Operations** section.
2. Next, select **RunShellScript** and **Run** the following shell script:

    > [!NOTE]
    > You must update the script to reflect your system distribution. This script runs on Red Hat variants only.

    ```bash
    #!/bin/bash
    
    #Script to change permissions on a file
   chmod 755 /var/empty/sshd;chown root:root /var/empty/sshd;systemctl restart sshd

    ```

4. After the script execution is completed, the output console window will provide "Enable succeeded" message.


If you can connect to the VM by using the SSH connection, and you want to analyze the details of the Run-command script execution, examine the **handler.log** file in the `/var/log/azure/run-command` directory.

### Repair the VM offline

If the VM serial console access isn't available and the Waagent is not ready, an alternative solution is to repair the vm offline. There are two ways to take an offline approach:

#### Use Azure Linux Auto Repair (ALAR)

Azure Linux Auto Repair (ALAR) scripts are a part of VM repair extension described in [Repair a Linux VM by using the Azure Virtual Machine repair commands](./repair-linux-vm-using-azure-virtual-machine-repair-commands.md).
Implement the following steps to automate the manual offline process:

1. Use the [az vm repair create](/cli/azure/vm/repair#az-vm-repair-create) command to create a repair VM. The repair VM has a copy of the OS disk for the problematic VM attached.
```azurecli-interactive
az vm repair create --verbose -g centos7 -n cent7 --repair-username rescue --repair-password 'password!234' --copy-disk-name  repairdiskcopy
 ```
2. Login to the repair vm. Mount and chroot to the filesystem of the attached copy of OS disk. Follow the detailed [chroot instructions](./chroot-environment-linux.md).
3. Next, follow the same steps to resolve the permission and ownership issue:
 ```bash
chmod 755 /var/empty/sshd
chown root:root /var/empty/sshd
systemctl restart sshd
```
4. Once the changes are applied, `az vm repair restore` command can be used to perform automatic OS disk swap with the original VM as shown below:
```azurecli-interactive
az vm repair restore --verbose -g centos7 -n cent7
 ``` 

> [!Note] 
   >The resource group name "centos7, vm name "cent7", and --copy-disk-name "repairdiskcopy" are examples and the values need to change accordingly.

#### Use Manual Method

If both serial console and ALAR approach isn't possible or fails, the repair has to be performed manually. Follow the steps here to manually attach the OS disk to a recovery VM and swap the OS disk back to the original VM:
* [Attach the OS disk to a recovery VM using the Azure portal](./troubleshoot-recovery-disks-portal-linux.md)
* [Attach the OS disk to a recovery VM using Azure CLI](./troubleshoot-recovery-disks-linux.md)

Once the OS disk is successfully attached to the recovery VM, follow the detailed [chroot instructions](./chroot-environment-linux.md) to mount and chroot to the filesystems of the attached OS disk. Then, implement the same steps to resolve permission and ownership issues.

[!INCLUDE [Azure Help Support](../../includes/azure-help-support.md)]
