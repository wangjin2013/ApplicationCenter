# iOS 端接入文档：ApplicationCenter 数据接口

本文档面向 **实现 iOS 客户端的开发者**。iOS App 通过一个公开的 JSON 接口获取所有 App 的介绍信息，在自己的界面里原生渲染（不嵌 WebView）。网页门户与 iOS App 读的是同一份数据，只需在仓库里改 `data/apps.json`，双端同时更新。

- 目标平台：**iOS 15+**（`AsyncImage` / `.refreshable` / `async await` 均需 iOS 15）
- 改动本仓库的 `data/apps.json` 后，约 1 分钟部署完成，CDN 边缘缓存最长 10 分钟

---

## 1. 接口

| 项 | 值 |
|---|---|
| URL | `https://wangjin2013.github.io/ApplicationCenter/data/apps.json` |
| 方法 | `GET` |
| 鉴权 | 无 |
| 编码 | UTF-8 |
| 响应 | `application/json` |
| 分页 | 无（全量，约 17 KB） |
| CORS | `Access-Control-Allow-Origin: *`（原生 App 不受此限制，无需关心） |

> 未经许可的域名/仓库改名会改变此 URL；如果换域名，需要同步改 iOS 端常量与 JSON 里的图片绝对地址。

---

## 2. 数据格式

```json
{
  "version": 1,
  "defaultLocale": "en",
  "updatedAt": "2026-10-13",
  "apps": [
    {
      "id": "knowcal",
      "name": "KnowCal",
      "icon": "https://wangjin2013.github.io/ApplicationCenter/assets/icons/knowcal.png",
      "iconSmall": "https://wangjin2013.github.io/ApplicationCenter/assets/icons/thumbs/knowcal.png",
      "appStoreUrl": "https://apps.apple.com/us/app/knowcal/id6813519210",
      "category": { "en": "Business", "zh": "商务" },
      "tagline": {
        "en": "Smart business card scanner & manager",
        "zh": "智能名片扫描与管理"
      },
      "description": {
        "en": "第一段。\n\n第二段。",
        "zh": "第一段。\n\n第二段。"
      },
      "features": {
        "en": ["要点一", "要点二"],
        "zh": ["要点一", "要点二"]
      },
      "screenshots": [
        "https://wangjin2013.github.io/ApplicationCenter/assets/screenshots/knowcal-1.jpg"
      ],
      "screenshotThumbnails": [
        "https://wangjin2013.github.io/ApplicationCenter/assets/screenshots/thumbs/knowcal-1.jpg"
      ]
    }
  ]
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | Int | 数据格式版本号，当前 `1` |
| `defaultLocale` | String? | 缺失语言时的回退语言 |
| `updatedAt` | String? | 最后更新日期 `YYYY-MM-DD` |
| `apps` | Array | App 列表，**数组顺序即展示顺序** |
| `apps[].id` | String | 稳定标识，可用于埋点/置顶；**不会变** |
| `apps[].name` | String | App 名称，各语言通用，**不是**本地化对象 |
| `apps[].icon` | String | 图标完整 URL，1024×1024 PNG |
| `apps[].iconSmall` | String? | 小图标完整 URL，256×256 PNG。**可能缺失**，缺失时用 `icon` |
| `apps[].appStoreUrl` | String? | App Store 链接。**可能为空字符串**（如尚未上架） |
| `apps[].category` | 本地化文本? | 分类标签 |
| `apps[].tagline` | 本地化文本 | 一句话简介 |
| `apps[].description` | 本地化文本 | 正文，段落间用 `\n\n` 分隔 |
| `apps[].features` | 本地化数组? | 要点列表 |
| `apps[].screenshots` | [String]? | 截图完整 URL，各语言共用 |
| `apps[].screenshotThumbnails` | [String]? | 截图缩略图完整 URL（宽 300px JPEG），**与 `screenshots` 等长同序**。可能缺失，缺失时用 `screenshots` |

**本地化文本** 有两种形态，客户端必须都支持：

- 对象 `{"en": "...", "zh": "..."}`
- 普通字符串（等价于只有 `en`）

取值规则：优先当前语言 → 缺失回退 `zh`（仅当当前语言是中文）→ 再回退 `en`。

---

## 3. 完整代码

可直接复制进项目的单文件实现：

```swift
import Foundation
import Combine
import SwiftUI

