# plugin.js

## Review

## 1. Summary  

**Purpose**  
The snippet implements CKEditor’s selection engine.  It provides:

* A **`selection` plugin** that watches for user‑initiated selection changes (mouse, keyboard, touch) and fires a `selectionChange` event with useful meta‑data (the element that was selected, the element path, etc.).
* A **`dom.selection`** abstraction that normalises the native browser selection APIs, handling differences between IE, Gecko, WebKit and Blink.
* A **`dom.range`** helper that can be used to read and manipulate ranges, including creating bookmarks for undo/redo.

**Key components**

| Component | Role |
|-----------|------|
| `CKEDITOR.plugins.add('selection', …)` | Registers the plugin, attaches listeners, creates the `SelectAll` command/button. |
| `getSelection()` / `forceNextSelectionCheck()` | Editor helpers that delegate to the document’s selection. |
| `CKEDITOR.dom.selection` | Core selection abstraction: cache, locking, range handling, type detection. |
| `CKEDITOR.dom.range` | Range wrapper with `select()` method that translates back into a native selection. |
| `createBookmarks` / `selectBookmarks` | Bookmark helpers for undo/redo or restoring a selection after DOM changes. |

**Design patterns & libraries**

* The code uses the **Module pattern** (`(function(){ … })();`) to encapsulate helper functions (`a`, `d`, `e`).
* It relies heavily on CKEditor’s **utility toolkit** (`CKEDITOR.tools.setTimeout`, `CKEDITOR.env`, `CKEDITOR.tools`).
* Cross‑browser differences are isolated into IE‑specific branches, using feature detection (`CKEDITOR.env.ie`, `CKEDITOR.env.gecko`, etc.).

---

## 2. Detailed Description  

### Initialization  

1. **Plugin registration** – `CKEDITOR.plugins.add('selection', { init: function(editor) { … } })`.  
2. Inside `init` the plugin registers an event listener on the editor’s `contentDom` event.  
3. Once the document is ready, it wires a set of handlers (`mousedown`, `mouseup`, `keydown`, `keyup`, `selectionchange`, `focusin`, `beforedeactivate`, etc.) that ultimately trigger a deferred `selectionChange` event.  
4. For non‑IE browsers the handlers call `d` (schedule a check) which in turn calls `a` (perform the actual selection comparison).

### Runtime behaviour  

* **Selection detection**  
  * `a()` obtains the current selection (`getSelection()`), resolves the start element, builds an `elementPath`, and compares it to the previous cached path (`k._.selectionPreviousPath`).  
  * If the path differs, it fires `selectionChange` with `{ selection, path, element }`.

* **Selection locking** – `CKEDITOR.dom.selection.lock()` freezes the current selection so that subsequent DOM changes (e.g., filtering, undo) do not alter it.  
* **Range handling** – `getRanges()` normalises the native ranges to CKEditor’s `CKEDITOR.dom.range` objects.  
* **Bookmark creation** – `createBookmarks()`/`createBookmarks2()` serialise the current selection into stable node identifiers so it can be restored after a document rewrite.  

### Cleanup  

* The plugin does not explicitly unregister listeners; it relies on CKEditor’s lifecycle (`editor.destroy`) to clean up the DOM references.  
* `reset()` clears the selection cache when the selection changes.  

### Assumptions & Constraints  

* The code assumes that `document.$` is the native DOM `document`.  
* IE is handled specially using IE’s `TextRange` and `ControlRange`.  
* The implementation expects a relatively small number of ranges (usually 1–2).  
* `CKEDITOR.tools.setTimeout` is used to throttle selection checks and avoid excessive event firing.

---

## 3. Functions/Methods  

