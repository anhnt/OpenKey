# OpenKey — Codebase Analysis for AI Agents

> **TL;DR:** OpenKey is a cross-platform (macOS + Windows, Linux WIP) open-source Vietnamese input method editor (IME) written in C++17/Objective-C/Win32, licensed GPLv3. It intercepts keyboard events via OS-level hooks, runs keystrokes through a shared portable C++ engine, and emits corrected Vietnamese text using backspace-rewrite. This document is a briefing for any AI agent that needs to understand, modify, or extend the codebase.

---

## 1. Project Overview

| Attribute | Detail |
|---|---|
| **Name** | OpenKey |
| **Purpose** | Vietnamese keyboard input method (Bộ gõ tiếng Việt) |
| **Author** | Mai Vũ Tuyên (maivutuyen.91@gmail.com) |
| **Homepage** | [open-key.org](http://open-key.org) |
| **Upstream Repo** | [github.com/tuyenvm/OpenKey](https://github.com/tuyenvm/OpenKey) |
| **This Fork** | [github.com/anhnt/OpenKey](https://github.com/anhnt/OpenKey) |
| **License** | GPL v3.0 |
| **Latest macOS** | 2.0.3 (versionCode 47) |
| **Latest Windows** | 2.0.2 (versionCode 131074) |
| **Language** | C++17 core, Objective-C++ (macOS bridge), C++/Win32 (Windows frontend) |

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     PLATFORM LAYER                       │
│  ┌───────────────────┐  ┌─────────────────────────────┐ │
│  │  macOS (ModernKey) │  │  Windows (Win32)            │ │
│  │  - CGEventTap hook │  │  - Low-level keyboard hook │ │
│  │  - ObjC++ bridge   │  │  - Win32 dialogs           │ │
│  │  - Cocoa UI        │  │  - System tray             │ │
│  └────────┬──────────┘  └──────────────┬──────────────┘ │
│           │                            │                 │
│           └──────────┬─────────────────┘                 │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              CROSS-PLATFORM C++ ENGINE               │ │
│  │  Engine.cpp  Vietnamese.cpp  Macro.cpp               │ │
│  │  ConvertTool.cpp  SmartSwitchKey.cpp                 │ │
│  └─────────────────────────────────────────────────────┘ │
│                      │                                   │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────────┐ │
│  │          PLATFORM KEYCODE MAPS                       │ │
│  │  platforms/mac.h  platforms/win32.h  platforms/linux.h│ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Key design principle:** The engine is a pure C++ library with zero OS dependencies (no UI, no system calls). Platforms feed it raw key codes via `vKeyHandleEvent()` and receive a `vKeyHookState` struct describing what backspaces to emit and what replacement characters to type. This makes the core fully portable across macOS, Windows, and (future) Linux.

---

## 3. Directory Layout

```
OpenKey/
├── README.md                        # Vietnamese-language README
├── CHANGELOG.md                     # Release history (v1.0 → v2.0.3)
├── LICENSE                          # GPLv3
├── version.json                     # {"latestVersion":...} — used by auto-updater
├── macOS_Build.md                   # Build instructions for macOS
├── agents.md                        # This file — codebase analysis for AI agents
├── Sources/
│   └── OpenKey/
│       ├── engine/                  # ★ CROSS-PLATFORM CORE (the brains)
│       │   ├── Engine.h             #   Public API header — all engine entry points
│       │   ├── Engine.cpp           #   Main keystroke processing loop
│       │   ├── DataType.h           #   Enums, structs, bitmask macros
│       │   ├── Vietnamese.h/.cpp    #   Vowel/consonant tables, code tables, tone rules
│       │   ├── Macro.h/.cpp         #   Text-expansion (abbreviation) system
│       │   ├── ConvertTool.h/.cpp   #   Encoding conversion (Unicode ↔ VNI ↔ TCVN3)
│       │   ├── SmartSwitchKey.h/.cpp#   Per-app input-method memory
│       │   └── platforms/
│       │       ├── mac.h            #   macOS keycode constants
│       │       ├── win32.h          #   Windows VK_* keycode constants
│       │       └── linux.h          #   Linux keycode constants (WIP)
│       ├── macOS/
│       │   ├── OpenKey.entitlements #   App sandbox entitlements
│       │   ├── OpenKey.xcodeproj/   #   Xcode project
│       │   ├── ModernKey/           #   ★ macOS main application
│       │   │   ├── AppDelegate.h/.m #     NSApplicationDelegate — menu bar, prefs, global vars
│       │   │   ├── OpenKey.mm       #     ★★ Core bridge: CGEventTap → Engine → CGEventPost
│       │   │   ├── OpenKeyManager.h/.m #  Event tap init/stop, clipboard, auto-update
│       │   │   ├── ViewController.h/.m  #  Settings UI
│       │   │   ├── AboutViewController.h/.m # About dialog + update check
│       │   │   ├── MacroViewController.h/.mm # Macro table UI
│       │   │   ├── ConvertToolViewController.h/.mm # Convert tool UI
│       │   │   ├── MyTextField.h/.m #     Custom text field for key event handling
│       │   │   ├── MJAccessibilityUtils.h/.m # Accessibility permission check
│       │   │   └── main.m           #     App entry point
│       │   └── OpenKeyHelper/       #   Login/update helper app
│       │       ├── AppDelegate.h/.m #     Minimal helper
│       │       └── main.m           #     Entry point
│       ├── win32/
│       │   ├── README.md
│       │   └── OpenKey/OpenKey/     #   ★ Windows main application
│       │       ├── main.cpp         #     wWinMain entry point
│       │       ├── AppDelegate.h/.cpp#    Global state, settings, dialog management
│       │       ├── OpenKeyManager.h/.cpp  # Engine init/free, keyboard hook
│       │       ├── OpenKeyHelper.h/.cpp   # Registry, clipboard, update check
│       │       ├── SystemTrayHelper.h/.cpp# System tray icon + menu
│       │       ├── MainControlDialog.h/.cpp# Main floating settings panel
│       │       ├── MacroDialog.h/.cpp     # Text expansion config dialog
│       │       ├── ConvertToolDialog.h/.cpp# Encoding conversion dialog
│       │       ├── BaseDialog.h/.cpp      # Base Win32 dialog class
│       │       ├── AboutDialog.h/.cpp     # About dialog
│       │       ├── OpenKey.rc / resource.h # Win32 resources
│       │       └── *.ico                  # Tray icons (Eng, Viet, Convert, etc.)
│       └── linux/
│           └── README.md            # Linux port status (WIP)
```

---
## 4. Engine Deep Dive (`Sources/OpenKey/engine/`)

### 4.1 Public API (`Engine.h`)

The platform layer interacts with the engine exclusively through these entry points:

```cpp
void* vKeyInit();                          // Init engine, returns HookState pointer
void vKeyHandleEvent(event, state, keyCode, capsStatus, otherControlKey);
                                           // ★ MAIN FUNCTION — process one keystroke
Uint32 getCharacterCode(const Uint32&);     // Convert engine-internal code → real Unicode
void startNewSession();                     // Reset word buffer (space, click, etc.)
void vEnglishMode(const vKeyEventState& state, const Uint16& data,
                  const bool& isCaps, const bool& otherControlKey);
                                           // Process keystroke in English mode (for macros)
void vTempOffSpellChecking();               // Ctrl+key: temporarily disable spell check
void vSetCheckSpelling();                   // Re-enable spell check
void vTempOffEngine(const bool& off=true);  // Temporarily disable entire engine (Cmd/Alt key)
wstring utf8ToWideString(const string& str);// UTF8 ↔ wide-string conversion helpers
string wideStringToUtf8(const wstring& str);
```

### 4.2 Global State Variables (all `extern int`)

These are **declared in `Engine.h`** and **defined by the platform layer** (`AppDelegate.m` for macOS, `AppDelegate.cpp` for Windows). They serve as the engine's configuration.

**Cross-platform variables:**

| Variable | Values | macOS Default | Win32 Default | Description |
|---|---|---|---|---|
| `vLanguage` | 0=English, 1=Vietnamese | 1 | 1 | Current input mode |
| `vInputType` | 0=Telex, 1=VNI, 2=SimpleTelex1, 3=SimpleTelex2 | 0 | 0 | Typing method (four types!) |
| `vFreeMark` | 0/1 | 0 (hardcoded) | 0 (hardcoded) | Free tone mark placement (legacy, disabled) |
| `vCodeTable` | 0=Unicode, 1=TCVN3, 2=VNI, 3=UniCompound, 4=CP1258 | 0 | 0 | Output encoding |
| `vSwitchKeyStatus` | packed bitfield | 0x7A000206 | 0x5A00025A | Switch-key shortcut (Option+Z / Alt+Z) |
| `vCheckSpelling` | 0/1 | 1 | 1 | Enable Vietnamese spelling check |
| `vUseModernOrthography` | 0=òa/úy, 1=oà/uý | 1 | 1 | Modern tone placement |
| `vQuickTelex` | 0/1 | 0 | 0 | cc→ch, gg→gi, kk→kh, nn→ng, qq→qu, pp→ph, tt→th, uu→ươ |
| `vRestoreIfWrongSpelling` | 0/1 | 0 | 1 | If word invalid, restore original keystrokes |
| `vFixRecommendBrowser` | 0/1 | 1 | 0 | Fix Chrome/Safari/Firefox/Edge autocorrect |
| `vUseMacro` | 0/1 | 1 | 1 | Enable text expansion |
| `vUseMacroInEnglishMode` | 0/1 | 1 | 1 | Expand macros even in English mode |
| `vAutoCapsMacro` | 0/1 | 0 | 0 | Auto-capitalize expanded macro text |
| `vSendKeyStepByStep` | 0/1 | 0 | 1 | Send keys one-by-one (0=batch for speed) |
| `vUseSmartSwitchKey` | 0/1 | 1 | 1 | Remember input method per application |
| `vUpperCaseFirstChar` | 0/1 | 0 | 0 | Auto-capitalize first letter after . ! ? |
| `vTempOffSpelling` | 0/1 | 0 | 0 | Allow Ctrl+key to temporarily skip spell check |
| `vAllowConsonantZFWJ` | 0/1 | 0 | 0 | Allow Z, F, W, J as initial consonants |
| `vQuickStartConsonant` | 0/1 | 0 | 0 | f→ph, j→gi, w→qu shorthand |
| `vQuickEndConsonant` | 0/1 | 0 | 0 | g→ng, h→nh, k→ch shorthand |
| `vRememberCode` | 0/1 | 1 | 1 | Auto-remember code table per app (v2.0+) |
| `vOtherLanguage` | 0/1 | 1 | 1 | Turn off Vietnamese when typing in another language (v2.0+) |
| `vTempOffOpenKey` | 0/1 | 0 | 0 | Temporarily disable OpenKey via hotkey (Cmd/Alt, v2.0+) |
| `vFixChromiumBrowser` | 0/1 | 0 | 0 | Shift+LeftArrow trick for Chromium browsers (beta, v2.0+) |

**macOS-only variables** (defined in `AppDelegate.m`):

| Variable | Values | Default | Description |
|---|---|---|---|
| `vShowIconOnDock` | 0/1 | 0 | Show app icon in macOS Dock (v2.0+) |
| `vPerformLayoutCompat` | 0/1 | 0 | Compatibility mode for keyboard layout |

**Windows-only variables** (defined in `AppDelegate.cpp`):

| Variable | Values | Default | Description |
|---|---|---|---|
| `vUseGrayIcon` | 0/1 | 0 | Use gray system tray icon |
| `vShowOnStartUp` | 0/1 | 0 | Show control panel on startup |
| `vRunWithWindows` | 0/1 | 1 | Auto-start with Windows |
| `vSupportMetroApp` | 0/1 | 1 | Support Windows Store (Metro) apps |
| `vCreateDesktopShortcut` | 0/1 | 0 | Create desktop shortcut |
| `vRunAsAdmin` | 0/1 | 0 | Run with administrator privileges |
| `vCheckNewVersion` | 0/1 | 0 | Auto-check for updates on startup |

### 4.3 `vSwitchKeyStatus` Bit Layout

The switch-key shortcut configuration is a packed 32-bit integer:

| Bits | Macro | Description |
|---|---|---|
| 0-7 | `GET_SWITCH_KEY(data)` | Key code for the switch hotkey |
| 8 | `HAS_CONTROL(data)` | Control modifier |
| 9 | `HAS_OPTION(data)` | Option/Alt modifier |
| 10 | `HAS_COMMAND(data)` | Command (macOS) modifier |
| 11 | `HAS_SHIFT(data)` | Shift modifier |
| 15 | `HAS_BEEP(data)` | Play beep on switch |

**Setters**: `SET_SWITCH_KEY(data, key)`, `SET_CONTROL_KEY(data, val)`, `SET_OPTION_KEY(data, val)`, `SET_COMMAND_KEY(data, val)`.

### 4.4 Result Protocol: `vKeyHookState`

After calling `vKeyHandleEvent()`, the platform checks the global `HookState`:

```cpp
struct vKeyHookState {
    Byte code;     // 0=vDoNothing, 1=vWillProcess, 2=vBreakWord,
                   //   3=vRestore, 4=vReplaceMaro, 5=vRestoreAndStartNewSession
    Byte backspaceCount;  // How many backspaces to emit
    Byte newCharCount;    // How many new characters to type
    Byte extCode;         // 1=WordBreak, 2=DeleteKey, 3=NormalKey, 4=NoEmptyChar
    Uint32 charData[32];  // Replacement character codes (Unicode code points)
    vector<Uint32> macroKey;   // Macro trigger key sequence
    vector<Uint32> macroData;  // Macro expansion character codes
};
```

> **Note:** The enum value is `vReplaceMaro` (note the typo: "Maro" not "Macro") in `HoolCodeState` in `DataType.h`.

**Platform processing pseudocode:**
```
1. Call vKeyHandleEvent(event, state, keyCode, caps, ctrl)
2. If code == vDoNothing → pass event through unchanged
3. If code == vWillProcess/vRestore:
   a. (Optional) emit invisible char to break browser autocomplete
   b. Emit backspaceCount backspaces
   c. Emit charData[0..newCharCount-1] as Unicode characters
4. If code == vReplaceMaro → emit macro expansion text
5. If code == vBreakWord → start new word session
6. Return NULL (swallow event) or return original event (pass through)
```

### 4.5 Internal Word Buffer (`Engine.cpp`)

The engine maintains a `TypingWord[32]` ring buffer where each element is a `Uint32` bit-packed character:

| Bits | Field | Description |
|---|---|---|
| 0-15 | `CHAR_MASK` (0xFFFF) | Character or key code |
| 16 | `CAPS_MASK` (0x10000) | Capital letter (Shift or CapsLock) |
| 17 | `TONE_MASK` (0x20000) | Has circumflex accent (^) |
| 18 | `TONEW_MASK` (0x40000) | Has breve/horn (w) |
| 19 | `MARK1_MASK` (0x80000) | Dấu Sắc (á) |
| 20 | `MARK2_MASK` (0x100000) | Dấu Huyền (à) |
| 21 | `MARK3_MASK` (0x200000) | Dấu Hỏi (ả) |
| 22 | `MARK4_MASK` (0x400000) | Dấu Ngã (ã) |
| 23 | `MARK5_MASK` (0x800000) | Dấu Nặng (ạ) |
| 19-23 | `MARK_MASK` (0xF80000) | Combined: has any tone mark |
| 24 | `STANDALONE_MASK` (0x1000000) | Standalone key (W for ư/ơ) |
| 25 | `CHAR_CODE_MASK` (0x2000000) | 1=character code, 0=key code |
| 14 | `END_CONSONANT_MASK` (0x4000) | Flag for quick-end-consonant shortcuts |
| 15 | `CONSONANT_ALLOW_MASK` (0x8000) | Flag for allow-consonant-ZFWJ shortcuts |
| 31 | `PURE_CHARACTER_MASK` (0x80000000) | Pure Unicode (no tone processing) |

### 4.6 Input Methods

The `ProcessingChar[][]` table defines which keys trigger special processing for each input method:

| Index | Method | Special Keys (in order: Tone Marks, Modifiers) |
|---|---|---|
| 0 | **Telex** | S, F, R, X, J, A, O, E, W, D, Z |
| 1 | **VNI** | 1, 2, 3, 4, 5, 6, 6, 7, 8, 9, 0 |
| 2 | **Simple Telex 1** | S, F, R, X, J, A, O, E, W, D, Z |
| 3 | **Simple Telex 2** | S, F, R, X, J, A, O, E, W, D, Z |

> **Key details:**
> - VNI has two KEY_6 entries (positions 5 and 6): first for tone mark ^, second for the dấu mũ vowel modifier.
> - Simple Telex 1 and 2 share the same key set but are distinct `vInputType` values.
> - The `IS_SPECIALKEY` macro in `DataType.h` handles all four types with nested ternary operators.

| Method | Tone Marks | Vowel Modifiers |
|---|---|---|
| **Telex** (0) | s=sắc, f=huyền, r=hỏi, x=ngã, j=nặng | w=ư/ơ, aa=â, ee=ê, oo=ô, dd=đ, aw=ă, ow=ơ, uw=ư |
| **VNI** (1) | 1=sắc, 2=huyền, 3=hỏi, 4=ngã, 5=nặng | 6=^, 7=ư/ơ, 8=ă, 9=đ |
| **Simple Telex 1** (2) | s=sắc, f=huyền, r=hỏi, x=ngã, j=nặng | aa=â, ee=ê, oo=ô, dd=đ, aw=ă, ow=ơ (no standalone w→ư) |
| **Simple Telex 2** (3) | s=sắc, f=huyền, r=hỏi, x=ngã, j=nặng | aa=â, ee=ê, oo=ô, dd=đ, aw=ă, ow=ơ (no standalone w→ư) |

### 4.7 Code Tables (`Vietnamese.cpp`)

`_codeTable[0..4]` maps internal character codes to real Unicode/encoding output:

| Index | Encoding | Description |
|---|---|---|
| 0 | Unicode dựng sẵn | Precomposed Unicode |
| 1 | TCVN3 (ABC) | Legacy Vietnamese encoding |
| 2 | VNI Windows | Legacy VNI encoding |
| 3 | Unicode tổ hợp | Unicode Compound (decomposed) |
| 4 | CP 1258 | Vietnamese Windows code page |

> **Note:** The `vCodeTable` comment in `Engine.h` only lists codes 0-2 (Unicode, TCVN3, VNI-Windows), but the actual engine supports all 5 tables (0-4).

### 4.8 Macro System (`Macro.h/.cpp`)

In-memory text expansion with disk persistence:

```cpp
struct MacroData {
    string macroText;           // The trigger text (e.g., "btw")
    string macroContent;        // The expansion text (e.g., "by the way")
    vector<Uint32> macroContentCode; // Pre-converted content for current code table
};
```

| Function | Purpose |
|---|---|
| `initMacroMap(const Byte* pData, const int& size)` | Load macros from saved binary blob |
| `getMacroSaveData(vector<Byte>& outData)` | Serialize all macros for disk persistence |
| `findMacro(vector<Uint32>& key, vector<Uint32>& macroContentCode)` | Look up expansion by key sequence; returns true if found |
| `hasMacro(const string& macroName)` | Check if macro name exists |
| `getAllMacro(vector<vector<Uint32>>& keys, vector<string>& texts, vector<string>& contents)` | Get all macros for UI display |
| `addMacro(const string& macroText, const string& macroContent)` | Add/update a macro; returns true on success |
| `deleteMacro(const string& macroText)` | Remove a macro; returns true on success |
| `onTableCodeChange()` | Rebuild all `macroContentCode` when code table changes |
| `saveToFile(const string& path)` | Write all macros to file as binary blob |
| `readFromFile(const string& path, const bool& append=true)` | Read macros from file |

> **Important:** `macroContentCode` is a pre-cached conversion of `macroContent` for the current `vCodeTable`. When `vCodeTable` changes, `onTableCodeChange()` must be called to rebuild these codes.

### 4.9 Convert Tool (`ConvertTool.h/.cpp`)

Clipboard-based encoding conversion. Supports converting between all 5 code tables.

| Variable | Type | Description |
|---|---|---|
| `convertToolFromCode` | `Uint8` | Source encoding (0-4) |
| `convertToolToCode` | `Uint8` | Target encoding (0-4) |
| `convertToolHotKey` | `int` | Hotkey for quick convert |
| `convertToolDontAlertWhenCompleted` | `bool` | Suppress completion dialog |
| `convertToolToAllCaps` | `bool` | Convert to ALL CAPS |
| `convertToolToAllNonCaps` | `bool` | Convert to all lowercase |
| `convertToolToCapsFirstLetter` | `bool` | Capitalize first letter |
| `convertToolToCapsEachWord` | `bool` | Capitalize each word |
| `convertToolRemoveMark` | `bool` | Strip tone marks |

### 4.10 Smart Switch Key (`SmartSwitchKey.h/.cpp`)

Remembers the input method (language bit 0, code table bits 1+) per application:

| Function | Purpose |
|---|---|
| `initSmartSwitchKey(const Byte* pData, const int& size)` | Load smart-switch data from binary blob |
| `getSmartSwitchKeySaveData(vector<Byte>& outData)` | Serialize all smart-switch entries |
| `getAppInputMethodStatus(const string& bundleId, const int& currentInputMethod)` | Returns -1 (not found), 0 (English), or 1 (Vietnamese). Sets `currentInputMethod` if first visit. |
| `setAppInputMethodStatus(const string& bundleId, const int& language)` | Set preferred language for an app |

---
## 5. macOS Platform Layer (`Sources/OpenKey/macOS/ModernKey/`)

### 5.1 Key Hook Mechanism

The macOS app uses **CGEventTap** (not `NSEvent` global monitor) to intercept ALL keyboard events system-wide before any application receives them:

```
Keyboard press
    ↓
CGEventTapCreate(kCGSessionEventTap, kCGHeadInsertEventTap, ...)
    ↓
OpenKeyCallback(CGEventTapProxy, CGEventType, CGEventRef, void*)
    ↓  [Calls vKeyHandleEvent()]
    ↓  [Checks HookState.code]
    ↓
Return NULL     →  swallow event (OpenKey processed it)
Return event    →  pass through unmodified
```

The event tap captures: `kCGEventKeyDown`, `kCGEventKeyUp`, `kCGEventFlagsChanged`, `kCGEventLeftMouseDown`, `kCGEventRightMouseDown`, `kCGEventLeftMouseDragged`, `kCGEventRightMouseDragged`.

### 5.2 Main Bridge: `OpenKey.mm` (795 lines)

This is the most critical macOS file. Key patterns:

- **Keycode translation:** `keyStringToKeyCodeMap` NSDictionary converts CGKeyCode → internal engine key codes.
- **Special handling for Chromium browsers:** Uses SendShiftAndLeftArrow trick instead of backspace to avoid triggering address bar autocomplete (controlled by `vFixChromiumBrowser`).
- **Unicode Compound workaround:** Apple apps (`com.apple.*`), Chrome, Brave, and Edge are blacklisted from receiving Unicode Compound characters (`_unicodeCompoundApp` array).
- **Sublime Text fix:** Sublime Text 2/3 gets special empty-character handling (`_niceSpaceApp` array).
- **Spotlight workaround:** Spotlight (`com.apple.Spotlight`) has `vFixRecommendBrowser` disabled (`_recommendWorkaroundDisabledApp`).
- **Event posting:** Characters are sent via `CGEventCreateKeyboardEvent()` + `CGEventTapPostEvent()`.
- **VNI double-byte tracking:** `_syncKey` vector tracks 2-code-unit sequences for correct backspace on delete.

### 5.3 AppDelegate (`AppDelegate.m`, 572 lines)

Manages: input method switching (Telex/VNI/SimpleTelex1/SimpleTelex2), code table selection (all 5 tables), modern orthography toggle, macro/convert dialogs, run-at-startup, gray menu bar icon (dark mode), dock icon show/hide (`vShowIconOnDock`), NSUserDefaults persistence. Also handles multi-user support by checking `uid_t` before enforcing single-instance.

### 5.4 OpenKeyManager (`OpenKeyManager.m`, 250 lines)

- `initEventTap` / `stopEventTap`: Setup/teardown CGEventTap + CFRunLoop
- `getTableCodes`: Returns 5-element array: Unicode, TCVN3(ABC), VNI Windows, Unicode tổ hợp, Vietnamese Locale CP 1258
- `checkNewVersion`: Fetches `version.json` from GitHub, compares `latestVersion.versionCode` with bundle version, prompts update
- `launchUpdateHelper`: Copies and launches bundled `OpenKeyUpdate.app` helper, then terminates main app
- `quickConvert`: Clipboard-based encoding conversion (HTML + plaintext)

### 5.5 Additional macOS Controllers

- **MacroViewController** (`MacroViewController.h/.mm`): UI for managing macro/abbreviation entries
- **ConvertToolViewController** (`ConvertToolViewController.h/.mm`): UI for the encoding conversion tool
- **MyTextField** (`MyTextField.h/.m`): Custom NSTextField subclass for handling key events in settings
- **MJAccessibilityUtils** (`MJAccessibilityUtils.h/.m`): Utility for checking/requesting Accessibility permissions

### 5.6 OpenKeyHelper

A minimal companion app launched at login. Handles auto-update replacement of the main app bundle.

---

## 6. Windows Platform Layer (`Sources/OpenKey/win32/OpenKey/OpenKey/`)

### 6.1 Key Hook Mechanism

Windows uses a **low-level keyboard hook** (`WH_KEYBOARD_LL`) via `SetWindowsHookEx()`:

```
Keyboard press
    ↓
LowLevelKeyboardProc (WH_KEYBOARD_LL)
    ↓  [Calls vKeyHandleEvent()]
    ↓  [Checks HookState.code]
    ↓
Return 1   →  swallow event
Return CallNextHookEx(...) → pass through
```

### 6.2 AppDelegate (`AppDelegate.cpp`, 306 lines)

Singleton application controller. Defines all `extern int` engine globals (with different defaults from macOS — see §4.2). Manages:
- Single-instance enforcement (`FindWindow`)
- System tray icon (`SystemTrayHelper`)
- Dialog lifecycle (main control, macro, convert tool, about)
- Settings persistence via Windows Registry (`OpenKeyHelper::setRegInt/getRegInt`)
- Auto-update checking (fetches `version.json`, compares `latestWinVersion.versionCode`)
- `loadDefaultConfig()`: Resets all settings to factory defaults

### 6.3 OpenKeyManager (`OpenKeyManager.h/.cpp`)

- `initEngine()` / `freeEngine()`: Initialize/free the engine
- `getInputType()`: Returns 3-element list: Telex, VNI, Simple Telex (note: UI shows 3, but engine supports 4)
- `getTableCode()`: Returns 5-element list: Unicode, TCVN3(ABC), VNI Windows, Unicode Tổ hợp, Vietnamese Locale CP 1258
- `checkUpdate()`: Fetches version.json, parses `latestWinVersion`, compares with current version

### 6.4 OpenKeyHelper (`OpenKeyHelper.h/.cpp`)

- **Registry access:** Persists settings under `HKCU\Software\TuyenMai\OpenKey`
- **Clipboard conversion:** Reads HTML/RTF/Unicode from clipboard, runs `convertUtil()`, writes back
- **Update check:** Downloads version.json, shows MessageBox prompt, launches `OpenKeyUpdate.exe`
- **Run on startup:** Registers in `HKCU\...\Run` or via Task Scheduler (admin mode)
- **Frontmost app detection:** For smart-switch feature (`getFrontMostAppExecuteName()`, `getLastAppExecuteName()`)

### 6.5 UI Dialogs

All UI is classic Win32 dialogs (resource-based):

| Dialog | File | Purpose |
|---|---|---|
| MainControlDialog | `MainControlDialog.h/.cpp` | Settings panel: input type, code table, all toggles |
| MacroDialog | `MacroDialog.h/.cpp` | Text expansion table CRUD |
| ConvertToolDialog | `ConvertToolDialog.h/.cpp` | Encoding conversion with options |
| BaseDialog | `BaseDialog.h/.cpp` | Base Win32 dialog class |
| AboutDialog | `AboutDialog.h/.cpp` | Version info + links |

---
## 7. Build System

### macOS
- **Xcode project:** `Sources/OpenKey/macOS/OpenKey.xcodeproj`
- **Targets:** OpenKey (main app), OpenKeyHelper (login helper), OpenKeyUpdate (update helper)
- **Requirements:** macOS Mojave+, Xcode 10+
- **Build:** Product → Archive → Distribute App

### Windows
- **Visual Studio solution:** `Sources/OpenKey/win32/OpenKey/OpenKey.sln` / `OpenKey.vcxproj`
- Uses Win32 API, Precompiled Headers (`stdafx.h`, `stdafx.cpp`, `targetver.h`)

---

## 8. Data Flow (Keystroke Processing)

```
1. User presses a key
2. OS hook captures event (CGEventTap on macOS, WH_KEYBOARD_LL on Windows)
3. Key code translated to engine-internal format
4. vKeyHandleEvent(Keyboard, KeyDown, keyCode, capsStatus, otherControlKey) called
5. Engine processes:
   a. If English mode → check for macro expansion
   b. If Vietnamese mode:
      - Determine if key is a "special key" (tone mark, vowel modifier) via IS_SPECIALKEY macro
      - Append to TypingWord[] buffer
      - Apply Vietnamese tone rules (Vietnamese.cpp vowel tables)
      - Check spelling against consonant/vowel patterns (checkSpelling function)
      - Handle Quick Telex shortcuts (IS_QUICK_TELEX_KEY → handleQuickTelex)
      - Build replacement character sequence via getCharacterCode()
   c. Set HookState.code and populate HookState.charData[]
6. Platform reads HookState:
   a. vDoNothing → pass event through
   b. vWillProcess → emit backspaces + replacement chars, swallow event
   c. vRestore → emit original key (invalid word), start fresh
   d. vReplaceMaro → emit macro expansion text (emit backspaces for trigger + expansion chars)
   e. vBreakWord → start new word
7. Character codes translated from internal → platform (Unicode, VNI, TCVN3, etc.)
```

---

## 9. Key Technical Patterns

### 9.1 Cross-Platform Abstraction

Platform-specific headers (`mac.h`, `win32.h`, `linux.h`) define the same `KEY_*` constants but with platform-appropriate values. The engine code uses only these constants, never raw OS key codes. The `DataType.h` file auto-includes the correct platform header:

```cpp
#ifdef LINUX
#include "platforms/linux.h"
#elif _WIN32
#include "platforms/win32.h"
#else
#include "platforms/mac.h"
#endif
```

### 9.2 Backspace-Rewrite Technique

Instead of composing characters during typing (which causes visible jitter), OpenKey uses a post-hoc approach:
1. Let the OS render the raw keystrokes first
2. When a tone mark or modifier key arrives, simulate backspaces to remove the raw text
3. Type the correctly-composed Vietnamese word

This avoids the "underline" issue common in macOS's built-in Vietnamese IME.

### 9.3 Chromium/Excel Autocomplete Fix

Chrome and Edge aggressively autocomplete in address bars. To prevent composed characters from being mangled, OpenKey uses one of two techniques:
- **Send shift+left-arrow** before backspacing (newer `vFixChromiumBrowser` method)
- **Send an invisible zero-width character** before backspacing (older `vFixRecommendBrowser` method)

### 9.4 VNI Double-Byte Handling

VNI Windows (code table 2) and Unicode Compound (code table 3) use 2 code units per character (like UTF-16 surrogates). The platform layer maintains a `_syncKey` vector to track double-byte sequences for correct backspace count on delete. The `IS_DOUBLE_CODE(code)` macro checks for these code tables.

### 9.5 State Save/Restore System

The engine maintains a `_typingStates` vector and `_longWordHelper` vector for state management:
- `insertState()`: Saves current TypingWord state before processing
- `saveWord()`: Archives a completed word for undo/restore
- `restoreLastTypingState()`: Recovers previous state on backspace
- This enables features like auto-mark-recovery when deleting characters

### 9.6 Multi-User Support (macOS)

On macOS, the single-instance check verifies the UID (`proc_pidinfo` with `PROC_PIDTBSDINFO`) to allow multiple users to each run their own instance of OpenKey simultaneously.

---

## 10. Configuration Persistence

| Platform | Storage |
|---|---|
| macOS | `NSUserDefaults` (keyed by `@"vLanguage"`, `@"vInputType"`, etc.) |
| Windows | Registry: `HKCU\Software\TuyenMai\OpenKey` |
| Macro data | macOS: `~/Library/Application Support/OpenKey/openkey.data`; Windows: binary file alongside executable |
| Smart Switch data | Same path as macro data (binary blob) |

---

## 11. Common Modification Scenarios

### Adding a new feature toggle (e.g., "Auto-spacing")

1. **`Engine.h`**: Declare `extern int vAutoSpacing;`
2. **`Engine.cpp`**: Reference it in the processing logic
3. **`AppDelegate.m`** (macOS): Define `int vAutoSpacing = 0;` with default, add `LOAD_DATA` in `OpenKeyInit()`, persist via `NSUserDefaults`
4. **`AppDelegate.cpp`** (Win32): Define `int vAutoSpacing = 0;` with default, add to registry save/load
5. **UI** (both platforms): Add checkbox, wire to persist/load

### Adding a new code table

1. **`Vietnamese.h`**: Ensure your encoding's character mapping exists
2. **`Vietnamese.cpp`**: Add to `_codeTable[]` array at new index
3. **Platform UI**: Add entry to code table dropdowns (`getTableCodes` on macOS, `getTableCode` on Windows)
4. **`ConvertTool.cpp`**: Add conversion to/from new code table

### Adding a new input method

1. **`DataType.h`**: Add enum value to `vKeyInputType` (e.g., `vNewMethod = 4`)
2. **`Engine.cpp`**: Add row to `ProcessingChar[][]` table
3. **`IS_SPECIALKEY` macro** in `DataType.h`: Add the new vInputType case with appropriate key patterns
4. **UI** (both platforms): Add to input type selectors

### Porting to Linux

1. `platforms/linux.h` keycode mapping is already present
2. Implement keyboard hook (e.g., X11 `XRecord` extension or `libinput`)
3. Implement UI (GTK/Qt)
4. Wire the hook → `vKeyHandleEvent()` → emit X11 key events

---

## 12. Debugging Tips

- **`#define IS_DEBUG 1`** is already set in `Engine.h` (line 21). This enables verbose console logging of the TypingWord buffer and tone processing steps. Comment out or set to 0 for release builds.
- On macOS, use `CGEventTap` diagnostic tools or Console.app. The event tap must have Accessibility permissions.
- On Windows, use `DebugView` to capture `OutputDebugString` messages.
- The `vKeyHookState.extCode` field carries metadata about the kind of processing that occurred — inspect it when keyboard output doesn't match expectations.

---

## 13. Dependencies

The engine has **zero external dependencies** — it uses only the C++ standard library (`<vector>`, `<map>`, `<string>`, `<locale>`, `<codecvt>`, `<algorithm>`, `<iostream>`).

Platform layers use only native OS APIs:
- **macOS**: Cocoa, CoreGraphics (CGEvent), ApplicationServices, Carbon, ServiceManagement, libproc
- **Windows**: Win32 API, COM (UrlMon for update downloads), Shell32

---
## 14. Engine Macros and Definitions Reference

### Engine Functions (alphabetical)

| Function | File | Purpose |
|---|---|---|
| `checkGrammar()` | Engine.cpp | Validate and auto-correct tone placement / spelling |
| `checkSpelling()` | Engine.cpp | Full Vietnamese spelling validation (consonant + vowel + end consonant) |
| `findAndCalculateVowel()` | Engine.cpp | Determine vowel range (start/end index) in current word |
| `getCharacterCode()` | Engine.cpp | Convert internal code to output character using current code table |
| `handleMainKey()` | Engine.cpp | Core tone-mark / vowel-modifier processing |
| `handleQuickTelex()` | Engine.cpp | Quick Telex shortcut expansion |
| `insertKey()` | Engine.cpp | Append key to TypingWord buffer |
| `startNewSession()` | Engine.cpp | Reset word buffer on space/click/break |
| `vEnglishMode()` | Engine.cpp | Handle key in English mode (macro only) |
| `vKeyHandleEvent()` | Engine.cpp | ★ Main keystroke entry point |
| `vKeyInit()` | Engine.cpp | Initialize engine, alloc buffers, returns &HookState |
| `vSetCheckSpelling()` | Engine.cpp | Refresh spelling state after toggle |
| `vTempOffEngine()` | Engine.cpp | Temporarily bypass engine (Cmd/Alt key) |
| `vTempOffSpellChecking()` | Engine.cpp | Skip spell check for current word (Ctrl) |

### Key Data Structures

| Struct | Defined In | Purpose |
|---|---|---|
| `vKeyHookState` | DataType.h | Result from engine → platform: what to emit |
| `MacroData` | Macro.h | One text-expansion entry (key + content + cached codes) |

### Key Enums

| Enum | Values |
|---|---|
| `vKeyEvent` | Keyboard, Mouse |
| `vKeyEventState` | KeyDown, KeyUp, MouseDown, MouseUp |
| `vKeyInputType` | vTelex(0), vVNI(1), vSimpleTelex1(2), vSimpleTelex2(3) |
| `HoolCodeState` | vDoNothing(0), vWillProcess(1), vBreakWord(2), vRestore(3), vReplaceMaro(4), vRestoreAndStartNewSession(5) |

### Key Engine Macros (in `Engine.cpp`)

| Macro | Definition |
|---|---|
| `IS_KEY_S(key)` | Sắc tone mark in current input method |
| `IS_KEY_F(key)` | Huyền tone mark in current input method |
| `IS_KEY_R(key)` | Hỏi tone mark in current input method |
| `IS_KEY_X(key)` | Ngã tone mark in current input method |
| `IS_KEY_J(key)` | Nặng tone mark in current input method |
| `IS_KEY_DOUBLE(key)` | Double-letter modifier (A, O, E) in current input method |
| `IS_KEY_D(key)` | Đ key in current input method |
| `IS_KEY_W(key)` | W / Ư/Ơ key in current input method |
| `IS_KEY_Z(key)` | Z (remove mark) key in current input method |
| `IS_MARK_KEY(keyCode)` | Any tone mark key in current input method |
| `IS_BRACKET_KEY(key)` | `key == KEY_LEFT_BRACKET || key == KEY_RIGHT_BRACKET` |
| `IS_QUICK_TELEX_KEY(code)` | Quick Telex double-letter trigger (cc, gg, kk, nn, qq, pp, tt) |
| `IS_NUMBER_KEY(code)` | 0-9 digit key |
| `IS_SPECIALKEY(keyCode)` | Any special processing key in current input method |
| `IS_DOUBLE_CODE(code)` | Code table uses double-byte encoding (2=VNI, 3=UniCompound) |

### Key Files (with verified line counts)

| File | Lines | Importance | Description |
|---|---|---|---|
| `Engine.cpp` | 1558 | ★★★★★ | Main processing logic, tone rules application |
| `OpenKey.mm` | 795 | ★★★★★ | macOS bridge: CGEventTap → Engine → CGEventPost |
| `Vietnamese.cpp` | 576 | ★★★★★ | All vowel/consonant tables, code table mappings |
| `AppDelegate.m` (macOS) | 572 | ★★★★ | macOS global state, menu bar, dialog management |
| `AppDelegate.cpp` (Win32) | 306 | ★★★★ | Windows: global state, dialog orchestration |
| `Macro.cpp` | 293 | ★★★ | Text expansion engine |
| `OpenKeyManager.m` (macOS) | 250 | ★★★ | Event tap lifecycle, update checking |
| `Engine.h` | 245 | ★★★★ | Public API, all `extern` config variables |
| `ConvertTool.cpp` | 180 | ★★★ | Encoding conversion logic |
| `DataType.h` | 156 | ★★★★ | Core types, platform import, bitmask macros |
| `SmartSwitchKey.cpp` | 73 | ★★★ | Per-app input method memory |

---

## 15. Quick Reference: Platform Differences

| Feature | macOS | Windows |
|---|---|---|
| Hook mechanism | CGEventTap (kCGSessionEventTap) | SetWindowsHookEx (WH_KEYBOARD_LL) |
| Global vars defined in | AppDelegate.m | AppDelegate.cpp |
| Default `vSendKeyStepByStep` | 0 (batch mode) | 1 (step-by-step) |
| Default `vRestoreIfWrongSpelling` | 0 | 1 |
| Default `vFixRecommendBrowser` | 1 | 0 |
| Default switch key | 0x7A000206 (Option+Z) | 0x5A00025A (Alt+Z) |
| Persistence | NSUserDefaults | Registry (HKCU) |
| Macro data path | `~/Library/Application Support/OpenKey/` | Next to executable |
| App filtering | Bundle IDs (com.apple.*, etc.) | Executable names |
| Single-instance check | By UID (multi-user aware) | FindWindow |
| Settings UI style | Cocoa windows | Win32 resource dialogs |

---

## 16. References

- **Homepage**: [open-key.org](http://open-key.org)
- **Upstream GitHub**: [github.com/tuyenvm/OpenKey](https://github.com/tuyenvm/OpenKey)
- **This Fork**: [github.com/anhnt/OpenKey](https://github.com/anhnt/OpenKey)
- **Facebook**: [OpenKeyVN](https://www.facebook.com/OpenKeyVN)
- **Releases**: [github.com/tuyenvm/OpenKey/releases](https://github.com/tuyenvm/OpenKey/releases)
