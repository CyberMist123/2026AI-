# Apple Health / Apple Watch：HRV 与 StressWatch 边界说明

> 本文件不是 Codex 强制实现项。
>
> 结论：当前 MVP 只导入并展示 Apple Health 原始指标与趋势，不在电脑端自行生成 StressWatch 式压力分数或压力等级。

## 为什么不做

1. Apple Health 可提供 HRV、静息心率、睡眠等数据，但没有统一、标准的 `stress` 健康类型。
2. StressWatch 的具体算法、滤波、采样选择、个人校准和阈值没有公开，无法验证电脑端规则与其一致。
3. Apple Watch 后台心率与 HRV 采样并非连续固定频率；快捷指令导出的只是 HealthKit 已保存样本。
4. HRV 同时受运动、睡眠、饮酒、疾病、测量姿势和心理压力影响，不能单独等同于心理压力。
5. 自行设置阈值会制造看似精确但未经验证的结果。

## MVP 保留内容

手机快捷指令继续上传：

```text
heart_rate_variability
resting_heart_rate
sleep
```

电脑端只做：

- 原始值存储；
- 每日/每周趋势；
- 与个人历史中位数的数值对比；
- 明确展示数据日期、样本数和最近同步时间；
- 不输出 `低压力 / 正常 / 高压力`；
- 不生成 0–100 压力分；
- 不写入正式记忆。

允许的透明描述示例：

```text
今日 HRV 为 42 ms，近 14 个有效日中位数为 55 ms。
该变化可能受睡眠、运动、身体状态或心理压力等多种因素影响。
```

禁止的输出：

```text
压力值 78
高压力
与 StressWatch 准确度相同
```

## 后续接入 StressWatch 的条件

只有满足以下任一条件，才另开任务：

- StressWatch 官方提供 Shortcuts 动作；
- StressWatch 将自身压力结果写回 Apple Health 且类型可稳定读取；
- StressWatch 提供 CSV、JSON 或公开 API；
- 获得可验证的公开算法说明。

届时优先导入 StressWatch 自己的结果，而不是在电脑端猜测复刻。

## 给 Codex

Phase 5 不实现 `stress_signal`、`health_stress_signal`、压力等级或压力分数。只实现原始 HRV/静息心率/睡眠的导入、趋势展示和查询。
