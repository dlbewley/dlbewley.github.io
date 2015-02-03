---
title: Tuning EXT4 Journal Parameters for Zimbra
layout: post
tags: [ zimbra ]
---

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
