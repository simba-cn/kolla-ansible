## openstack-yoga

* 如没有特殊说明操作均在控制节点操作

### 关闭防火墙和selinux

- 测试环境关闭防火墙和selinux是为了后面不必要的麻烦

> 此步骤官网没有（可选）

```
# 所有节点

systemctl stop firewalld  # 关闭firewalld防火墙

systemctl disable firewalld # 禁用firewalld


setenforce 0  # 临时关闭selinux


sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config   #永久禁用

```


### 主机网络设置

- controller节点

```
hostnamectl set-hostname controller #更改主机名

# 第一块网卡作为管理网卡使用

IP地址：192.168.200.23

网络掩码：255.255.255.0（或/24）

默认网关：192.168.200.1

DNS : 223.5.5.5

# 第二块网卡作为提供者网卡使用

编辑/etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME文件以包含以下内容：

cat >>/etc/sysconfig/network-scripts/ifcfg-ens34<<EOF
DEVICE=ens34
TYPE=Ethernet
ONBOOT="yes"
BOOTPROTO="none"
EOF

# 配置名称解析

 cat >>/etc/hosts <<EOF
192.168.200.23 controller 
192.168.200.24 compute
192.168.200.25 block
EOF

```

- compute节点

```
hostnamectl set-hostname compute #更改主机名

# 第一块网卡作为管理网卡使用

IP地址：192.168.200.24

网络掩码：255.255.255.0（或/24）

默认网关：192.168.200.1

DNS : 223.5.5.5

# 第二块网卡作为提供者网卡使用

编辑/etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME文件以包含以下内容：

cat >>/etc/sysconfig/network-scripts/ifcfg-ens34<<EOF
DEVICE=ens34
TYPE=Ethernet
ONBOOT="yes"
BOOTPROTO="none"
EOF

# 配置名称解析

 cat >>/etc/hosts <<EOF
192.168.200.23 controller 
192.168.200.24 compute
192.168.200.25 block
EOF

```

- block节点

```
hostnamectl set-hostname block #更改主机名



IP地址：192.168.200.25

网络掩码：255.255.255.0（或/24）

默认网关：192.168.200.1

DNS : 223.5.5.5

# 配置名称解析

 cat >>/etc/hosts <<EOF
192.168.200.23 controller 
192.168.200.24 compute
192.168.200.25 block
EOF

```

### 安装时间同步服务


```
# 所有节点

yum install chrony -y 

# controller节点

timedatectl set-timezone Asia/Shanghai # 设置时区为上海

# 向/etc/chrony.conf添加如下两行
# time1.aliyun.com NTP_SERVER
# allow 192.168.200.0/24 使其他节点能够连接到控制器节点上的 chrony 守护程序
cat >>/etc/chrony.conf<<EOF
server time1.aliyun.com iburst   
allow 192.168.200.0/24
EOF

# compute节点和block节点

echo 'server controller iburst'>>/etc/chrony.conf 

# 所有节点


systemctl enable --now chronyd  # 启用

systemctl status chronyd # 查看启动状态

chronyc sources # 同步时间

# 控制节点
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^- 119.28.206.193                2   6   107    16  +1831us[+1831us] +/-   28ms
^- 111.230.189.174               2   6    53    18  -1123us[-1123us] +/-   33ms
^- time.cloudflare.com           3   6    17    22  -8914us[-8914us] +/-  131ms
^- ntp1.flashdance.cx            2   6    27    18  +5834us[+5834us] +/-  172ms
^* 203.107.6.88                  2   6    17    21   +291us[ +378us] +/-   17ms


# 其他节点 （出现controller节点代表成功）

MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* time.cloudflare.com           3   6    17    21   +104us[-2640us] +/-  115ms
^+ ntp.wdc1.us.leaseweb.net      2   6    17    21    -22ms[  -25ms] +/-  268ms
^- ntp1.flashdance.cx            2   6    17    19  -5528us[-5528us] +/-  172ms
^- ntp5.flashdance.cx            2   6   131    13  -9506us[-9506us] +/-  193ms
^- controller                    3   6    17    16  -7055us[-7055us] +/- 5807ms



```

### OpenStack 软件包



```
 yum install centos-release-openstack-yoga -y # 安装openstack软件包

 yum config-manager --set-enabled powertools #  CentOS8需要启用 PowerTools 存储库

yum upgrade -y # 升级所有节点上的包

 yum install python3-openstackclient -y # 安装 OpenStack 客户端

yum install openstack-selinux -y #  CentOS 默认启用SELinux安装 openstack-selinux软件管理 OpenStack 服务的安全策略


```
### sql数据库

