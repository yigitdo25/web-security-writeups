# Reflected XSS into HTML context with most tags and attributes blocked — Writeup

**Platform:** PortSwigger Web Security Academy
**Zorluk:** Practitioner
**Tarih:** 2026-11-25

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (arama fonksiyonu, `?search=` parametresi)
- **Tespit Edilen Zafiyet:** Reflected XSS — arama terimi gövde (HTML text) context'ine encode'suz yansıyor. Araya bir **WAF/filtre** konmuş: tag ve attribute'ların **çoğunu** kara listeyle engelliyor ("Tag is not allowed" / "Attribute is not allowed").
- **Bypass:** Filtre bir **kara liste** (tümünü değil, çoğunu engeller). İzin verilen tag (`body`) ve o tag üzerinde izin verilen, **dışarıdan tetiklenebilir** bir event handler (`onresize`) brute-force ile bulunup birleştirildi.
- **Genel Etki:** Linke tıklayan / exploit sayfasını ziyaret eden kullanıcıda keyfi JavaScript çalışır.
- **Durum:** Çözüldü — exploit server'dan iframe ile teslim edildi, `print()` tetiklendi.
- **Tür:** Reflected XSS (filtre bypass — allowlist-discovery via brute-force)

---

## Reconnaissance & Vulnerability Analysis

### 1. Yansıma + context tespiti
`?search=test123` gönderildi, response'ta (`Ctrl+U` / Repeater) arandı:
```html
<h1>0 search results for 'test123'</h1>
```
- `test123`, `<h1>` ile `</h1>` **arasında** = **gövde (HTML text content)** context.
- Çevresinde HTML attribute tırnağı yok → **breakout gerekmez**, doğrudan tag enjekte edilebilir. (Sayfanın yazdığı `'...'` süsleme tırnakları breakout için sayılmaz.)

### 2. Filtrenin varlığı
Bilinen bir payload denendi:
```
<img src=x onerror=alert(1)>   →   "Tag is not allowed"
```
- Filtre **tag ADINA** bakıyor → bir tag kara/beyaz listesi var.
- Lab başlığı: "**most** tags and attributes blocked" → **çoğu** engelli = **bazıları izinli.** Görev: izinli tag + izinli event handler'ı bulmak.

---

## Exploitation

### Adım 1 — İzin verilen TAG'i bul (Burp Intruder, 1. tur)
- İsteği Intruder'a gönder. Pozisyon: `GET /?search=<§§> HTTP/2` → `<` ve `>` sabit, Intruder **aralarına tag adı** koyacak.
- Payload listesi: PortSwigger **XSS cheat sheet → "copy tags to clipboard"** (tüm HTML tag'leri, ~143).
- Sniper + Simple list + **Grep-Match** `Tag is not allowed` → Start attack.
- **Ayıklama:** Community'de sonuç filtreleme Pro özelliği → bunun yerine **Length sütununa göre sırala.** Engelliler hep aynı "Tag is not allowed" cevabını (aynı Length) döndürür; izinli olan **ayrık Length** ile ayrılır.
- **İzinli tag'ler:** `body` (ve uydurma `xss` gibi bilinmeyen tag'ler de geçti → filtrenin **kara liste** olduğunun kanıtı: tanımadığını geçiriyor). `body` seçildi — olayları (onload/onresize...) zengin destekler.

