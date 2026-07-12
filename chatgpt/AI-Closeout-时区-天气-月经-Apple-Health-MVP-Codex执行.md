# AI Closeout、时区、天气、月经、Apple Health MVP — Codex 执行单

> 目标：在现有 Cyberboss / 520 架构内做一个能实际运行的 MVP。只实现电脑端。iPhone 快捷指令由用户按另一份文档完成。
>
> 已放弃：iOS 屏幕使用时间。不要研究、不要留空壳按钮、不要申请 Family Controls。

## 一、执行方式

- 只开一个 Codex agent。
- 不并发，不调用子 agent。
- 先做 Phase 0，只读确认；确认无歧义后连续执行。
- 每个 Phase 单独 commit，失败只回滚当前 Phase。
- 不把长测试输出、完整 diff、整个文件原文塞回对话。只报告：变更文件、测试命令、通过数、失败摘要、commit SHA。
- 搜索具体符号，不通读整个仓库。
- 不重写 520 大文件；优先新增独立模块，再做很薄的接线。

建议模型：GPT-5.6 Sol，reasoning max，单 agent。

---

## 二、绝对边界

1. 不直接修改正在运行的 immutable release。
2. 不在 live 目录试验；必须使用 git 分支/worktree。
3. 不删除旧 memory、care、continuity、todo 数据。
4. 不把健康、月经数据写入：
   - `memory/reentry.md`
   - `memory/episodes.jsonl`
   - `memory/user_portrait.md`
   - `memory/ai_self_notes.md`
5. 生活数据默认只作为可选临时上下文；默认关闭 AI 注入。
6. 网页不能成为 Closeout 调度进程。关闭 520 后，定时 Closeout 仍须运行。
7. 手动 Closeout 与定时 Closeout 必须调用同一个后端链，不能出现两个 writer。
8. 不让 520 监听公网；Apple Health 上传使用独立窄入口。
9. 不将 token、`.env`、Health 原始数据、SQLite 数据库提交 Git。
10. 不新增 FastMCP 常驻服务。AI 查询接现有 Cyberboss tool host。
11. 不做原生 iOS App，不做 Apple Screen Time。
12. 不改 Telegram poller、模型切换、Soft Retrieval 语义逻辑，除非本任务的最小接线确实需要。

遇到以下情况立即停止，不猜：

- 找不到唯一源码仓库或当前部署 descriptor；
- 当前工作树存在与本任务重叠的未提交改动；
- `deployment/current.json` 指向的 release 与找到的源码无法对应；
- 现有 Closeout runner 或 writer lease 无法确认；
- 需要覆盖用户未提交的 520 页面改动；
- 现有部署脚本与文档互相矛盾。

停止时只报告证据路径、SHA 和冲突点，不做修复性写入。

---

## 三、已确认可复用的旧实现

远端历史分支：`origin/legacy-current`

不要凭记忆重写，优先用 `git show` 精确读取：

```powershell
# 旧 520 Care 页面/API
git show origin/legacy-current:cyberboss-deepseek-workspace/memory-kit/dashboard.py

# 旧天气服务
git show origin/legacy-current:src/services/weather-service.js

# 旧天气配置
git show origin/legacy-current:src/core/config.js

# 旧 AI weather tool
git show origin/legacy-current:src/tools/tool-host.js

# 旧 Closeout 语义说明
git show origin/legacy-current:cyberboss-deepseek-workspace/memory-kit/AUTOMATION.md
git show origin/legacy-current:cyberboss-deepseek-workspace/memory/closeout_guide.md
```

旧 Care 已有的骨架：

- `CARE_CONFIG_DEFAULTS`
- `care/config.json`
- `care/cycle.md`
- `/api/care/config`
- `/api/care/cycle`
- 520 关怀页 cycle 录入

旧 Weather 已有：

- AMap 当前天气
- 今日/明日/后日预报
- `current / forecast / summary / raw`
- timeout + retry
- 城市/adcode 配置
- `weather` AI tool

这些是迁移任务，不是重新设计任务。

---

# Phase 0 — 只读发现、建立安全工作区

## 0.1 找唯一部署入口

先读：

```powershell
$descriptor = 'C:\Users\18717\Documents\cyberlink\deployment\current.json'
Get-Content -LiteralPath $descriptor -Raw
```

