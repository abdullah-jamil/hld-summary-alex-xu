# System Design Interview: Final Master Revision Guide

## Purpose

This is the final interview oriented revision guide for **System Design
Interview: An Insiders Guide by Alex Xu**.

The goal is not to reproduce the book. The goal is to compress the
reusable engineering ideas, patterns, tradeoffs, formulas, and interview
talking points into one document that can be reviewed before an
interview.

------------------------------------------------------------------------

# 0. The One Mental Model to Remember

Almost every large system can be reasoned about through the same chain:

**Requirements -\> Scale -\> APIs -\> Data model -\> Core architecture
-\> Bottleneck -\> Scale strategy -\> Reliability -\> Consistency -\>
Failure handling -\> Tradeoffs**

When stuck, ask:

1.  What exactly must the system do?
2.  How much traffic and data are we handling?
3.  What is the read/write ratio?
4.  What needs low latency?
5.  What data must be strongly consistent?
6.  What can be eventually consistent?
7.  What can be cached?
8.  What can be processed asynchronously?
9.  Where is the bottleneck?
10. What happens when each important component fails?
11. How do we scale the bottleneck?
12. What tradeoff are we making?

------------------------------------------------------------------------

# 1. Scale From Zero to Millions of Users

## Core idea

Start simple and evolve the architecture only when scale requires it.

A typical evolution is:

**Client -\> DNS -\> Load Balancer -\> Stateless Web/API Servers -\>
Database**

Then progressively add:

**Cache -\> CDN -\> Replication -\> Sharding -\> Message Queue -\>
Dedicated Services -\> Multiple Data Centers**

## Stateless web tier

A stateless server does not depend on local server memory to remember a
user between requests.

Move session or persistent state to shared storage such as a database or
distributed cache.

Why?

-   Any request can go to any server.
-   Horizontal scaling becomes easy.
-   Failed servers do not lose session state.
-   Load balancing becomes simpler.
-   Rolling deployments become easier.

Avoid sticky sessions unless there is a strong reason to use them.

## Load balancer

Responsibilities:

-   Distribute traffic.
-   Detect unhealthy servers.
-   Remove failed servers from rotation.
-   Allow horizontal scaling.

Common patterns:

-   Round robin
-   Weighted routing
-   Least connections
-   Health checks

## Database scaling

### Vertical scaling

Add CPU, RAM, storage, or a more powerful machine.

Advantages:

-   Simple.
-   Minimal application changes.

Problems:

-   Hardware ceiling.
-   Expensive.
-   Large single point of failure.
-   Does not scale indefinitely.

### Horizontal scaling

Add more database machines.

The main technique is **sharding**.

A shard owns only part of the dataset.

Example:

`shard = hash(user_id) % N`

Important tradeoff:

-   Scaling reads can often use replicas.
-   Scaling writes and total data volume usually requires sharding.

## Cache

Use a cache when:

-   Reads dominate writes.
-   Data is expensive to compute.
-   Data is frequently requested.
-   Slight staleness is acceptable.

Typical flow:

**Client -\> API -\> Cache -\> Database on miss**

Important questions:

-   What is the cache key?
-   What is the TTL?
-   What happens on cache miss?
-   How is invalidation handled?
-   What happens if the cache fails?
-   Is stale data acceptable?

## CDN

Use a CDN for geographically distributed static or cacheable content:

-   Images
-   CSS
-   JavaScript
-   Videos
-   Other large immutable assets

CDN improves latency by serving content closer to users and reduces
origin load.

Important CDN concerns:

-   Cache expiry.
-   Invalidation.
-   Cost.
-   Origin fallback.

## Reliability

The book distinguishes:

-   **Fault**: a component deviates from its specification.
-   **Failure**: the system as a whole stops providing the required
    service.

The goal is not to eliminate every fault. The goal is to prevent
individual faults from becoming system failures.

Build redundancy at every important layer.

## Chapter 1 interview summary

When scaling from a small application:

1.  Keep API servers stateless.
2.  Put a load balancer in front.
3.  Add cache for hot data.
4.  Put static content behind a CDN.
5.  Replicate databases for availability and read scaling.
6.  Shard when one database cannot handle the data or write load.
7.  Use queues for asynchronous work.
8.  Split overloaded responsibilities into services.
9.  Add multiple data centers when required.
10. Monitor everything.

------------------------------------------------------------------------

# 2. Back-of-the-Envelope Estimation

## Core idea

Estimation is not about exact numbers. It is about discovering the order
of magnitude and identifying architectural consequences.

Always state assumptions.

## Important powers of two

-   1 KB = 2\^10 bytes
-   1 MB = 2\^20 bytes
-   1 GB = 2\^30 bytes
-   1 TB = 2\^40 bytes
-   1 PB = 2\^50 bytes

For rough interviews, decimal approximations are also acceptable if used
consistently.

## QPS

Basic formula:

`QPS = requests per day / 86,400`

Then estimate peak:

`Peak QPS = Average QPS x peak factor`

A factor such as 2x is only an assumption. Say so explicitly.

## Storage

`Storage = objects per day x average object size x retention period`

Then add overhead for:

-   Replicas
-   Indexes
-   Metadata
-   Compression assumptions
-   Growth

## Bandwidth

`Bandwidth = requests per second x average payload size`

For media:

`Bandwidth = users x consumption rate x average content size`

## Read/write ratio

Always ask:

-   How many reads?
-   How many writes?
-   Is traffic uniform?
-   Are there bursts?
-   Are there hot keys?

## Latency

Understand the difference:

-   **Latency**: time spent waiting before or during service processing
    depending on context.
-   **Response time**: what the client experiences, including network
    and queueing delays.

For interactive systems, tail latency matters.

Think about:

-   p50
-   p95
-   p99

A system with good average latency can still feel slow if p99 is
terrible.

## Interview rule

Do not spend five minutes making perfect arithmetic.

The useful output is:

**scale -\> bottleneck -\> architecture**

Example:

If the system needs tens of thousands of writes per second, one database
may be insufficient.

If it needs millions of reads per second, caching and replication become
important.

If objects are hundreds of MB, object storage and CDN become more
appropriate than database BLOB storage.

------------------------------------------------------------------------

# 3. Framework for System Design Interviews

## The four step framework

### Step 1: Understand the problem and establish scope

Do not jump into architecture.

Clarify:

-   Users
-   Clients
-   Core features
-   Read/write operations
-   Scale
-   Latency requirements
-   Consistency requirements
-   Availability requirements
-   Data retention
-   Geographic scope
-   Security constraints

Separate:

**Functional requirements** - What the system does.

**Non-functional requirements** - How well it must do it.

## Step 2: Back-of-the-envelope estimation

Estimate:

-   DAU
-   Requests per user
-   QPS
-   Peak QPS
-   Storage
-   Bandwidth
-   Cache size

Explain assumptions.

## Step 3: High-level design

Draw the major boxes first.

Typical components:

**Client -\> Load Balancer -\> Stateless API -\> Cache -\> Database**

Then add:

-   Queue
-   Workers
-   Object storage
-   Search index
-   CDN
-   Notification service
-   Analytics pipeline

Explain the main request flows.

Get interviewer agreement before diving deeper.

## Step 4: Deep dive

Do not discuss every component.

Choose the bottleneck or the component most related to the requirements.

Examples:

-   News feed -\> fanout
-   Chat -\> WebSocket and message ordering
-   URL shortener -\> ID generation and collision handling
-   Autocomplete -\> trie
-   Video -\> transcoding
-   Google Drive -\> synchronization and block storage
-   Rate limiter -\> distributed atomicity

## Wrap up

Discuss:

-   Failure handling
-   Bottlenecks
-   Scaling
-   Monitoring
-   Security
-   Tradeoffs
-   Future improvements

## What interviewers actually evaluate

They care about:

-   Clarifying ambiguity.
-   Structured thinking.
-   Communication.
-   Reasoning about tradeoffs.
-   Finding bottlenecks.
-   Handling failures.
-   Scaling intelligently.

There is no single perfect architecture.

**Do not over-engineer before the requirements justify it.**

------------------------------------------------------------------------

# 4. Design a Rate Limiter

## Why

A rate limiter protects services from:

-   Abuse
-   DDoS-like application traffic
-   Accidental overload
-   Expensive clients
-   Unfair resource consumption

It can be implemented at:

-   API gateway
-   Reverse proxy
-   Application layer
-   Network layer

## Common algorithms

### Token bucket

A bucket contains tokens.

-   Tokens are added at a fixed rate.
-   Each request consumes one or more tokens.
-   If there are no tokens, reject or delay the request.
-   Bucket capacity allows controlled bursts.

Good when:

-   You want to allow bursts.
-   You want a simple and flexible limiter.

### Leaky bucket

Requests enter a bounded queue.

Requests leave at a fixed rate.

Good when:

-   Stable output rate matters.

Problem:

-   A burst can fill the queue.
-   Old requests may delay newer requests.

### Fixed window

Divide time into fixed intervals.

Example:

100 requests per minute.

Problem:

A client can send 100 requests at the end of one window and 100 at the
start of the next.

### Sliding window log

Store timestamps of requests.

For each request, remove timestamps outside the window and count the
rest.

Accurate but memory expensive.

### Sliding window counter

Combine adjacent fixed windows using weighted counts.

Good compromise between:

-   Accuracy
-   Memory
-   Performance

## Distributed rate limiter

A local counter is insufficient when many application servers exist.

Use a shared store such as Redis.

Critical problem:

**Race condition**

Bad:

1.  Read counter.
2.  Check counter.
3.  Increment.
4.  Write counter.

Two requests can read the same value concurrently.

Better:

-   Atomic Redis operations.
-   Lua scripts.
-   Redis sorted sets for timestamp based algorithms.

## Synchronization

If multiple rate limiter servers exist, they need shared state or
deterministic partitioning.

## Important interview questions

-   Rate limit by user, IP, API key, or endpoint?
-   Global or per-region?
-   Hard or soft limit?
-   What happens after rejection?
-   What HTTP status is returned?
-   Should clients retry?
-   How do you prevent synchronized retries?

Use exponential backoff with jitter for clients.

------------------------------------------------------------------------

# 5. Design Consistent Hashing

## Problem with normal hashing

`hash(key) % N`

If N changes, many keys move.

Adding or removing one server can remap a huge portion of the dataset.

## Consistent hashing

Imagine a hash ring.

-   Hash servers onto the ring.
-   Hash keys onto the ring.
-   A key belongs to the first server encountered clockwise.

When a server is added or removed, only a portion of keys move.

## Virtual nodes

A single physical server gets many positions on the ring.

Benefits:

-   Better load distribution.
-   Less skew.
-   Smoother rebalancing.
-   Better handling of heterogeneous machines.

For heterogeneous servers, give stronger machines more virtual nodes.

## Replication

For replication factor N:

-   Find the key position.
-   Walk clockwise.
-   Select N distinct physical servers.

Replicas should preferably be distributed across failure domains such as
different data centers.

## When to use

Consistent hashing is useful for:

-   Distributed caches.
-   Key-value stores.
-   Partitioned services.
-   Request routing.
-   Distributed crawlers.

## Important caveat

Consistent hashing reduces data movement. It does not automatically
solve:

-   Hot keys.
-   Uneven workloads.
-   Replication.
-   Failure detection.
-   Data consistency.

------------------------------------------------------------------------

# 6. Design a Key-Value Store

## Core requirements

A distributed key-value store should provide:

-   Fast reads.
-   Fast writes.
-   High availability.
-   Horizontal scalability.
-   Fault tolerance.
-   Defined consistency behavior.

## Partitioning

Use consistent hashing to map keys to nodes.

## Replication

Replicate each key to multiple nodes.

Typical configuration:

`N = replication factor`

For a request:

`W = write quorum`

`R = read quorum`

A common rule:

`W + R > N`

provides overlap between read and write sets and is used to obtain
stronger consistency under the model assumed by the design.

Examples:

-   `R = 1, W = N` -\> optimized for fast reads.
-   `W = 1, R = N` -\> optimized for fast writes.
-   `R = 2, W = 2, N = 3` -\> common quorum configuration.

Important nuance:

Quorum does not magically guarantee all forms of strong consistency
under every failure and concurrency model.

## Consistency models

### Strong consistency

A read sees the latest committed write.

### Weak consistency

A later read may return an older value.

### Eventual consistency

If updates stop, replicas eventually converge.

## Conflict resolution

Concurrent writes can create multiple versions.

Use versioning.

A vector clock can identify whether:

-   One version causally follows another.
-   Two versions are concurrent.

Concurrent versions may require application-level reconciliation.

## Failure detection

Gossip protocol:

1.  Nodes maintain membership information.
2.  Nodes increment heartbeat counters.
3.  Nodes exchange heartbeat information.
4.  Missing heartbeat progress indicates possible failure.
5.  Failure information propagates.

## Temporary failures

### Sloppy quorum

Instead of requiring the normal replica nodes, use the first healthy
nodes available.

### Hinted handoff

When the original node returns, temporarily stored data is handed back.

## Permanent failures

Use anti-entropy.

A Merkle tree allows replicas to compare hashes efficiently and locate
inconsistent ranges without transferring the entire dataset.

## Big picture

**Consistent hashing -\> replication -\> quorum -\> conflict resolution
-\> failure detection -\> hinted handoff -\> anti-entropy**

That is the core interview story.

------------------------------------------------------------------------

# 7. Unique ID Generator

## Requirements

Typical requirements:

-   Unique.
-   Numeric if required.
-   Time sortable if required.
-   Fixed size such as 64 bits.
-   High throughput.
-   Highly available.
-   No central bottleneck if possible.

## Options

### Database auto increment

Simple but does not scale well across distributed writers.

### Multi-master

Each database increments by the number of masters.

Example:

Server 1: 1, 3, 5...

Server 2: 2, 4, 6...

Problems:

-   Harder across data centers.
-   Ordering is not truly time based.
-   Changing server count complicates the scheme.

### UUID

Advantages:

-   No coordination.
-   Easy distributed generation.

Problems:

-   128 bits.
-   Not compact.
-   Poor database locality in some versions and storage patterns.
-   Not naturally time sortable for classic UUID variants.

### Ticket server

A central service generates IDs.

Problem:

-   Central dependency.
-   Availability concern.
-   Scaling and synchronization challenges.

## Snowflake

A 64 bit style ID can be divided into:

-   1 sign bit.
-   41 timestamp bits.
-   5 data center bits.
-   5 machine bits.
-   12 sequence bits.

The exact bit allocation can be changed based on requirements.

The key idea is:

**timestamp + machine identity + sequence**

This provides:

-   Distributed generation.
-   High throughput.
-   Time ordering.
-   Compact IDs.

## Sequence

12 bits gives:

`2^12 = 4096`

IDs per millisecond per machine under that allocation.

## Timestamp

41 bits gives roughly 69 years at millisecond resolution.

## Critical issue: clock problems

If the system clock moves backward, time based IDs can become
problematic.

Possible strategies:

-   Wait.
-   Detect clock rollback.
-   Use logical sequencing.
-   Use a time service.
-   Reserve additional mechanisms for clock safety.

Do not ignore clock synchronization in a serious interview.

------------------------------------------------------------------------

# 8. URL Shortener

## Requirements

Typical operations:

1.  Long URL -\> short URL.
2.  Short URL -\> original URL.
3.  High read traffic.
4.  High availability.
5.  Permanent mapping.

## API

Example:

`POST /shorten`

Input:

`long_url`

Output:

`short_url`

Redirect:

`GET /{short_code}`

## Capacity reasoning

If 100 million URLs are created per day:

`100,000,000 / 86,400 ≈ 1,160 writes/sec`

At 10:1 reads:

`≈ 11,600 reads/sec`

Over 10 years:

`≈ 365 billion URLs`

## Short code size

Using:

-   0 to 9
-   a to z
-   A to Z

gives 62 characters.

Find the smallest n where:

`62^n >= required number of URLs`

For hundreds of billions, 7 characters is enough in the book example.

## Two approaches

### Hash + collision resolution

Hash the long URL.

Potential hashes:

-   MD5
-   SHA family
-   CRC

Truncate to the desired length.

Problem:

**Collisions**

Resolve collisions by generating another candidate and checking storage.

A Bloom filter can reduce unnecessary database lookups when checking
whether a candidate exists.

### Base62

Generate a numeric ID and convert it to base 62.

This is deterministic and avoids hash collision problems if the numeric
ID itself is unique.

## Storage

Use a database for:

`id -> short_code -> long_url`

Cache popular redirects.

## Scaling

-   Stateless web servers.
-   Load balancer.
-   Cache hot mappings.
-   Database replication.
-   Database sharding.
-   Analytics asynchronously.
-   Rate limiting to prevent abuse.

------------------------------------------------------------------------

# 9. Web Crawler

## Goal

Continuously discover and process web content.

## Main components

**Seed URLs -\> URL Frontier -\> Downloader -\> Parser -\> Deduplication
-\> Link Extractor -\> URL Filter -\> URL Frontier**

## URL Frontier

Stores URLs waiting to be downloaded.

It should be:

-   Durable.
-   Scalable.
-   Partitioned.
-   Polite.

A hybrid design can keep active buffers in memory while storing the
majority of the frontier on disk.

## Downloader

Uses HTTP.

Important concerns:

-   DNS.
-   Timeouts.
-   Retries.
-   robots.txt.
-   Connection reuse.
-   Geographic locality.

## robots.txt

Before crawling a site, respect the sites crawler rules.

Cache robots.txt results to avoid repeatedly downloading them.

## Deduplication

Two kinds:

### URL deduplication

Have we already crawled this URL?

### Content deduplication

Do two different URLs contain identical content?

Content fingerprints or hashes can help.

## Politeness

Do not overload one website.

Use:

-   Per-domain queues.
-   Rate limits.
-   Delays.
-   Concurrent request limits.

## Scaling

Partition URL space across workers.

Use consistent hashing or another partitioning scheme.

## Robustness

Persist crawl state.

If a worker crashes, another worker can resume unfinished work.

## Important edge cases

-   Spider traps.
-   Dynamic JavaScript generated links.
-   Spam.
-   Duplicate content.
-   Very large pages.
-   Slow or dead hosts.

------------------------------------------------------------------------

# 10. Notification System

## Notification types

-   Mobile push.
-   SMS.
-   Email.

## Core architecture

**Client service -\> Notification API -\> Metadata Cache/DB -\> Message
Queue -\> Workers -\> Third Party Provider -\> User**

Use separate queues by notification type when useful.

Example:

-   Push queue.
-   SMS queue.
-   Email queue.

This prevents an outage in one provider from blocking unrelated
notification types.

## Why queues

Queues provide:

-   Decoupling.
-   Buffering.
-   Retry support.
-   Load smoothing.
-   Independent scaling.

## Workers

Workers consume notification events and call external providers.

## Reliability

External services fail.

Use:

-   Retries.
-   Exponential backoff.
-   Dead letter queues.
-   Idempotency.
-   Monitoring.

Be careful with retries because notification systems can accidentally
send duplicates.

## User settings

Before sending:

-   Check opt-in status.
-   Check notification type.
-   Check frequency limits.
-   Check quiet periods if supported.

## Security

Authenticate producers.

The book discusses an AppKey/AppSecret style mechanism.

## Monitoring

Track:

-   Queue depth.
-   Delivery success rate.
-   Failure rate.
-   Retry count.
-   Provider latency.
-   Delivery latency.
-   Notification volume.

------------------------------------------------------------------------

# 11. News Feed System

## Core challenge

Feed generation creates a fanout problem.

A post may need to appear in thousands or millions of feeds.

## Two models

### Fanout on write

When a user posts:

1.  Find followers.
2.  Create feed entries for them.
3.  Store feed references in cache.

Advantages:

-   Very fast feed reads.
-   Work is performed ahead of time.

Disadvantages:

-   Expensive writes.
-   Celebrity or hot user creates huge fanout.
-   Wasteful for inactive users.

### Fanout on read

When user opens feed:

1.  Find followed users.
2.  Fetch recent posts.
3.  Merge and sort.

Advantages:

-   Cheap writes.
-   Good for inactive users.
-   No huge fanout during posting.

Disadvantages:

-   Slow reads.
-   Heavy computation at read time.

## Best practical answer

Use a hybrid.

### Normal users

Fanout on write.

### Celebrities or users with huge follower counts

Fanout on read.

This avoids the hot key problem while preserving fast reads for most
users.

## Feed cache

Store compact references such as:

`<user_id, post_id, timestamp>`

Do not store entire post objects if memory is a concern.

## Architecture

**Client -\> LB -\> API -\> Post Service -\> DB + Cache**

For fanout:

**Post Service -\> Queue -\> Fanout Workers -\> Feed Cache**

For relationships:

-   Graph database or suitable relational representation.

For notifications:

-   Notification service.

## Important interview phrase

**The main bottleneck is not simply post writes. It is fanout multiplied
by follower count.**

------------------------------------------------------------------------

# 12. Chat System

## Core requirements

