# el.js

## Review

## 1. Summary  

The snippet defines the Greek (`el`) language pack for **CKEditor**, a popular WYSIWYG rich‑text editor.  
It is a single JavaScript file that assigns an object to `CKEDITOR.lang.el`. The object contains:

* **Basic editor UI strings** (e.g., *source*, *preview*, *cut*, *bold*, etc.).  
* **Dialog definitions** for common widgets: image, link, table, form, flash, spell‑check, smiley, etc.  
* **UI configuration strings** for the toolbar, dialog buttons, menu items, and help/about dialogs.  
* **Colour palette** definitions for the colour picker.

The file is completely data‑driven – it contains no executable logic, only key/value pairs that CKEditor will read to display UI text in Greek. The design follows the same pattern used for all CKEditor language packs: a single nested object that mirrors the editor’s UI component hierarchy.

---

## 2. Detailed Description  

### Core Structure  
```js
CKEDITOR.lang.el = {
    dir: 'ltr',                     // Text direction
    editorTitle: 'Rich text editor, %1',
    source: 'HTML κώδικας',
    ...
    common: { ... },
    specialChar: { ... },
    link: { ... },
    anchor: { ... },
    findAndReplace: { ... },
    table: { ... },
    button: { ... },
    checkboxAndRadio: { ... },
    form: { ... },
    select: { ... },
    textarea: { ... },
    textfield: { ... },
    hidden: { ... },
    image: { ... },
    flash: { ... },
    spellCheck: { ... },
    smiley: { ... },
    elementsPath: { ... },
    numberedlist: ...,
    bulletedlist: ...,
    indent: ...,
    outdent: ...,
    justify: { ... },
    blockquote: ...,
    clipboard: { ... },
    pastefromword: { ... },
    pasteText: { ... },
    templates: { ... },
    showBlocks: ...,
    stylesCombo: { ... },
    format: { ... },
    font: { ... },
    fontSize: { ... },
    colorButton: { ... },
    colors: { ... },
    scayt: { ... },
    about: { ... },
    maximize: ...,
    minimize: ...,
    fakeobjects: { ... },
    resize: ...,
    colordialog: { ... }
};
```

* The top‑level object holds general editor strings (title, actions, UI buttons).  
* Nested objects correspond to dialogs or UI elements, matching the structure defined in CKEditor’s source.  
* Each key’s value is a Greek string; numeric keys are used for colour codes.

### Execution Flow  

1. **CKEditor loads** its language files during initialization.  
2. The `lang.el` object is merged into the editor’s language manager (`CKEDITOR.lang`).  
3. When a dialog or toolbar item is created, the editor looks up the appropriate key (e.g., `CKEDITOR.lang.el.link.title`) and displays the Greek text.  
4. No runtime code is executed beyond the assignment itself; the file is purely data.

### Assumptions & Dependencies  

* **CKEditor core** is already loaded; this file assumes `CKEDITOR` and `CKEDITOR.lang` exist.  
* It follows the **CKEditor 3.x/4.x** language API conventions.  
* No external libraries are required – it is a plain JavaScript object.  
* The file expects the environment to support standard JS objects (ES3/ES5 compatibility).

---

## 3. Functions/Methods  

