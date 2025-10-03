# tools.css

## Review

## 1. Summary  
The snippet is a lightweight CSS “utility” library authored by Mike Stenhouse. It supplies three primary sets of styles:

| Utility | Purpose |
|---------|---------|
| **Clearing** (`.stretch`, `.clear`, `.clearfix`) | Standard float‑clearing helpers for legacy browsers (IE 6/7) and a modern clearfix implementation. |
| **Replace** (`.replace`, `.replace *`, `.replace a`, `.replace a span`) | Image‑replacement technique that keeps accessible text in the DOM while hiding it visually, often used for icon sprites or text‑less links. |
| **Accessibility** (`span.accesskey`, `.accessibility`) | Basic tricks for hiding elements from sighted users but keeping them available to screen readers and keyboard navigation. |

The code is self‑contained, uses only CSS (no external frameworks), and is aimed at providing cross‑browser compatibility, especially for older IE versions.

---

## 2. Detailed Description  

### Clearing
1. **`.stretch` / `.clear`**  
   - Set `clear:both;` to prevent floating elements from colliding.  
   - Use a 1 px height and zero margins/padding to create a “gap” element.  
   - The `font-size` and `line-height` adjustments are to suppress the tiny invisible line that can appear in some browsers.

2. **`.clearfix:after`**  
   - The “after” pseudo‑element forces the parent to contain its floated children.  
   - The trick of using a hidden dot (`content: "."`) ensures the element exists in the layout while being visually invisible (`visibility:hidden`).  
   - `display:block; height:0; clear:both;` guarantees proper clearing.

3. **IE Hacks**  
   - `* html>body .clearfix` targets IE 6 in standards mode.  
   - `* html .clearfix` targets IE 6 in quirks mode.  
   - These hacks force the clearfix to behave correctly on those browsers.

### Replace
1. **`.replace`**  
   - Makes the element a block with no default background.  
   - Prepares it for image replacement by setting `background-repeat` and `background-position`.

2. **`.replace *`**  
   - Applies the same block‑style, no‑text rules to all descendants, ensuring that nested elements are also hidden from visual rendering.

3. **`.replace a`**  
   - Re‑enables text for anchor tags so that they are still keyboard‑and‑screen‑reader accessible.

4. **`.replace a span`**  
   - Hides the text inside the anchor’s span by a large negative text‑indent, which is a common sprite technique.

### Accessibility
1. **`span.accesskey`**  
   - Removes underlines from keyboard access key indicators (often shown by browsers).

2. **`.accessibility`**  
   - Moves elements off‑screen by positioning them at `-999em`.  
   - Used for “visually hidden” content that remains in the accessibility tree.

---

## 3. Functions/Methods  
Since this is CSS, there are no programmatic functions, but we can treat each selector group as a “utility”:

| Selector Group | Purpose | Inputs | Outputs | Side Effects |
|----------------|---------|--------|---------|--------------|
| `.stretch`, `.clear` | Float clearing element | HTML element | Layout shift: element occupies 1 px space | None |
| `.clearfix:after` | Contains floats | Container element | The container automatically expands to contain its floated children | Adds a pseudo‑element |
| `.replace` & descendants | Image replacement | Element + CSS background image | Visually shows background image, hides text | Requires appropriate HTML structure (e.g., `<span class="replace"><span>Text</span></span>`) |
| `span.accesskey` | Removes underline from access key indicators | `<span>` with `accesskey` | Underline removed | None |
| `.accessibility` | Visually hides content | Any element | Element positioned off‑screen | None |

---

## 4. Dependencies  
- **None**: The code uses only vanilla CSS.  
- **Browser Support**:  
  - Modern browsers: fully supported.  
  - IE 6/7: targeted with hacks (`* html` selectors).  
  - IE 8 and newer: no need for hacks but still functional.  
- **No external libraries or APIs** required.

---

## 5. Additional Notes  

### Strengths
- **Cross‑Browser Compatibility**: Explicit hacks for IE 6/7 show awareness of legacy issues.  
- **Accessibility Focus**: Provides helpers to keep text in the DOM while hiding it visually, preserving screen‑reader support.  
- **Modular Utilities**: Small, single‑purpose classes make them easy to mix and match.

### Weaknesses / Edge Cases  
1. **Deprecated IE Hacks** – The `* html` selectors are obsolete; newer projects might prefer more modern techniques (e.g., Flexbox or `::after` with `display: table; clear: both;`).  
2. **Hard‑Coded Measurements** – The 1 px height in `.clear` could cause layout flickers on high‑resolution displays or when zoomed.  
3. **Accessibility Overlap** – Using `position: absolute; top: -999em; left: -999em;` can create focusability issues if the element is reachable via keyboard (e.g., tabbing into a hidden menu).  
4. **CSS Specificity** – The universal selector `.replace *` may unintentionally affect nested elements that you might not want hidden.

### Suggested Enhancements
- Replace the `* html` hacks with a modern clearfix:  
  ```css
  .clearfix::after { content:""; display:table; clear:both; }
  ```  
  Modern browsers will ignore the old hacks but still work on IE 7+ with `hasLayout`.  
- For image replacement, consider using `background-image` with `text-indent: -9999px;` or the newer `aria-hidden="true"` attribute to improve accessibility.  
- Offer a “visually hidden” mixin for screen‑reader‑only content, using `clip-path` or `clip` for better keyboard navigation control.  
- Add optional class modifiers (`.clearfix--flex`, `.replace--icon`) to allow easier extension.  

### Potential Future Extensions
- **Responsive Utilities** – Add media‑query variants (e.g., `.clearfix@sm`, `.replace@md`) to adapt clearing and replacement styles at different breakpoints.  
- **Theming** – Introduce CSS variables for common colors or spacing used by the helpers.  
- **Component‑Based CSS** – Wrap these utilities in a small library that can be imported via npm or a CDN for easier inclusion in modern build workflows.  

---

**Conclusion**  
The CSS snippet is a concise, well‑intentioned collection of legacy‑friendly utilities. It covers essential tasks such as float clearing, image replacement, and accessibility hiding, and shows careful attention to older browsers. However, modern web development practices encourage moving away from the older IE hacks, tightening specificity, and leveraging CSS features like Flexbox, Grid, and CSS variables. Updating the utilities accordingly will future‑proof the library while keeping its minimal, reusable nature.

## Code Critique



## Code Preview

```css
/* A CSS Framework by Mike Stenhouse of Content with Style */

/* clearing */
	.stretch,
	.clear {
		clear:both; 
		height:1px; 
		margin:0; 
		padding:0; 
		font-size: 15px;
		line-height: 1px;
	}
	.clearfix:after {
		content: "."; 
		display: block; 
		height: 0; 
		clear: both; 
		visibility: hidden;
	}
	* html>body .clearfix {
		display: inline-block; 
		width: 100%;
	}
	
	* html .clearfix {
		/* Hides from IE-mac \*/
		height: 1%;
		/* End hide from IE-mac */
	}
/* end clearing */


/* replace */
	.replace {
		display:block;
		
		background-repeat: no-repeat;
		background-position: left top;
		background-color:transparent;
	}
	/* tidy these up */
	.replace * {
		text-indent: -10000px;
		display:block;
		
		background-repeat: no-repeat;
		background-position: left top;
		background-color:transparent;
	}
	.replace a {
		text-indent:0;
	}
	.replace a span {
		text-indent:-10000px;
	}
/* end replace */


/* accessibility */
     span.accesskey {
     	text-decoration:none;
     }
     .accessibility {
     	position: absolute;
     	top: -999em;
     	left: -999em;
     }
/* end accessibility */



```
