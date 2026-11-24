# ChatsOCR: Requirements

| | |
|---|---|
| Status | Draft 1, for review |
| Date | 2026-11-24 |
| Related | [High-level design](high-level-design.md) · [Output format](output-format.md) · [Background research](research/README.md) |

In this document, **must**, **should** and **may** mean what they mean in RFC 2119. Each requirement has a priority (**M** = Must, **S** = Should, **C** = Could) and a phase (**P1** = first release, **P2** and **P3** = later). Requirement IDs never change once published.

---

## 1. Purpose

ChatsOCR turns **screenshots of a chat conversation** into a **structured transcript** that an AI agent can process easily. The user supplies:

- one or more screenshots of one conversation, and
- who the participants are on each side of the screen.

The program finds the messages and reads the text with local OCR. From that it builds an ordered transcript that includes:

- senders
- dates and times
- system events
- media placeholders
- emoji
- reply links

The output is a JSON file, which is the source of truth, plus a compact text view generated from it. The user's original example of the text view:

```
0. System: 2026-11-24T08:33
1. Bar: how are you?
2. Yuval: i am great, how are you?
3. Bar: <image> I'm in the gym
0. System: 2026-11-24T16:33
4. Yuval: [reply: 3] you looks beautiful
```

The exact agreed format is in [output-format.md](output-format.md). It keeps the shape of this example.

**The most important quality is extensibility.** Supporting a new chat app, platform or theme must mean writing a new plugin class and its tests, with **no change to the engine**. The plugin API, documentation and tools must make it possible for an AI coding agent to write such a plugin from example screenshots.

## 2. Original request and decisions so far

**Requirements stated by the user:**
- Show participants by name and number each message.
- Write date separators and other system messages as `0. System:` lines.
- Show the time of each message, where the app shows one, in a form that is distinguishable from the message text.
- Resolve a weekday name to the most recent such day. Count back from the time the screenshot was taken if that is known, otherwise from the current local time.
- Write images as `<image>`, ideally linked to an image file cropped from the screenshot.
- Link each reply to the message it quotes, using the visual quote block. The quote holds part of the original text and may hold a thumbnail.
- Emoji reactions can wait for a later phase.
- Output in a format that is easy for an AI agent to process.
- Let the user choose the chat app, and offer auto-detection. Ask the user only when detection fails.
- The program must never call an AI agent at runtime.
- Make it extensible: one class per chat app × platform × theme, as needed. Each class says whether it can handle a given screenshot.

**Decisions taken in the clarification round on 2026-11-24:**

| Topic | Decision |
|---|---|
| Languages | Hebrew and English, mixed even inside one message |
| Input | A batch of overlapping screenshots, taken while scrolling, becomes one transcript. Later batches can be **appended** to an existing transcript, and existing message IDs stay the same. |
| Privacy | Everything runs locally and offline. No cloud OCR. |
| Interface | A command-line tool (CLI) and a Python library now. A small GUI may come later. |
| First apps | WhatsApp, Telegram and Hinge |
| Platforms and themes | Android, iPhone, desktop/web; light and dark. Each app × platform × theme can have its own implementation with its own auto-detection. |
| Chat types | One-to-one chats first. Group chats later, but the design must leave room for them. |
| Phase 1 content | Text, images, replies, times and dates, **emoji inside text (which emoji it is, not just that one is there)**, and **media placeholders** |
| Later phases | Emoji reactions; markers for edited, forwarded, deleted and read-status |
| Output | JSON as the source of truth, plus a text view generated from it |
| Uncertainty | Mark it in the output. Ask the user only in interactive mode; never block when not interactive. |
| Plugin technology | Classic computer-vision (CV) helpers in the engine. A plugin may also ship its own small local model. |
| Documents | Markdown in `docs/` in this repository |

## 3. Glossary

