## **How Discord Stores Billions of Messages**

#### January 13, 2017

### **Introduction**

- Discord grows faster and so does the user-generated context: 
    - July (2015): Discord registered 40 million messages / day
    - December (2015): Discord registered 100 million messages / day
    - January (2017 - as of this blog): 120 million messages / day

- Early decision made by Discord: Store all chat history forever
    - Data increases in velocity, size and must remain available -> Solution : **Cassandra**


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



### **Data Modeling**

