---
title: "从“Excel 大表”到“空口发包”：彻底搞懂 GATT 与 ATT 的分工真相"
date: 2026-09-13T21:00:00+08:00
draft: false
summary: "GATT 管的是“业务规约与表格设计”，ATT 管的是“空口发包与搬运”。一文拆解两者的职责切分、协议映射，以及 MTU / DLE / Long Write 的传输机制。"
---

在上一篇中，我们把 BLE 的数据存储拆解成了一张<strong>“Excel 属性大表”</strong>：最底层是每一行 Attribute，往上组合成 Characteristic，再组合成 Service。这个模型解决了<strong>“数据在设备里怎么存、怎么按业务划分”</strong>的问题。

但很多工程师在实际调试或抓包时会产生新的疑问：<strong>既然已经有了 GATT 这张大表，为什么协议栈底下还要叠一个 ATT？它们俩到底是什么关系？</strong>

一句话点破本质：

<strong>GATT 管的是“业务规约与表格设计”，负责让数据有意义；</strong>

<strong>ATT 管的是“空口发包与搬运”，负责把这行单元格里的裸字节准确发出去。</strong>


## 一、 概念联动：GATT 与 ATT 的职责切分

如果把整个 BLE 通信比作一次在线填表协作：


| <strong>对比维度</strong> | <strong>GATT（Generic Attribute Profile）</strong> | <strong>ATT（Attribute Protocol）</strong> |
| --- | --- | --- |
| <strong>在上一篇中的定位</strong> | <strong>“表格设计规范”</strong>：规定 Service 包含哪些 Characteristic，规定哪行是声明、哪行是数据、哪行是开关。 | <strong>“表格数据载体与搬运引擎”</strong>：提供最底层的四个要素（Handle, UUID, Value, Permissions），并负责按行读写传输。 |
| <strong>核心管辖范围</strong> | <strong>管业务层（应用语义）</strong>：<br>定义业务场景，比如“心率服务（0x180D）”、“当前心率特征（0x2A37）”。让手机 App 能看懂“75 bpm”代表心跳。 | <strong>管传输层（空口发包）</strong>：<br>定义无线电空中飞舞的原始 PDU 二进制报文。它不关心业务，只按 Handle 定位行号，搬运裸字节。 |
| <strong>它眼中看到的世界</strong> | 具有层次感的功能模块树：Profile → Service → Characteristic → Descriptor。 | 极其扁平的线性内存行，只有 0x0001 ~ 0xFFFF 的 Handle，互不嵌套。 |
| <strong>空中是否有独立报文？</strong> | <strong>没有。空口上并不存在独立的“GATT 协议包”；GATT 是业务规范与状态机，其操作通过 ATT PDU 落到传输层。</strong> | <strong>全部都是。</strong>空中跑的 100% 是 ATT 协议数据单元（如 Read Request, Handle Value Notification）。 |


## 二、 协议映射：GATT 的业务意图，如何翻译成 ATT 的空口动作？

延续上一篇心率手环的 Attribute 大表：

- Handle 0x0001：主服务声明（UUID 0x2800，Value 0x180D 心率服务）

- Handle 0x0002：心率测量特征声明（UUID 0x2803）

- Handle 0x0003：心率测量具体数值（UUID 0x2A37，当前值 0x00 0x4B，即 75 bpm）

- Handle 0x0004：CCCD 通知开关（UUID 0x2902）

- Handle 0x0006：传感器佩戴位置（UUID 0x2A38，当前值 0x01 胸部）

当手机 App 在业务层执行操作时，GATT 只是向底层下达业务目标，而在空口中，完全由 ATT 执行发包搬运：


| <strong>GATT 业务意图（手机 App 视角）</strong> | <strong>ATT 空口对应 PDU</strong> | <strong>表格上的物理动作</strong> |
| --- | --- | --- |
| <strong>1. 发现设备上的心率服务</strong> | Read By Group Type Request<br>(范围 0x0001~0xFFFF, 查找 0x2800) | 在整张表中按列筛选：找出所有 UUID 为 0x2800 的行，从而圈定心率服务位于第 1 行到第 6 行。 |
| <strong>2. 读取传感器佩戴位置</strong> | Read Request (Handle = 0x0006)<br>返回 Read Response (Value = 0x01) | 直接定位第 6 行，抓取 Value 单元格里的 1 个字节（0x01）发给手机。 |
| <strong>3. 开启心率测量实时订阅</strong> | Write Request (Handle = 0x0004, Value = 0x0001)<br>返回 Write Response | 找到第 4 行（CCCD 开关行），将单元格内容改写为 0x0001。 |
| <strong>4. 手环产生新心率，推送到手机</strong> | Handle Value Notification<br>(Handle = 0x0003, Value = 0x00 0x4B) | 手环传感器更新第 3 行，ATT 搬运工直接抓取该行内容并以 Notification 单向推给手机；它不等待 ATT 层确认（链路层仍可能进行确认与重传）。 |


