# pve-arm iso 安装 ＞ debian11 安装相关组件：
## 以下为pve-arm iso中安装的一些介绍和注意事项

## 1.修改网卡配置并重启网卡 
```
vim /etc/network/interfaces
systemctl restart networking
auto bond0
iface bond0 inet mamual
        bond-slaves enahisic2i2 enahisic2i3
        bond-miimon 100
        bond-mode 802.3ad
        bond-downdelay 200
        bond-updelay 200

auto vmbr0
iface vmbr0 inet static
        address 192.168.8.216/24
        gateway 192.168.8.1
        bridge-ports bond0
        bridge-stp off
        bridge-fd 0
```

## 2.hosts相关，安装界面要确定好相关hosts和主机名一般为arm-pve1.domain.com
pve服务需要host文件正确，否则会出现无法启动的问题。

## 3.如果需要更新到7.4.3 需要添加以下apt源、添加apt-key
```
echo "deb https://mirrors.apqa.cn/proxmox/ pvearm main">/etc/apt/sources.list.d/foxi.list
curl -L https://mirrors.apqa.cn/proxmox/gpg.key |apt-key add
```
wiki:
https://github.com/jiangcuo/Proxmox-Arm64/wiki/Install-Proxmox-VE-on-Debian-bullseye
## 4. 补丁包
### 4.1默认vnc是无法进行鼠标点击的，需要更新相关deb包
qemu-server_7.4-3_arm64.deb
### 4.2 不显示cpu型号的问题
```
apt install libipc-system-simple-perl
dpkg -i libpve-common-perl_7.3-4_all.deb
systemctl restart pvedaemon
```

## 5.arm-pve中添加vm需要注意：
### 5.1 BIOS必须为OVMF(UEFI)
### 5.2 需要添加EFI磁盘
### 5.3 需要添加串行端口
### 5.4 处理器为host或者默认
### 5.5 添加clou-init 磁盘 


## 6. arm-ubuntu  cloud-init模板机需要做的：
通常我们会使用以下命令来进行导入模板机和进行相关调整
```
qm importdisk 100  ubuntu-22.04-server-cloudimg-arm64.img   local-lvm   #把cloud-image 导入到序号为100的虚拟机
lvresize -l +1844MB   /dev/pve/vm-100-disk-0                            #默认导入的Cloud-image 磁盘为非整数大小，可通过这个命令扩容到整数大小
```
除此之外，我们想要使用clou-init 还需要在虚拟机内安装qemu-guest-agent
```
apt install qemu-guest-agent
```
### 6.1安装ccache
```
apt install ccache
```
### 6.2 安装tuned
```
apt install tuned
mkdir -p /etc/tuned/nebula
```
```
cat << EOF > /etc/tuned/nebula/tuned.conf
[main]
summary=Optimize for Nebula Graph DBMS
include=latency-performance

[vm]
transparent_hugepages=never

[sysctl]
kernel.core_pattern=core
kernel.core_uses_pid=1
kernel.numa_balancing=0

vm.swappiness=0
vm.oom_dump_tasks=1
# min_free_kbytes is suggested to set to approximately 2% of the total memory.
# 1GB at least and 5GB at most.
vm.min_free_kbytes=5242880
vm.max_map_count=131060
vm.dirty_background_ratio = 3
vm.dirty_ratio = 20
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100

net.core.busy_read=50
net.core.busy_poll=50
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=4096
net.core.netdev_max_backlog=10240
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_slow_start_after_idle=0
EOF
```
```
systemctl restart tuned
systemctl enable tuned
tuned-adm profile nebula
tuned-adm list
```
### 6.3 安装docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```
```
sudo cat << EOF >/home/vesoft/daemon.json
{
"experimental": true,
"registry-mirrors": ["http://hub-mirror.c.163.com,reg.vesoft-inc.com"]
}
EOF
sudo cp -af  /home/vesoft/daemon.json   /etc/docker/
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl restart docker
usermod -aG docker vesoft

```
### 6.4安装docker-compose
```
ARM64: curl -SL https://github.com/docker/compose/releases/download/v2.17.2/docker-compose-linux-aarch64 -o /usr/local/bin/docker-compose
X86: curl -SL https://github.com/docker/compose/releases/download/v2.17.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
### 6.5 python3-pip
```
apt install python3-pip
```
