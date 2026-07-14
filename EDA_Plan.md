# Rencana EDA Final — `02_eda.ipynb`

Panduan internal membangun insight dari clean data Student Placement System (SSDC 2026).
Format tiap poin: **Tujuan** (insight yang dicari) · **Data** (tabel + kolom + join) · **Cara hitung** (logika) · **Output** (chart/tabel + yang diperhatikan).

Anotasi halaman dashboard: `→ H1` Overview & Kelayakan · `→ H2` Matching & Geografi · `→ H3` Keberhasilan & Tren · `→ H4` Risiko Operasional.
Tanda `[+]` = insight-hunting di luar coverage BT dasar. Tanda `[✓data]` = sudah diverifikasi ke data, temuan konkret menanti.

---

## Prinsip yang berlaku di SELURUH EDA (baca dulu)

- **Denominator konsisten:** placement/acceptance rate **selalu** exclude proses yang _genuinely_ masih aktif. Definisi presisi (lihat Bagian 3.4a-c): setelah `final_outcome` dihitung, exclude hanya baris berstatus {≤1wk, FU1, FU2, FU3} — **Ghosting dihitung sebagai outcome resolved** (kegagalan), bukan dikecualikan. Ini lebih akurat dari sekadar "exclude semua On Progress" (versi itu keliru ikut membuang proses yang sudah ghosting, membuat rate terlihat lebih baik dari kenyataan — 37,1% vs 28,2% yang benar).
- **Rate, bukan count**, saat membandingkan entitas beda ukuran (perusahaan besar vs kecil). Terapkan volume minimum (mis. ≥20 proses) agar rate tidak bising.
- **Dua kolom status beda peran:** `rejection` = outcome final (untuk rate); `progress_student` = posisi funnel (untuk monitoring/ghosting). Jangan tertukar.
- **Aturan Ghosting RESMI panitia (FAQ, wajib dipatuhi untuk BT-05):** Ghosting **diasumsikan dari pihak perusahaan**, dihitung dari **`send_date`** dengan eskalasi: >1 minggu tanpa respons → FU 1; >2 minggu → FU 2; >3 minggu → FU 3; >4 minggu → **Ghosting**. Konsekuensi: (a) **anchor durasi proses = `send_date`, bukan `request_date`** (request→send = lead time CDC, bagian 6.3; send→outcome = waktu di tangan perusahaan); (b) tidak ada konsep "mahasiswa nge-ghost" — analisis ghosting selalu dari sisi perusahaan; (c) lihat catatan di kepala Bagian 3 soal cara menerapkan aturan ini pada data.
- **Nyatakan basis dengan jelas:** seluruh 25.000 mahasiswa vs hanya 10.174 yang pernah masuk proses — selalu sebut yang mana.
- **Untuk analisis per-kandidat-terkirim, pakai `tracking_student` langsung** (sudah 1 baris per mahasiswa-per-perusahaan, ada NIM). Tidak perlu explode `list_nim` — sudah divalidasi identik dengan `jumlah_dikirimkan` (**11.402/11.402** baris terkirim cocok; 598 sisanya Draft tanpa pengiriman, wajar). **Catatan pengumuman panitia:** kolom `list_nim` telah diperbaiki formatnya — pastikan me-load file `tracking_company` **versi terbaru**; kesimpulan "tidak perlu explode" tetap valid setelah perbaikan.
- **Dua lensa keberhasilan berbeda — pakai keduanya:** _per-proses_ (satu baris tracking = satu peluang) vs _per-mahasiswa_ (apakah orang ini akhirnya pernah placement). Keduanya menjawab pertanyaan berbeda.
- **Ghosting selalu disajikan DUA ANGKA.** Karena metrik utama Ghosting (26,7%) berasal dari reklasifikasi ~45% baris On Progress via aturan `send_date` panitia, headline apa pun tentang ghosting **wajib** menyandingkan label-mentah (8,2%) dan rumus-FAQ (26,7%), serta menyebut basisnya "aturan `send_date` resmi panitia". Ini melindungi dari pertanyaan juri soal interpretasi (detail 3.4b).
- **Gerbang funnel historis = `Active+CV`, bukan `Available`.** `ketersediaan` adalah snapshot terkini; memakainya sebagai gerbang membuat funnel patah (5.687 < 10.174 yang masuk proses). Untuk hero funnel end-to-end pakai `Active+CV` (16.732); `Available` hanya untuk funnel kelayakan snapshot 2.1 (detail 8.3).
- **EDA boleh liar, dashboard selektif.** Ikuti cabang menarik saat muncul; yang masuk dashboard hanya 3–4 temuan terkuat per halaman.

---

## Bagian 0 — Setup & Sanity Check

**0.1 Load 6 tabel clean**

- **Tujuan:** memuat data dengan tipe benar (CSV tidak menyimpan dtype).
- **Data:** keenam file `*_clean.csv`.
- **Cara hitung:** `read_csv` + `parse_dates` untuk `created_at`; `request_date`; `sync_date`; `request_date`+`send_date`; `last_update`. Perlakukan ID (C001, TR001, dst.) sebagai string; hati-hati NIM jangan kehilangan leading zero.
- **Output:** konfirmasi `shape` (1.500 / 12.000 / 25.000 / 25.000 / 12.000 / 41.600).

**0.2 Sanity check ringkas**

- **Tujuan:** memastikan integritas terjaga (bukan mengulang audit penuh).
- **Cara hitung:** cek cepat PK unik, tidak ada FK yatim, rentang tanggal wajar.
- **Output:** ringkasan "lolos/anomali". Diketahui: 598 baris `tracking_company` tanpa `send_date`/`list_nim` = Draft (belum dikirim), kondisi valid.

**0.2b Cross-check konsistensi STUDENT ALL ↔ STATUS STUDENT (BT-08 langsung)**

- **Tujuan:** BT-08 secara literal minta "data sinkron antara STUDENT ALL dan STATUS STUDENT" — uji field yang terduplikasi di kedua tabel, bukan cuma proxy staleness.
- **Cara hitung:** merge kedua tabel by NIM; banding `nama`, `program_studi`, `semester` (field yang ada di keduanya); hitung jumlah mismatch + orphan NIM di tiap arah.
- **Output:** satu baris kesimpulan. **Sudah dicek:** 0 NIM orphan (kedua arah), 0 mismatch nama/prodi/semester dari 25.000 baris — BT-08 terpenuhi 100% pada dimensi ini. Jadikan pembuka Bagian 7, baru lanjut ke 7.1-7.3 yang menguji dimensi _usia_ sync (staleness) — dua hal ini saling melengkapi: satu soal _kebenaran_ data, satu soal _kesegaran_ data.

---

## Bagian 1 — Profil Dasar Tiap Tabel `→ H1`

**1.1 Profil company**

- **Tujuan:** komposisi sisi permintaan.
- **Data:** `company_clean` (`company_type`, `skala_perusahaan`, `industry_sector`, `kota`).
- **Cara hitung:** `value_counts()`; `kota` top 10.
- **Output:** 4 bar chart; lihat sektor/tipe/skala dominan & konsentrasi geografis.

**1.2 Profil talent_request**

- **Tujuan:** bentuk permintaan talent.
- **Data:** `talent_request_clean` (`jenis_penempatan`, `working_arrangement`, `bidang_studi_dibutuhkan`, `headcount`).
- **Cara hitung:** `value_counts()` (bidang_studi top-N apa adanya dulu); `describe()` headcount.
- **Output:** 3 bar + 1 histogram; lihat dominasi Magang, sebaran WFO/Hybrid/WFH, headcount median.

