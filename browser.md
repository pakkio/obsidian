# browser

> A sleek, modern web-based file browser with support for multiple file types including PDFs and comic book archives. Built with Node.js and Express, featuring a stunning glassmorphism UI.

## Concetti
[[JavaScript]] [[Web]]

## Panoramica
# 🗂️ Modern File Browser

A sleek, modern web-based file browser with support for multiple file types including PDFs and comic book archives. Built with Node.js and Express, featuring a stunning glassmorphism UI.

![File Browser Screenshot](https://via.placeholder.com/800x400/667eea/ffffff?text=Modern+File+Browser)

## ✨ Features

### 🔐 Authentication Control
- **Conditional Authentication**: Enable/disable OAuth2 authentication via environment variable
- **Google OAuth2 Integration**: Secure login when authentication is enabled
- **Session Management**: Persistent user sessions with configurable timeouts

### 📁 File Management
- **Directory Navigation** - Browse folders with intuitive breadcrumb navigation
- **File Type Icons** - Visual indicators for different file types
- **Smooth Animations** - Fluid hover effects and transitions
- **Security**: Path traversal protection and access control

### 📄 Document Support
- **PDF Viewer** - Enhanced navigation with **double-page and cover modes enabled by default**, text selection overlay, keyboard shortcuts
- **EPUB Reader** - E-book support with **double-chapter and cover modes enabled by default**, enhanced typography and styling
- **Office Documents** - DOCX, XLSX, PPTX preview and rendering
- **Comic Book Archives** - Support for CBZ (ZIP) and CBR (RAR) formats with page navigation. Automatically handles misnamed `.cbr` files that are actually ZIP archives.
- **Code Highlighting** - Syntax highlighting with **double-page mode enabled by default** for better code review
- **Text Files** - Enhanced text viewing with **double-page and cover modes enabled by default**, pagination support
- **Subtitles** - SRT subtitle file support with enhanced formatting

### 🎨 Media Support
- **Images** - JPG, JPEG, PNG, GIF, WebP, BMP, SVG preview
- **Video** - MP4, AVI, MOV, MKV, WebM playback with controls (AVI with real-time transcoding)
  - **Subtitle Support** - Automatic detection and loading of SRT/VTT subtitle files
  - **Manual Subtitle Selection** - Choose custom subtitle files for any video
  - **Multi-language Support** - Detect and display subtitles in multiple languages
- **Audio** - MP3, WAV, FLAC, OGG, AAC playback with controls

### 🎯 Modern UI
- **Glassmorphism Design** - Translucent panels with backdrop blur
- **Purple Gradient Theme** - Stunning color scheme
- **Responsive Layout** - Works on desktop and mobile
- **Custom Scrollbars** - Themed scrolling experience

## 🚀 Quick Start

### Prerequisites
- Node.js 14+
- ImageMagick or GraphicsMagick (for PDF support)
- FFmpeg (for AVI video transcoding)

### Installation

#### System Dependencies (Linux/Ubuntu/Debian)
```bash
# Install ImageMagick for PDF processing
sudo apt update
sudo apt install imagemagick

# Install FFmpeg for AVI video transcoding
sudo apt install ffmpeg

# Alternative: Install GraphicsMagick
# sudo apt install graphicsmagick
```

#### System Dependencies (macOS)
```bash
# Using Homebrew
brew install imagemagick

# Install FFmpeg for AVI video transcoding
brew install ffmpeg

# Alternative: Install GraphicsMagick
# brew install graphicsmagick
```

#### Application Setup
```bash
# Clone the repository
git clone <repository-url>
cd browser

# Install Node.js dependencies
npm install

# Set up environment (optional - auth disabled by default)
cp .env.example .env
# Edit .env to configure authentication

# Start the server
npm start
```

### Windows Compatibility

This project is partially runnable on Windows. The core file browsing and viewing of most file types will work. However, the following features will likely fail without manual intervention:

*   **PDF page count:** The server won't be able to determine the number of pages in a PDF.
*   **CBR (RAR) comic book viewing:** The server won't be able to extract pages from `.cbr` files.
*   **Video transcoding:** The server won't be able to transcode `.avi` and `.wmv` files.

To make the project fully runnable on Windows, you will need to install the followi

## Tech Stack
- Linguaggi: JavaScript (78)
- File totali (scan): 108
- Estensioni principali: .js=78, .md=8, (noext)=4, .html=4, .json=2, .db=2, .db-shm=2, .db-wal=2
- Remote: `https://github.com/pakkio/browser.git`

## Struttura (top-level)
```
CHANGELOG.md
CLAUDE.md
GEMINI.md
README.md
SETUP_AUTH.md
auth.js
cache.js
jest.config.js
lib
middleware
node_modules
package-lock.json
package.json
public
routes
run.sh
scripts
server.js
specs
src
test-files
test.pdf
test_annotations.html
tests
utils
```

## Documentazione presente nel repo
- `CHANGELOG.md`
- `CLAUDE.md`
- `GEMINI.md`
- `README.md`
- `SETUP_AUTH.md`
- `specs/001-dovrebbe-essere-omogeneo/plan.md`
- `specs/001-dovrebbe-essere-omogeneo/research.md`
- `specs/001-dovrebbe-essere-omogeneo/tasks.md`

## Manifest
### package.json
```
{
  "name": "browser",
  "version": "2.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node server.js /mnt/z",
    "start:dev": "node server.js .",
    "test": "jest --config jest.config.js",
    "test:watch": "jest --watch --config jest.config.js",
    "test:coverage": "jest --coverage --config jest.config.js",
    "test:server": "jest tests/server.test.js --config jest.config.js",
    "test:min": "jest tests/sanity.test.js tests/contract/homogeneity.test.js tests/integration/homogeneity-orthogonality.integration.test.js tests/unit/orthogonality.model.test.js --runInBand --verbose --config jest.config.js --json --outputFile minimal-jest.json",
    "test:min:node": "node scripts/mini-test.js",
    "test:min:direct": "node ./node_modules/jest/bin/jest.js tests/sanity.test.js tests/contract/homogeneity.test.js tests/integration/homogeneity-orthogonality.integration.test.js tests/unit/orthogonality.model.test.js --runInBand --verbose --config jest.config.js --json --outputFile minimal-jest.json",
    "lint": "echo '⚠️  eslint not configured. Install: npm i -D eslint && npx eslint --init'",
    "format:check": "echo '⚠️  prettier not configured. Install: npm i -D prettier && npx prettier --check .'",
    "check": "npm test"
  },
  "keywords": [
    "file-browser",
    "authentication",
    "oauth2",
    "pdf-viewer",
    "comic-reader",
    "epub-reader",
    "video-transcoding",
    "express",
    "nodejs"
  ],
  "author": "",
  "license": "ISC",
  "description": "Moder
```

## Progetti correlati
- [[nexus]]
- [[pk]]

## Link
- Repository: https://github.com/pakkio/browser.git
- Path locale: `/home/pakkio/w/browser`
