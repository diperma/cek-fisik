# Desain: Kelengkapan Peralatan & Fasilitas

## Latar Belakang

Tabel "c. Kelengkapan Peralatan & Fasilitas" di Langkah Kerja 2 (Kondisi Fisik) saat ini memakai daftar datar 13 item dengan kolom Fungsi (Ya/Tidak) dan Keberadaan (Ada/Tidak) berbentuk radio button, plus kolom Jumlah statis (teks target, bukan isian).

Berdasarkan dokumen referensi Kementerian Koperasi (gambar kolom No/Kategori/Uraian/Ada/Tidak Ada/Catatan), daftar ini perlu disesuaikan menjadi lebih lengkap dan rinci per kategori, dengan hanya memasukkan item yang tidak disorot kuning pada dokumen sumber (item kuning = komponen/biaya pendukung yang tidak perlu dicek fisik terpisah).

## Tujuan

1. Ganti daftar `KELENGKAPAN_ITEMS` dengan daftar 23 kategori (hasil ekstraksi & konfirmasi dari gambar sumber, lihat "Daftar Final" di bawah).
2. Hapus kolom **Fungsi** dan **Keberadaan** (radio Ya/Tidak, Ada/Tidak).
3. Ganti kolom **Jumlah** dari teks statis menjadi input angka yang bisa diisi surveyor (bilangan bulat, `min=0`, `step=1`).
4. Kategori dengan >1 sub-item ditampilkan berkelompok (kolom No & Kategori memakai `rowspan`); kategori 1 item ditampilkan sebagai baris tunggal.
5. Penomoran kategori diurutkan ulang 1–23 (tidak mengikuti nomor 4–26 dari dokumen sumber).

## Daftar Final (23 Kategori)

Item yang disorot kuning pada dokumen sumber (komponen/biaya pendukung) **tidak** dimasukkan. Untuk kategori CCTV, hanya 3 item utama yang diminta: Kamera CCTV (gabungan indoor+outdoor, disebut sebagai 1 baris "8 unit"), HDD 2 TB, dan LED Display 19 Inch. Kategori Internet dan Software Kasir disederhanakan jadi satu baris tanpa rincian sub-item (semua sub-item sumbernya kuning). Kategori Keranjang Belanja dan Seragam KDKMP juga disederhanakan menjadi satu baris memakai nama kategori itu sendiri (tanpa merinci jenis keranjang/seragam).

| No | Kategori | Uraian (id) |
|----|----------|-------------|
| 1 | Mesin Kasir | POS (Point of Sales) `pos`; WiFi Router/Modem `wifi_router`; UPS `ups` |
| 2 | CCTV | Kamera CCTV (8 unit) `cctv_kamera`; HDD 2 TB `cctv_hdd`; LED Display 19 Inch `cctv_led` |
| 3 | Internet | Internet `internet` |
| 4 | Software Kasir | Software Kasir `software_kasir` |
| 5 | Air Conditioner | AC 2 PK `ac_2pk`; AC 1 PK `ac_1pk` |
| 6 | Furniture, Gerai, Apotek dan Klinik | Meja Koperasi Simpan Pinjam `furniture_meja_koperasi`; Laci Dorong (Gerai) `furniture_laci_gerai`; Rak Etalase `furniture_rak_etalase`; Meja Kasir `furniture_meja_kasir`; Lemari Display Apotek `furniture_lemari_apotek`; Meja Administrasi Klinik `furniture_meja_administrasi`; Laci Dorong (Klinik) `furniture_laci_klinik`; Kursi Dokter `furniture_kursi_dokter`; Kursi Pasien `furniture_kursi_pasien`; Kursi Klinik `furniture_kursi_klinik`; Kursi Kasir `furniture_kursi_kasir`; Kursi Tunggu `furniture_kursi_tunggu`; Meja Dokter `furniture_meja_dokter`; Tempat Tidur Periksa Pasien `furniture_tempat_tidur` |
| 7 | Perlengkapan Rak Gerai | Rak Display Single/Wall/Pinggir (90×40×170 cm) `rak_single`; Rak Display Double/Island/Tengah (90×80×150 cm) `rak_double`; Rak Display End/Endcap/Tutup (80×40×150 cm) `rak_endcap` |
| 8 | Alat Pemadam Api Ringan (APAR) | APAR 3 kg `apar_3kg`; APAR 9 kg `apar_9kg` |
| 9 | Keranjang Belanja | Keranjang Belanja `keranjang_belanja` |
| 10 | Seragam KDKMP | Seragam KDKMP `seragam_kdkmp` |
| 11 | Pallet | Pallet `pallet` |
| 12 | Printer | Printer `printer` |
| 13 | Brankas | Brankas `brankas` |
| 14 | Genset | Genset `genset` |
| 15 | ATK | Stempel `atk_stempel`; Struk `atk_struk`; Thermal Label `atk_thermal_label` |
| 16 | Alat Kebersihan | Tempat Sampah Besar `kebersihan_tempat_sampah`; Alat Pel dan Ember `kebersihan_pel_ember`; Sapu Lantai dan Pengki `kebersihan_sapu_pengki` |
| 17 | Computer (All in One) | PC All in One Intel, RAM 8 GB, SSD 512 GB `computer_aio` |
| 18 | Perlengkapan Beverages dan Frozen Foods | Chiller 2 Pintu (Tinggi ±197 cm) `chiller_2pintu`; Chest Freezer Kapasitas ±680 Liter `chest_freezer` |
| 19 | Kipas Ceiling | Kipas Plafon `kipas_ceiling` |
| 20 | Exhaust Fan | Exhaust Fan `exhaust_fan` |
| 21 | Loker Lemari | Loker Lemari `loker_lemari` |
| 22 | Alarm Toko | Alarm Toko `alarm_toko` |
| 23 | Mesin Absensi | Mesin Absensi `mesin_absensi` |

