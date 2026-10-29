# Prompt pro migraci vybalancujeme2.webnode.cz → GitHub Pages

> Zkopíruj celý obsah sekce "PROMPT" níže do nové AI session (opencode/Claude
> apod.), až budeš mít přístup k `vybalancujeme2.webnode.cz`. Prompt je
> samostatně použitelný a neobsahuje žádnou závislost na předchozí konverzaci.

---

## PROMPT

Potřebuju zmigrovat web z Webnode na GitHub Pages. Jde o DRUHÝ běh téhož
postupu, který už jednou proběhl úspěšně pro sesterský projekt (stejná firma,
jiný zdrojový Webnode projekt). Cílem je nahradit obsah v existujícím
repozitáři novou verzí z jiného zdroje, při zachování historie staré verze
v samostatné větvi.

### Kontext a cíl

- Zdrojový web k migraci: https://vybalancujeme2.webnode.cz/
- Cílový repozitář: https://github.com/vybalancujeme/vybalancujeme.github.io
  (GitHub Pages "user page" repo, custom doména vybalancujeme.cz přes CNAME
  soubor)
- Lokální pracovní adresář (už naklonovaný, remote "origin" nastavený):
  /Users/cz102829/git/Personal/vybalancujeme.github.io
- GitHub účet: vybalancujeme (přihlášený přes `gh auth login`, ověř `gh auth
  status` na začátku)
- Aktuální `main` větev obsahuje starou migraci z JINÉHO zdroje
  (vybalancujeme3.webnode.cz) — tenhle obsah bude nahrazen, ale NE smazán:

  KROK 0 (udělej jako úplně první věc): Vytvoř novou větev s názvem
  `vybalancujeme2`, která bude ukazovat na současný HEAD `main` (commit
  bd9f038720e239a4f1c156463d475d27ecfb3ee3 nebo novější, pokud se mezitím
  něco změnilo) — čistě jako záloha/archiv staré verze. Pak pokračuj
  s veškerou prací NA `main`. Použij `gh api -X POST repos/vybalancujeme/
  vybalancujeme.github.io/git/refs -f ref=refs/heads/vybalancujeme2 -f
  sha=<current-main-sha>` (viz důvod proč ne `git push` níže).

### KRITICKÉ omezení prostředí — přečti si nejdřív

**Příkaz `git push` je v tomhle prostředí blokovaný bezpečnostním pravidlem
(bash permission "deny"), bez možnosti obejít.** Jakýkoliv pokus o `git push`
(i obalený v jiném příkazu) selže. Pro nahrávání změn na GitHub proto MUSÍŠ
používat GitHub REST/Git Data API přes `gh api` (CLI je nainstalované,
přihlášené). Osvědčený postup z minula:

1. Lokálně dělej běžné `git add`, `git commit` (bez push)
2. Pro malý počet změn (pár souborů): použij Contents API
   (`gh api -X PUT .../contents/<path>` s base64 obsahem a SHA předchozí
   verze pro update, bez SHA pro nový soubor; `gh api -X DELETE
   .../contents/<path>` pro smazání)
3. Pro velký počet změn (desítky/stovky souborů, přejmenování): použij
   Git Data API naplno:
   - `gh api repos/.../git/refs/heads/main --jq '.object.sha'` — zjisti
     aktuální remote HEAD
   - `git diff --name-status -M HEAD~1 HEAD` — zjisti co se změnilo
     (status A/M/D/R)
   - Pro změněný/nový obsah (status A, M, nebo R kde se obsah změnil):
     nahraj blob přes `gh api -X POST .../git/blobs` s base64 obsahem,
     OVĚŘ že vrácené SHA odpovídá `git rev-parse HEAD:<cesta>` (musí sedět
     — je to kontrola, že se nahrálo přesně totéž co máš lokálně)
   - Pro čisté přejmenování (status R, obsah beze změny): není třeba
     nahrávat nový blob, stačí odkázat na existující SHA starého souboru
   - Sestav kompletní tree (`gh api -X POST .../git/trees` s `base_tree`
     = SHA rodičovského stromu a polem `tree` obsahujícím změny — `sha:
     null` pro smazání cesty, SHA blobu pro přidání/úpravu)
   - Vytvoř commit (`gh api -X POST .../git/commits` s `tree` a `parents`)
   - Updatni ref (`gh api -X PATCH .../git/refs/heads/main -f sha=<nový
     commit> -F force=false`) — NIKDY nepoužívej force, jen fast-forward
   - Po úspěšném update: `git fetch origin && git checkout -B main
     origin/main` pro synchronizaci lokální kopie
