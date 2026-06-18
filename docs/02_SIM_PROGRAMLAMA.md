# 02 — SIM Programlama (sysmoISIM-SJA5)

VoLTE'nin çalışması için SIM'in ağdaki abone kaydıyla **birebir** eşleşmesi gerekir.

## Temel parametreler

| Alan | Değer |
|------|-------|
| MCC | 001 |
| MNC | 01 |
| USIM auth | Milenage, **Type = OPc** (OP değil) |
| AMF | 8000 |
| Ki / OPc | karta özel (16 byte hex) |

## Kritik: SQN check'i KAPAT

sysmoISIM'de **sequence number (SQN) kontrolü hem USIM hem ISIM için kapatılmalı**.
Açık kalırsa kimlik doğrulama `MAC failure` veya `Synch failure` verir ve attach
başarısız olur. (Bu, test ağlarında en sık auth sorunudur.)

## Değerleri güvenilir okuma

> Hex değerleri (Ki/OPc) **gözle okuma** — kolayca yanlış okunur (0/O, B/8, vb.).
> Programlayıcının ürettiği CSV/çıktı dosyasından makine ile al.

Örnek (bir kart):
```
IMSI   : 001010000000001
Ki     : 0BDEB2CB463A5A5A29307A73F4FA0A86
OPc    : 19D7004F5D2C16EB68968F90A082556C
MSISDN : 0010000000001
AMF    : 8000
```

## Ağ tarafıyla tutarlılık

Aynı IMSI/Ki/OPc/AMF üç yerde de aynı olmalı:
1. **SIM** (programlama)
2. **open5gs HSS** (EPC — MongoDB/WebUI)
3. **pyHSS** (IMS — auc kaydı)

`./volte add subscriber --imsi ... --ki ... --opc ... --msisdn ...` üç yerin
ikisini (open5gs + pyHSS) tek seferde yazar; SIM'i ayrıca programlarsın.

## Programlama araçları

sysmocom'un `pySim` araçları (`pySim-prog.py` / `pySim-shell.py`) ile yazılır.
ADM anahtarı karta özeldir ve yazma için gereklidir. Ayrıntılı komutlar için
sysmocom SJA5 dökümanına bakın.
