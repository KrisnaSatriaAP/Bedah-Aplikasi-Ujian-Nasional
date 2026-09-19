# Exambro Client 26.0222: Analisis Jaringan

Dokumen pendamping [bedah-exambro-client-26.0222.md](bedah-exambro-client-26.0222.md). Isinya kebutuhan dan perilaku jaringan aplikasi. Untuk log, kode error, dan langkah diagnosa lihat [exambro-troubleshooting.md](exambro-troubleshooting.md).

Analisis ini statis (tanpa menjalankan aplikasi). Bagian .NET diobfuskasi, jadi beberapa endpoint tidak teridentifikasi.

## 1. Peta dependensi jaringan

| Fungsi | Tujuan | Mode | Dipakai oleh | Dampak jika gagal |
|---|---|---|---|---|
| Halaman ujian (daring) | `anbk-siswa.pusmendik.kemdikbud.go.id` :443 (log Feb 2026: `…kemendikdasmen.go.id`) | Daring (`moda`=2) | Chromium di `elekbrowser.exe` | Layar error `404.html` berisi kode Chromium |
| Halaman ujian (lokal) | `https://192.168.0.200/unbk` (login: `/unbk/account/login`) | Semi daring (`moda`=1) | Chromium | Layar error yang sama |
| Generate User-Agent | `anbk-webapi.pusmendik.kemdikbud.go.id/auth/api/v1/user-agent/generate` | Field `urlgetagent` | Launcher .NET (inferensi dari nama kelas `getAgent`) | Kemungkinan gagal meluncurkan atau ditolak server |
| Pesan pengawas dan status "kick" | `smart-production.pusmendik.kemdikbud.go.id/api/task-get-pesan/…` dan `…/task-get-status-peserta/…` | Jika `aktifkamera`=1 | Node `request` di proses utama Electron | Pesan dan kick tidak masuk, ujian tetap berjalan |
| Heartbeat / registrasi workstation | `192.168.0.200/puspendikunbkservice/CBTservices/WorkstationService.svc` | Semi daring | `ExamServices.exe` (WCF) | Peserta tidak terpantau di server lokal |
| Cek update | Endpoint tidak teridentifikasi | Saat start | `ExamBrowser.exe` | Baris "Tidak Terkoneksi Internet" di `updatelog.txt` |
| Snapshot webcam | Endpoint tidak teridentifikasi | `aktifkamera`=1 | `ExamCam.exe` | Belum bisa dipastikan |

Penanda mode terverifikasi dari log: `moda`=2 membuka `anbk-siswa…`, `moda`=1 membuka `192.168.0.x/unbk`.

Protokol yang terlihat hanya HTTPS (443). Layanan WCF berupa SOAP di atas HTTPS.

## 2. Tiga tumpukan jaringan yang berbeda

| Tumpukan | Proxy | Validasi TLS |
|---|---|---|
| Chromium (halaman ujian) | Mengikuti proxy sistem | Error sertifikat diabaikan (`ignore-certificate-errors`) |
| Node `request` (polling pesan/status) | Tidak mengikuti proxy sistem, hanya variabel lingkungan `HTTP_PROXY`/`HTTPS_PROXY` | Dimatikan (`NODE_TLS_REJECT_UNAUTHORIZED=0`) |
| .NET (launcher, WCF, updater) | Proxy default .NET (pengaturan sistem) | Ikut aturan OS |

Konsekuensi praktis:

- Di jaringan yang mewajibkan proxy eksplisit tanpa akses langsung ke internet, halaman ujian bisa terbuka normal sementara polling pesan/kick gagal.
- SSL inspection biasanya tidak mengganggu sisi Chromium dan Node karena validasi sertifikatnya dimatikan. Sisi .NET tetap mengikuti aturan OS.
- Polling hanya berjalan setelah username peserta terdeteksi.

## 3. Interval: jangan pegang satu angka

| Sumber | Nilai |
|---|---|
| JSON konfigurasi (terlihat di log) | `HeartInterval` 600000 ms, `ImageInterval` / `StatusInterval` / `PesanInterval` 300000 ms |
| `*.exe.config` | `HeartInterval` 60000 ms |
| Log runtime satu sesi | Polling gagal tiap sekitar 30 detik |

Nilai efektif polling dibaca `main.js` dari `setting_exambro.json`, jadi bisa berbeda dari angka di sisi .NET. Ukur dari log atau packet capture.

Beban traffic ringan: request kecil berkala, bukan aliran data besar. Yang menentukan kualitas adalah latensi dan stabilitas DNS, bukan bandwidth.

## 4. Checklist kebutuhan jaringan (lab atau sekolah)

- [ ] HTTPS (443) ke `anbk-siswa`, `anbk-layanan`, `anbk-webapi`, dan `smart-production` di domain Pusmendik (kemdikbud dan kemendikdasmen).
- [ ] DNS resolve normal untuk semua host di atas.
- [ ] Jalur semi daring: klien menjangkau server lokal (default `192.168.0.200`, port 443).
- [ ] Jika memakai proxy: pastikan jalur Node (variabel lingkungan) ikut terkonfigurasi, tidak hanya proxy sistem.
- [ ] Latensi dan stabilitas baik. Bandwidth bukan masalah utama.

Contoh whitelist berbasis FQDN di MikroTik:

```
/ip firewall address-list add list=exambro address=anbk-siswa.pusmendik.kemdikbud.go.id
/ip firewall address-list add list=exambro address=anbk-layanan.pusmendik.kemdikbud.go.id
/ip firewall address-list add list=exambro address=anbk-webapi.pusmendik.kemdikbud.go.id
/ip firewall address-list add list=exambro address=smart-production.pusmendik.kemdikbud.go.id
```

Karena domain ujian sudah muncul di `kemendikdasmen.go.id`, tambahkan juga padanan domain itu.

## 5. Yang belum bisa dipastikan

- Endpoint cek update dan upload snapshot webcam (kode .NET diobfuskasi). Cara melengkapi: catat FQDN yang dikueri klien saat simulasi (start aplikasi, login, satu siklus snapshot) lewat `/ip dns cache print where name~"pusmendik"` atau sniffer.
- Apakah `smart-production` dan `anbk-webapi` juga punya padanan `kemendikdasmen.go.id`. Verifikasi lewat DNS log.
