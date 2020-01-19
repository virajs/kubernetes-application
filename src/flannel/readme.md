#### Kubernetes的网络

部署好的kubernetes可以通过查看master和node是否正常,来了解kubernetes集群是否正常工作.

```bash
> kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

Kubernetes的网络和网络策略:

* [ACI](https://www.github.com/noironetworks/aci-containers) 通过Cisco ACI 提供集成的容器网络和安全网络。
* [Calico](https://docs.projectcalico.org/v3.11/getting-started/kubernetes/installation/calico)是一个安全的 L3 网络和网络策略提供者。
* [Canal](https://github.com/projectcalico/canal/tree/master/k8s-install) 结合 Flannel 和 Calico， 提供网络和网络策略。
* [Cilium](https://github.com/cilium/cilium)是一个 L3 网络和网络策略插件， 能够透明的实施 HTTP/API/L7 策略。 同时支持路由（routing）和叠加/封装（ overlay/encapsulation）模式。
* [CNI-Genie](https://github.com/Huawei-PaaS/CNI-Genie) 使Kubernetes 无缝连接到一种 CNI 插件，例如：Flannel、Calico、Canal、Romana 或者 Weave。
* [Contiv](https://contiv.io/)为多种用例提供可配置网络（使用 BGP 的原生 L3，使用 vxlan 的 overlay，经典 L2 和 Cisco-SDN/ACI）和丰富的策略框架。Contiv 项目完全开源。安装工具同时提供基于和不基于 kubeadm 的安装选项。
* [Contrail](https://www.juniper.net/us/en/products-services/sdn/contrail/contrail-networking/)它基于Tungsten Fabric，是一个开源的多云网络虚拟化和策略管理平台。
* [Flannel](https://github.com/coreos/flannel) 是一个可以用于 Kubernetes 的 overlay 网络提供者。
* [Knitter](https://github.com/ZTE/Knitter/)是为 kubernetes 提供复合网络解决方案的网络组件.
* [Multus](https://github.com/intel/multus-cni)是一个多插件，可在 Kubernetes 中提供多种网络支持，以支持所有 CNI 插件（例如 Calico，Cilium，Contiv，Flannel）is a Multi plugin for multiple network support in Kubernetes to support all CNI plugins (e.g. Calico, Cilium, Contiv, Flannel)，而且包含了在Kubernetes中基于 SRIOV，DPDK，OVS-DPDK 和 VPP 的工作负载.
* [NSX-T](http://docs.vmware.com/en/VMware-NSX-T-Data-Center/2.0/nsxt_20_ncp_kubernetes.pdf)容器插件（ NCP ）提供了 VMware NSX-T 与容器协调器（例如 Kubernetes）之间的集成，以及 NSX-T 与基于容器的 CaaS 或者PaaS 平台（例如关键容器服务（ PKS ）和 OpenShift ）之间的集成。
* [Nuage](https://github.com/nuagenetworks/nuage-kubernetes/blob/v5.1.1-1/docs/kubernetes-1-installation.rst)是一个SDN平台，可在Kubernetes Pods和非Kubernetes环境之间提供基于策略的联网，并具有可视化和安全监控。
* [Romana](http://romana.io/)是一个 pod 网络的层 3 解决方案，并且支持 NetworkPolicy API。Kubeadm add-on 安装细节可以在这里找到。
* [Weave network](https://kubernetes.io/zh/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/)提供了在网络分组两端参与工作的网络和网络策略，并且不需要额外的数据库。

服务发现:

[CoreDNS](https://coredns.io/)是一种灵活的，可扩展的 DNS 服务器，可以 [安装](https://github.com/coredns/deployment/tree/master/kubernetes)为集群内的 Pod 提供 DNS 服务。


在讨论Kubernetes网络之前，让我们先来看一下Docker网络。

Docker采用插件化的网络模式，默认提供`bridge`、`host`、`none`、`overlay`、`maclan`和`Network plugins`这六种网络模式，运行容器时可以通过–network参数设置具体使用那一种模式。

* bridge：这是Docker默认的网络驱动，此模式会为每一个容器分配Network Namespace和设置IP等，并将容器连接到一个虚拟网桥上。如果未指定网络驱动，这默认使用此驱动。

* host：开放式网络模式,此网络驱动直接使用宿主机的网络。

* none：对于此容器，禁用所有联网。因此none驱动不构造网络环境。采用了none 网络驱动，那么就只能使用loopback网络设备，容器只能使用127.0.0.1的本机网络。

* overlay：overlay 网络将多个Docker daemons连接在一起，并能够使用swarm服务之间进行通讯。也可以使用overlay网络进行swarm服务和容器之间、容器之间进行通讯，

* macvlan：此网络允许为容器指定一个MAC地址，允许容器作为网络中的物理设备，这样Docker daemon就可以通过MAC地址进行访问的路由。对于希望直接连接网络网络的遗留应用，这种网络驱动有时可能是最好的选择。

* Network plugins：可以安装和使用第三方的网络插件。可以在Docker Store或第三方供应商处获取这些插件。

然而docker的网络模式又分为单机模式和多机模式:

* 单主机网络模式 bridge、host、none、container

* 多主机网络模式 overlay、macvlan、flannel


下面我们分别来分析下docker的网络模式.

#### bridge网络模式

Docker Container的bridge桥接模式可以说是目前Docker开发者最常使用的网络模式。

Brdige桥接模式为Docker Container创建独立的网络栈，保证容器内的进程组使用独立的网络环境，实现容器间、容器与宿主机之间的网络栈隔离。

另外，Docker通过宿主机上的网桥(docker0)来连通容器内部的网络栈与宿主机的网络栈，实现容器与宿主机乃至外界的网络通信。

<p align="center">
<img width="600" align="center" src="src/images/6.jpg" />
</p>


Bridge桥接模式的实现步骤主要如下：

* Docker Daemon利用veth pair技术，在宿主机上创建两个虚拟网络接口设备，假设为veth0和veth1。而veth pair技术的特性可以保证无论哪一个veth接收到网络报文，都会将报文传输给另一方。
* Docker Daemon将veth0附加到Docker Daemon创建的docker0网桥上。保证宿主机的网络报文可以发往veth0；
* Docker Daemon将veth1添加到Docker Container所属的namespace下，并被改名为eth0。如此一来，保证宿主机的网络报文若发往veth0，则立即会被eth0接收，实现宿主机到Docker Container网络的联通性；同时，也保证Docker Container单独使用eth0，实现容器网络环境的隔离性。


1. bridge网络的构建过程

Bridge桥接模式，从原理上实现了Docker Container到宿主机乃至其他机器的网络连通性。然而，由于宿主机的IP地址与veth pair的 IP地址均不在同一个网段，如果只是依靠veth pair和namespace的技术，还不足以是宿主机以外的网络主动发现Docker Container的存在。
为了使得Docker Container可以让宿主机以外的世界感知到容器内部暴露的服务，Docker采用NAT（Network Address Translation，网络地址转换）的方式，让宿主机以外的世界可以主动将网络报文发送至容器内部。


因此当Docker Container需要暴露服务时，内部服务必须监听容器IP和端口号port_0，以便外界主动发起访问请求。

由于宿主机以外的世界，只知道宿主机eth0的网络地址，而并不知道Docker Container的IP地址，哪怕就算知道Docker Container的IP地址，从二层网络的角度来讲，外界也无法直接通过Docker Container的IP地址访问容器内部应用。

因此，Docker使用NAT方法，将容器内部的服务监听的端口与宿主机的某一个端口port_1进行“绑定”。


如此一来，外界访问Docker Container内部服务的流程为：

* 外界访问宿主机的IP以及宿主机的端口port_1；

* 当宿主机接收到这样的请求之后，由于DNAT规则的存在，会将该请求的目的IP（宿主机eth0的IP）和目的端口port_1进行转换，转换为容器IP和容器的端口port_0;

* 由于宿主机认识容器IP，故可以将请求发送给veth pair；

* veth pair的veth0将请求发送至容器内部的eth0，最终交给内部服务进行处理。


通过 ifconfig 命令可以查看docker0网桥的信息：
```bash
> ifconfig
  lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
          options=1203<RXCSUM,TXCSUM,TXSTATUS,SW_TIMESTAMP>
          inet 127.0.0.1 netmask 0xff000000
          inet6 ::1 prefixlen 128
          inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
          nd6 options=201<PERFORMNUD,DAD>
  gif0: flags=8010<POINTOPOINT,MULTICAST> mtu 1280
  stf0: flags=0<> mtu 1280
  XHC20: flags=0<> mtu 0
  XHC0: flags=0<> mtu 0
  en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
          ether 8c:85:90:3c:bb:89
          inet6 fe80::1c2d:7b3e:78ed:8d75%en0 prefixlen 64 secured scopeid 0x6
          inet 10.105.17.104 netmask 0xfffffc00 broadcast 10.101.15.255
          nd6 options=201<PERFORMNUD,DAD>
          media: autoselect
          status: active
  bridge0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
          options=63<RXCSUM,TXCSUM,TSO4,TSO6>
          ether 86:00:88:31:8e:01
          Configuration:
                  id 0:0:0:0:0:0 priority 0 hellotime 0 fwddelay 0
                  maxage 0 holdcnt 0 proto stp maxaddr 100 timeout 1200
                  root id 0:0:0:0:0:0 priority 0 ifcost 0 port 0
                  ipfilter disabled flags 0x2
          member: en1 flags=3<LEARNING,DISCOVER>
                  ifmaxaddr 0 port 9 priority 0 path cost 0
          member: en2 flags=3<LEARNING,DISCOVER>
                  ifmaxaddr 0 port 10 priority 0 path cost 0
          nd6 options=201<PERFORMNUD,DAD>
          media: <unknown type>
          status: inactive
          
  ....                