4. Síťové timeouty (`TLS handshake timeout`) se občas stávají u `gh api`
   volání — proveď retry, nejde o fatální chybu

Pracuj v menších, logicky oddělených commitech (ne jeden obří), stejně jako
u prvního běhu (tam to bylo cca 7 commitů: initial mirror, CNAME
temp-remove, ?ph= filename fix, font cleanup, hidden-CDN-refs fix, asset
reorg, CNAME restore).

### Postup migrace (přesně to, co fungovalo u vybalancujeme3)

#### 1. Průzkum zdroje
- Stáhni a prohlédni `https://vybalancujeme2.webnode.cz/sitemap.xml`
  (seznam všech stránek) a `/robots.txt`
- Zjisti strukturu stránek, zda má web formulář (kontakt), galerii,
  Google Maps embed, GTM/analytics
- Pokud existuje GTM id, zeptej se uživatele, jestli je jeho (stejně jako
  posledně — pravděpodobně NE, protože to byl Webnode default)

#### 2. Stažení (wget mirror)
```
wget --mirror --page-requisites --convert-links --adjust-extension \
  --span-hosts --no-parent -e robots=off \
  --domains=vybalancujeme2.webnode.cz,<cloudfront-cdn-domena>,<cbaul-cdn-domena> \
  https://vybalancujeme2.webnode.cz/
```
**NEVKLÁDEJ do `--domains` seznamu `fonts.gstatic.com`/`fonts.googleapis.com`**
— Google Fonts nech jako externí absolutní odkazy (viz bod 5, fonty se stejně
ukážou být skoro celé nepoužité a stahovat stovky fontových variant je
zbytečné plýtvání časem).

Po stažení přesuň obsah z `vybalancujeme2.webnode.cz/*` (podsložka
pojmenovaná podle zdrojové domény) o úroveň výš do kořene repa, a oprav
relativní odkazy v HTML (odeber jeden segment `../` ze všech odkazů na CDN
domény, protože se mění hloubka vnoření).

#### 3. Oprava `Content-Type` bugu (soubory s `?ph=...` v názvu)
GitHub Pages určuje `Content-Type` čistě podle přípony souboru. Soubory
stažené z Webnode CDN mají v názvu cache-busting parametr (např.
`logo.svg?ph=HASH`), takže "skutečná" přípona na konci názvu není
rozpoznatelná a server je servíruje jako `application/octet-stream` — to
rozbíjí `<embed>` elementy (nabídnou se ke stažení místo vykreslení) a
může rozbít `type="module"` skripty (striktní MIME type check u ES
modulů).

Najdi všechny soubory obsahující `?ph=` v názvu (`find . -name "*\?ph=*"`),
přejmenuj je useknutím všeho od prvního `?` dál (zachovej případnou
duplicitní příponu, kterou wget někdy přidal navíc u CSS/HTML), a over,
že žádný soubor nemá "?" v názvu doslova na disku, ale HTML odkazy musí
mít "%3Fph=..." PRYČ úplně (ne nahrazené %3F kódováním — to byla slepá
ulička, zkoušel jsem to a nefungovalo, protože python http.server/GitHub
Pages dekódují %3F zpět na "?" a hledají soubor s doslovným "?" v názvu;
jednodušší a správné řešení je parametr úplně odstranit).

#### 4. Analýza a čištění nepoužitých fontů (DŮKLADNĚ, i časově náročné)
Stejně jako u vybalancujeme3, Webnode bundluje JEDEN sdílený CSS soubor
s `@font-face` definicemi pro ~60 fontů dostupných v jejich vizuálním
editoru šablon — bez ohledu na to, kolik z nich web skutečně používá.
U vybalancujeme3 to bylo 353 `@font-face` pravidel / 177 `.woff2` souborů,
z nichž reálně web používal jen 2.

Postup:
1. Najdi CSS soubor s nejvíc `@font-face` bloky (`grep -c "@font-face"
   na všechny CSS soubory`)
2. Zjisti, které `font-family` jsou SKUTEČNĚ použité mimo `@font-face`
   definice — projdi VŠECHNY CSS soubory a hledej `font-family:` deklarace
   v běžných selektorech (`h1`, `h2`, `p`, `body`/`.t`, `blockquote`,
   menu, tlačítka, patička, formuláře) — ne jen na hlavní stránce, ale
   napříč kompletní sdílenou typografií (soubor s `data-wnd_typography_file`
   atributem v `<link>` tagu)
