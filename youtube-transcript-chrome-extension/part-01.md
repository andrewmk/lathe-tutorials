# Building a YouTube Transcript Viewer Chrome Extension

It's 3 a.m., you're watching a 2-hour lecture, and you've just missed the part where the professor said something important. You fumble through the comments, hoping someone typed it out — or worse, you open a new tab, paste the video URL into some sketchy website, and wait 15 seconds for a transcript that might not even be in the right language.

There's a better way.

YouTube already has the transcript. It's sitting right there in their internal API, the same one that powers the "Show transcript" button beneath every video. We just need to reach into it, pull it out, and display it in our own clean panel — no third-party websites, no API keys, no sign-ups.

This tutorial walks you through building a Chrome extension that adds a "Transcript" button to YouTube's sidebar. Click it, and a panel slides open showing the full video transcript, line by line, with timestamps. You can jump to any point in the video just by clicking a line.

By the end of this part, you'll have a working Chrome extension that displays YouTube transcripts in a sidebar panel — and you'll understand how Chrome extensions talk to web pages, how to make cross-origin requests from a background script, and how YouTube's undocumented InnerTube API works under the hood.

## What you'll build

A Chrome extension that:

- **Detects** when you're on a YouTube watch page (any video URL like `youtube.com/watch?v=...`).
- **Adds a "Transcript" button** to YouTube's sidebar, right next to the "Share" and "Save" buttons.
- **Fetches the transcript** from YouTube's InnerTube API — no API key needed.
- **Displays it in a clean panel** with timestamps, so you can click any line and jump to that moment in the video.

The extension is about 200 lines of JavaScript across three files. Every line will be explained.

## Prerequisites

- **Chrome 88+** (Manifest V3 support — virtually everything is on this now).
- **A YouTube video** to test with — any video with captions/transcripts will do.
- **A text editor** — VS Code, Sublime, even Notepad will work.
- **Zero JavaScript experience required.** We'll explain every concept as we go.

## Creating the extension folder

Every Chrome extension is just a folder of files. Chrome reads the files, packages them up, and runs them. No build tools, no `npm install`, no bundlers. Just files.

Create a new folder called `yt-transcript` and inside it, create three files:

```
yt-transcript/
├── manifest.json
├── content.js
└── background.js
```

We'll also add a simple icon file later, but for now, let's start with the manifest.

## The manifest — the extension's ID card

The `manifest.json` file is the first thing Chrome reads when it loads your extension. It tells Chrome:

- **What your extension is called** (and what version).
- **Which pages it should run on** (we only care about YouTube).
- **What permissions it needs** (we need to talk to YouTube's API).
- **Which files are entry points** (our content script and background script).

Open `manifest.json` and write this:

```json
{
  "manifest_version": 3,
  "name": "YouTube Transcript Viewer",
  "version": "1.0",
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
  "icons": {
    "16": "icon16.png",
    "48": "icon48.png",
    "128": "128.png"
  }
}
```

Let's break down each field:

- **`manifest_version: 3`** — this is the current version of Chrome's extension API. Manifest V2 was deprecated in 2022, so if you see old tutorials using V2, they're outdated.
- **`permissions: ["storage"]`** — gives our extension access to Chrome's local storage, where we'll save settings like which language to use.
- **`host_permissions`** — tells Chrome our extension needs to run on `youtube.com` pages. Without this, our background script can't make requests to YouTube's API.
- **`background.service_worker`** — declares `background.js` as the extension's background script. In Manifest V3, background scripts run as service workers — they're event-driven and sleep when idle.
- **`content_scripts.matches`** — tells Chrome to inject `content.js` into every YouTube watch page (URLs matching `youtube.com/watch*`).
- **`icons`** — optional but recommended. We'll create simple icon images in a moment.

> [!ASIDE]
> Why a service worker instead of a regular background page? Service workers are more efficient — they shut down when not in use and wake up only when needed. The tradeoff is they're stateless: you can't store variables that persist across events. That's why we use `chrome.storage` for persistent data.

### Creating icon files

You don't need to be an artist. Create a folder called `icons` and put three tiny PNG files in it:

- `icon16.png` — 16×16 pixels
- `icon48.png` — 48×48 pixels
- `128.png` — 128×128 pixels

You can use any free icon generator (like [favicon.io](https://favicon.io)) to create simple "T" or transcript-themed icons. For now, even a single 128×128 PNG named `128.png` will work — Chrome will scale it down.

## The content script — talking to YouTube's page

The content script is the part of our extension that lives *inside* the YouTube page. It has direct access to YouTube's DOM (the page structure), can read URLs, and can inject elements. But it **cannot** make cross-origin requests — it's sandboxed to YouTube's origin.

That's where the background script comes in. The content script asks the background script for data, and the background script fetches it from YouTube's API. This is the most common pattern in Chrome extensions: content scripts handle the UI, background scripts handle the network.

### Finding the video ID

Every YouTube watch page has a video ID in the URL. For example:

```
https://www.youtube.com/watch?v=dQw4w9WgXcQ
```

The video ID is `dQw4w9WgXcQ` — 11 characters, alphanumeric plus `-` and `_`. We extract it like this:

```javascript
function getVideoId() {
  const params = new URLSearchParams(window.location.search);
  return params.get("v");
}
```

`window.location.search` is the part of the URL after the `?`. `URLSearchParams` parses it into key-value pairs, and `get("v")` grabs the video ID.

> [!HEADS-UP]
> YouTube is a single-page application. When you click from one video to another, the page doesn't fully reload — it swaps the content. That means our content script runs once when the page first loads, and then sits idle. We need to watch for URL changes so we can fetch a new transcript when the user navigates to a different video.

### Watching for page changes

We use a `MutationObserver` to watch the URL bar. When it changes, we check if we're still on a YouTube watch page:

```javascript
let currentVideoId = getVideoId();

function watchForUrlChanges() {
  const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
      if (mutation.type === "attributes" && mutation.attributeName === "href") {
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
```

This watches the `ytd-app` element (YouTube's root web component) for any attribute changes. When the URL changes, we check if the video ID changed and call `onVideoChanged()` — a function we'll define next.

### Adding the transcript button

YouTube's sidebar has a container called `#transcript-button-container` that we can inject our button into. But first, we need to find the right spot. YouTube's layout uses a component called `ytd-watch-flexy`, and the sidebar lives inside `#secondary`.

Here's the function that creates and inserts our button:

```javascript
function addTranscriptButton() {
  const sidebar = document.querySelector("#sidebar #secondary");
  if (!sidebar) return;

  // Don't add the button twice
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
```

> [!ASIDE]
> `yt-button` is a YouTube web component — it's the same component used by YouTube's own sidebar buttons. Using it instead of a plain `<button>` ensures our button matches YouTube's styling, font, and hover effects automatically.

### The transcript panel

When the user clicks the button, we show (or hide) a panel that slides in from the right. The panel contains:

- A title bar ("Transcript")
- A loading spinner (while we fetch the transcript)
- The transcript text (line by line, with timestamps)
- A close button

```javascript
let panelOpen = false;

function toggleTranscriptPanel() {
  panelOpen = !panelOpen;

  if (panelOpen) {
    showTranscriptPanel();
  } else {
    hideTranscriptPanel();
  }
}

function showTranscriptPanel() {
  // Remove any existing panel
  hideTranscriptPanel();

  const panel = document.createElement("div");
  panel.id = "yt-transcript-panel";
  panel.innerHTML = `
    <div class="panel-header">
      <span>📝 Transcript</span>
      <button class="close-btn" onclick="document.getElementById('yt-transcript-panel').remove(); panelOpen = false;">✕</button>
    </div>
    <div class="panel-content">
      <div class="loading">Loading transcript...</div>
    </div>
  `;

  document.body.appendChild(panel);

  // Fetch the transcript
  fetchTranscript(currentVideoId).then((transcript) => {
    const content = panel.querySelector(".panel-content");
    content.innerHTML = "";

    if (!transcript || transcript.length === 0) {
      content.innerHTML = "<p>No transcript available for this video.</p>";
      return;
    }

    transcript.forEach((segment) => {
      const line = document.createElement("div");
      line.className = "transcript-line";
      line.innerHTML = `
        <span class="timestamp" data-time="${segment.start}">
          ${formatTime(segment.start)}
        </span>
        <span class="text">${escapeHtml(segment.text)}</span>
      `;
      line.addEventListener("click", () => seekToTime(segment.start));
      content.appendChild(line);
    });
  });
}

function hideTranscriptPanel() {
  const existing = document.getElementById("yt-transcript-panel");
  if (existing) existing.remove();
}
```

The panel is a simple `<div>` appended to `document.body`. It has two classes we'll style next:

- `.panel-header` — the title bar with the close button.
- `.panel-content` — the scrollable area where transcript lines appear.

Each transcript line is a `<div>` with a timestamp and text. Clicking a line calls `seekToTime()`, which jumps the video to that moment.

### Styling the panel

There are two ways to add CSS to a Chrome extension. We'll use **dynamic injection** — creating a `<style>` element from JavaScript and appending it to the page. This keeps everything in one file (no separate CSS file needed).

Add this function at the top of `content.js`, right after the helper functions:

```javascript
function injectStyles() {
  const style = document.createElement("style");
  style.textContent = `
    #yt-transcript-panel {
      position: fixed;
      top: 0;
      right: 0;
      width: 420px;
      height: 100vh;
      background: #0f0f0f;
      border-left: 1px solid #303030;
      z-index: 9999;
      display: flex;
      flex-direction: column;
      box-shadow: -4px 0 24px rgba(0, 0, 0, 0.5);
    }

    #yt-transcript-panel .panel-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 16px;
      background: #1a1a1a;
      border-bottom: 1px solid #303030;
      font-size: 15px;
      font-weight: 600;
      color: #fff;
    }

    #yt-transcript-panel .close-btn {
      background: none;
      border: none;
      color: #aaa;
      font-size: 18px;
      cursor: pointer;
      padding: 4px 8px;
    }

    #yt-transcript-panel .close-btn:hover {
      color: #fff;
    }

    #yt-transcript-panel .panel-content {
      flex: 1;
      overflow-y: auto;
      padding: 16px;
      font-size: 14px;
      color: #e0e0e0;
    }

    #yt-transcript-panel .transcript-line {
      padding: 8px 0;
      border-bottom: 1px solid #202020;
      cursor: pointer;
      transition: background 0.15s;
      line-height: 1.5;
    }

    #yt-transcript-panel .transcript-line:hover {
      background: #1a1a1a;
    }

    #yt-transcript-panel .timestamp {
      color: #aaa;
      margin-right: 12px;
      font-size: 13px;
      font-variant-numeric: tabular-nums;
    }

    #yt-transcript-panel .text {
      color: #e0e0e0;
    }

    #yt-transcript-panel .loading {
      color: #888;
      text-align: center;
      padding: 40px;
    }
  `;
  document.head.appendChild(style);
}
```

