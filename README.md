# HLBK Form Digitizer

> AI-powered digitization of handwritten German ecological survey forms (A03 Kartierbögen) for forest habitat mapping.

**Status:** Production &nbsp;|&nbsp; **Stack:** Python, Flask, Claude Vision API, SpatiaLite &nbsp;|&nbsp; **Source:** Private

---

## The Problem

Field ecologists in Germany collect biodiversity data on **paper forms** during forest habitat surveys (*Waldkartierung*). Each A03 form captures:

- **Species observations** — Latin names, vegetation layers, coverage percentages
- **Habitat structure** — checkboxes for relief, exposition, rock formations, deadwood
- **Site metadata** — survey area ID, date, surveyor initials, vegetation units

A single survey campaign produces **hundreds of handwritten forms** that must be manually transcribed into the HLBK database — a tedious, error-prone process that takes ecologists away from fieldwork.

## The Solution

This tool replaces manual transcription with an AI-assisted pipeline:

```
    Scan              Extract            Review              Save
┌───────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Upload   │───▶│  Claude AI   │───▶│  Human-in-   │───▶│     HLBK     │
│  PDF/HEIC │    │  Vision API  │    │  the-loop    │    │   Database   │
└───────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                 Reads handwriting    Validates with      Atomic writes
                 & checkbox states    confidence scores   with FK integrity
```

**One form takes ~10 seconds instead of ~10 minutes.**

---

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                       Flask Web Application                       │
│                                                                   │
│  ┌───────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│  │ Upload &  │  │  Template  │  │   Review   │  │   Batch    │  │
│  │ Extract   │  │  Editor    │  │     UI     │  │   Export   │  │
│  └─────┬─────┘  └────────────┘  └──────┬─────┘  └────────────┘  │
│        │                               │                         │
└────────┼───────────────────────────────┼─────────────────────────┘
         │                               │
         ▼                               ▼
┌──────────────────┐            ┌──────────────────┐
│  Claude Vision   │            │ Species Matcher  │
│  Extraction      │            │ (Fuzzy + Exact)  │
│                  │            │                  │
│  • Image pre-    │            │  • 21,500+       │
│    processing    │            │    species DB    │
│  • Token budget  │            │  • RapidFuzz     │
│  • Structured    │            │    WRatio scorer │
│    JSON output   │            │  • Confidence    │
│  • ~$0.10/form   │            │    thresholds    │
└──────────────────┘            └────────┬─────────┘
                                         │
                                         ▼
                               ┌──────────────────┐
                               │ Database Layer   │
                               │                  │
                               │ • Repository     │
                               │   pattern        │
                               │ • Unit of Work   │
                               │   transactions   │
                               │ • GUID compat    │
                               │   with legacy    │
                               │   desktop tool   │
                               └────────┬─────────┘
                                        │
                                        ▼
                               ┌──────────────────┐
                               │   SpatiaLite     │
                               │ (HLBK Schema)    │
                               │                  │
                               │ ke_obj           │
                               │ art_fund         │
                               │ art_beobachtung  │
                               │ ke_obj_hus       │
                               └──────────────────┘
```

---

## Key Features

### AI-Powered Extraction

The extraction pipeline uses **Claude Vision** to read handwritten ecological forms:

- **Image preprocessing** — Otsu's threshold + contrast enhancement for degraded scans
- **Token budgeting** — Caps input at ~50K tokens to control API costs (~$0.10/form)
- **Structured output** — Returns typed JSON matching the form's field schema
- **Multi-page support** — Handles A03 forms that span multiple pages

### Intelligent Species Matching

```
Handwritten:  "Q. robur"        ──▶  Quercus robur     ✅ 96% confidence
              "Carx silvatica"  ──▶  Carex sylvatica    ⚠️ 82% confidence
              "Plg commune"     ──▶  Polytrichum commune ❌ 45% confidence
```

- **Two-pass matching** — Exact match first, fuzzy only for unresolved names
- **RapidFuzz WRatio** — Handles abbreviations, misspellings, and shorthand notation
- **Tiered confidence** — Green (>90%), Yellow (70-90%), Red (<70%) with mandatory human review

### Template System

The form structure is defined in JSON templates (not hardcoded), enabling:

- **Visual template editor** with drag-and-drop sections
- **Auto-migrations** when template structure changes
- **Version tracking** to handle evolving form layouts across survey years

### Human-in-the-Loop Review

Every extraction goes through human validation before database write:

- Confidence-colored indicators for each field
- Side-by-side view: original scan vs. extracted data
- Inline editing for corrections
- Keyboard shortcuts for rapid review (Tab, Y/N)

### Database Integration

```
┌───────────────────────────────────────────────────────┐
│                    Write Service                      │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │   Lookup    │  │     GUID     │  │   Repos    │  │
│  │  Resolver   │  │  Generator   │  │   (CRUD)   │  │
│  │             │  │              │  │            │  │
│  │ checkbox →  │  │ Microsoft-   │  │   ke_obj   │  │
│  │ GUID FK     │  │ format IDs   │  │   art_*    │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │            Unit of Work (atomic)               │  │
│  │   All tables written in single transaction     │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

- **Repository pattern** — Clean separation per database table
- **Unit of Work** — All-or-nothing writes (no partial form saves)
- **Lookup resolution** — Maps checkbox codes to GUID foreign keys from reference tables
- **Legacy compatibility** — Generates Microsoft-format GUIDs (`{xxxxxxxx-xxxx-...}`) matching the existing desktop application

