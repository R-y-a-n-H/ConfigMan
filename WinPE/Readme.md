# WinPE Driver Injection Methodology

As many of you may know who follow me on twitter [@SCCM_Ryan](https://twitter.com/SCCM_Ryan) the struggle has been real with WinPE drivers.

## Backstory:
My environment consists of older Lenovo PCs and newer Dells. I was busy and wanted to get a newer model Dell working so I "mistakenly" injected ALL WinPE drivers (storage anyways) from the [Dell WinPE driver pack.](https://www.dell.com/support/kbdoc/000107478/dell-command-deploy-winpe-driver-packs) I say "mistakenly" because in all honesty ever since I've gotten into using Configuration Manager the advice of "just inject Dell's WinPE Driver pack and you should be good to go" has been thrown around a lot. I did this, and then many of my older models of computer started having WinPE driver issues. This most likely happened because of newer Intel RST/VMD drivers needing to be injected in the proper order otherwise they conflict and cause issues. Johan has a quick blog post about this [HERE](https://www.deploymentresearch.com/back-to-basics-driver-installation-order-in-winpe-matters/) 

The RST (Rapid Storage Technology) drivers are used while SATA is in AHCI mode and the VMD (Volume Management Device) drivers are used while in RAID mode. Since the 12th Gen Intel Processors have been coming out Dell has defaulted their SATA storage to RAID instead of the AHCI mode they used to be set to. This has required me to either inject an Intel RST VMD Controller driver into WinPE or manually go into BIOS and set the SATA setting from RAID to AHCI. I don't like to have to do anything manually if I don't have to so getting the correct driver into WinPE is my prefered senario. My task sequence also for the most part sets the BIOS settings back to the factory defaults because I like to have a "known" state to combat other potential issues. If I have it set to defualt then very shortly into the task sequence it will lose its disk driver and fail anyways.

## New Method of WinPE Driver Injection
My biggest issue of having the broken boot WIM is that I didn't have any documention of what was injected before I took the shotgun approach of injecting EVERYTHING. To try and prevent this I started adding the driver category of "REQUIRED" on the WinPE driver. So, how do I figure out which driver is required.

### Real World Example: Dell Latitude 5531
I have a WinPE 10 x64 with NO drivers imported created using the Windows ADK version 10.1.22000.1. When I boot into WinPE on the Dell Latitude 5531 I have no network or disk access (in RAID mode).

**No Network**
```
 X:\sms\bin\x64>ipconfig

 Windows IP Configuration
 
 ```

**No Disk**

*Disk 0 is the USB flash drive I use for testing*

```
X:\sms\bin\x64>diskpart

 Microsoft DiskPart version 10.0.22000.1

 Copyright (C) Microsoft Corporation.
 On computer: MININT-GCPIJ2J

 DISKPART> list disk


 Disk ###  Status         Size     Free     Dyn  Gpt
 --------  -------------  -------  -------  ---  ---
 Disk 0    Online         7377 MB      0 B        * 
 ```

1. So, since the first thing I do when we get a new model is download the [Driver Pack](https://www.dell.com/support/kbdoc/en-us/000198974/latitude-5531-windows-10-driver-pack) to import into Configuation Manager I go and look to see what drivers are availible in it. I find my Driver Package, right-click, and then show members. I then sort by class and look for "net" and "SCSIAdapter".

2. For the network driver I ignore the wireless drivers and look for an ethernet driver. In this case the "Intel(R) Ethernet Connection I217-LM" and version "12.19.1.37".

3. For the disk I look for VMD driver which is "Intel RST VMD Controller 9A0B" and version "19.2.0.1003".

4. I then go to the Dell WinPE 10 x64 (A26) drive pack I imported and the look at members. There I find the drivers with the same name and version.

5. I right-click each of the 2 drivers and go to "Properties" and then copy the Driver Source file path.

6. I copy the contents folder for each driver to my bootable WinPE flash drive. I usually name the folder the driver name + version like "Intel RST VMD Controller 9A0B - 19.2.0.1003"

7. I then boot the Dell Latitude 5531 with the USB stick into WinPE and then manually load the drivers

```
 X:\sms\bin\x64>drvload.exe "c:\Intel(R) Ethernet Connection I217-LM -12.19.1.37\e1d68x64.inf"
 DrvLoad: Successfully loaded c:\Intel(R) Ethernet Connection I217-LM -12.19.1.37\e1d68x64.inf.

 X:\sms\bin\x64>ipconfig

 Windows IP Configuration

 Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : lab.techmin.com
   Link-local IPv6 Address . . . . . : fe80::ecdf:190c:64d4:8139%3
   IPv4 Address. . . . . . . . . . . : 192.168.1.123
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

 X:\sms\bin\x64>drvload.exe "Intel RST VMD Controller 9A0B - 19.2.0.1003\iaStorVD.inf"
 DrvLoad: Successfully loaded Intel RST VMD Controller 9A0B - 19.2.0.1003\iaStorVD.inf.

 X:\sms\bin\x64>diskpart

 Microsoft DiskPart version 10.0.22000.1

 Copyright (C) Microsoft Corporation.
 On computer: MININT-GCPIJ2J

DISKPART> list disk

 Disk ###  Status         Size     Free     Dyn  Gpt
 \--------  -------------  -------  -------  ---  ---\
 Disk 0    Online         7377 MB      0 B        *
 Disk 1    Online          476 GB      0 B        * 
```

Since both drivers are working for this model I will then want to get these added to WinPE.

I've started adding a category to the drivers of "Required" so that when I go to import WinPE Drivers I can just search for them. I may also add the Model that originally required it to be added, and even possibly a date field... 

![Required WinPE Drivers]("C:\Users\hansardr\OneDrive - Verplank Enterprises, Inc\hansardr Profile\Documents\WinPE Driver Testing\REQUIRED.jpg")

