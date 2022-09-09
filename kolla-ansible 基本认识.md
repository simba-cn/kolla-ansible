#1. 基本认识
##1.1. kolla-ansible

kolla-ansible是从kolla项目中分离出来的一个可交付的项目。kolla-ansible负责部署容器化的openstack各个服务和基础设施组件；

而kolla项目现在则单独负责镜像的构建，为kolla-ansible部署提供生产级别的openstack各服务镜像。

##1.2. ansible和docker

kolla-ansible利用ansible进行openstack服务的配置、编排openstack各个服务容器的部署。

利用容器的隔离性，达到openstack各服务容器的升级、回退，控制升级、回退的影响范围，降低openstack集群运维的复杂度。

ansible是一种基于python开发的自动化运维工具，它只需要在服务端安装ansible，无需在每个客户端安装客户端程序，通过ssh的方式来进行客户端服务器的管理，基于模块来实现批量数据配置、批量设备部署以及批量命令执行。

大致工作原理就是ansible程序调用读取/etc/ansible/ansible.cfg配置文件获取主机列表清单/etc/ansible/hosts文件，获取所要处理的主机列表，然后查看剧本任务，在根据剧本中一系列任务生成一个临时的脚本文件，然后将该脚本文件发送给所管理的主机，脚本文件在远程主机上执行完成后返回结果，然后删除本地临时文件

###ansible主要模块：

      Ansible：Ansible核心程序。

       HostInventory：记录由Ansible管理的主机信息，包括端口、密码、ip等。

       Playbooks：“剧本”YAML格式文件，多个任务定义在一个文件中，定义主机需要调用哪些模块来完成的功能。

       CoreModules：核心模块，主要操作是通过调用核心模块来完成管理任务。

       CustomModules：自定义模块，完成核心模块无法完成的功能，支持多种语言。

       ConnectionPlugins：连接插件，Ansible和Host通信使用

#2. kolla-ansible源码目录


	ansible- 包含 Ansible 剧本，用于在 Docker 容器中部署 OpenStack 服务和基础设施组件。
	contrib- 包含 Heat、Magnum 和 Tacker 的演示场景以及 Vagrant 的开发环境
	doc- 包含文档。
	etc- 包含一个参考 etc 目录结构，该目录结构需要配置少量配置变量以实现有效的一体化 (AIO) 部署。
	kolla_ansible- 包含密码生成脚本。
	releasenotes- 包含 Kolla Ansible 中添加的所有功能的发行说明。
	specs- 包含 Kolla Ansible 社区关于代码库中架构转变的关键论点。
	tests- 包含功能测试工具。
	tools- 包含与 Kolla Ansible 交互的工具。
	zuul.d- 包含项目关口作业定义。



#3. kolla-ansible
 kolla-ansible源代码位于/opt/oslostack/venv/bin/kolla-ansible

## 代码部分内容
kolla-ansible代码调用

执行kolla-anisble -i multinode deploy 时调用如下：

	(deploy)
	        ACTION="Deploying Playbooks"
	        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=deploy"
	        ;;

执行的命令如下：

	GLOBALS_DIR="${CONFIG_DIR}/globals.d"
	EXTRA_GLOBALS=$(find ${GLOBALS_DIR} -maxdepth 1 -type f -name '*.yml' -printf ' -e @%p' 2>/dev/null)
	PASSWORDS_FILE="${PASSWORDS_FILE:-${CONFIG_DIR}/passwords.yml}"
	CONFIG_OPTS="-e @${CONFIG_DIR}/globals.yml ${EXTRA_GLOBALS} -e @${PASSWORDS_FILE} -e CONFIG_DIR=${CONFIG_DIR}"
	CMD="ansible-playbook $CONFIG_OPTS $EXTRA_OPTS $PLAYBOOK $VERBOSITY"
	for INVENTORY in ${INVENTORIES[@]}; do
	    CMD="${CMD} --inventory $INVENTORY"
	done
	process_cmd

其中 INVENTORY参数表示multinode文件，主要是指定主机角色

CONFIG_OPTS指定globals.yml,passwords.yml，指定配置文件目录为/etc/kolla主要是指定一些配置相关。

