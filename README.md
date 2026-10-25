# BNT Mining Lengkap: Relay + Miner — dari nol sampai jalan

Repo ini berisi **satu paket lengkap** untuk mining Blocknet
(BNT) ke **pool lama bntpool**, termasuk relay-nya. Satu repo,
dua mesin, dari download sampai miner jalan dan terverifikasi.

**Arus lengkapnya:**

```
Mesin miner (B) ──stratum──> Mesin relay (A):443 ──stratum──> bntpool.com:3333
```

**Pool tujuan (tetap di semua skrip repo ini):**
bntpool.com port 3333, fee pool 2,5%, PPLNS 6 jam,
minimum payout 0,1 BNT.

**Miner:** Seine v0.2.21, 1 thread, 1024 HugePages (2 MiB).

---

## 0. Gambaran + yang dibutuhkan

| Peran | Mesin | Yang dipasang |
|---|---|---|
| Relay (A) | VPS/droplet Linux apa pun, Python 3, port 443 bebas | `relay.py` + service systemd + firewall allowlist |
| Miner (B) | Linux 64-bit (VPS/Claw), RAM cukup buat 2 GiB HugePages | Seine v0.2.21 + `bakekok.sh` |

Yang dibutuhkan sebelum mulai:

- Akses root di kedua mesin.
- IP egress mesin miner — cek dengan `curl -s ifconfig.me`
  di terminal miner. IP inilah yang harus di-allowlist di relay.
- Kalau miner adalah VPS dengan systemd (bukan Claw), miner
  bisa juga dipasang sebagai service — lihat bagian
  "Miner sebagai service systemd" di bawah.

**Kenapa relay ini ada (baca dulu biar nggak salah jalur):**

1. Mesin yang egress-nya hanya membuka port 80/443 — direct
   ke port 3333 tidak bisa sama sekali.
2. Jalur yang koneksi stratum direct-nya ditutup di tengah
   jalan (RST/EOF sesaat setelah connect, tanpa pesan error,
   bahkan untuk koneksi idle) — terbukti dari jalur ArkClaw
   ke bntpool. Lewat relay port 443, miner mendapat
   `login accepted` normal.

Relay ini **bukan** HTTP proxy: ia hanya meneruskan byte
stratum dua arah. Ia tidak meneruskan API web pool, jadi
telemetri HTTP pool lewat IP relay akan gagal — itu tidak
menghentikan share.

**Konsekuensi pakai relay (jujur, di depan):**

- Dev fee Seine kembali ke 2,5% lewat relay (pool tidak
  terdeteksi langsung), dibanding 1% kalau direct.
- Relay adalah satu titik ketergantungan: kalau mesin relay
  mati, semua miner lewat relay ikut putus.
- Kalau direct ke bntpool sudah jalan di suatu mesin, mesin
  itu tidak perlu relay.

---

## 1. Cara cepat: satu skrip untuk semuanya

Ini cara yang disarankan. Satu file, dua perintah, dari nol
sampai jalan. Jalankan sebagai root, di mesin yang sesuai.

### 1a. Download `setup-complete.sh` dulu

Di **kedua** mesin (relay dan miner), download dulu:

```bash
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -o setup-complete.sh
chmod +x setup-complete.sh
bash -n setup-complete.sh
```

`bash -n` harus lolos tanpa error. Kalau download-nya
mentok (lihat bagian "Download alternatif kalau ke-blokir"
di bawah), pakai jalur alternatif di sana — jangan lanjut
dengan file setengah download.

### 1b. Pasang relay (di mesin A)

```bash
./setup-complete.sh relay <IP-EGRESS-MINER-B>
```

Contoh: kalau IP egress miner adalah `203.0.113.10`:

```bash
./setup-complete.sh relay 203.0.113.10
```

Yang dikerjakan skrip, berurutan:

1. Salin/download `relay.py`, dicek dulu targetnya harus
   `bntpool.com` (kalau bukan, skrip berhenti).
2. `py_compile`, pasang service systemd + enable + start.
3. Tambah ACCEPT per IP miner **sebelum** DROP 443, lalu
   simpan aturan permanen ke `/etc/iptables/rules.v4`.
