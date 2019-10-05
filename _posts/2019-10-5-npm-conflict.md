---
layout: post
title: NPM conflict
image: assets/images/learn-travis-ci/npmlogo.png
date: '2019-10-5 16:04:13'
categories: [learning, npm]
---

This is typical npm flexing, 

~~~.language-bash
The following packages have unmet dependencies:
 npm : Depends: some-library (>= 0.69.9) but it is not going to be installed because 𝗙𝗨𝗖* 𝗬𝗢𝗨, thats why
E: Unable to correct problems, you have held broken packages.
~~~

yeah, that kinda bothersome. This is happen when i tried to learn travis ci, and apparently, the example was using node js app for example. 

My problem was, npm need libssl1.0.0-ish and what i got was an updated version of it. People suggesting me for using `aptitude`, and apparently, `aptitude` also unable to solve my problem.

Solution ? I tried to install that specific version that need via apt
~~~.language-bash
$sudo apt install libssl1.0.0-dev 
~~~
Aaand, yea it's installed. and now when i do aptitude, i'm able to install npm

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/dependenshit.png" class="kg-image"></figure><!--kg-card-end: image-->

And did i success ? Yea, npm installed, and i'm able to install tool that i need from npm,

But

When apt downgrading stuff, it also uninstall all stuff that depends on the thing that i downgraded. I haven't search this on internet, but perhaps it's a mecahnism to prevent the app from broken. Whats make it suck ? It uninstall my WHOLE Ros and all it's dependencys. I haven't checked for another things that also got uninstalled, but yea that suck.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/ros-removed.png" class="kg-image"></figure><!--kg-card-end: image-->

 Yea npm is very cool.


