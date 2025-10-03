# find.js

## Review

## 1. Summary  

The script implements the **Find‑and‑Replace dialog** for CKEditor 4.x.  
It is wrapped in an IIFE that receives the CKEditor instance (`h`) and exposes
two dialog definitions (`find` and `replace`).  

Key components  

| Component | Role |
|-----------|------|
| **Utility functions (`a`, `b`, `c`, `d`, `e`, `f`)** | Small helpers for node testing, UI element copying, and dialog page switch handling. |
| **`g` – dialog factory** | Builds the dialog definition for either *find* or *replace* and wires UI events. |
| **`k` – walker wrapper** | Wraps `CKEDITOR.dom.walker` to expose `next`, `back`, and `move` logic with word‑boundary handling. |
| **`l` – range matcher** | Keeps track of the current match’s text nodes, provides DOM‑range conversion, highlighting, and navigation. |
| **`m` / `n` – range helpers** | Create ranges to the start/end of the editor body for search/replace. |
| **`r` – KMP pattern matcher** | Implements a fast, case‑insensitive string matcher with overlap handling. |
| **`s` / `t` – whitespace/word‑boundary check** | Detects characters that break a word for *matchWord* mode. |
| **`u` – search/replace controller** | Orchestrates finding, replacing, and UI state (e.g., cyclic search, snapshot saving). |
| **`v` – search range builder** | Generates a range covering the whole editor body or a selection. |

The plugin uses the **CKEditor dialog framework** and DOM utilities, and relies on the
`CKEDITOR.style` class to highlight matches.

---

## 2. Detailed Description  

### Execution Flow  

1. **Initialization**  
   - The IIFE is executed with the editor instance `h`.  
   - `g` is defined to build a dialog definition.  
   - Two dialog types (`find` and `replace`) are added via `CKEDITOR.dialog.add`.  

2. **Showing the Dialog**  
   - `onShow` creates a new search range (`u.searchRange = v();`).  
   - Depending on the mode, it focuses the appropriate input.  

3. **User Interaction**  
   - Buttons in the dialog call `u.find(...)` or `u.replace(...)`.  
   - These methods rely on the **`l`** (range matcher) and **`r`** (pattern matcher) objects.  
   - The matcher walks the DOM with **`k`**, collects the matching characters into `l._.cursors`, and produces a DOM range that is highlighted via `CKEDITOR.style`.  

4. **Replace All**  
   - When *Replace All* is pressed, a loop repeatedly calls `u.replace` until no more matches are found.  
   - Each replacement triggers a snapshot to enable undo.  

5. **Closing**  
   - `onHide` removes any remaining highlight and restores the selection to the last matched range.

### Assumptions & Constraints  

| Aspect | Assumption | Constraint |
|--------|------------|------------|
| DOM traversal | Only editable content is considered (non‑editable nodes are skipped). | Uses CKEDITOR.dtd to detect block boundaries. |
| Pattern matching | Case sensitivity can be toggled, word boundaries respected, and cyclic search enabled. | Implementation is tightly coupled to CKEditor's `dom.walker`. |
| Highlighting | Uses a style created from `h.config.find_highlight`. | Assumes this style exists and is correctly defined. |
| Undo support | `h.fire('saveSnapshot')` is called before/after changes. | Relies on the editor’s internal undo manager. |
| Performance | KMP algorithm ensures linear time search. | The matcher is re‑used for each character; still O(n+m). |

### Design Choices  

* The code opts for **procedural, imperative style** rather than modern ES6 classes or modules.  
* Heavy use of closures (`h`, `this`) preserves state but can make debugging harder.  
* The pattern matcher (`r`) is a custom KMP implementation rather than using `String.indexOf` or regex; this gives fine‑grained control over word boundaries and cyclic search.  

---

## 3. Functions/Methods  

### Utility helpers  

