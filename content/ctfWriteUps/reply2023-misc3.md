+++
title = 'Misc 3 - Memory is cheating on you...'
date = 2023-10-17
draft = false
series = "Reply Cybersecurity 2023"
+++

We're presented with a memory dump. Let's use Volatility to analyze it.

## Volatility analysis

We use volatility 3.

`vol -f dump windows.info` confirms that this is a windows memory dump.

`vol -f dump windows.filescan` reveals a **usefulinfo.zip** file at memory offset **0xc98f43903e00**. Most of the flags in this CTF are in zip files, so this is interesting.

We use `vol -f dump windows.dumpfile --virtaddr 0xc98f43903e00` to dump the file. Volatility throws an error message while dumping the file, but still manage to output something.

Trying to open the zip fle shows that it is encrypted, so we have to dig further. 

`vol -f dump windows.pstree` reveals that the keepass password maanger was loaded at dump time. Maybe the zip password is stored inside ?

## Keepass master password

Keypass passwords are usually stored in a database called Database.kdbx. Using `filescan`, we can see such a file located in **\Users\user\Documents\Database.kdbx** and dump it using `dumpfile`. 

Trying to open the file with keepass, we see that it is protected by a masterpassword. 

We tried to use `keepass2john` and rockyou.txt to crack it, but it gave nothing. 

As keepass was loaded during the dump, if the database was opened, we might find the password in memory. 

With a little bit of seaching, we found this [article](https://www.forensicxlab.com/posts/keepass/) where the author uses a custom volatility plugin : `windows.keepass`. We download the plugin, install it by placing it in `[VolRoot]/plugins/windows`, and lauch it.

It shows that the password ends with **GN7#zZJzWmwX45WfHQ**, but as mentionned in the article the first characters might have disappeared from memory.

To find the correct password, we use the following script (trial and error showed that we needed two characters) :

```pythonag-0-1hcs52q06ag-1-1hcs52q06ag-0-1hcs52q06ag-1-1hcs52q06
``python
import string

suf = "GN7#zZJzWmwX45WfHQ"

characters = string.ascii_letters + string.digits + string.punctuation + string.whitespace

wordlist = [char1 + char2 + suf for char1 in characters for char2 in characters for char3 in characters]

with open('wordlist.txt', 'w') as file:
    file.writelines('\n'.join(wordlist))and john :
```

and john :

```bash
keepass2john Database.kdbx > hash
john hash --wordlist wordlist.txt
john hash --show
```

With the password **4MGN7#zZJzWmwX45WfHQ** we can open the database in keepass, and find the password of the zip : **$*goLWqJks9%aR5**

In the zip, we find the flag : **{FLG:MaY_TH3_m3M0ry_BE_W1tH_y0U!}**
