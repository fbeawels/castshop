# jquery.fancybox-1.3.0.css

## Review

## 1. Summary  
The file is a **plain‑CSS stylesheet for FancyBox 1.3.0**, the lightweight jQuery lightbox plugin released in 2010.  
It defines the visual layout of all core UI elements (loading spinner, overlay, wrapper, content, navigation, title, close button, shadows, etc.). The code is heavily reliant on legacy selectors and positioning tricks (e.g., IE6 hacks) to support browsers that were common at the time of its release. There are no external libraries or frameworks used—only CSS (mostly standard, with some vendor prefixes for older browsers).  

**Key components**  
- `#fancybox-loading` – the loading animation.  
- `#fancybox-overlay` – the dark background.  
- `#fancybox-wrap` → `#fancybox-outer` → `#fancybox-inner` – the main container hierarchy.  
- `#fancybox-close` – the close button.  
- `#fancybox-title` – the title area (inside, outside, or over).  
- `#fancybox-left` / `#fancybox-right` – navigation arrows.  
- `div.fancy-bg` – nine‑slice background shadows around the lightbox.  

The stylesheet uses a mix of absolute, fixed and relative positioning to keep the lightbox centered and sized correctly, and relies on `z-index` stacking to overlay it above page content.

---

## 2. Detailed Description  

### 2.1 Execution Flow  
1. **Initialization** – When FancyBox is instantiated, the plugin injects elements with the IDs/classes defined here (e.g., `#fancybox-wrap`, `#fancybox-overlay`).  
2. **Display** –  
   * The `#fancybox-loading` spinner is shown while the content loads.  
   * Once ready, the overlay (`#fancybox-overlay`) fades in, followed by the main wrapper (`#fancybox-wrap`) positioned absolutely at the top‑left of the viewport (with `z-index` 1101).  
3. **Content rendering** – The actual image, iframe, or HTML content is placed inside `#fancybox-inner` (which can grow to the size of the container).  
4. **Navigation** – If a gallery is active, the left/right arrow elements appear; they are hidden by default and revealed on hover.  
5. **Title & close** – The title area appears according to the `fancyboxTitle` setting, and the close button appears in the top‑right corner.  
6. **Cleanup** – When the user clicks the close button or the overlay, the plugin removes the injected elements and restores focus.  

### 2.2 Assumptions & Constraints  
- **Browser support**: Designed for browsers from 2008‑2010, including IE6.  
- **Absolute paths**: All images are referenced relative to the CSS file (`../img/fancybox/...`).  
- **Hard‑coded pixel values**: Many dimensions (e.g., 40 px loader, 32 px close button) are fixed.  
- **z‑index ordering**: Assumes no other elements have z‑indexes above 1104; otherwise, stacking conflicts may occur.  

### 2.3 Architecture & Design Choices  
- **Flat CSS with IDs**: Each component is styled with a unique ID, which makes the rules highly specific but can lead to CSS bloat and difficulty re‑using styles elsewhere.  
- **IE6 hacks**: Uses the `* html` selector to target IE6 for positioning.  
- **Nine‑slice shadows**: Implements drop shadows via nine separate `<div>` elements (`fancy-bg-*`) instead of CSS `box-shadow`.  
- **Fixed vs absolute**: The overlay uses `position:fixed` for modern browsers, but falls back to `absolute` for IE6.  

---

## 3. Functions/Methods  
(While this is a stylesheet, each rule set behaves like a “method” that contributes to the UI.)

| Selector | Purpose | Key Properties |
|----------|---------|----------------|
| `#fancybox-loading` | Spinner container | `fixed`, centered, hidden by default |
| `#fancybox-loading div` | Spinner image | absolute background sprite |
| `#fancybox-overlay` | Dark background | `fixed`, full viewport, hidden |
| `#fancybox-wrap` | Main container | `absolute`, 20 px padding |
| `#fancybox-outer` | Border/background for lightbox | `relative`, white background |
| `#fancybox-inner` | Content holder | `absolute`, overflow hidden |
| `#fancybox-close` | Close button | absolute, 32 px icon |
| `#fancybox-title` | Title wrapper | `absolute`, bottom positioning |
| `.fancybox-title-inside`, `.fancybox-title-outside`, `.fancybox-title-over` | Title styles | background images, font styles |
| `#fancybox-left`, `#fancybox-right` | Navigation areas | `absolute`, full height |
| `#fancybox-left-ico`, `#fancybox-right-ico` | Navigation icons | background images |
| `div.fancy-bg` (and its variants) | 9‑slice shadows | positioned corners/edges with background images |

Each rule set is self‑contained; there are no computed properties or functions. If this were refactored into a modern CSS framework, many of these could be abstracted into mixins or reusable classes.

