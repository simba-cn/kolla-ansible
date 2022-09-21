## kolla-ansible(官网版本)

说明： controller节点和compute节点必须为双网卡

[官网链接](https://docs.openstack.org/project-deploy-guide/kolla-ansible/yoga/quickstart.html#host-machine-requirements)

### 系统配置

### 主机名设置hosts设置

```
配置主机名节点1：
# hostnamectl set-hostname oslostack01
配置主机名节点2：
# hostnamectl set-hostname oslostack02
配置主机名节点3：
# hostnamectl set-hostname oslostack03


# 填写 /etc/hosts（所有节点）
# 此步骤可以不做，因为是kolla-ansible bootstrap-servers的一个步骤
cat >> /etc/hosts <<EOF
192.168.200.10 oslostack01
192.168.200.20 oslostack02
192.168.200.30 oslostack03
EOF

```


### yum源设置

```
 sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.aliyun.com/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stream-AppStream.repo \
         /etc/yum.repos.d/CentOS-Stream-BaseOS.repo \
         /etc/yum.repos.d/CentOS-Stream-Extras.repo \
         /etc/yum.repos.d/CentOS-Stream-PowerTools.repo
```

### 安装 openstack 软件仓库

```

dnf install -y centos-release-openstack-yoga


```

### 网络配置(所有节点)

- 将第一个接口配置为管理接口(有ip)：

- 将第二个接口配置为提供者接口：
```
cat > /etc/sysconfig/network-scripts/ifcfg-ens34 <<EOF
DEVICE=ens34
TYPE=Ethernet
ONBOOT="yes"
BOOTPROTO="none"
EOF
```
> 提供者接口不能配置ip

- 重启
```

systemctl restart NetworkManager

```

- 打开centos路由转发
~~~
 echo 'net.ipv4.ip_forward = 1'>>  /etc/sysctl.conf && sysctl -p  /etc/sysctl.conf
~~~

### 安装依赖

~~~
# 安装 Python 构建依赖项 

sudo dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux -y

# 安装pip
dnf install python3-pip -y

# 确保安装了最新版本的 pip
pip3 install -U pip

# 安装Ansible
sudo pip install -U 'ansible>=4,<6'


# 安装 Kolla-ansible
sudo pip3 install git+https://opendev.org/openstack/kolla-ansible@stable/yoga

# 创建/etc/kolla目录
sudo mkdir -p /etc/kolla

# 复制/kolla/*到/etc/kolla目录
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla

# 复制主机清单文件到当前目录
cp /usr/local/share/kolla-ansible/ansible/inventory/* .

# 安装 Ansible Galaxy 依赖项
kolla-ansible install-deps


# 配置 Ansible 
[defaults]
host_key_checking=False
pipelining=True
forks=100
log_path = /var/log/ansible.log


~~~


### 准备初始配置

- 编辑修改multinode示例清单文件如下
```
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
oslostack03


[deployment]
localhost       ansible_connection=local

[baremetal:children]
control
network
compute
storage
monitoring

[tls-backend:children]
control

```


### 检查库存配置是否正确
~~~
ansible -i multinode all -m ping
~~~

### 生成kolla密码
~~~
kolla-genpwd
~~~


### 编辑globals.yml文件

globals.yml是 Kolla Ansible 的主要配置文件。

- 以下配置文件是必须选项

~~~
cat >> /etc/kolla/globals.yml<<EOF
kolla_base_distro: "centos"
kolla_install_type: "source"
network_interface: "ens33"
neutron_external_interface: "ens34"
kolla_internal_vip_address: "192.168.200.11"
EOF
~~~

## 部署

### 初始化节点

~~~
kolla-ansible -i ./multinode bootstrap-servers
~~~

### 检查节点

~~~
kolla-ansible -i ./multinode prechecks
~~~

### 下载所需docker镜像

> 此步骤可选

~~~
kolla-ansible -i ./multinode pull
~~~

### 正式部署

~~~
kolla-ansible -i ./multinode deploy
~~~


### 下载安装 OpenStack CLI 客户端

~~~
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/yoga
~~~

### 生成admin的凭据

~~~
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
~~~