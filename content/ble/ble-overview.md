---
title: "BLE 入门：广播、连接与 GATT"
date: 2026-09-12
draft: false
summary: "用一篇文章建立 BLE 工作流程的基本框架。"
---

BLE（Bluetooth Low Energy，蓝牙低功耗）常见于穿戴设备、传感器与智能家居。理解它可以从三个阶段开始。

## 1. 广播

外设（Peripheral）周期性发送广播包，让附近的中心设备（Central）发现自己。广播包通常携带设备名称、服务 UUID 和厂商数据。

## 2. 连接

中心设备扫描到目标后发起连接。连接建立后，双方按照连接间隔交换数据；连接间隔越短，响应越快，但功耗通常越高。

## 3. GATT 数据交换

连接后的数据组织采用 GATT：服务（Service）包含特征值（Characteristic），特征值可供读取、写入或订阅通知（Notification）。

最常见的数据上行方式是：手机订阅某个特征值的 Notification，设备有新数据时主动通知手机。
