# Antigravity Browser Feedback Loop: Complete Reverse Engineering

**Analysis Date:** January 5, 2026
**Focus:** Front-end development "special sauce" - Browser automation & feedback loop
**Key Innovation:** Tight integration between Jetski agent and Chrome browser via CDP

---

## Executive Summary

Antigravity's competitive advantage for front-end development lies in its **closed-loop browser automation system** that enables the Jetski AI agent to:

1. **Control the browser** (click, type, scroll, navigate)
2. **Observe behavior** (console logs, errors, network, DOM changes)
3. **Generate fixes** based on observed issues
4. **Apply changes** to code
5. **Verify fixes** automatically in the browser

This creates an **autonomous testing and debugging cycle** where the agent learns from actual runtime behavior.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANTIGRAVITY IDE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │  Jetski Agent    │◄────────│  Language Server │             │
│  │  (7.2MB React)   │  gRPC   │  (165MB binary)  │             │
│  └────────┬─────────┘         └──────────────────┘             │
│           │                                                      │
│           │ WebSocket (CDP Commands)                            │
│           ↓                                                      │
│  ┌──────────────────────────────────────────────────────┐       │
│  │     Browser Launcher Extension (29KB)                │       │
│  │  - Spawns Chrome with --remote-debugging-port=9222   │       │
│  │  - Manages isolated profile ~/.gemini/antigravity-*  │       │
│  │  - Polls http://127.0.0.1:9222/json/version          │       │
│  └────────────────────┬─────────────────────────────────┘       │
│                       │                                          │
└───────────────────────┼──────────────────────────────────────────┘
                        │ CDP over WebSocket
                        │ ws://127.0.0.1:9222
                        ↓
        ┌───────────────────────────────────────┐
        │         CHROME BROWSER                │
        ├───────────────────────────────────────┤
        │  Chrome Extension (eeijfnjmj...)      │
        │  - Installed from Chrome Web Store    │
        │  - Bridges CDP to browser DOM         │
        │  - Shows agent overlay during work    │
        └───────────────────────────────────────┘
                        ↓
                ┌───────────────┐
                │  Web App      │
                │  (localhost)  │
                └───────────────┘
```

---

## Component 1: Browser Launcher Extension

### Location
`/tmp/Antigravity/resources/app/extensions/antigravity-browser-launcher/`

### Capabilities

**1. Chrome Binary Discovery**
Searches multiple paths across platforms:

```javascript
// macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"/Applications/Chromium.app/Contents/MacOS/Chromium"

// Linux
"/usr/bin/google-chrome"
"/usr/bin/google-chrome-stable"
"/usr/bin/chromium-browser"
"/usr/bin/chromium"
"/snap/bin/chromium"

// Windows
"C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe"
"%LOCALAPPDATA%\\Google\\Chrome\\Application\\chrome.exe"
```

**2. Chrome Launch Flags**
```bash
--remote-debugging-port=9222           # Enable CDP
--user-data-dir=~/.gemini/antigravity-browser-profile  # Isolated profile
--disable-fre                           # No First Run Experience
--no-default-browser-check              # Skip default browser prompt
--no-first-run                          # Skip onboarding
--auto-accept-browser-signin-for-tests  # Auto-accept sign-in
--ash-no-nudges                         # Disable Chrome OS nudges
--disable-features=OfferMigrationToDiceUsers,OptGuideOnDeviceModel
```

**3. CDP Health Checks**
```javascript
// Polls every 100ms for up to 30 seconds
curl -s http://127.0.0.1:9222/json/version

// Response when ready:
{
  "Browser": "Chrome/120.0.6099.109",
  "Protocol-Version": "1.3",
  "User-Agent": "Mozilla/5.0 ...",
  "V8-Version": "12.0.267.8",
  "WebKit-Version": "537.36",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."
}
```

**4. SSH Reverse Proxy for Remote Development**
```bash
# When running over SSH (remoteAuthority: "ssh-remote+...")
ssh -R 9222:127.0.0.1:9222 user@remote-host -N \
  -o ExitOnForwardFailure=yes \
  -o LogLevel=ERROR \
  -o StreamLocalBindUnlink=yes
```

**5. Headless Warmup Mode**
```javascript
// Pre-launches browser to reduce latency
--headless=new
--disable-gpu
--no-sandbox
--disable-dev-shm-usage
--remote-debugging-port=9223  // Different port (9223, not 9222!)
about:blank

