# YouTube Transcript Viewer — Part 4: Timeline, Speed, Auto-Pause & Translation

Parts 1 through 3 gave you a fully functional transcript viewer with highlighting, search, SRT export, and a popup. But there's one thing every good transcript viewer has that yours is missing: a visual timeline.

Right now, your transcript is a list of lines. The user has no sense of where they are in the video, where the captions are dense, and where there are gaps. A timeline changes everything — it turns a static document into a navigable map.

Part 4 adds four features that transform the transcript from a passive reference into an active viewing tool:

- **Transcript timeline** — a visual bar showing caption density, with clickable segments that jump to that moment.
- **Speed control** — a playback speed slider (0.5× to 2.0×) that updates the transcript in real time without affecting pitch.
- **Auto-pause** — a toggle that pauses the video when each transcript line ends, letting you read at your own pace.
- **Translation** — fetch translated captions using YouTube's built-in translation endpoint, with a dropdown to pick the target language.

By the end of this part, your extension will have features that most commercial transcript viewers don't.

## Prerequisites

This part builds directly on Part 3. You should have the working extension from Part 03 already loaded in Chrome.

> [!RECALL]
> What does `URL.createObjectURL` do, and why do we call `URL.revokeObjectURL` after using it?

*(Answer: It creates a `blob:` URL pointing to an in-memory Blob, allowing us to download in-memory data as a file. We revoke it afterward to free the memory — the blob is only needed while the download is in progress.)*

## The transcript timeline

The timeline is a thin horizontal bar that sits above the transcript text. It shows:

1. **The video's total duration** — the bar spans the full length.
2. **Caption coverage** — segments where captions exist are highlighted.
3. **Current position** — a marker shows where you are in the video.
4. **Click-to-seek** — clicking anywhere on the bar jumps to that moment.

### The timeline structure

We'll add the timeline as a new section in the panel, between the search bar and the transcript body:

```html
<div class="timeline-bar">
  <div class="timeline-track">
    <div class="timeline-captions"></div>
    <div class="timeline-progress"></div>
  </div>
  <div class="timeline-labels">
    <span class="timeline-start">0:00</span>
    <span class="timeline-end">12:34</span>
  </div>
</div>
```

The structure has three layers:

- **`.timeline-track`** — the background track that spans the full duration.
- **`.timeline-captions`** — a set of segments showing where captions exist.
- **`.timeline-progress`** — a marker showing current playback position.

### Building the timeline from transcript data

We need to render the timeline after fetching the transcript. Each caption segment becomes a colored block on the timeline:

```javascript
function renderTimeline(panel, segments) {
  const track = panel.querySelector(".timeline-track");
  const captions = panel.querySelector(".timeline-captions");
  const progress = panel.querySelector(".timeline-progress");
  const startLabel = panel.querySelector(".timeline-start");
  const endLabel = panel.querySelector(".timeline-end");

  if (!segments || segments.length === 0) {
    track.style.display = "none";
    return;
  }

  track.style.display = "block";

  // Calculate total duration from the last segment
  const lastSegment = segments[segments.length - 1];
  const totalDuration = lastSegment.start + lastSegment.duration;

  // Update labels
  startLabel.textContent = formatTime(0);
  endLabel.textContent = formatTime(totalDuration);

  // Clear existing segments
  captions.innerHTML = "";

  // Render caption segments
  segments.forEach((segment) => {
    const block = document.createElement("div");
    block.className = "timeline-segment";
    block.title = `${formatTime(segment.start)} — ${escapeHtml(segment.text.substring(0, 50))}${segment.text.length > 50 ? "..." : ""}`;

    // Calculate position and width as percentages
    const left = (segment.start / totalDuration) * 100;
    const width = (segment.duration / totalDuration) * 100;

    block.style.left = `${left}%`;
    block.style.width = `${width}%`;

    // Click to seek
    block.addEventListener("click", (e) => {
      e.stopPropagation();
      seekToTime(segment.start);
    });

    captions.appendChild(block);
  });

  // Store total duration for progress updates
  track.dataset.duration = totalDuration;
}
```

> [!ASIDE]
> Why use percentage-based positioning instead of pixel values? Because the timeline bar resizes with the panel, and percentages automatically adjust. Pixel values would require a resize observer and recalculation every time the panel width changes.

### The progress marker

The progress marker shows where the user is in the video. It updates in the highlighting interval:

```javascript
function updateTimelineProgress(panel) {
  const track = panel.querySelector(".timeline-track");
  const progress = panel.querySelector(".timeline-progress");
  const totalDuration = parseFloat(track.dataset.duration) || 0;

  if (!totalDuration) return;

  const video = document.querySelector("video");
  if (!video) return;

  const percent = (video.currentTime / totalDuration) * 100;
  progress.style.width = `${percent}%`;
}
```

We call this in the existing `startHighlighting` function, right after updating the active line:

```javascript
// In startHighlighting, after the active line logic:
updateTimelineProgress(panel);
```

### Styling the timeline

```css
#yt-transcript-panel .timeline-bar {
  padding: 8px 16px;
  background: #141414;
  border-bottom: 1px solid #303030;
}

#yt-transcript-panel .timeline-track {
  position: relative;
  height: 6px;
  background: #2a2a2a;
  border-radius: 3px;
  cursor: pointer;
  margin-bottom: 4px;
}

#yt-transcript-panel .timeline-track:hover .timeline-progress {
  background: #3ea6ff;
}

#yt-transcript-panel .timeline-captions {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
}

#yt-transcript-panel .timeline-segment {
  position: absolute;
  top: 0;
  height: 100%;
  background: #3ea6ff44;
  border-radius: 3px;
  pointer-events: auto;
  cursor: pointer;
  transition: background 0.15s;
}

#yt-transcript-panel .timeline-segment:hover {
  background: #3ea6ff88;
}

#yt-transcript-panel .timeline-progress {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: #3ea6ff;
  border-radius: 3px;
  transition: width 0.1s linear;
}

#yt-transcript-panel .timeline-labels {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #666;
  font-variant-numeric: tabular-nums;
}
```

> [!HEADS-UP]
> The timeline segments use `pointer-events: none` on the container and `pointer-events: auto` on the individual segments. This prevents clicks on the background from seeking — only clicks on the caption segments or the progress track itself trigger seeking. Without this, clicking the empty space between segments would also seek, which is confusing.

### Clicking the timeline to seek

We need to handle clicks on the timeline track itself (not just the segments):

```javascript
function setupTimelineSeek(panel) {
  const track = panel.querySelector(".timeline-track");

  track.addEventListener("click", (e) => {
    const rect = track.getBoundingClientRect();
    const clickX = e.clientX - rect.left;
    const percent = clickX / rect.width;
    const totalDuration = parseFloat(track.dataset.duration) || 0;
    const seekTime = percent * totalDuration;

    seekToTime(seekTime);
  });
}
```

Call this after rendering the timeline:

```javascript
// In fetchTranscriptForLang, after renderTimeline:
renderTimeline(panel, response.transcript);
setupTimelineSeek(panel);
```

## Speed control

YouTube's video player supports playback speed adjustment via the `playbackRate` property. The extension adds a slider to the panel header so the user can adjust speed without opening YouTube's settings menu.

### The speed slider

We'll add a speed control to the header, next to the language dropdown:

```javascript
// In showTranscriptPanel, update the header HTML:
panel.innerHTML = `
  <div class="panel-header">
    <span>📝 Transcript</span>
    <div class="header-actions">
      <select id="lang-select" title="Caption language">
        <option value="">Loading...</option>
      </select>
      <div class="speed-control">
        <label for="speed-select">Speed:</label>
        <select id="speed-select">
          <option value="0.5">0.5×</option>
          <option value="0.75">0.75×</option>
          <option value="1" selected>1×</option>
          <option value="1.25">1.25×</option>
          <option value="1.5">1.5×</option>
          <option value="2">2×</option>
        </select>
      </div>
      <button id="copy-btn" title="Copy transcript">📋</button>
      <button id="export-btn" title="Export as SRT">💾</button>
      <button class="close-btn" id="yt-transcript-close">✕</button>
    </div>
  </div>
  <!-- ... -->
`;
```

### The speed change handler

When the user changes the speed, we update the video's `playbackRate`:

```javascript
function setupSpeedControl(panel) {
  const speedSelect = panel.querySelector("#speed-select");

  // Get the video element
  const video = document.querySelector("video");
  if (!video) return;

  // Set initial speed from saved preference
  chrome.storage.local.get(["transcriptSpeed"], (items) => {
    const savedSpeed = items.transcriptSpeed;
    if (savedSpeed) {
      speedSelect.value = savedSpeed;
      video.playbackRate = parseFloat(savedSpeed);
    }
  });

  speedSelect.addEventListener("change", (e) => {
    const speed = parseFloat(e.target.value);
    video.playbackRate = speed;

    // Save preference
    chrome.storage.local.set({ transcriptSpeed: speed.toString() });
  });
}
```

