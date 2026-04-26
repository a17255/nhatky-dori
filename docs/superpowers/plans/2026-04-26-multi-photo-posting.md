# Multi-Photo Posting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow each diary entry to hold multiple photos displayed in a Facebook-style collage, while keeping old single-`photoURL` entries working unchanged.

**Architecture:** Single `index.html` — add CSS for collage grid + composer thumbnails, replace single-photo state with `selectedPhotos[]` array, upload all new photos in parallel on save, and render entries with a `buildCollage()` helper that uses a backward-compat shim.

**Tech Stack:** Vanilla JS (ES2022), CSS Grid, Firebase Firestore, ImgBB API, no build step.

---

## File Map

| File | What changes |
|------|-------------|
| `index.html` | All changes (CSS, HTML, JS) — single-file app |

---

## Task 1: Add CSS for composer thumbnail grid

**Files:**
- Modify: `index.html:166` (after `.photo-remove` rule, before `.submit-btn`)

- [ ] **Step 1: Insert composer CSS after line 166**

Add the following block immediately after the `.photo-remove{...}` rule (line 166) and before the `.submit-btn{...}` rule (line 167):

```css
.composer-photos{display:flex;flex-wrap:wrap;gap:8px;margin-top:8px}
.composer-thumb{position:relative;width:80px;height:80px;border-radius:8px;overflow:hidden;flex-shrink:0}
.composer-thumb img{width:100%;height:100%;object-fit:cover;display:block}
.thumb-remove{position:absolute;top:2px;right:2px;background:rgba(0,0,0,0.7);color:#fff;border:none;width:22px;height:22px;border-radius:50%;cursor:pointer;font-size:0.85rem;display:flex;align-items:center;justify-content:center;padding:0}
.composer-add-btn{width:80px;height:80px;border:2px dashed var(--border);border-radius:8px;display:flex;align-items:center;justify-content:center;cursor:pointer;color:var(--text2);font-size:1.5rem;background:transparent;flex-shrink:0}
.composer-add-btn:hover{border-color:var(--pink);color:var(--pink)}
```

- [ ] **Step 2: Verify file saved, no syntax error**

Open `index.html` in a browser. DevTools Console should show no CSS errors.

---

## Task 2: Add CSS for Facebook collage display

**Files:**
- Modify: `index.html` — add after the composer CSS block from Task 1

- [ ] **Step 1: Insert collage CSS immediately after the composer CSS added in Task 1**

```css
.photo-collage{display:grid;gap:2px;margin-top:12px;border-radius:var(--radius-sm);overflow:hidden;max-height:500px}
.collage-1{grid-template-columns:1fr}
.collage-2{grid-template-columns:1fr 1fr;grid-template-rows:250px}
.collage-3{grid-template-columns:2fr 1fr;grid-template-rows:120px 120px}
.collage-3 .collage-cell:first-child{grid-row:1/3}
.collage-4{grid-template-columns:1fr 1fr;grid-template-rows:150px 150px}
.collage-5{grid-template-columns:2fr 1fr 1fr;grid-template-rows:150px 150px}
.collage-5 .collage-cell:first-child{grid-row:1/3}
.collage-cell{position:relative;overflow:hidden}
.collage-cell img{width:100%;height:100%;object-fit:cover;display:block;cursor:pointer}
.collage-1 .collage-cell img{max-height:500px;height:auto}
.collage-plus-overlay{position:absolute;inset:0;background:rgba(0,0,0,0.5);display:flex;align-items:center;justify-content:center;color:#fff;font-size:1.6rem;font-weight:700;cursor:pointer;pointer-events:auto}
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. No CSS errors in console.

---

## Task 3: Update HTML — make file input `multiple`, replace upload zone with composer container

**Files:**
- Modify: `index.html:326-333` (the `<input type="file">` and `<div class="photo-upload">` in the add modal)

- [ ] **Step 1: Replace the file input and photo upload zone**

Find this block (lines 326–333):
```html
        <input type="file" id="photoFile" accept="image/*" style="display:none">
        <div class="photo-upload" id="photoUpload" onclick="document.getElementById('photoFile').click()">
          <div id="uploadPlaceholder">
            <div class="upload-icon">📷</div>
            <div class="upload-text">Tap to add a photo</div>
            <div class="upload-hint">From gallery or camera</div>
          </div>
        </div>
```

Replace with:
```html
        <input type="file" id="photoFile" accept="image/*" multiple style="display:none">
        <div id="photoComposer">
          <div class="photo-upload" onclick="document.getElementById('photoFile').click()">
            <div class="upload-icon">📷</div>
            <div class="upload-text">Tap to add photos</div>
            <div class="upload-hint">Select one or multiple</div>
          </div>
        </div>
