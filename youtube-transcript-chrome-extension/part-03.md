# YouTube Transcript Viewer — Part 3: Active Highlighting, Shortcuts, SRT Export & Popup

You've built a transcript viewer that fetches captions, lets you switch languages, search through lines, and copy the text. But right now it's a static document — it doesn't *move* with the video.

Part 3 adds the features that make the transcript feel alive:

- **Active line highlighting** — the transcript highlights the line currently playing, auto-scrolling to keep it in view.
- **Keyboard shortcuts** — `Ctrl+Shift+T` to toggle the panel, `/` to focus search, `Escape` to close.
- **Export as SRT** — download the transcript as a proper SubRip subtitle file.
- **A popup UI** — click the extension icon to see quick stats about the current video (title, duration, language).

By the end of this part, your extension will feel like a polished tool, not a prototype.

## Prerequisites

This part builds directly on Part 2. You should have the working extension from Part 02 already loaded in Chrome.

> [!RECALL]
> What does `chrome.storage.local.set()` do, and what's the difference between it and `chrome.storage.sync`?

*(Answer: It saves key-value pairs to the extension's local storage, persisting across browser restarts. `sync` also syncs across devices via Google account but has a 100KB limit and is slower; `local` has ~100MB and is faster.)*

## Active line highlighting

The most satisfying feature of any transcript viewer is watching the active line scroll into view as the video plays. It's the difference between reading a document and *following* it.

### The polling loop

We need to check the video's current time periodically and highlight the matching transcript line. A `setInterval` is the simplest approach:

```javascript
let highlightInterval = null;

function startHighlighting(panel) {
  stopHighlighting(); // in case it's already running

  highlightInterval = setInterval(() => {
    if (!panelOpen || !panel) return;

    const video = document.querySelector("video");
    if (!video || video.paused) return;

    const currentTime = video.currentTime;
    const lines = panel.querySelectorAll(".transcript-line");

    let activeLine = null;
    let closestLine = null;
    let closestDiff = Infinity;

    lines.forEach((line) => {
      const start = parseFloat(line.dataset.start);
      const textEl = line.querySelector(".text");
      const originalText = textEl.getAttribute("data-original") || "";

      // Remove active class from all lines
      line.classList.remove("active");

      // Check if this line is the active one
      if (currentTime >= start && currentTime < start + (parseFloat(line.dataset.duration) || 10)) {
        activeLine = line;
      }

      // Track the closest line (for when video is paused)
      const diff = Math.abs(currentTime - start);
      if (diff < closestDiff) {
        closestDiff = diff;
        closestLine = line;
      }
    });

    // Highlight the active line, or the closest one if paused
    const targetLine = activeLine || (video.paused ? closestLine : null);
    if (targetLine) {
      targetLine.classList.add("active");

      // Auto-scroll to keep it in view
      const body = panel.querySelector(".transcript-body");
      const lineTop = targetLine.offsetTop - body.offsetTop;
      const lineBottom = lineTop + targetLine.offsetHeight;
      const bodyScroll = body.scrollTop;
      const bodyHeight = body.clientHeight;

      if (lineTop < bodyScroll + 60 || lineBottom > bodyScroll + bodyHeight - 20) {
        targetLine.scrollIntoView({ behavior: "smooth", block: "center" });
      }
    }
  }, 500); // poll every 500ms
}

function stopHighlighting() {
  if (highlightInterval) {
    clearInterval(highlightInterval);
    highlightInterval = null;
  }
}
```

> [!ASIDE]
> Why 500ms? Polling every 100ms would be smoother but wastes CPU — the video updates its time at 60fps, but we only need to check every half-second. The human eye can't perceive a difference between a line being highlighted at 0.3s vs 0.8s. 500ms is the sweet spot: responsive enough to feel live, light enough to not matter.

### Storing duration on each line

We need to know each segment's duration to determine if the current time falls within it. In Part 2, we stored `segment.start` but not `segment.duration`. Let's add it:

```javascript
// In fetchTranscriptForLang, when creating lines:
line.dataset.start = segment.start;
line.dataset.duration = segment.duration; // add this
```

### Starting and stopping the highlight loop

We need to start the loop when the panel opens and stop it when the panel closes or the video changes:

```javascript
// In showTranscriptPanel, after fetching the transcript:
function fetchTranscriptForLang(languageCode, videoId, panel) {
  // ... existing code ...

  chrome.runtime.sendMessage(
    { action: "getTranscript", videoId: videoId, languageCode: languageCode },
    (response) => {
      body.innerHTML = "";

      if (!response || !response.transcript || response.transcript.length === 0) {
        body.innerHTML = "<p>No transcript available for this language.</p>";
        return;
      }

      response.transcript.forEach((segment) => {
        const line = document.createElement("div");
        line.className = "transcript-line";
        line.dataset.start = segment.start;
        line.dataset.duration = segment.duration; // now we have duration
        line.innerHTML = `
          <span class="timestamp">${formatTime(segment.start)}</span>
          <span class="text" data-original="${escapeHtml(segment.text)}">${escapeHtml(segment.text)}</span>
        `;
        // ... existing click handler ...
        body.appendChild(line);
      });

      // Start the highlighting loop
      startHighlighting(panel);
    }
  );
}
```

And in the close handler:

```javascript
document.getElementById("yt-transcript-close").addEventListener("click", () => {
  panel.remove();
  panelOpen = false;
  stopHighlighting(); // stop the polling loop

  chrome.storage.local.set({
    [`panelOpen_${currentVideoId}`]: false
  });
});
```

> [!HEADS-UP]
> Always call `stopHighlighting()` when closing the panel. If you forget, the interval keeps running in the background, polling `video.currentTime` on every page — even when the user has navigated away from the video tab. This is a common source of battery drain in Chrome extensions.

### CSS for the active line

We already have a basic `.active` style from Part 2. Let's make it more visible:

```css
#yt-transcript-panel .transcript-line.active {
  background: #262626;
  border-left: 3px solid #3ea6ff;
  padding-left: 13px; /* account for the border */
}

#yt-transcript-panel .transcript-line.active .timestamp {
  color: #3ea6ff;
}
```

The blue left border and highlighted timestamp make the active line unmistakable. The `border-left` trick avoids shifting other lines — unlike `margin-left`, borders don't affect layout.

## Keyboard shortcuts

Keyboard shortcuts make the extension feel like a real tool. Three shortcuts, three use cases:

- **`Ctrl+Shift+T`** — toggle the transcript panel.
- **`/`** — focus the search box (when the panel is open).
- **`Escape`** — close the panel.

### Registering the global shortcut

Chrome extensions register keyboard shortcuts in the manifest:

```json
{
  "manifest_version": 3,
  "name": "YouTube Transcript Viewer",
  "version": "1.0.0",
  "description": "Shows YouTube video transcripts in a sidebar panel",
  "permissions": ["storage"],
  "host_permissions": ["https://www.youtube.com/"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.youtube.com/watch*"],
      "js": ["content.js"]
    }
  ],
  "commands": {
    "toggle-transcript": {
      "suggested_key": {
        "default": "Ctrl+Shift+T",
        "macos": "Cmd+Shift+T"
      },
      "description": "Toggle transcript panel"
    }
  },
  "action": {
    "default_popup": "popup.html"
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/128.png"
  }
}
```

We added two new fields:

- **`commands`** — registers the `Ctrl+Shift+T` shortcut. Chrome handles the key binding automatically.
- **`action.default_popup`** — declares `popup.html` as the popup shown when you click the extension icon. We'll create this next.

### Listening for the shortcut

The content script listens for the command:

```javascript
// In content.js, add this after the init section:

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === "toggleTranscript") {
    toggleTranscriptPanel();
    sendResponse({ ok: true });
    return true;
  }
});
```

Wait — that's not right. The content script doesn't receive `chrome.runtime.onCommand`. That event fires in the **background service worker**. We need to send a message from the background to the content script:

```javascript
// In background.js, add this:

chrome.commands.onCommand.addListener((command) => {
  if (command === "toggle-transcript") {
    chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
      if (tabs[0]) {
        chrome.tabs.sendMessage(tabs[0].id, { action: "toggleTranscript" });
      }
    });
  }
});
```

The flow is:

1. User presses `Ctrl+Shift+T`.
2. Chrome fires `chrome.commands.onCommand` in the background service worker.
3. The background script finds the active tab and sends a message to its content script.
4. The content script toggles the panel.

> [!ASIDE]
> Why not listen for the shortcut in the content script directly? Content scripts can't register global keyboard shortcuts — they only receive keyboard events from the page itself. The `chrome.commands` API is the only way to register extension-level shortcuts that work even when the extension isn't focused.

### The `/` and `Escape` shortcuts

These are handled in the content script itself, since they only apply when the panel is open:

```javascript
// In content.js, add this to the init section:

document.addEventListener("keydown", (e) => {
  // Escape closes the panel
  if (e.key === "Escape" && panelOpen) {
    e.preventDefault();
    toggleTranscriptPanel();
    return;
  }

  // / focuses the search box when panel is open
  if (e.key === "/" && panelOpen && document.activeElement.tagName !== "INPUT") {
    e.preventDefault();
    const searchInput = document.querySelector("#yt-transcript-panel #transcript-search");
    if (searchInput) {
      searchInput.focus();
    }
  }
});
```

> [!TIP]
> The `document.activeElement.tagName !== "INPUT"` check prevents the `/` shortcut from interfering when the user is already typing in the search box. Without it, pressing `/` would append another `/` to the search text.

## Export as SRT

The SubRip Text (SRT) format is the most widely supported subtitle format. It's plain text, human-readable, and works with virtually every video player.

### The SRT format

An SRT file looks like this:

```
1
00:00:01,000 --> 00:00:04,000
Hello world

2
00:00:04,500 --> 00:00:07,200
Welcome to the video

3
00:00:07,800 --> 00:00:10,000
Let's get started
```

Each entry has:

1. A sequence number.
2. A timestamp line: `HH:MM:SS,mmm --> HH:MM:SS,mmm`.
3. The text (can span multiple lines).
4. A blank line separator.

### The export function

```javascript
function exportSRT(panel) {
  const lines = panel.querySelectorAll(".transcript-line:not(.hidden)");
  const entries = Array.from(lines).map((line) => {
    const start = parseFloat(line.dataset.start);
    const duration = parseFloat(line.dataset.duration) || 10;
    const textEl = line.querySelector(".text");
    const text = textEl.getAttribute("data-original") || "";
    return { start, duration, text };
  });

  const srtContent = entries
    .map((entry, index) => {
      const startTime = formatSRTTime(entry.start);
      const endTime = formatSRTTime(entry.start + entry.duration);
      return `${index + 1}\n${startTime} --> ${endTime}\n${entry.text}`;
    })
    .join("\n\n");

  // Create a download link
  const blob = new Blob([srtContent], { type: "text/plain" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `transcript-${currentVideoId}.srt`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

function formatSRTTime(seconds) {
  const hours = Math.floor(seconds / 3600);
  const mins = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);
  const ms = Math.floor((seconds % 1) * 1000);
  return `${String(hours).padStart(2, "0")}:${String(mins).padStart(2, "0")}:${String(secs).padStart(2, "0")},${String(ms).padStart(3, "0")}`;
}
```

> [!ASIDE]
> Why `URL.createObjectURL` instead of just creating a link? Because the SRT content lives in memory, not on a server. `Blob` creates a virtual file in the browser, and `URL.createObjectURL` gives us a `blob:` URL that points to it. The user clicks the link, and the browser downloads the virtual file. After clicking, we revoke the URL to free memory.

### Adding the export button

We'll add an export button to the header, next to the copy button:

```javascript
// In showTranscriptPanel, update the header HTML:
panel.innerHTML = `
  <div class="panel-header">
    <span>📝 Transcript</span>
    <div class="header-actions">
      <select id="lang-select" title="Caption language">
        <option value="">Loading...</option>
      </select>
      <button id="copy-btn" title="Copy transcript">📋</button>
      <button id="export-btn" title="Export as SRT">💾</button>
      <button class="close-btn" id="yt-transcript-close">✕</button>
    </div>
  </div>
  <!-- ... -->
`;

// Add the export button handler:
panel.querySelector("#export-btn").addEventListener("click", () => {
  exportSRT(panel);
});
```

And the CSS:

```css
#yt-transcript-panel #export-btn {
  background: none;
  border: none;
  color: #aaa;
  font-size: 16px;
  cursor: pointer;
  padding: 4px 6px;
  border-radius: 4px;
}

#yt-transcript-panel #export-btn:hover {
  color: #fff;
  background: #333;
}
```

> [!NOTE]
> The `💾` icon is a visual shorthand for "save/download." It's not a standard Unicode character with a specific meaning — it's just an emoji that most platforms render as a floppy disk. If you prefer, you can use `⬇` or `↓` instead.

## The popup UI

The popup is a small window that appears when you click the extension icon. It shows quick stats about the current video and lets you toggle the transcript without opening the sidebar.

### Creating popup.html

Create a new file called `popup.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      width: 300px;
      padding: 16px;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: #0f0f0f;
      color: #e0e0e0;
      font-size: 14px;
      margin: 0;
    }

    .popup-title {
      font-size: 16px;
      font-weight: 600;
      margin-bottom: 12px;
      color: #fff;
    }

    .popup-section {
      margin-bottom: 16px;
    }

    .popup-label {
      font-size: 12px;
      color: #888;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      margin-bottom: 4px;
    }

    .popup-value {
      font-size: 14px;
      color: #e0e0e0;
      word-break: break-word;
    }

    .popup-button {
      background: #3ea6ff;
      color: #000;
      border: none;
      border-radius: 6px;
      padding: 10px 16px;
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      width: 100%;
      margin-top: 8px;
    }

    .popup-button:hover {
      background: #6bb8ff;
    }

    .popup-button.secondary {
      background: #2a2a2a;
      color: #e0e0e0;
    }

    .popup-button.secondary:hover {
      background: #333;
    }

    .popup-status {
      font-size: 12px;
      color: #888;
      text-align: center;
      padding: 20px 0;
    }

    .popup-error {
      color: #ff6b6b;
    }
  </style>
</head>
<body>
  <div class="popup-title">YouTube Transcript</div>

  <div id="popup-content">
    <div class="popup-status">Loading...</div>
  </div>

  <script src="popup.js"></script>
</body>
</html>
```

The popup is intentionally minimal — no frills, just the essentials. It's a tool, not a dashboard.

### Creating popup.js

The popup script does three things:

1. Finds the active YouTube tab.
2. Sends a message to the content script to get the video info and transcript.
3. Displays the info in the popup.

```javascript
// popup.js

document.addEventListener("DOMContentLoaded", async () => {
  const content = document.getElementById("popup-content");

  try {
    // Find the active tab
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });

    if (!tab || !tab.url || !tab.url.startsWith("https://www.youtube.com/watch")) {
      content.innerHTML = `
        <div class="popup-status">
          Open a YouTube video to see transcript info.
        </div>
      `;
      return;
    }

    // Extract video ID
    const params = new URLSearchParams(new URL(tab.url).search);
    const videoId = params.get("v");

    if (!videoId) {
      content.innerHTML = `
        <div class="popup-status popup-error">
          Could not extract video ID from URL.
        </div>
      `;
      return;
    }

    // Send message to content script
    const response = await chrome.tabs.sendMessage(tab.id, {
      action: "getPopupInfo",
      videoId: videoId
    });

    if (!response || !response.videoId) {
      content.innerHTML = `
        <div class="popup-status popup-error">
          Could not fetch transcript info. Make sure the extension is loaded on the page.
        </div>
      `;
      return;
    }

    // Display the info
    const duration = response.duration
      ? formatDuration(response.duration)
      : "Unknown";

    content.innerHTML = `
      <div class="popup-section">
        <div class="popup-label">Video ID</div>
        <div class="popup-value">${escapeHtml(videoId)}</div>
      </div>
      <div class="popup-section">
        <div class="popup-label">Duration</div>
        <div class="popup-value">${duration}</div>
      </div>
      <div class="popup-section">
        <div class="popup-label">Available Languages</div>
        <div class="popup-value">${response.languages || "None"}</div>
      </div>
      <div class="popup-section">
        <div class="popup-label">Transcript Lines</div>
        <div class="popup-value">${response.lineCount ?? "—"}</div>
      </div>
      <button class="popup-button" id="popup-toggle">Open Transcript</button>
      <button class="popup-button secondary" id="popup-copy">Copy Transcript</button>
    `;

    // Toggle button
    document.getElementById("popup-toggle").addEventListener("click", () => {
      chrome.tabs.sendMessage(tab.id, { action: "toggleTranscript" });
      window.close();
    });

    // Copy button
    document.getElementById("popup-copy").addEventListener("click", async () => {
      const copyResponse = await chrome.tabs.sendMessage(tab.id, {
        action: "copyTranscript",
        videoId: videoId
      });

      if (copyResponse && copyResponse.ok) {
        const btn = document.getElementById("popup-copy");
        btn.textContent = "✓ Copied!";
        btn.style.background = "#4caf50";
        setTimeout(() => {
          btn.textContent = "Copy Transcript";
          btn.style.background = "";
        }, 1500);
      }
    });
  } catch (error) {
    content.innerHTML = `
      <div class="popup-status popup-error">
        Error: ${escapeHtml(error.message)}
      </div>
    `;
  }
});

function formatDuration(seconds) {
  const hours = Math.floor(seconds / 3600);
  const mins = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);

  if (hours > 0) {
    return `${hours}h ${mins}m ${secs}s`;
  }
  return `${mins}m ${secs}s`;
}

function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}
```

> [!ASIDE]
> The popup is a separate HTML file, not injected into the page. It runs in its own browsing context, which means it can't access the page's DOM. That's why it sends messages to the content script instead of reading the transcript directly.

### Adding the popup message handler

The content script needs to handle two new message types from the popup:

```javascript
// In content.js, extend the chrome.runtime.onMessage listener:

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === "toggleTranscript") {
    toggleTranscriptPanel();
    sendResponse({ ok: true });
    return true;
  }

  if (message.action === "getPopupInfo") {
    // Return video info for the popup
    const video = document.querySelector("video");
    const duration = video ? video.duration : null;

    // Count transcript lines (if panel is open)
    const lines = document.querySelectorAll(".transcript-line");
    const lineCount = lines.length;

    sendResponse({
      videoId: currentVideoId,
      duration: duration,
      lineCount: lineCount,
      languages: "See panel"
    });
    return true;
  }

  if (message.action === "copyTranscript") {
    // Copy the current transcript
    const lines = document.querySelectorAll(".transcript-line:not(.hidden)");
    const text = Array.from(lines)
      .map((line) => {
        const time = line.querySelector(".timestamp").textContent;
        const text = line.querySelector(".text").getAttribute("data-original");
        return `[${time}] ${text}`;
      })
      .join("\n");

    navigator.clipboard.writeText(text).then(() => {
      sendResponse({ ok: true });
    }).catch((err) => {
      sendResponse({ ok: false, error: err.message });
    });
    return true;
  }
});
```

> [!NOTE]
> The popup's `chrome.tabs.sendMessage` uses the promise-based version (it returns a promise). The content script's `onMessage` listener must return `true` to keep the channel open for async responses. If you forget `return true`, the popup will receive `undefined`.

## The complete updated files

### Updated `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "YouTube Transcript Viewer",
  "version": "1.0.0",
  "description": "Shows YouTube video transcripts in a sidebar panel",
  "permissions": ["storage"],
  "host_permissions": ["https://www.youtube.com/"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.youtube.com/watch*"],
      "js": ["content.js"]
    }
  ],
  "commands": {
    "toggle-transcript": {
      "suggested_key": {
        "default": "Ctrl+Shift+T",
        "macos": "Cmd+Shift+T"
      },
      "description": "Toggle transcript panel"
    }
  },
  "action": {
    "default_popup": "popup.html"
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/128.png"
  }
}
```

### Updated `background.js`

```javascript
// background.js