These styles follow YouTube's dark theme (`#0f0f0f` background, `#e0e0e0` text). The panel is `420px` wide — enough to read comfortably without swallowing the entire page.

Call `injectStyles()` in the init section, before the button is added:

### Seeking to a timestamp

When the user clicks a transcript line, we jump the video to that moment:

```javascript
function seekToTime(seconds) {
  const player = document.querySelector("video");
  if (player) {
    player.currentTime = seconds;
    player.play();
  }
}
```

Simple — just set `video.currentTime` and call `play()`. YouTube's player handles the rest.

### Formatting time

We need to convert seconds into a readable timestamp like `1:23`:

```javascript
function formatTime(seconds) {
  const mins = Math.floor(seconds / 60);
  const secs = Math.floor(seconds % 60);
  return `${mins}:${secs.toString().padStart(2, "0")}`;
}
```

And a helper to prevent HTML injection in transcript text:

```javascript
function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}
```

## The background script — fetching the transcript

Now for the part that does the heavy lifting. The background script makes the actual HTTP requests to YouTube's InnerTube API. Remember: content scripts can't make cross-origin requests, but background scripts can.

### How YouTube's InnerTube API works

YouTube has an internal API called **InnerTube** — the same API that powers the website itself. It's undocumented, but we can use it because:

1. It doesn't require an API key for basic transcript fetching (the WEB client uses a shared key).
2. It doesn't enforce CORS — but our background script doesn't need to, since service workers are exempt from CORS.
3. It's the exact same API your browser calls when you load a YouTube page.

The flow is two steps:

1. **POST to `/youtubei/v1/player`** — asks YouTube for the available caption tracks for a video. The response includes URLs to the actual transcript data.
2. **GET the caption track URL** — fetches the transcript in JSON format.

### Step 1: Getting caption track URLs

```javascript
async function getCaptionTracks(videoId) {
  const response = await fetch("https://www.youtube.com/youtubei/v1/player?key=AIzaSyAO_FJ2SlqU8Q4STEHLGCilw_Y9_11qcW8&prettyPrint=false", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      context: {
        client: {
          clientName: "WEB",
          clientVersion: "2.20250626.01.00",
        },
      },
      videoId: videoId,
    }),
  });

  const data = await response.json();
  const captionTracks =
    data?.captions?.playerCaptionsTracklistRenderer?.captionTracks;

  if (!captionTracks || captionTracks.length === 0) {
    return null;
  }

  // Prefer English, fall back to first available
  const english = captionTracks.find((t) => t.languageCode.startsWith("en"));
  return english || captionTracks[0];
}
```

