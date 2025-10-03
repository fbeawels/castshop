# th.js

## Review

## 1. Summary  

The file is a **CKEditor language definition for Thai (th)**.  
It extends the `CKEDITOR.lang` namespace by adding a `th` object that contains all user‑interface strings used by the editor – buttons, dialog titles, tooltip texts, validation messages, etc.  
The code is pure JavaScript object literal syntax; there are no executable functions or side‑effects, so its sole purpose is to provide a dictionary of localized strings.

**Key components**  
| Component | Role |
|-----------|------|
| `CKEDITOR.lang.th` | Root object holding all Thai translations. |
| Sub‑objects (`common`, `link`, `image`, `table`, …) | Grouped translation strings for specific editor plugins or dialog sections. |
| Primitive string values | Actual localized text that CKEditor pulls at runtime. |

The file uses no design patterns beyond the common “module‑style” registration that CKEditor expects.

---

## 2. Detailed Description  

### Flow of execution  

1. **File loading** – CKEditor’s language loader imports this file during editor initialization (e.g., `CKEDITOR.lang = {}; CKEDITOR.lang.th = {...}`).
2. **Language selection** – When the editor is instantiated with `lang: 'th'`, CKEditor reads `CKEDITOR.lang.th` and replaces all UI strings accordingly.
3. **No runtime modification** – Once loaded, the object is immutable for the life of the editor instance; no cleanup is required.

### Architecture & Design Choices  

* **Flat namespace** – CKEditor uses a single global `CKEDITOR` object; language files populate it.  
* **Hierarchical grouping** – Strings are grouped into nested objects for readability and to match CKEditor's own dialog structure.  
* **String interpolation placeholders** – `%1`, `%2`, etc., are used for runtime substitution (e.g., “Rich text editor, %1”).  

The design is intentionally simple to minimize loading overhead and to allow translators to edit only the needed keys.

---

## 3. Functions/Methods  

The file contains **no executable functions** – it is purely data. The structure is a nested set of key/value pairs.

| Key | Description |
|-----|-------------|
| `dir` | Text direction (`'ltr'`). |
| `editorTitle` | Title template for the editor window. |
| `source`, `newPage`, `save`, … | Button/tooltip labels. |
| `common` | General UI strings (file upload, cancel/ok, form labels). |
| `link`, `image`, `table`, `button`, `checkboxAndRadio`, … | Plugin‑specific strings for dialogs. |
| `specialChar`, `findAndReplace`, `spellCheck`, … | Additional plugin support. |
| `colors`, `colordialog`, `showBlocks`, etc. | UI elements not tied to a single plugin. |

Because there are no methods, there are no inputs, outputs, or side‑effects to document.

---

## 4. Dependencies  

| Dependency | Nature | Notes |
|------------|--------|-------|
| `CKEDITOR` global | **Third‑party** | Provided by the CKEditor library. This file must be loaded after `ckeditor.js` or within the CKEditor package. |
| No external APIs or libraries | – | All strings are static. |

Platform‑specific assumptions: the file expects a browser environment that can execute JavaScript; no server‑side rendering is involved.

---

## 5. Additional Notes  

### Strengths  

* **Comprehensive coverage** – Covers the full set of core plugins that ship with CKEditor (e.g., link, image, table, smiley, paste‑from‑word, etc.).  
* **Consistent structure** – Mirrors CKEditor’s own language file format, making it easy for translators to locate keys.  

### Weaknesses / Issues  

1. **Inconsistent placeholder usage**  
   * Some strings use `%1`, others use `%2`. Ensure they match CKEditor’s interpolation logic.  
2. **Typos / Misspellings**  
   * e.g., `"columns": "สดมน์"` (should be `"columns": "คอลัมน์"`).  
   * `"lockRatio":"กำหนดอัตราส่วน กว้าง-สูง แบบคงที่"` – the word “อัตราส่วน” is repeated incorrectly.  
