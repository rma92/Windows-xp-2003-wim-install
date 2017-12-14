# Building a fast WIM-based Windows Server 2003 Installer

Goal: Create a WIM based installer for Windows 2003 to decrease the
install time.
Assumptions:
* You have a volume license and key for the version of Windows you will be imaging.
* You have a copy of Windows Fundamentals for Legacy PCs, as we will be modifying the WinPE on this disk as the base OS for the installer.
* You have a virtual machine available. (I did this using VirtualBox, as it allows two CD drives which vastly simplifies the process).
* You have access to Windows 8, 8.1, or 10.  This is necessary to use DISM to capture the partially installed system in the VM.  You may elect to mount a VHD and use a host OS to perform this step, or boot an installer/WinPE system in the VM and capture the WIM inside the VM.
* MagicISO for modifying the Windows FLP disc.

If you plan to mount the virtual machine disk so you can use the host OS to 
create the WIM, and you are using VirtualBox, you should make sure you use a 
VHD type virtual disk.  The disk must also be on an NTFS volume, and the VHD
file itself uncompressed for the mounting to work properly.

It is beneficial to capture using this method, as it will be easy to copy the resulting WIM file out of the virtual machine.

##Installing a Windows 2000/XP/2003 system in a way that can be captured
We want to be able to capture a WIM of the second stage of Windows setup
(the graphical part), before hardware detection occurs.

We also want all of the files on the hard drive, so that a Windows 2003
CD is not required.

The instructions should be followed using a VM.

### Preparing the OS image - installing Setup.
* Boot into a WINPE system, partition and format the hard drive.

**NOTE**: if you are going to be booting WinPE off a CD, create a VM with
two CD drives.  Put a Windows PE CD (or the Windows Fundamentals for 
Legacy PCs installer CD) in the first (bootable) CD drive, and put the 
CD for the version of Windows you are trying to capture in the second
CD drive.

* Run WINNT32 from the Windows installer CD.  (in the i386/AMD64 folder)

* For "Installation Type", select _New Installation (Advanced)_ and click _Next_.
* Accept the license agreement, click _Next_.
* Enter product key where required, click _Next_.
* On the "Setup Options" screen, click _Advanced Options..._
* Check _Copy all installation files from the Setup CD_ and click _OK_.  This is very important - if you do not check this box, a Windows CD will be required during the second stage of setup, vastly reducing the usefulness of this project.  If you prefer to use a different name for the Windows folder, that can be changed as well.
* If it asks you, choose if you want to run Windows Update or not.
* Setup will copy the files.
* Setup will prompt you to reboot the computer.  Reboot the computer and continue into the text mode setup.

At this point the CD for the version of Windows being installed is no longer needed.

### Preparing the OS image - Text mode setup.
* Boot the computer into the Setup that has been installed on the hard
  drive.
* At the "Welcome to Setup" screen, press enter.
* At the partitioning screen, choose the partition onto which you installed
  Setup. This is probably C:, but could be something else if you have an
  unusual configuration.
* Setup will copy the files.
* Setup will prompt you to reboot the computer.  Turn the computer off.

The setup is now ready to proceed into the second (graphical) stage, so
it is time to capture it!

## Capturing the Setup Image
A WIM image will need to be captured of the now ready second stage of setup.

**NOTE**: ImageX can not be used as it will not capture the Windows.$bt and Windows.$ls folders, which are necessary for setup.  This step will be performed using DISM from Windows 8 or later instead.

### Capturing the setup image - using DISM on a Windows 8/8.1/10 setup disc
Windows Vista and later install discs contain a copy of WinPE, which can
be used to assist in various tasks.  In Windows 8/8.1/10 setup, pressing
Shift+F10 at any time can be used to open a command prompt.

* Boot the machine from the installer disc.
* At the "Welcome to setup screen", hit Shift+F10 to open a command prompt.
* Determine which drive letter contains the installer files.  An easy way to do this is to open notepad (type notepad and hit enter), and use the File > Open screen to check the drives in the machine.
* You will use dism to capture the image.  Use the following command line

    `dism /capture-image /imagefile:<imgfile>.wim /captureDir:<letter>:\ /Name:<a name>`

    `<imgfile>` is the name and path of the wim file you would like to create. This may also be a full path.  You can also choose to create the wim file on the drive where setup is installed.
    `<letter>`    is the drive letter where Setup has been placed.
    `<a name>`    A name for the image.  This is shown when you get the image info.  You may use spaces if you wrap the name in quotes.

    For example, I used
      dism /capture-image /imagefile:C:\win2003.wim /captureDir:C:\ /Name:"Windows Server 2003 Enterprise"

    This created a WIM file on the C: drive called "win2003.wim", with a title "Windows Server 2003 Enterprise", and containing the contents of the C: drive.


