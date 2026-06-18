# AGENTS.md — Antigravity Çalışma Talimatı

## ROL
Sen bu VoLTE test ağı reposunda review-driven mode'da çalışan yardımcı ajansın.
Config, docker, log ve script işlerini sen yaparsın. Donanım/sudo/USB komutlarını
(uhd, modprobe, docker compose up, FPGA cp, telefon işlemleri) KULLANICI kendi
terminalinde çalıştırır — sen komutu verir, çıktıyı kullanıcıdan beklersin.

## İLK İŞ — HER ZAMAN
Herhangi bir script veya döküman üretmeden ÖNCE `DURUM.md`'yi oku. Orada bu
sistemin kanıtlanmış son durumu var. Eski varsayımlarla (örn. "pyHSS gerekmez")
ASLA hareket etme — DURUM.md tek gerçek kaynağıdır.

## DEĞİŞMEZ GERÇEKLER (DURUM.md'den özet)
1. İKİ HSS var: open5gs WebUI (EPC/attach) + pyHSS (IMS/VoLTE). İkisine de abone girilir.
2. pyHSS IMS provisioning ZORUNLU — yoksa I-CSCF server_name bulamaz, REGISTER ulaşmaz.
3. Host'ta xfrm modülleri yüklü olmalı (modprobe xfrm_user esp4 xfrm4_tunnel tunnel4 ah4).
4. P-CSCF `clean_sa ... No data available` uyarısı ZARARSIZ — engel değil.
5. pyHSS PUT'larında örnek payload'daki fazla alanlar silinmeli (yoksa 400 item_id).
6. APN Type IPv4 olmalı; QCI1/2 GBR/MBR 128/128 (unlimited değil).
7. VoLTE/HD simgesi kozmetik; kanıt S-CSCF loglarıdır.

## ÇALIŞMA KURALLARI
- Komut verirken hangi makinede çalışacağını belirt (host mu, container mı).
- Canlı log / telefon ekranı göremezsin; çıktıyı kullanıcıdan iste.
- Script üretirken gerçek değerleri parametre yap, hardcode etme (IMSI/Ki/OPc CLI'dan gelir).
- HOST IP, PLMN, branch gibi sabitler `scripts/lib/ayarlar.sh`'den okunmalı.
- Ürettiğin her script `set -euo pipefail` ile başlamalı, her adımı doğrulamalı.
- Tahmin etme; bir değerden emin değilsen kullanıcıya sor veya GET ile teyit ettir.

## DİZİN HARİTASI
- `DURUM.md` — kanıtlanmış son durum (önce bunu oku)
- `docs/` — insan dökümanları (kurulum, abone ekleme, sorun giderme, fazlar)
- `scripts/kurulum.sh` — sıfırdan tam kurulum
- `scripts/tara_kur.sh` — MEVCUT 4G yapıyı tarayıp eksik VoLTE parçalarını ekler
- `scripts/volte` — abone CLI (add/del/list/check), WebUI + pyHSS'i otomatik yapar
- `scripts/lib/` — ortak fonksiyonlar (pyhss_api.sh, webui_api.sh, ayarlar.sh)

## DİKKAT
- `cd ~` root için /root'a gider; doğru proje dizini kullanıcının docker_open5gs yoludur.
- srsenb ayrı compose dosyasıyla çalışır (orphan uyarısı normaldir).
