---
name: shortx
description: ShortX 指令文件生成器。根据用户需求生成可导入 ShortX 的 .txt 自动指令(Rule)或一键指令(DirectAction)文件。当用户提到 ShortX、自动指令、一键指令、触发器、打卡自动化、通知转发、定时任务、淘金币、支付宝自动化等手机自动化场景时使用。已验证可导入的格式与踩坑记录。
---

# ShortX 指令文件生成器

生成可直接导入 ShortX 的 `.txt` 指令文件。

## 文件格式

```
{JSON 内容}
###------###
{"type":"rule"} 或 {"type":"da"}
```

- 第一部分：完整 JSON
- 第二部分：固定分隔符 `###------###`（3个# + 6个- + 3个#）
- 第三部分：`{"type":"rule"}` 表示自动指令，`{"type":"da"}` 表示一键指令

## 关键踩坑记录（必须遵守）

### 1. 编码：UTF-8 无 BOM

**必须**用 UTF-8 无 BOM 编码写入。带 BOM 会导致 ShortX 导入时白屏崩溃。

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($path, $content, $utf8NoBom)
```

### 2. @type 前缀：不要加 `shortx.`

**错误**（白屏崩溃）：
```json
"@type": "type.googleapis.com/shortx.FindAndClickViewByText"
"@type": "type.googleapis.com/shortx.WaitForIdle"
```

**正确**：
```json
"@type": "type.googleapis.com/FindAndClickViewByText"
"@type": "type.googleapis.com/WaitForIdle"
```

所有 `@type` 统一用 `type.googleapis.com/组件名`，**不要**加 `shortx.` 命名空间。

**嵌套类型同样适用**：OcrDetect 的 `rectSrc` 内嵌类型也要去掉前缀：
```json
// 错误
"rectSrc": {"@type": "type.googleapis.com/shortx.RectSourceFullScreen"}
// 正确
"rectSrc": {"@type": "type.googleapis.com/RectSourceFullScreen"}
```

### 3. 顶层字段：精简

Rule（自动指令）只需：
```json
{
  "facts": [], "conditions": [], "actions": [],
  "id": "RULE-xxx", "title": "标题", "description": "描述",
  "isEnabled": true, "condOp": "ALL",
  "hook": {}, "quit": {}, "versionCode": "1"
}
```

DA（一键指令）只需：
```json
{
  "actions": [],
  "id": "DA-xxx", "title": "标题", "description": "描述",
  "versionCode": "1", "hook": {}, "quit": {}
}
```

**不要添加**：`actionAsyncMode`、`parameters`、`author`、`lastUpdateTime`、`createTime`、`ruleSetId`。

### 4. 文本字段：避免换行符

ShowDanmu、ShowToast 等文本中避免 `\n` 换行。

### 5. ID 格式

触发器 `F-001`，动作 `A-001`，条件 `C-001`，循环内 `A-L01`。

## 从手机导入指令

```bash
# 1. 推送文件到手机
adb push rule.txt /sdcard/Download/rule.txt

# 2. 通过 SEND intent 分享给 ShortX
adb shell am start -a android.intent.action.SEND -t 'text/plain' \
  --eu android.intent.extra.STREAM 'content://com.android.externalstorage.documents/document/primary%3ADownload%2Frule.txt' \
  -p tornaco.apps.shortx

# 3. 在弹出的分享菜单中选择「ShortX指令/代码导入」
# 4. 点击保存按钮
```

## 指令记录（调试日志）

在指令编辑页底部：**其他选项 → 开启指令记录**

查看日志：编辑页三点菜单 → 动作记录

## 变量系统

| 类型 | 语法 | 说明 |
|------|------|------|
| 上下文变量 | `{变量名}` | 触发器/动作自动填充 |
| 全局变量 | `globalVarOf$变量名` | 跨指令共享 |
| 局部变量 | `localVarOf$变量名` | 当前指令内 |

### 变量追踪模式（判断动作是否成功）

```json
// 1. 重置变量
{"@type": "type.googleapis.com/WriteLocalVar", "varName": "hit", "valueAsString": "0"}

// 2. FindAndClick 结果写入变量
{"@type": "type.googleapis.com/FindAndClickViewByText",
 "text": "去完成", "isRegex": false, "timeout": 3000, "method": 2,
 "customContextDataKey": {"keys": [{"first": "matchedViewText", "second": "hit"}]}}

