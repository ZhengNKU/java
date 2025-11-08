![[Pasted image 20250311113225.png]]
https://istio.io/latest/zh/docs/concepts/traffic-management/

## Istio是什么
Istio 是一个与 Kubernetes 紧密结合的服务网格(Service Mesh)，用于**服务治理**。
注意，Istio 是用于服务治理的，主要有流量管理、服务间安全、可观测性这几种功能。在微服务系统中，会碰到很多棘手的问题，Istio 只能解决其中一小部分。Istio 对业务是非侵入式的，**完全不需要改动项目代码**。Istio 服务网格在逻辑上分为数据平面和控制平面。

1. 流量管理包括以下功能：
- 动态服务发现
- 负载均衡
- TLS 终端
- HTTP/2 与 gRPC 代理
- 熔断器
- 健康检查
- 基于百分比流量分割的分阶段发布
- 故障注入
- 丰富的指标
2. 可观测性
Istio 支持 Jaeger、Zipkin、Skywalking 等链路追踪中间件，支持 Prometheus 收集指标数据
3. 安全性能
主要特点是可以实现零信任网络中的服务之间通讯加密。Istio 通过自动为服务之间的通信提供双向 TLS 加密来增强安全性，同时 Istio 还提供了强大的身份验证、授权和审计功能。

### 数据平面
数据平面是一组代理，用于调解和控制微服务之间的所有网络通信。 这些代理还可以收集和报告所有网格流量的可观测数据。
Istio 支持两种主要的数据平面模式：

- **Sidecar 模式**，此模式会为集群中启动的每个 Pod 都部署一个 Envoy 代理， 或者与在虚拟机上运行的服务并行运行一个 Envoy 代理。
- **Ambient 模式**，此模式在每个节点上使用四层代理， 另外可以选择为每个命名空间使用一个 Envoy 代理来实现七层功能。

#### Sidecar
每个 Pod 都有一个 Envoy 负责拦截、处理和转发进出 Pod 的所有网络流量，这种方式被称为 Sidecar。
#### Ambient 模式
Ambient 模式于 2022 年推出，旨在解决 Sidecar 模式用户报告的缺点。 从 Istio 1.22 开始，它在单集群使用场景就达到生产就绪状态。
- 所有流量都通过仅支持四层的节点代理进行代理
- 应用可以选择通过 Envoy 代理进行路由，以获得七层功能



### 控制平面
控制平面管理和配置数据平面中的这些代理。



### 流量管理
Istio 基本的服务发现和负载均衡能力为您提供了一个可用的服务网格， 但它能做到的远比这多的多。在许多情况下，您可能希望对网格的流量情况进行更细粒度的控制。 作为 A/B 测试的一部分，您可能想将特定百分比的流量定向到新版本的服务， 或者为特定的服务实例子集应用不同的负载均衡策略。您可能还想对进出网格的流量应用特殊的规则， 或者将网格的外部依赖项添加到服务注册中心。通过使用 Istio 的流量管理 API 将流量配置添加到 Istio， 就可以完成所有这些甚至更多的工作。

#### VirtualService
虚拟服务在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用， 实现方式是解耦客户端请求的目标地址与实际响应请求的目标工作负载。 虚拟服务同时提供了丰富的方式，为发送至这些工作负载的流量指定不同的路由规则。
如果没有虚拟服务， Envoy 会在所有的服务实例中使用轮询的负载均衡策略分发请求。

yaml：
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews  # 使用 `hosts` 字段列举虚拟服务的主机——即用户指定的目标或是路由规则设定的目标。 这是客户端向服务发送请求时使用的一个或多个地址。
  http:   
  - match:
    - headers:
        end-user:
          exact: jason  # 路由应用于来自 ”jason“ 用户的所有请求
      uri: 
	    prefix: "/ratings/v2/" 
	  ignoreUriCase: true
    route:
    - destination:  # route 部分的 `destination` 字段指定了符合此条件的流量的实际目标地址。 与虚拟服务的 `hosts` 不同，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。 可以是一个有代理的服务网格，或者是一个通过服务入口（service entry）被添加进来的非网格服务。
        host: reviews
        subset: v2
  - route:
    - destination: 
        host: reviews
        subset: v3
```

**路由规则**按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。本示例中， 不满足第一个路由规则的流量均流向一个默认的目标，该目标在第二条规则中指定。因此， 第二条规则没有 match 条件，直接将流量导向 v3 子集。

#### DestinationRule
您可以将虚拟服务视为将流量如何路由到给定目标地址， 然后使用目标规则来配置该目标的流量。在评估虚拟服务路由规则之后， 目标规则将应用于流量的“真实”目标地址。

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

![[Pasted image 20250311173333.png]]

#### Gateway
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
```
这个网关配置让 HTTPS 流量从 `ext-host.example.com` 通过 443 端口流入网格， 但没有为请求指定任何路由规则。要指定路由并让网关按预期工作，您必须把网关绑定到虚拟服务上。 正如下面的示例所示，使用虚拟服务的 `gateways` 字段进行设置：
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
    - ext-host-gwy
```

#### ServiceEntry
使用[服务入口（Service Entry）](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/#ServiceEntry) 来添加一个入口到 Istio 内部维护的服务注册中心。添加了服务入口后，Envoy 代理可以向服务发送流量， 就好像它是网格内部的服务一样。配置服务入口允许您管理运行在网格外的服务的流量，它包括以下几种能力：

- 为外部目标重定向和转发请求，例如来自 Web 端的 API 调用，或者流向遗留老系统的服务。
- 为外部目标定义[重试](https://istio.io/latest/zh/docs/concepts/traffic-management/#retries)、[超时](https://istio.io/latest/zh/docs/concepts/traffic-management/#timeouts)和[故障注入](https://istio.io/latest/zh/docs/concepts/traffic-management/#fault-injection)策略。
- 添加一个运行在虚拟机的服务来[扩展您的网格](https://istio.io/latest/zh/docs/examples/virtual-machines/single-network/#running-services-on-the-added-VM)。

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```
您可以配置虚拟服务和目标规则，以更细粒度的方式控制到服务入口的流量， 这与网格中的任何其他服务配置流量的方式相同。例如， 下面的目标规则调整了使用服务入口配置的 `ext-svc.example.com` 外部服务的连接超时：

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 1s
```

#### Sidecar
默认情况下，Istio 让每个 Envoy 代理都可以访问来自和它关联的工作负载的所有端口的请求， 然后转发到对应的工作负载。您可以使用 [Sidecar](https://istio.io/latest/zh/docs/reference/config/networking/sidecar/#Sidecar) 配置去做下面的事情：
- 微调 Envoy 代理接受的端口和协议集。
- 限制 Envoy 代理可以访问的服务集合。
![[Pasted image 20250311174447.png]]






