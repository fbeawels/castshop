# jquery.rating.css

## Review

## 1. Summary  
The CSS snippet is a classic “star rating” stylesheet that powers the jQuery.Rating plugin (http://www.fyneworks.com/jquery/star-rating/). It defines the visual representation of a rating control, including stars, hover effects, cancel/clear buttons, and read‑only styling. The code relies on background images (`star.gif` and `delete.gif`) placed in an `../img/` folder and uses simple selectors (`div.star-rating`, `div.rating-cancel`, etc.) to style both the container and the inner `<a>` elements that act as the clickable rating units.

### Key components
| Component | Purpose |
|-----------|---------|
| `div.rating-cancel` / `div.star-rating` | Container for the cancel button and star grid |
| `div.rating-cancel a`, `div.star-rating a` | Individual rating “units” (stars or delete icon) |
| `.star-rating-on a` | Styling for a star that is “active” (selected) |
| `.star-rating-hover a` | Styling for a star on mouse‑over |
| `.star-rating-readonly a` | Makes the control non‑interactive |
| `.star-rating` (background reset) | Overrides default background to hide any unwanted styling |

The plugin uses a CSS sprite approach: the background image changes position to show the appropriate sprite slice (full star, half star, empty star, or delete icon). The code also uses `text-indent: -999em` to hide any text inside the `<a>` elements, a common technique for accessibility before the widespread use of `aria-label`.

## 2. Detailed Description  
### Execution Flow  
1. **Markup**: The plugin injects a `<div class="star-rating">` element containing a series of `<a>` tags, one for each star. An optional `<div class="rating-cancel">` can be added for the delete icon.  
2. **Styling**:  
   * The container divs are floated left, given a fixed width (17 px) and height (15 px) so the layout remains consistent.  
   * Each `<a>` is styled as a block element (`display:block`) with a fixed width (16 px) and 100 % height to fill the container.  
   * `background:url(../img/star.gif)` is applied to all `<a>` elements in the rating container.  
   * Hovering or selecting a star changes the background position via `.star-rating-hover a` and `.star-rating-on a`.  
   * The cancel button uses its own background (`../img/delete.gif`).  
   * Read‑only mode (`.star-rating-readonly a`) simply overrides the cursor.  
3. **Interaction**: JavaScript from the jQuery.Rating plugin listens for mouse events on the `<a>` elements, updates the selected class (`.star-rating-on`) accordingly, and triggers callbacks (e.g., `onSubmit`, `onChange`).  

### Assumptions & Constraints  
* **Image paths** are relative (`../img/...`). The HTML must be placed such that this path resolves correctly.  
* **Fixed dimensions** assume a desktop‑centric layout; there is no responsiveness.  
* **Sprite logic** assumes the `star.gif` contains three 16 px × 16 px frames stacked vertically (full, half, empty).  
* **Accessibility** relies on the hidden text trick; modern accessibility would benefit from `aria-label` or `role="img"`.  
* **Browser support**: The code uses standard CSS properties (float, background, cursor). No vendor prefixes mean it may not work in very old browsers (IE 6/7).  

### Architecture & Design Choices  
The design follows a *sprite‑based, static‑size* pattern common in early 2000s web UI components. By using `<a>` tags for each star, the component gains built‑in focusability and keyboard accessibility (via the plugin’s JS). The use of CSS classes to denote state (`on`, `hover`, `readonly`) keeps the styling logic declarative and separate from behavior.

## 3. Functions/Methods  
CSS does not contain executable functions; however, the styles implicitly correspond to *behavioral states* handled by the plugin’s JavaScript:
| State Class | Visual Effect | Triggering Event |
|-------------|---------------|------------------|
| `.star-rating-on a` | Full star (background positioned -16px) | Click or keyboard selection |
| `.star-rating-hover a` | Hover star (background positioned -32px) | Mouseover |
| `.star-rating-readonly a` | Non‑interactive cursor | Read‑only mode enabled |
| `div.rating-cancel a` | Delete icon (background positioned 0,-16px) | Click on cancel |

The CSS itself is a collection of style rules; no reusable utility methods exist beyond these class definitions.

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `../img/star.gif`, `../img/delete.gif` | External assets | Must be present in the expected directory structure |
| jQuery | Third‑party library | Required by the plugin script (not shown here) |
| Browser rendering engine | Standard | No experimental prefixes used |

No platform‑specific APIs are referenced. The CSS is pure and should work in any modern browser that supports basic layout and background positioning.

## 5. Additional Notes  
### Edge Cases & Limitations  
1. **Responsive Design**: The fixed 17 px width/height prevents the rating from scaling on mobile devices or high‑density displays.  
2. **High‑DPI / Retina**: The background images are not retina‑ready; on a 2× device the icons will appear blurry.  
3. **No Fallback**: If the image files are missing, the `<a>` tags will be invisible but still clickable.  
4. **Accessibility**: Relying on `text-indent:-999em` hides the text but may still expose it to screen readers; using `aria-label` or visually hidden text would be safer.  
5. **Hover vs. Focus**: The `.star-rating-hover a` class is only applied on mouseover; keyboard navigation does not provide visual feedback unless the plugin explicitly handles focus styles.

### Potential Enhancements  
* **SVG or CSS‑only stars**: Replace bitmap sprites with inline SVG or CSS shapes (e.g., `clip-path`) for scalability and easier theming.  
* **CSS Variables**: Introduce `--star-size`, `--star-color`, etc., for easier customization.  
* **Flexbox/Grid**: Use modern layout modules instead of floats for cleaner markup.  
* **Transitions**: Add subtle `transition` effects for hover/active states to improve UX.  
* **Accessibility Improvements**: Add `role="radiogroup"` on the container and `role="radio"` on each `<a>`, along with `aria-checked` and `aria-label`.  
* **High‑DPI Sprites**: Provide 2× and 3× image versions and use media queries or `srcset` logic to swap them.  
* **Responsive Scaling**: Wrap the rating control in a container that can scale with `transform: scale()` or use relative units (em, rem).  

In summary, the CSS is a solid, minimal implementation for a legacy star‑rating widget. It works well for its original context but would benefit from modernization if used in a new project, particularly regarding responsiveness, accessibility, and image handling.

## Code Critique



## Code Preview

```css
/* jQuery.Rating Plugin CSS - http://www.fyneworks.com/jquery/star-rating/ */
div.rating-cancel,div.star-rating{float:left;width:17px;height:15px;text-indent:-999em;cursor:pointer;display:block;background:transparent;overflow:hidden}
div.rating-cancel,div.rating-cancel a{background:url(../img/delete.gif) no-repeat 0 -16px}
div.star-rating,div.star-rating a{background:url(../img/star.gif) no-repeat 0 0px}
div.rating-cancel a,div.star-rating a{display:block;width:16px;height:100%;background-position:0 0px;border:0}
div.star-rating-on a{background-position:0 -16px!important}
div.star-rating-hover a{background-position:0 -32px}
/* Read Only CSS */
div.star-rating-readonly a{cursor:default !important}
/* Partial Star CSS */
div.star-rating{background:transparent!important;overflow:hidden!important}
/* END jQuery.Rating Plugin CSS */


```
