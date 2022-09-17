---
title: 【译】OpenStack 虚拟机创建的请求流程
author: kiosk
tags: 
  - openstack
categories:
  - cloud_computing
date: 2022-09-14 11:23:00
---

翻译自原文： [Request Flow for Provisioning Instance in Openstack](https://ilearnstack.com/2013/04/26/request-flow-for-provisioning-instance-in-openstack/)

不过该网站已经无法访问了，可能是作者已经不维护了。有点可惜。

<br/>

<!--more-->

任何云中最重要的用例之一就是配置虚拟机。在本文中，我们将介绍一个基于Openstack云的实例（VM）。本文讨论Openstack下各种项目的请求流程和组件交互。最终结果将启动一个虚拟机。

<br/>

<img src="https://img1.kiosk007.top/static/images/blog/request-flow.png" alt="request-flow" style="zoom:150%;" />

<br/>

虚拟机创建过程涉及到了 OpenStack 中各个子模块之间的交互：

- **CLI命令行解释器**：用于向 OpenStack Compute 提交命令。

- **Dashboard Horizon**：为 OpenStack 服务提供一个人机交互界面
- **计算 Nova**：从 Glance 中获取虚拟机镜像（image），依附主机模板（flavor）和相关元数据（metadata）和转换用户的API请求到正在运行的虚拟机当中。
- **网络 Neutron**：为计算节点提供虚拟化网络服务，允许用户创建自己的网络并将它们跟虚拟机关联起来
- **块存储 Cinder**：为计算节点提供持久存储服务
- **虚拟机镜像 Glance**：存储虚拟机镜像
- **身份验证 Keystone**：为 OpenStack 所有服务提供身份验证和授权服务
- **消息队列 RabbitMQ**：处理 OpenStack 各模块（如 Nova、Neutron、Cinder）之间的内部通信

<br/>

创建一个虚拟机的请求流程大概是这样子的：

<br/>

1. Dashboard 或者 CLI 接收到了用户的身份验证信息，并将这些信息以 REST 调用的方式发送到 KeyStone
2. KeyStone 对信息进行验证，同时生成和返回授权 Token，这个 Token 将会用于向 OpenStack 中其他模块以 REST 调用方式发送请求
3. Dashboard 或 CLI 将创建虚拟机的类似 “Launch Instance” 或者 “nova-boot” 的请求转化为 REST API，并将其他模块以 REST 调用方式发送请求
4. nova-api 接收到请求，并向 KeyStone 发送验证 Token 和访问权限的请求
5. KeyStone 验证 Token，并返回更新的包含角色（role）和权限的确认
6. nova-api 与数据库 nova-database 进行交互
7. 在数据库中为虚拟机创建一条记录
8. nova-api 向调度器 nova-scheduler 发送一个 RPC 请求，期望获得更新的包含主机ID的虚拟机入口 （entry）
9. nova-scheduler 从消息队列中提取请求
10. nova-scheduler 与 nova-database 进行交互，通过过滤条件和权重，找到一个可用的主机 ID
11. nova-scheduler 返回更新后的包含主机ID 的虚拟机入口
12. nova-scheduler 向 nova-compute 发送 RPC 请求，以在可用的计算节点上启用虚拟机
13. nova-compute 从消息队列中提取请求
14. nova-compute 向 nova-conductor 发送 RPC 请求，以获取虚拟机信息，例如主机 ID、主机模板（Ram、CPU、磁盘）
15. nova-conductor 从消息队列中提取消息
16. nova-conductor 与  nova-database 进行交互
17. 返回虚拟机信息
18. nova-compute 从消息队列中提取虚拟机信息
19. nova-compute 通过传递 token 向 glance-api 发送 REST 请求，以通过镜像 ID 从 Glance 中获得镜像 URL 和从镜像存储池中上传镜像
20. glance-api 向 keyStone 验证 Token
21. nova-compute 获得镜像元信息（metadata）
22. nova-compute 通过传递 token 向 Network 发送 REST API 请求分配和配置网络资源，为虚拟机分配 IP
23. neutron-server 向 KeyStone 验证 Token
24. nova-compute 获得网络信息
25. nova-cimpute 通过传递 Token 向 Volume 发送 REST API 请求为虚拟机分配存储资源（块存储）
26. cinder-api 向 KeyStone 验证 Token
27. nova-compute 获得块存储信息
28. nova-compute 为 监控器（hypervisor）驱动生成数据并在其上执行请求 （通过 Libvirt）

<br/>

下表展示了虚拟机创建过程中在不同步骤下的状态：

| **Status** | **Task**             | **Power state** | **Steps** |
| :--------- | :------------------- | :-------------- | :-------- |
| Build      | scheduling           | None            | 3-12      |
| Build      | networking           | None            | 22-24     |
| Build      | block_device_mapping | None            | 25-27     |
| Build      | spawing              | None            | 28        |
| Active     | none                 | Running         |           |

<br/>

https://github.com/openstack/nova/blob/master/nova/compute/manager.py 的大约 2449 行可以看到 虚拟机创建流程。（第 19 步之后）

```python
def _build_and_run_instance:
    ...
    with self._build_resources(context, instance,
         requested_networks, security_groups, image_meta,
         block_device_mapping, provider_mapping,
         accel_uuids) as resources:
    instance.vm_state = vm_states.BUILDING
    instance.task_state = task_states.SPAWNING
    # NOTE(JoshNang) This also saves the changes to the
    # instance from _allocate_network_async, as they aren't
    # saved in that function to prevent races.
    instance.save(expected_task_state=
             task_states.BLOCK_DEVICE_MAPPING)
    block_device_info = resources['block_device_info']
    network_info = resources['network_info']
    accel_info = resources['accel_info']
    LOG.debug('Start spawning the instance on the hypervisor.',instance=instance)
```

<br/>

以下函数在 *self._build_resources* 中

1. 创建网络

```python
LOG.debug('Start building networks asynchronously for instance.',
                      instance=instance)
network_info = self._build_networks_for_instance(context, instance,
                    requested_networks, security_groups,
                    resource_provider_mapping, network_arqs)
resources['network_info'] = network_info
```

<br/>

2. 创建块设备

```python
# Verify that all the BDMs have a device_name set and assign a
# default to the ones missing it with the help of the driver.
self._default_block_device_names(instance, image_meta,
                                             block_device_mapping)

LOG.debug('Start building block device mappings for instance.',
                      instance=instance)
instance.vm_state = vm_states.BUILDING
instance.task_state = task_states.BLOCK_DEVICE_MAPPING
instance.save()

block_device_info = self._prep_block_device(context, instance,
                    block_device_mapping)
resources['block_device_info'] = block_device_info
```

<br/>

3. 创建 image，然后创建domain

```python
self.driver.spawn(context, instance, image_meta,
                  injected_files, admin_password,
                  allocs, network_info=network_info,
 				  block_device_info=block_device_info,
                  accel_info=accel_info)
LOG.info('Took %0.2f seconds to spawn the instance on '
                             'the hypervisor.', timer.elapsed(),
                             instance=instance)
```

以下在 *nova/virt/libvirt/driver.py* ， 是 libvirt 的创建虚拟机的实现、除 libvirt 以外，还有 vmwareapi 等。其内部实现是借用了 [libvirt python sdk](https://libvirt.org/python.html) 操纵libvirt。首先生成对应的 domain XML 文件，再将 XML 文件传递给 SDK。

```python
    def spawn(self, context, instance, image_meta, injected_files,
              admin_password, allocations, network_info=None,
              block_device_info=None, power_on=True, accel_info=None):
        disk_info = blockinfo.get_disk_info(CONF.libvirt.virt_type,
                                            instance,
                                            image_meta,
                                            block_device_info)
        injection_info = InjectionInfo(network_info=network_info,
                                       files=injected_files,
                                       admin_pass=admin_password)
        gen_confdrive = functools.partial(self._create_configdrive,
                                          context, instance,
                                          injection_info)
        created_instance_dir, created_disks = self._create_image(
                context, instance, disk_info['mapping'],
                injection_info=injection_info,
                block_device_info=block_device_info)

        # Required by Quobyte CI
        self._ensure_console_log_for_instance(instance)

        # Does the guest need to be assigned some vGPU mediated devices ?
        mdevs = self._allocate_mdevs(allocations)

        # If the guest needs a vTPM, _get_guest_xml needs its secret to exist
        # and its uuid to be registered in the instance prior to _get_guest_xml
        if CONF.libvirt.swtpm_enabled and hardware.get_vtpm_constraint(
            instance.flavor, image_meta
        ):
            if not instance.system_metadata.get('vtpm_secret_uuid'):
                # Create the secret via the key manager service so that we have
                # it to hand when generating the XML. This is slightly wasteful
                # as we'll perform a redundant key manager API call later when
                # we create the domain but the alternative is an ugly mess
                crypto.ensure_vtpm_secret(context, instance)

        xml = self._get_guest_xml(context, instance, network_info,
                                  disk_info, image_meta,
                                  block_device_info=block_device_info,
                                  mdevs=mdevs, accel_info=accel_info)
        self._create_guest_with_network(
            context, xml, instance, network_info, block_device_info,
            post_xml_callback=gen_confdrive,
            power_on=power_on,
            cleanup_instance_dir=created_instance_dir,
            cleanup_instance_disks=created_disks)
        LOG.debug("Guest created on hypervisor", instance=instance)

```

