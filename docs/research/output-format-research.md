# ChatsOCR research: output format and time semantics

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

## Key takeaways for the requirements/design

1. **Keep two layers.** JSON is the lossless source of truth. The text view is a deterministic, token-lean rendering built from it. I measured the same 6-message chat with the o200k_base tokenizer (OpenAI's, used only as a proxy; Claude's tokenizer is not public):
   - text view: 105 tokens
   - compact JSON: 368 tokens
   - pretty JSON: 603 tokens
   - Write JSON as UTF-8 (`ensure_ascii=False`). ASCII-escaping the Hebrew and emoji added about 38% more tokens.
2. **Text-view line shape:** `N. [HH:MM[?] ]Sender: {prefix-tags }body`.
   - All machine annotations (`[reply: 3]`, `<image media/3.png>`, `[?text 0.58]`, later `[edited]` and `[react: ❤️ Bar]`) are **prefix tags placed before the body**. The body is literal text.
   - Escaping is needed in one place only: a body that starts with `[`, `<` or `\` gets a leading `\`.
   - Continuation lines of a multi-line message are indented by exactly 2 spaces.
3. **Separators are their own items.** `0. System: [date 2026-11-24 Tue "Today"]` is a date separator (minute precision when the app shows a time, e.g. `[date 2026-11-24T08:33 Tue]`). Showing the weekday saves the LLM from computing it. The raw label is shown only when the date was inferred.
4. **Time model.** Store naive local wall-clock ISO 8601 (`2026-11-24T08:33`), never `Z` or an offset. Next to it store an explicit `precision` (`none|day|minute|second`), a `source` (`bubble|separator|relative_label|inferred|interpolated`) and the `raw` OCR string. The text view always shows 24h `HH:MM`.
5. **Time anchor per screenshot.** Relative labels ("Today", "אתמול", "Sunday") are resolved against the capture time of the screenshot they appear in. Anchor precedence: EXIF → filename timestamp → status-bar clock as a cross-check → mtime → now. Record which one was used.
6. **Stable IDs** are immutable dotted-decimal ordinals.
   - Messages get `1…N` on the first run, and appends continue at `N+1`.
   - Older or gap messages found later get `P.1, P.2…`, which sort between P and its successor. This is a human-readable form of fractional indexing.
   - Content hashes are only used as matching *fingerprints*, never as IDs.
7. **De-duplication across batches** is a sequence alignment of the new batch against the existing transcript, not a set of independent hash lookups. Identical short messages ("ok", "😂") recur too often for hashing alone.
8. **DD/MM vs MM/DD:** prefer the plugin's locale knowledge or month-name formats. Otherwise use three heuristics: a value >12, a decreasing sequence, and change frequency (the same three that whatsapp-chat-parser uses). Store the decision at transcript level so append runs stay consistent.
9. **Bidi.** Store text in logical order, NFC-normalized, with a per-message `dir` (`ltr|rtl|auto`, the W3C string-meta values).
   - Do not inject bidi controls into the JSON or the default text view. Each FSI/PDI costs 2 tokens and breaks exact matching.
   - Offer an optional `--isolate` flag that wraps strings in FSI…PDI for terminal display only.
10. **Unrecognized emoji or glyphs** render as U+FFFD `�` inside the text: one character, no escaping needed. JSON keeps the bbox so the glyph can be re-identified later.

---

## 1. Prior art: verified facts and what to borrow

**WhatsApp "Export chat" .txt (verified from parser source code, not an official spec).**
- Android lines: `date, time - Author: text`. iOS lines: `[date, time] Author: text`.
- One shared regex covers both, with optional brackets: `^(?:\u200E|\u200F)*\[?(\d{1,4}[-/.]\s?\d{1,4}[-/.]\s?\d{1,4})[,.]?\s\D*?(\d{1,2}[.:]\d{1,2}(?:[.:]\d{1,2})?)(?:\s([ap]\.?\s?m\.?))?\]?(?:\s-|:)?\s` [2].
- Lines with no `Author:` are system messages (author `null`). Lines that match neither pattern are continuation lines of the previous message [2].
- Attachments appear as `<attached: 00000042-PHOTO-2020-06-07-15-13-20.jpg>` (iOS) or `IMG-…jpg (file attached)` (Android). Without media they appear as `<Media omitted>` [2][3].
- iOS exports put LRM/RLM marks before lines, and parsers must strip them [2]. This shows the cost of injecting bidi controls.
- Dates are locale-dependent (`29.01.18`, `d/m/yy`, 12h or 24h) [3].
- The Pustur parser infers day-first vs month-first with `checkAbove12` → `checkDecreasing` (within one year months only increase) → `changeFrequencyAnalysis` (the component that changes more often is the day). Its fallback is day-first [2].
- Borrow: one line per message, the `Author: text` shape, and the date-order heuristics. Avoid: no IDs, no reply links, ambiguous continuation lines.

**Telegram Desktop JSON (verified) [1].**
- Messages have `id` (int), `type` (`message`|`service`), `date` (ISO 8601), `date_unixtime`, `edited`, `from`, `from_id` (`user123…`), `reply_to_message_id`, `forwarded_from` and `via_bot`.
- `text` is a string or an array of entities. `text_entities` types include `plain`, `bold`, `link`, `text_link`, `mention`, `custom_emoji`, `spoiler`, `blockquote` and others.
- Media is stored in `photo` / `file` / `thumbnail` (relative paths), `media_type` ∈ {`sticker`, `video_message`, `voice_message`, `animation`, `video_file`, `audio_file`}, plus `duration_seconds`, `width`, `height`, `mime_type`.
- Structured content uses `location_information`, `contact_information`, `poll`.
- Service messages use `action` (`pin_message`, `phone_call`, `invite_members`, …) and `actor`.
- Borrow: integer IDs plus a `reply_to` ID, `message` vs `service` typing, relative media paths, and a media-kind enum.

**Matrix (verified) [6].**
- Replies use `content["m.relates_to"]["m.in_reply_to"]["event_id"]`.
- Fallback quoting in `body` was **removed from the spec in v1.13** (PR #1994, MSC2781). Clients should still strip legacy fallbacks.
- Lesson: **the reply link is structured data, and quoted text is evidence, not content.** Store the quoted snippet separately from `text`.

**ActivityStreams 2.0 (verified) [7].** Relevant vocabulary: `Note`, `attributedTo`, `published` (xsd:dateTime), `inReplyTo`, `attachment` (`Image`, `Audio`, `Video`, `Document`, `Place`), `duration` as xsd:duration (`PT5S`), `Tombstone` for deleted objects. Borrow the vocabulary names and ISO durations in JSON. Full JSON-LD is unnecessary.

**Signal / iMessage exporters (verified, shallow).**
- carderne/signal-export writes Markdown lines such as `[2019-05-29, 15:04] Me: How is everyone?`, with images as `![name](path)` and other attachments as `[name](path)`. Its `--old DIR` option merges with a previous export without overwriting [4].
- imessage-exporter's TXT output puts multi-part messages on new lines, shows attachments as file paths, and annotates stickers, tapbacks and edits [5].
- Nothing in either is better than the Telegram model. Their main lesson is that media files sit next to the transcript with relative links.

## 2. Text view grammar (recommendation)

Design rules:
- **One header line per item.** Numbering is the stable ID.
- **Prefix tags carry metadata.** Tags are ASCII `[...]` / `<...>`; quoted strings inside tags use JSON escaping (`\"`, `\\`, `\n`).
- **Unknowns.** A trailing `?` marks a low-confidence value. `[?field score]` says which field is uncertain and points the agent to the JSON warnings.
- **Media paths** are relative to the transcript file.

