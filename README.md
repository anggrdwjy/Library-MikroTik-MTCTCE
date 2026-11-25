# MTCTCE — Implementation and Operation Guide

## Modul 1 — Packet Flow Diagram

### 1.1 Tujuan
- Mampu menjawab pertanyaan berikut: Melalui chain mana paket tersebut melewati router? Pada tahap mana paket dikenai NAT? Pada tahap mana paket dikenai Mangle? Mengapa aturan yang dibuat tidak berfungsi?
- Mampu membedakan paket yang ditujukan kepada router itu sendiri (Input/Output) dengan paket yang hanya diteruskan (Forward).
- Mampu memprediksi urutan proses: Routing, Firewall, NAT, Mangle, dan Queue.

### 1.2 Konsep Kunci — Versi Sederhana

Paket dapat dianalogikan sebagai tamu yang memasuki sebuah gedung. Tamu tersebut tidak langsung menuju ke ruangan tujuan. Tamu tersebut melewati gerbang, menjalani pemeriksaan identitas, diarahkan ke lift, diperiksa kembali, kemudian tiba di tujuan. Pada RouterOS, alur tersebut dapat digambarkan sebagai berikut (versi ringkas yang paling sering digunakan untuk penelusuran masalah):

```text
[ Paket Masuk ]
  -> Prerouting (Mangle Prerouting, DST-NAT)
  -> Routing Decision: Menuju Router atau Diteruskan?
     A) MENUJU ROUTER: Input (Firewall Filter Input) -> Proses Lokal -> Output -> Postrouting
     B) DITERUSKAN: Forward (Firewall Filter Forward) -> Postrouting (SRC-NAT, Mangle Postrouting)
  -> [ Paket Keluar ]

Bridging (Layer 2) memiliki jalur tersendiri: Bridge Input -> Bridge Forward -> Bridge Output.
Apabila paket di-bridge (tidak di-route), paket tersebut TIDAK melewati IP Firewall, kecuali opsi Use IP Firewall pada Bridge Settings diaktifkan.
```

| Titik | Proses yang Terjadi pada Titik Tersebut |
|---|---|
| Prerouting | Mangle Prerouting dan DST-NAT. Pada tahap ini, paket ditandai dan tujuannya dialihkan (port forwarding, pengalihan ke proxy). |
| Input | Firewall Filter untuk paket yang ditujukan kepada router (WinBox, SSH, DNS ke router, VPN). |
| Forward | Firewall Filter untuk paket yang diteruskan (LAN ke Internet, antar VLAN). |
| Output | Firewall Filter untuk paket yang dibuat oleh router itu sendiri (query DNS dari router, ping dari router). |
| Postrouting | SRC-NAT / Masquerade. Pada tahap ini, alamat IP privat diubah menjadi alamat IP publik. Mangle Postrouting merupakan tahap terakhir. |

### 1.3 Implementasi — Penggunaan Diagram untuk Penelusuran Masalah

1. **Langkah 1 — Menentukan arah paket:** Apakah paket berasal dari LAN menuju Internet? Dari Internet menuju server? Dari pengguna menuju router? Gambarkan satu garis alur saja, tidak seluruh alur.
2. **Langkah 2 — Menandai chain yang dilalui:** Contoh LAN menuju Internet = Prerouting -> Forward -> Postrouting. Apabila aturan dipasang pada Input, wajar apabila aturan tersebut tidak berfungsi, karena paket tidak melewati Input.
3. **Langkah 3 — Memeriksa urutan NAT dan Filter:** DST-NAT terjadi pada Prerouting (sebelum Filter Forward). Oleh karena itu, pada Filter Forward, alamat tujuan yang diperiksa adalah alamat IP hasil NAT, bukan alamat IP publik aslinya. Hal ini merupakan kesalahan yang paling sering terjadi.
4. **Langkah 4 — Memeriksa Mangle:** Apabila queue tidak berfungsi, umumnya penandaan (marking) tidak berjalan. Periksa pada Mangle Prerouting / Forward, bukan hanya pada Queue.

### 1.4 Verifikasi dan Monitoring

```routeros
# Menampilkan paket yang berjalan secara real-time (pada router produksi, gunakan filter)
/tool sniffer set filter-interface=bridge-LAN filter-ip-address=192.168.10.100
/tool sniffer start
/tool sniffer stop
/tool sniffer packet print

# Memeriksa counter firewall — aturan mana yang menerima paket
/ip firewall filter print stats
/ip firewall nat print stats
/ip firewall mangle print stats

# Memeriksa koneksi yang aktif
/ip firewall connection print where src-address~"192.168.10.100"
```

### 1.5 Studi Kasus Modul 1

**Kasus: Port Forwarding Web Server Tidak Dapat Diakses dari Internet.**
Administrator telah membuat DST-NAT port 80 menuju server 192.168.10.10, tetapi juga membuat Filter Forward yang menolak seluruh trafik dari WAN. Dari Internet, pengguna tidak dapat membuka situs web. Apa penyebabnya?

**Analisis Packet Flow:** Paket dari Internet masuk -> Prerouting DST-NAT mengubah tujuan menjadi 192.168.10.10 -> Routing Decision -> Forward. Pada Forward, aturan penolakan dari WAN akan menghentikan paket, karena setelah NAT, paket tetap dianggap berasal dari WAN. Solusinya: Pada Filter Forward, izinkan alamat tujuan 192.168.10.10 dan dst-port 80, bukan alamat IP publik. Atau, dengan cara yang lebih rapi, izinkan connection-state=established,related dan dstnat.

```routeros
# Solusi yang benar — Filter Forward harus menggunakan alamat IP setelah NAT
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="Allow Established"
/ip firewall filter add chain=forward connection-nat-state=dstnat in-interface-list=WAN action=accept comment="Allow Port Forwarded"
/ip firewall filter add chain=forward in-interface-list=WAN action=drop comment="Drop sisa dari WAN"
```

> **Kesalahan yang Sering Terjadi pada Modul 1:** 1) Menempatkan aturan pada chain yang tidak sesuai. 2) Memeriksa dst-address sebelum NAT, padahal seharusnya setelah NAT. 3) Mengabaikan bahwa trafik Bridge tidak melewati IP Firewall, kecuali opsi Use IP Firewall diaktifkan. 4) Mengabaikan bahwa DNS menuju router termasuk dalam chain Input, bukan Forward.



