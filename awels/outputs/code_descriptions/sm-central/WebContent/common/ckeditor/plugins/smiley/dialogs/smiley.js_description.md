# smiley.js

## Review

## 1. Summary  

The snippet is a **CKEditor 3.x plugin dialog** that displays a grid of smiley images for insertion into the editor.  
* **Purpose** – Allow users to pick a smiley from a pre‑configured set (`config.smiley_images`) and insert it at the current cursor position.  
* **Key components**  
  * `CKEDITOR.dialog.add('smiley', …)` – registers a dialog definition.  
  * `onLoad`, `focus`, `onClick` handlers – manage the dialog’s UI lifecycle.  
  * `h` – a key‑navigation helper that maps arrow/space/tab keys to focus movement.  
  * `g` – the click/space handler that actually creates and inserts the `<img>` element.  
  * The dialog content is built as an HTML table using a mix of string concatenation and the CKEditor `env` flags.  

The code follows CKEditor’s dialog API patterns, but it is heavily minified and relies on a few inline `onkeydown` attributes.

---

## 2. Detailed Description  

### 2.1 Core Flow  

1. **Dialog Registration** – `CKEDITOR.dialog.add('smiley', function(a){ … })`  
   * `a` is the dialog definition object (`CKEDITOR.dialog`).
   * Inside, `b` stores `a.config` and pulls `smiley_images` (array of image names) and `smiley_path` (base URL).
2. **Keyboard helper (`h`)** – Created via `CKEDITOR.tools.addFunction`, returns an integer ID that CKEditor uses to call the function from the inline `onkeydown` attribute.  
   * Handles arrow keys, Space, Tab, and Shift‑Tab for navigating the smiley grid.
3. **Build HTML** – The `i` array is built row‑by‑row, generating `<table>` rows (`<tr>`) and cells (`<td>`).  
   * Each cell contains an `<a>` wrapper with an `<img>` inside.  
   * Inline attributes (`onkeydown`) call the helper via the generated ID (`CKEDITOR.tools.callFunction(h, …)`).
4. **Dialog content** – A single `html` field (`j`) that renders the table.  
   * `onLoad` captures the dialog instance (`f = k.sender`), needed by the click handler.  
   * `focus` sets focus to the first smiley when the dialog opens.  
5. **Insertion (`g`)** – Executed on click or space:  
   * Retrieves the target image element, obtains its `src` (`cke_src`) and `title`.  
   * Creates a new `<img>` with `src`, `title`, and `alt`.  
   * Inserts the element into the editor (`a.insertElement(p)`).  
   * Closes the dialog (`f.hide()`).

### 2.2 Dependencies & Assumptions  

| Item | Type | Comment |
|------|------|---------|
| `CKEDITOR` | Framework | Core CKEditor 3.x API. |
| `CKEDITOR.tools` | Utility | Provides `addFunction`, `htmlEncode`, `callFunction`. |
| `CKEDITOR.env` | Runtime flag | Used for IE quirks handling. |
| `config.smiley_images`, `config.smiley_descriptions`, `config.smiley_path` | User config | Must be provided by the host application. |
| `a.document` | CKEditor DOM | Assumes a valid editor instance. |

The code assumes that `smiley_images` and `smiley_descriptions` arrays are of the same length and that `smiley_path` is a valid base URL. It also assumes the editor is in **wysiwyg mode** (not source mode), as the insertion uses `a.insertElement`.

### 2.3 Architecture & Design Choices  

* **Dialog API** – Uses the official CKEditor dialog mechanism, keeping the plugin lightweight.  
* **String‑based UI** – Rather than building DOM nodes programmatically, the table is constructed as a string. This is typical for CKEditor 3.x but can be fragile for internationalization or XSS concerns.  
* **Inline event handling** – The `onkeydown` attribute calls a global helper via `CKEDITOR.tools.callFunction`. While functional, it mixes declarative and imperative styles.  
* **Keyboard navigation** – A custom navigation system is implemented to provide a fully accessible grid of smileys. This manual approach ensures focus stays inside the dialog, but it could be replaced by a more declarative approach using `tabindex` and native keyboard handling.