**1.3 Profil student_all**

- **Tujuan:** komposisi sisi pasokan.
- **Data:** `student_all_clean` (`program_studi`, `bidang_minat`, `jenis_penempatan_diminati`, `semester`).
- **Cara hitung:** `value_counts()`.
- **Output:** 4 bar chart; lihat prodi terbesar, minat dominan, sebaran semester.

**1.4 Profil status_student**

- **Tujuan:** kesiapan/kelayakan populasi.
- **Data:** `status_student_clean` (`status`, `ketersediaan`, `CV`, `portofolio`, `IPK`).
- **Cara hitung:** `value_counts()` 4 kategorikal; `describe()`+histogram+boxplot IPK.
- **Output:** 4 bar + histogram/boxplot IPK; lihat proporsi Active, kepemilikan CV, sebaran IPK (2.0–4.0, median ~3.3).

**1.5 Rentang tanggal semua tabel**

- **Tujuan:** tak ada tanggal aneh.
- **Data:** semua kolom tanggal.
- **Cara hitung:** `min()`/`max()`; cek `send_date >= request_date`.
- **Output:** tabel rentang; request Feb 2023–Jan 2025, tracking berlanjut hingga Mei 2025.

---

## Bagian 2 — Kelayakan Mahasiswa & Audit Kepatuhan (BT-06) `→ H1`

**2.1 Funnel kelayakan bertingkat**

- **Tujuan:** berapa mahasiswa benar-benar "siap dikirim".
- **Data:** `status_student_clean` (`status`, `CV`, `ketersediaan`).
- **Cara hitung:** kumulatif Total (25.000) → `status`=='Active' → +`CV`=='Ada' → +`ketersediaan`=='Available'. Catat jumlah & % di tiap tahap + % drop.
- **Output:** funnel chart 4 tingkat; lihat penyusutan terbesar di mana.

**2.2 Asumsi IPK threshold (dokumentasikan eksplisit)**

- **Tujuan:** hindari asumsi diam-diam; dokumentasi tak beri angka IPK.
- **Cara hitung:** default **tanpa filter IPK** di funnel utama (semua ≥2.0 valid); tampilkan sensitivitas jumlah eligible pada threshold 2.5/2.75/3.0.
- **Output:** tabel "eligible di berbagai threshold" + catatan keputusan; lihat sensitivitas angka.

**2.3 AUDIT BT-06: apakah yang DIKIRIM memang eligible? `[✓data]`**

- **Tujuan:** inti BT-06 sebenarnya — "memastikan hanya yang memenuhi syarat yang dikirim". Bukan cuma memprofilkan pool, tapi **memverifikasi kepatuhan CDC**.
- **Data:** `tracking_student_clean` (NIM yang dikirim) join `status_student_clean` (`status`, `CV`, `ketersediaan`).
- **Cara hitung:** dari 10.174 mahasiswa terkirim, cek berapa yang non-Active / tanpa CV / bukan Available saat itu.
- **Output:** tabel kepatuhan. **Temuan terverifikasi:** 0 mahasiswa non-Active & 0 tanpa CV yang dikirim → **CDC 100% patuh syarat status+CV** (temuan positif: sistem bekerja benar). Soal `ketersediaan`: 5.138 terkirim ber-`ketersediaan` bukan Available — dan **terverifikasi seluruhnya (100%) berstatus persis "Placed"**, bukan campuran Placed/Tidak Aktif. Artinya mereka non-Available justru **karena sudah berhasil ditempatkan**, bukan pelanggaran matching. Framing kuat untuk laporan: _"seluruh kandidat yang kini non-Available adalah mereka yang sudah sukses placement — snapshot ketersediaan berubah setelah penempatan, bukan bukti CDC mengirim orang yang tidak siap."_

**2.4 Breakdown funnel per program_studi**

- **Tujuan:** prodi paling banyak/sedikit siap kirim.
- **Data:** `status_student_clean` (`program_studi` + mask eligible).
- **Cara hitung:** per prodi, % eligible = eligible/total prodi.
- **Output:** bar chart urut % eligible; bedakan "banyak mahasiswa" vs "banyak yang siap".

**2.5 Cross-tab CV × portofolio**

- **Tujuan:** kelengkapan dokumen.
- **Data:** `CV`, `portofolio`.
- **Cara hitung:** crosstab 2×2 + %; sorot "CV Ada tapi portofolio Tidak Ada".
- **Output:** heatmap/tabel; relevan untuk posisi yang syaratkan portofolio.

**2.6 IPK: Available vs Tidak Aktif**

- **Tujuan:** ketersediaan berkorelasi performa?
- **Data:** `IPK`, `ketersediaan`.
- **Cara hitung:** boxplot IPK per kategori ketersediaan; banding median.
- **Output:** boxplot; lihat apakah Tidak Aktif ber-IPK lebih rendah.

**2.7 Top tools/skill kelompok eligible**

- **Tujuan:** skill dominan kandidat siap (silang BT-01).
- **Data:** `tools` (split koma) + mask eligible.
- **Cara hitung:** explode tools, hitung frekuensi baris eligible, top 15–20.
- **Output:** bar chart; bandingkan nanti dengan skill yang diminta pasar.

---

## Bagian 3 — Beban Proses, Ghosting & Proses Mati (BT-02, BT-05) `→ H4`