EXTRA_OPTS主要是指定执行的动作

PLAYBOOK为roles的入口文件site.yml

这里我们解析下条命令：

	
执行

	kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy -vv

变成

	ansible-playbook $CONFIG_OPTS $EXTRA_OPTS $PLAYBOOK $VERBOSITY --inventory ./inventory 

最后执行的是

	ansible-playbook  -e @/opt/oslostack/oslostack.yml \
 	$(find /etc/kolla/globals.d -maxdepth 1 -type f -name '*.yml' -printf ' -e @%p' 2>/dev/null) \
	-e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla   -e kolla_action=deploy /opt/oslostack/kolla-ansible/ansible/site.yml \
	$VERBOSITY --inventory ./inventory 

所以调用过程大概描述为
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy ---->调用/opt/oslostack/kolla-ansible/ansible/site.yml---->根据site.yml文件的task调用执行各role


##3.1. kolla-ansible代码结构


	[root@oslostack01 kolla-ansible]# tree ansible/ -L 1

	ansible/

	├── action_plugins

	├── bifrost.yml

	├── certificates.yml

	├── destroy.yml

	├── filter_plugins

	├── gather-facts.yml

	├── group_vars

	├── inventory

	├── kolla-host.yml

	├── library

	├── mariadb_backup.yml

	├── mariadb_recovery.yml

	├── module_utils

	├── monasca_cleanup.yml

	├── nova-libvirt-cleanup.yml

	├── nova.yml

	├── octavia-certificates.yml

	├── post-deploy.yml

	├── prune-images.yml

	├── roles

	└── site.yml

##3.3. action_plugins目录
action_plugins目录下存放的是是kolla-ansible自定义的ansible插件

merge_configs.py，在playboy内通过使用merge_config来合并配置文件模板，生成openstack各服务的配置文件。

##3.4. ./group_vars/all.yml文件
all.yml文件作为ansible的变量文件，定义了各类配置信息。比如：配置文件路径、网卡、IP、端口号、各服务的开启等。（部分配置在globa.yml内也做了定义，global.yml具有更高优先级）
all.yml部分内容：

	###配置文件路径

	node_config_directory: "/etc/kolla/{{ project }}"

	 

	###################

	### Kolla options  定义vip

	###################

	kolla_internal_vip_address: "{{ kolla_internal_address }}"

	 

	####################

	### Networking options 网卡、端口等的配置

	####################

	network_interface: "eth0"

	api_interface: "{{ network_interface }}"

	aodh_api_port: "8042"

 
##3.5. inventory目录
inventory下存放的是主机清单

all-in-one用于单节点环境下，指定要部署的主机和该主机的角色
multinode用于多节点环境，指定要部署的主机和该主机的角色
主机清单也可作为定义变量的变量文件，我们所使用的主机清单是inventory文件。
### 部分内容
	指定节点到control组

	[control]

	node1

	#### 指定节点到network组

	[network]

	node1

##3.6. library目录
library目录下是kolla-ansible自定义的ansible模块

kolla_container_facts.py: 收集 Docker 容器事实的模块。 它用于检测容器是否在 Kolla 的主机上运行。

kolla_docker.py: 通过调用docker-py来驱动docker，进行启动容器、删除容器等操作

kolla_toolbox.py: 一个针对在 kolla_toolbox 中调用 ansible 模块的模块Kolla 项目使用的容器。

##3.7. roles目录
###3.7.1. ansible role简介

因为在实际中，会有很多不同的业务需要很多不同的playbook文件，很难进行维护。所以ansible采用role的方式对playbook进行目录结构规范。

在kolla-ansible内，对各个openstack服务进行部署，同样需要很多不同的playbook。

在roles目录内，有部署openstack各服务所需的各种playbook、定义的变量以及模板文件等。

与roles同级的还有site.yml文件，这是role引用的入口文件。


kolla-ansible代码roles目录分析

下面以roles目录下的neutron为例进行分析，其他服务的结构基本类似。

