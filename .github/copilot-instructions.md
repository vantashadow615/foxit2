PART A. MASTER BRIEF (goes in copilot-instructions.md)
A1. Mission
Build Foxit 2.0, a fast, polished, offline Windows desktop PDF editor that includes every everyday tool found in Foxit PDF Editor, plus a Tender Toolkit for people who assemble large bid documents (merge, separators, letterhead, stamps and signatures, serialization, blank-finder, consistency checks, pre-submission audit). The interface must look and feel like a premium commercial product.
Primary user: a director who prepares tender packs of 100 to 300 pages by hand today, mixing scanned certificates, filled forms, brochures and the tender document itself.
A2. Non-negotiable rules
1. Offline and private. No network calls, no telemetry, no cloud. All documents, IDs, PINs and certificates stay on the machine.
2. Never damage the original. Opening a file never modifies it. Saving writes a new file or an explicit overwrite after confirmation. Autosave goes to a recovery folder, never over the source.
3. Non-destructive editing with full undo/redo (command pattern) until the file is saved.
4. Never invent content. The app fills, stamps and signs only with data and images the user supplied. No generated signatures, stamps or certificates.
5. Performance: first page of a 135-page, 12 MB scanned PDF visible in under 2 seconds; smooth scrolling at 60 fps on a mid-range laptop; memory capped by an LRU page cache (default 300 MB).
6. TypeScript strict mode, no any, no unexplained dependencies. Every dependency must be named in docs/DEPENDENCIES.md with its licence.
7. Accessibility: keyboard-operable everywhere, WCAG AA contrast, visible focus, screen-reader labels.
8. Ask before deviating from this brief. If something is impossible, say so in the pull-request notes and propose the closest alternative; do not silently drop it.
A3. Technology stack
Layer
Choice
Shell
Electron (current stable) with electron-vite, contextIsolation: true, no Node in the renderer
UI
React 18, TypeScript (strict), Tailwind CSS, Radix UI primitives, lucide-react icons, Inter font bundled locally
State
Zustand (one store per document session plus an app store)
PDF rendering
pdfjs-dist in Web Workers (canvas pages, text layer, thumbnails)
PDF writing
pdf-lib with @pdf-lib/fontkit (permissive licence)
Structural ops, encryption, repair
bundled qpdf binary (Apache 2.0), run as a child process
Images
sharp for scan clean-up, resize and recompress; pixelmatch for visual compare
OCR
tesseract.js with English and Swahili language data bundled for offline use
Digital signatures
@signpdf/signpdf and node-forge (PKCS#12 / .pfx)
Storage
SQLite (better-sqlite3) for the Company Vault, recent files and settings; Electron safeStorage for sensitive fields (ID numbers, PINs)
Office conversion
Detect an installed LibreOffice (soffice) for Word/Excel/PowerPoint to PDF; do not bundle it
Testing
Vitest (unit), Playwright for Electron (end-to-end), golden-file PDF fixtures
Packaging
electron-builder, Windows NSIS installer, auto-update disabled
Licensing rule: avoid AGPL libraries (MuPDF, PyMuPDF, Ghostscript) in the core so the product can be sold later. They may only appear as optional, user-installed modules behind a clearly defined interface.
A4. Repository structure
foxit2/
  apps/desktop/
    src/main/            # Electron main: windows, IPC, file system, qpdf, SQLite, OCR scheduling
    src/preload/         # typed, minimal IPC bridge
    src/renderer/        # React app
      app/               # shell, ribbon, panels, tabs, command palette, themes
      features/          # viewer, pages, annotate, edit, forms, security, convert, ocr, compare, tender
      design-system/     # tokens, primitives, icons
  packages/
    pdf-engine/          # DocumentSession, command stack, page model, save pipeline (pure TS, no UI)
    pdf-render/          # pdf.js worker wrappers, page cache, thumbnails, tiles
    tender-kit/          # blank-finder, consistency checker, audit rules, pack builder (pure TS)
    vault/               # SQLite schema and access layer
  docs/                  # ARCHITECTURE.md, DEPENDENCIES.md, SHORTCUTS.md, TESTING.md
  fixtures/pdfs/         # sample PDFs: scanned, forms, encrypted, 300-page, corrupt
A5. Core architecture
• Process model: the main process owns files, qpdf, SQLite and OCR jobs. The renderer owns UI only. Heavy work (render, OCR, compress, merge) runs in worker threads or utilityProcess, with progress events and cancellation for every long job.
• DocumentSession (in pdf-engine): holds the loaded document bytes, a page model (page id, rotation, crop box, source page) and an operation log. Every user action is a Command with do() and undo(). Examples: RotatePages, DeletePages, InsertPages, AddTextOverlay, AddAnnotation, ApplyRedaction, StampPages.
• Overlay model: text, images, shapes, stamps, signatures and annotations are stored as objects in PDF point coordinates (origin bottom-left) tied to a page id, and are converted to screen coordinates only at render time. They stay editable until saved.
• Save pipeline: (1) rebuild the page order, (2) write real PDF annotations for annotation types that PDF supports (highlight, underline, strikeout, note, free text, ink, line, rectangle, ellipse, stamp), (3) flatten overlays marked “flatten on save”, (4) write with pdf-lib, (5) optionally pass through qpdf for object-stream compression and linearization, (6) verify the output reopens and the page count matches before replacing anything.
• Viewer: virtualized page list (render only the visible pages plus two on each side), tile rendering at high zoom, thumbnails generated lazily in a worker, an LRU bitmap cache, and a text layer for selection, search and copy.
• Plugin seam: define a ToolPlugin interface (id, ribbonItems, commands, panel?) and register every feature through it. Reserve an AI Assistant panel slot (disabled, hidden by default) for a later phase. Do not implement any AI now.
• IPC: one typed channel map in packages/shared, validated with Zod on both sides. No wildcard channels.
A6. UI and UX specification
The look is clean, calm and expensive: neutral greys with one accent colour, defined as design tokens (colour, spacing on an 8-pt grid, radius, shadow, type scale). Light and dark themes, switchable and remembered.
Layout
1. Title bar with document tabs (multi-document, drag to reorder, close buttons, unsaved dot).
2. Quick-access bar: Open, Save, Undo, Redo, Print, plus a search-the-commands box (Ctrl+K opens the command palette).
3. Ribbon tabs: Home, Edit, Comment, Organize, Forms, Protect, Convert, Tender. Tool groups with icon + label, collapsible; contextual tabs appear when something is selected (for example an image tab).
4. Left rail (icon strip with a panel): Thumbnails, Bookmarks, Search results, Layers, Attachments, Comments list, Blanks navigator (Tender).
5. Centre canvas with a floating zoom control and page navigator.
6. Right inspector showing the properties of whatever is selected (font, size, colour, opacity, position, size, lock).
7. Status bar: page X of Y, zoom, file size, background-job progress, theme toggle.
Behaviours: drag and drop files anywhere to open or add pages; right-click context menus; rubber-band multi-select; snap guides and rulers; arrow-key nudge; session restore on restart; crash recovery with a “restore unsaved work” prompt; skeleton loaders (no spinners that block the UI); every long task has a progress bar and a Cancel button; toasts for confirmations; confirm dialogs only for destructive actions.
Shortcuts follow Foxit and Adobe conventions (Ctrl+O, S, Shift+S, Z, Y, F, P, K, G for go to page, Ctrl+0 fit page, Ctrl+1 actual size, Ctrl+Plus and Minus). Document them in docs/SHORTCUTS.md and show them in tooltips.
A7. Error handling
Corrupt file: try qpdf repair, then offer a read-only open. Password-protected file: password prompt with a show/hide toggle. Very large file: warn and offer a low-memory mode. Any exception shows a friendly message with a “copy details” button and is written to a local log file.