| Function/Method | Purpose | Inputs | Outputs | Side‑effects |
|-----------------|---------|--------|---------|--------------|
| `a()` | Checks if the current selection path differs from the cached one; if so, fires `selectionChange`. | None (uses `this` = editor) | None | Fires `selectionChange` event |
| `d()` | Schedules a selection change check; ensures only one timer runs. | None | None | Sets a `setTimeout` timer |
| `e()` | Clears the scheduled timer and triggers a deferred check. | None | None | Calls `a()` immediately |
| `exec(h)` | Command handler for `selectAll`. In WYSIWYG mode, calls `execCommand('SelectAll')`. | `h` = editor instance | None | Executes browser command |
| `init(h)` | Plugin initialiser; attaches DOM listeners, registers command/button, sets up `selectionChange` handler. | `h` = editor instance | None | Registers event listeners |
| `getSelection()` (editor prototype) | Convenience wrapper to fetch the current selection. | None | `CKEDITOR.dom.selection` | None |
| `forceNextSelectionCheck()` (editor prototype) | Clears cached previous path so next selection will always trigger an event. | None | None | Deletes internal cache |
| `getSelection()` (document prototype) | Returns the current selection or `null` if invalid. | None | `CKEDITOR.dom.selection` or `null` | None |
| `CKEDITOR.dom.selection.constructor` | Initializes the selection abstraction; caches the native range and sets lock state. | `h` = document | `CKEDITOR.dom.selection` | Stores native range |
| `getNative()` | Returns the underlying native selection object, cached for efficiency. | None | `Selection` / `TextRange` | None |
| `getType()` | Determines the type of the selection (none, text, element). | None | `CKEDITOR.SELECTION_*` enum | Caches result |
| `getRanges()` | Normalises native ranges into `CKEDITOR.dom.range` objects. | None | Array of ranges | Caches result |
| `getStartElement()` | Returns the element where the selection starts. | None | `CKEDITOR.dom.element` or `null` | Caches result |
| `getSelectedElement()` | For element selections, returns the selected element node. | None | `CKEDITOR.dom.element` or `null` | Caches result |
| `lock()` | Freezes the current selection, preventing further changes until unlocked. | None | None | Sets lock flag, caches range/element |
| `unlock(h)` | Restores the selection after a DOM rewrite; optionally re‑selects ranges or element. | `h` = `true` if we should restore | None | Clears lock flag, resets selection |
| `reset()` | Clears all cached selection data. | None | None | Empties cache |
| `selectElement(h)` | Selects the entire element `h` using native ranges or IE control ranges. | `h` = `CKEDITOR.dom.element` | None | Changes native selection |
| `selectRanges(h)` | Selects an array of `CKEDITOR.dom.range` objects. | `h` = array of ranges | None | Changes native selection |
| `createBookmarks(h)` | Creates stable bookmark objects for the current selection, suitable for undo/redo. | `h` = optional boolean to include serialisable flag | Array of bookmarks | Caches bookmarks |
| `createBookmarks2(h)` | Same as `createBookmarks` but uses the newer bookmark format. | `h` | Array of bookmarks | |
| `selectBookmarks(h)` | Restores a selection from an array of bookmarks. | `h` | `this` (chainable) | Restores selection |
| `CKEDITOR.dom.range.prototype.select()` | Serialises the range into a native selection. | `a` = optional boolean to preserve anchor? | None | Calls native `select()` |

### Reusable / Utility methods

* `CKEDITOR.dom.selection.getNative()`, `getType()`, `getRanges()` – generic helpers for any selection object.  
* `CKEDITOR.dom.selection.lock()/unlock()` – useful for transactions where the selection must stay constant.  
* `createBookmarks()`/`selectBookmarks()` – widely used by the undo/redo stack.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **CKEDITOR core** (`CKEDITOR`, `CKEDITOR.env`, `CKEDITOR.tools`) | Standard | Provides environment detection, timers, node types, etc. |
| **DOM API** (`document`, `Selection`, `Range`, `TextRange`, `ControlRange`) | Standard | Browser‑native selection and range APIs. |
| **`CKEDITOR.dom.elementPath`** | CKEditor library | Used for building element path for the `selectionChange` event. |
| **`CKEDITOR.dom.range`** | CKEditor library | Provides the range abstraction that this module builds upon. |

