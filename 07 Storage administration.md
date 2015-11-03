# Overview

This section provides some reference and describes some tricks to management the storage devices in a Rocks cluster called biocluster. These include the hard drives in each compute node, the hard drives in the head node (frontend), and the hard drives in the storage node (NAS appliance).

# Hard drive selection

If you are building a NAS for heavy I/O work (since each compute node will need to access this NAS, and all user home directory is stored here), make sure to use enterprise grade drives, such as Seagate Constellation or WD Re drives, and make sure to use SAS drives rather than SATA drives (the price difference is very minor today). If you are building a NAS for occasional data back up and storage, then a drive designed for NAS can be used, such as WD Red drives. Just do not buy any drive for desktop applications, such as Seagate Burracuda.

For compute node, it does not really matter that much, so a cheap (but large-capacity) drive is probably preferred when budget is a concern; otherwise, use enterprise drive. The approximate capacity really depends on your application: for genomics application that often requires processing 100G of raw sequence data and generate even 500G temporary files, it makes the most sense to use $TMP in a compute node to store these temporary files (rather than using the NAS appliance itself), so a large drive (such as 3T) in compute node makes more sense. 

For head node, make sure to use enterprise drives, and use a RAID 1. Size does not matter, in fact 500G is more than enough.


# NAS storage administration

The NAS is a basically a compute box with many hard drives. Typical set up are 36 drives within a 4U enclosure, or 18 drives within a 2U enclosure. These drives are connected to a SAS RAID controller and a SAS expander (typically when you buy the computer box, these components are already there so you do not need to do anything). LSI is the most popular RAID controller, but some manufacturers such as Dell has their own controllers. My preference is to use LSI, even if I have access to a Dell machine with H700 controllers, I still unplug it and install a new LSI card. The reason is that manufacturers such as Dell/IBM/HP make things difficult for users to customize so that users can only buy from them with high premium; for example, the SAS cable for H700 is not compatible with a real SAS cable that everybody uses, even though they have identical appearance and have identical name and identical interface. It drived me crazy once.

The instructions below refer to LSI controllers.

MegaRaid Hardware information

System status information
http://mycusthelp.info/LSI/_cs/AnswerDetail.aspx?sSessionID=&aid=8264
use “./lsiget.sh” to generate the tar.gz file and send to SM.

# MegaCli
MegaCli installation
Download the program from http://www.lsi.com/support/Pages/download-results.aspx?keyword=megacli
By default, MegaCli is installed at /opt/MegaRAID/MegaCli/MegaCli64

MegaCli command line reference
Use the following command to print complete help message:
/opt/MegaRAID/MegaCli/MegaCli64 -h

The following link has more detailed description:
http://mycusthelp.info/LSI/_cs/AnswerPreview.aspx?sSessionID=&inc=8040

MegaCli cheat sheet
http://erikimh.com/megacli-cheatsheet/
http://genuxation.com/wiki/index.php/MegaCLI
https://supportforums.cisco.com/docs/DOC-16309
lsi.sh script https://calomel.org/megacli_lsi_commands.html 

Essential commands to use in a new system
[root@nas-0-1 ~]$ /opt/MegaRAID/MegaCli/MegaCli64  -AdpAllInfo -aALL | fgrep Failed
Security Key Failed              : No
  Failed Disks    : 2 
Deny Force Failed                       : No

This immediately tells us that two LD has failed!!!
To solve this, I cleared foreign configuration, and then manually stopped an ongoing ‘Rebuild’, then assigned the two ‘bad drives’ to ‘global hot spare’, then the rebuild starts automatically. I stopped an ongoing ‘Rebuild’ because it is rebuilt from a different enclosure.




show system informatoin:
MegaCli -AdpAllInfo -aALL
MegaCli -CfgDsply -aALL
MegaCli -adpeventlog -getevents -f lsi-events.log -a0 -nolog


Show adapter information:
/opt/MegaRAID/MegaCli/MegaCli64 -AdpAllInfo -a0

show enclosure information:
/opt/MegaRAID/MegaCli/MegaCli64 -EncInfo -aALL

show virtual dive information:
/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aALL

show physical drive information
/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL

/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL | fgrep S.M.A.R.T
#Identify discs that may fail in the future

show BBU infomration:
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -aALL

Manual relearn battery
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -BbuLearn -a0 

use WB(write-back) even if BBU is bad
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp CachedBadBBU -Lall -aAll

