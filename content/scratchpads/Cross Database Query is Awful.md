

Apart from title, I will rant about Redshift in general. Some may be not factually correct, given my limited experience in data engineering, but it's the experience that I got first hand in this field, take it with a grain of 🧂(Himalayan salt)


## Federated Query, No ETL, Win?

Federated query sounds good in theory? Right? No data movement, query data as you need, no need expensive and complicated ETL, right??????

Wrong. There's a reason why ETL / data movement might be something you called a necessary evil in data engineering. 

The recent Databrick's LTAP shows that... you can do analytics without data movement??
https://www.databricks.com/blog/lakebase-ltap-rethinking-database-storage
![[Pasted image 20260724171323.png]]
Uh oh,  you don't cobble bunch or systems / traditional CDC/ETL anymore, but strictly speaking, that data still moves around systems. Obviously this is way better, but still, data movement is unavoidable. If you want to do fast analytics, you need to transpose your row based data to columnar based. It's physical constraint that you cannot avoid.

Anyway, back to the topic. Why, at least on redshift-to-postgres, federated query sucks? Okay hear me out. I understand query optimization is one of the hardest topic in database system. BUT, sometimes, when I see dead simple query, like this:
```sql
select *
from event
join event_type on event.type_id = event_type.id
where event_type.name in ('opened', 'closed')
```

 I always thought "this filter should be easily pushed down to the data source, right?? It's a simple join with clear predicate filter"
 
 Nope.  Redshift never pushes down query with join. It WILL PULL ALL THE F* DATA across the network, and do local filtering (can either local ssd filter, or worse, remote storage / RMS filter, way slower). If you gotta pull these hundreds million event table, uh, good luck. 

You will ended up having to do something like this 
```sql
select *
from event
where type_id = (101, 206, ...)
```
Do this bespoke optimization for other hundreds of your raw table models. Good luck with that.


