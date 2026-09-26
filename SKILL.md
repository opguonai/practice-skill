---
name: practice
description: Use when the user wants a categorized practice / exam problem-set PDF built from a textbook chapter, course notes, or scanned PDF — keywords like 题集, 刷题, 习题, 考研真题, 讲义, 难度分级, 答案横线, 目录, LaTeX, xelatex, ctex. Covers OCR of scanned Chinese math PDFs (no text layer), problem sourcing/transcription, difficulty + knowledge-point tagging, the LaTeX build, and correctness/layout self-checks. Use ONLY for building a multi-problem document, NOT for solving a single problem or chat Q&A.
---

# Building a categorized problem-set PDF (esp. Chinese math)

This skill reproduces the workflow used to turn a scanned 数学分析 chapter
("第 2 章 极限理论") plus external sources (考研数学真题, 裴礼文《典型问题与方法》)
into a polished, difficulty-graded, answer-lined PDF with a table of contents.

**Length is content-driven.** Do NOT target a fixed page count. Let the document
be exactly as long as the number of problems and their answer space require.
Only compress/expand toward a specific page count if the user explicitly asks.

Follow the phases in order. Do not skip the self-check phase — several fixes
below were only found by re-reading rendered pages.

## Phase 0 — Recon (always first)

1. Confirm the source file and that it exists.
2. Detect available tools (Windows/MiKTeX is the common stack here):

```powershell
$tools = "pdftotext","pdfinfo","pdftoppm","pdfimages","xelatex","pdflatex","latexmk","magick","tesseract","python","git"
foreach ($t in $tools) { $c = Get-Command $t -ErrorAction SilentlyContinue; if ($c) { "$t -> $($c.Source)" } else { "$t -> NOT FOUND" } }
```

3. **Decide if the PDF has a usable text layer.** Scanned Chinese math books
   usually do NOT. Test:

```powershell
& pdfinfo "$src"            # page count
& pdftotext -f 1 -l 5 "$src" out.txt   # ~empty or mojibake => scanned / no ToUnicode
```

If the text is garbled (CJK maps to Latin junk) or nearly empty, you MUST go
through page images (Phase 1). Do not trust garbled extraction for equations.

## Phase 1 — Read the source via page images

Render pages to PNG and read them as images (the model reads images, not this
PDF format):

```powershell
$dir = "$env:TEMP\opencode\pl"; New-Item -ItemType Directory -Force -Path $dir | Out-Null
& pdftoppm -png -r 150 -f 1 -l 22 "$src" "$dir\toc"     # 150 dpi for TOC/structure
& pdftoppm -png -r 200 -f 23 -l 100 "$src" "$dir\c"     # 200 dpi for accurate reading of problems
```

### Bulk OCR for mapping (Windows built-in OCR, zh-Hans-CN)

Use this to *locate* content cheaply (section/test headers, page ranges). OCR
mangles formulas, so use it for structure, then read specific pages as images
before transcribing.

Save this as `ocr.ps1`:

```powershell
[CmdletBinding()]
param([Parameter(Mandatory=$true)][string[]]$Files,[Parameter(Mandatory=$true)][string]$Out)
$ErrorActionPreference='Stop'
Add-Type -AssemblyName System.Runtime.WindowsRuntime | Out-Null
$asTaskGeneric = ([System.WindowsRuntimeSystemExtensions].GetMethods() | Where-Object {
  $_.Name -eq 'AsTask' -and $_.GetParameters().Count -eq 1 -and $_.GetParameters()[0].ParameterType.Name -eq 'IAsyncOperation`1' })[0]
