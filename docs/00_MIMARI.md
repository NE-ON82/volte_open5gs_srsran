# 00 — Mimari

## Genel bakış

```
   ┌──────────┐   LTE-Uu    ┌──────────┐   S1     ┌─────────────── EPC ───────────────┐
   │ Telefon  │◄──────────► │   eNB    │◄───────► │  MME ─ HSS(open5gs) ─ SGW-C/U ─ ... │
   │ (UE+SIM) │   (USRP)    │ (srsRAN) │          │              │  PGW-C/U(SMF/UPF)   │
   └────┬─────┘             └──────────┘          └──────────────┼──────────────────────┘
        │                                                        │ (ims APN / bearer)
        │  SIP/IMS (internet+ims bearer üzerinden)               ▼
        │                                          ┌──────────── IMS ───────────────────┐
        └─────────────────────────────────────────►│ P-CSCF ─ I-CSCF ─ S-CSCF ─ pyHSS   │
                                                    │           rtpengine (RTP medya)    │
                                                    └─────────────────────────────────────┘
```

## Katmanlar

**1. RF / Erişim (eNB)**
srsRAN eNB, USRP B210 ile LTE hücresi yayınlar. Telefon buraya bağlanır (RACH →
RRC). Band/EARFCN/güç ortama göre ayarlanır (bkz. DURUM.md §8).

**2. EPC (Çekirdek — Open5GS)**
- **MME:** attach, kimlik doğrulama orkestrasyonu, bearer yönetimi.
- **HSS (open5gs):** abone DB'si (Ki/OPc + APN profili). **EPC attach için.**
- **SGW/PGW (SMF/UPF):** kullanıcı düzlemi; internet ve ims APN bearer'ları.

**3. IMS (Ses — Kamailio + pyHSS)**
- **P-CSCF:** UE'nin ilk temas noktası; IPsec (xfrm) burada kurulur.
- **I-CSCF:** gelen REGISTER için pyHSS'e sorar, doğru S-CSCF'i bulur.
- **S-CSCF:** asıl register/auth (AKAv1-MD5), oturum kontrolü (INVITE).
- **pyHSS:** IMS abone DB'si (IMPI/IMPU + S-CSCF ataması). **VoLTE için.**
- **rtpengine:** çağrı sırasında RTP ses medyasını taşır.

## İki HSS — neden ayrı?

| | open5gs HSS | pyHSS |
|--|-------------|-------|
| Protokol | Diameter S6a | Diameter Cx/Sh |
| Sorumluluk | EPC attach (4G bağlanma) | IMS register (VoLTE) |
| Erişim | WebUI :9999 / MongoDB | Swagger :8080 |
| Yoksa | Telefon ağa giremez | Attach olur, VoLTE olmaz |

REGISTER akışı (basitleştirilmiş):
```
UE → P-CSCF → I-CSCF → (pyHSS UAR: hangi S-CSCF?) → S-CSCF → (pyHSS MAR: auth vektörü)
   → 401 challenge → UE ikinci REGISTER (IPsec) → S-CSCF Auth succeeded → 200 OK
```
pyHSS'te IMS subscriber yoksa I-CSCF "server_name" alamaz, zincir S-CSCF'e ulaşamaz.

## Çağrı akışı (özet)
```
UE-A → INVITE → P/S-CSCF → S-CSCF(UE-B) → P-CSCF → UE-B
        180 Ringing ← ... ←                         (çalıyor)
        200 OK ← ... ←                              (cevaplandı)
        RTP medya: UE-A ↔ rtpengine ↔ UE-B          (ses)
```
