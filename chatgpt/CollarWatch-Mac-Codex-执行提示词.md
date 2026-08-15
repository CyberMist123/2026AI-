# Collar Watch / Apple Health — Mac Codex 执行提示词

> 用途：直接交给 Mac 上的 Codex 执行。
> 目标：fork `KKarsyline/Collar_watch`，完成 Apple Watch 客户端与 Mac 端自动 build / 续签基础设施；Windows 继续作为 24h 常驻服务器。不要把 Mac 设计成服务器。

## 已知环境与固定约束

- GitHub 用户：`CyberMist123`
- upstream：`KKarsyline/Collar_watch`
- 用户域名：`ler428.xyz`
- 计划健康上传域名：`health.ler428.xyz`
- 用户生活时区：`Australia/Sydney`
- 24h 常驻服务器：Windows 上现有 Cyberboss 环境
- Mac 经常使用，但会合盖/关机；Mac 只能承担开发、签名、重新安装，不得成为运行时依赖
- 设备：Apple Watch + iPhone + Mac
- 第一阶段优先让数据链和按需 AI 查询跑通；不要做大重构
- Health 原始数据、token、签名私密信息、Apple Team ID 不得提交 Git
- 健康数据默认不写入长期 memory / episodes；AI 需要时才通过 health tool / skill 查询

## 总原则

1. 先做只读检查，再改代码。
2. 每个阶段单独 commit；不要把所有修改揉成一个提交。
3. 保留 upstream remote，方便以后同步上游。
4. 不删除 upstream 现有能力，不改成另一套架构。
5. 不把 Mac 变成健康数据 server。
6. 不把秘密写进仓库；使用 `.local` / env / Keychain 等本地方式。
7. 不自行声称 stress 是医疗指标。可以做“生理压力/恢复信号”，但必须可解释、基于个人 baseline，并返回 confidence / factors。
8. Apple Health 的 HRV 第一版直接使用 HealthKit 已保存的 HRV SDNN 样本；不要把 SDNN 样本再次错误计算成 RMSSD。
9. 所有日期边界、睡眠日、today、baseline 都必须使用 `Australia/Sydney` IANA timezone，不能继续依赖固定 `UTC+10/+11`。
10. 不要求用户每一步确认。遇到真正需要人工操作的 Apple 授权、Developer Mode、Trust、Team 选择时才停下，并明确告诉用户只需要点什么。

---

# Phase 0 — GitHub fork + 本地安全基线

先执行：

```bash
gh auth status
git --version
xcodebuild -version
xcrun devicectl --version || true
xcodegen --version || true
```

如果 `gh` 未登录，停止并只让用户完成 GitHub 登录。

然后 fork 并 clone：

```bash
mkdir -p ~/Developer
cd ~/Developer
gh repo fork KKarsyline/Collar_watch --clone
cd Collar_watch
```

确认：

```bash
git remote -v
git status --short
git branch --show-current
git log -1 --oneline
```

预期：

- `origin` → `CyberMist123/Collar_watch`
- `upstream` → `KKarsyline/Collar_watch`

如果 fork 已存在，不重复创建；直接 clone 用户 fork，并确认 upstream 指向原作者。

创建工作分支：

```text
feat/cybermist-health-watch
```

不要直接在 main 开发。

### Phase 0 验收

- fork 存在于 `CyberMist123/Collar_watch`
- origin/upstream 正确
- 工作树干净
- 已创建 feature branch

Commit：无。

---

# Phase 1 — 只读理解 upstream，确认最小修改面

只读这些核心路径，避免全仓库无脑通读：

```text
README.md
watch/project.yml
watch/Sources/Config.swift
watch/Sources/Config.local.swift.example
watch/Sources/HealthCollector.swift
watch/Sources/Scheduler.swift
watch/Sources/CollarWatchApp.swift
server/health_store.py
mcp_server/server.py
.env.example
```

确认并记录：

- Watch 端 endpoint/token 从哪里读取
- 当前 HRV 类型和单位
- 当前 HealthKit anchor / retry 行为
- 后台 refresh 逻辑
- `health_now` / `health_detail` 输出结构
- `HEALTH_TZ_OFFSET_HOURS` 影响的所有代码路径
- command / realtime heart-rate 路径是否仍完整

不要改功能，只输出一个很短的修改清单，然后继续执行。