4. Verifikasi: service active + listener 443 terlihat.

Kalau ada miner kedua/ketiga yang mau ikut lewat relay yang
sama, ulangi dengan IP-nya (boleh beberapa IP sekaligus):

```bash
./setup-complete.sh relay <IP-MINER-1> <IP-MINER-2>
```

### 1c. Pasang miner (di mesin B)

**Lewat relay (cara utama):**

```bash
./setup-complete.sh mining --relay <IP-MESIN-RELAY-A> --run
```

**Direct ke bntpool (hanya kalau direct terbukti jalan
dari mesin ini):**

```bash
./setup-complete.sh mining --direct --run
```

Dari ArkClaw, direct terbukti gagal (RST), jadi di sana
pakai `--relay`, bukan `--direct`.

Yang dikerjakan skrip miner, berurutan:

1. Buat folder `/root/bnt-seine-runner`.
2. Download Seine v0.2.21 (GitHub dulu, fallback mirror
   otomatis kalau mentok), **verifikasi SHA-256 resmi
   sebelum ekstrak**, `chmod +x`, cek `./seine --version`.
3. Download `bakekok.sh`, lalu **hanya ubah baris
   `POOL_URL`-nya**: ke `stratum+tcp://<IP-RELAY>:443`
   kalau `--relay`, atau ke `stratum+tcp://bntpool.com:3333`
   kalau `--direct`. Skrip berhenti kalau POOL_URL gagal
   diubah.
4. Kalau pakai `--run`: miner langsung dijalankan dengan
   `nohup` + `disown`, lalu log di-`tail` 15 detik kemudian.

**PERINGATAN PENTING soal `bakekok.sh` dan pool:**

`bakekok.sh` di-download dari repo `bnt-seine-runner`, dan
versi terbaru di sana mengarah ke pool kc
(`stratum+tcp://103.54.127.250:3333`), **bukan** bntpool.
Karena itu skrip di repo ini selalu menimpa `POOL_URL`
setelah download — jangan pernah menjalankan `bakekok.sh`
hasil download mentah-mentah tanpa cek `grep POOL_URL`
dulu, kalau tujuanmu adalah bntpool.

**PERINGATAN soal `ADDRESS`:**

Skrip ini **tidak mengubah** `ADDRESS` di `bakekok.sh`
(di-download apa adanya dari repo `bnt-seine-runner`).
Kalau ini bukan mesin kamu, ganti dulu `ADDRESS` di
`bakekok.sh` sebelum `--run`. Yang diubah skrip hanya
`POOL_URL`, dan itu diverifikasi dengan `grep` sebelum jalan.

### 1d. Verifikasi miner jalan (wajib)

Setelah `--run`, atau setelah menjalankan manual, log
harus menunjukkan urutan ini:

1. `Huge pages currently: total=1024` / `HugePages_Total=1024`
   (kalau kurang dari 1024, miner tidak start — lihat
   troubleshooting HugePages).
2. `connecting to stratum+tcp://<IP-RELAY>:443`
   (atau `bntpool.com:3333` kalau direct).
3. `login accepted (protocol v2, required=submit_claimed_hash)`.
4. `new job` dengan difficulty awal (di bntpool biasanya
   puluhan sampai ratusan, jauh di bawah 600-nya kc).
5. Share pertama `accepted` dalam puluhan detik pertama.
6. Baris STATS: `hashes` bertambah, `rate` sekitar
   1,7–1,8 H/s untuk 1 thread di Claw (tergantung CPU),
   `submitted` dan `accepted` bertambah, `stale` mendekati 0.

Perintah ceknya:

```bash
cd /root/bnt-seine-runner
tail -n 50 bakekok.log
ps aux | grep -E '[s]eine'
grep HugePages /proc/meminfo
```

Kalau poin 3 tidak pernah muncul, jangan ulang install
membabi buta — lihat bagian troubleshooting sesuai gejalanya.

---

## 2. Cara manual: pasang relay satu per satu

Kalau lebih suka mengerjakan satu per satu (atau skrip
setup tidak bisa jalan di mesin itu), ini langkah manual
yang setara persis dengan mode `relay` di atas.

