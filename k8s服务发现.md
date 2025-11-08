
https://k8s.whuanle.cn/4.network/4.discovery.html
https://tinychen.com/20220627-k8s-09-service-discovery-and-traffic-exposure/#1-2-CNI%E5%9F%BA%E7%A1%80

### proxy
允许所有 IP 访问，又不需要认证，则可以使用：
```
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'
```

### Headless Services
Headless 服务是一种特殊的服务类型，它不会分配虚拟 IP，而是直接暴露所有 Pod 的 IP 和 DNS 记录。这使得我们可以直接访问 Pod IP 地址，并使用这些 IP 地址进行负载均衡，并没有使用k8s内置的kube-proxy进行负载均衡。
`Headless Services`**这种方式优点在于足够简单、请求的链路短，但是缺点也很明显，就是DNS的缓存问题带来的不可控。很多程序查询DNS并不会参考规范的TTL值，要么频繁的查询给DNS服务器带来巨大的压力，要么查询之后一直缓存导致服务变更了还在请求旧的IP**。


https://k8s.whuanle.cn/4.network/4.discovery.html
https://tinychen.com/20220627-k8s-09-service-discovery-and-traffic-exposure/#1-2-CNI%E5%9F%BA%E7%A1%80

### proxy
允许所有 IP 访问，又不需要认证，则可以使用：
```
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'
```

### Headless Services
Headless 服务是一种特殊的服务类型，它不会分配虚拟 IP，而是直接暴露所有 Pod 的 IP 和 DNS 记录。这使得我们可以直接访问 Pod IP 地址，并使用这些 IP 地址进行负载均衡，并没有使用k8s内置的kube-proxy进行负载均衡。
`Headless Services`**这种方式优点在于足够简单、请求的链路短，但是缺点也很明显，就是DNS的缓存问题带来的不可控。很多程序查询DNS并不会参考规范的TTL值，要么频繁的查询给DNS服务器带来巨大的压力，要么查询之后一直缓存导致服务变更了还在请求旧的IP**。

![[Pasted image 20250301111848.png]]

### ClusterIP Services
在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。 `kube-proxy` 负责为 Service 实现了一种 VIP（虚拟 IP）的形式，一般称之为`ClusterIP Services`。

![[Pasted image 20250301111959.png]]

`ClusterIP Services`这种方式的**优点是有VIP位于Pod前面，可以有效避免前面提及的直接DNS解析带来的各类问题；缺点也很明显，当请求量大的时候，kube-proxy组件的处理性能会首先成为整个请求链路的瓶颈。**

### NodePort Service
- 从NodePort开始，服务就不仅局限于在K8S集群内暴露，开始可以对集群外提供服务
- NodePort类型会从K8S的宿主机节点上面挑选一个端口分配给某个服务（默认范围是30000-32767），用户可以通过请求任意一个K8S节点IP的该指定端口来访问这个服务
- NodePort服务域名解析的解析结果是一个`CLUSTER-IP`，在集群内部请求的负载均衡逻辑和实现与`ClusterIP Service`是一致的
- NodePort服务的请求路径是**从K8S节点IP直接到Pod**，并不会经过ClusterIP，但是这个转发逻辑依旧是由`kube-proxy`实现
![[Pasted image 20250301112217.png]]

`NodePort Service`这种方式的**优点是非常简单的就能把服务通过K8S自带的功能暴露到集群外部；缺点也很明显：NodePort本身的端口限制（数量和选择范围都有限）以及请求量大时的kube-proxy组件的性能瓶颈问题。**

### LoadBalancer Service
- LoadBalancer服务类型需要K8S集群支持一个云原生的LoadBalancer，这部分功能K8S本身没有实现，而是将其交给云厂商/第三方，因此对于云环境的K8S集群可以直接使用云厂商提供的LoadBalancer，当然也有一些开源的云原生LoadBalancer，如MetalLB、OpenELB、PureLB等；
- LoadBalancer服务域名解析的解析结果是一个`CLUSTER-IP`；
- LoadBalancer服务同时会分配一个`EXTERNAL-IP`，集群外的机器可以通过这个`EXTERNAL-IP`来访问服务；
- LoadBalancer服务默认情况下会同时创建NodePort，也就是说一个LoadBalancer类型的服务同时是一个NodePort服务，同时也是一个clusterIP服务；一些云原生LoadBalancer可以通过指定`allocateLoadBalancerNodePorts: false`来拒绝创建NodePort服务；
![[Pasted image 20250301142411.png]]

`LoadBalancer Service`这种方式的**优点是方便、高效、适用场景广泛，几乎可以覆盖所有对外的服务暴露；缺点则是成熟可用的云原生LoadBalancer选择不多，实现门槛较高。**

