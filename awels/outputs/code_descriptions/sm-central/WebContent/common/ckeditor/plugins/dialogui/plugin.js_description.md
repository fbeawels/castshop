# plugin.js

## Review

## 1. Summary  

The file implements CKEditor’s **`dialogui` plugin**, which supplies a collection of reusable UI widgets for dialog boxes (text input, password, textarea, checkbox, radio, button, select, file picker, file button, and raw HTML).  

Key points  

| Component | Role |
|-----------|------|
| `CKEDITOR.plugins.add('dialogui')` | Declares the plugin and registers it with CKEditor. |
| `CKEDITOR.ui.dialog.uiElement` | Base class for all dialog UI elements (provides value handling, element creation, event wiring). |
| Factory functions (`b`, `c`) | Create UI element constructors (`text`, `password`, `textarea`, …). |
| Prototypes (`labeledElement`, `textInput`, …) | Extend `uiElement` to implement specific widget logic, including rendering and state handling. |
| Event processors (`e`) | Centralised logic for handling change events and binding DOM listeners. |
| `CKEDITOR.dialog.addUIElement` | Registers each widget type with CKEditor’s dialog system. |

The code heavily uses CKEditor’s internal APIs (`CKEDITOR.tools`, `CKEDITOR.event`, `CKEDITOR.dom`, `CKEDITOR.env`) and follows a **factory + prototype inheritance** pattern to avoid duplication across widget types.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

1. **Plugin registration** – `CKEDITOR.plugins.add('dialogui')` is called.  
2. **Immediately‑invoked function expression (IIFE)** creates a private scope and defines helpers (`a`, `b`, `c`, `d`, `e`, `f`, `g`).  
3. **Helper `a`** sets up internal state for each UI element (default value, init value, etc.).  
4. **Factory functions**  
   * `b` creates simple “text”‑style inputs.  
   * `c` creates widgets that delegate rendering to their `type` (e.g., checkbox, radio).  
5. **`d`** contains common change‑detection logic (`isChanged`, `reset`, etc.).  
6. **`e`** centralises event handling, especially the `onChange` processor.  
7. **`g`** sanitises user‑supplied definition objects by stripping out event handlers and reserved keys.  
8. **`CKEDITOR.ui.dialog` is extended** with a variety of element constructors: `labeledElement`, `textInput`, `textarea`, `checkbox`, `radio`, `button`, `select`, `file`, `fileButton`, and `html`.  
9. **Prototype chains** are built:  
   * `textInput` extends `labeledElement`, which extends `uiElement`.  
   * `textarea` extends `textInput`.  
   * `select`, `checkbox`, `radio` extend `labeledElement` and add value‑specific logic.  
   * `file` and `fileButton` extend `labeledElement` or `button`.  
10. **`CKEDITOR.dialog.addUIElement`** registers each widget type with the dialog system so that dialog definitions can refer to them via `ui`.  

### 2.2 Core Architecture  

* **Prototype‑Based Inheritance** – Most widgets extend `uiElement`, overriding only the parts that differ (rendering, value handling).  
* **Factory Registration** – Each widget type is registered via `CKEDITOR.dialog.addUIElement`, allowing dialog XML/JS definitions to reference them.  
* **Event Delegation** – `e.onChange` sets up DOM listeners lazily on dialog load, and uses CKEditor’s event bus to propagate changes.  
* **State Management** – Widgets store `initValue` and `value` internally, providing helpers (`isChanged`, `reset`, `setInitValue`, etc.).  

### 2.3 Assumptions & Constraints  

| Assumption | Reason |
|------------|--------|
| The plugin runs in the context of CKEditor 4.x | Uses CKEditor global objects and API version 4 conventions. |
| Browsers support `document.domain` manipulation and `iframe` src trick for file uploads | Required for the `file` widget to work in cross‑domain situations. |
| `CKEDITOR.env` flags correctly detect browser quirks (IE, Gecko, WebKit, etc.) | Used for conditional logic (e.g., `propertychange` events). |
| `CKEDITOR.tools.getNextNumber()` yields unique IDs | All elements rely on unique IDs for DOM interactions. |

### 2.4 Potential Issues  