###3.7.2 neutron目录结构


	[root@oslostack01 roles]# tree neutron/ -L 1

	neutron/

	├── defaults

	├── handlers

	├── tasks

	├── templates

	└── vars

5 directories, 0 files

roles/neutron目录下有5个文件夹：

default： 定义了部署neutron各服务的各类参数

handlers：定义了启动neutron各服务容器的操作

vars：vars目录用于定义变量 #在kolla-ansible里面主要用于定义项目名

tasks：部署neutron的各playbook

templates：neutron各服务配置文件的模板


###3.7.3. defaults
defaults下的main.yml，作为当前role的变量文件，定义了关于neutron及neutron各服务的相关参数

####部分内容：

---



	neutron_services:

	  neutron-server:

	    container_name: "neutron_server"  #定义容器名字

	    image: "{{ neutron_server_image_full }}" #使用的镜像

	    enabled: true  #开启

	    group: "neutron-server"

	    host_in_groups: "{{ inventory_hostname in groups['neutron-server'] }}"

	    volumes: "{{ neutron_server_default_volumes + neutron_server_extra_volumes }}"  # 容器和宿主机映射的卷



####################
#### Database  定义neutron数据库地址等
####################
	neutron_database_name: "neutron"
	neutron_database_user: "{% if use_preconfigured_databases | bool and use_common_mariadb_user | bool %}{{ database_user }}{% else %}neutron{% endif %}"
	neutron_database_address: "{{ database_address | put_address_in_context('url') }}:{{ database_port }}"



###3.7.4. handlers
handlers下的main.yml文件，实际是创建、启动neutron各服务容器的playbook。但handlers只能在被触发的情况下才会去执行相关被触发的Task。

部分内容：

	# 启动neutron-server容器

	---

	- name: Restart neutron-server container

	  #### vars：task下定义的变量

	  vars:

	    service_name: "neutron-server"
	    service: "{{ neutron_services[service_name] }}"

	  #### 调用kolla-docker模块，启动容器
	  kolla_docker:
	    action: "recreate_or_restart_container"
	    common_options: "{{ docker_common_options }}" # 指定启动容器所需镜像
	    name: "{{ service.container_name }}" # 指定容器名
	    image: "{{ service.image }}"# 指定启动容器所需镜像
	    volumes: "{{ service.volumes|reject('equalto', '')|list }}" # 容器内配置文件等和宿主机的映射关系
	    dimensions: "{{ service.dimensions }}"
	    privileged: "{{ service.privileged | default(False) }}"# 指定是否开启特权
	    healthcheck: "{{ service.healthcheck | default(omit) }}"#状态检查
	  when:  # when：只有当列表下的所有条件满足时，才执行该task
	    - kolla_action != "config"

 

###3.7.5. vars

vars 只定义了一个项目名

	[root@oslostack01 vars]# ls
	main.yml

main.yml全部内容：

	---
	project_name: "neutron"

###3.7.6. tasks

在tasks目录下，有很多的yml文件，其中main.yml是入口执行文件。

	[root@oslostack01 roles]# ll neutron/tasks/
	total 96
	-rw-r--r--. 1 root root   660 Aug 16 13:09 bootstrap_service.yml
	-rw-r--r--. 1 root root  1101 Aug 16 13:09 bootstrap.yml
	-rw-r--r--. 1 root root   675 Aug 16 13:09 check-containers.yml
	-rw-r--r--. 1 root root     4 Aug 16 13:09 check.yml
	-rw-r--r--. 1 root root   275 Aug 16 13:09 clone.yml
	-rw-r--r--. 1 root root  1963 Aug 16 13:09 config-host.yml
	-rw-r--r--. 1 root root  3373 Aug 16 13:09 config-neutron-fake.yml
	-rw-r--r--. 1 root root 15779 Aug 16 13:09 config.yml
	-rw-r--r--. 1 root root   162 Aug 16 13:09 copy-certs.yml
	-rw-r--r--. 1 root root    41 Aug 16 13:09 deploy-containers.yml
	-rw-r--r--. 1 root root   396 Aug 16 13:09 deploy.yml
	-rw-r--r--. 1 root root   314 Aug 16 13:09 legacy_upgrade.yml
	-rw-r--r--. 1 root root   165 Aug 16 13:09 loadbalancer.yml
	-rw-r--r--. 1 root root    46 Aug 16 13:09 main.yml
	-rw-r--r--. 1 root root  1656 Aug 16 13:09 precheck.yml
	-rw-r--r--. 1 root root    49 Aug 16 13:09 pull.yml
	-rw-r--r--. 1 root root    31 Aug 16 13:09 reconfigure.yml
	-rw-r--r--. 1 root root   236 Aug 16 13:09 register.yml
	-rw-r--r--. 1 root root  2892 Aug 16 13:09 rolling_upgrade.yml
	-rw-r--r--. 1 root root   136 Aug 16 13:09 stop.yml
	-rw-r--r--. 1 root root   174 Aug 16 13:09 upgrade.yml

