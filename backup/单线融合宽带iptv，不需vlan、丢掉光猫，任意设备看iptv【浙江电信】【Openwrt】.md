视频地址： [IPTV +宽带单线融合效果演示，丢掉机顶盒，任意设备看iptv 浙江电信 Openwrt](https://www.bilibili.com/video/BV1PW4y1S7MK)

# **前言**

因为家里的弱电箱在卧室，通往各个房间的网线也都在弱电箱，所以主路由器基本只能放在弱电箱里，而电信的iptv跟宽带是两根线而且到客厅只有一根网线，为了保证电视/机顶盒有稳定的网速(千兆)，但这样就必须在iptv和宽带之间二选一，除非装修前提前多埋线，相信大部分人都会存在这个纠结。我的折腾之路也由此开始～

效果先演示下:

[02:25IPTV +宽带单线融合效果演示，丢掉机顶盒，任意设备看iptv 浙江电信 Openwrt 1.1万  83视频_小愚_](https://www.bilibili.com/video/BV1PW4y1S7MK)

在看别人改造方案的时候，了解大致有几种选择：

1.分线器方案，只需要买一对分线器，将一根八芯的千兆线改成2跟4芯的百兆网线就行了。该方案成本极低，淘宝20块钱就能搞定，但缺点是无法享受原来的千兆网络，连到客厅的宽带只有百兆。

[https://i0.hdslb.com/bfs/article/6289297265813e26170cd8914649587c1cef65bb.png@1256w_694h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/6289297265813e26170cd8914649587c1cef65bb.png@1256w_694h_!web-article-pic.avif)

2.光猫 宽带+iptv 两根线(光猫支持融合一根线的话就是一根，不支持的话只能两根)到主路由器里划分vlan，然后在需要机顶盒的房间部署一个支持vlan的交换机，把宽带和iptv信号拆分出来，便可正常使用iptv和 千兆网络。该方案优势是复用网络，千兆宽带基本不受影响。缺点是每个想看iptv的位置都要准备一个支持vlan的交换机，看电视依然需要机顶盒

3.宽带+iptv融合进宽带网络，网关处配置udp代理，局域网内通过http请求访问代理获得视频流，可以在任意支持该协议的软件中播放iptv。缺点是需要网关支持udp代理，iptv播放信息需要通过抓包获取，比较麻烦。

4.宽带+iptv融合进宽带网络，网关处配置igmp代理，局域网内通过igmp组播信号的方式获得视频流，可以在任意支持该协议的软件中播放iptv。缺点是需要网关支持igmp代理，iptv播放信息需要通过抓包获取，比较麻烦。

起初的问题是如何解决一根线满足iptv和宽带问题，但后来发现甚至可以完全不需要机顶盒，iptv播放直接统一到一个电视盒子或电视自带系统里，那我当然选择可以直接放弃机顶盒的方案啦～

选择的实现方案是方案3，方案4在实操的过程中容易出现广播风暴造成网络异常（从别人方案的反馈来看），方案3就是一个纯http转udp协议的代理，逻辑清晰没有太多的风险点当然选它啦。**额外需要：一个Openwrt的路由器（支持抓包、dhcp option60鉴权、udpproxy功能的其他路由器亦可）。**

上个最终效果图

[https://i0.hdslb.com/bfs/article/watermark/aade72db7d8112f7479ecde696d5e5676601a55f.jpg@1256w_942h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/watermark/aade72db7d8112f7479ecde696d5e5676601a55f.jpg@1256w_942h_!web-article-pic.avif)

# **方案实现—抓包**

登陆光猫管理员账号，查看iptv和宽带是否可以融合在绑定在同一网口，刚开始调研vlan方案时觉得很遗憾，我的光猫只有两个口且不能把光猫和iptv绑定在一个口里。但后来发现这样走方案3、方案4时反而很简单。

首先我已知电信iptv的连接方式是dhcp鉴权，有些其他地方不需要鉴权或者直接使用ppoe拨号的方式，虽然我的方案是针对dhcp鉴权但他们的实现原理类似。如果iptv提供方可以直接提供拨号或者登陆iptv的账号密码那我们就可以直接在路由器上配置对应的连接方式。但这里运营商应该不会把信息给出，所以需要我们自己通过抓包的方式了解授权的实现方式。

因为我使用的是openwrt，登陆路由器之后可以使用tcpdump命令直接抓取对应端口的数据包，所以不需要单独给路由器划分vlan或者配置端口镜像等复杂的操作，直接把光猫出的iptv线和机顶盒的网线连到两个lan口（路由器默认的wan不接线，当交换机使用）这个时候相当于iptv机顶盒直接和光猫iptv口连接。这里只需要我们记住机顶盒连接的是哪个网口。这里我以连到下图的eth2举例。(电脑实际连的eth3，忘了画出来)

[https://i0.hdslb.com/bfs/article/watermark/cad2fbdf0a06ca395c9744df45e7c53b98bdd9e3.jpg@1256w_942h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/watermark/cad2fbdf0a06ca395c9744df45e7c53b98bdd9e3.jpg@1256w_942h_!web-article-pic.avif)

线路连接好以后，可以先确一下机顶盒是否可以正常登录看电视，如果登陆没问题，则先断电备用。

1. 电脑通过ssh登陆openwrt路由器(电脑实际连的eth3，图里忘了画出来)
    
    执行命令开始抓包: tcpdump -i eth2 -w dhcp.pcap
    
2. 给机顶盒上电启动，让机顶盒完成登陆进入主界面
3. 机顶盒登陆成功后，切到电脑ssh，执行ctrl+c，停止抓包，数据包会被保存在dhcp.pcap文件中
4. 抓包完成把数据包拷贝到电脑里备用。
5. 在电脑里用wireshark软件打开数据包，搜索protocol=dhcp的请求，可以看到两条路iptv机顶盒发出的请求，点开DHCP request请求，下方包的详细信息里dhcp协议中有
    
    option 61（客户端mac地址）展开后，在Client Mac Address点右键复制值，保存备用
    
    option 12（HostName）展开后，在Host Name点右键复制值，保存备用
    
    option 60  (Vendor class)展开后，在Vendor class identifier这行上，鼠标右键选择复制----as a Hex Stream 保存备用
    

[https://i0.hdslb.com/bfs/article/9a93f46564db58296f55bb6602fc3af04e8358b7.png@1256w_980h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/9a93f46564db58296f55bb6602fc3af04e8358b7.png@1256w_980h_!web-article-pic.avif)

网络鉴权信息抓取完毕，同时在抓包数据里搜索http协议，电信的http协议中会明显的看到一些初始化鉴权和获得播放列表的请求，比如下面这个请求，看名字就知道获取频道列表～

http://***/EPG/jsp/getchannellistHWCTC.jsp

打开response返回数据一看，好家伙～！保存备用

# **方案实现—网络配置**

至此抓包获取信息工作完成，把网络接线恢复到最终状态

[https://i0.hdslb.com/bfs/article/watermark/aade72db7d8112f7479ecde696d5e5676601a55f.jpg@1256w_942h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/watermark/aade72db7d8112f7479ecde696d5e5676601a55f.jpg@1256w_942h_!web-article-pic.avif)

回到Openwrt配置里选择新建接口，协议选择DHCP客户端，接口选择连接光猫的接口名。我这里以eth2为例，后选择保存

在基础配置里，添加请求DHCP时发送的主机名（上面抓包数据中的HostName）

[https://i0.hdslb.com/bfs/article/watermark/57cb378201afbcf69273504759aa368e0071739b.jpg@1256w_298h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/watermark/57cb378201afbcf69273504759aa368e0071739b.jpg@1256w_298h_!web-article-pic.avif)

在高级配置里，使用网关跃点填入一个比较大的数字，我这里写20

重设MAC地址这里，填写上面抓包数据的客户端mac地址

[https://i0.hdslb.com/bfs/article/watermark/066bbb0b80ff44a510a8a79606c7cdbb3da01232.jpg@1256w_1014h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/watermark/066bbb0b80ff44a510a8a79606c7cdbb3da01232.jpg@1256w_1014h_!web-article-pic.avif)

保存并应用设置后，再进入你的 WAN 接口设置，将它的 网络跃点设置为 1（比前面配的20小即可），**否则你会无法正常使用互联网**。

最后再ssh登陆到Openwrt路由器上，修改/etc/config/network中的IPTV参数

找到配置文件中iptv接口的dhcp配置（搜dhcp即可）在配置中新增一行：

option sendopts '0x3c:****'

- ***的内容为上面抓包中获得的Vendor class identifier(一长串字符串)，保存退出后回到openwrt，重连iptv接口，稍等一会如果看到有被分配了私网ip地址就说明配置成功了!

# 

# **方案实现—udpxy配置**

可以选择登陆到Openwrt路由器上命令行安装，安装命令:

opkg install udpxy luci-app-udpxy

或者直接在管理网页安装udpxy、luci-app-udpxy

安装好后进入udpxy管理页面配置:

Bind IP/Interface，可以直接配置本地lan的地址或者lan的接口名(br-lan)

端口只要不被占用即可，随意写

Source IP/Interface，填写电信iptv连接openwrt的接口，我这里是et2

最大连接数是你允许同时对多少个客户端进行代理（同时几个设备看iptv电视）

[https://i0.hdslb.com/bfs/article/5a5e184f6c0ff56a27c5d7c32ee26bebba8cc05d.png@1256w_1580h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/5a5e184f6c0ff56a27c5d7c32ee26bebba8cc05d.png@1256w_1580h_!web-article-pic.avif)

# **测试效果**

检查udpxy状态，本地访问:

http://[br-lan的ip]:[绑定端口]/status

我以路由器地址为192.168.1.1 绑定端口为9999为例

http://192.168.1.1:9999/status

网页会显示udpxy当前的连接状态，如果有显示状态说明正常(我截图的时候有人正在看电视，所以会显示active client为1，且有具体内容)

[https://i0.hdslb.com/bfs/article/19418f9a523ce15e9be0e0f0d345aa61967ccb83.png@1256w_810h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/19418f9a523ce15e9be0e0f0d345aa61967ccb83.png@1256w_810h_!web-article-pic.avif)

接下来我们翻出前面保存的抓包文件，找到这条请求的返回数据(我这个数据我这里只针对杭州电信):http://***/EPG/jsp/getchannellistHWCTC.jsp

这个请求抓到的数据包可以明确的看到电台列表，这里只举一个例子,搜索igmp可以看到这样的数据:

ChannelURL="igmp://233.50.201.118:5140|rtsp://115.233.41.137/PLTV/88888913/224/3221228078/10000100000000060000000002460690_0.smil******"

这里的igmp就是电视台的组播地址，我们通过这个ip端口号就能看iptv

这里需要调整下url的格式

A.抓包里的  igmp://233.50.201.118:5140

B.你路由器的udpxy访问路径为  http://192.168.1.1:9999/status

(以路由器地址为192.168.1.1 绑定端口为9999为例)

A+B组合后即为软件播放iptv使用的地址:

http://192.168.1.1:9999/udp/233.50.201.118:5140

http://address:port/udp/mcast_addr:mport/

在电脑使用VLC软件、添加链接即可进行播放。来跟我一起丢掉机顶盒吧~

[https://i0.hdslb.com/bfs/article/a7cfbc1a423e5f70e19ec183a01bae0a0d4150aa.png@1256w_866h_!web-article-pic.avif](https://i0.hdslb.com/bfs/article/a7cfbc1a423e5f70e19ec183a01bae0a0d4150aa.png@1256w_866h_!web-article-pic.avif)

**关于频道、台标、和电视节目单的配置问题，后续还计划再出一篇专栏讲一下，感兴趣的观众老爷可以点个关注收藏支持一下，谢谢大家！**

用了两天发现一个问题，udpxy在iptv接口重连以后，ip不会自动更着刷新，导致播放失败。

需要在hotplug创建一个脚本让udpxy跟着重新启动可以解决。

输入以下命令创建脚本

vi /etc/hotplug.d/iface/99-iptv.sh

贴入脚本内容

```jsx
#!/bin/sh
[ "$ACTION" = ifup ] && [ "$INTERFACE" = IPTV ] &&
/etc/init.d/udpxy restart
```

保存退出后为脚本文件添加可执行权限：

chmod +x /etc/hotplug.d/iface/99-iptv.sh