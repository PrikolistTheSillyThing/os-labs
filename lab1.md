# Operating Systems Lab Report: Observing the OS at Work
---

## Introduction

An operating system sits between user programs and the hardware, deciding who gets what and when. In lecture this was mostly theory; in this lab, that manager is observed directly.

The lab was carried out in a terminal on the provided Linux virtual machine. It covers the four core jobs of any OS:

- **Files:** how the OS organizes, stores, and protects data
- **Processes:** how running programs are created, scheduled, and tracked
- **Memory:** how the OS allocates and monitors RAM
- **Devices:** how hardware is exposed to programs and users

For each area, the report gives the commands used and, more importantly, **what they showed**: the output, what it says about how the OS behaves, and any surprises along the way.

```
vboxuser@Ubuntu:/home$ whoami
vboxuser
vboxuser@Ubuntu:/home$ uname -a 
Linux Ubuntu 7.0.0-34-generic #34-Ubuntu SMP PREEMPT_DYNAMIC Wed Sep  2 14:29:37 UTC 2026 x86_64 GNU/Linux
vboxuser@Ubuntu:/home$ uptime
 10:10:45 up  4:23,  1 user,  load average: 0.90, 0.68, 0.53
vboxuser@Ubuntu:/home$ 
```

**Environment:** Ubuntu 26.04, VirtualBox, kernel `7.0.0-34-generic`.

---

## Part 1. Files and directories

**Commands run:** `pwd`, `ls -la /`, `mkdir`, `cp`, `mv`, `rm`, `chmod 600`, `chmod 644`

**Observations**

1. Owner and group of my files: Both are vboxuser (columns 3 and 4 of ls -l). Compare that to everything in /, which is owned by root root:

```
ls -la /
...
lrwxrwxrwx   1 root root     7 Apr 20 08:46 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Sep 28 15:32 boot

ls -l
...
-rw-rw-r-- 1 vboxuser vboxuser 24 Oct  4 06:54 note.txt
-rw-rw-r-- 1 vboxuser vboxuser 24 Oct  4 06:54 renamed.txt
```


2. The ten characters of `ls -l note.txt` after `chmod 600`: 

**-rw-------**

* **-**: file type (a regular file; a directory would show d, a symlink l)
* **rw-**: owner (vboxuser) can read and write, not execute
* **---**: the group gets nothing
* **---**: everyone else gets nothing

Compare that to the result of chmod 644: 

**-rw-r--r--**

After chmod 644, the group and everyone else can read the file again, but only I can write to it.


3. Two directories under `/`: 

* **/home** - holds the users' personal directories:
```
vboxuser@Ubuntu:/home$ ls -l
total 4
drwxr-x--- 16 vboxuser vboxuser 4096 Oct  4 05:49 vboxuser
```

* **/dev** - holds the devices, presented as files.
**Interesting output**

```
vboxuser@Ubuntu:~/os-lab1/demo$ chmod 600 note.txt
vboxuser@Ubuntu:~/os-lab1/demo$ ls -l note.txt
-rw------- 1 vboxuser vboxuser 24 Oct  4 06:54 note.txt
vboxuser@Ubuntu:~/os-lab1/demo$ chmod 644 note.txt
vboxuser@Ubuntu:~/os-lab1/demo$ ls -l note.txt
-rw-r--r-- 1 vboxuser vboxuser 24 Oct  4 06:54 note.txt
```

I find this output pretty interesting because it directly demonstrates the difference between different access permissions for different entities (user, group, everyone)

## Part 2. Processes

**Commands run:** `ps aux`, `top`, `sleep 300 &`, `jobs`, `kill`, `/proc/$PID/status`

**Observations**

1. PID 1 and what it is: It is ``systemd``. It's the first process the kernel launches, and it starts and supervises most of the rest of the system.
```
root           1  0.2  0.3  26556 11040 ?        Ss   05:47   0:17 /usr/lib/systemd/systemd --switched-root --system --deserialize=51
```
2. Approximate process count on the idle VM: 249 from ``ps aux | wc -l`` (250 minus the header line), most processes are asleep waiting for something to happen, which is why the CPU showed 95% idle.
```
vboxuser@Ubuntu:/home$ top

top - 08:01:56 up  2:14,  1 user,  load average: 0.17, 0.46, 0.67
Tasks: 248 total,   2 running, 245 sleeping,   0 stopped,   1 zombie
%Cpu(s):  1.3 us,  2.8 sy,  0.0 ni, 95.4 id,  0.2 wa,  0.0 hi,  0.3 si,  0.0 st 
MiB Mem :   3398.1 total,    165.8 free,   2686.8 used,    660.1 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.    711.4 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  
   4382 vboxuser  20   0 2921736 320432  56620 R   9.0   9.2   1:28.27 ptyxis   
   3072 vboxuser  20   0 5219800 397844  84360 S   8.6  11.4  38:26.71 gnome-s+ 
  15468 root      20   0       0      0      0 I   1.7   0.0   0:11.05 kworker+ 
  14524 vboxuser  20   0 7365548 393152  79424 S   0.7  11.3   1:01.49 Isolate+ 
   1591 root      20   0  355748    448    164 S   0.3   0.0   0:15.63 VBoxDRM+ 
   4391 vboxuser  20   0  239944   3528   2820 S   0.3   0.1   0:02.38 ptyxis-+ 
  13762 vboxuser  20   0   11.8g 627732 136684 S   0.3  18.0   7:45.09 firefox  
  13982 vboxuser  20   0 2741696 160592  66328 S   0.3   4.6   0:14.43 Privile+ 
  14355 vboxuser  20   0 6865392  86756  59948 S   0.3   2.5   0:08.92 Isolate+ 
  15873 root      20   0       0      0      0 I   0.3   0.0   0:00.60 kworker+ 
  16343 vboxuser  20   0   13380   6284   4160 R   0.3   0.2   0:00.04 top      
      1 root      20   0   26556  11040   5096 S   0.0   0.3   0:17.46 systemd  
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.08 kthreadd 
      3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_wo+ 
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+ 
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+ 
```
3. `State:` line for the sleeping process: ``State: S (sleeping)``. This matches the S in the S column of ``top``, and it's the sensible state for ``sleep 300``, since it's waiting on a timer rather than using the CPU.