确认并记录：

- 当前 app/release root
- dashboard root
- workspace root
- state dir
- log dir
- 当前 release SHA（若 descriptor 内有）

禁止输出 token、env 内容。

## 0.2 找源码仓库

在 `C:\Users\18717\Documents\cyberlink` 下只查 `.git` 和 `git rev-parse`，不要递归读取源码。

选择规则：

1. 必须是 Git 仓库；
2. 必须含当前 520、continuity runner 和 Windows deployment 脚本；
3. 必须能对应当前远端 `CyberMist123/-0710-cyberboss-telegram-memory-`；
4. 若有多个候选且无法唯一判断，停止。

## 0.3 基线

```powershell
git fetch --all --prune
git status --short
git branch --show-current
git rev-parse HEAD
git log -1 --oneline
```

检查与本任务相关文件是否有未提交改动。相关路径至少包括：

- `extensions/relationship-memory/memory-kit/`
- `scripts/continuity/`
- `scripts/windows/`
- `src/core/config.js`
- `src/services/weather-service.js`
- `src/tools/tool-host.js`
- `test/`

若相关文件干净，创建：

```text
branch: feat/life-context-mvp-20260713
worktree: C:\Users\18717\Documents\cyberlink\worktrees\life-context-mvp-20260713
```

若分支名已存在，加 `-HHMMSS`，禁止覆盖。

## 0.4 跑基线测试

只使用仓库已有测试命令。先读 `package.json` 和相关测试文档，不猜命令。

保存：

- 命令
- 通过/失败数
- 基线 SHA

基线已有失败时，确认与本任务无关后记录；不可顺手修。

### Phase 0 验收

- [ ] 唯一源码仓库已确认
- [ ] descriptor 与源码关系可解释
- [ ] worktree 建立成功
- [ ] 相关路径无未提交覆盖风险
- [ ] 基线测试结果已记录

Commit：无。

---

# Phase 1 — Life Context 基础层与可编辑时区

## 1.1 新增独立模块

优先新增：

```text
extensions/relationship-memory/memory-kit/life_context.py
extensions/relationship-memory/memory-kit/tests/test_life_context.py
```

不要把全部逻辑继续塞进 `dashboard.py`。

状态目录从 `CYBERBOSS_STATE_DIR` 推导：

```text
<state-dir>/life-context/
  settings.json
  weather-cache.json
  closeout-status.json
  health-summary.json
  health-sync-status.json
  health.sqlite            # gitignored
  logs/
```

所有 JSON 写入：临时文件 + `os.replace` 原子替换。

## 1.2 settings schema

```json
{
  "version": 1,
  "timezone": "Australia/Sydney",
  "closeout": {
    "enabled": false,
    "local_time": "03:30"
  },
  "weather": {
    "enabled": false,
    "provider": "open_meteo",
    "city": "Sydney",
    "latitude": null,
    "longitude": null,
    "country_code": "AU",
    "ai_context_enabled": false
  },
  "cycle": {
    "cycle_period_days": 28,
    "pms_lead_days": 3,
    "cycle_prompt_enabled": false
  },
  "health": {
    "ai_context_enabled": false,
    "sleep_day_cutoff": "12:00",
    "session_gap_minutes": 120
  }
}
```

校验：

- `timezone`：Python `zoneinfo.ZoneInfo` 可解析的 IANA 名称；
- `local_time`：严格 `HH:mm`；
- 周期：21–40；
- PMS：0–10；
- sleep cutoff：严格 `HH:mm`；
- session gap：30–360。

默认时区可为 `Australia/Sydney`，但必须可编辑。

## 1.3 API

接入 520：

```text
GET  /api/life-context/settings
PUT  /api/life-context/settings
GET  /api/life-context/status
```

写 API 沿用本机 token 保护和 CSRF/Origin 约束；不能新开无认证写口。

## 1.4 页面

新增一个独立 Tab：`生活上下文`。

首屏卡片：

- 生活时区
- AI Closeout
- 天气
- 月经周期
- Apple Health

时区控件：

- 可搜索/输入 IANA 名称；
- 显示浏览器检测值，但不自动覆盖；
- 保存前后显示当前该时区时间；
- 无效时区阻止保存。

不要引入前端框架和 CDN。

