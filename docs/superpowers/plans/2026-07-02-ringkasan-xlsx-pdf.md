# Ringkasan/XLSX Fix & Download PDF Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the on-screen "Ringkasan Data Cek Fisik" and the exported `.xlsx` share one labeled-rows builder (adding the missing Selisih dimension values along the way), and add a "Download .pdf" button that exports the same data as a formatted PDF, per the approved spec at `docs/superpowers/specs/2026-07-02-ringkasan-xlsx-pdf-design.md`.

**Architecture:** Single static HTML file (`index.html`), vanilla JS in `<script>` tags, no build step, no test framework, no package.json. The core change is extracting a pure function `buildLabeledRows(d)` (takes the object returned by the existing `collectData()`, returns an array of `{title, fields: [[label, value], ...]}` groups) that both `renderSummary()` and `downloadXLSX()` consume, plus a new `downloadPDF()`/`loadJsPDF()` pair that mirrors the existing `downloadXLSX()`/`loadXLSX()` lazy-CDN-load pattern.

**Tech Stack:** Vanilla HTML/CSS/JS. `XLSX` (SheetJS, already loaded from `cdn.sheetjs.com`) stays as-is. New: `jsPDF` + `jsPDF-autoTable`, loaded from `cdnjs.cloudflare.com` only when the PDF button is first clicked. Verification uses the project's established method — the `preview_*` MCP tools (`preview_start` with `.claude/launch.json` config `cek-fisik`, `preview_eval`, `preview_console_logs`, `preview_stop`) — there is no automated test runner in this repo.

## Global Constraints

