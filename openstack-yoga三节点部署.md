## 系统环境配置

- 所有节点都要做

### 系统

- OpenStack大版本对centos系统的要求对应关系如下：

| OpenStack 版本 | CentOS 版本
----------------|-------------
  Train 以及更早|     7
  Ussuri 和 Victoria| 8
  Wallaby 到 Yoga | Stream 8

- CentOS stream 8选择最小化安装。

### 配置网络、主机名

- controller节点和compute节点必须都为两块网卡

- 修改和添加/etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME文件。替换INTERFACE_NAME为实际的接口名称

#### controller节点

配置网络:

- 将第一个接口配置为管理接口(有ip)：

- 将第二个接口配置为提供者接口：
```
	DEVICE=INTERFACE_NAME
	TYPE=Ethernet
	ONBOOT="yes"
	BOOTPROTO="none"
```
> 提供者接口不能配置ip

配置主机名：
```
# hostnamectl set-hostname controller
按ctrl+d 退出  重新登陆

```

#### compute节点

- 将第一个接口配置为管理接口(有ip)：

- 将第二个接口配置为提供者接口：
```
	DEVICE=INTERFACE_NAME
	TYPE=Ethernet
	ONBOOT="yes"
	BOOTPROTO="none"
```
> 提供者接口不能配置ip

配置主机名：
```
# hostnamectl set-hostname compute
按ctrl+d 退出  重新登陆

```

#### 块存储节点


配置主机名：
```
# hostnamectl set-hostname block
按ctrl+d 退出  重新登陆
```

### 配置hosts：
```
cat >>/etc/hosts<<EOF
10.0.0.11       controller
10.0.0.31       compute
10.0.0.41       block
EOF


# 各节点测试能否互通
ping controller
ping computer
ping block

```

### yum源修改
```
 mkdir /etc/yum.repos.d/bak
 sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.aliyun.com/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stre*.repo
 mv /etc/yum.repos.d/CentOS-Stream-*.bak /etc/yum.repos.d/bak/
yum clean all
yum makecache

```

### 关闭防火墙和selinux
```
systemctl stop firewalld && systemctl disable firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0

```
### 配置时间同步服务
```
# 控制节点

	timedatectl set-timezone Asia/Shanghai

# 各节点


yum install chrony -y

cp /etc/chrony.conf  /etc/chrony.conf.bak


# 控制节点
vi /etc/chrony.conf 
#pool 2.centos.pool.ntp.org iburst
server time1.aliyun.com iburst
allow 10.0.0.0/24

# 计算节点 块存储节点
vi /etc/chrony.conf 
#pool 2.centos.pool.ntp.org iburst
server controller iburst

# 各节点
systemctl enable --now chronyd

systemctl status chronyd

chronyc sources
# 控制节点
# MS Name/IP address         Stratum Poll Reach LastRx Last sample               
# ===============================================================================
# ^* 203.107.6.88                  2   6    17     1   +250us[ +296us] +/-   36ms
# 计算节点及块存储节点
# MS Name/IP address         Stratum Poll Reach LastRx Last sample               
# ===============================================================================
# ^* controller                    3   6    17     3  -2592ns[  -21us] +/-   35ms
# ^* 代表同步成功


```

### 配置openstack软件包

```
yum install centos-release-openstack-yoga -y

yum config-manager --set-enabled powertools
yum upgrade -y
yum install python3-openstackclient -y

yum install openstack-selinux –y

```

## 开始安装openstack

### 数据库安装配置

```
yum install mariadb mariadb-server python3-PyMySQL  -y
cp /etc/my.cnf.d/openstack.cnf /etc/my.cnf.d/openstack.cnf.bak
# 没有则新建
vi /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 10.0.0.10
# 根据控制节点管理网络 IP 修改
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
:wq
systemctl enable mariadb --now
systemctl status mariadb
mysql_secure_installation
Enter current password for root (enter for none): 回车
Set root password? [Y/n] y
# 将要求输入数据库 root 账户密码 MARIADB_PASS
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y

# 验证
mysql -u root -p
```

### 消息队列服务 (rabbitmq)
```
yum install rabbitmq-server -y
systemctl enable rabbitmq-server --now
systemctl status rabbitmq-server
rabbitmqctl add_user openstack RABBIT_PASS
# 注意将 RABBIT_PASS  修改为 消息队列密码
rabbitmqctl set_permissions openstack ".*" ".*" ".*"


```

### memcached内存对象缓存

