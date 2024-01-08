+++
title = 'Building Linux symbols for volatility 3'
date = 2023-11-22
draft = false
series = "Tutorials"
+++

# Building Linux symbols for volatility 3

If you've been doing a bit of Linux memory forensics, you might have faced cases where Volatility 3 could not handle a memory dump because the kernel version was not supported. The error message usually looks like this :

```
└─$ volatility -f memory.dmp linux.bash
Volatility 3 Framework 2.5.2
Progress:  100.00               Stacking attempts finished     
Unsatisfied requirement plugins.Bash.kernel.layer_name:
Unsatisfied requirement plugins.Bash.kernel.symbol_table_name:

A translation layer requirement was not fulfilled.  Please verify that:
        A file was provided to create this layer (by -f, --single-location or by config)
        The file exists and is readable
        The file is a valid memory image and was acquired cleanly

A symbol table requirement was not fulfilled.  Please verify that:
        The associated translation layer requirement was fulfilled
        You have the correct symbol file for the requirement
        The symbol file is under the correct directory or zip file
        The symbol file is named appropriately or contains the correct banner
```

In this tutorial, we will learn with an example how to create symbols for Volatility 3.

## Step 1 - Identifying the kernel version 

To build the symbols, we must identify the exact version of the Linux kernel that was used when the dump was made. To do so, we can run the volatility `banners` plugin on the dump :

```
└─$ volatility -f memory.dmp banners              
Volatility 3 Framework 2.5.2
Progress:  100.00               PDB scanning finished                      
Offset  Banner

0xc6001c0       Linux version 5.4.0-4-amd64 (debian-kernel@lists.debian.org) (gcc version 9.2.1 20200203 (Debian 9.2.1-28)) #1 SMP Debian 5.4.19-1 (2020-02-13)
```

So we are looking at a **Debian** machine running Linux kernel **5.4.0-4-amd64** compiled on **2020-02-13**.

## Step 2 - Finding the debug kernel

We now have to find the debug version of the kernel package. 

Doing so depends on the distribution. For Debian, we can look at [https://snapshot.debian.org/](https://snapshot.debian.org/) and go the last snapshot on the date the kernel was compiled, in our case: [https://snapshot.debian.org/archive/debian/20200213T231921Z/](https://snapshot.debian.org/archive/debian/20200213T231921Z/).

There, we can find the kernel package in `pool/main/l/linux/` under the name `linux-image-5.4.0-4-amd64-dbg_5.4.19-1_amd64.deb` (the `-dbg` indicating that we have the debug kernel).


## Step 3 - Generating the symbols

First, we have to install the package :

```
sudo apt install ./linux-image-5.4.0-4-amd64-dbg_5.4.19-1_amd64.deb
```

Then we build and use the [dwarf2json](https://github.com/volatilityfoundation/dwarf2json) utility provided by the volatility foundation :

```
git clone https://github.com/volatilityfoundation/dwarf2json
cd dwarf2json
go build

./dwarf2json linux --elf /usr/lib/debug/boot/vmlinux-5.4.0-4-amd64 > ./linux-image-5.4.0-4-amd64-dbg_5.4.19-1_amd64.json
```

Finally, we can compress the symbols and move them into the Volatility 3 folder.

```
xz -z ../linux-image-5.4.0-4-amd64-dbg_5.4.19-1_amd64.json

mv ../linux-image-5.4.0-4-amd64-dbg_5.4.19-1_amd64.json.xz [VOLATILITY 3 LOCATION]/volatility3/symbols/linux/
```

## Conclusion

This tutorial has provided a step-by-step guide for creating Volatility 3 symbols necessary to analyze a Linux memory dump.

Volatility is still a work in progress; be sure to report any issue on the official repository.

## References

- [1] GitHub.com - [Volatility 3](https://github.com/volatilityfoundation/volatility3)
- [2] GitHub.com - [dwarf2json](https://github.com/volatilityfoundation/dwarf2json)
- [3] volatility3.readthedocs.io - [Creating New Symbol Tables](https://volatility3.readthedocs.io/en/latest/symbol-tables.html)