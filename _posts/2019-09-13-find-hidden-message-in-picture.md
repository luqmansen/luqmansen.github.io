---
layout: post
title: Find Hidden Message in a Picture
image: assets/images/1200px-Wikipedia_favicon_hexdump.svg.png
date: '2019-08-27 17:16:13'
categories: [learning, ctf]
---

So this is not going to be a tutorial, but just some simple example about CTF's forensics challenge. This challenge is available at [ctflearn.com](https://ctflearn.com/problems/96) 

Basically, some strings are hidden inside an image, this is called steganography where we hide some information on image. The message is going to be messed up if the image is being compressed, eg: if you upload it on instagram, etc. enough talk, let's start !

## Find the flag on image below
<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/minion.jpg" class="kg-image"></figure><!--kg-card-end: image-->

Basically, an image is a binary file. now we just have to managed how to read a binary file.There are 3 ways i managed to do this.

### 1. Use binwalk
This is probably overkill, but this is what solve my problem for first time and my friend recommend me to use this

Install binwalk
~~~language-bash
sudo apt get install binwalk
~~~

First, try to find if a raw string on this image, use the flag format from the challenge

~~~language-bash
binwalk --raw=flag{ minion.jpg
~~~

the output is going to be 
~~~language-bash
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
9141          0x23B5          flag{
~~~

So there is a string that we find on ```0x23B5```, so we almost found our flag on that address. lets dig more, try dump the whole image

~~~language-bash
binwalk -W minion.jpg
~~~

And scroll where you can find ```0x2350```, basically, this is the first address of ```0x23b5```.
every line of the hexdump output, the hex address is incremented by 1, so just try to find that line

And we find that line
~~~language-bash
0x000023B0  D7 C6 28 C6 0E 66 6C 61 67 7B 77 6F 77 21 5F 64 |..(..flag{wow!_d|
0x000023C0  61 74 61 5F 69 73 5F 63 6F 6F 6C 7D 85 12 53 BE |ata_is_cool}..S.|
~~~

here we go, our flag was there

And we can simplify the process with simple grep-ing 2 string from the output

~~~language-bash
▶ binwalk -W minion.jpg | grep '23B0\|23C0'
0x000023B0  D7 C6 28 C6 0E 66 6C 61 67 7B 77 6F 77 21 5F 64 |..(..flag{wow!_d|
0x000023C0  61 74 61 5F 69 73 5F 63 6F 6F 6C 7D 85 12 53 BE |ata_is_cool}..S.|
~~~

Cool right ?

### 2. Use Hexdump

This one, the approach is almost the same, except we don't need to install fancy binwalk

~~~language-bash
▶ ▶ hd minion.jpg | grep '23b0\|23c0'
0x000023B0  D7 C6 28 C6 0E 66 6C 61 67 7B 77 6F 77 21 5F 64 |..(..flag{wow!_d|
0x000023C0  61 74 61 5F 69 73 5F 63 6F 6F 6C 7D 85 12 53 BE |ata_is_cool}..S.|
~~~

It just worth to mention that linux has its own native binary hex dumper

### 3. Use strings

This method was done after i submit the problem. Probably it's not really good, but some people just comment the clue too far in the ctflearn comment section. But it's just good to know tho

Simply
~~~language-bash
strings minion.jpg
~~~

we will get the output like the hexdump one, but without the hex, something like this

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/strings-minion.png" class="kg-image"></figure><!--kg-card-end: image-->

and simply
~~~language-bash
strings minion.jpg | grep flag{
~~~

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/output-strings-minion.png" class="kg-image"></figure><!--kg-card-end: image-->

by far, this one the most simple method. Probably there still a lot of method out there, just comment if you have another one.



Yeah, we are done kiddo, see you on the next post!
