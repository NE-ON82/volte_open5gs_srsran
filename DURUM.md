# DURUM.md — Güncel Proje Gerçeği (Antigravity için)

> Bu dosya, VoLTE test ağının **çalışan ve kanıtlanmış** son durumunu içerir.
> Antigravity bu dosyayı script/döküman üretmeden önce MUTLAKA okumalıdır.
> Buradaki her madde canlı sistemde loglarla doğrulanmıştır — varsayım değildir.

---

## 1. BAŞARILMIŞ HEDEF

İki gerçek COTS telefon arasında **VoLTE çağrısı** kuruldu ve doğrulandı:
- **Redmi Note 12** (Kart 1) ↔ **Xiaomi 11T Pro** (Kart 2): IMS REGISTER 200 OK + VoLTE çağrısı ✅
- **iPhone** (Kart 1 ile) de IMS'e register oldu ve VoLTE çalıştı ✅

Kanıt: S-CSCF logunda her iki/üç cihaz için `Auth succeeded` + `SAR success - 200 response sent` + `state="active" event="registered"`.

---

## 2. KRİTİK MİMARİ GERÇEĞİ — İKİ AYRI HSS VAR

Bu kurulumda **İKİ ayrı HSS** vardır ve VoLTE için İKİSİNE DE abone girmek ZORUNLUDUR:

| HSS | Erişim | Ne için | Olmazsa |
|-----|--------|---------|---------|
| **open5gs HSS** | WebUI `http://<HOST>:9999` (admin/1423) | EPC / 4G attach / data | Telefon ağa attach olamaz |
| **pyHSS** | Swagger `http://<HOST>:8080/docs/` | IMS / VoLTE register | Attach olur ama VoLTE register OLMAZ |

> **ÖNEMLİ:** "pyHSS yok, S-CSCF otomatik atar" varsayımı YANLIŞTIR.
> pyHSS provisioning yapılmazsa I-CSCF `cxdx_get_server_name: Failed finding avp`
> hatası verir ve REGISTER S-CSCF'e hiç ulaşmaz. Bu, bu projenin EN BÜYÜK takılma
> noktasıydı ve kök çözüm pyHSS'e IMS subscriber girmekti.

---

## 3. pyHSS IMS PROVISIONING — 5 ADIM (Swagger :8080/docs)

Sıra önemlidir. APN'ler bir kez; sonra HER KART için auc→subscriber→ims_subscriber.
Her PUT'ta **örnek payload'daki fazla alanlar SİLİNMELİ** (apn_id:0, ip_version:0 vb.),
yoksa `operation_log / item_id cannot be null` (400) hatası gelir.

### Adım 1 — APN (bir kez, iki çağrı): `PUT /apn/`
```json
{"apn": "internet", "apn_ambr_dl": 0, "apn_ambr_ul": 0}
{"apn": "ims",      "apn_ambr_dl": 0, "apn_ambr_ul": 0}
```
Dönen `apn_id` değerlerini not al. (Bu oturumda: internet=2, ims=3 geldi — ID'ler
ortamda değişebilir, GET ile teyit edilmeli.)

### Adım 2 — AUC (her kart): `PUT /auc/`
```json
{"ki": "<KI>", "opc": "<OPC>", "amf": "8000", "sqn": 0, "imsi": "<IMSI>"}
```
Dönen `auc_id` not al.

### Adım 3 — SUBSCRIBER (her kart): `PUT /subscriber/`
```json
{"imsi": "<IMSI>", "enabled": true, "auc_id": <AUC_ID>,
 "default_apn": <INTERNET_APN_ID>, "apn_list": "<INTERNET_APN_ID>,<IMS_APN_ID>",
 "msisdn": "<MSISDN>", "ue_ambr_dl": 0, "ue_ambr_ul": 0}
```

### Adım 4 — IMS_SUBSCRIBER (her kart): `PUT /ims_subscriber/`  ← EN KRİTİK
```json
{"imsi": "<IMSI>", "msisdn": "<MSISDN>", "sh_profile": "string",
 "scscf_peer": "scscf.ims.mnc001.mcc001.3gppnetwork.org",
 "msisdn_list": "[<MSISDN>]", "ifc_path": "default_ifc.xml",
 "scscf": "sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060",
 "scscf_realm": "ims.mnc001.mcc001.3gppnetwork.org"}
```
Bu adım I-CSCF'in aradığı "server_name"i sağlar. Girilince REGISTER S-CSCF'e ulaşır.

---

## 4. HOST ÖN KOŞULU — xfrm KERNEL MODÜLLERİ (IPsec için)

P-CSCF, IMS IPsec için host'ta xfrm modüllerine ihtiyaç duyar. Host'ta:
```bash
sudo modprobe xfrm_user esp4 xfrm4_tunnel tunnel4 ah4
```
Doğrulama: `lsmod | grep -E "xfrm|esp"` ve `sudo ip xfrm state` (hatasız boş liste).

> NOT: P-CSCF logundaki `clean_sa(): ... No data available` ve
> `Error cleaning IPSec ... during startup` mesajları ZARARSIZDIR — başlangıçta
> silinecek SA olmadığı için çıkar, REGISTER'ı engellemez. Asıl engel pyHSS
> provisioning eksikliğiydi, bu IPsec uyarısı değil.

