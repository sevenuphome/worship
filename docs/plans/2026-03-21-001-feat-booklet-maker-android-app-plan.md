---
title: "feat: Booklet Maker Android App"
type: feat
status: active
date: 2026-03-21
origin: docs/brainstorms/2026-03-21-booklet-maker-app-requirements.md
---

# feat: Booklet Maker — PDF Booklet Converter Android App

## Overview

Build an Android app (Kotlin, Jetpack Compose) that converts PDFs into saddle-stitch booklet format for folding and printing. The user picks a PDF, previews the imposed booklet layout, and exports a ready-to-print PDF — all on-device with no network required. Free with non-intrusive ads.

(see origin: `docs/brainstorms/2026-03-21-booklet-maker-app-requirements.md`)

## Problem Statement / Motivation

Printing booklets from standard PDFs requires manual page imposition — reordering and resizing pages so they print correctly for saddle-stitch folding. This is tedious, error-prone, and existing tools are either desktop-only, require technical knowledge, or are bloated. There's no clean, simple Android app that does this well. The core algorithm has been validated with a Python prototype.

## Proposed Solution

A single-purpose Android utility app with a 3-screen linear flow:

```
[Home] → Pick PDF → [Preview] → See booklet layout → [Export] → Save/Share
```

**Key technical decisions:**
- **PdfBox-Android** for PDF manipulation (Apache 2.0 license, preserves vector content, ~7MB)
- **PdfRenderer** (built-in) for preview thumbnails
- **Jetpack Compose** with Material 3 for UI
- **On-device only** — no network calls except AdMob
- **Coroutines** for background PDF processing

## Technical Approach

### Architecture

```
┌─────────────────────────────────────────────────┐
│                    UI Layer                       │
│  HomeScreen ─► PreviewScreen ─► ExportScreen     │
│  (Jetpack Compose + Navigation Compose)          │
├─────────────────────────────────────────────────┤
│                 ViewModel Layer                   │
│  BookletViewModel (StateFlow<UiState>)           │
│  - selectedUri, pageInfo, previewBitmaps         │
│  - exportProgress, outputUri                     │
├─────────────────────────────────────────────────┤
│                 Domain Layer                      │
│  BookletImposer     - saddle-stitch page ordering│
│  PdfAnalyzer        - page size detection        │
│  PreviewRenderer    - thumbnail generation       │
├─────────────────────────────────────────────────┤
│                  Data Layer                       │
│  PdfBox-Android     - read/compose/write PDFs    │
│  PdfRenderer        - render pages to Bitmap     │
│  SAF                - file picker, save          │
└─────────────────────────────────────────────────┘
```

**MVVM + UDF pattern:**

```kotlin
data class BookletUiState(
    val step: Step = Step.HOME,
    val selectedUri: Uri? = null,
    val pageInfo: PageInfo? = null,    // page count, size, detected format
    val previewSheets: List<Bitmap> = emptyList(),
    val isProcessing: Boolean = false,
    val progress: Float = 0f,
    val outputUri: Uri? = null,
    val error: ErrorType? = null
)

enum class Step { HOME, PREVIEW, EXPORTING, RESULT }
```

### PDF Library: PdfBox-Android

**Why PdfBox-Android** (see origin for decision context):
- **Apache 2.0 license** — fully compatible with free, ad-supported, closed-source app
- **Preserves vector content** via `LayerUtility.importPageAsForm()` — text stays text, lines stay sharp
- **~7MB APK impact** — acceptable for a utility app
- **Mature API** — upstream Apache PDFBox is well-documented, Android port API is nearly identical

**Eliminated alternatives:**
- iText 7 — AGPL license, incompatible with closed-source ad-supported app
- PSPDFKit/Nutrient — commercial pricing ($2,500+/year), overkill for imposition
- Built-in PdfRenderer+PdfDocument — rasterizes content, destroys vector quality, inflates file size
- OpenPDF — depends on `java.awt`, unavailable on Android

### Implementation Phases

#### Phase 1: Project Setup & Core Imposition Engine

