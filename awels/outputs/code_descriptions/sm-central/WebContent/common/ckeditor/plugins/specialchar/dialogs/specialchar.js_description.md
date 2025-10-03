# specialchar.js

## Review

## 1. Summary  

The file implements the **Special Characters** dialog for CKEditor.  
When a user opens the dialog, a table of printable characters is rendered.  
Clicking a character inserts it into the editor, while hovering over a character shows a large preview and its raw HTML code.  

Key points  
* **CKEditor integration** – the dialog is created via `CKEDITOR.dialog.add`.  
* **UI construction** – a 320 px wide table is generated in `onLoad`; each cell contains an `<a>` that calls a CKEditor callback.  
* **Keyboard navigation** – an inline key‑down handler (`h`) lets the user move the focus with arrow keys and insert a character with Space/Enter.  
* **Preview logic** – two helper functions (`f` & `g`) update the character preview area and clear it on mouse‑out.  

The code is tightly coupled to CKEditor’s legacy APIs (`CKEDITOR.tools.addFunction`, `CKEDITOR.dom.event`, etc.) and relies heavily on inline JavaScript and string‑based HTML generation.

---

## 2. Detailed Description  

### 2.1 Core Flow  

| Step | What happens | Why it matters |
|------|--------------|----------------|
| **Dialog registration** | `CKEDITOR.dialog.add('specialchar', function(a){…})` | Registers the dialog under the name `specialchar`. |
| **Variable `b`** | Holds the dialog instance (`this`). | Used by helper functions to access dialog elements. |
| **Function `c`** | Called when a character cell is clicked. Determines the target element (passed through `i.data` or `new CKEDITOR.dom.element(i)`) and inserts the character’s inner HTML. | Performs the actual insertion into the editor. |
| **Variable `d`** | `CKEDITOR.tools.addFunction(c)` – registers `c` and returns a function ID used in the cell’s `onclick`. | Binds the click handler to CKEditor’s callback system. |
| **Variables `e`, `f`, `g`** | `e` stores the currently hovered `<a>` (or its parent anchor). `f` updates the preview on mouse‑over; `g` clears the preview on mouse‑out. | Provide a richer UX for character selection. |
| **Function `h`** | Handles key‑down events on the table cells. Supports arrow navigation, Space/Enter for insertion, Tab for cycling, Shift+9 for reverse navigation. | Makes the dialog keyboard‑accessible. |
| **`onLoad`** | Builds the HTML table of characters, assigns event handlers (`onkeydown`, `onclick`). | Dynamically creates the UI each time the dialog is shown. |
| **`contents`** | Defines the dialog layout: a two‑column horizontal box (`hbox`) containing the character table and a preview panel. | Organises the UI into a predictable structure. |
| **Dialog options** | Title, size, button configuration, character list and columns. | Expose configuration options to CKEditor and to developers. |

### 2.2 Assumptions & Constraints  

* **CKEditor 3.x / 4.x** – The code uses legacy APIs (`addFunction`, `tools.callFunction`). It will not work unchanged with CKEditor 5.  
* **Browser support** – Uses `CKEDITOR.dom.event` and inline styles; older IE support is implicit.  
* **Character list** – Hard‑coded array of HTML entities and literal characters. Some duplicates (`&rsquo;`) exist; no filtering is performed.  
* **Focus management** – Relies on `setTimeout` to move focus to the first cell on `onShow`. This may cause timing issues on slow browsers.  

### 2.3 Architecture & Design Choices  

* **Closure‑based helpers** – All helper functions (`c`, `f`, `g`, `h`) are defined inside the dialog factory closure so they have access to `a` (CKEditor instance), `b` (dialog), and `e`.  
* **Dynamic UI construction** – The table is built on every `onLoad`, ensuring fresh state but incurring CPU cost.  
* **Inline handlers** – Event callbacks (`onclick`, `onkeydown`) are written as inline string references to `CKEDITOR.tools.callFunction(...)`. This is a legacy pattern, but it keeps the code concise.  
* **Keyboard navigation** – Implements a custom focus traversal over a non‑standard table layout, mimicking spreadsheet navigation.  