---

## 4. Dependencies  
| Item | Type | Notes |
|------|------|-------|
| `../img/fancybox/*.png` | Static assets | Must exist relative to the CSS file; otherwise, background images fail. |
| IE6 hacks (`* html`) | Browser specific | Needed only for legacy IE6 support; safe to remove for modern deployments. |
| No external libraries | N/A | All styling is native CSS. |

---

## 5. Additional Notes & Recommendations  

### 5.1 Modernization Opportunities  
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded pixel values** | Inflexible on high‑DPI screens. | Use relative units (`rem`, `em`, `%`) or CSS variables for dimensions. |
| **IE6 hacks** | Adds unnecessary complexity. | Drop `* html` selectors if IE6 support is no longer required. |
| **Nine‑slice shadow divs** | Heavy markup; poor performance. | Replace with `box-shadow` or an SVG drop shadow. |
| **Absolute IDs** | Makes reuse difficult; increases specificity. | Introduce classes (e.g., `.fancybox-wrapper`) and keep IDs for JavaScript hooks only. |
| **Fixed z‑index values** | Risk of stacking context clashes. | Use relative `z-index` values or CSS layering (`z-index: 10;`) and document them. |
| **Background images as sprites** | Requires multiple sprite files; may cause caching issues. | Consider using SVG icons or inline Base64 data URIs for small assets. |

### 5.2 Accessibility & UX  
- The close button is an empty `<div>` with a background image; it should be replaced with a `<button>` or `<a>` element with an `aria-label="Close"` for screen‑reader support.  
- The loader uses a background sprite; providing an `aria-live` region or a spinner element would improve perceived loading time for assistive technologies.  
- Keyboard navigation (e.g., `Esc` to close, arrow keys to navigate galleries) should be documented and styled accordingly.  

### 5.3 Performance & Bloat  
- Many duplicate rules (e.g., `#fancybox-title-wrap` vs. `.fancybox-title-inside`) can be consolidated.  
- Consider minifying the CSS and combining with the plugin’s JavaScript bundle.  
- Load the stylesheet asynchronously to avoid blocking rendering.  

### 5.4 Future Enhancements  
- **Responsive design**: Add media queries to adjust padding, max‑width/height on small screens.  
- **Theme support**: Use CSS custom properties to allow easy theming (color palette, font sizes).  
- **Transition effects**: Modernize the fade/slide animations with CSS transitions or the Web Animations API.  
- **Modular architecture**: Split the stylesheet into logical modules (core, overlay, navigation, title, shadows) and import them as needed.  

---

### Bottom Line  
The stylesheet is a **complete, historically accurate representation** of the original FancyBox 1.3.0 UI. While fully functional for its era, it would benefit significantly from modern CSS practices: removing legacy hacks, simplifying markup, improving accessibility, and embracing responsive design. The changes above are not required for legacy environments but would make the plugin easier to maintain, more performant, and more inclusive for contemporary users.

## Code Critique



## Code Preview

