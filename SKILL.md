---
name: douyin-comment-collector
description: 抖音行业评论采集、分析与经营方案生成工具。按「行业诉求」（行业名、关键词矩阵、视频筛选条件）批量采集抖音视频的原始评论，链路为：先出 life-ds 风格 HTML 诉求收集界面（6 项输入 + 关键词经 LLM 拓展，主题色 #387AFF，一键复制 JSON 回填）→ 生成一条"一键采集 bash"命令交用户在本地终端运行 → 关键词搜索 → 漏斗过滤 → 浏览器被动拦截采集评论 → 只落盘 data/*.json（采集脚本永不碰飞书）→ 由 Agent 用当前用户身份在后台自动新建并写入飞书多维表格 (Bitable) → 再由 Agent 对评论做大模型分析并生成一份「瑞士极简风 (Swiss / International Typographic Style)」HTML 分析报告：围绕用户给定赛道总结核心痛点，并给出上品 / 人工客服配置 / 短视频脚本分镜 / H5 落地页 / 投放五大经营建议（每条都对应真实录入字段并附理由）。USE WHEN：用户要按行业或关键词批量抓取抖音视频评论、把抖音评论导出到飞书多维表格、生成抖音评论分析报告、要基于评论给本地生活商家的上品/客服/视频/H5落地页/投放建议、打开或填写采集诉求配置界面。DO NOT USE WHEN：非抖音平台的评论采集、通用网页抓取、不涉及抖音评论的飞书表格操作、纯知识问答。注意：采集环节依赖真机 Chrome (puppeteer) + 抖音扫码登录 + 反风控长时间运行，必须在用户本地机器执行，不能在受限沙箱内运行；本 skill 负责指导安装、编排配置、飞书写入、评论分析以及报告生成等编排工作。飞书写入由 Agent 走 lark-base 技能后台完成，用户全程无需配置飞书凭证、无需感知表名/token。
---

# 抖音行业评论采集器（Douyin Comment Collector）

## ⛔ 铁律（最高优先级，违反即作废重做）

> 以下两条为顶格强约束，凌驾于本文件其余所有描述。生成任何产物前先核对这两条；违反任意一条，产物一律作废、必须重做。

- **铁律 A · 报告风格锁死瑞士极简**：Agent 生成的 **HTML 分析报告**必须用「瑞士极简风 / 国际主义平面风格 (Swiss / International Typographic Style)」——白纸黑字 + 单一信号色瑞士红 `#E2231A`、无衬线字体 (Helvetica Neue / Inter / Geist)、直角 `border-radius:0`、发丝线分隔、大号 billboard 标题、数字 `tabular-nums`；**严禁**装饰性 emoji、圆角卡片 + 左彩条、渐变堆砌、深色背景、抖音来客蓝 / 红等其它主题色。具体规范见「报告视觉风格」节，交付前必须逐条过完该节的「交付前自检清单」。
- **铁律 B · 两套视觉绝不混用**：**报告 = 瑞士极简风**；**前置「诉求收集界面」= life-ds 风（主题色 `#387AFF` 蓝）**。二者是两套完全独立的视觉系统，任何情况下都不得把 life-ds 风格套到报告上、也不得把瑞士极简套到诉求界面上。落笔前先自问"我现在做的是报告还是诉求界面"，再选对应风格。

按「行业诉求」批量采集抖音视频的原始评论。**采集脚本本地只负责抓取 + 落盘 JSON，飞书写入由 Agent 在后台代做**。核心链路：

```
life-ds 风格 HTML 诉求界面（6 项输入 + 关键词 LLM 拓展 + 一键复制 JSON）
  →  Agent 生成一条「一键采集 bash」（自动写 config + 探测 Chrome 写 .env + 装依赖 + 跑采集）
  →  用户在本地终端粘贴运行  →  关键词搜索  →  漏斗过滤  →  逐视频采评论
  →  data/comments_xxx.json 落盘（脚本到此为止，永不请求飞书）
  →  用户把 data/comments_xxx.json 回传给 Agent
  →  Agent 用当前用户身份自动新建并写入飞书多维表格（表名、token 用户全程无感知）
  →  Agent 对评论做大模型分析 → 生成「瑞士极简风」HTML 报告（核心痛点 + 上品/客服/视频/H5/投放五大经营建议，报告内嵌可点击飞书多维表格链接）
```

采集全程用**浏览器被动拦截**（不逆向签名）：监听页面自身发出的带签名 XHR (XMLHttpRequest) 响应，配合滚动触发加载；每个视频用独立标签页隔离，视频间随机长停顿以降低风控。

## 设计原则（方案 A：采集与飞书完全解耦）

