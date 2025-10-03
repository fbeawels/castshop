# lt.js

## Review

## 1. Summary  
The file is a **CKEditor language definition** for Lithuanian (`lt`).  
It assigns a large `CKEDITOR.lang.lt` object that maps UI strings used by CKEditor plugins (toolbar buttons, dialogs, validation messages, etc.) to their Lithuanian translations.  

Key components:  

| Component | Purpose | Notes |
|-----------|---------|-------|
| `CKEDITOR.lang.lt` | Top‑level namespace that holds all translation strings for the `lt` locale | Defined as a plain JavaScript object |
| Sub‑objects (`common`, `link`, `image`, `table`, `colorButton`, …) | Group translations by plugin or dialog | Many nested objects mirror CKEditor’s UI structure |
| Translation strings | Lithuanian human‑readable labels, tooltips, error messages | Mostly direct text, but a few remain in English |

The file uses no external libraries – it is simply a data file that CKEditor loads at runtime.  

---

## 2. Detailed Description  
The file is a single JavaScript statement that **initialises** the Lithuanian language dictionary for CKEditor.  
Execution flow:

1. **Namespace creation** – The script relies on the global `CKEDITOR` object. It simply adds/overwrites the `lang.lt` property.  
2. **Object definition** – A literal object with nested objects is supplied. Each key corresponds to a UI element (button, dialog field, validation message).  
3. **Completion** – The statement ends with a semicolon; the dictionary is now available for CKEditor when the `lt` locale is selected.

Because the file contains no dynamic code, there is no runtime behaviour beyond the dictionary registration.  

### Assumptions & Constraints  
- The `CKEDITOR` global exists and is initialised before this file is executed.  
- The translations are static; no fallback logic or pluralisation handling is present in this file (CKEditor handles that elsewhere).  
- Duplicate keys in the object literals will **override** earlier entries, potentially masking intended values.  

### Design Choices  
- **Flat structure**: The object closely follows CKEditor’s existing language file template, which keeps maintenance straightforward.  
- **No functions**: All data are constants; the file is pure data.  

---

## 3. Functions/Methods  
The file contains **no executable functions or methods** – it is purely a data definition. Therefore there are no inputs, outputs, or side‑effects to describe beyond the assignment to `CKEDITOR.lang.lt`.  

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Global namespace | Provided by the CKEditor core. |
| None else | | The file is entirely self‑contained; it does not import other scripts. |

No third‑party libraries or APIs are used.

---

## 5. Additional Notes & Recommendations  

### 5.1 Duplicate / Overridden Keys  
Several duplicate keys are present, especially inside the `link` sub‑object:

```javascript
link:{ …, target:'Paskirties vieta', …, target:'Paskirties vieta', … }
```

*Result:* the first `target` entry is silently discarded, which may mask the intended value.  
The same issue occurs with `langDir`, `langCode`, etc.  

**Recommendation:**  
- Remove duplicate keys; keep only one definition per key.  
- Run a quick lint or static‑analysis pass (`eslint --no-duplicate-keys`) to catch these mistakes.

### 5.2 Typographical Errors  
- `acccessKey` is misspelled (three “c”s).  
- In the `link` object, `acccessKey` should be `accessKey`.  

**Recommendation:** Correct the spelling; otherwise the translation will never be applied.

### 5.3 Inconsistent Translations  
A handful of strings are still in English, e.g.:

```javascript
validateNumberFailed:'This value is not a number.'
```

While this may be intentional (if no Lithuanian equivalent was found), consistency is important for user experience.

### 5.4 BOM / Invisible Characters  
The file starts with a **Unicode BOM** (`﻿`). This can cause issues in some environments (e.g., when concatenating with other files or loading via XHR).  

**Recommendation:** Ensure the file is saved as UTF‑8 without BOM.