3. **Redundant or duplicate keys**  
   * `link` contains `target` twice with the same value.  
   * `link`’s `popupFeatures` list includes `"popupResizable"` but the string is missing a comma before it.  
4. **Missing translations**  
   * Some entries remain in English (`"ValidateNumberFailed":"This value is not a number."`) – should be translated.  
5. **Encoding / character issues**  
   * The file contains a hidden character at the very start (U+FEFF) – may cause issues in some editors.  
6. **Unescaped characters**  
   * In the `colors` object, some keys are quoted inconsistently (`'000'` vs `000`). While JS accepts both, consistency improves readability.  
7. **Hard‑coded URLs**  
   * Strings like `alertUrl` mention “เซิร์ฟเวอร์” which may be ambiguous for a Thai user. Consider clarifying with a local server reference.  

### Recommendations  

| Action | Description |
|--------|-------------|
| **Lint the JSON** | Run a JS linting tool (ESLint) to catch syntax errors, duplicate keys, and style inconsistencies. |
| **Translate all placeholders** | Replace any English placeholders with Thai equivalents; maintain `%1`, `%2` for dynamic data. |
| **Run a spell‑check** | Use a Thai language checker to correct typos. |
| **Validate against CKEditor docs** | Cross‑check that all required keys for active plugins are present. |
| **Remove BOM** | Ensure the file starts with plain ASCII; strip any UTF‑8 BOM. |
| **Add unit tests** | If possible, write tests that load the language file and verify that key strings are non‑empty and contain expected placeholders. |
| **Documentation** | Add a header comment explaining the file’s purpose, required CKEditor version, and any custom notes (e.g., “translated for Thai language”). |

