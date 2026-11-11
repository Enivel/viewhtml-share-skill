---
name: viewhtml-share
description: 将本地单文件 HTML 发布为 ViewHTML 的预览链接，并管理已发布的页面：更新版本、上传图片和文件、设置公开/私有和密码、选择自己的页面地址、查看和恢复历史版本、收集表单数据。适用于用户要分享网页、原型或 HTML 成果，而不是部署完整网站。
---

# ViewHTML 网页分享

把用户指定的网页发布为预览页 `https://viewhtml.app/share/{id}`。分享页以预览容器展示渲染结果。

## 先判断身份，再决定能做什么

| 身份 | 怎么识别 | 能做什么 |
| --- | --- | --- |
| **匿名** | 没有 `vh_…` 凭证（默认） | 发布页面，链接 **3 天**后失效并删除，不能更新、不能管理 |
| **免费账号** | 提示词里带 `Authorization: Bearer vh_…`，`/api/me` 的 `plan` 是 `Free` | 页面永久保留（最多 3 个）、更新、上传文件、设为私有、删除 |
| **Pro 账号** | `/api/me` 的 `plan` 是 `Pro` | 免费账号的全部，加：页面不限数量、设置密码、自选页面地址、10 个历史版本和恢复、表单与数据、PNG/PDF 导出、去掉底栏 |

**不要猜，调 `GET /api/me`。** 带凭证时先调一次，读 `capabilities`：每项能力是 `true` 或 `false`，`upgrade.url` 是升级地址。要做的事对应的能力是 `false` 时，**不要硬试**：告诉用户「这是 Pro 功能」，给出 `upgrade.url`，问他要不要升级。

带了凭证就一律用下面的 `POST /api/pages` 发布，不要用匿名那节的 `/api/jobs`（那是匿名通道，只用于没有凭证时）。

没有凭证、但用户想要长期保留、更新、设密码等账号才有的能力时：告诉用户先在 viewhtml.app 登录，在控制台点「Create new page」里的「Copy prompt」，把得到的提示词交给你（里面带着凭证），再继续。

## 适用范围

- 适用于一个可独立打开的 `.html` 文件，或用户直接提供的完整 HTML。
- 不适用于需要服务器、登录态、私有 API、构建时路由或多个本地资源才能正常工作的应用；这类需求应使用项目的实际部署方式。
- 不要把含凭据、个人信息、内部链接、未公开业务数据或版权受限素材的内容提交到该公开服务。

## 发布前

1. 确认要分享的确切 HTML。若用户只指向一个项目目录，先找出其可独立打开的产物；没有明确产物时询问，不要猜测上传哪个文件。
2. 检查文件不超过 2 MB，并审阅其中是否含敏感信息。页面中的图片、样式和脚本应内联，或使用公开可访问的绝对 HTTPS URL，或（有账号时）先上传成页面文件；本地相对资源不会随 HTML 一起上传。
3. 如果用户没有明确要求公开分享，或内容的公开性不清楚，先说明链接是公开的（匿名发布 3 天后失效）并征求确认。

## 匿名发布

```bash
curl --fail-with-body --silent --show-error \
  -X POST https://viewhtml.app/api/jobs \
  -H 'Content-Type: application/json' \
  --data-binary "$(jq -n --rawfile html /absolute/path/page.html '{type: "url", html: $html}')"
```

先用 `jq` 检查响应的 `status` 为 `succeeded`、`result.kind` 为 `url`，再读取 `result.url`，这就是要交付的链接。不要自行拼接 URL 或依赖内部子域名。成功响应的 `result.expiresAt` 是失效时间。

同一份 HTML 在有效期内会复用同一个分享链接。每个来源 IP 有每日额度，收到 `429` 时停止重试并告知用户重置时间。

匿名发布后**必须告诉用户**：这个链接 3 天后失效，想长期保留、更新、设密码等需要在 viewhtml.app 登录（免费注册）。

## 账号操作（带 `vh_…` 凭证）

所有请求都带 `Authorization: Bearer vh_…`。基址以提示词里的 `API:` 为准（线上是 `https://viewhtml.app`，本地或预览环境不同），下文的 `/api/…` 都接在它后面，不要自己写死域名。凭证是用户的私有凭证：不要写进发布的页面、代码仓库或日志。

### 查看自己的套餐和能力

`GET /api/me`，响应里：