## 三、 深度辨析：GATT Properties 与 ATT Permissions 的区别

初学 BLE 时，最容易把“特征属性（GATT Properties）”与“访问权限（ATT Permissions）”搞混，因为它们看起来都在控制读写。两者的本质可以用一句话区分：

<strong>GATT Properties 是写在门上的“公开业务说明书”（给手机看的）；</strong>

<strong>ATT Permissions 是装在门里的“物理防盗机械锁”（由设备端本地协议栈强制把关的）。</strong>


| <strong>对比维度</strong> | <strong>GATT Characteristic Properties（特征属性）</strong> | <strong>ATT Attribute Permissions（属性权限）</strong> |
| --- | --- | --- |
| <strong>所属层级</strong> | GATT 层（应用业务规范） | ATT 层与安全管理层（SMP）（协议传输与安全策略） |
| <strong>存放位置</strong> | 公开存放在<strong>特征声明行（UUID 0x2803）的 Value 字段第 1 字节</strong>。 | 只保存在<strong>设备本地固件/协议栈内存中</strong>，不在大表 Value 里存储。 |
| <strong>是否向空口发送？</strong> | <strong>会发送</strong>。服务发现时，作为声明元数据明文传输给手机。 | <strong>绝不向空口发送。客户端无法从 ATT PDU 直接得知设备本地的权限定义。</strong> |
| <strong>定义的核心内容</strong> | <strong>业务功能支持</strong>：<br>• Read（可读）<br>• Write / Write Without Response（可写/无应答写）<br>• Notify / Indicate（支持订阅推送） | <strong>安全访问门槛</strong>：<br>• 读/写条件<br>• 是否要求加密链路（Encryption）<br>• 是否要求认证/配对绑定（Authentication）<br>• 是否要求授权（Authorization） |
| <strong>作用对象</strong> | 整个 <strong>Characteristic</strong> 业务组件。 | 具体的 <strong>某一行 Attribute</strong>（如专门对 Value 行或 CCCD 行设限）。 |
| <strong>违规行为与执行者</strong> | <strong>软性提示</strong>。供手机操作系统或 App 决定展示什么按钮、调用什么 API。手机即使无视它强行发包，在这一层也不会被拦截。 | <strong>硬性拦截</strong>。设备端 ATT 协议栈在收到报文的第一时间校验。若未达到安全条件，直接中断并返回错误码（如 0x05 Insufficient Authentication）。 |


### 两个关键工程实战避坑点：

- <strong>配对绑定弹窗的来源</strong>：当手机未配对直接读取一个受保护特征时，GATT 声明允许 Read，但底层的 ATT Permissions 要求加密认证。设备 ATT 层返回错误码 0x05 或 0x0F，手机系统（iOS/Android）收到该错误后才会自动弹出系统级“蓝牙配对请求”窗口。

- <strong>Notify 不是一种 Permission</strong>：很多初学者在固件代码里尝试把某个 Attribute 的 Permission 设为 Notify 会报编译错误。因为 Notify/Indicate 是 GATT 层的业务能力（告知客户端可订阅），空中推送时由设备端主动发起，根本不走客户端对该行的读取鉴权。


## 四、 继承与澄清：为什么要有 Characteristic 声明行？

在上一篇中我们提到：一个 Characteristic 由声明行（0x2803）、数值行、描述符行组成。结合 ATT 的空口机制，这一设计的深层原因变得极为清晰：

- <strong>ATT 只管无脑搬运</strong>：ATT 执行 Read Request 或 Write Request 时，它只看 Handle，压根不知道这行数据是只读的还是可订阅的，更不知道这几个字节是整数还是文本。

- <strong>GATT 必须插一行元数据（0x2803 特征声明）</strong>：这一行的作用是提前告诉客户端：“第 3 行是我的数值，它的 UUID 是 0x2A37，它的操作权限包含 Notify”。手机先读了声明，才能在业务上正确地去操作第 3 行和第 4 行。

- <strong>安全把关（ATT Permissions）</strong>：如果客户端不看声明，企图直接对未配对的加密行执行 ATT Write Request，底层的 ATT 协议栈会根据本地权限规则直接拦截，回复错误响应码 0x05 (Insufficient Authentication)。


