# Reflected XSS with event handlers and `href` attributes blocked — Writeup

**Platform:** PortSwigger Web Security Academy
**Zorluk:** Expert
**Tarih:** 2026-11-28

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (arama fonksiyonu, `?search=`)
- **Tespit Edilen Zafiyet:** Reflected XSS — gövde (HTML text) context'ine encode'suz yansıma. Filtre hem **event handler'ları** (`on*`) hem **`href` attribute'larını** engelliyor ("Attribute is not allowed"). Tag'ler serbest.
- **Teknik (arcane): SVG attribute smuggling** — `<animate attributeName=href values=javascript:alert(1)>` bir `<a>`'nın href'ini **runtime'da** `javascript:` yapar → statik `href` hiç yazılmadığı için filtre görmez.
- **Tetikleme:** href=javascript: **tıklama** ile ateşlenir. Lab'ın simüle kullanıcısı (bot) sadece metninde **"Click"** geçen linklere tıklar → etiket **"Click me"** olmalı. Exploit server yok.
- **Durum:** Çözüldü — `<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a></svg>`
- **Tür:** Reflected XSS (filtre bypass — SVG attribute smuggling)

---

## Reconnaissance & Vulnerability Analysis

- `?search=test123` → `<h1>0 search results for 'test123'</h1>` = **gövde/text** context (kanıt: `<a>` koyunca tag olarak parse edildi → "0 search results for ''"). Breakout gerekmez.
- Filtre yüzeyi (testlerle):
  - **Tag'ler serbest** (`<a>`, `<svg>` geçti).
  - **href** → "Attribute is not allowed".
  - **event handler'lar** (`onclick` vb.) → engelli.
- **Çerçeve:** iki ana vektör (event handler + href) kapalı → *"Event handler OLMADAN ve statik href OLMADAN JS'i nasıl çalıştırırım?"* Çözüm, bir attribute'u **statik yazmadan, runtime'da set etmek.**

---

## Exploitation

### Teknik: SVG attribute smuggling
- SVG'nin **animasyon** elementleri (`<animate>`, `<set>`) **deklaratif** (JavaScript'siz) olarak bir attribute'u değiştirebilir.
- `<animate attributeName=href values=javascript:alert(1)>` → hedef `<a>`'nın **href**'ini `javascript:alert(1)`'e set eder. Sen hiç statik `href=` yazmadığın için → statik attribute filtresi **görmez** → bypass.
- `attributeName` = animate edilen attribute'un **ADI** (`href`); yeni değer `values`'ta. **Filtre attribute ADLARINA bakar, değere değil** — bu yüzden `attributeName=href` (değeri "href") sorun olmaz.

