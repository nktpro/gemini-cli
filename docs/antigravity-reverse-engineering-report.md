# Google Antigravity IDE - Complete Reverse Engineering Analysis

**Analysis Date:** January 5, 2026
**Version Analyzed:** 1.13.3 (Internal: 1.104.0)
**Download Size:** 210MB (compressed) | ~500MB (extracted)
**Platform:** Linux x86-64

---

## Executive Summary

Google Antigravity is a heavily modified fork of VSCode/Electron, transformed into an **agentic IDE** powered by Google's Gemini 3 AI models. Released in November 2025, it introduces the **Jetski Agent System** - a proprietary autonomous coding agent capable of multi-step reasoning, code generation, and task execution.

The architecture reveals significant departures from VSCode, with custom Google-internal packages (`@exa/*`), a massive language server binary (165MB), and deep integration with Google's infrastructure for feature flags, telemetry, and authentication.

---

## 1. Architecture & Base Technology

### 1.1 Foundation
```
Base:           VSCode fork (possibly via Windsurf → VSCode fork chain)
Electron:       v37.3.1
Node.js:        v22.18.0
License:        MIT
Bundle ID:      com.google.antigravity (macOS)
Data Folder:    ~/.antigravity
URL Scheme:     antigravity://
Alias:          agy
```

### 1.2 Version Information
```json
{
  "name": "Antigravity",
  "version": "1.104.0",
  "distro": "1b9ed19ee46b59bbf285f8e6fd673e25dbf477f6",
  "author": { "name": "Google" },
  "engines": { "node": "22.18.0" }
}
```

### 1.3 File Structure
```
/tmp/Antigravity/
├── antigravity (194MB)          # Main Electron executable
├── chrome-sandbox (48KB)
├── libEGL.so, libGLESv2.so, libvulkan.so.1
├── libffmpeg.so (2.7MB)
├── libvk_swiftshader.so (4.6MB)
├── chrome_crashpad_handler (1.5MB)
├── icudtl.dat (10MB)
├── resources/ (427MB)
│   ├── app/
│   │   ├── package.json
│   │   ├── product.json
│   │   ├── node_modules/ (391 dirs)
│   │   ├── extensions/ (103 extensions)
│   │   │   ├── antigravity/ (179MB)
│   │   │   ├── antigravity-browser-launcher/ (29KB)
│   │   │   ├── antigravity-code-executor/ (14KB)
│   │   │   ├── antigravity-dev-containers/ (1.8MB)
│   │   │   ├── antigravity-remote-openssh/ (421KB)
│   │   │   └── antigravity-remote-wsl/ (52KB)
│   │   └── out/
│   │       ├── main.js (4.3MB)
│   │       ├── jetskiAgent/
│   │       │   ├── main.js (7.2MB) ← **Jetski Agent App**
│   │       │   └── main.css (78KB)
│   │       ├── jetskiMain.tailwind.css (81KB)
│   │       └── vs/code/electron-browser/workbench/
│   │           ├── workbench-jetski-agent.html
│   │           └── jetskiAgent.js (bootstrap)
│   └── completions/
└── locales/ (42MB)
```

---

## 2. Jetski Agent System (Core Innovation)

### 2.1 What is Jetski?
The **Jetski Agent** is Antigravity's autonomous AI subsystem that executes multi-step coding tasks:
- **Location:** `/out/jetskiAgent/main.js` (7.2MB)
- **Execution Environment:** Isolated webview (workbench-jetski-agent.html)
- **Bootstrap:** `/out/vs/code/electron-browser/workbench/jetskiAgent.js`

