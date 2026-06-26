---
title: "🗑️ Golang, Kubernetes, Thesis, and New Workplace"
date: 2021-04-30
tags:
  - life-update
---

![[golang-kubernetes-thesis-and-new-workplace__Untitled.png]]

Well, the title pretty much sums up how my life has been a dumpster of fire for the past 4 months. Just a heads up, this isn’t a technical post.

To begin with, there is Golang. I have a deep affection for this programming language (and I still am) which is why I choose selected it as the backend for my bachelor’s thesis project and I also applied as Go Backend Engineer position at my current workplace. However, no matter how much you may adore something if you must engage with it for more than 16 hours per day, it is inevitable that you may start to develop negative feelings towards it.

Don’t get me wrong, but working on a side project that you love can be extremely fulfilling, especially without any pressure. I believe the proverb saying “do what you love and you’ll never work a day in your life” if a fallacy. There will be a period in your life when you no longer desire to do something and you feel the need for a break. And that’s perfectly fine. Everyone experiences this. It’s acceptable to not always be at your desk and be productive. Over the past few months, I consistently felt like I wanted to stop, but then realized I just needed a break. So, if you’re feeling overwhelmed, just take some time yourself.

Moving on, what about the Kubernetes? Yep, I also use that stack on my thesis, aaand that's even more frustrating. This piece of technology is relatively new. Even more, since I mainly work with on-premise Kubernetes, which limits my options for using microk8s and k3s. Kubeadm isn’t an option since my testbed is very limited, and I try to find the lightweight one. And yeah, both k3s and microk8s have somewhat some kind of random error that appears out of nowhere. Like at the end of the day, after deploying everything into the cluster, and you think everything was done, there is * **always*** some component that suddenly not working. Yeah sure, you can just look at the log and google the error bla bla bla, but man, it really shattering my expectation when I think Kubernetes will just work, turns out It doesn’t.

![[golang-kubernetes-thesis-and-new-workplace__Untitled-1.png]]

I finally choose k3s as my choice since it is at least more reliable than the microk8s, which also I super hate when it is managed by snap and there is no way you can install microk8s without it, in event of failure of one component, the way you could fix that is very tedious things to do because you have to deal with all of snap bullsh!t. I spent more than a week trying to cope with that until I finally met k3s which way more reliable.

Next, is it about the new workplace, so far so good? perhaps. I am still not very used to work with people that I’ve physically never meet before, but I think that was a good exercise for me to get used to it. Fortunately, I met a really nice coworker, she’s very open to me and asks a lot about stuff that she’s working with. She’s relatively new to this field, but I really like her enthusiasm. Also having small interaction make this remote working less boring.

---

I remember when I said I still want to be backend because I want to handle complex systems, with top-notch tech stack, well, here I am. Also, there is a lot of “actual” software engineering practice that I’ve never know before. My current target is making sense of the whole software engineering process in this organization, because I started to realize there are *a lot* of things that I don’t know, and that’s good, means I know that there are many things to learn ahead.

![[golang-kubernetes-thesis-and-new-workplace__Untitled-2.png]]

Another thing that I notice on my new workplace, like every other organization, is they have a huge clusterfuck snowball of legacy code I had a few tickets working in this service for the last 2 weeks. I saw most of the testing code here in my opinion isn’t the very best practice. Like, I have to wait for 26 minutes for the test to be done just to see that my test failed is ridiculous. But the team that handles the service said that the development of this legacy code is being frozen. The team is currently trying to fix that, let us see how is it going to be.

That’s all for this post, see you next time!!!
