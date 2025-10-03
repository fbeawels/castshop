# wsc.js

## Review

## 1. Summary

The snippet is a **CKEditor dialog definition** that integrates the *WebSpellChecker (WSC)* spell‑checking service into a CKEditor instance.  
It dynamically creates a small UI (hidden textarea, error div, iframe), loads the external WSC script on demand, and hands control over to the WSC `doSpell` API.  When the user finishes spelling, the corrected text is injected back into the editor.

**Key components**

| Component | Purpose |
|-----------|---------|
| `CKEDITOR.dialog.add('checkspell', …)` | Registers the dialog under the name *checkspell*. |
| `i` | Concatenated HTML string that creates the hidden `<textarea>`, error `<div>`, and `<iframe>`. |
| `j` | URL of the WSC loader script (uses a custom script if supplied). |
| `k` | Polling helper that checks whether the WSC script has defined `doSpell`. |
| `l` | Initializes the WSC spell‑checker (creates language comparer, sets up callbacks). |
| `window._cancelOnError` | Global error handler that displays a fallback message if WSC cannot be loaded. |

The dialog itself (`return { … }`) specifies size, buttons, event handlers (`onShow`, `onHide`) and a single content pane that embeds the markup created in `i`.

**Design patterns / libraries**

* Closure‑based IIFE that scopes the dialog definition.  
* CKEditor API (`CKEDITOR.dialog.add`, `CKEDITOR.document`, `CKEDITOR.tools.getNextNumber`).  
* Global helper functions (`window.doSpell`, `window._SP_FCK_LangCompare`) provided by the external WSC script.

---

## 2. Detailed Description

### Execution Flow

1. **Dialog Registration** – `CKEDITOR.dialog.add` receives a factory function that returns the dialog definition.  
2. **On Show**  
   * The dialog’s content pane (`i`) is injected into the DOM.  
   * The current editor data is copied into the hidden `<textarea>` (`d`).  
   * If the WSC script is not yet loaded (`window.doSpell` missing), a `<script>` element is appended to the document head.  
   * A `setInterval` (`k`) starts polling every 250 ms to see if `doSpell` becomes available.  
3. **When WSC is ready**  
   * `k` calls `l`, which creates a language comparer (`_SP_FCK_LangCompare`) and invokes `doSpell`.  
   * `doSpell` opens the spell‑checker UI inside the provided iframe (`c`).  
   * Two callbacks are supplied:  
     * `onCancel` – hides the dialog.  
     * `onFinish` – receives the corrected text, writes it back into the editor, and hides the dialog.  
4. **On Hide** – The dialog clears a few global flags (`window.ooo`, `window.framesetLoaded`, `window.is_window_opened`).  
   * **Note:** The polling interval (`f`) is never cleared, which can lead to a dangling timer after the dialog closes.  
5. **Error Handling** – If `doSpell` never appears after 45 s (`k` reaches `o === 180`), `window._cancelOnError` shows a user‑friendly message in the error `<div>`.

### Assumptions & Constraints

| Assumption | Effect |
|------------|--------|
| The external script defines `doSpell`, `_SP_FCK_LangCompare`, and other globals. | The code will fail if the script is missing or corrupted. |
| The editor instance (`a`) exposes `lang`, `config`, `getData`, `focus`, etc. | Works only with standard CKEditor instances that support the WSC plugin configuration. |
| Only one instance of the dialog is active at a time. | Global variables (`f`, `window.ooo`, etc.) are overwritten on each show, potentially causing race conditions. |
| The user’s browser permits loading external iframes and script files from the WSC domain. | Cross‑origin restrictions could block the service. |

### Architecture & Design Choices

* **Dynamic UI creation** – The dialog builds its markup as a string (`i`) and injects it into the DOM. This keeps the dialog definition small but makes it harder to read and maintain.  
* **Polling for script readiness** – Instead of listening for the script’s `onload` event, the code repeatedly checks `window.doSpell`. Polling is simple but inefficient and can miss errors if the script is blocked before executing.  
* **Global helpers** – By attaching error handling and language comparison functions to `window`, the dialog avoids passing them through the `doSpell` callback chain. This works but pollutes the global namespace and risks name collisions with other plugins or third‑party code.  
* **Hard‑coded dimensions** – The dialog’s width and height are fixed (485 × 380 px), which may not adapt well to different UI themes or accessibility requirements.