- `plan` / `planId`：`Free` 或 `Pro`
- `pages`：已用 / 上限（`limit` 为 `null` 表示不限）
- `storage`、`maxFileBytes`、`versionsPerPage`
- `capabilities`：`publish`（还能不能发新页面）、`update`、`visibility`、`password`、`ownAddress`、`versionHistory`、`files`、`data`、`removeBranding`
- `upgrade.url`、`upgrade.pro`：升级地址，以及当前缺少的能力名

### 发布、更新、查看

- 新页面：`POST /api/pages`，请求体 `{"html": "…"}`，响应里的 `shareUrl` 就是要交付的链接。`pageUrl` 是页面自己的地址（形如 `名字.viewhtml.app`，用户改过自选地址后以它为准）；用户没特别要求时交付 `shareUrl`，用户问「我的页面地址」或刚改过地址时给 `pageUrl`。
- 更新页面：`POST /api/pages/{id}/versions`，请求体 `{"html": "…"}`，链接不变。内容和当前版本完全相同时不会新增版本（`changed: false`）。
- 页面列表：`GET /api/pages`。页面详情和版本列表：`GET /api/pages/{id}`。
- 删除页面：`DELETE /api/pages/{id}`，链接立即失效，所有版本一并删除，**不可恢复**，要先向用户确认。

页面数已满（免费账号 3 个）时发布新页面会返回 `402`，要么让用户删除一个旧页面，要么推荐升级 Pro（不限数量）。

### 上传图片、音频、视频和文件

`PUT /api/pages/{id}/assets/{文件名}`，请求体就是文件内容（不是 multipart），必须带 `Content-Length`。响应给出文件的绝对地址，写进 HTML 后再发布新版本。

```bash
curl --fail-with-body -X PUT "https://viewhtml.app/api/pages/$ID/assets/logo.png" \
  -H "Authorization: Bearer $VH_KEY" --data-binary @logo.png
```

- 允许的类型：图片（png、jpg、gif、webp、avif、svg、ico）、音频（mp3、wav、ogg、m4a、aac、flac）、视频（mp4、webm、mov）、字体（woff、woff2、ttf、otf）、pdf、json、csv、txt、md、vtt、srt、glb、gltf、docx、xlsx、pptx。其他类型（html、js、exe、zip 等）不允许。
- 免费账号单文件最大 2 MB、总空间 10 MB；Pro 单文件 50 MB、总空间 1 GB。超出会报错，推荐升级 Pro。
- 文件名里的中文等字符会被整理掉，扩展名保留，以响应里的 `url` 为准。
- 列表：`GET /api/pages/{id}/assets`。删除：`DELETE /api/pages/{id}/assets/{文件名}`。

### 公开、私有和密码

页面有三种访问方式，`access` 字段表示当前状态：

- `public`：任何人有链接都能看。
- `private`：只有用户自己能看，别人打开是 404。
- `password`：页面在线，但知道密码的人才能打开（用户自己也要输入密码）。

用 `PATCH /api/pages/{id}` 修改，请求体可以同时带下面的字段：

- `{"access": "private"}` 或 `{"access": "public"}`。转公开会清掉密码；公开页面不能同时有密码。
- `{"password": "至少 4 个字符"}`：设置密码（**Pro**）。页面从此是 `password` 状态。免费账号会返回 `402`，推荐升级 Pro。
- `{"password": null}`：取消密码，页面回到只有自己能看。
- `{"title": "新标题"}`：改后台显示的名字，不改页面内容。

不要把密码写进日志或回复；设置后告诉用户密码由他自己保管。

### 选择自己的页面地址（Pro）

默认地址是系统生成的（形如 `mango-kilobyte-5726.viewhtml.app`）。Pro 可以取自己的名字，比如 `my-launch.viewhtml.app`：

1. 先检查能不能用：`GET /api/pages/{id}/address/check?name=my-launch`，返回 `available` 和不可用的原因（被占用、保留词、少于 3 个字符等）。
2. 可用再改：`PUT /api/pages/{id}/address`，请求体 `{"subdomain": "my-launch"}`。

**改地址会让旧地址立刻失效**（分享链接 `/share/…` 不变），改之前向用户确认。文件超过 50 个的页面不能改。免费账号返回 `402`，推荐升级 Pro。

### 历史版本

每次发布都会保留上一个版本，数量按套餐：免费账号只保留当前版本，Pro 保留最近 10 个。