The file contains **no executable functions**. All entries are string literals or nested objects. Therefore there are no methods to document. The structure is a static data object, so any interaction occurs through CKEditor’s own mechanisms.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` global object | Third‑party (CKEditor) | Must be loaded before this file. |
| `CKEDITOR.lang` | Part of CKEditor | This file augments the language dictionary. |

No other libraries, APIs, or platform‑specific code are used.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation of concerns** – UI strings are isolated in one file, enabling easy maintenance.  
* **Consistent naming** – keys mirror CKEditor's own structure, making integration straightforward.  
* **Extensibility** – adding or editing a string is trivial: modify the relevant key.  
* **No runtime overhead** – the file simply loads data.

### Potential Issues  

1. **Encoding** – The file must be served with the correct character set (UTF‑8 or a charset that supports Greek). Otherwise, the Greek characters may appear garbled.  
2. **Version mismatch** – If CKEditor updates its language keys, this file may become outdated. Regular updates are required to keep compatibility.  
3. **Internationalization errors** – Some keys use English placeholders (`%1`, `%2`). If the string formatting logic changes, placeholders may not be replaced correctly.  
4. **Duplicate keys** – The `link` object contains the key `target` twice. While JavaScript objects allow duplicate keys, the latter definition will overwrite the former, potentially causing confusion.  

### Suggested Enhancements  

* **Automated linting** – Run a JSON/YAML linter or a CKEditor language pack validator to catch duplicate keys or missing translations.  
* **Version tagging** – Include a `version` field (e.g., `"CKEditor": "4.18.0"`) to help maintain compatibility.  
* **Internationalization tooling** – Use a tool (e.g., Crowdin, Transifex) to manage translations and keep track of changes.  
* **Color palette comments** – Add comments explaining the meaning of hex codes for easier manual editing.  

Overall, the snippet is a clean, standard CKEditor language definition file that should work seamlessly within the editor’s localization framework.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.el={dir:'ltr',editorTitle:'Rich text editor, %1',source:'HTML κώδικας',newPage:'Νέα Σελίδα',save:'Αποθήκευση',preview:'Προεπισκόπιση',cut:'Αποκοπή',copy:'Αντιγραφή',paste:'Επικόλληση',print:'Εκτύπωση',underline:'Υπογράμμιση',bold:'Έντονα',italic:'Πλάγια',selectAll:'Επιλογή όλων',removeFormat:'Αφαίρεση Μορφοποίησης',strike:'Διαγράμμιση',subscript:'Δείκτης',superscript:'Εκθέτης',horizontalrule:'Εισαγωγή Οριζόντιας Γραμμής',pagebreak:'Εισαγωγή τέλους σελίδας',unlink:'Αφαίρεση Συνδέσμου (Link)',undo:'Αναίρεση',redo:'Επαναφορά',common:{browseServer:'Εξερεύνηση διακομιστή',url:'URL',protocol:'Προτόκολο',upload:'Αποστολή',uploadSubmit:'Αποστολή στον Διακομιστή',image:'Εικόνα',flash:'Εισαγωγή Flash',form:'Φόρμα',checkbox:'Κουτί επιλογής',radio:'Κουμπί Radio',textField:'Πεδίο κειμένου',textarea:'Περιοχή κειμένου',hiddenField:'Κρυφό πεδίο',button:'Κουμπί',select:'Πεδίο επιλογής',imageButton:'Κουμπί εικόνας',notSet:'<χωρίς>',id:'Id',name:'Όνομα',langDir:'Κατεύθυνση κειμένου',langDirLtr:'Αριστερά προς Δεξιά (LTR)',langDirRtl:'Δεξιά προς Αριστερά (RTL)',langCode:'Κωδικός Γλώσσας',longDescr:'Αναλυτική περιγραφή URL',cssClass:'Stylesheet Classes',advisoryTitle:'Συμβουλευτικός τίτλος',cssStyle:'Στύλ',ok:'OK',cancel:'Ακύρωση',generalTab:'General',advancedTab:'Για προχωρημένους',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'Εισαγωγή Ειδικού Συμβόλου',title:'Επιλέξτε ένα Ειδικό Σύμβολο'},link:{toolbar:'Εισαγωγή/Μεταβολή Συνδέσμου (Link)',menu:'Μεταβολή Συνδέσμου (Link)',title:'Σύνδεσμος (Link)',info:'Link',target:'Παράθυρο Στόχος (Target)',upload:'Αποστολή',advanced:'Για προχωρημένους',type:'Τύπος συνδέσμου (Link)',toAnchor:'Άγκυρα σε αυτή τη σελίδα',toEmail:'E-Mail',target:'Παράθυρο Στόχος (Target)',targetNotSet:'<χωρίς>',targetFrame:'<πλαίσιο>',targetPopup:'<παράθυρο popup>',targetNew:'Νέο Παράθυρο (_blank)',targetTop:'Ανώτατο Παράθυρο (_top)',targetSelf:'Ίδιο Παράθυρο (_self)',targetParent:'Γονικό Παράθυρο (_parent)',targetFrameName:'Όνομα πλαισίου στόχου',targetPopupName:'Όνομα Popup Window',popupFeatures:'Επιλογές Popup Window',popupResizable:'Resizable',popupStatusBar:'Μπάρα Status',popupLocationBar:'Μπάρα Τοποθεσίας',popupToolbar:'Μπάρα Εργαλείων',popupMenuBar:'Μπάρα Menu',popupFullScreen:'Ολόκληρη η Οθόνη (IE)',popupScrollBars:'Μπάρες Κύλισης',popupDependent:'Dependent (Netscape)',popupWidth:'Πλάτος',popupLeft:'Τοποθεσία Αριστερής Άκρης',popupHeight:'Ύψος',popupTop:'Τοποθεσία Πάνω Άκρης',id:'Id',langDir:'Κατεύθυνση κειμένου',langDirNotSet:'<χωρίς>',langDirLTR:'Αριστερά προς Δεξιά (LTR)',langDirRTL:'Δεξιά προς Αριστερά (RTL)',acccessKey:'Συντόμευση (Access Key)',name:'Όνομα',langCode:'Κατεύθυνση κειμένου',tabIndex:'Tab Index',advisoryTitle:'Συμβουλευτικός τίτλος',advisoryContentType:'Συμβουλευτικός τίτλος περιεχομένου',cssClasses:'Stylesheet Classes',charset:'Linked Resource Charset',styles:'Στύλ',selectAnchor:'Επιλέξτε μια άγκυρα',anchorName:'Βάσει του Ονόματος (Name) της άγκυρας',anchorId:'Βάσει του Element Id',emailAddress:'Διεύθυνση Ηλεκτρονικού Ταχυδρομείου',emailSubject:'Θέμα Μηνύματος',emailBody:'Κείμενο Μηνύματος',noAnchors:'(Δεν υπάρχουν άγκυρες στο κείμενο)',noUrl:'Εισάγετε την τοποθεσία (URL) του υπερσυνδέσμου (Link)',noEmail:'Εισάγετε την διεύθυνση ηλεκτρονικού ταχυδρομείου'},anchor:{toolbar:'Εισαγωγή/επεξεργασία Anchor',menu:'Ιδιότητες άγκυρας',title:'Ιδιότητες άγκυρας',name:'Όνομα άγκυρας',errorName:'Παρακαλούμε εισάγετε όνομα άγκυρας'},findAndReplace:{title:'Find and Replace',find:'Αναζήτηση',replace:'Αντικατάσταση',findWhat:'Αναζήτηση:',replaceWith:'Αντικατάσταση με:',notFoundMsg:'Το κείμενο δεν βρέθηκε.',matchCase:'Έλεγχος πεζών/κεφαλαίων',matchWord:'Εύρεση πλήρους λέξης',matchCyclic:'Match cyclic',replaceAll:'Αντικατάσταση Όλων',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'Πίνακας',title:'Ιδιότητες Πίνακα',menu:'Ιδιότητες Πίνακα',deleteTable:'Διαγραφή πίνακα',rows:'Γραμμές',columns:'Κολώνες',border:'Μέγεθος Περιθωρίου',align:'Στοίχιση',alignNotSet:'<χωρίς>',alignLeft:'Αριστερά',alignCenter:'Κέντρο',alignRight:'Δεξιά',width:'Πλάτος',widthPx:'pixels',widthPc:'%',height:'Ύψος',cellSpace:'Απόσταση κελιών',cellPad:'Γέμισμα κελιών',caption:'Υπέρτιτλος',summary:'Περίληψη',headers:'Headers',headersNone:'None',headersColumn:'First column',headersRow:'First Row',headersBoth:'Both',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'Κελί',insertBefore:'Insert Cell Before',insertAfter:'Insert Cell After',deleteCell:'Διαγραφή Κελιών',merge:'Ενοποίηση Κελιών',mergeRight:'Merge Right',mergeDown:'Merge Down',splitHorizontal:'Split Cell Horizontally',splitVertical:'Split Cell Vertically',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'Σειρά',insertBefore:'Insert Row Before',insertAfter:'Insert Row After',deleteRow:'Διαγραφή Γραμμών'},column:{menu:'Στήλη',insertBefore:'Insert Column Before',insertAfter:'Insert Column After',deleteColumn:'Διαγραφή Κολωνών'}},button:{title:'Ιδιότητες κουμπιού',text:'Κείμενο (Τιμή)',type:'Τύπος',typeBtn:'Κουμπί',typeSbm:'Καταχώρηση',typeRst:'Επαναφορά'},checkboxAndRadio:{checkboxTitle:'Ιδιότητες κουμπιού επιλογής',radioTitle:'Ιδιότητες κουμπιού radio',value:'Τιμή',selected:'Επιλεγμένο'},form:{title:'Ιδιότητες φόρμας',menu:'Ιδιότητες φόρμας',action:'Δράση',method:'Μάθοδος',encoding:'Encoding',target:'Παράθυρο Στόχος (Target)',targetNotSet:'<χωρίς>',targetNew:'Νέο Παράθυρο (_blank)',targetTop:'Ανώτατο Παράθυρο (_top)',targetSelf:'Ίδιο Παράθυρο (_self)',targetParent:'Γονικό Παράθυρο (_parent)'},select:{title:'Ιδιότητες πεδίου επιλογής',selectInfo:'Πληροφορίες',opAvail:'Διαθέσιμες επιλογές',value:'Τιμή',size:'Μέγεθος',lines:'γραμμές',chkMulti:'Πολλαπλές επιλογές',opText:'Κείμενο',opValue:'Τιμή',btnAdd:'Προσθήκη',btnModify:'Αλλαγή',btnUp:'Πάνω',btnDown:'Κάτω',btnSetValue:'Προεπιλεγμένη επιλογή',btnDelete:'Διαγραφή'},textarea:{title:'Ιδιότητες περιοχής κειμένου',cols:'Στήλες',rows:'Σειρές'},textfield:{title:'Ιδιότητες πεδίου κειμένου',name:'Όνομα',value:'Τιμή',charWidth:'Μήκος χαρακτήρων',maxChars:'Μέγιστοι χαρακτήρες',type:'Τύπος',typeText:'Κείμενο',typePass:'Κωδικός'},hidden:{title:'Ιδιότητες κρυφού πεδίου',name:'Όνομα',value:'Τιμή'},image:{title:'Ιδιότητες Εικόνας',titleButton:'Ιδιότητες κουμπιού εικόνας',menu:'Ιδιότητες Εικόνας',infoTab:'Πληροφορίες Εικόνας',btnUpload:'Αποστολή στον Διακομιστή',url:'URL',upload:'Αποστολή',alt:'Εναλλακτικό Κείμενο (ALT)',width:'Πλάτος',height:'Ύψος',lockRatio:'Κλείδωμα Αναλογίας',resetSize:'Επαναφορά Αρχικού Μεγέθους',border:'Περιθώριο',hSpace:'Οριζόντιος Χώρος (HSpace)',vSpace:'Κάθετος Χώρος (VSpace)',align:'Ευθυγράμμιση (Align)',alignLeft:'Αριστερά',alignAbsBottom:'Απόλυτα Κάτω (Abs Bottom)',alignAbsMiddle:'Απόλυτα στη Μέση (Abs Middle)',alignBaseline:'Γραμμή Βάσης (Baseline)',alignBottom:'Κάτω (Bottom)',alignMiddle:'Μέση (Middle)',alignRight:'Δεξιά (Right)',alignTextTop:'Κορυφή Κειμένου (Text Top)',alignTop:'Πάνω (Top)',preview:'Προεπισκόπιση',alertUrl:'Εισάγετε την τοποθεσία (URL) της εικόνας',linkTab:'Σύνδεσμος',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'Ιδιότητες Flash',propertiesTab:'Properties',title:'Ιδιότητες flash',chkPlay:'Αυτόματη έναρξη',chkLoop:'Επανάληψη',chkMenu:'Ενεργοποίηση Flash Menu',chkFull:'Allow Fullscreen',scale:'Κλίμακα',scaleAll:'Εμφάνιση όλων',scaleNoBorder:'Χωρίς όρια',scaleFit:'Ακριβής εφαρμογή',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'Ευθυγράμμιση (Align)',alignLeft:'Αριστερά',alignAbsBottom:'Απόλυτα Κάτω (Abs Bottom)',alignAbsMiddle:'Απόλυτα στη Μέση (Abs Middle)',alignBaseline:'Γραμμή Βάσης (Baseline)',alignBottom:'Κάτω (Bottom)',alignMiddle:'Μέση (Middle)',alignRight:'Δεξιά (Right)',alignTextTop:'Κορυφή Κειμένου (Text Top)',alignTop:'Πάνω (Top)',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'Χρώμα Υποβάθρου',width:'Πλάτος',height:'Ύψος',hSpace:'Οριζόντιος Χώρος (HSpace)',vSpace:'Κάθετος Χώρος (VSpace)',validateSrc:'Εισάγετε την τοποθεσία (URL) του υπερσυνδέσμου (Link)',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'Ορθογραφικός έλεγχος',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'Δεν υπάρχει στο λεξικό',changeTo:'Αλλαγή σε',btnIgnore:'Αγνόηση',btnIgnoreAll:'Αγνόηση όλων',btnReplace:'Αντικατάσταση',btnReplaceAll:'Αντικατάσταση όλων',btnUndo:'Αναίρεση',noSuggestions:'- Δεν υπάρχουν προτάσεις -',progress:'Ορθογραφικός έλεγχος σε εξέλιξη...',noMispell:'Ο ορθογραφικός έλεγχος ολοκληρώθηκε: Δεν βρέθηκαν λάθη',noChanges:'Ο ορθογραφικός έλεγχος ολοκληρώθηκε: Δεν άλλαξαν λέξεις',oneChange:'Ο ορθογραφικός έλεγχος ολοκληρώθηκε: Μια λέξη άλλαξε',manyChanges:'Ο ορθογραφικός έλεγχος ολοκληρώθηκε: %1 λέξεις άλλαξαν',ieSpellDownload:'Δεν υπάρχει εγκατεστημένος ορθογράφος. Θέλετε να τον κατεβάσετε τώρα;'},smiley:{toolbar:'Smiley',title:'Επιλέξτε ένα Smiley'},elementsPath:{eleTitle:'%1 element'},numberedlist:'Λίστα με Αριθμούς',bulletedlist:'Λίστα με Bullets',indent:'Αύξηση Εσοχής',outdent:'Μείωση Εσοχής',justify:{left:'Στοίχιση Αριστερά',center:'Στοίχιση στο Κέντρο',right:'Στοίχιση Δεξιά',block:'Πλήρης Στοίχιση (Block)'},blockquote:'Blockquote',clipboard:{title:'Επικόλληση',cutError:'Οι ρυθμίσεις ασφαλείας του φυλλομετρητή σας δεν επιτρέπουν την επιλεγμένη εργασία αποκοπής. Χρησιμοποιείστε το πληκτρολόγιο (Ctrl+X).',copyError:'Οι ρυθμίσεις ασφαλείας του φυλλομετρητή σας δεν επιτρέπουν την επιλεγμένη εργασία αντιγραφής. Χρησιμοποιείστε το πληκτρολόγιο (Ctrl+C).',pasteMsg:'Παρακαλώ επικολήστε στο ακόλουθο κουτί χρησιμοποιόντας το πληκτρολόγιο (<STRONG>Ctrl+V</STRONG>) και πατήστε <STRONG>OK</STRONG>.',securityMsg:'Because of your browser security settings, the editor is not able to access your clipboard data directly. You are required to paste it again in this window.'},pastefromword:{toolbar:'Επικόλληση από το Word',title:'Επικόλληση από το Word',advice:'Παρακαλώ επικολήστε στο ακόλουθο κουτί χρησιμοποιόντας το πληκτρολόγιο (<STRONG>Ctrl+V</STRONG>) και πατήστε <STRONG>OK</STRONG>.',ignoreFontFace:'Αγνόηση προδιαγραφών γραμματοσειράς',removeStyle:'Αφαίρεση προδιαγραφών στύλ'},pasteText:{button:'Επικόλληση ως Απλό Κείμενο',title:'Επικόλληση ως Απλό Κείμενο'},templates:{button:'Πρότυπα',title:'Πρότυπα περιεχομένου',insertOption:'Αντικατάσταση υπάρχοντων περιεχομένων',selectPromptMsg:'Παρακαλώ επιλέξτε πρότυπο για εισαγωγή στο πρόγραμμα<br>(τα υπάρχοντα περιεχόμενα θα χαθούν):',emptyListMsg:'(Δεν έχουν καθοριστεί πρότυπα)'},showBlocks:'Show Blocks',stylesCombo:{label:'Στυλ',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'Μορφή Γραμματοσειράς',voiceLabel:'Format',panelTitle:'Μορφή Γραμματοσειράς',panelVoiceLabel:'Select a paragraph format',tag_p:'Κανονικό',tag_pre:'Μορφοποιημένο',tag_address:'Διεύθυνση',tag_h1:'Επικεφαλίδα 1',tag_h2:'Επικεφαλίδα 2',tag_h3:'Επικεφαλίδα 3',tag_h4:'Επικεφαλίδα 4',tag_h5:'Επικεφαλίδα 5',tag_h6:'Επικεφαλίδα 6',tag_div:'Normal (DIV)'},font:{label:'Γραμματοσειρά',voiceLabel:'Font',panelTitle:'Γραμματοσειρά',panelVoiceLabel:'Select a font'},fontSize:{label:'Μέγεθος',voiceLabel:'Font Size',panelTitle:'Μέγεθος',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'Χρώμα Γραμμάτων',bgColorTitle:'Χρώμα Υποβάθρου',auto:'Αυτόματο',more:'Περισσότερα χρώματα...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
