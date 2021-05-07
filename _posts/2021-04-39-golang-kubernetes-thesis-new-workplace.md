---
layout: post
title: Golang, Kubernetes, Thesis, and New Workplace 
date: '2021-04-30 22:47:50'
image: assets/images/go-k8s-workplace/dumpster.jpg
categories: [Self]
---


Yep, the title pretty much describes how <s>(dumpster of fire)</s> my life was for the past 4 months.  

Note: this isn't a technical blog post.

First of all, Golang, a programming language that I used to love (and probably still), that's why I also choose that as the backend for the system that I built for my bachelor thesis, also that's why I applying to a Go backend engineer roles at my current workplace. But, no matter how much you love something, if you have to work with that for more than 16 hours a day, you'd sure have a mental breakdown at some point.

Sure working with Go on the side project you love to learn is very pleasing. Moreover, if there is no pressure when working on that. But that wasn't always the case. I would say the sentiment about "Do what you love, and you’ll never work another day in your life" is a lie. You would have a phase of your life where you felt like you don't want to do it anymore. You felt like you want to stop, you need a break. And you know what? that's fine. Everyone felt the same thing. It's okay not to always sitting on your desk and being productive. For the past few months, I constantly felt the same thing. I felt like I want to stop. But then, I realize that I just need a break, soooo yeah, just take your time.

Moving on, what's about the Kubernetes? Yep, I also use that stack on my thesis, aaand that even more frustrating. This piece of technology is relatively new. Even more, since I mainly work with on-premise Kubernetes, which limits my options for using microk8s and k3s. Kubeadm isn't an option since my testbed is very limited, and I try to find the lightweight one. And yeah, both k3s and microk8s have somewhat some kind of random error that appears out of nowhere. Like at the end of the day, after deploying everything into the cluster, and you think everything was done, there is * **always*** some component that suddenly not working. Yeah sure, you can just look at the log and google the error bla bla bla, but man, it really shattering my expectation when I think Kubernetes will just work, turns out It doesn't.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/go-k8s-workplace/nope.png" class="kg-image"><figcaption>I'm not fine</figcaption></figure><!--kg-card-end: image-->

 I finally choose k3s as my choice since it is at least more reliable than the microk8s, which also I super hate when it is managed by snap and there is no way you can install microk8s without it, in event of failure of one component, the way you could fix that is very tedious things to do because you have to deal with all of snap bullsh!t.  I spent more than a week trying to cope with that until I finally met k3s which way more reliable.

Next, is it about the new workplace, so far so good? perhaps. I am still not very used to work with people that I've physically never meet before, but I think that was a good exercise for me to get used to it. Fortunately, I met a really nice coworker, she's very open to me and asks a lot about stuff that she's working with. She's relatively new to this field, but I really like her enthusiasm. Also having small interaction make this remote working less boring.

---
I remember when I said I still want to be backend because I want to handle complex systems, with top-notch tech stack, well, here I am. Also, there is a lot of "actual" software engineering practice that I've never know before. My current target is making sense of the whole software engineering process in this organization, because I started to realize there are \*a lot\* of things that I don't know, and that's good, means I know that there are many things to learn ahead.

<!--kg-card-begin: image--><figure class="kg-card kg-image-card"><img src="/assets/images/go-k8s-workplace/twiter.png" class="kg-image"><figcaption>Old tweet</figcaption></figure><!--kg-card-end: image-->


Other things that I notice on my new workplace, like every other organization, they have a huge clusterfuck snowball of legacy code which I had a few tickets working in this service for the last 2 weeks. I saw most of the testing code here in my opinion isn't very best practice. Like, I have to wait for 26 minutes for the test to be done just to see that my test was failed is ridiculous. But the team that handles the service said that the development of this legacy code is being frozen. The team is currently trying to fix that, let us see how is it going to be.

That's all for this post, see you next time!!!