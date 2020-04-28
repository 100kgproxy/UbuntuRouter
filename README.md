# UbuntuRouter

## 简述
这是一个将Ubuntu当作软路由，以及添加透明代理的简单教程。配辅助工具。仅适配Server，18.04及以上。其他Linux系统可参考。

## TODO
辅助工具编写。应该支持命令行和web。仅仅只是将下文手动部分自动化。

## 絮叨
本文其实很尬，大佬基本不需要，菜鸟看不懂，所以给日常使用linux但是还不是大佬的人（码农）做个借鉴好了。

如果你想在路由上做透明代理，家用或公司用，那么可选项有3：
1. 支持刷机的硬路由，500RMB以上，刷入Openwrt或类似的系统。
2. 淘宝买现成的工控软路由，500-1500。商家已刷机，你也可以自己再刷其他系统。
3. 买itx主机（散件拼凑即可）或利用其他闲置PC，刷软路由。（需要多网口网卡或多网卡）

1和2选项的跳过不表，针对选项3，对一个特别特别爱折腾的人来说，你可能仍然觉得它不够自由，功能不够丰富。很可能到手还要额外学习如何配置。
大部分软路由系统仍然是以嵌入式思路设计的，你都有一个PC了，还画地为牢就没啥必要了。直接将常用的操作系统改造成路由即可。本文因个人偏好和习惯，仅介绍Ubuntu。啰嗦一句，其实不仅是linux，bsd可以，windows也可以添加路由功能，甚至更简单。
本文systemctl和service混用，勿怪。

## 缺点
- 软路由系统已搭配好常用路由设置的图形界面，Ubuntu的没有，你也可以找找看
- 软路由一般自带流量监控，Qos等功能。通用系统的话以上功能都得自己找对应软件或手动改配置了。

## 优点
- 稳定。虽然软路由系统也是个linux（或bsd），但是配套软件改动的设置过多，也没有专门的人负责测试，所以总是存在适配和稳定性问题。
自己撸的对系统改动很少且完全已知，稳定性有保障。
- 完整的操作系统，日常ssh体验良好，敲个命令发现不存在或者少参数很难受
- 功能应有尽有，下载啊，同步啊，穿透啊，这样的功能就是个正常软件，不需要看软路由固件/软件中心/包管理的脸色。
- 不需要不停的更新固件

## 辅助工具说明
[辅助工具说明](tools.md)

<br>

---

仅使用工具的无需阅读以下内容

---

## 全手动修改（及辅助工具原理）
本文命令在root下操作，用自他用户的可以切root或者自行sudo。
本文实际使用的ubuntu版本为20.04
### 1.安装必要软件
```
apt update
apt install zip unzip net-tools ipset iptables dnsmasq redsocks netfilter-persistent ipset-persistent iptables-persistent
```
dnsmasq用于dns和dhcp
redsocks用于将普通tcp，udp流量转发到socks5代理。netfilter-persistent用于ipset iptables的持久化（否则这两个的配置重启会失效），会自动调用ipset-persistent和iptables-persistent，后两个是透明的只需安装。

### 2.实现基本路由功能
#### 开启内核的forward功能
```
vi /etc/sysctl.conf
```
添加内容或取消注释
```
net.ipv4.ip_forward = 1
```
使之生效
```
sysctl -p
```

#### 配置网卡和网桥
先看看自己的网口列表，以下命令都行
```
ifconfig -a
ip link
ls /proc/sys/net/ipv4/conf
```
ubuntu 16.04+已经不是eth0这样的命名方式了。我们假设获取到4个网口，分别为ens16 ens17 ens18 ens27。<br>
ubuntu 18.04+已经不再直接修改/etc/network/interfaces文件了，而使用netplan管理网络。<br>
修改netplan的配置文件，yaml格式，文件名可能是其他的，自行ls。
```
netplan generate
ls /etc/netplan/
vi /etc/netplan/00-installer-config.yam
```
假定使用ens27作为wan，使用DHCP从上级路由/交换机获取IP。剩下三口桥接作为lan。
文件内容:
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens27:
      dhcp4: true
      dhcp6: false
      optional: true
    ens16:
      dhcp4: false
      dhcp6: false
      optional: true
    ens17:
      dhcp4: false
      dhcp6: false
      optional: true
    ens18:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    br0:
      interfaces: [ens16,ens17,ens18]
      dhcp4: false
      dhcp6: false
      addresses: [192.168.2.1/24]
  version: 2
