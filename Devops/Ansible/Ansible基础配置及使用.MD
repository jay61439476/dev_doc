# Ansible基础配置及使用

## Conf file
```bash
# ansible执行的时候会按照以下顺序查找配置项
ANSIBLE_CONFIG #(环境变量)
ansible.cfg # (当前目录下)
.ansible.cfg #(用户家目录下)
/etc/ansible/ansible.cfg
```

### 配置文件解析
```ini
[defaults]
action_plugins=/usr/share/ansible_plugins/action_plugins  #用于加载一些外部插件

ansible_managed=Ansible managed: {file} modified on %Y-%m-%d %H:%M:%S by {uid} on {host} #可以在模板中插入{{ ansible_managed }}来表示执行

host_key_checking = False      #当主机不在known_hosts时是否提示
ask_pass=False                 #控制Ansible playbook执行时是否会自动弹出询问密码，使用SSH 密钥匙做身份认证时需要关闭
ask_sudo_pass=False            #控制Ansible playbook在执行sudo之前是否询问sudo密码
executable = /bin/bash
_forks=5
remote_port=22                 #设置默认远程SSH端口号，不指定为22
remote_user = root             #使用ansible playbook执行的默认用户名，不指定默认使用当前用户名称:
roles_path = /opt/mysite/roles #路径指的是’roles/’下的额外目录,用于playbook搜索Ansible roles，多个路径可以用冒号分隔。
```

##	基本使用
```bash
yum -y install ansible

ls /etc/ansible
ansible.cfg hosts roles
# ansible.cfg 是 Ansible 工具的配置文件；hosts 用来配置被管理的机器；roles 是一个目录，playbook 将使用它

# ssh
ssh-keygen -t rsa
ssh-copy-id root@agent_host_ip
# ssh的时候不会提示是否保存key
ssh-keyscan remote_servers >> ~/.ssh/known_hosts

# Add host
vim /etc/ansible/hosts

[Client]
angent_host_ip_1
angent_host_ip_2

[server]
host1 ansible_ssh_host=1.1.1.1 ansible_ssh_user=appadmin
host2 ansible_ssh_host=1.1.1.2 ansible_ssh_user=appadmin

# test
shell > ansible Client -m ping     # 操作 Client 组 ( all 为操作 hosts 文件中所有主机 )，-m 指定执行 ping 模块，下面是返回结果
192.168.12.129 | SUCCESS => {
"changed": false, 
"ping": "pong"
}

# -i          指定 hosts 文件位置
# -u username 指定 SSH 连接的用户名
# -k          指定远程用户密码
# -f          指定并发数
# -s          如需要 root 权限执行时使用 ( 连接用户不是 root 时 )
# -K          -s 时，-K 输入 root 密码

```

## Inventory
> Ansible只能管理指定的服务器，在inventory文件中进行配置对应的主机/分组的数据，其格式如下
```ini
# 组名（对系统进行分组）
[webservers]
# 主机名
foo.example.com
                             
# 指定系统的别名 + ssh的用户
jumper ansible_ssh_host=192.168.1.50 ansible_ssh_user=appadmin

#对于单个链接可以设置连接类型和用户名    
db1.jaydenz.org    ansible_connection=ssh    ansible_ssh_user=jaydenz    

# 01到50,一组相似的hostname
www[01:50].example.com

# 给host设定变量，后续playbook中可以使用
host1 http_port=80 maxRequestsPerChild=808

# 给group设定变量，应用于组内的所有host
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com    

# 组内组
[southeast:children]
atlanta
raleigh    

#其他参数说明
ansible_ssh_host              #将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.
ansible_ssh_port              #ssh端口号.如果不是默认的端口号,通过此变量设置.
ansible_ssh_user              #默认的 ssh 用户名
ansible_ssh_pass              #ssh密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)
ansible_sudo_pass             #sudo密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)
ansible_sudo_exe              #sudo命令路径(适用于1.8及以上版本)
ansible_connection            #与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.
ansible_ssh_private_key_file  #ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.
ansible_shell_type            #目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.
ansible_python_interpreter    #目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python；不是 2.X 版本的 Python.我们不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 可执行程序名不可为 python以外的名字(实际有可能名为python26)；与 ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....
```

