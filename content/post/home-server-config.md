---
title: "小主机配置笔记"
date: 2023-06-21T22:03:16+08:00
tags: ["x86", "eq12"]
categories: ["Linux"]
---

决定淘汰树梅派4B，上x86小主机了，整理下配置过程，免的下次重装系统再从头开始。暂时没有win的需求，所以只装了Ubuntu22.04，系统安装略。



## 挂载硬盘

暂时还是外置usb硬盘。


```shell
sudo mkdir /media/disk1
sudo chown x:x /media/disk1
```

查看UUID

```shell
$ sudo blkid 

/dev/sda1: UUID="31c2f09d-e3e6-4e46-bb08-0370165c4f96" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="LVM" PARTUUID="522e1944-2b81-4574-90f5-9da62a622ddd"
```

添加到/etc/fstab， 开机挂载

```
UUID=31c2f09d-e3e6-4e46-bb08-0370165c4f96 /media/disk ext4 defaults,auto,users,rw,nofail 0 0
```

重启。

## FTP Server

安装 vsftpd

```shell
sudo apt install vsftpd
```

修改ftp用户的home目录，也就是ftp登录的时候看到的目录。
```shell
sudo usermod -d /media/disk3/ ftp
```


限制ftp只能访问其home目录
编辑/etc/vsftpd.conf

```
chroot_local_user=YES
chroot_list_file=/etc/vsftpd.chroot_list
```

创建 vsftpd.chroot_list, 添加ftp用户


```shell
$ cat /etc/vsftpd.chroot_list
ftp

```

修改ftp密码
```shell
sudo passwd ftp
```

登了下报错500

```
500 OOPS: vsftpd: refusing to run with writable root inside chroot()
```
解决方法是修改 /etc/pam.d/vsftpd, 将

```
auth	required	pam_shells.so
```

修改为

```
auth	required	pam_nologin.so
```

最后重启ftp服务

```shell
sudo systemctl restart vsftpd.service
```

使用用户名：ftp和前面设置的密码登录。 


## Docker 服务

### 安装 Docker
官网步骤 https://docs.docker.com/engine/install/ubuntu/

```shell
 sudo apt-get update
 sudo apt-get install ca-certificates curl gnupg
```

添加 Docker  GPG key:

```shell
 sudo install -m 0755 -d /etc/apt/keyrings

 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

 sudo chmod a+r /etc/apt/keyrings/docker.gpg
```


```shell
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

安装
```shell
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


添加权限

```shell
sudo groupadd docker
sudo usermod -aG docker $USER

```

验证

```shell
docker run hello-world
```

  
使用docker-compose同时启动多个服务， 按照功能分成了几块，每种服务对应一个 docker-compose.yml

### 存储

```yml
version: '3.5'

services:
  db:
    image: mariadb:10.5
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - ./nextcloud/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=xxx
      - MYSQL_PASSWORD=xxx
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  nextcloud:
    image: nextcloud:latest
    #command: bash -c 'chown www-data:www-data /var/www/html/data'
    volumes:
      - ./nextcloud:/var/www/html:rw # moutn nextcloud files folder
      - /media/disk2/nextcloud:/var/www/html/data:rw # mount your personal data folder
      - /media/disk3:/media:rw

    links:
      - db
    environment:
      - MYSQL_PASSWORD=xxx
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    restart: always
    ports:
      - 8000:80

  transmission:
    image: ghcr.io/linuxserver/transmission
    container_name: transmission
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - TRANSMISSION_WEB_HOME=/ui/transmissionic #optional
      - USER=x #optional
      - PASS=xxx #optional
      #- WHITELIST=iplist #optional
      #- HOST_WHITELIST=dnsnane list #optional
    volumes:
      - ./transmission:/config
      - ./transmission/ui:/ui
      - /media/disk3:/downloads
      - /media/disk3/torrent:/watch
    ports:
      - 8002:9091
      - 51413:51413
      - 51413:51413/udp

  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    hostname: syncthing #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
    volumes:
      - ./syncthing/config:/config
      - /media/disk2/syncthing/data1:/data1

    ports:
      - 8004:8384
      - 22000:22000/tcp
      - 22000:22000/udp
      - 21027:21027/udp
    restart: unless-stopped

  
```

包括nextcloud、transmission、syncthing这几个服务。

transmission的 web ui需要单独下载 https://github.com/6c65726f79/Transmissionic/releases， 解压后放到./transmission/ui目录下。


启动各项服务：

```sh
docker compose up -d
```


修改nextcloud/config/config.php， 添加信任域名

```php
'trusted_domains' => 
  array (
    0 => '192.168.123.201:8000',
    1 => 'nc.xistor.top',
  ),


```

### 导航

Heimdall导航页

```yml
version: "3.5"
services:
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
    volumes:
      - ./heimdall/config:/config
    ports:
      - 8006:80
      - 8007:443
    restart: unless-stopped

```


### 媒体

jellyfin 媒体中心


```yml
version: '3.5'

services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - DOCKER_MODS=linuxserver/mods:jellyfin-opencl-intel
      # - JELLYFIN_PublishedServerUrl=192.168.0.5 #optional
    volumes:
      - ./jellyfin/library:/config
      - /media/disk3/tv:/data/tvshows
      - /media/disk3/movie:/data/movies
    ports:
      - 8096:8096
      - 8920:8920 #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    devices:
      - dev/dri:/dev/dri

    restart: unless-stopped

```