```

通过 docker network inspect bridge 可以查看网桥的子网网络范围和网关：
```bash
> docker network inspect bridge
  [
      {
          "Name": "bridge",
          "Id": "ccc8612439ed4a274e29c0f417274643454a9fb79c926611eae769d1b8959531",
          "Created": "2019-10-24T01:40:31.972461234Z",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": null,
              "Config": [
                  {
                      "Subnet": "172.17.0.0/16",
                      "Gateway": "172.17.0.1"
                  }
              ]
          },
          "Internal": false,
          "Attachable": false,
          "Ingress": false,
          "ConfigFrom": {
              "Network": ""
          },
          "ConfigOnly": false,
          "Containers": {
              "08bd3fbcc3d4818efd388e5ce4fe3700309eac361ed89fe4887e4ea86d1b0715": {
                  "Name": "keke-mysql",
                  "EndpointID": "c0292a3d8db661309e611968186ab42de253a612cd4903c9b5be980e561def2e",
                  "MacAddress": "02:42:ac:11:00:02",
                  "IPv4Address": "172.17.0.2/16",
                  "IPv6Address": ""
              }
          },
          "Options": {
              "com.docker.network.bridge.default_bridge": "true",
              "com.docker.network.bridge.enable_icc": "true",
              "com.docker.network.bridge.enable_ip_masquerade": "true",
              "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
              "com.docker.network.bridge.name": "docker0",
              "com.docker.network.driver.mtu": "1500"
          },
          "Labels": {}
      }
  ]
