# Divergence ATR Strategy 代码设计文档

## 1. 项目概述

### 1.1 策略简介
Divergence ATR Strategy 是一个基于多指标背离检测的量化交易策略，结合了ATR动态止损止盈机制和高级趋势跟随逻辑。该策略使用Pine Script v6开发，适用于TradingView平台。

### 1.2 核心特性
- **多指标背离检测**：支持10种技术指标的背离分析 (MACD、RSI、CCI、OBV等)
- **动态止损止盈**：基于ATR的自适应风险管理系统
- **趋势跟随机制**：5分钟级别MACD+RSI三重验证
- **灵活交易方向**：支持只做多、只做空、双向交易配置
- **入场价格保护**：将入场价格作为保护性止损的创新机制

### 1.3 技术规格
- **语言版本**：Pine Script v6
- **交易平台**：TradingView
- **策略类型**：多指标背离 + 趋势跟随混合策略
- **风险管理**：ATR动态止损 + 多层级风险控制

## 2. 系统架构设计

### 2.1 分层架构
```
┌─────────────────────────────────────────────┐
│               用户界面层                      │
│  参数配置 | 图形显示 | 信息表格 | 交易信号      │
├─────────────────────────────────────────────┤
│               业务逻辑层                      │
│  信号生成 | 风险管理 | 交易执行 | 状态管理      │
├─────────────────────────────────────────────┤
│               算法核心层                      │
│  背离检测 | 趋势分析 | ATR计算 | 价位算法      │
├─────────────────────────────────────────────┤
│               数据处理层                      │
│  指标计算 | 枢轴检测 | 数组管理 | 历史数据      │
└─────────────────────────────────────────────┘
```

### 2.2 核心模块

#### 2.2.1 参数配置模块 (Lines 5-47)
**职责**：管理策略的所有可配置参数
**关键组件**：
- 枢轴点检测参数 (周期、数据源、背离类型)
- ATR风险控制参数 (长度、平滑方式、倍数)
- 交易方向控制 (多头/空头/双向)
- 趋势分析配置 (启用状态、检查周期)
- 多指标开关 (10个指标的独立控制)

#### 2.2.2 技术指标计算模块 (Lines 48-89)
**职责**：计算策略所需的技术指标
**核心指标**：
```
基础指标：MACD(12,26,9), RSI(14), ATR(14)
辅助指标：CCI(10), 动量(10), 随机指标(14)
成交量指标：OBV, VW-MACD, CMF(21), MFI(14)
```

#### 2.2.3 高级趋势分析模块 (Lines 90-136)
**职责**：5分钟级别趋势跟随逻辑
**核心算法**：
```
趋势验证 = MACD斜率 AND MACD直方图持续性 AND RSI同向变化
多头确认：向上斜率 + 直方图增高 + RSI上升
空头确认：向下斜率 + 直方图降低 + RSI下降
```

#### 2.2.4 背离检测模块 (Lines 137-300)
**职责**：多指标背离模式识别
**检测类型**：
- 正向常规背离：指标新高 + 价格新低 → 看涨
- 负向常规背离：指标新低 + 价格新高 → 看跌
- 正向隐藏背离：指标新低 + 价格新高 → 看涨
- 负向隐藏背离：指标新高 + 价格新低 → 看跌

## 3. 核心算法详解

### 3.1 背离检测算法

#### 3.1.1 算法流程
```
1. 枢轴点识别
   ├── 使用ta.pivothigh/ta.pivotlow检测转折点
   ├── 根据prd参数确定检测敏感度
   └── 存储最近20个枢轴点的位置和价格

2. 背离模式匹配
   ├── 遍历历史枢轴点(最多maxpp个)
   ├── 检查距离限制(不超过maxbars根K线)
   ├── 验证指标和价格的反向关系
   └── 确保最小距离要求(len > 5)

3. 连接线验证
   ├── 计算指标连接线斜率
   ├── 计算价格连接线斜率  
   ├── 验证中间所有点都在连接线正确一侧
   └── 确保背离的连续性和有效性

4. 背离强度评估
   ├── 记录背离长度作为强度指标
   ├── 按类型分类存储到背离数组
   └── 统计总背离数量进行信号过滤
```

#### 3.1.2 关键函数设计
```pinescript
// 正向背离检测函数
positive_regular_positive_hidden_divergence(src, cond) =>
    // cond=1: 常规背离, cond=2: 隐藏背离
    // 返回背离长度，0表示无背离

// 负向背离检测函数  
negative_regular_negative_hidden_divergence(src, cond) =>
    // 逻辑与正向相反，检测下降趋势中的背离

// 综合背离计算
calculate_divs(cond, indicator) =>
    // 返回包含4种背离类型的数组
    // [正向常规, 负向常规, 正向隐藏, 负向隐藏]
```

