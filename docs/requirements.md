# ChatsOCR: Requirements

| | |
|---|---|
| Status | Draft 2, revised after an independent review |
| Date | 2026-11-24 |
| Related | [High-level design](high-level-design.md) · [Output format](output-format.md) · [Background research](research/README.md) |

In this document, **must**, **should** and **may** are used as in RFC 2119. Each requirement has a priority (**M** = Must, **S** = Should, **C** = Could) and a phase (**P1** = first release, **P2** and **P3** = later). Requirement IDs are stable.

---

## 1. Purpose

ChatsOCR turns **screenshots of a chat conversation** into a **structured transcript** that is easy for an AI agent to work with.

The user supplies screenshots of one conversation and says who is on each side of the screen. The program finds the messages and reads them with local OCR. It then produces an ordered transcript of senders, dates and times, system events, media placeholders, emoji and reply links.

The output is a JSON file, which is the source of truth, plus a compact text view generated from it. The user's original example of the text view:

```
0. System: 2026-11-24T08:33
1. Bar: how are you?
2. Yuval: i am great, how are you?
3. Bar: <image> I'm in the gym
0. System: 2026-11-24T16:33
4. Yuval: [reply: 3] you looks beautiful
```

The proposed format ([output-format.md](output-format.md)) keeps the shape of this example. It also adds a few deliberate marks, such as `[date …]` around dates, which the user still needs to approve (§13, Q1).

**Above all, the program must be extensible.** Supporting a new chat app, platform or theme must take only a new plugin class and its tests. **No engine change.** The plugin API, the documentation and the tools must let an AI coding agent write such a plugin from example screenshots.

## 2. Original request and decisions so far

**What the user asked for:**
- Participants are shown by name, and messages are numbered.
- Date separators and system messages appear as `0. System:` lines.
- When the app shows a time for each message, the time appears in a form that cannot be confused with the message text.
- A weekday name means the most recent such day, counted from the screenshot's creation time if known, otherwise from the current local time.
- An image is shown as `<image>`, ideally with a link to an image cropped from the screenshot.
- A reply is linked to the message it quotes, using the visual quote (part of the original text, maybe a thumbnail).
- Emoji reactions can come later.
- The output must be easy for an AI agent to process.
- The user can pick the chat app, or let the program detect it. The program asks only when detection fails.
- No AI agent is called at runtime.
- The program can be extended with one class per chat app × platform × theme, and each class declares whether it can handle a given screenshot.

**Decisions taken in the clarification round on 2026-11-24:**

| Topic | Decision |
|---|---|
| Languages | Hebrew and English, mixed even within one message |
| Input | A batch of overlapping scrolled screenshots becomes one transcript. Later batches can be **appended** to an existing transcript; message IDs stay the same. |
| Privacy | Local and offline only. No cloud OCR. |
| Interface | CLI and Python library now; maybe a small GUI later |
| First apps | WhatsApp, Telegram, Hinge |
| Platforms and themes | Android, iPhone and desktop/web; light and dark. Each app × platform × theme may have its own implementation with its own auto-detection. |
| Chat types | 1:1 chats first. Group chats later, but the design must leave room for them. |
| Phase 1 content | Text, images, replies, times and dates, **emoji inside text (which emoji it is, not just that there is one)**, **media placeholders** |
| Later phases | Emoji reactions; edited, forwarded, deleted and read-status markers |
| Output | JSON as the source of truth, plus a generated text view |
| Uncertainty | Flagged in the output. The program asks only in interactive mode and never blocks otherwise. |
| Plugin technology | Classic computer-vision helpers in the engine. A plugin may ship an optional small local model. |
| Documents | Markdown in `docs/` in this repository |

## 3. Glossary

