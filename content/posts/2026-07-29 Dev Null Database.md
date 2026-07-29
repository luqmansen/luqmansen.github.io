
Short post. 

Quite some time ago, I remember reading about "how fast is your database" or something along that line. Well, I do have a super fast database that can write 1 billion+ (or more!) insert per second, probably the fastest in the world, no joke. But, there's a catch, it won't persist anything. Yeah just pipe that write to `/dev/null` and you have virtually unbounded write speed. Pretty cool, huh?

Today, I figured out where THAT might be actually sometime that I need. AND, for some reason, PostgreSQL (and apparently MySQL too) does support it.

```sql
CREATE RULE your_important_table AS 
	ON INSERT TO your_important_table DO INSTEAD NOTHING;
```

I haven't investigated wh