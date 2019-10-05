---
layout: post
title: An Attempt to Learn CI/CD Part 1
image: assets/images/learn-travis-ci/TravisCI-Mascot-1.png
date: '2019-10-5 16:04:13'
categories: [learning, travis]
---

So, this weekend, i have some spare time to learn new stuff. I've already planned to learn about CI/CD for a long time
but yeah, men plans, god laughs.

So, i decide to use Travis CI, no specific reason for that, that just what i found on google on first search.
I following [this](https://github.com/dwyl/learn-travis) tutorial from folks at [dwyl](https://github.com/dwyl/), they had a very cool tutorial on their github.


So, let me just state what i grasp from this first Attempt
1. I made an application
2. Specify the app environtment on specific file (.travis.yml) -> so that TravisCI knows
3. Specify which part of app your app you want to test with what test function (or tool) you already define on specific file.

***

So in this case, i tried to make simple js app, 

~~~.language-javascript
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello Travis!\n');
}).listen(1337, '127.0.0.1');
console.log('Server running at http://127.0.0.1:1337/');
~~~

And on the second part, we should create a .travis.yml file to specify our working environtment
so that when the travis ci engine able to detect it. The most common on is specifying language that being used, and os. Look up in the documentation for more info. 

In this case, this app is based on node js, so we specify it

~~~.language-yml
language: node_js
node_js:
 - "node"
~~~

Then, we write the package.json. This is where we specify all the dependency, description, and stuff. We also specify our test script here.

 In this case we use `jshint`. Basically it's a tool to check if there is an error or a potential error on your app. This is the `package.json` look like

~~~.language-json
{
    "name": "learn-travis-luqmansen",
    "description": "Simple Travis-CI check for JSHint (Code Linting)",
    "author": "your name here :-)",
    "version": "0.0.1",
    "devDependencies": {
      "jshint": "^2.6.0"
    },
    "scripts": {
      "test": "jshint hello.js"
    }
  }
~~~

Okay, we're done. Now try to remove one semicolon on your `hello.js` and push it to the origin/master. This is what you got

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/error-firstbuild.jpg" class="kg-image"></figure><!--kg-card-end: image-->

Well, we don't want that. Try fix your code and push it again to origin/master

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/success.png" class="kg-image"></figure><!--kg-card-end: image-->

Great, now don't forget to add cool Travis CI build-passing logo to our repo

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/cool-badge.png" class="kg-image"></figure><!--kg-card-end: image-->

And copy that markdown format to your readme.md on your repo

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/learn-travis-ci/yea.png" class="kg-image"></figure><!--kg-card-end: image-->

Yea, cool. That just half part of the tutorial, on part 2 i'll continue to write the experience here. See ya
