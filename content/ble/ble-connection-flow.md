---
title: "BLE 连接过程：广播、扫描与连接的 HCI 参数详解"
date: 2026-09-12
draft: false
summary: "完整保留 HCI 参数、枚举值、边界条件与工程避坑经验的 BLE 连接过程笔记。"
---

本文说明 BLE 的广播、扫描与连接过程。内容以传统 LE HCI 命令为主，保留具体参数、单位、取值及工程实践要点。

![BLE 广播、扫描与连接的总流程](/images/ble/connection-flow.svg)

## 1. 角色

- **从机（Peripheral）**：负责广播、被发现。
- **主机（Central）**：负责扫描，并发起连接。

{{< hci-stage step="01" >}}

## 2. 广播发起过程

根据 HCI，从机发起广播有三个步骤：配置广播参数、配置广播数据、启动广播。

![广播状态与可更新内容](/images/ble/advertising-state.svg)

### 2.1 配置广播参数：`HCI_LE_Set_Advertising_Parameters`

#### `Advertising_Interval_Min`

广播间隔最小值，单位为 0.625 ms，范围 20 ms–10.24 s。

常见配置：与 `Advertising_Interval_Max` 一致。

#### `Advertising_Interval_Max`

广播间隔最大值，单位为 0.625 ms，范围 20 ms–10.24 s。

常见配置：与 `Advertising_Interval_Min` 一致。

**补充说明：**

1. 核心规范要求 Min ≤ Max，且协议建议不应完全相同，以便控制器动态调整、降低射频冲突。
2. 实际项目中出于功耗和时延确定性考量，通常设为相同值。
3. High Duty Cycle 定向广播 `ADV_DIRECT_IND (0x01)` 会忽略配置的 Min/Max 间隔参数；控制器以极高频率（通常小于 3.75 ms）连续发送，最长持续 1.28 秒。
4. Low Duty Cycle 定向广播 `ADV_DIRECT_IND (0x04)` 正常使用 `Advertising_Interval_Min` 与 `Advertising_Interval_Max`。
5. 常规广播的最小间隔为 20 ms。
6. **广播延迟（Advertising Delay）**：控制器会在每次广播事件之间自动加入 0–10 ms 的伪随机延迟，进一步降低多设备在同一广播信道上的碰撞概率。

#### `Advertising_Type`

| 值 | 类型 | 说明 |
| --- | --- | --- |
| `0x00` | `ADV_IND` | 可连接、可扫描、不定向；最常见配置 |
| `0x01` | `ADV_DIRECT_IND` | 定向、高占空比；忽略 Min/Max，以小于 3.75 ms 高频发送，最长 1.28 秒 |
| `0x02` | `ADV_SCAN_IND` | 可扫描、不可连接、不定向 |
| `0x03` | `ADV_NONCONN_IND` | 不可扫描、不可连接、不定向；常见于 iBeacon |
| `0x04` | `ADV_DIRECT_IND` | 定向、低占空比；正常使用 Min/Max 广播间隔 |

#### `Own_Address_Type`

广播设备自身使用的蓝牙地址类型：

| 值 | 地址类型 | 说明 |
| --- | --- | --- |
| `0x00` | Public Device Address | 固定公开地址，全球唯一，由 IEEE 注册分配 |
| `0x01` | Random Device Address | 静态随机地址；设备上电启动后固定不变，用于替代 Public 地址 |
| `0x02` | RPA | 可解析私有地址；周期性更换以防跟踪，无法解析时回退到 Public 地址 |
| `0x03` | RPA | 可解析私有地址；周期性更换以防跟踪，无法解析时回退到 Static 地址 |

**RPA 与回退机制（Fallback）工程实战说明：**