| Name | Purpose | Inputs | Outputs | Side‑effects |
|------|---------|--------|---------|--------------|
| `a(h)` | Checks if node is a non‑empty text node. | `h` (DOM node) | `boolean` | None |
| `b(h)` | Determines if node is a block boundary (empty or non‑editable). | `h` | `boolean` | None |
| `c()` | Helper for walker navigation – returns the current state (`textNode`, `offset`, `character`, `hitMatchBoundary`). | None | Object | None |
| `d` | Array `['find','replace']` – used in UI copy logic. | None | Array | None |
| `e` | Labels and IDs mapping between find/replace fields. | None | Array | None |
| `f(h)` | Copies values between the two pages when switching tabs. | `h` (`'find'` or `'replace'`) | None | Sets input values |
| `g(h,i)` | Factory that builds the dialog definition for the given mode (`i`). | `h` (editor), `i` (`'find'`/`'replace'`) | Object | Returns dialog config |
| `m(w,x)` | Creates a range from a text node to the end of the body. | `w` (text node), `x` (boolean) | `CKEDITOR.dom.range` | None |
| `n(w)` | Creates a range from the start of the body to a text node. | `w` (text node) | `CKEDITOR.dom.range` | None |
| `s` | Regular expression for word‑boundary characters. | None | RegExp | None |
| `t(w)` | Checks if a character breaks a word. | `w` (string) | `boolean` | None |
| `v(w)` | Builds a range covering the whole editor body or the current selection. | `w` (boolean) | `CKEDITOR.dom.range` | None |

### Core classes  

#### `k` – Walker wrapper  

| Method | Purpose | Inputs | Output | Side‑effects |
|--------|---------|--------|--------|--------------|
| `next()` / `back()` | Move forward/backward one character. | None | Object (state) | Updates internal state |
| `move(w)` | Generic move, `w` true → back, false → forward. | `w` (boolean) | Object | Updates state, handles word boundaries |
| `constructor(h,i)` | Sets up walker with evaluator `a`, guard based on `i`. | `h` (node), `i` (boolean) | None | Initializes `_` |

#### `l` – Range matcher  

| Method | Purpose | Inputs | Output | Side‑effects |
|--------|---------|--------|--------|--------------|
| `toDomRange()` | Convert stored cursors to a `CKEDITOR.dom.range`. | None | `CKEDITOR.dom.range` or `null` | None |
| `updateFromDomRange(w)` | Re‑compute cursors from an existing DOM range. | `w` (range) | None | Updates `cursors`, `rangeLength` |
| `setMatched()` / `clearMatched()` | Flags whether current range matches search. | None | None | Sets `_isMatched` |
| `highlight()` | Applies highlight style to the current match. | None | None | Calls `j.applyToRange`, scrolls into view |
| `removeHighlight()` | Removes highlight style. | None | None | Calls `j.removeFromRange` |
| `moveBack()` / `moveNext()` | Navigate one character backwards/forwards relative to walker. | None | `k` instance | Updates `_cursors` |
| `getEndCharacter()` | Return the last character in current match. | None | `string` | None |
| `getNextCharacterRange(w)` | Create new matcher starting at next character after the current match. | `w` (length) | `l` | None |
| `getCursors()` | Return internal cursor array. | None | Array | None |

#### `r` – KMP pattern matcher  

| Method | Purpose | Inputs | Output | Side‑effects |
|--------|---------|--------|--------|--------------|
| `feedCharacter(w)` | Process one character of the text. | `w` (char) | `0` (continue), `1` (partial), `2` (complete) | Updates internal state |
| `reset()` | Reset state to start. | None | None | Sets `state` to 0 |

#### `u` – Search/replace controller  

