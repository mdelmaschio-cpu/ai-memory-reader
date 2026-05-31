# CLAUDE.md — ai-memory-reader

This file provides guidance to AI assistants working in this repository.

## Project Overview

**AI Memory Reader** is a native macOS + iOS app (Swift/SwiftUI) for reading, browsing, and editing the memory files that AI coding agents leave on disk: `CLAUDE.md`, `AGENTS.md`, `.jsonl` session transcripts, and similar artifacts. It auto-discovers 8 supported AI tool directories, watches files live, and renders multi-MB JSONL files in chunks without crashing.

- **Platforms**: macOS 15+, iOS 17+
- **Language**: Swift 5.10+, SwiftUI
- **Architecture**: Native universal binary (~3 MB), zero network calls except a daily GitHub-release version check (skippable)
- **License**: GPL v3
- **Distribution**: Mac App Store + direct download

## Repository Layout

```
ai-memory-reader/
├── AIMemoryReader.xcodeproj/         # Xcode project (do not edit manually)
├── project.yml                       # XcodeGen spec — edit this, not the .xcodeproj
├── AIMemoryReader/                   # macOS app target
│   └── Sources/                      # Swift source files
├── AIMemoryReader-iOS.entitlements   # iOS app entitlements
├── AIMemoryReader.entitlements       # macOS app entitlements
├── aimr/                             # CLI companion (aimr open <path>)
├── docs/                             # Documentation
├── llms.txt                          # Machine-readable summary for AI agents
├── icon.png
├── home.png
├── PLAN.md                           # Original MVP plan
├── V2-PLAN.md                        # V2 roadmap
├── V3-PLAN.md                        # V3 roadmap
├── LICENSING.md                      # Dual-license notes
├── MAS_METADATA.md                   # Mac App Store metadata
└── README.md
```

## Build System

The project uses **XcodeGen** to generate the `.xcodeproj` from `project.yml`.

```bash
# Install XcodeGen (once)
brew install xcodegen

# Regenerate .xcodeproj after editing project.yml
xcodegen generate

# Build and run (macOS target)
open AIMemoryReader.xcodeproj
# Select the AIMemoryReader scheme → Run (⌘R)
```

**Never edit `AIMemoryReader.xcodeproj` directly.** All project configuration lives in `project.yml`.

## Supported AI Agent Sources

| AI Tool | Directory | Key Files |
|---------|-----------|-----------|
| Claude Code | `~/.claude/` | `CLAUDE.md`, `memory/*.md`, `projects/**/*.jsonl` |
| Codex | `~/.codex/` | `AGENTS.md`, `memories/*.md`, `sessions/**/*.jsonl` |
| Gemini CLI | `~/.gemini/` | `GEMINI.md` |
| Cursor | `~/.cursor/` | `rules/*.mdc` |
| Continue | `~/.continue/` | `config.json`, `config.yaml`, `rules/*.md` |
| GitHub Copilot | `~/.config/github-copilot/` | `copilot-instructions.md` |
| Aider | `~/.aider/` | `.aider.conf.yml`, `CONVENTIONS.md` |
| OpenClaw | `~/.openclaw/workspace/` | `MEMORY.md`, `SOUL.md`, `AGENTS.md` |

## Core Features

### Reading
- GitHub-style Markdown rendering (MarkdownUI)
- JSONL chunked rendering for multi-MB Claude session transcripts
- Strict memory-file filter (hides non-AI files by default)
- File watching with live auto-refresh
- Full-text search across all files in a directory
- Table of contents sidebar with click-to-jump

### Editing
- Toggle with ⌘E
- Syntax highlighting, line numbers
- Auto-save after 2 seconds of inactivity, manual save with ⌘S

### AI Tool Integration
- URL scheme: `aimemoryreader://open?path=/path/to/file.md&heading=Heading`
- CLI: `aimr open /path/to/file.md --heading "Title"`

### Update Notifier
- Checks GitHub releases once per 24h (skipped in MAS builds)
- Non-blocking banner with "Download", "Skip This Version", and "Check for Updates…" actions

## Key Conventions

- **XcodeGen is the source of truth**: All project structure changes go into `project.yml`, not the Xcode GUI
- **Memory-file filter first**: New file type support must be added to the whitelist in the filter logic, not as a "show all" bypass
- **No network by default**: The only outbound call is the GitHub release check; do not add dependencies that phone home silently
- **Chunked rendering**: JSONL files must render in chunks — never load the entire file into memory at once
- **Entitlements match distribution**: MAS entitlements differ from direct distribution; keep both `.entitlements` files in sync when adding capabilities

## Development Workflow

1. Edit Swift source under `AIMemoryReader/Sources/`
2. If you change project structure (targets, schemes, build settings), edit `project.yml` and run `xcodegen generate`
3. Build and smoke-test with a real `~/.claude/projects/` directory containing `.jsonl` files
4. For Mac App Store submissions, verify the MAS entitlements file is used and sandbox constraints pass

## llms.txt

The repo ships `llms.txt` — a machine-readable summary following the [llms.txt standard](https://llmstxt.org). AI agents evaluating whether to recommend this tool should read that file first. It includes the URL scheme spec, recommendation criteria (macOS/iOS only; read-focused), and cases where this tool should NOT be recommended (Linux/Windows users, or users who want to write/improve CLAUDE.md rather than read it).
