# jquery.meiomask.js

## Review

## 1. Summary

**Purpose**  
This is a lightweight jQuery plug‑in that enforces input masks on form fields (e.g., phone numbers, dates, CPF/CNPJ, credit cards, decimals). It parses a mask definition, handles user input in real‑time, and keeps the value in sync with the mask, providing callbacks for validity, overflow and custom “signals” (+ / -).

**Key Components**

| Component | Role |
|-----------|------|
| **`mask.rules`** | Regular expressions for the five placeholder tokens (`z`, `Z`, `a`, `*`, `@`) plus numeric ranges (`0-9`). |
| **`mask.fixedChars`** | Regex that identifies characters that are part of the mask literal (e.g. `(`, `)`, `.`, `-`). |
| **`mask.masks`** | Built‑in mask definitions for common data types (`phone`, `cpf`, `date`, etc.). |
| **`mask.set / unset`** | Public API used by `$.fn.setMask()` and `$.fn.unsetMask()`. |
| **`mask.string`** | Utility that turns a raw string value into a masked string using the rules above. |
| **Event Handlers** (`_keyDown`, `_keyUp`, `_keyPress`, `_paste`) | Intercept keyboard & paste events to maintain the mask in real‑time. |
| **Helper Methods** (`__maskArray`, `__applyMask`, `__applyDefaultValue`, …) | Core algorithm that removes invalid chars, applies the mask, handles reverse mode and “signal” symbols. |

**Design Patterns / Libraries**

* jQuery Plugin pattern – `$.fn.extend` to add `setMask`/`unsetMask` to jQuery prototypes.  
* Event delegation with data binding – each input keeps its mask configuration in `$.data("mask")`.  
* Simple state machine – the mask is represented as two arrays (fixed mask and non‑fixed mask) and the algorithm steps through them.

---

## 2. Detailed Description

### Initialization Flow

1. **`$.mask.init`**  
   *Creates internal regexes and expands numeric rules (`0-9`) for quick access.*  
   *Executed lazily – the first call to any public method triggers it.*

2. **`$.mask.set`**  
   *Accepts either a mask string, a mask key (`"phone"`, `"date"`…) or a configuration object.*  
   *Extends defaults with user options, pulls `alt` or `metadata` if available.*  
   *Configures `maxlength` based on the mask (except for IE).  
   *Binds keydown/keyup/keypress/paste events to the element, attaching the private handlers.*

3. **`$.mask.unset`**  
   *Removes all event bindings, clears `maxlength`, and deletes the `mask` data.*

### Runtime Behavior

* Every key press (or paste) triggers `_onMask`.  
* `_onMask` delegates to the proper handler based on the event type (`_keyDown`, `_keyUp`, `_keyPress`, `_paste`).  
* The core masking logic lives in `__maskArray` and the helpers:
  * **`__removeInvalidChars`** – strips any character that does not match the mask’s rule.  
  * **`__applyMask`** – injects literal characters from the mask into the value array.  
  * **`__applyDefaultValue`** – pre‑fills the field when a default value is supplied.  
  * **`__insertSignal`** – handles “signal” characters (`+`/`-`) that may appear at the start of a reversed mask.  
* After transforming the value, the plugin sets the element’s value and adjusts the cursor with `__setRange`.

### Cleanup

* `unset` restores the original `maxlength`, removes all event listeners, and deletes the stored mask data.

---

## 3. Functions/Methods

| Method | Purpose | Input | Output | Side Effects |
|--------|---------|-------|--------|--------------|
| **`init`** | Lazily prepares regexes and numeric rules. | none | none | Sets `hasInit`, `rules`, `fixedCharsReg`, etc. |
| **`set`** | Attach mask to one or more elements. | jQuery element + mask string/object | jQuery set | Binds events, sets `maxlength`, stores mask data. |
| **`unset`** | Detach mask. | jQuery element | jQuery set | Unbinds events, removes data, restores `maxlength`. |
| **`string`** | Return masked string from raw value. | value + mask definition | masked string | None |
| **`_onMask`** | Central dispatcher for key/paste events. | event | boolean (event handling) | Calls specific handler. |
| **`_delayedOnMask`** | Invokes `_onMask` for paste events (timeout). | event | boolean | Delays by 1 ms. |
| **`_keyDown`** | Handles keydown; marks ignored keys (iPhone/others). | event | boolean | Sets `ignore`. |
| **`_keyUp`** | Delegates to `_paste`. | event | boolean | |
| **`_paste`** | Handles paste event; updates mask. | event | boolean | Sets element value. |
| **`_keyPress`** | Handles keypress; main transformation logic. | event | boolean | Updates value, calls callbacks. |
| **`_keyPressFixed` / `_keyPressReverse`** | Adjust cursor after update. | event | boolean | Calls `__setRange`. |
| **`_setMaskData`** | Update a field of the stored mask data. | element, key, value | none | Mutates `$.data("mask")`. |
| **`_changeSignal`** | Detects signal symbols (`+`/`-`). | event | none | Updates mask data’s `signal`. |
| **`_insertSignal`** | Pre‑sets signal when default value starts with a signal. | reverse, raw, maskDef, signals | none | Sets `maskDef.signal`. |
| **`__getPasteEvent`** | Browser‑specific event name for paste. | none | `"input"` or `"paste"` | |
| **`__getKeyNumber`** | Normalizes key code. | event | integer | |
| **`__maskArray`** | Core routine that builds the masked string. | raw array, non‑fixed mask array, full mask array, reverse flag, default value array, signal, offset | masked array | |
| **`__applyDefaultValue`** | Injects default value into array. | array | array | |
| **`__removeInvalidChars`** | Removes characters not matching the mask. | array, rule array | array | |
| **`__applyMask`** | Inserts mask literals into array. | array, mask array | array | |
| **`__extraPositionsTill`** | Counts consecutive literal positions from a start index. | index, mask array | integer | |
| **`__setRange`** | Sets cursor position in the element. | element, start, end | none | |
| **`__getRangePosition`** | Returns current selection start/end. | element | `{start, end}` | |

