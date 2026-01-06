# RIP-
use trace to see the ip routing
use show cdp neighbour details 
use Tracert in cmd 

1. 你的现状 (The Context)
根据你的配置：
你有去往互联网的路： 你配置了静态默认路由 ip route 0.0.0.0 0.0.0.0 FastEthernet0/0。这意味着你的 GW 路由器知道：“如果要访问未知的 IP（比如 Google 或百度），就往 FastEthernet0/0 (WAN口) 扔”。

你在运行 RIP： 你在 192.168.103.0 网段运行 RIP 协议，这意味着你可能在和内网的其他路由器或三层交换机交换路由信息。

2. 存在的问题 (The Problem)
虽然 GW 知道怎么去互联网，但是 RIP 协议默认情况下是“自私”的。

RIP 默认只通告它直连的网段（比如 192.168.103.0）。

RIP 不会自动把你配置的那条静态默认路由（0.0.0.0/0）告诉给邻居路由器。

结果： 内网的其他路由器只知道怎么去 192.168.103.0，但不知道怎么去互联网。

3. 解决方案 (The Solution)
命令 default-information originate 的字面意思是“产生默认信息”。

当你输入这条命令后：

注入路由： GW 的 RIP 进程会生成一条 0.0.0.0/0 的默认路由更新。

广播/组播： GW 会通过 FastEthernet5/0 把这条信息告诉内网的其他 RIP 路由器。

最终效果： 内网的其他路由器会学到：“哦，原来 GW (192.168.103.253) 是通往世界的出口。” 它们会自动把“最后求助网关”指向你的 GW。

总结对比
场景	       GW 路由表          	下级邻居路由器的路由表	结果
没敲这条命令	有默认路由 (能上网)	没有默认路由	          邻居无法上网，只有 GW 能上。
敲了这条命令	有默认路由 (能上网)	学到了 RIP             默认路由 (R* 0.0.0.0/0)	整个网络都能通过 GW 上网。

1. 设备 GW (网关) 配置注解
Bash

GW#sh ru                             # 显示当前运行配置 (Show Running-config)
Building configuration...            # 系统提示：正在构建配置信息...

Current configuration : 804 bytes    # 当前配置文件大小
!
version 12.2                         # IOS 操作系统版本 12.2
no service timestamps log datetime msec   # 日志中不显示毫秒级时间戳 (通常建议开启，此处关闭)
no service timestamps debug datetime msec # 调试信息中不显示毫秒级时间戳
service password-encryption          #启用密码加密服务 (防止密码以明文显示在配置中)
!
hostname GW                          # 设置主机名为 GW
!
!
!
enable password 7 08294D420516       # 设置特权模式密码 (数字7表示已加密，原文是明文)
!
!
!
!
!
!
ip cef                               # 开启 CEF (Cisco Express Forwarding)，硬件级快速转发
no ipv6 cef                          # 关闭 IPv6 的 CEF (本网络不使用 IPv6)
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface FastEthernet0/0            # 进入快速以太网接口 0/0 (WAN口)
 ip address dhcp                     # IP地址通过 DHCP 自动获取 (通常从 ISP 运营商处获取)
 ip nat outside                      # 标记此接口为 NAT 的“外部”接口 (连接互联网的一侧)
!
interface FastEthernet5/0            # 进入快速以太网接口 5/0 (LAN口，连接 RTR2)
 ip address 192.168.103.253 255.255.255.0  # 配置静态 IP 地址和掩码
 ip nat inside                       # 标记此接口为 NAT 的“内部”接口 (连接局域网的一侧)
!
router rip                           # 启用 RIP 动态路由协议
 version 2                           # 使用版本 2 (支持子网掩码/VLSM)
 network 192.168.103.0               # 宣告网段：参与路由的接口网段
 default-information originate       # ★关键指令：向 RIP 邻居(RTR2)下发默认路由，告诉大家"跟我走能上网"
 no auto-summary                     # 关闭自动汇总 (发送精确路由，不汇总成 A/B/C 类)
!
ip nat inside source list 1 interface FastEthernet0/0 overload  # NAT 核心指令：将符合 ACL 1 的内网流量转换为 Fa0/0 的公网 IP (端口复用模式 PAT)
ip classless                         # 启用无类路由 (允许转发目的地址不在路由表中的数据包给默认路由)
ip route 0.0.0.0 0.0.0.0 FastEthernet0/0  # 静态默认路由：所有未知流量扔给 WAN 口 (指向互联网)
!
ip flow-export version 9             # 配置 NetFlow 流量监控数据的导出版本为 v9
!
!
access-list 1 permit any             # 访问控制列表 1：允许任何 IP (配合 NAT 使用，表示所有人都能上网)
!
!
!
!
!
!
line con 0                           # 配置 Console (控制台) 线路
 password 7 082243401A160912465F5C5D # 设置 Console 登录密码 (已加密)
 login                               # 启用登录验证 (必须输密码才能进)
