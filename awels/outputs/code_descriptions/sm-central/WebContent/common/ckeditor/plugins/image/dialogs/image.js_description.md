# image.js

## Review

## 1. Summary  
The script implements the **Image** and **Image Button** dialogs for CKEditor.  
It is a single‑file, immediately‑invoked function expression (IIFE) that registers two dialog definitions (`image` and `imagebutton`) with `CKEDITOR.dialog.add`.  

Key responsibilities  
* **Dialog UI definition** – an extensive object literal describes all tabs, fields, and widgets.  
* **Value handling** – several helper functions (`g`, `h`, `i`, `j`, `k`, `l`) manipulate field values, lock/unlock the aspect ratio, and keep the preview image in sync.  
* **Image preview** – loads the image via an `<img>` element, shows a loader while it loads, and falls back to a placeholder on error.  
* **Data binding** – `setup`, `commit`, `validate`, `onChange` callbacks keep the dialog and the underlying element in sync.

The module relies entirely on the CKEditor 3.x API; no external libraries are imported.

---

## 2. Detailed Description  

### Flow of Execution  

| Stage | What Happens | Key objects |
|-------|--------------|-------------|
| **IIFE runs** | Creates local variables (`a`–`e`), helper functions, and the dialog factory `l`. | `CKEDITOR` global |
| **`CKEDITOR.dialog.add('image', …)` / `'imagebutton'`** | Registers the dialog with the editor. Each call passes the editor instance (`m`) to `l`, which returns the full dialog definition object. | `editor` instance |
| **Dialog opening (`onShow`)** | *Creates a temporary `<img>`* to be used as the preview and to load the chosen URL. It also attaches load/error listeners. |
| **User input** | Fields have `onChange`, `onKeyUp`, etc., that call the helper functions to keep the preview, width/height ratio, and button lock state in sync. |
| **Commit on OK** | `commitContent` is invoked for each target (image, link, etc.). The image’s attributes (`src`, `alt`, `width`, `height`, …) and optional wrapper link are applied. |
| **Dialog hide** | Cleans up listeners and removes the temporary image element. |  |

### Core Components  

| Component | Role |
|-----------|------|
| `e`, `f` | Regular expressions to validate width/height values. |
| `g` | Handles width/height field changes, applies unit conversion, and triggers preview update. |
| `h` | Commits preview image attributes into the target element. |
| `i` | Manages the “lock ratio” state; toggles UI and calculates proportional dimensions. |
| `j` | Populates width/height fields from an existing element and triggers preview. |
| `k` | Parses style values when editing an element that already has a width/height in style or attribute. |
| `l` | Factory that builds the dialog definition object. It sets up tabs, elements, callbacks, and loads the image preview. |

### Design Choices  

* **Closure** – All helper functions are private to the IIFE, preventing namespace pollution.  
* **Object literals** – The dialog configuration follows CKEditor’s declarative style, making the UI layout clear albeit long.  
* **Dynamic behavior** – `onShow`, `onHide`, and field callbacks dynamically add/remove listeners and manipulate the DOM.  
* **Aspect‑ratio handling** – The lock button toggles `lockRatio` and calculates dimensions relative to the original image size.

---

## 3. Functions/Methods  

