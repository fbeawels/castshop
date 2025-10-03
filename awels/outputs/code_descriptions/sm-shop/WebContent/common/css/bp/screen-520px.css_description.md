# screen-520px.css

## Review

## 1. Summary

**Purpose & Scope**  
This CSS file is a classic “reset + layout” bundle that:

1. Normalises browser defaults (margin, padding, borders, fonts, etc.) for a wide range of HTML elements.  
2. Provides a **12‑point (actually 24‑point) grid system** based on *float* positioning (the familiar “span‑X” classes).  
3. Supplies common UI utilities (`.box`, `.error`, `.notice`, `.success`, `.clearfix`, etc.).  
4. Includes a few typography helpers (`.small`, `.large`, `.quiet`, `.loud`, `.highlight`).

**Key Components**

| Component | Role |
|-----------|------|
| Reset rules | Strip out all native spacing and sizing quirks |
| Typography styles | Base font‑size, headings, blockquotes, lists, tables |
| Grid helpers | `.container`, `.span‑X`, `.pull‑X`, `.push‑X`, `.append‑X`, `.prepend‑X`, `.column`, `.border` |
| Utility classes | `.box`, `.error`, `.notice`, `.success`, `.clearfix`, `.clear`, `.showgrid` |
| Form styling | Inputs, textareas, buttons, fieldsets, etc. |

**Design Patterns & Libraries**

- **Reset** – follows the “Eric Meyer” approach, but with extended element list.  
- **Grid System** – a classic 12/24 column float grid; reminiscent of 960.gs.  
- **Utility Classes** – BEM‑like but not strict; simple naming scheme.  
- **No external dependencies** – all styles are self‑contained except for the optional `grid.png` background used by `.showgrid`.

---

## 2. Detailed Description

### 2.1 Reset & Base Styling

The first block applies zero margin, padding, border, font‑inherit, and baseline line‑height to a **large set of elements**. This ensures a consistent baseline across browsers.  

- **Font sizing**: The body is set to `75%` of the browser default (`font-size:75%`). This is a common trick to allow `rem`‑based sizing in a relative scale but can be confusing for developers who expect 16 px as the default.  
- **Body & HTML font**: Helvetica Neue → Arial → Helvetica → sans-serif with a fallback of the generic sans serif.  
- **Headings**: Each level gets a specific `font-size` and margin to produce a typographic hierarchy.  
- **Lists & tables**: Standardised list styles and table padding.  
- **Quoting**: `blockquote` and `q` have no default quotes; quotes are manually defined (empty) to suppress the browser’s default quotation marks.

### 2.2 Layout – The Grid

#### 2.2.1 Container
```css
.container { width:520px; margin:0 auto; }
```
- Fixed width (520 px) and centered. Not responsive; ideal for desktop but will break on smaller viewports.

#### 2.2.2 Columns & Span
```css
.column,
div.span-1 … div.span-24 { float:left; margin-right:8px; }
```
- All columns share a right gutter of 8 px.  
- `.span‑X` classes set a width that scales linearly: 14 px, 36 px, … 520 px for `span-24`.  
- This is a 24‑column grid; 24 × 22 px (width) + 23 × 8 px (gutters) = 520 px.

#### 2.2.3 Pull / Push / Append / Prepend
- **Pull / Push**: Negative or positive left/right margins to move a column relative to the normal flow.  
- **Append / Prepend**: Add right/left padding to a column (for interior spacing).  
- Each set defines 24 individual rules (e.g., `.pull-1`, `.pull-2`, …), again heavily repetitive.

#### 2.2.4 Border Helpers
```css
div.border { padding-right:3px; margin-right:4px; border-right:1px solid #eee; }
div.colborder { padding-right:14px; margin-right:15px; border-right:1px solid #eee; }
```
- Small utilities to add a right border and accompanying spacing.

#### 2.2.5 Float & Positioning
- All grid items are `float:left`.  
- `.pull-*` and `.push-*` items also have `float:left` or `float:right` depending on direction.  
- `.clearfix` is a classic “after” hack to clear floats.

### 2.3 Utilities

- `.box`, `.error`, `.notice`, `.success`: Simple block elements with padding, margin, background, and border colours.  
- `.quiet`, `.loud`, `.highlight`, `.added`, `.removed`: Text colour helpers.  
- `.small`, `.large`: Font size modifiers.  
- `.showgrid`: Sets a background image (`grid.png`) to aid visual debugging of the grid.

### 2.4 Forms

- Input and textarea selectors share background, border, margin, and padding.  
- Specific sizing for `.text` and `.title` inputs.  
- `input[type=checkbox]` and `radio` are slightly offset for better alignment.  
- `.error`, `.notice`, `.success` styles extend to links within the messages.