---

## 3. Functions / Methods

| Function / Method | Purpose | Parameters | Returns | Side‑Effects |
|-------------------|---------|------------|---------|--------------|
| `k(m, n)` | Returns a polling function that waits for `doSpell`. | `m`: dialog instance, `n`: fallback error message | `function` | None (except calls `l` when ready) |
| `window._cancelOnError(m)` | Global error handler displayed in the error `<div>`. | `m`: error message | `undefined` | Manipulates the dialog’s `<div>` and iframe visibility |
| `l(m)` | Initializes and invokes the WSC spell‑checker. | `m`: dialog instance | `undefined` | Calls `doSpell`, shows/hides UI elements, writes corrected text to the editor |
| `CKEDITOR.dialog.add('checkspell', function(a){ … })` | Factory that returns the dialog definition. | `a`: CKEditor instance | `Object` (dialog configuration) | Creates UI elements, registers global helpers, starts polling |

#### Reusable / Utility Methods

* **`CKEDITOR.tools.getNextNumber()`** – Generates a unique identifier for each dialog instance.  
* **`CKEDITOR.document.createElement`** – Utility to create and insert `<script>` elements.  
* **`CKEDITOR.document.getById(id).setStyle(style)`** – Used to toggle visibility of the iframe and error message.

---

## 4. Dependencies

| Library / API | Type | Usage |
|---------------|------|-------|
| **CKEditor** | Third‑party | Core editor API, dialog framework, utilities. |
| **WebSpellChecker (WSC) Loader** | Third‑party | Provides `doSpell`, `_SP_FCK_LangCompare`, and related functions. URL configurable via `config.wsc_customLoaderScript`. |
| **Browser DOM** | Standard | Manipulation of `<textarea>`, `<div>`, `<iframe>`, `<script>`. |
| **Global `window` object** | Platform | Stores `doSpell`, `_SP_FCK_LangCompare`, error handlers, and timers. |

No other external libraries are required.

---

## 5. Additional Notes & Recommendations

### Edge Cases & Potential Problems

1. **Unclear Cleanup** – The polling interval (`f`) is never cleared when the dialog closes, leading to a memory leak if the dialog is opened/closed repeatedly.  
2. **Script Loading Failure** – If the external script fails to load (e.g., network error, CSP block), the user sees the error div, but the dialog stays open and the timer continues polling indefinitely.  
3. **Global Namespace Pollution** – Functions such as `window.doSpell`, `window._cancelOnError`, and global flags (`window.ooo`, `framesetLoaded`) can clash with other plugins or custom code.  
4. **Hard‑coded Dimensions & Styles** – The UI may not be responsive or accessible on mobile devices or within thematically styled editors.  
5. **Cross‑Origin Restrictions** – The iframe points to a remote domain (`loader.spellchecker.net`). Browsers with strict CSP or mixed‑content policies may block the iframe in HTTPS pages if the loader URL is not HTTPS.

### Suggested Improvements

| Issue | Fix |
|-------|-----|
| **Polling interval never cleared** | Store `f` on the dialog instance (`m._spellCheckInterval`) and clear it in `onHide` with `clearInterval(m._spellCheckInterval)`. |
| **Script load detection** | Replace polling with script `onload`/`onerror` events, or use `Promise` to await loading. |
| **Global helpers** | Encapsulate error handling and language comparison within the dialog closure or attach them to the dialog instance (`m`) rather than `window`. |
| **Responsive UI** | Use CSS classes instead of inline styles, allow width/height to be configurable via `config`. |
| **Error handling** | Show a modal or toast notification on script load failure, and provide a retry button. |
| **Security** | Ensure the script URL uses HTTPS and add CSP meta tags if required. |

### Future Enhancements