| Term | Meaning |
|---|---|
| **Screenshot** | One image file: a phone screenshot, or a capture of a desktop or web app window |
| **Batch** | The set of screenshots processed in one run. They all show the same conversation. |
| **Transcript** | The output: an ordered list of *items*, plus metadata (`transcript.json`). The text view (`transcript.txt`) is generated from it. |
| **Item** | One entry in a transcript: a *message*, a *date separator*, a *system message* or a *gap marker* |
| **Message** | Something a participant sent (a text and/or media bubble). It has a stable **message ID**. |
| **System item** | A date separator, a system message (e.g. the encryption notice, a missed call, Hinge's "you matched"), or a gap marker. Shown as `0. System:` in the text view. |
| **Side** | Where a message appears on screen: `left` or `right`. Centered elements have side `center`. |
| **Participant** | A person in the chat. The user maps participants to sides. One participant may be marked as **self** (the phone's owner). |
| **Variant** | One look of one app: **app × platform × theme × UI epoch**, e.g. `whatsapp / android / dark / 2024`. The UI epoch separates app redesigns. |
| **Plugin** | A Python class that knows how to read one or more variants. It does detection, layout parsing and app-specific text grammar. |
| **Engine** | Everything that is not app-specific: loading, OCR, text direction, emoji, merging, time resolution, replies, IDs and output |
| **Layout / IR** | The intermediate representation (IR) a plugin returns for one screenshot: boxes for elements and their parts, each with a role |
| **Observation** | One sighting of an item in one screenshot. Because screenshots overlap, one item may have several observations. |
| **Anchor time** | When a screenshot was taken. Relative date labels such as "Yesterday" are resolved against it. |
| **Flag / warning** | A machine-readable code that marks an uncertain or degraded result |
| **Interactive mode** | The program may ask the user questions (on a terminal, or through an API callback). In non-interactive mode it never asks. |
| **em** | The height of one line of body text, measured on the screenshot. Thresholds are expressed in em so they work at any resolution. |
| **dp** | Density-independent pixel, the unit in which apps design their layouts |

## 4. Users and stakeholders

1. **End user.** Runs the CLI or the library on their own screenshots and owns the data, which is private.
2. **Downstream AI agent.** Reads `transcript.txt` or `transcript.json` to answer questions, summarize, and so on. It needs output that is unambiguous, easy to parse, compact in tokens, and honest about uncertainty.
3. **Plugin author.** A human, or an AI coding agent, adding support for a new app variant from example screenshots. They need a small, stable plugin API, numeric inspection tools, visual debug output and an automatic evaluation loop.
4. **Engine maintainer.** Evolves the engine without breaking plugins.

## 5. Scope

**In scope for phase 1:**
- 1:1 chats in WhatsApp, Telegram and Hinge on Android, iOS and desktop/web, as listed in §7, in light and dark themes, with English or Hebrew app interfaces.
- Text, emoji, images and other media placeholders, replies, per-message times, date separators and system messages.
- Merging several overlapping screenshots, and appending to existing transcripts.
- CLI, Python API and plugin developer tools.

**Out of scope for phase 1:**
- Group chats (phase 2).
- Reactions (phase 2).
- Edited, forwarded, deleted, read-status and starred markers (phase 2). A deleted message's placeholder text is still recorded as a system-like media placeholder; see FR-MED-1.
- Text formatting such as bold and italic.
- Photos of a screen taken with a camera.
- Video or screen-recording input.
- A GUI.
- Languages other than Hebrew and English.
- Custom wallpapers and custom bubble colors are handled on a best-effort basis only (NFR-ACC-3).

**Never in scope:**
- Calling an LLM or AI agent at runtime.
- Sending any data off the machine.

## 6. Assumptions and constraints

| ID | Constraint |
|---|---|
| CON-1 | Written in Python. Must run on CPython ≥ 3.12 on Windows, macOS and Linux, on CPU only. |
| CON-2 | **No network access while processing.** Model files (OCR language data, emoji templates) are installed with the package or by an explicit setup command. |
| CON-3 | **No LLM or AI agent at runtime.** Classic CV and an OCR engine are allowed. A plugin may ship an optional small local ML model (e.g. an ONNX detector). |
| CON-4 | The input is original screenshots (PNG, JPEG or WebP), not photos of screens. Screenshots that were re-compressed (e.g. forwarded through a messenger) are handled on a best-effort basis. |
| CON-5 | Runtime dependencies use permissive licenses (MIT, BSD or Apache-2.0). No GPL or AGPL code is copied or linked. Emoji images that may not be redistributed (Apple, WhatsApp, Samsung) are never shipped; see FR-EMO-3. |
| CON-6 | The user knows which participant is on which side, or accepts names inferred from the chat header (FR-IN-3). |

---

## 7. Supported variants in phase 1

| App | Android | iOS | Desktop / Web | Themes | App UI language |
|---|---|---|---|---|---|
| WhatsApp | ✔ | ✔ | ✔ (Web and desktop app) | light, dark (default theme) | English, Hebrew (mirrored layout) |
| Telegram | ✔ | ✔ | ✔ (Telegram Desktop) | light, dark (default themes) | English, Hebrew |
| Hinge | ✔ | ✔ | — (no desktop app) | light, dark | English (Hinge has no Hebrew UI) |

- Message text may be in Hebrew, English or both, whatever the language of the app's interface.
- A plugin may cover several variants. A dedicated class for a single variant is allowed and takes precedence (FR-PLG-2).

---

## 8. Functional requirements

### 8.1 Input (FR-IN)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-IN-1 | Accept one or more image files and/or directories as one batch of one conversation. Supported formats: PNG, JPEG, WebP. | M | P1 |
| FR-IN-2 | Accept a participant name for each side (`--left NAME --right NAME`) and, optionally, which side is the user's own (`--me left\|right`). | M | P1 |
| FR-IN-3 | If names are missing, infer defaults: the other participant from the chat header's title, and self as `Me`. Self's side comes from outgoing-message evidence such as delivery ticks. In interactive mode, confirm the defaults with the user. In non-interactive mode, use them and add the flag `PARTICIPANTS_INFERRED`. | M | P1 |
| FR-IN-4 | The screenshots in a batch may be given in any order. The program works out their order from their content (FR-MRG-1). File order and timestamps are hints only. | M | P1 |
| FR-IN-5 | Append mode: process a new batch into an existing transcript (`--append transcript.json`). The new messages may be newer than, older than, or between the existing ones. | M | P1 |
| FR-IN-6 | Optional hints: `--app`, `--platform`, `--theme`, `--ui-lang he\|en`. | M | P1 |
| FR-IN-7 | Optional explicit capture time, either for the whole batch or per file, overriding metadata (`--captured-at`). | S | P1 |
| FR-IN-8 | Never modify input files. | M | P1 |
| FR-IN-9 | Accept HEIC/HEIF input. | C | P2 |

### 8.2 App detection (FR-DET)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-DET-1 | If the app is not specified, detect the variant automatically. Each plugin reports, for each screenshot, a confidence between 0 and 1, the variant it matched, and the evidence. | M | P1 |
| FR-DET-2 | Combine the scores over all screenshots of the batch. App and platform are decided once per batch. Theme may differ between screenshots. | M | P1 |
| FR-DET-3 | Decision policy (thresholds configurable): **accept** when the top score is ≥ 0.6 and leads the runner-up by ≥ 0.15. **Ambiguous** otherwise, when the top score is ≥ 0.25. **None** when the top score is < 0.25. | M | P1 |
| FR-DET-4 | Ambiguous, interactive: ask the user to choose from a ranked list that shows the evidence. Ambiguous, non-interactive: take the top candidate and add the flag `DETECTION_AMBIGUOUS`, listing the alternatives. | M | P1 |
| FR-DET-5 | None, interactive: ask the user to choose an app. None, non-interactive: exit with the distinct exit code from FR-CLI-7 and a report of the top 3 candidates. | M | P1 |
| FR-DET-6 | A user-specified app or platform restricts the candidates. If the chosen plugin scores below the floor, continue but add the flag `DETECTION_MISMATCH`. | M | P1 |
| FR-DET-7 | Use filename hints as a prior (e.g. Samsung's `Screenshot_…_WhatsApp.jpg`). | S | P1 |
| FR-DET-8 | `chatsocr detect FILES --explain` prints every plugin's score and evidence. | M | P1 |

### 8.3 Layout recognition (FR-LAY)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-LAY-1 | Find the app's fixed screen areas (status bar, header, compose bar, navigation bar, side panel on desktop) and exclude them from the chat content. These areas are called *chrome*. | M | P1 |
| FR-LAY-2 | Find the chat elements in reading order: messages, date separators and system messages. | M | P1 |
| FR-LAY-3 | Find UI overlays and leave them out of the transcript: the floating date pill, the scroll-to-bottom button, the "N unread messages" banner, the typing indicator and the keyboard. The floating date pill may still be used as evidence of the date (FR-TIM-1). | M | P1 |
| FR-LAY-4 | Decide each message's side (`left` or `right`) and map it to the participant on that side. This must work when the app's UI is mirrored for a right-to-left language (WhatsApp in Hebrew puts outgoing messages on the left). | M | P1 |
| FR-LAY-5 | Record evidence that a message is outgoing (e.g. delivery ticks) when it is visible. Use it to infer self's side (FR-IN-3) and to check the user's `--me` value; flag `SELF_SIDE_CONFLICT` if they disagree. | S | P1 |
| FR-LAY-6 | A message cut off at the top or bottom edge of a screenshot is marked `cut`. A complete observation of the same message is preferred. If only cut observations exist, the message is output with the flag `PARTIAL_MESSAGE`. | M | P1 |
| FR-LAY-7 | Find the parts of each message: body text, time, reply quote (quoted sender, quoted text, thumbnail), media (kind, area, duration or file name text) and link preview. | M | P1 |
| FR-LAY-8 | Each bubble is one message, including consecutive bubbles from the same sender. | M | P1 |
| FR-LAY-9 | An album (a grid of several images) is one message with several media entries. | S | P1 |
| FR-LAY-10 | Group chats: the sender name and avatar shown on each message. | M | P2 |
| FR-LAY-11 | Reactions, and edited, forwarded, read-status and starred markers. | M | P2 |

### 8.4 Text recognition (FR-OCR)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-OCR-1 | Recognize Hebrew, English and mixed Hebrew/English text, including digits, punctuation, URLs, phone numbers and currency signs (₪ $ €). | M | P1 |
| FR-OCR-2 | Output text in **logical Unicode order**, NFC-normalized, with no bidi control characters. Each message records its paragraph direction (`ltr` or `rtl`). When the direction cannot be decided from the pixels, add the flag `BIDI_AMBIGUOUS`. | M | P1 |
| FR-OCR-3 | Join visually wrapped lines with a space. Keep line breaks the sender typed when they can be detected: the line ends well before the bubble's maximum text width, and the next word would have fit. Add the flag `LINEBREAK_UNCERTAIN` when this cannot be decided. | S | P1 |
| FR-OCR-4 | Keep the confidence of each word. Give each message a combined text confidence. | M | P1 |
| FR-OCR-5 | The OCR engine is a replaceable backend behind an interface. The default is a local engine that supports Hebrew and English. | M | P1 |
| FR-OCR-6 | A secondary backend may re-read lines with low confidence or known-weak patterns. One such pattern is a Hebrew prefix letter, hyphen and Latin word, e.g. `ב-Zoom`. | S | P1 |
| FR-OCR-7 | Read times and date labels with role-specific settings: a character whitelist, single-line mode, and validation against a pattern. | M | P1 |
| FR-OCR-8 | Recognize text formatting such as bold, italic, monospace and strikethrough. | C | P3 |

### 8.5 Emoji (FR-EMO)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-EMO-1 | Detect emoji glyphs inside text lines, and emoji-only ("jumbo") messages. They must never reach the output as OCR junk. | M | P1 |
| FR-EMO-2 | Identify which emoji each glyph is, as a Unicode sequence. Matching is against templates in the emoji design (*emoji style*) that the variant uses, e.g. WhatsApp's own set, Apple, Google Noto or Samsung. Each identification has a confidence. | M | P1 |
| FR-EMO-3 | Get emoji templates without redistributing proprietary artwork: <br>(a) bundle the Noto Emoji set (OFL/Apache); <br>(b) build templates at runtime from emoji fonts installed on the user's machine, e.g. Apple Color Emoji on macOS; <br>(c) a **calibration** command that builds templates from the user's own screenshots of a generated emoji sheet sent in the target app (§8.13). <br>Templates stay in the user's local cache. | M | P1 |
| FR-EMO-4 | An emoji that cannot be identified confidently appears in the text as U+FFFD (`�`) with the flag `EMOJI_UNIDENTIFIED`. The JSON keeps the best guesses, the confidence and the box. | M | P1 |
| FR-EMO-5 | Skin-tone modifiers and ZWJ sequences: best effort. When unsure, fall back to the base emoji and add a flag. | S | P1 |
| FR-EMO-6 | Remember emoji the user labels in interactive mode, so they are identified automatically later. | C | P2 |

### 8.6 Media (FR-MED)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-MED-1 | Recognize media messages and output typed placeholders: `image`, `video` (+ duration), `gif`, `sticker`, `voice` (+ duration), `audio`, `document` (+ file name, size), `link` (+ URL and title), `location`, `contact`, `poll`, `call` (missed, voice or video), `deleted` (the "This message was deleted" placeholder), `view-once`, `unknown`. Phase 1 must support image, video, gif, sticker, voice, document, link, location and deleted; the others are Should. | M | P1 |
| FR-MED-2 | A plugin may define its own app-specific placeholder kinds, named with an `x-` prefix (e.g. `x-hinge-prompt`). The engine and output formats carry them without changes. | M | P1 |
| FR-MED-3 | A caption on a media message is the message's text. | M | P1 |
| FR-MED-4 | Crop each visual media item (image, video thumbnail, sticker, GIF frame) from the screenshot, save it in a `media/` folder next to the transcript, and link it from both the JSON and the text view. `--no-media` turns this off. | S | P1 |
| FR-MED-5 | If a media item appears in several screenshots, crop it from the most complete observation. | S | P1 |

### 8.7 Replies (FR-REP)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-REP-1 | Detect reply quote blocks. Extract the quoted sender, the quoted text (and whether it is truncated), the quoted media kind label, and the thumbnail. | M | P1 |
| FR-REP-2 | Find the replied-to message among the earlier messages of the transcript, including those from earlier appended batches. Matching combines: <br>• fuzzy, prefix-anchored, truncation-aware matching of the quoted text; <br>• the quoted sender; <br>• compatible media kind; <br>• thumbnail similarity; <br>• a small preference for more recent messages. | M | P1 |
| FR-REP-3 | Each reply ends in one of three states: <br>**resolved:** a target ID with a confidence; <br>**ambiguous:** a ranked list of candidates and the flag `REPLY_AMBIGUOUS`; <br>**unresolved:** the target is probably outside the captured range, flag `REPLY_UNRESOLVED`. <br>The quote's content is kept in every state. | M | P1 |
| FR-REP-4 | Keep the quoted snippet separate from the message's own text. Never merge it into the body. | M | P1 |
| FR-REP-5 | In append mode, retry unresolved and ambiguous replies against the newly added messages. Resolved replies are never changed. | S | P1 |

### 8.8 Dates and times (FR-TIM)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-TIM-1 | Each date separator becomes a system item with a resolved absolute date. The raw label is kept as well. A floating date pill only supplies date evidence for the messages above the first separator on that screenshot. | M | P1 |
| FR-TIM-2 | When the app shows a time on each message, attach that time to the message. The text view shows it in a fixed position, distinguishable from the text. | M | P1 |
| FR-TIM-3 | Resolve relative labels against the anchor time of the **screenshot in which the label appears**: <br>• "Today"/"היום" is the anchor date; <br>• "Yesterday"/"אתמול" is the day before; <br>• a weekday name (English or Hebrew, full or abbreviated) is the most recent such day *strictly before* the anchor date. | M | P1 |
| FR-TIM-4 | Choose each screenshot's anchor time from the first source available, in this order: <br>1. user override (FR-IN-7) <br>2. EXIF `DateTimeOriginal` (with offset) <br>3. timestamp in the file name <br>4. XMP creation date <br>5. file modification time (low confidence) <br>6. current local time (flag `ANCHOR_FALLBACK_NOW`) <br>The status-bar clock, when readable, is used to cross-check (flag `ANCHOR_CONFLICT`). Record the source and confidence of each anchor. | M | P1 |
| FR-TIM-5 | Give a date shown without a year the latest year that keeps it on or before the anchor. Then make sure separators never go backwards in date (flag `YEAR_INFERRED`). | M | P1 |
| FR-TIM-6 | Parse 12-hour and 24-hour clocks, including AM/PM in English and Hebrew forms (`לפנה״צ`, `אחה״צ`) and narrow no-break spaces. Output times in 24-hour form. | M | P1 |
| FR-TIM-7 | Resolve whether a numeric date is day-first or month-first. Use what the plugin knows about the locale first; then heuristics (a value > 12, dates that keep increasing, which part changes more often). Store the decision at transcript level and reuse it when appending. | S | P1 |
| FR-TIM-8 | A message without its own visible time gets the date of the preceding separator and no time of day. Times are never invented; date and time each record their source. | M | P1 |
| FR-TIM-9 | Check that (date, time) never goes backwards. On a violation, try alternative OCR readings of easily confused digits. Any correction keeps the raw value and adds the flag `TIME_CORRECTED`; otherwise add `TIME_NON_MONOTONIC`. Never correct silently. | S | P1 |
| FR-TIM-10 | All times are naive local wall-clock times (no time zone), in ISO 8601 format. | M | P1 |
| FR-TIM-11 | A separator that also shows a time (e.g. Hinge's "Today 8:33 PM") becomes a system item with a time of day. The messages that follow get that date, and no time of their own unless the app shows one. | M | P1 |

### 8.9 Several screenshots and append (FR-MRG)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-MRG-1 | Work out the vertical offsets between overlapping screenshots by **aligning the images** of the chat area, with chrome and overlays masked out. Every offset must be confirmed by at least one message matched in the overlap. When the evidence is weak, flag it instead of guessing (`OVERLAP_LOW_CONFIDENCE`). | M | P1 |
| FR-MRG-2 | Remove duplicate observations of the same message using position and content together. **Identical messages that really repeat (e.g. "ok", "ok") must stay separate.** | M | P1 |
| FR-MRG-3 | If a screenshot has no reliable overlap with the others, insert a **gap** system item, order the pieces by dates and times, and flag the order if unsure (`GAP_POSSIBLE`, `ORDER_UNCERTAIN`). | M | P1 |
| FR-MRG-4 | When a message is seen several times, take each field from the best observation (complete rather than cut, higher OCR confidence) and keep every observation in its provenance. | M | P1 |
| FR-MRG-5 | **Stable IDs in append mode:** existing message IDs never change. New messages after the end get the next integer. Messages that belong before or between existing messages get **dotted IDs** that sort into place, e.g. `4.1` and `4.2` between 4 and 5. Full rules are in [output-format.md §4](output-format.md#4-message-ids). | M | P1 |
| FR-MRG-6 | Appending is idempotent: appending a batch that is already in the transcript adds no items. | M | P1 |
| FR-MRG-7 | Before appending, check that the transcript's app and participants match the new batch. On a mismatch: in interactive mode, ask the user; in non-interactive mode, refuse with exit code 5, unless `--force` is given. | S | P1 |
| FR-MRG-8 | A `renumber` command compacts IDs to 1…N. It writes an ID map and increments the transcript's `id_epoch` field. | C | P2 |

### 8.10 Output (FR-OUT)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-OUT-1 | Write `transcript.json`, which follows a versioned JSON Schema shipped with the package. It is the source of truth. | M | P1 |
| FR-OUT-2 | Write `transcript.txt`, generated deterministically from the JSON using the grammar in [output-format.md](output-format.md). It is compact, easy to parse with a regular expression, and in the shape of the user's example. | M | P1 |
| FR-OUT-3 | Each message has: <br>• ID, sender and side; <br>• date and time (each with its source); <br>• text and paragraph direction; <br>• spans for emoji and low-confidence parts; <br>• media and reply; <br>• provenance: which screenshot(s), and the box in each; <br>• confidence and flags. | M | P1 |
| FR-OUT-4 | Store enough data to append later without the original images: fingerprints, image hashes, source file hashes and the time anchors. | M | P1 |
| FR-OUT-5 | Write output to `-o DIR`: `transcript.json`, `transcript.txt` and `media/`. In append mode, update the files in place, after writing a backup copy. | M | P1 |
| FR-OUT-6 | `chatsocr render transcript.json` regenerates the text view without processing images again. | S | P1 |
| FR-OUT-7 | All text output is UTF-8 without BOM and contains no bidi control characters. An optional display mode (`--isolate`) may add them. | M | P1 |
| FR-OUT-8 | Flags use a closed, documented list of codes ([output-format.md §6](output-format.md#6-flag-codes)), so agents can rely on them. | M | P1 |
| FR-OUT-9 | Debug output (`--debug DIR`): for each screenshot, an overlay image that shows the detected elements, parts and IDs. | S | P1 |

### 8.11 Uncertainty and interaction (FR-UNC)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-UNC-1 | Every uncertain decision adds a flag (code, field, confidence) to the affected item, or to the transcript as a whole. This covers detection, side, sender, text, emoji, date, time, reply, order and duplicate removal. The text view shows compact markers for flags whose confidence is below a configurable threshold (default 0.80). | M | P1 |
| FR-UNC-2 | The CLI is interactive when stdin and stdout are terminals, unless `--no-input` is given. `--interactive` forces interactive mode. The library never prompts by itself; it calls an optional resolver callback (FR-API-3). | M | P1 |
| FR-UNC-3 | Non-interactive runs never block or wait. They always finish with output (plus flags) or with a documented exit code. | M | P1 |
| FR-UNC-4 | Interactive questions cover at least: choosing the app when detection is ambiguous or fails, confirming inferred participant names, and an append mismatch. An optional review step (`--review`) asks about low-confidence replies, emoji and times. | M (review: S) | P1 |
| FR-UNC-5 | Record the user's answers in the JSON with source `user`, so that runs can be reproduced. | S | P1 |

### 8.12 CLI and library (FR-CLI, FR-API)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-CLI-1 | `chatsocr parse FILES… [--left N --right N --me S] [--app A --platform P --theme T] [-o DIR] [--append JSON] [--captured-at T] [--no-input\|--interactive] [--review] [--no-media] [--debug DIR]` | M | P1 |
| FR-CLI-2 | `chatsocr detect FILES… [--explain]` | M | P1 |
| FR-CLI-3 | `chatsocr render JSON [--isolate] [--threshold X]` | S | P1 |
| FR-CLI-4 | `chatsocr plugins list` shows id, variants, source (built-in, installed or local), version, and whether it is available (with the reason if not). | M | P1 |
| FR-CLI-5 | `chatsocr emoji calibrate …` and `chatsocr emoji status` (§8.13) | M | P1 |
| FR-CLI-6 | `chatsocr setup` installs optional model files (e.g. higher-accuracy OCR data). This is the only command that may use the network, and only when the user runs it explicitly. | S | P1 |
| FR-CLI-7 | Exit codes: <br>0 = success, possibly with flags <br>1 = internal error <br>2 = usage error <br>3 = no plugin matched <br>4 = no screenshot could be parsed <br>5 = append refused because the chat does not match <br>A one-line summary of the flags goes to stderr. | M | P1 |
| FR-API-1 | `chatsocr.parse(images, participants=…, app=None, …) -> Transcript`. Also `chatsocr.append(transcript, images, …) -> Transcript`, `Transcript.to_json()`, `Transcript.to_text()` and `Transcript.load(path)`. | M | P1 |
| FR-API-2 | `images` may be file paths, bytes, PIL images or NumPy arrays. Metadata such as capture time can be passed in explicitly. | S | P1 |
| FR-API-3 | An optional `resolver` callback object answers interactive questions in library use. Without one, non-interactive policy applies. | S | P1 |
| FR-GUI-1 | A small GUI: drag in screenshots, review the result, fix flagged items. | C | P3 |

### 8.13 Plugin system (FR-PLG): the extensibility requirements

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-PLG-1 | **Supporting a new app, platform, theme or UI epoch requires only a new plugin module (plus fixtures and tests). No engine code changes.** This is checked by a CI test that loads a plugin from outside the source tree. | M | P1 |
| FR-PLG-2 | A plugin class declares the variants it supports. One class may cover several variants. When two classes claim the same variant, the more specific one wins, then the higher declared priority. | M | P1 |
| FR-PLG-3 | Each plugin provides a cheap `detect` method that returns a confidence, the matched variant and the evidence (FR-DET-1). | M | P1 |
| FR-PLG-4 | **Plugins handle layout and app-specific grammar only.** <br>• *Layout:* chrome, overlays, elements and their parts with roles. <br>• *Grammar:* how to read the app's time, date-label and system-message strings, and the media kind. <br>The engine handles everything that is the same for every app: OCR, text direction, emoji, merging, IDs, time resolution, reply resolution, cropping and output. | M | P1 |
| FR-PLG-5 | The plugin API offers three ways to write a plugin: <br>(a) a whole-screen method `parse_screen`; <br>(b) a **step-by-step** base class with small steps that can be overridden one at a time; <br>(c) a **sequential** base class in which the author writes only `crop_next_element(screen, y_after_previous)`. Option (c) is the per-message model the user proposed. | M (c: S) | P1 |
| FR-PLG-6 | The engine provides a documented toolkit of CV helpers for plugins, so that typical plugin code is short. It includes: color clustering and masks, finding flat rounded regions (bubbles), finding centered chips, finding small glyph clusters (times, ticks), finding a quote bar, icon template matching, row and column profiles, and dp/em unit conversion. | M | P1 |
| FR-PLG-7 | Plugin constants are written in dp or em, not raw pixels. The engine supplies the scale for each screenshot. | M | P1 |
| FR-PLG-8 | Theme colors are data (a palette for each variant), optionally loaded from a TOML file next to the plugin and regenerated from labelled fixtures by a tool. | S | P1 |
| FR-PLG-9 | Plugins are discovered from three places: built into the package, installed Python packages (an entry-point group), and local directories (`--plugin-dir`, and a user config directory). On an ID collision, local beats installed beats built-in, with a warning. | M | P1 |
| FR-PLG-10 | Each plugin declares `api_version`. A plugin built for another major version is rejected with a clear message. Breaking changes require a major version bump; removals go through deprecation warnings first. | M | P1 |
| FR-PLG-11 | A plugin may declare optional dependencies and model files (e.g. an ONNX detector). If they are missing, the plugin is listed as *unavailable (reason)*; nothing crashes. | S | P1 |
| FR-PLG-12 | Importing a plugin is cheap (no heavy work at import time). `detect` stays within a time budget (default 50 ms for each screenshot and plugin). The conformance kit enforces both. | S | P1 |
| FR-PLG-13 | App-specific elements that have no generic meaning (e.g. Hinge's "liked your prompt" card) are turned by the plugin into generic items: a system item with text, or a message with an `x-` media kind. The engine needs no change for them. | M | P1 |
| FR-PLG-14 | Each variant declares its emoji style, its UI direction behaviour, how it lays out the paragraph direction of text, and its date and time label vocabulary. The engine has defaults for English and Hebrew. | M | P1 |

### 8.14 Tools for plugin developers and AI agents (FR-DEV)

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| FR-DEV-1 | `chatsocr plugin new --app X --platform Y --themes light,dark` creates the plugin module, palette file, fixture folders, a test file and an authoring checklist. | M | P1 |
| FR-DEV-2 | Numeric inspection tools, so that agents never have to guess pixel values by looking at an image: <br>• `inspect probe` (color, Lab, ΔE to the palette at a point or box); <br>• `inspect palette` (color clusters with area and sample coordinates); <br>• `inspect rows` (non-background row ranges and profiles); <br>• `inspect calibrate` (palette from annotated fixtures). | M | P1 |
| FR-DEV-3 | `chatsocr debug overlay IMG [--expected]` draws the detected elements, parts and IDs on the screenshot, optionally side by side with the ground truth. | M | P1 |
| FR-DEV-4 | `chatsocr eval --plugin X [--json]` scores the plugin against its golden fixtures (§12): missing, extra or mismatched elements; text errors; time, reply and media errors. It exits non-zero below the thresholds. | M | P1 |
| FR-DEV-5 | A conformance test kit, shipped as a pytest plugin, that every plugin must pass: <br>• valid metadata; <br>• `detect` is deterministic and within budget; <br>• the **cross-detection matrix** passes: ≥ 0.7 on the plugin's own fixtures and ≤ 0.3 on every other plugin's; <br>• the IR is valid; <br>• robustness: blank, foreign or tiny images give an empty result, never a crash; <br>• no network access. | M | P1 |
| FR-DEV-6 | Fixture format: each case is a folder with the screenshot(s), `expected.json` and `case.toml` (capture time, participants, locale, tolerances). `annotate bootstrap` writes a draft `expected.json` from the current plugin for a person to correct. | M | P1 |
| FR-DEV-7 | Documentation for authors: a plugin authoring guide, an `AGENTS.md` describing the agent workflow (probe → configure → overlay → eval → repeat), a tiered quality checklist (Bronze, Silver, Gold), and one **fully commented reference plugin** (WhatsApp Android). | M | P1 |
| FR-DEV-8 | A synthetic screenshot generator (HTML/CSS templates rendered in headless Chromium) that produces labelled screenshots, including mixed Hebrew/English, for unit and regression tests. | S | P1 |
| FR-DEV-9 | `fixture redact`: blur avatars, faces and phone numbers in fixture screenshots. | C | P2 |

---

## 9. Non-functional requirements

| ID | Requirement | Pri | Ph |
|---|---|---|---|
| NFR-PRIV-1 | No network access during `parse`, `detect`, `render`, `eval` or `emoji calibrate`. The test suite enforces this by blocking sockets. No telemetry. | M | P1 |
| NFR-PRIV-2 | Media crops, caches and emoji templates stay on the local machine. Templates built from proprietary fonts or screenshots are never packaged or committed. | M | P1 |
| NFR-PERF-1 | At most 3 s per phone screenshot on a 4-core laptop CPU with the default OCR backend, not counting model loading on first run. A batch of 20 screenshots takes at most 60 s. | S | P1 |
| NFR-PERF-2 | Peak memory at most 1.5 GB for a batch of 50 screenshots. | S | P1 |
| NFR-DET-1 | **Deterministic:** the same inputs, options and versions produce byte-identical JSON. The only exception is the processing timestamps stored under `batches[]`. | M | P1 |
| NFR-PORT-1 | Installs with `pip install chatsocr` on Windows, macOS and Linux (x86-64; arm64 where wheels exist), with no separate system installation of the OCR engine. | M | P1 |
| NFR-ROB-1 | Never crash on bad input. A screenshot that fails is reported (`SCREEN_UNPARSEABLE`) and the rest of the batch is still processed. An exception inside a plugin is caught and reported (`PLUGIN_ERROR`). | M | P1 |
| NFR-EXT-1 | The plugin API (`chatsocr.plugin_api`) is the only module plugins may import from the engine. It is typed (`py.typed`) and documented, and follows semantic versioning. | M | P1 |
| NFR-MAINT-1 | Engine code passes type checking (mypy or pyright, strict for `plugin_api`). Test coverage of the engine is at least 80% of lines. | S | P1 |
| NFR-OBS-1 | Logging has verbosity levels. In `--debug` mode the program writes the intermediate results of each pipeline stage. | S | P1 |
| NFR-SEC-1 | Plugins are arbitrary Python code. Non-built-in plugins are loaded only from installed packages and from explicitly configured directories, and the documentation says so. `CHATSOCR_NO_PLUGINS=1` loads built-in plugins only. | M | P1 |
| NFR-LIC-1 | Every runtime dependency and bundled asset has a permissive license, and its license is listed in a `THIRD_PARTY.md` file. | M | P1 |
| NFR-ACC-1 | Accuracy targets on the golden set, for each supported variant with default themes and original (not re-compressed) screenshots: <br>• message segmentation F1 ≥ 0.98 <br>• side accuracy ≥ 99% <br>• character error rate (CER) ≤ 3% on Hebrew-only and English-only text, ≤ 6% on mixed text <br>• per-message time exact match ≥ 98% <br>• date separator resolution ≥ 98% <br>• reply target accuracy ≥ 90%, where the target is in the transcript <br>• media kind accuracy ≥ 95% <br>• emoji top-1 identification ≥ 90% with calibrated templates <br>• **zero** lost repeated messages in the duplicate-removal test cases <br>These are initial targets, to be revisited once the golden set exists. | S | P1 |
| NFR-ACC-2 | Confidences are calibrated so that flagged items are where the errors are. Target: at least 80% of the wrong fields on the golden set carry a flag. | S | P1 |
| NFR-ACC-3 | Custom themes, custom bubble colors, photo wallpapers and re-compressed screenshots are handled on a best-effort basis. These conditions must lower the reported confidence; they must not produce confident wrong output. | S | P1 |

---

## 10. Acceptance scenarios

All of these are automated tests in the repository, run on fixtures.

1. **User's example.** A WhatsApp screenshot containing the user's example conversation produces the example's items: two separators, four messages, `<image>` with caption, and `[reply: 3]`. They are in the format of [output-format.md](output-format.md).
2. **Weekday resolution.** A screenshot whose anchor is Tuesday 2026-11-24 has a separator "Sunday" and another "יום ראשון". Both resolve to 2026-11-22. With metadata removed and no `--captured-at`, the anchor falls back to the current local time and the flag `ANCHOR_FALLBACK_NOW` is set.
3. **Mixed direction.** "נפגש ב-Zoom בשעה 18:30 מחר" and "I bought the חומוס you wanted" come out in logical order, matching the ground truth.
4. **Mirrored UI.** WhatsApp with a Hebrew UI, run with `--left Yuval --right Bar`, attributes every message correctly. Ticks on the left side, together with `--me left`, raise no conflict.
5. **Overlap and repeats.** Three overlapping screenshots, given in reverse order, produce one transcript with no duplicates. Two consecutive real "ok" messages are both kept.
6. **Gap.** Two screenshots that do not overlap produce a `[gap]` system item between them, with the flag `GAP_POSSIBLE`.
7. **Append.** A transcript built from batch A gets batch B appended, where B has newer messages and also older ones scrolled up to. Existing IDs are unchanged, newer messages continue the numbering, older ones get dotted IDs, and appending B again changes nothing.
8. **Reply to an image.** A reply that quotes an image (thumbnail plus label "Photo") resolves to the image message through the thumbnail.
9. **Unresolved reply.** A reply quoting a message outside the captured range is `unresolved` and keeps the quote's text.
10. **Emoji.** A text line containing ❤️😂 yields those exact code points and no OCR junk. An emoji with no matching template yields `�` and `EMOJI_UNIDENTIFIED`.
11. **Media.** A voice note produces `<voice 0:42>`; a document produces `<document "report.pdf" …>`; a link preview produces `<link "…">`.
12. **Detection, non-interactive.** An unrelated image (e.g. a photo) exits with code 3 and a report.
13. **Detection, interactive.** An ambiguous detection shows a ranked choice. The user's answer is stored with source `user`.
14. **Out-of-tree plugin.** A sample plugin package under `tests/` registers through an entry point, is discovered, passes the conformance kit and parses its fixture, all without changes to engine code.
15. **Offline.** The whole test suite passes with network sockets blocked.
16. **Determinism.** Running the same batch twice produces identical JSON, apart from `batches[].processed_at`.

---

## 11. Phases and priorities

| Phase | Content |
|---|---|
| **P1: first release** | Everything marked P1 above: the engine, plugin API v1 and developer tools; the WhatsApp, Telegram and Hinge variants of §7; text, emoji, media placeholders, replies, dates and times, merging and append; JSON and the text view; CLI and library. The internal milestone order is in [high-level-design.md §14](high-level-design.md#14-roadmap). |
| **P2** | Group chats (sender names, avatars, per-sender colors in quotes); reactions; edited, forwarded, deleted, read-status and starred markers; the `renumber` command; HEIC input; remembering emoji the user labelled; redaction tool. |
| **P3** | A small GUI; more apps (Instagram/Messenger, iMessage/SMS, Signal, …); screen-recording input; text formatting. |

## 12. Test data needed (a prerequisite)

No public dataset of labelled chat screenshots exists ([research](research/prior-art.md)). The project needs:

1. **A golden set of real screenshots** for each supported variant: about 10 screenshots each, covering every phase-1 feature in both themes, with Hebrew, English and mixed text and in both UI languages where applicable.
   - The screenshots must come from scripted conversations between consenting test accounts.
   - They must be stored outside the public repository, either in a private fixtures repository or in Git LFS with restricted access.
   - Ground truth is drafted with `annotate bootstrap` and corrected by hand.
2. **Synthetic screenshots** from the generator (FR-DEV-8), with exact ground truth, for unit and regression tests.

## 13. Open questions for the user

Each question has a working default, which the design currently uses.

1. **Inserted IDs.** Messages found later between or before existing messages get dotted IDs such as `4.1` or `0.3`. Is that acceptable, or should every append renumber everything (with an ID map)?
   - *Default:* dotted IDs, plus a `renumber` command in P2.
2. **Media crops.** You suggested these might wait for phase 2. They are cheap once the image's box is known, so they are planned for P1 as Should.
   - *Default:* P1, turned off with `--no-media`.
3. **Emoji calibration.** The emoji designs used by WhatsApp, Apple (iPhone, and Telegram everywhere) and Samsung cannot be shipped. Without calibration, emoji in those apps are identified only about 35% of the time, judging from cross-design matching tests. Is a one-time calibration acceptable: send a generated emoji sheet to yourself in each app, screenshot it, and run `chatsocr emoji calibrate`? On a Mac, Apple's emoji font can be used directly instead.
   - *Default:* yes, a calibration of the ~500 most common emoji, extendable.
4. **Golden screenshots.** Can you provide real screenshots, from consenting accounts, for each app × platform × theme? This matters most for **Hinge**, whose screen design is almost entirely unverified. Where should they be stored?
   - *Default:* a separate private repository.
5. **Second OCR engine.** Tesseract misreads a Hebrew prefix joined to an English word (`ב-Zoom` becomes `Zoom-1`). A young community model (`Rivok/paddleocr-hebrew`, Apache-2.0, ~180 MB) reads these correctly but is slower and weaker on English-heavy lines. Should it be an optional extra in P1?
   - *Default:* yes, optional (`chatsocr[hebrew-onnx]`), used only to re-read lines that are suspicious or have low confidence.
6. **Project license.**
   - *Default:* MIT or Apache-2.0, to be decided.
7. **Minimum Python version.**
   - *Default:* 3.12 or newer.
