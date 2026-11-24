# Background research (2026-11-24)

These reports were written by research agents to inform the [requirements](../requirements.md), the [high-level design](../high-level-design.md) and the [output format](../output-format.md).

- **They are not normative.** Where they disagree with those three documents, the documents win.
- **Claims are tagged by how well they were checked:** verified with a source or a local test, believed, or unverified. Re-check unverified claims, especially about app visuals, against real screenshots before relying on them.

| Report | Topic | Most useful for |
|---|---|---|
| [ocr-engines.md](ocr-engines.md) | Local OCR engines with Hebrew support, bidi pitfalls, preprocessing, backend interface | OCR subsystem |
| [ocr-spike.md](ocr-spike.md) | **Hands-on test:** Tesseract and a Hebrew ONNX model on synthetic WhatsApp-style bubbles: CER tables, install notes, emoji masking | OCR and emoji design, risks |
| [chat-ui-anatomy.md](chat-ui-anatomy.md) | What WhatsApp, Telegram and Hinge look like on Android, iOS and desktop, light and dark: colors, date-label rules, RTL mirroring, detection signals | **Plugin authors** |
| [cv-techniques.md](cv-techniques.md) | Bubble segmentation, quote, time and emoji detection, emoji template matching (with a local accuracy test), stitching, time anchors, library stack | Engine CV and the CV helper toolkit |
| [prior-art.md](prior-art.md) | Existing chat-screenshot parsers, stitching tools, UI-understanding research, datasets, synthetic generators, licence warnings | Design lessons, test data |
| [plugin-architecture.md](plugin-architecture.md) | Plugin discovery, detection protocol, variant modelling, critique of the per-message interface, conformance kit, AI-agent authoring tools | Plugin API |
| [output-format-research.md](output-format-research.md) | WhatsApp, Telegram and Matrix export formats, text grammar, token costs, stable IDs, time semantics, bidi in output | Output format |

The experiment scripts (synthetic screenshot generator, OCR evaluation harness, emoji matching tests) were run in a temporary workspace and were not committed.