| Function | Purpose | Inputs | Outputs / Side‑Effects |
|----------|---------|--------|------------------------|
| `g()` | Triggered on width/height keyup. Parses the field value, checks for `%`, updates lock state, and calls `h()` to refresh the preview. | Uses `this` (field), accesses `this.getDialog()`. | Sets `dialog.lockRatio`, modifies preview dimensions. |
| `h(m)` | Commits the preview element’s dimensions to the target. | `m` – dialog instance. | Calls `commitContent(c, m.preview)`. |
| `i(m, n)` | Toggles / checks the “lock ratio” button. | `m` – dialog; `n` – optional flag (`'check'`, `true`, `false`). | Updates `m.lockRatio`, adds/removes CSS class on the button. |
| `j(m)` | Initializes width/height fields from an existing element and updates preview. | `m` – dialog. | Sets field values, calls `h(m)`. |
| `k(m, n)` | Parses style or attribute values when editing an element that already has width/height. | `m` – field, `n` – element. | Returns parsed value for the field. |
| `l(m, n)` | Dialog factory. Builds and returns the dialog definition object. | `m` – editor, `n` – dialog type (`'image'` or `'imagebutton'`). | Creates `dialog` object with tabs, elements, callbacks, and attaches listeners. |
| `CKEDITOR.dialog.add('image', …)` / `imagebutton` | Registers the dialog definitions with the editor. | `m` – editor instance. | None directly; side‑effect: dialog becomes available. |

*Reusable utilities:*  
* Regular expression helpers (`e`, `f`).  
* `commitContent` (CKEditor API) is used within `h` and `j`.

---

## 4. Dependencies  

| Library / API | Version / Notes | Is it part of CKEditor? |
|---------------|-----------------|------------------------|
| **CKEditor 3.x** | Uses `CKEDITOR.dialog`, `CKEDITOR.tools`, `CKEDITOR.document`, `CKEDITOR.getUrl` | Core library |
| **Browser DOM** | Standard `document` API for element manipulation (`createElement`, `setAttribute`, etc.) | Native |
| **No third‑party libs** | The code is self‑contained | — |

Assumptions  
* The editor instance has the `image` plugin enabled (for `CKEDITOR.dialog.add`).  
* The skin path (`m.skinPath`) contains `images/noimage.png`.  
* The editor has filebrowser integration enabled if the browse button is shown.

---

## 5. Additional Notes  

### Strengths  
* **Encapsulation** – The IIFE keeps internal helpers private.  
* **Declarative UI** – The dialog configuration is readable and follows CKEditor conventions.  
* **Aspect‑ratio logic** – Provides a good user experience when resizing images.  

### Weaknesses & Edge Cases  
1. **Variable naming** – Short, cryptic names (`a`, `b`, `c`, …) reduce readability and maintainability.  
2. **Hard‑coded strings** – Many string literals (e.g., `"absBottom"`, `"cke_btn_locked"`) are duplicated, increasing the risk of typos.  
3. **Synchronous vs. asynchronous** – `setTimeout` is used to delay setting the URL value (`j()`), which may lead to race conditions in newer browsers.  
4. **Percent handling** – The code assumes `width` and `height` can be expressed as `%`, but the preview logic ignores the fact that the element’s intrinsic size may change when the container resizes.  
5. **Error handling** – The preview fallback only shows a placeholder image; no user notification is provided if the URL is invalid.  
6. **Hard‑coded sizes** – The dialog dimensions (`minWidth:420`, `minHeight:310`) are fixed; responsive design would be preferable.  
7. **No cleanup of event listeners** – While some listeners are removed in `onHide`, the temporary `<img>` element could leak if the dialog is hidden unexpectedly.  

### Suggested Enhancements  
* **Rename variables** to self‑descriptive identifiers (`WIDTH`, `HEIGHT`, `LOCK_RATIO`, etc.).  
* **Extract strings** into constants or localization resources to avoid duplication.  
* **Modernize** the code (ES6 modules, arrow functions, `const/let`, `class` syntax) for readability.  
* **Add unit tests** for helper functions (especially `g`, `i`, `k`) to ensure edge‑case correctness.  
* **Improve validation** – provide inline error messages or tooltip hints for invalid URLs or dimensions.  
* **Responsive dialog** – allow the dialog to adapt to different screen sizes or use CSS flexbox.  
* **Accessibility** – ensure keyboard navigation and screen‑reader support for the preview area.  

