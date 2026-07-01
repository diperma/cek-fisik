# Kelengkapan Peralatan & Fasilitas Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the flat, radio-button-based "Kelengkapan Peralatan & Fasilitas" table in `index.html` with a category-grouped table (23 categories, 49 items) where each item has a single numeric "Jumlah" input, per the approved spec at `docs/superpowers/specs/2026-07-01-kelengkapan-peralatan-fasilitas-design.md`.

**Architecture:** This is a single static HTML file (`index.html`) with vanilla JS embedded in `<script>` tags — no build step, no test framework, no package.json. All logic lives in plain functions operating on the DOM. The change touches three things in that one file: (1) the `KELENGKAPAN_ITEMS` data array and the table's HTML header, (2) the `renderKelengkapan()` function that builds table rows from that data, and (3) `collectData()` / `renderSummary()`, which read the rendered inputs back into a flat key-value object used for the on-screen summary and XLSX export.

**Tech Stack:** Vanilla HTML/CSS/JS, no frameworks, no build tooling. Verification is done by serving the file with the project's existing preview server (`.claude/launch.json`, config name `cek-fisik`, static file server on port 8531) and driving it with the `preview_*` MCP tools (`preview_start`, `preview_eval`, `preview_console_logs`, `preview_stop`) — there is no automated test runner in this repo, so this is the project's established verification method (used for the two prior changes on this branch: removing the Lokasi step and removing the Dimensi Ruangan table).

## Global Constraints