// MARK: - 数据模型

/// 本地化文本：可能是普通字符串，也可能是 {"en": ..., "zh": ...}。
struct Localized: Decodable {
    let values: [String: String]

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let text = try? container.decode(String.self) {
            values = ["en": text]
        } else {
            values = try container.decode([String: String].self)
        }
    }

    /// 取指定语言；缺失时回退英文。
    func text(_ language: String) -> String {
        if let exact = values[language], !exact.isEmpty { return exact }
        if language == "zh", let zh = values["zh"], !zh.isEmpty { return zh }
        return values["en"] ?? ""
    }
}

/// 本地化字符串数组，形态同上。
struct LocalizedList: Decodable {
    let values: [String: [String]]

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let list = try? container.decode([String].self) {
            values = ["en": list]
        } else {
            values = try container.decode([String: [String]].self)
        }
    }

    func items(_ language: String) -> [String] {
        if let exact = values[language], !exact.isEmpty { return exact }
        if language == "zh", let zh = values["zh"], !zh.isEmpty { return zh }
        return values["en"] ?? []
    }
}

struct AppCatalog: Decodable {
    let version: Int
    let defaultLocale: String?
    let updatedAt: String?
    let apps: [AppItem]
}

struct AppItem: Decodable, Identifiable {
    let id: String
    let name: String
    let icon: String
    let iconSmall: String?               // 小图标；缺失时用 icon
    let appStoreUrl: String?
    let category: Localized?
    let tagline: Localized
    let detail: Localized            // 对应 JSON 里的 "description"
    let features: LocalizedList?
    let screenshots: [String]?
    let screenshotThumbnails: [String]?  // 与 screenshots 等长同序；缺失时用 screenshots

    enum CodingKeys: String, CodingKey {
        case id, name, icon, iconSmall, appStoreUrl, category, tagline, features, screenshots, screenshotThumbnails
        case detail = "description"
    }

    /// 没有链接（例如尚未上架）时为 nil，界面据此隐藏下载按钮。
    var storeURL: URL? {
        guard let raw = appStoreUrl?.trimmingCharacters(in: .whitespaces), !raw.isEmpty else { return nil }
        return URL(string: raw)
    }

    /// 从商店链接里取出数字 id，用于 SKStoreProductViewController 应用内展示。
    var storeItemIdentifier: Int? {
        guard let raw = appStoreUrl,
              let range = raw.range(of: "id", options: .backwards) else { return nil }
        return Int(raw[range.upperBound...].prefix { $0.isNumber })
    }
}

// MARK: - 语言

enum CatalogLanguage {
    /// 英语为主：系统首选语言是中文才用中文，其余一律英文。
    static var current: String {
        Locale.preferredLanguages.first?.prefix(2).lowercased() == "zh" ? "zh" : "en"
    }
}

// MARK: - 网络 + 缓存

enum CatalogError: Error {
    case badStatus(Int)
}

final class AppCatalogService {
    static let shared = AppCatalogService()

    private let endpoint = URL(string: "https://wangjin2013.github.io/ApplicationCenter/data/apps.json")!
    private let cacheURL = FileManager.default
        .urls(for: .cachesDirectory, in: .userDomainMask)[0]
        .appendingPathComponent("app-catalog.json")
    private let decoder = JSONDecoder()

    /// 同步读上次成功的结果，用于立刻上屏、离线可读。
    func cachedCatalog() -> AppCatalog? {
        guard let data = try? Data(contentsOf: cacheURL) else { return nil }
        return try? decoder.decode(AppCatalog.self, from: data)
    }

    /// 拉取最新数据；成功则写缓存。失败返回 nil，调用方继续用缓存。
    @discardableResult
    func refresh() async -> AppCatalog? {
        do {
            let (data, response) = try await URLSession.shared.data(from: endpoint)
            if let http = response as? HTTPURLResponse, !(200..<300).contains(http.statusCode) {
                throw CatalogError.badStatus(http.statusCode)
            }
            let catalog = try decoder.decode(AppCatalog.self, from: data)
            try? data.write(to: cacheURL, options: .atomic)
            return catalog
        } catch {
            return nil
        }
    }
}