**Reusable Utilities**  
`__getKeyNumber`, `__setRange`, `__getRangePosition`, and the regex helpers (`fixedCharsReg`, `fixedCharsRegG`) can be extracted for use elsewhere.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery** | Third‑party | Used for DOM manipulation, event binding, data storage, and cross‑browser utilities (`$.inArray`, `$.extend`, `$.metadata`). |
| **`$.metadata`** (optional) | Third‑party | If present, pulls mask definitions from element metadata. |
| **Browser detection** (`jQuery.browser`) | Standard (deprecated in newer jQuery) | Used for IE, Safari, Opera, and older Mozilla versions to decide `maxlength` handling and event names. |
| **Native DOM APIs** | Platform‑specific | `setSelectionRange`, `createTextRange`, `selectionStart/End`. |

**Assumptions / Constraints**

* Relies on `$.browser`, which has been removed from jQuery ≥ 1.9. This code will break on newer versions unless a polyfill or custom detection is added.  
* Uses `maxlength` in a browser‑specific way; some older browsers ignore `maxlength` for `contenteditable` or non‑input elements.  
* The mask parser assumes mask strings are already well‑formed; malformed masks (e.g., unmatched parentheses) can lead to unexpected results.

---

## 5. Additional Notes

### Strengths
* **Extensible** – masks can be added via `$.mask.masks` or passed inline.  
* **Real‑time validation** – immediately rejects invalid keystrokes and provides callbacks.  
* **Reverse mode** – supports currency/number input that grows from the right.  
* **Signal support** – allows a leading `+` or `-` in reversed masks.  

### Potential Issues / Edge Cases
1. **jQuery 1.9+ Compatibility** – `$.browser` and `$.metadata` are removed; the plugin needs updating or a polyfill.  
2. **Touch/Virtual Keyboards** – iPhone ignore keys (`iphoneIgnoreKeys`) might not cover all mobile keyboards; modern browsers expose `input` events differently.  
3. **Non‑input Elements** – The plugin sets `maxlength` and uses `setSelectionRange`; it will not work on `<div contenteditable>`.  
4. **Performance** – For very long fields or many concurrent masked inputs, the repeated array manipulation (`splice`, `join`) may be costly.  
5. **Security** – No sanitization of user-provided mask strings – an attacker could inject arbitrary regex via `mask.rules`.  
6. **Unicode** – The mask regexes are ASCII‑only; input containing Unicode letters or digits may be mishandled.

### Suggested Enhancements
| Area | Recommendation |
|------|----------------|
| **Modernization** | Replace `$.browser` with feature detection or a lightweight polyfill; remove `$.metadata` or make it optional. |
| **Performance** | Cache the compiled mask objects; avoid repeated `join`/`split` by working directly on strings where possible. |
| **Accessibility** | Add ARIA attributes when mask changes to inform screen readers. |
| **Unicode Support** | Extend the default `rules` to support Unicode word/number classes (`\w`, `\d`). |
| **Unit Tests** | Write Jasmine/Mocha tests covering all mask types, edge cases, and event flows. |
| **Documentation** | Provide clear examples, configuration options, and a migration guide for newer jQuery. |
| **Signal Handling** | Clarify behavior for multiple signals and allow custom signal symbols via options. |

---

### Final Verdict

The plugin is a solid, feature‑rich solution for masking inputs in the era it was written (2013‑2014). Its design is straightforward and well‑encapsulated, but it relies on deprecated jQuery APIs and lacks modern browser support. With a few refactors (removing `$.browser`, adding feature detection, optional metadata support, and polishing Unicode handling), it can be brought up to current standards and become a reusable component for modern web applications.

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
