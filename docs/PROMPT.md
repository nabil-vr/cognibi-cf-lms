# Prompt: Perbaikan menyeluruh Adapt CF-LMS (end to end)

Berkas ini adalah instruksi kerja lengkap untuk agen AI atau pengembang yang melanjutkan Adapt CF-LMS. Baca seluruhnya sebelum mengubah apa pun. Kerjakan fase demi fase, berurutan; jangan lompat ke fase berikutnya sebelum kriteria selesai fase sebelumnya terpenuhi.

---

## 1. Peran kamu

Kamu adalah senior developer untuk Adapt CF-LMS, LMS adaptif berbasis *cognitive friendly* untuk sains SMA (topik sinaps, potensial aksi, pembuluh darah) yang juga menjadi wadah penelitian kuasi-eksperimen (kelas eksperimen dan kontrol, pre-test, post-test, kuesioner Leppink, N-Gain).

Tugasmu: merapikan tampilan dan **menyambungkan empat peran (siswa, guru, peneliti, admin)** supaya alur dari kelas sampai data penelitian berjalan utuh, tanpa merusak fitur yang sudah ada.

## 2. Aturan kerja (wajib)

1. **Pahami dulu, ubah kemudian.** Sebelum menyentuh satu fase, baca bagian kode yang relevan dan tulis rencana singkat di pesan commit atau catatan fase.
2. **Satu fase = satu rangkaian commit kecil** yang masing-masing bisa dijelaskan dalam satu kalimat. Pesan commit dalam bahasa Indonesia, kalimat biasa (contoh: `Sembunyikan catatan ATM dari tampilan siswa`).
3. **Jangan hapus fitur.** Kalau sebuah fitur diganti, pertahankan perilaku lama sampai pengganti terbukti jalan.
4. **Bahasa antarmuka:** Indonesia sebagai utama, sapaan siswa "kamu", guru/peneliti/admin "Anda". Setiap teks baru wajib punya padanan EN lewat `I18N` (kunci per pola, bukan string utuh).
5. **Aksesibilitas:** target sentuh minimal 24 px (44 px di HP), kontras AA, semua kontrol bisa dipakai dengan papan ketik, status tidak hanya dibedakan warna.
6. **Data contoh selalu diberi label "contoh".** Angka yang tidak berasal dari data aplikasi tidak boleh tampil seolah nyata.
7. **Verifikasi setiap fase** dengan Playwright (lihat bagian 9): 0 error konsol, tangkapan layar desktop 1280x860 dan HP 390x844 untuk halaman yang diubah, terang dan gelap.
8. **Jangan klaim selesai** sebelum kriteria fase terpenuhi dan diperiksa.

## 3. Sumber kebenaran (Fase 0 menentukan)

- Versi terbaru aplikasi ada di **artifact** `https://claude.ai/artifact/8X3qHvYMfyViSEg8oxMJew` (±37.000 baris, 5,6 MB).
- `index.html` di repo (±23.000 baris) **tertinggal**: tidak punya Studi peneliti, HUB lintas peran, admin versi penuh, kode/QR penelitian, Tutor AI adaptif, dan mode aman ujian.
- Artifact dibangun dari modul (`build.py`, `app_peneliti.js`, `app_lain.js`, dst.) yang **tidak ada di repo**.

Keputusan default bila pemilik tidak menjawab: **artifact menjadi baseline**. Simpan sebagai `index.html` di repo, lalu kerjakan semua perubahan di atasnya.

## 4. Peta kode (patokan, cari dengan nama, bukan nomor baris)

Semua LMS berjalan di shadow root `#app-host`. Pola utamanya:

| Hal | Pengenal di kode |
|---|---|
| Daftar halaman per peran | `reg(role, key, def)` mengisi `P['role:key']`; `def` punya `render`, `act`, `submit`, `dialog`, `input`, `change`, `atm` |
| Navigasi per peran | `NAV`, `NAVMAP`, `ALIAS`, `navBadge`, `railHtml`, `bnavHtml` |
| Persona (identitas tertulis langsung) | `PERSONA`, `PN_ME` (`'Bu Sri Wahyuni'`, `'Dimas Pratama'`), `CLS_OF` |
| State aplikasi | `A` (role, page, p, ui, cls, dlg, preview, asAdm), `SET` (pengaturan pribadi) |
| Kelas, materi, progres | `CLASSREG`, `ROSTER`, `COURSE`, `COURSES`, `PROG`, `PROGBY`, `item()`, `complete()` |
| Pengumpulan dan nilai | `SUBS`, `hub_submit`, `hub_gradeBack`, `gn_recs`, `gn_csv` |
| Penelitian | `PNS` (daftar Studi), `PN` (Studi aktif), `pn_flow` (alur 7 langkah), `pn_todo`, `PN.ins` (instrumen), `PN.slot` (jadwal instrumen), `PN.kond` (kondisi kelas) |
| Persetujuan guru untuk data | `gn_rs` (**satu variabel global**) |
| Gerbang data penelitian | `hub_izin` (izin admin), `gn_rs` (guru), `r.izin` per siswa (wali), `S.eks` (ekspor disetujui admin) |
| Beranda peneliti v2 | `pd_stats`, `pd_recs`, tile di sekitar `tile('Tahap'…)` dan `tile('N-Gain eksperimen'…)` |
| Admin | `adm_*`, `ACC`, `ADM_OFF`, `adm_AK` (matriks akses, saat ini hanya tampilan) |
| Log dan notifikasi | `hub_log` (`HUB_LOG`), `nlog` (`NLOG`, **per peran, bukan per orang**) |
| Catatan rancangan | `atmHtml(d.atm)` dipanggil di `render()` untuk semua peran |
| Pintu masuk aplikasi | `window.__APP.open({ role, page, p })`, `SH.show('ob' / 'lp')`, `SH.AUTH.open('masuk' / 'daftar')` |
| Bilah pratinjau | elemen `.pv-*`, `[data-pv]`, `sessionStorage 'pv-off'` |

Pola tambahan fitur yang dipakai sekarang adalah **membungkus fungsi lama** (`var x0 = fn; fn = function(){ … x0.apply(…) }`). Pola ini rapuh bila bertumpuk. Fase 1 menggantinya secara bertahap.

## 5. Kerangka hubungan antarperan (target)

```
                       ┌─────────────── ADMIN ───────────────┐
                       │ setujui akun · izin studi · ekspor  │
                       │ fitur · kelas · log audit           │
                       └──┬──────────────┬────────────────┬──┘
                          ▼              ▼                ▼
   SISWA ◄──── KELAS ──── GURU ◄═══ STUDI ═══► PENELITI
     │   materi, nilai,    │    kondisi, instrumen,    ▲
     │   diskusi, tim      │    jadwal, chat           │
     └── respons instrumen ┴──► DATA (anonim) ─────────┘
            ▲
          WALI
```

Objek inti dan pemiliknya:

| Objek | Pemilik | Yang terlibat |
|---|---|---|
| Sekolah | Admin platform | Admin sekolah |
| Kelas | Guru | Siswa (lewat kode), admin sekolah |
| Studi | Guru **atau** peneliti | Anggota guru, peneliti, kelas tertaut + kondisinya |
| Instrumen | Studi | Dibuat peneliti/guru, disetujui pihak lain, dijadwalkan ke pertemuan kelas |
| Respons | Siswa | Dibaca guru (dengan nama, kelasnya saja), peneliti (anonim, bila 5 gerbang terbuka) |

**Lima gerbang data penelitian** (semuanya harus terbuka sebelum peneliti melihat atau mengunduh data seorang siswa):
1. Izin penelitian dari admin (`hub_izin`).
2. Persetujuan guru **per Studi** (menggantikan `gn_rs` global).
3. Izin wali per siswa (`r.izin === 'ok'`).
4. Persetujuan siswa sendiri (`SET.research`, saat ini belum dipakai di ekspor).
5. Permintaan ekspor disetujui admin (`S.eks[].st === 'disetujui'`).

---

## 6. Fase kerja

### Fase 0: Fondasi dan baseline

**Tugas**
1. Ambil artifact versi terbaru, simpan sebagai `index.html` di repo. Commit terpisah: `Jadikan artifact terbaru sebagai baseline`.
2. Jalankan pemeriksaan dasar (bagian 9) pada baseline dan simpan hasilnya sebagai pembanding.
3. Buat lapisan identitas: `ME = { id, name, role, schoolId }` dari sesi (`AUTH`/`APP.open`). Ganti pemakaian `PN_ME`, `'Bu Sri'`, `'Dimas'` yang tertulis langsung dengan `ME` atau data pengguna.
4. Ubah `nlog(role, …)` menjadi `nlog(target, …)` dengan `target` berupa id pengguna atau peran (peran tetap didukung supaya pemanggil lama tidak rusak).

