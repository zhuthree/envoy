# 服务发现

当上游集群在[配置](https://www.envoyproxy.io/docs/envoy/latest/api-v1/route_config/route_config#config-http-conn-man-route-table/api-v1/cluster_manager/cluster#config-cluster-manager-cluster)中定义时，Envoy 需要知道如何解析集群中的成员。这被称为*服务发现*。


## 支持的服务发现类型

### 静态类型

静态类型是最简单的服务发现类型。配置中会明确指定每个上游主机的已解析网络名称（ IP 地址、端口、unix 域套接字等）。

### 严格的 DNS 类型

当使用严格的 DNS 服务发现时，Envoy 将持续并异步地解析指定的 DNS 目标。DNS 结果中的每个返回的 IP 地址将被视为上游集群中的显式主机。这意味着如果查询返回三个 IP 地址，Envoy 将假定该集群有三台主机，并且所有三台主机应该负载均衡。如果有主机从结果中删除，则 Envoy 会认为它不再存在，并且会将它从所有的当前连接池中排除。请注意，Envoy 从不同步解析转发路径中的 DNS 。基于最终一致性的策略，永远不用担心长时间运行的 DNS 查询会受到阻塞。


### 逻辑 DNS 类型

逻辑 DNS 使用类似异步解析机制来遵循严格的 DNS 策略。但是，这种策略并非严格基于 DNS 的查询结果，逻辑 DNS 集群假设它们构成了整个上游集群并*在需要初始化新连接时*仅使用返回的第一个 IP 地址。因此，单个逻辑连接池可能包含各种不同上游主机的物理连接。并且连接永远不会流失。这种服务发现类型适用于必须通过 DNS 访问的大型 Web 服务。此类服务通常使用循环 DNS 来返回许多不同的 IP 地址。通常每次查询返回不同的结果。如果在这种情况下使用了严格的 DNS 服务发现类型，Envoy 会假定集群的成员在每个解析间隔内都会发生变化，这将导致连接池枯竭，连接循环等。相反，使用逻辑 DNS，连接会一直保持活动状态直到它们循环。在与大型 Web 服务交互时，这几乎是所有可行策略中最好的方式：异步、最终一致的 DNS 解析、长连接、转发路径零阻塞。

### 原始目的地类型

当传入连接通过 iptables 重定向或 TPROXY（查看真实 IP ）目标或使用代理协议重定向到 Envoy 时，可以使用原始目标集群。在这些情况下，使用元数据重定向的策略会使路由到原始目标集群的请求转发到上游主机，并且不需要任何明确的主机配置或上游主机发现机制。与上游主机的连接会被放入链接，并且闲置的主机在空闲时间比 [cleanup_interval_ms](https://www.envoyproxy.io/docs/envoy/latest/api-v1/cluster_manager/cluster#config-cluster-manager-cluster-cleanup-interval-ms) 长（默认值为5000 ms）时会被刷新。如果原始目标地址不可用，则不会打开上游连接。这种原始目的地服务发现的方式必须与原始目的地[负载均衡](load_balancing.md#arch-overview-load-balancing-types-original-destination)一起使用。


### 服务发现服务（SDS）
Envoy 通过一个*发现服务的服务*获取集群成员，它是一个 [REST 风格的 API](https://www.envoyproxy.io/docs/envoy/latest/api-v1/cluster_manager/sds#config-cluster-manager-sds-api)。Lyft 提供了基于 Python 语言的 [发现服务](https://github.com/lyft/discovery)参考实现。该实现使用 AWS DynamoDB 作为存储类型，但该 API 非常简单，可以轻松地在各种不同的存储类型之上实施。对于每个 SDS 集群，Envoy 将定期从发现服务中获取集群成员。由于以下几个原因，SDS 是首选的服务发现机制：

- Envoy 对每个上游主机都有明确的了解（与通过 DNS 解析的负载均衡进行路由相比而言），并可以做出更智能的负载均衡决策。
- 在每个主机的发现 API 响应中携带的额外属性通知 Envoy 负载均衡权重、金丝雀状态、区域等。这些附加属性在负载均衡、统计信息收集等过程中会被 Envoy 网格全局使用。

通常，主动健康检查与最终一致的服务发现服务数据结合使用，以进行负载均衡和路由决策。这将在下一节中进一步讨论。

## 关于服务发现的最终一致性

许多现有的 RPC 系统将服务发现视为完全一致的过程。为此，它们使用完全一致的领导选举算法，如 Zookeeper、etcd、Consul 等。我们的经验是，大规模使用这些策略是很痛苦的。

Envoy 从一开始就设计服务发现不需要完全一致性。相反的，Envoy 假定主机以最终一致的方式来回穿过网格。我们推荐的将服务部署到服务 Envoy 网格配置的方式是使用最终一致的服务发现以及主动运行[健康检查](health_checking.md#arch-overview-health-checking)（Envoy 显式地对上游集群成员进行健康检查）来确定集群运行状况。这种方式有许多好处：

- 所有健康决定都是完全分布的。因此，网络分区可以得到适当的处理（应用程序是否正常处理分区是另一回事）。
- 为上游集群配置运行状况检查时，Envoy 使用 2x2 矩阵来确定是否路由到主机：

| 发现状态           | 健康检查成功| 健康检查失败         |
| ------------------|-----------|---------------------|
| 发现              | 路由       | 不要路由             |
| 未发现          | 路由       | 不用路由、删除        |

**发现主机，健康检查成功**

Envoy **将路由**到目标主机

**未发现主机，健康检查成功**

Envoy **将路由**到目标主机。这一点非常重要，因为我们在设计时假定发现服务随时可能失败。即使发现主机数据缺失，但健康检查通过，Envoy 仍会路由。虽然在这种情况下添加新主机是不可能的，但现有主机仍将继续正常运行。当发现服务再次正常运行时，数据将最终重新收集。

**发现主机，健康检查失败**

Envoy **不会路由**到目标主机。我们假设健康检查数据比发现数据更准确。

**未发现主机，健康检查失败**

Envoy **不会路由并将删除目标主机**。这是 Envoy 清除主机数据的唯一一种情况。