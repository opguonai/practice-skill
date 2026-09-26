# practice — opencode skill

An [opencode](https://opencode.ai) skill for building **categorized practice /
exam problem-set PDFs** from a textbook chapter, course notes, or a scanned PDF.

Typical triggers: 题集 / 刷题 / 习题 / 考研真题 / 讲义 / 难度分级 / 答案横线 /
目录 / LaTeX / xelatex / ctex — especially Chinese math.

## What it does

Given a source chapter (often a scanned Chinese math PDF with **no text layer**),
the skill guides the agent to:

1. **Recon** — detect tools, and whether the PDF has a usable text layer.
2. **Read** — render pages to images (`pdftoppm`) and OCR them with the Windows
   built-in OCR (`zh-Hans-CN`) when `tesseract` is unavailable.
3. **Source & tag** — transcribe genuinely-sourced problems (with citations) and
   author the rest; tag each by **difficulty** (★ / ★★ / ★★★) and knowledge point.
4. **Build** — a `ctexart` + `xelatex` document with colored difficulty stars,
   per-difficulty answer lines, colored section headings, page headers, and a
   table of contents.
5. **Self-check twice** — mathematical well-posedness, then rendered-page layout.

Length is content-driven: the document grows or shrinks with the number of
problems, never padded to an arbitrary page count (unless the user asks).

## Install

Copy the folder to your opencode skills directory:

```
~/.config/opencode/skill/practice/SKILL.md      # macOS / Linux
%USERPROFILE%\.config\opencode\skill\practice\SKILL.md   # Windows
```

Then **restart opencode** (skills are loaded at startup).

## Usage

Just ask, e.g.:

> 把教材第 2 章做成一份难度分级的刷题 PDF，证明题留横线，加目录。

The skill activates automatically from its description.

## License

MIT — see [LICENSE](LICENSE).