当我们执行kolla-ansible deploy时，main.yml将调用deploy.yml

main.yml全部内容：

	---
	- include_tasks: "{{ kolla_action }}.yml"


deploy-containers.yml

	[root@oslostack01 tasks]# cat deploy-containers.yml 
	---
	- import_tasks: check-containers.yml  #部署容器前先去调用check-containers.yml 

check-containers.yml 全部内容：

	---
	- name: Check neutron containers
	  become: true
	  kolla_docker:
	    action: "compare_container"
	    common_options: "{{ docker_common_options }}"
	    name: "{{ item.value.container_name }}"
	    image: "{{ item.value.image }}"
	    privileged: "{{ item.value.privileged | default(False) }}"
	    volumes: "{{ item.value.volumes }}"
	    dimensions: "{{ item.value.dimensions }}"
	    environment: "{{ item.value.environment | default(omit) }}"
	    healthcheck: "{{ item.value.healthcheck | default(omit) }}"
	  when:
	    - item.value.enabled | bool
	    - item.value.host_in_groups | bool
	  with_dict: "{{ neutron_services }}"
	  notify:
	    - "Restart {{ item.key }} container"


precheck.yml

部分内容：

检查是否开启Ironic否则报错

	- name: Checking whether Ironic enabled
	  fail:
	    msg: "Ironic must be enabled when using networking-baremetal/ironic-neutron-agent"
	  changed_when: false
	  run_once: True
	  when:
	    - enable_ironic_neutron_agent | bool
	    - not (enable_ironic | bool)


bootstrap_service.yml


全部内容：

	---
	# 启动bootstrap_neutron容器
	- name: Running Neutron bootstrap container
	  vars:
	    neutron_server: "{{ neutron_services['neutron-server'] }}"
	  become: true
	  kolla_docker:
	    action: "start_container"
	    common_options: "{{ docker_common_options }}"
	    detach: False
	    environment:
	      KOLLA_BOOTSTRAP:
	      KOLLA_CONFIG_STRATEGY: "{{ config_strategy }}"
	      NEUTRON_BOOTSTRAP_SERVICES: "{{ neutron_bootstrap_services | join(' ') }}"
	    image: "{{ neutron_server.image }}" 
	    labels:
	      BOOTSTRAP:
	    name: "bootstrap_neutron" 
	    restart_policy: no
	    volumes: "{{ neutron_server.volumes }}"
	  run_once: True
	  delegate_to: "{{ groups[neutron_server.group][0] }}"


##3.7.7. templates

templates目录下存放着很多j2格式的文件，他们都是neutron各服务的配置文件模板，这些模板将被config.yml根据需要生成为各服务的配置文件。
这里举neutron.conf.j2和neutron-server.json.j2为例进行分析