-   One-to-one messaging.
-   Group chat.
-   Online/offline status.
-   Message ordering.
-   Multi-device synchronization.
-   Offline delivery.
-   Push notifications.

## Protocols

### Polling

Client repeatedly asks:

"Do I have messages?"

Simple but inefficient.

### Long polling

Client keeps request open until:

-   Message arrives.
-   Timeout occurs.

Better than polling but still has connection management limitations.

### WebSocket

Persistent bidirectional connection.

Ideal for real-time chat.

## Architecture

**Client -\> API/LB -\> Chat Service**

Clients maintain WebSocket connections to chat servers.

Service discovery can select an appropriate chat server based on:

-   Geography.
-   Capacity.
-   Availability.

## Message flow

Typical flow:

1.  Sender sends message.
2.  Chat server gets message ID.
3.  Message enters synchronization queue.
4.  Message is persisted.
5.  If recipient is online, deliver through recipient chat server.
6.  If offline, send push notification.
7.  Recipient reconnects and synchronizes missing messages.

## Message IDs

Requirements:

-   Unique within the relevant scope.
-   Time sortable or otherwise ordered.

The book explains that local sequence numbers can be enough if ordering
is only required within one conversation.

## Multi-device synchronization

Each device tracks something like:

`cur_max_message_id`

On reconnect, it asks for messages newer than that ID.

This is a simple and powerful synchronization technique.

## Group chat

For small groups:

Copy a message into each recipient inbox or synchronization queue.

Benefits:

-   Simple recipient synchronization.
-   Easy delivery.

For very large groups:

Do not blindly copy every message to every member.

Use a different strategy based on group size and traffic.

## Core interview tradeoff

**Small group -\> push/fanout**

**Huge group -\> pull/on-demand or hybrid**

------------------------------------------------------------------------

# 13. Search Autocomplete

## Requirements

-   Very low latency.
-   Relevant results.
-   Ranked results.
-   High QPS.
-   High availability.

Autocomplete is extremely read heavy.

## Basic architecture

Two separate concerns:

### Data gathering service

Collect search queries and calculate frequencies.

### Query service

Given a prefix, return top K suggestions.

## Why not query a database directly

A relational database can store frequencies, but searching and sorting
all matching prefixes becomes expensive.

## Trie

A trie stores strings by prefix.

Example:

`car`

and

`cat`

share:

`ca`

This makes prefix lookup efficient.

## Basic query

1.  Find prefix node.
2.  Traverse matching subtree.
3.  Find valid queries.
4.  Sort by frequency.
5.  Return top K.

Problem:

Traversing a large subtree is expensive.

## Optimization 1

Limit maximum prefix length.

Most users do not type extremely long prefixes.

## Optimization 2

Store top K suggestions at every trie node.

Then:

`find prefix -> return cached top K`

This trades memory for speed.

The book uses top 5 as the example.

## Data gathering optimization

Do not rebuild the trie on every query.

Instead:

**User queries -\> Logging/Analytics -\> Batch processing -\> Aggregated
frequencies -\> Trie builder -\> New trie**

Swap in the new trie atomically.

This is especially appropriate when suggestions change slowly.

## Trending queries

For real-time trends:

-   Stream processing.
-   Recent-event weighting.
-   Sharded processing.
-   Incremental updates.

## Geographic differences

Use regional tries when rankings differ by country or region.

CDNs can help distribute read-only trie data.

------------------------------------------------------------------------

# 14. YouTube / Video Streaming

## Three major flows

1.  Upload.
2.  Transcode.
3.  Stream.

## Upload

Do not send the video through every API server.

Use object storage.

A simplified flow:

**Client -\> Upload service -\> Original storage**

Then asynchronous processing.

## Metadata

Metadata includes:

-   Video ID.
-   User.
-   File size.
-   Resolution.
-   Format.
-   Status.
-   Encoded versions.
-   URLs.

Keep metadata separate from large video objects.

## Transcoding

A video must often be converted into:

-   Multiple resolutions.
-   Multiple bitrates.
-   Multiple codecs.
-   Different streaming formats.

This is why the original upload cannot simply be served directly.

## DAG

Video processing is modeled as a directed acyclic graph.

Example:

**Input -\> Split -\> Video Encoding**

and in parallel:

**Input -\> Audio Encoding**

and:

**Input -\> Thumbnail**

The DAG allows independent tasks to run in parallel.

## Transcoding architecture

Main components:

-   Preprocessor.
-   DAG scheduler.
-   Resource manager.
-   Task queue.
-   Worker pool.
-   Temporary storage.
-   Encoded storage.
-   Completion queue.

## Preprocessor

Responsibilities:

-   Split video into GOP aligned chunks.
-   Generate DAG.
-   Cache temporary data.
-   Prepare processing metadata.

## Resource manager

Tracks:

-   Waiting tasks.
-   Available workers.
-   Running tasks.

Scheduler chooses the best task and worker.

## Message queues

Queues decouple processing stages.

Instead of:

`A must finish -> B starts`

you can use:

`A emits event -> queue -> B consumes`

This increases parallelism and fault isolation.

## Streaming

Use CDN.

Typical flow:

**Client -\> CDN -\> Encoded video**

CDN keeps popular videos close to users.

## Adaptive quality

Network conditions vary.

Offer multiple representations so the client can switch quality.

## Cost optimization

Video CDN bandwidth is expensive.

Possible strategies:

