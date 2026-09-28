# 森林扰动输入数据规范

> 版本：1.0  
> 更新日期：2026-09-25  
> 状态：基于 `20260912` 批次的三个图层建立；缺失信息使用方括号占位

## 文档目的

本文档定义森林扰动 Shapefile 的实际输入结构及其标准化方式，确保每个图斑都能被稳定地解析、分类和统计。

本文档只规定“数据如何进入系统”。斑块的四分类业务规则见[森林扰动数据图层语义与斑块分类规则](./01-data-layer-semantics.md)。

## 一、原始数据结构

### 1. 图层级信息

根据 `20260912` 批次的实际数据，三个图层具有相同的空间结构：

| 项目 | 当前值 |
|---|---|
| 图层 | `slgr_confirm`、`slgr_detected`、`slgr_detected_rec` |
| 坐标系 | `EPSG:4326`（WGS 84） |
| 几何类型 | `MultiPolygon` |
| 属性字段数 | 每个图层 10 个 |
| 稳定事件 ID | `[EVENT_ID_FIELD_OR_RULE]` |

QGIS FID 只是当前文件中的临时编号，不能作为跨批次的 `event_id`。

### 2. 字段存在情况

| 原始字段 | `slgr_confirm` | `slgr_detected` | `slgr_detected_rec` | 原始类型 |
|---|:---:|:---:|:---:|---|
| `agent` | ✓ | ✓ | ✓ | String |
| `brightness` | ✓ | ✓ | ✓ | Real |
| `wetness_ch` | ✓ | ✓ | ✓ | Real |
| `greenness` | ✓ | ✓ | ✓ | Real |
| `disturbanc` | ✓ | ✓ | ✓ | Real |
| `detection` | ✓ | ✓ | ✓ | String |
| `confirmed` | ✓ | — | — | String |
| `last_clear` | — | ✓ | ✓ | String |
| `disturba_1` | ✓ | ✓ | ✓ | String |
| `epoch` | ✓ | ✓ | ✓ | Integer64 |
| `province` | ✓ | ✓ | ✓ | String |

`disturbanc` 是 Shapefile 中的实际字段名，可能是 `disturbance` 受字段名长度限制后的结果。系统内部统一命名为 `disturbance_score`。

当前导出字段中没有发现 `event_id`、`conse_last` 或扰动概率字段。因此，现阶段可以根据来源图层使用算法状态，但不能仅凭导出的属性独立复核确认阈值。

### 3. 字段定义与标准化映射

| 原始字段 | 标准字段 | 含义 | 标准化规则 |
|---|---|---|---|
| `agent` | `disturbance_type` | 扰动驱动类型 | 转为小写枚举；未知值保留原值并标记待核查 |
| `brightness` | `brightness_change` | 相对 SCCD 预测状态的亮度变化分量 | 转为浮点数，不自行解释正负值 |
| `wetness_ch` | `wetness_change` | 相对 SCCD 预测状态的湿度变化分量 | 转为浮点数，不自行解释为实际含水量 |
| `greenness` | `greenness_change` | 相对 SCCD 预测状态的绿度变化分量 | 转为浮点数，不自行解释为植被覆盖率 |
| `disturbanc` | `disturbance_score` | 归一化多波段综合变化幅度 | 转为浮点数；数值越大通常表示偏离越明显，不直接换算严重等级 |
| `detection` | `detection_date` | 系统首次登记扰动对象的日期 | 解析后统一为 `YYYY-MM-DD` |
| `confirmed` | `confirmed_date` | 算法确认日期 | 解析后统一为 `YYYY-MM-DD`；仅确认层存在 |
| `last_clear` | `last_clear_date` | 最近一次通过质量控制的有效观测日期 | 解析后统一为 `YYYY-MM-DD`；不用于四分类 |
| `disturba_1` | `estimated_disturbance_date` | SCCD 估计的扰动发生日期 | 去除时间部分后统一为 `YYYY-MM-DD`；不用于四分类 |
| `epoch` | `epoch` | 增量处理批次序号 | 转为整数；确认层暂按 `conf_epoch`、未确认层暂按 `generation_epoch` 理解 |
| `province` | `region_code` | 图斑所属监测区域的代码 | 保留原始值，再通过 `[REGION_CODE_MAP]` 转换为展示名称 |
| 几何 | `geometry` | 扰动图斑空间范围 | 保留 `MultiPolygon`，标准坐标系为 `EPSG:4326` |

### 4. 日期输入格式

当前样例中存在多种日期字符串格式：

| 字段 | 已观察到的格式 | 示例 |
|---|---|---|
| `detection` | `YYYY-MM-DD` | `2026-09-12` |
| `confirmed` | `YYYY/MM/DD` | `2025/02/16` |
| `last_clear` | `YYYY-MM-DD` | `2026-09-08` |
| `disturba_1` | `YYYY/MM/DD HH:mm:ss` | `2026/09/08 00:00:00` |

解析时必须显式支持上述格式，并统一输出为 `YYYY-MM-DD`。无法解析的非空日期不得静默忽略，应标记为“待核查”。

## 二、标准斑块记录

系统读取原始图斑后，应生成以下统一结构：