- Input jumlah per item: `<input type="number" min="0" step="1" placeholder="0">`, no `required` attribute (matches other optional numeric fields already in the form).
- Categories with more than 1 item use `rowspan` on the No and Kategori columns; categories with exactly 1 item render as a plain single row (rowspan="1", no special-casing needed).
- Category numbering is renumbered sequentially 1–23 (not the source document's 4–26 numbering).
- No "Catatan" column is added — table stays at 4 columns: No, Kategori, Uraian, Jumlah.
- Only `index.html` is modified. Do not touch the "Kondisi Fisik Bangunan" table or the "Kendaraan" table.
- Full data set: 23 categories, 49 item rows total (exact list and IDs are given verbatim in Task 1 — copy them exactly, do not re-derive from the spec's prose table).

---

### Task 1: Rebuild `KELENGKAPAN_ITEMS` data model and table rendering

**Files:**
- Modify: `index.html:248-259` (h3 caption, description paragraph, table header)
- Modify: `index.html:402-416` (the `KELENGKAPAN_ITEMS` array)
- Modify: `index.html:424-441` (the `renderKelengkapan()` function)

**Interfaces:**
- Produces: `KELENGKAPAN_ITEMS` — `Array<{category: string, items: Array<{id: string, label: string}>}>`, 23 entries, 49 total items across all `items` arrays.
- Produces: `renderKelengkapan()` — no params, no return value; populates `#kelengkapan-body` with `<tr>` rows, one row per item (49 rows total), each item row containing an `<input type="number" id="klg_<id>">`. Category rows with `items.length > 1` place `rowspan="<items.length>"` `<td>` cells for No and Kategori only on the first item's `<tr>`.
- Consumes: nothing new (uses existing `document.getElementById`/DOM APIs already used elsewhere in the file).

- [ ] **Step 1: Confirm current (pre-change) row count as the baseline "failing" check**

There's no test framework in this repo, so the check-before/check-after pattern is done live in the browser via the preview server.

Run: `mcp__Claude_Preview__preview_start` with `name: "cek-fisik"` (reuses `.claude/launch.json`, already configured to serve the repo root on port 8531).

Then run `mcp__Claude_Preview__preview_eval` with:
```js
(function(){
  return {
    rows: document.querySelectorAll('#kelengkapan-body tr').length,
    numberInputs: document.querySelectorAll('#kelengkapan-body input[type="number"]').length,
    radioInputs: document.querySelectorAll('#kelengkapan-body input[type="radio"]').length
  };
})();
```
Expected (baseline, confirms the old structure is still in place): `{"rows": 13, "numberInputs": 0, "radioInputs": 26}`.

- [ ] **Step 2: Replace the table caption, description, and header**

In `index.html`, replace:
```html
      <h3 style="color:var(--hijau);margin:0 0 6px">c. Kelengkapan Peralatan &amp; Fasilitas</h3>
      <p style="font-size:13px;color:#888;margin-bottom:12px">Centang fungsi dan keberadaan setiap item berdasarkan hasil observasi visual.</p>
      <div style="overflow-x:auto;margin-bottom:24px">
        <table class="rab-table" id="tbl-kelengkapan">
          <thead><tr>
            <th style="width:36px">No</th><th>Kelengkapan</th>
            <th style="width:130px;text-align:center">Fungsi</th>
            <th style="width:130px;text-align:center">Keberadaan</th>
            <th style="width:110px;text-align:center">Jumlah</th>
          </tr></thead>
          <tbody id="kelengkapan-body"></tbody>
        </table>
```
with:
```html
      <h3 style="color:var(--hijau);margin:0 0 6px">c. Kelengkapan Peralatan &amp; Fasilitas</h3>
      <p style="font-size:13px;color:#888;margin-bottom:12px">Isi jumlah aktual tiap item berdasarkan hasil observasi fisik.</p>
      <div style="overflow-x:auto;margin-bottom:24px">
        <table class="rab-table" id="tbl-kelengkapan">
          <thead><tr>
            <th style="width:36px">No</th><th style="width:220px">Kategori</th>
            <th>Uraian</th>
            <th style="width:110px;text-align:center">Jumlah</th>
          </tr></thead>
          <tbody id="kelengkapan-body"></tbody>
        </table>
```

- [ ] **Step 3: Replace the `KELENGKAPAN_ITEMS` array**

Replace:
```js
    const KELENGKAPAN_ITEMS = [
      {id:'rak_display', label:'Rak display',                       jumlah:'1 set'},
      {id:'furniture',   label:'Furniture atau meubel',             jumlah:'1 set'},
      {id:'kasir_hw',    label:'Perangkat keras kasir',             jumlah:'1 set'},
      {id:'kasir_sw',    label:'Perangkat lunak (software) kasir',  jumlah:'1 set'},
      {id:'internet',    label:'Internet (LTE 4G / Satelit)',       jumlah:'1 set'},
      {id:'ac',          label:'Air conditioner',                   jumlah:'3 Pcs 2PK, 2 Pcs 1PK'},
      {id:'apar',        label:'Alat pemadam api ringan (APAR)',    jumlah:'2 pcs'},
      {id:'cctv',        label:'CCTV',                              jumlah:'1 set'},
      {id:'brankas',     label:'Brankas',                           jumlah:''},
      {id:'keranjang',   label:'Keranjang Belanja',                 jumlah:'1 set'},
      {id:'pallet',      label:'Pallet',                            jumlah:'1 set'},
      {id:'printer',     label:'Printer',                           jumlah:'1 Pcs'},
      {id:'seragam',     label:'Seragam Pegawai',                   jumlah:'1 Set'},
    ];
```
with:
```js
    const KELENGKAPAN_ITEMS = [
      { category: 'Mesin Kasir', items: [
          {id:'pos', label:'POS (Point of Sales)'},
          {id:'wifi_router', label:'WiFi Router/Modem'},
          {id:'ups', label:'UPS'},
      ]},
      { category: 'CCTV', items: [
          {id:'cctv_kamera', label:'Kamera CCTV (8 unit)'},
          {id:'cctv_hdd', label:'HDD 2 TB'},
          {id:'cctv_led', label:'LED Display 19 Inch'},
      ]},
      { category: 'Internet', items: [
          {id:'internet', label:'Internet'},
      ]},
      { category: 'Software Kasir', items: [
          {id:'software_kasir', label:'Software Kasir'},
      ]},
      { category: 'Air Conditioner', items: [
          {id:'ac_2pk', label:'AC 2 PK'},
          {id:'ac_1pk', label:'AC 1 PK'},
      ]},
      { category: 'Furniture, Gerai, Apotek dan Klinik', items: [
          {id:'furniture_meja_koperasi', label:'Meja Koperasi Simpan Pinjam'},
          {id:'furniture_laci_gerai', label:'Laci Dorong (Gerai)'},
          {id:'furniture_rak_etalase', label:'Rak Etalase'},
          {id:'furniture_meja_kasir', label:'Meja Kasir'},
          {id:'furniture_lemari_apotek', label:'Lemari Display Apotek'},
          {id:'furniture_meja_administrasi', label:'Meja Administrasi Klinik'},
          {id:'furniture_laci_klinik', label:'Laci Dorong (Klinik)'},
          {id:'furniture_kursi_dokter', label:'Kursi Dokter'},
          {id:'furniture_kursi_pasien', label:'Kursi Pasien'},
          {id:'furniture_kursi_klinik', label:'Kursi Klinik'},
          {id:'furniture_kursi_kasir', label:'Kursi Kasir'},
          {id:'furniture_kursi_tunggu', label:'Kursi Tunggu'},
          {id:'furniture_meja_dokter', label:'Meja Dokter'},
          {id:'furniture_tempat_tidur', label:'Tempat Tidur Periksa Pasien'},
      ]},
      { category: 'Perlengkapan Rak Gerai', items: [
          {id:'rak_single', label:'Rak Display Single/Wall/Pinggir (90×40×170 cm)'},
          {id:'rak_double', label:'Rak Display Double/Island/Tengah (90×80×150 cm)'},
          {id:'rak_endcap', label:'Rak Display End/Endcap/Tutup (80×40×150 cm)'},
      ]},
      { category: 'Alat Pemadam Api Ringan (APAR)', items: [
          {id:'apar_3kg', label:'APAR 3 kg'},
          {id:'apar_9kg', label:'APAR 9 kg'},
      ]},
      { category: 'Keranjang Belanja', items: [
          {id:'keranjang_belanja', label:'Keranjang Belanja'},
      ]},
      { category: 'Seragam KDKMP', items: [
          {id:'seragam_kdkmp', label:'Seragam KDKMP'},
      ]},
      { category: 'Pallet', items: [
          {id:'pallet', label:'Pallet'},
      ]},
      { category: 'Printer', items: [
          {id:'printer', label:'Printer'},
      ]},
      { category: 'Brankas', items: [
          {id:'brankas', label:'Brankas'},
      ]},
      { category: 'Genset', items: [
          {id:'genset', label:'Genset'},
      ]},
      { category: 'ATK', items: [
          {id:'atk_stempel', label:'Stempel'},
          {id:'atk_struk', label:'Struk'},
          {id:'atk_thermal_label', label:'Thermal Label'},
      ]},
      { category: 'Alat Kebersihan', items: [
          {id:'kebersihan_tempat_sampah', label:'Tempat Sampah Besar'},
          {id:'kebersihan_pel_ember', label:'Alat Pel dan Ember'},
          {id:'kebersihan_sapu_pengki', label:'Sapu Lantai dan Pengki'},
      ]},
      { category: 'Computer (All in One)', items: [
          {id:'computer_aio', label:'PC All in One Intel, RAM 8 GB, SSD 512 GB'},
      ]},
      { category: 'Perlengkapan Beverages dan Frozen Foods', items: [
          {id:'chiller_2pintu', label:'Chiller 2 Pintu (Tinggi ±197 cm)'},
          {id:'chest_freezer', label:'Chest Freezer Kapasitas ±680 Liter'},
      ]},
      { category: 'Kipas Ceiling', items: [
          {id:'kipas_ceiling', label:'Kipas Plafon'},
      ]},
      { category: 'Exhaust Fan', items: [
          {id:'exhaust_fan', label:'Exhaust Fan'},
      ]},
      { category: 'Loker Lemari', items: [
          {id:'loker_lemari', label:'Loker Lemari'},
      ]},
      { category: 'Alarm Toko', items: [
          {id:'alarm_toko', label:'Alarm Toko'},
      ]},
      { category: 'Mesin Absensi', items: [
          {id:'mesin_absensi', label:'Mesin Absensi'},
      ]},
    ];
```

- [ ] **Step 4: Rewrite `renderKelengkapan()`**

Replace:
```js
    function renderKelengkapan() {
      const tbody = document.getElementById('kelengkapan-body');
      if (!tbody) return;
      tbody.innerHTML = KELENGKAPAN_ITEMS.map(({id, label, jumlah}, i) => `
        <tr>
          <td style="text-align:center">${i+1}</td>
          <td>${label}</td>
          <td style="text-align:center">
            <label style="cursor:pointer;margin-right:6px"><input type="radio" name="klg_fungsi_${id}" value="Y"> Ya</label>
            <label style="cursor:pointer"><input type="radio" name="klg_fungsi_${id}" value="T"> Tidak</label>
          </td>
          <td style="text-align:center">
            <label style="cursor:pointer;margin-right:6px"><input type="radio" name="klg_${id}" value="Y"> Ada</label>
            <label style="cursor:pointer"><input type="radio" name="klg_${id}" value="T"> Tidak</label>
          </td>
          <td style="text-align:center;color:#555;font-size:0.9em">${jumlah}</td>
        </tr>`).join('');
    }
```
with:
```js
    function renderKelengkapan() {
      const tbody = document.getElementById('kelengkapan-body');
      if (!tbody) return;
      tbody.innerHTML = KELENGKAPAN_ITEMS.map(({category, items}, catIdx) => {
        return items.map(({id, label}, itemIdx) => {
          const jumlahCell = `<td style="text-align:center"><input type="number" id="klg_${id}" min="0" step="1" placeholder="0"></td>`;
          if (itemIdx === 0) {
            return `
        <tr>
          <td style="text-align:center" rowspan="${items.length}">${catIdx + 1}</td>
          <td rowspan="${items.length}">${category}</td>
          <td>${label}</td>
          ${jumlahCell}
        </tr>`;
          }
          return `
        <tr>
          <td>${label}</td>
          ${jumlahCell}
        </tr>`;
        }).join('');
      }).join('');
    }
```

- [ ] **Step 5: Reload the preview and verify the new structure**

Run `mcp__Claude_Preview__preview_eval` with `expression: "window.location.reload()"`, then re-run:
```js
(function(){
  const rows = document.querySelectorAll('#kelengkapan-body tr');
  return {
    totalRows: rows.length,
    numberInputs: document.querySelectorAll('#kelengkapan-body input[type="number"]').length,
    radioInputs: document.querySelectorAll('#kelengkapan-body input[type="radio"]').length,
    headerCols: Array.from(document.querySelectorAll('#tbl-kelengkapan thead th')).map(th => th.textContent.trim())
  };
})();
```
Expected: `{"totalRows": 49, "numberInputs": 49, "radioInputs": 0, "headerCols": ["No","Kategori","Uraian","Jumlah"]}`.

Then verify the Furniture category's rowspan specifically (it's the largest, 14 items — the row most likely to have an off-by-one error):
```js
(function(){
  const rows = Array.from(document.querySelectorAll('#kelengkapan-body tr'));
  const furnitureRow = rows.find(tr => tr.textContent.includes('Furniture, Gerai, Apotek dan Klinik'));
  const tds = furnitureRow.querySelectorAll('td');
  return { firstTdRowspan: tds[0].getAttribute('rowspan'), secondTdRowspan: tds[1].getAttribute('rowspan'), categoryText: tds[1].textContent.trim() };
})();
```
Expected: `{"firstTdRowspan": "14", "secondTdRowspan": "14", "categoryText": "Furniture, Gerai, Apotek dan Klinik"}`.

Check the browser console for errors:

Run `mcp__Claude_Preview__preview_console_logs` with `level: "error"`.
Expected: `No console logs.`

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: rombak tabel Kelengkapan Peralatan & Fasilitas jadi 23 kategori dengan input jumlah"
```

---

### Task 2: Update `collectData()` and `renderSummary()` for the new `Klg_<id>_Jumlah` shape

**Files:**
- Modify: `index.html:512-516` (the `KELENGKAPAN_ITEMS` block inside `collectData()`)
- Modify: `index.html:546-550` (the `Kelengkapan` group inside `renderSummary()`)

**Interfaces:**
- Consumes: `KELENGKAPAN_ITEMS` (`Array<{category, items: Array<{id, label}>}>`) and `klg_<id>` number inputs from Task 1.
- Consumes: existing `val(id)` helper (already defined in `index.html`, returns `document.getElementById(id)?.value || ''`).
- Produces: `collectData()` includes one key per item, `Klg_<id>_Jumlah: string` (the raw input value, empty string if blank) — 49 keys total.
- Produces: `renderSummary()`'s `groups` array has a `{ title: 'Kelengkapan', fields: [[label, value], ...] }` entry with 49 `[label, value]` pairs, `value` being the entered number as a string or `'-'` if blank.

- [ ] **Step 1: Confirm the current (pre-change) data shape as baseline**

With the preview server still running from Task 1 (or start it again with `mcp__Claude_Preview__preview_start`, `name: "cek-fisik"`), run:
```js
(function(){
  const d = collectData();
  return Object.keys(d).filter(k => k.startsWith('Klg_'));
})();
```
Expected (baseline — confirms `collectData()` is broken in the way this task fixes): Task 1 already changed `KELENGKAPAN_ITEMS` from a flat array of items to an array of categories (`{category, items}`), but this old `collectData()` code still does `KELENGKAPAN_ITEMS.flatMap(({id}) => ...)`, destructuring a top-level `id` that no longer exists on a category object. That destructures to `undefined` on every iteration, so the result collapses to exactly 2 keys: `["Klg_undefined_Keberadaan","Klg_undefined_Fungsi"]` (both empty strings) — not a crash, just silently wrong data. This confirms Step 2's fix is necessary before proceeding.

- [ ] **Step 2: Replace the `collectData()` block**

Replace:
```js
        ...Object.fromEntries(KELENGKAPAN_ITEMS.flatMap(({id}) => {
          const kb = document.querySelector(`input[name="klg_${id}"]:checked`)?.value || '';
          const fn = document.querySelector(`input[name="klg_fungsi_${id}"]:checked`)?.value || '';
          return [[`Klg_${id}_Keberadaan`, kb], [`Klg_${id}_Fungsi`, fn]];
        })),
```
with:
```js
        ...Object.fromEntries(
          KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id}) => [`Klg_${id}_Jumlah`, val(`klg_${id}`)])
        ),
