# 行迹：聊天推荐 × 手机相册接入方案

## 目标

把两路输入汇入同一条“待确认的旅途片段”流程：

1. 从聊天应用分享/复制旅行建议（文本、链接、图片）。
2. 从手机相册选择照片/视频，读取可用的拍摄时间和位置元数据。
3. 按时间与位置建议归入旅行、日期、城市/区域；用户确认后写入时间线，并补充日记感想。

“自动整理”只做建议，不静默发布、不擅自改写用户日记。

## 能力边界与入口

- **相册**：首选系统照片选择器，让用户挑选本次旅程的照片。iOS Photos Picker 对用户选中的项目授权，不需要授予整库权限；需要后台持续发现新照片时，再明确请求 PhotoKit 访问，并支持 iOS 的有限照片授权。[Apple Photos Picker](https://developer.apple.com/documentation/swiftui/view/photospicker%28ispresented%3Aselection%3Amaxselectioncount%3Aselectionbehavior%3Amatching%3Apreferreditemencoding%3Aphotolibrary%3A) · [PhotoKit 隐私说明](https://developer.apple.com/documentation/PhotoKit/delivering-an-enhanced-privacy-experience-in-your-photos-app)
- **Android 相册**：使用 Android Photo Picker 多选。它只把用户选中的媒体交给应用；不需要读取整个图库的权限。旧设备可由官方兼容路径回退到文档选择器。[Android Photo Picker](https://developer.android.com/training/data-storage/shared/photo-picker)
- **聊天推荐**：先做系统“分享至行迹”入口，接收分享文本、URL 和图片；若聊天应用不支持分享，则提供复制粘贴和手动添加。Android 可用 Sharesheet / `ACTION_SEND` 接收数据。[Android 接收分享](https://developer.android.com/develop/ui/compose/sharing/receive)
- **iOS 聊天接入**：实现 Share Extension，让用户从支持分享的 App 选中内容后交给行迹；同样保留粘贴入口。系统分享服务机制见 [Apple 分享内容文档](https://developer.apple.com/documentation/uikit/collaborating-and-sharing-copies-of-your-data)。
- **不读取聊天历史**：除非具体聊天产品提供正式授权 API，否则应用无法也不应绕过其沙箱批量抓取会话。分享/粘贴让用户明确选择要带入的内容。若“instinct”是特定平台，后续需按该平台公开 API 和分享能力适配。

## 建议的用户流程

1. 新建旅程，输入目的地与日期范围。
2. 点“导入聊天推荐”，从聊天 App 的分享菜单选择行迹；进入导入确认页，显示来源、文本/链接预览和建议城市/地点。用户可编辑、选旅程、保存为“灵感”或直接排入某日。
3. 点“连接相册 / 选照片”，调用系统照片选择器。本次所选图片在设备端读取拍摄时间、方向和可用 GPS；先显示匹配结果，用户确认后才生成时间线照片组。
4. 用户打开日期或区域卡片，补写感想。照片可拖到其它日期；地点/时间识别不确定时标为“待确认”。
5. 新收到的分享和新导入的媒体先进入“收件箱”，不会自动混入已完成日记。

## 数据流

```text
聊天 App --系统分享/粘贴--> 导入收件箱 --解析预览/用户确认--┐
                                                            ├--> TripEntry --> 时间线/区域日记
手机照片 App --系统照片选择器--> 设备端元数据提取/匹配预览--┘
```

默认在设备端完成媒体筛选和元数据匹配。服务端只同步用户明确保存的记录和所选媒体（如开启云同步）；提供“仅本机”模式。GPS 可在保存前移除，分享文本可随时删除。

## 统一数据结构（MVP）

```json
{
  "tripId": "trip_123",
  "entryId": "entry_456",
  "source": "chat_share | photo_picker | manual",
  "occurredAt": "2026-06-12T09:20:00+02:00",
  "timezone": "Europe/Rome",
  "place": { "name": "圣马可广场", "region": "威尼斯 · 圣马可区", "latitude": null, "longitude": null, "confidence": 0.72 },
  "text": "清晨的广场还很安静……",
  "recommendation": { "url": null, "sourceApp": null },
  "media": [{ "localAssetId": "platform-local-id", "capturedAt": "2026-06-12T09:20:00+02:00", "latitude": null, "longitude": null }],
  "status": "inbox | confirmed | archived",
  "createdAt": "2026-06-12T10:00:00+02:00"
}
```

不把本地照片路径或原始 EXIF 当跨设备 ID。云同步时为媒体创建应用自己的资源 ID；默认不保留未选图片的元数据。位置/地名解析失败仍可按日期归组。

## MVP 技术实施顺序

1. 把现有 HTML 原型迁移为 iOS/Android 共用的移动端应用外壳，保留当前时间线 UI。
2. 先接系统多选照片选择器与分享接收入口；建立统一 `ImportItem` / `TripEntry`，并将所有导入写入本地数据库。
3. 做日期匹配、重复照片去重、城市/区域手动确认，以及“照片组 + 日记感想”编辑。
4. 加可选登录和加密云同步；明确云端保留周期、删除同步和导出能力。
5. 在真实设备验证 iOS 有限图库授权、iCloud 未下载照片、Android 不同 Photo Picker 可用性、分享 App 的文本/链接/图片格式。

## 验收标准

- 用户可以从常见聊天应用分享一段文本或链接到行迹；行迹不需要聊天账号密码。
- 用户只选中的照片可被导入；拒绝权限、只授予部分照片、文件缺少 EXIF 时仍可继续手动整理。
- 根据照片时间建议日期；GPS 存在时建议地点；没有 GPS 时不会伪造地点。
- 用户可以确认/更改日期、地点、区域、日记、照片归属，并撤销导入。
- 删除旅程可同时清理应用副本和云端副本；不会删除系统相册原图。

## 仍需产品决策

- “instinct”具体指哪个聊天产品？这决定是否能做专用分享入口或官方 API 集成。
- 是否需要账号和多设备云同步？首版建议先本机使用，减少隐私范围和后端工作。
- 目标首发平台是 iOS、Android 还是同时上线？