| 标准字段 | 类型 | 来源或规则 | 是否必需 |
|---|---|---|:---:|
| `event_id` | String/null | `[EVENT_ID_FIELD_OR_RULE]` | 否（当前缺失） |
| `source_batch` | String | 输入批次，例如 `20260912` | 是 |
| `source_layer` | Enum | 原始图层名称 | 是 |
| `source_fid` | Integer | QGIS/文件 FID，仅用于本批次追溯 | 是 |
| `algorithm_status` | Enum | `slgr_confirm → confirmed`；另外两层 → `unconfirmed` | 是 |
| `event_time` | Date/null | 确认层优先 `confirmed_date`；未确认层优先 `detection_date` | 条件必需 |
| `period_class` | Enum | `current`、`historical` 或 `needs_review` | 是 |
| `classification` | Enum | 四分类之一；时间无法确定时为 `needs_review` | 是 |
| `classification_is_provisional` | Boolean | 没有稳定 `event_id` 时为 `true` | 是 |
| `disturbance_type` | Enum | 由 `agent` 标准化 | 是 |
| `region_code` | String | 原始 `province` | 是 |
| `region_name` | String/null | `[REGION_CODE_MAP]` | 条件必需 |
| `detection_date` | Date | `detection` | 是 |
| `confirmed_date` | Date/null | `confirmed` | 条件必需 |
| `last_clear_date` | Date/null | `last_clear` | 否 |
| `estimated_disturbance_date` | Date/null | `disturba_1` | 否 |
| `epoch` | Integer | `epoch` | 是 |
| `brightness_change` | Float/null | `brightness` | 否 |
| `greenness_change` | Float/null | `greenness` | 否 |
| `wetness_change` | Float/null | `wetness_ch` | 否 |
| `disturbance_score` | Float/null | `disturbanc` | 否 |
| `geometry` | MultiPolygon | 原始几何 | 是 |
| `area_m2` | Float/null | 由几何计算 | 条件必需 |
| `area_ha` | Float/null | `area_m2 / 10000` | 条件必需 |
| `centroid_lon` | Float/null | 由几何计算 | 条件必需 |
| `centroid_lat` | Float/null | 由几何计算 | 条件必需 |

### 1. 扰动类型映射

| `agent` 原始值 | 标准值 | 中文名称 |
|---|---|---|
| `logging` | `logging` | 采伐或人为清除 |
| `wildfire` | `wildfire` | 森林火灾 |
| `stress` | `stress` | 植被胁迫 |
| 其他值 | `unknown` | 其他/未知，保留原始值并待核查 |

### 2. 空间派生规则

- `EPSG:4326` 的坐标单位是度，不能直接用经纬度平面面积作为平方米面积。
- 面积和周长必须采用 `[AREA_CALCULATION_METHOD]` 计算，并明确输出单位。
- 中心点统一输出 WGS 84 经度和纬度。
- 无效或空几何应进入“待核查”，不得参与面积汇总。

### 3. 分类接口

四分类过程使用以下输入：

```text
[P_START]
[P_END]
[CURRENT_EPOCH]
[EPOCH_START、EPOCH_END 对照表]
event_id
source_layer
confirmed_date
detection_date
epoch
```

具体判断顺序和缺失数据处理方式以[四分类规则](./01-data-layer-semantics.md#二斑块四分类规则)为准。

## 三、当前样例映射

以下内容只用于验证字段映射，不代表图层的整体分布：

| 来源图层 | 临时 FID | 状态 | `event_time` 首选值 | `epoch` | 类型 | 区域代码 | 面积（m²） |
|---|---:|---|---|---:|---|---|---:|
| `slgr_confirm` | 437304 | 已确认 | `2025-02-16`（`confirmed`） | 59 | `logging` | `sichuan` | 7,199.08 |
| `slgr_detected` | 7494 | 未确认 | `2026-07-15`（`detection`） | 136 | `logging` | `hunan` | 77,362.97 |
| `slgr_detected_rec` | 270 | 未确认 | `2026-09-12`（`detection`） | 137 | `logging` | `hunan` | 25,210.07 |

由于 `[P_START]` 和 `[P_END]` 尚未确定，本表不直接给出四分类结果。

## 四、待补充项

| 占位符 | 需要确认的内容 |
|---|---|
| `[EVENT_ID_FIELD_OR_RULE]` | 跨批次稳定事件 ID 的字段或生成规则 |
| `[P_START]`、`[P_END]` | 本期开始和结束日期 |
| `[CURRENT_EPOCH]` | 当前报告周期对应的批次 |
| `[EPOCH_START、EPOCH_END 对照表]` | 每个 `epoch` 对应的时间范围 |
| `[REGION_CODE_MAP]` | `province` 代码到中文区域名称的完整映射 |
| `[AREA_CALCULATION_METHOD]` | 全国范围统一采用的面积和周长计算方法 |
| `[RECENT_SUBSET_RULE]` | `slgr_detected_rec` 的具体筛选条件 |
| `[CONSE_LAST_FIELD]` | `conse_last` 是否可从其他数据源补充 |
| `[DISTURBANCE_PROBABILITY_FIELD]` | 扰动概率字段是否可从其他数据源补充 |
