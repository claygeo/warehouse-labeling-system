# Inventory & Labeling System

[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![Vite](https://img.shields.io/badge/Vite-7-646CFF?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-3.4-38BDF8?logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)
[![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?logo=supabase&logoColor=white)](https://supabase.com/)
[![jsPDF](https://img.shields.io/badge/jsPDF-4-EE3524)](https://github.com/parallax/jsPDF)
[![Zebra](https://img.shields.io/badge/Zebra-ZT610%20thermal-000000)](https://www.zebra.com/)

A production warehouse-floor tool that turns messy inventory exports into scannable 4x6 thermal labels for a cannabis distribution facility. An operator loads the day's inventory, scans a barcode, and the system resolves the product, assigns a batch, and prints a clean label on a Zebra ZT610. I built it in July 2025 while working as a logistics coordinator, and the floor runs on it daily, currently v7.7.6. Since print tracking started in August 2026 it has generated 144,000+ labels, about 2,400 on a working day.

> **Note:** This is a showcase repo. The production codebase is private because it is an internal tool for a real employer and prints on real inventory. This README documents the architecture, the engineering decisions, and the field-earned problems I solved. It intentionally contains no code, no credentials, and no company data.

---

## The Problem

The warehouse relabels product before it ships. The source data comes out of a state track-and-trace system (BioTrack) and a transfer tool (Sweed) as CSV and Excel exports, and it is rough: three different report formats, columns that move between exports, product names abbreviated into codes a human cannot read, and Excel quietly corrupting 16-digit barcodes into scientific notation (`7.75984E+15`).

On top of that, two facilities run different label orientations, print runs can be thousands of labels at once, and the whole thing has to work on a warehouse floor where the network is not guaranteed. The old process was done by hand: print an Excel template, fill it out with a pen, then write the details on each box with a Sharpie. It was slow, it was error-prone, and a wrong label on a box is a real operational problem.

## What I Built

A single-page app the floor actually runs every day:

- Imports three different inventory export formats (Main Inventory, Sweed transfer, and the ~54-column BioTrack Import Report) with dynamic column detection so a slightly different export still loads.
- Resolves abbreviated BioTrack names against a master product catalog to recover the full retail name, strain, and case count.
- Scans barcodes and auto-assigns letter batch IDs (A to Z, then AA to ZZ) per SKU.
- Renders 4x6 thermal labels client-side and prints them on a Zebra ZT610, in two orientations depending on the facility.
- Keeps the catalog available when the network drops, backed by a verified local copy.
- Logs any product it cannot recognize and gives admins a screen to remap it, so the catalog heals over time.

The floor went from reading codes off a spreadsheet to scanning a barcode and pulling a finished label off the printer.

---

## Architecture

```
  CSV / XLSX exports                       master-list.xlsx
  (Main / Sweed / Import Report)           (real catalog, not shipped)
        │                                        │
        ▼                                        ▼
  ┌──────────────────┐                  ┌───────────────────────────┐
  │  DataProcessor   │                  │   Master Catalog          │
  │  PapaParse+xlsx  │                  │                           │
  │  column detect,  │                  │  Supabase  ─► source of   │
  │  dedup, allergen │                  │  localStorage ─► offline  │
  │  + date parsing  │                  │                           │
  └────────┬─────────┘                  └─────────────┬─────────────┘
           │  sessionStorage                          │  name / strain / case lookup
           ▼                                           ▼
  ┌──────────────────────────────────────────────────────────────┐
  │              React Context layer (useReducer)                 │
  │   AuthContext  │  InventoryContext  │  SessionContext         │
  │   role + site  │  3 datasets +      │  scanned set, batch IDs │
  │   (2 sites)    │  catalog autoload  │  enhanced per-SKU data  │
  └───────────────────────────┬──────────────────────────────────┘
                              │  scan → resolve → configure
                              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    Label render engine                        │
  │                                                              │
  │   PDFGenerator: Canvas 2D ─► JPEG (2.5x) ─► jsPDF [288x432pt] │
  │      Site A ─► horizontal (canvas rotated 90 degrees)        │
  │      Site B ─► vertical (true 4x6 portrait)                  │
  │      CODE128 barcode, dynamic font sizing, allergen box      │
  │                                                              │
  │   (secondary path: ZPLGenerator ─► Zebra Browser Print)      │
  └───────────────────────────┬──────────────────────────────────┘
                              ▼
                     Zebra ZT610 @ 203 DPI
                   (every box gets a label)

  Unrecognized product ─► unmapped_products_log (Supabase) ─► Admin remap screen
```

The app is a Vite React SPA. Working data lives in `sessionStorage`, durable config and the catalog live in `localStorage`, and Supabase is the cloud source of truth for the catalog. Business logic sits in a `utils/` layer of small, testable classes (`DataProcessor`, `StorageManager`, `PDFGenerator`, `BarcodeGenerator`) rather than in the components.

---

## Engineering Challenges

### 1. The scanner tilde-prefix hack

The single most field-earned line in the codebase. Many of the warehouse's handheld scanners are configured to strip the first character of whatever they read. Print a normal barcode and every scan comes back missing its first digit, which means every lookup fails.

The fix is one character: the encoded barcode value is prefixed with a tilde (`~`), so the scanner strips the tilde instead of the first real digit and the payload arrives intact. It is the kind of thing you only learn by standing next to the hardware, and it is exactly why the label renderer encodes `'~' + value`.

### 2. Excel corrupts long barcodes, and my first fix was wrong

Barcodes are 16-digit numbers. The moment one passes through Excel, it can come back as `7.75984E+15`. My first fix rounded that back into a full integer, which looked right and was not: those digits are gone, so it fabricated a barcode that could then be printed on a real box.

The importer now treats a scientific-notation identifier as a hard row error and never guesses. It also finds Package IDs that were stored as Excel numbers and quarantines only those label rows, so one unsafe cell does not block a whole mixed report.

### 3. Two facilities, two orientations, one renderer

One site prints horizontal labels; the other prints vertical. Rather than maintain two renderers, the label engine draws to a single Canvas 2D context and branches on the operator's facility: the horizontal site rotates the canvas 90 degrees so a 4x6 portrait sheet reads as 6x4 landscape once peeled, and the vertical site draws a true 4x6 portrait with a stacked layout. The orientation follows the logged-in user's facility, with a per-label toggle for the occasional vertical override.

### 4. Rendering thousands of labels in a browser without crashing

A print run can be hundreds of pages, each an oversampled canvas rendered to JPEG and appended to a single jsPDF document. Do that naively and the tab runs out of memory.

The generator degrades deliberately under load: full resolution for normal runs, then it steps the canvas scale down at 200 and 400 pages, yields back to the event loop every 20 pages so the UI stays alive, drives a live progress bar, and calls `window.gc()` when the browser exposes it. It trades a little print sharpness on huge runs for finishing the run at all.

### 5. A catalog that survives a dropped connection

The catalog that maps BioTrack codes to real product names is the heart of the tool, and the floor cannot wait on a flaky connection. Loading follows a fallback: Supabase first (source of truth), then a verified `localStorage` copy (offline). An earlier bundled static file was removed because it hardcoded one legacy worksheet. A successful Supabase read writes through to `localStorage` so the next offline session is covered.

Cache invalidation is version-gated: on startup the app compares a stored catalog version against the current one and clears stale keys while preserving the catalog and any operator overrides. When the app itself ships a new version, session data is cleared but durable config survives.

### 6. Three messy formats, one clean import

Each source export has its own shape: Main Inventory (29 columns, data starting row 3), the Sweed transfer report (12 columns, header on row 11), and the primary BioTrack Import Report (~54 columns). Rather than hardcode column positions that break the first time a report changes, the importer detects columns by matching header text against variation lists and only adopts a detected mapping if it resolves more fields than the static default. It also parses harvest and package dates straight out of BioTrack `Custom_1` / `Custom_3` code strings, validates them to a sane year range, and flags allergens by scanning product names.

### 7. A catalog that learns from what it cannot read

When the system meets a product it cannot resolve, it does not just drop it. It logs the unrecognized name to Supabase with an occurrence counter (silent and non-blocking, so it never interrupts a print run), and an admin screen lets a privileged user remap it. Resolution walks a chain of overrides (session, then persistent user override, then the master catalog, then a partial match), and remap history is kept. The catalog gets more accurate the more the tool is used.

---

## What a Label Carries

Each 4x6 label is composed on the fly from the resolved product and the operator's configuration:

- Full product name, auto-sized to fit (11 to 26pt) and wrapped to fit the box
- A scannable CODE128 barcode (tilde-prefixed, as above)
- Case quantity, box count (`box:total`, font-scaled so `300:310` still fits), and a letter warehouse batch
- Harvest date and package date when present
- A ruled hand-write store box
- An allergen warning box when the product name trips the allergen heuristic
- A package/batch reference with the meaningful digits emphasized
- An audit line: timestamp, operator, and a manual-label marker

---

## Stack

| Layer | Technology |
|-------|-----------|
| Framework | React 18, React Router 7 |
| Build | Vite 7, Tailwind CSS 3.4, PostCSS |
| PDF / render | jsPDF 4, Canvas 2D, JsBarcode 3.12 (CODE128) |
| Printer path | Zebra ZT610 @ 203 DPI, optional Zebra Browser Print (ZPL) |
| Import | PapaParse 5.4 (CSV), SheetJS / xlsx 0.20 (Excel) |
| Data | Supabase (PostgreSQL) catalog, accounts and audit log; sessionStorage + localStorage |
| Dates / UI | date-fns 3.6, lucide-react, react-hot-toast |
| Deploy | Netlify (SPA, CSP, immutable asset caching), GitHub Actions CI (unit, real-browser and migration tests, audit, build) |

---

## Production

| Metric | Value |
|--------|-------|
| Status | Built Jul 2025; in daily use on the floor |
| Labels generated | 144,000+ since tracking began Aug 2026, about 2,400 on a working day |
| Version | v7.8 (evolved through several label-format migrations) |
| Facilities | 2 (horizontal and vertical label formats) |
| Operators | Named per-user accounts with Administrator and Warehouse roles |
| Catalog | 1,000+ products, paginated past Supabase's default row limit |
| Print runs | Up to 5,000 labels per PDF, with a review step before every print |
| Storage model | Session working data + durable local config + cloud catalog |

The label format has been through real migrations, from a Uline legal-sheet 4-up layout on an HP sheet printer to the current single 4x6 thermal on the Zebra. The version number is not decoration; it is a tool that has been maintained through those changes since July 2025 while the floor kept printing on it.

---

## Scope, Honestly

Portfolio repos should be honest about what a thing is and is not, so:

- **These are operational labels, not certified compliance labels.** They exist to move product accurately through the warehouse: a readable name, a scannable barcode, batch and box tracking, dates, and an allergen flag. The system does not validate against any state's cannabis labeling regulation and is not a METRC/BioTrack compliance-certification product.
- **Auth is scoped for an internal tool.** Each operator has a named account (Supabase Auth) and every print is logged to an audit trail under that account. It is built for a fixed set of warehouse users, not for a public or multi-tenant app.
- **It is field-hardened and test-covered.** CI runs unit tests, real-browser label tests and migration tests on every push. Much of the reliability still comes from daily floor use and defensive coding (error boundaries, per-row try/catch, non-blocking cloud writes, barcode fallbacks).
- **The allergen flag is a keyword heuristic**, not an ingredient database. It is a useful safety prompt for the operator, not a regulatory guarantee.

The interesting engineering here is not scale for its own sake. It is the accumulated, specific knowledge of what actually breaks between a state track-and-trace export and a clean label on a physical box.

---

## Why This Repo is a Showcase

The production code is private because it is an internal tool that prints on a real employer's inventory and reads from their catalog. This repo documents the engineering without exposing anything sensitive: no company data, no credentials, no employee records, no code.

If you want to read my code, my open-source projects are the place. [solhunt](https://github.com/claygeo/solhunt) (agent for authorized smart-contract research), [distru-mcp](https://github.com/claygeo/distru-mcp) (MCP server) and [sandwich-rs](https://github.com/claygeo/sandwich-rs) (Rust) are fully open. A readable case study of this system is at [claygeo.dev](https://claygeo.dev/work/warehouse-labeling-system/).

---

*Built by [Clayton George](https://github.com/claygeo)*
