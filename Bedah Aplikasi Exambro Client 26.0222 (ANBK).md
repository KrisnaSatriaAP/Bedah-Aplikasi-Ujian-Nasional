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

## 9. Checklist jaringan (lab atau sekolah)

- [ ] HTTPS (443) ke `anbk-siswa`, `anbk-layanan`, dan `smart-production` di domain Pusmendik (kemdikbud dan kemendikdasmen).
- [ ] DNS resolve normal (error `ENOTFOUND` di log muncul saat DNS atau internet bermasalah).
- [ ] Jalur semi daring: klien bisa menjangkau server lokal (default `192.168.0.200`, port 443).
- [ ] Pastikan latensi dan stabilitas baik. Traffic aplikasinya ringan (polling dan heartbeat berupa request kecil), jadi bandwidth bukan masalah utama. Interval efektifnya berbeda antar sumber, lihat bagian 12.4.
- [ ] Verifikasi TLS dimatikan di aplikasi, jadi SSL inspection biasanya tidak mengganggu.

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

## 12. Panduan troubleshooting jaringan

### 12.1 Peta dependensi jaringan

| Fungsi | Tujuan | Mode | Dipakai oleh | Dampak jika gagal |
|---|---|---|---|---|
| Halaman ujian (daring) | `anbk-siswa.pusmendik.kemdikbud.go.id` :443 (log Feb 2026: `…kemendikdasmen.go.id`) | Daring (`moda`=2) | Chromium di `elekbrowser.exe` | Layar error `404.html` berisi kode Chromium |
| Halaman ujian (lokal) | `https://192.168.0.200/unbk` (login: `/unbk/account/login`) | Semi daring (`moda`=1) | Chromium | Layar error yang sama |
| Generate User-Agent | `anbk-webapi.pusmendik.kemdikbud.go.id/auth/api/v1/user-agent/generate` | Field `urlgetagent` | Launcher .NET (inferensi dari nama kelas `getAgent`) | Kemungkinan gagal meluncurkan atau ditolak server |
| Pesan pengawas dan status "kick" | `smart-production.pusmendik.kemdikbud.go.id/api/task-get-pesan/…` dan `…/task-get-status-peserta/…` | Jika `aktifkamera`=1 | Node `request` di proses utama Electron | Log `getapipesan=` dan `writestatusjson=` error; pesan dan kick tidak masuk, ujian tetap berjalan |
| Heartbeat / registrasi workstation | `192.168.0.200/puspendikunbkservice/CBTservices/WorkstationService.svc` | Semi daring | `ExamServices.exe` (WCF) | Peserta tidak terpantau di server lokal |
| Cek update | Endpoint tidak teridentifikasi | Saat start | `ExamBrowser.exe` | Baris "Tidak Terkoneksi Internet" di `updatelog.txt` |
| Snapshot webcam | Endpoint tidak teridentifikasi | `aktifkamera`=1 | `ExamCam.exe` | Belum bisa dipastikan |

Penanda mode terverifikasi dari log: `moda`=2 membuka `anbk-siswa…`, `moda`=1 membuka `192.168.0.x/unbk`.

### 12.2 Tiga tumpukan jaringan yang berbeda

| Tumpukan | Proxy | Validasi TLS |
|---|---|---|
| Chromium (halaman ujian) | Mengikuti proxy sistem | Error sertifikat diabaikan |
| Node `request` (polling pesan/status) | Tidak mengikuti proxy sistem, hanya variabel lingkungan `HTTP_PROXY`/`HTTPS_PROXY` | Dimatikan (`NODE_TLS_REJECT_UNAUTHORIZED=0`) |
| .NET (launcher, WCF, updater) | Proxy default .NET (pengaturan sistem) | Ikut aturan OS |

Konsekuensinya: di jaringan yang mewajibkan proxy eksplisit tanpa akses langsung ke internet, halaman ujian bisa terbuka normal sementara polling pesan/kick gagal. Gejala "ujian jalan tapi pengawas tidak bisa kirim pesan atau mengeluarkan peserta" mengarah ke jalur Node (DNS, proxy, atau rute langsung).

Polling juga hanya berjalan setelah username peserta terdeteksi (`username` tidak kosong). Kalau tidak pernah muncul baris `datausername{…}` di log, polling tidak aktif sama sekali.

### 12.3 Membaca `Browser.log`

Lokasi: `Application\service\elekbrowser\resources\app\Browser.log`.

| Baris log | Arti | Tindakan |
|---|---|---|
| `ReadPesan= / ReadKick= / Readname= … ENOENT` | File JSON belum ada saat start pertama | Normal, abaikan |
| `CONFIG= {…}` dengan level `[error]` | Hanya dump konfigurasi, bukan kegagalan | Abaikan (tapi log ini membocorkan konfigurasi) |
| `respone= 200` lalu `dfl=loading close&showpage` lalu `masuk kedalam url` + URL | Urutan sehat, halaman ujian termuat | Tidak ada |
| `kirim web login` | Fitur kamera aktif | Cocokkan dengan `CAM= 1` |
| `tidak kirimgambar web login` | Fitur kamera tidak aktif | Cocokkan dengan `aktifkamera` |
| `datausername{"username":"…","statuskirim":"1"}` | Username terdeteksi, polling aktif | Jika tidak pernah muncul, polling tidak jalan |
| Baris `null` berulang | Muncul di sesi yang akhirnya sukses, jadi bukan indikator gagal | Abaikan |
| `getapipesan=Error: getaddrinfo ENOTFOUND <host>` dan `writestatusjson= …` (berpasangan, teramati tiap sekitar 30 detik) | Resolusi DNS gagal untuk host API dari sisi klien | Periksa DNS klien/resolver, bukan firewall |
| `dfl.errcode=<n> errdesc=…` | `did-fail-load`, layar `404.html` tampil | Lihat tabel 12.3.1 |
| `dn=…` dengan status 500 | Server mengembalikan 500 | Sisi server, bukan jaringan lokal |