* **Internationalization** – Support additional languages and locale fallback.  
* **Custom Dictionary** – Expose an API to add words to a user‑specific dictionary.  
* **Batch Spell‑Check** – Allow the dialog to spell‑check the entire document rather than only the visible content.  
* **Unit Tests** – Write automated tests for the dialog’s initialization, error handling, and interaction with `doSpell`.  
* **Documentation** – Provide clear API documentation for developers who want to customize or extend the plugin.

---

**Overall,** the code achieves its goal of embedding a third‑party spell‑checker into CKEditor, but it could benefit from modern JavaScript practices (promises, module scopes, clean event handling) to improve maintainability, performance, and security.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('checkspell',function(a){var b=CKEDITOR.tools.getNextNumber(),c='cke_frame_'+b,d='cke_data_'+b,e='cke_error_'+b,f,g=document.location.protocol||'http:',h=a.lang.spellCheck.notAvailable,i='<textarea style="display: none" id="'+d+'"'+' rows="10"'+' cols="40">'+' </textarea><div'+' id="'+e+'"'+' style="display:none;color:red;font-size:16px;font-weight:bold;padding-top:160px;text-align:center;z-index:11;">'+'</div><iframe'+' src=""'+' style="width:485px;background-color:#f1f1e3;height:380px"'+' frameborder="0"'+' name="'+c+'"'+' id="'+c+'"'+' allowtransparency="1">'+'</iframe>',j=a.config.wsc_customLoaderScript||g+'//loader.spellchecker.net/sproxy_fck/sproxy.php'+'?plugin=fck2'+'&customerid='+a.config.wsc_customerId+'&cmd=script&doc=wsc&schema=22';if(a.config.wsc_customLoaderScript)h+='<p style="color:#000;font-size:11px;font-weight: normal;text-align:center;padding-top:10px">'+a.lang.spellCheck.errorLoading.replace(/%s/g,a.config.wsc_customLoaderScript)+'</p>';function k(m,n){var o=0;return function(){if(typeof window.doSpell=='function'){if(typeof f!='undefined')window.clearInterval(f);l(m);}else if(o++==180)window._cancelOnError(n);};};window._cancelOnError=function(m){if(typeof window.WSC_Error=='undefined'){CKEDITOR.document.getById(c).setStyle('display','none');var n=CKEDITOR.document.getById(e);n.setStyle('display','block');n.setHtml(m||a.lang.spellCheck.notAvailable);}};function l(m){var n=new window._SP_FCK_LangCompare(),o=CKEDITOR.getUrl(a.plugins.wsc.path+'dialogs/'),p=o+'tmpFrameset.html';window.gFCKPluginName='wsc';n.setDefaulLangCode(a.config.defaultLanguage);window.doSpell({ctrl:d,lang:n.getSPLangCode(a.langCode),winType:c,onCancel:function(){m.hide();},onFinish:function(q){a.focus();m.getParentEditor().setData(q.value);m.hide();},staticFrame:p,framesetPath:p,iframePath:o+'ciframe.html',schemaURI:o+'wsc.css'});CKEDITOR.document.getById(e).setStyle('display','none');CKEDITOR.document.getById(c).setStyle('display','block');};return{title:a.lang.spellCheck.title,minWidth:485,minHeight:380,buttons:[CKEDITOR.dialog.cancelButton],onShow:function(){var m=this.getContentElement('general','content').getElement();m.setHtml(i);if(typeof window.doSpell!='function')CKEDITOR.document.getHead().append(CKEDITOR.document.createElement('script',{attributes:{type:'text/javascript',src:j}}));var n=a.getData();CKEDITOR.document.getById(d).setValue(n);f=window.setInterval(k(this,h),250);},onHide:function(){window.ooo=undefined;window.int_framsetLoaded=undefined;
window.framesetLoaded=undefined;window.is_window_opened=false;},contents:[{id:'general',label:a.lang.spellCheck.title,padding:0,elements:[{type:'html',id:'content',style:'width:485;height:380px',html:'<div></div>'}]}]};});



```
