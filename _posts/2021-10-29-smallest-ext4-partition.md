---
layout: main
title:  "How small is the smallest valid ext4 partition?"
date:   2021-10-29 01:01:01 +0200
---

Looking around the web I had stumbled upon a blogpost detailing "The smallest FAT partition", which sparked curiosity to investigate the same problem, but for the ext4 filesystem.

This post is written in an exploratory fashion. This way we also get to see how the filesystem works in some regards.

<!--more-->

<style>
    .dcontainer {
        display: flex;
        flex-direction: row;
        background-color: whitesmoke;
        width: 500px;
        height: 4ch;
        border: 2px solid black;
        border-radius: 2px;
        margin: 1em 0px 1em 0px;
        font-family: monospace;
        font-weight: bold;
    }

    .dcontainer > div {
        padding-left: 0.5ch;
    }

    .dcontainer > div:not(:last-child) {
        border-right: 1px solid black;
    }


    .bb_hl  {color: black; background-color: hsl(0, 0%, 80%)}
    .S_hl   {color: black; background-color: hsl(268, 100%, 80%)}
    .GDT_hl {color: black; background-color: hsl(59, 100%, 80%)}
    .dBM_hl {color: white; background-color: hsl(0, 0%, 40%)}
    .iBM_hl {color: white; background-color: hsl(0, 0%, 40%)}
    .iT_hl  {color: black; background-color: hsl(212, 100%, 80%)}
    .D_hl   {color: black; background-color: hsl(17, 100%, 80%)}
</style>

