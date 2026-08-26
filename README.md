# 🍗 GEDİK Ürün Tespit & Üretim Takip Sistemi

Üretim hattından gelen canlı kamera görüntülerini YOLO modeli ile analiz ederek hatta üretilen tavuk ürününü otomatik olarak tespit eden, ürün başlangıç/bitiş sürelerini takip eden ve vardiya bazlı üretim kayıtları oluşturan web tabanlı üretim takip sistemi.

Sistem; RTSP kamera görüntüsünü işler, YOLO tahminlerini Norfair takibi ve zaman tabanlı ürün kilitleme mantığıyla kararlı hale getirir. Tespit edilen ürünlerin üretim süreleri aylık JSON verisine kaydedilir, günlük/aylık dashboard üzerinden görüntülenir ve gerektiğinde Excel olarak dışa aktarılabilir.

---

## 🛠️ Kullanılan Teknolojiler

| Teknoloji | İşlev |
|-----------|-------|
| **Python Flask** | Web dashboard'u, API endpoint'leri ve kullanıcı oturum yönetimi |
| **Ultralytics YOLO** | Canlı görüntüde ürün sınıfı tespiti |
| **Norfair** | Tespitlerin takip edilmesi ve kısa süreli hatalı algıların azaltılması |
| **OpenCV** | Görüntü işleme, JPEG önizleme ve kamera desteği |
| **PyAV** | RTSP kamera görüntüsünü daha kararlı almak için ana video backend'i |
| **JavaScript / HTML / CSS** | Dashboard ve yönetim arayüzü |
| **JSON** | Üretim kayıtları, kullanıcılar, servis durumu ve mail geçmişi |
| **Excel (openpyxl)** | Dashboard raporlarını ve vardiya kayıtlarını `.xlsx` olarak dışa aktarma |
| **Pillow** | Excel içerisine grafik/görsel ekleme işlemleri |

---

## 🌐 Erişim Linki

```text
http://yapayzeka:8507
```

**Varsayılan Kullanıcılar:**
- `admin` / `admin123`
- `operator1` / `operator123`

> Kullanıcılar ve şifreler `data/users_db.json` üzerinden yönetilir. Admin kullanıcı arayüz üzerinden yeni kullanıcı ekleyebilir, silebilir, şifre değiştirebilir ve aktif oturumları sonlandırabilir.

---

## 🤖 Yapay Zeka Modeli

**Model Türü:** Ultralytics YOLO Object Detection  
**Aktif Model Adı:** `urun_tespit_24_06_best`  
**Model Yolu:** `model/urun_tespit_24_06_best.pt`  
**Confidence Threshold:** `0.70`  
**Model Görüntü Ölçeği:** Orijinal görüntünün `0.5` katı  
**Takip Sistemi:** Norfair  
**Ürün Karar Sistemi:** 60 saniyelik oy toplama / ürün kilitleme

Ürün sınıfları ayrı bir sabit listeden değil, doğrudan yüklenen YOLO modelinin `model.names` bilgisinden okunur. Böylece modeldeki sınıf isimleri dashboard ve ürün takip sistemine otomatik aktarılır.

---

## 📁 Klasör Yapısı

