+++
date = "2014-08-26"
draft = false
title = "ext3/ext4 journal log size"
slug = "extX-journal-log-size"
description = "explanaition about ext3/4 journal log"
tag = ["linux", "kernel", "file system", "ext3/4"]
+++

### What is journal log?
A journal or log keeps the changes that are being made to the filesystem during disk writing that can be used to rapidly reconstruct corruptions that may occur due to events such a system crash or power outage.

### Why ext4 journal log is equal to `128MB`?
Journal log is created during filesystem creation. All revisions of extended filestems (ext3/4) are created by [mke2fs utility](http://git.kernel.org/cgit/fs/ext2/e2fsprogs.git/tree/misc/mke2fs.c).

This utility is taking configuration options from [mke2fs.conf.in](http://git.kernel.org/cgit/fs/ext2/e2fsprogs.git/tree/misc/mke2fs.conf.in) file.
For example ext3/ext4 options are:
```config
[defaults]
base_features = sparse_super,filetype,resize_inode,dir_index,
		ext_attr
default_mntopts = acl,user_xattr
enable_periodic_fsck = 0
blocksize = 4096
inode_size = 256
inode_ratio = 16384

[fs_types]
ext3 = {
	features = has_journal
}
ext4 = {
	features = has_journal,extent,
	huge_file,flex_bg,uninit_bg,
	dir_nlink,extra_isize
	auto_64-bit_support = 1
	inode_size = 256
}
```
So we can see from above that journal(`has_journal` option) is enabled, but position and size wasn't configured.

In general, there are 3 relevant options to journal location and size:

* `packed_meta_blocks` - Place the allocation bitmaps and the inode table at the beginning of the disk.
* `journal-size` - Create an internal journal (i.e., stored inside the filesystem) of size journal-size megabytes. The size of the journal must be at least 1024 filesystem blocks (i.e., 1MB if using 1k blocks, 4MB if using 4k blocks, etc.) and may be no more than 10,240,000 filesystem blocks or half the total file system size (whichever is smaller).
* `journal-location` - Specify the location of the journal. The argument journal-location can either be specified as a block number, or if the number has a units suffix (e.g., 'M', 'G', etc.) interpret it as the offset from the beginning of the file system.

Because no parameters from the above is set by default, Let's check how mke2fs code handles it.
```c
msic/mke2fs.c ->
  int main (int argc, char *argv[]) ->
   static void PRS(int argc, char *argv[]) ->
     packed_meta_blocks = get_bool_from_profile(fs_types, "pacmed_meta_blocks", 0);
```
==> by default `packed_met_blocks = 0` (disabled) and if we follow the code we
will find that journal log is located by default at the middle of hard drive
with special inode number 8.
```c
misc/mke2fs.c ->
  static blk64_t journal_location = ~0LL;
    ...
    int main (int argc, char *argv[]) ->
      retval = ext2fs_add_journal_inode2(fs, journal_blocks, journal_location, journal_flags);

lib/ext2fs/mkjournal.c ->
 /*
  * This function adds a journal inode to a filesystem, using either
  * POSIX routines if the filesystem is mounted, or using direct I/O
  * functions if it is not.
  */
  errcode_t ext2fs_add_journal_inode2(ext2_filsys fs, blk_t num_blocks, blk64_t goal, int flags) ->
  ...
  journal_ino = EXT2_JOURNAL_INO;
  if ((retval = write_journal_inode(fs, journal_ino, num_blocks, goal, flags))) ->
  /*
   * This function creates a journal using direct I/O routines.
   */
  static errcode_t write_journal_inode(ext2_filsys fs, ext2_ino_t journal_ino, blk_t num_blocks, blk64_t goal, int flags) ->
    ...
    es.goal = (goal != ~0ULL) ? goal : get_midpoint_journal_block(fs);

lib/ext2fs/ext2_fs.h ->
  /*
   * Special inode numbers
   */
  #define EXT2_BAD_INO         1  /* Bad blocks inode */
  #define EXT2_ROOT_INO        2  /* Root inode */
  #define EXT2_ACL_IDX_INO     3  /* ACL inode */
  #define EXT2_ACL_DATA_INO    4  /* ACL inode */
  #define EXT2_BOOT_LOADER_INO     5  /* Boot loader inode */
  #define EXT2_UNDEL_DIR_INO   6  /* Undelete directory inode */
  #define EXT2_RESIZE_INO      7  /* Reserved group descriptors inode */
  #define EXT2_JOURNAL_INO     8  /* Journal inode */
```
```bash
➜  raw  sudo tune2fs -l /dev/sda1
tune2fs 1.42 (29-Nov-2011)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
…
Journal inode:            8
Journal backup:           inode blocks
```

Let's check third important option - journal size:
```c
misc/mke2fs.c ->
  int main (int argc, char *argv[]) ->
    ...
    journal_size = -1;
     ...
    journal_blocks = figure_journal_size(journal_size, fs);

misc/util.c ->
  /*
   * Determine the number of journal blocks to use, either via
   * user-specified # of megabytes, or via some intelligently selected
   * defaults.
   *
   * Find a reasonable journal file size (in blocks) given the number of
   * blocks
   * in the filesystem.  For very small filesystems, it is not reasonable to
   * have a journal that fills more than half of the filesystem.
   */
  unsigned int figure_journal_size(int size, ext2_filsys fs) ->
    int j_blocks;
    j_blocks = ext2fs_default_journal_size(ext2fs_blocks_count(fs->super));

lib/ext2fs/mkjournal.c ->
  /*
   * Find a reasonable journal file size (in blocks) given the number of
   * blocks
   * in the filesystem.  For very small filesystems, it is not reasonable to
   * have a journal that fills more than half of the filesystem.
   */
  int ext2fs_default_journal_size(__u64 num_blocks)
  {
    if (num_blocks < 2048)
      return -1;
    if (num_blocks < 32768)
      return (1024);
    if (num_blocks < 256*1024)
      return (4096);
    if (num_blocks < 512*1024)
      return (8192);
    if (num_blocks < 1024*1024)
      return (16384);
    return 32768;
   }
```
==> As we can see, the journal log is driven by total filesystem block count.
For modern hardisks default journal log size will be always `128MB`.

and last check:
```bash
➜  raw  sudo dumpe2fs -h /dev/sda1
Journal backup:           inode blocks
Journal features:         journal_incompat_revoke
Journal size:             128M
Journal length:           32768
Journal sequence:         0x003900d1
Journal start:            25654
```