Total kategori: 23. Total baris item: 49.

## Struktur Data

`KELENGKAPAN_ITEMS` diubah dari array datar menjadi array kategori:

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
  // ... (lihat tabel "Daftar Final" di atas untuk seluruh 23 kategori)
];
```

Struktur `KENDARAAN_ITEMS` dan tabel-tabel lain (Kondisi Fisik Bangunan, Kendaraan) tidak berubah.

## Struktur HTML Tabel

Header tetap 4 kolom: `No | Kelengkapan | Uraian | Jumlah` — tapi karena tiap kategori kini bisa punya banyak sub-item, header disesuaikan menjadi `No | Kategori | Uraian | Jumlah`. Body (`<tbody id="kelengkapan-body">`) dibangun sepenuhnya oleh JS (seperti sekarang), bukan ditulis manual di HTML.

Untuk kategori dengan N item (N>1): baris pertama berisi `<td rowspan="N">No</td><td rowspan="N">Kategori</td><td>Uraian item 1</td><td><input.../></td>`, baris berikutnya (N-1 baris) hanya berisi `<td>Uraian item ke-n</td><td><input.../></td>`.

Untuk kategori 1 item: baris tunggal `<td>No</td><td>Kategori</td><td>Uraian (sama dengan label item)</td><td><input.../></td>` — tanpa rowspan khusus (rowspan=1 setara baris normal).

Setiap input jumlah: `<input type="number" id="klg_<id>" min="0" step="1" placeholder="0">`, tidak wajib diisi (tidak ada atribut `required`), konsisten dengan pola input jumlah lain di form (mis. Dimensi Bangunan).

## Perubahan JavaScript

### `renderKelengkapan()`

Ditulis ulang untuk iterasi `KELENGKAPAN_ITEMS` (array kategori), membangun HTML baris dengan rowspan sesuai `items.length`, dan nomor kategori berurutan (`i+1`) — bukan diambil dari indeks global per item seperti sekarang.

### `collectData()`

Entri lama:
```js
...Object.fromEntries(KELENGKAPAN_ITEMS.flatMap(({id}) => {
  const kb = document.querySelector(`input[name="klg_${id}"]:checked`)?.value || '';
  const fn = document.querySelector(`input[name="klg_fungsi_${id}"]:checked`)?.value || '';
  return [[`Klg_${id}_Keberadaan`, kb], [`Klg_${id}_Fungsi`, fn]];
})),
```

Diganti (iterasi semua item di semua kategori, ambil value input jumlah):
```js
...Object.fromEntries(
  KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id}) => [`Klg_${id}_Jumlah`, val(`klg_${id}`)])
),
```

### `renderSummary()`

Grup "Kelengkapan" tetap satu grup ringkasan, tapi field-nya dibangun dari flatten semua item semua kategori:
```js
{ title: 'Kelengkapan', fields: KELENGKAPAN_ITEMS.flatMap(({items}) => items).map(({id, label}) => {
  const jumlah = d[`Klg_${id}_Jumlah`];
  return [label, jumlah ? jumlah : '-'];
})},
```

### Kode yang dihapus

- Semua radio button generation (`klg_${id}` value Ada/Tidak, `klg_fungsi_${id}` value Ya/Tidak) di `renderKelengkapan()`.
- Kolom "Fungsi" dan "Keberadaan" di header tabel HTML.
- Deskripsi teks lama "Centang fungsi dan keberadaan setiap item berdasarkan hasil observasi visual." diganti dengan teks yang sesuai untuk isian jumlah, mis. "Isi jumlah aktual tiap item berdasarkan hasil observasi fisik."

## Cakupan Perubahan

- File: `index.html` — bagian HTML tabel Kelengkapan (id `tbl-kelengkapan`), array `KELENGKAPAN_ITEMS`, fungsi `renderKelengkapan()`, `collectData()`, `renderSummary()`.
- Tidak mengubah tabel Kondisi Fisik Bangunan atau Kendaraan.
- Tidak menambah kolom Catatan (di luar cakupan permintaan).

## Pengujian

Setelah implementasi, verifikasi di preview browser:
1. Tabel Kelengkapan menampilkan 23 kategori dengan rowspan yang benar untuk kategori multi-item (khususnya kategori 6 dengan 14 baris).
2. Input jumlah menerima angka bulat ≥ 0, tidak menerima desimal (step=1) — cek dengan mengisi nilai lalu membaca kembali via `collectData()`.
3. Ringkasan (`renderSummary()`) menampilkan tiap item dengan jumlah yang diisi, atau `-` bila kosong.
4. Tidak ada error console saat render, isi, dan pindah antar step.
