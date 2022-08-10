# 基于NVMe的Ceph块存储可行性研究与测试

# 一. 测试说明

## 1. 测试架构

![NVME+RBD.jpg](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/NVMERBD.jpg)

## 2. 测试组件及机器角色

| 机器 | 角色 | 组件 | 内核版本 |
| --- | --- | --- | --- |
| 222.184.252.177 | Target | nvmetcli-0.6-1.el7.noarch | CentOS Linux (5.4.204-1.el7.elrepo.x86_64) 7 (Core) |
| 222.184.252.181 | Host | nvme-cli-1.8.1-3.el7.x86_64 | CentOS Linux (5.4.206-1.el7.elrepo.x86_64) 7 (Core) |

ceph版本：ceph version 14.2.22

| 主机名 | 操作系统 | 内核 | IP地址 | 服务器角色 |
| --- | --- | --- | --- | --- |
| server177 | CentOS 7.8.2003 | CentOS Linux (5.4.204-1.el7.elrepo.x86_64) 7 (Core) | 192.168.252.177 | rgw |
| server178 | CentOS 7.8.2003 | CentOS Linux (3.10.0-1127.el7.x86_64) 7 (Core) | 192.168.252.178 | mon + mgr + rgw |
| server179 | CentOS 7.8.2003 | CentOS Linux (3.10.0-1127.el7.x86_64) 7 (Core) | 192.168.252.179 | mon + mgr + rgw |
| server180 | CentOS 7.8.2003 | CentOS Linux (3.10.0-1127.el7.x86_64) 7 (Core) | 192.168.252.180 | mon + mgr + rgw |
| server181 | CentOS 7.8.2003 | CentOS Linux (5.4.206-1.el7.elrepo.x86_64) 7 (Core) | 192.168.252.181 | rgw |

# 二. 依赖安装

## 1. 内核版本升级

要求linux系统的内核版本为linux-4.1之后的版本，早期版本不支持NVMe over TCP

### 1.1 安装

```bash
wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/elrepo-release-7.0-6.el7.elrepo.noarch.rpm
rpm -ivh elrepo-release-7.0-6.el7.elrepo.noarch.rpm

yum --enablerepo=elrepo-kernel install kernel-lt -y
yum --enablerepo=elrepo-kernel install kernel-lt-devel -y
```

### 1.2 查看已安装内核版本

```bash
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg

可以看到安装的内核版本:
CentOS Linux (5.4.204-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-1127.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-d1b34d9bece549ccb6eb5b124f069035) 7 (Core)
```

### 1.3 设置从新内核启动

```bash
设置操作系统从新内核启动
grub2-set-default "CentOS Linux (5.4.204-1.el7.elrepo.x86_64) 7 (Core)"

查看内核启动项是否修改
grub2-editenv list

重启生效
reboot
```

# 三. Ceph RBD准备

### 1. 创建块设备image

```bash
[root@server177 ~]# rbd create --size 20480 rbd_nvme0 --object-size 4096
[root@server177 ~]# rbd info rbd_nvme0
rbd image 'rbd_nvme0':
        size 20 GiB in 5242880 objects
        order 12 (4 KiB objects)
        snapshot_count: 0
        id: 3a4a72f7babc5
        block_name_prefix: rbd_data.3a4a72f7babc5
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features: 
        flags: 
        create_timestamp: Tue Jul 19 10:04:21 2022
        access_timestamp: Tue Jul 19 10:04:21 2022
        modify_timestamp: Tue Jul 19 10:04:21 2022

```

### 2. 映射到系统内核并开机自启动

```bash
写入挂载信息
[root@server177 ~]# echo "rbd/rbd_nvme0           id=admin,keyring=/etc/ceph/ceph.client.admin.keyring" >> /etc/ceph/rbdmap
[root@server177 ~]# systemctl enable rbdmap.service

映射到系统内核
[root@server177 ~]# rbdmap map
[root@server177 ~]# rbd device ls
id pool namespace image     snap device    
0  rbd            rbd_nvme0 -    /dev/rbd0

```

# 四. Target 端配置

## 1. 安装

