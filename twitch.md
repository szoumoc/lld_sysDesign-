# Twitch System Design
 Follow links to understand and revise for overall understanding:

 https://www.tldraw.com/f/WRDed_Mqb5bvWjv7yJeMj?d=v434.467.1199.1378.page
 https://youtu.be/MWjQs9I7clo?si=gYy_GzM7nX11klVK
 https://www.cloudflare.com/learning/video/what-is-adaptive-bitrate-streaming/
 https://www.mux.com/articles/rtmp-streaming-protocol
## Requirements

### Functional Requirements

* Streamers should be able to start live streams.
* Viewers should be able to watch streams with low latency.
* Support adaptive bitrate streaming.
* Real-time chat per channel.
* Store VODs for later playback.
* Support millions of concurrent viewers.

### Non-Functional Requirements

* High availability (99.99%+)
* Horizontal scalability
* Low latency
* Fault tolerance
* Disaster recovery
* Multi-region support

---

# Capacity Estimation

Assumptions:

| Metric              | Value      |
| ------------------- | ---------- |
| DAU                 | 10 Million |
| Concurrent Viewers  | 1 Million  |
| Concurrent Streams  | 100,000    |
| Avg Stream Bitrate  | 5 Mbps     |
| Avg Stream Duration | 2 Hours    |

---

## Ingestion Bandwidth

```text
100,000 streams × 5 Mbps

= 500,000 Mbps

= 500 Gbps

= 0.5 Tbps
```

Ingress Layer must handle approximately **0.5 Tbps**.

---

## Viewer Delivery Bandwidth

```text
1,000,000 viewers × 5 Mbps

= 5,000,000 Mbps

= 5 Tbps
```

CDNs are mandatory.

---

## HLS Segment Generation

Assume:

```text
2 second segments
```

Per stream:

```text
3600 / 2 = 1800 segments/hour
```

For 100K streams:

```text
1800 × 100,000

= 180 Million Segments/Hour
```

---

## Storage

Single Stream:

```text
5 Mbps
2 Hours
```

Storage:

```text
5 × 7200

= 36,000 Mb

≈ 4.5 GB
```

100K streams/day:

```text
≈ 450 TB/day
```

With multiple renditions:

```text
1080p
720p
480p
360p
```

Storage can exceed:

```text
1 PB/day
```

---

# High Level Architecture

```text
                    ┌──────────────────┐
                    │     Streamer     │
                    └────────┬─────────┘
                             │
                           RTMP
                             │
                             ▼
                ┌────────────────────────┐
                │ Regional Ingest Layer  │
                │ Mumbai / Tokyo / FRA   │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ Authentication Service │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ Session Management     │
                │ Redis / Cassandra      │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ Transcoding Cluster    │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ HLS Packaging Service  │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ Object Storage (S3)    │
                └──────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────────┐
                │ CDN Edge Locations     │
                └──────────┬─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Viewers    │
                    └─────────────┘
```

---

# Video Ingestion

## Why RTMP?

RTMP is still widely supported by:

* OBS
* Streamlabs
* XSplit

RTMP is used only for:

```text
Streamer → Twitch
```

It is NOT used for playback.

---

## Regional Ingest Points

To reduce latency:

```text
India     → Mumbai
Japan     → Tokyo
Germany   → Frankfurt
US East   → Virginia
```

Benefits:

* Lower latency
* Reduced packet loss
* Faster reconnects

---

## Stream Key Authentication

Each streamer owns a stream key.

```text
sk_live_a8fj29x8...
```

Flow:

```text
OBS
  │
  │ Stream Key
  ▼
Ingest Server
  │
  ▼
Auth Service
  │
  ▼
Database
```

Valid key:

```text
200 OK
```

Invalid key:

```text
403 Forbidden
```

Authentication occurs only once during session establishment.

---

# Session Management

## Problem

Internet disconnects are common.

Without session management:

```text
Offline
Online
Offline
Online
```

would occur repeatedly.

---

## Solution

Maintain a grace period.

```text
Disconnect
    │
    ▼
Keep Session Alive
    │
30-90 Seconds
```

If reconnect happens within grace period:

```text
Resume Existing Stream
```

instead of creating a new stream.

---

# Shared Session Store

```text
          ┌──────────────┐
          │    Redis     │
          └──────┬───────┘
                 │
      ┌──────────┴──────────┐
      │                     │
      ▼                     ▼
 Mumbai Ingest      Singapore Ingest
```

Session State:

```text
session_id
stream_id
channel_id
status
generation
owner
```

Any ingest node can resume a stream.

---

# Preventing Duplicate Streams

## Problem

Both Mumbai and Singapore may believe they own the stream.

```text
Mumbai     → Active
Singapore  → Active
```

Result:

```text
Duplicate Stream
```

---

## Lease Ownership

Store ownership in Redis.

```text
stream_id = 123

owner = mumbai-7

lease_expiry = 12:00:30
```

Only owner can push frames.

---

## Generation Numbers

```text
stream_id = 123
generation = 18
```

