# CSRF Vulnerability with No Defenses — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-04

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Kolay (Apprentice)
- **Tespit Edilen Zafiyet:** CSRF (Cross-Site Request Forgery) — e-posta değiştirme işleminde hiçbir savunma yok
- **Genel Etki:** Saldırgan, kurbanı kandırarak kurbanın hesap e-postasını değiştirebilir (hesap ele geçirmeye zemin)
- **Durum:** Tamamlandı — exploit server üzerinden PoC teslim edildi, kurbanın e-postası değişti, lab çözüldü

---

## Reconnaissance & Enumeration

Uygulamada "My Account" sayfasında bir **e-posta değiştirme** fonksiyonu var. İstek Burp ile yakalandı:

```
POST /my-account/change-email HTTP/1.1
Host: <lab-id>.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

İncelenen 4 kritik bilgi:
- **Metod:** POST
- **Endpoint:** `/my-account/change-email`
- **Parametre:** `email` (tek parametre)
- **CSRF token:** **YOK** — body'de sadece `email` var

Token olmaması "no defenses" durumunu doğrular: istek tamamen tahmin edilebilir, saldırgan birebir taklit edebilir.

---

## Vulnerability Analysis

- **Zafiyet:** CSRF (e-posta değiştirme işlemi)
- **Teknik Mantık:** İşlem şu üç CSRF şartını da sağlıyor:
  1. **Anlamlı eylem:** E-posta değiştirme (hesap ele geçirmeye götürür — şifre sıfırlama bu e-postaya gider)
  2. **Cookie tabanlı oturum:** İstek, tarayıcının otomatik eklediği `session` cookie'si ile yetkilendiriliyor
  3. **Tahmin edilemez parametre yok:** CSRF token olmadığı için saldırgan isteği önceden kurabilir
- **Sonuç:** Saldırgan, kurbanın tarayıcısından kendi oturumuyla bu isteği gönderttirebilir.

---

## Exploitation

### 1. PoC HTML (exploit sayfası)

Kurban açar açmaz otomatik olarak hedefe POST atan form:

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

Mantık: form hedef endpoint'e POST atar, `<script>` sayfayı açan herkeste formu otomatik gönderir. Kurbanın tarayıcısı `session` cookie'sini otomatik eklediği için istek kurbanın adına yetkili olarak çalışır.

### 2. Exploit server'a yükleme

PortSwigger'ın sağladığı exploit server kullanıldı (Burp Pro gerekmez):
- **File:** `/exploit` (içeriğin yayınlanacağı yol)
- **Head:** `HTTP/1.1 200 OK` + `Content-Type: text/html; charset=utf-8`
- **Body:** yukarıdaki PoC HTML
- **Store** ile kaydedildi

### 3. Test ve teslimat

- **View exploit** → kendi hesapta test (kendi e-posta değişti → PoC çalışıyor)
- **Deliver exploit to victim** → kurban bot'u (Chrome) `/exploit`'e erişti, kendi oturumuyla POST attı, kurbanın e-postası değişti → **lab çözüldü**

---

## Remediation (Çözüm Önerileri)

- **Anti-CSRF token:** Her state-değiştiren işlemde, tahmin edilemez, isteğe manuel eklenen bir token kullanılmalı. Saldırgan token'ı bilemeyeceği için isteği kuramaz.
- **SameSite cookie:** `session` cookie'si `SameSite=Lax` veya `Strict` olmalı. CSRF cross-site bir saldırı olduğu için, cookie cross-site isteklerde gitmeyince saldırı çöker.
- **Origin/Referer doğrulama:** Sunucu, state-değiştiren isteklerde isteğin kaynağını kontrol etmeli.
- **Hassas işlemleri GET ile yapma:** POST + token + SameSite birlikte (defense in depth).

---

## Gotchas (Dikkat Edilecekler)

- **Burp Pro GEREKMEZ.** "Generate CSRF PoC" Pro özelliği ama PoC HTML elle yazılabilir. Exploit server lab'ın ücretsiz parçası.
- **Form action'ı eksiksiz olmalı** — en çok zaman kaybettiren nokta buydu:
  - `https://` şeması şart (yoksa geçersiz adres)
  - Endpoint yolu (`/my-account/change-email`) şart (yoksa POST köke `/` gider → "Not Found")
  - Domain ile yol arasında `/` şart (`.net/my-account`, `.netmy-account` değil)
- **Exploit server "File" = yol, dosya yükleme değil.** İçeriğin hangi URL yolunda (`/exploit`) sunulacağını belirtir. Boş/yanlışsa "Resource not found".
- **Store şart.** Deliver, Store'daki sürümü gönderir. View çalışsa bile Store edilmemişse kurbana eski sürüm gider. Access log'da `POST /` görülmesi, kayıtlı sürümün hâlâ köke attığını gösterir.
- **Test değeri ≠ saldırı değeri.** Test e-postası ile saldırı e-postası aynı olursa, değişimi gözlemleyemezsin (değer zaten ayarlıysa "değişti" görünmez). Farklı değerler kullan (test: `test@test.com`, saldırı: `attacker@evil.com`).
- **Üç ayrı kimlik var, karıştırma:** Sen (wiener, analiz için), kurban (bot, lab'ı çözen), ve bu lab'da admin YOK. "View exploit" senin e-postanı, "Deliver to victim" kurbanınkini değiştirir. Lab, **kurbanın** e-postası değişince çözülür.
- **Tarayıcı farkı (SameSite):** Kurban Chrome. Firefox ve Chrome cookie'leri cross-site isteklerde farklı gönderir. Kendi testini Burp'ün tarayıcısı/Chrome ile yapmak, kurban davranışını yansıtması açısından önerilir. (Bu "no defenses" lab'ında kritik değildi ama ileri CSRF/SameSite lab'larında olacak.)
- **Access log saati UTC olabilir** — yerel saatinle uyuşmaması "kayıt yok" anlamına gelmez.

---

## CSRF vs IDOR — Kavramsal Not

Bu lab, daha önce çözülen "User ID/role controlled by request parameter" lab'larıyla **karıştırılmamalı**:

| | IDOR / Access Control | CSRF |
|---|---|---|
| Kimlik nasıl belirlenir | Saldırganın gönderdiği parametre (`id=administrator`) | Kurbanın otomatik giden cookie'si |
| Saldırgan kimliği seçer mi | Evet (`id`'yi değiştirir) | Hayır (kurban kim ise o) |
| CSRF'te `id=administrator` yapılır mı | — | **Hayır**, gerek yok |

CSRF'te saldırgan bir kullanıcıyı taklit etmez; kurbanın kendi oturumunu "ödünç alır".