```
yum install memcached python3-memcached -y
cp /etc/sysconfig/memcached /etc/sysconfig/memcached.bak
vi /etc/sysconfig/memcached
OPTIONS="-l 127.0.0.1,::1,controller"
# 如启动出现绑定失败问题，则修改为
# OPTIONS="-l 127.0.0.1,::1,管理网络IP地址"
:wq
systemctl enable memcached --now
systemctl status memcached
```

### etcd

```
# yum install etcd
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak

vi /etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.0.0.10:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.10:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.10:2379"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.10:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
# 注意 controller 为 控制节点的 hostname   10.0.0.10 为控制节点管理网络的 IP  确保一致性
systemctl enable etcd --now
systemctl status etcd
```


## controller 节点

### 配置管理账户

```
vi admin-openrc.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

# OS_USERNAME  登录 OpenStack 服务的用户名
# OS_PASSWORD  登录 OpenStack 服务的用户密码
# OS_PROJECT_NAME 登录时进入的项目名
# OS_USER_DOMAIN_NAME  登录时进入的域名
# OS_PROJECT_DOMAIN_NAME  登录时进入的项目域名
# OS_AUTH_URL 指定 Keystone（身份认证服务）的 URL  
# 如未部署 DNS 服务器，则需要在 hosts中指定 controller 映射，或将 controller 用控制节点 IP 替代
# OS_IDENTITY_API_VERSION 身份认证服务的 API 版本号 
# OS_IMAGE_API_VERSION 镜像服务的 API 版本号
```

### 安装keystone认证服务

- 数据库设置

```
$ mysql -u root -p
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhos
t' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \IDENTIFIED BY 'KEYSTONE_DBPASS';

# 替换KEYSTONE_DBPASS为合适的密码。

```

- 安装配置keystone组件

```
yum install openstack-keystone httpd python3-mod_wsgi -y

cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
vi /etc/keystone/keystone.conf
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
# KEYSTONE_DBPASS  为 Keystone 数据库账户密码
[token]
# ...
provider = fernet
:wq
su -s /bin/sh -c "keystone-manage db_sync" keystone
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
# ADMIN_PASS 为 admin 账户密码
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak
vi /etc/httpd/conf/httpd.conf
ServerName controller
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl enable httpd  --now
systemctl status httpd
source admin-openrc.sh
# service 项目 创建在 default 用于 OpenStack 服务
openstack project create --domain default \
  --description "Service Project" service
# +-------------+----------------------------------+
# | Field       | Value                            |
# +-------------+----------------------------------+
# | description | Service Project                  |
# | domain_id   | default                          |
# | enabled     | True                             |
# | id          | 9696d33d99334266a7dcc735ad068550 |
# | is_domain   | False                            |
# | name        | service                          |
# | options     | {}                               |
# | parent_id   | default                          |
# | tags        | []                               |
# +-------------+----------------------------------+
# 创建一个 RegionOne 域名作为后续云实例创建域名
openstack domain create --description "RegionOne Domain" RegionOne
#  在 RegionOne 域中创建一个 Yoga 项目
openstack project create --domain RegionOne \
  --description "Yoga Project" Yoga
# 在 RegionOne 域中创建普通用户 user_dog 
openstack user create --domain RegionOne \
  --password-prompt user_dog
# 创建普通用户 user_dog  的规则 user_dog_role
openstack role create user_dog_role
# 将规则与用户绑定
openstack role add --project Yoga --user user_dog user_dog_role


# 注：可以重复上边步骤以创建更多项目、用户及规则
```

- 验证
```
# 卸载当前用户环境
unset OS_AUTH_URL OS_PASSWORD
# 验证 admin 用户可用性
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
# 输入后将要求输入 管理员 admin 的密码
# 返回  token 信息则服务正常
# 验证 user_dog 用户可用性
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name RegionOne --os-user-domain-name RegionOne \
  --os-project-name Yoga --os-username user_dog token issue
source admin-openrc.sh
# 列举当前所有域名
openstack domain list
# +----------------------------------+-----------+---------+--------------------+
# | ID                               | Name      | Enabled | Description        |
# +----------------------------------+-----------+---------+--------------------+
# | d1eb84f97aa14741a3911f76e0bad1e7 | RegionOne | True    | RegionOne Domain   |
# | default                          | Default   | True    | The default domain |
# +----------------------------------+-----------+---------+--------------------+
```

### 安装glance镜像服务

- 数据库设置

```
mysql -u root -p
#  MARIADB_PASS
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
将 GLANCE_DBPASS 替换为 glance 服务的密码
MariaDB [(none)]> exit


```

` 安装配置glance

```

