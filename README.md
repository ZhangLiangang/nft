==============================
nftctl 使用说明
==============================


一、新服务器安装
------------------------------

SSH 登录服务器后，把完整安装脚本整段粘贴执行。

如果当前用户不是 root：

sudo bash <<'EOF'
...安装脚本内容...
EOF


如果当前用户就是 root：

bash <<'EOF'
...安装脚本内容...
EOF


安装完成后建议执行：

nftctl status

nftctl selftest

确认配置正常。


二、添加规则
------------------------------

交互式添加：

nftctl add


然后按提示输入：

Target IP: 目标服务器 IP
Listen port: 本机监听端口
Target port [监听端口]: 目标服务器端口


例如：

nftctl add

Target IP: 154.12.34.56
Listen port: 443
Target port [443]:


如果目标端口和监听端口相同，Target port 这里直接按回车即可。

成功后显示类似：

Reloaded. Fast path: on

443 -> 154.12.34.56:443
TCP + UDP


表示：

本机 TCP 443 -> 154.12.34.56:443
本机 UDP 443 -> 154.12.34.56:443


三、配置不同的目标端口
------------------------------

新版支持：

本机端口 -> 目标 IP:不同端口


例如：

本机 10086 -> 154.12.34.56:443


输入：

nftctl add


然后：

Target IP: 154.12.34.56
Listen port: 10086
Target port [10086]: 443


成功后：

10086 -> 154.12.34.56:443
TCP + UDP


实际表示：

本机 TCP 10086 -> 154.12.34.56:443
本机 UDP 10086 -> 154.12.34.56:443


四、命令行直接添加
------------------------------

除了交互模式，也可以直接使用命令添加。


1. 相同端口

例如：

443 -> 154.12.34.56:443

输入：

nftctl add 154.12.34.56 443


等同于：

Target IP: 154.12.34.56
Listen port: 443
Target port: 443


2. 不同端口

例如：

10086 -> 154.12.34.56:443

输入：

nftctl add 154.12.34.56 10086 443


参数顺序为：

nftctl add 目标IP 本机监听端口 目标端口


五、继续添加其他端口
------------------------------

再次执行：

nftctl add


例如：

Target IP: 154.12.34.56
Listen port: 8443
Target port [8443]:


即可增加：

8443 -> 154.12.34.56:8443


也可以直接：

nftctl add 154.12.34.56 8443


如果目标端口不同，例如：

8443 -> 154.12.34.56:443

可以直接：

nftctl add 154.12.34.56 8443 443


六、修改已有端口
------------------------------

不需要先删除。

规则以“本机监听端口”为唯一键。

假设原来：

443 -> 154.12.34.56:443


现在需要改成：

443 -> 89.20.30.40:443


直接输入：

nftctl add 89.20.30.40 443


脚本会自动替换原来的 443 规则。


如果需要同时修改目标端口，例如：

443 -> 89.20.30.40:8443


直接输入：

nftctl add 89.20.30.40 443 8443


最终规则变成：

443 -> 89.20.30.40:8443


七、查看所有规则
------------------------------

输入：

nftctl list


例如显示：

LISTEN       TARGET IP          TARGET PORT  PROTO
------       ---------------    -----------  -------
443          89.20.30.40        443          TCP+UDP
8443         154.12.34.56       443          TCP+UDP
10000        20.30.40.50        10000        TCP+UDP


各字段含义：

LISTEN
本机监听端口

TARGET IP
目标服务器 IP

TARGET PORT
目标服务器端口

PROTO
协议，目前同时配置 TCP 和 UDP


八、删除规则
------------------------------

规则按照本机监听端口删除。


例如删除本机 443：

nftctl del 443


成功后显示类似：

Reloaded. Fast path: on

Removed: 443


也可以输入：

nftctl del


然后按提示输入：

Listen port: 443


注意：

如果规则是：

10086 -> 1.2.3.4:443


需要删除的是：

nftctl del 10086


而不是：

nftctl del 443


九、重新加载规则
------------------------------

输入：

nftctl reload


正常显示类似：

Reloaded. Fast path: on


如果当前系统环境不适合启用 flowtable，也可能显示：

Reloaded. Fast path: off


