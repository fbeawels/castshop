# link.js

## Review

## 1. Summary

The file is the **Link dialog** definition for CKEditor, a rich‑text editor.  
It registers a dialog named `'link'` through `CKEDITOR.dialog.add`.  
The dialog allows users to create or edit a hyperlink (`<a>` or `<img>` anchor) and provides several tabs:

1. **Info** – choose between URL, Anchor, or Email, and input the target value.
2. **Target** – configure the target window/frame/popup options.
3. **Upload** – a file‑upload interface (only if the editor is configured for it).
4. **Advanced** – set advanced attributes such as `id`, `lang`, `accessKey`, etc.

The implementation contains a mix of helper functions (`b`, `c`, `d`, …), validation logic, and code that prepares the dialog’s initial state and commits the user’s changes back to the editor’s DOM.

**Key components**

| Component | Role |
|-----------|------|
| `b` | Handles visibility of the popup‑feature pane and updates the target name label. |
| `c` | Enables or disables the **Target** and **Upload** tabs based on the chosen link type. |
| `d … j` | Regular‑expression constants used to parse URLs, mailto links, anchors, and `window.open()` onclick handlers. |
| `k` | Regular‑expression used to read popup‑feature key/value pairs. |
| `l` | The *setup* function – reads the current selection or element and builds an object that describes the link (type, URL, target, advanced attributes, anchors). |
| `m,n,o,p,q,r` | Small helper functions that map dialog values to the `s` object that is later committed. |
| Dialog definition | Contains the full dialog configuration: titles, sizes, tab contents, element definitions, and the lifecycle callbacks (`onShow`, `onOk`, `onLoad`). |

The code is written for CKEditor 4.x (the file starts with the standard `CKEDITOR.dialog.add` call) and uses many internal CKEditor utilities (`CKEDITOR.tools`, `CKEDITOR.dom`, `CKEDITOR.env`, etc.).

---

## 2. Detailed Description

### Execution flow

| Phase | What happens |
|-------|--------------|
| **Registration** | `CKEDITOR.dialog.add('link',function(a){…})` – the callback receives the CKEditor instance (`a`). |
| **Dialog initialization** | The returned object contains the dialog definition: `title`, `minWidth`, `minHeight`, `contents`, `onShow`, `onOk`, `onLoad`. |
| **Showing the dialog** | `onShow` runs when the dialog is opened. It detects whether the user has selected an existing link (`<a>`) or anchor image. The helper `l` then populates the dialog with the current element’s data. |
| **User interaction** | The user can modify any tab. The dialog keeps internal state (`this._`) and has per‑field callbacks (`setup`, `commit`, `validate`, etc.). |
| **Closing** | When the user clicks **OK**, `onOk` is invoked. It builds a temporary object `u` with the final link attributes (URL, target, advanced attributes) and applies the changes to the document: either creating a new link or updating an existing one. |
| **Cleanup** | `onLoad` hides unused tabs based on configuration flags (`linkShowAdvancedTab`, `linkShowTargetTab`). |

### Core assumptions & constraints

- **Editor configuration** – the dialog respects a few config flags (e.g. `linkShowAdvancedTab`, `linkShowTargetTab`, `linkShowTargetTab`). If they are set to `false`, the related tab is hidden.
- **Cross‑browser support** – the code contains IE/ WebKit specific handling (e.g. `CKEDITOR.env.ie`), and uses a custom `onclick` string for popup windows that works in older browsers.
- **Anchors** – The dialog supports both `<a name="...">` anchors (real elements) and anchor images that are placeholders (the “fake element” concept).
- **Email links** – The dialog parses and builds `mailto:` URLs with optional `subject` and `body` query parameters.
- **Upload** – The `upload` tab is shown only if a file browser is configured for `info:url` or `uploadButton`.  
- **Performance** – All DOM manipulations happen through CKEditor’s abstraction (`CKEDITOR.dom`) to remain framework‑agnostic.

### Architectural notes