```
yum install mariadb mariadb-server python3-PyMySQL  -y # 安装数据库

# 创建和编辑/etc/my.cnf.d/openstack.cnf文件
# 创建一个[mysqld]section，允许其他节点通过管理网络访问。

cat >> /etc/my.cnf.d/openstack.cnf <<EOF
[mysqld]
bind-address = 192.168.200.23  # controller节点的管理IP地址
default-storage-engine = innodb # 设置存储引擎
innodb_file_per_table = on
max_connections = 4096   # 最大连接数
collation-server = utf8_general_ci # 设置字符编码
character-set-server = utf8 # 设置字符编码
EOF

# 启动数据库服务并将其配置为在系统启动时启动：

systemctl enable mariadb.service
systemctl start mariadb.service

# 运行脚本mysql_secure_installation设置密码

mysql_secure_installation

Enter current password for root (enter for none): 回车

Set root password? [Y/n] y
# 将要求输入数据库 root 账户密码 MARIADB_PASS
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y



mysql -u root -p   # 测试


```

### 消息队列服务

- 消息队列来协调服务之间的操作和状态信息。消息队列服务通常在控制器节点上运行。

```
yum install rabbitmq-server -y # 安装消息队列服务

systemctl enable rabbitmq-server  # 开机启动

systemctl start  rabbitmq-server # 启动
 
rabbitmqctl add_user openstack RABBIT_PASS   # 添加openstack用户注意将 RABBIT_PASS  修改为 消息队列密码

rabbitmqctl set_permissions openstack ".*" ".*" ".*" # 给openstack用户增加所有权限
```


### 内存对象缓存

```
yum install memcached python3-memcached -y #安装  memcached

# 编辑/etc/sysconfig/memcached文件

cat >>/etc/sysconfig/memcached<<EOF
OPTIONS="-l 127.0.0.1,::1,controller"
EOF
# 如启动出现绑定失败问题，则修改为
# OPTIONS="-l 127.0.0.1,::1,管理网络IP地址"



# 启动 Memcached 服务并将其配置为在系统启动时启动

 systemctl enable memcached

 systemctl start memcached



```



### etcd

- etcd是一种分布式可靠键值存储，用于分布式键锁定、存储配置、跟踪服务活动性和其他场景。etcd 服务在控制器节点上运行。 

```
yum install etcd -y # 安装etcd


# 编辑/etc/etcd/etcd.conf文件
# 将ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, 设置ETCD_LISTEN_CLIENT_URLS为控制器节点的管理 IP 地址以允许其他节点通过管理网络访问
# # 注意 controller 为 控制节点的 hostname   192.168.200.23 为控制节点管理网络的 IP  确保一致性

cat >>/etc/etcd/etcd.conf<<EOF
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.200.23:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.200.23:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.200.23:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.200.23:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.200.23:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

# 启用并启动 etcd 服务

systemctl enable etcd && systemctl start etcd

```


## 安装 OpenStack 服务（控制节点）

### keystone身份认证服务

- OpenStack Identity 服务提供了用于管理身份验证、授权和服务目录的单点集成

- 身份服务通常是用户与之交互的第一个服务。一旦通过身份验证，最终用户就可以使用他们的身份访问其他 OpenStack 服务。同样，其他 OpenStack 服务利用 Identity 服务来确保用户是他们所说的身份，并发现其他服务在部署中的位置。身份服务还可以与一些外部用户管理系统（例如 LDAP）集成。


```
# 使用数据库创建keystone数据库，创建keystone用户并给与访问keystone数据库的权限

mysql -u root -p

MariaDB [(none)]> CREATE DATABASE keystone;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
    -> IDENTIFIED BY '000000';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]>  GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
    -> IDENTIFIED BY '000000';
Query OK, 0 rows affected (0.000 sec)

# 替换000000 为自己的密码

```

- 安装和配置组件