**Selesai bila:** baseline jalan tanpa error, tidak ada nama orang yang tertulis langsung di logika (data contoh boleh), notifikasi bisa ditujukan ke satu orang.

### Fase 1: Perbaikan cepat (visual dan bug)

| # | Masalah | Lokasi | Perbaikan |
|---|---|---|---|
| 1.1 | Chip "Pratinjau" menimpa ikon Pengaturan/Bantuan di rel desktop dan menutup isi di HP | bilah `.pv-*` | Pindahkan ke pojok yang tidak dipakai (kanan atas di desktop, di atas navigasi bawah di HP tanpa menimpa FAB), atau beri ruang di rel |
| 1.2 | Catatan rancangan (ATM) tampil ke siswa | `atmHtml(d.atm)` di `render()` | Tampilkan hanya bila mode pratinjau aktif (`data-pv` bukan `off`) dan peran bukan siswa |
| 1.3 | Tag kondisi peneliti terbaca "E k s p e r i m e n" | CSS `.pn-kt { letter-spacing: .12em; font-size: 28px }` diwarisi `.tag` | Batasi gaya ke elemen yang memang besar, atau atur `.pn-kt .tag { letter-spacing: 0; font-size: inherit dari .tag }` |
| 1.4 | Angka peneliti saling bertentangan: Beranda "post-test 100%, 17/17", Studi "23 dari 32", Hasil "Responden 23" | `pd_stats` vs `PN.kelas[].post` statis vs `pn_calc` | Satu fungsi sumber: `studyStats(S)` dipakai ketiga halaman. Tulis penyebutnya ("dari siswa berizin") di setiap angka |
| 1.5 | Dua pelacak 7 langkah dengan angka berbeda ("Protokol 3 dari 7" di Beranda, "Alur kerja sama langkah 2 dari 7" di Studi) | `pd_*` checklist vs `pn_flow` | Pakai `pn_flow` saja; Beranda menampilkan versi ringkasnya |
| 1.6 | Beranda peneliti menampilkan "N-Gain kontrol 0,29" padahal kelas kontrol masih "Data menunggu guru" | tile N-Gain di `pd_*` | Hitung hanya dari kelas yang lolos gerbang 1 sampai 4; kelas lain tampil sebagai kunci dengan alasannya |
| 1.7 | Placeholder cari "Cari materi atau tugas" untuk semua peran; admin punya dua kotak cari | topbar + `adm_searchCard` | Placeholder dan cakupan cari mengikuti peran; admin cukup satu kotak (topbar memakai mesin `adm_searchCard`) |
| 1.8 | Tab meluap di HP tanpa tanda bisa digeser (ruang penelitian, ruang kelas) | `.tabs` | Gradasi atau bayangan di tepi, tab aktif digulir ke tampilan |
| 1.9 | Temuan lama `docs/ANALISIS.md` #3, #6, #7, #8, #10 | lihat berkas | Kerjakan sesuai saran di sana |

**Selesai bila:** setiap butir punya tangkapan layar sebelum dan sesudah, dan angka di Beranda, Studi, dan Hasil peneliti sama untuk Studi yang sama.

### Fase 2: Penataan per peran

#### Siswa
- Beranda desktop: ganti balok hitam tinggi "31% Progres" dengan bar mendatar seperti versi HP.
- Satu sumber "langkah berikutnya" untuk kartu Lanjutkan belajar, Jadwal hari ini, dan Perlu dikerjakan, supaya tidak memberi dua arahan berbeda.
- Kartu "Penelitian di kelasmu" di Pengaturan > Privasi dan di halaman kelas: status ikut/tidak, izin wali, dan daftar aktivitas yang termasuk instrumen penelitian.
- Instrumen penelitian tampil sebagai aktivitas biasa dengan penanda "Kuesioner penelitian · tidak dinilai".

#### Guru
- Kotak masuk **Permintaan** (satu tempat): usulan kondisi dan instrumen dari peneliti, permintaan ubah fitur, laporan pesan siswa, perubahan kelas dari admin. Lencana di rel menghitung semuanya.
- Kartu kelas di Beranda menampilkan penelitian yang menempel (judul Studi + tahap), bukan hanya label Eksperimen/Kontrol.
- Halaman padat (editor, pustaka, LKPD, nilai): pengaturan lanjutan dibuka bertahap (`details` atau panel samping), bukan semuanya sekaligus.

