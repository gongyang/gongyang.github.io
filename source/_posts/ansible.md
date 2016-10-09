---
title : 自动化运维工具ansible--入门使用
date: 2016-4-24
---
### 什么是ansible
Ansible is Simple IT Automation，是一个自动化运维的工具，基于Python开发，实现了批量系统配置、批量程序部署、批量运行命令等功能。

### 安装
Ansible默认通过 SSH 协议管理机器，安装Ansible之后,不需要启动或运行一个后台进程,或是添加一个数据库。只要在一台电脑(可以是一台笔记本)上安装好,就可以通过这台电脑管理一组远程的机器.在远程被管理的机器上,不需要安装运行任何软件,因此升级Ansible版本不会有太多问题。

目前,只要机器上安装了 Python 2.6 (windows系统不可以做控制主机),都可以运行Ansible。主机的系统可以是 Red Hat, Debian, CentOS, OS X, BSD的各种版本,等等

#### 源码安装
```
$ git clone git://github.com/ansible/ansible.git --recursive
$ cd ./ansible
$ source ./hacking/env-setup
```
如果没有安装pip, 请先安装对应于你的Python版本的pip:
```
$ sudo easy_install pip
```
以下的Python模块也需要安装:
```
sudo pip install paramiko PyYAML Jinja2 httplib2
```
注意,当更新ansible版本时,不只要更新git的源码树,也要更新git中指向Ansible自身模块的 “submodules” (不是同一种模块)
```
$ git pull --rebase
$ git submodule update --init --recursive
```