-   Cache popular videos.
-   Regional placement.
-   Do not replicate unpopular content everywhere.
-   Partner with ISPs.
-   Analyze viewing patterns.

## Security

Pre-signed upload URLs allow authorized clients to upload directly to
controlled storage locations.

## Failure handling

Recoverable:

-   Retry.
-   Reschedule.

Non-recoverable:

-   Stop processing.
-   Report failure.

------------------------------------------------------------------------

# 15. Google Drive / File Storage and Sync

## Core requirements

-   Upload.
-   Download.
-   Sync across devices.
-   Version history.
-   Sharing.
-   Notifications.
-   Strong consistency.
-   Reliability.
-   Low bandwidth usage.

## Separate metadata from file data

### Metadata database

Stores:

-   User.
-   File.
-   Block information.
-   Version.
-   Permissions.
-   Relationships.

### Object storage

Stores actual file blocks.

This separation is fundamental.

## Block storage

Large files are split into blocks.

Each block has a hash.

A file can be represented as:

`[block1, block2, block3, ...]`

To reconstruct the file, fetch and join the blocks in order.

## Why blocks

If a large file changes slightly, do not upload the entire file again.

Only upload changed blocks.

This is **delta sync**.

## Compression

Compress blocks before storage or transmission when appropriate.

## Encryption

Encrypt blocks before storage.

## Deduplication

If two blocks have the same content hash, they can potentially share the
same stored block.

This saves storage.

## Strong consistency

For file synchronization, users should not see contradictory metadata
between clients.

Important techniques:

-   Database transactions.
-   Cache invalidation on writes.
-   Consistent metadata replicas.
-   Careful synchronization logic.

## Notification service

The notification service tells clients:

"Something changed. Fetch the latest state."

The book chooses long polling rather than WebSocket because the
communication pattern is mostly server-to-client and changes are
relatively infrequent.

## Upload flow

Simplified:

1.  Client sends file.
2.  File is split into blocks.
3.  Blocks are compressed.
4.  Blocks are encrypted.
5.  Blocks are stored in object storage.
6.  Metadata DB records block mapping and file version.
7.  Notification service informs other clients.

## Download/sync flow

1.  Client receives change notification.
2.  Client requests latest metadata.
3.  Metadata DB returns changed block information.
4.  Client requests blocks.
5.  Block server retrieves blocks from cloud storage.
6.  Client reconstructs the file.

## Conflict

If two clients update the same file concurrently, a conflict can occur.

A simple strategy is:

-   One version wins based on processing order.
-   The other is surfaced as a conflict.
-   The user can merge or override.

## Failure handling

### API server fails

Stateless, so route request elsewhere.

### Metadata cache fails

Use replicas.

### Database master fails

Promote a replica.

### Database replica fails

Use another replica and replace the failed one.

### Object storage region fails

Use replicated storage in another region.

### Notification server fails

Clients reconnect to another server.

### Offline client

Queue synchronization information and reconcile when the client returns
online.

------------------------------------------------------------------------

# 16. The Learning Continues

The final chapter reinforces the main lesson:

System design knowledge is not a list of architectures.

The goal is to develop the ability to:

-   Recognize patterns.
-   Ask the right questions.
-   Identify bottlenecks.
-   Choose appropriate technologies.
-   Explain tradeoffs.
-   Evolve a design.

The individual technologies will change.

The principles remain.

------------------------------------------------------------------------

# 17. Master Pattern Library

## Pattern 1: Stateless services

Use when you want easy horizontal scaling.

**LB -\> many stateless servers**

State lives in:

-   DB
-   Cache
-   Object storage
-   External session store

------------------------------------------------------------------------

## Pattern 2: Cache

Use when data is frequently read.

**Read -\> Cache -\> DB on miss**

Questions:

-   TTL?
-   Invalidation?
-   Stale reads?
-   Cache failure?

------------------------------------------------------------------------

## Pattern 3: CDN

Use for large or static geographically distributed content.

**Client -\> CDN -\> Origin**

------------------------------------------------------------------------

## Pattern 4: Read replicas

Use when reads dominate.

**Primary -\> Replicas**

Writes go to primary.

Reads can be distributed.

Watch for replication lag.

------------------------------------------------------------------------

## Pattern 5: Sharding

Use when one database cannot handle:

-   Data size.
-   Write throughput.
-   CPU.
-   Storage.

Choose a shard key carefully.

Bad shard key can create a hot shard.

------------------------------------------------------------------------

## Pattern 6: Message queue

Use for asynchronous work.

**Producer -\> Queue -\> Consumer**

Benefits:

-   Decoupling.
-   Buffering.
-   Retry.
-   Load smoothing.
-   Independent scaling.

------------------------------------------------------------------------

## Pattern 7: Worker pool

Use when work is CPU or IO heavy and asynchronous.

**Queue -\> many workers**

Scale worker count independently.

------------------------------------------------------------------------

## Pattern 8: Fanout on write

Best when:

-   Reads dominate.
-   Followers are manageable.
-   Low read latency is critical.

------------------------------------------------------------------------

## Pattern 9: Fanout on read

Best when:

-   Users are inactive.
-   Fanout is huge.
-   Write amplification is unacceptable.

------------------------------------------------------------------------

## Pattern 10: Hybrid fanout

Default answer for social feeds.

Normal users:

**push**

Celebrities:

**pull**

------------------------------------------------------------------------

## Pattern 11: Object storage