- The dialog follows CKEditor’s **Dialog API**: it is a declarative configuration that contains nested `contents` arrays describing the UI hierarchy.
- Validation is performed by small inline functions that use `CKEDITOR.dialog.validate`.
- The `setup`/`commit` pattern keeps the dialog logic and the data model separated, making the code easier to maintain despite the minified style.

---

## 3. Functions / Methods

| Function | Purpose | Inputs | Outputs | Side‑effects |
|----------|---------|--------|---------|--------------|
| `b()` | Handles target tab changes – shows/hides popup features pane and updates the target name label. | `this` – dialog instance. | None (updates UI). | Calls `setLabel`, `show`/`hide`. |
| `c()` | Enables/disables Target and Upload tabs based on chosen link type. | `this`. | None. | Calls `showPage`, `hidePage`. |
| `k` | Regular expression `/(?:^|,)([^=]+)=(\d+|yes|no)/gi` – parses popup feature key/value pairs. | N/A | RegExp object. | N/A |
| `l(s, t)` | **Setup** function – extracts link data from the selected element (`t`) and returns a plain object describing the link. | `s` – editor instance, `t` – element (possibly `null`). | Object `{type, url, anchor, email, target, adv, anchors}` | Reads attributes, parses `href`, collects anchors. |
| `m(s, t)` | Helper that populates a field from the `t` object for a given key. | `this`, `t` (the object). | None | Calls `setValue`. |
| `n(s)` | Calls `m` for the `target` key. | `this`. | None | |
| `o(s)` | Calls `m` for the `adv` key. | `this`. | None | |
| `p(s, t)` | Helper that stores the value of a field into the object `t`. | `this`, `t`. | None | Calls `getValue`. |
| `q(s)` | Calls `p` for the `target` key. | `this`. | None | |
| `r(s)` | Calls `p` for the `adv` key. | `this`. | None | |

These helpers are used by the `setup`/`commit` functions of individual dialog fields (e.g. `linkTargetType`, `linkTargetName`, etc.).

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | **Core library** | Provides the Dialog API, utilities, DOM helpers, and environment detection. |
| `CKEDITOR.tools` | **Utility module** | Used for string manipulation, HTML encoding, and `getNextNumber`. |
| `CKEDITOR.dom` | **DOM abstraction** | Creates elements, queries, and manipulates attributes. |
| `CKEDITOR.env` | **Browser detection** | Determines IE/WebKit-specific logic. |
| `CKEDITOR.STYLE` | **Style API** | Applies inline `<a>` elements. |
| `CKEDITOR.dialog.validate` | **Validation** | Provides `notEmpty` validation. |
| `CKEDITOR.tools.getNextNumber` | **Unique ID generator** | Used for `javascript:void(0)/*...*/` placeholders. |

All of these are bundled with CKEditor and are **standard** third‑party libraries for CKEditor itself. No external APIs or platform‑specific dependencies are required.

---

## 5. Additional Notes

### Edge cases & limitations

| Area | Potential issue | Suggested fix |
|------|-----------------|---------------|
| **URL parsing** | The regexes for URLs (`h`) and mailto (`d`) are simplistic and may mis‑parse complex URLs with credentials or fragments. | Consider using the browser’s `URL` constructor or a more robust regex. |
| **Popup feature parsing** | The `k` regex expects commas as separators and “yes/no” values; any other value will be silently ignored. | Add support for numeric values and default handling. |
| **International characters** | Email addresses are decoded with `decodeURIComponent` but the subject/body may contain characters that require RFC‑6532 handling. | Use a dedicated MIME encoder or rely on the browser’s encoding. |
| **Fake element handling** | The logic for fake anchor images only supports a single “anchor” type; other elements (e.g., `<img>` used as anchors) may not be fully handled. | Extend fake‑element support to cover all anchor types. |
| **File upload** | The upload tab relies on a configured file browser. If the editor is used without one, the tab may break. | Guard against missing file browser plugins or provide a fallback UI. |
| **Internationalization** | The labels are taken from `a.lang` but the dialog may be used in languages that require direction changes (RTL). | Ensure all text and label orderings respect `dir="rtl"`. |