```ebnf
view        = { comment , NL } , { item } ;
comment     = "#" , { char - NL } ;                       (* metadata, ignorable *)
item        = ( system | message ) , NL , { "  " , { char - NL } , NL } ;  (* 2-space continuation *)
system      = "0. System: " , ( date-tag | sys-text ) ;
date-tag    = "[date " , date , [ "T" , hhmm ] , " " , wkday , [ " " , qstr ] , [ "?" ] , "]" ;
message     = id , ". " , [ hhmm , [ "?" ] , " " ] , sender , ": " , { tag , " " } , body ;
id          = nzint , { "." , int } ;                     (* 7 | 4.1 | 4.1.2 ; never 0 *)
hhmm        = digit , digit , ":" , digit , digit ;      (* 24h local wall clock *)
sender      = name ;                                     (* participant label; must not contain ": " or start with hh:mm *)
tag         = reply | media | flag | marker ;
reply       = "[reply: " , ( id , [ "?" ] | "?" ) , [ " " , name ] , [ " " , media ] , [ " " , qstr ] , "]" ;
media       = "<" , kind , { " " , attr } , ">" ;
kind        = "image" | "video" | "gif" | "sticker" | "voice" | "audio" | "document" | "link"
            | "location" | "contact" | "poll" | "call" | "deleted" | "view-once" | "unknown" ;
attr        = duration | path | qstr | size ;            (* duration "0:42"; path "media/3.png" *)
flag        = "[?" , field , { "," , field } , [ " " , score ] , "]" ;   (* field: text|time|sender|reply|media|order|dup *)
marker      = "[edited]" | "[forwarded]" | "[react: " , { char - "]" } , "]" ;   (* phase 2 *)
body        = [ "\" ] , { char - NL } ;                  (* "\" emitted iff text starts with [ < or \ *)
qstr        = '"' , { char-escaped } , '"' ;
```