Jalankan sebagai root di mesin relay:

```bash
mkdir -p /opt/bnt-relay
cd /opt/bnt-relay
# salin relay.py dan bnt-relay.service ke sini
# (dari repo ini, atau upload via scp/SFTP — lihat bagian download alternatif)
python3 -m py_compile relay.py
cp bnt-relay.service /etc/systemd/system/bnt-relay.service
systemctl daemon-reload
systemctl enable --now bnt-relay
systemctl status bnt-relay --no-pager
```

Cek log-nya:

```bash
journalctl -u bnt-relay -f
```

Harus muncul `relay listening 0.0.0.0:443 -> bntpool.com:3333`,
lalu `relay conn from <IP-miner>` setiap miner tersambung.

### Firewall allowlist (wajib, manual)

Aturan ACCEPT harus berada **sebelum** DROP. Contoh untuk
dua IP miner, lalu simpan permanen (Debian/Ubuntu pakai
`iptables-persistent`):

```bash
iptables -A INPUT -s <IP-MINER-1>/32 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -s <IP-MINER-2>/32 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
iptables-save > /etc/iptables/rules.v4
iptables -S INPUT
```

Kalau IP miner berubah (NAT baru, mesin baru), tambahkan
IP barunya dulu sebelum miner diarahkan ke relay. Tanpa
allowlist, connect ke relay akan timeout (bukan refused) —
itu tanda IP belum diizinkan, bukan relay mati.

---

## 3. Cara manual: pasang miner satu per satu

Ini langkah manual yang setara persis dengan mode `mining`
di atas. Jalankan sebagai root di mesin miner.

### Langkah 1 — folder + binary Seine v0.2.21 (WAJIB)

Langkah ini wajib dan tidak boleh dilewati: file
`seine.tar.gz` **harus** di-download dan diekstrak dulu,
karena `bakekok.sh` tidak men-download binary apa pun.

Penting soal lokasi: download dan ekstrak harus dilakukan
di **folder yang sama dengan `bakekok.sh` nanti**
(`/root/bnt-seine-runner`), bukan di home root polos
(`/root/`). `bakekok.sh` mencari binary di foldernya
sendiri (`./seine`) — kalau binary jatuh di `/root/seine`
sementara `bakekok.sh` ada di `/root/bnt-seine-runner/`,
skrip akan gagal dengan `Cannot find the existing Seine binary`.

```bash
mkdir -p /root/bnt-seine-runner
cd /root/bnt-seine-runner
curl -L https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz
ls -lh seine.tar.gz
sha256sum seine.tar.gz
tar -xzf seine.tar.gz
chmod +x ./seine
./seine --version
```

Yang harus muncul:

- `ls -lh` → sekitar **3,6 MB**.
- `sha256sum` →
  `a94248aac209e9a18195adb21ef1dab996a6fe2bde0090bd7eaf65e03c91da77`
  (SHA resmi tarball Seine v0.2.21). Kalau tidak cocok,
  **berhenti** — file corrupt atau bukan rilis resmi,
  jangan ekstrak.
- `./seine --version` → `seine v0.2.21`.

Kalau download dari GitHub mentok di 0 byte (semua kolom
progress 0, kecepatan 0), itu pola egress mesin itu
memblokir `github.com` — jangan ditungguin, tekan `Ctrl+C`
dan pakai jalur alternatif di bagian 5 (mirror repo,
yang isinya tarball yang sama persis).

Kalau download sempat jalan lalu putus di tengah
(misalnya sempat 1,2 MB dari 3,6 MB lalu timeout),
lanjutkan dengan resume, jangan ulang dari nol:

```bash
curl -L -C - https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz
```

### Langkah 2 — download `bakekok.sh` ke folder yang sama

Perintah ini dijalankan dari folder yang sama
(`/root/bnt-seine-runner`), tempat binary `./seine` dari
langkah 1 sudah berada. Jangan pindah folder di antara
langkah 1 dan 2.

```bash
curl -fsSL https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/bakekok.sh -o bakekok.sh
chmod +x bakekok.sh
bash -n bakekok.sh
ls -lh
```

