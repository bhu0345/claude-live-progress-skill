---
name: live-progress
description: 长任务开始时使用（预计超过约 5 分钟或 5 个以上步骤）：发布实时进度页 artifact，显示步骤、进度条、已用时和预计完成时间，并在每步前后更新。用户要进度条、ETA 或想看进度时也用。
---

# Live Progress：长任务实时进度页

长任务开始时，发布一个用户随时可以点开的页面，实时显示：现在在做什么、每一步的状态、分段进度条、已用时、预计剩余时间和预计完成时刻。页面每秒自己刷新计时和 ETA；Claude 只需要在检查点写入一次进度。

## 什么时候用

- 预计超过约 5 分钟，或有 5 个以上明显步骤的任务：调研、批量处理文件、多章节文档、写代码加测试等
- 用户要求"进度条""显示进度""还要多久""ETA"
- 几次工具调用就能做完的小任务不要用。每个检查点要 1 到 2 次工具调用，对小任务来说不划算

需要 `Artifact` 和 `ArtifactData` 两个工具。`ArtifactData` 是延迟加载的，先用 ToolSearch 加载（`select:ArtifactData`）。没有这两个工具的环境（例如本地终端里的 Claude Code），改用任务列表（TaskCreate / TaskUpdate）展示步骤，不发布页面。

## 原理

页面不写数据，只订阅 artifact 数据库里的一个文档：`run/state`。Claude 在检查点用 `ArtifactData` 往这个文档写合并更新，打开页面的人会立刻看到变化。计时、进度百分比和 ETA 都由页面根据时间戳计算。

## 步骤

### 1. 规划步骤和预估

把任务拆成 3 到 12 个步骤。每一步给一个 `est`：预计的实际分钟数，包括工具调用、等待和写作时间。可以参考下面的起点，再按任务调整：

| 工作 | 起点 |
|---|---|
| 一次网页搜索并读几页 | 1–2 分钟 |
| 读取或检查一批文件 | 1–3 分钟 |
| 写约 1000 字 | 2–4 分钟 |
| 写代码、运行、修一轮错 | 3–8 分钟 |
| 生成一个文件并检查 | 3–6 分钟 |
| 派子 agent 做一项调研 | 5–15 分钟 |

宁可略微高估。估错也没关系：页面会用已完成步骤的实际耗时自动校准（页面上叫"节奏"）。

### 2. 读取当前时间

运行 `date +%s`，得到 Unix 秒。之后每次写进度都要带时间戳。一分钟内的读数可以直接复用；也可以把 `; date +%s` 附在本来就要跑的 Bash 命令后面，省掉一次调用。

### 3. 生成并发布页面

页面模板在本文件末尾的 html 代码块里，已经处理好主题、手机宽度和数据订阅，不需要修改。用下面这条命令把它提取成文件并填上标题。`<SKILL_DIR>` 换成本 skill 加载时显示的目录（Base directory，通常是 `/mnt/skills/user/live-progress`）；`<页面名>` 用 2 到 4 个词的任务名，例如"季度报告进度"：

```bash
python3 -c 'import sys;f=chr(96)*3;t=open(sys.argv[1],encoding="utf-8").read();sys.stdout.write(t.rsplit(f+"html\n",1)[1].split(f,1)[0].replace("__TITLE__",sys.argv[2]))' "<SKILL_DIR>/SKILL.md" "<页面名>" > progress-<slug>.html
```

找不到 SKILL.md 时，把模板原样写成文件，只把 `__TITLE__` 换成页面名。

每个任务用新的文件名（同一路径会覆盖上一个任务的页面），放在工作目录或 scratchpad 目录。然后发布：

- `file_path`：刚生成的文件
- `icon`：`"progress"`
- `description`：一句话说明这个任务在做什么
- `capabilities`：`{"db": {}}`（必须有，否则页面读不到数据）

记下结果里的 artifact URL，后面每次 `ArtifactData` 调用都用它。

### 4. 写入初始状态

用 `ArtifactData` 的 `set`，`collection` 为 `"run"`，`doc_id` 为 `"state"`：

```json
{
  "title": "整理 2025 年销售数据并生成季度报告",
  "lang": "zh",
  "status": "running",
  "startedAt": 1790300000,
  "updatedAt": 1790300000,
  "activity": "正在读取 12 个 CSV 文件",
  "steps": {
    "s01": {"n": 1, "title": "读取并检查数据文件", "est": 3, "status": "running", "startedAt": 1790300000, "done": 0, "total": 12, "unit": "个文件"},
    "s02": {"n": 2, "title": "清洗数据", "est": 8, "status": "pending"},
    "s03": {"n": 3, "title": "计算季度指标", "est": 5, "status": "pending"},
    "s04": {"n": 4, "title": "写报告正文", "est": 10, "status": "pending"}
  },
  "log": {"1790300000": "开始任务"}
}
```

