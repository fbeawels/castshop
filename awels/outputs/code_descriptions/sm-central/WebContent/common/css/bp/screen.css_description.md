# screen.css

## Review

## 1. Summary
- **Purpose & Scope**  
  The stylesheet is a *base reset and grid system* implementation, largely based on the classic **960‑Grid** layout.  
  It normalizes browser defaults, defines typography, forms, tables, and a 24‑column grid layout with helpers for margins, paddings, floats, and visibility.  
- **Key Components**  
  - **Element Reset** – zeroes out margins/padding, normalizes font properties for all common HTML tags.  
  - **Typography** – sets global font families, sizes, line heights, and provides utility classes such as `.small`, `.large`, `.quiet`, `.loud`, `.highlight`, `.added`, `.removed`.  
  - **Grid System** – `.container`, `.span‑n`, `.column`, and helper classes (`.pull‑n`, `.push‑n`, `.append‑n`, `.prepend‑n`) provide a flexible 24‑column layout with configurable gutter widths.  
  - **Utilities** – clearing floats (`.clearfix`), visibility (`.hide`), and form styling (`input`, `textarea`, `select`, `.error`, `.notice`, `.success`).  
- **Design Patterns & Libraries**  
  - Uses **BEM‑like** naming for utility classes (e.g., `.pull‑3`, `.push‑5`).  
  - The grid follows the **“12‑column”** (here 24‑column) **box‑model** approach used by many CSS frameworks.  
  - No external dependencies; everything is pure CSS.

---

## 2. Detailed Description
### Execution Flow
1. **Reset** – Applied to all elements (`html, body, div, span, …`) to ensure a consistent baseline across browsers.  
2. **Base Styles** – Body text size, line height, font family, and link styling are declared.  
3. **Semantic Elements** – Headings (`h1`–`h6`) get distinct font sizes, margins, and colors.  
4. **Structural Utilities** – `.container`, `.column`, `.span‑n`, `.pull‑n`, `.push‑n`, etc. are defined next, allowing content to be wrapped in a grid.  
5. **Forms & Tables** – Input styles, error messages, and table layouts are added after grid utilities.  
6. **Miscellaneous** – Utility classes for visibility, margins, floats, and form feedback (`.error`, `.notice`, `.success`).  

### Interactions
- **Grid Columns** – Each `.span‑n` is a floated `div` that occupies a percentage width (`n * 40px` + gutters).  
- **Pull/Push** – These classes offset columns left or right by multiples of the gutter (`40px`).  
- **Append/Prepend** – Add extra gutter space to the right or left side of a column.  
- **Clearfix** – Ensures parent containers correctly wrap floated children.  

### Assumptions & Constraints
- Assumes a **fixed 960px** container width (`.container { width:950px; }`), with 10px gutters.  
- Works best in **desktop or laptop** browsers; mobile responsiveness is not addressed (no media queries).  
- Depends on the **box model**: content width is fixed; padding and borders are added outside (standard CSS).  
- The style sheet is **monolithic**; no modularization (Sass/LESS).  

---

## 3. Functions/Methods (Selectors & Utility Classes)

| Selector / Class | Purpose | Notes |
|-------------------|---------|-------|
| `html, body, div, span, …` | Element reset | Normalizes defaults; inherits font, removes margin/padding. |
| `body` | Global typography & colors | Sets line height, base font-size, background, and text color. |
| `h1`–`h6` | Headings | Distinct font sizes/margins; color inherits. |
| `blockquote, q` | Quotes | Removes default quote styling. |
| `a` | Links | Underlined, default color; hover/focus darkens. |
| `.small`, `.large` | Typography helpers | Smaller/larger font-size & line-height. |
| `.hide`, `.quiet`, `.loud` | Visibility & emphasis | Toggle display or color intensity. |
| `.highlight`, `.added`, `.removed` | Text markers | Background/foreground color for diff/markup. |
| `.container` | Grid container | Fixed width, centered. |
| `.span‑1 … .span‑24` | Grid columns | Float left, fixed width, gutter spacing. |
| `.column` | Alias for `.span‑n` | Supports generic column use. |
| `.last`, `.prepend‑n`, `.append‑n` | Gutter management | Adjust margin/padding for left/right spacing. |
| `.pull‑n`, `.push‑n` | Offset columns | Relative positioning to shift horizontally. |
| `.clearfix:after`, `.container:after` | Clear floats | Generates invisible block to clear children. |
| `input`, `textarea`, `select` | Form elements | Background, border, focus styles. |
| `.error`, `.notice`, `.success` | Feedback messages | Distinct background and border colors for UX. |