* **Hard‑coded string concatenation** for HTML (e.g., in `file` widget) may become unwieldy and error‑prone in future maintenance.  
* **Inline JavaScript** (`src="javascript:void(...)"`) in the `file` widget’s iframe may raise security concerns or fail in stricter CSP environments.  
* **IE-specific code** (e.g., `propertychange`, manual `add` method on select) is tightly coupled and may need updates for IE 11+.  
* **No defensive checks** for missing DOM elements (e.g., if `iframe` fails to load, or `this._.inputId` isn’t found).  
* **Event processor registration** relies on a `domOnChangeRegistered` flag that could potentially leak between instances if not reset correctly.  

---

## 3. Functions / Methods  

Below is a categorized list of the main functions and methods, their purpose, signatures, and side‑effects.  

| Name | Purpose | Parameters | Return | Side‑Effects |
|------|---------|------------|--------|--------------|
| **`a(h, …)`** | Initializes UI element state (`_default`, `initValue`). | `h` definition object; additional args for overrides. | Reference to `k._` (state object). | Sets `_default`, `_initValue`. |
| **`b.build(h,i,j)`** | Factory for text‑style inputs (e.g., `text`, `password`). | `h` dialog; `i` definition; `j` id. | New `CKEDITOR.ui.dialog.textInput`. | Creates element, attaches to dialog. |
| **`c.build(h,i,j)`** | Delegated factory that creates an element of the type specified in `i.type`. | Same as `b.build`. | Corresponding UI element. | Same as `b.build`. |
| **`d.isChanged()`** | Checks if current value differs from initial value. | – | Boolean. | – |
| **`d.reset()`** | Resets current value to initial value. | – | – |
| **`d.setInitValue()`** | Stores current value as the new initial value. | – | – |
| **`d.resetInitValue()`** | Restores initial value to the default. | – | – |
| **`d.getInitValue()`** | Retrieves current initial value. | – | String/Number/… | – |
| **`e.onChange(h,i)`** | Event processor that binds DOM `change` events lazily. | `h` event object; `i` callback. | – | Adds `change` listener to input element. |
| **`f`** | RegExp for identifying event handler keys (`onChange`, `onClick`, …). | – | – | – |
| **`g(h)`** | Sanitises a definition object by removing event handlers and reserved keys (`title`, `type`). | `h` definition. | Sanitised object. | – |
| **`CKEDITOR.ui.dialog.labeledElement(...)`** | Base class for widgets that have a label + content area. | `h` dialog; `i` definition; `j` id; `k` content builder. | New instance of `CKEDITOR.ui.dialog.uiElement`. | Creates a `<div>` with label and content. |
| **`CKEDITOR.ui.dialog.textInput(...)`** | Implements a single‑line text input. | `h`, `i`, `j`. | Instance. | Handles focus, key events, value. |
| **`CKEDITOR.ui.dialog.textarea(...)`** | Implements a multi‑line textarea. | `h`, `i`, `j`. | Instance. | Handles size, rows/cols, initial value. |
| **`CKEDITOR.ui.dialog.checkbox(...)`** | Implements a checkbox input. | `h`, `i`, `j`. | Instance. | Creates `<input type="checkbox">` and `<label>`. |
| **`CKEDITOR.ui.dialog.radio(...)`** | Implements a group of radio buttons. | `h`, `i`, `j`. | Instance. | Builds a horizontal box of radio inputs and labels. |
| **`CKEDITOR.ui.dialog.button(...)`** | Implements a generic dialog button. | `h`, `i`, `j`. | Instance. | Handles click, enable/disable, focus. |
| **`CKEDITOR.ui.dialog.select(...)`** | Implements a `<select>` element. | `h`, `i`, `j`. | Instance. | Handles options, multiple, size. |
| **`CKEDITOR.ui.dialog.file(...)`** | Implements a file picker via hidden iframe. | `h`, `i`, `j`. | Instance. | Sets up iframe, handles upload form. |
| **`CKEDITOR.ui.dialog.fileButton(...)`** | Implements a button that triggers file upload. | `h`, `i`, `j`. | Instance. | Calls `submit` on associated file widget. |
| **`CKEDITOR.ui.dialog.html(...)`** | Inserts arbitrary HTML into a dialog. | `h`, `i`, `j`. | Instance. | Parses tag, sets focus, wraps content. |
| **`CKEDITOR.ui.dialog.uiElement` methods** | Common API (`setValue`, `getValue`, `focus`, `select`, `isVisible`, `isEnabled`, `addEventListener`, `fire`, etc.). | – | – | – |
| **`CKEDITOR.ui.dialog.labeledElement.prototype.setLabel()`** | Sets/updates the label text. | `h` string. | Self. | Modifies DOM. |
| **`CKEDITOR.ui.dialog.labeledElement.prototype.getLabel()`** | Retrieves the label text. | – | String. | – |
| **`CKEDITOR.ui.dialog.textInput.prototype.getInputElement()`** | Returns the `<input>` element. | – | DOM element. | – |
| **`CKEDITOR.ui.dialog.textInput.prototype.focus()`** | Focuses the input. | – | – | Calls native focus. |
| **`CKEDITOR.ui.dialog.textInput.prototype.setValue()`** | Sets value, fires change. | `h` value. | Self. | Updates DOM, triggers change event. |
| **`CKEDITOR.ui.dialog.textarea.prototype`** | Inherits from `textInput`; no additional methods. |
| **`CKEDITOR.ui.dialog.select.prototype.add/remove/clear`** | Option manipulation. | – | – | Adds/ removes `<option>`s. |
| **`CKEDITOR.ui.dialog.checkbox.prototype.setValue()`** | Updates checked state. | Boolean. | – |
| **`CKEDITOR.ui.dialog.checkbox.prototype.getValue()`** | Reads checked state. | – | Boolean. | – |
| **`CKEDITOR.ui.dialog.checkbox.prototype.accessKeyUp()`** | Toggles state. | – | – |
| **`CKEDITOR.ui.dialog.radio.prototype.setValue()`** | Sets which radio is checked. | Value. | – |
| **`CKEDITOR.ui.dialog.radio.prototype.getValue()`** | Retrieves selected radio value. | – | Value or null. | – |
| **`CKEDITOR.ui.dialog.radio.prototype.accessKeyUp()`** | Focuses selected radio or first. | – | – |
| **`CKEDITOR.ui.dialog.file.prototype.getInputElement()`** | Returns `<input type="file">` inside iframe. | – | DOM element. | – |
| **`CKEDITOR.ui.dialog.file.prototype.submit()`** | Triggers form submission in the iframe. | – | Self. | – |
| **`CKEDITOR.ui.dialog.file.prototype.reset()`** | Re‑writes the iframe form for a fresh file picker. | – | – |
| **`CKEDITOR.ui.dialog.file.prototype.getAction()`** | Returns upload URL. | – | String. | – |
| **`CKEDITOR.ui.dialog.file.prototype.getValue()`** | Not applicable; always returns empty string. | – | '' | – |
| **`CKEDITOR.ui.dialog.button.prototype.click()`** | Programmatic click handling. | – | Bool (true if handled). | Calls `fire('click')`. |
| **`CKEDITOR.ui.dialog.button.prototype.enable/disable`** | Toggle button state. | – | – | Manipulates DOM class. |
| **`CKEDITOR.ui.dialog.button.prototype.isVisible/ isEnabled`** | State helpers. | – | Bool. | – |
| **`CKEDITOR.ui.dialog.button.prototype.accessKeyUp/Down`** | Keyboard shortcuts. | – | – | Triggers click or focus. |