### 1.6 Perbandingan RouterOS v6 dan v7 — Packet Flow

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| Diagram Packet Flow | Diagram klasik: Prerouting -> Input / Forward -> Output -> Postrouting | Diagram tetap sama pada dasarnya, dengan penegasan jalur Routing Decision yang lebih jelas dan pemisahan Bridge dan Routing |
| Bridge Firewall | `Use IP Firewall` pada Bridge Settings, dinonaktifkan secara default | Sama, `Use IP Firewall` tetap tersedia, dengan kinerja hardware offload yang lebih baik apabila diaktifkan |
| Connection Tracking | `/ip firewall connection tracking`, opsi `tcp-syncookies`, `enabled` | Opsi diperluas: `loose-tcp-tracking`, `tcp-established-timeout`, dan penanganan IPv6 yang lebih baik |
| Mangle dan NAT | Posisi Mangle dan NAT sama, sesuai diagram v6 | Posisi sama, tetapi Mangle pada v7 lebih efisien untuk trafik berkecepatan tinggi karena mesin routing baru |
| Kinerja | Cukup untuk router menengah, CPU menjadi beban pada Mangle yang padat | Lebih efisien untuk throughput tinggi, terutama pada perangkat ARM / ARM64 dan CCR |
| Rekomendasi | Tetap sesuai untuk jaringan yang sudah stabil dan tidak memerlukan fitur baru | Disarankan untuk instalasi baru dan untuk jaringan yang memerlukan throughput tinggi atau DoH / IPv6 |

#### Konfigurasi RouterOS v6 — Packet Flow

```routeros
# RouterOS v6 — Mengaktifkan IP Firewall pada Bridge (apabila diperlukan)
/interface bridge settings set use-ip-firewall=yes

# RouterOS v6 — Connection Tracking
/ip firewall connection tracking set enabled=yes tcp-syncookies=yes tcp-established-timeout=1d

# RouterOS v6 — Memeriksa alur paket
/ip firewall filter print stats
/ip firewall mangle print stats
```

#### Konfigurasi RouterOS v7 — Packet Flow

```routeros
# RouterOS v7 — Mengaktifkan IP Firewall pada Bridge (apabila diperlukan)
/interface bridge settings set use-ip-firewall=yes

# RouterOS v7 — Connection Tracking (opsi diperluas)
/ip firewall connection tracking set enabled=yes loose-tcp-tracking=yes tcp-established-timeout=1d tcp-syncookies=yes

# RouterOS v7 — Memeriksa alur paket (sama, dengan tambahan statistik IPv6)
/ip firewall filter print stats
/ip firewall mangle print stats
/ipv6 firewall filter print stats
```

> **Catatan Migrasi Modul 1:** Tidak terdapat perubahan perintah yang bersifat memutus (breaking change) pada Packet Flow antara v6 dan v7. Konfigurasi v6 dapat dipindahkan ke v7 dengan penyesuaian minimal, terutama pada Connection Tracking dan Bridge Settings. Disarankan untuk memverifikasi kembali bahwa trafik Bridge tidak melewati IP Firewall apabila opsi tersebut tidak diaktifkan.



## Modul 2 — Firewall Filter, NAT, Mangle

### 2.1 Tujuan
- Mengamankan router dan jaringan dengan Filter yang berlapis.
- Menerjemahkan alamat dengan NAT untuk akses Internet, server publik, dan pengalihan internal.
- Menandai koneksi dan paket dengan Mangle agar dapat digunakan untuk QoS, Policy Routing, dan monitoring.

### 2.2 Konsep Kunci

| Fitur | Fungsi Singkat |
|---|---|
| Connection Tracking | Router mencatat koneksi (TCP/UDP/ICMP). Fitur ini memungkinkan aturan established,related yang menjadikan firewall lebih cepat dan aman. |
| Filter | Allow / Drop / Reject. Urutan aturan berlaku dari atas ke bawah, aturan pertama yang cocok yang akan diterapkan. |
| NAT | Src-NAT / Masquerade (LAN ke Internet) dan Dst-NAT (Internet ke Server / Pengalihan). Helpers untuk FTP, SIP, PPTP. |
| Mangle | Mark Connection, Mark Packet, Mark Routing. Digunakan untuk QoS dan routing khusus. Penandaan harus hemat terhadap penggunaan CPU. |
| Address List, Interface List | Cara mengelompokkan alamat IP dan interface agar aturan tersusun secara rapi. |

### 2.3 Implementasi — Firewall Filter (Template Aman)

Urutan berikut telah teruji di lapangan. Urutan tersebut tidak disarankan untuk dibalik: Invalid terlebih dahulu, Established kedua, kemudian aturan spesifik, dan paling bawah Drop.

```routeros
# === FILTER INPUT — Melindungi Router ===
/ip firewall filter
add chain=input connection-state=invalid action=drop comment="Drop Invalid Input"
add chain=input connection-state=established,related,untracked action=accept comment="Allow Established Input"
add chain=input protocol=icmp action=accept comment="Allow Ping ke Router"
add chain=input in-interface-list=LAN action=accept comment="Allow LAN ke Router (WinBox, DNS)"
add chain=input in-interface-list=WAN action=drop comment="Drop WAN ke Router"

# === FILTER FORWARD — Melindungi Pengguna ===
add chain=forward connection-state=invalid action=drop comment="Drop Invalid Forward"
add chain=forward connection-state=established,related,untracked action=accept comment="Allow Established Forward"
add chain=forward connection-nat-state=dstnat action=accept comment="Allow Port Forward"
add chain=forward in-interface-list=LAN out-interface-list=WAN action=accept comment="Allow LAN ke Internet"
add chain=forward action=drop comment="Drop Sisa Forward"

# === Contoh Pemblokiran: Tamu tidak diizinkan mengakses Server Internal ===
/ip firewall filter add chain=forward src-address=192.168.10.200/29 dst-address=192.168.10.10 action=drop comment="Guest Block ke Server" place-before=3
```

### 2.4 Implementasi — NAT

```routeros
# === SRC-NAT: LAN ke Internet ===
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="Masquerade LAN ke Internet"

# === DST-NAT: Port Forwarding Web Server ===
/ip firewall nat add chain=dstnat in-interface-list=WAN dst-port=80,443 protocol=tcp action=dst-nat to-addresses=192.168.10.10 comment="Web Server"

# === DST-NAT: Pengalihan seluruh HTTP ke Proxy (lihat Modul 6) ===
/ip firewall nat add chain=dstnat in-interface-list=LAN dst-port=80 protocol=tcp action=redirect to-ports=8080 comment="Redirect ke Proxy"

# === NAT Helper — aktifkan hanya yang digunakan ===
/ip firewall service-port set ftp disabled=no sip disabled=no pptp disabled=no
```

### 2.5 Implementasi — Mangle (Penandaan untuk QoS)

**Prinsip Utama Mangle:** Tandai Connection terlebih dahulu satu kali, kemudian tandai Packet berdasarkan Connection-Mark. Tidak disarankan untuk langsung menandai Packet dari Prerouting untuk setiap paket, karena hal tersebut membebani CPU. Gunakan `passthrough=yes` hanya sampai penandaan selesai, kemudian `passthrough=no` agar tidak dilanjutkan ke aturan berikutnya yang tidak diperlukan.