- `buildLabeledRows(d)` is the single source of truth for group titles and human-readable labels; both `renderSummary()` and `downloadXLSX()` must call it instead of building their own label lists.
- The "Dimensi Bangunan" group in `buildLabeledRows(d)` includes 3 new rows: `Selisih Panjang`, `Selisih Lebar`, `Selisih Luas`, reading `d['Fisik_Selisih_Panjang']`, `d['Fisik_Selisih_Lebar']`, `d['Fisik_Selisih_Luas']` (these keys already exist in `collectData()` — no change needed there).
- `downloadXLSX()`'s row list gets a group-separator row `{ Kolom: '— <title> —', Nilai: '' }` before each group's fields.
- PDF library URLs (exact, from the spec): `https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.2/jspdf.umd.min.js` and `https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.8.4/jspdf.plugin.autotable.min.js`, loaded lazily (only on first click of the PDF button), matching the existing `loadXLSX()` lazy-load pattern.
- PDF filename convention: `CekFisik_<Desa>_<Tanggal>.pdf` — identical pattern to the existing XLSX filename (`Kelurahan/Desa` with spaces replaced by `_`, `Tanggal Kunjungan` with dashes stripped, fallback to today's date if empty).
- New PDF button uses classes `btn btn-download btn-download-pdf` (reuses existing `.btn-download` CSS, no new stylesheet rules needed beyond an inline `margin-top:8px`).
- Only `index.html` is modified.

---

### Task 1: Extract `buildLabeledRows(d)` and add Selisih rows to the summary

**Files:**
- Modify: `index.html:559-598` (the `renderSummary()` function)

**Interfaces:**
- Produces: `buildLabeledRows(d)` — takes the object returned by `collectData()` (already defined, has keys `No`, `Tanggal Kunjungan`, ..., `Fisik_Selisih_Panjang`, `Fisik_Selisih_Lebar`, `Fisik_Selisih_Luas`, `Klg_<id>_Jumlah` for every item in `KELENGKAPAN_ITEMS`, `Knd_<id>_Jumlah` for every item in `KENDARAAN_ITEMS`, `Catatan_Lapangan`) — returns `Array<{title: string, fields: Array<[string, string]>}>`, 5 groups: `Identitas`, `Dimensi Bangunan`, `Kelengkapan`, `Kendaraan`, `Catatan Lapangan`.
- Consumes: `KELENGKAPAN_ITEMS` and `KENDARAAN_ITEMS` (already defined earlier in `index.html`, same shapes used throughout the file: `Array<{category, items: Array<{id, label}>}>` and `Array<{id, label}>` respectively).

- [ ] **Step 1: Confirm current (pre-change) summary is missing Selisih — baseline check**

Start the preview server:

Run `mcp__Claude_Preview__preview_start` with `name: "cek-fisik"`.

Then run `mcp__Claude_Preview__preview_eval` with:
```js
(function(){
  showStep(2);
  document.getElementById('rel_pjg').value = '15';
  document.getElementById('rel_lbr').value = '25';
  hitungDimensiGerai();
  showStep(4);
  renderSummary();
  const text = document.getElementById('summary-content').textContent;
  return { hasSelisihPanjang: text.includes('Selisih Panjang'), selPjgSpan: document.getElementById('sel_pjg').textContent };
})();
```
Expected (baseline, confirms Selisih is not yet in the summary): `{"hasSelisihPanjang": false, "selPjgSpan": "+5.00"}` — the span itself is computed correctly by `hitungDimensiGerai()`, it just isn't surfaced in the summary yet.

- [ ] **Step 2: Replace `renderSummary()` with `buildLabeledRows()` + a slimmer `renderSummary()`**

Replace:
```js
    function renderSummary() {
      const d = collectData();
      const groups = [
        { title: 'Identitas', fields: [
          ['No / Kode', d['No']||'-'], ['Tanggal Kunjungan', d['Tanggal Kunjungan']||'-'],
          ['Provinsi', d['Provinsi']||'-'], ['Kabupaten/Kota', d['Kabupaten/Kota']||'-'],
          ['Kecamatan', d['Kecamatan']||'-'], ['Kelurahan/Desa', d['Kelurahan/Desa']||'-'],
          ['Nama KDKMP', d['Nama KDKMP']||'-'],
          ['Nama Pewawancara', d['Nama Pewawancara']||'-'],
          ['Nama Responden', d['Nama Responden']||'-'],
          ['Nomor HP Responden', d['Nomor HP Responden']||'-'],
        ]},
        { title: 'Dimensi Bangunan', fields: [
          ['Bangunan (P\xd7L)', d['Fisik_Panjang']&&d['Fisik_Lebar'] ? `${d['Fisik_Panjang']} m \xd7 ${d['Fisik_Lebar']} m = ${d['Fisik_Luas']} m\xb2` : '-'],
          ['Lantai Keramik', d['Fisik_LantaiKeramik']||'-'], ['Lantai Gudang', d['Fisik_LantaiGudang']||'-'],
          ['Dinding', d['Fisik_Dinding']||'-'], ['Atap', d['Fisik_Atap']||'-'],
          ['Pintu', d['Fisik_Pintu']||'-'], ['Jendela / Kaca', d['Fisik_Jendela']||'-'],
        ]},
        { title: 'Kelengkapan', fields: KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id, label}) => {
          const jumlah = d[`Klg_${id}_Jumlah`];
          return [label, jumlah ? jumlah : '-'];
        })},
        { title: 'Kendaraan', fields: KENDARAAN_ITEMS.map(({id, label}) => {
          const jumlah = d[`Knd_${id}_Jumlah`];
          return [label, jumlah ? jumlah : '-'];
        })},
        { title: 'Catatan Lapangan', fields: [
          ['Catatan', d['Catatan_Lapangan']||'-'],
        ]},
      ];
      document.getElementById('summary-content').innerHTML = groups.map(g => `
        <div class="summary-group">
          <div class="summary-group-title">${g.title}</div>
          ${g.fields.map(([label, value]) => `
            <div class="summary-row">
              <span class="summary-label">${label}</span>
              <span class="summary-value">${value}</span>
            </div>`).join('')}
        </div>`).join('');
    }
```
with:
```js
    function buildLabeledRows(d) {
      return [
        { title: 'Identitas', fields: [
          ['No / Kode', d['No']||'-'], ['Tanggal Kunjungan', d['Tanggal Kunjungan']||'-'],
          ['Provinsi', d['Provinsi']||'-'], ['Kabupaten/Kota', d['Kabupaten/Kota']||'-'],
          ['Kecamatan', d['Kecamatan']||'-'], ['Kelurahan/Desa', d['Kelurahan/Desa']||'-'],
          ['Nama KDKMP', d['Nama KDKMP']||'-'],
          ['Nama Pewawancara', d['Nama Pewawancara']||'-'],
          ['Nama Responden', d['Nama Responden']||'-'],
          ['Nomor HP Responden', d['Nomor HP Responden']||'-'],
        ]},
        { title: 'Dimensi Bangunan', fields: [
          ['Bangunan (P\xd7L)', d['Fisik_Panjang']&&d['Fisik_Lebar'] ? `${d['Fisik_Panjang']} m \xd7 ${d['Fisik_Lebar']} m = ${d['Fisik_Luas']} m\xb2` : '-'],
          ['Selisih Panjang', d['Fisik_Selisih_Panjang']||'-'],
          ['Selisih Lebar', d['Fisik_Selisih_Lebar']||'-'],
          ['Selisih Luas', d['Fisik_Selisih_Luas']||'-'],
          ['Lantai Keramik', d['Fisik_LantaiKeramik']||'-'], ['Lantai Gudang', d['Fisik_LantaiGudang']||'-'],
          ['Dinding', d['Fisik_Dinding']||'-'], ['Atap', d['Fisik_Atap']||'-'],
          ['Pintu', d['Fisik_Pintu']||'-'], ['Jendela / Kaca', d['Fisik_Jendela']||'-'],
        ]},
        { title: 'Kelengkapan', fields: KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id, label}) => {
          const jumlah = d[`Klg_${id}_Jumlah`];
          return [label, jumlah ? jumlah : '-'];
        })},
        { title: 'Kendaraan', fields: KENDARAAN_ITEMS.map(({id, label}) => {
          const jumlah = d[`Knd_${id}_Jumlah`];
          return [label, jumlah ? jumlah : '-'];
        })},
        { title: 'Catatan Lapangan', fields: [
          ['Catatan', d['Catatan_Lapangan']||'-'],
        ]},
      ];
    }

    function renderSummary() {
      const groups = buildLabeledRows(collectData());
      document.getElementById('summary-content').innerHTML = groups.map(g => `
        <div class="summary-group">
          <div class="summary-group-title">${g.title}</div>
          ${g.fields.map(([label, value]) => `
            <div class="summary-row">
              <span class="summary-label">${label}</span>
              <span class="summary-value">${value}</span>
            </div>`).join('')}
        </div>`).join('');
    }
```

- [ ] **Step 3: Reload and verify the summary now shows Selisih**

Run `mcp__Claude_Preview__preview_eval` with `expression: "window.location.reload()"`, then:
```js
(function(){
  showStep(2);
  document.getElementById('rel_pjg').value = '15';
  document.getElementById('rel_lbr').value = '25';
  hitungDimensiGerai();
  showStep(4);
  renderSummary();
  const text = document.getElementById('summary-content').textContent;
  return {
    hasSelisihPanjang: text.includes('Selisih Panjang'),
    hasSelisihLebar: text.includes('Selisih Lebar'),
    hasSelisihLuas: text.includes('Selisih Luas'),
    selPjgSpan: document.getElementById('sel_pjg').textContent,
    selLbrSpan: document.getElementById('sel_lbr').textContent,
    selLuasSpan: document.getElementById('sel_luas').textContent
  };
})();
```
Expected: `{"hasSelisihPanjang": true, "hasSelisihLebar": true, "hasSelisihLuas": true, "selPjgSpan": "+5.00", "selLbrSpan": "+5.00", "selLuasSpan": "+225.00"}`.

Check for console errors:

Run `mcp__Claude_Preview__preview_console_logs` with `level: "error"`.
Expected: `No console logs.`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: ekstrak buildLabeledRows dan tampilkan Selisih di ringkasan"
```

---

### Task 2: Make `downloadXLSX()` use `buildLabeledRows()` with human-readable labels

**Files:**
- Modify: `index.html:600-612` (the `downloadXLSX()` function)

**Interfaces:**
- Consumes: `buildLabeledRows(d)` from Task 1 (`Array<{title, fields: Array<[string, string]>}>`), `collectData()` (already defined), `XLSX` global (loaded by the already-existing `loadXLSX()`).
- Produces: `downloadXLSX()` — no change to its own signature (no params, no return value, called from the existing "↓ Download .xlsx" button's `onclick`).

- [ ] **Step 1: Confirm the current (pre-change) row shape uses raw keys — baseline check**

With the preview server still running from Task 1 (or start again with `mcp__Claude_Preview__preview_start`, `name: "cek-fisik"`), run:
```js
(function(){
  const d = collectData();
  const rows = Object.entries(d).map(([k,v]) => ({Kolom: k, Nilai: v}));
  return { firstKolom: rows[0].Kolom, hasRawKlgKey: rows.some(r => r.Kolom.startsWith('Klg_')) };
})();
```
Expected (baseline, confirms the current raw-key behavior this task replaces): `{"firstKolom": "No", "hasRawKlgKey": true}`.

- [ ] **Step 2: Replace `downloadXLSX()`**

Replace:
```js
    function downloadXLSX() {
      loadXLSX(() => {
        const d = collectData();
        const rows = Object.entries(d).map(([k,v]) => ({Kolom: k, Nilai: v}));
        const ws = XLSX.utils.json_to_sheet(rows);
        ws['!cols'] = [{wch:35},{wch:60}];
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, 'Cek Fisik');
        const nama = (d['Kelurahan/Desa']||'KDKMP').replace(/\s+/g,'_');
        const tgl = (d['Tanggal Kunjungan']||new Date().toISOString().slice(0,10)).replace(/-/g,'');
        XLSX.writeFile(wb, `CekFisik_${nama}_${tgl}.xlsx`);
      });
    }
