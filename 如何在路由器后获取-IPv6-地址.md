中山大学使用 Router Advertisement（路由通告，下文简称 RA）下发一个 /64 地址段，让终端通过 SLAAC （无状态地址自动配置）得到一个 IPv6 地址。通常一栋楼共用同一个 /64 地址段，不能再通过 Prefix Delegation（前缀下发）获取另外的地址段。

可用的有以下方案：
## 1. NAT66

与目前 IPv4 的情况类似，路由器 WAN 口获得一个 IPv6 地址，LAN 口使用内网 IPv6 地址，LAN 设备共用 WAN 的 IPv6 地址上网。
* 极路由

     极路由已自带 IPv6 NAT 功能，由 [NAPT66](https://github.com/mzweilin/napt66) 实现。

     进入设置页面，点击“云插件”，进入插件页后添加“教育网 IPv6”插件，选择“双栈接入”后一路下一步即可。内网设备将会获得 `4006:e024:680:<路由器 MAC 地址后 16 位>::/64` 的地址。

* OpenWRT/LEDE 类固件

     OpenWRT/LEDE 可使用 `kmod-ipt-nat6` 内核模块实现 NAT66。

     进入设置页面→网络→接口，在“IPv6 ULA 前缀”中填入内网 IPv6 地址段，如 `2001:db8:1234:5678::/64`，注意不能是保留地址或与某些网站的 IPv6 地址冲突。点击“保存&应用”；

     点击 LAN 的“编辑”，在 “IPv6 设置” 中把“路由器广告服务”（Router Advertisement）设置为“服务器模式”， DHCPv6 和 NDP 设置为禁用，“总是广播默认路由”选项打勾。

     进入系统→软件包，先刷新列表，然后安装 `kmod-ipt-nat6`（填入“下载并安装软件包”内点确认）；

     进入网络→防火墙→自定义规则，添加一行 `ip6tables -t nat -I POSTROUTING -s 你在步骤 1 中填入的地址段 -j MASQUERADE` ，点击“重启防火墙”。

* 华硕 Merlin/Padavan 固件：

     这类固件一般没有集成这类功能，只能自行编译集成 [NAPT66](https://github.com/mzweilin/napt66) 模块的固件。Padavan 可参考 http://www.jianshu.com/p/3a9ec169336e

## 2. IPv6 中继

在 WAN 口 与 LAN 口之间代理 NDP（邻居发现协议，作用类似于 IPv4 的 ARP），让网关知道发往路由器 LAN 下某设备的 IPv6 地址需要先经过路由器 WAN 口。LAN 设备因此可以得到公网 IPv6 地址。

* 极路由

     极路由已自带 IPv6 中继功能，由 [6relayd](https://github.com/sbyx/6relayd) 实现。

     进入设置页面，点击“云插件”，进入插件页后添加“教育网 IPv6”插件，选择“IPv6 中继”后一路下一步即可。内网设备将会获得公网地址。

     **注意：目前 6relayd 已被弃用，且实践证明不是特别稳定，存在掉线的情况，在 LAN 下长时间 ping 外网 IPv6 地址可以缓解**

* OpenWRT/LEDE 类固件

     早期版本的 OpenWRT 可以使用 6relayd，在系统→软件包中安装即可，但是缺点同上。现已被 odhcpd 取代：

     SSH/Telnet 登录路由器，编辑 `/etc/config/dhcp` ，修改 LAN 和 WAN6 的内容如下（如果原本没有 WAN6 的内容就直接添加）：
```
config dhcp 'lan'
	option dhcpv6 'disabled'
	option ra 'relay'
	option ndp 'relay'
config dhcp 'wan6'
	option interface 'wan'
	option dhcpv6 'disabled'
	option ra 'relay'
	option ndp 'relay'
	option master '1'
```


&emsp;&emsp;保存后重启 odhcpd （命令行下 `/etc/init.d/odhcpd restart` ，或设置页面的系统→启动项，点 odhcpd 旁边的重启）。<br>
&emsp;&emsp;内网设备将会获得公网地址。<br>&emsp;&emsp;**注意：odhcpd 似乎也不是特别稳定，在 LAN 下长时间 ping 外网 IPv6 地址可以缓解**
* Linux 通用方法

     将 WAN 口设置为静态 IPv6 地址，前缀长度 /128 ，然后把原先在 WAN 侧通告的 /64 地址段“强行”划分给 LAN，并使用 [ndppd](https://github.com/DanielAdolfsson/ndppd) 在 WAN 和 LAN 间代理 NDP，内网设备将会获得公网地址，且不存在掉线问题。

     在设置为静态地址之前，先用 `ifconfig`，`ip -6 route` 等命令查看 WAN 口的 IPv6 地址和 IPv6 网关。IPv6 网关为 `fe80::` 开头的链路地址。

     设置 WAN/LAN 口静态地址：
     * OpenWRT/LEDE 类系统：修改 WAN6 的设置，切换协议为静态地址并填入 IPv6 /128 地址和 IPv6 网关，在“IPv6 ULA 前缀”中填入 /64 地址段，在 LAN “IPv6 设置” 中把“路由器广告服务”（Router Advertisement）设置为“服务器模式”， DHCPv6 和 NDP 设置为禁用，“总是广播默认路由”选项打勾。
     * 极路由：修改 `/etc/config/network` ，在 `config interface 'wan'` 下加入 `option ip6addr 'WAN 口地址/128'` 和 `option ip6gw '网关地址'` 两行，保存后执行 `ifup wan` 重启 WAN 接口。使用 radvd 为 LAN 通告 /64 地址段。
     * 华硕 Merlin/Padavan 固件：直接在 IPv6 中设置为静态地址，填入 IPv6 地址（LAN 侧前缀长度 64， WAN 侧 128）和 IPv6 网关，打开 Router Advertisement。

     设置完后先在路由器上测试 IPv6 连通性，再进入下一步。

     安装 ndppd：
     * OpenWRT/LEDE 类固件：直接用 系统→软件包 安装 ndppd。如果无法安装，可能需要手动从 OpenWRT/LEDE 官网下载 ipk 到路由器上并 `opkg install 文件名` 安装之。
     * 极路由：同上，可使用 OpenWRT <s>15.05</s> 14.07 的 RAMIPS(MTK CPU) 软件源，但可能需要修改 `/etc/opkg.conf` ，见 https://sourceforge.net/p/openwrt-dist/wiki/Home/?version=14 。
     * 华硕 Merlin/Padavan 固件：安装 [Entware-ng](https://github.com/Entware-ng/Entware-ng) 后可直接使用 opkg 安装。无法安装 Entware 的请自行编译或 Google 已经编译好的文件。

     编辑 ndppd.conf，OpenWRT/LEDE/极路由的路径为 `/etc/ndppd.conf` ，其他固件须自行指定路径：
```
route-ttl 30000
address-ttl 30000

proxy WAN 口名称 {
   router yes
   timeout 500
   autowire no
   keepalive yes
   retries 3
   promiscuous no
   ttl 30000
   rule 2000::/3 {
      iface LAN 口名称
      autovia no
   }
}

``` 
&emsp;&emsp;注意修改 WAN 口和 LAN 口名称为 ifconfig 中显示的名称， rule 后的 `2000::/3` 修改为 LAN 口的 /64 地址段（默认的 `2000::/3` 为代理所有 NDP 请求）

* 开机启动 ndppd ：

     * OpenWRT/LEDE/极路由： /etc/init.d/ndppd enable

     * 其他： 在固件提供的开机启动脚本中加入 `ndppd -c ndppd.conf -d` ，注意 ndppd 和 ndppd.conf 必须为绝对路径，如 `/opt/bin/ndppd`，`/opt/etc/ndppd.conf`


     重启路由器，检查 LAN 侧的 IPv6 连通性。

     **注意：此方法存在以下几个缺点，但基本不影响使用：**
     * 将原本在 WAN 侧的 /64 地址段强行划分给 LAN，会导致处于相同 /64 段的 WAN 和 LAN 设备无法使用 IPv6 直接互通。应对方法：使用 IPv4，或再开一个 ndppd 进程把 LAN 的 NDP 代理到 WAN（新建一个 conf，交换 WAN 和 LAN 的名称，将 autowire 设置为 yes）
     * 有极小的概率 （n*2<sup>-64</sup>，n 为已获取 IPv6 地址的设备） 会使分别处于 WAN 和 LAN 侧的两个设备 SLAAC 到同一个地址，导致 IP 冲突上不了网。应对方法：重启设备或关闭 SLAAC 隐私扩展。
     * LAN 设备在两个处于同一个 /64 段，且都使用了 IPv6 中继方案的路由器下漫游时，其 IPv6 地址可能不会改变，这将导致因网关缓存了该地址的上一个 NDP 信息而无法把 IPv6 包转发到正确的路由器。应对方法：重启设备。

## 3. IPv6 桥接
**警告：该方法不适用于本校的环境，仅供参考**

使用 ebtables 和 brctl 在路由器 WAN 口和 LAN 口间建立一个只允许 IPv6 数据包通过的桥。此时路由器变成了交换机。

* 极路由

     极路由已自带 IPv6 桥接功能。

     进入设置页面，点击“云插件”，进入插件页后添加“教育网 IPv6”插件，选择“IPv6 桥接”后一路下一步即可。内网设备将会获得公网地址。

* 其他

     将以下命令添加到自定义防火墙规则中：
```
modprobe ip6table_mangle
ebtables -t broute -A BROUTING -p ! ipv6 -j DROP -i WAN 口名称
brctl addif LAN口名称 WAN口名称
```
&emsp;&emsp;注意修改对应的 WAN 口和 LAN 口名称，OpenWRT/LEDE 可能需要先安装 ebtables
* 此方法的缺点：
     * 路由器本身获取不到 IPv6 地址了，因此无法在路由器上使用需要 IPv6 地址的一些软件（如 本项目或 IPv6 SS）
          * 尝试手动添加 IPv6 地址到 LAN 口并添加默认路由到网关
     * LAN 设备的 MAC 地址被直接暴露给网关，因此不适用于 IPv6 需认证才能上的环境（如本校）
          * 但如果路由器接在了已经认证的且使用 NAT66 或中继方案的主路由器下，便可使用此方法