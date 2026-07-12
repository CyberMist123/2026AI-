# Apple Health / Apple Watch 快捷指令 — iPhone 配置

这份只做手机端。先等 Codex 完成电脑端，并给出：

```text
上传地址：http://<电脑局域网IP>:4320/v1/health/import
Health ingest token：<一长串随机字符>
电脑端状态：connected
```

远程上传以后再配 HTTPS Tunnel。第一次只在同一 Wi‑Fi 下测试。

> iOS 中文动作名称可能随版本略有变化。搜索关键词用：`健康样本`、`查找健康样本`、`获取健康样本的详细信息`、`词典`、`添加到变量`、`获取 URL 内容`。

---

## 一、最终结构

创建一个快捷指令：

```text
同步 Apple Health 到 Cyberboss
```

流程：

```text
设置参数
→ 计算最近 3 天
→ 读取 8 类普通健康指标
→ 读取睡眠分段
→ 生成 samples 数组
→ 生成顶层 JSON
→ POST 到电脑
→ 显示服务器结果
```

不要导出 txt，不要 AirDrop，不要把睡眠数据拆成三等份，也不要先交给 AI 解析。

---

# 二、先做最小连通测试

不要一上来堆完整快捷指令。先只上传一条测试数据。

## 2.1 新建快捷指令

快捷指令 App → `+` → 名称：

```text
同步 Apple Health 到 Cyberboss
```

## 2.2 添加三个“文本”动作

```text
ServerURL
http://<电脑局域网IP>:4320/v1/health/import

Token
<Codex 生成的 health ingest token>

LifeTimezone
Australia/Sydney
```

可直接用三个“文本”动作，也可以用一个“词典”保存：

```json
{
  "server_url": "http://192.168.x.x:4320/v1/health/import",
  "token": "...",
  "timezone": "Australia/Sydney"
}
```

不要截图分享包含 token 的快捷指令。

## 2.3 创建测试样本

添加“词典”：

```json
{
  "metric": "steps",
  "value": 1,
  "unit": "count",
  "start_at": "当前日期的 ISO 8601",
  "end_at": "当前日期的 ISO 8601",
  "source": "Apple Health Shortcut"
}
```

实际在快捷指令中：

1. `当前日期`；
2. `格式化日期`，格式选择 ISO 8601；
3. 将格式化结果填到 `start_at` 和 `end_at`。

把这个词典放进“列表”，命名变量：

```text
AllSamples
```

## 2.4 创建顶层请求词典

```json
{
  "schema_version": 1,
  "timezone": "Australia/Sydney",
  "exported_at": "当前日期 ISO 8601",
  "samples": "AllSamples"
}
```

注意 `samples` 的值要选择变量 `AllSamples`，不是输入字符串 `AllSamples`。

## 2.5 POST

添加 `获取 URL 内容`：

```text
URL：ServerURL
方法：POST
请求正文：JSON
JSON：上面的顶层请求词典
```

Headers：

```text
Authorization: Bearer <Token变量>
Content-Type: application/json
```

最后添加：

```text
快速查看 / 显示结果
```

### 连通测试预期

第一次运行：

```json
{
  "status": "ok",
  "inserted": 1,
  "updated": 0,
  "skipped": 0
}
```

第二次运行同一份数据，应看到重复被跳过或更新，不能再次新增相同记录。

若失败：

- `无法连接服务器`：检查电脑与手机是否同一 Wi‑Fi、Windows 防火墙和局域网 IP；
- `401/403`：检查 Bearer token；
- `400`：检查 JSON、metric 和 ISO 时间；
- `413`：一次上传太多，缩短查询日期或减少指标。

连通测试通过后，删掉这条 value=1 的测试样本，再做真实采集。

---

# 三、通用变量

在快捷指令开头创建：

```text
LookbackDays = 3
AllSamples = 空列表
Now = 当前日期
StartDate = 当前日期减去 3 天
```

为什么取最近 3 天：Apple Health 或 Apple Watch 数据可能晚到、修订或重新同步；电脑端会幂等去重，不会重复入库。

每一种指标都使用同样的结构：

```text
查找健康样本
→ 重复每一项
→ 取开始时间、结束时间、值
→ 组一个标准词典
→ 添加到 AllSamples
```

---

# 四、普通健康指标

需要采集：

| Apple Health 类型 | metric | 推荐 unit |
|---|---|---|
| 体重 Weight | `weight` | `kg` |
| 心率变异性 HRV | `heart_rate_variability` | `ms` |
| 步数 Steps | `steps` | `count` |
| 训练时间 Exercise Time | `exercise_time` | `min` |
| 活动能量 Active Energy | `active_energy` | `kcal` |
| 静息能量 Resting Energy | `resting_energy` | `kcal` |
| 静息心率 Resting Heart Rate | `resting_heart_rate` | `count/min` |
| 步行与跑步距离 | `walking_running_distance` | `km` |

每类分别添加一个“查找健康样本”：

```text
类型：对应健康类型
开始日期：在 StartDate 之后
排序：开始日期，最早优先
限制：关闭
```

对“重复项目”获取：

- 开始日期；
- 结束日期；
- 值；
- 单位（若动作能直接取得）；
- 来源（若动作能取得，可选）。

