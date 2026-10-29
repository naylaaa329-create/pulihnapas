# PulihNapas
Tugas Kelompok 6 IOT - PulihNapas: Uji Pemulihan Napas setelah Aktivitas

Sistem IoT untuk mengukur **waktu pemulihan napas** setelah aktivitas ringan, memakai dua smartphone sebagai edge device, MQTT sebagai komunikasi cloud dua arah, dan laptop sebagai dashboard.

> Bukan alat medis. Lakukan aktivitas ringan saja dan berhenti jika pusing.

## Arsitektur

```
HP 1 (edge)  --\                         /--> perintah (baseline/activity/recovery/stop)
                 >-- MQTT broker publik --<
HP 2 (edge)  --/                         \--> data (reps, napas/menit) --> Dashboard laptop
```

| Komponen | Peran |
|---|---|
| `phone.html?id=hp1` / `?id=hp2` | Edge: baca sensor, hitung repetisi, estimasi napas, kirim ke MQTT, terima perintah |
| `dashboard.html` | Kirim perintah, grafik napas, baseline, napas puncak, waktu pulih, latensi |
| `index.html` | Halaman peluncur (atur kode ruang, buka HP 1/HP 2/dashboard) |

## Pemenuhan ketentuan tugas

- **2 edge device:** dua HP Android/iPhone.
- **2 sensor:** akselerometer dan giroskop (event `devicemotion`).
- **2 jenis pemrosesan:** (1) penghitung repetisi dan estimasi napas per menit di HP, (2) analisis baseline dan waktu pulih di dashboard.
- **Cloud:** MQTT lewat WebSocket ke broker publik.
- **Dua arah:** dashboard mengirim perintah, HP mengirim data dan ack.

## Kontrak data MQTT

Ganti `ROOM` dengan kode ruang unik kelompok (default contoh: `pn-kelD-x7q3`).

| Topik | Arah | Isi (JSON) |
|---|---|---|
| `ROOM/cmd` | dashboard -> HP | `{"cmd":"baseline"\|"activity"\|"recovery"\|"stop","t":<ms>}` |
| `ROOM/dev/hpX/reps` | HP -> dashboard | `{"reps":12,"t":<ms>}` |
| `ROOM/dev/hpX/resp` | HP -> dashboard | `{"bpm":24.0,"amp":0.05,"phase":"recovery","t":<ms>}` (+ `a`, `g` = estimasi per sensor) |
| `ROOM/dev/hpX/ack` | HP -> dashboard | `{"cmd":"recovery","t":<t dari cmd>,"from":"hp1"}` (untuk hitung latensi) |

Broker: `wss://broker.hivemq.com:8884/mqtt` (cadangan otomatis: `wss://broker.emqx.io:8084/mqtt`). Broker publik tanpa login, jangan kirim data pribadi.

## Cara pakai

1. Ganti kode ruang di `phone.html` dan `dashboard.html` (konstanta `ROOM`), atau pakai parameter `?room=...`.
2. Upload ke GitHub, aktifkan Pages (Settings -> Pages -> branch `main`, folder `/root`).
3. HP 1: buka `https://USERNAME.github.io/pulihnapas/phone.html?id=hp1`, tekan **Mulai sensor**.
4. HP 2: sama, dengan `?id=hp2`.
5. Laptop: buka `dashboard.html`. Tanpa HP, centang **Mode simulasi** untuk mencoba dashboard.
6. Urutan demo: Baseline (rebahan 60 detik) -> Aktivitas (lompat/squat 60 detik) -> Pemulihan (rebahan, diam) -> baca waktu pulih.

## Algoritma

**Repetisi (di HP):** magnitudo akselerasi, estimasi gravitasi dengan low-pass, sisa gerakan dihaluskan, lalu deteksi puncak lokal di atas ambang (`REP_THRESH`, default 2,5 m/s2) dengan jeda minimal 0,4 detik.

**Napas per menit (di HP):** sinyal diratakan ke 10 Hz, jendela 30 detik. Tren pelan dibuang (moving average ~6 detik), lalu dihaluskan, kemudian puncak dihitung (ambang `0,5 x sd`, jarak minimal 1,2 detik). Dihitung terpisah untuk akselerometer dan giroskop (sudut hasil integrasi, sumbu dengan variasi terbesar). Fusi: bila selisih <= 3 bpm, diambil rata-rata; bila tidak, diambil estimasi dengan interval puncak yang lebih teratur.

**Waktu pulih (di dashboard):** baseline = median bpm selama fase baseline. Pulih bila bpm <= baseline x 1,15 pada 2 laporan berturut-turut. Waktu pulih = detik sejak perintah Pemulihan.

## Penyetelan lewat URL

`phone.html?id=hp1&room=KODE&k=0.7&th=3` dengan `k` = ambang puncak napas (naikkan jika hasil napas kacau) dan `th` = ambang repetisi.

## Skenario pengujian

1. Akurasi napas: hitung napas manual 30 detik, bandingkan dengan sistem, 5 kali.
2. Akurasi repetisi: 20 gerakan, bandingkan dengan hitungan sistem.
3. Latensi: rata-rata RTT/2 dari ack di dashboard (tidak bergantung sinkronisasi jam).
4. Dua HP bersamaan: tunjukkan grafik hp1 vs hp2.

Tombol **Unduh log CSV** di dashboard menyimpan semua data untuk tabel hasil.