Reverse above procedure
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp NoCachedBadBBU -Lall -aAll

Set virtual drive to writeback
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -LALL -aALL

Set virtual drive to readahead (ReadAdaptive)
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp RA -LALL -aALL

Identify all hot spares
/opt/MegaRAID/MegaCli/MegaCli64 -PDlist -a0 | fgrep Hotspare -B 10

set global hot spare
/opt/MegaRAID/MegaCli/MegaCli64 -PDHSP -Set -PhysDrv [E:S] -aN


Rebuild a drive

Perform diagnosis on adapter
/opt/MegaRAID/MegaCli/MegaCli64 -AdpDiag -a0

Start, stop, suspend, resume, and show the progress of a consistency check operation
/opt/MegaRAID/MegaCli/MegaCli64 -LDCC -Start|-Abort|-Suspend|-Resume|-ShowProg|-ProgDsply -Lx|-L0,1,2|-LALL -aN|-a0,1,2|-aALL

MegaRaid Storage Manager

Install: 
Download the package, then “./install.csh -a” for complete installation
However, every time I install it, I got the following message. It works for a while but after a while, the nas-0-0 can no longer be accessed from biocluster.
Can not find snmptrap in /usr/bin
Can not continue installation of LSI SAS-IR Agent
Please install the net-snmp agent
warning: %post(sas_ir_snmp-13.08-0401.x86_64) scriptlet failed, exit status 1


I solved the problem by installing net-snmp-utils (first search for packages that provides snmptrap, then install it):
[root@nas-0-0 disk]# yum provides */snmptrap  
[root@nas-0-0 disk]# yum install net-snmp-utils

Uninstall: 
/usr/local/MegaRAID\ Storage\ Manager/uninstaller.sh
or
/usr/local/MegaRAID\ Storage\ Manager/__uninstaller.sh if the former does not work (such as in nas-0-1)



# XFS volume repair

When the system shows IO error, one cannot read/write from the /export directory any more. Umounting and re-mounting shows "mount: strcuture needs cleaning" message. Restarting the computer did not help either. Therefore, we attempated xfs_repair to fix this problem.

```
[root@nas-0-1 ~]# xfs_repair /dev/mapper/nas--0--1-export
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ERROR: The filesystem has valuable metadata changes in a log which needs to
be replayed.  Mount the filesystem to replay the log, and unmount it before
re-running xfs_repair.  If you are unable to mount the filesystem, then use
the -L option to destroy the log and attempt a repair.
Note that destroying the log may cause corruption -- please attempt a mount
of the filesystem before doing this.
[root@nas-0-1 ~]# mount /export/
mount: Structure needs cleaning
[root@nas-0-1 ~]# mount /export 
mount: Structure needs cleaning
[root@nas-0-1 ~]# xfs_repair -L /dev/mapper/nas--0--1-export
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ALERT: The filesystem has valuable metadata changes in a log which is being
destroyed because the -L option was used.
        - scan filesystem freespace and inode maps...
agi unlinked bucket 6 is 26403654 in ag 0 (inode=26403654)
sb_icount 17162816, counted 34859264
sb_ifree 125, counted 1683
sb_fdblocks 69126412932, counted 60959141713
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
data fork in ino 2425892 claims free block 6063805942
data fork in ino 2425892 claims free block 6063805943
```

The XFS FAQ provides excellent guide: http://xfs.org/index.php/XFS_FAQ

# Clear RAID information from hard drive

If you insert an old hard drive into a compute node, yet the hard drive may previously be part of a RAID array, Rocks will refuse to install operating system into the drive. There are no commands in Rocks that can override this decision.

If you encounter this situation, that is, during Rocks installation in a compute node, a screen shows up telling you that the drive has RAID information, you can use the following procedure to wipe out RAID information from the hard drive:

First, just press Ctrl+Alt+F5 (or maybe F6 or F4) to switch to a text terminal from GUI terminal, then do

`blockdev --getsz /dev/sda` to get the size of drive. You can also do `cat /proc/partitions` to get the size of the drive in blocks.

Now we need to wipe the first 20 blocks and last 20 blocks from the drive:

```
dd if=/dev/zero of=/dev/sda count=20
dd if=/dev/zero of=/dev/sda seek=xxxx count=20
```

The command seeks to 20 blocks before the total block size (so you'll need to do some simple math).

Then use Ctrl+Alt+F1 (or maybe F2 or F3) to switch to GUI and press enter. Then Rocks will install.


