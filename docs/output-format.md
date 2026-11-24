# ChatsOCR: Output Format Specification

| | |
|---|---|
| Status | Draft 1, for review |
| Date | 2026-11-24 |
| Versions specified | JSON `schema_version` 1.0.0 · text view `chatsocr-text/1` |
| Related | [Requirements](requirements.md) (FR-OUT, FR-MRG-5, FR-UNC) · [High-level design](high-level-design.md) |

This document is the contract between ChatsOCR and anything that reads its output: AI agents, scripts and people. Plugins never produce this format directly. The engine renders it from the merged transcript, so every app produces the same format.

## 1. Files

One run writes the following into the output directory (`-o DIR`, default `./chatsocr-out/`):

| File | Purpose |
|---|---|
| `transcript.json` | **Source of truth.** Lossless: positions, confidences, provenance, flags, and fingerprints for appending later. |
| `transcript.txt` | **Text view.** Compact, line-based and cheap in tokens. Generated from the JSON, never edited by hand. |
| `media/` | Images cropped from the screenshots (FR-MED-4), named `<message-id>-<n>.png` (e.g. `5-1.png`). Paths in both files are relative to the transcript. |
| `transcript.json.bak` | In append mode, the previous version of the transcript, written before it is replaced. |

All text is UTF-8 without BOM. Text is always in **logical Unicode order**, NFC-normalized, with no bidi control characters (U+200E, U+200F, U+061C, U+202A–U+202E and U+2066–U+2069 are stripped). U+200D (zero-width joiner) and U+FE0F (variation selector-16) are kept, because emoji sequences need them.

---

## 2. Text view (`transcript.txt`)

### 2.1 Example

```
# chatsocr-text/1
# app: whatsapp/android/dark (auto 0.97)
# participants: Bar=left, Yuval=right(self)
# times: local wall clock, no time zone
0. System: [date 2026-11-22 Sun "Sunday"]
1. 21:04 Bar: מה קורה? how was the trip
2. 21:10 Yuval: <voice 0:42>
0. System: You turned off disappearing messages.
0. System: [date 2026-11-24 Tue "Today"]
3. 08:33 Bar: how are you?
4. 08:35 Yuval: i am great, how are you?
5. 08:40 Bar: <image media/5-1.png> I'm in the gym 💪
6. 16:33 Yuval: [reply: 5] you looks beautiful
7. 16:36? Bar: [?text 0.58] תודה!! נתראה ב-8
8. 16:40 Yuval: [reply: ? Bar "see you at 8 at the"] <link "https://maps.app.goo.gl/x"> here
9. 16:41 Bar: \[joke] two lines
  and a second line
```

For an app that shows no per-message times but has separators with times (e.g. Hinge), the same chat looks like this. It matches the user's original example exactly:

```
0. System: [date 2026-11-24T08:33 Tue "Today 8:33 AM"]
1. Bar: how are you?
2. Yuval: i am great, how are you?
3. Bar: <image media/3-1.png> I'm in the gym
0. System: [date 2026-11-24T16:33 Tue "Today 4:33 PM"]
4. Yuval: [reply: 3] you looks beautiful
```

### 2.2 Line rules

1. **Header.** Lines starting with `#` are comments with metadata. The first line is always `# chatsocr-text/<major>`. A consumer can ignore the rest of the header.
2. **One item per line.**
   - A **message** line starts with its ID: `5.`, or `4.1.` for a dotted ID (§4).
   - A **system item** line always starts with `0. System:`. System items cannot be referred to from the text view; their IDs are in the JSON.
3. **Time slot.** If the app shows a time on each message, it comes right after the ID as 24-hour `HH:MM`. A trailing `?` means the time is uncertain. If the message has no time of its own, the slot is left out. The message's date is the date of the nearest `[date …]` line above it.
4. **Sender.** The sender is the participant's display label, followed by `: `. Labels are cleaned when the transcript is created: they cannot contain `:`, `"`, `[`, `]`, `<`, `>` or a newline, and cannot start with a digit. The original name is kept in the JSON.
5. **Tags.** After the sender come zero or more **tags**, in a fixed order, separated by single spaces:
   1. flag tags `[?…]`
   2. a reply tag `[reply: …]`
   3. media tags `<…>`
