# eu.js

## Review

## 1. Summary  

This file is a **CKEditor language definition** for the Basque (eu) locale.  
It defines a single global object `CKEDITOR.lang.eu` that contains all user‑facing strings used by CKEditor’s UI, plugins, dialogs, and validation messages.  

* **Purpose** – Provide localized text for the editor’s menus, toolbars, dialog boxes, tooltips, and error messages.  
* **Structure** – A single nested object; each top‑level property represents a CKEditor component or dialog (e.g., `common`, `link`, `image`, `table`, `findAndReplace`, `spellCheck`, etc.).  
* **Design patterns** – The file follows CKEditor’s convention of declaring a language bundle as a plain JavaScript object. No frameworks or libraries are required; the code is pure ES3‑compatible JS.  

## 2. Detailed Description  

### Core components  

| Component | Role |
|-----------|------|
| `CKEDITOR.lang.eu` | Root namespace for all Basque translations. |
| `dir` | Text direction (`ltr`). |
| `editorTitle` | Tooltip for the editor frame. |
| `source`, `newPage`, `save`, … | Generic UI strings used throughout CKEditor. |
| `common` | Strings shared by many dialogs (e.g., browse, upload, submit). |
| `specialChar`, `link`, `anchor`, `findAndReplace` … | Sub‑objects that group strings belonging to specific plugins or dialogs. |

### Execution flow  

1. **Load** – The script is loaded by the CKEditor core (via `<script src="lang/eu.js"></script>`).  
2. **Initialization** – CKEditor registers the object automatically when it parses the global `CKEDITOR.lang` map.  
3. **Runtime** – Whenever a UI element is rendered, CKEditor fetches the string from `CKEDITOR.lang.eu` using the key path (e.g., `link.title`, `table.toolbar`).  
4. **Cleanup** – No runtime cleanup; the object remains in memory for the life of the page.

### Assumptions & constraints  

* **Encoding** – All non‑ASCII characters are encoded directly; the file must be UTF‑8 without BOM (although the leading invisible character “﻿” is present and could be problematic).  
* **Placeholder syntax** – Uses `%1`, `%2` style placeholders. Code that consumes these strings must perform the substitution.  
* **Naming consistency** – The CKEditor core expects the language code (`eu`) to match the file name.  
* **No dynamic content** – The file contains static strings; no functions or logic are executed.

## 3. Functions/Methods  

The file contains **no executable functions**; it is purely a data definition.  
The only side effect is the global assignment `CKEDITOR.lang.eu = {...}` which makes the translations available to the CKEditor core.

### Notable entries  

* `common.{browseServer, url, protocol, upload, ...}` – Generic dialog elements.  
* `link.{toolbar, title, ...}` – All link‑related UI strings.  
* `image.{title, ...}` – Image dialog strings.  
* `flash.{properties, title, ...}` – Flash plugin strings.  
* `spellCheck.{toolbar, title, ...}` – SCAYT/spell‑check UI strings.  
* `templates.{button, title, ...}` – Template dialog strings.  
* `scayt.{title, enable, disable, ...}` – SCAYT configuration strings.  

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party library | The object is a CKEditor language bundle. |
| `CKEDITOR.lang` | Internal CKEditor property | The script relies on the CKEditor core creating `CKEDITOR.lang` before this file executes. |
| None other | | The file is pure JS and does not import external libraries. |

The file is **platform‑agnostic**; it can be used in browsers, Node‑based build systems, or any environment that loads CKEditor.

## 5. Additional Notes  

### Duplicate / inconsistent keys  

* **`link` section** – `target` is declared twice (once with value “Target (Helburua)” and again with the same value). This redundancy can be removed without changing behavior.  
* **`link` section** – The key `acccessKey` is misspelled; it should be `accessKey`. CKEditor will not find the correct translation for “Sarbide‑gakoa”.  
* **`image` section** – `alertUrl` and `urlMissing` appear duplicated in semantics; consolidating may reduce confusion.  

### Encoding / invisible characters  

* The very first character in the file (`﻿`) is a **Byte‑Order Mark (BOM)** or a hidden zero‑width character. While browsers typically ignore it, it can cause problems in some build pipelines or when concatenated with other scripts. It is recommended to remove it and ensure the file is saved as **UTF‑8 without BOM**.