source admin-openrc.sh
openstack user create --domain default --password-prompt glance
# User Password: GLANCE_PASS
# Repeat User Password: GLANCE_PASS
# 为 Glance 用户添加 admin 规则到系统项目 service
openstack role add --project service --user glance admin
# 没有输出内容
# 为 Glance 添加管理镜像的服务
openstack service create --name glance \
  --description "OpenStack Image" image
# +-------------+----------------------------------+
# | Field       | Value                            |
# +-------------+----------------------------------+
# | description | OpenStack Image                  |
# | enabled     | True                             |
# | id          | 97a0f2713e504751a263d852bca8c5c6 |
# | name        | glance                           |
# | type        | image                            |
# +-------------+----------------------------------+
# 为 RegionOne 域名添加服务接口
openstack endpoint create --region RegionOne \
  image public http://controller:9292
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | aadbbde7bd9948258dc5f35acdbee92a |
# | interface    | public                           |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 97a0f2713e504751a263d852bca8c5c6 |
# | service_name | glance                           |
# | service_type | image                            |
# | url          | http://controller:9292           |
# +--------------+----------------------------------+
openstack endpoint create --region RegionOne \
  image internal http://controller:9292
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | 4a3ca38a9f50426d9fa215e4277ad4d6 |
# | interface    | internal                         |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 97a0f2713e504751a263d852bca8c5c6 |
# | service_name | glance                           |
# | service_type | image                            |
# | url          | http://controller:9292           |
# +--------------+----------------------------------+
openstack endpoint create --region RegionOne \
  image admin http://controller:9292
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | 155f303412c44aad8e375ef982e505da |
# | interface    | admin                            |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 97a0f2713e504751a263d852bca8c5c6 |
# | service_name | glance                           |
# | service_type | image                            |
# | url          | http://controller:9292           |
# +--------------+----------------------------------+
pip3 install boto3
yum install openstack-glance -y
cp /etc/glance/glance-api.conf  /etc/glance/glance-api.conf.bak
vi /etc/glance/glance-api.conf
[DEFAULT]
use_keystone_quotas = True
log_file = /var/log/glance/glance.log
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
# GLANCE_DBPASS 为 Glance 服务的数据库账户密码
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
password = GLANCE_PASS
service_token_roles_required = true
# GLANCE_DBPASS 为 Glance 服务的数据库账户密码
[paste_deploy]
# ...
flavor = keystone
[glance_store]
# ...
# stores = file,http
# default_store = file
default_backend = {'store_one': 'http', 'store_two': 'file'}
filesystem_store_datadir = /var/lib/glance/images/
# 具体多后端配置信息见官方链接 https://docs.openstack.org/glance_store/yoga/reference/api/glance_store.multi_backend.html 
# ######注：可忽略#####
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = admin
# 使用 admin
system_scope = all
password = ADMIN_PASS
# 使用 admin  用户的密码
endpoint_id = ENDPOINT_ID    
# 使用 openstack endpoint list 查询  glance 服务 对应 admin 用户的 endpoint_id
region_name = RegionOne
:wq
# 同步 Glance 数据到数据库
su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api  --now
systemctl status openstack-glance-api
```


- 配额限制（可选）

如果决定在 Glance 中使用每租户配额，则必须首先在 Keystone 中注册限制：

```
# 上传镜像的大小
$ openstack --os-cloud devstack-system-admin registered limit create \
  --service glance --default-limit 1000 --region RegionOne image_size_total
# 镜像的总数
openstack registered limit create \
  --service glance --default-limit 100 --region RegionOne image_count_total
# 镜像的上传数量
openstack registered limit create \
  --service glance --default-limit 100 --region RegionOne image_count_uploading

```


- 验证服务

```
source admin-openrc.sh
# 下载内核系统
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
glance image-create --name "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
# +------------------+----------------------------------------------------------------------------------+
# | Property         | Value                                                                            |
# +------------------+----------------------------------------------------------------------------------+
# | checksum         | b874c39491a2377b8490f5f1e89761a4                                                 |
# | container_format | bare                                                                             |
# | created_at       | 2022-07-24T17:50:11Z                                                             |
# | disk_format      | qcow2                                                                            |
# | id               | 4e022193-03c2-40c4-872f-0adb606f31e4                                             |
# | min_disk         | 0                                                                                |
# | min_ram          | 0                                                                                |
# | name             | cirros                                                                           |
# | os_hash_algo     | sha512                                                                           |
# | os_hash_value    | 6b813aa46bb90b4da216a4d19376593fa3f4fc7e617f03a92b7fe11e9a3981cbe8f0959dbebe3622 |
# |                  | 5e5f53dc4492341a4863cac4ed1ee0909f3fc78ef9c3e869                                 |
# | os_hidden        | False                                                                            |
# | owner            | e4bf08c8bd814c288852ec8bd48936d4                                                 |
# | protected        | False                                                                            |
# | size             | 16300544                                                                         |
# | status           | active                                                                           |
# | tags             | []                                                                               |
# | updated_at       | 2022-07-24T17:50:11Z                                                             |
# | virtual_size     | 117440512                                                                        |
# | visibility       | public                                                                           |
# +------------------+----------------------------------------------------------------------------------+
openstack image list
# +--------------------------------------+--------+--------+
# | ID                                   | Name   | Status |
# +--------------------------------------+--------+--------+
# | 4e022193-03c2-40c4-872f-0adb606f31e4 | cirros | active |
# +--------------------------------------+--------+--------+

