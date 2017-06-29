使用UDP传送以太网数据包扩展园区网
张焕杰，陈蕾
(中国科学技术大学网络信息中心，合肥 230026)
摘  要：本文将以太网数据包经过压缩、加密后利用UDP传送，实现了一种直接在以太网层将园区网通过Internet网络扩展的L2VPN功能。使用raw socket机制接收和发送以太网数据包，避免了其他实现方式的网桥处理步骤；使用PING/PONG机制在主备2个UDP连接间实现自动故障发现和主备自动切换，避免使用传统的生成树机制，简化了配置并能提高系统运行稳定性。使用高速的LZ4压缩算法，使得本文的实现可以在高带宽时尽量少影响性能。实现中还支持一方处于NAT环境，支持VLAN号的重新映射以避免VLAN冲突，适应更加灵活的应用场景。
关键词：Ethernet over UDP；L2VPN；RAW SOCKET；VLAN MAP
中图分类号：TP393.07                     文献标识码：A                     文章编号：
Extend campus network using Ethernet over UDP
ZHANG Huan-jie, CHEN Lei
(Network Information Center, University of Science and Technology of China, Hefei 230026, China)
Abstract: Ethernet over UDP (EthUDP) can extend campus layer 2 network using UDP to transfer compressed and en-crypted Ethernet packet. Using Linux raw socket to receive and send Ethernet packet and avoid bridge processing. PING/PONG method is used to do failure check and automatic switch over when there are two UDP connections. LZ4 high speed compress algorithm is used to do packet compress for high speed network. EthUDP also support NATed connection and vlan mapping to reduce vlan conflict.
Key words: Ethernet over UDP；L2VPN；RAW SOCKET；VLAN MAP

 
1  引言
通过Internet网络扩展园区网最简单的做法是建立L2VPN，即通过VPN隧道将以太网数据包经过加密后利用Internet网络传递，从而完成双方的通信。
常见的L2VPN或能实现类似L2隧道功能的程序对比如下表1所示：
表1 L2隧道功能实现方式对比
方式	NAT支持	加密	压缩
VxLAN[1]	X	X	X
VTun[2]	√	√	LZO/zlib
Tinc[3]	√	√	LZO/zlib
OpenVPN[4]	√	√	LZO
VxLAN主要用于数据中心内部解决VLAN扩展性问题，OpenVPN用于远程用户的连接，不适合用来建立隧道扩展网络。
 