// 3. 判断变量是否非空
{"@type": "type.googleapis.com/IfThenElse",
 "If": [{"@type": "type.googleapis.com/EvaluateLocalVar",
         "op": "IsNotEmpty", "varName": "hit", "payload": {"value": ""}}],
 "IfActions": [...], "ElseActions": [...]}
```

## FindAndClickViewById（最可靠）

用控件资源 ID 查找并点击，比坐标和文字都稳定。

```json
{
  "@type": "type.googleapis.com/FindAndClickViewById",
  "viewId": "com.taobao.taobao:id/homepage_pop_view",
  "isRegex": false,
  "timeout": 8000
}
```

### 已知淘宝 View ID

| 控件 | ID |
|------|-----|
| 淘金币按钮 | `com.taobao.taobao:id/homepage_pop_view` |

**获取 View ID 方法**：ShortX 控件 ID 查看器，或 ADB `uiautomator dump`

## FindAndClickViewByText 参数

```json
{
  "@type": "type.googleapis.com/FindAndClickViewByText",
  "text": "目标文字或正则",
  "isRegex": false,
  "timeout": 3000,
  "method": 2
}
```

- `method`: 0=仅无障碍, 1=仅OCR, 2=先无障碍后OCR（推荐）
- `isRegex`: true 时 text 为正则表达式
- **重要**：淘宝等复杂 UI 中 OCR 匹配率低，关键导航建议用坐标兜底

## 坐标导航（2608×1200 小米手机）

| 位置 | 坐标 |
|------|------|
| 解锁上滑起点 | (1304, 1050) |
| 解锁上滑终点 | (1304, 400) |
| 淘宝领淘金币 | (510, 540) |
| 淘金币赚更多金币 | (560, 720) |
| 任务列表下滑 | (1304, 900) → (1304, 300) |

**注意**：坐标因设备分辨率而异，需根据实际调整。

## 已验证可导入的组件

### 触发器（facts）

| @type | 用途 |
|-------|------|
| `Alarm` | 定时，`triggerAt:{hour,minutes}`, `repeat:{days:[...]}` |
| `AppBecomeFg` | 应用前台 |
| `NotificationPosted` | 新通知 |
| `ScreenOn` / `ScreenOff` | 亮屏/息屏 |
| `WifiConnectedTo` | WiFi连接 |
| `ShakeDevice` | 摇晃手机 |

### 动作（actions）

| @type | 用途 | 关键参数 |
|-------|------|----------|
| `ShowToast` | Toast | `message` |
| `ShowDanmu` | 弹幕 | `text`, `icon` |
| `SetRingerMode` | 铃声模式 | `mode: "normal"/"silent"/"vibrate"` |
| `PlayRingtone` | 铃声 | `ringtone: {title, uri, type}` |
| `LaunchAppByPkg` | 启动应用 | `pkgAndUsers: [{first:"包名", second:"0"}]` |
| `Delay` | 延时 | `timeString`, `timeUnit: "TimeUnit_S"` |
| `InputTap` | 点击坐标 | `xs`, `ys` (字符串) |
| `InputSwipe` | 滑动 | `startXS/startYS/endXS/endYS/swipeTimeS` |
| `InjectKeyCode` | 按键 | `keyCode: 3=HOME, 4=BACK` |
| `FindAndClickViewByText` | 文字查找点击 | `text`, `isRegex`, `timeout`, `method` |
| `WriteLocalVar` | 写局部变量 | `varName`, `valueAsString` |
| `ShowAlertDialog` | 对话框 | `title`, `message` |
| `Vibrate` | 震动 | `vib1` |
| `TTS` | 语音 | `text` |

### 条件（conditions）

| @type | 用途 |
|-------|------|
| `EvaluateLocalVar` | 判断局部变量 |
| `EvaluateGlobalVar` | 判断全局变量 |
| `EvaluateContextVar` | 判断上下文变量 |
| `CurrentPkgList` | 前台应用 |
| `BatteryPercent` | 电量 |
| `TimeInRange` | 时间范围 |

## 循环结构（WhileLoop）

```json
{
  "@type": "type.googleapis.com/WhileLoop",
  "conditions": [],
  "actions": [...],
  "condOp": "ALL",
  "condOpPayload": {},
  "delay": 500,
  "repeatTimes": 15,
  "actionAsyncMode": "ActionAsyncMode_Sync"
}
```

## 淘宝淘金币自动化模式（实战验证）

### 导航流程
1. WakeupScreen → 解锁上滑 → HOME键
2. LaunchAppByPkg 淘宝 → 等6秒
3. 双击返回键关弹窗
4. InputTap (510,540) 点领淘金币 → 等4秒
5. 尝试领取签到金币（FindAndClickViewByText "领取|立即领取"）
6. 返回键
7. InputTap (560,720) 点赚更多金币 → 等2秒

### 任务循环模式
```
WhileLoop (15轮):
  1. FindAndClick "立即领取" (里程碑奖励)
  2. FindAndClick "去完成|立即领" → 浏览任务:
     - 7次随机速度滑动（350-450ms间隔2s）
     - 6次下滑+1次上滑
     - 返回键
  3. FindAndClick "去逛逛" → 支付宝任务:
     - 等6秒 → 双击返回 → LaunchAppByPkg切回淘宝
     - 重新导航: InputTap领淘金币 → InputTap赚更多金币
  4. FindAndClick "逛一逛" → 同上
  5. 都没找到 → 上滑列表