```text
proje-root/
├── data/
│   ├── engine_state.json       # Servis restart sonrası aktif ürün durumunu korur
│   ├── uretim_aylik.json       # Gün/vardiya bazlı üretim kayıtları
│   ├── users_db.json           # Kullanıcılar ve oturum bilgileri
│   ├── mail_settings.json      # Mail alıcı ayarları
│   ├── mail_log.json           # Gönderilen mail geçmişi
│   ├── log_product_shift.txt   # Uygulama / servis logları
│   ├── gedik-logo.png          # Mail ve raporlarda kullanılan logo
│   └── _mail_tmp/              # Geçici vardiya Excel dosyaları
├── model/                      # Aktif YOLO model dosyası (.pt)
├── static/
│   ├── dashboard.html          # ★ Ana web arayüzü
│   ├── dashboard.js            # Dashboard JavaScript işlemleri
│   ├── style.css               # Arayüz stilleri
│   └── gedik_logo.png          # Dashboard logosu
├── templates/                  # Yardımcı HTML şablonları
├── main.py                     # ★ Kamera + YOLO + Norfair ana çalışma döngüsü
├── dashboard_server.py         # ★ Flask dashboard ve API sunucusu
├── product_event_engine.py     # ★ Ürün başlangıç/bitiş/değişim motoru
├── product_lock.py             # Ürün oy toplama ve kararlı lock sistemi
├── video_backends.py           # PyAV / OpenCV RTSP kamera motorları
├── shift_report.py             # Vardiya ve aylık JSON kayıt sistemi
├── shift_utils.py              # Vardiya zaman hesaplamaları
├── auth.py                     # Kullanıcı giriş ve yetkilendirme sistemi
├── send_email.py               # SMTP mail gönderimi
├── product_event_mailer.py     # Ürün START / END / CHANGE bildirim mailleri
├── mail_settings.py            # Mail alıcı ayarları
├── mail_log.py                 # Mail geçmişi
├── log_utils.py                # Merkezi loglama
├── config.py                   # ★ Merkezi sistem ayarları
├── run_product_service.py      # ★ Windows/NSSM servis başlatıcısı
├── product_service.py          # Alternatif servis çalıştırıcısı
└── requirements.txt            # Python bağımlılıkları
```

---

## 📋 Python Dosyaları

### `main.py` (★ Ana Tespit Motoru)
**Görev:** Kameradan canlı görüntüyü alıp YOLO ile ürün tespiti yapmak ve sonuçları üretim takip motoruna iletmek.

**Başlıca İşlevler:**
- RTSP kameraya bağlanma
- YOLO modelini yükleme
- Görüntüyü model öncesinde yeniden boyutlandırma
- YOLO tespitlerini Norfair formatına dönüştürme
- Baskın ürün sınıfını belirleme
- 60 saniyelik ürün oy/lock sistemini çalıştırma
- Ürün START / END / CHANGE olaylarını üretme
- Dashboard için canlı JPEG görüntüsü yayınlama
- Ürün kilitliyken adaptif inference ile CPU/GPU yükünü azaltma
- Servis restart sonrasında aktif üründen devam etme

### `dashboard_server.py` (★ Web Dashboard - Flask)
**Görev:** Üretim takip web arayüzünü, canlı kamera görüntüsünü ve tüm API endpoint'lerini sunmak.

**Başlıca İşlevler:**
- Kullanıcı girişi ve oturum kontrolü
- Günlük ve aylık üretim kayıtlarını sunma
- Canlı kamera görüntüsü yayınlama
- Aktif ürün bilgisini gösterme
- Ürün sınıflarını modelden okuma
- Raporları Excel olarak dışa aktarma
- Kullanıcı yönetimi
- Mail geçmişi ve mail alıcı ayarları

**Temel Endpoint'ler:**
- `/` → Ana dashboard
- `/api/login` → Kullanıcı girişi
- `/api/logout` → Oturum kapatma
- `/api/session` → Aktif oturum bilgisi
- `/data` → Üretim, log ve aktif ürün verileri
- `/api/classes` → YOLO model sınıfları
- `/api/shift-config` → Vardiya ayarları
- `/video_feed` → Canlı kamera MJPEG yayını
- `/camera/status` → Kamera ve model durumu
- `/export_excel` → Dashboard raporunu Excel olarak dışa aktar
- `/api/admin/users` → Kullanıcı yönetimi
- `/api/mail/history` → Mail geçmişi
- `/api/mail/settings` → Mail alıcı ayarları

### `product_event_engine.py` (★ Ürün Olay Motoru)
**Görev:** Kameradan gelen kararlı ürün bilgisini gerçek üretim başlangıç/bitiş/değişim olaylarına dönüştürmek.

**Başlıca İşlevler:**
- Yeni ürün başlangıcını doğrulama
- Ürün kaybolduğunda üretim bitişini belirleme
- Ürün değişimini süre bazlı doğrulama
- Çok kısa üretimleri filtreleme
- Aynı vardiyada ürün değişimlerini yönetme
- Servis restart sonrası `engine_state.json` üzerinden kaldığı yerden devam etme
- START, END ve CHANGE olayları üretme

