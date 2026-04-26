# Multi-Photo Posting — Design Spec
**Date:** 2026-04-26
**Status:** Approved

## Summary
Allow each diary entry to carry multiple photos (or a single photo), displayed Facebook-style in a collage. Text caption remains a single field per post.

## Data Model

```
Old entry (unchanged, still works):
  { photoURL: "https://...", text, date, child, createdAt, updatedAt }

New entry:
  { photos: ["url1", "url2", ...], text, date, child, createdAt, updatedAt }
```

**Read compatibility shim** (used everywhere photos are read):
```js
const photoList = entry.photos?.length ? entry.photos : entry.photoURL ? [entry.photoURL] : [];
```

No Firestore migration. Old entries with `photoURL` display via the shim. New entries written with `photos` array.

## Composer Upload UI

- Single dashed-border tap zone with `<input type="file" accept="image/*" multiple>` (replaces existing single-file input)
- After first selection: zone replaced by thumbnail grid
  - Each thumbnail: 80×80px, object-fit cover, `×` remove button
  - Last cell: `+` button that re-opens the file picker (appends to existing selection)
- In-memory state: `selectedPhotos = []` array of `File` objects
- No per-photo upload on add — all held in memory until save

## Save Logic (Option B — batch upload on save)

1. User taps "Save Entry"
2. All `File` objects in `selectedPhotos` uploaded to ImgBB in parallel (`Promise.all`)
3. Collected URLs written to Firestore as `photos: [url1, url2, ...]`
4. Progress bar shows aggregate progress (single bar, not per-photo)
5. If any upload fails: surface error, do not save to Firestore, re-enable Save button

For edit: existing `photos` array pre-populated as URL strings in the thumbnail grid. User can remove existing photos or add new `File` objects. On save, existing URLs kept as-is; new `File` objects uploaded.

## Display Collage

CSS grid collage, Facebook-style layout based on photo count:

| Count | Layout |
|-------|--------|
| 1 | Full width |
| 2 | Side by side (50/50) |
| 3 | Large left (full height) + 2 stacked right |
| 4 | 2×2 grid |
| 5+ | Large left (full height) + 2×2 right; 5th cell has `+N` overlay if count > 5 |

- Max 5 cells shown; if `photoList.length > 5`, 5th cell shows the 5th photo with `+N` overlay
- Clicking any cell opens lightbox at that photo index
- Lightbox arrow navigation shows all photos in the post (not just the 5 visible)

## Lightbox

Existing lightbox already has prev/next navigation. Change: populate it with the full `photoList` of the clicked post, starting at the clicked index. Currently it navigates across all posts — this behavior is preserved for single-photo old entries; for multi-photo posts the arrows navigate within the post first (TBD: or continue cross-post navigation — simplest is cross-post, all photos globally).

**Decision:** Keep existing global lightbox array (all `.entry-photo` images across all posts). Multi-photo posts contribute multiple entries to this global array. No change to lightbox logic needed.

## Backward Compatibility

- Old entries: `photoURL` string → rendered via shim as 1-photo collage (full width, identical to current)
- Edit old entry: loads `photoURL` as a single URL thumbnail; on re-save writes `photos: [url]` array
- No data migration script needed

## Out of Scope

- Video support
- Photo reordering in composer
- Per-photo captions
- ImgBB delete on cancel (API not available on free tier)
