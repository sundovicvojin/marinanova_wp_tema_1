# MarinaNova tema: PLAN (Faza 0)

Status: **čeka odobrenje**. Nijedan fajl teme još nije napisan.
Datum: 2026-11-12. Napisano za: vlasnika sajta (Vojin), na srpskom; kod i dev dokumentacija kasnije na engleskom.

---

## 0. Šta sam pročitao i šta je stvarno u folderima

| Ulaz | Nalaz |
|---|---|
| `tema1/SKILL.md` | Postoji samo SKILL.md. **Nema `scripts/search.py`, `references/` ni CSV podataka.** Našao sam još jednu kopiju skila u `~/.claude/skills/synced/...`, ali i ona sadrži samo SKILL.md. Zato `--design-system`, `--persist` i `--domain` pretrage **nisam pokretao** i ništa nisam izmislio. Koristim ugrađenu tabelu prioriteta iz skila (pristupačnost, dodir, performanse, stil, responsive, tipografija/boja, animacija, forme, navigacija). Dugme-dials (variance 6, motion 5, density 3) prevodim ručno: umereno asimetričan editorijal, suptilan pokret, prozračan razmak. |
| `proizvodi.csv` | 86 proizvoda (79 variable, 7 simple) + 219 varijacija. |
| `slike.zip` | 4.988 fajlova, 612 originala (482 webp, 109 jpg, 19 avif, 2 png, 7 svg). Uzorak originala: 2000×2000, 1000×1000, 900×954 px. **Veličine nisu ujednačene**, što utiče na 4:5 okvir. Nisam ništa raspakovao u projekat. |
| Brend i SEO | Pročitano: logo SVG, nacrt brend vodiča, SEO polazno stanje, pitanja za vlasnika, README prototipa. |
| Alati na ovom računaru | Node 24, npm, git, `cwebp`. **Nema PHP-a, Docker-a, Composer-a, WP-CLI-ja, Lighthouse-a, `avifenc`.** Ovo menja plan testiranja (vidi §6). |

### Nalazi u podacima koji utiču na dizajn

1. **Kolona `Brands` je prazna na svih 86 proizvoda.** Zato brend nigde ne može da se prikaže niti da ode u schemu dok se ne dodeli WooCommerce brend (taksonomija `product_brand`). To je i uzrok "no brand in Product schema". Tema ne piše schemu, ali će pokazati brend kad postoji. U demo uvozu ću ga dodeliti skriptom (Adidas). Na živom sajtu to vlasnik radi jednom (bulk edit ili CSV).
2. **Specifikacije NISU WooCommerce atributi, nego post meta.** Ključevi: `model`, `gornji_deo`, `srednji_djon`, `spoljasnji_djon`, `stil`, `zatvaranje`, `sifra_modela` (72 obuće) i `tip_proizvoda`, `materijal` (14 odeće/opreme). Spec tabela će zato čitati ova meta polja preko mape u kodu (ključ → srpska oznaka), plus vidljive WooCommerce atribute koji nisu varijacije. Prazna polja se ne prikazuju. Meta polja nisu vidljiva u standardnom admin ekranu, pa u Fazi 4 predlažem mali meta box "Detalji modela" (inače ih vlasnik ne može menjati).
3. **Kvalitet teksta u metama:** u 1 od 72 slučaja ostala je interna napomena ("široko korišćen opis za Deerupt liniju"). Tema to ne može da popravi; preporuka je ručna provera.
4. **Veličine:** globalni atribut `Veličina` (79 proizvoda) i "Boja" (4). Vrednosti su nedosledne: "37 1/3", "40 2/3", "2XL", "Bela". Prikaz na čipovima: "37⅓", "40⅔" preko formatera u kodu.
5. **Akcija je na varijacijama**, ne na roditelju: 50 od 219 varijacija ima sale cenu, a roditeljski `Sale price` je svuda prazan. Badge "-%" i precrtana cena moraju da se računaju iz varijacija (WooCommerce to radi preko `get_variation_prices`, ja koristim to).
6. **Zalihe su stvarne i vođene po varijaciji** (153 na stanju, 60 sa 0, brojevi 1 do 6+). "Poslednji komad / Još N" je moguć iz pravih podataka.
7. **7 simple proizvoda** (odeća, oprema) nemaju veličine, pa kupovni blok mora da radi i bez izbora veličine.
8. **Slike:** 4 do 16 po proizvodu, prva se u exportu ponavlja dva puta. Formati su već webp, što je dobro.
9. **SEO dokument pominje Yoast sitemap, a export sadrži Rank Math polja.** Treba mi potvrda koji je plugin aktivan (vidi pitanja).

