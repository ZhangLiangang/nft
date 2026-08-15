==============================
nftctl 使用说明
==============================

一、新服务器安装
------------------------------

SSH 登录服务器后，把安装脚本整段粘贴执行。

如果当前用户不是 root：

sudo bash <<'EOF'
...安装脚本内容...
EOF

如果当前用户就是 root：

bash <<'EOF'
...安装脚本内容...
EOF


二、添加规则
------------------------------

输入：

nft

然后按提示输入：

IP: 目标服务器IP
Port: 端口

例如：

nft

IP: 154.12.34.56
Port: 443

成功后显示：

OK
443 -> 154.12.34.56:443
TCP + UDP

表示：

本机 TCP 443 -> 154.12.34.56:443
本机 UDP 443 -> 154.12.34.56:443


三、继续添加其他端口
------------------------------

再次输入：

nft

例如：

IP: 154.12.34.56
Port: 8443

即可增加：

8443 -> 154.12.34.56:8443


四、修改已有端口
------------------------------

不需要先删除。

假设原来：

443 -> 154.12.34.56:443

现在需要改成：

443 -> 89.20.30.40:443

直接输入：

nft

然后：

IP: 89.20.30.40
Port: 443

脚本会自动替换原来的 443 规则。


五、查看所有规则
------------------------------

输入：

nftctl list

例如显示：

PORT       IP                 PROTO
-----      ---------------    -------
443        89.20.30.40        TCP+UDP
8443       154.12.34.56       TCP+UDP
10000      20.30.40.50        TCP+UDP


六、删除规则
------------------------------

例如删除 443：

nftctl del 443

成功显示：

OK
Removed: 443


也可以输入：

nftctl del

然后按提示输入：

Port: 443


七、重新加载规则
------------------------------

输入：

nftctl reload


八、查看 nft 实际规则
------------------------------

查看 nftctl 规则：

nft list table ip nftctl

查看系统全部 nftables 规则：

nft list ruleset


注意：

直接输入：

nft

表示进入添加界面。


输入：

nft list ...

表示使用系统原生 nft 命令。


九、服务器重启
------------------------------

不需要重新添加。

规则保存在：

/etc/nftctl/db

开机自动加载服务：

nftctl.service

查看服务状态：

systemctl status nftctl

手动重新加载：

nftctl reload


十、检查 IPv4 Forwarding
------------------------------

输入：

sysctl net.ipv4.ip_forward

正常应该显示：

net.ipv4.ip_forward = 1


十一、检查 BBR
------------------------------

输入：

sysctl net.ipv4.tcp_congestion_control

正常情况下显示：

net.ipv4.tcp_congestion_control = bbr


十二、检查 fq
------------------------------

输入：

sysctl net.core.default_qdisc

正常情况下显示：

net.core.default_qdisc = fq


十三、最常用的三个命令
------------------------------

添加或修改：

nft

查看：

nftctl list

删除：

nftctl del 443


十四、常用示例
------------------------------

添加 443：

nft

IP: 1.2.3.4
Port: 443


添加 8443：

nft

IP: 1.2.3.4
Port: 8443


查看：

nftctl list


删除 443：

nftctl del 443


重新加载：

nftctl reload


查看实际规则：

nft list table ip nftctl


十五、注意事项
------------------------------

1. 当前模式为：

本机端口 -> 目标IP:相同端口

例如：

443 -> 1.2.3.4:443

不能直接配置：

10086 -> 1.2.3.4:443

除非修改脚本支持独立目标端口。


2. TCP 和 UDP 会同时配置。

例如添加：

443 -> 1.2.3.4:443

实际包含：

TCP 443 -> 1.2.3.4:443
UDP 443 -> 1.2.3.4:443


3. 如果同一个端口再次添加，会覆盖旧 IP。

例如原来：

443 -> 1.1.1.1:443

重新执行：

nft

IP: 2.2.2.2
Port: 443

最终变成：

443 -> 2.2.2.2:443


4. 云服务器安全组也必须允许对应入站端口。

例如使用：

443

云平台安全组也需要允许相应的 TCP/UDP 443。

服务器内部 nftables 无法修改云平台外层安全组。


==============================
命令速查
==============================

添加：
nft

查看：
nftctl list

删除：
nftctl del 443

重新加载：
nftctl reload

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
