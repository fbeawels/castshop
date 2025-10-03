# cs.js

## Review

## 1. Summary  

* **Purpose** – This file is a *language plugin* for the CKEditor WYSIWYG editor.  
  It supplies Czech (`cs`) translations for all UI strings that CKEditor can display.  
* **Key Components** –  
  * `CKEDITOR.lang.cs` – the top‑level object that CKEditor looks up when the user
    selects Czech.  
  * Nested objects (`common`, `link`, `table`, `image`, `flash`, …) that group
    related strings (toolbar labels, dialog titles, error messages, etc.).  
  * Placeholders (`%1`, `%2`) are used in a few strings for runtime interpolation.  
* **Design** – A simple data‑driven approach: a flat object literal with nested
  namespaces. No executable code, only static strings.  
* **Libraries/Frameworks** – Relies on the **CKEditor** core (the `CKEDITOR` global).

---

## 2. Detailed Description  

### Execution Flow
1. CKEditor loads its core script.  
2. When the user selects Czech as the UI language, the core tries to load
   `lang/cs.js` (or similar).  
3. This file executes and registers the object `CKEDITOR.lang.cs`.  
4. All subsequent calls to `CKEDITOR.lang.get()` (or the internal helper)
   retrieve the appropriate string from this object.  

### Runtime Behavior
* The file contains **no side effects** – it simply assigns a large object
  to a global property.  
* CKEditor treats any missing key as an empty string; therefore, untranslated
  keys do not crash the editor but may produce empty UI elements.  

### Structure & Organization
* The object is hierarchically organized by feature (e.g. `link`, `table`,
  `image`).  
* Within each feature, sub‑objects like `common`, `toolbar`, `menu`, `title`,
  etc., hold the actual label texts.  
* Keys are written in *camelCase* (e.g. `targetNew`, `popupResizable`) to
  match CKEditor’s internal naming convention.

### Assumptions & Constraints
* **Encoding** – The file must be UTF‑8 (no BOM) so that Czech diacritics are
  displayed correctly.  
* **Placeholders** – `%1`, `%2` are assumed to be replaced by CKEditor’s
  string‑formatting routine.  
* **Missing Translations** – If a key is omitted, CKEditor falls back to the
  English default.  

### Architectural Choices
* **No code generation** – The file is hand‑written; a build step could
  auto‑merge multiple translation files or generate a JSON bundle.  
* **Duplication** – A few keys appear twice (e.g. `target` under `link`);
  this is harmless but could confuse translators or future maintainers.  
* **Hardcoded strings** – All UI text is stored directly; no lazy loading or
  on‑demand fetching. This keeps the language file lightweight.

---

## 3. Functions/Methods  

| Symbol | Type | Purpose | Notes |
|--------|------|---------|-------|
| *None* | – | – | The file is purely declarative; there are no functions, methods, or classes defined. |

> **Conclusion:**  
> Because there is no executable logic, there are no side‑effects, no public API,
> and no reusable utilities to review.

---

## 4. Dependencies  

| Dependency | Category | Remarks |
|------------|----------|---------|
| `CKEDITOR` global | **Framework** | Provided by the CKEditor core. The language file expects this global to exist. |
| None else | – | No third‑party libraries or platform‑specific APIs are used. |

> **Implication** – The file can be dropped into any CKEditor build that
> contains the core; no additional setup is required.

---

## 5. Additional Notes  

### 5.1 Strengths  
* **Clarity & Simplicity** – The object literal is straightforward to read and
  edit.  
* **Modular Grouping** – Logical separation by feature makes it easier for
  translators to focus on a specific UI area.  
* **Internationalization Friendly** – Uses placeholders (`%1`, `%2`) and
  consistent key names expected by CKEditor’s i18n system.

