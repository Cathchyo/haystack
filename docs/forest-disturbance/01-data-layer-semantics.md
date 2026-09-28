# 森林扰动数据图层语义与斑块分类规则

> 版本：1.2  
> 更新日期：2026-09-25  
> 状态：图层定义已确认，斑块分类采用暂行规则

## 文档目的

本文档用于统一森林扰动数据的业务口径，明确：

1. 三个数据图层分别表示什么，以及它们之间的集合关系。
2. 如何将任意一个扰动斑块划分到四个业务类别之一。

数据处理代码、统计结果和周报内容都应遵循本文档，避免重复统计或错误解释数据。

## 一、图层定义与集合关系

### 1. 图层定义

| 图层 | 定义 |
|---|---|
| `slgr_confirm` | **算法已确认扰动**。图斑内像元的连续异常观测数均值 `conse_last >= 6`，且扰动概率大于 `0.5`。这里的“确认”仅指算法确认，不代表人工审核或现场核实。 |
| `slgr_detected` | **尚未确认的候选扰动记录**。满足扰动概率条件，但 `conse_last < 6`，仍需等待后续有效观测确认。 |
| `slgr_detected_rec` | `slgr_detected` 中的**近期扰动子集**，不是独立的第三种扰动状态。具体“近期”规则待代码核验。 |

### 2. 集合关系

```text
slgr_detected_rec ⊆ slgr_detected
```

- `slgr_detected_rec` 是 `slgr_detected` 的子集，两者不能直接相加，否则会重复统计。
- 按当前算法定义，`slgr_confirm` 与 `slgr_detected` 分别表示已达到和未达到算法确认条件的斑块。

## 二、斑块四分类规则

### 1. 分类目标

系统需要把任意一个扰动斑块唯一划分到以下四类之一：

| 类别 | 确认状态 | 时间状态 |
|---|---|---|
| 本期新增未确认 | 算法未确认 | 本期新增 |
| 本期新增确认 | 算法已确认 | 本期新增 |
| 历史未确认 | 算法未确认 | 非本期新增 |
| 历史已确认 | 算法已确认 | 非本期新增 |

这四类由两个判断维度组合得到：

```text
算法确认状态：已确认 / 未确认
时间状态：本期新增 / 历史已有
```

### 2. 暂行分类规则

#### 2.1 定义本期

```text
本期开始 = [P_START]
本期结束 = [P_END]
当前 epoch = [CURRENT_EPOCH]
epoch 时间表 = [EPOCH_START、EPOCH_END 对照表]
```

日期统一转换成同一种格式后再比较。

#### 2.2 判断当前状态

```text
如果图层是 slgr_confirm：
    status = confirmed

如果图层是 slgr_detected 或 slgr_detected_rec：
    status = unconfirmed
```

如果同一个事件同时出现在确认层和未确认层，以 `confirmed` 为准。

#### 2.3 确定事件时间

```text
如果 status = confirmed：
    event_time = confirmed
    如果 confirmed 缺失：
        event_time = [conf_epoch 对应的日期]

如果 status = unconfirmed：
    event_time = detection
    如果 detection 不是“首次检测日期”：
        event_time = [generation_epoch 对应的日期]
```

其中：

- 确认层的 `epoch` 暂按 `conf_epoch` 理解；
- 未确认层的 `epoch` 暂按 `generation_epoch` 理解；
- `disturba_1`、`last_clear` 不用于判断这四类。

#### 2.4 执行四分类

```text
如果 status = unconfirmed：
    如果 P_START <= event_time <= P_END：
        本期新增未确认
    如果 event_time < P_START：
        历史未确认

如果 status = confirmed：
    如果 P_START <= event_time <= P_END：
        本期新增确认
    如果 event_time < P_START：
        历史已确认
```

| 状态 | 事件时间 | 分类 |
|---|---|---|
| 未确认 | 在本期内 | 本期新增未确认 |
| 未确认 | 早于本期 | 历史未确认 |
| 已确认 | 在本期内 | 本期新增确认 |
| 已确认 | 早于本期 | 历史已确认 |

#### 2.5 缺少数据时的处理规则

```text
如果没有稳定 event_id：
    暂用 detection / confirmed 判断
    但标记为“暂定结果”

如果没有 event_time：
    暂用 epoch + [epoch 时间表]

如果 event_time 仍无法确定：
    输出“待核查”，不要强行归入四类
```

#### 2.6 分类示例

对于以下图斑：

```text
图层 = slgr_confirm
confirmed = 2025/02/16
```

如果本期是 `[2026 年的当前周期]`，则该图斑属于：

```text
历史已确认
```
