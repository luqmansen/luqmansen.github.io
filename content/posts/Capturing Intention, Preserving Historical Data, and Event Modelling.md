---
title: "Capturing Intention, Preserving Historical Data, and Event Modelling"
date: 2023-11-09
tags:
  - software-engineering
---

For the last few months, I've been learning a lot of software engineering at Bowtie. Nothing fancy like *microservices* or such, but I think I learned something better, on how to design a good software. Let's start from here.

**Capturing Intention** 

This is very subtle concept, yet I recently thought that this is more important than it seems to be. This thing will mostly benefits the future developers that have to deal with your code later (in hope that they won't curse anyone on the gitblame)

Suppose you're having a subscription-based application, like Spotify. There might be two kinds of user, *free* user, *premium-trial* user, and *premium* user.

Suppose that you're offering a premium-trial to the free user. If they decided to accept the offer, we will create a billing coverage for the next month with $0 of invoice.

We definitely can simply do that, creating the billing with free invoice, but this will be a subtle logic that hard to figure out for the new developers. The only way to figure this out is to “find billing with $0 invoice”, that must be a premium-trial user, or, is it? What if someday, you decide to create a “subscription-waiver” voucher? Data-wise, you won't be able to tell where it comes from now 😕

Here, we need a domain object to be able to correctly preserve the state of:

- Free user, not offered
- Free user, offered, not accepted
- Free user, offered, accepted

This way, we can answer questions such as:

- Have I offered premium-trial to this user?
- How many times I have offered this user?
- Have this user ever accepted our offer?
- And many more

These questions will hard to tell if we rely from constructing the state of the billing and invoice like I previously mentioned. Having this intention captured is also beneficial for analytics. We can easily capture the effectiveness of this trial and the conversion rate it has 😁