### Placeholder usage  

* All placeholders use the `%1`/`%2` style. Verify that the CKEditor core consistently performs the substitution; any mismatch (e.g., missing `%1`) may produce raw placeholders in the UI.

### Potential improvements  

1. **Split into plugin files** – CKEditor already supports separate language files per plugin. If maintenance becomes an issue, consider moving each sub‑object (`link`, `image`, `table`, etc.) into its own file and loading only the needed ones.  
2. **Automated linting** – Use a JSON/YAML linter or a custom script to detect duplicate keys, typos (like `acccessKey`), and missing placeholders.  
3. **Unicode normalization** – Ensure all characters are in a consistent Unicode form (NFC) to avoid display differences across browsers.  

### Edge cases  

* **Non‑existent placeholders** – If a translation string contains a placeholder that the calling code does not replace, the placeholder will be shown verbatim (e.g., “%1”).  
* **Missing translations** – If a key is missing from this file but referenced by CKEditor, the UI will fall back to the English string, which may break layout or accessibility expectations.  

### Future enhancements  

* **Dynamic fallback** – Implement a fallback mechanism that automatically uses English strings when a key is missing.  
* **Internationalization API** – Replace static objects with a small helper that loads language data on demand (lazy loading).  
* **Unit tests** – Write tests that confirm each key is present and that placeholder counts match the expected number.  