Use for:

-   Images.
-   Videos.
-   Documents.
-   Large blobs.

Do not put huge media objects into a relational database unless there is
a specific reason.

------------------------------------------------------------------------

## Pattern 12: Metadata DB + object storage

Very common architecture:

**Metadata DB -\> describes objects**

**Object storage -\> stores actual objects**

Examples:

-   Google Drive.
-   YouTube.
-   Image storage.
-   Document systems.

------------------------------------------------------------------------

## Pattern 13: Delta sync

When a large object changes slightly:

**sync changed chunks only**

This saves:

-   Bandwidth.
-   Time.
-   Storage.

------------------------------------------------------------------------

## Pattern 14: Consistent hashing

Use for:

-   Distributed caches.
-   Key-value stores.
-   Request routing.

Main benefit:

**Minimize key movement when nodes change.**

------------------------------------------------------------------------

## Pattern 15: Quorum

For replicated systems:

-   N = replicas.
-   W = write acknowledgements.
-   R = read acknowledgements.

A common quorum relationship:

`W + R > N`

But always state the consistency assumptions.

------------------------------------------------------------------------

## Pattern 16: Retry

Retry transient failures.

Do not blindly retry permanent failures.

Use:

-   Exponential backoff.
-   Jitter.
-   Maximum retry count.
-   Dead letter handling.
-   Idempotency.

------------------------------------------------------------------------

## Pattern 17: Idempotency

If a request can be retried, make repeated execution safe.

Example:

`POST payment with idempotency_key`

The same key should not create two payments.

This concept is extremely important in real system design interviews.

------------------------------------------------------------------------

# 18. Failure Handling Cheat Sheet

When the interviewer asks:

"What happens if X fails?"

Use this sequence:

### 1. Detect

-   Health check.
-   Heartbeat.
-   Timeout.
-   Monitoring.

### 2. Isolate

-   Stop sending traffic.
-   Circuit breaker.
-   Remove unhealthy node.

### 3. Recover

-   Retry.
-   Restart.
-   Reschedule.
-   Fail over.

### 4. Replace

-   New worker.
-   New replica.
-   New API server.

### 5. Reconcile

-   Replication.
-   Hinted handoff.
-   Anti-entropy.
-   Replay queue.
-   Rebuild cache.

### 6. Monitor

-   Alert.
-   Metrics.
-   Logs.
-   Traces.

------------------------------------------------------------------------

# 19. Consistency Cheat Sheet

## Strong consistency

Use when correctness requires the latest value.

Examples:

-   Financial transactions.
-   Critical metadata.
-   Some file synchronization state.

Cost:

-   More coordination.
-   Higher latency.
-   Potentially lower availability during failures.

## Eventual consistency

Use when:

-   Temporary staleness is acceptable.
-   Availability matters.
-   Global scale matters.

Examples:

-   Some social feed data.
-   Some caches.
-   Some replicated key-value stores.

## Ask this question

**What is the user impact if the data is stale for one second?**

That question often determines the consistency model.

------------------------------------------------------------------------

# 20. Replication vs Sharding

Do not confuse them.

## Replication

Copies the same data to multiple nodes.

Primary goals:

-   Availability.
-   Read scaling.
-   Fault tolerance.

## Sharding

Splits different data across nodes.

Primary goals:

-   Write scaling.
-   Storage scaling.
-   Data distribution.

You often need both:

**Sharding + replication**

------------------------------------------------------------------------

# 21. Cache vs Database

Cache:

-   Faster.
-   Usually less durable.
-   Limited size.
-   Used to reduce database load.

Database:

-   Source of truth.
-   Durable.
-   Rich querying or structured access.

Do not make the cache the only source of truth unless the system
explicitly supports that model.

------------------------------------------------------------------------

# 22. Queue vs Database

A queue is optimized around:

-   Ordering.
-   Delivery.
-   Consumption.
-   Asynchronous processing.

A database is optimized around:

-   Persistent state.
-   Queries.
-   Updates.
-   Data relationships.

Kafka-like systems can blur this distinction because durable logs can
act as both messaging infrastructure and a persistent event source.

------------------------------------------------------------------------

# 23. WebSocket vs Long Polling

## WebSocket

Use when:

-   Bidirectional communication.
-   Real-time updates.
-   High frequency interaction.

Examples:

-   Chat.
-   Multiplayer games.
-   Collaborative applications.

## Long polling

Use when:

-   Server-to-client notification.
-   Updates are relatively infrequent.
-   Simpler HTTP infrastructure is desirable.

Examples:

-   Some file synchronization notifications.
-   Older real-time architectures.

------------------------------------------------------------------------

# 24. SQL vs NoSQL

Do not say:

"NoSQL is faster."

That is too vague.

Choose based on access pattern.

## SQL

Strong choice when you need:

-   Relationships.
-   Transactions.
-   Flexible queries.
-   Strong consistency.
-   Mature indexing.

## Key-value

Strong choice when:

-   Access is primarily key based.
-   Very high scale.
-   Simple data access.

## Document

Strong choice when:

-   Data is naturally document shaped.
-   Schema evolves frequently.
-   Related data can be stored together.

## Graph

Strong choice when:

-   Relationships are the primary query.

------------------------------------------------------------------------

# 25. The Most Important Interview Tradeoffs