```routeros
# === MANGLE — Menandai Game, Browsing, dan Download ===
/ip firewall mangle
# 1. Menandai koneksi Game (contoh port game)
add chain=prerouting protocol=udp dst-port=27015-27030 connection-mark=no-mark action=mark-connection new-connection-mark=conn-game passthrough=yes comment="Mark Conn Game"
add chain=prerouting protocol=tcp dst-port=80,443 connection-mark=no-mark action=mark-connection new-connection-mark=conn-web passthrough=yes comment="Mark Conn Web"

# 2. Menandai paket dari koneksi (yang digunakan oleh Queue Tree)
add chain=prerouting connection-mark=conn-game action=mark-packet new-packet-mark=pkt-game passthrough=no comment="Mark Packet Game"
add chain=prerouting connection-mark=conn-web action=mark-packet new-packet-mark=pkt-web passthrough=no comment="Mark Packet Web"

# 3. Contoh Mark Routing — Trafik Tamu melalui ISP2 (apabila terdapat 2 WAN)
# add chain=prerouting src-address=192.168.10.200/29 action=mark-routing new-routing-mark=via-ISP2 passthrough=no
```

### 2.6 Verifikasi dan Operasi

```routeros
# Memeriksa aturan mana yang paling banyak menerima paket
/ip firewall filter print stats
/ip firewall nat print stats
/ip firewall mangle print stats

# Memeriksa apakah connection tracking penuh atau tidak (penyebab router lambat)
/ip firewall connection print count-only
/ip firewall connection print where connection-mark=conn-game

# Log untuk aturan yang dicurigai (aktifkan sementara saja)
/ip firewall filter add chain=forward action=log log-prefix="DROP-FWD:" place-before=0
/log print where message~"DROP-FWD"
```

### 2.7 Studi Kasus Modul 2 — Kantor dengan Tiga Kebutuhan

**Skenario:** Kantor memiliki tiga kebutuhan: (1) Staf diizinkan mengakses Internet secara penuh, (2) Tamu WiFi hanya diizinkan untuk browsing, tidak diizinkan mengakses server internal dan tidak diizinkan mengunduh berkas berukuran besar, (3) Server Web harus dapat diakses dari Internet, tetapi tidak boleh diakses melalui SSH dari Internet.

**Solusi:** Gabungkan Filter, NAT, dan Mangle. Filter digunakan untuk isolasi Tamu, NAT untuk Server Web, dan Mangle untuk menandai Tamu agar dibatasi pada QoS (Modul 3).

```routeros
# Isolasi Tamu
/ip firewall address-list add list=Guest-IP address=192.168.10.200-192.168.10.220 comment="Guest WiFi"
/ip firewall filter add chain=forward src-address-list=Guest-IP dst-address=192.168.10.10 action=drop comment="Guest Block Server"
/ip firewall filter add chain=forward src-address-list=Guest-IP dst-port=22,8291 protocol=tcp action=drop comment="Guest Block SSH/WinBox"

# Menonaktifkan UPnP pada jaringan kantor (menghindari port terbuka secara otomatis)
/ip upnp set enabled=no
```

> **Peringatan Operasional — Firewall:** Tidak disarankan untuk membuat aturan `accept all` di atas aturan drop. Tidak disarankan untuk mengaktifkan `log` secara permanen pada aturan yang padat, karena disk dapat penuh dan CPU meningkat. Setiap perubahan firewall harus diuji dari tiga arah: dari LAN, dari WAN, dan menuju router itu sendiri.



### 2.8 Perbandingan RouterOS v6 dan v7 — Firewall, NAT, dan Mangle

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| Firewall Filter | `/ip firewall filter`, `connection-state`, `connection-nat-state`, urutan atas ke bawah | Sama, dengan penambahan opsi `connection-state` yang lebih lengkap dan dukungan IPv6 yang lebih terintegrasi |
| NAT | `srcnat`, `dstnat`, `masquerade`, `netmap`, `redirect` | Sama, dengan kinerja NAT yang lebih baik pada throughput tinggi |
| Mangle — Mark Connection / Mark Packet | `mark-connection`, `mark-packet`, `passthrough=yes/no` | Sama, dengan penekanan pada penandaan yang hemat CPU |
| Mangle — Mark Routing | `mark-routing` dengan `new-routing-mark` | `mark-routing` tetap didukung untuk kompatibilitas, tetapi disarankan menggunakan `new-routing-table` dan `/routing table` |
| FastTrack | `action=fasttrack-connection` pada Filter Forward | Sama, dengan FastTrack yang dioptimalkan untuk IPv6 dan koneksi yang lebih cepat |
| Address List dan Interface List | Tersedia, digunakan untuk menyusun aturan secara rapi | Sama, dengan penambahan `address-list` yang dapat diisi melalui DNS dan timeout dinamis |
| Rekomendasi | Sesuai untuk jaringan yang sudah berjalan dengan Mark Routing lama | Untuk instalasi baru, gunakan Routing Table pada v7 agar konfigurasi sesuai dengan arsitektur routing terbaru |

#### Konfigurasi RouterOS v6 — Firewall, NAT, dan Mangle

```routeros
# RouterOS v6 — Filter dan NAT
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="Allow Established v6"
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="Masquerade v6"

# RouterOS v6 — Mangle Mark Routing (cara lama)
/ip firewall mangle add chain=prerouting src-address=192.168.10.200/29 action=mark-routing new-routing-mark=via-ISP2 passthrough=no comment="Mark Routing v6"
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.2 routing-mark=via-ISP2 comment="Route via ISP2 v6"
```

#### Konfigurasi RouterOS v7 — Firewall, NAT, dan Mangle

```routeros
# RouterOS v7 — Filter dan NAT (sintaks tetap sama)
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="Allow Established v7"
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="Masquerade v7"

# RouterOS v7 — Mangle Mark Routing (cara baru, disarankan)
/routing table add name=via-ISP2 fib comment="Routing Table v7"
/ip firewall mangle add chain=prerouting src-address=192.168.10.200/29 action=mark-routing new-routing-table=via-ISP2 passthrough=no comment="Mark Routing v7"
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.2 routing-table=via-ISP2 comment="Route via ISP2 v7"

# RouterOS v7 — Kompatibilitas: routing-mark lama masih dapat digunakan, tetapi akan dikonversi ke routing-table
```

> **Catatan Migrasi Modul 2:** Perintah Filter dan NAT pada v6 dapat digunakan kembali pada v7 tanpa perubahan. Perbedaan utama terdapat pada Mark Routing. Apabila konfigurasi v6 menggunakan `routing-mark`, pada saat migrasi ke v7, disarankan untuk mengubahnya menjadi `routing-table` dan membuat entri pada `/routing table` agar routing tetap berfungsi dan mudah dikelola.


## Modul 3 — Quality of Service (QoS)

