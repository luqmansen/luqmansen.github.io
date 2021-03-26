---
layout: post
title: My First Docker run
image: assets/images/sadfasdr3gh2sz3.png
date: '2019-08-24 17:58:38'
categories: [programming,learning, docker]
---

So this is 00:38 AM, Docker for windows is suck (probably the windows tho). I just managed to get my first docker container working. I shouldn't have took that long if i just keep everything default, but since the default didn't fit with my system, so i need some configuration.

The default container vm by default was put on C: drives, and you know, my C: drives is just 120 GB, so i try to move it to my HDD. This is when things getting ugly.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="/assets/images/asdlhgrunvo1234osdvnOIu0-.png" class="kg-image"><figcaption>I already change it, the default was in c:</figcaption></figure><!--kg-card-end: image-->
1. Advanced setting has it settings for disk image location, but simply change it from there resulting the docker desktop hang, tried to start but it took so long, can't force close the docker, has to restart, and i tried this several times, no chance 
2. Changes from Users/Username /AppData/ Roaming/ Docker/ settings.json, basically, this should work, but windows permission was total mess. I pointed from this file to my D: drive
<!--kg-card-begin: markdown-->

    "dataFolder": "D:\\VM\\docker-vm",

<!--kg-card-end: markdown-->

but, somehow my VM folder on D: drive refuse to work with me, i have setting its permission to allow access to administrator and my account, docker still not able to access it. Then i just noticed that there is permission for &nbsp;"docker-user" in there, but still no luck.

My temporary solution now was set it to accessible to everyone, and everything &nbsp;fine now. I know i shouldn't do this, yeah, at this point, i really consider using &nbsp;linux.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="/assets/images/sadfasdr3gh2sz3.png" class="kg-image"><figcaption>Boo</figcaption></figure><!--kg-card-end: image-->