Every ownership transfer:

```text
generation++
```

Old packets:

```text
generation = 17
```

are discarded.

---

# Video Processing Pipeline

## Transcoding

Input:

```text
1080p @ 6 Mbps
```

Output:

```text
1080p @ 6 Mbps
720p  @ 3 Mbps
480p  @ 1 Mbps
360p  @ 500 Kbps
```

Supports adaptive bitrate streaming.

---

## HLS Packaging

Transcoded streams become:

```text
playlist.m3u8

segment1.ts
segment2.ts
segment3.ts
```

Master Playlist:

```text
1080p
720p
480p
360p
```

Player automatically switches qualities.

---

# Storage Design

Different workloads require different storage systems.

---

## Video Storage

Storage:

```text
Amazon S3
Google Cloud Storage
Azure Blob Storage
```

Stores:

```text
segment1.ts
segment2.ts
segment3.ts
```

Why?

* Cheap
* Durable
* Infinite Scale

---

## Metadata Storage

Stores:

```text
Stream Title
Category
Language
Viewer Count
Status
```

Technology:

```text
Cassandra
DynamoDB
Bigtable
```

---

## User Storage

Stores:

```text
Users
Followers
Subscriptions
```

Technology:

```text
PostgreSQL
MySQL
CockroachDB
```

Strong consistency is important.

---

## Chat Storage

Pipeline:

```text
Chat Server
      │
      ▼
    Kafka
      │
      ▼
  Cassandra
```

Supports bursty chat traffic.

---

# Mapping Chat To Video

Use:

```text
stream_id
```

as the shared identifier.

Example:

S3:

```text
s3://streams/12345/1080p/segment1.ts
```

Cassandra:

```text
Partition Key

stream_id = 12345
```

Metadata:

```text
stream_id = 12345
```

Everything joins through:

```text
stream_id
```

---

# Chat Architecture

```text
Viewer
   │
WebSocket
   │
   ▼
Chat Gateway
   │
   ▼
Chat Service
   │
   ▼
Kafka
   │
   ▼
Cassandra
```

Benefits:

* Real-time delivery
* Horizontal scalability
* Message durability

---

# CDN Architecture

```text
                S3 Origin
                    │
                    ▼
          ┌─────────────────┐
          │      CDN        │
          └───────┬─────────┘
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
   India        Europe       US
```

Benefits:

* Lower latency
* Reduced origin load
* Global scalability

---

# Reliability

## Encoder Failover

```text
Ingest
   │
   ▼
Scheduler
   │
   ▼
Encoder-17
```

If encoder dies:

```text
Heartbeat Missing
```

Scheduler reassigns:

```text
Encoder-23
```

---

## Multi-Region Deployment

```text
Mumbai
Singapore
Tokyo
Frankfurt
Virginia
```

If Mumbai fails:

```text
OBS
  │
Reconnect
  │
Singapore
```

---

## Leader Election

Needed for:

* Stream Scheduler
* Metadata Coordinator
* Job Assignment

Example:

```text
Scheduler-A
Scheduler-B
Scheduler-C
```

Only one leader exists.

Implemented using:

```text
ZooKeeper
etcd
Consul
```

---

## Health Checks

### Liveness

```http
GET /health
```

Checks:

```text
Process Alive?
```

---

### Readiness

Checks:

```text
Can Serve Traffic?
```

Examples:

```text
Database Available?
GPU Available?
Storage Reachable?
```

Load balancer routes traffic only to ready instances.

---

## Disaster Recovery

### Video Replication

```text
Mumbai S3
     │
Replication
     │
Singapore S3
```

---

### Metadata Replication

```text
Mumbai
Singapore
Frankfurt
```

using Cassandra multi-region replication.

---

### Recovery Targets

RTO:

```text
Recovery Time Objective
```

Example:

```text
15 Minutes
```

RPO:

```text
Recovery Point Objective
```

Example:

```text
5 Seconds
```

---

# Technology Choices

| Component         | Technology          |
| ----------------- | ------------------- |
| Ingest            | RTMP                |
| Session Store     | Redis               |
| Metadata          | Cassandra           |
| Video Storage     | S3                  |
| Chat Queue        | Kafka               |
| Chat Storage      | Cassandra           |
| User Data         | PostgreSQL          |
| CDN               | CloudFront / Akamai |
| Leader Election   | etcd / ZooKeeper    |
| Transcoding       | GPU Cluster         |
| Delivery Protocol | HLS                 |

---

# Final Architecture

```text
Streamer
   │
 RTMP
   │
Regional Ingest
   │
Authentication
   │
Session Store
   │
Transcoding Cluster
   │
HLS Packaging
   │
S3 Origin
   │
CDN
   │
Viewers

Chat:
Viewer
   │
WebSocket
   │
Chat Gateway
   │
Kafka
   │
Cassandra
```

This design supports millions of concurrent viewers, hundreds of thousands of concurrent streams, global distribution, adaptive bitrate streaming, fault tolerance, and multi-region disaster recovery.