### 2.2 Technology Stack
```javascript
// React/Preact UI Framework
"preact": "dist/preact.mjs"
"react": "preact/compat" (aliased)
"react-redux": "dist/react-redux.browser.mjs"

// State Management
"@reduxjs/toolkit": "^2.8.2"
"redux-thunk": "dist/redux-thunk.mjs"
"reselect": "dist/reselect.mjs"

// Text Editor
"lexical": "Lexical.prod.mjs"
"@lexical/react": "^0.34.0"
"lexical-beautiful-mentions": "index.js"

// Protocol Buffers & RPC
"@bufbuild/protobuf": "1.9.0"
"@connectrpc/connect": "1.4.0"
"@connectrpc/connect-node": "1.4.0"
"@connectrpc/connect-web": "1.4.0"

// Feature Flags
"unleash-proxy-client": "build/main.esm.js"

// UI Components
"react-tooltip": "dist/react-tooltip.mjs"
"@floating-ui/dom": "dist/floating-ui.dom.mjs"
"lucide-react": "dist/esm/lucide-react.js"

// Markdown Rendering
"react-markdown": "index.js"
"remark-gfm": "index.js"
"remark-github-blockquote-alert": "lib/index.js"
"rehype-raw": "index.js"
"rehype-sanitize": "index.js"
"rehype-slug": "index.js"

// Google Services
"google-auth-library": "build/src/index.js"
```

### 2.3 Webview Security Policy
```html
<!-- Content Security Policy from workbench-jetski-agent.html -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  img-src 'self' data: blob: vscode-remote-resource: https:;
  media-src 'self' data: blob: https://www.gstatic.com;
  frame-src 'self' vscode-webview:;
  script-src 'self' 'unsafe-eval' blob:;  <!-- unsafe-eval for modules -->
  style-src 'self' 'unsafe-inline';
  connect-src 'self' data:
    http://127.0.0.1:*
    http://jetski-unleash.corp.goog/    <!-- Feature flags -->
    http://antigravity-unleash.goog/
    https: ws:;
  font-src 'self' vscode-remote-resource:;
">
```

### 2.4 Jetski Capabilities
Based on package.json analysis:
- **Multi-step autonomous code generation**
- **Workflow-based task execution** (`.gemini/jetski*/global_workflows/*.md`)
- **Code hunk navigation and approval** (Alt+J/K, Alt+Enter)
- **Terminal command suggestions**
- **Embeddings computation** (up to 5000 workspace files)
- **Cascade panel** for code review
- **Browser automation** (via browser-launcher extension)

### 2.5 Keyboard Shortcuts
```javascript
// From antigravity extension package.json
"ctrl+shift+i" / "cmd+shift+i"  // Trigger agent
"ctrl+l" / "cmd+l"              // Open chat with agent
"ctrl+i" / "cmd+i"              // Prioritized command (editor/terminal)
"alt+j" / "alt+k"               // Navigate code hunks
"alt+enter"                     // Accept focused hunk
"alt+shift+backspace"           // Reject focused hunk
"ctrl+shift+l" / "cmd+shift+l"  // New conversation
"ctrl+shift+a" / "cmd+shift+a"  // Open conversation picker
```

---

## 3. Custom Extensions (5 Components)

### 3.1 antigravity (Core Extension - 179MB)
**Path:** `/extensions/antigravity/`

**Key Components:**
```bash
bin/
├── language_server_linux_x64 (165MB) # Massive LSP server (stripped ELF binary)
└── fd (4.1MB)                        # File discovery utility (fd-find)

dist/extension.js                      # Extension entry point
schemas/mcp_config.schema.json         # MCP configuration schema
```

**Features:**
- **Custom Editors:**
  - Workflow Editor (`.gemini/jetski*/global_workflows/*.md`)
  - Rule Editor (`.agent/rules/**/*.md`)
- **Authentication Provider:** `antigravity_auth`
- **40+ Commands** including:
  - `antigravity.login`
  - `antigravity.loginWithAuthToken`
  - `antigravity.copyApiKey`
  - `antigravity.generateCommitMessage`
  - `antigravity.triggerAgent`
  - `antigravity.openBrowser`
  - `antigravity.restartLanguageServer`
  - Settings import from Cursor, Windsurf, VSCode, Cider (Google internal)
- **MCP (Model Context Protocol) Support**
- **Embeddings:** Computes embeddings for up to 5000 workspace files (configurable)

**Language Server Binary:**
```bash
$ file language_server_linux_x64
ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
BuildID[md5/uuid]=0bd290d9d3ec29a44de8516c8a2d52e4, stripped
```

### 3.2 antigravity-browser-launcher (29KB)
**Path:** `/extensions/antigravity-browser-launcher/`

**Purpose:** Launch and manage Chrome/Chromium with Chrome DevTools Protocol (CDP) debugging

**Configuration:**
```javascript
{
  "commands": [
    "browserLauncher.launchBrowser",
    "browserLauncher.resetBrowserOnboarding",
    "browserLauncher.warmUpBrowser"
  ]
}
```