### `product_lock.py` (Ürün Kilitleme)
**Görev:** Tek karelik veya kısa süreli yanlış YOLO tahminlerinin gerçek ürün değişimi olarak algılanmasını engellemek.

**Başlıca İşlevler:**
- Belirli zaman penceresinde ürün tahminlerini toplar
- En baskın ürünü belirler
- Minimum gözlem sayısı kontrolü yapar
- Baskınlık oranına göre ürünü lock eder
- Ürün görünmediğinde absence oranına göre lock'u kaldırır

### `video_backends.py` (Kamera Motoru)
**Görev:** RTSP kameradan görüntüyü kararlı ve düşük gecikmeli şekilde almak.

**Desteklenen Backend'ler:**
- **PyAV** → Varsayılan ve önerilen yöntem
- **OpenCV** → PyAV kullanılamadığında fallback yöntem

PyAV tarafı bozuk/parazitli kareleri filtrelemek ve RTSP akışını daha stabil yönetmek için kullanılır.

### `shift_report.py` (Vardiya & Üretim Kaydı)
**Görev:** Tamamlanan ürün üretim aralıklarını aylık JSON dosyasına kaydetmek ve vardiya geçişlerini yönetmek.

**Başlıca İşlevler:**
- `data/uretim_aylik.json` dosyasına ürün kaydı
- GÜNDÜZ / GECE vardiya ayrımı
- Ürün başlangıç, bitiş ve toplam süre kaydı
- Minimum rapor süresinden kısa kayıtları filtreleme
- Vardiya sonunda aktif ürünü kapatma
- İstenirse vardiya Excel dosyası oluşturma
- Mail aktifse vardiya raporunu gönderme

> `save_count_to_xlsx()` fonksiyon adı eski uyumluluk nedeniyle korunmuştur; güncel sistem doğrudan Excel'e değil `uretim_aylik.json` dosyasına yazar.

### `shift_utils.py` (Vardiya Hesaplama)
**Görev:** `config.py` içerisindeki vardiya saatlerinden aktif vardiyayı hesaplamak.

Mevcut vardiyalar:
- **GÜNDÜZ:** 08:30
- **GECE:** 20:30

### `auth.py` (Kullanıcı ve Yetkilendirme)
**Görev:** Dashboard kullanıcı girişlerini ve admin yetkilerini yönetmek.

**Başlıca İşlevler:**
- Kullanıcı girişi / çıkışı
- Flask session yönetimi
- Admin / kullanıcı rol kontrolü
- Yeni kullanıcı oluşturma
- Kullanıcı silme
- Şifre değiştirme
- Aktif oturumları sonlandırma
- Oturum geçmişi tutma

### `send_email.py` + `product_event_mailer.py` (Mail Sistemi)
**Görev:** Vardiya raporlarını ve isteğe bağlı ürün başlangıç/bitiş bildirimlerini e-posta ile göndermek.

**Desteklenen Bildirimler:**
- Ürün başladı
- Ürün bitti
- Ürün değişti
- Vardiya raporu

Mail özellikleri `config.py` üzerinden ayrı ayrı açılıp kapatılabilir.

### `config.py` (★ Merkezi Ayarlar)
**Görev:** Kamera, model, vardiya, tespit, ürün lock, event engine, dashboard ve mail ayarlarını tek noktadan yönetmek.

### `run_product_service.py` (★ Windows Service Başlatıcısı)
**Görev:** Uygulamayı üretim ortamında arka planda çalıştırmak.

Servis başladığında:
1. Ağ/kamera için başlangıç gecikmesi uygulanır
2. Kamera + YOLO tespit thread'i başlatılır
3. Flask dashboard `0.0.0.0:8507` üzerinde açılır
4. Sistem 7/24 çalışmaya devam eder

---

## 🎯 Sistem Mantığı

