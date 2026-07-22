# OPTIMIZATIONS.md — index.html ("Melissa AI")

## 1) Optimizasyon Ozeti

Bu, tek dosyalik, tamamen istemci tarafinda calisan bir sohbet arayuzu (vanilla JS + CDN uzerinden importmap ile yuklenen `@google/genai`). Build adimi, bundler, test/lint araci yok — bu yuzden buradaki "optimizasyon" riskinin cogu derleme zamani veya algoritmik karmasiklik degil, calisma zamani maliyeti/bellek buyumesi ve birkac kolay DOM/CSS temizligi.

**En yuksek etkili ilk 3 iyilestirme**
1. Konusma gecmisi sinirsiz buyuyor — her tur, tum sohbet gecmisini (SDK tarafindan yonetilen) yeniden Gemini API'ye gonderiyor; bu yuzden token maliyeti ve gecikme sohbet uzunlugu ile dogrusal-ile-karesel arasinda bir hizda artiyor.
2. `#messages` icinde render edilen DOM node sayisi icin virtualization/sinir yok — uzun oturumlarda sinirsiz DOM elemani birikiyor.
3. `@google/genai` modulu (bir CDN uzerinden) sayfa her yuklendiginde, kullanici anahtar girmeden once bile istekli/eager olarak cekiliyor — bu, ilk boyama/etkilesim suresini biraz geciktiriyor ve kritik yola gereksiz bir ucuncu taraf bagimliligi ekliyor.

**Degisiklik yapilmazsa en buyuk risk:** uzun surelu oturumlarda kullanicilar, gorunurluk olmadan (token sayaci yok, kisaltma/ozetleme stratejisi yok) mesaj basina artan gecikme ve API maliyeti gorecek; ayrica cok uzun konusmalarda DOM bellegi sinirsiz buyuyecek.

---

## 2) Bulgular (Onceliklendirilmis)

### Bulgu 1 — Istek basina sinirsiz sohbet gecmisi buyumesi
- **Kategori:** Maliyet / Ag (Network)
- **Onem:** Orta
- **Etki:** API maliyeti, gecikme, uzun oturumlarda token siniri hatalari
- **Kanit:** `chat = genAI.chats.create(...)` (satir ~1042), ardindan tekrarlanan `chat.sendMessage({ message: text })` (satir ~1071) — SDK'nin `chat` nesnesi gecmisi biriktiriyor ve her cagride yeniden gonderiyor; dosyada hicbir yerde kisaltma, ozetleme veya kayan pencere (sliding window) mantigi yok.
- **Neden verimsiz:** Her ek tur, onceki tum turleri girdi tokeni olarak yeniden gonderiyor. Uzun bir konusma icin bu, oturum boyunca toplam token'da O(n²) demek ve sonunda modelin baglam penceresine takilma riski var.
- **Onerilen cozum:** Gecmisi sinirla (orn. son N turu tut) veya yeni bir `chat` oturumu olusturmadan once eski turleri periyodik olarak yogunlastirilmis bir baglam mesajina ozetle.
- **Riskler/Odunlesimler:** Ozetleme ekstra bir model cagrisi (maliyet) ekler ve onceki baglamdan incelik kaybina yol acabilir.
- **Beklenen etki tahmini:** Uzun oturumlarda maliyette yuksek azalma (yaklasik 20 turdan sonra potansiyel olarak %50+ token tasarrufu); kisa oturumlarda etkisi yok.
- **Kaldirma Guvenligi:** Dogrulama Gerekli (konusma hafizasinda davranis degisikligi)
- **Yeniden Kullanim Kapsami:** yerel dosya (`initAI`/`sendMsg` cifti)

### Bulgu 2 — `#messages` icinde sinirsiz DOM buyumesi
- **Kategori:** Bellek / Frontend
- **Onem:** Dusuk
- **Etki:** Cok uzun oturumlarda bellek, scroll/render performansi
- **Kanit:** `addMsg()` (satir ~1018) her zaman `messages.appendChild(div)` yapiyor; eski mesaj node'larini kaldiran hicbir sey yok.
- **Neden verimsiz:** Her mesaj gercek bir DOM node'u — tipik sohbet oturumlari (onlarca mesaj) icin bu bir sorun degil, ama patolojik oturumlar (yuzlerce/binlerce mesaj) icin bir tavan yok.
- **Onerilen cozum:** Yalnizca uzun oturumlar gercek bir kullanim durumuysa yapmaya deger — orn. DOM node sayisini sinirla ve eski mesajlari bir JS dizisinde tut, ya da listeyi virtualize et.
- **Riskler/Odunlesimler:** Kanitlanmis olmaktan cok "muhtemel" bir sorun icin eklenen karmasiklik.
- **Beklenen etki tahmini:** Dusuk/muhtemel — "muhtemel" darbogaz olarak isaretlendi; buraya yatirim yapmadan once gercek oturum uzunluklarini olcun.
- **Kaldirma Guvenligi:** Muhtemelen Guvenli
- **Yeniden Kullanim Kapsami:** yerel dosya