> **CATATAN INTERPRETASI ATURAN GHOSTING (baca dulu — keputusan penting):**
> **Cross-check dengan dokumentasi dataset asli:** dokumentasi (hal. 16) mendefinisikan FU1/2/3 sebagai _"tidak ada respons dari perusahaan **atau** mahasiswa"_ dan Ghosting sebagai _"tidak ada respons ... dari **semua** pihak"_ — sengaja ambigu, tidak menentukan pihak mana. **FAQ panitia justru dibuat untuk menjawab ambiguitas persis ini** (pertanyaannya sendiri: "berasal dari pihak perusahaan atau mahasiswa?"), dan menjawab tegas: perusahaan. Jadi FAQ bukan aturan terpisah, tapi klarifikasi resmi atas bagian dokumentasi yang sengaja tidak dijelaskan — tidak ada konflik antar sumber, FAQ hanya lebih spesifik.
>
> FAQ panitia memberi aturan turunan berbasis `send_date` (>1/2/3/4 minggu → FU1/FU2/FU3/Ghosting), tapi dua hal perlu diputuskan sendiri karena FAQ tidak merincinya — sudah diverifikasi ke data:
>
> - **Titik akhir hitungan = `last_update` milik baris itu sendiri** (`hari_terlewat = last_update − send_date`), bukan satu tanggal tetap ("hari ini"/tanggal maksimum dataset). **Alasan (terverifikasi):** memakai tanggal tetap (mis. 17 Mei 2025 — tanggal maks di seluruh data) membuat **100% baris On Progress otomatis "Ghosting"** karena `send_date`-nya terentang sejak Feb 2023 — benar secara hitungan tapi tidak membedakan apa-apa. `last_update` per baris jauh lebih bermakna: mencatat kapan baris itu **terakhir benar-benar disentuh/dicek**.
> - **Uji kepatuhan (3.4a) membuktikan label bawaan `progress_student` (FU1/FU2/FU3/Ghosting) TIDAK patuh ambang mingguan FAQ** (kepatuhan ketat cuma 0–5,9%, semua tahap ~1,5–2× lebih lama dari ambang resminya). **Bukti paling tegas:** dari 5.442 baris `progress_student`='Finish' (dok: _"proses selesai, baik berhasil maupun tidak, tanpa tindak lanjut lebih jauh"_), 2.864 di antaranya berpasangan wajar dengan `rejection` (Placement/Rejection\*/Ghosting — "ditutup dengan hasil jelas"), TAPI **2.578 baris (47,4%) berpasangan dengan `rejection`='On Progress'** — kontradiksi murni: "sudah ditutup, tanpa tindak lanjut" tapi hasilnya "belum ada keputusan". Dan dari sub-kelompok Finish+Ghosting yang tampak wajar pun, **170/516 (33%) belum lewat 28 hari** — jadi bahkan pasangan yang "make sense" tidak sepenuhnya patuh waktu. Karena itu, **rumus `send_date` diterapkan SERAGAM ke seluruh baris yang belum punya keputusan dari perusahaan** (`rejection` ∈ On Progress atau Ghosting) — bukan cuma ke baris yang belum berlabel, dan bukan cuma ke On Progress. Mencampur "pakai label kalau ada, derive kalau tidak ada" akan menghasilkan metrik yang tidak konsisten, karena label yang ada pun tidak sepenuhnya bisa dipercaya.
>   **Keputusan operasional (defensible, tanpa perlu tanya panitia):**
>
> 1. **Baris dengan keputusan nyata dari perusahaan** (`rejection` ∈ Placement/Rejection\*) **bukan kandidat ghosting** — dipakai apa adanya, tak peduli berapa lama prosesnya (mereka sudah direspons).
> 2. **Baris tanpa keputusan final** (`rejection` ∈ On Progress/Ghosting) **diklasifikasi ulang seragam** pakai `send_date` + ambang mingguan (poin 3.4b) — ini metrik Ghosting resmi yang dipakai di seluruh sisa analisis (3.5–3.7) dan di Bagian 4.
> 3. **Anchor semua durasi proses = `send_date`** (poin 3.3, 3.7 direvisi dari `request_date`).

**3.1 Concurrent process load per NIM**

- **Tujuan:** beban "satu mahasiswa banyak proses" (dok 2.3).
- **Data:** `tracking_student_clean` (`NIM`).
- **Cara hitung:** `groupby('NIM').size()`; median, % ikut ≥2 dan ≥3. Basis: dari 10.174 yang pernah proses, bukan 25.000.
- **Output:** histogram; median ~4, ekor sampai 19. Nyatakan basis jelas.

**3.2 Distribusi tracking_student per id_tracking_company**

- **Tujuan:** posisi mana serap kandidat jauh lebih banyak.
- **Data:** `id_tracking_company`.
- **Cara hitung:** groupby size + describe.
- **Output:** histogram; outlier (rata-rata ~3.6, maks 8).

**3.3 Waktu tinggal per tahap** _(anchor send_date, sesuai FAQ)_

- **Tujuan:** tahap mana paling lama mengendap di tangan perusahaan.
- **Data:** `tracking_student_clean` (`progress_student`, `last_update`) + `tracking_company_clean` (`send_date`) via `id_tracking_company`.
- **Cara hitung:** `durasi_hari = last_update − send_date` (anchor `send_date`, bukan `request_date`, karena FAQ menghitung "tanpa respons" sejak kandidat dikirim), kelompok per `progress_student`, ambil median. **Pisahkan tahap menjadi DUA jalur saat plotting** (lihat catatan monotonisitas di Output): (a) jalur substantif — Selecting → CDC Briefing → Study Case → Interview User → Final Interview; (b) jalur follow-up/eskalasi — FU 1 → FU 2 → FU 3 → Ghosting. Atau, urut boxplot berdasarkan median (bukan urutan funnel).
- **Output:** boxplot per tahap; tahap terlama = bottleneck. Catatan: `last_update` = update terakhir, jadi durasi "sejak dikirim sampai kondisi terakhir" — bukan durasi murni satu tahap. **Terverifikasi — monotonisitas berlaku PER JALUR, bukan gabungan:** dalam jalur substantif median naik 8 (Selecting) → 19 (Briefing/Study Case) → 29 (Interview User) → 41 (Final Interview); dalam jalur follow-up median naik 22 (FU 1) → 33 (FU 2) → 45 (FU 3) → 61 (Ghosting), konsisten dengan eskalasi FAQ. **⚠️ Jangan plot semua tahap dalam satu urutan funnel** — FU 1 (22 hari) median-nya lebih pendek dari Final Interview (41 hari), jadi rangkaian gabungan akan tampak "turun lalu naik" (cekungan di FU 1) dan bisa disalahartikan juri sebagai tidak monoton. Ini logis: follow-up dipicu ketiadaan respons dan bisa terjadi lebih dini, bukan kelanjutan linear dari tahap substantif. Menyajikan dua jalur terpisah membuat pola naik di masing-masing jelas dan tak ambigu.

**3.4a Uji kepatuhan: apakah label bawaan cocok dengan ambang mingguan FAQ? `[✓data]`**

- **Tujuan:** menjawab langsung "apakah rumus FAQ sesuai dengan data yang ada" sebelum dipakai untuk klasifikasi — pertanyaan yang wajar ditanyakan juri, dan harus dijawab dengan bukti, bukan asumsi.
- **Data:** `tracking_student_clean` (`progress_student`, `last_update`) + `tracking_company_clean` (`send_date`).
- **Cara hitung:** `hari_terlewat = last_update − send_date`; untuk tiap label (FU 1/FU 2/FU 3/Ghosting), hitung % baris yang `hari_terlewat`-nya benar-benar jatuh di jendela mingguan resminya, memakai konvensi **`(batas_bawah, batas_atas]`** (eksklusif bawah, inklusif atas): FU1 `(7, 14]`, FU2 `(14, 21]`, FU3 `(21, 28]`, Ghosting `> 28`; bandingkan median tiap label ke ambang; cek monotonisitas urutan. **⚠️ Konvensi bracket harus ditulis eksplisit di kode** — kalau memakai `[bawah, atas)` (inklusif bawah, eksklusif atas), angka kepatuhan berubah jadi 0% di semua tahap (karena batas bawah tiap label persis jatuh di ujung jendela). Nyatakan `(lo, hi]` agar hasil reproducible saat juri menghitung ulang.
- **Output:** tabel kepatuhan + narasi. **Temuan terverifikasi:** kepatuhan ketat sangat rendah (FU1 5,9%, FU2 3,9%, FU3 0%) — median tiap label sekitar 1,5–2× lebih lama dari ambang resminya, tapi **urutan tetap monoton sempurna** dan jarak antar tahap konsisten (~11–16 hari). **Pola pergeseran sistematis (bukti pendukung kuat):** nilai `hari_terlewat` **minimum** tiap label bergeser ~1 minggu lebih lambat dari FAQ — FU1 mulai di **14 hari** (2× ambang 7-hari), FU2 di **21**, FU3 di **30** — artinya sistem sumber memberi label satu tingkat "terlambat" secara konsisten dibanding aturan FAQ. Label `progress_student`='Ghosting' sendiri 100% patuh (min 30 hari) — _tapi ini baru sebagian dari `rejection`='Ghosting'; lihat 3.4b untuk anomali di sub-kelompok lain (`progress_student`='Finish') yang ternyata tidak sepenuhnya patuh._ **Kesimpulan:** data tidak dibangun dari penerapan harfiah rumus FAQ ke `last_update`; FAQ berfungsi sebagai panduan metodologi, bukan cetak biru pembuatan data. **Ini alasan kenapa 3.4b menerapkan rumus seragam ke seluruh bucket "tidak ada keputusan dari perusahaan", alih-alih memilah berdasarkan label mana yang sudah ada** — memakai label bawaan apa adanya justru akan mewariskan pergeseran ~1 minggu ini ke seluruh metrik.

