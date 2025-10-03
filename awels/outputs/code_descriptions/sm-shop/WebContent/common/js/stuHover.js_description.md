# stuHover.js

## Review

## 1. Summary
The snippet implements an **IE6/IE7 hover‑polyfill**.  
Modern browsers support the CSS `:hover` pseudo‑class on list items, but older versions of Internet Explorer only support it on links (`<a>`). This script:

1. Searches all stylesheets for rules that target `LI:hover`.
2. Generates equivalent rules for a custom class `iehover` and injects them.
3. Attaches mouse‑over/out handlers to all `<li>` elements inside the element with `id="nav"` to toggle the `iehover` class, thus emulating the hover effect.

Key components:
- A single global function `stuHover`.
- Inline event attachment using `window.attachEvent` (IE specific).
- Direct manipulation of the `document.styleSheets` collection.

No external libraries are required; it relies solely on vanilla JavaScript and the browser’s DOM/CSSOM APIs.

## 2. Detailed Description
### Flow of Execution
1. **Definition** – `stuHover` is defined but not executed immediately.
2. **Attachment** – If the browser exposes `attachEvent` (IE 5–8), `stuHover` is bound to the `onload` event.
3. **On Load** – When the page finishes loading, `stuHover` runs.
   - **Rule Collection**: Iterates over every stylesheet (`document.styleSheets`) and every rule (`rules`) inside it.
   - **Rule Matching**: For any rule where `selectorText` contains `LI:hover`, a new selector is created by replacing `LI:hover` with `LI.iehover`.  
   - **Rule Injection**: The new rule (with the same style declarations) is added to the stylesheet via `addRule`.
   - **Element Hooking**: Retrieves all `<li>` descendants of the element with id `nav` and attaches `onmouseover` and `onmouseout` handlers.  
     - On hover, it appends the class `iehover` to the element’s `className`.
     - On mouse out, it removes the trailing `iehover` class.

### Assumptions & Constraints
- The code assumes the browser supports the `document.styleSheets` API and that each stylesheet exposes a `rules` collection (IE only). Modern browsers expose `cssRules` instead, so the loop would fail in standards‑mode browsers.
- The polyfill targets only `<li>` elements that are descendants of a specific container (`#nav`). It won’t work for hover on other elements or on `<li>` elements outside that container.
- The script uses `attachEvent`/`onload` which are IE‑specific; it will silently do nothing in non‑IE browsers, leaving them untouched.
- No namespacing is used; the global `stuHover` function and the global `iehover` class can potentially collide with other code.

### Architecture & Design Choices
- **Global function**: Keeps the implementation simple but risks namespace pollution.
- **Style injection**: Instead of relying on CSS hacks (like `hasLayout` or `filter`), the script programmatically copies the style rules, which is robust across different stylesheets.
- **Class toggling**: The class manipulation is done via string concatenation/regex, which is lightweight but fragile if `className` already contains multiple spaces or the same class elsewhere.

## 3. Functions/Methods

| Function | Purpose | Inputs | Outputs / Side Effects |
|---|---|---|---|
| `stuHover()` | Polyfills `:hover` for `<li>` in IE6/7. | None | - Adds `LI.iehover` rules to stylesheets.<br>- Attaches mouseover/mouseout handlers to `<li>` elements under `#nav`. |
| `window.attachEvent("onload", stuHover)` | Binds `stuHover` to page load for IE. | None | None (event registration). |

**Reusable / Utility Methods**: None; the script is self‑contained with no helper functions.

## 4. Dependencies

| Dependency | Type | Notes |
|---|---|---|
| `document.styleSheets` | Browser DOM API | Standard in IE; `cssRules` is standard in modern browsers. |
| `attachEvent` | IE-specific event API | Deprecated; not available in standards browsers. |
| `addRule` | IE-specific stylesheet method | Equivalent to `insertRule` in modern browsers. |
| CSS class `iehover` | CSS | Must be defined in the author’s stylesheet to actually style the hovered `<li>`. |

No third‑party libraries or APIs are used.

## 5. Additional Notes & Recommendations

### Edge Cases / Limitations
1. **Non‑IE Browsers** – The script will silently do nothing, but modern browsers now support `:hover` on all elements, so the polyfill is unnecessary.  
2. **Multiple `LI:hover` Rules** – The replacement logic only handles the first occurrence of `LI:hover` in a selector. Compound selectors (e.g., `ul > li:hover, .menu li:hover`) will not be correctly transformed.  
3. **Whitespace / ClassName Handling** – The removal regex only strips a trailing space‑prefixed `iehover`. If the class appears elsewhere or with different spacing, it may not be removed correctly.  
4. **Performance** – Iterating over all stylesheets and rules on every page load can be expensive on pages with many CSS files.  
5. **Conflict with Other Scripts** – Using a global function and the unscoped `iehover` class can clash with other code or stylesheets.

### Potential Improvements
- **Namespace the script** (e.g., `var stu = {}; stu.hoverPolyfill = function(){…}`) to avoid global pollution.
- **Use feature detection** instead of `attachEvent`. Check for `document.querySelectorAll('li:hover')` support or `window.addEventListener` to decide whether the polyfill is needed.
- **Support `cssRules` and `insertRule`** to allow the same script to run in standards browsers for future proofing.
- **More robust class toggling**: Use `classList` (or a polyfill) to add/remove classes, or split `className` into an array and manipulate it safely.
- **Handle compound selectors**: Split selector text on commas, process each fragment separately, and join back.
- **Add comments & documentation**: Clarify why the script exists, how to use it, and any prerequisites (e.g., the `iehover` CSS rule).
- **Optional fallback**: Provide a CSS fallback rule for browsers that don’t support `:hover` but are not IE, using the `.iehover` class directly (though this is rarely needed).

### Future Enhancements
- **Modernize for IE8+**: Use `addEventListener` for newer IE and add a `hasLayout` hack if needed.
- **Add support for other elements**: Expose an API to specify which elements should receive the polyfill (not just `#nav > li`).
- **Performance Optimizations**: Cache references to styleSheets/rules, or provide a “manual” trigger to avoid running on every page load.

---

**Bottom line:** The code is a concise, single‑purpose polyfill for a very specific legacy scenario. While it works in the environments it targets, modern web development practices would encourage a different approach (CSS only, or a small feature‑detection script that skips old browsers). The suggested improvements would make the code more robust, maintainable, and compatible with future browsers.

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
