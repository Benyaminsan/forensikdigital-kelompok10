# Kunci Jawaban dan Write-Up

## Ringkasan Jawaban

| Informasi | Detail |
| --- | --- |
| **File target** | `\Foto\Liburan\thumbs.db` |
| **Nama file asli** | `kontrak_rahasia.pdf` |
| **Atribut file target** | Hidden + System (+ Archive, bawaan) |
| **Signature isi file** | `25 50 44 46 2D` (`%PDF-`) |
| **Signature `thumbs.db` asli** | `D0 CF 11 E0 A1 B1 1A E1` (OLE2) |
| **Flag** | `FLAG{3kst3ns1_b0h0ng_h1dd3n_syst3m_thumbsdb}` |

## Isi `kontrak_rahasia.pdf`

### Halaman 1

Berisi perjanjian kerja sama antara **PT Nusantara Digital Solusi** dan **CV Arjuna Teknik**. Nilai proyek disepakati sebesar **Rp 850.000.000,00**. Dokumen ditandatangani oleh Direktur Utama PT Nusantara Digital Solusi dan Direktur CV Arjuna Teknik. Dokumen bersifat rahasia.

### Halaman 2 — Lampiran

Terdapat lampiran untuk memverifikasi keaslian dokumen yang berisi kode:

```text
FLAG{3kst3ns1_b0h0ng_h1dd3n_syst3m_thumbsdb}
```

### Metadata PDF

Isi kolom **Title** pada metadata PDF dengan `Kontrak Rahasia`. Metadata ini menjadi petunjuk bagi peserta yang teliti untuk menebak nama asli file.

## Struktur Isi Flash Disk

**Label volume:** `BACKUP_KANTOR`

```text
BACKUP_KANTOR
├── Dokumen\
│   ├── notulen_rapat.txt
│   ├── kontrak_biasa.pdf       # Decoy: PDF normal tanpa flag
│   └── desktop.ini             # Decoy: file hidden + system yang asli (berisi teks)
├── Foto\Liburan\
│   ├── pantai_01.jpg
│   ├── pantai_02.jpg
│   ├── ...
│   └── thumbs.db               # TARGET: hidden + system, tetapi isinya PDF
└── Musik\
```

`desktop.ini` sengaja disertakan sebagai pengecoh. Peserta akan menemukan beberapa file beratribut hidden + system dan harus membedakan file yang isinya sesuai dengan ekstensinya dari file yang tidak sesuai. Hal ini membuat tingkat kesulitan soal menjadi **medium**, bukan sekadar mencari satu file hidden.

Folder `Foto\Liburan` dipilih karena `thumbs.db` lazim ditemukan di folder yang berisi gambar.

## Write-Up (Kunci Solving)

### Konsep di Balik Soal

Soal ini menggunakan dua lapis penyamaran:

1. **Ekstensi dan nama file:** ekstensi diubah dari `.pdf` menjadi `.db`, dengan nama `thumbs.db`. Nama tersebut dipilih karena file cache thumbnail Windows terlihat tidak menarik dan biasanya diabaikan.
2. **Atribut System:** atribut **Hidden + System** membuat file tidak terlihat di Explorer secara default, karena Windows menyembunyikan protected operating system files.

Kedua lapis tersebut hanya mengelabui manusia dan Windows Explorer. Bagi tools forensik:

- Tools forensik membaca file system secara langsung; atribut hidden hanya menjadi informasi tambahan.
- Ekstensi hanyalah bagian dari nama file. Mengganti nama file tidak mengubah signature pada isinya.

**Logika penyelesaian:** temukan file yang ekstensinya tidak cocok dengan signature-nya melalui signature analysis, lalu periksa isinya.

### Metode Penyelesaian

#### Cara A — Autopsy (GUI Berbasis Sleuth Kit)

1. **Buat case dan tambahkan data source**
   - Pilih **New Case**, lalu isi nama case dan examiner.
   - Pilih **Add Data Source** → **Disk Image or VM File**, lalu arahkan ke file `.E01` atau `.dd`.