// Timeout: 5 seconds (vs 30s for normal launch)
// Process is killed after successful startup
```

---

## Component 2: Chrome DevTools Protocol (CDP) Integration

### CDP Library
**Package:** `chrome-remote-interface` v0.33.0
**Location:** `/tmp/Antigravity/resources/app/node_modules/chrome-remote-interface/`

### CDP Endpoints
```
http://127.0.0.1:9222/json/list      - List all debuggable targets (tabs)
http://127.0.0.1:9222/json/version   - Browser version info
http://127.0.0.1:9222/json/protocol  - CDP protocol definition (1.2MB JSON)
http://127.0.0.1:9222/json/new?url   - Create new target/tab
http://127.0.0.1:9222/json/activate/:id - Activate target
http://127.0.0.1:9222/json/close/:id - Close target
```

### WebSocket Communication
```javascript
// Connect to WebSocket debugger URL
const ws = new WebSocket('ws://127.0.0.1:9222/devtools/page/...', [], {
  maxPayload: 256 * 1024 * 1024,  // 256MB max
  perMessageDeflate: false,
  followRedirects: true
});

// Send CDP commands (JSON-RPC)
ws.send(JSON.stringify({
  id: 1,
  method: "Runtime.evaluate",
  params: {
    expression: "document.querySelector('#login').click()",
    returnByValue: true
  }
}));

// Receive CDP events
ws.on('message', (data) => {
  const message = JSON.parse(data);
  if (message.method === 'Console.messageAdded') {
    // Handle console log
  }
});
```

### CDP Domains Used

**Runtime Domain** - JavaScript execution
```
Runtime.evaluate         - Execute JavaScript
Runtime.getProperties    - Inspect objects
Runtime.exceptionThrown  - Capture errors
```

**Console Domain** - Console output capture
```
Console.enable          - Enable console
Console.messageAdded    - Console.log/error/warn events
Console.clear           - Clear console
```

**DOM Domain** - DOM inspection/manipulation
```
DOM.getDocument         - Get DOM tree
DOM.querySelector       - Find elements
DOM.setAttributeValue   - Modify attributes
DOM.documentUpdated     - DOM change events
```

**Page Domain** - Page control
```
Page.navigate           - Navigate to URL
Page.reload             - Reload page
Page.captureScreenshot  - Take screenshots
Page.frameNavigated     - Navigation events
```

**Network Domain** - Network monitoring
```
Network.enable                  - Enable network tracking
Network.requestWillBeSent      - HTTP request events
Network.responseReceived       - HTTP response events
Network.loadingFailed          - Failed request events
```

**Debugger Domain** - Debugging
```
Debugger.enable            - Enable debugger
Debugger.setPauseOnExceptions - Break on errors
Debugger.paused            - Breakpoint hit
Debugger.resume            - Continue execution
```

**Target Domain** - Tab management
```
Target.getTargets       - List tabs
Target.attachToTarget   - Attach to tab
Target.createTarget     - Create new tab
```

---

## Component 3: Chrome Browser Extension

### Extension Identity
**Chrome Web Store URL:** `https://chromewebstore.google.com/detail/antigravity-browser-exten/eeijfnjmjelapkebgockoeaadonbchdd`
**Extension ID:** `eeijfnjmjelapkebgockoeaadonbchdd`
**Distribution:** Chrome Web Store (NOT bundled in IDE)

### Installation Flow
1. User launches browser via agent or command
2. Onboarding page displays: `/tmp/Antigravity/resources/app/out/vs/platform/browserOnboarding/static/browserOnboarding.html`
3. Page shows: "Install the extension to get started"
4. User clicks to install from Chrome Web Store
5. Extension connects to CDP port 9222

### Extension Capabilities
From `browserLanding.html` description:
> "The agent can **click, scroll, type, and navigate** web pages automatically. While working, it displays an **overlay** showing its progress and provides controls to stop execution if you need to intervene."

### Extension Role
- **DOM Bridge:** Bridges CDP to actual browser DOM
- **Visual Overlay:** Shows agent actions as they happen
- **User Controls:** Provides stop/pause buttons during automation
- **Event Forwarding:** Forwards browser events back to CDP

---

## Component 4: Jetski Agent Browser Integration

### Location
`/tmp/Antigravity/resources/app/out/jetskiAgent/main.js` (7.2MB)

### Browser Automation Commands (15+)

