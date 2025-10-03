# screenshot.js

## Review

## 1. Summary  

**Purpose**  
The script creates a lightweight “tooltip‑style” preview for images linked via `<a class="screenshot">`.  
When the user hovers over such a link, an image (taken from the link’s `rel` attribute) is displayed near the cursor, optionally accompanied by the link’s original `title`.  

**Key Components**  

| Component | Role |
|-----------|------|
| `screenshotPreview` function | Sets up the hover/mouse‑move handlers and handles the preview lifecycle. |
| jQuery event handlers (`hover`, `mousemove`) | Capture mouse events and position the preview element. |
| `<p id="screenshot">` element | Acts as the DOM container for the preview image and optional text. |

**Notable Patterns / Libraries**  

* **jQuery** – The code relies entirely on jQuery for DOM manipulation and event handling.  
* **Closure / IIFE‑style** – The function is assigned to `this.screenshotPreview`, but the implementation itself is straightforward procedural code, not using advanced design patterns.  

---

## 2. Detailed Description  

### Flow of Execution  

1. **Document ready**  
   ```js
   jQuery(document).ready(function(){
       screenshotPreview();
   });
   ```
   Once the DOM is fully loaded, `screenshotPreview()` is invoked.

2. **Configuration**  
   Inside `screenshotPreview()` two variables are defined:
   ```js
   xOffset = 10;   // vertical offset from cursor
   yOffset = 30;   // horizontal offset from cursor
   ```
   (Note: these are global because no `var/let/const` keyword is used.)

3. **Hover handler** (`mouseenter`)  
   * Saves the original `title` into `this.t` and clears the element’s `title` so that the browser’s default tooltip doesn’t appear.  
   * Builds optional caption markup (`c`) if a title exists.  
   * Appends a `<p id="screenshot">` element to the body, containing an `<img>` whose `src` comes from the link’s `rel` attribute.  
   * Positions the `<p>` element relative to the mouse coordinates (`e.pageX/Y`) using the offsets.  
   * Fades the element into view with `fadeIn("fast")`.

4. **Hover handler** (`mouseleave`)  
   * Restores the original title.  
   * Removes the preview element from the DOM.

5. **Mousemove handler**  
   * Continuously updates the preview element’s position as the mouse moves over the link.

### Assumptions & Constraints  

| Assumption | Why it matters |
|------------|----------------|
| The `<a>` element has a `rel` attribute containing a valid image URL. | No validation or fallback for missing/invalid URLs. |
| Only one preview element is needed at a time. | Uses a static ID `screenshot`; multiple simultaneous previews would conflict. |
| Browser supports jQuery and `rel` attributes for images. | Older browsers may misinterpret `rel`. |
| No other scripts modify `#screenshot` or the `title` attribute of the links. | Potential clashes if other tooltip systems are in use. |

### Architecture & Design Choices  

* **Procedural style** – The entire behavior is wrapped in a single function; no separation of concerns or modularization.  
* **Global variables** – `xOffset` and `yOffset` are global, which is not ideal for maintainability.  
* **Hard‑coded ID** – The preview container is identified by a fixed ID; better would be a dynamic class or data attribute to allow multiple previews.  
* **No error handling** – The script assumes every `<a.screenshot>` link is well‑formed.

---

## 3. Functions/Methods  

| Function | Purpose | Parameters | Return | Side‑Effects |
|----------|---------|------------|--------|--------------|
| `this.screenshotPreview` | Sets up event listeners for image preview on `<a.screenshot>` links. | None | None | Creates DOM element `<p id="screenshot">` on hover; removes it on mouseleave. |
| `jQuery(document).ready` | Initializes the preview on page load. | Anonymous callback | None | Calls `screenshotPreview()`. |

Internally, the script relies on jQuery methods:
* `hover()` – attaches mouseenter/mouseleave handlers.
* `mousemove()` – updates preview position.
* `append()` – injects preview element.
* `css()` – sets positional styles.
* `fadeIn()` – animates appearance.
* `remove()` – deletes the preview element.

