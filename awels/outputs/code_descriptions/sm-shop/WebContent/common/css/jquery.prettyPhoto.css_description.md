# jquery.prettyPhoto.css

## Review

## 1. Summary

The supplied code is a stylesheet that defines the visual appearance of the **PrettyPhoto** light‑box plugin for several themes (light_rounded, dark_rounded, dark_square, light_square, facebook). It contains:

* **Theme‑specific styling** – background images, colors, and layout for each theme.
* **Component styling** – generic classes such as `.pp_overlay`, `.pp_pic_holder`, `.pp_content`, navigation arrows, etc.
* **Utility classes** – clearfix, focus handling, and IE6 hacks.

The stylesheet relies on external PNG/GIF assets stored under `../img/prettyPhoto/[theme]/`. It uses classic CSS techniques (sprite images, absolute positioning, text‑indent trickery) to create a lightweight UI without heavy JavaScript or modern layout systems.

---

## 2. Detailed Description

### 2.1 Core components

| Component | Role | Key CSS rules |
|-----------|------|---------------|
| `.pp_overlay` | Dark overlay covering the page | `background:#000; display:none;` |
| `.pp_pic_holder` | Main container for the modal | `display:none; position:absolute; width:100px;` |
| `.pp_top`, `.pp_middle`, `.pp_right` | Header area (background images, color) | Various sprite background settings |
| `.pp_content` | Body of the modal (image/text) | `height:40px;` |
| `.pp_nav` | Navigation arrows & pagination | `float:left;` |
| `.pp_loaderIcon` | Loading spinner | `background:url(.../loader.gif)` |
| `.pp_expand`, `.pp_contract`, `.pp_close` | Control buttons | `cursor:pointer;` |
| `.pp_description` | Caption text | `display:none;` |

### 2.2 Execution flow

1. **Plugin initialization** – The plugin inserts `<div class="pp_overlay">` and `<div class="pp_pic_holder">` into the DOM when an image or content link is clicked.
2. **Theme application** – Based on the chosen theme (`light_rounded`, `dark_square`, etc.), the corresponding CSS rules are applied via the parent `<div>` class. For example, `<div class="light_rounded pp_pic_holder">`.
3. **UI rendering** – CSS handles positioning, background images, and interactivity (e.g., hover states) via pseudo‑classes.
4. **Cleanup** – When the modal is closed, `display:none;` hides the overlay and container; the DOM nodes are usually removed or left behind.

### 2.3 Assumptions & Constraints

* **Static assets** – All images are referenced by relative paths; the directory structure must match the CSS expectations.
* **Browser support** – The code includes hacks for IE6 (`* html`, `display:inline-block` for `clearfix`). Modern browsers will ignore these.
* **Fixed dimensions** – Many elements use fixed pixel values (`height:20px;`, `width:100px;`), limiting responsiveness.
* **Legacy CSS techniques** – Sprites, text‑indent for accessibility, and `display:none` for hover states indicate this stylesheet is aimed at older browsers (pre‑HTML5).

---

## 3. Functions/Methods

As this is pure CSS, there are no functions, but we can list the key selectors and their responsibilities:

| Selector | Purpose | Notable properties |
|----------|---------|--------------------|
| `div.pp_overlay` | Page mask | `background:#000;` |
| `div.pp_pic_holder` | Modal wrapper | `position:absolute;` |
| `.pp_top .pp_left / .pp_middle / .pp_right` | Header tiles | Background images from sprite |
| `.pp_content` | Main content container | `height:40px;` |
| `.pp_content .ppt` | Title text | `color:#000;` |
| `.pp_content_container .pp_left / .pp_right` | Edge padding | `background:#fff;` |
| `.pp_next:hover`, `.pp_previous:hover` | Hover states for navigation | `cursor:pointer;` |
| `.pp_expand / .pp_contract` | Zoom in/out button | `cursor:pointer;` |
| `.pp_close` | Close button | `cursor:pointer;` |
| `.pp_loaderIcon` | Loading indicator | `background:url(.../loader.gif);` |
| `.pp_arrow_previous / .pp_arrow_next` | Navigation arrows | `background:url(.../sprite.png);` |
| `.pp_nav p` | Page counter text | Font styling |
| `.clearfix` | Clearfix hack | `content: ".";` etc. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `../img/prettyPhoto/*` | Asset files (PNG, GIF) | Must be present relative to CSS location |
| Browser CSS features | Standard | Uses `background`, `float`, `position`, `text-indent`, `:hover` |
| IE6 hacks (`* html`) | Legacy | Only affects IE6, ignored by modern browsers |
| No external libraries | — | Pure CSS, no SASS/LESS pre‑processor shown |