```

### 安置服务placement

- 数据库设置

```
mysql -u root -p
# MARIADB_PASS
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
#PLACEMENT_DBPASS 为 placement 服务的密码
MariaDB [(none)]> exit

```

- 安装配置placement

```
openstack user create --domain default --password-prompt placement
# 执行后将要求输入 placement 服务的密码  PLACEMENT_PASS
# User Password: PLACEMENT_PASS
# Repeat User Password: PLACEMENT_PASS
# +---------------------+----------------------------------+
# | Field               | Value                            |
# +---------------------+----------------------------------+
# | domain_id           | default                          |
# | enabled             | True                             |
# | id                  | d6257b9730fd45c6864a5092d237a6a5 |
# | name                | placement                        |
# | options             | {}                               |
# | password_expires_at | None                             |
# +---------------------+----------------------------------+
openstack role add --project service --user placement admin
openstack service create --name placement \
  --description "Placement API" placement
# +-------------+----------------------------------+
# | Field       | Value                            |
# +-------------+----------------------------------+
# | description | Placement API                    |
# | enabled     | True                             |
# | id          | 3fe738e12ef24c59ad98fab578b263ca |
# | name        | placement                        |
# | type        | placement                        |
# +-------------+----------------------------------+

openstack endpoint create --region RegionOne \
  placement public http://controller:8778
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | ece91ac8a6054ce8920392fd88c88c1a |
# | interface    | public                           |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 3fe738e12ef24c59ad98fab578b263ca |
# | service_name | placement                        |
# | service_type | placement                        |
# | url          | http://controller:8778           |
# +--------------+----------------------------------+