| Method | Purpose | Inputs | Output | Side‑effects |
|--------|---------|--------|--------|--------------|
| `find(w,x,y,z,A,B)` | Execute a find operation. | `w` (search string), `x` (case), `y` (matchWord), `z` (cyclic), `A` (highlight), `B` (recursive) | `boolean` (found) | Manipulates `matchRange`, highlights, possibly repeats search |
| `replace(w,x,y,z,A,B,C)` | Execute a replace operation. | `w` (editor), `x` (find), `y` (replace), `z` (matchWord), `A` (matchCase), `B` (cyclic), `C` (skip snapshot) | `boolean` (replaced) | Performs DOM manipulation, snapshot, highlight |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | **Third‑party** | Global namespace provided by the CKEditor core. |
| `CKEDITOR.dom.*` | **Third‑party** | DOM traversal, walker, range, element utilities. |
| `CKEDITOR.style` | **Third‑party** | Used to highlight matches. |
| `CKEDITOR.tools` | **Third‑party** | `extend`, `override` utilities. |
| `CKEDITOR.dialog` | **Third‑party** | Dialog framework for defining UI. |
| `CKEDITOR.DIALOG_RESIZE_NONE` | Constant | Used for dialog sizing. |
| `CKEDITOR.POSITION_AFTER_START`, `CKEDITOR.POSITION_BEFORE_END` | Constants | Range anchor/marker positions. |

No native browser APIs are wrapped; everything operates through CKEditor’s abstraction layers.

---

## 5. Additional Notes  

### Strengths  

* **Performance** – KMP pattern matcher guarantees linear search time even for large documents.  
* **Granular control** – Word‑boundary, case‑sensitivity, and cyclic search are all handled at the character level.  
* **Undo integration** – Snapshots are taken before/after each change.  
* **Re‑usability** – The same matcher (`l`) is reused for both find and replace.

### Potential Issues & Edge Cases  

1. **Memory Leaks** – Objects like `matchRange` and the highlight style are kept in the `u` object. They are cleared on dialog close, but if the dialog is hidden/shown repeatedly, the same objects are reused. A stray reference could survive if the editor instance is destroyed incorrectly.  
2. **Recursive `find`** – The method calls itself via `arguments.callee`, which is deprecated in strict mode and may fail in future environments.  
3. **Case‑sensitivity and Unicode** – The matcher converts characters to lowercase for case‑insensitive search but only uses `toLowerCase()` without locale consideration. It may mishandle accented or non‑ASCII characters.  
4. **Highlight style existence** – `j` is created from `h.config.find_highlight`. If this style is missing or malformed, the dialog will silently fail to highlight.  
5. **Selection handling** – `u.searchRange` uses the current selection only when the editor has a selection; otherwise it covers the whole body. This may lead to unexpected results when the user has no selection and expects to search from the cursor.  
6. **Performance on huge documents** – While KMP is linear, the walker still traverses every text node character by character. For very large HTML documents, this may still be slow, especially when replacing all.  
7. **No support for non‑text nodes** – Images or other inline elements are ignored, which is usually desired, but if the user wants to match text inside a comment node, it is impossible.  

### Suggested Enhancements  

| Area | Suggestion |
|------|------------|
| **Modernization** | Replace the imperative, function‑based approach with ES6 classes and arrow functions for readability. |
| **Avoid `arguments.callee`** | Pass the `recursive` flag explicitly or use a helper that re‑invokes the function. |
| **Unicode awareness** | Use `String.prototype.localeCompare` or `Intl.Collator` for case‑insensitive comparison, or normalise strings via Unicode Normalization Forms. |
| **Highlighting robustness** | Verify that `find_highlight` is a valid style and provide a fallback. |
| **Performance** | Cache the walker for a given search string; skip re‑building the walker on every key stroke. |
| **Unit tests** | Add automated tests covering edge cases: empty string, single‑char patterns, patterns spanning multiple text nodes, and cyclic search. |
| **Accessibility** | Ensure the dialog is fully accessible (ARIA roles, keyboard navigation). |
| **Internationalization** | Add localisation support for the new user interface strings (e.g., *“not found”*, *“replace success”*). |

---  

