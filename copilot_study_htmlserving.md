# Study: Browser-Side HTML File Serving for Extracted SingleFile Archives

**Date:** April 5, 2026  
**Context:** Supporting local serving of extracted .zip.html files without external tools  
**Status:** Architectural Analysis & Solution Exploration

---

## Problem Statement

**Scenario:**
1. User downloads a SingleFile archive for instagram.com (or similar SPAs)
2. User manually extracts the .zip.html file locally
3. User opens the extracted index.html in the browser
4. The page fails to display properly due to complex linking/relative paths that require HTTP serving

**Challenge:**
- Detect that the loaded HTML came from SingleFile and was extracted
- Auto-serve it via HTTP without relying on external CLI tools or Node.js servers
- Solution must be **browser-side** or use existing browser capabilities

---

## Solution Space Analysis

### Option 1: Browser Extension Background Service Worker (Server)

**Feasibility:** ✅ **HIGHEST VIABLE**

**How it works:**
- SingleFile extension runs a lightweight Node.js-style HTTP server in the background
- When user loads a local file, content script detects it
- Extension routes requests to the background service handling local file serving
- Serves files with proper MIME types, relative path resolution, etc.

**Pros:**
- Full control over serving logic
- Can handle dynamic routing and relative path resolution
- Works with any site structure
- Can detect SingleFile metadata in HTML comments/headers

**Cons:**
- Requires extension to have `file://` access permission
- Each extraction needs explicit path configuration
- Still technically a "server" but internal to the extension

**Browser Support:** Chrome (MV3), Firefox (MV2/MV3)

**Implementation Location:**
- [src/core/bg/index.js](src/core/bg/index.js) - add HTTP server startup
- New module: `src/core/bg/local-file-server.js`
- Content script to intercept `file://` loads

---

### Option 2: File System Access API + Service Worker

**Feasibility:** ⚠️ **MEDIUM - Limited Browser Support**

**How it works:**
- Uses modern `FileSystemAccessAPI` to get handle to extracted folder
- Service Worker intercepts requests to a virtual origin `http://singlefile-local:8080/`
- Requests resolved against the file system handle

**Pros:**
- No server process needed
- Works entirely within browser sandbox
- Modern, future-proof approach

**Cons:**
- Limited to Chrome 86+, Edge 86+
- Requires user permission for each folder
- Complex implementation for relative path resolution
- Firefox support is limited

**Browser Support:** Chrome, Edge (limited)

---

### Option 3: IndexedDB/LocalStorage Serving

**Feasibility:** ❌ **LOW - Impractical**

**Problem:**
- Would require storing entire extracted archive in IndexedDB
- Not suitable for large sites (Instagram archives can be 50MB+)
- Poor performance for streaming files

---

### Option 4: Native Messaging to External Server

**Feasibility:** ✅ **HIGH - But violates "no external tool" constraint**

**How it works:**
- Extension uses `browser.runtime.connectNative()` to communicate with native host
- Native host (Python/Node.js script) runs local server on-demand
- Extension provides file path, native host serves it

**Pros:**
- Clean separation of concerns
- Can be OS-specific optimization
- User has control over port/lifecycle

**Cons:**
- Requires native application setup (violates brief)
- More complex deployment
- Cross-platform challenges

---

### Option 5: Hybrid Approach - Detection + Browser Prompt

**Feasibility:** ✅ **PRAGMATIC**

**How it works:**
1. Content script detects local `file://` load from SingleFile archive
2. Prompts user: "This archive should be served via HTTP. Start local server?"
3. If YES: Extension background worker starts local HTTP server (Option 1)
4. If NO: Load as-is with warnings about broken functionality

**Benefits:**
- User-controlled
- Falls back gracefully
- Clear intent

---

## Detection Strategy: Identifying SingleFile Archives

### Metadata Markers in HTML

SingleFile should embed markers in saved HTML:

```html
<!-- SingleFile metadata -->
<meta name="single-file-original-url" content="https://instagram.com/...">
<meta name="single-file-saved-date" content="2026-04-05T14:59:00Z">
<meta name="single-file-archive-format" content="zip-extracted">
<meta name="single-file-requires-serving" content="true">
<!-- Additional: for extracted archives -->
<meta name="single-file-extracted-root" content=".">
```

### Directory Structure Detection

```
extracted-archive/
├── index.html          ← SingleFile HTML with metadata
├── resources/
│   ├── styles.css
│   ├── scripts.js
│   └── ...
└── images/
    └── ...
```

### Detection Algorithm

1. **When user loads `file:///path/to/index.html`:**
   - Content script reads the HTML (without full page load if possible)
   - Parse `<meta>` tags looking for `single-file-*` markers
   - Check for `single-file-requires-serving` or analyze URL origin

