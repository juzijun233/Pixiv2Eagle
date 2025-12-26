# Copilot / AI Agent 指南 — Pixiv2Eagle

下面是让 AI 编码代理快速在本仓库中高效工作的要点。保持简洁、可操作，引用仓库中可被发现的实现位置与示例。

## 项目概览（大局）
- **类型**：单文件 Tampermonkey 用户脚本（`Pixiv.js`）。
- **目的**：在 Pixiv 页面一键将作品保存到本地运行的 Eagle 应用（通过 Eagle 的本地 HTTP API）。
- **关键外部依赖**：Eagle 本地 API（`http://localhost:41595`），Pixiv 静态资源域 `i.pximg.net`。

## 关键文件与位置（快速跳转）
- **入口 / 核心逻辑**：`Pixiv.js`（整个脚本在 IIFE 中实现，约 1 个文件包含所有行为）。
- **项目说明**：`README.md`（安装与使用说明）。
- **工作流 / CI**：`.github/workflows/`（存在，但与本脚本开发无关）。

## 可发现的重要模式与约定（仅描述仓库真实存在的做法）
- **配置存储**：使用 Tampermonkey 的 `GM_getValue` / `GM_setValue`，常用键名示例：`pixivFolderId`, `useUploadDate`, `saveDescription`, `createSubFolder`, `debugMode`, `autoCheckSavedStatus`, `folderNameTemplate`, `eagleSavedBadgeRecords`。
- **用户交互**：通过 `GM_registerMenuCommand` 注册菜单（在 `Pixiv.js` 顶部 line ~130 可见多条调用）。
- **网络请求封装**：封装为 `gmFetch(url, options)`、`gmFetchBinary(...)`、`gmFetchText(...)` 三个函数（返回 Promise 并基于 `GM_xmlhttpRequest`）。修改网络逻辑时请优先复用这些函数。
- **Eagle 集成点**：所有与 Eagle 交互的 URL 都以 `http://localhost:41595` 开头（例如 `api/item/list`、`api/application/info`）；分页逻辑和返回值解析在 `isArtworkSavedInEagle` 中实现——注意返回结构可能为 `data.data` 或 `data.data.items` 两种形式。
- **Eagle API 参考**：
  - 获取项目详情：`https://api.eagle.cool/item/info` (用于获取更准确的 URL 等元数据)
  - 列表查询：`https://api.eagle.cool/item/list`

### 推荐作品区域（Recommendation Area）
- **容器标识**：`<section class="sc-79c00fd3-0 fbGJOF">` 或 `<div class="sc-bf8cea3f-0 dKbaFf">`。
- **列表结构**：采用 `ul` > `li` 结构，每个 `li` 包含一个作品的所有信息。
- **关键元素**：
  - **作品链接**：`<a class="sc-fab8f26d-6 hOiByN" href="/artworks/{pid}">`（需提取 PID）。
  - **画师链接**：`<a class="sc-fbe982d0-2 caPKbc" href="/users/{uid}">`（需提取 UID）。
  - **缩略图容器**：`<div radius="4" class="sc-f44a0b30-9 cvPXKv">`（首选标记插入点）。
  - **备选容器**：`<div class="sc-fab8f26d-3 etVILu">`（若首选容器不可用，可向上查找至此层级）。
- **检测策略（高性能版）**：
  - **全局索引**：启动时通过 `ensureEagleIndex` 异步构建全局索引（`Map<ArtistUID, Set<PID>>`），递归遍历画师文件夹下的所有子文件夹（包括类型/系列文件夹）以收集 PID。
  - **状态检查**：直接在内存索引中查找 `uid` 和 `pid`，避免对每个推荐作品发起网络请求。
  - **重试机制**：
    - 维护 `pendingLis` 队列，存储 DOM 未加载完成（缺链接/图片）或索引未就绪的条目。
    - 使用高频定时器（200ms）重试队列中的条目。
    - 仅在确认状态（已保存且标记成功、未保存、或画师不在 Eagle）后才设置 `dataset.eagleChecked`。
  - **触发机制**：`MutationObserver` 监听 DOM 变化 + 低频定时器（2s）兜底扫描。

### 文件夹结构与标识规则
本脚本在 Eagle 中建立的文件夹树结构遵循以下约定：
1. **画师文件夹**（`findArtistFolder`）：在 Pixiv 根文件夹下，description 中包含 `pid = {artistId}` 用于识别。
2. **类型文件夹**（可选）：在画师文件夹下，仅当开启“按类型保存”时创建。
   - 插画：名称“插画”，description = `illustrations`
   - 漫画：名称“漫画”，description = `manga`
   - 小说：名称“小说”，description = `novels`