let cachedTracks = {}; // videoId → captionTracks array

async function fetchCaptionTracks(videoId) {
  try {
    const response = await fetch(
      "https://www.youtube.com/youtubei/v1/player?key=AIzaSyAO_FJ2SlqU8Q4STEHLGCilw_Y9_11qcW8&prettyPrint=false",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          context: {
            client: {
              clientName: "WEB",
              clientVersion: "2.20250626.01.00",
            },
          },
          videoId: videoId,
        }),
      }
    );

    const data = await response.json();
    const tracks =
      data?.captions?.playerCaptionsTracklistRenderer?.captionTracks;

    if (!tracks || tracks.length === 0) {
      return [];
    }

    return tracks.map((track) => ({
      languageCode: track.languageCode,
      name: track.name?.simpleText || track.languageCode,
      isAutoGenerated: track.kind === "asr",
      baseUrl: track.baseUrl,
    }));
  } catch (error) {
    console.error("Failed to fetch caption tracks:", error);
    return [];
  }
}

async function fetchTranscriptFromApi(videoId, languageCode = "en") {
  try {
    const playerResponse = await fetch(
      "https://www.youtube.com/youtubei/v1/player?key=AIzaSyAO_FJ2SlqU8Q4STEHLGCilw_Y9_11qcW8&prettyPrint=false",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          context: {
            client: {
              clientName: "WEB",
              clientVersion: "2.20250626.01.00",
            },
          },
          videoId: videoId,
        }),
      }
    );

    const playerData = await playerResponse.json();
    const captionTracks =
      playerData?.captions?.playerCaptionsTracklistRenderer?.captionTracks;

    if (!captionTracks || captionTracks.length === 0) {
      return [];
    }

    const track = captionTracks.find((t) =>
      t.languageCode.startsWith(languageCode)
    );

    if (!track) {
      return [];
    }

    const transcriptUrl = track.baseUrl + "&fmt=json3";
    const transcriptResponse = await fetch(transcriptUrl);
    const transcriptData = await transcriptResponse.json();

    const events = transcriptData?.events || [];
    const segments = [];

    for (const event of events) {
      if (!event.segs) continue;

      const text = event.segs
        .map((seg) => seg.utf8 || "")
        .join("")
        .trim();

      if (text) {
        segments.push({
          text: text,
          start: event.tStartMs / 1000,
          duration: event.dDurationMs / 1000,
        });
      }
    }

    return segments;
  } catch (error) {
    console.error("Transcript fetch error:", error);
    return [];
  }
}

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === "getTranscript") {
    const lang = message.languageCode || "en";
    fetchTranscriptFromApi(message.videoId, lang).then((transcript) => {
      sendResponse({ transcript: transcript });
    });
    return true;
  }

  if (message.action === "getCaptionTracks") {
    fetchCaptionTracks(message.videoId).then((tracks) => {
      cachedTracks[message.videoId] = tracks;
      sendResponse({ tracks: tracks });
    });
    return true;
  }

  if (message.action === "setLanguage") {
    chrome.storage.local.set({
      [`lang_${message.videoId}`]: message.languageCode
    });
    sendResponse({ ok: true });
    return true;
  }
});

