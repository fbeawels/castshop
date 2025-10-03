# decotemplate.css

## Review

## 1. Summary
**Purpose & Scope**  
The file is a single‑page stylesheet that defines the visual layout and style of an e‑commerce storefront (likely the front‑end of a catalog or shop). It covers:

- Global typography and basic resets (`html, body`, `p`, `form`, `a`).
- Structural layout (header, footer, columns, sections).
- Component styles (buttons, forms, product tiles, cart, pagination, navigation).
- Visual theming through image sprites (`background-image: url("…")`).
- A minimal drop‑down menu system defined via nested `<ul>` elements.

**Key Components**  
| Section | Role |
|---------|------|
| `body` / `p` | Default fonts and text styling. |
| `.header`, `.logo`, `.storeNav` | Site header and branding. |
| `.mainNav`, `.section-header`, `.tab-*` | Navigation menus and tabbed content. |
| `.product-*`, `.sales-product`, `.list-product` | Product grid and detail view. |
| `.cart-*`, `.checkout-*` | Shopping‑cart and checkout widgets. |
| `.footer` | Bottom footer and copyright. |
| `#menu` hierarchy | A simple horizontal menu with fly‑out sub‑menus. |

**Notable Design Choices**  
- Heavy reliance on **image sprites** for button backgrounds and UI decoration (e.g., `.button-left`, `.input-box1`).  
- Use of **relative paths** (`../../../img/...`) indicating the CSS lives in a nested folder relative to the image assets.  
- The menu system uses **hover/visibility toggling** with inline `display:none`/`display:block` rules instead of JavaScript.  
- There are legacy **IE hacks** (`expression(...)`, `background-position` hacks) and `!important` overrides.  
- No responsive media queries; layout is fixed to 950 px widths.

---

## 2. Detailed Description
### 2.1 Initialization
On document load the browser parses the CSS and applies rules in cascade order. All selectors target specific elements, e.g., `.header` is applied to the `<div>` that wraps the top banner.

### 2.2 Runtime Behavior
The stylesheet is static: it does not change dynamically at runtime. The only interactive behavior comes from CSS hover states:

- `a:hover`, `.mainNav-href:hover`, `.section-header-1stword:hover`, etc., change link colors.
- `#menu > li:hover ul` displays a sub‑menu.
- `.tab-selected` toggles styles for the active tab.

Because the menu uses CSS hover only, it will only function on non-touch devices; touch devices require JavaScript to toggle visibility.

### 2.3 Cleanup
No runtime cleanup is required; the CSS remains in the DOM until the page is unloaded. If the page transitions to a different template or view, the style will persist unless another stylesheet overrides it.

### 2.4 Dependencies & Constraints
- **Image assets**: The stylesheet references a large number of images (`input-box1.gif`, `button1a.gif`, `cart-price1.gif`, etc.). All must be available relative to the CSS path.
- **Browser quirks**: Uses old IE-specific expression syntax for dynamic padding; may break on modern browsers that strip unsupported properties.
- **Legacy CSS**: Some selectors use `float` and explicit widths; no flexbox or grid, so the design is brittle on smaller screens.
- **No external frameworks**: Pure CSS, no reliance on Bootstrap, Foundation, etc.

### 2.5 Architecture & Design Choices
- **Separation of concerns**: Visual styling is split into logical blocks (`header`, `footer`, `product`, `cart`).
- **Component naming**: Uses BEM‑like prefixes (e.g., `product-`, `cart-`) but without block/element separators.
- **Sprite usage**: Consolidates many UI elements into single image files for performance, though this increases maintenance overhead.
- **Global resets**: Minimal; relies on defaults for many elements, which can lead to cross‑browser inconsistencies.

---

## 3. Selectors/“Methods” (Functions)