```text
┌──────────────────────────────────────┐
│  1. RTSP kameradan görüntü alınır    │
│     PyAV / OpenCV                    │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  2. Görüntü YOLO modeline gönderilir │
│     confidence >= 0.70               │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  3. YOLO sonuçları + Norfair         │
│     takip sistemi                    │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  4. ProductVoteLock                  │
│     60 sn boyunca ürün oyları        │
│     toplanır ve baskın ürün seçilir  │
└──────────────────┬───────────────────┘
                   ↓
          ┌────────────────────┐
          │ Kararlı ürün var mı│
          └──────┬───────┬─────┘
              EVET       HAYIR
                ↓          ↓
┌──────────────────────┐  Ürün yok / bekle
│ ProductEventEngine   │
│ START / END / CHANGE │
└──────────┬───────────┘
           ↓
┌──────────────────────────────────────┐
│  5. Ürün başlangıç/bitiş süreleri    │
│     data/uretim_aylik.json'a yazılır │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  6. Dashboard                        │
│  Günlük / Aylık / Kamera / Mail      │
│  raporları görüntülenir              │
└──────────────────────────────────────┘
```

**Akış Özeti:**
1. Kamera görüntüsü sürekli alınır
2. YOLO görüntüdeki ürün sınıfını tespit eder
3. Norfair kısa süreli tespitleri takip eder
4. ProductVoteLock 60 saniyelik pencere ile baskın ürünü belirler
5. ProductEventEngine üretim başlangıç/bitiş/değişim kararını verir
6. Tamamlanan üretim aralığı aylık JSON dosyasına yazılır
7. Dashboard günlük ve aylık üretim sürelerini gösterir
8. Servis yeniden başlarsa `engine_state.json` sayesinde aktif üretim kaldığı yerden devam eder

---

## 💼 Web Arayüzü Dosyaları

### `static/dashboard.html` + `static/dashboard.js`
Ana üretim takip arayüzüdür.

**Dashboard Bölümleri:**
- **Genel Bakış** → Toplam üretim süresi, tamamlanan üretimler ve KPI bilgileri
- **Günlük Rapor** → Seçilen günün ürün başlangıç/bitiş ve süre kayıtları
- **Aylık Rapor** → Ay içerisindeki tüm üretim kayıtları
- **Kamera** → Canlı RTSP kamera görüntüsü ve aktif ürün bilgisi
- **Mail Geçmişi** → Gönderilen / gönderilemeyen mail kayıtları
- **Mail Ayarları** → Mail alıcı ve CC adresleri
- **Admin Paneli** → Kullanıcılar, şifreler, aktif oturumlar ve oturum logları

Dashboard masaüstü ve mobil kullanım için responsive olarak hazırlanmıştır.

---

## 🚀 Hızlı Başlangıç

### Kurulum

```bash
cd urun_tespit_26_08
python -m venv .venv_urun
.venv_urun\Scripts\activate
pip install -r requirements.txt
```

Aktif YOLO modelini aşağıdaki konuma yerleştirin:

```text
model/urun_tespit_24_06_best.pt
```

### Manuel Çalıştırma

```bash
python run_product_service.py
```

Ardından tarayıcıdan:

```text
http://yapayzeka:8507
```

adresine girin.

### Windows Service / NSSM

Üretim ortamında önerilen başlangıç dosyası:

```text
run_product_service.py
```

NSSM servisinde Python executable olarak proje sanal ortamındaki `python.exe`, argument olarak ise `run_product_service.py` kullanılmalıdır.

---

## 📊 Ürün Sınıfları

Ürün sınıfları `config.py` içinde sabit tutulmaz. Sistem sınıf isimlerini doğrudan aktif YOLO modelinden okur:

```python
model.names
```

Bu nedenle model değiştirilirse dashboard'daki ürün sınıfları da yeni modelin sınıf listesine göre otomatik güncellenir.

Mevcut üretim kayıtlarında görülen örnek sınıflar:

`nugget` | `schnitzel` | `citir_burger` | `citir_baby` | `mc_donalds_yaprak_kanat`

---

## ⚙️ Temel Ayarlar

