# sarjnerede — Native iOS App Spec

> Türkiye'deki tüm elektrikli araç şarj istasyonlarını harita üzerinde gösteren native iOS uygulaması.
> Referans: ChargeIQ, sarj.dev | Veri kaynağı: EPDK → sarj.dev API

---

## 1. Proje Özeti

**Ne yapar:** EPDK lisanslı tüm şarj istasyonlarını (8895+) haritada gösterir. Filtrele, yakınındakini bul, detay gör, navigasyona aç.

**Neden native iOS:** Performanslı harita rendering, MapKit/CoreLocation native entegrasyonu, background konum, widget desteği.

**Veri kaynağı:** `api.sarj.dev` (hazır backend, sarjdev/back-end reposu, Spring Boot + Elasticsearch)

---

## 2. Veri Modeli

### sarj.dev API

**Base URL:** `https://api.sarj.dev`

| Endpoint | Açıklama |
|---|---|
| `GET /v1/search` | Tüm istasyonlar (gzip, ~787KB) |
| `GET /v1/search/nearest?latitude=&longitude=&distance=10&size=10` | Yakın istasyonlar |
| `GET /v1/charging-station-providers` | Operatör listesi |
| `GET /v1/charging-station-detail/{provider}/{id}` | İstasyon detayı |

**Station modeli (ChargingStation):**
```json
{
  "id": 14585762,
  "title": "...",
  "geoLocation": { "lat": 41.01, "lon": 28.97 },
  "operator": { "id": 930848 },
  "address": "...",
  "phone": "...",
  "stationActive": true,
  "reservationUrl": "...",
  "plugs": [
    {
      "type": "CCS2",
      "subType": "DC",
      "power": 50.0,
      "price": 7.50,
      "availability": [...]
    }
  ],
  "paymentTypes": ["CREDIT_CARD", "APP"]
}
```

**Operatörler (ChargingProvider enum):**
- ESARJ, AKSAENERGY, ZES, EPDK, BEEFULL

### EPDK Ham Veri (kaynak)
- EPDK şarj istasyonu veritabanı, sarjdev tarafından `providers/epdk/data.json` olarak cache'leniyor
- Station alanları: id, title, address, lat, lng, phone, operatorId/Title, licenceActive, stationActive, cityId, districtId, sockets, paymentTypes
- Direkt EPDK API URL kesin bilinmiyor — sarj.dev API'si kullanılacak

---

## 3. Özellikler (MVP)

### 3.1 Harita Görünümü
- [ ] Türkiye merkezli MapKit haritası
- [ ] 8895 istasyon marker'ı (clustering zorunlu — performans)
- [ ] Marker rengi operatöre göre farklı (ESARJ: mor, ZES: mavi, AKSAENERGY: turuncu, vb.)
- [ ] Konum izni → "Yakınımdakiler" modu
- [ ] Zoom'a göre cluster → bireysel pin geçişi

### 3.2 İstasyon Detay (Bottom Sheet)
- [ ] İstasyon adı, adres, operatör logosu
- [ ] Soket listesi: tip (CCS2/CHAdeMO/Type2/AC), güç (kW), fiyat (₺/kWh)
- [ ] Müsaitlik durumu (Available / Occupied / Unknown)
- [ ] Telefon (tap to call)
- [ ] Navigasyona aç (Apple Maps / Google Maps)
- [ ] Rezervasyon URL varsa → Safari'de aç

### 3.3 Arama
- [ ] Şehir / ilçe / operatör adına göre
- [ ] `GET /v1/search/suggest` ile öneri
- [ ] Sonuçlara harita odaklanması

### 3.4 Filtreler
- [ ] Operatör (ESARJ, ZES, AKSAENERGY, EPDK, BEEFULL)
- [ ] Soket tipi (CCS2, CHAdeMO, Type 2 AC)
- [ ] Güç (hızlı şarj: 50kW+, ultra hızlı: 150kW+)
- [ ] Sadece aktif istasyonlar toggle

### 3.5 Yakınımdakiler
- [ ] Kullanıcı konumundan `distance` km içindekiler
- [ ] Liste görünümü (mesafe, operatör, müsaitlik)

---

## 4. Teknik Stack

### Zorunlu
| Katman | Seçim | Neden |
|---|---|---|
| Dil | Swift 5.9+ | Native iOS |
| UI | SwiftUI | Modern, hızlı geliştirme |
| Harita | MapKit + SwiftUI | Native, ücretsiz, offline cluster desteği |
| Networking | URLSession + async/await | Native, dependency yok |
| Veri Katmanı | sarj.dev REST API | Hazır, 8895 istasyon |

