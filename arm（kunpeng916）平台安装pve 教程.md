# pve-arm iso 安装 ＞ debian11 安装相关组件：
## 以下为pve-arm iso中安装的一些介绍和注意事项

## 1.修改网卡配置并重启网卡  systemctl restart networking 
/etc/network/interfaces

## 2.hosts相关，安装界面要确定好相关hosts和主机名一般为arm-pve1.domain.com
pve服务需要host文件正确，否则会出现无法启动的问题。

## 3.如果需要更新到7.4.3 需要添加以下apt源、添加apt-key
echo "deb https://mirrors.apqa.cn/proxmox/ pvearm main">/etc/apt/sources.list.d/foxi.list
curl -L https://mirrors.apqa.cn/proxmox/gpg.key |apt-key add 

## 4.arm-pve中添加vm需要注意：
### 4.1 BIOS必须为OVMF(UEFI)
### 4.2 需要添加EFI磁盘
### 4.3 需要添加串行端口
### 4.4 处理器为host或者默认


------------------------------------------------------------------------------------------
arm-ubuntu  cloud-init模板机需要做的：
1.安装ccache
apt install ccache
2.安装tuned
apt install tuned
3.安装docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

4.安装docker-compose
curl -SL https://github.com/docker/compose/releases/download/v2.17.2/docker-compose-linux-aarch64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
5.python3-pip
