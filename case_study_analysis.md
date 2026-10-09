## **How Discord Stores Billions of Messages**

#### January 13, 2017

----

### **Introduction**

- Discord grows faster and so does the user-generated context: 
    - July (2015): Discord registered 40 million messages / day
    - December (2015): Discord registered 100 million messages / day
    - January (2017 - as of this blog): 120 million messages / day

- Early decision made by Discord: Store all chat history forever
    - Data increases in velocity, size and must remain available -> Solution : **Cassandra**


----

### **What we were doing**

**Original version of discord:** 

- Built in only two months in 2015
- Thoughts when building the first version: **one of the best databases for iterating quickly is MongoDB**

```txt
**one of the best databases for iterating quickly is MongoDB** -> add our explanation of why @gjoliveira
```

- Everything on Discord was stored in a single Mongo DB replica. However, it always existed the plan of migrating to a new database - Why?

    - Developers knew MongoDB was known to be complicated to use and not known for stability

                                ↓

        **Company culture:** build quickly to prove oyt a product feature, but always with a path to a more robust solution


**First Approach:**
- Messages were stored in a Mongo DB collection, with a single compound index on `channel_id` ad `created_at`.



**Around November 2015:**
- Reached 100 million stored messages -> Started to see expected issues: 

    - the data and the index could no longer fit in RAM  
    - latencies started to become unpredictable

- It was time to migrate to a database more suitable to this task


### **Choosing the Right Database**

Before choosing the new database, Discord tried to understand the read/write patterns and the causes of the problems with the current solution (MongoDB). 

**Interesting findings:**

1. Discord's reads were extremely random

2. Read/write ratio was about 50/50

3. Voice chat heavy Discord servers send almost no messages -> send one or two messages evry few days

    - this kind of server is unlikely to reach 1000 messages
    
    - **problem:** even though this is a small amount of messages, it makes it harder to serve this data to the users - *why?*: 
        
        - **few messages != less complexity:** just returning 50 messages to a user can result in many random seeks on disk causing disk cache evictions


4. Private text chat heavy Discord servers send a decent number of messages (easy to reach 100 thousand - 1 million messages a year)

    - data requested here is usually very recent -> this is not the problem for this kind of servers

    - **actual problem:** since these servers usually have under 100 members, the **rate at which this data is requested** is **low** and **unlikely to be in disk cache**


5. Large public Discord servers send a lot of messages --> servers with thousands of members sending thousands of messages a day (millions of messages a year, etc...)
    - users are almost always requesting messages sent in the last hour and requesting them very often -> data is usually in the disk cache


6. Desire for new features in the coming year -> these would add more ways for users to issue random reads: 

    - ability to view your mentions for the last 30 days, then jump to that point in history;
    - viewing plus jumping to pinned messages; 
    - full-text search


**Requirements definition:**
- **Linear scalability** - the goal is to not reconsider the solutions later or manually re-shard (reorganize db shards) the data

- **Automatic failover** - Discord automatically switches to a backup system when a failure occurs (=failover), keeping the service running with minimal human intervention

- **Low maintenance** - the system should work with minimal maintenance after setup. More data should only mean adding more nodes

- **Proven to work** - reliable and tested technology (not interested in new / experimental)

- **Predictable performance** - systems should respond quickly and consistently 

    - expected that 95% of API response times < 80 ms
    - desired to avoid relying on Redis or Memcached to cache messages

- **Not a blob store** - system should store messages individually, rather than requiring large blocks of data to be repeatedly read, decoded and updated when new messages are added

- **Open source** - source code publicly available

    - allows Discord to control and manage its own system without depending entirely on a third-party company


> **Cassandra** -> only database that satisfied all these requiremets -> allows: 
> 1. add nodes to scale it 
> 2. tolerate a loss of nodes without any impact on the application

> - Related data is stored contigously on disk providing minimum seeks and easy distribution around the cluster 
> - Backed by DataStax


----