Overall, the code achieves its goal of providing a fully functional image dialog within CKEditor, but it would benefit from refactoring for clarity, maintainability, and modern best practices.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){var a=1,b=2,c=4,d=8,e=/^\s*(\d+)((px)|\%)?\s*$/i,f=/(^\s*(\d+)((px)|\%)?\s*$)|^$/i,g=function(){var m=this.getValue(),n=this.getDialog(),o=m.match(e);if(o){if(o[2]=='%')i(n,false);m=o[1];}if(n.lockRatio){var p=n.originalElement;if(p.getCustomData('isReady')=='true')if(this.id=='txtHeight'){if(m&&m!='0')m=Math.round(p.$.width*(m/p.$.height));if(!isNaN(m))n.setValueOf('info','txtWidth',m);}else{if(m&&m!='0')m=Math.round(p.$.height*(m/p.$.width));if(!isNaN(m))n.setValueOf('info','txtHeight',m);}}h(n);},h=function(m){if(!m.originalElement||!m.preview)return 1;m.commitContent(c,m.preview);return 0;},i=function(m,n){var o=m.originalElement,p=CKEDITOR.document.getById('btnLockSizes');if(o.getCustomData('isReady')=='true'){if(n=='check'){var q=m.getValueOf('info','txtWidth'),r=m.getValueOf('info','txtHeight'),s=o.$.width*1000/o.$.height,t=q*1000/r;m.lockRatio=false;if(!q&&!r)m.lockRatio=true;else if(!isNaN(s)&&!isNaN(t))if(Math.round(s)==Math.round(t))m.lockRatio=true;}else if(n!=undefined)m.lockRatio=n;else m.lockRatio=!m.lockRatio;}else if(n!='check')m.lockRatio=false;if(m.lockRatio)p.removeClass('cke_btn_unlocked');else p.addClass('cke_btn_unlocked');return m.lockRatio;},j=function(m){var n=m.originalElement;if(n.getCustomData('isReady')=='true'){m.setValueOf('info','txtWidth',n.$.width);m.setValueOf('info','txtHeight',n.$.height);}h(m);},k=function(m,n){if(m!=a)return;function o(t,u){var v=t.match(e);if(v){if(v[2]=='%'){v[1]+='%';i(p,false);}return v[1];}return u;};var p=this.getDialog(),q='',r=this.id=='txtWidth'?'width':'height',s=n.getAttribute(r);if(s)q=o(s,q);q=o(n.$.style[r],q);this.setValue(q);},l=function(m,n){var o=function(){var r=this;var q=r.originalElement;q.setCustomData('isReady','true');q.removeListener('load',o);q.removeListener('error',p);q.removeListener('abort',p);CKEDITOR.document.getById('ImagePreviewLoader').setStyle('display','none');if(!r.dontResetSize)j(r);if(r.firstLoad)i(r,'check');r.firstLoad=false;r.dontResetSize=false;},p=function(){var s=this;var q=s.originalElement;q.removeListener('load',o);q.removeListener('error',p);q.removeListener('abort',p);var r=CKEDITOR.getUrl(m.skinPath+'images/noimage.png');if(s.preview)s.preview.setAttribute('src',r);CKEDITOR.document.getById('ImagePreviewLoader').setStyle('display','none');i(s,false);};return{title:n=='image'?m.lang.image.title:m.lang.image.titleButton,minWidth:420,minHeight:310,onShow:function(){var w=this;w.imageElement=false;w.linkElement=false;w.imageEditMode=false;w.linkEditMode=false;
w.lockRatio=true;w.dontResetSize=false;w.firstLoad=true;w.addLink=false;CKEDITOR.document.getById('ImagePreviewLoader').setStyle('display','none');w.preview=CKEDITOR.document.getById('previewImage');var q=w.getParentEditor(),r=w.getParentEditor().getSelection(),s=r.getSelectedElement(),t=s&&s.getAscendant('a');w.originalElement=q.document.createElement('img');w.originalElement.setAttribute('alt','');w.originalElement.setCustomData('isReady','false');if(t){w.linkElement=t;w.linkEditMode=true;var u=t.getChildren();if(u.count()==1){var v=u.getItem(0).getName();if(v=='img'||v=='input'){w.imageElement=u.getItem(0);if(w.imageElement.getName()=='img')w.imageEditMode='img';else if(w.imageElement.getName()=='input')w.imageEditMode='input';}}if(n=='image')w.setupContent(b,t);}if(s&&s.getName()=='img'&&!s.getAttribute('_cke_protected_html'))w.imageEditMode='img';else if(s&&s.getName()=='input'&&s.getAttribute('type')&&s.getAttribute('type')=='image')w.imageEditMode='input';if(w.imageEditMode||w.imageElement){if(!w.imageElement)w.imageElement=s;w.setupContent(a,w.imageElement);i(w,true);}if(!CKEDITOR.tools.trim(w.getValueOf('info','txtUrl'))){w.preview.removeAttribute('src');w.preview.setStyle('display','none');}},onOk:function(){var r=this;if(r.imageEditMode){var q=r.imageEditMode;if(n=='image'&&q=='input'&&confirm(m.lang.image.button2Img)){q='img';r.imageElement=m.document.createElement('img');r.imageElement.setAttribute('alt','');m.insertElement(r.imageElement);}else if(n!='image'&&q=='img'&&confirm(m.lang.image.img2Button)){q='input';r.imageElement=m.document.createElement('input');r.imageElement.setAttributes({type:'image',alt:''});m.insertElement(r.imageElement);}}else{if(n=='image')r.imageElement=m.document.createElement('img');else{r.imageElement=m.document.createElement('input');r.imageElement.setAttribute('type','image');}r.imageElement.setAttribute('alt','');}if(!r.linkEditMode)r.linkElement=m.document.createElement('a');r.commitContent(a,r.imageElement);r.commitContent(b,r.linkElement);if(!r.imageEditMode){if(r.addLink){if(!r.linkEditMode){m.insertElement(r.linkElement);r.linkElement.append(r.imageElement,false);}else m.insertElement(r.imageElement);}else m.insertElement(r.imageElement);}else if(!r.linkEditMode&&r.addLink){m.insertElement(r.linkElement);r.imageElement.appendTo(r.linkElement);}else if(r.linkEditMode&&!r.addLink){m.getSelection().selectElement(r.linkElement);m.insertElement(r.imageElement);}},onLoad:function(){var r=this;if(n!='image')r.hidePage('Link');
var q=r._.element.getDocument();r.addFocusable(q.getById('btnResetSize'),5);r.addFocusable(q.getById('btnLockSizes'),5);},onHide:function(){var q=this;if(q.preview)q.commitContent(d,q.preview);if(q.originalElement){q.originalElement.removeListener('load',o);q.originalElement.removeListener('error',p);q.originalElement.removeListener('abort',p);q.originalElement.remove();q.originalElement=false;}},contents:[{id:'info',label:m.lang.image.infoTab,accessKey:'I',elements:[{type:'vbox',padding:0,children:[{type:'html',html:'<span>'+CKEDITOR.tools.htmlEncode(m.lang.image.url)+'</span>'},{type:'hbox',widths:['280px','110px'],align:'right',children:[{id:'txtUrl',type:'text',label:'',onChange:function(){var q=this.getDialog(),r=this.getValue();if(r.length>0){q=this.getDialog();var s=q.originalElement;q.preview.removeStyle('display');s.setCustomData('isReady','false');var t=CKEDITOR.document.getById('ImagePreviewLoader');if(t)t.setStyle('display','');s.on('load',o,q);s.on('error',p,q);s.on('abort',p,q);s.setAttribute('src',r);q.preview.setAttribute('src',r);h(q);}else if(q.preview){q.preview.removeAttribute('src');q.preview.setStyle('display','none');}},setup:function(q,r){if(q==a){var s=r.getAttribute('_cke_saved_src')||r.getAttribute('src'),t=this;this.getDialog().dontResetSize=true;setTimeout(function(){t.setValue(s);t.setInitValue();t.focus();},0);}},commit:function(q,r){var s=this;if(q==a&&(s.getValue()||s.isChanged())){r.setAttribute('_cke_saved_src',decodeURI(s.getValue()));r.setAttribute('src',decodeURI(s.getValue()));}else if(q==d){r.setAttribute('src','');r.removeAttribute('src');}},validate:CKEDITOR.dialog.validate.notEmpty(m.lang.image.urlMissing)},{type:'button',id:'browse',align:'center',label:m.lang.common.browseServer,hidden:true,filebrowser:'info:txtUrl'}]}]},{id:'txtAlt',type:'text',label:m.lang.image.alt,accessKey:'A','default':'',onChange:function(){h(this.getDialog());},setup:function(q,r){if(q==a)this.setValue(r.getAttribute('alt'));},commit:function(q,r){var s=this;if(q==a){if(s.getValue()||s.isChanged())r.setAttribute('alt',s.getValue());}else if(q==c)r.setAttribute('alt',s.getValue());else if(q==d)r.removeAttribute('alt');}},{type:'hbox',widths:['140px','240px'],children:[{type:'vbox',padding:10,children:[{type:'hbox',widths:['70%','30%'],children:[{type:'vbox',padding:1,children:[{type:'text',width:'40px',id:'txtWidth',labelLayout:'horizontal',label:m.lang.image.width,onKeyUp:g,validate:function(){var q=this.getValue().match(f);if(!q)alert(m.lang.common.validateNumberFailed);
return!!q;},setup:k,commit:function(q,r){var v=this;if(q==a){var s=v.getValue();if(s)r.setAttribute('width',s);else if(!s&&v.isChanged())r.removeAttribute('width');}else if(q==c){s=v.getValue();var t=s.match(e);if(!t){var u=v.getDialog().originalElement;if(u.getCustomData('isReady')=='true')r.setStyle('width',u.$.width+'px');}else r.setStyle('width',s+'px');}else if(q==d){r.setStyle('width','0px');r.removeAttribute('width');r.removeStyle('width');}}},{type:'text',id:'txtHeight',width:'40px',labelLayout:'horizontal',label:m.lang.image.height,onKeyUp:g,validate:function(){var q=this.getValue().match(f);if(!q)alert(m.lang.common.validateNumberFailed);return!!q;},setup:k,commit:function(q,r){var v=this;if(q==a){var s=v.getValue();if(s)r.setAttribute('height',s);else if(!s&&v.isChanged())r.removeAttribute('height');}else if(q==c){s=v.getValue();var t=s.match(e);if(!t){var u=v.getDialog().originalElement;if(u.getCustomData('isReady')=='true')r.setStyle('height',u.$.height+'px');}else r.setStyle('height',s+'px');}else if(q==d){r.setStyle('height','0px');r.removeAttribute('height');r.removeStyle('height');}}}]},{type:'html',style:'margin-top:10px;width:40px;height:40px;',onLoad:function(){var q=CKEDITOR.document.getById('btnResetSize'),r=CKEDITOR.document.getById('btnLockSizes');if(q){q.on('click',function(){j(this);},this.getDialog());q.on('mouseover',function(){this.addClass('cke_btn_over');},q);q.on('mouseout',function(){this.removeClass('cke_btn_over');},q);}if(r){r.on('click',function(){var w=this;var s=i(w),t=w.originalElement,u=w.getValueOf('info','txtWidth');if(t.getCustomData('isReady')=='true'&&u){var v=t.$.height/t.$.width*u;if(!isNaN(v)){w.setValueOf('info','txtHeight',Math.round(v));h(w);}}},this.getDialog());r.on('mouseover',function(){this.addClass('cke_btn_over');},r);r.on('mouseout',function(){this.removeClass('cke_btn_over');},r);}},html:'<div><a href="javascript:void(0)" tabindex="-1" title="'+m.lang.image.lockRatio+'" class="cke_btn_locked" id="btnLockSizes"></a>'+'<a href="javascript:void(0)" tabindex="-1" title="'+m.lang.image.resetSize+'" class="cke_btn_reset" id="btnResetSize"></a>'+'</div>'}]},{type:'vbox',padding:1,children:[{type:'text',id:'txtBorder',width:'60px',labelLayout:'horizontal',label:m.lang.image.border,'default':'',onKeyUp:function(){h(this.getDialog());},validate:function(){var q=CKEDITOR.dialog.validate.integer(m.lang.common.validateNumberFailed);return q.apply(this);},setup:function(q,r){if(q==a)this.setValue(r.getAttribute('border'));
},commit:function(q,r){var t=this;if(q==a){if(t.getValue()||t.isChanged())r.setAttribute('border',t.getValue());}else if(q==c){var s=parseInt(t.getValue(),10);s=isNaN(s)?0:s;r.setAttribute('border',s);r.setStyle('border',s+'px solid black');}else if(q==d){r.removeAttribute('border');r.removeStyle('border');}}},{type:'text',id:'txtHSpace',width:'60px',labelLayout:'horizontal',label:m.lang.image.hSpace,'default':'',onKeyUp:function(){h(this.getDialog());},validate:function(){var q=CKEDITOR.dialog.validate.integer(m.lang.common.validateNumberFailed);return q.apply(this);},setup:function(q,r){if(q==a){var s=r.getAttribute('hspace');if(s!=-1)this.setValue(s);}},commit:function(q,r){var t=this;if(q==a){if(t.getValue()||t.isChanged())r.setAttribute('hspace',t.getValue());}else if(q==c){var s=parseInt(t.getValue(),10);s=isNaN(s)?0:s;r.setAttribute('hspace',s);r.setStyle('margin-left',s+'px');r.setStyle('margin-right',s+'px');}else if(q==d){r.removeAttribute('hspace');r.removeStyle('margin-left');r.removeStyle('margin-right');}}},{type:'text',id:'txtVSpace',width:'60px',labelLayout:'horizontal',label:m.lang.image.vSpace,'default':'',onKeyUp:function(){h(this.getDialog());},validate:function(){var q=CKEDITOR.dialog.validate.integer(m.lang.common.validateNumberFailed);return q.apply(this);},setup:function(q,r){if(q==a)this.setValue(r.getAttribute('vspace'));},commit:function(q,r){var t=this;if(q==a){if(t.getValue()||t.isChanged())r.setAttribute('vspace',t.getValue());}else if(q==c){var s=parseInt(t.getValue(),10);s=isNaN(s)?0:s;r.setAttribute('vspace',t.getValue());r.setStyle('margin-top',s+'px');r.setStyle('margin-bottom',s+'px');}else if(q==d){r.removeAttribute('vspace');r.removeStyle('margin-top');r.removeStyle('margin-bottom');}}},{id:'cmbAlign',type:'select',labelLayout:'horizontal',widths:['35%','65%'],style:'width:90px',label:m.lang.image.align,'default':'',items:[[m.lang.common.notSet,''],[m.lang.image.alignLeft,'left'],[m.lang.image.alignAbsBottom,'absBottom'],[m.lang.image.alignAbsMiddle,'absMiddle'],[m.lang.image.alignBaseline,'baseline'],[m.lang.image.alignBottom,'bottom'],[m.lang.image.alignMiddle,'middle'],[m.lang.image.alignRight,'right'],[m.lang.image.alignTextTop,'textTop'],[m.lang.image.alignTop,'top']],onChange:function(){h(this.getDialog());},setup:function(q,r){if(q==a)this.setValue(r.getAttribute('align'));},commit:function(q,r){var s=this.getValue();if(q==a){if(s||this.isChanged())r.setAttribute('align',s);}else if(q==c){r.setAttribute('align',this.getValue());
if(s=='absMiddle'||s=='middle')r.setStyle('vertical-align','middle');else if(s=='top'||s=='textTop')r.setStyle('vertical-align','top');else r.removeStyle('vertical-align');if(s=='right'||s=='left')r.setStyle('styleFloat',s);else r.removeStyle('styleFloat');}else if(q==d)r.removeAttribute('align');}}]}]},{type:'vbox',height:'250px',children:[{type:'html',style:'width:95%;',html:'<div>'+CKEDITOR.tools.htmlEncode(m.lang.image.preview)+'<br>'+'<div id="ImagePreviewLoader" style="display:none"><div class="loading">&nbsp;</div></div>'+'<div id="ImagePreviewBox">'+'<a href="javascript:void(0)" target="_blank" onclick="return false;" id="previewLink">'+'<img id="previewImage" src="" alt="" /></a>Lorem ipsum dolor sit amet, consectetuer adipiscing elit. '+'Maecenas feugiat consequat diam. Maecenas metus. Vivamus diam purus, cursus a, commodo non, facilisis vitae, '+'nulla. Aenean dictum lacinia tortor. Nunc iaculis, nibh non iaculis aliquam, orci felis euismod neque, sed ornare massa mauris sed velit. Nulla pretium mi et risus. Fusce mi pede, tempor id, cursus ac, ullamcorper nec, enim. Sed tortor. Curabitur molestie. Duis velit augue, condimentum at, ultrices a, luctus ut, orci. Donec pellentesque egestas eros. Integer cursus, augue in cursus faucibus, eros pede bibendum sem, in tempus tellus justo quis ligula. Etiam eget tortor. Vestibulum rutrum, est ut placerat elementum, lectus nisl aliquam velit, tempor aliquam eros nunc nonummy metus. In eros metus, gravida a, gravida sed, lobortis id, turpis. Ut ultrices, ipsum at venenatis fringilla, sem nulla lacinia tellus, eget aliquet turpis mauris non enim. Nam turpis. Suspendisse lacinia. Curabitur ac tortor ut ipsum egestas elementum. Nunc imperdiet gravida mauris.'+'</div>'+'</div>'}]}]}]},{id:'Link',label:m.lang.link.title,padding:0,elements:[{id:'txtUrl',type:'text',label:m.lang.image.url,style:'width: 100%','default':'',setup:function(q,r){if(q==b){var s=r.getAttribute('_cke_saved_href');if(!s)s=r.getAttribute('href');this.setValue(s);}},commit:function(q,r){var s=this;if(q==b)if(s.getValue()||s.isChanged()){r.setAttribute('_cke_saved_href',decodeURI(s.getValue()));r.setAttribute('href','javascript:void(0)/*'+CKEDITOR.tools.getNextNumber()+'*/');if(s.getValue()||!m.config.image_removeLinkByEmptyURL)s.getDialog().addLink=true;}}},{type:'button',id:'browse',filebrowser:'Link:txtUrl',style:'float:right',hidden:true,label:m.lang.common.browseServer},{id:'cmbTarget',type:'select',label:m.lang.link.target,'default':'',items:[[m.lang.link.targetNotSet,''],[m.lang.link.targetNew,'_blank'],[m.lang.link.targetTop,'_top'],[m.lang.link.targetSelf,'_self'],[m.lang.link.targetParent,'_parent']],setup:function(q,r){if(q==b)this.setValue(r.getAttribute('target'));
},commit:function(q,r){if(q==b)if(this.getValue()||this.isChanged())r.setAttribute('target',this.getValue());}}]},{id:'Upload',hidden:true,filebrowser:'uploadButton',label:m.lang.image.upload,elements:[{type:'file',id:'upload',label:m.lang.image.btnUpload,style:'height:40px',size:38},{type:'fileButton',id:'uploadButton',filebrowser:'info:txtUrl',label:m.lang.image.btnUpload,'for':['Upload','upload']}]},{id:'advanced',label:m.lang.common.advancedTab,elements:[{type:'hbox',widths:['50%','25%','25%'],children:[{type:'text',id:'linkId',label:m.lang.common.id,setup:function(q,r){if(q==a)this.setValue(r.getAttribute('id'));},commit:function(q,r){if(q==a)if(this.getValue()||this.isChanged())r.setAttribute('id',this.getValue());}},{id:'cmbLangDir',type:'select',style:'width : 100px;',label:m.lang.common.langDir,'default':'',items:[[m.lang.common.notSet,''],[m.lang.common.langDirLtr,'ltr'],[m.lang.common.langDirRtl,'rtl']],setup:function(q,r){if(q==a)this.setValue(r.getAttribute('dir'));},commit:function(q,r){if(q==a)if(this.getValue()||this.isChanged())r.setAttribute('dir',this.getValue());}},{type:'text',id:'txtLangCode',label:m.lang.common.langCode,'default':'',setup:function(q,r){if(q==a)this.setValue(r.getAttribute('lang'));},commit:function(q,r){if(q==a)if(this.getValue()||this.isChanged())r.setAttribute('lang',this.getValue());}}]},{type:'text',id:'txtGenLongDescr',label:m.lang.common.longDescr,setup:function(q,r){if(q==a)this.setValue(r.getAttribute('longDesc'));},commit:function(q,r){if(q==a)if(this.getValue()||this.isChanged())r.setAttribute('longDesc',this.getValue());}},{type:'hbox',widths:['50%','50%'],children:[{type:'text',id:'txtGenClass',label:m.lang.common.cssClass,'default':'',setup:function(q,r){if(q==a)this.setValue(r.getAttribute('class'));},commit:function(q,r){if(q==a)if(this.getValue()||this.isChanged())r.setAttribute('class',this.getValue());}},{type:'text',id:'txtGenTitle',label:m.lang.common.advisoryTitle,'default':'',onChange:function(){h(this.getDialog());},setup:function(q,r){if(q==a)this.setValue(r.getAttribute('title'));},commit:function(q,r){var s=this;if(q==a){if(s.getValue()||s.isChanged())r.setAttribute('title',s.getValue());}else if(q==c)r.setAttribute('title',s.getValue());else if(q==d)r.removeAttribute('title');}}]},{type:'text',id:'txtdlgGenStyle',label:m.lang.common.cssStyle,'default':'',setup:function(q,r){if(q==a){var s=r.getAttribute('style');if(!s&&r.$.style.cssText)s=r.$.style.cssText;this.setValue(s);var t=r.$.style.height,u=r.$.style.width,v=(t?t:'').match(e),w=(u?u:'').match(e);
this.attributesInStyle={height:!!v,width:!!w};}},commit:function(q,r){var u=this;if(q==a&&(u.getValue()||u.isChanged())){r.setAttribute('style',u.getValue());var s=r.getAttribute('height'),t=r.getAttribute('width');if(u.attributesInStyle&&u.attributesInStyle.height)if(s){if(s.match(e)[2]=='%')r.setStyle('height',s+'%');else r.setStyle('height',s+'px');}else r.removeStyle('height');if(u.attributesInStyle&&u.attributesInStyle.width)if(t){if(t.match(e)[2]=='%')r.setStyle('width',t+'%');else r.setStyle('width',t+'px');}else r.removeStyle('width');}}}]}]};};CKEDITOR.dialog.add('image',function(m){return l(m,'image');});CKEDITOR.dialog.add('imagebutton',function(m){return l(m,'imagebutton');});})();



```