**Click Interactions:**
```javascript
browserClickElement({ selector, description })
  // Clicks on a specific DOM element by selector or AI-detected description

clickBrowserPixel({ x, y })
  // Clicks at exact pixel coordinates
```

**Input Interactions:**
```javascript
browserInput({ selector, text })
  // Types text into input fields

browserPressKey({ key })
  // Simulates keyboard presses (Enter, Tab, Escape, etc.)
```

**Mouse Interactions:**
```javascript
browserMoveMouse({ x, y })
  // Moves mouse cursor to position

browserDragPixelToPixel({ fromX, fromY, toX, toY })
  // Click and drag between coordinates

browserMouseWheel({ deltaY })
  // Scroll with mouse wheel
```

**Navigation & Scrolling:**
```javascript
browserScrollUp()
browserScrollDown()

browserResizeWindow({ width, height })
```

**Data Capture:**
```javascript
captureBrowserScreenshot()
  // Takes screenshot at current state

captureBrowserConsoleLogs()
  // Captures all console.log/error/warn output

browserGetDom()
  // Extracts full DOM structure as HTML

readBrowserPage()
  // Reads visible page content (text extraction)
```

**Session Management:**
```javascript
openBrowserUrl({ url })
  // Opens URL in browser

listBrowserPages()
  // Lists all open tabs/pages

setBrowserOpenConversation({ conversationId })
  // Associates browser with conversation

smartOpenBrowser()
  // Intelligent browser opening with context awareness
```

### Redux State Management

**Browser State:**
```javascript
{
  browserStateSnapshot: {
    url: "http://localhost:3000",
    title: "My App",
    consoleMessages: [...],
    errors: [...]
  },
  browserSubagentView: {
    subtrajectoryId: "abc123",
    isExecuting: true,
    currentStep: 3
  },
  browserStateDiff: "{...}"  // JSON diff of state changes
}
```

**Redux Actions:**
- `updateBrowserSubagentView` - Updates browser UI state
- `setIsAgentDriven` - Marks actions as agent-driven (vs user-driven)

### React UI Components

**Action Components:**
```javascript
<BrowserClickElement selector="#login-btn" />
<BrowserInput selector="#email" text="user@example.com" />
<BrowserPressKey key="Enter" />
<BrowserScrollDown distance={500} />
<CaptureBrowserScreenshot />
<ExecuteBrowserJavascript code="..." />
```

**Permission UI:**
```javascript
<BrowserValidateCascadeOrCancelOverlay
  hostname="localhost"
  onAllow={() => addToAllowlist()}
  onDeny={() => cancelAction()}
/>
```

**Browser Panel:**
```javascript
<BrowserPanel>
  <BrowserView src="http://localhost:3000" />
  <BrowserSubagent trajectory={...} />
</BrowserPanel>
```

---

## Component 5: Browser Allowlist & Security System

### Security Architecture

**Multi-Level Access Control:**

```javascript
// Protocol Buffer Definitions
BrowserAllowlistConfig {
  repeated string allowlisted_urls
}

BrowserToolsConfig {
  enum agent_browser_tools {
    UNSPECIFIED,
    ENABLED,
    DISABLED
  }
}

BrowserJavascriptExecutionConfig {
  enum browser_js_execution_policy {
    UNSPECIFIED,
    ALWAYS_ASK,      // User confirmation required
    DEFAULT_ALLOW,   // Auto-run in Turbo mode
  }
}
```

**Permission Layers:**
1. **System Allowlist** - Pre-approved domains (Google-controlled)
2. **User Allowlist** - User-approved domains
3. **System Denylist** - Blocked domains (security threats)
4. **User Denylist** - User-blocked domains

### RPC Methods
```javascript
AddToBrowserWhitelist({ url })
  // Adds URL to user allowlist

GetAllBrowserWhitelistedUrls()
  // Returns: { urls: ["localhost", "127.0.0.1", ...] }

GetBrowserWhitelistFilePath()
  // Returns: { path: "~/.gemini/antigravity-allowlist.json" }
```

### Commands
```javascript
antigravity.showBrowserAllowlist
  // Opens allowlist management UI

antigravity.openBrowser
  // Opens browser with permission checks
```

### Permission Workflow

