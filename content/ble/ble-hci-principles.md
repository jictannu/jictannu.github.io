---
title: "BLE 连接流程与 HCI 协议层原理解析及工程实践"
date: 2026-09-12
draft: false
summary: "不同芯片厂商的 SDK API 名称和封装层各有差异，但底层逻辑均遵循蓝牙核心规范中的 HCI 指令集。本文以 HCI 指令为主线，梳理 BLE 从广播、扫描到连接建立的全过程，并结合工程实战总结关键参数配置与常见踩坑点。"
---

不同芯片厂商的 SDK API 名称和封装层各有差异，但底层逻辑均遵循蓝牙核心规范中的 HCI（Host Controller Interface）指令集。本文以 HCI 指令为主线，梳理 BLE 从广播、扫描到连接建立的全过程，并结合工程实战总结关键参数配置与常见踩坑点。

## 1. 架构视角：为什么以 HCI 为主线？

典型 BLE 协议栈划分为 Host（主机层）与 Controller（控制器层）；两者通过标准 HCI 接口通信。从机（Peripheral）负责广播被发现，主机（Central）负责扫描并发起连接。

![BLE Host、HCI 与 Controller 的分层关系](/images/ble/hci-architecture.svg)

*Host 与 Controller 的分层，两者通过 HCI 指令与事件通信*

无论上层 SDK 封装如何变化，Host 控制 Controller 的操作本质都是下发 HCI Command；Controller 向上层上报状态或数据则通过 HCI Event。理解 HCI 层交互原理，才能跨越不同芯片平台的差异。

## 2. 角色

**从机（Peripheral）**：负责广播，被发现。

**主机（Central）**：负责扫描，发起连接。

## 3. 广播发起过程

根据 HCI，从机发起广播，有配置广播参数、配置广播数据、启动广播，3 个步骤。

![广播配置与启动的 HCI 时序](/images/ble/hci-advertising-sequence.svg)

*配置参数 → 配置数据 → 启动广播，三条 HCI 指令的时序*

### 3.1 配置广播参数：`HCI_LE_Set_Advertising_Parameters`

{{< param name="Advertising_Interval_Min" >}}
广播间隔最小值。单位 0.625 ms，范围 20 ms ~ 10.24 s。

**常见配置：** 和 Max 一致。
{{< /param >}}

{{< param name="Advertising_Interval_Max" >}}
广播间隔最大值。单位 0.625 ms，范围 20 ms ~ 10.24 s。

**常见配置：** 和 Min 一致。
{{< /param >}}

{{< note title="补充说明：" >}}
1. 核心规范要求 Min ≤ Max，且协议建议不应完全相同，以便控制器动态调整、防止射频冲突。
2. 实际项目中出于功耗和时延确定性考量，通常设为相同值。
3. High Duty Cycle 定向广播（`ADV_DIRECT_IND`, 0x01）会忽略配置的 Min 和 Max 间隔参数，底层由控制器以极高频率（通常间隔小于 3.75 ms）连续发送，最大持续时间为 1.28 秒。
4. Low Duty Cycle 定向广播（`ADV_DIRECT_IND`, 0x04）则正常使用配置的 Advertising_Interval_Min 和 Advertising_Interval_Max 间隔参数。
5. 常规广播的最小间隔为 20 ms。
6. 广播延迟（Advertising Delay）：蓝牙控制器会在每次广播事件之间自动引入一个 0 ~ 10 ms 的伪随机延迟，以进一步降低多设备在相同广播信道上的碰撞概率。
{{< /note >}}