```
with:
```js
    function downloadXLSX() {
      loadXLSX(() => {
        const d = collectData();
        const groups = buildLabeledRows(d);
        const rows = groups.flatMap(g => [
          { Kolom: `— ${g.title} —`, Nilai: '' },
          ...g.fields.map(([label, value]) => ({ Kolom: label, Nilai: value })),
        ]);
        const ws = XLSX.utils.json_to_sheet(rows);
        ws['!cols'] = [{wch:35},{wch:60}];
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, 'Cek Fisik');
        const nama = (d['Kelurahan/Desa']||'KDKMP').replace(/\s+/g,'_');
        const tgl = (d['Tanggal Kunjungan']||new Date().toISOString().slice(0,10)).replace(/-/g,'');
        XLSX.writeFile(wb, `CekFisik_${nama}_${tgl}.xlsx`);
      });
    }
```

- [ ] **Step 3: Reload and verify the row-building logic (without touching the XLSX library)**

Run `mcp__Claude_Preview__preview_eval` with `expression: "window.location.reload()"`, then run this — it re-implements the exact same `rows` construction `downloadXLSX()` now uses internally, so it verifies the row shape without invoking the CDN-loaded `XLSX` library or triggering a file download:
```js
(function(){
  document.getElementById('klg_pos').value = '1';
  const d = collectData();
  const groups = buildLabeledRows(d);
  const rows = groups.flatMap(g => [
    { Kolom: `— ${g.title} —`, Nilai: '' },
    ...g.fields.map(([label, value]) => ({ Kolom: label, Nilai: value })),
  ]);
  return {
    firstRow: rows[0],
    hasRawKlgKey: rows.some(r => r.Kolom.startsWith('Klg_') || r.Kolom.startsWith('Knd_')),
    hasHumanLabel: rows.some(r => r.Kolom === 'POS (Point of Sales)'),
    posValue: rows.find(r => r.Kolom === 'POS (Point of Sales)')?.Nilai
  };
})();
```
Expected: `{"firstRow": {"Kolom": "— Identitas —", "Nilai": ""}, "hasRawKlgKey": false, "hasHumanLabel": true, "posValue": "1"}`.

- [ ] **Step 4: Verify the full `downloadXLSX()` integration runs without error**

This step does trigger the real CDN load and file write (the only such check in this task) to confirm the end-to-end wiring works, not just the pure row-building logic:
```js
(function(){ downloadXLSX(); return 'called'; })();
```
Wait a moment for the CDN script to load (SheetJS, already used successfully in this project), then run `mcp__Claude_Preview__preview_console_logs` with `level: "error"`.
Expected: `No console logs.` (a file named like `CekFisik_..._....xlsx` will be written to the browser's default download location — no need to inspect its contents, Step 3 already verified the row data is correct).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: downloadXLSX pakai label manusiawi dari buildLabeledRows"
```

