---
title: "BLE 无感回连的两个小技巧：Android HID 与 iOS iBeacon"
date: 2026-09-14T23:23:39+08:00
draft: false
summary: "BLE 无感回连的本质，是 App 不活跃时仍能找到系统托管的连接入口。Android 靠 HID Host 主动回连，iOS 靠 Beacon 区域事件唤醒 App。两个思路及落地细节。"
---

BLE 所谓“无感回连”，本质上是在 App 不活跃时，仍然找到一个由系统托管的连接入口。下面分享两个常见思路：Android 把外设作为标准 BLE HID 设备交给系统管理；iOS 借助 Beacon 区域事件唤醒 App，再由 App 建立连接。

![](/images/ble/ble-auto-reconnect-fig1.png)

图 1 两种无感回连路径：Android 由系统回连，iOS 先唤醒 App


## 技巧一：Android 挂载 HID 服务，由系统主动回连


### 1. 为什么 Android 会主动回连

Android 看到已经配对、并且提供标准 HID 服务的蓝牙设备，会把它当作键盘、鼠标这类系统外设来管理。配对时，设备身份和密钥已经保存在系统里；设备下次重新发出可连接广播，系统认出它后就会主动连接。因此，App 没有运行，回连仍然可以发生。

![](/images/ble/ble-auto-reconnect-fig2.png)

图 2 Android：由系统 HID Host 主动完成回连


### 2. 外设端只抓住两件事

在完整 GATT 表里，真正影响系统 HID 回连的是 HIDS（0x1812）及其特征；自定义业务 Service 仍然只是数据通道。


| <strong>模块</strong> | <strong>UUID</strong> | <strong>关键项</strong> | <strong>实际作用</strong> |
| --- | --- | --- | --- |
| HID Service | 0x1812 | Report Map、Report + CCCD、Protocol Mode、HID Information、Control Point | 让 Android 按 HOGP 识别设备，并由系统 HID Host 管理连接。 |
| DIS / BAS | 0x180A / 0x180F | PnP ID、Battery Level + CCCD | 补充设备信息和常见外设能力，可按兼容性需要提供。 |
| 业务 Service | 自定义 UUID | RX：Write；TX：Notify + CCCD | 继续承载业务数据，不负责让系统识别为 HID。 |

<strong>最小 GATT 骨架（伪代码）</strong>


```
HIDS 0x1812
  Report Map         0x2A4B  Read
  Report             0x2A4D  Read | Notify + CCCD 0x2902
  Protocol Mode      0x2A4E  Read | Write Without Response
  HID Information   0x2A4A  Read
  HID Control Point  0x2A4C  Write Without Response
```

最小验证：首次配对成功后结束 App，让外设断开再重新发送可连接广播。若 App 未启动时外设仍收到手机的连接请求，连接发起者就是系统 HID Host。


### 3. 使用时需要注意

- <strong>身份要保持一致：</strong>首次配对后，外设应保留 Bond 数据；设备地址或 IRK 变化、密钥丢失，都会让系统把它当成新设备。

- <strong>广播必须可连接：</strong>App 退出后，外设仍要进入可连接广播状态，系统 HID Host 才有机会发起 ACL 连接。

- <strong>服务表变更要处理缓存：</strong>修改 Report Map 或特征结构后，应发送 Service Changed；调试阶段也可清除手机与外设两端的旧 Bond 后重测。


## 技巧二：iOS 用 Beacon 区域事件唤醒 App


### 1. 为什么 iOS 能唤醒后回连

App 先把目标 Beacon 登记给 iOS。之后即使 App 不在前台，系统也会继续留意这个 Beacon；手机从范围外进入范围内时，iOS 会给 App 一次后台运行机会。App 醒来后再扫描设备，并发起普通的 GATT 连接。所以，Beacon 解决的是“怎么把 App 叫起来”，连接本身还是由 App 完成。

![](/images/ble/ble-auto-reconnect-fig3.png)

图 3 iOS：Beacon 触发与可连接入口分工


### 2. Beacon 触发与连接入口

Beacon 广播负责触发区域事件。App 获得后台执行机会后，再通过可连接广播发现外设并建立 GATT 连接。


| <strong>阶段</strong> | <strong>iOS API</strong> | <strong>外设侧信号</strong> | <strong>可观察结果</strong> |
| --- | --- | --- | --- |
| 区域监听 | startMonitoring(for:) | iBeacon 标识：UUID / Major / Minor | 进入区域时收到 didEnterRegion |
| 扫描与连接 | scanForPeripherals → connect | 可连接广播，并携带用于过滤的 Service UUID | 依次看到 didDiscover、didConnect |

<strong>App 侧调用链（Swift 伪代码）</strong>


```
locationManager.requestAlwaysAuthorization()
let region = CLBeaconRegion(uuid: beaconUUID, identifier: "nearby-device")
locationManager.startMonitoring(for: region)

func locationManager(... didEnterRegion ...) {
    central.scanForPeripherals(withServices: [serviceUUID])
}

func centralManager(... didDiscover peripheral ...) {
    central.stopScan()
    central.connect(peripheral)
}
```

定位方法：分别记录 didEnterRegion、didDiscover 和 didConnect 的时间点。没有 didEnterRegion 是区域触发问题；有 didEnterRegion 没有 didDiscover 是广播过滤或扫描问题；最后再检查连接和认证。


### 3. 落地与调试细节

- 后台唤醒后应立即恢复蓝牙状态、发现目标并发起连接，避免执行窗口结束前流程还停留在扫描或初始化阶段。

- <strong>后台能力要配齐：</strong>区域监听需要位置授权；唤醒后若继续用 CoreBluetooth 扫描和连接，还要启用 bluetooth-central 后台模式，并确认 Background App Refresh 可用。

- <strong>它监听的是“状态变化”：</strong>区域进入或离开才是主要触发点，并不是手机一直停在附近就不断回调。调试时应设计明确的离开、等待与重新进入步骤。


## 两个技巧的差异


| <strong>对比维度</strong> | <strong>Android：HID 回连</strong> | <strong>iOS：Beacon 区域唤醒</strong> |
| --- | --- | --- |
| 系统入口 | HID Host / HOGP | Core Location 区域监听 |
| 谁发起连接 | 系统 HID Host | App 被唤醒后通过 CoreBluetooth 发起 |
| 配对关系 | 首次通常需要配对 | Beacon 触发本身不要求配对；后续 GATT 取决于安全配置 |
| 关键限制 | 系统实现、机型差异、旧配对状态 | 位置授权、区域状态变化、后台执行时机 |