**User Prompt:**
```
┌──────────────────────────────────────────────┐
│ Agent needs permission to act on localhost   │
├──────────────────────────────────────────────┤
│ The agent wants to:                          │
│ - Click login button                         │
│ - Input credentials                          │
│ - Capture console logs                       │
├──────────────────────────────────────────────┤
│  [ Always Allow ]  [ Allow Once ]  [ Deny ]  │
└──────────────────────────────────────────────┘
```

**Allowlist Storage:**
```json
// ~/.gemini/antigravity-allowlist.json
{
  "allowlisted_urls": [
    "http://localhost",
    "http://127.0.0.1",
    "https://my-app.dev",
    "https://*.example.com"
  ],
  "last_updated": "2026-01-05T12:00:00Z"
}
```

### Security Features

**Sandboxing:**
- Isolated browser profile (`~/.gemini/antigravity-browser-profile`)
- Separate CDP port (9222) not exposed externally
- URL validation before every navigation
- JavaScript execution governed by policy

**File/Directory Access Control:**
```javascript
file_allowlist    // Allowed files
dir_allowlist     // Allowed directories
file_denylist     // Blocked files
dir_denylist      // Blocked directories
```

**Remote Development Security:**
- SSH reverse proxy validates host keys
- CDP port tunneled securely over SSH
- No browser data transmitted over network (only CDP commands)

---

## The Feedback Loop: How It Works

### Phase 1: User Request
```
User: "Test the login form and fix any errors"
```

### Phase 2: Agent Planning
```javascript
// Jetski generates plan
{
  steps: [
    { action: "openBrowserUrl", params: { url: "http://localhost:3000" } },
    { action: "browserInput", params: { selector: "#email", text: "test@example.com" } },
    { action: "browserInput", params: { selector: "#password", text: "password" } },
    { action: "browserClickElement", params: { description: "Login button" } },
    { action: "captureBrowserConsoleLogs", params: {} },
    { action: "captureBrowserScreenshot", params: {} }
  ]
}
```

### Phase 3: Permission Check
```javascript
// Check if localhost is allowlisted
if (!isAllowlisted("localhost")) {
  showPermissionPrompt("localhost");
  await waitForUserDecision();
}
```

### Phase 4: Execution with CDP

**Step 1: Open URL**
```javascript
// Send CDP command
await CDP.send('Page.navigate', { url: "http://localhost:3000" });

// Wait for load
await CDP.once('Page.loadEventFired');

// Capture screenshot
const screenshot = await CDP.send('Page.captureScreenshot', {
  format: 'png',
  quality: 80
});
```

**Step 2: Fill form**
```javascript
// Find email input
const { root } = await CDP.send('DOM.getDocument');
const { nodeId } = await CDP.send('DOM.querySelector', {
  nodeId: root.nodeId,
  selector: '#email'
});

// Set value
await CDP.send('DOM.setAttributeValue', {
  nodeId,
  name: 'value',
  value: 'test@example.com'
});

// Or execute JavaScript
await CDP.send('Runtime.evaluate', {
  expression: `document.querySelector('#email').value = 'test@example.com'`
});
```

**Step 3: Click button**
```javascript
// Option A: CSS selector
await CDP.send('Runtime.evaluate', {
  expression: `document.querySelector('button[type="submit"]').click()`
});

// Option B: AI-detected element (via DOM analysis)
const { nodeId } = await findElementByDescription("Login button");
await CDP.send('DOM.focus', { nodeId });
await CDP.send('Input.dispatchKeyEvent', { type: 'keyDown', key: 'Enter' });
```

### Phase 5: Observation

**Console Logs:**
```javascript
// Enable console monitoring
await CDP.send('Console.enable');

// Listen for console messages
CDP.on('Console.messageAdded', (event) => {
  const { message } = event;
  console.log(`[${message.level}] ${message.text}`);

  if (message.level === 'error') {
    // Capture error for analysis
    captureError(message);
  }
});
```

**Network Monitoring:**
```javascript
// Enable network tracking
await CDP.send('Network.enable');

// Monitor requests
CDP.on('Network.requestWillBeSent', (event) => {
  console.log(`Request: ${event.request.method} ${event.request.url}`);
});

CDP.on('Network.responseReceived', (event) => {
  console.log(`Response: ${event.response.status} ${event.response.url}`);

  if (event.response.status >= 400) {
    // Capture error response
    captureNetworkError(event);
  }
});

CDP.on('Network.loadingFailed', (event) => {
  // Failed request (CORS, timeout, etc.)
  captureNetworkFailure(event);
});
```