### Batch Processing

- Scan a working directory for unprocessed PDFs
- Sequential extraction with real-time progress UI
- Incremental processing — skips already-extracted forms
- Cancel mid-batch without losing completed work

### PDF Export

- Generate printable PDF reports from extracted data
- WeasyPrint-based rendering with flexbox layout
- Bulk export with progress tracking

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | Python 3.13, Flask |
| **AI** | Claude Vision API (Anthropic) |
| **Database** | SpatiaLite (spatially-enabled SQLite) |
| **Fuzzy Matching** | RapidFuzz |
| **Image Processing** | Pillow, Otsu's threshold |
| **PDF Generation** | WeasyPrint |
| **Auth** | Flask-Login, CSRF protection |
| **Frontend** | Vanilla JS, Jinja2 templates |
| **Client Storage** | File System Access API with localStorage fallback |
| **Deployment** | Docker, Gunicorn, Caddy reverse proxy |
| **Dev Tools** | uv, mypy, ruff, pytest |

---

## Data Flow

```
                        ┌──────────────────┐
                        │   Scanned Form   │
                        │   (PDF / HEIC)   │
                        └────────┬─────────┘
                                 │
                        ┌────────▼─────────┐
                        │  Image Enhance   │
                        │  Otsu threshold  │
                        │  + contrast adj  │
                        └────────┬─────────┘
                                 │
                        ┌────────▼─────────┐
                        │  Claude Vision   │
                        │  Extraction      │
                        │                  │
                        │  Structured JSON │
                        │  with all fields │
                        └────────┬─────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  │              │              │
         ┌───────▼───────┐ ┌────▼─────┐ ┌─────▼──────┐
         │   Metadata    │ │ Species  │ │ Checkboxes │
         │  (date, area, │ │  Names   │ │ & Habitat  │
         │   surveyor)   │ │          │ │ Structure  │
         └───────┬───────┘ └────┬─────┘ └─────┬──────┘
                 │              │              │
                 │        ┌─────▼──────┐      │
                 │        │ Fuzzy Match │      │
                 │        │ against     │      │
                 │        │ 21.5K taxa  │      │
                 │        └─────┬──────┘      │
                 │              │              │
                 └──────────┐  │  ┌────────────┘
                            │  │  │
                        ┌───▼──▼──▼──┐
                        │  Review UI  │
                        │ Confidence  │
                        │ indicators  │
                        │  + inline   │
                        │   editing   │
                        └──────┬──────┘
                               │
                        ┌──────▼──────────┐
                        │  Atomic Write   │
                        │  to SpatiaLite  │
                        │  (all-or-none)  │
                        └─────────────────┘
```

---

## Project Structure

```
scan-form/
├── src/
│   ├── app.py                  # Flask application (27+ routes)
│   ├── extraction.py           # Claude Vision integration
│   ├── species_matcher.py      # Fuzzy taxonomic matching
│   ├── species_db.py           # Species database loader (21.5K taxa)
│   ├── form_templates.py       # Template system with auto-migrations
│   ├── migrations.py           # Session data migration engine
│   ├── image_processing.py     # HEIC/PDF → JPEG, Otsu preprocessing
│   ├── validation.py           # Form data validation
│   ├── database/
│   │   ├── write_service.py    # Database write orchestration
│   │   ├── unit_of_work.py     # Transaction management
│   │   ├── lookup_resolver.py  # Checkbox code → GUID resolution
│   │   ├── guid_generator.py   # Microsoft-format GUID generation
│   │   └── repositories/       # Table-specific data access
│   ├── templates/              # Jinja2 HTML templates
│   │   └── a03_template.json   # Form structure definition
│   └── static/                 # CSS, JS, client-side logic
│       ├── review.js           # Review UI with confidence display
│       ├── template_editor.js  # Drag-drop template editor
│       └── server_store.js     # File System Access API storage
├── data/
│   └── species.csv             # Taxonomic reference dataset
├── tests/                      # pytest test suite
├── Dockerfile                  # Production container
└── pyproject.toml              # Dependencies (managed with uv)
```

---

## Cost Analysis

| Resource | Per Form | Per 100 Forms |
|----------|----------|---------------|
| Claude Vision input (~50K tokens) | ~$0.075 | ~$7.50 |
| Claude Vision output (~2K tokens) | ~$0.03 | ~$3.00 |
| **Total** | **~$0.10** | **~$10.00** |

vs. **~10 minutes manual transcription** per form = **~17 hours saved per 100 forms**.

---

## Why This Project?

This was built as an internal tool for a German environmental consulting firm to solve a real operational bottleneck. Key engineering challenges:

- **Handwriting OCR at scale** — Ecological shorthand (Latin abbreviations, layer codes) requires domain-specific prompting
- **Data integrity** — The HLBK database has strict FK constraints and expects Microsoft-format GUIDs
- **Human-AI collaboration** — The confidence-based review UI ensures quality while maximizing throughput
- **Legacy system integration** — Must produce identical output to the existing desktop GIS tool

---

## License

This is a **portfolio showcase**. The source code is in a private repository.

If you're interested in the technical details or would like to discuss the implementation, feel free to reach out.

---

*Built by [Jannis Menzler](https://github.com/jmenzler)*