1. **采集脚本永不碰飞书**：本地 `run-collect.js` 始终等价于 dry-run —— 只把评论落盘到 `data/comments_xxx.json`，**不读取任何飞书凭证、不发起任何飞书请求**。`.env` 里**只有 `CHROME_PATH` 一项**，不再需要 `FEISHU_*`。
2. **一键采集 bash**：Agent 直接生成一条完整 bash 命令交给用户，命令内部自动完成：写 `config/industry.json`（用 heredoc 内联写入，**不要用 curl/下载链接**，避免下到 HTML 页面）、探测本机 Chrome 路径并写 `.env`、`npm install`、`node run-collect.js`。用户只需粘贴运行这一条。
3. **写飞书由 Agent 后台代做**：采集完成后，用户把 `data/comments_xxx.json` 内容回传给 Agent，**Agent 用「当前登录用户身份」通过 `lark-base` 技能自动新建 Base + 表并批量写入**。表名、base token、table id 由 Agent 内部持有，**默认不向用户暴露**，用户全程无感知。仅当用户主动索要链接时才返回可访问 URL。

## 运行环境要求（重要）

- 采集环节依赖 **真机 Chrome + puppeteer + 抖音扫码登录 + 反风控长停顿**，**必须在用户本地机器运行**，无法在受限沙箱执行。
- 需本机安装 Node.js 与 Chrome。**Chrome 路径由「一键采集 bash」自动探测并写入 `.env`**，无需用户手填。
- `server.js` 会监听端口（默认 3100），仅可在本地起服务，切勿在禁止监听端口的环境中运行。飞书写入不依赖 server.js。

## 本地项目文件夹与目录结构（硬规则，务必先建好再操作）

> 采集脚本跑在**用户本地机器**，涉及多个脚本、配置、登录态与产物文件。**必须先让用户在本地建立一个专门的项目文件夹，把本 skill 的全部内容放进去，再开始任何操作**；绝不能让用户散乱地在桌面/下载目录里东放一个脚本、西放一个 config。进入采集流程时，Agent 必须先明确告知用户下面这两条，缺一不可：

1. **必须建一个专门的项目文件夹**（硬规则）：统一放在一个目录里，推荐 `~/Desktop/douyin-comment-collector`（macOS）或 `~/douyin-comment-collector`（Linux）。后续所有命令都在这个目录内执行；「一键采集 bash」第一行的 `cd` 就指向它。**不要**把脚本、config、data 分散到不同目录，否则 `run-collect.js` 找不到相对路径下的 `config/`、`src/`、`data/`。
2. **文件夹结构必须长这样**（硬规则）：把本 skill 的项目文件完整放入该文件夹，最终结构如下（`config/`、`.env`、`chrome-profile/`、`data/` 首次运行时由「一键采集 bash」或脚本自动生成，无需用户手建）：

```
douyin-comment-collector/          # ← 用户在本地建的专门项目文件夹（所有操作都在这里）
├── run-collect.js                 # 入口：关键词搜索→漏斗过滤→采评论→落盘 data/*.json（永不碰飞书）
├── server.js                      # 起本地 Web 服务（默认 3100），打开 life-ds 风格诉求界面
├── package.json                   # 依赖清单（puppeteer 等），npm install 用
├── .env                           # 【自动生成】只有 CHROME_PATH 一项，无任何飞书凭证
├── config/
│   └── industry.json              # 【自动生成】采集配置（heredoc 内联写入，非下载）
├── src/                           # 采集内核
│   ├── search.js                  #   关键词综合搜索（需登录态，否则 2483）
│   ├── filter.js                  #   漏斗过滤（发布时间 / 评论量下限）
│   ├── comments.js                #   逐视频采评论（独立标签页 + 滚动触发 + 反风控长停顿）
│   ├── scraper-core.js            #   浏览器被动拦截内核（监听带签名 XHR，不逆向签名）
│   └── report.js                  #   本地 HTML 报告生成（可选；正式报告由 Agent 后台生成）
├── public/                        # life-ds 风格诉求界面（主题色 #387AFF）
│   ├── index.html                 #   6 项输入 + 关键词 LLM 拓展 + 「一键复制json」主按钮
│   ├── app.js                     #   交互 + 一键复制 config JSON 到剪贴板
│   ├── styles/                    #   life-ds tokens / base / components
│   └── assets/                    #   sprite.svg 等图标资源
├── chrome-profile/                # 【自动生成】抖音扫码登录态持久化（严禁提交公开仓库）
└── data/                          # 【自动生成】采集产物落盘目录
    └── comments_xxx.json          #   每次采集的评论 JSON（回传给 Agent，含真实评论，严禁公开）
```

> 判断依据：在项目根目录执行 `ls` 应能看到 `run-collect.js`、`server.js`、`config/`、`src/`、`public/`。若缺失，说明项目文件没放全或不在正确目录，先补齐再继续，不要硬跑。

## 触发词

- "采集抖音评论 / 抓取某行业的抖音视频评论"
- "按关键词批量采集抖音评论并导出飞书多维表格"
- "打开采集配置界面 / 填写采集诉求"
- "生成抖音评论分析报告"

## 诉求收集（强制前置步骤）

**触发采集流程、需要收集用户行业诉求时，必须先生成一个 HTML 诉求收集界面，绝不在对话里用纯文字表格逐项提问。**

### 界面风格（强制）

