---

layout: post
title: "openstack Bare Metal Service (ironic) 介绍"
keywords : ["openstack","ironic","bare metal"]
description : "openstack 裸机服务 ——ironic基础介绍"
category : "分享"
author : "黎斌"
tags : ["openstack","ironic","bare metal"]

---

{% include JB/setup %}

# openstack Bare Metal Service (ironic) 介绍

## 目录

1. 简介
    - ironic简介
    - ironic的前世今生
    - ironic提供的功能
    - ironic架构
        - 项目组成
        - 与其他组件的关系
        - 数据流关系
        - 部署架构
2. 剖析ironic
    - 相关技术
        - PXE
        - IPMI
        - iSCSI
    - ironic
        - 代码结构
        - API、Conductor、DB
        - Driver
    - Ironic-Python-Agent
        - 与Condutor通信机制
        - 多硬件支持

3. 安装与配置

4. 理解裸机部署
   - 裸机部署过程
   - PXE部署过程
   - Ironic 部署过程案例
       - Ironic pex部署过程
       - Ironic agent部署过程
 
5. 总结





## 简介

![ironic-logo]({{IMAGE_PATH}}/binli/ironic-presentation/Ironic_mascot_color.png)

### ironic简介

如今Openstack在虚拟化管理部分已经很成熟了， 通过nova我们可以创建虚拟机、枚举虚拟设备、管理电源状态、安装操作系统等。但是有时候虚拟机无法满足要求，比如以下几种情况需要直接使用物理机：

- 高性能的计算集群
- 计算任务需要访问无法虚拟化的硬件设备
- 数据库主机（有些数据库在hypervisor中运行效率很差）
- 单租户、专用硬件、安全性、可靠性和其他控制要求
- 快速部署云基础设施

但是在物理机管理上一直没有成熟的解决方案。在这样的背景下Ironic（Bare-Metal Provisioning）诞生了，它可以解决物理机的添加，删除，电源管理和安装部署。Ironic提供了一系列常用的驱动，同时提供了插件的机制让厂商可以开发自己的driver，这让它支持几乎所有的硬件。

![Ironic_bear_arch]({{IMAGE_PATH}}/binli/ironic-presentation/Ironic_bear_arch.png)


部署物理机跟部署虚拟机的概念在nova来看是一样，都是nova通过创建虚拟机的方式来触发，只是底层nova-scheduler和nova-compute的驱动不一样。虚拟机的底层驱动采用的libvirt的虚拟化技术，而物理机是采用Ironic技术，ironic可以看成一组Hypervisor API的集合，其功能与libvirt类似。


### 前世今生

最早baremetal的概念出现在nova里，物理机和虚拟机管理有很多地方非常相似，比如物理机和虚拟机都需要开机关机，安装部署，添加和删除，为了避免重复造轮子，他们在nova中实现了一个物理机的driver，这样把物理机管理做为计算资源管理的一个子集了。后来发现这样做有问题：

- 早期baremetal作为一个driver有自己的数据库，同一个项目中有两套数据库不合适。
- 在部署和管理baremetal的过程中有很多需要存储的信息是和部署管理虚拟机是不同的，通过nova api来获取这些信息比较尴尬，把baremetal剥离出来有助于划清baremetal和虚拟机部署的界限。
- 有时候baremetal需要做一些比较特殊的行为，比如discovery, hardware raid configuration, firmware updates, burn-in这些操作，它们不适合放在nova里面。比较好的办法是当完成这些操作的时候，向nova去注册信息，作为nova中的可用的资源，最后通过nova boot去调用这些资源。 

经过很多次讨论，开始社区把bare metal分离出来了, 命名为Ironic，从Icehouse版开始进入孵化项目，并在Juno版与Nova进行集成，从完成了项目毕业评审，在Kilo版正式的集成到openstack项目中来，今后会通过nova调用Ironic的api来实现对物理机资源的管理和控制。

传统的hypervisor一般包括创建虚拟机、枚举虚拟设备、管理电源、加载操作系统等功能，与之对应，Ironic可以看成结合多个驱动提供一套hypervisor API来操作物理机提供类似操作，所以ironic可以看成一个hypervisor驱动来给Nova来用。



## 架构


### 项目组成

- ironic： 包含ironic-api 和ironic-conductor进程
- python-ironicclinet: python clinet and CLI
- ironic-python-agent: 一个运行在deployment ramdisk中的Python程序，用于执行一系列部署动作
- pyghmi: 一个python的IPMI库，可以代替IPMItool
- ironic-inspector: 硬件自检工具
- ironic-lib： ironic的通用库函数
- ironic-webclinet ：web客户端
- ironic-ui：ironic的horizon插件
- bifrost：一套只运行Ironic的Ansible脚本

### 概念架构（与其他组件的关系）

下图显示了在提供物理机服务时各个组件的关系。（Ceilometer和Swift能与Ironic一起使用，但是在图中没有显示）

![概念架构]({{IMAGE_PATH}}/binli/ironic-presentation/conceptual_architecture.png)

### 逻辑架构（与其他组件的调用关系）

![Ironic 逻辑架构]({{IMAGE_PATH}}/binli/ironic-presentation/logical_architecture.png)


Ironic服务由以下组件构成：

- Ironic API，一个RESTful API服务，管理员和其他服务通过API与Ironic进行交互
- Ironic Conductor， 完成Ironic服务的绝大部分工作，通过API对外开放其功能，与Ironic API通过RPC进行交互；负责与其他组件进行交互
- Dirvers，真正管理物理机的模块，通过一系列的驱动来支持不同的硬件
- Database，用来存储资源信息
- 消息队列


### 部署架构

云平台管理员可以使用RESTful API注册硬件，制定硬件的属性，比如MAC地址、IPMI证书。可以开启多个API服务实例。

由于Ironic Conductor是唯一一个需要访问数据层和IPMI控制层的服务，为了安全起见，最好将conductor service 放在一个独立的主机上。为了支持各类驱动和管理故障迁移，可以有多个conductor实例存在，每个conductor实例可以运行多个drivers。

![ironic部署架构]({{IMAGE_PATH}}/binli/ironic-presentation/deployment_architecture_2.png)


**消息路由**

每一个Condutor实例在启动时想数据库注册自己，注册的信息包含了本实例支持的驱动列表，并且定期更新自己记录的时间戳，这就使得所有的服务能够知道哪些Condutor和哪些驱动可用。

物理机根据自己的驱动，使用一致性哈希算法映射在一组Condutor上。部署任务通过RPC从API层分发到合适的Condutor上。当Condutor实例加入或者退出集群，物理机会重新映射到不同的Condutor上，会触发驱动的多种动作，比如take-over 或者 clean-up动作。

![ironic_ha]({{IMAGE_PATH}}/binli/ironic-presentation/ironic_ha.png)


## 剖析ironic

### 关键技术

在安装操作系统时需要存储介质来存储系统镜像、需要控制物理机开关机，在网络部署环境中还需要预启动环境。
- PXE (预启动环境)
- IPMI（电源管理）
- iSCSI（存储）

#### 1. 什么是PXE

PXE(preboot execute environment) 预启动执行环境。PXE 是目前主流的无盘启动技术，它可以使计算机通过网络而不是从本地硬盘、光驱等设备启动，采用Client/Server的网络模式，在启动过程中，终端要求服务器分配IP地址，再用TFTP（trivial file transfer protocol）或MTFTP（multicast trivial file transfer protocol）协议下载一个启动软件包到本机内存中执行，由这个启动软件包完成终端基本软件设置，从而引导预先安装在服务器中的终端操作系统。

利用 PXE 进行系统安装需要被安装的主机上有 PXE 支持的网卡，不过现在的网卡一般都内嵌支持 PXE 的 ROM 芯片。当计算机引导时，BIOS 首先会把 PXE Client 调入内存中执行，PXE Client 被载入内存后，它便同时具有 DHCP client 和 TFTP Client 的功能，DHCP client 会向 DHCP server 请求 ip 分配给将要安装系统的主机，然后由 PXE Client 将放置在远端的文件通过 TFTP 下载到本地运行。

![pxe_demo]({{IMAGE_PATH}}/binli/ironic-presentation/pxe_demo.jpg)

安装流程如下：
1. 客户机从自己的PXE网卡启动，向本网络中的DHCP服务器索取IP，并搜寻引导文件的位置
2. DHCP服务器返回分给客户机IP以及NBP(Network Bootstrap Program )文件的放置位置(该文件一般是放在一台TFTP服务器上)
3. 客户机向本网络中的TFTP服务器索取NBP
4. 客户机取得NBP后之执行该文件
5. 根据NBP的执行结果，通过TFTP服务器加载内核和文件系统
6. 安装操作系统

PXE能提供操作系统镜像，但是如何远程开机呢，这件事就是由IPMI来做的。

#### 2. 什么是IPMI

智能平台管理接口（IPMI：Intelligent Platform Management Interface）是一项应用于服务器管理系统设计的标准，提供管理和监控CPU、固件（BIOS、UEFI）和操作系统等功能，由Intel、HP、Dell和NEC公司于1998年共同提出。利用此接口标准设计有助于在不同类服务器系统硬件上实施系统管理，使不同平台的集中管理成为可能，目前有超过200家公司支持IPMI。IPMI信息通过网络连接到**基板管理控制器 (BMC)**进行交流，不依赖BOIS或者操作系统，这使得在操作系统不响应或未加载的情况下其仍然可以进行开关机、信息提取等操作。Ironic 正是利用此技术可以远程的对裸机进行上下电或者其他操作，而不是依赖物理开关或者操作系统。

![ipmi]({{IMAGE_PATH}}/binli/ironic-presentation/ipmi.jpg)