**3.4b Klasifikasi final_outcome konsisten: 2 bucket, 1 rumus `[+][✓data]`** _(metodologi utama BT-05 — insight terkuat)_

- **Tujuan:** menghasilkan SATU metrik Ghosting yang konsisten. **Koreksi penting (setelah pengecekan lebih dalam):** dokumentasi menyebut `rejection` sebagai "status akhir", tapi dua hal membuat label bawaan tidak sepenuhnya bisa dipercaya apa adanya: (1) nilai `On Progress` secara definisi bukan status akhir; (2) label `Ghosting` sendiri ternyata **campuran** — 2.905 baris berlabel `progress_student`='Ghosting' semuanya patuh aturan (min 30 hari), TAPI 516 baris lain berlabel `progress_student`='Finish' meski `rejection`-nya 'Ghosting' (kombinasi anomali) — dan **170 dari 516 baris itu (33%) punya `hari_terlewat` ≤28 hari**, belum memenuhi syarat FAQ untuk disebut Ghosting. Karena label mentah tidak bisa dipercaya penuh bahkan untuk yang sudah "final", **rumus diterapkan seragam ke SELURUH baris tanpa keputusan dari perusahaan** (bukan cuma yang belum berlabel).
- **Data:** `tracking_student_clean` (`rejection`, `progress_student`, `last_update`) + `tracking_company_clean` (`send_date`) via `id_tracking_company`.
- **Cara hitung:** pisah 41.600 baris jadi 2 bucket berdasarkan makna —
  1. **Bucket "ada keputusan nyata dari perusahaan"**: `rejection` ∈ {Placement, Rejection Screening CV, Rejection Interview User, Rejection Study Case, Rejection Final Interview} — perusahaan sudah merespons; dipakai apa adanya.
  2. **Bucket "tidak ada keputusan dari perusahaan"**: `rejection` ∈ {On Progress, Ghosting} — terapkan **satu rumus seragam** ke SELURUH bucket ini (20.909 baris, termasuk yang sudah berlabel Ghosting): `hari_terlewat = last_update − send_date`; >28→Ghosting, >21→FU3, >14→FU2, >7→FU1, else ≤1wk.
     Gabungkan kedua bucket jadi satu kolom `final_outcome`.
- **Output:** tabel `final_outcome` tunggal untuk semua 41.600 baris (siap jadi sumber donut/bar chart utama H4). **Temuan terverifikasi:** dalam bucket "tidak ada keputusan" (20.909 baris), rate Ghosting versi **label mentah 16,4%**, versi **rumus FAQ konsisten 53,1%**. Rekonsiliasi: dari 3.421 baris berlabel Ghosting mentah, 3.251 tetap Ghosting setelah diuji ulang (2.905 yang memang patuh + 346 dari sub-kelompok Finish yang kebetulan sudah lewat 28 hari), 170 turun status (belum cukup lama); ditambah 7.843 baris On Progress yang ternyata sudah lewat 28 hari → **total Ghosting akhir = 11.094 (26,7% dari 41.600) — outcome TERBESAR di seluruh sistem, melebihi Placement (8.955, 21,5%)**.
  - **⚠️ WAJIB disajikan sebagai DUA ANGKA berdampingan, jangan hanya yang 26,7%.** Lonjakan dari label mentah **3.421 (8,2%)** ke rumus-FAQ **11.094 (26,7%)** adalah 3,2× — bersumber dari **44,8% baris On Progress (7.843/17.488)** yang direklasifikasi via `last_update − send_date > 28`. Ini keputusan interpretatif yang **defensible tapi bisa dipertanyakan juri**, karena proxy `last_update − send_date` mengukur "lama sejak dikirim sampai terakhir disentuh", bukan "lama perusahaan diam tanpa respons" (data tak punya timestamp antar-tahap untuk memisahkan proses lambat-tapi-jalan dari yang benar-benar ghost). Cara aman: tampilkan **Ghosting label-mentah 8,2% vs Ghosting rumus-FAQ 26,7%** dengan narasi bahwa label mentah *under-report* (banyak proses ditutup tanpa dicatat, lihat 3.4d), dan **frame headline sebagai "berdasarkan aturan `send_date` resmi panitia"** — menegaskan ini turunan aturan resmi FAQ, bukan klaim sepihak.
- **Headline dashboard (versi aman):** "Berdasarkan aturan `send_date` panitia, Ghosting menjadi outcome paling umum di seluruh sistem placement (26,7%) — melampaui keberhasilan penempatan (21,5%); jauh lebih besar dari yang tertangkap label mentah (8,2%), karena banyak proses berhenti tanpa dicatat."

**3.4c Breakdown lokasi & waktu ghosting terbesar `[✓data]`**

- **Tujuan:** dari `final_outcome`=='Ghosting' hasil 3.4b, cari di mana konsentrasinya — posisi, perusahaan, dan seberapa lama sebelum akhirnya dianggap ghosting.
- **Data:** hasil 3.4b (`final_outcome`) + `tracking_student_clean` (`company`, `position`) + `hari_terlewat`.
- **Cara hitung:** filter `final_outcome`=='Ghosting'; top posisi & company (rate, volume min ≥20); distribusi `hari_terlewat` untuk kelompok ini (berapa median hari sebelum akhirnya "menyerah").
- **Output:** tabel top posisi/company ghosting terbanyak + histogram `hari_terlewat`. Ini yang actionable untuk tim CDC — ke mana follow-up harus diprioritaskan.

**3.4d Catatan kualitas data: proses ditutup tanpa hasil tercatat `[✓data]`**

- **Tujuan:** flag celah pencatatan operasional tim CDC — bukan untuk dashboard utama, tapi layak masuk laporan sebagai rekomendasi perbaikan proses.
- **Data:** `tracking_student_clean` (`progress_student`=='Finish' & `rejection`=='On Progress').
- **Cara hitung:** filter kombinasi ini; breakdown per company/posisi kalau ingin tahu di mana paling sering terjadi.
- **Output:** angka + tabel breakdown. **Temuan terverifikasi:** **2.578 baris (47,4% dari seluruh baris berlabel Finish)** ditandai "proses selesai, tanpa tindak lanjut" (sesuai definisi dokumentasi) tapi kolom hasilnya masih "On Progress" — kontradiktif. Ini kemungkinan besar proses yang **ditutup CDC tanpa pernah dicatat hasil akhirnya**. Rekomendasi untuk laporan: tim CDC perlu SOP "wajib isi alasan penutupan" agar data hasil placement tidak bolong.

**3.5 Tahap terakhir sebelum ghosting**

- **Tujuan:** di titik mana perusahaan menghilang — di tahap substantif mana kandidat terakhir kali "terlihat" sebelum dianggap ghosting.
- **Data:** `final_outcome`=='Ghosting' (hasil 3.4b) + `progress_student` asli (tahap substantif terakhir yang tercatat sebelum masuk hitungan ghosting).
- **Cara hitung:** filter + `value_counts()` pada `progress_student` asli dari baris yang `final_outcome`-nya Ghosting.
- **Output:** bar chart; apakah kandidat "hilang" lebih sering di tahap awal (Selecting/Briefing) atau lanjut (Interview/Final Interview).