| Selector | Purpose | Notes |
|----------|---------|-------|
| `html, body` | Base font, colors, background. | Sets `font-family`, `font-size`, `color`, `background`. |
| `p` | Paragraph margin. | Sets top/bottom margin. |
| `form` | Reset form margins. | Ensures no default spacing. |
| `.form-hidden` | Hides elements. | `display:none`. |
| `a` | Cursor pointer and default link style. | Uses `cursor:hand` (old IE). |
| `.body` | Page wrapper background. | Duplicate of body background. |
| `.header`, `.logo`, `.storeNav`, `.storeName` | Header layout and typography. | Uses fixed widths (950 px). |
| `.categoryPath`, `.mainNav`, `.section-header`, `.section-header-1stword` | Navigation breadcrumbs and tab titles. | Uses uppercase and background images for underlines. |
| `.home-banner`, `.banner`, `.main` | Page section wrappers. | Float and padding. |
| `.copyright`, `.footer` | Bottom section styling. | Uses background colors and text styling. |
| `.col-left`, `.col-center`, `.col-right` | Three‑column layout. | Floats with fixed widths. |
| `.section` | Content container. | Padding bottom. |
| `.input-box*`, `.button-left/right`, `.href-button` | Form controls and button sprites. | Uses background images for visual edges. |
| `.advance-search`, `.link`, `.login-*`, `.signup-*` | Search, login, signup UI. | Various sprite backgrounds. |
| `.sales-product`, `.list-product`, `.product-*` | Product tile styling. | Fixed width/height, sprite backgrounds. |
| `.sub-categories`, `.sub-link` | Category listing. | Underline sprite. |
| `.pagination-*` | Pagination controls. | Color changes on hover/selected. |
| `.checkout-info`, `.checkout-basket`, `.checkout-text`, `.checkout-value`, `.checkout-button` | Checkout bar styling. | Sprite backgrounds for button. |
| `.cart-*` | Cart drawer styling. | Multiple sprite layers (`cart1`, `cart2`, `cart3`). |
| `.total-price*`, `.total-label`, `.price-label` | Total price display. | Sprite backgrounds and text styles. |
| `.detail-*` | Product detail page layout. | Fixed width for image and info. |
| `.qty-*` | Quantity selector. | Uses sprite background for input box. |
| `.tab-*` | Tabbed navigation. | Sprite backgrounds and color changes. |
| `.href-selected` | Active link styling. | Underlined text. |
| `#menu`, `#menu > li`, `#menu ul`, `#menu li ul`, `#menu li:hover ul`, etc. | Horizontal menu with fly‑out sub‑menus. | Uses nested `<ul>`; hover reveals sub‑menu. |

*No JavaScript functions are defined in this file; all interactive behavior relies on CSS pseudo‑classes.*

---

## 4. Best‑Practice Assessment & Recommendations

| Area | Current Status | Potential Issues | Suggested Fix |
|------|----------------|------------------|---------------|
| **Font & cursor** | `cursor:hand` (IE only). | Modern browsers ignore; may cause accessibility issues. | Use `cursor:pointer` everywhere. |
| **Legacy IE hacks** (`expression(...)`, `!important`) | Keeps old IE layout but will be stripped by modern browsers. | Breaks on Chrome/Firefox where the `padding-top` becomes 0. | Remove or replace with standard CSS (`padding-top: 0.5em;` etc.). |
| **Fixed widths** (950 px) | Works on large desktops only. | Not mobile‑friendly. | Add `@media` queries or switch to `flexbox`/`grid`. |
| **Sprites** | Improves load time but hurts maintainability. | Any change to sprite image requires updating many selectors. | Consider using SVG or CSS shapes for buttons, or modern `flex`/`grid` icons. |
| **Float‑based layout** | Simple but fragile. | Overflows and clearfix issues. | Add clearfix or switch to flex. |
| **Hover‑only menu** | Not touch‑friendly. | Touch devices ignore `:hover`. | Add minimal JS to toggle sub‑menus or use a mobile‑friendly menu plugin. |
| **Duplicate selectors** (`.body` vs `body`) | Confusing. | Increases risk of conflicting styles. | Consolidate to one rule. |
| **Selector specificity** | High due to many chained selectors. | May lead to unintended overrides. | Use more specific class names or a modular CSS approach. |
| **Color variables** | Colors repeated (`#790000`, `#8dc63f`, etc.). | Hard to maintain consistency. | Define CSS variables (`--primary-color: #8dc63f;`) or use SCSS/LESS for variables. |
| **Accessibility** | Color contrast sometimes low (e.g., `#790000` on light text). | Risk of poor readability. | Verify WCAG contrast ratios. |
| **Images path** | Relative paths assume a particular folder hierarchy. | Can break if the CSS is moved. | Prefer absolute paths or CSS `url()` with root (`/img/...`). |
| **Sprite images missing** | Not evident from CSS alone, but missing images will break UI. | | Verify that all referenced images exist. |

### Quick‑Fix Checklist