- [Introduction](#introduction)
    - [A recap of OS concepts](#recap)
    - [The relevant tooling](#tooling)
    - [Inspecting a real-world partition](#inspect)
- [How small can mkfs.ext4 go?](#how-small)
- [Going even lower by writing the partition by hand](#by-hand)
    - [Toying with the tiny partition](#toying-around)
- [Summoning the power of ext feature flags](#feature-flags)

<a name="introduction"></a>
# Introduction

Before we start, it's important to state our goal clearly. Apart from being as small as possible, the target partition should also meet the following criteria:
- it must be mountable _as an_ ext4 partition
- it must allow the creation _at least_ one file *with* data
- passing `e2fsck` without errors is not a cause for concern

Having that layed out let's dive in. But first, we should begin with the basics:

<a name="recap"></a>
## A recap of OS concepts

Chances are that upon hearing the words `format` and `partition`, the words `fat32` and `ntfs` pop in your mind. These are the options in the format window on windows after all. If you own a mac, you might recognize names such as `hfs`, `hfs+` or the newer `apfs`. If you've dived in the GNU/linux world, you might even know more trendy names such as `btrfs` or `bcachefs`. If you tried the BSD distributions, you should have heard about `ufs` and `zfs`, and also know by now that sometimes, we can't have nice things due to license incompatibilities.

These are all file systems, which simply put, are ways to organize files in a tree-shaped fashion, on a medium that's essentially a giant contiguous array. In the world of linux distros, ext4 is the default option, and has been so for quite some time. It's been tried-and-tested many times, and has proven it's stability.

The `ext4` filesystem is part of a family of filesystems, each one (as the name implies) extending the previous one. The versions go back to the first `ext` filesystem, which was an extension of the `MINIX` filesystem, a clone of the `ufs` filesystem made by Andrew Tanenbaum in 1987 as a teaching aid.

On GNU/linux, a common utility handles the mounting of `ext2`, `ext3` and `ext4` partitions. These are backwards-compatible, meaning one can mount an `ext2` partition as `ext4`, but not the other way around. If it's hard to visualize in which way this compatibility goes, think of USB revisions, it's the same concept. Since we're talking not just about an individual filesystem but a family of filesystems, I'll use the term `ext filesystem` to refer exclusively to any of the last 3 revisions of the filesystem, not the first one.

At it's core, `ext` filesystems are simply an array of specialized _blocks_, some containing data, others metadata. The size of these blocks (`BLOCK_SIZE`) are uniform and decided at format time, possible options being 1024, 2048 or 4096 bytes.

These blocks can be classified in the following categories:

- <span class="bb_hl">bb</span> - Boot block <br>
    A fixed 1024 byte block, where bootloader code is expected to reside. Of note is that it's size is independent from `BLOCK_SIZE`. Technically one could argue that this isn't even part of the filesystem. しょうがない.
- <span class="S_hl">S</span> - Superblock <br>
    The top-level metadata of the entire filesystem.
- <span class="GDT_hl">GDT</span> - Group descriptor table <br>
    An array of GDT structures. These are metadata that store offsets at which one can find the other sections, such as where the inode table is, or where the data blocks start.
- <span class="iT_hl">iT</span> - Inode table <br>
    An array of inode structures. These are metadata describing files. They also store at which offsets the blocks of said data files can be found.
- <span class="D_hl">D</span> - Data blocks <br>
    An area of blocks meant for storing data. A block is the smallest non-divisible unit within the filesystem.
- <span class="iBM_hl">iBM</span> - Inode bitmap <br>
    A bitmap used for quickly finding out which inodes are free
- <span class="dBM_hl">dBM</span> - Data block bitmap <br>
    A bitmap used for quickly finding out which data blocks are free

There's one more thing we need to mention in order to complete the picture of an ext partition: one partition is composed of multiple "groups". The size of each group is determined by a constant, determined at format time. Each group has two bitmaps, a inode table and data blocks. In addition to these, a few also contain the superblock, and the GDT. The first block group contains the main superblock and block groups, the rest contain backup copies of these structures. It makes sense to have these backups because these metadata blocks are quite critical for operation. How would you go by accessing data off the filesystem when you don't even know where to look for it?

If we were to describe an ext partition by a regular expression, it could look something like this:
```bash
bb(S GDT dBM iBM iT D)((S GDT)? dBM iBM iT D)*
```

Of course the positions of dBM, iBM, iT and D can be swapped at will. There's also toggleable features that permit the grouping of the bitmaps of multiple block groups into a single one (flex_bg), but we won't dwell into this functionality as it's outside the scope of this article.

Let's look at these sections through the lens of our objective: making the smallest ext partition.

|type|positioning|size?|
|----|-----------|----|
|bb  |address 0  |1024|
|S   |&group + 0 |1 block|
|GDT |&group + 1 |how many GD structs fit in as few blocks|
|dBM |specifiable|1 block|
|iBM |specifiable|1 block|
|iT  |specifiable|<= (#bits in `$BLOCK_SIZE`) * inode_size bytes|
|D   |specifiable|<= (#bits in `$BLOCK_SIZE`) blocks|

We can draw some educated assumptions about how our tiny ext4 partition would look like:
- only one block group
- smallest `BLOCK_SIZE` (1024 bytes)
- fewest inodes possible (more down below)
- smallest `inode_size`
- only 1 data block

Regarding inodes, the first couple of inodes are reserved, and have special functions. These are always marked as "occupied" in the inode table. We may or may not be able to use these for general data storage, depending on the ext driver implementation. Here is a table of some them:

|inode|introduced|Purpose|
|-|-|-|
|1    |ext2|badblocks list|
|2    |ext2|root directory|
|3    |ext2|user quota file|
|4    |ext2|group quota file|
|5    |ext2|boot loader stage 2?|
|6    |ext2|undelete directory?|
|7    |ext3|used to speed up resizing|
|8    |ext3|journal inode|
|9    |next3|snapshots feature|
|10   |?|?|
|11   |ext2|traditional first inode|

<a name="tooling"></a>
## The relevant tooling

The following utilities, in no particular order, have been of use:
- `dd` for creating files of fixed size
- `mkfs.ext4` for formatting partitions
- `dumpe2fs` for inspecting metadata of ext4 partitions
- `e2fsck` for fixing broken / corrupted ext4 partitions (though in our case, for quick error checking)
- `debuge2fs` for analysing ext partitions without mounting
- `xxd` for byte-level inspection of the partition

<a name="inspect"></a>
## Inspecting a real-world partition

Let's inspect the ext4 partition my WSL is installed on. Using `dumpe2fs`, we get the following:

```
[...]
Filesystem UUID:          3255683f-53a2-4fdf-91cf-b4c1041e2a62
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super [...]
Inode count:              16777216
Block count:              67108864
Reserved block count:     3355443
Free blocks:              56637027
Free inodes:              16072978
First block:              0
[...]
Filesystem created:       Wed Apr 10 19:35:05 2019
Last mount time:          Sun Oct 31 09:32:32 2021
[...]
```

This is just the information about the superblock. What follows right after is a long list (14k lines) of information about the block groups, 2048 in my case:

```
Group 0: (Blocks 0-32767) csum 0x9ac3 [ITABLE_ZEROED]
  [...]
Group 1: (Blocks 32768-65535) csum 0xde51 [ITABLE_ZEROED]
  [...]
Group 2: (Blocks 65536-98303) csum 0x2545 [ITABLE_ZEROED]
  [...]
[...]
Group 2047: (Blocks 67076096-67108863) csum 0xccd2 [INODE_UNINIT, ITABLE_ZEROED]
  [...]
```

For each block group, it's stated which blocks and inodes are free, as well as the positioning of the bitmaps and inode table within the block group:

```
Group 81: (Blocks 2654208-2686975) csum 0xe197 [ITABLE_ZEROED]
  Backup superblock at 2654208, Group descriptors at 2654209-2654240
  Reserved GDT blocks at 2654241-2655264
  Block bitmap at 1138 (bg #0 + 1138), csum 0x62dd54b4
  Inode bitmap at 3186 (bg #0 + 3186), csum 0x3f6bbe51
  Inode table at 48161-48672 (bg #1 + 15393)
  0 free blocks, 210 free inodes, 1303 directories
  Free blocks:
  Free inodes: 666966, 667735, 669118, 669120-669326
```

Looking at the structure of an actual ext4 partition can be dizzying, especially when factoring in all the extra features. These numbers, just like the wealth of trillionares, are hard to comprehend or quantify. A better case study would be to see how low can the formatting utility can go, and inspect that instead.

<a name="how-small"></a>
# How small can mkfs.ext4 go?

In the *nix world, everything is a file. Instead of using a physical device, we can simply use an ordinary file, which we can create using `dd`:

```bash
dd if=/dev/zero of=test_fs.iso bs=1M count=1
```

Then format it with `mkfs.ext4`:

```bash
└$ mkfs.ext4 test_fs.iso
mke2fs 1.45.5 (07-Jan-2020)

Filesystem too small for a journal
Discarding device blocks: done
Creating filesystem with 256 4k blocks and 128 inodes

Allocating group tables: done
Writing inode tables: done
Writing superblocks and filesystem accounting information: done
```

What's interesting is that `mkfs` skipped creating a journal. Normally this would be a cause for concern, but for our intents and pourposes, it's just "good riddance". Interesingly enough, `file` identifies the partition as ext2, not ext4. Apparently, not having a journal doesn't even qualify as being ext3!

```bash
└$ file test_fs.iso
test_fs.iso: Linux rev 1.0 ext2 filesystem data, UUID=0b28d59b-691b-47f3-8b1d-73715d2d9397 (extents) (64bit) (large files) (huge files)
```

Reading the `man` page of `mke2fs`, we see that it's stated that the defaults are taken from `/etc/mke2fs.conf`. The main points of interest in this config file that influence the size of the partition are `blocksize` and `inode_size`.

The lowest size for `blocksize` is 1024, mainly due to how the code is written. In the superblock the value is stored in a field signifying "how many bits to the left to shift the value 1024, in order to get the blocksize". In our case, the lowest would be 0, yielding 1024 bytes per block, or 1KiB.

The lowest we can go for `inode_size` is 128, stated in the `man` page of `mkfs.ext4`.

With this knowledge, we change our `mkfs` call to look like so:
```bash
mkfs.ext2 -v -b 1024 -I 128 -m 0 test_fs.iso
```

Applying a few rounds of grad student descent, I found the smallest partition I can format through `mkfs.ext2` to be 60KiB in size. This is the winning one-liner:

```bash
dd if=/dev/zero of=test_fs.iso bs=60K count=1 && mkfs.ext2 -v -b 1024 -I 128 -m 0 test_fs.iso
```

Below is a diagram of how this partition looks like:
<div class="dcontainer">
    <div class="bb_hl"  style="flex: 1"></div>
    <div class="S_hl"   style="flex: 1"></div>
    <div class="GDT_hl" style="flex: 1"></div>
    <div class="dBM_hl" style="flex: 1"></div>
    <div class="iBM_hl" style="flex: 1"></div>
    <div class="iT_hl"  style="flex: 2"></div>
    <div class="D_hl"   style="flex: 54"></div>
</div>

And this is how it looks like through the lens of `dumpe2fs`:
```
└$ dumpe2fs test_fs.iso
dumpe2fs 1.45.5 (07-Jan-2020)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          937b865c-a91d-47c4-96a4-65dbe0f67c1b
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      ext_attr resize_inode dir_index filetype sparse_super large_file
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              16
Block count:              60
Reserved block count:     0
Free blocks:              39
Free inodes:              5
First block:              1
Block size:               1024
Fragment size:            1024
Blocks per group:         8192
Fragments per group:      8192
Inodes per group:         16
Inode blocks per group:   2
Filesystem created:       Sun Oct 31 15:54:55 2021
Last mount time:          n/a
Last write time:          Sun Oct 31 15:54:55 2021
Mount count:              0
Maximum mount count:      -1
Last checked:             Sun Oct 31 15:54:55 2021
Check interval:           0 (<none>)
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               128
Default directory hash:   half_md4
Directory Hash Seed:      5c434863-5fee-49a7-afcf-93286b25a32a


Group 0: (Blocks 1-59)
  Primary superblock at 1, Group descriptors at 2-2
  Block bitmap at 3 (+2)
  Inode bitmap at 4 (+3)
  Inode table at 5-6 (+4)
  39 free blocks, 5 free inodes, 2 directories
  Free blocks: 21-59
  Free inodes: 12-16
```

We could've gone lower by using the `-g blocks-per-group` flag. We theoretically need only 2 blocks, but `mkfs` seems to reject any number that is not a multiple of 8, or smaller than 64. This is as far as the `mkfs` utility can get us. From now on, we have to get our hands a bit dirty with some C code.

<a name="by-hand"></a>
# Going even lower by writing the partition by hand

It may seem daunting to write a partition by hand, but there isn't that much complexity. We already know quite a bit about our target partition:
- block size of 1KiB
- inode size of 128
- only one block group (therefore one GDT and no backup superblocks)
- at least 11 inodes (rounding up to 16)
- two data blocks (one for the root dir, and one for a file)

We also have a valid reference partition to cross-check along the way.

With this knowledge, we can paint the following picture:
<div class="dcontainer">
    <div class="bb_hl"  style="flex: 1">bb</div>
    <div class="S_hl"   style="flex: 1">S</div>
    <div class="GDT_hl" style="flex: 1">GDT</div>
    <div class="dBM_hl" style="flex: 1">dBM</div>
    <div class="iBM_hl" style="flex: 1">iBM</div>
    <div class="iT_hl"  style="flex: 2">iT</div>
    <div class="D_hl"   style="flex: 2">D</div>
</div>

Since we're going to create a file in increments of 1KiB blocks, we can define a buffer of that size, and after that write our ext2 partition block by block. An array of functions that fill each block is iterated, and voilla, the partition is written.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "ext2_fs.h"

int main(void) {
  unsigned char block[1024];

  FILE* image = fopen("tiny.iso", "w");
  if (image == NULL)
      FAIL("couldn't open image");

  void(*funcs[])(void*) = {
    // TODO
  };

  for (int i = 0; i < sizeof(funcs) / sizeof(void(*)()); i++) {
      memset(block, 0, 1024);
      funcs[i](block);
      fwrite(block, sizeof(block), 1, image);
  }

  fclose(image);
  return 0;
}
```

The simplest block possible to write is the boot block, because it's virtually empty. This is where bootloader code would normally go. It's nothing but wasted space for a one-file filesystem, but nonetheless, we can't work our way around it...

```c
void fill_nothing(void *block) {}
```

Next, we complete the superblock. Metadata such as the UUID or time-of-creation aren't vital in mounting the filesystem, so we won't bother filling that in.
```c
void fill_superblock(void* block) {
    struct ext2_super_block* super = block;

    super->s_magic = EXT2_SUPER_MAGIC;
    super->s_inodes_count       = 16;
    super->s_blocks_count       = 9;
    super->s_free_blocks_count  = 1;
    super->s_free_inodes_count  = 6;
    super->s_first_ino          = FIRST_INODE;

    super->s_r_blocks_count     = 0; // we don't want to reserve any blocks for the superuser
    super->s_first_data_block   = 1; // must be 1 for 1k block sizes
    super->s_inode_size         = 128;
    super->s_inodes_per_group   = 16;
    super->s_blocks_per_group   = 8192;
    super->s_frags_per_group    = 8192;
    super->s_rev_level          = EXT2_GOOD_OLD_REV;

    super->s_state  = EXT2_VALID_FS;
    super->s_errors = EXT2_ERRORS_CONTINUE;
}
```

Up next is the GDT. Since we have only one group, this will take up only one block. Here we have to lay out the block offsets for all the remaining structures: the bitmaps, inode table, and the data blocks.
```c
void fill_group_descriptors(void* block) {
    struct ext2_group_desc* gd = block;

    gd->bg_block_bitmap = 3;
    gd->bg_inode_bitmap = 4;
    gd->bg_inode_table  = 5;
    gd->bg_free_blocks_count = 1;
    gd->bg_free_inodes_count = 6;
    gd->bg_used_dirs_count = 1;
    gd->bg_pad = 4;
}
```

Next up, the bitmaps. A bit of one will signify that the respective inode/block is occupied. The whole point of this structure is to speed up the search of free inodes / blocks. Counting starts from 1, so the first bit will correspond to inode 1, and so on and so forth.

We fill both bitmaps with ones.
For the block bitmap we unflip bit #8, which corresponds to the last data block.
For the inode bitmap we unflip bits 11-16, which are the non-reserved inodes.
```c
void fill_block_bitmap(void* block) {
    memset(block, 0xff, 1024);
    ((__u32*)block)[0] &= ~(1<<7);
}

void fill_inode_bitmap(void* block) {
    memset(block, 0xff, 1024);
    ((__u32*)block)[0] &= ~(0x3f<<(FIRST_INODE-1));
}
```

We've now reached the inode table. Inspecting the tiny partition created through `mkfs` by passing it through `xxd`, we can see that only 3 inodes are actually filled:
- #1, the badblocks inode. Though only the access times are filled, so it's probably safe to skip
- #2, the root directory inode. This one is a must for obvious reasons
- #11, the lost+found directory, which we can safely skip as it's not essential.

So, empirically, we only have to fill in the root directory inode:
```c
void fill_inode_table(void* block) {
    struct ext2_inode* inodes = block;

    inodes[1].i_mode = 16877;
    inodes[1].i_uid  = MY_UID;      // my uid, to avoid a chown after mount
    inodes[1].i_gid  = MY_GID;
    inodes[1].i_size = 1024;        // size of the file
    inodes[1].i_links_count = 2;    // one is "." and the other is created at mount?
    inodes[1].i_blocks   = 2;
    inodes[1].i_block[0] = 7;       // this is the only data block (of this inode)
}
```

Lastly, we have to create the dir_entry for our root folder. The previously mentioned partition had 3 entries there: `.`, `..` and `lost+found`. We only have to fill in the first two:
```c
void fill_direntry(void *block) {
    struct ext2_dir_entry_2* dent;

    dent = (struct ext2_dir_entry_2*)block;
    dent->inode = 2;
    dent->rec_len = 0xc;
    dent->name_len = 1;
    memcpy(dent->name, ".\0\0\0", 4);

    dent = (struct ext2_dir_entry_2*)(block + 0xc);
    dent->inode = 2;
    dent->rec_len = 1012;
    dent->name_len = 2;
    memcpy(dent->name, "..\0\0", 4);
}
```

Finally, this is how the partition will be layed out:
```c
void(*funcs[])(void*) = {
    fill_nothing,           // #0 - boot block (unused)
    fill_superblock,        // #1
    fill_group_descriptors, // #2
    fill_block_bitmap,      // #3
    fill_inode_bitmap,      // #4
    fill_inode_table,       // #5 - inodes 1-8
    fill_nothing,           // #6 - inodes 9-16
    fill_direntry,          // #7 - data block - root directory
    fill_nothing,           // #8 - data block - unallocated
};
```

Running the code, we get our partition:
```bash
└$ make tiny && ./tiny
cc     tiny.c   -o tiny
└$ file tiny.iso
tiny.iso: Linux rev 0.0 ext2 filesystem data, UUID=00000000-0000-0000-0000-000000000000
```

We can also mount it and create files within it. Interestingly, creating a file via `touch` does not allocate any data blocks. This means that our tiny partitions can have up to 6 files created!

<a name="toying-around"></a>
## Toying with the tiny partition

```bash
└$ sudo mount tiny.iso mpoint/
└$ ls mpoint/
└$ for i in (seq 1 7); touch mpoint/file$i; end
touch: cannot touch 'mpoint/file7': No space left on device
└$ ls mpoint/
file1  file2  file3  file4  file5  file6
```

This works because `touch` merley allocates an inode, and modifies the directory structure to have an entry containing the filename and the newly allocated inode. No data blocks are actually allocated. Only data block 7 is modified each time a new file is created.

Since this partition has only one data block, only one of these files will be able to hold any data:
```bash
└$ echo "test" > mpoint/file1
└$ echo "test" > mpoint/file2
write: No space left on device
└$ ls -lai mpoint/
total 5
    2 drwxr-xr-x 2 nomius nomius 1024 Oct 31 19:17 ./
16420 drwxr-xr-x 4 nomius nomius 4096 Oct 31 19:16 ../
   11 -rw-r--r-- 1 nomius nomius    5 Oct 31 19:17 file1
   12 -rw-r--r-- 1 nomius nomius    0 Oct 31 19:17 file2
   13 -rw-r--r-- 1 nomius nomius    0 Oct 31 19:17 file3
   14 -rw-r--r-- 1 nomius nomius    0 Oct 31 19:17 file4
   15 -rw-r--r-- 1 nomius nomius    0 Oct 31 19:17 file5
   16 -rw-r--r-- 1 nomius nomius    0 Oct 31 19:17 file6
```

In conclusion, for 9 KiB of space we get a one-file filesystem, with a max file size of 1KiB. If you want count empty files, which technically can be of use as lockfiles, then this FS can hold up to 6 files. Though we can always invalidate the last 5 inodes by filling the respective slots in the inode bitmap with ones.

This is pretty darn small! But... what if we could go _even lower_?

<a name="feature-flags"></a>
# Summoning the power of ext feature flags

The `ext` filesystems have toggleable features, meant to enhance / tune the filesystem for different situations. These are divided into three categories:
- compat - compatible feature set. A kernel will be able to read/write to the fs even if it doesn't understand a flag
- compat_ro - read-only compat. Like the previous, but it won't be able to write to the fs.
- incompat - incompatible feature set. A kernel will refuse to mount if it doesn't understand a flag.

Of major interest is the `INCOMPAT_INLINE_DATA` and `INCOMPAT_DIRDATA` flag. The second would allow us to store small files within the direntry itself. This would cut off one block from our last partition, however, as of yet, it hasn't been fully implemented. So we'll have to settle for the other one, which is equally useful.

The `inline_data` feature allows one to store tiny files directly in the inode. This makes sense, after all if the data you're storing is comparable in size with the inode itself, why allocate an entire 1-4KiB block only to waste most of it? As an added bonus it also reduces seeks, as the data is now packed with the metadata in the same location.