IPMI可以在操作系统启动之前、管理系统启动之前、操作系统或者管理系统失败（与带内系统管理的最大特性）时对物理机进行管理，利用IPMI可以实现以下功能：

1. 可以在服务器通电（没有启动操作系统）情况下，对它进行远程管理：开机，关机，重启
2. 基于文本的控制台重定向，可以远程查看和修改BIOS设置，系统启动过程，登入系统等
3. 可以远程通过串口IP映射(SoL)连接服务器，解决ssh服务无法访问，远程安装系统，查看系统启动故障等问题
4. 可以通过系统的串行端口进行访问
5. 故障日志记录和 SNMP 警报发送，访问系统事件日志 (System Event Log ,SEL) 和传感器状况


IPMI技术功能点总结：

- 远程电源控制 (on / off / cycle / status)
- 串口的IP映射 Serial over LAN (SoL)
- 支持健康关机（Graceful shutdown support） 
- 机箱环境监控 (温度, 风扇转速, CPU电压等) 
- 远程设备身份LED控制(Remote ID LED control) 
- 系统事件日志（System event log）
- 平台事件跟踪（Platform Event Traps） 
- 数据记录（Data logging） 
- 虚拟KVM会话（Virtual KVM）
- 虚拟媒介（Virtual Media）

参考资料：
[https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface)


#### 3. 什么是iSCSI

iSCSI是一个指令集，iSCSI技术是一种新储存技术，该技术是将现有SCSI接口与以太网络(Ethernet)技术结合，使服务器可与使用IP网络的储存装置互相交换资料。

早期的企业使用的服务器若有大容量磁盘的需求时，通常是透过SCSI来串接SCSI磁盘，因此服务器上面必须要加装SCSI卡，而且这个SCSI是专属于该服务器的。 后来这个外接式的SCSI设备被SAN的架构所取代，在SAN的标准架构下，虽然有很多的服务器可以对同一个SAN 进行存取的动作，不过为了速度需求，通常使用的是光纤通道。但是光纤通道很贵，不但设备贵，服务器上面也要有光纤卡，很麻烦，所以光纤的SAN在中小企业很难普及。

后来网络实在太普及，尤其是以IP封包为基础的LAN技术已经很成熟，再加上以太网路的速度越来越快，所以就有厂商将SAN的连接方式改为利用IP技术来处理。 然后再透过一些标准的设定，最后就得到Internet SCSI (iSCSI)这个的产生！ iSCSI主要是透过TCP/IP的技术，将储存设备端透过iSCSI target (iSCSI目标端)功能，做成可以提供磁盘的服务器端，再透过iSCSI initiator (iSCSI初始化用户)功能，做成能够挂载使用iSCSI target的用户端，如此便能透过iSCSI设置来进行磁盘的应用了。

也就是说，iSCSI 这个架构主要将储存装置与使用的主机分为两个部分，分别是： 
- iSCSI target：就是储存设备端，存放磁盘或RAID的设备，目前也能够将Linux主机模拟成iSCSI target了！ 目的在提供其他主机使用的磁盘。
- iSCSI initiator：就是能够使用target的用户端，通常是服务器。 也就是说，想要连接到iSCSI target的服务器，也必须要安装iSCSI initiator的相关功能后才能够使用iSCSI target提供的磁盘。

![iSCSI示意图]({{IMAGE_PATH}}/binli/ironic-presentation/iscsi.jpg)




## ironic内部技术

### ironic代码结构

下面是ironic liberty版的代码结构，其中省略了部分文件

```sh

	ironic-stable-liberty:.
	│ 
	│  babel.cfg
	│  CONTRIBUTING.rst
	│  driver-requirements.txt
	│  LICENSE
	│  MANIFEST.in
	│  openstack-common.conf
	│  README.rst
	│  RELEASE-NOTES
	│  requirements.txt
	│  setup.cfg      #pbr的配置文件
	│  setup.py 
	│  test-requirements.txt
	│  tox.ini        #tox配置文件
	│  vagrant.yaml
	│  Vagrantfile
	│  
	├─doc   #文档
	│              
	├─etc  #相关配置文件
	│              
	├─ironic
	│  │  
	│  ├─api        #api,使用peach框架
	│  │  │  acl.py
	│  │  │  app.py     #API入口
	│  │  │  app.wsgi
	│  │  │  config.py
	│  │  │  expose.py
	│  │  │  hooks.py
	│  │  │  __init__.py
	│  │  │  
	│  │  ├─controllers   #控制器
	│  │  │  │  base.py
	│  │  │  │  link.py
	│  │  │  │  root.py   #API总控制器
	│  │  │  │  __init__.py
	│  │  │  │  
	│  │  │  └─v1      #v1 版本API
	│  │  │          
	│  │  └─middleware
	│  │          
	│  ├─cmd            # 服务入口
	│  │      api.py
	│  │      conductor.py
	│  │      dbsync.py
	│  │      __init__.py
	│  │      
	│  ├─common  #常用方法
	│  │              
	│  ├─conductor  
	│  │      manager.py  
	│  │      rpcapi.py
	│  │      task_manager.py
	│  │      utils.py
	│  │      __init__.py
	│  │      
	│  ├─db    #数据库相关，包括sqlalchemy、alembic
	│  │                  
	│  ├─dhcp  #neutron dhcp api
	│  │      base.py
	│  │      neutron.py
	│  │      none.py
	│  │      __init__.py
	│  │      
	│  ├─drivers    #驱动相关
	│  │  │  agent.py    #agent_* 驱动
	│  │  │  base.py
	│  │  │  drac.py     #drac驱动
	│  │  │  fake.py
	│  │  │  ilo.py      #ilo驱动，惠普的
	│  │  │  irmc.py     #irmc
	│  │  │  pxe.py      #pxe_* 驱动
	│  │  │  raid_config_schema.json
	│  │  │  utils.py
	│  │  │  __init__.py
	│  │  │  
	│  │  └─modules     #驱动的具体方法
	│  │    
	│  │              
	│  ├─locale      #翻译
	│  │              
	│  ├─nova        #nova相关，这里包括与Nova computer冲突的临时解决办法
	│  │          
	│  ├─objects     #基本对象的方法，包括conducotr、node、port、field、chassis
	│  │      
	│  ├─openstack  #openstack相关的，这里包含处理镜像的方法的帮助函数
	│  │          
	│  └─tests    #测试相关
	│      
	│              
	├─releasenotes   #发布历史
	│              
	└─tools     #相关工具

           
```


### API
ironic-api提供一系列接口，详见[ironic API](http://developer.openstack.org/api-ref/baremetal/?expanded=change-node-provision-state-detail,get-console-detail,start-stop-console-detail,show-driver-logical-disk-properties-detail#)。

+ 节点相关（node）
  + 节点增删改查（List, Searching, Creating, Updating, and Deleting)
  + 合法性检查
  + 设置和清除维修状态
  + 设置和获取boot device
  + 获取节点当前综合信息，包括power, provision, raid, console等
  + 更改电源状态
  + 更改节点提供状态（ manage, provide, inspect, clean, active, rebuild, delete (deleted), abort）
  + 设置RAID
  + 启动、停止、获取console
  + 查看、调用厂商定制方法（passthru方法）
+ 端口相关(Port)
  + 对物理端口（Port）的增删改查（Listing, Searching, Creating, Updating, and Deleting ），新建的时候就要指定端口的物理地址(一般是MAC地址)与Node进行绑定。
  + 查看与Node连接的端口
+ 驱动相关（driver）
  + 列举所有驱动
  + 查看驱动的详细信息、属性
  + 查看和调用厂商的驱动
+ Chassis（机箱，一组node的集合）
  + 增删改查

Chassis这个资源类型是为了给节点分组用的，目前只有列举一组节点的功能。不赞成使用这个类型，将来可能会去除掉。


### Conductor

Ironic-Conductor是Ironic中最主要的模块，通过Ironic-API对外提供功能，与Ironic-API之间通过RPC进行通信，负责绝大部分工作，包括与Neutron通信为物理机配置网络信息，与Glance通信获取镜像，与Cinder和Swift通信进行为物理机提供存储。


同时控制着物理机状态的改变过程，下图是openstack Liberty中物理机状态转换图：

![Ironic’s State Machine]({{IMAGE_PATH}}/binli/ironic-presentation/states.svg)


在高可用方面，conductor 采用一致性哈希算法保证Condutor节点新增退出时不影响Bare metal节点。

![ironic_ha]({{IMAGE_PATH}}/binli/ironic-presentation/ironic_ha.png)


### DB

采用MySQL，存储物理机和驱动的状态信息，可以换成其他数据库。


### Drivers

驱动是真正操作物理机的模块，Ironic的驱动以插件形式设计，厂商可以实现自己的驱动来为自己的设备提供特色化功能。实现自己的驱动只需要实现几个相关的方法即可。

#### 驱动架构

驱动有不同的属性，每个属性能够完成一定的功能，这些属性分为三种接口：
 + 核心接口（core），最基本的功能，是其他服务的依赖，所有驱动都必须实现的，包括power,deploy.
 + 标准接口（standard）,实现通用功能，主要包括 management, console, boot, inspect, raid. 
 + 厂商接口(vendor)，提供个性化功能

接口功能：

