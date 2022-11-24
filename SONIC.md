# SONIC

## 简介

**官方解释：**sonic是一个**基于 Linux 的开源网络操作系统**，运行在来自多个供应商和 ASIC 的交换机上。**SONiC 提供全套网络功能**，如 BGP 和 RDMA。与ONL一样，SONiC需要通过ONIE环境安装系统到磁盘或者flash分区中。

SONIC的一些功能已在一些最大的云服务提供商的数据中心进行了生产强化。它为团队提供了创建所需网络解决方案的灵活性，同时利用大型生态系统和社区的集体力量。

------

**网上的解释：**SONiC是**构建网络设备（如交换机）所需功能的软件集合**。它可以通过交换机换抽象接口（SAI）运行在不同的ASIC平台。

<img src="https://img-blog.csdnimg.cn/6250d92e1453460cadc0c03446e1f9b4.png" alt="img" style="zoom:67%;" />

**SAI**向上给SONiC提供了一套统一的API接口，向下则对接不同的ASIC。

SONiC是一个将传统交换机操作系统软件分解成多个**容器化组件**的创新方案，这使得增加新的组件和功能变得非常方便。

SONiC大量使用了现有的**开源**项目和开源技术，如Docker，Redis，Quagga和LLDPD 以及自动化配置工具Ansible、Puppet和Chef等。

SONiC只是**构建交换机网络功能的软件集合**，**它需要运行在Base OS上**。SONiC所使用的Base OS 是ONL (Open Network Linux ) 。ONL是一款为白盒交换机而设计的开源Linux操作系统，ONL中包括了许多硬件（温度传感器、风扇、电源、CPLD控制器等）的驱动程序。

将SONiC和Base OS、SAI、ASIC平台对应的驱动打包制作成为一个文件，这个文件才是可直接安装到白盒交换机的**NOS镜像** 。

**SONiC与传统网络的区别**，如下图所示，左侧代表SONiC，右侧为传统网络架构，LAG App，BGP App等都属于控制层，生成控制面的表项，SAI，ASIC Control Software相当于软转发层面，用来将控制面的数据转化为ASIC的数据，然后下发驱动。硬件转发层包含SDK和硬件。

<img src="https://img-blog.csdnimg.cn/6b4244a848024050a7608b433a66d573.png" alt="img" style="zoom: 67%;" />

**总结来说，SONIC就是一个开源的linux网络操作系统，它能提供的功能使用docker分成多个容器，一个功能对应一个docker容器，可进行灵活的拆卸。然后下层使用SAI充当软硬件的中间对接层，用来实现其标准化。再向下就是到Base OS根据具体的交换机提供相应的驱动程序。在向下最后就是最底层的硬件。**



## SONIC系统架构

SONIC系统按功能将整个系统划分成多个模块，每个模块是一个独立的docker容器，一个docker由多个进程共同完成这个模块的功能。在SONIC中使用`show services`命令查看每个docker及其进程状态。

这个结构依赖于**`redis-database`**引擎的使用：即一个键-值数据库。它提供一个独立于语言的接口，一个用于所有SONiC子系统之间的数据持久性、复制和多进程通信的方法。

通过依赖`redis-database`基础架构提供的`publisher/subscriber` 的消息传递模式，应用程序可以仅订阅它们需要的数据视图，并避免与其功能无关的实现细节。

SONiC将每个模块放置在独立的docker容器中，以保持组件之间的高内聚性，同时减少不相连组件之间的耦合。这些组件中的每一个都被编写为完全独立于与较低层抽象交互所需的特定于平台的细节。

截至当前主要有以下**docker容器（不重要，下面会说）**

- Teamd：运行并实现链路聚合（LAG）功能。
- Pmon：记录硬件传感器读数并发出警报。
- Snmp：实现SNMP功能。
- Dhcp-relay：将DHCP请求从没有DHCP服务器的子网中连接到其他子网上的一台或多台DHCP服务器。
- Lldp：实现链路层发现协议功能。建立lLLDP连接。
- Bgp：运行支持的路由协议之一，例如ospf，isis，ldp，bgp等。
- Database：redis-engine托管的主要数据库。
- Swss：实现所有SONiC模块之间进行有效通信和与SONiC应用层之间的交互。监听并推送各个组件的状态。
- Syncd：实现交换机网络状态和实际硬件进行同步

下图显示了每个 docker-container 中包含的功能的高级视图，以及这些容器之间如何交互。请注意，并非所有 SONiC 应用程序都与其他 SONiC 组件交互，因为其中一些应用程序从外部实体收集它们的状态。使用蓝色箭头表示与集中式 redis 引擎的交互，使用黑色箭头表示所有其他引擎（netlink、/sys 文件系统等）。

尽管 SONiC 的大部分主要组件都包含在 docker 容器中，但在 linux 主机系统本身中也有一些关键模块。SONiC 的配置模块 (sonic-cfggen) 和 SONiC 的 CLI 就是这种情况。