### Bulgu 3 — `@google/genai` modulu kosulsuz olarak sayfa yuklenirken cekiliyor
- **Kategori:** Ag / Frontend
- **Onem:** Dusuk
- **Etki:** Etkilesime hazir olma suresi, kullanicinin henuz anahtari yokken gereksiz istek
- **Kanit:** Statik `importmap` (satir ~908) + ust seviye `import { GoogleGenAI } from '@google/genai'` (satir ~913), `apiKey` olsun ya da olmasin sayfa yuklenir yuklenmez calisiyor.
- **Neden verimsiz:** Kayitli anahtari olmayan kullanicilar, henuz hicbir sey yapmadan once SDK'yi cekme ve ayristirma maliyetini odemis oluyor.
- **Onerilen cozum:** `initAI()` icinde dinamik `import('@google/genai')` kullan, yalnizca bir anahtar mevcut oldugunda cagir (lazy-load).
- **Riskler/Odunlesimler:** Kucuk ek asenkron karmasiklik; SDK kucuk/CDN tarafindan onbelleklenmis olmadikca fayda ihmal edilebilir.
- **Beklenen etki tahmini:** Dusuk (SDK muhtemelen kucuk/onbelleklenmis), ama "bedava" bir duzeltme.
- **Kaldirma Guvenligi:** Guvenli
- **Yeniden Kullanim Kapsami:** yerel dosya

### Bulgu 4 — Tekrarlanan/kopyalanmis tema degisken bloklari
- **Kategori:** Frontend / Olu Kod & Yeniden Kullanim
- **Onem:** Dusuk
- **Etki:** Bakim yapilabilirlik, CSS boyutu
- **Kanit:** Sekiz neredeyse ozdes `[data-theme="..."]` bloku (satir ~27–130), her biri ayni 12 CSS ozel ozelligini yeniden tanimliyor.
- **Neden verimsiz:** Calisma zamani performans sorunu degil (CSS ozel ozellikleri ucuz), ama dosya boyutunu artiran ve sapma (drift) riski tasiyan kopyalanmis bir yapi — bir ozellik adini degistirmek 8 yeri duzenlemek demek.
- **Onerilen cozum:** Performans kritik bir degisiklik degil; bu dosyaya baska nedenlerle dokunuyorsaniz, tema bloklarini bir JS/build-time veri tablosundan uretmeyi dusunun. Su anda build adimi olmadigindan bu opsiyonel.
- **Riskler/Odunlesimler:** Su anda var olmayan bir arac zinciri (build adimi) getirmeyi gerektirir — muhtemelen tek bir statik dosya icin buna deger degil.
- **Beklenen etki tahmini:** Performansta ihmal edilebilir; bakim yapilabilirlikte orta duzeyde kazanc.
- **Kaldirma Guvenligi:** Gecerli degil (kaldirma degil, yeniden kullanim firsati)
- **Yeniden Kullanim Kapsami:** yerel dosya (stil bloku)

### Bulgu 5 — Baslatma yollarinda dagitilmis/tekrarlanan `localStorage` okumalari
- **Kategori:** Kod Yeniden Kullanimi
- **Onem:** Dusuk
- **Etki:** Bakim yapilabilirlik
- **Kanit:** `localStorage.getItem('melissa-lang')`/`'melissa-key'` hem en ust seviye durum baslatmasinda (satir ~971–972) hem de `loadSettings()` icinde (satir ~1090–1091) tekrar okunuyor.
- **Neden verimsiz:** Ayni anahtarlar iki farkli kod yolundan iki kez okunuyor; birinin varsayilan degeri digerinden saparsa durum senkronizasyonu bozulabilir (orn. ust seviyede varsayilan `'tr'` vs `loadSettings` icinde varsayilan `'tr'` — su anda tutarli ama kirilgan).
- **Onerilen cozum:** Baslangicta tum kalici anahtarlari bir kez okuyan ve tek okuyucu olan bir `loadState()` fonksiyonu.
- **Riskler/Odunlesimler:** Onemli bir risk yok.
- **Beklenen etki tahmini:** Dusuk performans etkisi, orta duzeyde dogruluk/bakim yapilabilirlik kazanci.
- **Kaldirma Guvenligi:** Guvenli
- **Yeniden Kullanim Kapsami:** yerel dosya