在TV上观看的话需要安装kodi以及jellyfin插件， 下载 [jellyfin repository installer](https://kodi.jellyfin.org/repository.jellyfin.kodi.zip)， 在kodi中安装此插件，这个插件只是仓库，并不是jellfin插件，然后再选择`从库中安装` -> `视频插件` 安装jellfin插件，添加服务器后就可以观看了。 具体的流程参考[这个](https://post.smzdm.com/p/a99vlpmp/)。


**硬件加速**

由于使用的Intel 的CPU 和核显， 前面的`DOCKER_MODS`使用了`jellyfin-opencl-intel`  

jellyfin也需要如下配置下： `控制台` -> `播放` -> `转码`  `硬件加速选择` `Intel QuickSync(QSV)`

{{< figure src="/img/home-server/jellyfin-hwacc.png"  class="center" title="jellyfin 硬件加速">}}

勾选所有格式，以及 `启用 VPP 色调映射` 和 `启用色调映射` 。  

这样在客户端播放不支持的视频格式时，jellyfin转码可以使用硬件加速，降低cpu使用率。

### 笔记

为知笔记

```yml
version: '3.5'

service:

  wiz:
    image: wiznote/wizserver
    container_name: wiz
    ports:
      - 8010:80
      - 9269:9269/udp
    volumes:
      - /media/disk2/wizdata:/wiz/storage
      - /etc/localtime:/etc/localtime
    restart: always
    stdin_open: true
    tty: true


```


### 智能家居

HomeAssissant

```yml
version: "3.5"

services:
  homeassistant:
    image: lscr.io/linuxserver/homeassistant:latest
    container_name: homeassistant
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
    volumes:
      - ./homeassistant/config:/config
    restart: unless-stopped

```

安装hacs

```shell
$ mkdir www
$ mkdir custom_components
$ mkdir custom_components/hacs
$ cd custom_components/hacs/
$ wget https://github.com/hacs/integration/releases/download/1.32.1/hacs.zip
$ unzip
$ 
```

重启homaassistant

进入集成搜索hacs, 这时候就能搜到了，按照提示继续完成安装。



### 电子书


```yml
version: "3.5"

services:
  calibre-web:
    image: lscr.io/linuxserver/calibre-web:latest
    container_name: calibre-web
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - DOCKER_MODS=linuxserver/mods:universal-calibre #optional
      - OAUTHLIB_RELAX_TOKEN_SCOPE=1 #optional
    volumes:
      - ./calibre-web/data:/config
      - ./calibre-web/library:/books
    ports:
      - 8020:8083
    restart: unless-stopped

```
默认用户名和密码 Username: admin Password: admin123。  
配置过程参考[这个](https://blog.mokeedev.com/2022/06/1113/)。


## 定时任务


定时更新证书， vps那边使用acme.sh申请的免费证书，会定时更新，这边定时同步。需要放到/root下所以使用sudo


```shell
$ sudo mkdir /root/certs
$ sudo crontab -e

35 8   22  *  * bash /home/x/bin/update_key.sh
```

update_key.sh的内容

```shell
#!/bin/bash

scp -i /home/x/.ssh/id_rsa root@123.123.123.123:/root/certs/xistor.top* /root/certs/

# 重启服务
service frpc restart 
cd /opt/docker-comp/stroge/ &&  docker-compose restart
...
```


## FRP

仅涉及客户端配置

下载frpc https://github.com/fatedier/frp/releases


开机启动frpc, 新建systemd 服务 /etc/systemd/system/frpc.service

```ini
[Unit]
Description=Frp Client Service
After=network.target

[Service]
TimeoutStartSec=30
WorkingDirectory=/home/x/bin/frp
ExecStart=/home/x/bin/frp/frpc -c /home/x/bin/frp/frpc.ini
ExecReload=/home/x/bin/frp/frpc reload -c /home/x/bin/frp/frpc.ini
Restart=on-failure
ReStartSec = 60

[Install]
WantedBy=multi-user.target
```
使用frpc 内网穿透，并将http转成https, 下面是frpc.ini 的配置。

```ini
[common]

#frps 服务器地址、端口
server_addr = xxx.xx
server_port = 7766
token = xxxxxx
login_fail_exit = false

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6666


#子域名配置

[next cloud https]
type = https
local_ip = 127.0.0.1
local_port = 80
subdomain = nc

plugin = https2http
# 转换成 http 后，发送到本机的80端口
plugin_local_addr = 127.0.0.1:80

# 指定代理方式为 frp
plugin_header_X-From-Where = frp
# 指定证书的路径
plugin_crt_path = /root/certs/xistor.top.cer
plugin_key_path = /root/certs/xistor.top.key



[wiz note https]
type = https
local_ip = 127.0.0.1
local_port = 8080
subdomain = wiz


plugin = https2http
plugin_local_addr = 127.0.0.1:8080
#plugin_host_header_rewrite = 127.0.0.1
plugin_header_X-From-Where = frp
plugin_crt_path = /root/certs/xistor.top.cer
plugin_key_path = /root/certs/xistor.top.key

```

启动服务

```shell
systemctl start  frpc.service
```

开机启动

```shell
systemctl enable  frpc.service
```

