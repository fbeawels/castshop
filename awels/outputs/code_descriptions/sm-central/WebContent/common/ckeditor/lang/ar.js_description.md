# ar.js

## Review

## 1. Summary

The file is a **localisation dictionary for the CKEditor rich‑text editor**.  
It attaches an Arabic (`ar`) language definition to the global `CKEDITOR` object:

```js
CKEDITOR.lang.ar = { … };
```

The object contains a flat list of translation strings and a hierarchy of nested
objects that mirror the CKEditor UI structure (buttons, dialogs, toolbars,
etc.).  No executable logic is present; the file’s sole purpose is to provide
human‑readable strings for the editor’s UI when Arabic is selected.

> **Key components**  
> * `dir` – text direction (right‑to‑left)  
> * `editorTitle`, `source`, `save`, … – top‑level UI strings  
> * Nested objects such as `common`, `link`, `image`, `table`, `colorButton`,
>   `scayt`, `about`, etc. – grouped translations for specific plugins or
>   dialogs.  

**Design patterns / frameworks**  
* Standard **object literal** used as a localisation resource.  
* No external frameworks; it plugs into CKEditor’s internal localisation
  loader which looks for `CKEDITOR.lang.<lang>` objects.

---

## 2. Detailed Description

### Structure

| Level | Key | Description |
|-------|-----|-------------|
| Root | `dir` | Text direction (`rtl`). |
| Root | `editorTitle`, `source`, … | UI strings used directly in the editor toolbar. |
| Root | `common` | Shared strings used by many dialogs (e.g. `browseServer`, `ok`). |
| Root | `link`, `anchor`, `image`, `flash`, `table`, `colorButton`, … | Plugin‑specific translation objects. |
| Root | `showBlocks`, `stylesCombo`, `format`, `font`, `fontSize`, `colorButton`, … | Miscellaneous UI parts. |
| Root | `about`, `maximize`, `minimize`, `resize`, `fakeobjects`, `colordialog` | Remaining plugin/UI strings. |

The object is a **flat dictionary** – key names map directly to string
values used by the editor.  Nested objects are also plain dictionaries; they
do not contain functions or logic, only strings.

### Execution Flow

1. **Load time** – When the CKEditor script is parsed, the global `CKEDITOR`
   namespace already exists (it is defined in `ckeditor.js`).  
2. **Locale registration** – This file is loaded *after* `CKEDITOR` has been
   defined.  The assignment `CKEDITOR.lang.ar = …` simply registers the
   Arabic language definition.  
3. **Runtime** – Whenever the editor is initialised with `lang: 'ar'`,
   CKEditor pulls the appropriate strings from `CKEDITOR.lang.ar` to build
   toolbars, dialogs, etc.  No code in this file is executed after the
   assignment.

### Assumptions & Constraints

* **CKEditor global object** – The file assumes that `CKEDITOR` exists.
* **Unicode support** – All strings are UTF‑8; no special escaping is
  required beyond normal JavaScript string delimiters.
* **No dynamic code** – The file is purely declarative; it cannot affect
  behaviour other than the displayed text.

---

## 3. Functions / Methods

