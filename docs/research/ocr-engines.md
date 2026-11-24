# Local OCR for mixed Hebrew and English chat screenshots: research report

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

## Key takeaways for the requirements and design

1. **Only two usable local engines read Hebrew well enough.**
   - Tesseract 5 with `heb+eng`.
   - `Rivok/paddleocr-hebrew`, a new Apache-2.0 set of fine-tuned PaddleOCR models exported to ONNX.

   None of the mainstream "modern" engines ship Hebrew: PaddleOCR PP-OCRv5/v6, PaddleOCR-VL, RapidOCR, EasyOCR, docTR pretrained models, Apple Vision. Surya does support Hebrew, but it is now a 689M-parameter vision-language model (VLM) with restricted weights. That conflicts with the "no LLM at runtime" rule and is very slow on CPU.
2. **Recommended primary engine:** Tesseract 5.5.x through **tesserocr 2.13** (2026-11-12). Its wheels bundle Tesseract 5.5.4 for Windows, macOS and Linux, so there is no system install. Use `tessdata_best` models with `heb+eng`, LSTM mode, one OCR call per bubble crop, and read the words from the TSV output.
   - Measured here: 0.06 s per bubble in-process with `tessdata_best`.
   - **Recommended secondary or fallback engine:** Rivok's ONNX models, as an optional extra.
3. **Bidi must be owned by our program, not trusted to the engine.** In tests on rendered chat bubbles, both engines returned logical order. However, both assumed an RTL paragraph for lines that the app had laid out as LTR, for example `WhatsApp שלי לא עובד :(`. That reverses the word order. The program needs a bidi normalizer that works from word bounding boxes and takes the paragraph direction as an explicit input.
4. **Paragraph direction is sometimes truly ambiguous from pixels.** In a single-line bubble that starts with an LTR word on one side and an RTL word on the other, both directions satisfy the first-strong-character rule. The design needs an app plugin hook for the direction rule, multi-line alignment as evidence, and a `bidi_ambiguous` flag in the output.
5. **Emoji must be detected and masked before OCR.** No engine recognizes them; Tesseract turned ❤️😂 into "2" or "שָ 63". A simple colour test found the emoji exactly, and masking them gave clean text (details in the emoji section). Recognizing which emoji it is needs a separate step, such as matching against templates of each app's emoji set.
6. **Preprocessing:**
   - Crop each bubble and segment lines.
   - Convert to grayscale and make the text dark on a light background.
   - Do not binarize: Otsu thresholding gave no gain.
   - Upscale 2–3x only when text is small (desktop/web screenshots, timestamps).
   - Read timestamps in their own call with `eng`, a character whitelist and regex validation.
   - **Set `OMP_THREAD_LIMIT=1`.** Without it, one Tesseract call took 8.5 s instead of 0.1 s in this container.
7. **Backend interface:** every engine returns lines, then words, each with a bounding box, text, a 0–1 confidence and a script. Each backend declares capability flags, including `text_order: logical|visual` and whether it mirrors brackets. A shared normalizer then produces NFC, logical-order text with bidi control characters stripped.

---

## Engine comparison

Footprint figures marked as measured come from this container. PyPI versions and dates were taken from the PyPI JSON API on 2026-11-24.

### Tesseract 5 (pytesseract or tesserocr)

| Aspect | Finding |
|---|---|
| Hebrew | Yes. `heb` alone turns Latin into Hebrew garbage, so use `heb+eng`. `eng+heb` was slightly worse. |
| Mixed-script line | Good. Logical order, brackets mirrored correctly, numbers/URLs/phone numbers correct. Weak spots: Hebrew prefix joined to Latin by a hyphen (`ב-gym` read as `gym-1 "IN`), `I` read as `\|`, and the paragraph-direction error. |
| Boxes and confidence | Block, paragraph, line and word boxes; word confidence 0–100 via `image_to_data` or `GetTSVText`. Per-symbol alternatives via `lstm_choice_mode`. |
| CPU speed (1 thread) | In-process (tesserocr): `fast` 0.022 s per bubble, `best` 0.063 s per bubble. Subprocess (pytesseract): 0.13–0.25 s per bubble. |
| Footprint (measured) | tesserocr wheel: 5.6 MB Linux, 3.6–4.1 MB macOS, 23 MB Windows. Models: `tessdata_best` heb 3.7 MB + eng 15.4 MB; `tessdata_fast` heb 0.96 MB + eng 4.1 MB. |
| Platforms | Windows, macOS, Linux. |
| License | Apache-2.0. tesserocr is MIT. |
| Maintenance | Active: 5.5.4 fixed 17 security advisories. |

