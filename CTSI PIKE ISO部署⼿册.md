# CTSI PIKE ISO部署⼿册

## 1. 安装系统

## 2. 系统初始化

```
默认的登陆信息： root/P@sw0rd
查看⽹卡信息 默认⽹卡1为pxe使⽤的⽹卡
```

## 3. 进入/opt/boot_setup 文件夹

```shell
执⾏： ./init.sh
等待初始化完成
##注意多机部署需要修改配置文件中的 hostname名称
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\init.sh文件.png)



```shell
开启pxe服务（cobbler），进行系统部署推送
执⾏： docker start cobbler
```

##4. 网络初始化

将环境各节点信息写⼊

```shell
vim /etc/hosts

192.168.52.7 deploy
192.168.52.8 control01
192.168.52.9 compute01
```

标明各节点⻆⾊（control,compute等）

```shell
vim /etc/hostname
```

##5. 部署前的配置

控制节点

```shell
将/root/kolla-ansible/ansible/inventory⽂件夹中multinode⽂件拷⻉⾄/root/kolla-ansible/tools⽂件夹
编辑 multinode 文件【根据实际需求进行填写】
```

##6. 建立本地镜像仓库

###6.1 部署推送节点

将镜像包拷贝并解压至/opt/registry 文件夹 ，结构必须满足如下要求

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\镜像仓库地址.png)

### 6.2 修改docker.server文件

```shell
vim  /usr/lib/systemd/system/docker.service  ##所有节点全部修改
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\docker.service.png)

```shell
这⾥改为control节点ip地址。
执⾏： systemctl daemon-reload
systemctl restart docker
执⾏ systemctl status docker，查看仓库是否配置完成。
```

![](F:\北京中北信科技发展有限公司\工作文档\openstack-pike\裸机部署openstack-P\picture\docker状态.png)

### 6.3 修改globals.yml

(根据实际情况进行修改)

```shell
vi  /etc/kolla/globals.yml
kolla_internal_vip_address：控制节点的IP
docker_registry：为部署推送节点的IP
kolla_internal_vip_address: "192.168.52.8"    #控制节点的IP
network_interface：为⽹卡1  #####各个节点的网卡名称必须相同
neutron_external_interface：为⽹卡2  #####各个节点的网卡名称必须相同
neutron_plugin_agent：根据测试环境所使⽤虚拟化对应选择（kvm：openvswitch， vmware： vmware_dvs 或 vmware_nsxv)

对接vmware需要开启(vmware使⽤vmware， kvm使⽤qemu)
nova_computer-virt_type: "vmware"
```

## 7. 进行部署

推送节点

```shell
cd /etc/kolla-ansible/tools 
ansibke -i multinode all -m ping   ##检查各节点的连通性

执⾏： ./kolla-ansible -i multinode pull，拉取镜像
执⾏： ./kolla-ansible -i multinode deploy，执⾏部署步骤
执⾏： ./kolla-ansible -i multinode post-deploy，执⾏部署后配置
```

# 附录-修改主机网卡

## 1.  编辑 grub 配置文件

```shell
vim /etc/sysconfig/grub   # 其实是/etc/default/grub的软连接
# 为GRUB_CMDLINE_LINUX变量增加2个参数，具体内容如下(**范围内)：
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap **net.ifnames=0 biosdevname=0**  rhgb quiet"
```

## 2. 重新生成 grub 配置文件

```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
然后重新启动 Linux 操作系统，通过 ip addr 可以看到网卡名称已经变为 eth0 。
```

##3. 修改网卡配置文件

```
将en***修改为eth*,并且将文件备份
```

