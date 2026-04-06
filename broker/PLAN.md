# Project 1: Mini-Kafka — Complete Implementation Guide

> Source spec: https://docs.google.com/document/d/1pVPpgd3NxclO2gntKl3wmGLZiXVdwp27G4secb96-zo/edit?tab=t.0

## What You're Building

A simplified Kafka-like distributed message broker with:
- Append-only log split into topics and partitions
- Leader/follower replication across brokers
- Produce and fetch HTTP APIs
- Consumer groups with partition assignment and rebalancing
- Observability metrics (lag, replication, throughput)

## Architecture Overview

```
┌─────────────┐        ┌─────────────┐
│  Producer    │───────▶│   Broker 1   │ (leader for partition 0)
└─────────────┘        │   Broker 2   │ (follower for partition 0, leader for partition 1)
┌─────────────┐        │   Broker 3   │ (follower for partitions 0 & 1)
│  Consumer   │◀───────│              │
│  Group      │        └──────┬───────┘
└─────────────┘               │
                       ┌──────┴───────┐
                       │  Controller   │
                       │ (metadata,    │
                       │  heartbeats,  │
                       │  failover)    │
                       └──────────────┘
```

**Broker** — Owns partition logs on disk. Acts as leader or follower per partition. Handles replication.
**Controller** — Stores cluster metadata (who leads which partition). Monitors broker heartbeats. Triggers failover. Coordinates consumer groups.

---

## Prerequisites: Concepts You Need First

Before writing any code, understand these fundamentals. You don't need deep expertise — just enough to know what they are and why they matter.

### 1. What is an append-only log?
A file where you only ever write to the end. You never modify or delete existing data. Each entry gets a sequential number (offset). This is the core data structure of Kafka.

**Why?** Append-only writes are the fastest possible disk I/O pattern. They're also naturally ordered, which makes them ideal for event streams.

### 2. What is a partition?
A single log file can only live on one machine. To scale, you split a topic into N partitions, each being its own independent log. Records are routed to partitions by hashing their key.