规则仍然可以正常工作。


十、查看 nft 实际规则
------------------------------

查看 nftctl 创建的规则：

nft list table ip nftctl


查看系统全部 nftables 规则：

nft list ruleset


注意：

新版不再修改或替代系统原生 nft 命令。


因此：

nft

就是系统原生 nftables 命令。


添加转发规则必须使用：

nftctl add


不要再使用旧版的：

nft

进入添加界面。


十一、服务器重启
------------------------------

服务器重启后不需要重新添加规则。

规则数据库保存在：

/etc/nftctl/db


配置文件：

/etc/nftctl/config


生成的 nftables 规则：

/etc/nftctl/rules.nft


开机自动加载服务：

nftctl.service


查看服务状态：

systemctl status nftctl


手动重新加载：

nftctl reload


十二、查看运行状态
------------------------------

输入：

nftctl status


例如：

nftctl

Rules:           3
Fast path:       on
IPv4 forward:    1
Conntrack:       152 / 1048576
Qdisc:           fq
TCP CC:          bbr
Interfaces:      eth0


主要项目说明：

Rules
当前规则数量

Fast path
flowtable fast path 是否启用

IPv4 forward
IPv4 转发是否启用

Conntrack
当前连接跟踪数量 / 最大容量

Qdisc
默认队列调度器

TCP CC
本机 TCP 拥塞控制算法

Interfaces
当前检测到的网络接口


十三、运行自检
------------------------------

输入：

nftctl selftest


正常情况下类似：

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

Result: OK


如果最后显示：

Result: OK

说明主要配置正常。


十四、查看详细网络诊断
------------------------------

输入：

nftctl diag


会显示包括：

系统信息

CPU 数量

内存容量

IPv4 路由

网络接口

网卡队列

GRO

GSO

TSO

Checksum Offload

Qdisc

Conntrack

Socket 统计

nftctl 实际规则


如果需要排查性能或网络问题，可以保存：

nftctl diag

的完整输出。


十五、检查 IPv4 Forwarding
------------------------------

输入：

sysctl net.ipv4.ip_forward


正常应该显示：

net.ipv4.ip_forward = 1


也可以直接：

nftctl status


查看：

IPv4 forward: 1


十六、检查 BBR
------------------------------

输入：

sysctl net.ipv4.tcp_congestion_control


如果系统支持并已经启用 BBR，通常显示：

net.ipv4.tcp_congestion_control = bbr


也可以：

nftctl status


查看：

TCP CC: bbr


注意：

BBR 主要作用于服务器本机 TCP 连接。

流量转发本身主要依赖 nftables、conntrack、网卡处理和 flowtable 等机制。


十七、检查 fq
------------------------------

输入：

sysctl net.core.default_qdisc


正常情况下可能显示：

net.core.default_qdisc = fq


也可以：

nftctl status


查看：

Qdisc: fq


十八、检查 Fast Path
------------------------------

输入：

nftctl status


查看：

Fast path: on


如果显示：

Fast path: on

说明 nftables flowtable fast path 已启用。


如果显示：

Fast path: off

规则仍然正常工作，只是当前没有启用 flowtable fast path。


默认配置：

/etc/nftctl/config


内容通常为：

FLOWTABLE=auto
IFACES=""


auto 表示自动判断是否启用。


十九、最常用命令
------------------------------

交互添加：

nftctl add


快速添加相同端口：

nftctl add 1.2.3.4 443


快速添加不同端口：

nftctl add 1.2.3.4 10086 443


查看：

nftctl list


删除：

nftctl del 443


重新加载：

nftctl reload


查看状态：

nftctl status


自检：

nftctl selftest


详细诊断：

nftctl diag


二十、常用示例
------------------------------

示例 1：

本机 443 -> 1.2.3.4:443

输入：

nftctl add 1.2.3.4 443


结果：

443 -> 1.2.3.4:443


--------------------------------------------------


示例 2：

本机 8443 -> 1.2.3.4:8443

输入：

nftctl add 1.2.3.4 8443


结果：

8443 -> 1.2.3.4:8443


--------------------------------------------------


示例 3：

本机 10086 -> 1.2.3.4:443

输入：