> [!ASIDE]
> Why store the speed in `chrome.storage.local` instead of just keeping it in memory? Because the user might close the panel, navigate to another video, and come back. The speed preference should persist across sessions. Plus, if Chrome updates the page (which it sometimes does), the in-memory state is lost.

### CSS for the speed control

```css
#yt-transcript-panel .speed-control {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
  color: #aaa;
}

#yt-transcript-panel #speed-select {
  background: #2a2a2a;
  color: #e0e0e0;
  border: 1px solid #404040;
  border-radius: 4px;
  padding: 4px 6px;
  font-size: 13px;
  cursor: pointer;
}

#yt-transcript-panel #speed-select:hover {
  border-color: #606060;
}
```

### The pitch correction note

YouTube's web player applies pitch correction automatically when you change `playbackRate`. This means the audio doesn't sound like a chipmunk at 2× speed or like a giant at 0.5×. This is handled by the browser's audio pipeline — you don't need to do anything special.

> [!NOTE]
> Not all browsers support pitch correction. Chrome and Edge do. Firefox does not. Safari's behavior varies by version. If you want to warn users, you can check for pitch support: `if (!audioContext.createGain().playbackRate) { /* no pitch correction */ }`. But for a personal extension, it's probably not worth the complexity.

## Auto-pause

Auto-pause is the feature that turns a transcript from a reference into a study tool. When enabled, the video pauses at the end of each line, letting you read at your own pace. When you're ready for the next line, you press space or click to continue.

### The auto-pause toggle

We'll add an auto-pause toggle to the header, next to the speed control:

```javascript
// In showTranscriptPanel, update the header HTML:
panel.innerHTML = `
  <div class="panel-header">
    <span>📝 Transcript</span>
    <div class="header-actions">
      <select id="lang-select" title="Caption language">
        <option value="">Loading...</option>
      </select>
      <div class="speed-control">
        <label for="speed-select">Speed:</label>
        <select id="speed-select">
          <option value="0.5">0.5×</option>
          <option value="0.75">0.75×</option>
          <option value="1" selected>1×</option>
          <option value="1.25">1.25×</option>
          <option value="1.5">1.5×</option>
          <option value="2">2×</option>
        </select>
      </div>
      <div class="autopause-control">
        <label>
          <input type="checkbox" id="autopause-toggle" />
          Auto-pause
        </label>
      </div>
      <button id="copy-btn" title="Copy transcript">📋</button>
      <button id="export-btn" title="Export as SRT">💾</button>
      <button class="close-btn" id="yt-transcript-close">✕</button>
    </div>
  </div>
  <!-- ... -->
`;
```

### The auto-pause logic

The auto-pause logic runs in the highlighting interval. When the current time passes the end of the active segment, we pause the video:

```javascript
let autoPauseEnabled = false;
let lastPausedSegmentEnd = null;

function setupAutoPause(panel) {
  const toggle = panel.querySelector("#autopause-toggle");

  // Load saved preference
  chrome.storage.local.get(["autoPauseEnabled"], (items) => {
    autoPauseEnabled = items.autoPauseEnabled === true;
    toggle.checked = autoPauseEnabled;
  });

  toggle.addEventListener("change", (e) => {
    autoPauseEnabled = e.target.checked;
    chrome.storage.local.set({ autoPauseEnabled: autoPauseEnabled });

    // If enabling, resume playback (the user probably paused it themselves)
    if (autoPauseEnabled) {
      const video = document.querySelector("video");
      if (video && video.paused) {
        video.play();
      }
    }
  });
}
```

And in the highlighting interval, we add the auto-pause check:

```javascript
// In startHighlighting, inside the interval callback:
let autoPauseEnabled = false;
let lastPausedSegmentEnd = null;

// Load preference
chrome.storage.local.get(["autoPauseEnabled"], (items) => {
  autoPauseEnabled = items.autoPauseEnabled === true;
});

highlightInterval = setInterval(() => {
  if (!panelOpen || !panel) return;

  const video = document.querySelector("video");
  if (!video || video.paused) {
    // If auto-pause is enabled and video is paused, reset the tracking
    if (autoPauseEnabled && video) {
      lastPausedSegmentEnd = null;
    }
    return;
  }

  const currentTime = video.currentTime;
  const lines = panel.querySelectorAll(".transcript-line");

  let activeLine = null;
  let closestLine = null;
  let closestDiff = Infinity;

  lines.forEach((line) => {
    const start = parseFloat(line.dataset.start);
    const duration = parseFloat(line.dataset.duration) || 10;
    const end = start + duration;

    line.classList.remove("active");

    if (currentTime >= start && currentTime < end) {
      activeLine = line;
    }

    const diff = Math.abs(currentTime - start);
    if (diff < closestDiff) {
      closestDiff = diff;
      closestLine = line;
    }

    // Auto-pause: if we've passed the end of a segment and haven't paused yet
    if (autoPauseEnabled && currentTime >= end && lastPausedSegmentEnd !== end) {
      video.pause();
      lastPausedSegmentEnd = end;
    }
  });

  // If the user resumes playback, reset the auto-pause tracker
  if (!video.paused && lastPausedSegmentEnd !== null) {
    // Find the segment that corresponds to the current time
    lines.forEach((line) => {
      const start = parseFloat(line.dataset.start);
      const duration = parseFloat(line.dataset.duration) || 10;
      const end = start + duration;
      if (currentTime >= start && currentTime < end) {
        lastPausedSegmentEnd = null; // reset, user is actively watching
      }
    });
  }

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

  updateTimelineProgress(panel);
}, 500);
```

> [!ASIDE]
> Why track `lastPausedSegmentEnd` instead of just checking `currentTime > end`? Because without the tracker, the auto-pause would fire on every interval tick after passing the segment end, calling `video.pause()` repeatedly. Calling `pause()` on an already-paused video is a no-op, but it's still wasteful and can trigger unnecessary event listeners. The tracker ensures we only pause once per segment.

### CSS for the auto-pause control

```css
#yt-transcript-panel .autopause-control {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
  color: #aaa;
}

#yt-transcript-panel #autopause-toggle {
  accent-color: #3ea6ff;
  cursor: pointer;
}

#yt-transcript-panel .autopause-control label {
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
}
```

## Translation

YouTube offers built-in translation for captions. When a video has English captions, you can often get them translated into dozens of other languages. The translation endpoint is part of the same caption URL we already use.

### How YouTube translation works

YouTube's caption URLs support a `tl=` parameter for translation:

```
https://www.youtube.com/api/timedtext?v=VIDEO_ID&lang=en&fmt=json3&tl=es
```

The `tl=es` parameter tells YouTube to translate the captions to Spanish. The source language (the `lang` parameter) is the original caption language, and `tl` is the target translation.

### The translation endpoint

We need to modify the transcript fetch to support translation. The caption track URL from the InnerTube API already includes the source language. We just need to append `&tl=TARGET_LANG`:

```javascript
async function fetchTranscriptFromApi(videoId, languageCode = "en", translateTo = null) {
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

    // Find the source language track
    const sourceTrack = captionTracks.find((t) =>
      t.languageCode.startsWith(languageCode)
    );

    if (!sourceTrack) {
      return [];
    }

    // Build the transcript URL
    let transcriptUrl = sourceTrack.baseUrl;

    // Strip any existing fmt parameter
    transcriptUrl = transcriptUrl.replace(/&fmt=[a-z0-9]+/, "");

    // Add translation if requested
    if (translateTo) {
      transcriptUrl += `&fmt=json3&tl=${translateTo}`;
    } else {
      transcriptUrl += "&fmt=json3";
    }

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
```

> [!HEADS-UP]
> YouTube's translation endpoint has a limitation: it only translates from the original caption language. If the video has English captions, you can translate to Spanish, French, German, etc. But if the video has no English captions, translation won't work — you need to start from an existing caption track.
>
> Also, auto-generated captions (ASR) often don't support translation. Translation is typically only available for manually created captions. If translation fails, the API returns the original text — there's no error code.

### The translation dropdown

We'll add a translation option to the header, next to the language selector:

```javascript
// In showTranscriptPanel, after loading caption tracks:
chrome.runtime.sendMessage(
  { action: "getCaptionTracks", videoId: currentVideoId },
  (response) => {
    const select = panel.querySelector("#lang-select");
    const translateSelect = panel.querySelector("#translate-select");
    select.innerHTML = "";
    translateSelect.innerHTML = "";

    if (!response || !response.tracks || response.tracks.length === 0) {
      const opt = document.createElement("option");
      opt.value = "";
      opt.textContent = "No captions available";
      opt.disabled = true;
      opt.selected = true;
      select.appendChild(opt);
      return;
    }

    // Populate language selector
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

      // Populate translation options (only for manually created captions)
      const sourceLang = select.value || (
        response.tracks.find(t => !t.isAutoGenerated)?.languageCode || response.tracks[0].languageCode
      );

      // Common translation targets
      const translateTargets = [
        { code: "", name: "Original" },
        { code: "es", name: "Spanish" },
        { code: "fr", name: "French" },
        { code: "de", name: "German" },
        { code: "pt", name: "Portuguese" },
        { code: "it", name: "Italian" },
        { code: "ja", name: "Japanese" },
        { code: "ko", name: "Korean" },
        { code: "zh-Hans", name: "Chinese (Simplified)" },
        { code: "zh-Hant", name: "Chinese (Traditional)" },
        { code: "ar", name: "Arabic" },
        { code: "hi", name: "Hindi" },
      ];

      translateTargets.forEach((target) => {
        const opt = document.createElement("option");
        opt.value = target.code;
        opt.textContent = target.name;
        translateSelect.appendChild(opt);
      });
    });

    // Language change handler
    select.addEventListener("change", (e) => {
      const lang = e.target.value;
      if (lang) {
        const translateTo = translateSelect.value || null;
        fetchTranscriptForLang(lang, currentVideoId, panel, translateTo);
        chrome.runtime.sendMessage({
          action: "setLanguage",
          videoId: currentVideoId,
          languageCode: lang
        });
      }
    });

    // Translation change handler
    translateSelect.addEventListener("change", (e) => {
      const lang = select.value;
      const translateTo = e.target.value || null;
      if (lang) {
        fetchTranscriptForLang(lang, currentVideoId, panel, translateTo);
      }
    });
  }
);
```

And update the header HTML to include the translation selector:

```javascript
panel.innerHTML = `
  <div class="panel-header">
    <span>📝 Transcript</span>
    <div class="header-actions">
      <select id="lang-select" title="Caption language">
        <option value="">Loading...</option>
      </select>
      <select id="translate-select" title="Translate to">
        <option value="">Original</option>
      </select>
      <div class="speed-control">
        <label for="speed-select">Speed:</label>
        <select id="speed-select">
          <option value="0.5">0.5×</option>
          <option value="0.75">0.75×</option>
          <option value="1" selected>1×</option>
          <option value="1.25">1.25×</option>
          <option value="1.5">1.5×</option>
          <option value="2">2×</option>
        </select>
      </div>
      <div class="autopause-control">
        <label>
          <input type="checkbox" id="autopause-toggle" />
          Auto-pause
        </label>
      </div>
      <button id="copy-btn" title="Copy transcript">📋</button>
      <button id="export-btn" title="Export as SRT">💾</button>
      <button class="close-btn" id="yt-transcript-close">✕</button>
    </div>
  </div>
  <div class="panel-content">
    <div class="timeline-bar">
      <div class="timeline-track" style="display: none;">
        <div class="timeline-captions"></div>
        <div class="timeline-progress"></div>
      </div>
      <div class="timeline-labels">
        <span class="timeline-start">0:00</span>
        <span class="timeline-end">0:00</span>
      </div>
    </div>
    <div class="search-bar">
      <input type="text" id="transcript-search" placeholder="🔍 Search transcript..." />
    </div>
    <div class="transcript-body">
      <div class="loading">Loading transcript...</div>
    </div>
  </div>
`;
```

### The updated fetch function

The `fetchTranscriptForLang` function now accepts an optional `translateTo` parameter:

```javascript
function fetchTranscriptForLang(languageCode, videoId, panel, translateTo = null) {
  const body = panel.querySelector(".transcript-body");
  body.innerHTML = '<div class="loading">Loading transcript...</div>';

  chrome.runtime.sendMessage(
    { action: "getTranscript", videoId: videoId, languageCode: languageCode, translateTo: translateTo },
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

      renderTimeline(panel, response.transcript);
      startHighlighting(panel);
    }
  );
}
```

