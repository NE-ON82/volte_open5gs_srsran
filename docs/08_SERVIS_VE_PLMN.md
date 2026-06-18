# 08 — Servis Yönetimi ve PLMN Değiştirme

## A) Yığını başlatma / durdurma / kalıcı yapma

PC kapanınca/yeniden başlayınca container'ların kendiliğinden gelmesini ve eNB
yayınını tek komutla açıp kapatmayı sağlar.

### Komutlar
```bash
volte start          # EPC+IMS (yoksa) + eNB yayınını başlat
volte stop           # sadece eNB'yi durdur (yayını kes, çekirdek ayakta kalır)
volte stop --all     # her şeyi durdur (çekirdek dahil)
volte restart        # eNB'yi yeniden başlat
volte status         # ne çalışıyor
```

### PC reboot'a dayanıklılık (kalıcı container'lar)
Sorun: PC kapanınca tüm docker'lar kapanıyor, srsLTE'yi açsan bile çekirdek
container'ları tek tek başlatman gerekiyor. Çözüm: container'lara
`restart=unless-stopped` politikası ver. Böylece sen `volte stop --all` demedikçe
(PC reboot dahil) ayakta kalırlar.

```bash
# Bir kez çalıştır (çekirdek ayaktayken):
volte start
volte enable-boot          # çalışan container'lara unless-stopped uygular
sudo systemctl enable docker   # docker servisi de boot'ta açık olsun
```

Bundan sonra:
- PC yeniden başladığında **çekirdek (EPC+IMS) kendiliğinden gelir**.
- Sadece eNB'yi (RF yayını) `volte start` / `volte stop` ile yönetirsin.

> **eNB neden otomatik başlamıyor?** RF yayınını boot'ta otomatik açmak genelde
> istenmez (USRP bağlı olmayabilir, RF emisyonu kontrollü olmalı). İstersen:
> `sudo docker update --restart unless-stopped srsenb`

### Geri alma
```bash
volte disable-boot   # restart politikasını 'no'ya çeker
```

---

## B) PLMN değiştirme — `volte create plmn`

PLMN (MCC/MNC) **çok sayıda dosyaya dağılır**. Elle değiştirmek S1 Setup Failure,
DNS çözümleme hatası ve sessiz IMS arızalarına yol açar. Bu komut hepsini tek yerden,
**yedek alarak** değiştirir.

```bash
volte create plmn --mcc 286 --mnc 11
volte create plmn --mcc 286 --mnc 11 --dry-run  # (Sadece hangi dosyaların değişeceğini raporlar)
```

### Neyi değiştirir
- **MME config** (`mme.yaml`) — plmn_id (mcc/mnc)
- **eNB** (`enb.conf`, `rr.conf`) — mcc/mnc
- **.env** — MCC/MNC değişkenleri
- **DNS zone dosyaları** — `epc.mncXX.mccYYY` ve `ims.mncXX.mccYYY` domain'leri
- **IMS/pyHSS config'leri** — `ims.mncXX.mccYYY.3gppnetwork.org` domain'i

Değiştirdiği her dosyanın yedeğini `${OPEN5GS_DIR}/.plmn_yedek/` altına alır.

### ⚠️ KRİTİK UYARILAR

**1. MNC hane sayısı (2 vs 3):**
`01` ve `11` iki hanelidir → domain `mnc01`, `mnc11`. Ama `110` üç hanelidir →
`mnc110`. Hane sayısı değişirse **domain formatı her yerde değişir** (DNS, Diameter
identity, IMS realm). Komut bunu algılar ve uyarır; o durumda elle kontrol şart.

**2. SIM kartlar:** Bu komut SIM'i **değiştiremez**. Yeni PLMN'e geçtikten sonra
SIM'leri de yeni MCC/MNC'ye programlamalısın — yoksa telefon attach **olmaz**.

**3. Diameter/freeDiameter:** Bazı kurulumlarda Diameter identity ve sertifikalar
domain'e bağlıdır. MNC hane sayısı değişirse bunların yeniden üretilmesi gerekebilir.

**4. Aboneler:** IMS domain değişince pyHSS'teki `ims_subscriber` scscf/realm
değerleri eskir. Aboneleri yeni domain'le tekrar gir (`volte add subscriber ...`),
ve `ayarlar.sh`'deki `IMS_DOMAIN`/`SCSCF_*` değerlerini güncelle.

### Önerilen sıra
```bash
volte stop --all                      # 1. yığını durdur
volte create plmn --mcc 286 --mnc 11  # 2. PLMN'i değiştir (onay ister)
# 3. ayarlar.sh'de MCC/MNC/IMS_DOMAIN güncelle
# 4. SIM'leri yeni PLMN'e programla
volte start                           # 5. yığını başlat
sudo docker logs -f mme | grep -iE "S1|PLMN|Attach"   # 6. doğrula
# 7. aboneleri tekrar ekle (yeni IMS domain)
volte add subscriber --imsi <yeni IMSI> --ki ... --opc ... --msisdn ...
```

### Sorun olursa geri alma
Yedekler `${OPEN5GS_DIR}/.plmn_yedek/` altında zaman damgalı durur. İlgili dosyayı
oradan geri kopyala.

> **Format notu:** Bu komut yaygın docker_open5gs dosya formatlarını hedefler
> (`mcc:`/`mnc:`, `mcc =`/`mnc =`, `mncXX.mccYYY`, `MCC=`/`MNC=`). Sürümünde format
> farklıysa "PLMN izi bulunamadı" uyarısı verir; o dosyayı elle düzelt. Çalıştırma
> sonrası raporu oku — hangi dosyaların güncellendiği listelenir.
