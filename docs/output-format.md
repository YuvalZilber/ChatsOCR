# ChatsOCR: Output Format Specification

| | |
|---|---|
| Status | Draft 2, revised after an independent review |
| Date | 2026-11-24 |
| Versions specified | JSON `schema_version` 1.0.0 · text view `chatsocr-text/1` |
| Related | [Requirements](requirements.md) (FR-OUT, FR-MRG-5, FR-UNC) · [High-level design](high-level-design.md) |

This document is the contract between ChatsOCR and anything that reads its output: AI agents, scripts and people. Plugins never write this format. The engine renders it from the merged transcript, so every app produces the same format.

## 1. Files

One run writes into the output directory (`-o DIR`, default `./chatsocr-out/`; in append mode, the transcript's own directory):

| File | Purpose |
|---|---|
| `transcript.json` | **Source of truth.** Lossless: positions, confidences, provenance, flags, and the fingerprints and image signatures needed for later appends. |
| `transcript.txt` | **Text view.** Compact, line-based, cheap in tokens. Generated from the JSON; never edit it by hand. |
| `media/` | Images cropped from the screenshots, named `<message-id>-<n>.png` (e.g. `5-1.png`, `4.1-1.png`). Paths in both files are relative to the transcript. |
| `*.bak`, `media/.replaced/<batch>/` | Append mode only: the previous `transcript.json` and `transcript.txt`, and any media files that were replaced. Written before anything is overwritten. |

All text is UTF-8 without BOM and always in **logical Unicode order**, NFC-normalized, with no bidi control characters. U+200E, U+200F, U+061C, U+202A–U+202E and U+2066–U+2069 are stripped. U+200D (ZWJ) and U+FE0F (VS16) are kept, because emoji sequences need them. Leading and trailing whitespace is trimmed from each text, and trailing whitespace from each line.

---

## 2. Text view (`transcript.txt`)

### 2.1 Example

A WhatsApp chat (per-message times, one date separator per day):

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
8. 16:40 Yuval: [reply: ? Bar "see you at 8 at the"…] <link "https://maps.app.goo.gl/x"> here
9. 16:41 Bar: \[joke] two lines
  and a second line
```

The user's original example comes from an app that shows times **only in separators**. Rendered by ChatsOCR, it keeps the same shape:

```
0. System: [date 2026-11-24T08:33 Tue "Today 8:33 AM"]
1. Bar: how are you?
2. Yuval: i am great, how are you?
3. Bar: <image media/3-1.png> I'm in the gym
0. System: [date 2026-11-24T16:33 Tue "Today 4:33 PM"]
4. Yuval: [reply: 3] you looks beautiful
```

It differs from the user's example in three deliberate ways, and the user still needs to approve them ([requirements §13](requirements.md#13-open-questions-for-the-user) Q1):

1. **Dates are wrapped as `[date …]`.** This keeps date lines distinguishable from other system text.
2. **The weekday and the raw label are added.** The raw label appears only when the date was *inferred*, as here from "Today".
3. **Images carry the link to their crop.** With `--no-media` the tag is exactly `<image>`.

### 2.2 Line rules

1. **Header.** Lines starting with `#` are comments with metadata. The first line is always `# chatsocr-text/<major>`. A reader may ignore every header line. `# warnings: …` lists transcript-level flags, if there are any.
2. **One item per line.**
   - A **message** line starts with its ID: `5.`, or `4.1.` for a dotted ID (§4).
   - A **system** line starts with `0. System:`. System items cannot be referred to from the text view; their IDs live in the JSON.
3. **Time slot.** If the app shows a time on each message, the time follows the ID as 24-hour `HH:MM`. A trailing `?` marks an uncertain time. A message without a time of its own has no slot.
4. **Date of a message.** It is the date of the nearest `[date …]` line above it, **never across a `[gap]` line**. Wherever a message's date comes from somewhere other than a separator (a pinned date pill), the renderer writes a **date context line** (`[date … from=pill]`), so this rule always gives the right date. A message whose date is unknown carries `[?date]`.
5. **Sender.** The sender is the participant's **label**, followed by `: `. Labels are cleaned once, when the participant is created, and the original name stays in the JSON:
   1. Remove `:`, `"`, `[`, `]`, `<`, `>` and line breaks.
   2. Trim the result.
   3. Prefix `_` if it starts with a digit.
   4. Use `p<N>` if it is empty.
   5. Rename `System` to `System_`.
   6. Append ` (2)`, ` (3)`, … if it collides with another participant's label.
6. **Tags.** After the sender come zero or more **tags**, separated by single spaces, in this order:
   1. flag tags `[?…]`
   2. a reply tag `[reply: …]`
   3. media tags `<…>`
   4. any other tags

   **A reader must skip tags it does not know.** This lets later versions add tags (§7).
7. **Body.** After the tags comes the **body**, the message's literal text, after one space.
   - If the body would start with `[`, `<` or `\`, a single `\` is written in front of it. No other escaping exists.
   - Emoji are written as real Unicode characters. An emoji that could not be identified is written as `�` (U+FFFD).
8. **Multi-line text.** Each further line of a message's text goes on its own line, starting with **exactly two spaces** that are not part of the text. An **empty** text line is written as an empty line. An item never starts with an empty line.
9. **No trailing whitespace.** A message with media and no text ends right after its last tag.

### 2.3 System lines

| Form | Meaning |
|---|---|
| `0. System: [date YYYY-MM-DD Www]` | Date separator. `Www` is the English weekday abbreviation. |
| `0. System: [date YYYY-MM-DDTHH:MM Www "raw"]` | A separator that also shows a time. The quoted raw label appears only when the date was **inferred** (from Today, Yesterday, a weekday name, or an inferred year). |
| `0. System: [date 2026-11-22? Sun "Sunday"]` | A `?` after the value marks the whole value (date and time) as uncertain. An inferred date is uncertain whenever its anchor is (§5). |
| `0. System: [date ? "raw"]` | The label could not be resolved |
| `0. System: [date 2026-11-24 Tue from=pill]` | **Date context line.** Not a separator in the chat. It tells the reader which date applies to the messages that follow (rule 4). |
| `0. System: [gap]` | Messages may be missing here: two screenshots did not overlap (FR-MRG-3). Dates are never carried across it. |
| `0. System: TEXT` | Any other system message (encryption notice, "You matched", …). The body rules apply. |

Flag tags may follow a date or gap tag (`0. System: [date 2026-11-24 Tue] [?dup]`), or come before system text.

### 2.4 Tags

**Flag tags** mark uncertainty. A value that is already printed on the line (time, date, reply target) is marked with a `?` suffix instead.

```
[?text 0.58]     text read with low confidence (the number is the lowest confidence among the listed fields)
[?emoji]         an emoji was not identified (written as �) or has low confidence
[?sender 0.61]   the side/sender assignment is uncertain
[?media 0.70]    the media kind is uncertain
[?time]          the app shows a time, but it could not be read
[?date]          the message's date is unknown
[?partial]       only a cut-off part of the message was ever visible
[?bidi]          text direction was ambiguous, so the word order may be wrong
[?dup]           may be a duplicate of a nearby message, kept to be safe
[?order]         its position relative to its neighbours is uncertain
```

- Several fields can share one tag: `[?text,emoji 0.55]`.
- A flag that has a score is shown only when its confidence is below the display threshold (default 0.80, `--threshold`).
- Flags without a score (unidentified, unreadable, partial, unknown) are always shown.

**Reply tags.** The quoted sender, media kind and snippet are shown whenever the target is not a confident ID:

| Form | Meaning |
|---|---|
| `[reply: 5]` | Replies to message 5, resolved with high confidence |
| `[reply: 5?]` | Resolved to message 5 with low confidence |
| `[reply: ~5\|3 Bar "how are"]` | **Ambiguous.** Candidates, best first. |
| `[reply: ? Bar "see you at 8 at the"…]` | **Unresolved.** The target is probably outside the transcript. |
| `[reply: ? Bar <image>]` | Unresolved reply to an image |

- The snippet is written exactly as quoted, with its own characters kept.
- An ellipsis **after** the closing quote (`"…"…`) means the app cut the quote short.

**Media tags.**
- Each media tag is `<kind attr attr …>`.
- An attribute is positional, or `key=value`.
- A value is a bare token, or a JSON-escaped string in quotes. A value containing a space, `"`, `<`, `>`, `[`, `]` or `=` must be quoted.
- Core kinds write their attributes in the order below and leave out unknown ones.

| Kind | Form | Example |
|---|---|---|
| `image` | `<image [path]>` | `<image media/5-1.png>` |
| `video` | `<video [m:ss] [path]>` (path = the cropped thumbnail) | `<video 0:31 media/7-1.png>` |
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
| `view-once` | `<view-once [image\|video\|voice]>` | `<view-once image>` |
| `unknown` | `<unknown>`: a media element the plugin could not classify | `<unknown>` |
| `deleted` | `<deleted>` (**P2**). In P1 a deleted message's visible text ("This message was deleted") is output as ordinary text. | `<deleted>` |
| `x-…` | Defined by a plugin (FR-MED-2). **Every attribute is `key=value`, in the order the plugin declares.** The declarations are stored in the JSON (`chat.media_kinds`), so `render` works without the plugin. | `<x-hinge-like on=prompt prompt="My simple pleasures" answer="coffee at sunrise">` |

A message with several media items (an album) has several media tags in a row.

### 2.5 Grammar (EBNF, normative)

```ebnf
(* ---- terminals ---- *)
NL       = "\n" ;   SP = " " ;
digit    = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
int      = "0" | ( digit - "0" ) , { digit } ;
char     = ? any Unicode scalar value except NL ? ;
tokchar  = char - ( SP | '"' | "<" | ">" | "[" | "]" | "=" ) ;
token    = tokchar , { tokchar } ;
lchar    = char - ( ":" | '"' | "[" | "]" | "<" | ">" ) ;
label    = ( lchar - ( digit | SP ) ) , { lchar } ;            (* cleaned label, rule 5; no trailing SP *)
qstr     = '"' , { ( char - ( '"' | "\" ) ) | "\" , char } , '"' ;   (* JSON string escaping *)
wkday    = "Mon" | "Tue" | "Wed" | "Thu" | "Fri" | "Sat" | "Sun" ;
date     = 4 * digit , "-" , 2 * digit , "-" , 2 * digit ;
hhmm     = 2 * digit , ":" , 2 * digit ;
score    = digit , "." , 2 * digit ;

(* ---- structure ---- *)
view     = header , { item } ;
header   = "# chatsocr-text/" , int , NL , { "#" , { char } , NL } ;
item     = ( sysline | msgline ) , NL , { contline } ;
contline = ( "  " , { char } | (* empty text line *) ) , NL ;

sysline  = "0. System: " , ( ( datetag | gaptag ) , { SP , flagtag }
                           | { flagtag , SP } , body ) ;
datetag  = "[date " , ( date , [ "T" , hhmm ] , [ "?" ] , SP , wkday | "?" ) ,
           [ SP , qstr ] , [ SP , "from=" , token ] , "]" ;
gaptag   = "[gap]" ;

msgline  = msgid , ". " , [ hhmm , [ "?" ] , SP ] , label , ": " , ( tags , [ SP , body ] | body ) ;
msgid    = int , { "." , int } ;                               (* never exactly "0" *)

tags     = tag , { SP , tag } ;
tag      = flagtag | replytag | mediatag | othertag ;          (* written in this order *)
flagtag  = "[?" , field , { "," , field } , [ SP , score ] , "]" ;
field    = "text" | "emoji" | "sender" | "media" | "time" | "date" | "partial" | "bidi" | "dup" | "order" ;
replytag = "[reply: " , target , [ SP , label ] , [ SP , "<" , kind , ">" ] , [ SP , qstr , [ "…" ] ] , "]" ;
target   = msgid , [ "?" ] | "~" , msgid , { "|" , msgid } | "?" ;
mediatag = "<" , kind , { SP , attr } , ">" ;
kind     = "image" | "video" | "gif" | "sticker" | "voice" | "audio" | "document" | "link" | "location"
         | "contact" | "poll" | "call" | "view-once" | "unknown" | "deleted" | "x-" , token ;
attr     = [ token , "=" ] , ( token | qstr ) ;
othertag = "[" , { char - ( "]" | '"' ) | qstr } , "]"
         | "<" , { char - ( ">" | '"' ) | qstr } , ">" ;       (* unknown future tags: readers skip them *)
body     = ( "\" , char | char - ( "[" | "<" | "\" | SP ) ) , { char } ;
```

**Disambiguation:**
- A tag is recognized by its prefix: `[?` is a flag, `[reply: ` a reply, `<` followed by a known kind or `x-` a media tag. Any other bracketed tag is an `othertag`.
- The body cannot start with an unescaped `[` or `<`, so the boundary between tags and body is unique.
- In a reply tag, the label runs up to the first ` <`, ` "` or `]`. Labels cannot contain `<`, `"` or `]`.

### 2.6 Parsing recipe

```python
SYS = re.compile(r'^0\. System: (?P<rest>.*)$')
MSG = re.compile(r'^(?P<id>\d+(?:\.\d+)*)\. (?:(?P<time>\d\d:\d\d)(?P<tq>\?)? )?(?P<who>[^:\n]+?): (?P<rest>.*)$')
TAG = re.compile(r'^(\[(?:[^\]"]|"(?:[^"\\]|\\.)*")*\]|<(?:[^>"]|"(?:[^"\\]|\\.)*")*>)(?: |$)')
```

0. Skip the header: leading lines that start with `#`.
1. A line that is **empty** or **starts with two spaces** continues the previous item's text (with the two spaces removed; an empty line is an empty text line).
2. Otherwise try `SYS` first, then `MSG`.
3. On `rest`, keep matching `TAG` at the start to take tags off, and classify each by its prefix (§2.5).
4. What remains is the body. Remove one leading `\` if present.
5. Dates: track the last `[date …]` line and reset it at `[gap]`.

The package ships a reference parser: `chatsocr.render.textview.parse(text: str) -> list[dict]` (and `parse_file(path)`).

### 2.7 What the text view leaves out

The text view is lossy on purpose. It leaves out:
- boxes and provenance;
- confidences above the threshold;
- the quote text of resolved replies;
- the sources of times and dates;
- fingerprints and signatures;
- the details of transcript-level warnings (only the codes appear, in the `# warnings:` header line).

Agents that need these read the JSON.

---

## 3. JSON (`transcript.json`)

### 3.1 Conventions

- Field names are `snake_case`. Optional values are `null`, or left out when they are lists.
- **`bbox`** is `[x0, y0, x1, y1]` in integer pixels of the **source screenshot**: top-left origin, `x1` and `y1` exclusive.
- **Dates** are `YYYY-MM-DD` and **times** `HH:MM`, as local wall clock with no time zone (FR-TIM-10).
- **Confidences** are numbers from 0 to 1, rounded to 2 decimals.
- **`start`/`end` offsets** count **Unicode code points** (Python `str` indices), not UTF-16 units.
- **Unknown fields:** readers must ignore them, and ChatsOCR keeps them when it rewrites a transcript in append mode. A minor version may add optional fields; only a major version removes fields or changes what they mean.
- **The JSON Schema** (draft 2020-12) ships in the package as `chatsocr/schema/transcript-1.schema.json`, with `$id` `urn:chatsocr:schema:transcript:1`.

### 3.2 Top level

```json
{
  "schema_version": "1.0.0",
  "generator": {"name": "chatsocr", "version": "0.1.0"},
  "id_epoch": 1,
  "chat": { … },
  "sources": [ … ],
  "batches": [ … ],
  "items": [ … ],
  "warnings": [ … ],
  "stats": {"messages": 9, "system_items": 3,
            "flags": {"OCR_LOW_CONFIDENCE": 1, "TIME_LOW_CONFIDENCE": 1, "REPLY_UNRESOLVED": 1}}
}
```

| Field | Meaning |
|---|---|
| `id_epoch` | Goes up by one only when IDs are renumbered (FR-MRG-8). Message IDs are valid within one epoch. |
| `items` | **Every item of the conversation, in order.** The order of this array *is* the conversation order (§4 rule 1). |
| `warnings` | Flags for the whole transcript, a source or a batch (§6), each with an optional `source`, `batch` or `item` reference |

### 3.3 `chat`

```json
"chat": {
  "kind": "direct",
  "app": "whatsapp",
  "participants": [
    {"id": "p1", "label": "Bar", "name": "Bar", "side": "left", "is_self": false, "source": "cli"},
    {"id": "p2", "label": "Yuval", "name": "Yuval", "side": "right", "is_self": true, "source": "cli"}
  ],
  "date_order": {"value": "DMY", "source": "plugin_locale", "confidence": 1.0},
  "media_kinds": {}
}
```

| Field | Values |
|---|---|
| `kind` | `direct` (P1), `group` (P2) |
| `participants[].side` | The side on the **first** batch. Each later batch maps its own sides to participants, and the mapping is recorded per source (§3.4). |
| `participants[].source` | `cli`, `api`, `user` (asked interactively) or `inferred` |
| `participants[].label` | The cleaned name used in the text view (§2.2 rule 5) |
| `date_order.source` | `plugin_locale`, `heuristic`, `user` or `default` |
| `media_kinds` | The `x-` kind declarations of every plugin used: `{"x-hinge-like": {"has_picture": true, "attr_order": ["on", "prompt", "answer"]}}` |

### 3.4 `sources` and `batches`

```json
"sources": [{
  "id": "shot-1", "batch": "b1",
  "path": "in/Screenshot_20261124-164210.png", "sha256": "9f2c…", "size": [1080, 2400],
  "variant": {"platform": "android", "theme": "dark", "ui_epoch": "2024", "plugin": "whatsapp.android"},
  "ui": {"direction": "ltr", "lang": "en", "lang_source": "inferred"},
  "sides": {"left": "p1", "right": "p2"},
  "scale": {"dp_px": 2.75, "em_px": 55},
  "anchor": {"value": "2026-11-24T16:42:10", "source": "exif", "confidence": 0.95,
             "checks": {"filename": "2026-11-24T16:42:10", "status_bar": "16:42", "mtime": "2026-11-24T16:42:11"}},
  "placement": {"component": 0, "y": 0},
  "status": "ok"
}],
"batches": [{
  "id": "b1", "processed_at": "2026-11-24T17:00:03", "sources": ["shot-1", "shot-2"],
  "generator": "0.1.0",
  "plugins": [{"id": "whatsapp.android", "version": "1.0.0", "api_version": [0, 1]}],
  "ocr": {"backend": "tesseract", "version": "5.5.4", "model": "heb+eng/fast"},
  "detection": {"mode": "auto", "app": "whatsapp", "confidence": 0.97,
                "alternatives": [{"app": "telegram", "confidence": 0.08}]},
  "options": {"now": "2026-11-24T17:00:00", "interactive": false, "threshold": 0.8,
              "config": {"detection": {"accept": 0.6, "margin": 0.15, "floor": 0.25}}},
  "removed_items": []
}]
```

| Field | Meaning |
|---|---|
| `anchor.source` | `user`, `exif`, `filename`, `xmp`, `mtime` or `now`. `checks` keeps the values that were not chosen. |
| `placement` | The screenshot's position on the merged vertical axis. `component` is a group of screenshots that overlap; there is one group per piece between gaps. |
| `status` | `ok`, `partial` or `unparseable` |
| `detection.mode` | `auto`, `hint` (`--app`/`--platform`) or `user` (chosen interactively) |
| `options.now` | The run's "now". It is an **input**: pin it with `--now` to reproduce a run. |
| `removed_items` | IDs of `gap` items deleted because this batch closed the gap (§4) |
| `processed_at` | The only field outside the determinism guarantee (NFR-DET-1) |

### 3.5 Items

Common fields:

| Field | Meaning |
|---|---|
| `id` | Message: the dotted-decimal ID (§4), as a string. System item: `s<k>`, opaque and never reused. |
| `type` | `message`, `date_separator`, `system` or `gap` |
| `provenance` | List of observations: `{"source", "bbox", "anchor_bbox", "cut": "none"\|"top"\|"bottom"\|"both", "selected": bool}` |
| `confidence` | Lowest confidence among the item's key fields |
| `flags` | List of `{"code", "field", "confidence"?, "detail"?}` (§6) |

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
             "source": "shot-1", "bbox": [40, 300, 700, 1100],
             "sig": {"phash": "c3a1e0f4b2d19a7e", "phash_center": "c3a1e2f4b2d1987e",
                     "hsv": [0.31, 0.12, 0.08, 0.05, 0.04, 0.03, 0.03, 0.02, 0.02, 0.02, 0.03, 0.05, 0.06, 0.06, 0.04, 0.04]},
             "confidence": 0.98}],
  "reply_to": null,
  "fingerprint": {"sender_id": "p1", "kind": "image", "text": "i'm in the gym 💪", "time": "08:40"},
  "provenance": [{"source": "shot-1", "bbox": [30, 290, 720, 1180], "anchor_bbox": [30, 290, 720, 1180],
                  "cut": "none", "selected": true}],
  "confidence": 0.93, "flags": []
}
```

| Field | Meaning |
|---|---|
| `sender`, `sender_id` | The participant's label and ID. Both are present, so an agent never needs to join tables. |
| `side` | The side on the source screenshot: `left` or `right` |
| `outgoing` | Evidence such as delivery ticks: `true`, `false`, or `null` if unknown |
| `timestamp.date`, `.time` | Each may be `null`. A missing time is never invented. `time: null` means the app showed no time for this message; with the flag `TIME_UNREADABLE` it means the time was there but could not be read. |
| `timestamp.date_source` | `separator`, `separator_relative` (resolved from Today, Yesterday or a weekday), `floating_pill`, `user` or `null` |
| `timestamp.time_source` | `bubble`, `user` or `null` |
| `timestamp.raw` | The time string as read by OCR, before parsing |
| `text` | Logical order, NFC, or `""` when there is no text. **A reply quote is never part of `text`.** Blank lines and line breaks are kept as `\n`. |
| `dir` | `ltr` or `rtl`: the paragraph direction (FR-OCR-2) |
| `spans` | Present only when needed. `type` is `emoji` (`value` is `"\uFFFD"` if unidentified) or `low_confidence` (a run of text OCR read with low confidence). |
| `media[].kind` | A kind from §2.4, including `x-…` kinds |
| `media[].attrs` | Standard keys: `duration_s` (int), `file_name`, `size_text`, `pages`, `url`, `title`, `label`, `emoji`, `call_type` (`voice`/`video`), `missed` (bool), `question`, `view_once_kind`. `x-` kinds use the keys their plugin declares. |
| `media[].file` | The crop, or `null` (`--no-media`, or a kind without a picture) |
| `media[].sig` | Image signature, used to match reply thumbnails and to align appends without pixels (FR-OUT-4): `phash` of the whole crop, `phash_center` of its center square, and `hsv`, a 16-bin histogram |
| `fingerprint` | Normalized keys for matching appended batches. It does not identify the message. |
| `revisions` | Optional. When a later batch saw this message more clearly: `[{"batch", "field", "old", "reason"}]` |

#### `reply_to`

```json
"reply_to": {
  "status": "resolved",
  "id": "5", "confidence": 0.93, "method": ["quote_text", "thumbnail"],
  "candidates": [{"id": "5", "score": 0.93}, {"id": "3", "score": 0.41}],
  "quote": {"sender": "Bar", "sender_id": "p1", "text": "I'm in the gym 💪", "truncated": false, "partial": false,
            "media_kind": "image", "sig": {"phash_center": "c3a1e2f4b2d1987e", "hsv": [ … ]},
            "source": "shot-1", "bbox": [700, 1400, 1040, 1480]}
}
```

| Field | Meaning |
|---|---|
| `status` | `resolved` (with `REPLY_LOW_CONFIDENCE` when confidence < 0.85), `ambiguous` (`id` = the best candidate), or `unresolved` (`id` = `null`) |
| `method` | The evidence used: `quote_text`, `thumbnail`, `kind`, `self_gate`, `user` |
| `quote` | **Always kept, whatever the status.** It is the evidence. `truncated` means the app cut the quote short; `partial` means a quote of part of the message (Telegram). |

#### `date_separator`

```json
{"id": "s1", "type": "date_separator", "date": "2026-11-22", "time": null, "weekday": "Sun",
 "label": "Sunday", "resolution": "weekday_label", "anchor_source": "shot-1",
 "provenance": [{"source": "shot-1", "bbox": [470, 212, 610, 262], "anchor_bbox": [470, 212, 610, 262],
                 "cut": "none", "selected": true}],
 "confidence": 0.95, "flags": []}
