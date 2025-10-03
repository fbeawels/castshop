# jquery.meiomask.js

## Review

## 1. Summary  

**Purpose**  
The code implements a jQuery‑based input‑masking plugin. It allows developers to define fixed or dynamic character masks (e.g., phone numbers, dates, CPF/CNPJ, numeric values) that are applied to `<input>` fields at runtime. The mask is enforced on key events (`keydown`, `keypress`, `keyup`) and paste events, with support for different browsers (IE, Safari, Firefox, Opera, iOS).

**Key components**

| Component | Role |
|-----------|------|
| `mask` object | Holds all configuration, masks, and the core logic. |
| `rules` | Regular expressions that validate each placeholder character (`z`, `Z`, `a`, `*`, `@`). |
| `masks` | Pre‑defined mask templates (e.g., phone, cpf, date). |
| `set` / `unset` | Public API to attach/detach a mask from a jQuery selection. |
| Event handlers (`_onMask`, `_keyPress`, `_paste`, …) | Handle user input and enforce the mask. |
| Utility helpers (`__maskArray`, `__applyMask`, `__setRange`, …) | Perform string manipulation and cursor positioning. |
| jQuery prototype extensions (`setMask`, `unsetMask`) | Convenient shortcuts for the API. |

**Design patterns & libraries**

* **Module pattern / IIFE** – isolates internal state, exposes only the `mask` object.  
* **jQuery plugin pattern** – extends `jQuery.fn` for easy usage.  
* **Strategy‑like pattern** – mask rules and predefined masks act as interchangeable strategies.  
* No other external libraries are used beyond jQuery (assumed available globally).

---

## 2. Detailed Description  

### Initialization

1. `init()`  
   * Runs once (`hasInit` guard).  
   * Builds regex for fixed characters and numeric ranges (`rules[0]…rules[9]`).  
   * Sets `ignore` flag and other flags used by event handlers.

### Public API  

#### `set(selector, optionsOrMask)`  

* **Parameters**  
  * `selector` – jQuery object or CSS selector.  
  * `optionsOrMask` – string (mask name or custom mask) or object with options.  

* **Process**  
  1. Merge user options with defaults (`options`).  
  2. Resolve mask string from `optionsOrMask`, metadata, or predefined masks.  
  3. Create mask definition (`maskArray`, `maskNonFixedCharsArray`, `defaultValue`, `maxlength`).  
  4. Apply mask to the current value of each element (or default value).  
  5. Attach event listeners (`keydown`, `keyup`, `keypress`, paste/​input).  

#### `unset(selector)`  

* Removes all data (`mask`) and unbinds the event handlers from each element.  

#### `string(value, maskOrOptions)`  

* Utility that returns the masked string for a given value.  
* Accepts either a mask name or an options object.  

### Execution Flow

1. **User types** → `_keyPress` is fired.  
2. `_keyPress` checks whether the key should be ignored (browser, key code).  
3. It builds the current value, applies mask rules via `__maskArray`.  
4. The masked value is written back to the input.  
5. Cursor position is updated by `__setRange`.  
6. On paste, `_paste` is called after a tiny delay (`_delayedOnMask`).  
7. `onValid`, `onInvalid`, `onOverflow` callbacks can be supplied by the user.

### Assumptions & Constraints

