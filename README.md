# Özel VoLTE Test Ağı (Open5GS + srsRAN + IMS)

Gerçek COTS telefonlar arasında, tamamen kendi kontrolündeki bir 4G ağı üzerinden
**VoLTE (IMS üzerinden ses)** çağrısı kurmak için uçtan uca kurulum, otomasyon ve
dökümantasyon. İki Xiaomi/Redmi cihaz arasında ve bir iPhone üzerinde VoLTE register
+ çağrı canlı olarak doğrulanmıştır.

> Donanım: USRP B210 (+ opsiyonel GPSDO), sysmoISIM-SJA5 SIM'ler. Yazılım: Open5GS
> (EPC), srsRAN (eNB), Kamailio P/I/S-CSCF + rtpengine + pyHSS (IMS), Docker.

---

## Neden iki HSS? (En önemli kavram)

Bu kurulumda **iki ayrı abone veritabanı** vardır ve VoLTE için ikisine de abone
girilmesi ZORUNLUDUR:

| HSS | Erişim | Görev |
|-----|--------|-------|
| **open5gs HSS** | WebUI `:9999` / MongoDB | 4G **attach** + veri (internet) |
| **pyHSS** | Swagger `:8080/docs/` | **IMS / VoLTE** register |

Sadece WebUI'ye abone girersen telefon ağa girer ve internete çıkar, ama **VoLTE
register olmaz**. pyHSS provisioning yapılmazsa I-CSCF, S-CSCF adresini bulamaz
(`cxdx_get_server_name: Failed finding avp`) ve REGISTER asla tamamlanmaz. Bu, en
sık yapılan hatadır. Ayrıntı: [`DURUM.md`](DURUM.md) §2-3.

---

## Bu repo ne yapar, ne yapmaz (önce oku)

Bu repo, **çalışan bir Open5GS/IMS/srsRAN kurulumunu VoLTE'ye yapılandırır** —
sıfırdan derleme/kurulum aracı değildir. Net olmak gerekirse:

- ✅ **Yapar:** iki HSS'e abone ekler, tam VoLTE APN profilini kurar, pyHSS IMS
  provisioning'i otomatikler, servis başlat/durdur + kalıcılık, PLMN değiştirme,
  attach/register doğrulama, ve tüm bunların kanıtlanmış dökümantasyonu.
- ❌ **Yapmaz:** B210/UHD sürücülerini kurmaz, `docker_open5gs`'i senin yerine
  derlemez, SIM'i programlamaz (pySim'i kullanman gerekir).

**Ön koşullar:**
1. Ubuntu 22.04 host, Docker + docker compose.
2. `docker_open5gs` (herlesupreeth) klonlanmış ve `.env` ayarlanmış —
   [`docs/01_KURULUM.md`](docs/01_KURULUM.md) baştan sona anlatır.
3. USRP B210 + UHD sürücüleri çalışır durumda (`uhd_usrp_probe` geçer).
4. sysmoISIM-SJA5 SIM'ler + pySim programlayıcı.

Hiç Open5GS kurmadıysan: **önce [`docs/01_KURULUM.md`](docs/01_KURULUM.md)** (sıfırdan,
adım adım), sonra aşağıdaki hızlı başlangıç. Zaten 4G ağın varsa direkt B) yolunu izle.

---

## Hızlı başlangıç

### A) Sıfırdan kurulum
```bash
git clone https://github.com/NE-ON82/volte_open5gs_srsran
cd volte_open5gs_srsran/scripts
./kurulum.sh          # orkestrasyon: adımları sırayla yönlendirir
```
> `kurulum.sh` donanım/sürücü adımlarını **sana bırakır** (B210/UHD, docker_open5gs
> derleme). Tam sıfırdan rehber: [`docs/01_KURULUM.md`](docs/01_KURULUM.md).

### B) Zaten çalışan bir 4G ağın varsa (telefon internete çıkıyor) → VoLTE ekle
```bash
cd scripts
./tara_kur.sh          # mevcut yapıyı tarar, eksik VoLTE parçalarını ekler
```

### C) Abone ekle (hem EPC hem IMS, tek komut)
```bash
./volte add subscriber \
  --imsi 001010000000001 \
  --ki   0BDEB2CB463A5A5A29307A73F4FA0A86 \
  --opc  19D7004F5D2C16EB68968F90A082556C \
  --msisdn 0010000000001
```

### D) Telefonu hazırla — cihaz türüne göre

| Cihaz türü | VoLTE açma yöntemi | Test durumu |
|------------|--------------------|-------------|
| **Xiaomi / Redmi** | Dialer'a `*#*#86583#*#*` → "VoLTE carrier check disabled" → Ayarlar → Mobil Ağlar → SIM → VoLTE aç | ✅ Register + çağrı doğrulandı |
| **iPhone** | Ayarlar → Hücresel → Ses ve Veri → LTE + VoLTE aç (gizli kod yok) | ⚠️ Bazı model/iOS'ta açılır; carrier bundle'a bağlı |
| **Diğer Android** | Genel: Ayarlar → SIM/Ağ → VoLTE toggle. Bazı markalarda gizli kod/bölge hilesi gerekir | Değişken |