### 3.2 趋势跟随算法

#### 3.2.1 三重验证机制
```pinescript
check_trend_continuation() =>
    if not enable_trend_analysis
        true  // 未启用时直接通过
    else
        // 1. MACD斜率方向检查
        macd_slope_bullish = macd_line_5m > macd_line_5m[1]
        macd_slope_bearish = macd_line_5m < macd_line_5m[1]
        
        // 2. MACD直方图持续性检查
        检查最近macd_trend_length根K线
        确认直方图持续增高/降低
        
        // 3. RSI同步变化检查
        检查最近rsi_trend_length根K线
        确认RSI与交易方向一致变化
        
        // 综合判断
        多头：三个条件都为true
        空头：三个条件都为true
        无仓：直接允许开仓
```

### 3.3 动态风险管理算法

#### 3.3.1 ATR计算体系
```pinescript
// ATR计算流程
ma_function(source_data, length) =>
    // 支持4种移动平均算法
    switch atr_smoothing
        "RMA" => ta.rma(source_data, length)  // 威尔德平滑
        "SMA" => ta.sma(source_data, length)  // 简单平均
        "EMA" => ta.ema(source_data, length)  // 指数平均
        "WMA" => ta.wma(source_data, length)  // 加权平均

// ATR值和边界计算
atr_value = ma_function(ta.tr(true), atr_length) * atr_multiplier
atr_upper = high + atr_value  // 阻力位参考
atr_lower = low - atr_value   // 支撑位参考
```

#### 3.3.2 最近价位算法
```pinescript
// 核心思想：找到距离当前价格最近的有效止盈/止损位
get_nearest_take_profit() =>
    遍历take_profit_history数组
    多头：找最近的未达到止盈位(最小值)
    空头：找最近的未达到止盈位(最大值)
    返回最优止盈位或na

get_nearest_stop_loss() =>
    遍历stop_loss_history数组
    多头：找最近的可触发止损位(最大值，包括入场价)
    空头：找最近的可触发止损位(最小值，包括入场价)
    返回最优止损位或na
```

## 4. 数据结构设计

### 4.1 状态管理变量
```pinescript
// 核心交易状态 (使用var关键字保持状态)
var float entry_price = na           // 入场价格记录
var float entry_stop_loss = na       // 初始ATR止损位
var float entry_take_profit = na     // 初始ATR止盈位
var float long_stop_loss = na        // 多头动态止损位
var float long_take_profit = na      // 多头动态止盈位
var float short_stop_loss = na       // 空头动态止损位
var float short_take_profit = na     // 空头动态止盈位
```

### 4.2 历史数据数组
```pinescript
// 枢轴点历史存储 (循环数组，最大20个元素)
var ph_positions = array.new<int>(20, 0)    // 高点K线位置
var pl_positions = array.new<int>(20, 0)    // 低点K线位置  
var ph_vals = array.new<float>(20, 0.)      // 高点价格值
var pl_vals = array.new<float>(20, 0.)      // 低点价格值

// 动态止盈止损历史 (动态扩展数组)
var take_profit_history = array.new<float>(0)  // 所有止盈位记录
var stop_loss_history = array.new<float>(0)    // 所有止损位记录(含入场价)
```

### 4.3 背离数据存储
```pinescript
// 多指标背离数据矩阵 (10指标 × 4背离类型 = 40元素)
var all_divergences = array.new<int>(40)

// 数据映射关系
指标索引 × 4 + 背离类型 = 数组位置
例如：MACD主线(0) × 4 + 正向常规(0) = 位置0
     RSI(2) × 4 + 负向隐藏(3) = 位置11
```

## 5. 业务流程设计

### 5.1 信号生成流程
```
指标计算 → 背离检测 → 信号汇总 → 方向过滤 → 最终信号
    ↓         ↓         ↓         ↓         ↓
MACD/RSI等  40维数组   4类背离    交易方向   开仓条件
                        ↓
                    数量过滤(showlimit)
```

### 5.2 风险管理流程  
```
开仓时刻 → ATR计算 → 初始SL/TP → 历史数组 → 入场价保护
                                   ↓
持仓期间 → 趋势检查 → 最近价位 → 平仓检查 → 执行平仓
    ↓         ↓         ↓         ↓         ↓
实时监控   三重验证   算法优选   优先级制   状态重置
```

### 5.3 平仓优先级设计
```
优先级1: 趋势反转检查 (check_trend_continuation() = false)
优先级2: 最近止盈位触及 (close >= nearest_take_profit)  
优先级3: 最近止损位触及 (close <= nearest_stop_loss)
优先级4: 传统止损止盈 (备用机制)
```

