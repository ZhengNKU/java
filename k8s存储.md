**常用分类：**  
emptyDir（临时目录）:Pod删除，数据也会被清除，这种存储成为emptyDir，用于数据的临时存储。  
hostPath (宿主机目录映射):  
本地的SAN (iSCSI,FC)、NAS(nfs,cifs,http)存储  
分布式存储（glusterfs，rbd，cephfs）  
云存储（EBS，Azure Disk）

> 一个emptyDir 第一次创建是在一个pod被指定到具体node的时候，并且会一直存在在pod的生命周期当中，正如它的名字一样，它初始化是一个空的目录，pod中的容器都可以读写这个目录，这个目录可以被挂在到各个容器相同或者不相同的的路径下。当一个pod因为任何原因被移除的时候，这些数据会被永久删除。注意：一个容器崩溃了不会导致数据的丢失，因为容器的崩溃并不移除pod.

## <font color="#ff0000">NFS</font>
### **安装**
关闭防火墙

`➜ systemctl stop firewalld.service ➜ systemctl disable firewalld.service`

安装配置 nfs

`➜ yum -y install nfs-utils rpcbind`

共享目录设置权限：

`➜ mkdir -p /var/lib/k8s/data ➜ chmod 755 /var/lib/k8s/data/`

配置 nfs，nfs 的默认配置文件在 `/etc/exports` 文件下，在该文件中添加下面的配置信息：

`➜ vi /etc/exports /var/lib/k8s/data  *(rw,sync,no_root_squash)`

配置说明：

- `/var/lib/k8s/data`：是共享的数据目录
- *：表示任何人都有权限连接，当然也可以是一个网段，一个 IP，也可以是域名
- rw：读写的权限
- sync：表示文件同时写入硬盘和内存
- no_root_squash：当登录 NFS 主机使用共享目录的使用者是 root 时，其权限将被转换成为匿名使用者，通常它的 UID 与 GID，都会变成 nobody 身份

| 参数命令（*：使用重要程度） | 参数用途                                                                                                                                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| rw(*)          | 表示可读写                                                                                                                                                                                                                                         |
| ro             | Read-only表示只能读权限                                                                                                                                                                                                                              |
| sync(*)        | 将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性（同步）                                                                                                                                                                                                          |
| async          | 将数据先保存在内存缓冲区中，必要时才写入磁盘（异步）                                                                                                                                                                                                                    |
| no_root_squash | 保留管理员权限，以服务器管理员的权限管理,用户应避免使用！                                                                                                                                                                                                                 |
| root_squash    | 将root用户的访问映射为匿名（nfsnobody）用户uid和gid                                                                                                                                                                                                           |
| all_squash(*)  | 不管访问nfs server共享目录的用户身份如何包括root，它的权限都将被压缩成为匿名用户，同时他们的udi和gid都会变成nobody或nfsnobody账户的uid，gid。在多个nfs客户端同时读写nfs server数据时，这个参数很有用可以确保大家写入的数据的权限是一样的。但不同系统有可能匿名用户的uid，gid不同。因为此处我们需要服务端和客户端之间的用户是一样的。比如说：服务端指定匿名用户的UID为2000，那么客户端也一定要存在2000这个账号才可以 |
| anonuid        | anonuid就是匿名的uid和gid。说明客户端以什么权限来访问服务端，在默认情况下是nfsnobody。Uid65534                                                                                                                                                                                |
| anongid        | 同anongid，就是把uid换成gid而已。                                                                                                                                                                                                                       |

**启动方式：**
启动服务 nfs 需要向 rpc 注册，rpc 一旦重启了，注册的文件都会丢失，向他注册的服务都需要重启 注意启动顺序，先启动 rpcbind
![[Pasted image 20250106143159.png]]

到这里我们就把 nfs server 给安装成功了，然后就是前往节点安装 nfs 的客户端来验证，安装 nfs 当前也需要先关闭防火墙：
然后安装 nfs
安装完成后，和上面的方法一样，先启动 rpc、然后启动 nfs
挂载数据目录 客户端启动完成后，我们在客户端来挂载下 nfs 测试下，首先检查下 nfs 是否有共享目录：

`➜ showmount -e 192.168.31.31 Export list for 192.168.31.31: /var/lib/k8s/data *`

然后我们在客户端上新建目录：

`➜ mkdir -p /root/course/kubeadm/data`

将 nfs 共享目录挂载到上面的目录：

`➜ mount -t nfs 192.168.31.31:/var/lib/k8s/data /root/course/kubeadm/data`



## GFS
接下来，**我们来看GFS和HDFS都有哪些具体特性，我们应该如何应用？**

1. GFS是一种适合大文件，尤其是GB级别的大文件存储场景的分布式存储系统。

2. GFS非常适合对数据访问延迟不敏感的搜索引擎服务。

3. GFS是一种有中心节点的分布式架构，Master节点是单一的集中管理节点，既是高可用的瓶颈，也是可能出现性能问题的瓶颈。

4. GFS可以通过缓存一部分Metadata到Client节点，减少Client与Master的交互。

5. GFS的Master节点上的Operation log和Checkpoint文件需要通过复制方式保留多个副本，来保障元数据以及中心管理功能的高可用性。

### Ceph
1. Ceph是一种统一了三种接口的统一存储平台，上层应用支持Object、Block、File 。

2. Ceph采用Crush算法完成数据分布计算，通过Tree的逻辑对象数据结构自然实现故障隔离副本位置计算，通过将Bucket内节点的组织结构，集群结构变化导致的数据迁移量最小。

3. Ceph保持数据强一致性算法，数据的所有副本都写入并返回才算写事务的完成，写的效率会差一些，所以更适合写少读多的场景。

4. 对象保存的最小单元为4M，相比GFS&HDFS而言，适合一些小的非结构化数据存储。


