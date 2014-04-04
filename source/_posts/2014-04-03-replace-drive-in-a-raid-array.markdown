---
layout: post
title: "Replace a Drive in a Raid Array"
date: 2014-04-03 21:12:21 -0700
comments: true
categories: [raid, raid5]
---


I built a raid 5 array some time ago, but never really tested it.  I wanted to figure out if I knew what to do in case a drive failed on me before I started loading them up with important data.  This is what I went through to remove a drive, partition a new one and add it to the array.

#####Removing a Drive

In order to remove a drive, it must be makred as faulty.  It may be marked faulty through failure or manually.

To manually mark a drive as faulty you can do it by using the -f/--fail flag, lisk so: `sudo mdadm /dev/md0 -f /dev/sdd1` where `/dev/md0` is your array and `/dev/sdd1` is the drive to be marked faulty.

You can see the drive has been marked as faulty in the array details.

    $ sudo mdadm --detail /dev/md0
    
     Number   Major   Minor   RaidDevice State
       0       8       97        0      active sync   /dev/sdg1
       1       0        0        1      removed
       3       8       33        2      active sync   /dev/sdc1

       1       8       17        -      faulty spare   /dev/sdb1

<!-- more -->
Now we're going to remove the faulty drive with the -r/--remove flag.

    $ sudo mdadm /dev/md0 -r /dev/sdb1
    mdadm: hot removed /dev/sdb1 from /dev/md0

and running the details we find that it is no longer there.

    $ sudo mdadm --detail /dev/md0

    Number   Major   Minor   RaidDevice State
       0       8       97        0      active sync   /dev/sdg1
       1       0        0        1      removed
       3       8       33        2      active sync   /dev/sdc1

**Caution** If you restart your machine at this time, you're going to encounter an array degredation warning on boot up.  Ubuntu is looking out for you and verifying that you wish to continue.

####Partitioning a drive

Now we'll parition a drive to replace the one that failed.  You want these drives to be identical for best results.  The drives I'm using are 3TB and I'll be using **parted** to get larger than 2TB hard drives partitioned with a GPT partition.  I'm just reformating the drive I failed which is still located on `/dev/sdb`.

Open the drive with **parted**, set the table, and create the partition


    $ sudo parted /dev/sdb
    (parted) mklabel gpt
    (parted) mkpart somename  ext4 1MiB -1

*I remember reading -1 was to get you the full size of the array.  Use with caution, YMMV.*

Now you can print the parition to see the changes

    (parted) print
    Model: ATA ST3000DM001-1CH1 (scsi)
    Disk /dev/sdb: 3001GB
    Sector size (logical/physical): 512B/4096B
    Partition Table: gpt
    
    Number  Start   End     Size    File system  Name  Flags
     1      1049kB  3001GB  3001GB  ext4         1

If you don't remember what the beginning and end of the other hard drives were, you can use `sfdisk` to copy the partition table from one of your working array devices.

    sudo sfdisk -d /dev/sdg | sfdisk /dev/sdb

####Adding a drive

Now that your drive is partitioned, we'll add the drive to the array.

    $ sudo mdadm --manage /dev/md0 --add /dev/sdb1
    mdadm: added /dev/sdb1

We can check the status of the addition by running

    $ cat /proc/mdstat
    Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
    md0 : active raid5 sdb1[4] sdg1[0] sdc1[3]
          5860267008 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
          [>....................]  recovery =  0.1% (4802048/2930133504) finish=403.5min speed=120816K/sec
    
    unused devices: <none>

Once the hard drives are sync'd, `mdstat` should look like

    $ cat /proc/mdstat
    Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
    md0 : active raid5 sdf1[3] sde1[4] sdd1[0]
          5860267008 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
    
          unused devices: <none></none>

#####Conclusion

Now I know what I need to do if a drive were to truly fail on me.  I also learned that you can add more hard drives using the same procedure for adding a drive here. Hopefully I can find a drive that's identical to the ones I already have.

Here are some links and commands that were really helpful in figuring out what to do.

**Partitioning** - http://www.sleeandtopher.com/how-to-partition-a-gpt-hard-drive-in-ubuntu/  
**blkid** - Print block device attributes such as name, label and UUID  
**fdisk -l** - List partition tables  
**lshw -C disk** - list Hardware of Class disk  
