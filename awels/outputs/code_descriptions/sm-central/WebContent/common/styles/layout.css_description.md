# layout.css

## Review

## 1. Summary  
The file is a **site‑specific CSS layout** written by Mike Stenhouse of Content with Style. It contains global styles for the page skeleton (`body`, `#page`, `#header`, `#content`, `#footer`) and a small navigation helper (`ul.subnav`). The design is simple, text‑centered and uses a few basic colors. No CSS preprocessors or frameworks are involved; the stylesheet is plain CSS intended to be dropped into an HTML document.

**Key components**  
| Section | Purpose |
|---------|---------|
| `body`, `#page` | General layout and centering of the main container |
| `#header`, `#branding`, `#search` | Header layout with left‑aligned branding and right‑aligned search area |
| `#header_spacer` | Visual spacer to push content below the header |
| `#content`, `#main`, `#sub` | Containers for the main content area and optional sub‑content |
| `#footer` | Page footer styling |
| `ul.subnav` | Utility navigation list with a highlighted item styling |

No design patterns are applied beyond the common “box model” and float layout. The file is intentionally minimalistic, relying on HTML structure rather than complex CSS features.

---

## 2. Detailed Description  
The stylesheet defines a *basic two‑column* layout:

1. **Page wrapper** (`div#page`)  
   - 98 % width, centered via `margin: 0 auto`.  
   - Text is centered by default; sub‑elements override as needed.

2. **Header area** (`div#header`)  
   - Left‑aligned brand and right‑aligned search bar are floated.  
   - No background color is set (`background: #FFFFFF;` actually overrides the white background inherited from `body`).  
   - A spacer block (`#header_spacer`) adds 5 em of height, presumably to separate the header from the content.

3. **Content area** (`div#content`)  
   - Empty in the stylesheet – left for the developer to add specific styles.  
   - Inside it there are two optional regions: `#main` and `#sub`.  Both are empty, meant to be filled later.