Overall, the file is structurally sound for CKEditor’s needs but would benefit from a few cleanup steps (duplicate keys, typo corrections, BOM removal) to improve maintainability and reliability.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.eu={dir:'ltr',editorTitle:'Testu aberastuentzako editorea, %1',source:'HTML Iturburua',newPage:'Orrialde Berria',save:'Gorde',preview:'Aurrebista',cut:'Ebaki',copy:'Kopiatu',paste:'Itsatsi',print:'Inprimatu',underline:'Azpimarratu',bold:'Lodia',italic:'Etzana',selectAll:'Hautatu dena',removeFormat:'Kendu Formatua',strike:'Marratua',subscript:'Azpi-indize',superscript:'Goi-indize',horizontalrule:'Txertatu Marra Horizontala',pagebreak:'Txertatu Orrialde-jauzia',unlink:'Kendu Esteka',undo:'Desegin',redo:'Berregin',common:{browseServer:'Zerbitzaria arakatu',url:'URL',protocol:'Protokoloa',upload:'Gora kargatu',uploadSubmit:'Zerbitzarira bidalia',image:'Irudia',flash:'Flasha',form:'Formularioa',checkbox:'Kontrol-laukia',radio:'Aukera-botoia',textField:'Testu Eremua',textarea:'Testu-area',hiddenField:'Ezkutuko Eremua',button:'Botoia',select:'Hautespen Eremua',imageButton:'Irudi Botoia',notSet:'<Ezarri gabe>',id:'Id',name:'Izena',langDir:'Hizkuntzaren Norabidea',langDirLtr:'Ezkerretik Eskumara(LTR)',langDirRtl:'Eskumatik Ezkerrera (RTL)',langCode:'Hizkuntza Kodea',longDescr:'URL Deskribapen Luzea',cssClass:'Estilo-orriko Klaseak',advisoryTitle:'Izenburua',cssStyle:'Estiloa',ok:'Ados',cancel:'Utzi',generalTab:'Orokorra',advancedTab:'Aurreratua',validateNumberFailed:'Balio hau ez da zenbaki bat.',confirmNewPage:'Eduki honetan gorde gabe dauden aldaketak galduko dira. Ziur zaude orri berri bat kargatu nahi duzula?',confirmCancel:'Aukera batzuk aldatu egin dira. Ziur zaude elkarrizketa-koadroa itxi nahi duzula?',unavailable:'%1<span class="cke_accessibility">, erabilezina</span>'},specialChar:{toolbar:'Txertatu Karaktere Berezia',title:'Karaktere Berezia Aukeratu'},link:{toolbar:'Txertatu/Editatu Esteka',menu:'Aldatu Esteka',title:'Esteka',info:'Estekaren Informazioa',target:'Target (Helburua)',upload:'Gora kargatu',advanced:'Aurreratua',type:'Esteka Mota',toAnchor:'Aingura orrialde honetan',toEmail:'ePosta',target:'Target (Helburua)',targetNotSet:'<Ezarri gabe>',targetFrame:'<marko>',targetPopup:'<popup leihoa>',targetNew:'Leiho Berria (_blank)',targetTop:'Goiko Leihoa (_top)',targetSelf:'Leiho Berdina (_self)',targetParent:'Leiho Gurasoa (_parent)',targetFrameName:'Marko Helburuaren Izena',targetPopupName:'Popup Leihoaren Izena',popupFeatures:'Popup Leihoaren Ezaugarriak',popupResizable:'Tamaina Aldakorra',popupStatusBar:'Egoera Barra',popupLocationBar:'Kokaleku Barra',popupToolbar:'Tresna Barra',popupMenuBar:'Menu Barra',popupFullScreen:'Pantaila Osoa (IE)',popupScrollBars:'Korritze Barrak',popupDependent:'Menpekoa (Netscape)',popupWidth:'Zabalera',popupLeft:'Ezkerreko  Posizioa',popupHeight:'Altuera',popupTop:'Goiko Posizioa',id:'Id',langDir:'Hizkuntzaren Norabidea',langDirNotSet:'<Ezarri gabe>',langDirLTR:'Ezkerretik Eskumara(LTR)',langDirRTL:'Eskumatik Ezkerrera (RTL)',acccessKey:'Sarbide-gakoa',name:'Izena',langCode:'Hizkuntzaren Norabidea',tabIndex:'Tabulazio Indizea',advisoryTitle:'Izenburua',advisoryContentType:'Eduki Mota (Content Type)',cssClasses:'Estilo-orriko Klaseak',charset:'Estekatutako Karaktere Multzoa',styles:'Estiloa',selectAnchor:'Aingura bat hautatu',anchorName:'Aingura izenagatik',anchorId:'Elementuaren ID-gatik',emailAddress:'ePosta Helbidea',emailSubject:'Mezuaren Gaia',emailBody:'Mezuaren Gorputza',noAnchors:'(Ez daude aingurak eskuragarri dokumentuan)',noUrl:'Mesedez URL esteka idatzi',noEmail:'Mesedez ePosta helbidea idatzi'},anchor:{toolbar:'Aingura',menu:'Ainguraren Ezaugarriak',title:'Ainguraren Ezaugarriak',name:'Ainguraren Izena',errorName:'Idatzi ainguraren izena'},findAndReplace:{title:'Bilatu eta Ordeztu',find:'Bilatu',replace:'Ordezkatu',findWhat:'Zer bilatu:',replaceWith:'Zerekin ordeztu:',notFoundMsg:'Idatzitako testua ez da topatu.',matchCase:'Maiuskula/minuskula',matchWord:'Esaldi osoa bilatu',matchCyclic:'Bilaketa ziklikoa',replaceAll:'Ordeztu Guztiak',replaceSuccessMsg:'Zenbat aldiz ordeztua: %1'},table:{toolbar:'Taula',title:'Taularen Ezaugarriak',menu:'Taularen Ezaugarriak',deleteTable:'Ezabatu Taula',rows:'Lerroak',columns:'Zutabeak',border:'Ertzaren Zabalera',align:'Lerrokatu',alignNotSet:'<Ezarri gabe>',alignLeft:'Ezkerrean',alignCenter:'Erdian',alignRight:'Eskuman',width:'Zabalera',widthPx:'pixel',widthPc:'ehuneko',height:'Altuera',cellSpace:'Gelaxka arteko tartea',cellPad:'Gelaxken betegarria',caption:'Epigrafea',summary:'Laburpena',headers:'Goiburuak',headersNone:'Bat ere ez',headersColumn:'Lehen zutabea',headersRow:'Lehen lerroa',headersBoth:'Biak',invalidRows:'Lerro kopurua 0 baino handiagoa den zenbakia izan behar da.',invalidCols:'Zutabe kopurua 0 baino handiagoa den zenbakia izan behar da.',invalidBorder:'Ertzaren tamaina zenbaki bat izan behar da.',invalidWidth:'Taularen zabalera zenbaki bat izan behar da.',invalidHeight:'Taularen altuera zenbaki bat izan behar da.',invalidCellSpacing:'Gelaxka arteko tartea zenbaki bat izan behar da.',invalidCellPadding:'Gelaxken betegarria zenbaki bat izan behar da.',cell:{menu:'Gelaxka',insertBefore:'Txertatu Gelaxka Aurretik',insertAfter:'Txertatu Gelaxka Ostean',deleteCell:'Kendu Gelaxkak',merge:'Batu Gelaxkak',mergeRight:'Elkartu Eskumara',mergeDown:'Elkartu Behera',splitHorizontal:'Banatu Gelaxkak Horizontalki',splitVertical:'Banatu Gelaxkak Bertikalki',title:'Gelaxken Ezaugarriak',cellType:'Gelaxka Mota',rowSpan:'Hedatutako Lerroak',colSpan:'Hedatutako Zutabeak',wordWrap:'Itzulbira',hAlign:'Lerrokatze Horizontala',vAlign:'Lerrokatze Bertikala',alignTop:'Goian',alignMiddle:'Erdian',alignBottom:'Behean',alignBaseline:'Oinarri-lerroan',bgColor:'Fondoaren Kolorea',borderColor:'Ertzaren Kolorea',data:'Data',header:'Goiburua',yes:'Bai',no:'Ez',invalidWidth:'Gelaxkaren zabalera zenbaki bat izan behar da.',invalidHeight:'Gelaxkaren altuera zenbaki bat izan behar da.',invalidRowSpan:'Lerroen hedapena zenbaki osoa izan behar da.',invalidColSpan:'Zutabeen hedapena zenbaki osoa izan behar da.',chooseColor:'Choose'},row:{menu:'Lerroa',insertBefore:'Txertatu Lerroa Aurretik',insertAfter:'Txertatu Lerroa Ostean',deleteRow:'Ezabatu Lerroak'},column:{menu:'Zutabea',insertBefore:'Txertatu Zutabea Aurretik',insertAfter:'Txertatu Zutabea Ostean',deleteColumn:'Ezabatu Zutabeak'}},button:{title:'Botoiaren Ezaugarriak',text:'Testua (Balorea)',type:'Mota',typeBtn:'Botoia',typeSbm:'Bidali',typeRst:'Garbitu'},checkboxAndRadio:{checkboxTitle:'Kontrol-laukiko Ezaugarriak',radioTitle:'Aukera-botoiaren Ezaugarriak',value:'Balorea',selected:'Hautatuta'},form:{title:'Formularioaren Ezaugarriak',menu:'Formularioaren Ezaugarriak',action:'Ekintza',method:'Metodoa',encoding:'Kodeketa',target:'Target (Helburua)',targetNotSet:'<Ezarri gabe>',targetNew:'Leiho Berria (_blank)',targetTop:'Goiko Leihoa (_top)',targetSelf:'Leiho Berdina (_self)',targetParent:'Leiho Gurasoa (_parent)'},select:{title:'Hautespen Eremuaren Ezaugarriak',selectInfo:'Informazioa',opAvail:'Aukera Eskuragarriak',value:'Balorea',size:'Tamaina',lines:'lerro kopurura',chkMulti:'Hautaketa anitzak baimendu',opText:'Testua',opValue:'Balorea',btnAdd:'Gehitu',btnModify:'Aldatu',btnUp:'Gora',btnDown:'Behera',btnSetValue:'Aukeratutako balorea ezarri',btnDelete:'Ezabatu'},textarea:{title:'Testu-arearen Ezaugarriak',cols:'Zutabeak',rows:'Lerroak'},textfield:{title:'Testu Eremuaren Ezaugarriak',name:'Izena',value:'Balorea',charWidth:'Zabalera',maxChars:'Zenbat karaktere gehienez',type:'Mota',typeText:'Testua',typePass:'Pasahitza'},hidden:{title:'Ezkutuko Eremuaren Ezaugarriak',name:'Izena',value:'Balorea'},image:{title:'Irudi Ezaugarriak',titleButton:'Irudi Botoiaren Ezaugarriak',menu:'Irudi Ezaugarriak',infoTab:'Irudi informazioa',btnUpload:'Zerbitzarira bidalia',url:'URL',upload:'Gora Kargatu',alt:'Ordezko Testua',width:'Zabalera',height:'Altuera',lockRatio:'Erlazioa Blokeatu',resetSize:'Tamaina Berrezarri',border:'Ertza',hSpace:'HSpace',vSpace:'VSpace',align:'Lerrokatu',alignLeft:'Ezkerrera',alignAbsBottom:'Abs Behean',alignAbsMiddle:'Abs Erdian',alignBaseline:'Oinan',alignBottom:'Behean',alignMiddle:'Erdian',alignRight:'Eskuman',alignTextTop:'Testua Goian',alignTop:'Goian',preview:'Aurrebista',alertUrl:'Mesedez Irudiaren URLa idatzi',linkTab:'Esteka',button2Img:'Aukeratutako irudi botoia, irudi normal batean eraldatu nahi duzu?',img2Button:'Aukeratutako irudia, irudi botoi batean eraldatu nahi duzu?',urlMissing:'Image source URL is missing.'},flash:{properties:'Flasharen Ezaugarriak',propertiesTab:'Ezaugarriak',title:'Flasharen Ezaugarriak',chkPlay:'Automatikoki Erreproduzitu',chkLoop:'Begizta',chkMenu:'Flasharen Menua Gaitu',chkFull:'Onartu Pantaila osoa',scale:'Eskalatu',scaleAll:'Dena erakutsi',scaleNoBorder:'Ertzik gabe',scaleFit:'Doitu',access:'Scriptak baimendu',accessAlways:'Beti',accessSameDomain:'Domeinu berdinekoak',accessNever:'Inoiz ere ez',align:'Lerrokatu',alignLeft:'Ezkerrera',alignAbsBottom:'Abs Behean',alignAbsMiddle:'Abs Erdian',alignBaseline:'Oinan',alignBottom:'Behean',alignMiddle:'Erdian',alignRight:'Eskuman',alignTextTop:'Testua Goian',alignTop:'Goian',quality:'Kalitatea',qualityBest:'Hoberena',qualityHigh:'Altua',qualityAutoHigh:'Auto Altua',qualityMedium:'Ertaina',qualityAutoLow:'Auto Baxua',qualityLow:'Baxua',windowModeWindow:'Leihoa',windowModeOpaque:'Opakoa',windowModeTransparent:'Gardena',windowMode:'Leihoaren modua',flashvars:'Flash Aldagaiak',bgcolor:'Atzeko kolorea',width:'Zabalera',height:'Altuera',hSpace:'HSpace',vSpace:'VSpace',validateSrc:'Mesedez URL esteka idatzi',validateWidth:'Zabalera zenbaki bat izan behar da.',validateHeight:'Altuera zenbaki bat izan behar da.',validateHSpace:'HSpace zenbaki bat izan behar da.',validateVSpace:'VSpace zenbaki bat izan behar da.'},spellCheck:{toolbar:'Ortografia',title:'Ortografia zuzenketa',notAvailable:'Barkatu baina momentu honetan zerbitzua ez dago erabilgarri.',errorLoading:'Errorea gertatu da aplikazioa zerbitzaritik kargatzean: %s.',notInDic:'Ez dago hiztegian',changeTo:'Honekin ordezkatu',btnIgnore:'Ezikusi',btnIgnoreAll:'Denak Ezikusi',btnReplace:'Ordezkatu',btnReplaceAll:'Denak Ordezkatu',btnUndo:'Desegin',noSuggestions:'- Iradokizunik ez -',progress:'Zuzenketa ortografikoa martxan...',noMispell:'Zuzenketa ortografikoa bukatuta: Akatsik ez',noChanges:'Zuzenketa ortografikoa bukatuta: Ez da ezer aldatu',oneChange:'Zuzenketa ortografikoa bukatuta: Hitz bat aldatu da',manyChanges:'Zuzenketa ortografikoa bukatuta: %1 hitz aldatu dira',ieSpellDownload:'Zuzentzaile ortografikoa ez dago instalatuta. Deskargatu nahi duzu?'},smiley:{toolbar:'Aurpegierak',title:'Aurpegiera Sartu'},elementsPath:{eleTitle:'%1 elementua'},numberedlist:'Zenbakidun Zerrenda',bulletedlist:'Buletdun Zerrenda',indent:'Handitu Koska',outdent:'Txikitu Koska',justify:{left:'Lerrokatu Ezkerrean',center:'Lerrokatu Erdian',right:'Lerrokatu Eskuman',block:'Justifikatu'},blockquote:'Aipamen blokea',clipboard:{title:'Itsatsi',cutError:'Zure web nabigatzailearen segurtasun ezarpenak testuak automatikoki moztea ez dute baimentzen. Mesedez teklatua erabili ezazu (Ctrl+X).',copyError:'Zure web nabigatzailearen segurtasun ezarpenak testuak automatikoki kopiatzea ez dute baimentzen. Mesedez teklatua erabili ezazu (Ctrl+C).',pasteMsg:'Mesedez teklatua erabilita (<STRONG>Ctrl+V</STRONG>) ondorego eremuan testua itsatsi eta <STRONG>OK</STRONG> sakatu.',securityMsg:'Nabigatzailearen segurtasun ezarpenak direla eta, editoreak ezin du arbela zuzenean erabili. Leiho honetan berriro itsatsi behar duzu.'},pastefromword:{toolbar:'Itsatsi Word-etik',title:'Itsatsi Word-etik',advice:'Mesedez teklatua erabilita (<STRONG>Ctrl+V</STRONG>) ondorego eremuan testua itsatsi eta <STRONG>OK</STRONG> sakatu.',ignoreFontFace:'Letra Motaren definizioa ezikusi',removeStyle:'Estilo definizioak kendu'},pasteText:{button:'Testu Arrunta bezala Itsatsi',title:'Testu Arrunta bezala Itsatsi'},templates:{button:'Txantiloiak',title:'Eduki Txantiloiak',insertOption:'Ordeztu oraingo edukiak',selectPromptMsg:'Mesedez txantiloia aukeratu editorean kargatzeko<br>(orain dauden edukiak galduko dira):',emptyListMsg:'(Ez dago definitutako txantiloirik)'},showBlocks:'Blokeak erakutsi',stylesCombo:{label:'Estiloa',voiceLabel:'Estiloak',panelVoiceLabel:'Estilo bat aukeratu',panelTitle1:'Bloke Estiloak',panelTitle2:'Inline Estiloak',panelTitle3:'Objektu Estiloak'},format:{label:'Formatua',voiceLabel:'Formatua',panelTitle:'Formatua',panelVoiceLabel:'Aukeratu paragrafo formatu bat',tag_p:'Arrunta',tag_pre:'Formateatua',tag_address:'Helbidea',tag_h1:'Izenburua 1',tag_h2:'Izenburua 2',tag_h3:'Izenburua 3',tag_h4:'Izenburua 4',tag_h5:'Izenburua 5',tag_h6:'Izenburua 6',tag_div:'Paragrafoa (DIV)'},font:{label:'Letra-tipoa',voiceLabel:'Letra-tipoa',panelTitle:'Letra-tipoa',panelVoiceLabel:'Aukeratu letra-tipoa'},fontSize:{label:'Tamaina',voiceLabel:'Tamaina',panelTitle:'Tamaina',panelVoiceLabel:'Aukeratu letraren tamaina'},colorButton:{textColorTitle:'Testu Kolorea',bgColorTitle:'Atzeko kolorea',auto:'Automatikoa',more:'Kolore gehiago...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Ortografia Zuzenketa Idatzi Ahala (SCAYT)',enable:'Gaitu SCAYT',disable:'Desgaitu SCAYT',about:'SCAYTi buruz',toggle:'SCAYT aldatu',options:'Aukerak',langs:'Hizkuntzak',moreSuggestions:'Iradokizun gehiago',ignore:'Baztertu',ignoreAll:'Denak baztertu',addWord:'Hitza Gehitu',emptyDic:'Hiztegiaren izena ezin da hutsik egon.',optionsTab:'Aukerak',languagesTab:'Hizkuntzak',dictionariesTab:'Hiztegiak',aboutTab:'Honi buruz'},about:{title:'CKEditor(r)i buruz',dlgTitle:'CKEditor(r)i buruz',moreInfo:'Lizentziari buruzko informazioa gure webgunean:',copy:'Copyright &copy; $1. Eskubide guztiak erreserbaturik.'},maximize:'Maximizatu',minimize:'Minimize',fakeobjects:{anchor:'Aingura',flash:'Flash Animazioa',div:'Orrialde Saltoa',unknown:'Objektu ezezaguna'},resize:'Arrastatu tamaina aldatzeko',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
