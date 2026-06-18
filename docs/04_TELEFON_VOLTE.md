# 04 — Telefonda VoLTE Aktifleştirme ve Doğrulama

Bu döküman: telefonu test ağına nasıl bağlarsın, VoLTE'yi nasıl açarsın, attach ve
IMS register'ın gerçekten olup olmadığını nasıl anlarsın.

---

## 1. Xiaomi / Redmi — uygulama olmadan VoLTE açma

Xiaomi/Redmi cihazlarda VoLTE, operatör beyaz-liste kontrolüne takılır. Bu kontrolü
gizli servis kodu ile kapatabilirsin (ek uygulama gerekmez):

1. **Telefon (dialer) uygulamasını aç**, şu kodu çevir:
   ```
   *#*#86583#*#*
   ```
   Ekranda kısa bir bildirim çıkar: **"VoLTE carrier check was disabled"**.
   - Bildirim çıkmazsa kodu bir kez daha çevir (bazen ikinci denemede çıkar).
   - Bu işlem beyaz-liste kontrolünü kapatır; geri açmak için aynı kod tekrar çevrilir
     ("...was enabled" der).

2. **VoLTE anahtarını aç:**
   `Ayarlar → Mobil Ağlar (SIM kartlar & mobil ağlar) → [SIM] → VoLTE` → Aç.

3. **VoLTE seçeneği görünmüyorsa:**
   - `Ayarlar → Bölge` → geçici olarak **Hindistan** yap → telefonu yeniden başlat
   - VoLTE anahtarı görünür hale gelir → aç → istersen bölgeyi geri al.

---

## 2. iPhone

iPhone'da `*#*#86583#*#*` **çalışmaz**. iOS, VoLTE'yi operatör profiline (carrier
bundle) bağlar.

- `Ayarlar → Hücresel → [SIM] → Ses ve Veri → LTE` seçili + **VoLTE Aç**.
- `Ayarlar → Hücresel → Hücresel Veri Ağı` görünüyorsa APN: `internet`.
- **Attach + internet her zaman çalışır.** VoLTE bazı model/iOS sürümlerinde test
  ağında açılır, bazılarında açılmaz (carrier bundle kısıtı). Bu kurulumda iPhone'da
  VoLTE register edildi, ama her iPhone'da garanti değildir.

---

## 3. Hangi kart hangi telefonda? (karışıklığı önle)

VoLTE çağrısı **iki ayrı uç** gerektirir; her uçta farklı bir SIM olmalı:
- Telefon A → Kart 1 (örn. MSISDN `0010000000001`)
- Telefon B → Kart 2 (örn. MSISDN `0010000000002`)

Bir kartı bir telefondan çıkarıp başka telefona takarsan, eski telefon artık o numarayla
ağda değildir. Çağrı testi için iki telefonda iki farklı kartın aynı anda kayıtlı
olması gerekir.

---

## 4. ATTACH oldu mu? (4G bağlantı — MME logu)

VoLTE'den önce telefon ağa **attach** olmalı (veri katmanı). Host'ta:

```bash
sudo docker logs --tail 30 mme 2>&1 | grep -iE "<IMSI>|Attach complete|Bearer added|Invalid APN"
```

**Başarılı attach:**
```
[<IMSI>] Attach complete
Bearer added (EBI=5 IMSI=<IMSI>)     ← internet
Bearer added (EBI=6 IMSI=<IMSI>)     ← ims
```

**Sık görülen geçici hata — `Invalid APN[ia]` veya `Invalid APN[internet ]`:**
Telefon ilk denemelerde yanlış/eksik APN gönderir. Genelde birkaç saniye içinde
kendiliğinden düzelir (telefon doğru APN'i bulur). Düzelmezse:
- Telefonda `Ayarlar → ... → APN` → yeni APN: ad `internet`, APN `internet`, tür `default`
- Uçak modunu aç-kapa yap.

---

## 5. IMS REGISTER oldu mu? (VoLTE — S-CSCF logu)

Attach olduktan sonra, VoLTE açıksa telefon IMS'e register etmeli. Host'ta canlı izle:

```bash
sudo docker logs -f scscf 2>&1 | grep -iE "<IMSI>|Auth succeeded|registered|200|User-Agent"
```

**Başarılı register akışı:**
```
SCSCF: REGISTER sip:<IMSI>@ims.mnc001.mcc001.3gppnetwork.org
ALGORITHM IS [AKAv1-MD5]  and User-Agent is [Xiaomi_Redmi Note 12_...]
Auth succeeded
SAR success - 200 response sent from module
state="active" event="registered" expires="3600"
```

Bunları görüyorsan telefon **VoLTE'ye kayıtlı** demektir.

### Telefonda VoLTE/HD simgesi görünmüyor — sorun mu?
**Hayır.** Simge kozmetiktir ve çoğu telefon onu yalnızca çağrı kurulabilir
durumda (ikinci uç da kayıtlıyken) gösterir. **Kanıt, yukarıdaki S-CSCF
loglarıdır**, simge değil. Bu kurulumda simge görünmeden register loglarla
kanıtlandı.

---

## 6. Register başlamıyorsa hızlı kontrol

`scscf` logunda hiç `REGISTER ... <IMSI>` görünmüyorsa, sırayla bak:

1. **VoLTE telefonda açık mı?** (`*#*#86583#*#*` + toggle)
2. **Attach oldu mu?** (§4) — attach yoksa register de olmaz.
3. **pyHSS'te IMS subscriber var mı?** En sık sebep budur:
   ```bash
   sudo docker logs --tail 20 icscf 2>&1 | grep -i "Failed finding avp"
   ```
   `cxdx_get_server_name: Failed finding avp` görüyorsan → pyHSS provisioning eksik
   → [`03_ABONE_EKLEME.md`](03_ABONE_EKLEME.md) §pyHSS veya `./volte add subscriber`.
4. **xfrm modülleri yüklü mü?** (host) — [`06_SORUN_GIDERME.md`](06_SORUN_GIDERME.md).

> Uçak modunu aç-kapa yapmak yeni bir register denemesini tetikler — değişiklik
> sonrası en hızlı yeniden deneme yöntemi budur.