图1 以太网透传隧道原理
如上图1所示，VTun和Tinc可以利用Internet网络建立隧道，在tap接口间转发数据包。将以太网接口和tap接口加入同一个网桥，由Linux的网桥处理逻辑实现网卡与tap接口间数据包的转发，从而可以完整实现以太网透传隧道功能。如果使用两个UDP连接（如不同的ISP线路）实现线路备用，需要配置两条隧道，并启用生成树协议以避免环路。由于公网的连通性不能实时反映到tap虚拟接口up/down状态，公网连接不稳定时会无法避免形成短时的环路，引起网络广播风暴，这对以太网是致命的。并且在生成树状态切换时会有若干秒网络处于中断状态。
本文实现的Ethernet Over UDP[5]（简称EthUDP，代码在https://github.com/bg6cq/ethudp），支持使用IPv4和IPv6建立隧道，支持一方使用动态地址或NAT环境后的连接，并可以验证对方IP地址。
2  RAW SOCKET接收和发送数据包
在Linux中，使用raw socket[6]可以跳过TCP/IP处理逻辑，直接在以太网接口接收和发送数据包。
使用raw socket机制从网卡接收以太网数据包并利用UDP发送给远端，同时从UDP接收远端发来的数据包并通过网卡直接发出，可以直接实现桥接功能，避免了VTun、Tinc两种实现方式中的网桥处理步骤。
从特定以太网接口接收数据包时，需要建立PF_PACKET类型的raw socket，将接口设置为混杂模式，并绑定到该接口。将来使用recvmsg调用即可接收数据包。接收到的数据包不含通过raw socket发送的数据包，正好避免了环路重复包。
int32_t ifindex, fdraw,val=1;
struct ifreq ifr;
struct sockaddr_ll sll;
fdraw = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
strncpy(ifr.ifr_name, ifname, IFNAMSIZ);
ioctl(fdraw, SIOCGIFINDEX, &ifr);
ifindex = ifr.ifr_ifindex;
ioctl(fdraw, SIOCGIFFLAGS, &ifr);
ifr.ifr_flags |= IFF_PROMISC;
ioctl(fdraw, SIOCSIFFLAGS, &ifr);
sll.sll_family = AF_PACKET;
sll.sll_protocol = htons(ETH_P_ALL);
sll.sll_ifindex = ifindex;
bind(fdraw, (struct sockaddr *)&sll, si-zeof(sll);
向特定接口发送数据包时，只要提供接口索引调用sendto既可将完整的以太网数据包发送出去：
struct sockaddr_ll sll;
sll.sll_family = AF_PACKET;
sll.sll_protocol = htons(ETH_P_ALL);
sll.sll_ifindex = ifindex;
sendto(fdraw, buf, len, 0, (struct sockaddr *)&sll, sizeof(sll));
3  PING/PONG机制
EthUDP每隔1秒钟利用UDP向对方发送PING消息，每一方收到PING消息后会立即应答PONG消息。
主备两个UDP连接的情况下，如果一方在连续的5秒钟内未从主连接收到对方的PONG消息，会认为主连接中断，立即将备用连接设定为当前在用连接，将来需要转发的以太网数据包将通过备用连接发送。一旦再次从主连接接收到对方的PONG消息，立即切换回主连接。这个简单的机制可以在主备2个UDP连接间实现自动故障发现和主备自动切换。 
发送以太网数据包时，发送方只会通过主备UDP连接中的一个发送，不需要使用生成树协议也能从根本上确保不会发生环路情况，彻底避免了广播风暴问题，能大大提高系统运行的稳定性。
3  LZ4压缩/AES加密
为了适应高速网络，EthUDP可选使用LZ4[7]高速算法对发送数据包进行压缩，在高带宽环境下尽量少影响性能。
启用压缩后，EthUDP在每个数据包后增加一个字节作为是否压缩的标志。当原始数据包使用LZ4压缩无效（压缩后的长度没有减少）时，该标志为0xaa；有效压缩时为0xff。通过这样的机制，一个数据包即便无法有效压缩，也仅仅多传输1个字节。
EthUDP可选使用openssl[8]提供的AES CBC算法对发送数据包进行简单加密。加密处理中使用openssl标准填充方式填充，每个数据包加密后可能会增加1-16字节。
4  VLAN映射
将两个局域网相连时，经常会碰到vlan编号冲突问题。通常的解决办法是切换到某一方不使用的vlan，以解决冲突问题。但这样对于双方已经在用的vlan比较麻烦。
EthUDP增加了可选的vlan编号映射步骤，在发送前将本地的一个vlan编号转换成另一个，接收时再转换回来，方便解决vlan冲突，适应更加灵活的应用场景。
5  EthUDP进程设计与实现
EthUDP是运行在用户态空间的多线程进程，每个线程处理一个方向的数据包转发。进程启动初始化时会建立相关的UDP连接（如果指明的UDP端口为0，则为NAT模式，暂时不建立连接），打开指定网卡的raw socket，然后启动以下3种线程：
5.1 以太网接口数据包接收转发线程
	线程处于如下循环：
从以太网接口接收数据包；
如果有MSS调整，对tcp syn包修改MSS；
如果有VLAN映射，将以太网包头的VLAN编号改为对方VLAN编号；
如果需要，进行压缩和加密处理；
通过在用的UDP连接发送给对方。
5.2 UDP数据包接收转发线程
	每个UDP连接对应一个接收线程，因此主备两个连接时启动2个线程。线程处于如下循环：
从UDP端口接收数据包；
如果需要，进行解密和解压缩处理；
如果是NAT环境，判断是否需要更新对方地址和端口号信息；
如果是PING消息，发送PONG消息应答；
如果是PONG消息，更新最后接收时间；
如果有VLAN映射，将以太网包头的VLAN编号改为我方VLAN编号；
如果有MSS调整，对tcp syn包修改MSS；
通过raw socket发送数据包。
5.3 ping/pong消息处理线程
线程处于如下循环：
向UDP连接发送PING消息；
如果主连接最后接收PONG消息是5秒前，设定备用连接为在用UDP连接；
每间隔1小时，输出收发数据包/字节/压缩效率等统计信息；
等待1秒。
EthUDP进程的每个线程仅仅处理一个方向的数据包转发，这样的设计可以充分发挥多核CPU的作用。正常运行时，上述5.1和5.2线程会占用较多的CPU时间，因此建议运行EthUDP的机器或虚拟机至少能允许2个线程同时执行。
6  效率评估和性能对比测试
隧道传输时，原有的数据包外增加一组UDP、IP、以太网数据包头，会导致传输效率的降低。
对于1G以太网链路，不启用TIME_STAMP选项的TCP连接，以128字节的有效TCP载荷为例，在以太网传输时需要添加TCP头、IP头、以太网头、帧间隔、帧开始、尾部CRC分别20、20、14、12、8、4字节（合计78字节）后为206字节。最大有效传输速率是1000*128/206=621Mbps。
使用不加密不压缩的UDP隧道时，增加UDP头、IP头、以太网头分别8、20、14字节（合计42字节）后，在以太网传输合计206+42=248字节，最大有效传输速率是1000*128/248=516Mbps。当使用AES-128-CBC或AES-256-CBC算法加密时，有效载荷、TCP头、IP头、以太网头分别是128、20、20、14字节合计182字节需要加密，加密填充后变为192字节，增加了10字节，最终在以太网传输时合计为258字节，最大有效传输速率是1000*128/258=496Mbps。
 
图2 效率评估测试环境
为了测试有效传输速率，按照图2搭建了效率评估测试环境。4台虚拟机（VMware虚拟机VPN1和VPN2的网卡eth1连接的网络需修改安全选项允许修改源地址）通过1G线路互连。每台虚拟机有4个Intel E5-2640 v2 @ 2.00GHz CPU核心，安装CentOS 6.9系统，测试用的iperf版本为2.0.5。VPN1和VPN2上运行EthUDP程序，分别使用不加密、AES-128-CBC加密和AES-256-CBC加密方式，在192.168.1.1上执行iperf –s作为服务器端，在192.168.1.2上执行iperf -c 192.168.1.1 -t 10 -m -M MSS（其中MSS依次使用128、256、512、1024等值）测试有效传输速率。每个MSS值测试5次取平均值，测试的结果如下图3所示（其中测试速率是直接在VPN1和VPN2之间直接通信的速率）。
从中可以得出结论：不做任何隧道时MSS 256字节逐步接近理论最大速率；不加密的隧道时MSS 512字节逐步接近理论最大值，而无论是AES-128-CBC或AES-256-CBC加密时MSS 1024字节才逐步接近理论最大速率。根据我校校园网统计，网上的数据包平均大小入方向为900字节左右，出方向为750字节左右。因此在实际使用中要注意，除非继续提高CPU单核性能，否则一旦启用加密可能无法充分使用1G信道带宽。
 
图3 效率评估测试结果
为了与Vtun、Tinc等可以实现类功能的程序对比测试，按照图4搭建了测试环境。其中VPN1位于合肥，VPN2位于上海，两者之间通过1G的以太网线路互连，ping延迟10ms。
 
图4 对比测试环境
4台虚拟机通过1G线路互连，每台虚拟机有4个Intel E5-2640 v2 @ 2.00GHz CPU核心，安装CentOS 6.9系统。其中192.168.1.1和192.168.1.2的内核升级到4.11.6并启用TCP BBR拥塞控制算法。在VPN1和VPN2之间分别运行EthUDP、VTun和Tinc不压缩不加密、仅启用压缩、同时启用压缩和加密三种模式，使用iperf测试从192.168.1.2向192.168.1.1的实际传输速率。
测试时共使用四种数据负载：第一种是iperf默认的数据负载（字符序列0-9的重复）；第二种是长度为408MB的CentOS 6.9 x86_64最小系统安装光盘镜像文件centos.iso；第三种是下载news.sina. com.cn和news.163.com页面镜像文件复制若干份后用tar打包成417MB的icp.tar文件；第四种是从https://cs.fit.edu/~mmahoney/compression/textdata.html下载enwik8.zip解压获取文件复制4份后用tar打包成382MB的enwik8.tar文件。
Vtun在启用加密功能时会出现异常中断无法完成测试过程，因此仅仅测试了不压缩不加密和仅压缩两种模式。传输速率测试结果如图5所示。测试结果表明EthUDP性能最优，Tinc性能最差。即便启用压缩和加密功能，EthUDP也可以达到700Mbps以上的带宽。而只要启用压缩功能，Vtun和Tinc的性能急剧下降到100Mbps附近。
 
图5 传输速率测试结果
每次测试时记录VPN2上eth0和eth1网卡的发送和接收字节数，计算隧道传输效率（eth1接收字节/eth0发送字节，越大传输效率越高）。各种测试的传输效率对比结果如图6所示。
 
图6 隧道接口传输效率测试结果
测试结果表明大数据包传输时，对于iperf默认负载的字符0-9序列内容，一旦启用压缩均可以实现10倍的传输效率；对于已经充分压缩过的内容，传输效率在0.95-0.99之间；对于测试的文本内容，Vtun和Tinc可以实现1.40的传输效率，而EthUDP仅可以实现1.20的传输效率（但传输速率并未显著降低）。
7  应用环境和效果
EthUDP已经应用在如下环境：
7.1 跨国互连的未来网络试验用透明以太网隧道
    2013年9月，中国科学技术大学未来网络实验室使用EthUDP利用Internet通道建立透明以太网隧道与美国西北大学的iGENI网络互连，通过透明隧道开展广域范围的内容中心网络（content centric networking）实验。
7.2 安徽省教育和科研计算机主干网备份线路
安徽省教育和科研计算机主干网中心设在合肥，连接安徽省的16个城市。从合肥至各个城市的设备间租用了运营商的1G以太网专线作为主用线路。线路支持IPv4/IPv6双栈协议数据包的转发，运行有OSPF/OSPFv3/BGP等动态路由协议。
从合肥到各个城市间使用EthUDP利用另一个运营商的Internet通道建立的透明以太网隧道作为备用线路，可以支持主用线路上所使用的所有协议。
从2014年5月建成至今，备用线路工作正常，网络主干未出现过单线路中断引起的断网情况，大大提高了网络可用性。
7.3 中国科学技术大学校园网延伸至中科大上海研究院
中国科学技术大学主要校区位于安徽省合肥市，同时位于上海市浦东新区的中科大上海研究院有使用校园网的需求。
2017年6月，使用EthUDP利用1G线路和电信100M Internet线路作为主备连接建立透明以太网隧道，将位于合肥市主校区的若干VLAN直接传送到位于上海浦东的研究院。位于上海浦东的用户可以直接使用主校区VLAN IP地址上网；无线AP可以直接连接主校区的AC，从而提供与主校区相同的无线网络环境，极大的方便了网络管理和用户使用。
8  结论
EthUDP使用raw socket机制接收和发送以太网数据包，将以太网数据包经过压缩、加密后利用UDP传送，实现了一种直接在以太网层将园区网通过Internet或其他网络扩展的L2VPN功能。
通过使用多个线程处理不同方向的数据包，并使用高速的LZ4压缩算法，EthUDP相比能提供类似功能的Vtun和Tinc运行更快，在高带宽时对性能影响少。使用PING/PONG机制在主备2个UDP连接间实现自动故障发现和主备自动切换，避免使用传统的生成树机制，简化了配置并能提高系统运行稳定性。EthUDP可以支持一方处于NAT环境，并且支持VLAN编号的重新映射以避免VLAN冲突，适应更加灵活的应用场景。
参考文献：
[1] 	Virtual eXtensible Local Area Network (VXLAN): A Framework    for Overlaying Virtualized Layer 2 Networks over Layer 3 Networks [EB/OL]. https://tools.ietf.org/html/rfc7348
[2] 	VTun - Virtual Tunnels over TCP/IP networks[EB/OL]. http://vtun.sourceforge.net/
[3] 	Tinc VPN[EB/OL]. https://www.tinc-vpn.org/
[4]	OpenVPN - Open Source VPN[EB/OL]. https://openvpn.net/
[5]	Ethernet Over UDP [EB/OL]. http://github.com/bg6cq/ethudp
[6]	A Guide to Using Raw Sockets[EB/OL]. http://opensourceforu.com/2015/03/a-guide-to-using-raw-sockets/
[7] 	LZ4 - Extremely fast compression[EB/OL]. http://lz4.github.io/lz4/
[8] 	openssl[EB/OL]. https://www.openssl.org/