6. **Body.** Then the **body**: the message's literal text, one space after the last tag. If the body itself starts with `[`, `<` or `\`, a single `\` is written in front of it. The body needs no other escaping. Emoji are real Unicode characters; an emoji that could not be identified is written as `�` (U+FFFD).
7. **Multi-line text.** Each line after the first in a message's text is written on its own line, starting with **exactly two spaces**, which are not part of the text.
8. **No trailing whitespace.** A message with media and no text ends right after its last tag.

### 2.3 System lines

| Form | Meaning |
|---|---|
| `0. System: [date YYYY-MM-DD Www]` | Date separator. `Www` is the English weekday abbreviation. |
| `0. System: [date YYYY-MM-DDTHH:MM Www "raw"]` | Separator that also shows a time (e.g. Hinge) |
| `… "raw"]` | The quoted raw label is added only when the date was **inferred**: from Today, Yesterday or a weekday name, or with the year inferred. |
| `0. System: [date 2026-11-22? Sun "Sunday"]` | A `?` after the value means the date is uncertain. |
| `0. System: [date ? "raw"]` | The label could not be resolved to a date. |
| `0. System: [gap]` | Messages may be missing here: two screenshots did not overlap (FR-MRG-3). |
| `0. System: TEXT` | Any other system message (encryption notice, "You matched", …). It may carry flag tags, and the body escaping rule applies. |

### 2.4 Tags

**Flag tags** mark uncertainty. Values that already appear on the line (time, date, reply target) are marked with a `?` suffix instead.

```
[?text 0.58]     text read with low confidence (the number is the lowest confidence among the listed fields)
[?emoji]         an emoji was not identified (written as �) or has low confidence
[?sender 0.61]   the side/sender assignment is uncertain
[?media 0.70]    the media kind is uncertain
[?time]          the app shows a time but it could not be read
[?date]          the message's date is unknown, despite any separator above it
[?partial]       only a cut-off part of the message was visible
[?bidi]          the text direction was ambiguous, so the word order may be wrong
[?dup]           possibly a duplicate of a neighbouring message, kept to be safe
[?order]         its position relative to its neighbours is uncertain
```

- Several fields can share one tag: `[?text,emoji 0.55]`.
- A flag that has a score is shown only when its confidence is below the display threshold (default 0.80, `--threshold`).
- Flags without a score (unidentified, unreadable, partial, unknown) are always shown.
- §6 lists every flag code and its marker.

**Reply tag:**

| Form | Meaning |
|---|---|
| `[reply: 5]` | Replies to message 5, resolved with high confidence |
| `[reply: 5?]` | Resolved to message 5, but with low confidence |
| `[reply: 5? Bar "how are"]` | Ambiguous. Best candidate 5; quoted sender and snippet shown. |
| `[reply: ? Bar "see you at 8 at the"]` | Unresolved. The target is probably outside the transcript. |
| `[reply: ? Bar <image>]` | Unresolved reply to an image (the media kind appears instead of, or before, a snippet) |

Snippets are written exactly as quoted. A trailing `…` means the quote itself was truncated.

**Media tags** have the form `<kind attr attr …>`.
- Each attribute is either positional or `key=value`.
- A value is a bare token, or a JSON-escaped string in quotes. Quotes are required when the value contains a space, `"`, `>` or `=`.
- Attributes are written in the order given below; unknown ones are left out.

| Kind | Form and attributes | Example |
|---|---|---|
| `image` | `<image [path]>` | `<image media/5-1.png>` |
| `video` | `<video [m:ss] [path]>` (path = cropped thumbnail) | `<video 0:31 media/7-1.png>` |
| `gif` | `<gif [path]>` | `<gif media/9-1.png>` |
| `sticker` | `<sticker [emoji] [path]>` | `<sticker 😂 media/4-1.png>` |
| `voice` | `<voice [m:ss]>` | `<voice 0:42>` |
| `audio` | `<audio [m:ss] ["title"]>` | `<audio 3:05 "song.mp3">` |
| `document` | `<document ["name"] [size] [pages=N]>` | `<document "report.pdf" 2.1MB pages=3>` |
| `link` | `<link "url" [title="…"]>` (a link-preview card) | `<link "https://x.com/a" title="Example">` |
| `location` | `<location ["label"]>` | `<location "Dizengoff Center">` |
| `contact` | `<contact ["name"]>` | `<contact "Dana">` |
| `poll` | `<poll ["question"]>` | `<poll "Pizza or sushi?">` |
| `call` | `<call [missed] [voice\|video] [m:ss]>` | `<call missed voice>` |
| `deleted` | `<deleted>` ("This message was deleted") | `<deleted>` |
| `view-once` | `<view-once [image\|video\|voice]>` | `<view-once image>` |
| `unknown` | `<unknown>`: a media element the plugin could not classify | `<unknown>` |
| `x-…` | Defined by a plugin (FR-MED-2). Same syntax. | `<x-hinge-prompt "My simple pleasures" answer="coffee at sunrise">` |