```

- [ ] **Step 3: Replace the `renderSummary()` Kelengkapan group**

Replace:
```js
        { title: 'Kelengkapan', fields: KELENGKAPAN_ITEMS.map(({id, label}) => {
          const kb = d[`Klg_${id}_Keberadaan`]==='Y'?'Ada':d[`Klg_${id}_Keberadaan`]==='T'?'Tidak ada':'-';
          const fn = d[`Klg_${id}_Fungsi`]==='Y'?', berfungsi':d[`Klg_${id}_Fungsi`]==='T'?', tidak berfungsi':'';
          return [label, kb+fn];
        })},
```
with:
```js
        { title: 'Kelengkapan', fields: KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id, label}) => {
          const jumlah = d[`Klg_${id}_Jumlah`];
          return [label, jumlah ? jumlah : '-'];
        })},
```

- [ ] **Step 4: Reload and verify `collectData()` reflects entered values**

Run `mcp__Claude_Preview__preview_eval` with `expression: "window.location.reload()"`.

Fill two inputs and read them back:
```js
(function(){
  document.getElementById('klg_pos').value = '1';
  document.getElementById('klg_ac_2pk').value = '3';
  const d = collectData();
  return { pos: d['Klg_pos_Jumlah'], ac2pk: d['Klg_ac_2pk_Jumlah'], unfilled: d['Klg_ups_Jumlah'], totalKlgKeys: Object.keys(d).filter(k => k.startsWith('Klg_')).length };
})();
```
Expected: `{"pos": "1", "ac2pk": "3", "unfilled": "", "totalKlgKeys": 49}`.

- [ ] **Step 5: Verify `renderSummary()` renders filled and empty values correctly**

```js
(function(){
  showStep(4);
  renderSummary();
  const text = document.getElementById('summary-content').textContent;
  return { hasPos: text.includes('POS (Point of Sales)'), noKeberadaanWord: !text.includes('berfungsi') && !text.includes('Tidak ada') };
})();
```
Expected: `{"hasPos": true, "noKeberadaanWord": true}`.

Check for console errors:

Run `mcp__Claude_Preview__preview_console_logs` with `level: "error"`.
Expected: `No console logs.`

Stop the preview server:

Run `mcp__Claude_Preview__preview_stop` with the `serverId` from Step 1/4.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: sesuaikan collectData dan ringkasan dengan input Jumlah Kelengkapan"
```

---

### Task 3: Push to origin

**Files:** none (git operation only)

**Interfaces:** none — this task only pushes the two commits from Task 1 and Task 2.

- [ ] **Step 1: Push**

```bash
git push origin main
```
Expected: output shows the local `main` branch ref updating the remote `main` ref on `https://github.com/diperma/cek-fisik.git`, no errors.

- [ ] **Step 2: Confirm no uncommitted changes remain**

```bash
git status
```
Expected: `nothing to commit, working tree clean` and `Your branch is up to date with 'origin/main'`.
