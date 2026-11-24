# OCR feasibility spike: mixed Hebrew + English chat-bubble text

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

## Key takeaways for the requirements/design

1. **Tesseract 5 with `heb+eng` works as a default engine if it reads one crop at a time.** On crops of pure-English text, pure-Hebrew text and date chips at phone scale, character error rate (CER) was 0–1%. That held for light and dark themes and three fonts. Over all 50 test text crops, CER was about 8% (preprocessing P1, defined below). Most of that error came from emoji junk and one specific mixed-script pattern.
2. **Tesseract consistently fails one common Israeli-chat pattern: a Hebrew prefix letter, a hyphen, then a Latin word.**
   - "ב-Zoom" came out as "Zoom-1" or "ב-2007"; "ה-iPhone" as "iPhone-n"; "ה-Google Meet" as "Google Meet-n"; "ב-WhatsApp" as gibberish.
   - That costs 15–25% CER on the affected line, and it happened in every Tesseract configuration.
   - Short Hebrew words are sometimes read as Latin ("יום" became "or", "אני" became "IX"), and "I" sometimes became "|".
3. **The only pip/ONNX deep-learning Hebrew recognizer I found is `Rivok/paddleocr-hebrew`, and it complements Tesseract rather than replacing it.** It is a community model, Apache-2.0, published 2026-11-23.
   - It read every Hebrew-prefix + Latin case perfectly, and pure Hebrew, pure English and chips at 0% CER.
   - It fails on English-dominant lines that contain a Hebrew island (34–41% CER): it reorders the runs as if the paragraph were RTL, and repeats words.
   - Its fast CTC head drops digits that sit inside Hebrew text.
   - It is slower and the models total about 180 MB.
4. **Both engines output text in logical order; never apply `get_display()` to OCR output.** On pure-Hebrew lines, CER against the logical ground truth was 0–1%, against the visual order 82%. Tesseract wraps opposite-direction runs in LRM/RLM marks (U+200E/U+200F) in 73–100% of mixed lines, so the engine must strip bidi control characters. When Tesseract misjudges a line's direction, it emits the words in visual order (one case: 77% CER).
5. **No OCR engine recognizes emoji.** Tesseract emits junk in their place ("@", "©", "טש", "J,", "9"). A simple colour-saturation blob detector that also checks hue difference from the bubble found 10 of 10 emoji with no false positives. Masking those blobs before OCR brought the rest of the text to 0% CER. Emoji therefore need their own detector plus identification step, separate from OCR.
6. **Plugins should return separate, tight bounding boxes for message text and timestamp.**
   - Timestamp crops read with Tesseract in single-line mode were 95–100% exact; with Rivok, 100%.
   - Timestamps taken from whole-page OCR were only 40–65% exact: ":" was read as "." (for example "16.15" and "(5.40") and the read ticks merged into the time.
   - Crops of the bubble including its timestamp did worse than text-only crops (about 12% vs 8% CER).
7. **Preprocessing:**
   - Converting to grayscale and inverting dark themes is essential for whole-page OCR. Without it, dark-theme pages had 44–49% CER; with it, 5–10%. For crops it makes no difference.
   - Upscaling 2× did not help at phone scale and sometimes hurt. It did help desktop/web screenshots at 1× zoom (about 15 px text): Tesseract best went from 14.3% to 9.4% CER.
   - Recommendation: upscale only when the text is small.
8. **Use `tesserocr` (MIT licence; the wheel bundles Tesseract 5.5.4) rather than `pytesseract`.** It runs in-process: about 85 ms per crop versus about 200 ms per call for `pytesseract`, which starts a subprocess and round-trips a PNG every time. It also removes the need for a system install on Windows and macOS.
9. **Design implication:** put a pluggable `OcrBackend` interface behind the engine. It should return lines or words with boxes, confidence and logical-order text, with optional script hints. The engine should own bidi normalization, emoji masking and line reassembly in the right direction. A second backend (Rivok, or a future fine-tuned model) can then be added, or used to re-read lines Tesseract is unsure of, without touching plugins.

## Setup: what installed and what failed

The environment was an Ubuntu 24.04 container with Python 3.13 and 4 vCPU.