- 该 HTML 界面**必须使用 `life-design-system`（@life-ds，life design system）的组件与视觉风格**（抖音来客白底平铺风格）。
- 生成前应加载 `life-design-system` skill，套用其 tokens / 组件（表单项、标签输入、下拉、开关、按钮等）与配色规范；对应静态资源为 `public/styles/*.css`、`public/assets/sprite.svg`。
- **主题色（强制）**：主题色 = life-ds 品牌主色 `--color-primary-normal` = **`#387AFF`**（蓝色），禁止使用青色或自行臆造的其他色值。全部主题色必须走 token 引用，不硬编码到具体元素上。推荐配套色值：
  - `--color-primary-normal`: `#387AFF`（默认主色：主按钮、序号圆点、开关开态、聚焦描边、关键词标签文字）
  - `--color-primary-hover`: `#5590FF`（悬停态）
  - `--color-primary-active`: `#2A63D6`（点击态）
  - `--color-primary-light-normal`: `#EBF2FF`（浅色主色：品牌标签底、提示条底、聚焦光晕）
  - 若已接入 `life-ds-tokens.css`，应直接引用其中的 `--color-primary-*` token；上述十六进制值仅为无法读取 token 文件时的等价回退。
- 打开方式：本地起静态服务后访问（`node server.js` → `http://localhost:3100`），或直接在浏览器打开生成的静态 HTML。

### 一键复制 JSON（强制交互）

- 界面**必须提供一个名为「一键复制json」的主按钮**，点击后把用户填写的 config JSON（字段结构见下文 config 示例）**一键复制到系统剪贴板**，便于用户回到对话中直接粘贴、快速回填诉求给 Agent，无需下载文件。
- 复制成功后**给出轻提示 toast**（如「已复制到剪贴板，回到对话粘贴给 Agent 即可」），不要用会阻塞的 `alert`。
- 复制实现：优先 `navigator.clipboard.writeText`，不支持时回退到隐藏 `textarea` + `document.execCommand('copy')`，仍失败再兜底弹出全文供手动复制。
- 可保留「下载 config.json」作为次级按钮（需要落盘给本地 CLI 时用），但**主操作是「一键复制json」**。

### 界面只收集这 6 项用户输入

| # | 字段 | 说明 |
| --- | --- | --- |
| 1 | 行业 / 主题名称 | 要分析的行业或主题 |
| 2 | 关键词（种子词） | 搜索用关键词，作为 LLM 拓展的种子 |
| 3 | 发布时间窗口 | 只采多少天内发布的视频 |
| 4 | 评论量下限 | 只采评论数 ≥ N 的视频 |
| 5 | 目标视频数 | 一共采多少个视频 |
| 6 | 是否采二级回复 | 评论下的回复是否一起采 |

**其余参数一律用系统推荐默认值，不在界面暴露**（避免打扰用户）：

| 参数 | 默认值 |
| --- | --- |
| 滚动轮数 scrollRounds | 30 |
| 输出方式 output | 采集脚本只落盘 JSON（飞书写入由 Agent 后台代做，界面不再暴露该开关） |

### 关键词 LLM 拓展（强制）

- 用户在界面填写的关键词**仅作为「种子词」**，不能只搜用户原始输入。
- 生成 `config` 之前，**必须由 LLM 基于种子词做同义 / 近义 / 场景化拓展**，产出更全的关键词矩阵后写入 `keywords`。
- 示例：种子「旧房改造」→ 拓展为「旧房改造、老房翻新、二手房装修、老破小改造、拆改、旧房翻新预算、老房子重装」等。
- 拓展逻辑：围绕行业/主题，覆盖同义词、口语说法、细分场景、预算/痛点等相关表达；数量适度（一般 5–15 个），去重后写入。

## 输入端：结构化诉求界面

界面为**纯静态 HTML + life-ds CSS**（抖音来客白底平铺风格，主题色 `#387AFF`），文件在 `public/`：

- `public/index.html`：诉求表单（按上文「诉求收集」规则：界面收集 6 项输入，关键词经 LLM 拓展，其余参数用默认，主按钮为「一键复制json」）。
- `public/app.js`：交互（关键词标签、下拉、开关、校验、生成 config、**一键复制json 到剪贴板**、下载 / 提交 config）。
- `public/styles/*.css` + `public/assets/sprite.svg`：tokens/base/components 与图标 sprite。

config 结构（示例，`keywords` 为 LLM 拓展后的完整矩阵）：

```json
{
  "industry": "家装-旧房改造",
  "keywords": ["旧房改造", "老房翻新", "二手房装修", "老破小改造", "拆改"],
  "filters": { "publishedWithinDays": 90, "minComments": 50 },
  "targetVideoCount": 50,
  "comments": { "scrollRounds": 30, "fetchReplies": true }
}
```

> `output` 字段可省略；即使传入，采集脚本也一律 dry-run，只落盘 JSON。

## 一键采集 bash（Agent 生成，用户只粘贴运行）

收到用户回填的诉求（或界面复制来的 config JSON）后，**Agent 直接产出一条完整 bash 命令**交给用户在本地终端运行。命令必须做到：

1. **用 heredoc 内联写 config**（严禁用 `curl`/下载链接写 config —— 下载链接可能返回 HTML 页面，导致 `run-collect.js` 报 `Unexpected token '<'`）。
2. **自动探测 Chrome 路径写 `.env`**（`.env` 只写 `CHROME_PATH` 一项，不含任何飞书凭证）。
3. `npm install`（首次）。
4. `node run-collect.js --config config/industry.json` 采集并落盘。

