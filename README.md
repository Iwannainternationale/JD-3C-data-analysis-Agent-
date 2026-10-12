# 品牌分层分析 Agent（brand-analysis-agent）

对「品牌 × 三级类目」粒度经营数据完成 **提取 → 分层分类 → 分析 → 发布** 全流程的 agent。输入一份经营数据源（CSV），输出一份发布到 joyspace 的五分区分析报告。

## 快速使用

```bash
git clone <本仓库URL>
cd brand-analysis-agent        # 关键：必须在仓库目录内启动，项目级 skill 才生效
```

启动 Claude Code 后：
1. 发送 `prompts/brand-tier-analysis-agent.md` 的内容（主 prompt：分区定义、工作流、纪律条款）
2. 上传数据源 CSV（表头须含：主品牌 / 三级类目 / Net GMV / Net GMV_同比；列顺序不限，各度量后的占比/贡献列可选）
3. 流程自动执行：profiling → 口径确认（一次性提问）→ 读名录与市占表 → 分层分类 → 发布报告 + 回读校验

## 内含 Skills（5 个）

| Skill | 作用 |
|---|---|
| profile-brand-category-csv | 源数据一次性画像（8 大板块），把所有口径问题一次暴露 |
| read-keybrand-list | 读重点品牌名录（aitable），数量漂移安全，分页取齐并校验 |
| read-joyspace-marketshare | 按结构地图最小化读取市占表 JD 区块（品牌/类目 × 最新三周） |
| publish-joyspace-report | 本地报告一次性上传为新 joyspace 文档 + 全文回读校验 |
| install-brand-agent | 一键安装器：环境自检 → 权限校验 → skill 落位到用户全局目录 |

## 环境依赖（使用前必读）

| 依赖 | 用途 | 缺失后果 |
|---|---|---|
| Python 3 | profiling / 分类 / 报告生成脚本 | 无法运行 |
| webcli（o2）已安装并登录 | `webcli joyspace upload/export`（发布 skill） | 无法发布文档 |
| joyspace MCP server | 名录读取（aitable.*）、市占表读取（sheets.*） | 阶段 3 无法执行 |
| SSO 登录态 | webcli 与 MCP 的鉴权 | 全部外部读写失败 |
| 文档权限：`B1Nn3iBEdr4zwfdcP00H`（重点品牌名录） | 分区判定的名录依据 | 触发"停下问用户"，无法继续 |
| 文档权限：`V4prlF3VZv16pXY3GTNN`（市占工作簿） | 风险/预警区判定 | 同上 |

**新环境部署**：把本仓库 `.claude/skills/install-brand-agent/` 复制到你的 `~/.claude/skills/`（Windows 为 `C:\Users\<用户名>\.claude\skills\`），然后对 Claude 说 **"按照 install-brand-agent 执行"**，它会自检环境、校验两个文档的权限、把 4 个业务 skill 安装到你的全局目录并汇报清单。MCP 配置与 SSO 登录无法自动完成，安装器会给出手动指引。

## Skill 维护约定

- skill 内的结构锚点（如市占表 "行4表头 / Wn=行5+n"）会随线上文档演进漂移。执行中发现失配时，skill 指示重探一次并更新 **本地** skill 文件——请将此类修正 **提 MR 回本仓库**，让修正回流而不是散落在个人副本。
- 名录条数（2026-09 为 107）持续增长属预期工况，skill 自带实时 recordsCount 对账，无需改代码。
- 本仓库建议开启分支保护：skill 修正一律走 MR，保持分区判定口径单一来源。

## 口径速记（详见主 prompt）

- 判定单元 = 品牌名称 + 三级类目；判定优先级：异常 → 机会 → 次要 → 重点二次分类
- 同比口径、合并口径（同名品牌跨ID）、别名口径（括号归一化）在每次任务的口径确认环节由用户一次性拍板，全程固化