---

# Phase 2 — Sydney timezone 修复

目标：彻底消除固定 UTC offset 对“今天 / 睡眠日 / 最近 N 天 / 聚合边界”的影响。

要求：

- 新增优先配置：

```text
HEALTH_TIMEZONE=Australia/Sydney
```

- Python 使用标准库 `zoneinfo.ZoneInfo`。
- 如果要兼容旧 `HEALTH_TZ_OFFSET_HOURS`，仅作为 legacy fallback，并在 README 标记 deprecated。
- 所有以下逻辑必须改为 IANA timezone：
  - local day
  - sleep date
  - daily filename boundary
  - 7/14/28d baseline date window
  - MCP 展示时间
  - command/result timestamp display（若适用）
- 写 DST 测试，至少覆盖 Sydney 的夏令时切换前后。
- 不硬编码 `+10` / `+11`。

Commit：

```text
fix(timezone): use IANA timezone for health aggregation
```

---

# Phase 3 — Watch endpoint 与秘密配置

目标 endpoint：

```text
https://health.ler428.xyz/api/health
```

但不要把 token 写进可提交文件。

要求：

- `Config.swift` 可提交公开 endpoint。
- `Config.local.swift` 继续 gitignored，保存 token / 本地敏感配置。
- 若 Team ID / Bundle ID 需要本地覆盖，优先增加不提交的 local config 或脚本参数，不把用户身份信息硬编码进 upstream 通用源码。
- 确保 `.gitignore` 覆盖本地 token、DerivedData、签名配置。

保持 upstream command API 兼容：

```text
GET  /command
POST /command/result
```

不要在本阶段实现公网 server，只准备 Watch 客户端契约。

Commit：

```text
feat(watch): configure personal health ingest endpoint safely
```

---

# Phase 4 — Mac 一键 build / install

新增：

```text
scripts/macos/collar-build.sh
scripts/macos/collar-status.sh
scripts/macos/collar-renew.sh
```

目标：用户不需要每次打开 Xcode 手动点 Run。

## collar-build.sh

职责：

1. 检查 Xcode / xcodegen。
2. `xcodegen generate`。
3. 使用 `xcodebuild` + automatic signing 构建 Watch app。
4. 使用 Apple 官方命令行设备工具完成可行的真机安装。
5. 不在日志打印 token、Team secret 或完整 provisioning 内容。
6. 成功后写本机状态文件，例如：

```text
~/Library/Application Support/CollarWatch/last-success.json
```

至少记录：

```json
{
  "last_success_at": "...",
  "git_sha": "...",
  "build_result": "success"
}
```

如果当前 Xcode / watchOS 组合无法稳定纯命令行安装：

- build 尽量自动化；
- 只把最终“选择 Team / Trust / 第一次安装”保留为人工步骤；
- 不写 GUI click automation 去猜 Xcode 坐标。

## collar-status.sh

只读检查：

- repo SHA / dirty status
- xcodegen 是否存在
- Xcode version
- iPhone / Watch 是否被工具识别
- last successful renewal time
- 是否建议续签

不得修改设备。

## collar-renew.sh

逻辑：

```text
if last_success < 4 days:
    exit 0
if Watch/iPhone unavailable:
    skip cleanly, exit 0
else:
    run collar-build.sh
```

目标是约每 4 天滚动续签一次，而不是等 7 天临界点。

不要每次 Mac 登录都重编译。

Commit：

```text
feat(macos): add automated watch build and renewal scripts
```

---

# Phase 5 — LaunchAgent 自动检查

新增模板：

```text
scripts/macos/xyz.ler428.collar-renew.plist
scripts/macos/install-renew-agent.sh
scripts/macos/uninstall-renew-agent.sh
```

要求：

- 用户登录时检查一次；
- Mac 开着时每日检查一次即可；
- 真正 build 仍由 `collar-renew.sh` 的“4 天阈值”决定；
- Watch 不可达时静默 skip，不反复弹窗；
- 不要求 Mac 常驻；关机/合盖不会影响 Windows server；
- 下次 Mac 登录后再检查即可；
- 日志放到用户 Library 下，限制大小，不能记录 secrets。

安装前先用 `plutil -lint` 验证 plist。

Commit：

```text
feat(macos): add launch agent for periodic watch renewal
```

---

# Phase 6 — Health 数据清洗 / AI skill 契约