```
yum install openstack-keystone httpd python3-mod_wsgi -y  # Centos8 及以上安装包 python3-mod_wsgi

# 编辑/etc/keystone/keystone.conf文件

[database]
# ...
connection = mysql+pymysql://keystone:000000@controller/keystone


[token]
# ...
provider = fernet




su -s /bin/sh -c "keystone-manage db_sync" keystone  # 填充keystone数据库


# 初始化 Fernet 密钥存储库
# 提供--keystone-user 和--keystone-group 是为了允许在另一个操作系统user/group下运行 keystone

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone


# 引导身份服务
# 000000 为 admin 账户密码
keystone-manage bootstrap --bootstrap-password 000000 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

# 配置 Apache HTTP 服务器
# 编辑/etc/httpd/conf/httpd.conf文件并配置 ServerName选项以引用控制器节点
echo 'ServerName controller'>>/etc/httpd/conf/httpd.conf


# 创建/usr/share/keystone/wsgi-keystone.conf文件的链接

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

# 启动 Apache HTTP 服务并将其配置为在系统启动时启动

systemctl enable httpd && systemctl start httpd


# 设置适当的环境变量来配置管理帐户


cat >> /etc/keystone/admin-openrc.sh <<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

# OS_USERNAME  登录 OpenStack 服务的用户名
# OS_PASSWORD  登录 OpenStack 服务的用户密码
# OS_PROJECT_NAME 登录时进入的项目名
# OS_USER_DOMAIN_NAME  登录时进入的域名
# OS_PROJECT_DOMAIN_NAME  登录时进入的项目域名
# OS_AUTH_URL 指定 Keystone（身份认证服务）的 URL  
# 如未部署 DNS 服务器，则需要在 hosts中指定 controller 映射，或将 controller 用控制节点 IP 替代
# OS_IDENTITY_API_VERSION 身份认证服务的 API 版本号 
# OS_IMAGE_API_VERSION 镜像服务的 API 版本号


source /etc/keystone/admin-openrc.sh # 获取权限


# 创建service 项目
 openstack project create --domain default \
  --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | c6a1ef118bbd455dbb79fe02ce267944 |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+


# 验证服务可用性
# 卸载 admin 用户的环境
unset OS_AUTH_URL OS_PASSWORD

# 验证 admin 用户可用性
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
# 输入后将要求输入 管理员 admin 的密码
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-09-16T07:17:43+0000                                                                                                                                                                |
| id         | gAAAAABjJBUHA4Lk0B2_Dv5oL8l-wHiKz_wLJ6bGo1KPNb9DJQRMUOPfj_8hLmvXLDvFazftG4Jesdgj7tS24KhWBowtr-e37ydPspRSkw5yZZKPs1d2PTLKiWaCHl2FCp5ZTPPmO0oR0UdQ86ugfFKqQ5vQBuJNPTyX7PsKpkYVRFYIhKqurCQ |
| project_id | cde067474dd741d08a2aa0f5a9443f82                                                                                                                                                        |
| user_id    | 4bfee6d33f874784b17e42f7def140bc                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
# 返回  token 信息则服务正常



```

### 镜像服务 Glance

```
yum install openstack-glance -y

mysql -u root -p
# 创建glance数据库
MariaDB [(none)]> CREATE DATABASE glance; 
# 授予对glance数据库的适当访问权限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY '000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY '000000';



  openstack user create --domain default --password-prompt glance # 创建glance用户


  openstack role add --project service --user glance admin # 将admin角色添加到glance用户和 service项目

# 为 Glance 添加管理镜像的服务
  openstack service create --name glance \
  --description "OpenStack Image" image

# 创建image服务 API 端点
  openstack endpoint create --region RegionOne \
  image public http://controller:9292

  openstack endpoint create --region RegionOne \
  image internal http://controller:9292

   openstack endpoint create --region RegionOne \
  image admin http://controller:9292

# 编辑/etc/glance/glance-api.conf文件
[DEFAULT]
use_keystone_quotas = True  # 启用每租户配额
log_file = /var/log/glance/glance.log # 确定日志文件

# 配置数据库访问
[database]
# ...
connection = mysql+pymysql://glance:000000@controller/glance

# [keystone_authtoken]和[paste_deploy]部分中，配置身份服务访问

[keystone_authtoken]
# ...
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = 000000

[paste_deploy]
# ...
flavor = keystone



# 配置本地文件系统存储和图像文件的位置
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/



# 配置对 keystone 的访问
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = admin
# 使用 admin
system_scope = all
password = 000000
# 使用 admin  用户的密码
endpoint_id = 6b58af720a9846829bee5330f52416c9   
# 使用 openstack endpoint list 查询  glance 服务 对应 admin 用户的 endpoint_id
region_name = RegionOne



# 确保 admin 帐户具有对系统范围资源的读者访问权限
openstack role add --user admin --user-domain Default --system all reader 



# 同步 Glance 数据到数据库

su -s /bin/sh -c "glance-manage db_sync" glance


# 启动服务并将它们配置为在系统启动时启动

  systemctl start openstack-glance-api&&systemctl enable openstack-glance-api


 # 验证服务可用性

 wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img

 openstack image create cirros --disk-format qcow2 --file cirros-0.5.2-x86_64-disk.img   --container-format bare --public 

  openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 1f7c5509-0828-48b1-a20b-3e51b185cdc3 | cirros | active |
+--------------------------------------+--------+--------+
```


