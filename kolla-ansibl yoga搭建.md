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

### 主机名设置hosts设置

```
配置主机名节点1：
# hostnamectl set-hostname oslostack01
配置主机名节点2：
# hostnamectl set-hostname oslostack02
配置主机名节点3：
# hostnamectl set-hostname oslostack03


填写 /etc/hosts（所有节点）
cat >> /etc/hosts <<EOF
192.168.200.10 oslostack01
192.168.200.20 oslostack02
192.168.200.30 oslostack03
EOF

```

### yum源设置

```
 sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stream-AppStream.repo \
         /etc/yum.repos.d/CentOS-Stream-BaseOS.repo \
         /etc/yum.repos.d/CentOS-Stream-Extras.repo \
         /etc/yum.repos.d/CentOS-Stream-PowerTools.repo
```

### 安装 openstack 软件仓库

```

dnf install -y centos-release-openstack-yoga
dnf -y update

```

### 网络配置

####  卸载NetworkManager

```
systemctl stop NetworkManager &&  systemctl disable NetworkManager
dnf remove -y NetworkManager

```


####  openvswitch


```
dnf install -y openvswitch
systemctl start openvswitch && systemctl enable openvswitch
# 配置网桥
cat >> /etc/sysconfig/network-scripts/ifcfg-br-ex <<EOF
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
cat > /etc/sysconfig/network-scripts/ifcfg-ens34 <<EOF
DEVICE=ens34
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


systemctl restart network
systemctl restart openvswitch

ip a


```


#### 打开路由转发功能

```
echo 'net.ipv4.ip_forward = 1'>> /etc/sysctl.conf

sysctl -p /etc/sysctl.conf
```

#### 防火墙设置

```
# 禁用firewalld
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

### 配置时钟同步

```
dnf install -y chrony
systemctl start chronyd && systemctl enable chronyd
chronyc sources
```

### 检查并清理硬盘

```
dd if=/dev/zero of=/dev/sdb count=10 bs=1M
```


## 在部署节点上进行

```
dnf install -y git vim python3 sshpass tmux python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-netaddr
```
> 后续操作步骤可选择在 tmux 终端中进行，避免会话丢失造成部署过程中断。

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

python3 -m venv /opt/oslostack/venv
source /opt/oslostack/venv/bin/activate


# 安装 python 依赖
pip install -U pip 
pip install 'ansible>=4,<6'

# 安装 kolla-ansible

cd ./kolla-ansible
git checkout stable/yoga
pip install .
cd ..
mkdir -p /etc/kolla
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


### 填写 ansible inventory

* control 组指控制节点，主要运行 OpenStack 控制层面的服务

* compute 组指计算节点，主要运行 hypervisor 以及 虚拟机工作负载

* mons 组指 ceph mon 节点，主要运行 ceph mon 管理服务

* osds 组指 ceph osd 节点，主要运行 ceph osd 服务

- 以下为部分内容

```
cp ./kolla-ansible/ansible/inventory/multinode ./inventory
vim ./inventory
[all:vars]
# Host user name and password must be required if no ssh trust
ansible_connection=ssh
#
# Using password
ansible_user=root
ansible_ssh_pass=123456

# These initial groups are the only groups required to be modified. The
# additional groups are for more control of the environment.
[control]
# These hostname must be resolvable from your deployment host
oslostack01

# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
oslostack01
oslostack02

[compute]
oslostack02

[monitoring]
#monitoring01

# When compute nodes and control nodes use different interfaces,
# you need to comment out "api_interface" and other interfaces from the globals.yml
# and specify like below:
#compute01 neutron_external_interface=eth0 api_interface=em1 storage_interface=em1 tunnel_interface=em1

[storage]
oslostack01
oslostack02

[mons]
oslostack01
oslostack02
oslostack03

[osds]
oslostack01
oslostack02
oslostack03

[grafana-server]
oslostack01

[deployment]
oslostack01

[baremetal:children]
control
network
compute
storage
monitoring

[tls-backend:children]
control

