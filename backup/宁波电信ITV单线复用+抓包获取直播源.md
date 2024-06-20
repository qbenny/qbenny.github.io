### 前言

在这个时代，我们已经习惯于使用 PC 和手机，通过互联网获取各种内容，这其中，视频节目是很重要的一部分。然而，许多电视剧和综艺节目都是在电视台首播的，但通过互联网观看电视台直播的体验一直都不好。虽然有些电视台官网或一些第三方渠道有不错的直播源，但这类直播会受到各种因素的影响，有时候并不能保证稳定流畅。另外，一些与视频平台有播放协议的节目，在电视台直播的时候会切断官网的直播信号，由此诞生了一批第三方电视直播App，只不过，那些App毕竟是第三方，直播的质量相对就不能得到保障，而且还可能需要另外付费开通会员等等。总之，“想看就看”似乎并不容易实现。

为了观看某些直播节目，笔者一直都在寻找长期稳定可靠，质量较好，且不受某些节目影响的观看电视直播的方法。前段时间，突发奇想，家里的电信套餐是带电信ITV的，虽然已经开通好几年了，但因为家里有广电的数字电视，电信ITV长期处于吃灰状态。ITV就是通过互联网获取电视直播或点播内容，把ITV的网络接入到宽带的网络后，不但把ITV利用了起来，而且还能比较完美的解决上面所提到的问题。ITV接入的是电信的专用网络，质量有保证，还不占用带宽。不仅如此，原本ITV需要接入光猫的专用接口才能使用，融合到宽带网络以后，不但可以接入任何一个路由器的lan口，甚至还可以直接通过WiFi使用，真是一举多得！想到就做，于是就有了这篇折腾记录。

事先声明，本文涉及到的某些内容比较复杂，由于本文只是个人折腾完成后的记录，因此不会解释各种基础知识，下面的内容默认您已经对光猫、路由器的配置及插件的安装，DHCP，VLAN，抓包，单播和组播等内容有了一定的了解。

### 光猫的设置

先来介绍一下这边的环境。宁波电信，光猫为华为HS8145C，一个千兆口，一个ITV口，两个百兆口。千兆口接了一个2网口的软路由，系统为 OpenWrt。因为软路由的网口不够，所以在他的lan口接了一个随手在某宝购买的最普通，没有任何设置功能的TP-Link交换机，在交换机上接了两个路由器，他们只作为AP，整个网络都由软路由处理。

动手之前，在网上找到了一篇介绍杭州电信ITV接入方式的文章，估计宁波电信的ITV大同小异，应该可以直接拿来参考，于是就好好参考了一番，本文的大部分操作都来源于此，在此感谢。

作者的接入方式是通过一根线，直接让上网和ITV都走光猫的千兆口。因为这个软路由网口不够，所以也打算参考这篇文章进行操作。如果接入方式是直接另外从光猫的ITV口接一根线到路由器的另一个网口的话，下面介绍配置光猫的部分无需操作，设置路由器的部分请按照实际的接口进行设置，无需VLAN。

设置光猫的第一步就是拿到电信的超级用户 telecomadmin 账号的密码。找了电信的装维师傅，但他说他也不知道密码，当然，到底是真的不知道还是不愿意告诉我就不好说了。那好吧，只能自己想其他办法获取了。至于最后怎么搞定他，在这里就不展开了，请自行解决。

拿到超级用户以后，首先登录快速装维入口，选择网络-网络设置-网络连接，将上网和ITV都改为桥接。在这个页面可以看到电信默认有3条线路，分别为

- 1_TR069_VOICE_R_VID_46
- 2_INTERNET_B_VID_41
- 3_OTHER_B_VID_43

从名称和查看到的对应设置可以发现， Internet 上网对应的VLAN ID 为 41，ITV为43。然后点击VLAN绑定，将千兆口的ID41绑定到 2_INTERNET_B_VID_41，将千兆口的43绑定到 3_OTHER_B_VID_43。设置完成后的效果如下表所示：

