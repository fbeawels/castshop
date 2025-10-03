# tr.js

## Review

## 1. Summary
The file defines the **Turkish (tr)** language pack for CKEditor.  
It extends the global `CKEDITOR.lang` namespace by assigning an object literal to `CKEDITOR.lang.tr`. The object contains all UI text that the editor displays – button labels, dialog titles, validation messages, and so on.  
The code follows a pure data‑definition style: no executable logic, only a large nested object.  It is intended to be loaded by CKEditor’s language loader at runtime.

**Key components**

| Component | Role |
|-----------|------|
| `CKEDITOR.lang.tr` | Top‑level object holding all translation strings |
| Nested objects (`common`, `link`, `image`, `table`, …) | Group translations by UI module |
| Placeholders (`%1`, `%2`, …) | Dynamic values that CKEditor substitutes at runtime |

**Design patterns / frameworks**

* **Internationalization (i18n) by key‑value mapping** – the classic CKEditor approach.
* **Module pattern** – the file itself is a module that augments the global `CKEDITOR` object.

---

## 2. Detailed Description
1. **Initialization**  
   The script is evaluated in the global context.  
   * `CKEDITOR.lang` is assumed to already exist (created by CKEditor’s core).  
   * The assignment `CKEDITOR.lang.tr = {...}` creates or overwrites the Turkish language definition.

2. **Runtime Behavior**  
   When the editor loads the *tr* locale, CKEditor looks up `CKEDITOR.lang.tr` and reads the nested strings.  
   * Buttons, dialog boxes, tooltips, validation messages, etc. are rendered by concatenating these strings with placeholders (e.g., `%1`).

3. **Cleanup**  
   No cleanup is required – the object stays in memory for the lifetime of the editor instance.

4. **Assumptions / Constraints**  
   * The editor’s core must load this file after `CKEDITOR.lang` is defined.  
   * All translation keys used by the editor must be present; missing keys will fall back to English defaults.  
   * Placeholders (`%1`, `%2`, …) must match the order used by the editor.

5. **Architecture**  
   The language pack follows a flat, nested key structure that mirrors the editor’s UI modules.  Each module (e.g., `link`, `image`, `table`) is a sub‑object containing all strings relevant to that module.  This structure simplifies both lookup and maintenance.

---

## 3. Functions/Methods
The file contains **no functions or methods** – it is purely declarative.  The only executable statement is the object assignment:

```js
CKEDITOR.lang.tr = { ... };
```

Therefore, there are no side‑effects beyond adding a new property to the `CKEDITOR.lang` namespace.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| **CKEditor core** | *Required* | The global `CKEDITOR` object and its `lang` namespace must exist. |
| JavaScript object literal | *Standard* | No external libraries are used. |

No platform‑specific code or APIs are involved; the file can be embedded in any web page that includes CKEditor.

---

## 5. Additional Notes & Recommendations
### Strengths
* **Clarity** – Each string is clearly named and grouped, making manual edits straightforward.  
* **Consistency** – Follows CKEditor’s established i18n format.  
* **Extensibility** – Adding new UI elements simply means appending keys to the appropriate sub‑object.

### Potential Edge Cases
* **Missing keys** – If a new editor feature adds a key that is not present in this file, the UI will fallback to English.  
* **Placeholder mismatches** – Some strings contain `%1`, `%2` placeholders. A typo in the placeholder count or order can produce incorrect runtime text.  
* **Non‑UTF‑8 characters** – The file contains Turkish characters. It must be served with the correct character set (`UTF‑8`) to avoid garbled output.

### Future Enhancements
1. **Validation Script** – Add a small helper that runs after load to verify that all expected keys are present and that placeholder counts match the editor’s expectations.  
2. **Auto‑generation** – Use a build step that extracts all translation keys from the editor’s source and generates a template file for translators, reducing the risk of missing entries.  
3. **Pluralization Support** – CKEditor 5 (and newer) support plural forms. If this file is upgraded to CKEditor 5, restructure it to accommodate pluralization arrays.  
4. **Localization Testing** – Integrate automated tests that render a sample dialog and confirm that all UI elements display the correct Turkish text.

### Minor Clean‑ups
* Some strings duplicate keys (e.g., `target` appears twice in the `link` object). Consolidate them to avoid accidental overrides.  
* Remove unused or obsolete strings (e.g., `acccessKey` typo – should be `accessKey`).

---

