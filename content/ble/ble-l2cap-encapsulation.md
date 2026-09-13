---
title: "从“Excel 大表”到底层天线（下篇）：深入 L2CAP，拆解报文层层加头与分流真相"
date: 2026-09-13T22:56:46+08:00
draft: false
summary: "ATT 的报文从未直接裸露在射频帧中，其上始终套着 L2CAP。本文拆解 L2CAP 的 4 字节头、CID 多路复用与分段重组，看清层层加头的真相。"
---

在上一篇《从“Excel 大表”到“空口发包”：彻底搞懂 GATT 与 ATT 的分工真相》中，我们理清了上层的逻辑边界：

- <strong>GATT</strong> 管的是“表格设计与业务语义”，把数据组织成 Service 与 Characteristic；

- <strong>ATT</strong> 管的是“具体操作与空口搬运”，只按 Handle 行号搬运裸字节，负责发送 Read、Write、Notify 等 PDU。

然而，当我们真正插上抓包器（如 Wireshark / Ellisys）去捕获空中的每一帧无线报文时，会发现一个关键细节：<strong>ATT 产生的报文，从来没有直接裸露在无线射频帧中。在它之上，始终死死套着一层 L2CAP（Logical Link Control and Adaptation Protocol，逻辑链路控制与适配协议）。</strong>

这就引出了很多嵌入式与移动端工程师调试时的深层困惑：

<strong>既然我们在代码里操作的都是某个 Handle 的读写，空中飞的也是 ATT 的指令，为什么协议栈底下非要硬塞一层 L2CAP？ATT 与 L2CAP 到底是什么从属关系？</strong>

网络协议栈最容易被忽略的物理真相只有四个字：“层层加头（封装）”与“层层扒头（解包）”。每往下一层，数据本身未必变了，但都会补上一段只有这一层才看得懂、才用得上的控制信息。L2CAP 的 4 字节头正是 ATT 从“操作属性的内容”变成“能在一条 BLE 链路上被识别、承载和转交的报文”的关键。本文不使用过度失真的生活修辞，紧接上篇的心率手环实例，从空口二进制字节切面与芯片硬件约束切入，彻底看清 ATT 是如何长在 L2CAP 之上的。


## 一、 承接上篇：一次真实的空中抓包与“层层加头”

延续上篇心率手环的 Attribute 大表模型：

- Handle 0x0003：心率数值（UUID 0x2A37）

- Handle 0x0004：CCCD 通知开关（UUID 0x2902）

在上篇第二节中，我们提到手机开启心率实时订阅的业务动作是：<strong>向 Handle 0x0004 写入 0x0001</strong>。

当手机操作系统把这个请求推向空口时，在物理天线上捕获到的这一帧链路层载荷（LL Payload），在内存中呈现为完全连续的一行十六进制裸字节：

05 00 04 00 12 04 00 01 00

把这 9 个字节切开，分层封装的静态结构一目了然：


<table>
<thead>
<tr><th colspan="2">L2CAP Header (4 字节)</th><th colspan="3">ATT PDU Payload (5 字节)</th></tr>
<tr><th>Length (2B)</th><th>Channel ID / CID (2B)</th><th>Opcode (1B)</th><th>Handle (2B)</th><th>Value (2B)</th></tr>
</thead>
<tbody>
<tr><td><strong>05 00</strong></td><td><strong>04 00</strong></td><td><strong>12</strong></td><td><strong>04 00</strong></td><td><strong>01 00</strong></td></tr>
<tr><td>声明后续载荷总长 5 字节</td><td>固定通道 0x0004（绑定 ATT）</td><td>0x12（ATT_WRITE_REQ）</td><td>操作目标行号 Handle 0x0004</td><td>写入的数值 0x0001</td></tr>
</tbody>
</table>

整个封装流程在代码与协议栈内部是这样发生的：

1. <strong>GATT 下达意图</strong>：业务层要求开启订阅。GATT 定位到 CCCD 位于第 4 行，确定写入值 0x0001。

