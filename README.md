# Programlanabilir IO ve CAN Modülü (PDM)

Araçlar için kullanıcının blok programlama aracıyla yapılandırabildiği, 2 CAN hatlı giriş/çıkış yönetim modülü.

> Durum: **Tasarım taslağı.** Bu belge şimdiye kadar alınan kararları ve açık konuları toplar.

---

## 1. Temel fikir

- **Çekirdek firmware** (bizim) flash'ın sabit, yazma korumalı bölgesinde durur. Donanım yönetimi, koruma ve CAN hep çekirdekte kalır.
- **Kullanıcı bölgesi** flash'ta ayrı sektörlerdedir. Burada kod değil, **veri** bulunur: IO yönlendirme tabloları, CAN eşlemeleri ve blok programı.
- Kullanıcı, **blok programlama aracımızla** mantık kurar. Araç bunu binary bir imaja derler ve imaj CAN üzerinden karta yüklenir.
- Çekirdek imajı doğrular ve blokları sabit bir döngüde çalıştırır. Kullanıcı yeni blok yazamaz, sadece çekirdekteki blokları birbirine bağlar.

## 2. Kapsam

| Konu | Karar |
|---|---|
| Hedef pazar | Sivil araç: ticari araç, iş makinesi, özel araç |
| Besleme | 12 V ve 24 V sistemler (çalışma aralığı ~9–36 V) |
| Standartlar | ISO 16750-2, ISO 7637-2 |
| Kanal sayısı | 22 evrensel giriş, 23 çıkış (bkz. §4.3, §4.4) |
| CAN | 2 hat, CAN FD destekli, **SAE J1939** (3. hat opsiyon olarak değerlendirilecek) |
| Çalışma sıcaklığı | −40…+85 °C |
| Zorunlu özellikler | 5 V ve 12 V sensör beslemesi, CAN ve dijital girişle uyandırma |
| Askeri versiyon | **Şimdilik kapsam dışı.** Güç girişi ayrı blok olarak çizilecek; ileride MIL-STD-1275E surge modülü eklenebilir |

## 3. Sistem mimarisi

```
 Akü (12/24 V) ──► Koruma ──► Güç kaynağı (5 V / 3,3 V)
                    │                  │
                    ▼                  ▼
 Girişler ──► Giriş koşullama ──► STM32 ──► Çıkış sürücüleri ──► Yükler
                                   │  ▲          │
                                   │  └─ akım ölçümü ┘
                                   ▼
                          2× CAN transceiver ──► CAN1 / CAN2
```

Tüm veri akışı **merkezi bir sinyal tablosu** üzerinden geçer:

```
Fiziksel girişler ─┐                         ┌─► Fiziksel çıkışlar
CAN RX ────────────┼─►  SİNYAL TABLOSU  ◄─┐  ├─► CAN TX
                   │                      │  │
                   └──► Blok programı ────┘──┘
```

## 4. Donanım

### 4.1 MCU

- **Prototip:** STM32G474QE (LQFP128). FDCAN, 5 ADC ve bol timer'ı var. LQFP100 pin sayısı açısından yetmiyor.
- İlk geliştirme Nucleo-G474RE ve değerlendirme kartlarıyla yapılabilir.
- **MCU ileride değişebilir.** Yazılım ve donanım buna göre modüler tasarlanacak (bkz. §6.5).
- Not: Standart STM32'ler AEC-Q100 sertifikalı değil. OEM/Tier-1 hedeflenirse otomotiv sınıfı bir MCU'ya geçilecek.

### 4.2 Güç ve koruma

- Ters kutup koruması: ideal diyot kontrolcüsü + MOSFET (ör. LM74700-Q1)
- Load dump / transient: TVS (24 V sisteme göre boyutlandırılacak)
- Mantık beslemesi çıkış güç yolundan ayrı ve filtreli
- Ana besleme girişi: cıvatalı terminal (M6/M8)
- Güç girişi ayrı bir blok olarak çizilecek (askeri modül için yer)

### 4.3 Çıkışlar

Toplam **23 çıkış**:

| Tip | Adet | Kullanım | Not |
|---|---|---|---|
| High-side, yüksek akım | 4 | Fan, pompa, kompresör | ~20 A |
| High-side, standart | 14 | Lamba, valf, röle, motor | ~10 A, PWM destekli |
| Half-bridge | 3 | DC motor (cam, kapak, aktüatör), çift yönlü yükler | High tarafı ~10 A. İkisi birlikte H-köprü kurabilir. Alt taraf low-side çıkış ihtiyacını karşılar |
| Akım kontrollü PWM | 2 | Oransal hidrolik valf | Kapalı çevrim akım kontrolü |

Tüm çıkışlarda:
- Akım ölçümü ADC'ye bağlanacak, yazılımda **elektronik sigorta**, açık devre ve kısa devre tespiti yapılacak
- PWM ve soft-start desteklenecek
- CAN timeout veya arıza durumunda konfig'den gelen **failsafe** duruma geçilecek