---

## 1. Stack (moja izjava, jer ga skill ne navodi)

WordPress block tema (FSE) + WooCommerce blokovi, `theme.json` v3, vanilla CSS i vanilla JS (ES modules), bez Tailwinda, Reacta i jQuery-ja u našem kodu. PHP 8.2/8.3.

## 2. Arhitektura: prihvatam, uz tri izmene koje predlažem

Blok tema ostaje. Razlozi za izmene su stvarni, ne stilski:

**Izmena A: kupovni blok na stranici proizvoda pišemo sami (dinamički blok), umesto WooCommerce "Add to Cart + Options" bloka.**
Zvanična dokumentacija WooCommerce-a i dalje ga označava kao **Beta**, uključujući izbor varijacija kao "pills" ([dokumentacija](https://woocommerce.com/document/add-to-cart-options-block/)). Beta blok može da promeni markup i ponašanje između WooCommerce ažuriranja, a vi sajt održavate sami. Klasični obrazac varijacija (`wc-add-to-cart-variation`) zavisi od jQuery-ja, što prekršava budžet. Naš blok: server-renderovani čipovi iz varijacija, radi **bez JS-a** kao običan `POST` (`add-to-cart`, `variation_id`, `attribute_pa_velicina`), a vanilla JS samo dodaje trenutno osvežavanje cene i stanja. Ostaje 100% WooCommerce košarica, cene i zalihe.
*Prvo radim kratki spike (Faza 1) da proverim da li je Beta blok u vašoj verziji dovoljno dobar. Ako jeste, uštedimo kod i ja odustajem od izmene.*

**Izmena B: mini-korpa.** WooCommerce Mini-Cart blok učitava svoj frontend JS (wc-blocks, Interactivity API) na svim stranicama. **Ne znam koliko to košta u gzip KB dok ne izmerim.** Budžet od 60 KB JS ukupno je realno ugrožen. U Fazi 1 merim. Ako prelazi budžet, drawer pišemo sami nad Store API (`/wc/store/v1/cart`) i učitavamo ga tek na prvu interakciju.

**Izmena C: filteri kataloga.** WooCommerce Product Filters blok postoji (cena, atribut, dostupnost, ocena, kategorija, brend) i radi uz Product Collection, ali iz pretrage nisam mogao da potvrdim stabilnost statusa niti da li postoji filter "akcija". Spike u Fazi 3. Rezervni plan: filteri kao obični linkovi sa URL parametrima koje obrađuje naš `woocommerce_product_query` kuka (rade bez JS-a, indeksiranje rešava Rank Math / noindex za filtrirane URL-ove).

**Nije u budžetu (iskreno):** Cart i Checkout blokovi učitavaju znatno više JS-a nego 60 KB. Budžet iz zadatka važi za početnu, kategoriju i proizvod, pa je to u skladu, ali cart i checkout ću izmeriti i prijaviti posebno. Provera pre odluke: **koji platni gateway koristite?** Mnogi domaći gateway-i ne podržavaju checkout blok. Ako vaš ne podržava, ostajemo na klasičnom checkoutu uz jQuery samo na toj stranici.

## 3. Design sistem (nacrt za odobrenje)

### 3.1 Paleta (tokeni u `theme.json`, bez sirovih hex vrednosti u komponentama)

| Token | Vrednost | Upotreba | Kontrast |
|---|---|---|---|
| `ink` | #111111 | tekst, primarna dugmad | 17,2:1 na off-white |
| `paper` | #FFFFFF | pozadina | n/a |
| `surface` | #F6F4F0 | topla off-white površina, pozadina slika | n/a |
| `surface-2` | #ECE9E3 | hover, prazni okviri | n/a |
| `line` | #D9D5CD | hairline ivice | dekor |
| `muted` | #5E5E5E | sekundarni tekst | 5,9:1 na `surface` |
| `accent` | **#BD0000** | cena akcije, badge, fokus akcenat, link hover | 6,6:1 na belom, 6,0:1 na surface |
| `accent-strong` | #9A0000 | hover/pressed | veći od 6,6:1 |
| `footer-bg` | #111111 + tekst #F6F4F0, sekundarni #A8A8A8 | tamni footer | 17,2:1 / 7,9:1 |
| success / warning / error | biram u Fazi 1 uz proveru AA | poruke | AA |

**Predlog crvene:** jedna primarna crvena **#BD0000** (vaša boja sa sajta, 6,6:1 na belom). Logo linija #FF0000 daje samo 4,0:1, što je dovoljno za grafiku, ali ne za tekst, pa se #FF0000 koristi samo unutar logoa. Crvena se pojavljuje retko: badge popusta, cena akcije, aktivni indikator, fokus akcenat. Dugmad su crna.

### 3.2 Tipografija (odluka vlasnika, vidi pitanja)

Predlog: **Archivo** (display, varijabilni, ose težine i širine, velika slova, široke varijante za naslove) + **Inter** ili **Hanken Grotesk** za tekst. Obe su OFL. Dok ne preuzmem fajlove i izmerim, ne tvrdim veličinu u KB. U Fazi 1:
- proveravam glife `č ć š ž đ Č Ć Š Ž Đ` pomoću fonttools skripte pre izbora,
- podskup `latin` + `latin-ext`, WOFF2, `font-display: swap`,
- preload najviše 2 fajla, bez Google Fonts CDN-a (preuzimanje je jednokratno, u razvoju).
Ako želite da ostane **Hind** (vaš postojeći font), mogu, ali deluje generički uz editorijalni pravac. Odluka je vaša.

Skala: `clamp()`; osnova 17 px, line-height 1,5; display uppercase `clamp(2.5rem, 1.2rem + 6vw, 6.5rem)`, line-height 0,95 do 1,0; letter-spacing blago negativan na displayu, blago pozitivan na uppercase malim oznakama.

### 3.3 Razmak, mreža, komponente

- Razmak u skali 4 px (`--space-1..9`), prozračna gustina (density 3): sekcije 64 do 128 px vertikalno.
- Kontejner 1440 px, mreža 12 kolona, 16 px gutter (mobil), 24 px (desktop).
- Hairline 1 px umesto kartica sa senkama; radijus 0 do 2 px (oštro, editorijalno).
- Dugmad: primarna crna (pun), sekundarna outline, tercijarna tekstualna sa podvlakom. Min visina 48 px.
- Kartica proizvoda: slika 4:5, `object-fit: contain`, `mix-blend-mode: multiply` na `surface` pozadini, naziv, cena, badge. Drugu sliku na hover ( samo `(hover:hover)`), čipovi veličina na hover i na fokus (i long-press na dodir kao progresivno poboljšanje, ne jedini put do informacije).
- Motion: samo `opacity`/`transform`, 150 do 300 ms, `cubic-bezier(.2,.7,.2,1)`, `prefers-reduced-motion` gasi sve.

## 4. Model sadržaja (šta se gde uređuje)

| Sadržaj | Gde se uređuje | Podrazumevano |
|---|---|---|
| Sekcije početne (hero, pločice kategorija, Novo/Akcija redovi, priča o brendu, USP traka, FAQ, newsletter, pomoć za veličinu, Instagram blok) | Site Editor / registrovani pattern-i; svaki naslov, paragraf i dugme je običan blok | Pattern-i sa pravim srpskim tekstom |
| Traka obaveštenja, prag besplatne dostave, kratki tekstovi dostave/povraćaja/plaćanja, kontakt, društvene mreže, tabela veličina, footer kolone, newsletter tekst | **Izgled > MarinaNova podešavanja** (Settings API, jedna opcija `marinanova_settings`) | Default u kodu + "vrati na podrazumevano" po polju |
| Stavke poverenja (ikona, naslov, tekst, uključeno) | isti ekran, ponavljajuća lista | **Sve ISKLJUČENO**, oznaka "potvrditi sa vlasnikom" |
| Proizvod: opis, kratki opis, atributi, brend | nativni WooCommerce | n/a |
| Spec tabela | meta polja + atributi automatski; prazno se krije | Mapa ključ → oznaka u kodu; meta box u Fazi 4 |
| Kategorija: uvodni tekst, slika pločice | opis termina, sličica termina | n/a |
| Tekstovi u kodu | samo preko `__()` sa domenom `marinanova` | `docs/EDITING-GUIDE.md` (srpski) popisuje svako mesto |

## 5. Stablo fajlova

```
marinanova/
  style.css                 (samo zaglavlje teme)
  theme.json
  functions.php
  inc/
    setup.php  performance.php  settings.php  settings-defaults.php
    woocommerce.php  product-specs.php  archive-title.php  svg-sprite.php
    blocks/            (dinamički blokovi: buy-box, size-guide, mini-cart ako bude potreban)
  templates/   (index, front-page, home, page, single, archive, search, 404,
                single-product, archive-product, taxonomy-product_cat,
                product-search-results, cart, checkout, page-cart, page-checkout,
                order-confirmation, my-account...)
  parts/       (header, header-minimal, footer, announcement-bar, mini-cart-drawer, breadcrumbs)
  patterns/    (home-*.php, faq, usp, newsletter, size-help, 404-help, story-slot)
  assets/css/  (base.css, critical.css, header.css, card.css, product.css, ...)
  assets/js/   (nav.js, drawer.js, buybox.js, reveal.js, modal.js; ES moduli, defer)
  assets/fonts/ assets/img/ (SVG sprite, logo; bez fotografija)
  languages/  docs/  README.md  CHANGELOG.md  .gitignore
  tools/build-zip.mjs       (Node skripta; isključuje dev fajlove)
```
Predlog lokacije: `Nova app i sajt/marinanova/` (pored `tema1`). Demo podaci i slike idu u git-ignored `Nova app i sajt/.local/` i nikad u temu.

## 6. Test okruženje (važno, iskreno)

Na računaru nema PHP-a ni Docker-a. Predlog:
- **WordPress Playground CLI** (`npx @wp-playground/cli`), radi u Node-u bez Dockera. Podržano je i kroz `wp-env --runtime=playground` ([make.wordpress.org](https://make.wordpress.org/playground/2026/02/06/wp-env-now-runs-wordpress-with-playground-runtime/)). Ograničenja: nema slanja mejla, ograničena mreža, slabija WP-CLI podrška. Sadašnju podršku za WooCommerce blueprint moram da proverim u Fazi 1; nisam našao zvaničan primer.
- **Lighthouse** preko `npx` (potreban je Chrome; u `/Applications` Chrome nisam našao, pa treba da ga instaliram ili da mi kažete gde je).
- **PHPCS/WPCS** zahteva PHP. Opcije: (a) `brew install php composer` (menja vaš računar, pa tražim dozvolu), (b) PHP kroz Playground WASM, što nije sigurno da radi. Bez ovoga PHPCS ne mogu da pokrenem i to ću eksplicitno napisati u izveštajima, a ne da tvrdim da je prošlo.
- **Merenje:** Playground nije LiteSpeed i TTFB nije reprezentativan. Prijavljujem LCP/CLS/INP/broj zahteva/KB, a TTFB samo kao napomenu. Prave brojeve na hostingu dobijate tek na staging kopiji.

## 7. Budžet performansi i kako ga postižemo

Ciljevi iz zadatka važe (LCP < 2,0 s, CLS < 0,05, INP < 200 ms, početna < 60 zahteva, JS < 60 KB gz, CSS < 50 KB gz, bez blokirajućeg JS-a). Postizanje: inline kritični CSS (samo above-the-fold, ciljano ispod 14 KB), CSS po šablonu, `defer`/module, `fetchpriority=high` na LCP slici, `srcset/sizes` i `width/height` svuda, `content-visibility:auto` na sekcijama ispod preloma, SVG sprite, Speculation Rules, uklanjanje emoji/embed/jQuery Migrate/dashicons i nepotrebnih WooCommerce stilova. Bez sopstvenog keša (LiteSpeed Cache). Svaki rezultat ide u `docs/PERFORMANCE.md` sa stvarnim brojem, ne procenom.

**Rizici za budžet:** wc-blocks/Interactivity JS (Izmena B), `wc-order-attribution` skripta, i neujednačene slike (900 do 2000 px). Za slike: prikaz u 4:5 okviru sa `contain` na `surface`, i preporuka za regeneraciju sličica (`wp media regenerate`) na stagingu. AVIF: nema `avifenc` lokalno, a 19 AVIF fajlova već postoji; izbor formata zavisi od servera (Imagick/GD). Pitanje za host, ne za temu.

## 8. Rizici i otvorene tačke

1. WooCommerce Beta blokovi (Izmena A/B/C): odluke tek posle spike-ova.
2. Platni gateway i checkout blok (vidi §2).
3. Brend je prazan u podacima; spec polja su meta bez editora.
4. 42 od 72 obuće imaju jednu veličinu: kartica i buy box moraju da izgledaju dobro sa jednom veličinom i sa rasprodatim; sve ću testirati sa stvarnim demo podacima.
5. Duplirani nazivi (Adilette Aqua x3, Shower M x2): tema može da prikaže šifru modela kao sekundarnu oznaku da se razlikuju.
6. Rank Math breadcrumbs vs WooCommerce breadcrumb blok: izbegavam dupli markup (jedan izvor, bira se kukom).
7. Tvrdnje o poverenju: sve isključeno dok vlasnik ne potvrdi (u skladu sa pravilom 3).
8. Slike modela (Adidas packshot-ovi) su vaše za korišćenje samo ako ste ovlašćeni (pitanja 14, 15 u vašem dokumentu). Tema ne sadrži Adidas logotipe.

## 9. Faze

| Faza | Sadržaj | Izlaz za pregled |
|---|---|---|
| 1 Temelj | `theme.json`, tokeni, osnovni CSS, header/footer, traka obaveštenja, skeleton podešavanja, zip skripta, Playground + uvoz CSV-a i slika; **spike**: Beta kupovni blok, mini-korpa KB, gateway/checkout | Lokalni sajt sa headerom/footerom, prvi izmereni brojevi, odluke A/B |
| 2 Početna | patterni sa srpskim tekstom, slot za scrollytelling/3D | Početna u Playground-u |
| 3 Katalog | arhiva, kartica, filteri (spike C), pretraga | Kategorija i pretraga |
| 4 Proizvod | buy box, čipovi, sticky bar, vodič za veličine, spec tabela, meta box | Stranica proizvoda (1 veličina, rasprodato, akcija) |
| 5 Korpa/checkout/nalog/404/blog/pravne | | |
| 6 Polish i QA | merenja, axe/Lighthouse, Safari/Firefox, EDITING-GUIDE, PERFORMANCE, README, CHANGELOG, `marinanova.zip` | Završen zip |

---

## 10. Šta mi treba od vas (blokira Fazu 1)

1. **Font:** Archivo + Inter (moj predlog) ili ostaje Hind?
2. **Crvena:** da li prihvatate #BD0000 kao jedinu crvenu (logo linija ostaje #FF0000)?
3. **Lokacija teme:** `Nova app i sajt/marinanova/` je u redu?
4. **Dozvola za instalaciju alata na ovom računaru:** `brew install php composer` (za PHPCS i lokalni PHP) i Chrome/Chromium za Lighthouse. Ili da pokušam bez toga (tada PHPCS neće biti pokrenut).
5. **Platni gateway** na sajtu (da proverim kompatibilnost sa checkout blokom).
6. **Rank Math ili Yoast** je trenutno aktivan?
7. **Brend:** da li je u redu da u demo uvozu postavim brend "Adidas" na sve proizvode (samo lokalno)?
8. **Fotografije za hero i pločice kategorija:** koristim vaše slike proizvoda; ako želite lifestyle hero, treba da ga date, jer ne koristim stock ni AI slike.
9. **Kopija za dostavu, povraćaj i plaćanje:** za podrazumevane tekstove nemam potvrđene činjenice (rok dostave, uslovi povraćaja, načini plaćanja). Predlažem neutralne tekstove bez brojeva dok ne potvrdite.
