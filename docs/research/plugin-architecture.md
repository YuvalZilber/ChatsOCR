# ChatsOCR research: plugin architecture and test-by-example workflow

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

## Key takeaways for the requirements/design

1. **Discovery: use a plain class registry with three sources. Don't use pluggy as the parser contract.** The sources are (a) built-ins in `chatsocr.plugins`, (b) pip-installed packages through the `importlib.metadata` entry-point group `chatsocr.plugins`, and (c) local `.py` files or folders passed with `--plugin-dir`. The engine picks one parser per batch, which is a strategy-selection problem. pluggy is built for N plugins that all contribute to a hook (1:N), so it fits poorly here. Keep it in mind for later engine-wide hooks such as extra output renderers.
2. **The plugin contract should be whole-screen: `parse_screen(ctx) -> ScreenLayout`.** The engine ships a `StepwisePlugin` base whose template method is made of small steps you can override. The user's `crop_next_message` idea becomes an optional `SequentialPlugin` adapter on top of it, so the engine only ever sees one interface.
3. **Detection is a separate, cheap class method: `detect(ctx) -> Detection(confidence, variant, evidence)`.** It runs on a downscaled image and features the engine has already cached. Scores are added up over the whole batch. User flags and filename hints narrow the candidate list. Confidence scores are made comparable across plugins by a **cross-detection test** in the conformance kit: every plugin runs on every other plugin's fixtures.
4. **Plugins do layout and app-specific grammar. The engine does everything that is the same for all apps.** Engine: OCR, bidi, emoji matching, stitching and dedup, ID assignment, reply and date resolution, media cropping, rendering. Plugin: regions, element kinds, which sub-box has which role, and parsing of time, date-label and placeholder strings.
5. **Avoid a combinatorial explosion of classes.** Use one class per app with platform subclasses where the geometry really differs. Treat theme as palette data, keyed by variant (optional TOML that tools can write). A dedicated class for one exact app×platform×theme combination is still allowed and wins because it is more specific.
6. **The intermediate representation (IR) is plain frozen dataclasses** with pixel bboxes, roles and an `extra` dict, so app-specific elements such as Hinge comments on profile photos or prompts fit. Include group-chat fields (sender name, avatar) in v1 even before they are used. Generate the output JSON schema with pydantic v2.
7. **Ship a conformance kit as a pytest plugin** (`pytest11` entry point), modelled on scikit-learn's `parametrize_with_checks`. Every plugin also gets golden fixtures (screenshot + annotation JSON) scored with tolerances (bbox IoU, text CER), not exact snapshots.
8. **Build tooling so an AI agent can author a plugin.** The agent should never have to guess pixel values from looking at an image. Needed: a scaffold command, numeric pixel, palette and row-profile probes, a debug-overlay renderer, `detect --explain`, and an `eval` command that prints a machine-readable diff. Add a tiered authoring checklist modelled on the Home Assistant quality scale.
9. **Resolution independence comes from the engine.** It supplies `ctx.scale` (pixels per dp). Plugin constants are written in dp, so one class covers 1080-, 1170- and 1440-pixel-wide devices.
10. **Optional ML models are declared, never required.** A plugin declares `requires=("onnxruntime",)` and ships its model through `importlib.resources`. If the dependency is missing, the plugin is marked *unavailable, with a reason* instead of crashing.

---

## 1. Discovery and registration

**Verified:**
- The PyPA guide lists three discovery approaches: a naming convention scanned with `pkgutil.iter_modules()`, namespace packages (iterating `ns_pkg.__path__`), and package-metadata entry points declared under `[project.entry-points.'myapp.plugins']` and read with `entry_points(group=...)`. https://packaging.python.org/en/latest/guides/creating-and-discovering-plugins/
- The guide warns that namespace packages are "a complex feature" and that the main top-level package must never be a namespace package.
- `importlib.metadata.entry_points(**select)` returns `EntryPoints`. `EntryPoint` has `.name .group .value .module .attr .extras .dist .load()`.
  - 3.10 added the selectable interface.
  - 3.12 always returns `EntryPoints`; the old dict interface is gone.
  - **3.13 removed the tuple-like `__getitem__` on `EntryPoint`.**
  - Source: https://docs.python.org/3.13/library/importlib.metadata.html