### 3. What is replication?
Each partition has one leader (handles reads/writes) and one or more followers (copy the leader's data). If the leader dies, a follower takes over. This is how you achieve durability.

### 4. What are consumer groups?
Multiple consumers form a group. The partitions of a topic are divided among them so each partition is read by exactly one consumer. When consumers join/leave, partitions get redistributed (rebalanced).

### 5. What is an offset?
A monotonically increasing integer assigned to each record in a partition. Consumers track their position by remembering the last offset they processed (committed offset).

---

## Language & Tech Stack

The spec doesn't prescribe a language. Recommended: **Go** or **Python**.

- **Go**: Better for learning systems programming. Native concurrency (goroutines), fast, compiles to a single binary, used heavily in infrastructure.
- **Python**: Faster to prototype. Use if you want to focus on concepts over performance.

This guide assumes **Go** but the structure applies to any language.

### Project Structure
```
broker/
├── cmd/
│   ├── broker/main.go        # Broker server entry point
│   └── controller/main.go    # Controller server entry point
├── internal/
│   ├── log/                   # Log, segment, index implementation
│   ├── broker/                # Broker server, produce/fetch handlers
│   ├── controller/            # Controller, metadata, heartbeats
│   ├── consumer/              # Consumer group coordination
│   └── replication/           # Follower fetch loop, ISR tracking
├── scripts/
│   ├── kill_broker.sh
│   └── kill_consumer.sh
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── PLAN.md                    # This file
└── README.md
```

---

## Week 1: Single-Node Log

**Goal**: A single process that can append records to a file and read them back by offset. Handle crashes mid-write.

### Concepts to Learn
- File I/O: opening files, seeking, writing, fsyncing
- Binary encoding: how to serialize a record to bytes
- Crash safety: what happens if the process dies mid-write

### Step 1.1: Define the Record Format

Each record on disk needs a fixed structure so you can read it back:

```
┌──────────┬──────────┬───────────┬──────────┬───────────┬──────────┬───────┐
│ length   │ CRC32    │ offset    │ eventTime│ keyLen    │ key      │ value │
│ (4 bytes)│ (4 bytes)│ (8 bytes) │ (8 bytes)│ (4 bytes) │ (varies) │(rest) │
└──────────┴──────────┴───────────┴──────────┴───────────┴──────────┴───────┘
```

- **length**: Total byte count of everything after this field. Lets you skip to the next record.
- **CRC32**: Checksum of the remaining bytes. Detects corruption from partial writes.
- **offset**: Monotonically increasing sequence number (0, 1, 2, ...).
- **eventTime**: Unix timestamp in milliseconds.
- **keyLen**: Length of the key in bytes. If 0, key is empty.
- **key**: The partition routing key (e.g., user ID).
- **value**: The actual message payload (rest of the bytes after key).

### Step 1.2: Implement the Log

Create a `Log` struct that manages a single file:

```
type Log struct {
    file       *os.File
    nextOffset uint64
    mu         sync.Mutex
}
```

Methods to implement:
1. **`NewLog(path string) (*Log, error)`** — Open or create the log file. If the file already exists (restart recovery), scan it to find the last valid offset. This is your crash recovery.
2. **`Append(key, value []byte, eventTime int64) (uint64, error)`** — Encode the record, write it to the file, fsync, increment nextOffset, return the offset.
3. **`Read(offset uint64) (*Record, error)`** — Scan from the beginning of the file, skipping records until you reach the desired offset. (This is slow — we'll fix it in Week 2.)

### Step 1.3: Crash Recovery

What if the process crashes mid-write? The last record might be partially written.

**Solution**: When opening an existing log file, scan through all records. For each record:
1. Read the length field (4 bytes). If you can't read 4 bytes, you've hit EOF — truncate here.
2. Read `length` bytes. If you can't read that many, the record is incomplete — truncate the file back to before this record's length field.
3. Compute the CRC32 of the payload. If it doesn't match, the record is corrupt — truncate.
4. If valid, record the offset and file position, continue to next record.

After scanning, `nextOffset` = last valid offset + 1.

### Step 1.4: Build an HTTP API

Stand up a basic HTTP server with two endpoints:

**`POST /produce`**
```json
Request:  { "topic": "orders", "key": "user-123", "value": "order data", "eventTime": 1700000000000 }
Response: { "offset": 0 }
```

**`GET /fetch?topic=orders&offset=0`**
```json
Response: { "records": [{ "offset": 0, "key": "user-123", "value": "order data", "eventTime": 1700000000000 }] }
```

For now, ignore the topic field — you only have one log. We'll add topics/partitions in Week 3.

### Step 1.5: Test It

Write tests that verify:
- Appending N records and reading them back returns the same data
- Offsets are sequential (0, 1, 2, ...)
- Kill the process after appending (simulate with truncating the file mid-record), restart, verify recovery works and only complete records survive
- After recovery, new appends continue from the correct offset

### Deliverable
A single-process server that appends records, reads by offset, and recovers from crashes.

---

## Week 2: Segments + Sparse Index

**Goal**: Split the log into fixed-size segments. Add an index for fast offset lookups instead of scanning from the beginning.

### Concepts to Learn
- Log segmentation: why a single file doesn't scale
- Sparse indexes: trading memory for lookup speed
- Memory-mapped files (optional): using mmap for index files
- Log retention: deleting old segments by time or size

### Step 2.1: Understand Why Segments

A single file that grows forever has problems:
- Reading offset 1,000,000 requires scanning past 999,999 records
- You can never delete old data (compaction)
- File operations get slower as files grow

**Solution**: Roll to a new segment file after the current one exceeds N bytes (e.g., 10 MB). Each segment is named by its base offset: `00000000000000000000.log`, `00000000000000050000.log`, etc.

### Step 2.2: Implement Segments

```
type Segment struct {
    baseOffset uint64
    log        *os.File    // segment.log — the actual records
    index      *os.File    // segment.idx — sparse offset-to-position mapping
    size       int64       // current size of the log file
    nextOffset uint64
}

type SegmentedLog struct {
    dir            string
    segments       []*Segment     // sorted by baseOffset
    activeSegment  *Segment       // the one we're currently writing to
    maxSegmentSize int64          // roll threshold in bytes
    mu             sync.Mutex
}
```

### Step 2.3: Build the Sparse Index

Instead of indexing every record, index every Nth record (e.g., every 100th). The index file stores fixed-size entries:

```
┌──────────────┬──────────────┐
│ offset (8B)  │ position (8B)│
└──────────────┴──────────────┘
```

**To look up offset X**:
1. Find which segment contains X (binary search on segment base offsets)
2. In that segment's index, binary search for the largest indexed offset ≤ X
3. Seek to that position in the log file
4. Scan forward until you reach offset X

This turns O(N) scans into O(log N) + a short linear scan.

### Step 2.4: Implement Segment Rolling

In the `Append` method:
1. Write the record to the active segment
2. If this is every Nth record, write an index entry
3. After writing, check if `activeSegment.size >= maxSegmentSize`
4. If so, close the current segment (make it read-only) and create a new active segment with `baseOffset = nextOffset`

### Step 2.5: Log Retention

Segments that grow forever waste disk. Implement a background goroutine that periodically checks for segments eligible for deletion:

```
type RetentionPolicy struct {
    MaxAge      time.Duration  // delete segments older than this (e.g., 7 days)
    MaxLogSize  int64          // delete oldest segments when total log exceeds this (e.g., 1 GB)
}
```

**Rules**:
1. Never delete the active segment (the one being written to)
2. Check the last modified time of each segment file — if older than `MaxAge`, delete it
3. If total log size exceeds `MaxLogSize`, delete the oldest segments first until under the limit
4. Run this check every 60 seconds in a background goroutine

This is how real Kafka manages disk usage. It's also why segment boundaries matter — you can only delete whole segments, not individual records.

### Step 2.6: Update Crash Recovery

On startup:
1. List all `.log` files in the directory, sorted by base offset
2. For each segment, validate records (same CRC/length check as Week 1)
3. Rebuild the index if it's missing or corrupt (just re-scan the log)
4. The last segment becomes the active segment

### Step 2.7: Test It

- Write enough records to trigger multiple segment rolls
- Verify fetch by offset works across segment boundaries
- Verify fetch is significantly faster than Week 1 for large offsets
- Corrupt an index file, restart, verify it gets rebuilt
- Kill mid-write, restart, verify only the last segment needs recovery
- Set a short retention policy, verify old segments get deleted
- Verify fetching a deleted offset returns an appropriate error (e.g., "offset out of range")

### Deliverable
Log is split into segments with sparse indexes. Fetch by offset is fast regardless of log size.

---

## Week 3: Partitions + Partitioner

**Goal**: Support multiple topics, each with multiple partitions. Route records to partitions by hashing the key.

### Concepts to Learn
- Hash-based partitioning: deterministic routing
- Topic/partition data model
- Why partition count matters for parallelism

### Step 3.1: Data Model

```
TopicPartition {
    Topic     string
    Partition int
}
```

Each TopicPartition gets its own SegmentedLog on disk:
```
data/
├── orders-0/         # topic "orders", partition 0
│   ├── 00000000.log
│   └── 00000000.idx
├── orders-1/         # topic "orders", partition 1
│   ├── 00000000.log
│   └── 00000000.idx
└── users-0/          # topic "users", partition 0
    ├── 00000000.log
    └── 00000000.idx
```

### Step 3.2: Implement the Partitioner

```
func partition(key []byte, numPartitions int) int {
    h := fnv.New32a()
    h.Write(key)
    return int(h.Sum32()) % numPartitions
}
```

If the key is empty, use round-robin assignment.

### Step 3.3: Implement TopicManager

```
type TopicManager struct {
    basePath string
    topics   map[string]*Topic
    mu       sync.RWMutex
}

type Topic struct {
    name       string
    partitions []*SegmentedLog
}
```

Methods:
- **`CreateTopic(name string, numPartitions int)`** — Create directories and initialize SegmentedLogs
- **`Produce(topic string, key, value []byte, eventTime int64) (partition int, offset uint64, err error)`** — Hash key → partition → append
- **`Fetch(topic string, partition int, offset uint64, maxRecords int) ([]Record, error)`** — Read from the right SegmentedLog

### Step 3.4: Update the HTTP API

**`POST /produce`**
```json
Request:  { "topic": "orders", "key": "user-123", "value": "..." }
Response: { "partition": 2, "offset": 147 }
```

**`GET /fetch?topic=orders&partition=2&offset=147&max=100`**
```json
Response: { "records": [...] }
```

Add a topic creation endpoint:
**`POST /topics`**
```json
Request:  { "name": "orders", "partitions": 4 }
Response: { "status": "created" }
```

### Step 3.5: Test It

- Same key always maps to the same partition
- Different keys distribute across partitions (verify rough uniformity)
- Each partition's offsets are independent and sequential
- Fetch from one partition doesn't affect others

### Deliverable
Multi-topic, multi-partition broker. Records route to partitions by key hash.

> **Heads-up**: In Week 4, you'll distribute partitions across multiple brokers. This means refactoring the produce/fetch paths so they route requests to the correct broker rather than always handling them locally. Design your `TopicManager` with this in mind — keep the partition-level log logic separate from the "am I the right broker for this?" routing logic, so the refactor is straightforward.

---

## Week 4: Controller + Cluster Metadata

**Goal**: Build a controller that tracks broker liveness and assigns partition leadership. Brokers send heartbeats; the controller maintains the source of truth for "which broker leads which partition."

### Concepts to Learn
- Heartbeat-based failure detection
- Cluster metadata management
- Leader assignment algorithms

### Step 4.1: Why a Controller?

With multiple brokers, you need a single source of truth:
- Which broker is the leader for each partition?
- Which brokers are alive?
- Which brokers are followers (replicas) for each partition?

In real Kafka, this is handled by ZooKeeper or KRaft. We'll build a simpler standalone controller.

### Step 4.2: Controller Data Structures

```
type Controller struct {
    brokers       map[int]*BrokerInfo       // brokerID → info
    assignments   map[TopicPartition]*Assignment
    mu            sync.RWMutex
}

type BrokerInfo struct {
    ID            int
    Address       string
    LastHeartbeat time.Time
    Alive         bool
}

type Assignment struct {
    Leader    int    // broker ID
    Followers []int  // broker IDs
}
```

### Step 4.3: Broker Registration & Heartbeats

**Broker side**: On startup, each broker sends a registration request to the controller, then sends heartbeats every 5 seconds.

**Controller side**:
- **`POST /broker/register`** — Register a broker, assign it an ID
- **`POST /broker/heartbeat`** — Update LastHeartbeat timestamp
- Run a background goroutine that checks every 10 seconds: if `time.Since(LastHeartbeat) > 30s`, mark broker as dead

### Step 4.4: Partition Assignment

When a topic is created, the controller decides which brokers get which partitions:

```
POST /topics/create  →  controller assigns partitions round-robin across live brokers
```

Example with 3 brokers, topic "orders" with 4 partitions:
```
orders-0: leader=broker1, followers=[broker2, broker3]
orders-1: leader=broker2, followers=[broker3, broker1]
orders-2: leader=broker3, followers=[broker1, broker2]
orders-3: leader=broker1, followers=[broker2, broker3]
```

### Step 4.5: Metadata Distribution

Brokers and clients need to know the assignments. The controller exposes:

**`GET /metadata`**
```json
{
  "brokers": [
    {"id": 1, "address": "broker1:9092", "alive": true},
    {"id": 2, "address": "broker2:9092", "alive": true}
  ],
  "assignments": {
    "orders-0": {"leader": 1, "followers": [2, 3]},
    "orders-1": {"leader": 2, "followers": [3, 1]}
  }
}
```

Brokers periodically fetch metadata to know:
- Am I the leader for this partition? → Accept produce requests
- Am I a follower? → Replicate from the leader (next week)

### Step 4.6: Update Produce/Fetch Flow

Now produce/fetch must be routed to the correct leader:
1. Client (or broker receiving the request) checks metadata
2. If this broker is the leader for the target partition → process locally
3. If not → return an error with the correct leader's address (client-side redirect) OR proxy the request

For simplicity, start with returning an error + leader address. The client retries against the correct broker.

### Step 4.7: Controller Persistence

The controller is a single point of failure. If it crashes, all metadata (broker registrations, partition assignments, committed offsets) is lost. Mitigate this by persisting state to disk:

1. Write the full metadata state (assignments, broker list, consumer group offsets) to a JSON file on every mutation
2. On startup, load from this file if it exists
3. Brokers re-register on startup anyway (via heartbeats), so the controller can reconcile stale broker info with live heartbeats

This doesn't make the controller highly available (that would require Raft/Paxos), but it survives restarts. In interviews, be ready to discuss: "What would you need for a fully fault-tolerant controller?" (Answer: replicated state machine via consensus protocol, e.g., what Kafka did with KRaft.)

### Step 4.8: Test It

- Register 3 brokers, verify metadata shows all alive
- Stop sending heartbeats from one broker, verify it's marked dead after timeout
- Create a topic, verify partitions are assigned across brokers
- Produce to a non-leader, verify you get redirected to the correct leader
- Kill and restart the controller, verify it recovers metadata from disk
- Verify brokers re-register and the cluster resumes normal operation

### Deliverable
Controller tracks broker liveness and partition assignments. Topic creation distributes partitions across brokers. Controller state persists across restarts.

---

## Week 5: Replication

**Goal**: Followers replicate data from leaders. Support acks=1 (leader only) and acks=all (all in-sync replicas). Data survives a leader crash.

### Concepts to Learn
- Leader-follower replication
- In-Sync Replicas (ISR)
- Acknowledgment levels and durability guarantees
- High watermark

### Step 5.1: The Replication Model

```
Producer ──▶ Leader Broker ──▶ writes to local log
                    │
                    ├──▶ Follower 1 fetches and appends
                    └──▶ Follower 2 fetches and appends
```

**Key principle**: Followers pull from the leader (the leader doesn't push). This is exactly how real Kafka works.

### Step 5.2: Follower Fetch Loop

Each broker, for each partition where it's a follower, runs a background loop:

```
func (b *Broker) replicatePartition(tp TopicPartition, leaderAddr string) {
    for {
        // 1. Get my current end offset for this partition
        myOffset := b.log.NextOffset(tp)

        // 2. Fetch records from the leader starting at myOffset
        records := fetchFromLeader(leaderAddr, tp, myOffset)

        // 3. Append each record to my local log (preserving the original offset!)
        for _, r := range records {
            b.log.AppendAt(tp, r.Offset, r)
        }

        // 4. Report my end offset back to the leader
        reportOffset(leaderAddr, tp, b.id, b.log.NextOffset(tp))

        sleep(100ms)
    }
}
```

**Important**: The follower must use the leader's offsets, not generate its own. `AppendAt` writes at a specific offset rather than auto-incrementing.

### Step 5.3: Leader Tracks Follower Progress

The leader maintains for each partition:
```
type PartitionState struct {
    LEO             uint64            // Log End Offset (next offset to write)
    FollowerOffsets map[int]uint64    // brokerID → their replicated offset
    ISR             []int             // In-Sync Replica broker IDs
    HighWatermark   uint64            // min(LEO of all ISR members)
}
```

**In-Sync Replicas (ISR)**: A follower is "in sync" if it's within some threshold of the leader's LEO (e.g., within 1000 records or within 10 seconds). If it falls behind, remove it from the ISR.

**High Watermark**: The highest offset that has been replicated to ALL ISR members. Consumers should only see records up to the high watermark — this prevents reading data that might be lost if the leader crashes.

### Step 5.4: Implement Ack Levels

**`acks=1`**: Return success as soon as the leader writes to its local log. Fast but you can lose data if the leader dies before followers catch up.

**`acks=all`**: Return success only after ALL ISR members have replicated the record. The leader must wait until `HighWatermark >= record's offset`.

Implementation — use a `sync.Cond` to avoid polling:
```
type PartitionState struct {
    // ... existing fields ...
    hwCond *sync.Cond  // signaled whenever HighWatermark advances
}

func (b *Broker) Produce(tp TopicPartition, record Record, acks int) (uint64, error) {
    offset := b.log.Append(tp, record)

    if acks == 1 {
        return offset, nil
    }

    // acks=all: wait for high watermark to advance past this offset
    ps := b.partitionState[tp]
    ps.hwCond.L.Lock()
    defer ps.hwCond.L.Unlock()

    deadline := time.Now().Add(10 * time.Second)
    for ps.HighWatermark < offset {
        // Wait with timeout — hwCond is signaled by the replication ack handler
        // whenever it updates HighWatermark
        if time.Now().After(deadline) {
            return 0, errors.New("replication timeout")
        }
        ps.hwCond.Wait()
    }
    return offset, nil
}

// In the replication ack handler, after updating HighWatermark:
// ps.hwCond.Broadcast()
```

### Step 5.5: Add Replication API Endpoints

On the leader broker:
**`GET /replicate?topic=X&partition=Y&offset=Z`** — Returns records starting from offset Z (used by followers)
**`POST /replicate/ack`** — Follower reports its current end offset

### Step 5.6: Update Fetch for High Watermark

Consumers should only see committed (fully-replicated) data:
```
GET /fetch?topic=X&partition=Y&offset=Z
→ only return records where offset < HighWatermark
```

### Step 5.7: Test It

- Produce with acks=1, verify the leader returns immediately
- Produce with acks=all, verify the leader waits for followers
- Produce records, verify followers have identical data
- Kill a follower, verify it catches up when it comes back
- Kill the leader (don't promote yet — that's Week 8), verify followers have data up to the high watermark
- Verify consumers only see records up to the high watermark

### Deliverable
Followers replicate from leaders. acks=1 and acks=all work. High watermark limits consumer visibility.

---

## Week 6: Consumer Groups v1

**Goal**: Multiple consumers form a group. The controller assigns partitions to consumers within the group. Consumers commit their offsets.

### Concepts to Learn
- Consumer group protocol
- Partition assignment strategies (round-robin)
- Offset management (committed vs. current position)

### Step 6.1: How Consumer Groups Work

Without groups: each consumer manually specifies which partition to read. Doesn't scale — you have to manage assignment yourself.

With groups:
1. Each consumer joins a group by name (e.g., "payment-processors")
2. The controller assigns partitions across group members
3. Each partition is assigned to exactly one consumer in the group
4. Consumers commit their offsets periodically so they can resume after restart

### Step 6.2: Controller-Side Group Management

```
type ConsumerGroup struct {
    GroupID     string
    Members    map[string]*ConsumerInfo  // consumerID → info
    Assignment map[TopicPartition]string // partition → consumerID
    Topic      string
}

type ConsumerInfo struct {
    ID            string
    LastHeartbeat time.Time
}
```

### Step 6.3: Join Protocol

**`POST /group/join`**
```json
Request:  { "groupId": "payment-processors", "topic": "transactions", "consumerId": "consumer-1" }
Response: { "consumerId": "consumer-1", "assignment": [{"topic": "transactions", "partition": 0}, ...] }
```

When a consumer joins:
1. Add it to the group's member list
2. Re-run partition assignment (round-robin across all members)
3. Return the new assignment to all members

**Round-robin assignment**:
```
partitions = [0, 1, 2, 3, 4, 5]
consumers  = [c1, c2, c3]
result:
  c1 → [0, 3]
  c2 → [1, 4]
  c3 → [2, 5]
```

### Step 6.4: Heartbeats

**`POST /group/heartbeat`**
```json
Request:  { "groupId": "...", "consumerId": "..." }
Response: { "status": "ok" }
```

Consumers send heartbeats every 5 seconds. If the controller hasn't heard from a consumer in 30 seconds, it's considered dead (handled in Week 7).

### Step 6.5: Offset Commits

Consumers periodically commit the offset they've processed up to:

**`POST /group/commit`**
```json
Request:  { "groupId": "...", "consumerId": "...", "offsets": {"transactions-0": 1500, "transactions-3": 2300} }
Response: { "status": "ok" }
```

The controller stores committed offsets:
```
type OffsetStore struct {
    offsets map[string]map[TopicPartition]uint64  // groupID → partition → offset
}
```

**`GET /group/committed?groupId=...&topic=...&partition=...`**
→ Returns the last committed offset for a partition in a group

### Step 6.6: Consumer Client Logic

A consumer client does:
```
1. POST /group/join → get my assigned partitions
2. For each assigned partition:
   a. GET /group/committed → get my starting offset
   b. Loop:
      - GET /fetch → get records from that offset
      - Process records
      - POST /group/commit → commit new offset
      - POST /group/heartbeat → stay alive
```

### Step 6.7: Test It

- 1 consumer joins, gets all partitions
- 3 consumers join, partitions split evenly
- Commit offset, restart consumer, verify it resumes from committed offset
- Two groups on same topic get independent assignments (both see all partitions)

### Deliverable
Consumer groups with round-robin partition assignment and offset commits.

---

## Week 7: Rebalance + Failure Handling

**Goal**: Detect dead consumers and redistribute their partitions. Consumers that rejoin pick up from committed offsets.

### Concepts to Learn
- Rebalance triggers and protocols
- Graceful vs. ungraceful consumer shutdown
- At-least-once delivery semantics

### Step 7.1: When Does Rebalance Happen?

Rebalance is triggered when:
1. A new consumer joins the group
2. A consumer leaves the group (graceful shutdown)
3. A consumer's heartbeat times out (crash/network partition)

### Step 7.2: Dead Consumer Detection

Controller background loop:
```
func (c *Controller) checkConsumerHealth() {
    for {
        for _, group := range c.groups {
            for id, member := range group.Members {
                if time.Since(member.LastHeartbeat) > 30*time.Second {
                    // Consumer is dead
                    delete(group.Members, id)
                    c.rebalance(group)
                }
            }
        }
        sleep(10 * time.Second)
    }
}
```

### Step 7.3: Rebalance Protocol

When rebalance is triggered:
1. Compute new assignment (round-robin across surviving members)
2. Store the new assignment
3. Each consumer's next heartbeat response includes a `rebalance: true` flag
4. On receiving this flag, the consumer calls `/group/join` again to get its new assignment
5. The consumer stops reading from old partitions and starts reading from new ones

```json
// Heartbeat response when rebalance is needed:
{ "status": "rebalance", "generation": 5 }
```

**Generation ID**: Incremented on every rebalance. Prevents stale consumers from committing offsets for partitions they no longer own.

### Step 7.4: Graceful Shutdown

Consumer sends a leave request before shutting down:
**`POST /group/leave`**
```json
Request:  { "groupId": "...", "consumerId": "..." }
```

This triggers an immediate rebalance rather than waiting for heartbeat timeout.

### Step 7.5: Resuming from Committed Offsets

After a rebalance, a consumer may receive a partition that was previously owned by a dead consumer. It must resume from the committed offset:

```
1. Get new assignment from /group/join
2. For each newly assigned partition:
   a. Look up the committed offset: GET /group/committed
   b. Start fetching from that offset
```

Records between the committed offset and the dead consumer's actual progress will be re-delivered. This gives you **at-least-once** delivery semantics.

### Step 7.6: Test It

- Start 3 consumers, kill one, verify its partitions are redistributed to survivors within heartbeat timeout
- After rebalance, verify the surviving consumers resume from committed offsets (no data loss)
- Verify no partition is left unassigned after rebalance
- Add a 4th consumer, verify partitions redistribute from 3 to 4 consumers
- Kill a consumer that hasn't committed recently, verify re-delivery of uncommitted records

### Chaos Script: `kill_consumer.sh`
```bash
#!/bin/bash
# Kill a random consumer process
CONSUMER_PID=$(pgrep -f "consumer" | shuf -n 1)
echo "Killing consumer PID: $CONSUMER_PID"
kill -9 $CONSUMER_PID
```

### Deliverable
Automatic rebalancing on consumer join/leave/death. At-least-once delivery guarantee.

---

## Week 8: Leader Failover + Polish

**Goal**: When a leader broker dies, the controller promotes an in-sync follower. Produce and fetch automatically reroute. Final polish and Docker setup.

### Concepts to Learn
- Leader election algorithms
- Fencing (preventing split-brain)
- Docker Compose for multi-process orchestration

### Step 8.1: Detecting a Dead Leader

The controller already tracks broker heartbeats (Week 4). When a broker dies:
```
func (c *Controller) onBrokerDeath(brokerID int) {
    // Find all partitions where this broker was the leader
    for tp, assignment := range c.assignments {
        if assignment.Leader == brokerID {
            c.electNewLeader(tp, assignment)
        }
    }
}
```

### Step 8.2: Leader Election

Choose the new leader from the ISR (in-sync replicas):
```
func (c *Controller) electNewLeader(tp TopicPartition, old *Assignment) {
    for _, followerID := range old.ISR {
        if c.brokers[followerID].Alive {
            // Promote this follower
            old.Leader = followerID
            old.Followers = remove(old.Followers, followerID)
            old.Followers = append(old.Followers, old.Leader) // old leader becomes follower
            c.notifyBrokers(tp, old)  // tell all brokers about the change
            return
        }
    }
    // No ISR member alive — partition is unavailable
    log.Printf("CRITICAL: partition %v has no available replicas", tp)
}
```

**Why ISR only?** A follower not in the ISR may be missing recent records. Promoting it could lose data.

### Step 8.3: Broker Handles Role Change

When a broker receives notification that it's now the leader for a partition:
1. Stop the follower fetch loop for that partition
2. Start accepting produce requests
3. Other followers start replicating from this new leader

When a broker learns it's now a follower:
1. Stop accepting produce requests for that partition
2. Start the follower fetch loop from the new leader

### Step 8.4: Fencing Stale Leaders

Edge case: the old leader isn't actually dead, just slow. It might still be accepting writes.

**Solution — Epoch/Fencing Token**:
- Each leadership assignment includes an epoch number (incremented on every leader change)
- Followers include the epoch when fetching from the leader
- A leader rejects requests with a newer epoch (it knows it's been superseded)
- Produce requests include the epoch; if the broker's epoch is stale, it rejects the request

### Step 8.5: Batched Produce and Fetch

Single-record produce/fetch has high overhead from per-request serialization, network round trips, and fsync calls. Batching amortizes these costs.

**Batched Produce API**:
```
POST /produce
{
  "topic": "orders",
  "records": [
    {"key": "user-1", "value": "...", "eventTime": 1700000000000},
    {"key": "user-2", "value": "...", "eventTime": 1700000000001},
    {"key": "user-3", "value": "...", "eventTime": 1700000000002}
  ],
  "acks": 1
}
Response: { "offsets": [{"partition": 0, "offset": 100}, {"partition": 1, "offset": 200}, ...] }
```

**Implementation**:
1. Group records by partition (hash each key)
2. For each partition, append all records in one write + single fsync (instead of fsync per record)
3. For acks=all, wait for the high watermark to cover all records in the batch
4. Return all (partition, offset) pairs in one response

**Batched Fetch API**:
```
GET /fetch?topic=orders&partition=0&offset=100&maxBytes=1048576
```

Instead of `max=N` records, use `maxBytes` — return as many records as fit within the byte limit. This is more efficient because it adapts to varying record sizes.

**Why this matters for interviews**: Batching is one of the main reasons Kafka achieves high throughput. Being able to explain the trade-off (latency vs. throughput) and how `linger.ms` works in real Kafka (the producer waits a short time to accumulate a batch before sending) shows operational depth.

### Step 8.6: Docker Compose Setup

```yaml
services:
  controller:
    build: .
    command: /app/controller
    ports:
      - "8080:8080"

  broker1:
    build: .
    command: /app/broker --id=1 --port=9091 --controller=controller:8080
    ports:
      - "9091:9091"
    volumes:
      - broker1-data:/data

  broker2:
    build: .
    command: /app/broker --id=2 --port=9092 --controller=controller:8080
    ports:
      - "9092:9092"
    volumes:
      - broker2-data:/data

  broker3:
    build: .
    command: /app/broker --id=3 --port=9093 --controller=controller:8080
    ports:
      - "9093:9093"
    volumes:
      - broker3-data:/data

volumes:
  broker1-data:
  broker2-data:
  broker3-data:
```

### Step 8.7: Chaos Script: `kill_broker.sh`
```bash
#!/bin/bash
BROKER=${1:-broker1}
echo "Killing $BROKER..."
docker-compose kill $BROKER
echo "Waiting 10 seconds..."
sleep 10
echo "Verifying failover via controller metadata..."
curl -s http://localhost:8080/metadata | python3 -m json.tool
echo ""
echo "Restarting $BROKER..."
docker-compose start $BROKER
sleep 5
echo "Verifying catch-up..."
curl -s http://localhost:8080/metadata | python3 -m json.tool
```

### Step 8.8: End-to-End Integration Test

Write a script that:
1. Starts the cluster (3 brokers + controller)
2. Creates a topic with 4 partitions, replication factor 3
3. Produces 10,000 records with acks=all
4. Starts a consumer group with 2 consumers
5. While consumers are running, kills the leader broker for partition 0
6. Verifies: failover happens, new leader is promoted, consumers resume, no records are lost
7. Restarts the killed broker, verifies it rejoins as a follower and catches up

### Step 8.9: Observability — `/metrics` Endpoint

Expose JSON counters on each broker:
```json
{
  "produce_qps": 150,
  "fetch_qps": 300,
  "partitions": {
    "orders-0": {
      "role": "leader",
      "leo": 50000,
      "high_watermark": 49980,
      "followers": {
        "broker-2": { "offset": 49975, "in_sync": true },
        "broker-3": { "offset": 49980, "in_sync": true }
      }
    }
  },
  "consumer_groups": {
    "payment-processors": {
      "committed_offsets": { "orders-0": 49500 },
      "lag": { "orders-0": 480 }
    }
  }
}
```

**Consumer lag** = High Watermark - Committed Offset. This is the single most important operational metric for any Kafka deployment.

### Step 8.10: Write the README

Document:
1. Architecture diagram
2. How to run: `docker-compose up`
3. How to produce/consume (curl examples)
4. Design invariants (monotonic offsets, at-least-once delivery, ISR-based failover)
5. Failure modes and how the system handles them
6. Demo commands showing failover and recovery

### Final Deliverables Checklist

- [ ] `docker-compose up` starts 3 brokers + 1 controller
- [ ] Producer writes records and receives (partition, offset) acknowledgment
- [ ] Batched produce/fetch endpoints work with multiple records per request
- [ ] Log retention deletes old segments by age or total size
- [ ] Consumer reads from offsets; consumer groups split partitions across members
- [ ] Leader failover: kill a broker, a new leader is elected, produce/fetch continue
- [ ] Controller persists metadata to disk and recovers on restart
- [ ] `/metrics` endpoint exposes lag, replication stats, QPS
- [ ] README documents invariants, failure modes, and demo commands
- [ ] Chaos scripts: `kill_broker.sh`, `kill_consumer.sh`
- [ ] All tests pass: monotonic offsets, durability, replication catch-up, rebalance

---

## Interview Talking Point: Why Not Exactly-Once?

This project achieves **at-least-once** delivery. Records may be re-delivered after a consumer crash (between processing and committing the offset). Understanding why **exactly-once** is harder is a common interview topic:

**What you'd need for exactly-once on the producer side (idempotent producer)**:
- Assign each producer a unique Producer ID (PID) and a sequence number per partition
- The broker deduplicates by (PID, partition, sequence number) — if it sees a duplicate, it returns the existing offset instead of appending again
- This handles retries (producer crashes after sending but before receiving the ack)

**What you'd need for exactly-once on the consumer side (transactional consume-process-produce)**:
- Consumers commit their offset and write their output in a single atomic transaction
- This requires the offset store and the output destination to participate in the same transaction (e.g., both in Kafka, or using a two-phase commit with an external database)

**Why we skip it**: Implementing idempotent producers and transactions adds significant complexity. In practice, most systems are designed to tolerate at-least-once delivery by making consumers idempotent (processing the same record twice produces the same result). Being able to explain this trade-off is more valuable in interviews than actually building exactly-once.

---

## Recommended Reading

### DDIA Reading Plan (aligned to weeks)

**Before starting (read first):**
- **Chapter 3: Storage and Retrieval** — Log-structured storage, SSTables, indexing. Directly maps to Weeks 1-2 (append-only log, segments, sparse index). Most important chapter to read first.
- **Chapter 5: Replication** (first half) — Leader/follower replication model. Read the concepts before Week 5 so you understand *why* before you build it.

**During implementation:**

| Week | DDIA Chapter | Why |
|------|-------------|-----|
| Week 3 (Partitions) | **Chapter 6: Partitioning** | Hash-based partitioning, rebalancing strategies |
| Week 4 (Controller) | **Chapter 8: The Trouble with Distributed Systems** | Failure detection, timeouts, heartbeats — why distributed coordination is hard |
| Week 5 (Replication) | **Chapter 5: Replication** (re-read fully) | ISR, consistency guarantees, replication lag |
| Week 7 (Rebalance) | **Chapter 9: Consistency and Consensus** | Leader election, fencing tokens, epoch numbers |
| Week 8 (Failover) | **Chapter 9** (re-read) | Fencing and split-brain prevention |

**After finishing Project 1:**
- **Chapter 11: Stream Processing** — The "why Kafka exists" big picture; sets up Project 2 (Streaming Engine)

**Skip:** Chapters 1-2 (too high-level), 4 (encoding — you'll learn by doing), 7 (transactions — not needed until exactly-once), 10 (batch processing — not relevant)

### Other Resources

- **Kafka documentation: Design section** — https://kafka.apache.org/documentation/#design
- **The Log (Jay Kreps)** — https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying — the foundational essay on why logs matter
- **Kafka ISR explanation** — understand in-sync replicas deeply, it's asked in every distributed systems interview
