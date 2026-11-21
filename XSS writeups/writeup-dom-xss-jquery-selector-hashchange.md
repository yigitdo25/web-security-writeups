# DOM XSS in jQuery Selector Sink using a hashchange Event — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-20

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** DOM-based XSS — sayfa, `location.hash`'i (saldırgan kontrollü) jQuery `$()` **selector sink'ine** geçiriyor; `$()` HTML gibi string'i element olarak **yaratıyor**
- **Genel Etki:** Saldırgan, kurbanın tarayıcısında keyfi JavaScript çalıştırabilir (iframe ile teslim)
- **Durum:** Tamamlandı — iframe + post-load hash değişimi + `print()` ile victim-delivery, lab çözüldü
- **Yeni öğrenilenler:** jQuery `$()` selector sink, hashchange/location.hash, iframe post-load-hash trick, `print()` vs `alert` gotcha

---

## Reconnaissance & Vulnerability Analysis

### Source → Sink zinciri
```javascript
$(window).on('hashchange', function() {
    var element = $(location.hash);      // SINK: $() selector, source: location.hash
    element[0].scrollIntoView();
});
```
- **Source:** `location.hash` → URL'in `#...` kısmı (saldırgan kontrollü)
- **Sink:** jQuery **`$()`** selector — normalde element *seçer*, ama HTML gibi string verilince element **yaratır**
- **Olay:** `hashchange` — hash değişince tetiklenir (sayfa "bölüme kaydır" özelliği)

### `$()` selector sink'i
- `$('<img src=x onerror=...>')` → jQuery bunu seçmez, **bir `<img>` elementi oluşturur** → onerror → XSS
- **Versiyon notu:** Eski jQuery HTML string'i her yerde yaratır (bu lab). Yeni jQuery (1.9+) sadece `<` ile BAŞLAYAN string'i HTML sayar → `$(location.hash)` (hep `#` ile başlar) yeni sürümde güvenli; ama `#` zorunlu olmayan tam-kontrol bir source → modern jQuery'de bile exploit

---

## Exploitation

### Kritik problem: hashchange timing
`hashchange`, hash **DEĞİŞİNCE** ateşlenir — sayfa ilk yüklenirken hash zaten oradaysa **ateşlenmez.** Yani `lab/#<img...>`'i direkt yüklemek **çalışmaz** (handler hiç koşmaz). Hash'in **load sonrası değişmesi** gerekiyor.

### Çözüm: iframe ile post-load hash değişimi
```html
<iframe src="https://<lab-id>.web-security-academy.net/#" onload="this.src+='<img src=1 onerror=print()>'"></iframe>
```
Akış:
1. İframe lab'ı **`#`** (boş hash) ile yükler
2. `onload` → `this.src`'ye payload eklenir → URL `#`'ten `#<img...>`'e **değişir** (sadece hash değiştiği için iframe reload olmaz)
3. **hashchange ateşlenir** → `$(location.hash)` = `$('<img src=1 onerror=print()>')` → jQuery img yaratır → `onerror` → `print()`

### Teslimat
Exploit server → **Body**'ye iframe (lab domain'ini düzelt) → Head default → Store → View exploit (test) → **Deliver to victim** → lab çözüldü.

### `alert` değil `print()`
Payload **cross-origin iframe** içinde çalışıyor; Chrome bu bağlamdan `alert`/`confirm`/`prompt`'u **bloklar.** View exploit (Firefox) `alert`'le çalışır ama Deliver (victim = Chrome) solve saymaz → `print()` kullan.

---

## Remediation

- **`$()`'e kullanıcı girdisi verme:** `location.hash`/`location.search` gibi source'ları jQuery selector'ına geçirme.
- **Element seçimi için güvenli yöntem:** `document.getElementById` + girdiyi doğrula, ya da selector'ı sabit tut.
- **jQuery'yi güncel tut** (yeni sürüm `#` ile başlayan hash'i HTML saymaz) — ama tek savunma olmasın.

---

## Gotchas (Dikkat Edilecekler)

- **hashchange ilk yüklemede ateşlenmez** — hash DEĞİŞİNCE tetiklenir. Direkt URL çalışmaz; **iframe ile load sonrası hash değiştir.** Bu lab'ın asıl bükülmesi bu.
- **`$()` HTML yaratır:** jQuery selector'ına HTML gibi string verince element oluşturur (seçmez). Versiyon nüansı: yeni jQuery'de input `<` ile başlamalı; `$(location.hash)` `#` yüzünden yeni sürümde güvenli (bu lab eski sürüm).
- **Victim-delivery + cross-origin iframe → `print()`, `alert` değil.** Chrome cross-origin iframe'den alert/confirm/prompt'u bloklar. Belirti: View exploit çalışır ama Deliver çözüm saymaz. Lab başındaki "victim uses Chrome" uyarısı işareti. (Bu lab o uyarıyı atlayınca yakalandı.)
- **Exploit server neden gerekli:** (1) iframe'i barındıracak bir sayfa lazım (post-load hash değişimini iframe sağlıyor); (2) XSS bir kurbanı hedefler → saldırıyı kurbanın ziyaret edeceği yerde host etmelisin. Kurban tıklamaz — sayfayı açınca iframe otomatik tetikler.
- **`#`'ten sonra tırnak/breakout yok:** `$()` ham HTML alıyor → `<img src=1 onerror=print()>` direkt; attribute context değil.
- **İframe'deki tırnaklar breakout değil:** `onload="this.src+='...'"` → çift tırnak attribute'u, tek tırnak string'i sarar; exploit sayfasının normal HTML/JS sözdizimi.

---

## Zincir Özeti

```
exploit server Body: <iframe src="https://<lab>/#" onload="this.src+='<img src=1 onerror=print()>'">
  → kurban exploit sayfasını açar
  → iframe lab'ı #'li yükler → onload → src += payload → hash # → #<img...> (DEĞİŞİR)
  → hashchange ateşlenir → $(location.hash) = $('<img src=1 onerror=print()>')
  → jQuery img yaratır → onerror → print()
  → (Chrome cross-origin iframe'de alert bloklu olduğu için print)
```