!
line aux 0                           # 配置 AUX 辅助线路 (通常未配置)
!
line vty 0 4                         # 配置 VTY 虚拟终端线路 (0-4 共5个会话，用于 Telnet/SSH)
 password 7 08354942071C11464058     # 设置远程登录密码 (已加密)
 login                               # 启用登录验证
!
!
!
end                                  # 配置结束
2. 设备 RTR2 (内部路由器) 配置注解
Bash

RTR2#sh ru                           # 显示当前运行配置
Building configuration...            # 系统提示...

Current configuration : 994 bytes    # 配置文件大小
!
version 12.2                         # IOS 版本 12.2
no service timestamps log datetime msec   # 关闭日志时间戳
no service timestamps debug datetime msec # 关闭调试时间戳
service password-encryption          # 启用密码加密
!
hostname RTR2                        # 设置主机名为 RTR2
!
!
!
enable password 7 08294D420516       # 设置特权模式密码 (已加密)
!
!
!
!
!
!
ip cef                               # 开启 CEF 快速转发
no ipv6 cef                          # 关闭 IPv6 CEF
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface FastEthernet0/0            # 接口 Fa0/0 (可能是服务器网段)
 ip address 192.168.110.190 255.255.255.224 # 配置 IP，注意掩码是 /27 (可用30个主机)
 ip helper-address 192.168.110.70    # ★DHCP 中继：收到 DHCP 请求广播后，转发给 IP 为 192.168.110.70 的 DHCP 服务器
 duplex auto                         # 端口双工模式自动协商
 speed auto                          # 端口速率自动协商
!
interface FastEthernet4/0            # 接口 Fa4/0 (可能是部门 A 网段)
 ip address 192.168.102.254 255.255.255.0   # 配置网关 IP /24
!
interface FastEthernet5/0            # 接口 Fa5/0 (核心互联接口，连接 GW)
 ip address 192.168.103.254 255.255.255.0   # 配置 IP (与 GW 在同一网段)
!
interface FastEthernet8/0            # 接口 Fa8/0 (可能是部门 B 网段)
 ip address 192.168.101.253 255.255.255.0   # 配置网关 IP /24
!
interface FastEthernet9/0            # 接口 Fa9/0 (可能是部门 C 网段)
 ip address 192.168.100.253 255.255.255.0   # 配置网关 IP /24
!
router rip                           # 启用 RIP 路由协议
 version 2                           # 使用版本 2
 network 192.168.100.0               # 宣告 Fa9/0 所在网段
 network 192.168.101.0               # 宣告 Fa8/0 所在网段
 network 192.168.102.0               # 宣告 Fa4/0 所在网段
 network 192.168.103.0               # 宣告 Fa5/0 所在网段 (用于和 GW 建立邻居)
 network 192.168.110.0               # 宣告 Fa0/0 所在网段
 no auto-summary                     # 关闭自动汇总
 ! 注意：这里没有配置静态默认路由，RTR2 依靠从 GW 学来的路由上网
!
ip classless                         # 启用无类路由转发
!
ip flow-export version 9             # 配置 NetFlow 版本
!
!
!
!
!
!
!
!
line con 0                           # Console 线路配置
 password 7 082A434013160A1B         # 设置 Console 密码
 login                               # 启用登录
!
line aux 0                           # Aux 线路
!
line vty 0 4                         # VTY 远程登录线路
 password 7 08354942071C11464058     # 设置 VTY 密码
 login                               # 启用登录
!
!
!
end                                  # 配置结束
总结与观察
GW vs RTR2 的角色区别：

GW 上有 ip nat ... 和 ip route 0.0.0.0 ...，说明它是出口网关。

RTR2 没有任何 NAT 配置，也没有默认路由。它完全依赖 RIP 协议从 GW 那里学习“如何去互联网”。

DHCP 的处理：

GW 本身作为客户端 (ip address dhcp) 从运营商获取 IP。

RTR2 在 Fa0/0 上配置了 ip helper-address，这说明它不是 DHCP 服务器，而是中继代理，它会把局域网内的 DHCP 请求转发给内网的一台专用服务器 (192.168.110.70)。