---

## 3) Hizli Kazanimlar (Once Bunlar)
- `@google/genai`'yi yalnizca bir anahtar mevcut oldugunda veya girildiginde lazy-load et (Bulgu 3).
- Tekrarlanan `localStorage` okuma yollarini tek bir baslatma fonksiyonunda birlestir (Bulgu 5).

## 4) Daha Derin Optimizasyonlar (Sonra Bunlar)
- Uzun sohbet oturumlari icin bir gecmis sinirlama/ozetleme stratejisi ekle (Bulgu 1) — en yuksek getiri ama urun karari gerektirir (kullanici sinirsiz baglam mi yoksa kayan pencere mi istiyor?).
- Uzun oturumlarin yaygin oldugu ortaya cikarsa (varsayim degil, kullanim verisiyle dogrulayin), render edilen mesajlari sinirlamayi/virtualize etmeyi dusunun (Bulgu 2).

## 5) Dogrulama Plani
- **Benchmarklar:** Tarayici Performance panelini kullanarak SDK'yi lazy-load etmeden once/sonra (Bulgu 3) sayfa yukleme suresini elle olcun — Time-to-Interactive'i karsilastirin.
- **Profilleme stratejisi:** Bulgu 2'yi gercek bir darbogaz olarak dogrulamadan/reddetmeden once, 100+ mesajlik sentetik bir oturumda bellek buyumesini kaydetmek icin Chrome DevTools Memory panelini kullanin.
- **Once/sonra karsilastirilacak metrikler:** Bulgu 1 icin gecmis sinirlamasi eklendikten sonra tasarrufu nicelendirmek amaciyla, uzun bir sentetik konusma boyunca istek basina girdi token sayisi (API yanit kullanim metadata'sindan, eger acikta ise).
- **Test senaryolari:**
  - 1 mesaj gonder → degisiklik oncesi/sonrasi ayni davranisi dogrula.
  - 30+ mesaj gonder → gecmis sinirlamasinin kullanicinin bekledigi konusma baglamini bozmadigini dogrula.
  - Kayitli anahtar olmadan sayfayi yukle → (Bulgu 3 duzeltmesi uygulanirsa) SDK'nin anahtar girisine kadar cekilmedigini dogrula.

## 6) Optimize Edilmis Kod / Yama

**Bulgu 3 duzeltmesi — lazy import** (statik import'u `initAI` icinde dinamik import ile degistir):
```js
// Ust seviye statik import'u kaldirin; <script type="importmap"> oldugu gibi kalabilir.
let GoogleGenAI;

async function initAI() {
    if (!apiKey) return false;
    try {
        if (!GoogleGenAI) {
            ({ GoogleGenAI } = await import('@google/genai'));
        }
        genAI = new GoogleGenAI({ apiKey });
        chat = genAI.chats.create({
            model: DEFAULT_MODEL,
            config: { systemInstruction: SYSTEM_PROMPT[currentLang] }
        });
        return true;
    } catch (e) {
        console.error(e);
        return false;
    }
}
```
*Ne degisti:* SDK modulu artik yalnizca `initAI()` gercekten calistiginda (yani bir anahtar mevcut oldugunda) ilk kez cekiliyor, sayfa yuklenirken buna takilmak yerine.

**Bulgu 5 duzeltmesi — tek durum yukleyici:**
```js
function loadState() {
    return {
        lang: localStorage.getItem('melissa-lang') || 'tr',
        key: localStorage.getItem('melissa-key') || '',
        theme: localStorage.getItem('melissa-theme') || 'default'
    };
}
const state = loadState();
let currentLang = state.lang;
let apiKey = state.key;
```
*Ne degisti:* Artik tek bir fonksiyon kalici durum icin tek gercek kaynak, `loadSettings()` icindeki tekrarlanan okumalari kaldiriyor.

**Bulgu 1**, hazir bir yama degil, bir tasarim kararidir (gecmis pencereleme/ozetleme) — yaklasim Bulgu 1'in duzeltmesinde sozde kodla isaretlendi; yalnizca uzun oturumlarin yaygin oldugu dogrulanirsa uygulayin (Dogrulama Plani'na bakin).