这部分可以在 fork 内完善代码，但不要在 Mac 上假装部署 Windows server。

保留 upstream 的两级调用思路：

```text
health_now()
health_detail(metric, range)
```

`health_now` 必须短，适合 agent 首次探查；`health_detail` 才返回较长时间序列。

建议 `health_now` 最终包含：

```text
freshness
sleep summary
HRV SDNN vs personal baseline
resting HR vs personal baseline
respiratory rate（若有）
activity summary
stress_signal summary（若数据足够）
```

### baseline

第一版使用透明、可重算的方法：

- 7d / 14d / 28d rolling baseline
- 优先 median
- 离群稳健尺度可使用 MAD
- 明确 sample_count
- 数据不足时返回 `insufficient_data`

### stress_signal

允许实现，但名称必须明确为派生信号，例如：

```json
{
  "stress_score": 0,
  "level": "low|normal|elevated|high",
  "confidence": 0.0,
  "factors": {},
  "possible_confounders": []
}
```

原则：

- 这是“生理压力/恢复信号”，不是医学诊断，也不是 Apple 官方压力测量。
- 使用个人 baseline，不使用统一人群 HRV 阈值。
- 第一版候选输入：
  - HRV SDNN deviation
  - resting HR deviation
  - sleep duration / sleep debt
  - respiratory-rate deviation（若有）
  - recent activity / exercise load 作为混杂因素或调节项
- 不让天气直接进入 score。
- 返回 factors，让 AI 能解释分数来源。
- 数据不足时宁可不打分。
- 不声称复制 StressWatch 算法。

为 stress_signal 写 deterministic tests。

Commit：

```text
feat(health): add explainable personal stress signal
```

---

# Phase 7 — Windows 交接契约，不在 Mac 部署

最终生成：

```text
WINDOWS-SERVER-HANDOFF.md
```

只写 Windows 后续需要做什么：

```text
health.ler428.xyz
  ↓ Cloudflare Tunnel
Windows localhost ingest
  ↓
health_store.py
  ↓
health_now / health_detail
  ↓
Cyberboss tool host / skill
```

公网最小暴露：

```text
POST /api/health
GET  /command          # 只有启用按需实时心率时
POST /command/result   # 同上
```

要求交接文档明确：

- HTTPS
- 长随机 token
- request body limit
- metric allow-list
- rate limit
- no public MCP
- no public file browsing
- no public SQLite / raw data query
- server timezone `Australia/Sydney`
- Windows 是唯一 24h runtime owner
- Mac 关机不能影响数据链

不要在 Mac 上创建 Cloudflare Tunnel，除非用户明确要求。

Commit：

```text
docs: add Windows health server handoff
```

---

# Phase 8 — 真机验收

按顺序进行，不要一次测试所有能力：

1. Watch app 能 build。
2. 第一次 HealthKit 授权正常。
3. Watch app 能安装并启动。
4. 不配置真实 server 时，网络错误不会丢 HealthKit anchor。
5. 配置测试 endpoint 后，可上传最小 sample。
6. HRV 识别为 SDNN / ms。
7. Sydney 时区和 DST 测试全部通过。
8. `collar-status.sh` 输出正常。
9. 手动执行 `collar-renew.sh`：未满 4 天应 skip。
10. 用测试状态模拟 >4 天：设备可达时 build/install；设备不可达时 clean skip。
11. LaunchAgent plist lint 通过。
12. repo 中不存在 token / Health 原始数据 / Apple 私密签名材料。

如果 realtime heart-rate 原 upstream 功能仍可用，再单独验：

```text
measure_heart_rate
→ command
→ Watch 30s measurement
→ result
```

不要为了这个功能破坏基础同步链。

---

# 最终汇报格式

不要长篇复述代码。最终只给：

```text
Fork:
Branch:

Commits:
- SHA — summary

Passed:
- ...

Manual Apple steps still required:
- ...

Windows handoff:
- path to WINDOWS-SERVER-HANDOFF.md

Risks / unresolved:
- ...
```

如果某一步因为 Apple 签名/Developer Mode/真机 Trust 必须用户点按钮，只在那个点停下，准确告诉用户：

```text
现在请在 Xcode / iPhone / Watch 上点击什么；完成后回复“好了”。
```

除此之外连续执行，不要每个阶段都询问确认。