---

## 3. Functions/Methods  

| Function | Purpose | Parameters | Return / Side‑Effects |
|----------|---------|------------|-----------------------|
| `c(i)` | Insert the character from the clicked cell into the editor. | `i`: event data from CKEditor (`i.data` contains the target element or the raw element). | Calls `a.insertHtml(k)`; hides dialog via `b.hide()`. |
| `d` | Result of `CKEDITOR.tools.addFunction(c)`. | None | Integer function ID used in `onclick`. |
| `f(i, j)` | Mouse‑over handler for character cells. Updates preview panels. | `i`: unused; `j`: target element. | Sets preview text; marks cell’s parent with `cke_light_background`; stores reference in `e`. |
| `g(i, j)` | Mouse‑out handler. Clears preview panels and background class. | `i`: unused; `j`: target element. | Clears preview HTML; removes `cke_light_background`; clears `e`. |
| `h(i)` | Key‑down handler for navigation and insertion. | `i`: DOM event. | Prevents default navigation; manipulates focus; may call `c`. |
| Dialog `onLoad` | Builds the character table. | None | Sets inner HTML of the `charContainer`. |
| `contents` | CKEditor dialog definition. | None | Returns an object describing dialog layout and behavior. |

**Reusable / Utility Methods**  
* `CKEDITOR.tools.addFunction` – registers a callback and returns a function ID.  
* `CKEDITOR.dom.event` – cross‑browser event wrapper used by `h`.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party (CKEditor 3.x/4.x) | Provides the dialog API, DOM utilities, event wrapper, and language support. |
| `CKEDITOR.tools` | CKEditor internals | `addFunction`, `callFunction`, `htmlEncode`. |
| `CKEDITOR.dom` | CKEditor internals | Element creation and manipulation. |
| CSS | Inline styles | All UI styling is hard‑coded in the generated HTML. |
| Browser | Standard DOM | No external polyfills; relies on CKEditor’s abstractions. |

---

## 5. Additional Notes  

### 5.1 Edge Cases & Limitations  

* **Duplicate entities** – `&rsquo;` appears twice in the list; no deduplication logic.  
* **Non‑renderable characters** – The code assumes every cell’s inner HTML can be inserted verbatim; non‑character entities or malformed HTML may break the editor.  
* **Focus issues** – The `setTimeout` hack may fail if the dialog is shown quickly or on a low‑performance device.  
* **Keyboard navigation** – Arrow handling is hard‑coded for a specific table layout; adding or removing columns changes the logic.  
* **Internationalization** – The dialog title and tab labels come from `a.lang`, but the character list is fixed; extending to other scripts would require a new list.  

### 5.2 Potential Enhancements  

| Area | Suggested Improvement |
|------|-----------------------|
| **Code readability** | Replace inline string callbacks with proper `addEventListener`/`addEvent` usage. |
| **Modernization** | Rewrite using ES6+ syntax (arrow functions, template literals) and remove `CKEDITOR.tools.addFunction` if migrating to CKEditor 5. |
| **Separation of concerns** | Extract table generation into a dedicated function; keep UI markup separate. |
| **Accessibility** | Add ARIA roles to the table cells; provide `aria-label` attributes for characters. |
| **Performance** | Cache the generated table HTML and reuse it instead of rebuilding on every `onLoad`. |
| **Extensibility** | Load the character list from a JSON or i18n file; allow user‑defined sets. |
| **Testing** | Add unit tests for the helper functions and integration tests for the dialog. |
| **Duplication cleanup** | Remove duplicate entries or deduplicate automatically. |
| **Error handling** | Gracefully handle cases where `j.getChild(0)` is undefined or not an element. |

### 5.3 Migration to CKEditor 5  

If the project moves to CKEditor 5, the entire dialog architecture must be replaced:  
* CKEditor 5 uses a completely different plugin API (`@ckeditor/ckeditor5-core`).  
* `CKEDITOR.tools.addFunction` is no longer available; use native event listeners.  
* The dialog UI would be built with the `@ckeditor/ckeditor5-ui` components.  

