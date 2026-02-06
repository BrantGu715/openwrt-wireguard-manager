WireGuard System （OpenWrt）
记录在OpenWRT上部署WireGuard VPN
同时新建一些基于 OpenWrt 的 WireGuard 命令行管理脚本，用于在路由器/软路由环境中 快速、安全的管理 WireGuard 服务、客户端和端口配置。
本项目的设计目标是：
不替代 wg / wg-quick 原生命令
不依赖 Web UI（LuCI）
提供一套 稳定、可组合、脚本友好的运维工具
所有操作推荐通过 wgm 统一入口执行，底层 wg-* 脚本作为实现细节存在。

网络架构与设计

物理拓扑结构
ISP
  │
  ├── 动态公网IP（通过openwrt DDNS绑定域名）
  │
光猫（桥接模式）
  │
TP-LINK （硬路由，主路由）
  │ 网关：192.168.0.1
  │ 功能：PPPoE拨号、DHCP、NAT、端口转发
  │
  ├── OpenWRT GDQ Version V1[2025]（软路由，旁路由）
  │   IP：192.168.0.101
  │   功能：WireGuard服务器、内网服务
  │   WireGuard接口：wg0 (10.7.0.1/24)
  │
  ├── 电脑（被备份设备）
  │   IP：192.168.0.102
  │
  └── NAS（备份目标）
      IP：192.168.0.103

逻辑拓扑（VPN建立后）
客户端（远程，10.7.0.2） ←── WireGuard隧道 ──→ OpenWRT (wg0:10.7.0.1)
                                      │
                                      ├── 硬路由 (192.168.0.1)
                                      │
                                      ├── 电脑 (192.168.0.102)
                                      │
                                      └── NAS (192.168.0.103)

主要任务是配置两个文件：firewall和network

/etc/config/firewall 配置端口转发状态
config zone
option name 'wg'
list network 'wg0'
option input 'ACCEPT'
option output 'ACCEPT'
option forward 'ACCEPT'

config forwarding
option src 'lan'
option dest 'wan'

config forwarding
option src 'wg'
option dest 'lan'

config rule
option name 'Allow-WireGuard'
option src 'wan'
option proto 'udp'
option dest_port '16144'
option target 'ACCEPT'

/etc/config/network 配置一个wg0接口的网络端口,配置私钥和监听端口

config interface 'wg0'
option proto 'wireguard'
option private_key 'SSS'
list addresses '10.7.0.1/24'
option listen_port '12345'

config wireguard_wg0
option public_key 'BBB'
list allowed_ips '10.7.0.2/32'

config wireguard_wg0
option public_key 'CCC'
list allowed_ips '10.7.0.3/32'

（每一个’config wireguard_wg0’就是一个客户端（Peer））
想要添加client只能在network文件里面添加

脚本目录结构
/usr/local/bin/
├── wgm # 总入口（推荐使用）
│
├── wg-service-start # 启动 WireGuard 服务
├── wg-service-stop # 停止 WireGuard 服务
├── wg-service-restart # 重启 WireGuard 服务
├── wg-service-status # 查看 WireGuard 服务状态
│
├── wg-client-add # 添加客户端
├── wg-client-remove # 删除客户端
├── wg-client-enable # 启用客户端
├── wg-client-disable # 禁用客户端
├── wg-client-qrcode # 显示客户端配置二维码
│
├── wg-port-change # 修改 WireGuard 监听端口
├── wg-port-check # 查看当前端口状态
│
├── wg-system-install # 安装 / 检查依赖
├── wg-system-setup # 系统初始化（首次部署）
├── wg-system-backup # 备份 WireGuard 配置
├── wg-system-monitor # 系统运行状态检查
├── wg-system-doctor # 故障诊断与修复建议
├── wg-system-audit # 安全与配置审计
├── wg-system-guide # 使用指南


推荐用法（统一入口）
wgm <command> [subcommand] [args]
wgm 是唯一推荐入口
所有功能均可通过 wgm 访问

wg-* 脚本可以直接调用，但仅用于调试或脚本内部调用

wgm 命令总览
服务管理
wgm service status        # 查看 WireGuard 服务状态
wgm service start         # 启动 WireGuard
wgm service stop          # 停止 WireGuard
wgm service restart       # 重启 WireGuard

适用场景：
服务启动失败排查
修改配置后的重载

确认 WireGuard 是否正在运行
客户端管理
wgm client add <name>           # 添加客户端
wgm client remove <name>        # 删除客户端
wgm client enable <name>        # 启用客户端
wgm client disable <name>       # 禁用客户端
wgm client status               # 查看所有客户端状态
wgm client qrcode <name>        # 显示客户端二维码

说明：
disable 不删除密钥，仅阻断该客户端访问
enable 用于恢复已禁用客户端
适合临时封禁设备、账号冻结等场景

端口管理
wgm port change                 # 修改 WireGuard 监听端口
wgm port check                  # 查看端口与防火墙状态

说明：
修改端口后通常需要 service restart
可配合审计脚本确认端口暴露风险

系统维护
wgm system install              # 安装 / 检查依赖
wgm system setup                # 初始化 WireGuard 环境
wgm system backup               # 备份配置文件
wgm system monitor              # 系统运行状态检查
wgm system doctor               # 常见问题诊断
wgm system audit                # 安全与配置审计
wgm system guide                # 查看使用指南

典型使用流程
首次部署
wgm system install
wgm system setup
wgm service start
添加客户端
wgm client add phone
wgm client qrcode phone
临时封禁客户端
wgm client disable laptop
配置审计
wgm system audit
设计原则

所有脚本只做一件事
不隐藏配置文件位置
所有修改操作可审计、可回滚（配合 backup）


适用环境
OpenWrt（实体路由 / x86 软路由）
BusyBox / ash
WireGuard 官方内核模块或 wireguard-tools

免责声明
本项目不会修改 OpenWrt 默认安全策略。
所有端口开放、访问控制策略需使用者自行评估安全风险。