- pluggy 1.7.0 (MIT; latest on PyPI, checked 2026-11-24): `HookspecMarker`/`HookimplMarker`, `PluginManager.load_setuptools_entrypoints()`, `firstresult`, LIFO call order, `tryfirst`/`trylast`, wrappers, and signature validation at registration. It powers pytest, tox, devpi and Kedro. https://pluggy.readthedocs.io/en/stable/
- yt-dlp:
  - Loads plugins from the namespace packages `yt_dlp_plugins.extractor` and `yt_dlp_plugins.postprocessor`. It searches config dirs, a dir next to the executable, `PYTHONPATH`/pip, and `.zip`/`.whl` archives.
  - It imports "all public classes with a name ending in `IE`/`PP`", and a leading `_` makes a class or module private.
  - "Extractor plugins take priority over built-in extractors."
  - A plugin replaces a built-in with `class MyPluginIE(ABuiltInIE, plugin_name='myplugin')`.
  - `YTDLP_NO_PLUGINS` disables plugins, and the README warns that all plugins are imported with no checks.
  - Source: https://github.com/yt-dlp/yt-dlp/blob/master/README.md#plugins
- Docling: entry-point group `"docling"`, whose module exposes `ocr_engines()`. External plugins load only with `allow_external_plugins=True` or `--allow-external-plugins`, and `--show-external-plugins` lists them. https://docling-project.github.io/docling/concepts/plugins/
- OCRmyPDF: pluggy-based. `--plugin path/to/file.py` or `--plugin module.name` loads a plugin, and the entry-point group is `"ocrmypdf"`. Hooks must be stateless because worker processes re-import plugins. https://ocrmypdf.readthedocs.io/en/latest/plugins.html
- beets: plugins live in the `beetsplug` namespace, subclass `BeetsPlugin`, and are enabled in config. Beets loads only the *last* `BeetsPlugin` subclass in a module, which is a known pitfall of subclass scanning. https://beets.readthedocs.io/en/stable/dev/plugins/index.html

**Recommendations:**
- **Built-ins:** a list kept explicitly in `chatsocr/plugins/__init__.py`. This is deterministic and works in frozen or PyInstaller builds.
- **Third-party:** the `[project.entry-points."chatsocr.plugins"]` group, where `name = "pkg.module:PluginClass"`. Entry points name the class directly, so nothing is collected by import side effects.
- **Local:** `--plugin-dir DIR`, repeatable, plus `~/.config/chatsocr/plugins/`. Load files with `importlib.util.spec_from_file_location`. From each loaded module, collect concrete `ChatPlugin` subclasses that are defined in that module (`cls.__module__ == mod.__name__`), are not abstract (`inspect.isabstract` is false) and have no leading `_`. This follows yt-dlp's public-class rule and avoids beets' "last class wins" pitfall.
- Use `__init_subclass__` for **validation only**, not as the discovery mechanism: check that `id`, `api_version` and `variants` are present and well formed when the class is defined. Pass `abstract=True` as a class keyword for intermediate bases.
- **Precedence on an `id` collision:** local > third-party > built-in (as in yt-dlp), with a warning. `chatsocr plugins list` shows each plugin's source, version, availability and reason.
- Set `CHATSOCR_NO_PLUGINS=1` to fall back to built-ins only, which is useful for debugging. A Docling-style opt-in gate is optional for a personal tool. Document that plugins are arbitrary code.
- **Import hygiene rule:** plugin modules must import cheaply. Load ML models and other heavy imports lazily inside methods. Detection needs every plugin class imported, so this matters.

## 2. Auto-detection protocol

**Recommendations:**
- **The engine builds `ScreenContext` once per image, cached lazily.** It holds:
  - the RGB array and a 1/4-scale copy
  - file mtime/EXIF time and filename tokens
  - size and aspect ratio (a platform prior)
  - background luminance (a light/dark prior)
  - dominant colours and `scale`
  - lazy cached `ocr(box)` and helpers
- **`detect` is a classmethod** and must not need an instance or model loading. Budget about 20 ms on the downscaled image; the conformance kit enforces this.
  - The cheap tier checks palette colours at expected locations, the header/composer bar geometry and icon templates.
  - A plugin may call `ctx.ocr()` for a header word or date pill, but those results are cached and shared, so the cost is paid once.
