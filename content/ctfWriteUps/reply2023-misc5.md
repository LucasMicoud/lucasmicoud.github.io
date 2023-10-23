+++
title = 'Misc 5 - Acropalooza'
date = 2023-10-17
draft = false
series = "Reply Cybersecurity 2023"
+++

For this challenge, we are presented with an archive containing :

- A data file named Secret.acrp

- A python script named ascreencapture.py

- An ELF 64bit library named acrop.so

## A lazy script

Trying to execute the python script, we get more info on its purpose :

```bash
┌─[✗]─[parrot@parrot]─[~/reply/misc5/Misc-500]
└──╼ $python3 ascreencapture.py -h
usage: ascreencapture.py [-h] {crop,screenshot,view} ...

Command-line tool with submenus

optional arguments:
  -h, --help            show this help message and exit

Subcommands:
  {crop,screenshot,view}
                        Available subcommands
    crop                call native function
    screenshot          Take a secure screenshot
    view                view a secure image
```

So it can take screenshot, crop them and show them, as they are stored "securely".

Going through the script, we see that the script itself does not do very much, and that instead most of the work is done by functions in the library. So understanding how the program works might be a little more complicated that reading python code...

## First contact

So ! Let's try to view our Secret picture : 

```bash
┌─[parrot@parrot]─[~/reply/misc5/Misc-500]
└──╼ $python3 ascreencapture.py view Secret.acrp 
Opening image in 'Secret.acrp'...
Password: 
```

So we need a password... 

## Let's reverse the library

The library can be disassembled with cutter. There we find multiple interesting things.

First, let us note the mention of md5 functions in the program. This will be useful later.

Secondly, when we enter a bad password for a specific file, the program prints "Wrong password". Using this string as a reference point, we can find in the binary where the password check is taking place, and maybe bypass it.

And indeed, looking up cross references on the string in cutter, we find the set of instructions we are looking for : 

```
0x000046a4      mov     rcx, qword [rbp - 0x20]
0x000046a8      mov     rax, qword [rbp - 0x10]
0x000046ac      mov     edx, 0x10
0x000046b1      mov     rsi, rcx
0x000046b4      mov     rdi, rax
0x000046b7      call    memcmp
0x000046bc      test    eax, eax
0x000046be      je      0x470a
0x000046c0      mov     rax, qword [PyExc_RuntimeError]
0x000046c7      mov     rax, qword [rax]
0x000046ca      lea     rdx, str.Wrong_password
```

So 16 bytes of data corresponding to the picture password and the one entered by the user are loaded and compared with a call to `memcmp`.

We could try to use a debugger to modify the register just before the jump to circumvent the password check.

## Debugging a library executed with python

As the library is executed by a python script, the easiest way to debug it would be to launch it normally and to attach gdb to it when the program's execution is halted by the password prompt.

We can do it by manually searching the pid of the program, but as we might have to do it multiple times, we will use the following command to do it automatically :

```bash
gdb -p $(pgrep -f 'python3 ascreencapture.py')
```

Now we would like to be able to add breakpoints to the program. However, as the library is being executed with python, the address are not static at all, and we need to find a reference point in the assembly. 

If we go a little further from the previous assembly code, we can find a named function, `PyInit_acrop()` at address `0x00004ab6`. This would be our reference point. Now, if we want to add a breakpoint at address `0x000046bc` in the library (where the result of `memcmp` is checked), we would do it in gdb the following way :

```
b *(PyInit_acrop - 0x4ab6 + 0x46bc)
```

Now, from this breakpoint, we can set the register value to zero with : 

```
set $eax=0
```

and continue execution with `c`. 

However the program then prompts us with "CRC check failed". This is a bad sign. CRC, or cyclic redundancy check, is used to detect errors in data, for exemple in PNG files. Even if we use the same method to bypass it, the picture we get has every chance to be corrupted. In fact, here is what we get for a random password :
![first_res.PNG](/images/8bb42503-5d20-490e-a26f-40cd9ab2823a.png)