{{< param name="Advertising_Type" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | `ADV_IND`，可连接可扫描不定向，最常见的配置。 |
| `0x01` | `ADV_DIRECT_IND`，定向，High Duty Cycle，忽略 Min/Max 间隔（以小于 3.75 ms 高频连续发送），最大持续 1.28 秒。 |
| `0x02` | `ADV_SCAN_IND`，可扫描不可连接不定向。 |
| `0x03` | `ADV_NONCONN_IND`，不可扫描不可连接不定向，常见于 iBeacon。 |
| `0x04` | `ADV_DIRECT_IND`，定向，Low Duty Cycle，正常使用配置的 Min/Max 广播间隔。 |
{{< /param >}}

{{< param name="Own_Address_Type" >}}
广播设备自身使用的蓝牙地址类型：

| 值 | 说明 |
| --- | --- |
| `0x00` | `Public Device Address`，固定公开地址，全球唯一，由 IEEE 注册分配。 |
| `0x01` | `Random Device Address`，静态随机地址（Static Random Address），上电启动后固定不变，用于替代 Public 地址。 |
| `0x02` | `RPA`，可解析私有地址（Resolvable Private Address），周期性动态更换以防止跟踪；无法解析时回退到 Public 地址。 |
| `0x03` | `RPA`，可解析私有地址（Resolvable Private Address），周期性动态更换以防止跟踪；无法解析时回退到 Static 地址。 |
{{< /param >}}

{{< note title="RPA 与回退机制（Fallback）工程实战说明：" >}}
- **核心作用与适用的广播类型：** 防止设备被陌生人定位跟踪，通过周期性动态更换 MAC 地址保护隐私。“回退”机制主要针对定向广播（Direct Advertising）；非定向广播属于“广撒网”，不强求指定设备立刻解密，因此在发广播阶段没有地址回退需求。
- **定向广播触发逻辑：** Controller 在发送定向广播前会先检索 Resolving List（解析列表）。若对端设备不在列表中（即未保存其 IRK 的未知设备），则自动触发回退机制，切换为固定的 Public 或 Random 地址兜底，确保对方能识别基础身份。
- **避坑 / 边缘场景（单边取消配对）：** 若本地 Resolving List 仍保留对端信息，但手机端已清空配对关系，设备端发送定向广播时仍会使用 RPA。手机端因无法解析该 RPA 将直接忽略，导致连接超时。此时需依赖上层应用（APP/Host）检测超时并主动重置广播与绑定状态。
{{< /note >}}

{{< param name="Peer_Address_Type" >}}
对端的地址类型，定向广播才需要。`0x00`：Public；`0x01`：随机。
{{< /param >}}

{{< param name="Peer_Address" >}}
对端的地址，定向广播才需要。
{{< /param >}}

{{< param name="Advertising_Channel_Map" >}}
用哪个信道广播（37、38、39），常规是都使用。
{{< /param >}}

{{< param name="Advertising_Filter_Policy" >}}
广播过滤政策：

| 值 | 说明 |
| --- | --- |
| `0x00` | 所有设备都可以扫描、连接。 |
| `0x01` | 只允许白名单扫描请求。 |
| `0x02` | 只允许白名单连接。 |
| `0x03` | 只允许白名单扫描请求和连接。 |
{{< /param >}}

### 3.2 配置广播数据

{{< param name="HCI_LE_Set_Advertising_Data" >}}
`Advertising_Data_Length`：长度 1 字节，参数范围 0 ~ 31。

`Advertising_Data`：长度 0 ~ 31 字节，格式 LTV。
{{< /param >}}

{{< param name="HCI_LE_Set_Scan_Response_Data" >}}
`Scan_Response_Data_Length`：长度 1 字节，参数范围 0 ~ 31。

`Scan_Response_Data`：长度 0 ~ 31 字节，格式 LTV。
{{< /param >}}

### 3.3 启动广播：`HCI_LE_Set_Advertising_Enable`

{{< param name="Advertising_Enable" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 关闭。 |
| `0x01` | 启动。 |
{{< /param >}}

{{< note title="要点：" >}}
- **广播参数（如间隔、信道等）**：运行中不能直接修改，若强行下发会收到协议返回的 `Command Disallowed (0x0C)` 错误，必须先 Disable 关闭广播再修改。
- **广播数据（如名称、传感器数据）**：支持热更新。在广播开启状态下可直接下发新数据，底层会在下一个广播事件到来时自动生效，无需先关闭广播。
{{< /note >}}

{{< note title="自动关闭广播场景：" >}}
- 连接成功，控制器自动关闭。
- 高占空比定向广播超时：`ADV_DIRECT_IND` (0x01) 在持续 1.28 秒未连上后，控制器会自动超时并关闭广播。
- 并发连接满载限制：当设备的并发连接数达到芯片最大槽位上限时，强行开启可连接广播（如 `ADV_IND`）会收到底层返回的 `Command Disallowed (0x0C)` 错误；而纯单向不可连接广播（如 `ADV_NONCONN_IND`）通常不受此限制，但需要看具体芯片。
{{< /note >}}

![广播状态与可更新内容](/images/ble/advertising-state.svg)

*广播参数需先 Disable 再改，广播数据可热更新*

## 4. 扫描发起过程

根据 HCI，主机发起扫描，有配置扫描参数、启动扫描、广播数据上报，3 个步骤。

![主动扫描与广播上报的 HCI 时序](/images/ble/hci-scanning-sequence.svg)

*配置参数 → 启动扫描 → 广播数据上报，主动扫描发送 SCAN_REQ 取得 SCAN_RSP*

### 4.1 配置扫描参数：`HCI_LE_Set_Scan_Parameters`

{{< param name="LE_Scan_Type" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 被动扫描，不发扫描请求。 |
| `0x01` | 主动扫描，发扫描请求。 |
{{< /param >}}

{{< param name="LE_Scan_Interval" >}}
单位 0.625 ms，范围 2.5 ms ~ 10.24 s。
{{< /param >}}

{{< param name="LE_Scan_Window" >}}
单位 0.625 ms，范围 2.5 ms ~ 10.24 s。
{{< /param >}}

{{< param name="Own_Address_Type" >}}
和广播参数的一样。
{{< /param >}}

{{< param name="Scanning_Filter_Policy" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 接收所有的广播包、扫描响应包。 |
| `0x01` | 只接收白名单的广播包、扫描响应包。 |
| `0x02` | 几乎不用。 |
| `0x03` | 几乎不用。 |
{{< /param >}}

### 4.2 启动扫描：`HCI_LE_Set_Scan_Enable`

{{< param name="LE_Scan_Enable" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 停止。 |
| `0x01` | 打开。 |
{{< /param >}}

{{< param name="Filter_Duplicates" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 关闭重复过滤。 |
| `0x01` | 打开重复过滤。 |
{{< /param >}}

{{< note title="说明：" >}}
- **重复过滤判断标准：** 广播地址 + 广播数据 + 扫描响应数据，3 者完全一样。
- **什么时候会重新上报：** 广播地址、广播数据或扫描响应数据任意一项发生改变；底层缓存满被 LRU 淘汰清理后再次收到（长时间运行下旧设备记录被踢出后重新上报）；扫描重启（手动 Disable 后重新 Enable）。
- **实战场景选择：** 开启 (0x01) 适用于短期搜寻并发起连接（防止重复广播刷屏，省 CPU 资源）；关闭 (0x00) 适用于长期后台扫描、连续数据监控或信标（Beacon/网关）追踪等需要抓取每一包广播的场景。
{{< /note >}}

{{< note title="自动关闭扫描场景：" >}}
成功建立连接。
{{< /note >}}

### 4.3 广播数据上报：`HCI_LE_Advertising_Report`

{{< param >}}
`Subevent_Code`：固定是 0x02，代表是 HCI_LE_Advertising_Report event。

`Num_Reports`：一次上报中的报告数量。底层射频芯片为提升效率，可能将同一瞬间抓到的多个广播包打包在一个 HCI 事件中批量上报。

以上 2 个参数，SDK 是不会上报的。

`Event_Type[i]`：第 i 个广播报告的事件类型（如可连接广播、扫描响应包等）。

`Address_Type[i]` 与 `Address[i]`：第 i 个设备的蓝牙地址类型与 MAC 地址。

`Data_Length[i]` 与 `Data[i]`：第 i 个广播包 / 扫描响应包的有效载荷长度与内容。

`RSSI[i]`：接收信号强度指示（单位 dBm），用于判定设备距离。
{{< /param >}}

## 5. 发起连接过程

核心 HCI 指令是 `HCI_LE_Create_Connection`。

![建立连接与 T_IFS 150 微秒响应](/images/ble/hci-connection-sequence.svg)

*收到广播后，须在 T_IFS = 150 µs 内回 CONNECT_IND 切换到数据信道*

### 5.1 参数

{{< param name="LE_Scan_Interval" >}}
扫描间隔。
{{< /param >}}

{{< param name="LE_Scan_Window" >}}
扫描窗口。
{{< /param >}}

{{< param name="Initiator_Filter_Policy" >}}
| 值 | 说明 |
| --- | --- |
| `0x00` | 连接 `Peer_Address_Type`、`Peer_Address` 指定的设备。 |
| `0x01` | 白名单重连：只要 Filter Accept List（白名单）中的任何设备发广播，底层就自动触发连接，`Peer_Address_Type`、`Peer_Address` 忽略。 |
{{< /param >}}

{{< param name="Peer_Address_Type" >}}
和广播的一样。
{{< /param >}}

{{< param name="Peer_Address" >}}
要连接的设备地址。
{{< /param >}}

{{< param name="Own_Address_Type" >}}
和 Peer_Address_Type 一样。
{{< /param >}}

{{< param name="Connection_Interval_Min" >}}
连接间隔。单位 1.25 ms，范围 7.5 ms ~ 4 s。
{{< /param >}}

{{< param name="Connection_Interval_Max" >}}
连接间隔。单位 1.25 ms，范围 7.5 ms ~ 4 s。
{{< /param >}}

{{< param name="Connection_Latency" >}}
Slave 延迟，允许从机在没有数据要发时，连续跳过 Connection_Latency 次 Connection Event，多数用于省功耗。
{{< /param >}}

{{< param name="Supervision_Timeout" >}}
连接超时时间。单位 10 ms，范围 100 ms ~ 32.0 s。
{{< /param >}}

{{< param name="Min_CE_Length / Max_CE_Length" >}}
连接事件的射频窗口大小，SDK 会搞定，不需要调。
{{< /param >}}

### 5.2 发起连接过程实战笔记

{{< note >}}
- **状态互斥：** 想发起连接，必须先关掉扫描（`Scan_Enable = 0x00`），然后才能去发起连接。
- **抓包瞬间回包：** 听到对方广播包后，要在极短时间窗口（`T_IFS`，固定 150 微秒）内回 `CONNECT_IND` 切到数据信道。
- **永不超时的卡死陷阱：** `HCI_LE_Create_Connection` 在底层没有自动超时机制。对方关机或跑远会导致芯片永远卡死在发起状态。必须在上层写定时器（如 5 秒），超时后主动下发 `HCI_LE_Create_Connection_Cancel` 强行取消。
- **连接成功标志：** 成功后拿到 `Connection_Handle`（连接句柄），后续读写全靠它。
{{< /note >}}

{{< note title="核心参数大白话：" >}}
Connection Interval（间隔）、Slave Latency（潜伏）、Supervision Timeout（监督超时，必须满足 `Timeout > (1 + Latency) × Interval_Max × 2`）。
{{< /note >}}