1. **Remove old IE expression hacks** – replace `padding-top: expression("4px")` with standard `padding-top: 4px;`.
2. **Normalize cursor** – change `cursor:hand` to `cursor:pointer`.
3. **Add clearfix** – after columns, e.g., `.col-left:after { content:""; clear:both; display:block; }`.
4. **Introduce Flexbox** – for `.header`, `.col-*`, `.footer` to simplify responsive behavior.
5. **Add media queries** – shrink widths or switch to a single‑column layout on `max-width: 768px`.
6. **Centralize color variables** – use `:root` to declare `--primary`, `--secondary`, etc., and reference them.
7. **Limit use of `!important`** – remove or refactor to avoid specificity wars.
8. **Update menu** – fallback to JavaScript for touch devices or replace with a more modern library.

---

## 4. Dependencies & External Resources
- **Image directory**: All sprite files reside under `../../../img/templates/decotemplate/`. Missing any file will cause visual degradation.
- **IE6/7**: Expression padding and `cursor:hand` are not supported in IE10+; they are ignored, which can leave unintended padding or default cursor.
- **CSS resets**: No `box-sizing:border-box` reset; widths and paddings add up, which can cause overflow bugs.
- **No CSS preprocessors**: While the file is pure CSS, many selectors are repeated (e.g., `.product-image`, `.product-name`). Using SCSS/LESS with mixins could reduce duplication.

---

## 5. Overall Evaluation
| Strengths | Weaknesses |
|-----------|------------|
| • Well‑structured, component‑based naming. | • 950 px fixed widths – not responsive. |
| • Extensive use of sprite images for performance. | • Legacy IE hacks that can break on modern browsers. |
| • No external dependencies – easy to bundle. | • Hover‑only menu fails on touch devices. |
| • Clear separation of header, footer, product, cart. | • Lack of modern layout techniques (flexbox/grid). |
| • Detailed typography and button styling. | • No CSS variables – hard to change theme colors. |

---

### Suggested Next Steps

1. **Modernize the layout**  
   Replace `float`/`width` with `display:flex` or CSS grid, making the design fluid and mobile‑first.  
2. **Refactor sprites**  
   Move to inline SVG or CSS shapes for buttons/inputs.  
3. **Add accessibility checks**  
   Run the page through aWCAG tool to ensure contrast ratios meet AA/AAA.  
4. **Encapsulate styling**  
   Adopt BEM conventions (`product__name`, `cart--highlight`) or a CSS framework to reduce duplication.  
5. **Implement responsive breakpoints**  
   Provide alternate styles for `max-width: 768px` and `max-width: 480px`.  
6. **Add JavaScript fallback for the menu**  
   Provide a simple script to toggle sub‑menus on touch or click, ensuring usability across all devices.  

By addressing the above points, the stylesheet will become more maintainable, accessible, and future‑proof while preserving the visual identity of the original design.

## Code Critique



## Code Preview

