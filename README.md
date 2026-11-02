# BNT Relay + Miner — Commands

Relay aktif: `stratum+tcp://82.39.206.183:443` (target bntpool.com:3333).
Allowlist: `124.174.3.209` (ArkClaw egress). IP miner lain harus ditambahkan dulu.

## Setup relay (mesin relay)

```bash
curl -O https://raw.githubusercontent.com/agenticspring/bnt-relay/main/setup-complete.sh
chmod +x setup-complete.sh
./setup-complete.sh relay <IP-MINER> [IP-MINER...]
```

Tambah IP miner ke allowlist relay yang sudah jalan:

```bash
iptables -I INPUT -s <IP-MINER>/32 -p tcp --dport 443 -j ACCEPT
iptables-save > /etc/iptables/rules.v4
```

## Setup miner (mesin miner)

Lewat relay (utama):

```bash
curl -O https://raw.githubusercontent.com/agenticspring/bnt-relay/main/setup-complete.sh
chmod +x setup-complete.sh
./setup-complete.sh mining --relay 82.39.206.183 --run
```

Direct ke pool (hanya kalau direct terbukti jalan dari mesin itu):

```bash
./setup-complete.sh mining --direct --run
```

## Ganti pool di miner yang sudah terpasang

```bash
cd /root/bnt-seine-runner
cp bakekok.sh bakekok.sh.orig
sed -i "s|stratum+tcp://[^'\"]*|stratum+tcp://82.39.206.183:443|" bakekok.sh
grep POOL_URL bakekok.sh
bash -n bakekok.sh
systemctl restart bnt-miner   # atau jalankan ./bakekok.sh lagi
```

Ganti wallet address:

```bash
sed -i "s|^ADDRESS=.*|ADDRESS=<WALLET-BNT>|" /root/bnt-seine-runner/bakekok.sh
grep ADDRESS /root/bnt-seine-runner/bakekok.sh
systemctl restart bnt-miner
```

## Verifikasi

```bash
# status service
systemctl is-active bnt-relay
systemctl is-active bnt-miner

# log miner — harus ada "login accepted" + share "accepted"
journalctl -u bnt-miner -n 40 --no-pager
# atau (non-systemd):
tail -n 40 /root/bnt-seine-runner/bakekok.log

# hashrate / HugePages
grep HugePages /proc/meminfo

# listener relay
ss -ltnp | grep ':443'

# allowlist aktif
iptables -S INPUT | grep -- '--dport 443'
```

## Monitoring pool

- Pool: `bntpool.com:3333` (fee 2,5%, PPLNS 6 jam, min payout 0,1 BNT)
- Wallet: cek balance/hashrate di dashboard bntpool.com dengan address `ADDRESS` di `bakekok.sh`

## Troubleshooting

```bash
# miner connect-loop / timeout -> cek IP egress miner sudah di-allowlist relay
curl -s ifconfig.me   # jalankan di mesin miner

# relay down -> restart
systemctl restart bnt-relay

# port 443 dipakai proses lain
ss -ltnp | grep ':443'

# HugePages kurang (butuh 1024 x 2MiB)
sysctl vm.nr_hugepages=1024
```

## Upgrade versi seine

Versi dipin di skrip (v0.2.45). Upgrade ke versi baru:

```bash
SEINE_VERSION=v0.2.46 SEINE_SHA256=<sha-dari-SHA256SUMS.txt> ./setup-complete.sh mining --relay 82.39.206.183 --run
```

SHA256 ambil dari `SHA256SUMS.txt` di release seine. Auto-restart: systemd `Restart=always` (VPS mati/hidup lagi = miner jalan sendiri); di mesin tanpa systemd, cron `@reboot` + cek tiap 5 menit.