`bash -n` harus lolos tanpa error, dan `ls` harus
menunjukkan `bakekok.sh` berdampingan dengan `./seine`.
Kalau file hasil download punya akhir baris Windows
(CRLF), `bash -n` bisa gagal — bersihkan dulu akhir
barisnya sebelum dijalankan.

Kalau perintah raw di atas menggantung tanpa pesan error
(karena `-fsS` itu mode silent) atau gagal total, jalankan
dulu tanpa silent supaya errornya kelihatan, lalu pakai
mirror jsDelivr (isi file sama persis dari repo):

```bash
curl -L https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/bakekok.sh -o bakekok.sh
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-seine-runner@main/bakekok.sh -o bakekok.sh
chmod +x bakekok.sh
bash -n bakekok.sh
grep POOL_URL bakekok.sh
```

### Langkah 3 — arahkan ke pool yang benar (WAJIB)

Ini langkah yang paling sering bikin salah: `bakekok.sh`
hasil download mengarah ke pool kc, bukan bntpool. Ubah
**hanya** baris `POOL_URL`-nya, lalu verifikasi.

**Lewat relay (cara utama):**

```bash
cp bakekok.sh bakekok.sh.orig
sed -i 's|stratum+tcp://103.54.127.250:3333|stratum+tcp://IP-MESIN-RELAY:443|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
```

**Direct ke bntpool (hanya kalau direct terbukti jalan):**

```bash
cp bakekok.sh bakekok.sh.orig
sed -i 's|stratum+tcp://103.54.127.250:3333|stratum+tcp://bntpool.com:3333|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
```

`grep POOL_URL` harus menunjukkan pool yang kamu mau,
bukan kc lagi. Kalau `sed`-nya tidak mengubah apa-apa
(misalnya POOL_URL awalnya sudah bukan kc), cek isi
`bakekok.sh` dulu — jangan asal jalan.

Jangan loncat ke langkah 4 sebelum `bakekok.sh`
benar-benar ada dan POOL_URL-nya benar: kalau file-nya
belum ada, `nohup` akan gagal dengan
`No such file or directory` dan miner tidak jalan sama sekali.

### Langkah 4 — jalankan miner di background

```bash
nohup ./bakekok.sh > bakekok.log 2>&1 &
disown
sleep 15
tail -n 50 bakekok.log
grep HugePages /proc/meminfo
```

Lalu verifikasi sesuai bagian 1d: `login accepted` +
share `accepted` + STATS bertambah + `stale` mendekati 0.
Tutup lalu buka lagi terminal: proses `./seine` harus
masih ada (tanda `nohup` + `disown` bekerja) — tutup
terminal itu aman, miner tidak ikut mati.

**Yang jangan ditekan di dashboard Claw:**

- Tombol **Restart** hanya me-restart aplikasi Claw,
  **bukan** OS-nya — jadi belum tentu membebaskan RAM
  seperti reboot beneran.
- Tombol **Factory Reset (恢复出厂)** jangan pernah
  ditekan — itu menghapus semua file, beda jauh dengan
  Restart.

---

## 4. Menjalankan, menghentikan, dan me-restart

### Relay

```bash
systemctl status bnt-relay --no-pager
journalctl -u bnt-relay -n 100 --no-pager
systemctl restart bnt-relay
systemctl stop bnt-relay
```

Restart relay akan memutus sesi miner yang sedang lewat
relay sesaat (miner akan reconnect sendiri) — satu baris
`pool offline` di log miner lama saat restart relay itu
efek restart, bukan loop.

### Miner (di mesin miner)

```bash
cd /root/bnt-seine-runner
ps aux | grep -E '[s]eine'
tail -n 50 bakekok.log
# hentikan miner saja (tanpa menghapus apa-apa):
kill <PID-miner>
sleep 2
ps aux | grep -E '[s]eine'
# jalankan lagi:
nohup ./bakekok.sh > bakekok.log 2>&1 &
disown
```

### Pindah miner yang sudah jalan ke pool lain

Miner yang sedang jalan tidak ikut berubah saat repo
di-update. Untuk memindahkannya (misalnya dari relay ke
direct, atau sebaliknya):