```

运行容器时，在宿主机上创建虚拟网卡veth pair设备，veth pair设备是成对出现的，从而组成一个数据通道，数据从一个设备进入，就会从另一个设备出来。

将veth pair设备的一端放在新创建的容器中，命名为eth0；另一端放在宿主机的docker0中，以veth为前缀的名字命名。通过 brctl show 命令查看放在docker0中的veth pair设备


使用DNAT方法，可以使得Docker宿主机以外的世界主动访问Docker Container内部服务。

2. 外部访问

那么Docker Container如何访问宿主机以外的世界呢。以下简要分析Docker Container访问宿主机以外世界的流程.


bridge的docker0是虚拟出来的网桥，因此无法被外部的网络访问。因此需要在运行容器时通过-p和-P参数对将容器的端口映射到宿主机的端口。

实际上Docker是采用 NAT的 方式，将容器内部的服务监听端口与宿主机的某一个端口port 进行绑定，使得宿主机外部可以将网络报文发送至容器。

* 通过-P参数，将容器的端口映射到宿主机的随机端口
```bash
> docker run -P {images}
```  
* 通过-p参数，将容器的端口映射到宿主机的制定端口
```bash
> docker run -p {hostPort}:{containerPort} {images}
```


Docker Container内部进程获悉宿主机以外服务的IP地址和端口port_2，于是Docker Container发起请求。容器的独立网络环境保证了请求中报文的源IP地址为容器IP（即容器内部eth0），另外Linux内核会自动为进程分配一个可用源端口（假设为port_3）;

* 请求通过容器内部eth0发送至veth pair的另一端，到达veth0，也就是到达了网桥（docker0）处；

* docker0网桥开启了数据报转发功能（/proc/sys/net/ipv4/ip_forward），故将请求发送至宿主机的eth0处；

* 宿主机处理请求时，使用SNAT对请求进行源地址IP转换，即将请求中源地址IP（容器IP地址）转换为宿主机eth0的IP地址；

* 宿主机将经过SNAT转换后的报文通过请求的目的IP地址（宿主机以外世界的IP地址）发送至外界。


对于Docker Container内部主动发起对外的网络请求，当请求到达宿主机进行SNAT处理后发给外界，当外界响应请求时，响应报文中的目的IP地址肯定是Docker宿主机的IP地址，那响应报文回到宿主机的时候，宿主机又是如何转给Docker Container的呢？

关于这样的响应，由于port_3端口并没有在宿主机上做相应的DNAT转换，原则上不会被发送至容器内部。为什么说对于这样的响应，不会做DNAT转换呢。原因很简单，DNAT转换是针对容器内部服务监听的特定端口做的，该端口是供服务监听使用，而容器内部发起的请求报文中，源端口号肯定不会占用服务监听的端口，故容器内部发起请求的响应不会在宿主机上经过DNAT处理。

其实，这一环节的内容是由iptables规则来完成，具体的iptables规则如下：

```bash
> iptables -I FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

