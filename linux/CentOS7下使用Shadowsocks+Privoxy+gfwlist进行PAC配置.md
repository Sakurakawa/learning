<!-- TOC -->

- [CentOS7 下使用 Shadowsocks+Privoxy+gfwlist 进行 PAC 配置](#centos7-下使用-shadowsocksprivoxygfwlist-进行-pac-配置)
    - [前言](#前言)
    - [Shadowsocks](#shadowsocks)
        - [1.安装](#1安装)
        - [2.配置](#2配置)
        - [访问测试](#访问测试)
        - [chrome 相关](#chrome-相关)
        - [配置启动服务](#配置启动服务)
    - [Privoxy](#privoxy)
        - [安装](#安装)
        - [配置](#配置)
        - [环境变量](#环境变量)
    - [PAC 配置](#pac-配置)
        - [GFWList2Privoxy](#gfwlist2privoxy)
        - [配置 privoxy](#配置-privoxy)
        - [GenPac](#genpac)

<!-- /TOC -->

## CentOS7 下使用 Shadowsocks+Privoxy+gfwlist 进行 PAC 配置

---

### 前言

> 最近学习 linux 操作系统，由于平时用 Windows 也在用 SS 翻墙，就打算把 linux 的也配置了。
> 使用的系统是 RedHat 系的 CentOS7

### Shadowsocks

shadowsocks 是一个翻墙用的代理服务器，支持 socks5 协议，原理是将你主机的 http/https 请求转发到本地代理（默认端口 1080），再由 ss 转发到远程代理服务器（需要配置 VPS），代理服务器将数据根据加密协议加密后转发到你实际要访问的 IP 地址。主机接受应答的过程则是一个逆过程。

#### 1.安装

安装需要 python 支持。  
注意这里有个坑，网上的安装

```
pip install shadowsocks  //2.8版本
```

安装的是 2.8 版本，这版是不支持 chacha20 加密方式的，即使你后面根据提示安装了 libsodium。  
这里我们需要安装 3.0 版本，顺便安装 libsodium

```
pip install -U https://github.com/shadowsocks/shadowsocks/archive/master.zip
yum install -y libsodium
```

然后就可以愉快的配置了。

#### 2.配置

方便起见使用 json 文件配置

```
$cd /etc
$mkdir shadowsocks
$cd shadowsocks
$vi config.json
```

在 config.json 中填写如下信息，注意#号后面的内容要删掉

```
{
  "server": your_server_address,　　　　　# Shadowsocks服务器地址（根据实际修改）
  "server_port": your_server_port,　　　　　# Shadowsocks服务器端口（根据实际修改）
  "local_address": "127.0.0.1",　# 本地IP
  "local_port": 1080,　　　　　# 本地端口（默认为1080，下面多处用到，建议不修改）
  "password": your_password,　　# Shadowsocks连接密码（根据实际修改）
  "timeout": 300,　　　　　　# 等待超时时间
  "method": your_method,　# 加密方式（根据实际修改）
  "workers": 1, 　　　　　　#工作线程数
  "fast_open": false　# true或false。不建议开启，开启可以降低延迟，但要求Linux内核在3.7+以上，若开启后访问网站出现502错误，请关闭。
}
```

然后使用命令启动

```
$sslocal -c /etc/shadowsocks/config.json
```

接下来主机的 1080 端口就打开了，它可以使用 socks5 协议进行网络代理。

#### 访问测试

配置成功后你的本地代理是开启了，但你发出去的请求并不会自动经过代理服务器的转发，我们通过修改环境变量来让流量全经过 1080 端口。输入命令

```
$export http_proxy="socks5://127.0.0.1:1080"
```

让终端的 http 代理通过 socks5 协议转发流量，我们可以通过支持多种协议的命令测试，比如 curl

```
[root@localhost sakugawa]# curl -I www.google.com
HTTP/1.1 200 OK
...
```

返回状态码 200，连接成功！  
但这存在局限性，常用的 wget 等命令，只支持 http 协议，并不能通过 http 代理去解析 socks5

```
[root@localhost sakugawa]# wget www.google.com
解析代理服务器 URL socks5://127.0.0.1:1080 时发生错误：不支持的协议类型 “socks5”。
```

这时就要用到 privoxy 了

#### chrome 相关

由上讨论，http/https 的流量并不会自动经过我们的代理，那浏览器肯定也不行了，以 chrome 为例，可以使用强大的代理插件 SwitchyOmega。  
CentOS7 下安装 google chrome 和 switchyomega 就不赘述了，代理配置的大致步骤如下

> 选项——新建情景模式——名称自己填，以 ss 为例——选择类型代理服务器——代理协议 SOCKS5，代理服务器 127.0.0.1，代理端口 1080 应用选项——switchyomega 选择 ss——愉快翻墙

#### 配置启动服务

新建启动脚本

```
$vi /etc/systemd/system/shadowsocks.service
```

输入内容

```
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks/config.json
[Install]
WantedBy=multi-user.target
```

通过下面命令使用

```
$systemctl enable shadowsocks.service //开机自启动
$systemctl start shadowsocks.service //启用服务
$systemctl stop shadowsocks.service //禁用服务
$systemctl restart shadowsocks.service //重启服务
$systemctl status shadowsocks.service //查看服务状态
```

---

### Privoxy

privoxy 也是一个代理服务器，它能做的事很多的，但我们只需要了解它的部分功能来帮助我们更好翻墙就完事了。  
privoxy 在我们的方案当中充当一个从请求发起到转发到 ss 本地代理（就是转发到本地 1080 端口这一步）之间的媒介，能让我们的 http、https、ftp 等协议流量统统支持 socks5 协议

#### 安装

```
yum install -y privoxy
```

#### 配置

privoxy 的配置文件位置为

```
/etc/privoxy/config
```

里面的内容我们只关注两处就行了。  
第一处

```
listen-address 127.0.0.1:8118
```

表示 privoxy 的监听端口，配置成功后流量会转发到这里。注意最前面是否有#号，有则删掉。  
第二处

```
forward-socks5t / 127.0.0.1:1080 .
```

这句话由四部分组成，由空格分隔。

1. forward-socks5t  
   表示要转发到的协议，privoxy 支持四种 socks 协议的转发
2. /<br>
   表示转发的对象为本机的所有请求
3. 127.0.0.1:1080  
   表示转发的目的端口
4. .<br>
   注意有个"."，根据 privoxy 的语法，"."表示请求经过了 ss 代理服务器后，不需要再转发到某个 http 代理，而是直接发送到目的 IP。同样注意删掉注释符号。

注意第二处的语句，所有请求都走代理，这就是全局代理。

#### 环境变量

上述配置只是打开了 8118 端口，我们要让所有流量都走这个端口，通过环境变量可以实现。这里我只设置了 http 和 https，还可以根据需求设置 ftp

```
export http_proxy="http://127.0.0.1:8118"
export https_proxy="http://127.0.0.1:8118"
```

注意这样写只能用于当前终端，把它写到/etc/profile 中才能保证每个终端都生效。

```
$echo PROXY=127.0.0.1:8118 >> /etc/profile
$echo export http_proxy=$PROXY >> /etc/profile
$echo export https_proxy=$PROXY >> /etc/profile
$source /etc/profile
```
现在来自终端的所有请求都能走代理了。最后别忘了启动服务哦。
```
$systemctl enable privoxy //开机自启动
$systemctl start privoxy //启用服务
$systemctl status privoxy //查看状态
```

### PAC 配置

上述配置成功后实现了全局代理，接下来要实现 PAC 代理，即根据用户自定义的规则决定哪些网站走代理，哪些不走。privoxy 支持以.action 文件实现 PAC 代理

#### GFWList2Privoxy

这个软件支持 gfwlist.txt 文件生成 privoxy 的.action 文件。  
先安装

```
$pip install --user gfwlist2privoxy
```

再从 github 上获取 gfwlist 文件

```
$wget https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt
```

生成.action 文件

```
~/.local/bin/gfwlist2privoxy -i gfwlist.txt -f gfwlist.action -p 127.0.0.1:1080 -t socks5
sudo cp gfwlist.action /etc/privoxy/
```

关于.action 文件，大致由一系列通配地址组成，把你想要走代理的地址写进去就行了，想让 google 走代理，就可以

```
echo .google.com >> gfwlist.action
```
#### 配置 privoxy
将.action 文件应用到 privoxy 中
```
echo actionsfile gfwlist.action >> /etc/privoxy/config
```
完成后重启一下 privoxy 服务即可。
#### GenPac
GenPac是基于gfwlist的多种代理软件配置文件生成工具，支持自定义规则，目前可生成的格式有pac, dnsmasq, wingy。具体可以查看github项目
> https://github.com/JinnLynn/genpac  

可以用它生成pac文件。先安装：
```
$ pip install -U genpac
```
在前面已经获取了gfwlist.txt文件了，可以本地生成：
```
$genpac --format=pac --pac-proxy="SOCKS5 127.0.0.1:1080" --gfwlist-local=~/gfwlist.txt --update-gfwlist-local
```
得到的pac文件可以用在chrome的switchyomega，也可以用在本地的系统代理：

> switchyomega选项——新建情景模式——选择PAC情景模式，填写名称——创建

然后把pac文件的内容粘贴到`PAC脚本`处就可以了