**Overall**, the code is a solid implementation of a find/replace dialog that respects CKEditor’s architecture. It is functional and reasonably efficient, but it would benefit from modernisation, better error handling, and some minor performance tweaks.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){function a(h){return h.type==CKEDITOR.NODE_TEXT&&h.getLength()>0;};function b(h){var i=CKEDITOR.dtd;return h.isBlockBoundary(CKEDITOR.tools.extend({},i.$empty,i.$nonEditable));};var c=function(){var h=this;return{textNode:h.textNode,offset:h.offset,character:h.textNode?h.textNode.getText().charAt(h.offset):null,hitMatchBoundary:h._.matchBoundary};},d=['find','replace'],e=[['txtFindFind','txtFindReplace'],['txtFindCaseChk','txtReplaceCaseChk'],['txtFindWordChk','txtReplaceWordChk'],['txtFindCyclic','txtReplaceCyclic']];function f(h){var i,j,k,l;i=h==='find'?1:0;j=1-i;var m,n=e.length;for(m=0;m<n;m++){k=this.getContentElement(d[i],e[m][i]);l=this.getContentElement(d[j],e[m][j]);l.setValue(k.getValue());}};var g=function(h,i){var j=new CKEDITOR.style(h.config.find_highlight),k=function(w,x){var y=new CKEDITOR.dom.walker(w);y[x?'guard':'evaluator']=a;y.breakOnFalse=true;this._={matchWord:x,walker:y,matchBoundary:false};};k.prototype={next:function(){return this.move();},back:function(){return this.move(true);},move:function(w){var y=this;var x=y.textNode;if(x===null)return c.call(y);y._.matchBoundary=false;if(x&&w&&y.offset>0){y.offset--;return c.call(y);}else if(x&&y.offset<x.getLength()-1){y.offset++;return c.call(y);}else{x=null;while(!x){x=y._.walker[w?'previous':'next'].call(y._.walker);if(y._.matchWord&&!x||y._.walker._.end)break;if(!x&&b(y._.walker.current))y._.matchBoundary=true;}y.textNode=x;if(x)y.offset=w?x.getLength()-1:0;else y.offset=0;}return c.call(y);}};var l=function(w,x){this._={walker:w,cursors:[],rangeLength:x,highlightRange:null,isMatched:false};};l.prototype={toDomRange:function(){var w=this._.cursors;if(w.length<1)return null;var x=w[0],y=w[w.length-1],z=new CKEDITOR.dom.range(h.document);z.setStart(x.textNode,x.offset);z.setEnd(y.textNode,y.offset+1);return z;},updateFromDomRange:function(w){var z=this;var x,y=new k(w);z._.cursors=[];do{x=y.next();if(x.character)z._.cursors.push(x);}while(x.character)z._.rangeLength=z._.cursors.length;},setMatched:function(){this._.isMatched=true;},clearMatched:function(){this._.isMatched=false;},isMatched:function(){return this._.isMatched;},highlight:function(){var y=this;if(y._.cursors.length<1)return;if(y._.highlightRange)y.removeHighlight();var w=y.toDomRange();j.applyToRange(w);y._.highlightRange=w;var x=w.startContainer;if(x.type!=CKEDITOR.NODE_ELEMENT)x=x.getParent();x.scrollIntoView();y.updateFromDomRange(w);},removeHighlight:function(){var w=this;if(!w._.highlightRange)return;j.removeFromRange(w._.highlightRange);
w.updateFromDomRange(w._.highlightRange);w._.highlightRange=null;},moveBack:function(){var y=this;var w=y._.walker.back(),x=y._.cursors;if(w.hitMatchBoundary)y._.cursors=x=[];x.unshift(w);if(x.length>y._.rangeLength)x.pop();return w;},moveNext:function(){var y=this;var w=y._.walker.next(),x=y._.cursors;if(w.hitMatchBoundary)y._.cursors=x=[];x.push(w);if(x.length>y._.rangeLength)x.shift();return w;},getEndCharacter:function(){var w=this._.cursors;if(w.length<1)return null;return w[w.length-1].character;},getNextCharacterRange:function(w){var x,y=this._.cursors;if(!(x=y[y.length-1]))return null;return new l(new k(m(x)),w);},getCursors:function(){return this._.cursors;}};function m(w,x){var y=new CKEDITOR.dom.range();y.setStart(w.textNode,x?w.offset:w.offset+1);y.setEndAt(h.document.getBody(),CKEDITOR.POSITION_BEFORE_END);return y;};function n(w){var x=new CKEDITOR.dom.range();x.setStartAt(h.document.getBody(),CKEDITOR.POSITION_AFTER_START);x.setEnd(w.textNode,w.offset);return x;};var o=0,p=1,q=2,r=function(w,x){var y=[-1];if(x)w=w.toLowerCase();for(var z=0;z<w.length;z++){y.push(y[z]+1);while(y[z+1]>0&&w.charAt(z)!=w.charAt(y[z+1]-1))y[z+1]=y[y[z+1]-1]+1;}this._={overlap:y,state:0,ignoreCase:!!x,pattern:w};};r.prototype={feedCharacter:function(w){var x=this;if(x._.ignoreCase)w=w.toLowerCase();for(;;)if(w==x._.pattern.charAt(x._.state)){x._.state++;if(x._.state==x._.pattern.length){x._.state=0;return q;}return p;}else if(!x._.state)return o;else x._.state=x._.overlap[x._.state];return null;},reset:function(){this._.state=0;}};var s=/[.,"'?!;: \u0085\u00a0\u1680\u280e\u2028\u2029\u202f\u205f\u3000]/,t=function(w){if(!w)return true;var x=w.charCodeAt(0);return x>=9&&x<=13||x>=8192&&x<=8202||s.test(w);},u={searchRange:null,matchRange:null,find:function(w,x,y,z,A,B){var K=this;if(!K.matchRange)K.matchRange=new l(new k(K.searchRange),w.length);else{K.matchRange.removeHighlight();K.matchRange=K.matchRange.getNextCharacterRange(w.length);}var C=new r(w,!x),D=o,E='%';while(E!==null){K.matchRange.moveNext();while(E=K.matchRange.getEndCharacter()){D=C.feedCharacter(E);if(D==q)break;if(K.matchRange.moveNext().hitMatchBoundary)C.reset();}if(D==q){if(y){var F=K.matchRange.getCursors(),G=F[F.length-1],H=F[0],I=new k(n(H),true),J=new k(m(G),true);if(!(t(I.back().character)&&t(J.next().character)))continue;}K.matchRange.setMatched();if(A!==false)K.matchRange.highlight();return true;}}K.matchRange.clearMatched();K.matchRange.removeHighlight();if(z&&!B){K.searchRange=v(true);K.matchRange=null;
return arguments.callee.apply(K,Array.prototype.slice.call(arguments).concat([true]));}return false;},replaceCounter:0,replace:function(w,x,y,z,A,B,C){var H=this;var D=false;if(H.matchRange&&H.matchRange.isMatched()&&!H.matchRange._.isReplaced){H.matchRange.removeHighlight();var E=H.matchRange.toDomRange(),F=h.document.createText(y);if(!C){var G=h.getSelection();G.selectRanges([E]);h.fire('saveSnapshot');}E.deleteContents();E.insertNode(F);if(!C){G.selectRanges([E]);h.fire('saveSnapshot');}H.matchRange.updateFromDomRange(E);if(!C)H.matchRange.highlight();H.matchRange._.isReplaced=true;H.replaceCounter++;D=true;}else D=H.find(x,z,A,B,!C);return D;}};function v(w){var x,y=h.getSelection(),z=h.document.getBody();if(y&&!w){x=y.getRanges()[0].clone();x.collapse(true);}else{x=new CKEDITOR.dom.range();x.setStartAt(z,CKEDITOR.POSITION_AFTER_START);}x.setEndAt(z,CKEDITOR.POSITION_BEFORE_END);return x;};return{title:h.lang.findAndReplace.title,resizable:CKEDITOR.DIALOG_RESIZE_NONE,minWidth:350,minHeight:165,buttons:[CKEDITOR.dialog.cancelButton],contents:[{id:'find',label:h.lang.findAndReplace.find,title:h.lang.findAndReplace.find,accessKey:'',elements:[{type:'hbox',widths:['230px','90px'],children:[{type:'text',id:'txtFindFind',label:h.lang.findAndReplace.findWhat,isChanged:false,labelLayout:'horizontal',accessKey:'F'},{type:'button',align:'left',style:'width:100%',label:h.lang.findAndReplace.find,onClick:function(){var w=this.getDialog();if(!u.find(w.getValueOf('find','txtFindFind'),w.getValueOf('find','txtFindCaseChk'),w.getValueOf('find','txtFindWordChk'),w.getValueOf('find','txtFindCyclic')))alert(h.lang.findAndReplace.notFoundMsg);}}]},{type:'vbox',padding:0,children:[{type:'checkbox',id:'txtFindCaseChk',isChanged:false,style:'margin-top:28px',label:h.lang.findAndReplace.matchCase},{type:'checkbox',id:'txtFindWordChk',isChanged:false,label:h.lang.findAndReplace.matchWord},{type:'checkbox',id:'txtFindCyclic',isChanged:false,'default':true,label:h.lang.findAndReplace.matchCyclic}]}]},{id:'replace',label:h.lang.findAndReplace.replace,accessKey:'M',elements:[{type:'hbox',widths:['230px','90px'],children:[{type:'text',id:'txtFindReplace',label:h.lang.findAndReplace.findWhat,isChanged:false,labelLayout:'horizontal',accessKey:'F'},{type:'button',align:'left',style:'width:100%',label:h.lang.findAndReplace.replace,onClick:function(){var w=this.getDialog();if(!u.replace(w,w.getValueOf('replace','txtFindReplace'),w.getValueOf('replace','txtReplace'),w.getValueOf('replace','txtReplaceCaseChk'),w.getValueOf('replace','txtReplaceWordChk'),w.getValueOf('replace','txtReplaceCyclic')))alert(h.lang.findAndReplace.notFoundMsg);
}}]},{type:'hbox',widths:['230px','90px'],children:[{type:'text',id:'txtReplace',label:h.lang.findAndReplace.replaceWith,isChanged:false,labelLayout:'horizontal',accessKey:'R'},{type:'button',align:'left',style:'width:100%',label:h.lang.findAndReplace.replaceAll,isChanged:false,onClick:function(){var w=this.getDialog(),x;u.replaceCounter=0;u.searchRange=v(true);if(u.matchRange){u.matchRange.removeHighlight();u.matchRange=null;}h.fire('saveSnapshot');while(u.replace(w,w.getValueOf('replace','txtFindReplace'),w.getValueOf('replace','txtReplace'),w.getValueOf('replace','txtReplaceCaseChk'),w.getValueOf('replace','txtReplaceWordChk'),false,true)){}if(u.replaceCounter){alert(h.lang.findAndReplace.replaceSuccessMsg.replace(/%1/,u.replaceCounter));h.fire('saveSnapshot');}else alert(h.lang.findAndReplace.notFoundMsg);}}]},{type:'vbox',padding:0,children:[{type:'checkbox',id:'txtReplaceCaseChk',isChanged:false,label:h.lang.findAndReplace.matchCase},{type:'checkbox',id:'txtReplaceWordChk',isChanged:false,label:h.lang.findAndReplace.matchWord},{type:'checkbox',id:'txtReplaceCyclic',isChanged:false,'default':true,label:h.lang.findAndReplace.matchCyclic}]}]}],onLoad:function(){var w=this,x,y,z=false;this.on('hide',function(){z=false;});this.on('show',function(){z=true;});this.selectPage=CKEDITOR.tools.override(this.selectPage,function(A){return function(B){A.call(w,B);var C=w._.tabs[B],D,E,F;E=B==='find'?'txtFindFind':'txtFindReplace';F=B==='find'?'txtFindWordChk':'txtReplaceWordChk';x=w.getContentElement(B,E);y=w.getContentElement(B,F);if(!C.initialized){D=CKEDITOR.document.getById(x._.inputId);C.initialized=true;}if(z)f.call(this,B);};});},onShow:function(){u.searchRange=v();if(i=='replace')this.getContentElement('replace','txtFindReplace').focus();else this.getContentElement('find','txtFindFind').focus();},onHide:function(){if(u.matchRange&&u.matchRange.isMatched()){u.matchRange.removeHighlight();h.focus();h.getSelection().selectRanges([u.matchRange.toDomRange()]);}delete u.matchRange;}};};CKEDITOR.dialog.add('find',function(h){return g(h,'find');});CKEDITOR.dialog.add('replace',function(h){return g(h,'replace');});})();



```
