---
title: "/proc/[PID]/stat Field Reference"
description: "The /proc/[PID]/stat file exposes process status information. Each field is listed below in order of appearance."
date: 2026-03-29T23:08:22-07:00
lastmod: 2026-03-31T06:38:27-07:00
image: 
categories:
    - linux
tags:
    -  kernel
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

The `/proc/[PID]/stat` file exposes process status information. Each field is listed below in order of appearance.

## Fields

| ID | Field | Description |
|----|-------|-------------|
| 1 | `pid` | Process ID |
| 2 | `tcomm` | Filename of the executable |
| 3 | `state` | Process state: `R` running, `S` sleeping, `D` uninterruptible sleep, `Z` zombie, `T` traced or stopped |
| 4 | `ppid` | Process ID of the parent process |
| 5 | `pgrp` | Process group ID |
| 6 | `sid` | Session ID |
| 7 | `tty_nr` | TTY the process uses |
| 8 | `tty_pgrp` | Process group ID of the TTY |
| 9 | `flags` | Task flags |
| 10 | `min_flt` | Number of minor faults |
| 11 | `cmin_flt` | Number of minor faults including children |
| 12 | `maj_flt` | Number of major faults |
| 13 | `cmaj_flt` | Number of major faults including children |
| 14 | `utime` | User mode jiffies |
| 15 | `stime` | Kernel mode jiffies |
| 16 | `cutime` | User mode jiffies including children |
| 17 | `cstime` | Kernel mode jiffies including children |
| 18 | `priority` | Priority level |
| 19 | `nice` | Nice level |
| 20 | `num_threads` | Number of threads |
| 21 | `it_real_value` | _(Obsolete, always 0)_ |
| 22 | `start_time` | Time the process started after system boot |
| 23 | `vsize` | Virtual memory size |
| 24 | `rss` | Resident set memory size |
| 25 | `rsslim` | Current limit in bytes on the RSS |
| 26 | `start_code` | Address above which program text can run |
| 27 | `end_code` | Address below which program text can run |
| 28 | `start_stack` | Address of the start of the main process stack |
| 29 | `esp` | Current value of ESP |
| 30 | `eip` | Current value of EIP |
| 31 | `pending` | Bitmap of pending signals |
| 32 | `blocked` | Bitmap of blocked signals |
| 33 | `sigign` | Bitmap of ignored signals |
| 34 | `sigcatch` | Bitmap of caught signals |
| 35 | _(placeholder)_ | _(Was wchan address — use `/proc/[PID]/wchan` instead)_ |
| 36 | _(placeholder)_ | _(Reserved)_ |
| 37 | _(placeholder)_ | _(Reserved)_ |
| 38 | `exit_signal` | Signal sent to the parent thread on exit |
| 39 | `task_cpu` | CPU the task is scheduled on |
| 40 | `rt_priority` | Realtime priority |
| 41 | `policy` | Scheduling policy (see `man sched_setscheduler`) |
| 42 | `blkio_ticks` | Time spent waiting for block I/O |
| 43 | `gtime` | Guest time of the task in jiffies |
| 44 | `cgtime` | Guest time of child tasks in jiffies |
| 45 | `start_data` | Address above which program data+BSS is placed |
| 46 | `end_data` | Address below which program data+BSS is placed |
| 47 | `start_brk` | Address above which the heap can be expanded with `brk()` |
| 48 | `arg_start` | Address above which the command line is placed |
| 49 | `arg_end` | Address below which the command line is placed |
| 50 | `env_start` | Address above which the environment is placed |
| 51 | `env_end` | Address below which the environment is placed |
| 52 | `exit_code` | Thread exit code as reported by `waitpid` |

## Example

```
871869 (VN-Presence) S 12198 871869 12198 34830 871869 4194560 1205 0 173 0 46734 8492 0 0 20 0 2 0 10523393 176685056 2837 18446744073709551615 94266814361600 94266814993397 140734575381584 0 0 0 0 0 16386 0 0 0 17 16 0 0 0 0 0 94266815325280 94266815330208 94267110821888 140734575389833 140734575389847 140734575389847 140734575394794 0
```

Parsed field by field:

