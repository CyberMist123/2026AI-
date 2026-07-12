# ChatGPT 交接文件

本目录保存可直接交给 Codex 执行的电脑端任务，以及需要在 iPhone 上手动完成的配置说明。

## 本次功能

- AI Closeout：520 网页手动执行、可编辑每日执行时间
- 可编辑生活时区
- 天气：恢复旧 AMap 实现并增加 Open-Meteo
- 月经周期：恢复旧 Care Cycle 实现
- Apple Health / Apple Watch：快捷指令上传、电脑端解析入库、520 展示、AI 查询工具
- HRV 压力信号：参考 StressWatch 思路，按个人 HRV、静息心率和睡眠基线生成可解释的非医疗压力/恢复信号
- iOS 屏幕使用时间：已放弃，不实施

## 文件

1. `AI-Closeout-时区-天气-月经-Apple-Health-MVP-Codex执行.md`
   - 交给 Codex。
   - 包含实施顺序、边界、验收、回滚和汇报格式。

2. `Apple-Health-HRV-压力信号-StressWatch补充.md`
   - Codex 执行 Health Phase 时必须一并读取。
   - HRV 原始指标已经在主方案内；本文件增加可解释的压力信号聚合、520 展示和测试约束。
   - 不抓取或复刻 StressWatch 的专有分数。

3. `Apple-Health-快捷指令-手机配置.md`
   - 用户在 iPhone 上操作。
   - 不属于 Codex 的电脑端实施范围。
   - 手机只上传 HRV、静息心率、睡眠等原始数据，压力信号在电脑端计算。