All prototypes are extended via `CKEDITOR.tools.extend`, ensuring that common properties (e.g., event processors, `keyboardFocusable`) are shared across widgets.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| **CKEDITOR** | Third‑party (core) | Global namespace providing all dialog infrastructure. |
| `CKEDITOR.tools` | CKEditor helper utilities | `extend`, `getNextNumber`, `htmlEncode`, `htmlDecode`, etc. |
| `CKEDITOR.event` | Event system | Used for `implementOn` and custom event propagation. |
| `CKEDITOR.dom` | DOM manipulation | `element`, `text`, `document`, etc. |
| `CKEDITOR.env` | Browser detection | Flags for IE, Gecko, WebKit, custom domain support. |
| `CKEDITOR.ui.dialog` | Internal registry | Holds UI element constructors and prototype chain. |
| `CKEDITOR.dialog` | API for registering UI elements | `addUIElement` used at the end. |

All dependencies are **standard CKEditor 4.x** modules. No external libraries (jQuery, lodash, etc.) are used.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Robustness  

1. **Missing IDs** – The code relies on `CKEDITOR.tools.getNextNumber()` to generate unique IDs. If for any reason this fails or is overridden, DOM lookups (`getElementById`) may return `null`, causing `TypeError`s.  
2. **Cross‑Domain Security** – The `file` widget writes an inline form inside an iframe using `document.domain`. Modern browsers may block this if CSP or `X-Frame-Options` are restrictive.  
3. **IE PropertyChange** – The custom `propertychange` handling is IE‑specific; it may not work in IE 11+ or in Edge’s legacy mode. A feature flag could guard this block.  
4. **Keyboard Accessibility** – While many widgets set `keyboardFocusable: true`, focus management in complex dialogs (especially with custom HTML) may still need manual handling.  

