# Using This Repo as a Layout / Print Reference for Visual Layout PDF Builder

This repository is a **Homebrewery-flavored layout + print experiment**. It is useful as a **reference implementation** for the *last stage* of your other project – the **Visual Layout PDF Builder** – where HTML content is turned into a print/PDF-friendly layout.

It is **not** intended to replace that project, only to:

- Demonstrate patterns for **screen vs. print views**.
- Show how to structure a **page-based viewer**.
- Provide CSS ideas for **page sizing, margins, and hard page breaks**.

The notes below assume the Visual Layout PDF Builder has the architecture you described (App, Controls, MarkdownEditor, Preview, etc.).

---

## 1. What This Repo Provides Conceptually

In this repo, the important pieces are:

- **`The Homebrewery V3/index.html`**
  - A small Vue app that:
    - Holds an array `pageDatas: string[]` of **already-laid-out page HTML**.
    - Renders one page at a time, with controls for **paging and zooming**.
    - Exposes a **print-only container** (with `media="print"` CSS) that hides the UI on print.

- **`example_balanced.html` / `.pdf`**
  - Example of **ready-to-print, page-oriented HTML** in a D&D/Book style.

From a design point of view, the key ideas are:

- **Separation of concerns**:
  - Layout/HTML **generation** happens elsewhere (not in the viewer).
  - The viewer just shows an array of finished pages.

- **Print-aware structure**:
  - UI elements are marked as **"do not print"**.
  - A dedicated container holds only the **printable content**.

- **Page-by-page mindset**:
  - Content is already split into **pages**, which map 1:1 to PDF pages.

These concepts map nicely onto the Visual Layout PDF Builder’s **Preview + Print/PDF** stage.

---

## 2. Summary of the Visual Layout PDF Builder

In your other project (high level):

- **App state** keeps track of:
  - `chapters[]: { id, title, content (Markdown), htmlContent (HTML) }`
  - `activeChapterId`
  - `activeStyle` (Book, Magazine, Academic, etc.)
  - Split-pane state (`sidebarWidth`, `isResizing`).

- **Left Pane – `<Controls />`**
  - File Import and Markdown Editor.
  - Smart tools (Clean, Smart Type).
  - Style selector.
  - **"Build Chapter Layout"** button that ultimately calls an `onGenerate()`-like handler.

- **Right Pane – `<Preview />`**
  - Header with **Print / Save PDF** button.
  - Loading / Empty views.
  - Main **printable canvas** (e.g. `id="printable-content"`) that loops through chapters and renders their built HTML.

This repo’s concepts are most relevant to the **Preview + Print/PDF path**:

- How to store **print-ready HTML**.
- How to **wrap** it in containers suitable for printing.
- How to define CSS for **print-only content, page size, and page breaks**.

---

## 3. Conceptual Layout-Prep Layer

To bridge between your **Markdown/AI pipeline** and this repo’s **page-based viewer ideas**, it helps to define a small conceptual layer:

> A **Layout-Prep** module that accepts HTML (from AI or generic sources) and returns **print-ready HTML**, optionally split into pages.

You can think of it like this, in your React/TypeScript project:

- **Input**:
  - A chapter’s **HTML** (`htmlContent`), which might come from:
    - Converting Markdown directly to HTML.
    - Gemini / AI output using Tailwind-based templates.
    - Imported generic HTML.

- **Output**:
  - One of:
    - **Per-chapter wrapper HTML** that the browser’s print engine paginates for you, or
    - An **array of page HTML strings** (similar to `pageDatas` in this repo) if you want precise control.

- **Options**:
  - Page size (`letter`, `A4`).
  - Margins.
  - Strategy for chapter/page breaks.
  - Style flavor (Book, Magazine, Academic, etc.).

In practical terms, the **Layout-Prep** layer runs **between** your `onGenerate()` handler and the `Preview` component.

---

## 4. Mapping This Repo’s Ideas Onto Your Components

### 4.1 `chapters[].htmlContent` and the Build Pipeline

In your **App/Controls** flow:

1. **User edits Markdown** in `<MarkdownEditor />` for the active chapter.
2. **User clicks "Build Chapter Layout"**.
3. Your `onGenerate()` handler does roughly:
   - Convert Markdown to a structured representation (if needed).
   - Call Gemini (or another engine) to generate a Tailwind-based layout **HTML**.
   - Pass that HTML through a **Layout-Prep** helper inspired by this repo’s patterns.
   - Update `chapters[activeChapterId].htmlContent` with the **print-ready HTML**.

