---
layout: post
title: Fixing a non booting kernel based on custom kernel config file
author: Gopi Krishna Menon
tags: quickfix
---
Recently I was trying to migrate my kernel config across Virtual Box, VMware and
Qemu based VM's and noticed that in some of the cases, I was not able to boot
into the kernel at all. I did enable vmware, virtual box and qemu specific
options in the kernel but still the kernel was not booting up (It was booting up
on my machine but not on a friend's machine). Looking deep into the problem I
noticed that the versions of vmware and qemu were different than that of my
machine.

## Now how to solve this issue?
Well, I cannot adjust the config for every version of vmware, qemu or virtual
box, so we need to find a way so that the person using this config can fix it by
themselves. Thats where localmodconfig comes in

So make has a target called `localmodconfig` that enables all the modules that are
currently being used by the kernel. It will also deselect the modules which are
currently not being used by kernel which in turn allows us to create a minimal
kernel config. Not only that, `localmodconfig` can also use the `.config` file
present in the kernel build directory. This allows us to have a base kernel with
necessary options already enabled.

So lets begin fixing our config

### Step 1 : Cleaning the Kernel
We will start with a fresh build of the kernel so reboot into the default kernel
and switch to the kernel build directory and run `make mrproper`. Invoking this
target will ensure that all the build artifacts generated during the build will
be removed (including the config file). 
<script id="asciicast-G1m1laR2RvG2p4MSm4jwvZpVm" src="https://asciinema.org/a/G1m1laR2RvG2p4MSm4jwvZpVm.js" async></script>
### Step 2 : Copying the config file
Copy the config file into the kernel build directory and rename it to .config.
After that run `make nconfig` or `make menuconfig` and press escape (and save
the config if a prompt appears). Doing this will update the config file for the
kernel version being used.
<script id="asciicast-6R2CewYM88Vp6UBdaM7U1Btb1" src="https://asciinema.org/a/6R2CewYM88Vp6UBdaM7U1Btb1.js" async></script>

### Step 3 : Running localmodconfig
Run `make localmodconfig` to enable modules presently being used by the kernel.
If there are no warnings or errors proceed with the kernel compilation (You can
run `menuconfig` or `nconfig` to see if important options are enabled).

But if there are some warnings saying that the config option is disabled or not
present go to **Step 4**

<script id="asciicast-SdX7cM2fPazx2PuT4cDSixI1s" src="https://asciinema.org/a/SdX7cM2fPazx2PuT4cDSixI1s.js" async></script>

### Step 4 : Setting additional config options 
Looking at the messages given by local modconfig, we can see that the last
word of these messages contains the `config option (starting with CONFIG)` that
needs to be set in the `.config` file. So just open the file using your
favourite text editor and add/edit the config options mentioned before and set
it to `y` (or `m`).Run `make menuconfig` or `make nconfig` and save the config
file. 

If there are too many options to be configured, you can use `awk` and `sed` to
add the config options automatically

- First save the output of `localmodconfig` in a text file
```bash
make localmodconfig > add_opts.txt 2>&1
```
- Now run the `awk` command to get the last column representing the config option
and save it to a new file
```bash
awk '{print $(NF)}' add_opts.txt > add_opts_cleaned.txt       
```
- Now to put `=y` in front of these configs we use the `sed` command
```bash
sed -i s/$/=y/ add_opts_cleaned.txt
```
- Finally open the file, take a look at if everything is correct and copy paste
  the entire content of the file into `.config` file
- Now run `make menuconfig or make nconfig`. These targets will automatically
  set the additional config options required. Now press `ESC` key on your
  keyboard and save the configuration.

<script id="asciicast-khyuEsWWS1QLj5b5SFWT16GuX" src="https://asciinema.org/a/khyuEsWWS1QLj5b5SFWT16GuX.js" async></script>

### Step 5 : Continuing the Compilation
Continue the kernel compilation by running `make -j$(nproc)`

## Conclusion
Using this method will help you to boot into the kernel. If you want to get 
a complete list of the kernel modules required (for running everything as per your
requirements, you can have a look at
[modprobed-db](https://github.com/graysky2/modprobed-db)

## Additional References
Introduction to Awk : [Learning Awk Is Essential For Linux Users](https://youtu.be/9YOZmI-zWok)

Introduction to Sed : [Intro to Sed](https://www.gnu.org/software/sed/manual/sed.html)
