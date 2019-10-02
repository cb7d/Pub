# BBR

## 什么是BBR

[项目地址](https://github.com/google/bbr)

Google 向 Linux Kernel 提交了 [pr](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=0f8782ea14974ce992618b55f0c041ef43ed0b78)。本质是改良版的 TCP 拥塞控制算法，能够最大化的利用网络连接，降低延迟。BBR通过实时计算带宽和最小RTT来决定发送速率pacing rate和窗口大小cwnd。完全摒弃丢包作为拥塞控制的直接反馈因素。

## 如何使用

由于笔者使用的远程主机为Ubuntu，这里就以Ubuntu 18为例，内核版本为

### 查看内核版本

```shell
uname -r
```

内核版本大于4.9即可直接开启

```shell
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

查看输出

```shell
sysctl -p
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

之后reboot就可以啦，重启之后可以验证

```shell
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
lsmod |grep bbr
// 能看到以下输出
net.ipv4.tcp_available_congestion_control = reno cubic bbr
net.ipv4.tcp_congestion_control = bbr
tcp_bbr                20480  6
```

