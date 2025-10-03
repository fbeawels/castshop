# fa.js

## Review

## 1. Summary  
This file is a **CKEditor language pack** for Persian (Farsi).  
It registers a large `CKEDITOR.lang.fa` object that contains all user‑visible strings used by the editor – tooltips, dialog titles, validation messages, UI button labels, etc. The file follows the standard CKEditor pattern of defining a `lang` sub‑object per language, which the editor loads when the `lang` configuration is set to `"fa"`.

### Key components
| Component | Purpose |
|-----------|---------|
| `CKEDITOR.lang.fa` | Root object that holds every localized string. |
| Nested objects (`common`, `link`, `image`, `table`, `findAndReplace`, …) | Group related strings by UI element or dialog. |
| Special keys such as `editorTitle`, `placeholder`, `direction` | Metadata used by CKEditor itself (e.g., `dir: 'rtl'`). |

The file relies on **no external libraries** beyond the CKEditor core; it is pure JavaScript that is executed in the browser or within a Node/webpack environment.

---

## 2. Detailed Description  
The file is a single script that is evaluated by the CKEditor runtime. When CKEditor is initialized with the language set to `"fa"` it looks for `CKEDITOR.lang.fa`. The object literal is parsed once, creating a memory footprint of all the string literals. The editor then uses these values in its UI through the `CKEDITOR.lang` lookup mechanism.

### Execution flow
1. **Script load** – The script is included in the page before CKEditor is instantiated (usually via `<script src="path/to/ckeditor/lang/fa.js"></script>`).  
2. **Object creation** – JavaScript engine parses the object literal and attaches it to the global `CKEDITOR` namespace.  
3. **Editor initialization** – CKEditor reads the `lang` configuration, looks up `CKEDITOR.lang.fa`, and substitutes all strings in the UI accordingly.  
4. **Runtime behavior** – No further code execution; the object simply serves as a static lookup table.

There is **no cleanup** necessary – the object remains in memory as long as the page is active.

### Assumptions & Constraints
- The code assumes that the `CKEDITOR` global variable exists; it is part of the CKEditor bundle.  
- All string values are expected to be plain text; placeholders (`%1`, `%2`, etc.) follow the CKEditor convention.  
- The file uses the UTF‑8 character set; the editor must be served with a matching `charset` to display Persian characters correctly.  
- Because the file is a language pack, changes to it only affect UI text; no functionality changes are introduced.

### Architectural notes
- **Modularity**: The file is a self‑contained module – no functions or side effects.  
- **Maintainability**: The nested structure mirrors CKEditor’s dialog XML, making it easier for translators to navigate.  
- **Performance**: Since the object is static, it can be minified or bundled with the rest of the editor for production, reducing network overhead.

---

## 3. Functions/Methods  
The file **does not define any functions or methods**. All code is declarative: a large JavaScript object. Therefore there are no inputs, outputs, or side effects to document beyond the object creation.

If an extension needs to be added (e.g., custom UI labels), the usual pattern would be:

```js
CKEDITOR.lang.fa.customLabel = 'متن دلخواه';
```

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party (CKEditor core) | Must be loaded beforehand. |
| UTF‑8 encoding | Standard | Required for Persian glyphs. |

No other libraries or APIs are referenced. The file is pure vanilla JavaScript.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code‑quality observations  
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Typo in key names** – e.g. `acccessKey` (three *c*s) and `validateNumberFailed` is still in English. | Inconsistent key names could cause bugs if CKEditor later references the correct key. | Correct to `accessKey` and translate all remaining English messages. |
| **Mixed case in key names** – e.g., `alignAbsBottom`, `alignAbsMiddle` use mixed case; CKEditor expects `alignAbsBottom`. | This is fine if CKEditor uses the same casing, but keep consistency across all keys. | Ensure all keys match the official CKEditor spec. |
| **Duplicate `target` key in the `link` section** (declared twice). | The second declaration overrides the first silently, but it’s confusing. | Remove the duplicate. |
| **English strings in an otherwise Persian file** – e.g., `validateNumberFailed`, `scayt`, `maximize`, `minimize`, `fakeobjects`. | Users may see untranslated UI elements. | Translate all English strings or remove unused sections. |
| **Inconsistent placeholder syntax** – Some strings use `%1` while others use `%s`. | CKEditor uses `%1`, `%2`, etc.; `%s` will not be substituted correctly. | Replace `%s` with the correct placeholder. |
| **Missing `charset` metadata** – The file does not declare its encoding. | While most browsers assume UTF‑8, it’s safer to add a BOM or server header. | Ensure the file is served with `Content-Type: text/javascript; charset=utf-8`. |
| **Hard‑coded numeric values** – e.g., `colors` object uses string keys like `'000'`. | Works, but could be expressed as hex values or constants for readability. | Keep as is; no functional impact. |

