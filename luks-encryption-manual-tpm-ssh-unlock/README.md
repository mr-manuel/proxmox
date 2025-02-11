# HOW TO - Encrypt complete Proxmox node with LUKS

I was searching myself a way on how to get a full encrypted node, but found no step by step how to, so here it is.

**NOTE:** I'm not responsible, if you break something. Make always sure to have a backup before you begin.

This HOW TO works for an existing or fresh Proxmox VE installation, but currently needs a ZFS RAID1 (mirror) and GRUB as boot loader. The HOW TO for other disk configurations and filesystems will follow. You are welcome to help to add those steps :-)

Upvote the feature request [here](https://bugzilla.proxmox.com/show_bug.cgi?id=6160) to get LUKS configuration integrated in the Proxmox installer. For reference here is also the [Proxmox Forum](https://forum.proxmox.com/threads/feature-request-add-luks-to-installer.162036/) entry, but the upvotes are needed in the Proxmox Bugzilla instance.

If you already have LUKS configured, you can use this guide to make the system bootable again or add unlock options.

What this HOW TO cover:

1. [Fresh installation](#fresh-installation)

   - [ZFS RAID1 (mirror)](#zfs-raid1-mirror)
2. [Things to check before starting](#things-to-check-before-starting)

   - [ZFS RAID1 (mirror)](#zfs-raid1-mirror-1)
3. [Backup data](#backup-data)
4. [Install requirements](#install-requirements)
5. [Enable LUKS](#enable-luks)

   - [ZFS RAID1 (mirror)](#zfs-raid1-mirror-2)
       - Remove one ZFS mirror disk
       - Delete all data on the removed partition and create a LUKS (crypted) partition
       - Mount the crypted volume
       - Resilver the empty crypted LUKS disk with the other disk that has still the data on it
       - Repeat the same step for the other disk
6. [Fix the boot procedure](#fix-the-boot-procedure)

    Reconfigure the boot loader to be able to boot and asking for the LUKS password

   - [OPTIONAL: Add possibility to unlock via SSH (dropbear-initramfs)](#optional-add-possibility-to-unlock-via-ssh-dropbear-initramfs)
   - [OPTIONAL: Add automated unlock via TPM](#optional-add-automated-unlock-via-tpm)

# Fresh installation

## ZFS RAID1 (mirror)

Start the Proxmox VE installer. In the `Target harddisk` step choose `Advanced options` and select `ZFS (RAID1)`, then proceed with the installation.

![Target harddisk](./img/target-harddisk.png)

![Filesystem](./img/filesystem.png)

# Things to check before starting

The following steps depend on your selected filesystem and disk configuration. Currently only the HOW TO for `ZFS RAID1 (mirror)` is completed.

Meanwhile the single disk instructions are work in progress, you can check this tutorial: https://forum.proxmox.com/threads/adding-full-disk-encryption-to-proxmox.137051/

## Single disk (🚨 HOW TO NOT YET COMPLETED, DO NOT TRY)

1. Reboot your node and make sure GRUB is used. You will see something like this on boot:

    ![GRUB](./img/grub.png)

2. Your current disk setup should look similar to something like this:

    **ZFS**
    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda         8:0    0  1.8T  0 disk
    ├─sda1      8:1    0 1007K  0 part
    ├─sda2      8:2    0    1G  0 part
    └─sda3      8:3    0  1.8T  0 part
    ```

    or

    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    nvme0n1     259:0    0  1.8T  0 disk
    ├─nvme0n1p1 259:1    0 1007K  0 part
    ├─nvme0n1p2 259:2    0    1G  0 part
    └─nvme0n1p3 259:3    0  1.8T  0 part
    ```

    **EXT4**
    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda                  8:0    0  1.8T  0 disk
    ├─sda1               8:1    0 1007K  0 part
    ├─sda2               8:2    0    1G  0 part
    └─sda3               8:3    0  1.8T  0 part
      ├─pve-swap       252:0    0  1.9G  0 lvm  [SWAP]
      ├─pve-root       252:1    0  1.8T  0 lvm  /
      ├─pve-data_tmeta 252:2    0    1G  0 lvm
      │ └─pve-data     252:4    0  1.8T  0 lvm
      └─pve-data_tdata 252:3    0  1.8T  0 lvm
        └─pve-data     252:4    0  1.8T  0 lvm
    ```

    or

    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    nvme0n1            259:0    0  1.8T  0 disk
    ├─nvme0n1p1        259:1    0 1007K  0 part
    ├─nvme0n1p2        259:2    0    1G  0 part
    └─nvme0n1p3        259:3    0  1.8T  0 part
      ├─pve-swap       252:0    0  1.9G  0 lvm  [SWAP]
      ├─pve-root       252:1    0  1.8T  0 lvm  /
      ├─pve-data_tmeta 252:2    0    1G  0 lvm
      │ └─pve-data     252:4    0  1.8T  0 lvm
      └─pve-data_tdata 252:3    0  1.8T  0 lvm
        └─pve-data     252:4    0  1.8T  0 lvm
    ```

    **BTRFS**
    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda      8:0    0  1.8T  0 disk
    ├─sda1   8:1    0 1007K  0 part
    ├─sda2   8:2    0    1G  0 part
    └─sda3   8:3    0  1.8T  0 part /
    ```

    or

    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    nvme0n1     259:0    0  1.8T  0 disk
    ├─nvme0n1p1 259:1    0 1007K  0 part
    ├─nvme0n1p2 259:2    0    1G  0 part
    └─nvme0n1p3 259:3    0  1.8T  0 part /
    ```

## ZFS RAID1 (mirror)

1. Reboot your node and make sure GRUB is used. You will see something like this on boot:

    ![GRUB](./img/grub.png)

2. Your current disk setup should look similar to something like this:

    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    sda         8:0    0   1.8T  0 disk
    ├─sda1      8:1    0  1007K  0 part
    ├─sda2      8:2    0     1G  0 part
    └─sda3      8:3    0   1.8T  0 part
    sdb         8:16   0   1.8T  0 disk
    ├─sdb1      8:17   0  1007K  0 part
    ├─sdb2      8:18   0     1G  0 part
    └─sdb3      8:19   0   1.8T  0 part
    ```

    or

    ```bash
    root@pve01:~# lsblk | grep -v zd
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    nvme0n1     259:0    0   1.8T  0 disk
    ├─nvme0n1p1 259:1    0  1007K  0 part
    ├─nvme0n1p2 259:2    0     1G  0 part
    └─nvme0n1p3 259:3    0   1.8T  0 part
    nvme1n1     259:4    0   1.8T  0 disk
    ├─nvme1n1p1 259:5    0  1007K  0 part
    ├─nvme1n1p2 259:6    0     1G  0 part
    └─nvme1n1p3 259:7    0   1.8T  0 part
    ```

# Backup Data

Before proceeding, ensure you have a backup of your data from the ZFS pool. These steps will result in data loss if not done correctly.

```bash
dd if=/dev/sda3 bs=4M of=/path/to/backup/sda3.img
```

# Install requirements

During the whole process you will see multiple times this errors:

```
cryptsetup: ERROR: Couldn't resolve device rpool/ROOT/pve-1
cryptsetup: WARNING: Couldn't determine root device
```

I did not manage to understand what caused them exactly, but everything still worked fine.

Install the required software:

```bash
apt update
apt install -y cryptsetup cryptsetup-initramfs
```

Now you can run a benchmark test to see, if the CPU power will limit your drive performance:

```bash
cryptsetup benchmark
```


# Enable LUKS

The following steps depend on your selected filesystem and disk configuration. Currently only the HOW TO for `ZFS RAID1 (mirror)` is completed.

## Single disk (🚨 HOW TO NOT YET COMPLETED, DO NOT TRY)

Not yet completed.

## ZFS RAID1 (mirror)

Check the name of your disks and partitions with

```bash
lsblk | grep -v zd
```

In this guide I will use `sda3` and `sdb3`. in case your disk names are different, you have to change them in the commands.

Check the current ZFS pool status this command and check that everything is `ONLINE` and not `DEGRADED`.

```bash
zpool status
```

Take the `sdb3` offline and remove it from the ZFS mirror:

```bash
zpool offline rpool sdb3
zpool detach rpool sdb3
```

Check the pool status again to ensure the disk was successfully removed:

```bash
zpool status
```

Now that `sdb3` is offline and removed from the pool, you can encrypt it using LUKS. Enter this command to encrypt with 512-bit.

NOTE: The key size you see (512-bit) is actually two 256-bit. AES-XTS requires two keys: one for encryption and one as a tweak key.

```bash
cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 --use-random /dev/sdb3
```

confirm with `YES` (needs to be uppercase) and then enter your encryption password.

Open the LUKS-encrypted partition:

```bash
cryptsetup luksOpen /dev/sdb3 luks-sdb3
```

Now that `sdb3` is encrypted, you can add it back to the ZFS pool as a mirror:

```bash
zpool attach rpool sda3 /dev/mapper/luks-sdb3
```

The system will begin to resilver and copy the data from `sda3` onto the new `luks-sdb3` disk.

Wait for the resilvering process to finish. This may take some time depending on the amount of data in your pool.

You can monitor the progress by checking:

```bash
zpool status
```

Once the resilvering is complete, proceed with encrypting `sda3` in the same way:

Take the `sda3` offline and remove it from the ZFS mirror:

```bash
zpool offline rpool sda3
zpool detach rpool sda3
```

Check the pool status again to ensure the disk was successfully removed:

```bash
zpool status
```

Now that `sda3` is offline and removed from the pool, you can encrypt it using LUKS. Enter this command to encrypt with 512-bit.

NOTE: The key size you see (512-bit) is actually two 256-bit. AES-XTS requires two keys: one for encryption and one as a tweak key.

```bash
cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 --use-random /dev/sda3
```

confirm with `YES` (needs to be uppercase) and then enter your encryption password.

Open the LUKS-encrypted partition:

```bash
cryptsetup luksOpen /dev/sda3 luks-sda3
```

Now that `sda3` is encrypted, you can add it back to the ZFS pool as a mirror:

```bash
zpool attach rpool sdb3 /dev/mapper/luks-sda3
```

The system will begin to resilver and copy the data from `sdb3` onto the new `luks-sda3` disk.

Wait for the resilvering process to finish. This may take some time depending on the amount of data in your pool.

You can monitor the progress by checking:

```bash
zpool status
```

**NOTE:** Your system now will not boot anymore, you need to execute also the next step!

# Fix the boot procedure

## ZFS

This is the most basic step and the password will be requested at every boot.

Get the UUID's of the encrypted disks:

```bash
blkid | grep "/dev/[a-z]*3"
```

Modify the crypttab file `/etc/crypttab` by adding these lines. Replace disk UUID with the values fetched before.

```
luks-sda3 UUID="<disk UUID>" none luks,discard,initramfs
luks-sdb3 UUID="<disk UUID>" none luks,discard,initramfs
```

Modify the kernel cmdline file `/etc/kernel/cmdline` to this:

```
# For ZFS
echo 'cryptdevice=/dev/sda3:luks-sda3 cryptdevice=/dev/sdb3:luks-sdb3 root=ZFS=rpool/ROOT/pve-1 resume=/dev/mapper/luks-sda3 resume=/dev/mapper/luks-sdb3 boot=zfs' > /etc/kernel/cmdline

# For all other
echo 'cryptdevice=/dev/sda3:luks-sda3 cryptdevice=/dev/sdb3:luks-sdb3 root=/dev/mapper/luks-sda3' > /etc/kernel/cmdline
```

Add module to initramfs file `/etc/initramfs-tools/modules`:

```
echo 'dmcrypt' >> /etc/initramfs-tools/modules
```

Update initramfs with this command:

```bash
update-initramfs -u -k all
```

Update the Proxmox bootloader with this command:

```bash
proxmox-boot-tool refresh
```

## OPTIONAL: Add possibility to unlock via SSH (dropbear-initramfs)

Install the requirements:

```bash
apt install dropbear-initramfs
```

Add this line to the dropbear config `/etc/dropbear/initramfs/dropbear.conf`:

```
DROPBEAR_OPTIONS="-s -c cryptroot-unlock"
```

Add your `authorized_keys` to `/etc/dropbear/initramfs/authorized_keys` and fix the permissions:

```
chmod 600 /etc/dropbear/initramfs/authorized_keys
```

Modify the GRUB config `/etc/default/grub` to add IP configuration and edit this line to:

**Dynamic IP config**
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet ip=dhcp"
```

or

**Static IP config**
Check for the interface name with `ip link` and replace the values with your's.
`192.168.1.101`: Server IP
`192.168.1.254`: Gateway
`255.255.255.0`: Subnet mask
`1.1.1.1`: DNS server
`enp4s0`: Interface name
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet ip=192.168.1.101::192.168.1.254:255.255.255.0:1.1.1.1:enp4s0:none"
```

Update initramfs with this command:

```bash
update-initramfs -u -k all
```

Update the Proxmox bootloader with this command:

```bash
proxmox-boot-tool refresh
```

## OPTIONAL: Add possibility to unlock via SSH (dropbear-initramfs)

### Sources

- https://forum.proxmox.com/threads/adding-full-disk-encryption-to-proxmox.137051/#-add-ssh-unlock
- https://www.cyberciti.biz/security/how-to-unlock-luks-using-dropbear-ssh-keys-remotely-in-linux/

## OPTIONAL: Add automated unlock via TPM

Verify if your system has a TPM chip:

```bash
ls /dev/tpm*
```

`/dev/tpm0` is a TPM1.2 or TPM2.0

`/dev/tpmrm0` is a TPM2.0

Verify if it was loaded at last boot:

```bash
dmesg | grep -i tpm | grep -i reserving
```

Install the necessary TPM2 tools to interact with the TPM module and Clevis with its related packages for LUKS and TPM2 integration at boot.

```bash
apt install -y tpm2-tools clevis clevis-luks clevis-tpm2 clevis-initramfs

# clear the TPM in case you made some tests before
# tpm2_clear
```

Bind the LUKS partition to TPM2 using Clevis.

```bash
clevis luks bind -d /dev/sda3 tpm2 '{"pcr_bank":"sha256", "pcr_ids":"1,7"}'

clevis luks bind -d /dev/sdb3 tpm2 '{"pcr_bank":"sha256", "pcr_ids":"1,7"}'
```

Test if the unlock works:

```bash
clevis luks unlock -d /dev/sda3
clevis luks unlock -d /dev/sdb3
```

Update initramfs with this command:

```bash
update-initramfs -u -k all
```

Update the Proxmox bootloader with this command:

```bash
proxmox-boot-tool refresh
```

### Sources

- https://silvenga.com/posts/tpm-luks-unlock/
- https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005


## OPTIONAL: Add automated unlock via remote server (MandOS)

- https://www.recompile.se/mandos