macOS 模板（替换 `PROJECT_DIR` 与 config 内容后交给用户）：

```bash
cd "$HOME/Desktop/douyin-comment-collector" || exit 1

# 1) 写采集配置（heredoc 内联，不走下载链接）
mkdir -p config && cat > config/industry.json << 'EOF'
{
  "industry": "家装-旧房改造",
  "keywords": ["旧房改造","老房翻新","二手房装修","老破小改造","拆改"],
  "filters": { "publishedWithinDays": 90, "minComments": 50 },
  "targetVideoCount": 20,
  "comments": { "scrollRounds": 30, "fetchReplies": true }
}
EOF
node -e "require('./config/industry.json') && console.log('✅ config 合法')"

# 2) 自动探测 Chrome 写 .env（只有 CHROME_PATH）
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
[ -x "$CHROME" ] || CHROME="$(command -v google-chrome-stable || command -v google-chrome || true)"
printf 'CHROME_PATH=%s\n' "$CHROME" > .env
echo "✅ .env: $(cat .env)"

# 3) 装依赖并采集（脚本只落盘 data/*.json，不碰飞书）
[ -d node_modules ] || npm install
node run-collect.js --config config/industry.json

# 4) 自动定位刚落盘的 JSON —— 打印绝对路径 / 大小 / 评论条数，并在访达高亮
LATEST="$(ls -t data/comments_*.json 2>/dev/null | head -1)"
if [ -n "$LATEST" ]; then
  ABS="$(cd "$(dirname "$LATEST")" && pwd)/$(basename "$LATEST")"
  echo "──────────────────────────────────────────"
  echo "✅ 采集结果 JSON 已生成："
  echo "   路径: $ABS"
  echo "   大小: $(du -h "$ABS" | cut -f1)"
  echo "   评论条数: $(node -e "const d=require('$ABS');let n=0;const c=Array.isArray(d)?d:(d.comments||d.records||[]);n=Array.isArray(c)?c.length:0;console.log(n)" 2>/dev/null || echo '见文件')"
  echo "──────────────────────────────────────────"
  echo "👉 把上面这个文件拖进 Agent 对话上传即可（macOS 已在访达高亮）"
  command -v open >/dev/null && open -R "$ABS"   # 在访达中高亮该文件
else
  echo "⚠️ 未在 data/ 下找到 comments_*.json，请检查采集是否成功"
fi
```

- Linux：Chrome 路径改探测 `google-chrome` / `chromium`；`open -R` 换成 `xdg-open "$(dirname "$ABS")"` 打开所在目录。
- 采集完成后，第 4 步会**自动打印 JSON 绝对路径 + 大小 + 评论条数，并在访达里高亮该文件**，用户无需手动去 `data/` 里翻找。**让用户把该文件拖进 Agent 对话上传（或粘贴 JSON）**，由 Agent 完成飞书写入。

### 随时定位「最新一次采集的 JSON」（独立小命令）

采集早已跑完、想再次找到那个 JSON 时，在项目目录里粘这一段即可（打印路径并在访达高亮）：

```bash
cd "$HOME/Desktop/douyin-comment-collector" || exit 1
LATEST="$(ls -t data/comments_*.json 2>/dev/null | head -1)"
if [ -n "$LATEST" ]; then
  ABS="$(cd "$(dirname "$LATEST")" && pwd)/$(basename "$LATEST")"
  echo "最新采集文件: $ABS  ($(du -h "$ABS" | cut -f1))"
  command -v open >/dev/null && open -R "$ABS"      # macOS：访达高亮
  # Linux 用: xdg-open "$(dirname "$ABS")"
else
  echo "data/ 下暂无 comments_*.json"
fi
```

> 想看全部历史采集文件（按时间倒序、带大小）：`ls -lt data/comments_*.json`。

## 输出端：飞书多维表格（由 Agent 后台代做）

采集脚本不写飞书。写入由 **Agent 通过 `lark-base` 市场技能、以当前登录用户身份**完成：

1. Agent 用 `lark-base +base-create` 新建一个 Base（名称由 Agent 内部命名，如「{行业} 抖音评论采集」），再用 `+table-create` 建表 + 建字段。
2. 用 `+record-create` / 批量写入把 `data/comments_xxx.json` 拍平后的每条评论写入。
3. **表名、base token、table id 默认不向用户暴露**；仅在用户主动索要时返回可访问链接。

**建表字段（列）**：

| 字段 | 类型 | 字段 | 类型 |
| --- | --- | --- | --- |
| 视频ID | 文本 | 评论内容 | 文本 |
| 视频标题 | 文本 | 评论层级 | 文本 |
| 视频作者 | 文本 | 评论用户 | 文本 |
| 视频链接 | 文本(url) | 点赞数 | 数字 |
| 视频评论总数 | 数字 | 回复数 | 数字 |
| 评论ID | 文本 | IP归属 | 文本 |
| 评论时间 | 文本 |  |  |

> 数字字段用 `{"type":"number","name":"...","style":{"type":"plain","precision":0}}`；链接字段用 `{"type":"text","name":"视频链接","style":{"type":"url"}}`。