### Rivok/paddleocr-hebrew (ONNX)

| Aspect | Finding |
|---|---|
| Hebrew | Yes. 120-character charset covering Hebrew, Latin, digits, punctuation, ₪ and €. |
| Mixed-script line | Good with the NRTR cascade (correct `ב-gym`, PDF, URL, prices). The fast CTC head alone drops Latin and digit runs, as its model card warns. Same paragraph-direction error as Tesseract. |
| Boxes and confidence | Line text plus per-step probabilities (CTC). A word detector is available. |
| CPU speed | CTC only: about 0.4 s per bubble. With the NRTR cascade: about 1.1 s per bubble (4 threads). |
| Footprint | CTC int8 20 MB, NRTR 72 + 27 MB, detector 4.6 MB, plus onnxruntime. |
| Platforms | Windows, macOS, Linux (onnxruntime). |
| License | Apache-2.0. |
| Maintenance | Created 2026-08-29, updated 2026-11-23, single maintainer. |

### Engines without usable Hebrew

| Engine | Finding |
|---|---|
| PaddleOCR 3.7.0 (PP-OCRv5/v6) | No Hebrew. Apache-2.0. |
| RapidOCR 3.10.0 | No Hebrew; runs PaddleOCR models. Apache-2.0. |
| EasyOCR 1.7.2 (2024-09) | No Hebrew. Apache-2.0. Barely maintained. |
| docTR 1.1.0 / OnnxTR 0.10.0 | A `hebrew` vocabulary exists, but no pretrained Hebrew model was found; the `multilingual` vocabulary excludes Hebrew. Apache-2.0. |
| kraken 7.2 | Supports RTL/bidi, but is built for historical manuscripts. Linux and macOS only. Hebrew models are research models, possibly CC BY-NC-SA. Apache-2.0. |
| Surya (surya-ocr 0.24.0, "Surya 2.5") | Hebrew supported (76.8% on Datalab's own benchmark). It is a 689M VLM that needs a llama.cpp server on CPU, and returns block-level HTML output only. Code Apache-2.0; weights modified OpenRAIL-M, free only below $5M funding/revenue. |
| Apple Vision (ocrmac 1.0.1) | No Hebrew in Apple's Live Text language list. macOS only. MIT wrapper. |
| Windows.Media.Ocr (winocr 0.0.15, 2024-10) | Hebrew unverified; I believe no Hebrew OCR pack exists. Windows only. |

### Sources and evidence

**Verified:**
- **Tesseract** latest release is 5.5.4: [releases](https://github.com/tesseract-ocr/tesseract/releases). tesserocr wheels "bundle Tesseract, Leptonica" ([PyPI](https://pypi.org/project/tesserocr/)), confirmed in a venv: `tesseract_version()` returned 5.5.4. The Ubuntu `tesseract-ocr-heb`/`eng` packages are byte-identical in size to `tessdata_fast`, so distro installs give the weaker models.
- **Tesseract ResultIterator** is documented as "capable of iterating in proper reading order over Bi Directional (e.g. mixed Hebrew and English) text" ([resultiterator.h](https://raw.githubusercontent.com/tesseract-ocr/tesseract/main/include/tesseract/resultiterator.h)).
- **Tesseract parameters:** `tessedit_do_invert=1` by default ("Try inverted line image if necessary"); `thresholding_method` 0 = Otsu, 1 = LeptonicaOtsu, 2 = Sauvola. Tesseract docs: "for 4.x version use dark text on light background", plus the page segmentation mode list ([ImproveQuality](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html)).
- **Rivok/paddleocr-hebrew:** [model card](https://huggingface.co/Rivok/paddleocr-hebrew) and its `config.json` state `"text_order": "logical"`. Vendor numbers for bilingual Hebrew/Latin lines: 2.33% character error rate (CER) vs Tesseract 16.14%. These are vendor-reported and not independently verified.
- **PaddleOCR:** PP-OCRv5 per-script model list has no Hebrew ([docs](https://www.paddleocr.ai/main/en/version3.x/algorithm/PP-OCRv5/PP-OCRv5_multi_languages.html)). PP-OCRv6 covers 50 languages: Chinese, English, Japanese and 46 Latin-script ones ([docs](https://www.paddleocr.ai/latest/en/version3.x/algorithm/PP-OCRv6/PP-OCRv6.html)). PaddleOCR-VL's supported-language table (Appendix B) has no Hebrew ([arXiv 2510.14528](https://arxiv.org/html/2510.14528v3)).
- **EasyOCR:** language list has no Hebrew or Yiddish ([jaided.ai](https://www.jaided.ai/easyocr/)).
- **docTR:** vocabularies in [vocabs.py](https://raw.githubusercontent.com/mindee/doctr/main/doctr/datasets/vocabs.py).
- **Surya:** license, VLM architecture and server requirement in the [README](https://raw.githubusercontent.com/datalab-to/surya/HEAD/README.md); Hebrew row in [multilingual.md](https://raw.githubusercontent.com/datalab-to/surya/HEAD/static/docs/multilingual.md).
- **Apple:** Live Text language list has no Hebrew (and no Arabic) ([apple.com/ios/feature-availability](https://www.apple.com/ios/feature-availability/)).
- **kraken:** Linux/macOS, Python 3.10–3.13, "historical and non-Latin" focus ([README](https://raw.githubusercontent.com/mittagessen/kraken/main/README.rst)).

**Believed but unverified:**
- Apple Vision's `supportedRecognitionLanguages` matches the Live Text list, so it has no Hebrew.
- Windows OCR has no Hebrew pack. Check with `Get-WindowsCapability -Online | ? Name -like 'Language.OCR*'`.
- The LSTM whitelist has been supported since Tesseract 4.1. It did work empirically in 5.3.4.

**Other Hebrew models found (not recommended):**
- `ronylicha/gigapdf-ocr-hebrew`: non-commercial license (PolyForm NC).
- `davidventura` / `trysem` PP-OCRv6 Hebrew fine-tunes: output in visual order, and only tested on 3 pictures.

---

## Our experiment

### Setup

- Chrome headless rendered WhatsApp-style bubbles using the real WhatsApp light/dark colours (`#D9FDD3`/`#FFFFFF` on `#EFEAE2`; `#005C4B`/`#202C33` on `#0B141A`), with `dir=auto`.
- 10 lines mixing Hebrew, English, digits, a URL, brackets and emoji.
- Rendered at device pixel ratio 1, 2 and 3.
- Bubbles were found by colour mask plus connected components (exact on all renders), then one OCR call per bubble.

### Results

CER below excludes the emoji line and is measured after stripping bidi marks (dpr = device pixel ratio):

| Configuration | CER | Exact lines |
|---|---|---|
| Tesseract `best heb+eng`, dpr 3 | 0.11–0.12 | 6/9 |
| Tesseract `fast heb+eng`, dpr 3 | 0.18 | 4/9 |
| Tesseract `heb` only | 0.36 | — |
| Rivok cascade | 0.15–0.18 | 5–6/9 |
| Rivok CTC only | 0.23–0.25 | — |

- **Light vs dark, raw vs inverted vs Otsu:** differences of 0.01 or less.
- **dpr 1 (desktop-like) with `best`:** CER fell from 0.155 to 0.112 with 2x upscaling (light theme), and from 0.103 to 0.082 with 3x (dark).
- **Timestamps** (11 CSS px at dpr 3): 9/10 correct at 1x and 10/10 at 2x with `-c tessedit_char_whitelist=0123456789:`. Without the whitelist, letters crept in.

**Caveat:** this is a small synthetic sample using Liberation Sans, not Roboto, SF or Segoe. The numbers are directional only.

---

## RTL output order: findings and pitfalls

| Engine or API | Order of returned text |
|---|---|
| Tesseract `GetUTF8Text`, TSV, `image_to_data` | **Logical**, brackets already mirrored (`(24/11)` correct). Inserts **U+200E/U+200F** (LRM/RLM) around opposite-direction runs, for example `‎WhatsApp‏`. These must be stripped. |
| **tesserocr `iterate_level(ri, RIL.WORD).GetUTF8Text()`** | **Visual (reversed) characters** for RTL words: `השיגפה`, `14:30-ב`, `)24/11(`. Words come in RTL reading order. TEXTLINE iteration returned only the first word. Avoid it; use `api.GetTSVText()` or hOCR. Verified with tesserocr 2.13.0 and Tesseract 5.5.4. |
| Rivok ONNX | Logical, per its config and confirmed in our runs. |
| CTC line models in general (gigapdf, PP-OCRv6 Hebrew fine-tunes) | **Visual.** Their own cards say the output must be reversed. |
| PaddleOCR (Arabic models) | Visual, then a heuristic `pred_reverse` that keeps `[a-zA-Z0-9 :*./%+-]` runs together. Brackets are not mirrored ([rec_postprocess.py](https://raw.githubusercontent.com/PaddlePaddle/PaddleOCR/main/ppocr/postprocess/rec_postprocess.py)). |
| EasyOCR (RTL scripts) | Applies `python-bidi get_display` to the output in utils.py. |

**Pitfalls observed:**
- **Paragraph direction.** Tesseract read `WhatsApp שלי לא עובד :(` (an LTR paragraph) as `); שלי לא עובד WhatsApp`. Rivok also got `I bought it for 120 ₪ at שופרסל` wrong, producing `שופרסל I bought it for 120 ₪`. A prototype that rebuilds the line from word boxes recovered the correct LTR reading. However, both the LTR and RTL hypotheses passed the first-strong check, so this case is ambiguous.
- **Hebrew prefix joined to an LTR token by a hyphen** (`ב-gym`, `ב-14:30`). Tesseract broke on it; Rivok was fine.
- **Emoji:** misread as junk text by every engine.
- **python-bidi 0.6.12's default `bidi.get_display` (Rust) does not mirror brackets.** `get_display('(א)', base_dir='R')` returns `')א('`. The legacy `bidi.algorithm.get_display` does mirror and returns `'(א)'`. Do not use the Rust version to produce glyph-level ground truth.

**Recommended normalization pipeline** (the `BidiNormalizer`):

1. Get words with bounding boxes from the engine. If the engine reports visual order, first reverse the characters inside RTL words and re-mirror their brackets, so each word is in logical order.
2. Sort words by x position to get the visual order.
3. Resolve each word's direction: R if it contains Hebrew, L if Latin, neutral if digits or punctuation only. Mixed words like `ב-gym` are treated as R.
4. Run a word-level inverse Unicode Bidi Algorithm with an explicit paragraph direction P:
   - Neutrals take their neighbours' direction when both sides agree, otherwise P.
   - Group words into runs, order the runs by P, and order words inside each run by the run's direction.
   - Mirror only characters with `Bidi_Mirrored` (see BidiMirroring.txt) that end up in RTL-resolved runs.
5. Choose P using, in order:
   - (a) the plugin's rule, defaulting to first strong character as in Android, HTML `dir=auto` and iOS "natural" alignment (believed; check per app);
   - (b) the ragged edge of wrapped lines;
   - (c) the engine's own guess, with `bidi_ambiguous=true` when both hypotheses are consistent.
6. NFC-normalize and strip U+200E, U+200F, U+061C, U+202A–U+202E and U+2066–U+2069. Keep `raw_text` and `paragraph_direction` per message.
7. Test by rendering real bubbles with headless Chrome or Pango from logical strings, then asserting a round-trip back to the same logical text.

---

## Preprocessing guidance

- **OCR the bubble, not the screenshot.** The plugin supplies the bubble box. Exclude the time, ticks, reactions, the bubble tail and rounded corners. Pad 10–20 px with the bubble's background colour; the Tesseract docs warn against both missing and huge borders.
- **Line segmentation:** a horizontal projection profile is reliable on flat bubble backgrounds. Line recognizers (the Rivok/Paddle family) need it. Tesseract page segmentation mode 6 on the whole bubble matched mode 7 per line.
- **Polarity:** convert to grayscale and invert when the bubble's luminance is below 0.5. Tesseract would try inversion internally anyway, but doing it explicitly is deterministic.
- **No hard binarization.** Otsu gave no gain, and anti-aliasing helps the LSTM.
- **Coloured runs:** links and mentions are drawn in other colours (WhatsApp links are blue, `#027EB5` light / `#53BDEB` dark per WhatsApp theme values, unverified). Use plain grayscale and treat colour only as a hint.
- **Emoji detector:** measure each pixel's distance from the line between the background and text colours in RGB space. Pixels more than about 40 away, after closing and connected-component analysis, are emoji. In both themes this found exactly the two emoji, and OCR after masking returned `אוהב אותך נתראה`. Record each emoji's box and insert it into the visual word order as a neutral token. Recognition happens elsewhere.
- **Scale:** aim for a glyph height of roughly 30–50 px. Phone screenshots (device pixel ratio 2.6–3.5) need no scaling. Desktop/web at 100% and small timestamp text should be upscaled 2–3x with cubic or Lanczos interpolation.
- **Timestamps and dates:** use a separate call with `eng`, mode 7 (single line), a whitelist such as `0123456789:.APMapm/`, 2x upscaling, and a strict regex such as `^\d{1,2}:\d{2}(\s?[AP]M)?$`. Hebrew weekday and "Yesterday"/"Today" separators should go through `heb+eng` and then a closed-vocabulary matcher.
- **Performance:** set `OMP_THREAD_LIMIT=1` and parallelize across bubbles with processes. Keep one tesserocr API instance per worker so models are not reloaded.

---

## Recommended engines and backend interface

**Primary engine:** `TesseractBackend` using tesserocr, `tessdata_best` `heb+eng` (about 19 MB, downloaded on first run or vendored), LSTM mode, page segmentation mode 6 per bubble, words read from TSV.

**Secondary engine:** `RivokOnnxBackend` (optional `pip install chatsocr[hebrew-onnx]`). Use it as a re-read for lines with low confidence or mixed scripts, or as a user-selectable primary. Apple, Windows and Surya backends are not viable for Hebrew; the interface leaves room to add them later.

```python
class BackendCaps(TypedDict):
    languages: set[str]            # {"he","en"}
    levels: set[Literal["line","word","char"]]
    text_order: Literal["logical","visual"]
    mirrors_brackets: bool
    needs_line_crops: bool         # True for CTC/NRTR line recognizers
    supports_whitelist: bool
    confidence_calibrated: bool

@dataclass
class OcrHint:
    languages: list[str]; level: Literal["block","line"] = "block"
    whitelist: str | None = None; pattern: str | None = None   # e.g., timestamp regex
    paragraph_direction: Literal["ltr","rtl","auto"] = "auto"

@dataclass
class OcrWord:  bbox: Box; text: str; raw_text: str; confidence: float  # 0..1
                script: Literal["Hebr","Latn","Zyyy","Emoji"]; alternatives: list[tuple[str,float]] = ()
@dataclass
class OcrLine:  bbox: Box; words: list[OcrWord]; text: str; raw_text: str; confidence: float
                paragraph_direction: Literal["ltr","rtl"]; bidi_ambiguous: bool
@dataclass
class OcrResult: lines: list[OcrLine]; engine: str; engine_version: str; model_id: str; elapsed_ms: float

class OcrBackend(Protocol):
    name: str; caps: BackendCaps
    def recognize(self, image: np.ndarray, hint: OcrHint) -> OcrResult: ...  # bbox in input-image px
```

How the pieces fit:
- Adapters return raw engine output. The engine core then runs the shared `BidiNormalizer` (driven by the `caps` flags), maps coordinates back to the original screenshot, and rescales confidences to 0–1.
- An `EnsembleBackend` can arbitrate between engines using confidence and regex or lexicon agreement.
- Each line in the output JSON records its engine, version and model for provenance.
- Results are cached by crop hash, which supports append runs.

---

## Risks and open questions

- **Accuracy on real screenshots is unmeasured.** Build a labelled corpus covering WhatsApp, Telegram and Hinge on iOS, Android and desktop, light and dark, before locking in the engine choice. Our CER figures come from 10 synthetic lines.
- **Paragraph-direction rules per app and OS locale are unverified.** Examples: does WhatsApp on iOS use first-strong? Do Hebrew-locale phones default to RTL? Some single-line cases are inherently ambiguous and must be flagged.
- **Rivok's models are young** (single maintainer, vendor benchmarks). NRTR decoding is about 1 s per line on CPU, because the split export re-runs the decoder for every token. Its PyPI/GitHub package was not reviewed.
- **Tesseract weaknesses:** `I`/`|` confusion, hyphen-prefixed mixed tokens, and Hebrew geresh/gershayim (`'` `"`). A lexicon and regex post-correction layer is recommended.
- **The tesserocr word iterator returns visual order.** This is a likely tesserocr quirk; the cause is not verified. Pin versions and keep a regression test.
- **Windows OCR's Hebrew support is unverified.** Apple Vision lacks Hebrew per the Live Text list.
- **Whether Surya, or any VLM OCR, counts as "an LLM at runtime"** is for the user to decide. Its weights are also commercially restricted above $5M funding/revenue.
- **Emoji identification** needs per-platform template sets (Apple, Google/Noto, Samsung, WhatsApp, Telegram). Their licenses need checking.
