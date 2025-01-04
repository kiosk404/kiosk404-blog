# Openstack 部署

本次安装环境是使用 ubuntu libvirtd 虚拟出3台 ubunut 虚拟机。
官方的安装方式：[openstack Installation overview](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/yoga/install-overview.html)

部署环境：ubuntu 24.10
使用 [libvirtd](https://www.libvirt.org/manpages/libvirtd.html) 启动虚拟机。
节点架构：1个controller控制节点、1个compute计算节点、1个cinder块存储节点

<br/>

# 节点硬件规划

|节点名称 | 节点角色|CPU|内存|磁盘|
|--|--|--|--| --|
| instance-vm00 | controller | 2C | 4G | 50G |
| instance-vm01 | compute-1  | 2C | 4G | 50G |
| instance-vm02 | cinder-1   | 2C | 4G | 50G |

<br/>

# 节点网络规划
本次搭建网络使用linuxbridge+vxlan模式，包含三个网络平面：管理网络，外部网络和租户隧道网络

|节点名称| 节点角色|网卡编号|网卡模式|网络类型|IP 地址|网关|
|--|--|--|--|--|--|--|
| instance-vm00 |controller | enp7s0 | 隔离模式 | 管理网 | 172.16.0.10 |
|  || enp8s0 | 隔离模式 | 隧道网络	 | 172.16.1.10 |
|  || enp1s0 | NAT模式 | 外部网络	 | 192.168.120.10 | 192.168.120.1 |
| instance-vm01 | compute-1 | enp7s0 | 隔离模式 | 管理网 | 172.16.0.20 |
|  || enp8s0 | 隔离模式 | 隧道网络	 | 172.16.1.20 |
|  || enp1s0 | NAT模式 | 外部网络	 | 192.168.120.20 | 192.168.120.1 |
| instance-vm01 | cinder-1 | enp7s0 | 隔离模式 | 管理网 | 172.16.0.30 |
|  || enp8s0 | NAT模式 | 外部网络	 | 192.168.120.30 | 192.168.120.1|

**网络规划说明：**

控制节点3块网卡，计算节点3块网卡，存储节点2块网卡。特别注意，计算节点和存储节点的最后一块网卡仅用于连接互联网部署Oenstack软件包，如果搭建有本地apt源，这两块网卡是不需要的，不属于openstack架构体系中的网络。

- 管理网络配置为隔离模式(vmware 中是 host-only)，官方解释通过管理网络访问互联网安装软件包，如果搭建的有内部yum源，管理网络是不需要访问互联网的，配置成隔离模式也可以。
- 隧道网络配置为仅主机模式，因为隧道网络不需要访问互联网，仅用来承载openstack内部租户的网络流量。
- 外部网络配置为NAT模式，控制节点的外部网络主要是实现openstack租户网络对外网的访问，另外openstack软件包的部署安装也走这个网络，

特别注意：计算节点和存储节点的外部网络仅用来部署openstack软件包，没有其他用途

**三种网络平面说明：**

- **管理网络（management/API网络）**：
提供系统管理相关功能，用于节点之间各服务组件内部通信以及对数据库服务的访问，所有节点都需要连接到管理网络， 支持Openstack服务之间的API调用和数据同步。 具体参考openstack官方安装指南中的[主机网络](https://docs.openstack.org/zh_CN/install-guide/environment-networking.html)
- **隧道网络（tunnel网络或self-service网络）**：
提供租户虚拟网络的承载网络（VXLAN or GRE）。openstack里面使用gre或者vxlan模式，隧道网络用于虚拟机之间的网络通信，特别是在不同的计算节点上运行的实例之间。另外提供多租户环境下的网络隔离。其特点是在物理网络之上创建虚拟网络(Overlay),不需要对物理网络进行大规模改动。具体参考openstack官方安装指南中的[网络(neutron)概念](https://docs.openstack.org/ocata/zh_CN/install-guide/neutron-concepts.html)
- **外部网络(external网络或者provider网络)** ：
openstack网络至少要包括一个外部网络，这个网络能够访问OpenStack安装环境之外的网络，并且非openstack环境中的设备能够访问openstack外部网络的某个IP。另外外部网络为OpenStack环境中的虚拟机提供浮动IP，实现openstack外部网络对内部虚拟机实例的访问。具体参考openstack官方安装指南中的[提供者网络](https://docs.openstack.org/zh_CN/install-guide/launch-instance-networks-provider.html)

> 注意，这里没有规划存储平面网络，cinder存储节点使用管理网络承载存储网络数据。

<br/>

<img src="https://img1.kiosk007.top/static/images/blog/cloudnative_openstak1.png" />

# 前置准备

- **准备 OpenStack 库**

安装官方文档：[openstack install guide](https://docs.openstack.org/install-guide/)

官网的中文文档只支持非常老旧的 Mitaka ，想要安装最新版本必须看英文文档。

本文安装 2024.1 年的版本 [Caracal openstack release series](https://releases.openstack.org/index.html)

```bash
apt update && apt upgrade
# 所有节点都需要安装
add-apt-repository cloud-archive:caracal
# client installation
apt install python3-openstackclient python3-pip
```

- **写入host解析**

```bash
# vim /etc/hosts
172.16.0.10 controller
172.16.0.20 compute
172.16.0.30 cinder
```

- **打开IP转发功能**

``` bash
# 所有的节点都做如下操作。
sysctl -w net.ipv4.ip_forward=1 
```

- **打开时间同步**

``` bash
# 所有节点
apt install chrony
# controller 节点配置 /etc/chrony/chrony.conf
allow 172.16.0.0/24
# 其余节点
server controller iburst
# 所有节点重启一下 chrony
systemctl restart chrony
# 查看是否同步controller 节点
chronyc sources -v
```

- **安装数据库等基础组件（以下均在 controller 执行）**

```bash
################## 安装 MySQL ##################
apt install mysql-server python3-pymysql
# 一旦安装完成，编辑配置文件 /etc/mysql/mysql.conf.d/mysqld.cnf 
[mysqld]
bind-address = 0.0.0.0

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8mb4_0900_ai_ci
character-set-server = utf8mb4
# 启动MySQL服务并验证。输入
systemctl start mysql
systemctl status mysql

################## 安装 RabbitMQ ##################
apt -y install rabbitmq-server

# 配置文件插入以下一句话, 并且启动
vim /etc/rabbitmq/rabbitmq-env.conf
NODENAME=rabbit@controller
systemctl restart rabbitmq-server

# 添加用户
rabbitmqctl add_user openstack openstack
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

################## 安装mencached ##################
apt -y install memcached python3-memcache
systemctl start memcached

```



<br/>



# 部署 Controller 节点

------------



## 安装 keystone 服务

本节描述如何在控制节点上安装和配置OpenStack身份认证服务，即称为keystone。

Identity 主要有两个功能，一：实现用户的管理、认证和授权。二：服务目录，存储所有可用服务的信息库，包含其API endpoint。

详见：[认识 OpenStack – Keystone](https://kiosk007.top/post/认识-openstack/#keystone)

安装：[Keystone installation Tutorial for Ubuntu](https://docs.openstack.org/keystone/yoga/install/keystone-install-ubuntu.html)



### 安装并初始化

1. **安装 keystone 组件**

```bash
apt install keystone
```

安装额外的 openstack 配置管理工具包，ubuntu里面叫`crudini`，CentOS 中叫做 `openstack-utils`。[OpenStack配置文件的快速修改方法](https://blog.csdn.net/u013469753/article/details/118515276)

```bash
apt install crudini
```



2. **配置MySQL**

因为ubuntu24.04 默认安装的是 mysql8.0，创建用户和授权的方式不太一样，用官网的命令会报错。

```mysql
mysql
> CREATE DATABASE keystone;
> CREATE USER 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
> GRANT ALL PRIVILEGES ON keystone.* to 'keystone'@'%' WITH GRANT OPTION;
```



3. **使用 crudini 修改配置文件 `/etc/keystone/keystone.conf`**

```bash
# MySQL 访问密码填充
crudini --set /etc/keystone/keystone.conf database connection "mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone"
# 配置 Fernet 令牌提供程序
crudini --set /etc/keystone/keystone.conf token provider fernet

```



4. **填充Identity service 数据库 (这一步有一些慢)**

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```



5. **初始化 Fernet 密钥存储库：**

> `--keystone-user` 和 `--keystone-group` 标志用于指定运行 keystone 所使用的操作系统用户/组。提供这些标志是为了允许在另一个操作系统用户/组下运行 keystone。在下面的示例中，我们将用户和组称为 `keystone`。

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```



6. **启动身份认证服务：**

```bash
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```



7. **配置 Apache HTTP 服务**

刚才安装的 keystone 会默认安装一个 Apache 服务器，为Keystone 提供restapi 服务。编辑 `/etc/apache2/apache2.conf` 文件并配置 ServerName 选项以控制 Controller 节点

```bash
ServerName controller
```

再通过 `systemctl restart apache2` 重启服务



8.  **将一些环境变量保存起来**

```bash
vim admin-openrc.sh
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3

vim kiosk-openrc.sh
export OS_USERNAME=kiosk_user
export OS_PASSWORD=KIOSK_USER_PASSWD
export OS_PROJECT_NAME=kiosk007
export OS_USER_DOMAIN_NAME=kiosk
export OS_PROJECT_DOMAIN_NAME=kiosk
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3

```



9. **创建一个 service project 后面会用到**

```bash
openstack project create --domain default \
  --description "Service Project" service
```



- **创建相应的domain、project、user和role**

Identity 服务为每个 Openstack 服务提供身份验证服务，身份验证服务需要 domain、project、user 和 role 配合使用。

**下面的操作仅是演示创建过程**：

> 假设我们要创建一个叫 kiosk 的集团公司（domain），该 domain 下有 kiosk007 和 kiosk404 两个项目部（project）



```bash
# 创建 kiosk domain
openstack domain create --description "Kiosk Domain" kiosk
# 创建2个project，分别是 kiosk404 和 kiosk007
openstack project create --domain kiosk \
  --description "kiosk404 Service Project" kiosk404
openstack project create --domain kiosk \
  --description "kiosk007 Service Project" kiosk007
# 创建一个用户 kiosk_user (这里需要输入一个密码)
openstack user create --domain kiosk \
  --password-prompt kiosk_user
# 创建一个角色 kiosk_role
openstack role create kiosk_role
# 将 kiosk_role 角色添加到 kiosk007项目中和 kiosk_user用户
openstack role add --project kiosk007 --user kiosk_user kiosk_role

```

<br/>

### 验证操作

1. 作为 admin 用户，请求一个身份验证令牌

```bash
# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                         |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-12-28T10:00:38+0000                                                                                                                      |
| id         | gAAAAABnb742A3_c0s9Ks0FgqldtnptdbSV4C70ZK9p4MEmnmRydqYlgZ7ZIxBlBfGOG9lXt9B45necROs4b0b4TNJ2a3x2RT4scY48LmM4FRvI-                              |
|            | wLPejkLq1NhrqXalhRYVi64P-nhic4Zy8fWzmQHlO4uN1lR7pTrskaAWKMVwmdEfunzI6kY                                                                       |
| project_id | e63b23475dbf4f23b650d0d69e4731dc                                                                                                              |
| user_id    | 5406c29ffb89473e9e56c43e9421e9ad                                                                                                              |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
```

2. 作为 kiosk_user ，请求一个身份验证令牌

> 取消设置临时OS_AUTH_URL和OS_PASSWORD环境变量：

```bash
# unset OS_AUTH_URL OS_PASSWORD
# openstack --os-auth-url http://controller:5000/v3   --os-project-domain-name kiosk --os-user-domain-name kiosk   --os-project-name kiosk007 --os-username kiosk_user token issue
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                         |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-12-28T10:04:21+0000                                                                                                                      |
| id         | gAAAAABnb78VUWRWfO4kKUb3BmrLjlgsbrtk8keb-n7JNFB5GC5-3I5m7l5xtq0AVh16y7Fv5tbNAymdc8Ii9shSSsUgK6XNrBQjgv-_4ChphExJ-                             |
|            | TpY8x5UbGUpxj4nMED5zDwPPrDyVf943xqSK7I8Op9-Nq-U23O1Adlp-AnufdhNbkOFVqQ                                                                        |
| project_id | 3cc5e81ccac74010ade75801594c5b39                                                                                                              |
| user_id    | 35570667a6464984a903a5464f323dbb                                                                                                              |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+

```



因为上面已经将用户信息作为环境变量保存了起来，可以通过 source env 的方式切换用户身份。

```bash
# . admin-openrc.sh 
# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                         |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2024-12-28T10:10:07+0000                                                                                                                      |
| id         | gAAAAABnb8BvhBB3fFxk3db0iCU5UyBcBrairhwJQYWeT0gKG5XkqBzl7GgmgJeGMYy6Z0sRm-                                                                    |
|            | yALYGaFFu847AkUZp06jmgbSeFfNuNAy5-QsTas30n-67s7rIraOWdYDOV7_saIGJlpQ-Cz2MeTjvFvczL6OeNzSn3WVJBo4VPsvsbUPYSmMc                                 |
| project_id | e63b23475dbf4f23b650d0d69e4731dc                                                                                                              |
| user_id    | 5406c29ffb89473e9e56c43e9421e9ad                                                                                                              |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
```



<br/>

---------------------------------------------

<br/>



## 安装 Glance 服务

Image Service 在 OpenStack 中注册、发现及获取 VM 的映像文件。VM的映像文件本身存储在对象存储或分布式文件系统中等。

详见：[认识 OpenStack – Glance](https://kiosk007.top/post/认识-openstack/#glance)

安装：[Glance installation Tutorial for Ubuntu](https://docs.openstack.org/glance/yoga/install/install-ubuntu.html)



### 安装并初始化

1. **配置数据库**

```mysql
mysql
> CREATE DATABASE glance;
> CREATE USER 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
> GRANT ALL PRIVILEGES ON glance.* to 'glance'@'%' WITH GRANT OPTION;
```

2. **创建 endpoint 和 user 等**

```bash
. admin-openrc
# 创建 glance 用户
openstack user create --domain default --password-prompt glance
# 将 admin 角色加到 glance 用户和 service 项目
openstack role add --project service --user glance admin
# 创建 glance 服务实体
openstack service create --name glance --description "OpenStack Image" image
# 创建Image service的API服务 endpoint
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

3. **安装 glance**

```bash
sudo apt install glance -y
```

4. **配置 `/etc/glance/glaance-api.conf`**

```bash
# [database] section 中，编辑访问数据库
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

# [keystone_authtoken] 和 [paste_deploy] section 中，编辑认证信息
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS  # 请替换

[paste_deploy]
flavor = keystone

# [glance_stone] section 中，编辑image存储的位置
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

# [oslo_limit] section 中编辑访问 keystone
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = glance
system_scope = all
password = openstack
endpoint_id = http://controller:9292
region_name = RegionOne
```

5. **确保 glance 用户对系统范围都有读权限**

```bash
openstack role add --user glance --user-domain Default --system all reader
```

6. **填充数据库**

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

7. **重启 glance 服务 `systemctl restart glance-api`**

<br/>

### 验证操作

下载一个镜像，再传到 glance 服务上。

```bash
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
# 或者使用 http://s1.kiosk007.top/cirros-0.4.0-x86_64-nocloud.img

$ glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public

```

可以使用`openstack image list或者glance image-list`查看镜像。

- 查看 image：glance image-show IMAGE_ID
- 下载 image：glance image-download –progress –file=./cloud.img IMAGE_ID
- 删除 image：glance image-delelte IMAGE_ID



> 磁盘映像文件可以自己制作，比如 VMBuilder、[VeeWee](https://github.com/jedi4ever/veewee)、imagefactory 等。



<br/>

-------------------------------

<br/>



## 安装 placement 服务

Placement 参与到 nova-scheduler 选择目标主机的调度流程中，负责跟踪记录 Resource Provider 的 Inventory 和 Usage，并使用不同的 Resource Classes 来划分资源类型，使用不同的 Resource Traits 来标记资源特征。

– 其负责跟踪云中可用资源的清单，并协助选择在创建虚拟机时将使用这些资源的哪个提供者。

> Ocata 版本的 Placement API 是一个可选项，建议用户启用并替代 CpuFilter、CoreFilter 和 DiskFilter。Pike 版本则强制要求启动 Placement [API 服务](https://cloud.tencent.com/product/apigw?from=10680)，否则 nova-compute service 无法正常运行



详见：[OpenStack Placement](https://cloud.tencent.com/developer/article/1440332)

官方文档：[OpenStack Compute (nova)](https://docs.openstack.org/nova/yoga/)

安装：[Install and configure Placement for Ubuntu](https://docs.openstack.org/placement/yoga/install/install-ubuntu.html#configure-user-and-endpoints)



### 安装并初始化

1. **配置数据库**

```mysql
mysql
> CREATE DATABASE placement;
> CREATE USER 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
> GRANT ALL PRIVILEGES ON placement.* to 'placement'@'%' WITH GRANT OPTION;
```



2. **创建 User 和 Endpoints**

```bash
. admin-openrc
# 创建 placement user
openstack user create --domain default --password-prompt placement
# 将 placement 用户加入 service project 和 admin role
openstack role add --project service --user placement admin
# 创建 placement API service
openstack service create --name placement \
  --description "Placement API" placement
# 创建 placement endpoints
openstack endpoint create --region RegionOne \
  placement public http://controller:8778
openstack endpoint create --region RegionOne \
  placement internal http://controller:8778
openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
```



3. **安装 placement**

```bash
sudo apt install placement-api -y
```



4. **编辑 `/etc/placement/placement.conf` 文件**

```bash
# [placement_database] section 中，编辑访问数据库
[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

# [keystone_authtoken] 和 [api] section 中，编辑认证信息
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS  # 请替换
```



5.**填充数据库**

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```





6. **重启 apache2**

```bash
systemctl restart apache2
```



### 验证操作

验证安置服务的运行情况。

```bash
$ placement-status upgrade check
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Missing Root Provider IDs          |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Incomplete Consumers               |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
```

执行 `pip3 install osc-placement` 安装，可以列出资源类和特征。

> 从 Ubuntu 22.04 开始，系统默认启用了 PEP 668 标准（[PEP 668](https://peps.python.org/pep-0668/)），该标准将系统的 Python 环境标记为 "externally managed"（外部管理）
>
> ubuntu 24.04 应该做如下操作：
>
> `sudo apt install pipx`
>
> `pipx install osc-placement`

```bash
$ openstack --os-placement-api-version 1.2 resource class list --sort-column name
+----------------------------------------+
| name                                   |
+----------------------------------------+
| DISK_GB                                |
| FPGA                                   |
| IPV4_ADDRESS                           |
| MEMORY_MB                              |
| MEM_ENCRYPTION_CONTEXT                 |
| NET_BW_EGR_KILOBIT_PER_SEC             |
| NET_BW_IGR_KILOBIT_PER_SEC             |
...
```



<br/>

-------------------------------

<br/>

## 安装 nova 服务

Compute Service ，主要提供计算服务，但本身Nova 又分为了计算节点和控制节点，不是那么简单的。

使用OpenStack Compute来托管和管理云计算系统。OpenStack Compute是基础架构即服务（IaaS）系统的重要组成部分。主要模块是用Python实现的。

OpenStack Compute与OpenStack Identity 进行交互以进行身份验证; 用于磁盘和服务器映像的OpenStack映像服务; 和用于用户和管理界面的OpenStack Dashboard。镜像访问受到项目和用户的限制; 每个项目的限额是有限的（例如，实例的数量）。OpenStack Compute可以在标准硬件上水平扩展，并下载映像以启动实例。
<br/>

Nova有很多组件组成，一些组件随着 Openstack 的更新拆出去了

组件：

- API： [nova-api](https://docs.openstack.org/nova/yoga/cli/nova-api.html)、[nova-api-metadata](https://docs.openstack.org/nova/yoga/cli/nova-api-metadata.html)
- Compute Core：[nova-compute](https://docs.openstack.org/nova/yoga/cli/nova-compute.html)、[nova-scheduler](https://docs.openstack.org/nova/yoga/cli/nova-scheduler.html)、[nova-conductor](https://docs.openstack.org/nova/yoga/cli/nova-conductor.html)、[nova-api-os-compute](https://docs.openstack.org/nova/yoga/cli/nova-api-os-compute.html)
- Network for VMs：nova-network、nova-dhcpagent (已经转为 Neutron 服务)
- Console Interface：nova-consoleauth、[nova-novncproxy](https://docs.openstack.org/nova/yoga/cli/nova-novncproxy.html)、nova-xvpnvnxproxy、nova-cert
- Command line and Other interfaces：nova、nova-manage

详见：[认识 OpenStack – Nova](https://kiosk007.top/post/认识-openstack/#nova)

安装：[Install and configure controller node for Ubuntu](https://docs.openstack.org/nova/yoga/install/controller-install-ubuntu.html)

<br/>



> **以下操作均在 控制器节点**

### 安装并初始化

1. **配置数据库**

```mysql
mysql
> CREATE DATABASE nova_api;
> CREATE DATABASE nova;
> CREATE DATABASE nova_cell0;

> CREATE USER 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova_api.* to 'nova'@'%' WITH GRANT OPTION;
> GRANT ALL PRIVILEGES ON nova.* to 'nova'@'%' WITH GRANT OPTION;
> GRANT ALL PRIVILEGES ON nova_cell0.* to 'nova'@'%' WITH GRANT OPTION;

```



2. **创建 endpoint、service 和 user 等**

```bash
# 创建 nova 用户
openstack user create --domain default --password-prompt nova
# 将 admin role 加载到 nova 用户上
openstack role add --project service --user nova admin
# 创建 nova 用到的 service
openstack service create --name nova \
  --description "OpenStack Compute" compute
# 创建计算节点的 endpoint
openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

```



3. **安装 Nova**

```bash
apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```



4. **编辑 `/etc/nova/nova.conf`  文件并完成以下操作**

```bash
# [api_database] 和 [database] 配置访问数据库
[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

# [DEFAULT] 中配置 RabbitMQ消息队列访问
[DEFAULT]
transport_url = rabbit://openstack:openstack@controller:5672/

# 在 [api] 和 [keystone_authtoken] 配置身份服务访问
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS  # 请替换

[service_user]
send_service_user_token = true
auth_url = http://controller:5000/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova 
password = NOVA_PASS # 请替换

# [DEFAULT] 配置 my_ip 控制节点的IP
[DEFAULT]
my_ip = 172.16.0.10

# [vnc] 中配置vnc的地址
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

# [glance] 中配置image service api
[glance]
api_servers = http://controller:9292

# [oslo_concurrency] 中配置锁路径
[oslo_concurrency]
lock_path = /var/lib/nova/tmp

# [placement] 中配置对 Placement 服务的访问
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS  # 请替换
```



5. **填充数据库**

```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

这步在执行的时候会出现如下报错

``` bash
su -s /bin/sh -c "nova-manage api_db sync" nova
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
```

可以执行 `export OS_NOVA_DISABLE_EVENTLET_PATCHING=true` 消除报错

```bash
# 注册 cell0 数据库
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
# 创建 cell1 cell
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
# 填充 nova 数据库
su -s /bin/sh -c "nova-manage db sync" nova

```



6. **重启服务**

```bash
systemctl restart nova-api
systemctl restart nova-scheduler
systemctl restart nova-conductor
systemctl restart nova-novncproxy
```





### 验证操作

```bash
# 验证 nova cell0 和 cell 是否注册正确
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |              Transport URL               |               Database Connection               | Disabled |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                  none:/                  | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | e3889de5-0fd7-4c2d-9dfb-b670001968d8 | rabbit://openstack:****@controller:5672/ |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
```





<br/>

---------------------------

<br/>



## 安装 Neutron 服务

- Controller 节点需要部署 neutron-server
- Compute 节点需要部署 neutron-*(plugin)-agent
- Network 节点需要部署 neutron-*(plugin)-agent、neutron-l3-agent、neutron-dhcp-agent

> neutron-*(plugin)-agent，如 neutron-linuxbridge-agent

Openstack 中的物理网络连接架构。

管理网络（management network）enp7s0 172.16.0.0/24

数据网络（management network）enp8s0 172.16.1.0/24

外部网络（external network）enp1s0   (Nat)

- Tenant network：tenant 内部使用的网络;
  - Flat：所有 VM 在同一个网络中，不支持 VLAN及其他网络隔离机制
  - Local：所有 VM 位于本地 Compute 节点，且不与 External network 通信。
  - VLAN：通过使用 VLAN IDs 创建多个 providers 或 tenant 网络。
- Provider network：不专属于某个 tenant，为各 tenant 提供通信承载的网络；

> 详见 [OpenStack Networking](https://docs.openstack.org/neutron/yoga/admin/intro-os-networking.html)

详见：[认识 OpenStack – Neutron](https://kiosk007.top/post/认识-openstack/#neutron) \ [Neutron’s documentation](https://docs.openstack.org/neutron/yoga/)

安装：[Install and configure for Ubuntu](https://docs.openstack.org/neutron/yoga/install/install-ubuntu.html)



### 安装并初始化

1. **创建数据库**

```mysql
mysql
> CREATE DATABASE neutron;
> CREATE USER 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
> GRANT ALL PRIVILEGES ON neutron.* to 'neutron'@'%' WITH GRANT OPTION;

```



2. **创建 user、service、endpoint**

```bash
# 创建neutron 用户
openstack user create --domain default --password-prompt neutron
# 绑定
openstack role add --project service --user neutron admin
# 创建 service
openstack service create --name neutron \
  --description "OpenStack Networking" network
# 创建endpoint
openstack endpoint create --region RegionOne \
  network public http://controller:9696
openstack endpoint create --region RegionOne \
  network internal http://controller:9696
openstack endpoint create --region RegionOne \
  network admin http://controller:9696

```



2. **安装 Neutron 网络**

您可以使用选项1和2所代表的两种体系结构之一来部署网络服务。

选项1部署了仅支持将实例附加到提供者（外部）网络的最简单的可能架构。 没有自助服务（专用）网络，路由器或浮动IP地址。
只有管理员或其他特权用户才能管理提供商网络。

选项2增加了选项1，其中支持将实例附加到自助服务网络的第3层服务。

演示或其他非特权用户可以管理自助服务网络，包括提供自助服务和提供商网络之间连接的路由器。此外，浮动IP地址可提供与使用来自外部网络（如Internet）的自助服务网络的实例的连接。 自助服务网络通常使用隧道网络。隧道网络协议（如VXLAN），选项2还支持将实例附加到提供商网络。
以下两项配置二选一：

- [x] Networking Option 1: Provider networks
- [x] Networking Option 2: Self-service networks
  

这里创建自服务网络 [Networking Option 2: Self-service networks](https://docs.openstack.org/neutron/yoga/install/controller-install-option2-ubuntu.html)

```bash
apt install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent
```





3. **编辑配置文件 /etc/neutorn/neutron.conf**

```bash
# [database] 配置访问数据库
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

# [DEFAULT] 配置 2层网络（ML2）插件，router service 和 overlapping IP
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

# [DEFAULT] 配置 rabbitmq
transport_url = rabbit://openstack:openstack@controller

# [DEFAULT] 和 [keystone_authtoken] 配置 keystone
auth_strategy = keystone

[experimental]
linuxbridge = true

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS   # 请修改

# [DEFAULT]和[nova]配置Networking以通知Compute网络拓扑更改
[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS # 请修改

# [oslo_concurrency]部分中配置锁路径
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

```



4. **配置网络二层插件 `/etc/neutron/plugins/ml2/ml2_conf.ini`**

```bash
# [ml2] 启用 flat,VLAN和VXLAN
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan # 启用VXLAN自助服务网络
mechanism_drivers = linuxbridge,l2population # 启用Linux桥接和2层填充
extension_drivers = port_security # 启用端口安全扩展

# [ml2_type_flat]
[ml2_type_flat]
flat_networks = provider

# [ml2_type_vxlan] VXLAN网络标识符范围
[ml2_type_vxlan]
vni_ranges = 1:1000

# [securitygroup] 启用ipset提供安全组规则效率
[securitygroup]
enable_ipset = true
```



5. **配置linux网桥代理 `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`**

Linux桥接代理为实例构建层-2（桥接和交换）虚拟网络基础结构，并处理安全组。

> 在 [linux_bridge]部分, 将提供者虚拟网络映射到提供者物理网络接口，这里的网卡填的是 外部网络的网卡（underlying provider physical network interface）
>
> 在[vxlan]部分中，启用vxlan隧道网络，配置处理隧道网络的物理网络接口的IP地址

```bash
[linux_bridge]
physical_interface_mappings = provider:enp1s0

[vxlan]
enable_vxlan = true
local_ip = 172.16.1.10
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

为了启用网络桥接支持，需要加载 **br_netfilter** 内核模块，使用命令 `modprobe br_netfilter` 加载。执行成功后，验证是否加载成功。

```bash
$ modprobe br_netfilter
$ sysctl -a |grep bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-filter-pppoe-tagged = 0
net.bridge.bridge-nf-filter-vlan-tagged = 0
net.bridge.bridge-nf-pass-vlan-input-dev = 0

```



6. **配置三层代理 /etc/neutron/l3_agent.ini**

Layer-3（L3）代理为自助虚拟网络提供路由和NAT服务。

```bash
[DEFAULT]
interface_driver = linuxbridge
```



7. **配置 DHCP 服务 /etc/neutron/dhcp_agent.ini**

DHCP代理为虚拟网络提供DHCP服务。

``` bash
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```



8. **配置元数据服务 /etc/neutron/metadata_agent.ini**

元数据代理为实例提供配置信息，例如凭据。

```bash
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```



9. **配置计算服务使用网络服务 /etc/nova/nova.conf**

```bash
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = redhat
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET  # 和上面的 metadata_secret 保持一致即可
```



10. **同步neutron数据库**

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

```



11. **重启服务**

```bash
service nova-api restart
service neutron-server restart
service neutron-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
service neutron-l3-agent restart

```



### 验证操作





<br/>

_______

<br/>



## 安装 Horizon 服务

本节介绍如何在控制节点上安装和配置仪表板。
仪表板所需的唯一核心服务是身份服务。 您可以将仪表板与其他服务结合使用，例如镜像服务，计算和网络。 还可以在具有独立服务（如对象存储）的环境中使用仪表板。

安装：[Install and configure for Ubuntu](https://docs.openstack.org/horizon/yoga/install/install-ubuntu.html)



### 安装并修改配置

1. **安装 dashboard 可视化界面**

```bash
apt install openstack-dashboard
```



2. **编辑文件 /etc/openstack-dashboard/local_settings.py**

配置仪表板以在controller节点上使用OpenStack服务 ：

```python
OPENSTACK_HOST = "controller"
```

允许您的主机访问仪表板：

```python
ALLOWED_HOSTS = ['*']
```

或者ALLOWED_HOSTS = [‘one.example.com’, ‘two.example.com’]
ALLOWED_HOSTS也可以[‘*’]接受所有主机。这对开发工作可能有用，但可能不安全，不应用于生产。

配置memcache会话存储服务：
```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
         'LOCATION': '127.0.0.1:11211',
    }
}
```



开启身份认证API 版本v3

```python
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
```

启用对域的支持：

```python
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```

配置API版本:

```python
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
```

配置Default为您通过仪表板创建的用户的默认域：

```python
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```

将用户配置为通过仪表板创建的用户的默认角色：

```python
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```

配置时区（可选）

```python
TIME_ZONE = "Asia/Shanghai"
```



<br/>

3. **添加以下内容 `/etc/apache2/conf-available/openstack-dashboard.conf`**

```bash
WSGIApplicationGroup %{GLOBAL}
```

并且重启 apache

```bash
systemctl restart apache2
```

<br/>

--------------------

<br/>



# 部署 Cinder 节点

## 安装 Cinder 服务

块存储服务（cinder）为访客实例提供块存储设备。 存储配置和使用的方法由块存储驱动程序确定，或者在多后端配置的情况下由驱动程序确定。 有多种可用的驱动程序：NAS / SAN，NFS，iSCSI，Ceph等。
 块存储API和调度程序服务通常在控制节点上运行。 根据所使用的驱动程序，卷服务可以在控制节点，计算节点或独立存储节点上运行。

其提供 Block Storage Service。其由 cinder-api、cinder-volume、cinder-scheduler 组成。

详见：[认识 OpenStack – Cinder](https://kiosk007.top/post/认识-openstack/#cinder)

安装：[Install and configure for Ubuntu](https://docs.openstack.org/cinder/yoga/install/cinder-controller-install-ubuntu.html)

### Controller 节点安装服务

安装：[Install and configure controller node](https://docs.openstack.org/cinder/yoga/install/cinder-controller-install-ubuntu.html)



1. **配置MySQL数据库**

```mysql
mysql
> CREATE DATABASE cinder;
> CREATE USER 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
> GRANT ALL PRIVILEGES ON cinder.* to 'cinder'@'%' WITH GRANT OPTION;

```



2. **创建相应的 user、service、endpoint**

```bash
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
# 创建 serivce
openstack service create --name cinderv3 \
  --description "OpenStack Block Storage" volumev3
# 创建 endpoint  
openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
```



3. **安装cinder 服务**

```bash
apt install cinder-api cinder-scheduler
```

编辑 `/etc/cinder/cinder.conf`

```bash
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = lioadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
enabled_backends = lvm
transport_url = rabbit://openstack:openstack@controller
my_ip = 172.16.0.10

[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASSWD

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```



4. **同步DB**

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```



5. **编辑 /etc/nova/nova.conf**

```bash
[cinder]
os_region_name = RegionOne
```



6. **重启服务**

```bash
systemctl restart nova-api
systemctl restart cinder-scheduler
systemctl restart apache2
```





### Cinder 节点安装服务

安装：[Install and configure a storage node](https://docs.openstack.org/cinder/yoga/install/cinder-storage-install-ubuntu.html)

### 创建设备卷LVM

1. **安装相关的工具包**

```bash
apt install lvm2 thin-provisioning-tools
apt install cinder-backup
```

2. **配置LVM**

这块需要添加一块新的磁盘 /dev/vdb , 这里我们使用 libvritd 可以直接添加一块新硬盘

```bash
# 查看当前的磁盘 
$ fdisk -l /dev/vdb
Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

# 创建 PV
pvcreate /dev/vdb

# 创建 VG
vgcreate cinder-volumes /dev/vdb
  Physical volume "/dev/vdb" successfully created.
  Volume group "cinder-volumes" successfully created

# 编辑 /etc/lvm/lvm.conf ，主要是为了加强安全性，如果不指定，所有的都可以访问，比如下面的 /dev/vdb 可以访问，其他全部拒绝
filter = [ "a/vdb/", "r/.*/"]
```



3. **安装 `cinder-volume`**

```
apt install cinder-volume tgt 
```

配置 `/etc/cinder/cinder.conf`

```bash
[DEFAULT]
my_ip = 172.10.0.30
enabled_backends = lvm
glance_api_servers = http://controller:9292


[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```



4. **重启服务**

```bash
# service tgt restart
# service cinder-volume restart
```



### 验证

```bash
$ . admin-openrc.sh
$ cinder create --display-name testVloume 2
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | false                                |
| cluster_name                   | None                                 |
| consistencygroup_id            | None                                 |
| consumes_quota                 | True                                 |
| created_at                     | 2024-12-29T04:50:00.000000           |
| description                    | None                                 ...

$ cinder list
+--------------------------------------+-----------+------------+------+----------------+-------------+----------+-------------+
| ID                                   | Status    | Name       | Size | Consumes Quota | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+------------+------+----------------+-------------+----------+-------------+
| 3a747ef8-5a85-4cdf-a2d0-528bfa28c828 | available | testVloume | 2    | True           | __DEFAULT__ | false    |             |
+--------------------------------------+-----------+------------+------+----------------+-------------+----------+-------------+

```





<br/>

------------------

<br/>



# 部署 compute 节点

安装：[Install and configure a compute node for Ubuntu](https://docs.openstack.org/nova/yoga/install/compute-install-ubuntu.html)

## 安装 Nova Agent

1. **安装 nova-compute**

```bash
apt install nova-compute
```

2. **修改配置文件 /etc/nova/nova.conf**

```bash
# [DEFAULT] 配置 RabbitMQ 消息队列
[DEFAULT]
transport_url = rabbit://openstack:openstack@controller

# [api] 和 [keystone_authtoken] 中编辑认证信息
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS   # 请替换
 
 
# [DEFAULT] 中替换为计算节点的IP管理网卡的IP
my_ip = 172.16.0.20

# [vnc] 部分启用和配置远程控制台访问
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

# [glance] 配置 image service API位置
[glance]
api_servers = http://controller:9292

# [oslo_concurrency] 配置锁路径
[oslo_concurrency]
lock_path = /var/lib/nova/tmp

#[placement] API
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS  # 请替换

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

3. **[可选] 修改配置文件 `/etc/nova/nova-compute.conf`**

执行 `egrep -c '(vmx|svm)' /proc/cpuinfo` 可以验证计算节点是否支持虚拟机的硬件加速，如果不支持则只能使用 qemu 这种完全虚拟化技术，否则可以使用 kvm 来加速CPU 和 内存相关的访问。



4. **重启服务**

```bash
systemctl restart nova-compute
```



### 将计算节点添加到 cell 数据库

> 以下操作切换到 Controller 节点操作

1. **首先确认数据库中有计算节点**

```bash
$ openstack compute service list --service nova-compute
+--------------------------------------+--------------+---------------+------+---------+-------+----------------------------+
| ID                                   | Binary       | Host          | Zone | Status  | State | Updated At                 |
+--------------------------------------+--------------+---------------+------+---------+-------+----------------------------+
| e319ce93-4e7c-44be-afe8-4b49a6289df3 | nova-compute | instance-vm01 | nova | enabled | up    | 2024-12-28T15:10:29.000000 |
+--------------------------------------+--------------+---------------+------+---------+-------+----------------------------+
```

2. **发现该节点**

```bash
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```



<br/>

### 验证操作

1. 列出服务组件，验证启动和注册成功

```bash
$ openstack compute service list
+--------------------------------------+----------------+---------------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host          | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+---------------+----------+---------+-------+----------------------------+
| 9b4f12e1-5b81-4d94-b0cc-988d91dab887 | nova-scheduler | instance-vm00 | internal | enabled | up    | 2024-12-28T15:13:26.000000 |
| 5b43ace8-942c-4159-b51f-86819c15c6c8 | nova-conductor | instance-vm00 | internal | enabled | up    | 2024-12-28T15:13:26.000000 |
| e319ce93-4e7c-44be-afe8-4b49a6289df3 | nova-compute   | instance-vm01 | nova     | enabled | up    | 2024-12-28T15:13:29.000000 |
+--------------------------------------+----------------+---------------+----------+---------+-------+----------------------------+

```

2. 列出身份服务

```bash
$ openstack catalog list
+-----------+-----------+------------------------------------------------------------------------+
| Name      | Type      | Endpoints                                                              |
+-----------+-----------+------------------------------------------------------------------------+
| glance    | image     | RegionOne                                                              |
|           |           |   public: http://controller:9292                                       |
|           |           | RegionOne                                                              |
|           |           |   admin: http://controller:9292                                        |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:9292                                     |
|           |           |                                                                        |
...
```

3. 检查 cell 和 placement API 是否运行成功

```bash
nova-status upgrade check
```

<br/>

------

<br/>

## 安装 Neutron Agent

安装：[Install and configure compute node](https://docs.openstack.org/neutron/yoga/install/compute-install-ubuntu.html)

1. **安装 linuxbridge-agent**

```bash
apt install neutron-linuxbridge-agent
```

2. **编辑 /etc/neutron/neutron.conf**

```bash
[DEFAULT]
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS   # 请替换

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```



### 选择自服务的配置

参考：[Networking Option 2: Self-service networks](https://docs.openstack.org/neutron/yoga/install/compute-install-option2-ubuntu.html)

1. **配置 Linux Bridge Agent 大二层 /etc/neutron/plugins/ml2/linuxbridge_agent.ini **

``` bash
[linux_bridge]
physical_interface_mappings = provider:enp1s0  # 外网网卡

[vxlan]
enable_vxlan = true
local_ip = 172.16.1.20     # 这里是隧道网络的接口
l2_population = true

# 启用 Linux 网桥 iptables 防火墙驱动程序
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

2. **配置 Nova /etc/nova/nova.conf**

```bash
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

3. **确保Linux操作系统内核支持网桥**

```bash
sysctl -a |grep bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

4. **重启服务**

```bash
systemctl restart nova-compute 
systemctl restart neutron-linuxbridge-agent

```

### 验证操作

以下操作在控制节点上

```bash
openstack extension list --network
openstack network agent list
```

<br/>

-------

<br/>



## 安装完毕

在浏览器上输入 `http://172.16.0.10/horizon` 可以访问 Openstack 的控制台。

用户：admin

密码：ADMIN_PASSWD

域：Default

![openstack_web](https://img1.kiosk007.top/static/images/blog/20241229134527-openstack_web.png)