## 分析与方案输出（由 Agent 后台生成瑞士极简风 HTML 报告）

写完飞书多维表格后，Agent **对采集到的评论做大模型分析，并产出一份 HTML 分析报告**。报告不是评论罗列，而是「洞察 → 经营动作」的完整闭环：先总结用户核心痛点，再给出上品 / 人工客服 / 短视频 / H5 落地页 / 投放五大经营建议，每条建议都**对应真实可录入的字段并附理由**。

### 铁律 0 · 始终围绕「用户给定赛道」分析（最高优先级）

- 报告分析的赛道 = **用户在诉求界面填写的行业 / 主题名称与关键词矩阵**（即 `config.industry` + `config.keywords`），这是**硬性锚点，绝不改写、绝不重新解读**。
- 评论里冒出来的高热子话题**只能作为该赛道下的子议题**，**不能顶替整个赛道**。示例：赛道是「旧房改造」，评论里「加装电梯 / 低层分摊」很热——它只是旧房改造下的一个子痛点，报告主线仍是「旧房改造」，不要把整份报告改写成「加装电梯分析」。
- 所有痛点、上品、客服、视频、H5、投放建议**都必须落回用户给定赛道**，不要跑题到评论里的旁支话题。

### 报告视觉风格（强制：瑞士极简风）

- **报告 HTML 必须用「瑞士极简风 / 国际主义平面风格 (Swiss / International Typographic Style)」**，通过 `huashu-design` 技能的设计规范实现。要点：
  - 严格网格、无衬线字体（Helvetica Neue / Neue Haas Grotesk / Inter / Geist），大量留白，左对齐。
  - 黑白灰 + **单一信号色瑞士红 `#E2231A`**（红仅用于强调、序号、关键数据高亮，不滥用）。
  - 直角（`border-radius:0`）、发丝线分隔、大号 billboard 标题、数字用 `tabular-nums`；label / kicker / 数字可用等宽字体（Geist Mono）。
  - 禁装饰性 emoji、禁圆角卡片+左彩条、禁渐变堆砌（反 AI slop）。
- 建议 CSS token（可直接复用）：`--ink:#0A0A0A; --ink-3:#767676; --paper:#FFFFFF; --paper-2:#F4F4F2; --red:#E2231A; --hair:#DCDCDA;`；字体 `--sans:"Helvetica Neue","Neue Haas Grotesk",Inter,Geist,...; --mono:"Geist Mono","SF Mono",...`。
- ⚠️ **只有报告用瑞士极简风**；前置的「诉求收集界面」仍必须用 life-ds 风格（主题色 `#387AFF`），两者不要混淆。

#### 交付前自检清单（强制：报告上传前必须逐条打勾，任一不符即作废重做）

> Agent 在调用 `mcp__runtime__upload_file` 交付报告**之前**，必须先在心里逐条核对以下 7 项。只要有一项为「否」，禁止交付，必须先改到全部为「是」。

1. [ ] 背景是**白 / 浅灰**（`#FFFFFF` / `#F4F4F2`），不是深色背景？
2. [ ] 信号色是**且仅是瑞士红 `#E2231A`**，没有出现抖音蓝 `#387AFF`、抖音红 `#FE2C55` 或其它主题色？
3. [ ] 字体是**无衬线**（Helvetica Neue / Inter / Geist），label / 数字用等宽 (Geist Mono)？
4. [ ] 所有卡片 / 色块 / 进度条都是**直角**（`border-radius:0`），没有圆角？
5. [ ] **没有**任何装饰性 emoji、圆角卡片 + 左彩条、渐变堆砌？
6. [ ] 数字用了 `tabular-nums`，有大号 billboard 标题与发丝线分隔？
7. [ ] 这是**报告**、用的是瑞士极简风（而非误把诉求界面的 life-ds 蓝套上来）？

> 复盘教训：曾出现过把「诉求界面的 life-ds 深色红风」误套到报告上的事故——第 2、7 项就是为拦截这类混用而设，务必认真核对。

### 报告结构（顺序固定）

1. **顶部 3 张指标卡**：① 评论量（本次采集总条数）② 核心赛道（= 用户给定赛道，如「家装-旧房改造」）③ 用户核心痛点总结（一句话概括最高热痛点）。
2. **可点击的飞书多维表格链接**：报告 hero 区与页脚各嵌一个指向本次评论 Base 的可点击链接，方便用户下钻原始评论。
3. **体系化打法模块**：一张流程图 + 说明，串起五大动作闭环——`视频 / 投放引流 → H5 落地页承接 → 客服留资 → 数据回流再优化选题与投放`。
4. **§1 评论分析（痛点洞察）**：给出赛道下的高热痛点热力排序（话题 + 提及量 + 占比），并挑选高赞评论原声作为佐证（引用原文 + 点赞数 + IP 归属分布）。
5. **§2–§6 五大经营建议**（每节 = 一张「配置卡」，逐字段给方案 + 理由，字段来源见下）。

### 五大经营建议模块（每条都对应真实录入字段 + 理由）

