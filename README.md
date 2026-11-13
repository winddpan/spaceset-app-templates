# spaceset-app-templates

`spaceset-app-templates` is Spaceset's application template repository. It collects and maintains YAML templates for identifying application windows, capturing window arguments, and restoring windows. Each template's behavior is defined by its YAML configuration and embedded scripts.

[Spaceset](https://spaceset.app) is a macOS window management tool that helps you organize and restore application windows by work context, making it easier to switch between workspaces.

Each YAML file defines one application template. The file and its embedded scripts are the source of truth for that template.

## Included templates

| Template | Arguments | Window matching |
| -------- | --------- | --------------- |
| [Default](templates/Default.yaml) | `none` | Closest bounds to the restore target; lower window ID breaks ties |
| [ChatGPT](templates/ChatGPT.yaml) | `none` | Same bounds matching as Default, for `com.openai.codex` |
| [DocumentApp](templates/DocumentApp.yaml) | `one` | Document file path |
| [Finder](templates/Finder.yaml) | `one` | Folder path |
| [Ghostty](templates/Ghostty.yaml) | `one` | Working directory from `documentPath`, then closest target bounds |
| [Terminal](templates/Terminal.yaml) | `one` | Working directory from `documentPath`, then closest target bounds |
| [VSCode](templates/VSCode.yaml) | `one` | Folder or workspace file path from VS Code's saved window state, associated by bounds; ambiguous paths are rejected |
| [Xcode](templates/Xcode.yaml) | `one` | Workspace document path |
| [Safari](templates/Safari.yaml) | `many` | First candidate containing all saved Tab hosts |
| [Chromium](templates/Chromium.yaml) | `many` | Saved Tab hosts, with bounds used to associate browser windows with candidates |

Terminal and Ghostty capture a working directory for matching existing windows. Their launch and new-window scripts currently open the default terminal session without passing that directory. Xcode opens the saved path when present; without a path, its `openNewWindow` script returns an existing document window ID when available.

## File naming

Use the template name as the filename:

```text
DocumentApp.yaml
Safari.yaml
Chromium.yaml
```

Use a stable PascalCase filename. The filename without `.yaml` is shown in the template picker. One template may contain multiple Bundle IDs.

## Minimal structure

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

## Fields

| Field               | Required | Meaning                                                  |
| ------------------- | -------- | -------------------------------------------------------- |
| `bundleIds`         | Yes      | Bundle IDs or regular-expression patterns                |
| `args`              | No       | Argument policy; defaults to `none`                      |
| `captureArguments`  | No       | Reads arguments from the current window                  |
| `resolveWindow`     | No       | Finds an existing window using arguments or target geometry |
| `launchApplication` | Yes      | Starts the application when it is not running            |
| `openNewWindow`     | Yes      | Creates a window when the application is already running |

`args` accepts one of these values:

| Value  | Meaning                                        |
| ------ | ---------------------------------------------- |
| `none` | Do not save or pass window arguments           |
| `one`  | Use only the first non-empty argument          |
| `many` | Keep and pass all non-empty arguments in order |

Each `bundleIds` entry can be an exact Bundle ID or a regular-expression pattern. Exact matches have higher priority than regular-expression matches. `"*"` is the reserved catch-all pattern and always has the lowest priority.

## Script fields

Each operation uses exactly one of these keys:

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

Reads values such as file paths, URLs, or a terminal working directory. A single value is normalized to a one-item argument list before the `args` policy is applied.

- `shell` writes either a JSON string, such as `"/Users/me/file.txt"`, or a JSON array, such as `["/Users/me/file.txt"]`, to stdout.
- `appleScript` returns either a string or a string list.
- An empty array or no return value means that no argument was captured or that the read failed.

### `resolveWindow`

Finds the live window that represents the saved window.

- `shell` writes a matched CG window ID as JSON, for example `12345`.
- `appleScript` returns a matched CG window ID, returns `missing value`, or returns nothing.
- No Shell stdout, an AppleScript `missing value`, or no AppleScript return value means that no window was matched.
- The returned ID must be the `windowID` of one of the objects in `{{candidateWindows}}`.

`{{windowInfo}}` provides the restore target's geometry without a live `windowID`. Templates can use it to choose the closest candidate when no argument match is available. Default and ChatGPT minimize the sum of absolute differences between the four bounds values, breaking ties by lower window ID. Terminal and Ghostty first try a normalized, case-sensitive `documentPath` match, then use the same bounds rule.

### `launchApplication`

Runs to launch the application. A successful Shell exit or a successful AppleScript execution means that the command completed; Spaceset then waits for a usable window.

The script may return the window's CG window ID: write it as JSON to Shell stdout or return it as an AppleScript integer. Spaceset uses that ID to identify the resulting window. If no ID is returned, Spaceset discovers the window from application focus and window inventory changes.

### `openNewWindow`

Runs to open a window in a running application. It follows the same completion and optional window ID rules as `launchApplication`. Finder and Safari explicitly return the window they create.

It may also use one of these reserved values:

```yaml
openNewWindow: reuseExisting
```

```yaml
openNewWindow: commandN
```

`reuseExisting` lets the restore planner assign an existing window; it does not execute a new-window command. `commandN` requests a new window with Command-N.

## Placeholders

Placeholders are replaced once. Template authors do not need to handle argument order or escaping. Use each placeholder as a complete expression and do not add another pair of quotes.

| Placeholder               | Available in       | Meaning                                                                        |
| ------------------------- | ------------------ | ------------------------------------------------------------------------------ |
| `{{bundleId}}`            | All operations     | Current Bundle ID                                                              |
| `{{appName}}`             | All operations     | Current application name                                                       |
| `{{windowInfo}}`          | `captureArguments`, `resolveWindow` | Current live window when capturing; target geometry when resolving |
| `{{candidateWindows}}`    | `resolveWindow`    | Candidate live window objects with IDs, geometry, and optional document paths |
| `{{pid}}`                 | All operations     | Current process ID, when available                                             |
| `{{arg}}`                 | All operations     | First non-empty argument                                                       |
| `{{args}}`                | All operations     | All arguments as a Shell array or AppleScript list                             |

Live window objects have this JSON structure; `documentPath` is optional:

```json
{
  "windowID": 123,
  "frame": { "x": 1, "y": 2, "width": 100, "height": 200 },
  "appleScriptBounds": [1, 2, 101, 202],
  "documentPath": "/Users/me/project"
}
```

For `resolveWindow`, `{{windowInfo}}` contains only `frame` and `appleScriptBounds` for the restore target. It has no `windowID` or document path. The objects in `{{candidateWindows}}` describe live windows and include `windowID`.

`documentPath`, when available, is an absolute file or working-directory path from the window's `AXDocument` attribute, with path components standardized and symlinks resolved. The current runtime reads this attribute for DocumentApp, Terminal, and Ghostty templates. Unavailable paths are omitted from Shell JSON and become `missing value` in AppleScript records.

In AppleScript, `{{windowInfo}}` expands to a record and `{{candidateWindows}}` expands to a list of records. In Shell, they expand to a JSON string and an array of JSON strings, respectively. Read `appleScriptBounds` or `frame` from the window record when a script needs window geometry. `appleScriptBounds` contains rounded integer values in `[left, top, right, bottom]` order.

For Shell:

```shell
targetArgs={{args}}
candidateWindows={{candidateWindows}}
open -b {{bundleId}} "${targetArgs[@]}"
```

For AppleScript:

```applescript
set targetURLs to {{args}}
set targetWindowInfo to {{windowInfo}}
set candidateWindowInfos to {{candidateWindows}}
```

An empty `{{args}}` becomes `()` in Shell and `{}` in AppleScript. An empty `{{candidateWindows}}` follows the same rule. If there is no current window, `{{windowInfo}}` expands to `missing value` in AppleScript. An empty `{{arg}}` becomes an empty string.

## Default example

Filename: `Default.yaml`

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

`Default.yaml` is the lowest-priority fallback template. A template with an exact Bundle ID always wins over this template. It matches the candidate closest to the restore target's bounds and reuses existing windows.

## Document App example

Filename: `DocumentApp.yaml`

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

## Safari example

Filename: `Safari.yaml`

This template matches only normalized Tab hosts. It returns the first candidate window that contains every saved host.

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
