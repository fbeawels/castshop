# et.js

## Review

## 1. Summary  

The file defines the Estonian (`et`) language pack for CKEditor. It is a pure data module – a single JavaScript object (`CKEDITOR.lang.et`) that maps UI strings, dialog titles, validation messages, and other textual resources to their Estonian translations.  

### Key components  
| Component | Purpose |
|-----------|---------|
| `dir` | Text direction (`ltr`). |
| `editorTitle` | Title for the editor window. |
| `source` | Label for the source‑code view. |
| `common` | Frequently used UI strings (buttons, dialogs, file browser labels). |
| `link`, `image`, `flash`, `table`, `button`, … | Dialog definitions for specific plugin UI. |
| `colorButton`, `colors` | Color picker labels and palette. |
| `about`, `maximize`, `minimize` | General UI actions. |

The file follows the structure required by CKEditor’s i18n system and is automatically loaded by the core when the `et` locale is requested.

## 2. Detailed Description  

### Initialization  
When CKEditor starts, it loads all language files present in the `lang/` directory.  
1. The core script resolves the desired locale (e.g., `et`).  
2. It executes this file, which assigns a new property to `CKEDITOR.lang`.  
3. The editor instance reads the translations via `CKEDITOR.lang[locale]` during UI construction.  

### Runtime behavior  
The translations are static – no runtime computation is performed. They are simply read by the UI rendering functions.  
- **Dialog construction** – When a dialog (e.g., `link`, `image`) is opened, the dialog factory reads `CKEDITOR.lang[locale].link` etc.  
- **Toolbar & button labels** – CKEditor looks up the label from the appropriate object (`common` or specific plugin).  
- **Validation** – The editor displays error messages from this file (e.g., `validateNumberFailed`).  

### Cleanup  
No special cleanup is required; the object remains in memory for the life of the page.

### Assumptions & constraints  
- The file assumes that the global `CKEDITOR` object already exists.  
- It uses the Estonian locale code `et`.  
- All string keys must match those expected by the core; a typo will result in missing UI text.  

### Architecture & design choices  
- **Flat object hierarchy**: The structure mirrors CKEditor’s own language definition conventions.  
- **Centralized translation strings**: Keeps all Estonian translations in one place for easy maintenance.  
- **No executable code**: Increases safety (no accidental side‑effects) and simplifies loading.

## 3. Functions/Methods  

The file contains no functions or executable methods; it is a data module.  
However, the CKEditor core will **consume** these properties via simple property access, e.g., `CKEDITOR.lang.et.common.ok`.  

### Notable entries (for reference)  

| Section | Key | Value |
|---------|-----|-------|
| `common` | `ok` | `'OK'` |
| `link` | `info` | `'Link info'` |
| `image` | `urlMissing` | `'Image source URL is missing.'` |
| `colors` | `'000'` | `'Black'` |
| `about` | `copy` | `'Copyright &copy; $1. All rights reserved.'` |

These entries are read by CKEditor’s UI builders and validation routines.

## 4. Dependencies  

| Library / Framework | Nature | Remarks |
|---------------------|--------|---------|
| **CKEditor 4.x** | Core framework | The file is a plug‑in to CKEditor; it relies on the `CKEDITOR` namespace being present. |
| **No other external libs** | – | All strings are static; there are no runtime dependencies. |

The file is platform‑agnostic – it runs in any browser that supports JavaScript.

## 5. Additional Notes  

### Edge cases  
- **Missing keys**: If a plugin expects a string that is not defined here, CKEditor will fall back to the default (usually English).  
- **Encoding**: The file is encoded in UTF‑8. Ensure the server delivers the file with the correct `Content-Type: text/javascript; charset=UTF-8` header to avoid mojibake.  
- **Pluralization**: CKEditor’s i18n system does not support plural forms; translators must provide correct singular strings.  

### Future enhancements  
1. **Modular language files** – Split the large object into smaller files per plugin (e.g., `link.js`, `image.js`) to reduce load size for browsers that only need a subset.  
2. **Locale fallback chain** – Add logic to fall back to a parent locale (`et-EE` → `et`) if a key is missing.  
3. **Dynamic validation** – Provide translations for dynamic validation messages (e.g., `fieldRequired: 'Pole väli %s.'`).  
4. **Testing harness** – Add a unit‑test suite that verifies that every required key is present for each plugin.  

