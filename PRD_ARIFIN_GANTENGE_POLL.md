# PRD — ARIFIN GANTENGE POLL

**Versi:** 2.0 (final specification)  
**Jenis:** Captive portal WiFi rumah + voucher gratis  
**Platform:** Ubuntu Server, USB-to-LAN, ZTE/Huawei bekas ISP sebagai access point  
**Bahasa:** Indonesia  
**Zona waktu:** Asia/Jakarta  
**Implementasi:** OpenCode, bertahap dengan approval untuk perubahan jaringan

## 1. Ringkasan produk

Bangun aplikasi ringan bernama **ARIFIN GANTENGE POLL** untuk memberi akses internet sementara kepada teman yang berkunjung ke rumah. Tamu menghubungkan perangkat ke SSID `ARIFIN GANTENGE POLL` (WiFi tamu terbuka), lalu sistem operasi **bila mendukung** menampilkan captive portal. Jika popup tidak muncul, sediakan URL lokal untuk dibuka manual. Halaman pertama bertuliskan **“Password-nya apa?”**. Password tamu awal: `Arifinganteng`. Setelah lolos, tamu melihat katalog voucher gratis yang masih dipublikasikan dan memiliki stok; mereka dapat mengklaim satu voucher dan langsung mengaktifkan akses internet sesuai durasi. Ketika masa berlaku habis, gateway benar-benar memutus akses.

Ini proyek **rumahan**, bukan SaaS atau bisnis. Tidak ada harga, pembayaran, laporan penjualan, multi-tenant, atau infrastruktur cloud wajib.

## 2. Sasaran dan batasan

- Mudah digunakan teman tanpa instal aplikasi.
- Admin rumah dapat mengatur paket, stok, durasi, kecepatan opsional, dan foto profil.
- Ringan pada Ubuntu Server dan browser captive portal HP.
- Aman bagi jaringan utama: tamu tidak boleh mengakses SSH, dashboard admin, atau perangkat LAN utama.
- Tidak menjanjikan popup otomatis di setiap OS/perangkat.
- Tidak mengubah jaringan aktif sebelum ada persetujuan eksplisit pemilik.

### Bukan bagian MVP

QRIS, billing, reseller, RADIUS cluster, Docker/Kubernetes, akun pelanggan, notifikasi WhatsApp, integrasi cloud, kuota data presisi, dan mode menghitung hanya waktu online. Kuota dan limit bandwidth lanjutan boleh menjadi tahap opsional setelah pengujian.

## 3. Topologi dan perangkat

```text
Internet / Router ISP utama
          |
     LAN bawaan PC (WAN, DHCP)
          |
      Ubuntu Server
      - Netplan/networkd
      - nftables NAT + firewall
      - dnsmasq DHCP/DNS
      - openNDS captive portal
      - aplikasi ARIFIN GANTENGE POLL
          |
     USB-to-LAN (LAN 10.10.10.1/24)
          |
    LAN port ZTE/Huawei
    AP/bridge, DHCP off
          |
   SSID: ARIFIN GANTENGE POLL
          |
      HP teman/tamu
```

Rencana alamat (harus dicek konflik): Ubuntu LAN `10.10.10.1/24`, AP `10.10.10.2`, DHCP tamu `10.10.10.100–250`. Nama interface **harus dideteksi**, bukan diasumsikan. Gunakan LAN-to-LAN, bukan port optik/GPON; jangan mengubah LOID/serial/firmware ISP. Bila firmware AP tidak bisa mematikan DHCP atau tidak mendukung AP bridge, laporkan hambatan dan alternatif, jangan paksa konfigurasi.

## 4. Stack sederhana

- **Frontend:** React + Vite + TypeScript + Tailwind CSS + Motion for React + Lucide.
- **Backend:** Node.js LTS + Fastify + TypeScript.
- **Data:** SQLite dengan better-sqlite3 atau Drizzle; WAL, migrasi, backup.
- **Gateway:** Netplan, dnsmasq, nftables, openNDS dengan FAS aman sesuai versi yang benar-benar tersedia.
- **Service:** systemd; Nginx/Caddy opsional bila dibutuhkan untuk HTTPS dan routing.
- **Aset:** semuanya lokal, tanpa ketergantungan CDN untuk captive portal.