### **Data Modeling**

**Cassadra** -> **KKV store**: 
- Two K's comprise the primary key: 
    - First K: **Partition Key** -> used to determine which node the data lives on and where it is found on disk

        - Partition contains multiple rows within it; 
        - A row within the partition is identified by the second K
    
    - Second K: **Clustering Key** -> identifies each row within a partition and determines the order in which the rows are stored



Remembering previous model: 

- Messages were indexed in MongoDB using `channel_id` and `created_at`. 

- `channel_id` -> became the **partition key** -> messages from the same channel are stored in the same partition - *why?* because queries usually search for messages within a specific channel

- `created_at` as clustering key? -> No, two messages can have the same creation time
    - every ID on Discord is a Snowflake (=chronologically sorted) -> `message_id` became the **clustering key**
        - each message has a unique ID (Snowflake) which contains time information and allows messages to be ordered chronologically


**primary key** -> **(channel_id, message_id)** -> `channel_id` tells Cassandra where the messages are stored, while `message_id` identifies and orders messages within that channel


#### Example of a simplified schema for the messages table

```sql
CREATE TABLE messages ( 
    channel_id bigint,
    message_id bigint,
    author_id bigint,
    content text,
    PRIMARY KEY (chanel_id, message_id)
) with CLUSTERING ORDER BY (message_id DESC);
```


- Cassandra has schemas (that defines the data structure and columns of the tables, and like relational db), however: 

    - they are cheap to alter 
    - do not impose temporary performance impact



**Problem in the process:**
- Cassandra can support partitions up to 2GB, but that does not mean that having very large partitions is a good idea

- When Discord started importing existing messages into Cassandra, it received warnings because some partitions were already larger than 100MB

- ***Why are large partitions a problem?***

    - **More memory pressure:** large partitions put more pressure on Cassandra's garbage collector (GC)

    - **Harder to distribute data:** a partition is stored on a particular part of the cluster, so Cassandra cannot split a single partition across multiple nodes

    - **Continuous growth:** a Discord channel can exist for years, meaning its partition could keep growing as new messages are added

- Discord realised that it needed to **limit the size of each partition** to avoid performance problems as channels continued to grow


**Decision:** Bucket the messages by time

- They looked at the largest channels and determined if they stored about 10 days of messages within a bucket that they could comfortably stay under 100MB

- Buckets had to be derivable from the `message_id` or a `timestamp`

```python
DISCORD_EPOCH = 1420070400000
BUCKET_SIZE = 1000 * 60 * 60 * 24 * 10


def make_bucket(snowflake):
   if snowflake is None:
       timestamp = int(time.time() * 1000) - DISCORD_EPOCH
   else:
       # When a Snowflake is created it contains the number of
       # seconds since the DISCORD_EPOCH.
       timestamp = snowflake_id >> 22
   return int(timestamp / BUCKET_SIZE)
  
  
def make_buckets(start_id, end_id=None):
   return range(make_bucket(start_id), make_bucket(end_id) + 1)
view raw
```


- Cassandra partition keys can be compounded, so the new primary key became **((channel_id, bucket), message_id)**

```sql
CREATE TABLE messages ( 
    channel_id bigint,
    bucket int,
    message_id bigint,
    author_id bigint,
    content text,
    PRIMARY KEY ((channel_id, bucket), message_id),
) WITH CLUSTERING ORDER BY (message_id DESC);
```

- To query for recent messages in the channel, they generate a bucket range from current time to `channel_id`

- Then, they sequentially query partitions until enough messages are collected

    - **Downside:** rarely active Discords will have to query multiple buckets to collect enough messages over time

        - In practice -> this has proved to be fine because for active Discords enough messages are usually found in the first partition and they are the majority

----

### **Dark Launch**

- Setup the code to double read/write to MongoDB and Cassandra before introducing the new system into production

- Got an unexpected bug -> `author_id` was null, which was impossible, because this is a required field

----

### **Eventual Consistency**