3. Pro každou skutečně použitou `font-family` najdi odpovídající
   `@font-face` bloky a jejich `url()` odkazy (mohou vést na Google Fonts
   `fonts.gstatic.com` — to je naprosto v pořádku, necháme to tak, NEBO
   na lokální `.woff2` soubor z Webnode CDN — ten musíme zachovat)
4. Smaž všechny `.woff2` soubory, které nejsou v tomhle seznamu "skutečně
   použité"
5. Vystřihni z CSS souboru všechny `@font-face` bloky pro nepoužité fonty
   (regex na `@font-face\{[^}]*\}`, zachovej jen bloky s potřebnou
   `font-family`) — u vybalancujeme3 to zmenšilo 117 KB CSS na 2,7 KB
6. Over, že počet `{` a `}` v CSS po úpravě sedí (základní validace)

#### 5. Hledání SKRYTÝCH odkazů na živé Webnode CDN (kritické, snadno se přehlédnou)
`wget --page-requisites` stahuje jen to, na co se odkazuje přes standardní
HTML atributy (`href`, `src`). U vybalancujeme3 jsme našli 3 schované
závislosti, které zůstaly ukazovat na živé (dočasné) Webnode CDN, a bez
opravy by při zániku Webnode projektu přestaly fungovat:

a) **Fotogalerie/lightbox** — pokud web má `PhotoGalleryBlock` nebo
   podobný widget, hledej atribut `data-content="{...JSON...}"` (HTML-entity
   encoded, obsahuje `&quot;`) — to je NEZÁVISLÁ kopie seznamu obrázků
   (pro JS lightbox komponent), oddělená od viditelných `<img>`/`<picture>`
   tagů. Typicky obsahuje i velikostní varianty (450px/700px), které
   `<picture><source>` markup vůbec nepoužívá a wget je tudíž nestáhl.
   Musíš: (1) rozparsovat JSON (html.unescape), (2) vytáhnout všechny
   `src` hodnoty, (3) zjistit, které z nich chybí lokálně, (4) dostáhnout
   je přímo z živého CDN, (5) přepsat absolutní URL v JSON na relativní
   lokální cesty

b) **`og:image` meta tag** — kontroluj, že neukazuje na absolutní
   `https://<cbaul-cdn-domena>/...` URL. Musí být absolutní URL (OG spec
   to vyžaduje), ale směřující na NAŠI doménu
   (`https://vybalancujeme.github.io/...` nebo po DNS cutoveru
   `https://vybalancujeme.cz/...`)

c) **Dynamicky importovaný polyfill** — hledej v inline `<script>` tagách
   vzor `import("https://...")` (typicky `container-query-polyfill` nebo
   podobný fallback pro starší prohlížeče) — není v žádném `<script src>`
   ani `<link>` tagu, je to JS string uvnitř kódu, wget ho nenajde.
   Stáhni ho ručně a uprav referenci na lokální cestu.

d) **Obsolentní `<link rel="preconnect">`** — wget někdy převede
   `<link rel="preconnect" href="https://NĚJAKÁ-CDN-DOMÉNA">` (bez cesty)
   na relativní odkaz na mirror té domény (`cdn-domena/index.html`), což
   je nesmyslné (preconnect na vlastní doménu nedává smysl, všechno teď
   hostujeme sami). Smaž tyhle `<link rel="preconnect">` tagy úplně,
   pokud ukazují na CDN domény, co teď mirrorujeme my sami.

