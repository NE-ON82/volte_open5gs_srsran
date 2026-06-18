# 06 — Sorun Giderme

Bu kurulumda gerçekten karşılaşılan ve çözülen sorunlar. Her biri canlı sistemde
loglarla doğrulandı.

---

## 1. Telefon attach oluyor ama VoLTE register OLMUYOR

**Belirti:** `mme` logunda `Attach complete` var, telefon internete çıkıyor; ama
`scscf` logunda hiç `REGISTER ... <IMSI>` yok.

**En olası sebep: pyHSS IMS provisioning eksik.** Kontrol:
```bash
sudo docker logs --tail 20 icscf 2>&1 | grep -i "Failed finding avp"
```
Şunu görüyorsan kesinleşir:
```
cxdx_get_server_name: Failed finding avp
cxdx_get_capabilities: Failed finding avp
```
I-CSCF, pyHSS'ten S-CSCF adresini alamıyor → REGISTER S-CSCF'e ulaşmıyor.

**Çözüm:** pyHSS'e abone gir (5 adım). Bkz. [`03_ABONE_EKLEME.md`](03_ABONE_EKLEME.md)
veya `./volte add subscriber ...`. Özellikle **Adım 4 (ims_subscriber)** scscf/
scscf_realm alanlarını sağlar; asıl eksik genelde budur.

---

## 2. `Invalid APN[ia]` / `Invalid APN[internet ]` (MME)

**Belirti:** `mme` logunda:
```
[esm] ERROR: Invalid APN[ia]
Removed Session: UE IMSI:[<IMSI>] APN:[Unknown]
```
Telefon yanlış/eksik APN gönderiyor (`ia`, ya da sonunda boşlukla `internet `).

**Çözüm:** Çoğu zaman telefon birkaç denemede doğru APN'i (`internet`) bulur ve
kendiliğinden `Attach complete` olur — bekle. Düzelmezse:
- Telefonda APN'i elle tanımla: ad `internet`, APN `internet`, tür `default`.
- Uçak modu aç-kapa.

> Bu hata bu oturumda hem Kart 1 hem Kart 2'de görüldü; ikisi de birkaç denemede
> kendiliğinden geçti.

---

## 3. P-CSCF `clean_sa ... No data available` — IPsec hatası (ZARARSIZ)

**Belirti:** `pcscf` logunda her REGISTER'da:
```
ims_ipsec_pcscf: clean_sa(): Error sending delete SAs command via netlink socket: No data available
ipsec_cleanall(): Error cleaning IPSec Security associations during startup
```

**Bu ZARARSIZDIR.** P-CSCF başlangıçta eski IPsec SA'larını temizlemeye çalışır;
silinecek SA olmadığı için "No data available" der. REGISTER'ı **engellemez** —
P-CSCF register'ı yine de RELAY eder. Bu mesaja bakıp asıl sorunu (genelde pyHSS,
bkz §1) atlama.

> Asıl IPsec SA kurulumu challenge'dan SONRA olur ve başarılı register'da
> `Security-Client=ipsec-3gpp ...` + `ipsec_forward: new destination` satırları görülür.

---

## 4. xfrm kernel modülleri (IPsec ön koşulu)

IMS IPsec için host'ta xfrm modülleri yüklü olmalı. Yükle:
```bash
sudo modprobe xfrm_user esp4 xfrm4_tunnel tunnel4 ah4
lsmod | grep -E "xfrm|esp|ah"
sudo ip xfrm state && echo "xfrm OK"   # hatasız boş liste dönmeli
```
P-CSCF servisinde `privileged: true` + `cap_add: [NET_ADMIN]` olmalı (docker_open5gs'te
vardır). Modülleri yükledikten sonra P-CSCF'i yeniden oluştur:
```bash
cd <docker_open5gs_dizini>
sudo docker compose -f 4g-volte-deploy.yaml stop pcscf
sudo docker compose -f 4g-volte-deploy.yaml rm -f pcscf
sudo docker compose -f 4g-volte-deploy.yaml up -d pcscf
```
> `restart` yerine `stop+rm+up` — container namespace'i tazelenir.

---

## 5. pyHSS PUT → `400 / operation_log item_id cannot be null`

**Belirti:** Swagger'da APN/AUC eklerken:
```
"result": "Failed",
"reason": "MySQLdb.OperationalError (1048, Column 'item_id' cannot be null") ... INSERT INTO operation_log ...
```

**Sebep:** Gönderdiğin payload'da Swagger'ın örnek fazla alanları (`apn_id:0`,
`ip_version:0`, vb.) duruyor.

**Çözüm:** Payload kutusunu temizle, **yalnızca gerekli alanları** gönder:
```json
{"apn": "internet", "apn_ambr_dl": 0, "apn_ambr_ul": 0}
```
`scripts/lib/pyhss_api.sh` zaten sadece gerekli alanları gönderir.

---

## 6. B210 / UHD / FPGA

- **USB 3.0 şart:** `lsusb -t` ile cihazın 5000M/SuperSpeed portta olduğunu doğrula.
  USB 2.0'da throughput yetmez.
- **FPGA imajı UHD sürümüyle uyumlu olmalı.** Yanlış imaj `fx3 is in state 5` benzeri
  hata verir. `uhd_images_downloader` ile uygun imajı al; özel imaj kullanılıyorsa
  yolu container'a mount et.
- Doğrulama: `uhd_usrp_probe` → "Operating over USB 3" + "Register loopback test passed".

---

## 7. SIM auth başarısız (MAC / Synch failure)

**Belirti:** MME/HSS'te auth reddi; attach tamamlanmaz.

**Sebep:** SIM'de SQN check açık veya Ki/OPc yanlış.

**Çözüm:**
- sysmoISIM'de **SQN check'i KAPAT** (hem USIM hem ISIM).
- Ki/OPc'yi gözle değil, makine-doğrulamayla (CSV/programlayıcı çıktısı) gir.
- pyHSS ve open5gs'teki Ki/OPc/AMF, SIM'dekiyle birebir aynı olmalı.

---

## 8. iPhone internet alıyor ama VoLTE yok

**Bu beklenen olabilir.** iOS VoLTE'yi carrier bundle'a bağlar; internet ise
carrier-bağımsızdır. Test ağında iPhone genelde attach + internet yapar, VoLTE bazı
model/sürümlerde açılır, bazılarında açılmaz. `scscf` logunda iPhone'dan REGISTER
görünmüyorsa iOS IMS'i denemiyordur — ağ sorunu değildir.

---

## Hızlı teşhis komutları

```bash
# Attach:
sudo docker logs --tail 30 mme   2>&1 | grep -iE "<IMSI>|Attach complete|Invalid APN"
# I-CSCF server_name (pyHSS eksik mi):
sudo docker logs --tail 20 icscf 2>&1 | grep -i "Failed finding avp"
# Register:
sudo docker logs -f  scscf 2>&1 | grep -iE "<IMSI>|Auth succeeded|registered|200"
# Çağrı:
sudo docker logs -f  scscf 2>&1 | grep -iE "INVITE|180|200 OK|ACK|BYE"
# Konteyner durumu:
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```
