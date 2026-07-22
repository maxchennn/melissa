### GUVENLIK DENETIMI: index.html — "Melissa AI" istemci tarafi Gemini sohbet uygulamasi

**Risk Degerlendirmesi:** Orta

#### **Bulgular:**

* **Sabitlenmemis ucuncu taraf CDN bagimliligi, sayfa/DOM'a tam guven ile** (Onem: Yuksek)
  * **Konum:** Satir 908–913 — `<script type="importmap">{ "imports": { "@google/genai": "https://esm.run/@google/genai" } }` ve `import { GoogleGenAI } from '@google/genai'`
  * **Istismar:** Uygulama, `esm.run`'dan versiyon sabitlemesi olmadan (belirteçte `@x.y.z` yok) ve Subresource Integrity (SRI) hash'i olmadan calisan JS iceri aliyor. `esm.run`, istek anindaki en guncel/sabitlenmemis paket versiyonuna cozulur. O CDN veya upstream npm paketi bir sekilde ele gecirilirse (tedarik zinciri saldirisi), enjekte edilen kod sayfanin DOM'una, `localStorage`'ina (kullanicinin API anahtari dahil) tam erisimle calisir ve veriyi sessizce disariya sizdirabilir veya sohbet arayuzunu yeniden yazabilir. Bu, dosyadaki en yuksek etkili risktir cunku uygulamanin kendi kodu bu soruna hic dokunmuyor — tamamen Claude'un/gelistiricinin kontrol etmedigi bir URL'ye devredilmis durumda.
  * **Cozum:** Import belirtecinde tam bir versiyon sabitleyin (orn. `https://esm.run/@google/genai@<versiyon>`) ve SRI'yi destekleyen bir URL/CDN tercih edin (`integrity="sha384-..."`) ya da paketi yerelde barindirip ayni origin'den servis edin.

* **API anahtari, telafi edici kontroller olmadan `localStorage`'da saklaniyor** (Onem: Orta)
  * **Konum:** Satir 972 (`apiKey = localStorage.getItem('melissa-key') || ''`), satir 1141 (`localStorage.setItem('melissa-key', key)`)
  * **Istismar:** `localStorage`, sayfanin origin'inde calisan herhangi bir JS tarafindan okunabilir. Yukaridaki sabitlenmemis CDN import'u ile birlikte, gelecekte olusabilecek herhangi bir XSS veya tedarik zinciri ele gecirmesi, tek satirlik bir anahtar hirsizligina yol acar (`localStorage.getItem('melissa-key')`) ve kullanicinin faturalandirmaya bagli Gemini API anahtari disariya sizdirilir. Su anda bu dosyada gorunur bir XSS vektoru yok (kullanici tarafindan kontrol edilen icerik icin mesaj render islemi `innerHTML` degil `textContent` kullaniyor), ancak bu saklama secimi, ileride bir tane eklenirse son savunma hattini ortadan kaldiriyor.
  * **Cozum:** Bu istemci tarafinda kalmak zorundaysa, en azindan anahtarin etki alanini sinirlayin (kullanicilari bu arac icin ozel, kisitli/dusuk kotali bir anahtar olusturmaya tesvik edin) ve bu riski arayuzde belgeleyin. Gercek anahtari sunucu tarafinda tutan ve tarayiciya kisa omurlu, kapsami sinirli token'lar veren hafif bir backend proxy'yi dusunun; boylece ham saglayici anahtari istemci tarafinda saklanmaz.

* **Content-Security-Policy yok** (Onem: Orta)
  * **Konum:** `<head>` (satir 4–7) — `<meta http-equiv="Content-Security-Policy">` mevcut degil
  * **Istismar:** CSP olmadan, ileride herhangi bir enjeksiyon vektoru eklenirse (orn. AI ciktisini `textContent` yerine HTML olarak render eden gelecekteki bir ozellik) hicbir derinlemesine savunma yok. Bir CSP, CDN ya da bir XSS ele gecirilmis olsa bile disariya sizdirma girisimlerini engellemis/kisitlamis olurdu.
  * **Cozum:** `script-src`'i `'self'` artı belirlenen sabit CDN origin'i ile sinirlayan bir CSP meta etiketi ekleyin ve `unsafe-inline`/`unsafe-eval`'a izin vermeyin.

* **AI ciktisi sanitizasyon siniri zorlanmadan render ediliyor (su an guvenli, ama kirilgan)** (Onem: Dusuk)
  * **Konum:** `addMsg()`, satir 1025 — `div.textContent = text;`
  * **Istismar:** Su an guvenli: hem asistan hem kullanici metni `textContent` ile eklenir, bu HTML/script calistirmaz. Ancak sistem prompt'u (satir 918–927) AI ciktisinda Markdown/link'i acikca yasakliyor — eger ileride bu, (Markdown benzeri formatlamayi render etmek icin) `innerHTML`'e cevrilirse, HTML/script etiketleri iceren herhangi bir model ciktisi, render edilmeden once baska bir sekilde sanitize edilmedigi icin, bir XSS vektoru haline gelir.
  * **Cozum:** AI ciktisinin HTML olarak render edilmesi eklenirse, eklemeden once bir sanitizer'dan (orn. DOMPurify) gecirin — ham model ciktisini asla `innerHTML` ile eklemeyin.

* **Istemci kaynagida acikta duran sabit kodlanmis atif/sistem promptu** (Onem: Dusuk)
  * **Konum:** Satir 918–936 (`SYSTEM_PROMPT` nesnesi)
  * **Istismar:** Geleneksel bir zafiyet degil, ama tum sistem prompt'u — "seni kim yarattı" zorunlu yaniti ve formatlama kurallari dahil — sayfa kaynaginda acik metin olarak gonderiliyor ve view-source ya da sohbetin icinden yapilan prompt injection yoluyla herkes tarafindan kolayca goruntulenebilir/cikarilabilir/gecersiz kilinabilir (sistem prompt'u, kullanici tarafindan saglanan prompt injection'a karsi kesin bir garanti degildir). Bunu, sertlestirilmis bir kural yerine "bilinen ve beklenen" bir istemci tarafi guven siniri olarak ele alin.
  * **Cozum:** Bu, herkese acik olarak kabul edilirse kod duzeltmesi gerekmiyor — sadece sistem prompt'una guvenlik acisindan hassas hicbir sey icin guvenmeyin (su anda burada bu sekilde kullanilmiyor).

#### **Gozlemler:**
* API anahtar girisi `type="password"` kullaniyor (satir 900) — iyi, alandaki ham degerin omuz sorfu/basit ekran yakalama ile gorulmesini onluyor.
* `keyModal` yalnizca arka plan tiklamasi veya acik kaydetme ile kapaniyor (satir 1146–1148) — istenmeyen kapanma yollari yoluyla kazara anahtar acikta kalma yok.
* Bu dosyada hicbir sunucu tarafi bileseni yok — "yetkisiz erisim kontrolu" / "enjeksiyon" kategorilerinin tumu (SQLi, IDOR, yetkilendirme kontrolleri) burada bir backend, veritabani veya kimlik dogrulama katmani olmadigi icin gecerli degil.
* `localStorage`'da saklanan ayarlar (`theme`, `lang`) hassas degil; burada bir endise yok.
