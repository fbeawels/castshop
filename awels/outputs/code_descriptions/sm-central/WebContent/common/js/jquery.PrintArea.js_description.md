# jquery.PrintArea.js

## Review

## 1. Summary
The file implements a **jQuery plugin** (`$.fn.printArea`) that renders a specified DOM element (or set of elements) into a printable context.  
It supports two printing modalities:

| Mode | Implementation | Notes |
|------|----------------|-------|
| **iframe** | Creates a hidden iframe, writes the HTML of the target element into it, and invokes the browser’s print dialog. | Best for browsers that allow printing an off‑screen document. |
| **popup** | Opens a new browser window, writes the same HTML into it, and prints. | Useful when a separate window is preferred or when the iframe method is blocked by pop‑up blockers. |

The plugin also:

* Copies relevant `<link rel="stylesheet">` tags (optionally filtering by `media="print"`).  
* Handles form controls (radio/checkbox, select, textarea) so that their current state is reflected in the printout.  
* Allows a set of configuration options (size/position of popup, title, `strict` HTML4 vs Transitional, etc.).  

A small wrapper of a custom `Iframe` and `Popup` constructor objects encapsulates the creation of the print context.

## 2. Detailed Description

### 2.1 Core Flow
1. **Invocation** – The plugin is called on a jQuery collection (e.g., `$("div.PrintArea").printArea(options)`).
2. **Option Merging** – Options are merged into a global `settings` object with defaults.
3. **Element Preparation** –  
   * `getFormData` traverses all form controls within the target and ensures their current values (checked, selected, text content) are persisted on the DOM itself.  
   * The collection (`this`) is passed to `getBody` to wrap it in a `<div>` with the same class attribute.
4. **Print Context Creation** –  
   * If `mode === "iframe"`, an `Iframe` instance is created.  
   * If `mode === "popup"`, a `Popup` instance is created.  
   Both expose a `doc` property pointing to the writable `document`.
5. **HTML Generation** –  
   * `docType` supplies a doctype when requested.  
   * `getHead` assembles a `<head>` containing the title and all matching stylesheet links.  
   * `getBody` builds the `<body>` section containing the target element.  
6. **Print Execution** –  
   * The document is written (`writeDoc.open/writeDoc.write/writeDoc.close`).  
   * The print context is focused and `print()` is called.  
   * For popup mode, an optional close after printing (`popClose`) is honored.

### 2.2 Assumptions & Constraints
* **Browser Compatibility** – The code was originally written for IE 8 and FF 3.6. Modern browsers generally support iframes and `window.print()`, but the code uses deprecated `$.browser` and assumes pop‑up blockers are disabled.  
* **Global Settings** – The `settings` object is shared across all plugin invocations, making concurrent prints on the same page potentially interleaved.  
* **Form Data Preservation** – Only form elements within the target are considered; any dynamic state outside that scope is ignored.  
* **Print Area Wrapping** – The printed content is always wrapped in a `<div>` that copies the class of the target element, which may alter layout if CSS targets the original element directly.  
* **Stylesheet Inclusion** – All `<link rel="stylesheet">` tags on the page are considered; those that are not `media="print"` are filtered out only if the `media` attribute is present.

### 2.3 Architecture
* **Plugin API** – Exposes a single method `printArea` on jQuery objects.  
* **Helper Functions** – Small, self‑contained utilities (`docType`, `getHead`, `getBody`, `getFormData`) handle string construction and form manipulation.  
* **Constructor Objects** – `Iframe` and `Popup` encapsulate the differences between the two printing modalities, each exposing a `doc` property for the writing context.  

The design keeps the main `printArea` logic concise and defers specific tasks to helper functions.

## 3. Functions/Methods

| Name | Purpose | Parameters | Returns | Side Effects |
|------|---------|------------|---------|--------------|
| `$.fn.printArea(options)` | Plugin entry point; renders selected elements to a printable context. | `options` – JSON object of user overrides. | None (side‑effect: initiates print). | Modifies global `settings`; creates iframe/window; writes documents; triggers print. |
| `docType()` | Generates an optional doctype string based on `settings.strict`. | None | `string` | None |
| `getHead()` | Builds `<head>` section with title and appropriate stylesheet links. | None | `string` | Reads `<link>` elements from the document. |
| `getBody(printElement)` | Wraps the target element in a `<div>` and returns a `<body>` string. | `printElement` – jQuery collection of the target. | `string` | None |
| `getFormData(ele)` | Traverses form controls inside `ele`, persisting their state on the DOM. | `ele` – jQuery collection. | `jQuery` collection (same `ele`). | Modifies element attributes (`checked`, `selected`, `value`). |
| `Iframe()` | Creates a hidden iframe, attaches it to the body, and exposes its `document`. | None | `HTMLIFrameElement` with added `doc` property. | Manipulates DOM; may throw if iframes are unsupported. |
| `Popup()` | Opens a new window with specified attributes, returns it with an exposed `doc`. | None | `Window` object with `doc` property. | Calls `window.open`; may be blocked by pop‑up blockers. |

