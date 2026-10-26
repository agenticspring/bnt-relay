# Semua Command — Copy-Paste Lengkap

Satu file, semua command yang dipakai repo ini, berurutan.
Ganti placeholder yang ada di `<...>` dengan nilai kamu
sebelum dijalankan. Jalankan semua sebagai root.

**Catatan dari lapangan (Arclaw, 25 Okt 2026):** download
jsDelivr bisa gagal sekali dengan
`curl: (35) Recv failure: Connection reset by peer`,
lalu berhasil di percobaan kedua (8.480 byte untuk
`setup-complete.sh`). Itu normal di jalur ini — **jangan
lanjut kalau file belum ke-download**, ulangi saja
command download-nya. Command download di bawah sudah
pakai loop retry otomatis supaya tidak perlu ulang manual.

---

## 0. Persiapan (di mesin miner, untuk dapat IP-nya)

```bash
curl -s ifconfig.me
```

Catat IP itu — IP inilah yang di-allowlist di relay.

---

## 1. Relay — cara cepat (di mesin relay A)

```bash
# Download skrip (loop retry: coba sampai 5x kalau kena reset)
for i in 1 2 3 4 5; do curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -o setup-complete.sh && [ -s setup-complete.sh ] && break; sleep 2; done
chmod +x setup-complete.sh
bash -n setup-complete.sh
ls -lh setup-complete.sh

# Pasang relay + allowlist IP miner (boleh beberapa IP sekaligus)
./setup-complete.sh relay <IP-MINER>

# Verifikasi relay
systemctl is-active bnt-relay
systemctl status bnt-relay --no-pager
journalctl -u bnt-relay -n 50 --no-pager
iptables -S INPUT | grep 443
ss -ltnp | grep 443
```

Yang harus muncul: service `active`, log
`relay listening 0.0.0.0:443 -> bntpool.com:3333`,
aturan ACCEPT untuk IP miner **sebelum** DROP 443.

---

## 2. Miner — cara cepat (di mesin miner B, lewat relay)

```bash
# Download skrip (loop retry)
for i in 1 2 3 4 5; do curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -o setup-complete.sh && [ -s setup-complete.sh ] && break; sleep 2; done
chmod +x setup-complete.sh
bash -n setup-complete.sh
ls -lh setup-complete.sh

# Pasang miner lewat relay + langsung jalan
./setup-complete.sh mining --relay <IP-RELAY> --run

# Verifikasi miner (wajib)
cd /root/bnt-seine-runner
tail -n 50 bakekok.log
ps aux | grep -E '[s]eine'
grep HugePages /proc/meminfo
grep POOL_URL bakekok.sh
```

Yang harus muncul di log: `login accepted (protocol v2)`,
share pertama `accepted`, STATS `rate` naik dan `accepted`
bertambah, `stale` mendekati 0, `HugePages_Total=1024`.

**Varian direct (hanya kalau direct terbukti jalan):**

```bash
./setup-complete.sh mining --direct --run
```

Dari ArkClaw direct terbukti gagal (RST), jadi di sana
pakai `--relay`.

---

## 3. Relay — cara manual (tanpa setup-complete.sh)

```bash
mkdir -p /opt/bnt-relay
cd /opt/bnt-relay

for i in 1 2 3 4 5; do curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/relay.py -o relay.py && [ -s relay.py ] && break; sleep 2; done
for i in 1 2 3 4 5; do curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/bnt-relay.service -o bnt-relay.service && [ -s bnt-relay.service ] && break; sleep 2; done

python3 -m py_compile relay.py
grep TARGET_HOST relay.py
cp bnt-relay.service /etc/systemd/system/bnt-relay.service
systemctl daemon-reload
systemctl enable --now bnt-relay
systemctl status bnt-relay --no-pager
journalctl -u bnt-relay -f
```

Firewall allowlist (ACCEPT dulu, baru DROP, lalu simpan):

```bash
iptables -A INPUT -s <IP-MINER>/32 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
iptables-save > /etc/iptables/rules.v4
iptables -S INPUT | grep 443
```

---

## 4. Miner — cara manual (tanpa setup-complete.sh)

### 4a. Folder + binary Seine v0.2.21

```bash
mkdir -p /root/bnt-seine-runner
cd /root/bnt-seine-runner

for i in 1 2 3 4 5; do curl -L https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz && [ -s seine.tar.gz ] && break; sleep 2; done
ls -lh seine.tar.gz
sha256sum seine.tar.gz
tar -xzf seine.tar.gz
chmod +x ./seine
./seine --version
```

SHA yang benar:
`a94248aac209e9a18195adb21ef1dab996a6fe2bde0090bd7eaf65e03c91da77`
Ukuran sekitar 3,6 MB, versi `seine v0.2.21`.

### 4b. Download bakekok.sh (di folder yang sama)

```bash
cd /root/bnt-seine-runner
for i in 1 2 3 4 5; do curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-seine-runner@main/bakekok.sh -o bakekok.sh && [ -s bakekok.sh ] && break; sleep 2; done
chmod +x bakekok.sh
bash -n bakekok.sh
ls -lh
```

### 4c. Arahkan ke pool yang benar (WAJIB)

Lewat relay:

```bash
cd /root/bnt-seine-runner
cp bakekok.sh bakekok.sh.orig
sed -i 's|stratum+tcp://103.54.127.250:3333|stratum+tcp://<IP-RELAY>:443|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
```

Direct ke bntpool:

```bash
cd /root/bnt-seine-runner
cp bakekok.sh bakekok.sh.orig
sed -i 's|stratum+tcp://103.54.127.250:3333|stratum+tcp://bntpool.com:3333|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
```

`grep POOL_URL` harus menunjukkan pool yang kamu mau,
bukan kc lagi.

### 4d. Jalankan miner (service systemd, default via --run)

Cara cepat tetap pakai `--run` dari bagian 1 — itu sudah
memasang service `bnt-miner`. Manual di bawah hanya
kalau service terhapus:

```bash
cp bnt-miner.service /etc/systemd/system/bnt-miner.service
systemctl daemon-reload
systemctl enable --now bnt-miner
sleep 15
journalctl -u bnt-miner -n 50 --no-pager
grep HugePages /proc/meminfo
ps aux | grep -E '[s]eine'
```

---

## 5. Menjalankan, menghentikan, me-restart

Relay:

```bash
systemctl status bnt-relay --no-pager
journalctl -u bnt-relay -n 100 --no-pager
systemctl restart bnt-relay
systemctl stop bnt-relay
```

Miner (service):

```bash
systemctl status bnt-miner --no-pager
journalctl -u bnt-miner -n 50 --no-pager
systemctl restart bnt-miner
systemctl stop bnt-miner
```

Pindah miner yang sudah jalan ke pool lain:

```bash
cd /root/bnt-seine-runner
cp bakekok.sh bakekok.sh.<POOL-LAMA>
sed -i 's|stratum+tcp://<POOL-LAMA>|stratum+tcp://<POOL-BARU>|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
systemctl restart bnt-miner
sleep 15
journalctl -u bnt-miner -n 50 --no-pager
```

Miner sebagai service systemd — unitnya di repo
(`bnt-miner.service`), sudah dipasang otomatis oleh
mode `mining --run`. Service lama manual bernama
`seine-bnt` tidak lagi dipakai untuk instalasi baru.

---

## 6. Download alternatif kalau ke-blokir

Urutan coba per file: GitHub/raw → jsDelivr → wget →
download di HP/laptop lalu upload via scp. Selalu
verifikasi setelah download (`bash -n`, `sha256sum`,
`grep POOL_URL` / `grep TARGET_HOST`).

```bash
# setup-complete.sh — alternatif raw dan wget:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/setup-complete.sh -o setup-complete.sh
wget https://cdn.jsdelivr.net/gh/agenticspring/bnt-relay@main/setup-complete.sh -O setup-complete.sh
bash -n setup-complete.sh

# relay.py / service — alternatif raw:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/relay.py -o relay.py
curl -L https://raw.githubusercontent.com/agenticspring/bnt-relay/main/bnt-relay.service -o bnt-relay.service
python3 -m py_compile relay.py
grep TARGET_HOST relay.py

# Seine tarball — alternatif mirror dan resume:
curl -L https://cdn.jsdelivr.net/gh/agenticspring/bnt-seine-runner@main/seine/seine-linux-x86_64.tar.gz -o seine.tar.gz
curl -L https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/seine/seine-linux-x86_64.tar.gz -o seine.tar.gz
curl -L -C - https://github.com/zpalmtree/seine/releases/download/v0.2.21/seine-linux-x86_64.tar.gz -o seine.tar.gz
ls -lh seine.tar.gz
sha256sum seine.tar.gz

# bakekok.sh — alternatif raw:
curl -L https://raw.githubusercontent.com/agenticspring/bnt-seine-runner/main/bakekok.sh -o bakekok.sh
bash -n bakekok.sh
grep POOL_URL bakekok.sh

# Kalau semua jalur dari dalam mesin buntu — upload dari HP/laptop:
scp setup-complete.sh root@<IP-MESIN>:/root/
scp relay.py bnt-relay.service root@<IP-MESIN>:/opt/bnt-relay/
scp seine.tar.gz bakekok.sh root@<IP-MESIN>:/root/bnt-seine-runner/
```

---

## 7. Tes dan troubleshooting satu baris

```bash
# Tes connect mentah ke relay (dari mesin miner):
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP-RELAY>/443 && echo RELAY_OK || echo RELAY_TIMEOUT'

# Cek IP egress miner:
curl -s ifconfig.me

# Cek HugePages (harus Total 1024):
grep HugePages /proc/meminfo

# Bebaskan RAM + compact sebelum retry bakekok:
echo 3 > /proc/sys/vm/drop_caches
echo 1 > /proc/sys/vm/compact_memory

# Cek log miner dan relay:
tail -n 50 /root/bnt-seine-runner/bakekok.log
journalctl -u bnt-relay -n 50 --no-pager
```

`RELAY_TIMEOUT` = IP belum di-allowlist atau relay mati,
bukan salah miner. `RELAY_OK` = jalur terbuka, lanjut
pasang miner.

**Jangan ditekan di dashboard Claw:** tombol Factory Reset
(恢复出厂) — itu menghapus semua file. Tombol Restart hanya
me-restart aplikasi Claw, bukan OS.
