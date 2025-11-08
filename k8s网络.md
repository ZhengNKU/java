## Pod IP 从哪里来
### 使用自定义网络插件部署：
 环境上使用cilium网络插件部署集群，因此pod ip为 Cilium 管理的 `cilium_host` 接口的 IP 网段，用于 Cilium 内部的路由和通信。
 `cilium_host` 是 Cilium 网络插件的一部分，负责 Kubernetes 集群中的网络连接、负载均衡和网络安全。

```shell
cilium_host: flags=4291<UP,BROADCAST,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 172.20.1.102  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fc00::10ca:1  prefixlen 128  scopeid 0x0<global>
        inet6 fe80::879:46ff:fefd:a380  prefixlen 64  scopeid 0x20<link>
        ether 0a:79:46:fd:a3:80  txqueuelen 1000  (Ethernet)
        RX packets 40677277  bytes 34785454856 (32.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 373  bytes 31719 (30.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 默认部署：
**Pod 的 IP 是 Docker 创建和分配的容器 IP**，**这个 IP 是带虚拟网卡的**，因此**这个 IP 是可以被 ping 的**，与此同时，**这个 IP 只能在当前节点中被访问**。

首先创建 Pod 时，Pod 会启动一个 pause 容器，这个容器创建了一个虚拟网卡，并被 Docker 分配 IP，接着 Pod 的容器会使用 container 网络模式连接到这个 pause 容器中，pause 容器的生命周期跟 Pod 的生命周期一致。可以在工作节点上使用 `docker ps -a | grep pause` 命令查看 pause 容器。
在部署了 Docker 的机器上，都会有一个 docker0 的东西，这个东西叫网桥。docker 的默认网桥叫 docker0，这个网桥的 IP 是 172.17.0.1，基于这个网桥创建的容器的虚拟网卡自然是 172.17.0.0 地址段。