---

## 5. Additional Notes

### 5.1 Strengths

* **Separation of concerns** – Themes are isolated into different selector prefixes (`light_rounded`, `dark_rounded`, etc.), making it easy to switch themes by changing a single class on the container.
* **Sprite usage** – Reduces HTTP requests, improving load time on older browsers.
* **Hover states and cursors** – Enhances interactivity without JavaScript.

### 5.2 Weaknesses & Edge Cases

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Fixed dimensions** | Not responsive; will break on mobile | Replace pixel values with relative units (`rem`, `%`) or use media queries to adjust layout. |
| **IE6 hacks** | Code is stale; may cause unnecessary overrides on modern browsers | Remove or guard with `[if lte IE 6]` conditional comments. |
| **Hard‑coded image paths** | Fragile if the asset directory changes | Use CSS variables or a build step to inject correct paths. |
| **Missing vendor prefixes** | Not needed for modern browsers, but for legacy support they might be required | Add prefixes if targeting IE9+ or older Android browsers. |
| **Hover selectors on disabled arrows** | They still consume `background-position` adjustments | Consider disabling the `pointer-events` property in modern browsers. |
| **No `:focus` styling for keyboard navigation** | Accessibility could be improved | Add visible focus outlines for interactive elements. |
| **Duplicated rules across themes** | Violates DRY; maintenance overhead | Factor out common rules into a base class (`.pp_top .pp_left` etc.) and override theme specifics. |

### 5.3 Potential Enhancements

1. **Responsive Design** – Use CSS Grid/Flexbox for layout, replace absolute positioning, and media queries to adapt the modal to various screen sizes.
2. **Theming via CSS Variables** – Define theme colors, font sizes, and sprite URLs as custom properties, enabling dynamic theme switching with minimal CSS changes.
3. **Modern CSS Features** – Replace sprite backgrounds with SVG icons or CSS masks for scalability.
4. **Accessibility** – Add `role="dialog"`, `aria-modal="true"`, and visible focus styles; ensure the close button is keyboard‑accessible.
5. **Performance** – Preload the most commonly used images (e.g., close button, arrows) or inline small icons as data URLs.
6. **Modularization** – Convert to SCSS/SASS to share mixins for shared layout patterns (`.pp_left`, `.pp_right`) and reduce duplication.

---

### 6. Quick Refactor Example (SCSS)

```scss
$theme-path: "../img/prettyPhoto";

@mixin sprite($img, $x, $y, $w: auto, $h: auto) {
  background: url("#{$theme-path}/#{$img}") $x $y no-repeat;
  width: $w;
  height: $h;
}

.pp_overlay {
  background: #000;
  display: none;
}

.pp_pic_holder {
  display: none;
  position: absolute;
  width: 100px; // consider responsive fallback
}

%corner {
  position: absolute;
  height: 20px;
}

.pp_top .pp_left  { @extend %corner; left: 0; width: 20px; }
.pp_top .pp_middle{ @extend %corner; left: 20px; right: 20px; }
.pp_top .pp_right { @extend %corner; right: 0; width: 20px; }

.light_rounded { @include sprite('light_rounded/sprite.png', -88px, -53px); }
// etc.

```

By using SCSS mixins, you can dramatically reduce duplication and centralise sprite handling.

---

## Conclusion