* **jQuery** must be loaded before the plugin.  
* Works in older browsers (IE6–11, Safari 4+, Firefox 3+, Opera 11+) but some features (e.g., `maxlength` handling) are browser‑specific.  
* Assumes the input element is a single line text field; not tested on `<textarea>`.  
* Uses `thisObj` binding to preserve the plugin context; relies on jQuery’s `.bind()` event data feature.

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `init()` | One‑time setup of regexes & internal flags. | None | Sets `hasInit`, builds `rules` and `fixedCharsReg` |
| `set(G, D)` | Attach mask to elements. | `G` – jQuery selector; `D` – mask string or options object | Returns jQuery set; modifies DOM (adds data, attributes, event listeners) |
| `unset(D)` | Remove mask from elements. | `D` – jQuery selector | Returns jQuery set; removes data, attributes, listeners |
| `string(F, D)` | Returns masked string for a value. | `F` – raw value; `D` – mask or options | Masked string |
| `_onMask(C)` | Central event dispatcher. | Event object `C` | Calls handler defined in `C.data.func` |
| `_delayedOnMask(C)` | Wrapper that delays paste handling. | Event object | Calls `_onMask` after 1 ms |
| `_keyDown(C, D)` | Handles `keydown`; sets `ignore` flag. | Event object | Returns true/false |
| `_keyUp(C, D)` | Handles `keyup`; triggers paste logic. | Event object | Returns result of `_paste` |
| `_paste(D, E)` | Applies mask after paste. | Event object, data object | Sets input value, cursor position |
| `_keyPress(H, I)` | Handles `keypress`; applies mask logic. | Event object, data object | Returns true/false |
| `_keyPressFixed(C, D)` | Cursor handling for fixed masks. | Event object, data object | Sets range |
| `_keyPressReverse(C, D)` | Stub for reverse masks (currently no-op). | Event object, data object | Always `false` |
| `_setMaskData(F, C, E)` | Helper to update mask data on an element. | Element, key, value | Modifies `data("mask")` |
| `_changeSignal(D, E)` | Detects signal characters (`+`, `-`). | Event type, data object | Updates mask data `signal` |
| `_insertSignal(D, G, F, C)` | Pre‑processes default values for signed masks. | Reverse flag, value string, options, signals | Modifies `options.signal` & `defaultValue` |
| `__getPasteEvent()` | Returns event name for paste input (browser‑dependent). | None | `"paste"` or `"input"` |
| `__getKeyNumber(C)` | Normalises key code across browsers. | Event | Number |
| `__maskArray(H, G, E, D, C, I, F)` | Core algorithm: filters, defaults, applies mask, returns array. | Value array, non‑fixed mask array, full mask array, reverse flag, default value, signal, extra positions | Array of characters |
| `__applyDefaultValue(E)` | Adds default value to masked array. | Array | Modified array |
| `__removeInvalidChars(E, D)` | Removes characters that do not match mask rules. | Array, mask array | Filtered array |
| `__applyMask(E, C, F)` | Inserts fixed mask characters into value array. | Value array, mask array, offset | Modified array |
| `__extraPositionsTill(E, C)` | Counts fixed characters ahead of a position. | Position, mask array | Count |
| `__setRange(E, F, C)` | Sets cursor range on input. | DOM element, start, end | Side‑effect: selection change |
| `__getRangePosition(F)` | Reads current cursor position. | DOM element | `{start, end}` |

**Reusable utilities**: `__maskArray`, `__applyMask`, `__removeInvalidChars`, `__setRange`, `__getRangePosition` could be exported for unit testing.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery** | Third‑party | The plugin expects a global `jQuery` variable (passed as `B`). No version is enforced but the code uses features from jQuery 1.3+. |
| **Browser APIs** | Standard | Uses `setSelectionRange`, `createTextRange`, `document.selection`, and `navigator.userAgent` (indirectly via `window.orientation`). |
| **CSS/HTML** | Platform | Works on any browser that supports the DOM input element. |

No external APIs or services are used.

---

## 5. Additional Notes & Recommendations  

### Strengths  

* **Extensibility** – Users can supply custom masks or use the predefined ones.  
* **Browser compatibility** – The code handles quirks in IE, Safari, Firefox, Opera, and iOS.  
* **Callback hooks** – `onValid`, `onInvalid`, `onOverflow` give developers fine control over UX.  
* **Non‑intrusive** – Works only on elements that the user explicitly selects.  

### Weaknesses / Edge Cases  

