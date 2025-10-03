# skin.js

## Review

## 1. Summary
The snippet is an **immediate‑invoked function expression (IIFE)** that extends CKEditor with a custom skin named **“kama”**.  
It performs the following high‑level tasks:

1. **Pre‑loads assets** for legacy browsers (IE < 7).  
2. **Tweaks the editor width** to account for internal borders.  
3. **Injects and updates UI‑color CSS** dynamically, allowing users to change the overall color scheme of the editor at runtime.  
4. **Handles UI events** (`menuShow` and dialog resize) to apply the UI color to new elements and to keep dialog dimensions consistent across browsers.  

Key components:
- `CKEDITOR.skins.add` – registers the skin with CKEditor.
- `g(j)` – creates a `<style>` tag in a given document.
- `h(j,k,l)` – updates CSS rules in a style element, handling WebKit vs. other browsers.
- `setUiColor` / `getUiColor` – public API added to the editor instance for UI‑color manipulation.
- Event listeners – adjust menu and dialog elements when the skin is active.

The code relies heavily on CKEditor’s own utilities (`CKEDITOR.tools.extend`, `CKEDITOR.document`, etc.) and on browser feature detection (`CKEDITOR.env`).

---

## 2. Detailed Description
### Core Flow
1. **Skin Registration**  
   `CKEDITOR.skins.add('kama', ...)` is called with an IIFE that returns the skin configuration object (`{preload, editor, dialog, templates, margins, init}`).

2. **Browser‑Specific Preload**  
   If the user is on IE < 7, a list of images (`icons.png`, `sprites_ie6.png`, `dialog_sides.gif`) is added to the `preload` array.

3. **Editor Initialisation (`init` method)**  
   - Adjusts `config.width` by subtracting 12 to compensate for the editor’s internal borders.
   - Builds an array `c` of `<style>` elements that will be added to the document when the UI color changes.
   - Prepares a long CSS string (`e`) containing all UI‑color selectors.  
     *The CSS is split into individual rules for WebKit browsers* (due to the way WebKit handles `CSSStyleSheet.rules`).
   - Defines helper functions `g` (create style tag) and `h` (apply new rules).

4. **UI‑Color API Extension**  
   `CKEDITOR.tools.extend(b, { uiColor: null, getUiColor, setUiColor })` adds two public methods to the editor instance `b`:
   - `getUiColor()` – returns the current UI color.
   - `setUiColor(j)` – sets the color, creates the necessary style elements, and updates all previously stored `<style>` tags.  
     The first call to `setUiColor` redefines itself into a faster version that only applies the new color.

5. **Event Listeners**  
   - **`menuShow`**: When a menu is shown, the code ensures the UI‑color style element exists in the menu’s iframe and applies the current color.
   - **`dialog.resize`** (outside the skin definition): Adjusts dialog sizes when the “kama” skin is active, using different CSS properties depending on the browser.

6. **Cleanup**  
   There is no explicit cleanup logic. All style elements remain in the document for the life of the editor instance.

### Dependencies & Assumptions
- **CKEditor global object** – the entire code is built as an extension to CKEditor.
- **Browser detection** via `CKEDITOR.env` (IE, WebKit, Gecko, quirks mode).
- **Legacy support**: Code paths for IE < 7 and IE 8/9 indicate support for very old browsers.
- **`config.uiColor`** – if defined, `setUiColor` is called during initialization.

---

