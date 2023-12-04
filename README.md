在 VPS 上愉快使用 CF 官方版 Warp —— 剔除官方版 Warp 客户端在 OS 里留的〇
https://iedon.com/2023/07/13/992.html
Jul 13, 2023
—

Posted by

iEdon
on 文章列表
背景
最近在自建 Mastodon 实例。Mastodon 利用 AcitivityPub 协议彼此通讯，来交换 Toot。这带来了一个潜在的安全问题：己方实例在请求对端服务的时候，会暴露真实 IP 给其他实例维护者。大家能首先想到的规避方法是为 Mastodon 实例挂一个出口方向的 VPN，来隐藏自己的真实 IP。

以下是网络环境：

Mastodon 利用 Docker 管理，有单独的业务网段
禁止 Docker 接管 iptables，全部利用 DNAT 来暴露目标服务到内网
VPS 关闭所有入站 TCP 连接
对外服务采用 Cloudflare Tunnel(旧称 Argo Tunnel) 公布，目的是能够避免源站探测
起因
既然要挂 Cloudflare 的 VPN，有官方客户端和第三方可以选择。然而遇到了以下问题：

VPS 所在地区被禁止使用诸如 wgcf 这样的第三方客户端，只能利用官方 Warp 客户端
官方 Warp 客户端虽然提供 Proxy 模式，但是内存泄漏尤为严重，并且存在大量资源不释放之问题，netstat 能看到大量 CLOSE_WAIT，故使用 VPN 模式(Warp 模式)
挂上官方 Warp 客户端后，遇到了这些问题：

VPS 立即丢失所有入站连接(所有流量被 Cloudflare 的规则压制)，SSH 中断，内网中断
Warp 客户端不停劫持 /etc/resolv.conf 来提供安全的 DNS，这不是我想要的
Warp 客户端不停地向 nftables 写入规则，这不是我想要的
Warp 客户端如果不能成功写入 DNS 和防火墙规则则直接拒绝启动
Warp 客户端在当前时间节点不提供任何参数关闭这些功能
内存泄漏严重的 warp-svc
内存泄漏严重的 warp-svc
而我想要的只是一个单纯的 tun。至于数据包的来去，应该由我掌管。否则怎么让 Docker 的网段单独走 Warp 呢？

Warp 客户端到底做了什么
搞清楚 Warp 客户端向系统灌输了什么东西，就能见招拆招，把它们统统剥离掉。

一个全新的路由表(ID: 65743)，内含几乎全网聚合后的 IP CIDR
一个 ip rule，用来做策略路由
针对与 Cloudflare Warp 服务端自己的通讯，打上 fwmark 0x100cf 标记
循环检测写入 nftables 规则(table inet cloudflare-warp)
循环检测写入 /etc/resolv.conf
Warp 客户端成功建立 VPN 隧道后，向内核灌了路由表，设置了策略路由。所有非 Cloudflare Warp 的通讯流量(没有 0x100cf 标记的流量)，统统走 65743 号路由表，让数据包从 VPN 隧道出。

通过监控 nftables 规则，来防止数据逃逸和意外的访问(规则很细致)。通过监控 /etc/resolv.conf，来防止 DNS 泄漏。

上述操作保证了单机环境下的理想安全上网需求。虽然很感谢这些工作，但这不是我需要的。

逐一剔除
知道了上述关键点，就可以写脚本来还原网络环境了。

#!/bin/bash

# 让 /etc/resolv.conf 可以被改写，防止 Warp 拒绝启动
chattr -i /etc/resolv.conf
sleep 10

# 接入 Warp
/usr/bin/warp-cli set-mode warp
/usr/bin/warp-cli connect
/usr/bin/warp-cli disable-dns-log

sleep 10

# 【关键部分】恢复网络环境，保护好 resolv.conf，使用真 nft cli 删除 cloudflare 的防火墙规则
ip rule del not from all fwmark 0x100cf lookup 65743
ip -6 rule del not from all fwmark 0x100cf lookup 65743
nft_real delete table inet cloudflare-warp # 参见后文
ip route add 0.0.0.0/0 dev CloudflareWARP scope link  table 65743
echo -e 'nameserver 1.1.1.1\nnameserver 2606:4700:4700::1111' > /etc/resolv.conf && chattr +i /etc/resolv.conf

