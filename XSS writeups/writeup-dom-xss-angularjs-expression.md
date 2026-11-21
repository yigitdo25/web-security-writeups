# DOM XSS in AngularJS Expression (angle brackets & double quotes HTML-encoded) — Writeup

**Platform:** PortSwigger Web Security Academy
**Tarih:** 2026-11-21

---

## Executive Summary

- **Hedef:** `https://<lab-id>.web-security-academy.net/` (PortSwigger lab)
- **Zorluk:** İleri (Advanced)
- **Tespit Edilen Zafiyet:** AngularJS kullanan sayfada **client-side template injection** — `location.search`'ten gelen girdi, `ng-app` bölgesinde Angular ifadesi olarak değerlendiriliyor. `<>` ve `"` HTML-encode edildiği için klasik XSS kapalı, ama Angular ifadesi (`{{ }}`) + sandbox escape ile JS çalışıyor
- **Genel Etki:** Saldırgan keyfi JavaScript çalıştırabilir
- **Durum:** Tamamlandı — `{{$on.constructor('alert(1)')()}}`, lab çözüldü
- **Yeni sınıf:** İlk AngularJS / template injection XSS'i

---

## Reconnaissance & Vulnerability Analysis

- Site **AngularJS** kullanıyor. **`ng-app` directive'i** Angular'ın işleyeceği bölgeyi işaretler; o bölgedeki `{{ }}` ifadeleri değerlendirilir (çalıştırılır)
- **Source:** `location.search` (search parametresi) → `ng-app` bölgesine yansıyor
- **Savunma:** `<>` ve `"` **HTML-encode** ediliyor → etiket enjeksiyonu (`<img>`) ve attribute kaçışı (`">`) **kapalı**. Klasik XSS yolları yok
- **Ama 3. yol açık:** Girdi Angular bölgesinde olduğu için **Angular ifadesi** enjekte edilebilir — `<>`, `"`, event handler **gerekmez**
- **Tespit testi:** search'e `{{7*7}}` → sayfada **`49`** çıktı → Angular girdiyi **değerlendiriyor** = template injection sink doğrulandı

---

## Exploitation

### `{{ }}` = ifade, HTML değil
Süslü parantez içine `<img>` konmaz. Angular, içindeki **JS-benzeri ifadeyi çalıştırır**, sonucu metin basar. `{{7*7}}`→`49` bunu kanıtladı. Yani `7*7` yerine **JS çalıştıran** bir ifade gerekli.

### Sandbox escape
AngularJS'in bir **sandbox**'ı var — ifadelerin tehlikeli şeylere (ör. `Function`) erişimini kısıtlar, doğrudan `{{alert(1)}}` çalışmaz. Aşmak için `constructor` zinciriyle JavaScript'in **Function constructor**'ına ulaşılır (string'i koda çevirir = eval gibi). (Not: Angular sandbox'ı güvenli yapılamadığı için **1.6'da kaldırıldı.**)

### Payload
```
{{$on.constructor('alert(1)')()}}
```
- `$on.constructor` → **Function constructor**'a ulaşır (sandbox'tan kaçış)
- `('alert(1)')` → gövdesi `alert(1)` olan yeni fonksiyon
- `()` → onu çağırır

### Teslimat
Search parametresine payload'ı koy, URL'i yükle. **Victim gerekmez** (reflected DOM, kendi tarayıcında çözülür) → `alert` yeter, `print` gerekmez.

---

## Remediation

- **Kullanıcı girdisini Angular template'ine koyma** — girdi `ng-app` bölgesinde değerlendirilirse template injection olur.
- Girdiyi Angular-kontrollü bölgenin **dışında** tut, ya da `ng-non-bindable` ile değerlendirmeyi kapat.
- **Output encode** tek başına yetmez (burada `<>`/`"` encode'luydu ama ifade yolu açıktı) — asıl çözüm girdiyi ifade olarak değerlendirtmemek.

---

## Gotchas (Dikkat Edilecekler)

- **`{{ }}` HTML değil, İFADE.** İçine `<img>`/etiket konmaz; Angular ifadeyi çalıştırır. Bu yüzden `<>` encode'lu olması önemli değil — Angular yolu `<>`/event gerektirmez ("execute JS without angle brackets or events").
- **`{{7*7}}` → `49` = template injection tespit testi.** Angular'ın girdiyi değerlendirip değerlendirmediğini anında gösterir. (Sunucu-tarafı SSTI'de eşdeğeri `${7*7}` / `#{7*7}`.) Eski JS/DOM sink'lerinde eşdeğeri yok — onlar marker+context yöntemi.
- **Sandbox escape payload'ı türetilemez** → lookup (PayloadsAllTheThings / HackTricks). Ezberleme; **mantığı** bil: `constructor` → Function constructor → string'i kod olarak çalıştır (eval-gibi) → keyfi JS. Tam zincir (`$on` vs.) versiyona-özel arcana.
- **AngularJS sandbox 1.6'da kaldırıldı** (güvenli yapılamadı) — "1.6+ daha güvenli" bu konuda yanıltıcı; gerçek savunma girdiyi template'e koymamak.
- **Victim yok, `alert` yeter** — cross-origin iframe teslimatı olmadığı için `print()` gerekmedi (önceki jQuery hashchange lab'ından farkı).
- **Sink'i canlı DOM'da gör:** Angular client-side işlediği için view-source değil, Elements/`{{7*7}}` testi. (DevTools: Console=komut, Elements=canlı DOM, Sources=kod.)

---

## Zincir Özeti

```
?search={{$on.constructor('alert(1)')()}}   (source: location.search, ng-app bölgesi)
  → Angular ifadeyi değerlendirir ({{7*7}}→49 ile doğrulandı)
  → $on.constructor = Function constructor (sandbox escape)
  → Function('alert(1)')() → keyfi JS
  → alert
  (<> ve " encode'lu ama ifade yolu gerektirmez)
```