With these improvements, the language file will be more reliable, easier to maintain, and provide a better user experience for Thai‑speaking CKEditor users.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.th={dir:'ltr',editorTitle:'Rich text editor, %1',source:'ดูรหัส HTML',newPage:'สร้างหน้าเอกสารใหม่',save:'บันทึก',preview:'ดูหน้าเอกสารตัวอย่าง',cut:'ตัด',copy:'สำเนา',paste:'วาง',print:'สั่งพิมพ์',underline:'ตัวขีดเส้นใต้',bold:'ตัวหนา',italic:'ตัวเอียง',selectAll:'เลือกทั้งหมด',removeFormat:'ล้างรูปแบบ',strike:'ตัวขีดเส้นทับ',subscript:'ตัวห้อย',superscript:'ตัวยก',horizontalrule:'แทรกเส้นคั่นบรรทัด',pagebreak:'แทรกตัวแบ่งหน้า Page Break',unlink:'ลบ ลิงค์',undo:'ยกเลิกคำสั่ง',redo:'ทำซ้ำคำสั่ง',common:{browseServer:'เปิดหน้าต่างจัดการไฟล์อัพโหลด',url:'ที่อยู่อ้างอิง URL',protocol:'โปรโตคอล',upload:'อัพโหลดไฟล์',uploadSubmit:'อัพโหลดไฟล์ไปเก็บไว้ที่เครื่องแม่ข่าย (เซิร์ฟเวอร์)',image:'รูปภาพ',flash:'ไฟล์ Flash',form:'แบบฟอร์ม',checkbox:'เช็คบ๊อก',radio:'เรดิโอบัตตอน',textField:'เท็กซ์ฟิลด์',textarea:'เท็กซ์แอเรีย',hiddenField:'ฮิดเดนฟิลด์',button:'ปุ่ม',select:'แถบตัวเลือก',imageButton:'ปุ่มแบบรูปภาพ',notSet:'<ไม่ระบุ>',id:'ไอดี',name:'ชื่อ',langDir:'การเขียน-อ่านภาษา',langDirLtr:'จากซ้ายไปขวา (LTR)',langDirRtl:'จากขวามาซ้าย (RTL)',langCode:'รหัสภาษา',longDescr:'คำอธิบายประกอบ URL',cssClass:'คลาสของไฟล์กำหนดลักษณะการแสดงผล',advisoryTitle:'คำเกริ่นนำ',cssStyle:'ลักษณะการแสดงผล',ok:'ตกลง',cancel:'ยกเลิก',generalTab:'General',advancedTab:'ขั้นสูง',validateNumberFailed:'This value is not a number.',confirmNewPage:'Any unsaved changes to this content will be lost. Are you sure you want to load new page?',confirmCancel:'Some of the options have been changed. Are you sure to close the dialog?',unavailable:'%1<span class="cke_accessibility">, unavailable</span>'},specialChar:{toolbar:'แทรกตัวอักษรพิเศษ',title:'แทรกตัวอักษรพิเศษ'},link:{toolbar:'แทรก/แก้ไข ลิงค์',menu:'แก้ไข ลิงค์',title:'ลิงค์เชื่อมโยงเว็บ อีเมล์ รูปภาพ หรือไฟล์อื่นๆ',info:'รายละเอียด',target:'การเปิดหน้าลิงค์',upload:'อัพโหลดไฟล์',advanced:'ขั้นสูง',type:'ประเภทของลิงค์',toAnchor:'จุดเชื่อมโยง (Anchor)',toEmail:'ส่งอีเมล์ (E-Mail)',target:'การเปิดหน้าลิงค์',targetNotSet:'<ไม่ระบุ>',targetFrame:'<เปิดในเฟรม>',targetPopup:'<เปิดหน้าจอเล็ก (Pop-up)>',targetNew:'เปิดหน้าจอใหม่ (_blank)',targetTop:'เปิดในหน้าบนสุด (_top)',targetSelf:'เปิดในหน้าปัจจุบัน (_self)',targetParent:'เปิดในหน้าหลัก (_parent)',targetFrameName:'ชื่อทาร์เก็ตเฟรม',targetPopupName:'ระบุชื่อหน้าจอเล็ก (Pop-up)',popupFeatures:'คุณสมบัติของหน้าจอเล็ก (Pop-up)',popupResizable:'Resizable',popupStatusBar:'แสดงแถบสถานะ',popupLocationBar:'แสดงที่อยู่ของไฟล์',popupToolbar:'แสดงแถบเครื่องมือ',popupMenuBar:'แสดงแถบเมนู',popupFullScreen:'แสดงเต็มหน้าจอ (IE5.5++ เท่านั้น)',popupScrollBars:'แสดงแถบเลื่อน',popupDependent:'แสดงเต็มหน้าจอ (Netscape)',popupWidth:'กว้าง',popupLeft:'พิกัดซ้าย (Left Position)',popupHeight:'สูง',popupTop:'พิกัดบน (Top Position)',id:'Id',langDir:'การเขียน-อ่านภาษา',langDirNotSet:'<ไม่ระบุ>',langDirLTR:'จากซ้ายไปขวา (LTR)',langDirRTL:'จากขวามาซ้าย (RTL)',acccessKey:'แอคเซส คีย์',name:'ชื่อ',langCode:'การเขียน-อ่านภาษา',tabIndex:'ลำดับของ แท็บ',advisoryTitle:'คำเกริ่นนำ',advisoryContentType:'ชนิดของคำเกริ่นนำ',cssClasses:'คลาสของไฟล์กำหนดลักษณะการแสดงผล',charset:'ลิงค์เชื่อมโยงไปยังชุดตัวอักษร',styles:'ลักษณะการแสดงผล',selectAnchor:'ระบุข้อมูลของจุดเชื่อมโยง (Anchor)',anchorName:'ชื่อ',anchorId:'ไอดี',emailAddress:'อีเมล์ (E-Mail)',emailSubject:'หัวเรื่อง',emailBody:'ข้อความ',noAnchors:'(ยังไม่มีจุดเชื่อมโยงภายในหน้าเอกสารนี้)',noUrl:'กรุณาระบุที่อยู่อ้างอิงออนไลน์ (URL)',noEmail:'กรุณาระบุอีเมล์ (E-mail)'},anchor:{toolbar:'แทรก/แก้ไข Anchor',menu:'รายละเอียด Anchor',title:'รายละเอียด Anchor',name:'ชื่อ Anchor',errorName:'กรุณาระบุชื่อของ Anchor'},findAndReplace:{title:'Find and Replace',find:'ค้นหา',replace:'ค้นหาและแทนที่',findWhat:'ค้นหาคำว่า:',replaceWith:'แทนที่ด้วย:',notFoundMsg:'ไม่พบคำที่ค้นหา.',matchCase:'ตัวโหญ่-เล็ก ต้องตรงกัน',matchWord:'ต้องตรงกันทุกคำ',matchCyclic:'Match cyclic',replaceAll:'แทนที่ทั้งหมดที่พบ',replaceSuccessMsg:'%1 occurrence(s) replaced.'},table:{toolbar:'ตาราง',title:'คุณสมบัติของ ตาราง',menu:'คุณสมบัติของ ตาราง',deleteTable:'ลบตาราง',rows:'แถว',columns:'สดมน์',border:'ขนาดเส้นขอบ',align:'การจัดตำแหน่ง',alignNotSet:'<ไม่ระบุ>',alignLeft:'ชิดซ้าย',alignCenter:'กึ่งกลาง',alignRight:'ชิดขวา',width:'กว้าง',widthPx:'จุดสี',widthPc:'เปอร์เซ็น',height:'สูง',cellSpace:'ระยะแนวนอนน',cellPad:'ระยะแนวตั้ง',caption:'หัวเรื่องของตาราง',summary:'สรุปความ',headers:'Headers',headersNone:'None',headersColumn:'First column',headersRow:'First Row',headersBoth:'Both',invalidRows:'Number of rows must be a number greater than 0.',invalidCols:'Number of columns must be a number greater than 0.',invalidBorder:'Border size must be a number.',invalidWidth:'Table width must be a number.',invalidHeight:'Table height must be a number.',invalidCellSpacing:'Cell spacing must be a number.',invalidCellPadding:'Cell padding must be a number.',cell:{menu:'ช่องตาราง',insertBefore:'Insert Cell Before',insertAfter:'Insert Cell After',deleteCell:'ลบช่อง',merge:'ผสานช่อง',mergeRight:'Merge Right',mergeDown:'Merge Down',splitHorizontal:'Split Cell Horizontally',splitVertical:'Split Cell Vertically',title:'Cell Properties',cellType:'Cell Type',rowSpan:'Rows Span',colSpan:'Columns Span',wordWrap:'Word Wrap',hAlign:'Horizontal Alignment',vAlign:'Vertical Alignment',alignTop:'Top',alignMiddle:'Middle',alignBottom:'Bottom',alignBaseline:'Baseline',bgColor:'Background Color',borderColor:'Border Color',data:'Data',header:'Header',yes:'Yes',no:'No',invalidWidth:'Cell width must be a number.',invalidHeight:'Cell height must be a number.',invalidRowSpan:'Rows span must be a whole number.',invalidColSpan:'Columns span must be a whole number.',chooseColor:'Choose'},row:{menu:'แถว',insertBefore:'Insert Row Before',insertAfter:'Insert Row After',deleteRow:'ลบแถว'},column:{menu:'คอลัมน์',insertBefore:'Insert Column Before',insertAfter:'Insert Column After',deleteColumn:'ลบสดมน์'}},button:{title:'รายละเอียดของ ปุ่ม',text:'ข้อความ (ค่าตัวแปร)',type:'ข้อความ',typeBtn:'Button',typeSbm:'Submit',typeRst:'Reset'},checkboxAndRadio:{checkboxTitle:'คุณสมบัติของ เช็คบ๊อก',radioTitle:'คุณสมบัติของ เรดิโอบัตตอน',value:'ค่าตัวแปร',selected:'เลือกเป็นค่าเริ่มต้น'},form:{title:'คุณสมบัติของ แบบฟอร์ม',menu:'คุณสมบัติของ แบบฟอร์ม',action:'แอคชั่น',method:'เมธอด',encoding:'Encoding',target:'การเปิดหน้าลิงค์',targetNotSet:'<ไม่ระบุ>',targetNew:'เปิดหน้าจอใหม่ (_blank)',targetTop:'เปิดในหน้าบนสุด (_top)',targetSelf:'เปิดในหน้าปัจจุบัน (_self)',targetParent:'เปิดในหน้าหลัก (_parent)'},select:{title:'คุณสมบัติของ แถบตัวเลือก',selectInfo:'อินโฟ',opAvail:'รายการตัวเลือก',value:'ค่าตัวแปร',size:'ขนาด',lines:'บรรทัด',chkMulti:'เลือกหลายค่าได้',opText:'ข้อความ',opValue:'ค่าตัวแปร',btnAdd:'เพิ่ม',btnModify:'แก้ไข',btnUp:'บน',btnDown:'ล่าง',btnSetValue:'เลือกเป็นค่าเริ่มต้น',btnDelete:'ลบ'},textarea:{title:'คุณสมบัติของ เท็กแอเรีย',cols:'สดมภ์',rows:'แถว'},textfield:{title:'คุณสมบัติของ เท็กซ์ฟิลด์',name:'ชื่อ',value:'ค่าตัวแปร',charWidth:'ความกว้าง',maxChars:'จำนวนตัวอักษรสูงสุด',type:'ชนิด',typeText:'ข้อความ',typePass:'รหัสผ่าน'},hidden:{title:'คุณสมบัติของ ฮิดเดนฟิลด์',name:'ชื่อ',value:'ค่าตัวแปร'},image:{title:'คุณสมบัติของ รูปภาพ',titleButton:'คุณสมบัติของ ปุ่มแบบรูปภาพ',menu:'คุณสมบัติของ รูปภาพ',infoTab:'ข้อมูลของรูปภาพ',btnUpload:'อัพโหลดไฟล์ไปเก็บไว้ที่เครื่องแม่ข่าย (เซิร์ฟเวอร์)',url:'ที่อยู่อ้างอิง URL',upload:'อัพโหลดไฟล์',alt:'คำประกอบรูปภาพ',width:'ความกว้าง',height:'ความสูง',lockRatio:'กำหนดอัตราส่วน กว้าง-สูง แบบคงที่',resetSize:'กำหนดรูปเท่าขนาดจริง',border:'ขนาดขอบรูป',hSpace:'ระยะแนวนอน',vSpace:'ระยะแนวตั้ง',align:'การจัดวาง',alignLeft:'ชิดซ้าย',alignAbsBottom:'ชิดด้านล่างสุด',alignAbsMiddle:'กึ่งกลาง',alignBaseline:'ชิดบรรทัด',alignBottom:'ชิดด้านล่าง',alignMiddle:'กึ่งกลางแนวตั้ง',alignRight:'ชิดขวา',alignTextTop:'ใต้ตัวอักษร',alignTop:'บนสุด',preview:'หน้าเอกสารตัวอย่าง',alertUrl:'กรุณาระบุที่อยู่อ้างอิงออนไลน์ของไฟล์รูปภาพ (URL)',linkTab:'ลิ้งค์',button2Img:'Do you want to transform the selected image button on a simple image?',img2Button:'Do you want to transform the selected image on a image button?',urlMissing:'Image source URL is missing.'},flash:{properties:'คุณสมบัติของไฟล์ Flash',propertiesTab:'Properties',title:'คุณสมบัติของไฟล์ Flash',chkPlay:'เล่นอัตโนมัติ Auto Play',chkLoop:'เล่นวนรอบ Loop',chkMenu:'ให้ใช้งานเมนูของ Flash',chkFull:'Allow Fullscreen',scale:'อัตราส่วน Scale',scaleAll:'แสดงให้เห็นทั้งหมด Show all',scaleNoBorder:'ไม่แสดงเส้นขอบ No Border',scaleFit:'แสดงให้พอดีกับพื้นที่ Exact Fit',access:'Script Access',accessAlways:'Always',accessSameDomain:'Same domain',accessNever:'Never',align:'การจัดวาง',alignLeft:'ชิดซ้าย',alignAbsBottom:'ชิดด้านล่างสุด',alignAbsMiddle:'กึ่งกลาง',alignBaseline:'ชิดบรรทัด',alignBottom:'ชิดด้านล่าง',alignMiddle:'กึ่งกลางแนวตั้ง',alignRight:'ชิดขวา',alignTextTop:'ใต้ตัวอักษร',alignTop:'บนสุด',quality:'Quality',qualityBest:'Best',qualityHigh:'High',qualityAutoHigh:'Auto High',qualityMedium:'Medium',qualityAutoLow:'Auto Low',qualityLow:'Low',windowModeWindow:'Window',windowModeOpaque:'Opaque',windowModeTransparent:'Transparent',windowMode:'Window mode',flashvars:'Variables for Flash',bgcolor:'สีพื้นหลัง',width:'ความกว้าง',height:'ความสูง',hSpace:'ระยะแนวนอน',vSpace:'ระยะแนวตั้ง',validateSrc:'กรุณาระบุที่อยู่อ้างอิงออนไลน์ (URL)',validateWidth:'Width must be a number.',validateHeight:'Height must be a number.',validateHSpace:'HSpace must be a number.',validateVSpace:'VSpace must be a number.'},spellCheck:{toolbar:'ตรวจการสะกดคำ',title:'Spell Check',notAvailable:'Sorry, but service is unavailable now.',errorLoading:'Error loading application service host: %s.',notInDic:'ไม่พบในดิกชันนารี',changeTo:'แก้ไขเป็น',btnIgnore:'ยกเว้น',btnIgnoreAll:'ยกเว้นทั้งหมด',btnReplace:'แทนที่',btnReplaceAll:'แทนที่ทั้งหมด',btnUndo:'ยกเลิก',noSuggestions:'- ไม่มีคำแนะนำใดๆ -',progress:'กำลังตรวจสอบคำสะกด...',noMispell:'ตรวจสอบคำสะกดเสร็จสิ้น: ไม่พบคำสะกดผิด',noChanges:'ตรวจสอบคำสะกดเสร็จสิ้น: ไม่มีการแก้คำใดๆ',oneChange:'ตรวจสอบคำสะกดเสร็จสิ้น: แก้ไข1คำ',manyChanges:'ตรวจสอบคำสะกดเสร็จสิ้น:: แก้ไข %1 คำ',ieSpellDownload:'ไม่ได้ติดตั้งระบบตรวจสอบคำสะกด. ต้องการติดตั้งไหมครับ?'},smiley:{toolbar:'รูปสื่ออารมณ์',title:'แทรกสัญลักษณ์สื่ออารมณ์'},elementsPath:{eleTitle:'%1 element'},numberedlist:'ลำดับรายการแบบตัวเลข',bulletedlist:'ลำดับรายการแบบสัญลักษณ์',indent:'เพิ่มระยะย่อหน้า',outdent:'ลดระยะย่อหน้า',justify:{left:'จัดชิดซ้าย',center:'จัดกึ่งกลาง',right:'จัดชิดขวา',block:'จัดพอดีหน้ากระดาษ'},blockquote:'Blockquote',clipboard:{title:'วาง',cutError:'ไม่สามารถตัดข้อความที่เลือกไว้ได้เนื่องจากการกำหนดค่าระดับความปลอดภัย. กรุณาใช้ปุ่มลัดเพื่อวางข้อความแทน (กดปุ่ม Ctrl และตัว X พร้อมกัน).',copyError:'ไม่สามารถสำเนาข้อความที่เลือกไว้ได้เนื่องจากการกำหนดค่าระดับความปลอดภัย. กรุณาใช้ปุ่มลัดเพื่อวางข้อความแทน (กดปุ่ม Ctrl และตัว C พร้อมกัน).',pasteMsg:'กรุณาใช้คีย์บอร์ดเท่านั้น โดยกดปุ๋ม (<strong>Ctrl และ V</strong>)พร้อมๆกัน และกด <strong>OK</strong>.',securityMsg:'Because of your browser security settings, the editor is not able to access your clipboard data directly. You are required to paste it again in this window.'},pastefromword:{toolbar:'วางสำเนาจากตัวอักษรเวิร์ด',title:'วางสำเนาจากตัวอักษรเวิร์ด',advice:'กรุณาใช้คีย์บอร์ดเท่านั้น โดยกดปุ๋ม (<strong>Ctrl และ V</strong>)พร้อมๆกัน และกด <strong>OK</strong>.',ignoreFontFace:'ไม่สนใจ Font Face definitions',removeStyle:'ลบ Styles definitions'},pasteText:{button:'วางแบบตัวอักษรธรรมดา',title:'วางแบบตัวอักษรธรรมดา'},templates:{button:'เทมเพลต',title:'เทมเพลตของส่วนเนื้อหาเว็บไซต์',insertOption:'แทนที่เนื้อหาเว็บไซต์ที่เลือก',selectPromptMsg:'กรุณาเลือก เทมเพลต เพื่อนำไปแก้ไขในอีดิตเตอร์<br />(เนื้อหาส่วนนี้จะหายไป):',emptyListMsg:'(ยังไม่มีการกำหนดเทมเพลต)'},showBlocks:'Show Blocks',stylesCombo:{label:'ลักษณะ',voiceLabel:'Styles',panelVoiceLabel:'Select a style',panelTitle1:'Block Styles',panelTitle2:'Inline Styles',panelTitle3:'Object Styles'},format:{label:'รูปแบบ',voiceLabel:'Format',panelTitle:'รูปแบบ',panelVoiceLabel:'Select a paragraph format',tag_p:'Normal',tag_pre:'Formatted',tag_address:'Address',tag_h1:'Heading 1',tag_h2:'Heading 2',tag_h3:'Heading 3',tag_h4:'Heading 4',tag_h5:'Heading 5',tag_h6:'Heading 6',tag_div:'Paragraph (DIV)'},font:{label:'แบบอักษร',voiceLabel:'Font',panelTitle:'แบบอักษร',panelVoiceLabel:'Select a font'},fontSize:{label:'ขนาด',voiceLabel:'Font Size',panelTitle:'ขนาด',panelVoiceLabel:'Select a font size'},colorButton:{textColorTitle:'สีตัวอักษร',bgColorTitle:'สีพื้นหลัง',auto:'สีอัตโนมัติ',more:'เลือกสีอื่นๆ...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Spell Check As You Type',enable:'Enable SCAYT',disable:'Disable SCAYT',about:'About SCAYT',toggle:'Toggle SCAYT',options:'Options',langs:'Languages',moreSuggestions:'More suggestions',ignore:'Ignore',ignoreAll:'Ignore All',addWord:'Add Word',emptyDic:'Dictionary name should not be empty.',optionsTab:'Options',languagesTab:'Languages',dictionariesTab:'Dictionaries',aboutTab:'About'},about:{title:'About CKEditor',dlgTitle:'About CKEditor',moreInfo:'For licensing information please visit our web site:',copy:'Copyright &copy; $1. All rights reserved.'},maximize:'Maximize',minimize:'Minimize',fakeobjects:{anchor:'Anchor',flash:'Flash Animation',div:'Page Break',unknown:'Unknown Object'},resize:'Drag to resize',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