---

### Task 3: Add the "Download .pdf" button, `loadJsPDF()`, and `downloadPDF()`

**Files:**
- Modify: `index.html:298` (insert new button after the existing "↓ Download .xlsx" button)
- Modify: `index.html` (add `loadJsPDF()` and `downloadPDF()` functions immediately after the existing `downloadXLSX()` function from Task 2)

**Interfaces:**
- Consumes: `buildLabeledRows(d)` and `collectData()` from Task 1/2 (same signatures).
- Produces: `loadJsPDF(callback)` — no return value, calls `callback()` once `window.jspdf` and `window.jspdf.jsPDF.API.autoTable` are available (loading them from CDN first if not).
- Produces: `downloadPDF()` — no params, no return value, called from the new button's `onclick`. Triggers `doc.save('CekFisik_<Desa>_<Tanggal>.pdf')`.

- [ ] **Step 1: Confirm current (pre-change) Penutup step has only one download button — baseline check**

With the preview server still running (or start again with `mcp__Claude_Preview__preview_start`, `name: "cek-fisik"`), run:
```js
(function(){
  showStep(4);
  return {
    downloadButtons: document.querySelectorAll('.btn-download').length,
    pdfButtonExists: !!document.querySelector('.btn-download-pdf'),
    jspdfLoaded: typeof window.jspdf !== 'undefined'
  };
})();
```
Expected (baseline): `{"downloadButtons": 1, "pdfButtonExists": false, "jspdfLoaded": false}`.

