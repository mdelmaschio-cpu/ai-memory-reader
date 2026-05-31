# AI Memory Reader — Codebase Guide

This file documents the project structure, build setup, architecture, and conventions for AI assistants working in this codebase.

## Project Overview

AI Memory Reader (AIMR) is a native macOS + iOS app that serves as a unified viewer for every AI agent memory file on disk. It auto-discovers memory directories for 8 supported agents (Claude Code, Codex, Cursor, Gemini, Continue, GitHub Copilot, Aider, OpenClaw), watches files live, and chunk-renders multi-MB JSONL session telemetry that crashes standard editors.

**App ID:** `com.aitools.ai-memory-reader` (macOS) / `com.aitools.ai-memory-reader-ios` (iOS)
**Current version:** 0.4.6 (build 9 macOS, build 7 iOS)
**License:** GPL-3.0

### Positioning

- macOS: full-featured viewer + editor with sidebar, TOC, file watching, and full-text search.
- iPhone: read-only companion with native navigation and Files app integration.
- The app makes **zero network calls** except for one daily GitHub-releases update check (skippable; App Store builds skip it entirely).

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Swift 6.0 (strict concurrency enabled) |
| UI framework | SwiftUI — `NavigationSplitView` on macOS, `NavigationStack` on iPhone |
| Markdown rendering | [MarkdownUI](https://github.com/gonzalezreal/swift-markdown-ui) 2.4+ (GitHub theme) |
| Syntax highlighting | [Splash](https://github.com/JohnSundell/Splash) 0.16+ |
| Text editing | `NSTextView` with custom syntax highlighting |
| State management | `@Observable` macro (Swift 5.9+) |
| File watching | FSEvents |
| Project generation | XcodeGen 2.40+ + SPM |
| Deployment targets | macOS 15.0+, iOS 17.0+ |
| Xcode version | 16.0+ |

## Repository Structure

```
ai-memory-reader/
├── project.yml                         # XcodeGen project definition
├── AIMemoryReader.xcodeproj/           # Generated Xcode project (do not edit directly)
│
├── AIMemoryReader/                     # Shared Swift sources
│   ├── AIMemoryReader.entitlements     # macOS sandbox entitlements (MAS release)
│   └── Sources/
│       ├── App/                        # App entry points and top-level coordinators
│       ├── Models/                     # Data models (AISource, file entries, etc.)
│       ├── Views/                      # SwiftUI views (sidebar, content, TOC, etc.)
│       ├── Utilities/                  # Helpers (file watching, JSONL chunker, search)
│       └── Resources/
│           ├── Assets.xcassets         # App icon, colors
│           ├── Info.plist
│           └── PrivacyInfo.xcprivacy   # Required for App Store submissions
│
├── AIMemoryReader-iOS.entitlements     # iOS-specific entitlements
│
├── aimr                                # CLI bridge script (bash) for URL scheme / headings
├── llms.txt                            # Machine-readable summary for AI agents
│
├── docs/                               # Additional documentation
├── CLAUDE_CLOUD_MEMORY_SPEC.md         # Feature spec: claude.ai cloud memory sync (v1/v2)
├── PLAN.md                             # Development roadmap
├── V2-PLAN.md                          # V2 feature planning
├── V3-PLAN.md                          # V3 feature planning
├── MAS_METADATA.md                     # Mac App Store metadata drafts
├── OUTREACH_DRAFTS.md                  # Marketing/outreach copy
├── README.md                           # User-facing README (English)
├── README_CN.md                        # User-facing README (Chinese)
└── LICENSING.md                        # Licensing details
```

### Source Directory Conventions

- `Sources/App/` — app entry points (`@main` struct, scene setup, URL scheme handler, update checker).
- `Sources/Models/` — plain Swift structs/classes: `AISource` (enum of 8 supported agents), file-tree models, JSONL chunk models.
- `Sources/Views/` — SwiftUI views only; no business logic. Each view file corresponds to one screen or reusable component.
- `Sources/Utilities/` — stateless helpers and services: FSEvents watcher, JSONL chunked renderer, full-text search index, CLI bridge.
- `Sources/Resources/` — asset catalogs, `Info.plist`, privacy manifest. No Swift files here.

## Supported AI Agents and Discovery Paths

| Agent | Root Directory | Key Files |
|---|---|---|
| Claude Code | `~/.claude/` | `CLAUDE.md`, `memory/*.md`, `projects/**/*.jsonl` |
| Codex | `~/.codex/` | `AGENTS.md`, `memories/*.md`, `sessions/**/*.jsonl` |
| Gemini | `~/.gemini/` | `GEMINI.md` |
| Cursor | `~/.cursor/` | `rules/*.mdc` |
| Continue | `~/.continue/` | `config.json`, `config.yaml`, `rules/*.md` |
| GitHub Copilot | `~/.config/github-copilot/` | `copilot-instructions.md` |
| Aider | `~/.aider/` | `.aider.conf.yml`, `CONVENTIONS.md` |
| OpenClaw | `~/.openclaw/workspace/` | `MEMORY.md`, `SOUL.md`, `AGENTS.md`, `memory/*.md` |

The app also supports opening any local folder or individual `.md` / `.json` / `.jsonl` file.

## Build and Development Setup

### Prerequisites

- macOS 15.0+
- Xcode 16.0+
- [XcodeGen](https://github.com/yonaskolb/XcodeGen) 2.40+
- Swift 6.0 (bundled with Xcode 16)

### First-time Setup

```bash
git clone https://github.com/nvwalj/ai-memory-reader.git
cd ai-memory-reader
brew install xcodegen   # if not already installed
xcodegen generate       # regenerates AIMemoryReader.xcodeproj from project.yml
open AIMemoryReader.xcodeproj
# Press Cmd+R to build and run (macOS target)
```

### Regenerating the Xcode Project

The `.xcodeproj` is generated from `project.yml` via XcodeGen. Run `xcodegen generate` after any change to `project.yml`. Do **not** edit `.xcodeproj` files directly — those changes will be lost on the next generation.

### Build Configurations

| Config | Entitlements | Notes |
|---|---|---|
| Debug | None (no sandbox) | Day-to-day development; behaves like the GitHub release zip |
| Release | `AIMemoryReader.entitlements` | Used for MAS submission; App Sandbox enabled |

The `SWIFT_STRICT_CONCURRENCY: complete` flag is active in both targets. All Swift concurrency warnings are treated as errors.

### CLI Tool Setup

```bash
cp aimr /usr/local/bin/
chmod +x /usr/local/bin/aimr
```

Usage:
```bash
aimr open ~/.claude/CLAUDE.md
aimr open ~/.claude/CLAUDE.md --heading "Architecture"
```

## Key Architectural Decisions

### Shared Sources for macOS and iOS

Both the macOS and iOS targets compile from the same `AIMemoryReader/Sources/` tree. Platform differences are handled via `#if os(macOS)` / `#if os(iOS)` conditional compilation. iOS is read-only — the edit mode (`NSTextView`-backed) is macOS-only.

### @Observable State Management

The app uses Swift 5.9's `@Observable` macro instead of `ObservableObject` / `@Published`. The main app state object (likely `AppState`) is injected via the environment and observed directly. Avoid introducing `ObservableObject` for new code.

### JSONL Chunked Rendering

Claude Code and Codex produce JSONL session telemetry files that can exceed tens of MB. The app renders these in chunks (paginated code blocks) rather than loading the full string into a `Text` view, which crashes SwiftUI on large files. When modifying the JSONL viewer, preserve this chunking approach.

### File Watching via FSEvents

File watching uses the macOS FSEvents API (not `DispatchSource` file descriptors). The watcher should remain low-latency and avoid duplicating events. On iOS the watcher is disabled — the user refreshes manually or via app foreground.

### Memory-file Filter

The file tree shows only known AI memory/config file types by default (Markdown, JSONL, known config filenames). A "Show All JSON/JSONL Files" toggle is available in the View menu. When adding new agent support, update the filter list alongside the `AISource` enum.

### URL Scheme

The app registers the `aimemoryreader://` URL scheme:
```
aimemoryreader://open?path=/path/to/file.md&heading=Heading
```
The CLI (`aimr`) wraps this scheme for use from the terminal and from AI agents.

### Update Checker

On launch (max once per 24 hours), the app queries the GitHub releases API for a newer tag and shows a non-blocking banner. App Store builds skip this entirely. The check must remain opt-outable and must not block app startup.

## Planned Features (see spec files)

### Claude.ai Cloud Memory Sync (CLAUDE_CLOUD_MEMORY_SPEC.md)

V1: read-only display of the user's claude.ai account-level memory (the `saffron` feature) inside the AIMR sidebar. Authentication via embedded `WKWebView` (one-time login; cookies persisted in system keychain). Endpoints:

- `GET /api/organizations/{org_id}/memory` — returns the account memory Markdown blob.
- `GET /api/organizations/{org_id}/memory/settings` — checks if memory is enabled (`enabled_saffron`).
- `GET /api/bootstrap` — discovers `org_id` after login.

New file to create: `Sources/Utilities/CloudMemoryService.swift`.

V2 (later): project-level memory tree, write-back, and bidirectional sync with local `CLAUDE.md`.

## Development Conventions

### Swift / SwiftUI

- Target Swift 6.0 with `SWIFT_STRICT_CONCURRENCY: complete`. All async code must be properly isolated — use `@MainActor` for UI-touching code, `actor` for shared mutable state.
- Use `@Observable` (not `ObservableObject`) for new observable types.
- Views must be pure: no file I/O, no FSEvents, no `URLSession` calls inside a `View` struct. Move side effects to `Utilities/` or `Models/`.
- Keep `#if os(macOS)` / `#if os(iOS)` blocks small and localized. Prefer protocol abstractions or wrapper types when the platform divergence is large.
- Do not import AppKit types into files that also import UIKit. Use `#if canImport(AppKit)` / `#if canImport(UIKit)` guards.

### Project Configuration

- All project changes go through `project.yml` + `xcodegen generate`. Never commit manual `.xcodeproj` edits.
- Bundle identifiers: `com.aitools.ai-memory-reader` (macOS), `com.aitools.ai-memory-reader-ios` (iOS).
- Marketing version follows semver (`MARKETING_VERSION`); build number (`CURRENT_PROJECT_VERSION`) increments monotonically per submission.

### Privacy and Security

- The app is sandboxed in Release builds. Any new file-system access outside the standard read-scope requires an entitlement change in `AIMemoryReader.entitlements`.
- Cloud memory text (if the cloud sync feature ships) must never be persisted to disk unless the user explicitly requests a snapshot. Cookies are stored only in the system keychain via `WKHTTPCookieStore`.
- No analytics on file content, ever.

### Naming

- Swift types: `UpperCamelCase`.
- Swift functions and properties: `lowerCamelCase`.
- Files: one type per file, filename matches the primary type name.
- Avoid abbreviations in public API; `src` → `source`, `mgr` → `manager`.

### Commit Style

- Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`.
- Reference issue numbers where applicable.

## Keyboard Shortcuts (macOS)

| Shortcut | Action |
|---|---|
| Cmd+O | Open file or folder |
| Cmd+E | Toggle edit / read mode |
| Cmd+S | Save (edit mode) |
| Cmd+F | Focus search |
| Cmd+1 | Switch to first detected AI source (Claude Code) |
| Cmd+2 | Open local files |
| Cmd+R | Refresh / refetch cloud memory |

## External Dependencies

| Package | Version | Purpose |
|---|---|---|
| [swift-markdown-ui](https://github.com/gonzalezreal/swift-markdown-ui) | 2.4.0+ | GitHub-style Markdown rendering |
| [Splash](https://github.com/JohnSundell/Splash) | 0.16.0+ | Swift syntax highlighting in code blocks |

Dependencies are managed via Swift Package Manager and declared in `project.yml`. Run `xcodegen generate` after adding a new package to regenerate the Xcode project with the updated SPM graph.