## 1.5 替换硬编码北京时间

只处理本任务相关的“生活时间”工具和显示：

- 保持旧工具名兼容；
- 新实现从 settings 读取时区；
- 输出必须包含时区名称；
- 不做无关全仓库时间重构。

### Phase 1 测试

- 默认 settings 创建；
- 原子写入；
- `Australia/Sydney`、`Asia/Shanghai` 有效；
- 无效时区被拒绝；
- DST 前后格式正确；
- PUT 不允许额外危险字段；
- API token 错误返回 401/403；
- 页面不出现密钥内容。

### Commit

```text
feat(life-context): add settings and editable timezone
```

---

# Phase 2 — AI Closeout：网页手动执行 + 网页编辑时间

## 2.1 单一后端链

必须复用现有：

```text
Closeout → Janitor → Auto Review → History Writer
```

优先复用：

- `scripts/continuity/run-phase3.js all`
- `scripts/windows/continuity-nightly.ps1`
- 现有 writer lease / daily idempotency

禁止复制 writer 或另写第二套 Closeout。

## 2.2 手动网页按钮

API：

```text
POST /api/closeout/run
GET  /api/closeout/status
```

行为：

- POST 立即返回 202，不阻塞 HTTP 请求；
- 后台启动同一 nightly chain；
- 已运行时返回 `running`；
- 同一生活日期已完成时返回 `already_done`；
- `no_output` 是成功状态，不是错误；
- 保存日志尾部和 exit code，但不在 API 返回整段模型输出。

页面按钮：`立即执行 AI Closeout`。

状态枚举：

```text
idle | running | success | no_output | already_done | failed
```

## 2.3 可编辑定时时间

网页保存：

```json
{
  "enabled": true,
  "local_time": "03:30",
  "timezone": "Australia/Sydney"
}
```

真实调度不在网页内。

实现一个轻量 tick，例如：

```text
scripts/continuity/life-context-schedule-tick.py
scripts/windows/life-context-schedule-tick.ps1
```

Windows 当前用户计划任务：

```text
cyberlink-continuity-scheduler
```

每 5 分钟运行一次 tick。tick 只做：

1. 读 settings；
2. 用 IANA timezone 算生活日期和当地时间；
3. 判断是否进入执行窗口；
4. 检查状态/lease；
5. 到点才调用现有 nightly chain；
6. 未到点立即 0 退出。

不得每次网页改时间就重建 Windows 任务。

若已有固定时间的 `cyberlink-continuity-nightly`：

- 先确认它是否会造成重复；
- 安装新 scheduler 后，将旧固定任务禁用或迁移；
- 不删除，保留可回滚证据；
- 文档写清楚 owner 只有一个。

执行窗口：计划时间起 10 分钟内。失败可 15 分钟后重试，最多 3 次/生活日。

状态至少保存：

```json
{
  "running": false,
  "last_attempt_at": "",
  "last_success_at": "",
  "last_local_date": "",
  "last_result": "",
  "last_exit_code": null,
  "next_run_at": "",
  "consecutive_failures": 0
}
```

## 2.4 Closeout 安全测试

必须用 fake runner，不调用真实模型：

- 手动按钮调用正确 runner 一次；
- 连点两次只启动一个进程；
- 同日重复返回 `already_done`；
- `no_output` 记成功；
- 关闭 dashboard 不影响 scheduler tick；
- Sydney DST 切换日 next run 正确；
- disabled 不执行；
- runner 非 0 状态显示 failed；
- 日志不含 token。

### Commit

```text
feat(closeout): add manual run and editable schedule
```

---

# Phase 3 — 恢复旧 Care Cycle

从 `origin/legacy-current` 精确迁移，不重做页面。

需要恢复/补齐：

- `care/config.json`
- `care/cycle.md`
- `/api/care/config`
- `/api/care/cycle`
- 关怀页周期录入
- `parse_cycle_starts(text)`
- `compute_cycle_status(today=None)`
- `build_cycle_care_hint(status, cfg)`
- `GET /api/care/cycle_status`

状态计算：

```text
next_start = last_start + cycle_period_days
pms_start  = next_start - pms_lead_days
day_index  = today - last_start + 1
phase      = period | pms | normal | unknown
```

约束：