**Deliverables:** Working imposition logic, can generate booklet PDFs from command-line tests.

**Tasks:**
- [ ] Create new Android project: `com.bookletmaker.app`
  - AGP 9.0, Kotlin 2.2.x, Compose BOM 2026.02, Min SDK 26, Target SDK 35
  - `app/build.gradle.kts`
- [ ] Add PdfBox-Android dependency (`com.tom_roush:pdfbox-android:2.0.27.0`)
- [ ] Initialize `PDFBoxResourceLoader.init(context)` in `Application.onCreate()`
  - `app/src/main/java/.../BookletApp.kt`
- [ ] Implement `PdfAnalyzer` — detect page size, count, orientation
  - `app/src/main/java/.../domain/PdfAnalyzer.kt`

```kotlin
// Page size detection with tolerance
fun detectPageSize(page: PDPage): PageFormat {
    val w = page.mediaBox.width
    val h = page.mediaBox.height
    val (width, height) = if (w > h) h to w else w to h // normalize to portrait
    return when {
        matches(width, height, 595f, 842f) -> PageFormat.A4
        matches(width, height, 420f, 595f) -> PageFormat.A5
        matches(width, height, 612f, 792f) -> PageFormat.US_LETTER
        else -> PageFormat.Custom(width, height)
    }
}
```

- [ ] Implement `BookletImposer` — core saddle-stitch page ordering + composition
  - `app/src/main/java/.../domain/BookletImposer.kt`

```kotlin
// Saddle-stitch ordering for N pages (padded to multiple of 4)
fun buildSignatureOrder(pageCount: Int): List<Pair<Int?, Int?>> {
    val padded = pageCount + (4 - pageCount % 4) % 4
    val sheets = padded / 4
    val result = mutableListOf<Pair<Int?, Int?>>()
    for (i in 0 until sheets) {
        val frontLeft = padded - 1 - 2 * i
        val frontRight = 2 * i
        val backLeft = 2 * i + 1
        val backRight = padded - 2 - 2 * i
        result.add(toNullable(frontLeft, pageCount) to toNullable(frontRight, pageCount))
        result.add(toNullable(backLeft, pageCount) to toNullable(backRight, pageCount))
    }
    return result
}

// Compose two source pages side-by-side on landscape output page using LayerUtility
fun imposePage(outputDoc: PDDocument, sourceDoc: PDDocument,
               leftIdx: Int?, rightIdx: Int?, outputSize: PDRectangle) {
    val page = PDPage(outputSize)
    outputDoc.addPage(page)
    val layerUtility = LayerUtility(outputDoc)
    val halfWidth = outputSize.width / 2
    val scale = halfWidth / sourceDoc.getPage(0).mediaBox.width

    val cs = PDPageContentStream(outputDoc, page)
    leftIdx?.let {
        val form = layerUtility.importPageAsForm(sourceDoc, it)
        cs.saveGraphicsState()
        cs.transform(Matrix(scale, 0f, 0f, scale, 0f, 0f))
        cs.drawForm(form)
        cs.restoreGraphicsState()
    }
    rightIdx?.let {
        val form = layerUtility.importPageAsForm(sourceDoc, it)
        cs.saveGraphicsState()
        cs.transform(Matrix(scale, 0f, 0f, scale, halfWidth, 0f))
        cs.drawForm(form)
        cs.restoreGraphicsState()
    }
    cs.close()
}
```

- [ ] Write unit tests for signature ordering (edge cases: 1 page, 2 pages, 4 pages, 26 pages)
  - `app/src/test/java/.../BookletImposerTest.kt`
- [ ] Write integration test: A4 PDF in → booklet PDF out, verify page count and dimensions
  - `app/src/androidTest/java/.../BookletImposerInstrumentedTest.kt`

**Success criteria:** `BookletImposer` can take any PDF and produce a correctly-imposed booklet PDF.

**Estimated effort:** 2-3 days

---

#### Phase 2: UI — Home Screen & File Picker

**Deliverables:** Home screen with file picker, share intent handling, Hilt DI.

