# 01 — Kurulum (Sıfırdan)

Hiçbir şey yokken tam VoLTE ağı. Zaten çalışan bir 4G ağın varsa bunun yerine
[`tara_kur.sh`](../scripts/tara_kur.sh) kullan (mevcut yapıya sadece VoLTE ekler).

## Ön koşullar

- Ubuntu 22.04 host (root/sudo).
- USRP B210 + (opsiyonel) GPSDO.
- sysmoISIM-SJA5 SIM'ler + pySim programlayıcı.
- Docker + docker compose.

## 1. UHD / B210

```bash
sudo apt update
sudo apt install -y uhd-host libuhd-dev
sudo uhd_images_downloader
# B210'u USB 3.0 porta tak, doğrula:
uhd_usrp_probe   # "Operating over USB 3" + "Register loopback test passed" görmeli
```
Sorun olursa: [`06_SORUN_GIDERME.md`](06_SORUN_GIDERME.md) §6.

## 2. docker_open5gs

```bash
git clone https://github.com/herlesupreeth/docker_open5gs
cd docker_open5gs
git checkout master
cp .env.example .env   # ve .env içinde HOST IP / DNS değerlerini ayarla
```
`.env`'de host IP'yi kendi ağına göre ayarla (bu repoda örnek: 192.168.23.6).

## 3. EPC + IMS servisleri

```bash
# IMS dahil 4G VoLTE yığını:
sudo docker compose -f 4g-volte-deploy.yaml up -d
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```
P/I/S-CSCF, pyHSS, rtpengine, mongo, mme, sgw, pgw vb. "Up" olmalı.

## 4. xfrm modülleri (IMS IPsec)

```bash
sudo modprobe xfrm_user esp4 xfrm4_tunnel tunnel4 ah4
lsmod | grep -E "xfrm|esp"
```
(Kalıcı olması için `/etc/modules-load.d/` altına eklenebilir.)

## 5. SIM programla

[`02_SIM_PROGRAMLAMA.md`](02_SIM_PROGRAMLAMA.md): PLMN 001/01, Type=OPc, **SQN kapalı**.

## 6. Abone ekle (EPC + IMS)

```bash
cd <repo>/scripts
./volte add subscriber --imsi 001010000000001 \
  --ki <KI> --opc <OPC> --msisdn 0010000000001
./volte add subscriber --imsi 001010000000002 \
  --ki <KI> --opc <OPC> --msisdn 0010000000002
```
Elle yöntemler: [`03_ABONE_EKLEME.md`](03_ABONE_EKLEME.md).

## 7. eNB başlat

```bash
cd <docker_open5gs_dizini>
sudo docker compose -f srsenb.yaml up -d
sudo docker container attach srsenb   # logları görmek için
```
Band/EARFCN/güç değerleri `srsenb` config'inde; bu kurulumda Band 5 / EARFCN 2525 /
tx_gain 50 kullanıldı (DURUM.md §8).

## 8. Telefon + doğrula

- VoLTE aç: [`04_TELEFON_VOLTE.md`](04_TELEFON_VOLTE.md).
- Attach + register doğrula, sonra çağrı: [`05_VOLTE_CAGRI.md`](05_VOLTE_CAGRI.md).

```bash
./volte check --imsi 001010000000001
```