```

- [ ] **Step 2: Verify modal still opens in browser**

Click the `+` button (admin mode). The modal should open with the photo upload zone visible.

---

## Task 4: Replace JS state + composer logic

**Files:**
- Modify: `index.html:512` — change `selectedPhoto`
- Modify: `index.html:666-687` — replace entire Photo Upload UI section

- [ ] **Step 1: Change state variable at line 512**

Find:
```js
let selectedPhoto = null;
```
Replace with:
```js
let selectedPhotos = []; // { type: 'file'|'url', data: File|string, preview: string }
```

- [ ] **Step 2: Replace the Photo Upload UI section (lines 666–687)**

Find this entire block:
```js
/* ── Photo Upload UI ── */
const photoFile = document.getElementById('photoFile');
const photoUpload = document.getElementById('photoUpload');

photoFile.addEventListener('change', () => {
  const file = photoFile.files[0];
  if (!file) return;
  selectedPhoto = file;
  const reader = new FileReader();
  reader.onload = (e) => {
    photoUpload.classList.add('has-photo');
    photoUpload.innerHTML = `<img class="photo-preview" src="${e.target.result}" alt="preview"><button class="photo-remove" onclick="event.stopPropagation();removePhoto()">&times;</button>`;
  };
  reader.readAsDataURL(file);
});

function removePhoto() {
  selectedPhoto = null;
  photoFile.value = '';
  photoUpload.classList.remove('has-photo');
  photoUpload.innerHTML = `<div id="uploadPlaceholder"><div class="upload-icon">📷</div><div class="upload-text">Tap to add a photo</div><div class="upload-hint">From gallery or camera</div></div>`;
}
```

Replace with:
```js
/* ── Photo Composer ── */
const photoFile = document.getElementById('photoFile');

photoFile.addEventListener('change', () => {
  const files = Array.from(photoFile.files);
  if (!files.length) return;
  files.forEach(file => {
    selectedPhotos.push({ type: 'file', data: file, preview: URL.createObjectURL(file) });
  });
  photoFile.value = '';
  renderComposer();
});

function renderComposer() {
  const container = document.getElementById('photoComposer');
  if (selectedPhotos.length === 0) {
    container.innerHTML = `<div class="photo-upload" onclick="document.getElementById('photoFile').click()"><div class="upload-icon">📷</div><div class="upload-text">Tap to add photos</div><div class="upload-hint">Select one or multiple</div></div>`;
    return;
  }
  let html = '<div class="composer-photos">';
  selectedPhotos.forEach((item, i) => {
    html += `<div class="composer-thumb"><img src="${item.preview}" alt="photo ${i+1}"><button class="thumb-remove" onclick="removeComposerPhoto(${i})">&times;</button></div>`;
  });
  html += `<button class="composer-add-btn" onclick="document.getElementById('photoFile').click()">+</button>`;
  html += '</div>';
  container.innerHTML = html;
}

function removeComposerPhoto(i) {
  if (selectedPhotos[i].type === 'file') URL.revokeObjectURL(selectedPhotos[i].preview);
  selectedPhotos.splice(i, 1);
  renderComposer();
}

function resetComposer() {
  selectedPhotos.forEach(p => { if (p.type === 'file') URL.revokeObjectURL(p.preview); });
  selectedPhotos = [];
  renderComposer();
}
```

- [ ] **Step 3: Verify in browser**

Open modal, tap the upload zone, select multiple photos. Thumbnails should appear with `×` buttons and a `+` button.

---

## Task 5: Update `openAddModal` to use `resetComposer` and load existing photos

**Files:**
- Modify: `index.html:689-713` — the `openAddModal` function

- [ ] **Step 1: Replace `removePhoto()` call and the photo-loading block**

Find in `openAddModal`:
```js
  removePhoto();
  if (id) {
    const entry = entries.find(e => e.id === id);
    if (entry) {
      document.getElementById('entryChild').value = entry.child || 'Dori';
      document.getElementById('entryDate').value = entry.date;
      document.getElementById('entryText').value = entry.text || '';
      if (entry.photoURL) {
        photoUpload.classList.add('has-photo');
        photoUpload.innerHTML = `<img class="photo-preview" src="${entry.photoURL}" alt="preview"><button class="photo-remove" onclick="event.stopPropagation();removePhoto()">&times;</button>`;
      }
    }
  }
