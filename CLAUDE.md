# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

AI Memory Reader is a native macOS/iOS app that provides a purpose-built viewer for AI agent memory files: `CLAUDE.md`, `AGENTS.md`, daily memory entries, and `*.jsonl` session transcripts. It auto-discovers memory directories for 8 AI tools and chunk-renders multi-MB JSONL files that crash editors like VSCode.

**Platform:** macOS 15.0+ (full features) and iOS 17.0+ (read-only companion).  
**Distribution:** Free download (universal binary, ad-hoc signed) + GPL-3.0 source.  
**Language:** Swift 6.0 / SwiftUI.  
**Project generation:** XcodeGen (`project.yml` → `.xcodeproj`).

## Repository Structure

```
ai-memory-reader/
├── project.yml                          # XcodeGen project definition (source of truth for targets)
├── AIMemoryReader.entitlements          # macOS entitlements (sandbox, file access)
├── AIMemoryReader-iOS.entitlements      # iOS entitlements
├── AIMemoryReader/
│   └── Sources/
│       ├── App/
│       │   └── AIMemoryReaderApp.swift  # SwiftUI App entry point, scene setup
│       ├── Models/
│       │   ├── AppState.swift           # Observable root state (selected source, file, search)
│       │   ├── AppTheme.swift           # Light/dark/system theme enum
│       │   ├── AISource.swift           # AI tool source model (path, display name, file filter)
│       │   ├── FileNode.swift           # Tree node for the sidebar (file/directory)
│       │   └── SettingsStore.swift      # NSUbiquitousKeyValueStore + UserDefaults settings sync
│       ├── Views/
│       │   ├── ContentView.swift        # Root NavigationSplitView (sidebar + detail)
│       │   ├── SidebarView.swift        # File tree sidebar with source picker
│       │   ├── DetailView.swift         # File content pane (renders markdown or JSONL)
│       │   ├── FindableReadableView.swift # Markdown reader with in-file search
│       │   ├── MarkdownEditorView.swift # NSTextView-based editor with syntax highlighting
│       │   ├── TOCView.swift            # Right sidebar table of contents (click-to-jump)
│       │   ├── FileFindBar.swift        # Find bar UI component
│       │   └── UpdateBanner.swift       # Non-blocking update notification banner
│       ├── Utilities/
│       │   ├── ReadableMarkdownRenderer.swift   # MarkdownUI render configuration (GitHub theme)
│       │   ├── MemoryReaderTheme.swift          # Custom MarkdownUI theme tokens
│       │   ├── MemoryFileMatcher.swift          # Filters file tree to known AI memory files
│       │   ├── FileTreeBuilder.swift            # Builds FileNode tree from a directory
│       │   ├── FileWatcher.swift                # FSEvents-based live reload
│       │   ├── SearchService.swift              # Full-text search across all files
│       │   ├── SplashCodeSyntaxHighlighter.swift # Code block syntax highlighting
│       │   ├── PDFExporter.swift                # NSPrintOperation → PDF (macOS only)
│       │   ├── LocalImageProvider.swift         # Resolves local image paths in markdown
│       │   ├── BookmarkStore.swift              # Security-scoped bookmark persistence
│       │   ├── UpdateChecker.swift              # GitHub Releases version check (max 1/day)
│       │   └── WindowChrome.swift               # Window title bar customisation (macOS)
│       └── Resources/
│           ├── Info.plist
│           ├── PrivacyInfo.xcprivacy
│           ├── AIMemoryReader.entitlements
│           └── AIMemoryReader-iOS.entitlements
├── docs/                                # Documentation assets
├── llms.txt                             # Machine-readable summary for AI agents
├── CLAUDE_CLOUD_MEMORY_SPEC.md          # Spec for cloud/sync memory features
├── PLAN.md                              # V1 development log
├── V2-PLAN.md                           # V2 development log
├── V3-PLAN.md                           # V3 plan and development log
└── aimr                                 # CLI wrapper script (shell) for URL-scheme invocation
```

## Build and Run

### Prerequisites

- macOS 15.0+
- Xcode 16.0+
- Swift 6.0
- XcodeGen: `brew install xcodegen`

### Steps

```bash
git clone https://github.com/nvwalj/ai-memory-reader.git
cd ai-memory-reader
xcodegen generate          # regenerates AIMemoryReader.xcodeproj from project.yml
open AIMemoryReader.xcodeproj
# Press ⌘R to build and run
```

**Important:** `AIMemoryReader.xcodeproj` is generated — never edit it directly. All target configuration lives in `project.yml`. After changing `project.yml`, re-run `xcodegen generate`.

### CLI Setup (optional)

```bash
cp aimr /usr/local/bin/
chmod +x /usr/local/bin/aimr
# Usage:
aimr open ~/.claude/CLAUDE.md
aimr open ~/.claude/CLAUDE.md --heading "Conventions"
```

## Tech Stack

