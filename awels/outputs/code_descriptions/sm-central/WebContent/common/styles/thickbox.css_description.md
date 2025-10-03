# thickbox.css

## Review

## 1. Summary
- **Purpose** – This stylesheet provides the visual styling for the **Thickbox** modal/light‑box component. It defines the overlay, modal window, captions, navigation links, loading indicator, and various IE6 hacks.
- **Key components**  
  - Global reset (`*{padding:0;margin:0;}`) to neutralise browser defaults.  
  - Style rules for the overlay (`#TB_overlay`), modal window (`#TB_window`), image, captions, close buttons, and AJAX content.  
  - A set of *IE6 hacks* that use conditional CSS selectors (`* html …`) to compensate for lack of support for `position:fixed` and other properties.  
- **Design patterns / frameworks** – The code is pure CSS, relying on standard selectors and properties. It assumes the accompanying Thickbox JavaScript library (not shown) that injects the IDs/classes referenced here.  

## 2. Detailed Description
1. **Reset & Base**  
   ```css
   *{padding: 0; margin: 0;}
   ```  
   Removes default margin/padding from all elements, ensuring consistent layout across browsers.

2. **Overlay** (`#TB_overlay`)  
   - Covers the entire viewport (`position:fixed; top:0; left:0; width:100%; height:100%`).  
   - Semi‑transparent black background via `opacity:0.75` and `filter:alpha(opacity=75)` for legacy IE.  
   - IE6 fallback (`* html #TB_overlay`) uses `expression` to set height to the document body’s scroll height.

3. **Modal Window** (`#TB_window`)  
   - Positioned at the centre of the viewport with `top:50%; left:50%` and `position:fixed`.  
   - Hidden by default (`display:none`).  
   - Styled with a 4‑pixel solid border and background white.  
   - IE6 fallback sets absolute positioning and a negative top margin to achieve vertical centering.

4. **Content & Controls**  
   - Image styling (`#TB_Image`) with borders and margin.  
   - Caption (`#TB_caption`) and close button (`#TB_closeWindow`) floats left/right.  
   - AJAX title (`#TB_ajaxWindowTitle`), caption (`#TB_title`), and content (`#TB_ajaxContent`) arranged with padding, overflow handling, and clear‑fixing.  
   - Modal content uses `.TB_modal` to change padding.  

5. **Loading Indicator** (`#TB_load`)  
   - Fixed positioned element that appears over the modal, centred with negative margins.  
   - Hidden by default (`display:none`).  

6. **Select Box Hack** (`#TB_HideSelect`)  
   - Covers the entire viewport with a fully transparent element to prevent legacy IE select boxes from bleeding through the overlay.  

7. **IFrame Content** (`#TB_iframeContent`)  
   - Basic reset for embedded iframes, mainly clearing margin.  

**Execution Flow** – When Thickbox is invoked, the JS script will inject the relevant DOM elements (`#TB_overlay`, `#TB_window`, etc.), set `display:block`, and populate the modal content. The CSS then renders the modal, overlay, and loading indicator accordingly. When the modal is closed, the JS removes or hides the elements again. No cleanup logic is needed from the CSS side.

**Assumptions & Constraints**  
- Browser support is aimed at all modern browsers plus IE6.  
- Requires a viewport size large enough for the overlay to cover the page; the modal relies on `position:fixed`.  
- Assumes the JavaScript layer manages the `display` state and sets appropriate classes (e.g., `.TB_modal`).

## 3. Functions/Methods
The file contains only CSS rules, so there are no functions or methods in the traditional programming sense. However, the **rules** act as implicit “style functions” that apply to the DOM elements:

| Selector | Purpose | Key Properties |
|----------|---------|----------------|
| `*` | Global reset | `padding:0; margin:0;` |
| `#TB_overlay` | Dark translucent overlay | `position:fixed; z-index:100; background-color:#000; opacity:0.75;` |
| `#TB_window` | Modal container | `position:fixed; z-index:102; border:4px solid #525252;` |
| `#TB_caption`, `#TB_closeWindow`, `#TB_ajaxWindowTitle`, `#TB_title`, `#TB_ajaxContent` | Layout of modal header and body | Various padding, float, clear, line-height |
| `#TB_load` | Loading spinner | `position:fixed; z-index:103;` |
| `#TB_HideSelect` | Prevents IE6 select box bleed | `position:fixed; opacity:0;` |
| `#TB_iframeContent` | Styling for embedded iframes | `border:none; margin-bottom:-1px;` |
| `* html …` rules | IE6 hacks | Use of `expression()` for dynamic positioning and sizing |

These “functions” are pure style declarations; they have no side‑effects beyond altering appearance.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| **Thickbox JavaScript** | Third‑party library | The CSS is designed to work with the Thickbox plugin; the JS creates and manages the elements referenced here. |
| **jQuery** | Third‑party (commonly bundled with Thickbox) | While not directly referenced in this file, Thickbox typically uses jQuery for DOM manipulation. |
| **IE6** | Legacy browser | Several hacks (`* html`, `expression()`, `filter:alpha`) target IE6 specifically. |

No external CSS frameworks (e.g., Bootstrap) or preprocessors are used.

