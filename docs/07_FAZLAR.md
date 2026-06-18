# 07 — Fazlar (Yol Haritası)

Projenin aşamalı planı. ✅ tamamlanan, ⏳ sıradaki, ⬜ planlanan.

---

## ✅ Faz 0 — Altyapı

- Host hazırlığı (Ubuntu 22.04), Docker, `docker_open5gs` (branch: **master**).
- `.env` yapılandırması, 16 container ayağa kalkması, DNS.
- UHD + B210: USB 3.0 doğrulama, uygun FPGA imajı, `uhd_usrp_probe` testi.
- sysmoISIM-SJA5 programlama (PLMN 001/01, USIM Type=OPc, **SQN check kapalı**).

## ✅ Faz 1 — VoLTE (temel hedef)

- eNB yayını (Band 5 / 850 MHz, EARFCN 2525).
- Telefon attach (internet + ims bearer).
- open5gs HSS abone kaydı (tam VoLTE APN: internet QCI9 + ims QCI5/1/2, IPv4, GBR 128/128).
- **pyHSS IMS provisioning** (5 adım: apn/auc/subscriber/ims_subscriber).
- Host xfrm modülleri + P-CSCF IPsec.
- **IMS REGISTER → 200 OK** (Auth succeeded, state=registered).
- **VoLTE çağrısı** iki telefon arası kuruldu (INVITE → 180 → 200 OK → ses).
- Doğrulanan cihazlar: Redmi Note 12, Xiaomi 11T Pro, iPhone (register oldu).

> Bu repodaki script ve dökümanlar Faz 0-1'i tekrarlanabilir kılar.

---

## ⏳ Faz 2 — SMS over IMS / SGs

- IMS üzerinden SMS (MESSAGE metodu) veya SGs ile CS fallback SMS.
- iFC (initial Filter Criteria) ile MESSAGE tetikleme.
- Test: iki telefon arası SMS gönder/al.

## ⬜ Faz 3 — Çağrı senaryoları (genişletme)

- Çağrı bekletme, konferans, görüntülü çağrı (video bearer).
- Çağrı sırasında kalite/medya analizi (rtpengine istatistikleri).
- Birden fazla eşzamanlı çağrı.

## ⬜ Faz 4 — VoNR (5G ses)

- 5G çekirdek (`sa-vonr-deploy.yaml`) + gNB (`srsgnb.yaml`).
- 5QI 1 (ses) bearer, IMS'in 5G'ye bağlanması.
- Test: 5G SA üzerinden VoNR çağrısı.

## ⬜ Faz 5 — Dayanıklılık / otomasyon

- `kurulum.sh` / `tara_kur.sh` ile tam otomatik kurulum.
- `volte` CLI ile toplu abone yönetimi.
- Sağlık kontrolleri, otomatik log toplama.

## ⬜ Faz 6 — Ölçüm / analiz

- KPI'lar: register süresi, çağrı kurulum gecikmesi, kayıp/jitter.
- Wireshark/pcap ile SIP+RTP analizi.

## ⬜ Faz 7 — Endüstrileştirme

- Çoklu hücre / çoklu kullanıcı ölçeklendirme.
- İzleme paneli, log/metric toplama.
- Dokümante edilmiş, sürümlenmiş dağıtım.

---

> Faz 0-1 için tüm kritik değerler ve çözülmüş sorunlar [`../DURUM.md`](../DURUM.md)
> ve [`06_SORUN_GIDERME.md`](06_SORUN_GIDERME.md) içindedir.
