---
title: "BLE 连接流程与 HCI 协议层原理解析及工程实践"
date: 2026-09-12
draft: false
summary: "以 HCI 指令为主线，梳理 BLE 广播、扫描、连接以及工程实践。"
---

摘要：不同芯片厂商的 SDK API 名称和封装层各有差异，但底层逻辑均遵循蓝牙核心规范中的 HCI（Host Controller Interface）指令集。本文以 HCI 指令为主线，梳理 BLE 从广播、扫描到连接建立的全过程，并结合工程实战总结关键参数配置与常见踩坑点。

## 1. 架构视角：为什么以 HCI 为主线？

典型 BLE 协议栈划分为 Host（主机层）与 Controller（控制器层）；两者通过标准 HCI 接口通信。从机（Peripheral）负责广播被发现，主机（Central）负责扫描并发起连接。

![BLE Host、HCI 与 Controller 的分层关系](/images/ble/hci-architecture.svg)

无论上层 SDK 封装如何变化，Host 控制 Controller 的操作本质都是下发 HCI Command；Controller 向上层上报状态或数据则通过 HCI Event。理解 HCI 层交互原理，才能跨越不同芯片平台的差异。

## 2. 广播发起过程（Advertising）

广播是从机宣告自身存在、提供服务或等待连接的核心机制。根据 HCI，发起广播包含配置广播参数、配置广播数据、启动广播三个步骤。

### 2.1 HCI 交互时序图

![广播配置与启动的 HCI 时序](/images/ble/hci-advertising-sequence.svg)

### 2.2 核心 HCI 指令解析

