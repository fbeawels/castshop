# vi.js

## Review

## 1. Summary  

**Purpose**  
This file supplies the Vietnamese localisation for CKEditor – a rich‑text WYSIWYG editor.  
It is a single JavaScript object (`CKEDITOR.lang.vi`) containing all UI text strings (tooltips, button labels, dialog titles, validation messages, colour palette, etc.) used by CKEditor when the editor’s language is set to Vietnamese.

**Key components**  

| Section | Role |
|---------|------|
| `dir` | Text direction (`ltr` – left‑to‑right). |
| `editorTitle`, `source`, `newPage`, … | Top‑level UI strings. |
| `common`, `specialChar`, `link`, `anchor`, `findAndReplace`, `table`, … | Nested objects that group strings by feature (dialogs, toolbars, form controls). |
| `colors`, `scayt`, `about`, `maximize`, `minimize`, … | Additional feature‑specific localisation blocks. |
| `colordialog` | Colour picker strings. |

**Design patterns / libraries**  
- **Module pattern** – the file simply attaches an object to the global `CKEDITOR.lang` namespace.  
- **No external libraries** – it is pure data, no runtime dependencies.  
- **JSON‑like object literal** – the syntax is standard ECMAScript, so any browser or Node environment can parse it.  

---

## 2. Detailed Description  

### Flow of execution  

1. **File inclusion** – CKEditor loads this file when the user selects the Vietnamese language (usually via `lang="vi"` attribute or programmatic language switch).  
2. **Object assignment** – The script executes and assigns the large object literal to `CKEDITOR.lang.vi`.  
3. **Runtime usage** – Throughout CKEditor, localisation keys such as `CKEDITOR.lang.vi.common.browseServer` are read and inserted into UI elements.  
4. **No cleanup** – The file contains no teardown logic; once loaded, the object stays in memory until the page unloads.

### Assumptions & constraints  

| Assumption | Constraint |
|------------|------------|
| `CKEDITOR.lang` already exists | The script must be executed after CKEditor’s core has been loaded. |
| All keys match CKEditor’s localisation lookup | If a key is missing or misspelled, the UI falls back to English. |
| UTF‑8 (or similar) encoding | The file contains Vietnamese characters; incorrect encoding would corrupt the strings. |
| Browser compatibility | The object literal uses only ECMAScript 3 features, so it is safe for all browsers that support CKEditor. |

### Architecture & design choices  

- **Flat namespace** – A single global variable (`CKEDITOR.lang.vi`) avoids polluting the global scope with many separate files.  
- **Nested grouping** – Features are grouped inside nested objects to keep related strings together and to mirror CKEditor’s internal i18n lookup paths.  
- **Hard‑coded colour table** – The palette is listed as an object mapping hex codes to colour names; this matches CKEditor’s expectations.  
- **Minimal code** – As a pure data file, it contains no functions, which keeps the bundle lightweight.  

---

## 3. Functions/Methods  

The file **does not** declare any functions or methods.  
All content is declarative; it simply provides an object that the CKEditor core references.  
If a reviewer expects a “functions/methods” section, it would be empty.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Global object | Must be defined before this script runs. |
| None other |  | No third‑party libraries are required. |

Platform‑specific constraints:  
- The file assumes a UTF‑8 (or compatible) encoding; on Windows some text editors might add a BOM which CKEditor tolerates but can cause subtle issues if the server sets the wrong MIME header.  

---

## 5. Additional Notes & Recommendations  

### 5.1 Encoding & File Integrity  
- Ensure the file is saved in **UTF‑8 without BOM** to avoid hidden characters that could break parsing in some environments.  
- Run a quick linter (`eslint --config=...`) to catch stray stray quotes or missing commas.  

### 5.2 Maintainability  
- **Automated extraction** – If the project grows, consider generating localisation files from a source (e.g., PO, JSON, or CSV) rather than editing raw JavaScript.  
- **Version control** – Tag the localisation file with the CKEditor version so that future upgrades are traceable.  

### 5.3 Completeness & Accuracy  
- **Missing translations** – A few strings use English placeholders (`<không thiết lập>`, `<khung>`) – double‑check that these are intentional and that there is no more natural Vietnamese rendering.  
- **Plural forms & gender** – CKEditor doesn’t support pluralization in this file, but if you add such strings, you’ll need a more advanced i18n system (e.g., ICU MessageFormat).  