```

### 淘宝开屏弹窗处理

淘宝启动后随机出现 88VIP 消费券弹窗。**必须先检测再关闭**，避免误点首页元素。

```json
// 1. OCR 识别全屏
{"@type": "type.googleapis.com/OcrDetect",
 "rectSrc": {"@type": "type.googleapis.com/shortx.RectSourceFullScreen"},
 "threads": 4, "useSlim": true, "separator": "\n", "output_type": 1},

// 2. 检查是否包含弹窗文字
{"@type": "type.googleapis.com/IfThenElse",
 "If": [{"@type": "type.googleapis.com/EvaluateContextVar",
         "op": "Contains", "varName": "ocrResult", "payload": {"value": "88VIP"}}],
 "IfActions": [
   // 有弹窗 → 点X关闭
   {"@type": "type.googleapis.com/InputTap", "xs": "600", "ys": "2170"},
   {"@type": "type.googleapis.com/Delay", "timeString": "1", ...}
 ],
 "ElseActions": []  // 无弹窗 → 跳过
}
```

弹窗 X 按钮坐标约 (600, 2170)（1208×2608）。检测关键词：`88VIP`、`平台消费券`。

### 淘金币控件找不到时回首页重试

```json
// 1. 尝试点击淘金币
{"@type": "type.googleapis.com/WriteLocalVar", "varName": "nav", "valueAsString": "0"},
{"@type": "type.googleapis.com/FindAndClickViewById",
 "viewId": "com.taobao.taobao:id/homepage_pop_view",
 "timeout": 5000,
 "customContextDataKey": {"keys": [{"first": "matchedViewId", "second": "nav"}]}},

// 2. 判断是否成功
{"@type": "type.googleapis.com/IfThenElse",
 "If": [{"@type": "type.googleapis.com/EvaluateLocalVar",
         "op": "IsNotEmpty", "varName": "nav"}],
 "IfActions": [
   // 成功 → 继续
   {"@type": "type.googleapis.com/Delay", "timeString": "4", ...}
 ],
 "ElseActions": [
   // 失败 → HOME + 重开淘宝 + 重试
   {"@type": "type.googleapis.com/InjectKeyCode", "keyCode": 3},
   {"@type": "type.googleapis.com/LaunchAppByPkg", ...},
   {"@type": "type.googleapis.com/Delay", "timeString": "5", ...},
   {"@type": "type.googleapis.com/FindAndClickViewById", ...},
   {"@type": "type.googleapis.com/Delay", "timeString": "4", ...}
 ]
}
```

### 关键经验
- 已完成任务自动从列表消失，无需判断
- 支付宝任务（去逛逛/逛一逛）会跳转支付宝 App 或 webview
- 支付宝返回后需 LaunchAppByPkg 切回淘宝并重新导航
- 浏览任务需要模拟滑动，静止等待可能不被识别
- 任务完成时顶部显示「已得 XX」
- 里程碑奖励（10次/20次）出现「立即领取」按钮

### 常用应用包名

| 应用 | 包名 |
|------|------|
| 淘宝 | `com.taobao.taobao` |
| 支付宝 | `com.eg.android.AlipayGphone` |
| 钉钉 | `com.alibaba.android.rimet` |
| 微信 | `com.tencent.mm` |
| QQ | `com.tencent.mobileqq` |

## 生成流程

1. 确定类型：Rule（自动触发）还是 DA（手动执行）
2. 选触发器 → 选条件（可选）→ 选动作
3. 构建 JSON，遵守格式规范（无BOM、无shortx.前缀、精简字段）
4. 写入 `.txt` 文件
5. 推送到手机并指导导入
6. 告知用户如何开启指令记录调试