The stylesheet provides a solid baseline for PrettyPhoto’s classic themes. However, it is heavily oriented toward legacy browsers and lacks responsiveness or modern CSS practices. A refactor that introduces CSS variables, modular mixins, and responsive units would greatly improve maintainability, accessibility, and future‑proof the plugin’s UI.

## Code Critique



## Code Preview

```css
 div.light_rounded .pp_top .pp_left{background: url(../img/prettyPhoto/light_rounded/sprite.png) -88px -53px no-repeat;}div.light_rounded .pp_top .pp_middle{background:#fff;}div.light_rounded .pp_top .pp_right{background: url(../img/prettyPhoto/light_rounded/sprite.png) -110px -53px no-repeat;}div.light_rounded .pp_content .ppt{color:#000;}div.light_rounded .pp_content_container .pp_left,div.light_rounded .pp_content_container .pp_right{background:#fff;}div.light_rounded .pp_content{background-color:#fff;}div.light_rounded .pp_next:hover{background: url(../img/prettyPhoto/light_rounded/btnNext.png) center right no-repeat;cursor: pointer;}div.light_rounded .pp_previous:hover{background: url(../img/prettyPhoto/light_rounded/btnPrevious.png) center left no-repeat;cursor: pointer;}div.light_rounded .pp_expand{background: url(../img/prettyPhoto/light_rounded/sprite.png) -31px -26px no-repeat;cursor: pointer;}div.light_rounded .pp_expand:hover{background: url(../img/prettyPhoto/light_rounded/sprite.png) -31px -47px no-repeat;cursor: pointer;}div.light_rounded .pp_contract{background: url(../img/prettyPhoto/light_rounded/sprite.png) 0 -26px no-repeat;cursor: pointer;}div.light_rounded .pp_contract:hover{background: url(../img/prettyPhoto/light_rounded/sprite.png) 0 -47px no-repeat;cursor: pointer;}div.light_rounded .pp_close{width:75px;height:22px;background: url(../img/prettyPhoto/light_rounded/sprite.png) -1px -1px no-repeat;cursor: pointer;}div.light_rounded #pp_full_res .pp_inline{color:#000;}div.light_rounded .pp_arrow_previous{background: url(../img/prettyPhoto/light_rounded/sprite.png) 0 -71px no-repeat;}div.light_rounded .pp_arrow_previous.disabled{background-position:0 -87px;cursor:default;}div.light_rounded .pp_arrow_next{background: url(../img/prettyPhoto/light_rounded/sprite.png) -22px -71px no-repeat;}div.light_rounded .pp_arrow_next.disabled{background-position: -22px -87px;cursor:default;}div.light_rounded .pp_bottom .pp_left{background: url(../img/prettyPhoto/light_rounded/sprite.png) -88px -80px no-repeat;}div.light_rounded .pp_bottom .pp_middle{background:#fff;}div.light_rounded .pp_bottom .pp_right{background: url(../img/prettyPhoto/light_rounded/sprite.png) -110px -80px no-repeat;}div.light_rounded .pp_loaderIcon{background: url(../img/prettyPhoto/light_rounded/loader.gif) center center no-repeat;}div.dark_rounded .pp_top .pp_left{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -88px -53px no-repeat;}div.dark_rounded .pp_top .pp_middle{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top left repeat;}div.dark_rounded .pp_top .pp_right{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -110px -53px no-repeat;}div.dark_rounded .pp_content_container .pp_left{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top left repeat-y;}div.dark_rounded .pp_content_container .pp_right{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top right repeat-y;}div.dark_rounded .pp_content{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top left repeat;}div.dark_rounded .pp_next:hover{background: url(../img/prettyPhoto/dark_rounded/btnNext.png) center right no-repeat;cursor: pointer;}div.dark_rounded .pp_previous:hover{background: url(../img/prettyPhoto/dark_rounded/btnPrevious.png) center left no-repeat;cursor: pointer;}div.dark_rounded .pp_expand{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -31px -26px no-repeat;cursor: pointer;}div.dark_rounded .pp_expand:hover{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -31px -47px no-repeat;cursor: pointer;}div.dark_rounded .pp_contract{background: url(../img/prettyPhoto/dark_rounded/sprite.png) 0 -26px no-repeat;cursor: pointer;}div.dark_rounded .pp_contract:hover{background: url(../img/prettyPhoto/dark_rounded/sprite.png) 0 -47px no-repeat;cursor: pointer;}div.dark_rounded .pp_close{width:75px;height:22px;background: url(../img/prettyPhoto/dark_rounded/sprite.png) -1px -1px no-repeat;cursor: pointer;}div.dark_rounded .currentTextHolder{color:#c4c4c4;}div.dark_rounded .pp_description{color:#fff;}div.dark_rounded #pp_full_res .pp_inline{color:#fff;}div.dark_rounded .pp_arrow_previous{background: url(../img/prettyPhoto/dark_rounded/sprite.png) 0 -71px no-repeat;}div.dark_rounded .pp_arrow_previous.disabled{background-position:0 -87px;cursor:default;}div.dark_rounded .pp_arrow_next{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -22px -71px no-repeat;}div.dark_rounded .pp_arrow_next.disabled{background-position: -22px -87px;cursor:default;}div.dark_rounded .pp_bottom .pp_left{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -88px -80px no-repeat;}div.dark_rounded .pp_bottom .pp_middle{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top left repeat;}div.dark_rounded .pp_bottom .pp_right{background: url(../img/prettyPhoto/dark_rounded/sprite.png) -110px -80px no-repeat;}div.dark_rounded .pp_loaderIcon{background: url(../img/prettyPhoto/dark_rounded/loader.gif) center center no-repeat;}div.dark_square .pp_left ,div.dark_square .pp_middle,div.dark_square .pp_right,div.dark_square .pp_content{background: url(../img/prettyPhoto/dark_square/contentPattern.png) top left repeat;}div.dark_square .currentTextHolder{color:#c4c4c4;}div.dark_square .pp_description{color:#fff;}div.dark_square .pp_loaderIcon{background: url(../img/prettyPhoto/dark_rounded/loader.gif) center center no-repeat;}div.dark_square .pp_content_container .pp_left{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top left repeat-y;}div.dark_square .pp_content_container .pp_right{background: url(../img/prettyPhoto/dark_rounded/contentPattern.png) top right repeat-y;}div.dark_square .pp_expand{background: url(../img/prettyPhoto/dark_square/sprite.png) -31px -26px no-repeat;cursor: pointer;}div.dark_square .pp_expand:hover{background: url(../img/prettyPhoto/dark_square/sprite.png) -31px -47px no-repeat;cursor: pointer;}div.dark_square .pp_contract{background: url(../img/prettyPhoto/dark_square/sprite.png) 0 -26px no-repeat;cursor: pointer;}div.dark_square .pp_contract:hover{background: url(../img/prettyPhoto/dark_square/sprite.png) 0 -47px no-repeat;cursor: pointer;}div.dark_square .pp_close{width:75px;height:22px;background: url(../img/prettyPhoto/dark_square/sprite.png) -1px -1px no-repeat;cursor: pointer;}div.dark_square #pp_full_res .pp_inline{color:#fff;}div.dark_square .pp_arrow_previous{background: url(../img/prettyPhoto/dark_square/sprite.png) 0 -71px no-repeat;}div.dark_square .pp_arrow_previous.disabled{background-position:0 -87px;cursor:default;}div.dark_square .pp_arrow_next{background: url(../img/prettyPhoto/dark_square/sprite.png) -22px -71px no-repeat;}div.dark_square .pp_arrow_next.disabled{background-position: -22px -87px;cursor:default;}div.dark_square .pp_next:hover{background: url(../img/prettyPhoto/dark_square/btnNext.png) center right no-repeat;cursor: pointer;}div.dark_square .pp_previous:hover{background: url(../img/prettyPhoto/dark_square/btnPrevious.png) center left no-repeat;cursor: pointer;}div.light_square .pp_left ,div.light_square .pp_middle,div.light_square .pp_right,div.light_square .pp_content{background:#fff;}div.light_square .pp_content .ppt{color:#000;}div.light_square .pp_expand{background: url(../img/prettyPhoto/light_square/sprite.png) -31px -26px no-repeat;cursor: pointer;}div.light_square .pp_expand:hover{background: url(../img/prettyPhoto/light_square/sprite.png) -31px -47px no-repeat;cursor: pointer;}div.light_square .pp_contract{background: url(../img/prettyPhoto/light_square/sprite.png) 0 -26px no-repeat;cursor: pointer;}div.light_square .pp_contract:hover{background: url(../img/prettyPhoto/light_square/sprite.png) 0 -47px no-repeat;cursor: pointer;}div.light_square .pp_close{width:75px;height:22px;background: url(../img/prettyPhoto/light_square/sprite.png) -1px -1px no-repeat;cursor: pointer;}div.light_square #pp_full_res .pp_inline{color:#000;}div.light_square .pp_arrow_previous{background: url(../img/prettyPhoto/light_square/sprite.png) 0 -71px no-repeat;}div.light_square .pp_arrow_previous.disabled{background-position:0 -87px;cursor:default;}div.light_square .pp_arrow_next{background: url(../img/prettyPhoto/light_square/sprite.png) -22px -71px no-repeat;}div.light_square .pp_arrow_next.disabled{background-position: -22px -87px;cursor:default;}div.light_square .pp_next:hover{background: url(../img/prettyPhoto/light_square/btnNext.png) center right no-repeat;cursor: pointer;}div.light_square .pp_previous:hover{background: url(../img/prettyPhoto/light_square/btnPrevious.png) center left no-repeat;cursor: pointer;}div.facebook .pp_top .pp_left{background: url(../img/prettyPhoto/facebook/sprite.png) -88px -53px no-repeat;}div.facebook .pp_top .pp_middle{background: url(../img/prettyPhoto/facebook/contentPatternTop.png) top left repeat-x;}div.facebook .pp_top .pp_right{background: url(../img/prettyPhoto/facebook/sprite.png) -110px -53px no-repeat;}div.facebook .pp_content .ppt{color:#000;}div.facebook .pp_content_container .pp_left{background: url(../img/prettyPhoto/facebook/contentPatternLeft.png) top left repeat-y;}div.facebook .pp_content_container .pp_right{background: url(../img/prettyPhoto/facebook/contentPatternRight.png) top right repeat-y;}div.facebook .pp_content{background:#fff;}div.facebook .pp_expand{background: url(../img/prettyPhoto/facebook/sprite.png) -31px -26px no-repeat;cursor: pointer;}div.facebook .pp_expand:hover{background: url(../img/prettyPhoto/facebook/sprite.png) -31px -47px no-repeat;cursor: pointer;}div.facebook .pp_contract{background: url(../img/prettyPhoto/facebook/sprite.png) 0 -26px no-repeat;cursor: pointer;}div.facebook .pp_contract:hover{background: url(../img/prettyPhoto/facebook/sprite.png) 0 -47px no-repeat;cursor: pointer;}div.facebook .pp_close{width:22px;height:22px;background: url(../img/prettyPhoto/facebook/sprite.png) -1px -1px no-repeat;cursor: pointer;}div.facebook #pp_full_res .pp_inline{color:#000;}div.facebook .pp_loaderIcon{background: url(../img/prettyPhoto/facebook/loader.gif) center center no-repeat;}div.facebook .pp_nav .pp_arrow_previous{background: url(../img/prettyPhoto/facebook/sprite.png) 0 -71px no-repeat;height:22px;margin-top:0;width:22px;}div.facebook .pp_arrow_previous.disabled{background-position:0 -96px;cursor:default;}div.facebook .pp_nav .pp_arrow_next{background: url(../img/prettyPhoto/facebook/sprite.png) -32px -71px no-repeat;height:22px;margin-top:0;width:22px;}div.facebook .pp_arrow_next.disabled{background-position: -32px -96px;cursor:default;}div.facebook .pp_nav{margin-top:0;}div.facebook .pp_nav p{font-size:15px;padding:0 3px 0 4px;}div.facebook .pp_next:hover{background: url(../img/prettyPhoto/facebook/btnNext.png) center right no-repeat;cursor: pointer;}div.facebook .pp_previous:hover{background: url(../img/prettyPhoto/facebook/btnPrevious.png) center left no-repeat;cursor: pointer;}div.facebook .pp_bottom .pp_left{background: url(../img/prettyPhoto/facebook/sprite.png) -88px -80px no-repeat;}div.facebook .pp_bottom .pp_middle{background: url(../img/prettyPhoto/facebook/contentPatternBottom.png) top left repeat-x;}div.facebook .pp_bottom .pp_right{background: url(../img/prettyPhoto/facebook/sprite.png) -110px -80px no-repeat;}div.pp_pic_holder a:focus{outline:none;}div.pp_overlay{background:#000;display: none;left:0;position:absolute;top:0;width:100%;z-index:9500;}div.pp_pic_holder{display: none;position:absolute;width:100px;z-index:10000;}.pp_top{height:20px;position: relative;}* html .pp_top{padding:0 20px;}.pp_top .pp_left{height:20px;left:0;position:absolute;width:20px;}.pp_top .pp_middle{height:20px;left:20px;position:absolute;right:20px;}* html .pp_top .pp_middle{left:0;position: static;}.pp_top .pp_right{height:20px;left:auto;position:absolute;right:0;top:0;width:20px;}.pp_content{height:40px;}.pp_content .ppt{left:auto;margin-bottom:5px;position: relative;top:auto;}.pp_fade{display: none;}.pp_content_container{position: relative;text-align: left;width:100%;}.pp_content_container .pp_left{padding-left:20px;}.pp_content_container .pp_right{padding-right:20px;}.pp_content_container .pp_details{margin:10px 0 2px 0;}.pp_description{display: none;margin:0 0 5px 0;}.pp_nav{clear: left;float: left;margin:3px 0 0 0;}.pp_nav p{float: left;margin:2px 4px;}.pp_nav a.pp_arrow_previous,.pp_nav a.pp_arrow_next{display:block;float: left;height:15px;margin-top:3px;overflow: hidden;text-indent: -10000px;width:14px;}.pp_hoverContainer{position:absolute;top:0;width:100%;z-index:2000;}a.pp_next{background: url(../img/prettyPhoto/light_rounded/btnNext.png) 10000px 10000px no-repeat;display:block;float: right;height:100%;text-indent: -10000px;width:49%;}a.pp_previous{background: url(../img/prettyPhoto/light_rounded/btnNext.png) 10000px 10000px no-repeat;display:block;float: left;height:100%;text-indent: -10000px;width:49%;}a.pp_expand,a.pp_contract{cursor: pointer;display: none;height:20px;position:absolute;right:30px;text-indent: -10000px;top:10px;width:20px;z-index:20000;}a.pp_close{display:block;float: right;text-indent: -10000px;}.pp_bottom{height:20px;position: relative;}* html .pp_bottom{padding:0 20px;}.pp_bottom .pp_left{height:20px;left:0;position:absolute;width:20px;}.pp_bottom .pp_middle{height:20px;left:20px;position:absolute;right:20px;}* html .pp_bottom .pp_middle{left:0;position: static;}.pp_bottom .pp_right{height:20px;left:auto;position:absolute;right:0;top:0;width:20px;}.pp_loaderIcon{display:block;height:24px;left:50%;margin: -12px 0 0 -12px;position:absolute;top:50%;width:24px;}#pp_full_res{line-height:1 !important;}#pp_full_res .pp_inline{text-align: left;}#pp_full_res .pp_inline p{margin:0 0 15px 0;}div.ppt{color:#fff;display: none;font-size:17px;left:0;position:absolute;top:0;z-index:9999;}.clearfix:after{content: ".";display:block;height:0;clear:both;visibility: hidden;}.clearfix {display: inline-block;}* html .clearfix {height:1%;}.clearfix {display:block;}


```
