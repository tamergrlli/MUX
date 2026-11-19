# Programlanabilir IO ve CAN Modülü

Araçlar için blok programlama ile yapılandırılabilen, 2 CAN hatlı giriş/çıkış yönetim modülü (PDM).

Proje tasarım aşamasındadır.

## İçindekiler

1. [Genel bakış](#1-genel-bakış)
2. [Kapsam](#2-kapsam)
3. [Sistem mimarisi](#3-sistem-mimarisi)
4. [Donanım](#4-donanım)
5. [Flash bölümleme](#5-flash-bölümleme)
6. [Firmware mimarisi](#6-firmware-mimarisi)
7. [Kullanıcı programlama akışı](#7-kullanıcı-programlama-akışı)
8. [Güvenlik ilkeleri](#8-güvenlik-ilkeleri)
9. [Açık konular](#9-açık-konular)
10. [Yol haritası](#10-yol-haritası)

## 1. Genel bakış

- Çekirdek firmware flash'ın sabit ve yazma korumalı bölgesinde çalışır. Donanım yönetimi, koruma fonksiyonları ve CAN haberleşmesi çekirdekte kalır.
- Kullanıcı bölgesi flash'ta ayrı sektörlerde bulunur ve yalnızca veri içerir: IO yönlendirme tabloları, CAN eşlemeleri ve blok programı.
- Kullanıcı mantığı blok programlama aracında oluşturulur, binary imaja derlenir ve CAN üzerinden karta yüklenir.
- Çekirdek imajı doğrular ve blokları sabit periyotlu bir döngüde çalıştırır. Kullanıcı yeni blok tipi tanımlayamaz; yalnızca çekirdekteki blokları birbirine bağlar.

## 2. Kapsam

| Konu | Değer |
|---|---|
| Uygulama | Ticari araç, iş makinesi, özel araç |
| Besleme | 12 V ve 24 V sistemler (çalışma aralığı 9–36 V) |
| Standartlar | ISO 16750-2, ISO 7637-2 |
| Kanal sayısı | 22 evrensel giriş, 23 çıkış |
| CAN | 2 hat, CAN FD, SAE J1939 |
| Çalışma sıcaklığı | −40…+85 °C |
| Yardımcı fonksiyonlar | 5 V ve 12 V sensör beslemesi, CAN ve dijital girişle uyandırma |

Askeri araç versiyonu (MIL-STD-1275E) şimdilik kapsam dışıdır. Güç girişi ayrı bir blok olarak tasarlanır; ileride surge koruma modülü eklenebilir.

## 3. Sistem mimarisi

```
 Akü (12/24 V) ──► Koruma ──► Güç kaynağı (5 V / 3,3 V)
                    │                  │
                    ▼                  ▼
 Girişler ──► Giriş koşullama ──► MCU ──► Çıkış sürücüleri ──► Yükler
                                   │  ▲          │
                                   │  └─ akım ölçümü ┘
                                   ▼
                          2× CAN transceiver ──► CAN1 / CAN2
```

Tüm veri akışı merkezi bir sinyal tablosu üzerinden geçer:

```
Fiziksel girişler ─┐                         ┌─► Fiziksel çıkışlar
CAN RX ────────────┼─►  SİNYAL TABLOSU  ◄─┐  ├─► CAN TX
                   │                      │  │
                   └──► Blok programı ────┘──┘
```

## 4. Donanım

### 4.1 MCU

- Mevcut seçim: STM32G474QE (LQFP128). 3× FDCAN, 5 ADC.
- Değerlendirilen alternatif: STM32H533 / STM32H523 (Cortex-M33, TrustZone, 2× FDCAN, 512 KB çift bank flash, 2 ADC). Bu seçenekte girişler analog mux üzerinden okunur.
- İlk geliştirme Nucleo kartlarıyla yapılabilir.
- Firmware MCU değişikliğine uygun şekilde katmanlı tasarlanır (bkz. [6.5](#65-taşınabilirlik)).

### 4.2 Güç ve koruma

- Ters kutup koruması: ideal diyot kontrolcüsü + MOSFET
- Load dump ve transient koruması: 24 V sisteme göre boyutlandırılmış TVS
- Mantık beslemesi çıkış güç yolundan ayrı ve filtreli
- Ana besleme girişi: cıvatalı terminal (M6/M8)

### 4.3 Çıkışlar

| Tip | Adet | Tipik yük | Akım |
|---|---|---|---|
| High-side, yüksek akım | 4 | Fan, pompa, kompresör | ~20 A |
| High-side, standart | 14 | Lamba, valf, röle, motor | ~10 A |
| Half-bridge | 3 | DC motor, çift yönlü aktüatör | ~10 A, iki yönde |
| Akım kontrollü PWM | 2 | Oransal hidrolik valf | Kapalı çevrim |

Tüm çıkışlarda:

- Akım ölçümü ile elektronik sigorta, açık devre ve kısa devre tespiti
- PWM ve soft-start
- CAN timeout veya arıza durumunda yapılandırılmış güvenli duruma geçiş

İki half-bridge birlikte H-köprü olarak kullanılabilir.

Yardımcı çıkışlar:

- Korumalı 5 V ve 12 V sensör beslemesi (akım sınırlı, arıza tespitli)

Değerlendirilen ek çıkışlar:

- 0–10 V veya 4–20 mA analog çıkış

### 4.4 Uyku ve uyandırma

- Park halinde düşük tüketim modu (hedef değer belirlenecek)
- CAN ve dijital girişlerle uyandırma (ignition dahil)
- Uyandırma kaynakları yapılandırılabilir

### 4.5 Girişler

22 giriş. Girişler evrenseldir; her pinin modu yazılımdan seçilir.

| Mod | Açıklama |
|---|---|
| Dijital | Aktif yüksek/alçak, yazılımdan seçilebilir pull-up/pull-down, debounce |
| Analog gerilim | 0–5 V ve 0–36 V |
| PWM / frekans | Duty ve frekans ölçümü (hız sensörü, debimetre). Timer'a bağlı en az 4 kanal |
| NTC / dirençli | Sıcaklık, şamandıra ve basınç sensörleri. Ölçüm aralığı en az 0–500 Ω |

Değerlendirilen ek giriş tipleri:

- 4–20 mA akım girişi
- Dirençli switch girişlerinde kablo arıza tespiti

Donanım notları:

- Her girişte bölücü, RC filtre ve clamp
- Pull-up/pull-down seçimi SPI üzerinden shift register zinciri ile yapılır (konfigürasyon değiştiğinde güncellenir)
- Çıkış akım ölçümleri ayrı bir ADC'de toplanır

### 4.6 CAN

- 2× CAN FD transceiver, uyandırma destekli
- SAE J1939 (PGN/SPN eşleme, adres talebi) ve serbest CAN ID'li mesajlar
- 120 Ω sonlandırma yazılımdan açılıp kapatılabilir
- Common-mode choke ve ESD koruması
- Bus-off kurtarma

### 4.7 Malzeme listesi (taslak)

Seçim ölçütleri:

- Tüm parçalar 24 V araç sistemine uygun olmalı (en az 36 V çalışma, 24 V load dump dayanımı). Bu nedenle PROFET+2 12V (BTS700x) ailesi kullanılmaz (en fazla 28 V çalışma).
- Prototip aşamasında AEC-Q100 sertifikası zorunlu değildir.

| Blok | Parça | Durum | Not |
|---|---|---|---|
| MCU | STM32G474QE | Seçildi | Alternatif: STM32H533 (bkz. 4.1) |
| High-side 10 A | Infineon BTT6010-1ERB | Seçildi | PROFET+ 24V, 5–36 V, 9 A nominal |
| High-side düşük akım | Infineon BTT6100-2ERA | Seçildi | PROFET+ 24V, çift kanal, ~2,2 A/kanal |
| High-side orta akım | Infineon BTT6030-2ERA | Aday | Çift kanal, ~4 A/kanal |
| High-side 20 A | TI TPS12S24F-Q1 + harici MOSFET veya Classic PROFET 24V | Açık | |
| Half-bridge | TI DRV8323SRTAR + 6× 60 V N-MOSFET | Seçildi | 6–60 V, bağımsız PWM modu, 3 dahili akım yükselticisi. Yedek: DRV8323RS (pin uyumlu değil) |
| Akım kontrollü PWM | Infineon TLE7242-2G veya MOSFET + INA181 + MCU kontrol döngüsü | Açık | |
| 5 V sensör beslemesi | Infineon TLE4250-2G | Aday | Kısa devre ve ters kutup korumalı tracker |
| 12 V sensör beslemesi | Akım sınırlı buck-boost | Açık | |
| CAN | TI TCAN1043A-Q1 veya NXP TJA1043 | Aday | INH/WAKE ile uyandırma |
| Pull-up/down seçimi | 74HC595 zinciri + sinyal MOSFET'leri | Aday | |

### 4.8 Mekanik

- Konnektör: Deutsch DT/DTM veya benzeri
- PCB: en az 4 katman, güç hatlarında 2 oz bakır ve termal via
- Conformal coating

## 5. Flash bölümleme

Örnek: 512 KB flash.

| Bölge | Adres | Boyut | İçerik | Koruma |
|---|---|---|---|---|
| Bootloader | `0x0800_0000` | 32 KB | CAN üzerinden firmware güncelleme | Yazma korumalı |
| Çekirdek | `0x0800_8000` | 352 KB | Ana firmware | Yazma korumalı |
| Kullanıcı A | `0x0806_0000` | 32 KB | Kullanıcı imajı | Çekirdek yazar |
| Kullanıcı B | `0x0806_8000` | 32 KB | Kullanıcı imajı | Çekirdek yazar |
| Ayrılmış | `0x0807_0000` | 64 KB | Kalibrasyon, arıza kaydı | Çekirdek yazar |

Yeni imaj boştaki bölgeye yazılır, doğrulanır ve ardından aktif yapılır. Yazma yarıda kalırsa önceki imaj çalışmaya devam eder.

## 6. Firmware mimarisi

### 6.1 Katmanlar

| Katman | Görev | MCU'ya bağımlı |
|---|---|---|
| HAL | GPIO, ADC, PWM, timer, FDCAN, flash | Evet |
| IO yöneticisi | Filtre, debounce, giriş modları, çıkış sürme, elektronik sigorta | Hayır |
| CAN katmanı | Frame ↔ sinyal dönüşümü, periyodik TX, RX timeout | Hayır |
| Sinyal tablosu | Mantıksal sinyallerin merkezi deposu | Hayır |
| Blok çalıştırıcı | Kullanıcı blok programını sabit döngüde yürütür | Hayır |
| Konfig yükleyici | İmajı doğrular (magic, versiyon, CRC), tabloları kurar | Hayır |
| Güvenlik | Failsafe, watchdog, arıza yönetimi | Hayır |

### 6.2 Kullanıcı imajı yapısı

```
+----------------------+
| Başlık               |  magic, format versiyonu, blok kütüphanesi versiyonu,
|                      |  uzunluk, CRC32
+----------------------+
| Giriş eşleme tablosu |  fiziksel pin -> sinyal, mod, filtre
+----------------------+
| Çıkış eşleme tablosu |  sinyal -> fiziksel çıkış, tip, failsafe
+----------------------+
| CAN eşleme tablosu   |  ID, yön, bit konumu, uzunluk, periyot, timeout
+----------------------+
| Blok listesi         |  topolojik sırada bloklar
+----------------------+
```

Kayıt yapıları:

```c
typedef struct {            /* Giriş: fiziksel pin -> sinyal */
  uint8_t  pin;
  uint8_t  mode;            /* dijital / analog / pwm / ntc */
  uint8_t  flags;           /* ters, pull-up, pull-down */
  uint16_t filter_ms;
  uint16_t signal_id;
} in_map_t;

typedef struct {            /* Çıkış: sinyal -> fiziksel çıkış */
  uint16_t signal_id;
  uint8_t  channel;
  uint8_t  type;            /* on/off, pwm, half-bridge yön */
  uint16_t fuse_ma;         /* elektronik sigorta eşiği */
  uint8_t  failsafe;
} out_map_t;

typedef struct {            /* CAN sinyal eşlemesi */
  uint32_t can_id;
  uint8_t  bus;             /* CAN1 / CAN2 */
  uint8_t  dir;             /* RX / TX */
  uint8_t  start_bit, length;
  uint16_t period_ms, timeout_ms;
  uint16_t signal_id;
} can_map_t;

typedef struct {            /* Blok programı kaydı */
  uint8_t  opcode;          /* BLK_AND, BLK_TON, ... */
  uint8_t  n_in;
  uint16_t in[4];           /* giriş sinyal ID'leri */
  uint16_t out;             /* çıkış sinyal ID'si */
  int32_t  param[2];        /* süre, eşik vb. */
} block_t;
```

Kurallar:

- Format little-endian ve byte bazında serileştirilmiştir; pointer içermez
- Sinyal ID'leri sabit ve versiyonludur
- Durum tutan bloklar (timer, sayaç) için RAM açılışta statik olarak ayrılır; dinamik bellek kullanılmaz

### 6.3 Blok kütüphanesi

İlk sürüm:

- Mantık: AND, OR, NOT, XOR
- Karşılaştırma: >, <, =, histerezisli eşik
- Zaman: on-delay, off-delay, pulse, flaşör
- Hafıza: SR/RS latch, toggle, sayaç
- Matematik: toplama, çarpma, ölçekleme, sınırlama
- Çıkış: PWM duty, half-bridge yön ve hız

### 6.4 Çalışma döngüsü

1. Girişleri oku, filtrele, sinyal tablosuna yaz
2. CAN RX sinyallerini güncelle, timeout kontrolü yap
3. Blok programını çalıştır (5–10 ms periyot)
4. Güvenlik kontrolleri: sigorta, failsafe
5. Çıkışları sür, CAN TX mesajlarını gönder

### 6.5 Taşınabilirlik

- MCU'ya özel kod yalnızca HAL katmanında bulunur (`hal_gpio_set`, `hal_adc_read`, `hal_can_send`, ...)
- Flash adresleri ve sektör boyutları tek bir yapılandırma dosyasında tanımlanır
- Donanım modüler çizilir; MCU değişikliği yalnızca MCU sayfasını etkiler

## 7. Kullanıcı programlama akışı

1. Blok programlama aracında girişler, bloklar, çıkışlar ve CAN sinyalleri bağlanır
2. Araç diyagramı doğrular, topolojik sıraya dizer ve imaj üretir
3. Araç karta bağlanır ve firmware'in desteklediği blok kütüphanesi versiyonunu sorgular
4. İmaj CAN üzerinden boştaki kullanıcı bölgesine yazılır
5. Çekirdek CRC ve versiyonu doğrular, imajı aktif eder
6. Canlı izleme: araç sinyal tablosunu CAN üzerinden okuyup blok diyagramı üzerinde gösterir

Planlanan araç özellikleri:

- DBC dosyası içe aktarma
- J1939 sinyal kütüphanesi (PGN/SPN listeden seçilir)

## 8. Güvenlik ilkeleri

- Kullanıcı programı çekirdek korumalarını (sigorta, failsafe, watchdog) devre dışı bırakamaz
- Geçersiz veya bozuk imaj durumunda varsayılan güvenli konfigürasyonla açılır
- Blok sayısı sınırlıdır; imaj kabul edilirken en kötü durum döngü süresi kontrol edilir
- CAN timeout durumunda her çıkış tanımlı failsafe durumuna geçer

## 9. Açık konular

- [ ] Çıkış akım sınıflarının kesinleşmesi
- [ ] Frekans ölçümüne bağlanacak giriş sayısı
- [ ] Half-bridge MOSFET seçimi (60 V)
- [ ] 20 A kanal çözümü
- [ ] Akım kontrollü PWM çözümü
- [ ] Uyku modu tüketim hedefi
- [ ] MCU kesinleşmesi ve pin bütçesi
- [ ] Toplam sürekli akım ve konnektör seçimi
- [ ] Blok aracının çıktı formatı ve imaj dönüştürücüsü
- [ ] CAN yükleme protokolü (özel protokol veya ISO-TP/UDS)
- [ ] Pin atama tablosu

## 10. Yol haritası

1. Çıkış akım sınıflarını kesinleştir, pin bütçesini hesapla
2. Entegre seçimleri ve pin atama tablosu
3. Güç giriş katı şeması
4. KiCad şematik ve PCB
5. Nucleo üzerinde çekirdek iskeleti (HAL, sinyal tablosu, blok çalıştırıcı)

## Lisans

Belirlenecek.