部分内容：

	[DEFAULT]
	debug = {{ neutron_logging_debug }}
	#生成日志目录
	log_dir = /var/log/kolla/neutron

	# 将会读取变量文件中api_interface_address和neutron_server_port的值，生成配置项

	bind_host = {{ api_interface_address }}

	bind_port = {{ neutron_server_port }}

	# 将根据if条件判断表达式，符合的表达式将生成对应的配置项

		{% if neutron_plugin_agent == 'vmware_nsxv' %}
		core_plugin = vmware_nsx.plugin.NsxVPlugin
		{% elif neutron_plugin_agent == 'vmware_nsxv3' %}
		core_plugin = vmware_nsx.plugin.NsxV3Plugin
		dhcp_agent_notification = False
		{% elif neutron_plugin_agent == 'vmware_nsxp' %}
		core_plugin = vmware_nsx.plugin.NsxPolicyPlugin
		dhcp_agent_notification = False
		{% elif neutron_plugin_agent == 'vmware_dvs' %}
		core_plugin = vmware_nsx.plugin.NsxDvsPlugin
		{% else %}
		core_plugin = ml2
		service_plugins = {{ neutron_service_plugins|map(attribute='name')|join(',') }}
		{% endif %}


neutron-server.json.j2

config.yml将把neutron-server.json.j2生成为config.json，存放在/etc/kolla/neutron-server/目录下，同时该目录下还有neutron.conf等其他neutron-server的配置文件。

在启动容器时/etc/kolla/neutron-server/会被映射到neutron_server容器的/var/lib/kolla/config_files/目录下。

此时生成的config.json的作用就是提供/var/lib/kolla/config_files/目录下neutron-server服务各配置文件与真正配置文件目录：/etc/neutron/ 的链接关系

全部内容：


	{
	    "command": "neutron-server --config-file /etc/neutron/neutron.conf {% if neutron_plugin_agent in ['openvswitch', 'linuxbridge', 'ovn'] %} --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/neutron_vpnaas.conf {% elif neutron_plugin_agent in ['vmware_nsxv', 'vmware_nsxv3', 'vmware_nsxp', 'vmware_dvs'] %} --config-file /etc/neutron/plugins/vmware/nsx.ini {% endif %}",
	    "config_files": [
	        {
	            "source": "{{ container_config_directory }}/neutron.conf", # 提供/var/lib/kolla/config_files/neutron.conf和/etc/neutron/neutron.conf的链接
	            "dest": "/etc/neutron/neutron.conf",
	            "owner": "neutron",
	            "perm": "0600"
	        },
	        {
	            "source": "{{ container_config_directory }}/neutron_vpnaas.conf",
	            "dest": "/etc/neutron/neutron_vpnaas.conf",
	            "owner": "neutron",
	            "perm": "0600"
	        },
	        {% if neutron_policy_file is defined %}{
	            "source": "{{ container_config_directory }}/{{ neutron_policy_file }}",
	            "dest": "/etc/neutron/{{ neutron_policy_file }}",
	            "owner": "neutron",
	            "perm": "0600"
	        },{% endif %}
	{% if neutron_plugin_agent in ['vmware_nsxv', 'vmware_nsxv3', 'vmware_nsxp', 'vmware_dvs'] -%}
	        {
	            "source": "{{ container_config_directory }}/nsx.ini",
	            "dest": "/etc/neutron/plugins/vmware/nsx.ini",
	            "owner": "neutron",
	            "perm": "0600"
	        },{% endif %}
	{% if check_extra_ml2_plugins is defined and check_extra_ml2_plugins.matched > 0 %}{% for plugin in check_extra_ml2_plugins.files %}
	        {
	            "source": "{{ container_config_directory }}/{{ plugin.path | basename }}",
	            "dest": "/etc/neutron/plugins/ml2/{{ plugin.path | basename }}",
	            "owner": "neutron",
	            "perm": "0600"
	        },{% endfor %}{% endif %}
	        {
	            "source": "{{ container_config_directory }}/ml2_conf.ini",
	            "dest": "/etc/neutron/plugins/ml2/ml2_conf.ini",
	            "owner": "neutron",
	            "perm": "0600"
	        },
	        {
	            "source": "{{ container_config_directory }}/id_rsa",
	            "dest": "/var/lib/neutron/.ssh/id_rsa",
	            "owner": "neutron",
	            "perm": "0600"
	        }
	    ],
	    "permissions": [
	        {
	            "path": "/var/log/kolla/neutron",
	            "owner": "neutron:neutron",
	            "recurse": true
	        }
	    ]
	}