- [ ] **Step 2: Add the PDF button in the Penutup step**

In `index.html`, replace:
```html
      <button class="btn btn-download" onclick="downloadXLSX()">&#8595; Download .xlsx</button>
```
with:
```html
      <button class="btn btn-download" onclick="downloadXLSX()">&#8595; Download .xlsx</button>
      <button class="btn btn-download btn-download-pdf" onclick="downloadPDF()" style="margin-top:8px">&#8595; Download .pdf</button>
```

- [ ] **Step 3: Add `loadJsPDF()` and `downloadPDF()` after `downloadXLSX()`**

Immediately after the closing `}` of `downloadXLSX()` (the function from Task 2, ending with `XLSX.writeFile(...)});\n    }`), insert:
```js

    function loadJsPDF(callback) {
      if (typeof window.jspdf !== 'undefined' && typeof window.jspdf.jsPDF.API.autoTable !== 'undefined') { callback(); return; }
      const btn = document.querySelector('.btn-download-pdf');
      btn.textContent = 'Memuat library...'; btn.disabled = true;
      const s1 = document.createElement('script');
      s1.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.2/jspdf.umd.min.js';
      s1.onload = () => {
        const s2 = document.createElement('script');
        s2.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.8.4/jspdf.plugin.autotable.min.js';
        s2.onload = () => { btn.textContent = '↓ Download .pdf'; btn.disabled = false; callback(); };
        s2.onerror = () => { btn.textContent = '↓ Download .pdf'; btn.disabled = false; alert('Gagal memuat library PDF. Periksa koneksi internet.'); };
        document.head.appendChild(s2);
      };
      s1.onerror = () => { btn.textContent = '↓ Download .pdf'; btn.disabled = false; alert('Gagal memuat library PDF. Periksa koneksi internet.'); };
      document.head.appendChild(s1);
    }

    function downloadPDF() {
      loadJsPDF(() => {
        const d = collectData();
        const groups = buildLabeledRows(d);
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF();

        doc.setFontSize(14);
        doc.text('Form Cek Fisik KDKMP', 14, 16);
        doc.setFontSize(10);
        doc.text(`Nama KDKMP: ${d['Nama KDKMP'] || '-'}`, 14, 23);
        doc.text(`Tanggal Kunjungan: ${d['Tanggal Kunjungan'] || '-'}`, 14, 28);

        let startY = 34;
        groups.forEach(g => {
          doc.autoTable({
            head: [[g.title, 'Nilai']],
            body: g.fields.map(([label, value]) => [label, String(value)]),
            startY,
            styles: { fontSize: 9 },
            headStyles: { fillColor: [45, 122, 60] },
            margin: { left: 14, right: 14 },
          });
          startY = doc.lastAutoTable.finalY + 8;
        });

        const nama = (d['Kelurahan/Desa']||'KDKMP').replace(/\s+/g,'_');
        const tgl = (d['Tanggal Kunjungan']||new Date().toISOString().slice(0,10)).replace(/-/g,'');
        doc.save(`CekFisik_${nama}_${tgl}.pdf`);
      });
    }
```