If a message has several media items (an album), it has several tags in a row.

### 2.5 Grammar (EBNF)

```ebnf
view        = header , { item } ;
header      = "# chatsocr-text/" , int , NL , { "#" , { char - NL } , NL } ;
item        = ( sysline | msgline ) , NL , { "  " , { char - NL } , NL } ;
sysline     = "0. System: " , ( datetag | "[gap]" | { flagtag , SP } , body ) ;
datetag     = "[date " , ( dateval , [ "?" ] , SP , wkday | "?" ) , [ SP , qstr ] , "]" ;
dateval     = yyyy , "-" , mm , "-" , dd , [ "T" , hhmm ] ;
msgline     = msgid , ". " , [ hhmm , [ "?" ] , SP ] , label , ": " ,
              { flagtag , SP } , [ replytag , SP ] , { mediatag , SP } , body ;
msgid       = int , { "." , int } ;            (* never exactly "0" *)
hhmm        = digit , digit , ":" , digit , digit ;
label       = lchar , { lchar } ;              (* see 2.2 rule 4 *)
flagtag     = "[?" , field , { "," , field } , [ SP , score ] , "]" ;
field       = "text" | "emoji" | "sender" | "media" | "time" | "date" | "partial" | "bidi" | "dup" | "order" ;
replytag    = "[reply: " , ( msgid , [ "?" ] | "?" ) , [ SP , label ] , [ SP , "<" , kind , ">" ] , [ SP , qstr ] , "]" ;
mediatag    = "<" , kind , { SP , attr } , ">" ;
kind        = "image" | "video" | "gif" | "sticker" | "voice" | "audio" | "document" | "link"
            | "location" | "contact" | "poll" | "call" | "deleted" | "view-once" | "unknown"
            | "x-" , token ;
attr        = [ token , "=" ] , ( token | qstr ) ;
body        = [ "\" ] , { char - NL } ;        (* "\" written iff the text starts with "[", "<" or "\" *)
qstr        = '"' , { char - ( '"' | "\" ) | "\" , char } , '"' ;   (* JSON string escaping *)
score       = digit , "." , digit , digit ;
```

Note on `replytag`: the optional label comes before any `<kind>` or quoted snippet, and labels cannot contain `<` or `"`. A parser can therefore take everything between the target and the first ` <`, ` "` or `]` as the label.

### 2.6 Parsing recipe

```python
MSG = re.compile(r'^(?P<id>\d+(?:\.\d+)*)\. (?:(?P<time>\d\d:\d\d)(?P<tq>\?)? )?(?P<who>[^:\n]+?): (?P<rest>.*)$')
SYS = re.compile(r'^0\. System: (?P<rest>.*)$')
TAG = re.compile(r'^(\[(?:[^\]"]|"(?:[^"\\]|\\.)*")*\]|<(?:[^>"]|"(?:[^"\\]|\\.)*")*>)(?: |$)')
```

1. Try `SYS` first, then `MSG`.
2. On `rest`, keep matching `TAG` at the start to strip off the tags.
3. What remains is the body. Remove one leading `\` if present.
4. Lines starting with two spaces continue the previous item's text.

The package also ships a reference parser: `chatsocr.textview.parse(path) -> list[dict]`.

### 2.7 What the text view leaves out

The text view is lossy on purpose. It drops:
- boxes and provenance;
- exact confidences above the threshold;
- the reply quote text when the reply is resolved;
- the source of each time;
- fingerprints;
- transcript-level warnings, apart from a summary comment line `# warnings: …` when there are any.

Agents that need these read the JSON.

---

## 3. JSON (`transcript.json`)

### 3.1 Conventions

