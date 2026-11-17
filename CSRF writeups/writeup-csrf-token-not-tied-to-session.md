# CSRF Where Token Is Not Tied to User Session — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-05

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** CSRF — token doğrulanıyor ama kullanıcının oturumuna bağlı değil (global token havuzu)
- **Genel Etki:** Saldırgan kendi hesabından geçerli bir token alıp kurbanın isteğinde kullanabilir; e-posta değiştirme CSRF'i gerçekleştirilebilir
- **Durum:** Tamamlandı — saldırganın kendi (taze) token'ı ile exploit teslim edildi, lab çözüldü

---

## Reconnaissance & Enumeration

E-posta değiştirme işlemi POST + CSRF token ile korunuyor:

```
POST /my-account/change-email
email=...&csrf=<token>
```

Token var ve POST'ta doğrulanıyor — yani "token being present" ya da "depends on method" bypass'ları burada çalışmaz. Ama token'ın **kime ait olduğu** kontrol edilmiyor olabilir.

---

## Vulnerability Analysis

- **Zafiyet:** CSRF token kullanıcı oturumuna bağlı değil (not tied to session)
- **Teknik Mantık:** Uygulama, ürettiği token'ları **global bir havuzda** tutuyor ve gelen istekte "bu token havuzda geçerli mi?" diye bakıyor — ama "bu token **bu kullanıcının** oturumuna mı ait?" diye **kontrol etmiyor.**
- **Sonuç:** Saldırgan kendi hesabıyla giriş yapıp **geçerli bir token** elde edebilir, bu token'ı kurbanın isteğine koyabilir. Token havuzda geçerli olduğu için (kime ait olduğu umursanmadığından) sunucu kabul eder.

---

## Exploitation

### 1. Token mekaniğini anlama (kritik)

Bu lab'ı çözmenin anahtarı, token'ın yaşam döngüsünü doğru anlamak:
- Token, **sayfa yüklendiğinde** (GET /my-account) üretilip forma gömülür — submit edince DEĞİL.
- Token **response HTML'inde** `<input type="hidden" name="csrf" value="...">` olarak bulunur — request'te değil.
- Token **tek kullanımlık**: bir kez submit edilince (harcanınca) geçersizleşir.

### 2. Taze (harcanmamış) token alma

1. `GET /my-account` isteğinin **response**'una bakıldı
2. Form içindeki `csrf` hidden input'unun değeri kopyalandı (ör. `bomhhEA9k0h3sYJUzNPtyq9OVUTfu9ro`)
3. Bu token submit edilmedi → bakir/kullanılmamış kaldı

### 3. Exploit

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="<taze-saldirgan-tokeni>">
</form>
<script>document.forms[0].submit();</script>
```

`csrf` değeri = saldırganın kendi oturumundan alınmış **taze** token. Kurban exploit'i açınca, istek kurbanın cookie'si + saldırganın token'ı ile gider. Sunucu token'ı havuzda geçerli bulur (oturum bağını kontrol etmez) → kabul eder → kurbanın e-postası değişir → **lab çözüldü.**

---

## Remediation (Çözüm Önerileri)

- **Token oturuma bağlanmalı:** Her token, onu üreten kullanıcının oturumuyla ilişkilendirilmeli. Doğrulamada "token geçerli mi?" değil, "token **bu oturuma** ait ve geçerli mi?" sorulmalı.
- **Tek kullanımlık + kısa ömür:** Token tek kullanımlık olmalı (zaten öyle görünüyor) ve makul sürede expire olmalı.
- **SameSite + token birlikte:** Defense in depth.

---

## Gotchas (Dikkat Edilecekler)

- **Token "değiştir"e basınca üretilmez — sayfa yüklenince üretilir.** En büyük yanlış anlama buydu. Token almak için formu submit etmene gerek yok (hatta submit edersen harcarsın); sayfayı AÇMAK yeterli.
- **Token request'te değil, response'ta.** `GET /my-account`'un **yanıt** gövdesindeki form HTML'inde. İstek başlıklarında sadece cookie var, token orada aranmaz.
- **Token tek kullanımlık — test ederken harcama.** "View exploit" ya da change-email submit, token'ı tüketir. Exploit'e koyduğun token **bakir** olmalı. Zaten kullanılmış bir token "Invalid csrf token" verir.
- **"Invalid token" ≠ "token yok".** Invalid demek sunucu token'ı gördü ve geçersiz buldu — yani kontrol çalışıyor. Bu lab'da token'ı silmek işe yaramaz (present kontrolü var); geçerli ama kendi token'ını koymak gerekir.
- **Bypass mantığı farklı:** Token'ı silmiyor ya da atlamıyorsun — **geçerli ama sana ait** bir token koyuyorsun. Sunucu oturum bağını kontrol etmediği için kabul ediyor.

---

## Token Bypass Ailesi — Nerede Durur

| Lab | Zafiyet | Çözüm |
|---|---|---|
| No defenses | Token yok | Düz POST form |
| Token being present | Yoksa kontrol atlanır | Token parametresini sil |
| Depends on request method | GET'te kontrol yok | POST→GET, token'ı sil |
| **Not tied to session (bu)** | Kime ait kontrol edilmez | Kendi taze token'ını kullan |

Ortak ders: CSRF token savunmasındaki zafiyetler, token'ın **üç şartından** birinin eksikliğinden doğar — (1) tahmin edilemez, (2) her zaman zorunlu, (3) oturuma bağlı. Bu lab 3. şartı ihlal ediyordu.
