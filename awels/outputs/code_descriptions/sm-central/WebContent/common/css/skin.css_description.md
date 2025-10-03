# skin.css

## Review

## 1. Summary  
The snippet is a pure‑CSS implementation of a **jCarousel** skin named *tango*.  
It defines all visual aspects of the carousel, including container styling, item dimensions, and the navigation buttons for both horizontal and vertical orientations, as well as right‑to‑left (RTL) support. The style sheet relies on PNG sprite images (`next-*.png`, `prev-*.png`) for the navigation icons.

**Key components**  
| Component | Role |
|-----------|------|
| `.jcarousel-skin-tango .jcarousel-container*` | Layout & appearance of the outer carousel wrapper |
| `.jcarousel-skin-tango .jcarousel-clip*` | The viewport that masks the scrolling items |
| `.jcarousel-skin-tango .jcarousel-item*` | Individual carousel items and placeholders |
| `.jcarousel-skin-tango .jcarousel-next/prev-*` | Navigation button styles (horizontal & vertical) |
| RTL classes (`.jcarousel-direction-rtl`) | Swap positioning of navigation for right‑to‑left layouts |

No external frameworks or libraries are required; it is fully standard CSS (with a legacy vendor prefix). The only external resources are the referenced PNG sprites.

---

## 2. Detailed Description  

### Execution Flow  
1. **Initial Render** – When the page loads, the jCarousel JavaScript creates a DOM structure that matches the CSS selectors.  
2. **Styling Application** – The browser applies the styles in the order they appear. Because the selectors are very specific (e.g., `.jcarousel-skin-tango .jcarousel-container-horizontal`), they override any generic defaults.  
3. **Dynamic Interaction** – On user interaction (hover, active, disabled states), the CSS rules for `:hover`, `:active`, and the disabled variants adjust the background positions of the sprite images to show the correct icon state.  
4. **Layout Adjustments** – The `width`/`height` and `padding` settings create a fixed‑size viewport. Items that overflow are clipped by the `.jcarousel-clip-*` element.  

### Design Choices  
- **Fixed‑size viewport** – Hard‑coded pixel values (245×75 for horizontal, 75×245 for vertical) give a predictable layout but limit responsiveness.  
- **Sprite usage** – All button images are packed into single PNGs; background positions change to switch icons, which is efficient for legacy browsers.  
- **Absolute positioning** – Navigation buttons are positioned absolutely inside the container, allowing overlay without affecting item flow.  
- **RTL support** – Small adjustments (changing `left`/`right` and swapping image URLs) ensure the UI works in languages that read right‑to‑left.

### Assumptions & Constraints  
- The carousel must be wrapped in an element that carries the `.jcarousel-skin-tango` class.  
- The referenced PNG sprite files must be in the same directory (or the path must be adjusted).  
- Browser support is targeted at desktop and older mobile browsers; no flexbox or CSS grid is used.  
- The code relies on the `:hover`, `:active`, and `:disabled` pseudo‑classes for interactivity; touch devices may need additional handling.

---

## 3. Functions/Methods (Selector Blocks)  

| Selector | Purpose | Notes |
|----------|---------|-------|
| `.jcarousel-skin-tango .jcarousel-container` | Global container styling (border, background). Uses `-moz-border-radius` for rounded corners (legacy). | Modern browsers support `border-radius` without vendor prefixes. |
| `.jcarousel-skin-tango .jcarousel-direction-rtl` | Applies RTL text direction. | Should also set `text-align: right` if needed. |
| `.jcarousel-skin-tango .jcarousel-container-horizontal` | Sets width/padding for horizontal carousel. | 245 px width matches the clip size; padding adds space for navigation. |
| `.jcarousel-skin-tango .jcarousel-container-vertical` | Same as above but for vertical layout. | Height 245 px, width 75 px. |
| `.jcarousel-skin-tango .jcarousel-clip-horizontal` / `.jcarousel-clip-vertical` | Viewport that clips overflowing items. | Fixed dimensions equal to container minus padding. |
| `.jcarousel-skin-tango .jcarousel-item` | Item size definition (75×75 px). | Each item fits into the clip exactly. |
| `.jcarousel-skin-tango .jcarousel-item-horizontal` / `.jcarousel-item-vertical` | Adds margin to separate items. | Horizontal uses right margin; vertical uses bottom margin. |
| `.jcarousel-skin-tango .jcarousel-item-placeholder` | Style for empty placeholder items. | White background, black text. |
| `*.next-horizontal`, `*.prev-horizontal` | Horizontal navigation button styles (size, position, background). | Uses background sprites; different states adjust background position. |
| `*.next-vertical`, `*.prev-vertical` | Vertical navigation button styles. | Positioned at bottom/right or top/right. |
| Disabled variants (`-disabled-*`) | Grayscale or inactive state styles. | Cursor set to default and background positioned to show disabled icon. |

Each block is self‑contained; no dynamic computation or JavaScript is involved beyond the jCarousel script that toggles the disabled class.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `next-horizontal.png`, `prev-horizontal.png`, `next-vertical.png`, `prev-vertical.png` | Asset | Must be placed in the same folder or path adjusted. |
| `-moz-border-radius` | Legacy vendor prefix | Modern browsers only need `border-radius`. |
| None else | | All other features are standard CSS. |

---

## 5. Additional Notes  

### Edge Cases  
- **Responsiveness** – The hard‑coded dimensions break on small screens; the carousel will overflow or become unusable.  
- **Touch devices** – `:hover` styles may not trigger; consider adding `:active` or a JS fallback for better mobile UX.  
- **Accessibility** – No focus styles or ARIA attributes are provided; keyboard navigation may be limited.  