### Code quality observations  
- **Readability**: The nested structure is verbose but clear.  
- **Consistency**: Most keys follow the same naming pattern; minor inconsistencies (e.g., `targetNotSet` vs. `targetFrame`) are acceptable because they reflect the core API.  
- **Duplication**: The string `'targetNotSet'` appears in multiple sections; if the same message changes, it must be updated everywhere.  
- **Potential improvements**:  
  - Use a helper function to generate repeated patterns (`targetPopupName`, `targetNew`, etc.) if the file were programmatically generated.  
  - Add comments for less obvious strings (e.g., `popupResizable`) to aid translators.

Overall, the file is well‑structured for its purpose: a comprehensive, static Estonian translation resource for CKEditor.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.et={dir:'ltr',editorTitle:'Rich text editor, %1',source:'Lähtekood',newPage:'Uus leht',save:'Salvesta',preview:'Eelvaade',cut:'Lõika',copy:'Kopeeri',paste:'Kleebi',print:'Prindi',underline:'Allajoonitud',bold:'Paks',italic:'Kursiiv',selectAll:'Vali kõik',removeFormat:'Eemalda vorming',strike:'Läbijoonitud',subscript:'Allindeks',superscript:'Ülaindeks',horizontalrule:'Sisesta horisontaaljoon',pagebreak:'Sisesta lehevahetuskoht',unlink:'Eemalda link',undo:'Võta tagasi',redo:'Korda toimingut',common:{browseServer:'Sirvi serverit',url:'URL',protocol:'Protokoll',upload:'Lae üles',uploadSubmit:'Saada serverissee',image:'Pilt',flash:'Flash',form:'Vorm',checkbox:'Märkeruut',radio:'Raadionupp',textField:'Tekstilahter',textarea:'Tekstiala',hiddenField:'Varjatud lahter',button:'Nupp',select:'Valiklahter',imageButton:'Piltnupp',notSet:'<määramata>',id:'Id',name:'Nimi',langDir:'Keele suund',langDirLtr:'Vasakult paremale (LTR)',langDirRtl:'Paremalt vasakule (RTL)',langCode:'Keele kood',longDescr:'Pikk kirjeldus URL',cssClass:'Stiilistiku klassid',advisoryTitle:'Juhendav tiitel',cssStyle:'Laad',ok:'OK',cancel:'Loobu',generalTab:'General',advancedTab:'Täpsemalt',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'Sisesta erimärk',title:'Vali erimärk'},link:{toolbar:'Sisesta link / Muuda linki',menu:'Muuda linki',title:'Link',info:'Lingi info',target:'Sihtkoht',upload:'Lae üles',advanced:'Täpsemalt',type:'Lingi tüüp',toAnchor:'Ankur sellel lehel',toEmail:'E-post',target:'Sihtkoht',targetNotSet:'<määramata>',targetFrame:'<raam>',targetPopup:'<hüpikaken>',targetNew:'Uus aken (_blank)',targetTop:'Pealmine aken (_top)',targetSelf:'Sama aken (_self)',targetParent:'Esivanem aken (_parent)',targetFrameName:'Sihtmärk raami nimi',targetPopupName:'Hüpikakna nimi',popupFeatures:'Hüpikakna omadused',popupResizable:'Resizable',popupStatusBar:'Olekuriba',popupLocationBar:'Aadressiriba',popupToolbar:'Tööriistariba',popupMenuBar:'Menüüriba',popupFullScreen:'Täisekraan (IE)',popupScrollBars:'Kerimisribad',popupDependent:'Sõltuv (Netscape)',popupWidth:'Laius',popupLeft:'Vasak asukoht',popupHeight:'Kõrgus',popupTop:'Ülemine asukoht',id:'Id',langDir:'Keele suund',langDirNotSet:'<määramata>',langDirLTR:'Vasakult paremale (LTR)',langDirRTL:'Paremalt vasakule (RTL)',acccessKey:'Juurdepääsu võti',name:'Nimi',langCode:'Keele suund',tabIndex:'Tab indeks',advisoryTitle:'Juhendav tiitel',advisoryContentType:'Juhendava sisu tüüp',cssClasses:'Stiilistiku klassid',charset:'Lingitud ressurssi märgistik',styles:'Laad',selectAnchor:'Vali ankur',anchorName:'Ankru nime järgi',anchorId:'Elemendi id järgi',emailAddress:'E-posti aadress',emailSubject:'Sõnumi teema',emailBody:'Sõnumi tekst',noAnchors:'(Selles dokumendis ei ole ankruid)',noUrl:'Palun kirjuta lingi URL',noEmail:'Palun kirjuta E-Posti aadress'},anchor:{toolbar:'Sisesta ankur / Muuda ankrut',menu:'Ankru omadused',title:'Ankru omadused',name:'Ankru nimi',errorName:'Palun sisest ankru nimi'},findAndReplace:{title:'Otsi ja asenda',find:'Otsi',replace:'Asenda',findWhat:'Leia mida:',replaceWith:'Asenda millega:',notFoundMsg:'Valitud teksti ei leitud.',matchCase:'Erista suur- ja väiketähti',matchWord:'Otsi terviklike sõnu',matchCyclic:'Match cyclic',replaceAll:'Asenda kõik',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'Tabel',title:'Tabeli atribuudid',menu:'Tabeli atribuudid',deleteTable:'Kustuta tabel',rows:'Read',columns:'Veerud',border:'Joone suurus',align:'Joondus',alignNotSet:'<Määramata>',alignLeft:'Vasak',alignCenter:'Kesk',alignRight:'Parem',width:'Laius',widthPx:'pikslit',widthPc:'protsenti',height:'Kõrgus',cellSpace:'Lahtri vahe',cellPad:'Lahtri täidis',caption:'Tabeli tiitel',summary:'Kokkuvõte',headers:'Headers',headersNone:'None',headersColumn:'First column',headersRow:'First Row',headersBoth:'Both',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'Lahter',insertBefore:'Sisesta lahter enne',insertAfter:'Sisesta lahter peale',deleteCell:'Eemalda lahtrid',merge:'Ühenda lahtrid',mergeRight:'Ühenda paremale',mergeDown:'Ühenda alla',splitHorizontal:'Poolita lahter horisontaalselt',splitVertical:'Poolita lahter vertikaalselt',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'Rida',insertBefore:'Sisesta rida enne',insertAfter:'Sisesta rida peale',deleteRow:'Eemalda read'},column:{menu:'Veerg',insertBefore:'Sisesta veerg enne',insertAfter:'Sisesta veerg peale',deleteColumn:'Eemalda veerud'}},button:{title:'Nupu omadused',text:'Tekst (väärtus)',type:'Tüüp',typeBtn:'Nupp',typeSbm:'Saada',typeRst:'Lähtesta'},checkboxAndRadio:{checkboxTitle:'Märkeruudu omadused',radioTitle:'Raadionupu omadused',value:'Väärtus',selected:'Valitud'},form:{title:'Vormi omadused',menu:'Vormi omadused',action:'Toiming',method:'Meetod',encoding:'Encoding',target:'Sihtkoht',targetNotSet:'<määramata>',targetNew:'Uus aken (_blank)',targetTop:'Pealmine aken (_top)',targetSelf:'Sama aken (_self)',targetParent:'Esivanem aken (_parent)'},select:{title:'Valiklahtri omadused',selectInfo:'Info',opAvail:'Võimalikud valikud',value:'Väärtus',size:'Suurus',lines:'ridu',chkMulti:'Võimalda mitu valikut',opText:'Tekst',opValue:'Väärtus',btnAdd:'Lisa',btnModify:'Muuda',btnUp:'Üles',btnDown:'Alla',btnSetValue:'Sea valitud olekuna',btnDelete:'Kustuta'},textarea:{title:'Tekstiala omadused',cols:'Veerge',rows:'Ridu'},textfield:{title:'Tekstilahtri omadused',name:'Nimi',value:'Väärtus',charWidth:'Laius (tähemärkides)',maxChars:'Maksimaalselt tähemärke',type:'Tüüp',typeText:'Tekst',typePass:'Parool'},hidden:{title:'Varjatud lahtri omadused',name:'Nimi',value:'Väärtus'},image:{title:'Pildi atribuudid',titleButton:'Piltnupu omadused',menu:'Pildi atribuudid',infoTab:'Pildi info',btnUpload:'Saada serverissee',url:'URL',upload:'Lae üles',alt:'Alternatiivne tekst',width:'Laius',height:'Kõrgus',lockRatio:'Lukusta kuvasuhe',resetSize:'Lähtesta suurus',border:'Joon',hSpace:'H. vaheruum',vSpace:'V. vaheruum',align:'Joondus',alignLeft:'Vasak',alignAbsBottom:'Abs alla',alignAbsMiddle:'Abs keskele',alignBaseline:'Baasjoonele',alignBottom:'Alla',alignMiddle:'Keskele',alignRight:'Paremale',alignTextTop:'Tekstit üles',alignTop:'Üles',preview:'Eelvaade',alertUrl:'Palun kirjuta pildi URL',linkTab:'Link',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'Flash omadused',propertiesTab:'Properties',title:'Flash omadused',chkPlay:'Automaatne start ',chkLoop:'Korduv',chkMenu:'Võimalda flash menüü',chkFull:'Allow Fullscreen',scale:'Mastaap',scaleAll:'Näita kõike',scaleNoBorder:'Äärist ei ole',scaleFit:'Täpne sobivus',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'Joondus',alignLeft:'Vasak',alignAbsBottom:'Abs alla',alignAbsMiddle:'Abs keskele',alignBaseline:'Baasjoonele',alignBottom:'Alla',alignMiddle:'Keskele',alignRight:'Paremale',alignTextTop:'Tekstit üles',alignTop:'Üles',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'Tausta värv',width:'Laius',height:'Kõrgus',hSpace:'H. vaheruum',vSpace:'V. vaheruum',validateSrc:'Palun kirjuta lingi URL',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'Kontrolli õigekirja',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'Puudub sõnastikust',changeTo:'Muuda',btnIgnore:'Ignoreeri',btnIgnoreAll:'Ignoreeri kõiki',btnReplace:'Asenda',btnReplaceAll:'Asenda kõik',btnUndo:'Võta tagasi',noSuggestions:'- Soovitused puuduvad -',progress:'Toimub õigekirja kontroll...',noMispell:'Õigekirja kontroll sooritatud: õigekirjuvigu ei leitud',noChanges:'Õigekirja kontroll sooritatud: ühtegi sõna ei muudetud',oneChange:'Õigekirja kontroll sooritatud: üks sõna muudeti',manyChanges:'Õigekirja kontroll sooritatud: %1 sõna muudetud',ieSpellDownload:'Õigekirja kontrollija ei ole installeeritud. Soovid sa selle alla laadida?'},smiley:{toolbar:'Emotikon',title:'Sisesta emotikon'},elementsPath:{eleTitle:'%1 element'},numberedlist:'Nummerdatud loetelu',bulletedlist:'Punktiseeritud loetelu',indent:'Suurenda taanet',outdent:'Vähenda taanet',justify:{left:'Vasakjoondus',center:'Keskjoondus',right:'Paremjoondus',block:'Rööpjoondus'},blockquote:'Blokktsitaat',clipboard:{title:'Kleebi',cutError:'Sinu veebisirvija turvaseaded ei luba redaktoril automaatselt lõigata. Palun kasutage selleks klaviatuuri klahvikombinatsiooni (Ctrl+X).',copyError:'Sinu veebisirvija turvaseaded ei luba redaktoril automaatselt kopeerida. Palun kasutage selleks klaviatuuri klahvikombinatsiooni (Ctrl+C).',pasteMsg:'Palun kleebi järgnevasse kasti kasutades klaviatuuri klahvikombinatsiooni (<STRONG>Ctrl+V</STRONG>) ja vajuta seejärel <STRONG>OK</STRONG>.',securityMsg:'Sinu veebisirvija turvaseadete tõttu, ei oma redaktor otsest ligipääsu lõikelaua andmetele. Sa pead kleepima need uuesti siia aknasse.'},pastefromword:{toolbar:'Kleebi Wordist',title:'Kleebi Wordist',advice:'Palun kleebi järgnevasse kasti kasutades klaviatuuri klahvikombinatsiooni (<STRONG>Ctrl+V</STRONG>) ja vajuta seejärel <STRONG>OK</STRONG>.',ignoreFontFace:'Ignoreeri kirja definitsioone',removeStyle:'Eemalda stiilide definitsioonid'},pasteText:{button:'Kleebi tavalise tekstina',title:'Kleebi tavalise tekstina'},templates:{button:'Šabloon',title:'Sisu šabloonid',insertOption:'Asenda tegelik sisu',selectPromptMsg:'Palun vali šabloon, et avada see redaktoris<br />(praegune sisu läheb kaotsi):',emptyListMsg:'(Ühtegi šablooni ei ole defineeritud)'},showBlocks:'Näita blokke',stylesCombo:{label:'Laad',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'Vorming',voiceLabel:'Format',panelTitle:'Vorming',panelVoiceLabel:'Select a paragraph format',tag_p:'Tavaline',tag_pre:'Vormindatud',tag_address:'Aadress',tag_h1:'Pealkiri 1',tag_h2:'Pealkiri 2',tag_h3:'Pealkiri 3',tag_h4:'Pealkiri 4',tag_h5:'Pealkiri 5',tag_h6:'Pealkiri 6',tag_div:'Tavaline (DIV)'},font:{label:'Kiri',voiceLabel:'Font',panelTitle:'Kiri',panelVoiceLabel:'Select a font'},fontSize:{label:'Suurus',voiceLabel:'Font Size',panelTitle:'Suurus',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'Teksti värv',bgColorTitle:'Tausta värv',auto:'Automaatne',more:'Rohkem värve...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