3. **系列文件夹**（`getSeriesFolder`）：在画师文件夹或类型文件夹（如“漫画”）下，description = `https://www.pixiv.net/user/{uid}/series/{seriesId}`。
4. **作品子文件夹**（`createEagleFolder`）：在画师/类型/系列文件夹下，description = 作品 ID（artwork pid）。

### 已保存作品检测逻辑（`findSavedFolderForArtwork`）
- 获取作品详情（包含 `seriesNavData` 和 illustType）。
- 通过"加入追更列表"按钮（DOM selector: `div.sc-487e14c9-0.doUXUo`）判断是否为漫画系列。
- 若作品属于系列，优先在系列文件夹中检测；否则在画师文件夹中检测。
- **两级匹配**：
  1. 检查当前文件夹中的 items（`item.url` 与 Pixiv 作品 URL 对比）。
  2. 递归检查所有子文件夹的 description 是否等于作品 ID。

### 画师文件夹标题生成（`ArtistMatcher` 类）
- 实现模板到正则的双向转换（模板变量：`$uid`、`$name`）。
- `generate(uid, name)` 用于生成文件夹名称；`match(str, uid)` 用于从文件夹名识别画师 ID。
- 示例：模板 `$uid_$name` 与画师 ID 123、名称"作者名" 生成 "123_作者名"。

### 多页/子文件夹策略
- 配置项 `createSubFolder` 支持三种模式（字符串值）：
  - `off`：不创建子文件夹。
  - `multi-page`：仅当 `pageCount > 1` 时创建子文件夹。
  - `always`：始终创建子文件夹。
- 漫画作品（`illustType = 1`）始终创建子文件夹。

### 作品列表页面标注（`markSavedInArtistList`）
- 在用户画师页面（`/users/{id}/illustrations`、`/manga`）和系列页面（`/user/{id}/series/{seriesId}`）上执行。
- **执行策略**：
  - 使用 `debouncedMarkSavedInArtistList` (300ms) 防抖，避免频繁触发。
  - 维护全局 `currentGalleryObserver`，确保每次执行前断开旧的监听器。
- **搜索范围限制**：
  - 系列页面：严格限制在 `div.sc-de6bf819-3.cNVLSX` 容器内检测，若未找到该容器则停止检测，防止误匹配底部导航链接。
  - 其他页面：默认为 `document.body`。
- **画师插画/漫画页面结构 (Artist Illustrations/Manga Page Specs)**：
  - **列表容器**：`div.sc-bf8cea3f-0.dKbaFf` (ul > li 结构)。
  - **作品项**：`li` 元素。
  - **缩略图容器 (标记插入点)**：`div.sc-f44a0b30-9.cvPXKv` (通常带有 `radius="4"` 和 `style="overflow: visible;"`)。
  - **作品链接**：`div.sc-fab8f26d-1.kcKSxC` > `a.sc-fab8f26d-6.hOiByN` (href="/artworks/{pid}")。
  - **识别逻辑**：提取 href 中的 PID，与 Eagle 中已保存作品的备注/名称/文件夹描述进行匹配。
- **锚点匹配策略（`scoreAnchor`）**：
  - 优先选择包含 `<img>` 或 `<picture>` 的锚点（+10分），解决系列页面中“下一章”文字链接导致的错位问题。
  - 其次考虑容器层级（系列缩略图容器 > 外层容器 > 列表卡片）。
- **收集可比对的标识**：
  1. 收集画师文件夹下所有 items 的 URL。
  2. 递归收集所有子文件夹的 description。
  3. 若当前为系列页面，额外调用 `collectSeriesFolderItems()` 递归遍历系列文件夹的全部层级。
- **插入标记**：在作品卡片容器（优先匹配 `div.sc-4822cddd-0.eCgTWT` 或 `div.sc-f44a0b30-9.cvPXKv`）的左下角浮动插入 ✅ 标记，并记录匹配元数据到 `GM_setValue('eagleSavedBadgeRecords')`。

### 小说保存功能开发规范 (Novel Support Specs)
**注意：此功能正在开发中，请严格遵循以下 DOM 选择器与逻辑。**

#### 1. 小说系列 (Novel Series)
- **URL 模式**：`https://www.pixiv.net/novel/series/{seriesId}`
- **文件夹层级**：
  - 画师文件夹 -> (可选) "小说"文件夹 -> 系列文件夹
  - **系列文件夹命名**：使用系列标题。
  - **系列文件夹备注**：`https://www.pixiv.net/novel/series/{seriesId}` (识别依据)。
- **元数据提取**：
  - **标题**：`h1.sc-41178ccf-3.irrkHK`
  - **简介**：`div.sc-fcc502d1-0.jNYFaO > p.sc-fcc502d1-1.fDflWh` (保存为 `简介.txt`，备注=系列URL)。
  - **封面**：`img.sc-11435b73-2.hnPyQB` (保存为 `cover.jpg`)。
  - **作者 ID**：`h2.sc-b6a5d604-0.kepWbf a[data-gtm-user-id]`。
