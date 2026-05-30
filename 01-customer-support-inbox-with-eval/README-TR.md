# Yapay Zeka Destekli Müşteri Hizmetleri Ajanı

> Yapay zeka destekli müşteri hizmetleri otomasyonu; LLM-as-Judge değerlendirmesi ve halüsinasyon engelleme — kurumsal düzeyde n8n iş akışı.

🇬🇧 İngilizce versiyon için [README.md](README.md) dosyasını görebilirsiniz.

---

## İçindekiler

- [Yapay Zeka Destekli Müşteri Hizmetleri Ajanı](#yapay-zeka-destekli-müşteri-hizmetleri-ajanı)
  - [İçindekiler](#i̇çindekiler)
  - [Problem](#problem)
  - [Çözüm ve Vizyon](#çözüm-ve-vizyon)
  - [Somut Çıktılar ve Başarı Kriterleri](#somut-çıktılar-ve-başarı-kriterleri)
  - [Özellikler](#özellikler)
  - [Mimari Özet](#mimari-özet)
  - [Mimari Kararlar](#mimari-kararlar)
    - [LLM-as-Judge](#llm-as-judge)
  - [Kurulum](#kurulum)
    - [Gereksinimler](#gereksinimler)
    - [Adımlar](#adımlar)
  - [Kullanım](#kullanım)
    - [Temel örnek](#temel-örnek)
  - [Konfigürasyon](#konfigürasyon)
  - [Deployment](#deployment)

---

## Problem

Manuel müşteri desteği yavaş, maliyetli ve insan hatasına açıktır. Kontrolsüz yapay zeka otomasyonları genellikle "halüsinasyon" riski taşır; yani AI sistemleri gerçek dışı politika veya fiyat bilgisi üretebilir. İşletmelerin, marka itibarını korumak için %100 doğruluk garantisiyle ve şirketin bilgi havuzuyla örtüşen otomatik yanıtlara ihtiyacı vardır.

---

## Çözüm ve Vizyon

Bu otomasyon sistemi, çift katmanlı bir doğrulama sistemi uygular: her AI taslağı, müşteriye gönderilmeden önce ikincil bir "Hakem LLM" tarafından bilgi havuzundaki verilerle karşılaştırılır. Bu otomasyonun amacı, rutin taleplerin %90'ını doğruluktan ödün vermeden otomatikleştirirken, karmaşık sorunlar için güvenli bir insan müdahalesi sağlamaktır.

---

## Somut Çıktılar ve Başarı Kriterleri

| Metrik             | Mevcut Durum      | Hedef (target) |
| ------------------ | ----------------- | -------------- |
| Yanıt Süresi       | 2–4 saat          | < 2 dakika     |
| Halüsinasyon Oranı | Bilinmiyor/Yüksek | < %1           |
| Otomasyon Oranı    | %0                | %70–85         |

> [!NOTE]
> Başarı, Hakem LLM'nin **"pass"** kararı vermesiyle tanımlanır. Her yanıt üç bağımsız boyutta (her biri 0–2) puanlanır:
> - **Kategori Tespiti** — 2: tam sorun türü eşleşmesi / 1: yakın ama tam olmayan eşleşme / 0: yanlış kategori
> - **Veri Tutarlılığı (Grounding)** — 2: tüm iddialar doğrulanabilir / 1: zararsız tek bir doğrulanamaz iddia / 0: halüsinasyon mevcut
> - **Eskalasyon Yönetimi** — 2: doğru karar / 1: eksik eskalasyon / 0: gerekli eskalasyon atlandı veya gereksiz eskalasyon yapıldı
>
> Karar: tüm boyutlar 2 → **pass** (otomatik gönder) · herhangi biri 1, hiçbiri 0 değil → **review** (insan kuyruğu) · herhangi biri 0 → **fail** (anında eskalasyon). Herhangi bir boyutun 0 alması, diğerlerinden bağımsız olarak her zaman **fail** kararı verir.

---

## Özellikler

- **Çok Aşamalı Sınıflandırma** — E-postaları Kurulum, Güvenlik, Fiyatlandırma, Faturalandırma, Hukuk, Satış veya İK kategorilerine ayırır.
- **RAG Entegrasyonu** — Bilgi havuzundan doğru bilgiyi getirmek için Supabase Vektör Veritabanı ve Gemini Embeddings kullanır.
- **LLM-as-Judge Değerlendirmesi** — Gemini 2.5 Flash tabanlı hakem LLM, yanıtları müşteriye göndermeden önce üç bağımsız boyutta (kategori, veri tutarlılığı, eskalasyon) puanlayarak denetler.
- **Güvenli Eskalasyon** — Karmaşık veya riskli durumları otomatik olarak insan temsilcilere yönlendirir.
- **Otomatik Bilgi Güncelleme** — Günlük çalışan scheduler, Google Sheets verilerini vektör veritabanıyla senkronize eder.

---

## Mimari Özet

```
[Webhook/Gmail] 
       ↓
[GPT-4o Mini Sınıflandırıcı]
       ↓
[AI Ajan + RAG]
       ↓
[Hakem LLM Değerlendirmesi]
       ├─→ pass (geç)    → [E-posta Gönder]
       ├─→ review (incele) → [İnsan Kuyruğu]
       └─→ fail (başarısız) → [İnsan Eskalasyonu]

[Günlük Zamanlanmış Görev] 
       ↓
[Sheets'i Supabase ile Senkronize Et]
```

**Teknoloji Yığını:**

| Teknoloji          | Rol                 | Seçilme Nedeni                                                 |
| ------------------ | ------------------- | -------------------------------------------------------------- |
| n8n                | Orkestrasyonu       | Low-code araç esnekliği.                                       |
| GPT-4o Mini        | Sınıflandırıcı/Ajan | Rutin görevler için yüksek performans-maliyet oranı.           |
| Gemini 2.5 Flash     | Hakem LLM           | Çok boyutlu değerlendirme için daha güçlü model.               |
| Gemini Embedding 2 | Vektörleştirme      | Belgeler için üstün semantik arama yetenekleri.                |
| Supabase           | Vektör Deposu       | Ölçeklenebilir, açık kaynaklı Postgres tabanlı vektör araması. |
| Google Sheets      | Bilgi Bankası       | Teknik olmayan personel için kolay veri güncelleme.            |

---

## Mimari Kararlar

### LLM-as-Judge

**Bağlam:** Yapay zeka ajanları bazen sistem talimatlarını görmezden gelebilir veya hatalı veri üretebilir (halüsinasyon). Tek katmanlı doğrulama üretim ortamı için yeterli değildir.

**Karar:** İlk ajanın çıktısını bilgi havuzundaki verilerle karşılaştırıp denetlemek ve puanlamak için tamamen ayrı bir LLM düğümü eklendi. Hakem, her yanıtı üç bağımsız boyutta değerlendirerek yapılandırılmış bir karar üretir.

**Rubrik:**

| Boyut | 2 — Tam puan | 1 — Kısmi puan | 0 — Başarısız |
|---|---|---|---|
| **Kategori Tespiti** | Tam sorun türü doğru tespit edildi (ör. fatura anlaşmazlığı, hesap ele geçirilmesi) | İlgili ama tam olmayan kategori (ör. fatura-hesap örtüşmesi yalnızca faturalama sorunu olarak ele alındı) | Tamamen yanlış kategori veya asıl sorun görmezden gelindi |
| **Veri Tutarlılığı (Grounding)** | Tüm politika, fiyat, zaman çizelgesi ve talimatlar doğrulanabilir; hiçbir şey uydurulmamış | Zararlı veya yanıltıcı olmayan tek bir doğrulanamaz iddia mevcut | Uydurulmuş politika, gerçek olmayan fiyat, yanlış zaman çizelgesi veya müşteriyi yanıltacak talimat içeriyor |
| **Eskalasyon Yönetimi** | Karmaşık/hassas sorunlar eskalasyona yönlendirildi; rutin sorunlar gereksiz eskalasyon olmadan çözüldü | Eskalasyon denendi ancak yetersiz (net bir yol gösterilmedi veya sorunlu self-servis tavsiyeden sonra eskalasyon yapıldı) | Gerekli eskalasyon tamamen atlandı ya da rutin bir istek için gereksiz eskalasyon eklendi |

**Karar mantığı:**

| Koşul | Karar | Eylem |
|---|---|---|
| Üç boyut da 2 puan aldı | **pass** | Yanıtı otomatik gönder |
| Herhangi biri 1 puan aldı, hiçbiri 0 değil | **review** | İnsan ajanı kuyruğuna yönlendir |
| Herhangi biri 0 puan aldı | **fail** | Anında eskalasyon |

> `grounding`, `escalation` veya `category` boyutundan 0 puan alınması, diğer puanlardan bağımsız olarak her zaman **fail** kararını zorlar.

**Diğer Seçenekler:**
- Daha güçlü sistem promptlarıyla tek ajan — ilgi alanlarının yeterli izolasyonunu sağlamaz.
- Her yanıt için manuel insan denetimi — darboğaz ve maliyet açısından verimsizdir.

**Ödünleşimler (Trade-off):**
- ✅ Hatalı yanıtlar önemli ölçüde azalır.
- ✅ AI çıktılarının doğruluğu boyut bazında otomatik denetlenir.
- ✅ Üç yönlü yönlendirme; sınır durumdaki yanıtların otomatik gönderilmesini veya sessizce düşürülmesini engeller.
- ⚠️ E-posta başına token maliyeti artar (~ticket başına 3 API çağrısı).
- ⚠️ Yanıt süresi biraz artar (yaklaşık +3 saniye).

---

## Kurulum

### Gereksinimler

- n8n (v1.5+ önerilir)
- OpenAI ve Gemini API anahtarı
- Google Cloud Console erişimi (Sheets ve Gemini için)
- Supabase hesabı

### Adımlar

1. `customer_support_inbox_evaluation.json` dosyasını bu repodan indirin.
2. n8n'de **Workflows** → **Import from File** seçeneğine tıklayın.
3. OpenAI, Google Gemini ve Supabase için kimlik bilgilerini yapılandırın.
4. Tüm Google Sheets düğümlerinde `YOUR_SPREADSHEET_ID` kısmını bilgi bankası tablo kimliğinizle güncelleyin.
5. **Webhook** düğümünün URL'sini gelen kutu webhook uç noktanız olarak ayarlayın (ör. Gmail yönlendirmesi veya Zapier).

---

## Kullanım

### Temel örnek

İş akışı aktifken, webhook URL'sine şu JSON gövdesiyle bir POST isteği gönderin:

```json
{
  "from": "musteri@ornek.com",
  "subject": "Kurulum sorunu",
  "body": "API bağlantısını nasıl yapabilirim?"
}
```

İş akışı şunları gerçekleştirir:
1. E-postayı bir kategoriye sınıflandırır.
2. İlgili belgeleri Supabase'den sorgulanır.
3. AI Ajan aracılığıyla bir yanıt oluşturur.
4. Yanıtın doğruluğu açısından değerlendirir.
5. E-postayı gönderir, insan inceleme kuyruğuna yönlendirir veya hakem kararına göre eskalasyon yapar.

---

## Konfigürasyon

| Değişken          | Zorunlu | Açıklama                                    |
| ----------------- | ------- | ------------------------------------------- |
| `openai_api_key`  | ✅       | Sınıflandırma ve denetleme için kullanılır. |
| `supabase_url`    | ✅       | Vektör veritabanı endpoint.                 |
| `spreadsheet_id`  | ✅       | Bilgi bankasını içeren Google Sheets ID'si. |
| `judge_threshold` | ❌       | Karar eşiği geçersiz kılma: **pass** için her boyutun alması gereken minimum puan (varsayılan: tümü 2). |

---

## Deployment

İş akışı n8n Cloud veya self-hosted ortamda etkin bir tetikleyici olarak çalışır. Ortam değişkenleri veya n8n Secrets aracılığıyla yapılandırın. Yüksek hacimli işlemler için **Queue Mode** etkinleştirerek API gecikmesinden kaynaklanan engellemeyi önleyin.