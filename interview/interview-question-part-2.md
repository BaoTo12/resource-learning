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
- ==Applications==: Dùng để lưu text, number, email or serialized objects. String can save up to 512MB, lý do là vì việc triển khai trong String là **SDS(String Dynamic Simple)** là một chuỗi có thể thay đổi tại runtime, và có cấu trúc/format đơn giản. SDS có vài loại mã hóa như sau
    + **Embedded String** chiếm 64 Bytes dung lượng và lưu <= 44 Bytes dữ liệu
    + **Raw String** lưu > 44 bytes
    + **int** ?? chắc là lưu string theo kiểu mã hóa URL64

**Hashes:**
- A hash is a map between fields and string values, ideal for representing objects.
- It allows direct modification of specific fields within an object.
- Common commands include GET and GETALL.
- Applications: storing user information (e.g., user ID, name, email as fields), or product details.

**Lists:**
- Lists are ordered collections of strings, functioning like doubled-linked lists.
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



## Transaction in Redis
Redis Transactions allow the execution of a group of commands in a single step, they are centered around the commands MULTI, EXEC, DISCARD and WATCH. Redis Transactions make two important guarantees:

- All the commands in a transaction are serialized and executed sequentially. A request sent by another client will never be served in the middle of the execution of a Redis Transaction. This guarantees that the commands are executed as a single isolated operation.

- The EXEC command triggers the execution of all the commands in the transaction, so if a client loses the connection to the server in the context of a transaction before calling the EXEC command none of the operations are performed, instead if the EXEC command is called, all the operations are performed. When using the append-only file Redis makes sure to use a single write(2) syscall to write the transaction on disk. However if the Redis server crashes or is killed by the system administrator in some hard way it is possible that only a partial number of operations are registered. Redis will detect this condition at restart, and will exit with an error. Using the redis-check-aof tool it is possible to fix the append only file that will remove the partial transaction so that the server can start again.


### Errors inside a transaction 
- A command may fail to be queued, so there may be an error ==before calling EXEC==
- A command may fail ==after EXEC is called==. Errors happening after EXEC instead are not handled in a special way: all the other commands will be executed even if some command fails during the transaction.
--> Starting with Redis 2.6.5, the server will detect an error during the accumulation of commands. It will then refuse to execute the transaction returning an error during EXEC, discarding the transaction.

### Rollback in Redis

- Redis does not support rollbacks of transactions since supporting rollbacks would have a significant impact on the simplicity and performance of Redis.

### Optimistic locking using check-and-set

- ==WATCH== is used to provide a check-and-set (CAS) behavior to Redis transactions.
- WATCHed keys are monitored in order to detect changes against them. If at least one watched key is modified before the EXEC command, the whole transaction aborts, and EXEC returns a Null reply to notify that the transaction failed.
- So what is WATCH really about? It is a command that will make the EXEC conditional: we are asking Redis to perform the transaction only if none of the WATCHed keys were modified

### Summary 
Transaction is a group of command that starts with "MULTI" command and ends with "EXEC" command 
Transactions in Redis have the following properties
* Redis doesn't have rollback mechanism like other databases. And Redis solves this problem by WATCH command, it is like optimistic lock with version checking
*  All commands in a transaction will be queued and performed sequentially 
* During the process of performing a transaction if a command is failed to execute, other commands will be performed normally
* Command Queueing Error (before EXEC)	❌ Entire transaction is discarded. Redis refuses to run EXEC.	Syntax error, wrong arity (e.g., SET key without a value).