### Potential Enhancements  
1. **Responsive layout** – Replace fixed pixel sizes with relative units (`rem`, `%`) or use media queries to adjust container/clip dimensions.  
2. **CSS Grid/Flexbox** – Modernize positioning of navigation buttons and items for easier maintenance.  
3. **SVG Sprites** – Switch PNGs to an SVG sprite for sharper scaling and reduced file size.  
4. **Accessibility** – Add focus states, `aria-label`s on navigation buttons, and support for screen readers.  
5. **Vendor‑free border radius** – Remove `-moz-border-radius` in favor of the standard property.  

### Summary  
The stylesheet provides a clear, well‑structured skin for jCarousel. Its design is simple and functional but would benefit from modern CSS techniques to improve responsiveness, accessibility, and maintainability.

## Code Critique



## Code Preview

```css
.jcarousel-skin-tango .jcarousel-container {
    -moz-border-radius: 10px;
    background: #F0F6F9;
    border: 1px solid #346F97;
}

.jcarousel-skin-tango .jcarousel-direction-rtl {
	direction: rtl;
}

.jcarousel-skin-tango .jcarousel-container-horizontal {
    width: 245px;
    padding: 20px 40px;
}

.jcarousel-skin-tango .jcarousel-container-vertical {
    width: 75px;
    height: 245px;
    padding: 40px 20px;
}

.jcarousel-skin-tango .jcarousel-clip-horizontal {
    width:  245px;
    height: 75px;
}

.jcarousel-skin-tango .jcarousel-clip-vertical {
    width:  75px;
    height: 245px;
}

.jcarousel-skin-tango .jcarousel-item {
    width: 75px;
    height: 75px;
}

.jcarousel-skin-tango .jcarousel-item-horizontal {
    margin-left: 0;
    margin-right: 10px;
}

.jcarousel-skin-tango .jcarousel-direction-rtl .jcarousel-item-horizontal {
	margin-left: 10px;
    margin-right: 0;
}

.jcarousel-skin-tango .jcarousel-item-vertical {
    margin-bottom: 10px;
}

.jcarousel-skin-tango .jcarousel-item-placeholder {
    background: #fff;
    color: #000;
}

/**
 *  Horizontal Buttons
 */
.jcarousel-skin-tango .jcarousel-next-horizontal {
    position: absolute;
    top: 43px;
    right: 5px;
    width: 32px;
    height: 32px;
    cursor: pointer;
    background: transparent url(next-horizontal.png) no-repeat 0 0;
}

.jcarousel-skin-tango .jcarousel-direction-rtl .jcarousel-next-horizontal {
    left: 5px;
    right: auto;
    background-image: url(prev-horizontal.png);
}

.jcarousel-skin-tango .jcarousel-next-horizontal:hover {
    background-position: -32px 0;
}

.jcarousel-skin-tango .jcarousel-next-horizontal:active {
    background-position: -64px 0;
}

.jcarousel-skin-tango .jcarousel-next-disabled-horizontal,
.jcarousel-skin-tango .jcarousel-next-disabled-horizontal:hover,
.jcarousel-skin-tango .jcarousel-next-disabled-horizontal:active {
    cursor: default;
    background-position: -96px 0;
}

.jcarousel-skin-tango .jcarousel-prev-horizontal {
    position: absolute;
    top: 43px;
    left: 5px;
    width: 32px;
    height: 32px;
    cursor: pointer;
    background: transparent url(prev-horizontal.png) no-repeat 0 0;
}

.jcarousel-skin-tango .jcarousel-direction-rtl .jcarousel-prev-horizontal {
    left: auto;
    right: 5px;
    background-image: url(next-horizontal.png);
}

.jcarousel-skin-tango .jcarousel-prev-horizontal:hover {
    background-position: -32px 0;
}

.jcarousel-skin-tango .jcarousel-prev-horizontal:active {
    background-position: -64px 0;
}

.jcarousel-skin-tango .jcarousel-prev-disabled-horizontal,
.jcarousel-skin-tango .jcarousel-prev-disabled-horizontal:hover,
.jcarousel-skin-tango .jcarousel-prev-disabled-horizontal:active {
    cursor: default;
    background-position: -96px 0;
}

/**
 *  Vertical Buttons
 */
.jcarousel-skin-tango .jcarousel-next-vertical {
    position: absolute;
    bottom: 5px;
    left: 43px;
    width: 32px;
    height: 32px;
    cursor: pointer;
    background: transparent url(next-vertical.png) no-repeat 0 0;
}

.jcarousel-skin-tango .jcarousel-next-vertical:hover {
    background-position: 0 -32px;
}

.jcarousel-skin-tango .jcarousel-next-vertical:active {
    background-position: 0 -64px;
}

.jcarousel-skin-tango .jcarousel-next-disabled-vertical,
.jcarousel-skin-tango .jcarousel-next-disabled-vertical:hover,
.jcarousel-skin-tango .jcarousel-next-disabled-vertical:active {
    cursor: default;
    background-position: 0 -96px;
}

.jcarousel-skin-tango .jcarousel-prev-vertical {
    position: absolute;
    top: 5px;
    left: 43px;
    width: 32px;
    height: 32px;
    cursor: pointer;
    background: transparent url(prev-vertical.png) no-repeat 0 0;
}

.jcarousel-skin-tango .jcarousel-prev-vertical:hover {
    background-position: 0 -32px;
}

.jcarousel-skin-tango .jcarousel-prev-vertical:active {
    background-position: 0 -64px;
}

.jcarousel-skin-tango .jcarousel-prev-disabled-vertical,
.jcarousel-skin-tango .jcarousel-prev-disabled-vertical:hover,
.jcarousel-skin-tango .jcarousel-prev-disabled-vertical:active {
    cursor: default;
    background-position: 0 -96px;
}



```