A parser regex for a message line:
`^(?P<id>[1-9]\d*(?:\.\d+)*)\. (?:(?P<t>\d\d:\d\d)(?P<tq>\?)? )?(?P<who>.+?): (?P<rest>.*)$`
After that, consume tags with
`^(\[(?:[^\]"]|"(?:[^"\\]|\\.)*")*\]|<(?:[^>"]|"(?:[^"\\]|\\.)*")*>) `
and drop one leading `\` from what remains.

Example (when a chat has no per-message times, the time field is omitted, which gives exactly the format the user imagined):

```
# chatsocr-text/1 app=whatsapp/android/dark detect=0.97 tz=local-naive
# participants: Bar=left Yuval=right(me)
0. System: [date 2026-11-22 Sun "Sunday"]
1. 21:04 Bar: מה קורה? how was the trip
2. 21:10 Yuval: <voice 0:42>
0. System: [date 2026-11-24 Tue "Today"]
3. 08:40 Bar: <image media/3.png> I'm in the gym 💪
4. 16:35 Yuval: [reply: 3] you looks beautiful
4.1. 16:36? Bar: [?text 0.58] תודה!! נתראה ב-8
5. 16:40 Yuval: [reply: ? Bar "see you at 8"] <link "https://maps.app.goo.gl/x"> here
6. 16:41 Bar: \[joke] two lines
  and a second line
