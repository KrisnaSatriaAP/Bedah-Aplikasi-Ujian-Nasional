# Bedah Aplikasi Exambro Client 26.0222 (ANBK)

Dokumen ini merangkum hasil analisis statis terhadap arsip `64BitExambrowserClient_26_0222.rar`. Aplikasi tidak dijalankan. Bagian .NET diproteksi obfuskator, jadi peran komponen di sana disimpulkan dari nama kelas, konfigurasi, dan string, bukan dari dekompilasi penuh.

## 1. Ringkasan

| Item | Keterangan |
|---|---|
| Nama | Exambro Client (Exam Browser), Pusmendik Kemdikbud |
| Versi | 26.0222 (22 Februari 2026) |
| Fungsi | Browser mode kiosk untuk ujian ANBK |
| Mode | Daring (server pusat) dan semi daring (server lokal di LAN sekolah) |
| Ukuran | RAR 67 MB, 908 entri, sekitar 194 MB setelah diekstrak |
| Penyusun | Launcher dan service berbasis .NET Framework, browser berbasis Electron |

## 2. Struktur folder

```
64BitExambrowserClient_26.0222/
├── ExamBrowser.exe            # updater / peluncur
├── DotNetZip.dll              # ekstrak paket update
├── setting_exambro.json       # konfigurasi terenkripsi
├── version.txt                # 26.0222
├── updatelog.txt              # riwayat cek dan update versi
├── eb000.dat, log.txt, cameralog.txt
└── Application/
    ├── runningexam.exe        # aplikasi utama (validasi, password, peluncuran)
    ├── LibraryExam.dll        # pustaka inti
    ├── OSVersionInfo.dll
    ├── AForge*.dll, RestSharp.dll, Newtonsoft.Json.dll
    └── service/
        ├── ExamCam.exe        # webcam / proctoring
        ├── ExamServices.exe   # heartbeat, jalur semi daring (komponen lama)
        └── elekbrowser/       # Electron 11.5.0 (Chromium 87, Node 12)
            ├── elekbrowser.exe (x64, 126 MB)
            └── resources/app/ # main.js, preload.js, renderer.js, 404.html, node_modules
```

Catatan: penamaan "64Bit" hanya berlaku untuk Electron. Semua binary .NET-nya x86 (32-bit).

## 3. Peran komponen

| Komponen | Teknologi | Peran |
|---|---|---|
| `ExamBrowser.exe` | .NET x86 | Updater dan peluncur (namespace `Downloadpembaruan`) |
| `runningexam.exe` | .NET x86 | Form utama, splash, validasi perangkat, form password, form settings |
| `elekbrowser.exe` | Electron 11.5.0 x64 | Browser ujian |
| `ExamCam.exe` | .NET + AForge | Snapshot webcam |
| `ExamServices.exe` | .NET + WCF | Heartbeat dan komunikasi ke server lokal (era UNBK) |
| `LibraryExam.dll` | .NET | Hook keyboard/mouse global, kontrol taskbar, desktop terpisah, baca dan dekripsi konfigurasi, validasi aplikasi, registry |
| `OSVersionInfo.dll` | .NET | Deteksi versi dan arsitektur OS |

Semua binary .NET diproteksi .NET Reactor dengan nama tipe diacak.

## 4. Alur kerja

```
ExamBrowser.exe (cek versi & update)
      └─► runningexam.exe (validasi perangkat + password)
              ├─► elekbrowser.exe   argumen: User-Agent, moda, URL ujian
              ├─► ExamCam.exe       snapshot webcam berkala
              └─► ExamServices.exe  heartbeat (jalur semi daring)
```

Antar proses berkomunikasi lewat file JSON kecil: `username.json`, `pesan.json`, `kick.json`.

## 5. Perilaku `main.js` (proses utama Electron)

- Jendela mode kiosk, `nodeIntegration` mati, `webviewTag` menyala, insecure content diizinkan.
- Cache dibersihkan saat load dan header `pragma: no-cache` dikirim.
- `setContentProtection(true)` untuk mencegah screenshot dan perekaman jendela.
- User-Agent diambil dari argumen peluncuran.
- Username peserta dibaca dari elemen `exambrousername` di halaman ujian, lalu ditulis ke `username.json` dengan penanda status (1 aktif, 2 berhenti saat logout atau halaman `Test/TestSelesai`).
- Dua timer polling (aktif jika `aktifkamera=1`):
  - Pesan pengawas: `GET {urlfr}/api/task-get-pesan/{user}/{kegiatan}`, ditampilkan sebagai dialog "Pesan Pengawas".
  - Status peserta: `GET {urlfr}/api/task-get-status-peserta/{user}/{kegiatan}`. Status 2 memunculkan dialog peringatan lalu aplikasi ditutup paksa (timeout cadangan 15 detik).
- Shortcut zoom (Ctrl `+`, `-`, `0`) dan F5 untuk reload dari halaman error.
- Halaman error `404.html` menampilkan kode dan deskripsi error dalam bahasa Indonesia.
- Unduhan disimpan otomatis ke folder Documents dengan prefix timestamp.
- Logging ke `Browser.log` lewat `electron-log`.

## 6. Konfigurasi dan endpoint

`setting_exambro.json` adalah teks terenkripsi (AES-256-CBC). Field yang terlihat dari jejak log:

| Kelompok | Field |
|---|---|
| Identitas | `nameapps`, `versi`, `jenisbit`, `hash64`, `hash32`, `jenisapps` |
| Mode dan kamera | `moda`, `aktifkamera`, `Webcam`, `sharingsession` |
| Interval (ms) | `HeartInterval` 600000, `ImageInterval` 300000, `StatusInterval` 300000, `PesanInterval` 300000 |
| Syarat perangkat | `validwidth` 1023, `validheight` 719, `validram` 1024, `osversion` (Win 7/8/10/2008/2012), `validdesktop` |
| Endpoint | `urlonline`, `urlsemionline`, `urlfr`, `urlgetagent` |
| Lainnya | `Kegiatan`, `unlockkeys`, `validseleksi`, `validparsemi`, `validparonline` |

Endpoint yang teramati:

| Tujuan | Alamat |
|---|---|
| Ujian daring | `anbk-siswa.pusmendik.kemdikbud.go.id` (log terbaru juga memuat `…kemendikdasmen.go.id`) |
| Halaman akun | `anbk-layanan.pusmendik.kemdikbud.go.id` |
| API pesan dan status | `smart-production.pusmendik.kemdikbud.go.id` |
| Semi daring (LAN) | `https://192.168.0.200/`, layanan WCF `puspendikunbkservice/CBTservices/WorkstationService.svc` |

Metode layanan WCF yang terlihat: `SubmitWorkstation`, `SubmitWorkstationV3`, `UnSubmitWorkstation`, `IsSubmitedRegistered`, `PesertaTesBeat` (heartbeat), `GetDataLove1/2/3Server`.

## 7. Riwayat update (`updatelog.txt`)

- Agustus 2024: beberapa kali cek versi, ada periode "Tidak Terkoneksi Internet", lalu update ke 24.0811.
- Mei 2025: update ke 24.1017.
- 22 Februari 2026: status terupdate (versi 26.0222).

## 8. Temuan keamanan dan kualitas paket

| # | Temuan | Dampak |
|---|---|---|
| 1 | Validasi TLS dimatikan (`ignore-certificate-errors`, `NODE_TLS_REJECT_UNAUTHORIZED=0`) | Panggilan API pesan dan status rawan dimanipulasi di jaringan tidak tepercaya |
| 2 | Electron 11.5.0 (Chromium 87) sudah end-of-life | Celah Chromium yang diketahui tidak ditambal |
| 3 | Kunci dekripsi konfigurasi tertanam di `main.js`; password bawaan plaintext di `*.exe.config` dengan nilai default lemah | Obfuskasi tidak memberi perlindungan berarti |
| 4 | `Browser.log` pernah mencatat isi konfigurasi terdekripsi dan menyimpan jejak mesin developer (path `D:\Applikasi\C#\…`, `D:\VirtualBox\…`) | Kebocoran informasi lewat paket distribusi |
| 5 | Folder dev ikut terbawa: `node_modules` lengkap dengan test, `README.md.bak`, `.gitignore`, README template GitLab | Paket membengkak dan permukaan serangan bertambah |
| 6 | Dependensi usang (`request` deprecated, paket `crypto` hanya stub) | Beban pemeliharaan |
| 7 | `ExamServices.exe.config` masih menunjuk Chrome di `Program Files (x86)` | Sisa mode lama, tidak dipakai jalur Electron |
| 8 | Binary .NET Reactor memuat penanda versi belum terdaftar (trial) | Indikasi proteksi tidak berlisensi penuh |

## 9. Jaringan dan troubleshooting

Analisis jaringan dan panduan troubleshooting dipisah ke dokumen tersendiri:

- [exambro-jaringan.md](exambro-jaringan.md): peta dependensi, tumpukan jaringan, interval, checklist kebutuhan jaringan.
- [exambro-troubleshooting.md](exambro-troubleshooting.md): membaca log, kode error, urutan diagnosa, perintah cek.

## 10. Hash SHA-256 binary utama

| File | SHA-256 |
|---|---|
| `runningexam.exe` | `84A20B66DD083EBC34DB8E5AD5A63CFFFFC93215B7891E6166057C79AC455B91` |
| `ExamBrowser.exe` | `062A01A2BD320AEF927127082C3AD4EDD8CE3EF43647DFB84B17ED6412045DD8` |
| `LibraryExam.dll` | `24764739EB8805BB4E51FD3E329353A28C8CB8D57D8E8987EAB75F7368EF44FA` |
| `ExamServices.exe` | `B326FC3B9372AC9699E537CB097E21E2A7E9E3FB53059EE0E2111F7109393B40` |
| `ExamCam.exe` | `981587734C953FC959B145BA8C68A3E563CCE8918AC41023E86F64A72EC308D0` |
| `OSVersionInfo.dll` | `A33662B4A78AA967266ED894B4D891B5FBF666D347B99FA268DDF7FF9D038BE9` |

## 11. Batasan analisis

- Analisis statis, tanpa menjalankan aplikasi.
- Peran binary .NET disimpulkan dari metadata dan string, bukan dari kode hasil dekompilasi.
- `setting_exambro.json` sengaja tidak didekripsi. Nilai kunci dan password tidak dicantumkan di dokumen ini.