High-side için aday aile: Infineon PROFET+ 24V (BTT6xxx). 12 V ailesiyle pin uyumlu olduğu için aynı footprint'e farklı akım sınıfları takılabilir. Half-bridge ve low-side parçaları 24 V uyumuna göre seçilecek.

**Standart yardımcı çıkışlar:**
- **Sensör beslemesi:** korumalı 5 V ve 12 V, akım sınırlı ve arıza tespitli

**Eklenebilecek diğer çıkış tipleri (değerlendirilecek):**
- **Analog çıkış:** 0–10 V veya 4–20 mA, harici kontrolörlere referans için

### 4.3.1 Uyku ve uyandırma

- Park halinde düşük tüketim modu (hedef: birkaç mA veya altı, netleşecek)
- **CAN** ve **dijital girişlerle** uyandırma (ignition dahil)
- Uyandırma kaynağı yapılandırılabilir; kullanıcı blok aracından seçer

### 4.4 Girişler

Toplam **22 giriş**. Tüm girişler mümkün olduğunca **evrensel** olacak: tipi kullanıcı blok aracından seçer, donanım buna hazır olur. (Rakipte giriş tipleri sabit dağılımlı; bu bizim farkımız.)

| Giriş modu | Açıklama |
|---|---|
| Dijital | Aktif yüksek/alçak, yazılımdan seçilebilir pull-up/pull-down, debounce |
| Analog gerilim | 0–5 V ve 0–36 V aralığı |
| PWM / frekans | Duty ve frekans ölçümü (hız sensörü, debimetre). Timer'a bağlı pinlerde (en az 4 kanal) |
| NTC / dirençli | Sıcaklık sensörü, dirençli şamandıra ve basınç sensörü. **Ölçüm aralığı en az 0–500 Ω**, kart üzerinde referans dirençle |

**Eklenebilecek diğer giriş tipleri (değerlendirilecek):**
- 4–20 mA akım girişi (endüstriyel sensörler)
- Kablo arıza tespiti (açık devre / kısa devre) dirençli switch girişlerinde
- Ignition / wake-up girişi

Donanım notları:
- Her girişte bölücü, RC filtre ve clamp bulunacak
- Pull-up/pull-down seçimi sadece konfig değişince güncelleniyor. MCU pini harcamamak için SPI üzerinden shift register zinciri (74HC595 vb.) kullanılacak
- ADC yükü 5 ADC'ye eşit dağıtılacak; çıkış akım ölçümleri ayrı bir ADC'de toplanacak

### 4.5 CAN

