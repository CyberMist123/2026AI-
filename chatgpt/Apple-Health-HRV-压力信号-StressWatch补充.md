# Apple Health / Apple Watch：HRV 压力信号补充

> 本文件是 `AI-Closeout-时区-天气-月经-Apple-Health-MVP-Codex执行.md` 的强制补充。Codex 执行 Phase 5 时一并实现。
>
> 目标不是复刻或抓取 StressWatch 的专有分数，而是基于 Apple Health 已导出的 HRV、静息心率和睡眠，生成一个透明、可解释、非医疗用途的压力信号。

## 一、边界

1. Apple Health 快捷指令已包含 `heart_rate_variability`，所以原始 HRV 已在现有方案内。
2. 不假设 Apple Health 有通用 `stress` 健康类型。
3. 不抓取 StressWatch 私有数据库，不逆向 App，不模拟其专有算法。
4. 除非 StressWatch 后续明确提供快捷指令动作、CSV 或 API，否则不导入它的专有压力分数。
5. 结果只能叫：`压力信号`、`恢复信号`或`HRV 相对状态`；禁止叫医学诊断或确定的心理压力值。
6. 数据不足时必须显示 `insufficient_data`，不能输出 0 分或“正常”。

## 二、MVP 输入

复用现有样本，不新增手机端必做动作：

```text
heart_rate_variability
resting_heart_rate
sleep
```

可选后续指标，不阻塞 MVP：

```text
respiratory_rate
heart_rate
wrist_temperature
```

快捷指令端只上传原始数据，不在手机上计算压力。

## 三、电脑端派生结果

在每日聚合中新增：

```text
metric: stress_signal
```

建议输出结构：

```json
{
  "local_date": "2026-07-14",
  "status": "insufficient_data|lower|usual|elevated",
  "confidence": "low|medium",
  "hrv_ms": 42.0,
  "hrv_baseline_ms": 55.0,
  "hrv_ratio": 0.764,
  "resting_hr": 68.0,
  "resting_hr_baseline": 62.0,
  "sleep_minutes": 342,
  "reasons": [
    "HRV 低于个人近期基线",
    "静息心率高于个人近期基线",
    "睡眠少于近期水平"
  ],
  "disclaimer": "仅为可穿戴数据形成的压力/恢复信号，不是医学或心理诊断。"
}
```

## 四、基线和规则

### 4.1 基线

- HRV 基线：最近 14 个有效生活日的中位数；
- 静息心率基线：最近 14 个有效生活日的中位数；
- 睡眠基线：最近 7 个有效睡眠日的中位数；
- 至少 7 个有效 HRV 日后才允许生成状态；不足则 `insufficient_data`；
- 使用中位数，不用平均数，减少异常测量影响；
- 只与用户自己的历史比较，不使用人群固定阈值。

### 4.2 第一版确定性分类

计算：

```text
hrv_ratio = today_hrv / hrv_baseline
rhr_delta = today_resting_hr - resting_hr_baseline
sleep_delta = today_sleep_minutes - sleep_baseline
```

规则按以下顺序：

```text
insufficient_data:
  HRV 基线有效日 < 7，或当天无 HRV

lower:
  hrv_ratio >= 1.10 且 rhr_delta <= 3

usual:
  0.85 <= hrv_ratio < 1.10，且 rhr_delta < 6

elevated:
  hrv_ratio < 0.85
  或 rhr_delta >= 6
  或同时满足：hrv_ratio < 0.95 且 sleep_delta <= -90
```

说明：

- `elevated` 表示生理压力/恢复不足信号升高，不等于用户主观上一定焦虑；
- `lower` 表示相对个人基线恢复信号较好，不等于“完全没有压力”；
- 首版不生成 0–100 分，避免制造伪精度；
- 每条结果必须保存触发原因，不能只存标签。

## 五、520 展示

Apple Health 卡增加：

```text
压力信号：数据不足 / 较低 / 日常范围 / 升高
依据：HRV、静息心率、睡眠相对个人基线
基线天数：x / 14
最近计算：...
```

展开后显示：

- 当日 HRV 与个人基线；
- 当日静息心率与个人基线；
- 睡眠与近期中位数；
- `reasons`；
- 非医疗提示。

不要使用红色“危险”警报。`elevated` 只用普通提醒样式。

## 六、AI 工具

现有工具中扩展：

```text
health_today
health_weekly_compare
```

并可新增：

```text
health_stress_signal
```

返回：

- 状态；
- 日期和时区；
- 基线有效天数；
- 触发原因；
- 原始支持指标；
- 非医疗提示。

默认仍不固定注入。只有 AI 主动调用或用户明确打开 `health.ai_context_enabled` 时，才允许形成极短临时 signal：

```xml
<health_signal>
今天的 HRV 和静息心率相对个人近期基线显示恢复不足；这只是可穿戴数据提示，不代表确定的心理压力。
</health_signal>
```

禁止自动写入 Re-entry、Episodes、用户画像或 AI Self-note。

## 七、测试

Codex 必须增加确定性测试：

- HRV 不足 7 天返回 `insufficient_data`；
- 14 日中位数计算正确；
- 极端离群值不明显拖动基线；
- HRV 明显下降触发 `elevated`；
- 静息心率明显上升触发 `elevated`；
- 单纯少睡但 HRV/心率正常，不轻易判高；
- 原因列表与触发规则一致；
- 同一输入重复聚合结果稳定；
- 时区和生活日期正确；
- 520 无数据不显示 0；
- AI 输出包含非医疗提示；
- 压力信号不进入正式记忆文件。

## 八、提交

可并入 Health Phase commit：

```text
feat(health): add Shortcut ingest, stress signal, dashboard and tools
```

若 Phase 5 已提交，则单独提交：

```text
feat(health): add explainable HRV stress signal
```