---

**Conclusion** – The code fulfills its purpose of providing a special‑character picker in CKEditor 3/4. However, its heavy reliance on legacy CKEditor APIs, inline event handlers, and manual string manipulation makes it hard to maintain and extend. Refactoring towards a modern, component‑based approach would improve readability, accessibility, and future‑proofness.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('specialchar',function(a){var b,c=function(i){var j,k;if(i.data)j=i.data.getTarget();else j=new CKEDITOR.dom.element(i);if(j.getName()=='a'&&(k=j.getChild(0).getHtml())){j.removeClass('cke_light_background');b.hide();a.insertHtml(k);}},d=CKEDITOR.tools.addFunction(c),e,f=function(i,j){var k;j=j||i.data.getTarget();if(j.getName()=='span')j=j.getParent();if(j.getName()=='a'&&(k=j.getChild(0).getHtml())){if(e)g(null,e);var l=b.getContentElement('info','htmlPreview').getElement();b.getContentElement('info','charPreview').getElement().setHtml(k);l.setHtml(CKEDITOR.tools.htmlEncode(k));j.getParent().addClass('cke_light_background');e=j;}},g=function(i,j){j=j||i.data.getTarget();if(j.getName()=='span')j=j.getParent();if(j.getName()=='a'){b.getContentElement('info','charPreview').getElement().setHtml('&nbsp;');b.getContentElement('info','htmlPreview').getElement().setHtml('&nbsp;');j.getParent().removeClass('cke_light_background');e=undefined;}},h=CKEDITOR.tools.addFunction(function(i){i=new CKEDITOR.dom.event(i);var j=i.getTarget(),k,l,m=i.getKeystroke();switch(m){case 39:if(k=j.getParent().getNext()){l=k.getChild(0);if(l.type==1){l.focus();g(null,j);f(null,l);}}i.preventDefault();break;case 37:if(k=j.getParent().getPrevious()){l=k.getChild(0);l.focus();g(null,j);f(null,l);}i.preventDefault();break;case 38:if(k=j.getParent().getParent().getPrevious()){l=k.getChild([j.getParent().getIndex(),0]);l.focus();g(null,j);f(null,l);}i.preventDefault();break;case 40:if(k=j.getParent().getParent().getNext()){l=k.getChild([j.getParent().getIndex(),0]);if(l&&l.type==1){l.focus();g(null,j);f(null,l);}}i.preventDefault();break;case 32:c({data:i});i.preventDefault();break;case 9:if(k=j.getParent().getNext()){l=k.getChild(0);if(l.type==1){l.focus();g(null,j);f(null,l);i.preventDefault(true);}else g(null,j);}else if(k=j.getParent().getParent().getNext()){l=k.getChild([0,0]);if(l&&l.type==1){l.focus();g(null,j);f(null,l);i.preventDefault(true);}else g(null,j);}break;case CKEDITOR.SHIFT+9:if(k=j.getParent().getPrevious()){l=k.getChild(0);l.focus();g(null,j);f(null,l);i.preventDefault(true);}else if(k=j.getParent().getParent().getPrevious()){l=k.getLast().getChild(0);l.focus();g(null,j);f(null,l);i.preventDefault(true);}else g(null,j);break;default:return;}});return{title:a.lang.specialChar.title,minWidth:430,minHeight:280,buttons:[CKEDITOR.dialog.cancelButton],charColumns:17,chars:['!','&quot;','#','$','%','&amp;',"'",'(',')','*','+','-','.','/','0','1','2','3','4','5','6','7','8','9',':',';','&lt;','=','&gt;','?','@','A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z','[',']','^','_','`','a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z','{','|','}','~','&euro;','&lsquo;','&rsquo;','&rsquo;','&ldquo;','&rdquo;','&ndash;','&mdash;','&iexcl;','&cent;','&pound;','&curren;','&yen;','&brvbar;','&sect;','&uml;','&copy;','&ordf;','&laquo;','&not;','&reg;','&macr;','&deg;','&plusmn;','&sup2;','&sup3;','&acute;','&micro;','&para;','&middot;','&cedil;','&sup1;','&ordm;','&raquo;','&frac14;','&frac12;','&frac34;','&iquest;','&Agrave;','&Aacute;','&Acirc;','&Atilde;','&Auml;','&Aring;','&AElig;','&Ccedil;','&Egrave;','&Eacute;','&Ecirc;','&Euml;','&Igrave;','&Iacute;','&Icirc;','&Iuml;','&ETH;','&Ntilde;','&Ograve;','&Oacute;','&Ocirc;','&Otilde;','&Ouml;','&times;','&Oslash;','&Ugrave;','&Uacute;','&Ucirc;','&Uuml;','&Yacute;','&THORN;','&szlig;','&agrave;','&aacute;','&acirc;','&atilde;','&auml;','&aring;','&aelig;','&ccedil;','&egrave;','&eacute;','&ecirc;','&euml;','&igrave;','&iacute;','&icirc;','&iuml;','&eth;','&ntilde;','&ograve;','&oacute;','&ocirc;','&otilde;','&ouml;','&divide;','&oslash;','&ugrave;','&uacute;','&ucirc;','&uuml;','&uuml;','&yacute;','&thorn;','&yuml;','&OElig;','&oelig;','&#372;','&#374','&#373','&#375;','&sbquo;','&#8219;','&bdquo;','&hellip;','&trade;','&#9658;','&bull;','&rarr;','&rArr;','&hArr;','&diams;','&asymp;'],onLoad:function(){var i=this.definition.charColumns,j=this.definition.chars,k=['<table style="width: 320px; height: 100%; border-collapse: separate;" align="center" cellspacing="2" cellpadding="2" border="0">'],l=0;
while(l<j.length){k.push('<tr>');for(var m=0;m<i;m++,l++){if(j[l])k.push('<td class="cke_dark_background" style="cursor: default"><a href="javascript: void(0);" style="cursor: inherit; display: block; height: 1.25em; margin-top: 0.25em; text-align: center;" title="',j[l].replace(/&/g,'&amp;'),'" onkeydown="CKEDITOR.tools.callFunction( '+h+', event, this )"'+' onclick="CKEDITOR.tools.callFunction('+d+', this); return false;"'+' tabindex="-1">'+'<span style="margin: 0 auto;cursor: inherit">'+j[l]+'</span></a>');else k.push('<td class="cke_dark_background">&nbsp;');k.push('</td>');}k.push('</tr>');}k.push('</tbody></table>');this.getContentElement('info','charContainer').getElement().setHtml(k.join(''));},contents:[{id:'info',label:a.lang.common.generalTab,title:a.lang.common.generalTab,padding:0,align:'top',elements:[{type:'hbox',align:'top',widths:['320px','90px'],children:[{type:'html',id:'charContainer',html:'',onMouseover:f,onMouseout:g,focus:function(){var i=this.getElement().getChild([0,0,0,0,0]);setTimeout(function(){i.focus();f(null,i);});},onShow:function(){var i=this.getElement().getChild([0,0,0,0,0]);setTimeout(function(){i.focus();f(null,i);});},onLoad:function(i){b=i.sender;}},{type:'hbox',align:'top',widths:['100%'],children:[{type:'vbox',align:'top',children:[{type:'html',html:'<div></div>'},{type:'html',id:'charPreview',style:"border:1px solid #eeeeee;background-color:#EAEAD1;font-size:28px;height:40px;width:70px;padding-top:9px;font-family:'Microsoft Sans Serif',Arial,Helvetica,Verdana;text-align:center;",html:'<div>&nbsp;</div>'},{type:'html',id:'htmlPreview',style:"border:1px solid #eeeeee;background-color:#EAEAD1;font-size:14px;height:20px;width:70px;padding-top:2px;font-family:'Microsoft Sans Serif',Arial,Helvetica,Verdana;text-align:center;",html:'<div>&nbsp;</div>'}]}]}]}]}]};});



```