And the background handler:

```javascript
// In background.js, update the getTranscript handler:
if (message.action === "getTranscript") {
  const lang = message.languageCode || "en";
  const translateTo = message.translateTo || null;
  fetchTranscriptFromApi(message.videoId, lang, translateTo).then((transcript) => {
    sendResponse({ transcript: transcript });
  });
  return true;
}
```

> [!ASIDE]
> Why is translation handled in the background script instead of the content script? Because the translation endpoint requires a cross-origin request to YouTube's API, and content scripts can't make those requests. The background script handles all API calls.

## The complete updated files

### Updated `manifest.json`

No changes from Part 3 — the manifest is already correct.

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

async function fetchTranscriptFromApi(videoId, languageCode = "en", translateTo = null) {
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

    const sourceTrack = captionTracks.find((t) =>
      t.languageCode.startsWith(languageCode)
    );

    if (!sourceTrack) {
      return [];
    }

    // Build the transcript URL
    let transcriptUrl = sourceTrack.baseUrl;

    // Strip any existing fmt parameter
    transcriptUrl = transcriptUrl.replace(/&fmt=[a-z0-9]+/, "");

    // Add translation if requested
    if (translateTo) {
      transcriptUrl += `&fmt=json3&tl=${translateTo}`;
    } else {
      transcriptUrl += "&fmt=json3";
    }

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
    const translateTo = message.translateTo || null;
    fetchTranscriptFromApi(message.videoId, lang, translateTo).then((transcript) => {
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
let autoPauseEnabled = false;
let lastPausedSegmentEnd = null;

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
        <select id="translate-select" title="Translate to">
          <option value="">Original</option>
        </select>
        <div class="speed-control">
          <label for="speed-select">Speed:</label>
          <select id="speed-select">
            <option value="0.5">0.5×</option>
            <option value="0.75">0.75×</option>
            <option value="1" selected>1×</option>
            <option value="1.25">1.25×</option>
            <option value="1.5">1.5×</option>
            <option value="2">2×</option>
          </select>
        </div>
        <div class="autopause-control">
          <label>
            <input type="checkbox" id="autopause-toggle" />
            Auto-pause
          </label>
        </div>
        <button id="copy-btn" title="Copy transcript">📋</button>
        <button id="export-btn" title="Export as SRT">💾</button>
        <button class="close-btn" id="yt-transcript-close">✕</button>
      </div>
    </div>
    <div class="panel-content">
      <div class="timeline-bar">
        <div class="timeline-track" style="display: none;">
          <div class="timeline-captions"></div>
          <div class="timeline-progress"></div>
        </div>
        <div class="timeline-labels">
          <span class="timeline-start">0:00</span>
          <span class="timeline-end">0:00</span>
        </div>
      </div>
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
      const translateSelect = panel.querySelector("#translate-select");
      select.innerHTML = "";
      translateSelect.innerHTML = "";

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
          const translateTo = translateSelect.value || null;
          fetchTranscriptForLang(lang, currentVideoId, panel, translateTo);
          chrome.runtime.sendMessage({
            action: "setLanguage",
            videoId: currentVideoId,
            languageCode: lang
          });
        }
      });

      translateSelect.addEventListener("change", (e) => {
        const lang = select.value;
        const translateTo = e.target.value || null;
        if (lang) {
          fetchTranscriptForLang(lang, currentVideoId, panel, translateTo);
        }
      });
    }
  );

  const initialLang = panel.querySelector("#lang-select").value;
  const initialTranslate = panel.querySelector("#translate-select").value || null;
  if (initialLang) {
    fetchTranscriptForLang(initialLang, currentVideoId, panel, initialTranslate);
  }

  // Setup controls
  setupSpeedControl(panel);
  setupAutoPause(panel);
  setupTimelineSeek(panel);

  // Search
  panel.querySelector("#transcript-search").addEventListener("input", (e) => {
    filterTranscript(e.target.value);
  });

  // Copy
  panel.querySelector("#copy-btn").addEventListener("click", () => {
    copyTranscript(panel);
  });

  // Export
  panel.querySelector("#export-btn").addEventListener("click", () => {
    exportSRT(panel);
  });
}

function hideTranscriptPanel() {
  const existing = document.getElementById("yt-transcript-panel");
  if (existing) existing.remove();
}