```bash
安装
yum install nvmetcli

加载内核模块
modprobe nvmet-tcp

开机启动
echo "modprobe nvme-tcp" >> /etc/sysconfig/modules/nvme.modules
chmod 755 /etc/sysconfig/modules/nvme.modules
source /etc/sysconfig/modules/nvme.modules
```

## 2. 系统运行检查

```bash
在target环境上，使用 lsmod | grep nvmet 命令，查看nvmet内核模块和nvmet_tcp内核模块确保都已经被正常加载

[root@server177 subsystems]# lsmod | grep nvmet
nvmet_tcp              24576  1 
nvmet                  86016  7 nvmet_tcp

在target环境上，使用ls /sys/kernel/config/ 命令确保此目录中已经有了nvmet目录；
```

## 3. 配置

### 3.1 启动 nvmetcli

```bash
# nvmetcli

/> ls
o- / ......................................................................................................................... [...]
  o- hosts ................................................................................................................... [...]
  o- ports ................................................................................................................... [...]
  o- subsystems .............................................................................................................. [...]
/>
```

### 3.2 创建NVMe子系统

```bash
方法一：
/ports> cd /subsystems 
/subsystems> create nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
/subsystems> cd nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632/
/subsystems/n...-c6c04f4e3632> ls
o- nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632 ........ [version=1.3, allow_any=0, serial=fbbc2e21524288d3]
  o- allowed_hosts ........................................................................................................... [...]
  o- namespaces .............................................................................................................. [...]
/subsystems/n...-c6c04f4e3632>

方法二：
mkdir /sys/kernel/config/nvmet/subsystems/nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
```

### 3.3 ****设置NVM subsystem允许访问的主机****

```bash
方法一：
/subsystems/n...-c6c04f4e3632> ls
o- nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632 ........ [version=1.3, allow_any=0, serial=fbbc2e21524288d3]
  o- allowed_hosts ........................................................................................................... [...]
  o- namespaces .............................................................................................................. [...]
/subsystems/n...-c6c04f4e3632> set attr allow_any_host=1
Parameter allow_any_host is now '1'.

方法二：
cd /sys/kernel/config/nvmet/subsystems/nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632/
echo 1 > attr_allow_any_host

如果需要指定允许特定主机， 则在allowed_hosts设置主机名，需要符合命名规则，这里使用hostnqn示意（如下）
/> cd hosts/
/hosts> create nqn
/hosts> cd /subsystems/nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632/allowed_hosts/
/subsystems/n...allowed_hosts> create hostnqn
```

### 3.4 创建namespace以及其关联设备

类似iSCSI Target给target添加LUN

```bash
方法一：
/subsystems/n...-c6c04f4e3632> cd namespaces 
/subsystems/n...32/namespaces> create nsid=1
/subsystems/n...32/namespaces> ls
o- namespaces ................................................................................................................ [...]
  o- 1 .......................................................... [path=(null), uuid=603dc232-ac53-44a3-b9de-eb8c20b44ba3, disabled] 
/subsystems/n...32/namespaces> cd 1 
/subsystems/n.../namespaces/1> set device path=/dev/rbd0
Parameter path is now '/dev/rbd0'.

enbale新创建的设备：
/subsystems/n.../namespaces/1> enable
The Namespace has been enabled.
/subsystems/n.../namespaces/1> ls
o- 1 .......................................................... [path=/dev/rbd0, uuid=603dc232-ac53-44a3-b9de-eb8c20b44ba3, enabled]

方法二：
cd /sys/kernel/config/nvmet/subsystems/nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632/namespace/
mkdir 1

cd 1
echo /dev/rbd0 > device_path
echo 1 > enable
```

### 3.5 ****创建NVMe over TCP的Transport层****

```bash
方法一：
/> cd ports 
/ports> create 1
/ports> ls
o- ports ..................................................................................................................... [...]
  o- 1 ................................................................................................ [trtype=, traddr=, trsvcid=]
    o- referrals ............................................................................................................. [...]
    o- subsystems ............................................................................................................ [...]
/ports> cd 1/
/ports/1> set addr adrfam=ipv4 trtype=tcp traddr=222.184.252.177 trsvcid=4420
Parameter trtype is now 'tcp'.
Parameter adrfam is now 'ipv4'.
Parameter trsvcid is now '4420'.
Parameter traddr is now '222.184.252.177'.
/ports/1> ls
o- 1 ............................................................................ [trtype=tcp, traddr=222.184.252.177, trsvcid=4420]
  o- referrals ............................................................................................................... [...]
  o- subsystems .............................................................................................................. [...]

方法二：
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1/

echo tcp > addr_trtype
echo ipv4 > addr_adrfam
echo 222.184.252.177 > addr_traddr
echo 4420 > addr_trsvcid
```