| 用户侧端口 | 用户侧VLAN | WAN侧VLAN |
| --- | --- | --- |
| 千兆口 | 41 | 41 |
| 千兆口 | 43 | 43 |

[**单线融合宽带iptv，不需vlan、丢掉光猫，任意设备看iptv【浙江电信】【Openwrt】**](https://www.notion.so/iptv-vlan-iptv-Openwrt-afe854d0575c45099d3f54efd486e2ba?pvs=21)

在实际使用中，发现这边用41拨号的话，就会不定时的断开连接重新拨号，但不指定VLAN ID 使用起来就一切正常，所以目前没有用到41，如果哪位大佬知道原因请告知。至此，光猫的设置全部完成！

### 获取ITV认证信息

配置完光猫以后，就可以正式开始折腾盒子了。首先获取ITV盒子的连接信息。还是因为网口的问题，这一次就先从光猫的ITV另外接了一根线到交换机，这样，机顶盒只要随意插在路由器的lan口，就可以连接ITV网络了。如果本来就是另接线从ITV口到路由器的话，就去路由器的接口里创建一个新接口，选择桥接，将ITV所在的口和机顶盒使用的口桥接起来，协议选择不配置。完成后，使用路由器的 tcpdump 插件进行抓包，捕获盒子所在 lan 口的流量，开始捕获，让机顶盒联通网络和电源，进行联网。稍等一会儿就联网成功了，此时停止捕获，将生成的.pcap文件下载回来，使用 [Wireshark](https://www.wireshark.org/index.html#download) 读取这个文件进行分析。

宁波电信的ITV认证方式为IPOE，也就是网上说的DHCP+鉴权模式。在过滤选择器的编辑框输入**dhcp**，就可以找到盒子发送的dhcp请求了。

在列表中选择盒子发送的dhcp包，在分组详情里可以看到*Ethernet II, Src: zte_b0:2c:c4 (xx:xx:xx:xx:xx:xx), Dst: Broadcast (ff:ff:ff:ff:ff:ff)*，是个中兴的盒子。这里有个用得到的信息，就是括号内的盒子MAC地址，把它保存下来。

继续分析数据包的内容。寻找并展开*Dynamic Host Configuration Protocol (Discover)*，继续展开*Option: (12) Host Name*，之后就可以看到**Host Name: XXXXX**。这是发送给服务器的主机名，是个铭文，内容其实就是盒子的序列号和MAC地址。直接把内容复制下来即可，可以在 HostName： 这一行右键菜单复制-值。

继续找到 *Option: (60) Vendor class identifier* 展开，找到*Vendor class identifier:*，右键菜单，选择复制-**…as a Hex Stream**，这就是认证的关键信息，包含了ITV的账号密码，当然，它是加密的，所以我们就直接复制Hex，到时候让路由器按照原样把数据发出去就好了。

此时已经获取了连接ITV网络所需的全部信息，接下来，就可以配置路由器了。

### 使用路由器连接ITV网络

本次配置的是 x86_64 的软路由，使用 OpenWrt 系统，其他路由器和系统环境请自行寻找对应的操作方法。

首先进入路由器后台，确认路由器WAN口的网卡名称，这边是 eth1。进入接口页面，新建一个接口，命名为“IPTV”，协议为“DHCP客户端”，接口选择自定义，并输入“eth1.43”（eth1.43表示使用网卡eth1的VLAN ID 43）,点击提交。在新页面中，将之前获取的 Host Name 的值填写在“请求 DHCP 时发送的主机名”编辑框中。点击高级设置，在“重设 MAC 地址”编辑框中填写盒子的MAC地址，并在“使用网关跃点”编辑框填写一个数字，例如20。在防火墙设置页面，新建一个区域，也叫做“IPTV”。点击“保存&应用”按钮完成新接口的配置。回到接口页面后，点击进入wan接口的配置页面，在高级设置页面给它的网关跃点设置一个比IPTV更小的值，例如10，这样配置的目的是让wan的网关比IPTV的网关优先级更高。

然后进入网络-防火墙，找到新建的防火墙区域“IPTV”，点击修改按钮，在修改界面修改“入站数据”和“转发”为“拒绝”，勾选“IP 动态伪装”和“MSS 钳制”，在最下方端口触发中，允许从源区域转发那里勾选“lan”后，点击“保存&应用”。这样，获取到IP后就可以直接ping通ITV的网关了。

经过以上的设置，离成功使用路由器接入ITV网络已经不远了。下面的操作需要通过ssh连接到路由器上进行。首先配置网络，把获取到的 Vendor class identifier 的值写入配置文件中。通过ssh连接到路由器，编辑 **/etc/config/network** 文件，在IPTV接口的配置段中加入 `option sendopts '0x3c:xxxxxxxx'`（0X3C为十六进制的60，xxxxxxxx为Hex的内容）即可。下面提供完整IPTV接口配置供参考：

```
config interface 'IPTV'
    option proto 'dhcp'
    option ifname 'eth1.43'
    option delegate '0'
    option macaddr 'xx:xx:xx:xx:xx:xx'
    option hostname 'xxxx'
    option sendopts '0x3c:xxxxxxxx'
    option metric '20'
```

另外还必须修改 OpenWrt 自带的 udhcpc 脚本，如果不修改的话，会导致他发出两个 option60，因此还是无法成功连接。

编辑文件 **/lib/netifd/proto/dhcp.sh** ，找到

```
Vendor
`    proto_run_command "$config" udhcpc \
        -p /var/run/udhcpc-$iface.pid \
        -s /lib/netifd/dhcp.script \
        -f -t 0 -i "$iface" \
        ${ipaddr:+-r $ipaddr} \
        ${hostname:+-x "hostname:$hostname"} \
        ${vendorid:+-V "$vendorid"} \
        $clientid $defaultreqopts $broadcast $release $dhcpopts
```

修改为

```
    proto_run_command "$config" udhcpc \
        -p /var/run/udhcpc-$iface.pid \
        -s /lib/netifd/dhcp.script \
        -f -t 0 -i "$iface" \
        ${ipaddr:+-r $ipaddr} \
        ${hostname:+-x "hostname:$hostname"} \
        -V '' \
        $clientid $defaultreqopts $broadcast $release $dhcpopts
```

此时重连IPTV接口，就成功获取到IP地址了。

### 获取ITV直播源

获取ITV直播源还是需要通过抓包完成。在抓包之前，首先需要把盒子的联网方式从原先的 IPOE 模式改为 DHCP 模式，直接连接到路由器的网络，并且让盒子全部走ITV的网络。如果路由器有 mwan3 插件，可以通过它完成设置，但这里没有，所以就添加了一个路由表规则实现这个功能。然后就可以愉快的抓包了。开启流量捕获，开启机顶盒，联网后播放一个直播视频，停止捕获，下载文件进行分析。

各地的数据都不太一样，但一般来说直播源都分为单播的 rtsp 和组播的 igmp ，因此，这一次在过滤里输入 `http && http contains "igmp://"` 或 `http && http contains "rtsp://"` ,意思是过滤出带有引号中内容的 http 包。以本次数据为例，过滤后可以看到一个文件大小277190字节的包。Line-based text data: text/html (882 lines)。这是个882行的html数据包。在这一行按下 ctrl+shift+x 导出分组字节流，把内容保存为一个文件，然后使用文本编辑工具查看里面的内容。在里面查找频道名称或者igmp、rtsp等关键词，就可以找到个频道的播放地址了。

下面随便展示一行频道的原始数据，这其中去除了ITV账号、盒子序列号等信息。

`jsSetConfig('Channel','ChannelID="ch00000000000000001236",ChannelName="浙江卫视高清",UserChannelID="6002",ChannelURL="igmp://233.50.201.100:5140",TimeShift="1",ChannelSDP="igmp://233.50.201.100:5140|rtsp://220.191.144.18:554/live/ch15112318015894677737.sdp?playtype=1&boid=001&backupagent=220.191.144.18:554&clienttype=1&time=20201025110803+08&life=172800&ifpricereqsnd=1&vcdnid=001&userid=xxx&mediaid=ch15112318015894677737&ctype=5&TSTVTimeLife=14400&contname=&authid=0&UserLiveType=1&stbid=xxxxx&nodelevel=3&terminalflag=1&bitrate=2000",TimeShiftURL="rtsp://220.191.144.18:554/live/ch15112318015894677737.sdp?playtype=1&boid=001&backupagent=220.191.144.18:554&clienttype=1&time=20201025110803+08&life=172800&ifpricereqsnd=1&vcdnid=001&userid=xxxxx&mediaid=ch15112318015894677737&ctype=5&TSTVTimeLife=14400&contname=&authid=0&UserLiveType=1&stbid=xxxxx&nodelevel=3&terminalflag=1&bitrate=2000",ChannelLogURL="",PositionX="0",PositionY="0",BeginTime="0",Interval="-1",Lasting="0",ChannelType="2",ChannelPurchased="",LocalTimeShift="0",UserTeamChannelID="2",FCCFunction="1",ChannelFCCIP="220.191.136.24",ChannelFCCPort="8027"');`

拿到频道数据以后，通过一些自己习惯的方式，将频道名称和地址整理成 m3u 播放列表文件，对路由器进行设置后，就可以正常使用了。

### 使用直播源

要使直播源能在PC、手机等平台使用，就需要对路由器进行一些设置，这里主要用到的插件有：

- igmpproxy ——让ITV接口的组播地址能直接在局域网中使用
- udpxy ——将组播igmp地址转为http单播地址

另外还用了一些辅助脚本，用于添加路由规则等。下面就来一一说明。

udpxy 和 igmpproxy 的配置，可以直接编辑 /etc/config 目录中的对应文件，这里直接把配置贴上来，并进行简单说明。首先是 igmpproxy：

```
config igmpproxy
option quickleave 1
config phyint
option network IPTV # IPTV 为创建的接口名称，注意大小写
option direction upstream
list altnet 0.0.0.0/0
config phyint
option network lan
option zone lan
option direction downstream
```

接着是 udpxy 的配置：

```
config udpxy
    option respawn '1'
    option verbose '0'
    option status '1'
    option port '4022' # udpxy 服务端口
    option bind 'br-lan' # 在指定接口或IP地址上监听端口
    option disabled '0'
    option source 'eth1.43' # IPTV 的接口，注意需填写网卡，而不是接口名，也就是在 OpenWrt 接口里创建的IPTV接口后面括号里的内容
    option max_clients '50' # 允许的最大连接数，默认为3，最大500
    option mcsub_renew '60' # 自动续订组播地址的时间（单位为每秒），默认值为0，也就是不续订。
```

这里重点吐槽一下 mcsub_renew 这个参数，这个默认配置让我踩了好大一个坑。一开始配置完服务以后可以正常使用，然而发现不管播放什么，一到4分钟左右就一定会停止，但重新播放又是正常的。最后通过仔细研究了各项参数的意义，才知道原来是这个参数的默认值搞的鬼。

其他的配置请自行查看帮助。

完成编辑并启动服务后，可以访问 `http://路由器IP:udpxy端口/status` 查看 udpxy 运行状态。没有问题的话，此时就已经可以通过 udpxy 提供的服务进行播放了。例如，浙江卫视高清的地址为`igmp://233.50.201.100:5140` ，使用 udpxy 转换后的地址为： `http://192.168.1.1:4022/udp/233.50.201.100:5140`。在状态页面，除了可以看到相关信息外，还有使用帮助，其他使用方法请自行查看。

网上有一些教程，还添加了防火墙的规则，允许IPTV的 igmp 数据和 udpxy 服务，顺手也加上。编辑 **/etc/config/firewall** ，加入以下内容：

```
config rule
    option target 'ACCEPT'
    option src 'IPTV'
    option name 'Allow-IGMP'
    option proto 'IGMP'

config rule
    option target 'ACCEPT'
    option src 'IPTV'
    option proto 'udp'
    option name 'Allow-UDP-igmpproxy'
    option family 'ipv4'
    option dest 'lan'
    option dest_ip '224.0.0.0/4'

config rule
    option target 'ACCEPT'
    option src 'IPTV'
    option proto 'udp'
    option name 'Allow-UDP-udpxy'
    option dest_ip '224.0.0.0/4'
```

为了使用 rtsp 地址，需要把 rtsp 服务器IP所在网段加入到路由表，让这个网段走ITV网关。下面是一部分宁波电信ITV用到的IP段，参考了介绍杭州电信ITV的那篇文章中的脚本，使用前请自行抓包获取当地IP后修改脚本。同时，还添加了一条让盒子整个走ITV网络的规则。其中的IP、接口、网关地址等信息务必自行修改。将此脚本放在 */etc/init.d* 中，并赋予执行权限后即可使用。

```
#!/bin/sh /etc/rc.common

START=99
STOP=15

boot() {
    ip rule flush table 100
    ip route flush table 100
    ip rule add from 192.168.1.11 to 0.0.0.0/0 table 100 # 让盒子全部走ITV网关，其中 192.168.1.11 请修改为自己的盒子IP
    ip rule add from 192.168.1.0/24 to 122.229.17.0/24 table 100
    ip rule add from 192.168.1.0/24 to 220.191.136.0/24 table 100
    ip rule add from 192.168.1.0/24 to 220.191.144.0/24 table 100
    ip rule add from 192.168.1.0/24 to 220.191.139.0/24 table 100
    ip rule add from 192.168.1.0/24 to 10.255.247.0/24 table 100
    ip rule add from 192.168.1.0/24 to 115.233.200.0/24 table 100
    ip rule add from 192.168.1.0/24 to 60.130.250.0/24 table 100
    ip route add default via 10.164.128.1 dev eth1.43 table 100
}

start() {
    boot
}

stop() {
    ip rule flush table 100
    ip route flush table 100
}

restart() {
    boot
}
```

在使用过程中，还发现了一些问题，例如当 IPTV 接口重新连接，获取新IP后， udpxy 竟然不会自动使用新IP，路由表规则也会丢失，导致无法使用。于是，还需要用到另一个脚本，当检测到此接口重连后重启这两个服务。

在 */etc/hotplug.d/iface/* 目录创建一个脚本，命名为 **99-iptv.sh** ，内容如下：

```
#!/bin/sh
[ "$ACTION" = ifup ] && [ "$INTERFACE" = IPTV ] &&
/etc/init.d/udpxy restart
/etc/init.d/iptvrule restart
```

编辑完成后赋予脚本执行权限，从此再也不用担心IP地址更换后无法使用了！

### 总结

经过上面的各种折腾，终于可以愉快的通过PC、手机等设备观看ITV电视直播了。不知不觉中发现，这似乎是本人写过的最长的一篇文章了。正如前言中所说，这篇文章其实只是一篇折腾记录，并不是教程，因为想要实现这个目的，对于我这个小白来说，确实是踩了很多坑，也学到了很多。我这个懒癌晚期患者写了这么多，目的就是为了以后再次动手的时候能避开踩过的坑，如果能帮助到一些朋友就更好了。

最后，感谢能看到这里的朋友们，希望我这个不会写文章的人写出来的东西能让大伙儿看明白！

[http://i7q.cn/5NdLpU](http://i7q.cn/5NdLpU)

[https://raw.githubusercontent.com/qbenny/iptv_cixi/main/iptv_cx.m3u](https://raw.githubusercontent.com/qbenny/iptv_cixi/main/iptv_cx.m3u)

[http://i7q.cn/6vApDI](http://i7q.cn/6vApDI)

[https://raw.githubusercontent.com/qbenny/iptv_cixi/main/iptv_hz.m3u](https://raw.githubusercontent.com/qbenny/iptv_cixi/main/iptv_hz.m3u)














































































































































































































































































































































































































































