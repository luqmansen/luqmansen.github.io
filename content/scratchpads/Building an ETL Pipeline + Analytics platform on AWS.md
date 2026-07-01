---
title: "Building an ETL Pipeline + Analytics platform on AWS"
draft: true
---

This post might be half a technical post and half a rant, from the POV of someone new to data engineering-related services on AWS, and may be specific to our use case. Though you may still find it relevant to you.

## Intro

For the past five months, I’ve been helping the newly founded data team (as the second member of the team) to build my company’s data infrastructure platform. 

For the past few years, since the company was founded, we have pretty much found that hooking up our production read replica with BI tools was sufficient to get whatever data we want. This is pretty much true for lots of early-stage companies that don’t have much data. As we grow, you might have guessed it, lots of analytical queries start to fail, because of transaction conflict with the master database. 

So, around 2 years ago, my colleague started to lay the groundwork for more proper data analytics in our company. Though this was some time ago, before the idea of a data team even existed, and it was more like an individual effort.

## What do I already have?

Long story short, some data infrastructure was formed. It was based on Redshift with a federated query to the RDS. It was good for a while until we hit the same transaction-cancellation problem again. 

The problem (or opportunity?) with federated query is that it does a push-down predicate to your source database. Means that if you do some heavy join/filter that operates on the RDS table, that operation will be executed by the RDS. This, in a sense, may be a good thing because you can minimize the data movement between these two databases. But this is the exact reason why we want to move off of directly querying on the RDS. 

So, the hack around was to spin up a new RDS instance from our daily snapshot and use the RDS instance as the source of the federated query. This way, essentially we only have one user that queries our DB, hence should not inflict any transaction conflict. 

So, it’s all good now? Kind of, but we have a somewhat ugly architecture, but it works….

![[building-an-etl-pipeline-analytics-platform-on-aws__image.png]]

## Redshift

Actually, it’s a good piece of software if you’re using it right. I won’t go deeper into Redshift specific feature/tuning, but more like the general idea of it, and where it doesn’t fit in our use case.

So, Redshift is promoted as “a fully managed, petabyte-scale data warehouse service in the cloud”. First of all, we don’t have a petabyte of data. Heck, our data barely even scratches a terabyte. So, the use of Redshift feels is overkill in our use case. 

A `DC2.large` node (which is the smallest variant of Redshift node) comes with 2 vCPU + 16GB of RAM + 160GB storage, which costs around US$240 per month. But in my opinion, that node spec is oddly configured. The storage is definitely way too low for that amount of RAM. If we want to fit 1TB of data, we’ll need at least 7 nodes, which will cost over $1500 per month!

What we ended up using was a single `DC2.large` node, but since we use federated query to the source database, we save lots of storage because we don’t need to move around the data to our Redshift instance. The final transformed data fits nicely under 160GB storage of Redshift.

So, it’s all good now? No, it’s not!

![[building-an-etl-pipeline-analytics-platform-on-aws__image-1.png]]

The node variant that we’ve been using will be killed next year :pepehang:

## What do I want?

Why don’t we just migrate to RA3 or even Redshift serverless? Again, I’m thinking that the value of Redshift within our scale is totally not worth the value that we paid for. 

I’m thinking, for less than 10 TB of data, an RDS instance with 8 CPU + 32GB of memory **plus** an **OLAP/columnar storage extension** should be sufficient. But this is such a bummer since you cannot install any of these extensions to RDS.  

So, I’m resorting to another option, like everyone else does, which is running analytics on your blob storage! 

## Storage - Compute Separation

So, the idea is that either side can scale infinitely, without constraining the other.  In our case, we’ll have our data stored in a virtually infinite S3 bucket, and the query engine will query our data in S3.

How do we query our data? AWS has two options for this:  Redshift Spectrum and AWS Athena.

1. Reshift Spectrum
    
    This is a query engine that comes with Redshift. Of course, you’ll need to provision the Redshift cluster itself before able to use it.
    
2. AWS Athena 
    
    Athena is based on [PrestoDB](https://prestodb.io/) from Meta. This is the “true” serverless query engine that you don’t have to provision any instance to be able to use it (though it comes with a caveat).
    

Sounds neat, right? Before we dive deeper, let’s step back and answer this: How do I get our production data to the S3?

## Loading to S3

(to be continued)