P-CSCF compose servisinde zaten olması gerekenler (docker_open5gs'te var):
`privileged: true` + `cap_add: [NET_ADMIN]`.

---

## 5. APN YAPISI (open5gs WebUI — her abone)

Resmi open5gs VoLTE dökümanıyla birebir. Her abonede 4 oturum:
- **internet**: IPv4, QCI 9, ARP 8, 1 Gbps
- **ims**: IPv4, QCI 5, ARP 1, Disabled/Disabled, 3850/1530 Kbps
- **ims**: IPv4, QCI 1, ARP 2, Enabled/Enabled, MBR 128/128 GBR 128/128
- **ims**: IPv4, QCI 2, ARP 4, Enabled/Enabled, 128/128

> KRİTİK: Type **IPv4** (IPv6 VoLTE'yi bozar). QCI1/2 GBR/MBR **128/128**
> (unlimited DEĞİL — çağrı kopmasını önler). `open5gs-dbctl` TAM APN kuramaz,
> WebUI veya pyHSS API tercih edilir.

---

## 6. TELEFONDA VoLTE AKTİFLEŞTİRME (Xiaomi/Redmi — app'siz)

1. Dialer'a çevir: `*#*#86583#*#*` → "VoLTE carrier check was disabled" çıkar
   (çıkmazsa tekrar çevir).
2. Ayarlar → Mobil Ağlar → SIM → VoLTE aç.
   (Gelmezse: bölge=Hindistan yap → yeniden başlat → VoLTE aç.)

> iPhone: `*#*#86583#*#*` YOK. iOS VoLTE'yi carrier bundle'a bağlar. Bu ağda
> attach + internet her zaman çalışır; VoLTE bazı model/iOS sürümlerinde çalıştı
> (bu oturumda iPhone'da VoLTE açıldı), ama garanti değildir.

---

## 7. ATTACH ve REGISTER DOĞRULAMA (loglardan)

### Attach oldu mu? (MME)
```bash
sudo docker logs --tail 30 mme 2>&1 | grep -iE "<IMSI>|Attach complete|Bearer added|Invalid APN"
```
- `Attach complete` + `Bearer added (EBI=5 ... internet)` + `Bearer added (EBI=6 ... ims)` = OK
- `Invalid APN[ia]` veya `Invalid APN[internet ]` = telefon yanlış APN gönderiyor;
  genelde birkaç denemede kendiliğinden düzelir (Kart 1 ve 2'de böyle oldu),
  ya da telefonda APN elle `internet` yapılır.

### IMS register oldu mu? (S-CSCF)
```bash
sudo docker logs -f scscf 2>&1 | grep -iE "<IMSI>|Auth succeeded|registered|200|User-Agent"
```
Beklenen: `Auth succeeded` → `SAR success - 200 response sent` →
`state="active" event="registered" expires="3600"`.

### Çağrı kuruluyor mu? (S-CSCF)
```bash
sudo docker logs -f scscf 2>&1 | grep -iE "INVITE|180|200 OK|ACK|Ringing|BYE"
```
Arama: bir telefondan diğerinin MSISDN'i çevrilir. INVITE → 180 Ringing → 200 OK → ACK.

> VoLTE/HD simgesinin telefonda görünmemesi REGISTER'ın başarısız olduğu anlamına
> GELMEZ — simge kozmetiktir, kanıt loglardır. (Bu oturumda simge görünmeden
> register loglarla kanıtlandı.)

---

## 8. eNB / RF (srsRAN — bu oturumdaki değerler)

- Band 5 (850 MHz), `dl_earfcn=2525`, `tx_gain=50`, `rx_gain=40`, `n_prb=50`, `tac=7`
- B210: `device_name` yorumlu (otomatik tespit). GPSDO yoksa `clock=external` yorumlu.
- USB 3.0 portu şart (SuperSpeed 5000M). FPGA imajı UHD sürümüne uygun olmalı.
- Bu değerler MÜHENDİSLİK KARARIDIR (band/güç ortama göre değişir); geri kalan
  config kaynak-temellidir.

---

## 9. SIM PROGRAMLAMA (sysmoISIM-SJA5)

- PLMN: MCC=001 MNC=01, USIM Type=OPc.
- **SQN check KAPATILMALI** (hem USIM hem ISIM) — yoksa auth MAC/Synch failure verir.
- Değerler makine-doğrulamayla (CSV) okunmalı; hex'i gözle okumak güvenilmez.

---

## 10. REFERANSLAR (taranıp doğrulandı)

- `open5gs.org/docs/tutorial/03-VoLTE-dockerized` — resmi VoLTE rehberi (APN, IPsec, pyHSS notları)
- `github.com/herlesupreeth/docker_open5gs` — README'de pyHSS 5-adım provisioning
- docker_open5gs branch: **master** (eski open5gs_hss_cx kaldırıldı), OS: Ubuntu 22.04
- IMS: Kamailio P/I/S-CSCF + rtpengine + pyHSS