---

## 3. Functions/Selectors (High‑Level Overview)

| Selector Group | Purpose | Key Attributes |
|----------------|---------|----------------|
| `html, body, ...` | Reset core elements | `margin:0; padding:0; border:0; font-weight:inherit; ...` |
| `body` | Base typography | `font-size:75%; color:#222; background:#fff; font-family:...` |
| `h1-h6` | Heading hierarchy | `font-size`, `margin`, `color` |
| `p`, `blockquote`, `pre`, `ul`, `ol`, `dl` | Block elements | Margins, padding, lists, code font |
| `table`, `th`, `td` | Table styling | `border-collapse`, `padding`, background |
| `.container` | Centered wrapper | Fixed width |
| `.column`, `.span-1…24` | Column widths | `float:left`, `margin-right:8px`, width |
| `.pull-*`, `.push-*`, `.append-*`, `.prepend-*` | Horizontal positioning helpers | `margin-left/right` or `padding-left/right` |
| `.border`, `.colborder` | Right border with spacing | `border-right` + paddings |
| `.clearfix`, `.container:after` | Clear floats | `content:"\0020";` hack |
| `.box`, `.error`, `.notice`, `.success` | UI state containers | `padding`, `margin`, `background`, `border` |
| Form inputs | Standard form styling | `background-color`, `border`, `margin`, `padding` |
| `input[type=checkbox]` | Align with text | `position:relative; top:.25em;` |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `grid.png` | Image | Used only by `.showgrid`; optional for debugging. |
| Browser CSS | Built‑in | No third‑party libraries required. |

No external CSS frameworks or pre‑processors are referenced.

---

## 5. Additional Notes & Recommendations

### 5.1 Maintainability

- **Repetition**: Almost every helper class (`.pull-1`, `.push-1`, `.append-1`, etc.) is written out manually. This creates a maintenance burden.  
  *Recommendation*: Use CSS pre‑processors (Sass/SCSS, Less) or CSS Custom Properties (variables) to generate these classes via loops or maps.  
- **Fixed Width**: `.container` is 520 px, which is non‑responsive.  
  *Recommendation*: Replace with a fluid width (`width: 100%; max-width:520px;`) or use `max-width` for better mobile support.  

### 5.2 Performance

- The use of many class selectors and repeated rules may inflate the CSS file size.  
- Floating layout can be sub‑optimal on modern devices; consider `flexbox` or `grid` for better performance and easier alignment.

### 5.3 Accessibility

- Font‑size of 75% may interfere with user’s browser scaling; test with high DPI displays.  
- `input` styles don’t include focus states beyond border‑color change; adding `outline` can improve keyboard navigation.  

### 5.4 Potential Edge Cases

- **Gutter Collapse**: Using `margin-right:8px` on all columns will double the gutter between adjacent columns (left margin of the second column is 8 px). This is intentional, but may cause misalignment if the grid is nested.  
- **Nested Grids**: The current grid doesn’t account for nested `.container`/`.span-*` pairs; additional rules or wrapper classes might be needed.  
- **IE10+**: Floats are well‑supported, but the `:after` clearfix hack may behave unexpectedly in some older browsers. Using the `display:flex; flex-wrap:wrap;` approach would be more robust.  

### 5.5 Modernization

| Feature | Current | Modern Alternative |
|---------|---------|--------------------|
| Grid system | 24‑column float grid | CSS Grid (`grid-template-columns: repeat(24, 1fr);`) |
| Positioning helpers | `.pull-*`, `.push-*` | `order`, `margin`, `justify-content` in Flexbox |
| Typography scaling | `75%` + rem‑like calculations | `clamp()` for fluid type |
| Reset | Manual rule set | `normalize.css` or `modern-normalize` |

---

### Bottom Line

The CSS provides a solid, if somewhat dated, foundation for a desktop‑first web project: it normalises cross‑browser differences, supplies a flexible grid, and includes handy utility classes. However, the sheer amount of repetitive code and the lack of responsiveness limit its long‑term scalability. Refactoring with a pre‑processor, adopting modern layout techniques (Flexbox/Grid), and making the container fluid would make the stylesheet easier to maintain and more robust on contemporary devices.

## Code Critique



## Code Preview