### 5.4 Performance  
- Because the file is large, it may increase the initial payload.  
  - **Lazy load**: CKEditor already supports loading language files on demand; ensure this file is only requested when the user selects Vietnamese.  
  - **Compression**: Serve it gzipped; the content is already static so HTTP compression is highly effective.  

### 5.5 Future Enhancements  
- **Right‑to‑left (RTL) support** – Add a separate `CKEDITOR.lang.vi-rtl` if Vietnamese texts ever need RTL layout (unlikely, but good for consistency).  
- **Dynamic colour palette** – Allow developers to override the default palette via a configuration flag.  
- **Spell‑checker localization** – The SCAYT block is included; verify that the `languagesTab` strings are in sync with the SCAYT plugin’s expectations.  

### 5.6 Edge Cases  
- **Undefined keys** – If CKEditor attempts to access a key that is missing, it will fall back to English. Verify that all required keys are present; missing ones can break UI or show untranslated fragments.  
- **Browser quirks** – Older browsers that do not support Unicode property escapes or advanced string methods are not affected, because the file only defines static strings.  

---

**Verdict** – The file is structurally sound, follows CKEditor’s localisation conventions, and contains no runtime errors. Minor improvements in encoding hygiene and documentation would make it more robust, but overall it is a solid localisation module.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.vi={dir:'ltr',editorTitle:'Trình biên tập trực quan, %1',source:'Mã HTML',newPage:'Trang mới',save:'Lưu',preview:'Xem trước',cut:'Cắt',copy:'Sao chép',paste:'Dán',print:'In',underline:'Gạch chân',bold:'Đậm',italic:'Nghiêng',selectAll:'Chọn Tất cả',removeFormat:'Xoá Định dạng',strike:'Gạch xuyên ngang',subscript:'Chỉ số dưới',superscript:'Chỉ số trên',horizontalrule:'Chèn Đường phân cách ngang',pagebreak:'Chèn Ngắt trang',unlink:'Xoá Liên kết',undo:'Khôi phục thao tác',redo:'Làm lại thao tác',common:{browseServer:'Duyệt trên máy chủ',url:'URL',protocol:'Giao thức',upload:'Tải lên',uploadSubmit:'Tải lên Máy chủ',image:'Hình ảnh',flash:'Flash',form:'Biểu mẫu',checkbox:'Nút kiểm',radio:'Nút chọn',textField:'Trường văn bản',textarea:'Vùng văn bản',hiddenField:'Trường ẩn',button:'Nút',select:'Ô chọn',imageButton:'Nút hình ảnh',notSet:'<không thiết lập>',id:'Định danh',name:'Tên',langDir:'Đường dẫn Ngôn ngữ',langDirLtr:'Trái sang Phải (LTR)',langDirRtl:'Phải sang Trái (RTL)',langCode:'Mã Ngôn ngữ',longDescr:'Mô tả URL',cssClass:'Lớp Stylesheet',advisoryTitle:'Advisory Title',cssStyle:'Mẫu',ok:'Đồng ý',cancel:'Bỏ qua',generalTab:'Chung',advancedTab:'Mở rộng',validateNumberFailed:'Giá trị này không phải là số.',confirmNewPage:'Mọi thay đổi không được không được lưu lại của nội dung này sẽ bị mất. Bạn có chắc chắn muốn tải một trang mới?',confirmCancel:'Một vài tùy chọn đã bị thay đổi. Bạn có chắc chắn muốn đóng hộp thoại?',unavailable:'%1<span class="cke_accessibility">, không có</span>'},specialChar:{toolbar:'Chèn Ký tự đặc biệt',title:'Hãy chọn Ký tự đặc biệt'},link:{toolbar:'Chèn/Sửa Liên kết',menu:'Sửa Liên kết',title:'Liên kết',info:'Thông tin Liên kết',target:'Đích',upload:'Tải lên',advanced:'Mở rộng',type:'Kiểu Liên kết',toAnchor:'Neo trong trang này',toEmail:'Thư điện tử',target:'Đích',targetNotSet:'<không thiết lập>',targetFrame:'<khung>',targetPopup:'<cửa sổ popup>',targetNew:'Cửa sổ mới (_blank)',targetTop:'Cửa sổ trên cùng(_top)',targetSelf:'Cùng cửa sổ (_self)',targetParent:'Cửa sổ cha (_parent)',targetFrameName:'Tên Khung đích',targetPopupName:'Tên Cửa sổ Popup',popupFeatures:'Đặc điểm của Cửa sổ Popup',popupResizable:'Có thể thay đổi kích cỡ',popupStatusBar:'Thanh trạng thái',popupLocationBar:'Thanh vị trí',popupToolbar:'Thanh công cụ',popupMenuBar:'Thanh Menu',popupFullScreen:'Toàn màn hình (IE)',popupScrollBars:'Thanh cuộn',popupDependent:'Phụ thuộc (Netscape)',popupWidth:'Rộng',popupLeft:'Vị trí Trái',popupHeight:'Cao',popupTop:'Vị trí Trên',id:'Định danh',langDir:'Đường dẫn Ngôn ngữ',langDirNotSet:'<không thiết lập>',langDirLTR:'Trái sang Phải (LTR)',langDirRTL:'Phải sang Trái (RTL)',acccessKey:'Phím Hỗ trợ truy cập',name:'Tên',langCode:'Đường dẫn Ngôn ngữ',tabIndex:'Chỉ số của Tab',advisoryTitle:'Advisory Title',advisoryContentType:'Advisory Content Type',cssClasses:'Lớp Stylesheet',charset:'Bảng mã của tài nguyên được liên kết đến',styles:'Mẫu',selectAnchor:'Chọn một Neo',anchorName:'Theo Tên Neo',anchorId:'Theo Định danh Thành phần',emailAddress:'Thư điện tử',emailSubject:'Tiêu đề Thông điệp',emailBody:'Nội dung Thông điệp',noAnchors:'(Không có Neo nào trong tài liệu)',noUrl:'Hãy đưa vào Liên kết URL',noEmail:'Hãy đưa vào địa chỉ thư điện tử'},anchor:{toolbar:'Chèn/Sửa Neo',menu:'Thuộc tính Neo',title:'Thuộc tính Neo',name:'Tên của Neo',errorName:'Hãy nhập vào tên của Neo'},findAndReplace:{title:'Tìm kiếm và Thay Thế',find:'Tìm kiếm',replace:'Thay thế',findWhat:'Tìm chuỗi:',replaceWith:'Thay bằng:',notFoundMsg:'Không tìm thấy chuỗi cần tìm.',matchCase:'Phân biệt chữ hoa/thường',matchWord:'Giống toàn bộ từ',matchCyclic:'Giống một phần',replaceAll:'Thay thế Tất cả',replaceSuccessMsg:'%1 vị trí đã được thay thế.'},table:{toolbar:'Bảng',title:'Thuộc tính bảng',menu:'Thuộc tính bảng',deleteTable:'Xóa Bảng',rows:'Hàng',columns:'Cột',border:'Cỡ Đường viền',align:'Canh lề',alignNotSet:'<Chưa thiết lập>',alignLeft:'Trái',alignCenter:'Giữa',alignRight:'Phải',width:'Rộng',widthPx:'điểm (px)',widthPc:'%',height:'Cao',cellSpace:'Khoảng cách Ô',cellPad:'Đệm Ô',caption:'Đầu đề',summary:'Tóm lược',headers:'Đầu đề',headersNone:'Không có',headersColumn:'Cột Đầu tiên',headersRow:'Hàng Đầu tiên',headersBoth:'Cả hai',invalidRows:'Số lượng hàng phải là một số lớn hơn 0.',invalidCols:'Số lượng cột phải là một số lớn hơn 0.',invalidBorder:'Kích cỡ của đường biên phải là một số nguyên.',invalidWidth:'Chiều rộng của Bảng phải là một số nguyên.',invalidHeight:'Chiều cao của Bảng phải là một số nguyên.',invalidCellSpacing:'Khoảng cách giữa các ô phải là một số nguyên.',invalidCellPadding:'Đệm giữa các ô phải là một số nguyên.',cell:{menu:'Ô',insertBefore:'Chèn Ô Phía trước',insertAfter:'Chèn Ô Phía sau',deleteCell:'Xoá Ô',merge:'Kết hợp Ô',mergeRight:'Kết hợp Sang phải',mergeDown:'Kết hợp Xuống dưới',splitHorizontal:'Tách ngang Ô',splitVertical:'Tách dọc Ô',title:'Thuộc tính của Ô',cellType:'Kiểu của Ô',rowSpan:'Kết hợp hàng',colSpan:'Kết hợp cột',wordWrap:'Word Wrap',hAlign:'Canh lề ngang',vAlign:'Canh lề dọc',alignTop:'Trên cùng',alignMiddle:'Chính giữa',alignBottom:'Dưới cùng',alignBaseline:'Đường cơ sở',bgColor:'Màu nền',borderColor:'Màu viền',data:'Dữ liệu',header:'Đầu đề',yes:'Có',no:'Không',invalidWidth:'Chiều rộng của Ô phải là một số nguyên.',invalidHeight:'Chiều cao của Ô phải là một số nguyên.',invalidRowSpan:'Số hàng kết hợp phải là một số nguyên.',invalidColSpan:'Số cột kết hợp phải là một số nguyên.',chooseColor:'Choose'},row:{menu:'Hàng',insertBefore:'Chèn Hàng Phía trước',insertAfter:'Chèn Hàng Phía sau',deleteRow:'Xoá Hàng'},column:{menu:'Cột',insertBefore:'Chèn Cột Phía trước',insertAfter:'Chèn Cột Phía sau',deleteColumn:'Xoá Cột'}},button:{title:'Thuộc tính Nút',text:'Chuỗi hiển thị (Giá trị)',type:'Kiểu',typeBtn:'Nút Bấm',typeSbm:'Nút Gửi',typeRst:'Nút Nhập lại'},checkboxAndRadio:{checkboxTitle:'Thuộc tính Nút kiểm',radioTitle:'Thuộc tính Nút chọn',value:'Giá trị',selected:'Được chọn'},form:{title:'Thuộc tính Biểu mẫu',menu:'Thuộc tính Biểu mẫu',action:'Hành động',method:'Phương thức',encoding:'Bảng mã',target:'Đích',targetNotSet:'<không thiết lập>',targetNew:'Cửa sổ mới (_blank)',targetTop:'Cửa sổ trên cùng(_top)',targetSelf:'Cùng cửa sổ (_self)',targetParent:'Cửa sổ cha (_parent)'},select:{title:'Thuộc tính Ô chọn',selectInfo:'Thông tin',opAvail:'Các tùy chọn có thể sử dụng',value:'Giá trị',size:'Kích cỡ',lines:'dòng',chkMulti:'Cho phép chọn nhiều',opText:'Văn bản',opValue:'Giá trị',btnAdd:'Thêm',btnModify:'Thay đổi',btnUp:'Lên',btnDown:'Xuống',btnSetValue:'Giá trị được chọn',btnDelete:'Xoá'},textarea:{title:'Thuộc tính Vùng văn bản',cols:'Cột',rows:'Hàng'},textfield:{title:'Thuộc tính Trường văn bản',name:'Tên',value:'Giá trị',charWidth:'Rộng',maxChars:'Số Ký tự tối đa',type:'Kiểu',typeText:'Ký tự',typePass:'Mật khẩu'},hidden:{title:'Thuộc tính Trường ẩn',name:'Tên',value:'Giá trị'},image:{title:'Thuộc tính Hình ảnh',titleButton:'Thuộc tính Nút hình ảnh',menu:'Thuộc tính Hình ảnh',infoTab:'Thông tin Hình ảnh',btnUpload:'Tải lên Máy chủ',url:'URL',upload:'Tải lên',alt:'Chú thích Hình ảnh',width:'Rộng',height:'Cao',lockRatio:'Giữ nguyên tỷ lệ',resetSize:'Kích thước gốc',border:'Đường viền',hSpace:'HSpace',vSpace:'VSpace',align:'Vị trí',alignLeft:'Trái',alignAbsBottom:'Dưới tuyệt đối',alignAbsMiddle:'Giữa tuyệt đối',alignBaseline:'Đường cơ sở',alignBottom:'Dưới',alignMiddle:'Giữa',alignRight:'Phải',alignTextTop:'Phía trên chữ',alignTop:'Trên',preview:'Xem trước',alertUrl:'Hãy đưa vào URL của hình ảnh',linkTab:'Liên kết',button2Img:'Bạn có muốn chuyển nút bấm bằng hình ảnh được chọn thành hình ảnh?',img2Button:'Bạn có muốn chuyển đổi hình ảnh được chọn thành nút bấm bằng hình ảnh?',urlMissing:'Image source URL is missing.'},flash:{properties:'Thuộc tính Flash',propertiesTab:'Thuộc tính',title:'Thuộc tính Flash',chkPlay:'Tự động chạy',chkLoop:'Lặp',chkMenu:'Cho phép bật Menu của Flash',chkFull:'Cho phép Toàn màn hình',scale:'Tỷ lệ',scaleAll:'Hiển thị tất cả',scaleNoBorder:'Không đường viền',scaleFit:'Vừa vặn',access:'Truy cập Mã',accessAlways:'Luôn luôn',accessSameDomain:'Cùng tên miền',accessNever:'Không bao giờ',align:'Vị trí',alignLeft:'Trái',alignAbsBottom:'Dưới tuyệt đối',alignAbsMiddle:'Giữa tuyệt đối',alignBaseline:'Đường cơ sở',alignBottom:'Dưới',alignMiddle:'Giữa',alignRight:'Phải',alignTextTop:'Phía trên chữ',alignTop:'Trên',quality:'Chất lượng',qualityBest:'TỐt nhất',qualityHigh:'Cao',qualityAutoHigh:'Cao Tự động',qualityMedium:'Trung bình',qualityAutoLow:'Thấp Tự động',qualityLow:'Thấp',windowModeWindow:'Cửa sổ',windowModeOpaque:'Mờ đục',windowModeTransparent:'Trong suốt',windowMode:'Chế độ Cửa sổ',flashvars:'Các biến số dành cho Flash',bgcolor:'Màu nền',width:'Rộng',height:'Cao',hSpace:'HSpace',vSpace:'VSpace',validateSrc:'Hãy đưa vào Liên kết URL',validateWidth:'Chiều rộng phải là số nguyên.',validateHeight:'Chiều cao phải là số nguyên.',validateHSpace:'HSpace phải là số nguyên.',validateVSpace:'VSpace phải là số nguyên.'},spellCheck:{toolbar:'Kiểm tra Chính tả',title:'Kiểm tra Chính tả',notAvailable:'Xin lỗi, dịch vụ này hiện tại không có.',errorLoading:'Lỗi khi đang nạp dịch vụ ứng dụng: %s.',notInDic:'Không có trong từ điển',changeTo:'Chuyển thành',btnIgnore:'Bỏ qua',btnIgnoreAll:'Bỏ qua Tất cả',btnReplace:'Thay thế',btnReplaceAll:'Thay thế Tất cả',btnUndo:'Phục hồi lại',noSuggestions:'- Không đưa ra gợi ý về từ -',progress:'Đang tiến hành kiểm tra chính tả...',noMispell:'Hoàn tất kiểm tra chính tả: Không có lỗi chính tả',noChanges:'Hoàn tất kiểm tra chính tả: Không có từ nào được thay đổi',oneChange:'Hoàn tất kiểm tra chính tả: Một từ đã được thay đổi',manyChanges:'Hoàn tất kiểm tra chính tả: %1 từ đã được thay đổi',ieSpellDownload:'Chức năng kiểm tra chính tả chưa được cài đặt. Bạn có muốn tải về ngay bây giờ?'},smiley:{toolbar:'Hình biểu lộ cảm xúc (mặt cười)',title:'Chèn Hình biểu lộ cảm xúc (mặt cười)'},elementsPath:{eleTitle:'%1 thành phần'},numberedlist:'Danh sách có thứ tự',bulletedlist:'Danh sách không thứ tự',indent:'Dịch vào trong',outdent:'Dịch ra ngoài',justify:{left:'Canh trái',center:'Canh giữa',right:'Canh phải',block:'Canh đều'},blockquote:'Khối Trích dẫn',clipboard:{title:'Dán',cutError:'Các thiết lập bảo mật của trình duyệt không cho phép trình biên tập tự động thực thi lệnh cắt. Hãy sử dụng bàn phím cho lệnh này (Ctrl+X).',copyError:'Các thiết lập bảo mật của trình duyệt không cho phép trình biên tập tự động thực thi lệnh sao chép. Hãy sử dụng bàn phím cho lệnh này (Ctrl+C).',pasteMsg:'Hãy dán nội dung vào trong khung bên dưới, sử dụng tổ hợp phím (<STRONG>Ctrl+V</STRONG>) và nhấn vào nút <STRONG>Đồng ý</STRONG>.',securityMsg:'Do thiết lập bảo mật của trình duyệt nên trình biên tập không thể truy cập trực tiếp vào nội dung đã sao chép. Bạn cần phải dán lại nội dung vào cửa sổ này.'},pastefromword:{toolbar:'Dán với định dạng Word',title:'Dán với định dạng Word',advice:'Hãy dán nội dung vào trong khung bên dưới, sử dụng tổ hợp phím (<STRONG>Ctrl+V</STRONG>) và nhấn vào nút <STRONG>Đồng ý</STRONG>.',ignoreFontFace:'Chấp nhận các định dạng phông',removeStyle:'Gỡ bỏ các định dạng Styles'},pasteText:{button:'Dán theo định dạng văn bản thuần',title:'Dán theo định dạng văn bản thuần'},templates:{button:'Mẫu dựng sẵn',title:'Nội dung Mẫu dựng sẵn',insertOption:'Thay thế nội dung hiện tại',selectPromptMsg:'Hãy chọn Mẫu dựng sẵn để mở trong trình biên tập<br>(nội dung hiện tại sẽ bị mất):',emptyListMsg:'(Không có Mẫu dựng sẵn nào được định nghĩa)'},showBlocks:'Hiển thị các Khối',stylesCombo:{label:'Kiểu',voiceLabel:'Kiểu',panelVoiceLabel:'Chọn một kiểu',panelTitle1:'Kiểu Khối',panelTitle2:'Kiểu Trực tiếp',panelTitle3:'Kiểu Đối tượng'},format:{label:'Định dạng',voiceLabel:'Định dạng',panelTitle:'Định dạng',panelVoiceLabel:'Chọn định dạng đoạn văn bản',tag_p:'Normal',tag_pre:'Formatted',tag_address:'Address',tag_h1:'Heading 1',tag_h2:'Heading 2',tag_h3:'Heading 3',tag_h4:'Heading 4',tag_h5:'Heading 5',tag_h6:'Heading 6',tag_div:'Normal (DIV)'},font:{label:'Phông',voiceLabel:'Phông',panelTitle:'Phông',panelVoiceLabel:'Chọn phông'},fontSize:{label:'Cỡ chữ',voiceLabel:'Kích cỡ phông',panelTitle:'Cỡ chữ',panelVoiceLabel:'Chọn kích cỡ phông'},colorButton:{textColorTitle:'Màu chữ',bgColorTitle:'Màu nền',auto:'Tự động',more:'Màu khác...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Kiểm tra chính tả ngay khi gõ chữ (SCAYT)',enable:'Bật SCAYT',disable:'Tắt SCAYT',about:'Thông tin về SCAYT',toggle:'Bật tắt SCAYT',options:'Tùy chọn',langs:'Ngôn ngữ',moreSuggestions:'Đề xuất thêm',ignore:'Bỏ qua',ignoreAll:'Bỏ qua Tất cả',addWord:'Thêm Từ',emptyDic:'Tên của từ điển không được để trống.',optionsTab:'Tùy chọn',languagesTab:'Ngôn ngữ',dictionariesTab:'Từ điển',aboutTab:'Thông tin'},about:{title:'Thông tin về CKEditor',dlgTitle:'Thông tin về CKEditor',moreInfo:'Vui lòng ghé thăm trang web của chúng tôi để có thông tin về giấy phép:',copy:'Bản quyền &copy; $1. Giữ toàn quyền.'},maximize:'Phóng to tối đa',minimize:'Minimize',fakeobjects:{anchor:'Neo',flash:'Hoạt họa Flash',div:'Ngắt Trang',unknown:'Đối tượng không rõ ràng'},resize:'Kéo rê để thay đổi kích cỡ',colordialog:{title:'Select color',highlight:'Highlight',selected:'Selected',clear:'Clear'}};



```
