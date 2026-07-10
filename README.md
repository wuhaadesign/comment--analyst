# 抖音行业评论采集器（Douyin Comment Collector）

按「行业诉求」批量采集抖音视频的原始评论，并由 Agent 后台完成飞书多维表格写入与「瑞士极简风」HTML 分析报告的生成。本仓库是该 skill 的定义与配套模板。

> ⚠️ 本仓库仅包含 skill 的**定义文件**（`SKILL.md`）与**自检模板**（`references/optimization-template.md`）。`SKILL.md` 中描述的采集脚本（`run-collect.js`、`server.js`、`src/*`、`public/*`）为**用户在本地自建**的运行产物，不随本仓库分发。

## 这个 skill 做什么

按行业名、关键词矩阵与视频筛选条件，批量采集抖音视频的原始评论，核心链路：

```
life-ds 风格 HTML 诉求界面（6 项输入 + 关键词 LLM 拓展 + 一键复制 JSON）
  → Agent 生成一条「一键采集 bash」（写 config + 探测 Chrome 写 .env + 装依赖 + 跑采集）
  → 用户在本地终端粘贴运行 → 关键词搜索 → 漏斗过滤 → 逐视频采评论
  → data/comments_xxx.json 落盘（脚本到此为止，永不请求飞书）
  → 用户把 JSON 回传给 Agent
  → Agent 用当前用户身份新建并写入飞书多维表格
  → Agent 大模型分析评论 → 生成「瑞士极简风」HTML 报告
    （核心痛点 + 上品 / 客服 / 视频 / H5 / 投放五大经营建议，内嵌飞书表链接）
```

采集全程用**浏览器被动拦截**（监听页面自身发出的带签名 XHR 响应 + 滚动触发加载），**不逆向签名**；每个视频用独立标签页隔离，视频间随机长停顿以降低风控。

## 设计原则（采集与飞书完全解耦）

1. **采集脚本永不碰飞书**：本地脚本始终等价于 dry-run，只把评论落盘到 `data/comments_xxx.json`，不读取任何飞书凭证、不发起任何飞书请求。`.env` 里**只有 `CHROME_PATH` 一项**。
2. **一键采集 bash**：Agent 直接生成一条完整 bash 命令，内部自动写 `config/industry.json`（heredoc 内联，不走下载链接）、探测本机 Chrome 写 `.env`、`npm install`、`node run-collect.js`。用户只需粘贴运行。
3. **写飞书由 Agent 后台代做**：采集完成后用户回传 JSON，Agent 以「当前登录用户身份」通过 `lark-base` 技能新建 Base + 表并批量写入。表名 / token 默认不向用户暴露。

## 铁律（最高优先级）

- **铁律 A · 报告风格锁死瑞士极简**：HTML 分析报告必须用「瑞士极简风 / 国际主义平面风格」——白纸黑字 + 单一信号色瑞士红 `#E2231A`、无衬线字体、直角、发丝线分隔、大号标题、数字 `tabular-nums`；严禁装饰性 emoji、圆角卡片 + 左彩条、渐变堆砌、深色背景。
- **铁律 B · 两套视觉绝不混用**：报告 = 瑞士极简风；前置「诉求收集界面」= life-ds 风（主题色 `#387AFF` 蓝）。两套视觉系统完全独立，任何情况下不得互套。

## 运行环境要求

- 采集环节依赖**真机 Chrome + puppeteer + 抖音扫码登录 + 反风控长停顿**，**必须在用户本地机器运行**，无法在受限沙箱执行。
- 需本机安装 Node.js 与 Chrome。Chrome 路径由「一键采集 bash」自动探测并写入 `.env`，无需用户手填。
- 采集必须在**登录态**下运行，否则触发 `status_code 2483`。

## 本地项目目录结构

用户需在本地建一个专门的 `douyin-comment-collector` 文件夹，把项目文件完整放入（`config/`、`.env`、`chrome-profile/`、`data/` 首次运行自动生成）：

```
douyin-comment-collector/
├── run-collect.js       # 入口：搜索→过滤→采评论→落盘 data/*.json（永不碰飞书）
├── server.js            # 起本地 Web 服务（默认 3100），打开 life-ds 风格诉求界面
├── package.json         # 依赖清单
├── .env                 # 【自动生成】只有 CHROME_PATH
├── config/industry.json # 【自动生成】采集配置
├── src/                 # 采集内核（search / filter / comments / scraper-core / report）
├── public/              # life-ds 风格诉求界面（主题色 #387AFF）
├── chrome-profile/      # 【自动生成】抖音扫码登录态持久化（严禁公开）
└── data/comments_xxx.json # 【自动生成】采集产物（含真实评论，严禁公开）
```

## 使用步骤

```bash
# 0)（可选）打开诉求界面填写诉求
node server.js            # 浏览器访问 http://localhost:3100
                          # 填完点「一键复制json」→ 回对话粘贴给 Agent

# 1) 粘贴运行 Agent 给你的「一键采集 bash」
#    自动写 config + .env + 装依赖 + 采集，只落盘 data/comments_xxx.json

# 2) 把 data/comments_xxx.json 回传给 Agent
#    → Agent 后台写飞书多维表格 → 大模型分析 → 生成瑞士极简风 HTML 报告
```

## 首次启动自检 & 优化流程

不同 Agent 依附的运行环境能力不同（能否读写本地文件、能否直连飞书、能否调大模型）。每个新接入的 Agent **首次进入采集流程时**先探一次自身能力，逐项与用户确认哪些「B 类人工点」可代做 / 简化，并把结果记录到项目根目录 `OPTIMIZATION.md`。

- **A 类护栏（A1–A3）永远保留人工**：业务诉求确认、抖音扫码登录 / 滑块 / 验证码 / 反风控长停顿、飞书身份授权。
- **B 类（B1–B7）逐项确认**：建目录、环境准备、执行采集、JSON 回传、飞书写入、LLM 拓展 + 分析 + 报告、失败补跑。
- 优化只减少**用户手动操作量**，绝不降低结果可靠性；任何绕过登录态 / 逆向签名 / 臆造内容的所谓自动化一律禁止。
- 模板见 [`references/optimization-template.md`](references/optimization-template.md)；一个 Agent + 环境一条记录，追加不覆盖。

## 敏感文件

以下文件**严禁提交公开仓库**，`.gitignore` 已默认忽略：

- `.env`（本机路径）
- `chrome-profile/`（抖音登录态）
- `data/*.json`（含真实用户评论）

## 文件清单

| 文件 | 说明 |
| --- | --- |
| `SKILL.md` | skill 定义（触发词、链路、约束、报告规范、自检流程） |
| `references/optimization-template.md` | 首次自检 `OPTIMIZATION.md` 模板 |
| `README.md` | 本说明文件 |