**3.6 Ghosting rate per jenis_penempatan & per company** _(pakai `final_outcome`)_

- **Tujuan:** skema/perusahaan rawan ghosting.
- **Data:** `final_outcome` (hasil 3.4b), `jenis_penempatan`, `company`.
- **Cara hitung:** ghosting rate = (`final_outcome`=='Ghosting') / total, per skema & per company (volume min ≥20), top 10.
- **Output:** 2 bar chart; pakai **rate** bukan count.

**3.7 Lama proses Ghosting vs Placement** _(anchor send_date)_

- **Tujuan:** ghosting butuh waktu lebih lama sebelum menyerah, dibanding proses yang berhasil?
- **Data:** `tracking_student_clean` (`last_update`) + `tracking_company_clean` (`send_date`) + `final_outcome` (hasil 3.4b).
- **Cara hitung:** `durasi_hari = last_update − send_date` (sama anchor 3.3), banding `final_outcome`=='Ghosting' vs `rejection`=='Placement'.
- **Output:** boxplot 2 kelompok; selisih median. **Catatan:** `last_update` proxy untuk tanggal keputusan/placement (sama asumsi 3.4/4.11) — sajikan sebagai estimasi, bukan durasi presisi.

---

## Bagian 4 — Funnel Hasil & Keberhasilan (BT-04, BT-07) `→ H3`

**4.1 Definisi placement rate yang benar (subbagian kunci) `[✓data]`**

- **Tujuan:** memperbaiki bug denominator + dokumentasi koreksi, sekarang dengan basis paling presisi (pakai `final_outcome` dari 3.4b).
- **Data:** `final_outcome` (hasil 3.4b).
- **Cara hitung:** tiga angka berdampingan — **(a) Salah** (basis penuh, termasuk semua baris apa adanya). **(b) Lama** (exclude seluruh `rejection`=='On Progress' mentah — problematik karena ikut membuang baris yang sebenarnya sudah ghosting). **(c) Benar** (denominator = total dikurangi baris yang _genuinely_ masih aktif menurut `final_outcome` ∈ {≤1wk, FU1, FU2, FU3}; Ghosting dihitung sebagai outcome resolved, bukan dikecualikan).
- **Output:** tabel 3 angka + catatan. **Temuan terverifikasi:** (a) 21,5% — terlalu rendah, bug asli. (b) 37,1% — versi "koreksi lama", ternyata **terlalu optimis** karena turut mengecualikan proses yang sudah ghosting (seharusnya dihitung sebagai kegagalan). (c) **28,2%** — angka paling akurat, dipakai sebagai basis resmi di seluruh Bagian 4 selanjutnya.
  - **⚠️ Label tegas "rate" vs "share" di dashboard (mudah tertukar — keduanya ~20-an%):** **28,2% = placement _rate_** (Placement ÷ baris resolved, metrik keberhasilan di 4.x); **21,5% = placement _share_** (Placement ÷ seluruh 41.600 baris, komposisi outcome di 3.4b/4.3). Dua-duanya benar tapi berbeda denominator — beri label eksplisit agar pembaca tidak menyimpulkan angka yang salah.

**4.2 PER-STUDENT outcome — lensa manusiawi `[+][✓data]`** _(hilang di rencana lama)_

- **Tujuan:** bukan "berapa % proses sukses" tapi "berapa % **mahasiswa** akhirnya dapat placement" — dan apakah gigih mencoba berbuah hasil.
- **Data:** `tracking_student_clean` (`NIM`, `rejection`).
- **Cara hitung:** per NIM, `ever_placed` = ada ≥1 Placement; hitung % ever-placed dari 10.174 partisipan; lalu silang dengan jumlah proses yang diikuti (bin: 1/2/3/4-5/6+).
- **Output:** angka headline + bar chart ever-placed rate per jumlah proses. **Temuan terverifikasi:** **56,6% partisipan akhirnya pernah placement**; dan **persistence pays off** — ikut 1 proses hanya 22%, ikut 6+ proses **81,7%**. Cerita kuat: makin banyak mencoba, makin besar peluang. Calon headline H3.

**4.3 Distribusi final_outcome keseluruhan**

- **Tujuan:** gambaran hasil akhir yang sudah konsisten (bukan campuran label mentah).
- **Data:** `final_outcome` (hasil 3.4b).
- **Cara hitung:** `value_counts()` + %.
- **Output:** bar/donut. **Sudah terverifikasi (3.4b):** Ghosting (26,7%, berdasarkan aturan `send_date` panitia) > Placement (21,5%) — outcome terbesar justru kegagalan karena tak direspons, bukan keberhasilan. Tampilkan pendamping label-mentah (8,2%) sesuai catatan penyajian 3.4b agar lonjakan reklasifikasi transparan.

**4.4 Placement rate per program_studi**

- **Tujuan:** prodi paling sukses.
- **Data:** `tracking_student_clean` (`NIM`) + `final_outcome` (hasil 3.4b) + `student_all_clean` (`program_studi`).
- **Cara hitung:** denominator = exclude {≤1wk, FU1, FU2, FU3} (masih genuinely aktif); per prodi rate = Placement/(baris resolved).
- **Output:** bar chart urut tertinggi; **terverifikasi: rentang ~10 pp** (min ~21,6% hingga maks ~31,7% antar prodi, volume ≥20) → didalami di 4.10.

**4.5 Placement rate per internship_semester**

- **Tujuan:** semester tinggi lebih mudah diterima?
- **Data:** `internship_semester` + `final_outcome`.
- **Cara hitung:** denominator = exclude {≤1wk, FU1, FU2, FU3}; rate per semester.
- **Output:** bar/line; tren naik/turun/datar.

**4.6 Placement rate per jenis_penempatan**

- **Tujuan:** skema paling tinggi.
- **Data:** `jenis_penempatan` + `final_outcome`.
- **Cara hitung:** denominator = exclude {≤1wk, FU1, FU2, FU3}; rate per Magang/Part-time/Full-time.
- **Output:** bar chart; Full-time lebih selektif?

**4.7 Placement rate per renumerasi (3 tingkat kompensasi) `[+][✓data]`**

- **Tujuan:** dimensi bayaran yang belum tersentuh — apakah posisi berbayar lebih sukses / lebih rendah ghosting.
- **Data:** `talent_request_clean` (`renumerasi`) join ke tracking via `id_talent_req`→`tracking_company`→`tracking_student` (jalur ini **terverifikasi 0 unmatched**, 41.600/41.600 baris ketemu); `tracking_student_clean` (`rejection`).
- **Cara hitung:** ekstrak **3 kategori, bukan biner** — mengandung 'Rp' → **Berbayar Nominal**; == 'Non-Paid' → **Non-Paid**; sisanya (== 'Uang transport saja') → **Transport Saja**. _(Terverifikasi: binari sederhana salah — 1.493/12.000 baris atau 12,4% berisi "Uang transport saja", tidak cocok kategori 'Rp' atau 'Non-Paid'.)_ Opsional lanjutan: untuk kategori Berbayar Nominal, bin lagi rendah/sedang/tinggi dari nominal. Hitung placement rate & ghosting rate (pakai `final_outcome`=='Ghosting') per 3 kategori.
- **Output:** bar chart placement/ghosting per 3 kategori kompensasi. Komposisi: 78,6% Berbayar Nominal, 12,4% Transport Saja, 9,0% Non-Paid. Hipotesis diuji: apakah Non-Paid/Transport Saja lebih sering ghosting dibanding Berbayar Nominal?