2. **Atur ingest modules**
   - Pada **Configure Ingest Modules**, aktifkan minimal:
     - **File Type Identification**
     - **Extension Mismatch Detector**
     - **Keyword Search** (opsional)
3. **Telusuri struktur file**
   - Buka **Data Sources** → image → volume → folder.
   - Temukan `thumbs.db` dan `desktop.ini`.
4. **Cari anomali ekstensi**
   - Buka **Results** → **Extension Mismatch Detected**.
   - `thumbs.db` akan muncul dengan MIME type `application/pdf`.
5. **Validasi melalui hex dan metadata**
   - Pilih `thumbs.db`, lalu buka tab **Hex**. Bytes awal menunjukkan `25 50 44 46 2D` (`%PDF-`).
   - Tab **File Metadata** mengonfirmasi atribut Hidden dan System.
6. **Ekstrak flag**
   - Klik kanan `thumbs.db` → **Extract File(s)**.
   - Simpan hasil ekstraksi, ubah ekstensi salinannya menjadi `.pdf`, lalu buka.
   - Flag terdapat pada halaman 2 (Lampiran).

#### Cara B — Sleuth Kit (Command Line Interface)

Periksa tabel partisi:

```bash
mmls BACKUP_KANTOR.dd
```

Daftar file secara rekursif:

```bash
fls -r -o <offset> BACKUP_KANTOR.dd
```

Lihat metadata file:

```bash
istat -o <offset> BACKUP_KANTOR.dd <inode>
```

Ekstrak isi file:

```bash
icat -o <offset> BACKUP_KANTOR.dd <inode> > hasil.bin
```

Periksa jenis file hasil ekstraksi:

```bash
file hasil.bin
```

Output akan menunjukkan bahwa `hasil.bin` adalah dokumen PDF. Ubah ekstensi salinannya menjadi `.pdf`, lalu buka dokumen tersebut.

#### Cara C — FTK Suites

**FTK Imager (jalur ringan):**

1. Pilih **File** → **Add Evidence Item** → **Image File**.
2. Buka partisi → `Foto` → `Liburan`.
3. Pilih `thumbs.db` dan periksa tab **Hex** untuk signature `%PDF-`.
4. Pilih **Export Files**, simpan hasilnya dengan ekstensi `.pdf`, lalu buka.

**FTK penuh:**

1. Pilih **Case Manager** → **New Case** → **Add Evidence**.
2. Buka tab **Overview** → **File Status** → **Bad Extension**.
3. `thumbs.db` akan muncul dalam daftar dan dapat ditampilkan pada tab **Viewer**.

#### Cara D — EnCase Forensic

1. Pilih **New Case** → **Add Evidence**.
2. Jalankan **Evidence Processor** dengan opsi **File signature analysis** aktif.
3. Pada **Table view**, kolom **Signature Analysis** untuk `thumbs.db` akan ditandai **Bad Signature**, sementara **File Type** terdeteksi sebagai PDF/Acrobat.
4. Gunakan opsi **Copy Files** untuk mengekspor dan membuka dokumen.

## Catatan untuk Panitia

### Hasil Akhir yang Benar

```text
FLAG{3kst3ns1_b0h0ng_h1dd3n_syst3m_thumbsdb}
```

### Kesalahan Umum Peserta

- Hanya mengandalkan Windows Explorer atau opsi **Show hidden files**, lalu mengira `thumbs.db` hanyalah file cache bawaan.
- Mengabaikan `thumbs.db` karena nama dan ekstensinya terlihat normal.
- Tidak memeriksa header hex file.

### Antisipasi Jalur Cepat (Keyword Search)

Peserta dapat mencari string `FLAG{` secara langsung. Untuk meningkatkan kompleksitas soal di masa depan, teks flag dapat ditempatkan sebagai gambar atau stempel yang di-embed di dalam dokumen PDF.

### Pengujian Soal

Pastikan panitia melakukan uji coba penyelesaian (*dry run*) menggunakan setidaknya satu forensic tool sebelum soal diujikan.