> 通用要求：每个模块先给「逐字段建议表」（字段 / 建议内容 / 理由——理由必须回指第 1 节的评论数据，如某痛点提及量、某高赞评论），再给一段「整体理由」。字段清单以**用户提供的对应截图为准**（上品 / 客服配置 / 投放三张截图；H5 参考落地页图），本节列出的是典型字段骨架。

- **§2 上品建议**：依据评论痛点，给本地生活商家的**上品（商品）录入建议**。按上品截图字段逐项给方案：商品标题、商品品类 / 类目、价格 / 套餐、商品卖点 / 服务说明、适用场景、主图 / 详情要点等——每项说明为什么这么填（对应哪个痛点 / 高赞诉求）。
- **§3 人工客服配置建议**：按客服配置截图字段逐项给话术方案：欢迎语、挽留语、常见问题自动回复、留资引导话术、转人工规则等——每条话术都要能接住评论里反复出现的顾虑。
- **§4 短视频脚本 + 分镜建议**：**结合短视频专业编导知识**给出脚本与分镜（钩子 / 冲突 / 解决方案 / 行动号召的叙事结构，分镜含画面、景别、时长、台词 / 字幕、BGM 提示），选题直接取自高热痛点与高赞评论，并说明为什么这样拍能戳中用户。
- **§5 H5 落地页建议**：H5 落地页是**视频与投放流量的共同承接页**。逐模块给建议：**头图 Banner、优势信息、案例信息、表单填写、常见问答 FAQ、转化组件（私信 / 电话 / 填写表单三键，标注主 / 次 / 强优先级与话术）**——每块内容与顺序都对应评论里的关键顾虑（信息顺序即说服顺序），并说明理由。
- **§6 投放建议**：按投放截图字段逐项给方案：定向人群（年龄 / 性别 / 地域——地域可回指评论 IP 归属分布）、兴趣 / 行为标签、投放目标、创意方向、预算 / 出价建议、落地页 / 表单回流等——每项说明依据。

### 报告落盘与交付

1. Agent 把报告写成单文件 `report.html`（图片 / 链接内嵌，双击即可打开）落到会话工作目录。
2. 通过 `mcp__runtime__upload_file` 上传，返回可访问 URL 交付用户。
3. 报告里的飞书多维表格链接用**真实可访问 URL**；表名 / token 仍默认不额外暴露，只暴露这一个可点击链接。

## 本地 .env（只有 Chrome 路径）

采集脚本本地只需一项配置：

```
CHROME_PATH=      # 本机 Chrome 可执行文件路径（由「一键采集 bash」自动探测写入）
```

- **不再需要任何 `FEISHU_*` 凭证**：飞书写入由 Agent 后台代做，本地不接触飞书。
- `run-collect.js` 始终 dry-run（只落盘 `data/*.json`）。

## 首次启动自检 & 优化流程（新接入 Agent 首次运行时执行一次）

> 目的：不同 Agent 依附的运行环境能力不同（有的能读写用户本地文件、有的能直连飞书、有的只能纯对话）。本节让**每个新接入的 Agent 在首次进入采集流程时**先探一次自身能力，逐项与用户确认哪些「B 类人工点」可以由 Agent 代做 / 简化，并把结果记录到项目根目录 `OPTIMIZATION.md`。**A 类护栏永远保留人工**，本流程只减少用户的手动操作量，绝不降低结果可靠性。

### 触发时机

- 某个 Agent + 某个运行环境**首次**进入本 skill 的采集流程时，触发一次自检。
- 若项目根目录 `OPTIMIZATION.md` 已存在「本 Agent + 本环境」的记录，则跳过自检，直接按记录里的执行路径运行。
- 一个 Agent + 环境一条记录，**追加不覆盖**；换 Agent 或换环境需重新自检并新增一条记录。

### 优化原则（硬约束）

- 自检只减少**用户的手动操作量**，**绝不降低结果可靠性**。
- 任何需要「绕过抖音登录态 / 逆向签名 / 臆造评论或分析内容」的所谓「自动化」**一律不算优化，禁止启用**。
- 能力**不确定时按「不可用」处理**，宁可保留人工，禁止假设 Agent 具备某能力。

### 能力探测（按此清单探，不确定=按不可用）

| 能力 | 探测什么 | 影响的优化项 |
| --- | --- | --- |
| 本地命令执行 | Agent 能否在用户本机跑命令 / 无则只能产出命令交用户手动粘贴 | B1 / B2 / B3 |
| 本地文件读写 | Agent 能否直接读写用户本机文件（建目录、读产物 JSON） | B1 / B4 / B7 |
| 飞书多维表格写入 + 授权 | Agent 能否以当前登录用户身份调 `lark-base` 建表写数 | B5 |
| 外部模型 API | Agent 能否调大模型做关键词 LLM 拓展与评论分析 / 报告生成 | 诉求界面关键词拓展、B6 |

> 本 skill **无 VLM 读图 / 抽帧 / 转写步骤**，故**不探 VLM 能力**，也不产生 videos.json / frames 类中间产物——这点与「视频分镜分析」类 skill 不同，自检清单按本 skill 实际链路裁剪。

### 人工点清单（A 类固定保留人工，仅对 B 类逐项确认）

**A 类 · 不可自动化护栏（永远保留人工，最多做「体验优化」，绝不「启用自动化」）**

