## 使用kolla-ansible部署openstack-yoga

- kolla-ansible是从kolla项目中分离出来的一个可交付的项目。kolla-ansible负责部署容器化的openstack各个服务和基础设施组件；而kolla项目现在则单独负责镜像的构建，为kolla-ansible部署提供生产级别的openstack各服务镜像。

- [参考文档](https://docs.openstack.org/project-deploy-guide/kolla-ansible/yoga/quickstart.html#host-machine-requirements)

### 系统安装


- OpenStack大版本对centos系统的要求对应关系如下：

| OpenStack 版本 | CentOS 版本
----------------|-------------
  Train 以及更早|     7
  Ussuri 和 Victoria| 8
  Wallaby 到 Yoga | Stream 8

- CentOS stream 8选择最小化安装。
- 使用默认的 LVM 分区方案。 除必要分区外，将剩余可用容量空间全部添加到 / 分区中。 无需添加 swap 分区。

### hosts设置

> 此步骤是kolla-ansible的一个step

```

填写 /etc/hosts（所有节点）
cat >> /etc/hosts <<EOF
172.240.12.11 SHA-STNC02-OST-ZJ-PSVR01
172.240.12.21 SHA-STNC02-OST-ZJ-PSVR02
172.240.12.31 SHA-STNC02-OST-ZJ-PSVR03
EOF

```

### yum源设置

```
 # 替换yum源为阿里源
 sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stream-AppStream.repo \
         /etc/yum.repos.d/CentOS-Stream-BaseOS.repo \
         /etc/yum.repos.d/CentOS-Stream-Extras.repo \
         /etc/yum.repos.d/CentOS-Stream-PowerTools.repo


# 清空之前的yum源
yum clean all 
# 生成缓存
yum makecache
```

### 禁用防火墙

~~~
systemctl stop firewalld && systemctl disable firewalld
~~~

### 打开centos路由转发

~~~
echo "net.ipv4.ip_forward = 1" >>/etc/sysctl.conf && sysctl -p /etc/sysctl.conf
~~~

### 安装 openstack 软件仓库

```

dnf install -y centos-release-openstack-yoga
dnf -y update

```

### 网络配置

####  openvswitch

```
# 安装openvswitch
dnf install -y openvswitch

# 启用openvswitch
systemctl start openvswitch && systemctl enable openvswitch

# 配置网桥
cat >>/etc/sysconfig/network-scripts/ifcfg-br-ex<<EOF
DEVICE=br-ex
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
DEVICETYPE=ovs
TYPE=OVSBridge
MTU=1500
OVS_EXTRA="set bridge br-ex fail_mode=standalone -- del-controller br-ex"
EOF

#配置bond0
cat >> /etc/sysconfig/network-scripts/ifcfg-bond0<<EOF
DEVICE=bond0
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-ex
BONDING_OPTS="mode=4 miimon=100"
MTU=1500
EOF

# eno1为bond0从属网卡
cat >> /etc/sysconfig/network-scripts/ifcfg-eno1<<EOF
DEVICE=eno1
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
MASTER=bond0
SLAVE=yes
BOOTPROTO=none
MTU=1500
EOF

# eno2为bond0从属网卡
cat >>/etc/sysconfig/network-scripts/ifcfg-eno2<<EOF
DEVICE=eno2
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
MASTER=bond0
SLAVE=yes
BOOTPROTO=none
MTU=1500
EOF

# 重启网络和网桥
systemctl restart network
systemctl restart openvswitch


# 检查网络和网桥
ip a


```

### 提前安装docker和podman

- Docker CE 安装需要一个包 containerd.io。来自 Docker 的这个 containerd.io 包包括 runc，此 runc 与 Podman 和 Skopeo 所需的 RHEL/CentOS 8 本机 runc 包冲突 

- 问题在 CentOS Stream 9 上不再发生（Stream 8 仍然存在）

- [问题详情](https://github.com/docker/containerd-packaging/issues/210)

- [问题详情](https://github.com/docker/containerd-packaging/pull/231)

```
# 安装 Podman 
sudo dnf install podman -y

# 测试podman是否成功
sudo podman run hello-world

# 安装docker仓库并禁用它
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf config-manager --set-disabled docker-ce-stable

# 安装 Docker containerd.io 
sudo rpm --install --nodeps --replacefiles --excludepath=/usr/bin/runc https://download.docker.com/linux/centos/8/x86_64/stable/Packages/containerd.io-1.6.8-3.1.el8.x86_64.rpm

# 安装 Docker CE，只为这个命令启用它的 repo
sudo dnf install --enablerepo=docker-ce-stable docker-ce

# 启用docker
sudo systemctl enable --now docker

# 测试docker
sudo docker run hello-world

# 测试podman
sudo podman run hello-world

# 更新
sudo dnf update



```

- [Red Hat Enterprise Linux 8 docker and podman](https://access.redhat.com/solutions/3696691)

- 如果需要可以安装[最新的软件包](https://download.docker.com/linux/centos/8/x86_64/stable/Packages/)



### 配置时钟同步

```
# 下载chrony
dnf install -y chrony
# 启用chrony
systemctl start chronyd && systemctl enable chronyd
# 同步时间
chronyc sources
```



### 安装 Python 构建依赖项

```
dnf install -y git  python3 sshpass tmux python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-netaddr
```

- 创建安装目录

```
mkdir -p /opt/oslostack && cd /opt/oslostack
```

### 获取部署脚本

```

git clone -b stable-6.0 https://github.com/ceph/ceph-ansible.git
git clone -b stable/yoga https://opendev.org/openstack/kolla-ansible.git

```
> 如果下载速度慢可以尝试在hosts文件中添加github.com或opendev.org这样下载的时候不走域名


###  openstack 所需 python 虚拟环境

```
# 生成kolla-ansible虚拟环境
python3 -m venv /opt/oslostack/venv
# 激活虚拟环境
source /opt/oslostack/venv/bin/activate


# 确保安装了最新版本的 pip
pip install -U pip 
# 安装ansible
pip install 'ansible>=4,<6'

cd ./kolla-ansible
# 检查版本
git checkout stable/yoga
# 安装
pip install .
cd ..
# 创建/etc/kolla目录
mkdir -p /etc/kolla
# 复制/kolla/*到/etc/kolla目录
cp ./venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla


# 安装 kolla-ansible ansible galaxy 依赖项
kolla-ansible install-deps

# 修改 ansible 配置文件
mkdir -p /etc/ansible
vim /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
log_path = /var/log/oslostack-ansible.log

[privilege_escalation]
become = True


```

### 准备初始配置

- 编辑修改multinode示例清单文件如下

```
# copy默认清单文件到inventory
cp ./kolla-ansible/ansible/inventory/multinode ./inventory
# 修改inventory文件
vi ./inventory

[all:vars]
# Host user name and password must be required if no ssh trust
ansible_connection=ssh
#
# Using password
ansible_user=root
ansible_ssh_pass=calpass123!@#
# These initial groups are the only groups required to be modified. The
# additional groups are for more control of the environment.
[control]
# These hostname must be resolvable from your deployment host
SHA-STNC02-OST-ZJ-PSVR01
SHA-STNC02-OST-ZJ-PSVR02
SHA-STNC02-OST-ZJ-PSVR03

# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
SHA-STNC02-OST-ZJ-PSVR01
SHA-STNC02-OST-ZJ-PSVR02
SHA-STNC02-OST-ZJ-PSVR03

[compute]
SHA-STNC02-OST-ZJ-PSVR01
SHA-STNC02-OST-ZJ-PSVR02
SHA-STNC02-OST-ZJ-PSVR03

[monitoring]
#monitoring01

# When compute nodes and control nodes use different interfaces,
# you need to comment out "api_interface" and other interfaces from the globals.yml
# and specify like below:
#compute01 neutron_external_interface=eth0 api_interface=em1 storage_interface=em1 tunnel_interface=em1
[storage:children]
control

[mons:children]
control

[osds:children]
compute

[grafana-server:children]
mons


[deployment]
localhost       ansible_connection=local

[baremetal:children]
control
network
compute
storage
monitoring

[tls-backend:children]

```

### 检查库存配置是否正确

```
ansible -i ./inventory all -m ping
```


### 编辑oslostack.yml文件

```
cat >> /opt/oslostack/oslostack.yml<<EOF
# 安装系统类型
kolla_base_distro: "centos"
# 源码安装
kolla_install_type: "source"
# 管理类型网络的默认接口
network_interface: "eno4"
# 业务网络接口
neutron_external_interface: "bond0"
# 为管理网络提供浮动IP
kolla_internal_vip_address: "172.240.12.50"
# 启用cinder
enable_cinder: "yes"
# 关闭cinder backup
enable_cinder_backup: "no"
# 关闭fluentd容器服务
enable_fluentd: "no"
# 关闭openvswitch容器服务（关闭这个服务是因为我们以及安装了openvswitch）
enable_openvswitch: "no"
# 关闭编排服务
enable_heat: "no"
```

### 生成 kolla 密码

```
kolla-genpwd
```



### 节点初始化

```
# 添加占位符
# 原因是我们并没有使用这个文件而是使用/opt/oslostack/oslostack.yml
vim /etc/kolla/globals.yml
---
dummy:


# 节点初始化
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml bootstrap-servers -vv
```

### 拉取openstack 容器镜像


```
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml pull -vv

```

- 如果速度慢可以采用国内镜像站

```
 vi /etc/docker/daemon.json 


{
    "bridge": "none",
    "ip-forward": false,
    "iptables": false,
    "registry-mirrors": ["https://zenjeblg.mirror.aliyuncs.com"], #可添加多个
    "log-opts": {
        "max-file": "5",
        "max-size": "50m"
    }
}

```

## 使用 ceph-ansible 部署 ceph 分布式存储

### 安装 ceph-ansible

#### 版本说明

* stable-3.2 支持 Ceph luminous 和 mimic 版本。这个分支需要 Ansible 2.6 版本。

* stable-4.0 支持 Ceph nautilus 版本。这个分支需要 Ansible 2.9 版本。

* stable-5.0 支持 Ceph pacific 版本。这个分支需要 Ansible 2.9 版本。

* stable-6.0 支持 Ceph pacific 版本。这个分支需要 Ansible 2.9 版本。

* stable-master 支持 Ceph master 分支 版本。这个分支需要 Ansible 2.10 版本。

- 切换到您希望使用的某个版本分支

```
cd ceph-ansible
git checkout stable-6.0

```

#### 准备 ceph-ansible python 虚拟环境

```
# 退出之前的虚拟环境
deactivate
# 生成ceph虚拟环境
python3 -m venv /opt/oslostack/ceph-venv
# 激活虚拟环境
source /opt/oslostack/ceph-venv/bin/activate
# 更新pip
pip install -U pip  -i https://mirrors.aliyun.com/pypi/simple
# 下载ansible
pip install -r /opt/oslostack/ceph-ansible/requirements.txt  -i https://mirrors.aliyun.com/pypi/simple
```

#### 安装 ceph-ansible ansible galaxy 依赖项

```
cd ceph-ansible
ansible-galaxy install -r requirements.yml
```
#### 填写 ceph-ansible 配置

```
cd /opt/oslostack
cat >> oslostack.yml <<EOF
# ceph
cephx: true

ceph_origin: distro
configure_firewall: false

osd_scenario: lvm
osd_objectstore: bluestore
osd_auto_discovery: true
# 显式设置 db 空间大小，单位为 bytes，默认 -1 为平分空间
block_db_size: -1

# 根据实际情况填写
public_network: "172.240.12.0/24"
cluster_network: "172.240.12.0/24"
monitor_interface: eno4
ntp_service_enabled: false

dashboard_admin_password: oslostack
grafana_admin_password: oslostack

crush_rule_config: true
crush_rule_hdd:
  name: hdd
  root: default
  type: host
  default: false
  class: hdd
# ssd crush
#crush_rule_ssd:
#  name: ssd
#  root: default
#  type: host
#  default: false
#  class: ssd
crush_rules:
  - "{{ crush_rule_hdd }}"
#  - "{{ crush_rule_ssd }}"

openstack_config: true
openstack_glance_pool:
  name: "images"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 0.2
openstack_nova_pool:
  name: "vms"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 0.1
# ssd pool
#openstack_cinder_ssd_pool:
#  name: "ssd-volumes"
#  rule_name: "{{ crush_rule_ssd.name }}"
#  application: "rbd"
#  pg_autoscale_mode: "on"
#  target_size_ratio: 1.0
openstack_cinder_hdd_pool:
  name: "hdd-volumes"
  rule_name: "{{ crush_rule_hdd.name }}"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 1.0
openstack_pools:
  - "{{ openstack_glance_pool }}"
  - "{{ openstack_cinder_hdd_pool }}"
  - "{{ openstack_nova_pool }}"
#  - "{{ openstack_cinder_ssd_pool }}"
openstack_keys:
  - { name: client.glance, caps: { mon: "profile rbd", osd: "profile rbd"}, mode: "0600" }
  - { name: client.cinder, caps: { mon: "profile rbd", osd: "profile rbd"}, mode: "0600" }
EOF
```

### 执使用 ceph-ansible 部署ceph

```
cd /opt/oslostack/ceph-ansible
ansible-playbook -i /opt/oslostack/inventory site.yml.sample -e @/opt/oslostack/oslostack.yml -vv
```


## 使用 kolla-ansible 部署 OpenStack

### 填写 OpenStack 与 Ceph 对接配置

```
# 配置ceph与openstack glance服务对接
# 创建 /etc/kolla/config/glance 目录
mkdir -p /etc/kolla/config/glance
# copyceph.client.glance.keyring 到/etc/kolla/config/glance/ 与glance服务对接
cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
# 复制ceph配置文件到/etc/kolla/config/glance/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/
# 创建cinder目录
mkdir -p /etc/kolla/config/cinder/cinder-volume
# 编辑cinder文件
vim /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=hdd
#enabled_backends=hdd,ssd

# 启用hdd配置
[hdd]
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
backend_host = rbd:hdd_volumes
rbd_pool = hdd-volumes
volume_backend_name = hdd
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = XXX 
# 查看 /etc/kolla/passwords.yml 中对应的 cinder_rbd_secret_uuid

#[ssd]
#rbd_ceph_conf = /etc/ceph/ceph.conf
#rbd_user = cinder
#backend_host = rbd:ssd_volumes
#rbd_pool = ssd-volumes
#volume_backend_name = ssd
#volume_driver = cinder.volume.drivers.rbd.RBDDriver
#rbd_secret_uuid = XXX # 查看 /etc/kolla/passwords.yml 中对应的 cinder_rbd_secret_uuid


# 配置ceph与openstack cinder 服务对接
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume
cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/
# 配置ceph与openstack nova 服务对接
mkdir -p /etc/kolla/config/nova
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/


# 开启 Glance 、 Cinder 和 Nova 的后端 Ceph 功能
cat >>  /opt/oslostack/oslostack.yml <<EOF
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
EOF

```

### ml2配置

```
# 此步骤是为了可以使用vlan网络，因为默认使用vxlan
mkdir /etc/kolla/config/neutron

vim /etc/kolla/config/neutron/ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
```


### 执行部署命令

```
# 退出ceph虚拟环境
deactivate
# 激活kolla-ansible虚拟环境
source /opt/oslostack/venv/bin/activate
# 部署之前的检查
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml prechecks -vv -e prechecks_enable_host_ntp_checks=false
# 正式部署
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy -vv
# 在部署节点上发布部署
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml post-deploy -vv
```


### 使用 OpenStack

- 安装 openstack 命令行

```
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/yoga
```

- 加载 openstack 凭证文件

```
source /etc/kolla/admin-openrc.sh
```

- 上传镜像 

```
# 下载cirros镜像
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
# 创建cirros镜像
openstack image create --name "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```

- 创建卷类型

```
openstack volume type create hdd --property volume_backend_name=hdd --property RESKEY:availability_zones=nova
```


- 获取 admin 用户密码登录openstack 172.240.12.50

```
cat /etc/kolla/admin-openrc.sh  |grep OS_PASSWORD  |awk -F "=" '{print $2}'
```










































