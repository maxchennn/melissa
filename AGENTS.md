# AGENTS.md

## Uyulmasi zorunlu kurallar
- Bu tek bir statik dosya (`index.html`): build adimi, bundler, package.json, linter veya test paketi yok. Acikca istenmedikce bunlardan birini eklemeyin.
- `@google/genai`, sabitlenmemis bir CDN import map'i uzerinden yukleniyor (`https://esm.run/@google/genai`). Buraya dokunursaniz, tam bir versiyon sabitleyin — akan (floating) birakmayin.
- `SYSTEM_PROMPT` (tr/en) icindeki sistem prompt'u, sabit bir "seni kim yarattı" yanitini ve Markdown yasagi kuralini zorunlu kiliyor. Acikca istenmedikce bu davranisi kaldirmayin veya degistirmeyin.
- Gemini API anahtari yalnizca `localStorage`'da (`melissa-key`) saklaniyor ve tamamen istemci tarafinda kullaniliyor. Anahtari resmi Gemini API endpoint'i disinda herhangi bir yere ileten kod eklemeyin.
- Mesaj render islemi, kullanici veya model tarafindan uretilen metin icin her zaman `textContent` kullaniyor, asla `innerHTML` degil. Sanitizasyon eklemeden (orn. DOMPurify) `innerHTML`'e gecmeyin — su anda XSS'i onleyen tek sey bu.

## Bitirmeden once dogrulama
- Otomatik test/lint/build yok. Tarayicida elle dogrulayin: sayfa yukleniyor, kayitli anahtar yokken anahtar modali cikiyor, bir mesaj Gemini'ye gidip geliyor ve render ediliyor, tema/dil degisiklikleri sayfa yenilendiginde kaliciyor.

## Depoya ozgu kurallar
- Tum markup, stiller ve script tek bir `index.html` dosyasinda yasiyor — acikca ayirmasi istenmedikce boyle birakin.
- Tema'lar, `:root` benzeri tanimlari gecersiz kilan `[data-theme="..."]` CSS ozel ozellik bloklariyla yapiliyor; bu deseni kopyalayarak yeni temalar ekleyin, bir CSS framework'u getirerek degil.
- Kullaniciya gorunen dizeler `TEXTS` nesnesinde (`tr`/`en`) yasiyor; yeni kullaniciya gorunen dizeleri satir ici sabit degerler yerine buraya ekleyin.

## Degisiklik guvenlik kurallari
- Kullanicilarin kayitli ayarlariyla geriye donuk uyumluluk icin varsayilan `tr` (Turkce) dilini ve mevcut localStorage anahtarlarini (`melissa-lang`, `melissa-key`, `melissa-theme`) koruyun.