// Handle keyboard shortcuts
chrome.commands.onCommand.addListener((command) => {
  if (command === "toggle-transcript") {
    chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
      if (tabs[0]) {
        chrome.tabs.sendMessage(tabs[0].id, { action: "toggleTranscript" });
      }
    });
  }
});
```

### Updated `content.js`

```javascript
// content.js

// --- Helpers ---

function getVideoId() {
  const params = new URLSearchParams(window.location.search);
  return params.get("v");
}

function formatTime(seconds) {
  const mins = Math.floor(seconds / 60);
  const secs = Math.floor(seconds % 60);
  return `${mins}:${secs.toString().padStart(2, "0")}`;
}

function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}

function escapeRegex(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

function seekToTime(seconds) {
  const player = document.querySelector("video");
  if (player) {
    player.currentTime = seconds;
    player.play();
  }
}

// --- Panel management ---

let panelOpen = false;
let currentVideoId = getVideoId();
let highlightInterval = null;

function toggleTranscriptPanel() {
  panelOpen = !panelOpen;
  if (panelOpen) {
    showTranscriptPanel();
  } else {
    hideTranscriptPanel();
  }
}

function showTranscriptPanel() {
  hideTranscriptPanel();

  const panel = document.createElement("div");
  panel.id = "yt-transcript-panel";
  panel.innerHTML = `
    <div class="panel-header">
      <span>📝 Transcript</span>
      <div class="header-actions">
        <select id="lang-select" title="Caption language">
          <option value="">Loading...</option>
        </select>
        <button id="copy-btn" title="Copy transcript">📋</button>
        <button id="export-btn" title="Export as SRT">💾</button>
        <button class="close-btn" id="yt-transcript-close">✕</button>
      </div>
    </div>
    <div class="panel-content">
      <div class="search-bar">
        <input type="text" id="transcript-search" placeholder="🔍 Search transcript..." />
      </div>
      <div class="transcript-body">
        <div class="loading">Loading transcript...</div>
      </div>
    </div>
  `;

  document.body.appendChild(panel);

  // Close button
  document.getElementById("yt-transcript-close").addEventListener("click", () => {
    panel.remove();
    panelOpen = false;
    stopHighlighting();

    chrome.storage.local.set({
      [`panelOpen_${currentVideoId}`]: false
    });
  });

  // Load caption tracks
  chrome.runtime.sendMessage(
    { action: "getCaptionTracks", videoId: currentVideoId },
    (response) => {
      const select = panel.querySelector("#lang-select");
      select.innerHTML = "";

      if (!response || !response.tracks || response.tracks.length === 0) {
        const opt = document.createElement("option");
        opt.value = "";
        opt.textContent = "No captions available";
        opt.disabled = true;
        opt.selected = true;
        select.appendChild(opt);
        return;
      }

      chrome.storage.local.get([`lang_${currentVideoId}`], (items) => {
        const savedLang = items[`lang_${currentVideoId}`];

        response.tracks.forEach((track) => {
          const opt = document.createElement("option");
          opt.value = track.languageCode;
          opt.textContent = track.isAutoGenerated
            ? `${track.name} (auto)`
            : track.name;
          opt.selected = track.languageCode === savedLang;
          select.appendChild(opt);
        });

        if (select.value === "") {
          const firstNonAuto = response.tracks.find((t) => !t.isAutoGenerated);
          select.value = firstNonAuto?.languageCode || response.tracks[0].languageCode;
        }
      });

      select.addEventListener("change", (e) => {
        const lang = e.target.value;
        if (lang) {
          fetchTranscriptForLang(lang, currentVideoId, panel);
          chrome.runtime.sendMessage({
            action: "setLanguage",
            videoId: currentVideoId,
            languageCode: lang
          });
        }
      });
    }
  );

  const initialLang = panel.querySelector("#lang-select").value;
  if (initialLang) {
    fetchTranscriptForLang(initialLang, currentVideoId, panel);
  }

  panel.querySelector("#transcript-search").addEventListener("input", (e) => {
    filterTranscript(e.target.value);
  });

  panel.querySelector("#copy-btn").addEventListener("click", () => {
    copyTranscript(panel);
  });

  panel.querySelector("#export-btn").addEventListener("click", () => {
    exportSRT(panel);
  });
}

function hideTranscriptPanel() {
  const existing = document.getElementById("yt-transcript-panel");
  if (existing) existing.remove();
}

function fetchTranscriptForLang(languageCode, videoId, panel) {
  const body = panel.querySelector(".transcript-body");
  body.innerHTML = '<div class="loading">Loading transcript...</div>';

  chrome.runtime.sendMessage(
    { action: "getTranscript", videoId: videoId, languageCode: languageCode },
    (response) => {
      body.innerHTML = "";

      if (!response || !response.transcript || response.transcript.length === 0) {
        body.innerHTML = "<p>No transcript available for this language.</p>";
        return;
      }

      response.transcript.forEach((segment) => {
        const line = document.createElement("div");
        line.className = "transcript-line";
        line.dataset.start = segment.start;
        line.dataset.duration = segment.duration;
        line.innerHTML = `
          <span class="timestamp">${formatTime(segment.start)}</span>
          <span class="text" data-original="${escapeHtml(segment.text)}">${escapeHtml(segment.text)}</span>
        `;
        line.addEventListener("click", () => {
          seekToTime(segment.start);
          panelOpen = false;
          panel.remove();
          stopHighlighting();
        });
        body.appendChild(line);
      });

      startHighlighting(panel);
    }
  );
}

function filterTranscript(query) {
  const lines = document.querySelectorAll("#yt-transcript-panel .transcript-line");
  const lowerQuery = query.toLowerCase();

  lines.forEach((line) => {
    const textEl = line.querySelector(".text");
    const originalText = textEl.getAttribute("data-original") || textEl.textContent;

    if (!query) {
      line.classList.remove("hidden");
      textEl.innerHTML = escapeHtml(originalText);
      return;
    }

    if (originalText.toLowerCase().includes(lowerQuery)) {
      line.classList.remove("hidden");
      const escaped = escapeHtml(originalText);
      const regex = new RegExp(`(${escapeRegex(query)})`, "gi");
      textEl.innerHTML = escaped.replace(regex, '<mark>$1</mark>');
    } else {
      line.classList.add("hidden");
    }
  });
}

function copyTranscript(panel) {
  const lines = panel.querySelectorAll(".transcript-line:not(.hidden)");
  const text = Array.from(lines)
    .map((line) => {
      const time = line.querySelector(".timestamp").textContent;
      const text = line.querySelector(".text").getAttribute("data-original");
      return `[${time}] ${text}`;
    })
    .join("\n");

  navigator.clipboard.writeText(text).then(() => {
    const copyBtn = panel.querySelector("#copy-btn");
    const original = copyBtn.textContent;
    copyBtn.textContent = "✓";
    copyBtn.style.color = "#4caf50";
    setTimeout(() => {
      copyBtn.textContent = original;
      copyBtn.style.color = "";
    }, 1500);
  }).catch((err) => {
    console.error("Failed to copy:", err);
  });
}

function exportSRT(panel) {
  const lines = panel.querySelectorAll(".transcript-line:not(.hidden)");
  const entries = Array.from(lines).map((line) => {
    const start = parseFloat(line.dataset.start);
    const duration = parseFloat(line.dataset.duration) || 10;
    const textEl = line.querySelector(".text");
    const text = textEl.getAttribute("data-original") || "";
    return { start, duration, text };
  });

  const srtContent = entries
    .map((entry, index) => {
      const startTime = formatSRTTime(entry.start);
      const endTime = formatSRTTime(entry.start + entry.duration);
      return `${index + 1}\n${startTime} --> ${endTime}\n${entry.text}`;
    })
    .join("\n\n");

  const blob = new Blob([srtContent], { type: "text/plain" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `transcript-${currentVideoId}.srt`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

function formatSRTTime(seconds) {
  const hours = Math.floor(seconds / 3600);
  const mins = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);
  const ms = Math.floor((seconds % 1) * 1000);
  return `${String(hours).padStart(2, "0")}:${String(mins).padStart(2, "0")}:${String(secs).padStart(2, "0")},${String(ms).padStart(3, "0")}`;
}

function startHighlighting(panel) {
  stopHighlighting();

  highlightInterval = setInterval(() => {
    if (!panelOpen || !panel) return;

    const video = document.querySelector("video");
    if (!video || video.paused) return;

    const currentTime = video.currentTime;
    const lines = panel.querySelectorAll(".transcript-line");

    let activeLine = null;
    let closestLine = null;
    let closestDiff = Infinity;

    lines.forEach((line) => {
      const start = parseFloat(line.dataset.start);
      const duration = parseFloat(line.dataset.duration) || 10;
      const textEl = line.querySelector(".text");
      const originalText = textEl.getAttribute("data-original") || "";

      line.classList.remove("active");

      if (currentTime >= start && currentTime < start + duration) {
        activeLine = line;
      }

      const diff = Math.abs(currentTime - start);
      if (diff < closestDiff) {
        closestDiff = diff;
        closestLine = line;
      }
    });

    const targetLine = activeLine || (video.paused ? closestLine : null);
    if (targetLine) {
      targetLine.classList.add("active");

      const body = panel.querySelector(".transcript-body");
      const lineTop = targetLine.offsetTop - body.offsetTop;
      const lineBottom = lineTop + targetLine.offsetHeight;
      const bodyScroll = body.scrollTop;
      const bodyHeight = body.clientHeight;

      if (lineTop < bodyScroll + 60 || lineBottom > bodyScroll + bodyHeight - 20) {
        targetLine.scrollIntoView({ behavior: "smooth", block: "center" });
      }
    }
  }, 500);
}

function stopHighlighting() {
  if (highlightInterval) {
    clearInterval(highlightInterval);
    highlightInterval = null;
  }
}

// --- Button injection ---

function addTranscriptButton() {
  const sidebar = document.querySelector("#sidebar #secondary");
  if (!sidebar) return;
  if (document.querySelector("#yt-transcript-btn")) return;

  const button = document.createElement("yt-button");
  button.id = "yt-transcript-btn";
  button.style.marginTop = "12px";
  button.innerHTML = `
    <span slot="icon">📝</span>
    Transcript
  `;

  button.addEventListener("click", toggleTranscriptPanel);
  sidebar.prepend(button);
}

// --- URL change detection ---

function onVideoChanged() {
  if (panelOpen) {
    showTranscriptPanel();
  }
  stopHighlighting();
}

function watchForUrlChanges() {
  const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
      if (
        mutation.type === "attributes" &&
        mutation.attributeName === "href"
      ) {
        const newVideoId = getVideoId();
        if (newVideoId && newVideoId !== currentVideoId) {
          currentVideoId = newVideoId;
          onVideoChanged();
        }
      }
    }
  });

  observer.observe(document.querySelector("ytd-app"), {
    attributes: true,
    subtree: true,
  });
}

