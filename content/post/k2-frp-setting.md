---
title: "K2路由器Pandavan配置frp实现内网穿透"
date: 2018-08-23T21:42:00+08:00
tags: ["raspberrypi", "frp", "路由器"]
---

树梅派跑起来后，肯定是能够随时随地访问才够方便，但是家中的宽带没有固定ip,而且pi又躲在路由器后面，因此想要访问的话必须曲线一下，有好多方法可以实现，手中刚好有闲鱼淘的k2路由器，入手后已经刷好了Padavan,
刷完之后很强大，就用它来搞定了。Padavan固件版本为3.4.3.9-099_7-02-21，我们需要的内网穿透工具frp已经在Padavan中内置了，相比与花生壳，使用frp我们需要自己搭建服务器，所以需要一台有固定ip的vps。内置的frp版本太低，frpc和frps版本差太多是跑不起来的。所以后面我们需要把它替换掉，ok,开干！
### 1.配置服务器端
使用一键安装脚本安装，脚本默认安装最新版的frp：

```bash   
$wget --no-check-certificate https://raw.githubusercontent.com/clangcn/onekey-install-shell/master/frps/install-frps.sh -O ./install-frps.sh
$chmod 700 ./install-frps.sh
$./install-frps.sh install
```   
安装过程中会每步询问，一般一路敲确定就行，想要配置也可以注意下。
下面是我几个主要配置，具体的可以去frp的[官方文档](https://github.com/fatedier/frp/blob/master/README.md)查看
   
    bind_port = 7000 #绑定的端口，服务器和客户端通信用
    kcp_bind_port = 7000
    dashboard port = 6443   #控制台端口
    dashboard_user = admin 
    dashboard_pwd = ****    #自己设置
    token = *****           #通信密码，需要客户端和服务器端保持一致
    privilege_mode = true
    
然后就可以打开一下http://your_vps_ip:6443/ 看一下有没有dashboard,如果有就说明安装成功了。

### 2.配置客户端
Padavan中frpc的设置在扩展功能->花生壳内网版->frp内网穿透中，打开frp内网穿透和frpc客户端功能。自带的frpc版本比较低所以我在frp_script中将它替换成了0.20版本的（和我服务器上安装的版本一致），第一次我是手工替换的，但重启后就没了，因为下载的frpc保存在内存中，而且k2的flash太小又不能插u盘，只能每次开机去网上下载替换，所以我将0.20版本的frpc上传到了阿里云上，frp_script脚本会自动去下载替换。脚本内容如下：
**（直接复制可能格式错误，最好手敲）**
```bash   

#!/bin/sh
export PATH='/opt/usr/sbin:/opt/usr/bin:/opt/sbin:/opt/bin:/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin'
export LD_LIBRARY_PATH=/lib:/opt/lib
killall frpc frps
mkdir -p /tmp/frp
#删除低版本的frpc
rm /opt/bin/frpc
#下载替换
wget -P /opt/bin/ https://code.aliyun.com/xistor/frpc-for-download/raw/master/frpc && chmod 777 /opt/bin/frpc
chmod 777 /opt/bin/frpc

#启动frp功能后会运行以下脚本
#使用方法请查看论坛教程地址: http://www.right.com.cn/forum/thread-191839-1-1.html
#frp项目地址教程: https://github.com/fatedier/frp/blob/master/README_zh.md
#请自行修改 auth_token 用于对客户端连接进行身份验证
# IP查询： http://119.29.29.29/d?dn=github.com

#客户端配置：
cat > "/tmp/frp/myfrpc.ini" <<-\EOF
[common]
server_addr =  #你服务器的ip
server_port = 7000
token = ******  #和你服务器上配置的一致
#authentication_timeout=0
[ssh]
type = tcp
local_ip = 192.168.1.32 #内网中树梅派的地址
local_port = 22
remote_port = 6666      #映射到外网的端口

EOF
```

ok,然后我们就可以用ssh -oPort=6666 pi@x.x.x.x 在外网访问派了！

其实Padanvn有个脚本会去下载frpc，只要把那个链接改成我们的0.20版本的下载链接就可以了，但是我只在/tmp目录下找到了sh_frp脚本里面有下载frpc的相关操作，但是这个脚本每次开机会被覆盖掉，应该还有其他源头，所以只好写在frp_script中了。
