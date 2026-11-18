# 鲸鲸 BLE 网关（fork 适配）

这个 fork 的目标是让「鲸鲸」外接设备（SenseCAP Watcher，ESP32-S3 BLE）能跟上 dsh-Go 上游的更新，而不再靠改官方 APK 交付。官方仓库暂不接受外部 PR（`CONTRIBUTING.md`），所以本分支只作为参考实现/讨论材料，配合 issue [#1487](https://github.com/ccch1mneyyy/dsh-Go/issues/1487) 使用。

## 分支

- `main`：跟随上游 `ccch1mneyyy/dsh-Go` 的 `main`，由 `.github/workflows/sync-upstream.yml` 每天尝试 merge。冲突时 Action 会变红，需要手动处理。
- `feat/whale-ble-gateway`：本分支，从 `main` 拉出。

## 上游 1.1.0 的结构变化（对本功能的直接影响）

1. **设置页改成了索引/spot 体系**：`app/lib/ui/settings/index/spots.dart` 的 `SettingsPageId` + `SettingsPageMeta` + `SettingsSpot`，页面不再是手写 SettingsScreen 列表。本分支按这个体系登记了「鲸鲸蓝牙」页，而不是覆盖旧文件。
2. **插件体系已完整**：插件可以声明 native capabilities（`app/lib/plugins/native`、`sdk/plugin-kit`）。桥接不再需要 guest 注入/替换 `DshProcess`/`RuntimeController`；应当作为普通 DSH 插件安装，宿主只提供 BLE 能力（或独立的网关）。
3. **原生插件注册**：`MainActivity.provideFlutterEngine` 里逐项 `engine.plugins.add(...)`；本分支只加了一行 `EmbodiedBlePlugin()`。
4. **targetSdk 仍是 28**：延续 legacy Bluetooth 模型，所以权限用 `BLUETOOTH` + `BLUETOOTH_ADMIN` + `ACCESS_FINE_LOCATION`（和 1.0.16 覆盖集一致）。上游原先把 `BLUETOOTH` 用 `tools:node="remove"` 去掉了，本分支已移除该 remove 并补权限。
5. 前台服务：`EmbodiedBleService` 声明为 `foregroundServiceType="specialUse"`，和上游其他常驻服务一致。

## 本分支已移植的内容

- `app/android/.../embodied/`：7 个 Kotlin 文件（GATT 扫描/连接/分帧/WebSocket/前台服务/Flutter 插件）。
- `app/lib/embodied/`：channel/state/strings。
- `app/lib/ui/settings/embodied_page.dart` + `spots.dart` 的页面/索引锚点。
- `MainActivity` 注册与 Manifest 权限/服务。
- **没有**移植 1.0.16 覆盖集里的 guest 注入、`DshProcess`/`RuntimeController`/assets 替换；这些在新插件体系下应被淘汰。

## 还没做

- 桥插件化：把 `bridge/` 打成 `sdk/plugin-kit` 能检查/安装的 DSH 插件（或插件市场包），通过官方 Plugins 页面安装，而不是塞进 APK assets。
- BLE 能力形式：目前仍是「App 内置网关 + 设置页」；若要更通用，应按 #1071 方向把 `ctx.dshGo.ble` 宿主能力开放给插件。
- 文案：本分支的页面索引标题暂时是中文硬编码，合并前应迁到 `app/lib/l10n/parts/*.arb` 并跑 l10n 生成器。
- 验证：`flutter analyze` 已无 error/warning（只剩项目原有的 info）；Kotlin 编译在本机被 `sqlite3` native asset 下载阻塞，未完成；真机未验证。