## 远程命令模块（command / script / shell）
```bash
ansible CentOS  -a ' systemctl stop telegraf '

# command 作为 Ansible 的默认模块，可以运行远程权限范围所有的 shell 命令，不支持管道符。
shell > ansible Client -m command -a "free -m"               # 查看 Client 分组主机内存使用情况

# script 的功能是在远程主机执行主控端存储的 shell 脚本文件，相当于 scp + shell 组合。
shell > ansible Client -m script -a "/home/test.sh 12 34"    # 远程执行本地脚本

# shell模块基本和command相同，但是shell支持管道符
shell > ansible Client -m shell -a "/home/test.sh"           # 执行远程脚本

ansible all -a "date"
```

## 常用指令
```ini
[local]
127.0.0.1

[remote]
192.168.1.2

# Run against a local server
ansible -i ./hosts local --connection=local -b --become-user=root \
    -m apt -a 'name=nginx state=installed update_cache=true'

127.0.0.1 | success >> {
    "changed": false
}

# Run against a remote server
ansible -i ./hosts remote -b --become-user=root \
    -m apt -a 'name=nginx state=installed update_cache=true'

127.0.0.1 | success >> {
    "changed": false
}

# or：
ansible -i ./hosts remote -v -m apt -a 'name=nginx state=installed update_cache=true' -u test -s -K --private-key=~/.ssh/id_rsa
```

```bash
remote，local，all # 使用这个标签的下定义的服务器hosts清单文件。“all”是针对文件中定义的每个服务器运行的特殊关键字
-m ping # 使用“ping”模块，它只是运行ping命令并返回结果
-c local| --connection=local # 在本地服务器上运行命令，而不是SSH

-i PATH --inventory=PATH # 指定host文件的路径，默认是在/etc/ansible/hosts
--private-key=PRIVATE_KEY_FILE_PATH # 使用指定路径的秘钥建立认证连接
-m DIRECTORY --module-path=DIRECTORY # 指定module的目录来加载module，默认是/usr/share/ansible
-c CONNECTION --connection=CONNECTION # 指定建立连接的类型，一般有ssh ，local

-i ./hosts # 设置inventory文件，命名为 hosts
-b # “成”，告诉可以成为另一个用户来运行命令
--become-user=root # 以用户“root”运行以下命令（例如，使用“sudo”命令）
-m apt  # 使用apt模块
-a 'name=nginx state=installed update_cache=true' #提供apt模块的参数，包括软件包名称，所需的结束状态以及是否更新软件包存储库缓存

# 常用命令：
-u USERNAME --user=USERNAME # 指定移动端的执行用户
-U SUDO_USERNAME --sudo-user=USERNAME
-s --sudo -u # 指定用户的时候，使用sudo获得root权限
-k --ask-pass  # 提示输入ssh的密码，而不是使用基于ssh的密钥认证
-K --ask-sudo-pass # 提示输入sudo密码，与--sudo一起使用

```

## playbook
```yml
- hosts: all    #执行tasks的主机，all表示所有
  become: yes   #使用特定用户执行tasks，该参数也可以配置在相应task中。
  become_user: root
  remote_user: username #the user log into machine.
  
  tasks:
    # 每个task都相当于在执行对应模块的功能
    # 每个task感觉都是单次的连接，执行完之后断掉，之前的环境变量设置不会在后续的task中生效
    # 描述task
    - name: copy local file to remote machine
      # 执行对应模块功能
      copy: 
        src: ~/test
        dest: ~/test
        owner: root 
        mode: 0600
      # 命令执行的结果存到变量中，方便后续使用
      register: rsa
      # 设置环境变量
      environment:
        JAVA_HOME: /usr/java/jre1.8.0_51
      # task有失败之后，相同host后续的task不会执行，该参数可在失败后继续执行。
      ignore_errors: yes
      # 给这部分task打上tags，可指定只执行相应tags的task  (命令中添加：-t deploy)
      tags: deploy
      # (call the tasks defined in handlers if module does some changes to the remote host)
      notify:
        - do something
            
    # defines a list of tasks
    handlers:
      - name: do something
        service: test

    - name: task 2
      debug: var={{ host_vars }} # 使用对应host的host_vars变量
```

## Doc
+ [ansible自动化运维详细教程及playbook详解](https://juejin.im/post/5b3a074d51882548bc3aa35e)
+ [ansible基础教程](https://www.jianshu.com/p/e390a01669f8)
+ [官方文档-latest](https://docs.ansible.com/ansible/latest)