##  Port概念辨析
- `nodePort`: 只存在于Loadbalancer服务和NodePort服务中，用于指定K8S集群的宿主机节点的端口，默认范围是`30000-32767`，K8S集群外部可以通过`NodeIP:nodePort` 来访问某个service；
- `port`: 只作用于`CLUSTER-IP`和`EXTERNAL-IP`，也就是对Loadbalancer服务、NodePort服务和ClusterIP服务均有作用，K8S集群内部可以通过`CLUSTER-IP:port`来访问，K8S集群外部可以通过`EXTERNAL-IP:port`来访问；
- `targetPort`: Pod的外部访问端口，port和nodePort的流量会参照对应的ipvs规则转发到Pod上面的这个端口，也就是说数据的转发路径是`NodeIP:nodePort -> PodIP:targetPort`、`CLUSTER-IP:port -> PodIP:targetPort`、`EXTERNAL-IP:port -> PodIP:targetPort`
- `containerPort`：和其余三个概念不属于同一个维度，`containerPort`主要是在`工作负载（Workload）`中配置，其余三者均是在`service`中配置。`containerPort`主要作用在Pod内部的container，用来告知K8S这个container内部提供服务的端口，因此理论上`containerPort`应该要和container内部实际监听的端口一致，这样才能确保服务正常；但是实际上由于各个CNI的实现不通以及K8S配置的网络策略差异，`containerPort`的作用并不明显，很多时候配置错误或者是不配置也能正常工作；


# 四层服务暴露
![[Pasted image 20250301151004.png]]

# 七层服务暴露
四层LoadBalancer服务暴露的方式优点是适用范围广，因为工作在四层，因此几乎能适配所有类型的应用。但是也有一些缺点：

- 对于大多数应用场景都是http协议的请求来说，并不需要给每个服务都配置一个`EXTERNAL-IP`来暴露服务，这样一来资源严重浪费（公网IP十分珍贵），二来包括IP地址已经HTTPS使用的证书管理等均十分麻烦
- 比较常见的场景是对应的每个配置都配置一个域名（virtual host）或者是路由规则（routing rule），然后统一对外暴露一个或者少数几个`EXTERNAL-IP`，将所有的请求流量都导入到一个统一个集中入口网关（如Nginx），再由这个网关来进行各种负载均衡（load balancing）、路由管理（routing rule）、证书管理（SSL termination）等等

![[Pasted image 20250301151956.png]]
针对上面四层服务暴露的架构图，我们在入口处的LoadBalancer后面加入一个Ingress，就可以得到七层Ingress服务暴露的架构图。

- 在这个图中的处理逻辑和四层服务暴露一致，唯一不同的就是HTTP协议的流量是先经过入口处的loadbalancer，再转发到ingress里面，ingress再根据里面的ingress rule来进行判断转发；
- k8s的ingress-nginx是会和集群内的api-server通信并且获取服务信息，因为`Ingress Controllers`本身就具有负载均衡的能力，因此在把流量转发到后端的具体服务时，不会经过ClusterIP（就算服务类型是ClusterIP也不经过），而是直接转发到所属的`Pod IP`上；

### Ingress
Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。Ingress 可以提供负载均衡（load balancing）、路由管理（routing rule）、证书管理（SSL termination）等功能。Ingress 公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。
通过配置，Ingress 可为 Service 提供外部可访问的 URL、对其流量作负载均衡、 终止 SSL/TLS，以及基于名称的虚拟托管等能力。 [Ingress 控制器](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress-controllers) 负责完成 Ingress 的工作，具体实现上通常会使用某个负载均衡器， 不过也可以配置边缘路由器或其他前端来帮助处理流量。

![[Pasted image 20250306112457.png]]

==ingress通过指定不同的路由规则实现负载均衡==
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080