No reusable utility functions are defined; all logic resides within the single `screenshotPreview` function.

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| **jQuery** | Third‑party | Required for DOM manipulation, event handling, and animation. |
| **CSS** (optional) | Standard | The preview element `<p>` has no styling in the script; styling is expected to be provided externally. |

No other frameworks, APIs, or platform‑specific features are used. The script is browser‑agnostic as long as jQuery is available.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Multiple simultaneous previews** – Since the preview element is identified by a single static ID, hovering over several links quickly will result in race conditions or duplicated IDs.  
2. **Broken image URLs** – No error handling for `src` failures; the preview will simply show a broken image icon.  
3. **Off‑screen positioning** – The preview can be positioned outside the viewport if the cursor is near an edge; no boundary checks are performed.  
4. **Accessibility** – The script manipulates the `title` attribute but does not provide ARIA roles or keyboard accessibility.  

### Suggested Enhancements  

| Improvement | Why | How |
|-------------|-----|-----|
| Use `let/const` for offsets | Avoid leaking globals | `const xOffset = 10;` |
| Store preview element in a variable or use a data attribute | Prevent ID collisions | `var preview = $('<p class="screenshot-preview">...</p>');` |
| Add delay before showing preview | Reduce flicker | `setTimeout` on `mouseenter`, `clearTimeout` on `mouseleave` |
| Handle image load errors | Show fallback UI | Attach `onerror` to `<img>` or use CSS `:after` |
| Clamp position to viewport | Prevent overflow | Compute max/min values for `top`/`left` |
| Make script configurable via options object | Flexibility | `function screenshotPreview(options) { … }` |
| Use `data-rel` or `href` instead of `rel` | Modern HTML5 semantics | `$(this).data('preview')` |
| Support keyboard navigation | Accessibility | Add `focus`/`blur` handlers and ARIA roles |

### Clean‑Up / Maintenance  

* The script could be converted into a jQuery plugin for reusability: `$.fn.screenshotPreview = function(options) { … }`.  
* Removing reliance on `this.title` would avoid tampering with native tooltip behavior.  
* Encapsulating the logic in a closure would prevent accidental global namespace pollution.

---

**Verdict** – The code provides a quick, working solution for image previews, but would benefit from modernizing its variable handling, improving robustness, and adding configurability and accessibility features for production use.

## Code Critique



## Code Preview

```javascript
/*
 * Url preview script 
 * powered by jQuery (http://www.jquery.com)
 * 
 * written by Alen Grakalic (http://cssglobe.com)
 * 
 * for more info visit http://cssglobe.com/post/1695/easiest-tooltip-and-image-preview-using-jquery
 *
 */
 
this.screenshotPreview = function(){	
	/* CONFIG */
		
		xOffset = 10;
		yOffset = 30;
		
		// these 2 variable determine popup's distance from the cursor
		// you might want to adjust to get the right result
		
	/* END CONFIG */
	jQuery("a.screenshot").hover(function(e){
		this.t = this.title;
		this.title = "";	
		var c = (this.t != "") ? "<br/>" + this.t : "";
		jQuery("body").append("<p id='screenshot'><img src='"+ this.rel +"' alt='X' />"+ c +"</p>");								 
		jQuery("#screenshot")
			.css("top",(e.pageY - xOffset) + "px")
			.css("left",(e.pageX + yOffset) + "px")
			.fadeIn("fast");						
    },
	function(){
		this.title = this.t;	
		jQuery("#screenshot").remove();
    });	
	jQuery("a.screenshot").mousemove(function(e){
		jQuery("#screenshot")
			.css("top",(e.pageY - xOffset) + "px")
			.css("left",(e.pageX + yOffset) + "px");
	});			
};


// starting the script on page load
jQuery(document).ready(function(){
	screenshotPreview();
});


```
