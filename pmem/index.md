# Persistent memory

Resources for working with persistent memory (PMem) on CloudLab hosts and the Hustle server.

## Introductory material

* [Persistent Memory Documentation](https://docs.pmem.io/persistent-memory)
* [Persistent Memory Development Kit (PMDK)](https://pmem.io/pmdk/)

## Provisioning PMem

This section describes how to provision a host with PMem. If you are simply emulating PMem in DRAM, you can skip to the next section.

0. If the device is already provisioned in a different mode, clear existing namespaces and goals.
    ```bash
    ndctl destroy-namespace -f all
    ipmctl delete -goal
    systemctl reboot
    ```
1. Create a new memory allocation goal. To configure all the PMem capacity in Memory Mode, run the following command.

    ```bash
    sudo ipmctl create -goal MemoryMode=100
    ```
    To configure all the PMem capacity as App Direct, run the following command.
    ```bash
    sudo ipmctl create -goal PersistentMemoryType=AppDirect
    ```
2. Reboot the host.

    ```bash
    sudo reboot
    ```
3. Verify that the provisioning was successful.

    ```bash
    sudo ipmctl show -memoryresources
    ```
4. Create new namespaces if necessary.

    First list existing namespaces and regions.
    ```bash
    sudo ndctl list -Ni
    sudo ndctl list --regions
    ```
    To create a namespace on region1, run the following command.
    ```bash
    sudo ndctl create-namespace -r region1 --mode fsdax -M dev
    ```
    The above command creates a namespace in region1 in the fsdax mode, with page metadata stored on the device.

For more information, refer to [IPMCTL User Guide](https://docs.pmem.io/ipmctl-user-guide/) and the [NDCTL User Guide](https://docs.pmem.io/ndctl-user-guide/).

## Emulating PMem

This section describes how to emulate PMem on a CloudLab hosts (or similar Linux environment). If you have a host that has been provisioned with real PMem, you can skip to the next section.

1. Determine the usable memory space by querying the e820 table.

    ```bash
    dmesg | grep BIOS-e820
    ```
    The output of this command will be similar to the following.
    ```
    [    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009d7ff] usable
    [    0.000000] BIOS-e820: [mem 0x000000000009d800-0x000000000009ffff] reserved
    [    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
    [    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000006a01afff] usable
    [    0.000000] BIOS-e820: [mem 0x000000006a01b000-0x000000006c90dfff] reserved
    [    0.000000] BIOS-e820: [mem 0x000000006c90e000-0x000000006ca7bfff] usable
    [    0.000000] BIOS-e820: [mem 0x000000006ca7c000-0x000000006d4cafff] ACPI NVS
    [    0.000000] BIOS-e820: [mem 0x000000006d4cb000-0x000000006f30bfff] reserved
    [    0.000000] BIOS-e820: [mem 0x000000006f30c000-0x000000006f7fffff] usable
    [    0.000000] BIOS-e820: [mem 0x000000006f800000-0x000000008fffffff] reserved
    [    0.000000] BIOS-e820: [mem 0x00000000fd000000-0x00000000fe7fffff] reserved
    [    0.000000] BIOS-e820: [mem 0x00000000fed20000-0x00000000fed44fff] reserved
    [    0.000000] BIOS-e820: [mem 0x00000000ff000000-0x00000000ffffffff] reserved
    [    0.000000] BIOS-e820: [mem 0x0000000100000000-0x000000303fffffff] usable
    ```
    In this example, there is usable memory between 4 GiB (0x100000000) and ~193 GiB (0x303fffffff).
2. Create a new memory mapping by modifying the GRUB entry. Ensure that the new mapping is in the usable space. The following example adds a new mapping of 16 GiB starting at 4 GiB.
    
    ```bash
    sudo vim /etc/default/grub
    ```
    Add or edit the `GRUB_CMDLINE_LINUX` to include the mapping.
    ```bash
    GRUB_CMDLINE_LINUX="memmap=16G!4G"
    ```
    There may be multiple redefinitions of `GRUB_CMDLINE_LINUX` in the file. You must edit the last one in order for the changes to take effect. To edit an existing `GRUB_CMDLINE_LINUX` entry, add the new option with a space separator. For example, the updated entry may look like the following.
    ```bash
    GRUB_CMDLINE_LINUX="console=ttyS0,115200 memmap=16G!4G"
    ```
3. Update GRUB and reboot the machine.

    ```bash
    sudo update-grub2
    sudo reboot
    ```
4. After rebooting, a new `/dev/pmem{N}` device should exist, one for each memmap region specified in the GRUB config. These can be shown using `ls /dev/pmem*`. You can also verify that a new user-defined e820 table entry shows that the range is now persistent.
    ```bash
    dmesg | grep user:
    ```
    The output of this command will be similar to the following.
    ```
    [    0.000000] user: [mem 0x0000000000000000-0x000000000009d7ff] usable
    [    0.000000] user: [mem 0x000000000009d800-0x000000000009ffff] reserved
    [    0.000000] user: [mem 0x00000000000e0000-0x00000000000fffff] reserved
    [    0.000000] user: [mem 0x0000000000100000-0x000000006a01afff] usable
    [    0.000000] user: [mem 0x000000006a01b000-0x000000006c90dfff] reserved
    [    0.000000] user: [mem 0x000000006c90e000-0x000000006ca7bfff] usable
    [    0.000000] user: [mem 0x000000006ca7c000-0x000000006d4cafff] ACPI NVS
    [    0.000000] user: [mem 0x000000006d4cb000-0x000000006f30bfff] reserved
    [    0.000000] user: [mem 0x000000006f30c000-0x000000006f7fffff] usable
    [    0.000000] user: [mem 0x000000006f800000-0x000000008fffffff] reserved
    [    0.000000] user: [mem 0x00000000fd000000-0x00000000fe7fffff] reserved
    [    0.000000] user: [mem 0x00000000fed20000-0x00000000fed44fff] reserved
    [    0.000000] user: [mem 0x00000000ff000000-0x00000000ffffffff] reserved
    [    0.000000] user: [mem 0x0000000100000000-0x00000004ffffffff] persistent (type 12)
    [    0.000000] user: [mem 0x0000000500000000-0x000000303fffffff] usable
    ```

For more information, refer to [Linux Environments - Persistent Memory Documentation](https://docs.pmem.io/persistent-memory/getting-started-guide/creating-development-environments/linux-environments).

## Creating and mounting a filesystem

This section describes how to create and mount a new filesystem on a `/dev/pmem{N}` device.

1. Create and mount a new filesystem. The following shows how to create and mount an EXT4 filesystem for the `/dev/pmem0` device.

    ```bash
    sudo mkfs.ext4 /dev/pmem0
    sudo mkdir /mnt/pmem0
    sudo mount -o dax /dev/pmem0 /mnt/pmem0
    ```
2. Verify that the new filesystem has been mounted and the `dax` flag has been set.

     ```bash
     sudo mount -v | grep /mnt/pmem0
     ```
     This output of this command will be similar to the following.
     ```
     /dev/pmem0 on /mnt/pmem0 type ext4 (rw,relatime,dax)
     ```
You can now write code that uses the `/mnt/pmem{0}` filesystem as a PMem store. The [Persistent Memory Development Kit (PMDK)](https://pmem.io/pmdk/) provides a collection of libraries to program PMem in DAX mode and also has [coding examples](https://github.com/pmem/pmdk/tree/master/src/examples) to help get started.