```bash
cd /root/bnt-seine-runner
ps aux | grep -E '[s]eine'
kill <PID-miner-lama>
sleep 2
ps aux | grep -E '[s]eine'
cp bakekok.sh bakekok.sh.<nama-pool-lama>
sed -i 's|stratum+tcp://<pool-lama>|stratum+tcp://<pool-baru>|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
nohup ./bakekok.sh > bakekok.log 2>&1 &
disown
sleep 15
tail -n 50 bakekok.log
```

Aturan yang sama berlaku: hanya POOL_URL yang diubah,
ADDRESS tidak disentuh, dan verifikasi `login accepted`
+ share accepted sebelum ditinggal.

### Miner sebagai service systemd (VPS, bukan Claw)

Kalau mesin miner adalah VPS dengan systemd dan kamu mau
miner hidup otomatis setelah reboot, buat unit ini
(sesuaikan pool-nya ke relay atau direct):

```ini
[Unit]
Description=BNT Seine miner
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/root/bnt-seine-runner
ExecStart=/root/bnt-seine-runner/bakekok.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
cp <unit-di-atas> /etc/systemd/system/seine-bnt.service
systemctl daemon-reload
systemctl enable --now seine-bnt
journalctl -u seine-bnt -f
```

Catatan: `bakekok.sh` sendiri sudah menyiapkan HugePages
dan menjalankan `./seine` di foreground di dalam service,
jadi unit ini tidak perlu `nohup`.

---

## 5. Download alternatif kalau ke-blokir

Pola blokir yang sudah terbukti di lapangan:

- `github.com` mentok 0 byte / timeout, tapi
  `raw.githubusercontent.com` lancar — atau sebaliknya.
- `raw.githubusercontent.com` menggantung tanpa error
  (mode silent), tapi jsDelivr lancar.
- Keduanya buntu dari dalam mesin, tapi download dari
  HP/laptop lancar.

**Aturan umumnya: coba berurutan, jangan stuck di satu
jalur.** Untuk setiap file, urutannya:

1. URL asli (GitHub / raw).
2. Mirror jsDelivr dari repo yang sama.
3. `wget` dengan URL yang sama.
4. Download di HP/laptop, upload via scp/SFTP ke folder
   yang benar, lalu verifikasi (SHA / `bash -n` / `grep`)
   sebelum dipakai.

### 5a. `setup-complete.sh` (repo ini)

```bash
# jalur 1 — jsDelivr (disarankan, sudah terbukti di Claw):
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -o setup-complete.sh
# jalur 2 — raw:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/setup-complete.sh -o setup-complete.sh
# jalur 3 — wget:
wget https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -O setup-complete.sh
chmod +x setup-complete.sh
bash -n setup-complete.sh
```

### 5b. `relay.py` + `bnt-relay.service` (repo ini)

```bash
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/relay.py -o relay.py
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/bnt-relay.service -o bnt-relay.service
# cadangan raw:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/relay.py -o relay.py
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/bnt-relay.service -o bnt-relay.service
python3 -m py_compile relay.py
grep 'TARGET_HOST' relay.py
```

Harus muncul `TARGET_HOST = "bntpool.com"`.

### 5c. Seine tarball (3,6 MB)

```bash
# jalur 1 — GitHub rilis resmi:
curl -L https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz
# jalur 2 — mirror repo (tarball yang sama, sudah di-vendor):
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-seine-runner@main/seine/seine-linux-x86_64.tar.gz -o seine.tar.gz
# jalur 3 — raw repo:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/seine/seine-linux-x86_64.tar.gz -o seine.tar.gz
# kalau sempat putus di tengah, lanjutkan (jangan ulang dari nol):
curl -L -C - https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz
ls -lh seine.tar.gz
sha256sum seine.tar.gz
```

SHA yang benar:
`a94248aac209e9a18195adb21ef1dab996a6fe2bde0090bd7eaf65e03c91da77`

### 5d. `bakekok.sh` (repo mining)

```bash
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-seine-runner@main/bakekok.sh -o bakekok.sh
curl -L https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/bakekok.sh -o bakekok.sh
chmod +x bakekok.sh
bash -n bakekok.sh
grep POOL_URL bakekok.sh
```