- **Hints filter the candidate set before scoring:** `--app`, `--platform`, `--theme` and per-run config. Detection still runs on the remaining candidates. If the user's choice scores very low, the output records a `detection_mismatch` flag instead of failing silently.
- **Filename hints are a prior, not proof.** Samsung One UI adds the app name to screenshot filenames, e.g. `Screenshot_<date>-<time>_WhatsApp.jpg`.
  - Verified that the app name is inserted (since Android Oreo): https://www.sammobile.com/news/oreo-samsung-naming-screenshots-capture
  - The exact pattern and the `.jpg` extension are *believed, unverified*.
  - Stock Android and iOS names carry no app name (believed).
- **Batch aggregation:** the screenshots in a batch are one chat, so score each variant by the **mean confidence over all screenshots**, skipping unparseable frames. Apply the decision rules below.

| Situation | Interactive run | Non-interactive run |
|---|---|---|
| Top score ≥ `accept` (0.6) and lead over runner-up ≥ `margin` (0.15) | Choose it | Choose it |
| Ambiguous | Prompt with a ranked list and the evidence strings | Choose the top score; output `detection: {chosen, alternatives[], ambiguous: true}` |
| Top score < `floor` (0.25) | Prompt the user | Exit with a distinct code and a report: "no plugin matched", top 3 candidates and evidence |

  The thresholds go in config. A non-interactive run never blocks.
- **Calibration:** scores from different authors aren't naturally comparable. The conformance kit's **cross-detection matrix** fixes this by requiring at least 0.7 on a plugin's own fixtures and at most 0.3 on every other plugin's fixtures. Adding a plugin re-runs the matrix, which also catches cases where the new plugin steals detections from existing ones.
- **Detection result:** `Detection` returns the *variant* it matched (e.g. the theme). One class covering 4 variants then needs one `detect` call, not four.

## 3. Sharing code across app × platform × theme

**Precedent (verified):** Calibre recipes subclass `BasicNewsRecipe`.
- Declarative attributes come first: `feeds`, `keep_only_tags`, `remove_tags`, `oldest_article`, `no_stylesheets`.
- Override hooks come second: `parse_index`, `preprocess_html`, `print_version`, `get_browser`.
- The docs say simple sites "need no code at all".
- Fast debug loop: `ebook-convert myrecipe.recipe .epub --test -vv --debug-pipeline debug`.
- Source: https://manual.calibre-ebook.com/news.html

**Recommendation: declarative-first with code hooks.**
- `WhatsAppBase(StepwisePlugin, abstract=True)` holds the app logic shared by all platforms:
  - date-label grammar (en/he: "Today", "Yesterday", weekdays, "היום", "אתמול")
  - time format and placeholder vocabulary ("Voice message", "Photo", durations)
  - reply-quote structure
- `WhatsAppAndroid`, `WhatsAppIOS` and `WhatsAppDesktop` override only geometry and step implementations that really differ.
- Theme is data. `variants: dict[VariantKey, Palette]` with `Palette` a frozen dataclass (bubble in/out, background, quote bar, time text, separator pill). It can optionally load from `palette.toml` next to the module, so `chatsocr inspect calibrate` can regenerate the colours from labelled fixtures without editing code.
- Prefer **composition over mixins.** The engine ships parameterised components such as `ColorBubbleSegmenter(palette, min_radius_dp=…)`, `CornerTimeFinder(regex=…)`, `QuoteBarFinder(color=…)` and `IconMatcher(templates=…)`. A plugin mostly configures them.
- Allow a dedicated class registered for a single `VariantKey`. The registry resolves the most specific match.
- Add a `ui_epoch` / `min_app_version` field to variants. App redesigns (e.g. a new WhatsApp bubble colour) become new variants instead of `if` branches.

## 4. Critique of `crop_next_message`, and the engine/plugin split

**Strengths of the per-message iterative interface:** a simple mental model, small units of work, and it matches how a human reads.

**Weaknesses:**
1. **Error propagation.** One wrong `last_message_bottom_px` misaligns everything that follows on that screen, and nothing global can correct it.
2. **Many decisions need whole-screen context.**
   - Partially cut messages at the top and bottom edges.
   - The floating sticky date pill and the "scroll-to-bottom" button, which *overlay* content and must be excluded from dedup.
   - Status bar, header and composer chrome.
   - Grouping of consecutive bubbles (tails appear only on the first bubble of a group).
   - Unread-messages banners.
