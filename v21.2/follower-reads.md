---
title: Follower Reads
summary: To reduce latency for read queries, you can choose to have the closest replica serve the request using the follower reads feature.
toc: true
---

CockroachDB can provide faster reads in situations where you can afford to read data that is slightly stale.  These stale reads (known as _follower reads_)  are available in read-only transactions that use the [`AS OF SYSTEM TIME`](as-of-system-time.html) clause.

Normally, reads have to be serviced by a replica's [leaseholder](architecture/overview.html#architecture-leaseholder). This can be slow, since the leaseholder may be geographically distant from the gateway node that is issuing the query.

A follower read is a read taken from the closest [replica](architecture/overview.html#architecture-replica), regardless of the replica's leaseholder status. This can result in much better latency in [multi-region deployments](multiregion-overview.html).

For instructions showing how to use follower reads to get low latency, historical reads in a multi-region deployment, see the [Follower Reads Pattern](topology-follower-reads.html).

{{site.data.alerts.callout_info}}
This is an [{{ site.data.products.enterprise }} feature](enterprise-licensing.html).
{{site.data.alerts.end}}

## How follower reads work

Each CockroachDB range tracks a property called its [_closed timestamp_](architecture/transaction-layer.html#closed-timestamps), which means that no new writes can ever be introduced below that timestamp. The closed timestamp advances forward by some target interval behind the current time. If the range receives a write at a timestamp less than its closed timestamp, the write is forced to change its timestamp, which might result in a [transaction retry error](transaction-retry-error-reference.html).

With follower reads, any replica on a node can serve a read for a key as long as the time at which the operation is performed (i.e., the [`AS OF SYSTEM TIME`](as-of-system-time.html) value) is less than or equal to the node's closed timestamp.

When a gateway node in a cluster receives a request to read a key with a sufficiently old [`AS OF SYSTEM TIME`](as-of-system-time.html) value, it forwards the request to the closest node that contains a replica of the data–– whether it be a follower or the leaseholder.

CockroachDB provides the following types of follower reads:

- [Exact staleness reads](#exact-staleness-reads): These are historical reads as of a static, user-provided timestamp.  Most often, this is the timestamp value returned by the `follower_read_timestamp()` convenience [function](functions-and-operators.html).
- <span class="version-tag">New in v21.2:</span> [Bounded staleness reads](#bounded-staleness-reads): These use a dynamic, system-determined timestamp to minimize staleness while being more tolerant to replication lag than exact staleness reads.

## Watch the demo

<iframe width="560" height="315" src="https://www.youtube.com/embed/V--skgN_JMo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## When to use follower reads

As long as your [`SELECT` operations](select-clause.html) can tolerate reading stale data, follower reads can reduce read latencies and increase throughput.

Follower reads are "read-only" operations; they cannot be used in any way in read-write transactions.

Use exact staleness reads when:

- You need multi-statement reads inside transactions
- You can tolerate reading older data (at least 4.8 seconds in the past), to reduce the chance that the historical query timestamp is not quite old enough to prevent blocking on a conflicting write and thus being able to be served by a local replica

Use bounded staleness reads when you need:

- Absolute minimum staleness possible from single statement reads from follower replicas.  This is possible because the historical timestamp is chosen dynamically and the least stale timestamp that can be served locally without blocking is used
- Higher availability than is provided by exact staleness reads, specifically:
    - Serving a read with low latency from a local replica rather than a leaseholder
    - The ability to serve reads from local replicas even in the presence of a network partition or other failure event that prevents the SQL gateway from communicating with the leaseholder.  This is possible because bounded staleness reads do not require you (or the `follower_read_timestamp()` function) to guess at the maximum timestamp that can be served locally and risk being wrong.

You should **not** use follower reads when your application cannot tolerate reading stale data, since the results of follower reads may not reflect the latest writes against the tables you are querying.  However, for many applications, especially in multi-region deployments, there are opportunities to still do useful work with follower reads.

## Using follower reads

### Enable/disable follower reads

Use [`SET CLUSTER SETTING`](set-cluster-setting.html) to set `kv.closed_timestamp.follower_reads_enabled` to:

- `true` to enable follower reads _(default)_
- `false` to disable follower reads

{% include copy-clipboard.html %}
~~~ sql
> SET CLUSTER SETTING kv.closed_timestamp.follower_reads_enabled = false;
~~~

If you have follower reads enabled, you may want to [verify that follower reads are happening](#verify-that-follower-reads-are-happening).

{{site.data.alerts.callout_info}}
If follower reads are enabled, but the time-travel query is not using [`AS OF SYSTEM TIME`](as-of-system-time.html) far enough in the past (as defined by the [follower read timestamp](#run-queries-that-use-follower-reads)), CockroachDB does not perform a follower read. Instead, the read accesses the [leaseholder replica](architecture/overview.html#architecture-leaseholder). This adds network latency if the leaseholder is not the closest replica to the gateway node.
{{site.data.alerts.end}}

### Verify that follower reads are happening

To verify that your cluster is performing follower reads:

1. Make sure that [follower reads are enabled](#enable-disable-follower-reads).
2. Go to the [Custom Chart Debug Page in the DB Console](ui-custom-chart-debug-page.html) and add the metric `follower_read.success_count` to the time series graph you see there. The number of follower reads performed by your cluster will be shown.

XXX: DOES THIS METRIC APPLY TO BOTH EXACT AND BOUNDED STALENESS READS?

### Run queries that use follower reads

Any [`SELECT` statement](select-clause.html) with an appropriate [`AS OF SYSTEM TIME`](as-of-system-time.html) value can be a follower read (i.e., served by any replica).

For exact staleness reads, we've added the convenience [function](functions-and-operators.html) `follower_read_timestamp()`, which returns a [`TIMESTAMP`](timestamp.html) that provides a high probability of being served locally while not [blocking on conflicting writes](#follower-reads-and-long-running-writes).

Use this function in an `AS OF SYSTEM TIME` statement as follows:

``` sql
SELECT ... FROM ... AS OF SYSTEM TIME follower_read_timestamp();
```

### Run read-only transactions that use follower reads

You can set the [`AS OF SYSTEM TIME`](as-of-system-time.html) value for all operations in a read-only transaction:

```sql
BEGIN;

SET TRANSACTION AS OF SYSTEM TIME follower_read_timestamp();
SELECT ...
SELECT ...

COMMIT;
```

Note that follower reads are "read-only" operations; they cannot be used in any way in read-write transactions.

{{site.data.alerts.callout_success}}
Using the [`SET TRANSACTION`](set-transaction.html#use-the-as-of-system-time-option) statement as shown in the example above will make it easier to use the follower reads feature from [drivers and ORMs](install-client-drivers.html).

 To set `AS OF SYSTEM TIME follower_read_timestamp()` on all implicit and explicit read-only transactions by default, set the `default_transaction_use_follower_reads` [session variable](set-vars.html) to `on`. When `default_transaction_use_follower_reads=on` and follower reads are enabled, all read-only transactions use follower reads.
{{site.data.alerts.end}}

### Follower reads and long-running writes

Long-running write transactions will create write intents with a timestamp near when the transaction began. When a follower read encounters a write intent, it will often end up in a "transaction wait queue", waiting for the operation to complete; however, this runs counter to the benefit follower reads provides.

To counteract this, you can issue all follower reads in explicit transactions set with `HIGH` priority:

```sql
BEGIN PRIORITY HIGH AS OF SYSTEM TIME follower_read_timestamp();
SELECT ...
SELECT ...
COMMIT;
```

## Exact Staleness Reads

## Bounded Staleness Reads

### INTERFACE

`with_min_timestamp(TIMESTAMPTZ)` | <span class="version-tag">New in v21.2:</span> XXX
`with_max_staleness(INTERVAL)` | <span class="version-tag">New in v21.2:</span> XXX

## See Also

- [Cluster Settings Overview](cluster-settings.html)
- [Load-Based Splitting](load-based-splitting.html)
- [Network Latency Page](ui-network-latency-page.html)
- [{{ site.data.products.enterprise }} Features](enterprise-licensing.html)
- [Follower Reads Pattern](topology-follower-reads.html)
