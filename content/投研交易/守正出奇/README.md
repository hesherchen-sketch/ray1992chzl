# 守正出奇轨道（对比轨）

> 2026-11-13 用户钦定：「守正出奇 单独开一个文件夹 … 写从守正出奇的角度选股和交易，之前的盘前预案还是一样，相当于我自身有一个对比。」

## 这是什么

两条轨道，同一份 R1–R7，两种决策框架：

| | 盘前预案（D1） | 守正出奇轨（D2，本目录） |
|---|---|---|
| 核心问题 | 今天怎么买（买什么价、什么条件、什么闸门） | 今天市场在**为什么**定价，值不值得动用资源 |
| 选票方式 | execution_pool 3–6 只 + watch_pool 15–25 只 + B 计划 | **正股 1–3 只 + 奇标 0–2 只**，无观察池 |
| 换手节奏 | 每天重建池子（结构上就在换手） | **逻辑未变不换股**；每周换股上限 2 次 |
| 卖出口径 | 止损/止盈/竞价四档/T+1 10:30 不创新高 | 逻辑证伪为主，价格破位/期满为辅；**不设节点规则** |
| 是否下单 | `auto_trade=false`，等你确认 | **永不下单**（`mode=compare_only`） |
| 产物 | `data/plans/<date>/watchlist_plan.json` + `chzl_kg/投研交易/盘前预案/<date>.md` | `data/plans/<date>/shouzheng_plan.json` + 本目录 `<date>.md` |

## 每天怎么跑

```bash
source .venv/bin/activate
python scripts/plan/build_shouzheng_inputs.py --date $DATE --write   # 输入包（确定性）
# agent 读输入包 → 写 data/plans/$DATE/shouzheng_plan.json（pydantic 校验）
python scripts/plan/render_shouzheng_md.py --date $DATE              # 渲染本目录 md
```

cron：`canghai-plan · 守正出奇轨 (08:30)`，交易日 08:30 起跑（D1 08:00 之后、竞价 09:15 之前）；job id `61a6d09e1bfa`，前置脚本 `canghai_shouzheng_track_prep.sh`。
skill：`shouzheng_chuqi_v1`。契约：`trading/schema/shouzheng_plan.py`（`shouzheng_plan_v1`）。

## 怎么评估（这才是这个目录存在的意义）

按用户钦定口径：**集合竞价/建仓参考价买入 → 次日开盘/尾盘/高点卖出，有盈利即算赢**。逐日累积四项指标：

1. **收益**：T+1 / T+3 / T+5 收盘收益（以 `trade.entry.price_zone` 中值为参考价；未建仓不计）。
2. **稳定性**：`metrics.thesis_age_days`（越长越好）、`switches_this_week`（≤2）、平均持有天数。
3. **对照**：与同日 D1 `execution_pool` 的收益差；差在哪个判断上（级别 / 位置 / 时机）。
4. **证伪质量**：`falsifier_check` 里 `hit` 的判断对不对——这条最值钱，因为它检验的是「我们是否真的知道自己在赌什么」。

判断标准很简单：**如果本轨的收益不低于 D1 而换手次数明显更低，那说明「逻辑驱动」在 A 股短线里是有效的；如果不是，那就说明我们的级别/资金判断还不到位，需要回到 R2/R7 去补。**

## 纪律

- 本目录的文件**只由渲染器生成**，不手改（要改就改 JSON 再重渲染），否则两套口径必然打架。
- 选不出票就写空（`selection: []` + `trade.action: observe`）——空着是结论。
- 缺口必须写进 `gaps`，不许用常识补宏观。