这条规则的意思是，在宿主机上发往docker0网桥的网络数据报文，如果是该数据报文所处的连接已经建立的话，则无条件接受，并由Linux内核将其发送到原来的连接上，即回到Docker Container内部。

可以说，bridger桥接模式从功能的角度实现了两个方面：第一，让容器拥有独立、隔离的网络栈；第二，让容器和宿主机以外的世界通过NAT建立通信。

然而，bridge桥接模式下的Docker Container在使用时，并非为开发者包办了一切。最明显的是，该模式下Docker Container不具有一个公有IP，即和宿主机的eth0不处于同一个网段。

导致的结果是宿主机以外的世界不能直接和容器进行通信。虽然NAT模式经过中间处理实现了这一点，但是NAT模式仍然存在问题与不便，如：容器均需要在宿主机上竞争端口，容器内部服务的访问者需要使用服务发现获知服务的外部端口等。

另外NAT模式由于是在三层网络上的实现手段，故肯定会影响网络的传输效率。


#### host网络模式

Docker Container中的host模式与bridge桥接模式有很大的不同。最大的区别当属，host模式并没有为容器创建一个隔离的网络环境。

而之所以称之为host模式，是因为该模式下的Docker Container会和host宿主机共享同一个网络namespace，故Docker Container可以和宿主机一样，使用宿主机的eth0，实现和外界的通信。换言之，Docker Container的IP地址即为宿主机eth0的IP地址。


Docker Container的other container网络模式:

<p align="center">
<img width="600" align="center" src="src/images/7.jpg" />
</p>

图中最左侧的Docker Container，即采用了host网络模式，而其他两个Docker Container依然沿用brdige桥接模式，两种模式同时存在于宿主机上并不矛盾。

