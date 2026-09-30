# nftctl 使用说明

## 一、功能

nftctl 用于管理 IPv4 TCP/UDP 端口转发规则。

支持：

- TCP + UDP 同时配置
- 相同端口转发
- 不同目标端口转发
- nftables map
- IPv4 forwarding
- conntrack
- Flowtable Fast Path
- 自动 Flowtable 兼容性检查
- BBR
- fq
- irqbalance
- 自动加载
- 状态检查
- 自检
- 网络诊断


规则格式：

```text
本机监听端口 -> 目标IP:目标端口
```


例如：

```text
443 -> 1.2.3.4:443
```

或者：

```text
10086 -> 1.2.3.4:443
```


---

# 二、安装

SSH 登录服务器。

如果当前用户不是 root：

```bash
sudo bash <<'EOF'
...完整安装脚本...
EOF
```

如果已经是 root：

```bash
bash <<'EOF'
...完整安装脚本...
EOF
```


安装完成后会自动：

- 安装依赖
- 开启 IPv4 forwarding
- 检测 BBR
- 配置 fq
- 调整 conntrack
- 启动 irqbalance
- 自动判断 Flowtable
- 加载现有规则
- 创建 nftctl.service


---

# 三、安装完成后检查

执行：

```bash
nftctl status
```

正常可能显示：

```text
nftctl

Rules:           2
Flow mode:       auto
Auto check:      clear
Fast path:       on
IPv4 forward:    1
Conntrack:       50 / 262144
Qdisc:           fq
TCP CC:          bbr
Interfaces:      eth0
```


重点关注：

```text
Auto check: clear
Fast path: on
IPv4 forward: 1
```


---

# 四、添加规则

交互式添加：

```bash
nftctl add
```

然后输入：

```text
Target IP:
Listen port:
Target port [Listen port]:
```


例如：

```text
Target IP: 154.12.34.56
Listen port: 443
Target port [443]:
```

目标端口直接回车，表示使用相同端口。

最终：

```text
443 -> 154.12.34.56:443
TCP + UDP
```


---

# 五、相同端口快速添加

例如：

```text
443 -> 154.12.34.56:443
```

直接：

```bash
nftctl add 154.12.34.56 443
```


等价于：

```text
Listen port: 443
Target port: 443
```


---

# 六、不同端口快速添加

例如：

```text
10086 -> 154.12.34.56:443
```

执行：

```bash
nftctl add 154.12.34.56 10086 443
```


参数顺序：

```text
nftctl add 目标IP 本机监听端口 目标端口
```


例如：

```bash
nftctl add 8.8.8.8 20000 53
```

表示：

```text
TCP 20000 -> 8.8.8.8:53
UDP 20000 -> 8.8.8.8:53
```


---

# 七、查看规则

执行：

```bash
nftctl list
```

例如：

```text
LISTEN       TARGET IP          TARGET PORT  PROTO
------       ---------------    -----------  -------
443          1.2.3.4            443          TCP+UDP
8443         2.3.4.5            443          TCP+UDP
10086        3.4.5.6            8443         TCP+UDP
```


---

# 八、修改已有规则

规则以本机监听端口作为唯一键。

例如原来：

```text
443 -> 1.1.1.1:443
```

现在执行：

```bash
nftctl add 2.2.2.2 443
```

最终变成：

```text
443 -> 2.2.2.2:443
```


不需要先删除。


也可以修改目标端口：

```bash
nftctl add 2.2.2.2 443 8443
```

最终：

```text
443 -> 2.2.2.2:8443
```


---

# 九、删除规则

例如删除本机监听端口 443：

```bash
nftctl del 443
```

也可以：

```bash
nftctl del
```

然后输入：

```text
Listen port: 443
```


例如规则是：

```text
10086 -> 1.2.3.4:443
```

删除时使用：

```bash
nftctl del 10086
```

不是：

```bash
nftctl del 443
```


---

# 十、重新加载

执行：

```bash
nftctl reload
```

正常可能显示：

```text
Reloaded. Fast path: on
```


如果自动兼容性检查认为当前系统不适合使用 Flowtable，则可能显示：

```text
Reloaded. Fast path: off
```


普通转发仍然正常工作。


---

# 十一、Flowtable 自动模式

默认配置文件：

```text
/etc/nftctl/config
```

默认：

```text
FLOWTABLE=auto
IFACES=""
```


推荐保持：

```text
FLOWTABLE=auto
```


自动模式会检查服务器现有的 nftables 配置。


以下情况不会被认为是冲突：