This POST request sends a JSON body with:

- **`context.client`** — tells YouTube which "client" we're mimicking. `WEB` is the web player client.
- **`videoId`** — the video we want captions for.

The response includes a `captions` object with `captionTracks` — an array of available caption languages, each with a `baseUrl` that points to the actual transcript data.

> [!ASIDE]
> That API key (`AIzaSyAO_FJ2SlqU8Q4STEHLGCilw_Y9_11qcW8`) is the same one YouTube's own web player uses. It's embedded in every YouTube page's HTML, visible in the source. It's not secret — it's just not advertised.

### Step 2: Fetching the transcript

The `baseUrl` from step 1 points to an XML-formatted transcript. We need to request it in JSON format by appending `&fmt=json3`:

```javascript
async function fetchTranscript(videoId) {
  const captionTrack = await getCaptionTracks(videoId);

  if (!captionTrack) {
    return [];
  }

  const transcriptUrl = captionTrack.baseUrl + "&fmt=json3";
  const response = await fetch(transcriptUrl);
  const data = await response.json();

  // Parse the transcript events into a clean array
  const events = data?.events || [];
  const segments = [];

  for (const event of events) {
    if (!event.segs) continue; // skip non-text events

    const text = event.segs
      .map((seg) => seg.utf8 || "")
      .join("")
      .trim();

    if (text) {
      segments.push({
        text: text,
        start: event.tStartMs / 1000, // convert ms → seconds
        duration: event.dDurationMs / 1000,
      });
    }
  }

  return segments;
}
```

Each event in the response has:

- **`tStartMs`** — when this segment starts (in milliseconds).
- **`dDurationMs`** — how long it lasts (in milliseconds).
- **`segs`** — an array of text segments. Some events are just timing markers with no text — we skip those.

We combine the `segs` into a single text string, convert milliseconds to seconds, and return a clean array of `{ text, start, duration }` objects.

### Messaging between content script and background script

Now we need to wire the content script and background script together. The content script sends a message to the background script asking for the transcript, and the background script sends the data back:

```javascript
// In content.js
async function fetchTranscript(videoId) {
  try {
    const response = await chrome.runtime.sendMessage({
      action: "getTranscript",
      videoId: videoId,
    });
    return response.transcript;
  } catch (error) {
    console.error("Failed to fetch transcript:", error);
    return [];
  }
}
```

```javascript
// In background.js
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === "getTranscript") {
    fetchTranscriptFromApi(message.videoId).then((transcript) => {
      sendResponse({ transcript: transcript });
    });
    return true; // keep the message channel open for async response
  }
});
```

> [!TIP]
> The `return true;` in `onMessage` is critical. Without it, Chrome closes the message channel before the async `fetchTranscriptFromApi` call completes, and `sendResponse` becomes a no-op. Returning `true` tells Chrome to keep the channel open until `sendResponse` is called.

The full `background.js`:

```javascript
// background.js

async function fetchTranscriptFromApi(videoId) {
  try {
    // Step 1: Get caption track URLs
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

    // Prefer English, fall back to first available
    const english = captionTracks.find((t) =>
      t.languageCode.startsWith("en")
    );
    const track = english || captionTracks[0];

    // Step 2: Fetch the transcript
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
    fetchTranscriptFromApi(message.videoId).then((transcript) => {
      sendResponse({ transcript: transcript });
    });
    return true; // keep channel open for async response
  }
});
```

## Putting it all together — the complete content.js

Here's the full `content.js`, with all the pieces we built above:

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
      <button class="close-btn" id="yt-transcript-close">✕</button>
    </div>
    <div class="panel-content">
      <div class="loading">Loading transcript...</div>
    </div>
  `;

  document.body.appendChild(panel);

  document.getElementById("yt-transcript-close").addEventListener("click", () => {
    panel.remove();
    panelOpen = false;
  });

  // Fetch transcript from background script
  chrome.runtime.sendMessage(
    { action: "getTranscript", videoId: currentVideoId },
    (response) => {
      const content = panel.querySelector(".panel-content");
      content.innerHTML = "";

      if (!response || !response.transcript || response.transcript.length === 0) {
        content.innerHTML = "<p>No transcript available for this video.</p>";
        return;
      }

      response.transcript.forEach((segment) => {
        const line = document.createElement("div");
        line.className = "transcript-line";
        line.innerHTML = `
          <span class="timestamp" data-time="${segment.start}">
            ${formatTime(segment.start)}
          </span>
          <span class="text">${escapeHtml(segment.text)}</span>
        `;
        line.addEventListener("click", () => seekToTime(segment.start));
        content.appendChild(line);
      });
    }
  );
}

function hideTranscriptPanel() {
  const existing = document.getElementById("yt-transcript-panel");
  if (existing) existing.remove();
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

// --- Init ---

injectStyles();
addTranscriptButton();
watchForUrlChanges();
```