| 分类 | 接口 | 功能 | 主要需要实现的方法 |
| :-- | :-- | :-- | :-- |
| core | power | 管理电源 |get_properties、validate、 get_power_state、set_power_state、reboot |
|  | deploy | 部署方式 |get_properties、validate、 deploy、tear_down、prepare、clean_up、take_over、prepare_cleaning、tear_down_cleaning |
| standard | console | 通过硬件得到物理机的控制台 | get_properties、validate、start_console、stop_console、get_console |
|  | management | 管理物理机硬件 | get_properties、validate、get_supported_boot_device、set_boot_device、get_boot_device 、get_sensors_data|
|  | boot | 启动相关的动作| get_properties、validate、prepare_ramdisk、clean_up_ramdisk、prepare_instance、clean_up_instance |
|  | inspect | 硬件自检，主要检查内存、CPU、本地分区 | inspect_hardware | 
|  | raid  | 设置raid | get_properties、validate、 create_configuration、delete_configuration、get_logical_disk_properties |
| vendor |   | 厂商的自定义功能 | get_properties、validate、... |


驱动架构示意图：

![驱动结构示意图]({{IMAGE_PATH}}/binli/ironic-presentation/ironic_drivers.png)



Liberty支持的驱动，在setup.cfg中可以看到

```
    agent_ilo = ironic.drivers.ilo:IloVirtualMediaAgentDriver
    agent_ipmitool = ironic.drivers.agent:AgentAndIPMIToolDriver
    agent_irmc = ironic.drivers.irmc:IRMCVirtualMediaAgentDriver
    agent_pyghmi = ironic.drivers.agent:AgentAndIPMINativeDriver
    agent_ssh = ironic.drivers.agent:AgentAndSSHDriver
    agent_vbox = ironic.drivers.agent:AgentAndVirtualBoxDriver
    agent_ucs = ironic.drivers.agent:AgentAndUcsDriver
    iscsi_ilo = ironic.drivers.ilo:IloVirtualMediaIscsiDriver
    iscsi_irmc = ironic.drivers.irmc:IRMCVirtualMediaIscsiDriver
    pxe_ipmitool = ironic.drivers.pxe:PXEAndIPMIToolDriver
    pxe_ipminative = ironic.drivers.pxe:PXEAndIPMINativeDriver
    pxe_ssh = ironic.drivers.pxe:PXEAndSSHDriver
    pxe_vbox = ironic.drivers.pxe:PXEAndVirtualBoxDriver
    pxe_seamicro = ironic.drivers.pxe:PXEAndSeaMicroDriver
    pxe_iboot = ironic.drivers.pxe:PXEAndIBootDriver
    pxe_ilo = ironic.drivers.pxe:PXEAndIloDriver
    pxe_drac = ironic.drivers.drac:PXEDracDriver
    pxe_snmp = ironic.drivers.pxe:PXEAndSNMPDriver
    pxe_irmc = ironic.drivers.pxe:PXEAndIRMCDriver
    pxe_amt = ironic.drivers.pxe:PXEAndAMTDriver
    pxe_msftocs = ironic.drivers.pxe:PXEAndMSFTOCSDriver
    pxe_ucs = ironic.drivers.pxe:PXEAndUcsDriver
    pxe_wol = ironic.drivers.pxe:PXEAndWakeOnLanDriver
    pxe_iscsi_cimc = ironic.drivers.pxe:PXEAndCIMCDriver
    pxe_agent_cimc = ironic.drivers.agent:AgentAndCIMCDriver
    
    #fake_ 驱动是假的驱动，里面没有具体实现，用于测试
    fake = ironic.drivers.fake:FakeDriver
    fake_agent = ironic.drivers.fake:FakeAgentDriver
    fake_inspector = ironic.drivers.fake:FakeIPMIToolInspectorDriver
    fake_ipmitool = ironic.drivers.fake:FakeIPMIToolDriver
    fake_ipminative = ironic.drivers.fake:FakeIPMINativeDriver
    fake_ssh = ironic.drivers.fake:FakeSSHDriver
    fake_pxe = ironic.drivers.fake:FakePXEDriver
    fake_seamicro = ironic.drivers.fake:FakeSeaMicroDriver
    fake_iboot = ironic.drivers.fake:FakeIBootDriver
    fake_ilo = ironic.drivers.fake:FakeIloDriver
    fake_drac = ironic.drivers.fake:FakeDracDriver
    fake_snmp = ironic.drivers.fake:FakeSNMPDriver
    fake_irmc = ironic.drivers.fake:FakeIRMCDriver
    fake_vbox = ironic.drivers.fake:FakeVirtualBoxDriver
    fake_amt = ironic.drivers.fake:FakeAMTDriver
    fake_msftocs = ironic.drivers.fake:FakeMSFTOCSDriver
    fake_ucs = ironic.drivers.fake:FakeUcsDriver
    fake_cimc = ironic.drivers.fake:FakeCIMCDriver
    fake_wol = ironic.drivers.fake:FakeWakeOnLanDriver
```

可以将上面的驱动进行分类，得到：

- pxe/iscis
- agent
- fake (假的驱动，用于示例，测试)

我们把fake驱动排除，其他的驱动可以分为两类：

- 一是以pex_ 或者 iscsi_ 为前缀的驱动采用PXE部署机制，这些驱动将根硬盘作为iSCSI设备暴露给ironic conductor，由conductor将镜像复制到这里.
- 二是以agent_  为前缀的驱动采用Agent部署机制，conductor准备一个存储在swift上的镜像URL给IPA，由IPA下载镜像和完成部署的操作。

从Kilo版开始，所有驱动使用agent进行部署。

每种驱动的功能会调用不同的模块来实现，比如:

- pxe_  deploy 用的是iscsi_deploy.ISCSIDeploy(), boot用的是pxe.PXEBoot(), power根据后缀不同使用的不同
- agent_ deploy用的是agent.AgentDeploy(), boot 用的是pxe.PXEBoot(), power根据后缀不同而不同
- iscis_ deploy用的iscsi_deploy.ISCSIDeploy()，power根据后缀不同而不同

#### 驱动列表

- 社区驱动

| 种类 | 用途 | 实现的驱动 |
| -- | -- | -- |
| SSH | 用VM模拟物理机时使用 | pxe-ssh使用pxe部署、ssh管理电源; agent-ssh使用agent部署、ssh管理电源 |
| VirtualBox | VirtualBox驱动用来将虚拟机模拟成裸机节点进行测试Ironic。Ironic使用pex_ssh和agent_ssh驱动连接VirtualBox主机（要安装在Linux下，Windows下SSH不太好用） | - pxe_vbox:使用基于iSCSI的部署机制- agent_vbox：使用基于agent的部署机制 |
| IPMI| 使用IPMItool进行管理电源 |  pxe_ipmitool; agent_ipmitool;|
|pyghmi | pyghmi是ironic实现的一个Python ipmitool lib 用于替代ipmitool |  pxe_ipminative;agent_pyghmi |


- 厂商驱动

| 种类 | 说明 | 实现的驱动 |
| -- | -- | -- |
| DRAC | 戴尔公司的一种系统管理硬件和软件解决方案（Dell Remote Access Controller） | pxe_drac |
| AMT | 英特尔主动管理技术，远程带外管理个人电脑的硬件和固件的技术，用于监控、维修、更新、升级硬件和固件 | pex_amt |
| SNMP | SNMP电源驱动用户管理数据中心机架的配电单元，可与PXE驱动结合起来用于网络部署和配置 | pxe_snmp |
| iLO | iLO是Integrated Ligths-out的简称，是HP服务器上集成的远程管理端口，只要将服务器接入网络并且没有断开服务器的电源，不管HP服务器的处于何种状态（开机、关机、重启），都可以允许用户通过网络进行远程管理 | iscsi_ilo, agent_ilo, pxe_ilo。 iscsi_ilo和agent_ilo使用iLO实现带外管理，iscsi_ilo使用diskimage-builder建立镜像，从网络启动；而agent_ilo使用IPA的部署镜像，裸机节点从本地启动。pxe_ilo使用PXE/iSCSI部署（和常规PXE驱动一样） |
| SeaMicro | SeaMicro服务器是由AMD公司发布的，基于ARM处理器，特点是节能。SeaMicro电源驱动可以使用SeaMicro服务器的电源周期管理功能。 | pxe_seamicro， 使用PXE/iSCSI进行部署镜像，然后使用SeaMicro替代IPMI对裸机进行管理。 |
| iRMC | 富士通的驱动，通过ServerView Common Command Interface(SCCI，富士通的卡)控制电源 | - pxe_irmc，使用PXE部署 - iscis_irmc，支持用虚拟光驱来部署镜像 - agent_irmc，支持使用虚拟光驱部署IPA |
| UCS | 思科的驱动，用户管理思科 UCS B/C 系列的服务器（类似IPMI） |- pxe_ucs： 使用PXE/iSCSI部署镜像，使用UCS代替IPMI管理节点 - agent_ucs：使用IPAramdisk部署镜像，使用UCS代替IPMI管理节点 |
| CIMC | 思科为standalone Cisco UCS C系列服务器提供的驱动，可以利用CIMC进行管理裸机（代替IPMI） | pxe_iscsi_cimc,使用PXE+iSCSI部署  - pxe_agent_cimc，使用PXE启动+Agent部署 |
| Wake-On-Lan | Wake-On-Lan是一个允许通过网络消息打开电脑电源的标准，不需要额外硬件，但还在测试阶段。Wake-On-Lan只有打开电源的功能，关闭电源需要手工执行。它没有获取电源状态的能力，任何电源状态的API调用它只返回数据库里的值。 | pex_wol：使用PXE/iPXE启动，iSCSI部署。 | 
| iBoot | iBoot power driver通过DxP协议对使用了Dataprobe（美国一家制造商）iBoot 设备的服务器进行电源周期管理 | pxe_iboot,使用PXE/iPXE启动，iSCSI部署。  |




### Ironic-Python-Agent

