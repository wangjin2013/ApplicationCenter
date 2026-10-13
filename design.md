# 设计文档：iOS App 介绍门户（GitHub Pages 服务端）

> 需求来源：[requirements.md](requirements.md)。本仓库作为"服务端"，托管在 GitHub Pages 上，同时服务两类消费者：**网页访客（人）** 与 **iOS App（程序）**。

## 1. 目标与解读

**目标**：维护一个介绍"我的所有 iOS App"的门户网站 + 数据接口。内容更新只需改仓库文件，iOS App 内的推荐信息即获得最新数据，无需发版。

**解读确认**（显式列出，如有偏差请指出）：

- iOS App 是**原生解析 JSON** 渲染推荐列表/详情，而非内嵌 WebView。依据：需求中"获得 App 列表 → 点击查看详情 → 跳转 App Store"是原生交互描述。
- "服务端" = GitHub Pages 托管的纯静态文件，无需服务器程序、无数据库、无构建流程。
- 网页访客与 iOS App **共用同一份数据文件**。

## 2. 总体方案

核心原则：**单一数据源**。

- 所有 App 介绍数据放在 `data/apps.json` 一个文件里；
- 网页 `index.html` 用原生 JS fetch 该 JSON，渲染列表与详情；
- iOS App 用 URLSession 请求同一 JSON；
- **新增/修改 App = 编辑 JSON + 放图片 + push**，网页与 iOS 同时生效。

```
编辑 data/apps.json / 添加图片
        │ git push
        ▼
GitHub Pages 自动部署（约 1 分钟）
        ├──> 网页访客 ── index.html ──────┐
        └──> iOS App ── GET apps.json ────┤  同一份数据
```

### 方案取舍

| 备选方案 | 不采用的原因 |
|---|---|
| 每个 App 手写一个静态 HTML 详情页 | 违背"只改一处"的目标：新增 App 要同时改列表页 + 新建详情页 |
| Jekyll / 其他静态站点生成器 | 引入构建依赖与模板复杂度；个位数 App 的数据规模用不上；反而需要 `.nojekyll` 规避 Pages 的默认 Jekyll 处理 |
| 自建服务器 / 云函数 | 需求本质是静态内容分发；GitHub Pages 免费、零运维、自带 HTTPS 与全球 CDN，足够 |

## 3. 仓库结构

```
（仓库根目录）
├── README.md                  # 维护者指南（如何新增/修改一个 App）
├── design.md                  # 本设计文档
├── requirements.md            # 需求记录（已有）
├── .nojekyll                  # 禁用 GitHub Pages 默认的 Jekyll 处理，原样输出静态文件
├── index.html                 # 门户页：列表 + 详情（hash 路由），读取 apps.json 渲染
├── assets/
│   ├── style.css
│   ├── icons/<id>.png         # 1024×1024 PNG，直接复用 App Store 图标
│   └── screenshots/<id>-1.png # 直接复用 App Store 截图
└── data/
    └── apps.json              # ★ 唯一数据源（网页与 iOS 共用）
```

## 4. 数据结构：apps.json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | int | 是 | 数据格式版本号，当前 `1` |
| `defaultLocale` | string | 是 | 默认语言 `"en"`；某字段缺当前语言时回退到它 |
| `updatedAt` | string | 否 | 最后更新日期 `YYYY-MM-DD`，便于排查"数据是否最新" |
| `apps` | array | 是 | App 列表；**数组顺序即展示顺序**（不设排序字段） |
| `apps[].id` | string | 是 | 稳定标识符（小写英文，如 `filingo`），用于详情路由；发布后不要更改 |
| `apps[].name` | string | 是 | App 名称（各语言通用，不翻译，故为普通字符串） |
| `apps[].tagline` | 本地化文本 | 是 | 一句话简介，列表卡片展示 |
| `apps[].icon` | string | 是 | 图标**完整 URL** |
| `apps[].category` | 本地化文本 | 否 | 分类标签 |
| `apps[].description` | 本地化文本 | 是 | 详细介绍；段落间用 `\n\n` 分隔 |
| `apps[].features` | 本地化数组 | 否 | 特性要点列表，如 `{"en": [...], "zh": [...]}` |
| `apps[].screenshots` | [string] | 否 | 截图完整 URL 数组（各语言共用） |
| `apps[].appStoreUrl` | string | 否 | App Store 页面链接；留空则网页自动隐藏下载按钮 |

设计说明：

- **面向用户的文案字段是「本地化文本」**：值为对象 `{"en": …, "zh": …}`，客户端按当前语言取值、缺该语言时回退 `defaultLocale`（`en`）。英语为主，故 `en` 为准。`name` 与 `icon` 各语言相同，保持普通字符串，不做无谓包装。
- **图片 URL 用完整绝对地址**（如 `https://wangjin2013.github.io/ApplicationCenter/assets/icons/filingo.png`）。原因：iOS App 拿到 JSON 后无法解析相对路径；绝对 URL 让任何消费者零逻辑直接使用。代价：若换域名/仓库名需批量替换 JSON 内 URL——可接受，发生概率低。
- **description 用纯文本**（`\n\n` 分段），不用 Markdown/HTML：iOS 端 UILabel 直接渲染零成本，网页端按空行分段即可。将来需要富文本时新增 `descriptionMarkdown` 字段，不破坏兼容。
- 素材**直接复用 App Store Connect 的图标（1024×1024 PNG）与截图**，无额外加工。

示例（占位数据）：