| ID | Field | Value | Notes |
|----|-------|-------|-------|
| 1 | `pid` | `871869` | Process ID |
| 2 | `tcomm` | `(VN-Presence)` | Executable name |
| 3 | `state` | `S` | Sleeping (interruptible) |
| 4 | `ppid` | `12198` | Parent PID |
| 5 | `pgrp` | `871869` | Process group ID |
| 6 | `sid` | `12198` | Session ID |
| 7 | `tty_nr` | `34830` | TTY device number |
| 8 | `tty_pgrp` | `871869` | Foreground process group of TTY |
| 9 | `flags` | `4194560` | Task flags |
| 10 | `min_flt` | `1205` | Minor page faults |
| 11 | `cmin_flt` | `0` | Minor faults including children |
| 12 | `maj_flt` | `173` | Major page faults |
| 13 | `cmaj_flt` | `0` | Major faults including children |
| 14 | `utime` | `46734` | User-mode CPU time (jiffies) — ~467s at 100 Hz |
| 15 | `stime` | `8492` | Kernel-mode CPU time (jiffies) — ~84s at 100 Hz |
| 16 | `cutime` | `0` | Children user-mode jiffies |
| 17 | `cstime` | `0` | Children kernel-mode jiffies |
| 18 | `priority` | `20` | Kernel priority |
| 19 | `nice` | `0` | Nice value (0 = default) |
| 20 | `num_threads` | `2` | Thread count |
| 21 | `it_real_value` | `0` | Obsolete, always 0 |
| 22 | `start_time` | `10523393` | Jiffies after boot when process started |
| 23 | `vsize` | `176685056` | Virtual memory size (168 MiB) |
| 24 | `rss` | `2837` | Resident set size (pages) — 2837 × 4 KiB ≈ 11 MiB |
| 25 | `rsslim` | `18446744073709551615` | RSS limit (0xFFFFFFFFFFFFFFFF = unlimited) |
| 26 | `start_code` | `94266814361600` | Start of executable text segment |
| 27 | `end_code` | `94266814993397` | End of executable text segment |
| 28 | `start_stack` | `140734575381584` | Stack start address |
| 29 | `esp` | `0` | Current stack pointer |
| 30 | `eip` | `0` | Current instruction pointer |
| 31 | `pending` | `0` | Pending signals bitmap |
| 32 | `blocked` | `0` | Blocked signals bitmap |
| 33 | `sigign` | `16386` | Ignored signals bitmap |
| 34 | `sigcatch` | `0` | Caught signals bitmap |
| 35 | _(placeholder)_ | `0` | _(Was wchan)_ |
| 36 | _(placeholder)_ | `0` | _(Reserved)_ |
| 37 | _(placeholder)_ | `0` | _(Reserved)_ |
| 38 | `exit_signal` | `17` | Signal sent to parent on exit (17 = SIGCHLD) |
| 39 | `task_cpu` | `16` | Last CPU the task ran on |
| 40 | `rt_priority` | `0` | Realtime priority (0 = not realtime) |
| 41 | `policy` | `0` | Scheduling policy (0 = SCHED_NORMAL) |
| 42 | `blkio_ticks` | `0` | Block I/O wait ticks |
| 43 | `gtime` | `0` | Guest time (jiffies) |
| 44 | `cgtime` | `0` | Children guest time (jiffies) |
| 45 | `start_data` | `94266815325280` | Start of data+BSS segment |
| 46 | `end_data` | `94266815330208` | End of data+BSS segment |
| 47 | `start_brk` | `94267110821888` | Start of heap |
| 48 | `arg_start` | `140734575389833` | Start of command-line args |
| 49 | `arg_end` | `140734575389847` | End of command-line args |
| 50 | `env_start` | `140734575389847` | Start of environment |
| 51 | `env_end` | `140734575394794` | End of environment |
| 52 | `exit_code` | `0` | Exit code |

## Notes

- Fields are space-separated and appear in the order listed above.
- Time values (`utime`, `stime`, etc.) are measured in **jiffies** (clock ticks). Divide by `sysconf(_SC_CLK_TCK)` to convert to seconds.

## References

- `The /proc Filesystem` [linux kernel doc](https://www.kernel.org/doc/html/latest/filesystems/proc.html#id12)
- Linux kernel source: `fs/proc/array.c`
