# table.js

## Review

## 1. Summary  
- **Purpose** – The file implements the **Table** and **Table Properties** dialogs used in CKEditor.  
- **Key components**  
  - Regular‑expression helpers (`a`, `b`) for parsing width/height values.  
  - Utility `c` that stores the dialog field value into `e.info`.  
  - Factory function `d(e, f)` that builds the dialog configuration object.  
  - Two public dialog registrations (`table`, `tableProperties`) that call `d`.  
- **Design patterns & libraries** – The code is written in an **IIFE** style and heavily uses CKEditor’s own API: `CKEDITOR.dialog`, `CKEDITOR.dom.element`, `CKEDITOR.env`, `CKEDITOR.tools`, `CKEDITOR.dialog.validate`. No external third‑party libraries are required.

---

## 2. Detailed Description  
### 2.1 Core Flow  
1. **Dialog creation** – `CKEDITOR.dialog.add('table', …)` and `CKEDITOR.dialog.add('tableProperties', …)` invoke `d(e, f)`.  
2. **`d(e, f)` returns a dialog descriptor**  
   - `title`, `minWidth`, `minHeight` are set.  
   - `onShow`:  
     - Detects whether a table is selected (`selectedElement`).  
     - Disables or enables fields based on whether a table is being edited or a new one is inserted.  
     - Calls `setupContent` on the existing table to pre‑populate fields.  
   - `onOk`:  
     - Commits user input into the `info` object.  
     - Either **creates** a new table (if no existing table) or **updates** the selected table.  
     - Handles header rows/columns, cell spacing/padding, caption, summary, alignment, border, width/height styles.  
3. **Contents definition** – Three main tabs (`info`, `advanced`, `caption`). Each contains nested containers (`hbox`, `vbox`) and fields (`text`, `select`). Every field defines `validate`, `setup`, and `commit` callbacks.  

### 2.2 Assumptions & Constraints  
- **Environment** – Relies on `CKEDITOR.env.ie` to apply IE‑specific sizing.  
- **Table structure** – Assumes a single `<thead>` or none, and that the first row contains header cells if present. Complex tables (multiple `<thead>`/`<tbody>`/`<tfoot>`) are not fully supported.  
- **Style parsing** – Only handles `width` in pixels or percentage, and `height` in pixels. Other style units are ignored.  
- **Browser compatibility** – Uses IE detection; may not cover all quirks of modern browsers.  

### 2.3 Architecture & Design Choices  
- **Single function per dialog** – Keeps the entire dialog logic in one closure, reducing global pollution.  
- **`c` helper** – Centralizes the pattern of saving field values to the dialog’s `info` object.  
- **Nested callbacks** – Each field’s `setup` and `commit` are defined inline, tightly coupling UI with data handling.  
- **Dynamic enabling/disabling** – Field states are changed on `onShow` to reflect whether the dialog is used for insertion or editing.  

---

## 3. Functions/Methods  
| Function / Method | Purpose | Inputs | Outputs / Side‑Effects |
|-------------------|---------|--------|------------------------|
| `c(e)` | Stores the current field’s value into `e.info` using the field’s id as key. | `e` – event object from the dialog field. | Adds/updates `info[id]` on the dialog instance. |
| `d(e, f)` | Factory that builds the dialog descriptor for either “table” or “tableProperties”. | `e` – dialog definition object; `f` – string (`'table'` or `'tableProperties'`). | Returns an object containing `title`, `minWidth`, `minHeight`, `onShow`, `onOk`, `contents`. |
| `g(h)` – inner to `d` | Helper that creates a new `CKEDITOR.dom.element` from tag name `h` in the dialog’s document. | `h` – tag name string. | New `CKEDITOR.dom.element` instance. |
| `onShow` – in dialog descriptor | Initializes dialog fields based on current selection or defaults. | – | Manipulates field values, enables/disables inputs. |
| `onOk` – in dialog descriptor | Persists user input to the editor – either creating a new table or updating the existing one. | – | Inserts/updates DOM elements, returns `true` to close dialog. |
| **Field callbacks** (`setup`, `commit`, `validate`) | Each dialog field defines these to (a) populate with current table values, (b) validate user input, (c) write changes back to the `info` object. | `h` – table element (for `setup`), `i` – info object (for `commit`). | Sets field values, updates `info`, shows alerts on validation failure. |