#### Yum安装
通过Yum安装RPM适用于 EPEL 6, 7
```
$ sudo yum install ansible
```
你也可以自己创建RPM软件包。在Ansible项目的checkout的根目录下,或是在一个tarball中,使用 make rpm 命令创建RPM软件包。然后可分发这个软件包或是使用它来安装Ansible。在创建之前,先确定你已安装了 rpm-build, make, and python2-devel 。
```
$ git clone git://github.com/ansible/ansible.git
$ cd ./ansible
$ make rpm
$ sudo rpm -Uvh ~/rpmbuild/ansible-*.noarch.rpm
```
其他系统的安装方式，请参考[官方文档](http://docs.ansible.com/ansible/intro_installation.html)。

### inventory file
在我们开始前要先理解Ansible是如何通过SSH与远程服务器连接是很重要的。
Ansible 1.3及之后的版本默认会在本地的 OpenSSH可用时会尝试用其进行远程通讯。这会启用ControlPersist(一个性能特性),Kerberos,和在~/.ssh/config中的配置选项如 Jump Host setup。然而,当你使用Linux企业版6作为主控机(红帽企业版及其衍生版如CentOS),其OpenSSH版本可能过于老旧无法支持ControlPersist。在这些操作系统中,Ansible将会退回并采用 paramiko (由Python实现的高质量OpenSSH库)。 如果你希望能够使用像是Kerberized SSH之类的特性,烦请考虑使用Fedora, OS X, 或 Ubuntu 作为你的主控机直到相关平台上有更新版本的OpenSSH可供使用,或者启用Ansible的“accelerated mode”。

现在你已经安装了Ansible,是时候从一些基本知识开始了. 编辑(或创建)/etc/ansible/hosts 并在其中加入一个或多个远程系统。你的public SSH key必须在这些系统的“authorized_keys”中
```
[ggyy@gy-vm02 ~]$ cat /etc/ansible/hosts
[vm03]
192.168.1.124:9999

[vm04]
192.168.1.140

[vm07]
192.168.1.108 ansible_ssh_user=root
```
这个文件叫做节点设置文件(inventory file)，默认的文件路径为 /etc/ansible/hosts，其中[]叫做组名，用于对系统进行分类,便于对不同系统进行个别的管理。一个系统可以属于不同的组,比如一台服务器可以同时属于 webserver组 和 dbserver组。
如果有主机的SSH端口不是标准的22端口,可在主机名之后加上端口号,用冒号分隔，例如上面的主机vm03。
如果有一组相似的 hostname , 可简写如下:
```
[webservers]
www[01:50].example.com
```
对于每一个 host,你还可以选择连接用户名,默认是管理主机的当前用户。
```
[vm07]
192.168.1.108 ansible_ssh_user=root
```
更多Inventory 参数的说明：
```
ansible_ssh_host
      将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.

ansible_ssh_port
      ssh端口号.如果不是默认的端口号,通过此变量设置.

ansible_ssh_user
      默认的 ssh 用户名

ansible_ssh_pass
      ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)

ansible_sudo_pass
      sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)

ansible_sudo_exe (new in version 1.8)
      sudo 命令路径(适用于1.8及以上版本)

ansible_connection
      与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.

ansible_ssh_private_key_file
      ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.

ansible_shell_type
      目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.

ansible_python_interpreter
      目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python
      不是 2.X 版本的 Python.我们不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 可执行程序名不可为 python以外的名字(实际有可能名为python26).

      与 ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....
```

### 运行第一个命令
命令格式：
```
ansible <pattern_goes_here> -m <module_name> -a <arguments>
```
例如：
```
[ggyy@gy-vm02 ~]$ ansible vm03 -m command -a "hostname"
192.168.1.124 | success | rc=0 >>
gy-vm03
```
我们还可以这样：
```
[ggyy@gy-vm02 ~]$ ansible vm03:vm04 -m command -a "hostname"
192.168.1.140 | success | rc=0 >>
gy-vm04

192.168.1.124 | success | rc=0 >>
gy-vm03
```
Ansible提供两种方式去完成任务,一是 ad-hoc 命令,一是写 Ansible playbook。前者可以解决一些简单的任务, 后者解决较复杂的任务。
如果我们敲入一些命令去比较快的完成一些事情,而不需要将这些执行的命令特别保存下来, 这样的命令就叫做 ad-hoc 命令。

#### 常用的一些模块
**setup**
用来查看远程主机的一些基本信息
```
[ggyy@gy-vm02 ~]$ ansible vm03 -m setup
192.168.1.124 | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "192.168.1.124", 
            "192.168.168.13", 
            "172.17.42.1"
        ], 
        "ansible_all_ipv6_addresses": [
            "fe80::20c:29ff:fe39:c20d", 
            "fe80::20c:29ff:fe39:c217", 
            "fe80::d0cd:7dff:fe2c:3fb"
        ], 
        "ansible_architecture": "x86_64", 
        "ansible_bios_date": "07/02/2015", 
        "ansible_bios_version": "6.00", 
        "ansible_cmdline": {
            "KEYBOARDTYPE": "pc", 
            "KEYTABLE": "us", 
            "LANG": "en_US.UTF-8", 
            "SYSFONT": "latarcyrheb-sun16", 
            "quiet": true, 
            "rd_NO_DM": true, 
            "rd_NO_LUKS": true, 
            "rd_NO_LVM": true, 
            "rd_NO_MD": true, 
            "rhgb": true, 
            "ro": true, 
            "root": "UUID=1bfa6437-81b1-4288-acb9-1c5ff1532d03"
        }, 
```
**ping**
用来测试远程主机的运行状态
```
[ggyy@gy-vm02 ~]$ ansible all -m ping
192.168.1.108 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.1.140 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.1.124 | success >> {
    "changed": false, 
    "ping": "pong"
}
```
**file**
设置文件的属性
force：需要在两种情况下强制创建软链接，一种是源文件不存在，但之后会建立的情况下；另一种是目标软链接已存在，需要先取消之前的软链，然后创建新的软链，有两个选项：yes|no
group：定义文件/目录的属组
mode：定义文件/目录的权限
owner：定义文件/目录的属主
path：必选项，定义文件/目录的路径
recurse：递归设置文件的属性，只对目录有效
src：被链接的源文件路径，只应用于state=link的情况
dest：被链接到的路径，只应用于state=link的情况
state：
       directory：如果目录不存在，就创建目录
       file：即使文件不存在，也不会被创建
       link：创建软链接
       hard：创建硬链接
       touch：如果文件不存在，则会创建一个新的文件，如果文件或目录已存在，则更新其最后修改时间
       absent：删除目录、文件或者取消链接文件

**copy**
复制文件到远程主机
相关选项如下：
backup：在覆盖之前，将源文件备份，备份文件包含时间信息。有两个选项：yes|no
content：用于替代“src”，可以直接设定指定文件的值
dest：必选项。要将源文件复制到的远程主机的绝对路径，如果源文件是一个目录，那么该路径也必须是个目录
directory_mode：递归设定目录的权限，默认为系统默认权限
force：如果目标主机包含该文件，但内容不同，如果设置为yes，则强制覆盖，如果为no，则只有当目标主机的目标位置不存在该文件时，才复制。默认为yes
others：所有的file模块里的选项都可以在这里使用
src：被复制到远程主机的本地文件，可以是绝对路径，也可以是相对路径。如果路径是一个目录，它将递归复制。在这种情况下，如果路径使用“/”来结尾，则只复制目录里的内容，如果没有使用“/”来结尾，则包含目录在内的整个内容全部复制，类似于rsync。
```
[ggyy@gy-vm02 ~]$ ansible vm03 -m copy -a "src=/home/ggyy/vm02.txt dest=/tmp/vm02.txt owner=ggyy group=ggyy mode=0644" 
192.168.1.124 | success >> {
    "changed": true, 
    "checksum": "da39a3ee5e6b4b0d3255bfef95601890afd80709", 
    "dest": "/tmp/vm02.txt", 
    "gid": 500, 
    "group": "ggyy", 
    "md5sum": "d41d8cd98f00b204e9800998ecf8427e", 
    "mode": "0644", 
    "owner": "ggyy", 
    "secontext": "unconfined_u:object_r:user_home_t:s0", 
    "size": 0, 
    "src": "/home/ggyy/.ansible/tmp/ansible-tmp-1461489632.3-235226806263683/source", 
    "state": "file", 
    "uid": 500
}
```
```
[ggyy@gy-vm02 ~]$ ansible vm03 -m command -a "ls /tmp/vm02.txt -l"
192.168.1.124 | success | rc=0 >>
-rw-r--r--. 1 ggyy ggyy 0 Apr 24 17:19 /tmp/vm02.txt
```

**command**
在远程主机上执行命令,默认模块
相关选项如下：
creates：一个文件名，当该文件存在，则该命令不执行
free_form：要执行的linux指令
chdir：在执行指令之前，先切换到该目录
removes：一个文件名，当该文件不存在，则该选项不执行
executable：切换shell来执行指令，该执行路径必须是一个绝对路径
```
[ggyy@gy-vm02 ~]$ ansible all -m command -a "date"
192.168.1.108 | success | rc=0 >>
Sun Apr 24 17:23:12 CST 2016

192.168.1.140 | success | rc=0 >>
Sun Apr 24 17:22:50 CST 2016

192.168.1.124 | success | rc=0 >>
Sun Apr 24 17:23:10 CST 2016
```
**shell**
切换到某个shell执行指定的指令，参数与command相同。
与command不同的是，此模块可以支持命令管道，同时还有另一个模块也具备此功能：raw
```
[ggyy@gy-vm02 ~]$ ansible all -m shell -a "ps aux | grep ntpd"
192.168.1.108 | success | rc=0 >>
ntp        786  0.0  0.2  29396  2028 ?        Ss   16:34   0:00 /usr/sbin/ntpd -u ntp:ntp -g
root      2816  0.0  0.1   9508  1144 pts/0    S+   17:25   0:00 /bin/sh -c ps aux | grep ntpd
root      2818  0.0  0.0   9040   672 pts/0    S+   17:25   0:00 grep ntpd

192.168.1.140 | success | rc=0 >>
ggyy      3358  0.0  0.1   9232  1032 pts/0    S+   17:24   0:00 /bin/sh -c ps aux | grep ntpd
ggyy      3360  0.0  0.0   6388   664 pts/0    S+   17:24   0:00 grep ntpd

192.168.1.124 | success | rc=0 >>
ggyy      4717  0.0  0.1   9232  1032 pts/1    S+   17:25   0:00 /bin/sh -c ps aux | grep ntpd
ggyy      4719  0.0  0.0   6388   660 pts/1    S+   17:25   0:00 grep ntpd
```
**更多模块**
更多模块请参考
```
[ggyy@gy-vm02 ~]$ ansible-doc -l
```
好了，关于ansible的入门基础就讲到这里，下一节，我们来深入了解Playbooks。