### 安置服务Placement

```
mysql -u root -p000000

CREATE DATABASE placement; # 创建placement数据库

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
  IDENTIFIED BY '000000'; # 给与placement用户本地访问权限

  GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
  IDENTIFIED BY '000000';# 给与远程用户访问权限
# 创建放置服务用户
openstack user create --domain default --password-prompt placement
# 将 Placement 用户添加到具有管理员角色的服务项目中
openstack role add --project service --user placement admin

# 在服务目录中创建 Placement API 条目
openstack service create --name placement \
  --description "Placement API" placement

# 创建 Placement API 服务端点
  openstack endpoint create --region RegionOne \
  placement public http://controller:8778


  openstack endpoint create --region RegionOne \
  placement internal http://controller:8778



  openstack endpoint create --region RegionOne \
  placement admin http://controller:8778




# 安装openstack-placement
yum install openstack-placement-api -y 

# 编辑/etc/placement/placement.conf文件

[placement_database]
# ...
connection = mysql+pymysql://placement:000000@controller/placement
# 配置数据库访问
# PLACEMENT_DBPASS 为 placement 服务的数据库账户密码


# 配置身份服务访问
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = 000000
# PLACEMENT_PASS 为 placement 服务的密码



# 重启httpd服务

systemctl restart httpd

```



### 计算服务Nova

```

# 数据库设置
mysql -u root -p000000
# 创建 nova_api，nova，nova_cell0数据库
MariaDB [(none)]> CREATE DATABASE nova_api，;
MariaDB [(none)]> CREATE DATABASE nova，;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
# 创建nova_api，nova，nova_cell0 用户 分别给与nova_api，nova，nova_cell0数据库远程访问本地访问权限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY '000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY '000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY '0000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY '000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY '000000';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY '000000';
# NOVA_DBPASS 为 nova 服务的密码





openstack user create --domain default --password-prompt nova# 创建nova用户


# 将admin角色添加到nova用户
 openstack role add --project service --user nova admin

 # 创建nova服务

 openstack service create --name nova \
  --description "OpenStack Compute" compute

 # 创建计算 API 服务端点

 openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1

   openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1


  openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

# 安装组件
yum install -y \
    openstack-nova-api \
    openstack-nova-scheduler \
    openstack-nova-conductor \
    openstack-nova-novncproxy \
    iptables

# 编辑/etc/nova/nova.conf文件


[DEFAULT]
# …
enabled_apis = osapi_compute,metadata
# 启用计算和元数据 API
transport_url = rabbit://openstack:000000@controller:5672/
# 配置RabbitMQ消息队列访问
my_ip = 192.168.200.23
# 控制节点控制网络的 IP
log_file = /var/log/nova/nova-controller.log
# 日志配置
rootwrap_config = /etc/nova/rootwrap.conf

[api_database]
# 配置数据库访问
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
# NOVA_DBPASS 为数据库 Nova 账户密码

[database]
# 配置数据库访问
# ...
connection = mysql+pymysql://nova:000000@controller/nova
# NOVA_DBPASS 为数据库 Nova 账户密码


# 配置身份服务访问
[api]
# ...
auth_strategy = keystone
# 配置身份服务访问
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
# NOVA_PASS 为 Nova 服务的密码



# 将 VNC 代理配置为使用控制器节点的管理接口 IP 地址
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...

# 配置图像服务 API 的位置
api_servers = http://controller:9292

[oslo_concurrency]
# ...
# 配置锁定路径
lock_path = /var/run/nova


# 配置对 Placement 服务的访问
[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
# PLACEMENT_PASS 为 placement 服务的密码



# 填充nova-api数据库
su -s /bin/sh -c "nova-manage api_db sync" nova 

# 注册cell0数据库
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

# 创建cell1单元格：
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

# 填充nova数据库
su -s /bin/sh -c "nova-manage db sync" nova


# 启动计算服务并将它们配置为在系统启动时启动


systemctl enable \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service


 systemctl start \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service


```


