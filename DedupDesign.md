## CleanOurData – Media Duplicate Finder Design

This document describes the UX, data model, and scanning architecture for a macOS app that finds duplicate media files (photos and videos) across multiple disks (local, removable, WebDAV) at TB scale.

---

## Goals

- **Detect duplicate media files** (photos and videos) across:
  - Local Mac storage
  - External/removable disks (U-disk, USB HDD/SSD, etc.)
  - WebDAV remote disks
- **Handle TB-scale data efficiently**:
  - Avoid reading every byte of every file when possible
  - Work incrementally and stay responsive
- Provide a **simple, safe, and friendly UI**:
  - Clear “select sources → scan → review & clean” flow
  - Easy to see what will be deleted and where it lives

---

## UX / Screens

The app flow is organized into three primary screens:

1. **Sources screen (Home)**
2. **Scan progress screen**
3. **Duplicates review screen**

### 1. Sources Screen (Home)

This replaces the current `ContentView`’s “Hello, world!” UI.

**Purpose**: Let the user choose where to scan and what to scan before doing any heavy work.

**Key elements**:

- **Header**
  - Title: “Clean Our Data”
  - Subtitle: “Find duplicate photos and videos across your disks.”

- **Source list**
  - Section: **Local & External Disks**
    - Rows for:
      - Local system volume, e.g. “Macintosh HD”
      - Each mounted volume under `/Volumes`, e.g. “Elements”, “BackupDisk”
    - Each row shows:
      - Volume name
      - Volume path (e.g. `/Volumes/Elements`)
      - Optional: capacity and free space
      - Toggle/checkbox “Include”
  - Section: **WebDAV Locations**
    - Row: “Add WebDAV server…”
      - Opens a sheet:
        - Fields: URL, username, password (or token)
        - Button: “Test & Save”
      - Saved WebDAV servers are listed as rows with a toggle “Include”.

- **Filter options**
  - Section: **Media types**
    - Toggle: “Photos (jpg, jpeg, png, heic, raw, …)”
    - Toggle: “Videos (mp4, mov, mkv, avi, …)”
  - Section: **Advanced**
    - Toggle: “Skip files smaller than 100 KB”
    - Toggle: “Skip files larger than 10 GB”
    - Toggle: “Use fast scan (partial hash)”
    - Helper text: “Fast scan may rarely miss tricky duplicates; recommended for very large data sets.”

- **Primary action**
  - Large button at the bottom: **“Scan for duplicates”**
    - Disabled until at least one source is selected.
    - On tap: navigates to **Scan progress screen** and starts the background scan.

Implementation-wise, `ContentView` becomes a `NavigationStack` hosting this sources view and holding a `ScanViewModel` as a `@StateObject`.

### 2. Scan Progress Screen

**Purpose**: Show that the app is working, provide high-level progress, and allow cancellation or pausing.

**UI elements**:

- **Progress summary**
  - Title text: “Scanning for media files…”
  - Status line: `X of Y files scanned` (Y can start as “estimating…”)
  - Line showing current path: `Current folder: /Volumes/Elements/Photos/2021/...`

- **Progress indicators**
  - Indeterminate `ProgressView` initially.
  - Optionally a phase label:
    - “Phase 1 of 2: index & group by size”
    - “Phase 2 of 2: hashing candidates”

- **Stats**
  - “Media found: 123,456”
  - “Potential duplicate groups: 1,234” (updated as we identify candidates)

- **Controls**
  - Button: **“Pause”** / **“Resume”**
  - Button: **“Stop & Keep Results So Far”** (keeps duplicates found up to now)
  - Button: **“Cancel Scan”** (discards results)

On completion, the app either automatically transitions to the **Duplicates review screen** or shows a button “View results”.

### 3. Duplicates Review Screen

**Purpose**: Let the user inspect and safely choose which duplicates to remove or move.

**Layout**:

- **Summary bar**
  - “Found N duplicate groups · M duplicate files”
  - “Estimated reclaimable space: X GB/TB”

- **List of duplicate groups**
  - Each group row shows:
    - Thumbnail of a representative media file (photo/video)
    - Text like: “5 duplicates · 1.3 GB total”
  - Tap a group to navigate to detail view.