### Reusable / Utility Methods
* **`docType`**, **`getHead`**, **`getBody`** – Pure string builders; could be unit‑tested independently.  
* **`Iframe`** & **`Popup`** – Factory functions that could be refactored into a more generic “PrintContext” abstraction.

## 4. Dependencies

| Library / API | Type | Notes |
|---------------|------|-------|
| **jQuery** | Third‑party | Core dependency; plugin uses `$` alias. |
| `$.browser` | Deprecated (jQuery 1.9+) | Used in `getFormData` for Mozilla detection; should be replaced with feature detection or a modern polyfill. |
| **DOM APIs** | Standard | `document.createElement`, `document.body.appendChild`, `window.open`, `element.contentDocument`, etc. |
| **CSS Media** | Standard | Filtered by `media="print"` when building `<head>`. |
| **HTML5** | Standard | Uses `innerHTML` and `textContent` for textarea content. |

No other external frameworks are required.

## 5. Additional Notes & Recommendations

### 5.1 Edge Cases / Potential Issues
1. **Concurrent Invocations** – Since `settings` is global, overlapping prints will override each other’s options.  
2. **IFrame Visibility** – The iframe is hidden via CSS but still appended to the body; older browsers might scroll it into view.  
3. **Form State Persistence** – The plugin modifies the original DOM. If the user expects the page to remain unchanged after printing, a deep clone would be safer.  
4. **Print Styles** – Only external stylesheets are copied; inline styles, `<style>` tags, and CSS rules that target specific IDs or classes may be lost.  
5. **Pop‑up Blocking** – Modern browsers often block `window.open` unless triggered by a direct user action; calling from a non‑click context could fail.  
6. **`$.browser` Deprecation** – The check for Mozilla will always be `false` in newer jQuery versions, causing incorrect textarea handling.

### 5.2 Suggested Enhancements
| Category | Recommendation |
|----------|----------------|
| **Configuration Isolation** | Replace the global `settings` with a local copy per invocation (e.g., `var settings = $.extend({}, defaults, options);`). |
| **Form Handling** | Clone the target element (`$(this).clone(true)`) and modify the clone, leaving the original untouched. |
| **Style Inclusion** | Extend `getHead` to also include `<style>` tags and inline styles, or let the user supply custom CSS via options. |
| **Feature Detection** | Remove `$.browser`; use `navigator.userAgent` or better, detect `textarea.firstChild && 'textContent' in textarea`. |
| **Error Handling** | Gracefully handle pop‑up blockers by checking `newWin` after `window.open` and notifying the user. |
| **Modernization** | Rewrite with ES6+ syntax, use `const/let`, and avoid global variables. |
| **Testing** | Add unit tests for helper functions and integration tests across major browsers. |
| **Documentation** | Provide clearer API docs and example usage (including handling of multiple elements). |

### 5.3 Future Extensions
* **Print Preview** – Offer a modal preview window before invoking `print()`.  
* **Multi‑Page Support** – Handle overflow and page breaks using CSS `@media print`.  
* **Accessibility** – Ensure the printed content retains ARIA roles and accessible labels.  
* **Server‑Side Rendering** – Offer a fallback that generates a printable PDF on the server.  

---

**Verdict** – The plugin is functional for its intended legacy browsers, but it relies on several outdated patterns. Refactoring to isolate configuration, preserve original DOM, and modernize browser feature checks would greatly improve robustness and maintainability.

## Code Critique



## Code Preview

