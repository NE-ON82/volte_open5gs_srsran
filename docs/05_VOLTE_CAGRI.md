# 05 — VoLTE Çağrı Testi

İki telefon da IMS'e register olduktan sonra gerçek VoLTE çağrısı.

## Ön koşul

İki ayrı telefonda iki ayrı kart, **ikisi de aynı anda** register olmalı. Kontrol:
```bash
sudo docker logs --tail 80 scscf 2>&1 | grep -iE "registered|<IMSI_1>|<IMSI_2>" | tail
```
Her iki IMPU için `State: [registered]` görmelisin. Görmüyorsan ilgili telefonu
uçak modu aç-kapa yap (yeniden register).

## Çağrıyı başlat

Telefon A'dan, Telefon B'nin **MSISDN**'ini çevir ve ara:
- A (Kart 1) → çevir `0010000000002` (Kart 2)
- veya B (Kart 2) → çevir `0010000000001` (Kart 1)

## Canlı izle (S-CSCF)

```bash
sudo docker logs -f scscf 2>&1 | grep -iE "INVITE|100|180|200 OK|ACK|BYE|Ringing"
```

Başarılı çağrı akışı:
```
INVITE sip:0010000000002@ims.mnc001.mcc001.3gppnetwork.org
100 Trying
180 Ringing          ← karşı telefon ÇALIYOR
200 OK               ← cevaplandı
ACK                  ← çağrı kuruldu, ses başladı
... (konuşma) ...
BYE                  ← kapatıldı
```

## Medya (ses) kontrolü

Ses RTP olarak `rtpengine` üzerinden akar:
```bash
sudo docker logs --tail 30 rtpengine 2>&1 | tail
```
Çağrı sırasında her iki yönde RTP akışı görülmeli. Tek yönlü ses genelde NAT/SDP
veya QCI1 GBR sorununa işaret eder (bkz. APN ayarları).

## İpuçları

- **Çağrı hemen düşüyorsa:** QCI1/2 GBR/MBR'ı `128/128` yap (unlimited bırakma).
  WebUI/MongoDB APN profilini kontrol et (DURUM.md §5).
- **Karşı telefon çalmıyor ama INVITE görünüyor:** alıcı register değil ya da
  contact süresi dolmuş — alıcıyı uçak modu aç-kapa.
- **INVITE hiç görünmüyor:** arayan telefon VoLTE'den değil CS'den (2G/3G) aramaya
  düşmüş olabilir; VoLTE açık ve register olduğundan emin ol.
- **Video:** VoLTE register'da `video` özelliği geliyorsa görüntülü çağrı da
  denenebilir (her iki uç desteklemeli).