- `steps` 是对象（键为 s01、s02 等），不要写成数组。这样之后可以只更新某一步的几个字段。
- `n` 决定显示顺序。中途插入步骤可以用小数，例如 2.5。
- `lang` 跟用户使用的语言一致，取 `"zh"` 或 `"en"`。

`set` 成功就说明页面已经能读到数据，不需要再检查。告诉用户一句话：进度页已发布，点开卡片就能实时看进度。然后开始干活。

### 5. 在检查点更新

每个检查点只调用一次 `ArtifactData` 的 `update`，只写有变化的字段。嵌套对象会递归合并，没写到的步骤和字段保持不变。把上一次写入结果里的 version 作为 `if_version` 传入；如果因为版本不符失败，先 `get` 一次再重写。

检查点：

- 一步结束、下一步开始：合并成一次写入
- 一步里要处理很多同类项目（文件、网页、章节）：给这一步设 `total`，每完成约 10%，或每过 3 到 5 分钟，更新一次 `done`。不要每处理一项就写一次
- 计划有变：新增步骤用新的键；放弃的步骤改成 `"status": "skipped"`，并在 `note` 里写原因
- 需要问用户、等回复时：`"status": "waiting"`；收到回复继续时改回 `"running"`
- 遇到问题要换方法时：记一条 `"level": "warn"` 的 log

示例：第 1 步完成，第 2 步开始。

```json
{
  "updatedAt": 1790300240,
  "activity": "正在清洗数据：合并重复客户记录",
  "steps": {
    "s01": {"status": "done", "endedAt": 1790300240, "done": 12, "note": "2 个文件是 GBK 编码，已转换"},
    "s02": {"status": "running", "startedAt": 1790300240}
  },
  "log": {"1790300240": "数据读取完成，开始清洗"}
}
```

写法要求：

- `activity` 用一句用户看得懂的话说明现在在做什么，不写工具名或内部术语。
- `log` 的键是时间戳字符串，值是一句话，或 `{"text": "...", "level": "warn"}`（`level` 可选 `warn`、`error`）。只记关键节点：开始、完成一步、发现问题、改计划。一个任务一般不超过 30 条。
- 控制开销：写入次数大约是步骤数加上长步骤里少量的子进度更新。不要为每个小动作写一次进度。

### 6. 收尾

任务完成时写最后一次更新，即使用户已经在对话里看到结果也要写。否则页面会一直显示"进行中"，几分钟后还会提示"没有新进度"。

```json
{
  "status": "done",
  "endedAt": 1790302100,
  "updatedAt": 1790302100,
  "activity": "",
  "summary": "报告已生成：共 18 页，含 6 张图表。Q3 营收环比增长 12%。",
  "steps": {"s04": {"status": "done", "endedAt": 1790302100}},
  "log": {"1790302100": "任务完成"}
}
```

失败或中止时写 `"status": "failed"`，在 `activity` 里写清楚卡在哪里、需要用户做什么，正在进行的步骤改成 `"status": "failed"`。

## 字段参考

`run/state` 文档：

| 字段 | 含义 |
|---|---|
| `title` | 任务名，显示为页面大标题 |
| `lang` | `"zh"` 或 `"en"`，决定界面文字的语言 |
| `status` | `running`、`waiting`（等用户回复）、`done`、`failed` |
| `startedAt` / `updatedAt` / `endedAt` | Unix 秒（也接受毫秒或 ISO 时间字符串） |
| `activity` | 一句话：现在在做什么 |
| `summary` | 完成后的结果摘要，显示在进度条下方 |
| `steps.<键>` | 一个步骤，字段见下表 |
| `log.<时间戳>` | 一条动态，字符串或 `{"text", "level"}` |

每个步骤：

| 字段 | 含义 |
|---|---|
| `n` | 显示顺序 |
| `title` | 步骤名 |
| `est` | 预估分钟数 |
| `status` | `pending`、`running`、`done`、`failed`、`skipped` |
| `startedAt` / `endedAt` | Unix 秒 |
| `done` / `total` / `unit` | 子进度，例如 7 / 12 个文件；有了它 ETA 会按实际速度推算 |
| `note` | 补充说明，显示在步骤名下面 |

## 页面怎么算 ETA

了解算法，才能写出好用的 `est` 和子进度：

- 节奏 k =（已完成步骤的实际耗时 + 进行中步骤的实测证据 + 先验）÷（对应的预估 + 先验），限制在 0.3 到 5 之间。先验约等于一个平均步骤，所以前一两步的偏差不会让 ETA 大起大落。
- 预计剩余 = 进行中步骤的剩余时间 + 所有待办步骤的 `est` × k。
- 进行中的步骤有 `done` / `total` 时，按已处理项目的实际速度推算；没有时按 `est` × k 推算。已经超时的步骤，剩余时间按已用时间的 25% 计。
- 分段进度条每一段的宽度按 `est` 分配；完成度也按 `est` 加权。
- 进行中的任务超过 max(6 分钟, 当前步骤预计耗时 × 1.5) 没有收到更新，页面会提示"没有新进度"。