4. **Footer** (`div#footer`)  
   - Blue background (#3381B7) with white text.  
   - Paragraphs inside are small (0.8 em) and padded.

5. **Navigation list** (`ul.subnav`)  
   - Removes default list styling.  
   - List items stack vertically; each link is bold, black, no underline by default.  
   - Hover state adds underline.  
   - The `strong` element is used to mark the active or highlighted item: it shows a left‑hand background image (`subnav-highlight.gif`) and changes link color to white on a gray (#818EBD) background.

The stylesheet assumes a **static page** structure: each region is identified by an ID, and the content inside each region can be styled separately in later files or inline.

---

## 3. Functions/Methods  
CSS does not contain executable functions, but each **selector** acts as a “rule set” that applies styles to matching elements. The most noteworthy selector sets are:

| Selector | Target | Purpose |
|----------|--------|---------|
| `body` | Root element | Basic reset (margin/padding) and center text |
| `div#page` | Main container | Width & centering |
| `div#header`, `#branding`, `#search` | Header layout | Float positioning |
| `div#header_spacer` | Spacer element | Adds vertical space |
| `div#content`, `#main`, `#sub` | Content containers | Reserved for future styles |
| `div#footer` | Footer | Background & text colors |
| `ul.subnav` | Navigation list | List reset, font size |
| `ul.subnav li a` | Links | Bold, black, no underline |
| `ul.subnav li strong` | Highlighted item | Background image |
| `ul.subnav li strong a` | Link inside highlighted item | White text on gray background |

These “functions” are pure presentation rules; no side effects beyond styling.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `../img/subnav-highlight.gif` | External image | Requires the image file to exist at the specified relative path. |
| None else | | The stylesheet uses only standard CSS3 properties (float, background, margin, padding, etc.). No third‑party libraries or preprocessors are referenced. |

The code is **browser‑agnostic** for modern browsers; older browsers may not support `box-sizing` or flexbox, but the layout relies on floats, which have wide support.

---

## 5. Additional Notes  

### Strengths
- **Simplicity**: Easy to understand and extend.  
- **Separation of concerns**: Layout structure is separate from content, allowing future stylesheets to target `#main`, `#sub`, etc.  
- **Minimal footprint**: Small file size, no heavy dependencies.

### Potential Issues & Edge Cases
1. **Float Clearing**: The floated `#branding` and `#search` elements are not cleared. If the page contains other content directly after the header, the layout may collapse. Adding a clearfix or using `overflow: hidden` on the header wrapper would mitigate this.  
2. **Responsive Design**: The layout is fixed‑width (98 % of viewport) with no media queries. On small screens, the `#branding` (50 %) and `#search` (40 %) floats may overlap or create excessive whitespace.  
3. **Accessibility**: The header has a white background on a white `body`, but the text color is also white – effectively invisible unless overridden.  
4. **Image Dependency**: The highlighted navigation image must exist; missing it will result in broken background.  
5. **Redundant/Unused Rules**: Several selector blocks (`#content`, `#main`, `#sub`) are empty; keeping them may be unnecessary unless they will receive styles later.

### Suggested Enhancements
- **Clearfix** for floated elements or switch to Flexbox for the header.  
- **Responsive breakpoints** to adapt column widths on smaller devices.  
- **Use of `:focus` styles** for navigation links to improve keyboard accessibility.  
- **Theme variables** (e.g., CSS custom properties) for colors to ease brand changes.  
- **Documentation comments** inline to explain the intended usage of each region.  

Overall, the stylesheet serves as a solid foundation for a simple page layout but would benefit from addressing the floating and responsiveness concerns to make it production‑ready.

## Code Critique



## Code Preview

```css
/* A CSS Framework by Mike Stenhouse of Content with Style */

/* SITE SPECIFIC LAYOUT */
    body {
        margin: 0;
        padding: 0;

        background: white;

        text-align: center;
    }
    div#page {
        width: 98%;
        margin:  0 auto;
        padding: 0;

        text-align: center;
    }

    /* HEADER */
        div#header {
            margin: 0;
            padding: 5px 0px 0 0;
            color: white;
            background: #FFFFFF;
            text-align: left;
        }
        div#branding {
            float: left;
            width: 50%;

            margin: 0;
            padding: 5px 0 5px 10px;

            text-align: left;
        }
        div#search {
            float: right;
            width: 40%;

            margin: 0;
            padding: 5px 0px 5px 10px;

            text-align: right;
        }
    div#header_spacer {
            display: block;
            height: 5em;
            background-color: transparent;
        }
    /* END HEADER */


    /* CONTENT */

    /* END HEADER */


    /* CONTENT */
        div#content {

        }
        /* MAIN */
            div#main {

            }
        /* END MAIN */

        /* SUB */
            div#sub {

            }
        /* END SUB */

    /* END CONTENT */


    /* FOOTER */
        div#footer {
            color: white;
            background-color: #3381B7;
            border-width:0;
	    margin-bottom: 4px;
        }
        div#footer p {
            font-size: 0.8em;
            margin: 0;
            padding: 5px;
        }
    /* END FOOTER */
/* END LAYOUT */




/* UL.SUBNAV */
    ul.subnav {
        margin: 0;
        padding: 0;

        font-size: 0.8em;
        list-style: none;
    }
    ul.subnav li {
        margin: 0 0 1em 0;
        padding: 0;
        list-style: none;
    }
    ul.subnav li a,
    ul.subnav li a:link,
    ul.subnav li a:visited,
    ul.subnav li a:active {
        text-decoration: none;
        font-weight: bold;
        color: black;
    }
    ul.subnav li a:hover {
        text-decoration: underline;
    }
    ul.subnav li strong {
        padding: 0 0 0 12px;
        background: url("../img/subnav-highlight.gif") left top no-repeat transparent;
    }
    ul.subnav li strong a,
    ul.subnav li strong a:link,
    ul.subnav li strong a:visited,
    ul.subnav li strong a:active {
        color: white;
        background-color: #818EBD;
    }
/* END UL.SUBNAV */



```