## 五、 传输瓶颈：从大表到空中包的尺寸限制（ATT MTU）

在 Excel 表中，一个单元格的 Value 最大可以定义到 512 字节。但在空口发送时，ATT 必须受限于射频单包的物理吞吐能力（MTU，最大传输单元）：

- <strong>默认 20 字节有效净荷</strong>：经典 BLE 默认 ATT MTU 为 23 字节。空中发出的每一个 ATT 包，必须包含 1 字节的操作类型（Opcode）和 2 字节的目标行号（Handle）。因此，23 - 3 = 20 字节，单次搬运的单元格数据上限就是 20 字节。

- <strong>协商大货车（Exchange MTU）：手机和设备会交换各自可接收的 MTU，后续取两者较小值。247 是常见的高吞吐配置；它与 Attribute Value 的 512 字节上限是两个不同概念。</strong>

- <strong>分批长包处理</strong>：如果单元格数据超过协商的 MTU，ATT 就必须动用分片搬运指令（读取用 Read Blob Request 带偏移量分批拉取；写入用 Prepare Write Request 分段暂存后执行 Execute Write Request 原子提交）。


## 六、 传输机制深究：MTU 与 Data Length 的关系，超长数据怎么发？

在很多人的认知里，数据超出 MTU 就会报错发不出去。但现实中，BLE 属性的 Value 最大允许 512 字节，手机甚至能一次性写入上百字节。这背后涉及到两个极其关键的通信概念：ATT MTU 与 Link Layer Data Length（DLE）的分层协作，以及 Long Write（长数据写）的底层分片机制。


### 1. ATT MTU 与 Data Length（DLE）的关系

用交通物流做比喻：

- ATT MTU 决定了“上层一趟车能装多少货”（应用层能塞的单包大小）；

- Data Length（DLE）决定了“空口物理跑道上一截车厢有多长”（链路层射频单包最大承载）。


| <strong>对比维度</strong> | <strong>ATT MTU（属性最大传输单元）</strong> | <strong>Data Length / DLE（链路层数据长度）</strong> |
| --- | --- | --- |
| 所属层级 | 协议顶层（ATT / GATT 协议层） | 协议底层（Link Layer 链路层 / 物理射频层） |
| 默认规格 | 默认 23 字节；扣除 3 字节包头后有效净荷 20 字节。 | 默认 27 字节；扣除 L2CAP 头 4 字节后实际承载 23 字节。 |
| 协商与扩展 | 双方交换可接收 MTU，最终取较小值；247 是常见优化取值。Attribute Value 的 512B 上限另行计算。 | BLE 4.2+ 通过 LL_LENGTH_REQ/RSP 扩容至 251 字节。 |
| 物理层协作 | 无 DLE：MTU 即使扩至 247，仍需拆成十多个小帧。 | 黄金组合 MTU 247 + DLE 251：247B ATT + 4B L2CAP = 251B，单射频帧发完，吞吐约提升 4～5 倍。 |


![ATT 图解](/images/ble/ble-att-mtu-dle.png)


### 2. 数据超过 MTU 到底能不能发？GATT 四种“写”的底层真相

答案是：普通写和推送不能超；但“长数据写入（Long Write）”完全可以发，因为底层走的根本不是普通的写指令。在 GATT 规范中，“写特征值”定义了 4 种独立过程，它们映射到 ATT 层的指令完全不同：


| <strong>GATT 写的业务过程</strong> | <strong>单次允许的数据长度</strong> | <strong>底层实际调用的 ATT PDU</strong> | <strong>机制特点与超长表现</strong> |
| --- | --- | --- | --- |
| Write Without Response（无应答写） | ≤ ATT_MTU - 3 | ATT_WRITE_CMD（0x52） | 单包盲发，无分片机制；超长 API 会报错或固件丢包。 |
| Write Characteristic Value（普通带应答写） | ≤ ATT_MTU - 3 | ATT_WRITE_REQ（0x12）→ ATT_WRITE_RSP（0x13） | 单包握手，无 Offset 字段；只能写入单包可容纳的短数据。 |
| Write Long Characteristic Values（Long Write） | 支持超长数据（最大 512 字节） | 多次 ATT_PREPARE_WRITE_REQ（0x16）→ 最后 ATT_EXECUTE_WRITE_REQ（0x18） | 分片暂存 + 原子提交；完全不使用 0x12 普通写。 |
| Reliable Writes（可靠写 / 回显校验） | 任意长度（可组合多特征） | 多包 Prepare Write + 回显校验 → Execute Write（0x01 提交 / 0x00 回滚） | 与 Long Write 共用 Prepare/Execute；客户端必须逐段核验回显，不一致则取消提交。 |


