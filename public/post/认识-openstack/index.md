# 认识 Openstack


最近机缘巧合，需要重新认识一下 openstack 这门技术，之前其实也接触过很多虚拟化的技术，但是不够统一，也打算将之前了解的虚拟化技术好好的回顾一遍。



<!--more-->

# 什么是OpenStack？

官方给出的定义是：

> Openstack controls large pools of compute, storage, and networking resources, all managed through APIs or a dashboard.
>
> Beyond standard infrastructure-as-a-service functionality, additional components provide orchestration, fault management and service management amongst other services to ensure high availability of user applications.

简单的翻译一下：

[OpenStack](https://www.openstack.org/) 是一个可以控制真个数据中心里大量的计算、存储和网络资源池的云操作系统，他通过一个既能赋予管理员控制资源的能力，也能让普通用户调配资源的可视化控制面板（Dashboard）来管理这一切。



<img src="https://img1.kiosk007.top/static/images/blog/openstack_overview.jpg" alt="openstack_overview" style="zoom:40%; clear: both; margin: auto; display: block;" />

<br/>

OpenStack 是由一系列开源组件构成的基础设施即服务（Infrastructure as a Service，简称 IaaS）范畴的云平台解决方案，它让用户方便的构建和管理自己的云平台。

OpenStack 要解决的问题就是如何管理物理主机上虚拟出来的虚拟机和虚拟资源。

虚拟资源主要由**计算**、**网络**、**存储** 三种构成，OpenStack 有对应的组件去管理这些资源。其组件的源码在 [https://github.com/openstack/](https://github.com/openstack/)



## 主要组件以及作用

| Project                                                      | Service                   | Catalog                         |
| ------------------------------------------------------------ | ------------------------- | ------------------------------- |
| {{< image src="https://img1.kiosk007.top/static/images/blog/nova_logo.webp" caption="**NOVA**" src_s="https://img1.kiosk007.top/static/images/blog/nova_logo.webp"  >}} | Compute Service 计算服务  | Compute                         |
| {{< image src="https://img1.kiosk007.top/static/images/blog/clance_logo.webp" caption="GLANCE" src_s="https://img1.kiosk007.top/static/images/blog/clance_logo.webp"  >}} | Image Service 镜像服务    | Compute                         |
| {{< image src="https://img1.kiosk007.top/static/images/blog/swift_logo.webp" caption="SWIFT" src_s="https://img1.kiosk007.top/static/images/blog/swift_logo.webp"  >}} | Object Store 对象存储     | Storage，Backup & Recovery      |
| {{< image src="https://img1.kiosk007.top/static/images/blog/cinder_logo.webp" caption="CINDER" src_s="https://img1.kiosk007.top/static/images/blog/cinder_logo.webp"  >}} | Block Storage 块存储      | Storage，Backup & Recovery      |
| {{< image src="https://img1.kiosk007.top/static/images/blog/keystone_logo.webp" caption="KEYSTONE" src_s="https://img1.kiosk007.top/static/images/blog/keystone_logo.webp"  >}} | Identity Service 认证服务 | Security，Identity & Compliance |
| {{< image src="https://img1.kiosk007.top/static/images/blog/neutron_logo.webp" caption="NEUTRON" src_s="https://img1.kiosk007.top/static/images/blog/neutron_logo.webp"  >}} | Networking 网络服务       | Networking & Content Delivery   |
| {{< image src="https://img1.kiosk007.top/static/images/blog/horizon_logo.webp" caption="HORIZON" src_s="https://img1.kiosk007.top/static/images/blog/horizon_logo.webp"  >}} | Dashboard 管理界面        | Management Tools                |

<br/>

组件内部也并不是完全一体，紧密耦合的状态，这样方便了组件的扩展和维护，组件之间也并不是孤立的。

<img src="https://img1.kiosk007.top/static/images/blog/openstack_level.png" alt="openstack_level" style="zoom:80%;clear:both;margin:auto;display:block;" />

<br/>

# Openstack 组成



![openstack_draft](https://img1.kiosk007.top/static/images/blog/openstack_draft.png)

<br/>

# Nova

Nova 在 OpenStack 中提供计算资源服务的项目，管理OpenStack 云中实例的生命周期的所有活动；是管理计算资源、网络认证所需的可扩展平台。

其**负责**虚拟机生命周期的管理和其他计算资源生命周期的管理。**不负责**承载虚拟机的物理主机自身的管理和全面系统状态的监控。

事实上Nova是OpenStack事实上最核心的项目。

[https://github.com/openstack/nova](https://github.com/openstack/nova)

## Nova 架构

常见术语解释

| 名词         | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| KVM          | 内核虚拟化，是基于内核的虚拟化技术，也是 OpenStack 中默认的 Hypersvisor 层 |
| Qemu         | 辅助KVM，提供存储、网络等虚拟化技术，不支持全虚拟化          |
| Flavor       | 新建虚拟机的配置列表，可以认为是创建虚拟机的模板，比如一种具有2个 VCPU、4GB内存、40GB存储的虚拟机 |
| Keypair      | ssh 连接访问实例的密钥对                                     |
| Server Group | 虚拟机亲和性/反亲和性，同一个亲和性组的虚拟机，在创建时会被调度到相同的物理主机上，同一个反亲和性组的虚拟机，在创建时会被调度到不同的物理主机上 |

Nova组件由**Nova-API**，**Nova-scheduler**,**Nova-conductor**,**Nova-compute**以及提供**消息传递的消息队列**（message queue）和**数据裤模块**（database）组成



- **Nova-API**：主要提供了统一风格的REST-API 接口，作为 Nova 组件的入口，接受用户的请求。
- **Nova-scheduler**：负责调度将实例分配到具体的计算节点
- **Nova-conductor**：主要负责与Nova数据库进行交互
- **Nova-compute**：计算节点，用于虚拟机实例的创建和管理
- **Message Queue**：主要用于Nova各个组件之间的消息传递

<br/>

## Nova 架构中各个组件协作

<img src="https://img1.kiosk007.top/static/images/blog/openstack_nova.png" alt="openstack_nova" style="zoom:40%;clear:both;margin:auto;display:block;" />

首先用户通过 CLI 命令行或 Horizon 向 Nova 组件提出创建实例的请求，

1. Nova-API作为Nova的入口，将会接受用户的请求，然后以消息队列的方式，将请求发送给 Nova-scheduler
2. Nova-scheduler 从消息队列中获取到Nova-API的消息后，去数据库中查询当前计算节点的负载和使用情况。
3. 由于Nova-scheduler不能直接和数据库交互，因此会借助与消息队列的方式，通过 Nova-conductor 组件与数据库进行交互，最后根据查询的结果，将虚拟机分配到当前负载最小并且满足启动虚拟机实例的计算节点上。
4. 最终实例的创建需要Nova-compute来完成，期间也离不开镜像、网络、存储等一些资源的配合，因此Nova-compute组件将会与 Nova-valume, Nova-network 等组件交互，最终完成虚拟机实例的创建。

<br/>

# Cinder

Cinder 在OpenStack 中为虚拟机实例提供 vloume 的块存储服务，可将卷挂载到实例上，作为虚拟机实例的本地磁盘使用。

一个 volume 卷可以同时挂载到多个实例上，但是同时只能有一个实例对卷进行写操作，其他都是只读的。

**Cinder 支持的逻辑卷系统类型**：LVM/LSCSI、NFS、NetAPP NFS、Gluster、DELL Equall Logic。

[https://github.com/openstack/cinder](https://github.com/openstack/cinder)



## Cinder 架构

常见术语解释

| 名词       | 解释                                    |
| ---------- | --------------------------------------- |
| Volum 备份 | 指的是 Volum 卷的备份存放在备份的设备中 |
| Volum 快照 | 指的是卷在某个时间点的状态              |

Cinder组件由 **Cinder API**、**Cinder Scheduler**、**Cinder Volume**、**Cinder Backup** 等组成。



- **Cinder API**：为 Cinder 请求提供统一风格的 Rest API服务，用来接收Cinder的请求，是Cinder服务的入口
- **Cinder Scheduler**：主要负责调度为新建卷分配块存储节点
- **Cinder Volume**：主要负责与存储的块设备交互，实现卷的创建、删除、修改等操作。
- **Cinder Backup**：备份服务，主要负责通过驱动和后端的备份设备打交道，备份的后端比如 Swift 组件。

<br/>

## Cinder 各架构组件的协作

<img src="https://img1.kiosk007.top/static/images/blog/openstack_cinder.png" alt="openstack_cinder" style="zoom:40%;clear:both;margin:auto;display:block;" />

首先由用户通过 CLI 命令行 或者 Nova-compute 提出创建卷的请求。

1. 由 Cinder API 接收请求，然后以消息队列的方式发送给 Cinder Scheduler 来调用。
2. Cinder Scheduler 侦听到来自Cinder API的消息队列后，到数据库中查询当前存储节点的状态信息，并根据预定策略，选择卷的最佳 Volume Service 节点，然后将调度的结果发布给 Volume Service 来调用。
3. 当Volume Service 收到 Volume Scheduler 的调度结果之后，会去查找 volume provider，从而在特定的存储节点上创建相关的卷，然后将相关的结果返回给用户，同时将修改的数据写入到数据库中。

<br/>

# Neutron

Neutron 是 OpenStack 中负责提供网络服务的组件，基于软件定义网络的思想，实现网络化的网络资源管理，在实现上充分利用了 Linux 系统中的各种网络相关技术，支持第三方插件。

[https://github.com/openstack/neutron](https://github.com/openstack/neutron)



## Neutron 架构

常见术语解释

| 名词       | 解释                                                 |
| ---------- | ---------------------------------------------------- |
| Bridge-int | 指的是综合网桥，常用语表示实现内部网络通信功能的网桥 |
| Br-ex      | 外部网桥，通常用于跟外部网络通信的网桥               |

Neutron 组件由 **Neutron-server**、**Neutron-L2-agent**、**Neutron-DHCP-agent**、**Neutron-L3-agent**、**Neution-metadata-agent**、**LBaas agent**



- **Neutron-server**：提供了API接口，把API的调用请求传递给已经配置好的插件，进行后续处理。
- **Neutron-L2-agent**：二层代理，用语管理 VLan 插件，接受 Neutron-server 的指令创建 Vlan
- **Neutron-DHCP-agent**：是OpenStack中用语创建子网，并为每一个创建的子网实现IP地址的自动分配的组件
- **Neutron-L3-agent**：负责租户网络和 floating IP 之间的地址转换，通过 Linux IP tables 的nat 功能实现地址转换
- **Neution-metadata-agent**：运行在网络节点上，用来响应 Navo 组件的 metadata 请求
- **LBaas agent**：主要为多台实例和 openvswitch agent 提供负载均衡服务。

<br/>

## Neutron 各架构组件的协作

<img src="https://img1.kiosk007.top/static/images/blog/openstack_neutron.png" alt="openstack_neutron" style="zoom:40%;clear:both;margin:auto;display:block;" />

当 Neutron 通过 API 接口接到来自用户或者其他组件的网络请求时。

1. 以消息队列的方式提交给二层或者三层代理
2. 其中 DHCP-agent 实现子网的创建和IP地址的自动分发
3. L2-agent实现相同 vlan 下的网络通信
4. L3-agent实现同一个租户网络下不同子网间的通信

<br/>

# Keystone

Keystone 在 OpenStack 框架中负责身份验证、服务规则和服务令牌的功能。提供了认证服务的API，为OpenStack中的各个组件提供了认证服务，类似于一个服务的总线，或者说是提供了整个OpenStack框架的注册表。

其他的服务都是通过 Keystone来注册其服务的 Endpoint，也就是服务的 URL 路径，来实现服务之间的相互访问与调用。

[https://github.com/openstack/keystone](https://github.com/openstack/keystone)



## Keystone 架构

常见术语解释

| 名词     | 解释                                                         |
| -------- | ------------------------------------------------------------ |
| User     | OpenStack 中最基本的用户，进行访问的人或者程序               |
| Project  | 项目或者是租户，指的是分配给使用者资源的集合，比如给租户A，我们可以分配 50 个实例，100 核CPU、200G内存 |
| Role     | 角色，代表一组用户可以访问资源的权限                         |
| Domain   | 域，定义了管理的边界，是最上层的一个集合，域里面可包含很多的 Project/tenant，user，role 等等 |
| Endpoint | 指的是服务的URL路径，可以认为是服务暴露出来的访问点。        |

**Keystone 认证模型**

<img src="https://img1.kiosk007.top/static/images/blog/openstack_keystone_explanation.png" alt="openstack_keystone_explanation" style="zoom:40%;clear:both;margin:auto;display:block;" />

首先创建两个域，集团1 和 集团2 。

集团下面可能还有分公司，他们分别创建三个 project 给三个分公司使用，在project 下创建不同的用户，用户可以跨多个project 存在。



## Keystone 认证原理

<img src="https://img1.kiosk007.top/static/images/blog/openstack_keystone.png" alt="openstack_keystone" style="zoom:40%;clear:both;margin:auto;display:block;" />

Keystone 如何实现身份验证？

**当用户在创建时，通过Keystone将会申请一张访问令牌的token**

1. 假设当用户提出创建虚拟机实例的请求时，首先将自己的访问令牌和访问请求，提交给Nova服务
2. Nova服务为确保用户的访问令牌没有被篡改过，因此Nova首先将访问令牌交给Keystone进行验证
3. 验证通过后，Nova为了启动虚拟机实例，还需要向 Glance 组件申请相关的镜像资源，Glance 为确保访问令牌在传递过程中没有被篡改过，也需要把令牌发送给 Keystone 确认，验证通过后将会发放镜像资源给 Nova 组件
4. 当然虚拟机实例的创建除了需要镜像文件，还需要网络、存储等资源，因此 Nova组件还需要给负责各种资源的模块来传递资源申请的请求，这些资源申请过程中都会伴随着访问令牌验证
5. 最终 Nova 拿到启动虚拟机实例的所有资源以后，进行实例的启动，然后分配给相关用户

<br/>

# Glance

Glance 是 OpenStack 中负责为Nova提供镜像服务，以便启动实例的组件，但通常不负责镜像的本地存储，可以实现对镜像做快照，备份镜像模板的管理。

Glance 支持的镜像格式：

- raw：非结构化的镜像格式
- vhd：一种通用的虚拟机磁盘格式，可用于Vmaware、Xen、Microsoft Virtual PC/Virtual Server/Hyper-V、VirtualBox 等
- vmdk：Vmware 的虚拟机磁盘格式，同样支持多种Hypervisor
- vdi：VirtualBox、QEMU 等支持的虚拟机磁盘格式
- iso：光盘存档格式
- qcow2：一种支持QEMU并可动态扩展的磁盘格式

[https://github.com/openstack/glance](https://github.com/openstack/glance)



## Glance 架构

Glance 由 Glance-API 和 Glance-registry 两个组件构成。

- **Glance-API**： 负责提供镜像服务的 rest api 服务，作为镜像服务请求的入口
- **Glance-registry**：主要负责与Glance使用的数据库的交互

比如镜像的创建、删除、修改等操作都会通过 Glance-registry 来与数据库交互来实现。

## Glance 各架构组件的协作

<img src="https://img1.kiosk007.top/static/images/blog/openstack_glance.png" alt="openstack_glance" style="zoom:40%;clear:both;margin:auto;display:block;" />

当有 Horizon、Glance-CLI 或者是 Nova-Compute 组件发送过来的镜像请求时，由 Glance-API接收处理

1. Glance-API将请求的消息传递给 Glance-registry 组件
2. Glance-register 组件到数据库中去查询镜像存储的位置信息返回给API
3. API将调用 Storage adapter 组件进行查询，用来查询 Glance 后端的存储，比如 Swift 、Ceph、Amazon S3 等
4. 最终获取镜像，返回给相关用户。



<br/>

# Ceilometer

Ceilometer 是 OpenStack 的子项目，它像一个漏斗一样，把 OpenStack 内部发生的几乎所有事件都收集起来，然后为计费、监控、以及其他的服务提供数据支撑。

之所以需要 Ceilometer 是因为：OpenStack 作为一个开源的 IaaS 平台，发展的速度越来越快，那么越来越多的公司基于OpenStack 做自己的共有云平台，而作为共有云，计量和监控这两个基础的服务往往是必不可少的，计量是为了获取平台中用户对各种资源的使用情况。

监控是为了保证资源处于一个健康的状态，因此 Ceilometer 在项目提出之初，它的定位就是专注于计量、计费。

[https://github.com/openstack/ceilometer](https://github.com/openstack/ceilometer)



## Ceilometer 架构

Ceilometer 由五个重要的组件以及一个 Message Bus 组成。

- **Ceilometer-agent-computer**：运行在计算节点上，是计算节点上数据收集的代理
- **Ceilometer-agent-contral**：运行在控制节点上，轮询服务的非持续化数据
- **Ceilometer-collector**：运行在一个或多个控制节点上，主要作用是监控消息总线，将收到的消息以及相关的数据写入到数据库中
- **Storage**：数据存储，支持 MySQL、MongoDB，用于存储收集到的样本数据
- **API Server**：运行在控制节点上，提供对数据库的数据访问
- **Message Bus**：计量数据的消息总线，收集数据给 collector

## Ceilometer 各架构组件的协作

<img src="https://img1.kiosk007.top/static/images/blog/openstack_ceilometer_fix.png" alt="openstack_glance" style="zoom:40%;clear:both;margin:auto;display:block;" />

在上图中 Ceilometer 组件采用了两种数据采集的方式：

- 其中一种是消费了 OpenStack 内各个服务自动发出的Notification 消息，对应图中Notification Message Bus
- 另一种是通过各个服务的API去主动轮询获取数据，对应图中 Central Pollsters 服务引出的箭头。

**采用两种方式的原因是**：

1. 因为在 OpenStack 内部，大部分事件都会发出 Notification 消息。 比如创建、删除实例的时候，这些计量、计费的重要信息时，都会发出对应的 Notification 消息。Ceilometer 组件是 Notification 消息最大的消费者，所以第一种方式是Ceilometer的首要数据来源。
2. 有些消息 Notification 获取不到，比如实例的CPU运行时间，CPU的使用率等等，这些信息不会通过Notification发出，因此Ceilometer 增加了第二种方式，即周期性的调用相关的 API 去轮询这些消息



<br/>

# Swift

Swift 是OpenStack中提供高可用分布式对象存储的服务，为Nova子项目提供虚拟机镜像存储服务，在软件层引入一致性散列技术和数据冗余，牺牲一定程度的数据一致性来达到高可用和伸缩性。支持多租户模式下，容器和对象的读写操作。

**适用于互联网场景下非结构化数据的存储**，如华为云盘等。

Swift 组件在物理结构上往往会保存存储对象的多个副本，通常按照物理位置的特征，将对象拷贝到不同对象的物理位置上，来保证数据地理位置上的可靠性。

[https://github.com/openstack/swift](https://github.com/openstack/swift)



## Swift 架构

常用术语

| 名词      | 解释                                                         |
| --------- | ------------------------------------------------------------ |
| Account   | 不是传统意义上的用户，而是用户定义的管理存储区域             |
| Container | 容器，存储的隔间，类似于子文件夹或者是目录                   |
| Object    | 对象，包含了我们上传的每一个基本的存储实体和它自身的元数据   |
| Ring      | 环，记录了磁盘上存储的实体名称和物理位置的映射关系，有 Account 环，Container 环和 Object 环 |

**术语间关系**

<img src="https://img1.kiosk007.top/static/images/blog/openstack_swift_explanation.png" alt="openstack_swift_explanation" style="zoom:40%;clear:both;margin:auto;display:block;" />

常用概念：

- **Region**：地域，地理位置上的一个概念，往往代表着不同城市的地理位置，从灾备考虑的概念
- **Zone**：可用区，强调的是物理的网络，供电，空调等基础设施的隔离，不同的可用区可能是同一个城市的不同数据中心机房，也可能是同一个数据中心，不同的供电、供水、网络、接入等隔离的系统
- **Node**：节点，表示一台存储服务器
- **Disk**：磁盘，代表着物理的存储设备
- **Cluster**：集群，是为冗余考虑而设计的架构

**常用概念间的关系**

<img src="https://img1.kiosk007.top/static/images/blog/openstack_swift_concept.png" alt="openstack_swift_concept" style="zoom:40%;clear:both;margin:auto;display:block;" />

<br/>

## Swift 各架构组件的协作

首先**用户提出了一个对象存储资源的申请**，由Swift的API接收和处理，**申请收到以后先去找Keystone认证结点**，**对于用户的身份进行认证**，

认证通过后将请求提交给名称为Swift Proxy的组件。

Swift Proxy 是 Swift 的代理，由它来确定，究竟应该将存储对象放在哪一个满足存储要求的存储节点上，最终将对象存储到指定的存储节点上，最终返回用户。

<br/>

# Heat

heat 是基于模板来创建相关资源的服务，是OpenStack 的核心项目之一，可以通过 yaml 文件生成模板，通过 heat-agent 组件在 OpenStack 中创建相关的资源，模板支持丰富的资源类型，不仅覆盖了常用的基础架构，包括计算、网络、存储、镜像等。

[https://github.com/openstack/heat](https://github.com/openstack/heat)



## Heat 架构

常用术语

| 名词          | 解释                                                      |
| ------------- | --------------------------------------------------------- |
| Stack         | 指的是 heat 要用到的所有设施和资源的集合                  |
| Heat template | 指的是 heat 模板，往往以 .yaml 结尾的文件，用于创建 Stack |
| Resouerce     | 底层各种服务的抽象集合，比如计算、存储、网络等等          |

Heat 由 **Heat-api**、**Heat-engine**、**Heat-cfntools**、**Heat-initial**、**heat-api-cloudwatch**、**heat-client** 等组件构成



- **Heat-api**：提供了 rest api 接口，是 heat 服务的入口，将 API 请求发送给 heat-engine 去执行
- **Heat-engine**：heat 的核心模块，接收api的请求在 OpenStack 中创建相关的资源
- **Heat-cfntools、Heat-initial**：往往是镜像中安装完成虚拟机实例操作任务的工具
- **Heat-api-cloudwatch**：编排服务的监控组件
- **Heat-client**：用于访问其他各个组件的Client 工具



## Heat 各架构组件交互逻辑

<img src="https://img1.kiosk007.top/static/images/blog/openstack_heat_explanation.png" alt="openstack_heat_explanation" style="zoom:47%;clear:both;display:block;margin:auto" />

当用户在 Horizon 中或者命令行（CLI）提交包含模板和参数的创建实例的请求时，Heat 服务接收请求，调用 Heat-api 或者 Heat-api-cfn。然后 heat-api 或者 heat-api-cfn 首先验证模板的正确性。

验证通过后，通过消息队列，异步传输给 heat-engine 进行处理。

<img src="https://img1.kiosk007.top/static/images/blog/openstack_heat.png" alt="openstack_heat" style="zoom:40%;clear:both;margin:auto;display:block;" />

接下来，当 heat-engine 拿到这种请求以后，会把请求解析为各种资源类型，而每种资源都有相关的Client 与对应服务的资源映射。

然后 client 通过发送 rest 的请求给其他的服务，从而获取其他的资源，最终完成请求的处理，heat-engine 在整个过程中的作用分3层。

- **第一层**：处理 heat 层面的请求，然后根据模板和输入的参数来创建 Stack，这里面的Stack 是由各种资源集合而成。
- **第二层**：解析Stack里面各种资源的依赖关系以及 Stack 的嵌套关系
- **第三层**：根据解析出来的关系，一次调用各种服务的client来创建各种资源。