// --- Keyboard shortcuts ---

document.addEventListener("keydown", (e) => {
  if (e.key === "Escape" && panelOpen) {
    e.preventDefault();
    toggleTranscriptPanel();
    return;
  }

  if (e.key === "/" && panelOpen && document.activeElement.tagName !== "INPUT") {
    e.preventDefault();
    const searchInput = document.querySelector("#yt-transcript-panel #transcript-search");
    if (searchInput) {
      searchInput.focus();
    }
  }
});

// --- Message handlers ---

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === "toggleTranscript") {
    toggleTranscriptPanel();
    sendResponse({ ok: true });
    return true;
  }

  if (message.action === "getPopupInfo") {
    const video = document.querySelector("video");
    const duration = video ? video.duration : null;
    const lines = document.querySelectorAll(".transcript-line");
    const lineCount = lines.length;

    sendResponse({
      videoId: currentVideoId,
      duration: duration,
      lineCount: lineCount,
      languages: "See panel"
    });
    return true;
  }

  if (message.action === "copyTranscript") {
    const lines = document.querySelectorAll(".transcript-line:not(.hidden)");
    const text = Array.from(lines)
      .map((line) => {
        const time = line.querySelector(".timestamp").textContent;
        const text = line.querySelector(".text").getAttribute("data-original");
        return `[${time}] ${text}`;
      })
      .join("\n");

    navigator.clipboard.writeText(text).then(() => {
      sendResponse({ ok: true });
    }).catch((err) => {
      sendResponse({ ok: false, error: err.message });
    });
    return true;
  }
});