- **章节列表检测**：
  - **容器**：`div.sc-794d489b-0.buoliH` (ul > li 结构)。
  - **章节 ID**：`a[data-gtm-value="{novelId}"]`。
  - **UI 标记**：在 `div.sc-3a91e6c3-6.eJoreT` 内，`button.sc-5d3311e8-0.iGxyRb` 之前插入。

#### 2. 小说章节 (Novel Chapter)
- **URL 模式**：`https://www.pixiv.net/novel/show.php?id={novelId}`
- **文件夹层级**：
  - 画师文件夹 -> (可选) "小说"文件夹 -> [系列文件夹] -> 章节文件夹
  - **章节文件夹命名**：使用章节标题。
  - **章节文件夹备注**：`{novelId}` (识别依据)。
- **系列归属检测**：
  - 检查 `section.sc-55920ee2-1...` 是否存在。
  - 提取系列链接：`a.sc-13d2e2cd-0.gwOqfd[href^="/novel/series/"]`。
- **元数据提取**：
  - **标题**：`h1.sc-41178ccf-3.irrkHK`
  - **简介**：`div.sc-fcc502d1-0.jNYFaO > p.sc-fcc502d1-1.YOSSS` (保存为 `简介.txt`，备注=章节ID)。
  - **封面**：`img.sc-41178ccf-19.cKuUeg` (保存为 `cover.jpg`)。
  - **作者 ID**：`h2.sc-b6a5d604-0.kepWbf a[data-gtm-value]`。
- **正文内容提取**：
  - **容器**：`div.sc-ejfMa-d.eXXQXn`。
  - **逻辑**：忽略开头空白 div，提取后续 `<p>` 标签文本，换行符拼接。
  - **保存**：`{Title}.txt` (备注=章节ID)。
- **保存按钮**：
  - **位置**：在 `<section class="sc-44936c9d-0 bmSdAW">` 末尾添加。
  - **功能**：与插画按钮一致（未保存显示“保存到 Eagle”，已保存显示“已保存”并提供打开功能）。
- **重要提示**：
  - 调用 Eagle API `addFromURLs` 时，`name` 字段必须包含正确的文件扩展名（如 `.txt`, `.jpg`），否则 Eagle 可能无法识别文件格式。

## 变更与修改守则（对 AI 的具体要求）
- **不要移除或破坏 userscript metadata block**（文件顶部 `// ==UserScript==` 到 `// ==/UserScript==`）——保持 `@grant` 和 `@connect` 包含所需域。新增外部访问时，务必在 metadata 中添加对应 `@connect`。
- **网络域更新**：如果新增访问的第三方域（例如 CDN 或第三方 API），请同时更新 `@connect` 列表并在 `gmFetch` 中处理异常。
- **保留 MIT 头部声明**：脚本顶部包含的 License 通常应保留在修改中。
- **小而明确的变更**：优先对 `Pixiv.js` 内的单一函数或类做原子改动（例如改进 `ArtistMatcher.createRegex`、提取常量），保持 IIFE 结构不变，避免拆分成多个模块，除非向作者/维护者确认。
- **考虑按类型保存**：在实现查找、保存或更新功能时，务必检查 `getSaveByType()` 设置，并考虑到作品可能位于“插画”、“漫画” or “小说”等子文件夹中，而不仅仅是画师根目录。

## 本地调试 / 验证步骤（可复制给人工测试者）
1. 启动 Eagle 应用（确保能访问 `http://localhost:41595/api/application/info`）。
2. 在浏览器中安装 Tampermonkey，将修改后的 `Pixiv.js` 以用户脚本方式安装或临时粘贴运行。
3. 打开任意 Pixiv 作品页面（`/artworks/xxxxx`），在 Tampermonkey 菜单中通过 `📁 设置 Pixiv 文件夹 ID` 粘贴 Eagle 文件夹链接或 ID。
4. 使用页面上的 “保存到 Eagle” 按钮验证流程；如遇错误，打开浏览器控制台查看由脚本输出的 `console.error` 日志。

## 示例：常见微改动（给 AI 的具体补丁示例）
- 增加新的 GM 配置项：
  - 在顶部配置区增加 `GM_getValue()/GM_setValue()` 使用示例，并通过 `GM_registerMenuCommand` 暴露配置入口。
- 改进网络错误处理：
  - 在 `gmFetch` 的 `onerror` 中，加入更详细的 `console.error` 日志，并在返回值上统一失败结构（例如 `{ ok: false, error }`），但调整时在仓库现有调用点统一适配返回格式。