### 

术语澄清：Long Write 与 Reliable Write 不是两套 ATT 传输机制，二者都使用 Prepare Write / Execute Write。前者解决“单包放不下”；后者额外规定客户端必须逐段比对 Prepare Write Response 中的 Handle、Offset 和数据，确认无误才以 Flags=0x01 提交；任一回显不一致则以 Flags=0x00 取消。也就是说，Reliable Write 的差别正是“是否规范化地校验回显”。


### 3. Long Write 报文时序拆解：它是怎么把超长数据发完的？

假设 ATT_MTU = 23，手机要写入一个 30 字节字符串。

为什么普通 Write Request 搞不定？因为 ATT_WRITE_REQ 的格式固定为 [Opcode（1B）+ Handle（2B）+ Value]，报文中根本没有偏移量（Offset）字段。若协议栈硬把它拆成两包普通写，从机收到第二包时根本不知道该把数据拼在第几字节后面。

Long Write 的标准报文时序如下：

1. 第 1 包（暂存前 18 字节）：手机发送 ATT_PREPARE_WRITE_REQ，携带 Handle=0x0003、Offset=0、Data=前 18 字节。从机将其存入 RAM 写入缓冲队列，并回显 ATT_PREPARE_WRITE_RSP 确认；此时大表数据尚未改动。

2. 第 2 包（暂存后 12 字节）：手机发送 ATT_PREPARE_WRITE_REQ，携带 Handle=0x0003、Offset=18、Data=后 12 字节。从机将其拼在缓冲区后方并回显确认。

3. 第 3 包（签字生效）：手机发送 ATT_EXECUTE_WRITE_REQ（Flags=0x01）。从机收到命令后，原子性地把缓冲区中的 30 字节整块刷入 Handle 0x0003 单元格，并回复 ATT_EXECUTE_WRITE_RSP。


![ATT 图解](/images/ble/ble-att-long-write.png)


## 七、 附录全景表：BLE ATT 全量指令 / PDU 速查

在 ATT 协议定义中，所有读写、检索与推送均由以下标准 ATT PDU 承载：