```css
html, body {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-color: #ffffff;
}


p {
	margin-top: 0px;
	margin-bottom: 15px;
}

form {
	margin: 0px;
	padding: 0px;
}

.form-hidden {
	display: none;
}

a {
	cursor: hand;
	text-decoration: none;
}

a:hover {
	cursor: hand;
}

.body {
	margin: 0px;
	background-color: #ffffff;
}

.topLine {
	margin: 0px;
	float: left;
	width: 100%;
	height: 5px;
	background-color: #464646;
	overflow: hidden;
}

.siteLayout {
	margin: 0px;
	float: left;
	width: 100%;
}

.site {
}

.minWidth {
	width: 1000px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
}

.header {
	width: 950px;
	height: 105px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
}

.logo {
	float: left;
	width: 790px;
}

.logo-image {

	width: 410px;
	overflow: hidden;
}

.storeNav {
	float: left;
	width: 375px;
	margin-top:5px;
	height: 25px;
	font-family: "Times New Roman";
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #a1a1a1;
	overflow:hidden;
}

.storeName {
	font-family: "Times New Roman";
	font-size: 40px;
	font-weight: normal;
	font-style: normal;
	color: #a1a1a1;
	overflow:hidden;
}

.boldText {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}




.categoryPath {
	font-family: "Times New Roman";
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #a1a1a1;
        overflow:hidden;
}

.categoryPath-href {
	color: #a1a1a1;
	text-decoration: none;
}

.categoryPath-href:hover {
	color: #8dc63f;
	text-decoration: none;
}

.mainNav {
	font-family: "Times New Roman";
	font-size: 14px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #a1a1a1;
        overflow:hidden;
}

.mainNav-href {
	color: #a1a1a1;
	text-decoration: none;
}

.mainNav-href:hover {
	color: #8dc63f;
	text-decoration: none;
}

.home-banner {
	width: 950px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
	border-left: 2px solid #cdcdcd;
	border-right: 2px solid #cdcdcd;
	overflow:hidden;
}

.banner {
	width: 730px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
	overflow:hidden;
}

.main {
	width: 950px;
	height: auto;
	margin-left: auto;
	margin-right: auto;
	margin-top: 32px;
	margin-bottom: 0px;
}

.copyright {
	float: left;
	width: 100%;
	background-color: #464646;
}

.copyright-container {
	width: 950px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
	padding: 0px;
	padding-top: 16px;
	padding-bottom: 12px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #ffffff;
}

.copyright-terms {
	width: 950px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
	padding: 0px;
	padding-bottom: 38px;
}

.terms {
	float: left;
	width: 732px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 9px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #a1a1a1;
}

.footer {
	float: left;
	width: 100%;
	background-color: #292929;
}

.footer-container {
	width: 950px;
	margin-left: auto;
	margin-right: auto;
	margin-top: 0px;
	margin-bottom: 0px;
	padding: 0px;
	padding-top: 11px;
	padding-bottom: 11px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #ffffff;
}

.footer-href {
	color: #ffffff;
	text-decoration: none;
}

.footer-href:hover {
	color: #8dc63f;
	text-decoration: none;
}

.home-left {
	float: left;
	width: 732px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.col-left {
	float: left;
	width: 178px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	padding-right: 20px;
}

.col-center {
	float: left;
	width: 534px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.col-right {
	float: right;
	width: 200px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.section-header {
	float: left;
	width: 100%;
	font-family: "Times New Roman";
	font-size: 18px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #a1a1a1;
	background-image: url("../../../img/templates/decotemplate/header-dot.gif");
	background-repeat: repeat-x;
	background-position: left bottom;
	padding-bottom: 6px;
}

.section-header-1stword {
	color: #535353;
}

.section {
	float: left;
	width: 100%;
	padding-bottom: 25px;
}

.line-2px {
	float: left;
	width: 100%;
	padding-top: 3px;
}

.line-3px {
	float: left;
	width: 100%;
	padding-top: 3px;
}

.line-4px {
	float: left;
	width: 100%;
	padding-top: 4px;
}

.line-5px {
	float: left;
	width: 100%;
	padding-top: 5px;
}

.line-8px {
	float: left;
	width: 100%;
	padding-top: 8px;
}

.line-10px {
	float: left;
	width: 100%;
	padding-top: 10px;
}

.line-15px {
	float: left;
	width: 100%;
	padding-top: 15px;
}

.line-20px {
	float: left;
	width: 100%;
	padding-top: 20px;
}

.input-box1 {
	float: left;
	width: 5px;
	height: 26px;
	background-image: url("../../../img/templates/decotemplate/input-box1.gif");
	background-repeat: no-repeat;
}

.input-box2 {
	float: left;
	width: 190px;
	height: 26px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/input-box2.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: expression("6px");
	margin: 0px;
	border: 0px;
}

.input-box3 {
	float: left;
	width: 5px;
	height: 26px;
	background-image: url("../../../img/templates/decotemplate/input-box3.gif");
	background-repeat: no-repeat;
}

.button-left {
	float: left;
	width: auto;
	height: auto;
	white-space: nowrap;
}

.button-right {
	float: right;
	width: auto;
	height: auto;
	white-space: nowrap;
}

.href-button {
	float: left;
	width: auto;
	height: auto;
}

.href-button span.button1-box1 {
	float: left;
	width: 4px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/button1a.gif");
	background-repeat: no-repeat;
}

.href-button span.button1-box2 {
	float: left;
	width: auto;
	height: 20px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #ffffff;
	background-image: url("../../../img/templates/decotemplate/button1b.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: 3px;
	padding-left: 4px;
	padding-right: 4px;
	border: 0px;
	margin: 0px;
	text-decoration: none !important;
}

.href-button span.button1-box2a {
	float: left;
	width: auto;
	height: 20px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #ffffff;
	background-image: url("../../../img/templates/decotemplate/button1b.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: 3px;
	padding-left: 4px;
	padding-right: 4px;
	border: 0px;
	margin: 0px;
	text-decoration: none;
}

.href-button span.button1-box3 {
	float: left;
	width: 4px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/button1c.gif");
	background-repeat: no-repeat;
}

.advance-search {
	float: left;
	width: auto;
	margin-top: 3px;
	margin-right: 10px;
}

.link {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #8dc63f;
	text-decoration: none;
}

.link:hover {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #8dc63f;
	text-decoration: underline;
}

.login-left {
	float: left;
	width: auto;
	font-size: 14px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	margin-top: 4px;
}

.login-right {
	float: right;
	width: auto;
	height: auto;
	white-space: nowrap;
}

.login-box1 {
	float: left;
	width: 5px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/input-box4.gif");
	background-repeat: no-repeat;
}

.login-box2 {
	float: left;
	width: 100px;
	height: 20px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/input-box5.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: expression("3px");
	border: 0px;
	margin: 0px;
}

.login-box3 {
	float: left;
	width: 5px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/input-box6.gif");
	background-repeat: no-repeat;
}

.signup-link {
	text-decoration: none;
}

.signup-link:hover {
	text-decoration: none;
}

.signup-box {
	float: left;
	width: 199px;
	height: 95px;
	background-image: url("../../../img/templates/decotemplate/signup.gif");
	background-repeat: no-repeat;
	text-transform: uppercase;
	cursor: hand;
}

.signup-text {
	position: absolute;
	width: auto;
	margin-top: 23px;
	margin-left: 85px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.signup-word {
	font-family: "Times New Roman";
	font-size: 24px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.signup-start {
	font-size: 10px;
	color: #ffffff;
}

.newsletter-image {
	margin-top: 7px;
	margin-right: 5px;
}

.newsletter {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.sales-product {
	float: left;
	width: 168px;
	height: 227px;
	margin-left: 17px;
	margin-top:10px;
	border: 1px solid #e1e1e1;
	background-color: #e1e1e1;
}

.list-product {
	float: left;
	width: 168px;
	height: 227px;
	margin-left: 11px;
	margin-top:10px;
	border: 1px solid #e1e1e1;
	background-color: #e1e1e1;
}

.product-first {
	margin-left: 0px;
}

.product-image {
	float: left;
	width: 168px;
	height: 140px;
	margin: 0px;
	padding: 0px;
	overflow: hidden;
}

.product-info {
	float: left;
	width: 158px;
	height: auto;
	margin: 0px;
	padding: 5px;
}

.product-title {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: bold;
	font-style: normal;
	color: #535353;
	margin: 0px;
	padding: 0px;
}

.product-price {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	/**font-size: 14px;**/
	font-weight: normal;
	font-style: normal;
	color: #535353;
	margin: 0px;
	margin-top: 3px;
	padding: 0px;
}

.product-value {
	color: #8dc63f;
}

.product-instock {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: bold;
	font-style: normal;
	color: #535353;
	margin: 0px;
	margin-top: 3px;
	padding: 0px;
}

.product-line {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: bold;
	font-style: normal;
	color: #535353;
	margin: 0px;
	margin-top: 3px;
	padding: 0px;
}

.sub-categories {
	float: left;
	width: 100%;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/header-dot.gif");
	background-repeat: repeat-x;
	background-position: left bottom;
	padding-top: 6px;
	padding-bottom: 6px;
}

.sub-link {
	text-decoration: none;
	color: #535353;
}

.sub-link:hover {
	text-decoration: none;
	color: #8dc63f;
}

.pagination-left {
	float: left;
	width: auto;
	font-weight: bold;
}

.pagination-right {
	float: right;
	width: auto;
	color: #a1a1a1;
}

.pagination-selected {
	color: #535353;
	font-weight: bold;
}

.pagination-right a {
	color: #8dc63f;
	text-decoration: none;
}

.pagination-right a:hover {
	color: #8dc63f;
	text-decoration: underline;
}



.checkout-info {
	float: left;
	width: auto;
}

.checkout-basket {
	float: left;
	width: 40px;
}

.checkout-text {
	float: left;
	width: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	text-align: right;
	padding-left: 5px;
	padding-right: 5px;
	/**border-right: 2px solid #a1a1a1;**/
}

.checkout-value {
	font-size: 16px;
	color: #8dc63f;
}

.checkout-item {
	font-size: 16px;
	color: #a1a1a1;
}

.checkout-button {
	float: left;
	width: auto;
	height: auto;
}

.checkout-button span.checkout-box1 {
	float: left;
	width: 5px;
	height: 30px;
	background-image: url("../../../img/templates/decotemplate/checkout1.gif");
	background-repeat: no-repeat;
}

.checkout-button span.checkout-box2 {
	float: left;
	width: auto;
	height: 30px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	text-transform: uppercase;
	color: #ffffff;
	background-image: url("../../../img/templates/decotemplate/checkout2.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: 2px;
	padding-left: 4px;
	padding-right: 4px;
	border: 0px;
	margin: 0px;
	text-align: center;
	text-decoration: none !important;
}

.checkout-button span.checkout-box3 {
	float: left;
	width: 5px;
	height: 30px;
	background-image: url("../../../img/templates/decotemplate/checkout3.gif");
	background-repeat: no-repeat;
}

.checkout-small {
	font-size: 9px;
	font-weight: bold;
}

.cart1 {
	float: left;
	width: 200px;
	height: 5px;
	background-image: url("../../../img/templates/decotemplate/cart1.gif");
	background-repeat: no-repeat;
	padding: 0px;
	margin: 0px;
	overflow: hidden;
}

.cart2 {
	float: left;
	width: 188px;
	height: auto;
	background-image: url("../../../img/templates/decotemplate/cart2.gif");
	background-repeat: repeat-y;
	padding: 1px;
	padding-left: 6px;
	padding-right: 6px;
	margin: 0px;
}

.cart3 {
	float: left;
	width: 200px;
	height: 5px;
	background-image: url("../../../img/templates/decotemplate/cart3.gif");
	background-repeat: no-repeat;
	padding: 0px;
	margin: 0px;
}

.cart-line {
	float: left;
	width: 188px;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/cart-line.gif");
	background-repeat: no-repeat;
	background-position: left top;
	padding: 0px;
	padding-top: 6px;
	padding-bottom: 5px;
}

.cart-first {
	background-image: url("../../../img/templates/decotemplate/spacer.gif");
	padding-top: 1px;
}

.cart-last {
	padding-bottom: 1px;
}

.cart-product {
	float: left;
	width: 50px;
	height: 48px;
	border: 1px solid #e1e1e1;
	overflow: hidden;
}

.cart-info {
	float: right;
	width: 132px;
	height: auto;
}

.cart-name {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: bold;
	font-style: normal;
	color: #535353;
	margin-top: 1px;
}

.cart-price {
	float: left;
	width: 100%;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 14px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	margin-top: 3px;
}

.cart-value {
	color: #8dc63f;
}

.cart-qty {
	float: left;
	width: auto;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	padding-right: 7px;
}

.cart-instock {
	float: left;
	width: auto;
	height: auto;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 10px;
	font-weight: bold;
	font-style: normal;
	color: #535353;
}

.total-price {
	float: left;
	width: 100%;
	height: 27px;
}

.total-price1 {
	float: left;
	width: 5px;
	height: 27px;
	background-image: url("../../../img/templates/decotemplate/cart-price1.gif");
	background-repeat: no-repeat;
}

.total-price2 {
	float: left;
	width: 189px;
	height: 27px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 18px;
	font-weight: normal;
	font-style: normal;
	color: #e1e1e1;
	background-image: url("../../../img/templates/decotemplate/cart-price2.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: 2px;
	border: 0px;
	text-align: right;
	padding-right: 1px;
}

.total-price3 {
	float: left;
	width: 5px;
	height: 27px;
	background-image: url("../../../img/templates/decotemplate/cart-price3.gif");
	background-repeat: no-repeat;
}

.total-label {
	font-size: 14px;
}

.price-label {
	color: #8dc63f;
}

.out-of-stock {
	color: #790000;
}

.detail-image {
	float: left;
	width: 168px;
	height: 140px;
	margin: 0px;
	padding: 0px;
	border: 1px solid #e1e1e1;
	overflow: hidden;
}

.detail-box {
	float: right;
	width: 310px;
	margin: 0px;
	padding: 0px;
}

.detail-title {
	float: left;
	width: 100%;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 18px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.detail-info {
	float: left;
	width: 100%;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 12px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	margin-top: 20px;
}

.detail-price {
	font-size: 18px;
}

.usual-price {
	color: #790000;
	padding-left: 38px;
}

.price-strike {
	text-decoration: line-through;
}

.qty {
	float: left;
	width: 100%;
	margin-top: 5px;
}

.qty-label {
	float: left;
	width: 38px;
}

.qty-box1 {
	float: left;
	width: 5px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/input-box4.gif");
	background-repeat: no-repeat;
}

.qty-box2 {
	float: left;
	width: 20px;
	height: 20px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/input-box5.gif");
	background-repeat: repeat-x;
	padding: 0px;
	padding-top: expression("4px");
	margin: 0px;
	border: 0px;
	text-align: right;
}

.qty-box3 {
	float: left;
	width: 5px;
	height: 20px;
	background-image: url("../../../img/templates/decotemplate/input-box6.gif");
	background-repeat: no-repeat;
}

.qty-button {
	float: left;
	width: auto;
	padding-left: 4px;
}

.tab-top {
	float: left;
	width: 530px;
	border-left: 1px solid #ababab;
}

.tab-box {
	float: left;
	width: auto;
	height: 13px;
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
	background-image: url("../../../img/templates/decotemplate/tab.gif");
	background-repeat: no-repeat;
	background-color: #ffffff;
	padding-top: 4px;
	padding-left: 15px;
	padding-right: 15px;
	padding-bottom: 4px;
	border-top: 1px solid #ababab;
	border-right: 1px solid #ababab;
}

.tab-box a {
	color: #535353;
	text-decoration: none;
}

.tab-box a:hover {
	color: #8dc63f;
	text-decoration: none;
}

.tab-selected {
	font-weight: bold;
	color: #ffffff;
	background-image: url("../../../img/templates/decotemplate/spacer.gif");
	background-repeat: no-repeat;
	background-color: #666666;
}

.tab-selected a {
	color: #ffffff;
	text-decoration: none;
}

.tab-selected a:hover {
	color: #8dc63f;
	text-decoration: none;
}


.tab-bottom {
	float: left;
	width: 500px;
	height: auto;
	padding: 15px;
	border: 1px solid #ababab;
}

.tab-content {
	font-family: Tahoma, Helvetica, Arial;
	font-size: 11px;
	font-weight: normal;
	font-style: normal;
	color: #535353;
}

.tab-content p {
	margin-top: 0px;
	margin-bottom: 15px;
}

.href-selected {
	color: #8dc63f;
	text-decoration: underline;
}

.href-selected:hover {
	color: #8dc63f;
	text-decoration: none;
}

/** MENU LANGUAGE **/

        #menu{ margin:0px; padding:0px; list-style:none; color:#fff; line-height:35px; display:inline-block; float:left; z-index:1000; }
        #menu a { color:#fff; text-decoration:none; }
	  #menu > li {
		background-image: url("../../../img/templates/decotemplate/sm_line.png"); 
		background-repeat: repeat-x;
		cursor:pointer;
		float:left; 
		position:relative;
		padding:0px 12px;
		width: auto;
		font-family: Tahoma, Helvetica, Arial;
		font-size: 12px;
		font-weight: normal;
		font-style: normal;
		border: 0px;
		margin: 0px;
		text-decoration: none;
	   }
	  #menu > li a:hover {color:#ffffff;}
        /* sub-menus*/
        #menu ul { padding:0px; margin:0px; display:block; display:inline;}
        #menu li ul { position:absolute; left:10px; top:0px; margin-top:35px; width:104px; line-height:16px; background-color:#ffffff; color:#0395CC; /* for IE */ display:none; }
	  #menu li:hover ul { display:block;}
        #menu li ul li{ display:block; margin:5px 20px; padding: 5px 0px;  border-top: dotted 1px #606060; list-style-type:none; }
        #menu li ul li:first-child { border-top: none; }
        #menu li ul li a { display:block; color:#a1a1a1; }
        #menu li ul li a:hover { color:#8dc63f; }
        /* main submenu */

        /* corners*/
        #menu .corner_inset_left { position:absolute; top:0px; left:-12px;}
        #menu .corner_inset_right { position:absolute; top:0px; left:102px;}
        #menu .last { background:transparent none repeat scroll 0% 0%; margin:0px; padding:0px; border:none; position:relative; border:none; height:0px;}
        #menu .corner_left { position:absolute; left:0px; top:0px;}
        #menu .corner_right { position:absolute; left:96px; top:0px;}
        #menu .middle { position:absolute; left:18px; height: 20px; width: 110px; top:0px;}


```