### 5.5 Maintenance & Future Enhancements  
- **Automated Extraction**: Use CKEditor’s tooling to generate language files and detect missing keys automatically.  
- **Pluralisation & Interpolation**: For more advanced locales, consider extending the structure to support plural forms (`CKEDITOR.lang.lt.plurals`) and placeholder interpolation.  
- **Unit Tests**: A simple test could load the file in a headless CKEditor instance and verify that every expected key is present and non‑empty.

### 5.6 Edge Cases  
- **Key Overwrite**: If an external plugin defines a key that duplicates one in this file, the plugin’s value will override the language string. This could unintentionally change the UI text.  
- **Missing Properties**: If a plugin expects a certain key (e.g., `link.target`) and it is missing due to a typo or deletion, CKEditor may fallback to a default English string.  

**Mitigation:** Run a diff against a baseline language file (e.g., `en`) to catch missing keys.

---

### Bottom Line  
The file is structurally correct as a data definition but contains several **duplicate keys** and **typos** that may cause UI strings to be lost or mis‑displayed. Cleaning up these issues will make the Lithuanian translation fully reliable and maintainable.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.lt={dir:'ltr',editorTitle:'Rich text editor, %1',source:'Šaltinis',newPage:'Naujas puslapis',save:'Išsaugoti',preview:'Peržiūra',cut:'Iškirpti',copy:'Kopijuoti',paste:'Įdėti',print:'Spausdinti',underline:'Pabrauktas',bold:'Pusjuodis',italic:'Kursyvas',selectAll:'Pažymėti viską',removeFormat:'Panaikinti formatą',strike:'Perbrauktas',subscript:'Apatinis indeksas',superscript:'Viršutinis indeksas',horizontalrule:'Įterpti horizontalią liniją',pagebreak:'Įterpti puslapių skirtuką',unlink:'Panaikinti nuorodą',undo:'Atšaukti',redo:'Atstatyti',common:{browseServer:'Naršyti po serverį',url:'URL',protocol:'Protokolas',upload:'Siųsti',uploadSubmit:'Siųsti į serverį',image:'Vaizdas',flash:'Flash',form:'Forma',checkbox:'Žymimasis langelis',radio:'Žymimoji akutė',textField:'Teksto laukas',textarea:'Teksto sritis',hiddenField:'Nerodomas laukas',button:'Mygtukas',select:'Atrankos laukas',imageButton:'Vaizdinis mygtukas',notSet:'<nėra nustatyta>',id:'Id',name:'Vardas',langDir:'Teksto kryptis',langDirLtr:'Iš kairės į dešinę (LTR)',langDirRtl:'Iš dešinės į kairę (RTL)',langCode:'Kalbos kodas',longDescr:'Ilgas aprašymas URL',cssClass:'Stilių lentelės klasės',advisoryTitle:'Konsultacinė antraštė',cssStyle:'Stilius',ok:'OK',cancel:'Nutraukti',generalTab:'Bendros savybės',advancedTab:'Papildomas',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'Įterpti specialų simbolį',title:'Pasirinkite specialų simbolį'},link:{toolbar:'Įterpti/taisyti nuorodą',menu:'Taisyti nuorodą',title:'Nuoroda',info:'Nuorodos informacija',target:'Paskirties vieta',upload:'Siųsti',advanced:'Papildomas',type:'Nuorodos tipas',toAnchor:'Žymė šiame puslapyje',toEmail:'El.paštas',target:'Paskirties vieta',targetNotSet:'<nėra nustatyta>',targetFrame:'<kadras>',targetPopup:'<išskleidžiamas langas>',targetNew:'Naujas langas (_blank)',targetTop:'Svarbiausias langas (_top)',targetSelf:'Tas pats langas (_self)',targetParent:'Pirminis langas (_parent)',targetFrameName:'Paskirties kadro vardas',targetPopupName:'Paskirties lango vardas',popupFeatures:'Išskleidžiamo lango savybės',popupResizable:'Resizable',popupStatusBar:'Būsenos juosta',popupLocationBar:'Adreso juosta',popupToolbar:'Mygtukų juosta',popupMenuBar:'Meniu juosta',popupFullScreen:'Visas ekranas (IE)',popupScrollBars:'Slinkties juostos',popupDependent:'Priklausomas (Netscape)',popupWidth:'Plotis',popupLeft:'Kairė pozicija',popupHeight:'Aukštis',popupTop:'Viršutinė pozicija',id:'Id',langDir:'Teksto kryptis',langDirNotSet:'<nėra nustatyta>',langDirLTR:'Iš kairės į dešinę (LTR)',langDirRTL:'Iš dešinės į kairę (RTL)',acccessKey:'Prieigos raktas',name:'Vardas',langCode:'Teksto kryptis',tabIndex:'Tabuliavimo indeksas',advisoryTitle:'Konsultacinė antraštė',advisoryContentType:'Konsultacinio turinio tipas',cssClasses:'Stilių lentelės klasės',charset:'Susietų išteklių simbolių lentelė',styles:'Stilius',selectAnchor:'Pasirinkite žymę',anchorName:'Pagal žymės vardą',anchorId:'Pagal žymės Id',emailAddress:'El.pašto adresas',emailSubject:'Žinutės tema',emailBody:'Žinutės turinys',noAnchors:'(Šiame dokumente žymių nėra)',noUrl:'Prašome įvesti nuorodos URL',noEmail:'Prašome įvesti el.pašto adresą'},anchor:{toolbar:'Įterpti/modifikuoti žymę',menu:'Žymės savybės',title:'Žymės savybės',name:'Žymės vardas',errorName:'Prašome įvesti žymės vardą'},findAndReplace:{title:'Surasti ir pakeisti',find:'Rasti',replace:'Pakeisti',findWhat:'Surasti tekstą:',replaceWith:'Pakeisti tekstu:',notFoundMsg:'Nurodytas tekstas nerastas.',matchCase:'Skirti didžiąsias ir mažąsias raides',matchWord:'Atitikti pilną žodį',matchCyclic:'Match cyclic',replaceAll:'Pakeisti viską',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'Lentelė',title:'Lentelės savybės',menu:'Lentelės savybės',deleteTable:'Šalinti lentelę',rows:'Eilutės',columns:'Stulpeliai',border:'Rėmelio dydis',align:'Lygiuoti',alignNotSet:'<Nenustatyta>',alignLeft:'Kairę',alignCenter:'Centrą',alignRight:'Dešinę',width:'Plotis',widthPx:'taškais',widthPc:'procentais',height:'Aukštis',cellSpace:'Tarpas tarp langelių',cellPad:'Trapas nuo langelio rėmo iki teksto',caption:'Antraštė',summary:'Santrauka',headers:'Antraštės',headersNone:'Nėra',headersColumn:'Pirmas stulpelis',headersRow:'Pirma eilutė',headersBoth:'Abu',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'Langelis',insertBefore:'Įterpti langelį prieš',insertAfter:'Įterpti langelį po',deleteCell:'Šalinti langelius',merge:'Sujungti langelius',mergeRight:'Sujungti su dešine',mergeDown:'Sujungti su apačia',splitHorizontal:'Skaidyti langelį horizontaliai',splitVertical:'Skaidyti langelį vertikaliai',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'Eilutė',insertBefore:'Įterpti eilutę prieš',insertAfter:'Įterpti eilutę po',deleteRow:'Šalinti eilutes'},column:{menu:'Stulpelis',insertBefore:'Įterpti stulpelį prieš',insertAfter:'Įterpti stulpelį po',deleteColumn:'Šalinti stulpelius'}},button:{title:'Mygtuko savybės',text:'Tekstas (Reikšmė)',type:'Tipas',typeBtn:'Mygtukas',typeSbm:'Siųsti',typeRst:'Išvalyti'},checkboxAndRadio:{checkboxTitle:'Žymimojo langelio savybės',radioTitle:'Žymimosios akutės savybės',value:'Reikšmė',selected:'Pažymėtas'},form:{title:'Formos savybės',menu:'Formos savybės',action:'Veiksmas',method:'Metodas',encoding:'Encoding',target:'Paskirties vieta',targetNotSet:'<nėra nustatyta>',targetNew:'Naujas langas (_blank)',targetTop:'Svarbiausias langas (_top)',targetSelf:'Tas pats langas (_self)',targetParent:'Pirminis langas (_parent)'},select:{title:'Atrankos lauko savybės',selectInfo:'Informacija',opAvail:'Galimos parinktys',value:'Reikšmė',size:'Dydis',lines:'eilučių',chkMulti:'Leisti daugeriopą atranką',opText:'Tekstas',opValue:'Reikšmė',btnAdd:'Įtraukti',btnModify:'Modifikuoti',btnUp:'Aukštyn',btnDown:'Žemyn',btnSetValue:'Laikyti pažymėta reikšme',btnDelete:'Trinti'},textarea:{title:'Teksto srities savybės',cols:'Ilgis',rows:'Plotis'},textfield:{title:'Teksto lauko savybės',name:'Vardas',value:'Reikšmė',charWidth:'Ilgis simboliais',maxChars:'Maksimalus simbolių skaičius',type:'Tipas',typeText:'Tekstas',typePass:'Slaptažodis'},hidden:{title:'Nerodomo lauko savybės',name:'Vardas',value:'Reikšmė'},image:{title:'Vaizdo savybės',titleButton:'Vaizdinio mygtuko savybės',menu:'Vaizdo savybės',infoTab:'Vaizdo informacija',btnUpload:'Siųsti į serverį',url:'URL',upload:'Nusiųsti',alt:'Alternatyvus Tekstas',width:'Plotis',height:'Aukštis',lockRatio:'Išlaikyti proporciją',resetSize:'Atstatyti dydį',border:'Rėmelis',hSpace:'Hor.Erdvė',vSpace:'Vert.Erdvė',align:'Lygiuoti',alignLeft:'Kairę',alignAbsBottom:'Absoliučią apačią',alignAbsMiddle:'Absoliutų vidurį',alignBaseline:'Apatinę liniją',alignBottom:'Apačią',alignMiddle:'Vidurį',alignRight:'Dešinę',alignTextTop:'Teksto viršūnę',alignTop:'Viršūnę',preview:'Peržiūra',alertUrl:'Prašome įvesti vaizdo URL',linkTab:'Nuoroda',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'Flash savybės',propertiesTab:'Properties',title:'Flash savybės',chkPlay:'Automatinis paleidimas',chkLoop:'Ciklas',chkMenu:'Leisti Flash meniu',chkFull:'Allow Fullscreen',scale:'Mastelis',scaleAll:'Rodyti visą',scaleNoBorder:'Be rėmelio',scaleFit:'Tikslus atitikimas',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'Lygiuoti',alignLeft:'Kairę',alignAbsBottom:'Absoliučią apačią',alignAbsMiddle:'Absoliutų vidurį',alignBaseline:'Apatinę liniją',alignBottom:'Apačią',alignMiddle:'Vidurį',alignRight:'Dešinę',alignTextTop:'Teksto viršūnę',alignTop:'Viršūnę',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'Fono spalva',width:'Plotis',height:'Aukštis',hSpace:'Hor.Erdvė',vSpace:'Vert.Erdvė',validateSrc:'Prašome įvesti nuorodos URL',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'Rašybos tikrinimas',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'Žodyne nerastas',changeTo:'Pakeisti į',btnIgnore:'Ignoruoti',btnIgnoreAll:'Ignoruoti visus',btnReplace:'Pakeisti',btnReplaceAll:'Pakeisti visus',btnUndo:'Atšaukti',noSuggestions:'- Nėra pasiūlymų -',progress:'Vyksta rašybos tikrinimas...',noMispell:'Rašybos tikrinimas baigtas: Nerasta rašybos klaidų',noChanges:'Rašybos tikrinimas baigtas: Nėra pakeistų žodžių',oneChange:'Rašybos tikrinimas baigtas: Vienas žodis pakeistas',manyChanges:'Rašybos tikrinimas baigtas: Pakeista %1 žodžių',ieSpellDownload:'Rašybos tikrinimas neinstaliuotas. Ar Jūs norite jį dabar atsisiųsti?'},smiley:{toolbar:'Veideliai',title:'Įterpti veidelį'},elementsPath:{eleTitle:'%1 element'},numberedlist:'Numeruotas sąrašas',bulletedlist:'Suženklintas sąrašas',indent:'Padidinti įtrauką',outdent:'Sumažinti įtrauką',justify:{left:'Lygiuoti kairę',center:'Centruoti',right:'Lygiuoti dešinę',block:'Lygiuoti abi puses'},blockquote:'Citata',clipboard:{title:'Įdėti',cutError:'Jūsų naršyklės saugumo nustatymai neleidžia redaktoriui automatiškai įvykdyti iškirpimo operacijų. Tam prašome naudoti klaviatūrą (Ctrl+X).',copyError:'Jūsų naršyklės saugumo nustatymai neleidžia redaktoriui automatiškai įvykdyti kopijavimo operacijų. Tam prašome naudoti klaviatūrą (Ctrl+C).',pasteMsg:'Žemiau esančiame įvedimo lauke įdėkite tekstą, naudodami klaviatūrą (<STRONG>Ctrl+V</STRONG>) ir paspauskite mygtuką <STRONG>OK</STRONG>.',securityMsg:'Dėl jūsų naršyklės saugumo nustatymų, redaktorius negali tiesiogiai pasiekti laikinosios atminties. Jums reikia nukopijuoti dar kartą į šį langą.'},pastefromword:{toolbar:'Įdėti iš Word',title:'Įdėti iš Word',advice:'Žemiau esančiame įvedimo lauke įdėkite tekstą, naudodami klaviatūrą (<STRONG>Ctrl+V</STRONG>) ir paspauskite mygtuką <STRONG>OK</STRONG>.',ignoreFontFace:'Ignoruoti šriftų nustatymus',removeStyle:'Pašalinti stilių nustatymus'},pasteText:{button:'Įdėti kaip gryną tekstą',title:'Įdėti kaip gryną tekstą'},templates:{button:'Šablonai',title:'Turinio šablonai',insertOption:'Pakeisti dabartinį turinį pasirinktu šablonu',selectPromptMsg:'Pasirinkite norimą šabloną<br>(<b>Dėmesio!</b> esamas turinys bus prarastas):',emptyListMsg:'(Šablonų sąrašas tuščias)'},showBlocks:'Rodyti blokus',stylesCombo:{label:'Stilius',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'Šrifto formatas',voiceLabel:'Format',panelTitle:'Šrifto formatas',panelVoiceLabel:'Select a paragraph format',tag_p:'Normalus',tag_pre:'Formuotas',tag_address:'Kreipinio',tag_h1:'Antraštinis 1',tag_h2:'Antraštinis 2',tag_h3:'Antraštinis 3',tag_h4:'Antraštinis 4',tag_h5:'Antraštinis 5',tag_h6:'Antraštinis 6',tag_div:'Normal (DIV)'},font:{label:'Šriftas',voiceLabel:'Font',panelTitle:'Šriftas',panelVoiceLabel:'Select a font'},fontSize:{label:'Šrifto dydis',voiceLabel:'Font Size',panelTitle:'Šrifto dydis',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'Teksto spalva',bgColorTitle:'Fono spalva',auto:'Automatinis',more:'Daugiau spalvų...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