Backend aplikasi **tidak** berjalan sebagai root. Jika operasi jaringan butuh privilege, gunakan helper terpisah dengan daftar operasi terbatas. Dashboard admin hanya di jaringan manajemen yang dipercaya.

## 5. Alur pengguna

1. Tamu tersambung ke SSID `ARIFIN GANTENGE POLL`.
2. Mendapat DHCP, DNS, dan gateway Ubuntu.
3. Perangkat mendeteksi captive portal jika didukung; URL portal manual tersedia jika tidak.
4. Halaman awal menampilkan avatar/foto profil, branding, teks **“Password-nya apa?”**, input password, tombol **“Masuk”**.
5. Password diverifikasi di server; awalnya `Arifinganteng` (disimpan sebagai hash Argon2id, tidak ditanam dalam JS frontend).
6. Setelah benar, tamu melihat **“Voucher Gratis”** dengan paket publik yang masih aktif dan stok tersedia.
7. Tamu memilih paket, menekan **“Klaim Gratis”**.
8. Backend secara atomik mengalokasikan voucher dan meminta openNDS mengizinkan sesi perangkat itu. Jika aktivasi gagal, stok tidak boleh hilang permanen.
9. Tampilkan **“Yeay, Internet Aktif!”**, nama paket, sisa waktu, waktu berakhir, tombol **“Mulai Browsing”**.
10. Saat expired/revoked, openNDS memutus akses. Portal menampilkan **“Waktunya Habis!”** dan opsi kembali ke voucher jika masih ada.

**Aturan:** password gerbang adalah syarat untuk melihat/mengklaim voucher; klaim publik berarti tidak perlu kode rahasia tambahan. Satu sesi tamu hanya satu voucher aktif. Password tamu berbeda dari kredensial admin. Tidak ada password WiFi WPA untuk SSID tamu jika captive portal otomatis menjadi prioritas; jelaskan trade-off keamanan SSID terbuka.

## 6. Halaman dan copy final

### 6.1 Password gate `/`

- Brand: **ARIFIN GANTENGE POLL**
- Heading: **Password-nya apa?**
- Subheading: **Cuma yang tahu password boleh masuk 😎**
- Placeholder: **Masukkan password...**
- Tombol: **Masuk**
- Kesalahan: **Password salah. Coba lagi!**
- Tampilkan avatar ganteng bundar di atas heading.
- Tombol show/hide password; submit via Enter; loading dan error state.

### 6.2 Katalog `/vouchers`

- Heading: **Voucher Gratis**
- Subheading: **Pilih voucher internet gratis buat kamu!**
- Kartu: nama, durasi, download speed (jika diaktifkan), stok tersisa, tombol **Klaim Gratis**.
- Jika tidak tersedia: **Voucher sedang tidak tersedia. Coba lagi nanti.**
- Hanya tampilkan paket `published=true`, `enabled=true`, dalam periode publikasi (jika ada), dan stok > 0.
- Jangan tampilkan kode voucher orang lain atau secret internal.

### 6.3 Berhasil `/connected`

- **Yeay, Internet Aktif!**
- Paket, waktu berakhir, countdown (berdasarkan timestamp server), status, tombol **Mulai Browsing**.
- Status koneksi harus bersumber dari backend/openNDS, bukan hanya UI.

### 6.4 Expired `/expired`

- **Waktunya Habis!**
- **Voucher kamu sudah habis. Pilih voucher lain kalau masih tersedia.**
- Tombol **Lihat Voucher Gratis**.

### 6.5 Admin `/admin`