{{< hci-command name="HCI_LE_Set_Advertising_Parameters" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `Advertising_Interval_Min / Max` | 0.625 ms；20 ms–10.24 s | 广播事件的最小与最大间隔，Min ≤ Max | 常设为相同值以获得确定性；Controller 仍会加入 0–10 ms Advertising Delay |
| `Advertising_Type` | `0x00 ADV_IND` | 可连接、可扫描、不定向 | 最常用通用广播 |
| 〃 | `0x01 ADV_DIRECT_IND` | 高占空比定向广播 | 忽略 Min/Max；间隔小于 3.75 ms，最长 1.28 秒 |
| 〃 | `0x02 ADV_SCAN_IND` | 可扫描、不可连接 | 用扫描响应补充数据 |
| 〃 | `0x03 ADV_NONCONN_IND` | 不可扫描、不可连接 | 常见于 iBeacon |
| 〃 | `0x04 ADV_DIRECT_IND` | 低占空比定向广播 | 正常使用 Min/Max |
| `Own_Address_Type` | `0x00 / 0x01 / 0x02 / 0x03` | Public、Static Random、RPA | RPA 周期性更换以防跟踪；解析失败分别回退 Public 或 Static 地址 |
| `Peer_Address_Type / Peer_Address` | 地址类型 / MAC 地址 | 定向广播的对端身份 | 仅定向广播需要 |
| `Advertising_Channel_Map` | 37 / 38 / 39 | 广播信道掩码 | 通常全选三个信道 |
| `Advertising_Filter_Policy` | `0x00 / 0x01 / 0x02 / 0x03` | 全部允许 / 仅白名单扫描 / 仅白名单连接 / 白名单扫描和连接 | 由白名单策略决定 |

{{< /hci-command >}}

{{< hci-command name="HCI_LE_Set_Advertising_Data / HCI_LE_Set_Scan_Response_Data" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `Advertising_Data / Scan_Response_Data` | 0–31 字节；LTV | 广播载荷与扫描响应载荷 | 广播数据支持热更新，无需关闭广播 |

{{< /hci-command >}}

{{< hci-command name="HCI_LE_Set_Advertising_Enable" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `Advertising_Enable` | `0x01 / 0x00` | 启动 / 关闭广播 | 修改广播参数前必须先 Disable |

{{< /hci-command >}}

### 2.3 工程实践与避坑指南

- **运行中参数修改约束**：广播参数（间隔、信道等）在运行中不能直接修改；强行下发会收到 `Command Disallowed (0x0C)`，必须先 Disable，修改后再 Enable。
- **广播数据热更新**：广播名称或传感器数据支持热更新；开启状态下直接下发，新数据会在下一个广播事件生效。
- **RPA 与定向广播回退**：发送定向广播前，Controller 检索 Resolving List；对端未保存 IRK 时自动回退到固定地址以保障识别。若设备端仍保存对端信息、手机端已单边取消配对，手机无法解析 RPA，需由 Host 检测超时并重置广播和绑定状态。
- **自动关闭与并发限制**：连接成功后 Controller 自动关闭广播；高占空比定向广播 1.28 秒未连接也会关闭。并发连接满载时，开启可连接广播会返回 `0x0C`；不可连接广播通常不受此限制。

## 3. 主机扫描过程（Scanning）

主机通过扫描信道捕获从机广播包。发起扫描包含配置参数、启动扫描、广播数据上报三个步骤。

### 3.1 HCI 交互与数据上报时序图

![主动扫描与广播上报的 HCI 时序](/images/ble/hci-scanning-sequence.svg)

### 3.2 核心 HCI 指令解析

{{< hci-command name="HCI_LE_Set_Scan_Parameters" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `LE_Scan_Type` | `0x00 / 0x01` | 被动扫描 / 主动扫描 | 主动扫描发送 `SCAN_REQ` 取得 `SCAN_RSP` |
| `LE_Scan_Interval / Window` | 0.625 ms；2.5 ms–10.24 s | 扫描间隔与窗口 | 决定监听占空比 |
| `Scanning_Filter_Policy` | `0x00 / 0x01` | 接收全部 / 只接收白名单设备 | 用于控制上报范围 |

{{< /hci-command >}}

{{< hci-command name="HCI_LE_Set_Scan_Enable" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `LE_Scan_Enable` | `0x01 / 0x00` | 打开 / 停止扫描 | 建立连接后自动关闭 |
| `Filter_Duplicates` | `0x01 / 0x00` | 打开 / 关闭重复过滤 | 短期搜索开；长期监控或 Beacon 追踪关 |

{{< /hci-command >}}

{{< hci-command name="HCI_LE_Advertising_Report" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `Subevent_Code` | `0x02` | 广播数据上报事件 | Controller 捕获广播后上报 |
| `Event_Type / Address / Data / RSSI` | 广播属性、MAC、载荷、dBm | 单个广播报告内容 | Host 用于发现、筛选与距离判断 |

{{< /hci-command >}}

### 3.3 工程实践与避坑指南

重复过滤以广播地址、广播数据、扫描响应数据三者完全一致为判断条件。地址或数据变化、缓存满后被 LRU 淘汰、或 Disable 后重新 Enable，都会重新上报。短期搜索并连接时建议开启过滤以减少 CPU 消耗；长期后台监控或 Beacon/网关追踪时建议关闭，以取得每一个广播包。连接建立成功后扫描会自动关闭。

## 4. 连接发起与建立过程（Connecting）

发起连接是从通用广播信道切换到一对一数据信道的关键环节，核心 HCI 指令为 `HCI_LE_Create_Connection`。

### 4.1 连接发起与 `T_IFS` 极速响应时序图

![建立连接与 T_IFS 150 微秒响应](/images/ble/hci-connection-sequence.svg)

### 4.2 核心 HCI 指令与参数解析

{{< hci-command name="HCI_LE_Create_Connection" >}}

| 参数 | 取值 / 范围 | 协议含义 | 工程备注 |
| --- | --- | --- | --- |
| `LE_Scan_Interval / Window` | 控制器扫描参数 | 连接前监听目标广播 | 与扫描阶段语义一致 |
| `Initiator_Filter_Policy` | `0x00 / 0x01` | 指定地址连接 / 白名单重连 | `0x01` 忽略指定 Peer 地址 |
| `Connection_Interval_Min / Max` | 1.25 ms；7.5 ms–4.0 s | 连接事件间隔范围 | 影响时延与功耗 |
| `Connection_Latency` | 非负整数 | 无数据时跳过连接事件 | 用于降低从机功耗 |
| `Supervision_Timeout` | 10 ms；100 ms–32.0 s | 连接监督超时 | 必须满足规范不等式 |

{{< /hci-command >}}

### 4.3 工程实践与避坑指南

- 发起连接前必须先关闭扫描（`Scan Enable = 0x00`）。
- Controller 听到目标广播后，必须在固定 `T_IFS = 150 µs` 的帧间间隔内回复 `CONNECT_IND`；这是射频硬件完成的过程，Host 无权干预。
- `HCI_LE_Create_Connection` 底层没有自动超时机制。目标关机或离开范围时，Controller 会持续处于发起状态；Host 必须维护定时器（例如 5 秒），超时后下发 `HCI_LE_Create_Connection_Cancel`。
- 成功后 Host 获得 `Connection_Handle`，后续读写依赖它。连接参数必须满足：

```text
Supervision_Timeout > (1 + Slave_Latency) × Connection_Interval_Max × 2
```

## 5. BLE HCI 关键指令与事件速查表

| HCI 指令 / 事件 | 主要功能 | 工程注意事项 |
| --- | --- | --- |
| `HCI_LE_Set_Advertising_Parameters` | 配置广播间隔、类型、地址 | 广播开启时不可修改，需先 Disable |
| `HCI_LE_Set_Advertising_Data` | 设置 31 字节 LTV 广播载荷 | 支持热更新，无需关闭广播 |
| `HCI_LE_Set_Advertising_Enable` | 控制广播开关 | 满载时开启可连接广播会报 `0x0C` |
| `HCI_LE_Set_Scan_Parameters` | 配置扫描类型、窗口与间隔 | 主动扫描会发送 `SCAN_REQ` |
| `HCI_LE_Set_Scan_Enable` | 控制扫描与重复过滤 | 开关过滤影响 CPU 与上报频率 |
| `HCI_LE_Advertising_Report` | 上报捕获到的广播包 | 包含 MAC、Data 与 RSSI |
| `HCI_LE_Create_Connection` | 发起连接并指定目标参数 | 必须先关扫描；底层无自动超时 |
| `HCI_LE_Create_Connection_Cancel` | 取消正在发起的连接 | 用于超时防护 |