```
应用配置
```
netplan try
netplan apply
```

optional表示启动系统时不用等此网口初始化完毕。
br0的addresses指定自己作为网关的ip，假设上层路由不是192.168.2.0网段。

#### 添加nat
```
iptables -t nat -A POSTROUTING -o ens27 -j MASQUERADE
iptables -A FORWARD -i ens27 -o br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i br0 -o ens27 -j ACCEPT
```

*啰嗦补充，iptables清空用“iptables -F”。删除是“iptables -D”，具体--help和google。*

持久化iptables
```
netfilter-persistent save
systemctl enable netfilter-persistent
systemctl start netfilter-persistent
```
enable设置为开机启动，start启动服务。

#### 开启DHCP和DNS
禁用自带的dns服务
```
systemctl disable systemd-resolved
systemctl stop systemd-resolved
```
配置dnsmasq
```
vi /etc/dnsmasq.conf
```
文件内容：
```
no-resolv
no-poll
server=127.0.0.1#5353
#server=223.5.5.5
cache-size=5000

dhcp-range=192.168.2.50,192.168.2.200,48h
dhcp-option=6,192.168.2.1

conf-dir=/routerfiles/foriptables/dnsmasq.d/,*.conf
```
no-resolv和no-poll是让dnsmasq不读取/etc/resolv.conf。
dhcp-range起始和结束ip，过期时间。
dhcp-option设置请google，6是指dns，设置为本路由192.168.2.1。<br>
127.0.0.1#5353是将上级服务器指向本地提供在5353端口的dns服务器。参考后文透明代理部分。在做透明代理之前可先用任意dns服务器ip。<br>
conf-dir用于加载其他配置，此文中用户处理动态的ipset

启动服务
```
systemctl restart dnsmasq
```

#### 启动时等待网络初始化需要两分钟的处理
本以为netplan 设置optional后可以不做下面这个，实测是需要的。禁用此服务也行哈。
```
vi /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service
```
修改ExecStart,任意网口准备好就算完成，并强制20秒超时
```
ExecStart=/lib/systemd/systemd-networkd-wait-online -q --timeout 20 --any
```

重启看看，应该各网口都可以上网了(请注意dns配置)。<br>
如果需要拨号，google下pppoeconf，本文不叙。

### 3.实现透明代理
#### 配置redsocks
```
vi /etc/redsocks.conf
```
修改或添加内容
```
base {
...
log_info = off;
rlimit_nofile = 65535;
...
}
redsocks {
...
local_ip = 0.0.0.0;
port = 你的代理服务的本地端口;
...
}
...
```
udp部分请自行更改。<br>
连接数受rlimit_nofile限制。redsocks使用redsocks用户启动，未配置rlimit_nofile时使用系统配置，你可以尝试修改ulimit,“vi /etc/security/limits.conf”，但是建议直接设置rlimit_nofile。这个是文件句柄数，实际连接数还要再除，关心的可以看redsocks源码。

启动服务
```
service redsocks restart
```

#### 配置ipset
```
mkdir /routerfiles
cd /routerfiles
mkdir foriptables
mkdir foriptables/dnsmasq.d
cd foriptables
# 获取大陆ip地址段
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > chnroutes.txt
#创建ipset
ipset -N chnroutes hash:net
ipset -N gfwlist hash:net
#for i in `cat chnroutes.txt`; do echo ipset -A chnroutes $i >> ipset.sh; done
#chmod +x ipset.sh && ./ipset.sh
cat chnroutes.txt | xargs -I ip ipset add chnroutes ip
```
chnroutes你可以从自己喜欢的地方获取，github也很多。

#### 配置iptables
iptables是所有软路由代理部分的核心，没有系统差别，有闲的可以自己深入一点学习。

大陆白名单模式，下面的your_server一定要改，别一股脑复制执行。
```
# 新建一条链 100kg
iptables -t nat -N 100kg

# 保留地址、私有地址、回环地址 不走代理
iptables -t nat -A 100kg -d 0.0.0.0/8 -j RETURN
iptables -t nat -A 100kg -d 10.0.0.0/8 -j RETURN
iptables -t nat -A 100kg -d 127.0.0.0/8 -j RETURN
iptables -t nat -A 100kg -d 169.254.0.0/16 -j RETURN
iptables -t nat -A 100kg -d 172.16.0.0/12 -j RETURN
iptables -t nat -A 100kg -d 192.168.0.0/16 -j RETURN
iptables -t nat -A 100kg -d 224.0.0.0/4 -j RETURN
iptables -t nat -A 100kg -d 240.0.0.0/4 -j RETURN

# your_server 是 远程代理服务器的 ip，自己改
iptables -t nat -A 100kg -d your_server -j RETURN

# gfwlist重定向至 redsocks端口
iptables -t nat -A 100kg -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-ports 12345
# 大陆地址不走代理
iptables -t nat -A 100kg -m set --match-set chnroutes dst -j RETURN
# 其余的全部重定向至 redsocks端口
iptables -t nat -A 100kg -p tcp -j REDIRECT --to-ports 12345

