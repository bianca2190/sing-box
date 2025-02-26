### Structure

```json
{
  "type": "loadbalance",
  "tag": "loadbalance",
  "outbounds": [
    "proxy-a",
    "proxy-b",
    "proxy-c"
  ],
  "providers": [
    "provider-a",
    "provider-b",
  ],
  "check": {
    "interval": "5m",
    "sampling": 10,
    "destination": "http://www.gstatic.com/generate_204",
    "connectivity": "http://connectivitycheck.platform.hicloud.com/generate_204"
  },
  "pick": {
    "objective": "leastload",
    "strategy": "random",
    "max_fail": 0,
    "max_rtt": "1000ms",
    "expected": 3,
    "baselines": [
      "30ms",
      "50ms",
      "100ms",
      "150ms",
      "200ms",
      "250ms",
      "350ms"
    ]
  }
}
```

### Fields

#### outbounds

List of outbound tags.

#### providers

List of provider tags.

#### check

See "Check Fields"

#### pick

See "Pick Fields"

### Check Fields

#### interval

The interval of health check for each node. Must be greater than `10s`, default is `5m`.

#### sampling

The number of recent health check results to sample. Must be greater than `0`, default is `10`.

#### destination

The destination URL for health check. Default is `http://www.gstatic.com/generate_204`.

#### connectivity

The destination of connectivity check. Default is empty. 

If a health check fails, it may be caused by network unavailability (e.g. disconnected from WIFI). Set this field to avoid the node being judged to be invalid under such circumstances, or this behavior will not occur.

### Pick Fields

#### objective

The objective of load balancing. Default is `alive`.

| Objective   | Description                                    |
| ----------- | ---------------------------------------------- |
| `alive`     | prefer alive nodes                             |
| `qualified` | prefer qualified nodes (`max_rtt`, `max_fail`) |
| `leastload` | least load nodes from qualified                |
| `leastping` | least latency nodes from qualified             |

Load balancing divides nodes into three classes:

1. Failed Nodes, that cannot be connected
2. Alive Nodes, that pass the health check
3. Qualified Nodes, that are alive and meet the constraints (`max_rtt`, `max_fail`)

It tries to pick from the class that the objective is targeting (see the table above). If there is no node for current class, it falls back to the next class. For example, the behavior of `leastload` could be:

- Pick least load nodes from qualified ones
- Pick least load nodes from alives ones
- Pick least load nodes from failed ones (could be temporarily failed)

Generally speaking, use `leastload`, `leastping` for better network quality; use `alive` for more quantity of outbound nodes.

#### strategy

The strategy of load balancing. Default is `random`.

| Strategy         | Description                                        |
| ---------------- | -------------------------------------------------- |
| `random`         | Pick randomly from nodes match the objective       |
| `roundrobin`     | Rotate from nodes match the objective              |
| `consistenthash` | Use same node for requests to same origin targets. |

Note: `consistenthash` requires a relatively stable quantity of nodes, it's available only when the objective is `alive`

#### max_rtt

The maximum round-trip time of health check that is acceptable for qulified nodes. Default is `0`, which accepts any round-trip time.

#### max_fail

The maximum number of health check failures for qulified nodes, default is `0`, i.e. no failures allowed.

#### expected / baselines

> Available only for `least*` objectives

`expected` is the expected number of nodes to be selected. The default value is 1.

`baselines` divide the nodes into different ranges. The default value is empty. For `leastload`, it divides according to the standard deviation (STD) of RTTs; For `leastping`, it divides according to the average of RTTs.

Here are typical configuration for `leastload`:

1. `expected: 3`, select 3 nodes with the smallest STD.

1. `baselines: ["30ms","50ms","100ms"]`, try to select nodes by different baselines. If there is no node matching any baseline, return the one with the smallest STD.

1. `expected:3, baselines =["30ms","50ms","100ms"]`, try different baselines until we find at least 3 nodes. Otherwise select the top 3 nodes with the smallest STD. The advantage is that it can find a proper quantity of nodes without wasting nodes with similar qualities.

1. If both not configured, select one node with the smallest STD.