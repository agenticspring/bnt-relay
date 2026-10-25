# BNT Stratum Relay — pool lama (bntpool)

Relay TCP mentah untuk miner Blocknet (BNT) yang tidak bisa
menjangkau pool lama secara langsung.

**Arus:** miner → `mesin-relay:443` → `bntpool.com:3333`

**Pool tujuan (tetap):** bntpool.com, port 3333, fee pool 2,5%,
PPLNS 6 jam, minimum payout 0,1 BNT.

## Kenapa relay ini ada

Dua kasus yang sudah terbukti butuh relay:

1. Mesin yang egress-nya hanya membuka port 80/443 — direct ke
   port 3333 tidak bisa sama sekali.
2. Jalur yang koneksi stratum direct-nya ditutup di tengah jalan
   (RST/EOF sesaat setelah connect, tanpa pesan error) — lewat
   relay port 443, miner mendapat `login accepted` normal.

Relay ini **bukan** HTTP proxy: ia hanya meneruskan byte stratum
dua arah. Ia tidak meneruskan API web pool, jadi telemetri HTTP
pool lewat IP relay akan gagal — itu tidak menghentikan share.

## Yang dibutuhkan

- Satu mesin Linux yang bisa dijangkau miner (VPS/droplet apa pun),
  dengan Python 3 (tidak ada dependensi lain) dan port 443 bebas.
- Di mesin miner: binary Seine + skrip `bakekok.sh` seperti biasa.
  Satu-satunya yang berubah di miner adalah `POOL_URL` menjadi
  `stratum+tcp://IP-MESIN-RELAY:443`.
- **IP egress tiap miner harus ada di allowlist firewall relay.**
  Tanpa itu, connect ke relay akan timeout (bukan refused).
  Cek IP egress miner dengan `curl -s ifconfig.me`.

## Jalankan manual (di mesin relay)

Jalankan sebagai root, berurutan:

```bash
mkdir -p /opt/bnt-relay
cd /opt/bnt-relay
# salin relay.py dan bnt-relay.service ke sini
# (dari repo ini, atau upload via scp/SFTP)
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

## Firewall allowlist (wajib)

Contoh: izinkan dua IP miner, tolak sisanya di port 443,
lalu simpan permanen (Debian/Ubuntu pakai `iptables-persistent`):

```bash
iptables -A INPUT -s <IP-MINER-1>/32 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -s <IP-MINER-2>/32 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
iptables-save > /etc/iptables/rules.v4
iptables -S INPUT
```

Aturan ACCEPT harus berada **sebelum** DROP. Kalau IP miner
berubah (NAT baru, mesin baru), tambahkan IP barunya dulu
sebelum miner diarahkan ke relay.

## Di sisi miner

Di mesin miner, di folder yang sama dengan `bakekok.sh`:

```bash
cp bakekok.sh bakekok.sh.direct
sed -i 's|stratum+tcp://bntpool.com:3333|stratum+tcp://IP-MESIN-RELAY:443|' bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
nohup ./bakekok.sh > bakekok.log 2>&1 &
disown
sleep 15
tail -n 30 bakekok.log
```

Verifikasi: `login accepted (protocol v2)` + share `accepted`
di log miner. Kalau log miner menunjukkan `pool offline`
berulang ke relay, cek dulu: (1) service relay aktif,
(2) IP egress miner sudah di allowlist, (3) relay benar-benar
mendengarkan di 443 (`ss -ltnp | grep 443`).

## Konsekuensi pakai relay (jujur)

- Dev fee Seine kembali ke 2,5% lewat relay (pool tidak
  terdeteksi langsung), dibanding 1% kalau direct.
- Relay adalah satu titik ketergantungan: kalau mesin relay
  mati, semua miner lewat relay ikut putus.
- Kalau direct ke bntpool sudah jalan di suatu mesin, mesin
  itu tidak perlu relay.

## Isi repo

- `relay.py` — relay-nya (Python 3 stdlib saja). Target
  `bntpool.com:3333`, lengkap dengan perbaikan timeout
  `settimeout(None)` setelah connect.
- `bnt-relay.service` — unit systemd.
- `README.md` — tutorial manual ini.