| Term | Meaning |
|---|---|
| **Screenshot** | One image file: a phone screenshot, or a capture of a desktop or web app window |
| **Batch** | The set of screenshots processed in one run. They all show the same conversation. |
| **Transcript** | The output: an ordered list of *items* plus metadata (`transcript.json`), and the text view generated from it (`transcript.txt`) |
| **Item** | One entry in a transcript: a *message*, a *date separator*, a *system message* or a *gap marker* |
| **Message** | Something a participant sent (a bubble with text and/or media). Has a stable **message ID**. |
| **System item** | A date separator, a system message (encryption notice, Hinge's "You matched", …) or a gap marker. Shown as `0. System:` in the text view. |
| **Side** | Where a message appears on screen: `left` or `right`. Centered elements have side `center`. |
| **Participant** | A person in the chat. The user maps participants to sides. One participant may be **self**, the owner of the device. |
| **Variant** | One look of one app: **app × platform × theme × UI epoch**, e.g. `whatsapp / android / dark / 2024`. The UI epoch separates app redesigns. |
| **Plugin** | A Python class that can read one or more variants: detection, layout recognition, and the app's own text formats. |
| **Engine** | Everything that is not specific to one app: loading, OCR, text direction, emoji, merging, time resolution, replies, IDs and output |
| **Layout (IR)** | What a plugin returns for one screenshot: boxes for elements and their parts, each with a role. IR stands for intermediate representation. |
| **Observation** | One sighting of an item in one screenshot. Screenshots overlap, so one item may have several observations. |
| **Anchor time** | When a screenshot was taken. Relative date labels ("Yesterday") are resolved against it. |
| **Overlay** | A UI element floating over the chat: the pinned date pill, the scroll-to-bottom button, the unread banner, the typing indicator, the keyboard |
| **Flag** | A machine-readable code that marks an uncertain or degraded result |
| **Interactive mode** | The program may ask the user questions (on a terminal, or through an API callback). In non-interactive mode it never asks. |
| **em / dp** | em is the measured height of one line of body text. dp is a density-independent pixel. Plugin geometry uses them so that it works at any resolution. |

## 4. Users and stakeholders

1. **End user.** Runs the CLI or library on their own screenshots and owns the data, which is private.
2. **Downstream AI agent.** Reads `transcript.txt` or `transcript.json`. Needs output that is unambiguous, easy to parse, compact in tokens and honest about uncertainty.
3. **Plugin author.** A person or an AI coding agent adding a new app variant from example screenshots. Needs a small plugin API, numeric inspection tools, visual debugging and an automatic evaluation loop.
4. **Engine maintainer.** Changes the engine without breaking plugins.

## 5. Scope

**In scope for phase 1:**
- 1:1 chats in WhatsApp, Telegram and Hinge on the platforms and themes in §7, with English or Hebrew app interfaces.
- Text, emoji, images and other media placeholders, replies, per-message times, date separators and system messages.
- Merging several overlapping screenshots, and appending to existing transcripts.
- The CLI, the Python API, and tools for plugin developers.

**Out of scope for phase 1:**
- Group chats (P2).
- Reactions (P2).
- Markers: edited, forwarded, deleted, read status, starred (P2). In P1, the visible text of a deleted message, such as "This message was deleted", is output as ordinary text.
- Text formatting.
- Photos of a screen.
- Video or screen-recording input.
- GUI.
- Languages other than Hebrew and English.
- Custom wallpapers and custom bubble colors are handled on a best-effort basis (NFR-ACC-3).

**Never in scope:**
- Calling an LLM or AI agent at runtime.
- Sending any data off the machine.

## 6. Assumptions and constraints

| ID | Constraint |
|---|---|
| CON-1 | Written in Python. Runs on CPython ≥ 3.12 on Windows, macOS and Linux, on CPU only. |
| CON-2 | **No network while processing.** Model files (OCR language data, emoji fonts) are bundled, or installed by an explicit setup command. |
| CON-3 | **No LLM or AI agent at runtime.** Classic computer vision and an OCR engine are allowed. A plugin may ship an optional small local ML model (e.g. an ONNX detector). |
| CON-4 | Input is original screenshots (PNG, JPEG, WebP), not photos of screens. Screenshots that were re-compressed, e.g. forwarded through a messenger, are handled on a best-effort basis. |
| CON-5 | Runtime dependencies use permissive licenses (MIT, BSD or Apache-2.0). No GPL or AGPL code. Proprietary emoji artwork (Apple, WhatsApp, Samsung) is never shipped (FR-EMO-3). |
| CON-6 | The user knows which participant is on which side, or accepts names inferred from the chat header (FR-IN-3). |

---

## 7. Supported variants in phase 1

| App | Android | iOS | Desktop / Web | Themes | App UI language |
|---|---|---|---|---|---|
| WhatsApp | ✔ | ✔ | ✔ (Web and desktop app) | light and dark (default theme) | English, Hebrew (mirrored layout) |
| Telegram | ✔ | ✔ | ✔ (Telegram Desktop) | light and dark (default themes) | English, Hebrew |
| Hinge | ✔ | ✔ | — (no desktop app) | light and dark | English (Hinge has no Hebrew UI) |

- Message text may be Hebrew, English or both, whatever the app's UI language.
- One plugin may cover several variants. A dedicated class for a single variant is allowed and wins over a more general one (FR-PLG-2).

---

## 8. Functional requirements

### 8.1 Input (FR-IN)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-IN-1 | Accept one or more image files and/or directories as one batch of one conversation. Formats: PNG, JPEG, WebP. | M | P1 |
| FR-IN-2 | Accept participant names per side (`--left NAME --right NAME`) and, optionally, which side is the user's own (`--me left\|right`). In append mode, the transcript's stored participants are the default. | M | P1 |
| FR-IN-3 | If names are missing, infer them. The other participant is the chat header's title; self is `Me`. Self's side comes from outgoing evidence (e.g. delivery ticks). <br>• Interactive: confirm with the user. <br>• Non-interactive: use the inferred names with `PARTICIPANTS_INFERRED`. <br>• With no outgoing evidence at all, self is assumed to be on the right, with `SELF_SIDE_UNKNOWN`. | M | P1 |
| FR-IN-4 | Screenshots may be given in any order. The program orders them by content (FR-MRG-1); file order and timestamps are only hints. | M | P1 |
| FR-IN-5 | Append mode: process a new batch into an existing transcript (`--append transcript.json`). New messages may be newer than, older than, or in between existing ones. | M | P1 |
| FR-IN-6 | Optional hints: `--app`, `--platform`, `--theme`, `--ui-lang he\|en`. The UI-language hint reaches the grammar of the plugin's labels and the mirroring decision, and is recorded with each source. | M | P1 |
| FR-IN-7 | Optional explicit capture time for the batch or for each file (`--captured-at [FILE=]ISO`, repeatable), overriding metadata. Optional `--now ISO` pins the fallback "current time" (NFR-DET-1). | S | P1 |
| FR-IN-8 | Never modify input files. | M | P1 |
| FR-IN-9 | Accept HEIC/HEIF input. | C | P2 |

### 8.2 App detection (FR-DET)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-DET-1 | If not told, detect the variant automatically. Every plugin reports, for each screenshot, a confidence from 0 to 1, the variant it matched, and its evidence. | M | P1 |
| FR-DET-2 | Combine the scores over all screenshots in the batch. The app is decided first, then the platform within that app; both are decided once per batch. The theme is decided per screenshot. | M | P1 |
| FR-DET-3 | App decision policy, with configurable thresholds: <br>• **accept**: top score ≥ 0.6 and ahead of the next by ≥ 0.15; <br>• **ambiguous**: otherwise, if the top score ≥ **floor** = 0.25; <br>• **none**: top score below the floor. | M | P1 |
| FR-DET-4 | When the app is **ambiguous**: <br>• Interactive: the user chooses the app from a ranked list with evidence. <br>• Non-interactive: take the top app and flag `DETECTION_AMBIGUOUS`. <br>Once the app is decided, the platform is picked automatically, with `DETECTION_AMBIGUOUS` if it was close. **The program asks only about the app.** | M | P1 |
| FR-DET-5 | When **none**: <br>• Interactive: the user chooses from all installed apps. <br>• Non-interactive: exit with code 3 and a top-3 report. | M | P1 |
| FR-DET-6 | When the user names the app (and optionally the platform), never exit with code 3. Use the best of that app's plugins and flag `DETECTION_MISMATCH` if it scores below the floor. | M | P1 |
| FR-DET-7 | Use file-name hints as a prior, e.g. Samsung's `Screenshot_…_WhatsApp.jpg`. | S | P1 |
| FR-DET-8 | `chatsocr detect FILES --explain` prints every plugin's score and evidence. | M | P1 |

### 8.3 Layout recognition (FR-LAY)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-LAY-1 | Find the app's chrome (status bar, header, compose bar, navigation bar, desktop sidebar) and leave it out of the chat content. | M | P1 |
| FR-LAY-2 | Find the chat elements in reading order: messages, date separators and system messages. | M | P1 |
| FR-LAY-3 | Find overlays (§3) and keep them out of the transcript. <br>• Content they cover is treated as partly hidden, not as complete. <br>• The pinned date pill is used only as date evidence (FR-TIM-1). | M | P1 |
| FR-LAY-4 | Decide each message's side and map it to that side's participant. This must work when the app's UI is mirrored for a right-to-left language: WhatsApp in Hebrew puts outgoing messages on the left. | M | P1 |
| FR-LAY-5 | Record outgoing evidence (e.g. ticks or a "Sent" label) when visible. Use it to infer self's side (FR-IN-3), and to check `--me`; a disagreement is flagged `SELF_SIDE_CONFLICT`. | S | P1 |
| FR-LAY-6 | Handle messages cut off at the screenshot's edges, including a message taller than the screen (cut at both ends): <br>• Pieces seen in different screenshots are joined into one message. <br>• `PARTIAL_MESSAGE` is flagged only if its top or its bottom was never seen. | M | P1 |
| FR-LAY-7 | Find the parts of each message: body text, time, reply quote (quoted sender, quoted text, thumbnail), media (kind, area, duration or file name), and link preview. | M | P1 |
| FR-LAY-8 | Each bubble is one message, including consecutive bubbles from the same sender. | M | P1 |
| FR-LAY-9 | An album (a grid of several images) is one message with several media entries. | S | P1 |
| FR-LAY-10 | Group chats: the sender name and avatar shown on each message. | M | P2 |
| FR-LAY-11 | Reactions; edited, forwarded, deleted, read-status and starred markers. | M | P2 |

### 8.4 Text recognition (FR-OCR)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-OCR-1 | Recognize Hebrew, English and mixed Hebrew and English text, including digits, punctuation, URLs, phone numbers, currency signs (₪ $ €), and Hebrew prefix letters attached to Latin words (`ב-Zoom`). | M | P1 |
| FR-OCR-2 | Output text in **logical Unicode order**, NFC, with no bidi control characters. Record each message's paragraph direction. If the direction cannot be decided from the pixels, flag `BIDI_AMBIGUOUS`. | M | P1 |
| FR-OCR-3 | Join visually wrapped lines with a space, and keep line breaks the sender typed. When it cannot be decided which a break is, keep the newline and flag `LINEBREAK_UNCERTAIN`. | S | P1 |
| FR-OCR-4 | Keep a confidence for each word, and a combined text confidence for each message. | M | P1 |
| FR-OCR-5 | The OCR engine is a replaceable backend behind an interface. The default is a local engine that supports Hebrew and English. | M | P1 |
| FR-OCR-6 | Repair known weak patterns by default; for example, Tesseract reads `ב-Zoom` as `Zoom-1`. An optional second backend may re-read lines that are low-confidence or still suspicious. | S | P1 |
| FR-OCR-7 | Read times and date labels with settings for that role: a character whitelist, single-line mode, and checking against a pattern. | M | P1 |
| FR-OCR-8 | Text formatting (bold, italic, monospace, strikethrough). | C | P3 |

### 8.5 Emoji (FR-EMO)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-EMO-1 | Detect emoji inside text lines, including runs of adjacent emoji (😂😂😂), and emoji-only "jumbo" messages. They must never appear as OCR junk. | M | P1 |
| FR-EMO-2 | Identify each emoji as its Unicode sequence by matching against templates of the **emoji style** the variant uses (WhatsApp, Apple, Google Noto, Samsung, …), on any bubble color, with a confidence. | M | P1 |
| FR-EMO-3 | Get templates without redistributing proprietary artwork: <br>(a) bundle Noto Emoji (OFL/Apache); <br>(b) render emoji fonts installed on the user's machine at runtime (e.g. Apple Color Emoji on macOS); <br>(c) **calibrate per style**: the user screenshots a generated emoji sheet in *any* app that renders that style, and the tool builds the templates. <br>Templates stay in the user's local cache. | M | P1 |
| FR-EMO-4 | An emoji that cannot be identified confidently is written as U+FFFD (`�`) and flagged `EMOJI_UNIDENTIFIED`. The JSON keeps the best guesses, the confidence and the box. | M | P1 |
| FR-EMO-5 | Skin-tone and ZWJ sequences are handled on a best-effort basis. Uncertain cases fall back to the component emoji with `EMOJI_LOW_CONFIDENCE`. | S | P1 |
| FR-EMO-6 | Remember emoji the user labelled in interactive mode. | C | P2 |

### 8.6 Media (FR-MED)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-MED-1 | Recognize media messages and output typed placeholders. <br>**Must:** `image`, `video` (+ duration), `gif`, `sticker`, `voice` (+ duration), `document` (+ name, size), `link` (+ URL, title), `location`. <br>**Should:** `audio`, `contact`, `poll`, `call`, `view-once`. <br>Anything else: `unknown`. <br>`deleted` comes in P2 together with the other markers. | M | P1 |
| FR-MED-2 | Plugins may define app-specific kinds (`x-…`, e.g. `x-hinge-like`). They declare whether the kind has a picture, its attribute order, and the reply-quote labels that refer to it. The engine and the output formats carry these kinds without changes. | M | P1 |
| FR-MED-3 | A caption on a media message is that message's text. | M | P1 |
| FR-MED-4 | Crop each image-like media item (image, video thumbnail, sticker, GIF frame) and save it in `media/` next to the transcript. Link it from both the JSON and the text view. `--no-media` turns this off. | S | P1 |
| FR-MED-5 | If a media item appears in several screenshots, crop it from the most complete observation with nothing covering it. | S | P1 |

### 8.7 Replies (FR-REP)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-REP-1 | Detect reply quotes, inside the bubble or as a separate block above it. Extract the quoted sender, quoted text (and whether it was truncated or partial), quoted media label, and thumbnail. | M | P1 |
| FR-REP-2 | Find the replied-to message among earlier messages, including those from earlier appended batches. Matching combines: <br>• quoted text (fuzzy, truncation-aware); <br>• thumbnail similarity, from **stored image signatures**, so no pixels are needed; <br>• compatible media kind; <br>• in 1:1 chats, a self/other check of the quoted sender; <br>• a small preference for recent messages. | M | P1 |
| FR-REP-3 | Each reply is in one state: <br>**resolved:** an ID and a confidence; `REPLY_LOW_CONFIDENCE` if the confidence is in the lower band. <br>**ambiguous:** ranked candidates, `REPLY_AMBIGUOUS`. <br>**unresolved:** `REPLY_UNRESOLVED`, the target is probably outside the captured range. <br>The quote's content is kept in every state. | M | P1 |
| FR-REP-4 | Keep the quoted snippet separate from the message's own text. | M | P1 |
| FR-REP-5 | In append mode, retry unresolved and ambiguous replies against newly added messages. Resolved replies are never changed. | S | P1 |

### 8.8 Dates and times (FR-TIM)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-TIM-1 | Date separators become system items with a resolved absolute date, keeping the raw label. A pinned date pill only supplies date evidence for the messages above the first separator on that screenshot. | M | P1 |
| FR-TIM-2 | When the app shows a time for each message, attach it to the message. The text view shows it in a fixed place, distinguishable from the message text. | M | P1 |
| FR-TIM-3 | Resolve relative labels against the anchor of **the screenshot that shows the label**: <br>• Today/היום = the anchor date; <br>• Yesterday/אתמול = the day before; <br>• a weekday name (English or Hebrew, full or abbreviated) = the most recent such day **strictly before** the anchor date. | M | P1 |
| FR-TIM-4 | Take the anchor for each screenshot from the first source available: <br>1. user override (FR-IN-7) <br>2. EXIF `DateTimeOriginal` (with offset) <br>3. the file-name timestamp <br>4. XMP creation date <br>5. file modification time (`ANCHOR_LOW_CONFIDENCE`) <br>6. the run's "now" (`ANCHOR_FALLBACK_NOW`) <br>Cross-check against the status-bar clock when it is readable (`ANCHOR_CONFLICT`). Record the source and confidence. | M | P1 |
| FR-TIM-5 | For a date shown without a year, use the latest year that is not after the anchor (`YEAR_INFERRED`). Separators must be **non-decreasing** in (date, time). Equal dates are allowed (several separators can share a day). An inferred year is never changed just to break a tie. | M | P1 |
| FR-TIM-6 | Parse 12-hour and 24-hour clocks, including English and Hebrew AM/PM forms (`לפנה״צ`, `אחה״צ`) and narrow no-break spaces. Output 24-hour times. | M | P1 |
| FR-TIM-7 | Decide day-first versus month-first for numeric dates once per batch: <br>1. the plugin's locale knowledge; <br>2. the transcript's earlier decision; <br>3. heuristics over all dates in the batch. <br>Store the decision in the transcript. When it cannot be decided, flag `DATE_ORDER_AMBIGUOUS`. | S | P1 |
| FR-TIM-8 | A message with no time of its own gets the date of the separator above it and no time of day. Dates never carry across a gap. Times are never invented. Each date and time records its source. | M | P1 |
| FR-TIM-9 | Messages must be non-decreasing in (date, time). When they are not, try the OCR alternatives for easily confused digits: <br>• a fix keeps the raw value and flags `TIME_CORRECTED`; <br>• otherwise flag `TIME_NON_MONOTONIC`. <br>Never correct silently. | S | P1 |
| FR-TIM-10 | All times are naive local wall-clock time (no time zone), written in ISO 8601. | M | P1 |
| FR-TIM-11 | A separator that also shows a time (e.g. Hinge's "Today 8:33 PM") becomes a system item with that time of day. The messages after it get its date but no time of their own. | M | P1 |
| FR-TIM-12 | A date resolved from a relative label or an inferred year is only as confident as its anchor. With an anchor from file mtime or "now", such dates are marked uncertain (`?`). | M | P1 |

### 8.9 Several screenshots and append (FR-MRG)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-MRG-1 | Find the vertical offsets between overlapping screenshots by **image alignment** of the chat area, with chrome and overlays masked out. Each offset must be confirmed by content (a matching message, or matching lines inside a long message). When the evidence is weak, flag it instead of guessing (`OVERLAP_LOW_CONFIDENCE`). | M | P1 |
| FR-MRG-2 | Remove duplicate observations using position and content together. **Messages that truly repeat (e.g. "ok", "ok") must stay separate.** | M | P1 |
| FR-MRG-3 | If a screenshot has no reliable overlap with the others, insert a **gap** item, order the pieces by date and time, and flag the result (`GAP_POSSIBLE`, plus `ORDER_UNCERTAIN` if only capture or input order decided). | M | P1 |
| FR-MRG-4 | When a message is seen more than once, take each field from the best observation: complete and unoccluded first, then higher confidence. Keep every observation in its provenance. | M | P1 |
| FR-MRG-5 | **Stable IDs:** existing message IDs never change. <br>• New messages after the end get the next integers. <br>• Messages before or between existing ones get **dotted IDs** that sort into place (`4.1`, `0.2`). <br>• The order of the `items` array is authoritative. <br>Rules: [output-format §4](output-format.md#4-message-ids). | M | P1 |
| FR-MRG-6 | Appending is idempotent: appending a batch that is already included adds nothing. | M | P1 |
| FR-MRG-7 | **Chat identity is the app plus the set of participants**, not the platform, theme or side. A batch of the same chat from another device or UI language can be appended. On a mismatch: <br>• Interactive: ask the user. <br>• Non-interactive: refuse with exit code 5, unless `--force` is given. | S | P1 |
| FR-MRG-8 | A `renumber` command compacts IDs to 1…N. It writes an ID map and increments `id_epoch`. | S | P1 |
| FR-MRG-9 | A gap that a later append fills is removed, and its ID is recorded as removed. | M | P1 |
| FR-MRG-10 | When an append shows that two pieces separated by a gap were in the wrong order, reorder the items and keep their IDs. | S | P1 |

### 8.10 Output (FR-OUT)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-OUT-1 | Write `transcript.json`, following a versioned JSON Schema shipped with the package. It is the source of truth. | M | P1 |
| FR-OUT-2 | Write `transcript.txt`, generated deterministically from the JSON by the grammar in [output-format.md](output-format.md). It must be compact, parseable with a regular expression, and in the shape of the user's example. | M | P1 |
| FR-OUT-3 | Each message has: <br>• ID, sender and side; <br>• date and time, each with its source; <br>• text and direction; <br>• spans; <br>• media; <br>• reply; <br>• provenance (which screenshot, which box); <br>• confidence and flags. | M | P1 |
| FR-OUT-4 | Store what later appends need without the original images: fingerprints, **image signatures** for media and quote thumbnails, source hashes, anchors, and the versions of the generator, plugin and OCR. | M | P1 |
| FR-OUT-5 | Output files: <br>• `-o DIR` receives `transcript.json`, `transcript.txt` and `media/`. <br>• A non-append run refuses to replace an existing transcript unless `--overwrite` is given. <br>• `--append` writes next to the existing transcript, and combining it with `-o` is a usage error. <br>• Append first backs up `transcript.json`, `transcript.txt` and any media it replaces. | M | P1 |
| FR-OUT-6 | `chatsocr render transcript.json` regenerates the text view without processing images again. | S | P1 |
| FR-OUT-7 | All text output is UTF-8 without a BOM and contains no bidi control characters. The optional display mode `render --isolate` writes bidi isolates to stdout only. | M | P1 |
| FR-OUT-8 | Flags come from a closed, documented list of codes ([output-format §6](output-format.md#6-flag-codes)). Plugins may set only a documented subset. | M | P1 |
| FR-OUT-9 | Debug output (`--debug DIR`): an overlay image for each screenshot showing elements, parts and IDs, plus a dump of each stage. | S | P1 |
| FR-OUT-10 | Version compatibility on append: <br>• the major schema version must be the same; <br>• a newer minor version is refused (exit 6) unless `--force` is given; <br>• an older minor version is upgraded in place; <br>• unknown fields are kept. | M | P1 |

### 8.11 Uncertainty and interaction (FR-UNC)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-UNC-1 | Every uncertain decision adds a flag (code, field, confidence) to the item it affects or to the transcript. The text view shows compact markers for flags below a configurable threshold (default 0.80). | M | P1 |
| FR-UNC-2 | The CLI is interactive when stdin and stdout are terminals, unless `--no-input` is given; `--interactive` forces it. The library never prompts; it calls an optional resolver (FR-API-3). Questions are asked only from the main process. | M | P1 |
| FR-UNC-3 | Non-interactive runs never block. They always finish with output (with flags) or with a documented exit code. | M | P1 |
| FR-UNC-4 | Interactive questions cover at least: <br>• choosing the app when detection is ambiguous or fails; <br>• confirming inferred participants; <br>• an append mismatch. <br>An optional review (`--review`) asks about low-confidence replies, emoji and times. | M (review: S) | P1 |
| FR-UNC-5 | Record user answers in the JSON with source `user`. | S | P1 |

### 8.12 CLI, library and configuration (FR-CLI, FR-API, FR-CFG)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-CLI-1 | `chatsocr parse FILES…` with the options listed in [HLD §8.1](high-level-design.md#81-cli-reference-authoritative), which is the authoritative reference: <br>• participants: `--left/--right/--me` <br>• hints: `--app/--platform/--theme/--ui-lang` <br>• output: `-o`, `--overwrite`, `--no-media`, `--threshold`, `--stdout` <br>• append: `--append`, `--force` <br>• time: `--captured-at`, `--now` <br>• interaction: `--interactive/--no-input/--review` <br>• plugins and config: `--plugin-dir`, `--config` <br>• diagnostics: `--debug`, `--no-cache`, `-q/-v/-vv`, `--log-file` | M | P1 |
| FR-CLI-2 | `chatsocr detect FILES… [--explain]` | M | P1 |
| FR-CLI-3 | `chatsocr render JSON [--threshold X] [--isolate]` | S | P1 |
| FR-CLI-4 | `chatsocr plugin list` shows each plugin's id, variants, source (built-in, installed or local), version and availability, with the reason if unavailable. | M | P1 |
| FR-CLI-5 | `chatsocr emoji sheet`, `emoji calibrate --style S`, `emoji status` (FR-EMO-3) | M | P1 |
| FR-CLI-6 | `chatsocr setup` installs optional model files. It is the only command that may use the network, and only when the user runs it. `chatsocr cache clear` empties the local caches. | S | P1 |
| FR-CLI-7 | Exit codes: <br>0 = success, possibly with flags <br>1 = internal error <br>2 = usage error <br>3 = no plugin matched <br>4 = no screenshot could be parsed <br>5 = append refused because the chat does not match <br>6 = the transcript was written by a newer schema version <br>A one-line summary of the flags goes to stderr. | M | P1 |
| FR-API-1 | `chatsocr.parse(images, participants=…, hints=…, …) -> Transcript`; `chatsocr.append(transcript, images, …) -> Transcript`; `Transcript.to_json()`, `.to_text()`, `.load(path)` | M | P1 |
| FR-API-2 | Images may be paths, bytes, PIL images or NumPy arrays. Capture times, `now`, plugin directories and configuration can be passed in. | S | P1 |
| FR-API-3 | An optional `resolver` object answers interactive questions in library use. Without one, non-interactive policy applies. | S | P1 |
| FR-CFG-1 | Configuration file `<user config dir>/chatsocr/config.toml`, plus `--config FILE`, with this precedence: CLI > environment (`CHATSOCR_<SECTION>_<KEY>`) > `--config` > user config > plugin values > built-in defaults. Every threshold in this document is configurable. The effective values are recorded in the transcript. | S | P1 |
| FR-GUI-1 | A small GUI: drag in screenshots, review the result, fix flagged items. | C | P3 |

### 8.13 Plugin system (FR-PLG): the extensibility requirements

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-PLG-1 | **Supporting a new app, platform, theme or UI epoch needs only a new plugin module (plus fixtures and tests). The engine is not changed.** A CI test checks this by loading plugins from outside the source tree. | M | P1 |
| FR-PLG-2 | A plugin class declares the variants it supports, and may cover several. Each screenshot is routed to a class that claims its detected variant and recognized it, in this order: <br>1. the more specific class (fewer variants); <br>2. local over installed over built-in; <br>3. higher declared priority. <br>A class that did not recognize the screenshot is never chosen. | M | P1 |
| FR-PLG-3 | Each plugin has a cheap `detect` method that returns a confidence, the matched variant and its evidence. | M | P1 |
| FR-PLG-4 | **Plugins do layout and the app's own grammar only.** <br>• Layout: chrome, overlays, and elements with their parts and roles. <br>• Grammar: reading the app's time, date-label and system strings, and its media kinds. <br>The engine does everything that every app shares: OCR, text direction, emoji, merging, IDs, times, replies, crops and output. | M | P1 |
| FR-PLG-5 | The plugin API offers three ways to write a plugin: <br>(a) a whole-screen `parse_screen`; <br>(b) a **step-by-step** base class whose steps can be overridden one at a time, with a declarative base that implements every step for typical bubble layouts; <br>(c) a **sequential** base class, the model the user proposed. Here the author writes only `detect` and `crop_next_element(ctx, area, y, overlays)`, and inherits chrome, overlays, direction and parts. | M | P1 |
| FR-PLG-6 | The engine provides a documented set of computer-vision helpers, so that typical plugin code stays short. They cover: <br>• color clustering and masks <br>• flat and shaded regions (bubbles) <br>• centered chips and centered text lines <br>• textured rectangles and color blobs <br>• keyboard detection <br>• small glyph clusters (times, ticks) <br>• quote bars <br>• icon matching <br>• row profiles <br>• local background color <br>• dp/em conversion | M | P1 |
| FR-PLG-7 | Plugin geometry is given in dp or em. The engine estimates the scale of each screenshot from measured text, so phones and desktop scaling of 100–200% work. A plugin may override the estimate. | M | P1 |
| FR-PLG-8 | Theme colors are data: a palette for each variant, optionally in a TOML file next to the plugin, which a tool can regenerate from fixtures. Gradient colors are supported. | S | P1 |
| FR-PLG-9 | Discovery sources: <br>• the package's built-in plugins; <br>• installed packages, through an entry-point group; <br>• `--plugin-dir` directories; <br>• the user's plugin directory. <br>Plugin directories may contain multi-file packages. | M | P1 |
| FR-PLG-10 | Each plugin declares `api_version`, and incompatible plugins are rejected with a clear message. The API is **provisional (0.x) until the M3 agent pilot ends**; after that, semantic versioning and deprecation warnings apply. | M | P1 |
| FR-PLG-11 | A plugin may declare optional dependencies and model files. If they are missing, the plugin is listed as unavailable with the reason, and nothing crashes. | S | P1 |
| FR-PLG-12 | Importing a plugin is cheap. `detect` stays within budget (default 50 ms for each screenshot and plugin, measured on a downscaled image). The conformance kit enforces both. | S | P1 |
| FR-PLG-13 | App-specific elements, such as Hinge's "liked your prompt" card, are turned by the plugin into ordinary message or system elements. These go through normal extraction (OCR, crops) without engine changes. | M | P1 |
| FR-PLG-14 | Each variant declares: <br>• its emoji style; <br>• whether its UI mirrors in right-to-left languages; <br>• whether its separators carry times; <br>• its numeric date order, if fixed; <br>• its label vocabulary. <br>The engine provides English and Hebrew defaults, and a plugin may extend or restrict them. | M | P1 |
| FR-PLG-15 | The configuration of the declarative base is **typed and validated**: a misspelled field is an error, not a silent default. Every default step documents its algorithm and the fields that drive it. | M | P1 |
| FR-PLG-16 | The box conventions are documented: what an element's box covers, versus its main body (the bubble); where status rows, detached quotes and decorations go. The same conventions apply to plugins and to ground-truth files. | M | P1 |

### 8.14 Tools for plugin developers and AI agents (FR-DEV)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-DEV-1 | `chatsocr plugin new --app X --platform Y --themes light,dark` creates the plugin module, palette file, fixture folders, a test file and an authoring checklist. | M | P1 |
| FR-DEV-2 | Numeric inspection tools, so agents never guess pixel values from images. None of them needs a plugin: <br>• `inspect blocks`: every flat region, chip, text line, textured rectangle and color blob, with box, fill and anchoring; <br>• `inspect ocr`: words and lines with boxes, confidence and direction; <br>• `inspect probe`: color, Lab and ΔE at a point or box; <br>• `inspect palette`, `inspect rows`; <br>• `inspect calibrate`: a palette from fixtures. | M | P1 |
| FR-DEV-3 | `chatsocr debug overlay IMG [--expected]` draws the detected elements, parts and IDs, optionally next to the ground truth. | M | P1 |
| FR-DEV-4 | `chatsocr eval --plugin X [--json]` scores a plugin against its fixtures and gives a machine-readable diff. <br>• Item level, from `expected.txt`: missing, extra and reordered items; text errors; time, date, reply, media and emoji errors. <br>• Layout level, from `expected.json` where present. <br>It exits non-zero below the thresholds. | M | P1 |
| FR-DEV-5 | A conformance kit, shipped as a pytest plugin, that every plugin must pass: <br>• metadata; <br>• `detect` deterministic and within budget; <br>• a **cross-detection matrix per variant**: ≥ 0.7 on fixtures of the variants the plugin declares, ≤ 0.3 on fixtures of the variants it does not; <br>• layout validity; <br>• fixtures for every default step; <br>• robustness; <br>• no network. <br>A small synthetic detection-fixture pack for every built-in variant ships inside the package, so out-of-tree plugins can run the matrix. | M | P1 |
| FR-DEV-6 | Fixtures: <br>• each case is a folder with screenshots, `case.toml` and **`expected.txt`**, the conversation typed in the text-view format (required, no boxes needed); <br>• `expected.json` with element boxes is optional; <br>• `annotate bootstrap [--generic]` drafts `expected.json`; <br>• `annotate check` validates it against the box conventions. | M | P1 |
| FR-DEV-7 | Documentation for authors: <br>• a plugin authoring guide; <br>• `AGENTS.md` describing the agent loop (inspect → configure → overlay → eval → repeat); <br>• Bronze/Silver/Gold quality tiers; <br>• a **fully commented reference plugin** (WhatsApp Android). | M | P1 |
| FR-DEV-8 | A synthetic screenshot generator: one generic bubble template driven by palette and layout values, rendered in headless Chromium. It produces labelled screenshots and `expected.txt`, including mixed Hebrew and English, emoji runs, tall messages and blank lines. Templates imitating particular apps are Could, P2. | S | P1 |
| FR-DEV-9 | `fixture redact`: blur avatars, faces and phone numbers in fixtures. | C | P2 |

---

## 9. Non-functional requirements

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| NFR-PRIV-1 | No network access during `parse`, `detect`, `render`, `eval` or `emoji calibrate`. The test suite enforces this with sockets blocked. No telemetry. | M | P1 |
| NFR-PRIV-2 | Media crops, caches and emoji templates stay on the machine. Templates made from proprietary fonts or screenshots are never packaged or committed. | M | P1 |
| NFR-PERF-1 | At most 3 s per phone screenshot on a 4-core laptop CPU with the default OCR backend, not counting first-run model loading. A batch of 20 screenshots takes at most 60 s **including merging**. | S | P1 |
| NFR-PERF-2 | Peak memory at most 1.5 GB for a batch of 50 screenshots. | S | P1 |
| NFR-DET-1 | **Deterministic.** The same inputs (file contents and metadata), options, versions and `now` produce byte-identical JSON, apart from `batches[].processed_at`. "now" is recorded, and can be pinned with `--now`. | M | P1 |
| NFR-PORT-1 | Installs with `pip install chatsocr` on Windows, macOS and Linux (x86-64; arm64 where wheels exist), with no separate system install of the OCR engine. | M | P1 |
| NFR-ROB-1 | Never crash on bad input. A screenshot that fails is reported (`SCREEN_UNPARSEABLE`/`PLUGIN_ERROR`) and the rest of the batch is still processed. | M | P1 |
| NFR-EXT-1 | `chatsocr.plugin_api` is the only engine module plugins may import, and CI enforces this. It is typed (`py.typed`) and documented. | M | P1 |
| NFR-MAINT-1 | The engine passes type checking (strict for `plugin_api`). Engine test coverage is at least 80% of lines. | S | P1 |
| NFR-OBS-1 | Logging levels: `-q` (warnings), default (progress), `-v` and `-vv` (debug); optional `--log-file`. The library logs to the `chatsocr` logger and does not configure handlers. Debug mode writes intermediate results for each stage. | S | P1 |
| NFR-SEC-1 | Plugins are arbitrary Python code. Only built-in plugins, installed packages, `--plugin-dir` directories and the user's own plugin directory are loaded, and the documentation says so. `CHATSOCR_NO_PLUGINS=1` loads built-in plugins only. | M | P1 |
| NFR-LIC-1 | Every runtime dependency and bundled asset has a permissive license, and each is listed in `THIRD_PARTY.md`. | M | P1 |
| NFR-ACC-1 | Initial accuracy targets on the golden set, for each supported variant with default themes and original screenshots: <br>• message segmentation F1 ≥ 0.98 <br>• side ≥ 99% <br>• character error rate (CER) ≤ 3% on Hebrew-only and English-only text, ≤ 6% on mixed text <br>• time exact ≥ 98% <br>• date-separator resolution ≥ 98% <br>• reply target ≥ 90% (where the target is in the transcript) <br>• media kind ≥ 95% <br>• emoji top-1 ≥ 90% with calibrated templates <br>• **zero** lost repeated messages in the de-duplication cases <br>These targets will be revisited once the golden set exists. | S | P1 |
| NFR-ACC-2 | Confidences are calibrated so that the flags point at the errors: at least 80% of wrong fields on the golden set carry a flag. | S | P1 |
| NFR-ACC-3 | Custom themes, custom bubble colors, photo wallpapers and re-compressed screenshots are best-effort. They must **lower the reported confidence** (through palette-fit measures), not produce confident wrong output. | S | P1 |

---

## 10. Acceptance scenarios

Every scenario is an automated test on fixtures. The milestone that must pass each one is in [HLD §14](high-level-design.md#14-roadmap).

1. **1a. User's conversation in WhatsApp.** A WhatsApp Android screenshot of the user's example conversation produces exactly:

   ```
   0. System: [date 2026-11-24 Tue "Today"]
   1. 08:33 Bar: how are you?
   2. 08:35 Yuval: i am great, how are you?
   3. 08:40 Bar: <image media/3-1.png> I'm in the gym
   4. 16:33 Yuval: [reply: 3] you looks beautiful
   ```

   **1b. User's example shape.** A synthetic screenshot from the sample plugin with `separators="datetime"` (no per-message times) produces the six lines of [output-format §2.1](output-format.md#21-example), second block. Two separators on the same day raise no flag.
2. **Weekday resolution.** A screenshot anchored on Tuesday 2026-11-24 shows the separators "Sunday" and "יום ראשון", which both resolve to 2026-11-22. With the metadata removed and no `--captured-at`, the run's `now` is used, `ANCHOR_FALLBACK_NOW` is set, and the dates are shown with `?`.
3. **Mixed direction.**
   - **3a.** The phrases "I bought the חומוס you wanted" and "נפגשים מחר 18:30 at Dizengoff" come out in logical order, matching the ground truth. A single-line text whose direction is truly ambiguous either matches or is flagged `BIDI_AMBIGUOUS`.
   - **3b.** "נפגש ב-Zoom בשעה 18:30 מחר" is read correctly (the prefix repair, or the optional second backend).
4. **Mirrored UI.** WhatsApp with a Hebrew UI, run with `--left Yuval --right Bar`, attributes every message correctly and finds the time in the left-hand corner. Ticks on the left plus `--me left` raise no conflict.
5. **Overlap and repeats.** Three overlapping screenshots given in reverse order give one transcript with no duplicates. Two consecutive real "ok" messages are both kept.
6. **Gap.** Two screenshots that do not overlap give a `[gap]` item and `GAP_POSSIBLE`. Dates are not carried across the gap.
7. **Append.**
   - Batch A, then batch B: newer messages, older messages (from scrolling up), and messages that fill A's gap.
   - Existing IDs do not change. Newer messages continue the numbering. Older ones get `0.x`, and gap-filling ones get `P.x`. The filled gap is removed.
   - Appending B again changes nothing.
8. **Reply to an image.** A reply quoting an image (thumbnail plus "Photo") resolves to that image through the thumbnail signature.
9. **Unresolved reply.** A reply quoting a message outside the captured range is `unresolved` and keeps the quote text.
10. **Emoji.** "❤️😂😂😂" in a text line gives exactly those code points and no OCR junk, on incoming and outgoing bubble colors. An emoji with no template gives `�` and `EMOJI_UNIDENTIFIED`.
11. **Media.** A voice note gives `<voice 0:42>`, a document gives `<document "report.pdf" …>`, a link preview gives `<link "…">`, and an album of 3 images gives three `<image …>` tags.
12. **Detection, non-interactive.** An unrelated photo gives exit code 3 and a report. With `--app whatsapp`, the same photo gives `DETECTION_MISMATCH` and no exit code 3.
13. **Detection, interactive.** An ambiguous app shows a ranked choice, and the answer is stored with source `user`. An ambiguous platform within a clear app does not prompt.
14. **Out-of-tree plugins.** Two out-of-tree plugins are discovered, pass the conformance kit and parse their fixtures, without engine changes: a single-file plugin installed through an entry point, and a multi-file plugin package loaded with `--plugin-dir`.
15. **Offline.** The whole suite passes with sockets blocked.
16. **Determinism.** The same batch run twice with `--now` pinned gives identical JSON, apart from `processed_at`.
17. **Append across devices.** A transcript from WhatsApp Android with a Hebrew UI gets a batch from WhatsApp Web with an English UI appended, with sides given per batch. Overlapping messages are matched, not duplicated.
18. **Tall message.** A message taller than the screen, seen cut off across three screenshots, becomes one complete message with no `PARTIAL_MESSAGE`.
19. **Reply across batches.** A reply in batch B to an image from batch A, run with `--no-media`, resolves through the stored signatures.
20. **Pinned date pill.** Scrolled screenshots with the pinned date pill visible produce no extra separators. Messages dated from the pill get a date context line.

---

## 11. Phases and priorities

| Phase | Content |
|---|---|
| **P1: first release** | Everything marked P1: <br>• the engine, the provisional plugin API and developer tools; <br>• the WhatsApp, Telegram and Hinge variants of §7; <br>• text, emoji, media placeholders, replies, dates and times; <br>• merging and append; <br>• JSON and the text view; <br>• CLI, library and configuration. <br>Milestones are in [HLD §14](high-level-design.md#14-roadmap). |
| **P2** | Group chats; reactions; markers (edited, forwarded, `deleted`, read status, starred); HEIC; remembering emoji the user labelled; templates imitating particular apps; redaction tool. |
| **P3** | Small GUI; more apps (Instagram/Messenger, iMessage/SMS, Signal, …); screen-recording input; text formatting. |

## 12. Test data needed (a prerequisite)

No public, labelled dataset of chat screenshots exists ([research](research/prior-art.md)).

1. **A golden set of real screenshots** for each supported variant:
   - About 10 screenshots per variant, covering all phase-1 features in both themes, with Hebrew, English and mixed text, and both UI languages where the app has them.
   - They come from scripted conversations between consenting test accounts.
   - They are stored outside the public repository, either in a private fixtures repository or in Git LFS with restricted access.
2. **Typed ground truth** for each fixture: `expected.txt`, the conversation as it should come out, written by a person by reading the screenshots. This is the **minimum input** for the "agent writes a plugin" goal: screenshots plus a typed transcript. Box-level ground truth is optional and is drafted by the tools.
3. **Synthetic screenshots** from the generator (FR-DEV-8), for unit and regression tests.

## 13. Open questions for the user

Each question has a working default that the documents already use.

1. **Text-view format.** Do you approve the proposed form of your example ([output-format §2.1](output-format.md#21-example))?
   - Dates are written as `0. System: [date 2026-11-24T08:33 Tue "Today 8:33 AM"]` rather than `0. System: 2026-11-24T08:33`. The wrapper keeps dates apart from other system text, and the weekday and raw label are added only when the date was inferred.
   - Images carry their crop path: `<image media/3-1.png>`.
   - A per-message time slot is added for apps like WhatsApp: `1. 08:33 Bar: …`.
   - There are comment lines starting with `#` at the top, and uncertainty tags such as `[?text 0.58]`.
   - *Default:* as proposed.
2. **Which app did your example come from?** Its two time-stamped separators on one day, with no per-message times, look like Hinge.
   - *Default:* the design supports both styles (scenario 1a/1b).
3. **Inserted IDs.** Messages found later before or between existing ones get dotted IDs such as `4.1` or `0.3`. Repeated prepends make them deeper (`0.0.1`). Is that acceptable, together with a `renumber` command?
   - *Default:* yes.
4. **Media crops in phase 1.** You suggested they might wait. They are cheap once the image's box is known, so they are planned for P1 as Should.
   - *Default:* P1, can be turned off with `--no-media`.
5. **Emoji calibration.** WhatsApp, Apple (iPhone, and Telegram everywhere) and Samsung emoji designs cannot be shipped. Without calibration, emoji in those styles are identified only about 35% of the time. Calibration means pasting a generated emoji sheet into any app that uses the style (WhatsApp "Message yourself", Telegram "Saved Messages", a notes app), taking about 4–6 screenshots per style, and running `chatsocr emoji calibrate`. On a Mac, Apple's emoji font can be used directly. Acceptable?
   - *Default:* yes, about 500 most common emoji per style, extendable.
6. **Golden screenshots and typed transcripts.** Can you provide real screenshots, about 10 per app × platform × theme, from consenting accounts, each with a typed `expected.txt`? This matters most for **Hinge**, whose design is mostly unverified. Can Hinge test conversations be obtained at all? Where should they be stored?
   - *Default:* a separate private repository.
7. **Second OCR engine.** Tesseract misreads `ב-Zoom`. A default repair step is planned. A young community model (`Rivok/paddleocr-hebrew`, Apache-2.0, about 180 MB) reads these correctly but is slower. Include it as an optional extra in P1?
   - *Default:* optional (`chatsocr[hebrew-onnx]`).
8. **`deleted` messages.** Following your phase choice (no markers in P1), a deleted message comes out in P1 as its visible text, e.g. "This message was deleted", not as a `<deleted>` tag. OK?
   - *Default:* yes.
9. **Project license.**
   - *Default:* MIT or Apache-2.0, to be decided.