### 5.2 Areas for Improvement  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| Duplicate keys (`target` appears twice under `link`). | Confusion, possible accidental override. | Remove one copy or consolidate under a single key. |
| Misspelled key `acccessKey` (three “c”). | Translations may never be used. | Correct to `accessKey`. |
| Inconsistent placeholder usage (e.g. `%1` used in many places). | No functional impact but could be confusing to translators. | Ensure placeholders are documented and consistently named. |
| Hard‑coded English defaults in comments (e.g. “Zadejte prosím URL odkazu”). | Not a bug, but some comments are in Czech, others in English. | Uniform comment language for maintainers. |
| Potential missing translations for newer CKEditor features. | UI elements might display English text. | Periodically run a translation completeness check. |
| No linting/formatting checks. | Typos or syntax errors could silently break the file. | Add a lint step (e.g., ESLint with `no-unused-vars` and `quotes` rules). |

### 5.3 Edge Cases  
* **Runtime Interpolation Errors** – If a translation string contains an
  unmatched `%` placeholder, CKEditor may crash or display a malformed string.
* **Encoding Issues** – Serving the file with the wrong charset (e.g., ISO‑8859‑1)
  will garble Czech characters.  
* **Missing `CKEDITOR`** – If the core fails to load, the global will be
  undefined, and the assignment will throw an error.  
* **Language Fallback** – If a key is missing, CKEditor falls back to the
  English string. This behavior is fine but should be documented for
  translators.

### 5.4 Future Enhancements  

1. **Automated Translation Pipeline**  
   * Store raw JSON and generate this file during build to avoid manual
     duplication.  
2. **Translation Validation**  
   * Script to compare available keys against the English reference file
     (`lang/en.js`) and report missing or duplicate keys.  
3. **Internationalization API**  
   * Expose a small helper that returns the translation object for a
     language code, making it easier to support dynamic language switching
     without reloading the page.  
4. **Unit Tests**  
   * Verify that all required keys exist and that placeholders are correctly
     formatted.  
5. **Versioned Language Files**  
   * Include a version stamp (e.g., `CKEDITOR.lang.cs.version = 1.2;`) to aid
     caching and debugging.

---

## 6. Final Verdict  

The file is a well‑structured, minimalistic representation of Czech
translations for CKEditor. Its design aligns with the framework’s
internationalization mechanism, and it contains no executable logic that
could introduce runtime bugs.  