- Login admin terpisah.
- Menu minimal: Ringkasan, Paket Voucher, Pengguna Online, Tampilan Portal, Pengaturan, Kesehatan Server.
- Aksi: buat/edit/nonaktifkan paket, set durasi/stok, lihat sesi, putus sesi, ubah password tamu, upload/ganti foto profil, ubah nama tampilan, lihat status WAN/openNDS.
- Tidak dapat diakses dari subnet tamu.

## 7. Foto profil orang ganteng

Tambahkan **profil avatar** sebagai identitas visual portal dan preview dashboard. Spesifikasi:

- Default: placeholder ilustrasi pria dewasa bergaya premium dan ramah, **bukan** foto orang nyata tertentu tanpa izin.
- Admin bisa upload gambar JPG/PNG/WebP milik sendiri, maksimal 2 MB; validasi MIME berdasarkan isi file, ukuran, dan dimensi; decode/re-encode di server untuk menghapus metadata; hindari SVG upload.
- Proses ke WebP teroptimasi dengan fallback; simpan di penyimpanan lokal non-eksekusi; nama file acak; akses melalui endpoint statis aman.
- Crop persegi 1:1, avatar bundar, ukuran ideal 512×512; dukung penggantian, reset default, preview.
- Visual: ring cyan-violet beranimasi lembut, glow tipis, partikel kecil, hover tilt halus (desktop), pulse sangat pelan; jangan mengganggu teks.
- Avatar muncul di password gate, halaman voucher (versi kecil), dan halaman connected.
- Gunakan `alt="Foto profil ARIFIN GANTENGE POLL"`.
- Aset lokal harus tetap bisa dimuat sebelum autentikasi; masukkan ke walled garden lokal.
- Jangan otomatis mengambil foto profil dari internet atau mengasumsikan foto pengguna tersedia.

## 8. Arah desain: full animation, tetap ringan

- Tema dark navy `#080E20`, surface `#121C38`, cyan `#00C6FF`, electric blue `#0A6CFF`, violet `#7B2FFF`.
- Glassmorphism secukupnya, gradient mesh halus, floating light orbs, animasi ikon WiFi, staggered voucher cards, loading shimmer, success check animation, transitions antarhalaman.
- Mobile-first 320px ke atas; kontras dan ukuran tap target nyaman; fokus keyboard terlihat; mendukung `prefers-reduced-motion`.
- Gunakan transform/opacity untuk animasi; hindari efek blur/particle berat di perangkat lemah.
- Tidak memuat Google Fonts/CDN sebelum login; host font lokal dengan lisensi sesuai atau gunakan system font.
- Tidak perlu video autoplay atau WebGL berat; batasi bundle JS dan uji browser mini captive portal.
- Berikan desain yang terlihat premium tanpa mengorbankan aksesibilitas, daya baterai, dan kecepatan portal.

## 9. Voucher dan aturan sesi

- Paket contoh: 30 menit, 1 jam, 3 jam, 6 jam, 12 jam, 24 jam; admin dapat membuat durasi custom.
- Semua gratis; tidak ada harga atau transaksi.
- Setiap paket punya `name`, `durationMinutes`, `claimLimit`, `claimedCount`, `isPublished`, `isEnabled`, `downloadKbps?`, `uploadKbps?`, dan opsional waktu publikasi.
- Klaim harus menggunakan transaksi database agar stok tidak minus pada klik bersamaan.
- Durasi dihitung **elapsed time** dari aktivasi pertama, bukan waktu online; reconnect tidak memperpanjang.
- Satu sesi satu voucher aktif; identitas sesi diverifikasi dengan mekanisme openNDS yang aman; MAC tidak cukup sebagai bukti tunggal.
- Jika admin menonaktifkan paket, voucher yang sudah aktif tetap berjalan kecuali admin secara eksplisit mencabutnya.
- Reboot server tidak mereset masa berlaku; pada startup rekonsiliasi SQLite dan sesi openNDS.
- Bila gateway/auth service bermasalah, **fail closed** untuk tamu baru; jangan memberikan akses tanpa login.
- Proteksi klaim berulang dan rate limit, namun jangan mengklaim anti-abuse sempurna pada SSID terbuka.