# OUTPUT 和 PREROUTING 链添加一条规则，重定向至 100kg 链
iptables -t nat -A OUTPUT -p tcp -j 100kg
iptables -t nat -I PREROUTING -p tcp -j 100kg
```
链名字无所谓哈，100kg是隔壁项目的名字，不管你用什么都可以，只是个名。

持久化ipset和iptables配置：
```
netfilter-persistent save
```

透明代理已生效，试试。

#### gfwlist模式
***模式二选一，照搬的悠着点***

先将dnsmasq和gfwlist搭配起来，也就是特定域名解析为ip后动态加入到ipset里。列表获取项目 [https://github.com/cokebar/gfwlist2dnsmasq](https://github.com/cokebar/gfwlist2dnsmasq)。本项目下也预先放了一份在conf下，可自取。配置根据dnsmasq配置，放到下面目录
```
cd /routerfiles/foriptables/dnsmasq.d
```

上文iptable配置时已经将gfwlist强制走了代理，只需修改默认规则（最后一条）为不转发即可

查看当前规则，带行号
```
iptables -t nat -L 100kg --line-numbers
```
删除最后一条规则（默认转发），也可以同时删掉chnroutes那一条，看你需求（啰嗦：多条规则先删大的，因为删了小的后后面的行号就立即变化了）
```
iptables -t nat -D 100kg 行号
```
添加默认不转发。（这条在最后其实不需要加，加了就是漂亮一点）
```
# 默认不走代理
iptables -t nat -A 100kg -j RETURN
```
持久化保存
```
netfilter-persistent save
```

#### 国外DNS
因为有dns污染，所以需要通过代理使用国外的dns服务。
本文使用 [https://github.com/shawn1m/overture](https://github.com/shawn1m/overture) 。你也可以使用smart dns等。一般就是提供国内的域名国内解析，国外的域名或者gfwlist域名国外解析。

安装overture到/routerfiles/overture。自己改为最新版本
```
mkdir /routerfiles/overture
cd /routerfiles/overture
wget https://github.com/shawn1m/overture/releases/download/v1.6.1/overture-linux-amd64.zip
unzip overture-linux-amd64.zip
```
获取国内ip，国内域名，gfwlist域名
```
cd /routerfiles/overture
wget https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt
wget https://raw.githubusercontent.com/zfl9/chinadns-ng/master/chnlist.txt
curl https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt | base64 -d | sort -u | sed '/^$\|@@/d'| sed 's#!.\+##; s#|##g; s#@##g; s#http:\/\/##; s#https:\/\/##;' | sed '/\*/d; /apple\.com/d; /sina\.cn/d; /sina\.com\.cn/d; /baidu\.com/d; /qq\.com/d' | sed '/^[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+$/d' | grep '^[0-9a-zA-Z\.-]\+$' | grep '\.' | sed 's#^\.\+##' | sort -u > /tmp/temp_gfwlist.txt
curl https://raw.githubusercontent.com/hq450/fancyss/master/rules/gfwlist.conf | sed 's/ipset=\/\.//g; s/\/gfwlist//g; /^server/d' > /tmp/temp_koolshare.txt
cat /tmp/temp_gfwlist.txt /tmp/temp_koolshare.txt | sort -u > gfw_all_domain.txt
```
修改配置
```
vi /routerfiles/overture/config.json
```
修改以下内容：
```
...
"BindAddress": ":5353",
...
"AlternativeDNS": [
...
  "Address": "8.8.8.8:53",
  "SOCKS5Address": "127.0.0.1:你的代理的本地端口",
...
]
...
"IPNetworkFile": {
  "Primary": "/routerfiles/overture/china_ip_list.txt",
  "Alternative": "/routerfiles/overture/ip_network_alternative_sample"
},
"DomainFile": {
  "Primary": "/routerfiles/overture/chnlist.txt",
  "Alternative": "/routerfiles/overture/gfw_all_domain.txt",
  "Matcher":  "full-map"
},
"HostsFile": {
  "HostsFile": "/routerfiles/overture/hosts_sample",
  "Finder": "full-map"
},
...
"DomainTTLFile" : "/routerfiles/overture/domain_ttl_sample",
...
```
将dnsmasq的上游服务器指向overture,"127.0.0.1#5353"

加入开机启动
```
vi /lib/systemd/system/overture.service
```
文件内容：
```
[Unit]
Description=overtureService
After=network.target

[Service]
Type=simple
WorkingDirectory=/routerfiles/overture
ExecStart=/routerfiles/overture/overture-linux-amd64

[Install]
WantedBy=default.target
```
Golang的相对路径一直有些问题，不改源码的话在Service下用WorkingDirectory是一种方式。

设为开机启动并启动服务
```
systemctl enable overture
systemctl start overture
```

全文结束。欢迎issue。