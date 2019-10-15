---
title: "Setting up a Linux virtual machine for the fMRI analysis pipeline"
permalink: /setup_vm_for_fmri/
date: 2019-10-11
share: true
header:
  image: "assets/images/neuro/fmri_header-2.png"
  teaser: "assets/images/neuro/fmri_header-2.png"
toc: true
toc_sticky: true
---

This is a guide to installing a virtual machine (VM) on Windows that will run the Linux operating system with the required software for the FreeSurfer/FSFAST fMRI analysis pipeline. The virtual machine part is detailed simply because that was required for me as we used Windows machines in our lab and a dedicated separate machine was too inconvenient. If you have a Linux machine you can ignore the parts about setting up the virtual environment and only consult the software installation section.


### Setup virtual machine
1. Install a VM virtualization software. I used the Oracle VM VirtualBox: https://www.virtualbox.org/wiki/Downloads. The instructions here will refer to this software.
2. Select or download a Linux distribution. I used the CentOS distribution (http://www.centos.org/download) because it was the only one on which I managed to install the BioImage Suite software (not part of this pipeline; used for marking intracranial electrode locations on a 3D brain as part of my ECoG pipeline).
3. Create the VM:
  * In Virtual Box, create a new Linux virtual machine:
  * Give it a name, and select "Linux" and "Other Linux" (assuming you're using a distribution you downloaded; 32 or 64 bit depending on your system).
  * Give it at least 1024MB RAM (I gave it 4MB out of my available 8GB).
  * Create a virtual hard drive (VDI) - you need to give it enough memory space for all the necessary software packages (it is complicated to add additional space once the machine is created) and data. This includes more or less 20GB for software (counted as everything but the /home directory on my machine), plus the size of all shared folders, plus the FreeSurfer subjects folder, which takes up about 1.3GB for every analyzed subject. Before you do this, make sure the virtual disk is stored where you can spare the space. You can set this at File->Preferences->General->Default machine folder.
    * Right-click -> settings:
    *	General- > Advanced -> set "shared clipboard" and "drag'n'drop" to "bidirectional.
    *	Display -> 64MB Video RAM (with 32MB, the brain images did not display).
    *	Display -> check Enable 3D acceleration
    *	Storage -> Under Controller: IDE, select the CD ROM icon, then click the small CD ROM icon on the right and find the downloaded ISO file.
    *	Shared Folders -> Add any needed folders (select "auto-mount" and "make permanent" - not sure if this is necessary)  
  *	Start the virtual machine and begin the installation. If at any time the disk is "ejected", it can be re-inserted from the menu bar; Devices->CD/DVD Devices.  
  *	You can skip the "media test".  
  *	Select a root password and make sure you do not forget it. You can use "humancognitive".  
  *	Select "Use All Space"  
  *	Selecting an installation configuration: "Desktop" is fine. Once the installation is done, restart and finish the setup. Select a user name and password.  
  *	You don't need to enable "kdump". When asked whether to reboot click "Yes".  
  * Until you complete the "Install Virtual Box Guest Additions" section (see below), you will not be able to copy/paste between the host and the virtual machine, or increase the virtual machine's screen resolution.  
4. Once the virtual machine is running:  
  *	To enable internet access: Click the network icon in the top bar and select an available network connection (this is necessary for the upcoming updates and downloads).  
  * Enable "super-user" for your user:   
    Open a terminal and enter:  
    <span style="font-family:monospace;"> su - </span>  
    <<enter root password in prompt>>  
    <span style="font-family:monospace;"> echo '<<user_name>> ALL=(ALL) ALL' >> /etc/sudoers </span>  
    Where <<user_name>> is your user name  
    Type <span style="font-family:monospace;"> 'exit' </span> to switch out of the root user or close the terminal.  
  *	Update system  
    Open a terminal and type:  
    <span style="font-family:monospace;">sudo yum update</span>  
    (this will take a while)  
    *	Install Virtual Box Guest Additions  
    Open a terminal and enter:  
    <span style="font-family:monospace;">sudo yum install gcc</span>  
    And:  
    <span style="font-family:monospace;">sudo yum groupinstall "Development Tools" </span>  
    reboot  
    After booting up (remember to activate the network connection again…) -  
    On the top menu, click Devices -> Insert Guest Additions CD Image, and run the installation.  
    Right-click on the desktop icon for the CD Drive and select 'Eject'.  
    Power off and restart the virtual machine.  
    * Give access permissions to the shared folders  
    Create a new folder for each shared folder where you want it to be, for example ~/shared_folders/<shared_folder_name> (~ = /home/[user]). Use either the file browser or from the terminal:   <br>
    <span style="font-family:monospace;">  mkdir ~/shared_folders/ </span>   <br>
    <span style="font-family:monospace;">  mkdir ~/shared_folders/[shared_folder_name] </span>   <br>
    Now enter in a terminal:  <br>
    <span style="font-family:monospace;">sudo mount -t vboxsf -o uid=1000,gid=1000,rw [shared_folder_id] [target_drive_location] </span>   <br>
    Where [shared_folder_id] is the name you gave the folder when you defined it in the Virtual Box program, and [target_drive_location] is the path of the folder you just created.   <br>
    You need to do this for every shared folder separately. If you don't remember the ID's you gave to the shared folders, look at Settings->Shared Folders in VirtualBox again or look at the contents of /media (but disregard the "sf_" prefix).  <br>
    You can subsequently add a shortcut to these folders from other locations, using:  <br>
    <span style="font-family:monospace;">ln -s [shared folder location] [shortcut location] </span>  <br>
    NOTE: Unfortunately this procedure is not preserved after restarting the machine (the mounting), so it's necessary to do this every time. To make this less inconvenient, my solution was to add a desktop shortcut to run this command. Here is how:  <br>
    Create a text file somewhere and give it an appropriate name, for example: ~/Documents/scripts/mount_shared_folders  <br>
    Open this file and enter the following lines:  <br>
    <span style="font-family:monospace;"> #!/bin/sh </span>  <br>
    <span style="font-family:monospace;"> [mount command] </span> <br>
    Where [mount command] is the same as above (sudo mount -t … ).  
    Now right-click on the desktop and select "Create Launcher". For Type select "Application in Terminal", for Name give it a name (like "mount shared folders"), and for Command enter:  <br>
    <span style="font-family:monospace;"> sh [path] </span>  <br>
    Where [path] is the full path of the script you created. When you run this launcher, it will open a terminal and run the script, asking you for the root password and activating the mount.  


### Installing required software
* **FreeSurfer**: Great tool that segments the brain and converts it to a cortical surface (which can be registered to a standard surface).
Follow download, installation, registration and configuration instructions on [http://freesurfer.net](http://freesurfer.net). It takes a few steps but it's quite straightforward.
One additional step that I needed to make: by default, FreeSurfer is installed to /usr/local. This means you will need root permissions to edit the files there. This is a minor nuisance during installation, for example when you need to create the license.txt file (but getting over this difficulty is part of learning to use basic Linux, so deal with it). However it's more of a problem when you actually run FS since the Subjects folder, to which all results are written is by default under the same folder (/usr/local/freesurfer/subjects). What I did is to edit the SUBJECTS_DIR variable in /usr/local/freesurfer/SetUpFreeSurfer.sh from the default value to my home folder.
Another minor point - for some reason a lot of the files produced by the freesurfer analysis are marked as hidden files - if you can't find a file try showing hidden files.
FreeSurfer citation information: [https://surfer.nmr.mgh.harvard.edu/fswiki/FreeSurferMethodsCitation](https://surfer.nmr.mgh.harvard.edu/fswiki/FreeSurferMethodsCitation).
* **MRIcron**:  A small and very handy utility for displaying f/MRI/CT images. Does not have a "3D volume" display, but is useful for showing several overlaid images (for example when testing the results of white/gray matter segmentation or brain surface generation). It's also used for converting MRI images from DICOM to NIFTI format.
Download: [https://people.cas.sc.edu/rorden/mricron/install.html](https://people.cas.sc.edu/rorden/mricron/install.html)
* **Matlab**: You will need Matlab to run the analysis scripts. Add the FreeSurfer Matlab folder to your path: it's in <FreeSurfer Home Directory>/matlab. Note that using a virtual machine may cause difficulties with accessing your network's Matlab license server.