### 3.1 Tujuan
- Menjamin aplikasi penting (Zoom, VoIP, aplikasi kantor) tetap berjalan dengan lancar meskipun jaringan sedang sibuk.
- Membagi bandwidth secara adil per pengguna atau per jenis trafik, bukan berdasarkan siapa yang lebih dahulu mengakses.
- Menggunakan HTB, Queue Tree, Simple Queue, dan PCQ secara tepat, bukan sekadar dipasang.

### 3.2 Konsep Kunci — HTB dan Queue

| Istilah | Penjelasan dalam Bahasa Operasional |
|---|---|
| HTB (Hierarchical Token Bucket) | Sistem antrean bertingkat. Parent membagi bandwidth kepada Child. Child dapat meminjam sisa bandwidth Parent apabila Parent masih memiliki kapasitas. |
| Limit-at / Max-limit | Limit-at = jaminan minimal. Max-limit = batas maksimal. Child tidak boleh melebihi max-limit Parent. |
| Priority | 1 = paling penting, 8 = paling tidak penting. Digunakan saat terjadi perebutan bandwidth. |
| Burst | Kecepatan tambahan sesaat. Cocok untuk browsing yang membutuhkan kecepatan tinggi dalam waktu singkat, bukan untuk unduhan yang berlangsung terus-menerus. |
| FIFO / SFQ / RED / PCQ | FIFO = antrean lurus. SFQ = antrean adil per aliran. RED = membuang paket lebih awal agar TCP melambat secara halus. PCQ = antrean adil per pengguna / alamat IP, paling sering digunakan untuk warnet / kantor. |
| Queue Tree vs Simple Queue | Queue Tree = fleksibel, menggunakan packet-mark, dapat bertingkat. Simple Queue = sederhana, target per alamat IP / subnet, tidak dapat menggunakan packet-mark sekompleks Queue Tree. Kedua jenis queue tersebut tidak disarankan untuk digunakan bersamaan pada trafik yang sama, karena akan menyebabkan pembatasan ganda. |

### 3.3 Desain QoS untuk Kantor 50 Mbps — Studi Kasus Utama

**Skenario:** Total bandwidth ISP 50 Mbps download / 20 Mbps upload. Jumlah pengguna 50 orang. Keluhan yang diterima: Saat terdapat pengguna yang mengunduh berkas, Zoom mengalami gangguan. Saat rapat berlangsung, browsing menjadi lambat. Permintaan yang diajukan: Zoom prioritas 1, Browsing prioritas 3, Download prioritas 7, Tamu dibatasi total 5 Mbps.

| Jenis Trafik | Packet-Mark (dari Modul 2) | Limit-at | Max-limit | Priority |
|---|---|---|---|---|
| Game / Zoom / VoIP | pkt-game, pkt-zoom | 10M | 30M | 1 (Paling Penting) |
| Browsing / Web | pkt-web | 10M | 45M | 3 |
| Download / Update | pkt-download | 2M | 20M | 7 (Paling Rendah) |
| Tamu WiFi | pkt-guest | 1M | 5M | 8 |

```routeros
# === 1. Queue Type PCQ — Pembagian rata per pengguna ===
/queue type add name=PCQ-Download kind=pcq pcq-rate=5M pcq-classifier=dst-address pcq-total-limit=2000 comment="PCQ Download per User"
/queue type add name=PCQ-Upload kind=pcq pcq-rate=2M pcq-classifier=src-address comment="PCQ Upload per User"

# === 2. Queue Tree — Download (Parent Global) ===
/queue tree
add name=DOWNLOAD parent=global queue=default max-limit=50M comment="Total Download ISP"
add name=DL-Zoom parent=DOWNLOAD packet-mark=pkt-zoom queue=default limit-at=10M max-limit=30M priority=1 comment="Zoom Prioritas 1"
add name=DL-Web parent=DOWNLOAD packet-mark=pkt-web queue=default limit-at=10M max-limit=45M priority=3 comment="Web Prioritas 3"
add name=DL-Download parent=DOWNLOAD packet-mark=pkt-download queue=PCQ-Download limit-at=2M max-limit=20M priority=7 comment="Download PCQ"
add name=DL-Guest parent=DOWNLOAD packet-mark=pkt-guest queue=PCQ-Download limit-at=1M max-limit=5M priority=8 comment="Guest Dibatasi"

# === 3. Queue Tree — Upload ===
add name=UPLOAD parent=global queue=default max-limit=20M comment="Total Upload ISP"
add name=UL-Zoom parent=UPLOAD packet-mark=pkt-zoom queue=default limit-at=5M max-limit=15M priority=1
add name=UL-Web parent=UPLOAD packet-mark=pkt-web queue=default limit-at=5M max-limit=18M priority=3

# === 4. Burst — Browsing diizinkan melampaui batas selama 5 detik pertama ===
/queue tree set DL-Web burst-limit=45M burst-threshold=20M burst-time=5s
```

### 3.4 Simple Queue — Alternatif Cepat untuk Kasus Sederhana

Apabila hanya diperlukan pembatasan per pengguna tanpa penandaan yang rumit, Simple Queue lebih cepat untuk dibuat. Contoh: Setiap staf maksimal 5 Mbps, Tamu maksimal 2 Mbps. Perlu diperhatikan: Simple Queue dan Queue Tree tidak disarankan untuk diaktifkan bersamaan pada alamat IP yang sama.

```routeros
# Simple Queue — Pembatasan per pengguna / per subnet
/queue simple
add name=Staff-100 target=192.168.10.100/32 max-limit=5M/5M comment="Staff 100"
add name=Guest-Subnet target=192.168.10.200/29 max-limit=2M/2M queue=PCQ-Download/PCQ-Upload comment="Guest Subnet"
add name=Server target=192.168.10.10/32 max-limit=20M/20M priority=1/1 comment="Server Prioritas"
```

### 3.5 Verifikasi dan Operasi QoS

```routeros
# Melihat queue mana yang penuh (rate mendekati max-limit)
/queue tree print stats
/queue simple print stats

# Melihat trafik per packet-mark secara real-time
/interface monitor-traffic bridge-LAN,ether1

# Memeriksa apakah penandaan berjalan — apabila packet-mark bernilai 0, queue tidak akan berfungsi
/ip firewall mangle print stats where packet-mark~"pkt-"

# Pengujian: Unduh berkas berukuran besar pada 1 PC, kemudian jalankan Zoom pada PC lain — Zoom harus tetap berjalan dengan lancar
# Apabila Zoom tetap mengalami gangguan, periksa: Apakah Priority sudah 1? Apakah Limit-at Parent sudah cukup? Apakah Burst terlalu besar?
```

> **Kesalahan yang Sering Terjadi pada QoS:** 1) Parent max-limit lebih kecil daripada total child limit-at — child tidak akan pernah memperoleh jaminannya. 2) Kesalahan penandaan upload dan download — queue download menggunakan dst-address, upload menggunakan src-address pada PCQ. 3) Memasang Simple Queue dan Queue Tree secara bersamaan — trafik dikenai pembatasan dua kali. 4) Burst threshold terlalu rendah — burst tidak pernah aktif.