```

#### 测试节点连通性

```
ansible -i ./inventory all -m ping
```


#### kolla-ansible 配置

```
cd /opt/oslostack
vim oslostack.yml


kolla_base_distro: "centos"
kolla_install_type: "source"

# 根据实际情况填写
network_interface: "ens33"
neutron_external_interface: "ens34"
kolla_internal_vip_address: "192.168.200.10"
nova_compute_virt_type: "qemu" 
# nova_compute_virt_type 虚拟化选项虚拟机为qemu默认为kvm
enable_haproxy: "no"
# enable_haproxy 关闭高可用是因为kolla_internal_vip_address 这里采用了管理地址
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_fluentd: "no"
enable_openvswitch: "no"
enable_heat: "no"

```

#### 生成 kolla 密码

```
kolla-genpwd
```

#### 可以提前安装docker和podman

```
sudo dnf install podman

sudo podman run hello-world

sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf config-manager --set-disabled docker-ce-stable

sudo rpm --install --nodeps --replacefiles --excludepath=/usr/bin/runc https://download.docker.com/linux/centos/8/x86_64/stable/Packages/containerd.io-1.6.8-3.1.el8.x86_64.rpm



sudo dnf install --enablerepo=docker-ce-stable docker-ce

sudo systemctl enable --now docker

sudo docker run hello-world

sudo podman run hello-world

sudo dnf update



```
- [Red Hat Enterprise Linux 8 docker and podman](https://access.redhat.com/solutions/3696691)
- 如果需要可以安装[最新的软件包](https://download.docker.com/linux/centos/8/x86_64/stable/Packages/)


#### 节点初始化

```
# 添加占位符
vim /etc/kolla/globals.yml
---
dummy:



kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml bootstrap-servers -vv
```

#### 拉取openstack 容器镜像


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
deactivate
python3 -m venv /opt/oslostack/ceph-venv
source /opt/oslostack/ceph-venv/bin/activate
pip install -U pip  -i https://mirrors.aliyun.com/pypi/simple
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
# 离线 registry
# ceph_docker_registry: "10.0.10.11:4000"
# containerized_deployment: true
# container_binary: docker
# container_package_name: docker-ce
ceph_origin: distro
configure_firewall: false

osd_scenario: lvm
osd_objectstore: bluestore
osd_auto_discovery: true
# 显式设置 db 空间大小，单位为 bytes，默认 -1 为平分空间
block_db_size: -1

# 根据实际情况填写
public_network: "192.168.200.0/24"
cluster_network: "192.168.200.0/24"
monitor_interface: ens33

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

#### 执行 ceph-ansible 部署命令

```
cd /opt/oslostack/ceph-ansible
ansible-playbook -i /opt/oslostack/inventory site.yml.sample -e @/opt/oslostack/oslostack.yml -vv
```


### 使用 kolla-ansible 部署 OpenStack

#### 填写 OpenStack 与 Ceph 对接配置

```
mkdir -p /etc/kolla/config/glance

cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/

mkdir -p /etc/kolla/config/cinder/cinder-volume

vim /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=hdd
#enabled_backends=hdd,ssd


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

cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume
cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/

mkdir -p /etc/kolla/config/nova
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/

cat >>  /opt/oslostack/oslostack.yml <<EOF
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
EOF

```

#### ml2配置

```
mkdir /etc/kolla/config/neutron
vim /etc/kolla/config/neutron/ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
```


#### 执行部署命令

```
deactivate
source /opt/oslostack/venv/bin/activate
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml prechecks -vv -e prechecks_enable_host_ntp_checks=false
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy -vv
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

- 执行 openstack 命令

```
openstack compute service list
```

- 获取 admin 用户密码

```
cat /etc/kolla/passwords.yml | grep keystone_admin_password | awk '{print $2}'
```

- 上传镜像 

```
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img


openstack image create --name "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```


- 创建卷类型

```
openstack volume type create hdd --property volume_backend_name=hdd --property RESKEY:availability_zones=nova
openstack volume type create ssd --property volume_backend_name=ssd --property RESKEY:availability_zones=nova
```










