### 网络服务Neutron

```
mysql -u root -p000000

# 创建neutron数据库
MariaDB [(none)] CREATE DATABASE neutron;
# 创建neutron用户给与本地访问权限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY '000000';
# 创建neutron用户给与远程访问权限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY '000000';
# NEUTRON_DBPASS 为数据库 neutron 账户的密码

# 创建neutron用户
openstack user create --domain default --password-prompt neutron
# 将要求输入密码 此密码为 neutron 服务的密码  NEUTRON_PASS
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 517d1a87268b447abe1ad0897fef1627 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
# 将admin角色添加到neutron用户
openstack role add --project service --user neutron admin
# 创建neutron服务
openstack service create --name neutron \
  --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | f5e3b3fe31c343a3a20b673e6a3197c2 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
# 创建网络服务 API 端点
openstack endpoint create --region RegionOne \
  network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c90d35bbd94047c1b3cb516ede133de5 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f5e3b3fe31c343a3a20b673e6a3197c2 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne \
  network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 823fb83c1cec4beeb59f646f309fb071 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f5e3b3fe31c343a3a20b673e6a3197c2 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne \
  network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 96c986ba58674023a5899a4b2661a55e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f5e3b3fe31c343a3a20b673e6a3197c2 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+


# 选择网络选项 2：自助服务网络
yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables -y

vi /etc/neutron/neutron.conf

# 配置数据库访问
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
# NEUTRON_DBPASS为 数据库 neutron 账户密码


[DEFAULT]
# ...
core_plugin = ml2
# 启用模块化第 2 层 (ML2) 插件
service_plugins = router
# 启用模块化第 2 层 (ML2) 插件路由器服务
allow_overlapping_ips = true
# 允许重叠 IP 地址
transport_url = rabbit://openstack:RABBIT_PASS@controller
# 配置RabbitMQ 消息队列访问
auth_strategy = keystone
# 配置身份服务访问
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
# 配置 Networking 以通知 Compute 网络拓扑更改
# RABBIT_PASS 为 消息队列密码

[keystone_authtoken]
# ...

# 配置身份服务访问
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
# NEUTRON_PASS为 neutron 服务密码


# 配置 Networking 以通知 Compute 网络拓扑更改
[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
# [nova]  没有则添加
# NOVA_PASS 为 Nova 服务密码



# 配置锁定路径
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp




cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak

vi /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
# ...
type_drivers = flat,vlan,vxlan
# flat、VLAN 和 VXLAN 网络
tenant_network_types = vxlan
# 启用 VXLAN 自助服务网络
mechanism_drivers = linuxbridge,l2population
# 启用 Linux 桥接和第 2 层填充机制
extension_drivers = port_security
# 启用端口安全扩展驱动程序

[ml2_type_flat]
# ...
flat_networks = provider
# 将提供者虚拟网络配置为flat

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000
# 为自助服务网络配置 VXLAN 网络标识符范围

[securitygroup]
# ...
enable_ipset = true
# 启用 ipset 以提高安全组规则的效率

# 没有则添加
:wq


cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak



# 编辑/etc/neutron/plugins/ml2/linuxbridge_agent.ini
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
# 提供者虚拟网络映射到提供者物理网络接口
# PROVIDER_INTERFACE_NAME 为 服务提供网络所对应的网卡编号

[vxlan]
enable_vxlan = true
# ，启用 VXLAN 覆盖网络
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
# 配置处理覆盖网络的物理网络接口的 IP 地址
l2_population = true
# 启用第 2 层填充
# OVERLAY_INTERFACE_IP_ADDRESS 为管理网络 控制节点的 IP  即 controller IP

[securitygroup]
# ...
enable_security_group = true
# 启用安全组
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
# 配置 Linux 网桥 iptables 防火墙驱动程序

# 没有则添加
:wq

modprobe br_netfilter
# 加载模块modprobe br_netfilter

# 配置自动加载br_netfilter模块
cat >>/etc/rc.sysinit<<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF

echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules

chmod 755 /etc/sysconfig/modules/br_netfilter.modules

# 验证模块
sysctl -a | grep net.bridge.bridge-nf-call
# net.bridge.bridge-nf-call-arptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1
# net.bridge.bridge-nf-call-iptables = 1


# 编辑/etc/neutron/l3_agent.ini文件
# 第 3 层 (L3) 代理为自助服务虚拟网络提供路由和 NAT 服务
vi /etc/neutron/l3_agent.ini
[DEFAULT]
# ...
interface_driver = linuxbridge
# ，配置 Linux 桥接接口驱动程序

# DHCP 代理为虚拟网络提供 DHCP 服务
vi /etc/neutron/dhcp_agent.ini
[DEFAULT]
# ...
interface_driver = linuxbridge
# 配置 Linux 网桥接口驱动程序
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
# Dnsmasq DHCP 驱动程序
enable_isolated_metadata = true
# 启用隔离元数据


vi /etc/neutron/metadata_agent.ini
[DEFAULT]
# ...
nova_metadata_host = controller
# 元数据主机设置
metadata_proxy_shared_secret = METADATA_SECRET
# 元数据密码
# METADATA_SECRET 为 元数据 的密钥


vi /etc/nova/nova.conf
# 配置访问参数、启用元数据代理和配置密钥
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
# NEUTRON_PASS  为 neutron 服务的密码
# METADATA_SECRET 为上边设置的元数据密钥

:wq


ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
# 网络服务初始化脚本需要一个 /etc/neutron/plugin.ini指向 ML2 插件配置文件的符号链接

su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
# 填充数据库

systemctl restart openstack-nova-api
# 重启计算 API 服务

systemctl status openstack-nova-api


# 启动网络服务并将它们配置为在系统启动时启动
systemctl enable  --now neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service  neutron-l3-agent.service

systemctl status neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service   neutron-metadata-agent.service  neutron-l3-agent.service


# 等待 计算节点 安装 neutron 后进行验证


```