### 3.6 ****Transport与NVM subsystem建立关联****

```bash
方法一：
/ports/1> cd subsystems
/ports/1/subsystems> create nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
/ports/1/subsystems> ls
o- subsystems ................................................................................................................ [...]
  o- nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632 .................................................... [...]

方法二：
使用ln -s 把刚才创建的NVM subsystem在此目录中建立一个软连接

cd /sys/kernel/config/nvmet/ports/1/subsystems
ln -s ../../../../nvmet/subsystems/nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
```

### 3.7 查看配置并保存退出

```bash
/> ls
o- / ......................................................................................................................... [...]
  o- hosts ................................................................................................................... [...]
  o- ports ................................................................................................................... [...]
  | o- 1 ........................................................................ [trtype=tcp, traddr=222.184.252.177, trsvcid=4420]
  |   o- referrals ........................................................................................................... [...]
  |   o- subsystems .......................................................................................................... [...]
  |     o- nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632 .............................................. [...]
  o- subsystems .............................................................................................................. [...]
    o- nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632 .... [version=1.3, allow_any=1, serial=fbbc2e21524288d3]
      o- allowed_hosts ....................................................................................................... [...]
      o- namespaces .......................................................................................................... [...]
        o- 1 .................................................. [path=/dev/rbd0, uuid=603dc232-ac53-44a3-b9de-eb8c20b44ba3, enabled]
/>**saveconfig savefile=/etc/nvmet/nvmet.json**
/>exit
```

### 3.8 添加开机启动加载配置

```bash
vi /usr/lib/systemd/system/nvmet.service

[Unit]
Description=Restore NVMe kernel target configuration
Requires=sys-kernel-config.mount
After=sys-kernel-config.mount network.target local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/nvmetcli restore /etc/nvmet/nvmet.json
ExecStop=/usr/sbin/nvmetcli clear
SyslogIdentifier=nvmetcli

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable nvmet.service
```

# 五. Host 端配置

## 1. 安装

```bash
安装
yum install nvme-cli

加载内核模块
modprobe nvme-tcp

开机启动
echo "modprobe nvme-tcp" >> /etc/sysconfig/modules/nvme.modules
chmod 755 /etc/sysconfig/modules/nvme.modules
source /etc/sysconfig/modules/nvme.modules
```

## 2. ****系统运行检查****

```bash
在host环境上，使用 lsmod | grep nvme 命令，查看nvme内核模块、nvme_core内核模块和nvme-fabrics内核模块，确保都已经被正常加载；

[root@server181 ~]# lsmod | grep nvme
nvme_tcp               36864  0 
nvme_fabrics           24576  1 nvme_tcp
nvme_core              94208  2 nvme_tcp,nvme_fabrics

在host环境上，确认已经安装了可执行的nvme命令，可以使用nvme -h查看帮助
```

## 3. 配置

### 3.1 Host端发现 NVMe over Fabric 目标

```bash
命令格式：
nvme discover -t TRANSPORT -a DISCOVERY_CONTROLLER_ADDRESS -s SERVICE_ID

[root@server181 ~]# nvme discover -t tcp -a 222.184.252.177 -s 4420

Discovery Log Number of Records 1, Generation counter 2
=====Discovery Log Entry 0======
trtype:  tcp
adrfam:  ipv4
subtype: nvme subsystem
treq:    not specified, sq flow control disable supported
portid:  1
trsvcid: 4420
subnqn:  nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
traddr:  222.184.252.177
sectype: none

```

### 3.2 connect target

```bash
nvme connect -t tcp -a 222.184.252.177 -s 4420 -n nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632

若 target 未设置允许所有主机访问，则查看本机 hostnqn，将其添加至 target 端ACL
[root@server181 ~]# more /etc/nvme/hostnqn 
nqn.2014-08.org.nvmexpress:uuid:3208a275-a37f-4af8-8a97-382323ddec10
```