```css
html, body, div, span, object, iframe,h1, h2, h3, h4, h5, h6, p, blockquote, pre,a, abbr, acronym, address, code,del, dfn, em, img, q, dl, dt, dd, ol, ul, li,fieldset, form, label, legend,table, caption, tbody, tfoot, thead, tr, th, td { margin:0; padding:0; border:0; font-weight:inherit; font-style:inherit; font-size:100%; font-family:inherit; vertical-align:baseline;}body { line-height:1.5; }table { border-collapse:separate; border-spacing:0; }caption, th, td { text-align:left; font-weight:normal; }table, td, th { vertical-align:middle; }blockquote:before, blockquote:after, q:before, q:after { content:""; }blockquote, q { quotes:"" ""; }a img { border:none; }html { font-size:100.01%; }body { font-size:75%; color:#222; background:#fff; font-family:"Helvetica Neue", Arial, Helvetica, sans-serif;}h1,h2,h3,h4,h5,h6 { font-weight:normal; color:#111; }h1 { font-size:3em; line-height:1; margin-bottom:0.5em; }h2 { font-size:2em; margin-bottom:0.75em; }h3 { font-size:1.5em; line-height:1; margin-bottom:1em; }h4 { font-size:1.2em; line-height:1.25; margin-bottom:1.25em; }h5 { font-size:1em; font-weight:bold; margin-bottom:1.5em; }h6 { font-size:1em; font-weight:bold; }h1 img, h2 img, h3 img, h4 img, h5 img, h6 img { margin:0;}p { margin:0 0 1.5em; }p img.left { float:left; margin:1.5em 1.5em 1.5em 0; padding:0; }p img.right { float:right; margin:1.5em 0 1.5em 1.5em; }a:focus, a:hover { color:#000; }a { color:#009; text-decoration:underline; }blockquote { margin:1.5em; color:#666; font-style:italic; }strong { font-weight:bold; }em,dfn { font-style:italic; }dfn { font-weight:bold; }sup, sub { line-height:0; }abbr, acronym { border-bottom:1px dotted #666; }address { margin:0 0 1.5em; font-style:italic; }del { color:#666; }pre { margin:1.5em 0; white-space:pre; }pre,code,tt { font:1em 'andale mono', 'lucida console', monospace; line-height:1.5; }li ul, li ol { margin:0 1.5em; }ul, ol { margin:0 1.5em 1.5em 1.5em; }ul { list-style-type:disc; }ol { list-style-type:decimal; }dl { margin:0 0 1.5em 0; }dl dt { font-weight:bold; }dd { margin-left:1.5em;}table { margin-bottom:1.4em; width:100%; }th { font-weight:bold; }thead th { background:#c3d9ff; }th,td,caption { padding:4px 10px 4px 5px; }tr.even td { background:#e5ecf9; }tfoot { font-style:italic; }caption { background:#eee; }.small { font-size:.8em; margin-bottom:1.875em; line-height:1.875em; }.large { font-size:1.2em; line-height:2.5em; margin-bottom:1.25em; }.hide { display:none; }.quiet { color:#666; }.loud { color:#000; }.highlight { background:#ff0; }.added { background:#060; color:#fff; }.removed { background:#900; color:#fff; }.first { margin-left:0; padding-left:0; }.last { margin-right:0; padding-right:0; }.top { margin-top:0; padding-top:0; }.bottom { margin-bottom:0; padding-bottom:0; }.container { width:520px; margin:0 auto;}.showgrid { background:url(grid.png); }.column, div.span-1, div.span-2, div.span-3, div.span-4, div.span-5, div.span-6, div.span-7, div.span-8, div.span-9, div.span-10, div.span-11, div.span-12, div.span-13, div.span-14, div.span-15, div.span-16, div.span-17, div.span-18, div.span-19, div.span-20, div.span-21, div.span-22, div.span-23, div.span-24 { float:left; margin-right:8px;}.last, div.last { margin-right:0; }.span-1 { width:14px; }.span-2 { width:36px; }.span-3 { width:58px; }.span-4 { width:80px; }.span-5 { width:102px; }.span-6 { width:124px; }.span-7 { width:146px; }.span-8 { width:168px; }.span-9 { width:190px; }.span-10 { width:212px; }.span-11 { width:234px; }.span-12 { width:256px; }.span-13 { width:278px; }.span-14 { width:300px; }.span-15 { width:322px; }.span-16 { width:344px; }.span-17 { width:366px; }.span-18 { width:388px; }.span-19 { width:410px; }.span-20 { width:432px; }.span-21 { width:454px; }.span-22 { width:476px; }.span-23 { width:498px; }.span-24 { width:520px; margin-right:0; }input.span-1, textarea.span-1, input.span-2, textarea.span-2, input.span-3, textarea.span-3, input.span-4, textarea.span-4, input.span-5, textarea.span-5, input.span-6, textarea.span-6, input.span-7, textarea.span-7, input.span-8, textarea.span-8, input.span-9, textarea.span-9, input.span-10, textarea.span-10, input.span-11, textarea.span-11, input.span-12, textarea.span-12, input.span-13, textarea.span-13, input.span-14, textarea.span-14, input.span-15, textarea.span-15, input.span-16, textarea.span-16, input.span-17, textarea.span-17, input.span-18, textarea.span-18, input.span-19, textarea.span-19, input.span-20, textarea.span-20, input.span-21, textarea.span-21, input.span-22, textarea.span-22, input.span-23, textarea.span-23, input.span-24, textarea.span-24 { border-left-width:1px!important; border-right-width:1px!important; padding-left:5px!important; padding-right:5px!important;}input.span-1, textarea.span-1 { width:2px!important; }input.span-2, textarea.span-2 { width:24px!important; }input.span-3, textarea.span-3 { width:46px!important; }input.span-4, textarea.span-4 { width:68px!important; }input.span-5, textarea.span-5 { width:90px!important; }input.span-6, textarea.span-6 { width:112px!important; }input.span-7, textarea.span-7 { width:134px!important; }input.span-8, textarea.span-8 { width:156px!important; }input.span-9, textarea.span-9 { width:178px!important; }input.span-10, textarea.span-10 { width:200px!important; }input.span-11, textarea.span-11 { width:222px!important; }input.span-12, textarea.span-12 { width:244px!important; }input.span-13, textarea.span-13 { width:266px!important; }input.span-14, textarea.span-14 { width:288px!important; }input.span-15, textarea.span-15 { width:310px!important; }input.span-16, textarea.span-16 { width:332px!important; }input.span-17, textarea.span-17 { width:354px!important; }input.span-18, textarea.span-18 { width:376px!important; }input.span-19, textarea.span-19 { width:398px!important; }input.span-20, textarea.span-20 { width:420px!important; }input.span-21, textarea.span-21 { width:442px!important; }input.span-22, textarea.span-22 { width:464px!important; }input.span-23, textarea.span-23 { width:486px!important; }input.span-24, textarea.span-24 { width:508px!important; }.append-1 { padding-right:22px; } .append-2 { padding-right:44px; } .append-3 { padding-right:66px; } .append-4 { padding-right:88px; } .append-5 { padding-right:110px; } .append-6 { padding-right:132px; } .append-7 { padding-right:154px; } .append-8 { padding-right:176px; } .append-9 { padding-right:198px; } .append-10 { padding-right:220px; } .append-11 { padding-right:242px; } .append-12 { padding-right:264px; } .append-13 { padding-right:286px; } .append-14 { padding-right:308px; } .append-15 { padding-right:330px; } .append-16 { padding-right:352px; } .append-17 { padding-right:374px; } .append-18 { padding-right:396px; } .append-19 { padding-right:418px; } .append-20 { padding-right:440px; } .append-21 { padding-right:462px; } .append-22 { padding-right:484px; } .append-23 { padding-right:506px; } .prepend-1 { padding-left:22px; } .prepend-2 { padding-left:44px; } .prepend-3 { padding-left:66px; } .prepend-4 { padding-left:88px; } .prepend-5 { padding-left:110px; } .prepend-6 { padding-left:132px; } .prepend-7 { padding-left:154px; } .prepend-8 { padding-left:176px; } .prepend-9 { padding-left:198px; } .prepend-10 { padding-left:220px; } .prepend-11 { padding-left:242px; } .prepend-12 { padding-left:264px; } .prepend-13 { padding-left:286px; } .prepend-14 { padding-left:308px; } .prepend-15 { padding-left:330px; } .prepend-16 { padding-left:352px; } .prepend-17 { padding-left:374px; } .prepend-18 { padding-left:396px; } .prepend-19 { padding-left:418px; } .prepend-20 { padding-left:440px; } .prepend-21 { padding-left:462px; } .prepend-22 { padding-left:484px; } .prepend-23 { padding-left:506px; } div.border { padding-right:3px; margin-right:4px; border-right:1px solid #eee;}div.colborder { padding-right:14px; margin-right:15px; border-right:1px solid #eee;}.pull-1 { margin-left:-22px; }.pull-2 { margin-left:-44px; }.pull-3 { margin-left:-66px; }.pull-4 { margin-left:-88px; }.pull-5 { margin-left:-110px; }.pull-6 { margin-left:-132px; }.pull-7 { margin-left:-154px; }.pull-8 { margin-left:-176px; }.pull-9 { margin-left:-198px; }.pull-10 { margin-left:-220px; }.pull-11 { margin-left:-242px; }.pull-12 { margin-left:-264px; }.pull-13 { margin-left:-286px; }.pull-14 { margin-left:-308px; }.pull-15 { margin-left:-330px; }.pull-16 { margin-left:-352px; }.pull-17 { margin-left:-374px; }.pull-18 { margin-left:-396px; }.pull-19 { margin-left:-418px; }.pull-20 { margin-left:-440px; }.pull-21 { margin-left:-462px; }.pull-22 { margin-left:-484px; }.pull-23 { margin-left:-506px; }.pull-24 { margin-left:-528px; }.pull-1, .pull-2, .pull-3, .pull-4, .pull-5, .pull-6, .pull-7, .pull-8, .pull-9, .pull-10, .pull-11, .pull-12, .pull-13, .pull-14, .pull-15, .pull-16, .pull-17, .pull-18, .pull-19, .pull-20, .pull-21, .pull-22, .pull-23, .pull-24 {float:left; position:relative;}.push-1 { margin:0 -22px 1.5em 22px; }.push-2 { margin:0 -44px 1.5em 44px; }.push-3 { margin:0 -66px 1.5em 66px; }.push-4 { margin:0 -88px 1.5em 88px; }.push-5 { margin:0 -110px 1.5em 110px; }.push-6 { margin:0 -132px 1.5em 132px; }.push-7 { margin:0 -154px 1.5em 154px; }.push-8 { margin:0 -176px 1.5em 176px; }.push-9 { margin:0 -198px 1.5em 198px; }.push-10 { margin:0 -220px 1.5em 220px; }.push-11 { margin:0 -242px 1.5em 242px; }.push-12 { margin:0 -264px 1.5em 264px; }.push-13 { margin:0 -286px 1.5em 286px; }.push-14 { margin:0 -308px 1.5em 308px; }.push-15 { margin:0 -330px 1.5em 330px; }.push-16 { margin:0 -352px 1.5em 352px; }.push-17 { margin:0 -374px 1.5em 374px; }.push-18 { margin:0 -396px 1.5em 396px; }.push-19 { margin:0 -418px 1.5em 418px; }.push-20 { margin:0 -440px 1.5em 440px; }.push-21 { margin:0 -462px 1.5em 462px; }.push-22 { margin:0 -484px 1.5em 484px; }.push-23 { margin:0 -506px 1.5em 506px; }.push-24 { margin:0 -528px 1.5em 528px; }.push-1, .push-2, .push-3, .push-4, .push-5, .push-6, .push-7, .push-8, .push-9, .push-10, .push-11, .push-12, .push-13, .push-14, .push-15, .push-16, .push-17, .push-18, .push-19, .push-20, .push-21, .push-22, .push-23, .push-24 {float:right; position:relative;}.prepend-top { margin-top:1.5em; }.append-bottom { margin-bottom:1.5em; } .box { padding:1.5em; margin-bottom:1.5em; background:#E5ECF9; }hr { background:#ddd; color:#ddd; clear:both; float:none; width:100%; height:.1em; margin:0 0 1.4em; border:none; }hr.space { background:#fff; color:#fff;}.clearfix:after, .container:after { content:"\0020"; display:block; height:0; clear:both; visibility:hidden; overflow:hidden; }.clearfix, .container {display:block;}.clear { clear:both; }label { font-weight:bold; }fieldset { padding:1.4em; margin:0 0 1.5em 0; border:1px solid #ccc; }legend { font-weight:bold; font-size:1.2em; }input[type=text], input[type=password],input.text, input.title, textarea, select { background-color:#fff; border:1px solid #bbb; }input[type=text]:focus, input[type=password]:focus, input.text:focus, input.title:focus, textarea:focus, select:focus { border-color:#666; }input[type=text], input[type=password],input.text, input.title,textarea, select { margin:0.5em 0;}input.text, input.title { width:300px; padding:5px; }input.title { font-size:1.5em; }textarea { width:390px; height:250px; padding:5px; }input[type=checkbox], input[type=radio], input.checkbox, input.radio { position:relative; top:.25em; }form.inline { line-height:3; }form.inline p { margin-bottom:0; }.error,.notice, .success { padding:.8em; margin-bottom:1em; border:2px solid #ddd; }.error { background:#FBE3E4; color:#8a1f11; border-color:#FBC2C4; }.notice { background:#FFF6BF; color:#514721; border-color:#FFD324; }.success { background:#E6EFC2; color:#264409; border-color:#C6D880; }.error a { color:#8a1f11; }.notice a { color:#514721; }.success a { color:#264409; }


```