---

## 4. Dependencies  
| Library / API | Type | Notes |
|--------------|------|-------|
| `CKEDITOR` | Core library | Provides dialog API, DOM utilities, environment detection, and validation helpers. |
| `CKEDITOR.dom.element` | CKEditor | Abstracted DOM manipulation. |
| `CKEDITOR.env` | CKEditor | Browser/environment detection (used for IE sizing). |
| `CKEDITOR.tools` | CKEditor | `trim` helper for string cleanup. |
| `CKEDITOR.dialog.validate` | CKEditor | Input validation helpers (integer, number). |
| No external third‑party libraries are referenced. |

---

## 5. Additional Notes  
### 5.1 Readability & Maintainability  
- **Obfuscated variable names** (`a`, `b`, `c`, `d`, `e`, `f`, `g`, `h`, `i`, …) make the code hard to read. Refactoring to descriptive names (`widthRegex`, `heightRegex`, `commitField`, `createDialog`, `doc`, `selected`, …) would greatly improve comprehension.  
- **Inline functions** (especially the many `setup`/`commit` callbacks) create deep nesting and reduce reusability. Extracting common logic (e.g., parsing integer values, handling style units) into helper functions would reduce duplication.  
- **Hard‑coded strings** for validation messages (`e.lang.table.invalidRows`) assume the presence of a language file. Consider using a dedicated i18n mechanism or passing a messages object.  

### 5.2 Edge Cases & Limitations  
- **Header detection**: The algorithm for `hasColumnHeaders` only checks the first cell of every row, which fails for tables with complex header configurations.  
- **Multiple `<thead>`/`<tbody>`**: The code only deals with a single `<thead>` and first `<tbody>`. Tables with multiple sections may be mishandled.  
- **Style parsing**: Only `px` and `%` are supported. Other units (`em`, `rem`, `vh`, etc.) are ignored, potentially losing user input.  
- **IE-specific logic** (`CKEDITOR.env.ie`) might be outdated for modern browsers.  

### 5.3 Potential Enhancements  
1. **Modularization** – Split the dialog into separate modules (`infoTab.js`, `advancedTab.js`, `captionTab.js`) for easier testing and extension.  
2. **TypeScript / ES6+** – Re‑implement with modern syntax to benefit from modules, classes, and clearer type safety.  
3. **Robust validation** – Add regex‑based validation for width/height, support for CSS units, and better error handling.  
4. **Accessibility** – Ensure ARIA attributes and keyboard navigation for the dialog fields.  
5. **Unit tests** – Write automated tests for the dialog logic, especially `onShow` and `onOk`, to guard against regressions when refactoring.  

### 5.4 Performance  
The dialog operates on small DOM fragments, so performance is unlikely to be a bottleneck. However, the repeated creation of many `CKEDITOR.dom.element` objects could be optimized by caching frequently used elements.  

---

