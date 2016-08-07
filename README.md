# devstack_network_flat
---------------------------------------------------------


**本文主要介绍在devstack下，以flat模式配置网络，并创建实例的方法。**


## 1：背景知识：关于openstack网络简单介绍：


Openstack中提供可以网络服务的有2个模块：nova-network，Neutron。

	  1：目前在devstack中，默认使用nova-network来提供网络服务。
	  
	  2：但是在openstack中，Neutron项目 被用于取代nova-network来提供网络服务。

---------------这2种网络的差异可以参考文章：[http://blog.csdn.net/beginning1126/article/details/41172365](http://blog.csdn.net/beginning1126/article/details/41172365 "openstack 网络架构 nova-network + neutron")


nova-network实现了三种网络模型,简介如下：

	1:flat: 在flat模式下，管理员需要手动创建一个网桥，所有的虚拟机都会关联到该网桥，所有的虚拟机也都处于同一个子网下，虚拟机的fixed ip都是从该子网分配，而且网络相关的配置信息会在虚拟机启动时注入到虚拟机镜像中。
	
	2:flatdhcp: flatdhcp模式与flat比较接近，但nova会自动创建网桥，维护已经分配的floating ip，并启动dnsmasq来配置虚拟机的fixed ip，创建虚拟机时，nova会给虚拟机分配一个fixed ip，并将MAC/IP的对应关系写入dnsmasq的配置文件中，虚拟机启动时通过DHCP获取其fixed ip，因此就不需要将网络配置信息注入到虚拟机中了。
	
	3:vlan: 模式比上面两种模式复杂，每个project都会分配一个vlan id，每个project也可以有自己的独立的ip地址段，属于不同project的虚拟机连接到不同的网桥上，因此不同的project之间是隔离的，不会相互影响。为了访问一个project的所有虚拟机需要创建一个vpn虚拟机，以此虚拟机作为跳板去访问该project的其他虚拟机。与flatdhcp类似，vlan模式下，也会为每个project启动一个dnsmasq来配置虚拟机的fixed ip。

--------------更详细的介绍可以参考文章：[http://www.cnblogs.com/yuxc/p/3426463.html](http://www.cnblogs.com/yuxc/p/3426463.html "nova network工作原理及配置")



## 2 在devstack以flat模式搭建网络

### 2.1 搭建前网络环境，配置准备
以下操作步骤均在vmave下创建的虚拟机（centos 7）下完成，centos 7网络环境如下，

		[ollylu@MiWiFi-R1CL]$ ifconfig
		eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
		        inet 192.168.31.194  netmask 255.255.255.0  broadcast 192.168.31.255
		        inet6 fe80::20c:29ff:fe42:51a7  prefixlen 64  scopeid 0x20<link>
		        ether 00:0c:29:42:51:a7  txqueuelen 1000  (Ethernet)
		        RX packets 8529  bytes 2769338 (2.6 MiB)
		        RX errors 0  dropped 0  overruns 0  frame 0
		        TX packets 13628  bytes 3554090 (3.3 MiB)
		        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
		
		lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
		        inet 127.0.0.1  netmask 255.0.0.0
		        inet6 ::1  prefixlen 128  scopeid 0x10<host>
		        loop  txqueuelen 0  (Local Loopback)
		        RX packets 79396  bytes 60684757 (57.8 MiB)
		        RX errors 0  dropped 0  overruns 0  frame 0
		        TX packets 79396  bytes 60684757 (57.8 MiB)
		        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
		
		virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
		        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
		        ether 52:54:00:b0:01:f6  txqueuelen 0  (Ethernet)
		        RX packets 0  bytes 0 (0.0 B)
		        RX errors 0  dropped 0  overruns 0  frame 0
		        TX packets 0  bytes 0 (0.0 B)
		        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

首先为虚拟机添加一个新的网卡（在vmave的设置中对虚拟机添加），仅主机模式，与主机共享专用网络。

	
	[ollylu@MiWiFi-R1CL]$ ifconfig eno33554984
	eno33554984: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet6 fe80::20c:29ff:fe42:51b1  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:b1  txqueuelen 1000  (Ethernet)
	        RX packets 3525  bytes 1528314 (1.4 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 34  bytes 2376 (2.3 KiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0



并新创建一个网桥，将网卡eno33554984，桥接到网桥下：

	[ollylu@MiWiFi-R1CL]$ sudo ifconfig eno33554984 0.0.0.0  
	[ollylu@MiWiFi-R1CL]$ sudo brctl addbr br200  
	[ollylu@MiWiFi-R1CL]$ sudo brctl addif br200 eno33554984  
	[ollylu@MiWiFi-R1CL]$ sudo ifconfig br200 10.10.10.1 netmask 255.255.255.0

并修改网卡eno33554984配置文件，如下：

	[ollylu@MiWiFi-R1CL]$ cat /etc/sysconfig/network-scripts/ifcfg-eno33554984
	NAME=eno33554984
	DEVICE=eno33554984
	BOOTPROTO=none
	ONBOOT=yes
	BRIDGE=br200
	NM_CONTROLLED=no

然后重启网络  `service network restart`

现在网络配置应当如下：

	[ollylu@MiWiFi-R1CL]$ ifconfig
	br200: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet 10.10.10.1  netmask 255.255.255.0  broadcast 10.10.10.255
	        inet6 fe80::20c:29ff:fe42:51b1  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:b1  txqueuelen 0  (Ethernet)
	        RX packets 21  bytes 2982 (2.9 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 9  bytes 690 (690.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet 192.168.31.194  netmask 255.255.255.0  broadcast 192.168.31.255
	        inet6 fe80::20c:29ff:fe42:51a7  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:a7  txqueuelen 1000  (Ethernet)
	        RX packets 8529  bytes 2769338 (2.6 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 13628  bytes 3554090 (3.3 MiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	eno33554984: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet6 fe80::20c:29ff:fe42:51b1  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:b1  txqueuelen 1000  (Ethernet)
	        RX packets 21  bytes 3276 (3.1 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 16  bytes 1296 (1.2 KiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
	        inet 127.0.0.1  netmask 255.0.0.0
	        inet6 ::1  prefixlen 128  scopeid 0x10<host>
	        loop  txqueuelen 0  (Local Loopback)
	        RX packets 79396  bytes 60684757 (57.8 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 79396  bytes 60684757 (57.8 MiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
	        ether 52:54:00:b0:01:f6  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

	[ollylu@MiWiFi-R1CL]$ brctl show
	bridge name     bridge id               STP enabled     interfaces
	br200           8000.000c294251b1       no              eno33554984
	                                                        vnet0
	virbr0          8000.525400b001f6       yes             virbr0-nic


### 2.2 devstack虚拟网络配置

在执行stack.sh前，先在local.conf 的 `[[local|localrc]]`添加如下配置：


	ADMIN_PASSWORD=ollylu
	DATABASE_PASSWORD=ollylu
	RABBIT_PASSWORD=ollylu
	SERVICE_PASSWORD=$ADMIN_PASSWORD
	SERVICE_TOKEN=azertytokenzhf


	FLOATING_RANGE=192.168.1.224/27
	FIXED_RANGE=10.10.10.0/24
	FIXED_NETWORK_SIZE=256
	FLAT_INTERFACE=eno33554984   #新网卡
	NETWORK_MANAGER=FlatManager  #网络模式
	PUBLIC_INTERFACE=br200       #新创建的网桥
	VLAN_INTERFACE=eno33554984   #新网卡
	FLAT_NETWORK_BRIDGE=br200    #新创建的网桥


然后执行`stack.sh`，

### 2.3 创建实例

stack.sh执行完毕后,执行`. openrc admin demo`---在demo项目下以admin进行配置

查看nova目前运行的虚拟机，刚开始为空，如下：

	[ollylu@MiWiFi-R1CL]$ nova list
	+----+------+--------+------------+-------------+----------+
	| ID | Name | Status | Task State | Power State | Networks |
	+----+------+--------+------------+-------------+----------+
	+----+------+--------+------------+-------------+----------+

检查可选虚拟机型号：

	[ollylu@MiWiFi-R1CL]$ nova flavor-list
	+----+-----------+-----------+------+-----------+------+-------+-------------+--                                                                                        ---------+
	| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | I                                                                                        s_Public |
	+----+-----------+-----------+------+-----------+------+-------+-------------+--                                                                                        ---------+
	| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | T                                                                                        rue      |
	| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | T                                                                                        rue      |
	| 42 | m1.nano   | 64        | 0    | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | T                                                                                        rue      |
	| 84 | m1.micro  | 128       | 0    | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| c1 | cirros256 | 256       | 0    | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| d1 | ds512M    | 512       | 5    | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| d2 | ds1G      | 1024      | 10   | 0         |      | 1     | 1.0         | T                                                                                        rue      |
	| d3 | ds2G      | 2048      | 10   | 0         |      | 2     | 1.0         | T                                                                                        rue      |
	| d4 | ds4G      | 4096      | 20   | 0         |      | 4     | 1.0         | T                                                                                        rue      |
	+----+-----------+-----------+------+-----------+------+-------+-------------+--                                                                                        ---------+

检查可以使用的镜像类型：

	[ollylu@MiWiFi-R1CL]$ nova image-list
	WARNING: Command image-list is deprecated and will be removed after Nova 15.0.0                                                                                         is released. Use python-glanceclient or openstackclient instead.
	+--------------------------------------+---------------------------------+------                                                                                        --+--------+
	| ID                                   | Name                            | Statu                                                                                        s | Server |
	+--------------------------------------+---------------------------------+------                                                                                        --+--------+
	| 5601bcbf-74d7-41ea-a3f4-d67d0cfc98da | cirros-0.3.4-x86_64-uec         | ACTIV                                                                                        E |        |
	| 34e72082-c128-4438-be2d-640e2f89bc76 | cirros-0.3.4-x86_64-uec-kernel  | ACTIV                                                                                        E |        |
	| 2f022c73-0470-4b4e-9982-522a281e7c87 | cirros-0.3.4-x86_64-uec-ramdisk | ACTIV                                                                                        E |        |
	+--------------------------------------+---------------------------------+------                                                                                        --+--------+

创建密钥认证：

	[ollylu@MiWiFi-R1CL]$ nova keypair-list
	+------+------+-------------+
	| Name | Type | Fingerprint |
	+------+------+-------------+
	+------+------+-------------+
	
	[ollylu@MiWiFi-R1CL]$ nova keypair-add vmkey >vmkey.pem
	
	[ollylu@MiWiFi-R1CL]$ nova keypair-list
	+-------+------+-------------------------------------------------+
	| Name  | Type | Fingerprint                                     |
	+-------+------+-------------------------------------------------+
	| vmkey | ssh  | b7:e4:cf:86:af:6f:b8:93:53:60:bf:81:2d:7e:67:5e |
	+-------+------+-------------------------------------------------+


删除默认的网段，在网桥br200上创建虚拟机所用的子网 vmnet：10.10.0.0/16

	[ollylu@MiWiFi-R1CL]$ nova network-delete private
	
	[ollylu@MiWiFi-R1CL]$ nova network-list
	+----+-------+------+
	| ID | Label | Cidr |
	+----+-------+------+
	+----+-------+------+
	~/openstack/devstack
	[ollylu@MiWiFi-R1CL]$ nova network-create vmnet --fixed-range-v4 10.10.0.0/16 --                                                                                        fixed-cidr 10.10.10.0/24 --bridge br200
	
	[ollylu@MiWiFi-R1CL]$ nova network-list
	+--------------------------------------+-------+--------------+
	| ID                                   | Label | Cidr         |
	+--------------------------------------+-------+--------------+
	| 277489d2-efa5-4894-aa9e-557bf760d6d9 | vmnet | 10.10.0.0/16 |
	+--------------------------------------+-------+--------------+

查看vmnet的配置详细：

	[ollylu@MiWiFi-R1CL]$ nova network-show vmnet
	+---------------------+--------------------------------------+
	| Property            | Value                                |
	+---------------------+--------------------------------------+
	| bridge              | br200                                |
	| bridge_interface    | eno33554984                          |
	| broadcast           | 10.10.255.255                        |
	| cidr                | 10.10.0.0/16                         |
	| cidr_v6             | -                                    |
	| created_at          | 2016-08-07T08:42:56.000000           |
	| deleted             | False                                |
	| deleted_at          | -                                    |
	| dhcp_server         | 10.10.0.1                            |
	| dhcp_start          | 10.10.0.2                            |
	| dns1                | 8.8.4.4                              |
	| dns2                | -                                    |
	| enable_dhcp         | True                                 |
	| gateway             | 10.10.0.1                            |
	| gateway_v6          | -                                    |
	| host                | -                                    |
	| id                  | 277489d2-efa5-4894-aa9e-557bf760d6d9 |
	| injected            | False                                |
	| label               | vmnet                                |
	| mtu                 | -                                    |
	| multi_host          | False                                |
	| netmask             | 255.255.0.0                          |
	| netmask_v6          | -                                    |
	| priority            | -                                    |
	| project_id          | -                                    |
	| rxtx_base           | -                                    |
	| share_address       | False                                |
	| updated_at          | -                                    |
	| vlan                | -                                    |
	| vpn_private_address | -                                    |
	| vpn_public_address  | -                                    |
	| vpn_public_port     | -                                    |
	+---------------------+--------------------------------------+
	

创建新实例，虚拟机

	[ollylu@MiWiFi-R1CL]$ nova boot --flavor 1 --image cirros-0.3.4-x86_64-uec-kerne                                                                                        l --security-groups default --key-name vmkey cirrvm
	+--------------------------------------+----------------------------------------                                                                                        -------------------------------+
	| Property                             | Value                                                                                                                                                         |
	+--------------------------------------+----------------------------------------                                                                                        -------------------------------+
	| OS-DCF:diskConfig                    | MANUAL                                                                                                                                                        |
	| OS-EXT-AZ:availability_zone          |                                                                                                                                                               |
	| OS-EXT-SRV-ATTR:host                 | -                                                                                                                                                             |
	| OS-EXT-SRV-ATTR:hostname             | cirrvm                                                                                                                                                        |
	| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                                                                                                                             |
	| OS-EXT-SRV-ATTR:instance_name        | instance-00000001                                                                                                                                             |
	| OS-EXT-SRV-ATTR:kernel_id            |                                                                                                                                                               |
	| OS-EXT-SRV-ATTR:launch_index         | 0                                                                                                                                                             |
	| OS-EXT-SRV-ATTR:ramdisk_id           |                                                                                                                                                               |
	| OS-EXT-SRV-ATTR:reservation_id       | r-z7m3q3p8                                                                                                                                                    |
	| OS-EXT-SRV-ATTR:root_device_name     | -                                                                                                                                                             |
	| OS-EXT-SRV-ATTR:user_data            | -                                                                                                                                                             |
	| OS-EXT-STS:power_state               | 0                                                                                                                                                             |
	| OS-EXT-STS:task_state                | scheduling                                                                                                                                                    |
	| OS-EXT-STS:vm_state                  | building                                                                                                                                                      |
	| OS-SRV-USG:launched_at               | -                                                                                                                                                             |
	| OS-SRV-USG:terminated_at             | -                                                                                                                                                             |
	| accessIPv4                           |                                                                                                                                                               |
	| accessIPv6                           |                                                                                                                                                               |
	| adminPass                            | 3a8wr5UDQFjn                                                                                                                                                  |
	| config_drive                         |                                                                                                                                                               |
	| created                              | 2016-08-07T08:44:30Z                                                                                                                                          |
	| description                          | -                                                                                                                                                             |
	| flavor                               | m1.tiny (1)                                                                                                                                                   |
	| hostId                               |                                                                                                                                                               |
	| host_status                          |                                                                                                                                                               |
	| id                                   | 96eaf718-dc3e-4f10-a89c-590503ad91d3                                                                                                                          |
	| image                                | cirros-0.3.4-x86_64-uec-kernel (34e7208                                                                                        2-c128-4438-be2d-640e2f89bc76) |
	| key_name                             | vmkey                                                                                                                                                         |
	| locked                               | False                                                                                                                                                         |
	| metadata                             | {}                                                                                                                                                            |
	| name                                 | cirrvm                                                                                                                                                        |
	| os-extended-volumes:volumes_attached | []                                                                                                                                                            |
	| progress                             | 0                                                                                                                                                             |
	| security_groups                      | default                                                                                                                                                       |
	| status                               | BUILD                                                                                                                                                         |
	| tags                                 | []                                                                                                                                                            |
	| tenant_id                            | 4d527552346f413796c8d624e578dbe2                                                                                                                              |
	| updated                              | 2016-08-07T08:44:30Z                                                                                                                                          |
	| user_id                              | 54bb99beff5e4247898d33b824f5fe3e                                                                                                                              |
	+--------------------------------------+----------------------------------------                                                                                        -------------------------------+

检查新实例，虚拟机运行状态
	
	[ollylu@MiWiFi-R1CL]$ nova list
	+--------------------------------------+--------+--------+------------+---------                                                                                        ----+------------------+
	| ID                                   | Name   | Status | Task State | Power St                                                                                        ate | Networks         |
	+--------------------------------------+--------+--------+------------+---------                                                                                        ----+------------------+
	| 96eaf718-dc3e-4f10-a89c-590503ad91d3 | cirrvm | ACTIVE | -          | Running                                                                                             | vmnet=10.10.10.2 |
	+--------------------------------------+--------+--------+------------+---------                                                                                        ----+------------------+

	[ollylu@MiWiFi-R1CL]$ nova list
	+--------------------------------------+--------+--------+------------+-------------+------------------+
	| ID                                   | Name   | Status | Task State | Power State | Networks         |
	+--------------------------------------+--------+--------+------------+-------------+------------------+
	| 96eaf718-dc3e-4f10-a89c-590503ad91d3 | cirrvm | ACTIVE | -          | Running     | vmnet=10.10.10.2 |
	+--------------------------------------+--------+--------+------------+-------------+------------------+

新实例，虚拟机运行后网络环境如下，注意新创建的vnet0，且vnet0网桥
br200下

	[ollylu@MiWiFi-R1CL]$ ifconfig
	br200: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet 10.10.10.1  netmask 255.255.255.0  broadcast 10.10.10.255
	        inet6 fe80::20c:29ff:fe42:51b1  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:b1  txqueuelen 0  (Ethernet)
	        RX packets 21  bytes 2982 (2.9 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 9  bytes 690 (690.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet 192.168.31.194  netmask 255.255.255.0  broadcast 192.168.31.255
	        inet6 fe80::20c:29ff:fe42:51a7  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:a7  txqueuelen 1000  (Ethernet)
	        RX packets 8529  bytes 2769338 (2.6 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 13628  bytes 3554090 (3.3 MiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	eno33554984: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet6 fe80::20c:29ff:fe42:51b1  prefixlen 64  scopeid 0x20<link>
	        ether 00:0c:29:42:51:b1  txqueuelen 1000  (Ethernet)
	        RX packets 21  bytes 3276 (3.1 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 16  bytes 1296 (1.2 KiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
	        inet 127.0.0.1  netmask 255.0.0.0
	        inet6 ::1  prefixlen 128  scopeid 0x10<host>
	        loop  txqueuelen 0  (Local Loopback)
	        RX packets 79396  bytes 60684757 (57.8 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 79396  bytes 60684757 (57.8 MiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
	        ether 52:54:00:b0:01:f6  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	vnet0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet6 fe80::fc16:3eff:fe7f:b69b  prefixlen 64  scopeid 0x20<link>
	        ether fe:16:3e:7f:b6:9b  txqueuelen 500  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	

	[ollylu@MiWiFi-R1CL]$ brctl show
	bridge name     bridge id               STP enabled     interfaces
	br200           8000.000c294251b1       no              eno33554984
	                                                        vnet0
	virbr0          8000.525400b001f6       yes             virbr0-nic



最后登录http://192.168.31.194，通过控制台进入vm，设置IP。


	
















