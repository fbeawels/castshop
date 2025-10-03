# jquery.preview.js

## Review

## 1. Summary  
The script adds a lightweight “image preview” tooltip to all links that carry the class `preview`. When the user hovers over such a link, a floating `<p>` element is appended to the `<body>` containing the target image and an optional title. The element follows the cursor as it moves and is removed when the cursor leaves the link.  

**Key components**  
- **Configuration section** – sets pixel offsets that position the preview relative to the cursor.  
- **Hover handlers** – create/destroy the preview element, preserve the original title attribute, and fade it in/out.  
- **Mouse‑move handler** – updates the preview’s position in real time.  
- **Document ready bootstrap** – invokes the `imagePreview` initializer when the DOM is ready.  

The code relies on jQuery 1.x for DOM manipulation and event handling. No advanced design patterns are employed; the implementation is a simple functional module that attaches event listeners directly to matched elements.

---

## 2. Detailed Description  
1. **Configuration**  
   ```js
   xOffset = 200;
   yOffset = 70;
   ```  
   These globals determine how far the preview element is offset from the cursor on the x‑ and y‑axes. Because they are declared without `var`, `let`, or `const`, they become **global variables** that can unintentionally collide with other code.  

2. **Hover handler (mouseenter)**  
   - Saves the link’s original `title` in `this.t` and clears the `title` attribute to suppress the browser tooltip.  
   - Constructs an optional caption `<br/>` + title.  
   - Appends a `<p id="preview">` to the body, containing the image referenced by `this.href`.  
   - Positions the preview element relative to the mouse event coordinates using `pageX`/`pageY` plus the offsets, then fades it in.  

3. **Hover handler (mouseleave)**  
   - Restores the original title.  
   - Removes the `<p id="preview">` from the DOM.  

4. **Mouse‑move handler**  
   Updates the preview element’s `top` and `left` CSS properties on every cursor move to keep it anchored near the mouse.  

5. **Bootstrap**  
   `jQuery(document).ready(function(){ imagePreview(); });` runs the module once the page’s DOM is ready.  

The code assumes that all links with class `preview` point directly to image files. It also assumes that only one preview element will exist at any time (since it uses a hard‑coded `id="preview"`). If multiple preview links are hovered simultaneously, the element will be recreated for each but the old one will be removed when the previous link is left, which may lead to flickering.

---

## 3. Functions/Methods  

| Function / Method | Purpose | Parameters | Return | Side‑Effects |
|-------------------|---------|------------|--------|--------------|
| `this.imagePreview = function(){ ... }` | Initializes event handlers for image preview. | None | None | Creates global vars `xOffset`, `yOffset`; attaches `hover` and `mousemove` listeners to `a.preview`. |
| `jQuery("a.preview").hover(enterFn, leaveFn)` | jQuery shortcut for `mouseenter`/`mouseleave`. | `enterFn(e)`, `leaveFn()` | None | Appends/removes the preview `<p>`, manipulates element’s title, fades in/out. |
| `enterFn(e)` | Executes on hover entry. | `e` (jQuery event) | None | Stores title, creates `<p id="preview">`, positions it. |
| `leaveFn()` | Executes on hover exit. | None | None | Restores title, removes `<p>`. |
| `jQuery("a.preview").mousemove(fn)` | Updates preview position during hover. | `fn(e)` | None | Moves `<p>` with cursor. |
| `jQuery(document).ready(fn)` | jQuery DOM ready event. | `fn()` | None | Calls `imagePreview()`. |

*Utility aspects:*  
- The code uses `this.t` to stash the original title; this is a quick, albeit non‑standard, technique to maintain state per element.  
- `jQuery("#preview")` selects the element by ID; because the ID is hard‑coded, no need for a reusable selector.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery** | Third‑party library | Required for `$`, event binding, DOM traversal, and animation (`fadeIn`). The code assumes a global `$`/`jQuery` variable is available. |
| Browser DOM APIs | Standard | `document`, `body`, CSS properties, `pageX`, `pageY`. |
| CSS (not shown) | Optional | The preview `<p>` will need CSS to style background, border, etc. |

There are no platform‑specific APIs; the script is intended for typical web browsers supporting ES5 and jQuery 1.x.

---

## 5. Additional Notes  
### Strengths  
- **Simplicity** – Easy to understand and integrate into existing pages.  
- **Minimal footprint** – Only a few lines of JavaScript and a single CSS selector.  
- **Immediate visual feedback** – Image appears instantly and follows the cursor.

### Weaknesses & Edge Cases  
1. **Global variables** – `xOffset`, `yOffset` leak into the global namespace; future code could inadvertently overwrite them.  
2. **ID collision** – Only one preview element can exist because it uses `id="preview"`. Multiple simultaneous hovers could cause flicker or race conditions.  
3. **Title preservation** – Storing the title in `this.t` works only while the element remains in the DOM; if the element is replaced or removed elsewhere, the original title is lost.  
4. **Image load times** – The image is inserted directly without preloading; if the image is large or slow to download, the tooltip may appear blank or delayed.  
5. **Accessibility** – The script does not provide ARIA attributes or focus handling, making it inaccessible to keyboard users or screen readers.  
6. **Mobile devices** – Hover events are not meaningful on touch screens; the tooltip will never appear.  
7. **CSS styling** – The script relies on external CSS for layout; without it, the preview may appear unstyled or overlap other content.

### Potential Enhancements  
- **Encapsulate in an IIFE or ES6 module** to avoid polluting the global namespace.  
- **Use data attributes** (`data-preview-title`, `data-preview-src`) instead of relying on the `title` attribute and link `href`.  
- **Preload images** or show a loading indicator while the preview is fetching.  
- **Support for multiple concurrent previews** by generating unique IDs or using a class instead of an ID.  
- **Add touch support**: show preview on tap or long‑press for mobile devices.  
- **Accessibility improvements**: ARIA roles, keyboard focus, and `alt` text handling.  
- **Configuration via options object**: allow users to customize offsets, fade duration, CSS classes, etc.  
- **Graceful degradation**: hide the tooltip if the image fails to load or if jQuery is unavailable.

Overall, the script delivers the core functionality effectively but could benefit from modern JavaScript practices, better encapsulation, and broader device support.

## Code Critique



## Code Preview

```javascript
/*
 * Image preview script 
 * powered by jQuery (http://www.jquery.com)
 * 
 * written by Alen Grakalic (http://cssglobe.com)
 * 
 * for more info visit http://cssglobe.com/post/1695/easiest-tooltip-and-image-preview-using-jquery
 *
 */
 
this.imagePreview = function(){	
	/* CONFIG */
		
		xOffset = 200;
		yOffset = 70;
		
		// these 2 variable determine popup's distance from the cursor
		// you might want to adjust to get the right result
		
	/* END CONFIG */
	jQuery("a.preview").hover(function(e){
		this.t = this.title;
		this.title = "";	
		var c = (this.t != "") ? "<br/>" + this.t : "";
		jQuery("body").append("<p id='preview'><img src='"+ this.href +"' alt='Image preview' />"+ c +"</p>");								 
		jQuery("#preview")
			.css("top",(e.pageY - xOffset) + "px")
			.css("left",(e.pageX + yOffset) + "px")
			.fadeIn("fast");						
    },
	function(){
		this.title = this.t;	
		jQuery("#preview").remove();
    });	
	jQuery("a.preview").mousemove(function(e){
		jQuery("#preview")
			.css("top",(e.pageY - xOffset) + "px")
			.css("left",(e.pageX + yOffset) + "px");
	});			
};


// starting the script on page load
jQuery(document).ready(function(){
	imagePreview();
});


```