```

`resolution` is `absolute`, `absolute_year_inferred`, `numeric`, `today`, `yesterday`, `weekday_label`, `user`, or `unresolved` (then `date` is `null`). Date context lines (`from=pill`) are **not** items; the renderer derives them from the messages' `date_source`.

#### `system`

```json
{"id": "s2", "type": "system", "subtype": "disappearing_messages",
 "text": "You turned off disappearing messages.", "dir": "ltr",
 "provenance": [ … ], "confidence": 0.96, "flags": []}
```

`subtype` comes from the plugin's vocabulary, for example `encryption_notice`, `missed_call`, `match_created`, `disappearing_messages` or `unknown`. Plugins may add `x-…` subtypes.

#### `gap`

Taken from another transcript:

```json
{"id": "s7", "type": "gap", "reason": "no_overlap", "confidence": 0.8, "provenance": [],
 "flags": [{"code": "GAP_POSSIBLE", "field": "order"}]}
```

`reason` is `no_overlap`, `alignment_weak` or `append_unanchored`. A gap that a later batch closes is removed; its ID goes into that batch's `removed_items`.

---

## 4. Message IDs

Message IDs are **dotted-decimal ordinals**, written as strings: `"1"`, `"7"`, `"4.1"`, `"0.3"`, `"4.0.1"`.

1. **Array order is authoritative.** The order of `items` is the conversation order. IDs sort in the same order, except around items flagged `ORDER_UNCERTAIN`: if a later append shows that two gap-separated pieces were in the wrong order, the array is reordered and the IDs stay the same.
2. **Comparing IDs.** Compare the dot-separated parts as integers, left to right. A prefix sorts first: `0.1 < 1 < 1.1 < 1.1.1 < 1.2 < 2`. **Never compare IDs as decimals or as strings** (`4.10` comes after `4.9`).
3. **First run.** Messages are numbered `1, 2, 3, …`. `0` alone is never a message ID; it marks system lines in the text view.
4. **Appending at the end.** New messages after the last one continue from the highest leading number: `N+1, N+2, …`.
5. **Inserting before or between.** For *k* new messages between neighbours P and Q, where P is a virtual `0` before the first message:
   1. Use `P.1 … P.k` if `P.k` sorts before Q.
   2. Otherwise (Q extends P, e.g. P = `4`, Q = `4.1`), append `.0` parts until the IDs sort before Q: `4.0.1 … 4.0.k`.

   Each further prepend before an earlier prepend adds one level (`0.1` → `0.0.1`). The optional `renumber` command compacts IDs again (FR-MRG-8).
6. **Stability.** Once given out, an ID never changes and is never reused within an `id_epoch`. The only exception is `chatsocr renumber`, which writes an ID map and increments `id_epoch`.
7. **System item IDs** (`s1`, `s2`, …) are given out in creation order, are never reused, and say nothing about position.

**Example 1: prepending.** After §2.1, a batch scrolled further up is appended. Its screenshots were taken on Wednesday 2026-11-25, so "Saturday" is 2026-11-21:

```
0. System: [date 2026-11-21 Sat "Saturday"]
0.1. 22:10 Bar: are you coming tomorrow?
0.2. 22:15 Yuval: yes!
0. System: [date 2026-11-22 Sun "Sunday"]
1. 21:04 Bar: מה קורה? how was the trip
…
```

**Example 2: filling a gap.** A transcript with a gap:

```
6. 16:33 Yuval: [reply: 5] you looks beautiful
0. System: [gap]
7. 16:36 Bar: תודה!!
```

After an append whose screenshots cover the gap, the `gap` item is removed (listed in `removed_items`):

```
6. 16:33 Yuval: [reply: 5] you looks beautiful
6.1. 16:34 Yuval: seriously
6.2. 16:35 Bar: 😳
7. 16:36 Bar: תודה!!
```

---

## 5. Time semantics

| Rule | Detail |
|---|---|
| No time zone | All dates and times are local wall clock with no time zone. A duration across a daylight-saving change may be off by an hour. |
| Day only | `time: null` with a date means the app showed only a separator date for this message. |
| Relative labels | Resolved against the anchor of the screenshot that contains them (FR-TIM-3). `resolution` and `label` keep the evidence. |
| Weekday names | The most recent such day **strictly before** the anchor date |
| Anchor confidence flows through | A date resolved from a relative label or an inferred year is only as certain as its anchor. With an anchor from file mtime or "now", these dates fall below the threshold and print with `?`. |
| Monotonic order | Separators are non-decreasing in (date, time). Messages are non-decreasing in (date, time). Exceptions are flagged (`DATE_NON_MONOTONIC`, `TIME_NON_MONOTONIC`). Several separators on one day are normal for apps whose separators carry times. |

---

## 6. Flag codes

**Level:**
- `info`: expected degradation.
- `warn`: an uncertain value is in the output.
- `error`: something is missing or failed.

**Shown:** the marker in the text view. *scored* means it is shown only below the threshold; *header* means it appears in the `# warnings:` line; — means JSON only.

