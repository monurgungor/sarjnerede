# sarjnerede — Native iOS App Spec
> Son güncelleme: 2026-02-27
> Veri: EPDK → sarj.dev API | Referans: ChargeIQ, Voltla, Lixhium

---

## 1. Proje Tanımı

**Ne:** Türkiye'deki tüm EPDK lisanslı elektrikli araç şarj istasyonlarını native iOS haritasında gösteren uygulama.

**Neden native iOS (web değil):**
- MapKit & CoreLocation native performansı
- Background konum desteği (navigasyon entegrasyonu)
- Offline capability, NSCache ile anlık response
- Widget & CarPlay potansiyeli (V2)
- App Store dağıtım

---

## 2. Pazar Analizi

### Türkiye'deki Rakipler (App Store, Mart 2026)

| Uygulama | Geliştirici | Rating | Yorum | Min iOS | Son Güncelleme |
|---|---|---|---|---|---|
| **Voltla** | Voltla A.Ş. | 4.78 | 9.563 | 15.6 | Aktif |
| **Lixhium** | Lixhium A.Ş. | 4.66 | 5.646 | ? | Aktif |
| **ChargeIQ** | Borusan Otomotiv | 4.60 | 1.245 | 15.1 | Feb 2026 |

### ChargeIQ Detaylı İnceleme
- **Geliştirici:** Borusan Otomotiv (kurumsal, büyük bütçe)
- **Sürüm:** 2.2.9 (aktif geliştirme, Oct 2022'den beri)
- **Boyut:** ~61MB
- **Diller:** TR + EN
- **Desteklenen operatörler:** ZES, eŞarj (ESARJ), Trugo, SharzNet, Voltrun, Borenco, Tora, WAT, Astor vd.

**ChargeIQ Özellikleri (kaynak: App Store açıklaması):**
- Türkiye geneli AC/DC istasyon haritası
- Gerçek zamanlı doluluk/uygunluk durumu
- Rota planlama (başlangıç–varış + ara şarj noktaları)
- Gelişmiş filtreler: operatör, soket tipi, kW, şarj ağı, puan, tarife, yeşil enerji, konum
- Tarife karşılaştırma + maliyet hesaplama (aracına özel şarj süresi/maliyet)
- Favori istasyonlar
- Kullanıcı yorumları, puanlama, fotoğraf, check-in
- Araç profili (model, plaka, menzil, soket tipi)
- Şarj geçmişi + tüketim analizi
- "Keşfet" sekmesi (EV içerikleri)

**ChargeIQ'nun Zayıflıkları:**
- Kurumsal (Borusan) — yavaş iterasyon
- UX feedback'leri: kalabalık UI, çok fazla özellik
- 1245 yorum (Voltla'nın 1/8'i) — pazar payı düşük

### Fark Yaratma Fırsatları
- Sade, hızlı, odaklı UX (ChargeIQ çok şişirilmiş)
- EPDK verisi zaten tam — ek operatör entegrasyonu gereksiz MVP için
- **Oncharge entegrasyonu** (Onur'un şirketi — operatör ID: 918339 "KALYON ELECTRICAL VEHICLE ENERJİ YATIRIM ANONİM ŞİRKETİ") ile gerçek zamanlı veri V2'de rakiplere karşı avantaj

---

## 3. Veri Kaynağı: sarj.dev

### Proje Özeti
- **Repo:** github.com/sarjdev/back-end, github.com/sarjdev/front-end
- **Lisans:** GPL-3.0 (API kullanımı serbest, kaynak kod fork'lanamaz kapalı source için)
- **Stack:** Java 17, Spring Boot, Elasticsearch 7.15, MySQL 8, Docker
- **Veri:** EPDK ham verisi statik JSON olarak backend'e gömülü (`providers/epdk/data.json`). Canlı EPDK API çekilmiyor — snapshot veri.

### API Endpoints

**Base URL:** `https://api.sarj.dev`

| Endpoint | Method | Açıklama | Response |
|---|---|---|---|
| `/v1/search` | GET | Tüm istasyonlar | GZIP JSON, ~787KB |
| `/v1/charging-stations/{id}/detail` | GET | İstasyon detayı | JSON |
| `/v1/charging-station-providers` | GET | Operatör listesi | JSON array |
| `/v1/search/nearest` | GET | Yakın istasyonlar | JSON (ES geo query) |
| `/v1/search/suggest` | GET | Arama önerisi | JSON (ES full-text) |
| `/v1/charging-map-indexer/index` | POST | ES reindex | Kapalı |

### /v1/search Response (Map data — minimal, 8895 istasyon)
```json
{
  "chargingStations": [
    {
      "id": 14585762,
      "geoLocation": { "lat": 38.469539, "lon": 35.159369 },
      "operator": { "id": 930848 }
    }
  ]
}
```
> **Önemli:** Map endpoint sadece id + konum + operatorId döndürür. Detay için ayrı API call gerekir.

### /v1/charging-stations/{id}/detail Response (Tam model)
```json
{
  "id": 14585762,
  "title": "İstasyon Adı",
  "location": {
    "cityId": 34,
    "cityName": "İstanbul",
    "districtId": 1234,
    "districtName": "Şişli",
    "address": "Tam adres metni",
    "lat": 41.012345,
    "lon": 28.978901
  },
  "geoLocation": { "lat": 41.012345, "lon": 28.978901 },
  "operator": {
    "id": 930848,
    "title": "EŞARJ ELEKTRİKLİ ARAÇLAR ŞARJ SİSTEMLERİ ANONİM ŞİRKETİ.",
    "brand": "eŞarj"
  },
  "reservationUrl": "https://...",
  "phone": "+90...",
  "stationActive": true,
  "plugs": [
    {
      "id": 1,
      "type": "DC",
      "subType": "DC_CCS",
      "socketNumber": "1",
      "power": 50.0,
      "price": 7.50,
      "count": 2
    }
  ],
  "plugsTotal": 4,
  "provider": "EPDK",
  "paymentTypes": [{ "name": "MOBILODEME" }],
  "provideLiveStats": false,
  "searchText": "İstasyon Adı Şişli İstanbul"
}
```

### /v1/search/nearest Parametreler
```
latitude=41.015137&longitude=28.979530&distance=10&size=10
```
- `distance`: 1–20 km arası (validation bu range'de)
- `size`: 1–30 arası
- ES geo_distance query, ASC sort, mesafe hesaplama script'le

### Veri Modeli Detayları

**PlugType (enum):**
- `AC` — Alternatif akım
- `DC` — Doğru akım

**PlugSubType (enum) — sadece 3 tip:**
- `DC_CCS` — DC Combo (CCS2) — hızlı şarj
- `AC_TYPE2` — Type 2 AC — yavaş/orta şarj
- `DC_CHADEMO` — CHAdeMO — eski DC standard

**PaymentTypes (enum):**
- `MOBILODEME` — Mobil ödeme (tek bilinen değer)

**provideLiveStats:** Her zaman `false` — gerçek zamanlı soket durumu YOK.

### Elasticsearch Sorgu Yapısı
- **Index alias:** `charging-stations` (versioned alias pattern)
- **Search cache:** `@Cacheable("charging-stations-search-result")` — server-side cache
- **Suggest:** Full-text match on `searchText` field, `<b>` highlight
- **searchText:** `title + address + districtName + cityName` concat

### sarjdev Frontend'den Öğrenilen Tasarım Kararları

**Harita tile:** Google Maps (`mt0.google.com/vt/scale={dpr}&hl=en&x={x}&y={y}&z={z}`)
> iOS'ta MapKit kullanacağız, Google Maps tile gereksiz.

**Harita sınırları (Turkey bounds):**
- SW: `30.0°N, 25.0°E`
- NE: `44.0°N, 45.0°E`
- Merkez: `39.9255°N, 32.8663°E` (Türkiye coğrafi merkezi)
- Min zoom: 7

**Cluster yoğunluk seviyeleri (sarjdev'den aynen alınabilir):**
| Seviye | Sayı | Renk önerisi |
|---|---|---|
| safe | 0 | — |
| low | 1–15 | Açık mavi |
| mid-low | 16–35 | Mavi |
| mid | 36–65 | Turuncu |
| mid-high | 66–85 | Kırmızı-turuncu |
| high | 86+ | Kırmızı |

**Operatör İkon Mapping (sarjdev'den — 14 operatör özel ikon):**
| Operator ID | İsim | sarjdev ikon dosyası |
|---|---|---|
| 919287 | ZES | mz.svg |
| 992950 | BEEFULL | mb.svg |
| 952581 | AKSAENERGY | ma.svg |
| 930848 | eŞarj | me.svg |
| 929899 | EVS | ms.svg |
| 929964 | Trugo | mtrugo.svg |
| 993744 | Tesla | mtesla.svg |
| 929436 | WAT | mwatt.svg |
| 1093161 | Voltrun | mvoltrun.svg |
| 1006519 | Tunçmatik | mtunc.svg |
| 919121 | Astor | mastor.svg |
| 919069 | CW Enerji | mcw.svg |
| 1092971 | Doğuş Şarj | mdcharge.svg |
| 931220 | PowerŞarj | mpowersarj.svg |
| Diğerleri | — | marker.svg (default) |

**Önemli Not — Oncharge/Kalyon:**
- Operatör ID `918339`: "KALYON ELECTRICAL VEHICLE ENERJİ YATIRIM ANONİM ŞİRKETİ"
- Onur'un şirketi EPDK verisi içinde. V2'de bu operatöre özel ikonla ve Oncharge backend API'siyle gerçek zamanlı durum eklenebilir.

**Tüm operatör listesi:** 130+ EPDK lisanslı şarj operatörü (`app/data/operators.ts` içinde tam liste var):
ESARJ, ZES, AKSAENERGY, TRUGO, TESLA, BEEFULL, VOLTRUN, SHELL, PETROL OFİSİ, AYDEM PLUS, DOĞUŞ ŞARJ, WAT, ASTOR, CW ENERJİ, TUNÇMATIK, TORA, INCHARGE, GCOW, EGESARJ, OTOJET vd.

---

## 4. Özellik Listesi

### MVP (V1)

#### 4.1 Harita
- [ ] MapKit haritası, Türkiye sınırlı (SW: 30N,25E — NE: 44N,45E)
- [ ] Harita merkezi: 39.9255°N, 32.8663°E
- [ ] Min zoom 7, max zoom 18
- [ ] App açılışta `/v1/search` çek, parse et, NSCache'e al
- [ ] Tüm 8895 istasyon marker olarak render (clustering ile)
- [ ] Cluster seviyeleri: low(1-15), mid-low(16-35), mid(36-65), mid-high(66-85), high(86+)
- [ ] 14 operatör için özel ikon, geri kalanlar default marker
- [ ] Kullanıcı konumu (mavi nokta), zoom ≥ 14'te göster
- [ ] "Konumuma git" butonu (floating)

#### 4.2 İstasyon Detayı (Bottom Sheet)
- [ ] Marker tap → zoom 15'e flyTo → `/v1/charging-stations/{id}/detail` çek
- [ ] Bottom sheet: istasyon adı, operatör adı/brand, adres
- [ ] Soket listesi: her soket için tip (DC_CCS / AC_TYPE2 / DC_CHADEMO), kW, fiyat, adet
- [ ] `stationActive` → aktif/pasif badge
- [ ] Telefon numarası (tap to call — `tel:` URL scheme)
- [ ] Rezervasyon linki varsa "Rezervasyon" butonu (Safari)
- [ ] "Yol Tarifi Al" → Apple Maps deeplink (`maps://`)
- [ ] `provideLiveStats: false` → "Anlık durum bilgisi yok" uyarısı göster

#### 4.3 Yakınımdakiler Filtresi
- [ ] Konum izni isteme (graceful, açıklama metni ile)
- [ ] Mesafe slider: 1–20 km
- [ ] Boyut slider: 1–30 sonuç
- [ ] `/v1/search/nearest` çağır, liste görünümünde göster
- [ ] Listedeki istasyona tap → haritada odaklan

#### 4.4 Arama
- [ ] Arama çubuğu (floating, haritanın üstünde)
- [ ] `/v1/search/suggest?q={query}&size=10` ile öneri
- [ ] Öneri listesi, tap → haritada o istasyona flyTo + detay aç

### V2 (Sonraki Sürüm)
- [ ] Operatör filtresi
- [ ] Soket tipi filtresi (DC_CCS / AC_TYPE2 / DC_CHADEMO)
- [ ] kW gücü filtresi (hızlı ≥50kW, ultra hızlı ≥150kW)
- [ ] Favoriler (SwiftData)
- [ ] Rota planlama (şarj noktaları dahil)
- [ ] **Oncharge/Kalyon gerçek zamanlı soket durumu** (operator 918339)
- [ ] Kullanıcı yorumu / puanlama
- [ ] Widget (en yakın istasyon)
- [ ] CarPlay desteği
- [ ] Araç profili (model, soket tipi → filtre otomatik ayarlanır)

---

## 5. Teknik Mimari

### Stack
| Katman | Seçim | Gerekçe |
|---|---|---|
| Dil | Swift 5.9+ | Native iOS |
| UI | SwiftUI | Modern, hızlı |
| Harita | MapKit (SwiftUI) | Native, ücretsiz, clustering desteği |
| Networking | URLSession + async/await | Dependency yok |
| State | `@Observable` + `@StateObject` | Swift 5.9 modern pattern |
| Cache | `NSCache` + `URLCache` | Offline/hız |
| Persistence | UserDefaults (MVP), SwiftData (V2) | |

### Klasör Yapısı
```
sarjnerede/
├── sarjneredeApp.swift
├── ContentView.swift
├── Features/
│   ├── Map/
│   │   ├── MapView.swift               ← Ana SwiftUI MapKit görünümü
│   │   ├── MapViewModel.swift          ← State, API çağrı, filtering
│   │   ├── StationAnnotation.swift     ← MKAnnotation subclass
│   │   └── ClusterAnnotationView.swift ← Cluster görünümü (6 seviye)
│   ├── StationDetail/
│   │   ├── StationDetailView.swift     ← Bottom sheet (SwiftUI .sheet)
│   │   ├── StationDetailViewModel.swift
│   │   └── PlugRowView.swift           ← Soket satırı bileşeni
│   ├── Search/
│   │   ├── SearchView.swift
│   │   └── SearchViewModel.swift
│   └── NearbyFilter/
│       ├── NearbyFilterView.swift
│       └── NearbyFilterViewModel.swift
├── Core/
│   ├── Network/
│   │   ├── APIClient.swift             ← URLSession async/await wrapper
│   │   ├── SarjDevEndpoints.swift      ← Endpoint enum
│   │   └── NetworkError.swift
│   ├── Models/
│   │   ├── ChargingStation.swift       ← Codable, search response
│   │   ├── StationDetail.swift         ← Codable, detail response
│   │   ├── PlugModel.swift             ← Plug + PlugType + PlugSubType
│   │   ├── OperatorModel.swift
│   │   └── SearchModels.swift          ← Nearest + Suggest responses
│   ├── Cache/
│   │   └── StationCache.swift          ← NSCache wrapper
│   └── Extensions/
│       ├── CLLocation+Ext.swift
│       └── MKCoordinateRegion+Turkey.swift
└── Resources/
    ├── Assets.xcassets
    │   └── OperatorIcons/              ← 14 operatör + default marker
    └── Info.plist
```

### API Client Tasarımı
```swift
// Gzip decompression dahil
struct APIClient {
    static func fetchAllStations() async throws -> SearchResponse
    static func fetchDetail(id: Int) async throws -> StationDetail
    static func fetchNearest(lat: Double, lon: Double, distance: Int, size: Int) async throws -> NearestResponse
    static func fetchSuggestions(query: String) async throws -> SuggestResponse
}
```

### Clustering Stratejisi
**Seçenek A: MKAnnotationView Native Clustering (önerilen MVP için)**
```swift
mapView.register(ClusterAnnotationView.self,
    forAnnotationViewWithReuseIdentifier:
    MKMapViewDefaultClusterAnnotationViewReuseIdentifier)
// Her annotation: clusteringIdentifier = "station"
```

**Seçenek B: Viewport-based Loading (8895 marker için daha performanslı)**
- `MKMapViewDelegate.regionDidChangeAnimated` dinle
- `mapView.visibleMapRect` içindeki istasyonları filtrele
- Sadece visible annotations render et

→ **Önerilen:** Seçenek A ile başla, performans sorunu çıkarsa B'ye geç.

---

## 6. API Davranış Notları (Keşfedilen Edge Cases)

1. **`/v1/search` minimal veri:** Sadece `{id, geoLocation, operator.id}` döner. Başlık, adres, soket bilgisi YOKTUR. Detay için ayrı call şart.

2. **Detay endpoint URL formatı:** `GET /v1/charging-stations/{numericId}/detail`
   - ❌ `/v1/charging-station-detail/EPDK/14585762` (eski format — 404)
   - ✅ `/v1/charging-stations/14585762/detail`

3. **`/v1/search` response gzip:** Header `Content-Encoding: gzip`. URLSession otomatik decompress eder.

4. **`/v1/search/nearest` 500 hatası:** Üretim API'sinde aralıklı hata var. iOS'ta retry logic + graceful fallback gerekli.

5. **`provideLiveStats: false`:** Tüm istasyonlarda. Anlık müsaitlik bilgisi hiçbir zaman gelmez. UI'da "Durum bilgisi mevcut değil" göster.

6. **`searchText`:** Elasticsearch autocomplete analyzer ile index'li. Türkçe karakter desteği var (analyzer konfigürasyonu backend'de).

7. **Cache:** Backend `/v1/search` sonucunu `@Cacheable` ile cache'liyor. EPDK verisi statik — günlük veya haftalık güncelleniyor (tahminen). iOS'ta da local cache yapılmalı.

8. **EPDK veri tazeliği:** sarjdev EPDK'dan canlı veri çekmiyor, `data.json` snapshot. Ne zaman güncellendiği bilinmiyor. Bu tüm rakipler için de geçerli (EPDK public API gerçek zamanlı değil).

---

## 7. Performans Kritik Noktaları

### 8895 Marker Problemi
```
Sorun: 8895 MKAnnotation aynı anda eklenmesi → UI freeze
Çözüm: MKAnnotationView clustering + clusteringIdentifier
Test: 10K annotation ile Instruments profiling
```

### Ağ Optimizasyonu
```
/v1/search → gzip ~787KB → açılınca ~3MB
Strateji:
1. App launch'da background Task ile fetch
2. NSCache'e al (session boyunca tekrar çekme)
3. URLCache ile HTTP disk cache (304 Not Modified)
```

### Marker Tap → Detay Gecikmesi
```
Sorun: Marker tap sonrası API call gecikme (UX kötü)
Çözüm:
- Bottom sheet hemen aç (skeleton/loading state ile)
- API call async olarak devam eder
- Data gelince UI güncellenir
```

---

## 8. iOS Gereksinimleri

- **iOS minimum:** 16.0+
  - SwiftUI `.presentationDetents` (bottom sheet) → iOS 16
  - `Map` SwiftUI MapKit yeni API → iOS 16
  - `@Observable` macro → iOS 17 (isterseniz 17+ yapın)
- **Xcode:** 15+
- **Swift:** 5.9+
- **Gerekli izinler (Info.plist):**
  - `NSLocationWhenInUseUsageDescription`
  - `NSLocationAlwaysAndWhenInUseUsageDescription` (V2 navigasyon için)

---

## 9. Geliştirme Planı

### Phase 1 — Harita Çalışıyor (3–4 gün)
1. Xcode projesi: SwiftUI + MapKit setup
2. `APIClient.fetchAllStations()` + Codable model
3. 8895 annotation haritaya ekle (clustering ile)
4. 14 operatör için ikon asset'leri ekle (sarjdev SVG → PNG convert)
5. Temel map interaction (zoom, pan, Turkey bounds)

### Phase 2 — İstasyon Detayı (2–3 gün)
6. Marker tap → `APIClient.fetchDetail()` async
7. Bottom sheet component (`presentationDetents: [.medium, .large]`)
8. Soket listesi, operatör, adres, telefon
9. "Yol Tarifi Al" → Apple Maps deeplink
10. Loading / error state

### Phase 3 — Yakınımdakiler + Arama (2–3 gün)
11. Konum izni flow
12. Yakınımdakiler bottom sheet (mesafe + boyut slider)
13. `/v1/search/nearest` entegrasyonu
14. Arama çubuğu + `/v1/search/suggest`

### Phase 4 — Polish (1–2 gün)
15. App icon + launch screen
16. Dark mode
17. Error handling polish
18. TestFlight build

---

## 10. sarj.dev Altyapı (Kendi Backend Kurulumu İçin)

sarjdev açık kaynak, kendin de host edebilirsin:

```bash
git clone https://github.com/sarjdev/back-end.git
cd back-end
docker-compose up  # Elasticsearch + MySQL
mvn spring-boot:run
# POST http://localhost:8080/v1/charging-map-indexer/index  (data.json → ES)
```

**Bağımlılıklar:**
- Elasticsearch 7.15.0
- MySQL 8.0
- Java 17+
- Spring Boot (Maven)

**EPDK data.json yolu:** `src/main/resources/environments/development/providers/epdk/data.json`
(gitignore'da değil — commit'lenmemiş. Kendi instance için EPDK'dan temin et veya sarj.dev API kullan.)

---

## 11. Gelecek Vizyonu

### Oncharge Entegrasyonu (Stratejik Fırsat)
- Operatör ID 918339 = "KALYON ELECTRICAL VEHICLE ENERJİ YATIRIM ANONİM ŞİRKETİ"
- Oncharge backend'i ile gerçek zamanlı soket durumu çekilebilir
- Rakiplerden (ChargeIQ, Voltla) tek fark yaratacak özellik
- Sadece Oncharge cihazlarında anlık müsaitlik göster → kullanıcı güveni

### EPDK Canlı API
- EPDK'nın seffaflik.epdk.gov.tr API'si kapalı/erişilemez durumda
- sarjdev de canlı API çekmiyor, statik snapshot kullanıyor
- Eğer EPDK resmi API açılırsa tüm rakipler için oyun değiştirici

### Katkı Modeli
- sarjdev GPL-3.0: uygulamayı da open-source yaparsan sarjdev frontend/backend kodu kullanılabilir
- Kapalı source yaparsan sadece sarj.dev API'sini tüketebilirsin (UI/logic sıfırdan)

---

## 12. Referanslar

| Kaynak | URL |
|---|---|
| sarjdev Backend | https://github.com/sarjdev/back-end |
| sarjdev Frontend | https://github.com/sarjdev/front-end |
| sarjdev Canlı Site | https://sarj.dev |
| sarjdev API | https://api.sarj.dev |
| ChargeIQ App Store | https://apps.apple.com/tr/app/chargeiq/id1609390820 |
| ChargeIQ Web | https://www.chargeiq.com.tr |
| Voltla App Store | https://apps.apple.com/tr/app/voltla/id6444761202 |
| Lixhium App Store | https://apps.apple.com/tr/app/lixhium/id1645154277 |
| EPDK | https://epdk.gov.tr |
