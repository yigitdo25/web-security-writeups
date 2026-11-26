# Reflected XSS into HTML context with all tags blocked except custom ones — Writeup

**Platform:** PortSwigger Web Security Academy
**Zorluk:** Practitioner
**Tarih:** 2026-11-26

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (arama fonksiyonu, `?search=`)
- **Tespit Edilen Zafiyet:** Reflected XSS — gövde (HTML text) context'ine encode'suz yansıma. Filtre **tüm standart HTML tag'lerini** engelliyor ("Tag is not allowed"), **custom (uydurma) tag'lere** izin veriyor.
- **Bypass:** Custom tag (`<xss>`) + **onfocus** (event) + **tabindex** (focusable yap) + **`id=x` & URL `#x` fragment** (focus'u tetikler).
- **Teslimat:** Exploit server'dan **top-level `location` redirect** → **Deliver to victim** (alert victim'in tarayıcısında çalışmalı; self-trigger çözmez).
- **Durum:** Çözüldü — `<xss id=x onfocus=alert(document.cookie) tabindex=1>#x`
- **Tür:** Reflected XSS (filtre bypass — custom tag + fragment-focus)

---

## Reconnaissance & Vulnerability Analysis

- `?search=test123` → `<h1>0 search results for 'test123'</h1>` = **gövde** context, breakout gerekmez.
- Standart tag (`<img ...>`) → **"Tag is not allowed"**.
- **Başlık izin verileni veriyor:** "all tags blocked except **custom** ones" → brute-force gereksiz; asıl zorluk **custom tag'de event ATEŞLEMEK** = bir **muhakeme** problemi.
- `<xss>` → geçer (gerçek HTML elementi değil → standart-tag kara listesinde yok). Ama **yerleşik olayı yoktur.**

---

## Exploitation

### Çözüm zinciri
- **`onfocus`** — element focus kazanınca ateşlenen event (herhangi bir elemente eklenebilir; bu lab'da filtrelenmemiş).
- **`tabindex=1`** — custom elementi **focus ALABİLİR** yapar (normalde `<xss>` odaklanamaz).
- **`id=x` + URL `#x` fragment** — fragment, `id=x` elementine **gider ve onu FOCUS eder** → onfocus ateşlenir. Kullanıcı etkileşimi gerekmez.

**Zincir:** `#x` fragment → `id=x` elementine git + focus (tabindex focusable yaptı) → `onfocus` → alert.

### Teslimat — KRİTİK NÜANSLAR (asıl ders burada)
Bu lab **self-trigger ile ÇÖZÜLMEZ** — solve koşulu, alert'in **VICTIM'in (bot)** tarayıcısında çalışması (başka kullanıcıya saldırabildiğini kanıtlamak). Exploit server'dan **Deliver to victim** gerekli.

Üç şey denendi, ilk ikisi victim'de başarısız:
- **`autofocus` neden OLMAZ:** autofocus, pencerenin "focused" olmasını ister. Victim bot (headless / arka plan) penceresi focused olmadığı için autofocus `onfocus`'u **tetiklemez.** (Kendi ön-plan tarayıcında çalışır = sadece **POC**, victim-güvenilir değil.)
- **iframe neden OLMAZ:** tarayıcılar **cross-origin iframe** içindeki elementlerin focus'unu **bloklar.**
- **Çalışan çözüm:** **top-level `location` redirect** (lab sekmesi aktif pencere olur) + **`#x` fragment** → focus gerçekleşir, bot'ta da çalışır.

**Exploit server body:**
```html
<script>
location = 'https://<lab-id>.web-security-academy.net/?search=<xss id=x onfocus=alert(document.cookie) tabindex=1>#x';
</script>
```
**Store → Deliver to victim** → üstteki banner "solved".

> **Sözdizimi notu:** string `'` ile açılır, payload bitince (`>` ten sonra, `#x` dahil) `'` ile **kapanır**, sonra `;`. Yani `...tabindex=1>#x';` (unterminated string = en sık bug).

---

## Remediation

- **Kara liste yetersiz:** standart tag'leri engellemek custom tag'leri + attribute'ları geçirir. **Context-aware output encoding** (`<`→`&lt;` vb.).
- Girdiyi HTML sink'e sokma; `textContent` / güvenli template.
- CSP ile satır içi event handler'ları kıs.

---

## Gotchas (Dikkat Edilecekler)

- **Başlık izin verileni veriyor → brute-force gereksiz, reasoning problemi.** (Önceki "most tags blocked" = enumeration; bu = muhakeme. Araç seçimini doğru yap.)
- **Custom tag = uydurma isim** (gerçek HTML elementi değil) → standart-tag kara listesini geçer, **ama yerleşik olayı yok.**
- **Custom tag'de event ateşleme = onfocus + tabindex + fragment-focus.**
- **`autofocus` victim bot'ta güvenilmez** (pencere focus'u ister) → **`#x` fragment** kullan. autofocus yalnızca kendi ön-plan tarayıcında POC verir.
- **iframe cross-origin focus'u bloklar** → **top-level `location` redirect** şart.
- **Self-trigger çözmez** → **Deliver to victim** (alert victim'de çalışmalı).
- **Metodoloji dersi (büyük):** önceki lab'ın makinesini (onresize'ın iframe+onload'u, ya da autofocus) **körü körüne taşıma.** Her problem için sıfırdan sor: *"BU problem neyi gerektiriyor?"* — en basit işleyen şeyi seç. (Bu lab'da teslimat aslında sade bir redirect; iframe/autofocus baggage'dı.)
- **Lab validation'ı finicky:** forum'da bu lab için "cannot be validated / not working" şikayetleri var → doğru payload'la olmuyorsa birkaç kez Deliver dene; sorun sende olmayabilir.

---

## Zincir Özeti

```
exploit server body:
  <script>location = 'https://<lab-id>...net/?search=<xss id=x onfocus=alert(document.cookie) tabindex=1>#x';</script>
  → Store → Deliver to victim
  → victim'in botu top-level olarak lab URL'ini yükler
  → <xss> custom tag filtreden geçer (standart-tag kara listesinde yok)
  → #x fragment → id=x elementine gider ve FOCUS eder (tabindex focusable yaptı)
  → onfocus → alert(document.cookie) VICTIM'in tarayıcısında → solved
```

**Not (öğrenme süreci):** çözüm önce `autofocus` ile denendi; kendi tarayıcıda alert çıktı (payload geçerli = POC) ama lab "solved" olmadı → fark edildi ki victim delivery gerekiyor ve autofocus bot'ta tetiklenmiyor. Doğrulanmış çözüm (#x fragment + top-level redirect) resmi writeup'larla teyit edildi.
