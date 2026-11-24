# Prior art: turning chat screenshots into structured transcripts

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

Tags: **[V]** verified, with the source named. **[B]** believed but not verified. **[R]** recommendation. Star counts and push dates come from GitHub search on 2026-11-24.

## Key takeaways for the requirements and design

- **Nobody has solved this problem in open source.** Every screenshot-to-chat tool I found is a small project (0–900 stars) aimed at one app, mostly WeChat. None handles replies, Hebrew or other right-to-left text, emoji identity, media placeholders, or appending to an existing transcript with stable IDs. We are building something new, not forking. [V]
- **The best heuristics come from jev-chat-windows (MIT).** It decides who sent each OCR box from the box's background colour and contrast, not from its x-position. It drops text that does not sit on a flat background, because that text is inside a photo or sticker. Each app is one `ChatApp` profile entry with callable predicates and self-tests that pin measured pixel values. This is a small, working version of our "one class per app" idea. [V]
- **Grey, low-contrast text is a class of its own.** Quotes, timestamps, sender names, system messages and link-card text are all grey. Existing tools throw this text away, but we need to parse it: reply quotes, per-message times and date separators all live there. [V]
- **Removing overlap by comparing text is a known failure.** Two projects document that matching text across screenshots swallows identical repeated messages ("ok", "😂"). The best stitching reference (whiteyun, MIT) aligns the images themselves, scores several candidate overlaps, ignores fixed headers and footers, and flags the pair for review when the evidence is weak. [V] **[R]** Remove overlap by image alignment, and say so in the requirements.
- **Plugin registries and per-language OCR routing are proven patterns.** manga-image-translator keeps a registry of detector and OCR classes. comic-translate sends each language to a different OCR engine. Both match our plugin design and our Hebrew/English need. [V]
- **UI-understanding research says: classic computer vision for crisp boxes, learned models for meaning, then rules for grouping and order.** UIED, Apple's Screen Recognition and the Gestalt grouping paper all point this way. Small on-device detectors work well (Google IconNet), so an optional per-plugin model is realistic. Pix2Struct, ScreenAI, Ferret-UI and similar vision-language models are ruled out by the no-LLM rule. [V]
- **There is no usable public dataset.** I found no permissively licensed, message-level labelled set of WhatsApp, Telegram or Hinge screenshots. **[R]** Plan for (a) a private golden set of real screenshots with hand-written JSON, and (b) a synthetic generator built from HTML/CSS templates rendered with Playwright, where the page structure supplies the labels.
- **Watch the licences.** Several projects are copyleft or have no licence at all: CSZSmaf/ChatOCR (AGPL), DaoChu (GPL), manga-image-translator (GPL), the older OmniParser detector (AGPL), and many repos with no licence file. **[R]** Borrow ideas from these, not code. Copy code only from MIT/Apache projects, with attribution.

---

## 1. Tools that turn chat screenshots into text

