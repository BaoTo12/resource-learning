### Database Sharding
A technique that splits a large database into smaller parts (shards) to distribute data across multiple servers.
*Note*: Use it when a single database cannot handle the read/write load or data volume.
![database-sharding](./images/database-sharding.png)

#### You need to know
- **Shard key**: Choosing the right shard key (e.g., user ID, region) affects data distribution and query efficiency. A bad choice can cause uneven load (a "hot shard").

- **Cross-shard** operations are costly: Joins, transactions, or queries across shards are complex and slow - design to avoid them.

- **Rebalancing** is hard: Moving data between shards when traffic grows unevenly or hardware changes is tricky and can require downtime or complex tooling.
