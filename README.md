# Rust Alternator client

## Glossary

- Alternator.
A DynamoDB API implemented on top of ScyllaDB backend.
Unlike AWS DynamoDB’s single endpoint, Alternator is distributed across multiple nodes.
Could be deployed anywhere: locally, on AWS, on any cloud provider.

- Client-side load balancing.
A method where the client selects which server (node) to send requests to,
rather than relying on a load balancing service.

- DynamoDB.
A managed NoSQL database service by AWS, typically accessed via a single regional endpoint.

- AWS Rust SDK.
The official AWS SDK for the Rust programming language, used to interact with AWS services like DynamoDB. Available [here](https://github.com/awslabs/aws-sdk-rust/tree/main/sdk/dynamodb).

- DynamoDB/Alternator Endpoint.
The base URL a client connects to.
In AWS DynamoDB, this is typically something like http://dynamodb.us-east-1.amazonaws.com.
In Alternator, it is the address of any node in the cluster.

- Datacenter (DC).
A physical or logical grouping of racks.
On Scylla Cloud in regular setup it represents cloud provider region where nodes are deployed.

- Rack.
A logical grouping akin to an availability zone within a datacenter.
On Scylla Cloud in regular setup it represents cloud provider availability zone where nodes are deployed.

## Introduction

This crate is a thin wrapper for the AWS Rust SDK that builds DynamoDB clients which load-balance across Alternator nodes.
Includes optimizations for Lightweight Transactions (LWTs), request compression, and header stripping.

## Using the crate


Add the crate to your `Cargo.toml`:

```toml
[dependencies]
alternator-driver = "0.1.0"
aws-sdk-dynamodb = "1"
tokio = { version = "1", features = ["full"] }
```
> **Note**: This crate is not yet published to crates.io. Depend on it via the GitHub URL.

The crate is a drop-in alongside the AWS Rust SDK for DynamoDB. If you have code using `aws_sdk_dynamodb::Client`, the migration is essentially three name swaps:

```rust
use alternator_driver::*;              // <-- new import
use aws_sdk_dynamodb::config::*;
use aws_sdk_dynamodb::types::*;

#[tokio::main]
async fn main() {
    // Build an AlternatorConfig instead of an aws_sdk_dynamodb::Config.
    let config = AlternatorConfig::builder() // <-- was aws_sdk_dynamodb::Config::builder()
        .endpoint_url("http://localhost:8000")
        .behavior_version(BehaviorVersion::latest())
        .allow_no_auth()
        .build();

    // Build an AlternatorClient instead of an aws_sdk_dynamodb::Client.
    let client = AlternatorClient::from_conf(config); // <-- was aws_sdk_dynamodb::Client::from_conf

    // From here on, the API is identical to the AWS SDK.
    client
        .put_item()
        .table_name("ExampleTable")
        .item("ExampleKey", AttributeValue::S("key".into()))
        .item("ExampleAttribute", AttributeValue::S("value".into()))
        .send()
        .await
        .unwrap();
}
```

## Load balancing
> **Note:** load balancing is not yet merged, the changes described here are implemented in PRs #30, #31, and #32.

A single Alternator cluster typically consists of multiple nodes, any of which can serve any request. This crate distributes requests across the live nodes of the cluster rather than sending everything to one address. There's no separate load-balancer process, routing happens entirely client-side.

### Seed hosts vs endpoint URL

The simplest way to construct a client is with `endpoint_url`, the same field the AWS SDK uses:

```rust
let config = AlternatorConfig::builder()
    .endpoint_url("http://10.0.0.1:8043")
    .behavior_version(BehaviorVersion::latest())
    .allow_no_auth()
    .build();
```

The host in the URL is treated as a *seed*: the client immediately calls `/localnodes` on that node to discover the full cluster, and from that point onward all requests fan out across the discovered nodes. The endpoint URL is never used for actual data-plane traffic after discovery completes.

For the client to instantly know where to distribute requests, or for deployments where the seed node might be down at startup time, you can pass multiple seed addresses directly along with the scheme and the port:

```rust
let config = AlternatorConfig::builder()
    .scheme("http")
    .port(8043)
    .seed_hosts([
        "10.0.0.1",
        "10.0.0.2",
        "10.0.0.3",
    ])
    .behavior_version(BehaviorVersion::latest())
    .allow_no_auth()
    .build();
```

The client tries each seed in turn until one responds successfully to `/localnodes`. Once discovery succeeds, the seed list is no longer consulted (except as fallback if all currently-known nodes become unreachable).

### Node discovery

The client maintains a list of live nodes that it refreshes in the background. The refresh has two cadences:

- **Active** (default 1s): used while the client is being called regularly.
- **Idle** (default 60s): used when no caller has touched the client recently.

Both intervals are configurable:

```rust
.active_interval(500)   // milliseconds
.idle_interval(30_000)
```

The refresh task runs in the background for the lifetime of the client. It terminates automatically when the client is dropped.

### Routing scope

By default, the client uses every Alternator node returned by `/localnodes`. For deployments spanning multiple datacenters or racks, you usually want requests to stay within a specific datacenter — or within a specific rack of a specific datacenter — to minimize cross-zone latency and bandwidth.

This is configured via `RoutingScope`:

```rust
use alternator_driver::RoutingScope;

// Restrict to a single datacenter:
let scope = RoutingScope::from_datacenter("dc1".to_string());

// Restrict to a specific rack within a datacenter:
let scope = RoutingScope::from_rack("dc1".to_string(), "rack1".to_string());

// Don't restrict (the default)
let scope = RoutingScope::from_cluster();

let config = AlternatorConfig::builder()
    .endpoint_url("http://10.0.0.1:8043")
    .routing_scope(scope)
    .behavior_version(BehaviorVersion::latest())
    .allow_no_auth()
    .build();
```

> **Note**: the default scope, `RoutingScope::from_cluster()`, does not actually mean "every node in the cluster." It means "don't filter by the datacenter or the rack at all." Alternator interprets this as "return nodes from the same datacenter as me," so the effective scope ends up being the datacenter of whichever node served the `/localnodes` request — typically the seed.

### Scope fallbacks

A scope can be narrow enough that no nodes match it — for example, a specific rack that has no live nodes at the moment. In that case the client uses the configured fallback scope instead. Fallbacks are explicit and chainable:

```rust
// Rack -> Datacenter -> Cluster fallback chain
let scope = RoutingScope::from_rack("datacenter1".to_string(), "rack1".to_string())
    .with_fallback(RoutingScope::from_datacenter("datacenter1".to_string()))
    .with_fallback(RoutingScope::from_cluster());

// Rack -> Another Rack -> Datacenter -> Cluster
let scope = RoutingScope::from_rack("datacenter1".to_string(), "rack1".to_string())
    .with_fallback(RoutingScope::from_rack("datacenter1".to_string(), "rack2".to_string()))
    .with_fallback(RoutingScope::from_datacenter("datacenter1".to_string()))
    .with_fallback(RoutingScope::from_cluster());
```

The first one says: "prefer rack1 of datacenter1, if no nodes there, use any node in datacenter1, if still nothing, use any node Alternator returns." The client walks the chain from preferred to broadest, picking the first scope that has live nodes.

Each `.with_fallback(...)` call appends to the end of the chain, so the order in code matches the order of preference. Fallbacks ideally should be broader than or equal to the previous scope.

### Load balancing strategies

For every request, the client picks a node and rewrites the request URI to point at that node before signing. The default strategy is round-robin across the live nodes — each request goes to the next node in the rotation, and an SDK-level retry moves on to a different node.

Round-robin is the right default for the vast majority of workloads. For workloads that perform many LWTs against the same partition keys, see [Key route affinity](#key-route-affinity) below.

## Key route affinity

When using Lightweight Transactions (LWT) in ScyllaDB/Alternator, routing requests for the same partition key to the same coordinator node can significantly improve performance. This is because LWT operations require consensus among replicas, and using the same coordinator reduces coordination overhead. KeyRouteAffinity is a way to reduce this overhead by ensuring that two queries targeting the same partition key will be routed to the same coordinator. Instead of round-robin random selection of nodes, it provides a deterministic mapping from partition key to coordinator.

### Alternator write isolation modes

Alternator supports different write isolation modes configured via `alternator_write_isolation`:

- **`always`**: All write operations use LWT (Paxos consensus). Maximum consistency but higher latency.
- **`only_rmw_uses_lwt`**: Only Read-Modify-Write operations (UpdateItem with conditions, DeleteItem with conditions) use LWT. This is the **recommended setting** for most use cases.
- **`forbid_rmw`**: LWTs are completely disabled. Conditional operations will fail.
- **`unsafe_rmw`**: Unsafe - does not use LWT for RMW operations.

### When to use KeyRouteAffinity

Enable KeyRouteAffinity when:
- Your Alternator cluster is configured with `alternator_write_isolation: only_rmw_uses_lwt` (use `KeyRouteAffinityType::Rmw`) or `always` (use `KeyRouteAffinityType::AnyWrite`)
- You perform conditional updates/deletes on the same items repeatedly
- You want to optimize LWT performance by ensuring the same coordinator handles requests for the same partition key

### Configuration options

There are three KeyRouteAffinity modes:

1. **`KeyRouteAffinityType::None`** (default): Disabled. Requests are distributed randomly across nodes.
2. **`KeyRouteAffinityType::Rmw`**: Enables route affinity for conditional write operations, operations that need read before write.
3. **`KeyRouteAffinityType::AnyWrite`**: Enables route affinity for all write operations.

### Automatic partition key discovery

When a request targets a table whose partition key the driver hasn't seen before, the driver calls `DescribeTable` once in the background to retrieve the partition key name. Subsequent requests for that table use the cached name. While discovery is in flight, that table's requests fall back to round-robin routing — they're not delayed waiting for the partition key to be discovered.

To skip discovery for a known set of tables, pre-configure their partition key names — see the configuration examples below.

### Configuring affinity

The simplest case: pass an affinity mode directly to the client builder.

```rust
use alternator_driver::{AlternatorConfig, AlternatorClient, KeyRouteAffinityType};

let client = AlternatorClient::from_conf(
    AlternatorConfig::builder()
        .endpoint_url("http://10.0.0.1:8043")
        .key_route_affinity(KeyRouteAffinityType::Rmw)
        .behavior_version(BehaviorVersion::latest())
        .allow_no_auth()
        .build(),
);
```

This enables affinity in RMW mode with no pre-configured tables. The driver discovers partition key names on first use of each table.

To pre-configure the partition key names for specific tables and skip the initial `DescribeTable` lookup, build a `KeyRouteAffinityConfig` and pass that instead:

```rust
use alternator_driver::{AlternatorConfig, AlternatorClient, KeyRouteAffinityConfig, KeyRouteAffinityType};

let affinity = KeyRouteAffinityConfig::builder()
    .with_type(KeyRouteAffinityType::Rmw)
    .with_pk_info("users", "user_id")
    .with_pk_info("orders", "order_id")
    .build();

let client = AlternatorClient::from_conf(
    AlternatorConfig::builder()
        .endpoint_url("http://10.0.0.1:8043")
        .key_route_affinity(affinity)
        .behavior_version(BehaviorVersion::latest())
        .allow_no_auth()
        .build(),
);
```
`with_pk_info` can be called multiple times to register more tables. Tables not pre-configured will be discovered on first use as usual.

`.key_route_affinity(...)` accepts either a `KeyRouteAffinityType` (for the simple case) or a full `KeyRouteAffinityConfig` (for pre-configured tables). The two forms are interchangeable at the call site — pick whichever matches your needs.

## Header stripping

By default, the AWS Rust SDK attaches a number of headers to every DynamoDB request — some are required (`Host`, `Authorization`, `X-Amz-Date`, etc.), others are SDK metadata that Alternator doesn't use (`User-Agent` flavors, internal telemetry, retry information). For a small client-side optimization, this crate strips non-essential headers before transmission, keeping only the ones Alternator actually needs:
- `host`
- `x-amz-target`
- `content-length`
- `accept-encoding`
- `content-encoding`
- `authorization`
- `x-amz-date`

This is on by default, you can disable it if needed:

```rust
let client = AlternatorClient::from_conf(
    AlternatorConfig::builder()
        .endpoint_url("http://10.0.0.1:8043")
        .enforce_header_whitelist(false)
        .behavior_version(BehaviorVersion::latest())
        .allow_no_auth()
        .build(),
);
```

## Request compression

Alternator accepts compressed request bodies, which can significantly reduce the bandwidth for write-heavy workloads (especially `BatchWriteItem` and large `PutItem` payloads).

> **Note**: currently the default is `CompressionAlgorithm::Gzip` with 1024 bytes threshold, but it will be changed to disabled.

To enable request compression, pass a `RequestCompression` configuration:

```rust
use alternator_driver::{AlternatorConfig, AlternatorClient, RequestCompression, CompressionAlgorithm, CompressionLevel};

let client = AlternatorClient::from_conf(
    AlternatorConfig::builder()
        .endpoint_url("http://10.0.0.1:8043")
        .request_compression(RequestCompression::enabled(
            CompressionAlgorithm::Gzip,
            CompressionLevel::default(),
            1024, // body-size threshold in bytes
        ))
        .behavior_version(BehaviorVersion::latest())
        .allow_no_auth()
        .build(),
);
```

To disable compression, use
```rust
.request_compression(RequestCompression::disabled())
```

The three parameters are the compression algorithm, the compression level, and the body-size threshold in bytes. Requests with bodies smaller than the threshold are sent uncompressed. Setting the threshold to zero compresses every request.

Two algorithms are supported: `CompressionAlgorithm::Gzip` (sends `Content-Encoding: gzip`) and `CompressionAlgorithm::Zlib` (sends `Content-Encoding: deflate`). Level is `flate2`'s `Compression` type re-exported as `CompressionLevel` — `CompressionLevel::default()` (level 6) is a reasonable balance of speed and ratio; `CompressionLevel::best()` (level 9) maximizes compression at the cost of CPU; `CompressionLevel::fast()` (level 1) is the opposite.

## Per-operation override

Some Alternator-specific settings can be overridden on a per-request basis, using the same `.customize()` mechanism the AWS SDK provides for its own configuration. Call `.alternator_config_override()` on the customizable operation and pass an `AlternatorConfig::builder()` with the settings you want to change for that single request:

```rust
#[tokio::main]
async fn main() {
    // ...
    client
        .put_item()
        .table_name("ExampleTable")
        .item("ExampleKey", AttributeValue::S("ExampleItemKey".into()))
        .item("ExampleAttribute", AttributeValue::S("ExampleItem".into()))
        .customize()
        
        .alternator_config_override(
            AlternatorConfig::builder()
                .request_compression(RequestCompression::disabled())
        )
        .send()
        .await
        .unwrap();
}
```

`.alternator_config_override()` is the Alternator-specific equivalent of the AWS SDK's `.config_override()`. The two work alongside each other — use `.config_override()` for SDK-level settings (retry behavior, timeouts) and `.alternator_config_override()` for Alternator-specific settings.

> **Note**: load-balancing and endpoint settings cannot be overridden per-operation. They take effect only when the client is constructed. Per-operation override is for settings that apply to individual request processing — compression and header stripping.