### Side‑Effects
- **Float/Positioning** – `.pull‑n` and `.push‑n` add relative positioning, which may affect stacking contexts.  
- **Gutter Overlap** – Adjacent `.span‑n` elements may accumulate 10px gutters on both sides; careful use of `.last` is required.  
- **Browser Specificity** – The reset uses `font-weight:inherit;` and `font-style:inherit;` which may not be supported in very old browsers.  

---

## 4. Dependencies
| Dependency | Type | Remarks |
|------------|------|---------|
| `Helvetica Neue`, Arial, Helvetica | Web‑safe fonts | Fallback chain; not loaded via `@font-face`. |
| `grid.png` | Image | Used by `.showgrid` class; needs to be present in the same directory. |
| None else | | No external libraries (Bootstrap, jQuery, etc.) |

> **Platform‑Specific** – The stylesheet relies on standard CSS and will work on any modern desktop browser. Mobile devices may require additional media queries for responsiveness.

---

## 5. Additional Notes & Recommendations

### Strengths
- **Simplicity & Predictability** – Classic 960‑Grid system; well‑understood by many developers.  
- **Comprehensive Reset** – Normalizes most common element defaults, reducing cross‑browser differences.  
- **Utility‑First** – Classes like `.small`, `.highlight`, `.error` allow quick styling without custom CSS.  

### Weaknesses / Edge Cases
- **Fixed Width** – 950px container doesn’t adapt to smaller viewports; no fluid or responsive design.  
- **Gutter Management** – Hard‑coded 10px gutters may lead to layout bugs if used with `margin` or `padding` on child elements.  
- **No Flexbox/Grid** – Modern layout techniques (Flexbox, CSS Grid) are absent, limiting flexibility for complex designs.  
- **Performance** – The rule list is large (hundreds of selectors); some browsers may experience slower CSS parsing.  
- **Over‑Specificity** – Some selectors like `.span‑1` or `.error` may clash with custom class names in large projects.  

### Future Enhancements
1. **Responsive Breakpoints** – Add media queries to collapse the 24‑column grid into a fluid 12‑column or single column layout on mobile.  
2. **Variable Grid** – Replace hard‑coded widths with CSS variables (`--gutter: 10px; --column-width: 40px;`) for easier theme changes.  
3. **Modern Layout** – Offer a Flexbox or CSS Grid variant as an alternative to floats.  
4. **Modularization** – Convert to SCSS/LESS modules (e.g., `_grid.scss`, `_reset.scss`) to enable mixins and imports.  
5. **Accessibility** – Ensure contrast ratios meet WCAG; consider `focus-visible` for keyboard navigation.  
6. **Optimizations** – Merge duplicate rules, remove unused selectors (e.g., `.append‑1` etc. that are never used in the project).  

### Testing Checklist
- Verify grid behavior across major browsers (Chrome, Firefox, Safari, Edge).  
- Test on devices of various widths; confirm layout breaks or adapts correctly.  
- Check that form controls maintain the desired appearance in different locales (e.g., right‑to‑left languages).  
- Ensure that error/notice/success classes render properly and are accessible (screen reader announcements).  

---

**Bottom line:**  
This stylesheet is a solid foundation for desktop‑only layouts using the 960‑Grid system. For modern web projects, consider adding responsive capabilities, leveraging native Flexbox/Grid, and modularizing the CSS to keep the codebase maintainable and performant.

## Code Critique



## Code Preview