```text
普通 NAT prerouting
普通 NAT postrouting
空的 FORWARD 链
policy accept
```


例如：

```text
chain forward {
    type filter hook forward priority filter;
    policy accept;
}
```

如果链里没有实际规则，则允许 Fast Path。


普通：

```text
type nat
```

规则也允许存在。


---

# 十二、什么情况下自动关闭 Fast Path

如果 forwarding 数据路径中存在其他非 NAT 实际规则，例如：

```text
ingress
prerouting
forward
postrouting
```

中的 filter、mangle 等规则，自动模式会保守地关闭 Fast Path。


如果相关 base chain 使用：

```text
policy drop
```

或其他非 ACCEPT policy，也会关闭。


这样可以避免 Fast Path 绕过服务器已有的数据包处理逻辑。


---

# 十三、检查 Flowtable 自动判断

执行：

```bash
nftctl flowcheck
```

正常情况下：

```text
Flowtable mode: auto

No conflicting forwarding-path rules detected.
```


这意味着自动模式允许启用 Flowtable。


如果存在冲突，则会列出对应：

```text
family
table
chain
type
hook
policy
rules
```


---

# 十四、查看 Fast Path 状态

执行：

```bash
nftctl status
```

例如：

```text
Flow mode:       auto
Auto check:      clear
Fast path:       on
```


含义：

```text
Flow mode: auto
```

使用自动判断。


```text
Auto check: clear
```

没有发现需要阻止 Fast Path 的规则。


```text
Fast path: on
```

Flowtable 已经创建并加载。


---

# 十五、检查实际进入 Fast Path 的连接

产生实际转发流量之后执行：

```bash
nftctl offload
```

如果存在 software flowtable 连接，会显示带：

```text
[OFFLOAD]
```

的 conntrack 项。


也可以直接：

```bash
conntrack -L 2>/dev/null | grep '\[OFFLOAD\]'
```


如果当前没有符合条件的活动连接，可能显示：

```text
None currently visible.
```

这本身不代表配置错误。


---

# 十六、查看实际 nftctl 规则

执行：

```bash
nft list table ip nftctl
```

Fast Path 开启时应该包含类似：

```text
flowtable fastpath {
    hook ingress priority filter
    devices = { eth0 }
}
```


并包含：

```text
chain forward {
    type filter hook forward priority filter;
    policy accept;

    ct mark 0x40000000 ip protocol { tcp, udp } flow add @fastpath
}
```


---

# 十七、查看全部 nftables

执行：

```bash
nft list ruleset
```


系统原生：

```text
nft
```

命令没有被 nftctl 替代。


因此：

```bash
nft list ruleset
```

仍然是正常系统命令。


管理 nftctl 使用：

```bash
nftctl ...
```


---

# 十八、状态检查

执行：

```bash
nftctl status
```

例如：

```text
Rules:           2
Flow mode:       auto
Auto check:      clear
Fast path:       on
IPv4 forward:    1
Conntrack:       46 / 262144
Qdisc:           fq
TCP CC:          bbr
Interfaces:      eth0
```


各项含义：

```text
Rules
```

当前转发规则数量。


```text
Flow mode
```

Flowtable 配置模式。


```text
Auto check
```

自动兼容性检查结果。


```text
Fast path
```

当前是否创建 Flowtable。


```text
IPv4 forward
```

IPv4 forwarding 是否开启。


```text
Conntrack
```

当前连接跟踪数量和最大容量。


```text
Qdisc
```

系统默认 qdisc。


```text
TCP CC
```

服务器本机 TCP congestion control。


```text
Interfaces
```

Flowtable 使用的网络接口。


---

# 十九、自检

执行：

```bash
nftctl selftest
```

正常情况下：

```text
Database:
  OK

IPv4 forwarding:
  OK

nftables table:
  OK

Routes:
  443 -> 1.2.3.4:443: OK
  10086 -> 2.3.4.5:443: OK

Saved ruleset:
  OK

Flowtable:
  ON

Result: OK
```


重点：

```text
Result: OK
```


---

# 二十、完整诊断

执行：

```bash
nftctl diag
```

输出包括：

```text
系统版本
CPU 数量
RAM

IPv4 route

网络接口

网卡 RX/TX queue

GRO
GSO
TSO
checksum offload
hw-tc-offload

qdisc

Flowtable compatibility check

OFFLOAD 连接

conntrack

socket statistics

完整 nftctl ruleset
```


排查网络或性能问题时，保存：

```bash
nftctl diag
```

完整输出即可。


---

# 二十一、检查 IPv4 Forwarding

执行：