| Parametre | Değer | Açıklama |
|-----------|-------|----------|
| Dashboard Portu | `8507` | Web arayüzü portu |
| Video Backend | `pyav` | Varsayılan RTSP kamera motoru |
| Confidence Threshold | `0.70` | YOLO minimum güven eşiği |
| Model Resize Scale | `0.5` | Görüntü modele yarı çözünürlükte gönderilir |
| Kamera Preview FPS | `15` | Dashboard canlı kamera hedef FPS değeri |
| Norfair Distance Threshold | `60` | Takip eşleştirme mesafesi |
| Product Lock Süresi | `60 sn` | Baskın ürün için oy toplama penceresi |
| Minimum Lock Gözlemi | `20` | Lock kararı için minimum gözlem |
| Minimum Baskınlık Oranı | `0.70` | Ürün lock kararı için gerekli oran |
| Minimum Yokluk Oranı | `0.70` | Lock kaldırmak için gerekli yokluk oranı |
| Minimum Geçerli Üretim | `60 sn` | Engine tarafındaki minimum ürün süresi |
| Ignore Gap | `20 sn` | Kısa tespit kopmalarını tolere etme süresi |
| Ürün Bitiş Süresi | `600 sn` | 10 dakika görünmeyen ürünün bitmiş sayılması |
| Ürün Değişim Onayı | `60 sn` | Yeni ürünün doğrulanma süresi |
| Minimum Ürün Değişim Aralığı | `3600 sn` | İki ürün değişimi arasında minimum 1 saat |
| Inference Throttle | `5 sn` | Ürün kilitliyken seyrek YOLO kontrolü |
| Minimum Rapor Süresi | `10 dk` | Daha kısa üretimler JSON'a kaydedilmez |
| Gündüz Vardiyası | `08:30` | Gündüz vardiyası başlangıcı |
| Gece Vardiyası | `20:30` | Gece vardiyası başlangıcı |
| Başlangıç Gecikmesi | `10 sn` | Windows açılışında ağ/kamera hazırlığı |
| Vardiya Maili | `False` | Varsayılan olarak kapalı |
| Ürün Olay Maili | `False` | START/END/CHANGE bildirimleri varsayılan kapalı |

---

## 📌 Önemli Notlar

- **Model Dosyası:** Aktif `.pt` model dosyası `model/` klasöründe bulunmalıdır.
- **Model Sınıfları:** Ürün isimleri doğrudan YOLO modelinden okunur; ayrıca sınıf listesi tutmaya gerek yoktur.
- **PyAV Önerilir:** `av` paketi kurulu değilse sistem OpenCV backend'ine düşebilir; ana kullanım PyAV olarak tasarlanmıştır.
- **Servis Durumu Korunur:** `data/engine_state.json`, servis veya bilgisayar yeniden başladığında aktif ürünün başlangıç zamanını korumak için kullanılır.
- **Üretim Kaydı JSON'dur:** `save_count_to_xlsx()` adına rağmen ana üretim kayıtları `data/uretim_aylik.json` dosyasına yazılır.
- **Excel Export:** Excel dosyası dashboard'dan rapor dışa aktarılırken veya vardiya mail eki hazırlanırken üretilir.
- **Kısa Üretimler:** `MIN_REPORT_DURATION_MINUTES = 10` nedeniyle 10 dakikadan kısa kayıtlar ana aylık JSON'a yazılmaz.
- **Vardiya Saatleri:** Saatler `config.py > SHIFTS` listesinden merkezi olarak değiştirilmelidir.
- **Kamera URL'si:** RTSP adresi `config.py > VIDEO_URL` üzerinden değiştirilir.
- **Kamera Önizleme Ayrıdır:** Dashboard videosu, YOLO inference hızından bağımsız ayrı bir preview thread'i ile yayınlanır.
- **Adaptif Inference:** Ürün değişimine daha uzun süre varsa YOLO her karede çalıştırılmaz; sistem yükünü azaltmak için belirli aralıklarla inference yapılır.
- **Mail Özellikleri:** Vardiya ve ürün olay mailleri birbirinden bağımsız olarak açılıp kapatılabilir.
- **Log Limiti:** Ana log dosyası yaklaşık 50.000 satırı geçtiğinde eski kayıtlar otomatik azaltılır.

---

## 👨‍💼 Maintainer

- Aleyna Kapusuz
- Hande Bandırmalı

---






## 👨‍💼 Maintainer

- Aleyna Kapusuz
- Hande Bandırmalı

---