2. <strong>ATT 组装指令（加 ATT 头）</strong>：生成 5 字节的 ATT PDU（12 04 00 01 00）。到这一步，它只描述了“对哪一行做什么操作”，根本不管这包数据该怎么发到天线上。

3. <strong>L2CAP 压入路由头（加 L2CAP 头）</strong>：L2CAP 在这 5 字节前强行插入 4 个字节——写入总长度 0x0005，写入通道号 0x0004。整个数据块变为 9 字节。

4. <strong>Link Layer 组帧发射</strong>：链路层给这 9 字节打上硬件前导码、LL Header（包含 SN/NESN 确认号）、CRC 校验码，经由 2.4GHz 射频天线打出。


## 二、 为什么不能跳过 L2CAP？两大工程死结

看清了层层加头的过程，再反推那个核心问题：<strong>为什么链路层不能直接拿 ATT 报文发射？</strong>


### 1. 多路复用死结：ATT 报文没有“协议类型”字段

在 BLE 连接建立后，两台设备之间的<strong>物理射频信道只有一条</strong>。但在这根物理管子里，并行的协议不仅有 ATT：

- <strong>ATT（属性协议）</strong>：传输应用层业务读写；

- <strong>SMP（安全管理协议）</strong>：传输配对请求、密钥协商、链路加密指令；

- <strong>L2CAP Signaling（控制信令）</strong>：传输连接参数更新（如从机请求更改连接间隔 Connection Interval）；

- <strong>LE CoC（面向连接通道）</strong>：传输大文件 OTA、高吞吐私有数据流。

仔细审视上篇第七节介绍的 ATT PDU 结构：它仅由 Opcode (1B) + Handle (2B) + Value 构成。

<strong>ATT 报文内部没有任何一个字节能表示“我是什么协议”。</strong>

如果没有 L2CAP 头部的 CID：

[从机天线接收到裸数据] ──► 12 04 00 01 00
此时固件协议栈彻底瘫痪：
这到底是业务层的 ATT 写入请求？
还是配对过程中的 SMP 密文交换？
还是链路控制层的参数协商指令？

正是因为 L2CAP 在最前端打上了 <strong>CID = 0x0004</strong>，接收端固件在剥离链路层报头后，第一眼读取第 3~4 字节，就能执行硬件级的协议分流：


| L2CAP CID | 归属协议 / 通道 | 协议栈内部去向 |
| --- | --- | --- |
| 0x0004 | <strong>ATT (Attribute Protocol)</strong> | 直接交由 <strong>ATT 状态机</strong>处理表格 Handle 读写 |
| 0x0006 | <strong>SMP (Security Manager)</strong> | 交由<strong>安全加密引擎</strong>，执行配对、绑定与密钥分发 |
| 0x0005 | <strong>LE Signaling Channel</strong> | 交由<strong>链路调度模块</strong>，解析并更新空口连接参数 |
| 动态 CID (0x0040~0x007F) | <strong>LE CoC (Channel-Oriented)</strong> | 绕开属性表，直接送入<strong>私有高速数据流缓冲区</strong> |

<strong>ATT 只是挂靠在 L2CAP 0x0004 专用通道下的一个业务应用，L2CAP 才是支撑单条物理链路承载多业务并发的多路分拣器。</strong>


### 2. 报文尺寸矛盾：解决上篇留下的 MTU 与 DLE 悬念

在上篇第五节中，我们探讨了传输瓶颈与 ATT MTU 的协商（例如扩充至 247 字节），并提出了 MTU 与 Data Length（DLE）的关系。这两个概念之所以同时存在，根源就在于 L2CAP 的<strong>分段与重组（Segmentation and Reassembly, SAR）</strong>机制。