所以 `est` 要写真实的分钟数，长循环一定要给 `total` 并定期更新 `done`。

## 页面模板

```html
<title>__TITLE__</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500&family=IBM+Plex+Sans:wght@400;500;600&display=swap">
<style>
:root{
  --ground:#F3F5F8;--surface:#FFFFFF;--ink:#172033;--muted:#5E6A7F;--line:#DADFE8;--track:#E5E9F0;
  --accent:#2E56D9;--accent-soft:#E2E8FB;
  --ok:#1D8757;--ok-soft:#DCF0E5;--warn:#9E5E00;--warn-soft:#F7EBD3;--bad:#C0392F;--bad-soft:#F8E0DD;
  --sans:"IBM Plex Sans","PingFang SC","Hiragino Sans GB","Noto Sans SC","Microsoft YaHei",system-ui,sans-serif;
  --mono:"IBM Plex Mono",ui-monospace,"SFMono-Regular",Menlo,Consolas,monospace;
}
@media (prefers-color-scheme:dark){
  :root:not([data-theme="light"]){color-scheme:dark;
    --ground:#0E131C;--surface:#151C28;--ink:#E3E8F1;--muted:#8D98AC;--line:#263042;--track:#212A3A;
    --accent:#7F9DFF;--accent-soft:#1E2A4A;--ok:#3DBA83;--ok-soft:#15302A;--warn:#E3A640;--warn-soft:#352A16;--bad:#F07169;--bad-soft:#3A1E1E}
}
:root[data-theme="dark"]{color-scheme:dark;
  --ground:#0E131C;--surface:#151C28;--ink:#E3E8F1;--muted:#8D98AC;--line:#263042;--track:#212A3A;
  --accent:#7F9DFF;--accent-soft:#1E2A4A;--ok:#3DBA83;--ok-soft:#15302A;--warn:#E3A640;--warn-soft:#352A16;--bad:#F07169;--bad-soft:#3A1E1E}
*{box-sizing:border-box}
body{margin:0;background:var(--ground);color:var(--ink);font:15px/1.55 var(--sans);-webkit-font-smoothing:antialiased}
.wrap{max-width:760px;margin-inline:auto;padding-inline:16px;padding-block:28px 56px;display:flex;flex-direction:column;gap:24px}
.meta-row{display:flex;flex-wrap:wrap;align-items:center;gap:6px 12px;font-size:13px;color:var(--muted)}
.chip{display:inline-flex;align-items:center;gap:6px;padding:3px 10px 3px 8px;border-radius:999px;font-size:12.5px;font-weight:600;letter-spacing:.02em;background:var(--track);color:var(--muted)}
.chip::before{content:"";width:7px;height:7px;border-radius:50%;background:currentColor}
.chip.running{background:var(--accent-soft);color:var(--accent)}
.chip.running::before{animation:pulse 1.6s ease-in-out infinite}
.chip.waiting{background:var(--warn-soft);color:var(--warn)}
.chip.done{background:var(--ok-soft);color:var(--ok)}
.chip.failed{background:var(--bad-soft);color:var(--bad)}
.heartbeat.stale{color:var(--warn);font-weight:500}
h1{margin:10px 0 0;font-size:clamp(22px,4.4vw,28px);line-height:1.25;font-weight:600;letter-spacing:-.005em;text-wrap:balance}
.activity{margin:6px 0 0;color:var(--muted);font-size:15px}
.activity:empty{display:none}
.panel{background:var(--surface);border:1px solid var(--line);border-radius:10px;padding:20px;display:grid;grid-template-columns:minmax(0,1fr) auto;gap:18px 32px;align-items:end}
.label{font-size:12px;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);font-weight:500}
.eta-value{display:flex;flex-wrap:wrap;align-items:baseline;gap:2px 10px;margin-top:4px;line-height:1.1}
.eta-value .n{font-family:var(--mono);font-size:clamp(32px,7vw,44px);font-weight:500;letter-spacing:-.03em;font-variant-numeric:tabular-nums}
.eta-value .u{font-size:17px;color:var(--muted);margin-left:4px}
.eta-value .word{font-size:clamp(24px,5vw,30px);font-weight:600}
.eta-sub{margin-top:8px;color:var(--muted);font-size:14px}
.stats{display:grid;grid-template-columns:repeat(2,auto);gap:12px 28px;margin:0}
.stats dt{font-size:12px;color:var(--muted)}
.stats dd{margin:2px 0 0;font-family:var(--mono);font-size:16px;font-weight:500;font-variant-numeric:tabular-nums}
.bar{grid-column:1/-1;display:flex;gap:3px;height:12px}
.seg{flex:var(--w) 1 0;min-width:6px;background:var(--track);border-radius:3px;overflow:hidden;position:relative}
.seg>i{position:absolute;inset:0 auto 0 0;width:var(--f);background:var(--accent)}
.seg.done>i{background:var(--ok)}
.seg.failed>i{background:var(--bad)}
.seg.running>i{background-image:repeating-linear-gradient(-45deg,transparent 0 5px,rgba(255,255,255,.25) 5px 10px);background-size:14.14px 14.14px;animation:slide 1s linear infinite}
.bar-note{grid-column:1/-1;margin:-8px 0 0;font-size:12px;color:var(--muted)}
h2{margin:0 0 10px;font-size:12px;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);font-weight:600}
.steps{list-style:none;margin:0;padding:0;background:var(--surface);border:1px solid var(--line);border-radius:10px}
.step{display:grid;grid-template-columns:18px minmax(0,1fr) auto;gap:4px 14px;padding:14px 16px}
.step+.step{border-top:1px solid var(--line)}
.glyph{width:18px;height:18px;border-radius:50%;border:2px solid var(--line);margin-top:2px;position:relative}
.step.running .glyph{border-color:var(--accent)}
.step.running .glyph::after{content:"";position:absolute;inset:3px;border-radius:50%;background:var(--accent);animation:pulse 1.6s ease-in-out infinite}
.step.done .glyph{border-color:var(--ok);background:var(--ok)}
.step.done .glyph::after{content:"";position:absolute;left:4px;top:1px;width:4px;height:8px;border:solid var(--surface);border-width:0 2px 2px 0;transform:rotate(45deg)}
.step.failed .glyph{border-color:var(--bad);background:var(--bad)}
.step.failed .glyph::after{content:"";position:absolute;left:6px;top:2px;width:2px;height:6px;background:var(--surface);box-shadow:0 8px 0 var(--surface);border-radius:1px}
.step.skipped .glyph{border-style:dashed}
.step-title{font-weight:500;overflow-wrap:anywhere}
.step.pending .step-title,.step.skipped .step-title{color:var(--muted)}
.step.skipped .step-title{text-decoration:line-through}
.step-note{font-size:13.5px;color:var(--muted);margin-top:2px;overflow-wrap:anywhere}
.step-sub{display:flex;align-items:center;gap:10px;margin-top:8px;font:12.5px var(--mono);color:var(--muted);font-variant-numeric:tabular-nums}
.mini{flex:1;max-width:240px;height:5px;background:var(--track);border-radius:3px;overflow:hidden}
.mini>i{display:block;height:100%;background:var(--accent)}
.step.done .mini>i{background:var(--ok)}
.step-time{text-align:right;font-family:var(--mono);font-size:13.5px;font-variant-numeric:tabular-nums;white-space:nowrap}
.t-main{display:block}
.t-main.over{color:var(--warn)}
.t-est{display:block;font-size:12px;color:var(--muted)}
.empty{padding:18px 16px;color:var(--muted);font-size:14px}
.summary{background:var(--surface);border:1px solid var(--line);border-radius:10px;padding:16px 18px;white-space:pre-wrap;overflow-wrap:anywhere}
.log{list-style:none;margin:0;padding:0;display:flex;flex-direction:column;gap:8px}
.log li{display:grid;grid-template-columns:48px minmax(0,1fr);gap:12px;font-size:14px}
.log time{font:12.5px var(--mono);color:var(--muted);padding-top:2px;font-variant-numeric:tabular-nums}
.log .txt{overflow-wrap:anywhere}
.log li.warn .txt{color:var(--warn)}
.log li.error .txt{color:var(--bad)}
.more{background:none;border:0;padding:0;margin-top:10px;color:var(--accent);font:inherit;font-size:13.5px;cursor:pointer}
.more:focus-visible{outline:2px solid var(--accent);outline-offset:3px;border-radius:2px}
.foot{margin:0;padding-top:14px;border-top:1px solid var(--line);font-size:12.5px;color:var(--muted)}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.35}}
@keyframes slide{to{background-position:14.14px 0}}
@media (max-width:560px){.panel{grid-template-columns:minmax(0,1fr)}.stats{grid-template-columns:repeat(2,minmax(0,1fr))}}
@media (prefers-reduced-motion:reduce){*,*::before,*::after{animation:none!important}}
</style>

<main class="wrap">
  <header>
    <div class="meta-row"><span id="chip" class="chip"></span><span id="heartbeat" class="heartbeat"></span></div>
    <h1 id="title"></h1>
    <p id="activity" class="activity"></p>
  </header>

  <section class="panel" aria-live="polite">
    <div>
      <div class="label" id="etaLabel"></div>
      <div class="eta-value" id="etaValue"></div>
      <div class="eta-sub" id="etaSub"></div>
    </div>
    <dl class="stats">
      <div><dt id="lElapsed"></dt><dd id="elapsed">—</dd></div>
      <div><dt id="lPct"></dt><dd id="pct">—</dd></div>
      <div><dt id="lPace"></dt><dd id="pace">—</dd></div>
      <div><dt id="lConf"></dt><dd id="conf">—</dd></div>
    </dl>
    <div class="bar" id="bar" role="progressbar" aria-valuemin="0" aria-valuemax="100"></div>
    <p class="bar-note" id="barNote"></p>
  </section>

  <section id="summarySec" hidden>
    <h2 id="lResult"></h2>
    <div class="summary" id="summary"></div>
  </section>

  <section>
    <h2 id="lSteps"></h2>
    <ol class="steps" id="steps"></ol>
  </section>

  <section id="logSec" hidden>
    <h2 id="lLog"></h2>
    <ul class="log" id="log"></ul>
    <button type="button" class="more" id="logMore" hidden></button>
  </section>

  <p class="foot" id="foot"></p>
</main>

<script>
(function(){
  "use strict";
  var I18N = {
    zh: {
      title:"任务进度", connecting:"正在连接……", wait:"等待数据", running:"进行中", waiting:"等你回复", done:"已完成", failed:"已中断",
      remaining:"预计剩余", paused:"已暂停", total:"总用时", stopped:"状态",
      finishAt:function(t){return "预计 "+t+" 完成";}, finishedAt:function(t){return "完成于 "+t;},
      resume:function(r){return "回复后继续，之后约需 "+r;}, soon:"即将完成",
      elapsed:"已用时", progress:"完成度", pace:"节奏", conf:"估算可信度",
      confRough:"粗略", confFair:"一般", confGood:"较准", paceNone:"暂无",
      paceTip:"已完成步骤的实际耗时 ÷ 预估耗时，大于 1 表示比预想的慢",
      steps:"步骤", log:"动态", result:"结果",
      showAll:function(n){return "显示全部 "+n+" 条";}, showLess:"收起",
      est:function(d){return "预估 "+d;}, skipped:"已跳过",
      justNow:"刚刚", minAgo:function(n){return n+" 分钟前";}, hrAgo:function(n){return n+" 小时前";},
      updated:function(a){return a+"更新";},
      stale:function(n){return "已 "+n+" 分钟没有新进度 · 可能在执行较长的一步，也可能任务已停止";},
      empty:"Claude 写入第一条进度后，这里会自动更新。",
      noRuntime:"这个页面要在 Claude 里打开，才能读取实时进度。",
      error:"实时连接中断，正在重试……",
      barNote:"每一段代表一个步骤，宽度按预估耗时分配",
      foot:"预计剩余 = 剩余步骤的预估耗时 × 节奏；有子进度（如 3/12）时按实际速度推算。计时由页面每秒刷新，进度由 Claude 在每步前后写入。",
      min:"分钟", minShort:"分", hr:"小时", lt1:"<1", locale:"zh-CN"
    },
    en: {
      title:"Task progress", connecting:"Connecting…", wait:"Waiting for data", running:"Running", waiting:"Needs your reply", done:"Done", failed:"Stopped",
      remaining:"Time left", paused:"Paused", total:"Total time", stopped:"Status",
      finishAt:function(t){return "Expected to finish at "+t;}, finishedAt:function(t){return "Finished at "+t;},
      resume:function(r){return "Resumes after your reply, then about "+r;}, soon:"Almost done",
      elapsed:"Elapsed", progress:"Progress", pace:"Pace", conf:"Estimate quality",
      confRough:"Rough", confFair:"Fair", confGood:"Good", paceNone:"n/a",
      paceTip:"Actual time of finished steps ÷ their estimates; above 1 means slower than planned",
      steps:"Steps", log:"Activity", result:"Result",
      showAll:function(n){return "Show all "+n;}, showLess:"Show less",
      est:function(d){return "est. "+d;}, skipped:"Skipped",
      justNow:"just now", minAgo:function(n){return n+" min ago";}, hrAgo:function(n){return n+" h ago";},
      updated:function(a){return "Updated "+a;},
      stale:function(n){return "No update for "+n+" min · a long step may be running, or the task stopped";},
      empty:"This page updates as soon as Claude writes the first progress entry.",
      noRuntime:"Open this page in Claude to see live progress.",
      error:"Live connection lost, retrying…",
      barNote:"Each segment is one step, sized by its estimated time",
      foot:"Time left = remaining estimates × pace; steps with item counts (e.g. 3/12) use their measured rate. Timers tick every second; Claude writes progress before and after each step.",
      min:"min", minShort:"min", hr:"h", lt1:"<1", locale:"en-US"
    }
  };

  var state = null, conn = "connecting", showAllLog = false, L = I18N.zh, lang = "zh";
  var $ = function(id){ return document.getElementById(id); };

  function esc(s){ return String(s == null ? "" : s).replace(/[&<>"']/g, function(c){ return {"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"}[c]; }); }
  function num(v, d){ var n = Number(v); return isFinite(n) ? n : d; }
  function ts(v){
    if (v == null || v === "") return null;
    var n = Number(v);
    if (isFinite(n) && n > 0) return n < 1e12 ? n * 1000 : n;
    var d = Date.parse(v); return isNaN(d) ? null : d;
  }
  function pickLang(st){
    if (st && (st.lang === "zh" || st.lang === "en")) return st.lang;
    return /^zh/i.test(navigator.language || "") ? "zh" : "en";
  }

  function durParts(min, short){
    if (min < 1) return [[L.lt1, short ? L.minShort : L.min]];
    var t = Math.round(min);
    if (t < 60) return [[String(t), short ? L.minShort : L.min]];
    var h = Math.floor(t / 60), m = t % 60;
    return m ? [[String(h), L.hr], [String(m), L.minShort]] : [[String(h), L.hr]];
  }
  function durText(min, short){ return durParts(min, short).map(function(p){ return p[0] + " " + p[1]; }).join(" "); }
  function durHTML(min){ return durParts(min).map(function(p){ return '<span><span class="n">' + esc(p[0]) + '</span><span class="u">' + esc(p[1]) + '</span></span>'; }).join(""); }
  function clock(ms){
    var d = new Date(ms), now = new Date();
    var hm = d.toLocaleTimeString(L.locale, {hour:"2-digit", minute:"2-digit", hour12:false});
    if (d.toDateString() === now.toDateString()) return hm;
    return d.toLocaleDateString(L.locale, {month:"numeric", day:"numeric"}) + " " + hm;
  }
  function watch(ms){
    var s = Math.max(0, Math.floor(ms / 1000)), h = Math.floor(s / 3600), m = Math.floor(s % 3600 / 60), x = s % 60;
    var p = function(n){ return (n < 10 ? "0" : "") + n; };
    return h ? h + ":" + p(m) + ":" + p(x) : m + ":" + p(x);
  }
  function ago(ms){
    var s = Math.max(0, ms / 1000);
    if (s < 45) return L.justNow;
    if (s < 3600) return L.minAgo(Math.max(1, Math.round(s / 60)));
    return L.hrAgo(Math.round(s / 3600));
  }

  function normSteps(st){
    var raw = st.steps || {}, list = [];
    if (Array.isArray(raw)) raw.forEach(function(s, i){ if (s && typeof s === "object") list.push(Object.assign({id: s.id || "s" + i, n: i + 1}, s)); });
    else Object.keys(raw).forEach(function(k){ var s = raw[k]; if (s && typeof s === "object") list.push(Object.assign({id: k}, s)); });
    list.forEach(function(s){
      s.status = ["pending","running","done","failed","skipped"].indexOf(s.status) >= 0 ? s.status : "pending";
      s.estMin = Math.max(num(s.est, 1), 0.1);
      s.t0 = ts(s.startedAt); s.t1 = ts(s.endedAt);
      s.doneN = num(s.done, 0); s.totalN = num(s.total, 0);
    });
    list.sort(function(a, b){ return (num(a.n, 999) - num(b.n, 999)) || String(a.id).localeCompare(String(b.id)); });
    return list;
  }

  // ETA model: Claude's per-step estimates, rescaled by a pace factor learned from finished steps.
  function compute(st, steps, now){
    var active = steps.filter(function(s){ return s.status !== "skipped"; });
    var sumE = 0, eDone = 0, aDone = 0, nDone = 0, progW = 0, remPending = 0;
    active.forEach(function(s){
      sumE += s.estMin;
      if (s.status === "done" || s.status === "failed"){
        progW += s.estMin;
        if (s.status === "done"){
          nDone++; eDone += s.estMin;
          aDone += (s.t0 && s.t1 && s.t1 >= s.t0) ? (s.t1 - s.t0) / 60000 : s.estMin;
        }
      }
    });
    // A running step is evidence too: with item counts it shows its rate, and an overrun shows at least its elapsed time.
    active.forEach(function(s){
      if (s.status !== "running" || !s.t0) return;
      var e = Math.max(0, (now - s.t0) / 60000);
      if (s.totalN > 0 && s.doneN > 0){ aDone += e; eDone += s.estMin * Math.min(s.doneN / s.totalN, 1); }
      else if (e > s.estMin){ aDone += e; eDone += s.estMin; }
    });
    var prior = active.length ? sumE / active.length : 1;
    var k = Math.min(5, Math.max(0.3, (aDone + prior) / (eDone + prior)));
    var runRem = [], maxExpected = 0;
    active.forEach(function(s){
      if (s.status === "running"){
        var e = s.t0 ? Math.max(0, (now - s.t0) / 60000) : 0;
        var expected = s.estMin * k, rem;
        maxExpected = Math.max(maxExpected, expected);
        if (s.totalN > 0 && s.doneN > 0 && s.doneN < s.totalN){
          var w = s.doneN / s.totalN, proj = w * (e * s.totalN / s.doneN) + (1 - w) * Math.max(expected, e);
          rem = Math.max(proj - e, 0);
        } else if (e < expected) rem = expected - e;
        else rem = e * 0.25;
        var frac = s.totalN > 0 ? Math.min(s.doneN / s.totalN, 0.99) : Math.min(e / (e + rem || 1), 0.95);
        s.live = {elapsed: e, over: e > expected * 1.25};
        progW += s.estMin * frac; runRem.push(rem);
      } else if (s.status === "pending") remPending += s.estMin * k;
    });
    var remRun = runRem.length > 1 ? Math.max.apply(null, runRem) : (runRem[0] || 0);
    var pct = sumE ? progW / sumE : 0;
    if (st.status === "done") pct = 1;
    var conf = nDone === 0 ? L.confRough : (nDone <= 2 && pct < 0.4 ? L.confFair : L.confGood);
    return {k: k, nDone: nDone, paced: eDone > 0, remaining: remRun + remPending, pct: Math.min(1, Math.max(0, pct)), conf: conf, maxExpected: maxExpected};
  }

  function setStaticLabels(){
    document.documentElement.lang = lang === "zh" ? "zh-CN" : "en";
    $("lElapsed").textContent = L.elapsed; $("lPct").textContent = L.progress;
    $("lPace").textContent = L.pace; $("lConf").textContent = L.conf;
    $("lSteps").textContent = L.steps; $("lLog").textContent = L.log; $("lResult").textContent = L.result;
    $("barNote").textContent = L.barNote; $("foot").textContent = L.foot;
    $("pace").title = L.paceTip;
  }

  function renderEmpty(){
    var msg = conn === "noruntime" ? L.noRuntime : conn === "error" ? L.error : conn === "live" ? L.empty : L.connecting;
    $("chip").className = "chip"; $("chip").textContent = L.wait;
    $("heartbeat").textContent = ""; $("title").textContent = L.title; $("activity").textContent = msg;
    $("etaLabel").textContent = L.remaining; $("etaValue").innerHTML = '<span class="n">—</span>'; $("etaSub").textContent = "";
    ["elapsed","pct","pace","conf"].forEach(function(id){ $(id).textContent = "—"; });
    $("bar").innerHTML = ""; $("steps").innerHTML = '<li class="empty">' + esc(msg) + '</li>';
    $("summarySec").hidden = true; $("logSec").hidden = true;
  }

  function render(){
    lang = pickLang(state); L = I18N[lang]; setStaticLabels();
    if (!state){ renderEmpty(); return; }
    var now = Date.now(), st = state;
    var status = ["running","waiting","done","failed"].indexOf(st.status) >= 0 ? st.status : "running";
    var steps = normSteps(st), c = compute(st, steps, now);
    var stampList = steps.map(function(s){ return Math.max(s.t0 || 0, s.t1 || 0); });
    var updated = ts(st.updatedAt) || Math.max.apply(null, [0].concat(stampList)) || null;
    var started = ts(st.startedAt) || Math.min.apply(null, steps.map(function(s){ return s.t0 || Infinity; }).concat([Infinity]));
    if (!isFinite(started)) started = null;
    var ended = ts(st.endedAt) || updated;

    $("chip").className = "chip " + status; $("chip").textContent = L[status];
    var hb = $("heartbeat"); hb.className = "heartbeat";
    if (updated){
      var since = (now - updated) / 60000, limit = Math.max(6, c.maxExpected * 1.5);
      if (status === "running" && since > limit){ hb.className = "heartbeat stale"; hb.textContent = L.stale(Math.round(since)); }
      else hb.textContent = L.updated(ago(now - updated));
    } else hb.textContent = "";
    $("title").textContent = st.title || L.title;
    $("activity").textContent = st.activity || "";

    var label = L.remaining, value, sub = "";
    if (status === "done"){
      label = L.total; value = started && ended ? durHTML((ended - started) / 60000) : '<span class="n">—</span>';
      sub = ended ? L.finishedAt(clock(ended)) : "";
    } else if (status === "failed"){
      label = L.stopped; value = '<span class="word">' + esc(L.failed) + '</span>';
      sub = ended ? clock(ended) : "";
    } else if (status === "waiting"){
      label = L.paused; value = '<span class="word">' + esc(L.waiting) + '</span>';
      sub = c.remaining > 0 ? L.resume(durText(c.remaining)) : "";
    } else if (c.remaining < 0.5 && c.pct > 0.9){
      value = '<span class="word">' + esc(L.soon) + '</span>';
    } else {
      value = durHTML(c.remaining); sub = L.finishAt(clock(now + c.remaining * 60000));
    }
    $("etaLabel").textContent = label; $("etaValue").innerHTML = value; $("etaSub").textContent = sub;

    var endRef = (status === "running" || status === "waiting") ? now : ended;
    $("elapsed").textContent = started && endRef ? watch(endRef - started) : "—";
    $("pct").textContent = Math.round(c.pct * 100) + "%";
    $("pace").textContent = c.paced ? "×" + c.k.toFixed(1) : L.paceNone;
    $("conf").textContent = status === "done" ? "—" : c.conf;
    var bar = $("bar"); bar.setAttribute("aria-valuenow", String(Math.round(c.pct * 100)));

    bar.innerHTML = steps.filter(function(s){ return s.status !== "skipped"; }).map(function(s){
      var f = s.status === "done" || s.status === "failed" ? 1 : s.status === "running"
        ? (s.totalN > 0 ? s.doneN / s.totalN : (s.live ? Math.min(s.live.elapsed / Math.max(s.estMin * c.k, 0.1), 0.95) : 0)) : 0;
      return '<div class="seg ' + s.status + '" style="--w:' + s.estMin + ';--f:' + (Math.max(0, Math.min(1, f)) * 100).toFixed(1) + '%" title="' + esc(s.title || "") + '"><i></i></div>';
    }).join("");

    $("steps").innerHTML = steps.length ? steps.map(function(s){
      var main = "", estTxt = L.est(durText(s.estMin, true));
      if (s.status === "done" || s.status === "failed"){
        if (s.t0 && s.t1){ var act = (s.t1 - s.t0) / 60000; main = '<span class="t-main' + (act > s.estMin * 1.25 ? " over" : "") + '">' + esc(durText(act, true)) + '</span>'; }
      } else if (s.status === "running" && s.live){
        main = '<span class="t-main' + (s.live.over ? " over" : "") + '">' + watch(s.live.elapsed * 60000) + '</span>';
      } else if (s.status === "skipped"){ estTxt = L.skipped; }
      var subBar = s.totalN > 0 ? '<div class="step-sub"><div class="mini"><i style="width:' + (Math.min(1, s.doneN / s.totalN) * 100).toFixed(1) + '%"></i></div><span>' + s.doneN + " / " + s.totalN + (s.unit ? " " + esc(s.unit) : "") + '</span></div>' : "";
      return '<li class="step ' + s.status + '"><span class="glyph" aria-hidden="true"></span><div><div class="step-title">' + esc(s.title || s.id) + '</div>' +
        (s.note ? '<div class="step-note">' + esc(s.note) + '</div>' : "") + subBar + '</div><div class="step-time">' + main + '<span class="t-est">' + esc(estTxt) + '</span></div></li>';
    }).join("") : '<li class="empty">' + esc(L.empty) + '</li>';

    $("summarySec").hidden = !st.summary; $("summary").textContent = st.summary || "";

    var raw = st.log && typeof st.log === "object" ? st.log : {}, entries = [];
    Object.keys(raw).forEach(function(key){
      var v = raw[key], o = typeof v === "string" ? {text: v} : (v && typeof v === "object" ? v : null);
      if (!o || !o.text) return;
      entries.push({t: ts(o.t) || ts(key), text: o.text, level: o.level === "warn" || o.level === "error" ? o.level : ""});
    });
    entries.sort(function(a, b){ return (b.t || 0) - (a.t || 0); });
    $("logSec").hidden = !entries.length;
    var shown = showAllLog ? entries : entries.slice(0, 8);
    $("log").innerHTML = shown.map(function(e){
      return '<li class="' + e.level + '"><time>' + (e.t ? esc(clock(e.t).slice(-5)) : "") + '</time><span class="txt">' + esc(e.text) + '</span></li>';
    }).join("");
    var more = $("logMore"); more.hidden = entries.length <= 8;
    more.textContent = showAllLog ? L.showLess : L.showAll(entries.length);
  }

  $("logMore").addEventListener("click", function(){ showAllLog = !showAllLog; render(); });

  function connect(){
    var cl = window.claude;
    if (!cl || typeof cl.use !== "function"){ conn = "noruntime"; render(); return; }
    Promise.resolve(cl.use("db")).catch(function(){ return null; }).then(function(db){
      if (!db){ conn = "noruntime"; render(); return; }
      var subscribe = function(){
        try {
          db.doc("run/state").onSnapshot(function(snap){
            conn = "live"; state = snap.exists ? snap.data() : null; render();
          }, function(err){
            conn = "error"; render();
            if (!err || err.code === "unavailable" || err.code === "resource_exhausted") setTimeout(subscribe, 8000);
          });
        } catch (e){ conn = "error"; render(); }
      };
      subscribe();
    });
  }

  render();
  connect();
  setInterval(function(){ if (state && (state.status === "running" || state.status === "waiting" || !state.status)) render(); }, 1000);
})();
</script>
```
