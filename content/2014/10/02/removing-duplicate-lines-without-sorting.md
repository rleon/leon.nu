+++
date = "2014-10-02"
draft = false
title = "removing duplicate lines without sorting"
slug = "removing-duplicate-lines-without-sorting"
description = "remove all duplicates without need to sort input prior removing."
tag = ["linux", "awk", "script", "tip", "bash"]
+++

Usually whenever we have to remove duplicate entries from a file, we do a sort of the entries and then eliminate the duplicates using `uniq` command.

This solution has number of drawbacks:

* doesn't preserve lines order
* not so easy to sort based on substring

In order to overcome the drawbacks above, the prefered way is to use `awk` utility.

For example, let's remove duplicates from the log while sorting will be done based on value number four in the string:

The log consists from the following lines:
```log
➜  cat /tmp/dump
  1_kswapd0-68    [003] ...1  1037.233657: remove_from_page_cache: s_dev=0:2 i_ino=0 offset=819870
  2_kswapd0-68    [003] ...1  1037.233757: remove_from_page_cache: s_dev=179:13 i_ino=106503 offset=25576
  3_kswapd0-68    [003] ...1  1037.233657: remove_from_page_cache: s_dev=0:2 i_ino=0 offset=819870
  4_kswapd0-68    [003] ...1  1037.233757: remove_from_page_cache: s_dev=179:13 i_ino=106503 offset=25576
  5_kswapd0-68    [003] ...1  1037.233657: remove_from_page_cache: s_dev=0:2 i_ino=0 offset=819870
  6_kswapd0-68    [003] ...1  1037.233757: remove_from_page_cache: s_dev=179:13 i_ino=106503 offset=25576
```

And the command itself will be `awk '!x[$4]++'`.
```log
➜  awk '!x[$4]++' /tmd/dump
  1_kswapd0-68    [003] ...1  1037.233657: remove_from_page_cache: s_dev=0:2 i_ino=    0 offset=819870
  2_kswapd0-68    [003] ...1  1037.233757: remove_from_page_cache: s_dev=179:13 i_i    no=106503 offset=25576
```