> [!NOTE]
> We use `chrome.runtime.sendMessage` (not the `await` version) in `showTranscriptPanel` because the callback-based version is simpler for this use case — we don't need to `await` the response, we just handle it in the callback. If you prefer the `async/await` style, you can use `chrome.runtime.sendMessage(...).then(...)` instead.

## Loading the extension in Chrome

Now let's test it.

1. Open Chrome and go to `chrome://extensions/`.
2. Enable **Developer mode** (toggle in the top-right corner).
3. Click **Load unpacked** (top-left).
4. Select your `yt-transcript` folder.

Your extension is now loaded. Navigate to any YouTube video page, and you should see a "📝 Transcript" button in the sidebar. Click it, and the transcript panel should slide open.

> [!PREDICT]
> Before you click the button: if the video has captions, you should see the transcript load within 1–2 seconds. If it doesn't load, check the Chrome DevTools console (F12) for errors. Common issues: the video has no captions, or the extension isn't loaded (check `chrome://extensions/`).

## Checkpoint

> [!PREDICT]
> Before you run this: what happens if you click the Transcript button on a video with no captions?

**Run this to verify your work so far:**

1. Load the extension (steps above).
2. Open a YouTube video with captions (try any tutorial or lecture).
3. Click the "📝 Transcript" button.
4. You should see a panel with timestamped lines.
5. Click any line — the video should jump to that moment.

**Likely errors:**

- **Panel shows "No transcript available"** — the video doesn't have captions. Try a different video.
- **Nothing happens when you click the button** — check the DevTools console. You might have a syntax error in `content.js`.
- **Extension won't load** — make sure `manifest.json` is valid JSON. Chrome will tell you exactly what's wrong.

## What's next

In this part, you've built a working transcript viewer. But the panel is static — it doesn't remember your language preference, it doesn't search through the transcript, and it doesn't look great on mobile. Part 2 will add:

- **Language selection** — switch between available caption languages.
- **Search** — type a keyword and jump to matching lines.
- **Copy transcript** — one-click copy of the full text.
- **Persistent settings** — remember your preferences using `chrome.storage`.

## Exercises

- [ ] **Dark mode toggle.** Add a button in the panel header that switches between dark and light themes. Hint: toggle a class on the panel and change the CSS variables.
- [ ] **Auto-open on page load.** Modify the extension to automatically show the transcript when you navigate to a YouTube video. Hint: call `showTranscriptPanel()` after the button is added, but only if the user hasn't dismissed it before.
- [ ] **Keyboard shortcut.** Add a keyboard shortcut (e.g., `Ctrl+Shift+T`) to open/close the transcript panel. Hint: use `chrome.commands` in the manifest and `chrome.runtime.onCommand` in the content script.
- [ ] **Highlight active line.** Make the transcript highlight the line corresponding to the current video position, scrolling to keep it in view. Hint: use `setInterval` to poll `video.currentTime` and compare it to each segment's `start` value.

## Sources

1. [Chrome Extensions — Content Scripts](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts) — official docs on how content scripts interact with web pages.
2. [Chrome Extensions — Cross-Origin Requests](https://developer.chrome.com/docs/extensions/develop/concepts/network-requests) — explains why content scripts can't make cross-origin requests and how background scripts solve this.
3. [InnerTube API Configuration (innertube)](https://github.com/tombulled/innertube/blob/main/innertube/config.py) — the Python library that documents YouTube's InnerTube client configurations, including API keys and client versions.
4. [How ytranscript Works](https://github.com/nadimtuhin/ytranscript/blob/main/HOW_IT_WORKS.md) — detailed breakdown of YouTube's caption API, including the two-step fetch process and response format.
5. [Right Side Comments — content.js](https://github.com/la5u/right-side-comments/blob/main/content.js) — a real Chrome extension that injects into YouTube's DOM, demonstrating the `MutationObserver` pattern for detecting SPA navigation.
