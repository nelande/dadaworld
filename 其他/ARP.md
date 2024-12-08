[toc]

# ARP

### 简介

用于发现IP地址对应的MAC地址 ，用于局域网中，因为在局域网中，通常用网络适配器（网卡）标识设备，而不是IP。

当主机记录了所有的设备的MAC地址后，不同主机通过MAC地址表通信



如，当一台设备刚接入网络时，会记录网关的MAC地址



### ARP 请求和相应 互发现

假设现在有一个LAN，LAN的网段是202.117.15.0/24

主机A的IP地址：202.117.15.10，MAC地址00:00:00:00:00:10

主机B的IP地址：202.117.15.11，MAC地址00:00:00:00:00:11

A此时要给B发送一个UDP包（三层包），A不知道B的MAC地址，过程如下：

![image-20240304015335410](assets/image-20240304015335410.png)

1. A 广播发送ARP请求。WHO HAS xxxx，TELL xxxx

   ![image-20240304015411297](assets/image-20240304015411297.png)

2. B 单播发送相应请求。xxxx is at xxxx

![image-20240304015434953](assets/image-20240304015434953.png)

## ARP Probe 与宣言

在局域网中，可能出现一个IP被多个MAC地址占用，所以RFC5227提出了一个新的机制：ACD（Address Conflict Detection）

其中包括探测包，和宣言包。

![image-20240304015858601](assets/image-20240304015858601.png)

ARP probe：用于检测IP地址冲突，**Sender IP填充为0，填充为0是为了避免对其他主机的ARP cache造成污染**（因为可能已经有主机正在使用候选IP地址了，不要影响别人的正常通行嘛），Target IP是候选IP地址（即本机想要使用的IP地址）

![image-20240304015937843](assets/image-20240304015937843.png)

ARP announcement：用于昭示天下（LAN）本机要使用某个IP地址了，是一个SenderIP和Traget IP**填充的都是本机IP地址的ARP request**。这会让LAN(VLAN)中所有主机都会更新自己的ARP cache，将IP地址映射到发送者的MAC地址。

![image-20240304020023882](assets/image-20240304020023882.png)

## 检测冲突

发送probe期间，发送主机可能收到ARP reply或者ARP probe

- 如果收到了ARP reply，说明候选IP地址已经有主机在用了
- 如果收到了一个Target IP地址为候选IP的ARPprobe，说明另外一个主机也同时想要使用该候选IP地址

这种情况下，两个主机都会提醒用户出现了IP地址冲突



如果上述两种ARP包都没有收到，说明候选IP地址可用。主机发送一个ARP announcement，告诉其他主机这个候选IP有人用了，这个ARP request会让LAN（VLAN）中其他主机更新ARP cache

**注意：系统运行期间，ACD会一直运行，主机会一直检测收到的ARP reqeust 和ARP reply，检测地址是否冲突，ACD 的执行频率可能是几分钟到几十分钟之间**

## 解决冲突

地址冲突处理：《RFC5527》提供了三种可选的解决机制：1）放弃使用该IP地址，2）发送一个ARP announcement来进行IP地址“守卫”，如果冲突仍然继续存在，放弃使用这个IP。3）无视冲突，继续使用这个IP

#### 