### 5.2 Performance  

* **Lazy binding** – `onChange` attaches listeners only after the dialog loads, which is efficient.  
* **String concatenation** – Many widgets build HTML via array joins, which is acceptable for small UI elements but could be refactored to use template literals or `createElement` for readability.  

### 5.3 Maintainability  

* The file is **monolithic**; splitting each widget into its own module or file would improve readability.  
* Adding comments or JSDoc would aid future contributors, especially around the less obvious parts (e.g., `file` widget’s iframe logic).  
* Refactoring the event processor logic (`e`) into a separate helper object or service could reduce duplication.  

### 5.4 Security  

* The plugin builds raw HTML fragments (`html` widget) directly from dialog definitions. If dialog XML is provided by an untrusted source, this could become an XSS vector. CKEditor’s existing `htmlEncode` is used in some places, but ensuring all user‑supplied strings are escaped is critical.  

### 5.5 Potential Enhancements  

| Idea | Benefit | Effort |
|------|---------|--------|
| **Modernize with ES6 modules** | Cleaner syntax, native classes, better tree‑shaking. | Medium (requires build step). |
| **Template Literals** for HTML builders | Easier to read and modify. | Low‑medium. |
| **Separate event processor registration** | Reduce complexity, allow per‑widget custom processors. | Low. |
| **Accessibility audit** – auto‑generate ARIA attributes for all widgets. | Improves usability for screen readers. | Medium. |
| **CSP‑friendly file upload** – use `fetch` or `XMLHttpRequest` with `FormData` instead of iframe. | Avoids `javascript:` URLs, works in stricter CSP environments. | High (requires re‑design). |
| **Unit tests** – Using CKEditor’s `unit` testing harness to cover each widget’s state changes. | Ensures regressions are caught early. | Medium‑high. |

---

### 5.4 Final Verdict  

The plugin is **functionally complete** for CKEditor 4.x dialogs and leverages the editor’s internal APIs effectively. Its design—factory + prototype inheritance—reduces duplication and keeps the public API minimal.  

However, the code’s **readability** is hindered by heavy minification and lack of documentation. Future‑proofing (modern CSP, Edge, IE 11+ compatibility) and **modularisation** are recommended to make the plugin easier to evolve. Security and accessibility aspects deserve closer scrutiny, especially for the `file` widget in contemporary browsers.  