### 3.6 Perbandingan RouterOS v6 dan v7 — QoS

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| Queue Tree | HTB, `parent=global`, `packet-mark`, `limit-at`, `max-limit`, `priority`, `burst` | Sama, dengan kinerja HTB yang lebih baik pada CPU multi-core |
| Simple Queue | `target`, `max-limit`, `queue` | Sama, dengan penambahan opsi `bucket-size` dan penanganan trafik yang lebih akurat |
| Queue Type | FIFO, SFQ, RED, PCQ | Sama, ditambah `CAKE` dan `FQ-CoDel` untuk mengatasi bufferbloat |
| PCQ | `pcq-classifier=dst-address / src-address`, `pcq-rate`, `pcq-total-limit` | Sama, dengan parameter PCQ yang diperluas dan penanganan per-pengguna yang lebih stabil |
| Burst | `burst-limit`, `burst-threshold`, `burst-time` | Sama |
| Bufferbloat | Terbatas, menggunakan RED / SFQ | Lebih baik, menggunakan CAKE / FQ-CoDel yang dirancang untuk latensi rendah |
| Rekomendasi | Sesuai untuk pembagian bandwidth dasar dengan PCQ dan HTB | Untuk jaringan yang memerlukan latensi rendah (Zoom, game, VoIP), gunakan CAKE pada v7 |

#### Konfigurasi RouterOS v6 — QoS

```routeros
# RouterOS v6 — Queue Type PCQ dan Queue Tree (HTB)
/queue type add name=PCQ-Download-v6 kind=pcq pcq-rate=5M pcq-classifier=dst-address pcq-total-limit=2000
/queue tree add name=DOWNLOAD-v6 parent=global queue=default max-limit=50M comment="Total Download v6"
/queue tree add name=DL-Zoom-v6 parent=DOWNLOAD-v6 packet-mark=pkt-zoom queue=default limit-at=10M max-limit=30M priority=1 comment="Zoom v6"
```

#### Konfigurasi RouterOS v7 — QoS

```routeros
# RouterOS v7 — Queue Type PCQ (tetap sama) dan CAKE (baru)
/queue type add name=PCQ-Download-v7 kind=pcq pcq-rate=5M pcq-classifier=dst-address pcq-total-limit=2000
/queue type add name=CAKE-Download-v7 kind=cake cake-bandwidth=50M cake-diffserv4=yes comment="CAKE untuk Latensi Rendah v7"
/queue type add name=FQ-CoDel-v7 kind=fq-codel fq-codel-quantum=1514 comment="FQ-CoDel v7"

# RouterOS v7 — Queue Tree HTB (sama dengan v6)
/queue tree add name=DOWNLOAD-v7 parent=global queue=default max-limit=50M comment="Total Download v7"
/queue tree add name=DL-Zoom-v7 parent=DOWNLOAD-v7 packet-mark=pkt-zoom queue=default limit-at=10M max-limit=30M priority=1 comment="Zoom v7"

# RouterOS v7 — Contoh penggunaan CAKE pada Simple Queue (alternatif untuk mengatasi bufferbloat)
/queue simple add name=Cake-Simple-v7 target=192.168.10.0/24 max-limit=50M/20M queue=CAKE-Download-v7/CAKE-Download-v7 comment="Simple Queue dengan CAKE v7"
```

> **Catatan Migrasi Modul 3:** Konfigurasi Queue Tree dan Simple Queue dari v6 dapat digunakan pada v7 tanpa perubahan. Apabila jaringan mengalami keluhan latensi (Zoom terputus, game mengalami jeda), disarankan untuk menguji Queue Type CAKE atau FQ-CoDel pada v7 sebagai pengganti FIFO / SFQ, kemudian mengukur kembali latensi dan jitter sebelum diterapkan secara permanen.


## Modul 4 — DNS Client, Cache

### 4.1 Tujuan
- Router dapat melakukan resolusi nama domain dengan cepat dan andal.
- Klien LAN dapat menggunakan router sebagai DNS cache, sehingga query ke Internet berkurang dan browsing menjadi lebih cepat.
- Administrator dapat membuat entri DNS statis untuk server internal dan untuk penyaringan sederhana.

### 4.2 Implementasi

```routeros
# === DNS Client & Cache ===
/ip dns
set servers=8.8.8.8,1.1.1.1,8.8.4.4 allow-remote-requests=yes cache-size=4096KiB cache-max-ttl=1d max-concurrent-queries=200

# Wajib: Klien LAN harus diizinkan menggunakan DNS router (melalui Filter Input)
/ip firewall filter add chain=input in-interface-list=LAN protocol=udp dst-port=53 action=accept comment="Allow DNS LAN"

# === Static DNS — Server Internal & Pemblokiran Iklan / Situs ===
/ip dns static
add name=server.kantor.local address=192.168.10.10 comment="Server Internal"
add name=app.kantor.local address=192.168.10.10 comment="Aplikasi Internal"
add name=ads.contoh-iklan.com address=127.0.0.1 comment="Blokir Iklan — arahkan ke localhost"

# === DHCP Network harus mengarahkan DNS ke Router ===
/ip dhcp-server network set [find address=192.168.10.0/24] dns-server=192.168.10.1
```

### 4.3 Verifikasi dan Operasi

```routeros
# Memeriksa apakah cache berfungsi — harus terdapat banyak entri setelah browsing
/ip dns cache print count-only
/ip dns cache print where name~"google"

# Pengujian dari klien
# Windows: nslookup server.kantor.local 192.168.10.1

# Flush apabila terdapat domain yang berubah alamat IP-nya
/ip dns cache flush

# Monitoring: Apabila cache hit rendah, periksa apakah klien benar-benar menggunakan DNS router
/ip firewall filter print stats where dst-port=53
```

### 4.4 Studi Kasus Modul 4 — Sekolah / Kantor

**Skenario:** Sekolah menginginkan seluruh siswa yang mengetik `portal.sekolah.local` langsung diarahkan ke server ujian 192.168.10.10, tanpa perlu mengatur hosts pada setiap PC. Sekolah juga ingin mengurangi beban query DNS ke Internet saat 200 siswa membuka Google secara bersamaan.

**Solusi:** Aktifkan DNS Cache pada router, jadikan router sebagai DNS utama pada DHCP, dan buat Static DNS untuk portal.sekolah.local. Hasilnya: Pembukaan portal menjadi lebih cepat, query ke Internet menurun karena telah disimpan dalam cache. Untuk penyaringan yang lebih kuat, gabungkan dengan Web Proxy pada Modul 6, bukan hanya Static DNS.