- **核心作用与适用广播类型**：RPA 通过周期性更换 MAC 地址防止陌生人定位追踪。回退机制主要针对定向广播；非定向广播属于“广撒网”，发广播阶段不强求指定设备立即解密，因此没有地址回退需求。
- **定向广播触发逻辑**：Controller 在发送定向广播前会检索 Resolving List。若对端不在列表中，即未保存其 IRK 的未知设备，则自动触发回退，切换为固定的 Public 或 Random 地址兜底，确保对方可识别基础身份。
- **避坑/边缘场景**：若本地 Resolving List 仍保留对端信息，但手机端已清空配对关系，设备端定向广播仍会使用 RPA。手机端无法解析该 RPA 会直接忽略，导致连接超时。需要依赖上层 APP/Host 检测超时，并主动重置广播与绑定状态。

#### `Peer_Address_Type` 与 `Peer_Address`

仅定向广播需要。

- `Peer_Address_Type`：`0x00` 为 Public，`0x01` 为随机地址。
- `Peer_Address`：对端地址。

#### `Advertising_Channel_Map`

选择在哪些信道广播：37、38、39。常规场景使用全部三个信道。

#### `Advertising_Filter_Policy`

| 值 | 过滤策略 |
| --- | --- |
| `0x00` | 所有设备都可以扫描、连接 |
| `0x01` | 只允许白名单扫描请求 |
| `0x02` | 只允许白名单连接 |
| `0x03` | 只允许白名单扫描请求和连接 |

### 2.2 配置广播数据

使用 `HCI_LE_Set_Advertising_Data`：

- `Advertising_Data_Length`：长度为 1 字节，参数范围 0–31。
- `Advertising_Data`：长度 0–31 字节，格式为 LTV。

使用 `HCI_LE_Set_Scan_Response_Data`：

- `Scan_Response_Data_Length`：长度为 1 字节，参数范围 0–31。
- `Scan_Response_Data`：长度 0–31 字节，格式为 LTV。

### 2.3 启动广播：`HCI_LE_Set_Advertising_Enable`

`Advertising_Enable`：

- `0x00`：关闭。
- `0x01`：启动。

**要点：**

- 广播参数（如间隔、信道）运行中不能直接修改。强行下发会收到 `Command Disallowed (0x0C)`；必须先 Disable 广播再修改。
- 广播数据（如名称、传感器数据）支持热更新。广播开启时可直接下发新数据，底层会在下一个广播事件自动生效，无需关闭广播。

**自动关闭广播场景：**

- 连接成功，控制器自动关闭。
- 高占空比定向广播超时：`ADV_DIRECT_IND (0x01)` 持续 1.28 秒未连上后，控制器自动超时关闭。
- 并发连接满载限制：当并发连接数达到芯片最大槽位时，强行开启可连接广播（如 `ADV_IND`）会收到 `Command Disallowed (0x0C)`。纯单向不可连接广播（如 `ADV_NONCONN_IND`）通常不受此限制，但仍需确认具体芯片实现。

{{< /hci-stage >}}

{{< hci-stage step="02" >}}

## 3. 扫描发起过程

根据 HCI，主机发起扫描有三个步骤：配置扫描参数、启动扫描、接收广播数据上报。

### 3.1 配置扫描参数：`HCI_LE_Set_Scan_Parameters`

#### `LE_Scan_Type`

- `0x00`：被动扫描，不发扫描请求。
- `0x01`：主动扫描，发扫描请求。

#### `LE_Scan_Interval` 与 `LE_Scan_Window`

单位均为 0.625 ms，范围 2.5 ms–10.24 s。

#### `Own_Address_Type`

与广播参数中的 `Own_Address_Type` 相同。

#### `Scanning_Filter_Policy`

| 值 | 策略 |
| --- | --- |
| `0x00` | 接收所有广播包、扫描响应包 |
| `0x01` | 只接收白名单广播包、扫描响应包 |
| `0x02` | 几乎不用 |
| `0x03` | 几乎不用 |

### 3.2 启动扫描：`HCI_LE_Set_Scan_Enable`

#### `LE_Scan_Enable`

- `0x00`：停止。
- `0x01`：打开。

#### `Filter_Duplicates`

- `0x00`：关闭重复过滤。
- `0x01`：打开重复过滤。

**重复过滤判断标准**：广播地址、广播数据、扫描响应数据三者完全一致。

**会重新上报的情况：**