## 10. Model data SQLite

- `admin_users`: id, username, password_hash, created_at.
- `settings`: key, value, updated_at; secret/password disimpan aman dan tidak dicatat mentah.
- `voucher_plans`: id, name, duration_minutes, download_kbps, upload_kbps, claim_limit, claimed_count, is_published, is_enabled, publish_start, publish_end, created_at, updated_at.
- `vouchers`: id, plan_id, code_hash (opsional jika auto-claim memakai token internal), status, activated_at, expires_at, created_at.
- `guest_sessions`: id, voucher_id, gateway_client_id, client_ip, client_mac, started_at, ended_at, last_seen_at, status.
- `audit_events`: id, actor_type, action, target, created_at, metadata_sanitized.
- `portal_assets`: id, kind, filename, mime, width, height, created_at.

Gunakan constraints, index, foreign key, migrasi, dan transaksi. Jangan menyimpan password tamu atau admin plaintext. Pastikan klaim dan aktivasi memiliki idempotency key/recovery flow untuk kegagalan jaringan.

## 11. API minimum

- `POST /api/guest/unlock` — verifikasi password tamu, buat sesi terbatas.
- `GET /api/guest/plans` — paket publik yang tersedia (setelah unlock).
- `POST /api/guest/claim` — klaim atomik dan otorisasi openNDS.
- `GET /api/guest/session` — status/sisa waktu dari server.
- `POST /api/admin/login`, `POST /api/admin/logout`.
- `GET/POST/PATCH /api/admin/plans`.
- `GET /api/admin/sessions`, `POST /api/admin/sessions/:id/disconnect`.
- `POST /api/admin/avatar` — upload foto tervalidasi.
- `DELETE /api/admin/avatar` — kembali ke default.
- `GET/PATCH /api/admin/settings`.
- `GET /api/admin/health`.

Validasi dengan schema, CSRF bila memakai cookie, rate limit, role guard, cookie `HttpOnly`/`SameSite`, `Secure` pada HTTPS, dan sanitasi log.

## 12. Keamanan jaringan

- Tamu tidak boleh mengakses SSH, admin panel, database, atau LAN utama.
- Buat aturan nftables yang eksplisit untuk subnet tamu dan forwarding hanya yang diizinkan.
- Pastikan portal/openNDS benar-benar mengendalikan traffic, bukan sekadar halaman login dekoratif.
- Deteksi dan kendalikan IPv6 agar tidak menjadi bypass.
- Tidak ada TLS interception atau sertifikat palsu; popup captive portal adalah fitur OS yang tidak selalu muncul.
- Pastikan DNS, asset portal lokal, dan endpoint autentikasi dapat diakses sebelum login tanpa membuka akses internet penuh.
- Admin dapat mengakses panel hanya melalui interface manajemen tepercaya; jangan publish panel di jaringan tamu.
- Jangan hardcode password tamu `Arifinganteng` di bundle frontend. Saat inisialisasi, set hash secara aman dan izinkan perubahan.
- Gunakan satu kata sandi admin yang berbeda dan kuat, dibuat saat setup; jangan memakai password tamu untuk admin.

## 13. Tahap implementasi untuk OpenCode

### Phase 0 — Discovery (read-only)

Periksa OS, hardware, interface, routing, SSH, layanan, firewall, port, versi openNDS, dan kemampuan AP. Buat `docs/discovery.md`, `docs/network-plan.md`, `docs/rollback.md`. **Jangan mengubah jaringan**.

### Phase 1 — Aplikasi lokal

Bangun frontend animasi, avatar default dan upload admin, password gate server-side, katalog, klaim voucher, SQLite, admin panel sederhana. Gunakan mock gateway adapter untuk pengujian; jangan berpura-pura internet sudah terautentikasi.

### Phase 2 — Gateway (butuh approval)