| Project | What it does and how | Apps | Signals and licence | What we learn |
|---|---|---|---|---|
| **jev-chat/jev-chat-windows** | Captures the Windows chat window and runs local OCR (RapidOCR for Chinese/English, Windows.Media.Ocr for Korean). Classifies each box by colour, merges lines into bubbles and removes scroll overlap. An LLM then drafts replies; that part is not relevant to us. | WeChat, KakaoTalk | ~900★, pushed 2026-09; README says MIT and asks to keep the NOTICE file (GitHub detects "other") | Best source of heuristics; details below |
| **pochy/chat-screenshot-parser** | PaddleOCR 2.9.1 (pinned: "v3.x has compatibility issues"), with separate Chinese and Japanese models. Sender is right of centre vs left. Timestamps and system lines are text near the centre that matches a regex. Output is JSONL. Optional LLM clean-up. | WeChat | 0★, no licence | Useful output fields: `id, timestamp, speaker, lang, type, text, reply_to, source_file, confidence, needs_review`. Reply detection is a stub in `extract.py` ("simplified", `reply_to = None`). Removes duplicates with a global set of (timestamp, speaker, text) plus text similarity, so repeats are lost. Finds images with Canny contours plus a variance above 1000. |
| **CSZSmaf/ChatOCR** ("拾语") | Offline Android app. Captures via a floating window or imports a screen recording. Recognises, de-duplicates and orders messages, crops images and stickers, and exports ZIP/HTML/JSON/TXT. | WeChat, QQ, 1:1 and group | 1★, AGPL-3.0, Kotlin | Closest to our feature set (cropped media, JSON export). Copyleft. The name is almost the same as ours. |
| **LANG-ming217/wechat-chat-ocr** | EasyOCR. Checks bubble colour first (share of green pixels next to the text) and falls back to x-position. Centred text with keywords is a system message. Exports Excel. | WeChat | 3★, no licence | Uses colour and position together because "pure colour detection is unstable" across screenshot sizes and brightness. |
| **salome-veritas/chat-screenshot-ocr** | OpenCV bubble detection plus Tesseract. Has a "simple mode" fallback that drops sender attribution. | claude.ai only | 0★, README says MIT | Shows a graceful fallback mode. |
| **jkarenko/wa-chat-analyser** | Tesseract, groups text into message blocks, custom participant names, handles low-confidence results. | WhatsApp | 1★, no licence | Small and simple. |
| **Thunder17/messenger-screenshot-parser** | Cloud OCR (Yandex Vision), then DBSCAN clustering to separate messages from timestamps. | Telegram | 0★ | Clustering as a layout step. Cloud OCR is not allowed for us. |
| **DaoChu** (PyPI `daochu` 0.2.0) | macOS. AppleScript scrolls WeChat and takes screenshots, Tesseract reads them, overlaps are removed, exports HTML/Markdown. | WeChat | GPL-3.0-or-later | Takes a capture-area offset as a CLI option. |
| **cmhac/chat-extract** | Sends video frames to an OpenAI vision LLM and writes CSV (sender, message, timestamp). | any | 4★, MIT | Shows the LLM approach we must avoid. Its sample output repeats a message across overlapping frames. |
| **nemoitis/ChatParser** | Gradio UI plus a `GET /api/v1/extract?url=` endpoint. Approach not documented. | ? | 4★, no licence | Nothing to reuse. |

**jev-chat details, all verified from `app/ocr.py` and `app/chatapps.py`:**
- **Who said it.** The function `who_said` takes the most common colour inside each OCR box as the background.
  - If that colour covers less than 45% of the box, the text is inside an image (avatar, photo, sticker) and is dropped.
  - If the profile's `is_me(bg)` matches, the line is "me": WeChat green, or KakaoTalk `#FEE500`.
  - Otherwise, contrast of 150 or more means "her", and anything lower means "gray".
  - Measured contrast: body text 178–208, own green bubble 142–150, grey text 93 or less. The code comment says the same rules work in light and dark themes.
- **Merging lines.** Lines from the same sender belong to one bubble when the gap is under 0.6 × the median line height.
- **Group names.** A short grey line on the chat background, without a colon, is taken as the sender name for the next message.
- **App profiles.** Each app is a frozen dataclass: `ChatApp(key, label, exes, main_title, skip_titles, ocr, join, is_me: Callable[[ndarray], bool], trim_top: Callable)`. The comment says "adding an app is one entry". Running the module asserts real measured RGB values.
- **Documented failures:**
  - Resized or compressed images blur the background into "hundreds of colours", so the whole screen is treated as an image.
  - Two identical consecutive messages from the same person collapse into one.
  - If OCR misses a sender-name line, the message is attributed to the previous person.
  - RapidOCR's built-in Chinese/English model read 0 of 8 Korean bubbles correctly (2.0 s per frame). Windows OCR read 9 of 9 (0.20 s).