**Plugin-settable:** plugins may set only `SIDE_UNCERTAIN`, `MEDIA_KIND_UNCERTAIN`, `ELEMENT_UNRECOGNIZED` and `PARTIAL_MESSAGE`. All other flags come from the engine.

**Transcript, source and batch level** (in `warnings`):

| Code | Level | Shown | Meaning |
|---|---|---|---|
| `DETECTION_AMBIGUOUS` | warn | header | The app or platform was close to an alternative; the top one was used |
| `DETECTION_LOW_CONFIDENCE` | warn | header | No plugin recognized a screenshot; its theme was taken from luminance |
| `DETECTION_MISMATCH` | warn | header | The app or platform the user chose scored below the floor |
| `PARTICIPANTS_INFERRED` | info | header | Names were inferred from the chat header and "Me" |
| `SELF_SIDE_UNKNOWN` | warn | header | No outgoing evidence; self assumed on the right |
| `SELF_SIDE_CONFLICT` | warn | header | `--me` disagrees with the outgoing evidence |
| `ANCHOR_FALLBACK_NOW` | warn | header | No capture time found; the run's `now` was used |
| `ANCHOR_LOW_CONFIDENCE` | warn | header | The anchor came from the file's modification time |
| `ANCHOR_CONFLICT` | warn | header | Metadata and the status-bar clock disagree, or the anchor is earlier than the visible "Today" messages |
| `DATE_ORDER_AMBIGUOUS` | warn | header | Day-first or month-first was guessed |
| `SCREEN_UNPARSEABLE` | error | header | A screenshot could not be read or parsed; it was skipped |
| `PLUGIN_ERROR` | error | header | The plugin failed on a screenshot |
| `OVERLAP_LOW_CONFIDENCE` | warn | — | The alignment between two screenshots is weak |
| `APPEND_ALIGNMENT_WEAK` | warn | header | An appended batch matched the transcript only weakly |
| `APPEND_FORCED` | warn | header | `--force` was used despite a chat or version mismatch |
| `SCHEMA_UPGRADED` | info | — | A transcript with an older minor schema version was upgraded in place |

