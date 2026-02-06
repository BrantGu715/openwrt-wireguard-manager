# WireGuard System（OpenWrt）

记录在 OpenWRT 上部署 WireGuard VPN。  
同时新建一些基于 OpenWrt 的 WireGuard 命令行管理脚本，用于在路由器 / 软路由环境中**快速、安全地管理 WireGuard 服务、客户端和端口配置。**

---

## 项目目标

- 不替代 `wg` / `wg-quick` 原生命令
- 不依赖 Web UI（LuCI）
- 提供一套**稳定、可组合、脚本友好的运维工具**

所有操作**推荐通过 `wgm` 统一入口执行**，  
底层 `wg-*` 脚本作为实现细节存在。

---

## 网络架构与设计

### 物理拓扑结构

```text
ISP
 │
 ├── 动态公网 IP（通过 OpenWrt DDNS 绑定域名）
 │
光猫（桥接模式）
 │
TP-LINK（硬路由，主路由）
 │ 网关：192.168.0.1
 │ 功能：PPPoE 拨号、DHCP、NAT、端口转发
 │
 ├── OpenWRT GDQ Version V1 [2025]（软路由，旁路由）
 │   IP：192.168.0.101
 │   功能：WireGuard 服务器、内网服务
 │   WireGuard 接口：wg0（10.7.0.1/24）
 │
 ├── 电脑（被备份设备）
 │   IP：192.168.0.102
 │
 └── NAS（备份目标）
     IP：192.168.0.103
```
	 
### 逻辑拓扑（VPN 建立后）

```text
客户端（远程，10.7.0.2） ←── WireGuard隧道 ──→ OpenWRT (wg0:10.7.0.1)
                                      │
                                      ├── 硬路由 (192.168.0.1)
                                      │
                                      ├── 电脑 (192.168.0.102)
                                      │
                                      └── NAS (192.168.0.103)
```									  
									  
### OpenWrt 核心配置
主要任务是配置这两个文件：
```sh
/etc/config/firewall
/etc/config/network
```

#### 配置端口转发状态：
```sh
/etc/config/firewall	
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
    option dest_port '12345'
    option target 'ACCEPT'
```

#### 配置 wg0 接口、私钥和监听端口：	
```sh
/etc/config/network		
config interface 'wg0'
    option proto 'wireguard'
    option private_key 'SSS'
    list addresses '10.7.0.1/24'
    option listen_port '12345'
	
# 客户端（Peer）配置：
config wireguard_wg0
    option public_key 'BBB'
    list allowed_ips '10.7.0.2/32'

config wireguard_wg0
    option public_key 'CCC'
    list allowed_ips '10.7.0.3/32'
```	


每一个 config wireguard_wg0 即一个客户端（Peer）

添加 client 只能通过修改 network 文件完成


### 脚本目录结构
```test
/usr/local/bin/
├── wgm                     # 总入口（推荐使用）
│
├── wg-service-start        # 启动 WireGuard 服务
├── wg-service-stop         # 停止 WireGuard 服务
├── wg-service-restart      # 重启 WireGuard 服务
├── wg-service-status       # 查看 WireGuard 服务状态
│
├── wg-client-add           # 添加客户端
├── wg-client-remove        # 删除客户端
├── wg-client-enable        # 启用客户端
├── wg-client-disable       # 禁用客户端
├── wg-client-qrcode        # 显示客户端配置二维码
│
├── wg-port-change          # 修改 WireGuard 监听端口
├── wg-port-check           # 查看当前端口状态
│
├── wg-system-install       # 安装 / 检查依赖
├── wg-system-setup         # 系统初始化
├── wg-system-backup        # 配置备份
├── wg-system-monitor       # 系统监控
├── wg-system-doctor        # 故障诊断
├── wg-system-audit         # 安全审计
├── wg-system-guide         # 使用指南
```


推荐用法（统一入口）

wgm [subcommand] [args]

wgm 是唯一推荐入口

所有功能均可通过 wgm 访问

wg-* 脚本仅用于调试或内部调用

wgm 命令总览

服务管理
```sh
wgm service status
wgm service start
wgm service stop
wgm service restart
```

适用场景：

服务启动失败排查

修改配置后的重载

确认 WireGuard 运行状态

客户端管理
```sh
wgm client add <name>
wgm client remove <name>
wgm client enable <name>
wgm client disable <name>
wgm client status
wgm client qrcode <name>
```

说明：
disable 不删除密钥，仅阻断访问
enable 用于恢复客户端
适合临时封禁、账号冻结场景

端口管理
```sh
wgm port change
wgm port check
```

修改端口后通常需要重启服务。

系统维护
```sh
wgm system install
wgm system setup
wgm system backup
wgm system monitor
wgm system doctor
wgm system audit
wgm system guide
```

设计原则

所有脚本只做一件事

不隐藏配置文件位置

所有修改可审计、可回滚

适用环境

OpenWrt（实体路由 / x86 软路由）

BusyBox / ash

WireGuard 官方内核模块或 wireguard-tools

免责声明

本项目不会修改 OpenWrt 默认安全策略。

所有端口开放与访问控制策略需使用者自行评估风险。