### 块存储服务Cinder

```
mysql -u root -p
# MARIADB_PASS

MariaDB [(none)]> CREATE DATABASE cinder;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'CINDER_DBPASS';
# CINDER_DBPASS 为 cinder 数据库账户密码

MariaDB [(none)]> exit


source admin-openrc.sh

openstack user create --domain default --password-prompt cinder
# 将要求输入 cinder 服务的密码 CINDER_PASS
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a5fb965da094432dbf455b45549a6a24 |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

openstack role add --project service --user cinder admin

openstack service create --name cinderv3 \
  --description "OpenStack Block Storage" volumev3
 +-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 343cd790d86f4be6a9ef1bca61b335e9 |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | b915d6f25c484e06ba4cdf0a38cd11dd         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 343cd790d86f4be6a9ef1bca61b335e9         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 4ee3b7d6c0234f7b8695a8b9e5cabc10         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 343cd790d86f4be6a9ef1bca61b335e9         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 750940653931468397c1618429dcf366         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 343cd790d86f4be6a9ef1bca61b335e9         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+


yum install openstack-cinder -y
##########################################


vi /etc/cinder/cinder.conf

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
# 配置RabbitMQ 消息队列访问
auth_strategy = keystone
# 配置身份服务访问
my_ip = 192.168.200.23
# 控制节点管理网络 IP

[database]
# 配置数据库访问
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
# CINDER_DBPASS 为数据库 Cinder 账户密码

[keystone_authtoken]

# 配置身份服务访问
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
# CINDER_PASS 为 Cinder 服务密码

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
# 锁定路径

:wq


su -s /bin/sh -c "cinder-manage db sync" cinder
# 填充块存储数据库

vi /etc/nova/nova.conf
[cinder]
os_region_name = RegionOne


systemctl restart openstack-nova-api.service
# 重启计算 API 服务

systemctl status openstack-nova-api.service

systemctl enable --now openstack-cinder-api.service openstack-cinder-scheduler.service
# 启动块存储服务并将它们配置为在系统启动时启动

systemctl status openstack-cinder-api.service openstack-cinder-scheduler.service

# 等待块存储节点 Cinder 安装完成后进行验证
```


### Web 管理页面（Dashboard）horizon

```
yum install openstack-dashboard -y
# 安装软件包



vi /etc/openstack-dashboard/local_settings
OPENSTACK_HOST = "controller"
# controller配置仪表板以在节点上使用 OpenStack 服务
ALLOWED_HOSTS = ['*']
# 允许您的主机访问仪表板
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
# 配置memcached会话存储服
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
# 启用身份 API 版本
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
# 启用对域的支持
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
# 配置 API 版本
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
# 默认域
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
# 默认域用户
TIME_ZONE = "Asia/Shanghai"
# 时区
# 有则修改没有则添加

:wq




python3 /usr/share/openstack-dashboard/manage.py make_web_conf --apache > /etc/httpd/conf.d/openstack-dashboard.conf
# 重新生成openstack-dashboard.conf

systemctl restart httpd
# 重启服务

systemctl status httpd


# 验证
# 访问 http://部署 Dashboard 的控制节点 ip
# 登录用户密码 可使用 admin 或 user
# 域名 使用 RegionOne
```

