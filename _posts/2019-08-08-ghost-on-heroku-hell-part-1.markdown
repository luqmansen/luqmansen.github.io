---
layout: post
title: Ghost-on-heroku Hell (part 1)
date: '2019-08-08 20:06:50'
image: assets/images/681f406a558dd262b50b6b6d29730c32.jpg
categories: [devops,heroku]
---

Ghost-on-heroku, a work by [cobyism](https://github.com/cobyism/ghost-on-heroku) that supposedly to be make us, free stuff lover whole lot easier. Well it is, but i dunno why it is took so long for me to make it work the way i would like it to be.

Okay, so here's the thing, for the past 3 days, i have been working for my personal blog with Ghost blog engine, and it while (perhaps, its also because of its based on node.js) the Ghost blog itself is not supposed to be hosted on Heroku where it used something called "ephemeral file system" that basically, if my site goes to sleep, then all of my assets (mainly the image for all assets that doesn't included when deployed) eg: post image assets, home cover image, etc, that normally should be stored in the file system itself, but because of file system that i mentioned before, &nbsp;it's all gone.

So, i started from [this](https://github.com/cobyism/ghost-on-heroku)repository, it got over 600 star and 400 fork, so this should be a good sign i guess. I open the page. So, there is lot of read me there, meh, who still read the read me ? Lets just fire it up

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-3.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/silly-me.png" class="kg-image"><figcaption>things i should know ? I don't think i need to know anything, silly me</figcaption></figure><!--kg-card-end: image-->

So, after the first deployment, i still not realize about that. Probably because i've been writing some blog post since then, so my blog is always up.

Few days later, i comeback to my blog and notice all of my post assets is gone, and i read &nbsp;that ephemeral things, and this is when things is messing up.

Aaand, thing getting worst from now

> Heroku app filesystems [aren’t meant for permanent storage](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem), so file uploads are disabled by default when using this repository to deploy a Ghost blog to Heroku. If you’re using Ghost on Heroku with S3 file uploads disabled, you should leave all environment variables beginning with `S3_…` blank.

This is stated clearly on the readme. I should have read that things in the first place. First, i backup all of my post from ghost admin page, i know i'll mess something later, so safety first.

Then, somehow, i think that upgrading my ghost to ghost 2.0 was a good idea, yea indeed. I followed [this](https://www.initialapps.com/upgrading-to-ghost-2-0-on-heroku/)tutorial and surprisingly it worked very smooth, but it didn't change the storage configuration, so my assets was still missing (obviously).

Then i started to reading something about [ghost storage adapter](https://ghost.org/docs/concepts/storage-adapters/) and there this things called [ghost-github](https://github.com/ifvictr/ghost-github).

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-5.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/ghost-gitub.png" class="kg-image"><figcaption>ghost-github</figcaption></figure><!--kg-card-end: image-->

It's a storage adapter for ghost, especially if you deploy it on heroku (or other platform that has same file system). It's basically going to make the assets of the ghost blog saved to your github account. After reading carefully, i still have no idea of how i should implement this.

Then i did some research on google, and i met [this one](https://github.com/tonyrewin/ghost-on-heroku).

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/tonyerwin.png" class="kg-image"></figure><!--kg-card-end: image-->

This is a ghost-on-heroku forked from original cobyism, but it already implemented that ghost-github storage adapter for us. Just launch the deploy and i thought i good to go. So I take down my current blog, and confidently delete the app from heroku because i will use the same app name for this one. Deploy it. and it is done.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://trinities.org/blog/wp-content/uploads/finally.jpg" class="kg-image" alt="Gambar terkait"></figure><!--kg-card-end: image-->

Or is it? I was wrong. it still same as before, and even more it is still ghost 1.x rather than my updated ghost 2.0. Something went wrong when i upload the cover image, it says some sort of alien code that i'am not yet understand. Meanwhile i already took down my main blog that supposed to be the production one.

> Lesson learned : Don't use production site to test your app.

Okay, so i &nbsp;created new app in heroku, called dev-blog-luqmansen for testing purposes. &nbsp;Okay done. Then i redirect someone who accessed my blog domain to a maintenance page (i knew even tho no one is probably visiting, this should be be good practice).

Then i did more research on google, I found another interesting github page, [this one](https://github.com/m1guelpf/ghost-heroku) and [this one](https://elements.heroku.com/buttons/intellectualjuggernaut/ghost-on-heroku-google-drive). The first one is work by m1guelfpg is a same concept with my first try, is using a external storage for the assets. This one is using imgur for that purposes. Considering that imgur didn't need you to login or have enter a credential to upload an image, i think this should be a good sign that this would work, right ?

Then i give it a shot, &nbsp;and somehow i knew this is going to be bad, but i tried anyway, and then this is happen

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/heroku-app-error.png" class="kg-image"><figcaption>huh ?</figcaption></figure><!--kg-card-end: image-->

I didn't modified anything, i just click the deploy button from the readme page. _"Okay then, try use heroku logs --tail, what it said ?"_

<!--kg-card-begin: markdown-->

    2019-08-08T12:59:05.450299+00:00 app[web.1]: npm ERR! A complete log of this run can be found in:
    2019-08-08T12:59:05.450444+00:00 app[web.1]: npm ERR! /app/.npm/_logs/2019-08-08T12_59_05_444Z-debug.log
    2019-08-08T12:59:05.515280+00:00 heroku[web.1]: State changed from starting to crashed
    2019-08-08T12:59:05.495689+00:00 heroku[web.1]: Process exited with status 1
    2019-08-08T12:59:06.430333+00:00 heroku[router]: at=error code=H10 desc="App crashed" method=GET path="/ghost" host=dev-blog-luqmansen.herokuapp.com request_id=4188455b-6912-4e32-b9ef-5dd5ef9b0aea fwd="202.80.213.72" dyno= connect= service= status=503 bytes= protocol=https
    2019-08-08T12:59:34.655532+00:00 heroku[router]: at=error code=H10 desc="App crashed" method=GET path="/ghost" host=dev-blog-luqmansen.herokuapp.com request_id=06cf0357-0140-4769-b400-04a993054914 fwd="202.80.213.72" dyno= connect= service= status=503 bytes= protocol=https

<!--kg-card-end: markdown-->

No idea, i just doing quick research of how to debug this deployment, and i knew if i keep digging to this problem, this going to took so long, so i ditched this thing.

This journey haven't finished, see ya in the next post....

Update part 2 : [https://blog.luqmansen.me/ghost-on-heroku-hell-part-2/](https://blog.luqmansen.me/ghost-on-heroku-hell-part-2/)

