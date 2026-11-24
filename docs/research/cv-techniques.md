# Classic-CV research for ChatsOCR (no LLM at runtime)

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

Legend: **[V]** = verified, with a source or a local test run in this container on 2026-11-24. **[B]** = believed but unverified. **[R]** = recommendation.

## Key takeaways for the requirements/design

1. **Normalize everything to an "em" unit.** Measure the median body-text line height per screenshot and express every plugin threshold as a multiple of it (gaps, corner radius, color tolerances in ΔE). Don't use raw pixels. Phone screenshots range from 1080 to 1440 px wide, desktop shots run at 1x, 1.25x or 2x, and users change font size and zoom. [R]
2. **Bubble segmentation should be color-model-first, with a shape fallback.** The pipeline: estimate the background color or pattern, find the flat bubble colors, close the mask morphologically, take connected components, then check that each component fills most of its rounded-rectangle box. Learn colors per screenshot or calibrate them per theme. Don't hard-code them: WhatsApp changed its dark palette in 2024 ([TechCrunch](https://techcrunch.com/2024/05/09/whatsapp-streamlining-navigation-darker-dark-mode)), and Telegram allows custom or gradient themes. [R]
3. **Multi-screenshot input: parse each shot, then merge the message lists (hybrid).** Use pixel NCC on the masked chat viewport only to estimate vertical offsets between shots. Do the final merge by geometry plus content, with sequence alignment as the fallback. Don't stitch first: overlays, ticks and reactions change between shots, and append mode needs message-level merging anyway. Never assume that capture time order equals content order, because users often scroll *up* while taking screenshots. [R]
4. **Emoji identification works only with templates from the same vendor.** Local test: nearest-neighbor matching on Lab color at 24×24 identified 3,761 Noto emoji sequences with **99.1% top-1** at 40 px, exact crop, and **96.5%** with ±1 px bbox jitter. Matching *across vendors* (Twemoji queries against Noto templates) reached only **≈35% top-1**. [V-local] The engine therefore needs a vendor-keyed template store per app×platform. Templates should be built at runtime from fonts on the user's machine, or from a **calibration chat** the user screenshots, because the Apple, WhatsApp and Samsung sets cannot be redistributed. [R]
5. **License situation for emoji sets.** Noto fonts are OFL-1.1, and Noto tools and most images are Apache-2.0 ([repo](https://github.com/googlefonts/noto-emoji)). Twemoji graphics are CC-BY-4.0 and its code is MIT; v17.0 covers Emoji 17.0 ([repo](https://github.com/jdecked/twemoji)). OpenMoji graphics are CC BY-SA 4.0 and its code is LGPL-3 ([repo](https://github.com/hfg-gmuend/openmoji)). [V] Apple, Samsung and WhatsApp sets are proprietary. [B] Only Noto can be bundled freely, and it matches only stock-Android apps that use system emoji, such as Hinge on a Pixel. [R]
6. **Reply resolution.** Normalize the quoted text and strip the trailing "…". Score candidates with `rapidfuzz` using a prefix-anchored alignment. Accept a match only if the best score is ≥85 *and* leads the runner-up by ≥8. Otherwise store `reply_to: null` plus `candidates[]` and an `ambiguous` flag. Match reply thumbnails by center-cropping candidate images and combining pHash, a color histogram and SSIM. [R]
7. **Time anchors follow a precedence chain, and the output records which source was used.** Stock Android (AOSP) writes `DateTimeOriginal` **with `OffsetTimeOriginal`** and names files `Screenshot_YYYYMMDD-HHMMSS.png` ([AOSP ImageExporter](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/src/com/android/systemui/screenshot/ImageExporter.java)). [V] Samsung appends the **app label**, e.g. `Screenshot_20230418_221813_WhatsApp_1.jpg`, which doubles as an app-detection hint. [V-examples] iOS PNGs often carry no capture date, and AirDrop resets file dates. [B] The status-bar clock (time of day only) works as a cross-check. [R]
8. **Library stack.** numpy, opencv-python-headless, Pillow, scikit-image, scipy, ImageHash, RapidFuzz and python-bidi, all with permissive licenses except python-bidi (LGPL). Note that **OpenCV 5.1's main wheel has ORB and SIFT but no AKAZE** [V-local], so don't design around AKAZE. [R]
9. **Plugin contract.** The engine should own the generic steps: chrome masks, the background and color model, components, em scale, OCR, emoji matching, merge and time anchoring. Plugins supply parameters plus a few hooks (classify a region, split a bubble, overlay masks), so writing a plugin is mostly configuration plus a few small functions that an AI agent can produce from example screenshots. [R]

---

## 1. Preprocessing and scale normalization

- **Viewport extraction [R]:**
  - *Batch of shots:* rows that are pixel-identical (ΔE < 2) at the same y in ≥80% of the shots are fixed chrome: status bar, app header, input bar.
  - *Single shot:* the plugin supplies dp/em hints, and the engine finds the first and last full-width horizontal edge or color step.
  - Mask the keyboard if one is present.
- **Scale [R]:** `em` = median text line height, measured by horizontal projection inside the largest bubbles, or from OCR line boxes. Use `dp = width / logical_width` only as an initial guess (typically 360–412 dp on Android and 375–440 pt on iPhone [B]). Desktop and web shots must rely on em alone.
- **JPEG artifacts:** Samsung saves JPEG, and screenshots forwarded through WhatsApp are recompressed. Compare colors in Lab with a ΔE tolerance of about 6–10, never by exact equality. [R]

## 2. Bubble segmentation

**Algorithm sketch [R]:**
1. **Background model.**
   - *Solid backgrounds:* take the mode color of a low-pass-filtered viewport.
   - *Patterned wallpaper* (WhatsApp's low-contrast doodles): median-filter or `cv2.pyrMeanShiftFiltering`, then build a "flatness map" from local standard deviation in an em-sized window. Bubbles are *large* flat regions whose color differs from the background mode. Doodle lines are thin and don't survive an opening with a kernel of about 0.3 em.
   - *Photo wallpapers:* flag low confidence and fall back to the shape-based path in step 5.
2. **Color palette.** Run k-means (`cv2.kmeans`, k = 6–10) on downsampled viewport pixels. Clusters with large flat area are the candidate bubble colors. The plugin maps them to incoming/outgoing/system, or the engine infers the mapping from x-anchoring. Cache the result per theme.
3. **Mask per bubble color.** `|Lab - c| < tol`, then morphological close with a kernel of about 0.6 em to fill the text holes, then `cv2.connectedComponentsWithStats`.
4. **Filters:**
   - width ≥ 1.5 em, height ≥ 1.2 em;
   - fill ratio of the bounding box ≥ 0.85 (rounded corners and the tail remove a little);
   - corner check: the four corner pixels at a distance of r ≈ 0.3–0.6 em are background colored;
   - `cv2.approxPolyDP` on the contour gives a near-rectangle.

   Keep the WhatsApp tail (a small triangle on the first bubble of a run) by allowing a protrusion of up to 0.5 em.
5. **Media without a bubble color** (Telegram images and stickers, and any image):
   - look for non-background components with high texture or color entropy and near-rectangular rounded outlines;
   - fall back to gradient magnitude → closed contours → rectangle fit;
   - WhatsApp images sit inside a thin frame of bubble color (≈3 dp) [B], which the step-3 mask catches as a "ring" component.

   Stickers and GIFs have transparent outlines, so take the union of non-background pixels and dilate.
6. **Emoji-only ("jumbo") messages** have no bubble. These are high-saturation, non-background components roughly 2–4 em tall with a small time chip beside or below them. Treat them as message candidates and assign the side as in step 7.
7. **Side.**
   - *Primary cue:* anchoring. Incoming bubbles start within about 1.5 em of the left margin; outgoing bubbles end within about 1.5 em of the right margin.
   - *Secondary cue:* bubble color.
   - Desktop and wide layouts may left-align everything [B], so plugins must be able to override this.
   - In an RTL (Hebrew) UI the sides are mirrored, so let the plugin declare the side mapping or detect it from the header layout.
8. **Reading order:** sort by top y, breaking ties by x. Mark bubbles that touch the viewport's top or bottom edge as `partial`.

## 3. Date-separator chips and system messages

- **Detection [R]:**
  - the component's horizontal center is within 3% of the viewport center;
  - left and right margins are equal within 1 em;
  - width is under 70% of the viewport;
  - it has a distinct chip color, or no chip but centered small text (Telegram desktop service messages).
- **Classification:** OCR the chip, then attempt date parsing in Hebrew and English. "Today", "Yesterday", "היום", "אתמול" and weekday names resolve against the screenshot anchor date. Anything that isn't a date becomes a `System` message, e.g. the encryption notice or "X changed the subject".
- **Sticky floating date chip:** WhatsApp and Telegram pin a date chip at the top of the viewport while scrolling. [B]
  - Detect it as a centered chip in the top ~3 em of the viewport.
  - If it duplicates a chip already found in the content, drop it.
  - Otherwise use it only as a *date hint* for the messages above the first in-content separator, and mask it out of the overlap computation.

## 4. Reply-quote block, time and tick regions inside a bubble

- **Quote block [R]:** a sub-rectangle inside the bubble that has
  - a fill that differs slightly from the bubble (ΔE ≈ 3–15) — WhatsApp uses a tinted rounded box [B], and newer Telegram versions tint the quote background [B];
  - a narrow (≈0.2 em), saturated vertical bar on the leading edge (on the right in RTL UIs);
  - a colored sender-name line at its top.

  Detect the bar with a column-wise saturation profile, then grow the region over the tinted fill and split the bubble just below it (horizontal projection gap). An optional square thumbnail sits at the trailing side of the quote block: a textured square of about 2.5–3 em.
- **Time and ticks [R]:** the trailing-bottom corner of the bubble holds a cluster of small glyphs with an x-height of about 0.7× the body text, in a gray or muted color. It may share the last text line or sit on a line of its own. Detect it with a bottom-strip connected-component analysis and the color filter, then OCR it with a whitelist of `0-9 : . AM PM` and the Hebrew `לפנה"צ אחה"צ` [B].
  - Ticks are check-mark components of one or two strokes. They are blue when read; WhatsApp uses roughly #53BDEB [B, measure it].
  - Plugins provide the exact geometry. The engine provides `find_small_glyph_cluster(region, corner, em)`.

## 5. Reply resolution

**Text quotes [R; behavior V-local]:**
- **Normalize:** NFKC, strip bidi controls (U+200E/F, U+202A–202E, U+2066–2069), collapse whitespace, map emoji to a single token or a CLDR name, strip the trailing `…` or `...`, and fold Hebrew final letters (ך→כ etc.) to absorb OCR confusions. Set `truncated = True` if an ellipsis was present or the quote filled its maximum line count (the plugin declares the line limit).
- **Score each earlier candidate:**
  - if truncated, use `fuzz.ratio(q, cand[:len(q)+3])`, i.e. prefix-anchored;
  - otherwise use `max(fuzz.ratio(q, cand), fuzz.partial_ratio_alignment(q, cand))`, requiring `dest_start` near 0.

  A local check on a Hebrew sentence: a truncated quote scored 100 against the original and 67.6 against a similar distractor. With two OCR substitutions the scores were 94.6 against 64.7.
- **Combine with priors:** sender filter (the quote header shows the name or "You"), type compatibility (an image quote must point to an image), and a mild recency bonus.
- **Decision rule:** accept if best ≥ 85 and (best − second) ≥ 8. Otherwise emit `reply_to: null, reply_candidates:[{id,score}], flags:["reply_ambiguous"]`. If nothing scores ≥ 60, mark `reply_target_outside_transcript`. Very short texts ("ok", "😂") are almost always ambiguous; prefer the nearest earlier exact match but keep the flag.

**Thumbnail quotes [R]:**
- Reply thumbnails are a square, center-cropped and downscaled copy of the original [B]. For each image candidate, center-crop it to the thumbnail's aspect ratio and resize both to 64×64.
- Score = 0.4·(1 − pHash Hamming/64) + 0.3·HSV-histogram correlation + 0.3·SSIM (`skimage.metrics.structural_similarity`).
- If the crop is not centered, fall back to multi-scale `cv2.matchTemplate` (TM_CCOEFF_NORMED, scales 0.1–1.0) of the thumbnail inside the original.
- ORB or SIFT feature matching helps only for large thumbnails; at 64 px they return too few keypoints.
- Same accept and margin rule as for text quotes.

## 6. Emoji recognition inside text lines

**Detection [R]:**
- Body text is near-monochrome. Build a chroma mask (`sqrt(a²+b²) > ~15` in Lab), take connected components, and merge them into near-square boxes of about 1.0–1.4 em per glyph.
- Exclude colored text (links, sender names) by glyph solidity: emoji are filled blobs with area fraction above 0.35.
- Low-chroma emoji (💀 ⚪ 🖤 ☁️) are caught as near-square OCR boxes with low confidence or junk output. Verify them by template distance.
- Re-tighten every box against the known bubble color before matching (this improved the jitter results below).

**Identification [V-local, Pillow 13 + system Noto Color Emoji, 3,761 of 3,963 Emoji 18.0 fully-qualified sequences that render as a single glyph]:**

Feature: Lab image resized to 24×24 with absolute color kept, nearest neighbor by L2 distance. Queries were composited on the bubble color, downscaled, JPEG q80, and their bbox jittered.

| glyph px | exact crop | ±1 px | ±2 px |
|---|---|---|---|
| 40 (typical phone) | 99.1% | 96.5% | 92.6% |
| 24 (desktop) | 95.7% | 87.1% | 70.3% |

Notes on these results:
- Most remaining errors are skin-tone or gender variants of the same base emoji.
- Mean-centered or normalized features did noticeably worse on skin tones, so keep absolute color.
- pHash alone gave 93–98% on a 441-emoji subset.
- **Cross-vendor matching is weak.** Twemoji queries against Noto templates reached 35% top-1 and 50% top-5, and pHash only about 10%.
- Expected improvements: a ±2 px shift search, a two-stage shortlist→verify, and a frequency prior.

**Vendor matrix:**
- WhatsApp uses its own set on Android and Web and Apple's on iOS ([Emojipedia](https://blog.emojipedia.org/whatsapp-releases-its-own-emoji-set/), [list](https://emojipedia.org/whatsapp)). [V]
- Telegram uses Apple-based emoji on all platforms by default ([Emojipedia](https://www.emojipedia.org/telegram)). [V] Telegram Desktop and Telegram X offer alternative sets, and Android has a "system emoji" option. [B]
- Hinge renders system emoji: Apple on iOS, Noto on Pixel, Samsung's set on Galaxy devices. [B]

**Template-acquisition strategies [R]:**
1. **Bundled Noto set (OFL/Apache).** Render with `ImageFont.truetype(NotoColorEmoji.ttf, 109)` and `embedded_color=True` (the CBDT strike is 109 px; this worked locally [V]). Covers stock Android and Hinge on Pixel.
2. **Runtime font harvesting.** On macOS, render `Apple Color Emoji.ttc` (sbix strikes) into a local cache that is never redistributed. [B: Pillow/FreeType sbix rendering must be tested]
3. **Calibration chat** (recommended for the WhatsApp, Samsung and Apple sets).
   - The CLI generates numbered blocks of all RGI sequences.
   - The user pastes them into "Message yourself" (WhatsApp) or Saved Messages (Telegram) and screenshots them.
   - The tool parses the grid in known order and stores templates per (app, platform, theme, em-size).
   - This is legally clean (the user's own rendering, kept locally) and pixel-accurate.
4. **Learn-as-you-go cache.** Cluster unknown emoji crops by hash. In interactive mode the user labels each cluster once; otherwise the output shows `<emoji?:cluster-17>` with the best guess and a confidence value.

Use Unicode `emoji-test.txt` as the canonical sequence list; version 18.0 is dated 2026-04-30 and has 3,963 fully-qualified sequences [V](https://unicode.org/Public/emoji/latest/emoji-test.txt). ZWJ sequences that the template font lacks render as several glyphs; detect them as such and output the components.

## 7. Overlapping screenshots: stitch vs. merge

- **Pairwise offset [R]:**
  - On the masked viewports, compute row signatures: each row downsampled to 64 Lab columns, gradient-weighted.
  - For each candidate offset d ∈ [0.05H, 0.95H], compute masked NCC over the overlapping rows.
  - Refine with `cv2.matchTemplate` of a 0.25H strip.
  - Accept if peak ≥ 0.95, the margin to the second peak is ≥ 0.03, and the overlap is ≥ 2 em.
- **Why not `cv2.phaseCorrelate` alone:** it handles pure translation [V](https://docs.opencv.org/3.4.2/d7/df3/group__imgproc__motion.html), but repetitive bubbles and wallpaper produce aliasing peaks. Use it only as a second opinion.
- **Why not `cv2.Stitcher` in SCANS mode:** it exists in 5.1 [V-local], but it blends pixels and can't mask changing overlays.
- **Mask before matching:** sticky date chip, scroll-to-bottom button, unread badge, typing indicator, keyboard. The plugin provides `overlay_masks(img)`.
- **Ordering:** build a directed graph of all pairwise offsets (N is small, so O(N²) is fine), place shots on an absolute y axis, and sort. Use metadata time only to break ties. Detect reverse capture order.
- **Gaps:** if a shot has no confident overlap with any other, and message-level alignment finds no shared messages, insert a `System: [gap: possible missing messages]` with `flags:["gap"]`. Order the disconnected pieces by date separators and message times, and flag uncertain order.
- **Merge:**
  1. Map each shot's messages into canvas y.
  2. Match messages by IoU ≥ 0.5 plus text similarity ≥ 85 plus same type and side.
  3. Prefer complete over `partial` versions.
  4. For mutable state (ticks, reactions, "edited"), take the later capture time.
  5. Fallback when pixel offsets fail (a theme or keyboard change): Needleman–Wunsch over message fingerprints (side, normalized text, time, type, image pHash).
- **Append mode:** store these fingerprints in the JSON and align new shots against the transcript's tail. Existing IDs never change; new messages get fresh IDs.

## 8. Time anchors

**Sources:**
- **AOSP Android** [V](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/src/com/android/systemui/screenshot/ImageExporter.java):
  - filename `Screenshot_%tY%tm%td-%tH%tM%tS.png` (display-ID variant `...-display-N.png`), PNG by default;
  - EXIF tags: `DateTimeOriginal`, `SubSecTimeOriginal`, **`OffsetTimeOriginal`**, `Software="Android <build>"`, `ImageUniqueID`.
  - Pillow reads PNG eXIf via `getexif().get_ifd(IFD.Exif)` [V-local].
- **Samsung:** `Screenshot_YYYYMMDD_HHMMSS_<AppLabel>.jpg`, with app-label naming since Oreo ([9to5Google](https://9to5google.com/2017/12/19/samsung-name-screenshots-location-based-android-oreo/)); a real example is `Screenshot_20230418_221813_WhatsApp_1.jpg`. Whether the label is localized in a Hebrew UI is unverified [B].
- **iOS:** files are named `IMG_1234.PNG` [B]. Photos detects screenshots via an EXIF or XMP `UserComment="Screenshot"` [B](https://discussions.apple.com/thread/256003636). PNG screenshots often lack a capture date, and AirDrop sets the file dates to the transfer time [B](https://discussions.apple.com/thread/254479920). Pillow's `getxmp()` needs `defusedxml` [V-local].
- **macOS:** `Screenshot YYYY-MM-DD at HH.MM.SS.png` (localized; older "Screen Shot") [V](https://www.macrumors.com/how-to/change-screenshot-filename-macos).
- **Windows:** Win+PrtScn gives `Screenshot (N).png`, with no date. The Snipping Tool gives `Screenshot YYYY-MM-DD HHMMSS.png` [B](https://learn.microsoft.com/en-us/answers/questions/3902410/screenshot-how-to-change-default-file-name-to-date).
- **WhatsApp-forwarded images:** `IMG-YYYYMMDD-WAxxxx.jpg` or `WhatsApp Image YYYY-MM-DD at HH.MM.SS.jpeg`. These carry the *transfer* date, not the capture date [B].

**Precedence [R]:**
1. EXIF DateTimeOriginal with offset
2. filename timestamp
3. XMP DateCreated
4. file birthtime or mtime (low confidence)
5. current local time

Cross-check against the OCR'd status-bar clock: if the time of day disagrees by more than 2 minutes, distrust the metadata. Also apply a sanity bound: the anchor must be ≥ the newest visible message time. Store `time_anchor:{value, source, confidence}` per screenshot, and resolve weekday names per screenshot (relevant when a batch spans midnight).

## 9. Library stack (versions and licenses from pip metadata, [V-local])

| lib | version | license | use |
|---|---|---|---|
| numpy | 2.5.5 | BSD-3 (+0BSD/MIT/Zlib parts) | arrays |
| opencv-python-headless | 5.1.0.95 | Apache-2.0 | morphology, CC, contours, matchTemplate, phaseCorrelate, kmeans, ORB/SIFT. **No AKAZE**; `img_hash`/`ximgproc` need opencv-contrib |
| Pillow | 13.0.0 | MIT-CMU | I/O, EXIF/eXIf, emoji font rendering |
| scikit-image | 0.26.0 | BSD-3 | SSIM, regionprops |
| scipy | 1.18.1 | BSD | signal processing, Hungarian assignment |
| ImageHash | 4.3.2 | BSD-2 | pHash/dHash/colorhash |
| RapidFuzz | 3.14.6 | MIT | quote matching |
| python-bidi | 0.6.12 | LGPL (v3 per distros), Rust `unicode-bidi` wrapper | visual↔logical order |
| defusedxml | – | PSF [B] | XMP parsing |

Also consider (all [B]): `regex` for `\X` grapheme clusters (emoji sequences), `emoji` (BSD) for CLDR names, and `dateparser` (BSD-3, Hebrew support) for chip dates. OCR-engine choice is out of scope here.

## Risks & open questions

- **Custom themes:** Telegram gradient bubbles and wallpapers, and photo wallpapers, break color segmentation. The shape fallback and confidence flags are needed. Which themes count as "supported" in phase 1?
- **Proprietary emoji sets:** the WhatsApp, Apple and Samsung sets can't be shipped. Is a calibration flow (`chatsocr emoji calibrate`) acceptable UX? Without one, emoji on WhatsApp-Android and iOS fall back to `<emoji?>` plus a guess (≈35% cross-vendor).
- **Small emoji:** desktop glyphs (~20–24 px) and JPEG damage drop accuracy to about 70–87% unless a shift search is used. Low-chroma emoji are easy to miss.
- **Repetitive content** (identical "ok" bubbles, empty wallpaper) can produce false overlaps. Always confirm with a message-level match.
- **Short or duplicate quotes** are inherently ambiguous. The JSON needs `reply_candidates` and flags.
- **iOS time anchor:** iOS PNGs may have no reliable date, so the status-bar clock and the in-chat "Today" markers become essential. This needs real-device samples to confirm.
- **Hebrew/RTL UI mirroring:** sides, quote-bar position and time position all flip. Plugins must declare their UI direction, or the engine must detect it.
- **Unverified UI geometry:** quote line limits, the WhatsApp image frame, thumbnail cropping and tick colors per app version all need a small labeled screenshot corpus. Recommend one per app×platform×theme in `tests/fixtures`, which also serves AI agents writing plugins.
- **OpenCV 5.x API drift** (AKAZE moved): pin the major version.
