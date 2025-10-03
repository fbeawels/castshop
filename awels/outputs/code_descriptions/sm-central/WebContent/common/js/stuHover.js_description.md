# stuHover.js

## Review

## 1. Summary  
The script implements a compatibility shim that allows CSS `:hover` selectors on `<li>` elements to work in older versions of Internet Explorer (IE 6/7). It dynamically generates equivalent rules for a custom class `iehover` and attaches mouse‑over/out event handlers that toggle this class.  

### Key components  
* **stuHover()** – Main initializer that scans all stylesheets, duplicates any `LI:hover` rule as an `LI.iehover` rule, and attaches event handlers to every `<li>` under the element with ID `nav`.  
* **IE event binding** – Uses `window.attachEvent("onload", stuHover)` to run the shim on page load in IE.  

### Design patterns / frameworks  
* Very lightweight, no external libraries.  
* Relies on **DOM 0 event handling** (`element.onmouseover`) and **IE‑specific stylesheet manipulation** (`document.styleSheets[i].addRule`).  

---

## 2. Detailed Description  

### Execution flow  
1. **Script load** – The file is included in the page, defining `stuHover`.  
2. **Onload** – In IE, `window.attachEvent("onload", stuHover)` triggers `stuHover`.  
3. **Rule duplication** –  
   * Iterate over all stylesheets in the document.  
   * For each rule, if its selector contains `LI:hover`, replace that text with `LI.iehover`.  
   * Add the new rule to the same stylesheet using `addRule`.  
4. **Event binding** –  
   * Grab all `<li>` elements inside the element with id `nav`.  
   * For each, attach `onmouseover` and `onmouseout` handlers that add/remove the `iehover` class.  
5. **Result** – When a user hovers over a `<li>`, the `iehover` class is applied; CSS rules targeting `LI.iehover` take effect, mimicking the `:hover` behavior.  

### Assumptions & constraints  
* Only IE 6/7 support is required (both lack real `:hover` on non‑link elements).  
* The target markup must contain an element with `id="nav"` that encloses the `<li>`s.  
* Stylesheets must be accessible (no cross‑domain restrictions).  
* The script does not handle dynamic addition of `<li>` elements after page load.  
* Uses deprecated IE methods (`addRule`), so it will fail in modern browsers or in strict mode.  

### Architecture & design choices  
* **Monolithic function** – All logic lives inside a single function; easy to read but not reusable.  
* **Direct DOM access** – No abstraction layer; acceptable for a tiny polyfill.  
* **Hardcoded selectors** – The script specifically targets `LI:hover`; if another selector (e.g., `.menu-item:hover`) is needed, the script must be altered.  

---

## 3. Functions/Methods  

| Function | Purpose | Parameters | Returns | Side‑effects |
|----------|---------|------------|---------|--------------|
| **stuHover** | Initializes the IE hover shim. | None | `undefined` | * Modifies document stylesheets.<br>* Adds event listeners to `<li>` elements.<br>* Creates the `iehover` class. |
| **(inline)** `onmouseover` handler | Adds `iehover` class to the hovered `<li>` element. | `this` refers to the `<li>` element | `undefined` | * Alters the element’s `className`. |
| **(inline)** `onmouseout` handler | Removes `iehover` class from the `<li>` when the mouse leaves. | `this` refers to the `<li>` element | `undefined` | * Alters the element’s `className`. |

The script does not expose any public API; its only observable effect is the addition of the `iehover` class.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `document.styleSheets` | Standard DOM | Exposes all stylesheets; may be limited by cross‑origin policies. |
| `addRule` (IE) | Third‑party/IE‑specific | Only available in IE < 9. |
| `window.attachEvent` | IE‑specific | Modern browsers use `addEventListener`. |
| `document.getElementById` / `getElementsByTagName` | Standard DOM | Simple element retrieval. |

No external libraries (e.g., jQuery) are required.

---

## 5. Additional Notes  

### Edge Cases & Limitations  
* **Non‑IE browsers** – The script will execute the `if (window.attachEvent)` guard, so it is inert in standards‑compliant browsers.  
* **Cross‑origin stylesheets** – If any stylesheet is loaded from another domain and is not accessible, the `for (var x=0; x<document.styleSheets[i].rules.length ; x++)` loop will throw a security error.  
* **Dynamic content** – Adding new `<li>` elements after load will not have the event handlers attached.  
* **Duplicate `iehover`** – If an `<li>` already has the class `iehover`, the onmouseover handler will append another instance (potentially leading to className duplication).  
* **Performance** – Scanning all rules in all stylesheets on page load can be costly for pages with many stylesheets or large rule sets.  

### Potential Improvements  
1. **Modularize** – Extract rule‑duplication logic into a helper function; allow configuration of the selector pattern (e.g., support `.menu-item:hover`).  
2. **Graceful degradation** – Detect if the browser already supports `:hover` on `<li>` and skip the shim entirely.  
3. **Dynamic handling** – Use mutation observers or delegated event handling to support elements added after load.  
4. **Modern fallback** – Provide a polyfill that works in modern browsers by using `addEventListener` and `classList`.  
5. **Avoid string concatenation** – Use `classList.add` / `remove` for cleaner class manipulation.  

### Historical Context  
This script dates from the era when IE6/7 were prevalent and did not support `:hover` on anything other than `<a>` elements. In 2025, all mainstream browsers support this CSS feature, rendering the shim unnecessary. If maintaining legacy sites that still run in these browsers, the code remains functional, but for new projects it should be removed.  

---

## Code Critique



## Code Preview

```javascript
/* ================================================================ 
This copyright notice must be kept untouched in the stylesheet at 
all times.

The original version of this script and the associated (x)html
is available at http://www.stunicholls.com/menu/pro_drop_1.html
Copyright (c) 2005-2007 Stu Nicholls. All rights reserved.
This script and the associated (x)html may be modified in any 
way to fit your requirements.
=================================================================== */
stuHover = function() {
	var cssRule;
	var newSelector;
	for (var i = 0; i < document.styleSheets.length; i++)
		for (var x = 0; x < document.styleSheets[i].rules.length ; x++)
			{
			cssRule = document.styleSheets[i].rules[x];
			if (cssRule.selectorText.indexOf("LI:hover") != -1)
			{
				 newSelector = cssRule.selectorText.replace(/LI:hover/gi, "LI.iehover");
				document.styleSheets[i].addRule(newSelector , cssRule.style.cssText);
			}
		}
	var getElm = document.getElementById("nav").getElementsByTagName("LI");
	for (var i=0; i<getElm.length; i++) {
		getElm[i].onmouseover=function() {
			this.className+=" iehover";
		}
		getElm[i].onmouseout=function() {
			this.className=this.className.replace(new RegExp(" iehover\\b"), "");
		}
	}
}
if (window.attachEvent) window.attachEvent("onload", stuHover);




```
