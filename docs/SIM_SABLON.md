# SIM Kart Bilgileri — ŞABLON

> ⚠️ Bu bir ŞABLONDUR. Gerçek Ki/OPc/ADM değerlerini buraya yazıp repoya
> **COMMIT ETME**. Gerçek değerleri ayrı, versiyon kontrolü dışında bir dosyada
> tut (`.gitignore` `*.csv` ve `sim_kartlar*.md`'yi zaten hariç tutar).

Her kart için doldurulacak alanlar:

| Alan | Kart 1 | Kart 2 |
|------|--------|--------|
| IMSI | 0010100000000XX | 0010100000000YY |
| ICCID | ... | ... |
| Ki (16B hex) | `<GİZLİ>` | `<GİZLİ>` |
| OPc (16B hex) | `<GİZLİ>` | `<GİZLİ>` |
| ADM | `<GİZLİ>` | `<GİZLİ>` |
| MSISDN | 00100000000XX | 0010000000YY |
| AMF | 8000 | 8000 |

Sabitler (tüm kartlar):
- PLMN: MCC=001, MNC=01
- USIM auth: Milenage, Type=OPc
- SQN check: KAPALI (USIM + ISIM)

Abone eklemek için (değerleri komut satırında ver, dosyadan değil):
```bash
./volte add subscriber --imsi <IMSI> --ki <KI> --opc <OPC> --msisdn <MSISDN>
```