## 6. 关键业务逻辑

### 6.1 开仓逻辑
```pinescript
// 多头开仓条件
if filtered_long_signal and strategy.position_size == 0
    strategy.entry("Long", strategy.long)
    
    // 风险参数设置
    entry_price := close
    entry_stop_loss := low - atr_value      // ATR止损
    entry_take_profit := close + atr_value  // ATR止盈
    
    // 历史数组初始化
    array.clear(take_profit_history)
    array.clear(stop_loss_history)
    array.push(take_profit_history, entry_take_profit)
    array.push(stop_loss_history, entry_stop_loss)
    array.push(stop_loss_history, entry_price)  // 关键：入场价作为保护
```

### 6.2 智能平仓逻辑
```pinescript
// 多层级平仓检查
if strategy.position_size > 0  // 多头持仓
    nearest_tp := get_nearest_take_profit()
    nearest_sl := get_nearest_stop_loss()
    
    // 第一优先级：趋势反转
    if not check_trend_continuation()
        strategy.close("Long", comment="Trend Reversal Exit")
        
    // 第二优先级：智能止盈
    else if not na(nearest_tp) and close >= nearest_tp
        strategy.close("Long", comment="Nearest TP Hit")
        
    // 第三优先级：智能止损
    else if not na(nearest_sl) and close <= nearest_sl
        strategy.close("Long", comment="Nearest SL Hit")
```

## 7. 性能优化策略

### 7.1 计算效率优化
- **条件计算**：使用enable_trend_analysis控制是否进行趋势计算
- **指标开关**：10个背离指标独立开关，按需计算
- **数组限制**：限制历史数据数组大小防止内存泄漏
- **更新频率**：信息表格仅在barstate.islast时更新

### 7.2 显示性能优化
```pinescript
// 条件显示减少不必要的绘制
plot(strategy.position_size > 0 ? long_stop_loss : na, ...)
plotshape(ph_detected and showpivot, ...)

// 数组管理优化
if array.size(ph_positions) > maxarraysize
    array.pop(ph_positions)  // 及时清理超出限制的数据
```

## 8. 错误处理与边界条件

### 8.1 数据有效性检查
```pinescript
// 空值验证
ph_detected = not na(ph)
pl_detected = not na(pl)

// 数组边界检查  
if array.get(pl_positions, x) == 0 or len > maxbars
    break  // 防止无效数据处理

// 背离数量验证
if total_div < showlimit
    array.fill(all_divergences, 0)  // 信号不足时清零
```

### 8.2 异常状态处理
```pinescript
// 趋势分析失效处理
if not enable_trend_analysis
    true  // 禁用时直接通过验证

// 价位计算异常处理
if na(nearest_tp) or na(nearest_sl)
    // 使用传统止损止盈作为备用机制
```

## 9. 测试验证建议

### 9.1 单元测试重点
1. **背离检测准确性**：测试各种背离模式的识别
2. **趋势验证逻辑**：验证三重验证机制的有效性  
3. **ATR计算精度**：确保风险控制参数的准确性
4. **最近价位算法**：验证智能止盈止损的正确性

### 9.2 集成测试策略
1. **多市场回测**：在不同品种上验证策略稳定性
2. **参数敏感性**：测试关键参数变化对策略的影响
3. **极端行情测试**：验证策略在异常市况下的表现
4. **实时性测试**：确保信号生成和执行的及时性

## 10. 部署运维指南

### 10.1 部署配置
1. **参数调优**：根据交易品种特性调整ATR倍数、背离周期等
2. **风险设置**：配置最大仓位、单笔损失限制
3. **监控告警**：设置关键指标异常提醒
4. **权限管理**：配置交易权限和操作日志

### 10.2 运维监控指标
- **信号质量**：背离信号生成频率和准确率
- **风险控制**：止损止盈触发率和及时性
- **趋势跟随**：趋势反转检测的有效性
- **系统稳定**：策略运行连续性和异常率

## 11. 扩展性设计

### 11.1 功能扩展方向
1. **新增指标**：支持更多技术指标的背离检测
2. **机器学习**：引入ML模型进行信号过滤优化
3. **多时间框架**：支持不同周期的组合分析
4. **情绪指标**：整合市场情绪和资金流向数据

### 11.2 架构扩展能力
- **模块化设计**：各功能模块相对独立，便于单独优化
- **参数可配**：所有关键逻辑都支持参数化配置
- **接口预留**：为新功能预留了扩展接口
- **数据兼容**：数据结构设计考虑了向前兼容性

---