## 5. Additional Notes
- **Maintainability** – The code is largely self‑contained, but the heavy reliance on IE6 hacks may become problematic if the project ever needs to drop legacy browser support. Removing those hacks would simplify the stylesheet.
- **Scalability** – If the modal needs to support additional features (e.g., video players, galleries), new selectors should be added carefully to avoid conflicts. Consider namespacing styles (e.g., `.thickbox-` prefixes) to prevent clashes with other components.
- **Accessibility** – The CSS does not provide any ARIA attributes or focus handling. Accessibility should be managed by the JavaScript layer, but ensuring focus traps and `role="dialog"` is essential for screen reader users.
- **Performance** – All styles are scoped to specific IDs, which is efficient. However, the use of `position:fixed` may cause re‑painting overhead on mobile browsers; profiling could confirm whether this is a concern.
- **Future Enhancements**  
  - Replace IE6 hacks with conditional comments or separate IE6‑only stylesheets.  
  - Add transitions/animations for opening/closing the modal.  
  - Implement a dark‑mode variant via CSS variables.  
  - Provide fallback loading indicators for slow network conditions.

Overall, the stylesheet is concise, purpose‑driven, and aligns well with the Thickbox plugin’s DOM structure. Its primary limitation is the outdated IE6 support, which could be safely removed if the project targets modern browsers.

## Code Critique



## Code Preview

```css
/* ----------------------------------------------------------------------------------------------------------------*/
/* ---------->>> global settings needed for thickbox <<<-----------------------------------------------------------*/
/* ----------------------------------------------------------------------------------------------------------------*/
*{padding: 0; margin: 0;}

/* ----------------------------------------------------------------------------------------------------------------*/
/* ---------->>> thickbox specific link and font settings <<<------------------------------------------------------*/
/* ----------------------------------------------------------------------------------------------------------------*/
#TB_window {
	font: 12px Arial, Helvetica, sans-serif;
	color: #333333;
}

#TB_secondLine {
	font: 10px Arial, Helvetica, sans-serif;
	color:#666666;
}

#TB_window a:link {color: #666666;}
#TB_window a:visited {color: #666666;}
#TB_window a:hover {color: #000;}
#TB_window a:active {color: #666666;}
#TB_window a:focus{color: #666666;}

/* ----------------------------------------------------------------------------------------------------------------*/
/* ---------->>> thickbox settings <<<-----------------------------------------------------------------------------*/
/* ----------------------------------------------------------------------------------------------------------------*/
#TB_overlay {
	position: fixed;
	z-index:100;
	top: 0px;
	left: 0px;
	height:100%;
	width:100%;
}

.TB_overlayMacFFBGHack {background: url(../common/img/macFFBgHack.png) repeat;}
.TB_overlayBG {
	background-color:#000;
	filter:alpha(opacity=75);
	-moz-opacity: 0.75;
	opacity: 0.75;
}

* html #TB_overlay { /* ie6 hack */
     position: absolute;
     height: expression(document.body.scrollHeight > document.body.offsetHeight ? document.body.scrollHeight : document.body.offsetHeight + 'px');
}

#TB_window {
	position: fixed;
	background: #ffffff;
	z-index: 102;
	color:#000000;
	display:none;
	border: 4px solid #525252;
	text-align:left;
	top:50%;
	left:50%;
}

* html #TB_window { /* ie6 hack */
position: absolute;
margin-top: expression(0 - parseInt(this.offsetHeight / 2) + (TBWindowMargin = document.documentElement && document.documentElement.scrollTop || document.body.scrollTop) + 'px');
}

#TB_window img#TB_Image {
	display:block;
	margin: 15px 0 0 15px;
	border-right: 1px solid #ccc;
	border-bottom: 1px solid #ccc;
	border-top: 1px solid #666;
	border-left: 1px solid #666;
}

#TB_caption{
	height:25px;
	padding:7px 30px 10px 25px;
	float:left;
}

#TB_closeWindow{
	height:25px;
	padding:11px 25px 10px 0;
	float:right;
}

#TB_closeAjaxWindow{
	padding:7px 10px 5px 0;
	margin-bottom:1px;
	text-align:right;
	float:right;
}

#TB_ajaxWindowTitle{
	float:left;
	padding:7px 0 5px 10px;
	margin-bottom:1px;
}

#TB_title{
	background-color:#e8e8e8;
	height:27px;
}

#TB_ajaxContent{
	clear:both;
	padding:2px 15px 15px 15px;
	overflow:auto;
	text-align:left;
	line-height:1.4em;
}

#TB_ajaxContent.TB_modal{
	padding:15px;
}

#TB_ajaxContent p{
	padding:5px 0px 5px 0px;
}

#TB_load{
	position: fixed;
	display:none;
	height:13px;
	width:208px;
	z-index:103;
	top: 50%;
	left: 50%;
	margin: -6px 0 0 -104px; /* -height/2 0 0 -width/2 */
}

* html #TB_load { /* ie6 hack */
position: absolute;
margin-top: expression(0 - parseInt(this.offsetHeight / 2) + (TBWindowMargin = document.documentElement && document.documentElement.scrollTop || document.body.scrollTop) + 'px');
}

#TB_HideSelect{
	z-index:99;
	position:fixed;
	top: 0;
	left: 0;
	background-color:#fff;
	border:none;
	filter:alpha(opacity=0);
	-moz-opacity: 0;
	opacity: 0;
	height:100%;
	width:100%;
}

* html #TB_HideSelect { /* ie6 hack */
     position: absolute;
     height: expression(document.body.scrollHeight > document.body.offsetHeight ? document.body.scrollHeight : document.body.offsetHeight + 'px');
}

#TB_iframeContent{
	clear:both;
	border:none;
	margin-bottom:-1px;
	margin-top:1px;
	_margin-bottom:1px;
}



```
