# ApplicationCenter

我的 iOS App 介绍门户。网页与 iOS App 读取**同一份数据** `data/apps.json`，改一处即双端同步更新。

- 网页：https://wangjin2013.github.io/ApplicationCenter/
- 数据接口（iOS 端用）：https://wangjin2013.github.io/ApplicationCenter/data/apps.json

## 目录

```
index.html            门户页面（列表 + 详情，读取 apps.json 渲染）
assets/style.css      样式
assets/icons/         App 图标：<id>.png（1024×1024）
assets/screenshots/   App 截图：<id>-1.png、<id>-2.png …
data/apps.json        ★ 唯一数据源
design.md             设计文档
```

## 新增 / 修改一个 App

1. **图标**：1024×1024 PNG（直接用 App Store Connect 的那张），命名为 `assets/icons/<id>.png`
2. **截图**（可选）：`assets/screenshots/<id>-1.png`、`<id>-2.png` …
3. **编辑 `data/apps.json`**：在 `apps` 数组里加一个对象（字段见下）
4. **本地预览**：
   ```sh
   python3 -m http.server 8000
   ```
   浏览器打开 http://localhost:8000 检查
5. **发布**：
   ```sh
   git add -A && git commit -m "Add XXX" && git push
   ```
   约 1 分钟后线上生效（CDN 缓存最长 10 分钟）

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 小写英文标识，如 `filingo`。**发布后不要改**（详情页链接依赖它） |
| `name` | string | 是 | App 名称（各语言相同，不翻译） |
| `icon` | string | 是 | 图标完整 URL |
| `appStoreUrl` | string | 否 | App Store 链接；留空则不显示下载按钮 |
| `category` | 本地化文本 | 否 | 如 `{"en": "Utilities", "zh": "工具"}` |
| `tagline` | 本地化文本 | 是 | 一句话简介 |
| `description` | 本地化文本 | 是 | 详细介绍，**段落之间空一行**（`\n\n`） |
| `features` | 本地化数组 | 否 | 要点列表，如 `{"en": ["…"], "zh": ["…"]}` |
| `screenshots` | URL 数组 | 否 | 截图完整 URL；留空则不显示画廊 |

- **本地化文本** = `{"en": "…", "zh": "…"}`。客户端按当前语言取值，缺该语言时回退到 `en`。
- 顶层 `apps` 数组的**顺序就是网页展示顺序**。
- 换仓库名 / 换域名时，需要把 JSON 里所有完整 URL 批量替换一次。

## 注意事项

- 图片 URL 必须写**完整地址**——iOS App 拿到 JSON 后无法解析相对路径。
- 缺 `icon` 或图片 404 时，页面自动显示首字母方块；缺 `appStoreUrl` 时自动隐藏下载按钮（所以占位数据也能正常预览）。
- 当前 `data/apps.json` 里的**所有文案都是占位**（`[Placeholder …]` / `【占位…】`），图标文件也还没放，需要替换。

## iOS 端接入

- 请求：`GET https://wangjin2013.github.io/ApplicationCenter/data/apps.json`，无鉴权，UTF-8
- 解析：Swift `Codable` 直接映射；**未知字段会被自动忽略**（这正是前向兼容规则要求的行为）
- 语言：按当前语言读 `{en, zh}` 字段，缺失回退 `en`
- 建议：把最近一次成功的结果持久化到本地；请求失败时展示缓存，不要空白