**Item level** (in `items[].flags`):

| Code | Field | Level | Shown |
|---|---|---|---|
| `OCR_LOW_CONFIDENCE` | text | warn | `[?text s]`, scored |
| `BIDI_AMBIGUOUS` | text | warn | `[?bidi]` |
| `LINEBREAK_UNCERTAIN` | text | info | — (the newline is kept) |
| `EMOJI_UNIDENTIFIED` | text | warn | `[?emoji]` and `�` |
| `EMOJI_LOW_CONFIDENCE` | text | warn | `[?emoji s]`, scored. This includes sequences shown as their component emoji. |
| `SIDE_UNCERTAIN` | sender | warn | `[?sender s]`, scored |
| `PARTIAL_MESSAGE` | text | warn | `[?partial]` |
| `MEDIA_KIND_UNCERTAIN` | media | warn | `[?media s]`, scored |
| `ELEMENT_UNRECOGNIZED` | media | warn | `<unknown>` |
| `TIME_UNREADABLE` | time | warn | `[?time]` |
| `TIME_LOW_CONFIDENCE` | time | warn | `HH:MM?`, scored |
| `TIME_CORRECTED` | time | warn | `HH:MM?` (the raw value is kept) |
| `TIME_NON_MONOTONIC` | time | warn | `HH:MM?` |
| `DATE_UNKNOWN` | date | warn | `[?date]` |
| `DATE_LOW_CONFIDENCE` | date | warn | `[date …?]`, scored. Covers implausible weekday results and weak anchors. |
| `DATE_LABEL_UNPARSED` | date | warn | `[date ? "raw"]` |
| `DATE_NON_MONOTONIC` | date | warn | `[date …?]` |
| `YEAR_INFERRED` | date | info | the raw label is shown |
| `REPLY_LOW_CONFIDENCE` | reply | warn | `[reply: N?]` |
| `REPLY_AMBIGUOUS` | reply | warn | `[reply: ~N\|M …]` |
| `REPLY_UNRESOLVED` | reply | info | `[reply: ? …]` |
| `DUPLICATE_UNCERTAIN` | order | warn | `[?dup]` |
| `ORDER_UNCERTAIN` | order | warn | `[?order]`, on the gap item and the first item of each piece after it |
| `GAP_POSSIBLE` | order | warn | `0. System: [gap]` |

New codes are added only in a minor schema version and must be added to this table. A reader must treat an unknown code as `warn`.

---

## 7. Versioning

| Artifact | Version | Rules |
|---|---|---|
| JSON | `schema_version` (semver) | **MAJOR:** fields removed, renamed or changed in meaning; ID rules changed. **MINOR:** new optional fields, flag codes, media kinds or system subtypes. **PATCH:** documentation only. |
| Text view | `chatsocr-text/<major>` in the first line | A new major version only if a reader following this grammar would misread the new output |

- **Appending across versions** (FR-OUT-10):
  - The same major version is required.
  - A transcript written by a newer minor version is refused (exit 6) unless `--force` is given.
  - A transcript from an older minor version is upgraded in place (`SCHEMA_UPGRADED`).
  - Unknown fields are kept.
- **Phase-2 additions are minor versions:**
  - reactions (`[react: ❤️ Bar]`)
  - markers (`[edited]`, `[forwarded]`)
  - `<deleted>`
  - group senders

  The new tags fall under `othertag`, which every v1 reader already skips (§2.2 rule 6).