**Tasks:**
- [ ] Set up Hilt dependency injection
  - `app/src/main/java/.../di/AppModule.kt`
- [ ] Set up Navigation Compose with type-safe routes
  - `app/src/main/java/.../navigation/Routes.kt`
  - `app/src/main/java/.../navigation/NavGraph.kt`

```kotlin
@Serializable object Home
@Serializable data class Preview(val fileUri: String)
@Serializable data class Result(val outputUri: String)
```

- [ ] Implement `HomeScreen` composable
  - `app/src/main/java/.../ui/home/HomeScreen.kt`
  - Large "Pick PDF" button (primary CTA)
  - App name/logo
  - Brief tagline: "Convert PDFs to printable booklets"
  - Banner ad at bottom

- [ ] Implement file picker via `ActivityResultContracts.OpenDocument`
  - Filter: `application/pdf`
  - Take persistable URI permission on selection

- [ ] Handle share intent / "Open with" in `MainActivity`
  - `app/src/main/java/.../MainActivity.kt`
  - Register intent filter for `application/pdf`
  - On receive: copy to app-internal cache (URI permissions may be temporary)
  - Navigate directly to Preview screen

- [ ] Implement `BookletViewModel`
  - `app/src/main/java/.../ui/BookletViewModel.kt`

**Success criteria:** User can pick a PDF or share one from another app, app navigates to preview.

**Estimated effort:** 2 days

---

#### Phase 3: UI — Preview Screen

**Deliverables:** Preview screen showing imposed booklet sheets.

**Tasks:**
- [ ] Implement `PreviewRenderer` using built-in `PdfRenderer`
  - `app/src/main/java/.../domain/PreviewRenderer.kt`
  - Render each source page to bitmap at reduced resolution (e.g., 1/4 scale)
  - Compose preview bitmaps: place two source bitmaps side-by-side per sheet
  - Generate previews lazily (only visible sheets) to handle large documents

- [ ] Implement `PreviewScreen` composable
  - `app/src/main/java/.../ui/preview/PreviewScreen.kt`
  - Top bar: PDF name, page info ("26 pages → 14 sheets, A4 → A5 booklet")
  - Scrollable `LazyColumn` of imposed sheet previews
  - Each item shows: sheet number, "Front" / "Back" label, preview bitmap
  - "Generate Booklet" FAB at bottom
  - Loading indicator while previews generate

- [ ] Handle edge cases in preview:
  - Single-page PDF → show 1 sheet with 3 blank areas
  - Large PDF (60+ pages) → show warning: "This booklet will be thick (15+ sheets). Saddle-stitch binding works best with fewer than 15 sheets."
  - Password-protected PDF → error screen with "This PDF is password-protected" message
  - Corrupt PDF → error screen with "Could not read this PDF" message

**Success criteria:** User sees accurate preview of how each printed sheet will look.

**Estimated effort:** 3 days

---

#### Phase 4: UI — Export & Result Screen

**Deliverables:** Export pipeline, result screen with save/share.

**Tasks:**
- [ ] Implement export pipeline in `BookletViewModel`
  - Run on `Dispatchers.IO` via `viewModelScope`
  - Emit progress updates (0..1f) as each sheet is composed
  - Write output to app cache directory first, then offer save/share
  - Handle cancellation via coroutine cancellation

- [ ] Implement `ExportScreen` composable (shown during processing)
  - `app/src/main/java/.../ui/export/ExportScreen.kt`
  - Circular progress indicator with percentage
  - "Generating booklet..." text
  - Cancel button

- [ ] Implement `ResultScreen` composable (shown after export)
  - `app/src/main/java/.../ui/result/ResultScreen.kt`
  - Success message with output info: "Booklet ready! 14 sheets, landscape A4"
  - Thumbnail of first sheet
  - "Save to Device" button → `ACTION_CREATE_DOCUMENT` with suggested name `{original}_booklet.pdf`
  - "Share" button → `ACTION_SEND` with `FileProvider` URI
  - "Make Another" button → return to Home

- [ ] Output filename convention: `{original_filename}_booklet.pdf`