<img src="https://github.com/Azure/SONiC/raw/master/images/sonic_user_guide_images/section4_images/section4_pic1_high_level.png" alt="Sec4Img1" style="zoom: 80%;" />

## SONIC各模块功能的说明

### Teamd container（一种数据链路的协议的应用）

在 SONiC 设备中运行链路聚合功能 (LAG:Link Aggregation functionality)。“teamd”是 LAG 协议的基于 linux 的开源实现。“teamsyncd”进程允许“teamd”和南向子系统之间的交互。

**ps:** LAG：链路聚合是在两个设备间使用多个物理链路创建一个逻辑链路的功能，这种方式允许物理链路间共享负载。交换机网络中使用的一种链路聚合的方法是EtherChannel。EtherChannel可以通过协议PAGP（Port Aggregation Protocol）或**LACP**（Link Aggregation Protocol）来配置。

### **Pmon container**（用于硬件的检测与报警器）

负责运行“sensord”，这是一个守护进程，用于定期记录来自硬件组件的传感器读数，并在发出警报时发出警报。Pmon 容器还托管“fancontrol”进程以从相应的平台驱动程序收集与风扇相关的状态。

### **Snmp container**（**用来收集网络交换机的状态**）

托管 snmp 功能。此容器中有两个相关进程：

- **Snmpd：**负责处理来自外部网络元素的传入 snmp 轮询的实际 snmp 服务器。
- **Snmp-agent (sonic_ax_impl)：**这是 SONiC 对 AgentX snmp 子代理的实现。该子代理向主代理 (snmpd) 提供从中央 redis 引擎中的 SONiC 数据库收集的信息。

**ps:** SNMP(Simple Network Management Protocol)：应用层协议，靠UDP进行传输。**常用于对路由器交换机等网络设备的管理**，管理人员通过它来**收集网络设备运行状况，了解网络性能、发现并解决网络问题**。SNMP分为管理端和代理端(agent)，管理端的默认端口为UDP 162，主要用来接收Agent的消息如TRAP告警消息;Agent端使用UDP 161端口接收管理端下发的消息如SET/GET指令等。

### **Dhcp-relay container**（将没有DHCP与有DHCP的组子网进行连接）

dhcp-relay 代理可以将 DHCP 请求从没有 DHCP 服务器的子网中继到其他子网的一台或多台DHCP服务器。

**ps:** DHCP（Dynamic Host Configuration Protocol）：动态主机配置协议，是一个应用层协议。当我们将客户主机ip地址设置为动态获取方式时，DHCP服务器就会根据DHCP协议给客户端分配IP，使得客户机能够利用这个IP上网。**集中的管理、分配IP地址**，使网络环境中的主机动态的获得IP地址、Gateway地址、DNS服务器地址等信息。工作过程如下图所示。

<img src="https://img-blog.csdnimg.cn/119c859e59874b4e936be6dd44ef9d72.png" alt="img" style="zoom: 33%;" />

### **Lldp container**（实现LLDP功能）

顾名思义，此容器承载 LLDP 功能。这些是在此容器中运行的相关进程：

- **Lldp**：具有 lldp 功能的实际 lldp 守护进程。这是与外部对等方建立 lldp 连接以通告/接收系统功能的过程。
- **lldp_syncd**：负责将lldp的发现状态上传到中心化系统的消息基础设施（redis-engine）的进程。通过这样做，lldp 状态将被传递给对使用此信息感兴趣的应用程序（例如 snmp）。
- **Lldpmgr**：进程为 lldp 守护进程提供增量配置功能；它通过在 redis 引擎中订阅 STATE_DB 来实现。有关本主题的详细信息，请参阅下文。

**ps:** LLDP（Link Layer Discovery Protocol）：链路层发现协议。设备通过在网络中发送LLDPDU（Data Unit）来通告其他设备自身的状态（理地址，设备标识，接口标识等）。可以使不同厂商的设备在网络中相互发现并**交互各自的系统及配置信息**。 当一个设备从网络中接收到其它设备的这些信息时，它就将这些信息以MIB的形式存储起来。**LLDP只传输，不管理**。

### BGP container()

运行支持的路由协议之一，例如**ospf，isis，ldp，bgp**等。

**bgpd**：路由的实现。外部的路由状态通过常规的tcp/udp sockets 接收，并通过zebra / fpmsyncd接口下推到转发平面。

**zebra**：充当传统的IP路由管理。它提供内核路由表的更新，接口的查找和路由的重新分配。将计算出的FIB下推到内核（通过netlink接口）和转发过程中涉及的南向组件（通过Forwarding-Plane-Manager，FPM接口）

**fpmsyncd**：收集zebra下发的FIB状态，将内容放入redis-engine托管的APPL-DB中。