The Layout-Prep helper is where you can:

- Inject **page wrappers** (e.g. `<div class="page">...</div>` per logical page).
- Add **chapter separators** and ensure each chapter ends with a page break.
- Normalize CSS hooks for printing (e.g. `.page`, `.page-break`, `.print` containers).

This repo shows how to **organize per-page HTML** and then switch between pages; your React app can instead **render them sequentially** for a long scroll, but still use the same underlying idea of a per-page wrapper.

### 4.2 `<Preview />` and Printable Canvas

In your **`<Preview />`** component, there are a few core ideas to borrow from this repo:

1. **Dedicated print container**
   
   - In this repo, there’s a `div` with `v-html="pageContent"` and a parallel `div` for print-only.
   - In your React app, the equivalent is `#printable-content` or a `div` marked as the **printable canvas**.

2. **Hide UI, show book on print**

   - Mark all non-print UI (toolbars, sidebars, editor) with a class like `.noprint`.
   - Wrap the main book/preview area in a `.print` (or similar) container.
   - Use `@media print` to hide `.noprint` and show/tune `.print`.

3. **Sequential chapter rendering**

   - Loop through `chapters` and render their `htmlContent` into the canvas, for example:
     - One wrapper per chapter.
     - Optional **hard page break** at the end of each chapter.

This lets the **Preview** behave like a scrollable book on screen, while the **print CSS** ensures correct page breaks and hides the app chrome when the user triggers `window.print()` from the Preview header.

---

## 5. CSS Patterns to Reuse

This repo’s `index.html` + CSS illustrate a few key patterns:

### 5.1 Screen vs. Print

Use **utility classes + `@media print`**:

- **Classes**:
  - `.noprint` – all app chrome/components that should never appear in the PDF.
  - `.print` – main book or document content.

- **CSS** (in your main React app’s styles, not necessarily copied verbatim):
  - In print mode, hide `.noprint` and show `.print` (or tweak their display).

This matches your requirement: when clicking **Print/Save PDF**, you want to show only the document, not the UI.

### 5.2 Page Size and Margins

For print/PDF correctness (Letter/A4, margins, etc.), use `@page` and print-specific rules in your React project’s global CSS:

- Set **page size** to match your intended output (e.g. US Letter for book-like layouts).
- Set **margins** so the visible content matches your design system.

This is aligned with how the examples in this repo assume a fixed page canvas.

### 5.3 Hard Page Breaks

Your Visual Layout PDF Builder wants a **hard page break after every chapter**:

- Define a reusable class such as `.page-break` or `.chapter-break`.
- Use `break-after: page` (and legacy `page-break-after`) in print styles.
- Have the Layout-Prep helper append a **break element** at the end of each chapter.

This mirrors the conceptual per-page thinking in this repo’s `pageDatas`, just applied to your React components and CSS.

---

## 6. Supporting Multiple Input Flavors

You can think of each content source as a different **flavor** that ultimately passes through the same Layout-Prep API:

- **Homebrewery-style HTML**
  - This repo as a reference for D&D/book flavor, background images, and typography.

- **Generic HTML**
  - Imported HTML files or non-AI content.
  - Layout-Prep can:
    - Wrap content into a fixed-width page container.
    - Insert breaks on `h1` or other logical markers.

- **AI / Tailwind HTML**
  - Output from Gemini based on your **Book / Magazine / Academic / Newsletter** templates.
  - Layout-Prep can:
    - Normalize `max-width`, fonts, and spacing to avoid overflow in print.
    - Insert page or chapter breaks.

All of these flavors can write to the same `chapters[].htmlContent` field, keeping the rest of your UI unchanged.

---

## 7. How to Use This Repo In Practice

You do **not** need to directly integrate this repo into the Visual Layout PDF Builder. Instead, treat it as a **design sandbox** and reference:

1. **Open** `The Homebrewery V3/index.html` and observe:
   - How the page viewer stores and swaps **per-page HTML**.
   - How the print-only container is structured.

2. **Inspect** the related CSS for:
   - Print vs. screen behavior.
   - Page sizing assumptions.

3. **Translate** the relevant ideas into your React app by:
   - Adding `.noprint` / `.print` classes and print media queries.
   - Implementing a **Layout-Prep helper** that writes into `chapters[].htmlContent`.
   - Ensuring `<Preview />` renders `htmlContent` inside a dedicated print container.

If desired, you can evolve this repo’s HTML/CSS files as a **style playground**, experimenting with new print layouts before porting the ideas into your main Visual Layout PDF Builder.