| 编号 | 人工点 | 为什么不可去除 |
| --- | --- | --- |
| A1 | 业务诉求确认（6 项输入 + 赛道 / 关键词种子） | 采集赛道是分析的硬锚点，必须人来定，机器臆造即跑题 |
| A2 | 抖音扫码登录 · 滑块 · 验证码 · 视频间反风控长停顿 | 无登录态触发 `2483`；绕过 = 逆向签名，违反铁律；长停顿是防风控刚需 |
| A3 | 飞书身份授权（以当前登录用户身份写表） | 涉及用户身份凭证，不能代持、不能绕过 |

**B 类 · 能力足够的 Agent 可代做 / 简化（自检逐项与用户确认）**

| 编号 | 人工点 | 现状 | 可优化方向（依赖的能力） |
| --- | --- | --- | --- |
| B1 | 建专门项目文件夹 + 放入脚本 + `ls` 自检结构 | 用户手动建目录、摆放文件 | 有本地文件读写：Agent 生成一键建目录 / 校验命令（本地命令执行） |
| B2 | 本地环境准备（Node、Chrome 探测、`npm install`） | 一键 bash 已自动探 Chrome | 扩展为自动校验 Node 版本、依赖缺失时给精确安装命令（本地命令执行） |
| B3 | 执行「一键采集 bash」 | 用户手动粘贴运行 | 需本地真机，无法代跑；但可把 config 内联、参数回填做得更省心 |
| B4 | 产物 JSON 回传方式 | 用户手动拖文件 / 粘贴 | 已有自动定位 + 访达高亮；有本地文件读写可直接读取（本地文件读写） |
| B5 | 飞书建表 + 批量写入 | Agent 后台代做 | 已自动；自检确认授权可用即标「启用」（飞书写入 + 授权） |
| B6 | 关键词 LLM 拓展 + 评论分析 + 瑞士极简报告生成 | Agent 后台代做 | 已自动；自检确认模型 API 可用即标「启用」（外部模型 API） |
| B7 | 采集失败项 / 漏采视频补跑 | 目前无显式机制 | 有能力的 Agent 读产物 JSON 找缺口、生成补采 config（本地文件读写） |

### 落盘：写入项目根目录 `OPTIMIZATION.md`

- 位置固定：用户本地项目文件夹根目录 `douyin-comment-collector/OPTIMIZATION.md`。
- 模板见 `references/optimization-template.md`，按其结构逐项填写：基本信息 → 能力探测结果 → 优化项确认结果（A1–A3 固定「保留人工」+ B1–B7 取「启用 / 保留人工 / 不适用」）→ 生效的执行路径调整 → 仍需用户参与的节点 → 备注。
- 每次自检**追加一条记录**（含 Agent 标识、运行环境、自检时间），不覆盖历史记录。

## 本地操作手把手引导（硬规则，不得直接甩命令）

> 采集跑在用户本地、且涉及建文件夹 / 装依赖 / 扫码登录 / 反风控长跑等多步操作，用户极易在"我现在该干嘛"上卡住。因此进入采集流程时，**Agent 必须按下面固定顺序手把手逐步引导，每一步只让用户做一件事，给出可直接复制的命令，并说明这一步的产物和下一步衔接**；检测到某步没做/缺环境，就先把这步补好再往下，绝不一次性把所有命令甩给用户让其自行拼装。

1. **第 0 步 · 先建专门项目文件夹并放入项目文件**：先按上文「本地项目文件夹与目录结构」硬规则，引导用户在本地建 `douyin-comment-collector` 文件夹，把本 skill 的项目文件（`run-collect.js`、`server.js`、`package.json`、`src/`、`public/`）完整放进去。明确告诉用户「后续所有操作都在这个文件夹里做」。用 `ls` 自检结构齐全后再进下一步。
2. **第 1 步 · 填诉求（life-ds 界面）**：引导用户在项目目录 `node server.js` 后浏览器打开 `http://localhost:3100`（或直接打开 `public/index.html`），填完 6 项 → 点「一键复制json」→ 回对话把 config JSON 粘给 Agent。明确这一步只为拿到 config，不采集。
3. **第 2 步 · 粘贴运行「一键采集 bash」**：Agent 收到 config 后按「一键采集 bash（Agent 生成）」节产出**一条**完整命令交用户在项目目录终端粘贴运行。命令内部自动完成写 `config/industry.json`（heredoc 内联）+ 探测 Chrome 写 `.env` + `npm install` + `node run-collect.js` 采集落盘。明确提示：**首次运行会弹出前台 Chrome，需扫码登录抖音**（登录态存 `chrome-profile/`）；**遇滑块/验证码在前台窗口人工滑动即可，脚本会等待**；**视频间有 8–15s 随机长停顿属正常反风控，勿中断**。
4. **第 3 步 · 定位并回传产物 JSON**：采集完成后，「一键采集 bash」第 4 步会自动打印 `data/comments_xxx.json` 的绝对路径 / 大小 / 评论条数，并在访达（macOS）高亮该文件。引导用户**把该文件拖进 Agent 对话上传**（优于粘贴长 JSON）。若采集早已跑完要重新找文件，给用户「随时定位最新一次采集的 JSON」那段独立小命令。
5. **第 4 步 · 交给 Agent 后台续跑（用户无需再操作）**：Agent 一旦收到 `comments_xxx.json`，**自动**用当前用户身份写入飞书多维表格 → 大模型分析评论 → 生成瑞士极简风 HTML 报告，无需用户再下指令。表名/token 默认不暴露，仅在用户主动索要时给可点击链接。

