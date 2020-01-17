#### Kubernetes的网络

部署好的kubernetes可以通过查看master和node是否正常,来了解kubernetes集群是否正常工作.

```bash
> kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

Kubernetes的网络:

* [Flannel](https://github.com/coreos/flannel)
* [Calico](https://www.projectcalico.org/)
* [Weave network](https://kubernetes.io/zh/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/)

在讨论Kubernetes网络之前，让我们先来看一下Docker网络。

Docker采用插件化的网络模式，默认提供`bridge`、`host`、`none`、`overlay`、`maclan`和`Network plugins`这六种网络模式，运行容器时可以通过–network参数设置具体使用那一种模式。

* bridge：这是Docker默认的网络驱动，此模式会为每一个容器分配Network Namespace和设置IP等，并将容器连接到一个虚拟网桥上。如果未指定网络驱动，这默认使用此驱动。

* host：开放式网络模式,此网络驱动直接使用宿主机的网络。

* none：对于此容器，禁用所有联网。因此none驱动不构造网络环境。采用了none 网络驱动，那么就只能使用loopback网络设备，容器只能使用127.0.0.1的本机网络。

* overlay：overlay 网络将多个Docker daemons连接在一起，并能够使用swarm服务之间进行通讯。也可以使用overlay网络进行swarm服务和容器之间、容器之间进行通讯，

* macvlan：此网络允许为容器指定一个MAC地址，允许容器作为网络中的物理设备，这样Docker daemon就可以通过MAC地址进行访问的路由。对于希望直接连接网络网络的遗留应用，这种网络驱动有时可能是最好的选择。

* Network plugins：可以安装和使用第三方的网络插件。可以在Docker Store或第三方供应商处获取这些插件。

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

Docker源码分析（七）：Docker Container网络 （上）
Posted on 2015年1月25日
1.前言(什么是Docker Container)
如今，Docker技术大行其道，大家在尝试以及玩转Docker的同时，肯定离不开一个概念，那就是“容器”或者“Docker Container”。那么我们首先从实现的角度来看看“容器”或者“Docker Container”到底为何物。

逐渐熟悉Docker之后，大家肯定会深深得感受到：应用程序在Docker Container内部的部署与运行非常便捷，只要有Dockerfile，应用一键式的部署运行绝对不是天方夜谭； Docker Container内运行的应用程序可以受到资源的控制与隔离，大大满足云计算时代应用的要求。毋庸置疑，Docker的这些特性，传统模式下应用是完全不具备的。然而，这些令人眼前一亮的特性背后，到底是谁在“作祟”，到底是谁可以支撑Docker的这些特性？不知道这个时候，大家是否会联想到强大的Linux内核。

其实，这很大一部分功能都需要归功于Linux内核。那我们就从Linux内核的角度来看看Docker到底为何物，先从Docker Container入手。关于Docker Container，体验过的开发者第一感觉肯定有两点：内部可以跑应用（进程），以及提供隔离的环境。当然，后者肯定也是工业界称之为“容器”的原因之一。

既然Docker Container内部可以运行进程，那么我们先来看Docker Container与进程的关系，或者容器与进程的关系。首先，我提出这样一个问题供大家思考“容器是否可以脱离进程而存在”。换句话说，能否创建一个容器，而这个容器内部没有任何进程。

可以说答案是否定的。既然答案是否定的，那说明不可能先有容器，然后再有进程，那么问题又来了，“容器和进程是一起诞生，还是先有进程再有容器呢？”可以说答案是后者。以下将慢慢阐述其中的原因。

阐述问题“容器是否可以脱离进程而存在”的原因前，相信大家对于以下的一段话不会持有异议：通过Docker创建出的一个Docker Container是一个容器，而这个容器提供了进程组隔离的运行环境。那么问题在于，容器到底是通过何种途径来实现进程组运行环境的“隔离”。这时，就轮到Linux内核技术隆重登场了。

说到运行环境的“隔离”，相信大家肯定对Linux的内核特性namespace和cgroup不会陌生。namespace主要负责命名空间的隔离，而cgroup主要负责资源使用的限制。其实，正是这两个神奇的内核特性联合使用，才保证了Docker Container的“隔离”。那么，namespace和cgroup又和进程有什么关系呢？问题的答案可以用以下的次序来说明：

父进程通过fork创建子进程时，使用namespace技术，实现子进程与其他进程（包含父进程）的命名空间隔离；
子进程创建完毕之后，使用cgroup技术来处理子进程，实现进程的资源使用限制；
系统在子进程所处namespace内部，创建需要的隔离环境，如隔离的网络栈等；
namespace和cgroup两种技术都用上之后，进程所处的“隔离”环境才真正建立，这时“容器”才真正诞生！
从Linux内核的角度分析容器的诞生，精简的流程即如以上4步，而这4个步骤也恰好巧妙的阐述了namespace和cgroup这两种技术和进程的关系，以及进程与容器的关系。进程与容器的关系，自然是：容器不能脱离进程而存在，先有进程，后有容器。然而，大家往往会说到“使用Docker创建Docker Container（容器），然后在容器内部运行进程”。对此，从通俗易懂的角度来讲，这完全可以理解，因为“容器”一词的存在，本身就较为抽象。如果需要更为准确的表述，那么可以是：“使用Docker创建一个进程，为这个进程创建隔离的环境，这样的环境可以称为Docker Container（容器），然后再在容器内部运行用户应用进程。”当然，笔者的本意不是想否定很多人对于Docker Container或者容器的认识，而是希望和读者一起探讨Docker Container底层技术实现的原理。

对于Docker Container或者容器有了更加具体的认识之后，相信大家的眼球肯定会很快定位到namespace和cgroup这两种技术。Linux内核的这两种技术，竟然能起到如此重大的作用，不禁为止赞叹。那么下面我们就从Docker Container实现流程的角度简要介绍这两者。

首先讲述一下namespace在容器创建时的用法，首先从用户创建并启动容器开始。当用户创建并启动容器时，Docker Daemon 会fork出容器中的第一个进程A（暂且称为进程A，也就是Docker Daemon的子进程）。Docker Daemon执行fork时，在clone系统调用阶段会传入5个参数标志CLONE_NEWNS、CLONE_NEWUTS、CLONE_NEWIPC、CLONE_NEWPID和CLONE_NEWNET（目前Docker 1.2.0还没有完全支持user namespace）。Clone系统调用一旦传入了这些参数标志，子进程将不再与父进程共享相同的命名空间（namespace），而是由Linux为其创建新的命名空间（namespace），从而保证子进程与父进程使用隔离的环境。另外，如果子进程A再次fork出子进程B和C，而fork时没有传入相应的namespace参数标志，那么此时子进程B和C将会与A共享同一个命令空间（namespace）。如果Docker Daemon再次创建一个Docker Container，容器内第一个进程为D，而D又fork出子进程E和F，那么这三个进程也会处于另外一个新的namespace。两个容器的namespace均与Docker Daemon所在的namespace不同。Docker关于namespace的简易示意图如下：

17
图1.1 Docker中namespace示意图
再说起cgroup，大家都知道可以使用cgroup为进程组做资源的控制。与namespace不同的是，cgroup的使用并不是在创建容器内进程时完成的，而是在创建容器内进程之后再使用cgroup，使得容器进程处于资源控制的状态。换言之，cgroup的运用必须要等到容器内第一个进程被真正创建出来之后才能实现。当容器内进程被创建完毕，Docker Daemon可以获知容器内进程的PID信息，随后将该PID放置在cgroup文件系统的指定位置，做相应的资源限制。
可以说Linux内核的namespace和cgroup技术，实现了资源的隔离与限制。那么对于这种隔离与受限的环境，是否还需要配置其他必需的资源呢。这回答案是肯定的，网络栈资源就是在此时为容器添加。当为容器进程创建完隔离的运行环境时，发现容器虽然已经处于一个隔离的网络环境（即新的network namespace），但是进程并没有独立的网络栈可以使用，如独立的网络接口设备等。此时，Docker Daemon会将Docker Container所需要的资源一一为其配备齐全。网络方面，则需要在用户指定的网络模式在，配置Docker Container相应的网络资源。

2.Docker Container网络分析内容安排
Docker Container网络篇将从源码的角度，分析Docker Container从无到有的过程中，Docker Container网络创建的来龙去脉。Docker Container网络创建流程可以简化如下图：

17
图2.1 Docker Container网络创建流程图
Docker Container网络篇分析的主要内容有以下5部分：
Docker Container的网络模式；
Docker Client配置容器网络；
Docker Daemon创建容器网络流程；
execdriver网络执行流程；
libcontainer实现内核态网络配置。
Docker Container网络创建过程中，networkdriver模块使用并非是重点，故分析内容中不涉及networkdriver。这里不少读者肯定会有疑惑。需要强调的是，networkdriver在Docker中的作用：第一，为Docker Daemon创建网络环境的时候，初始化Docker Daemon的网络环境（详情可以查看《Docker源码分析》系列第六篇），比如创建docker0网桥等；第二，为Docker Container分配IP地址，为Docker Container做端口映射等。而与Docker Container网络创建有关的内容极少，只有在桥接模式下，为Docker Container的网络接口设备分配一个可用IP地址。

本文为《Docker源码分析》系列第七篇——Docker Container网络（上）。

3.Docker Container网络模式
正如在上文提到的，Docker可以为Docker Container创建隔离的网络环境，在隔离的网络环境下，Docker Container独立使用私有网络。相信很多的Docker开发者也是体验过Docker这方面的网络特性。

其实，Docker除了可以为Docker Container创建隔离的网络环境之外，同样有能力为Docker Container创建共享的网络环境。换言之，当开发者需要Docker Container与宿主机或者其他容器网络隔离时，Docker可以满足这样的需求；而当开发者需要Docker Container与宿主机或者其他容器共享网络时，Docker同样可以满足这样的需求。另外，Docker还可以不为Docker Container创建网络环境。

总结Docker Container的网络，可以得出4种不同的模式：bridge桥接模式、host模式、other container模式和none模式。以下初步介绍4中不同的网络模式。

3.1 bridge桥接模式
Docker Container的bridge桥接模式可以说是目前Docker开发者最常使用的网络模式。Brdige桥接模式为Docker Container创建独立的网络栈，保证容器内的进程组使用独立的网络环境，实现容器间、容器与宿主机之间的网络栈隔离。另外，Docker通过宿主机上的网桥(docker0)来连通容器内部的网络栈与宿主机的网络栈，实现容器与宿主机乃至外界的网络通信。

Docker Container的bridge桥接模式可以参考下图：

17
图3.1 Docker Container Bridge桥接模式示意图
Bridge桥接模式的实现步骤主要如下：
Docker Daemon利用veth pair技术，在宿主机上创建两个虚拟网络接口设备，假设为veth0和veth1。而veth pair技术的特性可以保证无论哪一个veth接收到网络报文，都会将报文传输给另一方。
Docker Daemon将veth0附加到Docker Daemon创建的docker0网桥上。保证宿主机的网络报文可以发往veth0；
Docker Daemon将veth1添加到Docker Container所属的namespace下，并被改名为eth0。如此一来，保证宿主机的网络报文若发往veth0，则立即会被eth0接收，实现宿主机到Docker Container网络的联通性；同时，也保证Docker Container单独使用eth0，实现容器网络环境的隔离性。
Bridge桥接模式，从原理上实现了Docker Container到宿主机乃至其他机器的网络连通性。然而，由于宿主机的IP地址与veth pair的 IP地址均不在同一个网段，故仅仅依靠veth pair和namespace的技术，还不足以是宿主机以外的网络主动发现Docker Container的存在。为了使得Docker Container可以让宿主机以外的世界感知到容器内部暴露的服务，Docker采用NAT（Network Address Translation，网络地址转换）的方式，让宿主机以外的世界可以主动将网络报文发送至容器内部。

具体来讲，当Docker Container需要暴露服务时，内部服务必须监听容器IP和端口号port_0，以便外界主动发起访问请求。由于宿主机以外的世界，只知道宿主机eth0的网络地址，而并不知道Docker Container的IP地址，哪怕就算知道Docker Container的IP地址，从二层网络的角度来讲，外界也无法直接通过Docker Container的IP地址访问容器内部应用。因此，Docker使用NAT方法，将容器内部的服务监听的端口与宿主机的某一个端口port_1进行“绑定”。

如此一来，外界访问Docker Container内部服务的流程为：

外界访问宿主机的IP以及宿主机的端口port_1；
当宿主机接收到这样的请求之后，由于DNAT规则的存在，会将该请求的目的IP（宿主机eth0的IP）和目的端口port_1进行转换，转换为容器IP和容器的端口port_0;
由于宿主机认识容器IP，故可以将请求发送给veth pair；
veth pair的veth0将请求发送至容器内部的eth0，最终交给内部服务进行处理。
使用DNAT方法，可以使得Docker宿主机以外的世界主动访问Docker Container内部服务。那么Docker Container如何访问宿主机以外的世界呢。以下简要分析Docker Container访问宿主机以外世界的流程：

Docker Container内部进程获悉宿主机以外服务的IP地址和端口port_2，于是Docker Container发起请求。容器的独立网络环境保证了请求中报文的源IP地址为容器IP（即容器内部eth0），另外Linux内核会自动为进程分配一个可用源端口（假设为port_3）;
请求通过容器内部eth0发送至veth pair的另一端，到达veth0，也就是到达了网桥（docker0）处；
docker0网桥开启了数据报转发功能（/proc/sys/net/ipv4/ip_forward），故将请求发送至宿主机的eth0处；、
宿主机处理请求时，使用SNAT对请求进行源地址IP转换，即将请求中源地址IP（容器IP地址）转换为宿主机eth0的IP地址；
宿主机将经过SNAT转换后的报文通过请求的目的IP地址（宿主机以外世界的IP地址）发送至外界。
在这里，很多人肯定会问：对于Docker Container内部主动发起对外的网络请求，当请求到达宿主机进行SNAT处理后发给外界，当外界响应请求时，响应报文中的目的IP地址肯定是Docker宿主机的IP地址，那响应报文回到宿主机的时候，宿主机又是如何转给Docker Container的呢？关于这样的响应，由于port_3端口并没有在宿主机上做相应的DNAT转换，原则上不会被发送至容器内部。为什么说对于这样的响应，不会做DNAT转换呢。原因很简单，DNAT转换是针对容器内部服务监听的特定端口做的，该端口是供服务监听使用，而容器内部发起的请求报文中，源端口号肯定不会占用服务监听的端口，故容器内部发起请求的响应不会在宿主机上经过DNAT处理。

其实，这一环节的内容是由iptables规则来完成，具体的iptables规则如下：

iptables -I FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
这条规则的意思是，在宿主机上发往docker0网桥的网络数据报文，如果是该数据报文所处的连接已经建立的话，则无条件接受，并由Linux内核将其发送到原来的连接上，即回到Docker Container内部。

以上便是Docker Container中bridge桥接模式的简要介绍。可以说，bridger桥接模式从功能的角度实现了两个方面：第一，让容器拥有独立、隔离的网络栈；第二，让容器和宿主机以外的世界通过NAT建立通信。

然而，bridge桥接模式下的Docker Container在使用时，并非为开发者包办了一切。最明显的是，该模式下Docker Container不具有一个公有IP，即和宿主机的eth0不处于同一个网段。导致的结果是宿主机以外的世界不能直接和容器进行通信。虽然NAT模式经过中间处理实现了这一点，但是NAT模式仍然存在问题与不便，如：容器均需要在宿主机上竞争端口，容器内部服务的访问者需要使用服务发现获知服务的外部端口等。另外NAT模式由于是在三层网络上的实现手段，故肯定会影响网络的传输效率。

3.2 host模式
Docker Container中的host模式与bridge桥接模式有很大的不同。最大的区别当属，host模式并没有为容器创建一个隔离的网络环境。而之所以称之为host模式，是因为该模式下的Docker Container会和host宿主机共享同一个网络namespace，故Docker Container可以和宿主机一样，使用宿主机的eth0，实现和外界的通信。换言之，Docker Container的IP地址即为宿主机eth0的IP地址。

Docker Container的host网络模式:

<p align="center">
<img width="600" align="center" src="src/images/7.jpg" />
</p>

图中简单明了的显示了的Docker Container中的host的模式和brdige模式的区别，图中左边采用了host网络模式，而其他两个Docker Container依然沿用brdige桥接模式，两种模式同时存在于宿主机上并不矛盾。