**Other evidence:**
- **dev.to write-up (Tesseract.js).** [V] Detecting avatar blobs reached about 70–80% accuracy and was fragile, so the author switched to comparing each line's left edge with the image midpoint. Known failures: centred system messages, stickers and voice notes, theme changes, landscape screenshots. The tool asks the user to confirm the first 15 labels and has a "swap sides" button. https://dev.to/dngzihng114379/building-ai-ex-partner-chat-ocr-sender-detection-the-3-am-code-that-shouldnt-exist-1odf
- **Research that uses chat screenshots.** [V/B] AICompanionBench (arXiv 2606.04867) built 2,123 Replika conversations from Reddit screenshots; a search summary says speakers were assigned by left/right of the centre line with pytesseract (I did not see this in the paper). A scam study (arXiv 2605.16656) separated sender and receiver by side spacing and checked OCR by hand on 100 images. Chiatti et al. 2018 (arXiv 1801.01316) got 74% character accuracy on smartphone screenshots with OpenCV plus Tesseract LSTM, no training.
- **Forensic tools.** [V] Cellebrite Chat Capture (UFED/Responder, Android) scrolls and captures automatically and outputs "screenshots with textual data" for searching. I found no evidence that it builds parsed message records. MOBILedit "Smart Screenshots" is Android-only (Instagram, Signal, Skype, Slack, Telegram, Viber, WhatsApp, WhatsApp Business) and outputs images. Belkasoft X has an Android screen capturer with scrolling options. Takeaway: commercial forensic tools stop at capture plus searchable text, and they put provenance first. https://forensic.manuals.mobiledit.com/MM/smart-screenshots

## 2. Related fields

**Stitching scrolled screenshots**
- **touensan/whiteyun-long-screenshot-algorithms** (MIT, 2026-10). [V] Its rule: "visual evidence first, scroll trajectory only a prior; when evidence is insufficient, don't guess." It notes that repeating chat bubbles and fixed toolbars can make the highest-scoring overlap the wrong one. The algorithm:
  - finds fixed top and bottom bands across several frames;
  - searches coarse, then at about 4 px steps, then pixel by pixel;
  - checks agreement block by block;
  - keeps up to 8 candidates and uses the median of 3–5 confirmed seams as a prior, searching about ±48 px around it;
  - cuts at a low-ink row so no text line is split;
  - otherwise marks the pair for manual review.
- **gebn/stitcher** (MIT, PyPI 1.0.0, 2017). [V] Per-device profiles: `header_height`, `footer_height`, `additional_message_gap`, `reply_message_gap`.
- **OpenStitch4Mac** (Apache-2.0). [V] Finds overlaps automatically and shows an explicit error when two images do not overlap.

**Comic and manga translation** is the closest mature analogue: bubbles, OCR, reading order.
- **manga-image-translator** (GPL-3.0, ~10.6k★). [V] A `DETECTORS = {Detector.ctd: ComicTextDetector, …}` registry, a `CommonDetector`/`OfflineDetector` base class with `download()`, `load(device)` and `detect()`, and a cache of instances.
- **comic-translate** (Apache-2.0, ~3k★). [V] An RT-DETR-v2 bubble-and-text detector trained on 11k comic images, plus OCR routed by language: manga-ocr for Japanese, Pororo for Korean, PP-OCRv5 for everything else. **[R]** Use the same routing per line for Hebrew and English.

## 3. UI-understanding research we can borrow from

- **UIED** (MulongXie/UIED, Apache-2.0, 556★, last push 2023-11). [V] Classic computer vision ("top-down coarse-to-fine") finds non-text elements, a learned text detector finds text (EAST originally, now cloud Google OCR), and a CNN classifies the elements. The authors studied 7 methods on more than 50k GUI images. Lesson: flat UI suits classic CV for exact boxes, but text should come from a text detector, and you need a merge step. https://arxiv.org/abs/2008.05132
- **Gestalt grouping without training** (Xie et al., FSE 2022, arXiv 2206.10352). [V] Groups GUI widgets from pixels with no training data. F1 is 0.593 on 1,091 app screenshots. Relevant for grouping a bubble with its time, ticks, reactions and quote.
- **Apple Screen Recognition** (CHI 2021 best paper, arXiv 2101.04893). [V] An on-device detector trained on 77,637 screens from 4,068 iPhone apps, plus rules for grouping and ordering, plus extra models for content and state. It shipped in iOS 14 VoiceOver. The dataset was not released. Lesson: detection alone is not enough; rules for order and grouping do much of the work.
- **Google IconNet** (Voice Access). [V] A CenterNet-based on-device detector for 31 icon types, trained on more than 700k screenshots. 94.2% mAP at 9 FPS on a Pixel 3A; quantisation made it 6× faster (700 ms to 110 ms) for about 0.5% mAP loss. Suggests a plugin could ship a tiny quantised detector for ticks, voice-note, play or document icons. https://research.google/blog/improving-mobile-app-accessibility-with-icon-detection/
- **Rico.** [V] About 72k screens from 9.7k Android apps in 27 categories. [B] Licence CC BY 4.0, according to a Voxel51 mirror, not the official site. Old (2017) and has no chat labels.
- **ScreenAI** (Google 2024). [V] The model was not released. Datasets were: ScreenQA (CC BY 4.0, ~86k Q&A on ~35k Rico screens) and Screen Annotation (licence not stated). [B] The model has 5B parameters. Pattern worth copying: a text "screen annotation" of the form element type plus bounding box plus text, close to our compact text view.
- **Pix2Struct** (ICML 2023, 282M/1.3B parameters, Apache-2.0 on the Hugging Face card). [V] Generates simplified HTML from screenshots. [R] Do not use at runtime: it is generative, heavy, and close to the no-LLM line.
- **Ferret-UI, Ferret-UI 2, Ferret-UI Lite** (Apple; Lite is a 3B multimodal LLM, arXiv 2509.26539). [V] Excluded as LLMs.
- **OmniParser** (Microsoft, 25.6k★; V2 released 2025-02). [V] A YOLO detector for interactable regions plus a Florence-2 captioner; 0.6 s per frame on an A100. Weight licences: the older `icon_detect` is AGPL-3.0; `icon_detect_v3` (YOLOv9-E, added 2026-07) and the captioners are MIT. It is trained on clickable icons, not chat bubbles. Useful only as a licence warning and as proof that a small, fine-tuned YOLO pattern works.

