# Mean Reversion T (MR-T)

English · [简体中文](README.zh-CN.md)

一个**生产级的均值回归"做T"信号指标**,运行于 TradingView,Pine Script v6 编写。面向 **15 分钟图**与 **A 股 T+0 日内做T**。防重绘。运行时界面中英双语。

[![Pine Script](https://img.shields.io/badge/Pine%20Script-v6-yellow)](https://www.tradingview.com/pine-script-docs/)
[![License: MPL 2.0](https://img.shields.io/badge/License-MPL%202.0-brightgreen.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-2.0.0-informational)](CHANGELOG.md)

---

## 为什么需要 MR-T

多数均值回归指标在价格一偏离均值就触发信号,然后被"超卖其实只是趋势起点"的行情碾过去。MR-T 把均值回归当作**多条件、带计时器的生命周期**,而不是单一阈值:

- 只在**环境确实适合震荡**时才交易(1H 与 15m 两个周期的综合环境分)。
- 存在**硬趋势**时拒绝进场(斜率 + 效率比率双重否决)。
- 区分**温和漂移**(Range 引擎)与**正在衰竭的异常冲击**(Shock 引擎)。
- 用 **Ornstein-Uhlenbeck 半衰期**估计每笔交易的合理时长,长时间走不出结果就按时间止损,不任其流血。
- 跟踪**扣除成本后的虚拟盈亏**,让"保本"真正意味着扣掉手续费和滑点后的保本。

## 特性

| 能力 | 说明 |
|---|---|
| 双引擎 | Range 回归(温和 Z 偏离)+ Shock 反转(异常冲击 + 减速/拒绝) |
| 硬趋势否决 | 1H 与 15m 的斜率 + 效率比率守卫,拒绝趋势环境 |
| 半衰期计时 | OU 过程半衰期估计回归应耗时多久;无法估计时退化为固定值 |
| 成本感知盈亏 | 面板显示单笔/累计虚拟 P&L,已扣除往返成本 |
| 中英双语 | 面板 / 图表标签 / 报错随 *语言* 输入切换;警报消息与输入项标签为双语静态(Pine 把它们钉死在编译期 `const string`) |
| 防重绘 | 信号门控于已确认 K 线;高周期数据取上一根已确认的 1H bar;目标价进场时冻结 |
| 版本化 | 语义化版本(见 `CHANGELOG.md`),版本号显示在面板 |

---

## 工作原理

```mermaid
flowchart TD
    GATE["环境闸门 - 环境分(25x4)+ 硬趋势否决"]
    GATE --> CHECK{"环境合格?"}
    CHECK -- "否" --> NONE["不做"]
    CHECK -- "是" --> RANGE{"Range 引擎: z <= -Z_entry"}
    CHECK -- "是" --> SHOCK{"Shock 引擎: z <= -Z_shock + 减速 + 拒绝"}
    RANGE -- "是" --> RSETUP["Range 观察成立"]
    SHOCK -- "是" --> SSETUP["Shock 观察成立"]
    RSETUP --> CONFIRM["Z 回归确认 / K线 / 区间确认"]
    SSETUP --> CONFIRM
    CONFIRM --> ENTRY["T买 / T空"]
    ENTRY --> TRIM["T减 - 部分了结,止损移到保本"]
    TRIM --> CLOSE{"完全回归?"}
    CLOSE -- "是" --> CLOSED["T平"]
    CLOSE -- "否" --> EXITS["异常退出: T止损 / 趋势失效 / T超时 / T保本"]
```

### 生命周期(每笔)

`观察` → `T买 / T空` → `T减`(部分了结,止损上移保本)→ `T平`(完全回归)。
异常退出:`T止损`(统计或 ATR 风险线)、`趋势失效`(市场转为趋势)、`T超时`(超过半衰期)、`T保本`(首目标后触发)。

---

## 环境要求

- **Pine Script v6** 编辑器
- **15 分钟图**(脚本强制校验,其它周期直接报错;这是刻意的契约,见 *局限*)
- 一个用于市场状态的高周期(默认 **1H**)
- 足够的历史数据完成预热(`meanLen` + `zLen` 约 64 根,半衰期约 80 根)

## 安装

1. 打开 TradingView Pine 编辑器,粘贴 [`MRT.pine`](MRT.pine) 内容,*添加到图表*。
2. 或 clone 本仓库,用编辑器打开 `MRT.pine`。

> 它是 **indicator**,不是 strategy —— 不会下单。请用它的警报辅助你自己的决策。

---

## 使用

- **市场**:主要用于 A 股 T+0 日内做T,15 分钟周期。换品种 / 周期前必须**重新评估全部参数**。
- **语言**:将 *⑫ 语言与版本 → 语言* 设为 `zh` 或 `en`,切换面板、图表标签与报错。警报消息与输入项*标签*为双语静态文本(Pine 限制,见 *局限*)。
- **成本**:将 *⑩ 风险控制 → 每边成本(bp)* 设为你真实的往返成本(佣金 + 滑点 + 印花税,按每边)。默认 `3.0 bp/边` 是保守估计;仅影响虚拟盈亏与保本偏移。
- **警报**:内置 10 个 `MR-T ...` 警报条件——在警报面板新建并选择对应条件即可。

---

## 参数对照表

输入项标签为中文(Pine 的 input 标题要求编译期 `const string`,无法用运行时开关本地化)。下表为英文对照。

### ① 交易方向
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `allowLong` | bool | true | 允许 T买 |
| `allowShort` | bool | true | 允许 T空 |

### ② 均值与 Z-score
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `meanLen` | int | 32 | 15m 局部均值 EMA 长度 |
| `zLen` | int | 32 | Z-score 窗口 |
| `rangeEntryZ` | float | 1.65 | Range 模式观察 Z |
| `shockEntryZ` | float | 2.00 | Shock 模式观察 Z |
| `rangePartialZ` | float | 0.80 | Range T减目标 Z |
| `rangeExitZ` | float | 0.25 | Range T平目标 Z |
| `shockPartialZ` | float | 1.00 | Shock T减目标 Z |
| `shockExitZ` | float | 0.50 | Shock T平目标 Z |
| `stopZ` | float | 3.25 | 极端统计失效 Z |

### ③ 1H 市场状态
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `htfTf` | timeframe | 60 | 市场状态高周期 |
| `htfMeanLen` | int | 20 | 1H 均值 EMA 长度 |
| `htfSlopeLookback` | int | 4 | 1H 斜率周期 |
| `idealHtfSlope` | float | 0.10 | 1H "理想"最大斜率(ATR/Bar) |
| `htfERLen` | int | 10 | 1H 效率比率周期 |
| `idealHtfER` | float | 0.50 | 1H "理想"最大 ER |

### ④ 15m 市场状态
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `localSlopeLookback` | int | 8 | 15m 斜率周期 |
| `idealLocalSlope` | float | 0.12 | 15m "理想"最大斜率(ATR/Bar) |
| `localERLen` | int | 16 | 15m 效率比率周期 |
| `idealLocalER` | float | 0.60 | 15m "理想"最大 ER |

### ⑤ Regime Score
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `rangeMinScore` | float | 50 | Range 模式最低环境分 |
| `shockMinScore` | float | 45 | Shock 模式最低环境分 |
| `regimeSmoothLen` | int | 3 | 环境分平滑周期 |

### ⑥ Hard Trend Veto
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `vetoHtfSlope` | float | 0.18 | 1H 斜率否决阈值 |
| `vetoLocalSlope` | float | 0.23 | 15m 斜率否决阈值 |
| `vetoHtfER` | float | 0.75 | 1H ER 否决阈值 |
| `vetoLocalER` | float | 0.80 | 15m ER 否决阈值 |

### ⑦ Shock Engine
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `shockLookback` | int | 2 | 冲击测量 K 数 |
| `minShockATR` | float | 0.80 | 最小冲击强度(ATR) |
| `requireShockDeceleration` | bool | true | 要求冲击减速 |
| `decelerationRatio` | float | 0.85 | 减速比例(当前 vs 前一段) |
| `requireRejection` | bool | true | 要求拒绝/反向 K 线 |
| `minWickRatio` | float | 0.30 | 最低拒绝影线比例 |

### ⑧ Half-Life / Time Stop
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `halfLifeLookback` | int | 80 | 半衰期样本长度 |
| `timeStopMultiplier` | float | 3.0 | Time Stop = 半衰期 × N |
| `minTimeStopBars` | int | 8 | 最短 Time Stop(根) |
| `maxTimeStopBars` | int | 40 | 最长 Time Stop(根) |
| `fallbackTimeStopBars` | int | 24 | 半衰期无效时的 Time Stop |

### ⑨ Setup / 确认
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `rangeSetupBars` | int | 16 | Range Setup 有效 K 数 |
| `shockSetupBars` | int | 8 | Shock Setup 有效 K 数 |
| `requireCandleConfirm` | bool | true | 要求 K 线方向确认 |
| `requireBandReclaim` | bool | true | 要求价格重新进入 Z 区间 |
| `cooldownBars` | int | 3 | 退出后冷却 K 数 |

### ⑩ 风险控制
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `atrLen` | int | 14 | ATR 周期 |
| `hardStopATR` | float | 2.50 | ATR 紧急止损 |
| `stopMode` | string | Balanced | `Tight` / `Balanced` / `Loose` 止损模式 |
| `balancedWeight` | float | 0.50 | 统计止损与 ATR 止损的权重(0=宽松,1=收紧) |
| `moveStopToBE` | bool | true | T减后启用保本 |
| `costBps` | float | 3.0 | 每边往返成本(bp)——影响盈亏与保本偏移 |

### ⑪ 显示 / ⑫ 语言
| 输入 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `showBands` | bool | true | 显示均值回归区间 |
| `showBackground` | bool | true | 显示做T环境 |
| `showSetup` | bool | true | 显示观察信号 |
| `showPanel` | bool | true | 显示状态面板 |
| `showTradeLevels` | bool | true | 显示本轮目标/风险线 |
| `lang` | string | zh | `zh` / `en` 运行时界面语言 |

---

## 成本模型与盈亏面板

面板(右上角)显示最近一笔已平仓交易与累计:

- **本笔 P&L** —— 最近一笔交易的净盈亏,以 `ATR` 和 `%` 表示,已扣除 `2 × costBps` 往返成本。
- **累计 P&L** —— 累计净盈亏(ATR)与已平虚拟交易笔数。

T减后的保本位放在 `entry ± 往返成本`,因此"T保本"是真正扣掉成本后的打平。

**模型的诚实局限:**

- 退出按**收盘价**判定,不是盘中触发——真实止损可能在单根 K 线内被击穿又收回;本指标按收盘报告。
- 盈亏按**全额仓位**从入场到出场计算,忽略了 T减这一步的仓位权重。
- 成本是固定 bp 估算,真实成交会有出入。把面板当作 sanity 信号,不是记账系统。

---

## 防重绘

- 所有信号 / 事件写入都门控于 `barstate.isconfirmed`。
- 高周期数据取**上一根已确认的 1H bar**(`f_erPrev` / `f_slopeATRPrev` + `barmerge.lookahead_on`)——无未来数据泄漏。
- 目标价 / 止损**进场时冻结**(`MRState` 中的交易上下文)。
- 半衰期估计只用历史数据。

## 局限与边界

- **硬编码 15 分钟契约。** 脚本拒绝在其它周期运行。这是刻意的(窗口长度以 K 数计,且是在 15m 上调出来的);换周期必须重新调参。
- **输入项标签与警报消息为双语静态文本。** Pine 的 input 标题与 `alertcondition` 消息是编译期 `const string`,运行时语言开关无法本地化,故以中英双语写出。面板、图表标签与报错随 `lang` 输入切换。
- **A 股 T+0 偏好。** 引擎假设针对持有仓位做日内回归。不做重调参就套到趋势性强的加密 / 外汇上,会很失望。
- **不构成投资建议。** 不做任何明示或暗示的业绩承诺。

---

## 贡献

欢迎 issue 与 PR。请保持向后兼容,保持状态机集中在 `MRState`,每次行为变更都同步更新 `CHANGELOG.md` 与 `SCRIPT_VERSION`。

## 许可证

本 Pine Script 代码遵循 **Mozilla Public License 2.0** 条款,见 <https://mozilla.org/MPL/2.0/>。全文见 [LICENSE](LICENSE)。

## 作者

**Jasxu** - [TradingView](https://www.tradingview.com/u/Jasxu/) · [GitHub](https://github.com/Ye-Yu-Mo)

## 免责声明

本软件仅用于**教育与信息目的**。不构成投资建议;历史或模拟表现不保证未来结果。交易有风险,请自行对决策负责。
