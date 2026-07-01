# Desain: Perbaikan Ringkasan/XLSX & Fitur Download PDF

## Latar Belakang

Audit terhadap `collectData()`, `renderSummary()`, dan `downloadXLSX()` di `index.html` (per commit `3f796b8`) menemukan:

1. `downloadXLSX()` (`index.html:600`) melakukan `Object.entries(collectData())` mentah-mentah, sehingga nama kolom di file .xlsx memakai key internal (mis. `Klg_pos_Jumlah`, `Knd_truk_roda6_Jumlah`) alih-alih label manusiawi ("POS (Point of Sales)", "Truk Roda 6"). Data itu sendiri **lengkap** — tidak ada field yang hilang — tapi kurang enak dibaca saat file dibuka langsung.
2. `renderSummary()` (`index.html:559`) tidak menampilkan nilai Selisih (Panjang/Lebar/Luas bangunan terhadap standar 20m×30m) di layar, walau nilai itu ada di `collectData()` (`Fisik_Selisih_Panjang`, `Fisik_Selisih_Lebar`, `Fisik_Selisih_Luas`) dan sudah ikut ke XLSX.
3. Belum ada fitur download PDF — hanya XLSX.

## Tujuan

1. Satukan sumber label manusiawi antara Ringkasan layar dan XLSX supaya keduanya selalu sinkron, dengan menambah baris Selisih ke grup "Dimensi Bangunan".
2. Tambah tombol "Download .pdf" di step Penutup yang menghasilkan PDF berisi data yang sama persis dengan Ringkasan layar, memakai `jsPDF` + `jsPDF-autoTable` dimuat dari CDN saat tombol pertama kali diklik (pola yang sama dengan `loadXLSX()` yang sudah ada).

## Perubahan 1 & 2: `buildLabeledRows(d)` sebagai Sumber Tunggal

Extract logika grup dari `renderSummary()` menjadi fungsi baru `buildLabeledRows(d)` yang menerima hasil `collectData()` dan mengembalikan array grup `{title, fields: [[label, value], ...]}` — struktur identik dengan variabel `groups` yang sudah ada di `renderSummary()` sekarang, ditambah 3 baris Selisih di grup "Dimensi Bangunan":

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
```

`renderSummary()` dipangkas jadi:
```js
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

`downloadXLSX()` diubah untuk memakai `buildLabeledRows(d)` juga, di-flatten jadi baris `{Kolom, Nilai}` dengan judul grup sebagai baris pemisah (baris dengan `Nilai` kosong):
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

Ini menjamin XLSX dan Ringkasan layar selalu identik karena keduanya membaca dari `buildLabeledRows(d)`, dan baris Selisih otomatis ikut di XLSX juga (sudah ada di sana sebelumnya lewat key mentah, sekarang lewat label yang sama dengan layar).

## Perubahan 3: Fitur Download PDF

### Loading Library

Fungsi baru `loadJsPDF(callback)`, meniru pola `loadXLSX()` (`index.html:309`) persis: cek `typeof window.jspdf !== 'undefined'`, kalau belum ada, ubah teks tombol jadi "Memuat library...", `disabled = true`, lalu muat 2 script berurutan dari CDN:

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
```

### Menghasilkan PDF

```js
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

`doc.autoTable` otomatis menyisipkan halaman baru saat konten kepanjangan; `startY` dihitung ulang dari `doc.lastAutoTable.finalY` setiap grup supaya tabel berikutnya tidak tumpang tindih (dan otomatis lanjut ke halaman baru bila tidak cukup ruang — perilaku bawaan plugin).

### HTML: Tombol Download PDF

Di step Penutup (`index.html`, setelah tombol `.btn-download` existing), tambah tombol baru dengan class terpisah `.btn-download-pdf` (dipakai `loadJsPDF` untuk update teks tombol saat loading) dan margin atas kecil supaya tidak menempel tombol XLSX:

```html
<button class="btn btn-download" onclick="downloadXLSX()">&#8595; Download .xlsx</button>
<button class="btn btn-download btn-download-pdf" onclick="downloadPDF()" style="margin-top:8px">&#8595; Download .pdf</button>
```

Style `.btn-download-pdf` cukup mewarisi `.btn-download` yang sudah ada (hijau, lebar penuh) — tidak perlu CSS baru selain `margin-top` inline di atas.

## Cakupan Perubahan

- File: `index.html` — fungsi baru `buildLabeledRows()`, `loadJsPDF()`, `downloadPDF()`; `renderSummary()` dan `downloadXLSX()` disederhanakan untuk memakai `buildLabeledRows()`; satu tombol baru di step Penutup.
- Tidak ada perubahan pada struktur `collectData()`, `KELENGKAPAN_ITEMS`, atau `KENDARAAN_ITEMS`.
- Tidak menambah dependency baru selain CDN jsPDF + jsPDF-autoTable, dimuat lazy (hanya saat tombol PDF diklik), sama seperti pola XLSX.

## Pengujian

Verifikasi di preview browser:
1. Ringkasan layar menampilkan 3 baris Selisih baru di grup "Dimensi Bangunan".
2. `downloadXLSX()` masih berjalan tanpa error dan kolomnya memakai label manusiawi (mis. "POS (Point of Sales)" bukan "Klg_pos_Jumlah"), termasuk baris judul grup sebagai pemisah.
3. Tombol "Download .pdf" memuat library dari CDN saat pertama diklik (teks tombol berubah jadi "Memuat library..." lalu kembali), dan menghasilkan file .pdf berisi seluruh grup data yang sama dengan Ringkasan layar.
4. Tidak ada error console setelah mengisi beberapa field dan mengklik kedua tombol download.