2. **Heuristic Detection:**
   - If domain suggests SPA (instagram.com, twitter.com, etc.)
   - If lots of relative paths/imports detected
   - If HTML contains service worker registration (broken in file://)

---

## Recommended Implementation Path

### **Approach: Extension-Based Local Server (Option 1 + Option 5)**

**Rationale:**
- Simplest for users (no external setup)
- Robust and maintainable
- Leverages extension capabilities
- Best UX with auto-detection + user confirmation

### Architecture:

```
┌─────────────────────────────────────────┐
│  User loads: file:///path/to/index.html │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│    Content Script Detection               │
│  - Detect SingleFile markers              │
│  - Analyze page origin & structure        │
└─────────────────────┬───────────────────┘
                      │
          ┌───────────▼──────────┐
          │ Show User Prompt?    │
          └───────┬──────────────┘
                  │
        ┌─────────▼──────────┐
        │ User accepts       │
        └─────────┬──────────┘
                  │
    ┌─────────────▼────────────────┐
    │ Background Service Worker    │
    │ - Start local HTTP server    │
    │ - Map file path to routes    │
    │ - Handle relative imports    │
    └─────────────┬────────────────┘
                  │
    ┌─────────────▼────────────────┐
    │ Redirect to:                  │
    │ http://localhost:PORT/        │
    │ (shows proper version)        │
    └──────────────────────────────┘
```

---

## Technical Challenges & Solutions

### Challenge 1: Relative Path Resolution

**Problem:** Archive may reference `../resources/style.css` but extraction flattens structure

**Solution:**
- Parse archive metadata to understand original structure
- Rewrite relative paths during serving OR
- Serve from correct directory with proper base href

### Challenge 2: Service Worker / Fetch API in file://

**Problem:** Some features disabled in file:// protocol

**Solution:**
- Don't serve index.html as file://
- Redirect to http://localhost:PORT/ before service worker loads
- Special handling for first-load detection

### Challenge 3: MIME Type Handling

**Problem:** HTTP server must serve correct MIME types for JS/CSS/images

**Solution:**
- Extend Express server in MCP server (already exists!)
- Add static file serving middleware with proper MIME handling

### Challenge 4: Port Management

**Problem:** Finding available port, avoiding conflicts

**Solution:**
- Use port 0 (OS assigns free port)
- Return actual port to extension
- Store in extension state for duration of session

---

## Implementation Requirements

### Required Changes:

1. **Add metadata markers** to saved SingleFile HTML
   - Location: [src/core/bg/editor.js](src/core/bg/editor.js)
   - Add comment tags with SingleFile metadata

2. **Create detection content script** 
   - Location: New file `src/core/content/local-file-detector.js`
   - Runs on `file://` protocol pages
   - Detects SingleFile markers

3. **Local file server module**
   - Location: New file `src/core/bg/local-file-server.js`
   - Express server for serving extracted files
   - Port management and lifecycle

4. **UI prompt/notification**
   - Location: Extend [src/core/bg/index.js](src/core/bg/index.js)
   - User confirmation before starting server

5. **Manifest permissions**
   - Location: [manifest.json](manifest.json)
   - Add: `file:///` access if not present
   - Add: `webRequest` or `declarativeNetRequest` if intercepting

---

## UX Flow

```
User Action → Detection → Prompt → Server Start → Redirect → Serve
    ↓            ↓         ↓         ↓            ↓         ↓
Opens local  Content   "Serve    Background  Redirect  Works
HTML file    script    this?"    worker      URL       properly
             analyzes  dialog    HTTP
             metadata           server
```

**Example user experience:**
1. User extracts `instagram-2026-04-05.zip.html` → gets `instagram/` folder
2. User opens `instagram/index.html` in browser (file:// URL)
3. Notification appears: "This looks like an extracted SingleFile archive. Serve locally?"
4. User clicks "Yes"
5. Page redirects to `http://localhost:12345/` and loads properly with all links working

---

## Potential Issues & Mitigations

| Issue | Mitigation |
|-------|-----------|
| Extension won't load file:// URLs | Use permissions in manifest.json, or detect via content script |
| Port conflicts | Use dynamic port (0), track across sessions |
| Large archives | Stream files, don't load into memory |
| Security (file access) | Restrict to user-selected directories |
| Cross-browser compatibility | Graceful fallback for unsupported browsers |

---

## Success Criteria

- ✅ User extracts Instagram archive and opens locally
- ✅ Extension detects it's a SingleFile archive  
- ✅ User sees prompt (or auto-enables with setting)
- ✅ Page is served via HTTP and works correctly
- ✅ Relative paths resolve properly
- ✅ Images/CSS/JS load without errors
- ✅ Service workers function (if any)

---

## Questions for Implementation

1. Should detection/serving be **automatic** or **opt-in**?
2. Should we store metadata in HTML comments or separate `.singlefile-manifest.json`?
3. Should the server **auto-start** on every file:// load, or only when explicitly enabled?
4. Should we support **multiple archives simultaneously** served on different ports?
5. Should we add a UI panel showing "Locally served files"?

---

## Next Steps

- [ ] Implement metadata markers in saved HTML (step 1)
- [ ] Create detection content script (step 2)
- [ ] Build local file server module (step 3)
- [ ] Add UI prompts and messaging (step 4)
- [ ] Update manifest with required permissions (step 5)
- [ ] Test with real archives (Instagram, Twitter, etc.)