3. **Not every element is a message.** Date separators, system lines and Hinge's comment-on-prompt cards don't fit an `UnparsedMessage` return type.
4. **ML detectors and connected-component CV return all boxes at once,** so an iterative API forces an awkward adapter.
5. **Testing is easier against a whole-screen list.** Golden tests compare the full element list with the annotation (IoU matching), which is cleaner than stepping through calls.

**Recommended:**
- The single engine-facing contract is `parse_screen(ctx) -> ScreenLayout`.
- `StepwisePlugin` implements it as a template method: `find_chrome → find_blocks → classify_block → find_parts → postprocess`. Every step has a generic default, and plugins override 1–3 of them.
- `SequentialPlugin(StepwisePlugin)` implements `find_blocks` by looping over the user's `crop_next_element(ctx, y) -> RawElement | None`. Authors who think that way can still write that way.

**Responsibility split:**

| Engine (never touched by plugin authors) | Plugin |
|---|---|
| Loading images, EXIF/mtime reference time, colour normalisation, `ScreenContext`, `scale` | Chrome regions (status bar, header, composer), content area |
| Discovery, availability checks, batch detection and the decision policy | `detect` with evidence |
| OCR (Hebrew+English), Unicode NFC, bidi to logical order, stripping LRM/RLM | Element detection and classification (message, date separator, system, unread banner, overlay, app-specific) |
| Emoji recognition, using the emoji glyph set the plugin declares (Apple, Google, WhatsApp, …) | Side (in/out/centre), sender-name box (groups), avatar box |
| Stitching overlapping screenshots, dedup, stable IDs, append mode | Sub-box roles: body, time, ticks, reply quote (sender, text, thumbnail), media (kind, thumbnail, duration), link preview, reactions, edited/forwarded labels |
| Resolving date labels (weekday, "Yesterday") against the reference time | `parse_date_label(text) -> DateLabel` (absolute, weekday or relative), `parse_time(text)` |
| Reply resolution: fuzzy match of quote text plus perceptual hash of the thumbnail | `classify_media(text, icon) -> MediaKind`, system-message patterns |
| Cropping and saving media, JSON (source of truth), compact text view, aggregating flags | Per-element confidence and flags (e.g. `quote_truncated`) |

Plugins may call `ctx.ocr()` during layout when the text decides the layout, for example to tell a system line from a bubble. The engine OCRs every remaining text role afterwards and calls the plugin's interpretation hooks.

## 5. API versioning, typing and dependencies

**Recommendations:**
- **One public module:** `chatsocr.plugin_api` (re-exports plus a `py.typed` marker). Everything else is private.
  - Plugins declare `api_version = 1`, the major version. The engine rejects other majors with a clear message.
  - Minor versions add only optional fields with defaults.
  - Removals go through `DeprecationWarning` for at least one minor release. The conformance kit runs with `-W error::DeprecationWarning` so authors notice early.
- **Typing:**
  - ABCs for `ChatPlugin` and `StepwisePlugin`, because they carry default implementations.
  - `typing.Protocol` for the small component interfaces, such as segmenters.
  - Frozen `dataclass(slots=True, kw_only=True)` for the IR: fast, no dependencies.
  - pydantic v2 (2.14.4 on PyPI) only for the *output* models and the generated JSON Schema, which also documents the output for downstream AI agents.
  - The conformance kit runs pyright or mypy in strict mode on the plugin module as an optional check.
- **Optional ML:**
  - `chatsocr[ml]` adds `onnxruntime` (1.32.0, MIT, verified on PyPI).
  - The plugin declares `requires = ("onnxruntime",)` and `models = {"bubbles": "models/bubbles.onnx", sha256, license}`.
  - The engine checks `importlib.util.find_spec` before selection and marks the plugin *unavailable (reason)* if a dependency is missing.
  - Models load lazily via `importlib.resources`. Avoid torch as a runtime dependency.
  - Hooks stay stateless or instance-scoped, following OCRmyPDF's lesson for future multiprocessing.

## 6. Testing