- Field names are `snake_case`.
- Optional values are `null`, or left out when they are lists.
- `bbox` is `[x0, y0, x1, y1]` in integer pixels of the **source screenshot**: origin at the top-left, `x1`/`y1` exclusive.
- Dates are `YYYY-MM-DD` and times `HH:MM`. Both are local wall clock with no time zone (FR-TIM-10).
- Confidences are numbers from 0 to 1, rounded to 2 decimals.
- `start`/`end` text offsets count **Unicode code points** (Python `str` indices), not UTF-16 units.
- Readers **must ignore unknown fields**. A minor version may add optional fields; only a major version removes or changes the meaning of a field.
- The JSON Schema (draft 2020-12) ships in the package as `chatsocr/schema/transcript-1.schema.json`, with `$id` `urn:chatsocr:schema:transcript:1`. It is generated from the pydantic output models.

### 3.2 Top level

```json
{
  "schema_version": "1.0.0",
  "generator": {"name": "chatsocr", "version": "0.1.0",
                "ocr": {"backend": "tesseract", "version": "5.5.4", "model": "heb+eng/fast"}},
  "id_epoch": 1,
  "chat": { … },
  "sources": [ … ],
  "batches": [ … ],
  "items": [ … ],
  "warnings": [ … ],
  "stats": {"messages": 9, "system_items": 3, "flags": {"OCR_LOW_CONFIDENCE": 1, "REPLY_UNRESOLVED": 1}}
}
```

| Field | Meaning |
|---|---|
| `id_epoch` | Goes up by 1 only when IDs are renumbered (FR-MRG-8). References to message IDs are valid within one epoch. |
| `items` | **Every item of the conversation in order.** The order of this array is the conversation order. |
| `warnings` | Flags that apply to the whole transcript (§6), in the same shape as item flags plus an optional `item`, `source` or `batch` reference. |

### 3.3 `chat`

```json
"chat": {
  "kind": "direct",
  "app": {
    "id": "whatsapp", "platform": "android", "ui_epoch": "2024",
    "plugin": {"id": "whatsapp.android", "version": "1.0.0", "api_version": 1},
    "detection": {"mode": "auto", "confidence": 0.97,
                  "alternatives": [{"variant": "whatsapp/ios/dark", "confidence": 0.12}]}
  },
  "ui_direction": "ltr",
  "header_title": "Bar",
  "participants": [
    {"id": "p1", "label": "Bar", "name": "Bar", "side": "left", "is_self": false, "source": "cli"},
    {"id": "p2", "label": "Yuval", "name": "Yuval", "side": "right", "is_self": true, "source": "cli"}
  ],
  "date_order": {"value": "DMY", "source": "plugin_locale", "confidence": 1.0}
}
```

| Field | Values |
|---|---|
| `kind` | `direct` (P1), `group` (P2) |
| `detection.mode` | `auto`, `hint` (narrowed by `--app`/`--platform`) or `user` (chosen interactively) |
| `ui_direction` | `ltr` or `rtl` (a mirrored UI, e.g. WhatsApp in Hebrew) |
| `participants[].source` | `cli`, `api`, `user` (answered interactively) or `inferred` (FR-IN-3) |
| `participants[].label` | The cleaned name used in the text view |
| `participants[].name` | The original name |

### 3.4 `sources` and `batches`

```json
"sources": [{
  "id": "shot-1", "batch": "b1",
  "path": "in/Screenshot_20261124-164210.png", "sha256": "9f2c…", "size": [1080, 2400],
  "theme": "dark", "scale": {"dp_px": 2.75, "em_px": 42},
  "anchor": {"value": "2026-11-24T16:42:10", "source": "filename", "confidence": 0.9,
             "checks": {"exif": "2026-11-24T16:42:10", "status_bar": "16:42", "mtime": "2026-11-24T16:42:11"}},
  "placement": {"component": 0, "y": 0},
  "status": "ok"
}],
"batches": [{"id": "b1", "processed_at": "2026-11-24T17:00:03", "sources": ["shot-1", "shot-2"],
             "options": {"app": null, "interactive": false}}]
```

| Field | Values |
|---|---|
| `anchor.source` | `user`, `exif`, `filename`, `xmp`, `mtime` or `now` |
| `placement` | Where the screenshot sits on the merged vertical axis. `component` is the index of a group of screenshots that overlap one another; there is one group per piece between gaps. |
| `status` | `ok`, `partial` or `unparseable` |
| `processed_at` | The only field excluded from the determinism guarantee (NFR-DET-1) |

### 3.5 Items

Every item has these common fields:

| Field | Meaning |
|---|---|
| `id` | **Message:** the dotted-decimal message ID (§4), a string. <br>**System item:** `s<k>`, an opaque string that is never reused. |
| `type` | `message`, `date_separator`, `system` or `gap` |
| `provenance` | A list of observations: `{"source", "bbox", "cut": "none"\|"top"\|"bottom", "selected": bool}`. The selected observation is the one whose content was used. |
| `confidence` | The lowest confidence among the item's key fields |
| `flags` | A list of `{"code", "field", "confidence"?, "detail"?}` (§6) |

#### `message`

```json
{
  "id": "5", "type": "message",
  "sender": "Bar", "sender_id": "p1", "side": "left", "outgoing": false,
  "timestamp": {"date": "2026-11-24", "date_source": "separator_relative",
                "time": "08:40", "time_source": "bubble", "raw": "8:40", "confidence": 0.99},
  "text": "I'm in the gym 💪", "dir": "ltr",
  "spans": [{"type": "emoji", "start": 15, "end": 16, "value": "💪", "confidence": 0.93,
             "alternatives": [{"value": "🦾", "confidence": 0.41}],
             "source": "shot-1", "bbox": [610, 1120, 650, 1160]}],
  "media": [{"id": "5/m1", "kind": "image", "attrs": {}, "file": "media/5-1.png",
             "source": "shot-1", "bbox": [40, 300, 700, 1100], "phash": "c3a1e0f4b2d19a7e", "confidence": 0.98}],
  "reply_to": null,
  "fingerprint": {"side": "left", "kind": "image", "text": "i'm in the gym 💪", "time": "08:40",
                  "phash": ["c3a1e0f4b2d19a7e"]},
  "provenance": [{"source": "shot-1", "bbox": [30, 290, 720, 1180], "cut": "none", "selected": true}],
  "confidence": 0.93, "flags": []
}
```

| Field | Meaning |
|---|---|
| `sender`, `sender_id` | Participant label and ID. Both are given so an agent does not need to join tables. |
| `side` | `left` or `right` |
| `outgoing` | `true`, `false` or `null` (unknown): evidence such as delivery ticks |
| `timestamp.date` / `.time` | Each may be `null` when unknown. A missing time is never invented. |
| `timestamp.date_source` | `separator`, `separator_relative`, `floating_pill`, `neighbour`, `user` or `null` |
| `timestamp.time_source` | `bubble`, `separator`, `user` or `null` |
| `timestamp.raw` | The OCR string of the time, before parsing |
| `text` | Logical order, NFC. `""` when there is no text (e.g. media only). The **reply quote is never part of `text`.** |
| `dir` | `ltr` or `rtl`: the paragraph direction (FR-OCR-2) |
| `spans` | Present only when needed. `type` is `emoji` (identified, or `value` `"\uFFFD"` when not) or `low_confidence` (a run of text with low OCR confidence). |
| `media[].kind` | As in §2.4, including plugin-defined `x-…` kinds |
| `media[].attrs` | Standard keys: `duration_s` (int), `file_name`, `size_text`, `pages`, `url`, `title`, `label`, `emoji`, `call_type` (`voice`/`video`), `missed` (bool), `question`, `view_once_kind`. Plugin `x-` kinds may add their own keys. |
| `media[].file` | The cropped image, or `null` (with `--no-media`, or for kinds that have no picture, e.g. voice) |
| `fingerprint` | Normalized matching keys used to align appended batches (FR-OUT-4). Not an identity. |
| `revisions` | Optional. When an appended batch sees the message again more clearly: `[{"batch", "field", "old", "reason"}]` |

#### `reply_to`

```json
"reply_to": {
  "status": "resolved",
  "id": "5", "confidence": 0.93, "method": ["quote_text", "thumbnail"],
  "candidates": [{"id": "5", "score": 0.93}, {"id": "3", "score": 0.41}],
  "quote": {"sender": "Bar", "sender_id": "p1", "text": "I'm in the gym", "truncated": false,
            "media_kind": "image", "thumb_phash": "c3a1e0f4b2d19a7e",
            "source": "shot-1", "bbox": [700, 1400, 1040, 1480]}
}
```

| Field | Meaning |
|---|---|
| `status` | `resolved`, `ambiguous` (then `id` is the best candidate) or `unresolved` (then `id` is `null`) |
| `method` | Which evidence was used. `user` is included if the user decided interactively. |
| `quote` | Always kept, whatever the status. This is the evidence. |