function Await($WinRtTask,$ResultType){ $m=$asTaskGeneric.MakeGenericMethod($ResultType); $nt=$m.Invoke($null,@($WinRtTask)); $nt.Wait(-1)|Out-Null; $nt.Result }
[Windows.Media.Ocr.OcrEngine,Windows.Foundation,ContentType=WindowsRuntime] | Out-Null
[Windows.Graphics.Imaging.BitmapDecoder,Windows.Foundation,ContentType=WindowsRuntime] | Out-Null
[Windows.Storage.StorageFile,Windows.Foundation,ContentType=WindowsRuntime] | Out-Null
[Windows.Globalization.Language,Windows.Foundation,ContentType=WindowsRuntime] | Out-Null
$engine = [Windows.Media.Ocr.OcrEngine]::TryCreateFromLanguage((New-Object Windows.Globalization.Language 'zh-Hans-CN'))
$sb = New-Object System.Text.StringBuilder
foreach ($f in $Files){
  $path=(Resolve-Path -LiteralPath $f).Path
  $file=Await ([Windows.Storage.StorageFile]::GetFileFromPathAsync($path)) ([Windows.Storage.StorageFile])
  $s=Await ($file.OpenAsync([Windows.Storage.FileAccessMode]::Read)) ([Windows.Storage.Streams.IRandomAccessStream])
  $d=Await ([Windows.Graphics.Imaging.BitmapDecoder]::CreateAsync($s)) ([Windows.Graphics.Imaging.BitmapDecoder])
  $b=Await ($d.GetSoftwareBitmapAsync()) ([Windows.Graphics.Imaging.SoftwareBitmap])
  $r=Await ($engine.RecognizeAsync($b)) ([Windows.Media.Ocr.OcrResult])
  [void]$sb.AppendLine("===== FILE: $([IO.Path]::GetFileName($path)) ====="); [void]$sb.AppendLine($r.Text)
  $s.Dispose(); $b.Dispose()
}
[IO.File]::WriteAllText($Out,$sb.ToString(),(New-Object System.Text.UTF8Encoding($false)))
```

Run it on already-rendered PNGs (call it in-session, NOT via `-File`, because
`-File` does not bind array parameters):

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
$pngs = Get-ChildItem $dir -Filter "*.png" | Sort-Object Name | ForEach-Object { $_.FullName }
& "$env:TEMP\opencode\ocr.ps1" -Files $pngs -Out "$env:TEMP\opencode\ocr.txt"
```

Then `grep` the OCR text for section headers ("§", "练习", "习题") to map pages,
and **read the relevant pages as images at 200 dpi** to transcribe accurately.

## Phase 2 — Check local resources

Search the user's disks for the referenced books (they often have them):

```powershell
Get-ChildItem "C:\Users\<user>\Desktop","C:\Users\<user>\Downloads","C:\Users\<user>\Documents" -Recurse -File -Include *.pdf,*.djvu,*.epub |
  Where-Object { $_.Name -match '裴|数学分析|微积分|极限|考研|高等数学' } | Select FullName,Length
```

If a book is only available online (e.g. a GitCode/Gitee repo), fetch it via the
host's API `contents` endpoint to get the `download_url` blob URL, then
`Invoke-WebRequest -OutFile`.

## Phase 3 — Source & tag problems

- **Transcribe** genuinely-sourced problems from the pages you actually read
  (cite the book + section code, e.g. `【裴 1.2.3】`). Reproduce only reasonable
  excerpts for personal study; do not wholesale-copy a whole book.
- **Author** the remainder at the same level (考研数学一 + 数学分析).
- **Tag every problem** with (a) a difficulty tier and (b) knowledge points.
  Keep a running list while drafting.
- **Verify well-posedness** before typesetting (see self-check rules).

## Phase 4 — LaTeX build (ctexart + xelatex)

Use `xelatex`. Key working macros (copy this preamble):

```latex
\documentclass[UTF8,a4paper,12pt]{ctexart}
\usepackage{amsmath,amssymb,amsthm,amsfonts}
\usepackage[margin=2.3cm,top=2.7cm,bottom=2.5cm]{geometry}
\usepackage{enumitem}
\usepackage{pgffor}          % REQUIRED: answer-line loop
\usepackage{xcolor}
\usepackage{titlesec}
\usepackage{fancyhdr}
\usepackage{hyperref}

\definecolor{ansline}{gray}{0.55}
\definecolor{secblue}{RGB}{28,72,140}
\definecolor{subteal}{RGB}{18,108,108}

\titleformat{\section}{\normalfont\Large\bfseries\color{secblue}}{}{0pt}{}[\vspace{0.12em}{\color{secblue}\hrule height 1.2pt}]
\titleformat{\subsection}{\normalfont\large\bfseries\color{subteal}}{}{0pt}{}
\pagestyle{fancy}\fancyhf{}\renewcommand{\headrulewidth}{0.4pt}
\fancyhead[L]{\small\color{secblue}\textbf{第 2 章\quad 极限理论}}
\fancyhead[R]{\small\color{secblue}\textbf{\thepage}}

\newcounter{pb}
\newcommand{\sone}{\textcolor{green!55!black}{$\bigstar$}}
\newcommand{\stwo}{\textcolor{orange}{$\bigstar\bigstar$}}
\newcommand{\sthree}{\textcolor{red}{$\bigstar\bigstar\bigstar$}}

% Answer-line counts per difficulty. Choose by how much work each tier needs
% (e.g. 4/7/11). Do NOT pick these to force a page count.
\newcommand{\LA}{4}\newcommand{\LB}{7}\newcommand{\LC}{11}
\newcommand{\answerlines}[1]{%
  \par\vspace{3pt}%
  \foreach \i in {1,...,#1}{\noindent{\color{ansline}\rule{\linewidth}{0.5pt}}\par\vspace{6pt}}%
  \vspace{1pt}%
}
\newcommand{\prob}[2]{%
  \par\medskip\noindent\refstepcounter{pb}\textbf{#1}\ \textbf{\thepb.}\ #2\par
  \ifx#1\sone\answerlines{\LA}\else
  \ifx#1\stwo\answerlines{\LB}\else
  \answerlines{\LC}\fi\fi}
```

