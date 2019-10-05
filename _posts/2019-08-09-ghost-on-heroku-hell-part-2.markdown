---
layout: post
title: Ghost-on-heroku Hell (part 2)
date: '2019-08-09 04:06:47'
image: assets/images/hGPgUd4.jpg
categories: [devops,heroku]
---

Soo, after the last time the pre-implemented imgur storage adapter failed, then next i tried the google drive api method.

This one is work by robincsamuel for the custom adapter, but i tried the pre-implemented storage adapter method by [this guy](https://elements.heroku.com/buttons/intellectualjuggernaut/ghost-on-heroku-google-drive). This one was provided with some tutorial on how to create a API credential on google drive console. That was pretty neat, i'am pretty confident with this one.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/google-console.png" class="kg-image"><figcaption>My first google drive api with its console</figcaption></figure><!--kg-card-end: image-->

This one pretty straight forward, after I got my google drive api credential in form of json, i should enter that to config vars on Heroku deployment like this

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-3.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/config-vars.png" class="kg-image"><figcaption><em>Well, this should be pretty straight forward right ?</em></figcaption></figure><!--kg-card-end: image-->

Then after the deployment, i straight to the post and tried to upload cover image. The same error is happened, not exactly but just same kind of error, just different error message was displayed.

<!--kg-card-begin: markdown-->

`PEM routines:PEM_read_bio:no start line`

<!--kg-card-end: markdown-->

(that was the error message, but displayed over the cover image) still, &nbsp;no way i debug this things. Quick google search said this is probably the problem with json parser implementation, but, since i don't really wanna mess &nbsp;with lot of this stuff. But then somehow i decided to try this method and some of previous method several times make sure i didn't missed anything. Still same result, then &nbsp;i just stopped right here.

Then i was thinking, &nbsp;_"Why don't i implement my own storage adapter ?"_

Okay, i decided to give it a shot. I forked the repository straight from cobyism, clone it to my local, and try the (very limited) tutorial from the creator of each custom storage adapter.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-3.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/tutorial-ghost-github.png" class="kg-image"></figure><!--kg-card-end: image-->

I looked up to the pre-implemented Github storage adapter from [tonyerwin](https://github.com/tonyrewin/ghost-on-heroku), and saw some of the changes was (supposedly) pretty straight forward. Just some changes on app.json and bin/create-config

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/d898fc8709ff6da285090ab29cf1b0d0.js"></script><!--kg-card-end: html-->

And also add this block to app.json

<!--kg-card-begin: html--><script src="https://gist.github.com/luqmansen/199f8ba5206ee0b6a17e2c77e906de83.js"></script><!--kg-card-end: html-->

Commit it, push to the develop branch on your repo and (supposedly) you good to go.

After waiting for deployment, this deployment led to same message

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/heroku-app-error.png" class="kg-image"></figure><!--kg-card-end: image-->

I tried this one and other method like the ghost-github, ghost-google-drive, and ghost-imgur several times. But still no luck. Probably not because i missed anything, but i just don't know how node.js work, yeah that's my homework too.

> "I think i would just rollback to my medium to make my life easier. "

But then i thinking, why i don't made one that same as what cobyism did. Yeah that would be great idea, right ? So i install ghost locally on my computer. Install the node modules for the storage adapter, and stuff.

Long story short, i want to deploy it straight away to heroku. Just add some procfile like i did when deploy my flask app, what could go wrong ?

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/failed-to-push.png" class="kg-image"><figcaption>yeah, </figcaption></figure><!--kg-card-end: image-->

So, after some quick google and try to fix stuff. Still no luck. Then because it was late night, i did my last shot to [this one](https://elements.heroku.com/buttons/snathjr/ghost-on-heroku), made by awesome guy [SNathJr](https://github.com/SNathJr/ghost-on-heroku)

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-1.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/ghosst2-cloudinary.png" class="kg-image"><figcaption>last shot</figcaption></figure><!--kg-card-end: image-->

Long story short, the deployment success, and the image upload success (yaaay), and this is my very first cloudinary image assets

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-1.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/Untitled.png" class="kg-image"><figcaption>myself for the past 3 days</figcaption></figure><!--kg-card-end: image-->

My analysis is that this Cloudinary was by default is available on heroku as addons, so its probably has better integration with deployed heroku app. Other than that, it has pretty good free tier services. Probably its sufficient until my domain even expired.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="https://res-4.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/cloudinary-addons.png" class="kg-image"></figure><!--kg-card-end: image-->

Well, after that, i imported my json backup from previous blog, reupload all of the post image and finally it's done.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://res-5.cloudinary.com/hmfrvrfdc/image/upload/q_auto/v1/ghost-blog-images/it-works-it-5bff9f.jpg" class="kg-image"><figcaption>Thank you random stranger on internet, </figcaption></figure><!--kg-card-end: image-->

If you don't like all of this mess, indeed that Ghost blog isn't supposedly being hosted on heroku, in fact Ghost it self has it's own hosting and some pricing for different services. But yeah, if you love free stuff, all of this experience was sure worth it.

What to do next ?

- Keep writing everyday on your hard earned blog
- Learn Node.js i think

Well, that's the story of this blog, See ya !!!

In case you haven't read the first part, here [https://blog.luqmansen.me/ghost-on-heroku-hell-part-1/](https://blog.luqmansen.me/ghost-on-heroku-hell-part-1/)

