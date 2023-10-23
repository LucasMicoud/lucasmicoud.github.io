+++
title = 'Misc 1 - Kingply'
date = 2023-10-17
draft = false
series = "Reply Cybersecurity 2023"
+++

In this challenge, we are presented with an email exchange.

## Mails and attachments

In the exchange, we can see a zip file and a png being exchanged. The zip is protected with a password. 

## Picture analysis

The piture does not seem to contain any meaningful information :

![photo_misc1.png](static/images/d042e29a-4fa0-4748-8f72-743e6d9864ec.png)

Using *exiftool* we get something : 

```
ExifTool Version Number         : 12.16
File Name                       : photo.png
Directory                       : .
[...]
Artist                          : birthDateMail***R3ply!
Copyright                       : checkArtistFieldForPwdFormat
[...]
```

Here we get the format of the flag : `birthDateMail***R3ply!`

## Password pieces

During the CTF, we got information from the organization concerning the password format : the date format is YYMMDD and the "*" are random characters.

In the picture we can see the birthdate : **02/08/1990**, so, not knowing which one is the day or the month, the date part is either 900208 or 900802. 

"Mail" must be the user's mail : **jfeng@veryrealmail.com**

## Cracking the zip

Using the following script, we generate a wordlist of every possible password : 

```python
import string

pre1 = "900208jfeng@veryrealmail.com"
pre2 = "900802jfeng@veryrealmail.com"
suf = "R3ply!"

characters = string.ascii_letters + string.digits + string.punctuation + string.whitespace

wordlist = [pre1 + char1 + char2 + char3 + suf for char1 in characters for char2 in characters for char3 in characters]
wordlist += [pre1 + char1 + char2 + char3 + suf for char1 in characters for char2 in characters for char3 in characters]

with open('wordlist.txt', 'w') as file:
    file.writelines('\n'.join(wordlist))
```

Then we use john to crack the password :

```bash
zip2john flag.zip > hash
john hash --wordlist wordlist.txt
john hash --show
```

With the password (**900802jfeng@veryrealmail.com!@#R3ply!**) in our possession, we can unzip the archive and find the flag : **{FLG:J4n1c3_h4s_g0t_s0m3_b4d_t33th}**