## 计算节点

### nova服务

```
yum install openstack-nova-compute -y



vi /etc/nova/nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
# 启用计算和元数据 API
transport_url = rabbit://openstack:RABBIT_PASS@controller
# 配置RabbitMQ消息队列访问
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
# 设置主机ip地址
compute_driver=libvirt.LibvirtDriver
log_file = /var/log/nova/nova-computer.log
# MANAGEMENT_INTERFACE_IP_ADDRESS 替换为 管理网络 IP 地址

[api]
# ...
# 配置身份服务访问
auth_strategy = keystone

[keystone_authtoken]
# 配置身份服务访问
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
# ...
# 启用和配置远程控制台访问
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://ManagementIP:6080/vnc_auto.html
# 将 ManagementIP 修改为控制节点管理网络 IP 

[glance]
# ...
api_servers = http://controller:9292
# 配置图像服务 API 

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
# 配置锁定路径

[placement]
# 配置 Placement API
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
# PLACEMENT_PASS 为 Placement 服务密码

[neutron]
# 配置网络设置
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
# NEUTRON_PASS 为 Neutron 服务密码

:wq


egrep -c '(vmx|svm)' /proc/cpuinfo
# 如果返回值大于 1 则说明已经开启硬件虚拟化，无需配置 qemu
# 如等于 0 ，则需要配置 qemu 以代替默认的 kvm
vi /etc/nova/nova.conf
[libvirt]
# ...
virt_type = qemu

# 以上配置仅当 egrep -c '(vmx|svm)' /proc/cpuinfo 结果为 0 时才进行配置

mkdir -p /usr/lib/python3.6/site-packages/instances

chmod +777 /usr/lib/python3.6/site-packages/instances

systemctl enable libvirtd.service openstack-nova-compute.service --now

systemctl status libvirtd.service openstack-nova-compute.service


# 在控制节点执行验证
source admin-openrc.sh
openstack compute service list --service nova-compute
确认数据库中有计算主机
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+
| ID                                   | Binary       | Host     | Zone | Status  | State | Updated At                 |
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+
| 542d6911-ba76-450a-b235-014bb722097b | nova-compute | computer | nova | enabled | up    | 2022-07-27T08:34:59.000000 |
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+

su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
# 发现计算主机
# Found 2 cell mappings.
# Skipping cell0 since it does not contain hosts.
# Getting computes from cell 'cell1': ab6ff38c-d05a-40b9-bbb6-8306a048e38e
# Checking host mapping for compute host 'computer': f5c0a1d3-4380-4d5d-8579-4667998ca06a
# Creating host mapping for compute host 'computer': f5c0a1d3-4380-4d5d-8579-4667998ca06a
# Found 1 unmapped computes in cell: ab6ff38c-d05a-40b9-bbb6-8306a048e38e

openstack compute service list
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+
| 36fcb09d-1ef3-4e18-b5f5-26671d900e39 | nova-conductor | controller | internal | enabled | up    | 2022-07-27T08:37:13.000000 |
| 8f2e62b2-92cd-4a19-a25a-cbdebac5670f | nova-scheduler | controller | internal | enabled | up    | 2022-07-27T08:37:13.000000 |
| 542d6911-ba76-450a-b235-014bb722097b | nova-compute   | computer   | nova     | enabled | up    | 2022-07-27T08:37:09.000000 |
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+


nova-status upgrade check
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Cells v2                           |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Placement API                      |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Cinder API                         |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy Scope-based Defaults        |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Older than N-1 computes            |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: hw_machine_type unset              |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+

```



### 网络服务Neutron