```

Notes:
- An unresolved reply shows the quoted snippet plus the sender. When resolved with high confidence it is just `[reply: 3]`. An uncertain match is `[reply: 3? "how are"]`.
- A reply to an image quote is `[reply: 3 <image>]`.
- Media order is placeholders first, then the caption, which matches how bubbles are laid out.
- Link previews: the URL stays in the body if it was typed there. The preview card is `<link "url" "title">`.
- A sticker is `<sticker 😂>` when its associated emoji is known.

## 3. JSON schema skeleton

The schema follows JSON Schema draft 2020-12 [17] and ships inside the package (`$id: "urn:chatsocr:schema:transcript:1"`).
- `schema_version` uses semver. MAJOR means a field was removed, renamed or changed meaning. MINOR means a new optional field. PATCH means docs only.
- Readers must ignore unknown fields. Reserve `additionalProperties:false` for a `--strict` validation mode.
- The text view carries its own version in its header (`chatsocr-text/1`).

```json
{
  "schema_version": "1.0.0",
  "generator": {"name": "chatsocr", "version": "0.1.0"},
  "chat": {
    "app": {"id": "whatsapp", "platform": "android", "theme": "dark",
            "plugin": {"id": "whatsapp.android", "version": "0.3.0"},
            "detection": {"mode": "auto", "confidence": 0.97,
                          "alternatives": [{"id": "whatsapp.ios", "confidence": 0.02}]}},
    "kind": "direct",
    "participants": [{"id": "p1", "name": "Bar", "side": "left", "is_self": false},
                     {"id": "p2", "name": "Yuval", "side": "right", "is_self": true}],
    "time": {"zone": null, "date_order": {"value": "DMY", "source": "month_names", "confidence": 1.0}}
  },
  "sources": [{"id": "shot-1", "batch": "b1", "path": "in/Screenshot_20261124-164210.png",
               "sha256": "9f2c…", "size_px": [1080, 2400],
               "captured_at": {"value": "2026-11-24T16:42:10", "source": "filename",
                               "checks": {"status_bar": "16:42", "mtime": "2026-11-24T16:42:11"}}}],
  "batches": [{"id": "b1", "processed_at": "2026-11-24T17:00:03", "sources": ["shot-1"]}],
  "items": [
    {"id": "sys-2", "type": "date_separator",
     "date": {"value": "2026-11-24", "precision": "day", "source": "relative_label",
              "raw": "Today", "anchor": "shot-1"},
     "provenance": [{"source": "shot-1", "bbox": [470, 212, 610, 262]}]},
    {"id": "3", "type": "message", "sender": "p1", "side": "left",
     "time": {"value": "2026-11-24T08:40", "precision": "minute", "source": "bubble",
              "raw": "8:40", "confidence": 0.99},
     "text": "I'm in the gym 💪", "dir": "ltr",
     "spans": [{"type": "emoji", "start": 15, "end": 16, "confidence": 0.93, "bbox": [610, 1120, 650, 1160]}],
     "media": [{"id": "3/m1", "kind": "image", "file": "media/3.png", "source": "shot-1",
                "bbox": [40, 300, 700, 1100], "phash": "c3a1e0f4…"}],
     "reply_to": null,
     "provenance": [{"source": "shot-1", "bbox": [30, 290, 720, 1180]}],
     "fingerprint": "L|image|i'm in the gym 💪|08:40",
     "confidence": 0.95, "warnings": []},
    {"id": "4", "type": "message", "sender": "p2", "side": "right",
     "time": {"value": "2026-11-24T16:35", "precision": "minute", "source": "bubble", "raw": "16:35"},
     "text": "you looks beautiful", "dir": "ltr",
     "reply_to": {"id": "3", "quote": {"sender": "p1", "text": "I'm in the gym", "media_kind": "image",
                  "bbox": [700, 1400, 1040, 1480]},
                  "match": {"method": "quote_text+thumb_phash", "score": 0.91}},
     "provenance": [{"source": "shot-1", "bbox": [610, 1390, 1050, 1590]}],
     "confidence": 0.97, "warnings": []}
  ],
  "warnings": [{"code": "TIME_NON_MONOTONIC", "item": "4.1", "detail": "16:36 read as 18:36; corrected"}]
}
```

Conventions:
- `bbox` is `[x0, y0, x1, y1]` in source-pixel coordinates.
- `provenance` is a list, because overlapping screenshots observe the same message more than once.
- `spans` use code-point offsets (Python `str` indices) and are present only for emoji or low-confidence runs. This avoids duplicating `text`.
- Item `type` values: `message`, `date_separator`, `system` (encryption notice, Hinge "You matched", calls), and later `group_event`. UI artifacts ("N unread messages", the sticky floating date pill, typing indicators) are **not** items. A floating date pill can still be used as time evidence (believed; see risks).

## 4. Stable IDs and merging appended batches

| Strategy | Stable? | LLM-readable | Handles older or gap inserts |
|---|---|---|---|
| Renumber 1..N every run | no | yes | yes |
| Content-hash ID | breaks when OCR improves; collides on repeated "ok" | no | yes |
| Fractional keys (`a0`, `a0V`; rocicorp `generateKeyBetween`) [12] | yes | poor | yes, always |
| **Dotted decimal ordinals** (recommended) | yes | yes | yes, always |

How dotted ordinals work:
- They compare component-wise: `0.1 < 1 < 1.1 < 1.1.1 < 1.2 < 2`. A key can always be found between any two keys, which is the same property fractional indexing has.
- New messages after the end get `max+1`.
- k messages found between P and Q get `P.1…P.k`.
- Messages older than the first one get `0.1, 0.2…`. That is legal, because `0` alone is reserved for System lines. A second prepend before `0.1` gets `0.0.1`.
- System items get separate IDs (`sys-n`) and always render as `0.`.
- An explicit `chatsocr renumber` command may compact IDs, but it must write an `id_map` and bump a transcript `epoch` field so agents know old references are void.

Merge algorithm (recommended):
1. Within a batch, order the screenshots by overlap. That is the stitching module's job.
2. Build candidate items with fingerprints: side, normalized text (NFC, casefold, whitespace collapsed), time, media kind and pHash, reply quote.
3. Run a semi-global alignment (Needleman–Wunsch/Smith–Waterman style) of the candidate sequence against the existing item sequence. Use a fuzzy similarity: side must match; add text ratio (rapidfuzz), time equality, media pHash distance and separator anchors.
4. Matched items keep their ID. Add the new observation to `provenance`, and optionally improve the text by confidence voting, recording a `revisions` entry.
5. Unmatched runs become an append, a prepend or a dotted insertion.
6. If an alignment score falls below threshold, or a batch fits in two places, emit a warning and choose the best candidate. Never block in non-interactive mode.

## 5. Time semantics

**Representation.** `value` is an ISO 8601 local string truncated to `precision`, and `precision` is also stored explicitly.
- Python 3.13's `datetime.fromisoformat("2026-11-24T08:33")` parses minute precision (verified locally).
- Use `zone: null` for unknown timezones. Never append `Z`.
- RFC 9557 `[Zone]` suffixes [10] are a future option if a timezone is ever known.
- EDTF qualifiers (`2026-11-24?`, `XXXX-11-24`) [11] are expressive but most tooling can't parse them. Use `inferred: ["year"]` and `confidence` fields instead.
- Optional `bounds: {not_before, not_after}` covers messages known only by day, or between two timed neighbours.

**Anchor selection** (each candidate recorded under `sources[].captured_at`):
1. EXIF `DateTimeOriginal` plus `OffsetTimeOriginal`. iOS screenshots reportedly carry `DateTimeOriginal` and `UserComment = Screenshot` [14] (anecdotal).
2. Filename. The AOSP-style pattern is `Screenshot_20201215-090626.png` or `Screenshot_20201215-090626_Settings.png` [13]. Samsung appends the foreground app name [13], which is also a free hint for app detection.
3. The status-bar clock OCR'd from the screenshot. It gives only the time of day, but it checks the other candidates and caps "Today"'s latest message.
4. File mtime, which is unreliable after copying or messaging the file.
5. Now (local time).

Warn when the candidates disagree by more than a few minutes.

**Label resolution.** A = the anchor date of the screenshot that contains the label.
- "Today"/"היום" → A. "Yesterday"/"אתמול" → A−1.
- A weekday label means the most recent D < A with that weekday. Accepted forms:
  - English full or abbreviated names, plus "Last Saturday" style (seen in Discord [15]).
  - Hebrew `יום ראשון…יום שישי`, `יום שבת`/`שבת`, and the abbreviations `יום א׳…יום ו׳` (CLDR data, verified via Babel 2.18).
- If D comes out as A−7 or A−1, flag it, because apps usually say "Yesterday", or switch to a full date after about a week (generic chat-SDK convention [15]; WhatsApp's exact cutoff is unverified).
- A month-day without a year (e.g. "November 17", `17 בנובמבר`) gets the latest year with date ≤ A. Then walk the separators from newest to oldest and enforce strict monotonicity, decrementing the year when needed.

**Clock parsing.**
- Accept `4:33 PM`, `pm`, `p.m.`, and U+202F narrow no-break space before AM/PM. CLDR en-US short time renders `4:33\u202fPM` (verified via Babel).
- Accept Hebrew `לפנה״צ`/`אחה״צ` as well as `AM/PM`. Current CLDR `he` uses `AM/PM` (verified); the Hebrew forms appear in older ICU and Android builds (believed).
- 12h times shown without AM/PM: infer from separators-with-time and monotonicity, and flag the result.

**Monotonicity checks.** Down the transcript, (date, time) should be non-decreasing.
- On a violation, try OCR alternates for confusable digits (1/7, 3/8, 5/6, 0/8) and pick the one that restores order. Keep `raw` and add a warning; never correct silently.
- Real exceptions exist (a midnight crossing with no visible separator, messages sent offline). So a violation is a warning, never an error.

**Apps with only separators (or separators that carry times).** Messages get `precision: "day"` with `source: "separator"`, or minute precision with `source: "interpolated"` and bounds when the separator has a time. The text view then omits `HH:MM`, so it matches the user's own example exactly.

## 6. Bidi

- Store `text` in logical order. Some OCR engines emit visual order for RTL text, so the engine must normalize before writing JSON (this is the OCR researcher's concern).
- Strip stray LRM/RLM. Keep ZWJ (U+200D) and VS16 (U+FE0F) because emoji sequences need them.
- Per W3C string-meta [8]:
  - Use the field `dir` with values `ltr|rtl|auto`, plus a BCP 47 `lang`. Omit `dir` when truly unknown.
  - Specifications "MUST NOT require the production or use of paired bidi controls."
  - Inserting characters "changes the identity of the string."
  - First-strong detection is a fallback only. Here the bubble's paragraph alignment is better evidence.
- Code points for the optional display mode: LRI U+2066, RLI U+2067, FSI U+2068, PDI U+2069 (UAX #9 [9]; names verified with Python `unicodedata`). Measured with o200k: FSI/PDI cost 2 tokens each, and LRM costs 1.
- Recommendation: no controls by default. LLMs read logical order natively. Offer `--isolate` to wrap the sender and each body in FSI…PDI for human terminal viewing only.

## Risks & open questions

1. **Dotted IDs** get ugly after repeated gap fills. Is the user fine with `4.1.` style lines, or would they prefer renumbering plus an `id_map`? The answer affects how agents keep long-lived references.
2. **App-specific separator vocabularies are unverified:** the weekday cutoff, how year-omission is decided, Hebrew UI strings per app and platform, and how Hinge stamps times. Each plugin must declare its label vocabulary and date format, and needs golden screenshots.
3. **Sticky date pills** (WhatsApp/Telegram floating headers while scrolling) can be misread as in-flow separators and create duplicate System lines. Plugins must classify them (believed).
4. **Anchor reliability:** screenshots forwarded through messengers lose EXIF and get a new mtime. Should the user be able to pass `--anchor 2026-11-24T17:00`? Recommended: yes.
5. **Sender labels** containing `: ` or starting with `HH:MM` break the line regex. Require sanitized participant labels in the config, with the original name kept in JSON.
6. **Token figures** come from OpenAI's o200k_base, not Claude's tokenizer. The ratios (text roughly 3.5× cheaper than compact JSON) should hold roughly, not exactly.
7. **The uncertainty display threshold** (e.g. show `?` below 0.8) needs calibration per OCR engine. How verbose should `[?text 0.58]` be by default?
8. **DST and timezone travel** inside a chat cannot be detected from screenshots. Naive local time is the honest representation; document that durations computed across a DST change may be off by an hour.

## Sources

[1] Telegram data export schema: https://core.telegram.org/import-export
[2] whatsapp-chat-parser source: https://github.com/Pustur/whatsapp-chat-parser (`src/parser.ts`, `src/date.ts`)
[3] WhatsR `parse_ios` defaults: https://search.r-project.org/CRAN/refmans/WhatsR/html/parse_ios.html
[4] signal-export: https://github.com/carderne/signal-export
[5] imessage-exporter features: https://github.com/ReagentX/imessage-exporter/blob/develop/docs/features.md
[6] Matrix rich replies: https://spec.matrix.org/latest/client-server-api/ (module source https://github.com/matrix-org/matrix-spec/blob/main/content/client-server-api/modules/rich_replies.md); fallback removal: matrix-spec PR #1994 / MSC2781
[7] ActivityStreams 2.0 vocabulary: https://www.w3.org/TR/activitystreams-vocabulary/
[8] W3C Strings on the Web, Language and Direction Metadata: https://www.w3.org/TR/string-meta/
[9] UAX #9: https://www.unicode.org/reports/tr9/ ; W3C bidi controls: https://www.w3.org/International/questions/qa-bidi-unicode-controls
[10] RFC 9557 (IXDTF): https://datatracker.ietf.org/doc/html/rfc9557
[11] EDTF: https://www.loc.gov/standards/datetime/
[12] fractional-indexing: https://github.com/rocicorp/fractional-indexing
[13] AOSP-style screenshot names: https://gitlab.e.foundation/e/os/android_frameworks_base/-/commit/299dc3ae8b378c52fd49adb5fe26219a0e7e8c1b ; Samsung app-name suffix: https://www.androidpolice.com/2017/12/19/neat-samsung-adding-context-info-screenshot-names-android-oreo/
[14] iOS screenshot EXIF anecdote: https://omz-software.com/forum/archive/4453_new-photos-module-read-metadata.html
[15] Date-separator conventions: https://getstream.io/chat/docs/sdk/react-native/ui-cookbooks/message-list/date-separators/ ; https://support.discord.com/hc/pt-br/community/posts/360032400911-Relative-Message-Dates-misleading
[16] CLDR data checked locally with Babel 2.18.0 (Hebrew weekday names, `he` AM/PM, en-US U+202F); token counts measured with tiktoken o200k_base.
[17] JSON Schema 2020-12: https://json-schema.org/draft/2020-12/schema