**Precedents (verified):**
- scikit-learn `parametrize_with_checks(estimators, *, legacy=True, expected_failed_checks=None, xfail_strict=None)` generates one pytest case per (estimator, check). `expected_failed_checks` maps checks to known-failure reasons. https://scikit-learn.org/stable/modules/generated/sklearn.utils.estimator_checks.parametrize_with_checks.html
- pandas ships `pandas.tests.extension.base` test classes for third parties to subclass. A pandas issue reports 23 such classes and the friction of maintaining them. https://github.com/pandas-dev/pandas/issues/54420
- *Believed:* SQLAlchemy's dialect compliance suite (`sqlalchemy.testing.suite`) follows the same pattern.
- pytest plugins are registered via `[project.entry-points.pytest11]` and tested with the `pytester` fixture. https://docs.pytest.org/en/stable/how-to/writing_plugins.html
- pytest-regressions 2.11.0 (MIT): `data_regression` and friends, with `--force-regen` and `--regen-all`; baselines are stored via pytest-datadir. https://pytest-regressions.readthedocs.io/en/latest/overview.html
- syrupy 7.0.3 (MIT) is an alternative snapshot library.
- jiwer 4.0.0 (Apache-2.0) provides `cer` and `wer`. rapidfuzz 3.14.6 (MIT) provides fast edit distances. Versions were checked on PyPI. https://jitsi.github.io/jiwer/

**Recommended kit (`chatsocr.testing`, auto-registered as a pytest plugin):**
- **Generic checks via `parametrize_conformance(PluginCls, fixtures=...)`:**
  - metadata is valid
  - `detect` is in [0, 1], deterministic and within its time budget
  - the cross-detection matrix passes
  - `parse_screen` returns a valid IR: bboxes inside the image, elements sorted top to bottom and not overlapping (overlays excepted), parts inside their parent
  - robustness: blank, tiny, rotated and foreign images give an empty layout or `CannotParse`, never a crash
  - no network (sockets blocked)
  - no deprecated API use
  - `expected_failed_checks` is supported, as in scikit-learn
- **Golden fixtures:** `fixtures/<plugin_id>/<case>/screenshot.png`, `expected.json` and `case.toml`, where `case.toml` holds the capture time, participants, locale and per-case tolerances.
- **Annotation format: a simple JSON that mirrors the IR, not COCO.** COCO is flat, has no text and no nested parts. Its bbox is `[x, y, w, h]` (believed). Provide exporters to COCO and Label Studio (Apache-2.0, 1.23.2) for anyone training an optional model. Use bbox `[x0, y0, x1, y1]`, integer pixels, top-left origin, exclusive max. Note that LabelMe is GPL-3.0; that matters only if its code is bundled.
- **Two scoring levels:**
  1. **Layout:** Hungarian matching of elements at IoU ≥ 0.7; kind/role accuracy; part-level IoU.
  2. **Content:** CER on normalised text (≤ 0.03 default), exact match on time/date values, media-kind accuracy, reply-target accuracy, and stitching correctness on multi-screenshot cases.
- **No exact snapshots of OCR text,** because OCR upgrades would break them. Snapshot only deterministic geometry, if anything.
- **Privacy:** prefer real screenshots of scripted conversations between consenting test accounts, in Hebrew, English and mixed text. That gives real rendering with no third-party personal data. Add a `chatsocr fixture redact` tool that blurs avatars and phone numbers. Synthetic HTML renders made with Playwright are useful for unit tests because they come with perfect ground truth, but they don't replace real screenshots. Keep fixtures in Git LFS or a separate repository.

## 7. Tooling for an AI agent writing a plugin

**Precedents and why they work:**
- **yt-dlp:** a tiny surface (`_VALID_URL` matches a URL and acts as detection, `_real_extract` parses). Inline `_TESTS` with expected `info_dict` fields support `md5:`, `re:` and `count:` matchers. A rich helper library (`traverse_obj`, `int_or_none`, `unified_timestamp`). Explicit mandatory-versus-optional field rules, where only `id` and `url`/`formats` are fatal and everything else is non-fatal. https://github.com/yt-dlp/yt-dlp/blob/master/CONTRIBUTING.md
- **Home Assistant:** `python3 -m script.scaffold integration` generates the integration folder and its test folder and runs `hassfest` validation. New integrations must reach the "Bronze tier" of the quality scale, which works as a checklist. https://contributors.home-assistant.io/core/integrations/create/
- **Calibre:** declarative first, plus `--test` and a debug pipeline that dumps intermediate files.
- **AGENTS.md:** a plain Markdown file of agent instructions read by many coding agents. https://agents.md/