**4.8 Acceptance rate per company + tipe/skala**

- **Tujuan:** perusahaan mana paling menerima, beda per tipe/skala?
- **Data:** tracking_student join company via `id_tracking_company`→`tracking_company`→`company` (`company_type`, `skala_perusahaan`) + `final_outcome`.
- **Cara hitung:** denominator = exclude {≤1wk, FU1, FU2, FU3}; rate per company (volume min); agregasi per tipe & skala.
- **Output:** ranking + 2 bar chart; Startup vs Corporate vs BUMN beda perilaku?

**4.9 Rekap tabel BT-07 eksplisit**

- **Tujuan:** penuhi "rekap per semester/prodi/perusahaan/jenis penempatan".
- **Data:** gabungan tracking_student + student_all + company + `final_outcome`.
- **Cara hitung:** pivot kolom: dikirim/Placement/Rejected (semua Rejection\*)/Ghosting/masih aktif (≤1wk+FU1-3) — per company; ulang per periode akademik, per program_studi, dan per jenis_penempatan (4 dimensi sesuai definisi BT-07: semester/prodi/perusahaan/jenis penempatan).
- **Output:** tabel rekap (sumber tabel dashboard); cek konsistensi total.

**4.10 Multivariat: apakah gap prodi bertahan setelah dikontrol `[+]`**

- **Tujuan:** bedakan "prodi ini memang lebih diterima" vs "prodi ini kebetulan melamar ke perusahaan yang lebih mudah".
- **Data:** tracking_student + student_all (`program_studi`) + company (`company_type`/`skala`) + status_student (`IPK`, `tools`).
- **Cara hitung:** placement rate per prodi **di dalam** tiap tipe/skala (stratifikasi) — gap menyusut? Opsional: banding IPK & top tools prodi rate tinggi vs rendah.
- **Output:** heatmap prodi × tipe perusahaan + interpretasi; apakah ranking prodi berubah setelah dikontrol.

**4.11 Velocity / time-to-placement `[+]`**

- **Tujuan:** seberapa cepat sampai placement, siapa cepat/lambat memutuskan.
- **Data:** `tracking_student_clean` (Placement, `last_update`) + `tracking_company_clean` (`send_date`).
- **Cara hitung:** durasi = `last_update − send_date` untuk Placement; median per tipe/skala/jenis.
- **Output:** boxplot time-to-placement per segmen; segmen cepat vs berlarut. **⚠️ Catatan asumsi (konsisten dengan 3.4):** untuk baris Placement, `last_update` dipakai sebagai **proxy tanggal placement** — data tak punya timestamp keputusan terpisah, jadi ini "hari sejak dikirim sampai update terakhir", bukan durasi keputusan murni. Sebut "estimasi waktu-ke-placement", jangan diklaim sebagai durasi presisi. Asumsi yang sama berlaku di 3.7 (durasi Ghosting vs Placement).

**4.12 Tren placement rate per bulan-kuartal**

- **Tujuan:** membaik/memburuk sepanjang 2 tahun.
- **Data:** `last_update`, `final_outcome`.
- **Cara hitung:** resample per bulan/kuartal; placement rate (denominator = exclude {≤1wk, FU1, FU2, FU3}). Waspadai periode akhir bias "belum selesai".
- **Output:** line chart; tren + musiman.

**4.13 Breakdown tahap penyumbang penolakan terbesar**

- **Tujuan:** di tahap mana paling banyak gugur.
- **Data:** `rejection` (sub Rejection Screening CV/Interview User/Study Case/Final Interview).
- **Cara hitung:** `value_counts()` + % dari total penolakan.
- **Output:** bar chart; sinyal awal Interview User dominan → actionable persiapan kandidat.

---

## Bagian 5 — Kualitas Matching & Mismatch (BT-01) `→ H2`

_(Catatan: pakai `tracking_student` langsung untuk per-kandidat, tidak perlu explode `list_nim`.)_

_(Catatan: kriteria "ketersediaan" pada BT-01 sudah dijawab di 2.3 — audit menunjukkan 0 mahasiswa terkirim berstatus non-Active/tanpa CV, dan ketidaksesuaian ketersediaan yang ada murni efek snapshot pasca-proses, bukan pelanggaran matching. Tidak diulang di sini.)_

> **💡 SINTESIS asimetri matching (kandidat headline H2, terverifikasi dari 5.1+5.2+5.4):** sistem matching CDC menegakkan **satu** dimensi secara penuh dan mengabaikan dua lainnya — **prodi 100% cocok** (5.1), tapi **semester tidak ditegakkan** (50,5% di bawah syarat, 5.2) dan **lokasi tidak ditegakkan** (mismatch WFO 88,4% ≈ acak 86,8%, 5.4). Insight ini kuat justru karena **jujur pada sifat data** (memakai baseline & distribusi, bukan menuduh CDC lalai) — dan lebih menarik daripada tiga temuan terpisah. Sajikan sebagai satu narasi: _"matching berbasis bidang studi, buta terhadap semester & geografi."_ **Peringatan penyajian:** jangan pecah jadi klaim "CDC salah kirim 50%/88% kandidat" — itu over-claim yang mudah dipatahkan; bingkai sebagai karakteristik sistem matching yang terungkap dari data.

**5.1 Kecocokan prodi vs bidang dicari**

- **Tujuan:** verifikasi kirim sesuai bidang.
- **Data:** `tracking_student_clean` (`NIM`, join `tracking_company` untuk `bidang_studi_dicari`) + `student_all_clean` (`program_studi`).
- **Cara hitung:** cek `program_studi` termasuk dalam `bidang_studi_dicari` (multi-nilai koma); % cocok.
- **Output:** % kecocokan + contoh ketidakcocokan. **Terverifikasi: 100,0% cocok** — setiap kandidat terkirim program studinya termasuk dalam `bidang_studi_dicari`. Ini penting sebagai **jangkar kontras**: prodi adalah satu-satunya dimensi matching yang ditegakkan penuh di data (bandingkan 5.2 semester & 5.4 lokasi yang ternyata TIDAK ditegakkan — lihat catatan di sana).

**5.2 Kecocokan semester vs minimum_semester `[✓data]`**

- **Tujuan:** apakah filter semester ditegakkan? (bukan sekadar "ada yang tidak memenuhi", tapi **apakah dimensi ini di-enforce sama sekali**).
- **Data:** `tracking_student_clean` (`internship_semester`, join `talent_request` via tracking_company untuk `minimum_semester`).
- **Cara hitung:** banding semester vs `minimum_semester`; % pelanggaran; **cek distribusi selisih `(minimum_semester − internship_semester)`** untuk membedakan "beberapa human error" vs "tidak ada filter sama sekali".
- **Output:** angka + interpretasi. **⚠️ Terverifikasi — sajikan sebagai OBSERVASI KUALITAS DATA, bukan "human error CDC":** **50,5% baris (21.021)** punya `internship_semester` < `minimum_semester` — bukan segelintir, tapi separuh. Distribusi selisih menyebar mulus (−7 s/d +5, terpusat ~0/+1), pola khas dua variabel **yang di-generate independen** — artinya filter semester **tidak ditegakkan** di data, kontras tajam dengan 5.1 (prodi 100% ditegakkan). **Jangan klaim "CDC salah kirim separuh kandidat"** (kemungkinan besar artefak, bukan keputusan CDC nyata). Framing yang aman & lebih tajam: _"sistem matching menegakkan bidang studi (100%) tapi tidak semester — 50,5% kandidat di bawah syarat semester nominal."_ Ini temuan tentang **cara kerja matching**, bukan tuduhan kelalaian.

