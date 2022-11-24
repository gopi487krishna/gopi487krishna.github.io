---
layout: post
title: How to Implement a System Call in Linux
author: Gopi Krishna Menon, Ankit Kumar
tags: Saturday-Posts
---
R
In this post we will learn about how to implement a simple system call in Linux that will print `Invoked Hello System Call`  into the kernel log buffer.

## Background

Following files will be modified once we complete the implementation of this System Call :

- **syscall_64.tbl** -- located under **arch/x86/entry/syscalls**

syscall_64.tbl acts as the registry that kernel uses to maintain the record of all the system
calls available. The kernel scripts parse this file and associate the system call number with the
function to be invoked.

- **kernel Makefile** -- root directory of kernel source

This is the main makefile defined in the root directory of the `linux-kernel-x` folder. When creating
a system call we have 3 options on where to the define the system call itself.

- If it is related to any subsystem then define the SystemCall inside the perticular folder
- If it is a miscellaneous system call then define it inside `kernel/sys.c`
- Define it inside a seperate directory within the kernel source.

We are going with the third option here. So we will define a directory that contains the implementation
of the system call and we will link it back to the kernel. In order to link back to the kernel, we 
will be required to modify the makefile.


## Steps 


- Open `syscall_64.tbl` under `arch/x86/entry/syscalls` and add an entry as shown below

```
/* -- snip -- */
450 common  set_mempolicy_home_node sys_set_mempolicy_home_node                                                                       
451 common  hello_system_call   sys_hello
/* -- snip -- */

```
The first value indicates the unique number used to identify that system call. The second value
indicates the ABI ( In our case we dont care if its x86 or amd64 so common ). The third value
indicates the name of the system call which in our case is hello_system_call and fourth value
is the function itself that needs to be invoked when a system call with this number is invoked.

- Now create a directory named `hello_syscall` in the root of the linux kernel source and create
two files in the directory
    - Makefile
    - hello_syscall.c


- Write the Implementation of system call inside hello_syscall.c

```c
#include <linux/kernel.h>
#include <linux/syscalls.h>


SYSCALL_DEFINE0(hello)
{
    printk(KERN_INFO "Invoked Hello System Call");
    return 0;
}
```
The `SYSCALL_DEFINEN` macros allow you to define a system call without getting involved in the details of the function signature.
You can implement the system call without using the `SYSCALL_DEFINEN` macro as well (asmlinkage void ..) but there might be some additional
metadata passed by your archiecture to the systemcall which you will have to account for explicitly. Therefore it is best to use
these macros to define the system call in a portable manner

- Now implement the Makefile as follows

```
obj-y := hello_syscall.o
```
What this makefile does is that it tells the kernel build system (kbuild) to generate an object file called `hello_syscall.o` 
which will be generated from `hello_syscall.c`  or `hello_syscall.S` (asm file) in the same directory. `y` here refers
to yes it must be built. Other values include

    - m : Build it as a module
    - $(CONFIG_**) : The decision for building this file is based on the value set for the kernel config `CONFIG**`


- Now get back to the root of the kernel source and open the Makefile 
- Navigate to line `1103` or search for the first entry of `ifeq ($(KBUILD_EXTMOD),)`` in the Makefile
- There add your system call directory to the list of directories to be built and linked to the kernel 

```
ifeq ($(KBUILD_EXTMOD),)                                                                                                              
core-y          += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ hello_syscall/ 
core-$(CONFIG_BLOCK)    += block/                                                                                                     
core-$(CONFIG_IO_URING) += io_uring/
```

- Now save the file and and compile and install the kernel

```bash
make -j$(nproc)
sudo make modules_install
sudo cp -v arch/x86_64/boot/bzImage /boot/vmlinuz-***
sudo mkinitcpio -k 5.19.9 -g /boot/initramfs-***.img
grub-mkconfig -o /boot/grub/grub.cfg
reboot

# replace *** with any string like "linux_abc" or version number in both commands(eg. /boot/vmlinuz-linux_abc  & initramfs-linux_abc.img)
```

- Once you have rebooted the kernel(make sure you boot with the new compiled kernel from advanced options while rebooting), test the systemcall using the following C program

```c
#include <stdio.h>
#include <sys/syscall.h>
#include <linux/kernel.h>
#include <unistd.h>
#include <errno.h>

#define HELLO_SYS_CALL 451

int main()
{
    long sys_call_status;

    sys_call_status = syscall(HELLO_SYS_CALL);

    if (!sys_call_status)
    {
        printf("Successfully invoked system call 451\n");
    }

    return 0;
}
```
- Compile the file using 

```bash
gcc -o test test.c
```
- Run the compiled file

```bash
./test
```
- Now in the kernel buffer you should see a message printed as shown below

```bash
sudo dmesg

-- Snip -- 
[ 4196.683788] Invoked Hello System Call
```

## Additional references 
- [Introduction to System Call](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)
- [ Kbuild System] (https://lwn.net/Articles/21835/)
- [Writing a system call in linux](https://brennan.io/2016/11/14/kernel-dev-ep3/)

