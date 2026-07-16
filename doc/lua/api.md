# Synaxis Luacuit Lua API 参考

> 本文档涵盖 Luacuit 脚本环境中所有可用 API：定义阶段、运行阶段、数学库、控制 API、物理快照、全部 15 种 PlantPort 一览（含玩家视角行为描述、更新频率与延迟分析），以及完整示例脚本。

---

## 目录

1. [执行模型概览](#1-执行模型概览)
2. [脚本结构与执行阶段](#2-脚本结构与执行阶段)
3. [定义阶段 API (`define()`)](#3-定义阶段-api-define)
4. [运行阶段 API (`loop()`)](#4-运行阶段-api-loop)
5. [物理快照 `phys` / `Phys`](#5-物理快照-phys--phys)
6. [控制 API `control` / `Control`](#6-控制-api-control--control)
7. [数学库 (`luaml`)](#7-数学库-luaml)
8. [可用值类型与转换规则](#8-可用值类型与转换规则)
9. [执行预算 (Budget)](#9-执行预算-budget)
10. [可用 PlantPort 一览](#10-可用-plantport-一览)
11. [示例脚本](#11-示例脚本)

---

## 1. 执行模型概览

### Warm VM

同一个 Luacuit 实例会**复用**运行时 Lua VM。顶层 `local`、闭包 upvalue、全局变量会在重复 `loop()` 执行之间**保留**，但不会跨脚本，跨世界退出/重进保留，后者请查看state.*。

```lua
local t0 = 1
local t1 = 1

function loop()
    local now = input.real("x")
    t0 = t1
    t1 = now
end
```

以下情况会**重置** Warm VM：
- Apply / Load / 脚本重编译 / 端口结构变化
- 运行时错误 / 单次运行超时产生异常
- 显式取消

需要**可命名、可观察的持久存储**时，仍然应使用 `state.*`。

### 异步执行模式

- Luacuit 为主（端点方块）：`step()` 在 game-tick 或 physics-substep 线程捕获输入帧，提交到线程池异步执行。拥有完整 API（`bus`、`wireless`）
- Luacuit 为 Circuit 嵌入：`bus` 和 `wireless` API **始终不可用**（调用抛错），`phys` 和 `input`/`output` 正常。执行域取决于父电路中所有组件的域交集——如果交集包含 `PHYSICS_SUBSTEP_CONTROL`，嵌入的 Luacuit 也运行在物理子步
- 下次 `step()` 等待并应用已完成的结果
- 单实例最多一个 in-flight 任务；重复提交返回取消状态

### 执行域

| 域 | 说明 | 可用性 |
|----|------|--------|
| `GAME_TICK` | 默认，server tick（20 Hz） | 始终可用 |
| `PHYSICS_SUBSTEP_CONTROL` | 物理 substep（~20–200 Hz），BEFORE_CONTROLS 阶段 | 端点模式；Circuit 嵌入需父电路所有组件支持 |
| `PHYSICS_SUBSTEP_READONLY` | 只读物理快照 | 仅 bus 读取 |

---

## 2. 脚本结构与执行阶段

Luacuit 脚本分为两个阶段：

```lua
function define()
    -- 声明输入/输出端口
    -- 可用：defineInput / defineOutput / execution / control / debug（noop）/ print（noop）
    -- 不可用：input / output / state / bus / wireless
    -- phys 表存在但值为空快照
end

function loop()
    -- 始终可用：input / output / state / phys / control / debug / print
    -- 端点模式额外可用：bus / wireless
    -- Circuit 嵌入模式额外不可用：bus / wireless（调用抛错）
    -- 不可用：defineInput / defineOutput
end
```

两阶段都**非强制**——如果不需要声明端口，可以省略 `define()`。

### `print(...)`

```lua
print(...)
```

- 参数按空格拼接成一行
- 输出进入 Luacuit UI 第 1 页的 `Console` 区域
- **不会**发送聊天消息
- 控制台行数受 `maxConsoleLines` 预算约束（默认 128）

---

## 3. 定义阶段 API (`define()`)

仅在 `define()` 函数内可用。运行阶段调用会抛出类似 `"defineInput.real is unavailable during the execution pass"` 的错误。

### `defineInput`

声明输入端口。

```lua
defineInput.real(name, defaultValue)       -- defaultValue 缺省为 0.0
defineInput.bool(name, defaultValue)       -- defaultValue 缺省为 false
defineInput.vec3(name, x, y, z)            -- 各分量缺省为 0
defineInput.quaternion(name, x, y, z, w)   -- w 缺省为 1
```

参数 `name` 必须是合法端口名。缺省值仅在未声明时生效——例如 `defineInput.real("x")` 默认为 `0.0`，`defineInput.real("x", 5)` 默认为 `5.0`。

### `defineOutput`

声明输出端口。

```lua
defineOutput.real(name)
defineOutput.bool(name)
defineOutput.vec3(name)
defineOutput.quaternion(name)
```

### `execution` — 执行策略请求

脚本可以在 `define()` 中请求自己的执行策略。服务器配置拥有最终决定权：不允许的请求会自动降级为安全默认值，不会让脚本绕过服务器限制。

```lua
function define()
    execution.mode("async_wait")          -- 默认：异步执行，step 可等待一小段时间取结果
    execution.mode("async_non_blocking")  -- 异步执行，step 不主动等待本轮结果
    execution.mode("sync")                -- 请求游戏刻同步执行；需要服务器允许

    execution.trusted(false)              -- 默认：完整 watcher 预算检查
    execution.trusted("light")            -- 请求轻量 watcher；需要服务器允许
    execution.trusted("unwatched")        -- 请求关闭 watcher；仅可信脚本使用，需要服务器允许
end
```

说明：
- `"sync"` / `"synchronous"` / `"sync_game_tick"` 等价；默认只允许在 `GAME_TICK` 域同步执行。
- `"async"` / `"async_wait"` 表示异步执行并允许主线程等待预算内的结果；`"async_non_blocking"` 表示完全不等结果，输出通常延后一轮或更多轮可见。
- `execution.trusted("light")` 会降低 watcher 检查频率；`execution.trusted("unwatched")` 会关闭 Lua watcher hook，公共服务器不要对普通玩家开放。
- `execution.trust(...)` 是 `execution.trusted(...)` 的别名。

### 定义阶段可用的其他 API

| API | 说明 |
|-----|------|
| `execution` | 请求异步/同步执行模式和 watcher 信任级别；服务器配置可降级 |
| `control` / `Control` | 低通滤波器和 PID 控制器（同运行阶段） |
| `debug` / `Debug` | `debug.watch(name, value)` 在定义阶段是 no-op（不记录任何值） |
| `phys` / `Phys` | 物理快照 API 存在，但**值为空快照** |
| `print(...)` | 定义阶段是 no-op |

---

## 4. 运行阶段 API (`loop()`)

仅在 `loop()` 函数内可用。定义阶段调用会抛出类似 `"input.real is unavailable during the definition pass"` 的错误。

### `input` — 读取输入端口

```lua
input.get(name)          -- 返回端口声明类型的值
input.real(name)         -- 校验端口为 REAL，返回 number
input.bool(name)         -- 校验端口为 BOOLEAN，返回 boolean
input.vec3(name)         -- 校验端口为 VEC3，返回 Vector3d
input.quaternion(name)   -- 校验端口为 QUATERNION，返回 Quaterniond
```

未写入时读取端口默认值，不会返回 `nil`。

### `output` — 写入输出端口

```lua
output.set(name, value)        -- 自动按端口类型校验和转换
output.real(name, value)       -- 校验端口为 REAL，写入 number
output.bool(name, value)       -- 校验端口为 BOOLEAN，写入 boolean
output.vec3(name, value)       -- 校验端口为 VEC3，接受 Vector3d 或 {x,y,z} table
output.quaternion(name, value) -- 校验端口为 QUATERNION，接受 Quaterniond 或 {x,y,z,w} table
```

单次执行写入次数受 `maxOutputWrites` 预算约束（默认 128）。

### `state` — 持久状态存储

```lua
state.get(key)     -- key 不存在返回 nil
state.set(key, value)
state.clear(key)
```

- `value` 支持 `number` / `boolean` / `Vector3d` / `Quaterniond` / 带 `x,y,z,(w)` 的 table
- `state.set(key, nil)` 与 `state.clear(key)` 等价，都会删除该 key
- 条目数受 `maxStateEntries` 预算约束（默认 128）

`state` 与 Warm VM 的区别：
- Warm VM 中的 `local` 变量**不持久化**，VM 重置后丢失
- `state.*` 跨 VM 重置**持久保留**

### `bus` — Body Bus 读写

> **注意**：`bus` API 仅在 Luacuit **端点方块**模式下可用。Circuit 嵌入的 Luacuit 调用会抛出 `"unavailable during embedded execution"` 错误。嵌入的 Luacuit 仍可运行在物理子步——执行域取决于父电路，只是不能通过 bus 访问外部设备。

```lua
-- define() 阶段：声明希望零冷启动预取的 bus 读端口
bus.subscribe(deviceName, portName)
bus.reads(deviceName, portName)           -- subscribe 的别名

-- loop() 阶段：读取或写入 bus
bus.read(deviceName, portName)           -- 返回聚合后的 bus 端口值，不存在返回 nil
bus.read(deviceName, portName, fallback)  -- 不存在时返回 fallback
bus.write(deviceName, portName, value)    -- 记录一条 bus 写入请求，延迟提交
```

- `bus.subscribe` / `bus.reads` 只能在 `define()` 中调用，用于声明需要预取的 bus 读端口；超过 `maxDeclaredBusReads` 会导致定义阶段报错
- `bus.read` 只读取当前输入帧中已预取的端口；如果端口未预取或不存在，则返回 `fallback` / `nil`
- 未声明的动态 `bus.read` 会被记录为 observed read，并在下一次 Luacuit 执行前加入预取集合
- observed read 集合带容量保护：超过高水位时，运行时按最后活跃时间丢弃最旧的动态读端口，直到回到 `maxObservedBusReads`
- 当前只支持 `REAL`、`BOOLEAN`、`VEC3` 三种信号类型
- `bus.write` 不会直接从 worker 线程写 BE——写入请求记录在 output frame 中，由运行时在后续阶段提交
- 外部写入次数受 `maxBusWrites` 预算约束（默认 128）。该预算由 `bus.write`、`wireless.state.write` 和 `wireless.command.send` 共享。

---

## 5. 物理快照 `phys` / `Phys`

`phys` 与 `Phys` 是**同一个表**，大小写两种名字都可用。

每个 physics substep 会看到当前 owner body 的最新 physics snapshot。

```lua
phys.position()            -- → Vector3d  模型锚点在世界坐标系中的位置
phys.positionCenterModel() -- → Vector3d  质心在模型坐标系中的位置
phys.positionCenter()      -- → Vector3d  质心在世界坐标系中的位置
phys.velocity()            -- → Vector3d  模型锚点处的线速度
phys.angularVelocity()     -- → Vector3d  角速度 (omega)
phys.quaternionToWorld()   -- → Quaterniond  世界朝向四元数
phys.mass()                -- → number  质量
phys.inertia()             -- → number  惯量（惯量张量的 m00，遗留便利方法）
phys.inertiaTensor()       -- → Matrix3d  完整惯量张量
```

> **推荐**：对于 Sable/general rigid body，使用 `phys.inertiaTensor()` 而非 `phys.inertia()`。

---

### `debug` / `Debug` — 调试监视

在定义阶段和运行阶段**均可用**（定义阶段为 no-op）。

```lua
debug.watch(name, value)    -- 将 (name, value) 记录到 Luacuit UI 的监视面板，返回 value
```

- 用于在 UI 中实时观察变量值
- 监视条目数受 `maxConsoleLines` 预算约束；新 watch key 超过上限会抛出 `CONSOLE_BUDGET`
- 定义阶段调用不会报错，但不记录任何值

### `wireless` / `Wireless` — 无线通信

> **注意**：`wireless` API 仅在 Luacuit **端点方块**模式下可用。定义阶段没有 `wireless` 表；Circuit 嵌入的 Luacuit 会安装不可用占位表，调用时抛出 `"unavailable during embedded execution"` 错误。

无线通信允许同一 cluster 内的多个 Luacuit 实例通过命名信道共享状态和发送命令。

**状态读写**（共享键值存储，同一信道所有端点可见）：

```lua
wireless.state.write(channel, value, {scope = "cluster"})    -- 写入一个值到信道
wireless.state.read(channel, "real", fallback, {scope = "cluster"})  -- 读取最新值
wireless.state.real(channel, fallback, {scope = "cluster"})          -- 读取 REAL 类型
wireless.state.bool(channel, fallback, {scope = "cluster"})          -- 读取 BOOLEAN 类型
wireless.state.boolean(channel, fallback, {scope = "cluster"})       -- bool 的别名
wireless.state.vec3(channel, fallback, {scope = "cluster"})          -- 读取 VEC3 类型
wireless.state.quaternion(channel, fallback, {scope = "cluster"})    -- 读取 QUATERNION 类型
wireless.state.quat(channel, fallback, {scope = "cluster"})          -- quaternion 的别名
```

**命令收发**（FIFO 队列，不覆盖）：

```lua
wireless.command.send(channel, value, {scope = "cluster"})    -- 发送命令到信道
wireless.command.drain(channel, "real", {scope = "cluster", limit = 16})   -- 排出所有命令（返回 table）
wireless.command.drain_real(channel, {scope = "cluster", limit = 16})       -- 排出 REAL 命令
wireless.command.drain_bool(channel, {scope = "cluster", limit = 16})       -- 排出 BOOLEAN 命令
wireless.command.drain_boolean(channel, {scope = "cluster", limit = 16})    -- drain_bool 的别名
wireless.command.drain_vec3(channel, {scope = "cluster", limit = 16})       -- 排出 VEC3 命令
wireless.command.drain_quaternion(channel, {scope = "cluster", limit = 16}) -- 排出 QUATERNION 命令
wireless.command.drain_quat(channel, {scope = "cluster", limit = 16})       -- drain_quaternion 的别名
```

说明：
- `channel`：字符串，信道名
- `scope`：可选。`"cluster"`（默认）= 同一物理 cluster 内所有 Luacuit 共享；`"world"` = 同维度内所有 Cluster 之间共享。集群间通信需要 `WORLD` scope
- `limit`：最多排出多少条命令。缺省为 256，即 `CimulinkWirelessService.MAX_VALUES_PER_CHANNEL`；传入值会被限制在 `0..256`
- `state` 是共享覆盖写——后写入的值覆盖先前的。无 domain 限制，物理子步和游戏刻均可读写
- `command` 是 FIFO 队列——每条命令独立保留，直到被 `drain` 排出
- 类型不匹配时返回 fallback（或 nil）
- `wireless.state.read` / `wireless.command.drain` 的类型字符串支持 `real`/`number`、`bool`/`boolean`、`vec3`/`vector`/`vector3`/`vector3d`、`quat`/`quaternion`/`quaterniond`
- `wireless.state.write` 和 `wireless.command.send` 与 `bus.write` 共用 `maxBusWrites` 外部写入预算

---

## 6. 控制 API `control` / `Control`

`control` 与 `Control` 是**同一个表**。在定义阶段和运行阶段**均可用**。

> 本节代码块展示对象构造和方法签名。写完整 Luacuit 脚本时，不要在顶层直接构造 `control.lowpass(...)` 或 `control.pid(...)`；请参考后文示例，在 `loop()` 第一次执行时懒初始化。

### 低通滤波器 `control.lowpass(options)` / `control.lowPass(options)`

一阶指数移动平均（EMA）滤波器，用于平滑噪声信号。

```lua
local lp = control.lowpass({
    tau     = 0.25,   -- 时间常数（默认 0.25）
    initial = 0.0,    -- 初始值（默认 0.0）
})

local filtered = lp:update(value)        -- dt 默认 0.05
local filtered = lp:update(value, dt)
lp:reset()                                -- 重置为 initial
local s = lp:state()                      -- → { value = 当前滤波值 }
```

**数学公式**：

```
alpha = dt / (tau + dt)
output = output + alpha * (input - output)
```

**参数详解**：

| 参数 | 默认值 | 含义 |
|------|:------:|------|
| `tau` | 0.25 | 时间常数。**越大响应越慢、平滑越强**。`tau=0` 时 `alpha=1`，输出直接等于输入（无滤波）。`tau` 与 `dt` 的相对大小决定平滑程度：`tau >> dt` 时 alpha 很小，输出变化极慢；`tau ≈ dt` 时 alpha≈0.5，新旧值各占一半 |
| `initial` / `initialValue` | 0.0 | `reset()` 后恢复到的初始值。首次 `update()` 之前也为此值 |

**行为**：
- 每次 `update(value, dt)` 用指数加权更新内部状态
- `dt` 必须 >0；`dt` 越大（两次调用间隔越长），新值权重越大
- 滤波器内部状态跨 `loop()` 调用保留（存储在 `LowPassFilterMemory` 中）

**典型调参**：
- 传感器噪声平滑：`tau = 0.1~0.5`（在 20Hz 下 dt≈0.05，alpha≈0.17~0.33）
- 重度平滑：`tau = 1.0~2.0`（alpha≈0.02~0.05，输出缓慢跟踪输入）

---

### PID 控制器 `control.pid(options)`

标准 PID（比例-积分-微分）控制器，用于闭环控制。

```lua
local pid = control.pid({
    kp             = 1.0,        -- 比例增益（默认 1.0）
    ki             = 0.0,        -- 积分增益（默认 0.0）
    kd             = 0.0,        -- 微分增益（默认 0.0）
    min            = -1.0,       -- 输出下限（默认 -1.0；也可写 outputMin）
    max            = 1.0,        -- 输出上限（默认 1.0；也可写 outputMax）
    integralMin    = -1.0,       -- 积分累加下限（默认 -1.0）
    integralMax    = 1.0,        -- 积分累加上限（默认 1.0）
    deadband       = 0.0,        -- 死区（默认 0.0）
    derivative     = "measurement", -- 微分模式（默认 "measurement"；也可写 derivativeMode）
    antiWindup     = "clamp",       -- 抗积分饱和（默认 "clamp"）
    derivativeTau  = 0.0,        -- 微分低通滤波 tau（默认 0.0，无滤波；也可写 derivativeFilterTau）
    rateLimit      = 0.0,        -- 输出速率限制（默认 0.0，无限制；也可写 outputRateLimit）
    initialIntegral = 0.0,       -- 初始积分值（默认 0.0）
})

local output = pid:update(setpoint, measurement)          -- dt 默认 0.05
local output = pid:update(setpoint, measurement, dt)
pid:reset()
local s = pid:state()   -- → { integral, derivative, output, previousError,
                         --      previousMeasurement, initialized }
```

**数学公式**：

```
error = setpoint - measurement
if |error| <= deadband: error = 0

P = kp * error

integral += error * dt           （每步累加，受 integralMin/integralMax 限制）
I = ki * integral

derivative_raw = d(error)/dt      （"error" 模式）
              = -d(measurement)/dt （"measurement" 模式，默认）
derivative = lowpass(derivative_raw, derivativeTau, dt)  （若 derivativeTau > 0）
D = kd * derivative

unsaturated = P + I + D
output = clamp(unsaturated, min, max)
if rateLimit > 0: output = clamp(output, prev_output ± rateLimit * dt)
```

**参数详解**：

| 参数 | 默认值 | 含义 |
|------|:------:|------|
| `kp` | 1.0 | 比例增益。输出与当前误差成正比。**越大响应越快，但过大导致振荡**。设 0 禁用 P 项 |
| `ki` | 0.0 | 积分增益。输出与误差累积量成正比。**消除稳态误差**（让系统最终精确到达 setpoint）。**过大导致超调和低频振荡**。设 0 禁用 I 项 |
| `kd` | 0.0 | 微分增益。输出与误差变化率（或测量值变化率）成正比。**抑制振荡、提供阻尼**。**过大放大高频噪声**。设 0 禁用 D 项 |
| `min` / `outputMin` | -1.0 | 输出下限。最终输出被截断到此值。例如控制油门时设 `min=0` 防止反向 |
| `max` / `outputMax` | 1.0 | 输出上限。最终输出被截断到此值。`min` 和 `max` 自动排序（取小的为 min） |
| `integralMin` | -1.0 | 积分累加下限。限制积分项不会无限负向增长 |
| `integralMax` | 1.0 | 积分累加上限。限制积分项不会无限正向增长（防止 windup） |
| `deadband` | 0.0 | 死区宽度。`\|error\| ≤ deadband` 时 error 视为 0，PID 不产生纠偏。用于**消除 setpoint 附近的微颤**。设 0 禁用 |
| `derivative` / `derivativeMode` | `"measurement"` | 微分来源。`"measurement"`：对测量值求导（**推荐**，setpoint 突变不会产生微分冲击）；`"error"`：对误差求导 |
| `antiWindup` | `"clamp"` | 抗积分饱和策略。`"clamp"`：积分累加始终被 `integralMin/Max` 截断；`"conditional"`：输出饱和且误差继续推向饱和方向时**冻结积分**（更智能）；`"none"`：不限制积分累加 |
| `derivativeTau` / `derivativeFilterTau` | 0.0 | 微分项低通滤波时间常数。用与 `lowpass` 相同的 EMA 算法平滑微分项。**>0 可抑制高频噪声对 D 项的放大**。0 = 不过滤 |
| `rateLimit` / `outputRateLimit` | 0.0 | 输出最大变化速率（单位/秒）。每次 step 的输出变化不超过 `rateLimit * dt`。用于**防止输出突变**（如保护机械结构）。0 = 无限制 |
| `initialIntegral` | 0.0 | `reset()` 后积分项的初始值。可用于热启动（warm start）——从已知的稳态积分值开始 |

**状态查询 `pid:state()` 返回**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `integral` | number | 当前积分累加值 |
| `derivative` | number | 当前微分项值（滤波后） |
| `output` | number | 上次 `update()` 的输出 |
| `previousError` | number | 上一步的误差值 |
| `previousMeasurement` | number | 上一步的测量值 |
| `initialized` | boolean | 是否至少执行过一次 `update()`（首次调用前为 false，此时 D 项为 0） |

**行为细节**：
- 首次 `update()` 时 D 项为 0（无历史数据），之后正常计算
- `reset()` 清空所有历史：积分=initialIntegral，微分=0，输出=0，`initialized=false`
- 积分累加使用 `error * dt`，不是 `(error * ki) * dt`——ki 在最终求和时才乘

**典型调参流程**：

1. 设 `ki=0, kd=0`，从小的 `kp`（如 0.5）开始增大直到系统快速响应但不振荡
2. 加入 `ki`（如 0.1×kp）消除稳态误差；若出现低频振荡则减小 ki
3. 加入 `kd`（如 0.01×kp）抑制超调和振荡；若输出抖动则设 `derivativeTau=0.02~0.1` 过滤噪声
4. 根据物理限制设 `min/max`；设 `integralMin/Max` 防止 windup
5. 若 setpoint 附近有微颤，设 `deadband=0.01~0.05`

---

## 7. 数学库 (`luaml`)

Luacuit 启动时自动加载 `luaml.lua`，并导出以下全局：

```
luaml    — 模块表（含所有导出）
Vector3d
Quaterniond
Matrix3d
expectVector3d
expectQuaterniond
expectMatrix3d
```

即 `luaml.Vector3d:new(1,2,3)` 和 `Vector3d:new(1,2,3)` 等价。

### `Vector3d`

| 方法 | 签名 | 说明 |
|------|------|------|
| `new` | `(x, y, z)` | 构造；缺省均为 0 |
| `copy` | `()` | 深拷贝，返回等值新实例 |
| `unpack` | `()` | 返回 `x, y, z` 三个值（Lua 多返回值） |
| `set` | `(other)` 或 `(x, y, z)` | **可变** 设置，返回 self |
| `add` | `(other)` 或 `(dx, dy, dz)` | **不可变** 加法，返回新实例 |
| `sub` | `(other)` | **不可变** 减法 |
| `mul` | `(other)` 或 `(scalar)` | **不可变** 乘（向量分量乘 / 标量乘） |
| `div` | `(other)` 或 `(scalar)` | **不可变** 除 |
| `length` | `()` | 长度 |
| `lengthSquared` | `()` | 长度平方 |
| `normalize` | `()` | **不可变** 归一化 |
| `dot` | `(other)` | 点积 |
| `cross` | `(other)` | 叉积 |
| `distance` | `(other)` | 距离 |
| `angle` | `(other)` | 夹角 (rad)。`angle` 内部做了 `acos(clamp(dot, -1, 1))` 防越界 |
| `negate` | `()` | 取反 |
| `absolute` | `()` | 各分量取绝对值 |
| `lerp` | `(other, t)` | 线性插值：`self + (other - self) * t` |
| `min` | `(other)` | 各分量取两向量中较小值 |
| `max` | `(other)` | 各分量取两向量中较大值 |
| `project` | `(onto)` | 投影到 `onto` 方向上的分量（标量投影 × 单位方向） |
| `reject` | `(onto)` | 与 `onto` 正交的分量。`reject = self - project(onto)` |
| `clampLength` | `(maxLen)` | 若长度 > `maxLen`，缩放到该长度；否则返回拷贝 |
| `signedAngle` | `(other, axis)` | 带符号的夹角 (rad)。用 `axis` 方向判断正负（右手定则） |
| `equalsEpsilon` | `(other, eps)` | 各分量差的绝对值均 ≤ `eps`（默认 1e-9）时返回 true |

### `Quaterniond`

| 方法 | 签名 | 说明 |
|------|------|------|
| `new` | `(x, y, z, w)` | 构造；缺省均为 0，w 为 1 |
| `.identity()` | — | **静态方法**，返回单位四元数 `(0,0,0,1)` |
| `.fromAxisAngle` | `(axis, angle)` | **静态方法**，从轴角创建（axis 自动归一化） |
| `.fromEuler` | `(roll, pitch, yaw)` 或 `(Vector3d)` | **静态方法**，从欧拉角（弧度）创建，使用 ZYX 旋转顺序 |
| `.lookAlong` | `(direction, up)` | **静态方法**，从视线方向和上方向创建朝向四元数 |
| `copy` | `()` | 深拷贝 |
| `unpack` | `()` | 返回 `x, y, z, w` 四个值 |
| `set` | `(other)` 或 `(x, y, z, w)` | **可变** 设置 |
| `setFromAxisAngle` | `(axis, angle)` | **可变** 从轴角设置，axis 自动归一化 |
| `add` | `(other)` | **不可变** 加法 |
| `mul` | `(other)` 或 `(scalar)` | **不可变** 乘（四元数乘 / 标量乘） |
| `length` | `()` | 长度 |
| `normalize` | `()` | **不可变** 归一化 |
| `normalizeInPlace` | `()` | **可变** 归一化，返回 self |
| `conjugate` | `()` | **不可变** 共轭 |
| `inverse` | `()` | **不可变** 逆（拒绝零四元数） |
| `transform` | `(vec)` | 正向旋转变换向量，返回新 Vector3d |
| `inverseTransform` | `(vec)` | 逆向旋转变换向量（用逆四元数变换） |
| `toEuler` | `()` | 转为欧拉角，返回 `Vector3d(roll, pitch, yaw)`，单位弧度 |
| `errorTo` | `(current)` | 计算从 `current` 朝向到 `self` 朝向的误差四元数。`error = self * conj(current)`，归一化后返回 |
| `slerp` | `(other, t)` | 球面线性插值。当两个四元数夹角极小时退化为线性插值 |

### `Matrix3d`

| 方法 | 签名 | 说明 |
|------|------|------|
| `new` | `(m00..m22)` 或无参 | 构造；无参 = 单位矩阵 |
| `identity` | `()` | 返回新单位矩阵 |
| `copy` | `()` | 深拷贝 |
| `get` | `(row, col)` | 取元素；索引为 **0/1/2**（非 1/2/3） |
| `row` | `(index)` | 取行向量 (Vector3d)；索引为 0/1/2 |
| `col` | `(index)` | 取列向量 (Vector3d)；索引为 0/1/2 |
| `mul` | `(other)` 或 `(scalar)` | **不可变** 乘 |
| `transform` | `(vec)` | 矩阵 × 向量，返回新 Vector3d |
| `transpose` | `()` | **不可变** 转置 |
| `trace` | `()` | 迹 |
| `determinant` | `()` | 行列式 |
| `invert` | `()` | **不可变** 逆矩阵；奇异矩阵会 error |

> **注意**：`Matrix3d:get(row, col)`、`row(i)`、`col(i)` 使用 `0/1/2` 索引，**不是** Lua 常见的 `1/2/3`。

**模块级快捷函数**（`luaml.xxx` 和全局均可使用）：

```lua
luaml.vec3(x, y, z)              -- 等价于 Vector3d:new(x, y, z)
luaml.quat(x, y, z, w)           -- 等价于 Quaterniond:new(x, y, z, w)
luaml.matrix3(m00..m22 或无参)    -- 等价于 Matrix3d:new(...)
luaml.clamp(value, min, max)     -- 数值截断：max(min, min(max, value))
```

---

## 8. 可用值类型与转换规则

### Lua → Luacuit（写入方向）

| Lua 值 | 目标类型 | 说明 |
|--------|----------|------|
| `number` | `REAL` | |
| `boolean` | `BOOLEAN` | |
| `Vector3d` 实例 | `VEC3` | |
| `{x,y,z}` table | `VEC3` | 也可用 `{1,2,3}` 索引 |
| `Quaterniond` 实例 | `QUATERNION` | |
| `{x,y,z,w}` table | `QUATERNION` | 也可用 `{1,2,3,4}` 索引 |

### Luacuit → Lua（读取方向）

| Luacuit 类型 | Lua 值 |
|-------------|--------|
| `REAL` | `number` |
| `BOOLEAN` | `boolean` |
| `VEC3` | `Vector3d` 实例 |
| `QUATERNION` | `Quaterniond` 实例 |

### 当前限制

- `POSE` / `TWIST` / `BUNDLE` **不能**直接作为 Luacuit 端口类型
- Bus 读写仅支持 `REAL` / `BOOLEAN` / `VEC3`（不支持 `QUATERNION`）
- `print(...)` 只回流到 UI Console，不发聊天消息
- 定义阶段的 `phys` 快照为空，不应依赖它决定 schema

---

## 9. 执行预算 (Budget)

预算的作用域不完全相同：源码大小在编译前检查；step 等待预算用于 `step()` 等待已有 in-flight 任务；指令、墙钟、分配、输出、外部写入、console/watch、state 条目预算作用于单次运行帧。

| 预算项 | 默认值 | 检查方式 |
|--------|:------:|------|
| `maxSourceBytes` | 320 KB | 编译前一次性检查源码的 UTF-8 字节数 |
| `maxInstructions` | 2,500,000 | **每条指令检查**（`onInstruction` 钩子）。超过后抛出 `LuacuitBudgetExceededError(INSTRUCTION_LIMIT)` → 状态变为 `FAULTED` |
| `maxWallClockMillis` | 250 ms | 每 **100 条指令**检查一次实际耗时（`System.nanoTime()`）。超过后抛出 `TIMEOUT` → 状态变为 `TIMED_OUT` |
| `maxStepWaitGameTickMicros` | 24,000 µs | `step()` 调用时，等待已有 in-flight 任务完成的**最长阻塞时间**（游戏刻）。超时后本次跳过，下个 step 重试 |
| `maxStepWaitPhysicsControlMicros` | 8,000 µs | 同上，物理子步中的等待超时。物理子步的等待时间更短是为了避免阻塞物理管线 |
| `maxAllocatedBytes` | 4 MB | 每 **1000 条指令**检查一次 JVM 线程已分配内存增量（`ThreadMXBean.getThreadAllocatedBytes`）。超过后抛出 `MEMORY_LIMIT` → 状态变为 `FAULTED` |
| `maxOutputWrites` | 128 | 每次 `output.*` 调用时检查。超过后抛出 `OUTPUT_BUDGET` → 状态变为 `FAULTED` |
| `maxBusWrites` | 128 | 每次外部写入时检查；`bus.write()`、`wireless.state.write()`、`wireless.command.send()` 共享此预算。超过后抛出 `BUS_BUDGET` |
| `maxDeclaredBusReads` | 128 | `define()` 中 `bus.subscribe()` / `bus.reads()` 声明的 bus 读端口上限。超过后定义阶段报错 |
| `maxObservedBusReads` | 128 | 运行时动态 `bus.read()` 学习集合的目标容量。超过高水位后会裁剪最旧的 observed read |
| `maxConsoleLines` | 128 | 每次 `print()` 调用以及新增 `debug.watch()` key 时检查。超过后抛出 `CONSOLE_BUDGET` |
| `maxStateEntries` | 128 | `state.set()` 对**新的 key** 写入时检查。修改已有 key 或删除 key 不增加条目数 |

### 执行策略与服务器配置

服务器配置位于 `cimulink.luacuit`：

| 配置项 | 默认值 | 作用 |
|--------|:------:|------|
| `default_execution_mode` | `async_wait` | 未显式请求时的默认异步模式；支持 `async_wait` / `async_non_blocking` |
| `allow_script_sync` | `false` | 是否允许脚本通过 `execution.mode("sync")` 请求游戏刻同步执行 |
| `max_sync_game_tick_micros` | `1000` | 同步执行时 watcher 的墙钟上限 |
| `max_sync_luacuits_per_tick` | `64` | 每个 tick 最多允许多少个 Luacuit 走同步路径，超过后降级为异步 |
| `allow_sync_in_physics_substep` | `false` | 是否允许物理子步中同步执行；默认关闭 |
| `max_async_wait_game_tick_micros` | `24000` | 异步等待模式下，游戏刻最多等待结果的时间 |
| `max_async_wait_physics_micros` | `8000` | 异步等待模式下，物理子步最多等待结果的时间 |
| `allow_trusted_light` | `false` | 是否允许 `execution.trusted("light")` |
| `allow_trusted_unwatched` | `false` | 是否允许 `execution.trusted("unwatched")`；不建议公共服务器开启 |

实际等待时间还会受脚本自身 `LuacuitBudget` 中的 step wait 预算限制，最终取脚本预算和服务器上限中较小的值。

超预算行为：
- `TIMEOUT` / `INSTRUCTION_LIMIT` / `MEMORY_LIMIT`：当前帧输出被丢弃，Warm VM 清空，下次 `loop()` 重新编译
- `OUTPUT_BUDGET` / `BUS_BUDGET` / `CONSOLE_BUDGET` / `STATE_BUDGET`：当前帧输出被丢弃，Warm VM 清空
- 连续多次失败后 → `DISABLED`（需手动重新启用）

---

## 10. 可用 PlantPort 一览

> **阅读指南**：本章从玩家视角描述每个 PlantPort 的行为。重点是：给某个端口写一个值之后，游戏里会发生什么；读某个端口会得到什么数值；这些数值多久更新一次。

### 核心概念：游戏刻 vs 物理子步

Minecraft 服务器以 **20 TPS**（每秒 20 个游戏刻，每刻 50ms）运行。物理引擎在每个游戏刻内可能执行多个**物理子步**（substep，通常 ~20–200 Hz）。

- **游戏刻执行**（`GAME_TICK`）：你的 Luacuit 脚本在游戏刻中运行，通过 `bus.write` 写入 plant 端口
- **物理子步执行**（`PHYSICS_SUBSTEP_CONTROL`）：如果整个 Cimulink 网络都支持物理子步，它会在物理子步中运行——**写入立即在同一物理子步内生效**，不需要等下一个游戏刻
- 表格中的 `物理子步` 列含义：`✅` 表示这个端口可在物理子步中读取；`✅✅` 表示这个输入端口可在物理子步中写入并快速生效；`❌` 表示只能在游戏刻网络中使用。

### 物理子步写入的即时性

对于支持物理子步写入的设备（标记为 ✅✅），写入的值**在同一个物理子步内**参与设备控制。直观地说，脚本刚算出的舵角、推力或转速，可以立刻被这一小步物理模拟用上，不需要等下一个 50ms 游戏刻。这能显著降低控制环路延迟。

**这意味着：物理子步写入是同一子步内即时生效的。你不需要等下一个游戏刻。**

### 兼容层设备的限制

来自 Create 和 Simulated 模组的 PlantPort **不支持物理子步**。它们只能参与 `GAME_TICK` 域的 Cimulink 网络。如果一个网络中包含任何此类设备，整个网络都会被归类为 `GAME_TICK`。

---

### 10.1 Compact Flap（紧凑舵面）

> 方块外观：一个可偏转的空气动力学舵面。你可以用 Cimulink 控制它的偏转角和倾斜角来产生气动力。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `angle` | 输入 | REAL | ✅✅ | 目标偏转角（度）。写入后，舵面会按这个角度影响气动力；在物理子步网络中可同一子步生效。角度会按 [-180, 180] 包裹 |
| `tilt` | 输入 | REAL | ✅✅ | 目标倾斜角（度）。行为同 `angle` |
| `current_angle` | 输出 | REAL | ✅ | 当前偏转角，通常就是设备最近采用的目标角 |
| `current_tilt` | 输出 | REAL | ✅ | 当前倾斜角 |

**使用提示**：舵面适合放在物理子步控制环里，用于姿态稳定、转向、升力微调等高速反馈。

设备类型 ID：`synaxis:plant/compact_flap`

---

### 10.2 Control Chair（控制椅）

> 方块外观：一个可乘坐的座椅。玩家坐上去后，鼠标/键盘输入会被捕获并通过 Cimulink 总线暴露给其他设备。

**玩家输入如何变成信号**：

1. 玩家坐在控制椅上，移动鼠标 / 按键
2. 客户端以约 **60 Hz**（每 16.67ms）的频率把输入状态同步到服务器
3. Cimulink 在下一次游戏刻或物理子步评估时读取最近一次同步到服务器的输入

**典型延迟**：
- 客户端发送间隔：约 16.67ms（60 Hz）
- 到 Luacuit 可观测：需等到下一个 Cimulink 评估周期；游戏刻网络最多再等 50ms，物理子步网络通常更短
- **总延迟：通常约 16–67ms**

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `lx` | 输出 | REAL | ✅ | Yaw 轴（左右旋转）。范围约 [-1, 1] |
| `ry` | 输出 | REAL | ✅ | Pitch 轴（上下俯仰）。范围约 [-1, 1] |
| `rx` | 输出 | REAL | ✅ | Roll 轴（横滚）。范围约 [-1, 1] |
| `ly` | 输出 | REAL | ✅ | Thrust 轴（推力/前进）。范围约 [-1, 1] |
| `used` | 输出 | BOOLEAN | ✅ | 是否有玩家正在乘坐（`true`/`false`） |
| `hasIn` | 输出 | BOOLEAN | ✅ | 乘坐的玩家是否启用了输入（可在 GUI 中切换） |
| `f5` | 输出 | BOOLEAN | ✅ | 玩家是否处于第三人称视角 |
| `local_yaw` | 输出 | REAL | ✅ | 玩家视角的本地 yaw 角度（度），结合了机体朝向 |
| `local_pitch` | 输出 | REAL | ✅ | 玩家视角的本地 pitch 角度（度） |
| `abs_forward` | 输出 | VEC3 | ✅ | 玩家视角在世界坐标系中的**绝对前向单位向量**。可用于导航/瞄准 |
| `<button>` | 输出 | BOOLEAN | ✅ | 各个按键状态。按键名如 `jump`、`sprint`、`use` 等 |

> **只读设备**（无输入端口）。按键名由控制椅支持的玩家动作决定。

设备类型 ID：`synaxis:plant/control_chair`

---

### 10.3 Dynamic Motor（动态电机）

> 方块外观：一个可驱动的旋转关节。可以设定目标角度、输出扭矩、锁定状态。常用于机械臂、转塔、方向控制等。

| 端口 | 方向 | 类型 | 物理子步 | 说明                                                                                       |
|------|------|------|:--------:|------------------------------------------------------------------------------------------|
| `target` | 输入 | REAL | ✅✅ | 目标角度（弧度）。写入后电机会尝试转到该弧度；在物理子步网络中可同一子步参与控制                                                 |
| `torque` | 输入 | REAL | ✅✅ | 前馈扭矩。**直接叠加**到 PID 控制器计算的扭矩之上（不是上限）。设为 0 表示无额外扭矩，PID 仍然正常工作；设为正/负值会增加/减少电机输出的净扭矩。同一子步即时生效 |
| `lock` | 输入 | BOOLEAN | ✅✅ | 锁定请求。`true` = 尝试把电机锁在当前角度附近；`false` = 解除锁定                                               |
| `angle` | 输出 | REAL | ✅ | 当前实际角度（弧度）                                                                                |
| `speed` | 输出 | REAL | ✅ | 当前转速（弧度/秒）                                                                                |
| `connected` | 输出 | BOOLEAN | ✅ | 电机是否已连接到关节                                                                               |
| `locked` | 输出 | BOOLEAN | ✅ | 电机当前是否处于锁定状态                                                                             |

**关于 `lock`**：锁定不是瞬间刹停。它会在接下来的游戏刻应用，因此通常有最多约 50ms 的延迟；如果短时间内反复切换，最后一次请求最重要。

设备类型 ID：`synaxis:plant/dynamic_motor`

---

### 10.4 Hydraulic Linear Actuator（液压直线执行器）

> 方块外观：一个可伸缩的直线执行器。用于控制两体之间的距离。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `target` | 输入 | REAL | ✅✅ | 目标伸缩距离。写入后执行器会尝试伸缩到该距离；在物理子步网络中可同一子步参与控制 |
| `lock` | 输入 | BOOLEAN | ✅✅ | 锁定请求。行为同 Dynamic Motor 的 `lock`：会在接下来的游戏刻应用 |
| `distance` | 输出 | REAL | ✅ | 当前实际距离 |
| `speed` | 输出 | REAL | ✅ | 当前伸缩速度 |
| `connected` | 输出 | BOOLEAN | ✅ | 执行器是否已连接到两端 |
| `locked` | 输出 | BOOLEAN | ✅ | 是否处于锁定状态 |

设备类型 ID：`synaxis:plant/hydraulic_linear_actuator`

---

### 10.5 Jet（喷气引擎）

> 方块外观：一个可产生推力的喷气引擎。支持矢量推力（可偏转推力方向）。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `thrust` | 输入 | REAL | ✅✅ | 推力大小。写入后喷气引擎按该值输出推力；在物理子步网络中可同一子步生效 |
| `horizontal` | 输入 | REAL | ✅✅ | 水平矢量偏转角（度）。同一子步即时生效 |
| `vertical` | 输入 | REAL | ✅✅ | 垂直矢量偏转角（度）。同一子步即时生效 |
| `current_thrust` | 输出 | REAL | ✅ | 当前推力值 |
| `current_horizontal` | 输出 | REAL | ✅ | 当前水平偏转角 |
| `current_vertical` | 输出 | REAL | ✅ | 当前垂直偏转角 |
| `vectoring_enabled` | 输出 | BOOLEAN | ✅ | 矢量推力模式是否启用。启用时 `horizontal`/`vertical` 输入才会生效 |

设备类型 ID：`synaxis:plant/jet`

---

### 10.6 Propeller（螺旋桨）

> 方块外观：一个可旋转的螺旋桨。需要安装桨叶才能产生推力。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `target_speed` | 输入 | REAL | ✅✅ | 目标转速。写入后螺旋桨会按该转速调整推力和扭矩；在物理子步网络中可同一子步生效 |
| `current_speed` | 输出 | REAL | ✅ | 当前采用的目标转速 |
| `attached` | 输出 | BOOLEAN | ✅ | 是否已安装桨叶。没有桨叶时螺旋桨不产生推力 |
| `thrust_ratio` | 输出 | REAL | ✅ | 当前推力与最大推力的比率（0–1） |
| `torque_ratio` | 输出 | REAL | ✅ | 当前扭矩与最大扭矩的比率（0–1） |

设备类型 ID：`synaxis:plant/propeller`

---

### 10.7 Kinetic Resistor（转速阻尼器）

> 方块外观：一个可调节传动比的变速装置。输入轴和输出轴的转速比由 `ratio` 端口控制。你可以把它理解为可编程的齿轮箱——输入转速 × 变速比 = 输出转速。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `ratio` | 输入 | REAL | ✅✅ | 变速比。写入后同一物理子步即时生效。ratio=1.0 表示直连，>1 加速，<1 减速 |
| `source_speed` | 输出 | REAL | ✅ | 输入轴转速 |
| `applied_ratio` | 输出 | REAL | ✅ | 实际生效的变速比（可能因限制而与设定值不同） |
| `output_speed` | 输出 | REAL | ✅ | 输出轴转速（= source_speed × applied_ratio） |

设备类型 ID：`synaxis:plant/kinetic_resistor`

---

### 10.8 Camera（摄像机）

> 方块外观：一个可旋转的摄像机。可以发射射线采样、追踪方块/机体/实体/玩家，输出目标的位置、速度、距离等信息。

**所有输入端口均为上升沿触发**：你需要先写入 `false`，然后写入 `true` 才会触发一次操作。持续写入 `true` 不会重复触发。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `sample` | 输入 | BOOLEAN | ✅✅ | 上升沿触发一次射线采样。采样结果更新 `hit`、`distance` 输出 |
| `track_block` | 输入 | BOOLEAN | ✅✅ | 上升沿锁定追踪视线附近的**方块** |
| `track_body` | 输入 | BOOLEAN | ✅✅ | 上升沿锁定追踪视线附近的**机体** |
| `track_entity` | 输入 | BOOLEAN | ✅✅ | 上升沿锁定追踪视线附近的**实体** |
| `track_player` | 输入 | BOOLEAN | ✅✅ | 上升沿锁定追踪视线附近的**玩家** |
| `clear_target` | 输入 | BOOLEAN | ✅✅ | 上升沿**清除**当前追踪目标 |
| `active` | 输出 | BOOLEAN | ✅ | 摄像机是否处于激活状态（被红石信号或手动开启） |
| `hit` | 输出 | BOOLEAN | ✅ | 最近一次射线采样是否命中了目标 |
| `distance` | 输出 | REAL | ✅ | 最近一次射线采样的命中距离 |
| `local_yaw` | 输出 | REAL | ✅ | 摄像机视线在**机体本地坐标系**中的 yaw 角度。在「跟随机体」模式下，此值等于玩家鼠标输入的 yaw，不随 body 旋转变化；在「世界稳定」模式下，body 旋转时此值会变化（因为同一世界方向映射到不同的本地角度） |
| `local_pitch` | 输出 | REAL | ✅ | 同上，摄像机视线在机体本地坐标系中的 pitch 角度 |
| `abs_forward` | 输出 | VEC3 | ✅ | 摄像机在世界坐标系中的绝对前向向量 |
| `target_valid` | 输出 | BOOLEAN | ✅ | 当前追踪目标是否有效 |
| `target_kind` | 输出 | REAL | ✅ | 追踪目标种类编码：0=无目标，1=方块，2=机体，3=实体，4=玩家。**更新频率因类型不同**：机体/方块目标可按物理子步更新；实体/玩家目标按游戏刻更新 |
| `target_position` | 输出 | VEC3 | ✅ | 追踪目标在世界坐标系中的位置 |
| `target_velocity` | 输出 | VEC3 | ✅ | 追踪目标的线速度 |
| `target_hit_position` | 输出 | VEC3 | ✅ | 射线命中点的世界坐标 |
| `target_distance` | 输出 | REAL | ✅ | 到追踪目标的距离 |
| `target_omega` | 输出 | VEC3 | ✅ | 追踪目标的角速度 |

**追踪行为的更新机制**：`track_*` 锁定目标后，目标数据的刷新频率取决于你追踪的是什么：
- **机体/方块**：在物理子步网络中运行时，这些输出可以按物理子步频率更新，适合高速瞄准或拦截控制
- **实体/玩家**：目标数据按游戏刻刷新。即使你的 Luacuit 在物理子步中运行，实体/玩家目标数据也以 20Hz 左右刷新
- `sample` 射线采样在收到上升沿信号的当即执行，结果立即反映在 `hit`、`distance` 输出上

设备类型 ID：`synaxis:plant/camera`

---

### 10.9 Tweakerminal（手柄终端）

> 方块外观：一个可链接手柄控制器的终端。玩家用手柄操作时，摇杆和按钮状态通过此设备暴露给 Cimulink。

**手柄输入如何变成信号**：

1. 玩家操作手柄
2. Create Tweaked Controllers 每客户端游戏刻（20 Hz，每 50ms）把摇杆和按钮状态同步到服务器
3. 下一次 Cimulink 评估时，Luacuit 通过 `bus.read` 读取最近一次同步到服务器的手柄状态

**典型延迟**：
- 客户端发送间隔：50ms（20 Hz）
- 到 Luacuit 可观测：需等到下一个 Cimulink 评估周期；游戏刻网络最多再等 50ms
- **总延迟：约 50–100ms**

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `lx` | 输出 | REAL | ✅ | 左摇杆 X 轴。范围约 [-1, 1] |
| `ly` | 输出 | REAL | ✅ | 左摇杆 Y 轴 |
| `rx` | 输出 | REAL | ✅ | 右摇杆 X 轴 |
| `ry` | 输出 | REAL | ✅ | 右摇杆 Y 轴 |
| `lt` | 输出 | REAL | ✅ | 左扳机。范围 [0, 1] |
| `rt` | 输出 | REAL | ✅ | 右扳机。范围 [0, 1] |
| `linked` | 输出 | BOOLEAN | ✅ | 是否已成功链接手柄 |
| `b<N>` | 输出 | BOOLEAN | ✅ | 按钮状态。`b0`、`b1`、`b2`...，索引从 0 开始 |

> **只读设备**（无输入端口）。按钮数量由 Create Tweaked Controllers 模组定义。

设备类型 ID：`synaxis:plant/tweakerminal`

---

### 10.10 Create Speed Controller（兼容）

> 来源：Create 模组。方块外观：Create 的转速控制器。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `target_speed` | 输入 | REAL | ❌ | 目标转速。写入后改变 Create 转速控制器显示/采用的目标转速；只在游戏刻网络中有效 |
| `current_speed` | 输出 | REAL | ❌ | 当前目标转速的读数。**注意**：这是目标值，不是实际转速 |

**重要限制**：此设备**不支持物理子步**读写。包含它的 Cimulink 网络只能在游戏刻运行，因此转速变化通常要到下一个游戏刻才会被 Create 动力系统看到。

设备类型 ID：`synaxis:plant/create_speed_controller`

---

### 10.11 Create Speedometer（兼容）

> 来源：Create 模组。方块外观：Create 的转速表。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `speed` | 输出 | REAL | ❌ | Create 动力网络中的当前转速（带符号，方向相关） |
| `abs_speed` | 输出 | REAL | ❌ | 转速的绝对值 |
| `dial_target` | 输出 | REAL | ❌ | 转速表指针的目标角度 |
| `dial_state` | 输出 | REAL | ❌ | 转速表指针的当前状态（平滑动画中的实际角度） |
| `overstressed` | 输出 | BOOLEAN | ❌ | Create 动力网络是否过载 |

**重要限制**：**仅游戏刻有效**。只读设备。读数跟随 Create 动力系统刷新，通常约 20 Hz。

设备类型 ID：`synaxis:plant/create_speedometer`

---

### 10.12 Simulated Gimbal Sensor（兼容）

> 来源：Simulated 模组。方块外观：一个可检测倾斜角度的陀螺仪传感器。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `pitch` | 输出 | REAL | ❌ | 绕 X 轴的倾斜角度（度） |
| `roll` | 输出 | REAL | ❌ | 绕 Z 轴的倾斜角度（度） |

**重要限制**：**仅游戏刻有效**。只读设备。读数通常按游戏刻刷新，约 20 Hz。

设备类型 ID：`synaxis:plant/simulated_gimbal_sensor`

---

### 10.13 Simulated Nav Table（兼容）

> 来源：Simulated 模组。方块外观：一个导航台，可设定目标位置并计算相对方位。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `has_target` | 输出 | BOOLEAN | ❌ | 是否已设定导航目标 |
| `relative_angle` | 输出 | REAL | ❌ | 从自身到目标方向的相对角度（弧度） |
| `self_position` | 输出 | VEC3 | ❌ | 自身的投影位置（世界坐标） |
| `target_position` | 输出 | VEC3 | ❌ | 目标位置（世界坐标） |
| `distance` | 输出 | REAL | ❌ | 自身到目标的距离 |

**重要限制**：**仅游戏刻有效**。只读设备。读数通常按游戏刻刷新，约 20 Hz。

设备类型 ID：`synaxis:plant/simulated_navigation_table`

---

### 10.14 Simulated Swivel Bearing（兼容）

> 来源：Simulated 模组。方块外观：一个旋转轴承。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `angle` | 输出 | REAL | ❌ | 轴承的当前目标角度（弧度） |
| `assembled` | 输出 | BOOLEAN | ❌ | 轴承是否已组装 |

**重要限制**：**仅游戏刻有效**。只读设备。读数通常按游戏刻刷新，约 20 Hz。

设备类型 ID：`synaxis:plant/simulated_swivel_bearing`

---

### 10.15 Simulated Velocity Sensor（兼容）

> 来源：Simulated 模组。方块外观：一个速度传感器，检测其检测面方向的运动速度。

| 端口 | 方向 | 类型 | 物理子步 | 说明 |
|------|------|------|:--------:|------|
| `velocity` | 输出 | REAL | ❌ | 经过调整的检测速度（考虑 `maxSpeed` 缩放） |
| `axis` | 输出 | VEC3 | ❌ | 传感器检测面的法线方向向量 |

**重要限制**：**仅游戏刻有效**。只读设备。读数通常按游戏刻刷新，约 20 Hz。

设备类型 ID：`synaxis:plant/simulated_velocity_sensor`

---

### 领域兼容性速查

| PlantPort | 物理子步读取 | 物理子步写入 | 备注 |
|-----------|:----------:|:----------:|------|
| Compact Flap | ✅ | ✅✅ | 写入同一子步即时生效 |
| Control Chair | ✅ | — | 只读；客户端输入延迟 ~16–67ms |
| Dynamic Motor | ✅ | ✅✅ | `lock` 延迟到下次游戏刻消费 |
| Hydraulic Actuator | ✅ | ✅✅ | `target` 同一子步生效 |
| Jet | ✅ | ✅✅ | 推力同一子步生效 |
| Propeller | ✅ | ✅✅ | 转速同一子步生效 |
| Kinetic Resistor | ✅ | ✅✅ | 变速比同一子步生效 |
| Camera | ✅ | ✅✅ | 追踪目标以 20 Hz 更新 |
| Tweakerminal | ✅ | — | 只读；手柄输入延迟 ~50–100ms |
| Create Speed Controller | ❌ | ❌ | 仅游戏刻 |
| Create Speedometer | ❌ | ❌ | 仅游戏刻；只读 |
| Simulated Gimbal Sensor | ❌ | ❌ | 仅游戏刻；只读 |
| Simulated Nav Table | ❌ | ❌ | 仅游戏刻；只读 |
| Simulated Swivel Bearing | ❌ | ❌ | 仅游戏刻；只读 |
| Simulated Velocity Sensor | ❌ | ❌ | 仅游戏刻；只读 |

> **物理子步写入**（✅✅）：写入值在**同一物理子步**内被设备的物理控制器读取并施加力——无需等待游戏刻。
> **仅游戏刻**（❌）：包含这些设备的 Cimulink 网络无法在物理子步中运行，整个网络被归类为 `GAME_TICK`。

---

## 11. 示例脚本

> **重要**：Luacuit 为了执行 `define()` 会先运行一次脚本顶层代码。不要在顶层直接调用 `input` / `output` / `state` / `bus` / `phys` / `control` 等 API 来初始化对象。顶层只放普通 Lua 变量；需要 PID、低通滤波器等 API 对象时，在 `loop()` 中做一次懒初始化。

### 示例 1：简单信号传递

```lua
function define()
    defineInput.real("throttle", 0)
    defineInput.vec3("axis", 0, 1, 0)
    defineOutput.real("command")
end

function loop()
    local t = input.real("throttle")
    local axis = input.vec3("axis")
    output.real("command", t * axis:length())
end
```

### 示例 2：Warm VM 做平滑滤波器

```lua
local prev = 0

function loop()
    local raw = input.real("sensor")
    local smoothed = prev * 0.9 + raw * 0.1
    prev = smoothed
    output.real("filtered", smoothed)
end
```

### 示例 3：使用 PID 控制器

```lua
local pid = nil

function loop()
    if pid == nil then
        pid = control.pid({
            kp = 2.0,
            ki = 0.1,
            kd = 0.05,
            min = 0.0,
            max = 1.0,
        })
    end

    local target = input.real("target")
    local measured = input.real("feedback")
    local dt = 0.05
    local cmd = pid:update(target, measured, dt)
    output.real("command", cmd)
end
```

### 示例 4：从 Control Chair 读取轴并驱动 Flap

```lua
function define()
    defineOutput.real("flap_angle")
    defineOutput.real("debug_forward")
end

function loop()
    -- Control Chair 输出更新频率：玩家输入约 60Hz 同步到服务器
    -- 到 bus.read 可观测的延迟：约 16–67ms
    local lx = bus.read("Control Seat", "lx") or 0  -- yaw
    local ly = bus.read("Control Seat", "ly") or 0  -- thrust

    local fwd = bus.read("Control Seat", "abs_forward")
    if fwd then
        output.real("debug_forward", fwd:length())
    end

    -- Flap 物理子步写入：同一子步内即可影响舵面气动力
    bus.write("Compact Flap", "angle", lx)
    bus.write("Compact Flap", "tilt", ly)
    output.real("flap_angle", lx)
end
```

### 示例 5：从 Camera 追踪目标并计算相对位置

```lua
function define()
    defineInput.real("trigger", 0)
    defineOutput.vec3("target_delta")
    defineOutput.real("target_distance")
end

function loop()
    -- Camera 输入为上升沿触发：trigger 从 ≤0.5 变为 >0.5 时发送一次 sample
    -- 实体/玩家目标数据约 20Hz 刷新；机体/方块目标在物理子步网络中可更快刷新
    local trig = input.real("trigger")
    if trig > 0.5 then
        bus.write("Camera", "sample", true)
    end

    local valid = bus.read("Camera", "target_valid") or false
    if valid then
        local targetPos = bus.read("Camera", "target_position")
        local dist = bus.read("Camera", "target_distance") or 0
        local myPos = phys.position()

        if targetPos and myPos then
            local delta = targetPos:sub(myPos)
            output.vec3("target_delta", delta)
            output.real("target_distance", dist)
        end
    end
end
```

### 示例 6：使用 `state` 持久化计数器

```lua
function define()
    defineInput.real("pulse", 0)
    defineOutput.real("count")
end

function loop()
    local n = state.get("counter") or 0
    local pulse = input.real("pulse")
    if pulse > 0.5 then
        n = n + 1
        state.set("counter", n)
    end
    output.real("count", n)
end
```

### 示例 7：物理状态反馈控制

```lua
function define()
    defineInput.real("target_speed", 0)
    defineOutput.real("throttle_out")
end

function loop()
    local target = input.real("target_speed")
    local current = phys.velocity():length()
    local error = target - current
    local cmd = math.max(0, math.min(1, error * 0.1))
    output.real("throttle_out", cmd)
end
```

### 示例 8：使用 control.lowpass 做信号平滑

```lua
function define()
    defineInput.real("raw_sensor", 0)
    defineOutput.real("smoothed")
end

local lp = nil

function loop()
    if lp == nil then
        lp = control.lowpass({ tau = 0.3, initial = 0 })
    end

    local raw = input.real("raw_sensor")
    local dt = 0.05
    output.real("smoothed", lp:update(raw, dt))
end
```

### 示例 9：姿态稳定（用惯量张量）

```lua
function define()
    defineInput.vec3("target_up", 0, 1, 0)
    defineOutput.vec3("correction_torque")
end

function loop()
    local targetUp = input.vec3("target_up"):normalize()
    local q = phys.quaternionToWorld()

    local worldUp = Vector3d:new(0, 1, 0)
    local currentUp = q:transform(worldUp)

    local errorAngle = currentUp:angle(targetUp)
    local errorAxis = currentUp:cross(targetUp):normalize()

    local tensor = phys.inertiaTensor()
    local correction = tensor:transform(errorAxis:mul(errorAngle * 10))
    output.vec3("correction_torque", correction)
end
```

### 示例 10：多设备协同——Control Chair → PID → Motor + Jet

```lua
function define()
    defineOutput.real("debug_pitch")
    defineOutput.real("debug_roll")
end

local pitchPid = nil
local rollPid = nil

function loop()
    if pitchPid == nil then
        pitchPid = control.pid({ kp = 3.0, ki = 0.2, kd = 0.1, min = -1, max = 1 })
        rollPid = control.pid({ kp = 3.0, ki = 0.2, kd = 0.1, min = -1, max = 1 })
    end

    -- Control Chair: 玩家输入约 60Hz 同步到服务器
    local pitch = bus.read("Control Seat", "ry") or 0
    local roll  = bus.read("Control Seat", "rx") or 0
    local thrust = bus.read("Control Seat", "ly") or 0

    local omega = phys.angularVelocity()

    local dt = 0.05
    local pitchCmd = pitchPid:update(pitch, omega.y, dt)
    local rollCmd  = rollPid:update(roll, omega.x, dt)

    -- Flap/Jet 物理子步写入：同一子步即时生效
    bus.write("Left Flap",  "angle", -rollCmd)
    bus.write("Right Flap", "angle",  rollCmd)
    bus.write("Rear Flap",  "angle", pitchCmd)
    bus.write("Main Jet",   "thrust", math.max(0, thrust))

    output.real("debug_pitch", pitchCmd)
    output.real("debug_roll", rollCmd)
end
```

### 示例 11：使用 Camera 追踪 + Nav Table 导航

```lua
function define()
    defineInput.bool("lock_target", false)
    defineOutput.real("nav_distance")
    defineOutput.real("nav_angle")
end

local targetLocked = false

function loop()
    local trigger = input.bool("lock_target")

    -- Camera 上升沿触发机体追踪
    if trigger and not targetLocked then
        bus.write("Camera", "track_body", true)
        targetLocked = true
    elseif not trigger then
        targetLocked = false
    end

    -- Nav Table 数据约 20Hz 更新，仅游戏刻网络可用
    local hasTarget = bus.read("Nav Table", "has_target") or false
    if hasTarget then
        local dist = bus.read("Nav Table", "distance") or -1
        local angle = bus.read("Nav Table", "relative_angle") or 0
        output.real("nav_distance", dist)
        output.real("nav_angle", angle)
    end
end
```

---

> **版本**：基于 Synaxis Cimulink 当前代码库（截至报告生成时）。
> 如有新增 PlantPort 或 API 变更，请同步更新本文档。