nftctl add 1.2.3.4 10086 443


结果：

10086 -> 1.2.3.4:443


--------------------------------------------------


示例 4：

本机 20000 -> 8.8.8.8:53

输入：

nftctl add 8.8.8.8 20000 53


结果：

20000 -> 8.8.8.8:53


实际同时包含：

TCP 20000 -> 8.8.8.8:53
UDP 20000 -> 8.8.8.8:53


--------------------------------------------------


查看全部规则：

nftctl list


删除本机 10086：

nftctl del 10086


重新加载：

nftctl reload


查看实际 nftables 规则：

nft list table ip nftctl


二十一、注意事项
------------------------------

1. 新版支持相同端口和不同端口。

相同端口：

443 -> 1.2.3.4:443


不同端口：

10086 -> 1.2.3.4:443


都可以直接配置。


--------------------------------------------------


2. 添加规则时参数顺序必须注意。

相同端口：

nftctl add 目标IP 本机监听端口


例如：

nftctl add 1.2.3.4 443


不同端口：

nftctl add 目标IP 本机监听端口 目标端口


例如：

nftctl add 1.2.3.4 10086 443


--------------------------------------------------


3. 交互模式下目标端口可以直接回车。

例如：

Target IP: 1.2.3.4
Listen port: 443
Target port [443]:


如果直接按回车：

目标端口自动使用 443。


最终：

443 -> 1.2.3.4:443


--------------------------------------------------


4. TCP 和 UDP 会同时配置。

例如：

10086 -> 1.2.3.4:443


实际包含：

TCP 10086 -> 1.2.3.4:443
UDP 10086 -> 1.2.3.4:443


--------------------------------------------------


5. 相同监听端口再次添加会覆盖原规则。

例如原来：

443 -> 1.1.1.1:443


重新执行：

nftctl add 2.2.2.2 443


最终变成：

443 -> 2.2.2.2:443


原来的：

443 -> 1.1.1.1:443

会被替换。


--------------------------------------------------


6. 也可以修改目标端口。

例如原来：

443 -> 1.1.1.1:443


执行：

nftctl add 1.1.1.1 443 8443


最终：

443 -> 1.1.1.1:8443


--------------------------------------------------


7. 删除规则按照本机监听端口删除。

例如：

10086 -> 1.2.3.4:443


删除：

nftctl del 10086


--------------------------------------------------


8. 云服务器安全组必须允许本机监听端口。

例如：

10086 -> 1.2.3.4:443


云平台安全组需要允许的是：

TCP 10086
UDP 10086


不是目标端口 443。


服务器内部 nftables 无法修改云平台外层安全组。


--------------------------------------------------


9. 系统原生 nft 命令没有被修改。

因此：

nft list ruleset

nft list table ip nftctl

等命令都可以正常使用。


管理 nftctl 必须使用：

nftctl ...


例如：

nftctl add
nftctl list
nftctl del
nftctl reload


--------------------------------------------------


10. 规则数据库格式。

规则保存在：

/etc/nftctl/db


新版每条规则格式：

本机监听端口 目标IP 目标端口


例如：

443 1.2.3.4 443
10086 1.2.3.4 443
20000 8.8.8.8 53


旧版两列规则：

443 1.2.3.4

安装新版时会自动转换成：

443 1.2.3.4 443


==============================
命令速查
==============================

交互添加：

nftctl add


相同端口：

nftctl add 1.2.3.4 443


不同端口：

nftctl add 1.2.3.4 10086 443


查看：

nftctl list


删除：

nftctl del 443


重新加载：

nftctl reload


状态：

nftctl status


自检：

nftctl selftest


详细诊断：

nftctl diag


查看实际规则：

nft list table ip nftctl


查看全部 nft：

nft list ruleset


查看服务：

systemctl status nftctl


查看 IP Forward：

sysctl net.ipv4.ip_forward


查看 BBR：

sysctl net.ipv4.tcp_congestion_control


查看 fq：

sysctl net.core.default_qdisc


==============================
最简使用流程
==============================

添加：

nftctl add


查看：

nftctl list


删除：

nftctl del 本机监听端口


检查：

nftctl status


排错：

nftctl selftest

nftctl diag