Setelah pemilik menyetujui perubahan, buat backup konfigurasi dan mekanisme rollback, lalu terapkan WAN/LAN, DHCP, DNS, NAT, firewall, openNDS/FAS. Periksa agar koneksi manajemen tetap tersedia. Jangan jalankan `netplan apply`/ubah nftables aktif tanpa persetujuan.

### Phase 3 — Integrasi nyata

Hubungkan voucher dengan openNDS; uji popup dan fallback URL, login, klaim, timeout, disconnect, reboot, dan isolasi tamu dari jaringan utama.

### Phase 4 — QA dan dokumentasi

Uji pada Android dan iOS jika perangkat tersedia; browser desktop; 320px; reduced motion; upload file berbahaya; klaim bersamaan; password salah; stok habis; akses admin dari tamu; expiry; pemulihan layanan. Dokumentasikan keterbatasan yang tidak bisa diuji.

## 14. Acceptance criteria

1. Nama di UI dan dokumentasi adalah **ARIFIN GANTENGE POLL**.
2. SSID target menggunakan nama yang sama bila perangkat mendukung.
3. Halaman awal menampilkan foto profil dan teks **Password-nya apa?**.
4. Password tamu awal `Arifinganteng` diverifikasi server-side, bukan frontend.
5. Password salah ditolak; rate limiting aktif.
6. Setelah password benar, katalog menampilkan hanya paket publik dengan stok.
7. Tamu dapat klaim voucher gratis dan mendapatkan internet setelah otorisasi openNDS sukses.
8. Stok tidak minus pada klaim paralel; satu sesi tidak dapat klaim berulang saat aktif.
9. Akses benar-benar berakhir ketika voucher expired; reconnect/reboot tidak memperpanjang durasi.
10. Avatar default tampil offline, admin bisa upload/ganti/reset foto secara aman.
11. UI animatif, responsif, dan mendukung reduced motion.
12. Dashboard admin tidak dapat diakses dari jaringan tamu.
13. Pengujian fisik membuktikan jalur captive portal dan NAT berjalan.
14. Semua perubahan jaringan memiliki backup, approval, dan rollback yang terdokumentasi.

## 15. Struktur proyek

```text
arifin-gantenge-poll/
├── PRD.md
├── README.md
├── apps/
│   ├── portal/                # React + Vite
│   └── server/                # Fastify + SQLite
├── packages/
│   └── shared/                # Types, schemas
├── services/
│   └── gateway-adapter/       # Mock + openNDS adapter
├── assets/
│   └── default-avatar.webp   # Ilustrasi original/berlisensi
├── infrastructure/
│   ├── dnsmasq/
│   ├── nftables/
│   ├── opennds/
│   └── systemd/
├── scripts/
│   ├── diagnostics/
│   ├── backup/
│   └── rollback/
├── docs/
│   ├── discovery.md
│   ├── network-plan.md
│   ├── installation.md
│   ├── security.md
│   └── recovery.md
└── tests/
```

## 16. Instruksi eksekusi OpenCode

> Kamu adalah lead engineer proyek **ARIFIN GANTENGE POLL**. Baca seluruh `PRD.md` dan implementasikan sesuai spesifikasi. Ini sistem WiFi voucher gratis untuk rumah, bukan produk enterprise. Prioritaskan tampilan premium penuh animasi yang ringan, avatar orang ganteng yang dapat diganti admin, password gate **“Password-nya apa?”** dengan password awal `Arifinganteng` yang di-hash di server, katalog voucher publik, dan integrasi captive portal nyata. Mulai dengan Phase 0 read-only dan Phase 1 aplikasi. Jangan mengubah interface, Netplan, SSH, firewall, DHCP, routing, atau service aktif tanpa persetujuan eksplisit pemilik. Laporkan setiap fase: perubahan file, tes yang benar-benar dijalankan, hasil, hambatan, dan approval yang diperlukan. Jangan menyatakan integrasi gateway berhasil sebelum diuji pada perangkat nyata.
