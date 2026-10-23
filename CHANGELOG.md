# CHANGELOG.md — 隐藏助手 改造记录

> 本文件是 [`AGENTS.md`](./AGENTS.md) 的拆分产物，收录**历次改动的完整记录**（含根因、实现要点、验证结果）。
> 当前状态、关键架构知识、风险待办见 `AGENTS.md`；测试夹具 / 环境 / 事故复盘见 [`TESTING.md`](./TESTING.md)。
>
> 最后更新：2026-10-13

## 一、说明

- 本目录（`hide/`）是**独立的嵌套 git 仓库**（`hide/.git`），本文件与 `TESTING.md`、`AGENTS.md` 同为
  **AI / 开发用的文档**，不属于插件运行时资产（不会被 ST 加载）。
- 各节按**实现时间顺序**排列（标题里的"§2.x"沿用历史编号，便于老引用对照）。
- 状态：**全部为已实施并实测验证**的改动；尚未处理的遗留项见 AGENTS.md §四。
- 提交 / 发布由用户决定；AI 不主动 commit。

## 二、改动记录

### 2.1 修复「隐藏楼层」莫名失效（方案A：请求拦截兜底复活）

**根因（已锁定并复现）**：
- 隐藏楼层依赖全聊天共享的 `is_system` 标志。同装的 **小白盒（LittleWhiteBox）剧情总结模块** 的自动隐藏控制器
  （`LittleWhiteBox/modules/story-summary/hide-state.js` 的 `apply()`/`reconcile()`）会在
  `CHAT_CHANGED` / `MESSAGE_EDITED` / `MESSAGE_SWIPED` 等事件里把**所有**消息的 `is_system` 重写为它的"投影"；
  当其自身隐藏未激活时边界为 -1 → 把隐藏助手的隐藏标志**全部清空**并写盘。插桩调用栈已验证。
- 隐藏助手宣传的"请求拦截兜底"（`globalThis.HideHelper_interceptGeneration`，index.js 文件末尾）
  因 `manifest.json` 缺少 `generate_interceptor` 字段而**从未被调用**（死代码）——所以标志被清后必然泄漏。

**修复（已实施）**：
- `hide/manifest.json` 增加一行：`"generate_interceptor": "HideHelper_interceptGeneration"`。
- 实测：拦截器被调用 `14 → 4`（输入全部消息，输出仅保留最新 N 条），真实发送请求只带 4 条聊天消息；
  与 PromptInspector 无关（该机制是 ST 核心功能，不依赖任何其他扩展）。

**风险（务必牢记）**：隐藏助手自带「检查更新」（`hide/update.js`，从 `github.com/uhhhh15/hide` 拉取覆盖），
**点更新会冲掉这行 manifest 修改**。需向作者提 PR 或更新后手动补回。

### 2.2 主题切换「左下角 → 右上角」水波纹

- 实现：`hide/index.js` 的 `applyThemeWithRipple(newTheme, { afterFinish })`（约 102-213 行），
  `hide/style.css` 的 `::view-transition-*` 规则（约 131-155 行）。
- 要点：
  - 用 **View Transitions API** + **Web Animations**：对 `::view-transition-new(hh-theme-surface)` 做
    `clip-path: circle()` 扩散；圆心用**元素本地坐标的左下角** `(0, height)`（不是视口坐标！），
    半径 `hypot(width, height) + 2`，保证终点正好铺满右上角。
  - 时长 `620ms`，缓动 `cubic-bezier(0.4, 0.2, 0.2, 1)`；
  - 切换期间加 `html.hh-theme-switching` 冻结所有微过渡（CSS 通配 `transition: none !important`），
    并关闭页面根快照的默认动画；`saveSettingsDebounced` 延后到动画结束后执行（消除中段卡顿）；
  - 双入口都接了：`#hide-helper-theme-toggle`（使用说明页）与 `#hide-helper-switch-theme-now`（首次提示弹窗）；
  - 不支持 VT / 系统减少动态效果 → 自动回退瞬时切换；连点会 `skipTransition` 旧动画。
- **历史 bug 已修**：① 方向颠倒（视口坐标误用为元素本地坐标）；② 中段卡顿（ease-out 尾段爬行 + 微过渡叠加）。
- ⚠️ **待用户在真实浏览器目视确认最终效果**（自动化面板无法验证动画，见 TESTING.md §三）。

### 2.3 聊天统计页 UI 调整

- 已删说明文本：「（按实际发送内容折算，未能定位的条目不参与合计）」、`（已匹配 N/M 条 ⓘ）`及其 tooltip 与相关变量。
- 条目行内的「未匹配」灰显标签**保留**（用户未要求删除；如后续要删，改 `renderTokenStatsContent` 内 `renderEntriesList`）。
- 统计渲染完毕后会调用 `centerPopup($('#hide-helper-popup'))` 重新居中（约 1877-1880 行）。
- 已按用户要求**回滚**过三样东西：脏标记、智能刷新、统计工具栏（刷新按钮/时间戳），不要重新引入。

### 2.4 聊天统计「消息锚定模型」（已替代旧的"打开弹窗模拟发送"）

**设计（用户提出）**：统计不再在打开弹窗时"模拟发送"，而是**每次真实发送时**计算并锚定到
触发本次发送的那条消息上，随聊天文件持久化；打开统计页直接读锚定值。

**实现位置**（`hide/index.js`）：
- 常量（约 54-73 行）：`STATS_EXTRA_KEY = 'hideHelperStats'`、`STATS_ANCHOR_KEEP = 2`（§2.7 由 3 改）、
  `ANCHORED_GEN_TYPES = new Set(['normal','continue','swipe','regenerate','impersonate'])`。
- 写入链路（`setupEventListeners` 内，约 2155-2205 行）：
  - `GENERATION_STARTED` 记录 `pendingGenType`（`dryRun` 置 null）；
  - `CHAT_COMPLETION_SETTINGS_READY` **消费一次** `pendingGenType` 后清空；非白名单类型直接跳过
    （这样 `generateRawData('quiet')` 等不发 `GENERATION_STARTED` 的后台请求不会误配）；
    同时把 `lastScanEntries` **快照为 `lockedScanEntries`**（锁定"本次发送"的世界书条目集，见 §2.5）；
  - `GENERATION_ENDED / GENERATION_STOPPED / CHAT_CHANGED` 清空 `pendingGenType`；
  - 同步取锚点 `{ chatId, index: 最后一条, messageRef }`，异步 `computeRenderedStats` 完成后
    经 `writeAnchoredStats(anchor, stats)` 校验（chatId + 消息对象身份）再写入 `message.extra.hideHelperStats`。
- 读取链路：`getLatestAnchoredStats()`（从最新往回找，经 `readAnchoredStatsFromMessage`）→
  `updateTokenStatsUI()` → `renderTokenStatsContent()`；无数据时显示「暂无统计数据：发送一条消息后会自动统计并显示。」。
- 清理：`pruneAnchoredStats()` **按"带锚定的消息条数"**只保留最新 `STATS_ANCHOR_KEEP` 条（现为 2，§2.7；不受 chat 长度奇偶性影响）；
  `stripAnchoredStats()` 同时清理 `swipe_info[*].extra` 里的副本。
- 切换聊天时：清空缓存标记 + 立即用新聊天锚定数据重渲染（统计页开着时即时切换）。

**swipe 处理（本轮新增，重要）**：
- 同一条消息的各个 swipe 版本是"同一个锚点的不同版本"——`swipe` 生成时最后一条消息本身被
  `coreChat.pop()` 排除，因此各版本共享同一份输入上下文、同一份统计。
- 但 ST 的 `syncSwipeToMes`（`script.js:7010`）在切换 swipe 时用 `swipe_info[swipe_id].extra`
  **覆盖** `message.extra`。若只写 `message.extra`，滑到没有该锚定的旧版本时统计就会**变空**。
- 因此 `writeAnchoredStats` 把锚定**镜像写入** `message.extra` 与 `message.swipe_info[*].extra`
  （每个版本各一份）；`readAnchoredStatsFromMessage` 优先读 `message.extra`、回退读当前
  `swipe_info[swipe_id].extra`。实测：滑动生成 swipe1 后两份都带锚定，**滑回 swipe0 统计不再变空**。

**已删除（不要再加回来）**：`forceRefreshTokenStats`、`isFakeGenerating`、`renderedStats`、
`renderedWiDetailedStats`、`hasStatsRefreshedThisSession`，以及 `Generate / stopGeneration / is_send_press` 三个 import。

**该模型解决的问题（实测验证）**：
- ✅ 彻底消除旧机制**副作用**：旧的在打开弹窗时调用 `Generate('normal')` 会吞输入框草稿、把草稿作为用户消息
  插入聊天并写盘、执行草稿里的斜杠命令、触发 `send_if_empty`——现在打开统计页**零副作用**（草稿保留、聊天不变、零生成）。
- ✅ 口径纯净：锚定数据 = **上次实际发送的上下文**（不含发送后才到达的 AI 回复；实测锚定在用户消息上，
  随后 AI 回复到达，锚定不变）。
- ✅ 后台 quiet 生成（如剧情总结）被过滤，不污染统计（实测对照：quiet 不写、normal 写）。
- ✅ 打开秒开（约 1s 直接读，不再等 5s 假发送）；重载聊天后锚定从磁盘读回。
- ✅ 窗口剪枝生效（实测：连发后窗口恒为最新 `STATS_ANCHOR_KEEP` 条带锚定消息，旧的被清；该实测当时 KEEP=3，§2.7 起为 2）。
- ✅ 总量口径准确：实测拦截真实请求体逐条独立复算，`total` 与锚定值**逐条完全一致**
  （3219 / 3366 / 3465 / 3490 …）。

### 2.5 Tokens 统计算法审查与修复（本轮新增）

**结论**：锚定模型方向正确、总量准确；但"世界书/聊天"的拆分口径有 BUG，已修。共 4 处代码修复：

1. **【P0，已修】世界书按"来源"而非 `role` 归因**（`computeRenderedStats`）。
   - 问题：旧逻辑 `isChat = role==='user'||'assistant'` 决定一切。而世界书条目可带 `role`
     注入到指定深度（`openai.js` 的 `getPromptRole`），最终上下文里会出现 **role=assistant 的世界书消息**。
     实测 Eldoria 条目（`constant, position=4, depth=4, role=2(assistant)`）以 assistant 消息出现，
     其 **367 token** 被错算进「聊天」，世界书显示 0、条目行还误报「未匹配」。
   - 修复：对**所有**消息（含 user/assistant）做条目锚点认领（`findEntrySpans(..., slots)` 不再只收非聊天消息），
     再把各消息 token 减去被认领部分。实测同一场景：`聊天 155 / 世界书 367 / 其他 2696`，
     `155+367+2696 = 3218 = 总计`，条目显示 `eldoria 367 matched`。
   - 辅助改动：`extractSpanTexts` → `extractSpanPiecesBySlot`（按"所属消息"返回片段，便于逐槽位扣减）。
2. **【已修】`lastScanEntries` 来源不纯**：任何来源的 `WORLDINFO_SCAN_DONE` 都会覆盖它
   （ST-PT/其他扩展的 dry-run 扫描等）。修复：`SETTINGS_READY` 时刻快照为 `lockedScanEntries`，
   归因只读该快照；切聊天时清空。`getEntriesVisibleTexts` 只遍历 `lockedScanEntries`。
3. **【已修】多模态/工具调用口径对齐 ST**（`countFinalMessageTokens`）：图片 85、视频 263×40、
   音频 32×300（对齐 `openai.js` 的 `Message.addImage/addVideo/addAudio` 兜底常量）；
   工具调用按 JSON 字符串计数（与 `setToolCalls` 一致）。此前图片/视频不纳入会导致"实际发送"偏低。
4. **【已修】切聊天"判废"守卫是死代码**（BUG3）：`clearedAt` 在 `startedAt` 之前取得，`clearedAt > startedAt`
   恒 false。修复：`computeRenderedStats` 开头 `statsClearedAt = 0`；`CHAT_CHANGED` 把它设为晚于本次开始的
   时间戳；统计过程中检查 `statsClearedAt > startedAt` 才能真正丢弃。与 `writeAnchoredStats` 的
   chatId+对象身份校验叠加，两层防护。
5. **【已修】"聊天栏"扣减小世界书时用 `slice(0, claimedChars)` 近似**（本轮新增）。
   - 问题：从聊天消息里扣世界书 token 时，旧实现扣的是"**文本最前面 N 个字符**"（N = 被认领的字符数），
     只当认领区间**位于文本开头**时才等于真正被认领的内容。**认领区间在中间**（大块世界书、消息很长时常见）
     时，扣错文本 → "聊天"偏大、"其他"偏小（总量/世界书不受影响，恒等式仍成立，只是聊天↔其他边界漂）。
   - 修复：改为**逐 piece 精确扣减**——`computeRenderedStats` 里新增 `slotClaimedTokens`（Map<slot, tokens>），
     在条目归因时对每个 `piece` 计数累加到所属槽位（与条目 token 用**同一套 piece 计数**，故 Σ 各槽位认领 ≡ `stats.wi`）；
     "聊天" = `Σ_{isChat} (slot.tokens − slotClaimedTokens[slot])`。不再重新 tokenize、无近似。
   - 验证：`node` 单元测试（抽取真实函数 + 非均匀桩 tokenizer，世界书文本置于消息**中部**）：
     旧法 `聊天=79`（错）、新法 `聊天=100=total−wi`（对），恒等式成立。真实浏览器的空模拟亦通过：
     `总共 32985 = 聊天 3960 + 世界书 21768 + 其他 7257`，且独立 tokenizer 复算 `Σ 消息 token = 32985`。

**口径定义（修复后，三者相加恒等于总计）**：
```
总上下文 = 所有最终消息 token 之和
世界书   = 在最终消息中定位到的已激活条目 token（不限消息角色）
聊天     = 各最终消息 token 减去其中被世界书认领的部分
其他     = 总上下文 − 世界书 − 聊天（预设、角色卡描述等）
```

**分析过程文档**：本轮另产出审查报告（未落盘为文件，结论已并入本节）。

**未修的既有行为（如需再评估）**：`wiDetailedStats` 每条锚点较大（实测 43 条记录 ≈ 2657 字节，
汇总仅 ≈ 96 字节），且因 `syncSwipeToMes` 覆盖语义不能简单移到内存；已按用户决定暂不处理。

### 2.6 两个边界场景的实测确认（本轮新增）

**（1）手动 swipe 会被统计到 —— 确认 ✅**
- 用户真实聊天 `WuWa ...@2026-10-08` 的 `chat[6]` 在会话期间被手动 swipe 出 `swipes=[4497, 7117]`
  （`swipe_id=1`）。检查磁盘：该消息 `extra.hideHelperStats.total = 28947`（chat 1208 / wi 21045 / other 6694），
  且 `swipe_info[1].extra.hideHelperStats` 也带同一份；锚点时间戳 `16:29:49` 正是 swipe 发生的时刻。
- 结论：swipe（生成类型 `swipe`）走 `GENERATION_STARTED('swipe')` → `SETTINGS_READY`，被 `ANCHORED_GEN_TYPES`
  收录并锚定，**token 有被统计到**，并镜像进对应 swipe 版本（§2.4 swipe 处理）。

**（2）发送后立即终止，仍会统计到本次发送 —— 确认 ✅（重要）**
- 原因（时序，见 AGENTS.md §三 3.1）：`CHAT_COMPLETION_SETTINGS_READY` 在 **真正 fetch 之前** 触发，
  而终止（abort）只能打断 fetch/流式返回，打断不了已经发生的事件。
- 实测（一次性聊天，发送后约 **75ms** 即 `stopGeneration()`，AI 回复为空 `len=0`）：
  事件只到 `CHAT_COMPLETION_SETTINGS_READY` 就被中止，但用户消息上**仍有锚定**
  （`chat[12] total=3218`，面板显示 `聊天155 / 世界书367 / 其他2696`）。
- 设计上这是**期望行为**：本次请求的上下文确实已被组装并发出，统计口径就是"实际发送"。
  注意这与"终止时聊天里多出的空 AI 回复"无关——锚定落在**触发发送的那条 user 消息**上，
  不依赖 AI 回复是否产出。
- 推论：只要 `SETTINGS_READY` 已触发（通常发送后数百毫秒内），无论后续成功/失败/中止，锚定都已写入。

### 2.7 聊天统计页：锚点切换 + 多维度对比（本轮新增；同日修订）

**背景**：`STATS_ANCHOR_KEEP` 决定一个聊天里最多保留几条"带锚定"的消息；此前统计页只会
展示**最新一条**（`getLatestAnchoredStats`）。本次把"保留的多条锚定"变成可浏览、可对比的资源。
**修订（按用户要求）**：保留条数由 3 改为 **2**；对比默认以**上一个锚定**为基准且基准列**放最左**；
移除「元信息」表；条目差异按世界书分组、组内按 EJS>动态>常驻 排序、无差异不显示。

**改动（`hide/index.js`，`hide/style.css`）**：
- **`STATS_ANCHOR_KEEP = 2`**（由 3 改）。剪枝逻辑不变，只是窗口变 2 → 对比固定两列。
- **顶部锚定栏 `#tub-anchor-bar`**：明细模式显示 `当前消息 #N · HH:MM:SS`（N = 锚定消息在 chat 数组的实时下标，
  时间取锚定对象 `at`；缺 `at` 显示 `--:--:--`）。右侧有「明细 / 对比」切换。
- **明细视图**：原四宫格 + 饼图 + 已激活条目列表。锚点 > 1 条时栏内出现 **chips**，点击切换查看；
  点**最新** chip = 恢复"自动跟随最新"，点其它 = 固定（显示「已固定」）。
- **对比视图 `#tub-compare-section`**：并排列 = 各锚点。**默认基准 = 上一个锚定**（`list[length-2]`，即较早那条）；
  可选基准（下拉 `#tub-anchor-baseline-select`）；**基准列固定放在最左**，其余按时间升序跟在右边，
  各列标注 `#N` + 时间 + (`基准`)。其余列每格显示相对基准 Δ（`+n (+x%)`）。维度：
  - **总量与世界书构成**：总共 / 聊天 / 世界书 / 其他（含占比与 Δ）、常量 / 动态 / EJS、已激活 / 已匹配条目数；
    `system` 恒 0 的死字段不展示。**标签列左对齐**。
  - **世界书条目差异**：按世界书**分组**（组标题 = 书名，按该书名下差异量降序）；组内按 **EJS > 动态 > 常驻** 排序，
    同类型按 |Δ| 降序。行 = 条目名（按类型着色：`tub-stat-color-ejs/-dynamic/-constant`），列 = 各锚点 tokens；
    **不显示世界书来源括号**（已分组）。**完全无差异的条目及其世界书整组隐藏**；两锚点条目全一致时显示"完全一致"。
  - 差值配色：增多 = 暖色 `--hh-stats-color-ejs`，减少 = 绿色 `--hh-stats-green-dark`。
- **已删除**：对比视图的「元信息」表（角色/生成类型/摘要），以及只服务它的 `describeAnchorRole`、
  `describeAnchorSnippet`。`stats.genType` 的写入仍保留（无害，供将来使用）。
- **新增读取函数** `getAnchoredStatsList()`：正向遍历 chat 收集全部锚定 → 升序 `[{index, stats}]`
  （取代旧的 `getLatestAnchoredStats`）。**只读**，不改口径/剪枝/写入逻辑。
- **新增 UI 状态**（模块级，仅进程内）：`statsPinnedIndex`、`statsViewMode`、`statsBaselineIndex`；
  基准下拉选回默认值（上一锚点）时归 `null`（跟随默认）。**`CHAT_CHANGED` 时三者一并重置**。
- **`renderTokenStatsContent` 签名**改为单一 model 对象 `{index,totalTokens,chatTokens,wiTokens,otherTokens,statsObj}`。
- **popup HTML**：`#token-stats-content` 内 `#tub-anchor-bar` + `#tub-detail-section` + `#tub-compare-section`。

**明确未做（避免回归）**：不动统计口径/锚定写入/剪枝/镜像逻辑（仅改 KEEP 常量）、
**不重新引入**被回滚的脏标记 / 智能刷新 / 统计工具栏、不新增 import、不写入 `extension_settings`。

**验证**：`node --check` 通过；浏览器 E2E（在测试夹具上**注入合成锚定**，因真实发送被「柏宝书」拦截、本机走不到
`SETTINGS_READY`）覆盖过：顶部标签/时间、chip 切换、点最新清除固定、基准切换、剪枝回退、<2 条提示、空状态、
写入→saveChat→reload 往返一致。**修订后**另用"从 `index.js` 抽取真实函数 + 桩 DOM"的 Node 单元测试
（27 断言全过）覆盖：条目差异的分组/排序（EJS>动态>常驻）/无差异隐藏/Δ 数值/类型着色/去括号、
`renderCompareContent` 的基准列在最左且无「元信息」、`getAnchorSummary`/`fmtDelta` 数值。
（KEEP=2、默认基准=上一锚点、标签列左对齐属确定性改动，未再驱动浏览器实机验证。）
夹具与用户设置已还原（见 TESTING.md §五）。

### 2.9 聊天统计页第二轮优化（本轮新增）

**改动（`hide/index.js`，`hide/style.css`）**：
1. **饼图按世界书过滤 + SVG 抗锯齿重写**：`renderPieView` 改为 **SVG 实心扇形**（`<path>` 弧，从 12 点顺时针；
   单一构成用整圆 `<circle>`），抛弃 `conic-gradient`（锯齿根源）。**保持实心饼状，不要做成圆环**（用户明确否决圆环）。
   **扇区上标注百分比**（`<text class="tub-pie-text">`，标在扇区中线、半径 30 处；`<5%` 不标注避免溢出；单一构成时 100% 标于圆心）。
   颜色放在 **inline `style="fill:var(--hh-*)"/stroke:var(...)"`**，❗不能放进 SVG 表现属性 `fill="var(...)"`（浏览器不认 → 不变色）。
   单色扇形用同色 0.6px 描边消除扇区间抗锯齿接缝；`.tub-pie-wedge { shape-rendering: geometricPrecision }`；
   `.tub-pie-text` 用 `fill:var(--hh-color-white)` + `text-anchor:middle` + `dominant-baseline:central`。
   饼图数据由 `computePieForBook` 按当前选中的世界书标签**实时聚合**（点某世界书标签 → 饼图只显示该书；点"所有条目" → 全部）。
2. **标签页 iOS 风格滑动**：新增 `.tub-tab-indicator` 滑动下划线（JS `updateTabIndicator()` 用
   `getBoundingClientRect` 定位，**width/transform 由内联写入，故 CSS 里这两项不能加 `!important`**，
   否则会被样式表覆盖 → 指示条卡在 0）。面板切页动画改用 `hhSlideInLeft/Right` 方向性滑入
   （按 tab 顺序判断方向）。弹窗打开后测量一次 + `requestAnimationFrame` 再校正一次。
3. **明细「已激活条目」工具栏**：`tub-book-filters-row` = 左（世界书标签流）+ 右（`tub-entries-tools`：排序按钮 +
   类型过滤下拉）。排序按钮在 **默认排序**（EJS>动态>常驻、同类按 token 降序）与 **上下文顺序** 之间切换
   （后者按锚定时记录的 `pos` 升序）。类型过滤：所有/EJS/动态/常量。排序与过滤可叠加（模块级状态
   `statsListSortMode`、`statsListTypeFilter`）。
4. **锚定条目新增 `pos`**：`computeRenderedStats` 给每条最终消息槽位加 `index`，条目命中时记录
   `pos = min(消息序号*1e6 + 字符偏移)`，写入 `wiDetailedStats[book][type][].pos`（**向后兼容**，旧锚点无 `pos`
   → 上下文顺序排序时排到末尾）。
   **`pos` 的语义已对照 ST 源码确认（`D:\SillyTavern\public\scripts\openai.js`）**：发给 Chat Completion 的
   `messages` 是**自然发送顺序**——index 0 = 最上方（`main`/`worldInfoBefore`），最后一个 = 最新聊天消息；
   `generate_data.messages` 就是序列化进 fetch body 的**同一个数组**（`openai.js:3138` 附近 emit
   `CHAT_COMPLETION_SETTINGS_READY`）。深度注入 `depth:N` 是"距末尾 N 条"。因此 **`pos` 升序 = 顶部（旧）在前、
   最新（底部）在后**，即"实际 API 上下文顺序"。浏览器实测：`TOP=1000000 / MID=2000000 / BOT=3000000`
   （分别对应 messages 索引 1/2/3）。
5. **对比「世界书条目差异」菜单栏 + 过滤**：`renderCompareContent` 把条目差异数据抽成 `buildEntryDiff()`；
   世界书数量 ≥2 个有差异时，在子标题下渲染 `.tub-cmp-book-bar`（"所有条目/世界书一/世界书二…" + 右侧**类型过滤**，
   **不含排序**）；只有 1 个世界书有差异时**不显示菜单栏**，把类型过滤放到子标题（`.tub-cmp-subheading-row`）最右。
   过滤状态 `statsCmpBookFilter`、`statsCmpTypeFilter`（切换聊天重置）。
6. **`0` 的两种含义要分开显示**（对比条目差异子表）：
   - 该锚点 **未激活** 该条目（条目不在该锚点的 `wiDetailedStats` 里）→ 显示 `未激活`；
   - 该锚点 **已激活但未匹配**（条目在 `wiDetailedStats` 里但 `tokens=0`）→ 显示 `未匹配`。
   依据：`wiDetailedStats` 的条目**本身就来自 `WORLDINFO_SCAN_DONE` 的 activated 集合，即已激活条目**；
   所以"不在其中=未激活"、"在其中但 0=已激活未匹配"。`buildEntryDiff` 保留每个锚点的 `present[]`（是否激活）
   以区分二者；`renderEntryDiffTable` 据此输出 `未激活`/`未匹配`（Δ 仍照常显示）。**不要再合并成"未匹配/未激活"**。

**验证**：`node --check` 通过；Node 单元测试（抽取真实函数 + 桩 DOM）覆盖 SVG 扇形段数/着色、`buildEntryDiff` 分组排序、
类型/世界书过滤、菜单栏显隐规则。**浏览器实机**（夹具 `StarTest - fixture`）：① **用真实事件驱动插件自身**
（`emit GENERATION_STARTED` → `WORLDINFO_SCAN_DONE` → `CHAT_COMPLETION_SETTINGS_READY`，**不发起任何网络请求**）
确认真实 `computeRenderedStats` 产出的 `pos` 顺序 = `TOP/MID/BOT = 1e6/2e6/3e6`（顶部旧、底部新）、锚定总量正确、
零错误；② 内存注入两锚定确认饼图为 **3 个 `<path>` 实心扇形**（非圆环）、`MatchOnly` 行 P 列显示 `未匹配`、
N 列显示 `未激活`、**零 console 错误**。测后 `reloadCurrentChat` 丢弃注入，夹具从会话前备份还原
（`StarTest - fixture.jsonl` → md5 `740d15ff…` / 5915 字节 / 13 行），用户聊天未改动（见 TESTING.md §五）。

### 2.10 聊天统计「模拟发送」（本轮新增；同日二次修订）

**功能**：在「聊天统计」页**发送前预估 / what-if** 一次上下文，统计口径与真实发送完全一致（含 EJS）。
实现方式 = **真的跑一遍 ST 完整组装管线，然后在 `CHAT_COMPLETION_SETTINGS_READY` 里截获最终上下文并
立即 abort 掉真正的请求**——这是唯一能拦住请求的时机（`SETTINGS_READY` 之后紧接着 `fetch`），
且此时 ST-Prompt-Template 等扩展已渲染完毕（见 AGENTS.md §三 3.1）。

**UI（二次修订后的最终形态）**：
- 「模拟发送」按钮位于**锚定栏左侧标签区**（`tub-anchor-label`）——原有「当前消息 #N …」等文字信息
  **已全部移除**，该区域现在只放这个按钮。**不再有常驻输入框**（旧的 `tub-simulate-bar` 工具栏已删除）。
- 点击按钮 → 弹出 ST 原生 `POPUP_TYPE.INPUT` 弹窗（`callGenericPopup`）收集自定义内容（可留空）
  → 按「模拟发送」确认后才真正模拟；按「取消」→ **不触发任何生成**。
  ❗ INPUT 弹窗**确认返回字符串、取消/否定返回 `false`（不是 `null`）**，守卫必须写 `typeof value !== 'string'`。

**锚定与生命周期（核心语义）**：
- 模拟结果作为一条**临时锚定**，用哨兵 id（`SIM_ANCHOR_ID = 'sim'`，真实锚点是消息下标数字）
  **并入统一锚点列表 `getAnchoredStatsList()`（追加在最右 = 最新）**，因此它能像真实锚点一样：
  在 `tub-anchor-chips` 出现（**最右、虚线边框、标签「模拟」**）、被点选（明细）、被选为对比基准、
  出现在对比视图的列/条目差异里。
- 数据只存模块级变量 `simulateAnchor`，**绝不写 `message.extra`、绝不 saveChat**；因此：
  **只有切换聊天 / 刷新网页（初始化）时丢失**；**重新模拟发送只覆盖这一条**，不会新增、不影响真实锚点。
- `pruneAnchoredStats` 只作用于真实消息锚点；模拟锚定常驻直到被覆盖或切聊天清空。
- **对比默认基准**仍是"上一个锚点"（`list[length-2]`，即第 2 新）；当模拟锚定在最右时，
  默认基准 = 倒数第二个锚点（通常是最近的真实锚点），「模拟」列在对比表格中恒为**最右列**。

**副作用治理（务必保留）**：真跑管线会触发核心与其它扩展（LWB）改写 `chat_metadata`
（`tainted`/`lastInContextMessageId`/`LWB_SNAP` 等），且管线中途可能落盘一次。
`runSimulateSend` 因此**前后快照 `chat_metadata`、结束时整体还原并重新 `saveChat`**，同时保留/还原
`#send_textarea` 草稿、复位发送按钮状态、最后 `updateTokenStatsUI()` 复位「模拟中…」按钮文案。
实测：模拟后内存与磁盘**逐字节还原**（夹具 md5 不变），**不产生空回复**（历史长度、最后一条 `send_date`
均不变；`saveReply` 在 `SETTINGS_READY` 之后才发生，被 abort 提前打断）。

**其它边界**：仅 `openai`（Chat Completion）可用（其它 API 走不到可拦截的 `SETTINGS_READY`，会真的发出请求 → 直接拒绝）；
`simulateInProgress` 防重入；`GENERATING` 中拒绝；空内容合法（只统计当前上下文）。

**注入消息必须携带变量（重要，实测踩坑）**：有自定义内容时注入的那条临时用户消息**必须复制"上一条消息的
`variables`/`variables_initialized`"**（ST-Prompt-Template 的 `clonePreviousMessage` 语义：这两个字段是**按
swipe 索引的数组** `[varsObj]` / `[true]`）。否则 MVU/EJS 世界书会**以空变量渲染**，导致含 `getvar` 的条目
内容缩水、总量偏低、与真实发送不一致。实测（用户聊天 WuWa MVU）：修复前的"输入内容"比"留空"少 ~3400 tokens
（正是 EJS 依赖变量渲染的大块变短）；修复后，模拟与**真实发送**逐条组装结果**几乎完全一致**
（16 条消息里 15 条逐字节相同，仅末条 `status_current_variable` 15 个字符差异=表达数计数，可忽略）。
因此**不需要**改用"点 `#send_but` 真实发送后拦截"——那条路要额外回滚 `sendMessageAsUser` 的落盘/DOM/
正则/horae 钩子等副作用（实测极易漏出空消息），而修复变量后模拟已足够忠实。
（留空与输入内容的数值差异本质上来自**真实上下文不同**：世界书按关键词/变量动态激活、渲染不同角色档案。）

**验证**：`node --check` 通过；`StarTest - fixture` 浏览器实测（页面内注入 `fetch` 守卫屏蔽
`/api/backends/chat-completions/generate`，操作前断言 `chatId === 'StarTest - fixture'`）：
弹窗输入→确认→模拟（overview 3215，世界书 367 = Eldoria assistant 条目，证明含 EJS 的真实最终上下文）、
取消不生成、重新模拟仍只有 1 个模拟 chip、chips 顺序 `#4 #8 模拟`（模拟最右且 `margin-left:auto` 对齐）、
切到另一聊天后模拟 chip 消失、对比视图「模拟」为最右列且 Δ 正确、`chat_metadata` 前后一致。
夹具与用户聊天测后均还原（见 TESTING.md §五）。

### 2.8 测试注意事项（已迁出）

> 「测试注意事项」类内容（自动化浏览器会把页面切回上次角色、发送前必须复核 chatId、
> 「柏宝书」拦截导致走不到 `SETTINGS_READY`、注入守卫等）已迁至 [`TESTING.md`](./TESTING.md) §三。

### 2.11 「模拟发送」副作用加固（本轮新增；严格审查后修复）

**背景**：对 §2.10 的「模拟发送」做了源码级副作用审查（读了 ST 核心 `Generate`、`openai.js`、
`ST-Prompt-Template/src`、`JS-Slash-Runner/src`、`LittleWhiteBox/modules/story-summary`）。结论：
原实现只快照/还原了 `chat_metadata`，**若干全局存储与竞态未覆盖**，且**每次无条件 `saveChat()`**。

**修复（均在 `hide/index.js` 的 `runSimulateSend`，并新增 4 个纯工具函数）**：

1. **快照范围补齐（核心）**：除 `chat_metadata` 外，再快照并还原
   - `extension_settings.variables.global`（ST/ST-PT 全局变量；**实测确认** ST-PT 的 `updateTokens`
     会把 `LAST_SEND_TOKENS/CHARS` 写成模拟值并落盘）；
   - `extension_prompts`（ST-PT `flushWIInjections` / LWB 召回写入的注入条目）；
   - 每条消息的 `variables` / `variables_initialized`（`setVariable(withMsg)` / `replaceVariables` 可写真实楼层）。
   工具函数：`safeDeepClone`（structuredClone 退化 JSON）、`restoreObjectFromSnapshot`（删新增键+回填深层）、
   `roughDeepEqual`（判断是否需要落盘）、`snapshotMessageVariables` / `restoreMessageVariables`。
2. **异步写入的延迟补还原**：ST-PT `updateTokens` 用 `setTimeout` + 异步分词，**晚于** `finally` 里的同步还原
   （实测 3216 落在同步还原之后）。故对全局变量再加一次 `setTimeout(800ms)` 补还原，守卫：期间若
   `simulateInProgress` / `document.body.dataset.generating==='true'` / 已切聊天 → 放弃补还原。
3. **落盘守卫（不再无条件 saveChat）**：仅当「聊天未切换（chatId 同）+ 消息数不变 + 确有改动」时才
   `context.saveChat()`；否则跳过（避免误写别的聊天 / 把幽灵楼层落盘）。
4. **精确移除临时消息**：按对象引用 `indexOf` + `splice`（原实现依赖"最后一条且 is_user"，若管线末尾多出
   消息就不 pop）。并给注入消息补 `swipes`/`extra` 字段，降低窗口期被别的扩展误读。
5. **快照移入 `try` + 输入框保护**：`structuredClone` 等移入 `try`（否则不可克隆值会让按钮永久卡"模拟中…"）；
   `finally` 仅在 `textareaEl.value===''`（未被用户新输入）时才恢复草稿。

**验证（本轮）**：
- **Node 单元测试**（从 `index.js` 抽取真实函数 + 桩 context/DOM）：**38 断言全过**，覆盖
  `safeDeepClone`/`restoreObjectFromSnapshot`/`snapshotMessageVariables` 深拷贝与就地改写；以及 `runSimulateSend`
  7 个集成场景：正常还原+按改动落盘、无副作用不落盘、切聊天跳过落盘、长度不一致跳过落盘、用户期间输入不被覆盖、
  无新输入恢复草稿、空内容同样走完整还原。（测试文件为临时产物，事后已删。）
- **浏览器 E2E（夹具 `StarTest - fixture`，fetch 守卫）**：真实执行 `runSimulateSend`（弹窗输入→确认）后，
  **`chat_metadata` / 每条消息 `variables` / `extension_prompts` / `LAST_SEND_TOKENS·CHARS` 全部还原**、
  `chat.length` 恒 12、无幽灵楼层；fetch 守卫显示 `/api/backends/chat-completions/generate` **命中且被 abort**（未真发）；
  磁盘夹具 md5 恒 `740d15ff84b61ae635557268355ed0e0`（未变），用户聊天 md5 恒 `330c9df8…`（未变）。
  「取消」路径：**零 fetch、零改动**。统计结果正确（总共 3217 = 聊天 153 + 世界书 367 + 其他 2697）。
- **事故与还原**：审查期间一度真实执行旧版模拟，把 `LAST_SEND_TOKENS` 由 33007 污染为 3216 并落盘；
  已改回 **33007 / 60726** 并 `saveSettings` 落盘（`settings.json` 已恢复）。

**说明**：`scripts/extensions/third-party/hide/index.js` 顶部 `import` 未新增（`saveSettingsDebounced` 原就在
import 列表中，本轮才真正使用）。

### 2.12 「模拟发送」事件屏蔽：根治幻影落盘与第三方生成后处理（本轮新增；已实施并验证）

**背景（对照实测）**：先做了"手动发送→立即停止→删除"与"模拟发送"的逐项对照，又对修复方案做了最不利条件下的
A/B 实验，确认两个关键事实：

- 模拟发送**比手动发送更干净**：不动聊天记录（临时消息只活约 60ms）、不落盘、不触发 `MESSAGE_SENT` /
  `USER_MESSAGE_RENDERED` / `MESSAGE_DELETED`、不丢草稿；仅"完整跑一遍生成生命周期"这一点两者相同。
- **但 `GENERATION_STOPPED` / `GENERATION_ENDED` 会真实发射**（`stopGeneration()` → `script.js:5618`；
  `hideStopButton()` → `script.js:3536`），第三方无法区分"模拟"与"真实生成结束"。**实测抓到幻影层被落盘**：
  `chat-history-backup` 的 `GENERATION_ENDED` 强制保存（`index.js:2112-2124` → `performBackupConditional`）
  与模拟的清理保存存在竞态，慢路径（模拟中止延迟 ~300ms）**稳定复现**——
  `/api/chats/save` 请求体 `msgCount=13` 且含注入消息原文，**永久写入 IndexedDB 备份快照**（已读出
  `AUDIT_PROBE_7f3a` / `AUDIT_SIM_2` 全文），并挤占每实体 3 条的备份配额。LWB 定时任务亦在
  `GENERATION_ENDED` 写入 `settings.json` 的 `processedMessages` 标记。

**修复（`hide/index.js`，4 处）**：
1. **事件屏蔽（核心）**：新增 `shieldGenerationEndEvents()` / `unshieldGenerationEndEvents()`，在
   `context.generate()` **之前**把 `eventSource.events[GENERATION_STOPPED / GENERATION_ENDED]` 置空，
   `finally` 首行解除。依据：`lib/eventemitter.js` 的 `emit()` 在调用时对监听器表做 `slice()`，置空即可拦住当次事件。
   **只摘这两个事件**；`GENERATION_STARTED` / `CHAT_COMPLETION_SETTINGS_READY` 必须保留（统计依赖）。
   解除时用 `(prior||[]).concat(added)`，**保留屏蔽期间新增的监听器**（追加在原有之后）；重入有保护。
2. **超时兜底**：`SIMULATE_SHIELD_TIMEOUT_MS = 30000`，到期强制解除屏蔽，避免真实生成的"生成后处理"被长期吞掉。
   ❗**只解除屏蔽、不复位 `simulateInProgress`**——卡住的管线稍后走到 `SETTINGS_READY` 时仍须被拦截并 abort，
   绝不能放行成真实请求（代价是按钮保持禁用，属期望的互斥）。
3. **`pendingGenType` 提升为模块级**（原为 `setupEventListeners` 局部 `let`，且 `GENERATION_STOPPED` 分支
   `pendingGenType = null` 在静默创建全局变量——顺带修掉）。屏蔽期间其"清空"监听器不执行，故 `finally` 手动
   `pendingGenType = null`，防止模拟在 `SETTINGS_READY` 之前被拦截器中止时残留 `'normal'`
   （否则后续无来源的后台请求会被错误锚定）。
4. **模拟统计的聊天一致性校验**：新增 `simulateChatId`（发起时记录、`CHAT_CHANGED` 清理），
   统计完成写 `simulateAnchor` 前核对 `getContext().chatId !== simChatIdNow → 丢弃`。

**验证**：
- **Node 单元测试**（从真实 `index.js` 抽取屏蔽实现 + 桩 EventEmitter）：**12/12 通过**——置空/内容等价还原、
  其它事件不受影响、屏蔽期间 emit 不触达第三方、解除后正常触达、重入保护、新增监听器保留且排最后、
  未屏蔽时解除为空操作、定时器建立与清理、源码守卫断言。测试文件：`.zcode/audit/shield.test.mjs`。
- **浏览器 E2E（夹具 `StarTest - fixture`，`fetch` 守卫注入 300ms 延迟制造最不利条件）**：
  - 修复后模拟：**AUDIT 落盘 0 次**、备份库**零变化**、LWB 标记**零新增**、
    `chat_metadata`/全局变量/`extension_prompts` **逐字段一致**、`chat.length` 恒 12、无幽灵层；
    统计 `3216 = 聊天152 + 世界书367 + 其他2697`（与修复前对照实验**逐位一致**，证明屏蔽不影响口径）。
  - 监听器数量前后恒 `STOPPED=19 / ENDED=26`；跑 3 次连续模拟 + 1 次取消，均无漂移。
  - **真实生成不被吞**（关键回归）：手动发送→停止，`STOPPED/ENDED` 各触达第三方 1 次，
    备份照常执行（新增 2 条）、LWB 标记照常写入——证明屏蔽只作用于模拟窗口。
  - **`SETTINGS_READY` 前中止**：临时把 LWB 拦截器改为立即 `abort(true)`；模拟 `genCount=0 / srCount=0`
    （未到 fetch）、监听器复原、消息移除；随后发一次"无来源 `SETTINGS_READY`"探测，**未被锚定**（无残留）。
  - 对照：`GENERATION_STARTED('normal') → SETTINGS_READY` 仍正常锚定（`genType='normal'`）。
- **磁盘**：夹具 md5 恒 `740d15ff84b61ae635557268355ed0e0`；用户聊天 md5 恒 `330c9df8…`；
  测试产生的 LWB 标记已清、测试备份已删、`LAST_SEND_TOKENS/CHARS` 恢复 **33007 / 60726**。

**风险与未覆盖**：① 屏蔽实现直接操作 `eventSource.events[...]`（公共 API 是 `on/removeListener`，但无"临时停用"
语义，只能这样）；若 ST 未来改用别的监听器容器，需同步调整。② 若第三方把"生成后处理"挂在
`SETTINGS_READY` 之后、`STOPPED` 之外的时机（如自己的 `setTimeout`），屏蔽覆盖不到。
③ ST-PT 的异步 `updateTokens` 仍靠既有的 800ms 延迟补还原兜底（§2.11）。

### 2.13 聊天统计页第三轮：上下文楼层行 + 条目详情子窗 + 对比组合选择（本轮新增）

**（1）功能一：`上下文顺序` 下插入消息楼层行**
- 「已激活条目」在 `statsListSortMode === 'context'` 时，不再只是排序，而是**按条目"首次出现"的上下文槽位
  分组**，每组前插入一条"消息楼层行"（`tub-ctx-msg`）：显示 `#楼层号` + 用户/角色 + 名字 + 该消息 token
  + 内容开头摘要（`head`，60 字符）。未匹配条目单独归入末尾"未定位到内容的条目"分组。
- 归因数据来源：`computeRenderedStats` 现在接收 `options = { chatSnapshot, genType, injectedRef }`，
  并把**槽位映射**存入锚点 `stats.slotMap`：
  `[{ i, role, floor, isInjected, floorUser, floorName, tokens, head }]`（floor = 聊天数组下标，null = 非聊天）。
  - `chatSnapshot`：**真实发送时**为 `context.chat.slice(0, anchorIndex + 1)`（不含发送后才产生的 AI 回复）；
    **模拟发送时**为当前聊天快照（含注入的临时消息，`injectedRef` 指向它 → 该槽位标 `isInjected`，
    显示为"模拟内容"而非楼层）。
  - `matchSlotFloors`：在快照里为每个上下文槽位找对应聊天楼层；规则 = 隐藏楼层（`is_system`）不参与、
    `swipe` 时最后一条被 `coreChat.pop()` 排除、文本用"包含 + 结尾分隔"匹配（`slotContainsChatText`，
    短消息只在"末尾且前缀是名字/换行分隔"时才认，防误配）。
  - **向后兼容**：旧锚点没有 `slotMap` → 楼层行退化为"上下文片段 #N"（只显示槽位序号），不报错。
- 新函数：`matchSlotFloors` / `buildSlotMap` / `groupEntriesByContext`（抽成纯函数便于单测）。

**（2）功能二：条目详情子弹窗（点击条目查看定义）**
- 点击「已激活条目」列表或「世界书条目差异」表格中的条目 → 弹出**条目详情子窗** `#hide-helper-entry-popup`：
  条目名称 / 世界书名 / 类型徽章（EJS/常驻/动态）/ 主关键词 + 次关键词（含选择性逻辑标签）/ Position
  （`↑Char/↓Char/↑AN/↓AN/@D ⚙👤🤖/↑EM/↓EM/Outlet`）/ Depth（仅 @D）/ Order / 递归（允许递归 /
  不可递归 / 防止进一步递归，命中时高亮）/ content 全文（转义后用等宽字体 pre-wrap 显示）。
- **定位规则（`positionEntryPopup`）**：
  - 桌面端（`8 + 主宽 + 12 + 主宽 + 8 ≤ 视口宽`）：子窗固定在**主窗正右方**；若主窗偏右导致右侧放不下，
    把"主窗 + 子窗"作为一对**整体左移最小必要位移**，保证并排（关闭子窗时还原主窗原位：`entryPopupShift`）。
  - 窄屏/移动端（放不下并排）：子窗**居中**显示在主窗之上（z-index 10000 > 主窗 9999）；ESC 优先关子窗。
  - **尺寸始终与主窗一致**；且**不能用 `getBoundingClientRect` 当布局坐标**——主窗入场动画 `hhFadeIn`
    的 `scale(0.95)` 会让 rect 偏小 5%，需按 `offsetWidth` 反算未缩放布局框（`layoutBox`）。
- 读取：`loadWorldInfo(book)`（**新增按需 import**，只读缓存/API，无副作用）→ 先按 `uid` 精确匹配、
  回退按条目名匹配。连续点击不同条目**复用同一子窗刷新**（`entryPopupSeq` 序号丢弃过期异步结果）。
- 入口：明细与对比两个列表的条目行都带 `data-book/data-name/data-uid/data-type`（`tub-entry-clickable` /
  `tub-cmp-entry-row`），统一走 `openEntryDetailPopup({book, name, uid, type})`。为此 `wiDetailedStats`
  条目新增 **`uid` 字段**（旧锚点无 → 按名字匹配）。
- 生命周期：切聊天 / 切标签页离开统计页 / 关主弹窗 / ESC 均关闭子窗；主题跟随（applyTheme 目标含子窗）。

**（3）功能三：对比页「对比组合」下拉（替代原"基准"单选）**
- `renderAnchorBar` 的对比模式改为枚举**有序组合**（基准 → 对比）：N 个锚点 N×(N-1) 种（2 个锚点 2 种、
  3 个锚点 6 种）。选择组合 = 直接指定基准与对比两列。
- **默认组合** = 「上一条真实 message（基准）→ 最新真实 message（对比）」——`getDefaultCompareCombo`：
  模拟锚定不算"真实"；只有 1 个真实锚点时，基准=该真实锚点、对比=最新锚点（通常是模拟）。
- 状态：`statsBaselineIndex` + `statsCompareIndex`（**新增**）成对保存；选回默认组合 → 两者归 `null`（自动跟随）；
  任一无效/与另一相同 → 成对回退默认。切聊天重置。
- `renderCompareContent(list, baselineIndex, compareIndex)`：**只显示这两列**（基准列在最左，带「基准」标签；
  对比列带「对比」标签），Δ 均相对基准列；`buildEntryDiff(heads, 0)` 的列空间 = [基准, 对比]。
- ❗ **坑（已修，勿踩）**：`<option value>` 里**不能用 `\u0000` 分隔**——HTML 属性解析会把 NUL 变成 U+FFFD，
  回读时 `split('\u0000')` 得到错误 id → 组合切换静默失效。现用 `'|'` 分隔（`6|8` / `sim|4`）。
- 单测已加守卫：源码断言"option value 用 `|`" + 数字/模拟哨兵的往返解析。

**验证**：`node --check` 通过；Node 单元测试 **57 断言全过**（`.zcode/audit/stats2.test.mjs`，抽取真实函数 +
桩数据：包含匹配边界、隐藏楼层/swipe 排除、slotMap 注入标记、上下文分组、组合枚举/默认组合、`|` 往返、
条目详情 HTML 内容与转义、差异表 data-* 属性）；浏览器实机（真实聊天 `WuWa...` + 夹具注入）：
上下文顺序楼层行与分组合并显示正确（含未匹配组）、`#6/#8` 两锚点对比列与 Δ 正确、
组合下拉切换 `6|8` ↔ `8|6` 后表头/Δ/条目数值全部翻转、切回默认恢复、条目子窗 420×551 与主窗**逐像素等大**、
并排间距 12px、窄屏（420×780）自动居中且尺寸仍一致、ESC 优先关子窗、关闭子窗主窗位置还原、
真实聊天未产生任何写入（对话未变、无新消息）。

**测试事故与处置（本轮）**：
- ① ST 的"上次角色/聊天"恢复机制在页面 reload 后会按**当前角色卡 × 聊天文件名**组合重新解析。
  测试期间再次出现该错位，在 `chats/WuWa Solaris-3 MVU Edition/` 下生成了一个
  "WuWa 角色 + StarTest - fixture 文件名"的空聊天（1 条开场白、无用户数据）→ 已删除。
  **务必坚持：任何 reload 后先核对 `characters[characterId].avatar` 再操作 / 再打开聊天。**
- ② 清理脚本误用 `unlink` 连带删掉了测试夹具 `StarTest - short.jsonl` →
  已按"meta + fixture 前 3 条消息"（4 行）重建（见 TESTING.md §一），并在 TESTING.md §一 记录了新字节数/md5。
  **教训：删除散落文件时只用精确路径 + `rm -f`，不要用带着通配/兄弟命令的复合语句。**
- 用户真实聊天 `WuWa ...@2026-10-08` 全程 **md5 与逐条结构核对一致**（`95fdcf61…`），夹具
  `StarTest - fixture.jsonl` 亦恒为 `740d15ff…`；hide 设置逐字段核对未变。

### 2.14 聊天统计页第四轮：两处 BUG 修复 + 五项功能优化（本轮新增；已实施并验证）

**（1）【BUG 修复】"上下文顺序"下楼层显示误导（用户报 `#14`）**
- 现象：聊天只有 9 条消息（最大下标 8），但上下文顺序视图里出现 `上下文片段 #14`。
- 根因：旧锚点没有 `slotMap`，`ctxSlotHeaderHtml` 的兜底分支把**上下文槽位序号**（本次实际发送里第几条消息，
  含预设/世界书深度注入/Role 注入等，远比聊天楼层多）当作 `#N` 显示，被误读成聊天楼层号。
- 修复：① 明确标为「发送顺序第 N 条消息」（1 起）+ 悬浮说明"不是聊天楼层号"；
  ② 有 `slotMap` 但未映射到楼层时显示「{角色}消息（发送顺序第 N 条）」；
  ③ 新锚点走 `matchSlotFloors` 正常显示 `#楼层`。

**（2）【BUG 修复】条目详情子窗无法滚动**
- 根因：**`showEntryPopup` 把子窗设为 `display:block`**，而子窗内部靠 flex 布局（`.hide-entry-body`
  的 `flex:1 1 auto` + `min-height:0`）约束高度。block 下该属性失效 → 内容区撑到全高
  （实测 8309px），被父级 `overflow:hidden` 裁掉 → 完全滚不动。
- 修复：子窗固定 `display:flex`（不再跟随主窗计算值）。
- 同时按要求加**透明滚动条**（`scrollbar-width:thin` + `::-webkit-scrollbar` 8px，轨道透明、
  滑块半透明；亮色 `--hh-scrollbar-thumb: rgba(0,0,0,.22)`，暗色 `rgba(255,255,255,.28)`）。
  注意：`.hide-entry-body *` 仍保留 `scrollbar-width:none`（后代不显示），只有 body 自身显示。

**（3）【功能】条目详情子窗：命中位置 + 片段高亮**
- `computeRenderedStats` 给每条命中条目记录 `hit = { slot, start, end, fragment }`（fragment 截断 160 字符，
  `HIT_FRAGMENT_MAX`）；未匹配条目记录 `sample`（EJS 渲染后的前 3 行，截断 200 字符，`MATCH_SAMPLE_MAX`）
  + `diag = { visibleChars, anchors, ejsRendered }`。
- 子窗展示「命中位置（锚点 #N / 模拟）」+ `describeHitPosition` 的人话描述
  （聊天楼层号 + 角色 + 名字 + 上下文第几条 + 第 M 字符起） + `<mark>` 高亮片段。
- 从对比视图点开时**逐锚点各列一块**（`buildEntryHitInfos(heads, records)`，`buildEntryDiff` 因此保留
  `records[]` = 各锚点原始条目记录）。
- 旧锚点无 hit/sample → `buildSingleEntryHitInfo` 返回 null（**不能**据此判定"未匹配"，
  这是修掉的一个真实 bug：起初把"无诊断信息"渲染成了"未匹配"），子窗改显示"旧版本数据"说明。

**（4）【功能】差异表按 UID 去重**
- `buildEntryDiff` 的行键由 `书名 + 条目名` 改为 `书名 + UID`（缺 UID 时回退条目名，兼容旧锚点）；
  同一本书里的同名条目现在分开显示。

**（5）【功能】「其他」栏构成拆解（按提示词标识）**
- `computeRenderedStats` 新增可选 `options.identifiers`；有值时把"其他"按 `categorizePromptIdentifier`
  归类（预设 / 角色卡 / 作者注释 / 记忆摘要 / 向量记忆 / 故事串 / 深度注入 / 第三方注入 …），
  存入 `stats.otherBreakdown = [{ label, tokens }]`，并**归一化使合计恰等于 `stats.other`**。
- ❗ **对齐时机是本轮最大的坑（已解决，勿回退）**：先后试过三种方案，只有第三种成立，均已浏览器插桩证实：
  1. SETTINGS_READY 时按位置与 `promptManager` 对表 → **错位**（柏宝书在 PROMPT_READY 里 splice 注入 1 条，
     实测 16 → 17，之后每条都偏一位）；
  2. PROMPT_READY 时按"对象引用"建映射 → **0 命中**：`ChatCompletion.getChat()` 产出的是新建的普通对象，
     且数组在 PROMPT_READY → AFTER_DATA 之间**再次被替换**（实测 `arr id 1→4`，首元素 `2→5`），引用不可用；
  3. ✅ **GENERATE_AFTER_DATA 采样**：此刻 PROMPT_READY 的所有监听器（含第三方 splice）都已执行完，
     而 EJS 渲染尚未开始（ST-PT 对 `main_api==='openai'` 的 AFTER_DATA 处理直接 return），
     内容与 `promptManager` 原文逐条一致 → 用**顺序内容匹配**把 identifier 贴到数组下标上
     （匹配不上 = 第三方插入 → `null`；`LOOKAHEAD=6` 允许跳过被预算截断的 pm 项；
     附 `loose` 前缀匹配兜底 `squash_system_messages` 合并）。
- 到 SETTINGS_READY 时 EJS 只改 content、不改条数，`alignPromptIdentifiers` **按下标取用**即可
  （捕获与对齐都是"消费一次"，过期数据绝不错配到后续 quiet 请求）。
- 对齐阈值不足（`ids` 与 `messages` 条数不一致，或 ST 侧命中 < 50%）→ 返回 `null`，
  UI 显示"未记录构成明细"而非错误数字。`identifier === null` 归为「第三方注入（其他扩展插入）」。
- **实测（用户真实聊天 WuWa）**：其他 2697 = 第三方注入 2665（柏宝书的状态快照）+ 预设 22 + 角色卡 10，
  逐位吻合，确认柏宝书那 4390 字符的注入确实是"其他"的绝对大头。

**（6）【性能】分词结果缓存**
- 新增 `cachedTokenCount(text)`：以**文本本身为键**（等价内容哈希，省掉另算哈希），
  `Map` 缓存 promise（并发去重）、失败不缓存（可重试）、上限 `TOKEN_CACHE_MAX = 3000` 条、
  超出按插入序淘汰最旧。所有统计路径（最终消息、世界书认领片段）统一走它，不再直接调 `getTokenCountAsync`。
- ❗ 注意 `countFinalMessageTokens` 的对外语义不变（仍是"每消息 token 数"），只把底层计数换成缓存版。

**验证**：`node --check` 通过；Node 单元测试 **83 断言全过**（`.zcode/audit/stats3.test.mjs`：
identifier 归类、uid 去重（含同名分开 / 旧锚点回退 / present 区分）、命中信息构造与旧锚点不误报、
命中位置文案、拆解渲染（含 null 提示与"其他为 0 不渲染"）、UID+tokens 元信息位置、
缓存命中/失败重试/空文本、标识捕获与对齐 7 个场景）；既有 `stats2.test.mjs` **57 断言**与
`shield.test.mjs` **12 断言**同步回归通过。
浏览器实机（夹具 `StarTest - fixture`，`fetch` 守卫）：模拟发送产出正确拆解、命中位置与片段高亮、
未匹配诊断（诊断行 + 可见文本样例）、对比行点开显示双锚点命中块、同名不同 UID 分两行、
条目子窗可滚动（445 视口 / 8552 内容）且滚动条可见、UI 全流程 0 console 错误。
夹具与用户聊天 **md5 逐字节不变**（`740d15ff…` / `95fdcf61…`），hide 设置逐字段核对未变。

### 2.15 聊天统计页第五轮：条目详情瘦身 + 上下文顺序按真实插入点（本轮新增；已实施并验证）

**（1）【删除功能】条目详情子窗「命中位置」，前后端一并移除**
- 删除锚点里 `wiDetailedStats[*]` 的 `hit` / `sample` / `diag` 字段（`computeRenderedStats` 不再写入），
  连带删除 `HIT_FRAGMENT_MAX` / `MATCH_SAMPLE_MAX` 常量、`buildMatchSample`、
  `buildEntryHitInfos` / `buildSingleEntryHitInfo` / `describeHitPosition` / `renderEntryHitBlock`，
  以及明细/对比两处点击传参里的 `hits` / `fromAnchor`。`buildEntryDiff` 的行不再携带 `records`。
  子窗 HTML 去掉了「命中位置」区块与 `hide-entry-hit-*` 样式（style.css 同步清理）。
  收益：锚点体积不再因 `hit.fragment`(160) / `sample`(200) 膨胀；旧锚点里的这些字段渲染时被直接忽略。

**（2）【修正】`hide-entry-content-meta` 的 token 改为"条目自身 content 的 token"**
- 原显示 `extra.tokens`（= 本次发送中**文本匹配命中**的 token，与条目大小是两回事）；
  现改为 `getTokenCountAsync(entry.content)`（走插件内的 `cachedTokenCount` 缓存），
  口径与世界书编辑器对条目显示的一致（宏未替换、keys 不计入，依据 `world-info.js:3670-3672`）。
  子窗本就 `await loadWorldInfo` 取到完整 `entry`，故 `buildEntryDetailHtml` 改为 `async`。

**（3）【重做】明细「上下文顺序」：按世界书条目的**真实插入点**归组，不再按"命中槽位"**
- **旧实现的偏差**：分组标题用的是"该条目命中的**最终消息槽位**"（`pos` 的消息序号），
  而槽位序号把预设/角色卡/深度注入/第三方注入全算在内（远多于聊天楼层），且楼层匹配靠文本包含（扩展改写后大面积失效）——
  于是标题的 `#N` 或"发送顺序第 N 条"都不是聊天楼层，也无法表达"插在哪条消息前/后"。
- **新实现**：扫描时在 `lastScanEntries` 里保留每个条目的 `position / depth / role / outletName`
  （`WORLDINFO_SCAN_DONE`），`computeRenderedStats` 用纯函数 `entryContextAnchor(entry, sendable)` 由
  **position/depth** 直接算出真实插入点，写进锚点记录 `record.anchor = { key, kind, label, badge }`：
  - position 0 / 1 → `世界书·前置/后置（位于全部聊天消息之前）`；
  - position 4（@D 深度 d）→ `在聊天 #k 之前插入（角色消息 · 名字）`，其中 `k` = 发送列表第 `n-d` 条消息的聊天下标
    （d=0 → 末尾；d≥n → 顶部）——**以 chat 数组为基准线**，与 ST 的 `populationInjectionPrompts`/`setOpenAIMessages` 语义一致；
  - position 2/3 → 作者注释（深 x）；5/6 → 对话示例；7 → 出口（名字）；
  - `sendable = computeSendableChat(chatSnapshot, genType)`：过滤隐藏楼层（is_system），swipe 时排除最后一条
    （与 ST 的 `coreChat` 对齐）；该函数与 `entryContextAnchor` 取代了旧的 `slotContainsChatText`/`matchSlotFloors`/`buildSlotMap`。
- **渲染**：分组标题行改用 `groupEntriesByAnchor`（按 `anchor.key` 自上而下），
  徽标显示 `前置/后置/#k/末尾/AN↑/AN↓/EM↑/EM↓/Outlet`。排序键同步改为 `anchor.key`。
- **旧锚点兼容**：无 `anchor` 字段时**不再整组当成"未匹配"**，而是退回按旧 `pos` 的槽位分组，
  标题标「旧」+`发送顺序第 N 条消息（旧数据，按命中位置）`；`stats.slotMap` 整条链路删除（不再写、不再读）。
- **附带**：`stats.slotMap` 与 `simulateInjectedRef` 一并移除；`buildSlotMap`/`matchSlotFloors` 删除。

**验证**：`node --check` 通过；抽取式 Node 单测回归全绿——
`stats2.test.mjs`（60 断言：`computeSendableChat` 隐藏楼层/swipe 排除、`entryContextAnchor` 各 position/depth 与"深度越小越靠下"、
`groupEntriesByAnchor` 分组排序/旧锚点回退、`buildEntryDiff` 无 `records`、组合枚举、`buildEntryDetailHtml` 无命中区块且 token=自身内容）
与 `stats3.test.mjs`（74 断言）与 `shield.test.mjs`（12 断言）。
浏览器实机（用户真实聊天 `WuWa…` 只读）：打开统计页（总共 28947 = 聊天 1207 + 世界书 21045 + 其他 6695）、
条目子窗显示 `UID 0 · 500 tokens` 且**无**「命中位置」；上下文顺序视图真实锚点走**旧数据回退**（`旧` 徽标、无误报未匹配）；
内存注入新版 `anchor` 后重渲染，分组标题正确显示`世界书·前置（位于全部聊天消息之前）`/`在聊天 #3 之前插入（用户消息 · User）`/`在所有聊天消息之后（末尾）`。
**事故记录（非本插件引起）**：浏览器打开该聊天时，第三方扩展 **SillyTavern-Horae** 刷新了其内嵌
`horae_meta.timestamp.absolute`（`05:15:33Z`→`07:29:56Z`）并触发一次 `saveChat`，故用户聊天 md5 由 `b4d79e1c…` 变为 `ea1adf22…`；
逐字节 diff 确认**唯一差异**就是该时间戳字段——消息正文、结构、hide 锚点（`total=28947`）全部不变，且锚点里**没有**测试期间内存注入的 `anchor` 字段。
（后续再次打开该聊天时 Horae 又刷新了一次时间戳 → `969be949…`，差异同样只有该字段。）

### 2.16 「其他」构成重做：世界书改按**来源槽位**归因（方案 A，本轮新增；已实施并验证）

**背景（§2.14 遗留的矛盾）**：旧口径里「世界书」靠**文本匹配**（在最终消息里查找条目文本）计，「其他」= 总计 − 世界书 − 聊天 是残差。
凡是"确实是世界书、但文本没匹配上"的内容（深度注入被 EJS 渲染成不同文本、被正则改写、超长文本只匹配到一部分……）
都会从「世界书」掉进「其他」——于是「其他构成」里赫然出现 `深度注入（世界书等）2679` + `世界书包装格式 1018`
（用户实测，`其他=6695`），语义自相矛盾。根因：ST 明明知道每条最终消息的来源（来源槽位），插件却靠猜。

**新口径（方案 A）**：
```
总上下文 = 所有最终消息 token 之和
世界书   = ST 用**来源槽位**确定注入的整块内容（整条计入，**不限消息角色**）：
           · identifier = worldInfoBefore / worldInfoAfter（position 0/1 的世界书）
           · 深度注入槽位：identifier = customDepthWI_* / customWIOutlet_*，
             或最终消息里"来自 position=4(@D) 条目、且被世界书命中覆盖 ≥ 50%"的那条（深度注入在 ST 里被命名成 chatHistory-N，故不能只看标识）
聊天     = 其余聊天消息（role=user/assistant）整条
其他     = 总上下文 − 世界书 − 聊天（预设 / 角色卡 / 作者注释 / 向量 / 第三方注入 …）
```
- **文本匹配不再决定"世界书总计"**，只用于 ① 判定深度注入槽位（覆盖阈值 + 命中条目是否都来自 @D）；
  ② 在被判为世界书的槽位内部，尽量精确地把 token 分摊到各条目——精确匹配不到的余量**按条目体积加权分摊**到该槽位的归属条目组，
  因此"世界书内容漏匹配"**不再掉进「其他」**。归属组：前置槽 → 全部 pos0 条目；后置槽 → 全部 pos1 条目；深度槽 → 同 `(position,depth,role)` 的 @D 条目。
- **细节自洽**：明细 token 四舍五入后，把差额（含"无归属条目的世界书槽位"）补给最大的条目，保证 **Σ 明细 ≡ 世界书栏**（同时三栏仍恒等于总计）。
  无 identifier 对齐数据时，明细仍按来源统计（只是「其他」拆解显示"未记录构成明细"）。

**实现（`hide/index.js`）**：
- 新增纯函数 `classifyPromptSlot({identifier,isChat,tokens,ownClaimed,matchedPositions})` → 槽位来源分类；
  `attributeSlotsBySource({slots,slotKind,entries,entryMatchToken})` → `{ wi, chat, perEntry, otherSlots }`。
- `computeRenderedStats` 重构：先逐条目算"精确命中量→槽位"映射（`findEntrySpans`/`extractSpanPiecesBySlot` 保留，仅用于此），
  再按来源槽位归因；`sendable`（`computeSendableChat`）仍用于 `entryContextAnchor` 的真实插入点。
- 删除旧的 `slotClaimedTokens` 逐槽扣减逻辑与 `wiBookBuckets` 增量累加。

**验证**：
- `node --check` 通过；**Node 单测**：`stats2.test.mjs` **80 断言全过**（新增 `classifyPromptSlot` 11 项：
  worldInfo 前后置 / customDepthWI / customWIOutlet / chatHistory-N 的 @D 识别与 pos0 兜底 / 聊天低覆盖不误判 / 无 identifier 兜底；
  `attributeSlotsBySource` 集成：`世界书=前置200+深度300=500`、聊天=30、其他槽=[预设,第三方]、
  前置余量按体积 100:100 平分（A=130/B=70）、深度余量全给单条目（d4=300）、**Σ 明细 ≡ 世界书**）；
  `stats3.test.mjs` 74 断言、`shield.test.mjs` 12 断言同步回归通过。
- **浏览器实机**（夹具 `StarTest - fixture`，用真实 `eventSource.emit` 驱动插件自身、不发起网络请求）：
  载入 `Eldoria` 世界书（uid0=position4/depth4/role2，uid2=position1），
  `GENERATION_STARTED('normal')` → `WORLDINFO_SCAN_DONE` → `CHAT_COMPLETION_SETTINGS_READY`，
  合成上下文 = `[预设, worldInfoAfter, 聊天user, 聊天assistant, 深度注入(eldoria, role=assistant)]`。
  结果：`总计 609 = 聊天 14 + 世界书 582 + 其他 13`（其他=预设）；
  明细 `Eldoria/eldoria=365`（正是那条 **role=assistant** 的深度注入）与 `Eldoria/glade=217`——
  证明**不再按 role 判定来源**、世界书内容不会被错算进「其他」或「聊天」；`Σ 明细 582 ≡ 世界书 582`。
- **测试事故与还原**：合成事件流被插件的锚定逻辑写进了夹具最后一条消息并落盘（fixture md5 `740d15ff…`→`a1404879…`）；
  已从会话前备份 `backups/chat_startest_20261012-164311.jsonl`（`740d15ff…`）**逐字节还原**夹具，
  并删除本次污染的备份 `chat_startest_20261012-164417.jsonl`；`StarTest - short.jsonl` 恒 `472b8baa…`。
  用户聊天 `WuWa…` 再次被 Horae 刷新时间戳（`ea1adf22…`→`969be949…`），diff 同样只有 `horae_meta` 字段。

---

### 2.17 世界书统计：squash 合并槽（D1）+ 同槽混入（D2′）+ 聊天照抄（D3′）+ 槽位判定留痕（D4）（本轮新增；已实施并验证）

**背景**：`世界书统计方案测试与审查.md` 两轮独立审查给出的修复清单（§九-D1/D2/D3、§十-D1′~D4、§11.9 三维裁定表）。
本轮把其中三条**判定缺陷**一次修掉，并补一条可观测性留痕；旧口径（文本匹配）不动，仍作为对照实现保留在审计装置里。

**修的三件事**：
1. **D1｜squash 合并槽**（`squash_system_messages=true` 时最严重）：ST 会把相邻 system 槽
   （`main + worldInfoBefore + worldInfoAfter + charDescription …`）合并成一条并**沿用首条标识**（如 `main`），
   世界书整块因此藏在一个"标识已知、名字却与来源不符"的槽里 → 旧逻辑按标识直接放行成「其他」。
   现在：**已知标识的非聊天槽，只要命中全部来自世界书位置（0/1/4）且覆盖 ≥ 50%，就判为 `wi-merged`**
   ——按"命中量"计入世界书，其余量按原角色归回「其他」。
2. **D2′｜同 depth+role 混入第三方注入**（ST-PT `[GENERATE:x:BEFORE/AFTER]`、JS-Slash-Runner `injectPrompt`、
   LWB 剧情总结与世界书条目会被 ST 并进**同一条消息**）：判据换成"**结构互补**"——
   该槽所属 `depth+role` 组的世界书条目体量（`win`）**撑不满整槽**（缺口 > 5%/5 token）
   **且**世界书自己已被匹配全（命中量 ≥ 0.9×win）→ 判为 `wi-merged`，只认命中量；
   既避免整槽判成世界书，也避免第三方文本被算成世界书（此前实测：同槽只加 16 token 就能让整槽在 +368/−421 间翻转）。
3. **D3′｜聊天照抄条目正文**：聊天槽（role=user/assistant）原先只要命中覆盖 ≥ 50% 就可能被整条算成世界书
   （实测 E3c 误报 +17）。现在聊天槽阈值抬到 **90%**，且要求命中全来自世界书位置；
   低于阈值一律算聊天——玩家/角色自己写的内容不再被"劫走"。

**新增可观测性（D4）**：锚点里新增 `slotDecisions`：每个**非聊天槽位**一条
`{ i, id, kind, tokens, claimed, win }`，并在 debug 日志打一行汇总
（`【统计】槽位判定：#0 main=wi-merged(13551/12420) | …`）。旧锚点没有该字段，视图按缺失回退。

**实现（`hide/index.js`，统计段）**：
- `classifyPromptSlot` 新增返回类型 **`wi-merged`**（"世界书内容与别的来源混在同一条消息里"）：
  分支 ③（已知标识 + 非聊天）不再要求 `win`（合并槽拿不到该结构信息，而"命中全来自 0/1/4"本身已是强证据）；
  分支 ① 的聊天槽改用 `WI_CHAT_SLOT_COVERAGE = 0.9`。
- 新增纯函数 `depthGroupKeyOf(hitDepthKeys)`：由"命中本槽的 @D 条目"反推该槽所属 `depth+role` 组；
  新增 `isSlotMixed(win, tokens, claimed)`（阈值 `SLOT_MIX_MATCH_RATIO=0.9`、`SLOT_MIX_FILL_TOLERANCE=0.05`、`SLOT_MIX_MIN_TOKENS=5`）。
  **删除 `depthSlotTokensHint`**——它按 `chatHistory-N` 的 N 推 depth，而实测 N 是"自上而下第 N 条"的顺序号，
  会随注入条数与预算裁剪漂移、**与 depth 无稳定对应**（见下"实测"），用它查表会查错组。
- `attributeSlotsBySource`：`wi-merged` 槽只把"世界书那一部分"计入 `wi`，其余量按原角色归回 `chat`/`other`；
  混合槽的明细分摊改为"**本槽命中哪些条目就在哪些条目之间分摊**"（此前按命中条目的 position 归纳一个组，
  会把另一 position 的命中量漏在明细外——虽然 `Σ明细 ≡ wi` 仍成立，但单条明细会偏向一侧）。

**实测（本轮受控浏览器探针，`populationInjectionPrompts` 语义）**：深度注入在最终消息上的标识是
`chatHistory-N`，其中 **N 是"自上而下第 N 条"的顺序号**（受注入条数与预算裁剪影响），
**不能当 depth 用**；因此 D2′ 的"该槽世界书体量"改由**内容推导**（命中同一槽的 @D 条目必然同组）。

**验证**：
- `node --check` 通过；**Node 单测**：`stats2.test.mjs` **115 断言全过**（新增 D1/D2′/D3′ 与 `depthGroupKeyOf`/`isSlotMixed` 共 35 项），
  `stats3.test.mjs` 74、`shield.test.mjs` 12 同步全过。
- **审计装置重标定**：`.zcode/audit/wbaudit.mjs` 由 `scratch/genharness.mjs` 按新行号重新切片生成（26 块 / 882 行，
  新增 `isSlotMixed`/`depthGroupKeyOf`/`SLOT_MIX_*` 常量导出），块边界逐条核对无截断。
- **归档捕获数据回归**（浏览器内跑真实 ST 分词器，新方案 vs 旧方案；GT = 逐槽人工标注真值）：

  | 用例 | 配置 | 真值 | 旧方案(文本匹配) | 改前新方案 | **改后新方案** |
  |---|---|---|---|---|---|
  | A1 | squash=off | 585 | 574 | 585 | **585（不变）** |
  | A2 | squash=off | 661 | 614 | 625 | **625（不变）** |
  | E1 | **squash=on** | 689（净 663） | 614 | 514 | **624（+110，反超旧方案）** |
  | E2 | wi_format 包装 | 681 | 614 | 641 | **641（不变）** |
  | E3b | 尾部引用条目 | 680 | 614 | 641 | **641（不变）** |
  | E3c | 聊天照抄条目 | 679 | 620 | 658（含 +17 误报） | **641（误报归零，余 −38 全为漏判）** |

  槽位留痕佐证：E1 出现 `#0 main=wi-merged(138/110)`（合并槽按命中量计入 110）；E3c 不再出现被劫走的聊天槽。
- **受控 squash 实机实验**（临时世界书 3 条：pos0/pos1/@D(depth0)，跑完即删）：同一聊天、同一上下文，
  `squash=off` → 新方案 `wi=24463`；`squash=on` → **`wi=23370`**（改前仅 10942，旧方案恒 21313）；
  其余量（预设/角色卡等 4133）正确留在「其他」，三栏恒等于总计。
- **插件本体 ↔ 复刻装置端到端交叉校验**（页面重载加载新代码，走插件自己的 SETTINGS_READY 监听器 + 锚定）：
  同一次发送，插件锚点 `总计 29383 = 世界书 24372 + 聊天 1971 + 其他 3040` 与复刻装置**逐值完全一致**，
  且锚点里带 `slotDecisions` 留痕（6 个非聊天槽的判定与依据一目了然）。

**已知边界（本轮未改，已在审查文档登记）**：
- 深度槽"渲染后零命中"（如随机 EJS 条目）仍会漏判进「聊天/其他」（HB09，−36 量级）——文本匹配本质无法处理；
- 聊天槽命中覆盖 ≥ 90% 时仍按"照抄条目"计入世界书（阈值是折中，不是真理）；
- `wi_format` 包装串只有在**世界书槽位可识别**时才随槽位整条计入。

**环境还原**：临时世界书全部删除（`worlds/` 仍 7 个文件）；`settings.json` 的 `squash_system_messages=false`、
`world_info_format=null`（均为默认值，等价 `{0}`）；夹具 md5 未变（`740d15ff…` / `472b8baa…`）；
**用户聊天**：端到端校验时插件按 `STATS_ANCHOR_KEEP=2` 剪枝，把临时锚点计入后挤掉了较旧的一个真实锚点，
已用 `backups/chat_wuwa_…_20261013-002620.jsonl`（不含临时消息、两个锚点俱全）**逐字节还原**并重载核对
（`#4 28947/24742`、`#5 31826/26870` 两个锚点原样回来）。`.playwright-mcp/` 浏览器产物已删除。

---

### 2.18 世界书统计修复（第二轮审查的 D4/D5；本轮新增，已实施并验证）

**背景**：`世界书统计方案审查报告.md`（2026-10-13 第二轮独立审查，含与旧方案的逐值对照）实锤了两个缺陷：
一个是代码 bug（D4），一个是判定缺口（D5）。本轮一次修掉，并同步重建了审计装置。

**D4【P0，代码 bug】数字角色归一化错误 → 系统槽混槽误计世界书**：
- 位置：`normalizedSlotRole`。ST 运行时（`WORLDINFO_SCAN_DONE` 的 `activated.entries`）里 `entry.role`
  是**数字枚举 0/1/2**（world-info.js 的 `extension_prompt_roles`），而旧实现把"非 user/assistant 字符串"一律归
  `'system'` → **0/1/2 全被归成同一组** → `depthGroups` 把"同 depth 不同角色"条目算进同一 `win` →
  `isSlotMixed` 的"结构互补"判据在混合槽上永不触发 → 整槽（含第三方文本）被判为世界书。
- 修复：`normalizedSlotRole` 先按数字映射（0→system / 1→user / 2→assistant），字符串路径不变。
- 实测（修复前）：插件本体模拟发送，系统槽混入 11 token 第三方文本 → 面板「世界书 444」（真值 433，+11 误计）；
  合成复现 `win=38`（应 19）。修复后：`win=32`（正确分组），面板与真值一致。

**D5【P1，判定缺口】混槽被第三方"稀释"到阈值之下 → 整条世界书内容被漏计**：
- 两种子情形：① 聊天角色槽（@D 条目被分配 user/assistant 角色）被第三方文本稀释到覆盖 <0.9 →
  整条判「聊天」（实测 A/B/C/T8 漏计 29~34）；② 非聊天槽被稀释到覆盖 <0.5 → 整槽归「其他」（实测 S5 型漏计 12）。
- 修复：新增**混槽救援**——
  - 新增纯函数 `isSlotRescueMerged(win, tokens, claimed)`（阈值 `SLOT_MIX_RESCUE_MIN_TOKENS=2` 绝对下限，
    其余同 `isSlotMixed`：命中量盖住 win 的 90%）；与 `isSlotMixed` 的区别是不要求整槽 5% 比例容差
    （救援场景的"第三方小尾巴"可能恰好 ≤5%，不能被吞掉）。
  - 新增 `slotIsChatFloor(slotText, chatSnapshot, genType)` + `slotContainsChatText`（楼层包含判定）：
    **只有明确"不是真实聊天楼层"（`false`）的聊天槽才允许救援**；真实聊天楼层不救援（保住 D3′ 防照抄），
    快照缺失（`null`）时救援被禁用（保持旧的保守行为，避免影响无快照的合成/历史用例）。
  - `classifyPromptSlot` 新增参数 `isChatFloor`：聊天分支在 `isChatFloor === false && win !== null &&
    命中全来自世界书位置 && isSlotRescueMerged(...)` 时返回 `wi-merged`；非聊天分支在覆盖 <0.5 后同样补一条
    救援判定（`win === null` 时不改判）。救援结果由 `attributeSlotsBySource` 按既有 `wi-merged` 语义处理：
  **只认命中量计入世界书，其余量按原角色归回「聊天」/「其他」**。
- 实测（修复后）：浏览器真实分词器，5 组此前的失败捕获（A/B/C/T7/T8）**全部回到真值 494**；
  A/B/C/T8 的救援槽判定为 `wi-depth`（命中量=win，直接归位）、T7 为 `wi-merged:43/32`（11 归回「其他」）。

**实现（`hide/index.js`，统计段）**：改动 6 处——
`normalizedSlotRole`（数字映射）、新增 `slotContainsChatText`/`slotIsChatFloor`（`computeSendableChat` 之后）、
新增 `isSlotRescueMerged`（`isSlotMixed` 之后）、`classifyPromptSlot` 签名 + 两处救援分支、
`computeRenderedStats` 里预计算 `slotFloorFlags` 并传入。

**验证**：
- `node --check` 通过；审计装置（`audit-dev/`）由生成器在新行号上重新切片（边界断言全过）。
- `node audit-dev/fix2.test.mjs` **18/18**：修复目标 5 组（T8 型 34 / A-B 型 34 / C 型 29 / S5 型 12 / D4 型 32）
  + 护栏 2 组（真实楼层照抄不救援、无快照不救援）+ 回归 4 组（前后置 / 引用不触发 / 渲染漂移不触发 / squash 不回归）。
- `node audit-dev/compare.test.mjs` **53/53**：12 组合成对照，补丁版在所有场景 ≥ 旧方案。
- **浏览器实机（插件本体端到端）**：临时世界书（azure user@d2 + beacon system@d2）+ 两个第三方注入
  （user 小尾巴 + system 小段）→ 模拟发送 → 面板 `世界书 433`（真值 433 = 367+34+32），
  第三方 11 token 正确落在「其他」的「深度注入（世界书等）」栏；修复前同场景为 444（D4 误计）。
- 回归确认：T2/T3/T4/T5/T9 等既有场景不受影响（`compare.test.mjs` 覆盖）。

**环境还原**：临时世界书删除（`worlds/` 恒 7 文件）；`globalSelect=['Eldoria']`；测试期产生的备份已清理
（本轮 08:xx 共 9 份：7 份夹具 + 2 份用户聊天）；夹具与用户聊天已从精确备份还原至基准 md5
（`740d15ff…` / `472b8baa…` / `d3e97c2b…`，用户聊天差异仅为 Horae 自身时间戳记账，正文/锚点逐字节一致）。

---

### 2.19 世界书统计第三轮修复：WI 标识槽内容护栏（D6）+ D4 残差（数字字符串角色）（本轮新增；已实施并验证）

**背景**：对「世界书」新旧两套口径做严格审查，并复核另一份独立审查报告的 D4/D5 结论。复核确认
D4/D5 技术判定成立（§2.18 已修）；本轮**新发现并修复一个触发条件更贴近真实配置的缺陷 D6**，
并补上 D4 未覆盖的**数字字符串角色**（D4′）。完整结论、逐场景对照与「修复方案 2.0」见
[`世界书统计方案最终审计报告与修复方案2.0.md`](./世界书统计方案最终审计报告与修复方案2.0.md)。

**D6【P1，新发现】`worldInfoBefore/After` 只认标识 → squash 存活者错位时把非世界书内容算进「世界书」**
- ST 事实（`openai.js:3922-3954`）：`squashSystemMessages()` 把**相邻 system 槽**合并、**沿用首条标识与 role**；
  空 system 槽被强制跳过（`3931-3933`）。默认提示词顺序里 `main` 在最前（`PromptManager.js:2087-2136`），
  所以通常是"main 吞掉整块"——那条路径 §2.17 的 D1 已处理（判 `wi-merged`，只认命中量）。
- **漏掉的路径**：`main_prompt` 为空时（ST 默认值，用户配置实测也是空串）最终 `main` 消息仍带 ST 的
  默认提示词（`openai.js:102/691` `default_main_prompt`），因此**只要把 main 禁用**（或在提示词顺序里
  把它挪到 `worldInfoBefore` 之后），存活者就变成 `worldInfoBefore`：角色卡描述/人设/场景/NSFW 等
  **非世界书内容**被按标识整块算进「世界书」（改动前对此分支没有任何内容护栏）。
- 实现：`classifyPromptSlot` 的 `worldInfoBefore/After` 分支加**内容护栏**（与 D1 同源）——
  该块世界书体量（`wiBlockTokens`，含被 squash 并入的另一块）已被命中全（`claimed ≥ 0.9×块体量`）
  而槽位明显更大（超出 > max(5%, 5t)）→ 判 `wi-merged`：只认命中量，余量按原角色归回「其他」。
  `computeRenderedStats` 新增 `wiBlockTokens[0/1]`、`slotInputs`（判定输入，供判定与留痕共用），
  `slotDecisions[].win` 对前置/后置槽改为显示该块体量（可观测性增强）。
- 代价（与 D1 一致）：合并槽里"比世界书多出来的部分"（`wi_format` 包装串、被并入的其它提示词）落「其他」——
  宁可少算包装，也不把别人的文本算成世界书。

**D4′【P2，D4 未覆盖】角色写成数字字符串（`"1"`/`"2"`）**
- ST 侧 `setExtensionPrompt` 用 `Number(role ?? 0)` 折算角色（`script.js:8932`），`"1"` 就是 user；
  而 `normalizedSlotRole` 只认数字与 `'user'/'assistant'` 字符串 → `"1"/"2"` 被归成 system，
  与真正的 system 条目并组 → win 虚高 → 稀释槽救援失效 → 世界书整条漏计（合成实测 184→150，−34）。
- 可达路径：角色卡世界书导入原样照抄 `entry.extensions?.role`（`world-info.js:5656`），手写/AI 生成的
  卡片可能写成字符串。
- 实现：`normalizedSlotRole` 数字优先 + 接受数字字符串（含大小写/空格归一），`'0'`/`NULL`/未知值 → system。

**验证**
- `node --check` 通过；审计装置按新行号重新切片；生成器（`audit-dev/gen-h2.mjs`）改为**按锚点注释推导切片终点**，
  行号漂移不再让生成失败。
- 单测：`audit-dev/fix3.test.mjs` **18/18**（D6 目标 2 + 护栏回归 2 + D4′ 1 + 角色表 11 + 探针）、
  `audit-dev/fix2.test.mjs` 18/18、`audit-dev/compare.test.mjs` 53/53、
  `audit-dev/rolebug.mjs` **9/9**（重写为断言式回归守卫：数字/字符串/数字字符串角色逐值一致，
  且"同 depth 不同角色不并组"）。
- **浏览器实机（插件本体端到端，真实分词器，「模拟发送」；夹具 `StarTest - fixture` + 临时条目只写前端缓存）**：

  | 配置 | 槽 #0 标识 | 槽 token | 面板「世界书」 | 真值 | 结论 |
  |---|---|---|---|---|---|
  | squash off（改前/改后） | `worldInfoBefore`（纯） | 172 | 675 | 675 | 无回归 |
  | squash on + main 启用（存活者 `main`，D1 路径） | `main`（合并） | 439 | 675 | 675 | 既有 D1 已正确 |
  | squash on + **main 禁用**（存活者 `worldInfoBefore`，**改动前代码**） | `worldInfoBefore`（含角色卡） | 423 | **790** | 675 | **+115 误计（D6 实锤）** |
  | 同上，**改动后代码** | `worldInfoBefore`（含角色卡） | 448 | **675** | 675 | 修正；余量 140 落「其他」 |

- 旧方案（文本匹配）同场合同值 675/675（它在"命中多少算多少"上天然近似 `wi-merged`）——即 D6 属于
  "新方案在混槽场景一度输给旧方案"的那一类；修复后新方案在全部已测场景 ≥ 旧方案。

**残余边界（本轮未改，已在审查报告登记）**：
S6 型（**被改写**的真实聊天楼层恰好含满某注入组的世界书文本时会被救援，实测合成 +34）；
非数字且非 `'user'/'assistant'` 的角色字符串（ST 侧 `Number()` 得 NaN → 该注入被整条丢弃，
插件仍按其语义归组，可能让 win 略虚高）；位置 2/3/5/6/7（AN/对话示例/出口）注入的世界书内容按
"来源槽位"定义归「其他」；"渲染后零命中"（纯随机 EJS）仍漏判；合并槽里的 `wi_format` 包装串落「其他」。

**测试环境还原**：临时世界书条目只写前端缓存（`Eldoria.json` md5 未变 `cf8ac228…`）；
`settings.json` 因 `chatCompletionSettings` 是**实时对象**、页面自动落盘而被污染（squash=true / main 禁用 /
nsfw_prompt），已用备份 `settings_default-user_20261013-063044.json` **逐字节还原**（`28175ec5…`）；
夹具 `StarTest - fixture.jsonl`（`740d15ff…`）、`StarTest - short.jsonl`（`472b8baa…`）、
角色卡 `StarTest.png`（`aecf09be…`）均未变；用户真实聊天仅 Horae 自身 `horae_meta.timestamp` 记账差异，
已按 `chat_wuwa_…_20261013-060523.jsonl` 逐字节还原（`d3e97c2b…`）；本次测试窗口产生的 25 份备份
（fixture / 用户聊天 / 被污染的 settings 快照）已删除。`.playwright-mcp/` 产物无新增。

**同日追加：第三轮代码审查反馈的三处加固（已实施并复验）**

- **A①（纯槽空白残余）**：`worldInfoBefore/After` 的护栏加 `hasForeignText` 门控——"命中区间之外存在**非空白文本**"
  才是"槽里混了别的内容"的必要条件；纯槽里条目之间/首尾的空行不再被误判成混合槽（否则世界书少算 ≤10% 块体量）。
  `audit-dev/fix3.test.mjs` 新增 A4 用例：断言"只看阈值会误判（`isSlotMixed=true`）、加门控后不误判"。
- **B（合并槽里另一块世界书没匹配到）**：`wiBlockTokensOfSlot` 改为
  "标识所指那块 ∪ 命中到的另一块，**若另一块体量装得进这一槽**"——防止世界书文本被 Regex
  （`regex_placement.WORLD_INFO`，`world-info.js:5205`）改写、或 EJS 条目超过 `MAX_RENDERED_EJS_ENTRIES=80`
  未渲染时，把"其实是世界书"的整块漏进「其他」。`fix3` 新增 B 用例（真值 = 整槽）。
- **D（非数字角色字符串）**：`normalizedSlotRole` 完全对齐 ST 的 `Number(role ?? SYSTEM)`
  （`script.js:8926-8934`）：数字 / 数字字符串（`"1"`/`"01"`/`" 1 "`）等价；**非数字（`'user'`/`'AI'`/对象…）→
  独立分组键**——ST 侧 `Number()` 得 NaN，`getExtensionPrompt` 按数字严格比较会把它整条过滤掉
  （`script.js:3309-3315`），原实现却把它并进真实角色组 → `win` 虚高 → 混槽护栏与 D5 救援双双失效。
  同批新增 `depthGroupKeyOfEntry`：`depth` 非有限数（ST 侧同样丢弃，`openai.js:820-822`）独立成键，
  `null/undefined` 按 `DEFAULT_DEPTH=4`（`world-info.js:96`）计。`fix3` 新增 D 用例 + 21 项角色折算取值表。
- 复验：`fix3` **32/32**、`fix2` 18/18、`compare` 53/53、`rolebug` 9/9；
  **浏览器实机第二轮（加固后代码）**：`squash off` → 世界书 675 / 聊天 147；`squash on + main 禁用` → 世界书 675
  —— 与加固前逐值一致（真实管线上的判定未被改变，加固只堵合成反例）。
- 环境：第二轮产生的 13 份备份已删除；`settings.json` 的自动回写只涉及
  `extension_settings.variables.global.LAST_RECEIVE_CHARS/TOKENS` 两字段（变量扩展自身记账，不含本轮实验开关），
  已回退到基准 md5 `28175ec5…`；用户聊天再次仅 Horae 时间戳差异，已逐字节还原（`d3e97c2b…`）。

---

### 2.20 世界书统计第四轮：独立复验（实机 A/B）+ 两处「多算」缺陷修复（P2-2/P2-3）（本轮新增；已实施并验证）

**背景**：对 §2.18/§2.19 的成果做一次**独立复验**——不重跑旧实验，而是① 重跑全部审计装置；
② 用真实 ST 管线（插件「模拟发送」+ 真实分词器 + 夹具 `StarTest - fixture`）重做**受控 A/B**；
③ 派只读 code-reviewer 子代理对统计段做对抗式审查，我逐条**独立复现**后才采纳。完整记录：
[`世界书统计方案最终审计报告与修复方案2.0.md`](./世界书统计方案最终审计报告与修复方案2.0.md) §十二、§八-10…13。

**A. 复验证据（插件本体端到端，真实分词器；真值 = 逐槽独立算术和）**

| 场景 | 槽结构 | 面板「世界书」 | 独立真值 | 结论 |
|---|---|---|---|---|
| 基线（`squash=off`，仅 Eldoria @D 条目） | 1 个 @D 槽 367t | **367**（聊天 147 / 其他 2697 / 总 3211） | 367 | ✓ 逐项一致；旧方案同值 |
| Run C：临时 pos0 455t + pos1 328t（**只写前端 `worldInfoCache`**）+ @D 367t，`squash=off` | 三个纯槽 | **1150** | 455+328+367=1150 | ✓ |
| Run B：同内容 + `squash=on` + **禁用 `main`**（存活者 `worldInfoBefore`，合并槽 1273t） | 判定 `wi-merged`（claimed 783 / win 783） | **1150** | 1150 | ✓ **D6 护栏在真实管线上生效**；无护栏会同场景算成 1273+367=1640（+490） |
| Run D：同一个 `depth+role` 注入槽混入 96t 第三方文本（模拟 LWB/柏宝书同槽） | 槽 463t（367 世界书 + 96 第三方） | **367**（聊天 **243** = 147+96） | 367 | ✓ **D5 救援在插件本体生效**（离线复刻装置传 `chatSnapshot: []` 会禁用救援，故该场景只能看面板，见报告 §四①） |

**B. 已修缺陷（第四轮独立代码审查发现 → 我用 `audit-dev/probe4.test.mjs` 复现 → 修复 → 复测）**

- **P2-2【多算，已修】合并 system 槽的「另一块世界书装得下就计入体量」会关掉内容护栏**：
  当另一块世界书**其实不在本槽**（它有自己的槽，或已出现在别的槽）时，`wiBlockTokensOfSlot` 只按
  「体量装得下」就把它的体积加进 `own` → `claimed < 0.9×own` → 护栏不触发 → **整槽**（角色卡/预设立场文本）
  算进「世界书」。复现：`wi 2968` vs 正确 `1113`（+1855）。修复：新增 `presentIdentifiers`（本请求实际存在的槽标识）
  与 `blocksWithHits`（在任意槽里命中过的块），**只有"那一块既没命中任何槽、也没有自己的标识槽（= 被 squash 合并掉）"**
  才计入体量。复测：`wi 1113` = 正确；槽判定由 `wi-before` 变 `wi-merged`（claim 703 / win 703）。
- **P2-3【少算，已修】深度分组键的「非有限 depth 独立成键」是死代码**：分组表用 `Number(e.depth) || 0` 折算值建键，
  而查表键用原始值 → 两处口径不一致；被 ST 丢弃的注入（`Number('abc')=NaN`，`setExtensionPrompt`/`getExtensionPrompt`
  严格比较后整条不进上下文）被并进**真实 depth 组** → `win` 虚高 → D5 救援失效。复现：`wi 0` vs 正确 `139`。
  修复：`depthGroupKeyOfEntry` 完全按 ST 的 `Number(depth)` 口径归一（**数字字符串 `'4'` 与 `4` 等价**），
  分组表改用新增的 `entriesMeta.depthRaw`（原始值）建键。复测：`wi 139` = 正确。

**C. 本轮登记、未修（理由见报告 §八）**

| 编号 | 症状 | 方向 | 触发条件 | 为何本轮不改 |
|---|---|---|---|---|
| P1-1 | `squash=on` 且 `main` 非空时合并槽标识是 `main`；该槽世界书占比 <50% → 整块落「其他」（复现：槽 2535t / 世界书 623t=25% → `wi 0`，理想 623） | 少算 | squash on + main 非空 + 世界书占比 <50% | **已于 §2.21 修复**（新增"被 squash 合并掉的块"结构护栏 + 分支③救援；实机 squash on 由 367 修正为 990） |
| P2-4 | 深度注入槽只要**引用了 ≥8 字符的待发送聊天消息原文**，`slotIsChatFloor` 判 true → 救援禁用 → 整块归「聊天」（复现：`wi 0`，理想 626） | 少算 | 条目正文含聊天原文 | 放宽"楼层"判定会削弱 D3′/S6 护栏（反向误计），属取舍 |
| P4-5 | 救援阈值 `SLOT_MIX_RESCUE_MIN_TOKENS=2` + 无 5% 容差 → 1 token 抖动即整组归属跳变（100/101/90 不救援 vs 103 救援） | 阈值固有 | 边界值 | 任何阈值都有跳跃；调小会削弱"小尾巴"判据 |
| P4-6 | 「模拟发送」与真实发送共用 `lockedScanEntries`/`statsComputing`：并发时模拟锚点可能用错条目集，**真实发送会被 `if (statsComputing) return` 静默丢掉锚点** | 统计缺失 | 模拟发送后立刻真实发送 | 属事件接入层，正确改法是**排队**而非丢弃；超出本轮范围 |

**D. 装置与复跑**

- 新增 `audit-dev/probe4.test.mjs`（14 断言：P2-2/P2-3 断言正确值、P1-1/P2-4 断言已登记现状 + 恒等式/明细≡wi）。
- `node audit-dev/gen-h2.mjs` 重新切片（`wbaudit-new.mjs` 终点 2456）。
- 复跑：`probe4` 14/14、`fix2` 18/18、`fix3` 32/32、`compare` 53/53、`rolebug` 9/9。

**E. 环境与残留（本轮实机只动内存/前端缓存，跑完逐项还原）**

- 还原对象：`worlds/Eldoria.json`（`cf8ac228…`，临时条目只写 `worldInfoCache`）、`worlds/__perf_regression_test__.json`、
  `characters/StarTest.png`（`aecf09be…`，描述只在内存改过）、夹具 `StarTest - fixture.jsonl`（`740d15ff…`）、
  `StarTest - short.jsonl`（`472b8baa…`）、用户真实聊天（`d3e97c2b…`）**逐字节还原**（测试期仅 Horae 自身
  `horae_meta.timestamp` 记账差异）；`settings.json` 因 `chatCompletionSettings` 是实时对象被页面自动落盘
  （squash/main/临时描述）→ 用测试前备份逐字节还原（`e971c3e9…`）。测试窗口产生的 17 份备份（10 份夹具 +
  6 份用户聊天 + 1 份被污染 settings 快照）已删除（`backups/` 回到 318）。
- **清理上一轮残留两处**：① `chats/WuWa Solaris-3 MVU Edition/StarTest - fixture.jsonl`（173049B，09:50 由
  "先切聊天再切角色"的守卫漏网产生：WuWa 角色下新建的聊天被存成夹具名，内容只有该卡开场白的 17 个 swipe）；
  ② `chats/StarTest/StarTest - audit-wb.jsonl`（1328B，单条测试消息）。两者均**移出**到会话隔离目录
  `…/scratch/…/quarantine-round4/`（未删除，字节与原文件一致）。夹具目录现为 `fixture` + `short`（与文档一致），
  页面最终停在用户真实角色/聊天。

---

### 2.21 P1-1 修复（squash 合并槽世界书整块漏计）+ 移除「其他」构成明细（本轮新增；已实施并验证）

**背景**：用户要求对 §2.20 C 表登记的 P1-1 先做**必要性审查 + 实机测试**再决定；同时决定**移除「其他」构成明细**
（口径鸡肋、来源难辨），把重点保留在「世界书」激活条目明细上。

**A. P1-1 必要性（ST 源码 + 实机 A/B）**

- ST 事实（`openai.js:3922-3954` `squashSystemMessages()`）：相邻可合并 system 消息**合并进该段第一条**并沿用其标识
  （`lastMessage.content += …`）；空 system 消息被跳过；`excludeList` 仅 `newMainChat/newChat/groupNudge`（带 `name` 的也不合并）。
  `squash_system_messages` 默认 **false**，但随预设保存（用户预设 `2525.json` 为 `false`）。
- 触发条件（三者同时）：① 打开 squash；② 首个**非空**可合并 system 消息不是世界书块（标准顺序 `main → worldInfoBefore → …`，
  ST 默认 main 非空、用户预设 main 78 字符非空 → 存活标识即 `main`）；③ 该合并消息里"卡片/预设"体量 > 世界书前置+后置。
  → 卡重 + 常驻世界书不多的用户会**整块丢失**；世界书本身很大时（如用户真实聊天 wi 26870）走既有 `cov ≥ 50%` 分支本来就对。
- **实机复现（修复前；插件本体 + 真实管线 + 真实分词器；夹具 `StarTest - fixture`，临时 pos0 常量条目 623t 只写前端缓存）**：

  | 配置 | 槽结构 | 面板「世界书」 | 真值 | 结论 |
  |---|---|---|---|---|
  | squash off | 7 槽，`worldInfoBefore` 623t 独立 | **990** | 990 | ✓ |
  | squash on | 5 槽，**`main` 2269t** 合并（含 623t 世界书） | **367** | 990 | **−623（整块丢失，−63%）** |

  同场装置对照：新方案 367（错）/ **旧方案 990（对）**；槽判定留痕 `main: other (t2269/c623/wnull)`——
  623 已被精确匹配，只是被"覆盖过半"那条线整槽丢弃。
- **修复（2 处，共 +917 字符）**：
  1. `computeRenderedStats` 的 `slotInputs`：把"该槽里的世界书块体量"从"只算 `worldInfoBefore/After` 槽"扩展到
     **任意命中全来自 position 0/1 的槽**（`blockHitsOnly`）；关键护栏是**"那一块自己的标识槽在本请求里不存在"**
     （`mergedAway` = 确实被 squash 合并掉了）——squash 关闭时块仍有自己的槽 → 该值恒为 `null`：既不会把
     "别处照抄条目文本"算成世界书，也避免"块在自己槽里整块计入 + 又被救援一次"的双重计数。
  2. `classifyPromptSlot` 分支③（已知标识 + 非聊天）：`wiBlockTokens > 0 && isSlotRescueMerged(...)` → 返回既有类型
     `wi-merged`（只认命中量，余量按原角色归回「其他」）。
- **修复后实机复验**：squash on → 面板 **990**（`main: wi-merged t2089/c623/w623`，明细 `eldoria 367 + P11B-before 623`）；
  squash off → **990**；两组**逐项一致**（总 5274 / 聊天 147 / 世界书 990 / 其他 4137）——统计不再随 squash 开关漂移。
- 残余（未改，报告 §八）：块文本被 Regex 改写 / EJS 未渲染导致匹配不到时救援不触发（与改前同）；槽标识对齐失败（`null`）不救援；
  squash 开启下"预设真的照抄整块世界书原文"仍可能多算（结构护栏只挡住了 squash 关闭情形）。

**B. 移除「其他」构成明细**

- 移除：`otherBreakdown` 计算（含 `attributeSlotsBySource` 的 `otherSlots` 收集与返回）、`categorizePromptIdentifier`、
  统计页 HTML 的 `#tub-other-breakdown`、`renderOtherBreakdown`、`renderTokenStatsContent` 的相关入参与清理分支、
  `style.css` 的 `--hh-stats-bg-other`（亮/暗 + hh 别名）与 `#tub-other-*` 规则；新锚点不再写 `otherBreakdown`（旧锚点该字段被忽略）。
- **保留（关键）**：`capturePromptIdentifiers` / `alignPromptIdentifiers` / `lastPromptCapture` 与 `GENERATE_AFTER_DATA` 采样——
  槽位来源判定（`worldInfoBefore/After`、`chatHistory-N`、`main`）全靠它；「其他」总数（`total − chat − wi`）保留；相关注释同步改写。
- 影响：锚点体积略降；squash 合并槽里"余量被标成「预设」/「世界书包装格式」"这类**误导标签彻底消失**。

**C. 验证**

- `node --check`（index.js + 装置）通过；`audit-dev/gen-h2.mjs` 边界断言全过（新行号 1447/1515/1517 与 1539-1640）。
- 装置复跑：`probe4` **20/20**（新增 P1-1 与 P1-1b：块仍有自己的槽时**不得**救援）、`fix2` 18/18、`fix3` 32/32、
  `compare` 53/53、`rolebug` 9/9。
- 实机：见 A 表（双组），面板数字与装置逐值一致；`#tub-other-breakdown` 在 DOM 中已不存在。

**D. 观察（未改，非本轮引入）**

- 「模拟发送」跑真实管线时 ST-PT 会真执行 EJS → **聊天级变量（`chat_metadata.variables`）会被写入**，再由插件的
  `saveChat()` 落盘：实测夹具自带的 `<% setvar('数据库用户名','User') %>` 被写进夹具文件（+75B，含 LWB 快照）。
  统计自身的渲染有副作用屏蔽（`READONLY_ENV_OVERRIDES`），但**管线运行无法屏蔽**——属"模拟发送"既有行为，已还原夹具；
  若在意，可后续把"模拟发送"改为不落盘 / 恢复 `chat_metadata`。

**E. 环境还原**

- 本轮唯一被改文件：`hide/index.js`、`hide/style.css`；其余全部逐字节还原——`settings.json`（`e971c3e9…`）、
  预设 `2525/Default/yhrhtrh`、`worlds/Eldoria.json`（`cf8ac228…`）、`characters/StarTest.png`（`aecf09be…`）、
  夹具 `StarTest - fixture.jsonl`（`740d15ff…`）/`- short.jsonl`（`472b8baa…`）、用户真实聊天（`d3e97c2b…`，
  测试期仅 Horae 自身 `horae_meta.timestamp` 记账差异）；测试窗口产生的 19 份备份与 `.playwright-mcp/` 已清理
  （`backups/` 回到 318）；页面最终停在用户真实角色/聊天。
- **失误与纠正（如实记录）**：① 给 `openCharacterChat()` 传了**带 `.jsonl` 扩展名**的名字 → ST 新建了一张 1 条消息的
  聊天、多出 `StarTest - fixture.jsonl.jsonl`、并写脏 `StarTest.png` 的 `chat` 字段与 settings 的 `LAST_RECEIVE_*`
  → 已删除多余文件并逐字节还原；② 一次读取用了旧快照，把放大文本误写进 **WuWa 卡片描述**（仅内存、未落盘）
  → 立即还原为空串。

---

### 2.22 第十一轮：P4-6 统计串行队列 + P4-8 无变量楼层还原 + P2-5（R3）门控（本轮新增；已实施并验证）

**背景**：用户审阅《第六轮审查报告》与《交叉验证与补充》后，裁定先修「与 squash 开关无关的多算、并发丢锚点、无变量楼层还原、措辞订正」四项；
P2-4（引用聊天原文误判楼层）单列，留待后续单独复核（本轮**未动**）。

**修复 1：P4-6 —— 统计改为 FIFO 串行队列 + owner token + 排队告警（`index.js`）**

- 旧实现用布尔量 `statsComputing` 做互斥，两处 `if (statsComputing) return;` 直接丢弃并**静默无日志**；
  且 `return` 位于 `try` 内 → 其 `finally` 会把**正在执行者**持有的锁一并释放 → 后到的统计"穿透"并发、
  互相覆盖锚点（交叉验证实测：连发 400/40000/8000 → 40000 静默丢弃、8000 穿透写入）。
- 新实现：模块级 FIFO 队列 `statsJobQueue` + `statsRunOwner`（**引用比较的 owner token**，
  `finally` 里 `if (statsRunOwner === owner) statsRunOwner = null;` 只释放自己的锁）；
  `enqueueStatsJob(job)` 空闲立即调度、忙碌排队（有 `Logger.warn`）；`drainStatsQueue()` 串行取下一个；
  队列上限 `STATS_QUEUE_MAX = 8`（异常积压时丢弃最早的**待执行**任务并告警）。
- 两个入队点（真实发送 / 模拟发送）均把"本次条目集"改为**随任务携带**（`realEntries` / `simEntries`），
  `computeRenderedStats` 新增 `options.entries`（缺省回退模块级 `lockedScanEntries`，兼容审计装置）；
  排队等待期间模块级变量再被覆盖也不影响本次任务的归因。`lockedScanEntries` 模块级变量保留
  （`WORLDINFO_SCAN_DONE` 写入、`CHAT_CHANGED` 清空、装置回退路径使用）。
- 验证：`audit-dev/probe8.test.mjs` **16/16**（串行性 maxActive=1、三连发全部执行且严格有序、
  抛错任务 onError 且不泄漏锁、队列上限丢最早+告警、源码守卫 4 项）；`probe5` E 节翻转为修复后守卫（52/52）。

**修复 2：P4-8 —— 无 `variables` 字段楼层的还原漏洞（`index.js`）**

- 旧 `snapshotMessageVariables` 对"两个字段都为 null"的楼层记 `null`（跳过保护）；若管线给这类老楼层
  写入变量（ST-PT `withMsg` 指名写入），就会在内存残留并随插件自身 `saveChat()` 落盘。
- 修复：这类楼层也纳入快照（`{ variables: null, variables_initialized: null }`），还原时
  **删除字段**（而不是跳过）；只有消息对象无效/克隆失败才记 `null`。
- 验证：`probe5` F 节翻转（52/52：无变量楼层被快照、还原为"字段不存在"、changed=true 仍可由落盘守卫使用）。

**修复 3：P2-5（R3）—— cov≥50% 改判加 `mergedAway` 门控（`classifyPromptSlot` 分支③）**

- 旧逻辑"命中全来自世界书位置 + 覆盖≥50%"即判 `wi-merged`；当块自己的标识槽仍在（squash 关闭）时，
  别处"照抄 ≥50% 条目正文"同样满足 → 双重计数（实测 wi=2×、明细超条目真实体积）。
- 修复：命中里若含 0/1（前置/后置块），必须 `wiBlockTokens > 0`（= 该块自己的标识槽在本请求里不存在、
  确实被 squash 合并掉、且命中已盖住块体量）才允许改判；纯 @D(4) 命中不受影响。
- 验证：`probe7` T2 翻转为修复后守卫（17/17：squash 关闭照抄槽判 `other`、wi 只算一份）；
  P1-1（块确被合并掉）不回归（probe4 20/20）；已知残留：块确被合并掉时槽内"真照抄"仍无法与"并入内容"区分
  （probe7 T1a 保留现状断言），标识对齐失败（null）分支不经过该门控——均在报告中登记。

**措辞订正（两份报告共同指向）**

- `index.js`：`runSimulateSend` 上方注释与模块顶部注释删除"不落盘、不改动聊天记录"的绝对化表述，
  改为说明"**先还原、后落盘** + 落盘守卫（chatId 同 + 消息数同 + 确有改动）"，
  并注明"窗口内第三方落盘（ST-PT autosave / LWB 定时器 save）会把临时消息与未还原元数据写盘、
  且被 ST 服务端 `backups/` 备份机制留存"（第六轮 P4-7 实证）。
- `AGENTS.md` §四-5/§四-6 同步改写（P4-6/P2-5/P4-8 标"已修"，P4-7 表述订正）。

**验证汇总（第十一轮）**

- `node --check index.js` 通过；`gen-h2.mjs` 按新行号重切片（c1：1489-1557；c2：1559-2466；rep：1581-1682），
  边界断言全过。
- Node：`probe8` **16/16**（新）、`probe5` **52/52**（含 3 节翻转）、`probe7` **17/17**（含 1 节翻转）、
  `probe4` 20/20、`fix2` 18/18、`fix3` 32/32、`compare` 53/53、`rolebug` 9/9。
- 浏览器实机（真实管线 + 真实分词器，夹具 `StarTest - fixture`）：
  ① **P4-6**：连发合成事件 400/40000/8000 → 三者**依次全部执行**（修复前 40000 被丢弃、8000 穿透）；
  ② **P4-8**：给"本来无 `variables` 字段"的楼层在模拟期间写入变量 → `finally` 后字段被删除
  （修复前残留）；③ **P2-5**：squash 关闭 + 照抄 ≥50% → 面板 wi 只算一份（修复前 2×）；
  ④ 真实发送锚定链路回归（合成 SETTINGS_READY → 锚点正常写入与覆盖）。
- 环境：夹具 `740d15ff…`、settings `e971c3e9…`、Eldoria `cf8ac228…`、用户聊天 `d3e97c2b…` 全部逐字节还原；
  测试窗口备份已清理；页面回到用户真实角色/聊天。

### 2.23 第十二轮：P2-4 修复 —— 用「结构身份」判定聊天楼层（不再靠内容相似度）（本轮新增；已实施并验证）

**背景**：第六轮审查实机复现了 P2-4：一条"深度注入"世界书条目只要**引用了 9 个字符的聊天原文**，
`slotIsChatFloor()` 就判"这是真实聊天楼层" → 混槽救援被禁用 → 该槽 1210 t 世界书整块掉进「聊天」
（面板 世界书 1577 → **367**、聊天 147 → **2017**）。用户裁定先做方案 A（结构身份），失败再退回比例判据。

**根因（ST 源码链）**：`chatHistory-N` 的 N = **合并数组（聊天楼层 + 注入块）里自上而下第 N 条**，
且编号发生在**预算裁剪之前**（`openai.js:810-874` 注入插入 + `948-955` 编号）；但
`ChatCompletion.getChat()`（`openai.js:4120-4141`）**只输出 `{role, content, name?}`，把 identifier 丢掉**，
于是 `getChat()` 之后只剩"内容"可看 —— 旧实现只能用"槽里出现过 8 个字符聊天原文"这种松判据。

**修复（`index.js`，四条）**

1. **结构身份预测**（新纯函数）：`buildInjectionBlocks(presetPrompts, extPrompts)` 复刻 ST 的注入块枚举
   （(depth, order 组, role) 各一条；order 组从大到小、组内 system→user→assistant），
   `predictInjectionNs(floorCount, blocksByDepth)` 复刻插入与编号（"新→旧"数组上 `splice(depth + 已插入数)`，
   再反转为"旧→新"），返回**注入块占用的 N 集合**与合并长度。数据源与 ST 同源：
   `promptManager.getPromptCollection(genType)`（`openai.js:1470` 用的同一个）+ `extension_prompts`（IN_CHAT）。
   形状不可信（depth/role 非有限数）→ 返回 null。
2. **采集与自检**（`capturePromptIdentifiers`）：在 `GENERATE_AFTER_DATA` 与 identifier 同时产出
   `lastPromptKindCapture`（`'floor' | 'injection' | null`）；**自检**：树里出现的最大 N 必须等于预测的合并长度，
   否则整体回退（`null`）；配合 `alignPromptKinds()` 与 identifier 同步消费（条数/命中率同套校验）。
3. **判定改用身份**（`computeRenderedStats`）：`kinds[i] === 'floor'` → `isChatFloor = true`（禁救援）；
   `'injection'` → `false`（允许救援）；否则回退原 `slotIsChatFloor()` 内容启发式。
   `classifyPromptSlot` 新增 `isIdentityFloor`：为真时**直接归「聊天」**（连 NARRATOR 等 system 角色楼层也正确）。
4. **留痕**：`slotDecisions[i].by = 'identity-floor' | 'identity-injection' | 'heuristic'`（旧锚点无此字段）；
   留痕条件由"非聊天槽"改为"kind ≠ chat"，身份判定的注入型聊天槽也会被记录。
   顺带：`computeSendableChat` 补 `extra[IGNORE_SYMBOL]` 跳过（与 ST `setOpenAIMessages` 对齐；不依赖外部函数，
   用 `Symbol.for('ignore')` 兜底），NARRATOR 楼层仍计入。

**为什么这样就不再两难**：判定不再看内容相似度 → P2-4（引用聊天原文的注入块被当成楼层）与
S6 / D3′（真楼层照抄条目正文被救援进世界书）**同时消失**，P4-5 的阈值讨论对聊天侧失效；
预测失败/条数不符时**逐字节回退**到旧行为（`kinds = null`）。

**验证**

- Node：新增 `audit-dev/probe6.test.mjs` **32/32**（① 预测纯函数：单块 N=11/13、depth0→13、depth≥楼层数→1、
  三角色连续、预设 in-chat 计入、非有限值→null；② 身份优先判定；③ 端到端：兜底 wi=0/chat=782 ≡ 改前、
  身份=注入块 → wi=626（=命中量）/chat=156、身份=真楼层 → 整槽留「聊天」；④ **S6 反向对照**：
  兜底 wi=1250（≈2×625，多算）→ 修复后 wi=625）。
  回归：`probe4` 20、`probe5` 52、`probe7` 17、`probe8` 16、`fix2` 18、`fix3` 32、`compare` 53、`rolebug` 9 全绿。
  `gen-h2.mjs` 按新行号重切片（c1：1489-1557；c2：1559-2631；cap 段扩为 1581-1789，含结构身份块；
  `runScheme` 新增 `kinds` 透传），边界断言全过。
- 浏览器实机（真实管线 + 真实分词器 + 面板为准，夹具 `StarTest - fixture`）：
  临时条目 95003（`pos4/depth2/role=user`，正文引用聊天原文 `（用户第 5 句）` 9 字）+ 同 depth+role 的
  1160 字符第三方注入（`setExtensionPrompt`）→ **世界书 1577 / 聊天 807**（修复前同场景 367 / 2017；
  条目由「未匹配」变为 1210 t）；清理后基线与文档一致（总 3211 / 聊天 147 / 世界书 367 / 其他 2697）。
  S6 反向的**实机**对照未跑通（构造用的 EJS 楼层渲染后内容为空 → 被 `getChat()` 当空消息跳过，属装置问题）：
  该方向由 `probe6` §B/C/D 的离线断言覆盖。原先同时登记的那条「待定位观察」**已查清（无新 bug）**：
  ① 「楼层消失」= 上述空消息跳过（`openai.js:4125` 只保留有 `content`/`tool_calls` 的项）；
  ② 条目「未匹配」= 条目正文里的 9 字聊天引用触发了兜底判据 `slotIsChatFloor` → 整个注入槽被判「聊天」→
  已命中的命中量不分摊给世界书（**即 P2-4**；结构身份启用后同场景 `wi=496 / chat=290`，异常消失）。
  装置：`scratch/p47-repro.mjs`（离线最小复现 + 二分）+ 打桩副本；详见 AGENTS.md §四-7。

**残差（登记，未改）**

- 结构预测依赖"块的条数与顺序"：若第三方直接写 `extension_prompts`（depth/role 非有限数）或
  某个 IN_CHAT 值经宏替换后变空 → 预测偏移，但自检（maxN ≠ 合并长度）会整体回退到旧行为（届时 P2-4 回归旧状）。
- 被预算裁掉的项不影响编号，但若**最靠下的一条块**恰好被裁掉，maxN ≠ 合并长度 → 同样回退（罕见）。
- 身份**不**处理"注入块里的第三方尾巴"的世界书/其他归属（仍由既有 `wi-merged` 明细分摊处理）。

**环境还原**：夹具 md5 `740d15ff…`、`StarTest - short` `472b8baa…`、`Eldoria.json` `cf8ac228…`、`StarTest.png` `aecf09be…`、
用户聊天 SHA `9EEB0E51…`（290908 B）、`settings.json` `e971c3e9…`（1057412 B）**逐字节一致**（含 Horae 时间戳差异）；
说明：窗口内 settings 只被写过 4 个**运行时字段**（star 扩展 `lastUpdateCheck`/`updateInfo.checkedAt`、全局变量
`LAST_SEND_TOKENS`/`LAST_SEND_CHARS`=7/10，后者是「模拟发送」副产物，与上一轮同源）→ 收尾已按 13:32 参考副本**回写**；
ST 再次保存设置时会自行重写这 4 个字段（正常）；测试窗口 14 份备份已清理
（**331 → 317 条目 = 316 文件 + 1 空子目录**；其中 `settings_*` 恰为 50 = `config.yaml backups.common.numberOfBackups`
上限，`util.js:657` 每写一份即删最旧一份，故总数差 1 属 ST 自身轮换，非文件丢失）；全量扫描
`PHANTOM-TEST`/`P47_X`/`audit_p24_tail` **0 命中**；页面回到用户真实角色/聊天。

### 2.24 第十三轮：统计页六项 UI/交互修复（列表末项 / 上下文顺序标题 / 关闭图标 / 条目详情子窗重做 / 类型过滤改菜单 / 水波纹回归）（本轮新增；已实施并验证）

**① `#tub-row-entries` 最后一项无法完整显示（根因＝嵌套滚动容器 + 禁止滚动链）**

- 根因：统计页的列表是**内层**滚动容器（`#token-stats-panel .tub-row-3 { max-height:300px; overflow-y:auto }`），
  外面还套着同样可滚动的 `.popup-tabs-content`（`max-height:470px`），且两者的滚动条都被 CSS 隐藏（`scrollbar-width:none`）。
  内层原先写了 `overscroll-behavior: contain` ⇒ 鼠标一旦停在列表上，滚轮只滚列表、**不再把滚动权交回外层**；
  外层于是停在半途，列表下半截（含最后一项）永远卡在可视区之外，而用户既滚不动也看不到滚动条提示。
  实测（自动化页面）：外层 `scrollTop=60` 时列表底部比可视区下沿低 **230px**，最后一项完全在可视区外。
- 修复（`style.css`）：`contain` → `auto`（允许滚动链在列表到底后继续推动外层），并加注释说明**不能**再用 contain。
- 验证：终态（外层到底 + 列表到底）下"列表完整落在可视区 + 最后一项完整可见"（其下沿距可视区下沿 8px = 行自身 `margin-bottom`）；
  视口 1536×674 与 1200×500 两档均通过。⚠️ 真实滚轮 A/B 无法在自动化页面里跑（该页 `visibilityState=hidden`，
  `page.mouse.wheel` 不生效），故"滚轮能带动外层"这条属 CSS 语义 + 终态可达性验证，需用户用一次滚轮目视确认。

**② 「上下文顺序」分组标题改用聊天里的 mesIDDisplay 序号**

- 原状：标题 = `在聊天 #2 之前插入（用户消息 · User）`，与左侧 `#2` 徽标重复，且读不出"#2 就是聊天里那条消息的编号"。
- 实现（`index.js`）：
  - `entryContextAnchor()` 深度分支新增结构字段 `{ mesid, who, name }`（`mesid` = 该楼层在 `chat` 数组里的下标——
    与 ST `addOneMessage` 写进 `.mesIDDisplay` 的 `` `#${messageId}` `` 同源），label 改为 `在此消息之前插入（…）`；
  - 新增纯函数 `contextAnchorView(anchor)`：统一产出 `{badge,title,tooltip}`。深度注入 → 徽标 `#N`（与聊天界面**逐字一致**）
    + 标题 `用户消息 · User` / `角色消息 · 名字`；非楼层位置（前置/后置/作者注释/示例消息/出口/末尾/顶部）→ 固定文案；
    **旧锚点**（没有结构字段）用正则从 label 回退解析出 `#N` 与归属，不会显示空白；
  - 顺带修正旧文案口误：后置世界书原写成"位于全部聊天消息**之前**"→ 现按 `kind` 显示"**之后**"（旧锚点也一并纠正）；
    ⚠️ **这一句判断是错的**：经查 ST `promptManagerDefaultPromptOrder`，`worldInfoAfter`(后置) 同样在 `chatHistory`
    **之前**，原来的"之前"文案才是对的 —— 已在 §2.24-R2 更正（排序与序号口径一并改回）。
  - 深度解析口径对齐 ST（`world-info.js:5204`：`entry.depth ?? DEFAULT_DEPTH`）：`depth` 缺省按 4 算插入点
    （原先按 0 处理，会把这类条目错误地归进"末尾"组）。
- 口径说明：序号放在徽标里（`#N`，与 mesIDDisplay 逐字一致），标题留给"是哪条消息"。若还想让**标题本身**只剩 `#N`，
  只改 `contextAnchorView` 一处即可。
- 验证：实机分组标题 = `前置` / `后置（之后）` / `#2 用户消息 · User`（tooltip = 插到聊天消息 #2 之前）/ `末尾` / `未匹配`；
  其中 `#2` 与 `chat[2]`（`is_user=true`、`name='User'`）一一对应；`probe7` 覆盖新旧两种锚点与 0/1/2/3/5/6/7/未知位置。

**③ 关闭图标被「使用说明」标签页覆盖**

- 根因：关闭按钮是 `position:absolute` 且 `z-index:auto`，标签导航排在它之后（同为定位元素）⇒ 最后一格标签的盒子压住
  按钮右半部分（实测标签右沿 952 > 按钮左沿 933），点上去命中的是标签而不是关闭按钮。
- 修复（`style.css`）：关闭图标 `z-index: 6`；导航右侧留出按钮的位置 `padding: 0 44px 0 15px`。
- 验证：四个标签页下 `document.elementFromPoint(按钮中心) === 关闭按钮`；标签右沿 901 与按钮左沿 933 之间留 32px；
  四格标签均未折行；滑动下划线定位不受影响；点击按钮弹窗正常关闭。

**④ 条目详情子窗按新规格重做（`index.js` + `style.css`）**

- 条目名按类型着色：红 = EJS / 蓝 = 常驻 / 绿 = 动态（复用 `--hh-stats-color-*`）；
- 类型徽标 `hide-entry-type` 从 body 首行移到**子窗头部 `.hide-entry-sub` 最右侧**（flex `space-between`，右对齐）；
- 文案：`主关键词` → `关键词`；`Order` → `顺序`；
- `Position` + `Depth` 合并为一行 `插入位置`（新增纯函数 `entryInsertPositionText`）：@D → `系统 @D 3` / `用户 @D 3` /
  `AI @D 3`（角色口径同 ST `extension_prompt_roles`，深度同 `?? DEFAULT_DEPTH`）；其余 → `角色定义前` / `角色定义后` /
  `作者注释前` / `作者注释后` / `示例消息前` / `示例消息后` / `锚点`；
- `递归` 行 = 两个**只读勾选项**（`不可递归` / `防止进一步递归`）：勾选（该标志已生效）→ 绿色 + 实心框，
  未勾选 → 红色 + 空心框。纯展示：子窗本来就是只读视图，本轮的"点一下直接改世界书"未在范围内（已在报告里注明可再加）；
- 去掉「内容（content）」整块，`UID` 单独一行放到它原来的位置（原行里的 `N tokens` 一并撤掉——那是**该条内容**的体积）；
- 右侧滚动条：`::-webkit-scrollbar-button { display:none }` **去掉首尾倒三角**；默认整条隐藏（轨道 + 滑块透明）；
  显色三档由 JS 状态类驱动（`.hh-sb-scrolling` 滚动中 = 浅、`.hh-sb-hover` 鼠标在滚动条上 = 深一档、
  `.hh-sb-dragging` 按住 = 最深），且标准属性（`scrollbar-color`）与 WebKit 伪元素**同步取值**，
  避开 Chrome 121+ 里 `scrollbar-color` 与 `::-webkit-scrollbar-thumb` 的优先级差异
  （三档变量：`--hh-scrollbar-thumb-faint` / `--hh-scrollbar-thumb` / `--hh-scrollbar-thumb-hover`，亮暗各一套）。
- 验证：三种类型条目的名称色 = `#f87171` / `#60a5fa` / `#4ade80`；徽标确实落在副标题行最右（其右沿比书名右沿多 253px）；
  行序 = `关键词 / 插入位置 / 顺序 / 递归 / UID`，`内容` 块与旧的首行徽标 `hide-entry-badges` 均不存在；
  `uid=519`（两项递归标志皆 false）→ 两个**红色未勾选**项，`uid=350`（皆 true）→ **绿色已勾选**项；
  滚动条三态实测 `transparent → rgba(255,255,255,0.2) → 0.28 → 0.45`（暗主题），`scrollbar-button: none`。

**⑤ 类型过滤：下拉框改「左下角菜单」，且只列实际存在的类型**

- 触发按钮沿用原 id（`#tub-type-filter` / `#tub-cmp-type-filter`）；点开的是挂在**外层弹窗**直下的浮层菜单
  （不随内容区滚动、不被裁切），位置默认落在**触发器的左下角**（左边缘对齐、向下展开），越界时自动内收 / 上翻；
  再点触发器收起，点别处 / ESC / 滚动 / 缩放都自动关闭；`closeTypeFilterMenu()` 也会在整块重绘、切页、关弹窗时调用。
- 菜单项只列「当前数据里真实存在的类型」：明细视图按当前世界书过滤后的 `statsObj` 统计；对比视图按当前世界书过滤后
  **实际有差异**的类型统计。当前选中类型在切换世界书后不再存在时自动回退「所有」，避免"选了 EJS 却一条都列不出来"。
- 定位必须做**缩放换算**：弹窗的 `hhFadeIn` 入场动画带 `scale(0.95)`，动画未结束时 `getBoundingClientRect` 小于布局尺寸
  （自动化 / 后台页面里文档时间轴冻结，实测常驻 0.95）；直接用 rect 差值会把触发器位置算小 5%，菜单歪十几像素并压在触发器上。
  现按"视觉长度 ÷ 缩放比 = 布局长度"换算（与 `positionEntryPopup` 的 layoutBox 同源）。
- 验证：明细视图菜单项 = `所有 / EJS / 动态 / 常量`（该数据三种都有）；切到只有 1 条常驻条目的 `Eldoria` 后菜单项自动变为
  `所有 / 常量`（同时把原 EJS 选择回退）；对比视图（无动态差异）菜单项 = `所有 / EJS / 常量`，选 EJS 后差异行 18 → 10、
  复位回 18；菜单位置实测均在弹窗内、不与触发器重叠（触发器在弹窗最底一排时自动上翻）。

**⑥ 主题切换水波纹（回归修复）**

- 根因：`transition.ready` 里对 `::view-transition-new(hh-theme-surface)` 只给了**一个**关键帧
  `clipPath: ['circle(0px …)']`；浏览器会把缺失的另一端补成元素自身的底层值 `clip-path: none`（= 新快照整块可见），
  动画于是变成"从整块收缩到左下角一点"——视觉上就是"水波纹丢了"。实测该动画 t≈30ms 为 `none`、t≈230ms 已到 `circle(0px)`。
- 修复：补齐起止两帧 `[circle(0px at 左下角), circle(半径 at 左下角)]`，半径仍为 `hypot(w,h)+2`（终点正好铺满右上角），
  时长 620ms、缓动 `cubic-bezier(0.4,0.2,0.2,1)`、`fill: forwards`、伪元素不变。
- 验证：用桩替换 `document.startViewTransition` 让该代码路径真正跑一遍，捕获到
  `clipPath: ['circle(0px at 0px 523.83px)', 'circle(660.48px at 0px 523.83px)']`（半径随弹窗尺寸自算）与
  `duration 620 / easing / fill forwards / ::view-transition-new(hh-theme-surface)` 全部正确；动画结束后
  `hh-theme-switching` 与 `hh-ripple-surface` 均被清理、按钮文案同步。
  ⚠️ 自动化页面 `visibilityState=hidden`（文档时间轴冻结、rAF 不触发）**无法目视确认播放效果**——需用户在真实浏览器点一次确认手感（同 §2.2 的注意事项）。

**离线探针**：`scratch/probe7.test.mjs`（从 `index.js` 抽取真实函数独立运行）**69 断言全通过**，覆盖
`entryContextAnchor`（depth 缺省 / 数字字符串 / 超界 / 各 position）、`contextAnchorView`（新旧锚点、口误纠正、空值安全）、
`entryInsertPositionText`（8 种位置 + 角色/深度边界）、`typeFilterOptionsOf` / `availableDiffTypes`（可用类型过滤与顺序）。
另：`node --check index.js` 通过、`style.css` 花括号 313/313 配平；浏览器端到端另跑了"打开弹窗 → 四页签轮询 →
动画遮挡命中测试 → 明细/对比切换 → 世界书过滤切换 → 搜索/排序 → 条目详情（三种类型 + 递归两态）→ 菜单四类交互"。

**环境还原**：`settings.json` 与会话前的自动备份**逐字段 0 差异**（测试期把主题切到过 dark，已用真实切换还原 light，
`logLevel=0`、`last_active_tab=token-stats-panel` 与会话前一致）；聊天文件仅差 `chat_metadata.timestamp.absolute`
（其它扩展每次刷新写入的运行时字段，内容 0 差异）；世界书 `WuWa World Info MVU.json` md5 `5B7CEA5E…` 刷新前后不变；
备份目录已清掉本会话窗口内产生的 15 份自动备份（325 → 310；`settings_*` 49/50 属 ST 自身轮换，非文件丢失）。

**2.24-R1 复查修正（用户第二轮反馈，四处）**

上一轮把三处理解错了，按用户口头纠正重做（全部已实机复验）：

1. **④ 条目详情：内容（content）不该删；「UID / tokens」是"左 UID、右 tokens"**
   （原文"将'内容（content）'去除"指的是**去掉那行左侧的标签文字**，不是去掉内容块——上一轮理解反了）。
   现在：`关键词 / (次关键词) / 插入位置 / 顺序 / 递归` 之后是**一行两列**：
   `<span class="hide-entry-uid">UID 519</span>`（左） + `<span class="hide-entry-tokens">500 tokens</span>`（右，
   `.hide-entry-meta-row { justify-content: space-between }`），紧接着是**恢复的 `.hide-entry-content` 内容块**
   （等宽、pre-wrap、带边框，样式与 §2.13 原版一致）。tokens 口径恢复为 `cachedTokenCount(entry.content)`
   （条目自身体积，与列表里"本次命中多少 token"不同）。
   实机：`UID 0` 左边缘 = 行左边缘、`500 tokens` 右边缘 = 行右边缘；内容块紧随其后（733 字符，`<CONCEPT_The_Lament>…`）。

2. **② 上下文顺序标题：只留 messageID（不再有"世界书·前置（位于全部聊天消息之前）"这类描述）**
   - 口径：凡能定位到聊天消息的分组都显示该消息的 mesIDDisplay 序号；"位于全部聊天消息之前"的块
     （前置 / 顶部 / 作者注释 / 示例消息）用**第一条待发送消息**的序号，"位于全部聊天消息之后"的块（后置 / 末尾）
     用**最后一条待发送消息**的序号；深度注入用它自己的插入点。`tub-ctx-msg-main` 里**只有 `#N`**，
     解释性文字（在哪条消息之前/之后、消息归属）移到行的 `title`（悬停可见）。
   - 实现：`entryContextAnchor` 给所有非深度位置也带上 `mesid`（首/末条），`contextAnchorView(anchor, edges)`
     统一产出 `#N`；`edges`（首/末条序号）由新函数 `contextSendableEdges(model)` 从**当前聊天**推出，
     因此**旧锚点**（没有结构字段）也能立刻显示序号，不必等下一次发送；`model` 增加 `genType` 以便 swipe 口径一致。
   - 例外（用户说的"前面后面都无世界书条目"）：确实定不出序号的分组保留原文案——未匹配条目组、出口条
     （落点由 `{{outlet::}}` 宏决定）、更早的旧格式锚点。
   - 实机（用户真实聊天，旧锚点）：分组显示 `#0`（前置，tooltip=世界书·前置…对应 #0）/ `#5`（后置）/
     `#2`（深度注入，tooltip 含"插到聊天消息 #2 之前（用户消息 · User）"）/ `#5`（末尾）/ `未匹配`（无 title）。
     `chat[0]`/`chat[5]` 与序号一一对应；所有分组 `tub-ctx-msg-title` 元素**不再渲染**。

3. **⑤ 类型过滤按钮去掉倒三角**：`#tub-type-filter` 与 `#tub-cmp-type-filter` 两处触发器都删掉
   `<span class="tub-type-filter-caret">`，`.tub-type-filter-caret` 规则一并删除（触发按钮现在只有文字）。
   实机：两个按钮的 `outerHTML` 里无 caret 元素，CSSOM 里 0 条 caret 规则；菜单项与选择行为不受影响
   （明细 `所有/EJS/动态/常量`、对比 `所有/EJS/常量`——后者无动态差异）。

**探针更新**：`scratch/probe7.test.mjs` 增至 **90 断言全通过**（新增：非深度位置的 `mesid` 首/末条口径、
空聊天 → `mesid=null`、`contextAnchorView` 只输出 `#N` 且 `title` 为空、旧锚点用 `edges` 兜底、
出口/未匹配/无 edges 时回退原文案、自带 `mesid` 优先于 `edges`）。

**2.24-R2 复查修正（用户第三轮反馈，两处）**

**① 「上下文顺序」的分组排序错了**（用户指出的实机现象：`#0 → #5 → #2 → #5`，`#2` 被夹在两个 `#5` 之间）

- 复现（用户真实聊天、选 `#5 · 17:06:50` 锚点）：分组顺序 = `#0 前置` → `#5 后置` → `#2 深度注入` → `#5 末尾`。
- 根因（两个错误叠加）：
  1. **排序用的是锚点里存下的 `key`**。旧方案的键值 = 前置 0 / **后置 1** / 作者注释 50,51 / 对话示例 500,501 /
     深度 100+b / 出口 510 / 其他 900 —— 后置的 1 小于深度槽的 102/106，于是"后置"被排到深度注入之前。
     用户这份锚点是**旧格式**（没有 `mesid` 字段、key 仍是旧值），所以显示层必须自己重推顺序，不能信旧 key。
  2. **我把"后置"错当成"位于全部聊天消息之后"**（上一轮还把原本文案里的"之前"当笔误"纠正"成了"之后"）。
     查 ST 源码：`PromptManager.js` 的 `promptManagerDefaultPromptOrder` = `main` → **`worldInfoBefore`(前置)** →
     persona/charDescription/charPersonality/scenario/nsfw → **`worldInfoAfter`(后置)** → `dialogueExamples`(对话示例) →
     **`chatHistory`** → `jailbreak`；而 `public/script.js:4635-4666` 把 `worldInfoExamples`(EM 条目) 塞进示例消息数组
     （= `dialogueExamples` 槽）。⇒ **前置、后置、作者注释、对话示例**全都位于 **chatHistory 之前**，位置 1 的语义是
     "角色定义之后（↓Char）"，与聊天楼层无关。原插件的"之前"文案本来就是对的。
- 修复：
  - `entryContextAnchor` 的键值改为**与最终上下文一致**的方案：系统区 `前置 0 / 后置 10 / 作者注释 20,21 / 对话示例 30,31`；
    与聊天交错的深度注入 `1000 + b`；位置未知的出口 `90000`、其他 `95000`；
    `mesid` 口径同步：前置/后置/作者注释/对话示例/顶部注入 = **第一条待发送消息**，深度注入 = 自己的插入点，末尾 = 最后一条。
  - 新增纯函数 **`contextAnchorMesid(anchor, edges)`**（序号推导）与 **`contextAnchorOrderKey(anchor, mesid)`**（显示顺序键），
    `groupEntriesByAnchor(entries, edges)` 与 `renderEntriesList` 的条目级 `ctxKey` **一律改用这两个函数**——
    **不再读取锚点里存下的旧 `key`**，所以旧锚点在界面上立刻按新顺序显示（无需重新发送消息）。
- 验证（实机、旧锚点、用户点名的 `#5 · 17:06:50`）：顺序变为 **`#0（前置）→ #0（后置）→ #2（深度注入）→ #5（末尾）→ 未匹配`**，
  即 `#2` 回到两个 `#5` 之前；两个 `#0` 的区别挂在 tooltip（`世界书·前置（↑Char）…` / `世界书·后置（↓Char，角色定义之后）…`）。
  面板其余部分不受影响（概览 31826 = 聊天 1958 + 世界书 26870 + 其他 2998、条目合计 26870 与改前逐项一致）。

**② `hide-entry-checks` 两个递归选项的颜色统一**（用户要求：不再绿/红）

- `.hide-entry-check-on/off` 的文字改成正文色（亮色主题 `rgb(52,58,64)`，即"黑"；暗色主题走 `--hh-text-primary`）；
  勾选框同样中性：透明底 + `1px var(--hh-text-secondary)` 边框，勾选态只把边框加深一档，✓ 用正文色绘制。
  勾/未勾完全由"有没有 ✓"区分，不再有颜色语义。
- 实机：已勾选（`uid=350`）与未勾选（`uid=519`）两条条目的文字色均为 `rgb(52, 58, 64)`，未勾选仅有灰边框（`rgb(134,142,150)`）与无 ✓ 的差别。

**探针**：`scratch/probe7.test.mjs` 增至 **107 断言全通过**，新增用例即本次两个 bug 的回归：
`wi-after` 的 `mesid` 必须是首条（不是末条）、键值方案（0/10/20/21/30/31/1000+b/90000/95000）、
以及"用旧锚点的 key 组合（0/1/102/106）排序后必须得到 `前置 → 后置 → 深度#2 → 末尾`"。

**2.24-R3 复查修正（用户第四轮反馈，两处）**

**① 「上下文顺序」改为"messageID 位置标记"制：同一 messageID 只出现一次，条目按前后排在标记行两侧**

- 用户口径（原文）："列表中的 messageID 有且只应该出现一次…`#0` 后面有七个条目，这七条在上下文顺序中算是 `#0`
  **前面**的，你干嘛要放到它后面？"
- 事实核对（**用用户自己的配置**，`settings.json → oai_settings.prompt_order`，与 ST 默认
  `promptManagerDefaultPromptOrder` 同序）：
  `main` → `worldInfoBefore` → `charDescription` → `charPersonality` → `scenario` → `enhanceDefinitions` →
  `nsfw` → **`worldInfoAfter`** → `dialogueExamples` → **`chatHistory`** → `jailbreak`。
  ⇒ **前置（↑Char）与后置（↓Char）都在 `chatHistory` **之前****，它们的"对应消息"都是第一条待发送消息；
  因此**不可能给出两个不同的 messageID**（用户希望的"用两个 messageID 区分前置/后置"与事实冲突）。
  处理方式：两者共用 `#0` 这一个标记行（同一 id 只出现一次），各自的说明保留在标记行 `title` 里；两段条目
  在标记行上方各自成段（前置段、后置段）仍可分辨。若用户更想按"名字直觉"把后置显示成聊天之后（`#5`），
  只改 `CTX_ANCHOR_BEFORE_KINDS` / `CTX_ANCHOR_AFTER_KINDS` 两处即可（已在报告里说明）。
  ⇒ **前置（↑Char）与后置（↓Char）都在 `chatHistory` 之前**（后一句更正 R1 的错误判断），它们的"对应消息"都是第一条待发送消息；
- 实现：
  - `groupEntriesByAnchor` 为每组补 `mesid`（对应哪条聊天消息）与 `side`（`'before'` = 条目位于该消息之前 /
    `'after'` = 之后；`depth-end`（末尾/深度 0）与"定不出序号"的老路径为 `after`）。
  - `renderEntriesList`（上下文顺序）改为"位置块"渲染：把同一 `mesid` 的相邻分组收拢成一个块，
    **先渲染 `side==='before'` 的条目 → 再渲染 `#N` 标记行（同一条消息只出现一次）→ 再渲染 `side==='after'` 的条目**；
    标记行右侧数字 = 挂在该标记上的条目总数；多个来源共用标记时，`title` 里用"；"拼接各自的说明。
- 实机（用户真实聊天，旧锚点、`#5 · 17:06:50`）：渲染顺序 =
  `20 条（前置 7 + 后置 13）→【#0·20】→ eldoria →【#2·1】→【#5·11】→ 11 条（末尾）→【未匹配·8】→ 8 条`
  —— 每个 messageID 只出现一次；`#0` 的 20 条全部排在 `#0` **上方**（即"在 #0 之前"），末尾 11 条排在 `#5` 下方。
  `#0` 的 tooltip = "世界书·前置（↑Char）：位于全部聊天消息之前（对应聊天消息 #0）；世界书·后置（↓Char，角色定义之后）：位于全部聊天消息之前（对应聊天消息 #0）"。

**② 递归勾选框：勾选态改为黑底白钩**

- `.hide-entry-checkbox-on` 改为 `background: var(--hh-text-primary)` + `color: var(--hh-color-white)`（亮色主题即黑底白钩）；
  未勾选仍是透明底 + 灰边框 + 无钩。实机：勾选态 `bg=rgb(52,58,64)`、`✓` 为 `rgb(255,255,255)`、边框同色。

**探针**：`scratch/probe7.test.mjs` 增至 **113 断言全通过**（新增 `groupEntriesByAnchor` 用例：旧锚点分组的
`mesid`/`side`/顺序——`wi-before@0/before , wi-after@0/before , depth@2/before , depth-end@5/after`）。

**2.24-R4 复查修正（用户第五轮反馈）：注入点**两侧**的 messageID 都要出现**
**2.24-R4 复查修正（用户第五轮反馈）：注入点两侧的 messageID 都要出现**
- 用户口径（原文）："我记得我有说过 messageID 显示的标准吧：**除非该 messageID 相邻的前面和后面都没有世界书条目，
  否则一律显示该 messageID**。"（并追问 `·eldoria` 到底在 `#1–#2` 之间还是 `#0–#2` 之间，若在 `#1–#2` 之间为何不显示 `#1`）
- 事实：该条 `eldoria` 是深度注入（缺省 depth=4 → b = n−4 = 2，n=6）⇒ 它落在**发送列表第 2 条之前**，
  也就是 **`#1` 与 `#2` 之间**（不是 `#0–#2`）。R3 只渲染了"它插入到哪条之前"（`#2`），漏掉了**紧邻它前面**的 `#1`，
  于是读者无法判断它夹在哪两条消息之间 —— 与用户的显示标准不符。
- 修复：
  - `contextSendableEdges(model)` 追加返回 `prev`（`Map<floor, 前一条 floor|null>`）；
  - `groupEntriesByAnchor` 为 `side==='before'` 的分组补 `predMesid`（注入点前面那条消息；首条/未知 → null）；
  - `renderEntriesList` 的"位置块"渲染改为：`emitMarker(predMesid)`（若首次出现）→ 条目行 → `emitMarker(mesid)`；
    标记用 `Set` 去重，**每个 messageID 全列表只出现一次**；没有条目归属它的纯位置标记（如 `#1`）只显示 `#N`、
    不带数字，tooltip = "聊天消息 #N（位置标记：它后面紧邻着世界书注入）"。
- 实机（用户真实聊天、`#5 · 17:06:50`）新序列：
  `[20 条（前置7+后置13）] →【#0·20】→【#1】→ ·eldoria →【#2·1】→【#5·11】→ [11 条（末尾）] →【未匹配·8】→ [8 条]`
  —— `eldoria` 现在明确夹在 `#1` 与 `#2` 之间，且 `#1` 有标记；`#0/#1/#2/#5` 各只出现一次，`#2` 仍按你之前的
  要求排在 `#5` 之前。

**探针**：`scratch/probe7.test.mjs` 增至 **119 断言全通过**（新增 `predMesid` 用例：深度注入在 `#2` 之前 →
`predMesid=1`；首条 → `null`；末尾组 → `null`；缺 `prev` 映射时安全退化为 `null`）。

**2.24-R5 复查修正（用户第六轮反馈）：标记行右侧由"条目数"改为该消息的 role（`AI` / `user`）**

- 用户口径：「对于对应的 messageID 我希望把 `tub-ctx-msg-tokens` 改为 message 的 role（AI/user）」。
- 实现：`contextSendableEdges()` 追加返回 `roleOf`（`Map<floor, 'user'|'AI'>`，取自 `computeSendableChat` 的
  `isUser`）；`ctxMarkerHtml` 里凡是**能对应到聊天消息**的标记行（`mesid !== null` 且 role 可得）右侧改渲染
  `<span class="tub-ctx-msg-role">user|AI</span>`（新增样式：与原来的数字同尺寸/同次要色，`letter-spacing:0.02em`）。
  **定不出序号的块**（未匹配、出口、更早的旧格式）没有消息可对应 → 仍按原样显示归属它的条目数
  （所以「未匹配」行右侧还是 `8`）。
- 实机（用户真实聊天）：标记行 = `【#0·AI】【#1·AI】【#2·user】【#5·AI】【未匹配·8】`，
  与 `chat` 的 `0:AI 1:AI 2:user 3:AI 4:user 5:AI` 逐项一致（#2 正是那条用户消息）；
  条目行数量不变：默认排序 40 条、上下文顺序 45 行（40 条目 + 5 标记行）。
- 说明：本轮只改**显示层**，`scratch/probe7.test.mjs` 仍是 **119 断言全通过**（`roleOf` 依赖 `getContext`，不进探针；
  已用浏览器端到端复验：标记行 role、默认排序、条目详情（关键词/插入位置/顺序/递归黑底白钩/UID·tokens/内容块）、
  对比视图 31 行表格 / 18 行差异）。

### 2.25 第十四轮：「聊天」统计子弹窗（实际发送范围 / AI 平均 tokens / 生成速度 / 最近十次折线）（本轮新增；已实施并浏览器端到端验证）

**需求（用户）**：鼠标点击「聊天」`tub-stat-box` 弹出一个小子弹窗（参考世界书条目详情子窗的实现方式），显示
① 实际发送的 messageID 范围；② 发送上下文里 role=AI 的平均 tokens；③ 生成速度（tokens/耗时）+ 最近十条数据的折线图。

**实现**：

- **数据（`index.js` · `computeRenderedStats`）**：新增两个**体积很小**的持久化字段（随锚点写入 `message.extra`）：
  - `sentChat = { first, last, count }`：本次**实际发送**的聊天楼层范围（首/末条 mesIDDisplay 序号 + 条数），
    直接复用已有的 `computeSendableChat(chatSnapshot, genType)`（隐藏楼层/忽略符号已排除，swipe 已排除末条）——
    因此这与"实际发送"口径**完全一致**，不是重新按 role 猜。
  - `aiTokens = { avg, count, sum }`：发送上下文里 `role==='assistant'` **且是真实聊天楼层**（`slotFloorFlags[i]===true`）
    的消息 token 均值；深度注入（role 也可能是 assistant）不计入。
- **UI（`updateTokenStatsUI` → `renderTokenStatsContent`）**：把完整 `selected.stats` 一并透传进渲染模型（新增 `model.stats`）。
  概览行「聊天」框加 `tub-stat-box tub-stat-box-clickable` + `data-stat="chat"`，点击 `openChatStatsPopup`。
- **子弹窗（新增一组函数）**：`ensureChatStatsPopup` / `openChatStatsPopup` / `closeChatStatsPopup` / `toggleChatStatsPopup` /
  `positionChatStatsPopup` / `pickReplyForAnchor` / `replyOutputTokens` / `replySeconds` / `collectGenerationHistory` / `renderSparkline`。
  - 形态：**贴框小浮层**（`position:fixed`，宽 300，默认落在「聊天」框正下方，越界自动内收/上翻），
    与条目详情子窗的"整窗并排"刻意区分（用户要的是"小的子弹窗"）。
  - 行：实际发送范围（`#first → #last` + 条数）、AI 平均 tokens（+ 条数）、本次生成速度、最近 ≤10 次生成折线（SVG 折线 + 圆点，点悬停显示该次 tokens 与 t/s，标题带窗口内平均 t/s）。
  - **速度/折线不新增持久化**：tokens 取回复消息自身的 `extra.token_count`（ST 原生，与消息计时同源；缺失时回退按正文分词），
    耗时取 `gen_started` / `gen_finished`（ST 原生）——因此刷新页面或切换聊天后折线仍能从聊天历史**立即重建**。
    本次生成速度对应 `pickReplyForAnchor(chat, model.index)`：normal → 锚点之后那条 AI 回复；swipe/regenerate/continue →
    锚点自身；模拟锚定 → 最新一条 AI 回复。
  - **旧锚点兜底**：无 `sentChat` 时用 `contextSendableEdges(model)` 由当前聊天推首/末条序号（标注"旧数据（按当前聊天推算）"）；
    无 `aiTokens` 时显示占位"旧数据无此统计"，不报错。
- **生命周期**：进入 `updateTokenStatsUI` 先 `closeChatStatsPopup()`（内容区整体重绘会替换锚点框，避免无主浮层）；
  `CHAT_CHANGED` 时关闭；`applyTheme` 的主题目标加入该子窗（跟随亮/暗）；点外部 `mousedown` 或 `Esc` 关闭；
  再点同一框 = 收起。
- **样式（`style.css`）**：新增 `#hide-helper-chat-stats-popup` 一套（标题/行/折线/关闭按钮，复用 `--hh-*` 变量与 `hhFadeIn`）；
  暗色选择器加入该 id；`.tub-stat-box-clickable` 悬停描边上浮，提示"可点击"。

**验证（实机 `127.0.0.1:8000`，真实聊天 + 一条旧格式锚点）**：

- 离线：`node --check` 通过；DOM shim 端到端复跑渲染路径（新/旧锚点、toggle/close、折线平整数据无 NaN）共 **33 断言全通过**（临时脚本，已清理）。
- 浏览器端到端：点击「聊天」框 → 浮层贴框显示，内容 = `实际发送范围 #0 → #5（旧数据按当前聊天推算）/ AI 平均 tokens —（旧数据无此统计）/
  本次生成速度 120.2 t/s（2909 tokens · 24.2s）/ 最近 3 次生成（输出 tokens）· 平均 106.8 t/s + 折线`；
  注入新格式字段后重渲染 = `#1 → #5 共 3 条 / 1234 共 3 条 AI 消息 / …`（新格式分支正确）；
  再点收起、点外部关闭、`Esc` 关闭均通过；暗色主题下浮层配色正确。测试后已还原亮色主题、关闭弹窗、删除临时注入字段。

### 2.26 第十五轮：子弹窗口径重做（AI 平均速度 + 双折线 + 最新七条 AI）× 锚定改落在"生成的那条消息" × 完整上下文内存快照 × 预算预警（本轮新增；已实施并浏览器端到端验证）

**需求（用户四点）**：① 子弹窗要的是 **AI 平均生成速度**（不是"本次生成速度"），折线改两条——输出 tokens（折线一）与生成速度（折线二，
两者用**不同配色**）；AI 平均 tokens / AI 平均速度 / 折线数据**一律取当前聊天最新的七条 AI messageID**（_不要求_是实际发送的那些）；
「实际发送范围」仍然必须用**实际发送**的 messageID。② 锚定数据应存在**生成的那条 messageID** 上（存触发的那条会让人疑惑）；
生成 swipe 时也要更新该 swipe 对应 messageID 的锚定。③ 完整上下文**存内存**（刷新/切聊天即丢）即可 —— 正因如此才不必担心聊天文件膨胀，
且可在当前活跃聊天里查看分析。④ 预算预警：总量 ≥ 200000 时「总共」框标红。

**实现**：

- **子弹窗口径重做（`index.js`）**：新增 `collectLatestAiMessages(chat, AI_HISTORY_COUNT=7)` —— 从聊天**尾部**倒着取最新 7 条
  AI 消息（不看是否发送过）。AI 平均 tokens = 这批消息的 token 均值；AI 平均生成速度 = 这批里"有耗时记录"的那些的 t/s 均值；
  `renderSparkline(points, {tone})` 泛化为双折线：`tone='tokens'`（绿，`--hh-stats-color-dynamic`）与 `tone='speed'`（蓝，
  `--hh-stats-color-constant`），并跳过 `null`/非有限值（无耗时的那条只出现在 tokens 折线里，悬停提示"无耗时记录"）。
  token 缺失（未开"消息 token 计数"）时按正文分词补算。**删掉了按锚点定位的 `pickReplyForAnchor` 与旧 `collectGenerationHistory`**
  ——那套是"本次生成"口径，已被本轮的"最新 N 条 AI"取代。「实际发送范围」仍读锚点 `sentChat`（真实发送口径），旧锚点由
  `contextSendableEdges` 兜底。`stats.aiTokens` 字段因此**不再写入**（`computeRenderedStats` 里只保留 `sentChat`），
  AI 相关统计一律实时算 —— 也顺带保证它们**永不落盘**。
- **锚定改落在生成的消息上（`resolveAnchorTarget` + `pendingReplyAnchor`）**：
  - `swipe` / `regenerate` / `continue` → 被重生成/续写的就是 chat 末条消息本身 → 直接写它（`mode:'self'`）；
    swipe 靠 `writeAnchoredStats` 既有的 `swipe_info[*]` 镜像规则覆盖"该消息每个 swipe 各存一份"——正是用户要的"更新该 swipe 的锚定"。
  - `normal` → 生成的是**即将追加**的 AI 回复，统计完成时它可能还没到 → `mode:'reply'` 挂起（`pendingReplyAnchor`），
    在 `MESSAGE_RECEIVED` / `GENERATION_ENDED` / `GENERATION_STOPPED` 时补写到 `chat[triggerIndex+1]`；
    若回复始终没出现（被中止/失败）则**回退**写到触发消息上，保证统计不丢。另有 2s 间隔的超时兜底，
    **生成中（`body.dataset.generating==='true'`）跳过**——慢速回复可能几十秒，绝不能提前回退。
  - `impersonate` 等无消息产出 → 回退锚在触发消息上。
  - 切聊天时 `pendingReplyAnchor` 一并清空。
- **完整上下文内存快照（`contextDumpStore` / `saveContextDump` / `openContextViewer`）**：`computeRenderedStats` 结束前把
  **实际发送的每条消息**（序号/角色/tokens/来源槽位/正文）存进内存（`lastContextDump`），由调用方（写锚定 / 模拟锚定）连同
  键 `${chatId}|${anchorId}` 入库，LRU 保留 **3** 份。**刻意不落盘**：聊天文件不因它膨胀，刷新/切聊天即清空（`CHANGED` 时 `clear()`）。
  子弹窗底部出现「查看完整上下文（N 条）」按钮（仅当该锚点确有快照时），点开是右侧独立窗口：逐条列出
  `#i · role · 来源槽位 · tokens` + 正文摘要，点条目在摘要/全文之间切换。
- **预算预警**：`STATS_BUDGET_TOKENS = 200000`；`totalTokens >= 200000` 时「总共」框加 `tub-stat-box-over-budget`
  （红底 `--hh-stats-overbudget-bg`(亮/暗两套) + 红字 + 2px 红描边），副标题改「超出预算」，并挂 `title` 说明阈值。
- **样式（`style.css`）**：双折线配色（`.hcs-spark-line-tokens` / `-speed` 及对应圆点）、`.hcs-ctx-btn`、
  预算红框与两条新变量、完整上下文查看器整套（复用 `.hide-helper-popup` 基础类 + 头部固定/内容区滚动，与条目详情子窗同构）；
  暗色选择器加入 `#hide-helper-context-viewer`；`applyTheme` 主题目标同步加入。

**验证（实机 `127.0.0.1:8000` + 离线 DOM shim）**：

- `node --check` 通过。
- 离线断言 **60 条全通过**（三组临时脚本，已清理）：窗口取数（最新 7 条、升序、边界）、`resolveAnchorTarget` 五种生成类型分派、
  双折线 tone/null 跳过/无 NaN、`replySeconds`、弹窗渲染（两图块 + 两种配色 + 无 undefined/NaN）；
  锚定落点（回复已到→写回复、未到→保持挂起、结束仍无→回退写触发消息、swipe→写自身且 **swipe_info 全版本镜像**、
  切聊天→判废）、dump LRU 上限 3 与淘汰、预算阈值（199999 不标 / 200000 与 250000 标红）、查看器渲染（序号/角色/槽位/tokens/HTML 转义/空态）。
- 浏览器端到端：重载后点击「聊天」框 = `实际发送范围 #0 → #5 共 6 条`（来自真实发送的快照）+ `AI 平均 tokens 1759 最近 4 条 AI 消息`
  + `AI 平均生成速度 82.6 t/s` + **两条折线（绿=tokens、蓝=速度，8 个数据点）**；
  点「模拟发送」跑真实管线后按钮出现 `查看完整上下文（11 条）`，点开查看器 = `模拟发送 · 11 条消息 · 共 31445 tokens`，
  逐行 `#0 system 其他·main 30t` / `#1 system 世界书·前置·worldInfoBefore 2048t` / `#4 assistant 聊天·chatHistory-2` …
  来源槽位归因与 tokens 均正确；预算红框经类注入验证配色（红底/红字/红描边）。测试后已还原（关闭窗口、移除注入类、恢复字段）。

### 2.27 第十六轮：子窗尺寸统一 × 双折线坐标轴 × **世界书条目与 messageID 的真实上下文位置**（据 ST 源码厘清并修正）（本轮新增；已实施并浏览器端到端验证）

**需求（用户三点）**：① 「聊天」统计子窗改为**与主弹窗同尺寸**；「完整上下文」查看器改为**贴主窗右侧 + 与主弹窗同尺寸**（参照条目详情子窗）。
② 两条折线要显示**横纵坐标**；删掉「最近 N 条 AI 消息」「最近 N 条有耗时记录」两个说明文本。
③ 「已激活」的「上下文顺序」里 `worldInfoAfter` 排在了 `#0` **之后**（不对），应以「完整上下文」展示的顺序为准（`worldInfoAfter` 在 `#0` **之前**）——
并据此**彻底查清**世界书条目与聊天 messageID 的上下文位置关系。

**一、据 ST 源码厘清的上下文位置事实（本轮核心结论）**

调用链：`script.js:5227` `prepareOpenAIMessages()` → `openai.js:1185` `populateChatCompletion()`。
关键事实：`addToChatCompletion(source)` 里 `chatCompletion.add(collection, prompts.index(source))` ——
**最终位置由 `promptManager` 的提示词顺序决定**（`PromptManager.js:2083` `promptManagerDefaultPromptOrder`），
不是由代码里的调用次序决定。默认顺序与各世界书 position 的落点：

| position | 含义 | 在最终上下文中的位置（默认顺序下） |
|---|---|---|
| 0 | 世界书·前置（↑Char） | `main` 之后、`charDescription` 之前 → **在全部聊天消息之前** |
| 1 | 世界书·后置（↓Char） | 在 char/scenario/nsfw **之后**、`dialogueExamples`/`chatHistory` 之前 → **仍在全部聊天消息之前** |
| 2 / 3 | 作者注释 顶 / 底 | 并进 AN 文本；AN 落点由用户设置决定（见下） |
| 4 | @D 深度注入 | 插到发送列表第 `n − depth` 条之前（`world-info.js`：`entry.depth ?? DEFAULT_DEPTH`；越界 → 顶部/末尾） |
| 5 / 6 | 对话示例 顶 / 底 | 并进 `dialogueExamples` → **在全部聊天消息之前** |
| 7 | 出口 | 由 `{{outlet::名字}}` 宏出现位置决定，无法反推 |

- **证实用户观察正确**：默认顺序是 `main(0) → worldInfoBefore(1) → persona(2) → charDescription(3) → charPersonality(4) →
  scenario(5) → enhanceDefinitions(6) → nsfw(7) → worldInfoAfter(8) → dialogueExamples(9) → chatHistory(10) → jailbreak(11)`，
  ⇒ **前置与后置都在 `chatHistory` 之前**（后置只是"角色定义之后"，不是"聊天之后"）。
- **作者注释（AN）的落点**由 `chat_metadata.note_position` 决定（`authors-note.js:324` `setFloatingPrompt()`）：
  `0`(IN_PROMPT，主提示词后) / `2`(BEFORE_PROMPT，主提示词前) → 经 `openai.js:1305` `injectToMain()` 插进 **`main` 块内部**（`insert(message,'main',position)`）；
  `1`(IN_CHAT，注入深度 d) → 与 @D 同落点（第 `n−d` 条之前）。AN 为空（本次未插入）时不产生任何落点。
- **`newMainChat`**（`[Start a new Chat]`）由 `chatCompletion.insertAtStart(newChatMessage,'chatHistory')` 插入 `chatHistory` 块的**最前面**，
  即**紧邻全部聊天消息之前**（实机查看器的 `#3` 就是它，其下 `#4` 才是第一条聊天消息）。

**二、修复（用户报告的排序错误）**

- `entryContextAnchor(entry, sendable, anInfo)`：新增 `anInfo` 参数（发送时刻由 `captureAnInfo()` 捕获 AN 的 `{position, depth, role}`）；
  AN 顶/底 条目在 `position=1`（深度注入）时改走与 @D 相同的落点换算（新 kind `an-depth`），否则紧随主提示词（key `-20/-19`，在 `wi-before=0` 之前）。
- **显示层排序键**：`CTX_ANCHOR_SECTION_KEYS` 由 `{wi-before:0, wi-after:10, an-top:20, an-bottom:21, em-top:30, em-bottom:31}`
  改为 `{an-top:-20, an-bottom:-19, wi-before:0, wi-after:10, em-top:30, em-bottom:31}`（AN 移到最前）；`an-depth` 归入深度带。
- **标记行渲染根治**（用户bug的直接原因）：原实现"每个分组渲染完就补自己的 `#N` 标记行"，
  于是同一 `#0` 之前的第一组（前置）先补出 `#0`，**后一组（后置）就掉到了 `#0` 下方**。
  现改为「位于 `#mesid` 之前的**连续分组共享同一个标记行**」：攒到换消息时才补标记（`pendingBeforeMesid` + `flushBeforeMarker`），
  保证该消息之前的所有条目全部渲染在它上方。
- 实机验证（用户真实聊天，40 个条目）：修复后标记骨架 `#0(AI) → #7(AI) → 未匹配`，
  `worldInfoAfter` 的 14 条（秧秧/白芷/炽霞/漂泊者/各"后标"…）与 `worldInfoBefore` 的 8 条**全部位于 `#0` 之上**，
  仅深度注入（`eldoria`，插到 `#2` 之前）与"末尾"组在下 —— 与「完整上下文」查看器中
  `#0 main → #1 世界书·前置·worldInfoBefore → #2 世界书·后置·worldInfoAfter → #3 newMainChat → #4 chatHistory` 的次序**逐项一致**。

**三、子窗尺寸统一（用户要求）**

- 把条目详情子窗的并排定位抽成**共用**函数 `positionChildPopup(popupEl)` / `releaseChildPopupShift()` / `repositionChildPopups()`：
  桌面端贴主窗正右方（12px 间距，空间不足时主窗+子窗作为一对整体左移），窄屏居中；**宽高一律内联写入、与主窗逐像素一致**。
  三个子窗（条目详情 / 聊天统计 / 完整上下文）共用同一套逻辑与同一份位移还原表（`childPopupShifts`）。
- 「聊天」统计子窗从"贴框小浮层"改为**与主弹窗同尺寸的并排右窗**（实机 420×551 vs 420×551，间距 12px）；
  随之取消"点外部关闭"（点主窗是正常操作，不应连带关闭），保留 `Esc` 与关闭按钮。
- 「完整上下文」查看器改用同一套定位（实机 420×551、贴主窗右侧 12px）。
- 两个子窗**互斥**（都贴右侧，同时开只显示最后点的那个）。
- ⚠️ 踩坑记录：子窗 CSS 里 `width/max-width/max-height` **绝不能加 `!important`**——会盖掉 JS 内联写入的尺寸
  （首次实现时写成 `width:auto !important`，实测被撑到 466px 且位置算错）。已改为非 important 覆盖 `.hide-helper-popup` 基础类的 `width:90%; max-width:420px`。

**四、折线坐标轴 + 删说明文本（用户要求）**

- `renderSparkline(points, { tone, valueText })` 重写：新增**纵轴**（max/mid/min 三档刻度文字 + 浅色横向网格线）、
  **横轴**（每个数据点的聊天 messageID `#N`）、左侧与底部轴线；画布放大到 380×132；平坦数据自动上下留余量（不再贴边/NaN）；
  `valueText` 分别格式化为整数（tokens）与一位小数（t/s）。
- 删除「最近 N 条 AI 消息」「最近 N 条有耗时记录」两处说明文本（窗口范围由横轴 messageID 直接体现）。

**探针**：`scratch/t6.test.mjs`（新增，**53 断言全通过**）——覆盖窗口取数、AN-aware 落点与顺序键、折线坐标轴与转义、
五种生成类型的锚定落点、swipe 镜像、内存快照 LRU、预算阈值、`captureAnInfo` 空 AN 分支。

**验证**：`node --check` 通过；浏览器端到端（真实聊天）——统计子窗 / 完整上下文查看器均 420×551 贴主窗右侧；
双折线含坐标轴（纵轴 4773/2487/200、横轴 #0/#1/#3/#5/#7）；「上下文顺序」标记骨架与条目归属完全正确。

### 2.29 第十八轮：折线图**每个点直接标数值**（峰上 / 谷下，且不压线、不越界）（本轮新增；已实施并真实 Chromium 量测）

**用户提议**：把纵坐标的数值直接标注在对应点的上方（凸点）/ 下方（凹点）——更美观直观吗？

**结论**：是，采用；但"直接峰上谷下"会踩两个坑，都已在实现里解决：

1. **底部谷点无处可放**：数据区一直铺到横轴，谷点下方没有空间 → 只能翻到点的上方 → 正好落进 V 形夹角里**被折线穿字**
   （实测 20~46/240 个采样点落在数字上）。⇒ 数据区底部**留 16px 余量**（`dataBottomInset`），谷点得以稳定标在下方。
2. **陡段切进标注框角**：单调上升/下降序列里，相邻那一段折线会从标注框角上切过去（实测 7~13 个采样点）。
   ⇒ 标注按"危险侧"水平微移最多 8px：**标上方时**危险侧 = 邻居**更高**的那边（折线正往上爬）；**标下方时** = 邻居**更低**的那边；
   两侧都危险/都不危险则居中。首点再额外夹 `x ≥ padL-2`，不许压到纵轴刻度列。
3. 顶部同理：`padT` 由 10 提到 22，给最高点上方那行数值留位置（否则最大值的标注会被画布切掉）。

**实现（`renderSparkline`）**：

- 每点一条 `<text class="hcs-pt-label hcs-pt-label-${tone}">`，内容 = `valueText(value)`（tokens 取整 / 速度一位小数），
  与折线**同色**（tokens=绿 / speed=蓝，见新增 CSS `.hcs-pt-label-*`）；
- `sideOf`：峰（含平坦段）→ 上方；谷 → 下方；非极值按走向（上坡朝上、下坡朝下）；
- `labelPos`：上方 = `vy-7`、下方 = `vy+12`；撞上沿或压到横轴则翻面，最后夹在 `[9, axisBottom-3]`；
- 边界锚点：首点 `text-anchor="start"`、末点 `end`、中间 `middle`（贴边的两个不会伸出画布）；
- `padT` 10→22，并把数据区底部内缩 16px（新的 `plotTop` / `plotBottom`）。

**验证（真实 Chromium 量测；夹具 `scratch/chart-fixture.html`，由 `scratch/chart-fixture.gen.mjs` 生成，
内含 7 个场景：上升 / 下降 / 峰谷交替 7 点 / 平坦 / 速度（一位小数）/ 首末同为最大 / 100000:1 剧烈锯齿）**：

- **0 个**标注越界（全部落在画布内）、**0 个**压在折线上（沿每段采样 240 点，落在标注框内的样本数为 0）、
  **0 个**压到横轴 messageID 行或纵轴刻度列；
- 每个点的上下侧与"峰上 / 谷下"规则**逐一吻合**。
- `scratch/t7.test.mjs` 新增 **G 组（13 断言）**：校验文字、上下侧、边界范围、与横轴留空隙、tone 类名；
  `t7` **103 断言** / `t6` **53 断言** 全通过；`node --check index.js` 通过。

#### 2.28-R2 追加：真正的现场是「聊天统计详情」子窗（用户截图）——三个子窗现在完全统一

**用户反馈（附截图）**：截图里那条又粗又黑、带首尾倒三角、始终可见的滚动条，出在**「聊天统计详情」**子窗
（标题就是"聊天统计详情"），而不是上一轮改的「完整上下文」查看器。上一轮只改了查看器 → 漏了这个子窗。

**根因（同机 Chromium 实测，四种写法对照）**：

| 写法 | 滚动条槽宽 | 结果 |
|---|---|---|
| 不给任何滚动条属性 | 15px | 平台原生滚动条（Windows 上带首尾倒三角、始终可见） |
| 只给 `scrollbar-width: thin` | 10px | 仍是**平台样式**、**默认可见** ← 旧 `.hcs-body` 就是这一种 |
| `scrollbar-width: thin` + `scrollbar-color: transparent transparent` | 10px | **完全不可见**（= 条目详情子窗的做法） |
| 只给 `scrollbar-color` | 15px | 颜色生效但槽宽仍是原生尺寸 |
⇒ **只设 `scrollbar-width` 不会让滚动条消失**，必须成对给出 `scrollbar-color`（否则 Chrome 走平台样式）。

**修复**：

- `style.css`：`.hcs-body` 补齐与另两个子窗**逐条同款**的规则——默认整条隐藏（`scrollbar-width: thin` +
  `scrollbar-color: transparent transparent !important`）、轨道透明、`::-webkit-scrollbar-button { display:none; width:0; height:0 }`
  （去倒三角）、滑块透明、`.hh-sb-*` 三档显色；另加 `#hide-helper-chat-stats-popup ::-webkit-scrollbar { display:none !important }`
  + `#hide-helper-chat-stats-popup * { scrollbar-width: none !important }`（窗内其余滚动条隐藏）。
- `index.js`：`ensureChatStatsPopup()` 补 `initHiddenScrollbar(el.querySelector('#hcs-body'))`
  （此前**漏绑** → 连"滚动/悬停才显色"也从未生效）。
- 三窗统一清单（改一处必须改全）：条目详情 `.hide-entry-body` / 完整上下文 `.hide-entry-body` / 聊天统计 `.hcs-body`。

**验证（真实 Chromium，夹具 `scratch/scrollbar-fixture.html` 已扩成三窗并排）**：三者 `scrollbar-width=thin`、
`scrollbar-color=rgba(0,0,0,0) rgba(0,0,0,0)`（= 不可见）、槽宽 10px、三档显色 0.15 / 0.22 / 0.38 **完全一致**；
三个窗口各自的 `::-webkit-scrollbar-button` 复位与窗内隐藏规则都在规则表内。
`scratch/t7.test.mjs` 新增 **F2 组（25 断言）**：对**三个子窗逐一**校验上述规则 + JS 绑定 + 内联兜底，
防止再出现"只改了一个子窗"；`t7` **90 断言** / `t6` **53 断言** 全通过。

#### 2.28-R1 追加：为什么"改了 CSS 却看不到"——样式表不热更新（用户复查反馈）

**现象（用户复查）**：改完 `style.css` 后重开「完整上下文」子窗，滚动条**仍然**又粗又丑、默认可见、还带首尾倒三角。

**排查结论：插件侧没问题，是"页面里的那份 CSS 还是旧的"**

1. 服务端确认：`GET http://127.0.0.1:8000/scripts/extensions/third-party/hide/style.css` 返回的就是磁盘新版
   （78968 字节、含 `#hide-helper-context-viewer .hide-entry-body.hh-sb-*` 全部规则、已无旧的
   `#hide-helper-context-viewer ::-webkit-scrollbar { display: block }`），`Cache-Control: public, max-age=0` + 新 ETag →
   **刷新即取新版**。插件目录里只有这一个 CSS（`manifest.json: "css": "style.css"`），不存在第二份样式表。
2. 真实 Chromium 量测（新增夹具 `scratch/scrollbar-fixture.html`，直接 `<link>` 插件的 `style.css`、DOM 结构与运行时一致）：
   - 条目详情子窗 `.hide-entry-body` 与查看器 `#hide-context-body`：`scrollbar-width: thin`、
     `scrollbar-color: rgba(0,0,0,0) rgba(0,0,0,0)`（**完全透明 = 默认不可见**）、
     `offsetWidth−clientWidth−边框 = 10px`、`clientHeight = 488`——**两者逐项完全相同**；
   - 三档显色可取到：滚动中 `rgba(0,0,0,.15)` / 悬停滚动条 `rgba(0,0,0,.22)` / 拖拽 `rgba(0,0,0,.38)`；
   - `::-webkit-scrollbar-button` 规则对**两个子窗**都在规则表里（`display:none; width:0; height:0` → 无倒三角）；
   - 页面里只有 `#hide-context-body` 是滚动容器（`scrollHeight 2362 > clientHeight 488`），其父 `overflow: hidden`。
3. 因此"看不到变化"的原因只能是：**ST 在页面加载时用 `<link>` 注入扩展 CSS，之后不会热更新**——
   页面没刷新时，DOM 里还是旧样式表（旧规则正是"永远可见的 8px 滑块 + 平台倒三角"）。**刷新页面（F5）即生效。**

**顺带加固（本轮新增）**：`initHiddenScrollbar` 现在把「默认隐藏」**同时内联**写到内容区
（`scrollbar-width: thin` + `scrollbar-color: transparent transparent`），不再只依赖样式表是否为最新；
三档显色仍由 CSS 的 `.hh-sb-*`（带 `!important`）负责，实测**胜过内联值**（0.15 / 0.22 / 0.38），
两个子窗默认值仍然完全一致。

**自检方式（10 秒）**：浏览器打开
`http://127.0.0.1:8000/scripts/extensions/third-party/hide/scratch/scrollbar-fixture.html`
—— 左侧是条目详情子窗、右侧是完整上下文查看器，两者滚动条外观应当**完全一致**（默认看不见，滚动/悬停才浅色显现）。

**复跑**：`scratch/t7.test.mjs`（65 断言）+ `scratch/t6.test.mjs`（53 断言）仍全通过；`node --check index.js` 通过。

### 2.28 第十七轮：区块顺序改按**发送时刻的真实提示词顺序**（不再写死 ST 默认顺序）× 「完整上下文」滚动条同款 × AI 统计剔除"无生成时间"的消息（本轮新增；已实施并离线验证）

**需求（用户三点）**：① 当前插件是否**写死了 `promptManager` 的默认顺序**？若是则不稳定——`prompt_order` 随时可能被改动，
发送上下文之前就可能已经变了，不能写死所谓"默认顺序"。② 「完整上下文」子弹窗右侧**滚动条的隐藏机制与样式**要和「条目详情子窗」一致。
③ AI 平均 tokens / AI 平均生成速度 / 折线图的数据：`messageID = #0` 若**没有生成时间数据**就不该统计（ST 里 #0 可以是人为预置/提取的）；
生成时间**必须取自后端字段**，不能读前端 DOM（DOM 里的计时显示可被用户隐藏）。

**一、确认用户判断正确：确实写死了默认顺序**

- 原实现：`entryContextAnchor` 对区块直接写死 key（`wi-before=0 / wi-after=10 / em-top=30 / em-bottom=31 / AN=-20/-19`），
  `contextAnchorOrderKey` 再用固定的 `CTX_ANCHOR_SECTION_KEYS` 排序——这正是"ST 默认顺序"
  （`PromptManager.js:2087 promptManagerDefaultPromptOrder`），属于把**初始值当规矩**。
- 据 ST 源码，真实落点由 `prompts = promptManager.getPromptCollection(type)` 的**集合下标**决定
  （`openai.js:1470` 取集合 → `addToChatCompletion` 里 `chatCompletion.add(collection, prompts.index(source))`
  → `ChatCompletion.add` 实为 `messages.collection[position] = collection`，**按下标就位**）。
  而 `getPromptCollection` 的顺序来自 `getPromptOrderForCharacter(activeCharacter)`，
  即用户在提示词管理器里**可随时增删/启停/拖动**的 `prompt_order`；集合里没有的标识会被
  `prompts.add(newPrompt)`（`openai.js:1473-1493`）或 `chatCompletion.add(collection, -1)`（1106/890）**push 到末尾**。

**二、实现（三层：发送时刻捕获 → 写进锚点 → 渲染只读锚点）**

- 新增 `capturePromptOrderSnapshot(genType)`：与 ST 同源（`promptManager.getPromptCollection(type).collection`），
  产出 `{ ranks: Map<identifier, 下标>, chatOrd, length }`；在 `CHAT_COMPLETION_SETTINGS_READY` 里**同步**捕获
  （真实发送用 `pendingGenType`；模拟发送用消费前的 `simGenType`），随 `computeRenderedStats` 的 `options.promptOrder` 传入——
  异步统计期间用户即便改了顺序，也不会影响本次结果。
- `entryContextAnchor(entry, sendable, anInfo, orderSnap)`：
  · 区块（前置/后置/AN/EM）→ `ord` = 本次**真实顺序号**（AN = `main 下标 ±0.5`，EM = `dialogueExamples 下标 +0.1/+0.2`）；
  · 新增 `after`：**该区块是否整块落在全部聊天消息之后**（把 `chatHistory` 拖到它前面时就会发生）→ 对应消息改取**末条**（否则仍取首条）；
    label 与悬停说明随 `after` 实时改写，不再固定写"位于全部聊天消息之前"；
  · 标识不在集合里 → 顺序号取 `length`（复刻 ST 的"追加到末尾"），`after = true`；
  · 深度注入 → `chatOrd` ＋ 渲染期按 mesid 归一化的分数，稳定落在 `(chatOrd, chatOrd+1)`，天然与"被拖到聊天之后的区块"分开（`depth-end` 取区间最末）；
  · 拿不到快照（旧锚点 / 调用失败）→ `ord/chatOrd = null`，`contextAnchorOrderKey` **逐字节回退**旧的固定键（旧行为不变）。
- `contextAnchorOrderKey(anchor, mesid, edges)`：新锚点按 `ord`（区块）或 `chatOrd + 分数`（深度）排序；出口/其他仍为 90000/95000；
  `groupEntriesByAnchor` 的 `side` 改由 `anchor.after` 决定（旧锚点仍按 kind 表）；`contextAnchorView` 的说明文案同步改口径。

**三、「完整上下文」滚动条与条目详情子窗同款（用户要求）**

- `initEntryBodyScrollbar` → 更名 **`initHiddenScrollbar`**（两个子窗共用；条目详情子窗行为不变），
  `ensureContextViewer` 里对新内容区调用 `initHiddenScrollbar(el.querySelector('#hide-context-body'))`。
- `style.css`：`#hide-helper-context-viewer .hide-entry-body` 补齐**默认整条隐藏**
  （`scrollbar-color: transparent transparent` + 滑块透明）、轨道透明、去掉平台自带的倒三角按钮，
  以及 ①滚动中（`--hh-scrollbar-thumb-faint`）②鼠标在滚动条上（`--hh-scrollbar-thumb`）③拖拽（`--hh-scrollbar-thumb-hover`）
  三档显色（`.hh-sb-scrolling / .hh-sb-hover / .hh-sb-dragging`，与条目详情子窗同源变量）；该窗内其余滚动条一律 `display:none`（同条目详情子窗）。

**四、AI 统计剔除"无生成时间"的消息（用户要求）**

- `replySeconds(msg)`：只读**后端字段** `message.gen_started / message.gen_finished`
  （ST 在 `script.js` 生成收尾时写到消息对象上、随聊天落盘；swipe 时由 `syncSwipeToMes` 同步为当前版本的值），
  统一用 `new Date()` 解析（Date / ISO 字符串 / 毫秒数都能吃）；**绝不读 DOM**——消息上的计时显示可被用户关闭，DOM 取不到 ≠ 没有记录。
- `collectLatestAiMessages`：**没有生成时间的 AI 消息一律不采集** → AI 平均 tokens / 平均生成速度 / 两条折线全部不含它们
  （典型情形：#0 是人为预置或从别处提取的回复，以及被导入/清理过时间的老消息）；均值分母随之改为"有生成时间的条数"。

**探针**：`scratch/t7.test.mjs`（新增，**65 断言全通过**）+ `scratch/t6.test.mjs`（**53 断言全通过**，无回归）。
t7 覆盖：默认顺序下的区块顺序号与 `after/mesid`、把 `worldInfoBefore` 拖到 `chatHistory` 之后的**顺序与落点整体翻转**、
深度注入区间单调性与末尾收束、AN 紧随主提示词（前/后）、标识缺失时"追加到末尾"、
旧锚点逐字节回退、`capturePromptOrderSnapshot` 的失败/空集合回退、AI 统计过滤与均值/折线 messageID、滚动条三档 CSS 与初始化。

**验证**：`node --check index.js` 通过；离线探针合计 **118 断言**全通过（新增 65 + 既有 53）。
按 TESTING.md §3.4 的既定结论（用户可能正在使用该聊天），本轮**不做**浏览器内注入，为纯离线验证 + ST 源码逐行比对。

### 2.30 第十九轮：「完整上下文」检索/筛选 × 每条消息内部滚动区与放大镜 × 世界书消息「条目来源」子窗（主弹窗左侧）（本轮新增；已实施并真实 Chromium 实测）

**用户需求（原话要点）**
1. 「完整上下文」增加**检索**（检索 + 高亮 + 跳转显示）与**筛选**（聊天/世界书/其他 & system/user/assistant）；
2. `hc-msg` 内部也要有滚动条（默认隐藏同款），并在 `hc-msg-tokens` 提供**放大镜**图标 → 对单独一条做内部检索/跳转；
3. **世界书来源**的消息：放大镜**左侧**再加一个展开图标 → 在插件主弹窗**左侧**（移动端显示在最上方）显示子弹窗：
   顶部 = 该消息的 `hc-msg-head` 数据，其次 = 该消息的饼图（`tub-row-wi-chart` 口径），
   下方 = `tub-entries-header-sticky` 那一段（主题换成该消息的世界书），**去掉 `tub-sort-btn`**、排序固定「上下文顺序」。

**一、数据层（快照仍是「仅内存」，不落盘）**
- `attributeSlotsBySource` 新增返回 `perSlotEntries: Map<slotIdx, Map<key, tokens>>` —— 与 `perEntry` **同源同值**，只是按槽位保留（`perEntry` 只能回答"整体有多少"，新子窗要回答"**哪一条消息**里有哪些条目"）。
- `entriesMeta` 补展示字段 `name / book / type / uid`；完整上下文快照的每条消息新增 `wiEntries`（**只有世界书来源的消息**才有，其余为 `null`）：
  `[{ name, book, type, uid, tokens, anchor, pos }]`。其中 `anchor` 与 `wiDetailedStats` 的 record **共用同一份推导**（`entryContextAnchor` 只算一次，`entryAnchors` 表两处共用）。
- 顺带修正：快照里的逐条明细**不做**"舍入差额补到最大条目"那一步（那是 `wiDetailedStats` 为了与 `wi` 总额自洽才需要的；这里是"这条消息有哪些条目"）。

**二、抽出共用纯函数（主面板 ↔ 新子窗共用，避免两套渲染必然漂移）**
- `sortEntriesByMode(combined, mode, edges)`：原 `renderEntriesList` 里内联的排序（上下文顺序键 / 默认 EJS>动态>常驻 / 未匹配排后）。
- `createEntryRowBuilder({ booksCount, bookFilter, absoluteMax })` → `{ html, records }`：原 `entryRowHtml` + `rowRecords`（行样式、进度底色、书名标签、未匹配态、`data-*` 全套照搬）。
- `contextAnchorMarkerHtml(blockGroups, mesid, edges)`：原 `ctxMarkerHtml`（`#N` 位置标记行 + 右侧 role）。
- `renderContextOrderHtml(entries, edges, rowHtml)`：原"**连续分组共享标记行**"的整段分组/标记渲染。
- `renderPieViewInto(container, c, d, e, total)`：`renderPieView` 的容器版（主面板保留薄包装，行为不变）。

**三、「完整上下文」查看器：检索 + 筛选**
- 顶部工具栏（`flex:0 0 auto`，不随内容滚动）：检索框（🔍 + 输入 + 命中计数 + 上一个/下一个/清空）＋ 两个维度筛选
  （来源 `全部/聊天/世界书/其他`、角色 `全部/system/user/assistant`）＋ `显示 x / y 条`。
- 高亮：`highlightTextHtml(text, term, { attr, base })` **先按命中区间切分、再逐段转义**（绝不在已转义文本上做替换，避免跨实体误匹配）；
  全部命中 `mark.hc-hit`、当前命中 `mark.hc-hit-current`（同时给所在行加 `.hc-msg-hit-current` 左侧描边）。
- 跳转：`Enter` / `Shift+Enter` 或上/下按钮，环形；计数为 `第 n / 共 m`；**命中多于摘要可见范围的消息自动展开全文**（否则"跳得到、看不见"）。
- 两条编号通道：整窗检索 → `data-hit`；被「本条内检索」接管的那条 → `data-mhit`（互不干扰，整窗计数会剔除后者，实测 `1/5` 与 `1/10` 并存）。
- 筛选：`ctxSourceGroupOf(kind)`（chat / wi（5 种 `wi-*`）/ other）× `ctxMessageVisible(m, { source, role })`（两维叠加）。

**四、每条消息的内部滚动区 + 放大镜**
- 行结构：`.hc-msg-head`（tokens 里放两个图标）＋ `.hc-msg-search`（本条检索行，默认隐藏）＋ `.hc-msg-body`（`max-height:148px; overflow-y:auto`）。
- `.hc-msg-body` 复用 `initHiddenScrollbar`（默认隐藏 + 滚动/悬停/拖拽三档显色）。
  ⚠️ **`scrollbar-width / scrollbar-color` 必须带 `!important`**：同一窗口里"其余滚动条一律隐藏"那两条用的是 `*` 选择器 **且带 `!important`**，
  而 `!important` 优先于特异度 —— 不加就被压成 `scrollbar-width: none`，三档显色完全失效（真实 Chromium 实测抓到，见 §九）。
- `initHiddenScrollbar` 的全局 `mouseup` 改为**只注册一次**（用 `.hh-sb-dragging` 类名统一清态）：现在每条消息体都要绑一次，逐条往 `window` 挂监听器会持续累积。

**五、世界书消息「条目来源」子窗（`#hide-helper-wi-msg-popup`，贴主弹窗左侧）**
- `positionChildPopup(popupEl, { side, minHeight, allowShift })` 三个新参数：
  · `side='left'` → 贴主窗正左方；左侧放不下时把"主窗 + 子窗"整体**右移**（最小必要位移，复用 `recordMainShift` 的还原记录）；
  · `minHeight` → 主窗很矮时保证"head + 饼图 + 条目列表"都看得见（实测主窗仅 139px 高时列表被压成 **0 高度**、条目完全看不见）；
  · `allowShift=false` → 被平移后重新贴合的那一轮不再挪主窗（否则左右两个子窗会互相把主窗推来推去）。
- **跟随重新贴合**：主窗因并排被平移后调 `repositionOtherChildPopups(exceptEl)`，让其它已打开的子窗按新位置重排
  （实测：不跟随 → 查看器与主窗新位置错开、互相压住）。
- 内容：① `#hwmp-head` = 该消息 `hc-msg-head`（序号 / 角色 / 来源槽位 + 标识 / tokens）；
  ② `#hwmp-pie` = 该消息的世界书构成饼图（常量/动态/EJS，`renderPieViewInto`）；
  ③ 条目区 = `tub-entries-header-sticky` 同款头部（标题「本条世界书条目」+ 检索 + 总数 + 书签 + 类型过滤，**无排序按钮**）
  ＋ 列表（`sortEntriesByMode(records, 'context', edges)` **固定「上下文顺序」**，分组/位置标记与主面板同一套）；行点击 → 条目详情子窗。
- 生命周期：换快照 / 关查看器时一并收起；`applyTheme` 的目标列表加上新子窗；`repositionChildPopups` 的登记表新增 `['hide-helper-wi-msg-popup','left',WI_MSG_POPUP_MIN_HEIGHT]`。

**六、类型过滤菜单的宿主（含一个真 bug 修复）**
- `openTypeFilterMenu` 新增 `host`（菜单挂载窗口 —— 菜单坐标是相对**挂载窗口的 padding box** 算的）；
- **`toggleTypeFilterMenu` 之前没有透传 `host`** → 子窗的过滤按钮会把菜单挂到**主弹窗**上（真实浏览器实测：菜单出现在主弹窗里）。本轮补上透传。

**七、CSS**
- 新增样式块：查看器工具栏 / 命中高亮两档 / `.hc-msg-body` 内部滚动条（含三档）/ 行内检索行 / 子窗独有布局（head、饼图行、条目区 flex、列表滚动条三档）。
- 主面板那批 `#token-stats-panel` 规则**镜像一份**到 `#hide-helper-wi-msg-popup`（一次性脚本 `add_wi_popup_scope.py` 处理 **55 条**：
  **只追加选择器、不动任何声明** → 选择器分组不改变原选择器行为，主面板零影响；子窗里没有的元素自然不生效）。
- `width / max-width / max-height` 一如既往**不能**加 `!important`（会盖掉 JS 内联尺寸）。

**八、顺带清理**
- 删掉从未使用的 `contextViewerSeq`；
- 行 HTML 直接写进 `#hide-context-body`（去掉多余的一层 `#hc-list` 嵌套容器），并且**行首屏就带摘要**（不再依赖后续刷新才有内容）。

**九、验证（离线 320 断言 + 真实 Chromium 夹具实测）**
- 新增 `scratch/t8.test.mjs`（**164 断言全通过**）：检索区间（大小写/不重叠/下标）、高亮切分转义与编号、摘要容器、来源分组与两维筛选、
  排序口径（上下文/默认/未匹配）、条目行构建器（data-* / 进度底色 / 书名标签 / 未匹配态 / 记录表）、位置标记行与整段分组（含"同一 messageID 只出现一次"）、
  行与两个图标结构、**端到端 `wiEntries`**（`computeRenderedStats` → 快照：name/book/type/uid/anchor、Σ 明细 ≡ 单条 token、非世界书为 null、`Σ 明细 ≡ wi` 未回归）、
  子窗 DOM 写入（head / 饼图 / 列表 / 无排序按钮）、定位三行为（贴左 / 最小必要右移 / 跟随重新贴合）、接线与 CSS 不变式（含"内部滚动条两条必须带 `!important`"）。
- 回归：`scratch/t6.test.mjs` 53 + `scratch/t7.test.mjs` 103 **全通过**（合计 **320 断言**）；`node --check index.js` 通过。
- 真实 Chromium 夹具（**不碰用户聊天、不碰 ST 页面**）：`scratch/ctx-fixture.gen.mjs` 生成 `scratch/ctx-fixture.lib.mjs`
  （ST 全局桩 + 去掉 ST import 的 index.js + 追加导出），`scratch/ctxviewer-fixture.html` 载入**真实 index.js + 真实 style.css**，
  用 `saveContextDump` 注入合成快照（9 条，含一条 60 行的世界书消息）后驱动真实交互：
  · 检索：65 处命中（60 处在长世界书消息内）→ 计数 `1/65`；连点 30 次"下一个" → 内部滚动区自己滚到 **346px** 且命中在可视区内（内外两层滚动都对）；
  · 筛选：来源=世界书 → `0,5`；来源=世界书 ∩ 角色=system → `0,5`；界面显示 `显示 2 / 9 条`；
  · 本条内检索：`1/10`、跳一次后内部滚动 **42.4px** 且命中在体内可见；与整窗检索并存时计数 `1/5` + `1/10`，关闭后回到 `1/65`；
  · 内部滚动条三档：默认 `scrollbar-color: transparent transparent`（`scrollbar-width: thin`）→ 滚动 `rgba(0,0,0,0.15)` → 鼠标进窄带 `rgba(0,0,0,0.22)`；移开回落到全透明；
  · 左侧子窗（用内联盒而非视觉 rect —— 夹具里入场动画被冻结在 0.95 缩放）：`左子窗 8..480 | 主窗 492..964 | 查看器 975.8..1447.8`（贴左、查看器跟随、三者不重叠、都在视口内）；
    head = `#0 system 世界书·前置 · worldInfoBefore 900 t`；饼图 67%/33% + 图例 `常量:600 / 动态:300`；列表 2 行（600/300）+ 1 个位置标记 + **无排序按钮** + 总数 `900 (300 + 600)`；
  · 子窗类型过滤菜单**挂在本子窗内**（修复前在主弹窗里）→ 选"动态"→ 1 行 / 总数 `300 (300)`；条目检索"动态"→ 1 行；点条目 → 条目详情子窗打开；关闭子窗 → 主窗还原（`(未位移)`）；
  · 窄屏 700×900：左子窗 `top: 8px`（按要求"显示在最上方"）、`left: 114px`（居中）、高度 420（最小高度生效）；
  · 健壮性：世界书消息但快照里没有逐条来源（旧快照 / 文本匹配不到）→ 饼图与列表给空状态、不报错。

### 2.31 第二十轮：子窗落点重排（主窗最左 / 中间子窗居中 / 查看器最右）× 条目来源子窗两处口径修正 × 新增「上下文目录」子窗（本轮新增；已实施并真实 Chromium 实测）

**一、用户需求（原话五条 + 两处追问结论）**
1. 查看世界书消息详情时，`#hide-helper-wi-msg-popup` **显示在网页中间**，插件主弹窗 `#hide-helper-popup` **显示在最左侧**；
2. 在 `#hide-helper-wi-msg-popup` 里打开 `#hide-helper-entry-popup` 时，条目详情显示在**主弹窗正上方**；
3. 该子弹窗提供 `tub-book-filters` 以切换不同世界书来源的条目；`tub-new-list-item tub-entry-clickable` 标题**括号显示来源世界书**；
4. 该子弹窗的条目列表（`hwmp-row-entries`）**不显示 `tub-ctx-msg`**；
5. 查看器工具栏 `hc-filters` 的**最右侧（右对齐）**新增 `hc-icon-btn hc-wi-expand` 图标 → 打开一个**居中**子弹窗（主弹窗在最左），
   列出**所有消息**的 `hc-msg-head`，其中"世界书"来源的消息**细分到各个条目**；同时 `#hide-helper-context-viewer` 显示完整上下文；
   点左侧子弹窗里的 `hc-msg-head` → 右侧上下文弹窗**跳转滚动**到对应位置。

**追问确认（避免歧义，本轮实现按此口径）**
- 第 3/4 条里的"子弹窗"= **世界书消息条目来源子窗 `#hide-helper-wi-msg-popup`**（唯一同时含 `tub-book-filters` / `hwmp-row-entries` /
  `tub-new-list-item` 的窗口；条目详情子窗本身只有单条详情，没有列表）；
- 宽屏布局取"**A：主弹窗最左 → 中间子弹窗居中 → 完整上下文查看器贴最右边缘**（三窗互不重叠）"。

**二、实现要点（`hide/index.js`）**
1. **落点系统扩三种**（`positionChildPopup` 的 `side`）：`'center'` = 主窗钉最左（`recordMainShift` 复用同一套"关闭还原"记账）+ 本子窗
   水平居中（`round((vw - 主窗宽)/2)`）；`'above'` = 主窗**正上方**（左右对齐，`top = 主窗 top − 12 − 本窗高`，上方不够→下方→贴顶兜底）；
   `'right-edge'` = 贴视口**最右边缘**（居中模式生效时"完整上下文查看器"由内部自动改写为它，避免压在居中的子窗上）。
   窄屏（三窗并排放不下）时居中子窗退化为"居中 + 贴顶"（与旧版移动端"叠加显示"一致）。
2. **居中模式状态**：`CENTER_POPUP_IDS = ['hide-helper-wi-msg-popup', 'hide-helper-ctx-outline-popup']` + `isCenterModeActive()`（两者互斥，
   开一个必关另一个）；`childPopupTable()` 里两者的 side 统一为 `'center'`（`WI_MSG_POPUP_MIN_HEIGHT` / `CTX_OUTLINE_MIN_HEIGHT`）；
   视口变化走同一张表重新贴合。
3. **条目详情落点**（需求 2）：`openEntryDetailPopup` 打开时按"当时有没有居中子窗"定 `entryPopupPlacement ∈ {'above','right'}`；
   新增 `onCenterPopupClosed()`：居中子窗关闭后把查看器收回"贴主窗右侧"、并让 `'above'` 的条目详情**继续**贴着主窗正上方
   ——⚠️ **不重置** `entryPopupPlacement`：`wi-msg → 目录` 切换会先关一个居中子窗再开另一个，若在这一瞬重置成 `'right'`，
   条目详情会掉到主窗右侧（真实 Chromium 实测复现后修正）。
4. **世界书消息子窗列表口径**（需求 3/4）：`createEntryRowBuilder` 增 `alwaysShowBook`（行尾**一律** `(书名)`，不随"筛选/书目数量"隐藏）；
   `renderContextOrderHtml(entries, edges, rowHtml, { markers })` 增开关（`markers:false` → 不渲染任何 `tub-ctx-msg` 标记行，
   含"未匹配"分组头；分组与顺序逻辑不变）；`refreshWiMsgList` 改用 `{ markers:false }` + `alwaysShowBook:true`，
   `tub-book-filters` 改为"有书就渲染"（不再要求"多于一本"）。
5. **新子窗「上下文目录」**（需求 5）：`ensureContextOutlinePopup` / `openContextOutlinePopup` / `closeContextOutlinePopup` /
   `toggleContextOutlinePopup` / `renderContextOutlineList` / `refreshContextOutlineIfOpen` / `syncOutlineActiveHead` /
   `scrollContextViewerToMessage`。列表 = 所有消息的 `hc-msg-head`（序号/角色/槽位/tokens）+ 世界书消息的条目明细
   （行样式复用 `createEntryRowBuilder`，条目名带 `(书名)`，点击 → 条目详情子窗）。点消息头：
   `row.scrollIntoView({ block: 'start' })` + 给该行加 `.hc-msg-target`（左侧描边 + 底色），目录里同步 `hco-item-active`；
   若该消息正好被查看器的筛选藏住，**先把筛选复位到「全部」**再定位（保证"点了一定跳得到"）。
6. **接线**：查看器 `hc-filters` 末尾（`.hc-count` 之后 = 最右）加图标 `#hc-wi-expand-all` + 点击 toggle；
   `renderContextList` 尾部与空分支都调 `refreshContextOutlineIfOpen()`；`openContextViewer` 换快照时同时收起 wi-msg 与目录；
   `closeContextViewer` 同时收起两者；`applyTheme` 目标加入 `#hide-helper-ctx-outline-popup`。

**三、实现要点（`hide/style.css`）**
- 新增 `#hide-helper-ctx-outline-popup` 整套（尺寸随 JS 内联、`z-index:10000`、头部固定 + 内容区滚动 + 滚动条"默认隐藏 + 三档显色"、
  关闭按钮、`hco-*` 列表/消息头/条目区样式；条目行镜像主面板用到的几个类，不动主面板那一整段规则）；
- `#hide-helper-context-viewer .hc-filters .hc-wi-expand`（右对齐）+ `.hc-msg-target`（目标行高亮，与"当前命中"同一视觉语言）；
- **补暗色变量选择器**：`#hide-helper-wi-msg-popup[data-theme="dark"]`（此前 JS 打了 `data-theme` 但选择器列表里没有它 →
  暗色下该子窗仍是亮色变量）与新窗的 `#hide-helper-ctx-outline-popup[data-theme="dark"]`；同时确认没有顶掉原有的
  `#hide-helper-context-viewer[data-theme="dark"]` / `.hide-modal-overlay[data-theme="dark"]`（首版改动曾误删，已在夹具/断言里复查）。

**四、验证**
- `node --check index.js` 通过。
- 离线断言：新增 `scratch/t9.test.mjs`（**48 断言全通过**：三种落点的坐标与兜底、条目详情落点判定与收尾、`alwaysShowBook`、
  `markers:false`、wi-msg 列表"无标记行 + 书名括号 + 筛选按钮"、目录子窗"全部消息 + 条目明细 + 无标记行"、点击跳转与高亮、CSS/接线不变式）；
  回归 `scratch/t8.test.mjs` **164**（按新口径更新 I19/I28/J11–J14：单本书也给筛选按钮、"成对模式"改用独立桩、wi-msg 改 `side:'center'`、
  `applyTheme` 两个新 id、关查看器收起两个子窗）+ `scratch/t7.test.mjs` **103** + `scratch/t6.test.mjs` **53** —— 合计 **368 断言全通过**。
- 真实 Chromium（经 ST 静态服务打开 `scratch/ctxviewer-fixture.html`，真实 `index.js` + 真实 `style.css`，**不碰用户聊天**）实测：
  · 需求 1：`主窗 left=8（最左）| wi-msg left=590、w=420（1600 视口水平居中）、top 与主窗一致 | 查看器 left=1172、1172+420=1592=vw−8（贴最右）`，
    三者互不重叠；`resize` 后按同一张表重新贴合；
  · 需求 2：条目详情 `left=8（与主窗左边缘对齐）、top=66 = 224−12−146（正上方，间距 12）`；目录打开后仍是 `leftAligned=true / gapAbove=12`；
    关闭目录 → 主窗还原 `590`、查看器回到 `1022 = 590+420+12`（贴主窗右侧）；
  · 需求 3/4：wi-msg 条目行 `前置常驻条目 (Eldoria)` / `前置动态条目 (EldoriaNSFW)`，`tub-ctx-msg` 计数 **0**，
    筛选按钮 `[所有条目, Eldoria, EldoriaNSFW]`；
  · 需求 5：图标在 `.hc-filters` 内且是**最后一个子节点**、类名 `hc-icon-btn hc-wi-expand`；目录 `left=590`（居中）、主窗 `8`、查看器 `1172`，
    `heads=9/9`（全部消息）、条目明细 3 行、标记行 0；点 `#5` → 查看器 `scrollTop 0 → 447.2`、目标行 `.hc-msg-target`、行顶与滚动容器顶对齐（offset 0）、
    目录里 `hco-item-active=5`；被筛选藏住的 `#3` 也能点（筛选自动复位 `all/all` 后定位成功）；
  · 窄屏 900×800：目录"居中(240) + 贴顶(8)"、主窗仍钉最左、查看器退化贴右（与旧版窄屏叠加行为一致）。
- 夹具自纠（本轮）：`scratch/ctxviewer-fixture.html` 补 `box-sizing: border-box`（ST 全局有，standalone 页没有 → 子窗会宽出 52px、
  量测结论整体失真）与"量测前等 0.3s 入场动画"（`hhFadeIn` 的 `scale` 会让 rect 偏小/偏移）；世界书消息的条目跨两本书（验证筛选与书名标签）。
- 截图：`scratch/round20-a-wimsg-center.png`（需求 1）、`scratch/round20-b-entry-above-main.png`（需求 2）、
  `scratch/round20-c-outline-context.png`（需求 5，含需求 2 的条目详情）。

**五、已知取舍（有意为之）**
- 三窗并排放不下时（视口宽 < 3×主窗宽 + 间距），中间子窗退化为"居中 + 贴顶"、查看器退化为贴右边缘 → 两者会重叠（旧的窄屏行为本就是叠加显示）；
- 目录列出**所有**消息（含被查看器筛选隐藏的）——点它会把筛选复位到「全部」（否则"点了一定跳得到"不成立）；
- 中间子窗互斥（`wi-msg` ↔ 目录，都占网页中间）；目录里的条目行不做类型过滤（本轮未要求）；
- `wi-msg` 子窗的列表**固定**上下文顺序（无排序按钮，沿用第十九轮口径）。

### 2.31-R1 修复「点目录里的消息头 → 整个网页/插件 UI 一起向上偏移」（用户实测；已定位到根因并在真实 ST 页 A/B 验证）

**现象**：在「上下文目录」子窗（`#hide-helper-ctx-outline-popup`）里点某条消息 → 不只是右侧查看器滚到该条，
**整个网页（连插件弹窗一起）向上偏移**了。

**根因（真实 ST 页实测，两条叠加）**：
1. **ST 的 `<html>` 带 `transform`**（实测计算值 `matrix(1, 0, 0, 1, 0, 0)`）。按规范，任何非 `none` 的 transform 都会让该元素成为
   **fixed 后代的 containing block** —— 于是我们那几个 `position: fixed` 的弹窗，实际是"跟着文档滚动一起走"的
   （不再相对视口固定）。夹具里没有这条 → 之前所有夹具都复现不出这个 bug。
2. **`scrollIntoView` 会把 target 的"所有可滚祖先"都滚一遍**（不只我们自己的容器，还包括文档 html；ST 页文档确实可滚：
   实测 `scrollHeight 904 > clientHeight 674`）。于是点一下目录消息头，文档 `scrollTop` 被改 108~349px，
   而弹窗又跟着文档走 → **整页与整个插件 UI 一起上移**。

**真实 ST 页 A/B（同一目标行、同一容器，逐字跑的 index.js 实现）**：

| 实现 | 文档 scrollY | 弹窗容器 top | 内层滚动容器 | 行顶对齐 |
|---|---|---|---|---|
| 旧：`scrollIntoView({block:'start'})` | 0 → **108**（页面被滚） | 100 → **−8**（UI 跟着上移） | 1200 | 0 |
| 新：`scrollIntoViewWithin(root, target, 'start')` | 0 → **0** ✓ | 100 → **100** ✓ | 1200 | 0 ✓ |

夹具复现（`scratch/ctxviewer-fixture.html` 补了 `html{transform}`）：修复前 `pageAfter 349.6`、`viewerTop 224 → −125`；
修复后 `pageAfter 0`、`viewerTop 224 → 224`（`uiStayed/pageStayed` 均 true）。

**修法（`hide/index.js`）**：新增"只滚自己的容器"的一组纯函数并替换两处 `scrollIntoView`：
- `isScrollableY(el)`（`scrollHeight > clientHeight` + `overflow-y ∈ auto/scroll/overlay`）、
  `scrollLayerTo(container, target, block)`（单层：`block='start'` 对齐到顶；否则"最小滚动"——与旧的 `nearest` 语义一致）；
- `scrollIntoViewWithin(root, target, block)`：由内向外只滚 **target ⇄ root 之间**的那几层滚动容器，最外层 root 按 `block` 滚动，
  **root 之上的容器（含文档）一个都不碰**；
- 替换点：① `scrollContextViewerToMessage`（目录点消息头，`'start'`）② `applyHitCursor` 的检索跳转（`'nearest'`，
  仍保留"命中位于 `.hc-msg-body` / `.hc-msg-full` 内层滚动区时也能一起跳进视野"的旧能力——那两层现在由内层循环负责）。

**验证**：`node --check` 通过；`scratch/t9.test.mjs` 增至 **50 断言**（新增 E9a 只改查看器自身 `scrollTop = 476`、
E9b 假行上的 `scrollIntoView` **零调用**、E9c 文档滚动容器 **零写入**），`t8` 164 / `t7` 103 / `t6` 53 **无回归**（合计 **370**）；
真实 Chromium 夹具复核：点目录消息头 → `scrollTop 0 → 447.2`、行顶与容器顶对齐（offset 0）、高亮正常、`pageStayed/uiStayed` true、
连点两次幂等；检索"下一个"仍能滚进视野（`bodyScrollTop 71`、命中在体内可见）且页面不动。
**回归复核（嵌套滚动区仍要好用）**：整窗检索"下一个"连点 30 次（序号 `31/65`）→ 命中深处时**内层 `.hc-msg-body` 自己滚起来**
（首次非零 38 → 末次 787），当前命中在消息体内部**可见** ✓，外层容器 `scrollTop 165` ✓，而**页面始终 0**（`pageStayed true`）；
单条内检索"下一个"同样正常（`bodyScrollTop 210.4`、`curVisibleInBody true`）。

### 2.31-R2 `hide-helper-entry-popup` 落点收尾：上下都放不下时改"垂直居中"（用户实测反馈）

**现象**：条目详情子窗没有垂直居中，"向上偏移"了。

**根因**：`'above'` 落点的兜底链是「主窗正上方 → 主窗下方 → **贴顶 M=8px**」。ST 主弹窗的 `max-height: 90vh`，
统计页内容又长，所以主窗高度 ≈ 0.9×视口 —— 上方、下方都装不下（要容下"同高子窗 + 12 间距"得主窗高度 ≤ 约 0.45×视口），
于是**几乎总是走到贴顶那一支** → 出现在屏幕最上边缘，看起来就是"没居中的向上偏移"。

**修法**：把最后一支从"贴顶"改成**垂直居中** —— `top = max(M, round((vh − 子窗高) / 2))`
（`M=8` 仅用于"子窗比视口还高"的极端情况；水平仍与主窗左边缘对齐）。分支表：

| 条件 | 落点 |
|---|---|
| 主窗上方放得下（`主窗 top − 12 − 子窗高 ≥ 8`） | 主窗**正上方**（需求 2 的原行为，保持不变） |
| 上方放不下、主窗下方放得下 | 主窗**正下方**（原行为，不变） |
| 上下都放不下（**主窗接近 90vh 时的常态**） | **垂直居中**（新；原来是贴顶 8px） |

**验证（真实 Chromium 夹具，复刻 ST 的 `html{transform}` 与"主窗 0.9×视口"）**：
- 主窗矮（146）：条目详情 `left 8 / top 66`，即主窗正上方、间距 12 —— 与 R2 之前逐项一致（未回归）；
- 主窗高（606 = 0.9×674）：条目详情 `top 34 / left 8 / 高 606`，**中心与视口中心偏差 0px**、`gluedToTop=false`（不再是 8px）；
  从 wi-msg 打开与从「上下文目录」打开**两条入口一致**；
- 离线断言：`scratch/t9.test.mjs` → **51**（B2a `top=225` 即 (1000−551)/2 取整、B2b 明确断言 ≠ 贴顶 8px、B6 断言行中心 ≈ 视口中心 500），
  `t8` 164 / `t7` 103 / `t6` 53 无回归（合计 **371**）；截图 `scratch/round20-d-entry-centered-tallmain.png`。

### 2.32 第二十一轮：回退三窗布局 → 「单槽位 + 层级栈」（屏幕上恒为"主窗 + 一个子窗"）+ 子窗内「← 返回」（用户决定；已实施并真实 Chromium 实测）

**用户决定（原话）**："算了。三个窗口太分散注意力，还是两个窗口好些。但是我又感觉套入的子弹窗太多了？"
→ 追问后选定：**B 路线（最省改动：保留现有子窗，但"同时最多一个"）** + **贴主窗右侧（与主窗同尺寸）** +
**条目详情放进子窗内部（替换内容 + 「← 返回」）**。

**改动前的问题**：共 6 个窗口（主窗 + 条目详情 / 聊天统计 / 完整上下文 / 条目来源 / 上下文目录），最深链路 5 层
（主窗 → 统计 → 查看器 → 目录|条目来源 → 条目详情），**同时最多 4 个窗口**在屏幕上。用户观感 = "套娃太多"。

**实现（`hide/index.js`）**：
1. **落点回退**：`positionChildPopup` 删掉第二十轮加的 `'center'` / `'above'` / `'right-edge'` 三个分支与
   `CENTER_POPUP_IDS` / `isCenterModeActive()` / `entryPopupPlacement` / `entryPopupSide()` / `onCenterPopupClosed()`；
   五个子窗在 `childPopupTable()` 里**全部** `'right'`（贴主窗正右方、同宽同顶、放不下时整对平移、窄屏退化为居中显示）。
2. **单槽位 + 层级栈**（新增）：`CHILD_POPUP_IDS`（五个子窗）+ `childStack` + `visibleChildId()` / `focusChildPopup(id)` /
   `popChildPopup()` / `closeChildStack()` / `syncChildBackButtons()`。
   · 每个子窗的"打开路径"（`openChatStatsPopup` / `openContextViewer` / `openWiMsgPopup` / `openContextOutlinePopup` /
     `showEntryPopup`）**先** `focusChildPopup(自己)`：把"当前可见的别的子窗"隐藏并压栈 → 屏幕上恒为两个窗口；
   · `popChildPopup()`（= 点「← 返回」）：收起当前子窗、恢复栈顶子窗（按登记表重新贴合）；
   · 子窗的 ×/Esc/切聊天走 `closeChildStack()`：**清空**层级栈 → 回到"只有主窗"（不留"幽灵上一层"）；
     ⚠️ 只有用户主动关闭才清空；"打开新窗"这条路径**不能**清空（那是返回目标）。
3. **「← 返回」按钮**：五个子窗头部各加一个 `.hide-helper-popup-back`（`id = <子窗id>-back`，默认 `display:none`），
   由 `syncChildBackButtons()` 控制在"当前可见 + 栈非空"时显示，并给该子窗打 `hh-has-back` 类让标题行让位（`padding-left:64px`）。
4. **目录跳转改"进查看器"**（验证中发现的设计缺口）：目录与查看器共用一个槽位 ⇒ 点目录里的消息头时查看器其实**被压在栈里**，
   对着一个隐藏窗口滚动＝什么也看不见。`scrollContextViewerToMessage` 现在先 `focusChildPopup('hide-helper-context-viewer')`
   把查看器切到台前（目录自动进栈）再滚动 → 查看器左上角「← 返回」可回目录。

**验证中发现并修掉的两个真 bug**：
- `#hide-helper-ctx-outline-popup` 与 `#hide-helper-chat-stats-popup`、`#hide-helper-wi-msg-popup` 三个返回按钮的 id
  最初写成短名（`-back` 而非 `<子窗id>-back`）→ `syncChildBackButtons()` 查不到它们，返回按钮永远不显示（浏览器实测发现）；
- 目录跳转对着隐藏的查看器滚动（上面第 4 条）。

**验证（真实 Chromium 夹具 + 离线断言）**：
- 夹具新增 step：`child-slot`（当前可见子窗数 / 窗口数 / 是否贴右侧同顶同宽 / 每个返回按钮的存在与显示 / `hh-has-back`）、
  `child-back`（点返回 → 恢复上一层）、`pair-position`（贴右几何）、`entry-slot`（条目详情接管槽位 + 返回按钮显示），
  并把 `outline-open` 改为"贴右 + 查看器被压栈隐藏"；原 `entry-above`（正上方）随落点回退删除。
- ⚠️ 夹具修正：几何判定改用**内联 `style.left/top`**（自动化标签页里 `hhFadeIn` 入场动画**冻结在起始帧**，
  `transform: matrix(0.95…)` 会让 rect 缩小/偏移：实测主窗 inline left=558 而 rect.left=569、rect.w=399 → 拿 rect 判定"贴右/同顶"必然误报）。
- 实测（1600×900 与 1536×674 各跑一遍）：
  · **每一步 `visibleCount=1`、`windowCount=2`**（主窗 + 一个子窗）——"两个窗口"目标达成；
  · 打开查看器 → 打开条目来源（查看器被隐藏+压栈，**条目来源的「← 返回」出现**）→ 点条目（条目详情接管槽位、其它子窗全隐藏、
    返回按钮显示）→ 点返回 → 条目来源恢复 → 再点返回 → 查看器恢复（`restored: true`，窗口数始终 2）；
  · 目录路径：目录贴右（内核 `childAtRightOfMain/sameTop/sameWidth` 全 true）→ 点消息头 → **查看器切到台前**、滚动 `0 → 424.8`、
    目标行高亮、**页面 scrollY 始终 0** → 点返回 → 目录恢复；
  · 窄屏 700×1000：子窗居中式显示（`left 140 / top 224.5`）——老的窄屏行为未回归。
- 离线断言：`t9` **60**（A 组 14 条：单槽位 + 贴右 + 栈进出 + 主动关闭清栈；B 组 8 条：条目详情接管槽位 + 返回；
  E10a–d：目录跳转切台前/返回）+ `t8` **169**（J11–J14e：落点全 `'right'` + 单槽位四件套 + 五个返回按钮 + 样式齐备）
  + `t7` 103 + `t6` 53 = **385 断言全通过**；`node --check` 通过。

### 2.33 统计页第二十二轮（**A 方案**：目录 → 查看器内部左侧栏；条目来源 → 行内展开；窗口数仍恒为 2）

> ⚠️ **本方案已于第二十三轮整体回退**（用户实测"体验很差"）：目录与条目来源恢复为**独立子窗**，
> 并改为「父窗在左、子窗在右」的整对布局 —— 见 §2.34。本节保留作为"走过的弯路"记录，别再照它改回去。

**起因（用户口径）**：第二十一轮把屏幕压到"最多两个窗口"之后，"套娃"这一层其实还在 —— 目录与条目来源各自要**占用
那唯一的子窗槽位**（想打开它就得把查看器压进栈里）。本轮定论 **A 方案**：把这两样东西**收进查看器窗口内部**，
槽位只留给三个真正的窗口（条目详情 / 聊天统计 / 完整上下文）。

**① 「上下文目录」→ 查看器内部左侧栏（不再是窗口）**（`hide/index.js` + `style.css`）
- 查看器结构改为 `#hc-main`（横向 flex）= `#hc-outline`（左，目录栏）+ `#hide-context-body`（右，消息列表）；
  `#hc-outline` 默认 `display:none`，由工具栏 `hc-filters` **最右**的 `hc-wi-expand` 图标开/关
  （`toggleContextOutlineSidebar()` / `isContextOutlineOpen()` / `openContextOutlineSidebar()` / `closeContextOutlineSidebar()`）；
  打开时图标加 `.active`（新增 `#hide-helper-context-viewer .hc-icon-btn.active`）。
- 目录内容渲染逐字节沿用旧子窗那套：`renderContextOutlineList()` 只把目标从 `#hco-body` 换成 `#hc-outline-body`
  （全部消息的 `hc-msg-head`：#序号 / 角色 / 来源槽位 / tokens；世界书消息再列条目明细，`alwaysShowBook` + `markers:false`）。
- 目录栏内容区滚动条与查看器消息区同款（默认隐藏 + `.hh-sb-*` 三档显色，`initHiddenScrollbar`）。
  ⚠️ 同窗已有 `#hide-helper-context-viewer * { scrollbar-width: none !important }` —— 新增的 `scrollbar-*`
  **必须也带 `!important`**（`!important` 压过特异度），否则会被压成 `none`、三档显色全失效。
- `scrollContextViewerToMessage()` **删掉**第二十一轮那段 `focusChildPopup('hide-helper-context-viewer')`——
  目录栏就在查看器**内部**，"对着隐藏窗口滚动"的前提已消失；其余不变（被筛选藏住时先复位筛选、只经
  `scrollIntoViewWithin` 滚查看器自己的容器）。
- 目录栏的开/关是**查看器自己的界面状态**：换快照沿用、关查看器随窗口一起隐藏（不再需要单独 close）。

**② 世界书「条目来源」→ 行内展开（不再是窗口）**
- `ctxMessageRowHtml` 给**世界书来源**的消息行内加 `<div class="hc-msg-wi" data-i="${m.i}" style="display:none;"></div>`；
  点 `hc-msg-tokens` 里放大镜左侧的展开图标 → `toggleWiEntriesInline(i)`：就地 `renderWiEntriesInline(i)`
  （摘要行「条目来源 · N 条 · T t」+ 常驻/动态/EJS 分色图例 + 条目列表；行构建器与主面板同一套、排序固定「上下文顺序」、
  `markers:false`），点条目行 → `openEntryDetailPopup`（仍然进子窗槽位）。
- 互斥：`ctxWiInlineIndex` —— **同时只展开一条**（点另一条 = 换一条；点同一条 = 收起并清空）；
  列表整段重绘 / 关查看器 / 换快照时 `collapseWiEntriesInline()` 收起（并清 DOM）。
- 行点击守卫加上 `.hc-msg-wi`（点行内区不该冒泡成外层行的"展开/收起全文"）。
- **已知取舍（用户已接受）**：行内列表**不提供**「按世界书筛选 / 按类型筛选 / 条目检索」——一条消息里的世界书条目
  通常只有个位数、条目名已带 `(书名)`，换来的是不必为它多开一个窗口。
- ⚠️ **实测抓到的唯一真 bug（结构位置）**：`${wiBox}` 最初被插到 `.hc-msg-search`（默认 `display:none`）**内部** →
  DOM 与文案都在、`getBoundingClientRect()` 高度却是 **0**（截图里根本看不见）。修正为检索行的**兄弟节点**
  （`.hc-msg-body` 之前）；`t9` 新增结构不变式 D19 防回归。

**③ 删除两个旧子窗（彻底移除，不留死代码）**
- `index.js`：`CHILD_POPUP_IDS` 5→3、`childPopupTable()` 3 行、`WI_MSG_POPUP_MIN_HEIGHT` / `CTX_OUTLINE_MIN_HEIGHT`、
  `wiMsg*` 状态与 `ensureWiMsgPopup` / `openWiMsgPopup` / `closeWiMsgPopup` / `refreshWiMsgList`、
  `ensureContextOutlinePopup` / `openContextOutlinePopup` / `closeContextOutlinePopup` / `toggleContextOutlinePopup`、
  两个子窗的返回按钮与 `applyTheme` 选择器条目全部删除（层级链最长 3 层：聊天统计 → 完整上下文 → 条目详情）。
- `style.css`：删除 `#hide-helper-wi-msg-popup`（含 `.hwmp-*`、列表滚动条三档、暗色变量条目、`hh-has-back` 让位）
  与 `#hide-helper-ctx-outline-popup` 两整段；旧 wi-msg 子窗当年**镜像主面板**的那 60+ 条 `.tub-*` 选择器
  **原地改挂到 `#hide-helper-context-viewer`**（脚本替换 64 处、逐行核对）→ 行内条目来源与目录明细**直接复用**
  （不再单写一份，避免两份漂移）；`mark.hc-hit` 两处合并为单作用域。

**验证（真实 Chromium 夹具 `scratch/ctxviewer-fixture.html` + 4 套离线断言）**
- 夹具新增/改写 step：`wi-inline-open`（默认收起且为空 → 就地展开：摘要/图例/条目行/书名/无标记行/图标 active/区块在行内）、
  `wi-inline-switch`（互斥）、`wi-inline-entry`（点行内条目 → 条目详情，`windowCount=2`、返回按钮出现）、
  `wi-inline-entry-back`（返回后查看器恢复、**行内仍开着**）、`wi-inline-collapse`（收起并清空）、
  `wi-inline-click-guard`（点行内区不触发行展开）、`outline-open`（目录栏在查看器**内部**：`insideViewer` / `leftOfList` /
  `listNarrowed` 全 true、9 条消息头、3 条条目行、0 条标记行、两个旧弹窗均不存在）、`outline-click`（滚动生效、
  **页面 scrollY 恒 0**、`uiStayed`）、`outline-click-filtered`（筛选改用 `assistant` —— 夹具里 `#5` 是 `system`，
  用 `system` 筛选藏不住它、等于没测到"自动复位"路径）、`outline-close`（收起后消息列表宽度回到 350、查看器仍在）、
  `child-slot` / `child-back` / `pair-position` / `entry-slot`（ID 表 5→3）。
- 实测（1536×674）：`wi-inline-open` → 行内区块 **321×117**、父元素 = `.hc-msg`（不再是 `.hc-msg-search`）、
  正文区紧随其后 4px；目录栏 `1025..1180`（查看器 `1001..1400`）与消息列表 `1190..1375` 并排、不重叠；
  目录栏滚动条默认 `rgba(0, 0, 0, 0)`（隐藏）；每一步 `visibleCount=1` / `windowCount=2`；
  截图 `scratch/round22-sidebar-inline.png`。
- 离线断言：`t9` 重写为 **81**（A 单槽位/贴右/窄屏 · B 条目详情接管槽位 · C 行构建器口径 · D 行内条目来源 19 条 ·
  E 目录侧栏 + 跳转不滚页面 22 条 · F 静态接线 / CSS 12 条）；`t8` 的 J / K 组改为**新口径**（**164**）；
  `t7` 103 + `t6` 53 → **401 断言全通过**；`node --check index.js` 通过；两个源文件保持 CRLF、无 BOM。

### 2.34 统计页第二十三轮（**父窗在左、子窗在右**；第二十二轮 A 方案整体回退）

**起因（用户口径，2026-10-16）**
- 「算了，还是改回上一轮的方案吧。A方案体验很差。」→ 第二十二轮把「上下文目录」搬进查看器内部侧栏、
  把「条目来源」改成行内展开，**全部回退**：两者恢复为独立子窗（与第二十一轮一致）。
- 「我希望实现上一级父弹窗在左侧，它的子弹窗在右侧。而不是左侧一直显示插件的主弹窗。」
  → 追问确认走 **A 选项**：屏幕上摊开的永远只有两个窗口，**左槽 = 当前窗的直接父窗、右槽 = 当前窗**；
  父窗不是插件主弹窗时，**主弹窗隐藏**（不再"左侧一直显示主弹窗"）。

**① A 方案整体回退（第二十二轮的改动全部撤销）**
- `hide/index.js`：恢复 `#hide-helper-wi-msg-popup`（世界书条目来源：head + 饼图 + 条目列表 + 检索 + 类型过滤）与
  `#hide-helper-ctx-outline-popup`（上下文目录：全部消息头 + 世界书条目明细）两套子窗代码；
  `CHILD_POPUP_IDS` 回到 5 个、`childPopupTable()` 回到 5 行（两条最小高度）；`ctxMessageRowHtml` 去掉行内容器、
  恢复"打开子窗"的展开图标；`bindContextRowEvents` 改回调 `openWiMsgPopup(i)`；查看器标记去掉 `#hc-main` / `#hc-outline`
  （消息列表回到直接子节点）；`applyTheme` 重新覆盖两个子窗；**删除** `toggleWiEntriesInline` / `renderWiEntriesInline` /
  `collapseWiEntriesInline` / `syncCtxWiExpandButtons` / `toggleContextOutlineSidebar` / `open|closeContextOutlineSidebar` /
  `isContextOutlineOpen` 等 A 方案函数。
- `hide/style.css`：**按"改动前备份"恢复** —— 实测 `diff` 只剩两处差异：① 尾部那段注释块按本轮口径改写；
  ② 文件末尾新增两个 z-index 类。A 方案新增的 CSS（`#hc-main` / `#hc-outline` / 行内展开 / 查看器内部目录栏）**全部消失**，
  两个子窗整套样式与 `.tub-*` 镜像规则原样回来（108248 字节 / 1236 行 ⇒ **108745 字节 / 1242 行**）。

**② 新的整对布局（父左子右）**
- 新增/改写（`hide/index.js`）：
  · `PAIR_GAP = 12` / `PAIR_MARGIN = 8`；状态：`pairCurrentId`（右槽窗）、`pairMainSlot`（主窗自己的槽位缓存）、
    `pairMainShift`（整对左移时主窗 left 的还原信息）、`pairMainPrevDisplay`（主窗被隐藏前的 inline display）；
  · `captureMainSlot()`：主窗**可见**时随手记下它自己的 `{left, top, width, height}` —— 用 inline `left/top` + `offsetWidth/Height`，
    ⚠️ **不能用 rect**（入场动画 `hhFadeIn` 的 `scale(0.95)` 会让 rect 偏小，实测主窗 inline 558 而 rect 569）；
  · `currentChildId()` / `parentChildId()`（栈顶 = 直接父窗）/ `visibleChildIds()`（父 + 当前）；
    `visibleChildId()` 保留旧名，语义＝**当前窗**（右槽），父窗不算；
  · `focusChildPopup(id)`：把"当前窗"压栈（它成为新窗的父窗、留左槽），并把 `pairCurrentId` 指向新窗；
  · `layoutChildPair()`：**唯一**的落点决定处 —— 可见集合只留「父窗 + 当前窗」（其余子窗，以及"不是父窗时的主窗"，
    一律隐藏；隐藏/还原主窗走 `setMainHiddenForPair()`，按原 inline display 精确还原）；左槽 = 主窗自己的槽位、
    右槽 = 左槽 + 宽 + 间距；放不下时整对左移（改写主窗 left 前先记原值 `pairMainShift`，重排/关闭时还原）；
    窄屏（`2 × 宽 + GAP` 放不下）→ 两窗居中叠放并给当前窗打 `.hh-pair-front`（父窗 `.hh-pair-under`）；
  · `popChildPopup()` / `closeChildStack()` 都走 `layoutChildPair()` 重排；`syncChildBackButtons()` 只给**当前窗**
    且层级非空时显示「← 返回」。
- **删除**旧机制：`childPopupShifts` / `recordMainShift` / `releaseChildPopupShift` / `repositionOtherChildPopups`、
  `positionChildPopup` 的 `side` / `minHeight` / `allowShift` 参数与"贴主窗右侧 + 位移还原"逻辑
  （`positionChildPopup(el)` 现在只剩"登记当前窗 + 整对重排"的入口语义）。
- `hide/style.css` 末尾新增两条（子窗自带 `z-index:10000 !important`，内联值压不过它，必须用类 + `!important`）：
  `.hh-pair-under { z-index: 10000 !important; }`、`.hh-pair-front { z-index: 10001 !important; }`。

**验证（真实 Chromium 夹具 `scratch/ctxviewer-fixture.html` + 4 套离线断言）**
- 夹具新增 step：`pair-l1`（父 = 主窗：左槽主窗、右槽查看器、主窗可见）、`pair-l2`（从查看器开条目来源：**主窗隐藏**、
  查看器在左槽、条目来源在右槽、`windowCount=2`、返回按钮只出现在当前窗）、`pair-l3-entry`（再开条目详情：左 = 条目来源、
  右 = 条目详情，查看器与主窗都隐藏）、`pair-l3-back` / `pair-back-to-main`（逐级返回，最后一层主窗重新出现）、
  `pair-narrow`（700×1000 → 两窗居中叠放、当前窗带 `hh-pair-front`）；
  改写 step：`child-slot` / `pair-position` / `entry-slot` / `child-back` 一律按 `currentChildId()` / `parentChildId()` 判定
  （旧版按"ID 表里第一个可见窗"判 —— 父窗 id 排在前面时判错，实测踩过）；`outline-*` 恢复子窗版并断言父左子右；
  恢复 step：`openwi` / `wi-list-check` / `wi-filter-type` / `wi-search` / `wi-entry-click` / `closewi`。
- 实测（1536×674）：L1 主窗 `left=558` + 查看器 `990`（主窗可见）；L2 查看器 `558` + 目录/条目来源 `990`（**主窗 hidden**）；
  L3 条目来源 `558` + 条目详情 `990`；每一步 `visibleCount=2` / `windowCount=2`（父 + 当前）；
  窄屏 700×1000 两窗都 `left=140` 叠放、当前窗 `.hh-pair-front`；目录跳转后页面 `scrollY` 恒 0；
  截图 `scratch/round23-parent-left-child-right.png`。
- ⚠️ 夹具自身修了两处"量不出来"的坑：① 主窗替身原先用 `display:flex !important` —— 会压过我们隐藏主窗的内联 `display:none`，
  改成内联 `style="display: flex;"`（否则"主窗被隐藏"根本复现不出来）；② "与主窗同宽"的断言在主窗被隐藏时会量到 `0`，
  新增 `pairRefWidth()` 改量当前对里另一个可见窗。
- 离线断言：`t9` 重写为 **86**（A 登记表 / 最小高度 · B L1 父左子右 · C L2 主窗隐藏 · D L3 与逐级返回 · E 窄屏叠放 + z-index 类 ·
  F 行构建器口径 · G 条目来源子窗内容 · H 目录子窗 + 跳转不动页面 · I 静态接线 + CSS）；`t8` 的 I / J / K 组改为新口径（**160**）；
  `t7` 103 + `t6` 53 → **402 断言全通过**；`node --check index.js` 通过；`index.js` / `style.css` 保持 CRLF、无 BOM。

### 2.34-R1 第二十三轮复验修的两处「叠放层级」缺陷（窄屏"当前窗压父窗"原本**没生效**）

复验方式：真实 Chromium 夹具（`scratch/ctxviewer-fixture.html`）里逐窗读 `getComputedStyle(el).zIndex` ——
**不再只看类名**（这次两个缺陷都"带着正确的类名"却是错的）。

1. **CSS 特异性**：`.hh-pair-front { z-index: 10001 !important; }` 是**类选择器**，而五个子窗各自带
   `#hide-helper-xxx { z-index: 10000 !important; }`（**id 选择器**）；两条同为 `!important` 时按特异性比大小 → id 胜
   ⇒ 实测父子两窗计算值都是 `10000`，"当前窗在上"完全没生效（当时只是靠 DOM 顺序歪打正着）。
   **修法**：两条规则改成 **id 级选择器**（`#hide-helper-entry-popup.hh-pair-front, …` 六个 id 一组，含主窗），
   front = `10001` / under = `10000`。复验（窄屏 700×1000）：父窗 `10000` / 当前窗 `10001`、`frontWins = true`。
2. **陈旧类残留**：`layoutChildPair()` 原先只对"这一次的父窗 / 当前窗"做 `classList.toggle('hh-pair-front', !fitsWide)` ——
   曾在窄屏当过"当前窗"的子窗会**永久**带着 `hh-pair-front`；下次叠放它若成了**父窗**，反而压住真正的当前窗
   （实测：查看器同时带 `hh-pair-front` + `hh-pair-under` → 计算值 `10001`，父窗压住当前窗）。
   **修法**：新增 `clearPairStacking(mainEl)`；`layoutChildPair()` 开头先把两个类从**主窗 + 全部子窗**（含此刻隐藏的）清掉，
   末尾只在 `!fitsWide` 时 `add`（不再 toggle）。
3. 夹具桩同步修正：`scratch/t9.test.mjs` 的 `mkClassList()` 原先把 `add/remove` 写成**单 token**（真实 DOM 支持多 token）
   ⇒ `classList.remove('hh-pair-front', 'hh-pair-under')` 只生效一次，出现 E2b 假失败；改成变参 `(...cs)`。
   离线断言：`t9` **88**（新增 E2b「陈旧类必须全清」+ I9/I9b「z-index 规则必须是 id 级选择器、不得回退成纯类写法」），
   `t8` 160 + `t7` 103 + `t6` 53 → **404 全通过**；截图 `scratch/round23-r1-narrow-overlap.png`（窄屏叠放，当前窗在上）。
- ⚠️ 结论（写进判据）：**"叠放谁在上"必须量 `getComputedStyle(...).zIndex`**，类名与内联值都不足以证明层级正确。
- 备注（行为未改，留给用户决定）：子窗 **×** / `Esc` 仍是第二十一轮定的"结束这轮层级"语义（→ 回到只有主窗），
  而不是"只关当前这一层、父窗留下"。要改成"只关当前层"只需把各 `closeXxxPopup()` 里的 `closeChildStack()`
  换成 `popChildPopup()`（一处一行，本轮未动）。

### 2.35 统计页第二十四轮（**父窗原位 + 子窗在"另一侧"**；子窗「← 返回」整套删除）

**起因（用户口径，2026-10-16）**
- 「很好，但是还是有点差强人意。我希望子弹窗是显示在另一侧的，而不是上一级父弹窗移至左侧，子弹窗显示在右侧这种逻辑。」
- 「另外，把子弹窗的"返回"按钮去除了，这个没什么用。」
- 追问"父窗正好居中时子窗开哪边" → 用户答："主弹窗的位置不用管。你可以把他当做最高一级的父弹窗。"
  ⇒ 主窗**不再被移动到任何槽位**，它就是最高一级父窗（位置由插件自己管，永不改写）。

**① 落点规则重写（`layoutChildPair()`，`hide/index.js`）**
- **父窗一动不动**：父窗 = 主窗时用 `captureMainSlot()`；父窗 = 子窗时用新增的 `readWindowSlot(parEl)`
  （inline `left/top` + `offsetWidth/Height`；⚠️ 仍**不用** `getBoundingClientRect` —— 入场动画会缩放 rect）。
- **子窗开在父窗的另一侧**：`parentOnLeft = (slot.left + w / 2) <= vw / 2` → 往右，否则往左（逐层自动左右交错）；
  首选那侧放不下 → 改用另一侧；两侧都放不下 → 贴着父窗（父窗在视口内即完全重叠）并把当前窗 `clampIn` **钳进视口**。
- 子窗与父窗**同宽同顶**（`curH = max(父窗高, 该子窗最小高度)`，更高时只上移必要量）；`.hh-pair-front/under` 只在重叠时打
  （判据仍是 `getComputedStyle().zIndex`，见 §2.34-R1）。
- **删除"整对左移"**：`pairMainShift` / `restoreMainShift` 与"把父窗 `place()` 到左槽"的代码全部删掉。

**② 子窗「← 返回」整套删除**
- 5 个子窗标记里的 `<button class="hide-helper-popup-back">` 与各自的 `#…-back` 点击绑定全部删除；
  `syncChildBackButtons()`（13 处调用）与 `popChildPopup()` 一并删除。
- `style.css`：`.hide-helper-popup-back`（含 `:hover`）与 `.hh-has-back` 的三条"标题行让位"规则删除。
- 关窗（× / `Esc` / 切聊天）仍是第二十一轮定的"结束这轮层级"语义（`closeChildStack()` → 回到只有主窗）。

**③ 验证（真实 Chromium 夹具 + 四套离线）**
- 宽屏 1536×674：L1 `主窗 558 | 查看器 990`；L2 `查看器 990（**原位不动**）| 条目来源 558`；
  L3 `条目来源 558（**原位不动**）| 条目详情 990`；每步 `windowCount = 2`、页面 `scrollY` 恒 0、`.hide-helper-popup-back` 计数 0。
- 窄屏 700×1000：两窗重叠且当前窗在上（`z-index` 10001 > 10000）；强制 90% 宽度的"手机态"：父窗 630 宽、子窗完全重叠、父窗仍在原位。
- 离线断言：`t9` 重写为 **96**、`t8` **160**、`t7` 103、`t6` 53 → **412 全通过**（`node --check index.js` 通过）。
- 截图：`scratch/round24-parent-stays-child-other-side.png`（L1：主窗 558 不动 + 子窗 990）。
- 夹具新增 step：`pair-close-stack`（× → 结束层级）、`pair-overlay`（强制"两侧都放不下"）；改写 `pair-l1` / `pair-l2` /
  `pair-l3-entry` / `pair-narrow` / `child-slot` / `child-close`（原 `child-back`）/ `pair-position` / `entry-slot` /
  `outline-open` / `openwi` / `outline-close`；新增助手 `childOnOtherSide(child, parent)` 与基线 `window.__fxMainLeft0`。
- ⚠️ 备注：§2.34-R1 结尾那条"想改成只关当前层就换 `popChildPopup()`"**已失效**（该函数本轮删除）；若以后要恢复"逐级回退"，
  得重新引入栈回退逻辑 —— 目前用户明确说返回按钮没用。

### 2.36 统计页第二十五轮（**关子窗只退一层**：不再连父窗一起关）

**起因（用户口径，2026-10-16）**：「有一个非常影响体验的逻辑需要修改。就是关闭子弹窗不要连同把上一级的父弹窗给关闭了。」

**① 新增"只退一层"的关闭入口（`hide/index.js`）**
- `closeChildPopupAt(id)`：关闭某个子窗 → **退回它自己的父窗**（父窗成为新的当前窗）：
  - 它就是"当前窗" → `pairCurrentId = childStack.pop()`（退一层）；
  - 它是**父窗**（它在栈里、上面还压着更深的子窗）→ 截掉它和压在它上面的那些，当前窗换成它的父窗；
  - 它本来就是最外层子窗 → 回到"只有主窗"。
- `popChildPopup()` = `closeChildPopupAt(当前窗)`。
- ⚠️ **本轮实测抓到的真 bug**：退层后新的"当前窗"DOM 仍是 `display:none`（它是被上一层压着隐藏的），
  而 `currentChildId()` 把 `display:none` 的窗口当成"没有当前窗" ⇒ 整对被当成"只剩主窗"、父窗根本没回来。
  修法：`closeChildPopupAt` 里先把新的当前窗置 `display='flex'` 再 `layoutChildPair()`。
- 五个子窗的关闭路径（× / `Esc` / 图标 toggle）全部改走 `closeChildPopupAt`：
  `closeEntryPopup` / `closeChatStatsPopup` / `closeContextViewer` / `closeWiMsgPopup` / `closeContextOutlinePopup`。
  其中 `closeContextViewer` 对「条目来源」「目录」改成**直接隐藏 + 清状态**（若调它们自己的 close*，会各退一层、把查看器的父窗也带下去）。
- **`Esc` 同样改成"关当前窗（退一层）"**：`keydown.hideHelperEsc` 里若有子窗开着 → `popChildPopup()`；没有子窗才关主弹窗。
  原先挂在「聊天」统计子窗上的那条全局 Esc 监听删除（由统一处理器负责）。
- **"重置"路径仍然"一次清空整轮层级"**（继续用 `closeChildStack()`，共 5 处）：关主弹窗 / 换页签 / 切聊天 /
  `updateTokenStatsUI` 重绘前 / 换快照。

**② 验证（真实 Chromium 夹具 + 四套离线）**
- L1（主窗 + 查看器）→ L2（查看器 + 条目来源）→ L3（条目来源 + 条目详情）→ 点条目详情的 × →
  **回到 L2**（查看器 + 条目来源仍在屏幕上、主窗仍隐藏）→ 再 × → L1（主窗 + 查看器）→ 再 × → 只有主窗。
- 目录 × → 查看器成为新的当前窗（查看器仍在屏幕上）；条目来源 × → 查看器仍在；`windowCount` 逐层正确；页面 `scrollY` 恒 0。
- 离线：`t9` **101**（新增"关当前窗退一层 / 关父窗连带其上子窗 / 重置一次清空"三组断言）、`t8` **161**、
  `t7` 103、`t6` 53 → **418 全通过**；`node --check index.js` 通过。
- 截图：`scratch/round25-close-child-keeps-parent.png`（关掉子窗后父窗还在）。
- 夹具：`pair-close-stack` 拆成 `pair-close-one`（关当前窗 → 退一层）与 `pair-close-to-main`（逐层关到只剩主窗）；
  `child-close` / `closewi` / `closeviewer` / `outline-close` 都改为断言"父窗留下 + 谁是新的当前窗"。

### 2.37 开发体验：快照落盘 + 三条"免重新生成消息"的入口

**起因（用户口径，2026-10-16）**：「每次我需要手动查看效果时，都要重新生成一遍消息，以便显示"完整上下文"，
这在现在的开发测试阶段非常麻烦。」

**① 快照落盘（localStorage 持久副本）**
- `saveContextDump` 现在同步写一份持久副本到 `localStorage['hide-helper.contextDumps.v1']`（新函数 `persistContextDumps`）；
  插件初始化时 `loadPersistedContextDumps()` 恢复 —— **刷新页面 / 切聊天回来都还在**（改完 `style.css` 刷新即可直接复看，
  不用再重发一条消息）。
- 保留策略：内存仍然是 `CONTEXT_DUMP_KEEP = 3`（LRU 语义不变）；持久副本上限 4 MB，超了从最旧的丢；
  配额写爆 / 隐私模式 → 打一条 warn 并退化成纯内存（`contextDumpPersistOff`）。**依然不写入聊天文件**。
- `CHAT_CHANGED` **不再清空快照**（store 以 `${chatId}|${anchorId}` 为键，切回来还在）。

**② 三条"免重发"的入口**
- **`refreshContextSnapshot()`**：用当前上下文**静默跑一次模拟发送**（= 点「模拟发送」但跳过输入弹窗），
  快照一落库就自动打开「完整上下文」查看器 —— 内容与真实发送一致（含世界书 / EJS 渲染），不消耗 token、不新增消息。
- **`openLatestContextViewer()`**：直接打开本聊天**最近一份**快照（配合持久化，刷新页面后也在）。
- **`makeDemoContextDump()`**：塞一份**合成快照**（7 条：main / 世界书前置+后置（带条目明细）/ AN / 三条聊天，含 40 行长正文）
  并打开查看器 —— **完全不跑管线、不需要任何发送**，纯看 UI / 布局 / 折叠 / 目录用。
- UI 入口：查看器空态 = 「立即生成快照（模拟发送）」按钮；「聊天」统计子窗底部在没有快照时 = 「没有快照 · 点此生成」按钮
  （文案明确写"**不用重新生成消息**"）。
- 控制台入口：`hideHelperDev.snapshot()` / `.openViewer()` / `.demo()` / `.clear()` / `.latest(chatId)`。
- 环境兼容：新增 `notify(kind, msg)`（`toastr` 不存在时静默跳过），让这些入口在离线夹具 / 单测里也能跑。

**③ 验证**
- 真实 Chromium（夹具）：`demo()` → 7 条消息 / 1576 tokens、查看器 `display:flex`、正文渲染 7 行 + 世界书展开图标存在、
  localStorage 写入 8089 B；清空内存 → `loadPersistedContextDumps()` 恢复 2 份（含 demo）；空态文案与按钮正确；`clear()` 清干净。
- 离线：`t9` **112**（新增 J 组 11 条：持久化写入 / 恢复、demo 快照、空态按钮、`hideHelperDev` 暴露）、
  `t6` 53（E7 文案跟进）、`t8` 161、`t7` 103 → **429 全通过**；`node --check index.js` 通过。

---

### 2.38 统计页第二十七轮（**「上下文目录」= 目录页 + 新增「完整上下文·合并内容」= 内容区**；「条目来源 → 条目详情」三窗例外）（本轮新增；已实施并离线 + 真实 Chromium 实测）

**用户口径（原话）**

> ① 「当在 `hide-helper-wi-msg-popup` 弹窗中打开 `hide-helper-entry-popup` 弹窗时，弹窗显示在 `hide-helper-wi-msg-popup` 弹窗的左侧，而不是显示在右侧。此时，网页会显示左、中、右三个弹窗，这是符合预期的。」
> ② 「当在 `hide-helper-context-viewer` 打开 `hc-wi-expand-all` 的 `hide-helper-ctx-outline-popup` 弹窗时，该弹窗显示在左侧。同时，右侧要出现一个同层级的弹窗，弹窗显示展开的完整上下文合并内容。点击左侧的 `hide-helper-ctx-outline-popup` 的 `tub-nli-label` 或 `hco-item` 选项时，右侧内容可以直接滚动到对应的位置。你可以将 `hide-helper-ctx-outline-popup` 当做目录页，而「完整上下文合并」当做内容区域。」

追问后选定：② 「两个：目录(左) + 合并内容(右)，查看器收起」；① 「只在这一步保留，其它步骤维持现在的两个窗口」；
③ 「右侧的内容区域是显示的发送的完整上下文字符。它们的内容合并成一起，不再以「条目」隔开了……因此就滚动到对应的内容首行就行了。」

**① 场景一：三窗例外（`pairExtraId`）**

- `layoutChildPair()` 新增 `pairExtraId`：从「条目来源」里点条目开「条目详情」时（`openEntryDetailPopup` 判 `currentChildId() === 'hide-helper-wi-msg-popup'`），
  把**条目来源的父窗**（通常是完整上下文查看器）也留在屏幕上 ⇒ 「条目详情 | 条目来源 | 查看器」三窗并存（屏幕上三个窗口的唯一例外）。
- 落点改为**开在 extra 的对面**：`goLeft = (extra 中心 > 父窗中心)`；没有 extra 时仍是原规则（父窗中心在视口右半边 → 往左，否则往右）。
  这样三窗天然互不重叠（既不会压住"额外保留的那一层"，也不会压住父窗）。
- `pairExtraId` 的清空时机：`focusChildPopup`（任何一次新的打开都回到两窗语义）、`closeChildPopupAt`、`closeChildStack`、
  以及 `layoutChildPair` 的"没有当前窗"分支；`showEntryPopup({ keepId })` 在 `focusChildPopup` **之后**再设回它。
- 实测（夹具 1536×674）：`条目详情 126 | 条目来源 558 | 查看器 990`，三窗 420×551 互不重叠、页面 scrollY 恒 0；
  关条目详情 → **只退一层**回到「条目来源 + 查看器」两窗（第三窗只在条目详情开着时才留）。

**② 场景二：目录页 + 合并内容（成对窗口）**

- **新增窗口** `#hide-helper-ctx-merged-popup`（常量 `CTX_MERGED_ID`）：`ensureContextMergedPopup()` / `renderMergedContext()`。
  内容是**这次真正发送出去的完整上下文字符**：逐条槽位的 `text` 首尾相接渲染成 `<pre class="hcm-chunk" data-i="N">`，
  **无标题 / 无边框 / 无外边距** ⇒ 看起来就是一整篇连续文本（不再按"条目/消息"分隔）；`.hcm-chunk` 只当目录跳转的定位锚。
- **成对模式**：点查看器工具栏 `#hc-wi-expand-all` → `openContextOutlinePopup()` 先确保查看器在台前（不在就 `focusChildPopup` 切上来，它要给槽位）
  → `enterContextOutlineMergeMode(viewerEl)`：**先** `focusChildPopup(CTX_OUTLINE_ID)`（把查看器压栈，当 × / `Esc` 的退回目标）、**再**收起查看器并记下它的槽位
  → 目录与合并内容一起显示。`ctxMergeMode` / `ctxMergeViewerSlot` 两个状态。
- **落点** `layoutOutlineMergePair()`（`layoutChildPair()` 在成对模式下**直接转交**给它，是这两窗唯一的摆位权威）：
  ① 首选：合并内容**沿用查看器原来的槽位**（视线不用挪），目录贴在它**左侧**；
  ② 左边放不下：**目录留在查看器槽位、合并内容挪到它右侧**（"目录在左、内容在右"这条不变）；
  ③ 两侧都放不下（窄屏）：两窗重叠、目录压上层；主弹窗在本模式里一律让位（屏幕上就是这两个窗口）。
- **关闭是一对**：目录 × / 合并内容 ×（`closeContextMergedPopup()` → `closeContextOutlinePopup()`）/ 再点图标 / `Esc`（`popChildPopup()` → `closeChildPopupAt(CTX_OUTLINE_ID)`）
  → `exitContextOutlineMergeMode()`（清状态 + 收起合并内容窗）+ 退回查看器（`closeChildPopupAt` 的"新当前窗"逻辑把查看器亮回来）。
  `closeChildStack()` / `closeContextViewer()` 也一并清该模式（重置路径不留残窗）。
- **跳转**：目录里的消息行（`hco-item` / `hco-msg-head`）→ `jumpContextOutlineToMessage(i)` 滚到该条消息在合并内容里的**第一行**；
  条目行（`tub-nli-label` 所在整行）→ `jumpContextOutlineToEntry(i, {book,name,uid})` 滚到**该条目正文的第一行**——
  开窗后 `fetchWorldEntryDetail` 取条目正文，`findEntryOffsetInText(text, content)` 三级放宽定位（整段 → 第一行非空行 → 前 60 字符），
  再用 `document.createRange()` 量出该字符处的纵坐标，**只滚内容区自己**（`scrollIntoViewWithin`，绝不 `scrollIntoView`——
  否则会连带滚文档、整个插件 UI 跟着整页上移）；定位不到就退回"该条消息的第一行"。两处都加 `.hcm-chunk-target` 高亮（与查看器 `.hc-msg-target` 同款）+ 目录 `.hco-item-active`。
- ⚠️ **行为变更**（用户明确要求）：目录里的**条目行**由"打开条目详情子窗"改成**导航**（条目详情仍可从「条目来源」子窗打开）；
  行提示语改由 `createEntryRowBuilder({ rowTitle })` 提供（主面板 / 条目来源子窗仍是默认的"点击查看条目详情"）。
- 实测（夹具 1536×674）：`目录 558 | 合并内容 990`（= 查看器原槽位）各 420×551、无重叠、页面 scrollY 恒 0；
  点目录里的消息行 → 内容区 `scrollTop` 跳到该条消息处（例：`#5` → 2469、`#0` → 2），**目标行贴在容器顶（残差 ≤9px；重复点同一条结果一致）**、
  `.hcm-chunk-target[data-i="N"]` 跟着走；点 0 号消息里的两个条目 → 内容区滚到**该条目正文**在发送文本里的那一行
  （第 195 字符 = 第 6 行 / 第 780 字符 = 第 21 行，残差 5px）——两条落在不同位置，证明是"滚到**该条目正文**首行"而不是"滚到消息开头"；
  ⚠️ 跳转是**两遍收敛**（先滚一次、再按残差纠一次）：真实 Chromium 实测首遍会偏 13~33px（从别处滚过来时 rect 与 scrollTop 还没对齐）、
  消息跳转首遍甚至偏 100px 以上（`#0` 停在 124 而不是 2），第二遍**重新量残差**才收干净（`scrollMergedToOffset` / `scrollMergedToElement`）；
  ⚠️ 夹具的世界书桩正文**必须真的出现在消息文本里**才会走"条目正文"这条精确路径（原先 `content: 'ELDORIA-LINE-005 ELDORIA-LINE-006 …'`
  与 `LONG_WI` 的文本不符 → 一直走"退回消息首行"的兜底，量到的数看着也对、其实没测到核心逻辑；已按本条修正夹具）；
  关掉任一窗 → 查看器回到台前、`merge.on=false`、屏幕上只剩「主窗 + 查看器」。

**③ 其它改动**

- `CHILD_POPUP_IDS` 5 → **6**（加 `hide-helper-ctx-merged-popup`）、`childPopupTable()` 6 行（新窗最小高度 0）；`applyTheme` 的主题目标选择器加新窗 id。
- `style.css`：新增 `#hide-helper-ctx-merged-popup` 整套（关闭按钮 / 头部 / 内容区滚动条"默认隐藏 + 滚动·悬停·拖拽三档"`!important` 成对写法 / `.hcm-text` / `.hcm-chunk` / `.hcm-chunk-target`）、
  暗色变量选择器列表 + `.hh-pair-front/under` 的 id 级规则各加一行；内容区补 `overflow-anchor: none`（跳转是我们**显式**写 `scrollTop` 的，
  浏览器自动锚定会在重排时把位置挪走）；目录子窗段的注释按新口径改写。

**④ 验证**

- 离线（DOM shim 加载真实 `index.js`）：`t9` **130**（H 组按成对模式改写 + 新增 H8b/H8c、H9/H10 内容区结构、H11–H18 跳转与高亮、H19/H20 关窗一对、K 组三窗例外、K5–K9 `findEntryOffsetInText` 纯函数）
  + `t8` 161 + `t7` 103 + `t6` 53 = **447 全通过**；`node --check index.js` 通过。
- 真实 Chromium（`scratch/round27-fixture.html` = 真实 `index.js` + 真实 `style.css`，只桩 ST 环境）：两个场景的落点、可见集合、无重叠、页面 scrollY 恒 0、跳转 scrollTop 与高亮、
  "关一窗收起一对并回到查看器"逐项量测通过；截图 `scratch/round27-outline-merged.png`（目录 + 合并内容）、`scratch/round27-three-windows.png`（三窗）。

### 2.38-R1 第二十七轮修订（用户实测两条：换条目不跳位 + 目录跟随滚动 + 分块浅灰底）

**用户口径（原话）**

> ① 「不同条目的 `hide-helper-entry-popup` 弹窗切换时，该弹窗要显示到原有的位置，而不是切换到右侧或者其他位置。」
> ② 「在 `hide-helper-ctx-merged-popup` 滚动到对应的位置时，左侧的 `hide-helper-ctx-outline-popup` 也要滚动显示到对应的选项。同时，我希望在右侧每个 `hcm-chunk` 增加浅灰色的背景，区分不同的来源文本。」

**① 换条目不动位置 / 不动层级**（`showEntryPopup`）

- 根因：`keepId` 只在"当前窗是「条目来源」"时才算得出来。弹窗**已经开着**时再点另一条条目，`currentChildId()` 是**条目详情自己** →
  `keepId = null` → `pairExtraId` 被清空（第三窗被丢掉）→ 落点退回"父窗落在视口哪半边"的判据 → 弹窗从最左（126）**跳到右侧**（990 一带，压在查看器上）。
- 修法：`showEntryPopup()` 先记 `sameWindow = currentChildId() === 'hide-helper-entry-popup'`；是同一个窗（= 只是"换一条条目"）时
  **沿用原来的 `pairExtraId`**（`prevExtra`），位置仍由同一套（父窗槽位 + extra 槽位）关系算出 → 与原位一致；`el.style.display = 'flex'` + 重排都不动层级。
- 实测（夹具 1536×674）：三窗 `条目详情 126 | 条目来源 558 | 查看器 990`；在「条目来源」里点**另一条**条目前后，
  三窗的 `left/top` **逐个字节不变**（126/558/990）、无重叠、页面 scrollY 恒 0。

**② 内容区 → 目录联动（scroll-spy）**（`scheduleOutlineSpy` / `syncOutlineFromMergedScroll`）

- 判定：容器顶下方**第一条"还看得见"的消息块**（`chunk.bottom - body.top > 4`）就是当前条；**只有当前条变了**才动目录（否则每帧都滚一下会抖），
  高亮改用 `syncOutlineActiveHead(i)`，目录自身用 `scrollIntoViewWithin(#hco-body, item, 'nearest')` 滚到对应选项 —— **只滚目录自己的容器**（绝不碰文档）。
- ⚠️ 程序化跳转（点目录）后 **250ms 静默期**（`ctxOutlineSpySuppressUntil`）：点最后一条时滚动被 `maxTop` 钳住、容器顶会停在倒数第二条里，
  没有静默期目录会被立刻改回上一条（跟用户刚点的选项打架）。之后用户再手动滚动，目录照常跟随"实际可见的那条"。
- ⚠️ 调度用 **rAF + `setTimeout(60)` 双保险**（不是二选一）：后台/隐藏标签页里 rAF 被节流、`scroll` 事件甚至**根本不派发**
  （真实 Chromium 实测：`visibilityState: hidden` 时给一个对照 div 写 `scrollTop` 也收不到 scroll 事件）。`ctxOutlineSpyPending` 保证只跑一次。
- 实测（夹具）：把第 4 / 1 / 0 条滚到容器顶 → 目录高亮并**滚到**对应选项（`activeIndex === firstVisibleIndex`、该行完整可见、`docScroll` 恒 0）；
  把第 6 条滚到顶时容器已被 `maxTop` 钳住、实际可见的是第 5 条 → 目录跟随 **5**（跟随"实际看见的"，不是"想要的"）。

**③ 分块浅灰底**（`style.css`）

- `#hide-helper-ctx-merged-popup .hcm-chunk`：`background: var(--hh-background-secondary)`（亮色 `#F1F3F5` / 暗色 `#2D2D2D`）
  + `margin: 0 0 6px` + `padding: 6px 8px` + `border-radius: 6px` ⇒ 一眼看出"上一段到哪、下一段从哪开始"（内容本身仍是首尾相接的发送字符）。
  实测（真实 Chromium `getComputedStyle`）：`rgb(241,243,245)` / `6px` / `6px 8px` ✓。

**④ 验证（本轮复测）**

- 离线：`t9` **141**（新增 L 组：L1–L4 三窗下换条目不跳位/第三窗不丢；L5–L11 联动高亮 + 目录自身滚动 `scrollTop=114` + 不碰文档 + rAF/setTimeout 接线 + 浅灰底静态断言）
  + `t8` 161 + `t7` 103 + `t6` 53 = **458 全通过**；`node --check index.js` 通过。
- 真实 Chromium（`scratch/round27-fixture.html` 新增 `step_entry_switch` / `step_outline_spy`）：上面两条逐项量测通过；截图 `scratch/round27-outline-merged.png`（含浅灰分块）、`scratch/round27-three-windows.png`。
- ⚠️ 夹具的两个坑（都写进注释了）：① 联动量测要把目标块滚到"容器顶下方"必须按**内容坐标**算（`rect.top − 容器 rect.top + 当前 scrollTop`），
  用"增量"会被上一步残留的 `scrollTop` 带偏；② 自动化页是 hidden 状态 → 需手动 `dispatchEvent(new Event('scroll'))` 才能触发联动（真实可见标签页由浏览器照常派发）。

### 2.38-R2 第二十七轮第二轮（用户新需求 A–D + 采纳的体验建议 ①②④⑤⑥）

**用户口径（原话）**

> ① 「提高『完整上下文』的层级，不再嵌入到 `hide-helper-chat-stats-popup` 中，而是**直接点击 `tub-stat-box` 的『总共』**就可以打开 `hide-helper-context-viewer` 弹窗。」
> ② 「在 `hide-helper-ctx-outline-popup` 上下文目录中，对于**连续的『聊天/其他』来源**可以像『世界书』一样，可以归到一个 `hco-item` 下方。」
> ③ 「将 `hide-helper-ctx-merged-popup` 右上角的**关闭按钮去除**。」
> ④ 「优化 `hide-helper-context-viewer` 中 `hc-toolbar` 的布局：『来源/角色』做成**两个菜单按钮**（点开菜单项过滤）；去掉 `hc-search` 搜索区域，**放大镜移到 `hc-wi-expand-all` 左侧**，点它在下方展开一行搜索框 + 条状按钮，再点收起；『清空检索』改成**橡皮擦**图标；**去掉 `hc-toolbar` 下方的分割线**。」
> ⑤ 体验建议采纳：①分块来源色条 ②键盘导航 ④大快照懒渲染 ⑤目录搜索框 + 来源分组 ⑥条目来源子窗加「跳到正文」（**③窗口宽度可调 + 记忆未采纳**，不做）。

**A. 「总共」框 → 直接开「完整上下文」查看器（层级提上来）**

- 概览行的「总共」框改成 `tub-stat-box tub-stat-box-clickable data-stat="total"`，`onclick` → `toggleContextViewerFromStats(model)`
  （点击开/关）；`title` 仍带预算预警文案。
- 层级：`openContextViewer()` 开头本就会 `closeChildStack()` 重置整轮层级 ⇒ 查看器**直接挂在主弹窗下**（关掉它回主弹窗），
  不再需要先点「聊天」→ 聊天统计子窗 → 「查看完整上下文」。实测：`main 558 | 查看器 990`，主窗可见、统计子窗不出现、`parentChildId() === null`。

**B. 目录分组：连续的「聊天 / 其他」归到一个 `hco-item` 下**

- 新增纯函数 `groupOutlineMessages(messages)`：世界书消息各自一项；**连续**的非世界书消息合成一组。
- 渲染：世界书项 = 消息行 + 条目明细（原样）；分组项 = 分组头（`.hco-group-head`，`#首–#末` + `聊天 N · 其他 M` + 条数 + tokens 合计）
  + `.hco-sublist > .hco-row`（逐条明细，`.hco-sub-head`）。
- 所有"消息行"统一 `.hco-row` + `data-i`：跳转 / 高亮（`.hco-row-active` + 所在项 `.hco-item-active`）/ 滚动联动都按它找。
  单条不分组（退化成一条独立消息行）。实测（案例十六）：8 条消息 → **4 项**（含两组 `#1–#4 聊天 3 · 其他 1`、`#6–#7 聊天 2`），行数 8 不变。

**C. 合并内容窗去掉右上角关闭按钮**

- DOM / 绑定 / CSS 三处都去掉；一对窗口仍由「目录的 × / `Esc` / 再点目录图标」收起（`closeContextMergedPopup()` 保留，供图标路径复用）。

**D. `hc-toolbar` 重排**

- 第一行：`来源` `角色` **两个菜单按钮**（`.hc-menu-btn`，文案随当前值更新、非"全部"时高亮）+ 显示条数 + **放大镜** + 目录图标
  （放大镜紧挨目录图标左侧）；第二行（**默认收起**）：检索框 + 命中数 + 上/下/橡皮擦，点放大镜展开（并聚焦）/再点收起（收起即清空检索词）。
- 下拉菜单新写 `openCtxFilterMenu()`：**挂 `document.body` + `position: fixed` + 触发器视口坐标**——
  ⚠️ 不能复用 `openTypeFilterMenu`：它把菜单挂进窗口内部，而查看器是 `overflow: hidden`（会被裁掉），
  且查看器可见时主弹窗可能正好是隐藏的。菜单带 `data-theme="dark"`（挂在 body 上拿不到窗口的暗色变量）。
- 「清空检索」图标改 `HC_ICON_ERASER`；`.hc-toolbar` 去掉 `border-bottom`（分割线）。

**采纳的体验建议**

- **① 分块来源色条 + 图例**：`.hcm-chunk` 带 `hcm-kind-<kind>`，左边缘 3px 色条（聊天绿 / 前置蓝 / 后置青 / 深度紫 / 混入·合并橙 / 其他灰，亮暗共用一套）；
  头部 sub 行追加 `.hcm-legend`（色点 + 来源名 + 条数）。跳转高亮改为整块描边（左边缘已被色条占用）。
- **② 键盘导航**：`handleContextHotkeys`（文档级绑一次）—— `↑/↓` 或 `J/K` 在消息间跳、`Home/End` 到首/末条；
  成对模式里跳内容区、否则滚查看器；**输入框 / 文本域里一律不拦**；只在"自己窗口是当前窗"时生效。
- **④ 大快照懒渲染**：`renderMergedContext()` 改成**分批**（每批 ≤8 块且 ≤40KB，`setTimeout(0)` 依次追加，只追加不改已有块）；
  跳转到还没渲染的消息时先 `ensureMergedChunksUpTo(i)` 同步补齐再跳。`ctxMergedRenderToken` 让换快照时旧批次自动作废。
- **⑤ 目录检索**：目录头部加放大镜 + 输入框（点开/收起），`contextOutlineMatch(m, term)` 命中 `#序号 / 角色 / 来源 / identifier / 正文 / 条目名+书名`；
  只过滤目录（内容区不动）。
- **⑥ 条目来源子窗「跳到正文」**：`createEntryRowBuilder({ jumpIcon:true })` 每行多一个箭头图标（不冒泡，仍可点行看条目详情 → `jumpToMergedFromWiMsg()`：
  需要时先开出「目录 + 合并内容」，再把内容区滚到该条目正文首行）。

**修掉的真 bug（本轮实测发现）**

- ⚠️ **从「条目来源」的"跳到正文"图标进成对模式时，其它子窗没被收起** → 屏幕上堆出 3~4 个重叠窗口。
  根因：`layoutChildPair()` 的成对模式分支只摆「目录 + 合并内容」，**没有把其它子窗收起来**（原来只有"从查看器进"这一条路，恰好没暴露）。
  修法：该分支先遍历 `CHILD_POPUP_IDS` 把除这两窗以外的窗口全部 `display:none`，再交给 `layoutOutlineMergePair()`。实测：图标点击后可见集合 = `目录 + 合并内容`，无重叠、页面 scrollY 恒 0。

**验证**

- 离线：`t9` **166**（H/L 组按新结构更新 + 新增 M 组 24 条：M1–M4「总共」框层级 / M5–M7 分组纯函数 / M8–M10 目录检索纯函数 /
  M11–M16 结构与工具栏断言 / M17–M20 键位 / M21–M22 懒渲染分批与补齐 / M23–M24 跳跃图标）
  + `t8` 161 + `t7` 103 + `t6` 53 = **483 全通过**；`node --check index.js` 通过。
- 真实 Chromium（硬刷新后量测）：工具栏 `searchRowHidden=true`、`toolbarBorder=0px`、放大镜在目录图标左侧（index 3 vs 4）、
  「来源 全部 / 角色 全部」两个菜单按钮、橡皮擦图标在、点放大镜 `收起 → 展开 → 收起`；菜单宿主 = `BODY`、四项 全部/聊天/世界书/其他、
  选「世界书」后按钮文案变「世界书」且列表只剩 2 条；「总共」框 → 查看器 `990`（主窗 `558` 仍可见、统计子窗不出现、`parentIsMain=true`、再点收起）；
  目录 4 项（含两组连续聊天/其他）+ 6 条明细行 + 8 行总数；分块色条 `wi-before=rgb(59,130,246)` / `other=rgb(148,163,184)` / `chat=rgb(34,197,94)` / `wi-depth=rgb(139,92,246)`；
  图例文案「世界书·前置 1 其他 1 聊天 5 世界书·深度 1」；目录检索输入 `worldInfoBefore` → 只剩 1 项；
  键位 `↓/J` 前进、`End` 到底、`Home` 回顶（`active=#0`）、输入框里按键不动；
  「跳到正文」图标 → 只剩「目录 + 合并内容」且内容区滚到该条目正文（`scrollTop=202`，目标块 `#0`）。
- 截图：`scratch/round28-outline-merged-colorbar.png`（目录分组 + 来源色条 + 图例）、`scratch/round28-toolbar-search-menu.png`（工具栏：检索行展开 + 来源菜单）。
- 新增夹具步骤：`step_toolbar` / `step_total_box_opens_viewer` / `step_outline_grouping`；`step_merge_close` 改点目录的 ×（合并窗已无关闭按钮）。

### 2.38-R3 目录列表整体重设计（用户口径："还太杂、太别扭"）

**用户口径（原话）**：「把这个列表 `hide-entry-body hco-list` 重新设计一遍，现在看起来还太杂、太别扭了。」

**先诊断"为什么杂"**（对着 R2 的产出逐项看）：
1. **三种行长得几乎一样**：项头（消息）、分组头（连续 N 条）、明细行都用同一套 `.hc-msg-head` 外观 ⇒ 看不出谁包着谁；
2. **条目行沿用了主面板的"卡片"**（`.tub-new-list-item`：类型渐变底 + 阴影 + hover 上浮）——那是"条目面板"的语言，
   放进目录里就是噪声，而且一行一个彩色进度条、整页花掉；
3. **缩进只有 `margin-left: 14px`、没有树线** ⇒ 从属关系要"读"出来，不是"看"出来；
4. **列不对齐**：序号 / 角色 / 来源 / tokens 各行的右边缘都不一样（tokens 紧跟在来源后面），竖着扫读不出任何东西；
5. **间距层级不分**：项间 8px、项内 6px、明细 4px，差距太小 ⇒ "项"和"行"分不开；
6. 分组头文案冗长（"连续 4 条（下面逐条列出）"）。

**重设计（三层 + 一套列 + 平面行）**：

- **三层结构**：① 项头 `.hco-head`（一条消息，或"连续 N 条"分组头）= 铺底色的一整行；
  ② 子项区 `.hco-kids` = 左侧一条**树线**（`border-left: 1px`）+ 缩进 14/10px；
  ③ 子行 `.hco-kid`（分组里的消息明细 / 世界书条目明细）= 统一缩进 + 更小字号。
  间距改成 **项间 10px / 子行间 1px**，层级一眼可分。
- **一套四列**：`.hco-idx`（序号徽标，等宽数字）/ `.hco-role`（角色 chip）/ `.hco-src`（来源，`flex: 1` + 省略号）/ `.hco-tokens`（右对齐、`tabular-nums`）；
  项头与子行**同一套**（实测 tokens 右边缘 935/936、条目 tokens 936 —— 全在同一列）。
- **分组头**：左侧一条 3px 短竖条 + 药丸标签（`聊天 3 · 其他 1`）+ "连续 N 条"，与"单条消息"完全区分开。
- **条目行平面化**：抹掉渐变底 / 阴影 / hover 上浮（`background: transparent !important`、`transform/box-shadow: none !important`），
  类型改用行首一个 7px 小方块（`::before` + `[data-type=…]` 着色：常驻蓝 / 动态绿 / EJS 红）—— 信息还在，噪声没了。
- **字号层级**：项头 12px、子行/条目 11px。
  ⚠️ **必须写 px、不能写 em**：相对字号在"最小字号 12px"的浏览器里会被抬到 12px（本机真实 Chromium 实测：0.5em / 0.68em / 0.72em 全部渲染成 12px，
  而 10px / 11.5px 照原样）⇒ 项头 0.72em 与子行 0.68em 会变成一样大、层级直接消失。这条踩过一次（第一版就是 em）。
- 保留 R2 的交互契约不变：`.hco-item` / `.hco-row` / `.hco-group-head` / `.hco-row-active` / `.hco-item-active` /
  `.hco-entries .tub-entry-clickable`（点击跳转、高亮、滚动联动、目录检索都按这些类找），故 JS 只换"行的外观标签"。

**实测（真实 Chromium，目录宽 350px）**：项头 `350×26`（分组头 `350×29`）、明细行 `326×19`、
左边缘 项头全 593 / 明细全 617、tokens 右边缘 935(头)/936(明细) 、条目 tokens 936、字号 12/11/11、树线 1px；
条目行的 `transform`/`box-shadow` 全为 `none`（卡片味已消失）；
四类交互回归：点明细行 → 滚到 #3 并高亮（含所在项）、点条目行 → 内容区滚到该条目正文（`#0`，无整页滚动）、点分组头 → 跳到该组第一条、目录检索 `#6` → 只剩 1 项、清空 → 4 项。

**验证**：`t9` **167**（H4/H5 按新类名、新增 H6b 版式断言：`.hco-head` 项头 + `.hco-kid hco-sub` 明细行 + `.hco-kids hco-entries` 条目区）
 + `t8` 161 + `t7` 103 + `t6` 53 = **484 全通过**；截图 `scratch/round28-outline-redesign.png`。

### 2.38-R4 第二十七轮第四轮（目录/内容区七条口径；本轮新增；已实施并离线 + 真实 Chromium 实测）

**用户口径（原话）**：
> 第一，每个条目行的间隔分开点。第二，"其他"来源的条目不要和"聊天"来源条目归入到一块。
> 第三，对于 `hco-src` 不要带单位 `t`。第四，把 `hco-search` 这个去除，而把 `hcm-legend` 的显示改为
> `聊天/世界书/其他` 三个 `hcm-legend-dot hcm-kind-wi-after`。同时显示其条目数和 tokens 数。
> 第五，不再显示 `hco-idx`，而聊天条目则显示 messageID。第六，我希望右侧滚动到对应位置时，
> 左侧条目可以高亮显示或者是其他显眼的提示。第七，把 `hide-entry-name` 的标题改为"完整上下文"。

**逐条落地**：
1. **行距拉开**：`.hco-list` gap 10 → **14px**、`.hco-kids` gap 1 → **5px**（消息明细行与**世界书条目行**共用
   这个容器，两处一起拉开）、`.hco-kid` padding 3px → **5px 8px**、条目行 `.tub-entry-clickable` padding 3px → **5px 8px**、
   项头 padding 6px → 7px 9px。实测 `listGap=14px / kidsGap=5px / kidPadding=5px 8px / entryPadding=5px 8px`。
2. **分组按来源拆开**：`groupOutlineMessages` 的 run 以 `ctxSourceGroupOf(m.kind)`（`chat` / `other`）为界 ——
   **来源一变就另起一项**；分组头药丸只显示本组标签（`聊天 3` / `其他 1`）。夹具实测：7 条消息 → 5 项
   （世界书 1 + `其他 1` + `聊天 3` + 世界书（深度）1 + `聊天 2`），`main` 不再并进聊天那一组。
3. **tokens 不带单位**：目录行 / 分组头由 `${m.tokens} t` 改成 `${Number(m.tokens) || 0}`；单位由头部
   "共 N tokens" 交代（判定见下"语义判定"）。
4. **删检索框、图例固定 3 组**：`ensureContextOutlinePopup` 里的 `.hco-search` 标记与放大镜/输入框/防抖绑定、
   `ctxOutlineTerm` 变量**整体删除**（目录恒显示全部消息；检索纯函数 `contextOutlineMatch` 保留备用）。
   `hcmLegendHtml` 改为固定三项 `聊天 / 世界书 / 其他`，每项 `色点 + 名称 + N 条 + tokens`，色点用
   `hcm-kind-chat / hcm-kind-wi-after / hcm-kind-other`，`title` 里写全口径（世界书额外给"N 个条目"）。
   同时 `hcmChunkHtml` 的 class 由 7 种细分 kind 收成**同一套 3 组**（与图例一一对应）——
   否则图例说"世界书"是蓝的、分块却是橙的（原 7 色规则保留在 CSS 里备用）。
   ⚠️ **顺带修掉一个真 bug**：`.hcm-legend-dot` 此前只有 `background: var(--hcm-kind-other)` 一条规则
   （`.hcm-kind-*` 那批只改 `border-left-color`）⇒ **三个色点原本全是灰的**；本轮补 `.hcm-legend-dot.hcm-kind-*` 上色。
5. **不显示快照序号，聊天条目显示 messageID**：新增纯函数 `outlineChatMesids(messages)` ——
   ST 把每条待发送的聊天消息**按顺序**塞成 `chatHistory-N` 槽位，故快照里第 k 条 `kind==='chat'` 对应
   `computeSendableChat()` 的第 k 条（`floor` = 聊天数组下标 = messageID）；错位风险用**文本包含对齐**
   （复用 `slotContainsChatText`，与"这槽是不是聊天楼层"同一套判定）兜住，对不上再退回按序，拿不到聊天就
   **不显示**（宁缺毋滥）。渲染用 `.hco-mesid`（原 `.hco-idx` 全删）；聊天分组头额外给区间 `#2–#4`。
   实测：夹具两条聊天组分别显示 `#2 #3 #4` / `#6 #7`，分组头 `#2–#4` / `#6–#7`；`hasIdx=false`。
6. **高亮强化**：`.hco-row-active` 由"只有 3px 左描边"改成 **accent 底色 + 4px 左描边 + 加粗**，
   所在项 `.hco-item-active` 用 `--hh-accent-color-bg-light`。实测（滚到第 4 块）：行 `background rgba(148,0,211,.08)`
   + `inset 4px` 描边 + `font-weight 700`，`firstVisibleIndex=4` = 目标块、`activeRowIndex=4`（所在项 `data-i=2`，即组首）。
7. **标题**：合并内容窗 `#hcm-title` 由"完整上下文（合并内容）" → **"完整上下文"**（`#hco-title` 仍是"上下文目录"）。

**语义判定（两处，需用户确认）**：
- 第 3 条说的 `hco-src` 本身**不带单位**（它是"世界书·前置 · worldInfoBefore"或"连续 N 条"）——
  按"目录里唯一带 `t` 的东西"判定：用户指的是 **tokens 显示**（原为 `900 t`）⇒ 去掉 ` t`。
- 第 7 条的 `hide-entry-name` 有三个（条目详情 / 查看器 / 合并内容窗），本轮上下文是"目录 + 完整上下文"
  一对窗口，故改的是**合并内容窗**标题；查看器 `#hide-context-title` 本来就是"完整上下文"。

**验证**：`t9` **178**（H5 改判 messageID/无 idx/无检索/tokens 无单位、H10b/H10c/H10d 图例 3 组 + 条数 tokens +
分块 3 组、M5 拆组 + 新增 M5b 回归、"其他→聊天"不再并组、M25–M32 标题/检索移除/messageID 映射/CSS 行距）
 + `t8` 161 + `t7` 103 + `t6` 53 = **495 全通过**；`node --check index.js` OK、CSS 大括号 554/554。
真实 Chromium 夹具（1536×674）：目录 `558` | 内容 `990`（不重叠、`docScroll=0`）、三窗例外回归 `126 | 558 | 990`；
图例 `聊天 5 条 · 70 / 世界书 3 条 · 1120 / 其他 1 条 · 40`，三个色点 `rgb(34,197,94) / rgb(14,165,233) / rgb(148,163,184)`；
截图 `scratch/round29-r4-grouping-legend.png`、`scratch/round29-r4-outline-merged-highlight.png`。

**过程中修掉的两个真缺陷（都会让功能整体失效，离线 shim 抓不到）**：
1. ⚠️ **HTML 注释里写了反引号 → 模板字符串被提前截断**：`$('body').append(\`…\`)` 里的注释写了 `` `.hco-search` ``，
   反引号把模板字符串提前闭合 ⇒ `ensureContextOutlinePopup` 运行到那行抛 `ReferenceError: search is not defined`、
   **目录窗根本创建不出来**。`node --check` 通过（截断后仍语法合法：后半段被解析成第二个模板字符串），
   离线 shim 也抓不到（`$('body').append()` 是空桩）——**只有真实 Chromium 夹具能暴露**（夹具 step 直接抛错）。
   ⇒ 弹窗模板里的注释一律不写反引号。
2. ⚠️ **夹具量测本身的两个假象**（自动化里页面是 hidden）：① 入场动画 `hhFadeIn`（起于 `scale .95`）**不会跑完**，
   `rect` 全被缩放 5% ⇒ "滚到某块"与 scroll-spy 的 rect 比较系统性偏移（实测目标 4 却判成 0）；
   ② CSS `transition` 的时钟同样冻结 ⇒ 刚加 `.hco-row-active` 时 `getComputedStyle` 读到**起点值**（transparent）、
   截图也看不出高亮。夹具新增 `unfreezeGeometry()`：`transform: none !important`（必须先 `!important`，
   CSS 动画会压过普通内联声明）+ 注入 `* { transition: none !important; animation: none !important }`。

### 2.38-R5 第二十七轮第五轮（三条回头口径：t 单位恢复 / 去掉激活粉底 / 条目「插入位置 · 顺序」；已实施并离线 + 真实 Chromium 实测）

**用户口径（原话）**：「算了，目录行/分组头 `${m.tokens} t` 还是保留 t 单位吧。然后，`hco-item hco-item-active`
激活时的粉红色背景也去除，太难看了。最后，就是我想让各个世界书来源的条目可以显示具体的"插入位置和顺序"，
但是不知道以什么方式显示、显示在哪。目前的 `tub-book-tag` 位置是不够用的。你有什么想法吗？」

**① tokens 单位恢复**（R4 去掉、R5 按用户口径改回）：目录行 `${Number(m.tokens) || 0} t`、分组头 `${tokensSum} t`，
**图例也一并带上**（`聊天 5 条 · 70 t` 这类；R4 是我主动去的单位，一并恢复，免得两处口径不一致）。

**② 激活项不再铺粉底**：`.hco-item-active` 的 `background` 规则**删除**（类保留 —— 它是 `syncOutlineActiveHead` /
夹具 / 离线单测的钩子）；视觉提示全部落在"当前那一行"：`.hco-row-active` = **中性灰底**
（`var(--hh-btn-secondary-hover-bg)`）+ 4px accent 左描边 + 加粗 + 正文色。
实测：行 `background rgb(233,236,239)` + `inset 4px accent` + `font-weight 700`，所在项 `rgba(0,0,0,0)`（粉底已无）。

**③ 条目显示「插入位置 · 顺序」**（用户提问："以什么方式显示、显示在哪"）——

**为什么放"第二行"**：`tub-book-tag` 是跟在条目名后面的行内片段（条目名 + `(书名)` 已经占掉大半行），
再往那儿加"插入位置 + 顺序"两段文字，要么把条目名挤成省略号、要么两段都看不清。
故**不动名字行**，改成条目**自己的第二行**（`.tub-nli-meta`）：名字行完全不动、两行左边缘对齐，
扫读时"位置 / 顺序"竖着成列。这条也顺带回答了"显示在哪"——**条目名下方、缩进对齐名字起点**。

- **文案与「条目详情」窗完全同源**（不新造词汇）：插入位置 = `entryInsertPositionText({position, depth, role})`
  （角色定义前/后、作者注释前/后、示例消息前/后、`系统/AI/用户 @D n`）；顺序 = ST 条目 `order` 的数值。
- **数据层**（快照里原本没有这两个字段）：
  · `entriesMeta` 增 `positionRaw` + `order` —— ⚠️ **绝不能直接改 `position`**：归因纯函数（`=== 0 / === 1` 判定、
    D 系列护栏）正在用它，把 `undefined` 变成 `null` 会改变那些判定（会动 t7/t8 的口径），故另存原始值；
  · `wiDetailedStats` 的 record 与快照 `wiEntries[]`（= 「条目来源」子窗与目录的数据源）都带上
    `position / depth / role / order`（`slotEntriesOf` 从 `entriesMeta` 取，同源同口径）；
  · 旧快照 / 演示快照没有这些字段 → **整行不渲染**（宁可不显示，也不显示一个错的插入位置）。
- **渲染**：`createEntryRowBuilder` 新增选项 `insertMeta`（默认 **false**）：只有「上下文目录」与「条目来源」子窗
  打开它；主面板条目列表保持紧凑（那里本来就有「上下文顺序」视图回答次序问题）。
- **CSS**：`.tub-nli-meta { flex: 1 0 100% }`（占满一行）+ 两处条目行加 `flex-wrap: wrap`
  （⚠️ 不加就换不了行，第二行会跟名字挤在同一行）；目录里条目行行首有 7px 类型方块，故第二行缩进 15px 对齐名字。
- **实测**（1536×674 真实 Chromium）：目录条目行第二行 3 条 ——
  `插入位置 角色定义前 · 顺序 100` / `插入位置 角色定义后 · 顺序 80` / `插入位置 系统 @D 4 · 顺序 30`，
  条目行高 50px（两行）；「条目来源」子窗同样 2 条第二行（行高 50px）；三窗例外回归 `126 | 558 | 990`、
  不重叠、`docScroll=0`。

**备选方案**（没做，供用户挑；说一声就能换）：
1. **行内右侧双 chip**（`[角色定义前] [#100]`）：一行看完，但条目名会被挤到 ~100px，长名字全是省略号；
2. **只挂悬停 / 留在「条目详情」窗**：零布局成本，但扫读不到（用户要的正是"一眼看出"）；
3. **按插入位置分组 + 组内序号**：同一处插入的条目归一个小标题，信息结构最清楚，但行数会明显变多。

**验证**：`t9` **186**（H5/H10c 改成 tokens 带单位、新增 N1–N8：t 单位 / 图例带单位 / 目录与子窗的第二行文案 /
`insertMeta` 默认关闭 / `@D` 角色+深度文案 / 字段缺失不渲染 / CSS 断言） + `t8` 161 + `t7` 103 + `t6` 53 = **503 全通过**；
`node --check index.js` OK、CSS 大括号 557/557；截图 `scratch/round30-r5-entry-meta.png`、
`scratch/round30-r5-highlight-neutral.png`。

### 2.38-R6 第二十七轮第六轮（用户回头两条：目录行的 `t` 单位 + 条目行第二行重排；已实施并离线 + 真实 Chromium 实测）

**用户口径（原话）**：
> 存在两个问题。第一，对于"聊天来源"的 `hco-kid hco-sub hco-row` 不用带单位 `t`。
> 第二，对"世界书"来源的条目显示的内容 `hco-kid hco-sub hco-row`，你不觉得布局很乱吗，
> 并且还没有对齐美？尤其是这个"插入位置"标题文本就不用显示，直接显示具体的插入位置即可。
> 同时我希望也要显示条目的 order。

**① 目录行 / 分组头的 tokens 去掉单位 `t`**（R5 加上的那条，用户回头要求去掉）：
`msgRowHtml` 的 `.hco-tokens` 由 `${Number(m.tokens) || 0} t` → `${Number(m.tokens) || 0}`、
分组头由 `${tokensSum} t` → `${tokensSum}`。**保留单位的只有合并内容窗的图例**
（`聊天 5 条 · 70 t`——用户点的是"目录里那些行"，图例在另一个窗口、上一轮口径要的是带单位）。
⇒ 如果要连图例一起去掉，说一声即可（一行改动）。

**② 条目行第二行重排**（用户：乱、没对齐、别显示"插入位置"这个词、顺序也要看得见）：
先定位了三个**真实原因**，都不是"再调两像素"能解决的：

- **顺序为什么会看不见**：原实现把「插入位置 … · 顺序 …」两段拼成一段文本，外层是
  `white-space: nowrap; overflow: hidden; text-overflow: ellipsis` —— 位置文案一长（"深度注入（@D 4）"之类），
  **省略号先吃掉的就是尾巴上的顺序**。⇒ 改成**两列 flex**：位置在左（`flex: 1 1 auto` + `min-width: 0`，长了先截它）、
  顺序在右（`flex: 0 0 auto` 的小 chip，**永远完整**），右边缘与上一行的 tokens 列对齐。
- **为什么"乱"**：那一行有 15px 左缩进，而 `flex-basis: 100%` 是按**内容盒**算的 —— 页面若没有 ST 全局的
  `* { box-sizing: border-box }`（本仓夹具就是），整行会宽出 15px、右边缘越过 tokens 列
  （真实 Chromium 实测 **949 vs 934**）。⇒ 该行**显式** `box-sizing: border-box`；
  同时把条目行的 `gap: 8px` 收成 `gap: 2px 8px`（第二行的行距不再等于列间距）、去掉 `margin-top: 2px`。
- **对齐**：两行的左边缘都落在**条目名**左侧（行首 7px 类型方块 + 8px gap = 15px 缩进），
  右边缘都落在 tokens 列右侧 ⇒ 竖着扫是两列整齐的值。

**文案**：`<b>插入位置 X</b> · <b>顺序 N</b>` → 只留值：`<span class="tub-nli-pos" title="插入位置：…">X</span>`
+ `<span class="tub-nli-order" title="顺序（ST 世界书条目的 order，数字越大越靠后）：…">N</span>`
—— 口径没丢，挪进 `title`（悬停可见）。顺序用 chip（`--hh-input-bg` 底 + 等宽数字）与位置区分开，
一个光秃秃的数字不至于被误解成 tokens。
覆盖「上下文目录」的条目区 + 「条目来源」子窗（共用 `createEntryRowBuilder({ insertMeta: true })`）；主面板不受影响。

**⚠️ 本轮自己踩的一个坑（已加防回归）**：修 CSS 时分了**两次** `Edit` 调用，第二次仍按**上一次读到的行号**
写范围 —— 而第一次编辑已经让文件变长了 10 行，于是范围整体偏移，把 `.tub-nli-pos` 的目录选择器与
`padding-left: 15px` 那条规则**覆盖掉了**，还留下一个"选择器行 + 选择器行 + `{`"的畸形块。
**大括号数照样 558/558 配平**（解析器把多余的选择器行当无效声明跳过），肉眼与配平检查都看不出 ——
只有真实 Chromium 量测暴露（图例三个色点变透明、位置列不再与名字对齐）。
⇒ ① 改完一定要**重新量** ② 新增 `scratch/t9` 的 N9：剥注释 + 放行 at-rule 后扫描"普通块内嵌套 `{`"（配平检查抓不到这一类）；
N10 断言行内第二行显式带 `box-sizing: border-box`。

**验证**：`t9` **188**（H5/N1 改为"目录行与分组头不带单位"、N3/N5 改为"只显示值 + 顺序在位 + 口径进 title"、
N7 CSS 两列布局、新增 N9/N10 防回归） + `t8` 161 + `t7` 103 + `t6` 53 = **505 全通过**；
`node --check index.js` OK；CSS 结构体检 558/558、无非法嵌套。
真实 Chromium 夹具（1536×674）逐条量测三行条目：两行制 `rowH=42`、`line2BelowLine1=true`、
`posLeftAlignedWithName=true`、`orderRightAlignedWithTokens=true`、`orderFullyVisible=true`，
`metaText` 为 `角色定义前100` / `角色定义后80` / `系统 @D 430`（值 + 值，无标题词）；
目录 tokens 文本 `900 / 40 / 39 / 12 / 18 / 9 / 220 / 31 / 24 / 7`（无 `t`）；
图例色点恢复为 `rgb(34,197,94) / rgb(14,165,233) / rgb(148,163,184)`；截图 `scratch/round31-r6-entry-meta.png`。

### 2.39 第二十八轮（「上下文目录」六条口径：折叠 / tokens 单位 / 条目行对齐 / 聊天子行改版 / 项头去角色 / 条目行滚动联动；本轮新增；已实施并离线 + 真实酒馆 + 真实 Chromium 夹具实测）

**用户口径（原话）**：
> 第一，点击 `hco-head hco-row` 可以折叠/展开下方列表项，同时它们的右侧 tokens 单位带 `t`，列表的 tokens 单位不带 `t`。
> 第二，对于 `tub-nli-label` 不进行换行，同时 `tub-nli-value` 固定在最右侧右对齐。
> 第三，对于聊天 `hco-kid hco-sub hco-row`，去除 `hco-src`，并且 `hco-role hco-role-ai` 的背景颜色改为浅灰色，`role=assistant` 改为 `ai`。同时，显示该 messageID 的前 N 个字符（不换行）。
> 第四，把所有 `hco-head hco-row` 的 `hco-role hco-role` 给去除。
> 第五，当右侧滚动到不同位置时，左侧上下文目录的各个项要激活显示。目前只有"聊天"列表的项才会这样显示，世界书列表的项不会这样显示。
> 第六，对世界书的 `hco-src` 不管是"世界书·前置"还是"世界书·后置"一律显示为"世界书"。

**两条先行确认（`ask_user_question`，用户已选）**：
- 第三条的"该 messageID 的前 N 个字符"⇒ **消息正文的前 N 个字符（文本预览）**（用户选）；
- 第五条的"世界书列表的项"⇒ **世界书条目行也要激活**（用户选；我先实测确认世界书**消息行**本来就会高亮 ——
  激活底色 `rgb(233,236,239)` + 4px 左描边 + 加粗，普通态 `rgb(241,243,245)`，真正从不高亮的是条目行）。

**逐条落地**：

1. **项头/分组头可折叠 + tokens 单位**：`renderContextOutlineList` 的行构建器拆成三个 ——
   `msgHeadHtml`（项头：折叠箭头 + messageID + 来源 + 跳转图标 + `tokens t`）、`msgKidHtml`（聊天子行：
   messageID + 角色 chip + 正文预览 + `tokens`）、分组头（同项头版式 + 药丸 + messageID 区间 + `tokensSum t`）。
   项头/分组头统一加 `.hco-collapsible`，点击切 `.hco-collapsed`（CSS 藏掉 `.hco-kids`），
   折叠状态存模块级 `ctxOutlineCollapsed`（键 = 行首 `data-i`；打开目录时清零，不跨快照沿用）。
   ⚠️ **跳转不能丢**：行尾加 `.hco-jump` 小图标（与「条目来源」子窗的 jump 图标同款）负责"跳到该消息第一行"，
   `stopPropagation` 免得连带折叠 —— 否则折叠后（尤其 `main` / 世界书这种只有一行的项）就没法跳了。
2. **条目行 `tub-nli-label` / `tub-nli-value` 对齐**：`.hco-entries .tub-nli-label` 的 `flex: 1 1 auto` → **`flex: 1 1 0`**。
   根因（真实 Chromium 复现）：`auto` 的基准尺寸 = 条目名内容宽度，长名字一超过整行，外层 `flex-wrap: wrap`
   就把 tokens **挤到第二行行首**（实测 value 从行尾 384 跑到行首 67、行高 26→53）；归 0 后标签只在剩余
   空间里生长，tokens 永远钉在最右，长名字走省略号（`nowrap` 本就就位）。实测长名下行高仍 26、tokens 仍在 384。
3. **聊天子行改版**：去掉来源列；`roleChip` 文案与类名统一（`assistant` → `ai`）；`.hco-role-ai` 底色由
   绿色改 `var(--hh-btn-secondary-hover-bg)`（浅灰）；中间新增 `.hco-preview`（正文前 `OUTLINE_PREVIEW_CHARS`=80
   字符、空白压成单空格、不换行 + 省略号，全文进 `title`）。`.hco-preview` 同样 `flex: 1 1 0`（tokens 不被挤走）。
4. **项头/分组头去掉角色 chip**：`msgHeadHtml` / 分组头都不再渲染 `roleChip`（角色只在聊天子行上出现）。
5. **世界书条目行的滚动联动**（本轮唯一的新交互）：
   - 新增 `ensureEntryLayout(msgIdx)`：用快照 `wiEntries[].pos`（= `槽位序号*1e6 + 该条目第一个行锚点在槽位正文里的
     字符偏移`，`pos % 1e6` 即条目正文起点偏移）量出各条目正文的纵坐标（`Range`，内容坐标系），
     按 `ctxMergedRenderToken`（内容区渲染代次）缓存；槽位号与当前消息对不上的条目不参与（宁可不亮也不亮错）。
   - 新增 `syncOutlineEntryActive(i, chunkEl, force)`：容器顶的内容坐标落在哪个条目的 `[top, 下一个 top)` 里就亮哪行；
     落在第一个条目之前但已进入该消息的框内 → 亮第一条（否则刚滚到消息开头那一小段谁都不亮，像"没反应"）；
     只有"当前条目"变了才动 DOM（每帧加删类会抖），亮的那行同样 `scrollIntoViewWithin` 滚到看得见。
   - ⚠️ **`syncOutlineFromMergedScroll` 重构**：旧实现在"消息没变"时就 `return`，于是同一条世界书消息里继续往下滚
     当前条目水远不会换 —— 改成"消息没变只跳过消息行那半段，条目行每次都要更新"。
   - `jumpContextOutlineToMessage` / `jumpContextOutlineToEntry` 跳完也调一次 `syncOutlineEntryActive`（落点可能正某条目正文里）；
     `renderContextOutlineList` 末尾 `refreshOutlineEntryHighlight()`（重绘后行是新的，类没了）。
   - 样式 `.hco-entry-active`：与消息行同一套语言但左描边细一半（2px）—— 一眼看出"外层是消息、内层是条目"；
     底色必须 `!important`（条目行的进度渐变是 JS 写的**内联** background）。
6. **世界书来源列一律"世界书"**：`srcOf` 对 `isWiKind` 恒返回 `世界书`，全称（`世界书·前置 · worldInfoBefore`）
   挪进该列的 `title`（`srcTitleAttr`）。

**顺带修正**：`makeDemoContextDump` 的两条世界书条目 `pos` 由 `0 / 1` 改成真实口径 `1e6+0 / 1e6+15`
（槽位序号*1e6+偏移）—— 否则演示快照里条目联动不可见；浏览器夹具 `round27-fixture.html` 同步改
（并新增量测步 `step_outline_entry_spy(i, line)`：把第 i 条消息的第 line 行滚到容器顶，报亮哪条条目）。

**验证**：`t9` **194**（H4/H5/H6b/N1 改判新版式与单位、新增 O1–O5：折叠结构 / 聊天子行改版 / 世界书来源列 /
CSS 两条 / 条目行联动的**行为**单测——含"同一消息内继续滚会换亮哪条"这条旧实现永远不成立断言）
+ `t8` 161 + `t7` 103 + `t6` 53 = **511 全通过**；`node --check index.js` OK、CSS 大括号 567/567、无非法嵌套。
真实酒馆（`hideHelperDev.demo()` + 目录图标）逐条实测：
折叠/展开（项头与分组头，子项 2↔0、3↔0）、折叠后点跳转图标仍跳得动（`scrollTop=46`、`targetChunk=1`）且不连带折叠、
条目行联动（第 1 行→`前置常驻条目`、第 2 行→`前置动态条目`、滚到聊天组→撤掉）、长条目名行高仍 26 且 tokens 仍在最右、
世界书两行都显示"世界书"、项头/分组头 tokens 带 `t`（`900 t`/`216 t`）子行不带（`22`/`180`）、项头无角色 chip、
`hco-role-ai` 文案 `ai` + 浅灰底；`docScroll` 恒 0、窗口数恒 2。
真实 Chromium 夹具（1536×674）：`step_outline_grouping` 的 `tokensTexts` = `900 t / 40 t / 39 t / 12 / 18 / 9 / 220 t / 31 t / 24 / 7`
（头带 `t`、子行不带）、`groupHeads` 带 `t`、`hasIdx=false`；`step_outline_entry_spy` 三行逐一对应条目；
`step_merge_open` 两窗不重叠、`docScroll=0`。截图 `scratch/round32-outline-six-items.png`。

**过程中修掉的一个真缺陷（离线 shim 抓不到）**：夹具 `ctx-fixture.lib.mjs` 是 index.js 的**生成快照**
（`node scratch/ctx-fixture.gen.mjs`）—— 本轮改完 index.js 后没重新生成，夹具跑的还是旧代码，
条目联动在夹具里"不生效"（真实酒馆里是生效的）⇒ 已重新生成（401180 B）。


---

### 2.40 第二十九轮：删掉第一层「完整上下文」查看器 + 工具栏迁到目录 + 两处右对齐 + 世界书条目定位三修 + 目标块四边描边

**用户口径（原话）**：
1. 「直接去除点击『总共』出现的 `hide-helper-context-viewer` 弹窗，直接显示`上下文目录`和右侧`完整上下文`弹窗。」
   （追问后明确：「我指的是把第一层的"完整上下文"弹窗给去除了。而下一级"上下文目录"和"完整上下文"的保存，
   因为我觉得第一层的"完整上下文"弹窗内容太过重复了。」）
2. 「把上下文目录 `hide-helper-ctx-outline-popup` 的 `hide-entry-sub` 去除，同时将 `hide-helper-context-viewer`
   的过滤和搜索功能移植到 `hide-helper-ctx-outline-popup` 的 `hide-entry-sub` 的原有位置替代。」
3. 「将完整上下文的 `hide-entry-sub` 移至标题 `hide-entry-name` 的最右侧（右对齐），同时 `hcm-legend` 也改为右对齐。」
4. 「对于上下文目录中，"世界书"来源的条目的滚动定位还是不够准确。我在右侧内容滚动时，总是一下子就跳了好几个
   世界书条目高亮显示，感觉很奇怪。」
5. 「当 `hcm-chunk hcm-kind-wi-after` 等内容处于 `hcm-chunk hcm-kind-wi-after hcm-chunk-target` 目标状态时，
   能不能实现这样的样式效果：左边依然显示对应的蓝色/绿色/灰色竖线，而另外三边则显示当前紫色的边框颜色，
   而不是现在左边会有两种不同颜色的竖线叠在一起非常难看。」

**① 查看器窗口整体删除（`#hide-helper-context-viewer`）**
- `ensureContextViewer` / `openContextViewer` / `closeContextViewer` / `toggleContextViewerFromStats` /
  `renderContextList` / `ctxMessageRowHtml` / `bindContextRowEvents` / `refreshMsgTexts` / `applyHitCursor` /
  `jumpCtxHit` / `openMsgSearch` / `closeMsgSearch` / `scrollContextViewerToMessage` / `topVisibleMessageIndex(mergeOn)`
  / `ctxMsgTextHtml` / `ctxPreviewText` / `CTX_PREVIEW_CHARS` / `ctxExpanded` / `ctxMsgSearch` / `ctxMsgHit*` 全部删除；
  `CHILD_POPUP_IDS` 6→5、`childPopupTable()` 去掉该行、`applyTheme` 选择器同步去掉。
- 状态部分抽成 `loadContextSnapshot(model)`（找 dump、复位检索/筛选/折叠、更新标题 title），
  开窗仍由 `openContextOutlinePopup(model)` 负责。
- **入口重定向**：「总共」→ `toggleContextOutlineFromStats(model)`（再点收起）；
  「聊天」子窗的「查看完整上下文」按钮 → `closeChatStatsPopup(); openContextOutlinePopup(model)`；
  控制台 `hideHelperDev.{snapshot,openViewer,demo}` → 一律打开这一对窗口。
- **「条目来源」的新入口**：查看器删除后原入口随之消失 → 目录里**世界书消息的项头行尾**新增
  `.hco-wi-src` 图标（`HC_ICON_WI`），点它 `openWiMsgPopup(i)`。
- **挂起机制（`ctxMergeSuspended`）**：打开「条目来源」时右侧完整上下文暂时收起 → 屏幕上只留
  「目录 | 条目来源」；关掉子窗（× / Esc）→ `closeChildPopupAt` 末尾自动 `resumeOutlineMergeAfterChild()` 放回内容窗；
  从条目来源点「跳到正文」→ 先 `closeWiMsgPopup()` 再跳（否则三窗会被成对分支一起收起，实测踩过）。
- ⚠️ `layoutChildPair` 的成对分支里，**"挂起中"必须排在"目录不是当前窗"之前**：挂起时当前窗本来就是条目来源，
  用旧判定会顺手把 `ctxMergeMode` 清掉，关掉子窗后内容窗再也回不来（真实页实测复现并修复）。

**② 工具平移到目录窗（口径②）**
- 目录窗头部改成「标题 + `.hc-toolbar`」：两个菜单按钮（来源 / 角色）+ `#hco-count` + 放大镜 + 检索行
  （`#hco-search-input` / 计数 / 上/下 / 橡皮擦）；`bindContextOutlineToolbar(el)` 一处绑定。
- **筛选与检索只作用于左侧目录列表**（右侧恒显示全部内容——它是"实际发送的字符"，不该被裁剪）：
  `renderContextOutlineList` 里先 `ctxMessageVisible()` 过滤得到 `shown`。
- 检索高亮沿用查看器那套纯函数（`highlightTextHtml` + `data-hit` 连续编号）；整段重绘后
  `ctxHitCount = querySelectorAll('mark[data-hit]').length`，`applyOutlineHitCursor()` 负责当前命中 + 只滚目录自己的容器。
- 键盘导航（↑/↓、J/K、Home/End）重新绑到目录窗（`ensureContextOutlinePopup` 里绑一次），
  作用于右侧内容区（`jumpContextOutlineToMessage`），Home/End = 首/末条。

**③ 两处右对齐（口径③）**
- `.hide-entry-head`（完整上下文窗）`flex-direction: row`；`.hide-entry-sub` 加 `justify-content: flex-end` +
  `flex: 1 1 auto` + `flex-wrap` ⇒ 贴标题行**最右**，窄窗自动折行仍右对齐；
- `.hcm-legend` 由 `margin-left: 8px` 改 `justify-content: flex-end` ⇒ 图例也右对齐。

**④ 世界书条目滚动定位三修（口径④）**
- **逐槽位 pos**：新增 `entrySlotFirstPos: entryKey -> Map<slotIdx, 首个命中 start>`；`wiEntries[].pos` 改为
  "**本槽位内**的首个命中偏移"（编码不变 `槽位号*1e6+偏移`）。旧口径取"全部槽位里的最小偏移"，
  同一条目在多条消息里都命中时，其他消息里的那条**槽位号对不上被直接丢弃**，剩下几个边界被拉得极开 ⇒ 滚动时"一下跳好几个"。
- **锚点文本重定位**：快照条目新增 `hint = entryAnchorHint(visibleText)`（与 `extractAnchors` 同源）；
  `ensureEntryLayout` 先用它在正文里 `indexOf` 出起点，找不到才退回 pos 偏移，并且**不再 `Math.min(off, len-1)` 钳到末尾**
  （旧实现会把多条条目钳到同一个纵坐标，边界被抹平 ⇒ 一起亮）。量不到 / 倒挂的一律跳过（宁缺毋滥）。
- **判定语义**：`syncOutlineEntryActive` 改成"容器顶下方第一条还没滚过去的条目"（默认第一条），
  与消息块同一套语义 + 同一档 4px 容差。

**⑤ 目标块三边描边（口径⑤）**
- `.hcm-chunk-target` 由 `box-shadow: inset 0 0 0 2px accent`（整圈，左边与来源色条叠成两色）改成
  **上 / 右 / 下三条内阴影**、左边不画 ⇒ 左边缘保留的仍是 `.hcm-chunk` 自己的来源色条（蓝/绿/灰），其余三边紫色。
  不用真 `border` 是因为 2px 边框会改变盒尺寸、高亮切换时整篇文字横向跳动。

**验证（真实酒馆 1536×674 + `hideHelperDev.demo()`）**
- 「总共」→ 直接开出 `目录 126..546 | 完整上下文 558..978`（同顶同高、不重叠、页面 scrollY 恒 0、`#hide-helper-context-viewer` 为 null）；
- 目录窗内 `.hide-entry-sub` 计数 0、`#hco-toolbar` 存在；`hcm-head` = `flex-row`、sub 与 legend 均 `flex-end`；
- 筛选「世界书」→ 7 行降为 2 行、`#hco-count` = `显示 2 / 7 条`；检索 `ELDORIA` → `mark[data-hit]`=3、`1/3`、当前命中 + 所在行描边、目录 `scrollTop` 111；
- 滚动联动逐 8px 采样：`#1` 内依次 `前置常驻条目 → 前置动态条目 → 前置细节条目`、`#2` 内 `后置常驻条目 → 后置动态条目`，
  每条边界只切一次（旧实现会成片跳）；
- 键盘 ↑/↓/Home/End → 内容区 `0 → 46 → 112 → 566 → 0`；
- 点条目行 → `.hcm-chunk-target` = 第 1 块，`box-shadow` 三条（上/下/右）为 accent 紫、`border-left` 仍是 `#0ea5e9`；
- 目录 → 条目来源（`目录 | 条目来源`，内容窗收起）→ 点条目 → 三窗 `条目详情 872..1292 | 条目来源 440..860 | 目录 8..428` 互不重叠 →
  逐级关闭后回到成对窗口（内容窗 `flex`）；
- `node --check index.js` OK、CSS 大括号配平、无"普通规则块内嵌套 `{`"。

---

### 2.40-R1 第二十九轮修订：检索行默认收起 / 快照信息与标题同行 / 目录居中 + 内容贴右 / 去掉跳转箭头 / 详情窗去掉「插入位置」

**用户口径（原话）**：
1. 「这个检索框 `hc-search hc-search-row` 请默认收起。点击放大镜展开，再次点击则收起。」
2. 「`hide-entry-book` 要与标题"完整上下文"在一行。」
3. 「"上下文目录"弹窗请显示网页中央，而"完整上下文"弹窗显示在网页右侧。」
4. 「去除条目中的 `hco-jump` 图标及功能。」
5. 「对于条目详情弹窗来说，不显示 `tub-nli-pos` 信息。」（追问后明确：指**点条目后弹出的「条目详情」窗**里那行「插入位置」）

**① 检索行默认收起 —— 根因是 `!important` 压掉了内联样式**
- CSS 原来写的是 `.hc-search { display: flex !important; }`，而模板上是 `style="display: none;"`。
  `!important` 的优先级高于**内联**样式 ⇒ 检索行永远展开，JS 里 `style.display = 'none'` 根本不生效。
- 修法：CSS 改成 `.hc-search { display: none !important; }` + `.hc-search.hco-search-open { display: flex !important; }`，
  `applySearchRowState(open)` 同步切 `hco-search-open` 类；`ctxSearchRowOpen` 仍是模块级状态。
- 实测：默认 `display:none`、点放大镜 `flex`、再点回 `none`。

**② `hide-entry-book` 与标题同一行**
- 结构改动：`#hcm-book` 从 `.hide-entry-sub` **里面**提出来，成为 `.hide-entry-head` 的第二个孩子
  （`标题 → 快照信息 → sub(图例)`），`renderMergedContext` 分别写 `#hcm-book` / `#hcm-sub`。
- CSS：`.hide-entry-book` = `flex: 0 1 auto; min-width: 0; nowrap + ellipsis`（长了只截自己，不挤标题和图例）；
  `.hide-entry-sub` 由 `flex: 1 1 auto` 改 `0 0 auto + margin-left: auto`（图例继续贴最右）。
- 实测：`title.top == book.top`（同一行，差 7px 属于基线对齐）；图例右缘 = 内容区右缘（`overflow = 0`）。

**③ 目录居中、完整上下文贴右**
- `layoutOutlineMergePair()` 重写：内容窗 `left = vw - w - PAIR_MARGIN`（贴右、距边 8px）；
  目录优先 `(vw - w) / 2`（居中）。
- **互不重叠优先于居中**：若居中后的目录会压到内容窗，目录左移到 `mrgLeft - PAIR_GAP - w` 刚好让开；
  只有连"两窗并排"都放不下（`outLeftMax < PAIR_MARGIN`）才退回"居中叠放、目录压上层"。
- **顺带补一个真缺口**：窗口 `resize` 原来只在「魔杖按钮打开主弹窗」时绑，demo / 控制台入口打开的窗口不跟随
  ⇒ 视口一变（如 1536 → 900）两窗还停在旧坐标、直接跑出视口。新增全局只绑一次的
  `ensureChildPopupResizeHandler()`（`openContextOutlinePopup` 里调用；`pairCurrentId || ctxMergeMode` 时重排）。
- 实测：1536×674 → 目录 `left=558`（中心正好 768 = 视口中心，偏差 **0**）、内容窗 `left=1108`（距右 **8**）、
  间隔 130、不重叠；900×674 → 目录 `40`、内容窗 `472`（仍贴右，距右 8）、间隔 12、不重叠、都在视口内、未叠放。

**④ 去掉 `.hco-jump` 图标及功能**
- 项头 / 分组头行尾的"跳到正文"箭头（`.hco-jump`）+ 它的 `onclick` 绑定 + CSS 规则全部删除；
  `HC_ICON_JUMP` **保留**（「条目来源」子窗的 `.tub-entry-jump` 还在用，那是另一个功能）。
- 跳转能力没丢：聊天子行 / 条目行点击仍会跳（`jumpContextOutlineToMessage` / `jumpContextOutlineToEntry`）。
- 实测：`#hco-body .hco-jump` 计数 **0**。

**⑤ 条目详情窗去掉「插入位置」行**
- `buildEntryDetailHtml` 删掉 `row('插入位置', …)` 与配套的 `positionText = entryInsertPositionText(entry)`；
  现在详情窗只剩 **关键词 / 次关键词 / 顺序 / 递归 / UID+tokens / 内容**。
- `entryInsertPositionText` 本身**保留**（条目列表行第二行 `.tub-nli-pos` 仍在用）。
- 实测（静态提取 `buildEntryDetailHtml` 里的 `row()` 调用）：`关键词 / 次关键词 / 顺序 / 递归`，无「插入位置」。

**验证**：`node --check index.js` OK、CSS 大括号 510/510 配平；真实酒馆 1536×674 逐条实测上述五项；
窄屏 900×674 复核不重叠且不出界；截图 `round34-five-items.png` / `round34-final.png`。

---

### 2.41 第三十轮：成对窗口（目录 | 完整上下文）间距回到 PAIR_GAP = 12px

**用户口径（原话）**：
1. 「两个弹窗之间的间距怎么隔这么远？我让你改位置，你别把两个弹窗之间的间距给改了啊。」
2. 「我只是让你把"目录弹窗"放到左边，"上下文"弹窗显示在右侧。你给我瞎改什么间距？」

**① 根因：落点被拆成"两条各自独立的规则"，间距就不再由 PAIR_GAP 决定**
- 第二十九轮 R1 把 `layoutOutlineMergePair()` 写成：
  - 目录 = 视口水平居中 `(vw - w) / 2`
  - 完整上下文 = 贴右边缘 `vw - w - PAIR_MARGIN`
- 这两个位置**各自算各自的**，中间留多少完全取决于视口宽度：
  1536 宽 + 420 窗 ⇒ 目录 `558..978`、内容窗 `1108..1528` ⇒ **间隔 130px**。
  `PAIR_GAP = 12` 在那条分支里只用于"会不会重叠"的判断（`outLeftMax`），**不参与实际落点**。
- 与第二十七轮的原始口径偏离：`CHANGELOG §2.38` 记的是 `目录 558 | 合并内容 990`（间距 **12**），
  即和其它所有父子窗**同一套**规则（父窗原位、子窗 `+w+PAIR_GAP`）。

**② 改法：只改位置算法，间距恒定 `PAIR_GAP`**
```js
const pairWidth = 2 * w + PAIR_GAP;
const overlay = pairWidth > (vw - 2 * PAIR_MARGIN);   // 并排放不下才叠放
let outLeft;
if (overlay) {
    outLeft = clampIn(Math.round((vw - w) / 2));       // 叠放：两窗同坐标，目录压上层
} else {
    outLeft = slot.left;                               // ① 目录 = 主弹窗原槽位
    if (outLeft + pairWidth > vw - PAIR_MARGIN)        // ② 整对放不下 → 整对一起左移
        outLeft = vw - PAIR_MARGIN - pairWidth;
    if (outLeft < PAIR_MARGIN) outLeft = PAIR_MARGIN;  // ③ 仍越界 → 整对贴左
}
const mrgLeft = overlay ? outLeft : outLeft + w + PAIR_GAP;   // 内容窗永远贴目录右侧 12px
```
要点：
- **整对一起移动**（不是把某一窗单独钉到边缘）—— 这是间距恒定的前提；
- 目录不再死抠视口正中，改为落在**主弹窗原槽位**（与 `layoutChildPair` 的其它父子窗一致）；
- 内容窗不再钉右边缘，而是 `outLeft + w + PAIR_GAP`；
- `PAIR_MARGIN`（8）只当视口安全边距用，不再充当"贴右"的锚点。

**③ 实测（真实酒馆，`getAnimations().finish()` 结束入场动画后量 rect）**

| 视口 | 目录 | 完整上下文 | 间距 | 距右 | 判定 |
|---|---|---|---|---|---|
| 1536×674 | `558..978` | `990..1410` | **12** | 126 | 并排、同顶 |
| 900×674 | `40..460` | `472..892` | **12** | **8** | 并排（整对左移到贴右）、都在视口内 |
| 700×700 | `140..560` | `140..560` | — | — | `2×420+12=852 > 684` ⇒ **叠放**，目录 `.hh-pair-front`(10001) 压内容窗 `.hh-pair-under`(10000) |

- `node --check index.js` OK；
- 窄屏叠放时两窗同坐标、层级正确（`hh-pair-front` / `hh-pair-under`）；
- 900 宽下"整对左移"生效：右窗 `892` = `vw - 8`，目录跟着左移到 `40`，间距仍是 12（旧实现是目录 40 / 内容窗 472，间距 12 属巧合——那条分支恰好也用了 `PAIR_GAP`）。

**④ 踩坑记录（别再犯）**
- 不要用 `getBoundingClientRect()` 判断间距：入场动画 `hhFadeIn` 的 `scale(0.95)` 会让 rect 缩 5%
  （实测 rect 间距 **33** vs inline 间距 **12**）；自动化页面 hidden 时动画时钟冻结、`finish()` 也不会自动跑完。
  判据一律用 **inline `left` + `offsetWidth`**（`readWindowSlot` 同一套口径）。
- `demo()` 打开的窗口不经过"魔杖按钮"那条路径，`resize` 监听要依赖 `ensureChildPopupResizeHandler()`；
  量测前先重新 `demo()` 触发一次落点重算，别指望 resize 一定已绑上。

**验证**：`node --check index.js` OK；真实酒馆 1536×674 / 900×674 / 700×700 三档实测；
截图 `pair-gap12-1536-final.png`。

---

### 2.42 第三十一轮：「完整上下文」右对齐 / 目录筛选修复 / 目录行样式统一 / 模拟发送自定义弹窗

**用户口径（原话）**：
1. 「`hide-entry-book` 要右对齐。」
2. 「`hc-filters` 的筛选功能根本没用，只有"全部"。」
3. 「`hco-src` 标题文本颜色加深些，现在的文本颜色看不清，同时对于聊天 `hco-pill` 去除，`hco-tokens` 右对齐。
   而 `hco-role hco-role-ai` 的 "ai" 大写 "AI"，并且 `hco-role hco-role-user` 的背景颜色也改为
   `hco-role hco-role-ai` 一样的灰色。」
4. 「模拟发送的弹窗你美化一下，不要使用 popup，使用与插件一套风格主题的自定义弹窗（日夜双主题）。」

#### ① `hide-entry-book` 右对齐 —— 不能直接给 book 写 `margin-left: auto`

第一版改法（错）：给 `.hide-entry-book` 加 `margin-left: auto`。结果 book 与 `.hide-entry-sub` 的
`margin-left: auto` **平分**剩余空间 ⇒ book 停在中间而不是右侧。
（实测：head `466..834`、title `466..546`、book 左缘 `824` 但宽度 0 —— 两段 auto 边距把空间撕成两半。）

正解：**整行 `justify-content: flex-end` + 标题 `margin-right: auto`**，并把 `.hide-entry-sub` 的
`margin-left: auto` **删掉**（留着会与标题的 auto 打架）。这样右侧那一串（book + 图例）整体贴右，
标题被 auto 边距顶到最左。

实测（1536×674，`demo()` 快照）：`head 466..834`、`title 466..546`（左缘对齐行首）、
`sub 右缘 = 834 = head 右缘`（`subAtRightEdge = 0`）。

#### ② `hc-filters` 筛选"根本没用" —— 根因是**菜单没有任何基础样式**

排查过程：
- 点「来源」→ 菜单 DOM **建出来了**（4 项：全部/聊天/世界书/其他）、`toggleCtxFilterMenu` 正常、
  `onPick` 回调正常、选完之后 `ctxSourceFilter` 也确实变成了 `wi`、目录行 **7 → 2** —— **筛选逻辑本身是好的**。
- 但 `getComputedStyle(menu)` 全是空值：`position: ""`、`zIndex: ""`、`bg: ""`。实测 rect：
  `{x: 0, y: 674, w: 1536, h: 24}` —— **菜单排在文档流末尾、满宽、正好在 1536×674 视口的下边缘之外**。
- 根因：`.hh-menu` 在 `style.css` 里**只有一个** `.hh-menu[data-theme="dark"]` 的**变量**选择器
  （第二十七轮 R2 加的那条，注释写着"菜单挂在 body 上 → 暗色要自带一份变量"），
  **从来就没有**
  `.hh-menu { position: fixed; ... }` 与 `.hh-menu-item { ... }` 这套基础规则。
  于是菜单是个普通 `div`：`position: static`、`z-index: auto`，内联的 `left/top` 对 static 元素无效。
- 用户感受：点「来源」时菜单其实弹了出来，只是在视口外面 ⇒ 看起来"只有全部、点了没反应"。

修法：补一整套基础规则 ——
`position: fixed !important` + `z-index: 100000` + `flex column` + `min-width 88px` +
`background: var(--hh-background-popup)` + 边框/圆角/阴影 + item 的 padding/hover/active。
⚠️ **left/top 绝不能写 `!important`**：`openCtxFilterMenu` 是用**内联** `style.left/top` 定位的，
`!important` 会把它压掉（与 §2.40-R1 检索行踩的是同一类坑）。

实测（修复后）：`position: fixed`、`zIndex: 100000`、`88×128`、`rect {x:584, y:135}`、`inViewport: true`；
点「世界书」→ 目录 7 行 → **2 行**、按钮文案 `来源 世界书`、计数 `显示 2 / 7 条`；
「角色」菜单 4 项（全部 / system / user / assistant）同样在视口内。

#### ③ 目录行样式四条

| 项 | 改法 | 实测 |
|---|---|---|
| `.hco-src` 颜色加深 | 显式 `color: var(--hh-text-primary)`（原来继承 `--hh-text-secondary`，在浅灰项头上几乎看不清） | 亮色 `rgb(52,58,64)`、暗色 `rgb(224,224,224)` |
| **聊天**分组头去掉 `.hco-pill` | JS 侧按 `g.source` 分支：「聊天」组不渲染 pill（它和 `#首–#末` 区间徽标 + 「连续 N 条」三处重复）；「其他」组保留（否则看不出是哪种来源） | `#hco-body .hco-pill` 计数 **0**（demo 快照里只有聊天分组） |
| `.hco-tokens` 右对齐 | `margin-left: auto` + `text-align: right`（即使前面没有 `flex:1` 的可伸缩列也钉在行尾） | 每条 tokens 右缘距父行右缘 **9px**（= 行 padding）、`text-align: right` |
| `ai` → **`AI`** | JS `roleChip` 的 label 单独取（类名仍是 `hco-role-ai`，暗色/单测都按类选） | chip 文本 `AI` |
| `user` 改灰底 | `hco-role-user` 与 `hco-role-ai` 合并成同一条规则（都 `--hh-btn-secondary-hover-bg`） | 亮色两者都 `rgb(233,236,239)`、暗色都 `rgb(61,61,61)` |

#### ④ 「模拟发送」弹窗改为插件自绘（日夜双主题）

原来走 ST 的 `callGenericPopup(..., POPUP_TYPE.INPUT)` —— 外观是酒馆原生弹窗，与插件的紫调/暗色
主题完全两套。改为新函数 **`askSimulateContent()`**（`index.js`，`triggerSimulate` 之前）：

- 骨架复用 `.hide-modal-overlay` + `.hide-modal-box`（与「更新日志」弹窗同一套）；
- 变量全部取 `--hh-*`，`askSimulateContent` 按 `extension_settings.hide.theme` 给遮罩打 `data-theme="dark"`
  ⇒ **日夜自动跟随插件主题**（`applyTheme` 本来就覆盖 `.hide-modal-overlay`）；
- 结构：标题行（含右上 ×）+ 说明条（左侧 accent 竖条 + `--hh-accent-color-bg-very-light` 底）+
  「自定义内容（可留空）」label + `textarea`（focus 时 accent 描边 + 3px 光晕）+
  右下「取消 / 模拟发送」；
- 交互：`Esc` / 点遮罩空白 / × / 取消 → `resolve(null)`；确认 → `resolve(ta.value)`（可为空串）；
  **Ctrl/Cmd+Enter 也确认**（textarea 里裸 Enter 留给换行）；
- 返回 Promise 的契约与原 `callGenericPopup` 那一段完全一致（`typeof value !== 'string'` 即取消），
  故 `runSimulateSend` 一行没动。
- **清理**：`index.js` 顶部 `import { POPUP_TYPE, callGenericPopup } from "../../../popup.js";` **整条删除**
  （已无任何使用点）。

实测：亮色 `boxBg rgb(255,255,255)` / 暗色 `rgb(30,30,30)` + `color rgb(224,224,224)`、
textarea 同步换色、提示条暗色 `rgba(216,168,231,.08)` + 左边框 `rgba(216,168,231,.95)`；
`cancelWorks: true`（点取消后遮罩被移除）；`isStPopup: false`（确认不再是 ST 原生弹窗）。

**验证**：`node --check index.js` OK；CSS 大括号 **525 / 525** 配平；
真实酒馆 1536×674 亮/暗双主题逐项实测（见上表数字）；
截图 `r31-01-pair.png`（成对窗口 + book 右对齐）、`r31-02-filter-menu.png`（筛选菜单在视口内）、
`r31-03-outline-final.png`（亮色目录：AI 大写 / 灰底 chip / 无 pill / tokens 右对齐）、
`r31-04-sim-light.png` / `r31-05-sim-dark.png`（模拟发送弹窗日夜双主题）、`r31-06-outline-dark.png`（暗色目录）。

---

### 2.43 第三十二轮：「条目来源」子窗去掉插入位置 / 筛选同步右侧完整上下文

**用户口径（原话）**：
1. 「对于 `hide-helper-wi-msg-popup` 的世界书条目详情弹窗中，去除 `tub-nli-pos` 显示。」
2. 「当在 `上下文目录` 中筛选 `hc-filters` 时，右侧完整上下文弹窗内容也要更新至筛选后的内容与数据。」

#### ① 「条目来源」子窗去掉 `.tub-nli-pos`（保留顺序）

改法：`createEntryRowBuilder` 增参数 **`insertPos = true`**；`false` 时 `metaLine` 只渲染顺序 chip，
容器类名改成 `tub-nli-meta-orderonly`（CSS 里 `justify-content: flex-end` 把 chip 推到最右，
右缘与上一行的 tokens 列对齐）。`mao` 只改**调用点**：

```js
// 「条目来源」子窗（~6510）
createEntryRowBuilder({ ..., insertMeta: true, insertPos: false });
// 「上下文目录」条目区（~7117）不动 —— 那里每条条目的插入位置不一样，是有信息量的
```

理由：`#hide-helper-wi-msg-popup` 里的条目**全属于同一条消息**，插入位置每行都一样（本例都是
「角色定义前」），纯冗余；顺序才是个体差异。

CSS 兜底：`#hide-helper-wi-msg-popup .tub-nli-pos { display: none !important; }` ——
万一 DOM 里还残留（旧快照 / 其它调用点），也不会露出来。

实测（demo 快照，3 个条目）：`.tub-nli-pos` **0** 个、`.tub-nli-order` **3** 个（10 / 20 / 30）、
`metaCls = tub-nli-meta tub-nli-meta-orderonly`、`justify-content: flex-end`、
**顺序 chip 右缘 − tokens 列右缘 = 0**（三行都是 0）。
「上下文目录」对照：条目行仍是「角色定义前 / 10」两列（未被误伤）。

⚠️ **自踩**：第一次验证时 `posCount=0` 但 `orderCount` **也是 0** —— 不是改错了，而是
`makeDemoContextDump()` 的演示条目**从来没带 `position/order` 字段**，`metaLine` 按"字段缺失就整行不渲染"
把整行都省了。已给演示数据补上 `position/depth/role/order`（前置=0、后置=1，order 10/20/30/40/50），
这样"只去掉插入位置、顺序仍在"才是可验证的。**这个坑值得记**：凡"某行没显示"的报障，先查数据字段是否齐，
再怀疑 CSS/DOM。

#### ② 筛选同步右侧「完整上下文」

原来筛选**只作用于左侧目录**（第二十九轮的设计口径就是"右侧恒显示全部内容"），于是"左边筛出 2 条、
右边还是 7 条"自相矛盾。本轮按用户口径改成**两侧同源同口径**。

改法（关键是**只留一个筛选入口**，避免两处各写一份 `filter(...)` 而漂移）：

```js
// 新增纯入口（供目录 / 内容区 / 计数 / 键盘导航共用）
function ctxVisibleMessages() { return (ctxDump?.messages || []).filter(m => ctxMessageVisible(m, {...})); }
function ctxFilterActive() { return ctxSourceFilter !== 'all' || ctxRoleFilter !== 'all'; }
```

替换的四处：

| 位置 | 原来 | 现在 |
|---|---|---|
| `renderContextOutlineList` | 内联 `all.filter(...)` | `ctxVisibleMessages()` |
| `renderMergedContext` | **`all`**（全部） | `ctxVisibleMessages()` —— 分块 / 图例 / 头部计数全按它 |
| `ensureMergedChunksUpTo`（懒渲染补块） | `ctxDump.messages` | `ctxVisibleMessages()`（否则会去补一个根本没渲染的块） |
| `handleContextHotkeys`（↑↓/J/K/Home/End） | 内联 filter | `ctxVisibleMessages()` |

顺带两处一致性修正：

- **头部计数改报筛选后**：`模拟发送 · 筛选后 2 / 7 条 · 共 1040 tokens`（原来是恒定的"7 条 · 1576 tokens"，
  与右侧实际内容不符）；未筛选时仍显示原来的文案。
- **图例跟筛选走**：`hcmLegendHtml(shown)`（原来是 `hcmLegendHtml(all)`）—— 否则图例说"聊天 3 条"、
  下面正文一条聊天都没有。
- **新增 `ctxVisibleMsgByIndex`**（序号 → 记录，只含可见消息）：`ensureEntryLayout` / `jumpContextOutlineToEntry`
  这些"按序号找消息"的地方改为优先读它。它们原来无条件读 `ctxMsgByIndex`（**全部**消息），
  筛选后内容区里没有那条块、却还能从全量表里取到正文，条目定位会错位。
- 筛选事件处理器：`onPick` 里 **`renderContextOutlineList()` + `renderMergedContext()`** 成对调用
  （原来只有前者）。
- 空态：筛选到 0 条时右侧显示「当前筛选条件下没有消息。」（与左侧同一句），不再显示"没有可用的快照"。

实测（`demo()` 快照，7 条 / 1576 tokens）：

| 操作 | 左目录 | 右内容区 | 头部 | 图例 |
|---|---|---|---|---|
| 无筛选 | 7 行 | 7 块 | `模拟发送 · 7 条消息 · 共 1576 tokens` | `聊天 3 条 · 216 t / 世界书 5 条 · 1040 t / 其他 2 条 · 320 t` |
| 来源 = 世界书 | **2 行** | **2 块**（`hcm-kind-wi-after` ×2） | `筛选后 2 / 7 条 · 共 1040 tokens` | `聊天 0 · 0 t / 世界书 5 条 · 1040 t / 其他 0 · 0 t` |
| 再叠加角色 = assistant | 0 行 | 0 块 | `筛选后 0 / 7 条 · 共 0 tokens` | 全 0；正文 = 「当前筛选条件下没有消息。」 |
| 角色单独 = assistant（来源复位） | 1 行 | 1 块 | `筛选后 1 / 7 条 · 共 180 tokens` | — |

**验证**：`node --check index.js` OK；CSS 大括号 **527 / 527** 配平；
真实酒馆 1536×674 逐项实测（上表数字全部来自 `browser_evaluate` 实测，非推断）；
截图 `r32-01-filter-sync.png`（筛选世界书：目录 2 行 + 右侧只剩 2 段世界书正文 + 头部「筛选后 2 / 7 条」）、
`r32-04-wimsg-final.png`（条目来源子窗：只剩顺序 chip 10/20/30、无「角色定义前」）。

### 2.44 第三十三轮：聊天统计四条口径（过滤未匹配 / 去导航分割线 / 对比页三项 / 锚点 chip 与模拟发送重排）

**用户口径（原话）**：
1. 「把『已激活』条目中『未匹配』的条目过滤掉，不显示。而把『对比』页面中，两侧都是未匹配/未激活的条目也过滤掉。」
2. 「把 `popup-tabs-nav` 的下面的 border 分割线去除。」
3. 「当切换到『对比』页面时，不显示 `tub-simulate-btn` 模拟发送按钮。并且，`tub-anchor-baseline-select` 去除右侧的倒三角图标，同时将下拉列表改为像 `tub-type-filter` 一样菜单的形式弹窗。」
4. 「将 `tub-anchor-chip` 中的时间去除，只显示 messageID。而对于模拟生成的则将标题文本改为 `%{#N+1}`，同时移至 `tub-anchor-chip` 的右侧紧贴着。最后，将模拟发送 `tub-anchor-chip` 按钮移至原来 `tub-anchor-chip tub-anchor-chip-sim` 的位置，并且将边框改为圆角。」

追问确认（`ask_user_question`，两条）：
- `%{#N+1}` = **算出真实数字**（`#9` → 显示 `#10`），而不是字面写 `#N+1`；
- 锚定栏最终排法 = 第一行只有「明细 / 对比」、第二行 `[#7] [#9] [#10] …… [模拟发送]`。

#### ① 过滤「未匹配」条目（明细列表 + 对比差异表）

明细「已激活条目」：`renderEntriesList` 的 `pushEntries` 在入列前加一行守卫 ——

```js
// matched=false = 已激活但没在实际发送的上下文里定位到内容（tokens 恒为 0）
if (!e.matched) return;
```

顺带修掉一个连带死路：`availableListTypes()` 原来按 `(rec[t] || []).length` 判断"该类型存在"，
过滤未匹配后会出现「菜单里有『动态』、点开却一条都列不出来」——改成 `.some(e => e.matched)`。

对比「世界书条目差异」：`buildEntryDiff` 在原本的"差异签名"判断**之前**加一条 ——

```js
// 两侧都是未匹配（tokens=0）或未激活（present=false）→ 这行没信息量，直接不显示
const isBlankCell = (i) => !present[i] || vals[i] === 0;
if (present.every((_, i) => isBlankCell(i))) continue;
```

只影响"**两侧都空**"的行：一侧有真实 token、另一侧未激活/未匹配的差异行**仍然保留**
（那正是用户要看的"这次多激活了这条 / 这次它没匹配上"）。

实测（WuWa 聊天、锚点 #9）：明细列表 **39 → 34** 行、`未匹配` 行 **5 → 0**；
「上下文顺序」排序下同样 `unmatchedRows = 0`（两条渲染路径共用同一个 `pushEntries`）；
对比页 `#7 → #10` / `#9 → #7` / `#7 → #9` 三种组合下「两格都未匹配/未激活」的行数 **均为 0**。

#### ② 去掉 `popup-tabs-nav` 的下分割线

```css
#hide-helper-popup .popup-tabs-nav { border-bottom: none !important; /* 原：1px solid var(--hh-border-color) */ }
```

实测 `getComputedStyle(nav).borderBottom` = `0px|none`。滑动下划线的落点（`bottom: -1px`）未动。

#### ③ 对比页：不显示模拟发送 + 对比组合改菜单

- **模拟发送按钮只在明细视图渲染**：`renderAnchorBar` 里 `simBtn` 原本无条件拼进 head 的
  `.tub-anchor-label`；本轮它只拼进 `isDetail` 分支的 `.tub-anchor-chips`。对比页实测
  `!!document.getElementById('tub-simulate-btn') === false`。
- **`tub-anchor-baseline-select` 由原生 `<select>` 换成菜单按钮**：原生 select 右侧的倒三角
  （`appearance` / 背景图）去不干净，故整体换成 `<button class="tub-anchor-baseline-select">`，
  点开复用**同一套** `openTypeFilterMenu`（与 `tub-type-filter` 完全一致的浮层、同一时刻只允许一个、
  点别处 / `Esc` / 滚动 / 缩放收起）。新增 `bindBaselineMenu()` 承载这段（选项 = `enumerateCompareCombos` 的
  `基准 → 对比` 全组合，`value` 仍用 `|` 分隔）。
- CSS：`.tub-anchor-baseline-select` 改成与 `.tub-type-filter` 同款药丸按钮（`border-radius: 15px`、
  透明底、`appearance: none`、`background-image: none`），并补 `.open`（展开高亮）/`.is-empty`（无可比组合）两态。

实测：触发器 `tagName = BUTTON`、`backgroundImage = none`、`appearance = none`；
点开菜单 2 项（`#7 → #9` / `#9 → #7`，有模拟锚点时 6 项并含 `#10`）、
选中后头部两列变 `#9基准 / #7对比`、`Esc` 收起且 `.open` 类被清掉。

#### ④ 锚点 chip 只留 messageID；模拟 chip = `#N+1` 紧贴右侧；模拟发送移到行尾并改圆角

- **`formatAnchorLabel(item, simDisplayId)`**：真实锚点 `#${index}`（时间不再进 chip，挪进 `title`
  「锚定时间 HH:MM:SS」）；模拟锚定 `#${simDisplayId}`。
- 新增 **`simAnchorDisplayIndex(list)`**（纯函数口径）：取**最后一条真实锚点 + 1**（无真实锚点时退回
  `getContext().chat.length`）。⚠️ 这里刻意读 `getContext()` 而非 `getContextOptimized()` —— 后者有缓存，
  会滞后于新消息。
- **排布**：`.tub-anchor-chip-sim` 的 `margin-left: auto` 删掉（原来推到行尾），改为紧贴真实 chips；
  `margin-left: auto` 转给 `.tub-anchor-chips .tub-simulate-btn`，于是**模拟发送落在行尾**——
  正好是原来模拟 chip 的位置。`.tub-simulate-btn` 的 `border-radius: 8px → 999px`（药丸），字号与 chip 对齐（0.75em）。
- 对比视图的列头 / 差异表表头里的「模拟」也统一成 `#${simDisplayId}`（同口径，避免一处 `#10`、一处「模拟」）；
  `renderEntryDiffTable` 增第 6 个参数 `simDisplayId`（由 `renderCompareContent` 按**整份列表**算好传入 ——
  在 `cols` 里现算会偏小，因为 `cols` 只有两列）。

实测（模拟发送一条后，锚点 #7 / #9 / 模拟）：
chips = `#7` / `#9` / `#10`（模拟 chip 的 `title` = 「模拟内容：…」）；
模拟 chip 左缘 − `#9` 右缘 = **6px**（= 行内 `gap`）；模拟发送按钮右缘 **956** = chips 行右缘 956（贴行尾）、
`border-radius = 999px`、`marginLeft = 153.3px`（吃满剩余空白）；
对比页表头 = `#9 / #7`，含模拟的组合表头 = `#7 / #10`。

#### 文档 / 验证

- `node --check index.js` OK；`style.css` 大括号 **529 / 529** 配平，无"普通块内嵌套 `{`"
  （自检脚本只报 4 处 `@keyframes` 里的 `from`/`to`，属合法键帧）。
- 真实酒馆（1536×674，WuWa 聊天，10 条消息、锚点 #7/#9 + 模拟）逐项 `browser_evaluate` 实测，数字见上；
  截图 `r33-detail2.png`（明细：无导航分割线、`#7 #9` 同行 + 「模拟发送」药丸在最右）。
- ⚠️ `style.css` 由酒馆在**页面加载时**注入、不热更新 —— 改完必须刷新页面才生效。
- ⚠️ 截图工具本轮在对比页会卡在 `waiting for fonts`（Playwright 侧超时，非页面报错）；
  对比页的三项改动因此改用 `browser_evaluate` 量测（`tagName` / `backgroundImage` / 表头文本 / 菜单项）验证。

### 2.45 第三十四轮：去掉「聊天」统计子窗里的 `hcs-make-snapshot` 按钮

**用户口径（原话）**：「把 `hcs-make-snapshot` 按钮去除。」

**改法**（`openChatStatsPopup`，`index.js`）：`ctxBtnHtml` 原本是「有快照 → 查看完整上下文 / 无快照 → 没有快照 · 点此生成」二选一，
本轮把**无快照那支直接改成空串**，并删掉配套的 `mkBtn.onclick` 绑定（`refreshContextSnapshot({ openViewer: true })` 这条调用
随之消失；函数本身**保留** —— 控制台 `hideHelperDev.snapshot()` 仍在用它）：

```js
const ctxBtnHtml = ctxHasDump
    ? `<button type="button" class="hcs-ctx-btn" id="hcs-open-context">查看完整上下文（… 条）</button>`
    : '';                       // ← 原来是 `hcs-make-snapshot`「没有快照 · 点此生成」
```

`.hcs-ctx-btn` 的样式**未动**（「查看完整上下文」还在用它）。

**为什么不需要"生成"入口了**：统计页顶部本来就有「模拟发送」按钮（第三十三轮已挪到锚点 chips 行最右），
它跑完同样会落快照；子窗里再放一个生成按钮属于重复入口。

**实测**（真实酒馆 1536×674，WuWa 聊天）：
- **无快照分支**：点「聊天」统计框 → 子窗打开，`#hcs-make-snapshot` 为 **false**、`#hcs-body` 内 `button` 计数 **0**，
  内容到第二条折线（生成速度）就结束，DOM 尾部是 `</div>\n        \n    `（空行，无按钮）。
- **有快照分支**（先点一次「模拟发送」产生快照 → 锚点切到 `#10`）：子窗里只剩
  `<button id="hcs-open-context">查看完整上下文（16 条）</button>`，`#hcs-make-snapshot` 仍为 **false**。
- `node --check index.js` OK；全仓 `hcs-make-snapshot` 仅剩一行注释提及。

⚠️ 量测时踩到一次"假空"：第一次读按钮时子窗其实是 `display:none` —— 再点「聊天」框等于**收起**（toggle 语义），
不是渲染失败。验证子窗内容前要先确认 `getComputedStyle(popup).display === 'flex'`。

---

### 2.46 第三十五轮：锚定栏重构 + 十一项 UI 口径

**用户口径（原话，十一条）**：
①「将 `tub-anchor-chip` 合并成一个选项菜单，默认显示最新 messageID，菜单项名称改为 `第 %{messageID} 楼/模拟发送`。
   然后，去除原有的右侧的『模拟发送』按钮 `tub-simulate-btn`。最后，再原来 `tub-simulate-btn` 按钮位置增加两个图标
   用来打开 `hide-helper-ctx-outline-popup` 和 `hide-helper-chat-stats-popup`。」
②「在 `hide-helper-chat-stats-close` 中过滤掉 messageID=0（#0）的数据统计。」
③「`hide-helper-chat-stats-close` 这个区域的布局不太合理… 第二行 `WuWa World Info MVU` 的右侧是有大片空白区域的，
   但是 `Eldoria` 却挤到了第三行。」
④「在移动端 `tab-button` 的标题文本都换行，这非常影响美观。」
⑤「在『模拟发送』生成阶段，请出现一个提示弹窗提示正在生成。」
⑥「在暗黑主题下，`hide-entry-checks` 的勾选框中的对钩请改为黑色。」
⑦「将 `tub-anchor-baseline-select` 中的『→』换成『：』冒号。」
⑧「在 `hide-helper-ctx-outline-close` 上下文目录中，对于没有展开内容的项目来说，隐藏其 `hco-caret` 按钮图标。」
⑨「在移动端时，『完整上下文』弹窗 `hide-helper-ctx-merged-popup` 的左上角显示一个折叠栏图标，点击则可以展开/显示
   `hide-helper-ctx-outline-close` 弹窗。同时，`hide-helper-ctx-outline-close` 左上角也有个折叠图标用来展开/收起，
   右上角的关闭按钮默认隐藏起来，而 `hide-helper-ctx-merged-popup` 右上角显示关闭按钮。」
⑩「将 `hide-helper-ctx-merged-popup` 中 `hcm-book` 的『模拟发送』字符改为 `模拟 %{messageID}`。」
⑪「去除 `hide-helper-wi-msg-popup` 条目详情弹窗中的 `hc-msg-role hc-msg-role`，而对于 `hc-msg-kind` 统一与
   上下文目录弹出中条目的 `hco-src` 标题保持一致。最后去除 `hc-msg-tokens` 和 `tub-entry-jump` 按钮图标。」

追问确认两处歧义：①菜单里的「模拟发送」= **触发一次新的模拟发送**（已有模拟锚点仍以 `第 N 楼` 显示）；
⑨这套交互 **只在移动端生效**（桌面维持两窗并排 + 目录带 ×）。

#### ① 锚定栏重构（`renderAnchorBar` / `bindAnchorBarEvents` / 新增 `bindAnchorMenu`、`currentStatsModel`）

旧结构两行：`明细/对比` 切换 + 一长串 `tub-anchor-chip`（每条锚点一个）+ 行尾 `#tub-simulate-btn`。
新结构仍两行，但第二行只剩**一个菜单按钮**：

```
┌──────────────────────────────────────────┐
│ [明细][对比]                 [▤] [💬]    │   ← 行尾两个图标 = 原模拟按钮的位置
│ 锚定 [第 9 楼 ▾]                          │   ← 菜单：全部锚点 + 「模拟发送」动作项
└──────────────────────────────────────────┘
```

- **菜单项文案**：新纯函数 `anchorMenuItemLabel(item, simDisplayId)` → `第 ${index} 楼`；
  模拟锚定 → `第 ${simDisplayId} 楼`（= 最后一条真实 messageID + 1，与 `simAnchorDisplayIndex` 同源）。
  拿不到编号时退回 `模拟发送`，不显示错的楼层号。
- **菜单复用** `openTypeFilterMenu`：`normalizeMenuOptions()` 让它同时接受 `[v,t]` 与 `{value,text,cls}`
  （后者给「模拟发送」动作项挂 `.tub-type-menu-item-action`、给模拟锚点项挂 `.tub-type-menu-item-sim`）。
- **选中语义不变**：点最新一条 = 恢复"自动跟随"（`statsPinnedIndex = null`），否则固定。
  点「模拟发送」→ `triggerSimulate()`（与旧按钮同一条路径，含输入弹窗）。
- **两个图标**：`#tub-open-outline` → `toggleContextOutlineFromStats(model)`；
  `#tub-open-chat-stats` → `toggleChatStatsPopup(model, btn)`。
  模型由新函数 `currentStatsModel()` 现算（与 `renderTokenStatsContent` 入参同一套字段）。
- **`#tub-simulate-btn` 整体删除**：模板、CSS（`.tub-simulate-btn` 三条 + `.tub-anchor-chips .tub-simulate-btn`）、
  `runSimulateSend` 里对它 `disabled/textContent` 的读写全删；`bindAnchorBarEvents` 里 `.tub-anchor-chip`
  的点击绑定也一并删除（chips 已不存在）。`setupEventListeners` 里那条委托保留作兜底（无按钮时形同不存在）。
- **`formatAnchorLabel` 保留**：对比表列头（`#N` 的紧凑写法）还在用它。

#### ⑦ 对比组合的「→」→「：」

`renderAnchorBar` 的 `comboText` 与 `bindBaselineMenu` 的菜单项**两处一起改**（口径必须一致）：
`第 8 楼：第 9 楼`。触发器的 `title` 也从「基准 → 对比」改成「基准：对比」。

#### ② 聊天统计过滤 #0

`collectLatestAiMessages()` 循环里加 `if (i === 0) continue;`。原有"无生成时间就跳过"的规则保留 ——
但实测本机聊天里 **#0 是带 `gen_started/gen_finished` 的**（用户截图里的问题正来自此），只靠"无时间"过滤不掉，
必须显式按 messageID 剔除。

#### ③ 世界书筛选行布局（根因 + 修法）

**根因**：`.tub-book-filters-row` 是 `display:flex; justify-content:space-between`，左侧书签区 `flex:1 1 auto`、
右侧工具组是同行第二个 flex item。于是**书签区可用宽 = 整行宽 − 工具组宽**（实测 1440 视口 / 主弹窗 368px 时
只剩 **238px**）：「所有条目」62 + 「WuWa World Info MVU」128 就满了，`Eldoria` 只能换到下一行 ——
而右边明明还空着 121px（工具组自己的位置）。

**修法**：改成**两行** —— 书签区 `flex: 1 1 100%` 独占一行（拿到整行宽度），工具组 `margin-left: auto` 靠右独占下一行。
实测三个书签同排 `y=636`（`所有条目 / WuWa World Info MVU / Eldoria`），工具组落到 `y=666` 且靠右。

#### ④ 移动端标签换行

**根因**：390 视口 / 弹窗 351px / 导航可用宽 300px ⇒ 每格约 60px。四个汉字 ≈ 56px + 左右 8px padding
刚好越界，每格都折成两行（实测标签高 **60px**）。
**修法**：`.tab-button` 加 `white-space: nowrap` + 省略号 + `line-height: 1.5`；再用媒体查询把
`≤480px` 收成 `padding: 10px 4px / font-size: 13px`、`≤360px` 再收一档。实测 390 视口标签高 **38px**、四格都单行。

#### ⑤ 模拟发送"正在生成中"提示

新增 `showSimulateProgress()` / `hideSimulateProgress()`（+ `SIM_PROGRESS_ID`）：
`runSimulateSend` 开头 show、`finally` 里 hide（幂等，重复调用不会残留两个遮罩）。
遮罩走 `.hide-modal-overlay` 骨架 + `.hide-sim-progress-*`：转圈（`hhSimSpin`）+ 标题「正在模拟发送…」+ 一行说明；
**不可关闭、不阻止键盘**，`z-index: 100010`（高于输入弹窗 100002，避免"输入框还开着"的错觉）。

#### ⑥ 暗黑主题勾选对钩

**根因**：勾选态底色是 `--hh-text-primary`，而它在暗色下是**浅灰 `#e0e0e0`** ⇒ 白色对钩压上去对比度只有 ≈ 1.3:1。
**修法**：`#hide-helper-entry-popup[data-theme="dark"] .hide-entry-checkbox-on { color: rgba(0,0,0,1) }`。
实测：亮色 `color: rgb(255,255,255)` / `bg: rgb(52,58,64)`；暗色 `color: rgb(0,0,0)` / `bg: rgb(224,224,224)`。

#### ⑧ 无子项的项头不显示折叠箭头

`msgHeadHtml(m, hasKids)`：调用方传 `entries.length > 0`；为 false 时不渲染 `.hco-caret`、
也**不加 `.hco-collapsible`**（连折叠行为一起没有，避免"点了箭头不动"）。
实测目录：`其他 · main` / `其他 · authorsNote` 无箭头，两条世界书项 + `#0–#2` 分组头有箭头（共 3 个 caret）。

#### ⑨ 移动端单窗模式（成对窗口）

**新增状态**：`ctxPairMobile`（layout 算出的档位）、`ctxMobilePane`（`'merged'` 默认 / `'outline'`）。
**新增函数**：`pairFittedSlot()`（把进入模式时记下的槽位钳进当前视口）、`applyPairMobileChrome()`（打类）、
`setMobilePairPane()` / `collapseContextOutlineOnMobile()` / `expandContextOutlineOnMobile()` / `isMobilePairLayout()`。

- `layoutOutlineMergePair()` 末尾调 `applyPairMobileChrome(!!overlay)`：给 `<body>` 打 `hh-pair-mobile`，
  给两窗分别打 `hh-mobile-pane-merged` / `hh-mobile-pane-outline`。
- CSS：`body.hh-pair-mobile #…outline.hh-mobile-pane-merged` 与 `…merged.hh-mobile-pane-outline` → `display:none !important`。
- 折叠图标 `.hh-pair-toggle`：**桌面档 `display:none`**，移动档 `flex`（左上角 `top/left: 6px`）；
  两窗的标题同时 `padding-left: 34px` 让位。
- 关闭按钮：移动档 **目录隐藏 ×、完整上下文显示 ×**；桌面档保持原样（目录有 ×、内容窗无 ×）。
  `#hide-helper-ctx-merged-popup .hide-helper-popup-close-icon { display: none }` 显式写出来表明桌面口径。
- **实测修复（两个真 bug）**：
  1. **宽度溢出**：`ctxMergeViewerSlot` 是进入模式那一刻抓的，1440 记下的 420 宽在 390 视口里让右缘冲到 **1362px**，
     「共 1576 tokens」与右上角 × 全被挤出屏幕 → 新增 `pairFittedSlot()` 钳宽高/坐标（实测修后 374 宽、右缘 382）。
  2. **折叠图标点了没反应**：`isMobilePairLayout()` 原本自己重算档位（`ctxMergeViewerSlot || captureMainSlot()`），
     但这两个槽位在刚打开的页面上都是 null ⇒ 返回 false，而 layout 用的是带兜底对象的同一套。
     → 改为读 layout 算出的 `ctxPairMobile`，判定**只有一处**。

实测（390×844）：初始 `merged=flex / outline=none`、目录 × = none、内容 × = flex；
点内容窗图标 → `outline=flex / merged=none`；点目录图标 → 回 `merged=flex`。

#### ⑩ `hcm-book` 的「模拟发送」→「模拟 N」

新纯函数 `ctxDumpSourceLabel(dump)`：`source === 'simulate'` → `模拟 ${simDisplayIndex}`（拿不到编号退回 `模拟发送`）；
真实发送 → 仍 `实际发送`。`renderMergedContext` 的 `bookEl` 与 `renderContextOutlineList` 的
`titleEl.title` **两处共用**（避免口径漂移）。实测：`模拟 10 · 7 条消息 · 共 1576 tokens`。

#### ⑪ 条目来源子窗头部瘦身

- 删 `.hc-msg-role`（标记 + 三条 CSS：`.hc-msg-role` / `-ai` / `-user`）；
- 删行尾 `.hc-msg-tokens`（标记 + CSS）；
- `hc-msg-kind` 改用模块级 `ctxSourceTextOf(m)` / `ctxSourceTitleAttr(m)`（从 `renderContextOutlineList`
  里的局部 `srcOf` / `srcTitleAttr` **提出来**）⇒ 与目录 `hco-src` **同源同口径**：世界书 → 「世界书」、
  其余 → `来源 · identifier`，全称进 title。CSS 颜色也从次要色改正文主色（原来比目录浅一档，像两种东西）。
- `tub-entry-jump` 整套删除：`createEntryRowBuilder` 的 `jumpIcon` 参数、`jumpHtml` 拼接、
  `refreshWiMsgList` 里的绑定、三条 CSS 全删。`jumpToMergedFromWiMsg()` **函数保留**（附注释说明当前无 UI 调用点），
  因为它那条"挂起中先关子窗"的逻辑还与 `closeChildPopupAt` 的 resume 路径耦合。
- 实测头部 HTML：`<span class="hc-msg-idx">#1</span><span class="hc-msg-kind" title="世界书·前置 · worldInfoBefore">世界书</span>`；
  `roleChips=0` / `tokensEls=0` / `jumpIcons=0`。

#### 验证

- `node --check index.js` OK；CSS 括号 553/553 配平，且"普通块内嵌套 `{`"扫描通过（at-rule 放行）。
- 真实酒馆 1440×900 + 390×844 双档实测（详见各条"实测"）。
- ⚠️ 本轮踩坑：`isMobilePairLayout` 与 layout 各自重算档位导致判定不一致（见 ⑨-2）；
  `ctxMergeViewerSlot` 不随视口更新导致宽度溢出（见 ⑨-1）。两者都只有真实浏览器能量出来。

---

### 2.47 第三十六轮：功能启用/禁用开关 + 无快照提示 + 锚定栏再调整 + 灯色文案

**用户口径（原话，四条）**：
①「在『使用说明』页面的『UI主题』下面新增一个开关页面。提供『隐藏楼层/限制楼层/聊天统计』功能的启用/禁用开关。
   禁用功能后，对应的功能的标签页会隐藏。」
②「当对应『messageID』中没有存储完整上下文时。点击 `tub-stat-box tub-stat-box-clickable` 会提示通知
   『无该历史请求的完整上下文数据存储』。同时，`tub-open-outline` 图标按钮变为灰色。」
③「对 `tub-view-toggle` 区域进行修改。将 `tub-view-toggle` 移至右上角。`tub-view-toggle` 的标题改为『选择楼层』，
   同时 `tub-anchor-select` 的样式改为与 `tub-anchor-baseline-select` 样式一致；而 `tub-anchor-icons` 中的
   两个按钮图标移至 `tub-anchor-select` 右侧，并且把它们的背景样式去除（边框、颜色）。」
④「将里面所有的『常量』替换成『蓝灯』，『动态』替换成『绿灯』字符。」

追问确认两处：①的开关「当然是隐藏标签页 + 停掉后台功能。而且插件本身就提供有『禁用隐藏楼层/禁用限制楼层』的开关，
就差『聊天统计』了。」③的「选择楼层」= **第二行的标签**（替换原来的「锚定」二字），右上角仍是【明细】【对比】；
④「常驻」（同一个 constant 类型的另一个写法）也一并改成「蓝灯」。

#### ① 功能启用/禁用开关

**UI**（`createUI` 的 `instructions-panel`）：紧接 `.theme-switch-container` 之后插入
`#hh-feature-toggles`（`.hh-feature-toggle-container`：标题 + 三行「名称 / 说明 / 复选框」）。
三个复选框 id：`#hh-feature-toggle-hide` / `-limiter` / `-stats`。实测位置：UI主题 y=248 → 开关区 y=329。

**状态**（`DEFAULT_SETTINGS`）：新增 `stats_enabled: true` 与
`tab_visible: { hide: true, limiter: true, stats: true }`。
⚠️ **不新建并行状态**：前两项的判断**直接复用**既有功能开关（用户原话点明插件本来就有）——
「隐藏楼层」= `autoHideEnabled`、「限制楼层」= `limiter_isEnabled`。新建一套只会与功能页里那两个开关互相打脸。
`tab_visible` 仅存"标签页藏不藏"，写开关时与功能字段一起写。

**核心函数**（`index.js`，紧跟 `updateTabIndicator`）：
- `FEATURE_TABS`：key ↔ tab ↔ label 的**唯一**映射表；
- `isFeatureEnabled(key)`：唯一的状态读取口（hide/limiter 读老字段、stats 读新字段，缺省 true）；
- `applyFeatureToggles()`：导航按钮 + 面板 + 复选框回显。**显隐一律用 class `.hh-tab-off`**；
- `ensureActiveTabAvailable()`：当前页被禁 → 跳到第一个可用页（三个全关则落「使用说明」）；
- `setFeatureEnabled(key, enabled)`：写功能字段 + `tab_visible`，限制楼层还要同步 `chat_truncation`；
- `syncFeatureToggleUI(key, enabled, { keepTab })`：只同步表面状态，给功能页里**原有**开关反向用。

**⚠️ 踩坑（class 而非 `.hide()`）**：标签页的 display 是 CSS 里带 `!important` 的
（`.tab-panel { display:none!important }` / `.tab-panel.active { display:block!important }`），
jQuery 的 `.hide()/.show()` 写内联 display **根本压不住**，写了等于没写。
故 CSS 里新增 `#hide-helper-popup .tab-button.hh-tab-off, #hide-helper-popup .tab-panel.hh-tab-off { display: none !important; }`，
JS 一律 `toggleClass('hh-tab-off', !show)`。

**后台停用**（三处入口，全部走 `isFeatureEnabled`）：
- 聊天统计：`writeAnchoredStats()` 开头短路（不写锚定、不存快照）；
  `CHAT_COMPLETION_SETTINGS_READY` 监听器**最前面**短路（不跑统计、不排队），并顺手解掉
  `simulateInProgress`（否则"统计关着 + 点了模拟"会留一个清不掉的标志，后续真实发送被误当成模拟）；
  `triggerSimulate()` 前置拒绝（不弹输入框）。
- 隐藏楼层：`shouldProcessHiding()` 读 `autoHideEnabled`（既有），关掉即全面停。
- 限制楼层：`limiter_isEnabled` 关掉时把 `power_user.chat_truncation` 与 `#chat_truncation` 一起归 0（交还原生）。

**反向同步**：`#hide-auto-process-toggle` 与 `#limiter-enabled` 的 change 处理里各加一行
`syncFeatureToggleUI(...)`，`keepTab: true` —— 用户正站在这页上操作，不该因为关掉就被弹走。

**打开弹窗的顺序**（`#hide-helper-wand-button` click）：先 `applyFeatureToggles()`、再恢复 `last_active_tab`，
且 `last_active_tab` 若已被禁用则回退到第一个可用页。⚠️ 顺序不能反：先亮起来再隐藏会留下"看不见却 active"的鬼页。

实测：关「聊天统计」→ 导航按钮 `display:none` + 面板 `hh-tab-off` + `stats_enabled=false`；恢复 → 全部回来。
关「隐藏楼层」→ 按钮藏 + `autoHideEnabled=false` + 功能页里 `#hide-auto-process-toggle` 同步变未勾选。

#### ② 无快照的提示 + 置灰

新增两个纯函数：`currentAnchorDumpKey(item)`（键 = `${chatId}|${anchorId}`，与开窗处**完全同源**）、
`currentAnchorHasDump(item)`。`renderAnchorBar` 里算 `hasCtxDump`，据此给 `#tub-open-outline` 挂 `.is-disabled`
（`opacity .38` + `cursor: not-allowed` + `aria-disabled` + 提示文案换成「该楼层没有存储完整上下文数据」）。
点击时若无快照 → `notify('warning', '无该历史请求的完整上下文数据存储')`，**不打开窗口**。

**两处入口都要拦**（用户口径点名的是 `tub-stat-box tub-stat-box-clickable`，图标是"同时"那条）：
① 锚定栏图标 `#tub-open-outline`（`bindAnchorBarEvents` 里判）；
② **「总共」统计框** —— 统一放在 `toggleContextOutlineFromStats()` 这个唯一入口里判（统计框与图标最终都走它），
   一处判断覆盖两个入口。⚠️ 只在"打开"方向拦，关闭方向照常放行（已开的窗口要能收起）。
`openContextOutlinePopup()` 的直接调用方（`hideHelperDev.demo/openViewer/snapshot`、聊天统计子窗的
「查看完整上下文」按钮、`jumpToMergedFromWiMsg`）**不经过这个守卫** —— 它们所在的语境本来就已有快照，
多拦一层反而会让"刚生成的快照却打不开"。

**⚠️ 用 `.is-disabled` 而不是 `disabled` 属性**：`disabled` 的按钮在部分浏览器里不派发 click，提示就弹不出来。

实测（有/无快照两态）：无 → `tub-anchor-icon-btn is-disabled` / opacity 0.38 / toastr 收到
`无该历史请求的完整上下文数据存储` / 窗口未创建；有（给 anchor=9 补一份快照后刷新）→ `tub-anchor-icon-btn` / opacity 1。

#### ③ 锚定栏再调整

- `tub-view-toggle`（明细 / 对比）**移到右上角**：`.tub-anchor-head` 改 `justify-content: flex-end`
  + `margin-bottom: 2px`，head 里只留这一个元素。实测 `viewToggle.right = 892 = 行右缘`。
- 第二行改成 `选择楼层 [第 9 楼 ▾] [▤][💬]`：`.tub-anchor-row-label` 文案「锚定」→ **「选择楼层」**。
- `tub-anchor-select` 与 `tub-anchor-baseline-select` **合并成同一组 CSS 选择器**（透明底 + 1px 边框 +
  15px 药丸圆角 + 0.75em），删掉 baseline 原来那份独立规则 —— 两个控件以后要一起改。
  实测：`bg: rgba(0,0,0,0)` / `border: 0.8px` / `radius: 15px` / `font-size: 9.56px`（= 0.75em × 12.75 基准）。
- `tub-anchor-icons` 从 head **移进** `.tub-anchor-row`（紧跟选择框右侧）；按钮样式去掉边框与底色
  （`border: none` / `background: transparent`，24×24，hover 才显底色）。
  实测间距：标签→选择框 8px、选择框→图标 8px、图标之间 2px。

#### ④ 常量→蓝灯、动态→绿灯（含「常驻」）

`TYPE_FILTER_LABELS` 改为 `{ all: '所有', ejs: 'EJS', dynamic: '绿灯', constant: '蓝灯' }`，
其余**四处显示文案全部改为引用它**（避免再各写一份而漂移）：
- 类型过滤菜单（`typeFilterOptionsOf` 本来就引用它）；
- 饼图图例（`renderPieViewInto` 的「动态 / 常量」两行）；
- 对比视图总量表（`numRow('常量') / numRow('动态')` → `numRow(TYPE_FILTER_LABELS.constant) / …dynamic`）；
- 条目详情窗与条目来源子窗口头的类型徽标（原 `{ constant: '常驻', dynamic: '动态' }` 两处）。

⚠️ **只改显示文案**，内部 key（`constant` / `dynamic` / `ejs`）与 CSS 类名一律不动 ——
它们被统计口径、快照字段、离线夹具大量引用。

实测：图例 `含EJS: 26839 / 绿灯: 4267 / 蓝灯: 14523`；对比表行标签 `蓝灯 / 绿灯 / EJS`。

#### 验证

- `node --check index.js` OK；CSS 括号 558/558 配平 + "块内嵌套 `{`"扫描通过。
- 真实酒馆 1440×900 实测（各条见上）。
- ⚠️ 本轮环境干扰：同机会有其他插件把浏览器导航到 `localhost:3210`，多次打断量测 ——
  验证前先确认 `location.href` 是 `127.0.0.1:8000` 且 `#hide-helper-popup` 存在。

---

### 2.48 第三十七轮：删开关说明 / 禁用提示 / 标签栏下间距

**用户口径（原话，三条）**：
①「`hh-feature-toggle-desc` 说明文本给删了，`hh-feature-toggle-desc` 也去除，太多余了。」
②「当在"标签页"禁用"隐藏楼层/限制楼层"时，出现提示消息"隐藏楼层/限制楼层功能已禁用，可以在【使用说明】页面重新开启。"」
③「`popup-tabs-nav` 下面增加下间距，与内容区域隔开。」

#### ① 删除说明文本

三行 `.hh-feature-toggle-desc` 标记全删、CSS 规则也删（JS/CSS 双侧零残留，只剩注释提及）。
配合收紧了行距：容器 `gap: 8px → 6px`、行 `padding: 4px 0 → 2px 0`。
实测开关区高度 **165px**、三行文字只剩「隐藏楼层 / 限制楼层 / 聊天统计」、`descCount = 0`。

#### ② 禁用提示

新函数 `showFeatureDisabledHint(label)` → `toastr.warning(`${label}功能已禁用，可以在【使用说明】页面重新开启。`)`。

**为什么需要**：功能页里那两个页内开关（`#hide-auto-process-toggle` / `#limiter-enabled`）关掉后，
**该页的标签按钮会立刻消失** —— 用户站在原地，看不到"怎么再打开"，容易以为功能坏了。

**三处调用**：
- `#hide-auto-process-toggle` 的 change → `showFeatureDisabledHint('隐藏楼层')`；
- `onLimiterSettingsChange`（`#limiter-enabled`）→ `showFeatureDisabledHint('限制楼层')`，放在
  `updateCurrentHideSettingsDisplay()` 之后（先让页内"已禁用"占位文案就位再弹）；
- `setFeatureEnabled()`（使用说明页那三个开关）→ 按 `FEATURE_TABS` 取 label 弹。
  ⚠️ 用户只点名了前两个，但「聊天统计」同样会让标签消失，不提示会让人觉得"页面不见了"，故一并加上。

**不会重复弹**：页内开关走各自的 change 处理器，那里用的是 `syncFeatureToggleUI`（只同步表面状态，不写功能字段、
不调 `setFeatureEnabled`）；而 `setFeatureEnabled` 里回写页内开关用的是 `.prop('checked', …)`，**不派发 change**。
两条路径互斥。实测三个开关各拨一次 → 每次都只收到 **1 条**提示，文案与用户给的一字不差。

#### ③ 标签栏下间距（**真根因不是"忘了加"**）

实测发现 `nav.bottom === content.top === 130`，**间距从来就是 0**。根因：
`#hide-helper-popup` 的基础类是 `display: none; flex-direction: column; gap: 20px` ——
**没有写 `display: flex`**。打开时 jQuery 的 `.show()` 按元素默认值把它设成 `display: block`，
于是父容器的 `gap: 20px` 对这两个 block 兄弟**完全不生效**（gap 只对 flex/grid 容器起作用）。

**修法**：不动父容器（改 `display:flex` 会牵动整个弹窗排版，风险大），
给内容区加 `margin-top: 12px`（位置可控、不影响导航的下划线指示条 —— 那条是 `bottom: -1px` 绝对定位）；
导航再加 `padding-bottom: 10px`（让标签的点击/悬停区不贴内容）。
⚠️ 导航这条 `padding` 在文件末尾有两条媒体查询会**整条重写**，必须同步补上 `padding-bottom`，否则窄屏下又归 0。

实测：间距 **0 → 11px**（`contentMarginTop: 12px`，取整差）。

#### ⚠️ 本轮自踩的坑（必须记住）

改注释时在**模板字符串内的 HTML 注释里写了反引号** → 模板字符串提前截断，
后面的 `.hh-feature-toggle-desc` 被当成减法表达式求值 → `ReferenceError: feature is not defined`
→ `createPopup` 抛异常 → **整个插件弹窗都建不出来**（页面上连 `#hide-helper-popup` 都没有）。
`node --check` **抓不到**（语法是合法的），只有真实浏览器 + 控制台才暴露。

教训：模板字符串内的注释**一律不用反引号**，类名直接写不加反引号；改完扫一遍
「HTML 注释行是否含反引号」再刷新验证。

#### 验证

- `node --check index.js` OK；CSS 557/557 配平；「HTML 注释行含反引号」扫描 clean。
- 真实酒馆 1536×674 实测：`descCount 0` / 三行只剩名称 / 间距 0→11 / 三个开关各弹 1 条正确提示 /
  限制楼层关闭后 `chat_truncation` 归 0 / 关闭时标签隐藏、重新启用后四个标签全回来。

---

### 2.49 第三十八轮：饼图图例「含EJS」→「EJS」

**用户口径（原话）**：「把"含EJS"改为"EJS"。」

**改法**（`renderPieViewInto`，`index.js`）：图例第一行原本是**硬编码**的 `含EJS: ${e}`，
改成 `${TYPE_FILTER_LABELS.ejs}`，与同段里的「绿灯 / 蓝灯」两行**统一从同一张表取文案**：

```js
${e > 0 ? `<div class="tub-legend-item">…</div>`  // 原来是 含EJS: ${e}
```

顺带把两处 `{ ejs: 'EJS', … }` 的硬编码也改为引用 `TYPE_FILTER_LABELS.ejs`
（`buildEntryDetailHtml` 与 `setEntryPopupHead` 的类型徽标），
现在**整个界面上的 "EJS / 绿灯 / 蓝灯" 三个词只有 `TYPE_FILTER_LABELS` 一处来源**。

⚠️ 三处**注释**里的「含 EJS」（带空格、描述"统计口径含 EJS 渲染"）保持原样 —— 那是语义描述，不是界面文案。

#### 验证

- `node --check index.js` OK；全仓 `含EJS` 零残留（`Select-String` 无命中）。
- 真实酒馆实测：图例 `EJS: 26839 / 绿灯: 4267 / 蓝灯: 14523`，`document.body.innerHTML.includes('含EJS') === false`。
- 「条目来源」子窗的饼图用的是同一个 `renderPieViewInto`，口径自动同步（该演示快照无 EJS 条目，只渲染绿灯/蓝灯两行）。

---

### 2.50 第三十九轮：`tub-open-chat-stats` 换成统计图图标

**用户口径（原话）**：「把 `tub-open-chat-stats` 的图标换成统计图图标。这样更直观些。」

**改法**（`HC_ICON_CHAT`，`index.js`）：原来是个**对话气泡**（`M21 11.5a8.38…`，容易和"聊天"这个词的表面含义混淆，
看不出它其实打开的是**统计**子窗）。换成**三根高低柱 + 底部基线**的统计图造型，
与子窗里那两张「输出 tokens / 生成速度」柱状折线图的语义一致：

```html
<line x1="3" y1="21" x2="21" y2="21"></line>
<rect x="5"    y="12"  width="3.4" height="9"></rect>
<rect x="10.3" y="7"   width="3.4" height="14"></rect>
<rect x="15.6" y="3.5" width="3.4" height="17.5"></rect>
```

**⚠️ 自踩并修正**：第一版柱高写成 6 / 11 / 14.5，三根柱子的**底边都停在 y=18**，而基线画在 **y=21** ——
柱子**悬空 3px 没站在基线上**。15×15 的小尺寸下肉眼完全看不出，是把每个 `rect` 的
`y + height` 与基线的 `y1` 逐个比对才发现的。改成 `height = 21 − y`（9 / 14 / 17.5）后三根柱底边齐齐压在基线上。

#### 验证

- `node --check index.js` OK；`<rect>` × 3 + `<line>` × 1，旧气泡的 `path d="M21 11.5a8.38…"` 已不存在。
- DOM 几何校验：三根柱 `bottom` 均为 **21**（`allOnBaseline: true`）、高度三档各不相同（9 / 14 / 17.5）、
  彼此无重叠、全部落在 `viewBox 0..24` 内。
- `getBBox()` = `{x:3, y:3.5, w:18, h:17.5}`（完全在 viewBox 内）、渲染尺寸 15×15、
  `stroke` 继承 `currentColor`（暗色主题下为 `rgba(255,255,255,0.85)`，亮色下自动跟随主题）。