```

![[Pasted image 20250306145439.png]]










### ClusterIP Services
在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。 `kube-proxy` 负责为 Service 实现了一种 VIP（虚拟 IP）的形式，一般称之为`ClusterIP Services`。

![[Pasted image 20250301111959.png]]

`ClusterIP Services`这种方式的**优点是有VIP位于Pod前面，可以有效避免前面提及的直接DNS解析带来的各类问题；缺点也很明显，当请求量大的时候，kube-proxy组件的处理性能会首先成为整个请求链路的瓶颈。**

### NodePort Service
- 从NodePort开始，服务就不仅局限于在K8S集群内暴露，开始可以对集群外提供服务
- NodePort类型会从K8S的宿主机节点上面挑选一个端口分配给某个服务（默认范围是30000-32767），用户可以通过请求任意一个K8S节点IP的该指定端口来访问这个服务
- NodePort服务域名解析的解析结果是一个`CLUSTER-IP`，在集群内部请求的负载均衡逻辑和实现与`ClusterIP Service`是一致的
- NodePort服务的请求路径是**从K8S节点IP直接到Pod**，并不会经过ClusterIP，但是这个转发逻辑依旧是由`kube-proxy`实现
![[Pasted image 20250301112217.png]]

`NodePort Service`这种方式的**优点是非常简单的就能把服务通过K8S自带的功能暴露到集群外部；缺点也很明显：NodePort本身的端口限制（数量和选择范围都有限）以及请求量大时的kube-proxy组件的性能瓶颈问题。**

### LoadBalancer Service
- LoadBalancer服务类型需要K8S集群支持一个云原生的LoadBalancer，这部分功能K8S本身没有实现，而是将其交给云厂商/第三方，因此对于云环境的K8S集群可以直接使用云厂商提供的LoadBalancer，当然也有一些开源的云原生LoadBalancer，如MetalLB、OpenELB、PureLB等；
- LoadBalancer服务域名解析的解析结果是一个`CLUSTER-IP`；
- LoadBalancer服务同时会分配一个`EXTERNAL-IP`，集群外的机器可以通过这个`EXTERNAL-IP`来访问服务；
- LoadBalancer服务默认情况下会同时创建NodePort，也就是说一个LoadBalancer类型的服务同时是一个NodePort服务，同时也是一个clusterIP服务；一些云原生LoadBalancer可以通过指定`allocateLoadBalancerNodePorts: false`来拒绝创建NodePort服务；
![[Pasted image 20250301142411.png]]

`LoadBalancer Service`这种方式的**优点是方便、高效、适用场景广泛，几乎可以覆盖所有对外的服务暴露；缺点则是成熟可用的云原生LoadBalancer选择不多，实现门槛较高。**

##  Port概念辨析
- `nodePort`: 只存在于Loadbalancer服务和NodePort服务中，用于指定K8S集群的宿主机节点的端口，默认范围是`30000-32767`，K8S集群外部可以通过`NodeIP:nodePort` 来访问某个service；
- `port`: 只作用于`CLUSTER-IP`和`EXTERNAL-IP`，也就是对Loadbalancer服务、NodePort服务和ClusterIP服务均有作用，K8S集群内部可以通过`CLUSTER-IP:port`来访问，K8S集群外部可以通过`EXTERNAL-IP:port`来访问；
- `targetPort`: Pod的外部访问端口，port和nodePort的流量会参照对应的ipvs规则转发到Pod上面的这个端口，也就是说数据的转发路径是`NodeIP:nodePort -> PodIP:targetPort`、`CLUSTER-IP:port -> PodIP:targetPort`、`EXTERNAL-IP:port -> PodIP:targetPort`
- `containerPort`：和其余三个概念不属于同一个维度，`containerPort`主要是在`工作负载（Workload）`中配置，其余三者均是在`service`中配置。`containerPort`主要作用在Pod内部的container，用来告知K8S这个container内部提供服务的端口，因此理论上`containerPort`应该要和container内部实际监听的端口一致，这样才能确保服务正常；但是实际上由于各个CNI的实现不通以及K8S配置的网络策略差异，`containerPort`的作用并不明显，很多时候配置错误或者是不配置也能正常工作；


# 四层服务暴露
![[Pasted image 20250301151004.png]]

# 七层服务暴露
四层LoadBalancer服务暴露的方式优点是适用范围广，因为工作在四层，因此几乎能适配所有类型的应用。但是也有一些缺点：

- 对于大多数应用场景都是http协议的请求来说，并不需要给每个服务都配置一个`EXTERNAL-IP`来暴露服务，这样一来资源严重浪费（公网IP十分珍贵），二来包括IP地址已经HTTPS使用的证书管理等均十分麻烦
- 比较常见的场景是对应的每个配置都配置一个域名（virtual host）或者是路由规则（routing rule），然后统一对外暴露一个或者少数几个`EXTERNAL-IP`，将所有的请求流量都导入到一个统一个集中入口网关（如Nginx），再由这个网关来进行各种负载均衡（load balancing）、路由管理（routing rule）、证书管理（SSL termination）等等

![[Pasted image 20250301151956.png]]
针对上面四层服务暴露的架构图，我们在入口处的LoadBalancer后面加入一个Ingress，就可以得到七层Ingress服务暴露的架构图。

- 在这个图中的处理逻辑和四层服务暴露一致，唯一不同的就是HTTP协议的流量是先经过入口处的loadbalancer，再转发到ingress里面，ingress再根据里面的ingress rule来进行判断转发；
- k8s的ingress-nginx是会和集群内的api-server通信并且获取服务信息，因为`Ingress Controllers`本身就具有负载均衡的能力，因此在把流量转发到后端的具体服务时，不会经过ClusterIP（就算服务类型是ClusterIP也不经过），而是直接转发到所属的`Pod IP`上；

### Ingress
Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。Ingress 可以提供负载均衡（load balancing）、路由管理（routing rule）、证书管理（SSL termination）等功能。Ingress 公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。
通过配置，Ingress 可为 Service 提供外部可访问的 URL、对其流量作负载均衡、 终止 SSL/TLS，以及基于名称的虚拟托管等能力。 [Ingress 控制器](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress-controllers) 负责完成 Ingress 的工作，具体实现上通常会使用某个负载均衡器， 不过也可以配置边缘路由器或其他前端来帮助处理流量。

![[Pasted image 20250306112457.png]]

==ingress通过指定不同的路由规则实现负载均衡==
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080

```

![[Pasted image 20250306145439.png]]








