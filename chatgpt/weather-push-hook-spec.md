# 天气推送（Hook 版，无 520 页）—— Codex 执行规格

> 定位：`AI-Closeout-时区-天气-月经-Apple-Health-MVP-Codex执行.md` 的 **Phase 4 精简版**。
> 本版**砍掉 520 配置页**、**不做每轮查询 tool**；天气以「定时抓取 + 后台库 + closeout hook 注入」形态存在，
> AI 每轮零 token 开销，只有当天有预警才缝几行到上下文。
>
> 继承原执行单的全部**绝对边界**（不改 immutable release、用分支/worktree、不新增常驻服务、
> 单一 writer/closeout 链、token/.env/原始数据不进 Git、AMap key 从 env 读且网页不回显）。

## 一、形态总览

```
定时抓取器(REST, 无常驻)  ──写──▶  后台库(7天实测 + 7天预报 + 今日预警旗标)
                                          │
                                          ├─读─▶  closeout hook：每天开场时，若 enabled 且今日有未缝的预警 → 缝一条简报进上下文（当日至多一次）
                                          └─读/写─▶  city 字段：AI 极少写（换地区时），hook 与抓取器读
```

- **不做**：520 配置页、每轮查询 `weather` tool、AMap（除非后续要，默认只 Open-Meteo）、新常驻/MCP 服务。

## 二、数据源与抓取

- Provider：`open_meteo`（默认，适合 Sydney/海外）。纯 REST：
  - Geocoding API：城市名 → 经纬度；
  - Forecast API：当前、未来 7 天每日高/低温、体感、降雨概率、降水量；`timezone=auto`；
  - 10 秒 timeout、最多 3 次 retry、30 分钟缓存。
- `amap` 作为可选 provider 保留接口（key 从现有 env 读，不回显）；默认不启用。
- 抓取频率：每日至少 1 次（建议每 3–6 小时刷新预报，便于预警及时）；调度用现有定时机制，不新起服务。
- 缓存过期且请求失败：返回旧缓存并标 `stale` + 最后更新时间。

## 三、后台库（留存）

- **最近 7 天实测**：每日一条聚合（实际高/低温、是否降水），追加式，滚动保留 7 天。
- **未来 7 天预报**：每次抓取覆盖写。
- 存本地文件（JSON/JSONL），不进 Git，不入长期 memory/episodes（继承边界：生活数据默认不写记忆）。

## 四、预警与「当日单次注入」

- **预警条件**（任一即触发，阈值可配）：
  - 🌧️ 降雨：未来 24h 降雨概率 ≥ `rain_prob_pct`（默认 60）或有降水量；
  - 🌡️ 温度剧变：今日最高或最低较**昨日同项**变化 ≥ `temp_delta_c`（默认 6℃）。
- **注入 = closeout hook，不是 tool**：
  - 每天构建 TG AI 开场上下文时，hook 读后台库；
  - 若 `inject_enabled` 且今日有预警且**今日尚未缝过**（用 `stitched_date` 幂等守卫）→ 缝一条简短天气预警进上下文，并记 `stitched_date=今日`；
  - 无预警 → 不注入，仅留后台库；
  - 一天至多缝一次，重复构建不重复注入。
- `inject_enabled` 默认遵循原边界 #5（默认关闭 AI 注入）；由 Owner 显式开启本功能。

## 五、配置字段（后台，AI 可改 city）

```jsonc
{
  "enabled": true,
  "provider": "open_meteo",     // open_meteo | amap
  "city": "Sydney",             // AI 可改：Owner 换地区时
  "country": "AU",
  "lat": null, "lon": null,      // geocoding 自动解析后回填，可手动覆盖
  "rain_prob_pct": 60,
  "temp_delta_c": 6,
  "inject_enabled": true,        // 是否允许每日预警注入
  "stitched_date": null          // 幂等守卫，勿手填
}
```

- **改 city 的写路径**：优先复用 cyberboss 现有配置写机制；若无，加一个**只写不查**的窄动作
  `set_weather_location(city, country?)`（重解析 lat/lon，清 `stitched_date`）。
  该写极少触发（Owner 说"我到 X 了"时 AI 调一次），**不做成每轮存在的查询 tool**。

## 六、状态机

```
not_configured | connected | stale | failed | disabled
```

未配置时返回 `not_configured`，不猜 IP/GPS。

## 七、验收清单

- [ ] 全 mock fetch，不依赖真实网络：geocoding、current/forecast 解析、timeout/retry、stale cache、not_configured、provider 非法被拒。
- [ ] 预警阈值：降雨 ≥60% 触发、温度较昨日 ≥6℃ 触发；边界值测试。
- [ ] 当日单次注入幂等：同日多次构建上下文只缝一次；跨日重置。
- [ ] `inject_enabled=false` 时完全不注入，只落后台库。
- [ ] 留存：实测滚动 7 天、预报覆盖 7 天。
- [ ] `set_weather_location('Bali')` 改 city 后重解析经纬度、清 `stitched_date`，下次抓取用新城市。
- [ ] key/原始数据不进 Git 与日志；无新增常驻服务；注入走现有 closeout 链、无第二 writer。

提交按 cyberboss 仓约定，用分支/worktree，别推 secrets。