- 广播地址、广播数据或扫描响应数据任意一项改变。
- 底层缓存满后被 LRU 淘汰；长时间运行时旧设备记录被踢出，后续再次收到则重新上报。
- 扫描重启，即手动 Disable 后重新 Enable。

**实战场景选择：**

- 开启 `0x01`：适用于短期搜寻并发起连接，防止重复广播刷屏、节省 CPU。
- 关闭 `0x00`：适用于长期后台扫描、连续数据监控，或 Beacon/网关追踪等需要抓取每一个广播包的场景。

扫描成功建立连接后会自动关闭。

### 3.3 广播数据上报：`HCI_LE_Advertising_Report`

- `Subevent_Code`：固定为 `0x02`，代表 `HCI_LE_Advertising_Report` event。
- `Num_Reports`：一次上报中的报告数量。为提升效率，底层射频芯片可能把同一时刻抓到的多个广播包打包到一个 HCI 事件中批量上报。
- 以上两个参数，SDK 不会上报。
- `Event_Type[i]`：第 i 个广播报告的事件类型，例如可连接广播、扫描响应包。
- `Address_Type[i]` 与 `Address[i]`：第 i 个设备的地址类型与 MAC 地址。
- `Data_Length[i]` 与 `Data[i]`：第 i 个广播包/扫描响应包的有效载荷长度与内容。
- `RSSI[i]`：接收信号强度指示，单位 dBm，可用于判断设备距离。

{{< /hci-stage >}}

{{< hci-stage step="03" >}}

## 4. 发起连接过程

核心 HCI 指令为 `HCI_LE_Create_Connection`。

![建立连接的关键时序和参数关系](/images/ble/connection-timing.svg)

### 参数

{{< param name="LE_Scan_Interval / LE_Scan_Window" >}}

扫描间隔与扫描窗口。

{{< /param >}}

{{< param name="Initiator_Filter_Policy" >}}

- `0x00`：连接由 `Peer_Address_Type` 与 `Peer_Address` 指定的设备。
- `0x01`：白名单重连。只要 Filter Accept List 中任一设备广播，底层自动触发连接，忽略 `Peer_Address_Type` 和 `Peer_Address`。

{{< /param >}}

{{< param name="Peer_Address_Type / Peer_Address" >}}

分别为要连接设备的地址类型和地址。

{{< /param >}}

{{< param name="Own_Address_Type" >}}

与 `Peer_Address_Type` 的地址类型说明相同。

{{< /param >}}

{{< param name="Connection_Interval_Min / Connection_Interval_Max" >}}

连接间隔，单位 1.25 ms，范围 7.5 ms–4 s。

{{< /param >}}

{{< param name="Connection_Latency" >}}

从机延迟。允许从机在没有数据时连续跳过 `Connection_Latency` 次 Connection Event，多用于省电。

{{< /param >}}

{{< param name="Supervision_Timeout" >}}

连接超时时间，单位 10 ms，范围 100 ms–32.0 s。

{{< /param >}}

{{< param name="Min_CE_Length / Max_CE_Length" >}}

连接事件的射频窗口大小，SDK 会处理，通常不需要应用调节。

{{< /param >}}

### 4.1 发起连接过程实战笔记

- **状态互斥**：想发起连接，必须先关闭扫描（`Scan_Enable = 0x00`），然后才能发起连接。
- **抓包瞬间回包**：听到对方广播包后，要在极短时间窗口 `T_IFS`（固定 150 μs）内回 `CONNECT_IND`，并切换到数据信道。
- **永不超时的卡死陷阱**：`HCI_LE_Create_Connection` 底层没有自动超时机制。对方关机或跑远会使芯片长期卡在发起状态。必须在上层设置定时器（例如 5 秒），超时后主动下发 `HCI_LE_Create_Connection_Cancel` 强制取消。
- **连接成功标志**：成功后拿到 `Connection_Handle`，后续读写都依赖它。

**核心参数大白话：**

- Connection Interval：间隔。
- Slave Latency：潜伏。
- Supervision Timeout：监督超时，必须满足：

```text
Timeout > (1 + Latency) × Interval_Max × 2
```

{{< /hci-stage >}}