- 2× CAN FD transceiver (ör. TCAN1042-Q1, TJA1462), uyandırma destekli
- Protokol: **SAE J1939** (PGN/SPN eşleme, adres talebi) ve serbest CAN ID'li özel mesajlar
- 3. CAN hattı opsiyon olarak değerlendirilecek (G474'te 3 FDCAN mevcut)
- 120 Ω sonlandırma yazılımdan açılıp kapanabilir
- Common-mode choke ve ESD koruması
- Bus-off kurtarma firmware'de

### 4.6 Mekanik

- Konnektör: Deutsch DT/DTM veya benzeri otomotiv konnektörü
- PCB: en az 4 katman, güç hatlarında 2 oz bakır ve termal via'lar
- Conformal coating, −40…+85 °C bileşen seçimi

## 5. Flash bölümleme (STM32G474QE, 512 KB, öneri)

| Bölge | Adres | Boyut | İçerik | Koruma |
|---|---|---|---|---|
| Bootloader | `0x0800_0000` | 32 KB | CAN üzerinden firmware güncelleme | Yazma korumalı |
| Çekirdek | `0x0800_8000` | 352 KB | Ana firmware | Yazma korumalı (WRP) |
| Kullanıcı A | `0x0806_0000` | 32 KB | Kullanıcı imajı (aktif / yedek) | Çekirdek yazar |
| Kullanıcı B | `0x0806_8000` | 32 KB | Kullanıcı imajı (aktif / yedek) | Çekirdek yazar |
| Ayrılmış | `0x0807_0000` | 64 KB | Fabrika kalibrasyonu, arıza kaydı | Çekirdek yazar |

**A/B mantığı:** Yeni imaj boştaki bölgeye yazılır ve doğrulanır, sonra aktif yapılır. Yazma yarıda kalırsa eski imaj çalışmaya devam eder, cihaz kilitlenmez.

## 6. Firmware mimarisi

### 6.1 Katmanlar

| Katman | Görev | MCU'ya bağımlı mı? |
|---|---|---|
| HAL | GPIO, ADC, PWM, timer, FDCAN, flash | **Evet** (MCU değişince sadece bu yeniden yazılır) |
| IO yöneticisi | Filtre, debounce, giriş modları, çıkış sürme, elektronik sigorta | Hayır |
| CAN katmanı | Frame ↔ sinyal dönüşümü, periyodik TX, RX timeout | Hayır |
| Sinyal tablosu | Tüm mantıksal sinyallerin merkezi deposu | Hayır |
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

Örnek kayıtlar:

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
- Format **little-endian**, byte-byte serileştirilmiş, pointer yok. Böylece MCU değişse de imaj aynen çalışır
- Sinyal ID'leri sabit ve versiyonlu
- Durum tutan bloklar (timer, sayaç) için RAM açılışta statik olarak ayrılır, dinamik bellek kullanılmaz

### 6.3 Blok kütüphanesi (çekirdekte)

İlk sürüm için önerilen bloklar:
- Mantık: AND, OR, NOT, XOR
- Karşılaştırma: >, <, =, histerezisli eşik
- Zaman: on-delay, off-delay, pulse, flaşör
- Hafıza: SR/RS latch, toggle, sayaç
- Matematik: toplama, çarpma, ölçekleme, sınırlama
- Çıkış: PWM duty, half-bridge yön ve hız

### 6.4 Çalışma döngüsü

1. Girişleri oku ve filtrele, sinyal tablosuna yaz
2. CAN RX sinyallerini güncelle, timeout'ları kontrol et
3. Blok programını sırayla çalıştır (5–10 ms periyot)
4. Güvenlik kontrolleri: sigorta, failsafe
5. Çıkışları sür, CAN TX mesajlarını gönder

### 6.5 Taşınabilirlik

- MCU'ya özel her şey HAL'de (`hal_gpio_set`, `hal_adc_read`, `hal_can_send`, ...)
- Flash adresleri ve sektör boyutları tek bir yapılandırma dosyasında
- Donanım modüler çizilecek: MCU değişince sadece MCU bölümü yeniden çizilir

## 7. Kullanıcı programlama akışı

1. Kullanıcı blok programlama aracında girişleri, blokları, çıkışları ve CAN sinyallerini bağlar
2. Araç diyagramı doğrular, topolojik sıraya dizer ve imaj üretir
3. Araç karta bağlanır ve **firmware'in desteklediği blok kütüphanesi versiyonunu sorgular**
4. İmaj CAN üzerinden boştaki kullanıcı bölgesine (A/B) yazılır
5. Çekirdek CRC ve versiyonu doğrular, imajı aktif eder
6. **Canlı izleme:** araç sinyal tablosunu CAN üzerinden okur ve değerleri blokların üzerinde gösterir

Ek öneriler:
- CAN eşlemesi için DBC dosyası içe aktarma
- Hazır **J1939 sinyal kütüphanesi**: kullanıcı PGN/SPN'i listeden seçer, bit konumu elle girilmez

## 7.1 Rakip referansı ve farklılaşma

Referans ürün: Ortem oMux 22×24. Sayfada görülen özellikler: 9–36 V, 22 giriş (8 analog: 0–500 Ω / 0–5 V / NTC, 12 aktif yüksek, 2 aktif düşük), 24 çıkış (21× 10 A high-side, 3 half-bridge), 5 V ve 12 V sensör beslemesi, 3× CAN (J1939), CAN ve dijital girişle uyandırma, −40…+85 °C.

Bizim farklılaşma noktalarımız:
- **Blok programlama:** kullanıcı veya araç üreticisi kendi mantığını kurar, fabrika müdahalesi gerekmez
- **Evrensel girişler:** her pinin modu yazılımdan seçilir
- **Karışık çıkış sınıfları:** 20 A kanallar ve akım kontrollü valf çıkışları
- **Canlı izleme** ve blok aracından ayarlanabilen elektronik sigorta

## 8. Güvenlik ilkeleri

- Kullanıcı programı **hiçbir koşulda** çekirdek korumalarını kapatamaz (sigorta, failsafe, watchdog)
- Geçersiz veya bozuk imaj varsa güvenli varsayılan konfigürasyonla açılır
- Blok sayısına üst sınır var. İmaj kabul edilirken en kötü durum döngü süresi kontrol edilir
- CAN timeout'ta her çıkış kendi failsafe durumuna geçer

## 9. Açık konular

- [ ] Çıkış akım sınıflarının kesinleşmesi (4× 20 A + 14× 10 A + 3 half-bridge + 2 akım kontrollü PWM taslak)
- [ ] Kaç girişin frekans ölçümüne bağlanacağı
- [ ] Half-bridge ve akım kontrollü PWM entegre seçimi (24 V uyumlu)
- [ ] Uyku modu tüketim hedefi
- [ ] 3. CAN hattı gerekli mi?
- [ ] 22 giriş + 23 çıkış ile MCU pin bütçesi (LQFP128 yetiyor mu, yoksa GPIO genişletici mi?)
- [ ] Toplam sürekli akım ve konnektör seçimi
- [ ] Blok aracının mevcut çıktı formatı (JSON/XML/C) ve imaj dönüştürücüsü
- [ ] CAN yükleme protokolü: özel protokol mü, ISO-TP/UDS mi?
- [ ] Pin atama tablosu (çıkış listesi kesinleşince yeniden yapılacak)

## 10. Sonraki adımlar

1. Çıkış akım sınıflarını kesinleştir, pin bütçesini yeniden hesapla
2. Entegre seçimleri ve pin atama tablosu
3. Güç giriş katı şeması
4. KiCad şematik
5. Paralelde: Nucleo üzerinde çekirdek iskeleti (HAL, sinyal tablosu, blok çalıştırıcı)
