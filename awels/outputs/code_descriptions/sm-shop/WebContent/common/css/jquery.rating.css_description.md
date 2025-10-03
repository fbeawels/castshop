# jquery.rating.css

## Review

## 1. Summary  
The snippet is the stylesheet for the **jQuery.Rating** plugin (http://www.fyneworks.com/jquery/star-rating/).  
It supplies the visual representation of star‑rating widgets, cancel icons, hover states, and read‑only/partial‑star variants.  

### Key components  
| Element | Role |
|---------|------|
| `div.rating-cancel` / `div.rating-cancel a` | The “cancel” icon (usually a little X or trash can) that clears the rating. |
| `div.star-rating` / `div.star-rating a` | The star container and individual star anchors that display the rating value. |
| `div.star-rating-on a` | The style applied when a star is “on” (selected). |
| `div.star-rating-hover a` | The style applied during mouse‑over for hover feedback. |
| `div.star-rating-readonly a` | The style that disables pointer events for read‑only ratings. |
| `div.star-rating` (second rule) | Overrides any other background/overflow settings to keep stars contained. |

The CSS relies on background images (`delete.gif`, `star.gif`) located relative to the stylesheet (`../img/`).  
It uses classic *float* layout and absolute pixel values, a common pattern for the time when the plugin was created.

## 2. Detailed Description  
### Execution Flow  
1. **HTML structure**  
   ```html
   <div class="star-rating">
     <a href="#0"></a>
     <a href="#1"></a>
     …
   </div>
   <div class="rating-cancel"><a href="#clear"></a></div>
   ```  
   The plugin populates `<a>` tags for each rating level; the JS manipulates CSS classes (`star-rating-on`, `star-rating-hover`, `star-rating-readonly`) to reflect the current state.

2. **Styling logic**  
   * The default state shows the first part of `star.gif` (empty star).  
   * Hovering triggers `star-rating-hover` → background shifts to a different sprite segment.  
   * Clicking a star adds `star-rating-on` to that star and all preceding ones, revealing the "filled" sprite segment.  
   * The cancel icon uses `delete.gif` with its own sprite offsets.  

3. **Read‑only**  
   Applying `star-rating-readonly` prevents pointer events (cursor: default) and disables hover/selection logic.

4. **Partial stars**  
   The second rule for `div.star-rating` (`background:transparent!important; overflow:hidden!important`) ensures that partial star states (e.g., 3.5 stars) are clipped correctly.

### Design choices  
* **Sprite sheets**: Minimizes HTTP requests but hard‑codes pixel offsets.  
* **Float layout**: Keeps stars inline without flexbox or grid.  
* **Hard‑coded pixel dimensions** (17×15px) make the component inflexible for different resolutions.  
* **`!important`** is used sparingly but can create specificity headaches.

## 3. Functions/Methods  
The file is pure CSS – no executable functions. The “methods” are implicit through class names:

| Selector | Effect |
|----------|--------|
| `div.star-rating a` | Base star (empty) |
| `div.star-rating-on a` | Filled star |
| `div.star-rating-hover a` | Hovered star |
| `div.rating-cancel a` | Cancel icon |
| `div.star-rating-readonly a` | Read‑only state |

Each selector changes the `background-position`, effectively switching sprite frames.

## 4. Dependencies  
| Item | Type | Notes |
|------|------|-------|
| `../img/delete.gif` | Image sprite | Must reside in `img/` relative to the CSS file. |
| `../img/star.gif` | Image sprite | Same location requirement. |
| jQuery | Third‑party JS | Required by the plugin to manipulate classes. |
| Browser support | Classic CSS (float, sprites) | Works in IE6+, but newer browsers favor flex/grid. |

The stylesheet is **not** dependent on any CSS preprocessors or frameworks; it’s plain vanilla CSS.

## 5. Additional Notes  
### Edge Cases & Limitations  
* **Responsive design** – Fixed pixel sizes mean the component does not scale on mobile devices.  
* **Accessibility** – No ARIA attributes or keyboard navigation hooks; rely entirely on mouse interactions.  
* **Image load failures** – If `star.gif` or `delete.gif` are missing, the stars appear blank; no fallback.  
* **`!important` usage** – Can make overriding styles difficult for developers extending the component.

### Potential Enhancements  
1. **SVG or icon fonts** – Replace sprite images with inline SVG or an icon font for scalability and easier theming.  
2. **Flexbox / Grid layout** – Remove `float` and `text-indent` tricks for cleaner, modern code.  
3. **CSS variables** – Expose dimensions (`--star-size`) so the widget can be customized via CSS without editing the stylesheet.  
4. **Accessibility** – Add `role="slider"` or `role="radiogroup"` with appropriate `aria-*` attributes; support keyboard navigation.  
5. **Responsive media queries** – Scale stars on small screens or allow different star sizes.  
6. **Avoid `!important`** – Use proper selector specificity to make overrides easier.  

### Suggested refactor (snippet)  
```css
:root {
  --star-size: 16px;
  --star-gap: 1px;
}

.star-rating,
.rating-cancel {
  float: left;
  width: calc(var(--star-size) + var(--star-gap));
  height: var(--star-size);
  cursor: pointer;
}

.star-rating a,
.rating-cancel a {
  width: var(--star-size);
  height: 100%;
  display: block;
  background-repeat: no-repeat;
}

.star-rating a {
  background-image: url('../img/star.svg'); /* SVG sprite */
  background-position: 0 0;
}
```

Implementing these changes would modernize the component, improve maintainability, and broaden its usability across devices and accessibility standards.

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