With modest refactoring and added documentation, this plugin can serve as a solid foundation for custom dialog widgets in CKEditor projects.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.plugins.add('dialogui');(function(){var a=function(h){var k=this;k._||(k._={});k._['default']=k._.initValue=h['default']||'';var i=[k._];for(var j=1;j<arguments.length;j++)i.push(arguments[j]);i.push(true);CKEDITOR.tools.extend.apply(CKEDITOR.tools,i);return k._;},b={build:function(h,i,j){return new CKEDITOR.ui.dialog.textInput(h,i,j);}},c={build:function(h,i,j){return new CKEDITOR.ui.dialog[i.type](h,i,j);}},d={isChanged:function(){return this.getValue()!=this.getInitValue();},reset:function(){this.setValue(this.getInitValue());},setInitValue:function(){this._.initValue=this.getValue();},resetInitValue:function(){this._.initValue=this._['default'];},getInitValue:function(){return this._.initValue;}},e=CKEDITOR.tools.extend({},CKEDITOR.ui.dialog.uiElement.prototype.eventProcessors,{onChange:function(h,i){if(!this._.domOnChangeRegistered){h.on('load',function(){this.getInputElement().on('change',function(){this.fire('change',{value:this.getValue()});},this);},this);this._.domOnChangeRegistered=true;}this.on('change',i);}},true),f=/^on([A-Z]\w+)/,g=function(h){for(var i in h)if(f.test(i)||i=='title'||i=='type')delete h[i];return h;};CKEDITOR.tools.extend(CKEDITOR.ui.dialog,{labeledElement:function(h,i,j,k){if(arguments.length<4)return;var l=a.call(this,i);l.labelId=CKEDITOR.tools.getNextNumber()+'_label';var m=this._.children=[],n=function(){var o=[];if(i.labelLayout!='horizontal')o.push('<div class="cke_dialog_ui_labeled_label" id="',l.labelId,'" >',i.label,'</div>','<div class="cke_dialog_ui_labeled_content">',k(h,i),'</div>');else{var p={type:'hbox',widths:i.widths,padding:0,children:[{type:'html',html:'<span class="cke_dialog_ui_labeled_label" id="'+l.labelId+'">'+CKEDITOR.tools.htmlEncode(i.label)+'</span>'},{type:'html',html:'<span class="cke_dialog_ui_labeled_content">'+k(h,i)+'</span>'}]};CKEDITOR.dialog._.uiElementBuilders.hbox.build(h,p,o);}return o.join('');};CKEDITOR.ui.dialog.uiElement.call(this,h,i,j,'div',null,null,n);},textInput:function(h,i,j){if(arguments.length<3)return;a.call(this,i);var k=this._.inputId=CKEDITOR.tools.getNextNumber()+'_textInput',l={'class':'cke_dialog_ui_input_'+i.type,id:k,type:'text'},m;if(i.validate)this.validate=i.validate;if(i.maxLength)l.maxlength=i.maxLength;if(i.size)l.size=i.size;var n=this,o=false;h.on('load',function(){n.getInputElement().on('keydown',function(q){if(q.data.getKeystroke()==13)o=true;});n.getInputElement().on('keyup',function(q){if(q.data.getKeystroke()==13&&o){h.getButton('ok')&&h.getButton('ok').click();
o=false;}},null,null,1000);});var p=function(){var q=['<div class="cke_dialog_ui_input_',i.type,'"'];if(i.width)q.push('style="width:'+i.width+'" ');q.push('><input ');for(var r in l)q.push(r+'="'+l[r]+'" ');q.push(' /></div>');return q.join('');};CKEDITOR.ui.dialog.labeledElement.call(this,h,i,j,p);},textarea:function(h,i,j){if(arguments.length<3)return;a.call(this,i);var k=this,l=this._.inputId=CKEDITOR.tools.getNextNumber()+'_textarea',m={};if(i.validate)this.validate=i.validate;m.rows=i.rows||5;m.cols=i.cols||20;var n=function(){var o=['<div class="cke_dialog_ui_input_textarea"><textarea class="cke_dialog_ui_input_textarea" id="',l,'" '];for(var p in m)o.push(p+'="'+CKEDITOR.tools.htmlEncode(m[p])+'" ');o.push('>',CKEDITOR.tools.htmlEncode(k._['default']),'</textarea></div>');return o.join('');};CKEDITOR.ui.dialog.labeledElement.call(this,h,i,j,n);},checkbox:function(h,i,j){if(arguments.length<3)return;var k=a.call(this,i,{'default':!!i['default']});if(i.validate)this.validate=i.validate;var l=function(){var m=CKEDITOR.tools.extend({},i,{id:i.id?i.id+'_checkbox':CKEDITOR.tools.getNextNumber()+'_checkbox'},true),n=[],o={'class':'cke_dialog_ui_checkbox_input',type:'checkbox'};g(m);if(i['default'])o.checked='checked';k.checkbox=new CKEDITOR.ui.dialog.uiElement(h,m,n,'input',null,o);n.push(' <label for="',o.id,'">',CKEDITOR.tools.htmlEncode(i.label),'</label>');return n.join('');};CKEDITOR.ui.dialog.uiElement.call(this,h,i,j,'span',null,null,l);},radio:function(h,i,j){if(arguments.length<3)return;a.call(this,i);if(!this._['default'])this._['default']=this._.initValue=i.items[0][1];if(i.validate)this.validate=i.valdiate;var k=[],l=this,m=function(){var n=[],o=[],p={'class':'cke_dialog_ui_radio_item'},q=i.id?i.id+'_radio':CKEDITOR.tools.getNextNumber()+'_radio';for(var r=0;r<i.items.length;r++){var s=i.items[r],t=s[2]!==undefined?s[2]:s[0],u=s[1]!==undefined?s[1]:s[0],v=CKEDITOR.tools.extend({},i,{id:CKEDITOR.tools.getNextNumber()+'_radio_input',title:null,type:null},true),w=CKEDITOR.tools.extend({},v,{id:null,title:t},true),x={type:'radio','class':'cke_dialog_ui_radio_input',name:q,value:u},y=[];if(l._['default']==u)x.checked='checked';g(v);g(w);k.push(new CKEDITOR.ui.dialog.uiElement(h,v,y,'input',null,x));y.push(' ');new CKEDITOR.ui.dialog.uiElement(h,w,y,'label',null,{'for':x.id},s[0]);n.push(y.join(''));}new CKEDITOR.ui.dialog.hbox(h,[],n,o);return o.join('');};CKEDITOR.ui.dialog.labeledElement.call(this,h,i,j,m);this._.children=k;},button:function(h,i,j){if(!arguments.length)return;
if(typeof i=='function')i=i(h.getParentEditor());a.call(this,i,{disabled:i.disabled||false});CKEDITOR.event.implementOn(this);var k=this;h.on('load',function(m){var n=this.getElement();(function(){n.on('click',function(o){k.fire('click',{dialog:k.getDialog()});o.data.preventDefault();});})();n.unselectable();},this);var l=CKEDITOR.tools.extend({},i);delete l.style;CKEDITOR.ui.dialog.uiElement.call(this,h,l,j,'a',null,{style:i.style,href:'javascript:void(0)',title:i.label,hidefocus:'true','class':i['class']},'<span class="cke_dialog_ui_button">'+CKEDITOR.tools.htmlEncode(i.label)+'</span>');},select:function(h,i,j){if(arguments.length<3)return;var k=a.call(this,i);if(i.validate)this.validate=i.validate;var l=function(){var m=CKEDITOR.tools.extend({},i,{id:i.id?i.id+'_select':CKEDITOR.tools.getNextNumber()+'_select'},true),n=[],o=[],p={'class':'cke_dialog_ui_input_select'};if(i.size!=undefined)p.size=i.size;if(i.multiple!=undefined)p.multiple=i.multiple;g(m);for(var q=0,r;q<i.items.length&&(r=i.items[q]);q++)o.push('<option value="',CKEDITOR.tools.htmlEncode(r[1]!==undefined?r[1]:r[0]),'" /> ',CKEDITOR.tools.htmlEncode(r[0]));k.select=new CKEDITOR.ui.dialog.uiElement(h,m,n,'select',null,p,o.join(''));return n.join('');};CKEDITOR.ui.dialog.labeledElement.call(this,h,i,j,l);},file:function(h,i,j){if(arguments.length<3)return;if(i['default']===undefined)i['default']='';var k=CKEDITOR.tools.extend(a.call(this,i),{definition:i,buttons:[]});if(i.validate)this.validate=i.validate;var l=function(){k.frameId=CKEDITOR.tools.getNextNumber()+'_fileInput';var m=CKEDITOR.env.isCustomDomain(),n=['<iframe frameborder="0" allowtransparency="0" class="cke_dialog_ui_input_file" id="',k.frameId,'" title="',i.label,'" src="javascript:void('];n.push(m?"(function(){document.open();document.domain='"+document.domain+"';"+'document.close();'+'})()':'0');n.push(')"></iframe>');return n.join('');};h.on('load',function(){var m=CKEDITOR.document.getById(k.frameId),n=m.getParent();n.addClass('cke_dialog_ui_input_file');});CKEDITOR.ui.dialog.labeledElement.call(this,h,i,j,l);},fileButton:function(h,i,j){if(arguments.length<3)return;var k=a.call(this,i),l=this;if(i.validate)this.validate=i.validate;var m=CKEDITOR.tools.extend({},i),n=m.onClick;m.className=(m.className?m.className+' ':'')+('cke_dialog_ui_button');m.onClick=function(o){var p=i['for'];if(!n||n.call(this,o)!==false){h.getContentElement(p[0],p[1]).submit();this.disable();}};h.on('load',function(){h.getContentElement(i['for'][0],i['for'][1])._.buttons.push(l);
});CKEDITOR.ui.dialog.button.call(this,h,m,j);},html:(function(){var h=/^\s*<[\w:]+\s+([^>]*)?>/,i=/^(\s*<[\w:]+(?:\s+[^>]*)?)((?:.|\r|\n)+)$/,j=/\/$/;return function(k,l,m){if(arguments.length<3)return;var n=[],o,p=l.html,q,r;if(p.charAt(0)!='<')p='<span>'+p+'</span>';if(l.focus){var s=this.focus;this.focus=function(){s.call(this);l.focus.call(this);this.fire('focus');};if(l.isFocusable){var t=this.isFocusable;this.isFocusable=t;}this.keyboardFocusable=true;}CKEDITOR.ui.dialog.uiElement.call(this,k,l,n,'span',null,null,'');o=n.join('');q=o.match(h);r=p.match(i)||['','',''];if(j.test(r[1])){r[1]=r[1].slice(0,-1);r[2]='/'+r[2];}m.push([r[1],' ',q[1]||'',r[2]].join(''));};})()},true);CKEDITOR.ui.dialog.html.prototype=new CKEDITOR.ui.dialog.uiElement();CKEDITOR.ui.dialog.labeledElement.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.uiElement(),{setLabel:function(h){var i=CKEDITOR.document.getById(this._.labelId);if(i.getChildCount()<1)new CKEDITOR.dom.text(h,CKEDITOR.document).appendTo(i);else i.getChild(0).$.nodeValue=h;return this;},getLabel:function(){var h=CKEDITOR.document.getById(this._.labelId);if(!h||h.getChildCount()<1)return '';else return h.getChild(0).getText();},eventProcessors:e},true);CKEDITOR.ui.dialog.button.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.uiElement(),{click:function(){var h=this;if(!h._.disabled)return h.fire('click',{dialog:h._.dialog});h.getElement().$.blur();return false;},enable:function(){this._.disabled=false;var h=this.getElement();h&&h.removeClass('disabled');},disable:function(){this._.disabled=true;this.getElement().addClass('disabled');},isVisible:function(){return!!this.getElement().$.firstChild.offsetHeight;},isEnabled:function(){return!this._.disabled;},eventProcessors:CKEDITOR.tools.extend({},CKEDITOR.ui.dialog.uiElement.prototype.eventProcessors,{onClick:function(h,i){this.on('click',i);}},true),accessKeyUp:function(){this.click();},accessKeyDown:function(){this.focus();},keyboardFocusable:true},true);CKEDITOR.ui.dialog.textInput.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.labeledElement(),{getInputElement:function(){return CKEDITOR.document.getById(this._.inputId);},focus:function(){var h=this.selectParentTab();setTimeout(function(){var i=h.getInputElement();i&&i.$.focus();},0);},select:function(){var h=this.selectParentTab();setTimeout(function(){var i=h.getInputElement();if(i){i.$.focus();i.$.select();}},0);},accessKeyUp:function(){this.select();},setValue:function(h){h=h||'';return CKEDITOR.ui.dialog.uiElement.prototype.setValue.call(this,h);
},keyboardFocusable:true},d,true);CKEDITOR.ui.dialog.textarea.prototype=new CKEDITOR.ui.dialog.textInput();CKEDITOR.ui.dialog.select.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.labeledElement(),{getInputElement:function(){return this._.select.getElement();},add:function(h,i,j){var k=new CKEDITOR.dom.element('option',this.getDialog().getParentEditor().document),l=this.getInputElement().$;k.$.text=h;k.$.value=i===undefined||i===null?h:i;if(j===undefined||j===null){if(CKEDITOR.env.ie)l.add(k.$);else l.add(k.$,null);}else l.add(k.$,j);return this;},remove:function(h){var i=this.getInputElement().$;i.remove(h);return this;},clear:function(){var h=this.getInputElement().$;while(h.length>0)h.remove(0);return this;},keyboardFocusable:true},d,true);CKEDITOR.ui.dialog.checkbox.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.uiElement(),{getInputElement:function(){return this._.checkbox.getElement();},setValue:function(h){this.getInputElement().$.checked=h;this.fire('change',{value:h});},getValue:function(){return this.getInputElement().$.checked;},accessKeyUp:function(){this.setValue(!this.getValue());},eventProcessors:{onChange:function(h,i){if(!CKEDITOR.env.ie)return e.onChange.apply(this,arguments);else{h.on('load',function(){var j=this._.checkbox.getElement();j.on('propertychange',function(k){k=k.data.$;if(k.propertyName=='checked')this.fire('change',{value:j.$.checked});},this);},this);this.on('change',i);}return null;}},keyboardFocusable:true},d,true);CKEDITOR.ui.dialog.radio.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.uiElement(),{setValue:function(h){var i=this._.children,j;for(var k=0;k<i.length&&(j=i[k]);k++)j.getElement().$.checked=j.getValue()==h;this.fire('change',{value:h});},getValue:function(){var h=this._.children;for(var i=0;i<h.length;i++)if(h[i].getElement().$.checked)return h[i].getValue();return null;},accessKeyUp:function(){var h=this._.children,i;for(i=0;i<h.length;i++)if(h[i].getElement().$.checked){h[i].getElement().focus();return;}h[0].getElement().focus();},eventProcessors:{onChange:function(h,i){if(!CKEDITOR.env.ie)return e.onChange.apply(this,arguments);else{h.on('load',function(){var j=this._.children,k=this;for(var l=0;l<j.length;l++){var m=j[l].getElement();m.on('propertychange',function(n){n=n.data.$;if(n.propertyName=='checked'&&this.$.checked)k.fire('change',{value:this.getAttribute('value')});});}},this);this.on('change',i);}return null;}},keyboardFocusable:true},d,true);CKEDITOR.ui.dialog.file.prototype=CKEDITOR.tools.extend(new CKEDITOR.ui.dialog.labeledElement(),d,{getInputElement:function(){var h=CKEDITOR.document.getById(this._.frameId).getFrameDocument();
return h.$.forms.length>0?new CKEDITOR.dom.element(h.$.forms[0].elements[0]):this.getElement();},submit:function(){this.getInputElement().getParent().$.submit();return this;},getAction:function(h){return this.getInputElement().getParent().$.action;},reset:function(){var h=CKEDITOR.document.getById(this._.frameId),i=h.getFrameDocument(),j=this._.definition,k=this._.buttons;function l(){i.$.open();if(CKEDITOR.env.isCustomDomain())i.$.domain=document.domain;var m='';if(j.size)m=j.size-(CKEDITOR.env.ie?7:0);i.$.write(['<html><head><title></title></head><body style="margin: 0; overflow: hidden; background: transparent;">','<form enctype="multipart/form-data" method="POST" action="',CKEDITOR.tools.htmlEncode(j.action),'">','<input type="file" name="',CKEDITOR.tools.htmlEncode(j.id||'cke_upload'),'" size="',CKEDITOR.tools.htmlEncode(m>0?m:''),'" />','</form>','</body></html>'].join(''));i.$.close();for(var n=0;n<k.length;n++)k[n].enable();};if(CKEDITOR.env.gecko)setTimeout(l,500);else l();},getValue:function(){return '';},eventProcessors:e,keyboardFocusable:true},true);CKEDITOR.ui.dialog.fileButton.prototype=new CKEDITOR.ui.dialog.button();CKEDITOR.dialog.addUIElement('text',b);CKEDITOR.dialog.addUIElement('password',b);CKEDITOR.dialog.addUIElement('textarea',c);CKEDITOR.dialog.addUIElement('checkbox',c);CKEDITOR.dialog.addUIElement('radio',c);CKEDITOR.dialog.addUIElement('button',c);CKEDITOR.dialog.addUIElement('select',c);CKEDITOR.dialog.addUIElement('file',c);CKEDITOR.dialog.addUIElement('fileButton',c);CKEDITOR.dialog.addUIElement('html',c);})();



```
