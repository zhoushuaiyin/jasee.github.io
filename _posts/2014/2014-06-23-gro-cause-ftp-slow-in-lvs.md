---
layout: post
title: 都是GRO惹的祸--记LVS中FTP速度慢问题
category: 运维
description: 记录一次排查LVS上FTP传输速度特别慢的过程。
tags: ["GRO", "LVS", "FTP", "MTU"]
---

### 背景
我们有一台FTP服务器磁盘没有做RAID，存在风险，准备进行替换，所以让大家都通过LVS的VIP来访问该FTP服务。前段时间LVS主服务器重新安装了CentOS 6.3的系统，后来发现上传文件的时候，多个文件只有一个文件传上去了，开始排查问题。

### 为什么少文件
当收到文件缺失的反馈的时候，基本就确定是LVS的速度出了问题了，测试了一下，果然只有10k/s的速率。
在使用LVS时，FTP这种服务是有些特殊的，他的控制流和数据流不在一个端口，如果单个文件传输时间超过了LVS超时时间，那么21端口的连接就断开了，后续的命令自然也就无法发送，剩余的文件不会继续上传。因为目前的内网环境中还没有单个文件特别大的情况，所以我们的LVS超时时间设置为120s，以前也没出现过问题。收到反馈完成简单测试后，首先将FTP实例切换到LVS备机上，并手工重传了文件，之后开始寻找FTP数据传输缓慢的原因。

### 测试及定位
由于最近的变更只有更新系统，那么原因应该就在新系统上了，不过具体为什么，只能进行测试排查。
首先就在测试环境的一台古老的台式机上使用CentOS 6.3尝试复现，结果速度并不慢，事实上这一步给后续的排查带来一定的误导。
既然测试环境无法复现，就再次到主LVS上开了一个测试实例，能够观察到上传数据较慢的问题。比较奇怪的是，该LVS上有其他很多业务流量，都没有观察到该问题，只能尝试抓包。
通过在客户端、LVS以及FTP服务器上抓包发现，在开始进行数据传输后，LVS不断收到数据长度为2920或更大的包，并向客户端返回"need to frag (mtu 1500)"的错误，客户端只能重新分片发送，重发导致的延迟非常大，整体速度也就只有10k/s了。比较奇怪的原因是:

1. 三次握手阶段已经协商MSS为1460(MTU为1500)了，为什么会收到2920的包？
2. 客户端重发时，已经能够观察到新发送的包都是1460了，为什么LVS依然收到了2920的包？

我把抓包结果拿给网络工程师看了，他也感觉很奇怪。因此还对交换机的MTU进行了验证，也确定交换机不会允许通过MTU超过1500的包。再回想了一下，测试环境同线上环境，使用的是同样的系统和部署方法，但结果不同，看来和硬件环境也有关系，那问题可能出在驱动这块。

### 原来有个东西叫GRO

Google一下，发现了一个神奇的东西叫GRO，之前也[有兄弟在这上面栽过跟头了][1]，不过刚开始不知道关键字，没搜到这坑。
简单来说，内核版本在2.6.29和2.6.39之间时，如果网卡和驱动都支持，默认是开启GRO功能的。这个功能是远古时代MTU设置偏小的一个补救，网卡会将符合条件的数据包组合后再发往协议栈，以便降低多个小数据包分别在协议栈中处理带来的消耗。这个功能一般来说是没什么问题的，因为终端组合数据包之后直接向上层发送，不会出现超过网卡MTU问题，但对于LVS来说就大大不妙了。LVS将这些大数据包向RS节点转发的时候，会被毫不犹豫的拒绝，只能要求客户端重发并反复此过程。
该LVS上其他流量未收到影响是因为这些流量都很小，数据大小没有超过1460的。测试环境没能复现，是因为那机器太破了网卡不支持这种高级功能。
为了能继续愉快的玩耍，不得不含泪把这高级功能禁用掉。我们是做了网卡bonding的，要将实际的网卡都禁用掉：

```sh
$ ethtool -K em1 gro off
$ ethtool -K em2 gro off
```

最好也写在`rc.local`里，防止系统重启后死灰复燃。

[1]: http://blog.hellosa.org/2012/07/21/kernel-2-6-29-2-6-39-lvs-director-gro.html