- 这是用户自定义提醒，不是医学预测；
- 默认 `cycle_prompt_enabled=false`；
- 不主动发 Telegram；
- 不写正式记忆；
- 页面清楚标注“估算”；
- 不做复杂图表。

AI hint 第一版只提供函数/API。只有开关打开才可生成极短临时 signal；不要直接永久注入到所有轮次。

### 测试

- 无记录为 unknown；
- 最近开始日解析；
- 28 天计算；
- 21/40 边界；
- 非法日期拒绝；
- PMS 0/10 边界；
- 默认无 hint；
- period/PMS 开启后有短 hint；
- cycle 数据没有写入 memory 正史。

### Commit

```text
feat(care): restore cycle tracking and optional PMS hint
```

---

# Phase 4 — 天气：恢复 AMap + 增加 Open-Meteo

## 4.1 恢复旧代码

从旧分支迁移：

- `src/services/weather-service.js`
- config 字段
- `weather` tool
- 对应测试

保留兼容：

```text
current | forecast | summary | raw
```

不要破坏旧 AMap 用户。

## 4.2 增加 Open-Meteo provider

不要部署额外 MCP。直接 REST：

- Geocoding API：城市 → 经纬度；
- Forecast API：当前、今日高低温、体感温度、降雨概率；
- `timezone=auto`；
- 10 秒 timeout；
- 最多 3 次 retry；
- 30 分钟缓存。

provider：

```text
amap | open_meteo
```

默认 `open_meteo`，适合 Sydney/海外；AMap 需要 key 时从现有 env 读取，网页不得回显 key。

配置优先级：

1. life-context settings 中用户保存的 provider/location；
2. 旧 env 配置兼容；
3. 未配置时返回 `not_configured`，不猜 IP/GPS。

状态：

```text
not_configured | connected | stale | failed | disabled
```

缓存过期且请求失败时可返回旧缓存，但必须标 `stale` 和最后更新时间。

## 4.3 页面

字段：

- 启用
- provider
- 城市
- 国家代码
- 经纬度（自动解析后展示，可编辑）
- 测试连接
- 最近更新时间
- 当前状态
- 是否允许 AI 使用，默认关

### 测试

全部 mock fetch，不依赖真实网络：

- Open-Meteo geocoding；
- current/forecast 解析；
- AMap 旧行为不变；
- timeout/retry；
- stale cache；
- 未配置；
- key 不出现在日志/API；
- provider 非法被拒绝。

### Commit

```text
feat(weather): restore AMap and add Open-Meteo provider
```

---

# Phase 5 — Apple Health / Apple Watch 快捷指令导入

Apple Watch 数据先经 iPhone Apple Health 汇总。电脑端只接收快捷指令上传。

## 5.1 独立窄入口

新增：

```text
extensions/relationship-memory/memory-kit/life_context_ingest.py
extensions/relationship-memory/memory-kit/tests/test_life_context_ingest.py
scripts/windows/life-context-ingest.ps1
```

使用 Python 标准库：

- `http.server`
- `sqlite3`
- `zoneinfo`
- `hashlib`

默认：

```text
host: 0.0.0.0
port: 4320
```

配置项：

```text
CYBERBOSS_LIFE_CONTEXT_HOST
CYBERBOSS_LIFE_CONTEXT_PORT
CYBERBOSS_HEALTH_INGEST_TOKEN
```

只开放：

```text
GET  /health
POST /v1/health/import
```

禁止暴露：

- 520
- SQLite 查询
- 文件浏览
- MCP
- 任意命令执行

POST 必须 Bearer token。最大 body 1 MiB，最多 5000 samples。

## 5.2 输入 schema

接受：

```json
{
  "schema_version": 1,
  "timezone": "Australia/Sydney",
  "exported_at": "2026-07-14T08:00:00+10:00",
  "samples": [
    {
      "metric": "steps",
      "value": 6821,
      "unit": "count",
      "start_at": "2026-07-14T00:00:00+10:00",
      "end_at": "2026-07-14T08:00:00+10:00",
      "source": "Apple Watch"
    },
    {
      "metric": "sleep",
      "stage": "core",
      "start_at": "2026-07-13T23:14:00+10:00",
      "end_at": "2026-07-14T01:10:00+10:00",
      "duration_seconds": 6960,
      "source": "Apple Watch"
    }
  ]
}
```

