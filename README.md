# PulihNapas

Sistem IoT untuk mengukur dan memprediksi **waktu pulih napas** setelah aktivitas ringan. Memakai 2 HP sebagai edge device, MQTT sebagai komunikasi ke cloud, dan laptop sebagai dashboard.

> Bukan alat medis. Lakukan aktivitas ringan saja dan berhenti jika pusing.

> Catatan: proyek ini dikembangkan dengan bantuan Claude (AI). Kelompok menguji, memahami, dan menjalankan seluruh sistem.

## File

| File | Fungsi |
|---|---|
| `phone.html` | Dibuka di HP (`?id=hp1` atau `?id=hp2`). Membaca sensor, menghitung repetisi dan napas, mengirim ke MQTT |
| `dashboard.html` | Dibuka di laptop. Mengirim perintah, menampilkan grafik, baseline, waktu pulih, dan prediksi AI |
| `index.html` | Halaman tautan ke tiga halaman di atas |

## Memenuhi ketentuan tugas

- **2 edge device:** HP 1 dan HP 2.
- **2 sensor:** akselerometer dan giroskop.
- **Pemrosesan data (lebih dari 2 jenis):** hitung repetisi dan napas per menit (di HP), baseline dan waktu pulih serta prediksi AI (di dashboard).
- **Algoritma / AI:** deteksi puncak, penggabungan dua sensor, dan regresi linear untuk prediksi.
- **Cloud:** MQTT lewat broker publik HiveMQ.
- **Dua arah:** dashboard mengirim perintah, HP mengirim data.

## Alur kerja

1. **Baseline:** rebahan, HP di perut, diam. Napas normal dicatat.
2. **Aktivitas:** HP di saku atau dipegang, lompat atau squat. Repetisi dihitung.
3. **Pemulihan:** rebahan lagi. Dashboard memantau napas, memprediksi kapan pulih, lalu mencatat waktu pulih sebenarnya.

Pulih = napas ≤ baseline × 1,15 sebanyak 2 laporan berturut-turut.

## Topik MQTT

Ganti `ROOM` dengan kode ruang kelompok (di `phone.html` dan `dashboard.html`, harus sama).

| Topik | Arah | Isi |
|---|---|---|
| `ROOM/cmd` | dashboard → HP | `{"cmd":"baseline" / "activity" / "recovery" / "stop","t":waktu}` |
| `ROOM/dev/hpX/reps` | HP → dashboard | `{"reps":12,"t":waktu}` |
| `ROOM/dev/hpX/resp` | HP → dashboard | `{"bpm":24,"phase":"recovery","t":waktu}` |
| `ROOM/dev/hpX/ack` | HP → dashboard | `{"cmd":"recovery","t":waktu}` (untuk hitung latensi) |

Broker publik tidak punya login, jadi jangan kirim data pribadi.

## Cara pakai

1. Upload file ke GitHub, aktifkan GitHub Pages (Settings → Pages → branch `main`).
2. HP 1: buka `.../phone.html?id=hp1`, tekan **Mulai sensor**. HP 2: sama dengan `?id=hp2`.
3. Laptop: buka `dashboard.html`, lalu tekan Baseline → Aktivitas → Pemulihan.

## Algoritma singkat

- **Repetisi:** besar percepatan dikurangi gravitasi (9,8). Kalau selisihnya melewati ambang dan sudah lewat jeda minimal, dihitung 1 repetisi.
- **Napas per menit:** sinyal dicatat tiap 0,1 detik (30 detik terakhir), gerakan pelan dibuang, dihaluskan, lalu puncak dihitung. Satu puncak = satu tarikan napas. Dilakukan untuk akselerometer dan giroskop. Kalau hasilnya selisih ≤ 3, diambil rata-rata; kalau tidak, dipercaya akselerometer.
- **Waktu pulih:** baseline = nilai tengah napas saat fase baseline. Dibandingkan dengan batas baseline × 1,15.
- **Prediksi AI:** napas yang pulih turun seperti kurva `baseline + A·e^(−t/τ)`. Dengan logaritma, kurva ini menjadi garis lurus, lalu dicari dengan regresi linear. Garis diperpanjang sampai menyentuh batas pulih untuk mendapat perkiraan sisa waktu. Prediksi pertama setelah detik ke-30 dibandingkan dengan waktu pulih sebenarnya. Galat = prediksi − aktual.

Regresi linear termasuk machine learning sederhana. Model belajar dari data satu sesi tiap orang, bukan model besar.

## Pengujian

1. **Akurasi napas:** hitung napas sendiri 30 detik, bandingkan dengan sistem, 5 kali.
2. **Akurasi repetisi:** lakukan 20 gerakan, bandingkan dengan hitungan sistem.
3. **Latensi:** rata-rata waktu pulang-pergi dibagi 2, tampil di dashboard.
4. **Dua HP bersamaan:** jalankan keduanya, bandingkan grafik.
5. **Akurasi prediksi AI:** ulangi siklus 3-5 kali, catat "Galat prediksi" tiap sesi.

## Keterbatasan

- Hasil napas bergantung pada posisi HP dan diam saat mengukur.
- Ambang repetisi (`AMBANG_REP`, `JEDA_REP`) perlu disetel sesuai HP dan gerakan saat pengujian.
- Broker publik tidak aman untuk data pribadi.