### Capturing the setup image - using Windows 8/8.1/10 PE
If you have a WinPE disc, boot the machine off of the WinPE disc.  
It will boot to a command prompt.  Check the drive letter that contains
setup, and follow the DISM instructions above.

### Using DISM on host Windows 8/8.1/10 (or Server 2012/2012 R2/2016)
* Mount the VHD file for the VM.  This can be done with a point and click
  interface through the disk management snap in.  It can be accessed by
  running "diskmgmt.msc".  Select one of the volumes, then in the _Action_
  menu, select _Attach VHD_.  You can then browse for the VHD you created.
  You may elect to mount the disk read-only, but it is not required.
    **NOTE**: This will only work with VHD files, not VDI or VMDK files.
* Use DISM to capture the WIM.  Use the same instructions as above, but
  make sure CaptureDir is the correct drive letter.  This can be seen in 
  the Disk Management window. 
    **NOTE**: DISM must be run from an Administrator command prompt.
* Unmount the VHD.  This can be done by right-clicking the VHD in the 
  list of disks (on the left-most column) and selecting _Eject_.

If you followed these instructions correctly, you should have a WIM of 
the second stage of Windows setup.  This can be applied to various
machines, and setup will begin from the second stage.

## Modifying the Windows Fundamentals for Legacy PCs installer
The Windows Fundamentals for Legacy PCs installer is actually based on
a WinPE from Windows Server 2003, and has a graphical setup.  When that
disc is booted, the following happens:
* Windows PE is booted off of the disc.
* It launches \Setup\SetupLauncher.exe.  SetupLauncher.exe prompts for
    several seconds to "press any key to open command prompt.".  If the
    command prompt is opened, setup will not begin, and the computer will
    reboot once the command prompt is closed.
* SetupLauncher.exe opens \Setup\Setup.exe, which performs the actual
    setup process.  Interestingly, Setup.exe is a .net 1.1 application,
    and the .net framework 1.1 is actually installed in this WinPE.
    Also interestingly, the setup program uses a WIM file, where the 
    different installable options are different images within the WIM.

Considerations to do a simple job:
* Replace SetupLauncher.exe with Cmd.exe, and add imagex.exe and the new
  WIM file we created.  This will allow for manually running diskpart,
  format, and imagex /apply to set up the image.  
* A benefit of using a Windows XP/2003 PE instead of a later version -
  format will format the drive with an appropriate NTLDR bootsector,
  simplifying installation.  If a Vista or later PE is used, it is 
  necessary to run bootsect.exe /nt52 on the newly formated drive.
* For some reason, I was unable to alter \i386\system32\winpeshl.ini in
  a satisfactory manner.  I may have made a mistake while doing so.
* Future: It would be nice to boot this PE from a WIM image, to reduce
  the total size of the CD image (and increase the speed of the PE booting
  if this will be run off an actual CD.)
* Future: Write a GUI tool that allows partitioning and formatting the
  drive(s), and applying the disk image.  With .net 1.1 already available,
  this could be developed against that platform with minimal hassle.
* Deleting files from an ISO does not work cleanly.  It is better to
  create a new ISO to minimize size and placement of the files.

### Creating a "base" PE ISO from the WinFLP setup ISO
Tools required:
* MagicISO

Because deleting files from ISO images can be messy, it is recommended
to create a new ISO with the files that are needed.  You will probably
also want to create a folder in which all of the files that are extracted
can be stored.
* Open the WinFLP iso in MagicISO.
* Save the bootable image file.  To do this, go to Tools > Save boot image.
* Save the boot image in your working folder. (Ideally, append ".bif" onto the end, as MagicISO does not do so automatically, and this will make things a bit simpler)
* Extract the contents of the ISO to a folder.  This can be done by selecting all the files in the ISO, right-clicking, and choosing Extract.  
    **NOTE**: Drag-n-drop out of MagicISO is supported, but may store the
    files in the temp folder during copying.  Using the extract option 
    will be quicker.

Now we can create a new ISO and start fresh.
* In MagicISO, select File > New > Bootable CD/DVD Image.
* Select _From bootable Image File_.  Browse for the file you extracted. Click _OK_.
* Open the folder you extracted the contents of the FLP CD to in either explorer, or the MagicISO file browser.
* Drag I386, Program Files, win51, win51is, win51is.sp1, and winbom.ini into the ISO.
* Right click an empty area of the ISO file list, choose _New Folder_.
* Name the folder "Setup".  
* Double-click the "Setup" folder to open it.
* Drag I386\System32\Cmd.exe (from the computer file system) into the Setup folder.  Right-click it, rename it "SetupLauncher.exe".

Save the ISO. (File > Save As).  Boot a VM from the ISO to make sure it works. It should boot to a command prompt.