## 4. Datasets

- **None found** with message-level labels (sender, text, time, reply target) for WhatsApp, Telegram or Hinge under a permissive licence. Searches of the Hugging Face API for "chat screenshot", "imessage" and "messenger screenshot" returned nothing relevant. [V]
- **Mobiusi/Chat-Record-Image-Dataset** (Hugging Face). Chat screens photographed under different lighting. CC-BY-NC-SA-4.0. Access needs a login (401). A vendor sample, not usable for commercial work. [V/B]
- **Roboflow Universe:** `apple-ml/chatscreenshot` (CC BY 4.0, one class "ChatS"), `collec/collec_chat` (classes bubble and chat_bubble), `chat-text-detection`. The pages returned 403 to me, so I could not check their contents or quality. [B]
- **Reis et al., ICWSM 2020:** fact-checked images shared on WhatsApp (Zenodo 3779157). These are images that circulated on WhatsApp, not chat UI screenshots. [V]
- **Suji04/Chat_ScreenShot_Classifier** (22★, 2021). A CNN that tells chat screenshots from other images; a possible baseline for app detection. [V]

## 5. Generators of fake or synthetic chat screenshots

| Generator | Style | Notes |
|---|---|---|
| **bral/text-message-image-creator** | Android Messages and iOS, light and dark | Built "for ML training data". 19 or more resolutions, 6 font sizes, a set mix of message lengths, `POST /generate-image` takes JSON. Node, no licence. [V] |
| **tinkeringmachines/chatshot** | WhatsApp iOS, dark mode | MIT. YAML/JSON input with `from`, `text`, `time`. Android style "coming soon". [V] |
| **hackerman-peter/telegram-screenshot** (npm) | Telegram, light and dark | JSON input with `incoming`, `time`, `sender`, `read`. Has read receipts and emoji-only styling. [V] |
| **AmirPhenomenal/fake-chat** | WhatsApp Android dark | Roboto, image messages, ticks, "Today" chip. React, no licence, 17★. [V] |
| **Saganaki22/iSpoof** | iMessage | Apache-2.0, html2canvas. [V] |
| **gaopengbin/wechat-dialog-generator** | WeChat | Batch import from Markdown or text, 1125×2436 export, 89★. [V] |
| **pai-pixel/feishu-privatechat-dataset-gen** | Feishu | Lesson: redrawing the whole UI from scratch in Pillow "always differs slightly"; masking and redrawing only the bubble text on a real screenshot looked most realistic. [V] |
| **MolmoPoint / CoSyn** (arXiv 2603.28069) | General UIs | Renders HTML with Playwright and uses custom JS to get every element's bounding box, so labels come free. [V] |

None of these READMEs mentions replies, right-to-left/Hebrew text or a ground-truth export. [B]