- `GET /api/pages/{id}` 的 `versions` 列出所有版本。
- 恢复：`POST /api/pages/{id}/versions/{版本号}/restore`，会把旧内容作为新的当前版本，历史不丢。
- `capabilities.versionHistory` 是 `false`（免费账号）时没有旧版本可恢复，用户想要「能回退」时推荐升级 Pro。

### 表单和数据（Pro）

让静态页面能收集表单、显示可随时修改的内容、计数。所有接口都以 `/api/pages/{id}/data` 开头，免费账号一律返回 `402`，推荐升级 Pro。

**页面里怎么用**（发布后的页面会自动带上 SDK `window.viewhtml`，不用自己引入）：

- 零代码表单：`<form data-viewhtml-collection="signups">`，访客提交就存进名为 `signups` 的表，页面会显示成功或失败提示。
- 脚本写入：`viewhtml.collection("signups").add({...})`。
- 可改的内容：`viewhtml.kv.get("price")` 读一项（没有返回 `null`），`viewhtml.kv.all()` 读全部。内容由你通过下面的接口修改，**改了不用重新发布**。
- 计数器：`viewhtml.counter("likes").incr()` 加一，`.get()` 读取。
- 所有方法返回 Promise，失败只会 reject，不会弄坏页面。

**你（Agent）管理数据的接口：**

| 做什么 | 接口 |
| --- | --- |
| 总览（所有键值、集合、用量） | `GET /data` |
| 创建集合 | `POST /data/collections`，`{"name": "signups", "kind": "form"}`（`kind` 是 `form` 或 `counter`；名字只能是小写字母、数字、`-`、`_`，**要先创建，页面才能写入**） |
| 删除集合（连同记录） | `DELETE /data/collections/{name}`，不可恢复，先问用户 |
| 设置可改的内容 | `PUT /data/kv/{key}`，`{"value": …}`（单个值不超过 4 KB） |
| 删除一项内容 | `DELETE /data/kv/{key}` |
| 读记录 | `GET /data/collections/{name}/records?limit=50&before=时间`（按时间倒序，`limit` 最多 200；用上一页最后一条的 `createdAt` 作 `before` 翻页） |
| 导出 CSV | `GET /data/collections/{name}/export.csv` |
| 新增记录（只限 `form` 表） | `POST /data/collections/{name}/records`，`{"data": {…}}` |
| 修改记录 | `PATCH /data/records/{recordId}`，`{"data": {…}}` |
| 删除记录 | `DELETE /data/records/{recordId}`；批量 `POST /data/records/delete`，`{"ids": [...]}`（一次最多 100 个） |

Pro 的上限：每页最多 10,000 条记录，数据总量 50 MB，每天最多写入 10,000 次。超出会报错，不要循环重试。

### PNG / PDF 导出（Pro）

`POST /api/jobs`，带上 `Authorization: Bearer vh_…`，请求体 `{"type": "image" 或 "pdf", "html": "…"}` 创建导出任务（返回 `202`），用 `GET /api/jobs/{id}` 轮询到 `status` 为 `succeeded`，再从 `/api/jobs/{id}/result` 下载文件。Pro 每天最多 100 次，超出返回 `429`；免费账号和匿名返回 `402`。凭证无效返回 `401`，不会退回匿名。

## 遇到 402 怎么办

`402` 表示当前套餐没有这个能力。响应是 `{"error": "…", "code": "PAYMENT_REQUIRED", "details": {"upgradeUrl": "…"}}`。

1. 不要重试，也不要换个办法绕过。
2. 用一两句话告诉用户缺的是什么（比如「设置密码是 Pro 功能」），把 `details.upgradeUrl` 给他，说明升级后能得到什么。
3. 问用户要不要升级；他不升级，就说明现有套餐下能做到的替代方案（比如「页面设为私有，只有你自己能看」）。

只推荐用户确实需要的那一项能力，不要一次列出所有 Pro 功能。

## 发布后验证与交付

对交付的链接发起一次 `GET`，确认返回 `200`；不要把来源 HTML、内部 iframe 地址或任何敏感内容放进最终回复。向用户交付链接，并简要说明它来自哪个本地文件。

若请求失败：

- `400`：修正 HTML 为空、超过 2 MB 或请求结构不对的问题。
- `401`：凭证无效或已被撤销，请用户重新复制提示词。
- `402`：见上一节。
- `404`：页面不存在，或不属于这个账号。
- `429`：等待响应中的重置时间，不能通过重试绕过额度。
- 网络或 5xx：只重试一次；仍失败则报告错误，不要假称已发布。
