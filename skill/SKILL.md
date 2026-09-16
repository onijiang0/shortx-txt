---
name: shortx
description: ShortX 指令文件生成器。根据用户需求生成可导入 ShortX 的 .txt 自动指令(Rule)或一键指令(DirectAction)文件。当用户提到 ShortX、自动指令、一键指令、触发器、打卡自动化、通知转发、定时任务等手机自动化场景时使用。已验证可导入的格式与踩坑记录。
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
"@type": "type.googleapis.com/shortx.OcrDetect"
```

**正确**：
```json
"@type": "type.googleapis.com/FindAndClickViewByText"
"@type": "type.googleapis.com/WaitForIdle"
"@type": "type.googleapis.com/OcrDetect"
```

所有 `@type` 统一用 `type.googleapis.com/组件名`，**不要**加 `shortx.` 命名空间。

### 3. 顶层字段：精简

只保留必要字段，多余字段可能导致解析失败：

```json
{
  "facts": [],
  "conditions": [],
  "actions": [],
  "id": "RULE-xxx",
  "title": "标题",
  "description": "描述",
  "isEnabled": true,
  "condOp": "ALL",
  "hook": {},
  "quit": {},
  "versionCode": "1"
}
```

**不要添加**：`actionAsyncMode`、`parameters`、`author`、`lastUpdateTime`、`createTime`、`ruleSetId`（这些是 ShortX 导出时的内部字段，导入时不需要）。

### 4. 文本字段：避免换行符

ShowDanmu、ShowToast 等文本中避免 `\n` 换行，可能导致 JSON 解析问题。

### 5. ID 格式

触发器用 `F-001`、`F-002`，动作用 `A-001`、`A-002`，条件用 `C-001`，依次编号。

## 已验证可导入的组件

### 触发器（facts）

| @type | 用途 | 关键参数 |
|-------|------|----------|
| `type.googleapis.com/Alarm` | 定时触发 | `triggerAt: {hour, minutes}`, `repeat: {days: ["MONDAY",...]}` |
| `type.googleapis.com/AppBecomeFg` | 应用前台 | `apps: [{pkgName}]` |
| `type.googleapis.com/AppBecomeBg` | 应用后台 | `apps: [{pkgName}]` |
| `type.googleapis.com/NotificationPosted` | 新通知 | `record: {title, contentText, apps}` |
| `type.googleapis.com/ScreenOn` | 亮屏 | 无参数 |
| `type.googleapis.com/ScreenOff` | 息屏 | 无参数 |
| `type.googleapis.com/ChargerPlug` | 插入充电器 | 无参数 |
| `type.googleapis.com/WifiConnectedTo` | WiFi连接 | `ssidList: ["SSID"]` |
| `type.googleapis.com/BTConnectedTo` | 蓝牙连接 | `device: "设备名"` |

### 动作（actions）

| @type | 用途 | 关键参数 |
|-------|------|----------|
| `type.googleapis.com/ShowToast` | Toast提示 | `message` |
| `type.googleapis.com/ShowDanmu` | 弹幕 | `text`, `icon` |
| `type.googleapis.com/SetRingerMode` | 铃声模式 | `mode: "normal"/"silent"/"vibrate"` |
| `type.googleapis.com/PlayRingtone` | 播放铃声 | `ringtone: {title, uri, type}` |
| `type.googleapis.com/LaunchAppByPkg` | 启动应用 | `pkgAndUsers: [{first: "包名", second: "0"}]` |
| `type.googleapis.com/Delay` | 延时 | `timeString: "5"`, `timeUnit: "TimeUnit_S"` |
| `type.googleapis.com/FindAndClickViewByText` | 文本查找并点击 | `text`, `isRegex`, `timeout`, `method` |
| `type.googleapis.com/ShellCommand` | Shell命令 | `command` |
| `type.googleapis.com/HttpRequest` | HTTP请求 | `url`, `method`, `headers`, `requestBody` |
| `type.googleapis.com/ShowAlertDialog` | 对话框 | `title`, `message` |
| `type.googleapis.com/Vibrate` | 震动 | `vib1` (毫秒) |
| `type.googleapis.com/TTS` | 语音播报 | `text` |

### 条件（conditions）

| @type | 用途 | 关键参数 |
|-------|------|----------|
| `type.googleapis.com/CurrentPkgList` | 前台应用判断 | `pkgs: [{pkgName}]` |
| `type.googleapis.com/BatteryPercent` | 电量判断 | `value`, `op: "IntLessThan"` |
| `type.googleapis.com/TimeInRange` | 时间范围 | `range: {start: {hour, minutes}, end: {hour, minutes}}` |
| `type.googleapis.com/RequireWifiConnected` | WiFi已连接 | `requiredSSID: "*"` |
| `type.googleapis.com/EvaluateContextVar` | 上下文变量 | `varName`, `op`, `payload: {value}` |
| `type.googleapis.com/ScreenIsOn` | 屏幕亮着 | 无参数 |
| `type.googleapis.com/ChargeState` | 充电状态 | `requireIsCharge: true` |

## FindAndClickViewByText 参数详解

```json
{
  "@type": "type.googleapis.com/FindAndClickViewByText",
  "text": "打卡",
  "isRegex": false,
  "timeout": 8000,
  "method": 2,
  "customContextDataKey": {},
  "id": "A-007"
}
```

- `text`：要查找的文本，支持正则
- `isRegex`：是否正则匹配
- `timeout`：超时毫秒
- `method`：查找方式
  - `0` = FTM_UI_AUTO（仅无障碍）
  - `1` = FTM_OCR（仅OCR）
  - `2` = FTM_UI_AUTO_OCR（先无障碍，失败走OCR）**推荐**

上下文变量：`{matchedViewText}`, `{matchedViewId}`

## 完整示例：定时打开钉钉打卡

```json
{
  "facts": [{
    "@type": "type.googleapis.com/Alarm",
    "triggerAt": {"hour": 7, "minutes": 40},
    "repeat": {"days": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY", "SATURDAY"]},
    "customContextDataKey": {},
    "id": "F-001"
  }],
  "conditions": [],
  "actions": [{
    "@type": "type.googleapis.com/SetRingerMode",
    "mode": "normal",
    "customContextDataKey": {},
    "id": "A-001"
  }, {
    "@type": "type.googleapis.com/ShowToast",
    "message": "要打卡了",
    "customContextDataKey": {},
    "id": "A-002"
  }, {
    "@type": "type.googleapis.com/LaunchAppByPkg",
    "pkgAndUsers": [{"first": "com.alibaba.android.rimet", "second": "0"}],
    "customContextDataKey": {},
    "id": "A-003"
  }, {
    "@type": "type.googleapis.com/Delay",
    "timeString": "5",
    "useAlarm": false,
    "showCD": false,
    "timeUnit": "TimeUnit_S",
    "customContextDataKey": {},
    "id": "A-004"
  }, {
    "@type": "type.googleapis.com/FindAndClickViewByText",
    "text": "打卡",
    "isRegex": false,
    "timeout": 8000,
    "method": 2,
    "customContextDataKey": {},
    "id": "A-005"
  }],
  "id": "RULE-auto-clockin",
  "title": "到点自动打卡",
  "description": "自动打开钉钉并点击打卡",
  "isEnabled": true,
  "condOp": "ALL",
  "hook": {},
  "quit": {},
  "versionCode": "1"
}
###------###
{"type":"rule"}
```

## 变量系统

| 类型 | 语法 | 说明 |
|------|------|------|
| 上下文变量 | `{变量名}` | 触发器/动作自动填充 |
| 全局变量 | `globalVarOf$变量名` | 跨指令共享 |
| 局部变量 | `localVarOf$变量名` | 当前指令内 |
| 系统环境变量 | `%变量名%` | 如 `%BatteryLevel%` |

常用上下文变量：
- 通知：`{title}`, `{contentText}`, `{pkgName}`
- 应用：`{pkgName}`, `{appLabel}`
- Shell：`{shellOut}`, `{shellErr}`
- FindAndClick：`{matchedViewText}`, `{matchedViewId}`

## 生成流程

1. 确定类型：Rule（自动触发）还是 DA（手动执行）
2. 选触发器 → 选条件（可选）→ 选动作
3. 构建 JSON，遵守上述格式规范
4. 写入 `.txt` 文件，UTF-8 无 BOM 编码
5. 告知用户导入方式和注意事项