// MARK: - 状态管理

@MainActor
final class AppCatalogStore: ObservableObject {
    @Published private(set) var apps: [AppItem] = []
    @Published private(set) var isLoading = false

    /// 先上缓存，再静默刷新。网络失败不报错、不清空。
    func load() async {
        if let cached = AppCatalogService.shared.cachedCatalog() {
            apps = cached.apps
        }
        isLoading = true
        defer { isLoading = false }
        if let fresh = await AppCatalogService.shared.refresh() {
            apps = fresh.apps
        }
    }
}

// MARK: - 界面

struct AppListView: View {
    @StateObject private var store = AppCatalogStore()
    private let language = CatalogLanguage.current

    var body: some View {
        NavigationView {
            List(store.apps) { app in
                NavigationLink {
                    AppDetailView(app: app, language: language)
                } label: {
                    HStack(spacing: 12) {
                        // 列表里图标只有 52pt，用 256px 小图即可；iconSmall 缺失时回退原图
                        AsyncImage(url: URL(string: app.iconSmall ?? app.icon)) { image in
                            image.resizable().scaledToFill()
                        } placeholder: {
                            Color.secondary.opacity(0.15)
                        }
                        .frame(width: 52, height: 52)
                        .clipShape(RoundedRectangle(cornerRadius: 12, style: .continuous))

                        VStack(alignment: .leading, spacing: 3) {
                            Text(app.name).font(.headline)
                            Text(app.tagline.text(language))
                                .font(.subheadline)
                                .foregroundStyle(.secondary)
                                .lineLimit(2)
                        }
                    }
                }
            }
            .task { await store.load() }
            .refreshable { await store.load() }
            .navigationTitle(language == "zh" ? "我的应用" : "My Apps")
        }
    }
}