#### `date_separator`

```json
{"id": "s1", "type": "date_separator", "date": "2026-11-22", "time": null, "weekday": "Sun",
 "label": "Sunday", "resolution": "weekday_label", "anchor_source": "shot-1",
 "provenance": [{"source": "shot-1", "bbox": [470, 212, 610, 262], "cut": "none", "selected": true}],
 "confidence": 0.97, "flags": []}
```

`resolution` is one of `absolute`, `absolute_year_inferred`, `today`, `yesterday`, `weekday_label`, `user` or `unresolved` (then `date` is `null`).

#### `system`

```json
{"id": "s2", "type": "system", "subtype": "disappearing_messages",
 "text": "You turned off disappearing messages.", "dir": "ltr",
 "provenance": [ … ], "confidence": 0.96, "flags": []}
```

`subtype` comes from the plugin's vocabulary. Common values are `encryption_notice`, `missed_call`, `match_created`, `disappearing_messages` and `unknown`. Plugins may add `x-…` subtypes.

#### `gap`

```json
{"id": "s3", "type": "gap", "reason": "no_overlap", "confidence": 0.8, "provenance": [], "flags": [{"code": "GAP_POSSIBLE", "field": "order"}]}
```

`reason` is one of `no_overlap`, `alignment_weak` or `append_unanchored`.

---

## 4. Message IDs

Message IDs are **dotted-decimal ordinals**, written as strings: `"1"`, `"7"`, `"4.1"`, `"0.3"`, `"4.0.1"`.

1. **Ordering.** Compare the dot-separated parts as integers, from left to right. When one ID is a prefix of another, the shorter comes first. So: `0.1 < 1 < 1.1 < 1.1.1 < 1.2 < 2`.
2. **First run.** Messages are numbered `1, 2, 3, …`. `0` is never a message ID; it is the system-line marker in the text view.
3. **Appending at the end.** New messages after the last one continue from the highest leading number: `N+1, N+2, …`.
4. **Inserting before or between.** Insert *k* new messages between existing neighbours P and Q (P = virtual `0` before the first message):
   1. Use `P.1 … P.k` if `P.k` sorts before Q.
   2. Otherwise (Q itself starts with P, e.g. P = `4`, Q = `4.1`), add `.0` parts until the IDs sort before Q: `4.0.1 … 4.0.k`.
   Examples:
   - Between `4` and `5`: `4.1, 4.2`
   - Before `1`: `0.1, 0.2, …`
   - Before `0.1`: `0.0.1, …`
5. **Stability.** An ID, once given out, never changes and is never reused within an `id_epoch`. The only exception is an explicit `chatsocr renumber` (P2), which writes an ID map and increments `id_epoch`.
6. **System item IDs** (`s1`, `s2`, …) are given out in creation order, are never reused, and say nothing about position. Position comes only from the order of the `items` array.

Example: the transcript in §2.1, after appending a batch that reaches further up (two older messages, taken on Wednesday 2026-11-25, so "Saturday" is 2026-11-21) and that also shows one message between 6 and 7:

```
0. System: [date 2026-11-21 Sat "Saturday"]
0.1. 22:10 Bar: are you coming tomorrow?
0.2. 22:15 Yuval: yes!
0. System: [date 2026-11-22 Sun "Sunday"]
1. 21:04 Bar: מה קורה? how was the trip
…
6. 16:33 Yuval: [reply: 5] you looks beautiful
6.1. 16:34 Yuval: seriously
7. 16:36? Bar: [?text 0.58] תודה!! נתראה ב-8
```

---

## 5. Time semantics

| Rule | Detail |
|---|---|
| No time zone | Every date and time is local wall-clock time with no time zone. Durations across a daylight-saving change may be off by an hour. |
| Day only | A message with a date but no time (`time: null`) means the app showed only a separator date. |
| Relative labels | Resolved against the anchor of the screenshot that contains them (FR-TIM-3). `resolution` and `label` keep the evidence. |
| Weekday names | A weekday name means the most recent such day **strictly before** the anchor date. |
| Monotonic order | Dates and times never go backwards in `items` order unless flagged (`TIME_NON_MONOTONIC`). |

---

## 6. Flag codes

**Level:**
- `info`: expected degradation, e.g. inferred participants.
- `warn`: an uncertain value is in the output.
- `error`: something is missing or failed.

