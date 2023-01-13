### 结构

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

### 字段

#### outbounds

出站标签列表。

#### providers

订阅标签列表。

#### check

参见“健康检查字段”

#### pick

参见“节点挑选字段”

### 健康检查字段

#### interval

每个节点的健康检查间隔。不小于`10s`，默认为 `5m`。

#### sampling

对最近的多少次检查结果进行采样。大于 `0`，默认为 `10`。

#### destination

用于健康检查的链接。默认使用 `http://www.gstatic.com/generate_204`。

#### connectivity

网络连通性检查地址，默认为空。

健康检查失败，可能是由于网络不可用造成的（比如断开 WIFI 连接）。设置此项，可避免此类情况下将节点判定为失效，否则不会有此行为。

### 节点挑选字段

#### objective

负载均衡的目标。默认为 `alive`。

| 目标        | 描述                                              |
| ----------- | ------------------------------------------------- |
| `alive`     | 优先使用存活节点                                  |
| `qualified` | 优先使用合格节点 (符合 `max_rtt`, `max_fail`)     |
| `leastload` | 优先使用低负载的合格节点 (历次检查中表现更稳定的) |
| `leastping` | 优先使用低延时的合格节点                          |

#### strategy

负载均衡的策略。默认为 `random`。

| 策略             | 描述                             |
| ---------------- | -------------------------------- |
| `random`         | 从符合目标的节点中，随机挑选     |
| `roundrobin`     | 从符合目标的节点中，轮流选择     |
| `consistenthash` | 使用同一节点处理同源站点的请求。 |

注意：`consistenthash` 仅当目标为 `alive` 时可用。

#### max_rtt

合格节点可接受的健康检查最大往返时间。 默认为 `0`，即接受任何往返时间。

#### max_fail

合格节点健康检查最大失败次。默认为 `0`，即不允许任何失败。

#### expected

`least*` 目标期望选出的节点数量。默认为 `1`。

#### baselines

`least*` 目标选择节点的基准线，它将节点划分为不同的档位。默认为空。

- 对于 `leastload`，根据往返时间标准差划分；
- 对于 `leastping`，根据往返时间平均值划分。

### 概念

`loadbalance` 将节点分为三类:

1. 无效节点: 无法连接的节点 (可能是临时失效)
2. 存活节点: 通过健康检查的节点
3. 合格节点: 存活且满足限制条件 (`max_rtt`, `max_fail`)

正常情况下，负载均衡将尝试从当前目标所面向的分类中挑选（参见[objective](#objective)）。没有合适节点时，负载均衡将退而求其次，从次一级分类中选择。举例来说，`leastload` 实际执行的策略也可能为：

- 从存活节点中选择最小负载
- 从无效节点中选择最小负载

无论节点情况如何恶劣，选一些总比不选好。

`loadbalance` 通过 `expected` 和 `baselines` 配置控制 `least*` 目标的筛选行为。

以 `leastload` 为例，几种典型配置为：

1. `expected: 3`，选出往返时间标准差最小的 3 个节点。

1. `baselines: ["30ms","50ms","100ms"]`，依次尝试不同基准线，若没有符合任何基准线的节点，返回标准差最小的一个。

1. `expected:3, baselines =["30ms","50ms","100ms"]`，依次尝试不同基准线，直到找到至少3个节点。否则返回标准差最小的3个。这种配置的好处是，既找到合适数量的节点，又不浪费素质相近的更多节点。

1. 若两者均未配置，选择出往返时间最短的一个节点。