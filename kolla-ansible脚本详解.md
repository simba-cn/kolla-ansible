kolla-ansible 脚本详解
====

预习
-----

	>&2 也就是把结果输出到和标准错误一样；之前如果有定义标准错误重定向到某file文件，那么标准输出也重定向到这个file文件。



openstack-yoga
---

源码下载地址

	git clone -b stable/yoga https://opendev.org/openstack/kolla-ansible.git

1、kolla-ansible命令入口文件
-----

	(venv) [root@oslostack01 ~]# which kolla-ansible
	/opt/oslostack/venv/bin/kolla-ansible

	#查看文件格式为Bourne-Again shell，即bash脚本
	(venv) [root@oslostack01 ~]# file /opt/oslostack/venv/bin/kolla-ansible
	/opt/oslostack/venv/bin/kolla-ansible: Bourne-Again shell script, ASCII text executable


2、kolla-ansible脚本结构
-----

脚本处理流程是，先定义几个函数，再用case … esac处理不同参数情况的处理。最后一行的process_cmd是脚本入口。

2.1、get_python_bin 函数
-----

主要用于获取python执行目录

	function get_python_bin {
    if [ -n "$_PYTHON_BIN" ]; then
      echo -n "$_PYTHON_BIN"
      return
    fi

    local ansible_path #定义局部变量ansible_path 
    ansible_path=$(which ansible) # 找到当前环境下ansible环境

    if [[ $? -ne 0 ]]; then   #检查上一个命令执行情况如果不等于0则报错！(显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误)
        echo "ERROR: Ansible is not installed in the current (virtual) environment." >&2
        exit 1
    fi

    local ansible_shebang_line #定义局部变量ansible_shebang_line
    ansible_shebang_line=$(head -n1 "$ansible_path") #找到当前环境下ansible—shebang所需的python3解释程序

    if ! echo "$ansible_shebang_line" | egrep "^#!" &>/dev/null; then #检测ansible—shebang所需的python3解释程序是否正确，如果不正确则报错
        echo "ERROR: Ansible script is malformed (missing shebang line)." >&2
        exit 1
    fi

    # NOTE(yoctozepto): may have multiple parts
    _PYTHON_BIN=${ansible_shebang_line#\#\!} #把python3解释程序位置赋值给_PYTHON_BIN ，“#\#\!”取消#！只保留目录
    echo -n "$_PYTHON_BIN" #不换行输出
	}

shebang
-----

Shebang，也念做 hashbang，Linux中国翻译组的GOLinux将其翻译为“释伴”，即“解释伴随行”的简称。Unix 术语中，井号通常称为 sharp，hash 或 mesh；而叹号则常常称为 bang。

Shebang通常出现在类Unix系统的脚本中第一行，作为前两个字符。在Shebang之后，可以有一个或数个空白字符，后接解释器的绝对路径，用于指明执行这个脚本文件的解释器。在直接调用脚本时，系统的程序载入器会分析 Shebang 后的内容，将这些内容作为解释器指令，并调用该指令，将载有 Shebang 的文件路径作为该解释器的参数，执行脚本，从而使得脚本文件的调用方式与普通的可执行文件类似。

例如，以指令#!/bin/sh开头的文件，在执行时会实际调用 /bin/sh 程序（通常是 Bourne shell 或兼容的 shell，例如 bash、dash 等）来执行。

如果#!之后的解释程序是一个可执行文件，那么执行这个脚本时，它就会把文件名及其参数一起作为参数传给那个解释程序去执行。


2.2 check_environment_coherence函数
------

主要用于检查python3环境，ansible版本是否正确

	function check_environment_coherence {
    local ansible_python_cmdline #定义局部变量ansible_python_cmdline
    ansible_python_cmdline=$(get_python_bin)# 将获取到python3环境赋值给ansible_python_cmdline
    ansible_python_version=$($ansible_python_cmdline -c 'import sys; print(str(sys.version_info[0])+"."+str(sys.version_info[1]))')  #获取pyhton版本

    if ! $ansible_python_cmdline --version &>/dev/null; then #如果获取不到python版本则报错
        echo "ERROR: Ansible Python is not functional." >&2
        echo "Tried '$ansible_python_cmdline'" >&2
        exit 1
    fi

    # Check for existence of kolla_ansible module using Ansible's Python.
    # 使用 Ansible 的 Python 检查是否存在 kolla_ansible 模块。
    if ! $ansible_python_cmdline -c 'import kolla_ansible' &>/dev/null; then
        echo "ERROR: kolla_ansible has to be available in the Ansible PYTHONPATH." >&2
        echo "Please install both in the same (virtual) environment." >&2
        exit 1
    fi

    local ansible_full_version #定义局部变量ansible_full_version
    ansible_full_version=$($ansible_python_cmdline -c 'import ansible; print(ansible.__version__)') #获取ansible版本

    if [[ $? -ne 0 ]]; then #如果上一条执行失败输出下面内容
        echo "ERROR: Failed to obtain Ansible version:" >&2
        echo "$ansible_full_version" >&2
        exit 1
    fi

    local ansible_version #定义局部变量ansible_version
    ansible_version=$(echo "$ansible_full_version" | egrep -o '^[0-9]+\.[0-9]+') #获取ansible版本的前两个大版本如2.11.12 输出2.11

    if [[ $? -ne 0 ]]; then #检查ansible版本是否获取成功
        echo "ERROR: Failed to parse Ansible version:" >&2
        echo "$ansible_full_version" >&2
        exit 1
    fi

    #定义ansible版本需要在2.11到2.12之间
    local ANSIBLE_VERSION_MIN=2.11 
    local ANSIBLE_VERSION_MAX=2.12

    #检查ansible版本是否在2.11到2.12之间，如果没有则报错
    if [[ $(printf "%s\n" "$ANSIBLE_VERSION_MIN" "$ANSIBLE_VERSION_MAX" "$ansible_version" | sort -V | head -n1) != "$ANSIBLE_VERSION_MIN" ]] ||
       [[ $(printf "%s\n" "$ANSIBLE_VERSION_MIN" "$ANSIBLE_VERSION_MAX" "$ansible_version" | sort -V | tail -n1) != "$ANSIBLE_VERSION_MAX" ]]; then
        echo "ERROR: Ansible version should be between $ANSIBLE_VERSION_MIN and $ANSIBLE_VERSION_MAX. Current version is $ansible_full_version which is not supported."
        exit 1
    fi
	}

2.3 find_base_dir函数
-----

只用来找到kolla-ansible目录

	function find_base_dir {
    local dir_name #定义局部变量dir_name
    local python_dir #定义局部变量python_dir
    # "$0"Shell本身的文件名，根据dirname "$0"，获取dir_name
    dir_name=$(dirname "$0") 
    dir_name=$(readlink -e "$dir_name")
    # 获取python版本
    python_dir="python${ansible_python_version}"
    #根据dir_name，找到当前环境下kolla-ansible脚本所在的目录。
    if [ -z "$SNAP" ]; then
        if [[ ${dir_name} == "/usr/bin" ]]; then
            if test -f /usr/lib/${python_dir}/*-packages/kolla-ansible.egg-link; then
                # Editable install.
                BASEDIR="$(head -n1 /usr/lib/${python_dir}/*-packages/kolla-ansible.egg-link)"
            else
                BASEDIR=/usr/share/kolla-ansible
            fi
        elif [[ ${dir_name} == "/usr/local/bin" ]]; then
            if test -f /usr/local/lib/${python_dir}/*-packages/kolla-ansible.egg-link; then
                # Editable install.
                BASEDIR="$(head -n1 /usr/local/lib/${python_dir}/*-packages/kolla-ansible.egg-link)"
            else
                BASEDIR=/usr/local/share/kolla-ansible
            fi
        elif [[ ${dir_name} == ~/.local/bin ]]; then
            if test -f ~/.local/lib/${python_dir}/*-packages/kolla-ansible.egg-link; then
                # Editable install.
                BASEDIR="$(head -n1 ~/.local/lib/${python_dir}/*-packages/kolla-ansible.egg-link)"
            else
                BASEDIR=~/.local/share/kolla-ansible
            fi
        elif [[ -n ${VIRTUAL_ENV} ]] && [[ ${dir_name} == "$(readlink -e "${VIRTUAL_ENV}/bin")" ]]; then
            if test -f ${VIRTUAL_ENV}/lib/${python_dir}/site-packages/kolla-ansible.egg-link; then
                # Editable install.
                BASEDIR="$(head -n1 ${VIRTUAL_ENV}/lib/${python_dir}/*-packages/kolla-ansible.egg-link)"
            else
                BASEDIR="${VIRTUAL_ENV}/share/kolla-ansible"
            fi
        else
            # Running from sources (repo).
            BASEDIR="$(dirname ${dir_name})"
        fi
    else
        BASEDIR="$SNAP/share/kolla-ansible"
    fi
	}

2.4、 install_deps函数
----------------------

主要用于安装和检查ansible-galaxy依赖项

	function install_deps {
	    echo "Installing Ansible Galaxy dependencies"
	    ansible-galaxy collection install -r ${BASEDIR}/requirements.yml --force
	    if [[ $? -ne 0 ]]; then
	        echo "ERROR: Failed to install Ansible Galaxy dependencies" >&2
	        exit 1
	    fi
	}


2.5、 process_cmd 函数 
-----

直接执行拼接的CMD命令（CMD命令即：ansible-playbook -i 加指定参数）。


	function process_cmd {
	    echo "$ACTION : $CMD"
	    $CMD
	    if [[ $? -ne 0 ]]; then
	        echo "Command failed $CMD"
	        exit 1
	    fi
	}

2.6、 usage函数
-----

使用手册打印值（即在xshell中直接执行kolla-ansible后返回值，用于命令行添加参数--help返回的帮助文档）

	function usage {
	    cat <<EOF
	Usage: $0 COMMAND [options]

	Options:
	    --inventory, -i <inventory_path>   Specify path to ansible inventory file. \
	Can be specified multiple times to pass multiple inventories.
	    --playbook, -p <playbook_path>     Specify path to ansible playbook file
	    --configdir <config_path>          Specify path to directory with globals.yml
	    --key -k <key_path>                Specify path to ansible vault keyfile
	    --help, -h                         Show this usage information
	    --tags, -t <tags>                  Only run plays and tasks tagged with these values
	    --skip-tags <tags>                 Only run plays and tasks whose tags do not match these values
	    --extra, -e <ansible variables>    Set additional variables as key=value or YAML/JSON passed to ansible-playbook
	    --passwords <passwords_path>       Specify path to the passwords file
	    --limit <host>                     Specify host to run plays
	    --forks <forks>                    Number of forks to run Ansible with
	    --vault-id <@prompt or path>       Specify @prompt or password file (Ansible >=  2.4)
	    --ask-vault-pass                   Ask for vault password
	    --vault-password-file <path>       Specify password file for vault decrypt
	    --check, -C                        Don't make any changes and try to predict some of the changes that may occur instead
	    --diff, -D                         Show differences in ansible-playbook changed tasks
	    --verbose, -v                      Increase verbosity of ansible-playbook
	    --version                          Show version

	Environment variables:
	    EXTRA_OPTS                         Additional arguments to pass to ansible-playbook

	Commands:
	    install-deps         Install Ansible Galaxy dependencies
	    prechecks            Do pre-deployment checks for hosts
	    check                Do post-deployment smoke tests
	    mariadb_recovery     Recover a completely stopped mariadb cluster
	    mariadb_backup       Take a backup of MariaDB databases
	                             --full (default)
	                             --incremental
	    monasca_cleanup      Remove unused containers for the Monasca service
	    bootstrap-servers    Bootstrap servers with kolla deploy dependencies
	    destroy              Destroy Kolla containers, volumes and host configuration
	                             --include-images to also destroy Kolla images


	.................
	}

2.7 bash_completion
-----

用于配合bash-completion工具，实现命令行自动补全增强功能

	function bash_completion {
	cat <<EOF
	--inventory -i
	--playbook -p
	--configdir
	--key -k
	--help -h
	--skip-tags
	--tags -t
	--extra -e
	--passwords
	--limit
	--forks
	--vault-id
	--ask-vault-pass
	--vault-password-file
	--check -C
	--diff -D
	--verbose -v
	--version
	install-deps
	prechecks
	check
	mariadb_recovery
	mariadb_backup
	monasca_cleanup
	bootstrap-servers
	destroy
	deploy
	deploy-bifrost
	deploy-containers
	deploy-servers
	gather-facts
	post-deploy
	pull
	reconfigure
	stop
	certificates
	octavia-certificates
	upgrade
	upgrade-bifrost
	genconfig
	prune-images
	nova-libvirt-cleanup
	EOF
	}



2.8、  version函数
-----

用于获取kolla_ansible版本

	function version {
	    local python_bin 
	    python_bin=$(get_python_bin)

	    $python_bin -c 'from kolla_ansible.version import version_info; print(version_info)'
	}

2.9
-----

	check_environment_coherence #调用函数

	#定义长短参数选项
	SHORT_OPTS="hi:p:t:k:e:CD:v"
	LONG_OPTS="help,version,inventory:,playbook:,skip-tags:,tags:,key:,extra:,check,diff,verbose,configdir:,passwords:,limit:,forks:,vault-id:,ask-vault-pass,vault-password-file:,yes-i-really-really-mean-it,include-images,include-dev:,full,incremental"

	RAW_ARGS="$*" #所有参数列表。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数，此选项参数可超过9个。
	# getopt是用来解析传入shell的命令行参数的 ，指定长选项和短选项，$@ 跟$*类似，但是可以当作数组用
	#n >& m	将输出文件 m 和 n 合并。
	ARGS=$(getopt -o "${SHORT_OPTS}" -l "${LONG_OPTS}" --name "$0" -- "$@") || { usage >&2; exit 2; }
	#set：可以模拟传入参数
	#eval可以把echo的字符串当做命令解析    
	eval set -- "$ARGS" #生成命令

	find_base_dir #找到kolla-ansible目录

	INVENTORY="${BASEDIR}/ansible/inventory/all-in-one"  # INVENTORY默认值
	PLAYBOOK="${BASEDIR}/ansible/site.yml" # 若不指定PLAYBOOK变量，则默认值/opt/oslostack/venv/share/kolla-ansible/ansible/site.yml 
	VERBOSITY=
	EXTRA_OPTS=${EXTRA_OPTS} #EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=deploy"
	CONFIG_DIR="/etc/kolla" #定义kolla默认配置文件目录
	DANGER_CONFIRM=
	INCLUDE_IMAGES=
	INCLUDE_DEV=
	BACKUP_TYPE="full" #备份类型
	# Serial is not recommended and disabled by default. Users can enable it by
	# configuring ANSIBLE_SERIAL variable.
	ANSIBLE_SERIAL=${ANSIBLE_SERIAL:-0}
	INVENTORIES=()


2.10 while循环
-----

While判断，当参数个数不为0，开始匹配参数，以”kolla-ansible -i ./multinode deploy”为例分析

	while [ "$#" -gt 0 ]; do
	    case "$1" in

	    (--inventory|-i)
	            INVENTORIES+=("$2") # 获取第二个参数，如上例，即把’/multinode’赋值给INVENTORY
	            shift 2   #shift命令用于对参数的移动(左移)，通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理（常见于Linux中各种程序的启动脚本）。
	            ，如上例，shift 2执行前，$1是’-i’，执行后$1是’deploy’，（多个条件处理时，用shift可适配不同个数的参数）
	            ;;   # case固定用法，匹配发现取值符合某一模式后，其间所有命令开始执行直至 ;;

	    (--playbook|-p)
	            PLAYBOOK="$2"#定义剧本
	            shift 2 #shift命令用于对参数的移动(左移)
	            ;;

	    (--skip-tags)
	            EXTRA_OPTS="$EXTRA_OPTS --skip-tags $2" #跳过第二个参数
	    ..........
	    
	    esac
		done
		case "$1" in
		#  EXTRA_OPTS的值根据shift之后的$1来确定，例如deploy对应值为"$EXTRA_OPTS -e kolla_action=deploy"（EXTRA_OPTS初始值为空）
		# BASEDIR /opt/oslostack/venv/share/kolla-ansible
		(prechecks)
		        ACTION="Pre-deployment checking"
		        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=precheck"
		;;
		(mariadb_recovery)
		        ACTION="Attempting to restart mariadb cluster"
		        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=deploy"
		        PLAYBOOK="${BASEDIR}/ansible/mariadb_recovery.yml"
		        ;;
		(bootstrap-servers)
		        ACTION="Bootstrapping servers"
		        PLAYBOOK="${BASEDIR}/ansible/kolla-host.yml"
		        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=bootstrap-servers"
		        ;;
		(deploy)
		        ACTION="Deploying Playbooks"
		        EXTRA_OPTS="$EXTRA_OPTS -e kolla_action=deploy"
		(......)
		esac   # esac是case的结束标记。case ... esac，与switch ... case 语句类似，是一种多分枝选择结构。case匹配一个值或一个模式成功，执行匹配的命令。
3 
-----

	GLOBALS_DIR="${CONFIG_DIR}/globals.d"  #  CONFIG_DIR变量默认值为："/etc/kolla"；
	EXTRA_GLOBALS=$(find ${GLOBALS_DIR} -maxdepth 1 -type f -name '*.yml' -printf ' -e @%p' 2>/dev/null)
	PASSWORDS_FILE="${PASSWORDS_FILE:-${CONFIG_DIR}/passwords.yml}"  #默认密码文件
	CONFIG_OPTS="-e @${CONFIG_DIR}/globals.yml ${EXTRA_GLOBALS} -e @${PASSWORDS_FILE} -e CONFIG_DIR=${CONFIG_DIR}" #执行选项
	CMD="ansible-playbook $CONFIG_OPTS $EXTRA_OPTS $PLAYBOOK $VERBOSITY" #ansible-playbook指定参数变量的调用，参数都是以上shell脚本内容处理之后的值
	for INVENTORY in ${INVENTORIES[@]}; do
	    CMD="${CMD} --inventory $INVENTORY"  #主要是指定主机角色
	done
	process_cmd #调用cmd命令




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
	-e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla   -e kolla_action=deploy /opt/oslostack/kolla-ansible/ansible/site.yml \
	$VERBOSITY --inventory ./inventory 

所以调用过程大概描述为

kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy ---->调用/opt/oslostack/kolla-ansible/ansible/site.yml---->根据site.yml文件的task调用执行各role





