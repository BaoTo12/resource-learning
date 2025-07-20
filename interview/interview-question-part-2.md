## Introduction of B-Tree

==A B-Tree is a specialized m-way tree designed== to optimize data access, especially on disk-based storage systems.
- In a B-Tree of order m, each node can have up to m children and m-1 keys, allowing it to efficiently manage large datasets.
- The value of m is decided based on disk block and key sizes.
- One of the standout features of a B-Tree is its ability to store a significant number of keys within a single node, including large key values. It significantly reduces the tree’s height, hence reducing costly disk operations.
- B Trees allow faster data retrieval and updates, making them an ideal choice for systems requiring efficient and scalable data management. By maintaining a balanced structure at all times,
- B-Trees deliver consistent and efficient performance for critical operations such as search, insertion, and deletion.

> **What is m-way Tree**?

The m-way search trees are multi-way trees which are generalised versions of binary trees where each node contains multiple elements. In an m-Way tree of order m, each node contains a maximum of m - 1 elements and m children.
![alt text](./images/m-way-tree.png)

The goal of m-Way search tree of height h calls for O(h) no. of accesses for an insert/delete/retrieval operation. Hence, it ensures that the height h is close to log_m(n + 1).
The number of elements in an m-Way search tree of height h ranges from a minimum of h to a maximum of m^h^ - 1

A B-Tree is a specialized m-way tree designed to optimize data access, especially on disk-based storage systems.

___

**Properties of a B-Tree**
A B Tree of order m can be defined as an m-way search tree which satisfies the following properties:
All leaf nodes of a B tree are at the same level, i.e. they have the same depth (height of the tree).
The keys of each node of a B tree (in case of multiple keys), should be stored in the ascending order.
In a B tree, all non-leaf nodes (except root node) should have at least m/2 children.
All nodes (except root node) should have at least m/2 - 1 keys.
If the root node is a leaf node (only node in the tree), then it will have no children and will have at least one key. If the root node is a non-leaf node, then it will have at least 2 children and at least one key.
A non-leaf node with n-1 key values should have n non NULL children.
> **Interesting Facts about B-Tree**
> The minimum height of the B-Tree that can exist with n number of nodes and m is the maximum number of children of a node can have is: **h~min~ = [log~m~(n + 1)] - 1**
> The maximum height of the B-Tree that can exist with n number of nodes and t is the minimum number of children that a non-root node can have is: **h~max~ = [log~t~(n + 1)/2 ] and t = [m / 2]**
> 
>>
**Need of a B-Tree**






## Introduction of B+tree


## What is a Distributed Lock?
A distributed lock is a mechanism that allows coordinated access to shared resources in a distributed environment, ensuring that only one process can access a particular resource at a time


## Why use Redis?

Redis (**RE**mote **DI**ctionary **S**erver) is an open source, in-memory, NoSQL key/value store that is used primarily as an application cache or quick-response database.

**Benefits**
- One of the main advantages of using Redis for caching is its fast read and write speeds. Redis can handle millions of operations per second, which allows it to serve webpages faster than traditional databases.
- It also offers excellent support for transactions, allowing applications to perform multiple operations atomically. Additionally, Redis supports the use of pub/sub channels for fast data sharing between applications.
- Redis is also highly scalable and can be deployed across multiple machines for high availability
-  Data Types: Redis supports a richer variety of data types including key-value pairs, lists, connections, and streams, allowing for more diverse use cases


**Drawbacks**
-  It stores data entirely in memory, which means that it can be sensitive to data loss in the event of a crash or shutdown. To address this issue, Redis provides features such as persistence and replication. But this add complexity overhead
- Another drawback of Redis is that it is a single-threaded system, which means that it can only process one command at a time. This can limit the performance and scalability of Redis in applications that require high concurrency and parallelism. To address this issue, Redis provides clustering and sharding features that allow data to be distributed across multiple servers, but these features can be complex to set up and manage.


## What are the common data structures in Redis?
**Strings (Key-Value):**
- This is the most basic data structure, storing a simple mapping between a key and a string value.
- Common commands include GET and SET.
- Applications: caching general data, implementing counters (e.g., number of likes), or storing small blocks of data.

**Hashes:**
- A hash is a map between fields and string values, ideal for representing objects.
- It allows direct modification of specific fields within an object.
- Common commands include GET and GETALL.
- Applications: storing user information (e.g., user ID, name, email as fields), or product details.

**Lists:**
- Lists are ordered collections of strings, functioning like linked lists.
- They are versatile and have many application scenarios.
- Common commands include LPUSH, RPUSH, LPOP, RPOP, and LRANGE.
- Applications: Implementing Facebook's follower/following lists or activity feeds (like Instagram's feed) where order matters.

**Sets:**
- Sets are unordered collections of unique strings.
- They are useful for storing distinct items and performing set operations.
- Common commands include SADD, SMEMBERS, and SUNION.
- Applications: Storing all followers for a user (e.g., on Instagram) or finding common interests between different groups by using set intersection operations.

**Sorted Sets (ZSETs):**
- Similar to Sets, but each member is associated with a score, which is a floating-point number. This score allows the elements to be ordered.
- Applications: Building leaderboards (e.g., gift rankings on Momo), managing delayed messages, or maintaining lists of online users ordered by their last activity



## How to set an expiration time for a key in Redis?
Redis provides mechanisms to automatically manage data lifecycle by setting expiration times for keys, which helps conserve memory resources by removing data that is no longer needed. Memory is a finite resource that cannot be expanded indefinitely, making efficient management crucial.
- **Expiration Mechanism**: Redis intelligently cleans its environment by deleting expired data. For example, a popular news article initially stored in Redis might be automatically deleted once its relevance fades after a few months.
- **Deletion Methods**:
    + **Periodic Deletion**: By default, Redis periodically selects a random subset of expired keys and deletes them, typically every 100 milliseconds.
    + **Lazy Deletion (Expiration upon Access)**: When a client attempts to access an expired key, Redis will delete it at that moment before returning an empty response.
    + **Manual/Scheduled Bulk Deletion**: The speaker suggests running a command to delete all expired keys during off-peak hours, such as 12 AM when user traffic is low, to avoid performance issues during peak times.
- **Criticality of Deletion Timing**: It's crucial to avoid deleting cached data during periods of high user traffic. If the cache is cleared when many users are active, their requests will be directed to the database, potentially overwhelming it and leading to system slowdowns or failures