function fetchTranscriptForLang(languageCode, videoId, panel, translateTo = null) {
  const body = panel.querySelector(".transcript-body");
  body.innerHTML = '<div class="loading">Loading transcript...</div>';

  chrome.runtime.sendMessage(
    { action: "getTranscript", videoId: videoId, languageCode: languageCode, translateTo: translateTo },
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

      renderTimeline(panel, response.transcript);
      startHighlighting(panel);
    }
  );
}

function renderTimeline(panel, segments) {
  const track = panel.querySelector(".timeline-track");
  const captions = panel.querySelector(".timeline-captions");
  const progress = panel.querySelector(".timeline-progress");
  const startLabel = panel.querySelector(".timeline-start");
  const endLabel = panel.querySelector(".timeline-end");

  if (!segments || segments.length === 0) {
    track.style.display = "none";
    return;
  }

  track.style.display = "block";

  const lastSegment = segments[segments.length - 1];
  const totalDuration = lastSegment.start + lastSegment.duration;

  startLabel.textContent = formatTime(0);
  endLabel.textContent = formatTime(totalDuration);

  captions.innerHTML = "";

  segments.forEach((segment) => {
    const block = document.createElement("div");
    block.className = "timeline-segment";
    block.title = `${formatTime(segment.start)} — ${escapeHtml(segment.text.substring(0, 50))}${segment.text.length > 50 ? "..." : ""}`;

    const left = (segment.start / totalDuration) * 100;
    const width = (segment.duration / totalDuration) * 100;

    block.style.left = `${left}%`;
    block.style.width = `${width}%`;

    block.addEventListener("click", (e) => {
      e.stopPropagation();
      seekToTime(segment.start);
    });

    captions.appendChild(block);
  });

  track.dataset.duration = totalDuration;
}

function setupTimelineSeek(panel) {
  const track = panel.querySelector(".timeline-track");

  track.addEventListener("click", (e) => {
    const rect = track.getBoundingClientRect();
    const clickX = e.clientX - rect.left;
    const percent = clickX / rect.width;
    const totalDuration = parseFloat(track.dataset.duration) || 0;
    const seekTime = percent * totalDuration;

    seekToTime(seekTime);
  });
}