- [ ] **Step 4: Reload and verify the button exists and starts in the right state**

Run `mcp__Claude_Preview__preview_eval` with `expression: "window.location.reload()"`, then:
```js
(function(){
  showStep(4);
  const btn = document.querySelector('.btn-download-pdf');
  return { exists: !!btn, text: btn?.textContent.trim(), disabled: btn?.disabled };
})();
```
Expected: `{"exists": true, "text": "↓ Download .pdf", "disabled": false}`.

- [ ] **Step 5: Trigger `downloadPDF()` and verify the library loads and the button recovers**

```js
(function(){ downloadPDF(); return 'called'; })();
```

This starts an async CDN load (two chained `<script>` tags) — it will take a second or two over the network. Run a second, separate `preview_eval` call to check the result (if `jspdfLoaded` is still `false`, wait a moment and re-run this same check once more before treating it as a failure):
```js
(function(){
  const btn = document.querySelector('.btn-download-pdf');
  return {
    jspdfLoaded: typeof window.jspdf !== 'undefined',
    autoTableLoaded: typeof window.jspdf !== 'undefined' && typeof window.jspdf.jsPDF.API.autoTable !== 'undefined',
    buttonText: btn.textContent.trim(),
    buttonDisabled: btn.disabled
  };
})();
```
Expected: `{"jspdfLoaded": true, "autoTableLoaded": true, "buttonText": "↓ Download .pdf", "buttonDisabled": false}` (a file named like `CekFisik_..._....pdf` will be written to the browser's default download location).

Check for console errors:

Run `mcp__Claude_Preview__preview_console_logs` with `level: "error"`.
Expected: `No console logs.`

Stop the preview server:

Run `mcp__Claude_Preview__preview_stop` with the `serverId` used in this task.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: tambah fitur download PDF via jsPDF + autoTable"
```

---

### Task 4: Push to origin

**Files:** none (git operation only)

**Interfaces:** none — this task only pushes the three commits from Task 1, 2, and 3.

- [ ] **Step 1: Push**

```bash
git push origin main
```
Expected: output shows the local `main` branch ref updating the remote `main` ref on `https://github.com/diperma/cek-fisik.git`, no errors.

- [ ] **Step 2: Confirm no uncommitted changes remain**

```bash
git status
```
Expected: `nothing to commit, working tree clean` (aside from the pre-existing untracked PDF/`.claude` files already present before this feature) and `Your branch is up to date with 'origin/main'`.