**[R]** Build our own small generator: HTML/CSS templates per app × platform × theme, rendered by headless Chromium through Playwright. The page structure gives exact bounding boxes, sender, time, the reply target's ID, emoji code points and media types as JSON ground truth. Use `dir="auto"` to produce mixed Hebrew/English text. Add random JPEG re-compression and scaling. Treat this as unit and regression data only, and measure real accuracy on a real golden set.

**Emoji prior art is thin.** Bataineh & Shambour 2019 (CC BY 4.0) only detect that an emoji is present (RGB-channel analysis, connected components, decision tree); they do not identify which emoji. `shreeshrii/tessdata_emoji` offers Tesseract trained data for emoji, quality unknown. [V/B] **[R]** In a text line, a coloured blob is almost always an emoji. Identify it by matching against glyphs rendered from the platform's emoji font.

---

## Most useful lessons for our design

1. **App plugins should be declarative profiles plus a few predicates.** Each one pins measured colours and thresholds and comes with self-tests on real pixel samples (jev-chat `ChatApp`). The engine owns OCR, line merging, overlap removal and output; the plugin owns colours, regions, regexes and the bubble-cropping function. Grouping and ordering rules in the engine come after detection (Screen Recognition, UIED).
2. **Classify every text box by its background and contrast.** Classes: own bubble, other bubble, grey metadata, text inside an image. Then send grey metadata to parsers for times, dates, quotes and names instead of dropping it. Prior art throws away exactly the information our replies and timestamps need.
3. **Remove overlap with image alignment, never text alone.** Detect fixed header and footer bands, score several candidate overlaps, and when the evidence is weak, flag the result instead of guessing (whiteyun). Make "identical repeated messages are kept" an explicit acceptance test.
4. **Every message gets provenance and confidence:** source file, bounding box, OCR confidence and a `needs_review` flag (pochy schema, forensic practice). Uncertain side, reply target or date is recorded as a flag, never a guess. This is also what makes non-interactive mode safe.
5. **Use a registry of engines and detectors, with per-language OCR routing** (manga-image-translator, comic-translate). Test Hebrew early: jev-chat showed a zh/en-only model turning Korean into 0/8 correct.

**Is any project worth building on directly? No.** jev-chat-windows (MIT) is the best source of heuristics, with attribution, but it is a live-capture tool for Windows WeChat and KakaoTalk in Chinese and Korean. CSZSmaf/ChatOCR is the closest in features but is an AGPL Kotlin Android app. The whiteyun documents (MIT) are the best reference for the overlap algorithm.

## Risks and open questions

- **Compressed or resized input.** Screenshots forwarded through WhatsApp are JPEG-compressed and rescaled, which breaks exact-colour rules (jev-chat warns about this). Requirements should state which inputs are supported (original PNG vs re-compressed), and helpers should use colour clustering with tolerance.
- **Python 3.13.** [V] `rapidocr-onnxruntime` 1.4.4 declares `requires_python <3.13`. The newer `rapidocr` 3.10.0 accepts Python 3.8 up to 4. pochy pins PaddleOCR 2.9.1 because 3.x broke its code. Every OCR dependency must be checked on 3.13.
- **Hebrew and mixed-direction text.** No chat-screenshot project handles it. OCR engines may output characters in visual order (a pattern reported for a Hebrew CTC model). Bidi ordering of the output needs its own requirement and test.
- **Replies.** No prior art links a reply to the message it quotes. The quoted message may lie outside the captured screenshots, so the schema needs an "unresolved reply" state holding the quoted text and thumbnail.
- **Hinge.** No prior art at all, and its UI (comments on prompts and photos, likes) differs from messengers. We need real samples early.
- **Test data and privacy.** No public dataset exists. A golden set of real chats is private by nature: where is it stored, how is it redacted, and can it go in the repo?
- **Licences.** Code from repos without a licence (pochy, LANG-ming217, bral, AmirPhenomenal, nemoitis) must not be copied. The same goes for AGPL/GPL code (ChatOCR, DaoChu, manga-image-translator) and the older OmniParser detector. Apple emoji images are proprietary [B], so emoji templates should come from Noto or Twemoji, with lower accuracy on iPhone screenshots as a possible cost.
- **Name.** "ChatOCR" (CSZSmaf, 拾语) is almost the same as "ChatsOCR". Probably minor.