白名单：

```text
weight
heart_rate_variability
steps
exercise_time
active_energy
resting_energy
resting_heart_rate
walking_running_distance
sleep
```

睡眠 stage 白名单：

```text
core | deep | rem | awake | asleep_unspecified | in_bed
```

时间必须是带 offset 的 ISO 8601。拒绝未来超过 24 小时或早于 2 年的数据。

## 5.3 SQLite

路径：

```text
<state-dir>/life-context/health.sqlite
```

表：

```sql
health_samples(
  sample_id TEXT PRIMARY KEY,
  metric TEXT NOT NULL,
  start_at TEXT NOT NULL,
  end_at TEXT,
  value REAL,
  unit TEXT,
  stage TEXT,
  source TEXT,
  raw_json TEXT NOT NULL,
  imported_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

health_daily(
  local_date TEXT NOT NULL,
  timezone TEXT NOT NULL,
  metric TEXT NOT NULL,
  value REAL,
  unit TEXT,
  details_json TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  PRIMARY KEY(local_date, timezone, metric)
);

health_sync_runs(
  run_id TEXT PRIMARY KEY,
  received_at TEXT NOT NULL,
  inserted_count INTEGER NOT NULL,
  updated_count INTEGER NOT NULL,
  skipped_count INTEGER NOT NULL,
  error_count INTEGER NOT NULL,
  status TEXT NOT NULL
);
```

`sample_id`：若客户端未提供，用 canonical JSON 的关键字段做 SHA-256：

```text
metric + start_at + end_at + value + unit + stage + source
```

重复上传必须幂等。

## 5.4 睡眠聚合

禁止用“把 txt 三等分”或让模型解析固定格式。

算法：

1. 按开始时间排序；
2. 相邻记录间隔超过 `session_gap_minutes`，切成不同 session；
3. 以配置 `sleep_day_cutoff` 作为睡眠日边界；
4. 选择截止时间前最后一个完整 session；
5. 计算：开始、结束、总跨度、各阶段分钟数；
6. 保留 `asleep_unspecified` 和 `in_bed`；
7. 原始样本永远保留，可重新聚合。

输出示例：

```json
{
  "sleep_date": "2026-07-13",
  "sleep_start": "2026-07-13T23:14:00+10:00",
  "sleep_end": "2026-07-14T07:14:00+10:00",
  "total_sleep_minutes": 439,
  "core_minutes": 245,
  "deep_minutes": 50,
  "rem_minutes": 126,
  "awake_minutes": 18,
  "timezone": "Australia/Sydney",
  "quality": "complete"
}
```

不要给医疗结论和“睡眠质量好/坏”诊断。

## 5.5 导出给 520 和 Node

每次成功导入后原子更新：

```text
health-summary.json
health-sync-status.json
```

520 只读这些摘要，不直接在页面查询整库。

新增独立当前用户启动任务：

```text
cyberboss-life-context-ingest
```

要求：

- pythonw 隐藏启动；
- PID 文件；
- 重复启动检测；
- 与 dashboard/watchdog 独立；
- 安装、状态、卸载命令；
- 日志轮转；
- 默认仅 LAN，不创建 Cloudflare Tunnel。

## 5.6 520 健康卡

显示：

```text
Apple Health：快捷指令桥
Apple Watch：经 Apple Health 间接接入
最近同步
新增 / 更新 / 跳过
睡眠
静息心率
HRV
步数
活动能量
状态
```

状态：

```text
not_configured | connected | syncing | stale | failed | permission_missing
```

不得在页面显示 ingest token、原始心率序列或全部样本。

## 5.7 AI 查询工具

接现有 tool host，读取摘要 JSON；不要让 Node 直接依赖 SQLite 包。

工具：

```text
health_today
health_sleep
health_range
health_weekly_compare
health_sync_status
```

限制：

- 默认仅在 AI 主动调用时读取；
- `ai_context_enabled=false` 时不固定注入；
- 输出附带数据日期、时区、最近同步时间；
- 不输出医疗诊断；
- 无数据时明确 `not_available`。

## 5.8 Health 测试

