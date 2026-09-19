# Exambro Client 26.0222: Panduan Troubleshooting

Dokumen pendamping [bedah-exambro-client-26.0222.md](bedah-exambro-client-26.0222.md). Kebutuhan dan perilaku jaringan ada di [exambro-jaringan.md](exambro-jaringan.md).

## 1. Urutan diagnosa

1. **Semua klien atau satu klien?** Semua klien berarti gateway, DNS, atau upstream. Satu klien berarti NIC, proxy, DNS lokal, atau jam.
2. **DNS:** resolve semua FQDN yang dipakai (daftar di dokumen jaringan) dari klien yang bermasalah.
3. **TCP 443:** uji ke tiap tujuan, termasuk `192.168.0.200` untuk semi daring.
4. **Proxy:** cocokkan pengaturan proxy sistem dengan variabel lingkungan `HTTP(S)_PROXY`.
5. **Sisi server:** status 500 atau server lokal tidak merespons bukan masalah jaringan klien.

## 2. Gejala dan dugaan penyebab

Dugaan ini disimpulkan dari analisis statis dan log, belum diuji langsung.

| Gejala | Dugaan | Periksa |
|---|---|---|
| Halaman ujian tidak terbuka, muncul layar error | DNS, rute, atau port 443 | Kode error (bagian 4), lalu langkah 2 dan 3 |
| Halaman terbuka, tapi pesan pengawas atau kick tidak masuk | Jalur Node: DNS, proxy, atau rute langsung | Baris `getapipesan=` di log, proxy vs variabel lingkungan |
| Baris `datausername{…}` tidak pernah muncul | Username tidak terdeteksi, polling tidak aktif | Sesi login benar-benar selesai, halaman ujian termuat penuh |
| Semi daring: peserta tidak terpantau di server lokal | Heartbeat WCF ke `192.168.0.200` gagal | TCP 443 ke server lokal, layanan `WorkstationService.svc` |
| `updatelog.txt` berisi "Tidak Terkoneksi Internet" | Cek update gagal saat start | Internet dan DNS saat aplikasi dibuka |
| Kamera aktif di konfigurasi tapi tidak terkirim | `ExamCam.exe` atau jalur upload snapshot | Log `kirim web login` vs `tidak kirimgambar web login`, `ExamCam.exe` berjalan |

## 3. Membaca `Browser.log`

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
| `dfl.errcode=<n> errdesc=…` | `did-fail-load`, layar `404.html` tampil | Lihat bagian 4 |
| `dn=…` dengan status 500 | Server mengembalikan 500 | Sisi server, bukan jaringan lokal |

Layar `404.html` menampilkan kode dan deskripsi error. F5 memuat ulang halaman.

Catatan: log ini berisi konfigurasi terdekripsi dan bisa memuat identitas peserta. Jangan dibagikan mentah-mentah.

## 4. Kode error Chromium yang lazim

| Kode | Nama | Arah diagnosa |
|---|---|---|
| -105 | `ERR_NAME_NOT_RESOLVED` | DNS: resolver, filtering, atau domain diblokir |
| -106 | `ERR_INTERNET_DISCONNECTED` | Link atau gateway putus |
| -102 | `ERR_CONNECTION_REFUSED` | Port ditolak; untuk semi daring cek layanan di server lokal atau rule reject |
| -118 / -7 | `ERR_CONNECTION_TIMED_OUT` / `ERR_TIMED_OUT` | Paket di-drop: firewall, routing, NAT |
| -109 | `ERR_ADDRESS_UNREACHABLE` | Tidak ada rute: routing atau VLAN |
| -101 | `ERR_CONNECTION_RESET` | Direset middlebox (IPS/proxy) atau server kelebihan beban |
| -130 / -111 | `ERR_PROXY_CONNECTION_FAILED` / `ERR_TUNNEL_CONNECTION_FAILED` | Masalah proxy sistem |
| 500 | Internal Server Error | Sisi server |

Error sertifikat (`ERR_CERT_*`) jarang jadi penyebab di browser ini karena pengecekan sertifikat dimatikan.

## 5. Perintah cek

### Windows (klien)

```powershell
Resolve-DnsName anbk-siswa.pusmendik.kemdikbud.go.id
Resolve-DnsName smart-production.pusmendik.kemdikbud.go.id
Resolve-DnsName anbk-webapi.pusmendik.kemdikbud.go.id
Test-NetConnection anbk-siswa.pusmendik.kemdikbud.go.id -Port 443
Test-NetConnection 192.168.0.200 -Port 443
curl.exe -vkI https://smart-production.pusmendik.kemdikbud.go.id/
netsh winhttp show proxy
```

### MikroTik RouterOS

```
# nama yang dikueri klien (jika router jadi resolver DNS)
/ip dns cache print where name~"pusmendik"

# koneksi aktif dari klien tertentu
/ip firewall connection print where src-address~"192.168.x.x"

# tangkap paket klien selama simulasi
/tool sniffer quick ip-address=192.168.x.x port=443
```

## 6. Hipotesis yang belum diverifikasi

- PC lama (Win 7/8): launcher .NET berjalan di mode kompatibilitas runtime 4.0, sehingga TLS 1.2 bisa tidak aktif. Jika launcher gagal terhubung padahal browser biasa bisa, periksa dukungan TLS 1.2 di OS.
- Interval polling efektif bisa berbeda dari angka di konfigurasi .NET. Ukur dari log atau capture.