**Features:**
- Chrome/Chromium launch with CDP debugging (port 9222)
- Browser profile: `~/.gemini/antigravity-browser-profile`
- SSH reverse proxy for remote sessions
- Headless warmup mode
- Browser allowlist management

### 3.3 antigravity-code-executor (14KB)
**Path:** `/extensions/antigravity-code-executor/`

**Purpose:** Execute AI-generated JavaScript code in a sandboxed VM

**Features:**
- Sandboxed VM execution
- **Whitelisted modules:**
  - `fs`, `path`, `child_process`
  - `http`, `https`
  - VSCode API
- Async/await support with IIFE wrapping
- Output capture and error handling
- Used by Cascade panel for code generation

### 3.4 antigravity-dev-containers (1.8MB)
**Path:** `/extensions/antigravity-dev-containers/`

**Purpose:** Docker devcontainer integration (similar to VS Code Dev Containers)

**Features:**
- Docker devcontainer support
- Remote authority: `dev-container://`
- SSH agent forwarding
- Auto-server installation in containers
- Uses `@devcontainers/cli` npm package

### 3.5 antigravity-remote-openssh (421KB) & antigravity-remote-wsl (52KB)
**Path:** `/extensions/antigravity-remote-{openssh,wsl}/`

**antigravity-remote-openssh:**
- SSH remote development (`ssh-remote://` scheme)
- Cloudtop support (Google internal cloud development)
- Downloads/installs Antigravity server on remote systems
- SSH key forwarding

**antigravity-remote-wsl:**
- WSL 2 integration (`wsl://` scheme)
- WSL distro management
- Auto-server installation in WSL environments

---

## 4. Google-Internal @exa Packages (Proprietary)

### 4.1 Package References
From `package.json`:
```json
{
  "dependencies": {
    "@exa/agent-ui-toolkit": "file:../exa/agent_ui_toolkit/out",
    "@exa/proto-ts": "file:../exa/proto_ts/out"
  },
  "resolutions": {
    "@exa/typescript-utils": "file:../../../exa/typescript_utils",
    "@exa/agent-ui-toolkit": "file:../../../exa/agent_ui_toolkit"
  }
}
```

**Status:** ❌ **Not distributed** - These packages exist only in Google's internal build environment

### 4.2 @exa/proto-ts (Protocol Buffers)
Protocol buffer definitions discovered in import maps:

```javascript
"@exa/proto-ts/agent_manager_pb"          // Agent lifecycle management
"@exa/proto-ts/jetski_cortex_pb"          // Jetski decision-making engine
"@exa/proto-ts/chat_client_server_pb"     // Chat protocol
"@exa/proto-ts/chat_client_params_pb"     // Chat parameters
"@exa/proto-ts/language_server_pb"        // LSP integration
"@exa/proto-ts/language_server_connect"   // LSP connection
"@exa/proto-ts/unified_state_sync_pb"     // State synchronization
"@exa/proto-ts/diff_action_pb"            // Code editing actions
"@exa/proto-ts/trajectory_pb"             // Code generation tracking
"@exa/proto-ts/cascade_plugins_pb"        // Cascade panel plugins
"@exa/proto-ts/extension_server_pb"       // Extension server protocol
"@exa/proto-ts/reactive_component_pb"     // Reactive UI components
"@exa/proto-ts/chat_pb"                   // Chat messages
"@exa/proto-ts/jetski_service_pb"         // Jetski service definitions
"@exa/proto-ts/codeium_common_pb"         // Codeium integration (?)
"@exa/proto-ts/cortex_pb"                 // Cortex engine
```

**Purpose:** These protobufs define the RPC interfaces between:
- Jetski agent ↔ Language server
- Extension host ↔ Agent manager
- Chat client ↔ Chat server
- Code editor ↔ Diff engine

### 4.3 @exa/agent-ui-toolkit
React UI component library for the Jetski agent interface:
- Integrated with Tailwind CSS build pipeline
- Provides agent-specific UI components
- Used by AntigravityPanelManager

### 4.4 @exa/chat-client
Main chat UI rendering:
- `AntigravityPanelManager` component
- Chat message rendering
- Conversation history

---

## 5. Differences from Stock VSCode