| Concern | Technology |
|---------|-----------|
| UI framework | SwiftUI (NavigationSplitView on Mac, NavigationStack on iPhone) |
| Markdown rendering | [MarkdownUI](https://github.com/gonzalezreal/swift-markdown-ui) — GitHub theme |
| Editor | NSTextView with custom syntax highlighting |
| State management | `@Observable` macro (Swift 5.9+ / Swift 6 observation) |
| File watching | FSEvents via `FileWatcher.swift` |
| Settings sync | NSUbiquitousKeyValueStore (iCloud) + UserDefaults fallback |
| PDF export | NSPrintOperation (macOS only, `#if os(macOS)`) |
| Project generation | XcodeGen (`project.yml`) |
| Package management | Swift Package Manager |

## Supported AI Sources

| Source | Directory | Key Files |
|--------|-----------|----------|
| Claude Code | `~/.claude/` | CLAUDE.md, memory/*.md, projects/**/*.jsonl |
| Codex | `~/.codex/` | AGENTS.md, memories/*.md, sessions/**/*.jsonl |
| Gemini | `~/.gemini/` | GEMINI.md |
| Cursor | `~/.cursor/` | rules/*.mdc |
| Continue | `~/.continue/` | config.json, config.yaml, rules/*.md |
| GitHub Copilot | `~/.config/github-copilot/` | copilot-instructions.md |
| Aider | `~/.aider/` | .aider.conf.yml, CONVENTIONS.md |
| OpenClaw | `~/.openclaw/workspace/` | MEMORY.md, SOUL.md, AGENTS.md, memory/*.md |

## Architecture

### State Flow

`AppState` (Observable) is the single root state object injected at the top level. It holds:
- Selected `AISource`
- Selected `FileNode`
- Current search query
- Theme preference

### File Tree

`FileTreeBuilder` builds a `FileNode` tree from a root directory. `MemoryFileMatcher` filters nodes to known AI memory file extensions (`.md`, `.mdx`, `.mdc`, `.jsonl`, known config filenames). "Show All JSON/JSONL Files" toggle bypasses the filter.

### Rendering

- `.md` / `.mdc` / `.mdx` files → `FindableReadableView` wrapping `ReadableMarkdownRenderer` (MarkdownUI)
- `.jsonl` / `.json` files → chunked code blocks in `DetailView` to avoid memory issues on large session logs
- Edit mode → `MarkdownEditorView` (NSTextView wrapper with syntax highlighting)

### Settings Sync (iCloud)

`SettingsStore` wraps `NSUbiquitousKeyValueStore` with a `UserDefaults` fallback. On first launch it migrates existing UserDefaults values to iCloud. All writes are double-written to both stores. Requires the `com.apple.developer.ubiquity-kvstore-identifier` entitlement (present in entitlements files; activate in Xcode's Signing & Capabilities tab).

### URL Scheme

`aimemoryreader://open?path=/path/to/file.md&heading=Heading`

Handled in `AIMemoryReaderApp.swift` via `.onOpenURL`. The `aimr` CLI script is a thin wrapper that opens this URL scheme.

### Update Checker

`UpdateChecker` hits the GitHub Releases API once per 24 hours. It compares the latest release tag against the bundle's `CFBundleShortVersionString`. On newer version, shows `UpdateBanner` (non-blocking). Suppressed on Mac App Store builds.

## Platform Differences

| Feature | macOS | iOS |
|---------|-------|-----|
| Edit mode | Yes (NSTextView) | No (read-only) |
| TOC sidebar | Yes | No |
| PDF export | Yes (⌘P) | No |
| iCloud settings sync | Yes (with entitlement) | Yes |
| File access | Security-scoped bookmarks | Files app integration |

Use `#if os(macOS)` / `#if os(iOS)` guards for platform-specific code. `PDFExporter.swift` is already macOS-only.

## Key Conventions

- **No network calls at runtime** except the daily GitHub-releases update check (skippable in settings)
- **`project.yml` is source of truth** for Xcode targets — never edit `.xcodeproj` manually
- **`@Observable`** macro preferred over `ObservableObject`/`@Published` for all state objects
- **Chunked JSONL rendering** — large files are split into pages to prevent OOM crashes; don't attempt to load full file into a single Text view
- **Memory file filter is strict by default** — only show known AI memory filenames; power-user override via View menu
- **Auto-save** in edit mode: 2-second debounce after last keystroke, then write

## Versioning and Plans

- `PLAN.md` — V1 design decisions and development log
- `V2-PLAN.md` — V2 additions (iOS, edit mode, JSONL viewer)
- `V3-PLAN.md` — V3 additions (PDF export, iCloud settings sync, iPhone verification) — all completed

Next planned features: MCP server integration (not started), multi-window support (not planned for V3).

## Privacy

- `PrivacyInfo.xcprivacy` declares that the app accesses the file system under user direction
- Zero telemetry or analytics
- The daily update check is the only outbound network request; it can be disabled