Use `\prob{\stwo}{...}` per problem. For a TOC, keep headings unnumbered but
listed:

```latex
\setcounter{secnumdepth}{0}   % hide numbers
\setcounter{tocdepth}{2}      % include subsections
...
\clearpage \tableofcontents \clearpage
```

Compile **twice** (TOC/refs need a second pass):

```powershell
& xelatex -interaction=nonstopmode -halt-on-error "book.tex"
& xelatex -interaction=nonstopmode -halt-on-error "book.tex"
& pdfinfo book.pdf | Select-String Pages   # verify
```

## Length control (only when a page count is required)

By default the length is whatever the content produces. If the user DOES name a
target, tune with `pages ≈ base + total_lines × pitch / text_height`. Measure
once at a baseline, then scale by changing only the tier line counts (`\LA \LB
\LC`) or the per-line `\vspace`. Reference (A4 / 12pt / 2.3cm margins, ~22pt
pitch): ~1800 answer lines ≈ 81 pages, base ≈ 23 pages. Never sacrifice problem
count to hit a page goal — adjust whitespace instead.

## Self-check (do this twice, both passes)

**Pass 1 — mathematical correctness.** Re-read every problem; fix:
- under-determined "求 a,b" (only one of two constants determined);
- "no solution" piecewise-continuity set-ups;
- recurrence problems whose stated range diverges (e.g. `x₁+xₙ²/2` needs
  `0<x₁≤½`, not `<1`);
- false statements asked to be "proved"; missing initial conditions.

**Pass 2 — layout/output.** Render pages to images and check:
- numbering is continuous 1..N and any section page ranges stated in the
  instruction box MATCH reality (this was a real bug);
- no dropped content (verify with the `\foreach` test below);
- answer lines appear at the right count per tier;
- no blank pages / orphaned headings.

### Verifying no content loss (the loop bug)

A `\loop ... \repeat` body containing `\par` **silently drops problems**. Prove
your macro first with a minimal file (10 labeled items — all 10 must appear):

```latex
\documentclass[UTF8]{ctexart}
\usepackage{xcolor}\usepackage{pgffor}
\newcommand{\answerlines}[1]{\par\vspace{3pt}\foreach \i in {1,...,#1}{\noindent{\color{gray}\rule{\linewidth}{0.5pt}}\par\vspace{13pt}}\vspace{1pt}}
\newcommand{\QT}[1]{\par\medskip\noindent\textbf{PROBLEM-#1}\par\answerlines{4}}
\begin{document}\QT{A}\QT{B}\QT{C}\QT{D}\QT{E}\QT{F}\QT{G}\QT{H}\QT{I}\QT{J}\end{document}
```

## Critical pitfalls (all encountered for real)

1. `\P` is a built-in LaTeX command (¶). Naming a macro `\P` → "Command \P
   already defined". Use `\prob`, `\QT`, etc. Do NOT blanket-replace `\P`
   (it corrupts `\par`, `\pi`, `\pm`).
2. `\loop`/`\repeat` with `\par` inside drops content silently. Use `\foreach`
   from `pgffor`.
3. `\hrule` followed by `\\` (e.g. in a title) → "There's no line here to end."
   Use `\rule{\linewidth}{h}` or `\rule{0.7\linewidth}{1.2pt}` instead.
4. Scanned CJK PDFs: `pdftotext` gives mojibake; render to images + read them.
   Windows OCR (`zh-Hans-CN`) is available when tesseract is not.
5. `powershell -File script.ps1 -Files $array` does not bind arrays. Run the
   script in-session (`& script.ps1 -Files $pngs`).
6. `Remove-Item` may be denied by policy — use fresh output directories.
7. Keep the deliverable current: recompile and copy to the requested location,
   then report the actual page count and problem count.

## Deliverable conventions

- Final PDF copied to the requested path (often the Desktop) with a descriptive
  Chinese filename.
- Report: page count, how many problems, difficulty tiers, answer-line rules,
  and any caveats (not verbatim from copyrighted books, no solutions included).
- Keep the `.tex` source in the build dir so it can be re-tuned later.