| Feature | VSCode | Antigravity |
|---------|--------|-------------|
| **AI System** | GitHub Copilot (optional) | Jetski Agent (built-in) |
| **Agent Capabilities** | Code completion | Multi-step autonomous tasks |
| **Language Server** | ~10-50MB typical | 165MB proprietary binary |
| **Extensions** | Standard marketplace | 5 custom Google extensions |
| **Marketplace** | Microsoft marketplace | Open-VSX by default |
| **Authentication** | Microsoft/GitHub | Google OAuth + API keys |
| **Telemetry** | Microsoft Application Insights | Google telemetry |
| **Remote Development** | Microsoft Remote extensions | Custom SSH/WSL/devcontainer |
| **Browser Automation** | ❌ None | ✅ Chrome CDP integration |
| **Code Execution** | ❌ None | ✅ Sandboxed VM executor |
| **Workflow System** | ❌ None | ✅ `.gemini/jetski*/global_workflows/` |
| **Feature Flags** | Internal toggles | Unleash (jetski-unleash.corp.goog) |
| **Settings Import** | ❌ Manual | ✅ Cursor, Windsurf, Cider |
| **Protocol** | Language Server Protocol | LSP + Custom gRPC/Protobuf |

---

## 6. External Services & Infrastructure

### 6.1 Google Services
```
Download CDN:       edgedl.me.gvt1.com
Feature Flags:      jetski-unleash.corp.goog
                    antigravity-unleash.goog
Documentation:      antigravity.google/docs
Pricing:            one.google.com/ai (Gemini One subscription)
Media CDN:          www.gstatic.com
```

### 6.2 Extension Marketplace
```json
{
  "extensionsGallery": {
    "serviceUrl": "https://open-vsx.org/vscode/gallery",
    "itemUrl": "https://open-vsx.org/vscode/item"
  }
}
```
Uses **Open-VSX** instead of Microsoft's marketplace (VSCodium approach).