> 引导语气要"手把手"：每一步开头点明「现在做这一步」，只给这一步要跑的命令，跑完告诉用户产物是什么、下一步做什么；不要预先把第 1~4 步的命令一股脑贴出来。

## 运行步骤

```bash
# 0) （可选）打开 life-ds 风格诉求界面填写诉求
node server.js            # 浏览器访问 http://localhost:3100
                          # 填完点「一键复制json」→ 回对话粘贴给 Agent

# 1) 粘贴运行 Agent 给你的「一键采集 bash」（自动写 config + .env + 装依赖 + 采集）
#    脚本只落盘 data/comments_xxx.json，不碰飞书

# 2) 把 data/comments_xxx.json 回传给 Agent
#    → Agent 后台写入飞书多维表格 → 大模型分析评论 → 生成瑞士极简风 HTML 报告（含上品/客服/视频/H5/投放五大建议）
```

## 关键约束（务必遵守）

- **必须先建专门项目文件夹并核对目录结构**：进入采集流程第一件事，是引导用户在本地建一个专门的 `douyin-comment-collector` 文件夹、把项目文件完整放入，`ls` 见到 `run-collect.js`/`server.js`/`config/`/`src/`/`public/` 才继续（结构见「本地项目文件夹与目录结构」硬规则）；绝不允许脚本/config/data 散落在不同目录。
- **本地操作必须手把手逐步引导，不得直接甩命令**：按「本地操作手把手引导」硬规则的固定 5 步（建文件夹 → 填诉求 → 一键采集 bash → 定位回传 JSON → 交 Agent 后台续跑）逐步引导，每步只让用户做一件事，说明产物与下一步衔接。
- **诉求收集必须先出 life-ds 风格 HTML 界面**，界面只收集 6 项输入，其余用默认；主题色必须为 life-ds 品牌蓝 `#387AFF`（禁用青色 / 臆造色值）；界面必须带「一键复制json」主按钮（复制 config JSON 到剪贴板 + toast 提示）；关键词必须经 LLM 拓展后再进入采集，而不是只搜用户原始输入。
- **采集脚本永不请求飞书**：本地只落盘 JSON；飞书写入一律由 Agent 后台用当前用户身份代做；表名/token 默认不向用户暴露。
- **config 必须用 heredoc 内联写入，不要用下载链接**（下载链接可能返回 HTML，导致 `run-collect.js` 报 `Unexpected token '<'`）。
- **综合搜索必须登录态运行**，否则触发 `status_code 2483`；首次运行会弹出前台浏览器窗口，需扫码登录（登录态持久化在项目内 `chrome-profile/`）。
- **单页面连续采大量视频会触发风控**（每视频仅 5 条）。已通过「独立标签页隔离 + 容器 scrollTop + 鼠标滚轮双触发 + 视频间随机长停顿 (8–15s)」解决，勿改回单页连采。
- 遇验证码时前台窗口人工滑动即可，脚本会等待。
- **敏感 / 隐私文件严禁提交公开仓库**：`.env`、`chrome-profile/`、`data/*.json`（含真实用户评论）。
- **分析报告必须围绕用户给定赛道**（`config.industry` + `config.keywords`），绝不改写、绝不用评论里的高热子话题顶替整个赛道；评论子话题只作赛道下的子议题。
- **报告用瑞士极简风、诉求界面用 life-ds 风格**，两套视觉不要混用；报告须含 3 张指标卡（评论量 / 核心赛道 / 核心痛点）、体系化打法闭环图、可点击飞书多维表格链接、以及上品 / 客服 / 视频 / H5 落地页 / 投放五大建议（每条对应真实录入字段 + 理由）。

## 关键文件

- 入口：`run-collect.js`（采集 → 落盘 data/*.json，永不碰飞书）、`server.js`（Web 界面服务）
- 采集内核：`src/search.js`、`src/filter.js`、`src/comments.js`、`src/scraper-core.js`
- 输出：`src/report.js`（HTML 报告）；飞书写入由 Agent 走 `lark-base` 技能后台完成；评论大模型分析 + 瑞士极简风 HTML 分析报告（上品/客服/视频/H5/投放五大建议）由 Agent 后台生成，报告风格套用 `huashu-design` 瑞士极简风、诉求界面套用 `life-design-system`
- 界面：`public/index.html`、`public/app.js`（含一键复制json）、`public/styles/`、`public/assets/`（life-ds 风格，主题色 #387AFF）
- 配置：`config/industry.json`、`.env`（只含 `CHROME_PATH`）
- 自检：`references/optimization-template.md`（OPTIMIZATION.md 模板）；首次自检结果落盘到项目根目录 `OPTIMIZATION.md`（一个 Agent + 环境一条记录，追加不覆盖）
