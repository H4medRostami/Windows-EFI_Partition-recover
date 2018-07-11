How to Restore Deleted EFI Boot Partition in Windows 
   
In this article we’ll show how to manually repair accidentally deleted Windows boot partition on UEFI system. Initially, the article described my experience with restoring the boot EFI partition on Windows 7, but the article is also relevant for modern Microsoft operating systems (from Windows 7 to Windows 10). This article often helped me after accidentally formatting or deleting the EFI partition on Windows 10. I’ll show you an easy way to manually rebuild the boot EFI and MSR partitions on Windows.

Suppose that the EFI boot partition on your UEFI (non-BIOS) system was accidentally deleted or formatted (for example, when you tried to remove OEM recovery partition), and as a result Windows 10 (8.1 / 7) doesn’t boot constantly offering to select the boot device (Reboot and select proper boot device or insert boot media in selected). Is it possible to restore the Windows performance when removing the Boot Manager partition without reinstalling the system?
Warning. This guide implements working with disk partitions and is not recommended for beginners. If you interpret the commands wrongly, you can accidentally delete all data on your hard disk. It is also strongly recommended to back up important data on a separate media.

Contents:

    GPT Partition Structure
    How to Manually Create EFI and MSR partition on GPT-based HDD
    Repairing EFI bootloader and Windows BCD

GPT Partition Structure

Consider how the partition table on the bootable hard disk with GUID partition table (GPT) on the UEFI system should look like. You should have at least the following partitions:

    EFI System Partition or ESP (Extensible Firmware Interface) — 100 MB (partition type — EFI).
    Microsoft Reserved partition — 128 MB (partition type — MSR).
    Primary Windows partition (the partition containing Windows).

Default efi partitoions

This is the minimum configuration. These partitions are created by the Windows Installer when you install the system on an unformatted disk. Computer manufacturers or users can create their own  partitions containing, for example, Windows Recovery Environment (Windows RE) in the winre.wim file, a partition with the system image backup provided by the manufacturer (allows to roll back to the default state of the system), user partitions, etc.

The EFI partition with the FAT32 file system is a mandatory partition on GPT disks on UEFI systems. This partition, similar to the System Reserved partition on drives with MFT partition table, stores the boot configuration store (BCD) and a number of files needed for Windows boot. When the computer boots, the UEFI environment loads the bootloader (EFI\Microsoft\Boot\bootmgfw.efi) from the EFI (ESP) partition and transfers control to it. If this partition is deleted, Windows can’t boot correctly.

The MSR partition on the GPT disk is used to simplify partition management and is used for service operations (for example, when converting a disk from simple to dynamic). This is a backup partition that does not have a partition code assigned. This partition can’t store user data. In Windows 10, the size of the MSR partition is only 16 MB (in Windows 8.1 the size of the MSR partition is 128 MB), the file system is NTFS.
Tip. To install Windows on computers with UEFI you will need an original DVD or a specially prepared bootable USB flash drive with Windows 7, Windows 8 / Windows Server 2012 or Windows 10 / Server 2016.
How to Manually Create EFI and MSR partition on GPT-based HDD

Since the system doesn’t boot correctly, we’ll need Windows installation disk with Windows 10 (Win 8 or 7) or any other boot disk. Boot from the installation media and on the first installation screen press Shift+F10. The command prompt window opens.

Shift+F10 - command prompta

Run the disk and partition management utility:

      Diskpart

Display the list of hard disks in the system (in our example, there is only one disk, disk 0. The asterisk in the GPT column means that it uses the GUID partition table).

      list disk

Select this disk:

      Select disk 0

Display the list of partitions on this disk:

      List partition

In our example, only two partitions are left in the system:

    MSR partition — 128 MB;
    Windows system partition — 9 GB.

As you can see, the EFI partition is missing (it has been deleted).

diskpart list partition

Our task is to remove the remaining MSR partition so that we have at least 228 MB of unallocated space on the drive (for MSR and EFI partitions). You can remove this partition using the graphical Gparted or directly from the command prompt (we’ll choose the last variant).
Important! Please, be extremely attentive here and do not accidentally delete Windows partition or partitions containing user data (if there are any).

Select the partition to remove:

      Select partition 1

      And delete it

      Delete partition override

Make sure that there is only Windows partition left:

      List partition

      delete msr partition

Now you can re-create EFI and MSR partitions manually. To do it, run these commands in diskpart context one by one.

Select the disk:

      select disk 0
 
      create partition efi size=100

Make sure that the 100 MB partition (an asterisk in front of the Partition 1) is selected:

      list partition
      select partition 1
      format quick fs=fat32 label="System"
      assign letter=G
      create partition msr size=128
      list partition
      list vol

In our case, disk letter C: is already assigned to our Windows partition. Otherwise, assign the drive letter to it as follows:

      select vol 1
      assign letter=C
      exit

recreate efi and msr partitions
Repairing EFI bootloader and Windows BCD

After you have created a minimal disk partition structure for the UEFI system, you can proceed to copy the EFI boot files to the disk and create a bootloader configuration file (BCD).

Copy the EFI environment files from the directory of the installed Windows 10:
   
      mkdir G:\EFI\Microsoft\Boot
 
      xcopy /s C:\Windows\Boot\EFI\*.* G:\EFI\Microsoft\Boot

copy efi files

Let’s re-create the Windows 10 / 7 bootloader configuration:

      g:
      cd EFI\Microsoft\Boot
      bcdedit /createstore BCD
      bcdedit /store BCD  /create {bootmgr} /d “Windows Boot Manager”
      bcdedit /store BCD /create /d “My Windows 10” /application osloader

You can replace the caption “My Windows 10” for any other.
Tip. If only EFI files were damaged on the EFI partition and the partition itself was not deleted, you can skip the process of recreating partitions using diskpart. Although in most cases it is enough to repair the EFI bootloader in Windows 10 / 8.1. You can manually recreate the BCD on an MBR+BIOS system using this article.

The command returns the GUID of the created record, in the next command put this GUID instead of {your_guid}.

 create bcd store

      bcdedit /store BCD /set {bootmgr} default {your_guid}
      bcdedit /store BCD /set {bootmgr} path \EFI\Microsoft\Boot\bootmgfw.efi
      bcdedit /store BCD /set {bootmgr} displayorder {default}

bcdedit bootmgr

The following commands are run in the {default} context:

      bcdedit /store BCD /set {default} device partition=c:
      bcdedit /store BCD /set {default} osdevice partition=c:
      bcdedit /store BCD /set {default} path \Windows\System32\winload.efi
      bcdedit /store BCD /set {default} systemroot \Windows
      exit

bcdedit winload efi

Restart your computer… In our case it didn’t boot from the first time. Try the following:

    Turn your PC off.
    Unplug your hard drive.
    Turn your PC on, wait till the boot error window appears and turn it off again.
    Plug your disk.

Then in our case (the test took place on the VMWare virtual machine with UEFI system) we had to add a new item to the boot menu by selecting EFI\Microsoft\Boot\bootmgrfw.efi on the EFI partition.

In some UEFI menus, by analogy, you need to change the priority of the boot partitions.

efi boot option FI\Microsoft\Boot\bootmgrfw.efi

After all these actions, your Windows should boot correctly.

setup is starting services

Tip. If it doesn’t work, it is recommended to make sure that only EFI partition has the boot flag. You can do it using GParted LiveCD.


woshub.com
