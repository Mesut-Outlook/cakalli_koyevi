# CLAUDE.md — Çakallı Köyevi ve Bahçe Projesi Bütçe Paneli

> Bu dosya her oturumda otomatik yüklenir. Kodu yeniden okumaya gerek kalmadan direkt çalışmaya başlanabilir.

## Memory Dosyaları

Kalıcı proje notları (session'lar arası korunur):

```
~/.claude/projects/-Users-mesutozdemir--PROJELER-CAKALLI-PROJESI/memory/
├── MEMORY.md                    ← index
├── project_handoff_state.md     ← son durum, açık işler, commit geçmişi
├── project_architecture.md      ← veri modeli, Firebase yapısı, satır referansları
└── project_business_rules.md    ← kullanıcılar, dengeleme, taksit, form kuralları
```

**Her oturumun başında** `project_handoff_state.md` okunmalı.

## Proje Özeti

Tek sayfalık bütçe takip uygulaması. Mesut ve Egemen'in ortak arazi/yapı projesinin harcamalarını, nakit transferlerini ve kredi kartı taksitlerini Firebase ile anlık senkronize eder.

**URL (canlı)**: https://mesut-outlook.github.io/cakalli_koyevi/  
**Local**: http://localhost:8000 (herhangi bir HTTP sunucu ile)  
**Deploy**: `git push origin main` → GitHub Pages ~1-2 dk

## Stack

- **Tek dosya**: `index.html` (~1040 satır) — HTML + CSS + JS hepsi içinde
- **Tailwind**: CDN üzerinden (build yok)
- **Firebase**: Firestore compat SDK v10.12.0, proje: `cakalli-koyevi`
- **Auth**: Basit kullanıcı adı/şifre — `localStorage` ile session
- **Hosting**: GitHub Pages, `main` branch

## Kullanıcılar (Login)

| Kullanıcı | Şifre |
|-----------|-------|
| mesut     | 2025  |
| egemen    | 2025  |
| halil     | 2025  |

Kod: satır ~764 (`const users = { ... }`)

## Firebase Yapısı

- `app/transactions` → `{ items: [...] }` (tüm işlemler)
- `app/categories` → `{ items: [...] }` (kategori listesi)

İlk açılışta `defaultData` ile seed yapılır (satır ~282).

## Veri Modeli — Transaction Objesi

```js
{
  // Ortak alanlar
  id: Date.now(),           // unique ID (epoch ms)
  date: "2026-05-14",       // ISO tarih (string)
  time: "14:32",            // HH:MM (sadece yeni kayıtlarda)
  type: "expense" | "transfer",
  user: "Mesut" | "Egemen" | "Halil",
  amount: 15000,            // TOPLAM tutar (taksitli ise full)
  desc: "AÇIKLAMA",
  category: "Fidan/Ağaç",  // expense için; transfer → "Nakit Ödeme"
  createdAt: 1234567890123,

  // Expense alanları
  source: "personal" | "vault" | "credit",
  installments: 1,          // credit ise: 1=tek çekim, 2-60=taksit sayısı

  // Transfer alanları (sadece type='transfer' ise)
  paymentType: "cash" | "credit_payment",  // KK taksit ödemesi mi
  recipient: "Mesut" | "Egemen" | "Halil",  // alıcı
  relatedCreditTxId: 12345,  // credit_payment ise hangi KK işlemi
  installmentNumber: 3       // credit_payment ise kaçıncı taksit
}
```

**Önemli**: KK işlemlerinde `tx.amount` TOPLAM tutardır. Aylık taksit = `amount / installments`.

**source değerleri:**
- `personal` → Kendi cebinden
- `vault` → Ortak kasadan
- `credit` → Kredi kartından (badge: turuncu "KK · N taksit")

**Eski seed kayıtlar** (`createdAt: 0-11`): `time` ve `installments` alanı yok — kod bunu tolere eder.

## Badge Renkleri (İşlem Listesi)

| Alan | Değer | Renk |
|------|-------|------|
| user | Egemen | `bg-indigo-100 text-indigo-700` (mor-mavi) |
| user | Mesut | `bg-purple-100 text-purple-700` (mor) |
| user | Halil | `bg-teal-100 text-teal-700` (yeşil-mavi) |
| source | vault | `bg-blue-600 text-white` "KASA" |
| source | credit | `bg-amber-500 text-white` "KK · N taksit" |

## Kritik Satırlar (index.html)

| Satır | İçerik |
|-------|--------|
| ~40 | Login ekranı HTML |
| ~80 | Ana uygulama wrapper |
| ~135 | İşlem geçmişi tablosu (desktop) |
| ~153 | Kenar panel (harcama dağılımı + katkı + taksit takvimi) |
| ~184 | Harcama ekleme/düzenleme modal'ı |
| ~222 | Taksit seçeneği grubu (installment-group) |
| ~269 | Firebase init |
| ~282 | defaultData (seed verisi) |
| ~297 | setSyncStatus |
| ~313 | saveData() — Firestore'a yaz |
| ~333 | initApp() — realtime listener |
| ~458 | editTransaction / deleteTransaction |
| ~475 | handleFormSubmit — form kaydetme (taksit okuma dahil) |
| ~540 | updateMonthFilter — ay filtresi |
| ~565 | applyFilters — arama + ay filtresi |
| ~577 | updateUI — tüm UI'ı güncelle |
| ~584 | renderTransactions — tablo + mobil kart render |
| ~651 | renderStats — özet kartlar + partner stats + kategori |
| ~764 | users const (login) |
| ~800 | showApp — login sonrası uygulama başlatma |
| ~828 | resetInstallmentUI |
| ~839 | handleSourceChange — kaynak değişince taksit göster/gizle |
| ~850 | setInstallmentType — tek çekim / taksitli toggle |
| ~866 | renderInstallmentCalendar — taksit takvimi paneli |
| ~963 | toggleInstMonth — taksit ayını aç/kapat |
| ~977 | openModal — form aç (yeni veya düzenleme) |
| ~1036 | closeModal |

## Sıralama Mantığı

```js
transactions.sort(function(a, b) {
  var diff = new Date(b.date) - new Date(a.date);
  return diff !== 0 ? diff : ((b.createdAt || 0) - (a.createdAt || 0));
});
```
Aynı tarihte birden fazla kayıt varsa `createdAt` ile sıralanır (son girilen üstte).

## Dengeleme (Settlement) Mantığı — 3-yönlü, M-E 50/50

- **TÜM** harcamalar (vault hariç, personal+credit dahil) → M ve E arası 50/50 bölünür
- Halil ödese bile paylaşım M-E olur: M ve E her biri Halil'e ödediği tutarın yarısını borçludur
- `vault` harcamaları → kasa bakiyesinden düşülür, dengelemeye dahil DEĞİL
- `transfer` → gönderenin net katkısı artar, alıcının azalır

**Algoritma** (minimum transfer): Creditors ve debtors sıralı listede eşleştirilir, en büyük borçlu en büyük alacaklıya min(borç,alacak) öder. Çoklu mesaj UI'da satır satır gösterilir.

**Eski transfer verisi** için `recipient` yoksa: M↔E inference (M→E, E→M).

## Taksit Takvimi Paneli

- Sadece `source === 'credit'` olan `expense` işlemlerinden üretilir
- Her taksit `tx.date`'in ayından başlayarak N ay ileriye yayılır
- Ay sütunları: geçmiş (soluk, üstü çizili) / bu ay (amber vurgu) / gelecek
- Tıklanınca alt kalemler açılır (toggleInstMonth)

## Responsive Tasarım

- **Desktop** (≥768px): tablo görünümü, `.desktop-table`
- **Mobil** (<768px): kart görünümü, `.mobile-cards`
- CSS `@media` ile toggle edilir (display:none)

## Kategori Yönetimi

- Modal ile ekle/sil/düzenle
- Firestore'a ayrı olarak kaydedilir (`app/categories`)
- Varsayılan kategoriler: satır ~274

## Son Değişiklikler (2026-05-14)

### Bugün:
- 3-yönlü dengeleme algoritması (minimum transfer)
- KK Taksit Ödemesi: transfer modal'da yeni mod
- Ödenmiş taksit takibi (✓ yeşil tik, üstü çizili)
- Taksit sayısı serbest input (2-60), select değil
- Transfer modal'a "Kime?" alanı eklendi
- Yedek İndir / Geri Yükle JSON butonları (header)
- Settlement card çoklu mesaj destekliyor (`<div>` + multiple lines)

### Önceki:
- `installments` alanı eklendi (kredi kartı taksit)
- `source: 'credit'` ödeme kaynağı eklendi
- Halil kullanıcısı eklendi (login + form + teal badge)
- Taksit Takvimi paneli eklendi (kenar panel)
- İşlem tarih sütununda saat altında ayrı satırda gösteriliyor
- Kayıt sıralaması: son girilen en üstte
