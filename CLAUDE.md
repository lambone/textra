# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

textra is a command-line tool that extracts text from images, PDFs, and audio files using Apple's Vision, VisionKit, and Speech APIs. Written in Swift, it requires macOS 13+ and is distributed as a compiled binary.

Version: 0.2.1

## Development Commands

### Building
```bash
# Build from Xcode project
xcodebuild -project textra.xcodeproj -scheme textra build

# Or open in Xcode
open textra.xcodeproj
```

### Testing
```bash
# Ensure textra is on your PATH, then run:
cd test
python test.py
```

The test suite uses testdata/ fixtures and validates outputs against expected results.

### Running Locally
```bash
# After building, run directly:
./textra [options] FILE1 [FILE2...] [outputOptions]

# Example:
./textra test/testdata/sample.png -o output.txt
```

## Architecture Overview

### File Structure
```
textra/
├── main.swift           # Main program: types, conversion pipeline, CLI parsing (1460 lines)
├── version.swift        # Version constant
└── Progress/
    ├── Progress.swift         # Progress bar core
    ├── ProgressElements.swift # Visual components (bar, ETA, percentage)
    └── Utilities.swift        # Helper functions
```

### Core Data Flow

The conversion pipeline follows this flow:

```
CLI Args → Parse → Detect File Types → Calculate Progress → Execute Conversions → Write Output
```

**Key Types:**
- `CLIInput`: Represents all CLI arguments (help, version, silent, inputFiles)
- `InputFile`: Groups files with unified output options (stdout, full text, page text, positional JSON)
- `ConvertFile`: Enum for `.pdf`, `.image`, or `.audio` file types
- `ConvertResponse`: Callback enum with `.update` or `.error` cases

### Conversion Pipeline

1. **File Type Detection** (lines 1295-1309): Uses UTType to classify files
2. **Page Count Gathering** (lines 145-167): PDFs use CGPDFDocument, audio uses AVAudioPlayer duration
3. **Type-Specific Conversion**:
   - **PDF** (lines 410-449): Rasterizes each page to CGImage at 600 DPI, then processes
   - **Image** (lines 393-400): Loads NSImage, converts to CGImage, then processes
   - **Audio** (lines 458-569): Uses SFSpeechRecognizer with on-device recognition, returns JSON with segments

4. **Text Extraction** (lines 338-384):
   - macOS 13+: Uses `ImageAnalyzer.analyze()` for full text
   - macOS <13: Falls back to Vision API's `VNRecognizeTextRequest`
   - Positional text always uses Vision API for bounding box precision

### Output System

**Pattern Expansion** (lines 645-692): Handles flexible file naming
- Pattern with `{}`: Substitutes page number or base filename
- Example: `doc.pdf` with `"page-{}.txt"` → `page-1.txt`, `page-2.txt`

**Output Formats:**
- `--outputStdout` / `-x`: Print to stdout (default if no options specified)
- `--outputText` / `-o`: Combined full text to single file
- `--outputPageText` / `-t`: Each page to separate text file
- `--outputPositions` / `-p`: Positional JSON with bounding boxes per page

**Positional JSON Structure:**
```json
{
  "observations": [
    {
      "observation": {
        "text": "recognized text",
        "confidence": 0.95,
        "bounds": {"x1": 0.1, "y1": 0.2, "x2": 0.9, "y2": 0.8},
        "subBounds": [
          {"text": "word", "offset": [0, 4], "bounds": {...}}
        ]
      }
    }
  ],
  "info": {"program": "textra", "version": "0.2.1"}
}
```

### Progress Module Integration

The Progress module provides visual feedback:
- `ProgressBar`: Core structure with weighted progress (audio files use duration-based weights)
- `ProgressPrinter` / `SilentProgressPrinter`: Pluggable output handlers
- Visual elements: bar `[=====    ]`, index "5 of 10", ETA, percentage
- Uses ANSI codes for colored terminal output when attached to TTY

Progress calculation:
- **Visible index**: Simple page count for display
- **Weighted index**: Audio uses `duration × 1/3` to normalize against typical page processing time

### Key Patterns

1. **Callback Pattern**: All conversion functions use `callback: @escaping (ConvertResponse) -> Void` for async streaming
2. **Smart Defaults**: Computed properties like `shouldOutputToStdout` enable intelligent fallback behavior
3. **Dual-Mode Text Recognition**: Feature detection for macOS 13+ ImageAnalyzer vs older Vision API
4. **ANSI Terminal Formatting**: Full color support with automatic fallback when not attached to terminal
5. **Locale Support**: Handles different identifier formats (CLDR for Vision, BCP47 for VisionKit, Locale objects for Speech)

## Important Implementation Details

### Text Recognition APIs

- **Vision API** (`VNRecognizeTextRequest`): Used for positional text and as fallback for older macOS. Requires CLDR-style locale identifiers (`en_US`)
- **VisionKit** (`ImageAnalyzer`): macOS 13+ only, faster for full text extraction. Requires BCP47-style identifiers (`en-US`)
- **Speech API** (`SFSpeechRecognizer`): For audio transcription. Requires `Locale` objects and on-device recognition enabled

### Sub-Bounds Extraction (lines 186-280)

The `extractSubBounds()` function walks through recognized text character-by-character to extract word-level bounding boxes. It merges adjacent boxes that share the same bounding box (handles character coalescence in the Vision API) and includes offset ranges for each word.

### Audio Transcription

- Uses `SFSpeechURLRecognitionRequest` with `requiresOnDeviceRecognition = true`
- Requires dictation enabled in macOS System Settings (see README troubleshooting)
- Returns JSON with segments containing text, timestamps, and confidence scores
- Shows partial results in real-time when attached to terminal (with progress updates)

### CLI Parsing (lines 1316-1429)

State-machine approach:
1. Iterate through arguments
2. Track current InputFile group
3. File arguments trigger file type detection and add to current group
4. Output options modify current group's output configuration
5. Support chained files with different outputs: `textra img1.png -o out1.txt img2.png -o out2.txt`

## Testing Notes

- Test suite in `test/test.py` validates all output formats
- Test data in `test/testdata/` includes sample images, PDFs, and audio files
- Tests check both stdout and file outputs
- Locale-specific tests verify language support (lines in test.py)