```
yum install openstack-neutron-linuxbridge ebtables ipset -y 

# 安装组件


vi /etc/neutron/neutron.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
# 配置RabbitMQ 消息队列访问
# RABBIT_PASS  为 控制节点 消息队列 密码
auth_strategy = keystone


# keystone访问
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
# NEUTRON_PASS  为控制节点 neutron 服务密码
# # keystone访问


[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
# 锁定目录



vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
# 提供者虚拟网络映射到提供者物理网络接口
# PROVIDER_INTERFACE_NAME 为 计算节点 服务提供网络对应的网卡名

[vxlan]
enable_vxlan = true
# 启用 VXLAN 覆盖网络
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
# 配置处理覆盖网络的物理网络接口的 IP 地址
l2_population = true
# 启用第 2 层
# OVERLAY_INTERFACE_IP_ADDRESS  为 计算节点 管理网络的 IP 地址

[securitygroup]
# ...
enable_security_group = true
# 启用安全组
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
# 配置 Linux 网桥 iptables 防火墙驱动程序
:wq


modprobe br_netfilter

cat >>/etc/rc.sysinit<<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF

echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules

chmod 755 /etc/sysconfig/modules/br_netfilter.modules
# sysctl通过验证确保您的 Linux 操作系统内核支持网桥过滤器
sysctl -a | grep net.bridge.bridge-nf-call
# net.bridge.bridge-nf-call-arptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1
# net.bridge.bridge-nf-call-iptables = 1

vi /etc/nova/nova.conf
[neutron]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
# NEUTRON_PASS 为 Neutron 服务密码
# 配置访问参数

# 重启服务并开机自启动
systemctl restart openstack-nova-compute.service

systemctl enable neutron-linuxbridge-agent.service --now

systemctl status neutron-linuxbridge-agent.service

# 验证
# 控制节点执行
source admin-openrc.sh
# 检测
openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 553e6d15-f6c0-4d3e-9e75-7d49adaa88a6 | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 7ac38a45-d2e6-48db-8043-ae267978d825 | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| b10f100f-d9fb-4675-8cc2-f00e6140a553 | Linux bridge agent | compute    | None              | :-)   | UP    | neutron-linuxbridge-agent |
| b40bb34d-40b5-48fb-aced-e60ff7972601 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| c8d902f0-a18f-41df-be00-ab96a2f8111d | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
# 确保以上五个 Agent 都为 :-) 及 UP
```

## 块存储节点

### 块存储服务Cinder

```
yum install lvm2 device-mapper-persistent-data -y
# 安装 LVM 包

systemctl enable lvm2-lvmetad.service --now
# 如显示不存在则说明系统默认安装了 lvm  以上步骤可忽略

fdisk -l
# 查看 块存储 所部署的 磁盘 代号

pvcreate /dev/sdb
# 创建 LVM 物理卷
# Physical volume "/dev/sdb" successfully created.

vgcreate cinder-volumes /dev/sdb
# 创建 LVM 卷组cinder-volumes
# Volume group "cinder-volumes" successfully created
# sdb 为划分给块存储使用的磁盘
# 如有多个磁盘，则需重复以上两个命令



vi /etc/lvm/lvm.conf
# 添加一个接受 /dev/sdb设备并拒绝所有其他设备的过滤器
devices {
	...
	filter = [ "a/sdb/", "r/.*/"]
}
# 如有多个磁盘，则将磁盘编号以固定格式添加到过滤设备中，例如有两个磁盘 sdb sdc ，则为 filter = [ "a/sdb/", "a/sdc/","r/.*/"]


yum install openstack-cinder targetcli -y
# 安装软件包


vi /etc/cinder/cinder.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
# 配置RabbitMQ 消息队列访问
auth_strategy = keystone
# 配置keystone认证
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
enabled_backends = lvm
# 启用lvm
glance_api_servers = http://controller:9292
# MANAGEMENT_INTERFACE_IP_ADDRESS  为块存储节点 管理网络 的接口IP

[database]
# ...
# 配置数据库访问
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
# CINDER_DBPASS 为数据库 Cinder 账户密码

[keystone_authtoken]
# 配置keystone访问
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
# CINDER_PASS 为 cinder 数据库账户密码

[lvm]
# 使用 LVM 驱动程序、卷组、iSCSI 协议和适当的 iSCSI 服务[lvm]配置 LVM 后端
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
# [lvm]  没有则新建

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
# 锁定目录

:wq


systemctl enable openstack-cinder-volume.service target.service --now
# 启动并开机自启动


systemctl status openstack-cinder-volume.service target.service

# 验证
# 控制节点执行
source admin-openrc.sh
# 验证
openstack volume service list
+------------------+------------+------+---------+-------+----------------------------+
| Binary           | Host       | Zone | Status  | State | Updated At                 |
+------------------+------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller | nova | enabled | up    | 2022-09-19T06:23:44.000000 |
| cinder-volume    | block@lvm  | nova | enabled | up    | 2022-09-19T06:23:44.000000 |
+------------------+------------+------+---------+-------+----------------------------+
```





































