- Xiaomi'de VoLTE seçeneği yoksa: bölge geçici **Hindistan** → yeniden başlat → VoLTE görünür.
- VoLTE/HD **simgesinin görünmemesi** register'ın başarısız olduğu anlamına gelmez — kanıt loglardır.
- Tam adımlar ve doğrulama: [`docs/04_TELEFON_VOLTE.md`](docs/04_TELEFON_VOLTE.md)

### E) Doğrula
```bash
./volte check --imsi 001010000000001   # attach + register durumu
```

---

## SIM programlama — kritik nüanslar

VoLTE'nin çalışması için SIM ile ağdaki abone kaydı **birebir** eşleşmeli. Sık atlanan noktalar:

- **SQN check KAPALI olmalı** (hem USIM hem ISIM). Açık kalırsa auth `MAC failure` /
  `Synch failure` verir, attach başarısız olur. En sık auth sorunu budur.
- **USIM auth tipi = OPc** (OP değil). Ağ tarafıyla aynı tip kullanılmalı.
- **Ki/OPc'yi gözle okuma** — programlayıcının CSV/çıktısından makine ile al (0/O, 8/B karışır).
- **AMF, PLMN (MCC=001/MNC=01)** üç yerde de aynı: SIM, open5gs HSS, pyHSS.
- Ayrıntı ve araçlar (pySim): [`docs/02_SIM_PROGRAMLAMA.md`](docs/02_SIM_PROGRAMLAMA.md)

## Dökümanlar

| Döküman | İçerik |
|---------|--------|
| [`DURUM.md`](DURUM.md) | Kanıtlanmış son durum, tüm kritik değerler (önce bunu oku) |
| [`docs/00_MIMARI.md`](docs/00_MIMARI.md) | EPC + IMS bileşenleri, iki-HSS mantığı |
| [`docs/01_KURULUM.md`](docs/01_KURULUM.md) | Sıfırdan kurulum (Faz 0-1) |
| [`docs/02_SIM_PROGRAMLAMA.md`](docs/02_SIM_PROGRAMLAMA.md) | sysmoISIM, SQN kapatma |
| [`docs/03_ABONE_EKLEME.md`](docs/03_ABONE_EKLEME.md) | WebUI/MongoDB + pyHSS 5 adım + CLI |
| [`docs/04_TELEFON_VOLTE.md`](docs/04_TELEFON_VOLTE.md) | VoLTE aktifleştirme, attach/register doğrulama |
| [`docs/05_VOLTE_CAGRI.md`](docs/05_VOLTE_CAGRI.md) | Çağrı testi, log izleme |
| [`docs/06_SORUN_GIDERME.md`](docs/06_SORUN_GIDERME.md) | xfrm, Invalid APN[ia], IPsec clean_sa, vb. |
| [`docs/07_FAZLAR.md`](docs/07_FAZLAR.md) | Faz 0-7 yol haritası (VoLTE → SMS → VoNR) |
| [`docs/08_SERVIS_VE_PLMN.md`](docs/08_SERVIS_VE_PLMN.md) | volte start/stop, kalıcı docker, PLMN değiştirme |

---

## Script'ler

| Script | İşlev |
|--------|-------|
| `scripts/kurulum.sh` | Sıfırdan tam kurulum |
| `scripts/tara_kur.sh` | Mevcut 4G yapıyı tarar, eksik VoLTE parçalarını ekler |
| `scripts/volte` | CLI: abone (`add`/`del`/`list`/`check`) + servis (`start`/`stop`/`status`/`restart`/`enable-boot`) + `create plmn` |
| `scripts/lib/ayarlar.sh` | Merkezi config (HOST_IP, PLMN, dosya yolları, URL'ler) |
| `scripts/lib/pyhss_api.sh` | pyHSS 5-adım IMS provisioning fonksiyonları |
| `scripts/lib/webui_api.sh` | open5gs EPC abonesini MongoDB'ye tam APN ile yazar |
| `scripts/lib/servis.sh` | Docker yığını + eNB yönetimi, PC reboot'a dayanıklı kalıcılık |
| `scripts/lib/plmn.sh` | PLMN (MCC/MNC) değiştirme — tüm config'lerde, yedekli |

---

## Uyarı

Bu ağ yalnızca **kendi sahip olduğun donanım ve kapalı/lisanslı/test ortamında**
kullanılmalıdır. RF yayını yapmak çoğu ülkede lisans gerektirir. Test PLMN'i
(MCC=001, MNC=01) canlı şebekelerle çakışmayacak şekilde seçilmiştir. Sorumluluk
kullanıcıdadır.

---

> Antigravity ajanı için: herhangi bir script/döküman üretmeden önce
> [`AGENTS.md`](AGENTS.md) ve [`DURUM.md`](DURUM.md) okunmalıdır.
