# fo.js

## Review

## 1. Summary  

The file is a **locale definition for CKEditor 4.x** (Faroese language, `fo`).  
It simply assigns an object to `CKEDITOR.lang.fo`. The object contains **hundreds of
key‑value pairs** that map internal identifiers (e.g., `link`, `table`, `colorButton`)
to the Faroese strings displayed in dialogs, tooltips, menus, etc.  

* **Purpose** – provide user‑facing text for the CKEditor UI in Faroese.  
* **Key components** – nested objects (`common`, `link`, `table`, `spellCheck`,
  `about`, `colordialog`, etc.) each holding translation strings for a specific
  part of the editor.  
* **Design** – a flat data structure with no logic, loaded as part of the
  CKEditor plugin set. No external frameworks are involved beyond CKEditor
  itself.  

---

## 2. Detailed Description  

### Flow of Execution  

1. **CKEditor Core** loads its language files on demand (usually via
   `CKEDITOR.plugins.load`).  
2. When the `fo` locale is requested, the script is executed.  
3. `CKEDITOR.lang.fo` is created/overwritten with the large object literal.
4. Subsequent CKEditor code accesses translations through the
   `CKEDITOR.lang` API (e.g., `CKEDITOR.lang.fo.link.title`).

There is **no runtime logic** or state cleanup; the object lives for the
lifetime of the page.

### Interaction Between Components  

- Each top‑level key (`link`, `image`, `table`, etc.) represents a dialog or
  toolbar feature.  
- Nested keys are simple string constants.  
- The editor reads the values using its internal i18n lookup; e.g.
  `CKEDITOR.lang.fo.link.title` supplies the dialog title for the link
  dialog.

### Assumptions / Constraints  

- The code assumes CKEditor’s core has already defined `CKEDITOR` and the
  `lang` namespace.  
- The locale file is **static**; no dynamic loading or runtime evaluation.  
- All string keys are unique within their namespace. Duplicates silently
  override earlier values in JavaScript objects, which can hide mistakes.

### Architecture & Design Choices  

* **Simplicity** – a single plain object keeps the file lightweight.  
* **Extensibility** – adding a new language simply requires another file
  with the same structure.  
* **No external dependencies** – everything is bundled with CKEditor.  

---

## 3. Functions/Methods  

The file contains **no functions or methods**.  
All behaviour is provided by CKEditor’s core which consumes the data object.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| **CKEditor Core** | Third‑party | The file must be loaded *after* `ckeditor.js`. |
| `CKEDITOR.lang` | CKEditor namespace | Standard CKEditor i18n infrastructure. |

There are **no platform‑specific** dependencies; the script is pure JavaScript
and works in all browsers supported by CKEditor.

---

## 5. Additional Notes  

### Duplicate / Overwritten Keys  

| Duplicate key | Location | Effect |
|---------------|----------|--------|
| `link.target` | Appears twice in the `link` object | Second value overrides first (no functional impact but redundant). |
| `link.targetNotSet` | Appears twice | Same override. |
| `link.acccessKey` | Misspelled (`acccessKey` instead of `accessKey`) | Potentially not used by CKEditor; may be a typo. |
| `link.target` (again) | Another occurrence in the same object | Redundant. |

**Recommendation:** Remove duplicates and correct the typo to avoid confusion and
ensure consistency across the translation set.

### Incomplete / Mixed Language Strings  

* `common.validateNumberFailed` still contains English text:  
  ```'This value is not a number.'```.  
  This should be translated to Faroese (e.g. “Eyðir verður at vera tal.”) for
  consistency.

* `editorTitle` contains an English placeholder:  
  ```'Rich text editor, %1'```.  
  If the surrounding UI uses English for placeholders, this is fine; otherwise
  translate the whole string.

### Potential Edge Cases  

* **Missing keys** – If a CKEditor component expects a key that isn’t present,
  it falls back to English or a default. Ensure that all mandatory keys are
  defined.  
* **Encoding** – The file uses standard UTF‑8. If the server serves it with
  a different charset, some Faroese characters might render incorrectly.

### Future Enhancements  

1. **Automated Linting** – Run a script to detect duplicate keys, typos,
   and missing translations.  
2. **Placeholders Verification** – Ensure that all `%1`, `$1`, `{{var}}` etc.
   are preserved in the translated strings.  
3. **Unit Tests** – Verify that each key is present and non‑empty.  
4. **Version Control** – Keep locale files in a separate repo or branch for
   easier collaboration.

---

### Summary of Issues to Address  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| Duplicate `link.target` / `link.targetNotSet` | Redundant, may hide real overrides | Remove duplicates |
| `link.acccessKey` typo | May not be recognized by CKEditor | Rename to `accessKey` |
| `common.validateNumberFailed` in English | Inconsistent localization | Translate string |
| Redundant keys elsewhere | Code maintenance overhead | Clean up |
| No functions | Not an issue; file is intentionally data‑only | – |