**JavaScript Errors:**
```javascript
// Enable exception tracking
await CDP.send('Runtime.enable');

CDP.on('Runtime.exceptionThrown', (event) => {
  const { exceptionDetails } = event;
  console.error('Exception:', exceptionDetails.text);
  console.error('Stack:', exceptionDetails.stackTrace);

  // Send to Jetski for analysis
  reportException(exceptionDetails);
});
```

**DOM Changes:**
```javascript
// Monitor DOM updates
await CDP.send('DOM.enable');

CDP.on('DOM.documentUpdated', () => {
  // DOM structure changed
  captureNewDOMSnapshot();
});
```

### Phase 6: Analysis

**Jetski analyzes captured data:**
```javascript
{
  consoleErrors: [
    "TypeError: Cannot read property 'email' of null at login.js:42"
  ],
  networkErrors: [
    { status: 401, url: "/api/auth/login", message: "Unauthorized" }
  ],
  screenshot: "data:image/png;base64,...",
  domSnapshot: "<html>...</html>"
}
```

**Language Server generates fix:**
```javascript
// Identified issue: Missing null check
// Location: src/components/LoginForm.js:42

// Original code:
const email = formData.email.trim();

// Proposed fix:
const email = formData?.email?.trim() || '';
```

### Phase 7: Code Modification

**Generate diff:**
```javascript
{
  file: "src/components/LoginForm.js",
  oldContent: "const email = formData.email.trim();",
  newContent: "const email = formData?.email?.trim() || '';",
  lineNumber: 42
}
```

**Apply changes:**
```javascript
// Write to file
await fs.writeFile('src/components/LoginForm.js', newContent);

// Trigger hot reload (if supported)
// Browser auto-reloads via dev server
```

### Phase 8: Verification

**Re-test automatically:**
```javascript
// Wait for hot reload
await sleep(1000);

// Re-run test steps
await openBrowserUrl("http://localhost:3000");
await browserInput({ selector: "#email", text: "test@example.com" });
await browserInput({ selector: "#password", text: "password" });
await browserClickElement({ description: "Login button" });

// Check for errors
const consoleLogs = await captureBrowserConsoleLogs();
const hasErrors = consoleLogs.some(log => log.level === 'error');

if (!hasErrors) {
  reportSuccess("Login form fixed! No more console errors.");
} else {
  reportFailure("Still seeing errors. Analyzing further...");
  // Continue iteration
}
```

### Phase 9: User Feedback

**Display results in IDE:**
```javascript
// Jetski UI shows:
✅ Step 1: Opened http://localhost:3000
   [Screenshot]

✅ Step 2: Filled email input
   [Screenshot]

✅ Step 3: Clicked Login button
   [Screenshot]

❌ Step 4: Console error detected
   TypeError: Cannot read property 'email' of null
   📄 src/components/LoginForm.js:42

✅ Step 5: Applied fix
   [Diff view]

✅ Step 6: Verified fix works
   [Screenshot showing success]
   No console errors!
```

---

## Real-World Example: Full Cycle

### User Request
```
"Build a todo app with React and test that adding items works"
```

### Jetski Execution

**1. Code Generation**
```javascript
// Generates React component
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [input, setInput] = useState('');

  const addTodo = () => {
    setTodos([...todos, { id: Date.now(), text: input }]);
    setInput('');
  };

  return (
    <div>
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={addTodo}>Add</button>
      <ul>
        {todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
      </ul>
    </div>
  );
}
```

**2. Browser Testing**
```javascript
// Permission check
await requestBrowserAccess("localhost");

// Launch browser
await openBrowserUrl("http://localhost:3000");
await captureBrowserScreenshot(); // Initial state

// Add first todo
await browserInput({ selector: "input", text: "Buy milk" });
await browserClickElement({ description: "Add button" });
await captureBrowserScreenshot(); // After first add

// Verify todo appears
const pageContent = await readBrowserPage();
assert(pageContent.includes("Buy milk"));

// Add second todo
await browserInput({ selector: "input", text: "Walk dog" });
await browserClickElement({ description: "Add button" });
await captureBrowserScreenshot(); // After second add

// Check for errors
const consoleLogs = await captureBrowserConsoleLogs();
const errors = consoleLogs.filter(log => log.level === 'error');
```

**3. Issue Detection**
```javascript
// Agent notices: Input not clearing after add
// Screenshot shows: "Walk dog" still in input field
// Console: No errors
// Analysis: Missing setInput('') call... wait, it's there!
// Deeper analysis: React state update timing issue
```

