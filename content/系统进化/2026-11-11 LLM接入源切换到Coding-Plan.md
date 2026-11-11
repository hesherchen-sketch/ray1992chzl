# 2026-11-11 LLM 接入源切换到 Coding Plan

## 一、授权与要求

**用户原话**：「帮我改一下整个系统的llm 接入源，换成arkcli的coding plan，Agent plan的额度用完了」

口径：全系统 LLM 通道（4 个 canghai profile、12 个 agent 型 cron、默认 home 会话、研究链 shell 硬编码、D1 安全网）统一改走 `volcengine-coding-plan`。

## 二、结论速览

| 项 | 改前 | 改后 | 验证 |
|---|---|---|---|
| 4 个 profile（fetch/research/plan/review） | volcengine-agent-plan | **volcengine-coding-plan** | `hermes -p <p> fallback list` 四条全对 |
| 12 个 agent cron | volcengine-agent-plan/deepseek-v4.1-flash | **coding-plan/deepseek-v4.1-flash** | `cron/jobs.json` Counter 全为 coding-plan |
| 默认 home 会话 | minimax-cn/MiniMax-M3.1-Flash | **coding-plan/deepseek-v4.1-flash** | `hermes fallback list` 主=deepseek-v4.1-flash |
| 备池 fallback | agent-plan（同厂商、也快耗尽） | **minimax-cn/MiniMax-M3.1-Flash**（异厂商） | 真实调用 19,829 token 成功 |
| 研究链 shell 默认池 | `CHZL_RESEARCH_PROVIDER:-volcengine-agent-plan` | **coding-plan** | `bash -n` 通过 |
| 研究链 shell 备池 | coding-plan（与主池同厂商） | **minimax-cn/MiniMax-M3.1-Flash** | 同上 |
| D1 安全网（ensure_watchlist_plan） | agent-plan/deepseek-v4-flash-ga-260731 | **coding-plan/deepseek-v4.1-flash** | `py_compile` 通过 |

## 三、逐项真相（可核验）

1. **真实调用证明 endpoint 已切换**：`logs/agent.log` 17:51:52 记录
   `provider=custom base_url=https://ark.cn-beijing.volces.com/api/coding/v3 model=deepseek-v4.1-flash`；
   17:47:39 之前的调用仍是 `api/plan/v3`（旧会话未重启，符合预期）。
2. **真实一次会话**：`hermes -p canghai-plan -z "只回复 OK"` → 返回 OK，input 20,572 token；
   `hermes -p canghai-research chat --provider volcengine-coding-plan` → 返回 OK；
   `--provider minimax-cn -m MiniMax-M3.1-Flash` → 返回 OK，19,829 token（备池可用）。
3. **coding 端点三模型可跑**：deepseek-v4.1-flash / deepseek-v4.1-flash-economy / ark-code-latest 均 HTTP 200。
4. **额度事实（arkcli usage plan，SSO 只读查询）**

| 计划 | 5h/会话 | 周 | 月 |
|---|---|---|---|
| agent-plan（large） | 5.04%（1,259/25,000） | 38.93%（34,065/87,500） | **93.08%（219,412/235,725）← 用户说的"用完"** |
| coding-plan | **1.38%**（18:07 重置） | 6.19% | 24.50% |

→ 结论：Agent Plan 是**月额度**将满（还剩 6.9%），不是 5h 窗口爆；Coding Plan 三档都很空。

5. **沉默回归点（这次一并修掉）**：`run_research_shell.sh` 与 `ensure_watchlist_plan.py` 都把 agent-plan **写死在代码里**，
   只改 profile 等于没改——10-30 事故就是这种"配置看着换了、调度还在旧池"。

## 四、风险（必须记住）

- Coding Plan 的 **session 窗口偏小**：本次验证消耗约 7 万 token 记了 1.38%，线性折算窗口约 500 万 token 量级；
  而 10-29 全链实测约 40–45M。**明天 06:10 七连发有较大概率撞 session 墙。**
  兜底已就位：hermes 主调用失败会自动落到 minimax（异厂商），shell 的 attempt≥2 也切 minimax。
- 我的当前会话仍跑在 plan/v3（配置只对新会话生效），重启后才是 coding-plan。
- 未做：额度预检闸门（开跑前查 coding-plan session%，>50% 则本次直接走 minimax）。**建议做，需点头。**

## 五、变更文件清单

- `~/.hermes-canghai/profiles/{canghai-fetch,canghai-research,canghai-plan,canghai-review}/config.yaml`（model 块 → coding-plan；新增 `minimax-cn` provider 块；fallback → minimax）
- `~/.hermes-canghai/config.yaml`（`model.provider/default/base_url` → coding-plan；`fallback_providers` → minimax）
- `~/.hermes-canghai/cron/jobs.json`（12 个 agent job 的 provider/model）
- `scripts/research/run_research_shell.sh`（主池/备池/备池模型默认值）
- `scripts/plan/ensure_watchlist_plan.py`（D1 安全网 provider/model 默认值）
- 备份：`chzl_kg/系统进化/backups/model-switch-20261111-175010/`（jobs.json + 5 份 config）
- 审计：`data/evolution/2026-W46/cron_switch_audit.jsonl`

## 六、回滚

```bash
cp chzl_kg/系统进化/backups/model-switch-20261111-175010/{jobs.json,config.yaml,canghai-*-config.yaml} <对应位置>
```
