# jquery.Jcrop.css

## Review

## 1. Summary

This snippet is a **CSS stylesheet** that styles the Jcrop image‑cropping plugin.  
Its purpose is to:

* Apply the original Jcrop visual elements (handles, guide lines, tracker area) with a consistent look across browsers.
* Fix a known bug (`http://code.google.com/p/jcrop/issues/detail?id=1`) that affected the layout of the cropper holder.
* Provide a **custom theme** (`.custom` selector) that overrides the default colors and borders for a different visual style.

Key components:

| Component | Role |
|-----------|------|
| `.jcrop-holder` | The main container for the cropper. |
| `.jcrop-vline`, `.jcrop-hline` | Vertical and horizontal guide lines. |
| `.jcrop-handle` | The draggable handles that define the crop rectangle. |
| `.jcrop-tracker` | Invisible overlay that captures mouse events. |
| `.custom …` | Optional namespace that changes the default Jcrop appearance. |

No external frameworks are referenced here; the stylesheet relies on the Jcrop plugin’s HTML structure and a single sprite image (`Jcrop.gif`).

---

## 2. Detailed Description

### Execution Flow

1. **Page Load** – The browser parses this stylesheet after the Jcrop plugin has inserted its markup (`<div class="jcrop-holder">…</div>`).  
2. **Style Application** – For every element matching the selectors, the browser calculates the final computed style, taking into account cascade, specificity, and any `!important` rules.  
3. **Rendering** – The cropper is drawn on the page. The `background: white url('../img/Jcrop.gif')` sprite provides the visible cross‑hair lines.  
4. **User Interaction** – The `.jcrop-tracker` receives mouse/touch events, which the Jcrop JavaScript handles to reposition the crop rectangle.  
5. **Optional Customization** – If a `.custom` wrapper is present (e.g., `<div class="custom">…</div>`), the custom rules override the defaults for the nested Jcrop elements.

### Design Choices & Assumptions

* **Use of `font-size: 0`** on the line elements: This is an old trick to remove whitespace that can appear in inline elements or for certain older browsers.  
* **`!important`** on dimensions: The original Jcrop code uses `!important` to override inline styles added by JavaScript.  
* **Legacy hacks (`*width`, `*height`)**: These are targeted at old versions of IE (≤8).  
* **Image path** `../img/Jcrop.gif`: Assumes the CSS file is in a folder that is one level deeper than the `img` directory.  
* **Vendor prefixes**: Only `-moz-` and `-webkit-` are used for border radius. Modern browsers no longer need them, but they provide backward compatibility.

---

## 3. Selectors / “Functions”

| Selector | Purpose | Notes |
|----------|---------|-------|
| `.jcrop-holder` | Sets the text alignment of the cropper container. | Small but can affect layout if the container holds inline elements. |
| `.jcrop-vline, .jcrop-hline` | Base styling for the vertical and horizontal guide lines. | Uses `font-size: 0` to suppress text gaps; background sprite provides the visible line. |
| `.jcrop-vline` | Sets full height, 1px width (forced with `!important`). | Forces vertical line across entire container. |
| `.jcrop-hline` | Sets full width, 1px height (forced with `!important`). | Forces horizontal line across entire container. |
| `.jcrop-handle` | Styling for the draggable handles. | Sets 7x7px square, border, background color; includes IE6/7 hacks for 9px sizing. |
| `.jcrop-tracker` | Invisible overlay covering the crop area to capture events. | Full width/height ensures it sits over all handles and lines. |
| `.custom .jcrop-vline, .custom .jcrop-hline` | Overrides the default line color for custom themes. | Background set to `yellow`. |
| `.custom .jcrop-handle` | Customizes handle appearance. | Dark background, black border, rounded corners via vendor prefixes. |

Although these are CSS selectors rather than executable functions, they serve the same “utility” purpose by encapsulating styling logic.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| **Jcrop Plugin** | Third‑party | The stylesheet is designed to work with the DOM structure and inline styles that Jcrop injects. |
| **Jcrop.gif** | Asset | A small sprite image that contains the cross‑hair line graphics. Must be located at `../img/Jcrop.gif` relative to this CSS file. |
| **Browser Rendering Engine** | Standard | Relies on support for CSS positioning, absolute/fixed layout, and background images. |
| **Legacy Browser Support** | Platform‑specific | Uses IE hacks (`*width`, `*height`) and vendor prefixes for older WebKit browsers. |

No additional JavaScript libraries or CSS frameworks are referenced here.

---

## 5. Additional Notes & Recommendations

### Strengths

* **Clear separation** between default and custom styles via the `.custom` namespace.  
* Minimal CSS overrides (`!important`) only where required, preserving Jcrop’s dynamic inline styles.  
* Maintains backward compatibility with very old browsers while still functional in modern ones.

### Potential Issues & Edge Cases

1. **Path to `Jcrop.gif`** – If the directory structure changes, the background image will break, leaving invisible guide lines. Consider using a relative URL based on the stylesheet’s location or embedding the sprite as a data URL for portability.  
2. **`!important` on dimensions** – While necessary for legacy support, it can make future overrides difficult. Modern Jcrop versions often remove the need for `!important`; you might refactor if using a newer release.  
3. **Vendor prefixes** – Only `-moz-` and `-webkit-` are used. Newer browsers (e.g., Edge) use the standard `border-radius`. Adding `border-radius` without prefixes is harmless.  
4. **IE6/7 hacks (`*width`, `*height`)** – These hacks are no longer relevant; they only serve older versions that are not widely used. Removing them would reduce CSS size.  
5. **Color contrast** – The default handle is dark (`#333`) on a white background, which may not satisfy accessibility guidelines for some users. Providing a higher‑contrast option or theming support could improve usability.  
6. **Custom Theme Inheritance** – The `.custom` selectors override only the colors, not the `!important` flags. If you need to change dimensions or other properties, you may need to replicate the `!important` rules.  

### Suggested Enhancements

* **Use CSS variables** for colors and sizes so that theming becomes easier (e.g., `--handle-color`, `--line-color`).  
* **Add a fallback** for browsers that do not support the sprite (e.g., use `border-left`, `border-top` to draw lines).  
* **Leverage media queries** if you plan to support touch‑only devices; e.g., larger handles for easier tapping.  
* **Remove legacy hacks** to streamline the stylesheet, documenting that support for IE6/7 is dropped.  
* **Include a `@media` rule** to adjust the handle size on very high‑resolution displays (retina).  

Overall, the CSS is concise and correctly targets the Jcrop markup. Minor refactors can modernize the code and improve maintainability while preserving the intended functionality.

## Code Critique



## Code Preview

```css
/* Fixes issue here http://code.google.com/p/jcrop/issues/detail?id=1 */
.jcrop-holder { text-align: left; }

.jcrop-vline, .jcrop-hline
{
	font-size: 0;
	position: absolute;
	background: white url('../img/Jcrop.gif') top left repeat;
}
.jcrop-vline { height: 100%; width: 1px !important; }
.jcrop-hline { width: 100%; height: 1px !important; }
.jcrop-handle {
	font-size: 1px;
	width: 7px !important;
	height: 7px !important;
	border: 1px #eee solid;
	background-color: #333;
	*width: 9px;
	*height: 9px;
}

.jcrop-tracker { width: 100%; height: 100%; }

.custom .jcrop-vline,
.custom .jcrop-hline
{
	background: yellow;
}
.custom .jcrop-handle
{
	border-color: black;
	background-color: #C7BB00;
	-moz-border-radius: 3px;
	-webkit-border-radius: 3px;
}



```