- 无 token 401；
- 错 token 403；
- body 超限 413；
- metric/stage 非法 400；
- 不带 offset 的时间拒绝；
- 重复上传 inserted=0、skipped>0；
- 修订样本可更新；
- 多睡眠 session 正确分段；
- 跨午夜、Sydney DST 正确；
- 原始数据不进 Git；
- 520 不回显 token；
- tool host 无数据/有数据均稳定；
- 服务重启后数据仍在。

### Commit

```text
feat(health): add Shortcut ingest, aggregation, dashboard and tools
```

---

# Phase 6 — 集成、UI 与文档收口

## 6.1 页面原则

一个 `生活上下文` Tab，不要分散到多个重复页面。

每张卡必须显示真实状态：

```text
not_configured | setup_required | connected | stale | failed | disabled
```

未接入不能显示绿色；无数据不能显示 0 冒充真实值。

## 6.2 TODO

把本任务同步到现有 520 Coding TODO。不要新建第二套 TODO 系统。

勾选时必须有 commit SHA 或测试证据；未完成项保留原因。

明确写：

```text
iOS Screen Time：放弃，不实施。
```

## 6.3 文档

只新增一个状态文档：

```text
docs/LIFE_CONTEXT_MVP.md
```

包含：

- 架构图；
- 文件位置；
- API；
- 状态目录；
- 安装/卸载；
- 手机快捷指令输入 schema；
- 安全边界；
- 回滚步骤；
- 已知限制。

不要制造一串重复 MD。

### Commit

```text
docs(life-context): document MVP and operations
```

---

# Phase 7 — 总验收与部署门

## 7.1 全量离线验收

必须通过：

1. 原项目既有测试；
2. 新 Python 测试；
3. 新 Node 测试；
4. Python compile；
5. portability/static path 检查；
6. git diff 无 token、用户名绝对路径、健康数据；
7. `git status --short` 只含预期文件。

敏感扫描至少查：

```text
Bearer
API_TOKEN
HEALTH_INGEST_TOKEN
.cyberboss
C:\Users\18717
health.sqlite
```

文档示例允许占位符，不允许真实值。

## 7.2 521 离线验收

使用临时 state-dir 和 521 端口：

- 520/521 页面正常；
- settings 保存；
- 时区修改；
- fake Closeout 手动执行；
- fake scheduled tick；
- Care cycle；
- mock weather；
- Health import curl；
- Health 卡片；
- tools 读取摘要。

不得碰 live 520、TG token 和真实健康库。

## 7.3 Live 部署条件

只有全部满足才部署：

- 所有新测试通过；
- 基线没有新增失败；
- 521 smoke 通过；
- 相关工作树干净；
- commits 已推到远端 feature branch；
- 已生成 descriptor 备份；
- 有明确 rollback release。

使用项目现有 immutable release/descriptor 部署流程，不发明新流程。

如果部署工具或 descriptor 语义不清楚：停止在已推送 feature branch，不碰 live。

部署后只做一次受控重启：

- dashboard；
- life-context ingest；
- scheduler；
- Node runtime 仅在 weather/tool host 变更确实需要时重启；
- 禁止启动第二个 Telegram poller。

## 7.4 Live smoke

- `127.0.0.1:520` 可打开；
- 旧页面功能未丢；
- 生活上下文 Tab 正常；
- timezone 保存后重启仍在；
- Closeout 状态 API 正常，不实际强制重复跑；
- scheduler 任务只有一个 owner；
- health ingest `/health` 正常；
- 错 token 被拒；
- weather 未配置时诚实显示；
- Telegram 仍只有一个 poller；
- 日志无 traceback/409/重复 writer。

失败立刻回滚 descriptor 到上一 release，并恢复旧任务状态。

---

# 最终汇报格式

只按下面格式回报，避免长篇复述：

```text
## 结果
完成 / 部分完成 / 已停止

## 分支与提交
branch:
commits:
remote:

## 测试
baseline:
new tests:
full tests:
521 smoke:
live smoke:

## 已实现
- Closeout 手动：
- Closeout 定时：
- 时区：
- Care：
- Weather：
- Apple Health：

## 未实现/限制
- ...

## 部署
release:
descriptor backup:
rollback release:

## 需要用户做的唯一事情
按 `Apple-Health-快捷指令-手机配置.md` 在 iPhone 建快捷指令，并填入电脑地址与 ingest token。
```

不要贴完整 diff、完整日志或数据库内容。
