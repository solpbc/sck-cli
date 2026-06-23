# CLAUDE.md

Developer guide for `sck-cli`, a macOS ScreenCaptureKit video + audio capture CLI. Read it before writing code. (`AGENTS.md` is a symlink to this file.)

## Project Overview

A lightweight macOS command-line tool that captures video and audio using Apple's ScreenCaptureKit framework. The tool captures all connected displays simultaneously, supports configurable frame rates, flexible capture durations (timed or indefinite), and optional audio recording of both system audio and microphone input. Video is encoded using hardware HEVC in .mov format with efficient NV12 pixel format.

## Build Commands

```bash
# Build debug version (default)
make build

# Build optimized release version (native architecture)
make release

# Build universal binary (arm64 + x86_64)
make release-universal

# Create distribution package (universal, stripped, zipped)
make dist

# Run the tool with default settings (builds if needed)
make run

# Run comprehensive tests
make test

# Clean build artifacts
make clean

# Clean distribution artifacts only
make clean-dist

# Clean captured output files (video and audio)
make clean-output

# Install universal binary to /usr/local/bin
make install
```

## Architecture

The application is a modular Swift CLI tool organized into separate concerns:

### Source Files (Sources/sck-cli/)

- **SCKShot.swift** - CLI entry point with ArgumentParser, display discovery, stream orchestration
- **AudioWriter.swift** - Multi-track M4A writing with AVAssetWriter (system audio + microphone)
- **AudioDeviceInfo.swift** - CoreAudio queries for default input and output device metadata
- **AudioDeviceMonitor.swift** - CoreAudio property listeners for device change detection
- **VideoWriter.swift** - HEVC hardware encoding to .mov with AVAssetWriter
- **StreamOutput.swift** - SCStreamOutput protocol implementations for video and audio capture
- **WindowMask.swift** - Window detection and pixel buffer masking for confidential mode
- **Output.swift** - Unbuffered stdout/stderr utilities for immediate output visibility

## Key Technical Details

- **Platform**: macOS 15.0+ required
- **Swift version**: 6.1
- **Video format**: HEVC in .mov container with NV12 pixel format, sRGB color space
- **Video encoding**: Hardware HEVC (8 Mbps default bitrate, sparse keyframes)
- **Audio format**: M4A with AAC codec (64 kbps per track, 48kHz, mono tracks)
- **Frameworks used**: ScreenCaptureKit, AVFoundation, CoreMedia, CoreAudio, ArgumentParser
- **Multi-display**: Captures all connected displays simultaneously
- **Output files**:
  - `<base>_<displayID>.mov` - One HEVC video file per display
  - `<base>.m4a` - Multi-track audio file with 2 separate tracks (system audio + microphone)
- **CLI Options**:
  - `-r, --frame-rate` - Frame rate in Hz (default: 1.0)
  - `-l, --length` - Duration in seconds
  - `--audio/--no-audio` - Enable/disable audio (default: enabled)
  - `--mask <app>` - App name(s) to mask with black rectangles (can specify multiple times)
  - `-v, --verbose` - Enable verbose logging
- **Output**: JSONL to stdout (one line per display, one for audio, one for stop); all logging to stderr

## Permissions Required

The tool requires macOS system permissions for:
- Screen & System Audio Recording
- Microphone (if capturing microphone input)

These must be granted in System Settings > Privacy & Security before the tool can function.

## Development Notes

- **Always run `make test` after code changes** to validate functionality
- Classes use `@unchecked Sendable` with NSLock for thread safety
- Factory method pattern for writer and output classes
- One SCStream per display; audio attached to first display's stream only
- Five completion modes: audio-driven, timer-driven, indefinite (Ctrl-C), auto-restart on system interruption, or device change
- **Auto-restart**: On error -3821 (system stopped stream due to low disk space), streams are automatically restarted up to 10 times while continuing to write to the same output files
- **Device change detection**: When default audio input or output device changes, capture stops cleanly (exit 0) with JSONL stop event indicating which device changed. Ideal for watchdog loops that restart on device changes.

## Dependencies

- **ffmpeg**: Used only for testing (not required at runtime)
  - Install via Homebrew: `brew install ffmpeg`

## Engineering Principles

sol pbc's coding standards, distilled — inlined because a coding agent working
in this repo can't read the private org standards.

- **Agent-native by design.** The contract with callers is JSONL on stdout (one line per display, one for audio, one for stop) and all logging on stderr — keep that split clean so a watchdog/agent can parse it. `--help` stays comprehensive; errors suggest the next step.
- **Fail clearly, with honest state.** A capture that can't start (missing permission, no device) reports the specific reason on stderr and exits non-zero — never a silent success or a zero-byte file presented as a recording. The device-change and auto-restart exit paths already model this; preserve them.
- **KISS / YAGNI.** This is a focused capture tool. Don't add modes, options, or fallbacks for cases that don't exist yet. No backwards-compatibility shims — update call sites directly.
- **Privacy is architecture.** Local capture only; no analytics, telemetry, crash reporting, or phone-home. The `--mask` confidential-window path exists to keep sensitive surfaces out of the recording — respect it.
- **Verify before you claim.** ScreenCaptureKit / AVFoundation / CoreAudio behavior is verified against the live API and real hardware, not recalled — run `make test` after every change.