### Önerilen Ekstra
| Kütüphane | Neden | Alternatif |
|---|---|---|
| Cluster | `MapClusterAnnotation` (MapKit native) veya [MapLibre](https://github.com/maplibre/maplibre-gl-native) | performans şart |
| Caching | `NSCache` + `URLCache` | Offline/hız |
| State | `@Observable` (Swift 5.9) | Basit, modern |

---

## 5. Mimari

```
sarjnerede iOS App
├── App/
│   ├── sarjneredeApp.swift
│   └── ContentView.swift
├── Features/
│   ├── Map/
│   │   ├── MapView.swift          ← Ana harita
│   │   ├── MapViewModel.swift     ← State, filtering
│   │   └── StationAnnotation.swift
│   ├── StationDetail/
│   │   ├── StationDetailView.swift  ← Bottom sheet
│   │   └── StationDetailViewModel.swift
│   ├── Search/
│   │   └── SearchView.swift
│   └── Filters/
│       └── FilterView.swift
├── Core/
│   ├── Network/
│   │   ├── APIClient.swift         ← URLSession wrapper
│   │   └── SarjDevAPI.swift        ← Endpoint tanımları
│   ├── Models/
│   │   ├── ChargingStation.swift   ← Codable model
│   │   ├── Socket.swift
│   │   └── Operator.swift
│   └── Extensions/
│       └── CLLocation+Ext.swift
└── Resources/
    └── Assets.xcassets             ← Operatör logoları
```

---

## 6. Performans Kritik Noktaları

### 8895 Marker Sorunu
Tüm marker'ları aynı anda haritaya basmak app'i kilitler.

**Çözüm 1: MapKit Native Clustering**
```swift
// MKAnnotationView subclass + displayPriority
annotation.displayPriority = .defaultLow
mapView.register(ClusterAnnotationView.self, 
    forAnnotationViewWithReuseIdentifier: MKMapViewDefaultClusterAnnotationViewReuseIdentifier)
```

**Çözüm 2: Viewport-based loading**
- Sadece görünen bölgedeki istasyonları render et
- `mapView.visibleMapRect` değişince filtrele

### API Response
- `/v1/search` 787KB gzip → açılınca ~3MB JSON
- App başlangıçta bir kere çek, `NSCache`'e al
- Background `Task` ile fetch, UI bloklanmaz

---

## 7. UI/UX Referansları

### ChargeIQ'dan Alınacaklar
- Bottom sheet ile istasyon detayı (swipe up/down)
- Operatöre göre renkli marker'lar
- Filtre sheet (slide up)
- "Yakınımdakiler" butonu — haritanın üstünde floating

### sarj.dev Web'den Alınacaklar
- Soket tipi ikonları (CCS, CHAdeMO, Type 2)
- Operatör logoları (sarjdev/front-end repo'sunda mevcut: mstor.svg, mz.svg, vb.)

---

## 8. Geliştirme Aşamaları

### Phase 1 — Çalışan Harita (3-5 gün)
1. Xcode projesi oluştur, SwiftUI + MapKit setup
2. `api.sarj.dev/v1/search` çek, JSON parse et
3. Tüm istasyonları haritada göster (clustering ile)
4. Marker'a tıklayınca basit bilgi popup

### Phase 2 — İstasyon Detay (2-3 gün)
5. Bottom sheet component (SwiftUI `.sheet` veya custom)
6. Soket listesi, fiyat, operatör
7. Navigasyona aç (Apple Maps / Google Maps deeplink)

### Phase 3 — Filtreler & Arama (2-3 gün)
8. Operatör, soket tipi, güç filtreleri
9. Arama + suggest API entegrasyonu

### Phase 4 — Polish (1-2 gün)
10. Operatör logoları
11. Dark mode
12. App icon, launch screen

---

## 9. iOS Minimum Requirements

- **iOS:** 17.0+ (SwiftUI `.observable`, `MapKit` yeni API'ler)
- **Xcode:** 15+
- **Swift:** 5.9+
- **Cihaz:** iPhone (iPad opsiyonel)

---

## 10. Sonraki Versiyonlar (V2+)

- [ ] Favoriler (SwiftData)
- [ ] Rota planlama (şarj noktaları dahil)
- [ ] Gerçek zamanlı müsaitlik (operatör API'leri gerekir — ZES, ESARJ public API yok)
- [ ] Widget (en yakın istasyon)
- [ ] CarPlay desteği
- [ ] Bildirim: "Şarj noktası X km'de, batarya düşük"

---

## 11. Notlar

- **Lisans:** sarjdev/back-end ve sarjdev/front-end GPL-3.0 lisanslı. API kullanımı free, kod fork'lanamaz kapalı source için.
- **EPDK verisi** resmi kaynak: Türkiye'deki tüm lisanslı şarj noktaları kapsanıyor.
- **Gerçek zamanlı müsaitlik yok:** EPDK/sarj.dev API'si anlık soket durumunu vermiyor. Operatör bazlı API entegrasyonu gerekir (V2).
- **Oncharge entegrasyonu:** Oncharge cihazları EPDK'ya kayıtlıysa zaten bu veri içinde. V2'de Oncharge API'si ile gerçek zamanlı durum eklenebilir.

---

*Son güncelleme: 2026-02-27*
*Referans repolar: sarjdev/back-end, sarjdev/front-end*
*API: https://api.sarj.dev*