function updateTimelineProgress(panel) {
  const track = panel.querySelector(".timeline-track");
  const progress = panel.querySelector(".timeline-progress");
  const totalDuration = parseFloat(track.dataset.duration) || 0;

  if (!totalDuration) return;

  const video = document.querySelector("video");
  if (!video) return;

  const percent = (video.currentTime / totalDuration) * 100;
  progress.style.width = `${percent}%`;
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

function setupSpeedControl(panel) {
  const speedSelect = panel.querySelector("#speed-select");
  const video = document.querySelector("video");
  if (!video) return;

  chrome.storage.local.get(["transcriptSpeed"], (items) => {
    const savedSpeed = items.transcriptSpeed;
    if (savedSpeed) {
      speedSelect.value = savedSpeed;
      video.playbackRate = parseFloat(savedSpeed);
    }
  });

  speedSelect.addEventListener("change", (e) => {
    const speed = parseFloat(e.target.value);
    video.playbackRate = speed;
    chrome.storage.local.set({ transcriptSpeed: speed.toString() });
  });
}

function setupAutoPause(panel) {
  const toggle = panel.querySelector("#autopause-toggle");

  chrome.storage.local.get(["autoPauseEnabled"], (items) => {
    autoPauseEnabled = items.autoPauseEnabled === true;
    toggle.checked = autoPauseEnabled;
  });

  toggle.addEventListener("change", (e) => {
    autoPauseEnabled = e.target.checked;
    chrome.storage.local.set({ autoPauseEnabled: autoPauseEnabled });

    if (autoPauseEnabled) {
      const video = document.querySelector("video");
      if (video && video.paused) {
        video.play();
      }
    }
  });
}

function startHighlighting(panel) {
  stopHighlighting();

  // Load auto-pause preference
  chrome.storage.local.get(["autoPauseEnabled"], (items) => {
    autoPauseEnabled = items.autoPauseEnabled === true;
  });

  highlightInterval = setInterval(() => {
    if (!panelOpen || !panel) return;

    const video = document.querySelector("video");
    if (!video || video.paused) {
      if (autoPauseEnabled && video) {
        lastPausedSegmentEnd = null;
      }
      return;
    }

    const currentTime = video.currentTime;
    const lines = panel.querySelectorAll(".transcript-line");

    let activeLine = null;
    let closestLine = null;
    let closestDiff = Infinity;

    lines.forEach((line) => {
      const start = parseFloat(line.dataset.start);
      const duration = parseFloat(line.dataset.duration) || 10;
      const end = start + duration;

      line.classList.remove("active");

      if (currentTime >= start && currentTime < end) {
        activeLine = line;
      }

      const diff = Math.abs(currentTime - start);
      if (diff < closestDiff) {
        closestDiff = diff;
        closestLine = line;
      }

      if (autoPauseEnabled && currentTime >= end && lastPausedSegmentEnd !== end) {
        video.pause();
        lastPausedSegmentEnd = end;
      }
    });

    if (!video.paused && lastPausedSegmentEnd !== null) {
      lines.forEach((line) => {
        const start = parseFloat(line.dataset.start);
        const duration = parseFloat(line.dataset.duration) || 10;
        const end = start + duration;
        if (currentTime >= start && currentTime < end) {
          lastPausedSegmentEnd = null;
        }
      });
    }

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

    updateTimelineProgress(panel);
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

No changes from Part 3 — the popup is already correct.

## Checkpoint

> [!PREDICT]
> Before you test: what happens when you click on a gap between caption segments on the timeline?

**Run this to verify your work so far:**

1. Reload the extension in `chrome://extensions/`.
2. Open a YouTube video and click the Transcript button.
3. Play the video — watch the active line highlight and the timeline progress move.
4. Click on the timeline bar — the video should jump to that moment.
5. Change the speed to 1.5× — the transcript should stay in sync.
6. Enable auto-pause — the video should pause at the end of each line.
7. Switch the translation to Spanish (if available) — the transcript should update.

**Likely errors:**

- **Timeline doesn't show** — the video has no captions, so there's nothing to render. Try a different video.
- **Auto-pause doesn't work** — make sure the video is actually playing (not just loaded). Auto-pause only triggers during playback.
- **Translation doesn't work** — the source captions might be auto-generated (ASR), which don't support translation. Try a video with manually created captions.
- **Speed control doesn't affect playback** — check the DevTools console. Some YouTube videos disable programmatic speed changes.

## What's next

You now have a complete, production-quality transcript viewer with timeline, speed control, auto-pause, and translation. There's still room to grow:

- **Chapter markers** — overlay YouTube's chapter data on the timeline for easier navigation.
- **Multiple transcript views** — switch between a flat list, a timeline view, and a split-screen view.
- **Shared transcripts** — generate a shareable link with the transcript embedded, so you can send it to friends.
- **Analytics** — track which parts of videos you spend the most time reading.

## Exercises

- [ ] **Chapter markers.** Fetch YouTube's chapter data (available in `ytInitialPlayerResponse`) and overlay chapter markers on the timeline. Hint: parse `playerOverlays` or `engagementPanels` for chapter information.
- [ ] **Debounced search.** Right now, `filterTranscript` runs on every keystroke. Add a 200ms debounce so it only runs after the user stops typing. Hint: use `setTimeout`/`clearTimeout` with a closure.
- [ ] **Auto-pause with resume key.** Instead of auto-pausing at the end of each line, add a setting to auto-pause only when the user clicks a "pause on line end" button. This gives the user more control.
- [ ] **Timeline tooltip.** Show a tooltip on hover over a timeline segment that displays the segment's text. Hint: use `mouseenter`/`mouseleave` events on the segment elements and position an absolutely-positioned tooltip div.

## Sources

1. [YouTube Player API — playbackRate](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/playbackRate) — the `playbackRate` property for adjusting video speed.
2. [YouTube InnerTube — caption translation](https://github.com/nadimtuhin/ytranscript/blob/main/HOW_IT_WORKS.md) — the caption URL parameters including `tl=` for translation.
3. [SubRip Text (SRT) Format](https://www.media.mit.edu/pia/Research/deepview/srtool/srt_subtitles.html) — the SRT subtitle format specification.
4. [Chrome Extensions — Storage](https://developer.chrome.com/docs/extensions/reference/storage) — the `chrome.storage` API for persistent settings.
5. [Right Side Comments — content.js](https://github.com/la5u/right-side-comments/blob/main/content.js) — demonstrates the `MutationObserver` + interval pattern used for active state management in YouTube extensions.