**Recommended commands:**
- `chatsocr plugin new --app hinge --platform ios --themes light,dark`: creates the module, `palette.toml`, the fixture dirs, a test file and a checklist.
- `chatsocr inspect probe IMG --at 540,1200` or `--box …`: hex colours, Lab values and ΔE to the palette.
- `chatsocr inspect palette IMG`: colour clusters with area and sample coordinates.
- `chatsocr inspect rows IMG`: y-ranges that aren't background, plus column profiles.
- `chatsocr inspect calibrate`: writes the palette from annotated fixtures.
- `chatsocr detect IMG --explain`: every plugin's score and evidence.
- `chatsocr debug overlay IMG [--expected]`: a PNG with role-coloured boxes and indices, side by side with the ground truth. Agents can look at it; numbers come from the probe tools.
- `chatsocr annotate bootstrap IMG`: a draft `expected.json` from the current plugin, for correction.
- `chatsocr eval --plugin X [--json]`: per-fixture scores, missing/extra/mismatched elements, CER lines and a non-zero exit below threshold.
- **Docs:** `PLUGIN_GUIDE.md` and `AGENTS.md` with the workflow (probe → configure → overlay → eval loop), robustness rules borrowed from yt-dlp, and tiers:
  - **Bronze:** detect, text, time, date separators
  - **Silver:** replies, media placeholders, emoji, both themes
  - **Gold:** reactions and markers
- **One very thoroughly commented reference plugin** (WhatsApp Android) that agents can copy.

---

## Recommended interface sketch

```python
# chatsocr/plugin_api.py
API_VERSION = 1
BBox = tuple[int, int, int, int]            # x0,y0,x1,y1 px, origin top-left, max exclusive
Platform = Literal["android", "ios", "desktop", "web"]; Theme = Literal["light", "dark"]

@dataclass(frozen=True, slots=True)
class VariantKey: app: str; platform: Platform; theme: Theme; ui_epoch: str = "default"

@dataclass(frozen=True, slots=True)
class Detection:
    confidence: float; variant: VariantKey; evidence: tuple[str, ...] = ()

class Role(StrEnum):
    BODY="body"; TIME="time"; STATUS="status"; SENDER_NAME="sender_name"; AVATAR="avatar"
    REPLY_QUOTE="reply_quote"; QUOTE_SENDER="quote_sender"; QUOTE_TEXT="quote_text"
    QUOTE_THUMB="quote_thumb"; MEDIA="media"; CAPTION="caption"; LINK_PREVIEW="link_preview"
    REACTION="reaction"; EDITED="edited"; FORWARDED="forwarded"; EMOJI="emoji"

class Kind(StrEnum):
    MESSAGE="message"; DATE_SEPARATOR="date_separator"; SYSTEM="system"
    UNREAD_BANNER="unread_banner"; OVERLAY="overlay"; APP_SPECIFIC="app_specific"

@dataclass(slots=True, kw_only=True)
class Part:
    role: Role; bbox: BBox; text: str | None = None; value: object = None
    children: list["Part"] = field(default_factory=list); confidence: float = 1.0

@dataclass(slots=True, kw_only=True)
class RawElement:
    kind: Kind; bbox: BBox; side: Literal["in", "out", "center"] | None = None
    parts: list[Part] = field(default_factory=list); media_kind: str | None = None
    cut: Literal["none", "top", "bottom"] = "none"; subtype: str | None = None
    confidence: float = 1.0; flags: list[str] = field(default_factory=list)
    extra: dict[str, object] = field(default_factory=dict)

@dataclass(slots=True, kw_only=True)
class ScreenLayout:
    content_area: BBox; elements: list[RawElement]
    chrome: dict[str, BBox] = field(default_factory=dict); header_title: str | None = None
    flags: list[str] = field(default_factory=list)

class ScreenContext(Protocol):          # engine-provided, lazily cached
    image: np.ndarray; small: np.ndarray; scale: float; luminance: float
    source_path: Path | None; reference_time: datetime; filename_tokens: frozenset[str]
    cv: "CvHelpers"
    def ocr(self, box: BBox, *, langs: Sequence[str] = ("he", "en")) -> "OcrResult": ...

class ChatPlugin(ABC):
    id: ClassVar[str]; api_version: ClassVar[int] = API_VERSION
    variants: ClassVar[Mapping[VariantKey, "Palette"]]
    emoji_style: ClassVar[str] = "apple"; requires: ClassVar[tuple[str, ...]] = ()
    priority: ClassVar[int] = 0
    def __init_subclass__(cls, abstract: bool = False, **kw): ...   # validate metadata only
    @classmethod
    @abstractmethod
    def detect(cls, ctx: ScreenContext) -> Detection | None: ...
    def __init__(self, variant: VariantKey, options: Mapping[str, object] = {}) -> None: ...
    @abstractmethod
    def parse_screen(self, ctx: ScreenContext) -> ScreenLayout: ...
    # interpretation hooks (defaults use engine helpers)
    def parse_time(self, text: str) -> time | None: ...
    def parse_date_label(self, text: str) -> "DateLabel | None": ...   # Absolute|Weekday|Relative
    def classify_media(self, el: RawElement, ctx: ScreenContext) -> str | None: ...

class StepwisePlugin(ChatPlugin, abstract=True):   # template method
    def parse_screen(self, ctx):
        chrome = self.find_chrome(ctx); area = self.content_area(ctx, chrome)
        els = [self.classify_block(ctx, b) for b in self.find_blocks(ctx, area)]
        for e in els: self.find_parts(ctx, e)
        return self.postprocess(ctx, ScreenLayout(content_area=area, elements=els, chrome=chrome))

class SequentialPlugin(StepwisePlugin, abstract=True):  # user's idea as an adapter
    @abstractmethod
    def crop_next_element(self, ctx: ScreenContext, y: int) -> RawElement | None: ...
```