# 只让 Docker 网段访问 Warp，并且内网还是走原来的内网路由,仅本主题需要。
function setrules () {
        ip rule $1 from 172.17.0.0/24 lookup 65743
        ip rule $1 from 172.17.2.0/23 lookup 65743
        ip rule $1 from 172.17.0.1 lookup main
        ip rule $1 from 172.17.2.1 lookup main
        ip rule $1 to 172.17.0.1 lookup main
        ip rule $1 to 172.17.2.1 lookup main
        ip rule $1 to 10.0.0.0/8 lookup main
        ip rule $1 to 172.20.0.0/14 lookup main
}

setrules del
setrules add

exit 0
接下来让该脚本能够在 warp-svc 启动后再调用(仅一次)。当然利用 systemd 来完成。

[Unit]
Description=Cloudflare Warp Network Restore
After=pre-network.target
After=warp-svc.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/data/wgcf
ExecStart=/data/wgcf/init.sh

[Install]
WantedBy=multi-user.target
做假的防火墙
注意到上面脚本中的 nft_real 了吗？ Warp 客户端通过不断调用 nft 来管理规则。

想要让 Warp 客户端不要这么做只有做一个假的防火墙来欺骗它。将 /usr/sbin/nft 更名为 /usr/sbin/nft_real，并新建一个 /usr/sbin/nft。该脚本用来欺骗 Warp 客户端，但允许我们自己正常调用，一切都是透明的。

#!/bin/bash

# Ignore Cloudflare's call to nft

PS_NAME=`ps -e | grep $PPID| awk '{print $4}'`
if [[  "$PS_NAME" == *"warp-svc"* ]]; then
        exit 0
fi

/usr/sbin/nft_real "$@"
该脚本检测到调用方是 warp-svc 的时候，便会直接退出(返回 0)。这让 Warp 客户端以为自己的规则已经加好了。而我们自己的请求则直接原封不动传递给 nft_real。

至此，我们的系统较为干净了。

让 Docker 走 Warp 出站
接下来就是大家喜闻乐见的 SNAT 了。由于 Warp 客户端分配给我们的 IP 一般都是 172.16.0.2，我们只需要 SNAT 到 172.16.0.2 就能实现 Docker 走 Warp 出站了。

#!/bin/bash
source $1
# ...

# Docker 原来走公网接口出站，现在可以注释掉了
# iptables -t nat -A POSTROUTING -s 172.17.2.0/23 -o $PUB_NET_IF -j MASQUERADE
# 到 1.1.1.1 的 DNS 请求还是走原来的公网出站
iptables -t nat -A POSTROUTING -s 172.17.2.0/23 -d 1.1.1.1 -o $PUB_NET_IF -j MASQUERADE
# 让 Docker 走 Warp 出站
iptables -t nat -A POSTROUTING -s 172.17.2.0/23 -j SNAT --to-source 172.16.0.2
以上，让 Docker 走 Warp 出站的需求就满足了。

最后
有关为何第三方客户端的 wgcf 无法使用(无法 handshake) 的说法有这么一个：虽然 Warp 客户端使用了 WireGuard 协议，但是在 WireGuard 协议中的 Reserved 字段加入了 特殊值①，导致 Cloudflare 可以检测是否是官方客户端。最近看到 GitHub 中的 一些脚本② 已经开始尝试魔改 WireGuard Go，来支持指定 Reserved 字节，不过本文撰写时，尚无稳定可用的发布，尚处于开发中的样子。

以上。

参考链接
WireGuard: Next Generation Kernel Network Tunnel, https://www.wireguard.com/papers/wireguard.pdf
GitHub: fscarmen/warp, https://github.com/fscarmen/warp
Cloudflare  Docker  Mastodon  Tunnel  VPN  VPS  Warp
