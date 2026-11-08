# spaceset-app-templates

`spaceset-app-templates` 是 Spaceset 的应用模板仓库，收集并维护用于识别应用窗口、保存窗口参数和恢复窗口的 YAML 模板。每个模板由 YAML 配置和其中的脚本共同定义，便于为不同应用扩展一致的窗口管理行为。

[Spaceset](https://spaceset.app) 是一款 macOS 窗口管理工具，帮助你按工作场景组织和恢复应用窗口，让切换工作空间更简单。

每个 YAML 文件描述一个应用模板。YAML 文件和其中的脚本共同构成模板行为的真实来源。

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
| `resolveWindow`     | 否   | 根据已保存的参数寻找现有窗口            |
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

- `shell` 匹配成功时通过 stdout 返回 JSON 窗口 ID，例如 `12345`。
- `appleScript` 匹配成功时直接返回窗口 ID；没有匹配时可以返回 `missing value`，也可以不返回值。
- Shell 没有 stdout，或 AppleScript 返回 `missing value`、没有返回值，都表示没有匹配窗口。
- 返回的窗口 ID 必须来自 `{{candidatesWindowIDs}}`。

### `launchApplication`

应用未运行时执行。脚本返回值不会作为结果使用。Shell 成功退出或 AppleScript 正常结束，都表示启动命令执行完成。

### `openNewWindow`

应用已运行时执行，成功规则与 `launchApplication` 相同。

也可以使用以下保留值：

```yaml
openNewWindow: reuseExisting
```

```yaml
openNewWindow: commandN
```

`reuseExisting` 表示复用已有窗口，`commandN` 表示请求创建一个新窗口。

## 占位符

占位符只替换一次。模板作者不需要处理参数顺序和转义。每个占位符都应作为完整表达式使用，不要再额外添加引号。

| 占位符                    | 可用操作           | 含义                                                     |
| ------------------------- | ------------------ | -------------------------------------------------------- |
| `{{bundleId}}`            | 所有操作           | 当前 Bundle ID                                           |
| `{{appName}}`             | 所有操作           | 当前应用名称                                             |
| `{{windowID}}`            | `captureArguments` | 当前窗口 ID                                              |
| `{{pid}}`                 | 所有操作           | 当前进程 ID，可能为空                                    |
| `{{candidatesWindowIDs}}` | 所有操作           | 候选窗口 ID 列表，会展开为 Shell 数组或 AppleScript 列表 |
| `{{arg}}`                 | 所有操作           | 第一个非空参数                                           |
| `{{args}}`                | 所有操作           | Shell 数组或 AppleScript 列表形式的全部参数              |

Shell 写法：

```shell
targetArgs={{args}}
candidateIDs={{candidatesWindowIDs}}
open -b {{bundleId}} "${targetArgs[@]}"
```

AppleScript 写法：

```applescript
set targetURLs to {{args}}
set candidateIDs to {{candidatesWindowIDs}}
```

没有参数时，`{{args}}` 在 Shell 中展开为 `()`，在 AppleScript 中展开为 `{}`。`{{candidatesWindowIDs}}` 遵循相同规则。没有参数时，`{{arg}}` 展开为空字符串。

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
    return missing value

launchApplication:
  shell: |
    targetArgs={{args}}
    open -b {{bundleId}} "${targetArgs[@]}"

openNewWindow: reuseExisting
```

`Default.yaml` 是优先级最低的 fallback 模板。存在精确 Bundle ID 模板时，优先使用精确匹配的模板。

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
    tell application id {{bundleId}}
        try
            set documentPath to path of document of window id {{windowID}}
            return {POSIX path of documentPath}
        on error
            return {}
        end try
    end tell

resolveWindow:
  appleScript: |
    set targetPath to {{arg}}
    set candidateIDs to {{candidatesWindowIDs}}

    if targetPath is "" then return missing value

    tell application id {{bundleId}}
        repeat with candidateIDRef in candidateIDs
            set candidateID to (contents of candidateIDRef) as integer
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
    tell application id {{bundleId}}
        set tabURLs to {}
        repeat with browserTab in tabs of window id {{windowID}}
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

    set candidateIDs to {{candidatesWindowIDs}}

    tell application id {{bundleId}}
        repeat with candidateIDRef in candidateIDs
            set candidateID to (contents of candidateIDRef) as integer
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
            tell front window
                repeat with tabIndex from 2 to count of targetURLs
                    make new tab at end of tabs with properties {URL:(item tabIndex of targetURLs)}
                end repeat
            end tell
        else
            make new document
        end if

        activate
    end tell

openNewWindow:
  appleScript: |
    tell application id {{bundleId}}
        launch
        set targetURLs to {{args}}
        if (count of targetURLs) > 0 then
            make new document with properties {URL:(item 1 of targetURLs)}
            tell front window
                repeat with tabIndex from 2 to count of targetURLs
                    make new tab at end of tabs with properties {URL:(item tabIndex of targetURLs)}
                end repeat
            end tell
        else
            make new document
        end if

        activate
    end tell
```