#### Peneliti
- Beranda menjawab tiga pertanyaan: giliran siapa sekarang (`pn_flow`), data mana yang sudah boleh dilihat (lima gerbang per kelas), dan apa yang menghambat.
- Data yang belum boleh dilihat tampil sebagai kunci dengan alasan tertulis, tidak pernah sebagai angka.
- Placeholder cari "Cari penelitian, instrumen, atau kelas".

#### Admin
- Beranda dirampingkan menjadi **Perlu tindakan** + **Kesehatan sistem**; grafik, adopsi fitur, penyimpanan, dan pengumuman pindah ke halaman masing-masing atau di balik tautan.
- Hapus tile "Aktivitas sesi ini".
- Rel 10 ikon: beri label (rel bisa dilebarkan) atau tooltip; kelompokkan Pengelolaan dan Sistem.

#### Semua peran
- Penanda peran yang halus di topbar (chip peran atau aksen di logo), terutama saat "Masuk sebagai" dan "Lihat sebagai siswa".
- Sapaan konsisten (siswa "kamu", lainnya "Anda") juga di pesan sistem dan toast.

**Selesai bila:** tangkapan layar desktop dan HP untuk Beranda keempat peran, terang dan gelap, sudah diperiksa; tidak ada teks EN yang hilang untuk teks baru.

### Fase 3: Kontrak peran dan izin

1. Ubah `adm_AK` dari tabel tampilan menjadi aturan yang dijalankan: `can(user, action, resource)`. Halaman dan tombol memeriksa `can()`, bukan `A.role === '…'` yang tersebar.
2. Matriks di halaman Data dan akses dibaca dari aturan yang sama, jadi tampilan dan perilaku tidak bisa berbeda.
3. Jalur masuk admin: lewat `AUTH` dengan kode akses admin atau undangan, tidak lewat pilihan peran di onboarding publik.
4. Pisahkan **admin platform** (semua sekolah, sakelar fitur global, backup, versi) dan **admin sekolah** (akun, kelas, izin penelitian, ekspor untuk sekolahnya). Sakelar fitur bisa per sekolah.
5. Notifikasi dan log memakai id pengguna (dari Fase 0).

**Selesai bila:** mencoba aksi terlarang dari peran yang salah (mis. peneliti membuka data dengan nama) ditolak di logika, bukan hanya disembunyikan dari menu.

### Fase 4: Menyambung peneliti ke siswa (inti penelitian)

Ini satu-satunya alur yang sekarang benar-benar putus: kode siswa tidak membaca `PN.ins` sama sekali, pre-test/post-test/Leppink siswa ditulis tetap di `COURSE`, dan halaman Hasil memakai angka acak berbenih (`pn_rng`, `pn_calc`).

1. **Instrumen jadi aktivitas.** Setiap `PN.slot` yang sudah punya pertemuan (`mt`) dan instrumennya disetujui (`st === 'setuju'`) dibuat menjadi `item()` di kursus kelas yang tertaut, sesuai kondisi (`u`: eksperimen, kontrol, atau kedua). Posisi mengikuti `w` (sebelum bab, di dalam unit, sesudah bab, tertunda).
2. **Pemutar instrumen** di sisi siswa untuk semua tipe butir `PN_TIPE` (likert, pg, pilihan, uraian, slider, angka, urut, jodoh, reaksi, media, observasi, unggah). Pakai ulang komponen kuis/kuesioner yang ada.
3. **Respons tersimpan** per siswa per instrumen per versi (`ins.v`). Instrumen yang sudah dibekukan (`beku`) tidak boleh berubah setelah ada respons.
4. **Hasil dan Ekspor membaca respons nyata.** `pn_calc` hanya dipakai bila belum ada respons, dan hasilnya diberi label "contoh". Kamus data tetap dibentuk dari instrumen (`pn_vars`).
5. **Persetujuan guru per Studi:** `S.guruSetuju = { st, at }` menggantikan `gn_rs` global. Halaman Riset guru menampilkan satu kartu per Studi.
6. **Gerbang ke-4:** siswa yang mematikan "Ikut penelitian" dikecualikan dari `gn_csv` anonim dan `hub_csv`, sama seperti wali yang menolak, dan jumlah yang dikecualikan ditulis di ekspor.
7. Kondisi kelas tetap satu sumber (`hub_syncKond`); kelas kontrol tidak menerima instrumen atau media yang hanya untuk eksperimen.

