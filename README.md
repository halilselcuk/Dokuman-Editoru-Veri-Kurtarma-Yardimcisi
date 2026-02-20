# 🛡️ Doküman Editörü Veri Kurtarma Yardımcısı

> **UYAP Doküman Editörü** kilitlendiğinde bellekte kalan belge verilerini Windows bellek döküm (`.DMP`) dosyasından kurtarmaya yarayan, tamamen tarayıcı tabanlı tek sayfalık bir araç.

---

## İçindekiler

- [Genel Bakış](#genel-bakış)
- [Nasıl Çalışır?](#nasıl-çalışır)
- [Altı Fazlı Tarama Motoru](#altı-fazlı-tarama-motoru)
  - [Faz 1 — İmza Taraması (Chunked)](#faz-1--imza-taraması-chunked)
  - [Faz 2 — Şablon Eşleştirme](#faz-2--şablon-eşleştirme)
  - [Faz 3 — UTF-8 XML Şablon Çıkartma](#faz-3--utf-8-xml-şablon-çıkartma)
  - [Faz 4 — Sıkıştırılmış ZIP/UDF Girdileri](#faz-4--sıkıştırılmış-zipudf-girdileri)
  - [Faz 5a — UTF-16LE Java Heap Şablon Taraması](#faz-5a--utf-16le-java-heap-şablon-taraması)
  - [Faz 5b — Kaydedilmemiş GapContent Tampon Taraması](#faz-5b--kaydedilmemiş-gapcontent-tampon-taraması)
  - [Faz 6 — CDATA Blokları ve Türkçe Marker Taraması](#faz-6--cdata-blokları-ve-türkçe-marker-taraması)
- [Kullanım Adımları](#kullanım-adımları)
  - [Adım 1: Bellek Döküm Dosyası Alma](#adım-1-bellek-döküm-dosyası-alma)
  - [Adım 2: Dosyayı Araçla Yükleme](#adım-2-dosyayı-araçla-yükleme)
  - [Adım 3: Sonuçları İnceleme ve Dışa Aktarma](#adım-3-sonuçları-inceleme-ve-dışa-aktarma)
- [Teknik Mimari](#teknik-mimari)
  - [Dosya Yapısı](#dosya-yapısı)
  - [Kullanılan Teknolojiler](#kullanılan-teknolojiler)
  - [Temel Veri Yapıları](#temel-veri-yapıları)
  - [Önemli Fonksiyonlar](#önemli-fonksiyonlar)
  - [ZIP / UDF Oluşturma](#zip--udf-oluşturma)
  - [Deduplication (Tekrar Eleme)](#deduplication-tekrar-eleme)
- [Gizlilik ve Güvenlik](#gizlilik-ve-güvenlik)
- [Bilinen Kısıtlamalar](#bilinen-kısıtlamalar)
- [Tarayıcı Desteği](#tarayıcı-desteği)
- [Sık Sorulan Sorular (SSS)](#sık-sorulan-sorular-sss)

---

## Genel Bakış

UYAP Doküman Editörü Java tabanlı bir uygulamadır. Uygulama kilitlendiğinde veya yanıt vermez hale geldiğinde Windows Görev Yöneticisi aracılığıyla sürecin anlık bellek görüntüsü (memory dump / `.DMP` dosyası) alınabilir. Bu araç, söz konusu `.DMP` dosyasını tarayarak içinde kalmış belge içeriklerini (XML şablonları, CDATA metin blokları, sıkıştırılmış UDF dosyaları, Java heap verileri) bulmaya ve kurtarmaya çalışır.

**Temel özellikler:**

- Tamamen istemci tarafında çalışır; hiçbir veri sunucuya gönderilmez.
- Kurulum gerektirmez; tek bir `.html` dosyasıdır.
- Birden fazla kurtarma stratejisini (6 faz) art arda çalıştırarak kurtarma olasılığını artırır.
- Kurtarılan belgeleri `.udf` veya düz metin alma imkânı sunar.
- Büyük dosyaları (birkaç GB'a kadar) parça parça okuyarak bellek tüketimini kontrol altında tutar.

---

## Nasıl Çalışır?

UYAP Doküman Editörü belgeleri bellekte **XML tabanlı bir "template" formatı** içinde tutar. Bu şablonun köklü yapısı şöyledir:

```xml
<template format_id="...">
  <properties>...</properties>
  <content><![CDATA[ ... belge metni ... ]]></content>
</template>
```

Program kilitlendiğinde bu veri yapısı RAM'de canlı kalmaya devam eder. Windows Görev Yöneticisi'nin oluşturduğu `.DMP` dosyası, o andaki RAM içeriğinin ham bayt görüntüsüdür. Bu araç söz konusu ham bayt akışını çeşitli imzalar ve kodlamalar için tarar.

Bunun yanı sıra UYAP, son kaydedilmiş belge sürümünü **UDF formatında** (ZIP tabanlı kapsayıcı, `content.xml` + `documentproperties.xml` içeren) saklar. Bu sıkıştırılmış verinin bir anlık görüntüsü de dump içinde bulunabilir.

Son olarak, Java Swing düzenleyicisi (`javax.swing.text.GapContent`) klavyeden girilen ancak hiç kaydedilmemiş metni kendi iç karakter tamponunda (`char[]`, UTF-16LE) saklar. Faz 5b ve Faz 6 bu tamponları hedef alır.

---

## Altı Fazlı Tarama Motoru

Tarama `processFile()` ana fonksiyonu tarafından yönetilir ve aşağıdaki fazlar sırayla çalıştırılır.

### Faz 1 — İmza Taraması (Chunked)

**Hedef:** Dosyayı parça parça okuyarak tüm `<template format_id=` başlangıç imzalarını, `</template>` bitiş imzalarını ve `PK\x03\x04` (ZIP local file header) + `content.xml` dosya adı kombinasyonlarını saptar.

**Nasıl çalışır:**

- Dosya `~32 MB`'lık parçalar (chunk) halinde `File.slice()` + `FileReader.readAsArrayBuffer()` ile okunur.
- Parça sınırlarında imzaların kaybolmaması için her parça önceki parçayla **4 KB örtüşecek** şekilde okunur.
- Her chunk üzerinde `findPattern()` (Boyer-Moore benzeri basit KMP tarama) çalıştırılır.
- PK başlıkları için ZIP yerel dosya başlığı formatı (bayt konumu 26–30: dosya adı uzunluğu ve adı) doğrulanır.

```
[Offset 0]  PK\x03\x04  → ZIP local file header imzası
[Offset 26] uint16-LE   → dosya adı uzunluğu (fnameLen)
[Offset 30] fnameLen bayt → dosya adı ("content.xml" kontrolü)
```

İmzalar toplanıp sıralandıktan sonra **Set** ile tekrarlar kaldırılır.

---

### Faz 2 — Şablon Eşleştirme

**Hedef:** Faz 1'de bulunan başlangıç–bitiş imzaları çiftlere eşleştirilir.

**Algoritma (`matchTemplatePairs`):**

1. Bitiş ofseti sıralı tutulur.
2. Her başlangıç için, ondan sonra gelen ve **200 bayt < boyut < 2 MB** koşulunu sağlayan ilk (en yakın) bitiş seçilir.
3. Başlangıç ofsetleri **100 KB içinde** olan çiftler "kopya" kabul edilir ve bunların en büyüğü saklanır.

Bu eleme, Java JVM'nin aynı nesneyi birden fazla JVM iç bölgesine (eden space, survivor space, tenured space) kopyalamasından kaynaklanan mükerrer kayıtları temizler.

---

### Faz 3 — UTF-8 XML Şablon Çıkartma

**Hedef:** Faz 2'deki her çift için dosyadan ilgili bayt aralığı okunur, UTF-8 olarak çözülür ve geçerli bir XML şablonu olup olmadığı kontrol edilir.

**Geçerlilik kontrolleri:**

| Kontrol | Açıklama |
|---|---|
| `<content>` içermeli | Boş şablonlar atlanır |
| `</template>` içermeli | Kesik kayıtlar atlanır |
| `\u0000` veya `\uFFFD` içermemeli | Bozuk UTF-8 veya yanlış kodlama tespiti |

Geçerli şablonlardan metin içeriği `<![CDATA[...]]>` bloğu ayrıştırılarak çıkartılır. Bulunan her belge `recoveredDocuments` dizisine eklenir.

---

### Faz 4 — Sıkıştırılmış ZIP/UDF Girdileri

**Hedef:** Faz 1'de bulunan `PK\x03\x04` + `content.xml` imzalarından sıkıştırılmış (Deflate) verinin şifresini çözmek.

**Nasıl çalışır (`extractCompressedUDFs`):**

1. ZIP local file header'dan sıkıştırılmış veri boyutu ve yöntemi okunur.
2. Sıkıştırılmış ham baytlar `DecompressionStream('deflate-raw')` ile açılır (Web Streams API).
3. Açılan `content.xml` içeriği UTF-8 olarak çözülür ve `<template>` yapısı ayrıştırılır.
4. Başarısız decompress denemeleri sessizce atlanır; hata durumunda bir sonraki PK girişine geçilir.

---

### Faz 5a — UTF-16LE Java Heap Şablon Taraması

**Hedef:** Java String nesneleri UTF-16LE kodlamasında saklanır. JVM heap'inde `<template format_id=` dizisinin UTF-16LE karşılığı aranır.

**Nasıl çalışır (`searchUTF16Templates`):**

- `encodeUTF16LE('<template format_id=')` ile ikili imza oluşturulur.
- Dosya yeniden chunk'lanır; her chunk'ta imza aranır.
- Bulunan konumdan itibaren UTF-16LE olarak `</template>` bitiş imzası aranır.
- Veri `TextDecoder('utf-16le')` ile çözülür.
- Aynı geçerlilik kontrolleri uygulanır; geçerli belgeler `recoveredDocuments`'a eklenir.

---

### Faz 5b — Kaydedilmemiş GapContent Tampon Taraması

**Hedef:** Java Swing metin bileşeninin `GapContent` sınıfı yazılan metni iç bir `char[]` tamponunda UTF-16LE olarak tutar. Bu tampon, kullanıcı kaydetmeden önce uygulama kilitlenirse son klavye girişlerini içerebilir.

**Nasıl çalışır (`searchUnsavedContent`):**

- Bellekte Türkçe karakterler içeren yeterince uzun (genellikle > 100 karakter) UTF-16LE metin bloklarını arar.
- `GapContent` özelliğiyle oluşan belirgin bayt kalıpları (null gap baytları arasındaki metin segmentleri) kullanılır.
- Kurtarılan ham metin, sentetik bir `<template>` kabuğuna sarılarak sonuç listesine eklenir.
- Kaynak Faz etiketleri (`Faz 5b`) ayrı tutulur, böylece kullanıcı hangi fazdan geldiğini görür.

---

### Faz 6 — CDATA Blokları ve Türkçe Marker Taraması

**Hedef:** Hem UTF-8 hem UTF-16LE kodlamalarında bağımsız `<![CDATA[...]]>` bloklarını ve belirli Türkçe metin marker'larını içeren büyük metin tamponlarını tarar.

**Nasıl çalışır (`searchRawTextContent`):**

- `<![CDATA[` imzası hem UTF-8 hem UTF-16LE olarak aranır.
- CDATA blokları çıkartılır; yeterli uzunlukta (> 50 karakter) metin içerenler saklanır.
- Türkçe'ye özgü karakterler (`ş, ğ, ü, ö, ı, ç` vb.) içeren bloklar önceliklendirilir.
- Tam `<template>` yapısına uymayan ama anlamlı metin içeren "marker region" bulguları `allMarkerRegions` dizisine alınır; bunlar arayüzde **"Eşleşmeyen Marker Hit'leri"** sekmesinde gösterilir.

---

## Kullanım Adımları

### Adım 1: Bellek Döküm Dosyası Alma

> ⚠️ **KRİTİK:** Uygulama kilitlendiğinde **kesinlikle kapatmayın.** Kapatırsanız bellekteki veriler silinir.

1. `Ctrl + Shift + Esc` ile **Görev Yöneticisi**'ni açın.
2. Listede **UYAP Doküman Editörü** işlemini bulun (genellikle en yüksek CPU kullanımında).
3. İşleme **sağ tıklayın** → **"Bellek dökümü dosyası oluştur"** seçeneğini seçin.
4. Windows açılan iletişim kutusunda dosyanın kaydedildiği yolu gösterir. **"Yolu kopyala"** düğmesine tıklayın.

Oluşturulan `.DMP` dosyası genellikle birkaç GB büyüklüğündedir ve `%TEMP%\` klasörüne kaydedilir.

---

### Adım 2: Dosyayı Araçla Yükleme

1. `Dokuman-Editoru-Veri-Kurtarma-Yardimcisi.html` dosyasını bir tarayıcıda açın (Chrome veya Edge önerilir).
2. **"(2) Bellek Döküm Dosyasını Yükleyin"** bölümündeki alana tıklayın veya dosyayı sürükleyip bırakın.
3. Tarama otomatik başlar. İlerleme çubuğu hangi fazın çalıştığını ve mevcut konumu gösterir.
4. Dosya boyutuna göre tarama birkaç dakika ile onlarca dakika sürebilir. **Sayfayı kapatmayın.**

---

### Adım 3: Sonuçları İnceleme ve Dışa Aktarma

Tarama tamamlandığında sonuç ekranı üç sekme içerir:

| Sekme | İçerik |
|---|---|
| **Kurtarılan Belgeler** | Deduplicate edilmiş, en temiz belge listesi |
| **Tüm Bulunanlar** | Deduplicate uygulanmamış ham sonuçlar; faz bilgisi dahil |
| **Eşleşmeyen Marker Hit'leri** | Tam şablon yapısına uymayan ama anlamlı metin içeren ham bellek bölgeleri |

Her belge kartında:
- **Kaynak faz** ve **bellek ofseti** gösterilir.
- **"Ham XML Göster"** → orijinal `<template>` XML'ini gösterir.
- **"Metin Önizleme"** → insan tarafından okunabilir düz metin görünümü.
- **"UDF İndir"** → `content.xml` + `documentproperties.xml` içeren ZIP (.udf) olarak indirir.
- **"Metin İndir"** → `.txt` dosyası olarak indirir.

Üst kısımdaki **"Tümünü İndir"** çubuğu, kurtarılan tüm belgeleri tek tıkla ayrı ayrı `.udf` dosyaları olarak indirmeyi başlatır.

---

## Teknik Mimari

### Dosya Yapısı

```
Dokuman-Editoru-Veri-Kurtarma-Yardimcisi.html   ← Tek dosya; tüm CSS, HTML ve JS
```

Proje hiçbir harici bağımlılık veya build adımı içermez. Dosya doğrudan bir tarayıcıda açılabilir.

---

### Kullanılan Teknolojiler

| Teknoloji | Kullanım Amacı |
|---|---|
| **Vanilla JavaScript (ES2020+)** | Tüm iş mantığı |
| **File API** (`File`, `FileReader`, `Blob`) | Büyük dosyaları chunk'larla okumak |
| **Web Streams API** (`DecompressionStream`, `CompressionStream`) | ZIP Deflate açma/sıkıştırma |
| **TextDecoder / TextEncoder** | UTF-8 ve UTF-16LE dönüşümleri |
| **Uint8Array / ArrayBuffer** | Ham bayt manipülasyonu |
| **CSS Custom Properties (Variables)** | Tema renkleri |
| **CSS Flexbox** | Duyarlı (responsive) düzen |

Herhangi bir JavaScript çerçevesi (framework), harici kütüphane veya CDN bağlantısı **yoktur**.

---

### Temel Veri Yapıları

```javascript
// Kurtarılan bir belge nesnesi
{
    index: number,           // Görüntüleme sıra numarası
    webId: string,           // <template> etiketinden çıkartılan belge ID'si
    xmlContent: string,      // Tam <template>...</template> XML metni
    textContent: string,     // CDATA bloğundan çıkartılan saf metin
    size: number,            // XML içeriğinin karakter sayısı
    offsetStart: number,     // Dump dosyasındaki bayt başlangıç konumu
    offsetEnd: number,       // Dump dosyasındaki bayt bitiş konumu
    sourcePhase: string      // 'Faz 3', 'Faz 4', 'Faz 5a', 'Faz 5b', 'Faz 6', ...
}

// Tüm belge listeleri
let recoveredDocuments = [];         // Mevcut sonuç kümesi (dedup sonrası)
let allDocumentsBeforeDedup = [];    // Ham, dedup öncesi tüm sonuçlar
let allMarkerRegions = [];           // Faz 6 eşleşmeyen marker bölgeleri
let allMemoryTexts = [];             // Her fazdan bulunan tüm metin parçaları
```

---

### Önemli Fonksiyonlar

| Fonksiyon | Açıklama |
|---|---|
| `processFile(file)` | Ana tarama orkestratörü; fazları sırayla çalıştırır |
| `findPattern(data, pattern, maxResults)` | Uint8Array içinde ikili kalıp arar |
| `findPatternFrom(data, pattern, startOffset)` | Belirli bir ofsetten ileri doğru kalıp arar |
| `matchTemplatePairs(starts, ends)` | Başlangıç–bitiş imzalarını çiftleştirir |
| `extractCompressedUDFs(file, offsets)` | ZIP Deflate ile sıkıştırılmış UDF'leri çıkartır |
| `searchUTF16Templates(file)` | Faz 5a: UTF-16LE şablon taraması |
| `searchUnsavedContent(file)` | Faz 5b: GapContent karakter tamponu taraması |
| `searchRawTextContent(file)` | Faz 6: CDATA ve Türkçe marker taraması |
| `deduplicateDocuments(docs)` | Benzer içerikleri eleme; büyük olanı saklar |
| `createUDF(contentXml, docPropsXml)` | Geçerli bir ZIP/UDF dosyası oluşturur |
| `deflateData(data)` | `CompressionStream('deflate-raw')` ile sıkıştırır |
| `encodeUTF8(str)` | String → UTF-8 Uint8Array |
| `encodeUTF16LE(str)` | String → UTF-16LE Uint8Array |
| `decodeUTF8(bytes)` | UTF-8 Uint8Array → String |
| `buildZip(entries)` | Sıfırdan geçerli ZIP byte dizisi oluşturur |
| `updateProgress(pct, msg)` | İlerleme çubuğunu ve durum mesajını günceller |

---

### ZIP / UDF Oluşturma

Kurtarılan her belge, UYAP tarafından açılabilmesi için **geçerli bir ZIP arşivi** olarak paketlenir.

ZIP yapısı `buildZip(entries)` tarafından sıfırdan oluşturulur:

```
[Local File Header 1]  PK\x03\x04 + meta + dosya adı
[Compressed Data 1]    Deflate verisi (CompressionStream)
[Local File Header 2]  …
[Compressed Data 2]    …
[Central Directory]    Her giriş için Central Directory Header
[End of Central Dir]   PK\x05\x06 + özet bilgiler
```

Arşiv iki dosya içerir:
- `content.xml` — Kurtarılan belge XML içeriği
- `documentproperties.xml` — Varsayılan boş properties XML'i

`CompressionStream` API'si desteklenmiyorsa (eski tarayıcılar) veri sıkıştırılmadan (STORE yöntemi, method = 0) paketlenir.

CRC-32 hesabı `buildZip()` içinde tablolu CRC-32 algoritmasıyla gerçekleştirilir.

---

### Deduplication (Tekrar Eleme)

`deduplicateDocuments(docs)` fonksiyonu farklı fazların aynı belgeyi birden fazla kez bulması durumunda tekilleştirme yapar:

**Parmak izi (fingerprint):**
```
metinİlk500Karakter + "|" + webId + "|" + sourcePhase
```

- Aynı parmak ize sahip belgeler arasından **boyutu (XML karakter sayısı) en büyük olanı** korunur.
- Farklı fazlardan gelen sonuçlar ayrı tutulur (kullanıcı hangi fazın ne bulduğunu görebilsin diye `sourcePhase` de parmak izine dahil edilir).
- Boş metin içeriğine sahip belgeler sonuç listesinden çıkarılır.

---

## Gizlilik ve Güvenlik

- **Hiçbir veri dışarıya gönderilmez.** Tüm işlemler tarayıcının JavaScript motoru içinde gerçekleşir.
- **Sunucu bağlantısı yoktur.** Araç internet bağlantısı olmadan da çalışır.
- `.DMP` dosyasının içeriği yalnızca tarayıcı belleğinde işlenir; diske yazılmaz.
- Hassas içerikler (kişisel veriler, mahkeme belgeleri vb.) içeren dump dosyaları için tarama sonrası `.DMP` dosyasının güvenli biçimde silinmesi önerilir.

---

## Bilinen Kısıtlamalar

| Durum | Açıklama |
|---|---|
| **JVM Garbage Collector** | Java'nın GC'si verileri taşımış veya temizlemiş olabilir; bu durumda kurtarma başarısız olur. |
| **Windows bellek yönetimi** | İşletim sistemi bazı bellek sayfalarını sıfırlamış veya geri yazmış olabilir. |
| **Hiç kaydedilmemiş belge** | Eğer belge hiç kaydedilmediyse `<template>` yapısı hiç oluşmamış olabilir; yalnızca Faz 5b/6 ham metin bulabilir. |
| **Çok büyük dosyalar** | Gigabaytlarca büyüklükteki dump dosyaları için tarama onlarca dakika sürebilir; sekmeyi kapatmayın. |
| **Sonuçlar doğrulanmamış adaylardır** | Kurtarılan belgeler kesin doğrulanmış kayıtlar değil, adaylardır; içeriği açmadan önce gözden geçirin. |
| **CompressionStream desteği** | Çok eski tarayıcılarda `CompressionStream` yoksa UDF çıktısı STORE yöntemini kullanır (daha büyük dosya). |

---

## Tarayıcı Desteği

| Tarayıcı | Durum |
|---|---|
| Google Chrome 89+ | ✅ Tam destek |
| Microsoft Edge 89+ | ✅ Tam destek |
| Mozilla Firefox 102+ | ✅ Tam destek |
| Safari 16+ | ⚠️ Kısmi (CompressionStream 'deflate-raw' desteği sınırlı olabilir) |
| Internet Explorer | ❌ Desteklenmiyor |

**Önerilen:** Google Chrome veya Microsoft Edge (Chromium tabanlı) güncel sürüm.

---

## Sık Sorulan Sorular (SSS)

**S: Araç çalışmıyor, sonuç bulunamadı. Ne yapmalıyım?**

Şunları kontrol edin:
1. Dump dosyasını almadan önce uygulamayı kapattıysanız kurtarma mümkün olmayabilir.
2. Belge hiç kaydedilmemişse (yeni oluşturulmuş ve hiç Kaydet yapılmamışsa) şablon yapısı bellekte bulunmayabilir.
3. Dump alındıktan uzun süre sonra tarama yapıyorsanız bellekteki veriler geçersiz kalmış olabilir. Dump'ı mümkün olan en kısa sürede tarayın.

**S: `.DMP` dosyası çok büyük. Sayfayı kapatırsam ne olur?**

Tarama duraklar ve tüm ilerleme kaybolur. Sayfa açık kaldığı sürece tarama devam eder.

**S: Kurtarılan belgeyi UYAP'ta açabilir miyim?**

İndirdiğiniz `.udf` dosyasını UYAP Doküman Editörü ile açmayı deneyebilirsiniz. İçerik bütünlüğü kurtarma koşullarına bağlı olarak değişebilir.

**S: Aynı belge birden fazla kez listeleniyor.**

"Kurtarılan Belgeler" sekmesi zaten deduplicate uygulanmış sonuçları gösterir. "Tüm Bulunanlar" sekmesinde kasıtlı olarak ham, tekrarlı sonuçlar da yer almaktadır; farklı fazların ne bulduğunu karşılaştırmak için kullanılabilir.

**S: Araç hangi UYAP sürümleriyle çalışır?**

XML `<template format_id=...>` yapısını kullanan tüm UYAP Doküman Editörü sürümleriyle çalışması beklenir. UDF formatı (ZIP tabanlı) değiştirildiğinde Faz 4 güncellenmesi gerekebilir.

---

*Bu araç tamamen tarayıcı tabanlı olup herhangi bir kurulum, sunucu veya internet bağlantısı gerektirmez.*
