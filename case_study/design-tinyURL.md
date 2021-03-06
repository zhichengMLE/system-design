# Design Tiny URL

> Ref: https://www.educative.io/collection/5668639101419520/5649050225344512

## 1. Why Tiny URL?

Saving space

> Origin: https://github.com/zhichengMLE/system-design/new/master
>
> TinyURL: https://tiny.com/abcdefg

## 2. Requirements

### 2.1. Functional Requirement

- Convert origin URL to tiny URL [len(tiny URL) <= len(origin URL)]
- Convert back Tiny URL to origin URL
- User has expiration time (user could specify)
- User check all tiny URL assgin to him
- User could cutomize the tiny URL address (fixed length ?!)
- User could analyse the accessment of tiny URL
- User could set private or public for assigned tiny URL

### 2.2. Non-Function Requirement

- High availability, otherwise, cannot redirect -> cannot access -> user will no longer access the site.
- Low latency, otherwise, redirect takes long time -> user will no longer access the site.
- High concurrency, otherwise, if many requests, redirect takes long time -> user will no longer access the site.
- Convertion of tiny URL to origin URL should not be predictable.

## 3. Capability

Assumption: 
- Generate 500M new URL per month
- Url redirect request 50 B per month
- Each data rows takes 500 bytes

### 3.1. Traffic (per second)

New URL per second: 

    500M / 30 / 24 / 3600 ~= 200 URLs/s


URL redirect request per second: 

    50B / 30/ 24 / 3600 ~= 20K URLs/s

### 3.2. Bandwidth (per second)

New URL per second:
    
    200 * 500 bytes / 1000 = 100 KB/s

URL redirect request per second: 
    
    20K * 500 bytes / 1000 / 1000 = 10 MB/s


### 3.3. Storage (5 years)

Data rows in 5 years: 

    500M * 12 * 5 * 500 bytes / 1000 / 1000 / 1000 / 1000 = 15TB


### 3.4. Memory

80:20 rules -- cache 20% of hot URLs

Total URLs per day:

    20K * 24 * 3600 / 1000 ~= 1.7B

Storage of 20% of URls cache:

    1.7B * 20% * 500 bytes / 1000 / 1000 / 1000 = 170GB 


### 3.5. Summary

- New URL request : 200 URLs/s
- URL redirect request: 20K URLs/s
- incoming data: 100 KB/s
- outgoing data: 10 MB/s
- Storage for 5 years: 15 TB
- Memory fo cache: 170 GB

## 4. System Design

Basic system diagram:

![1](https://github.com/zhichengMLE/system-design/blob/master/_figs/tinyurl_1.jpg?raw=true)

Problems:
1. Concurrency may causes same keys get assigned.
2. High latency: No cache.
3. Database getting larger and larger -> eventually full.

## 5. Adding Key Generation Server


![2](https://github.com/zhichengMLE/system-design/blob/master/_figs/tinyurl_2.jpg?raw=true)


### 5.1. Main Strategy.
As problem #1 described, adding key generation server to reduce the chance of this problem. There are two tables: unused keys table, used keys table. Once any key is assigned, move that to used key table. If query key is assgined, generate another key and query again (loop, until find the key unused).

### 5.2. Increase Key Amount

Use base64 encoding to increase the number of keys assigned. If the key contains 6 letters, then the total number of keys is:

    64 * 64 * 64 * 64 * 64 * 64 = 64 ** 6 = 68.7B

### 5.3. Improve Key Generated Speed

1. Add cache to save some unused keys (e.g. queue), once request comes, just return the unused key and remove that from the queue. Finally, update DB.
2. Pre-assigned (DB had marked these key as used) some keys to the cache (e.g. queue), once request comes. just return unused key and remove that from the queue.

   - Pros: reduce the traffic of DB
   - Cons: once the cache is down, all the keys are lost.

3. Pre-assigned (DB had marked those keys as used) some keys to the application server (e.g. queue), once request is made by user, just do everything in the application server.

    - Pros: reduce the traffic of DB and key generation server.
    - Cons: once the application server is down, corresponding keys are lost.

### 5.4. Improve availability


![3](https://github.com/zhichengMLE/system-design/blob/master/_figs/tinyurl_3.jpg?raw=true)


## 6. Database partitioning and replication

### 6.1. Partitioning

1. Range based partitioning:
   - Pros: Simple to implement
   - Cons: Imbalanced, for example, a lot of key start with "A", but only few start with "E".
2. Hashed based partitioning (consist hashing):
   - Pros: Balanced usage.

### 6.2. Replication

1. Transactional Replication 
2. Snapshot Replication 
3. Merge Replication

## 7. Cache

![4](https://github.com/zhichengMLE/system-design/blob/master/_figs/tinyurl_4.jpg?raw=true)

80:20 rules of cache. Since we only keep 20% of the most recently used URL, so we dicard least recently used URL. Use Linked Hash Map as data structure.

## 8. Load Balancing

Algorithm:

Round Robin:
- Pros: Equally assigned
- Cons: Cannot know about the server with heave load

Wighted Round Robin:
- Pros: Assigned traffic based on the weight of server
- Cons:  Cannot know about the server with heave loa

Periodically query the load of each server, adjust the weight based on loads.

## 9. APIs Design

For example:

    CreateURL(apiKey, originalUrl, customizeUrl=None, alias=None, user=None, isPrivate=False, expireTime=3600)

## 10. Database Design

Users:
- ID (PK): varchar(16)
- Name: varchar(20)
- Email: varchar(32)
- CreateAt: datatime
- LastLogin: datetime

URLs:

- HashID (PK): varchar(16)
- OriginalUrl: varchar(500)
- CreatedAt: datetime
- ExpireTime: datetime
- IsPrivate: boolean
- UserID: varchar(20)

## 11. Others

### 11.1. Security

We can have statistics about the country of the visitor, date and time of access, web page that refers the click, browser or platform from where the page was accessed and more.


### 11.2. Analysis URL

We can store isPrivate with each URL in the database.

### 11.3. Cleanup expired URL

Cleanup every week.

![5](https://github.com/zhichengMLE/system-design/blob/master/_figs/tinyurl_5.jpg?raw=true)