### 3.3 查看新添加到系统的nvme盘

连接成功后，执行nvme list就能看到NVMe over TCP相关的盘。

```bash
[root@server181 ~]# nvme list
Node             SN                   Model                                    Namespace Usage                      Format           FW Rev  
---------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1     d3884252212ebcfb     Linux                                    1          10.74  GB /  10.74  GB    512   B +  0 B   5.4.204-
```

可见，在host端添加了一个10G左右的nvme盘

### 3.4 断开连接

```bash
nvme disconnect -n nqn.2014-08.org.nvmexpress:uuid:4c4c4544-0039-3510-804c-c6c04f4e3632
```

### 3.5 磁盘格式化

```bash
mkfs.ext4 -b 4096 /dev/nvme0n1
```

# 六. nvme 与 iscsi 比较测试

两者块均来自于后端ceph存储

```bash
Target:
rbd create --size 20480 rbd_nvme --object_size 4096
rbd create --size 20480 rbd_iscsi --object_size 4096

Host:
mkfs.ext4 -b 4096 /dev/nvme0n1
mkfs.ext4 -b 4096 /dev/sdh
```

## 1. 顺序写，10线程，4k块文件

### 1.1 测试cmd

```bash
顺序写，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/nvme0n1 -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=write -group_reporting -name="nvme_4KB_write_test"

顺序写，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/sdh -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=write -group_reporting -name="iscsi_4KB_write_test"
```

### 1.2 测试结果

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled.png)

## 2. 随机写，10线程，4k块文件

### 2.1 测试cmd

```bash
随机写，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/nvme0n1 -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randwrite -group_reporting -name="nvme_4KB_randwrite_test"

随机写，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/sdh -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randwrite -group_reporting -name="iscsi_4KB_randwrite_test"
```

### 2.2 测试结果

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%201.png)

### 2.3 顺序写与随机写比较

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%202.png)

结论：

1. iscsi 顺序写性能 高于 随机写
2. nvme 顺序写性能 低于 随机写
3. nvme 写性能高于iscsi 写性能

## 3. 顺序读，10线程，4k块文件

### 3.1 测试cmd

```bash
顺序读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/nvme0n1 -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=read -group_reporting -name="nvme_4KB_read_test"

顺序读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/sdh -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=read -group_reporting -name="iscsi_4KB_read_test"
```

### 3.2 测试结果

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%203.png)

## 4. 随机读，10线程，4k块文件

### 4.1 测试cmd

```bash
随机读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/nvme0n1 -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randread -group_reporting -name="nvme_4KB_randread_test"

随机读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/sdh -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randread -group_reporting -name="iscsi_4KB_randread_test"
```

### 4.2 测试结果

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%204.png)

### 4.3 顺序读与随机读比较

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%205.png)

结论：

性能基本一致

## 5. 随机混合读写，10线程，4k块文件

### 5.1 测试cmd

```bash
随机混合读写，70%读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/nvme0n1 -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randrw -rwmixread=70 -group_reporting -name="nvme_4KB_randrw_test"

随机混合读写，70%读，每个线程写的数据量是1G，4k块文件，测试线程数10
fio -filename=/dev/sdh -direct=1 -ioengine=libaio -bs=4k -size=1G -numjobs=10 -iodepth=16 -thread -rw=randrw -rwmixread=70 -group_reporting -name="iscsi_4KB_randrw_test"
```

### 5.2 测试结果

![Untitled](%E5%9F%BA%E4%BA%8ENVMe%E7%9A%84Ceph%E5%9D%97%E5%AD%98%E5%82%A8%E5%8F%AF%E8%A1%8C%E6%80%A7%E7%A0%94%E7%A9%B6%E4%B8%8E%E6%B5%8B%E8%AF%95%20764e55ea8e7e420c84ff2fbcce6c3931/Untitled%206.png)

[nvme 与 iscsi 顺序写](https://www.notion.so/nvme-iscsi-5534d848a73944d99764f06720d5144d)

[调优过程记录](https://www.notion.so/dcdd422e876549f997658c481785df85)

[测试对比2](https://www.notion.so/2-346c77011c5d410393daeabedddc49bf)