Ingat: habis download, **wajib** ubah POOL_URL ke
relay/bntpool (bagian 3 langkah 3) sebelum jalan.

### 5e. Kalau semua jalur download dari dalam mesin buntu

Download file-nya di HP/laptop dari URL-URL di atas,
lalu upload via scp/SFTP ke folder yang benar:

```bash
# dari HP/laptop (ganti IP-MESIN):
scp setup-complete.sh root@IP-MESIN:/root/
scp relay.py bnt-relay.service root@IP-MESIN:/opt/bnt-relay/
scp seine.tar.gz bakekok.sh root@IP-MESIN:/root/bnt-seine-runner/
```

Lalu di terminal mesin itu, verifikasi dulu sebelum
dipakai: `bash -n` untuk skrip, `sha256sum` untuk
tarball, `grep POOL_URL` / `grep TARGET_HOST` untuk
pool yang dituju. Jangan pernah menjalankan file hasil
upload tanpa verifikasi itu.

---

## 6. Troubleshooting

### HugePages tidak sampai 1024, miner tidak start

Gejala: log `ERROR: Only <N> HugeTLB pages are free;
Seine needs at least 1024. Not starting the miner.`
Ini masalah RAM/fragmentasi, bukan salah install.
Alokasi HugePages cenderung **bertahan dan naik tiap
percobaan** (contoh nyata: 548 → 924 → 958 → 1024),
jadi ulangi perintah yang sama beberapa kali dulu:

```bash
echo 3 > /proc/sys/vm/drop_caches
echo 1 > /proc/sys/vm/compact_memory
nohup ./bakekok.sh > bakekok.log 2>&1 &
disown
sleep 15
tail -n 30 bakekok.log
grep HugePages /proc/meminfo
```

Yang tidak perlu diulang tiap kali: `free` dan `ps`
(cuma diagnostik). Yang tidak boleh dilakukan: mematikan
proses sistem/desktop Claw (xfce, Theia, containerd)
demi RAM — totalnya tidak cukup dan bisa memutus
terminal/Claw itu sendiri.

Kalau setelah 2–3 kali ulang masih di bawah 1024 dan
`MemAvailable` sudah sangat kecil, berhenti memaksa:
reboot mesin lewat terminal (`reboot`, bukan tombol
Restart dashboard), lalu jalankan `bakekok.sh` secepatnya
setelah boot, selagi RAM masih kosong, tanpa membuka
aplikasi lain dulu.

Di Claw Small (RAM total ~3,8 GiB), miner butuh 1024
pages × 2 MiB = 2 GiB — jadi memang mepet dan percobaan
berulang adalah bagian normal dari prosesnya, bukan
kegagalan skrip.

### `Cannot find the existing Seine binary`

Binary `./seine` tidak ada di folder yang sama dengan
`bakekok.sh`. Ini sering terjadi kalau tarball
di-download/diekstrak di `/root/` polos sementara
`bakekok.sh` ada di `/root/bnt-seine-runner/`.
Solusinya: ulangi langkah 1 dari dalam folder
`/root/bnt-seine-runner` (atau pindahkan binary `./seine`
ke folder itu), lalu jalankan lagi.

### `No such file or directory` saat `nohup ./bakekok.sh`

File `bakekok.sh` belum ada di folder itu; kembali ke
langkah 2, dan pastikan kamu masih di
`/root/bnt-seine-runner` saat download.

### `SHA-256 mismatch`

Arsip tidak diekstrak; ulangi download (download corrupt
atau manifest berubah). Jangan paksa ekstrak file yang
SHA-nya tidak cocok.

### Download mentok 0 / menggantung tanpa error

Lihat bagian 5. Intinya: `Ctrl+C`, ganti jalur (GitHub →
jsDelivr → raw → wget → upload HP/laptop), dan selalu
verifikasi ukuran + SHA / `bash -n` setelah download.
Perintah dengan `-fsSL` itu mode silent — kalau
menggantung, ulang tanpa `-s` supaya errornya kelihatan.

### Pool offline / connection reset berulang (direct)