```javascript
/**
 *  Version 2.1
 *      -Contributors: "mindinquiring" : filter to exclude any stylesheet other than print.
 *  Tested ONLY in IE 8 and FF 3.6. No official support for other browsers, but will
 *      TRY to accomodate challenges in other browsers.
 *  Example:
 *      Print Button: <div id="print_button">Print</div>
 *      Print Area  : <div class="PrintArea"> ... html ... </div>
 *      Javascript  : <script>
 *                       $("div#print_button").click(function(){
 *                           $("div.PrintArea").printArea( [OPTIONS] );
 *                       });
 *                     </script>
 *  options are passed as json (json example: {mode: "popup", popClose: false})
 *
 *  {OPTIONS} | [type]    | (default), values      | Explanation
 *  --------- | --------- | ---------------------- | -----------
 *  @mode     | [string]  | ("iframe"),"popup"     | printable window is either iframe or browser popup
 *  @popHt    | [number]  | (500)                  | popup window height
 *  @popWd    | [number]  | (400)                  | popup window width
 *  @popX     | [number]  | (500)                  | popup window screen X position
 *  @popY     | [number]  | (500)                  | popup window screen Y position
 *  @popTitle | [string]  | ('')                   | popup window title element
 *  @popClose | [boolean] | (false),true           | popup window close after printing
 *  @strict   | [boolean] | (undefined),true,false | strict or loose(Transitional) html 4.01 document standard or undefined to not include at all (only for popup option)
 */
(function($) {
    var counter = 0;
    var modes = { iframe : "iframe", popup : "popup" };
    var defaults = { mode     : modes.iframe,
                     popHt    : 500,
                     popWd    : 400,
                     popX     : 200,
                     popY     : 200,
                     popTitle : '',
                     popClose : false };

    var settings = {};//global settings

    $.fn.printArea = function( options )
        {
            $.extend( settings, defaults, options );

            counter++;
            var idPrefix = "printArea_";
            $( "[id^=" + idPrefix + "]" ).remove();
            var ele = getFormData( $(this) );

            settings.id = idPrefix + counter;

            var writeDoc;
            var printWindow;

            switch ( settings.mode )
            {
                case modes.iframe :
                    var f = new Iframe();
                    writeDoc = f.doc;
                    printWindow = f.contentWindow || f;
                    break;
                case modes.popup :
                    printWindow = new Popup();
                    writeDoc = printWindow.doc;
            }

            writeDoc.open();
            writeDoc.write( docType() + "<html>" + getHead() + getBody(ele) + "</html>" );
            writeDoc.close();

            printWindow.focus();
            printWindow.print();

            if ( settings.mode == modes.popup && settings.popClose )
                printWindow.close();
        }

    function docType()
    {
        if ( settings.mode == modes.iframe || !settings.strict ) return "";

        var standard = settings.strict == false ? " Trasitional" : "";
        var dtd = settings.strict == false ? "loose" : "strict";

        return '<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01' + standard + '//EN" "http://www.w3.org/TR/html4/' + dtd +  '.dtd">';
    }

    function getHead()
    {
        var head = "<head><title>" + settings.popTitle + "</title>";
        $(document).find("link")
            .filter(function(){
                    return $(this).attr("rel").toLowerCase() == "stylesheet";
                })
            .filter(function(){ // this filter contributed by "mindinquiring"
                    var media = $(this).attr("media");
                    return (media.toLowerCase() == "" || media.toLowerCase() == "print")
                })
            .each(function(){
                    head += '<link type="text/css" rel="stylesheet" href="' + $(this).attr("href") + '" >';
                });
        head += "</head>";
        return head;
    }

    function getBody( printElement )
    {
        return '<body><div class="' + $(printElement).attr("class") + '">' + $(printElement).html() + '</div></body>';
    }

    function getFormData( ele )
    {
        $("input,select,textarea", ele).each(function(){
            // In cases where radio, checkboxes and select elements are selected and deselected, and the print
            // button is pressed between select/deselect, the print screen shows incorrectly selected elements.
            // To ensure that the correct inputs are selected, when eventually printed, we must inspect each dom element
            var type = $(this).attr("type");
            if ( type == "radio" || type == "checkbox" )
            {
                if ( $(this).is(":not(:checked)") ) this.removeAttribute("checked");
                else this.setAttribute( "checked", true );
            }
            else if ( type == "text" )
                this.setAttribute( "value", $(this).val() );
            else if ( type == "select-multiple" || type == "select-one" )
                $(this).find( "option" ).each( function() {
                    if ( $(this).is(":not(:selected)") ) this.removeAttribute("selected");
                    else this.setAttribute( "selected", true );
                });
            else if ( type == "textarea" )
            {
                var v = $(this).attr( "value" );
                if ($.browser.mozilla)
                {
                    if (this.firstChild) this.firstChild.textContent = v;
                    else this.textContent = v;
                }
                else this.innerHTML = v;
            }
        });
        return ele;
    }

    function Iframe()
    {
        var frameId = settings.id;
        var iframeStyle = 'border:0;position:absolute;width:0px;height:0px;left:0px;top:0px;';
        var iframe;

        try
        {
            iframe = document.createElement('iframe');
            document.body.appendChild(iframe);
            $(iframe).attr({ style: iframeStyle, id: frameId, src: "" });
            iframe.doc = null;
            iframe.doc = iframe.contentDocument ? iframe.contentDocument : ( iframe.contentWindow ? iframe.contentWindow.document : iframe.document);
        }
        catch( e ) { throw e + ". iframes may not be supported in this browser."; }

        if ( iframe.doc == null ) throw "Cannot find document.";

        return iframe;
    }

    function Popup()
    {
        var windowAttr = "location=yes,statusbar=no,directories=no,menubar=no,titlebar=no,toolbar=no,dependent=no";
        windowAttr += ",width=" + settings.popWd + ",height=" + settings.popHt;
        windowAttr += ",resizable=yes,screenX=" + settings.popX + ",screenY=" + settings.popY + ",personalbar=no,scrollbars=no";

        var newWin = window.open( "", "_blank",  windowAttr );

        newWin.doc = newWin.document;

        return newWin;
    }
})(jQuery);


```