struct AppDetailView: View {
    let app: AppItem
    let language: String

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 18) {
                HStack(spacing: 14) {
                    AsyncImage(url: URL(string: app.icon)) { image in
                        image.resizable().scaledToFill()
                    } placeholder: {
                        Color.secondary.opacity(0.15)
                    }
                    .frame(width: 68, height: 68)
                    .clipShape(RoundedRectangle(cornerRadius: 16, style: .continuous))

                    VStack(alignment: .leading, spacing: 4) {
                        Text(app.name).font(.title3.bold())
                        Text(app.tagline.text(language))
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                    }
                }

                // 未上架时 storeURL 为 nil，按钮自动不显示
                if let url = app.storeURL {
                    Link(destination: url) {
                        Text(language == "zh" ? "前往 App Store" : "View on the App Store")
                            .font(.headline)
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 12)
                            .background(Color.accentColor, in: Capsule())
                            .foregroundStyle(.white)
                    }
                }

                if let shots = app.screenshots, !shots.isEmpty {
                    let thumbs = app.screenshotThumbnails ?? []
                    ScrollView(.horizontal, showsIndicators: false) {
                        HStack(spacing: 12) {
                            ForEach(Array(shots.enumerated()), id: \.offset) { index, url in
                                // 渐进加载：缩略图先上屏，原图到达后淡入替换
                                AsyncImage(url: URL(string: url),
                                           transaction: Transaction(animation: .easeInOut(duration: 0.25))) { image in
                                    image.resizable().scaledToFit()
                                } placeholder: {
                                    if index < thumbs.count, let thumb = URL(string: thumbs[index]) {
                                        AsyncImage(url: thumb) { image in
                                            image.resizable().scaledToFit()
                                        } placeholder: {
                                            Color.secondary.opacity(0.15)
                                        }
                                    } else {
                                        Color.secondary.opacity(0.15)
                                    }
                                }
                                .frame(height: 340)
                                .clipShape(RoundedRectangle(cornerRadius: 14, style: .continuous))
                            }
                        }
                    }
                }

                // 正文按空行分段
                ForEach(Array(app.detail.text(language)
                    .components(separatedBy: "\n\n")
                    .filter { !$0.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty }
                    .enumerated()), id: \.offset) { _, paragraph in
                    Text(paragraph).font(.body)
                }

                let features = app.features?.items(language) ?? []
                if !features.isEmpty {
                    Text(language == "zh" ? "主要特性" : "Highlights").font(.headline)
                    ForEach(features, id: \.self) { item in
                        Text("• " + item).font(.subheadline)
                    }
                }
            }
            .padding(20)
        }
        .navigationTitle(app.name)
        .navigationBarTitleDisplayMode(.inline)
    }
}
```

### 说明

- **`description` 改名 `detail`**：`description` 在 Swift 里与 `CustomStringConvertible` 同名，容易踩坑，故用 `CodingKeys` 映射。
- **截图渐进加载**：把加载缩略图的 `AsyncImage` 嵌在原图 `AsyncImage` 的 `placeholder` 里，即可实现「缩略图先上屏、原图到达后替换」；`screenshotThumbnails` 缺失时代码自动退化为只加载原图。缩略图单张约 12–43 KB，原图约 40–177 KB。
- **`NavigationView` 而非 `NavigationStack`**：前者 iOS 13+ 可用（iOS 16 起被弃用但仍能跑），为了兼容 iOS 15 才这么写。**如果 App 最低版本是 iOS 16+，换成 `NavigationStack` 更好。**
- **iOS 13/14 兼容**：把 `URLSession.shared.data(from:)` 换成 `dataTask(with:)` + `withCheckedThrowingContinuation`；`AsyncImage` 换成第三方图片库或自建缓存。
- **UIKit 项目**：第 3 节的模型与 `AppCatalogService` 不依赖 SwiftUI，可直接用；把 `AppCatalogStore` 换成你自己的 ViewModel 即可。
- **应用内展示 App Store 页**（不跳出 App）：
  ```swift
  import StoreKit

  let controller = SKStoreProductViewController()
  controller.loadProduct(withParameters: [SKStoreProductParameterITunesItemIdentifier: app.storeItemIdentifier!])
  present(controller, animated: true)
  ```
  取不到數字 id 时回退到 `UIApplication.shared.open(url)`。

---

## 4. 行为约定（请务必遵守）

| 约定 | 原因 |
|---|---|
| **必须忽略未知字段** | 服务端会随时新增可选字段。Swift `Codable` 默认即如此，不要手写严格解析 |
| **`appStoreUrl` 可能为空** | 例如 Filingo 尚未上架。此时隐藏下载按钮，不要显示死链 |
| **`icon` / `screenshots` 可能加载失败** | 一律提供占位图，不要留空白或崩溃 |
| **`iconSmall` / `screenshotThumbnails` 可能缺失或长度不齐** | 回退 `icon` / `screenshots`（按索引对应），不要报错 |
| **必须本地缓存最近一次成功的结果** | 请求失败时展示缓存，避免推荐页空白 |
| **不要阻塞启动** | 进入推荐页时 `task` 拉取即可；先渲染缓存，再静默刷新 |
| **不要依赖 `version` 做判断** | 除非它变化；当前恒为 `1` |
| **拉取频率不要高于 10 分钟一次** | CDN 缓存 `max-age=600`，更频繁没有收益 |

---

## 5. 验证

```sh
# 接口可达性与内容
curl -s https://wangjin2013.github.io/ApplicationCenter/data/apps.json | python3 -m json.tool | head -40

# 缩略图可达性（应返回 200）
curl -sI https://wangjin2013.github.io/ApplicationCenter/assets/screenshots/thumbs/knowcal-1.jpg | head -1

# 只看关键字段
curl -s https://wangjin2013.github.io/ApplicationCenter/data/apps.json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
      print(d['version'], d['updatedAt']); \
      [print(' ', a['id'], '|', a['name'], '| store:', bool(a['appStoreUrl'])) for a in d['apps']]"
```

当前线上数据：

```
1 2026-10-15
  knowcal  | KnowCal                      | store: True
  myclock  | MyClock - Weather and Alarm  | store: True
  filingo  | Filingo                      | store: False   ← 尚未上架，按钮应隐藏
```

联调通过标准：列表渲染出 3 个 App、点击能进详情、Filingo 不显示下载按钮、其余两个能跳到 App Store、断网后仍能看到上次的数据。