**Interesting output**

```
vboxuser@Ubuntu:/home$ PID=$!
   Name:	sleep
   State:	S (sleeping)
   Pid:	16587
   PPid:	4430
```

I find this line particularly interesting because it preserves the PID of the last process ``sleep 300`` as a variable named PID, which greatly simplifies process management.

## Part 3. Memory

**Commands run:** `free -h`, `cat /proc/meminfo`, `grep VmRSS /proc/$PID/status`

**Observations**

1. Total / free RAM: about 3.3 GiB total (``MemTotal: 3479664 kB``, the same number as ``top``'s 3398 MiB). free shows about 200 MiB free, but the number that matters more is available: 701 MiB. Most of the "used" memory is real use (2.6 GiB), but 640 MiB is ``buff/cache``, which the OS uses to keep recently read disk data handy and gives back instantly when a program needs it.

```
vboxuser@Ubuntu:/home$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.3Gi       2.6Gi       200Mi        77Mi       640Mi       701Mi
Swap:             0B          0B          0B
```
2. What swap is, and how much is configured: disk space the OS can use as overflow when RAM runs out, by moving less-used memory pages out to disk. Terminal shows 0B configured (``Swap: 0B 0B 0B``, and ``top`` said ``0.0 total`` too). That means this VM has no overflow area, so it only has its 3.3 GiB to work with.
3. VmRSS of a bare `sleep`, and whether it surprised me: 7856 kB, about 7.7 MB, which is really kind of surprising, since it takes this much memory to be doing nothing. VmRSS is 7856 kB, about half of the 16112 kB VmSize I saw in Part 2, because only part of a process's address space is in RAM.

```
vboxuser@Ubuntu:/home$ grep VmRSS /proc/$PID/status
VmRSS:	    7856 kB
```

**Interesting output**

```
vboxuser@Ubuntu:/home$ cat /proc/meminfo | head -6
MemTotal:        3479664 kB
MemFree:          195972 kB
MemAvailable:     725808 kB
Buffers:            8776 kB
Cached:           641200 kB
SwapCached:            0 kB
```
The concept of Swap memory seems pretty interesting, and the fact that inactive data can be stored in it when there is no RAM space.


## Part 4. Devices and storage

**Commands run:** `df -h`, `lsblk`, `du -sh`, `ls -l /dev`, `mount`

**Observations**

1. Device mounted on `/`: ``/dev/sda2``. Three commands agree on it: ``df -h`` shows ``/dev/sda2`` mounted on ``/`` (25G, 32% used), ``lsblk`` shows ``sda2`` with mountpoint ``/``, and mount says ``/dev/sda2 on / type ext4``. So the filesystem type is ``ext4``, and ``sda2`` is a partition of the 25G virtual disk ``sda``.
2. One `/dev` entry and the real thing it stands for: ``sda`` is the VM's whole virtual hard disk, and ``cdrom -> sr0`` is the virtual optical drive (lsblk shows sr0 as a 1024M rom).
3. "Everything is a file," in one sentence: the OS shows disks, terminals and other hardware as entries in the same file tree, so I can inspect them with the same tools (``ls -l``, ``cat``) I use for ordinary files, and the kernel handles the actual hardware behind that.

**Interesting output**

```
sda      8:0    0    25G  0 disk 
├─sda1   8:1    0     1M  0 part 
└─sda2   8:2    0    25G  0 part /
```

---

## Conclusion

In this lab I watched the OS manage four resources: files, processes, memory and devices. I saw each one with a command: `ls -l` and `chmod` showed file ownership and permissions, `ps aux` and `/proc/$PID/status` showed processes and their states, `free -h` showed memory use, and `df -h` and `lsblk` showed the disk behind `/`. What stood out is that the OS presents all of this in the same file-like way, from `/dev/sda` to `/proc`, so one small set of tools was enough to inspect every resource.