### 4.5 Perbandingan RouterOS v6 dan v7 — DNS Client dan Cache

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| DNS Client | `/ip dns set servers=...`, `allow-remote-requests` | Sama, dengan penambahan dukungan DoH (DNS over HTTPS) |
| DNS Cache | `cache-size`, `cache-max-ttl`, `max-concurrent-queries` | Sama, dengan `cache-size` dan `max-concurrent-queries` yang lebih besar pada perangkat dengan memori besar |
| Static DNS | `/ip dns static add name=... address=...` hanya untuk tipe A | Diperluas: `type=A / AAAA / CNAME / TXT / SRV / MX`, `match-subdomain`, `ttl`, `forward-to` |
| DoH (DNS over HTTPS) | Tidak tersedia secara bawaan | Tersedia: `use-doh-server`, `verify-doh-cert`, `doh-max-server-connections` |
| AdList / Penyaringan Iklan | Tidak tersedia, menggunakan Static DNS manual | Tersedia: `/ip dns adlist` untuk memblokir domain iklan dan pelacak secara otomatis |
| Rekomendasi | Sesuai untuk DNS Cache dasar dan Static DNS internal | Untuk keamanan dan privasi yang lebih baik, gunakan DoH dan AdList pada v7 |

#### Konfigurasi RouterOS v6 — DNS

```routeros
# RouterOS v6 — DNS Client dan Cache
/ip dns set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes cache-size=4096KiB cache-max-ttl=1d max-concurrent-queries=200

# RouterOS v6 — Static DNS (hanya A record)
/ip dns static add name=server.kantor.local address=192.168.10.10 comment="Server Internal v6"
```

#### Konfigurasi RouterOS v7 — DNS

```routeros
# RouterOS v7 — DNS Client, Cache, dan DoH
/ip dns set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes cache-size=4096KiB cache-max-ttl=1d max-concurrent-queries=200 use-doh-server=https://cloudflare-dns.com/dns-query verify-doh-cert=yes comment="DNS v7 dengan DoH"

# RouterOS v7 — Static DNS (diperluas: tipe, match-subdomain, TTL)
/ip dns static add name=server.kantor.local address=192.168.10.10 type=A match-subdomain=yes ttl=1d comment="Server Internal v7"
/ip dns static add name=app.kantor.local address=server.kantor.local type=CNAME ttl=1d comment="Alias CNAME v7"

# RouterOS v7 — AdList (baru, untuk pemblokiran iklan)
/ip dns adlist add url=https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts ssl-verify=yes comment="AdList v7"
```

> **Catatan Migrasi Modul 4:** Konfigurasi DNS dasar v6 dapat dipindahkan ke v7 tanpa perubahan. Apabila DoH diaktifkan, pastikan sertifikat terverifikasi dan Firewall mengizinkan trafik TCP 443 ke server DoH. Apabila AdList diaktifkan, pantau penggunaan memori, karena daftar domain yang besar memerlukan RAM yang cukup.


## Modul 5 — DHCP Client, Relay, Server

### 5.1 Tujuan
- Memahami alur DHCP: Discover - Offer - Request - Ack.
- Mengonfigurasi DHCP Client (untuk WAN dari ISP), DHCP Server (untuk LAN), dan DHCP Relay (untuk cabang / VLAN yang jauh).
- Mengatur Pool, Lease, Option, dan pengecualian untuk perangkat penting.

### 5.2 Implementasi — DHCP Server Kantor

```routeros
# === 1. IP Pool — Pisahkan per lantai / per fungsi ===
/ip pool
add name=pool-staff ranges=192.168.10.100-192.168.10.150 comment="Staff"
add name=pool-guest ranges=192.168.10.200-192.168.10.220 comment="Guest WiFi"
add name=pool-printer ranges=192.168.10.50-192.168.10.60 comment="Printer"

# === 2. DHCP Server ===
/ip dhcp-server
add name=dhcp-LAN interface=bridge-LAN address-pool=pool-staff lease-time=8h authoritative=yes disabled=no comment="DHCP Kantor"

# === 3. DHCP Network — Gateway, DNS, NTP ===
/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 ntp-server=192.168.10.1 comment="Network Kantor"

# === 4. Lease Statis — Server & Printer harus selalu menggunakan alamat IP yang sama ===
/ip dhcp-server lease
add address=192.168.10.10 mac-address=AA:BB:CC:DD:EE:01 server=dhcp-LAN comment="Server Tetap"
add address=192.168.10.50 mac-address=AA:BB:CC:DD:EE:02 server=dhcp-LAN comment="Printer Tetap"

# === 5. Option Custom — Contoh: TFTP / VoIP Phone ===
/ip dhcp-server option add name=opt-tftp code=66 value="'192.168.10.10'" comment="TFTP Server untuk IP Phone"
/ip dhcp-server network set [find address=192.168.10.0/24] dhcp-option=opt-tftp
```

### 5.3 Implementasi — DHCP Client dan Relay

```routeros
# === DHCP Client — WAN dari ISP (apabila ISP menggunakan DHCP) ===
/ip dhcp-client add interface=ether1 use-peer-dns=no use-peer-ntp=no add-default-route=yes comment="WAN DHCP ISP"

# === DHCP Relay — Cabang tanpa DHCP Server sendiri ===
# Topologi: Cabang 192.168.20.0/24 -> Relay pada Router Cabang -> Server pada Pusat 192.168.10.1
/ip dhcp-relay add name=relay-cabang interface=bridge-Cabang dhcp-server=192.168.10.1 local-address=192.168.20.1 disabled=no comment="Relay ke Pusat"

# Pada Pusat, DHCP Server harus mengenali subnet cabang
/ip pool add name=pool-cabang ranges=192.168.20.100-192.168.20.200
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.10.1
```

### 5.4 Verifikasi dan Penanganan Masalah

```routeros
# Melihat siapa yang memperoleh alamat IP, dari mana, dan sisa lease
/ip dhcp-server lease print
/ip dhcp-server lease print where status=bound

# Memeriksa konflik — apakah terdapat 2 DHCP Server pada jaringan?
/ip dhcp-server alert add interface=bridge-LAN on-alert="/log warning message=\"Rogue DHCP Terdeteksi!\"" valid-server=AA:BB:CC:DD:EE:FF

# Apabila klien tidak memperoleh alamat IP: periksa 1) Apakah DHCP Server disabled? 2) Apakah Pool habis? 3) Apakah Firewall memblokir UDP 67-68?
/ip firewall filter print where dst-port=67
/ip pool print

# Melepas lease yang bermasalah
/ip dhcp-server lease remove [find address=192.168.10.105]
```

### 5.5 Studi Kasus Modul 5 — Warnet / Coworking

**Skenario:** Coworking memiliki 80 kursi, tetapi sering kehabisan alamat IP karena lease-time terlalu lama (3 hari). Pengunjung yang sudah meninggalkan lokasi masih memegang alamat IP, sehingga pengunjung baru tidak memperoleh alamat IP.

