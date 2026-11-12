# Quackville: Story Quests (multiplayer)

Game belajar narrative text + grammar. Satu kelas dibagi beberapa kelompok (maks 6 orang per kelompok).
Tiap kelompok berada di desa yang sama, bisa saling lihat, dan bisa mencuri koin (setelah menjawab soal kilat).
Ranking koin dihitung untuk seluruh kelas.

## 1. Siapkan Firebase (gratis, sekitar 5 menit)
1. Buka https://console.firebase.google.com, buat project baru.
2. Build > **Realtime Database** > Create database (pilih lokasi terdekat, mis. Singapore). Mulai dengan *locked mode*.
3. Tab **Rules**: tempel isi `database.rules.json`, lalu Publish.
4. Project settings (ikon roda gigi) > **Your apps** > tambah Web app (`</>`). Salin objek `firebaseConfig`.
5. Isi `config.js` dengan nilai itu. **`databaseURL` wajib diisi** (ada di tab Realtime Database).

## 2. Deploy
- Push folder ini ke GitHub, lalu di **Netlify** atau **Vercel**: *Add new project* > pilih repo > deploy (tanpa build command, folder root).
- Bagikan link ke siswa.

## 3. Cara memakai di kelas
- Guru membuat **kode kelas** (contoh `XIA2025`) dan membagi siswa ke Grup 1, 2, 3, dst.
- Siswa: buka link > PLAY > buat bebek dan nama > tombol 👥 di menu awal > masukkan kode kelas > Join Class. Sistem otomatis membagi siswa ke kelompok yang masih memiliki tempat (maksimal 5 siswa per kelompok); siswa tidak dapat memilih kelompok sendiri.
- Tombol 🏆 di layar permainan menampilkan ranking kelas.
- Mencuri: tap bebek teman yang dekat, jawab soal kilat. Berhasil = ambil sekitar 10% koin korban (5 sampai 50).
  Gagal = kehilangan 3 koin. Jeda 30 detik antar percobaan. Korban dapat perisai 60 detik, dan koin di bawah 20 tidak bisa dicuri.
  Hanya pemain yang sedang online dan tidak di dalam rumah yang bisa dicuri.

## Fitur tambahan
- **Grup otomatis:** siswa cukup memasukkan kode kelas lalu menekan Join Class. Game menetapkan grup secara otomatis; siswa tidak dapat memilih nomor grup sendiri. Batas 5 siswa per grup.
- **Emote dan frasa cepat:** tombol 😀 di layar permainan. Pilihannya tetap (emoji dan 4 frasa), jadi tidak ada chat bebas.
- **Halaman guru:** buka `teacher.html` di alamat situs yang sama (contoh `https://situsmu.netlify.app/teacher.html`), masukkan kode kelas. Terlihat siapa online di tiap grup, ranking koin, tombol pengumuman ke semua pemain, progres Story Quest, dan unduh CSV.
  Halaman ini tidak dikunci, jadi jangan bagikan alamatnya ke siswa.
- **Story Quest:** empat chapter harus diselesaikan secara berurutan. Nilai lulus minimal 4/5. Setiap chapter memiliki 3 percobaan; jika belum lulus setelah 3 percobaan, status menjadi *Needs Teacher Review*.
- **Teacher Control Center:** pada halaman guru, bagian *Story Quest Progress* menampilkan status tiap chapter. Tombol **Reset attempts** mengembalikan hitungan percobaan chapter ke 0, sedangkan **+1 attempt** memberi satu kesempatan tambahan. Perubahan akan tersinkron ke game siswa saat mereka online.
- **Game ending:** setelah semua 4 chapter lulus, siswa mendapatkan gelar *Story Master*. Mereka kemudian dapat kembali ke desa dan melanjutkan eksplorasi, mengumpulkan koin, serta mendekorasi rumah.
- **Visit a Friend's House:** semua siswa dalam kode kelas yang sama dapat melihat rumah teman yang muncul sebagai ikon 🏡 di peta. Dekorasi rumah dan taman disimpan ke Firebase agar dapat dilihat teman. Dekati ikon rumah teman lalu tekan tombol kunjungan; mode tamu hanya untuk melihat, tidak bisa mengubah dekorasi.
- **House Photo:** saat berkunjung, tekan **📸 Take Photo** untuk menyimpan gambar interior rumah teman sebagai file PNG di perangkat.

## Pengaturan
Di `index.html`, cari `MAXG=6` (maksimal pemain per grup). Jumlah koin curian ada di fungsi `stealUI`.

## Catatan
- Aturan database di atas terbuka untuk siapa pun yang tahu kode kelas. Cukup untuk kelas, tapi pakai kode yang tidak mudah ditebak.
- Koin disimpan di browser pemain, jadi bisa diakali lewat developer tools. Cocok untuk kelas, bukan untuk kompetisi serius.
- Paket gratis Firebase membatasi koneksi bersamaan (setahuku sekitar 100). Cek batas terbaru di dokumentasi Firebase.
- Tanpa isi `config.js`, game tetap bisa dimainkan solo.
