
Short post. 

Quite some time ago, I remember reading about "how fast is your database" or something along that line. Well, I do have a super fast database that can write 1 billion+ (or more!) insert per second, probably the fastest in the world, no joke. But, there's a catch, it won't persist anything. Yeah just pipe that write to `/dev/null` and you have virtually unbounded write speed. Pretty cool, huh?

Today, I figured out where THAT might be actually sometime that I need. AND, for some reason, PostgreSQL (and apparently MySQL too) does support it.

```sql
CREATE RULE make_write_faster_rule AS 
	ON INSERT TO your_super_important_table DO INSTEAD NOTHING;
```

To my limited understanding, `DO NOTHING` here is the keyword usually being used together with `ON CONFLICT`. But what's kinda crazy to think about is, that this code path even available. I mean, this SQL is parsed and executed, meaning that this combination of execution path must be explicitly made, not something accidental. And now, is one of the day where people like me could use it.

Ok, the less interesting part. What do I use it for? Well, we do host an opensource application at my company. And, because of my personal preference, I deliberately choose to use multi-tenant database, aka same postgres instance, shared across many apps (still with a proper access control of course). But