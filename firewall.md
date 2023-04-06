
firewall3应用程序 ，（现在是4）

内核中的netfilter钩子

kmodule用来检查数据包

一组内核参数，用来配置堆栈和防火墙模块



1，firewall应用程序

运行在用户态中，主要用来生成netfilter/nftables规则并发送到内核中的kmodule模块

目前的fw4已经使用nftables来代替iptables来配置linux的网络规则

（关于nftables之后详谈）

fw4程序通过配置文件/etc/init.d/firewall管理

service_triggers() {
procd_add_reload_trigger firewall
}

restart() {
fw4 restart
}
start_service() {
fw4 ${QUIET} start
}

stop_service() {
fw4 flush
}

reload_service() {
fw4 reload
}

boot() {
# Be silent on boot, firewall might be started by hotplug already,
# so don't complain in syslog.
QUIET=-q
start
}
boot：这是在系统初始化（引导）过程中调用的

start：解析配置文件并写入netfilter内核模块

stop：从内核模块中清除配置规则（它们不会被卸载）

restart，reload：从内核读取netfilter规则，使用配置文件进行替换，然后写回netfilter内核模块。

flush：（危险）删除所有规则，删除非默认链，并将默认策略重置为ACCEPT。

在后台，/etc/init.d/firewall调用fw4，将参数传递给二进制文件。在某些情况下，参数将伴随额外的标志来抑制日志消息，或者调用如上所述的内部函数来验证配置文件。



选项syn_flood 1或选项mtu_fix 1各自转换为复杂的nftables规则。

选项masq 1转换为NAT的“-j MASQUERADE”目标。

问：选项在哪配置的？文章里面并没有体现出来 ﻿王飞杨10311248﻿

这些选项基本都是在/etc/config/firewall里面配置的，下面的防火墙关键配置文件会进行介绍

而且目前的fw4转换成nftables而不是iptables，可能相关的文档不在准确，还需实验下



2，netfilter钩子

这个不用过多介绍，还是内核里的经典老一套，PRE_ROUTING、LOCAL_IN、FORWARD、LOCAL_OUT、POST_OUTING



3，kmodule模块

加载哪些模块取决于配置，相关的内核模块很多，一般都只提供一个特定功能，比如进行ip端口的匹配，进行分段调整，哪些目标reject等等

nf_conntrack负责链接跟踪



4，一组内核参数，用来配置堆栈和防火墙模块

通过/etc/init.d/sysctl这个shell脚本来配置，在引导时会执行

可以看出他会执行apply_defaults函数，并且会加载/etc/sysctl.conf以及/etc/sysctl.d/*.conf所有配置

start() {
apply_defaults
for CONF in /etc/sysctl.d/*.conf /etc/sysctl.conf; do
[ -f "$CONF" ] && sysctl -e -p "$CONF" >&-
done
}
目前内核中的netfilter桥接禁用了可能要手动开启

bridge-nf-call-iptables - BOOLEAN
1 : pass bridged IPv4 traffic to iptables' chains.
0 : disable this.
Default: 1


二，防火墙关键配置文件/etc/config/firewall

LUCI和UCI都是通过最终修改这个文件来配置防火墙，

/etc/config/firewall在修改后，需要通过上面介绍过的/etc/init.d/firewall脚本来进行reload配置

注：目前网上的配置相关资料都是对于fw3的，目前的fw4不知道具体变化在哪，但比较配置文件目前未发现差异

/etc/config/firewall文件当中配置一般是这种形式，

config  <名称>
option  <设置>  <选项>
而其中所包含的config项有

1.defaults         声明了不属于特定区域的全局防火墙设置

2.zone              部分将一个或多个网络接口组合在一起，作为转发、规则和重定向的源或目的

3.forwarding    控制 zone 之间的转发, 可以实现特定方向的 MSS clamping      

4.rule                用于定义基本的接受、丢弃或拒绝规则，以允许或限制对特定端口或主机的访问

5.redirect         DNAT SNAT

6.IP sets           支持引用或创建IP 集以简化大型地址或端口列表的匹配，而无需为每个项目创建一个规则进行匹配。（这个在fw4中还没看）

7.Includes        用于添加自定义的防火墙脚本

 注：最大區段長度 (Maximum Segment Size, MSS)。MSS = MTU - 20 octet (TCP 固定表頭) - 20 octet (IP 固定表頭)

而clamping以 PPPoE 為例，會多 8 ~ 12 bytes 的標頭檔，所以 MSS 必須再少 8 ~ 12 bytes。要求全部 client 更改 OS 的 MSS 設定並不實際，所以現行的作法是 PPPoE 的 server 會偷改 SYN 封包裡的 MSS ，讓它不要超出上限，這個行為稱為 MSS clamping（ＭＳＳ箝位）。也就是說 client 以為它用 MSS = 1460，但 PPPoE server 在經手的時候改成 MSS = 1452 (假設 PPP 標頭是 8 bytes)。　　

(一个台湾的朋友说的)



defaults







zone

openwrt防火墙中的域(zone),可以将一个或者多个接口归到一个域内。一般我们所熟知的安全域有lan和wan。它们通常是几个接口的集合。通过对接口域的划分，简化了防火墙的规则逻辑

config zone
option name      lan
list   network  'lan'
option input     ACCEPT
option output    ACCEPT
option forward   ACCEPT

config zone
option name       wan
list   network   'wan'
list   network   'wan6'
option input      REJECT
option output     ACCEPT
option forward    REJECT
option masq1
option mtu_fix1


bridge firewall

在超哥整理的框架文档中，通过network配置文件可以生成一个br-lan桥接口，通过这个接口可以来管理无线和有线的端口，同时也有针对桥接口的bridge firewall
