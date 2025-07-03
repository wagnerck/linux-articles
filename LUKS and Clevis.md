# Auto-unlocking a LUKS-encrypted root volume with Clevis and a TPM (Debian/Ubuntu)

_v1_ : 07/03/2025 : Christian Wagner - wagnerck@fastmail.com : https://rainygarden.net/ : https://github.com/wagnerck : written by a human being and not a language model

The ability for Windows and Mac OS to automatically unlock an encrypted boot volume using security hardware (a TPM on PC equipment, the T2 chip on recent Macs) is a feature that Linux distributions mostly lack by default. You can implement this functionality yourself on most Linux installs, using a number of different tools (like `systemd-cryptenroll` and others), but these implementations can tend to be fragile. It becomes easy to lock yourself out of your boot device if something goes slightly wrong, and the initial configuration can be difficult and risky.

This article outlines a very quick and relatively robust method for automatically unlocking a LUKS-encrypted root volume via a Trusted Platform Module on Debian Linux (tested on version 12 "Bookworm" and up) and on Ubuntu (tested on v24.04 LTS "Noble Numbat" and up). It uses the Clevis decryption framework and makes minimal changes to the existing configuration; specifically, it works with traditional `mkinitramfs` and does not require switching to Dracut or another toolset for creating a boot image. It only uses packages from the default repositories, and does not require modifying any system files like `/etc/crypttab`. It can be configured with varying kinds of seals on the encryption keys (known as "PCRs"), so that hardware or firmware modifications will prevent the auto-unlock from occurring. And, most importantly I believe, it also leaves the existing method for manual unlocking in place, so the root volume can still be accessed if something goes awry (like one of the seals getting triggered accidentally).

Due to a lack of time and hardware resources, this method has only been tested on on actual physical hardware with Debian 13 "Trixie", with the other configurations tested in virtual machines. Additional testing by other folks would be great to hear about.

This method requires a supported Linux distro, a PC or virtual machine with a Trusted Platform Module (v2), and an existing encrypted root volume, preferably set up by the distro installer using LUKS and LVM during the initial system install, as those volumes will be the most consistent. These are the tested configurations:

* _Debian 12 and 13 installers_: choose the "Guided - Use entire disk and set up encrypted LVM" option
* _Ubuntu 24.04 LTS installer_: choose "Advanced Features" on the "Disk setup" page, and then "Use LVM and encryption"
* _Ubuntu 25.04 installer_: select "Encrypt with a passphrase" during the installation process

Encrypting an existing cleartext root partition, using more complicated LUKS/LVM configurations, using Clevis on other distributions, or using other drive encryption methods are currently outside the scope of this guide. These instructions should also not be performed on any system that already has an auto-unlock method configured, like with `systemd-cryptenroll` or a similar tool.

---

**DISCLAIMER**: These instructions are provided in good faith on an "as is" basis, and for informational purposes only. This document comes with no warranty, and no guarantee is given that these instructions will perform exactly as described. The author will not be liable for damages of any kind resulting from the use of these instructions, including but not limited to data loss, financial loss, or hardware damage.

---


## The quick install summary:

If you're in a hurry, these are the steps to implement a default setup with reasonable security parameters. All of these commands should be run from a root prompt, or via `sudo`. There will be more details on how this process works later on in this article.

### Step zero: back up your stuff

You should never start to experiment with changing drive encryption settings unless you are absolutely sure that all of your important files and data are backed up somewhere else. Everyone ought to be doing this anyway, but in this case it is especially important. Please don't continue unless you're comfortable with the possibility of losing everything on your root volume if something goes wrong.

### Step one: find the device with your encrypted root volume

Run the following command to give us a view of the current drive layout.

```
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

This is an example output from a Debian 13 "Trixie" system with a SATA SSD for the storage device, and where the "Guided - Use entire disk and set up encrypted LVM" option was chosen during installation:

```
NAME                        SIZE TYPE  FSTYPE      MOUNTPOINTS
sda                       238.5G disk              
├─sda1                      976M part  vfat        /boot/efi
├─sda2                      977M part  ext4        /boot
└─sda3                    236.6G part  crypto_LUKS 
  └─sda3_crypt            236.6G crypt LVM2_member 
    ├─hostname--vg-root   224.5G lvm   ext4        /
    └─hostname--vg-swap_1    12G lvm   swap        [SWAP] 
