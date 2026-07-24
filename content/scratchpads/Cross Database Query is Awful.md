

Apart from title, I will rant about Redshift in general. Some may be not factually correct, given my limited experience in data engineering, but it's the experience that I got first hand in this field, take it with a grain of 🧂


## Federated Query, No ETL, Win?

Federated query sounds good in theory? Right? No data movement, query data as you need, no need expensive and complicated ETL, right??????

Wrong. There's a reason why ETL / data movement might be something you called a necessary evil in data engineering. 

The recent Databrick's LTAP shows that... you can do analytics without data movement??
https://www.databricks.com/blog/lakebase-ltap-rethinking-database-storage
![[Pasted image 20260724171323.png]]
Uh oh, yeah you don't cobble bunch or systems / traditional CDC/ETL anymore, but strictly speaking, that data still moves around systems. Obviously this is way better, but still, data movement is unavoidable. If you want to do fast analytics, you need to transpose your row based data to columnar based. It's physical constraint that you cannot avoid.

Anyway, back to the topic. Why, at least on redshift-to-postgres, federated query sucks? Okay hear me out. I understand query optimization is one of the hardest topic in database system. BUT, sometimes, when I see dead simple query, I always thought "this filter should be easily pushed down to the data source, right??". 

Like