**Overall assessment** – The code delivers the required functionality but suffers from poor readability, heavy coupling, and limited robustness. Refactoring with modern JavaScript practices and better separation of concerns would greatly improve maintainability and future extensibility.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){var a=/^(\d+(?:\.\d+)?)(px|%)$/,b=/^(\d+(?:\.\d+)?)px$/,c=function(e){var f=this.id;if(!e.info)e.info={};e.info[f]=this.getValue();};function d(e,f){var g=function(h){return new CKEDITOR.dom.element(h,e.document);};return{title:e.lang.table.title,minWidth:310,minHeight:CKEDITOR.env.ie?310:280,onShow:function(){var o=this;var h=e.getSelection(),i=h.getRanges(),j=null,k=o.getContentElement('info','txtRows'),l=o.getContentElement('info','txtCols'),m=o.getContentElement('info','txtWidth');if(f=='tableProperties'){if(j=e.getSelection().getSelectedElement()){if(j.getName()!='table')j=null;}else if(i.length>0){var n=i[0].getCommonAncestor(true);j=n.getAscendant('table',true);}o._.selectedElement=j;}if(j){o.setupContent(j);k&&k.disable();l&&l.disable();m&&m.select();}else{k&&k.enable();l&&l.enable();k&&k.select();}},onOk:function(){var A=this;var h=A._.selectedElement||g('table'),i=A,j={};A.commitContent(j,h);if(j.info){var k=j.info;if(!A._.selectedElement){var l=h.append(g('tbody')),m=parseInt(k.txtRows,10)||0,n=parseInt(k.txtCols,10)||0;for(var o=0;o<m;o++){var p=l.append(g('tr'));for(var q=0;q<n;q++){var r=p.append(g('td'));if(!CKEDITOR.env.ie)r.append(g('br'));}}}var s=k.selHeaders;if(!h.$.tHead&&(s=='row'||s=='both')){var t=new CKEDITOR.dom.element(h.$.createTHead());l=h.getElementsByTag('tbody').getItem(0);var u=l.getElementsByTag('tr').getItem(0);for(o=0;o<u.getChildCount();o++){var v=u.getChild(o);if(v.type==CKEDITOR.NODE_ELEMENT){v.renameNode('th');v.setAttribute('scope','col');}}t.append(u.remove());}if(h.$.tHead!==null&&!(s=='row'||s=='both')){t=new CKEDITOR.dom.element(h.$.tHead);l=h.getElementsByTag('tbody').getItem(0);var w=l.getFirst();while(t.getChildCount()>0){u=t.getFirst();for(o=0;o<u.getChildCount();o++){var x=u.getChild(o);if(x.type==CKEDITOR.NODE_ELEMENT){x.renameNode('td');x.removeAttribute('scope');}}u.insertBefore(w);}t.remove();}if(!A.hasColumnHeaders&&(s=='col'||s=='both'))for(p=0;p<h.$.rows.length;p++){x=new CKEDITOR.dom.element(h.$.rows[p].cells[0]);x.renameNode('th');x.setAttribute('scope','row');}if(A.hasColumnHeaders&&!(s=='col'||s=='both'))for(o=0;o<h.$.rows.length;o++){p=new CKEDITOR.dom.element(h.$.rows[o]);if(p.getParent().getName()=='tbody'){x=new CKEDITOR.dom.element(p.$.cells[0]);x.renameNode('td');x.removeAttribute('scope');}}var y=[];if(k.txtHeight)y.push('height:'+k.txtHeight+'px');if(k.txtWidth){var z=k.cmbWidthType||'pixels';y.push('width:'+k.txtWidth+(z=='pixels'?'px':'%'));}y=y.join(';');if(y)h.$.style.cssText=y;
else h.removeAttribute('style');}if(!A._.selectedElement)e.insertElement(h);return true;},contents:[{id:'info',label:e.lang.table.title,elements:[{type:'hbox',widths:[null,null],styles:['vertical-align:top'],children:[{type:'vbox',padding:0,children:[{type:'text',id:'txtRows','default':3,label:e.lang.table.rows,style:'width:5em',validate:function(){var h=true,i=this.getValue();h=h&&CKEDITOR.dialog.validate.integer()(i)&&i>0;if(!h){alert(e.lang.table.invalidRows);this.select();}return h;},setup:function(h){this.setValue(h.$.rows.length);},commit:c},{type:'text',id:'txtCols','default':2,label:e.lang.table.columns,style:'width:5em',validate:function(){var h=true,i=this.getValue();h=h&&CKEDITOR.dialog.validate.integer()(i)&&i>0;if(!h){alert(e.lang.table.invalidCols);this.select();}return h;},setup:function(h){this.setValue(h.$.rows[0].cells.length);},commit:c},{type:'html',html:'&nbsp;'},{type:'select',id:'selHeaders','default':'',label:e.lang.table.headers,items:[[e.lang.table.headersNone,''],[e.lang.table.headersRow,'row'],[e.lang.table.headersColumn,'col'],[e.lang.table.headersBoth,'both']],setup:function(h){var i=this.getDialog();i.hasColumnHeaders=true;for(var j=0;j<h.$.rows.length;j++)if(h.$.rows[j].cells[0].nodeName.toLowerCase()!='th'){i.hasColumnHeaders=false;break;}if(h.$.tHead!==null)this.setValue(i.hasColumnHeaders?'both':'row');else this.setValue(i.hasColumnHeaders?'col':'');},commit:c},{type:'text',id:'txtBorder','default':1,label:e.lang.table.border,style:'width:3em',validate:CKEDITOR.dialog.validate.number(e.lang.table.invalidBorder),setup:function(h){this.setValue(h.getAttribute('border')||'');},commit:function(h,i){if(this.getValue())i.setAttribute('border',this.getValue());else i.removeAttribute('border');}},{id:'cmbAlign',type:'select','default':'',label:e.lang.table.align,items:[[e.lang.table.alignNotSet,''],[e.lang.table.alignLeft,'left'],[e.lang.table.alignCenter,'center'],[e.lang.table.alignRight,'right']],setup:function(h){this.setValue(h.getAttribute('align')||'');},commit:function(h,i){if(this.getValue())i.setAttribute('align',this.getValue());else i.removeAttribute('align');}}]},{type:'vbox',padding:0,children:[{type:'hbox',widths:['5em'],children:[{type:'text',id:'txtWidth',style:'width:5em',label:e.lang.table.width,'default':200,validate:CKEDITOR.dialog.validate.number(e.lang.table.invalidWidth),setup:function(h){var i=a.exec(h.$.style.width);if(i)this.setValue(i[1]);},commit:c},{id:'cmbWidthType',type:'select',label:'&nbsp;','default':'pixels',items:[[e.lang.table.widthPx,'pixels'],[e.lang.table.widthPc,'percents']],setup:function(h){var i=a.exec(h.$.style.width);
if(i)this.setValue(i[2]=='px'?'pixels':'percents');},commit:c}]},{type:'hbox',widths:['5em'],children:[{type:'text',id:'txtHeight',style:'width:5em',label:e.lang.table.height,'default':'',validate:CKEDITOR.dialog.validate.number(e.lang.table.invalidHeight),setup:function(h){var i=b.exec(h.$.style.height);if(i)this.setValue(i[1]);},commit:c},{type:'html',html:'<br />'+e.lang.table.widthPx}]},{type:'html',html:'&nbsp;'},{type:'text',id:'txtCellSpace',style:'width:3em',label:e.lang.table.cellSpace,'default':1,validate:CKEDITOR.dialog.validate.number(e.lang.table.invalidCellSpacing),setup:function(h){this.setValue(h.getAttribute('cellSpacing')||'');},commit:function(h,i){if(this.getValue())i.setAttribute('cellSpacing',this.getValue());else i.removeAttribute('cellSpacing');}},{type:'text',id:'txtCellPad',style:'width:3em',label:e.lang.table.cellPad,'default':1,validate:CKEDITOR.dialog.validate.number(e.lang.table.invalidCellPadding),setup:function(h){this.setValue(h.getAttribute('cellPadding')||'');},commit:function(h,i){if(this.getValue())i.setAttribute('cellPadding',this.getValue());else i.removeAttribute('cellPadding');}}]}]},{type:'html',align:'right',html:''},{type:'vbox',padding:0,children:[{type:'text',id:'txtCaption',label:e.lang.table.caption,setup:function(h){var i=h.getElementsByTag('caption');if(i.count()>0){var j=i.getItem(0);j=j.getChild(0)&&j.getChild(0).getText()||'';j=CKEDITOR.tools.trim(j);this.setValue(j);}},commit:function(h,i){var j=this.getValue(),k=i.getElementsByTag('caption');if(j){if(k.count()>0){k=k.getItem(0);k.setHtml('');}else{k=new CKEDITOR.dom.element('caption',e.document);if(i.getChildCount())k.insertBefore(i.getFirst());else k.appendTo(i);}k.append(new CKEDITOR.dom.text(j,e.document));}else if(k.count()>0)for(var l=k.count()-1;l>=0;l--)k.getItem(l).remove();}},{type:'text',id:'txtSummary',label:e.lang.table.summary,setup:function(h){this.setValue(h.getAttribute('summary')||'');},commit:function(h,i){if(this.getValue())i.setAttribute('summary',this.getValue());}}]}]}]};};CKEDITOR.dialog.add('table',function(e){return d(e,'table');});CKEDITOR.dialog.add('tableProperties',function(e){return d(e,'tableProperties');});})();



```
