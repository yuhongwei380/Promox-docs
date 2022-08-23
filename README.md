# Promox-docs
# Promox教程

## 第一部分：PVE安装部署

​	本节主要涉及到PVE的安装教程。PVE作为开源的虚拟化集群管理系统，相比较vmware 自然是有一定的优势。本文主要介绍Promox VE的安装教程。 （Promox VE以下将简称为PVE或pve）

​	**需要主机开启BIOS开启虚拟化支持。**

​	PVE基于debian构建，其本质还是linux，所以在使用上和linux 并无本质区别。

​	首先我们进入promox 的官网。

[PVE官网]: https://www.proxmox.com/en/downloads
[PVE官方指导]: https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_copy_and_clone

目前我们能看到比较新的固件版本是7.1，题主公司的版本主要为6.2的老版本，本次演示就基于虚拟机嵌套来开展的。**如果你是直接安装PVE可以跳过 第一步**。

### 1. PVE虚拟嵌套

#### 	1.1 虚拟化嵌套是否支持的查询。

```
egrep --color 'vmx|svm' /proc/cpuinfo
```

#### 	1.2 开启pve主机的nested,关闭所有虚拟机

```
cat /sys/module/kvm_intel/parameters/nested
N
输出N，表示未开启，输出Y，表示已开启
```

#### 	1.3 关闭虚拟机，开启内核支持。

```
modprobe -r kvm_intel
modprobe kvm_intel nested=1
cat /sys/module/kvm_intel/parameters/nested
Y
再次检查nested,输出Y，即为开启成功。
```

#### 	1.4 设置系统启动后自动开启nested。

```
echo "options kvm_intel nested=1" >> /etc/modprobe.d/modprobe.conf
```

#### 	1.5 虚拟系统vm的cpu类型为host

图形界面设置：选择vm—硬件—处理器—类型—host

------

### 2. 制作PVE镜像

​	如果是第一次安装pve 推荐使用rufus 这个工具来制作启动镜像。

[Rufus工具]: https://rufus.ie/zh/

​	制作完镜像后，选择U盘启动即可。

------

### 3. 安装PVE

#### 	3.1 Install Promox VE

​		从U盘启动后，会进入安装pve 的安装界面，同时也会有debug 模式和rescue boot 以及内存测试模式。**选择Install Promox VE**

<img src="D:/pic/1.png" alt="1" style="zoom: 50%;" />

#### 3.2 License Agreement  【I agree】

#### ![2](D:/pic/2.png)

#### 3.3 Target harddisk 选择你的安装磁盘【NEXT】

​	选择option 可更改filesystem 和 磁盘大小，这边选择默认即可。

![3](D:/pic/3.png)

#### 3.4 Location and Time zone  地区和时间选择【NEXT】

![4](D:/pic/4.png)

#### 3.5 Administration password and Email address 管理密码和邮件

​	邮件地址必填。

![5](D:/pic/5.png)

#### 3.6 Management Network Configuration 管理网络配置

​	hostname：vm-pve

![6](D:/pic/6.png)

#### 3.7 Summary总结【Install】

​	确认你的配置是否正确。等待安装即可。

![7](D:/pic/7.png)

#### 3.8 登录

​	https://IP:8006/

​	浏览器输入以上网址即可进入管理web页面进行管理。

​	登录后先修改语言为简体中文。

![8](D:/pic/8.png)

------

### 4. 虚拟化集群（多台pve主机）

#### 4.1 创建集群

​	点击左侧列表至数据中心—集群；选择创建集群。

![9](D:/pic/9.png)

![10](D:/pic/10.png)

#### 4.2 加入集群

​	复制加入信息，输入对端root密码 加入集群。其他pve节点点击加入集群同理。

![11](D:/pic/11-16381716690272.png)

![13](D:/pic/13.png)![12](D:/pic/12.png)

#### 4.3 网络

网络设置根据自己的需要。可创建链路聚合等。

![13-1 network](D:/pic/13-1%20network.png)

------



## 第二部分：镜像与磁盘管理

### 1. 镜像管理

​	在数据中心视图下，找到存储—添加—NFS；在添加NFS（读取群晖的固件iso）要注意，ID 一般为ISO，服务器为群晖NAS的地址，192.168.8.X，Export 为 /volume1/test/template/iso/   

![14](D:/pic/14.png)

![15](D:/pic/15.png)

------

### 2. 磁盘管理

#### 	2.1 虚拟机新磁盘

​		PVE WEB界面中，选择节点—磁盘 中添加磁盘即可，一般选用LVM-Thin。

![16](D:/pic/16.png)

#### 	2.2 vm2无法创建同名LVM-Thin

​		数据中心—存储中双击编辑yhw-data 勾选所有。

![17](D:/pic/17.png)

vm2中：

```
fdisk -l
fdisk /dev/sdb
n
p
1
[enter]#跳过First sector 和Last sector
t
8e
w
```

回到vm2中的磁盘 LVM-Thin中，选择创建Thinpool；去掉添加存储，即可创建同名LVM-Thin。

![18](D:/pic/18.png)

------

#### 2.3 vm主机扩容（新加磁盘）

##### 2.3.1 LVM相关基础知识

在进行vm主机扩容前，我们需要先了解LVM相关的一些基础知识。以下我总结为一幅图。

<img src="D:/pic/19.png" alt="19" style="zoom: 67%;" />

可以看到，按照图中的解释，我们将步骤大致分为四个：**对实体磁盘的处理、PV实体卷组的创建、VG的创建或扩容、LV逻辑卷组的创建或扩容。**

##### 2.3.2 实体阶段