```css
/*
 * FancyBox - jQuery Plugin
 * Simple and fancy lightbox alternative
 *
 * Copyright (c) 20010 Janis Skarnelis
 * Examples and documentation at: http://fancybox.net
 *
 * Version: 1.3.0 (02/02/2010)
 * Requires: jQuery v1.3+
 *
 * Dual licensed under the MIT and GPL licenses:
 *   http://www.opensource.org/licenses/mit-license.php
 *   http://www.gnu.org/licenses/gpl.html
 */
 
#fancybox-loading {
	position: fixed;
	top: 50%;
	left: 50%;
	height: 40px;
	width: 40px;
	margin-top: -20px;
	margin-left: -20px;
	cursor: pointer;
	overflow: hidden;
	background: transparent;
	z-index: 1104;
	display: none;
}

* html #fancybox-loading {	/* IE6 */
	position: absolute;
	margin-top: 0;
}

#fancybox-loading div {
	position: absolute;
	top: 0;
	left: 0;
	width: 40px;
	height: 480px;
	background: transparent url('../img/fancybox/fancy_loading.png') no-repeat;
}

#fancybox-overlay {
	position: fixed;
	top: 0;
	left: 0;
	bottom: 0;
	right: 0;
	background: #000;
	z-index: 1100;
	display: none;
}

* html #fancybox-overlay {	/* IE6 */
	position: absolute;
	width: 100%;
}

#fancybox-tmp {
	padding: 0;
	margin: 0;
	border: 0;
	overflow: auto;
	display: none;
}

#fancybox-wrap {
	position: absolute;
	top: 0;
	left: 0;
	margin: 0;
	padding: 20px;
	z-index: 1101;
	display: none;
}

#fancybox-outer {
	position: relative;
	width: 100%;
	height: 100%;
	background: #FFF;
}

#fancybox-inner {
	position: absolute;
	top: 0;
	left: 0;
	width: 1px;
	height: 1px;
	padding: 0;
	margin: 0;
	outline: none;
	overflow: hidden;
}

#fancybox-hide-sel-frame {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background: transparent;
}

#fancybox-close {
	position: absolute;
	top: -15px;
	right: -15px;
	width: 32px;
	height: 32px;
	background: url('../img/fancybox/fancy_close.png') top left no-repeat;
	cursor: pointer;
	z-index: 1103;
	display: none;
}

#fancybox_error {
	color: #444;
	font: normal 12px/20px Arial;
}

#fancybox-content {
	height: auto;
	width: auto;
	padding: 0;
	margin: 0;
}

#fancybox-img {
	width: 100%;
	height: 100%;
	padding: 0;
	margin: 0;
	border: none;
	outline: none;
	line-height: 0;
	vertical-align: top;
	-ms-interpolation-mode: bicubic;
}

#fancybox-frame {
	position: relative;
	width: 100%;
	height: 100%;
	border: none;
	display: block;
}

#fancybox-title {
	position: absolute;
	bottom: 0;
	left: 0;
	font-family: Arial;
	font-size: 12px;
	z-index: 1102;
}

.fancybox-title-inside {
	padding: 10px 0;
	text-align: center;
	color: #333;
}

.fancybox-title-outside {
	padding-top: 5px;
	color: #FFF;
	text-align: center;
	font-weight: bold;
}

.fancybox-title-over {
	color: #FFF;
	text-align: left;
}

#fancybox-title-over {
	padding: 10px;
	background: url('../img/fancybox/fancy_title_over.png');
	display: block;
}

#fancybox-title-wrap {
	display: inline-block;
}

#fancybox-title-wrap span {
	height: 32px;
	float: left;
}

#fancybox-title-left {
	padding-left: 15px;
	background: transparent url('../img/fancybox/fancy_title_left.png') repeat-x;
}

#fancybox-title-main {
	font-weight: bold;
	line-height: 29px;
	background: transparent url('../img/fancybox/fancy_title_main.png') repeat-x;
	color: #FFF;
}

#fancybox-title-right {
	padding-left: 15px;
	background: transparent url('../img/fancybox/fancy_title_right.png') repeat-x;
}

#fancybox-left, #fancybox-right {
	position: absolute;
	bottom: 0px;
	height: 100%;
	width: 35%;
	cursor: pointer;
	outline: none;
	background-image: url('../img/fancybox/blank.gif');
	z-index: 1102;
	display: none;
}

#fancybox-left {
	left: 0px;
}

#fancybox-right {
	right: 0px;
}

#fancybox-left-ico, #fancybox-right-ico {
	position: absolute;
	top: 50%;
	left: -9999px;
	width: 30px;
	height: 30px;
	margin-top: -15px;
	cursor: pointer;
	z-index: 1102;
	display: block;
}

#fancybox-left-ico {
	background: transparent url('../img/fancybox/fancy_nav_left.png') no-repeat;
}

#fancybox-right-ico {
	background: transparent url('../img/fancybox/fancy_nav_right.png') no-repeat;
}

#fancybox-left:hover, #fancybox-right:hover {
	visibility: visible;    /* IE6 */
}

#fancybox-left:hover span {
	left: 20px;
}

#fancybox-right:hover span {
	left: auto;
	right: 20px;
}

div.fancy-bg {
	position: absolute;
	padding: 0;
	margin: 0;
	border: 0;
	z-index: 1001;
}

div#fancy-bg-n {
	top: -20px;
	left: 0;
	width: 100%;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_n.png') repeat-x;
}

div#fancy-bg-ne {
	top: -20px;
	right: -20px;
	width: 20px;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_ne.png') no-repeat;
}

div#fancy-bg-e {
	top: 0;
	right: -20px;
	height: 100%;
	width: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_e.png') repeat-y;
}

div#fancy-bg-se {
	bottom: -20px;
	right: -20px;
	width: 20px;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_se.png') no-repeat;
}

div#fancy-bg-s {
	bottom: -20px;
	left: 0;
	width: 100%;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_s.png') repeat-x;
}

div#fancy-bg-sw {
	bottom: -20px;
	left: -20px;
	width: 20px;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_sw.png') no-repeat;
}

div#fancy-bg-w {
	top: 0;
	left: -20px;
	height: 100%;
	width: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_w.png') repeat-y;
}

div#fancy-bg-nw {
	top: -20px;
	left: -20px;
	width: 20px;
	height: 20px;
	background: transparent url('../img/fancybox/fancy_shadow_nw.png') no-repeat;
}


```