Overall, the file is well‑structured for its purpose. Addressing the small
typos and duplicate entries will improve maintainability and ensure that the
Faroese locale behaves exactly as the rest of the editor.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.fo={dir:'ltr',editorTitle:'Rich text editor, %1',source:'Kelda',newPage:'Nýggj síða',save:'Goym',preview:'Frumsýning',cut:'Kvett',copy:'Avrita',paste:'Innrita',print:'Prenta',underline:'Undirstrikað',bold:'Feit skrift',italic:'Skráskrift',selectAll:'Markera alt',removeFormat:'Strika sniðgeving',strike:'Yvirstrikað',subscript:'Lækkað skrift',superscript:'Hækkað skrift',horizontalrule:'Ger vatnrætta linju',pagebreak:'Ger síðuskift',unlink:'Strika tilknýti',undo:'Angra',redo:'Vend aftur',common:{browseServer:'Ambætarakagi',url:'URL',protocol:'Protokoll',upload:'Send til ambætaran',uploadSubmit:'Send til ambætaran',image:'Myndir',flash:'Flash',form:'Formur',checkbox:'Flugubein',radio:'Radioknøttur',textField:'Tekstteigur',textarea:'Tekstumráði',hiddenField:'Fjaldur teigur',button:'Knøttur',select:'Valskrá',imageButton:'Myndaknøttur',notSet:'<ikki sett>',id:'Id',name:'Navn',langDir:'Tekstkós',langDirLtr:'Frá vinstru til høgru (LTR)',langDirRtl:'Frá høgru til vinstru (RTL)',langCode:'Málkoda',longDescr:'Víðkað URL frágreiðing',cssClass:'Typografi klassar',advisoryTitle:'Vegleiðandi heiti',cssStyle:'Typografi',ok:'Góðkent',cancel:'Avlýst',generalTab:'Generelt',advancedTab:'Fjølbroytt',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'Set inn sertekn',title:'Vel sertekn'},link:{toolbar:'Ger/broyt tilknýti',menu:'Broyt tilknýti',title:'Tilknýti',info:'Tilknýtis upplýsingar',target:'Mál',upload:'Send til ambætaran',advanced:'Fjølbroytt',type:'Tilknýtisslag',toAnchor:'Tilknýti til marknastein í tekstinum',toEmail:'Teldupostur',target:'Mál',targetNotSet:'<ikki sett>',targetFrame:'<ramma>',targetPopup:'<popup vindeyga>',targetNew:'Nýtt vindeyga (_blank)',targetTop:'Alt vindeygað (_top)',targetSelf:'Sama vindeygað (_self)',targetParent:'Upphavliga vindeygað (_parent)',targetFrameName:'Vís navn vindeygans',targetPopupName:'Popup vindeygans navn',popupFeatures:'Popup vindeygans víðkaðu eginleikar',popupResizable:'Resizable',popupStatusBar:'Støðufrágreiðingarbjálki',popupLocationBar:'Adressulinja',popupToolbar:'Amboðsbjálki',popupMenuBar:'Skrábjálki',popupFullScreen:'Fullur skermur (IE)',popupScrollBars:'Rullibjálki',popupDependent:'Bundið (Netscape)',popupWidth:'Breidd',popupLeft:'Frástøða frá vinstru',popupHeight:'Hædd',popupTop:'Frástøða frá íerva',id:'Id',langDir:'Tekstkós',langDirNotSet:'<ikki sett>',langDirLTR:'Frá vinstru til høgru (LTR)',langDirRTL:'Frá høgru til vinstru (RTL)',acccessKey:'Snarvegisknappur',name:'Navn',langCode:'Tekstkós',tabIndex:'Inntriv indeks',advisoryTitle:'Vegleiðandi heiti',advisoryContentType:'Vegleiðandi innihaldsslag',cssClasses:'Typografi klassar',charset:'Atknýtt teknsett',styles:'Typografi',selectAnchor:'Vel ein marknastein',anchorName:'Eftir navni á marknasteini',anchorId:'Eftir element Id',emailAddress:'Teldupost-adressa',emailSubject:'Evni',emailBody:'Breyðtekstur',noAnchors:'(Eingir marknasteinar eru í hesum dokumentið)',noUrl:'Vinarliga skriva tilknýti (URL)',noEmail:'Vinarliga skriva teldupost-adressu'},anchor:{toolbar:'Ger/broyt marknastein',menu:'Eginleikar fyri marknastein',title:'Eginleikar fyri marknastein',name:'Heiti marknasteinsins',errorName:'Vinarliga rita marknasteinsins heiti'},findAndReplace:{title:'Finn og broyt',find:'Leita',replace:'Yvirskriva',findWhat:'Finn:',replaceWith:'Yvirskriva við:',notFoundMsg:'Leititeksturin varð ikki funnin',matchCase:'Munur á stórum og smáðum bókstavum',matchWord:'Bert heil orð',matchCyclic:'Match cyclic',replaceAll:'Yvirskriva alt',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'Tabell',title:'Eginleikar fyri tabell',menu:'Eginleikar fyri tabell',deleteTable:'Strika tabell',rows:'Røðir',columns:'Kolonnur',border:'Bordabreidd',align:'Justering',alignNotSet:'<Einki valt>',alignLeft:'Vinstrasett',alignCenter:'Miðsett',alignRight:'Høgrasett',width:'Breidd',widthPx:'pixels',widthPc:'prosent',height:'Hædd',cellSpace:'Fjarstøða millum meskar',cellPad:'Meskubreddi',caption:'Tabellfrágreiðing',summary:'Samandráttur',headers:'Headers',headersNone:'None',headersColumn:'First column',headersRow:'First Row',headersBoth:'Both',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'Meski',insertBefore:'Set meska inn áðrenn',insertAfter:'Set meska inn aftaná',deleteCell:'Strika meskar',merge:'Flætta meskar',mergeRight:'Flætta meskar til høgru',mergeDown:'Flætta saman',splitHorizontal:'Kloyv meska vatnrætt',splitVertical:'Kloyv meska loddrætt',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'Rað',insertBefore:'Set rað inn áðrenn',insertAfter:'Set rað inn aftaná',deleteRow:'Strika røðir'},column:{menu:'Kolonna',insertBefore:'Set kolonnu inn áðrenn',insertAfter:'Set kolonnu inn aftaná',deleteColumn:'Strika kolonnur'}},button:{title:'Eginleikar fyri knøtt',text:'Tekstur',type:'Slag',typeBtn:'Knøttur',typeSbm:'Send',typeRst:'Nullstilla'},checkboxAndRadio:{checkboxTitle:'Eginleikar fyri flugubein',radioTitle:'Eginleikar fyri radioknøtt',value:'Virði',selected:'Valt'},form:{title:'Eginleikar fyri Form',menu:'Eginleikar fyri Form',action:'Hending',method:'Háttur',encoding:'Encoding',target:'Mál',targetNotSet:'<ikki sett>',targetNew:'Nýtt vindeyga (_blank)',targetTop:'Alt vindeygað (_top)',targetSelf:'Sama vindeygað (_self)',targetParent:'Upphavliga vindeygað (_parent)'},select:{title:'Eginleikar fyri valskrá',selectInfo:'Upplýsingar',opAvail:'Tøkir møguleikar',value:'Virði',size:'Stødd',lines:'Linjur',chkMulti:'Loyv fleiri valmøguleikum samstundis',opText:'Tekstur',opValue:'Virði',btnAdd:'Legg afturat',btnModify:'Broyt',btnUp:'Upp',btnDown:'Niður',btnSetValue:'Set sum valt virði',btnDelete:'Strika'},textarea:{title:'Eginleikar fyri tekstumráði',cols:'kolonnur',rows:'røðir'},textfield:{title:'Eginleikar fyri tekstteig',name:'Navn',value:'Virði',charWidth:'Breidd (sjónlig tekn)',maxChars:'Mest loyvdu tekn',type:'Slag',typeText:'Tekstur',typePass:'Loyniorð'},hidden:{title:'Eginleikar fyri fjaldan teig',name:'Navn',value:'Virði'},image:{title:'Myndaeginleikar',titleButton:'Eginleikar fyri myndaknøtt',menu:'Myndaeginleikar',infoTab:'Myndaupplýsingar',btnUpload:'Send til ambætaran',url:'URL',upload:'Send',alt:'Alternativur tekstur',width:'Breidd',height:'Hædd',lockRatio:'Læs lutfallið',resetSize:'Upprunastødd',border:'Bordi',hSpace:'Høgri breddi',vSpace:'Vinstri breddi',align:'Justering',alignLeft:'Vinstra',alignAbsBottom:'Abs botnur',alignAbsMiddle:'Abs miðja',alignBaseline:'Basislinja',alignBottom:'Botnur',alignMiddle:'Miðja',alignRight:'Høgra',alignTextTop:'Tekst toppur',alignTop:'Ovast',preview:'Frumsýning',alertUrl:'Rita slóðina til myndina',linkTab:'Tilknýti',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'Flash eginleikar',propertiesTab:'Properties',title:'Flash eginleikar',chkPlay:'Avspælingin byrjar sjálv',chkLoop:'Endurspæl',chkMenu:'Ger Flash skrá virkna',chkFull:'Allow Fullscreen',scale:'Skalering',scaleAll:'Vís alt',scaleNoBorder:'Eingin bordi',scaleFit:'Neyv skalering',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'Justering',alignLeft:'Vinstra',alignAbsBottom:'Abs botnur',alignAbsMiddle:'Abs miðja',alignBaseline:'Basislinja',alignBottom:'Botnur',alignMiddle:'Miðja',alignRight:'Høgra',alignTextTop:'Tekst toppur',alignTop:'Ovast',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'Bakgrundslitur',width:'Breidd',height:'Hædd',hSpace:'Høgri breddi',vSpace:'Vinstri breddi',validateSrc:'Vinarliga skriva tilknýti (URL)',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'Kanna stavseting',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'Finst ikki í orðabókini',changeTo:'Broyt til',btnIgnore:'Forfjóna',btnIgnoreAll:'Forfjóna alt',btnReplace:'Yvirskriva',btnReplaceAll:'Yvirskriva alt',btnUndo:'Angra',noSuggestions:'- Einki uppskot -',progress:'Rættstavarin arbeiðir...',noMispell:'Rættstavarain liðugur: Eingin feilur funnin',noChanges:'Rættstavarain liðugur: Einki orð varð broytt',oneChange:'Rættstavarain liðugur: Eitt orð er broytt',manyChanges:'Rættstavarain liðugur: %1 orð broytt',ieSpellDownload:'Rættstavarin er ikki tøkur í tekstviðgeranum. Vilt tú heinta hann nú?'},smiley:{toolbar:'Smiley',title:'Vel Smiley'},elementsPath:{eleTitle:'%1 element'},numberedlist:'Talmerktur listi',bulletedlist:'Punktmerktur listi',indent:'Økja reglubrotarinntriv',outdent:'Minka reglubrotarinntriv',justify:{left:'Vinstrasett',center:'Miðsett',right:'Høgrasett',block:'Javnir tekstkantar'},blockquote:'Blockquote',clipboard:{title:'Innrita',cutError:'Trygdaruppseting alnótskagans forðar tekstviðgeranum í at kvetta tekstin. Vinarliga nýt knappaborðið til at kvetta tekstin (CTRL+X).',copyError:'Trygdaruppseting alnótskagans forðar tekstviðgeranum í at avrita tekstin. Vinarliga nýt knappaborðið til at avrita tekstin (CTRL+C).',pasteMsg:'Vinarliga koyr tekstin í hendan rútin við knappaborðinum (<strong>CTRL+V</strong>) og klikk á <strong>Góðtak</strong>.',securityMsg:'Trygdaruppseting alnótskagans forðar tekstviðgeranum í beinleiðis atgongd til avritingarminnið. Tygum mugu royna aftur í hesum rútinum.'},pastefromword:{toolbar:'Innrita frá Word',title:'Innrita frá Word',advice:'Vinarliga koyr tekstin í hendan rútin við knappaborðinum (<strong>CTRL+V</strong>) og klikk á <strong>Góðtak</strong>.',ignoreFontFace:'Forfjóna Font definitiónirnar',removeStyle:'Strika typografi definitiónir'},pasteText:{button:'Innrita som reinan tekst',title:'Innrita som reinan tekst'},templates:{button:'Skabelónir',title:'Innihaldsskabelónir',insertOption:'Yvirskriva núverandi innihald',selectPromptMsg:'Vinarliga vel ta skabelón, ið skal opnast í tekstviðgeranum<br>(Hetta yvirskrivar núverandi innihald):',emptyListMsg:'(Ongar skabelónir tøkar)'},showBlocks:'Vís blokkar',stylesCombo:{label:'Typografi',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'Skriftsnið',voiceLabel:'Format',panelTitle:'Skriftsnið',panelVoiceLabel:'Select a paragraph format',tag_p:'Vanligt',tag_pre:'Sniðgivið',tag_address:'Adressa',tag_h1:'Yvirskrift 1',tag_h2:'Yvirskrift 2',tag_h3:'Yvirskrift 3',tag_h4:'Yvirskrift 4',tag_h5:'Yvirskrift 5',tag_h6:'Yvirskrift 6',tag_div:'Normal (DIV)'},font:{label:'Skrift',voiceLabel:'Font',panelTitle:'Skrift',panelVoiceLabel:'Select a font'},fontSize:{label:'Skriftstødd',voiceLabel:'Font Size',panelTitle:'Skriftstødd',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'Tekstlitur',bgColorTitle:'Bakgrundslitur',auto:'Automatiskt',more:'Fleiri litir...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