开始和结束日期统一格式化成带时区 offset 的 ISO 8601。

每条生成词典：

```json
{
  "metric": "steps",
  "value": 123,
  "unit": "count",
  "start_at": "2026-07-14T07:00:00+10:00",
  "end_at": "2026-07-14T08:00:00+10:00",
  "source": "Apple Health Shortcut"
}
```

然后：

```text
添加 词典 到变量 AllSamples
```

### 单位处理

优先使用健康样本原单位。如果快捷指令难以稳定取得单位，则按上表写固定单位，并在 Health 动作中把数值转换到相应单位。

不要把数字和单位拼成一个文本，例如不要发送 `6821 steps`；必须分成：

```json
{"value": 6821, "unit": "count"}
```

---

# 五、睡眠分段

## 5.1 查询

添加“查找健康样本”：

```text
类型：睡眠
开始日期：在 StartDate 之后
排序：开始日期，最早优先
限制：关闭
```

对每条睡眠样本取：

- 开始日期；
- 结束日期；
- 健康样本值/睡眠类型；
- 来源（可选）。

持续秒数：

```text
结束日期 与 开始日期 之间的时间
→ 转成秒
```

## 5.2 睡眠阶段映射

根据手机返回的中文或英文值，用“如果”映射：

```text
核心睡眠 / Core                 → core
深度睡眠 / Deep                 → deep
快速眼动睡眠 / REM             → rem
清醒 / Awake                   → awake
睡眠 / Asleep                  → asleep_unspecified
卧床 / In Bed                  → in_bed
```

不要把未知值硬改成 core。出现无法识别的类型时：

- 第一版直接跳过该条；
- 记录到一个 `UnknownSleepStages` 文本变量；
- 快捷指令结束时提示，便于后续补映射。

## 5.3 睡眠样本词典

```json
{
  "metric": "sleep",
  "stage": "core",
  "start_at": "2026-07-13T23:14:00+10:00",
  "end_at": "2026-07-14T01:10:00+10:00",
  "duration_seconds": 6960,
  "source": "Apple Health Shortcut"
}
```

每条追加到 `AllSamples`。

手机端只发送原始分段。不要在快捷指令里自己决定“昨晚几点睡”“总共睡了多久”。这些由电脑端统一分 session、处理跨午夜和时区。

---

# 六、生成和上传真实 JSON

所有指标完成后，创建顶层词典：

```json
{
  "schema_version": 1,
  "timezone": "Australia/Sydney",
  "exported_at": "当前日期 ISO 8601",
  "samples": "AllSamples"
}
```

上传仍使用：

```text
获取 URL 内容
方法：POST
正文：JSON
Authorization: Bearer <Token>
Content-Type: application/json
```

建议在 POST 前加：

```text
获取 AllSamples 的项目数
如果大于 5000：显示提醒并停止
```

上传成功后显示：

```text
同步成功
新增：...
更新：...
跳过：...
```

失败时不要无限自动重试。显示错误，稍后手动再跑即可。

---

# 七、分步启用顺序

避免一次排查十个问题，按这个顺序加：

1. 测试样本；
2. 步数；
3. 静息心率；
4. HRV；
5. 活动能量；
6. 训练时间；
7. 体重；
8. 距离与静息能量；
9. 最后加睡眠。

每加一类：

- 手动运行；
- 看服务器返回；
- 打开 520 看最近同步；
- 再运行一次确认幂等；
- 没问题才继续。

睡眠完成后核对一天：

- Apple 健康 App 中的入睡和醒来范围；
- 520 聚合出的开始/结束；
- Core、Deep、REM、Awake 总分钟；
- 是否错误混入午睡或前一天残留。

发现不一致先保存原始样本，不要手改数据库，让 Codex修聚合算法后重算。

---

# 八、自动化

手动运行至少稳定 2 天后，再创建个人自动化。

推荐任选一个：

```text
每天上午 09:00 运行
```

或：

```text
连接到家中 Wi‑Fi 后运行
```

自动化动作：

```text
运行快捷指令 → 同步 Apple Health 到 Cyberboss
```

启用“立即运行/不询问”（不同 iOS 版本文字可能不同）。

不要设置几分钟一次。健康数据不需要高频同步；每天一次足够，手动运行用于补传。

---

# 九、远程传输以后再做

局域网 MVP 跑通后，若需要人在外面也能上传：

- 使用 HTTPS；
- Tunnel 只转发 `life-context ingest` 端口；
- 不暴露 520；
- 不暴露 SQLite；
- 继续要求 Bearer token；
- 更换为更长的随机 token；
- URL 改为 HTTPS 域名。

不要把普通 `http://公网IP:4320` 暴露到互联网。

---

# 十、隐私与记忆边界

- 健康原始样本只进电脑本地 `health.sqlite`；
- GitHub 不保存真实健康数据；
- 520 只展示摘要；
- AI 默认按需查询，不固定注入；
- 健康数据不自动变成 Episode、画像或 Re-entry；
- 不让 AI 给医疗诊断；
- 分享快捷指令前必须删掉服务器地址和 token。

最终只需要保留一个快捷指令：

```text
同步 Apple Health 到 Cyberboss
```