Docker Container的host网络模式在实现过程中，由于不需要额外的网桥以及虚拟网卡，故不会涉及docker0以及veth pair。

其中, 父进程在创建子进程时，如果不使用CLONE_NEWNET这个参数标志，那么创建出的子进程会与父进程共享同一个网络namespace。Docker就是采用了这个简单的原理，在创建进程启动容器的过程中，没有传入CLONE_NEWNET参数标志，实现Docker Container与宿主机共享同一个网络环境，即实现host网络模式。

可以说，Docker Container的网络模式中，host模式是bridge桥接模式很好的补充。采用host模式的Docker Container，可以直接使用宿主机的IP地址与外界进行通信，若宿主机的eth0是一个公有IP，那么容器也拥有这个公有IP。

同时容器内服务的端口也可以使用宿主机的端口，无需额外进行NAT转换。当然，有这样的方便，肯定会损失部分其他的特性，最明显的是Docker Container网络环境隔离性的弱化，即容器不再拥有隔离、独立的网络栈。另外，使用host模式的Docker Container虽然可以让容器内部的服务和传统情况无差别、无改造的使用，但是由于网络隔离性的弱化，该容器会与宿主机共享竞争网络栈的使用；

另外，容器内部将不再拥有所有的端口资源，原因是部分端口资源已经被宿主机本身的服务占用，还有部分端口已经用以bridge网络模式容器的端口映射。


#### none网络模式

网络环境为none，即不为Docker Container任何的网络环境。一旦Docker Container采用了none网络模式，那么容器内部就只能使用loopback网络设备，不会再有其他的网络资源。

可以说none模式为Docker Container做了极少的网络设定，但是俗话说得好“少即是多”，在没有网络配置的情况下，作为Docker开发者，才能在这基础做其他无限多可能的网络定制开发。这也恰巧体现了Docker设计理念的开放。

#### container网络模式

该模式类似于host模式，只不过host模式是主机与容器共享Network Namespace,而container模式是容器之间共享Network Namespace。

新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

其配置方式为`--net=container:<name_or_id>`

#### overlay网络模式

<p align="center">
<img width="600" align="center" src="src/images/8.jpg" />
</p>

overlay网络可以实现跨主机的容器VLAN，主要用于 docker swarm 上，docker 文档中也基本是两者同时出现，因此docker swarm是跨主机的容器与容器之间的官方标准通信方案。

容器在两个跨主机进行通信的时候，是使用overlay network这个网络模式进行通信，如果使用host也可以实现跨主机进行通信，直接使用这个物理的ip地址就可以进行通信。

overlay它会虚拟出一个网络比如10.0.9.3这个ip地址，在这个overlay网络模式里面，有一个类似于服务网关的地址，然后把这个包转发到物理服务器这个地址，最终通过路由和交换，到达另一个服务器的ip地址。

那么,在docker容器里面overlay 是怎么实现的呢？

我们会有一个服务发现，比如说是consul，会定义一个ip地址池，比如10.0.9.0/24之类的，上面会有容器，容器的ip地址会从上面去获取，获取完了后，会通过eth1进行通信，贼这实现跨主机的东西。

<p align="center">
<img width="600" align="center" src="src/images/9.jpg" />
</p>

需要创建一个consul的服务容器:

```bash
> docker run -d -p 8400:8400 -p 8500:8500 -p 8600:53/udp -h consul progrium/consul -server -bootstrap -ui-dir /ui
```
修改它的启动参数:
```bash
> ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --cluster-store=consul://192.168.59.100:8500 --cluster-advertise=enp0s8:2376 --insecure-registry=0.0.0.0/0
```
注意这里,hostA和hostB都需要修改.

#### macvlan网络模式

macvlan模式可以创建出一个新的自定义模式，其新启动的容器可以按新的模式配置网络环境。
    