- **Duplicate group detail view**
  - Shows all files in the group:
    - Thumbnail
    - Filename
    - Full path (volume + folder)
    - File size, modified date, possibly checksum
    - Checkboxes or toggles to “Keep” vs “Delete”
  - **Smart selection helpers**:
    - “Select all except newest file”
    - “Select all except one per folder”
  - Bottom bar with:
    - **“Delete selected”** (with confirmation)
    - Optionally **“Move selected to Trash/Archive folder”**

- **Global actions**
  - “Auto-select safe duplicates” (e.g., same file existing in obvious backup locations).
  - Optional: “Export report…” to a CSV or text file.

The review screen is backed by an array of `DuplicateGroup` models provided by the scan engine / view model.

---

## Data Model

We use simple Swift models to represent sources, media files, and groups of duplicates.

```swift
enum MediaType {
    case photo
    case video
}

struct ScanSource: Identifiable {
    enum Kind {
        case localVolume(path: URL)
        case webDAV(name: String, baseURL: URL, username: String, password: String)
    }

    let id: UUID
    var displayName: String
    var kind: Kind
    var isSelected: Bool
}

struct MediaFile: Identifiable, Hashable {
    let id = UUID()
    let url: URL            // For WebDAV, this may represent a remote URL
    let size: Int64
    let modifiedDate: Date?
    let type: MediaType
    // Additional fields (hashes, etc.) can be stored in a database instead.
}

struct DuplicateGroup: Identifiable {
    let id = UUID()
    let files: [MediaFile]
    let totalSize: Int64
}
```

These models are owned by an `ObservableObject` view model that coordinates scanning and UI updates.

---

## Architecture Overview

### High-level components

- **Views**
  - `SourceSelectionView` (in place of the current `ContentView` body)
  - `ScanProgressView`
  - `DuplicateListView`
  - `DuplicateGroupDetailView`

- **View model**
  - `ScanViewModel : ObservableObject`
    - Holds sources, progress, and results.
    - Bridges SwiftUI with the scanning engine.

- **Scanning engine**
  - `ScanEngine`
    - Performs disk enumeration, hashing, grouping.
    - Runs mostly off the main thread, uses async APIs.

### ScanViewModel sketch

Conceptual responsibilities of the view model:

- Track the list of sources with selection state.
- Start, pause, resume, and cancel scans.
- Expose progress and result data to the UI.

Pseudocode outline:

```swift
final class ScanViewModel: ObservableObject {
    @Published var sources: [ScanSource] = []
    @Published var isScanning = false
    @Published var progress: Double? // nil means indeterminate
    @Published var scannedFilesCount: Int = 0
    @Published var foundDuplicateGroups: [DuplicateGroup] = []

    private let scanEngine = ScanEngine()

    func startScan(selectedTypes: [MediaType], filters: ScanFilters) {
        isScanning = true
        scannedFilesCount = 0
        progress = nil
        foundDuplicateGroups = []

        Task {
            for await update in scanEngine.scan(
                sources: sources.filter { $0.isSelected },
                types: selectedTypes,
                filters: filters
            ) {
                await MainActor.run {
                    self.scannedFilesCount = update.scannedFiles
                    self.progress = update.progress
                    self.foundDuplicateGroups = update.duplicateGroups
                    if update.isFinished {
                        self.isScanning = false
                    }
                }
            }
        }
    }

    func cancelScan() {
        scanEngine.cancel()
        isScanning = false
    }
}
```

`ScanEngine.scan` can be modeled as an `AsyncSequence` emitting periodic progress updates.

---

## Duplicate Detection Algorithm (TB-scale)

The main challenge is TB-scale data. A naive “hash every file fully” approach is too slow. Instead, use a **multi-phase** strategy:

1. **Enumerate media files**
2. **Group by size**
3. **Partial hashing for candidates**
4. **Full hashing for strong candidates**
5. **Build duplicate groups**

### Phase 0: Enumerate candidates (media only)

For each selected source:

- Recursively walk the directory tree (e.g., using `FileManager.enumerator(at:includingPropertiesForKeys:options:errorHandler:)` for local and external volumes).
- Filter to **media file extensions**:
  - Photos: `jpg`, `jpeg`, `png`, `heic`, `heif`, `tiff`, `cr2`, `nef`, etc.
  - Videos: `mp4`, `mov`, `m4v`, `avi`, `mkv`, etc.