**Conclusion**  
The file is a straightforward, well‑structured language definition for CKEditor.  It serves its purpose with minimal complexity and no functional bugs.  Minor housekeeping and tooling additions would further improve maintainability and reduce the risk of translation drift.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.tr={dir:'ltr',editorTitle:'Rich text editor, %1',source:'Kaynak',newPage:'Yeni Sayfa',save:'Kaydet',preview:'Ön İzleme',cut:'Kes',copy:'Kopyala',paste:'Yapıştır',print:'Yazdır',underline:'Altı Çizgili',bold:'Kalın',italic:'İtalik',selectAll:'Tümünü Seç',removeFormat:'Biçimi Kaldır',strike:'Üstü Çizgili',subscript:'Alt Simge',superscript:'Üst Simge',horizontalrule:'Yatay Satır Ekle',pagebreak:'Sayfa Sonu Ekle',unlink:'Köprü Kaldır',undo:'Geri Al',redo:'Tekrarla',common:{browseServer:'Sunucuyu Gez',url:'URL',protocol:'Protokol',upload:'Karşıya Yükle',uploadSubmit:'Sunucuya Yolla',image:'Resim',flash:'Flash',form:'Form',checkbox:'Onay Kutusu',radio:'Seçenek Düğmesi',textField:'Metin Girişi',textarea:'Çok Satırlı Metin',hiddenField:'Gizli Veri',button:'Düğme',select:'Seçim Menüsü',imageButton:'Resimli Düğme',notSet:'<tanımlanmamış>',id:'Kimlik',name:'Ad',langDir:'Dil Yönü',langDirLtr:'Soldan Sağa (LTR)',langDirRtl:'Sağdan Sola (RTL)',langCode:'Dil Kodlaması',longDescr:'Uzun Tanımlı URL',cssClass:'Biçem Sayfası Sınıfları',advisoryTitle:'Danışma Başlığı',cssStyle:'Biçem',ok:'Tamam',cancel:'İptal',generalTab:'Genel',advancedTab:'Gelişmiş',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'Özel Karakter Ekle',title:'Özel Karakter Seç'},link:{toolbar:'Köprü Ekle/Düzenle',menu:'Köprü Düzenle',title:'Köprü',info:'Köprü Bilgisi',target:'Hedef',upload:'Karşıya Yükle',advanced:'Gelişmiş',type:'Köprü Türü',toAnchor:'Bu sayfada çapa',toEmail:'E-Posta',target:'Hedef',targetNotSet:'<tanımlanmamış>',targetFrame:'<çerçeve>',targetPopup:'<yeni açılan pencere>',targetNew:'Yeni Pencere(_blank)',targetTop:'En Üst Pencere (_top)',targetSelf:'Kendi Penceresi (_self)',targetParent:'Anne Pencere (_parent)',targetFrameName:'Hedef Çerçeve Adı',targetPopupName:'Yeni Açılan Pencere Adı',popupFeatures:'Yeni Açılan Pencere Özellikleri',popupResizable:'Resizable',popupStatusBar:'Durum Çubuğu',popupLocationBar:'Yer Çubuğu',popupToolbar:'Araç Çubuğu',popupMenuBar:'Menü Çubuğu',popupFullScreen:'Tam Ekran (IE)',popupScrollBars:'Kaydırma Çubukları',popupDependent:'Bağımlı (Netscape)',popupWidth:'Genişlik',popupLeft:'Sola Göre Konum',popupHeight:'Yükseklik',popupTop:'Yukarıya Göre Konum',id:'Id',langDir:'Dil Yönü',langDirNotSet:'<tanımlanmamış>',langDirLTR:'Soldan Sağa (LTR)',langDirRTL:'Sağdan Sola (RTL)',acccessKey:'Erişim Tuşu',name:'Ad',langCode:'Dil Yönü',tabIndex:'Sekme İndeksi',advisoryTitle:'Danışma Başlığı',advisoryContentType:'Danışma İçerik Türü',cssClasses:'Biçem Sayfası Sınıfları',charset:'Bağlı Kaynak Karakter Gurubu',styles:'Biçem',selectAnchor:'Çapa Seç',anchorName:'Çapa Adı ile',anchorId:'Eleman Kimlik Numarası ile',emailAddress:'E-Posta Adresi',emailSubject:'İleti Konusu',emailBody:'İleti Gövdesi',noAnchors:'(Bu belgede hiç çapa yok)',noUrl:"Lütfen köprü URL'sini yazın",noEmail:'Lütfen E-posta adresini yazın'},anchor:{toolbar:'Çapa Ekle/Düzenle',menu:'Çapa Özellikleri',title:'Çapa Özellikleri',name:'Çapa Adı',errorName:'Lütfen çapa için ad giriniz'},findAndReplace:{title:'Bul ve Değiştir',find:'Bul',replace:'Değiştir',findWhat:'Aranan:',replaceWith:'Bununla değiştir:',notFoundMsg:'Belirtilen yazı bulunamadı.',matchCase:'Büyük/küçük harf duyarlı',matchWord:'Kelimenin tamamı uysun',matchCyclic:'Match cyclic',replaceAll:'Tümünü Değiştir',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'Tablo',title:'Tablo Özellikleri',menu:'Tablo Özellikleri',deleteTable:'Tabloyu Sil',rows:'Satırlar',columns:'Sütunlar',border:'Kenar Kalınlığı',align:'Hizalama',alignNotSet:'<Tanımlanmamış>',alignLeft:'Sol',alignCenter:'Merkez',alignRight:'Sağ',width:'Genişlik',widthPx:'piksel',widthPc:'yüzde',height:'Yükseklik',cellSpace:'Izgara kalınlığı',cellPad:'Izgara yazı arası',caption:'Başlık',summary:'Özet',headers:'Başlıklar',headersNone:'Yok',headersColumn:'İlk Sütun',headersRow:'İlk Satır',headersBoth:'Her İkisi',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'Hücre',insertBefore:'Hücre Ekle - Önce',insertAfter:'Hücre Ekle - Sonra',deleteCell:'Hücre Sil',merge:'Hücreleri Birleştir',mergeRight:'Birleştir - Sağdaki İle ',mergeDown:'Birleştir - Aşağıdaki İle ',splitHorizontal:'Hücreyi Yatay Böl',splitVertical:'Hücreyi Dikey Böl',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'Satır',insertBefore:'Satır Ekle - Önce',insertAfter:'Satır Ekle - Sonra',deleteRow:'Satır Sil'},column:{menu:'Sütun',insertBefore:'Kolon Ekle - Önce',insertAfter:'Kolon Ekle - Sonra',deleteColumn:'Sütun Sil'}},button:{title:'Düğme Özellikleri',text:'Metin (Değer)',type:'Tip',typeBtn:'Düğme',typeSbm:'Gönder',typeRst:'Sıfırla'},checkboxAndRadio:{checkboxTitle:'Onay Kutusu Özellikleri',radioTitle:'Seçenek Düğmesi Özellikleri',value:'Değer',selected:'Seçili'},form:{title:'Form Özellikleri',menu:'Form Özellikleri',action:'İşlem',method:'Yöntem',encoding:'Encoding',target:'Hedef',targetNotSet:'<tanımlanmamış>',targetNew:'Yeni Pencere(_blank)',targetTop:'En Üst Pencere (_top)',targetSelf:'Kendi Penceresi (_self)',targetParent:'Anne Pencere (_parent)'},select:{title:'Seçim Menüsü Özellikleri',selectInfo:'Bilgi',opAvail:'Mevcut Seçenekler',value:'Değer',size:'Boyut',lines:'satır',chkMulti:'Çoklu seçime izin ver',opText:'Metin',opValue:'Değer',btnAdd:'Ekle',btnModify:'Düzenle',btnUp:'Yukarı',btnDown:'Aşağı',btnSetValue:'Seçili değer olarak ata',btnDelete:'Sil'},textarea:{title:'Çok Satırlı Metin Özellikleri',cols:'Sütunlar',rows:'Satırlar'},textfield:{title:'Metin Girişi Özellikleri',name:'Ad',value:'Değer',charWidth:'Karakter Genişliği',maxChars:'En Fazla Karakter',type:'Tür',typeText:'Metin',typePass:'Parola'},hidden:{title:'Gizli Veri Özellikleri',name:'Ad',value:'Değer'},image:{title:'Resim Özellikleri',titleButton:'Resimli Düğme Özellikleri',menu:'Resim Özellikleri',infoTab:'Resim Bilgisi',btnUpload:'Sunucuya Yolla',url:'URL',upload:'Karşıya Yükle',alt:'Alternatif Yazı',width:'Genişlik',height:'Yükseklik',lockRatio:'Oranı Kilitle',resetSize:'Boyutu Başa Döndür',border:'Kenar',hSpace:'Yatay Boşluk',vSpace:'Dikey Boşluk',align:'Hizalama',alignLeft:'Sol',alignAbsBottom:'Tam Altı',alignAbsMiddle:'Tam Ortası',alignBaseline:'Taban Çizgisi',alignBottom:'Alt',alignMiddle:'Orta',alignRight:'Sağ',alignTextTop:'Yazı Tepeye',alignTop:'Tepe',preview:'Ön İzleme',alertUrl:"Lütfen resmin URL'sini yazınız",linkTab:'Köprü',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'Flash Özellikleri',propertiesTab:'Properties',title:'Flash Özellikleri',chkPlay:'Otomatik Oynat',chkLoop:'Döngü',chkMenu:'Flash Menüsünü Kullan',chkFull:'Allow Fullscreen',scale:'Boyutlandır',scaleAll:'Hepsini Göster',scaleNoBorder:'Kenar Yok',scaleFit:'Tam Sığdır',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'Hizalama',alignLeft:'Sol',alignAbsBottom:'Tam Altı',alignAbsMiddle:'Tam Ortası',alignBaseline:'Taban Çizgisi',alignBottom:'Alt',alignMiddle:'Orta',alignRight:'Sağ',alignTextTop:'Yazı Tepeye',alignTop:'Tepe',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'Arka Renk',width:'Genişlik',height:'Yükseklik',hSpace:'Yatay Boşluk',vSpace:'Dikey Boşluk',validateSrc:"Lütfen köprü URL'sini yazın",validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'Yazım Denetimi',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'Sözlükte Yok',changeTo:'Şuna değiştir:',btnIgnore:'Yoksay',btnIgnoreAll:'Tümünü Yoksay',btnReplace:'Değiştir',btnReplaceAll:'Tümünü Değiştir',btnUndo:'Geri Al',noSuggestions:'- Öneri Yok -',progress:'Yazım denetimi işlemde...',noMispell:'Yazım denetimi tamamlandı: Yanlış yazıma rastlanmadı',noChanges:'Yazım denetimi tamamlandı: Hiçbir kelime değiştirilmedi',oneChange:'Yazım denetimi tamamlandı: Bir kelime değiştirildi',manyChanges:'Yazım denetimi tamamlandı: %1 kelime değiştirildi',ieSpellDownload:'Yazım denetimi yüklenmemiş. Şimdi yüklemek ister misiniz?'},smiley:{toolbar:'İfade',title:'İfade Ekle'},elementsPath:{eleTitle:'%1 element'},numberedlist:'Numaralı Liste',bulletedlist:'Simgeli Liste',indent:'Sekme Arttır',outdent:'Sekme Azalt',justify:{left:'Sola Dayalı',center:'Ortalanmış',right:'Sağa Dayalı',block:'İki Kenara Yaslanmış'},blockquote:'Blok Oluştur',clipboard:{title:'Yapıştır',cutError:'Gezgin yazılımınızın güvenlik ayarları düzenleyicinin otomatik kesme işlemine izin vermiyor. İşlem için (Ctrl+X) tuşlarını kullanın.',copyError:'Gezgin yazılımınızın güvenlik ayarları düzenleyicinin otomatik kopyalama işlemine izin vermiyor. İşlem için (Ctrl+C) tuşlarını kullanın.',pasteMsg:'Lütfen aşağıdaki kutunun içine yapıştırın. (<STRONG>Ctrl+V</STRONG>) ve <STRONG>Tamam</STRONG> butonunu tıklayın.',securityMsg:'Gezgin yazılımınızın güvenlik ayarları düzenleyicinin direkt olarak panoya erişimine izin vermiyor. Bu pencere içine tekrar yapıştırmalısınız..'},pastefromword:{toolbar:"Word'den Yapıştır",title:"Word'den Yapıştır",advice:'Lütfen aşağıdaki kutunun içine yapıştırın. (<STRONG>Ctrl+V</STRONG>) ve <STRONG>Tamam</STRONG> butonunu tıklayın.',ignoreFontFace:'Yazı Tipi tanımlarını yoksay',removeStyle:'Biçem Tanımlarını çıkar'},pasteText:{button:'Düz Metin Olarak Yapıştır',title:'Düz Metin Olarak Yapıştır'},templates:{button:'Şablonlar',title:'İçerik Şablonları',insertOption:'Mevcut içerik ile değiştir',selectPromptMsg:'Düzenleyicide açmak için lütfen bir şablon seçin.<br>(hali hazırdaki içerik kaybolacaktır.):',emptyListMsg:'(Belirli bir şablon seçilmedi)'},showBlocks:'Blokları Göster',stylesCombo:{label:'Biçem',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'Biçim',voiceLabel:'Format',panelTitle:'Biçim',panelVoiceLabel:'Select a paragraph format',tag_p:'Normal',tag_pre:'Biçimli',tag_address:'Adres',tag_h1:'Başlık 1',tag_h2:'Başlık 2',tag_h3:'Başlık 3',tag_h4:'Başlık 4',tag_h5:'Başlık 5',tag_h6:'Başlık 6',tag_div:'Paragraf (DIV)'},font:{label:'Yazı Türü',voiceLabel:'Font',panelTitle:'Yazı Türü',panelVoiceLabel:'Select a font'},fontSize:{label:'Boyut',voiceLabel:'Font Size',panelTitle:'Boyut',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'Yazı Rengi',bgColorTitle:'Arka Renk',auto:'Otomatik',more:'Diğer renkler...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