### 6.3 Telemetry
```json
{
  "crashReporter": {
    "productName": "Antigravity",
    "companyName": "Google"
  },
  "enabledTelemetryLevels": {
    "error": true,
    "usage": true
  }
}
```
**SDK:** `@microsoft/1ds-core-js` (Microsoft's Application Insights, reused)

---

## 7. Security Considerations

### 7.1 Sandboxing
- **Code Executor:** VM-based with whitelisted modules (`fs`, `path`, `child_process`, `http`, `vscode`)
- **Webview CSP:** Allows `unsafe-eval` for ES modules
- **Chrome Sandbox:** SUID binary for browser isolation

### 7.2 Authentication
- Google OAuth flow (`antigravity_auth` provider)
- Token-based API key management (`antigravity.copyApiKey`)
- SSH key forwarding support (remote development)

### 7.3 Network Access
- **Chrome DevTools Protocol:** Port 9222 (localhost)
- **WebSocket:** Real-time features (chat, state sync)
- **Google CDN:** Downloads, updates, media
- **Feature Flags:** HTTP to unleash.corp.goog

### 7.4 Privileged Operations
```bash
-rwsr-xr-x  chrome-sandbox  # SUID root for sandboxing
-rwxr-xr-x  language_server_linux_x64  # 165MB native binary
-rwxr-xr-x  fd  # File discovery (can traverse filesystem)
```

---

## 8. Key Findings & Observations

### 8.1 Architectural Insights
1. **Jetski is the differentiator** - The 7.2MB agent app is where the "magic" happens
2. **Protocol Buffers everywhere** - RPC-heavy architecture, not REST/JSON
3. **Massive language server** - 165MB suggests bundled ML models or extensive static analysis
4. **@exa packages are the secret sauce** - Not open-sourced, Google-internal only
5. **Electron as a shell** - Core logic moved to gRPC services, Electron is just UI
6. **Feature flags for everything** - Unleash integration for A/B testing

### 8.2 Reverse Engineering Challenges
- **@exa packages not distributed** - Cannot fully replicate Jetski without them
- **Language server is stripped** - No symbols, hard to reverse engineer
- **Protobuf schemas not included** - Only compiled JS definitions in bundles
- **Feature flags server is internal** - `jetski-unleash.corp.goog` not public
- **Workflow format undocumented** - `.gemini/jetski*/global_workflows/*.md` structure unknown

### 8.3 Codebase Lineage
Evidence suggests fork chain:
```
VSCode → Windsurf (?) → Antigravity
```
References to Windsurf settings import and similar naming conventions.

### 8.4 Business Model
- **Free tier:** Public preview with generous Gemini 3 rate limits
- **Paid tier:** Gemini One subscription (one.google.com/ai)
- **Enterprise:** Likely Google Cloud integration (Cloudtop support)

---

## 9. Technical Deep Dive: Key Files

### 9.1 jetskiAgent/main.js (7.2MB)
**Import Map Snippet:**
```javascript
{
  imports: {
    "react": "../../../../../node_modules/preact/compat/dist/compat.mjs",
    "@exa/agent-ui-toolkit": "../../../../../node_modules/@exa/agent-ui-toolkit/dist/index.js",
    "@exa/proto-ts/jetski_cortex_pb": "../../../../../node_modules/@exa/proto-ts/dist/exa/jetski_cortex_pb/jetski_cortex_pb.js",
    "unleash-proxy-client": "../../../../../node_modules/unleash-proxy-client/build/main.esm.js",
    "google-auth-library": "../../../../../node_modules/google-auth-library/build/src/index.js"
  }
}
```

### 9.2 product.json (Branding)
```json
{
  "nameShort": "Antigravity",
  "nameLong": "Antigravity",
  "applicationName": "antigravity",
  "aliasName": "agy",
  "dataFolderName": ".antigravity",
  "darwinBundleIdentifier": "com.google.antigravity",
  "urlProtocol": "antigravity",
  "updateUrl": "https://example.com",  // Placeholder
  "extensionsGallery": {
    "serviceUrl": "https://open-vsx.org/vscode/gallery"
  }
}
```

### 9.3 antigravity Extension (package.json)
```json
{
  "name": "antigravity",
  "displayName": "Antigravity",
  "description": "Extension that powers many of the AI features in Antigravity.",
  "version": "0.2.0",
  "publisher": "google",
  "activationEvents": ["*"],
  "enabledApiProposals": [
    "contribSourceControlInputBoxMenu",
    "antigravityEditorNudge",
    "antigravityAuth",
    "inlineCompletionsAdditions",
    "antigravityTerminalSuggestions",
    "antigravityUnifiedStateSync"
  ]
}
```

---

## 10. Reproduction Possibilities

### 10.1 What Can Be Replicated
✅ **UI Shell:** Electron + React structure
✅ **Extension System:** VSCode extension API
✅ **Basic Agent Framework:** Chat interface, message routing
✅ **Browser Launcher:** Chrome CDP integration
✅ **Remote Development:** SSH/WSL/devcontainer patterns

### 10.2 What Cannot Be Replicated (Without @exa)
❌ **Jetski Decision Engine:** `jetski_cortex_pb` logic
❌ **Language Server:** 165MB binary with proprietary logic
❌ **State Synchronization:** `unified_state_sync_pb` protocol
❌ **Trajectory Tracking:** Code generation analytics
❌ **Agent UI Toolkit:** Google-internal React components

### 10.3 Alternative Approaches
To build an Antigravity-like system without @exa packages:
1. **Use Cursor as inspiration** - Similar architecture, open approach
2. **Build on Language Server Protocol** - Standard LSP + custom extensions
3. **Implement agent with LangChain/LangGraph** - Alternative to Jetski
4. **Protocol Buffers from scratch** - Define your own agent protocols
5. **Open-source UI toolkit** - Shadcn/UI, Radix, etc.

---

## 11. Conclusions

Google Antigravity represents a **fundamental reimagining of VSCode** as an agentic development platform. The Jetski agent system, powered by proprietary `@exa` packages and a massive language server, goes far beyond traditional code completion.

**Key Innovations:**
- **Autonomous multi-step agents** (Jetski)
- **Protocol buffer-based agent communication**
- **Browser automation for web development**
- **Sandboxed code execution**
- **Comprehensive remote development platform**

**Limitations for Reverse Engineering:**
- Deep dependency on Google-internal `@exa` packages
- Stripped language server binary (165MB black box)
- Feature flag infrastructure requires Google services
- Workflow/rule system format not documented

**Competitive Positioning:**
- **vs GitHub Copilot:** More autonomous, multi-step tasks
- **vs Cursor:** Similar concept, but Google-backed with Gemini integration
- **vs VSCode:** AI-first architecture, not bolt-on extensions

The system is **designed for Google's Gemini ecosystem** with deep integration points that would be difficult to replicate without access to the `@exa` packages.

---

## 12. File Inventory

### 12.1 Critical Binaries
```
antigravity                        194MB  Main executable
language_server_linux_x64          165MB  LSP server (stripped ELF)
fd                                 4.1MB  File discovery
chrome_crashpad_handler            1.5MB  Crash reporting
libGLESv2.so                       6.3MB  OpenGL ES
libvk_swiftshader.so               4.6MB  Vulkan SwiftShader
libffmpeg.so                       2.7MB  Media codec
```

### 12.2 JavaScript Bundles
```
jetskiAgent/main.js                7.2MB  Jetski agent app
main.js                            4.3MB  Main process
cli.js                             201KB  CLI entry point
```

### 12.3 Data Files
```
LICENSES.chromium.html             15MB   Chromium licenses
ThirdPartyNotices.txt              2.2MB  OSS notices
icudtl.dat                         10MB   ICU data
resources.pak                      6.0MB  Electron resources
locales/                           42MB   Translations
```

---

## 13. Next Steps for Deeper Analysis

1. **Decompile language_server_linux_x64** (165MB)
   - Use Ghidra/IDA Pro to reverse engineer
   - Look for embedded ML models (TensorFlow Lite, ONNX)
   - Identify gRPC service implementations

2. **Analyze jetskiAgent/main.js** (7.2MB)
   - Beautify/de-minify with source maps
   - Trace Redux state management
   - Identify agent decision logic

3. **Reconstruct @exa/proto-ts schemas**
   - Parse compiled protobuf JS definitions
   - Generate `.proto` files from TypeScript definitions
   - Map RPC service interfaces

4. **Monitor network traffic**
   - Launch Antigravity with Wireshark/mitmproxy
   - Capture gRPC calls to language server
   - Reverse engineer feature flag API

5. **Explore workflow system**
   - Create test `.gemini/jetski*/global_workflows/*.md` files
   - Observe agent behavior changes
   - Document workflow syntax

6. **Test MCP integration**
   - Review `mcp_config.schema.json`
   - Connect custom MCP servers
   - Understand Model Context Protocol usage

7. **Examine browser integration**
   - Launch with `--inspect` flag
   - Attach Chrome DevTools to port 9222
   - Analyze browser automation patterns

---

## Appendix A: Download Locations

```bash
# Latest version (1.13.3)
wget https://edgedl.me.gvt1.com/edgedl/release2/j0qc3/antigravity/stable/1.13.3-4533425205018624/linux-x64/Antigravity.tar.gz

# Extract
tar -xzf Antigravity.tar.gz

# Launch
cd Antigravity
./antigravity
```

**Official Pages:**
- Download: https://antigravity.google/download/linux
- GitHub: https://github.com/oslook/antigravity-downloads
- Docs: https://antigravity.google/docs

---

## Appendix B: Command Reference

```bash
# File analysis
file language_server_linux_x64
strings language_server_linux_x64 | head -100
objdump -T language_server_linux_x64 | grep -i "grpc\|proto"
ldd language_server_linux_x64

# JavaScript inspection
cat package.json | jq '.dependencies | keys[]'
cat product.json | jq '.extensionsGallery'
head -1000 jetskiAgent/main.js | grep "import"

# Extension analysis
ls -lh extensions/antigravity*/package.json
cat extensions/antigravity/package.json | jq '.contributes.commands'

# Network monitoring
strace -e connect ./antigravity 2>&1 | grep "jetski\|antigravity"
tcpdump -i any -A port 9222
```

---

**Report compiled from:**
- Direct filesystem analysis of extracted Antigravity 1.13.3
- package.json dependency inspection
- product.json configuration review
- Extension manifest parsing
- Binary file analysis
- Import map reconstruction

**Analyst Notes:**
This is a **static analysis** based on the distributed binary. Runtime behavior analysis would require:
- Network traffic capture
- Memory dumps during agent execution
- gRPC message interception
- Feature flag server simulation

---

*End of Report*
