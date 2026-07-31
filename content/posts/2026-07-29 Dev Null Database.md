---
title: Dev Null Database
---
Quite some time ago, I remember reading about "how fast is your database" or something along that line. Well, I do have a super fast database that can write 1 billion+ (or more!) inserts per second, probably the fastest in the world, no joke. But, there's a catch, it won't persist anything. Yeah just pipe that write to `/dev/null` and you have virtually unbounded write speed. Pretty cool, huh?

Today, I figured out where THAT might be actually something that I need. AND, for some reason, PostgreSQL (and apparently MySQL too) does support it.

```sql
CREATE RULE make_write_faster_rule AS 
	ON INSERT TO your_super_important_table DO INSTEAD NOTHING;
```

To my limited understanding, `DO NOTHING` here is the keyword usually being used together with `ON CONFLICT`. But what's kinda crazy to think about is, that this code path even available. I mean, this SQL is parsed and executed, meaning that this combination of execution path must be explicitly made, not something accidental. And now, is one of the day where people in my situation could use it.

Ok, the less interesting part. What did I use it for? We do host an opensource application at my company. And, because of my personal preference, I deliberately choose to use multi-tenant database, aka same postgres instance, shared across many apps (still with a proper access control of course). But, quite naive of me to think that all those apps will behave nicely to each other. Since Postgres does not support storage size limit on specific schema, well you guess it, one of the tenant decided to run amok and deplete the storage quota, firing all the AWS alert during the weekend 🚒

Well, I won't blame AI, but the app that we're hosting (it's quite popular actually) has high number of AI-contributions on github. Idk what's the state of the codebase, but this is not the first time I encounter an issue that not just annoying, but at the point where it makes the app itself is unusable AND only this time when it even affects other apps that share the same database too.

Initially, I wanted to set a trigger that auto deletes the rows from that table. But considering that it inserts like few hundreds rows (containing really large stack traces 🤮) per second, that means my trigger would be executed the same amount of time per second, which could be quite dangerous, like concurrent, overlapping execution and depleting my IOPS quota. Then....I remembered the quote earlier in the article and ask Claude if there's something like `/dev/null` in Postgres. To my surprise, there is. So, here we are. 