// --- Init ---

addTranscriptButton();
watchForUrlChanges();
```

### The popup files

**`popup.html`** — the popup UI (shown above in the popup section).

**`popup.js`** — the popup logic (shown above in the popup section).

## Checkpoint

> [!PREDICT]
> Before you test: what happens to the highlighting interval when you close the panel?

**Run this to verify your work so far:**

1. Reload the extension in `chrome://extensions/`.
2. Open a YouTube video and click the Transcript button.
3. Play the video — the active line should highlight and auto-scroll.
4. Press `Ctrl+Shift+T` — the panel should toggle open/closed.
5. Open the panel, type in the search box, press `/` — the search should re-focus.
6. Press `Escape` — the panel should close.
7. Click the extension icon — the popup should show video stats.
8. Click "Export as SRT" — a `.srt` file should download.

**Likely errors:**

- **No active line highlighting** — make sure the video is playing (paused videos show the closest line but don't auto-scroll).
- **Popup says "Could not fetch transcript info"** — the content script might not have loaded. Refresh the YouTube page.
- **SRT export downloads an empty file** — make sure you haven't filtered the transcript to zero lines with search.
- **Keyboard shortcuts don't work** — check that `commands` is in the manifest and that `chrome.commands.onCommand` is in the background script.

## What's next

You now have a complete, polished transcript viewer with active highlighting, keyboard shortcuts, SRT export, and a popup UI. Part 4 will add:

- **Transcript timeline** — a visual timeline bar showing where captions exist, with clickable segments.
- **Speed control** — adjust playback speed without affecting pitch, with the transcript line updating in real time.
- **Auto-pause** — pause the video when the transcript line ends, so you can read at your own pace.
- **Translation** — fetch translated captions using YouTube's translation endpoint.

## Exercises

- [ ] **Debounced search.** Right now, `filterTranscript` runs on every keystroke. Add a 200ms debounce so it only runs after the user stops typing. Hint: use `setTimeout`/`clearTimeout` with a closure.
- [ ] **Transcript timeline.** Add a horizontal bar below the search box that shows the video's duration, with colored segments indicating where captions exist. Hint: use a `<div>` with `position: relative` and child `<div>` elements with `position: absolute` and `width` set to `(duration / totalDuration) * 100%`.
- [ ] **Auto-pause.** Add a toggle that pauses the video when the current transcript line ends. Hint: in the highlighting interval, check if `currentTime >= start + duration` and call `video.pause()`.
- [ ] **Translation.** Add a "Translate to English" option to the language dropdown. Hint: YouTube's caption URLs support `&tl=en` for translation — append it to the baseUrl before fetching.

## Sources

1. [Chrome Extensions — Commands](https://developer.chrome.com/docs/extensions/reference/commands/) — the `chrome.commands` API for registering keyboard shortcuts.
2. [SubRip Text (SRT) Format](https://www.media.mit.edu/pia/Research/deepview/srtool/srt_subtitles.html) — the SRT subtitle format specification.
3. [Blob API — URL.createObjectURL](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL) — creating downloadable URLs from in-memory data.
4. [Chrome Extensions — Popup](https://developer.chrome.com/docs/extensions/reference/action/) — the popup API for extension icon clicks.
5. [Active Line Highlighting Pattern](https://github.com/la5u/right-side-comments/blob/main/content.js) — demonstrates the `MutationObserver` + interval pattern used for active state management in YouTube extensions.
