# Lab 2: Meet the OS You Will Build

**Course:** Operating Systems, FCIM / FAF, UTM, 2026-2027
**Track:** B (build your own small OS, xv6)
**Environment:** Ubuntu VM (VirtualBox), RISC-V cross-compiler, QEMU

## Introduction

xv6 is a small teaching operating system: a re-implementation of Unix Version 6 in about 9,000 lines of C, targeting RISC-V and run under the QEMU emulator. The goal of this lab is to build it from source, boot it, use it as a tiny Unix, read some of its code, and write a first program that runs inside it.

The key idea is the split in the source tree:

- `kernel/` is the operating system itself (runs privileged): scheduler, memory, file system, system-call handlers.
- `user/` holds the ordinary programs that run on top of it: `ls`, `cat`, `echo`, the shell.

## Part 1. Build it and boot it

### 1. Getting the source

The VM image did not have `git`, so `~/xv6-riscv` did not exist yet. I installed git and cloned the repo:

```bash
sudo apt install git
git clone https://github.com/mit-pdos/xv6-riscv ~/xv6-riscv
cd ~/xv6-riscv
ls    # kernel/  user/  Makefile  README ...
```

### 2. Building

```bash
make qemu
```

The kernel and all user programs compiled with `riscv64-linux-gnu-gcc` (the cross-compiler was already installed), and `mkfs` packed the user programs into the disk image `fs.img`.

The build then failed at the last step with `qemu-system-riscv64: No such file or directory`. The `qemu-system-misc` package was installed, but on this Ubuntu release the RISC-V emulator lives in a separate package. Fix:

```bash
sudo apt install -y qemu-system-riscv
which qemu-system-riscv64    # /usr/bin/qemu-system-riscv64
```

### 3. Booting

```bash
make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ 
```


To leave xv6 I press Ctrl-A, release, then X. This is the QEMU exit sequence, not an xv6 command.

## Part 2. Use xv6 as the Unix it is

At the `$` prompt I ran xv6's built-in programs:

```
$ ls
$ cat README
$ echo hello xv6          # hello xv6
$ ls | grep c             # cat, echo, wc, sync, console
$ wc README               # 48 336 2441 README
$ usertests -q            # ALL TESTS PASSED
```

Notes on the output:

- `wc README` reports 48 lines, 336 words and 2441 bytes. The byte count matches the size `ls` printed for `README` (2441).
- `ls | grep c` kept only the entries with a "c" in the name.
- `usertests -q` printed a lot of `usertrap(): unexpected scause ...` lines. This is expected: several tests (`kernmem`, `MAXVAplus`, `nowrite`, `lazy_unmap`, ...) deliberately make a process touch memory it must not access, to check that the kernel kills the process instead of crashing. The run ended with `ALL TESTS PASSED`.

### Observe

**1. Three programs xv6 ships with**

`cat`, `echo` and `grep` (the list from `ls` also has `wc`, `ls`, `sh`, `kill`, `mkdir`, `rm`, `ln` and more).

**2. Which two OS features must exist for a pipe to work?**

1. **Processes** (`fork`/`exec`): the shell starts `ls` and `grep` as two separate processes running at the same time.
2. **Inter-process communication** (the `pipe()` system call, a kernel buffer connected to file descriptors): `ls` writes into one end and `grep` reads from the other, so the output of one becomes the input of the other.

**3. xv6 shell vs. the Linux shell from Lab 1**

The xv6 shell is a very small version of the same idea: it runs programs and supports pipes and redirection, but it lacks the comfort features of the Linux shell (tab completion, command history, variables, globbing, scripting).

## Part 3. Read the source

Commands run:

```bash
sed -n "1,30p" user/cat.c
cat user/user.h
grep -n "sys_read\|sys_write" kernel/sysfile.c | head
```

`user/cat.c` reads from a file descriptor into a 512-byte buffer and writes the bytes to descriptor 1 (standard output). `user/user.h` lists the system calls a user program can use, and the matching handlers live in `kernel/sysfile.c`:

```
69:sys_read(void)
83:sys_write(void)
```

### Observations

**1. System calls used by `user/cat.c`**

- `read(fd, buf, n)`: asks the kernel to copy up to `n` bytes from the open file `fd` into `buf`, and returns how many it got (0 at end of file, negative on error).
- `write(1, buf, n)`: asks the kernel to write `n` bytes from `buf` to descriptor 1, the console.
- `open(path, flags)`: asks the kernel to open a file and hand back a file descriptor.
- `close(fd)`: asks the kernel to release that descriptor.
- `exit(status)`: asks the kernel to terminate the process.

(`fprintf` is not a system call itself; it is a user library function that ends up calling `write`.)

**2. Where `sys_read` is implemented**

`kernel/sysfile.c`, line 69 (the function name; its return type `uint64` is on line 68).

**3. `kernel/` vs `user/`**

`kernel/` is the privileged operating system that implements the system calls, while `user/` holds ordinary programs that can only ask the kernel for services through those calls.

## Part 4. Write your first xv6 program
 
I added a new user program, `user/sleep.c`, that pauses for a number of clock ticks.
 
One difference from the lab handout: in this version of xv6, `user/user.h` declares `int pause(int);` and has no `sleep`. The system call was renamed, so the program calls `pause()` instead of `sleep()`. Everything else follows the handout.
 
```c
#include "kernel/types.h"
#include "user/user.h"
 
int
main(int argc, char *argv[])
{
  if(argc != 2){
    fprintf(2, "usage: sleep <ticks>\n");
    exit(1);
  }
  pause(atoi(argv[1]));   // system call into the kernel
  exit(0);
}
```
 
I registered it in the `UPROGS` list in the `Makefile`, on its own line starting with a tab:
 
```
	$U/_sleep\
```
 
Then rebuilt and booted with `make qemu`. The build compiled `user/sleep.c`, linked `user/_sleep`, and `mkfs` included it in `fs.img`.
 
Test inside xv6:
 
```
$ sleep 10
$
$ sleep
usage: sleep <ticks>
$
```
 
- `sleep 10` pauses for 10 ticks, then returns to the prompt.
- `sleep` with no argument fails the `argc != 2` check, prints the usage message to standard error (descriptor 2), and exits with status 1.
