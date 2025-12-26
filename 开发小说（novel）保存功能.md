接下来，我们将开发之前尚未完成的小说（novel）保存功能。

请将下面的内容归纳整理后记录到 .github\copilot-instructions.md 文件中，供后续开发参考：

# 小说系列的说明：
- 小说系列的URL为：`https://www.pixiv.net/novel/series/xxxxxx`其中xxxxxx为小说系列的ID。小说系列将会保存在画师文件夹下，如果开启了“按类型保存”，则在画师文件夹下的“小说”文件夹下创建（如果“小说”文件夹不存在，则建立它，注意检查之前是否已有相关逻辑），否则则在画师文件夹下创建。创建的文件夹名称为小说系列的标题，备注为小说系列的URL，使用备注中的URL作为识别依据。

- 小说系列的标题位于：`<h1 class="sc-41178ccf-3 irrkHK">xxxxxx</h1>`，将标题提取作为文件夹名称。

- 小说系列的简介位于：`<div class="sc-fcc502d1-0 jNYFaO">`中的`<p id="expandable-paragraph-_r_m_" class="sc-fcc502d1-1 fDflWh">`标签，请提取其中的文本内容，作为小说的简介，保存于“小说系列文件夹”中，并将文件命名为“简介”，格式为txt，编码为utf-8，文件备注为小说系列的URL，用于识别简介是否已被正确保存。

- 小说系列的封面位于：`<img class="sc-11435b73-2 hnPyQB" src="xxxxxx.jpg">`，请将封面保存于“小说系列文件夹”中，并将文件名命名为“cover”，文件格式不变（默认应为jpg)。

- 小说系列中的章节位于：`<div class="sc-794d489b-0 buoliH">`中，采用ul、li的表格形式。其中，每一个li标签都代表一个章节。章节的URL位于`<a class="sc-fab8f26d-7 sc-6d7a6a5e-14 cUhcIP jToerZ" data-gtm-value="xxxxx" href="/novel/show.php?id=xxxxx">xxxxx</a>`中，其中`data-gtm-value`参数为小说ID。在系列页面中检测章节保存状态时，使用`data-gtm-value`作为小说ID，在eagle的备注中搜索具有此ID的文件夹（注意文件夹必须位于画师文件夹-小说（根据按类型保存功能开关决定）-系列文件夹（series ID）-章节文件夹（novel ID）中，避免ID出现与插画PID重复的可能）。如果章节已保存，则在`<div class="sc-3a91e6c3-6 eJoreT">`中创建已保存标记，标记在`<button type="button" class="sc-5d3311e8-0 iGxyRb">`按钮之前。

- 系列作者：小说作者即为画师，信息位于`<h2 class="sc-b6a5d604-0 kepWbf">`中的`<a class="inline-flex items-center gap-[2px]" data-gtm-user-id="xxxxx" href="/users/xxxxx/request">`中，其中`data-gtm-user-id`参数为画师ID。

# 小说章节的说明：
- 小说章节的URL为：`https://www.pixiv.net/novel/show.php?id=xxxxx`其中xxxxx为小说ID。，保存时使用小说章节的标题作为文件夹名称，备注为小说ID，使用备注中的ID作为识别依据。小说章节（作品）将会保存在画师文件夹下，如果开启了“按类型保存”，则在画师文件夹下的“小说”文件夹下创建（如果“小说”文件夹不存在，则建立它，注意检查之前是否已有相关逻辑），否则则在画师文件夹下创建。

- 检查小说是否是系列的一部分：请检查小说作品页面中是否存在`<section class="sc-55920ee2-1 sc-55920ee2-3 sc-a307effc-0 bNBzcc jKXiZT cmCMBu">`，如果存在，说明小说是系列的一部分，请提取其中的`<a class="sc-13d2e2cd-0 gwOqfd" href="/novel/series/xxxxxx">`作为系列的链接，并改为通过系列文件夹来搜索、保存章节内容。

- 小说章节的标题位于：`<h1 class="sc-41178ccf-3 irrkHK">xxxxxx</h1>`，将标题提取作为文件夹名称。

- 小说简介位于：`<div class="sc-fcc502d1-0 jNYFaO">`中的`<p id="expandable-paragraph-_r_2_" class="sc-fcc502d1-1 YOSSS">`标签，请提取其中的文本内容，作为小说的简介，保存于“小说章节文件夹”中，并将文件命名为“简介”，格式为txt，编码为utf-8，文件备注为小说章节的ID，用于识别简介是否已被正确保存。

- 小说章节的封面位于：`<img alt="xxxxxx" class="sc-41178ccf-19 cKuUeg" src="xxxxxx.jpg">`，请将封面保存于“小说章节文件夹”中，并将文件名命名为“cover”，文件格式不变（默认应为jpg)。

- 小说作者：小说作者即为画师，信息位于`<h2 class="sc-b6a5d604-0 kepWbf">中的<a data-gtm-value="xxxxxx" href="/users/xxxxxx">`中，其中`data-gtm-value`参数为画师ID。

- 小说内容位于：`<div class="sc-ejfMa-d eXXQXn">`中，开头会有一些空白的div，请忽略这些空白div，提取后续的<p>标签中的文本内容，按顺序组合，每一个组p标签后使用换行符，保存格式为txt，编码为utf-8，文件命名为“小说章节的标题”，文件备注为小说章节的ID，用于识别内容是否已被正确保存。

- 小说的保存标记：在<section class="sc-44936c9d-0 bmSdAW">末尾添加保存按钮，其功能、效果与插画中的按钮相似（未保存时显示为保存，已保存后显示为打开）