---

## 3. Functions / Methods  

| Name | Purpose | Inputs | Outputs | Side‑effects |
|------|---------|--------|---------|--------------|
| `CKEDITOR.dialog.add('smiley', function(a){ … })` | Registers the smiley dialog. | `a` – dialog context object. | Dialog definition object. | Adds dialog to CKEditor. |
| `g` (named inline) | Handles click/space on a smiley. | `k` – event data. | `void` | Inserts `<img>` into editor, hides dialog. |
| `h` (via `CKEDITOR.tools.addFunction`) | Keyboard navigation helper. | `k` – key event; `l` – DOM element. | `void` | Moves focus, triggers `g` on Space. |
| `onLoad` | Captures dialog instance. | `k` – event. | `void` | Stores `f = k.sender`. |
| `focus` | Sets initial focus when dialog opens. | None | `void` | Calls `focus()` on first smiley element. |
| `onClick` | Alias to `g`. | `k` – event. | `void` | Same as `g`. |

Other helper functions are CKEditor built‑ins: `CKEDITOR.tools.addFunction`, `CKEDITOR.tools.callFunction`, `CKEDITOR.tools.htmlEncode`.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party | Core CKEditor 3.x library. |
| `CKEDITOR.tools` | CKEditor API | Provides utility functions. |
| `CKEDITOR.env` | CKEditor API | Runtime flags for browser quirks. |
| `config.smiley_images` / `config.smiley_descriptions` / `config.smiley_path` | User config | Must be set in the editor config. |
| HTML/DOM APIs | Standard | `document.createElement`, `getAttribute`, etc. |

No other external libraries are used. The code is **browser‑agnostic** except for the IE quirks block that adds a temporary width attribute for the smileys.

---

## 5. Additional Notes  

### 5.1 Strengths  

* **Compact** – The entire dialog is defined in a single block, keeping the plugin lightweight.  
* **Accessibility** – Keyboard navigation is explicitly coded, making the dialog usable with arrow keys and tab.  
* **Compatibility** – Handles IE quirks and uses CKEditor’s built‑in environment flags.

### 5.2 Potential Issues & Edge Cases  

1. **String Escaping** – While `CKEDITOR.tools.htmlEncode` is used for the image source, the `title` attribute is inserted directly from `b.smiley_descriptions[e]`. If a description contains quotes or angle brackets it could break the HTML.  
2. **Missing Descriptions** – If `smiley_descriptions` is shorter than `smiley_images`, the code will read `undefined` and insert `alt="undefined"`.  
3. **Inline Event Handlers** – Using `onkeydown` attributes tied to `CKEDITOR.tools.callFunction` can lead to global namespace pollution and is harder to debug. Modern CKEditor 4+ prefers attaching events via `addListener`.  
4. **IE Width Hack** – The `onload` handler that sets a width of `2` and then removes it is a known hack for old IE; it could be omitted for modern browsers or replaced with CSS.  
5. **Performance** – For very large smiley sets, building a huge HTML string can be expensive. A more dynamic approach (generating rows on demand) might be preferable.

### 5.3 Future Enhancements  

| Area | Suggested Improvement |
|------|-----------------------|
| **Localization** | Move all hard‑coded strings (`smiley.title`, etc.) to language files. |
| **Accessibility** | Add `aria-label` to each `<a>` for screen readers. |
| **Dynamic Layout** | Allow `config.smiley_columns` to control columns instead of hard‑coded `d=8`. |
| **Event Binding** | Replace inline `onkeydown` with CKEditor’s `addListener` to avoid `CKEDITOR.tools.callFunction` ID juggling. |
| **Responsive Design** | Use CSS grid instead of a table for better mobile support. |
| **Unit Tests** | Add tests for `g`, `h`, and keyboard navigation to catch regressions. |
| **Cleanup** | Remove unused variables (`e`, `f`, `g` naming could be clearer) and format code for readability. |