**Worked:**
- `apt-get install -y --no-install-recommends tesseract-ocr tesseract-ocr-eng tesseract-ocr-heb fonts-noto-core`
  - Installed in 5 s: tesseract **5.3.4**-1build5.
  - `tesseract-ocr-heb` (version 1:4.1.0-2) is byte-identical to `tessdata_fast/heb.traineddata`: 961 KB, md5 `935468444ded38098c86383ae0ae0a2b` (I checked the md5 myself).
- tessdata_best downloads: `curl -L -o heb.traineddata https://raw.githubusercontent.com/tesseract-ocr/tessdata_best/main/heb.traineddata`
  - heb is 3.7 MB, eng 15.4 MB, `script/Hebrew` 12.2 MB.
  - The `github.com/.../raw/` URL was blocked by this sandbox's proxy; `raw.githubusercontent.com` worked.
- `pip install pillow numpy opencv-python-headless pytesseract python-bidi onnxruntime huggingface_hub rapidfuzz` installed:
  - Pillow 13.0.0 (wheel bundles raqm 0.11.0, so `features.check('raqm')` is True), numpy 2.5.5, opencv 5.1.0, pytesseract 0.3.13, python-bidi 0.6.12, onnxruntime 1.32.0.
- `pip install tesserocr` installed 2.13.0, which bundles **Tesseract 5.5.4** (confirmed by running it). [PyPI](https://pypi.org/project/tesserocr/) says the wheels bundle Tesseract and Leptonica for Linux, macOS and Windows x64, need Python 3.10+, and that language data is installed separately.
- Rivok models: `huggingface_hub.snapshot_download('Rivok/paddleocr-hebrew', allow_patterns=[...])`. Its Python package is on github.com, which was blocked here, so I reimplemented the pipeline from its `config.json` in about 150 lines (DBNet line detector, CTC decoding and a greedy NRTR loop).

**Not tested:**
- **Surya** (`pip install surya-ocr`, version 0.24.0, 2026-11-13). It is now only a client for a ~690M-parameter vision-language model served by `llama-server` on CPU.
  - The CPU model download is 629 MB (`surya-2.5-Q8_0.gguf`) plus a 207 MB image-projection file.
  - The `llama-server` binary could not be obtained: GitHub releases were blocked, there is no apt package for noble, and no pip binary exists.
  - The model weights use a modified OpenRAIL-M licence: free only for research, personal use and organisations under $5M in funding or revenue.
  - On Surya's own benchmark, Hebrew scores 76.8% versus English 90.3%.
  - Sources: [PyPI](https://pypi.org/project/surya-ocr/), [multilingual table](https://github.com/datalab-to/surya/blob/master/static/docs/multilingual.md), and the Hugging Face file listing for `datalab-to/surya-ocr-2.5-gguf`.
- **EasyOCR 1.7.2 has no Hebrew.** Hebrew is absent from every language list in [config.py](https://raw.githubusercontent.com/JaidedAI/EasyOCR/master/easyocr/config.py). Not tested.
- **PaddleOCR 3.7 has no official Hebrew.** The [PP-OCRv5 multilingual docs](https://www.paddleocr.ai/latest/en/version3.x/algorithm/PP-OCRv5/PP-OCRv5_multi_languages.html) do not list it. RapidOCR reuses PaddleOCR's models, so it has no Hebrew either (believed).

## Test set

`gen_synth.py` produced 5 synthetic WhatsApp-like screenshots, 1080 px wide, each with a header, 2 date chips ("Yesterday" and the Hebrew weekday "יום שלישי") and 8 bubbles with timestamps and read ticks.

| Variant | Theme | Fonts | Body text size |
|---|---|---|---|
| android_light_noto | light | Noto Sans + Noto Sans Hebrew | 42 px |
| android_dark_noto | dark | Noto Sans + Noto Sans Hebrew | 42 px |
| android_dark_dejavu | dark | DejaVu Sans | 42 px |
| iphone_light_liberation | light | Liberation Sans (an Arial clone) | 50 px |
| web1x_light_noto | light | Noto Sans + Noto Sans Hebrew | ~15 px (desktop at 1×) |

- Colours: light theme `#EFEAE2` background with `#FFFFFF` incoming and `#D9FDD3` outgoing bubbles; dark theme `#0B141A` background with `#202C33` / `#005C4B` bubbles and `#E9EDEF` text.
- Messages: an English sentence; a Hebrew sentence; "נפגש ב-Zoom בשעה 18:30 מחר"; "I bought the חומוס you wanted"; an English line and a Hebrew line, each ending in an emoji; a 2-line Hebrew message; a 2-line mixed message.
- Rendering: Pillow has no font-fallback API. To emulate Android's fallback, I converted each line with `get_display()` (python-bidi), split it into Hebrew, Latin and emoji runs, and drew each run with its own font using Pillow's BASIC layout. The same approach would work for a test-data generator.

**Preprocessing variants used in the tables:**
- **P0:** raw image.
- **P1:** grayscale, inverted when the background is dark.
- **P2:** P1 plus 2× upscale.
- **P3:** P1 plus scaling so the text is about 84 px.

**Granularities:**
- **whole:** OCR the whole page, then assign words to bubbles by box geometry.
- **bubble:** crop the whole bubble, including its timestamp.
- **text:** crop only the message text (or the date chip).
- **time:** a tight crop of the timestamp.

CER is measured against the logical-order ground truth after NFC normalization, stripping bidi control characters and collapsing whitespace. It is computed with emoji removed from the ground truth, so the junk an engine emits for an emoji still counts as error.

## Results

**Text-crop CER (%) per message type** (P1, mean over the 5 variants):

| Engine | EN | HE | HE+"ב-Zoom"+18:30 | EN+HE word | EN/HE 2-line | HE 2-line | "Yesterday" chip | HE weekday chip | Emoji lines (EN / HE) | All 50 crops |
|---|---|---|---|---|---|---|---|---|---|---|
| tess_fast heb+eng | 0 | 0 | 20.8 | 5.5 | 7.2 | 0 | 0 | 8.9 | 20 / 20 | 8.2 |
| tess_best heb+eng | 1.2 | 0 | 19.2 | 1.4 | 3.4 | 4.5 | 0 | 17.8* | 16.7 / 18.7 | 8.3 |
| tess_best eng+heb | 0 | 0 | 16.9 | 4.8 | 3.1 | 2.1 | 0 | 17.8* | 16.7 / 18.7 | 8.0 |
| tess_best script/Hebrew | 0 | 0 | 24.6 | 17.9 | 1.2 | 0 | 0 | 0 | 16.7 / 20 | 8.0 |
| rivok CTC only | 0 | 0 | 21.5 | 51.0 | 38.8 | 0 | 0 | 0 | 0 / 0 | 11.1 |
| rivok cascade (CTC → NRTR) | 0 | 0 | **0** | 33.8 | 41.2 | 0 | 0 | 0 | 13.3 / 0 | 8.8 |

\* The Hebrew weekday chip failure is only in the web 1× variant ("יום שלישי" read as "ww or"). Upscaling (P2) fixes it partly ("or שלישי"). The `script/Hebrew` model and Rivok read it correctly.

**CER (%) by granularity and preprocessing** (all variants and items):

| Engine | Text crop P1 | Bubble crop P1 | Whole page P0 | Whole page P1 | Whole page, dark theme, P0 → P1 |
|---|---|---|---|---|---|
| tess_fast heb+eng | 8.2 | 11.7 | 23.6 | 9.0 | 45 → 6 |
| tess_best heb+eng | 8.3 | 12.8 | 23.3 | 7.8 | 44–49 → 5–10 |
| rivok cascade | 8.8 | 11.7 | 16.3 | 17.5 | (line detector merges timestamps into text lines) |

**Timestamps, exact match:**
- Tight time crop: Tesseract 95–100% (best `heb+eng`: 98–100%; with a digits-and-colon whitelist, the same). Rivok: 100%.
- Whole page: Tesseract 40–65%, Rivok 57–60%.

**Runtime.** This was one process on a 4-vCPU container shared with other agents (load average 6–12), so treat these as upper bounds:

| What | Time |
|---|---|
| Tesseract whole page (1080×1424), 1 thread | fast 0.57 s; best 1.28 s |
| `pytesseract`, 8 text + 8 time crops (best) | 3.3 s per screenshot |
| `tesserocr` 5.5.4, 8 text crops | 0.68 s |
| Rivok CTC, whole page | 7.1 s (1 thread); 4.1 s (4 threads); INT8 2.8 s (4 threads) |
| Rivok cascade, whole page | 10.7 s (1 thread) |
| Rivok cascade, 8 crops | 4.3 s (4 threads) |

My NRTR loop has no KV cache, so the cascade could be made considerably faster.

**Hebrew prefix-letter test** (single lines, Tesseract fast, best `heb+eng`, best `eng+heb`, and Rivok cascade):
- "שלחתי לך ב-WhatsApp": Tesseract gave gibberish or "WhatsApp-1" (21–47% CER); Rivok 0%.
- "קניתי את ה-iPhone החדש" and "ה-Google Meet לא עובד": all Tesseract configurations gave "...-n" (about 18% CER); Rivok 0%.
- "נתראה ב-9 בערב": Tesseract `heb+eng` 0%; Rivok dropped the "9", because the CTC head deleted the digit and so the fallback to the NRTR head never triggered.
- "זה עלה 120 ₪ בסוף": Tesseract 0%; Rivok dropped "120".
- A URL inside Hebrew text: all engines 0%.

**Other observations:**
- Tesseract `heb+eng` and `eng+heb` gave almost the same results. tessdata_fast was no worse than tessdata_best here and about 2× faster on a whole page.
- On the 8 crops I compared, Tesseract 5.5.4 (`tesserocr`) produced nearly identical text to 5.3.4.

## Recommendations

1. **Default OCR backend:** Tesseract 5 through `tesserocr`, using `heb+eng`. Run it per element crop: message text with page segmentation mode 6 (single block), timestamps and chips with mode 7 (single line) and a whitelist for times. Use grayscale with inversion for dark themes, and upscale only when the text is smaller than about 25 px.
   - Ship or download tessdata_fast by default, with best as an option.
   - Pin the tessdata version and check its md5 in the test suite.
2. **`OcrBackend` interface,** owned by the engine rather than the plugins:
   - `recognize(crop, hints) -> [Line{box, text_logical, conf, words[]}]`
   - Hints could include expected scripts, whether the crop is a single line, and a whitelist.
   - Keep room for a second backend (Rivok ONNX, or a later fine-tuned model) and a per-line policy: use it when Tesseract's confidence is low, or when the output matches suspicious patterns such as `\S+-[1nh]\b` or Latin–Hebrew confusions.
3. **Bidi handling:**
   - Store logical order only. Apply NFC and strip U+200E, U+200F, U+202A–U+202E, U+2066–U+2069 and U+061C.
   - When a detector splits one row into several boxes, rebuild the row in logical order using the paragraph direction (first strong character).
   - For the text view, keep plain logical text, which is best for AI agents. Bidi isolates are only useful for display.
4. **Emoji pipeline:**
   - Detect colour blobs inside the text box using a saturation threshold plus hue distance from the bubble colour. The hue check is needed: without it, the dark outgoing `#005C4B` bubble produced false positives.
   - Mask the blobs, run OCR, then reinsert each emoji at its position, using bidi-aware mapping to word boxes.
   - Identify emoji by template matching against per-platform emoji sets (Google, Apple, WhatsApp, Samsung). This is untested.
   - Flag emoji whose match is uncertain.
5. **Timestamps:** validate against `^\d{1,2}:\d{2}( ?[AP]M)?$`, check they increase within a day, and flag violations.
6. **Turn this spike's harness into the project's test bench:** a synthetic generator with ground truth plus a small labelled set of real screenshots per app, platform and theme. Every plugin then gets a CER and timestamp-accuracy regression test, which also gives an AI agent writing a new plugin something to check its work against.

## Risks and open questions

- **Synthetic is not real.** Real screenshots add JPEG and compression artefacts, Apple SF / SF Hebrew fonts, Samsung One UI fonts, link previews, bold and monospace formatting, and more. A labelled corpus of about 50 bubbles per app, platform and theme is needed before the engine choice is final.
- **Rivok's maturity is a risk.** It was published the day before this test, had 309 downloads, has a single maintainer, and its accuracy figures are vendor-reported. Our test confirmed it is strong on Hebrew-dominant lines and weak on English-dominant ones.
- **Surya and other vision-language models are unevaluated.** Questions remain on CPU speed, hallucination and licence (commercial use is restricted).
- **The Hebrew-prefix pattern is Tesseract's main gap.** The options are a regex-plus-second-backend fallback, fine-tuning a Tesseract `heb` model, or adopting a deep-learning recognizer. This needs a decision.
- **Small desktop text at 1× zoom** caused Hebrew-to-Latin confusions ("יום" read as "or"). Should the CLI require or suggest 2× upscaling for desktop screenshots?
- **Mixed paragraphs whose direction changes between lines** are untested beyond one case. Group chats will add RTL sender names.
- **The timing figures are noisy** because the host was shared with other agents. Re-benchmark on target hardware.