### 5.2 Suggested enhancements  
1. **Automated validation** – Run a linter that checks for duplicate keys, missing placeholders, and typos in a language file.  
2. **Centralized placeholder handling** – Use a helper function that verifies all placeholders (`%1`, `%2`, …) are present where expected.  
3. **Consistent key ordering** – Alphabetize or group keys to improve readability for translators.  
4. **Locale metadata** – Add a small comment block at the top with language code, author, and translation notes.  
5. **Versioning** – Include a `CKEDITOR.lang.fa.version` field to track when the language pack was last updated.

### 5.3 Edge‑case considerations  
- **Right‑to‑Left rendering**: The `dir: 'rtl'` key is correctly set, but the editor also needs to flip certain UI elements (e.g., the order of toolbar buttons). Verify that the rest of CKEditor’s RTL support is intact.  
- **Dynamic content**: Some strings such as `unavailable` use `<span>` tags with a class. Ensure that CSS for `.cke_accessibility` is loaded; otherwise the placeholder may appear broken.  
- **Accessibility**: The file references a class `cke_accessibility` for screen readers. Verify that the CSS and ARIA attributes are consistent across all dialogs.  

### 5.4 Minor formatting suggestions  
- Convert the long object into multiple smaller files if your build system supports it.  
- Use a consistent indentation (2 spaces) for readability.  
- For extremely long strings (e.g., `confirmNewPage`), consider breaking them into multiple lines with template literals to avoid horizontal scrolling in editors.

---