**Selesai bila (uji ujung ke ujung):**
peneliti membuat instrumen → guru menyetujui → guru menjadwalkan ke pertemuan → admin memberi izin → siswa di kelas eksperimen melihat dan mengerjakan → guru melihat status dengan nama → peneliti melihat hasil anonim dari respons nyata → siswa yang opt-out atau walinya menolak tidak muncul di ekspor.

### Fase 5: Etika dan tata kelola

1. **Alur wali:** tautan persetujuan per siswa (tanpa akun wali di tahap ini), status tersimpan (`ok`, `tunggu`, `tolak`) dan bisa dicabut. Ganti tombol demo `sa-parent`.
2. **"Masuk sebagai" dibatasi:** alasan wajib, banner mencolok selama sesi, baca saja secara bawaan, durasi terbatas, pengguna yang bersangkutan diberi tahu, tercatat di log.
3. **Perubahan tahap Studi oleh admin** wajib alasan dan memberi tahu seluruh anggota Studi.
4. **Deviasi protokol** (admin mematikan fitur yang dipakai kelas eksperimen) tetap tercatat di log Studi seperti sekarang, ditambah notifikasi ke peneliti dan guru.

### Fase 6: Menuju produksi

1. Pecah berkas tunggal menjadi modul; muat LMS secara malas setelah masuk (ANALISIS #11).
2. Backend: login Google (OAuth), basis data untuk objek di bagian 5, API, penyimpanan berkas.
3. Ekspor dan unduhan dari server (ANALISIS #1), log audit permanen.
4. SRI atau hosting sendiri untuk three.js dan font (ANALISIS #2).
5. Lengkapi i18n EN (ANALISIS #4).

---

## 7. Hal yang tidak boleh rusak

- Onboarding → landing 3D → masuk/daftar → LMS untuk tiap peran.
- Pemutar langkah siswa, kuis dengan petunjuk, pre/post/ulangan dengan mode aman, tugas dan rubrik, lab 3D dan fallback 2D.
- Editor blok guru, pembuat kuis/tugas, penilaian, buku nilai, N-Gain, analisis butir, kelompok, diskusi, kalender.
- Alur kerja sama guru dan peneliti, persetujuan admin, log audit.
- Mode terang/gelap, mode Tenang (tanpa animasi), ukuran huruf, ID/EN.

## 8. Keputusan yang perlu dari pemilik (pakai default bila belum dijawab)

| Pertanyaan | Default |
|---|---|
| Sumber kebenaran: artifact atau modul + `build.py`? | Artifact |
| Tetap prototipe satu berkas atau mulai backend? | Prototipe sampai Fase 5 selesai, backend di Fase 6 |
| Admin: satu untuk semua atau admin sekolah + admin platform? | Dua tingkat (Fase 3) |
| Wali: peran dengan akun atau tautan persetujuan saja? | Tautan saja dulu |
| Stack backend | Ditanyakan saat masuk Fase 6 |

## 9. Cara memeriksa

1. Buka berkas lewat Playwright (Chromium di `/opt/pw-browsers` bila ada), sembunyikan bilah pratinjau dengan `sessionStorage.setItem('pv-off','1')`.
2. Untuk tiap peran dan halaman yang diubah: `window.__APP.open({ role, page, p })`, tunggu 500 ms, ambil tangkapan layar desktop 1280x860 dan HP 390x844, terang dan gelap.
3. Catat semua `pageerror` dan error konsol; targetnya 0.
4. Telusuri semua tautan `data-go` per peran dan klik setiap tombol (seperti metode di `docs/ANALISIS.md`); pastikan tidak ada teks `undefined`, `NaN`, `[object`, dan tidak ada gulir horizontal.
5. Untuk Fase 4, jalankan uji ujung ke ujung yang ditulis di kriteria selesai.

## 10. Laporan per fase

Setiap fase ditutup dengan laporan singkat:
- Apa yang berubah (daftar commit).
- Tangkapan layar sebelum dan sesudah.
- Hasil pemeriksaan (error, halaman yang ditelusuri).
- Yang sengaja belum dikerjakan dan alasannya.
- Keputusan yang masih ditunggu dari pemilik.