### Adım 2 — İzin verilen EVENT HANDLER'ı bul (Intruder, 2. tur)
- Yeni Intruder sekmesi. Tag'i sabitle, pozisyonu attribute'a taşı: `GET /?search=<body%20§§=1> HTTP/2`
  - `%20` = boşluk (tag adı ile attribute arası; istek satırında ham boşluk kullanılmaz, URL'de `%20`).
- Payload listesi: cheat sheet **→ "copy events to clipboard"** (tüm `on...` handler'ları).
- Length'e göre sırala → izinli handler'lar: `oncontentvisibilityautostatechange`, `onwebkitfullscreenchange`, **`onresize`** (vd.).
- **Not:** `onresize` Community sonuç listesinde aranamadı (Pro) → **Repeater'da direkt test** edilerek izinli olduğu doğrulandı (`<body onresize=print()>` → engellenmedi).

### Adım 3 — Hangi event? "İzinli" ≠ "tetiklenir"
Filtreden geçmek yetmez; olayın victim'de **ateşlenmesi** gerekir. Lab victim'i bir **bottur**: sayfayı yalnızca **yükler**, tıklamaz / mouse gezdirmez / fullscreen'e geçmez.
- `onclick`, `onmouseover`, `onwebkitfullscreenchange` → **kullanıcı jesti** ister → bot tetiklemez. (Fullscreen ayrıca tarayıcı tarafından gerçek jeste kilitli.) → **elendi.**
- `onresize` → elementin **boyutu değişince** ateşlenir → **dışarıdan zorlanabilir** (iframe'i yeniden boyutlandırarak). → **seçildi.**
- `oncontentvisibilityautostatechange` → tetiklenebilir ama daha dolambaçlı → yedek.

### Adım 4 — Teslimat (exploit server, iframe resize trick)
Çekirdek payload (gövde context, breakout'suz):
```
<body onresize=print()>
```
- **`print()` (alert değil):** victim Chrome kullanıyor; Chrome **cross-origin iframe**'den gelen `alert`/`confirm`/`prompt`'u bloklar → `print()` ile solve tetiklenir.

Exploit server body'sine konan tam exploit:
```html
<iframe src="https://<lab-id>.web-security-academy.net/?search=<body onresize=print()>" onload="this.style.width='100px'">
```
**Mantık:** iframe lab sayfasını payload'la yükler → `onload` iframe genişliğini değiştirir → içindeki sayfa **yeniden boyutlanır** → `<body onresize>` ateşlenir → `print()`.

**Store → Deliver to victim** → lab çözüldü.

> Bu teslimat deseni, jQuery `hashchange` lab'ındakiyle **birebir aynı:** kendiliğinden ateşlenmeyen bir olayı, exploit server'daki iframe'in `onload`'uyla **dışarıdan zorlamak.** Orada `src` (hash) değişiyordu; burada **boyut** değişiyor.

---

## Remediation

- **Kara liste değil, context-aware output encoding:** kullanıcı girdisini HTML gövdesine basarken `<`, `>`, `&`, `"` vb. karakterleri entity'ye çevir (`<`→`&lt;`...). Kara liste her zaman eksiktir (uydurma `xss` tag'inin geçmesi bunun kanıtı).
- Mümkünse girdiyi HTML'e hiç sokma; `textContent` / güvenli template kullan.
- CSP (Content-Security-Policy) ile satır içi script/handler'ları kıs (derinlemesine savunma).

---

## Gotchas (Dikkat Edilecekler)

- **"most ... blocked" = kara liste ipucu** → tümünü değil çoğunu engeller; **izin verileni ara.** Uydurma bir tag'in (`xss`) geçmesi → kara liste (allowlist değil) olduğunu doğrular.
- **Brute-force iki turlu:** önce izinli **tag**, sonra o tag üzerinde izinli **event handler.** Araç: Burp Intruder + cheat sheet listeleri.
- **Community'de sonuç arama/filtre Pro.** Çözüm: **Length'e göre sırala** (ayrık = izinli) ve/veya adayı **Repeater'da direkt test et.**
- **"İzinli" ≠ "exploitable" ≠ "ateşlenir":** victim bot **etkileşmez** → sadece **otomatik** (`onload`/`onerror`) ya da **dışarıdan zorlanabilen** (`onresize`) olaylar işe yarar. Etkileşim/fullscreen gerektirenler çöp.
- **onresize teslimatı = iframe resize trick:** `onload="this.style.width='100px'"` → içteki sayfa boyutlanır → onresize. (hashchange lab'ının kardeşi.)
- **print() vs alert:** cross-origin iframe + Chrome victim → `alert` bloklanır, `print()` kullan.
- **Gövde context → breakout yok:** `<h1>...GİRDİ...</h1>` text content → doğrudan `<body ...>`; `">` gerekmez.
- **Sözdizimi (teslimat):** iframe boyutunu değiştirmek = **atama** → `this.style.width='100px'` (property = value; `this.src`'yi *okumak* bir şey değiştirmez).

---

## Zincir Özeti

```
?search=test123  →  <h1>...'test123'...</h1>  (gövde context, encode yok)
  → <img ...> denendi → "Tag is not allowed"  (tag kara listesi)
  → Intruder 1. tur (tag listesi, Length-sort) → izinli tag: body
  → Intruder 2. tur (event listesi, <body §§=1>) → izinli event: onresize
  → onresize seçildi (bot etkileşmez → otomatik/zorlanabilir olay şart)
  → <body onresize=print()>  (print: Chrome cross-origin iframe alert'i bloklar)
  → exploit server iframe: src=lab?search=PAYLOAD + onload=this.style.width='100px'
  → iframe yüklenir → genişlik değişir → sayfa resize → onresize → print()  → ÇÖZÜLDÜ
```