- Apply size filters:
  - Skip files smaller or larger than configured thresholds.

Each discovered file is modeled as a `MediaFile`. For WebDAV sources, a similar enumeration is done using HTTP/WebDAV APIs.

### Phase 1: Group by size

Assumption: True duplicates must have the **same file size**.

- Group files by `size`:
  - Map `size -> [MediaFile]`.
  - Any `size` bucket with a single file cannot contain duplicates and can be dropped.

To handle TB-scale data:

- Rather than keeping all `MediaFile` objects in memory, consider:
  - Persisting file metadata to a local database (e.g., SQLite/Core Data).
  - Using a query like `SELECT size, COUNT(*) FROM files GROUP BY size HAVING COUNT(*) > 1` to identify candidate sizes.

### Phase 2: Partial hash (fast fingerprint)

For each size group with more than one file:

- Compute a **partial content hash**:
  - Read the first few KB of the file (e.g., 4 KB).
  - Read the last few KB of the file (e.g., 4 KB) if the file is large enough.
  - Combine those bytes and hash them (SHA-256 or a fast hash).
- Group by `(size, partialHash)`:
  - Any group with only one file is unlikely to be a duplicate and can be discarded.

This significantly reduces the number of files that require full hashing.

For WebDAV or slow network storage:

- Tune the partial hash strategy:
  - Possibly larger chunk size for better discrimination.
  - Avoid reading full content unless absolutely necessary.

### Phase 3: Full hash (confirm duplicates)

For the remaining candidate groups that still look large or ambiguous:

- Compute a **full hash** for each file:
  - Read the entire file in streaming fashion (buffered reads).
  - Hash using SHA-256 or another reliable algorithm.
- Group by full hash:
  - Each full-hash group of size ≥ 2 is a **confirmed duplicate set**.

Optionally:

- Make full hashing conditional:
  - E.g., for very large network files, compute full hash only if the user asks for strict validation or when partial hash collisions are suspected.

### Phase 4: Build DuplicateGroup models

For each full-hash group:

- Create a `DuplicateGroup`:
  - `files: [MediaFile]` representing all duplicates.
  - `totalSize: sum(file.size)` for storage reclaim estimation.

These groups are then:

- Presented to the user in the Duplicates review screen.
- Optionally persisted to disk for later review without re-scanning.

---

## Deletion & Safety

- **Deletion strategy**:
  - Prefer moving files to the system Trash if possible (using appropriate APIs).
  - For remote/WebDAV files, provide clear warnings if “delete” is permanent.
- **User control**:
  - Require explicit confirmation before deleting.
  - Default to **keeping at least one copy** in each duplicate group.
  - Offer presets for selection (e.g., keep newest, keep in preferred volume, etc.).

---

## Implementation Phases

Suggested incremental implementation plan:

1. **UI scaffolding**
   - Replace `ContentView` with a `NavigationStack` and a basic `SourceSelectionView`.
   - Add models for `ScanSource`, `MediaType`, `ScanViewModel` with mocked scan results.

2. **Local disk scanning (no WebDAV yet)**
   - Automatically detect volumes under `/Volumes`.
   - Allow selecting one or more volumes.
   - Enumerate files on those volumes and filter to media extensions.
   - Group by size and show simple stats.

3. **Partial + full hashing engine**
   - Implement `ScanEngine` with:
     - Enumeration
     - Group-by-size
     - Partial hashing
     - Full hashing for candidates
   - Integrate with `ScanViewModel` using async updates for progress.

4. **Duplicates review UI**
   - Implement `DuplicateListView` and `DuplicateGroupDetailView`.
   - Enable basic selection and deletion for local files.

5. **Persistence & resume**
   - Store scan metadata and duplicate groups in a local database.
   - Allow resuming or reusing previous results without full re-scan.

6. **WebDAV support**
   - Add configuration UI for WebDAV servers.
   - Implement enumeration and content reads over WebDAV.
   - Reuse the same grouping and hashing logic as for local files.

7. **Refinement and performance tuning**
   - Optimize IO patterns (buffer sizes, concurrency).
   - Add more advanced selection heuristics for safe auto-selection.
   - Improve thumbnails and media previews where appropriate.

This design keeps the UI straightforward and the scanning pipeline scalable, while leaving room to iterate on performance, heuristics, and advanced features as the app matures.

