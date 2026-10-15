# Analisis desain dan bug: Adapt CF-LMS (index.html)

Metode: Chromium headless (Playwright), desktop 1280x800 dan HP 390x844, terang dan gelap. Semua halaman dijelajah otomatis lewat tautan `data-go` (siswa 18, guru 49, peneliti 10, admin 9 halaman), lalu setiap tombol/ringkasan/pilihan di tiap halaman diklik (termasuk tombol di dalam dialog), dan mode EN diperiksa. Three.js diambil dari npm karena CDN diblokir di sandbox.

## Desain
- Satu berkas ±2,2 MB: pratinjau alur (bilah kuning) → onboarding (shadow root `#ob-host`) → landing 3D → masuk/daftar → LMS (shadow root `#app-host`).
- Token: stone netral, aksen teal `#0F766E` (gelap `#2DD4BF`), tombol hitam/putih, sudut membulat, kaca (glass) hanya pada lapisan melayang. Font judul Bricolage Grotesque (alternatif Fraunces/Schibsted), isi Inter, label IBM Plex Mono.
- LMS: rel ikon kiri (navigasi bawah di HP), kotak cari, sapaan besar + chip tanggal, Kelas → Unit → Pelajaran → Langkah (satu langkah satu layar), Tutor AI sebagai FAB, tema terang/gelap dan ID/EN di mana-mana.
- Menu per peran: siswa (Hari Ini, Kelas Saya, Nilai, Tampilan Saya), guru (Dasbor, Kelas & Pelajaran, Penilaian, Analitik), peneliti (data, N-Gain, Leppink), admin.

## Hasil uji fungsi
- Sintaks JS valid; **0 error runtime** (pageerror/console) di seluruh penjelajahan dan fuzz klik, desktop.
- Tidak ada teks rusak (`undefined`, `NaN`, `[object`), tidak ada halaman kosong, tidak ada overflow horizontal di desktop maupun HP.
- Onboarding 3 peran, landing, dan masuk/daftar berjalan tanpa error. Fallback bila three.js gagal dimuat bekerja (osiloskop 2D tampil).
- Penghitung "Langkah 1 dari 5" dengan 6 batang progres BUKAN bug (batang ke-6 = ringkasan).

## Temuan (urut prioritas)
1. **Tombol Ekspor (guru/peneliti) memakai unduhan berkas.** Di viewer artifact unduhan diblokir, jadi tidak berfungsi di sana. Di peramban biasa tidak masalah. Sebelum produksi: pakai unduhan sisi server.
2. **Ketergantungan CDN tanpa SRI** (three.js cdnjs/jsdelivr/unpkg, Google Fonts). Tambahkan `integrity`, atau hosting sendiri; tanpa internet font jatuh ke sistem dan 3D ke fallback 2D.
3. **Fallback tanpa 3D:** kotak keterangan di bawah osiloskop hero kosong (hanya batang abu-abu), instruksi "Tekan Rangsang kuat…" hilang.
4. **i18n EN belum lengkap** untuk teks dinamis/gabungan: "Jam tenang … Anda bisa mengubahnya di" (notif, semua peran), "Komentar untuk <nama>", "Pembuat ulangan · Unit 8 · 5 soal", "Istilah yang akan kamu temui…", isi editor guru, rubrik tugas, prompt tutor. Perbaiki dengan kunci terjemahan per pola, bukan kamus string utuh.
5. **Sapaan tidak konsisten:** siswa "kamu", guru/peneliti/admin "Anda" (disengaja?), tetapi pesan sistem bercampur.
6. **FAB Tutor AI di HP** memakai ikon "spark" yang terlihat seperti spinner pemuatan, dan menimpa tombol "Selesai" di baris daftar paling kanan. Beri ruang bawah (`padding-bottom`) atau ganti ikon.
7. **Topbar kaca di HP:** latar tembus pandang sehingga teks konten terlihat di bawah/atasnya saat digulir (terutama di sekitar padding 18px). Naikkan opasitas latar.
8. **Target sentuh kecil:** judul item terkunci di "Setelah ini" tinggi 18-20 px; pegangan seret "Seret untuk memindah" 22x36; beberapa tombol ikon 20x20. Target minimal 24 px (WCAG 2.2), idealnya 44 px di HP.
9. **Bilah pratinjau menyita 120 px** di HP (4 baris). Wajar untuk prototipe, tetapi jadikan ringkas/dapat dilipat.
10. **Kebersihan HTML:** `<title>`, `<link>` font ada di dalam `<body>` (setelah `</head>`); pindahkan ke head. `#app-host` dan `#ob-host` memakai `window.onresize` dan `scrollTo` global; pakai `addEventListener`.
11. **Berkas 2,2 MB satu halaman** (±1,77 MB JS, kamus EN besar): pecah jadi modul, muat LMS secara malas setelah masuk.
12. **Admin tidak punya jalur masuk** dari onboarding/landing (hanya lewat tombol pratinjau).

## Belum diuji
Gambar 3D secara visual di GPU asli, kinerja animasi, pembaca layar sungguhan, kontras warna menyeluruh, perilaku di Safari/Firefox, data nyata (semua data masih contoh).