在PXE部署环境中，deploy模块是通过打开一个iSCSI设备，ironic-conductro将OS的镜像文件写到iSCSI的设备，所以deploy_ramdisk只是完成了iSCSI部署的工作，但开发者觉得既然已经把kernel和ramdisk传过去了，只做一个工作是不是太少了，而且还太缺乏灵活性了，所以就想在ramdisk里装一个Python Agent。 实际上就是多提供了一个Restful API,控制节点可以通过这个agent远程实现与物理机节点互动，而不仅仅使用dd命令。

Ironic Python Agent（简称IPA或者agent）是一个基于python的代理，用于处理ironic中裸机节点的一系列动作，比如检查、配置、清除和部署镜像。运行在ramdisk中，暴露出REST API给conductor。Ironic-Python-Agent可以在deploy模块直接访问硬件，提供以下功能：

- 磁盘格式化
- 磁盘分区
- 安装OS( Bootloaders, OS)
- 固件升级
- raid配置

在Condutor端使用agent驱动，物理机端使用IPA，IPA通过暴露API给Condutor调用，则可完成相应功能。IPA启动时通过发送lookup()请求给Condutor获取UUID，相当于注册自己，并且每隔一段时间给Condutor发送心跳包进行连接。

##### IPA架构

###### 与conductor的交互

IPA使用lookup和hearteat机制与Ironic Conductor进行交互

1. 启动时agent给Conductor的vendor_passthru lookup endpoint（地址为/v1/drivers/{driver}/vendor_passthru/lookup）发送一个硬件的profile
2. 然后Ironic就可以得出该节点的uuid，在成功发现该节点之后，agent隔N秒发送心跳包给Conductor（hearteat地址为/v1/nodes/{node_ident}/vendor_passthru/heartbeat ）
3. conductor执行一系列动作，包括查询已经运行的命令的状态，

###### 与ramdisk、agent的关系
IPA是一个运行在ramdisk中python程序，当物理机注册时使用agent为前缀的驱动时，则会使用agent方式部署，即允许包含有IPA的ramdisk。

#### IPA如何管理硬件

**硬件管理器**

硬件管理器（HardwareManager）是IPA中的一个概念，IPA通过重写硬件管理器来支持多种硬件平台。通过自定义 hardware managers 可以允许用户引入特定的硬件工具集、文件和清除步骤等等，比如可以引入 BIOS flashing utility and BIOS file，然后在cleaning step中重写BIOS版本。

修改硬件的方法按照优先顺序发送给每个管理器，管理器检查是否包含该方法，如果没有则抛出IncompatibleHardwareMethodError异常，IPA继续发送给下一个管理器，直到某个管理器包含该方法并且返回方法的结果，如果所有的管理器都没有改方法则抛出 HardwareManagerMethodNotFound异常。


##### pxe部署与agent部署

使用pxe部署流程：

![iscsi部署流程]({{IMAGE_PATH}}/binli/ironic-presentation/pxe_deploy.jpg)


使用IPA部署流程：

![IPA部署流程]({{IMAGE_PATH}}/binli/ironic-presentation/agent_deploy.jpg)

------------------------------------------------