Obecná metoda hledání: po celém procesu migrace spusť
`grep -oE '"[^"]*<cdn-domena>[^"]*"' na všech 5(+) HTML stránkách a
zkontroluj úplně VŠECHNY zbývající výskyty (i uvnitř JS/JSON blobů, ne
jen HTML atributů) — jediné, co smí zůstat, jsou neškodné `wnd.$system`
JS config hodnoty (`staticFiles`, `staticCDNServers`, `awsS3FilesPath`,
`localFilesPath`) — to je jen metadata popisující Webnode infrastrukturu,
nic se podle nich nenačítá, lze nechat být.

#### 6. Odstranění Webnode brandingu a GTM
- `<div class="wnd-free-stripe cf">...</div>` (spodní lišta "Vytvořeno
  Webnode") — smazat celé
- `<span class="sf b">...Vytvořeno službou...Webnode...</span>` v patičce
  — smazat (zachovej "Obrázky poskytl Pexels" credit a "Cookies" odkaz)
- GTM `<noscript><iframe src="...googletagmanager.com...">` — smazat
  (pokud není uživatelův vlastní GTM — zeptej se, stejně jako posledně
  bude pravděpodobně Webnode default, podle uživatele bude situace
  identická jako minule)
- Google Maps iframe (pokud web má kontakt s mapou) — pokud používá
  Webnode proxy (`web-2022.webnode.it/widgets/googlemaps/`), nahraď
  přímým Google Maps embedem: `https://www.google.com/maps?q=<lat>,<lng>
  &z=15&output=embed`

#### 7. Kontaktní formulář → mailto placeholder
Nahraď `<form>` element (POST na Webnode backend, GitHub Pages neumí
zpracovat server-side) stylovaným odkazem/tlačítkem:
```html
<!-- TODO: nahradit odkazem na Formspree/Web3Forms, aby formulář fungoval přímo na webu. Dočasně jen mailto odkaz. -->
<div class="b-text"><p>Napište nám e-mail a my se vám co nejdříve ozveme zpět.</p></div>
<div class="form-submit b-btn cf b-btn-fs b-btn-1" style="margin-top:20px;">
    <a class="b-btn-l" href="mailto:test@test.cz?subject=Dotaz%20z%20webu%20Vybalancujeme">
        <span class="form-submit-text b-btn-t">Napsat e-mail</span>
    </a>
</div>
```
(stejný `test@test.cz` placeholder jako posledně, uživatel email zatím
neřeší)

#### 8. Lokální ověření PŘED každým nahráním na GitHub
Po každé logické fázi (ne až na konci!) spusť lokální HTTP server a
kompletní kontrolu:
```
python3 -m http.server 8743   # na pozadí přes nohup+disown, jinak timeoutuje
```
Napiš si Python skript, co projde všech 5 (či více) HTML stránek, vytáhne
VŠECHNY `href`/`src`/`data-src` odkazy (a separátně i skryté odkazy
z galerie JSON — `&quot;src&quot;:&quot;...&quot;` vzor) a pro každý
unikátní odkaz ověří: (a) HTTP status 200, (b) `Content-Type` NENÍ
`application/octet-stream`. Nehledě na to, jak malá se zdá změna, spusť
tuhle kontrolu znovu pokaždé.

#### 9. Reorganizace do čitelné struktury
Po vyřešení všech bugů přesuň soubory z hash-pojmenovaných CDN-mirror
složek do:
```
assets/
├── css/     (base.css, base-desktop.css, color-scheme.css,
              color-scheme-desktop.css, color-scheme-additive.css,
              typography.css, typography-desktop.css, fonts.css,
              print.css, container-query-polyfill-source.css)
├── js/      (widgets.js, site-runtime.js, translations-cs.js,
              container-query-polyfill.js)
├── fonts/   (jen skutečně použité .woff2, popisně pojmenované)
├── icons/   (favicon.svg, favicon-16.svg, favicon.ico, logo.svg,
              icon-email.svg, icon-phone.svg, icon-address.svg,
              + cokoliv dalšího jako lang-flag-sprite.png)
└── images/  (photo-<pexels-nebo-jiné-id>[-450|-700].jpeg/.webp)
```
Název každého CSS souboru urči podle jeho `data-wnd_*` atributu nebo
`media=` atributu v originálním `<link>` tagu (např.
`data-wnd_typography_file` → `typography.css`, `media="print"` →
`print.css`) — NE podle hádání z obsahu.

Po přesunu MUSÍŠ zkontrolovat a opravit:
- Všechny `href`/`src`/`data-src` v HTML (s ohledem na hloubku vnoření
  stránky)
- `url()` reference UVNITŘ přesunutých CSS souborů (typicky `fonts.css`
  odkazující na `.woff2` a `base.css`/hlavní layout CSS odkazující na
  nějaký sprite `.png`) — najdi je přes `grep -oE "url\([^)]*\)" *.css |
  grep -v "fonts.gstatic.com\|data:"` PŘED přesunem, abys věděl, co
  všechno musíš po přesunu přepočítat
- `og:image` absolutní URL (znovu, protože se teď mění cesta k obrázku)
- JSON galerii (znovu, stejný důvod)

Po reorganizaci smaž teď už prázdné staré CDN-mirror složky a znovu
spusť kompletní lokální kontrolu z bodu 8.

#### 10. Nahrání a nasazení
GitHub Pages je na tomhle repu (`vybalancujeme.github.io` — user/org page)
automaticky zapnuté, build type "legacy", žádný GitHub Actions workflow
potřeba. Po update `main` větve počkej ~15-30s a zkontroluj
`gh api repos/vybalancujeme/vybalancujeme.github.io/pages --jq '.status'`
dokud nehlásí `"built"`. Pak spusť kompletní kontrolu (bod 8) přímo proti
`https://vybalancujeme.github.io/` (NE lokálně).

