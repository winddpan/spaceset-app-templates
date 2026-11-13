# spaceset-app-templates

`spaceset-app-templates` 是 Spaceset 的应用模板仓库，收集并维护用于识别应用窗口、保存窗口参数和恢复窗口的 YAML 模板。每个模板由 YAML 配置和其中的脚本共同定义，便于为不同应用扩展一致的窗口管理行为。

[Spaceset](https://spaceset.app) 是一款 macOS 窗口管理工具，帮助你按工作场景组织和恢复应用窗口，让切换工作空间更简单。

每个 YAML 文件描述一个应用模板。YAML 文件和其中的脚本共同构成模板行为的真实来源。

## 内置模板

| 模板 | 参数策略 | 窗口匹配方式 |
| ---- | -------- | ------------ |
| [Default](templates/Default.yaml) | `none` | 选择边界最接近恢复目标的窗口，距离相同时选择较小的窗口 ID |
| [ChatGPT](templates/ChatGPT.yaml) | `none` | 为 `com.openai.codex` 使用与 Default 相同的边界匹配方式 |
| [DocumentApp](templates/DocumentApp.yaml) | `one` | 文档文件路径 |
| [Finder](templates/Finder.yaml) | `one` | 文件夹路径 |
| [Ghostty](templates/Ghostty.yaml) | `one` | 优先匹配 `documentPath` 中的工作目录，否则匹配目标边界 |
| [Terminal](templates/Terminal.yaml) | `one` | 优先匹配 `documentPath` 中的工作目录，否则匹配目标边界 |
| [VSCode](templates/VSCode.yaml) | `one` | 从 VS Code 保存的窗口状态读取文件夹或工作区文件路径，通过边界关联窗口；路径存在歧义时不匹配 |
| [Xcode](templates/Xcode.yaml) | `one` | 工作区文档路径 |
| [Safari](templates/Safari.yaml) | `many` | 第一个包含全部已保存 Tab 域名的候选窗口 |
| [Chromium](templates/Chromium.yaml) | `many` | 匹配已保存的 Tab 域名，通过边界关联浏览器窗口与候选窗口 |

Terminal 和 Ghostty 保存工作目录用于匹配现有窗口，当前启动和新建窗口脚本只打开默认终端会话，不传入该目录。Xcode 有已保存路径时打开该路径；没有路径时，`openNewWindow` 脚本会在存在文档窗口的情况下返回其窗口 ID。

## 文件名

文件名使用模板名称：

```text
DocumentApp.yaml
Safari.yaml
Chromium.yaml
```

建议使用稳定的 PascalCase 文件名。去掉 `.yaml` 后的文件名会显示在模板选择列表中。一个模板可以包含多个 Bundle ID。

## 最小结构

```yaml
bundleIds: [com.apple.TextEdit]
args: one

captureArguments:
  appleScript: |
    ...

resolveWindow:
  appleScript: |
    ...

launchApplication:
  shell: |
    ...

openNewWindow:
  appleScript: |
    ...
```

## 字段

| 字段                | 必填 | 含义                                    |
| ------------------- | ---- | --------------------------------------- |
| `bundleIds`         | 是   | 使用此模板的 Bundle ID 或正则表达式模式 |
| `args`              | 否   | 参数策略，默认值为 `none`               |
| `captureArguments`  | 否   | 读取当前窗口的参数                      |
| `resolveWindow`     | 否   | 根据参数或目标几何信息寻找现有窗口      |
| `launchApplication` | 是   | 应用未运行时启动应用                    |
| `openNewWindow`     | 是   | 应用已运行时创建新窗口                  |

`args` 支持以下固定值：

| 值     | 含义                           |
| ------ | ------------------------------ |
| `none` | 不保存或传递窗口参数           |
| `one`  | 只使用第一个非空参数           |
| `many` | 按原顺序保存和传递全部非空参数 |

`bundleIds` 中的每一项都可以是完整 Bundle ID 或正则表达式模式。完整匹配的优先级高于正则匹配。`"*"` 是保留的全匹配模式，优先级最低。

## 脚本字段

每个操作只能使用以下两个字段中的一个：

```yaml
captureArguments:
  shell: |
    ...
```

```yaml
captureArguments:
  appleScript: |
    ...
```

### `captureArguments`

读取文件路径、URL 或终端工作目录等参数。单个值会先归一化为只有一个元素的参数列表，再应用 `args` 策略。

- `shell` 通过 stdout 返回 JSON 字符串，例如 `"/Users/me/file.txt"`，或 JSON 数组，例如 `["/Users/me/file.txt"]`。
- `appleScript` 可以直接返回字符串或字符串列表。
- 返回空数组或不返回值，都表示没有读取到参数或读取失败。

### `resolveWindow`

寻找与已保存窗口对应的当前窗口。

- `shell` 匹配成功时通过 stdout 返回 JSON CG 窗口 ID，例如 `12345`。
- `appleScript` 匹配成功时直接返回 CG 窗口 ID；没有匹配时可以返回 `missing value`，也可以不返回值。
- Shell 没有 stdout，或 AppleScript 返回 `missing value`、没有返回值，都表示没有匹配窗口。
- 返回的窗口 ID 必须是 `{{candidateWindows}}` 中某个窗口对象的 `windowID`。

`{{windowInfo}}` 提供恢复目标的几何信息，不包含当前窗口的 `windowID`。没有参数匹配时，模板可以据此选择最接近目标的候选窗口。Default 和 ChatGPT 对四个边界值的绝对差求和，选择总距离最小的窗口；距离相同时选择较小的窗口 ID。Terminal 和 Ghostty 优先对规范化后的 `documentPath` 进行区分大小写的匹配，没有匹配时再使用相同的边界规则。

### `launchApplication`

用于启动应用。Shell 成功退出或 AppleScript 正常结束，表示命令执行完成；随后 Spaceset 会等待可用窗口。

脚本可以返回窗口的 CG 窗口 ID：Shell 通过 stdout 输出 JSON 窗口 ID，AppleScript 返回整数。Spaceset 会使用该 ID 确认命令对应的窗口。没有返回 ID 时，Spaceset 会通过应用焦点和窗口清单变化发现窗口。

### `openNewWindow`

用于在运行中的应用内打开窗口，命令完成和可选窗口 ID 的规则与 `launchApplication` 相同。Finder 和 Safari 会显式返回其创建的窗口 ID。

也可以使用以下保留值：

```yaml
openNewWindow: reuseExisting
```

```yaml
openNewWindow: commandN
```

`reuseExisting` 由恢复规划器分配已有窗口，不执行新建窗口命令。`commandN` 表示通过 Command-N 请求创建新窗口。

## 占位符

占位符只替换一次。模板作者不需要处理参数顺序和转义。每个占位符都应作为完整表达式使用，不要再额外添加引号。

| 占位符                      | 可用操作           | 含义                                                            |
| --------------------------- | ------------------ | --------------------------------------------------------------- |
| `{{bundleId}}`              | 所有操作           | 当前 Bundle ID                                                  |
| `{{appName}}`               | 所有操作           | 当前应用名称                                                    |
| `{{windowInfo}}`            | `captureArguments`、`resolveWindow` | 保存参数时为当前窗口对象，匹配窗口时为恢复目标的几何信息 |
| `{{candidateWindows}}`      | `resolveWindow`    | 当前候选窗口对象，包含窗口 ID、几何信息及可选文档路径         |
| `{{pid}}`                   | 所有操作           | 当前进程 ID，可能为空                                           |
| `{{arg}}`                   | 所有操作           | 第一个非空参数                                                  |
| `{{args}}`                  | 所有操作           | Shell 数组或 AppleScript 列表形式的全部参数                     |

当前窗口对象的 JSON 结构如下，其中 `documentPath` 为可选字段：

```json
{
  "windowID": 123,
  "frame": { "x": 1, "y": 2, "width": 100, "height": 200 },
  "appleScriptBounds": [1, 2, 101, 202],
  "documentPath": "/Users/me/project"
}
```

在 `resolveWindow` 中，`{{windowInfo}}` 只包含恢复目标的 `frame` 和 `appleScriptBounds`，不包含 `windowID` 或文档路径。`{{candidateWindows}}` 中的对象描述当前窗口，包含 `windowID`。

`documentPath` 是从窗口 `AXDocument` 属性获取的文件或工作目录的绝对路径，已经过路径规范化并解析符号链接。当前运行时为 DocumentApp、Terminal 和 Ghostty 模板读取此属性。路径不可用时，Shell JSON 会省略该字段，AppleScript record 中该字段为 `missing value`。

在 AppleScript 中，`{{windowInfo}}` 会展开为 record，`{{candidateWindows}}` 会展开为 record 列表。在 Shell 中，它们分别展开为 JSON 字符串和 JSON 字符串数组。脚本需要窗口几何信息时，可从窗口 record 读取 `appleScriptBounds` 或 `frame`。`appleScriptBounds` 按 `[左, 上, 右, 下]` 排列，使用四舍五入后的整数坐标。

Shell 写法：

```shell
targetArgs={{args}}
candidateWindows={{candidateWindows}}
open -b {{bundleId}} "${targetArgs[@]}"
```

AppleScript 写法：

```applescript
set targetURLs to {{args}}
set targetWindowInfo to {{windowInfo}}
set candidateWindowInfos to {{candidateWindows}}
```

没有参数时，`{{args}}` 在 Shell 中展开为 `()`，在 AppleScript 中展开为 `{}`。没有候选窗口时，`{{candidateWindows}}` 也遵循相同规则。没有当前窗口时，`{{windowInfo}}` 在 AppleScript 中展开为 `missing value`。没有参数时，`{{arg}}` 展开为空字符串。

## Default 示例

文件名：`Default.yaml`

```yaml
bundleIds: ["*"]
args: none

captureArguments:
  appleScript: |
    return {}

resolveWindow:
  appleScript: |
    set targetWindowInfo to {{windowInfo}}
    if targetWindowInfo is missing value then return missing value
    set targetBounds to appleScriptBounds of targetWindowInfo
    set candidateWindowInfos to {{candidateWindows}}
    set bestDistance to missing value
    set matchedWindowID to missing value

    repeat with candidateWindowInfoRef in candidateWindowInfos
        set candidateWindowInfo to contents of candidateWindowInfoRef
        set candidateBounds to appleScriptBounds of candidateWindowInfo
        set candidateID to windowID of candidateWindowInfo
        set candidateDistance to 0
        repeat with boundsIndex from 1 to 4
            set difference to (item boundsIndex of candidateBounds) - (item boundsIndex of targetBounds)
            if difference < 0 then set difference to 0 - difference
            set candidateDistance to candidateDistance + difference
        end repeat
        if bestDistance is missing value then
            set bestDistance to candidateDistance
            set matchedWindowID to candidateID
        else if candidateDistance < bestDistance or (candidateDistance is bestDistance and candidateID < matchedWindowID) then
            set bestDistance to candidateDistance
            set matchedWindowID to candidateID
        end if
    end repeat

    return matchedWindowID

launchApplication:
  shell: |
    targetArgs={{args}}
    open -b {{bundleId}} "${targetArgs[@]}"

openNewWindow: reuseExisting
```

`Default.yaml` 是优先级最低的 fallback 模板。存在精确 Bundle ID 模板时，优先使用精确匹配的模板。它会匹配边界最接近恢复目标的候选窗口，并复用已有窗口。

## Document App 示例

文件名：`DocumentApp.yaml`

```yaml
bundleIds:
  - com.apple.TextEdit
  - com.apple.Preview
  - com.coteditor.CotEditor
  - com.microsoft.Word
args: one

captureArguments:
  appleScript: |
    set targetWindowInfo to {{windowInfo}}

    tell application id {{bundleId}}
        try
            set documentPath to path of document of window id (windowID of targetWindowInfo)
            return {POSIX path of documentPath}
        on error
            return {}
        end try
    end tell

resolveWindow:
  appleScript: |
    set targetPath to {{arg}}
    set candidateWindowInfos to {{candidateWindows}}

    if targetPath is "" then return missing value

    tell application id {{bundleId}}
        repeat with candidateWindowInfoRef in candidateWindowInfos
            set candidateWindowInfo to contents of candidateWindowInfoRef
            set candidateID to windowID of candidateWindowInfo as integer
            try
                set candidateDocument to path of document of window id candidateID
                set candidatePath to POSIX path of candidateDocument
                if candidatePath is targetPath then return candidateID
            end try
        end repeat
    end tell

    return missing value

launchApplication:
  shell: |
    targetArgs={{args}}
    open -b {{bundleId}} "${targetArgs[@]}"

openNewWindow:
  appleScript: |
    tell application id {{bundleId}}
        set targetPath to {{arg}}
        if length of targetPath > 0 then
            try
                open (POSIX file targetPath)
            end try
        else
            try
                make new document
            on error
                try
                    make new window
                end try
            end try
        end if
    end tell
```

## Safari 示例

文件名：`Safari.yaml`

此模板只匹配规范化后的 Tab 域名。它返回第一个包含全部已保存域名的候选窗口。

```yaml
bundleIds: [com.apple.Safari]
args: many

captureArguments:
  appleScript: |
    set targetWindowInfo to {{windowInfo}}

    tell application id {{bundleId}}
        set tabURLs to {}
        repeat with browserTab in tabs of window id (windowID of targetWindowInfo)
            set tabURL to URL of browserTab
            if tabURL is not missing value then set end of tabURLs to tabURL
        end repeat
        return tabURLs
    end tell

resolveWindow:
  appleScript: |
    use framework "Foundation"

    property ca : current application

    on normalizedHost(rawURL)
        set value to rawURL as text
        if value does not contain "://" then set value to "https://" & value
        set parsedURL to ca's NSURL's URLWithString:value
        if parsedURL is missing value then return ""
        set valueHost to parsedURL's host()
        if valueHost is missing value then return ""
        set lowerHost to valueHost's lowercaseString()
        return lowerHost as text
    end normalizedHost

    on containsAll(targetHosts, liveHosts)
        repeat with targetRef in targetHosts
            if (contents of targetRef) is not in liveHosts then return false
        end repeat
        return true
    end containsAll

    set targetHosts to {}
    repeat with targetRef in {{args}}
        set targetHost to my normalizedHost(contents of targetRef)
        if targetHost is not "" then set end of targetHosts to targetHost
    end repeat

    if (count of targetHosts) is 0 then return missing value

    set candidateWindowInfos to {{candidateWindows}}

    tell application id {{bundleId}}
        repeat with candidateWindowInfoRef in candidateWindowInfos
            set candidateWindowInfo to contents of candidateWindowInfoRef
            set candidateID to windowID of candidateWindowInfo as integer
            try
                set candidateWindow to window id candidateID
                set liveHosts to {}

                repeat with browserTab in tabs of candidateWindow
                    set tabURL to URL of browserTab
                    if tabURL is not missing value then
                        set tabHost to my normalizedHost(tabURL)
                        if tabHost is not "" then set end of liveHosts to tabHost
                    end if
                end repeat

                if my containsAll(targetHosts, liveHosts) then return candidateID
            end try
        end repeat
    end tell

    return missing value

launchApplication:
  appleScript: |
    tell application id {{bundleId}}
        launch
        set targetURLs to {{args}}
        if (count of targetURLs) > 0 then
            make new document with properties {URL:(item 1 of targetURLs)}
            set targetWindow to front window
            tell targetWindow
                repeat with tabIndex from 2 to count of targetURLs
                    make new tab at end of tabs with properties {URL:(item tabIndex of targetURLs)}
                end repeat
            end tell
        else
            make new document
            set targetWindow to front window
        end if
        activate
        return id of targetWindow
    end tell

openNewWindow:
  appleScript: |
    tell application id {{bundleId}}
        launch
        set targetURLs to {{args}}
        if (count of targetURLs) > 0 then
            make new document with properties {URL:(item 1 of targetURLs)}
            set targetWindow to front window
            tell targetWindow
                repeat with tabIndex from 2 to count of targetURLs
                    make new tab at end of tabs with properties {URL:(item tabIndex of targetURLs)}
                end repeat
            end tell
        else
            make new document
            set targetWindow to front window
        end if
        activate
        return id of targetWindow
    end tell
```