| 对比维度 | 上层 ATT 的世界 | 底层 Link Layer 的世界 |
| --- | --- | --- |
| <strong>关注的度量衡</strong> | <strong>ATT MTU</strong>（业务单包最大大小） | <strong>Data Length (DLE)</strong>（射频单帧物理承载力） |
| <strong>典型尺寸上限</strong> | 双方可协商为 <strong>247 字节</strong>乃至 512 字节 | 经典 BLE 4.0/4.1 硬件限制单帧仅 <strong>27 字节</strong> |

如果让 ATT 直接面对物理射频：当应用层一次性 Push 一个 150 字节的传感器波形时，ATT 必须在内部自己写一套代码，把它拆解成 6 个带序号的射频小碎片，自己跟踪谁丢包、谁需要重传。这样一来，应用协议直接被硬件物理帧规格绑死。

<strong>L2CAP 在中间切断了这种强耦合：</strong>

1. <strong>发送端封装与分片：ATT 把协商好的 247 字节 ATT PDU 交给 L2CAP；L2CAP 在最前端写上 Length = 243（不含自身 4 字节）与 CID = 0x0004，形成完整的 L2CAP message。若它超过当前 Link Layer 可承载的单帧长度（如 27 字节），随后由 Link Layer / Controller 将该 message 切成多个空口 LL Data PDU 发送。</strong>

2. <strong>接收端重组（Reassembly）：接收端的 Link Layer / Controller 依据 LL Header 中的 LLID 识别起始帧与后续帧，重组出完整的 L2CAP message；L2CAP 再依据自身 Header 的 Length 与 CID 确认边界和去向，并把其中完整的 247 字节 ATT PDU 一次性交给 ATT。</strong>

上层的 ATT 和 GATT 完全感知不到物理空口到底切了多少刀，它们眼中的通信始终是端到端的完整大包。

<strong>再次回看上篇黄金组合的数学闭环：</strong>

247 字节 (ATT MTU) + 4 字节 (L2CAP Header) = 251 字节 (Link Layer 最大 DLE)

当 BLE 4.2+ 双方同时握手使能了 DLE=251 与 ATT MTU=247 时，L2CAP 恰好不需要进行任何物理切片，一枪射频脉冲将整包完整打出，这就是 BLE 吞吐量能从几 KB/s 跃升至几十 KB/s 的核心物理秘密。


## 三、 贯通全景：三层协议的最终定位

将两篇内容串联起来，整个 BLE 数据栈的层次分工形成了坚固的技术闭环：


| 层级 | 协议全称 | 核心管辖范围 | 报文物理表现 |
| --- | --- | --- | --- |
| <strong>GATT</strong> | Generic Attribute Profile | <strong>管业务语义与数据组织</strong>：定义 Service、Characteristic，赋予字节医学、工程含义。 | <strong>空中无独立报文</strong>；纯粹是应用层规范与逻辑状态机。 |
| <strong>ATT</strong> | Attribute Protocol | <strong>管属性指令与动作</strong>：针对指定 Handle，生成 Read、Write、Notify 等二进制 PDU。 | 表现为 [Opcode] + [Handle] + [Value]，构成了 L2CAP 内部的有效载荷。 |
| <strong>L2CAP</strong> | Logical Link Control & Adaptation | <strong>管通道寻址与分段重组</strong>：提供 CID 多路复用，屏蔽射频硬件单包限制。 | 在最外侧压入 [Length: 2B] + [CID: 2B]，充当底层单物理链路的路由与封装基座。 |

<strong>结论：</strong>

GATT 决定“读写什么”；ATT 决定“怎么操作那一行”；L2CAP 则负责“打上通道封条（CID 0x0004），让这条 ATT 指令在链路中有身份、有边界、有明确去向”。至于这条完整 L2CAP message 是否需要切成多段空口帧，由底层 Link Layer / Controller 按当前单帧承载能力处理。

没有 L2CAP，ATT 既无法在单根天线上与配对信令（SMP）共存，也无法越过底层硬件仅几十字节的射频载荷鸿沟。