### Payload (final)
```html
<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a></svg>
```
- `<a>` içinde: `<animate>` (href'i set eder) + `<text>Click me</text>` (tıklanacak görünür link).
- Tıklanınca anchor'ın (artık `javascript:`) href'i çalışır → alert.

### Teslimat / tetikleme
- **Exploit server YOK.** Payload'ı `?search=` URL'ine koymak yeter.
- Lab'ın **simüle edilmiş kullanıcısı (bot)** sayfadaki linklere otomatik tıklar — ama **sadece metninde "Click" geçen** linklere (cazip/kurbanın tıklayacağı türden, social-engineering modeli).
- Bot "Click me"ye tıklar → alert **victim bağlamında** çalışır → **solved.**

---

## İterasyon tarihçesi (ne denendi, neden başarısız)

1. **Başlangıç = araştırmayla bulunan advanced vektör** (`dur`, `repeatCount`, `keytimes`, `&semi;`, çoklu/decoy `values` — bir WAF'ı yenmek için tasarlanmış). Tüm modifikasyon bunun **ÜZERİNDE** yürüdü. → **İlk iş: WAF-ekstralarını sıyır** (bu lab'da gereksiz) → sade `attributeName=href values=javascript:alert(1)` yapısı.
2. Kalan `<animate xlink:href=#xss ...>` (dışarıdan hedefleme) → **"Attribute is not allowed".** Sebep: **`xlink:href` "href" içeriyor**, filtre kaba string-eşleşmesiyle onu da kesiyor. → **Fix: nested** (animate'i `<a>` içine al, `xlink:href`/`id` at).
3. `attributeName=click` → **yanlış.** `attributeName`, set edilecek attribute'un adı = `href`. ("click" sezgisi doğruydu ama **yanlış yere** konmuştu — click'in evi `<text>` etiketiydi.)
4. `<text>XSS</text>` → payload geçti, kendi tıklamamda alert çıktı (POC ✓) **ama lab solved olmadı.** Sebep: **bot "XSS" linkine tıklamaz** (sadece "Click"). → **Fix: `<text>Click me</text>`.**

---

## Remediation

- **Birincil: context-aware output encoding** (`<`→`&lt;` vb.). Reflected XSS'in kök çözümü bu.
- **SVG animasyon savunması:** user content'te `attributeName`'e izin verme, ya da hangi attribute'ların animate edilebileceğini **kısıtla** (sadece guard'sız, global-güvenli attribute'lar). Kara liste ("href'i kes") yetersiz — smuggling bunu atlar.
- Sanitizer (JS'te **DOMPurify**) kullan ve **güncel tut** (bu teknik gerçek, yakın tarihli bir sanitizer zafiyeti). CSP ile derinlemesine savunma.

---

## Gotchas (Dikkat Edilecekler)

- **Filtre/WAF girdiyi STRING olarak tarar, execution'ı değil** → `xlink:href` "href" içerdiği için takıldı. Bypass = yasak **pattern**'i gizle (nested ile substring'i kaldır, encode et), execution'ı değil.
- **Nested şart:** `xlink:href`'ten kaçmak için animate'i `<a>` İÇİNE koy → doğrudan parent'ı hedefler, "href" substring'i kalmaz.
- **`attributeName` = attribute ADI; değer `values`'ta.** Filtre adlara bakar → `attributeName=href` sorun değil.
- **Simüle victim sadece "Click" linklerine tıklar** → `<text>` "Click me" olmalı, "XSS" olmaz. (Social-engineering modeli: kurban cazip linke tıklar.)
- **Self-click alert çıkarır (POC) ama solved etmez** → bot (victim) tıklamalı.
- **Bu lab'da exploit server yok** — URL yeter, bot kendi tıklar.
- **Gerçek-dünya bağı:** bu, gerçek bir yamalanmış zafiyet sınıfı — **OWASP Java HTML Sanitizer** advisory (SVG attribute smuggling, yama `20261102.2`). **Kompozisyon-seviyesi bypass:** sanitizer `values`'ı düz string denetler ama `animate`'in o değeri runtime'da `href`'e **yazdığını** kaçırır. Parçalar tek tek güvenli, birleşimi değil.
- **"click sezgisi" dersi:** bir şeyin önemli olduğuna dair sezgi yanlış yerde görünse bile atma — "başka bir yere mi ait?" diye sor. (Burada "click" attributeName'e değil text'e aitti.)

---

## Zincir Özeti

```
?search=<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a></svg>
  → gövde context, tag'ler serbest ama href + on* engelli ("Attribute is not allowed")
  → <animate attributeName=href values=javascript:alert(1)> → <a>'nın href'ini RUNTIME'da set eder
     (statik href yazılmadı → filtre görmedi; nested → "href" substring'i yok)
  → bot "Click me" linkine tıklar (sadece "Click" linklerine tıklar)
  → anchor'ın javascript: href'i çalışır → alert VICTIM bağlamında → solved
```

---

## Kaynaklar

- **PortSwigger lab:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-event-handlers-and-href-attributes-blocked
- **SVG `animate` tekniğinin kaynağı:** Gareth Heyes — "SVG animate XSS vector", PortSwigger Research (2020): https://portswigger.net/research/svg-animate-xss-vector
- **Gerçek-dünya bağı — OWASP Java HTML Sanitizer, SVG animation attribute smuggling (advisory):** _(link eklenecek — sende varsa)_