### Future enhancements

1. **Modernize the regex logic** – Use the native `URL` API to parse and build URLs instead of manual regexes.
2. **Simplify the dialog definition** – Split the large configuration into separate modules (info, target, upload, advanced) for readability.
3. **Accessibility** – Add ARIA attributes to form elements, improve keyboard navigation, and expose focus‑related events.
4. **Unit tests** – Write automated tests for `l` (the setup function) and the commit logic using CKEditor’s test harness.
5. **Performance** – Cache the list of anchors once per editor instance instead of recomputing on every dialog open.

---

### Closing

The Link dialog is a comprehensive, well‑structured component that adheres to CKEditor’s design patterns. Although heavily minified, the code demonstrates careful handling of different link types, target behaviors, and advanced attributes. With minor refactoring and modernisation, it could be made even more maintainable and robust for future CKEditor releases.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('link',function(a){var b=function(){var s=this.getDialog(),t=s.getContentElement('target','popupFeatures'),u=s.getContentElement('target','linkTargetName'),v=this.getValue();if(!t||!u)return;t=t.getElement();if(v=='popup'){t.show();u.setLabel(a.lang.link.targetPopupName);}else{t.hide();u.setLabel(a.lang.link.targetFrameName);this.getDialog().setValueOf('target','linkTargetName',v.charAt(0)=='_'?v:'');}},c=function(){var s=this.getDialog(),t=['urlOptions','anchorOptions','emailOptions'],u=this.getValue(),v=s.definition.getContents('upload').hidden;if(u=='url'){if(a.config.linkShowTargetTab)s.showPage('target');if(!v)s.showPage('upload');}else{s.hidePage('target');if(!v)s.hidePage('upload');}for(var w=0;w<t.length;w++){var x=s.getContentElement('info',t[w]);if(!x)continue;x=x.getElement().getParent().getParent();if(t[w]==u+'Options')x.show();else x.hide();}},d=/^mailto:([^?]+)(?:\?(.+))?$/,e=/subject=([^;?:@&=$,\/]*)/,f=/body=([^;?:@&=$,\/]*)/,g=/^#(.*)$/,h=/^((?:http|https|ftp|news):\/\/)?(.*)$/,i=/^(_(?:self|top|parent|blank))$/,j=/\s*window.open\(\s*this\.href\s*,\s*(?:'([^']*)'|null)\s*,\s*'([^']*)'\s*\)\s*;\s*return\s*false;*\s*/,k=/(?:^|,)([^=]+)=(\d+|yes|no)/gi,l=function(s,t){var u=t?t.getAttribute('_cke_saved_href')||t.getAttribute('href'):'',v='',w='',x=false,y={};if(u){v=u.match(d);w=u.match(g);x=u.match(h);}if(v){var z=u.match(e),A=u.match(f);y.type='email';y.email={};y.email.address=v[1];z&&(y.email.subject=decodeURIComponent(z[1]));A&&(y.email.body=decodeURIComponent(A[1]));}else if(w){y.type='anchor';y.anchor={};y.anchor.name=y.anchor.id=w[1];}else if(u&&x){y.type='url';y.url={};y.url.protocol=x[1];y.url.url=x[2];}else y.type='url';if(t){var B=t.getAttribute('target');y.target={};y.adv={};if(!B){var C=t.getAttribute('_cke_pa_onclick')||t.getAttribute('onclick'),D=C&&C.match(j);if(D){y.target.type='popup';y.target.name=D[1];var E;while(E=k.exec(D[2]))if(E[2]=='yes'||E[2]=='1')y.target[E[1]]=true;else if(isFinite(E[2]))y.target[E[1]]=E[2];}}else{var F=B.match(i);if(F)y.target.type=y.target.name=B;else{y.target.type='frame';y.target.name=B;}}var G=this,H=function(N,O){var P=t.getAttribute(O);if(P!==null)y.adv[N]=P||'';};H('advId','id');H('advLangDir','dir');H('advAccessKey','accessKey');H('advName','name');H('advLangCode','lang');H('advTabIndex','tabindex');H('advTitle','title');H('advContentType','type');H('advCSSClasses','class');H('advCharset','charset');H('advStyles','style');}var I=s.document.getElementsByTag('img'),J=new CKEDITOR.dom.nodeList(s.document.$.anchors),K=y.anchors=[];
for(var L=0;L<I.count();L++){var M=I.getItem(L);if(M.getAttribute('_cke_realelement')&&M.getAttribute('_cke_real_element_type')=='anchor')K.push(s.restoreRealElement(M));}for(L=0;L<J.count();L++)K.push(J.getItem(L));for(L=0;L<K.length;L++){M=K[L];K[L]={name:M.getAttribute('name'),id:M.getAttribute('id')};}this._.selectedElement=t;return y;},m=function(s,t){if(t[s])this.setValue(t[s][this.id]||'');},n=function(s){return m.call(this,'target',s);},o=function(s){return m.call(this,'adv',s);},p=function(s,t){if(!t[s])t[s]={};t[s][this.id]=this.getValue()||'';},q=function(s){return p.call(this,'target',s);},r=function(s){return p.call(this,'adv',s);};return{title:a.lang.link.title,minWidth:350,minHeight:230,contents:[{id:'info',label:a.lang.link.info,title:a.lang.link.info,elements:[{id:'linkType',type:'select',label:a.lang.link.type,'default':'url',items:[[a.lang.common.url,'url'],[a.lang.link.toAnchor,'anchor'],[a.lang.link.toEmail,'email']],onChange:c,setup:function(s){if(s.type)this.setValue(s.type);},commit:function(s){s.type=this.getValue();}},{type:'vbox',id:'urlOptions',children:[{type:'hbox',widths:['25%','75%'],children:[{id:'protocol',type:'select',label:a.lang.common.protocol,'default':'http://',style:'width : 100%;',items:[['http://'],['https://'],['ftp://'],['news://'],['<other>','']],setup:function(s){if(s.url)this.setValue(s.url.protocol);},commit:function(s){if(!s.url)s.url={};s.url.protocol=this.getValue();}},{type:'text',id:'url',label:a.lang.common.url,onLoad:function(){this.allowOnChange=true;},onKeyUp:function(){var x=this;x.allowOnChange=false;var s=x.getDialog().getContentElement('info','protocol'),t=x.getValue(),u=/^(http|https|ftp|news):\/\/(?=.)/gi,v=/^((javascript:)|[#\/\.])/gi,w=u.exec(t);if(w){x.setValue(t.substr(w[0].length));s.setValue(w[0].toLowerCase());}else if(v.test(t))s.setValue('');x.allowOnChange=true;},onChange:function(){if(this.allowOnChange)this.onKeyUp();},validate:function(){var s=this.getDialog();if(s.getContentElement('info','linkType')&&s.getValueOf('info','linkType')!='url')return true;if(this.getDialog().fakeObj)return true;var t=CKEDITOR.dialog.validate.notEmpty(a.lang.link.noUrl);return t.apply(this);},setup:function(s){var u=this;u.allowOnChange=false;if(s.url)u.setValue(s.url.url);u.allowOnChange=true;var t=u.getDialog().getContentElement('info','linkType');if(t&&t.getValue()=='url')u.select();},commit:function(s){if(!s.url)s.url={};s.url.url=this.getValue();this.allowOnChange=false;}}],setup:function(s){if(!this.getDialog().getContentElement('info','linkType'))this.getElement().show();
}},{type:'button',id:'browse',hidden:'true',filebrowser:'info:url',label:a.lang.common.browseServer}]},{type:'vbox',id:'anchorOptions',width:260,align:'center',padding:0,children:[{type:'html',id:'selectAnchorText',html:CKEDITOR.tools.htmlEncode(a.lang.link.selectAnchor),setup:function(s){if(s.anchors.length>0)this.getElement().show();else this.getElement().hide();}},{type:'html',id:'noAnchors',style:'text-align: center;',html:'<div>'+CKEDITOR.tools.htmlEncode(a.lang.link.noAnchors)+'</div>',setup:function(s){if(s.anchors.length<1)this.getElement().show();else this.getElement().hide();}},{type:'hbox',id:'selectAnchor',children:[{type:'select',id:'anchorName','default':'',label:a.lang.link.anchorName,style:'width: 100%;',items:[['']],setup:function(s){var v=this;v.clear();v.add('');for(var t=0;t<s.anchors.length;t++)if(s.anchors[t].name)v.add(s.anchors[t].name);if(s.anchor)v.setValue(s.anchor.name);var u=v.getDialog().getContentElement('info','linkType');if(u&&u.getValue()=='email')v.focus();},commit:function(s){if(!s.anchor)s.anchor={};s.anchor.name=this.getValue();}},{type:'select',id:'anchorId','default':'',label:a.lang.link.anchorId,style:'width: 100%;',items:[['']],setup:function(s){var u=this;u.clear();u.add('');for(var t=0;t<s.anchors.length;t++)if(s.anchors[t].id)u.add(s.anchors[t].id);if(s.anchor)u.setValue(s.anchor.id);},commit:function(s){if(!s.anchor)s.anchor={};s.anchor.id=this.getValue();}}],setup:function(s){if(s.anchors.length>0)this.getElement().show();else this.getElement().hide();}}],setup:function(s){if(!this.getDialog().getContentElement('info','linkType'))this.getElement().hide();}},{type:'vbox',id:'emailOptions',padding:1,children:[{type:'text',id:'emailAddress',label:a.lang.link.emailAddress,validate:function(){var s=this.getDialog();if(!s.getContentElement('info','linkType')||s.getValueOf('info','linkType')!='email')return true;var t=CKEDITOR.dialog.validate.notEmpty(a.lang.link.noEmail);return t.apply(this);},setup:function(s){if(s.email)this.setValue(s.email.address);var t=this.getDialog().getContentElement('info','linkType');if(t&&t.getValue()=='email')this.select();},commit:function(s){if(!s.email)s.email={};s.email.address=this.getValue();}},{type:'text',id:'emailSubject',label:a.lang.link.emailSubject,setup:function(s){if(s.email)this.setValue(s.email.subject);},commit:function(s){if(!s.email)s.email={};s.email.subject=this.getValue();}},{type:'textarea',id:'emailBody',label:a.lang.link.emailBody,rows:3,'default':'',setup:function(s){if(s.email)this.setValue(s.email.body);
},commit:function(s){if(!s.email)s.email={};s.email.body=this.getValue();}}],setup:function(s){if(!this.getDialog().getContentElement('info','linkType'))this.getElement().hide();}}]},{id:'target',label:a.lang.link.target,title:a.lang.link.target,elements:[{type:'hbox',widths:['50%','50%'],children:[{type:'select',id:'linkTargetType',label:a.lang.link.target,'default':'notSet',style:'width : 100%;',items:[[a.lang.link.targetNotSet,'notSet'],[a.lang.link.targetFrame,'frame'],[a.lang.link.targetPopup,'popup'],[a.lang.link.targetNew,'_blank'],[a.lang.link.targetTop,'_top'],[a.lang.link.targetSelf,'_self'],[a.lang.link.targetParent,'_parent']],onChange:b,setup:function(s){if(s.target)this.setValue(s.target.type);},commit:function(s){if(!s.target)s.target={};s.target.type=this.getValue();}},{type:'text',id:'linkTargetName',label:a.lang.link.targetFrameName,'default':'',setup:function(s){if(s.target)this.setValue(s.target.name);},commit:function(s){if(!s.target)s.target={};s.target.name=this.getValue();}}]},{type:'vbox',width:260,align:'center',padding:2,id:'popupFeatures',children:[{type:'html',html:CKEDITOR.tools.htmlEncode(a.lang.link.popupFeatures)},{type:'hbox',children:[{type:'checkbox',id:'resizable',label:a.lang.link.popupResizable,setup:n,commit:q},{type:'checkbox',id:'status',label:a.lang.link.popupStatusBar,setup:n,commit:q}]},{type:'hbox',children:[{type:'checkbox',id:'location',label:a.lang.link.popupLocationBar,setup:n,commit:q},{type:'checkbox',id:'toolbar',label:a.lang.link.popupToolbar,setup:n,commit:q}]},{type:'hbox',children:[{type:'checkbox',id:'menubar',label:a.lang.link.popupMenuBar,setup:n,commit:q},{type:'checkbox',id:'fullscreen',label:a.lang.link.popupFullScreen,setup:n,commit:q}]},{type:'hbox',children:[{type:'checkbox',id:'scrollbars',label:a.lang.link.popupScrollBars,setup:n,commit:q},{type:'checkbox',id:'dependent',label:a.lang.link.popupDependent,setup:n,commit:q}]},{type:'hbox',children:[{type:'text',widths:['30%','70%'],labelLayout:'horizontal',label:a.lang.link.popupWidth,id:'width',setup:n,commit:q},{type:'text',labelLayout:'horizontal',widths:['55%','45%'],label:a.lang.link.popupLeft,id:'left',setup:n,commit:q}]},{type:'hbox',children:[{type:'text',labelLayout:'horizontal',widths:['30%','70%'],label:a.lang.link.popupHeight,id:'height',setup:n,commit:q},{type:'text',labelLayout:'horizontal',label:a.lang.link.popupTop,widths:['55%','45%'],id:'top',setup:n,commit:q}]}]}]},{id:'upload',label:a.lang.link.upload,title:a.lang.link.upload,hidden:true,filebrowser:'uploadButton',elements:[{type:'file',id:'upload',label:a.lang.common.upload,style:'height:40px',size:29},{type:'fileButton',id:'uploadButton',label:a.lang.common.uploadSubmit,filebrowser:'info:url','for':['upload','upload']}]},{id:'advanced',label:a.lang.link.advanced,title:a.lang.link.advanced,elements:[{type:'vbox',padding:1,children:[{type:'hbox',widths:['45%','35%','20%'],children:[{type:'text',id:'advId',label:a.lang.link.id,setup:o,commit:r},{type:'select',id:'advLangDir',label:a.lang.link.langDir,'default':'',style:'width:110px',items:[[a.lang.link.langDirNotSet,''],[a.lang.link.langDirLTR,'ltr'],[a.lang.link.langDirRTL,'rtl']],setup:o,commit:r},{type:'text',id:'advAccessKey',width:'80px',label:a.lang.link.acccessKey,maxLength:1,setup:o,commit:r}]},{type:'hbox',widths:['45%','35%','20%'],children:[{type:'text',label:a.lang.link.name,id:'advName',setup:o,commit:r},{type:'text',label:a.lang.link.langCode,id:'advLangCode',width:'110px','default':'',setup:o,commit:r},{type:'text',label:a.lang.link.tabIndex,id:'advTabIndex',width:'80px',maxLength:5,setup:o,commit:r}]}]},{type:'vbox',padding:1,children:[{type:'hbox',widths:['45%','55%'],children:[{type:'text',label:a.lang.link.advisoryTitle,'default':'',id:'advTitle',setup:o,commit:r},{type:'text',label:a.lang.link.advisoryContentType,'default':'',id:'advContentType',setup:o,commit:r}]},{type:'hbox',widths:['45%','55%'],children:[{type:'text',label:a.lang.link.cssClasses,'default':'',id:'advCSSClasses',setup:o,commit:r},{type:'text',label:a.lang.link.charset,'default':'',id:'advCharset',setup:o,commit:r}]},{type:'hbox',children:[{type:'text',label:a.lang.link.styles,'default':'',id:'advStyles',setup:o,commit:r}]}]}]}],onShow:function(){var y=this;
y.fakeObj=false;var s=y.getParentEditor(),t=s.getSelection(),u=t.getRanges(),v=null,w=y;if(u.length==1){var x=u[0].getCommonAncestor(true);v=x.getAscendant('a',true);if(v&&v.getAttribute('href'))t.selectElement(v);else if((v=x.getAscendant('img',true))&&(v.getAttribute('_cke_real_element_type')&&v.getAttribute('_cke_real_element_type')=='anchor')){y.fakeObj=v;v=s.restoreRealElement(y.fakeObj);t.selectElement(y.fakeObj);}else v=null;}y.setupContent(l.apply(y,[s,v]));},onOk:function(){var s={href:'javascript:void(0)/*'+CKEDITOR.tools.getNextNumber()+'*/'},t=[],u={href:s.href},v=this,w=this.getParentEditor();this.commitContent(u);switch(u.type||'url'){case 'url':var x=u.url&&u.url.protocol!=undefined?u.url.protocol:'http://',y=u.url&&u.url.url||'';s._cke_saved_href=y.indexOf('/')===0?y:x+y;break;case 'anchor':var z=u.anchor&&u.anchor.name,A=u.anchor&&u.anchor.id;s._cke_saved_href='#'+(z||A||'');break;case 'email':var B=u.email&&u.email.address,C=u.email&&encodeURIComponent(u.email.subject||''),D=u.email&&encodeURIComponent(u.email.body||''),E=['mailto:',B];if(C||D){var F=[];E.push('?');C&&F.push('subject='+C);D&&F.push('body='+D);E.push(F.join('&'));}s._cke_saved_href=E.join('');break;default:}if(u.target)if(u.target.type=='popup'){var G=["window.open(this.href, '",u.target.name||'',"', '"],H=['resizable','status','location','toolbar','menubar','fullscreen','scrollbars','dependent'],I=H.length,J=function(T){if(u.target[T])H.push(T+'='+u.target[T]);};for(var K=0;K<I;K++)H[K]=H[K]+(u.target[H[K]]?'=yes':'=no');J('width');J('left');J('height');J('top');G.push(H.join(','),"'); return false;");s[CKEDITOR.env.ie||CKEDITOR.env.webkit?'_cke_pa_onclick':'onclick']=G.join('');}else{if(u.target.type!='notSet'&&u.target.name)s.target=u.target.name;t.push('_cke_pa_onclick','onclick');}if(u.adv){var L=function(T,U){var V=u.adv[T];if(V)s[U]=V;else t.push(U);};if(this._.selectedElement)L('advId','id');L('advLangDir','dir');L('advAccessKey','accessKey');L('advName','name');L('advLangCode','lang');L('advTabIndex','tabindex');L('advTitle','title');L('advContentType','type');L('advCSSClasses','class');L('advCharset','charset');L('advStyles','style');}if(!this._.selectedElement){var M=w.getSelection(),N=M.getRanges();if(N.length==1&&N[0].collapsed){var O=new CKEDITOR.dom.text(s._cke_saved_href,w.document);N[0].insertNode(O);N[0].selectNodeContents(O);M.selectRanges(N);}var P=new CKEDITOR.style({element:'a',attributes:s});P.type=CKEDITOR.STYLE_INLINE;P.apply(w.document);if(u.adv&&u.adv.advId){var Q=this.getParentEditor().document.$.getElementsByTagName('a');
for(K=0;K<Q.length;K++)if(Q[K].href==s.href){Q[K].id=u.adv.advId;break;}}}else{var R=this._.selectedElement;if(CKEDITOR.env.ie&&s.name!=R.getAttribute('name')){var S=new CKEDITOR.dom.element('<a name="'+CKEDITOR.tools.htmlEncode(s.name)+'">',w.document);M=w.getSelection();R.moveChildren(S);R.copyAttributes(S,{name:1});S.replace(R);R=S;M.selectElement(R);}R.setAttributes(s);R.removeAttributes(t);if(R.getAttribute('name'))R.addClass('cke_anchor');else R.removeClass('cke_anchor');if(this.fakeObj)w.createFakeElement(R,'cke_anchor','anchor').replace(this.fakeObj);delete this._.selectedElement;}},onLoad:function(){if(!a.config.linkShowAdvancedTab)this.hidePage('advanced');if(!a.config.linkShowTargetTab)this.hidePage('target');}};});



```