```bash
sysctl net.ipv4.ip_forward
```

正常：

```text
net.ipv4.ip_forward = 1
```


也可以：

```bash
nftctl status
```


---

# 二十二、检查 BBR

执行：

```bash
sysctl net.ipv4.tcp_congestion_control
```

支持并启用时：

```text
net.ipv4.tcp_congestion_control = bbr
```


也可以：

```bash
nftctl status
```

查看：

```text
TCP CC: bbr
```


BBR主要作用于服务器本机 TCP socket。

转发数据路径主要由 nftables、conntrack、routing、Flowtable 和网络接口处理。


---

# 二十三、检查 fq

执行：

```bash
sysctl net.core.default_qdisc
```

通常：

```text
net.core.default_qdisc = fq
```


---

# 二十四、Conntrack

执行：

```bash
nftctl status
```

例如：

```text
Conntrack: 46 / 262144
```


表示：

```text
当前连接跟踪数量：46
最大数量：262144
```


脚本会根据服务器 RAM 自动选择容量，并且不会主动降低系统已有的更大配置。


---

# 二十五、服务器重启

不需要重新添加规则。


规则数据库：

```text
/etc/nftctl/db
```


配置：

```text
/etc/nftctl/config
```


生成的 nftables 文件：

```text
/etc/nftctl/rules.nft
```


服务：

```text
nftctl.service
```


查看：

```bash
systemctl status nftctl
```


手动重新加载：

```bash
nftctl reload
```


---

# 二十六、数据库格式

数据库：

```text
/etc/nftctl/db
```


每一行：

```text
LISTEN_PORT TARGET_IP TARGET_PORT
```


例如：

```text
443 1.2.3.4 443
10086 1.2.3.4 8443
20000 8.8.8.8 53
```


安装程序兼容旧版两列格式。


旧：

```text
443 1.2.3.4
```

会自动转换成：

```text
443 1.2.3.4 443
```


---

# 二十七、网络接口

默认：

```text
IFACES=""
```


表示自动检测。


一般不需要修改。


如果明确需要指定接口：

```text
/etc/nftctl/config
```


例如：

```text
FLOWTABLE=auto
IFACES="eth0"
```


修改以后：

```bash
nftctl reload
```


---

# 二十八、Flowtable 手动模式

推荐使用：

```text
FLOWTABLE=auto
```


也支持手动开启：

```text
FLOWTABLE=on
```


以及关闭：

```text
FLOWTABLE=off
```


修改：

```bash
nano /etc/nftctl/config
```


修改以后：

```bash
nftctl reload
```


正常情况下不需要手动设置。


---

# 二十九、云平台安全组

云平台安全组需要允许的是：

```text
本机监听端口
```


例如：

```text
10086 -> 1.2.3.4:443
```


云平台需要允许：

```text
TCP 10086
UDP 10086
```


目标端口：

```text
443
```

不是本机安全组需要开放的端口。


---

# 三十、最常用命令

交互添加：

```bash
nftctl add
```


相同端口：

```bash
nftctl add 1.2.3.4 443
```


不同端口：

```bash
nftctl add 1.2.3.4 10086 443
```


查看：

```bash
nftctl list
```


删除：

```bash
nftctl del 443
```


重新加载：

```bash
nftctl reload
```


状态：

```bash
nftctl status
```


Flowtable 检查：

```bash
nftctl flowcheck
```


检查实际 offload：

```bash
nftctl offload
```


自检：

```bash
nftctl selftest
```


完整诊断：

```bash
nftctl diag
```


查看实际 nftctl 表：

```bash
nft list table ip nftctl
```


查看全部 nftables：

```bash
nft list ruleset
```


查看服务：

```bash
systemctl status nftctl
```


---

# 三十一、推荐安装后检查流程

安装完成以后依次执行：

```bash
nftctl status

nftctl flowcheck

nftctl selftest

nft list table ip nftctl
```


正常目标：

```text
Flow mode:       auto
Auto check:      clear
Fast path:       on
IPv4 forward:    1
```


并且：

```text
Result: OK
```


有实际转发流量之后：

```bash
nftctl offload
```


如果连接已经进入 software fast path，可以看到：

```text
[OFFLOAD]
```


---

# 三十二、最简使用流程

添加相同端口：

```bash
nftctl add 1.2.3.4 443
```


添加不同端口：

```bash
nftctl add 1.2.3.4 10086 443
```


查看：

```bash
nftctl list
```


删除：

```bash
nftctl del 10086
```


状态：

```bash
nftctl status
```


排错：

```bash
nftctl selftest

nftctl diag
```