## 3. Functions/Methods
| Name | Purpose | Inputs | Outputs | Side‑effects |
|------|---------|--------|---------|--------------|
| `g(j)` | Creates a `<style>` tag in document `j` and returns it. | `j`: document (`CKEDITOR.document` or an iframe's document). | `<style>` element. | Adds `<style>` to DOM. |
| `h(j, k, l)` | Updates CSS rules of style elements in `j` using rules `k` and replacements `l`. Handles WebKit vs. other browsers. | `j`: array of style elements, `k`: rules (array or string), `l`: array of `[pattern, replacement]`. | none. | Modifies CSS of style elements. |
| `setUiColor(j)` (first call) | Public API to set the UI color; first call constructs helpers and stores style elements. | `j`: new color string. | Returns self‑defining function (fast path). | Creates style tags, updates CSS, sets `uiColor`. |
| `setUiColor(j)` (fast path) | Applies new UI color by updating all stored style elements. | `j`: new color string. | Returns `this`. | Replaces `$color` placeholders in CSS. |
| `getUiColor()` | Returns the currently set UI color. | None. | `uiColor` string. | None. |
| `menuShow` listener | On menu display, ensures UI‑color style exists in the menu’s iframe and applies color. | `j`: event object. | None. | Creates style element in iframe, updates CSS. |
| `dialog.resize` listener | Adjusts dialog size and inner elements for the “kama” skin. | `a`: event object (containing `data` with width/height). | None. | Sets styles on dialog parts, uses `setTimeout` for IE. |

Utility/inline functions:
- Regular expressions `d`, `i` for `$color` placeholder matching.
- Inline string manipulation for CSS generation.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | **CKEditor core** | Provides skin registration, document handling, and utilities. |
| `CKEDITOR.env` | **Browser detection** | Determines IE, WebKit, Gecko, quirks, and version numbers. |
| `CKEDITOR.tools.extend` | **Object mixin** | Adds methods to the editor instance. |
| `CKEDITOR.document` | **DOM abstraction** | Allows cross‑browser document access. |
| `CKEDITOR.dialog` | **Dialog handling** | Provides `on('resize')` event. |
| No external libraries. |  | The code is pure CKEditor/JavaScript. |

---

## 5. Additional Notes
### Strengths
- **Dynamic UI‑color handling**: Allows runtime changes of the editor’s color scheme without reloading.
- **Legacy browser support**: Explicit paths for IE < 7 and IE 8/9 show careful backward compatibility.
- **Self‑defining `setUiColor`**: The first call builds the necessary infrastructure; subsequent calls are faster.

### Potential Issues / Edge Cases
1. **Global variable leakage**: The IIFE creates many variables (`a`, `c`, `e`, etc.) but never declares them with `var/let/const` inside the inner functions (`g`, `h`). While they are declared at the top of the IIFE, their scope is limited to the IIFE – OK, but readability suffers.
2. **Use of `isNaN` on `config.width`**: If `config.width` is defined as a string that cannot be parsed to a number, `isNaN` returns `true`, leaving the width unchanged. It might be clearer to explicitly parse with `parseInt` and check for `NaN`.
3. **Hard‑coded pixel offsets** (e.g., `-12`, `-28`, `-31-14`) rely on the exact layout of the skin. If the skin’s HTML changes, these values may break.
4. **IE Quirks Handling**: The `setTimeout` in the `dialog.resize` listener assumes that the DOM layout will settle after 100 ms. This is brittle and could produce flicker or incorrect sizes on slower machines.
5. **No `use strict`**: Modern JavaScript best practice is to enforce strict mode to avoid accidental global variables.
6. **Deprecated APIs**: Methods like `document.styleSheet` and `removeRule`/`addRule` are legacy IE APIs. In current CKEditor versions, these paths are probably no longer necessary.
7. **No unit tests**: The code is tightly coupled to CKEditor internals, making it hard to test in isolation.

### Future Enhancements
- **Refactor into ES6 modules**: Extract helper functions (`g`, `h`) into separate modules for clarity and easier testing.
- **Use `CSSStyleSheet` methods**: Replace legacy `addRule`/`removeRule` with `insertRule`/`deleteRule` where supported.
- **Parameterize pixel offsets**: Store layout offsets in a configuration object so that skin designers can adjust them without touching JavaScript.
- **Add unit tests**: Wrap the logic in testable functions and use a testing framework (e.g., Jest) to ensure future changes don’t regress UI‑color behaviour.
- **Remove legacy browser support**: If the target user base no longer uses IE < 9, eliminate the heavy browser checks and simplify the code.
- **Expose a public API for skin options**: Allow developers to customize offsets and UI color behaviour via CKEditor config instead of hard‑coding them.

---

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.skins.add('kama',(function(){var a=[];if(CKEDITOR.env.ie&&CKEDITOR.env.version<7)a.push('icons.png','images/sprites_ie6.png','images/dialog_sides.gif');return{preload:a,editor:{css:['editor.css']},dialog:{css:['dialog.css']},templates:{css:['templates.css']},margins:[0,0,0,0],init:function(b){if(b.config.width&&!isNaN(b.config.width))b.config.width-=12;var c=[],d=/\$color/g,e='/* UI Color Support */.cke_skin_kama .cke_menuitem .cke_icon_wrapper{\tbackground-color: $color !important;\tborder-color: $color !important;}.cke_skin_kama .cke_menuitem a:hover .cke_icon_wrapper,.cke_skin_kama .cke_menuitem a:focus .cke_icon_wrapper,.cke_skin_kama .cke_menuitem a:active .cke_icon_wrapper{\tbackground-color: $color !important;\tborder-color: $color !important;}.cke_skin_kama .cke_menuitem a:hover .cke_label,.cke_skin_kama .cke_menuitem a:focus .cke_label,.cke_skin_kama .cke_menuitem a:active .cke_label{\tbackground-color: $color !important;}.cke_skin_kama .cke_menuitem a.cke_disabled:hover .cke_label,.cke_skin_kama .cke_menuitem a.cke_disabled:focus .cke_label,.cke_skin_kama .cke_menuitem a.cke_disabled:active .cke_label{\tbackground-color: transparent !important;}.cke_skin_kama .cke_menuitem a.cke_disabled:hover .cke_icon_wrapper,.cke_skin_kama .cke_menuitem a.cke_disabled:focus .cke_icon_wrapper,.cke_skin_kama .cke_menuitem a.cke_disabled:active .cke_icon_wrapper{\tbackground-color: $color !important;\tborder-color: $color !important;}.cke_skin_kama .cke_menuitem a.cke_disabled .cke_icon_wrapper{\tbackground-color: $color !important;\tborder-color: $color !important;}.cke_skin_kama .cke_menuseparator{\tbackground-color: $color !important;}.cke_skin_kama .cke_menuitem a:hover,.cke_skin_kama .cke_menuitem a:focus,.cke_skin_kama .cke_menuitem a:active{\tbackground-color: $color !important;}';if(CKEDITOR.env.webkit){e=e.split('}').slice(0,-1);for(var f=0;f<e.length;f++)e[f]=e[f].split('{');}function g(j){var k=j.getHead().append('style');k.setAttribute('id','cke_ui_color');k.setAttribute('type','text/css');return k;};function h(j,k,l){var m,n,o;for(var p=0;p<j.length;p++)if(CKEDITOR.env.webkit){for(n=0;n<j[p].$.sheet.rules.length;n++)j[p].$.sheet.removeRule(n);for(n=0;n<k.length;n++){o=k[n][1];for(m=0;m<l.length;m++)o=o.replace(l[m][0],l[m][1]);j[p].$.sheet.addRule(k[n][0],o);}}else{o=k;for(m=0;m<l.length;m++)o=o.replace(l[m][0],l[m][1]);if(CKEDITOR.env.ie)j[p].$.styleSheet.cssText=o;else j[p].setHtml(o);}};var i=/\$color/g;CKEDITOR.tools.extend(b,{uiColor:null,getUiColor:function(){return this.uiColor;
},setUiColor:function(j){var k,l=g(CKEDITOR.document),m='#cke_'+b.name.replace('.','\\.'),n=[m+' .cke_wrapper',m+'_dialog .cke_dialog_contents',m+'_dialog a.cke_dialog_tab',m+'_dialog .cke_dialog_footer'].join(','),o='background-color: $color !important;';if(CKEDITOR.env.webkit)k=[[n,o]];else k=n+'{'+o+'}';return(this.setUiColor=function(p){var q=[[i,p]];b.uiColor=p;h([l],k,q);h(c,e,q);})(j);}});b.on('menuShow',function(j){var k=j.data[0],l=k.element.getElementsByTag('iframe').getItem(0).getFrameDocument();if(!l.getById('cke_ui_color')){var m=g(l);c.push(m);var n=b.getUiColor();if(n)h([m],e,[[i,n]]);}});if(b.config.uiColor)b.setUiColor(b.config.uiColor);}};})());if(CKEDITOR.dialog)CKEDITOR.dialog.on('resize',function(a){var b=a.data,c=b.width,d=b.height,e=b.dialog,f=e.parts.contents,g=!CKEDITOR.env.quirks;if(b.skin!='kama')return;f.setStyles(CKEDITOR.env.ie||CKEDITOR.env.gecko&&CKEDITOR.env.version<10900?{width:c+'px',height:d+'px'}:{'min-width':c+'px','min-height':d+'px'});if(!CKEDITOR.env.ie)return;setTimeout(function(){var h=f.getParent(),i=h.getParent(),j=i.getChild(2);j.setStyle('width',h.$.offsetWidth+'px');j=i.getChild(7);j.setStyle('width',h.$.offsetWidth-28+'px');j=i.getChild(4);j.setStyle('height',h.$.offsetHeight-31-14+'px');j=i.getChild(5);j.setStyle('height',h.$.offsetHeight-31-14+'px');},100);});



```