```

Replace with:
```js
  resetComposer();
  if (id) {
    const entry = entries.find(e => e.id === id);
    if (entry) {
      document.getElementById('entryChild').value = entry.child || 'Dori';
      document.getElementById('entryDate').value = entry.date;
      document.getElementById('entryText').value = entry.text || '';
      const photoList = entry.photos?.length ? entry.photos : entry.photoURL ? [entry.photoURL] : [];
      photoList.forEach(url => selectedPhotos.push({ type: 'url', data: url, preview: url }));
      renderComposer();
    }
  }
```

- [ ] **Step 2: Remove the `const photoUpload` line (now unused)**

Find and delete this line (around line 668):
```js
const photoUpload = document.getElementById('photoUpload');
```

- [ ] **Step 3: Verify edit flow in browser**

Open an existing entry for editing. Its photo should appear as a thumbnail in the composer.

---

## Task 6: Update `closeAddModal` to call `resetComposer`

**Files:**
- Modify: `index.html:715-718`

- [ ] **Step 1: Update `closeAddModal`**

Find:
```js
function closeAddModal() {
  document.getElementById('addModal').classList.remove('active');
  editingId = null;
}
```

Replace with:
```js
function closeAddModal() {
  document.getElementById('addModal').classList.remove('active');
  editingId = null;
  resetComposer();
}
```

---

## Task 7: Replace `saveEntry` with multi-photo batch upload

**Files:**
- Modify: `index.html:721-781`

- [ ] **Step 1: Replace the entire `saveEntry` function**

Find the entire block from `/* ── Save Entry ── */` through the closing `}` of `saveEntry` (lines 720–781).

Replace with:
```js
/* ── Save Entry ── */
async function saveEntry() {
  const date = document.getElementById('entryDate').value;
  const text = document.getElementById('entryText').value.trim();
  if (!date) { toast('Please pick a date'); return; }
  if (!text && selectedPhotos.length === 0 && !editingId) { toast('Add a photo or some text'); return; }

  const btn = document.getElementById('submitBtn');
  btn.disabled = true;
  btn.innerHTML = '<span class="spinner"></span>';

  try {
    const newFiles = selectedPhotos.filter(p => p.type === 'file');
    const existingURLs = selectedPhotos.filter(p => p.type === 'url').map(p => p.data);

    let newURLs = [];
    if (newFiles.length > 0) {
      document.getElementById('uploadProgress').style.display = '';
      const fill = document.getElementById('progressFill');
      const ptext = document.getElementById('progressText');
      fill.style.width = '10%';
      ptext.textContent = `Uploading ${newFiles.length} photo${newFiles.length > 1 ? 's' : ''}...`;

      newURLs = await Promise.all(newFiles.map(async (item) => {
        const base64 = await new Promise(resolve => {
          const reader = new FileReader();
          reader.onload = () => resolve(reader.result.split(',')[1]);
          reader.readAsDataURL(item.data);
        });
        const formData = new FormData();
        formData.append('key', IMGBB_API_KEY);
        formData.append('image', base64);
        const resp = await fetch('https://api.imgbb.com/1/upload', { method: 'POST', body: formData });
        const result = await resp.json();
        if (!result.success) throw new Error('Photo upload failed');
        return result.data.image?.url || result.data.display_url;
      }));

      fill.style.width = '100%';
      ptext.textContent = 'Done!';
    }

    const photos = [...existingURLs, ...newURLs];
    const child = document.getElementById('entryChild').value;
    const data = { date, text, photos, child, updatedAt: firebase.firestore.FieldValue.serverTimestamp() };

    if (editingId) {
      await db.collection('entries').doc(editingId).update(data);
      toast('Entry updated!');
    } else {
      data.createdAt = firebase.firestore.FieldValue.serverTimestamp();
      await db.collection('entries').add(data);
      toast('Entry saved!');
    }

    closeAddModal();
    loadEntries();
  } catch (err) {
    console.error(err);
    toast('Error: ' + err.message);
    btn.disabled = false;
    btn.textContent = 'Save Entry';
    document.getElementById('uploadProgress').style.display = 'none';
  }
}
```

- [ ] **Step 2: Test save in browser**

Create a new entry with 2–3 photos and some text. Click Save. Entry should appear in the timeline with a collage (after Task 8 is done).

---

## Task 8: Add `buildCollage` helper and update `renderTimeline`

**Files:**
- Modify: `index.html` — add `buildCollage` function before `renderTimeline`
- Modify: `index.html:854-856` — replace single `<img>` with collage

- [ ] **Step 1: Add `buildCollage` function**

Find the comment `/* ── Load & Render Entries ──` and insert the following function immediately before it:

```js
function buildCollage(photoList, entry) {
  const count = photoList.length;
  if (count === 0) return '';
  const shown = photoList.slice(0, 5);
  const extra = count - 5;
  const cls = count === 1 ? 'collage-1' : count === 2 ? 'collage-2' : count === 3 ? 'collage-3' : count === 4 ? 'collage-4' : 'collage-5';
  const text = escapeHtml(entry.text || '');
  const date = escapeHtml(formatDate(entry.date) || '');
  let html = `<div class="photo-collage ${cls}">`;
  shown.forEach((url, i) => {
    const isLastShown = i === 4 && extra > 0;
    html += `<div class="collage-cell"><img class="entry-photo" src="${url}" alt="photo" loading="lazy" data-full="${url}" data-text="${text}" data-date="${date}">`;
    if (isLastShown) html += `<div class="collage-plus-overlay">+${extra}</div>`;
    html += `</div>`;
  });
  html += `</div>`;
  return html;
}
```

- [ ] **Step 2: Update the photo rendering block in `renderTimeline`**

Find (lines ~854-856):
```js
    if (entry.photoURL) {
      html += `<img class="entry-photo" src="${entry.photoURL}" alt="diary photo" loading="lazy" data-full="${entry.photoURL}" data-text="${escapeHtml(entry.text || '')}" data-date="${escapeHtml(formatDate(entry.date) || '')}">`;
    }
```

Replace with:
```js
    const photoList = entry.photos?.length ? entry.photos : entry.photoURL ? [entry.photoURL] : [];
    if (photoList.length) html += buildCollage(photoList, entry);
```

- [ ] **Step 3: Verify in browser**

Reload. Old entries with `photoURL` should display as single full-width images (unchanged). New multi-photo entries should show a collage grid.

---

## Task 9: Update lightbox click handler to support `+N` overlay

**Files:**
- Modify: `index.html:1110-1118`

- [ ] **Step 1: Update the timeline click handler**

Find:
```js
document.getElementById('timeline').addEventListener('click', e => {
  const img = e.target.closest('.entry-photo');
  if (!img || !img.dataset.full) return;
  // Build list of all visible photos in DOM order, find clicked index
  const all = Array.from(document.querySelectorAll('.entry-photo'));
  const list = all.map(el => ({ url: el.dataset.full, text: el.dataset.text || '', date: el.dataset.date || '' }));
  const idx = all.indexOf(img);
  openLightbox(list, idx);
});
```

Replace with:
```js
document.getElementById('timeline').addEventListener('click', e => {
  const all = Array.from(document.querySelectorAll('.entry-photo'));
  const list = all.map(el => ({ url: el.dataset.full, text: el.dataset.text || '', date: el.dataset.date || '' }));

  const overlay = e.target.closest('.collage-plus-overlay');
  if (overlay) {
    const cell = overlay.closest('.collage-cell');
    const img = cell?.querySelector('.entry-photo');
    if (img) openLightbox(list, all.indexOf(img));
    return;
  }

  const img = e.target.closest('.entry-photo');
  if (!img || !img.dataset.full) return;
  openLightbox(list, all.indexOf(img));
});
```

- [ ] **Step 2: Verify lightbox in browser**

Click any photo in a multi-photo collage — lightbox should open on that photo. Click `+N` overlay — lightbox should open on the 5th photo. Arrow keys/buttons should navigate through all photos globally.

---

## Task 10: Commit

- [ ] **Step 1: Commit all changes**

```bash
git add index.html
git commit -m "feat: multi-photo posting with Facebook-style collage"
```

- [ ] **Step 2: Push to GitHub**

```bash
git push origin main
```

- [ ] **Step 3: Verify GitHub Pages**

Open `https://a17255.github.io/nhatky-dori/` (after ~1 min deploy). Test: create new entry with 3 photos, verify collage renders. Open old entry — single photo should still display correctly.

---

## Self-Review Checklist

- [x] CSS collage: 1/2/3/4/5+ layouts all defined ✓
- [x] Composer thumbnails: add/remove/reset all covered ✓
- [x] `selectedPhoto` (old) fully replaced with `selectedPhotos[]` ✓
- [x] `removePhoto` replaced with `resetComposer` everywhere ✓
- [x] `openAddModal` loads both `photos[]` and legacy `photoURL` ✓
- [x] `saveEntry` parallel upload + Firestore write ✓
- [x] `buildCollage` backward-compat shim applied ✓
- [x] Lightbox `+N` overlay click handled ✓
- [x] `photoUpload` const removed (element no longer in DOM) ✓
- [x] No TBD/TODO placeholders ✓