**4. Fix Generation**
```javascript
// Updated code
const addTodo = () => {
  if (input.trim()) {  // Add validation
    setTodos([...todos, { id: Date.now(), text: input }]);
    setInput('');
  }
};
```

**5. Verification**
```javascript
// Hot reload triggers
// Re-run test
await browserInput({ selector: "input", text: "Do laundry" });
await browserClickElement({ description: "Add button" });
await captureBrowserScreenshot();

// Verify input cleared
const inputValue = await CDP.send('Runtime.evaluate', {
  expression: `document.querySelector('input').value`
});
assert(inputValue === '');

// ✅ Success!
```

---

## Key Innovations

### 1. **Visual AI Element Detection**
Instead of requiring CSS selectors, Jetski can find elements by description:
```javascript
browserClickElement({ description: "Login button" })
// Agent uses DOM analysis + visual understanding to find the button
// Works even if element has no ID/class or changes structure
```

### 2. **Overlay UI During Execution**
Chrome extension shows real-time overlay:
```
┌───────────────────────────┐
│  Jetski Agent Running     │
│  Step 3 of 5              │
│  Clicking login button... │
│  [ ⏸ Pause ] [ ⏹ Stop ]   │
└───────────────────────────┘
```

### 3. **Screenshot-Based Debugging**
Every step captures screenshot:
- User sees exactly what agent saw
- Debug timing issues visually
- Verify UI state before/after actions

### 4. **Async Verification**
Agent doesn't just apply fixes and hope:
```
Fix → Hot Reload → Re-test → Verify → Report
```

### 5. **Multi-Modal Learning**
Agent learns from:
- Console logs (text)
- Network responses (JSON/HTML)
- Screenshots (visual)
- DOM structure (HTML)
- User feedback (corrections)

---

## Competitive Advantages vs Other IDEs

| Feature | Antigravity + Jetski | GitHub Copilot | Cursor |
|---------|---------------------|----------------|--------|
| **Browser Control** | ✅ Full CDP automation | ❌ No | ⚠️ Limited |
| **Visual Testing** | ✅ Screenshots + overlay | ❌ No | ⚠️ Basic |
| **Console Capture** | ✅ Real-time | ❌ No | ⚠️ Manual |
| **Error Detection** | ✅ Automatic | ❌ Manual | ⚠️ Manual |
| **Fix Verification** | ✅ Automatic re-test | ❌ No | ❌ No |
| **Element Detection** | ✅ AI-powered | N/A | N/A |
| **Feedback Loop** | ✅ Closed loop | ❌ Open loop | ⚠️ Semi-closed |
| **Multi-step Testing** | ✅ Full flows | ❌ No | ⚠️ Limited |

---

## Technical Implementation Details

### Port Configuration
- **Default CDP Port:** 9222
- **Warmup Port:** 9223 (avoids conflict)
- **Configurable:** Via `BrowserCdpPortConfig`

### Browser Profile
- **Path:** `~/.gemini/antigravity-browser-profile`
- **Isolation:** Separate from user's main Chrome
- **Persistence:** Maintains state across sessions
- **Reset:** `browserLauncher.resetBrowserOnboarding`

### Platform Support
- **macOS:** Uses `open` command with `--background` flag
- **Linux:** Direct Chrome spawn with `--remote-debugging-port`
- **Windows:** Similar to Linux with Windows paths
- **Remote SSH:** Reverse proxy via `ssh -R`

### Error Handling
```javascript
// Comprehensive diagnostics
{
  platform: "darwin",
  port_in_use: "false",
  chrome_processes_found: "true",
  running_cdp_ports: "9222",
  running_user_data_dirs: "~/.gemini/antigravity-browser-profile",
  launch_command: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome ...",
  error_message: "...",
  error_stack: "..."
}
```

### Telemetry Events
- `BROWSER_LAUNCH_SUCCESS`
- `BROWSER_LAUNCH_FAILURE_DIAGNOSTICS`
- `BROWSER_ACTION_EXECUTED`
- `BROWSER_ERROR_DETECTED`
- `BROWSER_FIX_APPLIED`

---

## Reverse Engineering Insights