### Verdict  
The file serves its intended purpose as a language pack and follows CKEditor’s conventions. However, it contains a handful of typographical errors and untranslated strings that can lead to inconsistent UI behavior. Addressing the issues above will improve reliability, maintainability, and user experience for Persian‑speaking users.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.fa={dir:'rtl',editorTitle:'Rich text editor, %1',source:'منبع',newPage:'برگهٴ تازه',save:'ذخیره',preview:'پیشنمایش',cut:'برش',copy:'کپی',paste:'چسباندن',print:'چاپ',underline:'خطزیردار',bold:'درشت',italic:'خمیده',selectAll:'گزینش همه',removeFormat:'برداشتن فرمت',strike:'میانخط',subscript:'زیرنویس',superscript:'بالانویس',horizontalrule:'گنجاندن خط ِافقی',pagebreak:'گنجاندن شکستگی ِپایان ِبرگه',unlink:'برداشتن پیوند',undo:'واچیدن',redo:'بازچیدن',common:{browseServer:'فهرستنمایی سرور',url:'URL',protocol:'پروتکل',upload:'انتقال به سرور',uploadSubmit:'به سرور بفرست',image:'تصویر',flash:'Flash',form:'فرم',checkbox:'خانهٴ گزینهای',radio:'دکمهٴ رادیویی',textField:'فیلد متنی',textarea:'ناحیهٴ متنی',hiddenField:'فیلد پنهان',button:'دکمه',select:'فیلد چندگزینهای',imageButton:'دکمهٴ تصویری',notSet:'<تعیننشده>',id:'شناسه',name:'نام',langDir:'جهتنمای زبان',langDirLtr:'چپ به راست (LTR)',langDirRtl:'راست به چپ (RTL)',langCode:'کد زبان',longDescr:'URL توصیف طولانی',cssClass:'کلاسهای شیوهنامه(Stylesheet)',advisoryTitle:'عنوان کمکی',cssStyle:'شیوه(style)',ok:'پذیرش',cancel:'انصراف',generalTab:'General',advancedTab:'پیشرفته',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'گنجاندن نویسهٴ ویژه',title:'گزینش نویسهٴویژه'},link:{toolbar:'گنجاندن/ویرایش ِپیوند',menu:'ویرایش پیوند',title:'پیوند',info:'اطلاعات پیوند',target:'مقصد',upload:'انتقال به سرور',advanced:'پیشرفته',type:'نوع پیوند',toAnchor:'لنگر در همین صفحه',toEmail:'پست الکترونیکی',target:'مقصد',targetNotSet:'<تعیننشده>',targetFrame:'<فریم>',targetPopup:'<پنجرهٴ پاپاپ>',targetNew:'پنجرهٴ دیگر (_blank)',targetTop:'بالاترین پنجره (_top)',targetSelf:'همان پنجره (_self)',targetParent:'پنجرهٴ والد (_parent)',targetFrameName:'نام فریم مقصد',targetPopupName:'نام پنجرهٴ پاپاپ',popupFeatures:'ویژگیهای پنجرهٴ پاپاپ',popupResizable:'Resizable',popupStatusBar:'نوار وضعیت',popupLocationBar:'نوار موقعیت',popupToolbar:'نوارابزار',popupMenuBar:'نوار منو',popupFullScreen:'تمامصفحه (IE)',popupScrollBars:'میلههای پیمایش',popupDependent:'وابسته (Netscape)',popupWidth:'پهنا',popupLeft:'موقعیت ِچپ',popupHeight:'درازا',popupTop:'موقعیت ِبالا',id:'Id',langDir:'جهتنمای زبان',langDirNotSet:'<تعیننشده>',langDirLTR:'چپ به راست (LTR)',langDirRTL:'راست به چپ (RTL)',acccessKey:'کلید دستیابی',name:'نام',langCode:'جهتنمای زبان',tabIndex:'نمایهٴ دسترسی با Tab',advisoryTitle:'عنوان کمکی',advisoryContentType:'نوع محتوای کمکی',cssClasses:'کلاسهای شیوهنامه(Stylesheet)',charset:'نویسهگان منبع ِپیوندشده',styles:'شیوه(style)',selectAnchor:'یک لنگر برگزینید',anchorName:'با نام لنگر',anchorId:'با شناسهٴ المان',emailAddress:'نشانی پست الکترونیکی',emailSubject:'موضوع پیام',emailBody:'متن پیام',noAnchors:'(در این سند لنگری دردسترس نیست)',noUrl:'لطفا URL پیوند را بنویسید',noEmail:'لطفا نشانی پست الکترونیکی را بنویسید'},anchor:{toolbar:'گنجاندن/ویرایش ِلنگر',menu:'ویژگیهای لنگر',title:'ویژگیهای لنگر',name:'نام لنگر',errorName:'لطفا نام لنگر را بنویسید'},findAndReplace:{title:'جستجو و جایگزینی',find:'جستجو',replace:'جایگزینی',findWhat:'چهچیز را مییابید:',replaceWith:'جایگزینی با:',notFoundMsg:'متن موردنظر یافت نشد.',matchCase:'همسانی در بزرگی و کوچکی نویسهها',matchWord:'همسانی با واژهٴ کامل',matchCyclic:'Match cyclic',replaceAll:'جایگزینی همهٴ یافتهها',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'جدول',title:'ویژگیهای جدول',menu:'ویژگیهای جدول',deleteTable:'پاککردن جدول',rows:'سطرها',columns:'ستونها',border:'اندازهٴ لبه',align:'چینش',alignNotSet:'<تعیننشده>',alignLeft:'چپ',alignCenter:'وسط',alignRight:'راست',width:'پهنا',widthPx:'پیکسل',widthPc:'درصد',height:'درازا',cellSpace:'فاصلهٴ میان سلولها',cellPad:'فاصلهٴ پرشده در سلول',caption:'عنوان',summary:'خلاصه',headers:'Headers',headersNone:'None',headersColumn:'First column',headersRow:'First Row',headersBoth:'Both',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'سلول',insertBefore:'افزودن سلول قبل از',insertAfter:'افزودن سلول بعد از',deleteCell:'حذف سلولها',merge:'ادغام سلولها',mergeRight:'ادغام به راست',mergeDown:'ادغام به پایین',splitHorizontal:'جدا کردن افقی سلول',splitVertical:'جدا کردن عمودی سلول',title:'ویژگیهای سلول',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'سطر',insertBefore:'افزودن سطر قبل از',insertAfter:'افزودن سطر بعد از',deleteRow:'حذف سطرها'},column:{menu:'ستون',insertBefore:'افزودن ستون قبل از',insertAfter:'افزودن ستون بعد از',deleteColumn:'حذف ستونها'}},button:{title:'ویژگیهای دکمه',text:'متن (مقدار)',type:'نوع',typeBtn:'دکمه',typeSbm:'Submit',typeRst:'بازنشانی (Reset)'},checkboxAndRadio:{checkboxTitle:'ویژگیهای خانهٴ گزینهای',radioTitle:'ویژگیهای دکمهٴ رادیویی',value:'مقدار',selected:'برگزیده'},form:{title:'ویژگیهای فرم',menu:'ویژگیهای فرم',action:'رویداد',method:'متد',encoding:'Encoding',target:'مقصد',targetNotSet:'<تعیننشده>',targetNew:'پنجرهٴ دیگر (_blank)',targetTop:'بالاترین پنجره (_top)',targetSelf:'همان پنجره (_self)',targetParent:'پنجرهٴ والد (_parent)'},select:{title:'ویژگیهای فیلد چندگزینهای',selectInfo:'اطلاعات',opAvail:'گزینههای دردسترس',value:'مقدار',size:'اندازه',lines:'خطوط',chkMulti:'گزینش چندگانه فراهم باشد',opText:'متن',opValue:'مقدار',btnAdd:'افزودن',btnModify:'ویرایش',btnUp:'بالا',btnDown:'پائین',btnSetValue:'تنظیم به عنوان مقدار ِبرگزیده',btnDelete:'پاککردن'},textarea:{title:'ویژگیهای ناحیهٴ متنی',cols:'ستونها',rows:'سطرها'},textfield:{title:'ویژگیهای فیلد متنی',name:'نام',value:'مقدار',charWidth:'پهنای نویسه',maxChars:'بیشینهٴ نویسهها',type:'نوع',typeText:'متن',typePass:'گذرواژه'},hidden:{title:'ویژگیهای فیلد پنهان',name:'نام',value:'مقدار'},image:{title:'ویژگیهای تصویر',titleButton:'ویژگیهای دکمهٴ تصویری',menu:'ویژگیهای تصویر',infoTab:'اطلاعات تصویر',btnUpload:'به سرور بفرست',url:'URL',upload:'انتقال به سرور',alt:'متن جایگزین',width:'پهنا',height:'درازا',lockRatio:'قفلکردن ِنسبت',resetSize:'بازنشانی اندازه',border:'لبه',hSpace:'فاصلهٴ افقی',vSpace:'فاصلهٴ عمودی',align:'چینش',alignLeft:'چپ',alignAbsBottom:'پائین مطلق',alignAbsMiddle:'وسط مطلق',alignBaseline:'خطپایه',alignBottom:'پائین',alignMiddle:'وسط',alignRight:'راست',alignTextTop:'متن بالا',alignTop:'بالا',preview:'پیشنمایش',alertUrl:'لطفا URL تصویر را بنویسید',linkTab:'پیوند',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'ویژگیهای Flash',propertiesTab:'Properties',title:'ویژگیهای Flash',chkPlay:'آغاز ِخودکار',chkLoop:'اجرای پیاپی',chkMenu:'دردسترسبودن منوی Flash',chkFull:'Allow Fullscreen',scale:'مقیاس',scaleAll:'نمایش همه',scaleNoBorder:'بدون کران',scaleFit:'جایگیری کامل',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'چینش',alignLeft:'چپ',alignAbsBottom:'پائین مطلق',alignAbsMiddle:'وسط مطلق',alignBaseline:'خطپایه',alignBottom:'پائین',alignMiddle:'وسط',alignRight:'راست',alignTextTop:'متن بالا',alignTop:'بالا',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'رنگ پسزمینه',width:'پهنا',height:'درازا',hSpace:'فاصلهٴ افقی',vSpace:'فاصلهٴ عمودی',validateSrc:'لطفا URL پیوند را بنویسید',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'بررسی املا',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'در واژهنامه یافت نشد',changeTo:'تغییر به',btnIgnore:'چشمپوشی',btnIgnoreAll:'چشمپوشی همه',btnReplace:'جایگزینی',btnReplaceAll:'جایگزینی همه',btnUndo:'واچینش',noSuggestions:'- پیشنهادی نیست -',progress:'بررسی املا در حال انجام...',noMispell:'بررسی املا انجام شد. هیچ غلطاملائی یافت نشد',noChanges:'بررسی املا انجام شد. هیچ واژهای تغییر نیافت',oneChange:'بررسی املا انجام شد. یک واژه تغییر یافت',manyChanges:'بررسی املا انجام شد. %1 واژه تغییر یافت',ieSpellDownload:'بررسیکنندهٴ املا نصب نشده است. آیا میخواهید آن را هماکنون دریافت کنید؟'},smiley:{toolbar:'خندانک',title:'گنجاندن خندانک'},elementsPath:{eleTitle:'%1 element'},numberedlist:'فهرست شمارهدار',bulletedlist:'فهرست نقطهای',indent:'افزایش تورفتگی',outdent:'کاهش تورفتگی',justify:{left:'چپچین',center:'میانچین',right:'راستچین',block:'بلوکچین'},blockquote:'بلوک نقل قول',clipboard:{title:'چسباندن',cutError:'تنظیمات امنیتی مرورگر شما اجازه نمیدهد که ویرایشگر به طور خودکار عملکردهای برش را انجام دهد. لطفا با دکمههای صفحهکلید این کار را انجام دهید (Ctrl+X).',copyError:'تنظیمات امنیتی مرورگر شما اجازه نمیدهد که ویرایشگر به طور خودکار عملکردهای کپیکردن را انجام دهد. لطفا با دکمههای صفحهکلید این کار را انجام دهید (Ctrl+C).',pasteMsg:'لطفا متن را با کلیدهای (<STRONG>Ctrl+V</STRONG>) در این جعبهٴ متنی بچسبانید و <STRONG>پذیرش</STRONG> را بزنید.',securityMsg:'به خاطر تنظیمات امنیتی مرورگر شما، ویرایشگر نمیتواند دسترسی مستقیم به دادههای clipboard داشته باشد. شما باید دوباره آنرا در این پنجره بچسبانید.'},pastefromword:{toolbar:'چسباندن از Word',title:'چسباندن از Word',advice:'لطفا متن را با کلیدهای (<STRONG>Ctrl+V</STRONG>) در این جعبهٴ متنی بچسبانید و <STRONG>پذیرش</STRONG> را بزنید.',ignoreFontFace:'چشمپوشی از تعاریف نوع قلم',removeStyle:'چشمپوشی از تعاریف سبک (style)'},pasteText:{button:'چسباندن به عنوان متن ِساده',title:'چسباندن به عنوان متن ِساده'},templates:{button:'الگوها',title:'الگوهای محتویات',insertOption:'محتویات کنونی جایگزین شوند',selectPromptMsg:'لطفا الگوی موردنظر را برای بازکردن در ویرایشگر برگزینید<br>(محتویات کنونی از دست خواهند رفت):',emptyListMsg:'(الگوئی تعریف نشده است)'},showBlocks:'نمایش بلوکها',stylesCombo:{label:'سبک',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'فرمت',voiceLabel:'Format',panelTitle:'فرمت',panelVoiceLabel:'Select a paragraph format',tag_p:'نرمال',tag_pre:'فرمتشده',tag_address:'آدرس',tag_h1:'سرنویس 1',tag_h2:'سرنویس 2',tag_h3:'سرنویس 3',tag_h4:'سرنویس 4',tag_h5:'سرنویس 5',tag_h6:'سرنویس 6',tag_div:'بند'},font:{label:'قلم',voiceLabel:'Font',panelTitle:'قلم',panelVoiceLabel:'Select a font'},fontSize:{label:'اندازه',voiceLabel:'Font Size',panelTitle:'اندازه',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'رنگ متن',bgColorTitle:'رنگ پسزمینه',auto:'خودکار',more:'رنگهای بیشتر...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