| Area | Issue | Impact | Fix / Improvement |
|------|-------|--------|-------------------|
| **Variable naming** | Obscure single‑letter variables (`A`, `B`, `C`, `D`, …). | Hard to read, maintain, and debug. | Rename to descriptive identifiers; use `const`/`let`. |
| **Global flag `A`** | Detects `window.orientation` to adjust key handling for iOS. | May break on devices that expose `orientation` differently. | Detect iOS via `navigator.userAgent` or feature detection. |
| **Hard‑coded key codes** | `ignoreKeys` and `iphoneIgnoreKeys` rely on numeric key codes. | Modern browsers use `KeyboardEvent.key` or `code`. | Use `key`/`code` and fall back to keyCode for legacy browsers. |
| **Synchronous paste handling** | `_delayedOnMask` uses `setTimeout(..., 1)` to postpone paste logic. | Still potentially race‑y in some browsers. | Use `paste` event directly with `event.clipboardData`. |
| **Fixed `maxlength` handling** | Special case for IE and Safari. | May not work on newer browsers or with `input type="number"`. | Use the standard `maxlength` attribute and let the browser enforce it. |
| **No support for `<textarea>`** | Masking is only applied to single‑line inputs. | Not useful for multi‑line data. | Extend the plugin to detect and handle `<textarea>`. |
| **`_keyPressReverse`** is a no‑op stub. | Reverse masks may not correctly position the cursor. | Users may experience cursor jumps. | Implement proper cursor logic for reverse masks. |
| **No unit tests** | The algorithm is complex. | Hard to guarantee correctness after changes. | Add Jest / Karma tests covering all mask types. |
| **Error handling** | Many `try`/`catch` blocks are missing. | Exceptions may silently break the mask. | Add defensive checks and throw meaningful errors. |
| **Documentation** | Minimal inline comments. | New developers will struggle. | Add JSDoc comments and a README with usage examples. |

### Future Enhancements  

1. **ES6+ Modernisation** – Convert to a module (ES6 module or UMD) with proper export, use `class` or factory functions.  
2. **Custom placeholder support** – Allow developers to define custom placeholder characters.  
3. **Internationalisation** – Add localisation for error messages.  
4. **Performance** – Debounce input handling for large masks.  
5. **Accessibility** – Ensure that screen readers see the unmasked value.  
6. **Testing** – Build a comprehensive test suite covering edge cases (copy‑paste, delete, backspace, selection, cursor movement).  

---  

**Verdict**  
The plugin is a solid, battle‑tested solution for masking user input across a wide range of browsers. However, the code would benefit from modern refactoring, clearer naming, better event handling, and expanded test coverage. Implementing the suggestions above would make the library easier to maintain, extend, and integrate into contemporary projects.

## Code Critique



## Code Preview