`CNAME` soubor (obsahuje `vybalancujeme.cz`) už v repu je a NESMÍ se
odstraňovat natrvalo — pokud ho kvůli náhledu dočasně odstraníš (aby
`vybalancujeme.github.io` nepřesměrovávalo na ještě nenastavenou doménu),
NEZAPOMEŇ ho vrátit zpět jako poslední krok před tím, než řekneš
uživateli "hotovo" (tohle se mi minule stalo, že jsem na to zapomněl).

### Co NEřešit / co nechat beze změny
- DNS nastavení (`vybalancujeme.cz` u Forpsi) — řeší uživatel sám, viz
  sekce "DNS nastavení u Forpsi" níže pro přesné hodnoty, kdyby se na to
  uživatel zeptal, ale samotné zadání do Forpsi není úkol téhle session
- Formspree/Web3Forms skutečná integrace — odloženo, jen mailto placeholder
- Neoptimalizuj/nemaž obsah velkých layout CSS souborů (base.css,
  base-desktop.css, color-scheme*.css) — i když vypadají velké (stovky
  KB), jde o skutečně používaný responzivní kód aktivovaný přes `disabled`
  atribut + JS podle šířky okna, ne bloat jako fonty
- Neuprabuj obsah velkých JS bundlů (widgets.js apod.) — jen přejmenuj/
  přesuň soubor jako celek, neriskuj úpravu minifikovaného/React kódu

### Na konci presentuj uživateli
- Finální velikost/počet souborů repa (před/po)
- Seznam všech nalezených a opravených "skrytých" bugů (bod 5)
- Potvrzení, že `CNAME` soubor je v repu přítomný
- Potvrzení, že branch `vybalancujeme2` existuje na GitHubu jako záloha
  staré verze
- Odkaz na živý web k finální kontrole uživatelem

---

## DNS nastavení u Forpsi (pro referenci, řeší uživatel sám)

Doména `vybalancujeme.cz` je registrovaná a DNS spravovaná u Forpsi
(nameservery `ns.forpsi.cz`, `ns.forpsi.it`, `ns.forpsi.net`). Aby doména
fungovala s GitHub Pages, je potřeba ve správě DNS u Forpsi nahradit
současné záznamy (ty teď míří na starý/mrtvý Webnode projekt
`vybalancujeme2.webnode.cz`) těmito:

### Apex doména (`vybalancujeme.cz`) — 4× A záznam
Smazat současné A záznamy (Webnode IP) a přidat tyto 4 oficiální GitHub
Pages IP adresy:

| Typ | Host | Hodnota |
|---|---|---|
| A | @ (nebo prázdné) | `185.199.108.153` |
| A | @ (nebo prázdné) | `185.199.109.153` |
| A | @ (nebo prázdné) | `185.199.110.153` |
| A | @ (nebo prázdné) | `185.199.111.153` |

### `www` subdoména — CNAME
Smazat současný CNAME na `vybalancujeme2.webnode.cz` a nahradit:

| Typ | Host | Hodnota |
|---|---|---|
| CNAME | www | `vybalancujeme.github.io.` |

### Poznámky
- **TTL**: nechat defaultní, nebo nízké (300–3600s) pro rychlejší
  propagaci
- Propagace DNS trvá obvykle několik minut až pár hodin (max 24–48h
  u Forpsi)
- Po úspěšné propagaci DNS GitHub automaticky nabídne/vystaví HTTPS
  certifikát (Let's Encrypt) — to se stane samo, může trvat pár hodin
  navíc po zafungování DNS; pak lze v Pages nastavení zapnout "Enforce
  HTTPS"
- `CNAME` soubor v repozitáři (obsahuje `vybalancujeme.cz`) musí zůstat
  přítomný, aby GitHub věděl, že požadavky na tuhle doménu patří do
  tohoto repozitáře — viz vysvětlení v hlavní konverzaci / sekce výše