## 安装及配置
由于Ironic的配置很长，下面我们简短的说一下安装和配置过程，具体的安装配置教程参考[官方手动配置教程](http://docs.openstack.org/developer/ironic/deploy/install-guide.html)或者[使用devstack安装](http://docs.openstack.org/developer/ironic/dev/dev-quickstart.html#deploying-ironic-with-devstack)。

Ironic需要与Nova、Neutron、Glance、Keystone等进行交互，于是先要对这些进行配置。

### 1.配置KeyStone
首先我们需要在keystone中创建ironic用户、服务、endpoint信息。
1. keystone user-create
2. keystone user-add-role
3. keystone servcie-create
4. keystone endpoint-create

### 2.配置ironic
1. 安装MySQL，创建ironic数据库，并授权给ironic用户
2. 安装ironic基础组件，ironic-api、ironic-condutor、python-ironicclient
3. 配置/etc/ironic/ironic.conf,设置DB、keystone、Nova、Neutron、Glance、RabbitMQ等地址、用户名、密码等相关信息
4. 重启ironic-api、ironic-condutor服务

### 3.配置Nova
配置nova controller和computer节点/etc/nova/nova.conf，将ironic设置Nova的驱动，比如：
```
[DEFAULT]  
scheduler_host_manager = nova.scheduler.ironic_host_manager.IronicHostManager  
compute_driver = nova.virt.ironic.driver.IronicDriver  
compute_manager = ironic.nova.compute.manager.ClusteredComputeManager  
[ironic]  
admin_username = ironic  
admin_password = unset  
admin_url = http://127.0.0.1:35357/v2.0  
admin_tenant_name = service  
```
然后重启nova-computer和 nova-scheduler服务。

### 4.配置Neutorn
为了裸机能够和neutron通信，需要增加虚拟网络设置，以便裸机能够获取DHCP、PXE boot等服务。同时需要提供每个裸机的MAC地址给ironic，然后ironic将信息发送给neutron来给节点获取DHCP、PXE boot配置等信息。
1. 在/etc/neutron/plugins/ml2/ml2_conf.ini中配置网络模式、防火墙驱动、桥接映射等
2. 将配置加入到虚拟网络中，`ovs-vsctl add-br br-int`
3. 重启Open vSwitch agent， `service neutron-plugin-openvswitch-agent restart`
4. 创建网格和子网
5. 在ironic-Condutor节点上配置cleaning_network_uuid，用于Node Clear(将节点回收之前的动作) (/etc/ironic/ironic.conf，`cleaning_network_uuid = NETWORK_UUID`)
6. 重启ironic-Condutor服务

### 5.配置Glance
部署裸机过程中需要两套镜像:deploy image 和user image，deploy image负责进行部署过程，包括获取user image、硬件管理、清除部署等等动作；user image是用户使用的系统镜像。
1. 生成部署镜像(deploy image)和用户镜像(user image)
2. 将部署镜像和用户镜像加入到Glance中

### 6.创建flavor
在Nova-computer节点创建裸机的flavor，指定RAM_MB、CPU、DISK_GB、ARCH等信息。

### 7.配置驱动的相关环境
不同驱动需要一些不同的环境，需要在Ironic-Condutor安装和配置与这些驱动的环境

**PXE**

如果使用PXE环境启动，那么要配置PXE环境，包括DHCP、TFTP服务，由于DHCP服务在Neutron中配置了，所以还需要在ironic-Condutor配置TFTP服务，具体步骤见[PXE setup](http://docs.openstack.org/developer/ironic/liberty/deploy/install-guide.html#pxe-setup)

**PXE UEFI**

新型UEFI，全称“统一的可扩展固件接口”（Unified Extensible Firmware Interface）， 是一种详细描述类型接口的标准。这种接口用于操作系统自动从预启动的操作环境，加载到一种操作系统上，其主要目的是为了提供一组在 OS 加载之前（启动前）在所有平台上一致的、正确指定的启动服务，被看做是有近20多年历史的 BIOS 的继任者。

如果想部署一台使用UEFI的物理机，则需要配置PXE环境和UEFI环境，具体步骤见[PXE-UEFI](http://docs.openstack.org/developer/ironic/liberty/deploy/install-guide.html#pxe-uefi-setup)

**iPXE**

是一种PXE的替代版，与PXE相比能做更多的事，从J版引入，需要在/etc/ironic/ironic.conf中开启服务，同时需要在neutron中的DHCP Agent中更新相关配置（/etc/neutron/dhcp_agent.ini,`dnsmasq_config_file = /etc/dnsmasq-ironic.conf`）并重启DHCP Agent。
以下是iPXE的特性。

```
boot from a web server via HTTP
boot from an iSCSI SAN
boot from a Fibre Channel SAN via FCoE
boot from an AoE SAN
boot from a wireless network
boot from a wide-area network
boot from an Infiniband network
control the boot process with a script
```

** IPMI**

ipmitool可实现远程开关机、显示系统日志、获取传感器信息等操作，一般系统里自带，如果系统使用的是openipmi代替ipmitool的话则不可以使用IPMITool Driver,比如一些Mac OS X和SLES系统。

IPMITool Driver和IPMICommand必须装在Ironic-Condutor节点上，同时能够连接到IPMI Controller（BMC,有独立的IP）上，从K版开始，支持传输传感器数据（硬件温度、风扇、电压、电流），但是默认关闭，要在irnic.conf中开启。

支持传输传感器数据的驱动有pxe_ipmitool, pxe_ipminative, agent_ipmitool, agent_pyghmi, agent_ilo, iscsi_ilo, pxe_ilo,pxe_irmc。

**boot model**

节点的启动模式（Legacy BIOS or UEFI）可以设置，默认使用BIOS，但是一个节点只能设置一种启动方式。如果需要设置的话，需要在ironic服务中指定
```
ironic node-update <node-uuid> add properties/capabilities='boot_mode:uefi'
```
然后在Nova-Computer节点上更新

```
nova flavor-key ironic-test-3 set capabilities:boot_mode="uefi"
nova boot --flavor ironic-test-3 --image test-image instance-1
```

支持设置启动模式的驱动有xe_ipmitool。

**local boot**

从K版以后支持部署阶段完成后使用本地启动，即部署阶段完成后进入用户系统的过程从物理机本地启动，而不使用PXE启动。设置本地启动需要在节点信息中加入本地启动的信息，镜像内需要安装grab2。
```
#1.不需要nova-computer节点
ironic node-update <node-uuid> add instance_info/capabilities='{"boot_option": "local"}'

#2.或者需要nova-computer节点
ironic node-update <node-uuid> add properties/capabilities="boot_option:local"
nova flavor-key baremetal set capabilities:boot_option="local"
```

### 8.注册裸机
下面以Liberty版本的API来说明，驱动为pxe_ipmitool。

1.在Ironic-Condutor节点创建一个裸机节点（node），指定驱动

```
#注册一个驱动为pxe_ipmitool的节点， 在Liberty版本（API版本1.1以上）节点状态变为enroll,指ironic知道了这个节点，但是没有还管理它
ironic --ironic-api-version 1.11 node-create -d pxe_ipmitool -n post11

+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| uuid         | 0eb013bb-1e4b-4f4c-94b5-2e7468242611 |
| driver_info  | {}                                   |
| extra        | {}                                   |
| driver       | pxe_ipmitool                         |
| chassis_uuid |                                      |
| properties   | {}                                   |
| name         | post11                               |
+--------------+--------------------------------------+


ironic --ironic-api-version 1.11 node-list

+--------------------------------------+--------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name   | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+--------+---------------+-------------+--------------------+-------------+
| 0eb013bb-1e4b-4f4c-94b5-2e7468242611 | post11 | None          | None        | enroll             | False       |
+--------------------------------------+--------+---------------+-------------+--------------------+-------------+
```

2.设置driver参数（driver_info），包括驱动相关配置、部署镜像id

```
ironic driver-properties pxe_ipmitool
+----------------------+---------------------------------------------------------------------------+
| Property             | Description                                       |
+----------------------+---------------------------------------------------------------------------+
| ipmi_address         | IP address or hostname of the node. Required.                 |
| ipmi_password        | password. Optional.                                   |
| ipmi_username        | username; default is NULL user. Optional.                    |
| ...                  | ...                                           |
| deploy_kernel        | UUID (from Glance) of the deployment kernel. Required.             |
| deploy_ramdisk       | UUID (from Glance) of the ramdisk that is mounted at boot time. Required.  |
+----------------------+----------------------------------------------------------------------------+
# 设置IPMI BMC
ironic node-update $NODE_UUID add \
driver_info/ipmi_username=$USER \
driver_info/ipmi_password=$PASS \
driver_info/ipmi_address=$ADDRESS

#设置部署镜像
ironic node-update $NODE_UUID add \
driver_info/deploy_kernel=$DEPLOY_VMLINUZ_UUID \
driver_info/deploy_ramdisk=$DEPLOY_INITRD_UUID
```

3.设置节点属性（properties） 

```
#设置flavor，要与之前创建的匹配
ironic node-update $NODE_UUID add \
properties/cpus=$CPU \
properties/memory_mb=$RAM_MB \
properties/local_gb=$DISK_GB \
properties/cpu_arch=$ARCH

#设置过滤条件，可选
ironic node-update $NODE_UUID add \
properties/capabilities=key1:val1,key2:val2
```

4.设置MAC地址

```
ironic port-create -n $NODE_UUID -a $MAC_ADDRESS
```

5.检查驱动接口信息

```
ironic node-validate $NODE_UUID          

+------------+--------+--------+
| Interface  | Result | Reason |
+------------+--------+--------+
| console    | True   |        |
| deploy     | True   |        |
| management | True   |        |
| power      | True   |        |
+------------+--------+--------+
```
6.执行manage操作，更改状态enroll -> manageable 

manage过程中节点状态为verifying，检查成功后，物理机的变为manageable，指物理机拥有足够的信息来管理硬件，但是还不是provisioning的状态，provisioning的状态是available。
```
ironic --ironic-api-version 1.11 node-set-provision-state $NODE_UUID manage

ironic node-show $NODE_UUID

+------------------------+--------------------------------------------------------------------+
| Property               | Value                                                              |
+------------------------+--------------------------------------------------------------------+
| ...                    | ...                                                                |
| provision_state        | manageable                                                         | <- verify correct state
| uuid                   | 0eb013bb-1e4b-4f4c-94b5-2e7468242611                               |
| ...                    | ...                                                                |
+------------------------+--------------------------------------------------------------------+
```

7.执行provide操作，更改状态manageable -> available 

provide是清理节点上的ramdisk等，将节点变为能够重新部署的状态，如果设置了node Clean操作，则会进行指定的步骤进行节点清理。
```
ironic --ironic-api-version 1.11 node-set-provision-state $NODE_UUID provide

ironic node-show $NODE_UUID

+------------------------+--------------------------------------------------------------------+
| Property               | Value                                                              |
+------------------------+--------------------------------------------------------------------+
| ...                    | ...                                                                |
| provision_state        | available                                                          | < - verify correct state
| uuid                   | 0eb013bb-1e4b-4f4c-94b5-2e7468242611                               |
| ...                    | ...                                                                |
+------------------------+--------------------------------------------------------------------+
```
节点状态变成available之后就可以被调度了。

结合状态装换图来理解

![Ironic’s State Machine]({{IMAGE_PATH}}/binli/ironic-presentation/states.svg)


##理解裸机部署过程

部署物理机跟部署虚拟机的概念在nova来看是一样，都是nova通过创建虚拟机的方式来触发，只是底层nova-scheduler和nova-compute的驱动不一样。虚拟机的底层驱动采用的libvirt的虚拟化技术，而物理机是采用Ironic技术，ironic可以看成一组Hypervisor API的集合，其功能与libvirt类似。

###操作系统安装过程
####Linux系统启动过程

- bootloader（引导程序，常见的有GRUB、LILO）
- kernel（内核）
- ramdisk（虚拟内存盘）
- initrd/initramfs （初始化内存磁盘镜像）


下面我们分别介绍每个概念：
- **引导加载程序**是系统加电后运行的第一段软件代码。PC机中的引导加载程序由BIOS(其本质就是一段固件程序)和位于硬盘MBR（主引导记录，通常位于第一块硬盘的第一个扇区）中的OS BootLoader（比如，LILO和GRUB等）一起组成。BIOS在完成硬件检测和资源分配后，硬盘MBR中的BootLoader读到系统的RAM中，然后控制权交给OS BootLoader。
- **bootloader**负责将kernel和ramdisk从硬盘读到内存中，然后跳转到内核的入口去运行。
- **kernel**是Linux的内核，包含最基本的程序。
- **ramdisk**是一种基于内存的虚拟文件系统，就好像你又有一个硬盘，你可以对它上面的文件添加修改删除等等操作。但是一掉电，就什么也没有了，无法保存。一般驱动程序放在这里面。
- **initrd**是boot loader initialized RAM disk, 顾名思义，是在系统初始化引导时候用的ramdisk。也就是由启动加载器所初始化的RamDisk设备，它的作用是完善内核的模块机制，让内核的初始化流程更具弹性；内核以及initrd，都由 bootloader在机子启动后被加载至内存的指定位置，主要功能为按需加载模块以及按需改变根文件系统。**initramfs**与initrd功能类似，是initrd的改进版本，改进了initrd大小不可变等等缺点。

![Linux-booting-process]({{IMAGE_PATH}}/binli/ironic-presentation/Linux-Booting-process.png)

![boot procedure]({{IMAGE_PATH}}/binli/ironic-presentation/linux-boot-procedure2.jpg)

![Linux boot process]({{IMAGE_PATH}}/binli/ironic-presentation/linux-boot-process3.PNG)

#####为什么需要initrd?
在早期的Linux系统中，一般就只有软盘或者硬盘被用来作为Linux的根文件系统，因此很容易把这些设备的驱动程序集成到内核中。但是现在根文件系统 可能保存在各种存储设备上，包括SCSI, SATA, U盘等等。因此把这些设备驱动程序全部编译到内核中显得不太方便，违背了“内核”的精神。在Linux内核模块自动加载机制中可以利用udevd可以实现内核模块的自动加载，因此我们希望根文件系统的设备驱动程序也能够实现自动加载。但是这里有一个矛盾，udevd是一个可执行文件，在根文件系统被挂载前，是不可能执行udevd的，但是如果udevd没有启动，那就无法自动加载根根据系统设备的驱动程序，同时也无法在/dev目录下建立相应的设备节点。

为了解决这个矛盾，于是出现了initrd(boot loader initialized RAM disk)。initrd是一个被压缩过的小型根目录，这个目录中包含了启动阶段中必须的驱动模块，可执行文件和启动脚本。包括上面提到的udevd，当系统启动的时候，bootload会把内核和initrd文件读到内存中，然后把initrd的起始地址告诉内核。内核在运行过程中会解压initrd，然后把initrd挂载为根目录，然后执行根目录中的/initrc脚本，可以在这个脚本中运行initrd中的udevd，让它来自动加载设备驱动程序以及 在/dev目录下建立必要的设备节点。在udevd自动加载磁盘驱动程序之后，就可以mount真正的根目录，并切换到这个根目录中。



#####Linux启动一定要用initrd么？
如果把需要的功能全都编译到内核中(非模块方式)，只需要一个内核文件即可。initrd 能够减小启动内核的体积并增加灵活性，如果你的内核以模块方式支持某种文件系统(例如ext3, UFS)，而启动阶段的驱动模块放在这些文件系统上,内核是无法读取文件系统的，从而只能通过initrd的虚拟文件系统来装载这些模块。这里有些人会问: 既然内核此时不能读取文件系统，那内核的文件是怎么装入内存中的呢?答案很简单，Grub是file-system sensitive的，能够识别常见的文件系统
　通用安装流程：
 1. 开机启动，BIOS完成硬件检测和资源分配，选择操作系统的启动（安装）模式（此时，内存是空白的）
 2. 然后根据相关的安装模式，寻找操作系统的引导程序（bootloader）（不同的模式，对应不同的引导程序当然也对应着不同的引导程序存在的位置）
 3. 引导程序加载文件系统初始化（initrd）程序和内核初始镜像（vmlinuz），完成操作系统安装前的初始化
 4. 操作系统开始安装相关的系统和应用程序。

###PXE部署过程

PXE协议分为client和server两端，PXE client在网卡的ROM中，当计算机启动时，BIOS把PXE client调入内存执行，并显示出命令菜单，经用户选择后，PXE client将放置在远端的操作系统通过网络下载到本地运行。

安装流程如下：
1. 客户机从自己的PXE网卡启动，向本网络中的DHCP服务器索取IP，并搜寻引导文件的位置
2. DHCP服务器返回分给客户机IP以及NBP(Network Bootstrap Program )文件的放置位置(该文件一般是放在一台TFTP服务器上)
3. 客户机向本网络中的TFTP服务器索取NBP
4. 客户机取得NBP后之执行该文件
5. 根据NBP的执行结果，通过TFTP服务器加载内核和文件系统
6. 安装操作系统

![PXE过程图]( {{IMAGE_PATH}}/binli/ironic-presentation/pxe_demo.jpg)

流程小结：
客户端广播dhcp请求——服务器相应请求，建立链接——由dhcp和tftp配置得到ip还有引导程序所在地点——客户端下载引导程序并开始运行——引导程序读取系统镜像-安装操作系统

相关文件位置与内容:
- dhcp配置文件/etc/dhcpd/dhcp.conf——ip管理与引导程序名称
- tftp配置文件/etc/xinetd.d/tftp——tftp根目录，和上面的引导程序名称组成完整路径
- 引导程序读取的配置文件/tftpboot/pxelinux.cfg/default——启动内核其他

参考资料：
[PXE网络安装操作系统过程](http://www.blogjava.net/qileilove/archive/2013/10/14/404942.html)


###Ironic部署过程

####部署流程

![Bare Metal Deployment Steps]({{IMAGE_PATH}}/binli/ironic-presentation/deployment_steps.png)

此图是Liberty版的官方裸机部署过程图，部署过程描述如下：

1. 部署物理机的请求通过 Nova API 进入Nova；
2. Nova Scheduler 根据请求参数中的信息（指定的镜像和硬件模板等）选择合适的物理节点；
3. Nova 创建一个 spawn 任务，并调用 Ironic API 部署物理节点，Ironic 将此次任务中所需要的硬件资源保留，并更新数据库；
4. Ironic 与 OpenStack 的其他服务交互，从 Glance 服务获取部署物理节点所需的镜像资源，并调用 Neutron 服务为物理机创建网路端口；
5. Ironic 开始部署物理节点，PXE driver 准备 tftp bootloader，IPMI driver 设置物理机启动模式并将机器上电；
6. 物理机启动后，通过 DHCP 获得 Ironic Conductor 的地址并尝试通过 tftp 协议从 Conductor 获取镜像，Conductor 将部署镜像部署到物理节点上后，通过 iSCSI 协议将物理节点的硬盘暴露出来，随后写入用户镜像，成功部署用户镜像后，物理节点的部署就完成了。

下面我们通过代码来分析Ironic的部署流程。

####Ironic使用PXE部署物理机过程

#####配置

在/etc/nova/nova.conf中修改manager和driver，比如修改成如下：

```
[DEFAULT]  
scheduler_host_manager = nova.scheduler.ironic_host_manager.IronicHostManager  
compute_driver = nova.virt.ironic.driver.IronicDriver  
compute_manager = ironic.nova.compute.manager.ClusteredComputeManager  
[ironic]  
admin_username = ironic  
admin_password = unset  
admin_url = http://127.0.0.1:35357/v2.0  
admin_tenant_name = service  
```

compute_manager的代码实现是在ironic项目里面。 

#####启动流程

**第一步,**
nova-api接收到nova boot的请求，通过消息队列到达nova-scheduler

**第二步,**
nova-scheduler收到请求后，在scheduler_host_manager里面处理。nova-scheduler会使用flavor里面的额外属性extra_specs，像cpu_arch，baremetal:deploy_kernel_id，baremetal:deploy_ramdisk_id等过滤条件找到相匹配的物理节点，然后发送RPC消息到nova-computer。

**第三步,**
nova-computer拿到消息调用指定的driver的spawn方法进行部署，即调用nova.virt.ironic.driver.IronicDriver.spawn()， 该方法做了什么操作呢？我们来对代码进行分析（下面的代码只保留了主要的调用）。

```python
 def spawn(self, context, instance, image_meta, injected_files,  
             admin_password, network_info=None, block_device_info=None):  
        
        #获取镜像信息
        image_meta = objects.ImageMeta.from_dict(image_meta)
 
        ......

        #调用ironic的node.get方法查询node的详细信息,锁定物理机，获取该物理机的套餐信息
        node = self.ironicclient.call("node.get", node_uuid)
        flavor = instance.flavor
        
        #将套餐里面的baremetal:deploy_kernel_id和baremetal:deploy_ramdisk_id信息
        #更新到driver_info，将image_source、root_gb、swap_mb、ephemeral_gb、
        #ephemeral_format、preserve_ephemeral信息更新到instance_info中，
        #然后将driver_info和instance_info更新到ironic的node节点对应的属性上。
        self._add_driver_fields(node, instance, image_meta, flavor)

        .......

        # 验证是否可以部署，只有当deply和power都准备好了才能部署
        validate_chk = self.ironicclient.call("node.validate", node_uuid)
        .....

        # 准备部署
        try:
            #将节点的虚拟网络接口和物理网络接口连接起来并调用ironic API
            #进行更新，以便neutron可以连接
            self._plug_vifs(node, instance, network_info)
            self._start_firewall(instance, network_info)
        except Exception:
            ....

        # 配置驱动
        onfigdrive_value = self._generate_configdrive(
                instance, node, network_info, extra_md=extra_md,
                files=injected_files)


        # 触发部署请求
        try:
            #调用ironic API，设置provision_state的状态ACTIVE
            self.ironicclient.call("node.set_provision_state", node_uuid,
                                   ironic_states.ACTIVE,
                                   configdrive=configdrive_value)
        except Exception as e:
            ....

        #等待node provision_state为ATCTIVE
        timer = loopingcall.FixedIntervalLoopingCall(self._wait_for_active,
                                                     self.ironicclient,
                                                     instance)
        try:
            timer.start(interval=CONF.ironic.api_retry_interval).wait()
        except Exception:
              ...
```
nova-compute的spawn的步骤包括：

1. 获取节点
2. 配置网络信息
2. 配置驱动信息
3. 触发部署，设置ironic的provision_state为ACTIVE
4. 然后等待ironic的node provision_state为ACTIVE就结束了。

**第四步**
ironic-api接收到了provision_state的设置请求，然后返回202的异步请求，那我们下来看下ironic在做什么？

首先，设置ironic node的provision_stat为ACTIVE相当于发了一个POST请求：PUT  /v1/nodes/(node_uuid)/states/provision。那根据openstack的wsgi的框架，注册了app为ironic.api.app.VersionSelectorApplication的类为ironic的消息处理接口，那PUT  /v1/nodes/(node_uuid)/states/provision的消息处理就在ironic.api.controllers.v1.node.NodeStatesController的provision方法。

```python
@expose.expose(None, types.uuid_or_name, wtypes.text,
                   wtypes.text, status_code=http_client.ACCEPTED)
    def provision(self, node_ident, target, configdrive=None):
        ....
        
        if target == ir_states.ACTIVE:
            #RPC调用do_node_deploy方法
            pecan.request.rpcapi.do_node_deploy(pecan.request.context,
                                                rpc_node.uuid, False,
                                                configdrive, topic)
       ...

```

然后RPC调用的ironic.condutor.manager.ConductorManager.do_node_deploy方法,在方法中会先检查电源和部署信息，其中部署信息检查指定的节点的属性是否包含驱动的要求，包括检查boot、镜像大小是否大于内存大小、解析根设备。检查完之后调用ironic.condutor.manager.do_node_deploy方法

```python
   def do_node_deploy(task, conductor_id, configdrive=None):
    """Prepare the environment and deploy a node."""
    node = task.node
    ...
    
    try:
        try:
            if configdrive:
                _store_configdrive(node, configdrive)
        except exception.SwiftOperationError as e:
            with excutils.save_and_reraise_exception():
                handle_failure(
                    e, task,
                    _LE('Error while uploading the configdrive for '
                        '%(node)s to Swift'),
                    _('Failed to upload the configdrive to Swift. '
                      'Error: %s'))

        try:
            #调用驱动的部署模块的prepare方法，不同驱动的动作不一样
            #1. pxe_* 驱动使用的是iscsi_deploy.ISCSIDeploy.prepare，
            #然后调用pxe.PXEBoot.prepare_ramdisk()准备部署进行和环境，包括cache images、 update DHCP、
            #switch pxe_config、set_boot_device等操作
            #cache images 是从glance上取镜像缓存到condutor本地，
            #update DHCP指定bootfile文件地址为condutor
            #switch pxe_config将deploy mode设置成service mode
            #set_boot_device设置节点pxe启动
            #2. agent_*  生成镜像swift_tmp_url加入节点的instance_info中
            #然后调用pxe.PXEBoot.prepare_ramdisk()准备部署镜像和环境
            task.driver.deploy.prepare(task)
        except Exception as e:
            ...

        try:
            #调用驱动的deploy方法，不同驱动动作不一样
            #1. pxe_* 驱动调用iscsi_deploy.ISCSIDeploy.deploy（）
            #进行拉取用户镜像，然后重启物理机
            #2. agent_*驱动，直接重启
            new_state = task.driver.deploy.deploy(task)
        except Exception as e:
            ...

        # NOTE(deva): Some drivers may return states.DEPLOYWAIT
        #             eg. if they are waiting for a callback
        if new_state == states.DEPLOYDONE:
            task.process_event('done')
        elif new_state == states.DEPLOYWAIT:
            task.process_event('wait')
             
    finally:
        node.save()
```

至此，ironic-conductor的动作完成，等待物理机进行上电。

值得说明的是，task是task_manager.TaskManager的一个对象，这个对象在初始化的时候将self.driver初始化了
`self.driver = driver_factory.get_driver(driver_name or  self.node.driver) `
driver_name是传入的参数，默认为空；这个self.node.driver指物理机使用的驱动，不同物理机使用的驱动可能不同，这是在注册物理机时指定的。


**第五步**

在上一步中已经设置好了启动方式和相关网络信和给机器上电了，那么下一步就是机器启动，进行部署了。下面以PXE和agent两种部署方式分别来说明。

####使用PXE部署
我们知道安装操作系统的通用流程是：首先，bios启动，选择操作系统的启动（安装）模式（此时，内存是空白的），然后根据相关的安装模式，寻找操作系统的引导程序（不同的模式，对应不同的引导程序当然也对应着不同的引导程序存在的位置），引导程序加载文件系统初始化程序（initrd）和内核初始镜像（vmlinuz），完成操作系统安装前的初始化；接着，操作系统开始安装相关的系统和应用程序。

PXE启动方式的过程为：

1. 物理机上电后，BIOS把PXE client调入内存执行，客户端广播DHCP请求
2. DHCP服务器（neutron）给客户机分配IP并给定bootstrap文件的放置位置
3. 客户机向本网络中的TFTP服务器索取bootstrap文件
4. 客户机取得bootstrap文件后之执行该文件
5. 根据bootstrap的执行结果，通过TFTP服务器(conductor)加载内核和文件系统
6. 在内存中启动安装

启动后运行init启动脚本，那么init启动脚本是什么样子的。

首先，我们需要知道当前创建deploy-ironic的镜像，使用的diskimage-build命令，参考[diskimage-builder/elements/deploy-ironic](https://github.com/openstack/diskimage-builder/tree/master/elements/deploy-ironic)这个元素，最重要的是[init.d/80-deploy-ironic](https://github.com/openstack/diskimage-builder/blob/master/elements/deploy-ironic/init.d/80-deploy-ironic)这个脚本，这个脚本主要其实就是做以下几个步骤：

1. 找到磁盘，以该磁盘启动iSCSI设备
2. Tftp获取到ironic准备的token文件
3. 调用ironic的api接(POST v1/nodes/{node-id}/vendor_passthru/pass_deploy_info)
4. 启动iSCSI设备, 开启socket端口 10000等待通知PXE结束
5. 结束口停止iSCSI设备。

```sh

# 安装bootloader
function install_bootloader {
   #此处省略很多
   ...
}


#向Ironic Condutor发送消息，开启socket端口10000等待通知PXE结束
function do_vendor_passthru_and_wait {

    local data=$1
    local vendor_passthru_name=$2

    eval curl -i -X POST \
         "$TOKEN_HEADER" \
         "-H 'Accept: application/json'" \
         "-H 'Content-Type: application/json'" \
         -d "$data" \
         "$IRONIC_API_URL/v1/nodes/$DEPLOYMENT_ID/vendor_passthru/$vendor_passthru_name"

    echo "Waiting for notice of complete"
    nc -l -p 10000
}


readonly IRONIC_API_URL=$(get_kernel_parameter ironic_api_url)
readonly IRONIC_BOOT_OPTION=$(get_kernel_parameter boot_option)
readonly IRONIC_BOOT_MODE=$(get_kernel_parameter boot_mode)
readonly ROOT_DEVICE=$(get_kernel_parameter root_device)

if [ -z "$ISCSI_TARGET_IQN" ]; then
  err_msg "iscsi_target_iqn is not defined"
  troubleshoot
fi

#获取当前linux的本地硬盘  
target_disk=
if [[ $ROOT_DEVICE ]]; then
    target_disk="$(get_root_device)"
else
    t=0
    while ! target_disk=$(find_disk "$DISK"); do   
      if [ $t -eq 60 ]; then
        break
      fi
      t=$(($t + 1))
      sleep 1
    done
fi

if [ -z "$target_disk" ]; then
  err_msg "Could not find disk to use."
  troubleshoot
fi

#将找到的本地磁盘作为iSCSI磁盘启动，暴露给Ironic Condutor
echo "start iSCSI target on $target_disk"
start_iscsi_target "$ISCSI_TARGET_IQN" "$target_disk" ALL   
if [ $? -ne 0 ]; then
  err_msg "Failed to start iscsi target."
  troubleshoot
fi

#获取到相关的token文件，从tftp服务器上获取，token文件在ironic在prepare阶段就生成好的。  
if [ "$BOOT_METHOD" = "$VMEDIA_BOOT_TAG" ]; then
  TOKEN_FILE="$VMEDIA_DIR/token"
  if [ -f "$TOKEN_FILE" ]; then
    TOKEN_HEADER="-H 'X-Auth-Token: $(cat $TOKEN_FILE)'"
  else TOKEN_HEADER=""
  fi
else
  TOKEN_FILE=token-$DEPLOYMENT_ID

  # Allow multiple versions of the tftp client
  if tftp -r $TOKEN_FILE -g $BOOT_SERVER || tftp $BOOT_SERVER -c get $TOKEN_FILE; then
      TOKEN_HEADER="-H 'X-Auth-Token: $(cat $TOKEN_FILE)'"
  else
      TOKEN_HEADER=""
  fi
fi


#向Ironic请求部署镜像，POST node的/vendor_passthru/pass_deploy_info请求
echo "Requesting Ironic API to deploy image"
deploy_data="'{\"address\":\"$BOOT_IP_ADDRESS\",\"key\":\"$DEPLOYMENT_KEY\",\"iqn\":\"$ISCSI_TARGET_IQN\",\"error\":\"$FIRST_ERR_MSG\"}'"
do_vendor_passthru_and_wait "$deploy_data" "pass_deploy_info"

#部署镜像下载结束，停止iSCSI设备
echo "Stopping iSCSI target on $target_disk"
stop_iscsi_target

#如果是本地启动，安装bootloarder
# If localboot is set, install a bootloader
if [ "$IRONIC_BOOT_OPTION" = "local" ]; then
    echo "Installing bootloader"

    error_msg=$(install_bootloader)
    if [ $? -eq 0 ]; then
        status=SUCCEEDED
    else
        status=FAILED
    fi

    echo "Requesting Ironic API to complete the deploy"
 bootloader_install_data="'{\"address\":\"$BOOT_IP_ADDRESS\",\"status\":\"$status\",\"key\":\"$DEPLOYMENT_KEY\",\"error\":\"$error_msg\"}'"
    do_vendor_passthru_and_wait "$bootloader_install_data" "pass_bootloader_install_info"
fi
```

下面我们来看一下node的/vendor_passthru/pass_deploy_info都干了什么？Ironic-api在接受到请求后，是在ironic.api.controllers.v1.node.NodeVendorPassthruController._default()方法处理的，这个方法将调用的方法转发到ironic.condutor.manager.CondutorManager.vendor_passthro()去处理,进而调用相应task.driver.vendor.pass_deploy_info()去处理，这里不同驱动不一样，可以根据源码查看到，比如使用pxe_ipmptoos驱动, 则是转发给ironic.drivers.modules.iscsi_deploy.VendorPassthru.pass_deploy_info()处理，其代码是

```python
@base.passthru(['POST'])
    @task_manager.require_exclusive_lock
    def pass_deploy_info(self, task, **kwargs):
        """Continues the deployment of baremetal node over iSCSI.

        This method continues the deployment of the baremetal node over iSCSI
        from where the deployment ramdisk has left off.

        :param task: a TaskManager instance containing the node to act on.
        :param kwargs: kwargs for performing iscsi deployment.
        :raises: InvalidState
        """
        node = task.node
        LOG.warning(_LW("The node %s is using the bash deploy ramdisk for "
                        "its deployment. This deploy ramdisk has been "
                        "deprecated. Please use the ironic-python-agent "
                        "(IPA) ramdisk instead."), node.uuid)
        task.process_event('resume')   #设置任务状态
        LOG.debug('Continuing the deployment on node %s', node.uuid)

        is_whole_disk_image = node.driver_internal_info['is_whole_disk_image']
        
        #继续部署的函数，连接到iSCSI设备，将用户镜像写到iSCSI设备上，退出删除iSCSI设备，
        #然后在Condutor上删除镜像文件
        uuid_dict_returned = continue_deploy(task, **kwargs)
        
        root_uuid_or_disk_id = uuid_dict_returned.get(
            'root uuid', uuid_dict_returned.get('disk identifier'))

        # save the node's root disk UUID so that another conductor could
        # rebuild the PXE config file. Due to a shortcoming in Nova objects,
        # we have to assign to node.driver_internal_info so the node knows it
        # has changed.
        driver_internal_info = node.driver_internal_info
        driver_internal_info['root_uuid_or_disk_id'] = root_uuid_or_disk_id
        node.driver_internal_info = driver_internal_info
        node.save()

        try:
            #再一次设置PXE引导，为准备进入用户系统做准备
            task.driver.boot.prepare_instance(task)

            if deploy_utils.get_boot_option(node) == "local":
                if not is_whole_disk_image:
                    LOG.debug('Installing the bootloader on node %s',
                              node.uuid)
                    deploy_utils.notify_ramdisk_to_proceed(kwargs['address'])
                    task.process_event('wait')
                    return

        except Exception as e:
            LOG.error(_LE('Deploy failed for instance %(instance)s. '
                          'Error: %(error)s'),
                      {'instance': node.instance_uuid, 'error': e})
            msg = _('Failed to continue iSCSI deployment.')
            deploy_utils.set_failed_state(task, msg)
        else:
            #结束部署，通知ramdisk重启，将物理机设置为ative
            finish_deploy(task, kwargs.get('address'))
```

在continue_deploy函数中，先解析iscsi部署的信息，然后在进行分区、格式化、写入镜像到磁盘。
然后调用prepare_instance在设置一遍PXE环境，为进入系统做准备，我们知道在instance_info上设置了ramdisk、kernel、image_source 3个镜像，其实就是内核、根文件系统、磁盘镜像。这里就是设置了ramdisk和kernel，磁盘镜像上面已经写到磁盘中去了，调用switch_pxe_config方法将当前的操作系统的启动项设置为ramdisk和kernel作为引导程序。
最后向节点的10000发送一个‘done’通知节点关闭iSCSI设备，最后节点重启安装用户操作系统，至此部署结束。

在部署过程中，节点和驱动的信息都会被存入ironic数据库，以便后续管理。

####使用agent部署过程

在部署阶段的prepare阶段与PXE一样，但是由于创建的ramdisk不一样所以部署方式则不一样，在PXE中，开机执行的是一段init脚本，而在Agent开机执行的是IPA。

机器上电后，ramdisk在内存中执行，然后启动IPA，入口为cmd.agent.run()，然后调用ironic-python-agent.agent.run()，其代码如下

```python
def run(self):
        """Run the Ironic Python Agent."""
        # Get the UUID so we can heartbeat to Ironic. Raises LookupNodeError
        # if there is an issue (uncaught, restart agent)
        self.started_at = _time()

        #加载hardware manager
        # Cached hw managers at runtime, not load time. See bug 1490008.
        hardware.load_managers()

        if not self.standalone:
            # Inspection should be started before call to lookup, otherwise
            # lookup will fail due to unknown MAC.
            uuid = inspector.inspect()
            
            #利用Ironic API给Condutor发送lookup()请求，用户获取UUID，相当于自发现
            content = self.api_client.lookup_node(
                hardware_info=hardware.dispatch_to_managers(
                                  'list_hardware_info'),
                timeout=self.lookup_timeout,
                starting_interval=self.lookup_interval,
                node_uuid=uuid)

            self.node = content['node']
            self.heartbeat_timeout = content['heartbeat_timeout']

        wsgi = simple_server.make_server(
            self.listen_address[0],
            self.listen_address[1],
            self.api,
            server_class=simple_server.WSGIServer)

        #发送心跳包
        if not self.standalone:
            # Don't start heartbeating until the server is listening          
            self.heartbeater.start()

        try:
            wsgi.serve_forever()
        except BaseException:
            self.log.exception('shutting down')
        #部署完成后停止心跳包
        if not self.standalone:
            self.heartbeater.stop()
```
其中self.api_client.lookup_node调用到ironic-python-api._do_lookup()，然后发送一个GET /{api_version}/drivers/{driver}/vendor_passthru/lookup请求。
Condutor API在接受到lookup请求后调用指定驱动的lookup函数处理，返回节点UUID。

IPA收到UUID后调用Ironic-API发送Heartbeat请求（/{api_version}/nodes/{uuid}/vendor_passthru/heartbeat）,Ironic-API把消息路由给节点的驱动heartbeat函数处理。Ironic-Condutor周期执行该函数，每隔一段时间执行该函数检查IPA部署是否完成，如果完成则进入之后的动作.目前agent*驱动使用的是ironic.drivers.modouls.agent.AgentVendorInterface类实现的接口，代码如下。

```python
@base.passthru(['POST'])
    def heartbeat(self, task, **kwargs):
        """Method for agent to periodically check in.

        The agent should be sending its agent_url (so Ironic can talk back)
        as a kwarg. kwargs should have the following format::

         {
             'agent_url': 'http://AGENT_HOST:AGENT_PORT'
         }

        AGENT_PORT defaults to 9999.
        """
        node = task.node
        driver_internal_info = node.driver_internal_info
        LOG.debug(
            'Heartbeat from %(node)s, last heartbeat at %(heartbeat)s.',
            {'node': node.uuid,
             'heartbeat': driver_internal_info.get('agent_last_heartbeat')})
        driver_internal_info['agent_last_heartbeat'] = int(_time())
        try:
            driver_internal_info['agent_url'] = kwargs['agent_url']
        except KeyError:
            raise exception.MissingParameterValue(_('For heartbeat operation, '
                                                    '"agent_url" must be '
                                                    'specified.'))

        node.driver_internal_info = driver_internal_info
        node.save()

        # Async call backs don't set error state on their own
        # TODO(jimrollenhagen) improve error messages here
        msg = _('Failed checking if deploy is done.')
        try:
            if node.maintenance:
                # this shouldn't happen often, but skip the rest if it does.
                LOG.debug('Heartbeat from node %(node)s in maintenance mode; '
                          'not taking any action.', {'node': node.uuid})
                return
            elif (node.provision_state == states.DEPLOYWAIT and
                  not self.deploy_has_started(task)):
                msg = _('Node failed to get image for deploy.')
                self.continue_deploy(task, **kwargs)         #调用continue_deploy函数，下载镜像
            elif (node.provision_state == states.DEPLOYWAIT and
                  self.deploy_is_done(task)):             #查看IPA执行下载镜像是否结束
                msg = _('Node failed to move to active state.')
                self.reboot_to_instance(task, **kwargs)       #如果镜像已经下载完成，即部署完成，设置从disk启动，重启进入用户系统，
            elif (node.provision_state == states.DEPLOYWAIT and
                  self.deploy_has_started(task)):
                node.touch_provisioning()             #更新数据库，将节点的设置为alive
            # TODO(lucasagomes): CLEANING here for backwards compat
            # with previous code, otherwise nodes in CLEANING when this
            # is deployed would fail. Should be removed once the Mitaka
            # release starts.
            elif node.provision_state in (states.CLEANWAIT, states.CLEANING):
                node.touch_provisioning()
                if not node.clean_step:
                    LOG.debug('Node %s just booted to start cleaning.',
                              node.uuid)
                    msg = _('Node failed to start the next cleaning step.')
                    manager.set_node_cleaning_steps(task)
                    self._notify_conductor_resume_clean(task)
                else:
                    msg = _('Node failed to check cleaning progress.')
                    self.continue_cleaning(task, **kwargs)

        except Exception as e:
            err_info = {'node': node.uuid, 'msg': msg, 'e': e}
            last_error = _('Asynchronous exception for node %(node)s: '
                           '%(msg)s exception: %(e)s') % err_info
            LOG.exception(last_error)
            if node.provision_state in (states.CLEANING, states.CLEANWAIT):
                manager.cleaning_error_handler(task, last_error)
            elif node.provision_state in (states.DEPLOYING, states.DEPLOYWAIT):
                deploy_utils.set_failed_state(task, last_error)
```

根据上面bearthead函数，首先根据当前节点的状态node.provision_state==DEPLOYWAIT，调用continue_deploy()函数进行部署.

```python
@task_manager.require_exclusive_lock
    def continue_deploy(self, task, **kwargs):
        task.process_event('resume')
        node = task.node
        image_source = node.instance_info.get('image_source')
        LOG.debug('Continuing deploy for node %(node)s with image %(img)s',
                  {'node': node.uuid, 'img': image_source})

        image_info = {
            'id': image_source.split('/')[-1],
            'urls': [node.instance_info['image_url']],
            'checksum': node.instance_info['image_checksum'],
            # NOTE(comstud): Older versions of ironic do not set
            # 'disk_format' nor 'container_format', so we use .get()
            # to maintain backwards compatibility in case code was
            # upgraded in the middle of a build request.
            'disk_format': node.instance_info.get('image_disk_format'),
            'container_format': node.instance_info.get(
                'image_container_format')
        }
        
        #通知IPA下载swift上的镜像，并写入本地磁盘
        # Tell the client to download and write the image with the given args
        self._client.prepare_image(node, image_info)

        task.process_event('wait')
```
Condutor然后依次调用

1. deploy_is_done()检查IPA执行下载镜像是否结束，
2. 如果镜像已经下载完成，即部署完成，设置从disk启动，重启进入用户系统reboot_to_instance()
3. 然后调用node.touch_provisioning() 更新数据库，将节点的设置为alive

至此，使用agent方式进行部署操作系统的过程到处结束。下面我们用两张图来回顾一下部署过程：


1. 使用pxe_* 为前缀的驱动的部署过程


![pxe_deploy_process]({{IMAGE_PATH}}/binli/ironic-presentation/pxe_deploy_process2.png)

2. 使用agent_* 为前缀的驱动的部署过程


![agent_deploy_process]({{IMAGE_PATH}}/binli/ironic-presentation/agent_delopy_process2.png)


###Ironic状态转换图
下图是Liberty版的状态装换图

![Ironic’s State Machine]({{IMAGE_PATH}}/binli/ironic-presentation/states.svg)



-----------------------------------------

## 总结

Ironic 是一个管理物理机的服务，将管理物理机像管理虚拟机一样，能够进行物理机的添加，删除，电源管理和安装部署。Ironic使用不同的驱动来支持不同的硬件，厂商能够实现自己的驱动以提供不同的功能。不同的驱动有不同的部署方式，分为pxe和agent两种。Ironic 是一个管理物理机的服务，将管理物理机像管理虚拟机一样，能够进行物理机的添加，删除，电源管理和安装部署。Ironic使用不同的驱动来支持不同的硬件，厂商能够实现自己的驱动以提供不同的功能。不同的驱动有不同的部署方式，分为pxe和agent两种。