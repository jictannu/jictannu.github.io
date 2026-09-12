---
title: "BLE 连接全流程：从广播、扫描到建立连接"
date: 2026-09-12
draft: false
summary: "用 HCI 指令串起外设广播、中心扫描与连接建立，并总结最容易踩到的工程细节。"
---

BLE 的一次连接可以看作三个连续阶段：外设先让自己被发现，中心设备扫描并筛选目标，随后在极短的时序窗口中建立连接。本文以传统 LE 广播和 HCI 指令为主线，梳理这条路径。

![BLE 广播、扫描与连接的总流程](/images/ble/connection-flow.svg)

## 先认识两个角色

- **外设（Peripheral）**：负责广播、被发现；例如传感器、手环或标签。
- **中心设备（Central）**：负责扫描、选择目标并发起连接；例如手机或网关。

连接后，双方使用 Connection Handle 识别该连接，后续 GATT 读写和通知都会基于它进行。

## 1. 外设：配置并启动广播

传统广播的启动按三步走：先配置参数，再写入广播数据，最后 Enable。

![广播状态与可更新内容](/images/ble/advertising-state.svg)

### 配置广播参数

使用 `HCI_LE_Set_Advertising_Parameters`。最需要关注的是：

- **Advertising Interval Min / Max**：单位为 0.625 ms，常规范围 20 ms 到 10.24 s。工程上常将两者设为相同，以获得较确定的时延与功耗表现。
- **Advertising Delay**：控制器会在每个广播事件之间额外加入 0–10 ms 的伪随机延迟，降低多个设备同时占用广播信道时的碰撞概率。
- **Advertising Type**：`ADV_IND` 是最常见的可连接、可扫描、非定向广播；`ADV_NONCONN_IND` 不可连接也不可扫描，常用于 Beacon；定向广播只面向指定对端。
- **Channel Map**：常规场景使用 37、38、39 三个广播信道。
- **Filter Policy**：可限制哪些设备能扫描或连接，例如仅允许白名单中的设备。

高占空比定向广播 `ADV_DIRECT_IND` 会忽略普通广播间隔，以很高频率发送，最长持续 1.28 秒；超时仍未连上时，控制器会自动关闭广播。

### 写入广播数据与扫描响应

使用 `HCI_LE_Set_Advertising_Data` 写广播数据，使用 `HCI_LE_Set_Scan_Response_Data` 写扫描响应数据。传统广播和扫描响应各自的有效载荷上限是 31 字节，内容采用 LTV（Length-Type-Value）格式。

### Enable 广播，以及运行时修改规则

`HCI_LE_Set_Advertising_Enable` 中，`0x01` 开启、`0x00` 关闭。

这里有一个常见坑：**广播参数不能在运行中直接修改**。需先 Disable，再重新配置，否则通常会得到 `Command Disallowed (0x0C)`。反过来，名称或传感器值等**广播数据通常可热更新**，新内容会在下一个广播事件生效。

## 2. 中心设备：扫描并接收广播报告

扫描同样分为配置与 Enable 两步。

### 配置扫描

`HCI_LE_Set_Scan_Parameters` 的关键项：

- **LE Scan Type**：`0x00` 为被动扫描，只接收广播；`0x01` 为主动扫描，会发出扫描请求以获得扫描响应。
- **LE Scan Interval / Window**：单位 0.625 ms，范围 2.5 ms 到 10.24 s。Window 不应大于 Interval；Window 越接近 Interval，发现设备越快，但接收功耗越高。
- **Scanning Filter Policy**：可只接收白名单设备的广播与扫描响应。

`HCI_LE_Set_Scan_Enable` 用来开关扫描。`Filter_Duplicates` 决定是否过滤重复报告：短期搜寻目标时建议开启，避免广播刷屏；需要持续观测 Beacon 或广播数据变化时则关闭。

控制器通过 `HCI_LE_Advertising_Report` 上报结果。应用层通常最关心 `Event_Type`、地址类型与地址、数据及 `RSSI`；一次事件中可能会批量携带多个报告。

## 3. 发起连接：从发现目标到 Connection Handle

确认目标后，中心使用 `HCI_LE_Create_Connection` 发起连接。底层在听到目标广播包后，需要在固定的 **T_IFS = 150 μs** 时间内回送连接请求，并切换到数据信道。

![建立连接的关键时序和参数关系](/images/ble/connection-timing.svg)

连接参数中最重要的三项是：

| 参数 | 含义 | 工程影响 |
| --- | --- | --- |
| Connection Interval | 两次连接事件之间的间隔，单位 1.25 ms，范围 7.5 ms–4 s | 越短响应越快、功耗通常越高 |
| Slave Latency | 从机无数据时允许连续跳过的连接事件数 | 用延迟换取省电 |
| Supervision Timeout | 多久未收到有效通信后断开，单位 10 ms | 必须覆盖最长的有效通信间隔 |

其中 Timeout 必须满足：

```text
Supervision Timeout > 2 × (1 + Slave Latency) × Connection Interval Max
```

## 4. 工程上最容易漏掉的四件事

### 连接发起要有上层超时

`HCI_LE_Create_Connection` 在底层没有自动超时机制。目标设备关机或离开范围时，发起端可能长期停留在连接发起状态。建议上层设置定时器（例如 5 秒），超时后下发 `HCI_LE_Create_Connection_Cancel`。

### RPA 不是万能的

RPA（可解析私有地址）通过周期性变化地址保护隐私。定向广播时，控制器会查 Resolving List；如果双方的绑定状态不一致，例如手机已清除配对而设备端仍保留 IRK，手机可能无法解析设备发出的 RPA，最终造成连接超时。超时后应重置相关的广播与绑定状态。

### 连接成功会改变广播状态

可连接广播在连接建立成功后，控制器通常自动关闭广播。若产品需要继续接受其他中心设备连接，应在连接策略中明确是否以及何时重新开启广播。

### 并发连接数也会限制可连接广播

当芯片的并发连接槽位已满，再开启可连接广播可能收到 `Command Disallowed (0x0C)`。不可连接广播是否仍可运行则与具体芯片实现有关，应以平台文档和实测为准。

## 一句话复盘

外设配置并广播，中心配置并扫描；中心从报告中选中目标，在 150 μs 的窗口内发起连接；连接建立后，以 Connection Handle 和连接参数进入稳定的数据通信阶段。把这条主线和几个状态/超时坑点掌握住，BLE 连接问题的定位就有了明确入口。