No external third‑party libraries are used; all dependencies are part of the CKEditor core distribution. Platform‑specific handling is limited to IE via feature detection (`CKEDITOR.env.ie`).

---

## 5. Additional Notes  

### Edge cases & potential bugs  

* **IE edge cases** – The code uses `try/catch` blocks around `createRange()` and `select()` calls. If the range becomes invalid (e.g., after a DOM modification), the catch silently swallows the error, which could mask bugs during development.  
* **Selection collapse in WebKit** – The logic that appends an invisible text node (`&#65279;`) to create a collapsible range is a known WebKit trick. However, it may interfere with content that contains zero‑width spaces.  
* **Large selection counts** – `getRanges()` assumes a small number of ranges; in browsers that support multiple non‑contiguous ranges (e.g., Chrome’s `document.getSelection().rangeCount > 1`), the code still processes all ranges but may not handle them correctly if the ranges overlap.  
* **Bookmark serialisation** – `createBookmarks` stores `startNode`/`endNode` ids but does not guard against node removal between bookmark creation and restoration. A missing node will cause an exception when `selectBookmarks` is called.  
* **Undo/redo integration** – The plugin does not expose a dedicated API for the undo manager; it relies on `CKEDITOR.dom.selection`’s internal `createBookmarks`/`selectBookmarks`. If the undo stack is extended (e.g., to support “replace all”), this code may need adaptation.

### Performance considerations  

* The use of `CKEDITOR.tools.setTimeout` with `0` and `50` ms delays helps throttle events but introduces a small latency for the `selectionChange` event. For high‑frequency scenarios (drag‑select, rapid key‑presses) this may lead to slightly delayed UI updates.  
* IE’s `TextRange` creation and `moveToElementText()` are expensive; the plugin caches ranges (`_cache`) to mitigate repeated work.

### Future enhancements  

1. **Centralised event throttling** – Move the scheduling logic into a dedicated `SelectionChangeThrottle` helper that can be reused by other plugins.  
2. **More robust bookmark handling** – Validate node existence before restoring; optionally provide a fallback “select nearest” strategy.  
3. **Support for `RangeList`** – In browsers that support `RangeList` (Chrome, Safari), expose a lightweight API to work directly with the native list.  
4. **Unit tests** – Add a comprehensive test suite covering all branches (IE vs. non‑IE, text vs. element selections, multi‑range).  
5. **Accessibility hooks** – Emit ARIA events (`aria-selected`) when a range is selected programmatically.  

---

### Bottom line  