---

**Overall Verdict** – The code fulfills its purpose within the CKEditor 3.x ecosystem, but its minified, inline‑style nature makes maintenance and future upgrades cumbersome. Refactoring to a more declarative, modular style would improve readability, accessibility, and testability.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('smiley',function(a){var b=a.config,c=b.smiley_images,d=8,e,f,g=function(k){var l=k.data.getTarget(),m=l.getName();if(m=='td')l=l.getChild([0,0]);else if(m=='a')l=l.getChild(0);else if(m!='img')return;var n=l.getAttribute('cke_src'),o=l.getAttribute('title'),p=a.document.createElement('img',{attributes:{src:n,_cke_saved_src:n,title:o,alt:o}});a.insertElement(p);f.hide();},h=CKEDITOR.tools.addFunction(function(k,l){k=new CKEDITOR.dom.event(k);l=new CKEDITOR.dom.element(l);var m,n,o=k.getKeystroke();switch(o){case 39:if(m=l.getParent().getNext()){n=m.getChild(0);n.focus();}k.preventDefault();break;case 37:if(m=l.getParent().getPrevious()){n=m.getChild(0);n.focus();}k.preventDefault();break;case 38:if(m=l.getParent().getParent().getPrevious()){n=m.getChild([l.getParent().getIndex(),0]);n.focus();}k.preventDefault();break;case 40:if(m=l.getParent().getParent().getNext()){n=m.getChild([l.getParent().getIndex(),0]);if(n)n.focus();}k.preventDefault();break;case 32:g({data:k});k.preventDefault();break;case 9:if(m=l.getParent().getNext()){n=m.getChild(0);n.focus();k.preventDefault(true);}else if(m=l.getParent().getParent().getNext()){n=m.getChild([0,0]);if(n)n.focus();k.preventDefault(true);}break;case CKEDITOR.SHIFT+9:if(m=l.getParent().getPrevious()){n=m.getChild(0);n.focus();k.preventDefault(true);}else if(m=l.getParent().getParent().getPrevious()){n=m.getLast().getChild(0);n.focus();k.preventDefault(true);}break;default:return;}}),i=['<table cellspacing="2" cellpadding="2"',CKEDITOR.env.ie&&CKEDITOR.env.quirks?' style="position:absolute;"':'','><tbody>'];for(e=0;e<c.length;e++){if(e%d===0)i.push('<tr>');i.push('<td class="cke_dark_background cke_hand cke_centered" style="vertical-align: middle;"><a href="javascript:void(0)" class="cke_smile" tabindex="-1" onkeydown="CKEDITOR.tools.callFunction( ',h,', event, this );">','<img class="hand" title="',b.smiley_descriptions[e],'" cke_src="',CKEDITOR.tools.htmlEncode(b.smiley_path+c[e]),'" alt="',b.smiley_descriptions[e],'"',' src="',CKEDITOR.tools.htmlEncode(b.smiley_path+c[e]),'"',CKEDITOR.env.ie?" onload=\"this.setAttribute('width', 2); this.removeAttribute('width');\" ":'','></a>','</td>');if(e%d==d-1)i.push('</tr>');}if(e<d-1){for(;e<d-1;e++)i.push('<td></td>');i.push('</tr>');}i.push('</tbody></table>');var j={type:'html',html:i.join(''),onLoad:function(k){f=k.sender;},focus:function(){var k=this.getElement().getChild([0,0,0,0]);k.focus();},onClick:g,style:'width: 100%; height: 100%; border-collapse: separate;'};
return{title:a.lang.smiley.title,minWidth:270,minHeight:120,contents:[{id:'tab1',label:'',title:'',expand:true,padding:0,elements:[j]}],buttons:[CKEDITOR.dialog.cancelButton]};});



```