Memorize these pairs.

  Requirement                           Typical direction
  ------------------------------------- --------------------------------------
  Lower latency                         Cache, CDN, precomputation
  Higher availability                   Replication, redundancy
  More write capacity                   Sharding, batching, async processing
  More read capacity                    Cache, replicas, CDN
  Lower bandwidth                       Compression, delta sync, CDN
  Less coupling                         Message queues
  Better burst handling                 Queue, token bucket
  Stronger consistency                  Coordination, transactions
  Higher availability under partition   Eventual consistency
  Lower storage                         Compression, deduplication
  Faster reads                          Precompute, indexes, materialization
  Cheaper writes                        Fanout on read
  Faster feed reads                     Fanout on write
  Better geographic latency             Multi-region, CDN
  Safer retries                         Idempotency

------------------------------------------------------------------------

# 26. Common Red Flags to Avoid

Do not say:

-   "Just add more servers."
-   "Use Redis for everything."
-   "Use Kafka because it is scalable."
-   "Use NoSQL because SQL does not scale."
-   "Use microservices" without a reason.
-   "Use WebSocket" for every real-time requirement.
-   "Use strong consistency everywhere."
-   "Use eventual consistency everywhere."
-   "Shard the database" without explaining the shard key.
-   "Add a cache" without explaining invalidation.
-   "Add replicas" without discussing replication lag.
-   "Use retries" without discussing duplicate operations.
-   "Use a load balancer" without explaining health checks or failure
    handling.

Every technology choice should answer:

**Why this component? Why here? What problem does it solve? What
tradeoff does it introduce?**

------------------------------------------------------------------------

# 27. The 60 Minute Interview Template

## Minutes 0 to 5

Clarify requirements.

Say:

"I will first clarify the functional and non-functional requirements so
that we design for the correct scope."

Ask:

-   Users?
-   Core features?
-   Read/write ratio?
-   Scale?
-   Latency?
-   Consistency?
-   Availability?
-   Geography?
-   Data retention?

## Minutes 5 to 10

Estimate:

-   QPS.
-   Peak QPS.
-   Storage.
-   Bandwidth.

## Minutes 10 to 20

Draw high-level architecture.

Start simple.

Then add:

-   Cache.
-   DB replicas.
-   Queue.
-   Object storage.
-   CDN.
-   Search.
-   Workers.

Only when justified.

## Minutes 20 to 40

Deep dive into the most important bottleneck.

Examples:

-   Feed fanout.
-   Chat message delivery.
-   Video transcoding.
-   URL ID generation.
-   Autocomplete trie.
-   File synchronization.

## Minutes 40 to 50

Discuss:

-   Failure handling.
-   Scaling.
-   Data consistency.
-   Hot keys.
-   Retry.
-   Monitoring.

## Minutes 50 to 60

Tradeoffs and extensions.

Examples:

-   Multi-region.
-   Disaster recovery.
-   Security.
-   Analytics.
-   Cost optimization.

------------------------------------------------------------------------

# 28. Final Pre-Interview Checklist

Before an interview, make sure you can explain these without notes:

### Fundamentals

-   Load balancer.
-   Stateless service.
-   Cache.
-   CDN.
-   SQL vs NoSQL.
-   Replication.
-   Sharding.
-   Consistent hashing.
-   Message queue.
-   Worker pool.
-   Object storage.
-   WebSocket.
-   Long polling.
-   Rate limiting.

### Distributed systems

-   CAP intuition.
-   Strong vs eventual consistency.
-   Quorum.
-   Replication lag.
-   Failure detection.
-   Idempotency.
-   Retry with backoff.
-   Hot partitions.
-   Hot keys.
-   Leader/follower concepts.

### Algorithms and structures

-   Token bucket.
-   Leaky bucket.
-   Fixed window.
-   Sliding window.
-   Consistent hashing.
-   Bloom filter.
-   Trie.
-   Snowflake.
-   Merkle tree.

### Classic designs

-   Rate limiter.
-   Consistent hashing.
-   Key-value store.
-   Unique ID generator.
-   URL shortener.
-   Web crawler.
-   Notification system.
-   News feed.
-   Chat.
-   Autocomplete.
-   YouTube.
-   Google Drive.

------------------------------------------------------------------------

# 29. The Ultimate Mental Shortcut

When given any system design question, think:

## Step 1

**What does the user need?**

## Step 2

**How big is it?**

## Step 3

**What is the critical read path?**

## Step 4

**What is the critical write path?**

## Step 5

**What is the source of truth?**

## Step 6

**What can be cached?**

## Step 7

**What can be asynchronous?**

## Step 8

**Where does data partition?**

## Step 9

**How is data replicated?**

## Step 10

**What consistency is required?**

## Step 11

**What happens when something fails?**

## Step 12

**What is the bottleneck at 10x scale?**

## Step 13

**What tradeoff am I making?**

If you can answer these questions clearly, you can design systems that
you have never seen before.

------------------------------------------------------------------------

# Final takeaway

The most important lesson from the book is not any individual
technology.

It is the ability to move from:

**requirements -\> numbers -\> bottleneck -\> architecture -\> tradeoff
-\> failure handling -\> evolution**

A strong system design interview answer is therefore not:

"Here is my architecture."

It is:

"Here are the requirements. Here is the scale. Given those constraints,
I choose this architecture because it solves this bottleneck. Here is
what happens when it fails. Here is how it scales. Here is the
tradeoff."

That is the skill the entire book is trying to teach.