```css
html, body, div, span, object, iframe,h1, h2, h3, h4, h5, h6, p, blockquote, pre,a, abbr, acronym, address, code,del, dfn, em, img, q, dl, dt, dd, ol, ul, li,fieldset, form, label, legend,table, caption, tbody, tfoot, thead, tr, th, td { margin:0; padding:0; border:0; font-weight:inherit; font-style:inherit; font-size:100%; font-family:inherit; vertical-align:baseline;}body { line-height:1.5; }table { border-collapse:separate; border-spacing:0; }caption, th, td { text-align:left; font-weight:normal; }table, td, th { vertical-align:middle; }blockquote:before, blockquote:after, q:before, q:after { content:""; }blockquote, q { quotes:"" ""; }a img { border:none; }html { font-size:100.01%; }body { font-size:75%; color:#222; background:#fff; font-family:"Helvetica Neue", Arial, Helvetica, sans-serif;}h1,h2,h3,h4,h5,h6 { font-weight:normal; color:#111; }h1 { font-size:3em; line-height:1; margin-bottom:0.5em; }h2 { font-size:2em; margin-bottom:0.75em; }h3 { font-size:1.5em; line-height:1; margin-bottom:1em; }h4 { font-size:1.2em; line-height:1.25; margin-bottom:1.25em; }h5 { font-size:1em; font-weight:bold; margin-bottom:1.5em; }h6 { font-size:1em; font-weight:bold; }h1 img, h2 img, h3 img, h4 img, h5 img, h6 img { margin:0;}p { margin:0 0 1.5em; }p img.left { float:left; margin:1.5em 1.5em 1.5em 0; padding:0; }p img.right { float:right; margin:1.5em 0 1.5em 1.5em; }a:focus, a:hover { color:#000; }a { color:#009; text-decoration:underline; }blockquote { margin:1.5em; color:#666; font-style:italic; }strong { font-weight:bold; }em,dfn { font-style:italic; }dfn { font-weight:bold; }sup, sub { line-height:0; }abbr, acronym { border-bottom:1px dotted #666; }address { margin:0 0 1.5em; font-style:italic; }del { color:#666; }pre { margin:1.5em 0; white-space:pre; }pre,code,tt { font:1em 'andale mono', 'lucida console', monospace; line-height:1.5; }li ul, li ol { margin:0 1.5em; }ul, ol { margin:0 1.5em 1.5em 1.5em; }ul { list-style-type:disc; }ol { list-style-type:decimal; }dl { margin:0 0 1.5em 0; }dl dt { font-weight:bold; }dd { margin-left:1.5em;}table { margin-bottom:1.4em; width:100%; }th { font-weight:bold; }thead th { background:#c3d9ff; }th,td,caption { padding:4px 10px 4px 5px; }tr.even td { background:#e5ecf9; }tfoot { font-style:italic; }caption { background:#eee; }.small { font-size:.8em; margin-bottom:1.875em; line-height:1.875em; }.large { font-size:1.2em; line-height:2.5em; margin-bottom:1.25em; }.hide { display:none; }.quiet { color:#666; }.loud { color:#000; }.highlight { background:#ff0; }.added { background:#060; color:#fff; }.removed { background:#900; color:#fff; }.first { margin-left:0; padding-left:0; }.last { margin-right:0; padding-right:0; }.top { margin-top:0; padding-top:0; }.bottom { margin-bottom:0; padding-bottom:0; }.container { width:950px; margin:0 auto;}.showgrid { background:url(grid.png); }.column, div.span-1, div.span-2, div.span-3, div.span-4, div.span-5, div.span-6, div.span-7, div.span-8, div.span-9, div.span-10, div.span-11, div.span-12, div.span-13, div.span-14, div.span-15, div.span-16, div.span-17, div.span-18, div.span-19, div.span-20, div.span-21, div.span-22, div.span-23, div.span-24 { float:left; margin-right:10px;}.last, div.last { margin-right:0; }.span-1 { width:30px; }.span-2 { width:70px; }.span-3 { width:110px; }.span-4 { width:150px; }.span-5 { width:190px; }.span-6 { width:230px; }.span-7 { width:270px; }.span-8 { width:310px; }.span-9 { width:350px; }.span-10 { width:390px; }.span-11 { width:430px; }.span-12 { width:470px; }.span-13 { width:510px; }.span-14 { width:550px; }.span-15 { width:590px; }.span-16 { width:630px; }.span-17 { width:670px; }.span-18 { width:710px; }.span-19 { width:750px; }.span-20 { width:790px; }.span-21 { width:830px; }.span-22 { width:870px; }.span-23 { width:910px; }.span-24 { width:950px; margin-right:0; }input.span-1, textarea.span-1, input.span-2, textarea.span-2, input.span-3, textarea.span-3, input.span-4, textarea.span-4, input.span-5, textarea.span-5, input.span-6, textarea.span-6, input.span-7, textarea.span-7, input.span-8, textarea.span-8, input.span-9, textarea.span-9, input.span-10, textarea.span-10, input.span-11, textarea.span-11, input.span-12, textarea.span-12, input.span-13, textarea.span-13, input.span-14, textarea.span-14, input.span-15, textarea.span-15, input.span-16, textarea.span-16, input.span-17, textarea.span-17, input.span-18, textarea.span-18, input.span-19, textarea.span-19, input.span-20, textarea.span-20, input.span-21, textarea.span-21, input.span-22, textarea.span-22, input.span-23, textarea.span-23, input.span-24, textarea.span-24 { border-left-width:1px!important; border-right-width:1px!important; padding-left:5px!important; padding-right:5px!important;}input.span-1, textarea.span-1 { width:18px!important; }input.span-2, textarea.span-2 { width:58px!important; }input.span-3, textarea.span-3 { width:98px!important; }input.span-4, textarea.span-4 { width:138px!important; }input.span-5, textarea.span-5 { width:178px!important; }input.span-6, textarea.span-6 { width:218px!important; }input.span-7, textarea.span-7 { width:258px!important; }input.span-8, textarea.span-8 { width:298px!important; }input.span-9, textarea.span-9 { width:338px!important; }input.span-10, textarea.span-10 { width:378px!important; }input.span-11, textarea.span-11 { width:418px!important; }input.span-12, textarea.span-12 { width:458px!important; }input.span-13, textarea.span-13 { width:498px!important; }input.span-14, textarea.span-14 { width:538px!important; }input.span-15, textarea.span-15 { width:578px!important; }input.span-16, textarea.span-16 { width:618px!important; }input.span-17, textarea.span-17 { width:658px!important; }input.span-18, textarea.span-18 { width:698px!important; }input.span-19, textarea.span-19 { width:738px!important; }input.span-20, textarea.span-20 { width:778px!important; }input.span-21, textarea.span-21 { width:818px!important; }input.span-22, textarea.span-22 { width:858px!important; }input.span-23, textarea.span-23 { width:898px!important; }input.span-24, textarea.span-24 { width:938px!important; }.append-1 { padding-right:40px; } .append-2 { padding-right:80px; } .append-3 { padding-right:120px; } .append-4 { padding-right:160px; } .append-5 { padding-right:200px; } .append-6 { padding-right:240px; } .append-7 { padding-right:280px; } .append-8 { padding-right:320px; } .append-9 { padding-right:360px; } .append-10 { padding-right:400px; } .append-11 { padding-right:440px; } .append-12 { padding-right:480px; } .append-13 { padding-right:520px; } .append-14 { padding-right:560px; } .append-15 { padding-right:600px; } .append-16 { padding-right:640px; } .append-17 { padding-right:680px; } .append-18 { padding-right:720px; } .append-19 { padding-right:760px; } .append-20 { padding-right:800px; } .append-21 { padding-right:840px; } .append-22 { padding-right:880px; } .append-23 { padding-right:920px; } .prepend-1 { padding-left:40px; } .prepend-2 { padding-left:80px; } .prepend-3 { padding-left:120px; } .prepend-4 { padding-left:160px; } .prepend-5 { padding-left:200px; } .prepend-6 { padding-left:240px; } .prepend-7 { padding-left:280px; } .prepend-8 { padding-left:320px; } .prepend-9 { padding-left:360px; } .prepend-10 { padding-left:400px; } .prepend-11 { padding-left:440px; } .prepend-12 { padding-left:480px; } .prepend-13 { padding-left:520px; } .prepend-14 { padding-left:560px; } .prepend-15 { padding-left:600px; } .prepend-16 { padding-left:640px; } .prepend-17 { padding-left:680px; } .prepend-18 { padding-left:720px; } .prepend-19 { padding-left:760px; } .prepend-20 { padding-left:800px; } .prepend-21 { padding-left:840px; } .prepend-22 { padding-left:880px; } .prepend-23 { padding-left:920px; } div.border { padding-right:4px; margin-right:5px; border-right:1px solid #eee;}div.colborder { padding-right:24px; margin-right:25px; border-right:1px solid #eee;}.pull-1 { margin-left:-40px; }.pull-2 { margin-left:-80px; }.pull-3 { margin-left:-120px; }.pull-4 { margin-left:-160px; }.pull-5 { margin-left:-200px; }.pull-6 { margin-left:-240px; }.pull-7 { margin-left:-280px; }.pull-8 { margin-left:-320px; }.pull-9 { margin-left:-360px; }.pull-10 { margin-left:-400px; }.pull-11 { margin-left:-440px; }.pull-12 { margin-left:-480px; }.pull-13 { margin-left:-520px; }.pull-14 { margin-left:-560px; }.pull-15 { margin-left:-600px; }.pull-16 { margin-left:-640px; }.pull-17 { margin-left:-680px; }.pull-18 { margin-left:-720px; }.pull-19 { margin-left:-760px; }.pull-20 { margin-left:-800px; }.pull-21 { margin-left:-840px; }.pull-22 { margin-left:-880px; }.pull-23 { margin-left:-920px; }.pull-24 { margin-left:-960px; }.pull-1, .pull-2, .pull-3, .pull-4, .pull-5, .pull-6, .pull-7, .pull-8, .pull-9, .pull-10, .pull-11, .pull-12, .pull-13, .pull-14, .pull-15, .pull-16, .pull-17, .pull-18, .pull-19, .pull-20, .pull-21, .pull-22, .pull-23, .pull-24 {float:left; position:relative;}.push-1 { margin:0 -40px 1.5em 40px; }.push-2 { margin:0 -80px 1.5em 80px; }.push-3 { margin:0 -120px 1.5em 120px; }.push-4 { margin:0 -160px 1.5em 160px; }.push-5 { margin:0 -200px 1.5em 200px; }.push-6 { margin:0 -240px 1.5em 240px; }.push-7 { margin:0 -280px 1.5em 280px; }.push-8 { margin:0 -320px 1.5em 320px; }.push-9 { margin:0 -360px 1.5em 360px; }.push-10 { margin:0 -400px 1.5em 400px; }.push-11 { margin:0 -440px 1.5em 440px; }.push-12 { margin:0 -480px 1.5em 480px; }.push-13 { margin:0 -520px 1.5em 520px; }.push-14 { margin:0 -560px 1.5em 560px; }.push-15 { margin:0 -600px 1.5em 600px; }.push-16 { margin:0 -640px 1.5em 640px; }.push-17 { margin:0 -680px 1.5em 680px; }.push-18 { margin:0 -720px 1.5em 720px; }.push-19 { margin:0 -760px 1.5em 760px; }.push-20 { margin:0 -800px 1.5em 800px; }.push-21 { margin:0 -840px 1.5em 840px; }.push-22 { margin:0 -880px 1.5em 880px; }.push-23 { margin:0 -920px 1.5em 920px; }.push-24 { margin:0 -960px 1.5em 960px; }.push-1, .push-2, .push-3, .push-4, .push-5, .push-6, .push-7, .push-8, .push-9, .push-10, .push-11, .push-12, .push-13, .push-14, .push-15, .push-16, .push-17, .push-18, .push-19, .push-20, .push-21, .push-22, .push-23, .push-24 {float:right; position:relative;}.prepend-top { margin-top:1.5em; }.append-bottom { margin-bottom:1.5em; } .box { padding:1.5em; margin-bottom:1.5em; background:#E5ECF9; }hr { background:#ddd; color:#ddd; clear:both; float:none; width:100%; height:.1em; margin:0 0 1.4em; border:none; }hr.space { background:#fff; color:#fff;}.clearfix:after, .container:after { content:"\0020"; display:block; height:0; clear:both; visibility:hidden; overflow:hidden; }.clearfix, .container {display:block;}.clear { clear:both; }label { font-weight:bold; }fieldset { padding:1.4em; margin:0 0 1.5em 0; border:1px solid #ccc; }legend { font-weight:bold; font-size:1.2em; }input[type=text], input[type=password],input.text, input.title, textarea, select { background-color:#fff; border:1px solid #bbb; }input[type=text]:focus, input[type=password]:focus, input.text:focus, input.title:focus, textarea:focus, select:focus { border-color:#666; }input[type=text], input[type=password],input.text, input.title,textarea, select { margin:0.5em 0;}input.text, input.title { width:300px; padding:5px; }input.title { font-size:1.5em; }textarea { width:390px; height:250px; padding:5px; }input[type=checkbox], input[type=radio], input.checkbox, input.radio { position:relative; top:.25em; }form.inline { line-height:3; }form.inline p { margin-bottom:0; }.error,.notice, .success { padding:.8em; margin-bottom:1em; border:2px solid #ddd; }.error { background:#FBE3E4; color:#8a1f11; border-color:#FBC2C4; }.notice { background:#FFF6BF; color:#514721; border-color:#FFD324; }.success { background:#E6EFC2; color:#264409; border-color:#C6D880; }.error a { color:#8a1f11; }.notice a { color:#514721; }.success a { color:#264409; }


```
