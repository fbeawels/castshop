# plugin.js

## Review

## 1. Summary  
The snippet is a **CKEditor plugin named `popup`**.  
It extends the editor’s prototype with a single utility method `popup(url, width, height)`.  
When invoked, the method opens a new browser window (popup) pointing to the supplied URL and sizes it relative to the user’s screen.  
The plugin relies only on the core CKEditor APIs (`CKEDITOR.plugins.add`, `CKEDITOR.tools.extend`) and the standard `window` object; no external libraries are required.

---

## 2. Detailed Description  

### Core Flow  
1. **Plugin registration**  
   ```js
   CKEDITOR.plugins.add('popup');
   ```
   The call registers a plugin named `popup`. No `init` function is supplied, so the plugin only adds a method to the editor prototype.

2. **Method extension**  
   ```js
   CKEDITOR.tools.extend(CKEDITOR.editor.prototype, {
       popup: function(url, w, h) { … }
   });
   ```
   The `popup` method is appended to every `CKEDITOR.editor` instance.

3. **Parameter handling**  
   * `w` and `h` default to `'80%'` and `'70%'` if undefined.  
   * If they are percentage strings (e.g. `"80%"`), the function converts them to pixel values based on the screen dimensions.  
   * Minimum size enforcement: width ≥ 640 px, height ≥ 420 px.

4. **Position calculation**  
   The popup is centered on the screen using simple arithmetic:  
   ```js
   top  = (screen.height - height) / 2;
   left = (screen.width  - width)  / 2;
   ```

5. **Feature string construction**  
   ```js
   var features = 'location=no,menubar=no,toolbar=no,...,width=...,height=...,top=...,left=...';
   ```

6. **Window opening & fallback**  
   * `window.open('', null, features, true)` is used first to create a *blank* window.  
   * If that succeeds, the script attempts to move/resize/focus the window and then navigates it to the desired URL via `location.href`.  
   * If any of these operations throw an exception (e.g., due to cross‑origin restrictions or popup‑blockers), a second `window.open(url, null, features, true)` is executed.  
   * The function returns `true` on success and `false` if the initial `window.open` fails.

### Design & Assumptions  
* **Simplicity** – the plugin only adds a helper; no UI integration or configuration options are provided.  
* **Browser environment** – the code assumes a normal browser with `window` and `screen` objects; it does not handle non‑browser contexts.  
* **Popup blockers** – the logic tries to mitigate failures by opening a blank window first, but it still relies on the browser allowing popups.  
* **Security** – navigating to `url` via `location.href` may trigger same‑origin policy checks; the fallback `window.open(url, …)` may bypass some restrictions but still respects browser security.

---

## 3. Functions/Methods  

| Name | Purpose | Parameters | Return Value | Side Effects |
|------|---------|------------|--------------|--------------|
| `popup` | Opens a new browser window pointing to `url` with optional width/height. | `url` (string) – target URL.<br>`w` (string|number, optional) – width (`px` or `%`).<br>`h` (string|number, optional) – height (`px` or `%`). | `boolean` – `true` if the window was opened successfully, `false` otherwise. | Creates a new window (or reuses one), navigates it to `url`, may trigger focus/resize. |

*Reusable helpers* – None beyond the single method; the logic is self‑contained.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party (CKEditor core) | Provides `plugins`, `tools.extend`, and the `editor` prototype. |
| `window` | Standard browser API | Used for `open`, `moveTo`, `resizeTo`, `focus`, and `location.href`. |
| `screen` | Standard browser API | Used to compute screen dimensions for percentage sizing. |

No other libraries or platform‑specific APIs are required.

---

## 5. Additional Notes  

### Edge Cases & Limitations  
* **Popup blockers** – If the browser blocks the first `window.open('', …)`, the function returns `false` immediately.  
* **Cross‑origin restrictions** – Setting `location.href` on a window opened with a different origin may throw, which is caught and handled by opening the URL directly.  
* **Non‑percentage numbers** – The code treats any numeric value as pixels; passing a string like `"500"` will be parsed as 500 px.  
* **Screen availability** – Uses `screen.width/height`, which may not reflect the available window area on some browsers or OS setups (e.g., taskbars).  
* **Feature string** – Some features (e.g., `alwaysRaised`) may be ignored by modern browsers.

### Possible Enhancements  
1. **Configuration API** – Allow default width/height or custom feature strings to be set per editor instance.  
2. **Better positioning** – Use `screen.availWidth/availHeight` to respect OS UI elements.  
3. **Callback support** – Return a reference to the opened window or provide callbacks for success/failure.  
4. **Graceful degradation** – Offer a fallback inline dialog or modal if popups are blocked.  
5. **Modernization** – Replace legacy `window.open` tricks with `window.open` that directly accepts the URL and rely on the browser to handle positioning.

Overall, the plugin is concise and functional for simple popup needs within CKEditor, but could benefit from additional robustness and configurability for production use.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.plugins.add('popup');CKEDITOR.tools.extend(CKEDITOR.editor.prototype,{popup:function(a,b,c){b=b||'80%';c=c||'70%';if(typeof b=='string'&&b.length>1&&b.substr(b.length-1,1)=='%')b=parseInt(window.screen.width*parseInt(b,10)/100,10);if(typeof c=='string'&&c.length>1&&c.substr(c.length-1,1)=='%')c=parseInt(window.screen.height*parseInt(c,10)/100,10);if(b<640)b=640;if(c<420)c=420;var d=parseInt((window.screen.height-c)/(2),10),e=parseInt((window.screen.width-b)/(2),10),f='location=no,menubar=no,toolbar=no,dependent=yes,minimizable=no,modal=yes,alwaysRaised=yes,resizable=yes,width='+b+',height='+c+',top='+d+',left='+e,g=window.open('',null,f,true);if(!g)return false;try{g.moveTo(e,d);g.resizeTo(b,c);g.focus();g.location.href=a;}catch(h){g=window.open(a,null,f,true);}return true;}});



```