**5.3 IPK minimum tersirat di deskripsi_requirement (opsional)**

- **Tujuan:** syarat IPK tersembunyi di teks?
- **Data:** `deskripsi_requirement`.
- **Cara hitung:** regex "IPK/GPA ≥ x.xx"; jika berantakan → skip + catat alasan.
- **Output:** ringkasan berapa request sebut IPK; jangan dipaksakan bila tak reliabel.

**5.3b Kecocokan tools yang diminta vs tools kandidat `[opsional]`**

- **Tujuan:** BT-01 eksplisit sebut "tools" sebagai kriteria matching — belum diuji di plan ini.
- **Data:** `talent_request_clean.deskripsi_requirement` (teks bebas) + `status_student_clean.tools` (kandidat terkirim).
- **Cara hitung:** ekstrak keyword tools dari deskripsi_requirement dengan mencocokkan ke daftar tools unik yang sudah ada di kolom `tools` (jadi tidak perlu NLP rumit — cukup substring match ke vocabulary yang sudah dikenal). Kalau match rate terlalu rendah/tidak reliabel, hentikan dan dokumentasikan kenapa (sama seperti 5.3 untuk IPK).
- **Output:** % request yang tools-nya bisa diekstrak + % kandidat terkirim yang overlap tools-nya dengan requirement. Kalau tidak reliabel: satu kalimat di laporan "tools tidak diuji kuantitatif karena deskripsi_requirement tidak terstruktur", jangan dibiarkan diam-diam hilang dari cakupan BT-01.

**5.4 Domisili vs kota perusahaan untuk WFO `[✓data]`**

- **Tujuan:** apakah lokasi diperhitungkan dalam matching WFO? (bukan asumsi "mismatch = friksi", tapi **apakah beda dari acak**).
- **Data:** `tracking_student_clean` join `talent_request` (`working_arrangement`) + `company` (`kota`) + `status_student` (`domisili`).
- **Cara hitung:** filter WFO; cek `domisili`==`kota`; % beda kota. **WAJIB bandingkan ke baseline acak** = 1 − Σ(share domisili²); tanpa baseline, angka mismatch tak bisa diinterpretasi.
- **Output:** % beda-kota WFO + **perbandingan baseline**. **⚠️ Terverifikasi — SAJIKAN DENGAN BASELINE, jangan sebagai "friksi logistik CDC":** mismatch WFO aktual **88,4%** vs baseline acak **86,8%** — **nyaris identik**, artinya domisili mahasiswa **independen** terhadap kota perusahaan (lokasi tidak dikondisikan di data, konsisten dengan asimetri 5.1/5.2). **Jangan klaim "88% kandidat WFO menghadapi friksi logistik"** — itu over-claim pada yang esensinya acak; kalau juri tanya "beda dari acak?", baseline 86,8% membongkarnya. Framing aman: _"matching tidak mempertimbangkan geografi — mismatch WFO (88,4%) tak berbeda dari penempatan acak (86,8%)."_ Melengkapi cerita: sistem menegakkan prodi, tapi bukan semester maupun lokasi.

**5.5 Skor kecocokan gabungan**

- **Tujuan:** metrik ringkas pelanggaran per pengiriman.
- **Data:** hasil 5.1+5.2+5.4.
- **Cara hitung:** per kandidat, jumlah pelanggaran (prodi tak cocok +1, semester kurang +1, beda kota WFO +1) = 0–3.
- **Output:** histogram 0/1/2/3; % pengiriman bersih vs bermasalah.

**5.6 Preferensi vs aktual jenis_penempatan (opsional)**

- **Tujuan:** mahasiswa dapat jenis yang diinginkan?
- **Data:** `student_all` (`jenis_penempatan_diminati`) + `tracking_student` (Placement, `jenis_penempatan`).
- **Cara hitung:** untuk Placement, banding preferensi vs aktual; % sesuai.
- **Output:** % terpenuhi + crosstab; banyak yang "settle"?

**5.7 Mismatch supply–demand `[+]`** _(jantung sistem placement)_

- **Tujuan:** peta ketimpangan pasokan vs permintaan per bidang.
- **Data:** supply `student_all_clean` (`program_studi`/`bidang_minat`); demand `talent_request_clean` (`bidang_studi_dibutuhkan`/`industry_sector`).
- **Cara hitung:** _(perhatian: `bidang_studi_dibutuhkan` multi-nilai dipisah koma — terverifikasi 10.284/12.000 baris talent_request punya >1 bidang; harus di-split & explode dulu sebelum dihitung, jangan diperlakukan sebagai satu kategori utuh)_. Normalisasi kategori kedua sisi; share supply vs share demand per bidang; selisih = surplus (mahasiswa banyak, demand sedikit) atau defisit (demand tinggi, kandidat langka).
- **Output:** diverging bar/heatmap; bidang surplus & langka = rekomendasi strategis kuat.

**5.8 Geografi supply vs demand `[+]`**

- **Tujuan:** konsentrasi geografis demand vs supply.
- **Data:** demand `company_clean.kota`; supply `status_student_clean.domisili`.
- **Cara hitung:** jumlah perusahaan/request per kota vs mahasiswa per domisili; identifikasi kota timpang. _(Terverifikasi: penamaan kota konsisten antar tabel, tak perlu normalisasi teks. Semua 11 kota company ada di domisili; domisili punya 4 kota tambahan tanpa satupun perusahaan — Klaten, Magelang, Karanganyar, Boyolali — ini sendiri temuan: mahasiswa di kota-kota ini otomatis harus WFO ke kota lain atau bergantung remote.)_
- **Output:** bahan peta interaktif (tabel kota); relevan untuk WFO.

---

## Bagian 6 — Manajemen Talent Request (BT-03) `→ H2/H4`

**6.1 Tren talent_request per bulan**

- **Tujuan:** musiman permintaan.
- **Data:** `request_date`.
- **Cara hitung:** resample bulanan.
- **Output:** line chart; puncak/lembah (awal semester?).

**6.2 Distribusi headcount**

- **Tujuan:** ukuran tipikal request.
- **Data:** `headcount`.
- **Cara hitung:** histogram + describe.
- **Output:** histogram; mayoritas kecil (median 2)?

**6.3 Lead time request→send, headcount besar vs kecil** _(ini sisi CDC, beda dari anchor ghosting)_

- **Tujuan:** request besar direspons lebih lambat oleh CDC? Catatan: `request_date`→`send_date` = kecepatan **CDC** memproses (bukan "tanpa respons perusahaan"; itu di Bagian 3 yang anchor-nya `send_date`).
- **Data:** `tracking_company_clean` (`request_date`, `send_date`, `jumlah_permintaan`).
- **Cara hitung:** `lead_time = send_date − request_date` (abaikan Draft yang `send_date` kosong); kelompok besar (≥4) vs kecil (≤2); banding median.
- **Output:** boxplot lead time per kelompok.

**6.4 % request Draft (tak pernah diproses)**

