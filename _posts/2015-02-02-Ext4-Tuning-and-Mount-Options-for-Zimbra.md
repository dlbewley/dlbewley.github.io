---
title: Tuning ext4 Creation and Mount Options for Zimbra
layout: post
tags: [ ansible, zimbra ]
---

[Zimbra](http://www.zimbra.com/) is a email collaboration suite. Its various compontents perform MTA duties, message store, full text indexing. In a large environment, the number of files and I/O operations can really add up. How we ensure the filesystem is ready to support it?

# Zimbra's Recommendations #

Zimbra [offers some guidance](http://wiki.zimbra.com/index.php?title=Performance_Tuning_Guidelines_for_Large_Deployments#File_System) for tuning the filesystem, with tips like:

- Mount file systems with the `noatime` option.

  It generally is not important to know the last access time of all the files, so the extra write ops are wasteful.

- Enable `dirsync` for ext3/ext4 file systems. (mailstores, indexes, MTA queues)

  The mailbox and MTA fsync for files, but it is possible to lose the directory entry for the file in  crash.
  It is also possible to affect this with `chattr -R +d`.

  More [dirsync info](http://lwn.net/2002/0214/a/dirsync.php3).

Mount Options      | Description
-------------------|------------
`noatime`          | Do not update access time
`dirsync`          | Immediately flush directory operations

The following filesystem creation options are also recommended.

Filesystem Options | Description
-------------------|------------
`-O dir_index`     | Use hashed b-trees to speed up lookups in large directories.
`-m 2`             | By default 5% of space is reserved. That can be a lot on a big filesystem.
`-i 10240`         | Bytes per Inode or `inode_ratio`. An inode will be created for every _X_ bytes. So _X_ should be your average file size.
`-J size=400`      | Journal size can influence metadata performance, so boost the size.

How many filesystems do we need? Which filesystems need which options? Let's see what directories make good candidates for separation.

# Zimbra Filesystems #

How fine grained do you split up the filesystems? Below are some key directories within Zimbra which may or may not be candidates for unique filesystems.

Dir                | I/O Type         | Latency Sensitivity | Function
-------------------|--------------------------------------------------
/opt/zimbra        | Random           | Low                 | Application root
/opt/zimbra/backup | High write       | Low                 | Nightly dump of all other dirs. (Use a NFS mount)
/opt/zimbra/db     | Random           | High                | Message metadata in MySQL. Disambiguates message blob location and tags etc
/opt/zimbra/data   | Random           | High                | Data for amavisd, clamav, LDAP, postfix MTA, etc
/opt/zimbra/data/amavisd/tmp | Random | High                | Temp files created when Amavisd feeds mail to ClamAV and Spamassassin can be sped up with a [RAM disk](http://wiki.zimbra.com/wiki/SpamAssassin_Customizations#2._Put_Amavis.27s_Temp_Dir_on_a_RAM_Disk)
/opt/zimbra/index  | High Random      | High                | Lucene full text index
/opt/zimbra/redolog| High Write       | High                | Transaction log of all activity
/opt/zimbra/store  | Random           | High                | Message blob store

# Separating out Filesystems #

For starters I'm going to go with:

- /opt/zimbra
  Nothing special yet. Just noatime.
- /opt/zimbra/backup
  Backups will go to a NFS filer. I'm not concerned about the filesystem config there. That a job for the NAS admin.
- /opt/zimbra/index
- /opt/zimbra/redolog
- /opt/zimbra/store


# Selecting Filesystem Options #

Based on those recommendations, let's decide which to follow, and how.

I will be working on RHEL 6, and I'm ignoring that fact that anything other than ext4 exists.

## Directory Index ##

The dir_index option speeds up directories with large numbers of files. It turns out this is on by default in `/etc/mke2fs.conf`. You can confirm by looking for `dir_index` in the filesystem features in the output of `tune2fs`. So, nothing to do here.

{% highlight text %}
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
{% endhighlight %}

## Bytes per Inode ##

Inodes store metadata about files. Every file consumes an inode and you can't add more inodes. Zimbra stores message blobs as individual files, as opposed to one file per mail folder (ala mbox). Therefore the inode usage can be very high. You may have tons of space free, but if you run out of inodes, pack it in. Use `df -i` to examine your inode usage.

Zimbra suggests `-i 10240` mke2fs option. This says create 1 inode for every 10KB of space on the filesystem. This assumes that you expect to fill the filesystem up with 10KB files. 

How many inodes do I have on my test filesystem?

{% highlight text %}
[root@zimbra-mbox-10 ~]# tune2fs -l /dev/mapper/VGzstore-LVstore | grep 'Inode count'
Inode count:              67108864
{% endhighlight %}

If I use the `-i 10240` option, how many inodes will I have then? You can find out with `mke2fs` in dry run mode. See line 4 below.

{% highlight text linenos %}
[root@zimbra-mbox-10 ~]# umount /opt/zimbra/store
[root@zimbra-mbox-10 ~]# mke2fs -n -i 10240 /dev/mapper/VGzstore-LVstore | grep inodes
mke2fs 1.41.12 (17-May-2010)
107479040 inodes, 268435456 blocks
13120 inodes per group
{% endhighlight %}

Wow! That is support for up to 107 million files.

How big is your average message? It looks like in my case we are looking at about 190 million emails split across numerous servers, and an average message size of 106KB. So an inode per 10KB is pretty generous. I can probably increase that ratio, and lower the inode count. 

Doing the math, it looks like by default I'm getting one inode per 16K. This is borne out in `/etc/mk2fs.conf`.

{% highlight text %}
[defaults]
        base_features = sparse_super,filetype,resize_inode,dir_index,ext_attr
        blocksize = 4096
        inode_size = 256
        inode_ratio = 16384
{% endhighlight %}

So the default may be just fine. I could even raise the number, but I won't.

## Journal Size ##

Journal size reportedly can have a great impact on metadata operations. [Zimbra suggests](http://wiki.zimbra.com/index.php?title=Performance_Tuning_Guidelines_for_Large_Deployments) an arbitrary 400M for journal size for filesystems. 

If I've already created a filesystem, so how do I find out the size of the existing journal?

Looks like `tune2fs -l` doesn't do the trick.

{% highlight text %}
[root@zimbra-mbox-10 ~]# tune2fs -l /dev/mapper/VGzstore-LVstore
tune2fs 1.41.12 (17-May-2010)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          f859c1f6-28b5-4c4b-8bd2-618abcd33607
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash
Default mount options:    (none)
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              67108864
Block count:              268435456
Reserved block count:     2684354
Free blocks:              264172519
Free inodes:              67108853
First block:              0
Block size:               4096
Fragment size:            4096
Reserved GDT blocks:      960
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Mon Feb  2 21:41:09 2015
Last mount time:          Mon Feb  2 21:41:38 2015
Last write time:          Mon Feb  2 21:41:38 2015
Mount count:              1
Maximum mount count:      32
Last checked:             Mon Feb  2 21:41:09 2015
Check interval:           15552000 (6 months)
Next check after:         Sat Aug  1 22:41:09 2015
Lifetime writes:          16 GB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               256
Required extra isize:     28
Desired extra isize:      28
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      2394798b-a767-4a9c-b92f-dabd75e5f1e4
Journal backup:           inode blocks
{% endhighlight %}

After a little googling, I found [this page](http://blog.dailystuff.nl/2012/07/getting-ext34-journal-size/) that suggested `dumpe2fs`. It looks like the default size for this 1TB filesystem was 128M.

{% highlight text %}
[root@zimbra-mbox-10 ~]# dumpe2fs /dev/mapper/VGzstore-LVstore | grep ^Journal
dumpe2fs 1.41.12 (17-May-2010)
Journal inode:            8
Journal backup:           inode blocks
Journal features:         (none)
Journal size:             128M
Journal length:           32768
Journal sequence:         0x00000001
Journal start:            0
{% endhighlight %}

What size is actually a good size? Well, [this article](http://www.linux-mag.com/id/7666/) suggested that 256MB may be a good number. Also somewhat arbitrary.

Let's just go with 256MB for now. I don't believe you can adjust the journal size after the fact, but in my case it doesn't matter. I'm going to recreate it.

I'll of course use the ansible [filesystem module](http://docs.ansible.com/filesystem_module.html) and add a value of `-J size=256` in the `opts` parameter.

# Summary #

So, in the end I'll be using Ansible [filesystem](http://docs.ansible.com/filesystem_module.html) and [mount](http://docs.ansible.com/mount_module.html) modules to setup filesytems whith a journal size of 256MB, and a reserve of 1%, and use noatime and dirsync mount options. I create a dictionary like this to do that.

{% highlight yaml %}
zimbra_storage:
  nfs:
    backup:
      export: nfs-server:/zimbra/backup
      path: /opt/zimbra/backup
  devices:
    opt_disk:
      dev: /dev/sdb
      size: 500G
      vg: VGzopt
      volumes:
        opt:
          name: LVzimbra
          path: /opt/zimbra
          size: 50G
          fs_type: ext4
          fs_opts:
          mount_opts: "noatime,dirsync"
        index:
          name: LVindex
          path: /opt/zimbra/index
          size: 100G
          fs_type: ext4
          fs_opts: "-J size=256"
          mount_opts: "noatime,dirsync"
        redo:
          name: LVredo
          path: /opt/zimbra/redolog
          size: 200G
          fs_type: ext4
          fs_opts:
          mount_opts: "noatime"
    store_disk:
      dev: /dev/sdc
      size: 2T
      vg: VGzstore
      volumes:
        store:
          name: LVstore
          path: /opt/zimbra/store
          size: 1T
          fs_type: ext4
          fs_opts: "-J size=256 -m 1"
          mount_opts: "noatime,dirsync"
{% endhighlight %}