The code is a solid, pragmatic implementation of selection handling for a complex WYSIWYG editor.  It judiciously isolates browser quirks, offers a clean API for other parts of CKEditor, and leverages the existing toolkit effectively.  Minor clean‑ups around error handling, documentation, and test coverage would raise the maintainability further, but the core logic is sound and well‑structured.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){function a(){var k=this;try{var h=k.getSelection();if(!h)return;var i=h.getStartElement(),j=new CKEDITOR.dom.elementPath(i);if(!j.compare(k._.selectionPreviousPath)){k._.selectionPreviousPath=j;k.fire('selectionChange',{selection:h,path:j,element:i});}}catch(l){}};var b,c;function d(){c=true;if(b)return;e.call(this);b=CKEDITOR.tools.setTimeout(e,200,this);};function e(){b=null;if(c){CKEDITOR.tools.setTimeout(a,0,this);c=false;}};var f={exec:function(h){switch(h.mode){case 'wysiwyg':h.document.$.execCommand('SelectAll',false,null);break;case 'source':}},canUndo:false};CKEDITOR.plugins.add('selection',{init:function(h){h.on('contentDom',function(){var i=h.document;if(CKEDITOR.env.ie){var j,k;i.on('focusin',function(){if(j){try{j.select();}catch(n){}j=null;}});h.window.on('focus',function(){k=true;m();});h.document.on('beforedeactivate',function(){k=false;h.document.$.execCommand('Unselect');});i.on('mousedown',l);i.on('mouseup',function(){k=true;setTimeout(function(){m(true);},0);});i.on('keydown',l);i.on('keyup',function(){k=true;m();});i.on('selectionchange',m);function l(){k=false;};function m(n){if(k){var o=h.document,p=o&&o.$.selection;if(n&&p&&p.type=='None')if(!o.$.queryCommandEnabled('InsertImage')){CKEDITOR.tools.setTimeout(m,50,this,true);return;}j=p&&p.createRange();d.call(h);}};}else{i.on('mouseup',d,h);i.on('keyup',d,h);}});h.addCommand('selectAll',f);h.ui.addButton('SelectAll',{label:h.lang.selectAll,command:'selectAll'});h.selectionChange=d;}});CKEDITOR.editor.prototype.getSelection=function(){return this.document&&this.document.getSelection();};CKEDITOR.editor.prototype.forceNextSelectionCheck=function(){delete this._.selectionPreviousPath;};CKEDITOR.dom.document.prototype.getSelection=function(){var h=new CKEDITOR.dom.selection(this);return!h||h.isInvalid?null:h;};CKEDITOR.SELECTION_NONE=1;CKEDITOR.SELECTION_TEXT=2;CKEDITOR.SELECTION_ELEMENT=3;CKEDITOR.dom.selection=function(h){var k=this;var i=h.getCustomData('cke_locked_selection');if(i)return i;k.document=h;k.isLocked=false;k._={cache:{}};if(CKEDITOR.env.ie){var j=k.getNative().createRange();if(!j||j.item&&j.item(0).ownerDocument!=k.document.$||j.parentElement&&j.parentElement().ownerDocument!=k.document.$)k.isInvalid=true;}return k;};var g={img:1,hr:1,li:1,table:1,tr:1,td:1,embed:1,object:1,ol:1,ul:1,a:1,input:1,form:1,select:1,textarea:1,button:1,fieldset:1,th:1,thead:1,tfoot:1};CKEDITOR.dom.selection.prototype={getNative:CKEDITOR.env.ie?function(){return this._.cache.nativeSel||(this._.cache.nativeSel=this.document.$.selection);
}:function(){return this._.cache.nativeSel||(this._.cache.nativeSel=this.document.getWindow().$.getSelection());},getType:CKEDITOR.env.ie?function(){var h=this._.cache;if(h.type)return h.type;var i=CKEDITOR.SELECTION_NONE;try{var j=this.getNative(),k=j.type;if(k=='Text')i=CKEDITOR.SELECTION_TEXT;if(k=='Control')i=CKEDITOR.SELECTION_ELEMENT;if(j.createRange().parentElement)i=CKEDITOR.SELECTION_TEXT;}catch(l){}return h.type=i;}:function(){var h=this._.cache;if(h.type)return h.type;var i=CKEDITOR.SELECTION_TEXT,j=this.getNative();if(!j)i=CKEDITOR.SELECTION_NONE;else if(j.rangeCount==1){var k=j.getRangeAt(0),l=k.startContainer;if(l==k.endContainer&&l.nodeType==1&&k.endOffset-k.startOffset==1&&g[l.childNodes[k.startOffset].nodeName.toLowerCase()])i=CKEDITOR.SELECTION_ELEMENT;}return h.type=i;},getRanges:CKEDITOR.env.ie?(function(){var h=function(i,j){i=i.duplicate();i.collapse(j);var k=i.parentElement(),l=k.childNodes,m;for(var n=0;n<l.length;n++){var o=l[n];if(o.nodeType==1){m=i.duplicate();m.moveToElementText(o);m.collapse();var p=m.compareEndPoints('StartToStart',i);if(p>0)break;else if(p===0)return{container:k,offset:n};m=null;}}if(!m){m=i.duplicate();m.moveToElementText(k);m.collapse(false);}m.setEndPoint('StartToStart',i);var q=m.text.replace(/(\r\n|\r)/g,'\n').length;while(q>0)q-=l[--n].nodeValue.length;if(q===0)return{container:k,offset:n};else return{container:l[n],offset:-q};};return function(){var t=this;var i=t._.cache;if(i.ranges)return i.ranges;var j=t.getNative(),k=j&&j.createRange(),l=t.getType(),m;if(!j)return[];if(l==CKEDITOR.SELECTION_TEXT){m=new CKEDITOR.dom.range(t.document);var n=h(k,true);m.setStart(new CKEDITOR.dom.node(n.container),n.offset);n=h(k);m.setEnd(new CKEDITOR.dom.node(n.container),n.offset);return i.ranges=[m];}else if(l==CKEDITOR.SELECTION_ELEMENT){var o=t._.cache.ranges=[];for(var p=0;p<k.length;p++){var q=k.item(p),r=q.parentNode,s=0;m=new CKEDITOR.dom.range(t.document);for(;s<r.childNodes.length&&r.childNodes[s]!=q;s++){}m.setStart(new CKEDITOR.dom.node(r),s);m.setEnd(new CKEDITOR.dom.node(r),s+1);o.push(m);}return o;}return i.ranges=[];};})():function(){var h=this._.cache;if(h.ranges)return h.ranges;var i=[],j=this.getNative();if(!j)return[];for(var k=0;k<j.rangeCount;k++){var l=j.getRangeAt(k),m=new CKEDITOR.dom.range(this.document);m.setStart(new CKEDITOR.dom.node(l.startContainer),l.startOffset);m.setEnd(new CKEDITOR.dom.node(l.endContainer),l.endOffset);i.push(m);}return h.ranges=i;},getStartElement:function(){var o=this;
var h=o._.cache;if(h.startElement!==undefined)return h.startElement;var i,j=o.getNative();switch(o.getType()){case CKEDITOR.SELECTION_ELEMENT:return o.getSelectedElement();case CKEDITOR.SELECTION_TEXT:var k=o.getRanges()[0];if(k)if(!k.collapsed){k.optimize();for(;;){var l=k.startContainer,m=k.startOffset;if(m==(l.getChildCount?l.getChildCount():l.getLength()))k.setStartAfter(l);else break;}i=k.startContainer;if(i.type!=CKEDITOR.NODE_ELEMENT)return i.getParent();i=i.getChild(k.startOffset);if(!i||i.type!=CKEDITOR.NODE_ELEMENT)return k.startContainer;var n=i.getFirst();while(n&&n.type==CKEDITOR.NODE_ELEMENT){i=n;n=n.getFirst();}return i;}if(CKEDITOR.env.ie){k=j.createRange();k.collapse(true);i=k.parentElement();}else{i=j.anchorNode;if(i&&i.nodeType!=1)i=i.parentNode;}}return h.startElement=i?new CKEDITOR.dom.element(i):null;},getSelectedElement:function(){var h=this._.cache;if(h.selectedElement!==undefined)return h.selectedElement;var i;if(this.getType()==CKEDITOR.SELECTION_ELEMENT){var j=this.getNative();if(CKEDITOR.env.ie)try{i=j.createRange().item(0);}catch(l){}else{var k=j.getRangeAt(0);i=k.startContainer.childNodes[k.startOffset];}}return h.selectedElement=i?new CKEDITOR.dom.element(i):null;},lock:function(){var h=this;h.getRanges();h.getStartElement();h.getSelectedElement();h._.cache.nativeSel={};h.isLocked=true;h.document.setCustomData('cke_locked_selection',h);},unlock:function(h){var m=this;var i=m.document,j=i.getCustomData('cke_locked_selection');if(j){i.setCustomData('cke_locked_selection',null);if(h){var k=j.getSelectedElement(),l=!k&&j.getRanges();m.isLocked=false;m.reset();i.getBody().focus();if(k)m.selectElement(k);else m.selectRanges(l);}}if(!j||!h){m.isLocked=false;m.reset();}},reset:function(){this._.cache={};},selectElement:function(h){var k=this;if(k.isLocked){var i=new CKEDITOR.dom.range(k.document);i.setStartBefore(h);i.setEndAfter(h);k._.cache.selectedElement=h;k._.cache.startElement=h;k._.cache.ranges=[i];k._.cache.type=CKEDITOR.SELECTION_ELEMENT;return;}if(CKEDITOR.env.ie){k.getNative().empty();try{i=k.document.$.body.createControlRange();i.addElement(h.$);i.select();}catch(l){i=k.document.$.body.createTextRange();i.moveToElementText(h.$);i.select();}k.reset();}else{i=k.document.$.createRange();i.selectNode(h.$);var j=k.getNative();j.removeAllRanges();j.addRange(i);k.reset();}},selectRanges:function(h){var n=this;if(n.isLocked){n._.cache.selectedElement=null;n._.cache.startElement=h[0].getTouchedStartNode();n._.cache.ranges=h;n._.cache.type=CKEDITOR.SELECTION_TEXT;
return;}if(CKEDITOR.env.ie){if(h[0])h[0].select();n.reset();}else{var i=n.getNative();i.removeAllRanges();for(var j=0;j<h.length;j++){var k=h[j],l=n.document.$.createRange(),m=k.startContainer;if(k.collapsed&&CKEDITOR.env.gecko&&CKEDITOR.env.version<10900&&m.type==CKEDITOR.NODE_ELEMENT&&!m.getChildCount())m.appendText('');l.setStart(m.$,k.startOffset);l.setEnd(k.endContainer.$,k.endOffset);i.addRange(l);}n.reset();}},createBookmarks:function(h){var i=[],j=this.getRanges(),k=j.length,l;for(var m=0;m<k;m++){i.push(l=j[m].createBookmark(h,true));h=l.serializable;var n=h?this.document.getById(l.startNode):l.startNode,o=h?this.document.getById(l.endNode):l.endNode;for(var p=m+1;p<k;p++){var q=j[p],r=q.startContainer,s=q.endContainer;r.equals(n.getParent())&&q.startOffset++;r.equals(o.getParent())&&q.startOffset++;s.equals(n.getParent())&&q.endOffset++;s.equals(o.getParent())&&q.endOffset++;}}return i;},createBookmarks2:function(h){var i=[],j=this.getRanges();for(var k=0;k<j.length;k++)i.push(j[k].createBookmark2(h));return i;},selectBookmarks:function(h){var i=[];for(var j=0;j<h.length;j++){var k=new CKEDITOR.dom.range(this.document);k.moveToBookmark(h[j]);i.push(k);}this.selectRanges(i);return this;}};})();CKEDITOR.dom.range.prototype.select=CKEDITOR.env.ie?function(a){var j=this;var b=j.collapsed,c,d,e=j.createBookmark(),f=e.startNode,g;if(!b)g=e.endNode;var h=j.document.$.body.createTextRange();h.moveToElementText(f.$);h.moveStart('character',1);if(g){var i=j.document.$.body.createTextRange();i.moveToElementText(g.$);h.setEndPoint('EndToEnd',i);h.moveEnd('character',-1);}else{c=a||!f.hasPrevious()||f.getPrevious().is&&f.getPrevious().is('br');d=j.document.createElement('span');d.setHtml('&#65279;');d.insertBefore(f);if(c)j.document.createText('﻿').insertBefore(f);}j.setStartBefore(f);f.remove();if(b){if(c){h.moveStart('character',-1);h.select();j.document.$.selection.clear();}else h.select();d.remove();}else{j.setEndBefore(g);g.remove();h.select();}}:function(){var d=this;var a=d.startContainer;if(d.collapsed&&a.type==CKEDITOR.NODE_ELEMENT&&!a.getChildCount())a.append(new CKEDITOR.dom.text(''));var b=d.document.$.createRange();b.setStart(a.$,d.startOffset);try{b.setEnd(d.endContainer.$,d.endOffset);}catch(e){if(e.toString().indexOf('NS_ERROR_ILLEGAL_VALUE')>=0){d.collapse(true);b.setEnd(d.endContainer.$,d.endOffset);}else throw e;}var c=d.document.getSelection().getNative();c.removeAllRanges();c.addRange(b);};



```