### What We Found
1. ✅ **Complete CDP integration** via `chrome-remote-interface`
2. ✅ **15+ browser automation commands** in Jetski
3. ✅ **Chrome extension** (Chrome Web Store, ID: eeijfnjmj...)
4. ✅ **Security allowlist system** with multi-level permissions
5. ✅ **SSH remote support** via reverse tunneling
6. ✅ **Headless warmup** to reduce launch latency
7. ✅ **Screenshot capture** for visual debugging
8. ✅ **Console/network monitoring** for error detection
9. ✅ **Redux state management** for browser state
10. ✅ **React UI components** for each browser action

### What We Couldn't Find
1. ❌ **Chrome extension source code** (not bundled, only Web Store ID)
2. ❌ **AI element detection algorithm** (in 165MB language server binary)
3. ❌ **Visual overlay rendering** (in Chrome extension)
4. ❌ **Exact protocol buffer schemas** (only compiled JS)
5. ❌ **Fix generation logic** (in language server)

### Key Files for Further Analysis
```
/tmp/Antigravity/resources/app/
├── extensions/
│   ├── antigravity-browser-launcher/dist/extension.js (29KB)
│   └── antigravity/dist/extension.js (contains browser allowlist logic)
├── out/
│   ├── jetskiAgent/main.js (7.2MB - browser commands)
│   └── vs/platform/browserOnboarding/static/
│       ├── browserOnboarding.html
│       └── browserLanding.html
└── node_modules/
    └── chrome-remote-interface/ (CDP library)
```

---

## Reproduction Guide

To replicate Antigravity's browser feedback loop:

### 1. **Browser Launch**
```javascript
import { spawn } from 'child_process';

const chromePath = '/usr/bin/google-chrome';
const profilePath = '~/.my-agent-browser';
const cdpPort = 9222;

const chrome = spawn(chromePath, [
  `--remote-debugging-port=${cdpPort}`,
  `--user-data-dir=${profilePath}`,
  '--disable-fre',
  '--no-default-browser-check',
  '--no-first-run'
]);
```

### 2. **CDP Connection**
```javascript
import CDP from 'chrome-remote-interface';

const client = await CDP({ port: 9222 });
const { Page, Runtime, Console, Network, DOM } = client;

await Page.enable();
await Runtime.enable();
await Console.enable();
await Network.enable();
await DOM.enable();
```

### 3. **Event Listeners**
```javascript
Console.messageAdded((event) => {
  console.log(`[${event.message.level}] ${event.message.text}`);
});

Runtime.exceptionThrown((event) => {
  console.error('Exception:', event.exceptionDetails);
});

Network.responseReceived((event) => {
  if (event.response.status >= 400) {
    console.error('Error response:', event.response);
  }
});
```

### 4. **Automation**
```javascript
// Navigate
await Page.navigate({ url: 'http://localhost:3000' });
await Page.loadEventFired();

// Execute JavaScript
await Runtime.evaluate({
  expression: `
    document.querySelector('#email').value = 'test@example.com';
    document.querySelector('button[type="submit"]').click();
  `
});

// Capture screenshot
const screenshot = await Page.captureScreenshot({ format: 'png' });
```

### 5. **AI Integration**
```javascript
// Send context to LLM
const context = {
  consoleLogs: capturedLogs,
  networkErrors: capturedErrors,
  screenshot: screenshotBase64,
  domSnapshot: await Runtime.evaluate({
    expression: 'document.documentElement.outerHTML'
  })
};

const fix = await llm.generateFix(context);

// Apply fix
await fs.writeFile(fix.file, fix.newContent);

// Verify
await Page.reload();
const newLogs = await captureConsoleLogs();
const success = !newLogs.some(log => log.level === 'error');
```

---

## Conclusion

Antigravity's browser integration represents a **paradigm shift** in front-end development tooling:

**Traditional IDE Workflow:**
```
Code → Save → Switch to browser → Refresh → Check console → Find bug → Switch to IDE → Fix → Repeat
```

**Antigravity/Jetski Workflow:**
```
Describe feature → Agent writes code → Agent tests in browser → Agent detects issues → Agent fixes → Agent verifies → Done
```

The **special sauce** is the **closed feedback loop** enabled by:
1. Seamless browser automation via CDP
2. Comprehensive observation (console, network, screenshots, DOM)
3. AI-powered element detection and interaction
4. Automatic fix generation and verification
5. User-friendly permission system

This transforms the IDE from a **code editor** into an **autonomous front-end development agent** that can build, test, debug, and fix UI applications with minimal human intervention.

---

*End of Report*
