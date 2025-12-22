+++
date = '2025-12-23T00:44:39+08:00'
draft = false
title = 'K8s Troubleshooting'
+++

前言
最近在本地虚拟机环境（CentOS 7）搭建 Kubernetes 集群运行微服务 Demo 时，遇到一个非常诡异的“灵异事件”。

起因：由于宿主机休眠，我挂起（Suspend）了一段时间虚拟机。 现象：恢复运行后，集群状态看起来一切正常（Node Ready，Pod Running），但访问 NodePort 暴露的服务时，直接报 502 Bad Gateway。

排查过程极其曲折，从 HTTP 协议一路查到 Linux 内核参数，最终发现是操作系统在网络重置时的“安全机制”坑了 Kubernetes。本文记录了完整的排查思路，希望能帮大家避坑。

🕵️‍♂️ 第一阶段：表象排查（Application Layer）
首先，我尝试访问前端服务：

Bash

curl -I http://192.168.6.141:30007/
# HTTP/1.1 502 Bad Gateway
初步分析： 502 通常意味着网关（Service/Kube-proxy）找不到后端 Pod，或者连接被拒绝。

检查 Pod 状态： kubectl get pods -A 显示所有 Pod 均为 Running 且 Ready (1/1)。应用没挂。

检查 Service 关联： kubectl get ep frontend 显示 Endpoints 存在且 IP 正确（如 10.244.104.23:8080）。

看日志： 查看 Frontend Pod 日志，没有报错，甚至显示 Server started。

奇怪点：应用活着，配置没动过，重启前好好的，恢复后就断了。

🕵️‍♂️ 第二阶段：网络层排查（Network Layer）
既然应用层没问题，怀疑是网络链路不通。我决定做一个最简单的测试：Ping。

我进入 Node1 节点，尝试 Ping 位于 Node2 上的 Pod IP：

Bash

ping 10.244.104.23
# 100% packet loss
关键转折：Ping 不通！这说明问题不在 HTTP 层，而是更底层的 IP 连通性 出了问题。

路由表分析
我的集群使用的是 Calico CNI（IPIP 模式）。查看路由表：

Bash

ip route
# 输出包含：
# 10.244.104.0/26 via 192.168.6.142 dev tunl0 proto bird onlink
这说明去往 Node2 Pod 的流量，应该经过 tunl0 隧道接口封装，发往 Node2 的物理 IP (192.168.6.142)。路由表看起来是正常的。

🕵️‍♂️ 第三阶段：内核层排查（The Root Cause）
为了搞清楚包到底丢在哪，我祭出了大杀器：tcpdump。

1. 双端抓包
我在 发送端（Node1） 和 接收端（Node2） 同时抓包：

Node1: tcpdump -i any host <PodIP>

现象：能看到发出的 ICMP Request，也能看到 TCP SYN 包。包发出去了。

Node2: tcpdump -i any host <PodIP>

现象：能收到 Node1 发过来的包。

推论：物理网络没问题，隧道也没断。Node2 收到了包，但没有回复，也没有转发给 Pod。这是一个典型的“黑洞”现象。

2. 检查内核参数
Node2 收到了包却不转发，嫌疑最大的就是 Linux 内核的转发开关。

检查 Node2 的 IP Forwarding 配置：

Bash

sysctl net.ipv4.ip_forward
# 输出：net.ipv4.ip_forward = 0  <-- 凶手找到了！
真相大白： Kubernetes 的 Node 节点本质上是一台路由器。它需要把物理网卡收到的包，转发给 CNI 的虚拟网卡（cali*）。 当虚拟机从挂起状态恢复时，CentOS 的网络服务（NetworkManager）重启了网卡，并出于安全策略，将 ip_forward 默认重置为了 0（关闭）。

这就导致 Node2 变成了一个“自闭”的节点：它收到了发给 Pod 的包，但因为它被禁止转发流量，所以直接把包丢弃了。

✅ 解决方案
临时修复
立即开启转发功能，网络瞬间恢复：

Bash

sysctl -w net.ipv4.ip_forward=1
永久修复（防止重启失效）
为了避免下次重启或挂起后再挂，必须将配置写入文件：

Bash

cat <<EOF > /etc/sysctl.d/k8s-forward.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.tunl0.rp_filter = 0
EOF
sysctl --system
📝 总结与教训
这次排查让我对 Kubernetes 的底层网络有了更直观的认识：

环境避坑：K8s 对宿主机内核参数高度依赖。虚拟机挂起/恢复操作非常容易导致 ip_forward、rp_filter 等关键参数被系统重置。

排查心法：不要盲目猜测。

curl 不通 -> 查 ping。

ping 不通 -> 查 ip route（路由）。

路由没问题 -> 用 tcpdump 确定丢包位置。

确定丢包位置 -> 查 sysctl 内核参数。

核心认知：Kubernetes Node 就是路由器。net.ipv4.ip_forward = 1 是容器网络的生命线。