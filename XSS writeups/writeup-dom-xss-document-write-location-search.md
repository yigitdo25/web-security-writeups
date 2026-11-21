# DOM XSS in document.write Sink using source location.search — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-19

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Apprentice/Practitioner)
- **Tespit Edilen Zafiyet:** DOM-based XSS — client-side JS, `location.search`'ten (saldırgan kontrollü) aldığı veriyi `document.write` sink'ine ham olarak geçiriyor, bir `<img>` attribute'u içine
- **Genel Etki:** Saldırgan, URL'e payload gömerek kurbanın tarayıcısında keyfi JavaScript çalıştırabilir
- **Durum:** Tamamlandı — `"><script>alert(1)</script>` payload'ı ile attribute breakout, lab çözüldü
- **İlkler:** İlk DOM-based XSS + ilk attribute context breakout

---

## Reconnaissance & Vulnerability Analysis

### Source → Sink zinciri
- **Source:** `location.search` → URL'in `?search=...` kısmı (saldırgan kontrolünde)
- **Sink:** `document.write()`
- Zafiyetli kod:
  ```javascript
  function trackSearch(query) {
      document.write('<img src="/resources/images/tracker.gif?searchTerms=' + query + '">');
  }
  var query = (new URLSearchParams(window.location.search)).get('search');
  if(query) {
      trackSearch(query);
  }
  ```
- `query` (location.search'ten) → `document.write` ile **ham birleştirme** (`'+ query +'`) → bir `<img src="...">` attribute'u içine. Hiçbir encode/sanitization yok.

### Context tespiti (metodoloji)
- **Önemli ders:** `document.write` çıktısı client-side üretilir → **"View source" (Ctrl+U) içinde GÖRÜNMEZ** (o sadece sunucu HTML'i). Sink'i bulmak için ya **canlı DOM** (devtools → Elements), ya da doğrudan **JS kodu** aranır (`document.write` / `location.search` araması).
- **Kod > render takibi:** render olmuş img (bozuk/görünmez tracker görseli) gözden kaçabilir; JS kodunu okumak sink'i ve context'i kesin gösterir.
- Context: girdi `<img src="...searchTerms=GİRDİ">` → **çift tırnaklı HTML attribute** içinde.

---

## Exploitation

### Breakout mantığı (attribute → yeni etiket)
Girdi `<img src="...searchTerms=` **BURADA** `">` konumunda. Üç adım:
1. `"` → `src` değerini **kapat** (çift tırnaklı değeri sonlandırır)
2. `>` → `<img>` etiketini **kapat** (artık dışarıdasın)
3. `<script>alert(1)</script>` → kendi çalıştırılabilir etiketini **sok**

### Payload
```
"><script>alert(1)</script>
```
`?search=`'e konur (URL ya da arama kutusu).

### Üretilen HTML
```html
<img src="/resources/images/tracker.gif?searchTerms="><script>alert(1)</script>">
```
- `"` → src kapandı (`src = ...searchTerms=`)
- `>` → img kapandı
- `<script>alert(1)</script>` → img'nin **dışında**, çalışır
- Sondaki `">` → zararsız kalıntı metin

### Teslimat
URL'e payload'lı `?search=...` ile git (ya da arama kutusuna yaz) → `document.write` sırasında script yazılır ve çalışır → lab çözüldü.

---

## Remediation

- **DOM sink'lere kullanıcı girdisi ile yazma:** `location.search`/`location.hash` gibi source'lardan gelen veriyi `document.write`/`innerHTML`'e ham geçirme.
- **Context-aware encode:** attribute'a basılıyorsa en azından `"`, `<`, `>` encode edilmeli.
- **Güvenli API:** `document.write` yerine `textContent` / `createElement` + güvenli DOM API kullan.
- Mümkünse `document.write`'ı tamamen bırak (zaten modern olmayan, tehlikeli bir API).

---

## Gotchas (Dikkat Edilecekler)

- **Kod vs runtime değeri:** JS kaynağında `query` (değişken) görürsün, literal `test123` değil — kod bir şablondur, girdiyi runtime'da alır. Gerçek değeri görmek için debugger'da breakpoint + hover (ya da canlı DOM çıktısı). Kodda "test123 yok" **normaldir.**
- **`document.write` çıktısı view-source'ta yok:** client-side üretildiği için Ctrl+U'da görünmez. Canlı DOM (Elements) ya da JS kodunu ara. Sink avlarken render yerine **kodu oku** — daha güvenilir.
- **`document.write` istisnası (kritik):** HTML sink'lerde genelde ham `<script>` çalışmaz (`innerHTML` böyle → event handler gerekir). AMA `document.write` ile yazılan `<script>` **ÇALIŞIR.** Bu yüzden burada hem `"><script>alert(1)</script>` hem `"><svg onload=alert(1)>` olur.
- **Attribute breakout = `">`:** çift tırnaklı attribute + etiketten çıkmak için `"` (değeri kapat) + `>` (etiketi kapat), sonra yeni etiket. Bu kalıp sonraki tüm attribute-context lab'larının temeli.
- **İki yansımayı karıştırma:** search terimi hem `<h1>... 'girdi'</h1>` başlığında (sunucu yansıması, view-source'ta) hem de `document.write` img'sinde (DOM sink) görünür. Exploit edilecek olan **document.write sink'i** (attribute context), h1 değil.
- **Source → sink + sanitization yokluğu, üçü birden** gerekli; burada `'+query+'` ham birleştirme = sanitization yok.

---

## Zincir Özeti

```
URL ?search="><script>alert(1)</script>   (source: location.search)
  → query = o değer
  → document.write('<img src="...searchTerms=' + query + '">')   (sink)
  → <img src="...searchTerms="> <script>alert(1)</script> ">
     " src'yi kapatır · > img'yi kapatır · <script> dışarıda çalışır
  → alert
```

---

## Varyant: ... inside a select element (2026-11-19)

Aynı sink (`document.write`) + aynı source (`location.search`), ama girdi bir **`<select>` elementinin içine** (stok kontrolü dropdown'ı) düşüyor; parametre `storeId`.

- Zafiyetli kod: `document.write('<option selected>' + store + '</option>')` — `store` = `location.search`'ten `storeId`
- **Context farkı:** girdi attribute içinde DEĞİL, `<option>` **text content**'inde → tırnak (`"`) gerekmez. Ama `<select>` içinde etiketler çalışmaz → önce **elementi kapat**.
- **Breakout:** `</select>` (sarmalayan select'i erken kapat) → sonra event-handler'lı etiket
- **Payload:** `storeId=</select><img src=1 onerror=alert(1)>` (URL'de productId=1 de gerekli — sayfa yüklensin diye)
- **Ders:** breakout context'e göre değişir — attribute → `">`, sarmalayan element → `</tag>`. (Detay: `notlar-xss-kavramlar.md` → Breakout genel prensibi)
- Writeup yerine kavram notuna işlendi (aynı sink ailesi, yeni olan tek şey element-breakout prensibi — lab'lar-üstü).
