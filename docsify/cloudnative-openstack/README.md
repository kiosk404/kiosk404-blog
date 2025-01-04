# Openstack 学习

> The Most Widely Deployed Open Source Cloud Software in the World
Deployed by thousands. Proven production at scale. OpenStack is a set of software components that provide common services for cloud infrastructure.

本项目主要记录 Openstack 相关云原生相关知识，并加以自己的分析与注解。


# 内容更新
- 本人博客: [**kiosk's Blog**](https://kiosk007.top) 
  
- Openstack 官方网站：https://www.openstack.org/

# 重新认识 Openstack 
OpenStack是一个自由、开源的云计算平台。它主要作为基础设施即服务（IaaS）部署在公用云和私有云中，提供虚拟服务器和其他资源给用户使用。该软件平台由相互关联的组件组成，控制着整个数据中心内不同的厂商的计算、存储和网络资源的硬件池。用户可以通过基于网络的仪表盘、命令行工具或RESTful网络服务来管理。

<img src="https://img1.kiosk007.top/static/images/blog/20250104130421-openstack_readme.svg" style="display:block;margin:0 auto;">

## 核心服务

![openstack_readme_core_service](https://img1.kiosk007.top/static/images/blog/20250104130610-openstack_readme_core_service.png)

**全局组件：**

- `Keystone`：管理全局认证和授权的组件；
- `Ceilometer`：监控集群的状态，监控集群虚拟机的使用量；
- `Horizon`：控制台可以控制OpenStack架构内部的所有功能；

**辅助组件：**

- `Ironic`：裸金属管理和控制基础硬件资源；
- `Trove`：管理数据库的服务，管理关系型数据库和非关系型数据库，可以存储虚拟机和各组件调用的数据，以及各种日志；
- `Heat` 和 `Sahara`：做数据的分析，编排和处理，精细化的管理；

**核心组件：**为虚拟机/实例提供服务

- `Nova`：负责虚拟机实例的生命周期管理、网络管理、存储卷管理、用户管理以及其他的相关云平台管理功能，支持虚拟机核心资源的横向扩展，支持虚拟机数量的横向扩展（将资源提供给虚拟机）；
- `Neutron`：实现实例与实例之间以及实例与外部网络之间的通信；
- `Cinder`：提供对Volume从创建到删除整个生命周期的管理；
- `Glance`：提供发现、注册和下载的镜像服务，虚拟机镜像的集中式仓库；通过虚拟机镜像创建虚拟机，对镜像进行精细化管理，提供管理镜像的服务（快照），修改镜像的元数据；
- `Swift`：使用普通硬件来构建冗余的、可扩展的分布式对象存储集群，存储容量可达PB级。
  Swift属于对象存储，用于永久类型的静态数据的长期存储(如虚拟机镜像、图片存储、邮件存储和存档备份)。