**Engine pipeline:**
1. Load images and build the contexts.
2. Discover plugins, filter by availability and hints, run detection over the batch, and apply the decision policy.
3. Per screen: `parse_screen`, then OCR the text roles, bidi normalisation and emoji matching, then the plugin's interpretation hooks.
4. Stitch and dedup, then resolve dates and replies, then assign IDs (append-aware).
5. Crop media, then write the JSON with flags and the text view.

---

## Risks and open questions

- **Confidence calibration and look-alike variants.** WhatsApp Android dark and iOS dark may be close. The cross-detection matrix only catches what the fixtures cover, so a hierarchical detection step (app, then platform) may be needed.
- **Fragile shared base classes.** A change to `WhatsAppBase` can break sibling variants. CI must run all variants' golden tests on any change, and the eval report should be part of PR review.
- **App redesigns break plugins silently.** The `ui_epoch` variants help, and `detect` must not score an unknown new layout as confident. Fixtures should be dated.
- **OCR or emoji-set upgrades shift text output.** Tolerance thresholds need tuning; it is undecided whether to pin the OCR version in the fixture metadata.
- **Agents misread pixel positions.** The numeric probe tools are essential, not optional. Agent authoring has not been validated yet; recommend a pilot in which an agent writes the Telegram plugin from fixtures alone.
- **Unclear boundary between steps:** stitching needs plugin-marked overlays and cut flags. Who owns "same message seen twice with a different cut"? Recommended: the engine, using `cut` plus text and thumbnail hashes.
- **Hinge's structure** (comments on profile photos or prompts, match banners): *believed* to need `Kind.APP_SPECIFIC` + `subtype`. This needs real screenshots to confirm.
- **Plugin trust:** decide whether non-built-in plugins should require an explicit opt-in (Docling style) or only a documented warning (yt-dlp style).
- **Fixture logistics:** acquiring consenting real screenshots for 3 apps × 3 platforms × 2 themes × Hebrew/English, and deciding where the large binary fixtures live.
- **Unverified items:** the exact Samsung filename pattern and extension; the COCO bbox convention (the source page didn't render); which emoji glyph sets each app uses on each platform; SQLAlchemy's suite as a precedent; the Home Assistant tier names beyond "Bronze".