**Solusi:** Ubah lease-time Tamu menjadi 1 jam, perbesar pool Tamu, dan aktifkan DHCP Alert untuk mendeteksi adanya pengunjung yang menghubungkan router pribadi yang menjadi DHCP liar. Setelah perubahan tersebut, keluhan 'tidak memperoleh alamat IP' tidak lagi muncul. Untuk perangkat penting seperti printer dan CCTV, gunakan Lease Statis agar alamat IP tidak berubah dan tidak mengalami perebutan.



### 5.6 Perbandingan RouterOS v6 dan v7 — DHCP Client, Relay, dan Server

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| DHCP Server | `/ip dhcp-server`, `address-pool`, `lease-time`, `authoritative` | Sama, dengan penambahan `lease-script`, `radius`, dan `option sets` yang lebih fleksibel |
| DHCP Pool | `/ip pool ranges=...` | Sama, dengan dukungan `next-pool` untuk failover pool |
| DHCP Network / Option | `/ip dhcp-server network`, `/ip dhcp-server option` | Sama, ditambah `option sets` dan `matcher` untuk perangkat khusus (VoIP, printer, IoT) |
| DHCP Lease | `/ip dhcp-server lease`, `make-static`, `blocked` | Sama, dengan penambahan `address-lists` dan `mac-address` yang lebih fleksibel |
| DHCP Client | `/ip dhcp-client`, `use-peer-dns`, `add-default-route` | Sama, dengan penambahan `script` dan `use-peer-ntp` yang lebih lengkap |
| DHCP Relay | `/ip dhcp-relay`, `dhcp-server`, `local-address` | Sama, dengan `delay-threshold` dan penanganan Relay Agent Information yang lebih baik |
| DHCPv6 | Terbatas, melalui `/ipv6 dhcp-server` yang sederhana | Diperluas: `/ipv6 dhcp-server`, `pool`, `binding`, dan `option` untuk IPv6 |
| Rekomendasi | Sesuai untuk DHCP IPv4 dasar | Untuk jaringan yang menggunakan IPv6 atau memerlukan Option yang kompleks, gunakan v7 |

#### Konfigurasi RouterOS v6 — DHCP

```routeros
# RouterOS v6 — DHCP Server dan Pool
/ip pool add name=pool-staff-v6 ranges=192.168.10.100-192.168.10.150 comment="Pool Staff v6"
/ip dhcp-server add name=dhcp-LAN-v6 interface=bridge-LAN address-pool=pool-staff-v6 lease-time=8h authoritative=yes disabled=no comment="DHCP v6"
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 comment="Network v6"
/ip dhcp-server option add name=opt-tftp-v6 code=66 value="'192.168.10.10'" comment="TFTP v6"
```

#### Konfigurasi RouterOS v7 — DHCP

```routeros
# RouterOS v7 — DHCP Server dan Pool (sama, dengan next-pool dan lease-script)
/ip pool add name=pool-staff-v7 ranges=192.168.10.100-192.168.10.150 next-pool=pool-guest-v7 comment="Pool Staff v7 dengan Failover"
/ip pool add name=pool-guest-v7 ranges=192.168.10.200-192.168.10.220 comment="Pool Guest v7"
/ip dhcp-server add name=dhcp-LAN-v7 interface=bridge-LAN address-pool=pool-staff-v7 lease-time=8h authoritative=yes disabled=no lease-script=":log info (\"Lease baru: \" . \$leaseActIP)" comment="DHCP v7 dengan Lease Script"

# RouterOS v7 — DHCP Network dan Option Sets (baru)
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 comment="Network v7"
/ip dhcp-server option add name=opt-tftp-v7 code=66 value="'192.168.10.10'" comment="TFTP v7"
/ip dhcp-server option sets add name=set-voip-v7 options=opt-tftp-v7 comment="Option Set VoIP v7"

# RouterOS v7 — DHCPv6 (diperluas)
/ipv6 pool add name=pool-v6-v7 prefix=2001:db8:10::/64 prefix-length=64 comment="Pool IPv6 v7"
/ipv6 dhcp-server add name=dhcpv6-LAN-v7 interface=bridge-LAN address-pool=pool-v6-v7 lease-time=8h disabled=no comment="DHCPv6 v7"
```

> **Catatan Migrasi Modul 5:** Konfigurasi DHCP IPv4 v6 dapat dipindahkan ke v7 tanpa perubahan mendasar. Apabila menggunakan Option yang kompleks atau DHCPv6, disarankan untuk menguji Option Sets pada v7 di laboratorium terlebih dahulu, karena sintaks Option Sets berbeda dari Option tunggal pada v6. Pastikan tidak terdapat dua DHCP Server yang aktif pada satu segmen yang sama setelah migrasi.


## Modul 6 — Web Proxy

### 6.1 Tujuan
- Mengontrol siapa yang diizinkan membuka apa, melalui Access List dan Direct Access List.
- Menghemat bandwidth dengan Cache List untuk berkas yang sering diunduh.
- Menggunakan Regular Expression untuk aturan yang fleksibel.

> **Catatan penting operasional:** Pada era HTTPS, Web Proxy MikroTik paling efektif untuk HTTP, pengalihan, cache berkas, dan penyaringan berbasis domain / alamat IP. Untuk penyaringan HTTPS yang mendalam, gabungkan dengan firewall dan penyaringan DNS, bukan hanya proxy.

### 6.2 Implementasi — Proxy Dasar dan Pengalihan

```routeros
# === Mengaktifkan Proxy ===
/ip proxy set enabled=yes port=8080 cache-on-disk=yes cache-administrator=admin@kantor.local max-cache-size=10G max-cache-object-size=100M cache-path=disk1/proxy

# === Mengalihkan seluruh HTTP dari LAN ke Proxy (Transparent Proxy) ===
/ip firewall nat add chain=dstnat in-interface-list=LAN dst-port=80 protocol=tcp action=redirect to-ports=8080 comment="Transparent Proxy HTTP"

# === Mengizinkan klien LAN menggunakan Proxy (Filter Input) ===
/ip firewall filter add chain=input in-interface-list=LAN protocol=tcp dst-port=8080 action=accept comment="Allow Proxy LAN"
```

### 6.3 Implementasi — Access, Direct, Cache List

