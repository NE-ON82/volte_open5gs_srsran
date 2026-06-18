# 03 — Abone Ekleme (EPC + IMS)

VoLTE için **iki yere** abone girilir: open5gs HSS (EPC/attach) ve pyHSS (IMS/VoLTE).
Bu döküman üç yöntemi de gösterir: tek komut CLI (önerilen), WebUI/MongoDB (EPC),
pyHSS Swagger (IMS).

> Neden iki yer? Bkz. [`00_MIMARI.md`](00_MIMARI.md) ve [`DURUM.md`](../DURUM.md) §2.
> Sadece birine girersen ya attach olmaz ya VoLTE olmaz.

---

## Yöntem 1 — Tek komut (önerilen): `volte` CLI

Hem EPC (MongoDB) hem IMS (pyHSS 5 adım) işini tek seferde yapar:

```bash
cd scripts
./volte add subscriber \
  --imsi 001010000000001 \
  --ki   0BDEB2CB463A5A5A29307A73F4FA0A86 \
  --opc  19D7004F5D2C16EB68968F90A082556C \
  --msisdn 0010000000001
```

Diğer komutlar:
```bash
./volte list subscribers              # her iki HSS'teki aboneler
./volte del subscriber --imsi <IMSI>  # her ikisinden siler
./volte check --imsi <IMSI>           # attach + register durumu (loglardan)
```

> CLI değerleri `scripts/lib/ayarlar.sh`'den okur (HOST_IP, PLMN, scscf adresleri).
> Ortamın farklıysa önce o dosyayı düzenle.

---

## Yöntem 2 — EPC abonesi: MongoDB'ye doğrudan (tam APN)

open5gs WebUI'nin REST'i ve `open5gs-dbctl` tam VoLTE APN yapısını eksiksiz
kuramaz. En sağlamı, WebUI'nin de yazdığı yere — MongoDB `open5gs.subscribers`
koleksiyonuna — doğrudan yazmaktır. `scripts/lib/webui_api.sh` bunu yapar:

```bash
source scripts/lib/ayarlar.sh
source scripts/lib/webui_api.sh
webui_saglik
webui_abone_ekle 001010000000001 \
  0BDEB2CB463A5A5A29307A73F4FA0A86 \
  19D7004F5D2C16EB68968F90A082556C \
  0010000000001
```

Bu, aşağıdaki **tam VoLTE APN yapısını** kurar (DURUM.md §5):

| APN | Tip | QCI | ARP | AMBR / GBR |
|-----|-----|-----|-----|-----------|
| internet | IPv4 | 9 | 8 | 1 Gbps |
| ims | IPv4 | 5 | 1 | 3850/1530 Kbps |
| ims | IPv4 | 1 | 2 | GBR/MBR 128/128 Kbps |
| ims | IPv4 | 2 | 4 | GBR/MBR 128/128 Kbps |

> **Kritik:** Tip **IPv4** (IPv6 VoLTE'yi bozar). QCI1/2 GBR/MBR **128/128**
> (unlimited değil — çağrı kopmasını önler).

### Şema doğrulama (önemli)
MongoDB doküman şeması (özellikle AMBR `unit` kodları) open5gs sürümüne göre küçük
farklılık gösterebilir. Emin olmak için **WebUI'den elle bir abone ekleyip** şemayı
gör, sonra script'i ona göre teyit et:
```bash
docker exec mongo mongosh --quiet open5gs --eval 'db.subscribers.findOne({}, {security:0})'
```

### WebUI ile elle (görsel)
Tarayıcıdan `http://<HOST_IP>:9999` (admin / 1423). Subscriber → Add. Yukarıdaki
tabloyu birebir gir. 4 oturum: internet (QCI9) + 3× ims (QCI5/1/2). Her birinde
Type=IPv4. QCI1/2'de GBR/MBR 128/128.

---

## Yöntem 3 — IMS abonesi: pyHSS Swagger (5 adım)

pyHSS provisioning olmadan VoLTE register OLMAZ (I-CSCF S-CSCF'i bulamaz).
Tarayıcıdan `http://<HOST_IP>:8080/docs/`. Sıra önemlidir.

> **Her PUT'ta**, Swagger'ın örnek payload'ındaki FAZLA ALANLARI SİL (apn_id:0,
> ip_version:0, qci:0 vb.). Sadece aşağıdaki alanları gönder. Aksi halde
> `operation_log / item_id cannot be null` (400) hatası gelir.

### Adım 1 — APN (bir kez, iki çağrı): `PUT /apn/`
```json
{"apn": "internet", "apn_ambr_dl": 0, "apn_ambr_ul": 0}
```
```json
{"apn": "ims", "apn_ambr_dl": 0, "apn_ambr_ul": 0}
```
Dönen `apn_id` değerlerini NOT AL (örn. internet=2, ims=3 — ortamda değişebilir).

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

### Adım 4 — IMS_SUBSCRIBER (her kart): `PUT /ims_subscriber/`  ← en kritik
```json
{"imsi": "<IMSI>", "msisdn": "<MSISDN>", "sh_profile": "string",
 "scscf_peer": "scscf.ims.mnc001.mcc001.3gppnetwork.org",
 "msisdn_list": "[<MSISDN>]", "ifc_path": "default_ifc.xml",
 "scscf": "sip:scscf.ims.mnc001.mcc001.3gppnetwork.org:6060",
 "scscf_realm": "ims.mnc001.mcc001.3gppnetwork.org"}
```

> Bu son adım, I-CSCF'in aradığı S-CSCF "server_name"ini sağlar. Girilince REGISTER
> S-CSCF'e ulaşır ve 200 OK alınır.

### pyHSS — terminalden (script ile)
```bash
source scripts/lib/ayarlar.sh
source scripts/lib/pyhss_api.sh
pyhss_saglik
pyhss_kart_ekle 001010000000001 \
  0BDEB2CB463A5A5A29307A73F4FA0A86 \
  19D7004F5D2C16EB68968F90A082556C \
  0010000000001
```
Bu fonksiyon APN'leri garanti eder, auc→subscriber→ims_subscriber'ı sırayla yapar,
ID'leri otomatik yakalar.

---

## Ekledikten sonra

1. Telefonu uçak modu aç-kapa (yeni attach + register).
2. Doğrula: [`04_TELEFON_VOLTE.md`](04_TELEFON_VOLTE.md) §4-5 veya `./volte check --imsi <IMSI>`.