Layar `404.html` menampilkan kode dan deskripsi error. F5 memuat ulang.

#### 12.3.1 Kode error Chromium yang lazim

| Kode | Nama | Arah diagnosa |
|---|---|---|
| -105 | `ERR_NAME_NOT_RESOLVED` | DNS: resolver, filtering, atau domain diblokir |
| -106 | `ERR_INTERNET_DISCONNECTED` | Link atau gateway putus |
| -102 | `ERR_CONNECTION_REFUSED` | Port ditolak; untuk semi daring cek layanan di server lokal atau rule reject |
| -118 / -7 | `ERR_CONNECTION_TIMED_OUT` / `ERR_TIMED_OUT` | Paket di-drop: firewall, routing, NAT |
| -109 | `ERR_ADDRESS_UNREACHABLE` | Tidak ada rute: routing atau VLAN |
| -101 | `ERR_CONNECTION_RESET` | Direset middlebox (IPS/proxy) atau server kelebihan beban |
| -130 / -111 | `ERR_PROXY_CONNECTION_FAILED` / `ERR_TUNNEL_CONNECTION_FAILED` | Masalah proxy sistem |

Error sertifikat (`ERR_CERT_*`) jarang jadi penyebab di browser ini karena pengecekan sertifikat dimatikan.

### 12.4 Interval: jangan pegang satu angka

- JSON konfigurasi (terlihat di log): `HeartInterval` 600000 ms, `ImageInterval`/`StatusInterval`/`PesanInterval` 300000 ms.
- `*.exe.config`: `HeartInterval` 60000 ms.
- Log runtime satu sesi menunjukkan polling gagal tiap sekitar 30 detik.

Nilai efektif polling dibaca `main.js` dari `setting_exambro.json`, jadi bisa berbeda dari angka di sisi .NET. Ukur dari log atau packet capture.

### 12.5 Diagnosa berurutan

1. **Semua klien atau satu klien?** Semua klien berarti gateway, DNS, atau upstream. Satu klien berarti NIC, proxy, DNS lokal, atau jam.
2. **DNS:** resolve semua FQDN di 12.1 dari klien yang bermasalah.
3. **TCP 443:** uji ke tiap tujuan, termasuk `192.168.0.200` untuk semi daring.
4. **Proxy:** cocokkan pengaturan proxy sistem dengan variabel lingkungan `HTTP(S)_PROXY`.
5. **Sisi server:** status 500 atau server lokal tidak merespons bukan masalah jaringan klien.

#### Windows (klien)

```powershell
Resolve-DnsName anbk-siswa.pusmendik.kemdikbud.go.id
Resolve-DnsName smart-production.pusmendik.kemdikbud.go.id
Resolve-DnsName anbk-webapi.pusmendik.kemdikbud.go.id
Test-NetConnection anbk-siswa.pusmendik.kemdikbud.go.id -Port 443
Test-NetConnection 192.168.0.200 -Port 443
curl.exe -vkI https://smart-production.pusmendik.kemdikbud.go.id/
netsh winhttp show proxy
```

#### MikroTik RouterOS

```
# nama yang dikueri klien (jika router jadi resolver DNS)
/ip dns cache print where name~"pusmendik"

# whitelist berbasis FQDN
/ip firewall address-list add list=exambro address=anbk-siswa.pusmendik.kemdikbud.go.id
/ip firewall address-list add list=exambro address=smart-production.pusmendik.kemdikbud.go.id
/ip firewall address-list add list=exambro address=anbk-webapi.pusmendik.kemdikbud.go.id

# koneksi aktif dari klien tertentu
/ip firewall connection print where src-address~"192.168.x.x"

# tangkap paket klien selama simulasi
/tool sniffer quick ip-address=192.168.x.x port=443
```

Karena domain ujian sudah muncul di `kemendikdasmen.go.id`, tambahkan juga padanan domain itu dan cek lewat DNS cache apakah host API ikut berpindah.

### 12.6 Yang belum bisa dipastikan secara statis

- Endpoint cek update dan upload snapshot webcam (kode .NET diobfuskasi). Cara melengkapi: catat FQDN lewat `/ip dns cache` dan sniffer saat simulasi (start aplikasi, login, satu siklus snapshot).
- Apakah host `smart-production` dan `anbk-webapi` juga punya padanan `kemendikdasmen.go.id`. Verifikasi lewat DNS log.
- Hipotesis untuk PC lama (Win 7/8): launcher .NET berjalan di mode kompatibilitas runtime 4.0, sehingga TLS 1.2 bisa tidak aktif. Jika launcher gagal terhubung padahal browser biasa bisa, periksa dukungan TLS 1.2 di OS. Belum diverifikasi.