## 何处查找示例实现
- `Pixiv.js` 中的关键函数及行号：
  - `ArtistMatcher` 类（line ~113）：模板生成/匹配逻辑。
  - `gmFetch` / `gmFetchBinary` / `gmFetchText`（line ~383-430）：网络请求封装。
  - `isArtworkSavedInEagle`（line ~538）：Eagle 分页查询与 item.url 对比。
  - `findArtistFolder`（line ~456）：递归查找画师文件夹（通过 pid = {artistId} 识别）。
  - `findSavedFolderForArtwork`（line ~613）：综合检测已保存作品，包含系列页面判断与递归子文件夹搜索。
  - `markSavedInRecommendationArea`（line ~2300）：推荐区域检测，包含 `ensureEagleIndex` 全局索引构建与 `pendingLis` 重试队列逻辑。
  - `markSavedInArtistList`（line ~1429）：标注作品列表页面已保存作品，含系列页面的 `collectSeriesFolderItems` 递归遍历。
  - `updateSaveButtonIfSaved`（line ~1380）：检测作品详情页保存状态并更新按钮。
  - 顶部菜单命令（line ~145-152）：如何注册配置菜单。

## 常见修改模式
### 添加新的 GM 配置项
1. 在顶部添加 getter/setter 函数（参考 `getDebugMode()` / `toggleDebugMode()`）。
2. 通过 `GM_registerMenuCommand` 注册菜单项以暴露配置入口。
3. 在业务逻辑中通过 getter 读取配置，setter 持久化。

### 扩展 Eagle API 调用
1. 所有 HTTP 请求使用 `gmFetch` 包装，确保异常处理。
2. 新增 Eagle API endpoint 时，检查返回值结构（是否为 `data.data` 或 `data.data.items`）。
3. 在 Tampermonkey metadata 中添加对应 `@connect` 声明。

### 修改文件夹结构或匹配规则
- 涉及 description 字段时，同步更新 `findArtistFolder`、`getSeriesFolder`、`findSeriesFolderInArtist` 等函数。
- 若改变子文件夹 description 含义，需同步修改 `findInSubfolders` 递归逻辑（目前假设 description = artworkId）。

### 漫画系列章节更新功能 (Update Series Chapter Numbers)
- **功能入口**：在漫画系列页面（`/user/{id}/series/{seriesId}`）的“从第一话开始阅读”按钮旁，注入“更新系列章节序号”按钮。
- **核心逻辑**（`updateSeriesChapters`）：
  1. **文件夹定位**：在画师文件夹（或其下的“漫画”子文件夹）中查找 description 匹配系列 URL 的文件夹。
  2. **章节匹配**：遍历页面上的章节列表，提取 PID 和标题。
  3. **文件夹重命名**：在系列文件夹中查找 description 匹配 PID 的子文件夹，将其重命名为 `#序号 标题` 格式。
  4. **文件重命名**：遍历章节文件夹内的所有文件，将其重命名为 `#序号 标题`（保留 `_p0` 等页码后缀）。
- **注意事项**：
  - 必须处理“按类型保存”开启时的文件夹层级（画师 -> 漫画 -> 系列）。
  - 标题提取需移除 `.eagle-saved-badge` 和 `R-18` 等干扰标签。
  - 文件重命名需保留多页作品的后缀（如 `_0`, `_1` 或 `_p0`）。

## 风险与注意点
- **Eagle API 偶发不可达**：所有 `http://localhost:41595` 调用必须容错（参考 `checkEagle()` 的处理思路）。
- **脚本沙箱限制**：脚本运行在浏览器沙箱中，避免同步阻塞操作；所有 Eagle 交互必须异步化。
- **Pixiv 页面结构变化**：DOM selector（如 `div.sc-4822cddd-0.eCgTWT`、`div.sc-487e14c9-0.doUXUo`）依赖 Pixiv 前端版本，更新时可能失效。
- **大规模数据集性能**：当画师作品数>1000 或系列章节数>500 时，`getAllEagleItemsInFolder` 的分页遍历可能导致明显延迟；可考虑缓存或增量更新。
- **系列页面的递归遍历**：`collectSeriesFolderItems` 递归收集所有层级的 items 和 descriptions，深层结构过复杂时可能堆栈溢出（当前实现基于 async/await 应不会触发，但需注意）。

## 依赖库与外部资源
- **Pixiv API**：通过浏览器 fetch 调用 `https://www.pixiv.net/ajax/illust/{id}` 获取作品元数据。
- **fflate / gif.js**：动图（ugoira）转 GIF 时从 CDN（jsdelivr）动态加载，参考 `convertUgoiraToGifBlob` 函数。
- **Tampermonkey GM_* API**：`GM_xmlhttpRequest`、`GM_getValue`、`GM_setValue`、`GM_registerMenuCommand`。

---

**欢迎提问**：如需对特定区域进行深入讨论（例如系列页面的文件夹结构设计、性能优化策略），请指出具体章节。