```json
{
  "version": 1,
  "defaultLocale": "en",
  "updatedAt": "2026-10-13",
  "apps": [
    {
      "id": "example",
      "name": "Example App",
      "tagline": { "en": "One-line summary", "zh": "一句话简介" },
      "icon": "https://wangjin2013.github.io/ApplicationCenter/assets/icons/example.png",
      "category": { "en": "Utilities", "zh": "工具" },
      "description": { "en": "Paragraph one.\n\nParagraph two.", "zh": "第一段。\n\n第二段。" },
      "features": { "en": ["Highlight one"], "zh": ["亮点一"] },
      "screenshots": [
        "https://wangjin2013.github.io/ApplicationCenter/assets/screenshots/example-1.png"
      ],
      "appStoreUrl": "https://apps.apple.com/app/id123456789"
    }
  ]
}
```

## 5. API 契约（iOS 端对接规范）

- **端点**：`GET https://wangjin2013.github.io/ApplicationCenter/data/apps.json`（仓库 `wangjin2013/ApplicationCenter`）
- 无鉴权；UTF-8 JSON；GitHub Pages 自带 `Access-Control-Allow-Origin: *`（原生 App 本就不受 CORS 限制，此头只影响其他网页跨域读取）
- **前向兼容规则**：客户端必须忽略未知字段（Swift `Codable` 默认即如此）；新增可选字段不升级 `version`；删除字段/改语义才升级 `version`
- **生效延迟**：push 后 GitHub Pages 部署约 1 分钟；CDN 边缘缓存最长约 10 分钟
- **iOS 端建议**（客户端职责，此处仅作约定）：
  - 本地持久化最近一次成功解析的结果；请求失败时展示缓存数据，不出现空白
  - 数据量小（个位数 KB），全量拉取即可，无需分页
  - `appStoreUrl` 用 `openURL` 跳转系统 App Store，或用 `SKStoreProductController` 应用内展示

## 6. 网页端设计（index.html）

单文件单页，无框架、无构建：

- **路由**：无 hash → App 列表；`#app=<id>` → 对应详情（链接可直接分享）；无效 id → 回落列表
- **语言**：右上角 EN / 中文 切换，默认英语，选择记在 `localStorage`；UI 文案与数据字段同步切换
- **列表**：卡片网格（桌面多列、手机单列）：icon + 名称 + tagline + 分类标签
- **详情**：头部（icon、名称、tagline、"App Store 下载"按钮）→ 截图横滑画廊 → 描述分段 → 特性列表
- **样式**：`assets/style.css`，响应式；深浅色跟随系统 `prefers-color-scheme`；极简风格
- **容错**：fetch 失败时显示"数据加载失败，请重试"与重试按钮

## 7. 内容更新工作流（新增一个 App）

1. 图标与截图放入 `assets/icons/`、`assets/screenshots/`
2. 在 `data/apps.json` 的 `apps` 数组追加一个对象（放在期望的展示位置）
3. 本地预览：`python3 -m http.server` → 浏览器打开 `http://localhost:8000` 检查
4. commit + push；约 1 分钟后线上生效

详细步骤与规范固化在 README.md，作为维护者指南。

## 8. 验证方式

| 验证项 | 命令/操作 | 通过标准 |
|---|---|---|
| JSON 合法 | `python3 -m json.tool data/apps.json` | 无报错，输出格式化 JSON |
| 本地网页 | `python3 -m http.server` + 浏览器 | 列表渲染、点击进详情、App Store 链接可达、深浅色正常 |
| 线上 JSON | `curl -s https://…/data/apps.json` | HTTP 200，内容与仓库一致 |
| 线上网页 | 浏览器打开 Pages URL | 同"本地网页"标准 |
| iOS 联调 | App 内请求线上端点 | Codable 解析成功，列表/详情/跳转正常 |

## 9. 实施步骤

```
✅ 1. 建目录结构 + .nojekyll                     → 结构与第 3 节一致
✅ 2. 写 data/apps.json（4 个 App，占位文案）     → json.tool 校验通过
✅ 3. 写 index.html + assets/style.css           → 本地 http.server 冒烟通过
   4. 放图标/截图（待用户提供素材）                → verify: 页面显示真实图标
   5. git init + GitHub 建仓库 + push + 开启 Pages → verify: 通过第 8 节"线上 JSON/网页"项
   6. iOS 端按第 5 节契约接入（另一工程，另行开展） → verify: 模拟器内解析并展示
```

## 10. 未决问题（需要确认，不阻塞设计）

**已确认**：

1. **用户名 / 仓库名**：`wangjin2013` / `ApplicationCenter` → 站点 `https://wangjin2013.github.io/ApplicationCenter/`
2. **首批 App**：KnowCal、PCalcBox、MyClock - weather and alarm、Filingo
3. **语言**：中英双语、英语为主（`defaultLocale: "en"`），文案字段用 `{"en", "zh"}` 结构

**待用户提供**（当前均为占位内容）：

- 各 App 的正式文案（tagline / description / features）与分类
- 各 App 的图标（`assets/icons/<id>.png`）与截图
- 各 App 的 App Store 链接（`appStoreUrl` 当前留空 → 页面自动隐藏下载按钮）

## 11. 未来扩展（仅备忘，现在不做）

- `bundleId` 字段：iOS 端可判断"已安装 → 直接打开；未安装 → 跳 App Store"
- 多语言字段（`name: {"zh": …, "en": …}`，schema v2）
- 每个 App 的"最新版本与更新日志"