```javascript
/**
 * @version 1.0.3
 * The MIT License
 * Copyright (c) 2008 Fabio M. Costa http://www.meiocodigo.com
 */
(function(B){var A=(window.orientation!=undefined);B.extend({mask:{rules:{"z":/[a-z]/,"Z":/[A-Z]/,"a":/[a-zA-Z]/,"*":/[0-9a-zA-Z]/,"@":/[0-9a-zA-ZçÇáàãéèíìóòõúùü]/},fixedChars:"[(),.:/ -]",ignoreKeys:[8,9,13,16,17,18,33,34,35,36,37,38,39,40,45,46,91,116],iphoneIgnoreKeys:[10,127],signals:["+","-"],options:{attr:"alt",mask:null,type:"fixed",defaultValue:"",signal:false,onInvalid:function(){},onValid:function(){},onOverflow:function(){}},masks:{"phone":{mask:"(99) 9999-9999"},"phone-us":{mask:"(999) 9999-9999"},"cpf":{mask:"999.999.999-99"},"cnpj":{mask:"99.999.999/9999-99"},"date":{mask:"39/19/9999"},"date-us":{mask:"19/39/9999"},"cep":{mask:"99999-999"},"time":{mask:"29:69"},"cc":{mask:"9999 9999 9999 9999"},"integer":{mask:"999.999.999.999",type:"reverse"},"decimal":{mask:"99,999.999.999.999",type:"reverse",defaultValue:"000"},"decimal-us":{mask:"99.999,999,999,999",type:"reverse",defaultValue:"000"},"signed-decimal":{mask:"99,999.999.999.999",type:"reverse",defaultValue:"+000"},"signed-decimal-us":{mask:"99,999.999.999.999",type:"reverse",defaultValue:"+000"}},MAXLENGTH:{ie:2147483647},init:function(){if(!this.hasInit){var C;this.ignore=false;this.fixedCharsReg=new RegExp(this.fixedChars);this.fixedCharsRegG=new RegExp(this.fixedChars,"g");for(C=0;C<=9;C++){this.rules[C]=new RegExp("[0-"+C+"]")}this.hasInit=true}},set:function(G,D){var C=this,E=B(G),F="maxlength";this.init();return E.each(function(){var N=B(this),O=B.extend({},C.options),M=N.attr(O.attr),H="",J=C.__getPasteEvent();H=(typeof D=="string")?D:(M!="")?M:null;if(H){O.mask=H}if(C.masks[H]){O=B.extend(O,C.masks[H])}if(typeof D=="object"){O=B.extend(O,D)}if(B.metadata){O=B.extend(O,N.metadata())}if(O.mask!=null){if(N.data("mask")){C.unset(N)}var I=O.defaultValue,L=N.attr(F),K=(O.type=="reverse");O=B.extend({},O,{maxlength:L,maskArray:O.mask.split(""),maskNonFixedCharsArray:O.mask.replace(C.fixedCharsRegG,"").split(""),defaultValue:I.split("")});if(K){N.css("text-align","right")}if(N.val()!=""){N.val(C.string(N.val(),O))}else{if(I!=""){N.val(C.string(I,O))}}N.data("mask",O);switch(true){case (B.browser.msie):N.attr(F,C.MAXLENGTH.ie);break;case (B.browser.safari):N.removeAttr(F);break;default:if(L>-1){N.removeAttr(F)}break}N.bind("keydown",{func:C._keyDown,thisObj:C},C._onMask).bind("keyup",{func:C._keyUp,thisObj:C},C._onMask).bind("keypress",{func:C._keyPress,thisObj:C},C._onMask).bind(J,{func:C._paste,thisObj:C},C._delayedOnMask)}})},unset:function(D){var C=B(D),E=this;return C.each(function(){var H=B(this);if(H.data("mask")){var F=H.data("mask").maxlength,G=E.__getPasteEvent();if(F==-1){H.removeAttr("maxlength")}else{H.attr("maxlength",F)}H.unbind("keydown",E._onMask).unbind("keypress",E._onMask).unbind("keyup",E._onMask).unbind(G,E._delayedOnMask).removeData("mask")}})},string:function(F,D){this.init();var E={};if(typeof F!="string"){F=String(F)}switch(typeof D){case"string":if(this.masks[D]){E=B.extend(E,this.masks[D])}else{E.mask=D}break;case"object":E=D;break}var C=(E.type=="reverse");this._insertSignal(C,F,E,this.signals);return this.__maskArray(F.split(""),E.mask.replace(this.fixedCharsRegG,"").split(""),E.mask.split(""),C,E.defaultValue,E.signal)},_onMask:function(C){var E=C.data.thisObj,D={};D._this=C.target;D.$this=B(D._this);if(D.$this.attr("readonly")){return true}D.value=D.$this.val();D.nKey=E.__getKeyNumber(C);D.range=E.__getRangePosition(D._this);D.valueArray=D.value.split("");D.data=D.$this.data("mask");D.reverse=(D.data.type=="reverse");return C.data.func.call(E,C,D)},_delayedOnMask:function(C){C.type="paste";setTimeout(function(){C.data.thisObj._onMask(C)},1)},_keyDown:function(C,D){if(A){this.ignore=(B.inArray(D.nKey,this.iphoneIgnoreKeys)>-1);return this._keyPress(C,D)}else{this.ignore=(B.inArray(D.nKey,this.ignoreKeys)>-1);return true}},_keyUp:function(C,D){return this._paste(C,D)},_paste:function(D,E){this._changeSignal(D.type,E);var C=this.__maskArray(E.valueArray,E.data.maskNonFixedCharsArray,E.data.maskArray,E.reverse,E.data.defaultValue,E.data.signal);E.$this.val(C);if(!E.reverse){this.__setRange(E._this,E.range.start,E.range.end)}return true},_keyPress:function(H,I){if(this.ignore||H.ctrlKey||H.metaKey||H.altKey){I.data.onValid.call(I._this,"",I.nKey);return true}this._changeSignal(H.type,I);var J=String.fromCharCode(I.nKey);rangeStart=I.range.start,rawValue=I.value,maskArray=I.data.maskArray;if(I.reverse){var F=rawValue.substr(0,rangeStart),C=rawValue.substr(I.range.end,rawValue.length);rawValue=(F+J+C);if(I.data.signal){rangeStart-=I.data.signal.length}}var E=rawValue.replace(this.fixedCharsRegG,"").split(""),D=this.__extraPositionsTill(rangeStart,maskArray);I.rsEp=rangeStart+D;if(!this.rules[maskArray[I.rsEp]]){I.data.onOverflow.call(I._this,J,I.nKey);return false}else{if(!this.rules[maskArray[I.rsEp]].test(J)){I.data.onInvalid.call(I._this,J,I.nKey);return false}else{I.data.onValid.call(I._this,J,I.nKey)}}var G=this.__maskArray(E,I.data.maskNonFixedCharsArray,maskArray,I.reverse,I.data.defaultValue,I.data.signal,D);I.$this.val(G);if(I.reverse){return this._keyPressReverse(H,I)}else{return this._keyPressFixed(H,I)}},_keyPressFixed:function(C,D){if(D.rangeStart==D.range.end||D.data.defaultValue){if((D.rsEp==0&&D.value.length==0)||D.rsEp<D.value.length){this.__setRange(D._this,D.rsEp,D.rsEp+1)}}else{this.__setRange(D._this,D.rangeStart,D.range.end)}return true},_keyPressReverse:function(C,D){return false},_setMaskData:function(F,C,E){var D=F.data("mask");D[C]=E;F.data("mask",D)},_changeSignal:function(D,E){if(E.data.signal!==false){var C=(D=="paste")?E.value.substr(0,1):String.fromCharCode(E.nKey);if(B.inArray(C,this.signals)>-1){if(C=="+"){C=""}this._setMaskData(E.$this,"signal",C);E.data.signal=C}}},_insertSignal:function(D,G,F,C){if(D&&F.defaultValue){if(typeof F.defaultValue=="string"){F.defaultValue=F.defaultValue.split("")}if(B.inArray(F.defaultValue[0],C)>-1){var E=G.substr(0,1);F.signal=(B.inArray(E,C)>-1)?E:F.defaultValue[0];F.defaultValue.shift()}}},__getPasteEvent:function(){return(B.browser.opera||(B.browser.mozilla&&parseFloat(B.browser.version.substr(0,3))<1.9))?"input":"paste"},__getKeyNumber:function(C){return(C.charCode||C.keyCode||C.which)},__maskArray:function(H,G,E,D,C,I,F){if(D){H.reverse()}H=this.__removeInvalidChars(H,G);if(C){H=this.__applyDefaultValue.call(H,C)}H=this.__applyMask(H,E,F);if(D){H.reverse();if(!I||I=="+"){I=""}return I+H.join("").substring(H.length-E.length)}else{return H.join("").substring(0,E.length)}},__applyDefaultValue:function(E){var C=E.length,D=this.length,F;for(F=D-1;F>=0;F--){if(this[F]==E[0]){this.pop()}else{break}}for(F=0;F<C;F++){if(!this[F]){this[F]=E[F]}}return this},__removeInvalidChars:function(E,D){for(var C=0;C<E.length;C++){if(D[C]&&this.rules[D[C]]&&!this.rules[D[C]].test(E[C])){E.splice(C,1);C--}}return E},__applyMask:function(E,C,F){if(typeof F=="undefined"){F=0}for(var D=0;D<E.length+F;D++){if(C[D]&&this.fixedCharsReg.test(C[D])){E.splice(D,0,C[D])}}return E},__extraPositionsTill:function(E,C){var D=0;while(this.fixedCharsReg.test(C[E])){E++;D++}return D},__setRange:function(E,F,C){if(typeof C=="undefined"){C=F}if(E.setSelectionRange){E.setSelectionRange(F,C)}else{var D=E.createTextRange();D.collapse();D.moveStart("character",F);D.moveEnd("character",C-F);D.select()}},__getRangePosition:function(F){var C={start:0,end:0};if(F.setSelectionRange){C.start=F.selectionStart;C.end=F.selectionEnd}else{if(document.selection&&document.selection.createRange){var E=document.selection.createRange();var D=E.duplicate();C.start=0-D.moveStart("character",-100000);C.end=C.start+E.text.length}}return C}}});B.fn.extend({setMask:function(C){return B.mask.set(this,C)},unsetMask:function(){return B.mask.unset(this)}})})(jQuery)


```