## Back to the binary

So we cannot really bypass the password check, as the user's password seems to be used to encrypt the picture. We have to find the right password. 

As we pointed out before, the code is littered with mentions to md5 functions. In addition, we saw earlier that the data compared is 16 bytes long, which seems to corroborate the fact that the password is stored as a md5 checksum.

Using gdb, we can dump the hash without having to understand how it is stored in the image. 

The checksum is **bb62ddeec478a117b4088eda899ca965**, and a little crackstation search reveals that the password is **porc**. Using that, we can view the image :

![noflag.png.PNG](/images/9397b55f-8676-4803-852d-5e070110223a.png)

## One final effort

We can finally view the picture, but there is no flag in it. Nothing in the raw file, nor in the metadata. However exiftool shows something interesting :

```
[...]
Image Width                     : 1920
Image Height                    : 971
[...]
```

What a strange resolution... If we remember that the program can also crop pictures, we can easily imagin that the picture was somthing like 1920x1080 and that the flag was in the cropped part. Depending on how the data is stored in the `.acrp` files, the data might still be inside.

So let's try to use the program to get it back :

```bash
┌─[parrot@parrot]─[~/reply/misc5/Misc-500]
└──╼ $python3 ascreencapture.py crop -h
usage: ascreencapture.py crop [-h] --xy1 XY1 --xy2 XY2 filename

positional arguments:
  filename    Name of the output screenshot file

optional arguments:
  -h, --help  show this help message and exit
  --xy1 XY1   X,Y-coordinates of the top-left corner of the crop region
  --xy2 XY2   X,Y-coordinates of the down-right corner of the crop region
┌─[parrot@parrot]─[~/reply/misc5/Misc-500]
└──╼ $python3 ascreencapture.py crop --xy1 0,0 --xy2 1920,1080 Secret.acrp 
Cropping image in 'Secret.acrp'...
Password: porc
Bad crop boundaries!
```

Too bad ! It seems the program isn't keen on letting us expand the picture this way. If we look at the assembly, arround where the "Bad crop boundaries" string is referenced, we can see the following code :

```
0x000031f8      call    fcn.00002d6c ; fcn.00002d6c
0x000031fd      movzx   edx, word [rbp - 2]
0x00003201      movzx   eax, word [rbp - 0xa]
0x00003205      cmp     dx, ax
0x00003208      jae     0x324b
0x0000320a      movzx   edx, word [rbp - 6]
0x0000320e      movzx   eax, word [rbp - 0xa]
0x00003212      cmp     dx, ax
0x00003215      jae     0x324b
0x00003217      movzx   eax, word [rbp - 2]
0x0000321b      movzx   edx, word [rbp - 6]
0x0000321f      cmp     dx, ax
0x00003222      jb      0x324b
0x00003224      movzx   edx, word [rbp - 4]
0x00003228      movzx   eax, word [rbp - 0xc]
0x0000322c      cmp     dx, ax
0x0000322f      jae     0x324b
0x00003231      movzx   edx, word [rbp - 8]
0x00003235      movzx   eax, word [rbp - 0xc]
0x00003239      cmp     dx, ax
0x0000323c      jae     0x324b
0x0000323e      movzx   eax, word [rbp - 4]
0x00003242      movzx   edx, word [rbp - 8]
0x00003246      cmp     dx, ax
0x00003249      jae     0x3271
0x0000324b      mov     rax, qword [PyExc_ValueError] ; 0x6fa8
0x00003252      mov     rax, qword [rax]
0x00003255      lea     rdx, str.Bad_crop_boundaries ; 0x507a
```

Here the program is doing multiple checks before printing the strings. If we use gdb to break at the first one, and bypass each of them the same way we bypassed the password check, the execution continues normally , and we can then use the view function to finally see the full picture :

![flag.png](/images/7dc021ea-6352-4db8-981d-e7f13bc8048e.png)

Where the flag is written in the bottom right corner : **FLG{7hisACr0p4l00z4isForL00z4}**