- [ ] Error handling:
  - Export failure (OOM, disk full) → error message with retry option
  - File write failure → error message

**Success criteria:** User can export booklet PDF and save or share it.

**Estimated effort:** 2 days

---

#### Phase 5: Ads Integration

**Deliverables:** AdMob banner and interstitial ads.

**Tasks:**
- [ ] Add AdMob dependency: `com.google.android.gms:play-services-ads`
- [ ] Initialize AdMob in `Application.onCreate()`
- [ ] Add `<meta-data>` for AdMob app ID in `AndroidManifest.xml`
- [ ] Implement `AdBanner` composable using `AndroidView` wrapper
  - `app/src/main/java/.../ui/components/AdBanner.kt`
  - Adaptive banner at bottom of HomeScreen only
  - Use test ad unit IDs in debug builds

- [ ] Implement `AdManager` for interstitial ads
  - `app/src/main/java/.../ads/AdManager.kt`
  - Pre-load interstitial when app launches
  - Show after successful export (between export completion and result screen)
  - **If ad fails to load (offline, timeout): skip ad, proceed to result screen immediately**
  - Frequency cap: max 1 interstitial per session
  - Re-load after showing

- [ ] Add `INTERNET` permission to manifest (required for AdMob, but app functionality works without it)
- [ ] Add `ACCESS_NETWORK_STATE` permission (AdMob recommended)

**Ad placement strategy** (see origin for decision):
- Banner: Home screen only — no ads on preview or export screens
- Interstitial: After successful export, before result screen
- Offline: All app functionality works without ads; ads are purely opportunistic

**Success criteria:** Ads show when online, app works fully when offline.

**Estimated effort:** 1 day

---

#### Phase 6: Polish & Play Store Prep

**Deliverables:** App icon, privacy policy, Play Store listing, crash reporting.

**Tasks:**
- [ ] App icon (adaptive icon, Material 3 themed)
  - `app/src/main/res/mipmap-*/ic_launcher.xml`
- [ ] Splash screen (API 31+ SplashScreen API, fallback for API 26-30)
- [ ] First-launch onboarding: single dismissable card explaining saddle-stitch folding with illustration
- [ ] Add Firebase Crashlytics for crash reporting
- [ ] Privacy policy page (GitHub Pages or Firebase Hosting):
  - Disclose AdMob data collection (Advertising ID, device info, IP)
  - State that PDF files never leave the device
  - Link to Google ad settings for opt-out
- [ ] Play Store assets:
  - Screenshots (phone + tablet)
  - Feature graphic (1024x500)
  - Short description, full description
  - Content rating questionnaire (IARC)
  - Data Safety form (declare AdMob data collection)
- [ ] ProGuard/R8 rules for PdfBox-Android
- [ ] Test on Android 8.0 (API 26) and Android 15 (API 35)
- [ ] Test with various PDF types: A4, Letter, scanned, multi-page, single-page, password-protected (verify error message)

**Success criteria:** App passes Play Store review, installs and runs on API 26-35.

**Estimated effort:** 2-3 days

## Error Handling Strategy

| Error Case | User-Facing Message | Behavior |
|---|---|---|
| Not a valid PDF | "This file doesn't appear to be a valid PDF." | Return to Home |
| Password-protected PDF | "This PDF is password-protected. Please remove the password first." | Return to Home |
| Empty PDF (0 pages) | "This PDF has no pages." | Return to Home |
| Mixed page sizes | Use first page's size, warn: "Pages have different sizes — using the first page's size." | Continue |
| 60+ pages | "This booklet will be very thick (N sheets). Saddle-stitch works best under 15 sheets." | Warning, allow proceed |
| Export OOM / failure | "Failed to generate booklet. The PDF may be too large." | Retry button |
| Disk full | "Not enough storage space to save the booklet." | Return to Result |
| Ad fails to load | (no message) | Skip ad silently |

## System-Wide Impact

**Interaction graph:** User picks file → SAF returns URI → ViewModel loads PDF via PdfBox → PdfAnalyzer detects size → PreviewRenderer generates thumbnails → User confirms → BookletImposer generates output → AdManager shows interstitial → ResultScreen offers save/share.