查看网络模式:
```bash
>  docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
3fb4debc2c4f        apollo_default        bridge              local
ccc8612439ed        bridge                bridge              local
2da252aafa83        deployments_default   bridge              local
19d49c5334fc        etcd-manage_default   bridge              local
e2e60f36631e        host                  host                local
211e521edd2b        none                  null                local
```

创建新的macvlan:
```bash
> docker network create -d macvlan  --subnet=192.168.100.0/24 --gateway=192.168.100.1 -o parent=ens33 macvlan-net
```

```bash
> docker run --net=gitlab-net --ip=192.168.111.201 -dit --name mysql keke-mysql
```

因此就可以创建新的容器，通过--net=macvlan-net指定其为新的模式,--ip参数指定容器ip。

相对于前面几种模式，该模式更加灵活，可根据实际需求配置出新的模式，满足较为复杂的网络需求。


#### kubernetes的flannel网络模式

上面分析了那么的docker的网络模式,那么在k8s下面的flannel网络模式是什么样呢?

Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址。

在默认的Docker配置中，每个节点上的Docker服务会分别负责所在节点容器的IP分配。这样导致的一个问题是，不同节点上容器可能获得相同的内外IP地址。并使这些容器之间能够之间通过IP地址相互找到，也就是相互ping通。

Flannel的设计目的就是为集群中的所有节点重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信。

Flannel为每个主机提供独立的子网，整个集群的网络信息存储在etcd上。对于跨主机的转发，目标容器的IP地址，需要从etcd获取。

Flannel利用Kubernetes API或者etcd用于存储整个集群的网络配置，其中最主要的内容为设置集群的网络地址空间。例如，设定整个集群内所有容器的IP都取自网段“10.1.0.0/16”。

Flannel实质上是一种“覆盖网络(overlaynetwork)”，也就是将TCP数据包装在另一种网络包里面进行路由转发和通信，目前已经支持udp、vxlan、host-gw、aws-vpc、gce和alloc路由等数据转发方式，默认的节点间数据通信方式是UDP转发。

Flannel在每个主机中运行Flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。

Flanneld再将本主机获取的subnet以及用于主机间通信的Public IP，同样通过kubernetes API或者etcd存储起来。

Flannel利用各种backend ，例如udp，vxlan，host-gw等等，跨主机转发容器间的网络流量，完成容器间的跨主机通信。


Flannel的特点:

1. 使集群中的不同Node主机创建的Docker容器都具有全集群唯一的虚拟IP地址。

2. 建立一个覆盖网络（overlay network），通过这个覆盖网络，将数据包原封不动的传递到目标容器。覆盖网络是建立在另一个网络之上并由其基础设施支持的虚拟网络。覆盖网络通过将一个分组封装在另一个分组内来将网络服务与底层基础设施分离。在将封装的数据包转发到端点后，将其解封装。

3. 创建一个新的虚拟网卡flannel0接收docker网桥的数据，通过维护路由表，对接收到的数据进行封包和转发（vxlan）。

4. etcd保证了所有node上flanned所看到的配置是一致的。同时每个node上的flanned监听etcd上的数据变化，实时感知集群中node的变化。


Flannel对网络要求提出的解决办法:

互相不冲突的ip:

1.flannel利用Kubernetes API或者etcd用于存储整个集群的网络配置，根据配置记录集群使用的网段。

2.flannel在每个主机中运行flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。

如测试环境中ip分配：

1. master1节点
```bash
> cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

2. master2节点
```bash
> cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.1.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

3. master3节点
```bash
> cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.2.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

4. node1节点
```bash
> cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.4.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

5. node2节点
```bash
> cat /run/flannel/subnet.env
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.5.1/24
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
```

6. node3节点
```bash
> cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.6.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

在flannel network中，每个pod都会被分配唯一的ip地址，且每个K8s node的subnet各不重叠，没有交集。

Pod之间互相访问:

* flanneld将本主机获取的subnet以及用于主机间通信的Public IP通过etcd存储起来，需要时发送给相应模块。
* flannel利用各种backend mechanism，例如udp，vxlan等等，跨主机转发容器间的网络流量，完成容器间的跨主机通信。

