---
layout: post
title: Dealing with Docker Image Location on NTFS
image: assets/images/9b750e95-set-up-docker-on-ubuntu-18.04-lts.jpg
date: '2019-08-27 17:16:13'
categories: [learning, docker]
---

TL;DR : you can't, at least for now

Long Version:

So, i learning docker recently, and i notice that docker took quite a lot of space on my drive for just random image that i pull from dockerhub (yes, just for curiosity). I just being considerate my tiny weeny ssd write cycle. After my previous post on [here](https://luqmansen.github.io/my-first-docker-run/) about almost same problem, now i try to do that on linux.

After struggling with this problem for 2 days and much stack overflow later, the best solution (or most recommended) was

1. Make a new file called daemon.json on /etc/docker
2. And just fill it with this 

~~~.language-json
    {
        "data-root": "/some/external/drive/you/want"
    }
~~~


Great, we're done.

But wait

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Screenshot-at-2019-08-27-23-41-32.png" class="kg-image"></figure><!--kg-card-end: image-->

What ? okay, calm down, what could probably be the problem here, it mention about overlay stuff, what was that ?

So, apparently, docker use some kind of storage driver for writing to container writable layer. I'm not expert on docker yet, but in this case, probably docker was trying to write that image on our container, but because our container location was on ntfs file system, it has some problem.

The default storage driver being used was overlay2, this is supported for newest docker version. The storage drive that might work is vfs driver or overlayFS, but it might rather slow, as mentioned in this [issue](https://github.com/moby/moby/issues/23930).

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-3.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Screenshot-at-2019-08-27-23-52-31.png" class="kg-image"></figure><!--kg-card-end: image-->

Apparently, there is something called backing-filesystem that are know for incompatible with docker distribution, ntfs was one of them ([check here](https://github.com/moby/moby/issues/25328)). Honestly, i din't really need that fast, but I want it, because I can ;)

My solution ? Since i got 1 TB hard drive, i actually planned to convert whole drive filesystem to ext4 to fulfill my self perfectionist because i don't like many partition on my drive. But since my drive was 60% occupied, that would take an ages (not really, but just so long). Fast solution, shrink main ntfs partition, create new ext4 partition.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Screenshot-at-2019-08-27-23-59-08.png" class="kg-image"><figcaption>gotta go fast, </figcaption></figure><!--kg-card-end: image-->

Done, now create a mount point for this drive, you can totally do that by changing file /etc/fstab, but because i want to make my life easier, just do that on Disk application.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-5.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Screenshot-at-2019-08-28-00-02-12.png" class="kg-image"><figcaption>Select partition,click gear icon. select "edit mount options"</figcaption></figure><!--kg-card-end: image-->

and now change the daemon.json, but first stop the docker daemon


~~~.language-bash
    sudo systemctl stop docker
~~~


edit /etc/docker/daemon.json

~~~.language-json

    {
        "data-root" : "/your/new/ext4/mount/point/
    }

~~~

restart the docker daemon


~~~.language-bash
    sudo systemctl start docker
~~~
<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-5.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Screenshot-at-2019-08-28-00-15-15.png" class="kg-image"></figure><!--kg-card-end: image-->

Yeahh, we're done boys

