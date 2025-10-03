# nav-horizontal.css

## Review

## 1. Summary  
The stylesheet defines a **horizontal navigation bar** that is positioned at the bottom of the page.  
* **Purpose** – Create a simple, cross‑browser compatible menu that displays a row of links with a highlighted “active” item.  
* **Key components**  
  * `div#nav` – the root container.  
  * `div#nav div.wrapper` – a positioned wrapper that anchors the nav to the page bottom.  
  * `div#nav ul` – the unordered list that holds the navigation items.  
  * `div#nav li` – each list item (menu entry).  
  * `div#nav a` – links inside the items.  
  * `div#nav strong` – an element used to indicate the currently selected page (usually wrapped around the active link).  
* **Design patterns / techniques**  
  * Classic float‑based layout for horizontal lists.  
  * Use of the old “\* html” hack to target IE6 and older browsers.  
  * Repeated `display: inline-block; display: block;` pairs to hide the `inline-block` from legacy browsers that don’t understand it.  

## 2. Detailed Description  
### Core Flow  
1. **Markup** – The page is expected to contain a `<div id="nav">` element.  
2. **Styling** –  
   * The nav container is given a small font‑size for compactness.  
   * The `wrapper` is absolutely positioned at the bottom, stretching full width.  
   * The list (`ul`) is reset to remove default margins/padding and to allow list items to float horizontally.  
   * Each `li` is floated left, with a right border that is removed on the last item via the `.last` class.  
   * Links are styled as block‑level elements (with a fallback for older browsers) so that the full clickable area is the padded rectangle.  
   * The “active” state is represented by a `<strong>` element that encloses the link; this changes text/background colors.  
3. **Runtime behaviour** – When the page loads, the CSS is applied; the menu is rendered horizontally at the bottom of the viewport. Hovering over links underlines them.  
4. **Cleanup** – No JavaScript or dynamic cleanup is required; the menu is static.  

### Assumptions & Constraints  
* **Browser support** – The code explicitly targets legacy browsers (IE6+) with hacks. Modern browsers interpret `inline-block` correctly.  
* **Semantic structure** – The active item relies on a `<strong>` element; this is semantically dubious but acceptable for visual styling.  
* **Layout** – The absolute positioning assumes no other elements are positioned relative to the viewport that might overlap the nav.  

### Architecture & Design Choices  
* **Float‑based horizontal list** – Chosen for maximum browser compatibility before Flexbox/Grid were widely available.  
* **Absolute positioning** – Ensures the nav stays at the bottom regardless of page content height.  
* **Inline‑block fallback** – The duplicate `display` property values hide the `inline-block` from browsers that don’t support it.  

## 3. Functions/Methods (Selectors)  
| Selector | Purpose | Key Properties | Notes |
|---|---|---|---|
| `div#nav` | Root container | `font-size: 0.8em` | Sets a smaller base font. |
| `* html div#nav` | IE6 hack | `height: 1%` | Forces layout in old IE. |
| `div#nav div.wrapper` | Bottom anchor | `position: absolute; left: 0; bottom: 0; width: 100%` | Keeps nav at page bottom. |
| `div#nav ul` | List reset | `margin:0; padding:0; list-style:none; width:100%` | Removes default spacing. |
| `div#nav li` | List items | `float:left; display:inline; border-right:1px solid #aaa;` | Floats horizontally. |
| `div#nav li.last` | Last item | `border-right:none;` | Removes trailing border. |
| `div#nav a, div#nav a:link, div#nav a:active, div#nav a:visited` | Links | `display:inline-block; display:block; font-weight:bold; padding:5px 38px; color:black; background:#ddd` | Provides clickable area and styling. |
| `div#nav a:hover` | Hover | `text-decoration:underline` | Visual feedback. |
| `div#nav strong` | Active item indicator | `color:white; background:black;` | Highlights active state. |
| `div#nav strong a...` | Links inside active item | `color:white; background-color:black` | Ensures active link inherits colors. |

## 4. Dependencies  
* **None** – Pure CSS.  
* **Browser support** – Relies on legacy hacks for IE6 (`* html` selector, `display:inline-block` fallback).  
* **No external libraries or frameworks** – The code is standalone.  

## 5. Additional Notes  
### Edge Cases & Potential Issues  
1. **Mobile / Touch Devices** – The fixed bottom position may obscure content on small screens.  
2. **Accessibility** – The active item is indicated by a `<strong>` tag, which may not convey state to screen readers. Using `aria-current="page"` on the `<a>` element would be clearer.  
3. **Scalability** – Hard‑coded padding (`38px` left/right) limits the menu width; adding many items can cause overflow.  
4. **IE6** – Modern sites rarely need the IE6 hack; it may be removed to simplify the stylesheet.  

### Future Enhancements  
* **Modern Layout** – Replace floats with Flexbox (`display:flex;`) for more robust alignment and easier vertical/horizontal centering.  
* **CSS Variables** – Define colors and padding as variables for easier theming.  
* **Responsive Design** – Add media queries to collapse the menu or change positioning on narrow viewports.  
* **Semantic Markup** – Use `<nav>` instead of a generic `<div id="nav">` and `<li class="active">` instead of `<strong>` for state indication.  
* **Accessibility** – Apply `role="navigation"` and `aria-current` to improve screen‑reader support.  

Overall, the stylesheet is functional and highly compatible with legacy browsers, but modern CSS techniques and accessibility best practices could further improve its maintainability and user experience.

## Code Critique



## Code Preview

```css
/* A CSS Framework by Mike Stenhouse of Content with Style */

/* NAV */
	div#nav {
        font-size: 0.8em;
    }
    * html div#nav {
        /* hide ie/mac \*/
        height: 1%;
        /* end hide */
    }
    div#nav div.wrapper {
        position: absolute;
        left: 0;
        bottom: 0;
        width: 100%;
    }
    div#nav ul {
        width: auto; 
        width: 100%;
                
        margin: 0;
        padding: 0;
        
        line-height: 1em;
        list-style: none;
    }
    div#nav li {
        float: left;
        display: inline;
   
        list-style: none;
        
        margin: 0;
        padding: 0;
        
        line-height: 1em;
        border-right: 1px solid #aaa;
    }
    div#nav li.last {
        border-right: none;
    }
    div#nav a,
    div#nav a:link,
    div#nav a:active,
    div#nav a:visited {
        display: inline-block;
        /* hide from ie/mac \*/
        display: block;
        /* end hide */
        font-weight: bold;
        text-decoration: none;
        
        margin: 0;
        padding: 5px 38px 5px 38px;
        
        color: black;
        background: #ddd;
    }
    div#nav a:hover {
        text-decoration: underline;
    }
    div#nav strong {
        display: inline-block;
        /* hide from ie/mac \*/
        display: block;
        /* end hide */
        
        color: white;
        background: black;
    }
    div#nav strong a,
    div#nav strong a:link,
    div#nav strong a:active,
    div#nav strong a:visited,
    div#nav strong a:hover {
       color: white;
       background-color: black;
    }
/* END NAV */


```