The file contains **no functions or methods**.  It only defines a large object
literal.  The only side‑effect is the assignment to `CKEDITOR.lang.ar`.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` global object | *Third‑party* (CKEditor core) | Must be loaded before this file. |
| None | *Standard* | Pure JavaScript – no external libraries. |

There are **no platform‑specific** requirements; the file works in any browser
or Node environment that provides a `CKEDITOR` namespace.

---

## 5. Additional Notes & Recommendations

### Duplicate / Overridden Keys

| Key | Occurrence | Issue |
|-----|------------|-------|
| `target` | Twice in `link` | The second definition will silently overwrite the first. |
| `type` | Twice in `link` | Same overwrite problem. |
| `acccessKey` | Misspelled | Should be `accessKey`. |
| `dlgTitle` in `about` | `عن rotidEKC` | Likely a copy‑paste typo – should be something like `عن CKEditor`. |

These duplications can lead to confusing behaviour or missing translations.
A quick audit with a script that reports duplicate keys would catch them
early.

### Inconsistencies & Typos

* In `link`: `"acccessKey"` (three `c`’s) – should be `"accessKey"`.
* In `about`: `"dlgTitle":"عن rotidEKC"` – probably intended to read `"عن CKEditor"`.
* Several strings contain the Arabic word `بدون تحديد` wrapped in `<…>` tags;
  ensure the tags are intentional and correctly rendered.

### Style & Readability

* Consider grouping related strings into sub‑objects to improve maintainability.
* Use a consistent naming scheme (`menu`, `info`, `title`, `label`, etc.) – most
  of them already follow this but a few keys (`type`) are ambiguous.

### Future Enhancements

1. **Automated Validation** – A small linting script that checks for:
   * Duplicate keys  
   * Missing required keys for each plugin  
   * Correct camelCase/underscore naming
2. **Unicode Normalization** – Ensure all non‑ASCII characters are stored in
   the same Unicode form to avoid subtle bugs when merging translations.
3. **Externalisation of Common Strings** – Move shared strings (e.g. `ok`,
   `cancel`, `browseServer`) to a separate file or module that can be
   reused across languages.
4. **Internationalisation API** – If CKEditor moves to a newer localisation
   format (e.g. JSON with interpolation placeholders), adapt this file
   accordingly.

### Edge Cases

* **Missing keys** – If a key referenced by a plugin is omitted, CKEditor
  falls back to a default string or displays an empty value.  The file
  should be reviewed to ensure all required keys are present.
* **Right‑to‑left rendering** – The `dir: 'rtl'` setting is correct, but
  any embedded HTML (e.g. `<span class="cke_accessibility">`) must also
  respect RTL rules.  Verify that all nested markup renders correctly in
  Arabic contexts.

---

**Conclusion**  
The file serves its purpose as a language pack for Arabic CKEditor.  The
main technical issues are duplicated and misspelled keys that could silently
override translations.  Correcting those, performing a thorough audit,
and optionally adding automated validation will make the localisation
more robust and maintainable.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.ar={dir:'rtl',editorTitle:'محرر النص المنسق, %1',source:'المصدر',newPage:'صفحة جديدة',save:'حفظ',preview:'معاينة الصفحة',cut:'قص',copy:'نسخ',paste:'لصق',print:'طباعة',underline:'تسطير',bold:'غامق',italic:'مائل',selectAll:'تحديد الكل',removeFormat:'إزالة التنسيقات',strike:'يتوسطه خط',subscript:'منخفض',superscript:'مرتفع',horizontalrule:'خط فاصل',pagebreak:'إدخال صفحة جديدة',unlink:'إزالة رابط',undo:'تراجع',redo:'إعادة',common:{browseServer:'تصفح',url:'الرابط',protocol:'البروتوكول',upload:'رفع',uploadSubmit:'أرسل',image:'صورة',flash:'فلاش',form:'نموذج',checkbox:'خانة إختيار',radio:'زر اختيار',textField:'مربع نص',textarea:'مساحة نصية',hiddenField:'إدراج حقل خفي',button:'زر ضغط',select:'اختار',imageButton:'زر صورة',notSet:'<بدون تحديد>',id:'الرقم',name:'الاسم',langDir:'إتجاه النص',langDirLtr:'اليسار لليمين (LTR)',langDirRtl:'اليمين لليسار (RTL)',langCode:'رمز اللغة',longDescr:'الوصف التفصيلى',cssClass:'فئات التنسيق',advisoryTitle:'عنوان التقرير',cssStyle:'نمط',ok:'موافق',cancel:'إلغاء الأمر',generalTab:'عام',advancedTab:'متقدم',validateNumberFailed:'لايوجد نتيجة',confirmNewPage:'ستفقد أي متغييرات اذا لم تقم بحفظها اولا. هل أنت متأكد أنك تريد صفحة جديدة؟',confirmCancel:'بعض الخيارات قد تغيرت. هل أنت متأكد من إغلاق مربع النص؟',unavailable:'%1<span class="cke_accessibility">, غير متاح</span>'},specialChar:{toolbar:'إدراج  خاص.ِ',title:'اختر الخواص'},link:{toolbar:'رابط',menu:'تحرير رابط',title:'إرتباط تشعبي',info:'معلومات الرابط',target:'هدف الرابط',upload:'رفع',advanced:'متقدم',type:'نوع الربط',toAnchor:'مكان في هذا المستند',toEmail:'بريد إلكتروني',target:'هدف الرابط',targetNotSet:'<بدون تحديد>',targetFrame:'<إطار>',targetPopup:'<نافذة منبثقة>',targetNew:'إطار جديد (_blank)',targetTop:'صفحة كاملة (_top)',targetSelf:'الاطار الحالى (_self)',targetParent:'الإطار الأصلي (_parent)',targetFrameName:'اسم الإطار المستهدف',targetPopupName:'اسم النافذة المنبثقة',popupFeatures:'خصائص النافذة المنبثقة',popupResizable:'قابلة التشكيل',popupStatusBar:'شريط الحالة',popupLocationBar:'شريط العنوان',popupToolbar:'شريط الأدوات',popupMenuBar:'القوائم الرئيسية',popupFullScreen:'ملئ الشاشة (IE)',popupScrollBars:'أشرطة التمرير',popupDependent:'تابع (Netscape)',popupWidth:'العرض',popupLeft:'التمركز لليسار',popupHeight:'الإرتفاع',popupTop:'التمركز للأعلى',id:'هوية',langDir:'إتجاه النص',langDirNotSet:'<بدون تحديد>',langDirLTR:'اليسار لليمين (LTR)',langDirRTL:'اليمين لليسار (RTL)',acccessKey:'مفاتيح الإختصار',name:'الاسم',langCode:'كود النص',tabIndex:'الترتيب',advisoryTitle:'عنوان التقرير',advisoryContentType:'نوع التقرير',cssClasses:'فئات التنسيق',charset:'ترميز المادة المطلوبة',styles:'نمط',selectAnchor:'اختر علامة مرجعية',anchorName:'حسب الاسم',anchorId:'حسب رقم العنصر',emailAddress:'عنوان البريد إلكتروني',emailSubject:'موضوع الرسالة',emailBody:'محتوى الرسالة',noAnchors:'(لا توجد علامات مرجعية في هذا المستند)',noUrl:'من فضلك أدخل عنوان الموقع الذي يشير إليه الرابط',noEmail:'من فضلك أدخل عنوان البريد الإلكتروني'},anchor:{toolbar:'إشارة مرجعية',menu:'تحرير الإشارة المرجعية',title:'خصائص الإشارة المرجعية',name:'اسم الإشارة المرجعية',errorName:'الرجاء كتابة اسم الإشارة المرجعية'},findAndReplace:{title:'بحث واستبدال',find:'بحث',replace:'إستبدال',findWhat:'البحث بـ:',replaceWith:'إستبدال بـ:',notFoundMsg:'لم يتم العثور على النص المحدد.',matchCase:'مطابقة حالة الأحرف',matchWord:'مطابقة بالكامل',matchCyclic:'مطابقة دورية',replaceAll:'إستبدال الكل',replaceSuccessMsg:'تم استبدال 1% من الحالات '},table:{toolbar:'جدول',title:'خصائص الجدول',menu:'خصائص الجدول',deleteTable:'حذف الجدول',rows:'صفوف',columns:'أعمدة',border:'الحدود',align:'المحاذاة',alignNotSet:'<بدون محاذاة>',alignLeft:'يسار',alignCenter:'وسط',alignRight:'يمين',width:'العرض',widthPx:'بكسل',widthPc:'بالمئة',height:'الإرتفاع',cellSpace:'تباعد الخلايا',cellPad:'المسافة البادئة',caption:'الوصف',summary:'الخلاصة',headers:'العناوين',headersNone:'بدون',headersColumn:'العمود الأول',headersRow:'الصف الأول',headersBoth:'كلاهما',invalidRows:'عدد الصفوف يجب أن يكون عدداً أكبر من صفر.',invalidCols:'عدد الأعمدة يجب أن يكون عدداً أكبر من صفر.',invalidBorder:'حجم الحد يجب أن يكون عدداً.',invalidWidth:'عرض الجدول يجب أن يكون عدداً.',invalidHeight:'ارتفاع الجدول يجب أن يكون عدداً.',invalidCellSpacing:'المسافة بين الخلايا يجب أن تكون عدداً.',invalidCellPadding:'المسافة البادئة يجب أن تكون عدداً',cell:{menu:'خلية',insertBefore:'إدراج خلية قبل',insertAfter:'إدراج خلية بعد',deleteCell:'حذف خلية',merge:'دمج خلايا',mergeRight:'دمج لليمين',mergeDown:'دمج للأسفل',splitHorizontal:'تقسيم الخلية أفقياً',splitVertical:'تقسيم الخلية عمودياً',title:'خصائص الخلية',cellType:'نوع الخلية',rowSpan:'امتداد الصفوف',colSpan:'امتداد الأعمدة',wordWrap:'التفاف النص',hAlign:'محاذاة أفقية',vAlign:'محاذاة رأسية',alignTop:'أعلى',alignMiddle:'وسط',alignBottom:'أسفل',alignBaseline:'خط القاعدة',bgColor:'لون الخلفية',borderColor:'لون الحدود',data:'بيانات',header:'عنوان',yes:'نعم',no:'لا',invalidWidth:'عرض الخلية يجب أن يكون عدداً.',invalidHeight:'ارتفاع الخلية يجب أن يكون عدداً.',invalidRowSpan:'امتداد الصفوف يجب أن يكون عدداً صحيحاً.',invalidColSpan:'امتداد الأعمدة يجب أن يكون عدداً صحيحاً.',chooseColor:'اختر'},row:{menu:'صف',insertBefore:'إدراج صف قبل',insertAfter:'إدراج صف بعد',deleteRow:'حذف صفوف'},column:{menu:'عمود',insertBefore:'إدراج عمود قبل',insertAfter:'إدراج عمود بعد',deleteColumn:'حذف أعمدة'}},button:{title:'خصائص زر الضغط',text:'القيمة/التسمية',type:'نوع الزر',typeBtn:'زر',typeSbm:'إرسال',typeRst:'إعادة تعيين'},checkboxAndRadio:{checkboxTitle:'خصائص خانة الإختيار',radioTitle:'خصائص زر الخيار',value:'القيمة',selected:'محدد'},form:{title:'خصائص النموذج',menu:'خصائص النموذج',action:'اسم الملف',method:'الأسلوب',encoding:'تشفير',target:'الهدف',targetNotSet:'<بدون تحديد>',targetNew:'نافذة جديدة (_blank)',targetTop:'نافذة بالاعلى (_top)',targetSelf:'نفس النافذة (_self)',targetParent:'النافذة الأصل (_parent)'},select:{title:'خصائص اختيار الحقل',selectInfo:'اختار معلومات',opAvail:'الخيارات المتاحة',value:'القيمة',size:'الحجم',lines:'الأسطر',chkMulti:'السماح بتحديدات متعددة',opText:'النص',opValue:'القيمة',btnAdd:'إضافة',btnModify:'تعديل',btnUp:'أعلى',btnDown:'أسفل',btnSetValue:'إجعلها محددة',btnDelete:'إزالة'},textarea:{title:'خصائص مساحة النص',cols:'الأعمدة',rows:'الصفوف'},textfield:{title:'خصائص مربع النص',name:'الاسم',value:'القيمة',charWidth:'عرض السمات',maxChars:'اقصى عدد للسمات',type:'نوع المحتوى',typeText:'نص',typePass:'كلمة مرور'},hidden:{title:'خصائص الحقل المخفي',name:'الاسم',value:'القيمة'},image:{title:'خصائص الصورة',titleButton:'خصائص زر الصورة',menu:'خصائص الصورة',infoTab:'معلومات الصورة',btnUpload:'أرسلها للخادم',url:'موقع الصورة',upload:'رفع',alt:'عنوان الصورة',width:'العرض',height:'الإرتفاع',lockRatio:'تناسق الحجم',resetSize:'إستعادة الحجم الأصلي',border:'سمك الحدود',hSpace:'تباعد أفقي',vSpace:'تباعد عمودي',align:'محاذاة',alignLeft:'يسار',alignAbsBottom:'أسفل النص',alignAbsMiddle:'وسط السطر',alignBaseline:'على السطر',alignBottom:'أسفل',alignMiddle:'وسط',alignRight:'يمين',alignTextTop:'أعلى النص',alignTop:'أعلى',preview:'معاينة',alertUrl:'فضلاً أكتب الموقع الذي توجد عليه هذه الصورة.',linkTab:'الرابط',button2Img:'هل تريد تحويل زر الصورة المختار إلى صورة بسيطة؟',img2Button:'هل تريد تحويل الصورة المختارة إلى زر صورة؟',urlMissing:'عنوان مصدر الصورة مفقود'},flash:{properties:'خصائص الفلاش',propertiesTab:'الخصائص',title:'خصائص فيلم الفلاش',chkPlay:'تشغيل تلقائي',chkLoop:'تكرار',chkMenu:'تمكين قائمة فيلم الفلاش',chkFull:'ملء الشاشة',scale:'الحجم',scaleAll:'إظهار الكل',scaleNoBorder:'بلا حدود',scaleFit:'ضبط تام',access:'دخول النص البرمجي',accessAlways:'دائماً',accessSameDomain:'نفس النطاق',accessNever:'مطلقاً',align:'محاذاة',alignLeft:'يسار',alignAbsBottom:'أسفل النص',alignAbsMiddle:'وسط السطر',alignBaseline:'على السطر',alignBottom:'أسفل',alignMiddle:'وسط',alignRight:'يمين',alignTextTop:'أعلى النص',alignTop:'أعلى',quality:'جودة',qualityBest:'أفضل',qualityHigh:'عالية',qualityAutoHigh:'عالية تلقائياً',qualityMedium:'متوسطة',qualityAutoLow:'منخفضة تلقائياً',qualityLow:'منخفضة',windowModeWindow:'نافذة',windowModeOpaque:'غير شفاف',windowModeTransparent:'شفاف',windowMode:'وضع النافذة',flashvars:'متغيرات الفلاش',bgcolor:'لون الخلفية',width:'العرض',height:'الإرتفاع',hSpace:'تباعد أفقي',vSpace:'تباعد عمودي',validateSrc:'فضلاً أدخل عنوان الموقع الذي يشير إليه الرابط',validateWidth:'العرض يجب أن يكون عدداً.',validateHeight:'الارتفاع يجب أن يكون عدداً.',validateHSpace:'HSpace يجب أن يكون عدداً.',validateVSpace:'VSpace يجب أن يكون عدداً.'},spellCheck:{toolbar:'تدقيق إملائي',title:'التدقيق الإملائي',notAvailable:'عفواً، ولكن هذه الخدمة غير متاحة الان',errorLoading:'خطأ في تحميل تطبيق خدمة الاستضافة: %s.',notInDic:'ليست في القاموس',changeTo:'التغيير إلى',btnIgnore:'تجاهل',btnIgnoreAll:'تجاهل الكل',btnReplace:'تغيير',btnReplaceAll:'تغيير الكل',btnUndo:'تراجع',noSuggestions:'- لا توجد إقتراحات -',progress:'جاري التدقيق الاملائى',noMispell:'تم التدقيق الإملائي: لم يتم العثور على أي أخطاء إملائية',noChanges:'تم التدقيق الإملائي: لم يتم تغيير أي كلمة',oneChange:'تم التدقيق الإملائي: تم تغيير كلمة واحدة فقط',manyChanges:'تم إكمال التدقيق الإملائي: تم تغيير %1 من كلمات',ieSpellDownload:'المدقق الإملائي (الإنجليزي) غير مثبّت. هل تود تحميله الآن؟'},smiley:{toolbar:'ابتسامات',title:'إدراج ابتسامات'},elementsPath:{eleTitle:'عنصر 1%'},numberedlist:'ادخال/حذف تعداد رقمي',bulletedlist:'ادخال/حذف تعداد نقطي',indent:'زيادة المسافة البادئة',outdent:'إنقاص المسافة البادئة',justify:{left:'محاذاة إلى اليسار',center:'توسيط',right:'محاذاة إلى اليمين',block:'ضبط'},blockquote:'اقتباس',clipboard:{title:'لصق',cutError:'الإعدادات الأمنية للمتصفح الذي تستخدمه تمنع القص التلقائي. فضلاً إستخدم لوحة المفاتيح لفعل ذلك (Ctrl+X).',copyError:'الإعدادات الأمنية للمتصفح الذي تستخدمه تمنع النسخ التلقائي. فضلاً إستخدم لوحة المفاتيح لفعل ذلك (Ctrl+C).',pasteMsg:'الصق داخل الصندوق بإستخدام زرائر (<STRONG>Ctrl+V</STRONG>) في لوحة المفاتيح، ثم اضغط زر  <STRONG>موافق</STRONG>.',securityMsg:'نظراً لإعدادات الأمان الخاصة بمتصفحك، لن يتمكن هذا المحرر من الوصول لمحتوى حافظتك، لذلك يجب عليك لصق المحتوى مرة أخرى في هذه النافذة.'},pastefromword:{toolbar:'لصق من وورد',title:'لصق من وورد',advice:'الصق داخل الصندوق بإستخدام مفاتيح (<STRONG>Ctrl+V</STRONG>) في لوحة المفاتيح، ثم اضغط مفتاح <STRONG>موافق</STRONG>.',ignoreFontFace:'تجاهل تعريفات أسماء الخطوط',removeStyle:'إزالة تعريفات الأنماط'},pasteText:{button:'لصق كنص بسيط',title:'لصق كنص بسيط'},templates:{button:'القوالب',title:'قوالب المحتوى',insertOption:'استبدال المحتوى',selectPromptMsg:'اختر القالب الذي تود وضعه في المحرر',emptyListMsg:'(لم يتم تعريف أي قالب)'},showBlocks:'مخطط تفصيلي',stylesCombo:{label:'أنماط',voiceLabel:'أنماط',panelVoiceLabel:'اختر نمط',panelTitle1:'أنماط الفقرة',panelTitle2:'أنماط مضمنة',panelTitle3:'أنماط الكائن'},format:{label:'تنسيق',voiceLabel:'تنسيق',panelTitle:'تنسيق الفقرة',panelVoiceLabel:'اختر تنسيق الفقرة',tag_p:'عادي',tag_pre:'منسّق',tag_address:'عنوان',tag_h1:'العنوان 1',tag_h2:'العنوان  2',tag_h3:'العنوان  3',tag_h4:'العنوان  4',tag_h5:'العنوان  5',tag_h6:'العنوان  6',tag_div:'عادي (DIV)'},font:{label:'خط',voiceLabel:'حجم الخط',panelTitle:'حجم الخط',panelVoiceLabel:'اختر حجم الخط'},fontSize:{label:'حجم الخط',voiceLabel:'حجم الخط',panelTitle:'حجم الخط',panelVoiceLabel:'اختر حجم الخط'},colorButton:{textColorTitle:'لون النص',bgColorTitle:'لون الخلفية',auto:'تلقائي',more:'ألوان إضافية...'},colors:{'000':'أسود',800000:'كستنائي','8B4513':'بني فاتح','2F4F4F':'رمادي أردوازي غامق','008080':'أزرق مخضر','000080':'أزرق داكن','4B0082':'كحلي',696969:'رمادي داكن',B22222:'طوبي',A52A2A:'بني',DAA520:'ذهبي داكن','006400':'أخضر داكن','40E0D0':'فيروزي','0000CD':'أزرق متوسط',800080:'بنفسجي غامق',808080:'رمادي',F00:'أحمر',FF8C00:'برتقالي داكن',FFD700:'ذهبي','008000':'أخضر','0FF':'تركواز','00F':'أزرق',EE82EE:'بنفسجي',A9A9A9:'رمادي شاحب',FFA07A:'برتقالي وردي',FFA500:'برتقالي',FFFF00:'أصفر','00FF00':'ليموني',AFEEEE:'فيروزي شاحب',ADD8E6:'أزرق فاتح',DDA0DD:'بنفسجي فاتح',D3D3D3:'رمادي فاتح',FFF0F5:'وردي فاتح',FAEBD7:'أبيض عتيق',FFFFE0:'أصفر فاتح',F0FFF0:'أبيض مائل للأخضر',F0FFFF:'سماوي',F0F8FF:'لبني',E6E6FA:'أرجواني',FFF:'أبيض'},scayt:{title:'تدقيق إملائي أثناء الكتابة',enable:'تفعيل SCAYT',disable:'تعطيل SCAYT',about:'عن SCAYT',toggle:'تثبيت SCAYT',options:'خيارات',langs:'لغات',moreSuggestions:'المزيد من المقترحات',ignore:'تجاهل',ignoreAll:'تجاهل الكل',addWord:'إضافة كلمة',emptyDic:'اسم القاموس يجب ألا يكون فارغاً.',optionsTab:'خيارات',languagesTab:'لغات',dictionariesTab:'قواميس',aboutTab:'عن'},about:{title:'عن CKEditor',dlgTitle:'عن rotidEKC',moreInfo:'للحصول على معلومات الترخيص ، يرجى زيارة موقعنا على شبكة الانترنت:',copy:'حقوق النشر &copy; $1. جميع الحقوق محفوظة.'},maximize:'تكبير',minimize:'تصغير',fakeobjects:{anchor:'إرساء',flash:'رسم متحرك بالفلاش',div:'فاصل صفحة',unknown:'كائن غير معروف'},resize:'اسحب لتغيير الحجم',colordialog:{title:'اختر لون',highlight:'إلقاء الضوء',selected:'مُختار',clear:'مسح'}};



```