```routeros
# === Access List — Siapa yang diizinkan / tidak diizinkan ===
/ip proxy access
add dst-host=:facebook.com action=deny redirect-to=192.168.10.10/block.html comment="Block Facebook Jam Kerja"
add dst-host=:youtube.com action=deny comment="Block Youtube Jam Kerja"
add src-address=192.168.10.10 action=allow comment="Server Boleh Semua"
add action=allow comment="Allow Sisa"

# === Direct Access List — Yang tidak melalui Cache (Perbankan, dan sebagainya) ===
/ip proxy direct
add dst-host=:bank-*.co.id action=deny comment="Banking Jangan Di-Cache"
add dst-host=:*.go.id action=deny comment="Situs Pemerintah Langsung"

# === Cache List — Yang wajib disimpan dalam cache (Update Windows, Berkas Kantor) ===
/ip proxy cache
add dst-host=:*.microsoft.com action=allow comment="Cache Update Windows"
add dst-host=:kantor.local action=allow comment="Cache File Kantor"

# === Regular Expression — Pola Fleksibel ===
# Contoh: Memblokir seluruh URL yang mengandung kata judi
/ip proxy access add dst-host=":.*judi.*" action=deny comment="Block Pola Judi via Regex"
```

### 6.4 Verifikasi dan Operasi

```routeros
# Memeriksa apakah proxy aktif, hit rate, dan cache
/ip proxy monitor
/ip proxy cache print count-only
/ip proxy access print

# Pengujian dari klien: buka http://example.com — harus melalui proxy (periksa pada monitor)
/ip firewall nat print stats where to-ports=8080

# Apabila situs web tidak dapat dibuka: periksa 1) Apakah Proxy enabled? 2) Apakah NAT redirect sudah benar? 3) Apakah Access List terdapat deny di atas allow?
# Apabila cache penuh: /ip proxy cache clear atau tambah disk
```

### 6.5 Studi Kasus Modul 6 — Sekolah

**Skenario:** Sekolah menginginkan: (1) Pada jam belajar, siswa tidak dapat membuka media sosial dan game, (2) Guru tetap diizinkan, (3) Pembaruan Windows pada 100 PC tidak menyebabkan Internet lambat karena unduhan berulang, (4) Situs ujian tidak boleh disimpan dalam cache agar selalu menampilkan versi terbaru.

**Solusi:** Buat dua kelompok alamat IP: Siswa (192.168.10.100–150) dan Guru (192.168.10.10–20). Pada Access List, tolak media sosial untuk alamat IP siswa, izinkan untuk guru. Pada Cache List, izinkan *.microsoft.com agar pembaruan Windows cukup diunduh satu kali kemudian disimpan dalam cache. Pada Direct List, tolak situs ujian agar selalu langsung menuju server asli. Hasilnya: Bandwidth dapat dihemat sebesar 30–40% pada saat pembaruan massal, dan guru tidak mengajukan keberatan karena aksesnya tetap bebas.


### 6.6 Perbandingan RouterOS v6 dan v7 — Web Proxy

| Aspek | RouterOS v6 | RouterOS v7 |
|---|---|---|
| Proxy Engine | `/ip proxy`, `enabled`, `port=8080`, `cache-on-disk`, `max-cache-size` | Sama, dengan kinerja proxy yang sedikit lebih baik pada v7, tetapi tetap tidak dirancang untuk HTTPS yang mendalam |
| Cache | `cache-path`, `cache-administrator`, `max-cache-object-size` | Sama, dengan `cache-path` yang lebih fleksibel terhadap jenis disk (SATA / NVMe / USB) |
| Access List | `/ip proxy access`, `dst-host`, `action=allow / deny`, `redirect-to` | Sama, dengan dukungan regex yang lebih konsisten |
| Direct / Cache List | `/ip proxy direct`, `/ip proxy cache` | Sama |
| HTTPS | Terbatas, hanya untuk HTTP dan pengalihan, tidak dapat memeriksa isi HTTPS yang terenkripsi | Sama, tetap terbatas. Untuk penyaringan HTTPS, disarankan menggunakan Firewall + DNS + DoH pada v7 |
| Rekomendasi | Sesuai untuk cache HTTP dan penyaringan dasar pada jaringan kecil | Untuk jaringan modern yang didominasi HTTPS, gunakan proxy sebagai pelengkap, bukan sebagai satu-satunya penyaring. Kombinasikan dengan Firewall, DNS Filter, dan AdList |

#### Konfigurasi RouterOS v6 — Web Proxy

```routeros
# RouterOS v6 — Proxy Dasar
/ip proxy set enabled=yes port=8080 cache-on-disk=yes cache-administrator=admin@kantor.local max-cache-size=10G max-cache-object-size=100M cache-path=disk1/proxy comment="Proxy v6"

# RouterOS v6 — Redirect dan Access
/ip firewall nat add chain=dstnat in-interface-list=LAN dst-port=80 protocol=tcp action=redirect to-ports=8080 comment="Redirect v6"
/ip proxy access add dst-host=:facebook.com action=deny redirect-to=192.168.10.10/block.html comment="Block Facebook v6"
```

#### Konfigurasi RouterOS v7 — Web Proxy

```routeros
# RouterOS v7 — Proxy Dasar (sintaks sama, cache-path disesuaikan dengan disk)
/ip proxy set enabled=yes port=8080 cache-on-disk=yes cache-administrator=admin@kantor.local max-cache-size=10G max-cache-object-size=100M cache-path=disk1/proxy anonymous=yes comment="Proxy v7"

# RouterOS v7 — Redirect dan Access (sama)
/ip firewall nat add chain=dstnat in-interface-list=LAN dst-port=80 protocol=tcp action=redirect to-ports=8080 comment="Redirect v7"
/ip proxy access add dst-host=:facebook.com action=deny redirect-to=192.168.10.10/block.html comment="Block Facebook v7"

# RouterOS v7 — Pelengkap untuk HTTPS (disarankan, karena proxy tidak memeriksa HTTPS)
/ip dns adlist add url=https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts ssl-verify=yes comment="Pelengkap HTTPS via DNS v7"
/ip firewall filter add chain=forward tls-host=*facebook.com action=drop comment="Block via TLS-Host v7 (alternatif)"
```

> **Catatan Migrasi Modul 6:** Konfigurasi Web Proxy v6 dapat dipindahkan ke v7 tanpa perubahan. Namun, karena mayoritas trafik saat ini menggunakan HTTPS, efektivitas proxy sebagai penyaring utama menurun pada kedua versi. Disarankan untuk tidak mengandalkan proxy saja. Gunakan kombinasi: Proxy untuk cache HTTP, Firewall Filter dengan `tls-host` untuk penyaringan HTTPS dasar, dan DNS / DoH / AdList pada v7 untuk penyaringan berbasis domain.


### Perintah Darurat — Apabila Jaringan Tidak Berfungsi Setelah Perubahan

```routeros
# 1. Menonaktifkan aturan firewall terakhir yang dicurigai
/ip firewall filter disable [find comment="Rule Terakhir"]

# 2. Menonaktifkan Queue Tree sementara (bukan menghapus)
/queue tree set [find] disabled=yes

# 3. Menonaktifkan pengalihan Proxy sementara
/ip firewall nat disable [find comment="Transparent Proxy HTTP"]

# 4. Mengembalikan cadangan apabila jaringan masih tidak berfungsi
/system backup load name=backup-mingguan
```