Gejala: `pool offline at stratum+tcp://bntpool.com:3333
(Connection reset by peer)` berulang tiap ~2 detik, STATS
terus `hashes=0, rate=0.000, accepted=0`, dan tidak pernah
ada baris `login accepted`.

Diagnosis yang sudah terbukti (Claw ArkClaw): TCP connect
selalu berhasil, tapi koneksi ditutup ~0,22 detik kemudian
— bahkan koneksi idle tanpa data pun ditutup.
`mining.authorize` tidak pernah terkirim, jadi penyebabnya
bukan wallet, bukan worker, bukan versi Seine. Ini masalah
jalur egress ke pool tersebut, bukan masalah install.

Solusinya: **pakai relay** (bagian 1c/3 langkah 3).
Mengulang install, ganti worker, atau ganti hostname pool
dengan IP-nya langsung sudah dicoba dan **tidak mengubah
apa-apa** — polanya identik.

### Pool offline ke relay / timeout ke relay

Cek berurutan:

1. Service relay aktif? `systemctl is-active bnt-relay`
   di mesin relay, dan log-nya ada `relay listening`?
2. IP egress miner sudah di allowlist? Di mesin relay:
   `iptables -S INPUT | grep 443` — harus ada ACCEPT
   untuk IP itu **sebelum** DROP.
3. Dari mesin miner, tes connect mentah:
   ```bash
   timeout 5 bash -c 'cat < /dev/null > /dev/tcp/IP-RELAY/443 && echo RELAY_OK || echo RELAY_TIMEOUT'
   ```
   `RELAY_TIMEOUT` = IP belum di-allowlist (atau relay
   mati), bukan salah miner.

### Probe generik ditolak pool — bukan berarti pool tidak cocok

Kalau kamu mengetes pool dengan probe `mining.subscribe`
generik, server bisa menjawab `{"error":"unknown method"}`.
Itu **bukan** bukti pool tidak cocok: Seine memakai
protokol stratum Blocknet v2 (terlihat di log sebagai
`login accepted (protocol v2, required=submit_claimed_hash)`),
bukan `mining.subscribe` generik. Satu-satunya tes
kompatibilitas yang valid adalah menjalankan Seine itu
sendiri ke endpoint itu dan melihat `login accepted` +
share accepted.

### Peringatan `pool telemetry ... rate limited` / gagal

Peringatan telemetri HTTP pool boleh diabaikan: itu batas
API telemetri pool (atau, lewat relay, API pool memang
tidak diteruskan relay). Mining tetap jalan selama
`login accepted` dan share accepted bertambah.

---

## 7. Checklist akhir (sebelum ditinggal jalan)

- [ ] Relay: service `bnt-relay` active, log `relay listening
      0.0.0.0:443 -> bntpool.com:3333`, aturan ACCEPT IP miner
      ada sebelum DROP, aturan tersimpan permanen.
- [ ] Miner: `./seine --version` = v0.2.21, SHA tarball cocok,
      `bakekok.sh` ada berdampingan dengan `./seine`.
- [ ] Miner: `grep POOL_URL bakekok.sh` = relay:443 (atau
      bntpool direct kalau memang direct) — **bukan** kc.
- [ ] Miner: `bash -n bakekok.sh` lolos.
- [ ] Miner: log `login accepted` + share pertama `accepted`
      + STATS bertambah + `stale` mendekati 0.
- [ ] Miner: `HugePages_Total=1024`, proses `./seine` masih
      ada setelah terminal ditutup lalu dibuka lagi.
- [ ] Kalau pakai service systemd: `systemctl is-enabled`
      dan `is-active` untuk service yang dipakai.

---

## Isi repo

- `setup-complete.sh` — setup lengkap: mode `relay` dan
  mode `mining` (bagian 1). Satu-satunya file yang perlu
  di-download untuk mulai dari nol.
- `relay.py` — relay-nya (Python 3 stdlib saja). Target
  `bntpool.com:3333`, lengkap dengan perbaikan timeout
  `settimeout(None)` setelah connect.
- `bnt-relay.service` — unit systemd relay.
- `README.md` — tutorial lengkap ini.
