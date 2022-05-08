PLEASE READ THIS FILE ENTIRELY AND CAREFULLY BECAUSE IT MAY BE HELPFUL TO LET THE TOOL WORK CORRECTLY
AND AVOID CORRUPTING YOUR DISK.

 --- INTRO ---

This is a tool made by gabrik, that allows the deletion of any given file, bypassing every security check.
How does this tool (not malware!) work?

The UI of this application is made in C#.
The UI does not directly send messages to the driver, instaed i made a c++ DLL
which makes it easier to use windows weird APIs.

The C++ DLL also exports the "sendDeleteFile" function to send the real IOCTL

The driver is made in C, and it is a driver that runs in kernel-mode
This means:
 1 - the driver runs in the same address space of ntoskrnl.exe, aka System, aka pid 4
 2 - the driver can access ANY address of the entire RAM, or can obtain access to it
 
Windows defender often detects kdu.exe as a malware, if it does, you can turn off windows defender for a minute
because trust me, it is not a malware



 --- HOW TO USE ---
 
Run FileDeleterUI as administrator.
Click "Driver Setup"
Click "Install driver" and select FileDeleter.sys, from the same directory of FileDeleterUI
Click "Start driver"
Close the window of Driver Setup
Have fun

 
 
 --- WARNINGS ---
 
This tool was written for Windows 10 but it might also work for Windows 8 and Windows 11
It never works on Windows 7 tho

I tested this tool A LOT both ON MY REAL PC and on a VM, and nothing bad happened.
But it is NOT FULLY TESTED, meaning that I don't know if this tool might cause damage to the disk on unknown circumstances.
Gabrik is not responsible if you corrupt your disk by using this tool. This shouldn't happen anyway.
If you get a BSOD when trying to delete files, you better remove this from your computer and never install it again

This tool can be useful for removing malwares.
This tool can be fun because you can empty System32 while windows is running (in a VM please!)
This tool is not a malware
This tool seriously can delete any file, so do not play with it! It can even uninstall kaspersky if you select avp.exe
There are some special files which can't be deleted: swapfile.sys, pagefile.sys, system32\config\... and ntuser.dat

I am aware that the feature "Delete Directory" does not work as one would expect.
It deletes every file recursively under the directory, but does not delete the directory itself.
This is because i haven't yet figured out how to delete a directory from kernel-mode

WHAT IM GONNA SAY IS IMPORTANT AND READ IT CAREFULLY OR YOU WILL GET A BSOD
If FileDeleterUI hangs upon clicking "Start Driver" you can do this manually.
Start a cmd.exe as administrator
Enter the directory of the tool with "cd"

Do the following commands:
  kdu.exe -dse 0
  sc start FileDeleter
  kdu.exe -dse OLDVALUE
  
Where the OLDVALUE is the value printed in the command of disabling the DSE, converted to decimal
Example, if kdu.exe -dse 0 says:
  ...
  DSE Flags (...) value: 4006, new value to be written ...
  ...
  
Then in the third command you must type 16390 (0x4006 converted to decimal)



 --- HOW THE DRIVER WORKS ---

The driver receives an IOCTL call from the FileDeleterMgmt, (the DeviceIoControl function)
and parses it.
In the IOCTL message there is a struct containing the path of the file to delete and the NTSTATUS
which the driver must fill to let FileDeleterUI know if the deletion was successful.

How does the driver delete files? 

After parsing the path, it uses the function IoCreateFileEx, and not ZwCreateFile, in order to access some more options which
increases chances of opening the file without encountering a STATUS_ACCESS_DENIED or any other error.
After obtaining the handle, i do ObReferenceObjectByHandle to acccess the struct of settings of the handle.
We then have a FILE_OBJECT which contains the access, path, and other stuff of the handle.
After obtaining this struct,
  1 - we put the Busy field to FALSE
  2 - we put the DeletePending field to FALSE
  3 - we put the granted access to all access
  4 - we destroy the sections cache
  
We now have a FILE_OBJECT, fully unlocked, to perform low level operations on the filesystem driver.
The last step for deleting a file is sending a message to the filesystem driver, that the file must be deleted
once all handles are closed.
We send an IRP_MJ_SET_INFORMATION request packet to the filesystem, with the some special settings such that:
  1 - The file will be deleted forcibly (we put the SL_FORCE_DIRECT_WRITE flag in the stack of the request packet)
  2 - The file will be deleted even if there are open handles (FILE_DISPOSITION_POSIX_SEMANTICS flag in the request packet)
  3 - The file will be deleted even if it is read only (FILE_DISPOSITION_IGNORE_READONLY_ATTRIBUTE flag in the request packet)
  4 - The AccessMode is KernelMode, meaning we bypass all check regarding the access control
  5 - We bypass any hook to ZwSetInformationFile or ZwDeleteFile because we are directly sending a request to the filesystem driver
Once this request is processes, our file is marked to be deleted on close
We call ObCloseHandle to close the file forcibly, and... the file will be annihilated instantly