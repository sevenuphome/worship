---
date: 2026-03-21
topic: booklet-maker-app
---

# Booklet Maker — PDF Booklet Converter App

## Problem Frame

Printing A5 booklets from A4 (or any size) PDFs requires manual page imposition — reordering and resizing pages so they print correctly for saddle-stitch folding. Existing tools are either desktop-only, require technical knowledge, or are bloated with features. There's no clean, simple Android app that does this well.

**Target users:** Anyone who prints booklets — worship teams, zine makers, students, teachers, small businesses, hobbyists.

## Requirements

- R1. User can pick a PDF from their device (file picker or share intent)
- R2. App detects the PDF's page size and offers appropriate booklet conversions (e.g., A4 → A5 booklet on A4, Letter → Half-Letter booklet on Letter)
- R3. App imposes pages in correct saddle-stitch order (e.g., for 28 pages: sheet 1 front = pages 28+1, sheet 1 back = pages 2+27, etc.)
- R4. App pads to the nearest multiple of 4 pages with blank pages when needed
- R5. User can preview the booklet layout before exporting — see how pages will be arranged on each sheet
- R6. App exports a ready-to-print PDF in landscape orientation with two source pages per sheet
- R7. All processing happens on-device — no network calls, no uploads, works fully offline
- R8. App is free with non-intrusive ads (banner on home screen, interstitial after export)
- R9. App supports Android 8.0+ (API 26+)

## Success Criteria

- User can go from picking a PDF to holding a printed, folded booklet with no external tools or manual page reordering
- Export takes under 10 seconds for a typical document (< 50 pages)
- 4.0+ star rating on Play Store within first 3 months
- 1,000+ downloads in first month

## Scope Boundaries

- v1 is Android only (iOS, web, desktop are future phases)
- v1 supports A4 → A5 booklet as the primary flow; other size combinations are stretch goals
- No cloud processing, no accounts, no sign-up
- No PDF editing (crop, rotate individual pages, reorder) — this is a converter, not an editor
- No direct printing from the app — user exports PDF and prints via their preferred method
- Odd/even page splitting for manual duplex is a nice-to-have, not required for v1
- Batch processing is out of scope for v1

## Key Decisions

- **Android first:** Plays to developer's strengths (Kotlin, Jetpack Compose). Expand to other platforms later, likely via KMP or Flutter.
- **On-device processing:** Privacy differentiator, no server costs, works offline. Constrains PDF library choice to something that runs on Android.
- **Free with ads:** Maximizes downloads for a utility app. Can add premium tier later if there's demand.
- **Preview before export:** Key differentiator — most competing apps just blindly convert. Seeing the layout builds confidence.

## Dependencies / Assumptions

- A suitable Android PDF manipulation library exists that can read pages, resize, and compose new pages (candidates: Apache PDFBox Android, iText, PSPDFKit)
- Ads via AdMob or similar

## Outstanding Questions

### Resolve Before Planning

(none)

### Deferred to Planning

- [Affects R3][Needs research] Which PDF library works best on Android for page imposition? Evaluate PDFBox-Android vs iText vs alternatives for: license compatibility (free+ads model), APK size, performance, API ergonomics
- [Affects R5][Technical] What's the best approach for preview — render pages via PdfRenderer into a composable, or generate a low-res preview PDF?
- [Affects R2][Technical] How to reliably detect source page size from the PDF metadata?
- [Affects R8][Technical] Ad placement strategy — where to show ads without degrading the simple UX?

## Next Steps

-> `/ce:plan` for structured implementation planning