**Shown:** the marker in the text view. *scored* means it is shown only when the confidence is below the threshold. — means JSON only.

**Transcript, source and batch level** (in `warnings`):

| Code | Level | Shown | Meaning |
|---|---|---|---|
| `DETECTION_AMBIGUOUS` | warn | header | Several variants scored close together; the top one was used |
| `DETECTION_MISMATCH` | warn | header | The app the user chose scored below the floor |
| `PARTICIPANTS_INFERRED` | info | header | Names were inferred from the chat header / "Me" |
| `SELF_SIDE_CONFLICT` | warn | header | `--me` disagrees with the outgoing-message evidence |
| `ANCHOR_FALLBACK_NOW` | warn | header | No capture time found; the current time was used |
| `ANCHOR_LOW_CONFIDENCE` | info | — | The anchor came from the file modification time |
| `ANCHOR_CONFLICT` | warn | header | Metadata and the status-bar clock disagree |
| `DATE_ORDER_AMBIGUOUS` | warn | header | Day-first vs month-first was guessed |
| `SCREEN_UNPARSEABLE` | error | header | A screenshot could not be parsed and was skipped |
| `PLUGIN_ERROR` | error | header | The plugin raised an exception on a screenshot |
| `OVERLAP_LOW_CONFIDENCE` | warn | — | The alignment between two screenshots is weak |
| `APPEND_ALIGNMENT_WEAK` | warn | header | An appended batch matched the existing transcript only weakly |
| `APPEND_FORCED` | warn | header | The append went ahead with `--force` despite a chat mismatch |

**Item level** (in `items[].flags`):

| Code | Field | Level | Shown |
|---|---|---|---|
| `OCR_LOW_CONFIDENCE` | text | warn | `[?text s]` scored |
| `BIDI_AMBIGUOUS` | text | warn | `[?bidi]` |
| `LINEBREAK_UNCERTAIN` | text | info | — |
| `EMOJI_UNIDENTIFIED` | text | warn | `[?emoji]` and `�` |
| `EMOJI_LOW_CONFIDENCE` | text | warn | `[?emoji s]` scored |
| `SIDE_UNCERTAIN` | sender | warn | `[?sender s]` scored |
| `PARTIAL_MESSAGE` | text | warn | `[?partial]` |
| `MEDIA_KIND_UNCERTAIN` | media | warn | `[?media s]` scored |
| `ELEMENT_UNRECOGNIZED` | media | warn | `<unknown>` |
| `TIME_UNREADABLE` | time | warn | `[?time]` |
| `TIME_LOW_CONFIDENCE` | time | warn | `HH:MM?` scored |
| `TIME_CORRECTED` | time | warn | `HH:MM?` (raw value kept) |
| `TIME_NON_MONOTONIC` | time | warn | `HH:MM?` |
| `DATE_UNKNOWN` | date | warn | `[?date]` |
| `DATE_LOW_CONFIDENCE` | date | warn | `[date …?]` scored |
| `DATE_LABEL_UNPARSED` | date | warn | `[date ? "raw"]` |
| `YEAR_INFERRED` | date | info | raw label shown |
| `REPLY_LOW_CONFIDENCE` | reply | warn | `[reply: N?]` |
| `REPLY_AMBIGUOUS` | reply | warn | `[reply: N? …]` |
| `REPLY_UNRESOLVED` | reply | info | `[reply: ? …]` |
| `DUPLICATE_UNCERTAIN` | order | warn | `[?dup]` |
| `ORDER_UNCERTAIN` | order | warn | `[?order]` |
| `GAP_POSSIBLE` | order | warn | `0. System: [gap]` |

New codes may be added only in a minor schema version and must be added to this table. Consumers must treat an unknown code as `warn`.

---

## 7. Versioning

| Artifact | Version | Rules |
|---|---|---|
| JSON | `schema_version`, semver | **MAJOR:** fields removed, renamed or with a new meaning; ID rules changed. **MINOR:** new optional fields, flag codes, media kinds or system subtypes. **PATCH:** documentation only. |
| Text view | `chatsocr-text/<major>` in the first line | A new major version is needed only if a parser written to this grammar would misread the new output. |

Phase 2 adds reactions (`[react: ❤️ Bar]`), markers (`[edited]`, `[forwarded]`) and group senders. All of these are **minor** additions: new optional JSON fields and new tag kinds that a parser following §2.6 skips correctly.