此处我们假设通过fdisk -l 查看到新加的磁盘为sdc。

```
fdisk /dev/sdc
n	#新增磁盘
p	#primary 磁盘分区
1	#number1
[enter]   #跳过First sector 和Last sector
t	#更改格式
L	#列出信息
8e	#选择为Linux LVM
w	#保存
```

##### 2.3.3 pv实体卷组阶段

新磁盘需要新建一个pv。

```
pvs	#查看pv信息、pvdisplay
pvcreate /dev/sdc
```

##### 2.3.4 vg卷组阶段

将pv加入vg。

```
vgs	#查看vg信息、可以使用 vgdisplay yhw-data 检查 VG是否扩容成功
vgextend yhw-data /dev/sdc
```

##### 2.3.5 lv逻辑卷组阶段

```
lvs #查看lv信息、lvdisplay
lvscan	#记录yhw-data 的挂载目录
lvresize --extents +100%FREE  /dev/vm-data/vm-data
创建新LV时，需要以下步骤：
vgdisplay --> 找到PE total  例如625736
lvcreate -L 625736 -n vm-data  vm-data
```

FAQ：遇到Device /dev/sdc excluded by a filter  创建PV失败

A：该磁盘已经有了分区表，虚拟机无法识别磁盘的分区表，需要删除分区并重建PV。



------

### 第三部分：新建虚拟机

![20](D:/pic/20.png)

![21](D:/pic/21.png)

![22](D:/pic/22.png)

![23](D:/pic/23-16381743415013.png)

![24](D:/pic/24.png)

![25](D:/pic/25.png)

![26](D:/pic/26.png)

![27](D:/pic/27.png)

![28](D:/pic/28.png)



------

### 第四部分：Cloud-init模板机

#### 1.使用Cloudinit镜像制作模板机

##### 1.1 获取Cloud-init系统模板镜像

```
Centos7:http://cloud.centos.org/centos/7/images/
Debian:http://cdimage.debian.org/cdimage/cloud/OpenStack/current-10/
Ubuntu18.04 LTS:https://cloud-images.ubuntu.com/bionic/current/
https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cloud-images/bionic/current/
```

##### 1.2 利用ssh 等工具上传.qcow2等系统镜像

使用xftp或者ssh 工具上传centos7.qcow2 系统镜像到PVE服务器、本节试用的为winscp导入到PVE中的/tmp目录。

##### 1.3 创建虚拟系统&转换成模板

方法一：

```
qm create 001 --name centos7 --memory 2048 --net0 virtio,bridge=vmbr0
#新建虚拟机名称为：centos7，vmid为：001 （id可自定义,不存在即可）
qm importdisk 001 CentOS-7-x86_64-GenericCloud-2009.qcow2  yhw-data
#导入磁盘文件
qm set 001 --virtio0 local-lvm[lvm-data]:vm-001-disk-0
#定义磁盘总线类型为virtio
qm set 001 --boot c --bootdisk virtio0
#设置virtio0磁盘为第一引导设备
qm set 001 --serial0 socket --vga serial0
#添加并设置显卡设备为serial0
qm set 001 --ide2 local-lvm:cloudinit
#添加Cloudinit Drive设备
# qm set 001 --sshkey ~/.ssh/id_rsa.pub
#导入ssh公钥到虚拟系统
qm template 001
#将虚拟机转换成系统模板，可用之快速生成克隆的系统，不转换成模板也可直接使用。
```

方法二：

先创建一个虚拟机，分离和删除原磁盘。

![29](D:/pic/29.png)

pve宿主机的shell中输入以下命令，240为你创建的虚拟机的vm-ID，yhw-data 是所选的磁盘。

```
qm importdisk 240 ubuntu-20.04-server-cloudimg-amd64.img  yhw-data
```

![30](D:/pic/30.png)

等待进度完成后，可到虚拟机的磁盘页面中双击添加。**记得勾选SSD仿真和丢弃。**

![31](D:/pic/31.png)

某些镜像文件importdisk后，磁盘大小非整，例如2252M。先分离磁盘，再用下面的命令进行扩容到整数。

```
lvresive -L +1844M /dev/vm-data/vm-240-disk-0
```

双击添加磁盘后，即可发现磁盘容量已经整数。

FAQ：如果你的模板机不小心转换成模板，但你需要进一步修改。

A：可进入/etc/pve，找到对应的ID*.conf，修改并删除"`template: 1`" ，刷新即可。

方法三：

直接安装系统并安装Cloudinit内部依赖包：

```
yum -y install  Qemu-guest-agent
yum -y install cloud-init
```

注意：

虚拟机创建好后添加并配置Cloud-init

特别注意：硬件中的：cpu-host-numa

选项中的QEMU Guest Agent 是否启用、fstrim_cloned_disks

![32](D:/pic/32.png)

![33](D:/pic/33-16381759892724.png)

------

### 第五部分：Cloudinit 自定义设置

#### 1 修改配置文件

修改/etc/cloud/cloud.cfg文件，可实现一些自定义功能，比如开机运行脚本等。

以安装docker 和加入docker组为例子

```
vim /etc/cloud/cloud.cfg
runcmd:
  - curl -fsSL https://get.docker.com -o get-docker.sh; sh get-docker.sh
  - systemctl enable docker
  - systemctl restart docker
  
#找到system_info:
name: vesoft
lock_passwd: false
gecos: Cloud User
groups: [wheel, adm, systemd-journal, docker] #统一添加docker用户组
```





其他：

```
fstrim -a #回收未用空间，在lvm-thin 的情况下可用
```