| <strong>Opcode</strong> | <strong>ATT PDU 指令名称</strong> | <strong>发起方向</strong> | <strong>机制类型</strong> | <strong>指令功能与映射到大表的操作</strong> |
| --- | --- | --- | --- | --- |
| 0x01 | <strong>ATT_ERROR_RSP</strong> | Server → Client | Response | 通用错误应答。当请求的 Handle 不存在、权限不足（如未加密/未认证）时返回。 |
| 0x02 | <strong>ATT_EXCHANGE_MTU_REQ</strong> | Client → Server | Request | 客户端请求协商双方的 ATT 最大传输单元 (MTU)。 |
| 0x03 | <strong>ATT_EXCHANGE_MTU_RSP</strong> | Server → Client | Response | 服务端回复自身支持的 MTU，双方取二者较小值作为后续单包上限。 |
| 0x04 | <strong>ATT_FIND_INFO_REQ</strong> | Client → Server | Request | 在指定 Handle 区间内查找所有行对应的 Handle 和 UUID（常用于查找描述符 CCCD 0x2902）。 |
| 0x05 | <strong>ATT_FIND_INFO_RSP</strong> | Server → Client | Response | 返回指定区间内的一组 Handle 与 UUID 列表。 |
| 0x06 | <strong>ATT_FIND_BY_TYPE_VALUE_REQ</strong> | Client → Server | Request | 按 Attribute 类型 UUID 及特定 Value 内容搜索匹配的 Handle 范围（常用于按服务 UUID 快速检索服务）。 |
| 0x07 | <strong>ATT_FIND_BY_TYPE_VALUE_RSP</strong> | Server → Client | Response | 返回匹配行的起始 Handle 与结束 Handle 列表。 |
| 0x08 | <strong>ATT_READ_BY_TYPE_REQ</strong> | Client → Server | Request | 在给定 Handle 区间搜索指定 UUID 的行并返回其 Value（常用于读取 0x2803 特征声明行）。 |
| 0x09 | <strong>ATT_READ_BY_TYPE_RSP</strong> | Server → Client | Response | 返回搜索到的 Handle 及其 Value 列表（包含特征属性、值 Handle、特征 UUID）。 |
| 0x0A | <strong>ATT_READ_REQ</strong> | Client → Server | Request | 直接指定 Handle，请求读取该行的 Value 内容。 |
| 0x0B | <strong>ATT_READ_RSP</strong> | Server → Client | Response | 返回该 Handle 对应的 Value 单元格字节内容。 |
| 0x0C | <strong>ATT_READ_BLOB_REQ</strong> | Client → Server | Request | 长数据分片读取：指定 Handle 与偏移量 (Offset)，拉取超长 Value 的后续分块。 |
| 0x0D | <strong>ATT_READ_BLOB_RSP</strong> | Server → Client | Response | 返回从指定偏移量开始的 Value 切片字节。 |
| 0x0E | <strong>ATT_READ_MULTIPLE_REQ</strong> | Client → Server | Request | 一次性指定多个 Handle 列表，请求打包读取多个定长 Attribute 的 Value。 |
| 0x0F | <strong>ATT_READ_MULTIPLE_RSP</strong> | Server → Client | Response | 按请求顺序串联返回多个 Attribute 的 Value 组合流。 |
| 0x10 | <strong>ATT_READ_BY_GROUP_TYPE_REQ</strong> | Client → Server | Request | 按分组类型 UUID（如 0x2800 主服务）扫描表，获取每个服务区块的起始与结束 Handle 范围。 |
| 0x11 | <strong>ATT_READ_BY_GROUP_TYPE_RSP</strong> | Server → Client | Response | 返回服务分组列表（各主服务的 Start Handle, End Handle, Service UUID）。 |
| 0x12 | <strong>ATT_WRITE_REQ</strong> | Client → Server | Request | 向指定 Handle 写入 Value（带应用层确认），需等待服务端回复才能发下一包。 |
| 0x13 | <strong>ATT_WRITE_RSP</strong> | Server → Client | Response | 服务端确认写入完成的响应报文（无数据 Payload）。 |
| 0x52 | <strong>ATT_WRITE_CMD</strong> | Client → Server | Command | 无应答写入（Write Without Response）。客户端无需等回包，常用于高吞吐量写或 OTA。 |
| 0xD2 | <strong>ATT_SIGNED_WRITE_CMD</strong> | Client → Server | Command | 带签名写入。尾部附加 12 字节基于 CSRK 的 MAC 认证码，用于未加密连接下的可信写。 |
| 0x16 | <strong>ATT_PREPARE_WRITE_REQ</strong> | Client → Server | Request | 长数据切片写入预存：携带 Handle、Offset 和切片 Value，暂存到服务端的写入队列缓冲区。 |
| 0x17 | <strong>ATT_PREPARE_WRITE_RSP</strong> | Server → Client | Response | 服务端回显校验收到的 Handle、Offset 和切片内容，确认缓冲成功。 |
| 0x18 | <strong>ATT_EXECUTE_WRITE_REQ</strong> | Client → Server | Request | 提交执行长数据写入：标志位传 0x01 执行全量写入，传 0x00 则放弃并清空写入缓冲队列。 |
| 0x19 | <strong>ATT_EXECUTE_WRITE_RSP</strong> | Server → Client | Response | 服务端确认队列数据已原子提交写入完成。 |
| 0x1B | <strong>ATT_HANDLE_VALUE_NTF</strong> | Server → Client | Notification | 无 ATT 层确认的数据推送（Notification）。服务端主动将指定 Handle 的 Value 推给客户端；链路层仍可能进行确认与重传。 |
| 0x1D | <strong>ATT_HANDLE_VALUE_IND</strong> | Server → Client | Indication | 带确认数据指示（Indication）。服务端主动推送数据，要求客户端必须回复确认。 |
| 0x1E | <strong>ATT_HANDLE_VALUE_CFM</strong> | Client → Server | Confirmation | 客户端回复 Indication 的确认报文，服务端收到后方可触发下一个指示。 |


## 八、 总结：一文理顺 BLE 通信全景

将两篇内容合二为一，BLE 核心数据层与传输层的关系便一览无余：

- <strong>Attribute 是砖块</strong>：由 Handle、UUID、Value、Permissions 构成的最小原子数据条目。

- <strong>GATT 是建筑图纸</strong>：用这些砖块搭出了 Service（功能区）和 Characteristic（数据组件），赋予裸数据明确的业务语义。

- <strong>ATT 是物流卡车</strong>：负责根据图纸上的 Handle 门牌号，在空口无线信道中收发 PDU 报文，准确执行读、写、检索、推送与确认指令。