openstack endpoint create --region RegionOne \
  placement internal http://controller:8778
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | 1d3931e0d4ad47ee9e38c0c66736f87f |
# | interface    | internal                         |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 3fe738e12ef24c59ad98fab578b263ca |
# | service_name | placement                        |
# | service_type | placement                        |
# | url          | http://controller:8778           |
# +--------------+----------------------------------+
openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | 37bd6f1c52454a87909f039c7ff5b4fb |
# | interface    | admin                            |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | 3fe738e12ef24c59ad98fab578b263ca |
# | service_name | placement                        |
# | service_type | placement                        |
# | url          | http://controller:8778           |
# +--------------+----------------------------------+
yum install openstack-placement-api -y
cp /etc/placement/placement.conf /etc/placement/placement.conf.bak
vi /etc/placement/placement.conf
[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
# PLACEMENT_DBPASS 为 placement 服务的数据库账户密码

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
password = PLACEMENT_PASS
# PLACEMENT_PASS 为 placement 服务的密码
:wq
su -s /bin/sh -c "placement-manage db sync" placement
systemctl restart httpd
systemctl status httpd
cp /etc/placement/policy.json /etc/placement/policy.json.bak
oslopolicy-convert-json-to-yaml --namespace placement \
  --policy-file /etc/placement/policy.json \
 --output-file /etc/placement/policy.yaml
rm -f /etc/placement/policy.json

```

- 验证服务

```
source admin-openrc.sh
placement-status upgrade check
# +-------------------------------------------+
# | Upgrade Check Results                     |
# +-------------------------------------------+
# | Check: Missing Root Provider IDs          |
# | Result: Success                           |
# | Details: None                             |
# +-------------------------------------------+
# | Check: Incomplete Consumers               |
# | Result: Success                           |
# | Details: None                             |
# +-------------------------------------------+
# | Check: Policy File JSON to YAML Migration |
# | Result: Success                           |
# | Details: None                             |
# +-------------------------------------------+
yum install python3-osc-placement -y
cp /etc/httpd/conf.d/00-placement-api.conf /etc/httpd/conf.d/00-placement-api.conf.bak
vi /etc/httpd/conf.d/00-placement-api.conf
# 在 listen 8778 下一行处添加
<Files "placement-api">
    Require all granted
</Files>
:wq
systemctl restart httpd
systemctl status httpd
# 验证
openstack --os-placement-api-version 1.2 resource class list --sort-column name
# +----------------------------------------+
# | name                                   |
# +----------------------------------------+
# | DISK_GB                                |
# | FPGA                                   |
# | IPV4_ADDRESS                           |
# | MEMORY_MB                              |
# | MEM_ENCRYPTION_CONTEXT                 |
# | NET_BW_EGR_KILOBIT_PER_SEC             |
# | NET_BW_IGR_KILOBIT_PER_SEC             |
# | NET_PACKET_RATE_EGR_KILOPACKET_PER_SEC |
# | NET_PACKET_RATE_IGR_KILOPACKET_PER_SEC |
# | NET_PACKET_RATE_KILOPACKET_PER_SEC     |
# | NUMA_CORE                              |
# | NUMA_MEMORY_MB                         |
# | NUMA_SOCKET                            |
# | NUMA_THREAD                            |
# | PCI_DEVICE                             |
# | PCPU                                   |
# | PGPU                                   |
# | SRIOV_NET_VF                           |
# | VCPU                                   |
# | VGPU                                   |
# | VGPU_DISPLAY_HEAD                      |
# +----------------------------------------+
openstack --os-placement-api-version 1.6 trait list --sort-column name
# +---------------------------------------+
# | name                                  |
# +---------------------------------------+
# | COMPUTE_ACCELERATORS                  |
# | COMPUTE_ARCH_AARCH64                  |
# | COMPUTE_ARCH_MIPSEL                   |
# | COMPUTE_ARCH_PPC64LE                  |
# | COMPUTE_ARCH_RISCV64                  |
# | COMPUTE_ARCH_S390X                    |
# | COMPUTE_ARCH_X86_64                   |
# | COMPUTE_DEVICE_TAGGING                |
# ...
```

### 计算服务nova

- 数据库设置

```
mysql -u root -p
# MARIADB_PASS
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
# NOVA_DBPASS 为 nova 服务的密码
MariaDB [(none)]> exit
```

- 安装配置nova

```
source admin-openrc.sh
openstack user create --domain default --password-prompt nova
# 将要求输入 nova 服务的密码  NOVA_PASS
# User Password: NOVA_PASS
# Repeat User Password: NOVA_PASS
# +---------------------+----------------------------------+
# | Field               | Value                            |
# +---------------------+----------------------------------+
# | domain_id           | default                          |
# | enabled             | True                             |
# | id                  | ea8cc01ac5094751bdac3c49ead28bec |
# | name                | nova                             |
# | options             | {}                               |
# | password_expires_at | None                             |
# +---------------------+----------------------------------+
openstack role add --project service --user nova admin
openstack service create --name nova \
  --description "OpenStack Compute" compute
# +-------------+----------------------------------+
# | Field       | Value                            |
# +-------------+----------------------------------+
# | description | OpenStack Compute                |
# | enabled     | True                             |
# | id          | b427b5b5434f4edba1dd157a01a45d12 |
# | name        | nova                             |
# | type        | compute                          |
# +-------------+----------------------------------+
openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | e8689643fe714c459c2b7d1b885ec72d |
# | interface    | public                           |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | b427b5b5434f4edba1dd157a01a45d12 |
# | service_name | nova                             |
# | service_type | compute                          |
# | url          | http://controller:8774/v2.1      |
# +-------------+----------------------------------+
openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | d6ac010e63c8455f98ea04f6886adfb5 |
# | interface    | internal                         |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | b427b5b5434f4edba1dd157a01a45d12 |
# | service_name | nova                             |
# | service_type | compute                          |
# | url          | http://controller:8774/v2.1      |
# +--------------+----------------------------------+
openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
# +--------------+----------------------------------+
# | Field        | Value                            |
# +--------------+----------------------------------+
# | enabled      | True                             |
# | id           | 695e0d4dba934af2844fef47488630bc |
# | interface    | admin                            |
# | region       | RegionOne                        |
# | region_id    | RegionOne                        |
# | service_id   | b427b5b5434f4edba1dd157a01a45d12 |
# | service_name | nova                             |
# | service_type | compute                          |
# | url          | http://controller:8774/v2.1      |
# +--------------+----------------------------------+
yum install -y \
    openstack-nova-api \
    openstack-nova-scheduler \
    openstack-nova-conductor \
    openstack-nova-novncproxy \
    iptables
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
vi /etc/nova/nova.conf
[DEFAULT]
# …
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
# RABBIT_PASS rabbitmq 密码
my_ip = 10.0.0.10
# 控制节点控制网络的 IP
log_file = /var/log/nova/nova-controller.log
rootwrap_config = /etc/nova/rootwrap.conf
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
# NOVA_DBPASS 为数据库 Nova 账户密码
[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
# NOVA_DBPASS 为数据库 Nova 账户密码
[api]
# ...
auth_strategy = keystone
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
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://controller:9292
[oslo_concurrency]
# ...
lock_path = /var/run/nova
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
:wq
su -s /bin/sh -c "nova-manage api_db sync" nova
cp /etc/nova/policy.json /etc/nova/policy.json.bak
oslopolicy-convert-json-to-yaml --namespace nova \
  --policy-file /etc/nova/policy.json \
  --output-file /etc/nova/policy.yaml
rm -f /etc/nova/policy.json
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
# --transport-url not provided in the command line, using the value [DEFAULT]/transport_url from the configuration file
# --database_connection not provided in the command line, using the value [database]/connection from the configuration file
# ab6ff38c-d05a-40b9-bbb6-8306a048e38e
# 如有以上提示请忽略，cell 将以 nova.conf 配置文件内的地址进行创建
su -s /bin/sh -c "nova-manage db sync" nova
# 验证
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |              Transport URL               |               Database Connection               | Disabled |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                  none:/                  | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | ab6ff38c-d05a-40b9-bbb6-8306a048e38e | rabbit://openstack:****@controller:5672/ |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
systemctl enable --now \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
systemctl status \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```


### neutron网络服务

- 数据库设置
```
mysql -u root -p
# MARIADB_PASS
MariaDB [(none)] CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
# NEUTRON_DBPASS 为数据库 neutron 账户的密码
MariaDB [(none)]> exit
```

- 安装配置neutron
```
source admin-openrc.sh
openstack user create --domain default --password-prompt neutron
# 将要求输入密码 此密码为 neutron 服务的密码  NEUTRON_PASS
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 33703cb73a484af4b6ec741e2c02e348 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
openstack role add --project service --user neutron admin
openstack service create --name neutron \
  --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 76470167718f4710a721374a929ab204 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

openstack endpoint create --region RegionOne \
  network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1b706912b617465892e1a7e1e5d3a924 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 76470167718f4710a721374a929ab204 |
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
| id           | 27a6cddb66e443fa801a72b56bacb5c4 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 76470167718f4710a721374a929ab204 |
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
| id           | 9fbeeec5c30a4c7e86ad3052c78c4084 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 76470167718f4710a721374a929ab204 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
# 选择安装 大二层 网络
yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables -y
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
vi /etc/neutron/neutron.conf
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
# NEUTRON_DBPASS为 数据库 neutron 账户密码
[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
# RABBIT_PASS 为 消息队列密码
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
# NEUTRON_PASS为 neutron 服务密码
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
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
:wq
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
vi /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true
# 没有则添加
:wq
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
# PROVIDER_INTERFACE_NAME 为 服务提供网络所对应的网卡编号
[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
# OVERLAY_INTERFACE_IP_ADDRESS 为管理网络 控制节点的 IP  即 controller IP

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
# 没有则添加
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
sysctl -a | grep net.bridge.bridge-nf-call
# net.bridge.bridge-nf-call-arptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1
# net.bridge.bridge-nf-call-iptables = 1
cp /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.bak
vi /etc/neutron/l3_agent.ini
[DEFAULT]
# ...
interface_driver = linuxbridge
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
vi /etc/neutron/dhcp_agent.ini
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak
vi /etc/neutron/metadata_agent.ini
[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
# METADATA_SECRET 为 元数据 的密钥
vi /etc/nova/nova.conf
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
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
systemctl restart openstack-nova-api

systemctl status openstack-nova-api
systemctl enable  --now neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service  neutron-l3-agent.service
systemctl status neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service   neutron-metadata-agent.service  neutron-l3-agent.service
# 等待 计算节点 安装 neutron 后进行验证

```
### 块存储服务

- 数据库设置

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

```
- 安装cinder

```
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
| id                  | 6aebd9fadf2d4d1fa16a6dd87ed704c5 |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
openstack role add --project service --user cinder admin
openstack service create --name cinderv3 \
  --description "OpenStack Block Storage" volumev3
openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | fef484b5fc364720a7bd613fc60eb814 |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+
openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | fa2322b2a3014d8a9b23932978330f4b         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | fef484b5fc364720a7bd613fc60eb814         |
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
| id           | 34f4c7d3a2714a4ab9be1e42c319de98         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | fef484b5fc364720a7bd613fc60eb814         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+


yum install openstack-cinder -y
cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
vi /etc/cinder/cinder.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 10.0.0.10
# 控制节点管理网络 IP
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
# CINDER_DBPASS 为数据库 Cinder 账户密码

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
# CINDER_PASS 为 Cinder 服务密码

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
:wq
su -s /bin/sh -c "cinder-manage db sync" cinder
vi /etc/nova/nova.conf
[cinder]
os_region_name = RegionOne
systemctl restart openstack-nova-api.service
systemctl status openstack-nova-api.service
systemctl enable --now openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl status openstack-cinder-api.service openstack-cinder-scheduler.service
# 等待块存储节点 Cinder 安装完成后进行验证
```

### horizon （Dashboard页面管理服务）

```
yum install openstack-dashboard -y
cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.bak
vi /etc/openstack-dashboard/local_settings
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*']
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
TIME_ZONE = "Asia/Shanghai"
# 有则修改没有则添加
:wq
cp /etc/httpd/conf.d/openstack-dashboard.conf /etc/httpd/conf.d/openstack-dashboard.conf.bak
python3 /usr/share/openstack-dashboard/manage.py make_web_conf --apache > /etc/httpd/conf.d/openstack-dashboard.conf
systemctl restart httpd
systemctl status httpd
# 验证
# 访问 http://部署 Dashboard 的控制节点 ip
# 登录用户密码 可使用 admin 或 user_dog
# 域名 使用 RegionOne
```


## compute 节点

### 计算服务nova
```
yum install openstack-nova-compute -y
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak

vi /etc/nova/nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
compute_driver=libvirt.LibvirtDriver
log_file = /var/log/nova/nova-computer.log
# MANAGEMENT_INTERFACE_IP_ADDRESS 替换为 管理网络 IP 地址
[api]
# ...
auth_strategy = keystone

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
[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://ManagementIP:6080/vnc_auto.html
# 将 ManagementIP 修改为控制节点管理网络 IP 
[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

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
# PLACEMENT_PASS 为 Placement 服务密码
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
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+
| ID                                   | Binary       | Host     | Zone | Status  | State | Updated At                 |
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+
| 542d6911-ba76-450a-b235-014bb722097b | nova-compute | computer | nova | enabled | up    | 2022-07-27T08:34:59.000000 |
+--------------------------------------+--------------+----------+------+---------+-------+----------------------------+

su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
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

openstack catalog list
+-----------+-----------+----------------------------------------------------------------------+
| Name      | Type      | Endpoints                                                            |
+-----------+-----------+----------------------------------------------------------------------+
| placement | placement | RegionOne                                                            |
|           |           |   internal: http://controller:8778                                   |
|           |           | RegionOne                                                            |
|           |           |   admin: http://controller:8778                                      |
|           |           | RegionOne                                                            |
|           |           |   public: http://controller:8778                                     |
|           |           |                                                                      |
| keystone  | identity  | RegionOne                                                            |
|           |           |   admin: http://controller:5000/v3/                                  |
|           |           | RegionOne                                                            |
|           |           |   internal: http://controller:5000/v3/                               |
|           |           | RegionOne                                                            |
|           |           |   public: http://controller:5000/v3/                                 |
|           |           |                                                                      |
| neutron   | network   | RegionOne                                                            |
|           |           |   public: http://controller:9696                                     |
|           |           | RegionOne                                                            |
|           |           |   internal: http://controller:9696                                   |
|           |           | RegionOne                                                            |
|           |           |   admin: http://controller:9696                                      |
|           |           |                                                                      |
| glance    | image     | RegionOne                                                            |
|           |           |   admin: http://controller:9292                                      |
|           |           | RegionOne                                                            |
|           |           |   internal: http://controller:9292                                   |
|           |           | RegionOne                                                            |
|           |           |   public: http://controller:9292                                     |
|           |           |                                                                      |
| nova      | compute   | RegionOne                                                            |
|           |           |   admin: http://controller:8774/v2.1                                 |
|           |           | RegionOne                                                            |
|           |           |   internal: http://controller:8774/v2.1                              |
|           |           | RegionOne                                                            |
|           |           |   public: http://controller:8774/v2.1                                |
|           |           |                                                                      |
| cinderv3  | volumev3  | RegionOne                                                            |
|           |           |   admin: http://controller:8776/v3/e4bf08c8bd814c288852ec8bd48936d4  |
|           |           | RegionOne                                                            |
|           |           |   public: http://controller:8776/v3/e4bf08c8bd814c288852ec8bd48936d4 |
|           |           |                                                                      |
+-----------+-----------+----------------------------------------------------------------------+

openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 4e022193-03c2-40c4-872f-0adb606f31e4 | cirros | active |
+--------------------------------------+--------+--------+


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
### neutron 网络服务

```
yum install openstack-neutron-linuxbridge ebtables ipset -y 

cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
vi /etc/neutron/neutron.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
# RABBIT_PASS  为 控制节点 消息队列 密码
auth_strategy = keystone
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
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
:wq
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
# PROVIDER_INTERFACE_NAME 为 计算节点 服务提供网络对应的网卡名

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
# OVERLAY_INTERFACE_IP_ADDRESS  为 计算节点 管理网络的 IP 地址
[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
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
systemctl restart openstack-nova-compute.service
systemctl enable neutron-linuxbridge-agent.service --now
systemctl status neutron-linuxbridge-agent.service
# 验证
# 控制节点执行
source admin-openrc.sh
openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 17ad640e-4133-4cb7-b6b0-ad8fe928d2ef | Linux bridge agent | computer   | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 935d609d-2a90-4c3c-8676-a577d5f755a4 | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| af61a325-8aee-41b5-9997-6ff9a92e928e | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| c4ad2fee-93b7-4dd8-813f-0fbc2ec9dd2e | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| e4889820-19b4-4fd3-a5af-98f8586c2882 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
# 确保以上五个 Agent 都为 :-) 及 UP

```

## cinder节点

```
yum install lvm2 device-mapper-persistent-data -y
systemctl enable lvm2-lvmetad.service --now
# 如显示不存在则说明系统默认安装了 lvm  以上步骤可忽略
fdisk -l
# 查看 块存储 所部署的 磁盘 代号
pvcreate /dev/sdb
# Physical volume "/dev/sdb" successfully created.
vgcreate cinder-volumes /dev/sdb
# Volume group "cinder-volumes" successfully created
# sdb 为划分给块存储使用的磁盘
# 如有多个磁盘，则需重复以上两个命令
cp /etc/lvm/lvm.conf /etc/lvm/lvm.conf.bak
vi /etc/lvm/lvm.conf
devices {
	...
	filter = [ "a/sdb/", "r/.*/"]
}
# 如有多个磁盘，则将磁盘编号以固定格式添加到过滤设备中，例如有两个磁盘 sdb sdc ，则为 filter = [ "a/sdb/", "a/sdc/","r/.*/"]


yum install openstack-cinder targetcli -y
cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak

vi /etc/cinder/cinder.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
enabled_backends = lvm
glance_api_servers = http://controller:9292
# MANAGEMENT_INTERFACE_IP_ADDRESS  为块存储节点 管理网络 的接口IP
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
# CINDER_DBPASS 为数据库 Cinder 账户密码
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
# CINDER_PASS 为 cinder 数据库账户密码
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
# [lvm]  没有则新建

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
:wq
systemctl enable openstack-cinder-volume.service target.service --now
systemctl status openstack-cinder-volume.service target.service
# 验证
# 控制节点执行
source admin-openrc.sh
openstack volume service list
+------------------+------------+------+---------+-------+----------------------------+
| Binary           | Host       | Zone | Status  | State | Updated At                 |
+------------------+------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller | nova | enabled | up    | 2022-07-27T08:54:07.000000 |
| cinder-volume    | block@lvm  | nova | enabled | up    | 2022-07-27T08:54:04.000000 |
+------------------+------------+------+---------+-------+----------------------------+
```


## 各节点日志

### controller

```
Keystone /var/log/keystone/keystone.log
Glance /var/log/glance/glance.log
Placement /var/log/placement/placement-api.log
Nova /var/log/nova/nova-controller.log
Neutron 
Dhcp 服务  /var/log/neutron/dhcp-agent.log
Linux 网桥 /var/log/neutron/linuxbridge-agent.log
Neutron /var/log/neutron/server.log
三层网络 /var/log/neutron/l3-agent.log
元数据 /var/log/neutron/metadata-agent.log
Cinder 
/var/log/cinder/api.log     /var/log/cinder/scheduler.log
Dashboard 
Apache 登录 /var/log/httpd/access_log
Apache 错误 /var/log/httpd/error_log
Keystone 登录 /var/log/httpd/keystone_access.log
Keystone /var/log/httpd/keystone.log
Dashboard 登录 /var/log/httpd/openstack_dashboard-access.log
Dashboard 错误 /var/log/httpd/openstack_dashboard-error.log

```
### compute

```

Nova /var/log/nova/nova-computer.log
# libvirt 服务 连接底层虚拟化
ll /var/log/libvirt/

Neutron /var/log/cinder/volume.log
```

### cinder

``` 
Cinder /var/log/cinder/volume.log
```