- **Tujuan:** pola request mangkrak.
- **Data:** `tracking_company_clean` (`progress`=='Draft', join company `company_type`, `jenis_penempatan`).
- **Cara hitung:** % Draft; breakdown per tipe & skema (rate).
- **Output:** bar chart proporsi Draft per segmen.

**6.5 Talent request tidak terpenuhi**

- **Tujuan:** request selesai tapi gagal menempatkan.
- **Data:** `tracking_company_clean` (`progress`, `jumlah_dikirimkan`, headcount, `id_tracking_company`) + `tracking_student_clean` (Placement per company).
- **Cara hitung:** untuk Closed, hitung Placement aktual; tandai 0 placement atau `jumlah_dikirimkan`<headcount; cari pola bidang/sektor.
- **Output:** tabel tak terpenuhi + breakdown bidang; bidang langka/syarat ketat.

**6.6 Skor prioritas request `[+]`**

- **Tujuan:** request mana didahulukan.
- **Data:** `talent_request_clean` (`request_date`, `headcount`, `jenis_penempatan`) + status pemenuhan 6.5.
- **Cara hitung:** skor = norm(lama menunggu) + norm(headcount) + bobot jenis; urut request terbuka/tak terpenuhi.
- **Output:** tabel top prioritas (rekomendasi actionable).

**6.7 Distribusi sumber_baris_form `[+]`**

- **Tujuan:** kanal masuk request (audit & prioritas).
- **Data:** `sumber_baris_form`.
- **Cara hitung:** `value_counts()`; opsional silang % Draft (kanal tertentu lebih mangkrak?).
- **Output:** bar + opsional crosstab kanal×Draft; dominasi Google Form vs manual.

---

## Bagian 7 — Sinkronisasi Data Mahasiswa (BT-08) `(diagnostik — untuk laporan, BUKAN dashboard)`

> **Ekspektasi realistis (terverifikasi):** BT-08 punya dua sisi, dan keduanya sudah "selesai" secara temuan — bukan sumber insight dashboard-worthy. **Sisi kebenaran** (0.2b): konsistensi STUDENT ALL ↔ STATUS STUDENT **100%** (0 mismatch nama/prodi/semester dari 25.000). **Sisi kesegaran** (7.1–7.3): **87,7% mahasiswa punya `sync_date` >90 hari** (median umur 369 hari, maks 730) — artinya "usang" adalah **norma di seluruh populasi**, bukan anomali segelintir. Jadi 7.x akan menghasilkan "hampir semua data lama secara seragam", bukan pola yang bisa ditindaklanjuti per-segmen. **Kesimpulan: sajikan BT-08 sebagai satu paragraf di laporan** (data konsisten tapi jarang di-refresh → rekomendasi SOP sync berkala), jangan alokasikan panel dashboard untuknya.

**7.1 Distribusi umur data sync**

- **Tujuan:** kemutakhiran data status.
- **Data:** `sync_date`.
- **Cara hitung:** `umur = maks tanggal data − sync_date`; histogram.
- **Output:** histogram. **Terverifikasi:** median 369 hari, 87,7% >90 hari — konfirmasi bahwa sync jarang dilakukan menyeluruh; angka ini yang masuk paragraf laporan.

**7.2 Mahasiswa data "usang"**

- **Tujuan:** status mungkin ketinggalan realita.
- **Data:** `status_student` (`sync_date`, `NIM`) + `tracking_student` (`last_update` per NIM).
- **Cara hitung:** banding `sync_date` vs `last_update` terbaru; tandai jika sync jauh lebih lama (>90 hari).
- **Output:** daftar/jumlah usang; kandidat refresh CDC.

**7.3 Ringkasan segar vs usang**

- **Tujuan:** metrik kesehatan data.
- **Cara hitung:** threshold ≤90 hari = segar; % segar vs usang.
- **Output:** angka + donut; cukup besar untuk dashboard atau ke laporan saja?

---

## Bagian 8 — Sintesis Naratif & Pemetaan Dashboard `[+]` (penentu arah)

**8.1 Kumpulkan temuan terkuat**

- **Tujuan:** menyaring puluhan chart jadi cerita.
- **Cara hitung:** daftar 3–4 temuan paling menonjol/mengejutkan per calon halaman (H1–H4).
- **Output:** daftar bernarasi singkat.

**8.2 Tag tiga dimensi tiap temuan**

- **Tujuan:** jaga dashboard selektif (semua chart masuk = berantakan, skor desain turun).
- **Cara hitung:** beri label tiap temuan:
- **Kepentingan:** "headline KPI" (tampil di dashboard) vs "diagnostik" (cukup di laporan).
- **Interaktivitas** _(bobot 20% penilaian)_: kandidat jadi **filter** (mis. per prodi/periode), **drill-down** (mis. company → daftar kandidat), atau **hover detail**.
- **Halaman tujuan:** H1/H2/H3/H4.
- **Output:** tabel temuan × 3 label. Ini jembatan EDA → desain 4 halaman.

**8.3 Funnel end-to-end sebagai hero visual `[+]`**

- **Tujuan:** satu visual utama menceritakan seluruh sistem dalam satu tarikan napas.
- **Cara hitung:** sambung Bagian 2 + Bagian 4 jadi satu funnel: **25.000 terdaftar → eligible (Active+CV = 16.732) → benar-benar masuk proses (10.174) → pernah placement (5.759 / 56,6% dari partisipan)**.
  - **⚠️ Gerbang kelayakan hero funnel = `Active+CV` saja, BUKAN `Active+CV+Available`.** Alasan (terverifikasi): `ketersediaan` adalah **snapshot terkini**, sehingga `Active+CV+Available` hanya **5.687** — lebih kecil dari jumlah yang benar-benar masuk proses (10.174), membuat funnel **turun→naik→turun** (patah, tidak valid sebagai funnel). Ini karena banyak mahasiswa yang **dulunya Available saat dikirim** kini berstatus Placed/Tidak Aktif (persis nuansa 2.3). Memakai `Active+CV` (16.732) menjaga urutan monoton: 25.000 → 16.732 → 10.174 → 5.759. ✓ **Kolom `Available` tetap dipakai di funnel kelayakan 2.1** (konteks snapshot pool, valid di situ), tapi tidak di hero funnel historis.
  - **Bukan sekadar monoton — ini funnel bersarang sejati (terverifikasi):** audit BT-06 (2.3) membuktikan 100% dari 10.174 partisipan berstatus Active & punya CV, jadi `10.174 ⊆ 16.732` benar-benar himpunan bagian, bukan kebetulan angkanya menurun. Tiap tahap adalah subset ketat tahap sebelumnya — funnel valid secara logika, aman dari pertanyaan "kenapa tahap X bisa lebih kecil dari Y".
- **Output:** funnel besar untuk H1 atau pembuka dashboard; angka sudah terverifikasi, tinggal dirangkai.

---

### Pemetaan 4 halaman dashboard

- **H1 — Overview & Kelayakan:** profil dasar + funnel kelayakan + audit BT-06 + funnel end-to-end (hero).
- **H2 — Matching & Geografi:** mismatch supply-demand + kualitas matching + peta geografis.
- **H3 — Keberhasilan & Tren:** placement rate benar (basis `final_outcome`) + per-student lens (persistence) + multivariat + velocity + tren waktu.
- **H4 — Risiko Operasional:** klasifikasi `final_outcome` (Ghosting = outcome terbesar) + breakdown lokasi/waktu ghosting + beban proses + prioritas request.
