# 命令行实现openstack裸机服务部署

## 1. 创建镜像

###1.1 创建kernel镜像

```shell
glance image-create --name kernel-01 --visibility public --disk-format aki --container-format aki< ironic-agent.kernel
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\创建镜像.png)

### 1.2 创建ramdisk镜像

```shell
glance image-create --name ramdisk-01 --visibility public --disk-format ari --container-format ari <ironic-agent.initramfs
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\ramdisk镜像.png)

### 1.3 创建普通镜像

```shell
glance image-create --name ironic-image --visibility public --disk-format qcow2 --container-format bare < centos7-amd64-2.qcow2
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\普通镜像.png)

## 2. 创建网络

### 2.1 创建一个vlan网络

```shell
neutron net-create --tenant-id 5cd8bed396ee4a9492d9b912566649da --shared --provider:network_type vlan --provider:segmentation_id 508 --provider:physical_network physnet1 ironic-net
--provider:segmentation_id 508
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\创建vlan网络.png)

### 2.2 创建一个可以和ironic管理网通信的子网

```shell
neutron subnet-create --tenant-id 5cd8bed396ee4a9492d9b912566649da --name vlan_508 --gateway 192.168.55.254 --allocation-pool start=192.168.55.24,end=192.168.55.28 --enable-dhcp --ip-version 4 cc888c19-be1b-45b3-94d0-5d3bcb8091fd 192.168.55.0/24
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\创建子网.png)

### 2.3 修改ironic-conductor文件

```shell
将网络名称【ironic-net】配置到ironic-conductor的配置文件中，然后重启ironic-conductor服务：
重启命令：
docker restart ironic_conductor
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\ironic-conductor.png)

### 2.4 修改ironic-dnsmasq配置文件

```shell
将dhcp地址池配置到ironic-dnsmasq服务的配置文件中(96h直接填写)，然后重启ironic-dnsmasq服务：
重启服务的命令是：
docker restart ironic_dnsmasq
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\ironic-dnsmasq.png)

## 3.  创建裸服务器实例化需要的nova flavor

```shell
以本次实例化为例：2核cpu，4G内存，50G硬盘，x86_64的cpu架构
```

```shell
nova flavor-create --is-public true ironic-flavor auto 4096 50 2
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\nova flavor.png)

## 4. 注册ironic节点并更新信息

###4.1 注册节点

```shell
ironic node-create -d pxe_ipmitool_socat -n ironic-test-03
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\ironic节点.png)

### 4.2 更新ipmi属性信息

```shell
ironic node-update 0993a51c-2e07-4adc-a159-b507628e4ed7 add driver_info/ipmi_terminal_port=8901 driver_info/ipmi_username=root driver_info/ipmi_address=192.168.56.116 driver_info/ipmi_password=passw0rd@ctsi driver_info/ipmi_protocol_version=2.0
```

### 4.3 更新资源信息

```shell
ironic node-update 0993a51c-2e07-4adc-a159-b507628e4ed7 add properties/memory_mb=4096 properties/cpu_arch=x86_64 properties/local_gb=50 properties/cpus=2
```

### 4.4 更新镜像信息

```shell
ironic node-update 0993a51c-2e07-4adc-a159-b507628e4ed7 \
add driver_info/deploy_kernel=d206336a-d497-4164-aa71-859c48c6844a \
driver_info/deploy_ramdisk=2a0e5d94-b603-4902-9c52-b352a1828b0d
```

### 4.5 查看裸机服务器资源信息

```
如果ironic的节点注册正确无误，那么可以控制裸服务器的开关机，可以在hypervisor中看到完成注册的裸服务器
命令：
nova hypervisor-list
nova hypervisor-show ID
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\节点信息.png)



```shell
当查看信息报错的时候，执行以下命令，并查看状态
ironic node-set-provision-state 0993a51c-2e07-4adc-a159-b507628e4ed7 manage
ironic node-set-provision-state 0993a51c-2e07-4adc-a159-b507628e4ed7 provide
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\nova-list状态.png)

## 5. 修改openstack的配额信息

```shell
nova quota-update 5cd8bed396ee4a9492d9b912566649da --instance=-1 --cores=-1 --ram=-1 --metadata-items=-1 --injected-files=-1 --injected-file-content-bytes=-1 --injected-file-path-bytes=-1 --key-pairs=-1 --server-groups=-1 --server-group-members=-1
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\nova配额.png)

```shell
cinder quota-update 5cd8bed396ee4a9492d9b912566649da --backup-gigabytes=-1 --backups=-1 --gigabytes=-1 --per-volume-gigabytes=-1 --snapshots=-1 --volumes=-1
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\cinder配额.png)

```shell
neutron quota-update 5cd8bed396ee4a9492d9b912566649da --floatingip=-1 --network=-1 --port=-1 --rbac-policy=-1 --router=-1 --security-group=-1 --security-group-rule=-1 --subnet=-1 --subnetpool=-1
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\neutron配额.png)

## 6. 创建ironic的port

```shell
ironic port-create -a 08:C0:21:74:ED:25 -n b35ffba2-c49d-4940-a45a-23ab37d20140
## 08:C0:21:74:ED:25 实例化服务器的网卡ID
```

## 7.  实例化一个裸服务器的节点

```shell
nova boot --flavor cd9f44ab-4c56-4489-89d0-00bf4a756cdf --image 0a9f785d-7cbc-412f-afbe-db0240750aaf --nic net-id=cc888c19-be1b-45b3-94d0-5d3bcb8091fd ironic-test-17
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\实例化裸服务器节点.png)



# 问题处理

##ironic裸服务器实例化成功以后，console控制台打不开

```shell
解决方案：ironic放开对console的限制
命令如下：
ironic node-set-console-mode bece036b-6d96-4545-afa5-3b890626b07f true
```

```
再使用ironic获取console
ironic node-get-console bece036b-6d96-4545-afa5-3b890626b07f
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\console.png)