**Error propagation:** All errors caught in ViewModel, mapped to `ErrorType` sealed class, displayed by UI. No unhandled exceptions reach the user.

**State lifecycle risks:** If app is killed during export, output file in cache is incomplete. On next launch, app starts fresh at Home (no partial state to clean up). Cache files are cleaned on next export attempt.

## Acceptance Criteria

### Functional Requirements

- [ ] User can pick a PDF via file picker (R1)
- [ ] User can share a PDF from another app to Booklet Maker (R1)
- [ ] App detects A4 page size and labels it correctly (R2)
- [ ] Pages are imposed in correct saddle-stitch order (R3)
- [ ] PDFs with non-multiple-of-4 pages are padded with blanks (R4)
- [ ] Preview shows all imposed sheets before export (R5)
- [ ] Output is a landscape PDF with 2 source pages per sheet (R6)
- [ ] App works fully offline (except ads) (R7)
- [ ] Banner ad on home screen, interstitial after export (R8)
- [ ] Runs on Android 8.0+ (R9)

### Non-Functional Requirements

- [ ] Export < 10 seconds for a 50-page document
- [ ] Output PDF preserves vector content (text is selectable, sharp at any zoom)
- [ ] APK size < 20MB
- [ ] No crashes on password-protected, corrupt, or empty PDFs

### Quality Gates

- [ ] Unit tests for signature ordering algorithm (all edge cases)
- [ ] Integration test: end-to-end PDF in → booklet PDF out
- [ ] Manual test on API 26 and API 35 devices/emulators
- [ ] Privacy policy published and linked

## Success Metrics

(see origin: `docs/brainstorms/2026-03-21-booklet-maker-app-requirements.md`)

- User can go from picking a PDF to holding a printed, folded booklet with no external tools
- Export < 10 seconds for typical documents (< 50 pages)
- 4.0+ star rating on Play Store within 3 months
- 1,000+ downloads in first month

## Dependencies & Prerequisites

- PdfBox-Android 2.0.27.0 (Apache 2.0) — single maintainer risk mitigated by stable codebase + Apache license allowing fork
- AdMob SDK
- Firebase Crashlytics
- Android Studio with AGP 9.0
- Google Play Developer account ($25 one-time)

## Risk Analysis & Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| PdfBox-Android unmaintained | Medium | Apache 2.0 license allows forking; core API is stable |
| Large/complex PDFs cause OOM | Medium | Process pages one at a time; set max page warning at 60 |
| AdMob rejection (policy violation) | Low | Follow placement guidelines, no accidental clicks near ads |
| Competing apps copy the idea | Low | Ship fast, iterate on UX, build brand |

## Scope Boundaries

(see origin)

- v1 is Android only
- v1: A4 → A5 primary flow; US Letter → Half-Letter as stretch goal
- No PDF editing, no direct printing, no batch processing, no cloud/accounts
- Odd/even splitting is nice-to-have for v2

## Sources & References

### Origin

- **Origin document:** [docs/brainstorms/2026-03-21-booklet-maker-app-requirements.md](docs/brainstorms/2026-03-21-booklet-maker-app-requirements.md) — Key decisions: Android first (Kotlin/Compose), on-device processing, free with ads, preview before export.

### External References

- [PdfBox-Android GitHub](https://github.com/TomRoush/PdfBox-Android) — Apache 2.0, v2.0.27.0
- [Apache PDFBox LayerUtility API](https://pdfbox.apache.org/docs/2.0.11/javadocs/org/apache/pdfbox/multipdf/LayerUtility.html)
- [Navigation Compose Type-Safe Routes](https://developer.android.com/guide/navigation/design/type-safety)
- [AdMob Implementation Guidelines](https://support.google.com/admob/answer/6128877)
- [AGP 9.0 Release Notes](https://developer.android.com/build/releases/agp-9-0-0-release-notes)
- [Play Store Target API Requirements](https://developer.android.com/google/play/requirements/target-sdk)