```

The results on your system will almost certainly not look like this. But any system that is capable of using the instructions in this article will have a device that says `crypto_LUKS` in the FSTYPE column. That is how we recognize it as the encrypted partition that contains the root volume, find the one that says `crypto_LUKS` and then follow the row back to the NAME column.

It will very likely be something along the lines of `sda3` for a SATA drive (as in this example), `vda3` for a drive within a virtual machine, or `nvme0n1p3` for an nVME drive. Remember this device name and/or write it down, as we'll need it a few times in later steps.

### Step two: install the required packages

This command will pull in the correct packages on Debian 12/13 and Ubuntu v24.04/v25.04.

```
apt-get install clevis clevis-luks clevis-tpm2 clevis-initramfs tpm2-tools
```

### Step three: test the TPM

Run the following command to have the Trusted Platform Module run a self-test:

```
tpm2_selftest -f
```

No output means the test has passed. If the test fails, the TPM may need to be enabled or reset in the system's firmware configuration (commonly if incorrectly called the "BIOS").

### Step four: use Clevis to bind the decryption key to the device and seal it in the TPM

This command needs to be edited to include the device found in step one. Replace `device` in this command with the appropriate device name. For example, it may be `sda3`, `vda3`, or `nvme0n1p3`, and will be specific to the system currently in use. The command will then will ask for your existing encryption password.

This command uses PCRs 1 and 7 to seal the encryption key, which will cause the auto-unlock to fail if the system's serial numbers (as stored in the firmware) are changed, if major hardware (like the CPU) is changed, or if the status of Secure Boot is changed (such as it being disabled). See the more detailed descriptions later in the article if you would like to change what PCRs are used to seal the key.

```
clevis luks bind -d /dev/device tpm2 '{"pcr_bank":"sha256", "pcr_ids":"1,7"}'
```

This command, if successful, will produce no output.

### Step five: verify the binding and seal

Run this command, again replacing "device" with the name of the device containing the root volume:

```
clevis luks list -d /dev/device
```

The output should look like this, if nothing was changed in the command in step three:

```
1: tpm2 '{"hash":"sha256","key":"ecc","pcr_bank":"sha256","pcr_ids":"1,7"}'
```

### Step six: rebuild the boot image

Run this command to rebuild the boot image so it includes Clevis and its configuration:

```
update-initramfs -k all -u
```

### Step seven: reboot

That should be everything you need to do. On restart, you will still see the unlock prompt come up for your boot device, but you can now ignore it. The system will automatically unlock the root volume and continue booting.




## The more detailed explanations:

### So what are we actually _doing_ here?

Let's start with explaining what LUKS is in the first place. Linux Unified Key Setup (LUKS) is a specification for drive encryption that allows the use of multiple decryption keys for the same volume, and the changing of those keys without having to re-encrypt the drive. This lets a drive be unlocked in multiple ways, like using as a manually-entered passphrase, or a decryption key retrieved from a Trusted Platform Module.

The TPM is a piece of hardware, embedded in nearly all modern PCs, that allows for the secure storage and retrieval of encryption and decryption keys. It includes a feature called Platform Configuration Registers (PCRs), which indicate the current security status of the hardware: whether Secure Boot is on or off, the presence of specific hardware configuration details, the version of the firmware, the boot loader setup, and many other measurements of the state of the system.

Like everything related to cryptography, this is extremely complicated in its implementation, but in short: a decryption key stored in the TPM can be sealed behind a specific set of PCRs, so that if aspects of the system change, that key will no longer be available. For example, if a decryption key is sealed behind the status of Secure Boot (as in the command in the quick install), and Secure Boot is then later disabled, then the TPM will no longer deliver the key.

This lets us use a decryption tool called Clevis to automate the unlocking of an encrypted drive, while at the same time choosing what system changes or events will keep that from happening. Such a configuration is obviously somewhat less secure than one that always requires the manual entry of a passphrase, but it is still much better than having a completely unencrypted drive, and significantly more convenient. Information security is _always_ about finding the right trade-off between convenience and safety, and these tools allow you to choose from a variety of failsafes depending on your individual needs. Or you can choose to just not set up auto-unlock at all.

### How does Clevis work?

Let's look at the example command from the quick install summary:

```
clevis luks bind -d /dev/device tpm2 '{"pcr_bank":"sha256", "pcr_ids":"1,7"}'
```

When this command is run, it is "binding" a particular decryption method to a specific encrypted storage device, so that the Clevis framework knows how to automatically unlock it, and "sealing" the key inside the TPM at the same time. Let's break it down further.

`clevis` is the executeable file called by the operating system, `luks` is the command noun that tells Clevis that we will be operating on an encrypted LUKS device, and `bind` is the command verb that tells Clevis that we will be binding that device to a decryption method (or "policy" as the Clevis documentation calls it).

`-d /dev/device` is the parameter that specifies the LUKS device in question, and `tpm2` is the policy type, where we store and seal a decryption key in the TPM. Most systems only have one TPM, so we don't normally have to specify which one to use.

The last part is the most complicated one. `{"pcr_bank":"sha256", "pcr_ids":"1,7"}` is a set of two parameters that are being passed to Clevis to specify how the decryption key will be stored in the TPM. `pcr_bank` tells it to use the SHA256 cryptographic hash function for storing PCR values (which is more secure than other alternatives), and `pcr_ids` tells it which Platform Configuration Registers it should use to protect the decryption key. This protection is called "sealing" in this context.

PCR 1 uses the system's firmware configuration, including its embedded serial numbers, and the system's CPU and RAM configuration to monitor for changes in the system setup. PCR 7 monitors whether Secure Boot is on or off, and whether any Secure Boot certificates have been changed. These provide a basic level of protection again the most common kinds of attacks against an encrypted drive, and since they change very rarely if at all, these seals will be unlikely to cause a false alarm. Later in this article we'll go over some of the other options for PCRs which can harden a system further, while increasing the risk that a seal will be tripped and auto-unlock will stop working.

### What happens on startup to unlock the root volume?

When we set up Clevis, we installed several packages, including `clevis`, `clevis-luks`, `clevis-tpm2`, and `clevis-initramfs`. Aside from the basic tool itself, these packages each add functionality to the Clevis framework: `clevis-luks` allows it to decrypt LUKS drives, and `clevis-tpm2` allows it to use a Trusted Platform Module, like described previously. And `clevis-initramfs` is where things get very interesting.

An "initramfs" is also known as a "boot image", and is short for "Initial RAM File System". It is a special drive image loaded by the Linux startup process immediately after the bootloader, and it contains the drivers and tools necessary for loading the rest of the operating system. The boot image is always unencrypted so that the system's firmware can access it, and it's where the tools for decrypting an encrypted root volume live.

(If you run `lsblk` again, like you did at the start of the quick install summary, you can most likely see that there is a `/boot` partition that is separate from the root volume. This is where the initramfs resides, in a partition set up to be easily accessible to the system's boot loader.)

The `clevis-initramfs` package adds a "hook" to the toolchain that Linux uses to generate an boot image, so that when a command like `update-initramfs` is run, it will automatically include the appropriate parts of Clevis framework in it. When the startup code in the boot image is run, it also runs the Clevis tools that pull the decryption key from the TPM, unlock the root volume, and tell the system to continue booting. This happens basically automatically with the proper packages installed, once Clevis is configured correctly.

What's interesting about Clevis is that it can coexist with other methods for unlocking the root volume, so that if something goes wrong, the regular methods (like manually entering in a passphrase) will continue to work, since they are implemented with other tools entirely. This makes it more robust than some other auto-unlock options, although some people are likely to dislike the fact that the manual unlock prompt still appears even as the drive is being automatically unlocked.




## Repairing a broken Clevis seal:

If the sealed encryption key becomes unavailable because a change in the PCR registers triggers a seal, the manual unlock mechanism (i.e. typing in the passphrase instead) should continue to work even if the Clevis auto-unlock does not. Undoing whatever caused the change to the PCR status should usually re-enable the auto-unlock.

In a situation where the change cannot be undone, or where the change was deliberate and needs to remain, the sealed encryption passphrase can be deleted and re-created to use the new PCR values. This procedure may also be required after a major operating system upgrade, but that usually depends on what PCRs you chose for the seal.

### Step one: identify the encrypted root volume again

This should not have changed from the time when Clevis was installed and configured, but just in case you've forgotten, you can run `lsblk` again per the original install instructions to find the device name for the encrypted root volume. As before, it will typically be something like `sda3`, `vda3`, or `nvme0n1p3`, but could be something else.

### Step two: confirm the existing (but nonfunctional) key and seal

Run the following command, replacing `device` with the actual device name:

```
clevis luks list -d /dev/device
```

The output should still look something like this:

```
1: tpm2 '{"hash":"sha256","key":"ecc","pcr_bank":"sha256","pcr_ids":"1,7"}'
```

Make a note of the number at the beginning of the output. It will normally be `1` for the `tpm2` method, but we should double-check.

### Step three: remove the existing key and seal

Run the following command, replacing `device` with the actual encrypted device name. If the output of the previous step was not `1`, replace the digit in `-s 1` with the correct one.

```
clevis luks unbind -d /dev/device -s 1
```

This will ask you to confirm the removal of the seal, and ask you for your manual decryption passphrase.

### Step four: confirm the key and seal removal

Run the same command from step two. There should now be no output at all.

### Step five: re-bind and re-seal the encryption key

Run the following command, the same as we did during the original setup, replacing `device` with the actual encrypted device name.

```
clevis luks bind -d /dev/device tpm2 '{"pcr_bank":"sha256", "pcr_ids":"1,7"}'
```

###  Step six: confirm the new seal

Run the same command from step two or step four. It should now report something similar to this again:

```
1: tpm2 '{"hash":"sha256","key":"ecc","pcr_bank":"sha256","pcr_ids":"1,7"}'
```

### Step six: rebuild the boot image

Run this command to rebuild the boot image, just like we did the first time:

```
update-initramfs -k all -u
```

### Step seven: reboot

The auto-unlock should now be working again.




## Platform Configuration Registers

This is an incomplete list of firmware-level PCRs that one might find useful for sealing an encryption key with Clevis. You can find more complete lists at [the Arch Linux Wiki](https://wiki.archlinux.org/title/Trusted_Platform_Module), [this Arch Linux manpage](https://man.archlinux.org/man/systemd-cryptenroll.1#TPM2_PCRs_and_policies), and [the Linux TPM PCR Registry](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/).

* PCR 0: _Core system firmware executable code_ - changes to the system firmware itself will trigger the seal, and the key and seal will need to be recreated per the earlier instructions unless the firmware is rolled back
* PCR 1: _Core system firmware data_ - certain changes to the firmware's configuration will trigger the seal, including changes to the number of CPU cores, the amount of RAM installed, recorded serial or model numbers, and possibly other options like the default boot order, depending on the firmware itself
* PCR 2: _External or pluggable code_ - adding or removing external option ROMs (like the firmware on a PCI-E card) can trigger the seal
* PCR 3: _External or pluggable data_ - changing the configuration of external option ROMs can trigger the seal
* PCR 4: _Boot loader and additional drivers_ - changes to the Linux bootloader and its extensions can trigger the seal, requiring either a rollback or the recreation of the key and seal
* PCR 5: _Boot manager configuration and data_ - changes to the partition table can trigger the seal
* PCR 7: _Secure Boot Policy_ - changes to the Secure Boot settings can trigger the seal

PCRs 1 and 7 are the defaults for the example commands in this article, and it's generally believed that these are also the PCRs that Microsoft Windows uses to secure its Bitlocker drive encryption system. If you do not expect to ever upgrade your firmware, PCR 0 is a reasonable addition to the set. PCR 4 can also prevent attacks based on downgrading the boot loader code, but this will also trigger the seals on boot loader _upgrades_, which might happen during a normal system upgrade.

PCRs 8 and above are typically modified after the boot image loads, and are unlikely to have any useful security benefits for Clevis.




## Other sources used in developing this article:

* [https://github.com/latchset/clevis/](https://github.com/latchset/clevis/) - the Clevis framework itself
* [https://silvenga.com/posts/tpm-luks-unlock/](https://silvenga.com/posts/tpm-luks-unlock/) - another post on this topic, but with a different focus and a method that uses Dracut instead
* [https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/) - the full list of TPM Platform Configuration Registers (PCRs)
* [https://wiki.archlinux.org/title/Clevis](https://wiki.archlinux.org/title/Clevis) and [https://wiki.archlinux.org/title/Trusted_Platform_Module](https://wiki.archlinux.org/title/Trusted_Platform_Module) - the Arch Linux documentation, ever-useful even for other distros