A few housekeeping items (duplicate keys, typos, and consistency in
placeholders/comments) would make it even more robust.  Implementing a
lightweight validation pipeline and automating the generation of these
language files would streamline future translations and reduce the chance
of human error.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.cs={dir:'ltr',editorTitle:'Editor formátovaného textu, %1',source:'Zdroj',newPage:'Nová stránka',save:'Uložit',preview:'Náhled',cut:'Vyjmout',copy:'Kopírovat',paste:'Vložit',print:'Tisk',underline:'Podtržené',bold:'Tučné',italic:'Kurzíva',selectAll:'Vybrat vše',removeFormat:'Odstranit formátování',strike:'Přeškrtnuté',subscript:'Dolní index',superscript:'Horní index',horizontalrule:'Vložit vodorovnou linku',pagebreak:'Vložit konec stránky',unlink:'Odstranit odkaz',undo:'Zpět',redo:'Znovu',common:{browseServer:'Vybrat na serveru',url:'URL',protocol:'Protokol',upload:'Odeslat',uploadSubmit:'Odeslat na server',image:'Obrázek',flash:'Flash',form:'Formulář',checkbox:'Zaškrtávací políčko',radio:'Přepínač',textField:'Textové pole',textarea:'Textová oblast',hiddenField:'Skryté pole',button:'Tlačítko',select:'Seznam',imageButton:'Obrázkové tlačítko',notSet:'<nenastaveno>',id:'Id',name:'Jméno',langDir:'Orientace jazyka',langDirLtr:'Zleva do prava (LTR)',langDirRtl:'Zprava do leva (RTL)',langCode:'Kód jazyka',longDescr:'Dlouhý popis URL',cssClass:'Třída stylu',advisoryTitle:'Pomocný titulek',cssStyle:'Styl',ok:'OK',cancel:'Storno',generalTab:'Obecné',advancedTab:'Rozšířené',validateNumberFailed:'Zadaná hodnota není číselná.',confirmNewPage:'Jakékoliv neuložené změny obsahu budou ztraceny. Skutečně chete otevrít novou stránku?',confirmCancel:'Některá z nastavení byla změněna. Skutečně chete zavřít dialogové okno?',unavailable:'%1<span class="cke_accessibility">, nedostupné</span>'},specialChar:{toolbar:'Vložit speciální znaky',title:'Výběr speciálního znaku'},link:{toolbar:'Vložit/změnit odkaz',menu:'Změnit odkaz',title:'Odkaz',info:'Informace o odkazu',target:'Cíl',upload:'Odeslat',advanced:'Rozšířené',type:'Typ odkazu',toAnchor:'Kotva v této stránce',toEmail:'E-Mail',target:'Cíl',targetNotSet:'<nenastaveno>',targetFrame:'<rámec>',targetPopup:'<vyskakovací okno>',targetNew:'Nové okno (_blank)',targetTop:'Hlavní okno (_top)',targetSelf:'Stejné okno (_self)',targetParent:'Rodičovské okno (_parent)',targetFrameName:'Název cílového rámu',targetPopupName:'Název vyskakovacího okna',popupFeatures:'Vlastnosti vyskakovacího okna',popupResizable:'Umožňující měnit velikost',popupStatusBar:'Stavový řádek',popupLocationBar:'Panel umístění',popupToolbar:'Panel nástrojů',popupMenuBar:'Panel nabídky',popupFullScreen:'Celá obrazovka (IE)',popupScrollBars:'Posuvníky',popupDependent:'Závislost (Netscape)',popupWidth:'Šířka',popupLeft:'Levý okraj',popupHeight:'Výška',popupTop:'Horní okraj',id:'Id',langDir:'Orientace jazyka',langDirNotSet:'<nenastaveno>',langDirLTR:'Zleva do prava (LTR)',langDirRTL:'Zprava do leva (RTL)',acccessKey:'Přístupový klíč',name:'Jméno',langCode:'Orientace jazyka',tabIndex:'Pořadí prvku',advisoryTitle:'Pomocný titulek',advisoryContentType:'Pomocný typ obsahu',cssClasses:'Třída stylu',charset:'Přiřazená znaková sada',styles:'Styl',selectAnchor:'Vybrat kotvu',anchorName:'Podle jména kotvy',anchorId:'Podle Id objektu',emailAddress:'E-Mailová adresa',emailSubject:'Předmět zprávy',emailBody:'Tělo zprávy',noAnchors:'(Ve stránce není definována žádná kotva!)',noUrl:'Zadejte prosím URL odkazu',noEmail:'Zadejte prosím e-mailovou adresu'},anchor:{toolbar:'Vložít/změnit záložku',menu:'Vlastnosti záložky',title:'Vlastnosti záložky',name:'Název záložky',errorName:'Zadejte prosím název záložky'},findAndReplace:{title:'Najít a nahradit',find:'Hledat',replace:'Nahradit',findWhat:'Co hledat:',replaceWith:'Čím nahradit:',notFoundMsg:'Hledaný text nebyl nalezen.',matchCase:'Rozlišovat velikost písma',matchWord:'Pouze celá slova',matchCyclic:'Procházet opakovaně',replaceAll:'Nahradit vše',replaceSuccessMsg:'%1 nahrazení.'},table:{toolbar:'Tabulka',title:'Vlastnosti tabulky',menu:'Vlastnosti tabulky',deleteTable:'Smazat tabulku',rows:'Řádky',columns:'Sloupce',border:'Ohraničení',align:'Zarovnání',alignNotSet:'<nenastaveno>',alignLeft:'Vlevo',alignCenter:'Na střed',alignRight:'Vpravo',width:'Šířka',widthPx:'bodů',widthPc:'procent',height:'Výška',cellSpace:'Vzdálenost buněk',cellPad:'Odsazení obsahu v buňce',caption:'Popis',summary:'Souhrn',headers:'Záhlaví',headersNone:'Žádné',headersColumn:'První sloupec',headersRow:'První řádek',headersBoth:'Obojí',invalidRows:'Počet řádků musí být číslo větší než 0.',invalidCols:'Počet sloupců musí být číslo větší než 0.',invalidBorder:'Zdaná velikost okraje musí být číselná.',invalidWidth:'Zadaná šířka tabulky musí být číselná.',invalidHeight:'zadaná výška tabulky musí být číselná.',invalidCellSpacing:'Zadaná vzdálenost buněk musí být číselná.',invalidCellPadding:'Zadané odsazení obsahu v buňce musí být číselné.',cell:{menu:'Buňka',insertBefore:'Vložit buňku před',insertAfter:'Vložit buňku za',deleteCell:'Smazat buňky',merge:'Sloučit buňky',mergeRight:'Sloučit doprava',mergeDown:'Sloučit dolů',splitHorizontal:'Rozdělit buňky vodorovně',splitVertical:'Rozdělit buňky svisle',title:'Vlastnosti buňky',cellType:'Typ buňky',rowSpan:'Spojit řádky',colSpan:'Spojit sloupce',wordWrap:'Zalamování',hAlign:'Vodorovné zarovnání',vAlign:'Svislé zarovnání',alignTop:'Nahoru',alignMiddle:'Doprostřed',alignBottom:'Dolů',alignBaseline:'Na účaří',bgColor:'Barva pozadí',borderColor:'Barva okraje',data:'Data',header:'Hlavička',yes:'Ano',no:'Ne',invalidWidth:'Zadaná šířka buňky musí být číslená.',invalidHeight:'Zadaná výška buňky musí být číslená.',invalidRowSpan:'Zadaný počet sloučených řádků musí být celé číslo.',invalidColSpan:'Zadaný počet sloučených sloupců musí být celé číslo.',chooseColor:'Výběr'},row:{menu:'Řádek',insertBefore:'Vložit řádek před',insertAfter:'Vložit řádek za',deleteRow:'Smazat řádky'},column:{menu:'Sloupec',insertBefore:'Vložit sloupec před',insertAfter:'Vložit sloupec za',deleteColumn:'Smazat sloupec'}},button:{title:'Vlastnosti tlačítka',text:'Popisek',type:'Typ',typeBtn:'Tlačítko',typeSbm:'Odeslat',typeRst:'Obnovit'},checkboxAndRadio:{checkboxTitle:'Vlastnosti zaškrtávacího políčka',radioTitle:'Vlastnosti přepínače',value:'Hodnota',selected:'Zaškrtnuto'},form:{title:'Vlastnosti formuláře',menu:'Vlastnosti formuláře',action:'Akce',method:'Metoda',encoding:'Kódování',target:'Cíl',targetNotSet:'<nenastaveno>',targetNew:'Nové okno (_blank)',targetTop:'Hlavní okno (_top)',targetSelf:'Stejné okno (_self)',targetParent:'Rodičovské okno (_parent)'},select:{title:'Vlastnosti seznamu',selectInfo:'Info',opAvail:'Dostupná nastavení',value:'Hodnota',size:'Velikost',lines:'Řádků',chkMulti:'Povolit mnohonásobné výběry',opText:'Text',opValue:'Hodnota',btnAdd:'Přidat',btnModify:'Změnit',btnUp:'Nahoru',btnDown:'Dolů',btnSetValue:'Nastavit jako vybranou hodnotu',btnDelete:'Smazat'},textarea:{title:'Vlastnosti textové oblasti',cols:'Sloupců',rows:'Řádků'},textfield:{title:'Vlastnosti textového pole',name:'Název',value:'Hodnota',charWidth:'Šířka ve znacích',maxChars:'Maximální počet znaků',type:'Typ',typeText:'Text',typePass:'Heslo'},hidden:{title:'Vlastnosti skrytého pole',name:'Název',value:'Hodnota'},image:{title:'Vlastnosti obrázku',titleButton:'Vlastností obrázkového tlačítka',menu:'Vlastnosti obrázku',infoTab:'Informace o obrázku',btnUpload:'Odeslat na server',url:'URL',upload:'Odeslat',alt:'Alternativní text',width:'Šířka',height:'Výška',lockRatio:'Zámek',resetSize:'Původní velikost',border:'Okraje',hSpace:'H-mezera',vSpace:'V-mezera',align:'Zarovnání',alignLeft:'Vlevo',alignAbsBottom:'Zcela dolů',alignAbsMiddle:'Doprostřed',alignBaseline:'Na účaří',alignBottom:'Dolů',alignMiddle:'Na střed',alignRight:'Vpravo',alignTextTop:'Na horní okraj textu',alignTop:'Nahoru',preview:'Náhled',alertUrl:'Zadejte prosím URL obrázku',linkTab:'Odkaz',button2Img:'Skutečně chcete převést zvolené obrázkové tlačítko na obyčejný obrázek?',img2Button:'Skutečně chcete převést zvolený obrázek na obrázkové tlačítko?',urlMissing:'Zadané URL zdroje obrázku nebylo nalezeno.'},flash:{properties:'Vlastnosti Flashe',propertiesTab:'Vlastnosti',title:'Vlastnosti Flashe',chkPlay:'Automatické spuštění',chkLoop:'Opakování',chkMenu:'Nabídka Flash',chkFull:'Povolit celoobrazovkový režim',scale:'Zobrazit',scaleAll:'Zobrazit vše',scaleNoBorder:'Bez okraje',scaleFit:'Přizpůsobit',access:'Přístup ke skriptu',accessAlways:'Vždy',accessSameDomain:'Ve stejné doméně',accessNever:'Nikdy',align:'Zarovnání',alignLeft:'Vlevo',alignAbsBottom:'Zcela dolů',alignAbsMiddle:'Doprostřed',alignBaseline:'Na účaří',alignBottom:'Dolů',alignMiddle:'Na střed',alignRight:'Vpravo',alignTextTop:'Na horní okraj textu',alignTop:'Nahoru',quality:'Kvalita',qualityBest:'Nejlepší',qualityHigh:'Vysoká',qualityAutoHigh:'Vysoká - auto',qualityMedium:'Střední',qualityAutoLow:'Nízká - auto',qualityLow:'Nejnižší',windowModeWindow:'Okno',windowModeOpaque:'Neprůhledné',windowModeTransparent:'Průhledné',windowMode:'Režim okna',flashvars:'Proměnné pro Flash',bgcolor:'Barva pozadí',width:'Šířka',height:'Výška',hSpace:'H-mezera',vSpace:'V-mezera',validateSrc:'Zadejte prosím URL odkazu',validateWidth:'Zadaná šířka musí být číslo.',validateHeight:'Zadaná výška musí být číslo.',validateHSpace:'Zadaná H-mezera musí být číslo.',validateVSpace:'Zadaná V-mezera musí být číslo.'},spellCheck:{toolbar:'Zkontrolovat pravopis',title:'Kontrola pravopisu',notAvailable:'Omlouváme se, ale služba nyní není dostupná.',errorLoading:'Chyba nahrávání služby aplikace z: %s.',notInDic:'Není ve slovníku',changeTo:'Změnit na',btnIgnore:'Přeskočit',btnIgnoreAll:'Přeskakovat vše',btnReplace:'Zaměnit',btnReplaceAll:'Zaměňovat vše',btnUndo:'Zpět',noSuggestions:'- žádné návrhy -',progress:'Probíhá kontrola pravopisu...',noMispell:'Kontrola pravopisu dokončena: Žádné pravopisné chyby nenalezeny',noChanges:'Kontrola pravopisu dokončena: Beze změn',oneChange:'Kontrola pravopisu dokončena: Jedno slovo změněno',manyChanges:'Kontrola pravopisu dokončena: %1 slov změněno',ieSpellDownload:'Kontrola pravopisu není nainstalována. Chcete ji nyní stáhnout?'},smiley:{toolbar:'Smajlíky',title:'Vkládání smajlíků'},elementsPath:{eleTitle:'%1 objekt'},numberedlist:'Číslování',bulletedlist:'Odrážky',indent:'Zvětšit odsazení',outdent:'Zmenšit odsazení',justify:{left:'Zarovnat vlevo',center:'Zarovnat na střed',right:'Zarovnat vpravo',block:'Zarovnat do bloku'},blockquote:'Citace',clipboard:{title:'Vložit',cutError:'Bezpečnostní nastavení Vašeho prohlížeče nedovolují editoru spustit funkci pro vyjmutí zvoleného textu do schránky. Prosím vyjměte zvolený text do schránky pomocí klávesnice (Ctrl+X).',copyError:'Bezpečnostní nastavení Vašeho prohlížeče nedovolují editoru spustit funkci pro kopírování zvoleného textu do schránky. Prosím zkopírujte zvolený text do schránky pomocí klávesnice (Ctrl+C).',pasteMsg:'Do následujícího pole vložte požadovaný obsah pomocí klávesnice (<STRONG>Ctrl+V</STRONG>) a stiskněte <STRONG>OK</STRONG>.',securityMsg:'Z důvodů nastavení bezpečnosti Vašeho prohlížeče nemůže editor přistupovat přímo do schránky. Obsah schránky prosím vložte znovu do tohoto okna.'},pastefromword:{toolbar:'Vložit z Wordu',title:'Vložit z Wordu',advice:'Do následujícího pole vložte požadovaný obsah pomocí klávesnice (<STRONG>Ctrl+V</STRONG>) a stiskněte <STRONG>OK</STRONG>.',ignoreFontFace:'Ignorovat písmo',removeStyle:'Odstranit styly'},pasteText:{button:'Vložit jako čistý text',title:'Vložit jako čistý text'},templates:{button:'Šablony',title:'Šablony obsahu',insertOption:'Nahradit aktuální obsah',selectPromptMsg:'Prosím zvolte šablonu pro otevření v editoru<br>(aktuální obsah editoru bude ztracen):',emptyListMsg:'(Není definována žádná šablona)'},showBlocks:'Ukázat bloky',stylesCombo:{label:'Styl',voiceLabel:'Styly',panelVoiceLabel:'Výběr stylu',panelTitle1:'Blokové styly',panelTitle2:'Řádkové styly',panelTitle3:'Objektové styly'},format:{label:'Formát',voiceLabel:'Formátování',panelTitle:'Formát',panelVoiceLabel:'Volba formátu odstavce',tag_p:'Normální',tag_pre:'Naformátováno',tag_address:'Adresa',tag_h1:'Nadpis 1',tag_h2:'Nadpis 2',tag_h3:'Nadpis 3',tag_h4:'Nadpis 4',tag_h5:'Nadpis 5',tag_h6:'Nadpis 6',tag_div:'Normální (DIV)'},font:{label:'Písmo',voiceLabel:'Písmo',panelTitle:'Písmo',panelVoiceLabel:'Volba písma'},fontSize:{label:'Velikost',voiceLabel:'Velikost písma',panelTitle:'Velikost',panelVoiceLabel:'Volba velikosti písma'},colorButton:{textColorTitle:'Barva textu',bgColorTitle:'Barva pozadí',auto:'Automaticky',more:'Více barev...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Kontrola pravopisu během psaní (SCAYT)',enable:'Zapnout SCAYT',disable:'Vypnout SCAYT',about:'O aplikaci SCAYT',toggle:'Vypínač SCAYT',options:'Nastavení',langs:'Jazyky',moreSuggestions:'Více návrhů',ignore:'Přeskočit',ignoreAll:'Přeskočit vše',addWord:'Přidat slovo',emptyDic:'Název slovníku nesmí být prázdný.',optionsTab:'Nastavení',languagesTab:'Jazyky',dictionariesTab:'Slovníky',aboutTab:'O aplikaci'},about:{title:'O aplikaci CKEditor',dlgTitle:'O aplikaci CKEditor',moreInfo:'Pro informace o lincenci navštivte naši webovou stránku:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximalizovat',minimize:'Minimalizovat',fakeobjects:{anchor:'Záložka',flash:'Flash animace',div:'Zalomení stránky',unknown:'Neznámý objekt'},resize:'Uchopit pro změnu velikosti',colordialog:{title:'Výběr barvy',highlight:'Zvýraznit',selected:'Vybráno',clear:'Vyčistit'}};



```