**ps**：**FIB**（Forward Information dataBase）：转发信息库。路由一般手段：先到**路由缓存**（RouteTable）中查找表项，如果能查找到，就直接将对应的一项作为路由规则；如果查不到，就到FIB中根据规则换算出来，并且增加一项新的，在路由缓存中将项目添加进去。

**ps:** **BGP边界网关协议**（Border Gateway Protocol）是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或“前缀”表来实现自治系统（AS）之间的可达性，属于矢量路由协议。BGP不使用传统的内部网关协议（IGP）的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。

### **数据库 container**（使用socket可以访问的数据库功能）

托管 redis 数据库引擎。**SONiC 应用程序可以通过 redis-daemon 为此目的公开的 UNIX 套接字访问此引擎中保存的数据库**。这些是 redis 引擎托管的主要数据库：

- **APPL_DB**：存储所有应用程序容器生成的状态——路由、下一跳、邻居等。这是所有希望与其他 SONiC 子系统交互的应用程序的南向入口点。
- **CONFIG_DB**：存储由 SONiC 应用程序创建的配置状态——端口配置、接口、vlan 等。
- **STATE_DB**：存储系统中配置的实体的“关键”操作状态。此状态用于解决不同 SONiC 子系统之间的依赖关系。例如，LAG portchannel（由 teamd 子模块定义）可能指的是系统中可能存在或不存在的物理端口。另一个例子是 VLAN 的定义（通过 vlanmgrd 组件），它可能引用在系统中存在未确定的端口成员。本质上，这个数据库存储了所有被认为是解决跨模块依赖关系所必需的状态。
- **ASIC_DB**：存储必要的状态来驱动 asic 的配置和操作——这里的状态以 asic 友好的格式保存，以简化 syncd（详见下文）和 asic SDK 之间的交互。
- **COUNTERS_DB**：存储与系统中每个端口关联的计数器/统计信息。此状态可用于满足 CLI 本地请求，或为远程消费提供遥测通道。

### **Swss container**（用于SONIC各模块之间进行沟通，由一组工具组成）

开关状态服务 (SwSS) 容器由一组工具组成，**允许所有 SONiC 模块之间进行有效通信。**如果说数据库容器擅长提供存储功能，那么 Swss 主要侧重于提供机制来促进各方之间的沟通和仲裁。

Swss 还托管负责与 SONiC 应用层进行北向交互的进程。如前所述，例外情况是 fpmsyncd、teamsyncd 和 lldp_syncd 进程，它们分别在 bgp、teamd 和 lldp 容器的上下文中运行。无论这些进程在何种环境下运行（在 swss 容器内部或外部），它们都有相同的目标：提供允许 SONiC 应用程序和 SONiC 的集中式消息基础设施（redis 引擎）之间连接的方法。这些守护进程通常由正在使用的命名约定来标识：*syncd。

https://github.com/sonic-net/SONiC/wiki/Architecture#sonic-subsystems-description

### **Syncd container**（用于同步网络状态和实施的硬件）

简而言之，**syncd 的容器目标是提供一种机制**，**允许交换机的网络状态与交换机的实际硬件/ASIC 同步**。这包括交换机的 ASIC 当前状态的初始化、配置和收集。

这些是 syncd 容器中存在的主要逻辑组件：

- **Syncd：**负责执行上述同步逻辑的进程。在编译时，syncd 链接硬件供应商提供的 ASIC SDK 库，并通过调用为此效果提供的接口将状态注入 ASIC。Syncd 订阅 ASIC_DB 以从 SWSS 参与者接收状态，同时注册为发布者以推送来自硬件的状态。
- **SAI API：**交换机抽象接口 (SAI) 定义了 API，以提供独立于供应商的方式来控制转发元素，例如以统一方式交换 ASIC、NPU 或软件交换机。有关 SAI API 的更多详细信息，请参阅 [3]。
- **ASIC SDK：**硬件供应商应提供驱动其 ASIC 所需的 SDK 的 SAI 友好实现。此实现通常以动态链接库的形式提供，它连接到负责驱动其执行的驱动进程（在本例中为 syncd）。

### **CLI / sonic-cfggen container**（实现CLI功能）

**负责提供 CLI 功能和系统配置功能**的 SONiC 模块。

- **CLI** 组件严重依赖 Python 的 Click 库来为用户提供灵活且可定制的方法来构建命令行工具。
- **Sonic-cfggen** 组件由 SONiC 的 CLI 调用以执行配置更改或任何需要与 SONiC 模块进行配置相关交互的操作。

**ps:**CLI（Command-Line Interface）命令行界面。



## SONIC子系统交互

sonic系统里前面的那些模块与state的互动（也就是每个模块与redis-database的互动）就不再赘述，详情可以看官方文档https://github.com/sonic-net/SONiC/wiki/Architecture







