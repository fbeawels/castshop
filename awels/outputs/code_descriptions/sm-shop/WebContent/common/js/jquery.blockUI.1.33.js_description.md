# jquery.blockUI.1.33.js

## Review

## 1. Summary  

| Item | What it is | How it fits into the system |
|------|------------|----------------------------|
| **Purpose** | Prevents user interaction with a page or a selected element while an asynchronous operation (usually Ajax) is in progress. | Acts as a lightweight modal overlay that blocks input and optionally displays a message. |
| **Key Components** | • `$.blockUI` / `$.unblockUI` (global helpers)<br>• `$.fn.block` / `$.fn.unblock` (per‑element helpers)<br>• `$.fn.displayBox` (special “display‑box” mode for images/Flash)<br>• `$.blockUI.defaults` (configurable defaults)<br>• `$.blockUI.impl` (internal implementation object) | The API is split into public surface methods (attached to `$` or `$.fn`) that delegate to the private implementation. |
| **Design patterns** | • **Module/IIFE** – isolates the code from the global namespace.<br>• **Facade** – the public methods provide a simple interface while the complex logic lives in `$.blockUI.impl`.<br>• **Strategy/Template** – the implementation can render either a page‑wide block, an element block, or a special display‑box. | The plugin uses the common jQuery plugin pattern of extending `$.fn` for element‑based calls. |

---

## 2. Detailed Description  

### 2.1 Execution Flow  

| Phase | What happens | Notes |
|-------|--------------|-------|
| **Initialization** | The IIFE runs immediately, receiving the jQuery object. `$.blockUI` and `$.unblockUI` are added to the namespace. | The plugin registers the jQuery methods once at load time. |
| **Blocking a page** | `$.blockUI(msg, css, opts)` calls `$.blockUI.impl.install(window, …)`. | `el == window` triggers “full‑page” mode. |
| **Blocking a selected element** | `$('selector').block(msg, css, opts)` iterates over the jQuery collection, normalises the element’s CSS (`position: relative` + `zoom: 1` for IE) and then calls `install`. | The element is marked with `el.$pos_checked` to avoid repeated style adjustments. |
| **Display‑box mode** | `$('selector').displayBox(css, fn, isFlash)` calculates element dimensions, constructs overlay parameters, and calls `install` with `displayMode`. | Useful for showing large images or Flash objects over the page overlay. |
| **Internal `install` logic** | <ul><li>Creates three DOM nodes: an optional iframe (`f`) for IE, a transparent overlay (`w`) and a message container (`m`).</li><li>Applies style overrides from the defaults, `msg`, and user‑supplied `css`.</li><li>Appends the nodes to the body (for page blocks) or to the target element (for element blocks).</li><li>Sets up event handlers (`bind`) to swallow clicks, keys, etc. unless in display‑box mode.</li></ul> | Uses old IE specific hacks (e.g., `setExpression`, `opacity` trick) to emulate CSS 3 behavior. |
| **Unblocking** | `$.unblockUI(opts)` or `$('selector').unblock(opts)` calls `$.blockUI.impl.remove`. | Elements are removed either instantly or with a fade‑out animation (`fadeTime`). The `bind` function also detaches the event handlers. |
| **Display‑box cleanup** | `boxRemove` (invoked by the global `boxHandler`) hides the overlay and calls a user‑supplied callback (`boxCallback`). | The overlay is hidden by unbinding click/keypress handlers and then removing the DOM elements. |

### 2.2 Assumptions & Constraints  

| Constraint | Description |
|------------|-------------|
| **jQuery ≥ 1.1.1** | The code explicitly requires `$.browser` and the old `$.boxModel` property, both removed after jQuery 1.8. |
| **Browser support** | Written for IE6/7/8, Firefox, Chrome, Safari, and Netscape‑based browsers. Relies heavily on IE‑specific CSS expressions. |
| **No CSS Flexbox/Grid** | The layout is based on absolute positioning and manual calculations. |
| **Modal overlay** | The overlay covers the entire viewport; scrolling is disabled (unless using the older “fixed” style hacks). |
| **No async cleanup** | If a page is navigated away before the overlay is removed, a dangling overlay may remain (rare in practice). |

### 2.3 Architecture & Design Choices  

* **Encapsulation** – the implementation lives in `$.blockUI.impl`, exposing only a very small API.  
* **Dynamic Element Creation** – the plugin builds the overlay and message elements on‑the‑fly rather than reusing a single set.  
* **IE Fallbacks** – the use of an iframe (`f`) is to shield the overlay from select elements (`select`, `object`, `embed`) in older IE.  
* **Event Swallowing** – a custom `handler` function ensures that focus, keyboard, and mouse events are captured and discarded unless the event originates inside the message container.  
* **Legacy Browser Work‑arounds** – use of `setExpression`, `zoom`, and `filter: alpha(opacity=…)` indicates a design that prioritised compatibility over modern standards.

---

## 3. Functions / Methods  

| Function / Method | Signature | Purpose | Inputs | Outputs | Side‑effects |
|--------------------|-----------|---------|--------|---------|--------------|
| **`$.blockUI(msg, css, opts)`** | `(String|Element|jQuery, Object, Object)` | Public entry‑point to block the whole page. | `msg` – message element or string.<br>`css` – optional style overrides.<br>`opts` – optional options (e.g., `displayMode`, `fadeOut`). | None (renders overlay). | Appends overlay elements to `<body>`; sets global handlers. |
| **`$.unblockUI(opts)`** | `(Object)` | Public entry‑point to unblock the whole page. | `opts` – optional options (e.g., `fadeOut`). | None (removes overlay). | Removes overlay elements; detaches handlers. |
| **`$.fn.block(msg, css, opts)`** | `(String|Element|jQuery, Object, Object)` | Blocks a selected element(s). | Same as above. | None. | Adds overlay to each matched element. |
| **`$.fn.unblock(opts)`** | `(Object)` | Unblocks selected element(s). | `opts` – optional options. | None. | Removes overlay from each matched element. |
| **`$.fn.displayBox(css, fn, isFlash)`** | `(Object, Function, Boolean)` | Shows an element (image/Flash) in a modal box over the page. | `css` – style overrides for box.<br>`fn` – optional callback on close.<br>`isFlash` – flag for Flash content. | None. | Creates overlay with click/keypress handler to close. |
| **`$.blockUI.defaults`** | *object* | Default configuration object. | N/A | N/A | Can be overridden by user. |
| **`$.blockUI.impl.install(el, msg, css, opts)`** | `(HTMLElement|Window, *, Object, Object)` | Internal routine that builds and displays the overlay. | `el` – target element or `window`. | None. | Creates and appends DOM nodes; sets event handlers. |
| **`$.blockUI.impl.remove(el, opts)`** | `(HTMLElement|Window, Object)` | Internal routine that removes the overlay. | `el` – target. | None. | Removes DOM nodes; detaches handlers. |
| **`$.blockUI.impl.boxRemove(el)`** | `(HTMLElement)` | Internal routine to close display‑box overlays. | `el` – target element. | None. | Hides overlay; calls user callback. |
| **`$.blockUI.impl.handler(e)`** | `(Event)` | Global event handler that swallows most events while blocked. | `e` – event object. | Boolean – whether to allow event propagation. | Prevents focus, key navigation, etc. |
| **`$.blockUI.impl.boxHandler(e)`** | `(Event)` | Handles key‑press (Esc) or click to close a display‑box. | `e` – event object. | Boolean – whether to allow event propagation. | Triggers `boxRemove`. |
| **`$.blockUI.impl.bind(b, el)`** | `(Boolean, HTMLElement)` | Binds or unbinds the `handler` to child elements. | `b` – `true` to bind, `false` to unbind. | None. | Attaches/detaches event listeners. |
| **`$.blockUI.impl.focus(back)`** | `(Boolean)` | Sets focus to the first/last input in the blocked form. | `back` – `true` to focus last. | None. | Calls `.focus()` on the target input. |
| **`$.blockUI.impl.center(el)`** | `(HTMLElement)` | Centers the message container within its parent. | `el` – message element. | None. | Adjusts `left`/`top` style properties. |
| **`$.blockUI.impl.sz(el, p)`** | `(HTMLElement, String)` | Parses a CSS dimension property (e.g., `borderTopWidth`). | `el` – element.<br>`p` – CSS property name. | Integer (px). | None. |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery ≥ 1.1.1** | Third‑party library | Uses `$.browser`, `$.boxModel`, `$.css`, `$.extend`, `$.fn`, and the event API. Modern jQuery (≥1.9) removed many of these, making the plugin incompatible without polyfills. |
| **Browser-specific CSS/JS** | Platform‑specific | Relies on IE filters, `setExpression`, `zoom`, and `iframe` tricks to cover old browsers. |
| **Optional Flash** | Legacy plugin | `displayBox` accepts a flag for Flash objects; modern browsers no longer support Flash. |
| **No external APIs** | – | The plugin is self‑contained. |

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality & Maintainability  

* **Obsolete APIs** – `$.browser`, `$.boxModel`, and the jQuery 1.1 `filter` style are all deprecated. Modern code should detect features instead of hard‑coding browser checks.  
* **Hard‑coded z‑indexes** – All overlays use `z-index: 1000/1001/1002`. In complex applications this can clash with other components. Allowing the user to override via `opts.zIndex` would be safer.  
* **Inline Styles** – The plugin creates elements with many inline styles (e.g., `style="display:none"`). Separating these into CSS classes would simplify debugging and theming.  
* **No Namespacing of Events** – Event listeners are attached directly (`bind`/`unbind`) without namespacing. If multiple instances of the plugin run on the same page, handlers could interfere. Using event namespacing (`'click.blockUI'`) is advisable.  
* **No Modularisation** – The code is a monolithic IIFE. Modern tooling (ES6 modules, Webpack) would allow tree‑shaking and easier testing.  

### 5.2 Compatibility & Edge Cases  

| Edge Case | Current Behaviour | Potential Issue |
|-----------|-------------------|-----------------|
| **Multiple simultaneous blocks** | Each `install` call appends new elements; the overlay is appended again. | If a second block is applied before the first is removed, you end up with stacked overlays that may interfere. |
| **Dynamic content changes** | `center` calculates position only once. If the target element resizes (e.g., responsive layout), the message may drift. | Need a resize observer or manual re‑center on window resize. |
| **Touch devices** | Event handlers rely on `mousedown`, `mouseup`, `keypress`, etc. Touch events are not captured, potentially allowing interaction. | Add `touchstart`/`touchend` handlers to swallow touch. |
| **Accessibility** | Focus is set to the first input inside the overlay but not to the overlay itself, meaning screen readers may still read the page. | Provide `aria-modal="true"` and set focus on the overlay container. |
| **Flash removal** | `displayBox` uses `isFlash` flag; Flash is no longer supported in browsers. | The feature is obsolete and can be removed. |
| **IE6** | Uses `zoom:1` and `setExpression` to emulate fixed positioning. | No longer needed; the code can be simplified. |

### 5.3 Potential Enhancements  

1. **Modernize** – Rewrite to use jQuery ≥ 1.9+ API or vanilla JS; drop `$.browser` checks.  
2. **Event namespacing** – Use `$(element).on('click.blockUI', handler)` for easier cleanup.  
3. **Configurable z-index & class names** – Let the user override overlay, message, and iframe CSS classes.  
4. **Responsive & Touch support** – Add resize listeners and touch event suppression.  
5. **Accessibility** – Add ARIA roles, set focus to the overlay, and allow keyboard navigation to close.  
6. **Modular build** – Convert the IIFE to an ES module or UMD wrapper for better bundling.  
7. **Testing** – Write unit tests (e.g., with QUnit) to cover blocking/unblocking, overlay visibility, and style overrides.  

### 5.4 Summary of Strengths  

* Very small footprint and straightforward API.  
* Works on a wide range of legacy browsers thanks to extensive IE hacks.  
* Provides both full‑page and element‑level blocking with optional display‑box mode.  

### 5.5 Summary of Weaknesses  

* Relies on deprecated jQuery features; incompatible with modern builds.  
* Lacks touch, accessibility, and responsive design considerations.  
* Event handling is fragile (no namespacing).  
* Hard‑coded style values make customization clunky.  

---  

**Conclusion:**  
The plugin serves its original purpose well on older browsers, but it is largely outdated. A rewrite that removes IE‑specific hacks, embraces modern CSS (e.g., `position: fixed`, `opacity`), and follows best practices for event handling and accessibility would significantly improve maintainability, security, and user experience.

## Code Critique



## Code Preview

```javascript
/*
 * jQuery blockUI plugin
 * Version 1.33  (09/14/2007)
 * @requires jQuery v1.1.1
 *
 * $Id: jquery.blockUI.js 3291 2007-09-14 23:56:25Z malsup $
 *
 * Examples at: http://malsup.com/jquery/block/
 * Copyright (c) 2007 M. Alsup
 * Dual licensed under the MIT and GPL licenses:
 * http://www.opensource.org/licenses/mit-license.php
 * http://www.gnu.org/licenses/gpl.html
 */
 (function($) {
/**
 * blockUI provides a mechanism for blocking user interaction with a page (or parts of a page).
 * This can be an effective way to simulate synchronous behavior during ajax operations without
 * locking the browser.  It will prevent user operations for the current page while it is
 * active ane will return the page to normal when it is deactivate.  blockUI accepts the following
 * two optional arguments:
 *
 *   message (String|Element|jQuery): The message to be displayed while the UI is blocked. The message
 *              argument can be a plain text string like "Processing...", an HTML string like
 *              "<h1><img src="busy.gif" /> Please wait...</h1>", a DOM element, or a jQuery object.
 *              The default message is "<h1>Please wait...</h1>"
 *
 *   css (Object):  Object which contains css property/values to override the default styles of
 *              the message.  Use this argument if you wish to override the default
 *              styles.  The css Object should be in a format suitable for the jQuery.css
 *              function.  For example:
 *              $.blockUI({
 *                    backgroundColor: '#ff8',
 *                    border: '5px solid #f00,
 *                    fontWeight: 'bold'
 *              });
 *
 * The default blocking message used when blocking the entire page is "<h1>Please wait...</h1>"
 * but this can be overridden by assigning a value to $.blockUI.defaults.pageMessage in your
 * own code.  For example:
 *
 *      $.blockUI.defaults.pageMessage = "<h1>Bitte Wartezeit</h1>";
 *
 * The default message styling can also be overridden.  For example:
 *
 *      $.extend($.blockUI.defaults.pageMessageCSS, { color: '#00a', backgroundColor: '#0f0' });
 *
 * The default styles work well for simple messages like "Please wait", but for longer messages
 * style overrides may be necessary.
 *
 * @example  $.blockUI();
 * @desc prevent user interaction with the page (and show the default message of 'Please wait...')
 *
 * @example  $.blockUI( { backgroundColor: '#f00', color: '#fff'} );
 * @desc prevent user interaction and override the default styles of the message to use a white on red color scheme
 *
 * @example  $.blockUI('Processing...');
 * @desc prevent user interaction and display the message "Processing..." instead of the default message
 *
 * @name blockUI
 * @param String|jQuery|Element message Message to display while the UI is blocked
 * @param Object css Style object to control look of the message
 * @cat Plugins/blockUI
 */
$.blockUI = function(msg, css, opts) {
    $.blockUI.impl.install(window, msg, css, opts);
};

// expose version number so other plugins can interogate
$.blockUI.version = 1.33;

/**
 * unblockUI removes the UI block that was put in place by blockUI
 *
 * @example  $.unblockUI();
 * @desc unblocks the page
 *
 * @name unblockUI
 * @cat Plugins/blockUI
 */
$.unblockUI = function(opts) {
    $.blockUI.impl.remove(window, opts);
};

/**
 * Blocks user interaction with the selected elements.  (Hat tip: Much of
 * this logic comes from Brandon Aaron's bgiframe plugin.  Thanks, Brandon!)
 * By default, no message is displayed when blocking elements.
 *
 * @example  $('div.special').block();
 * @desc prevent user interaction with all div elements with the 'special' class.
 *
 * @example  $('div.special').block('Please wait');
 * @desc prevent user interaction with all div elements with the 'special' class
 * and show a message over the blocked content.
 *
 * @name block
 * @type jQuery
 * @param String|jQuery|Element message Message to display while the element is blocked
 * @param Object css Style object to control look of the message
 * @cat Plugins/blockUI
 */
$.fn.block = function(msg, css, opts) {
    return this.each(function() {
		if (!this.$pos_checked) {
            if ($.css(this,"position") == 'static')
                this.style.position = 'relative';
            if ($.browser.msie) this.style.zoom = 1; // force 'hasLayout' in IE
            this.$pos_checked = 1;
        }
        $.blockUI.impl.install(this, msg, css, opts);
    });
};

/**
 * Unblocks content that was blocked by "block()"
 *
 * @example  $('div.special').unblock();
 * @desc unblocks all div elements with the 'special' class.
 *
 * @name unblock
 * @type jQuery
 * @cat Plugins/blockUI
 */
$.fn.unblock = function(opts) {
    return this.each(function() {
        $.blockUI.impl.remove(this, opts);
    });
};

/**
 * displays the first matched element in a "display box" above a page overlay.
 *
 * @example  $('#myImage').displayBox();
 * @desc displays "myImage" element in a box
 *
 * @name displayBox
 * @type jQuery
 * @cat Plugins/blockUI
 */
$.fn.displayBox = function(css, fn, isFlash) {
    var msg = this[0];
    if (!msg) return;
    var $msg = $(msg);
    css = css || {};

    var w = $msg.width()  || $msg.attr('width')  || css.width  || $.blockUI.defaults.displayBoxCSS.width;
    var h = $msg.height() || $msg.attr('height') || css.height || $.blockUI.defaults.displayBoxCSS.height ;
    if (w[w.length-1] == '%') {
        var ww = document.documentElement.clientWidth || document.body.clientWidth;
        w = parseInt(w) || 100;
        w = (w * ww) / 100;
    }
    if (h[h.length-1] == '%') {
        var hh = document.documentElement.clientHeight || document.body.clientHeight;
        h = parseInt(h) || 100;
        h = (h * hh) / 100;
    }

    var ml = '-' + parseInt(w)/2 + 'px';
    var mt = '-' + parseInt(h)/2 + 'px';

    // supress opacity on overlay if displaying flash content on mac/ff platform
    var ua = navigator.userAgent.toLowerCase();
    var opts = {
        displayMode: fn || 1,
        noalpha: isFlash && /mac/.test(ua) && /firefox/.test(ua)
    };

    $.blockUI.impl.install(window, msg, { width: w, height: h, marginTop: mt, marginLeft: ml }, opts);
};


// override these in your code to change the default messages and styles
$.blockUI.defaults = {
    // the message displayed when blocking the entire page
    pageMessage:    '<h1>Please wait...</h1>',
    // the message displayed when blocking an element
    elementMessage: '', // none
    // styles for the overlay iframe
    overlayCSS:  { backgroundColor: '#fff', opacity: '0.5' },
    // styles for the message when blocking the entire page
    pageMessageCSS:    { width:'250px', margin:'-50px 0 0 -125px', top:'50%', left:'50%', textAlign:'center', color:'#000', backgroundColor:'#fff', border:'3px solid #aaa' },
    // styles for the message when blocking an element
    elementMessageCSS: { width:'250px', padding:'10px', textAlign:'center', backgroundColor:'#fff'},
    // styles for the displayBox
    displayBoxCSS: { width: '400px', height: '400px', top:'50%', left:'50%' },
    // allow body element to be stetched in ie6
    ie6Stretch: 1,
    // supress tab nav from leaving blocking content?
    allowTabToLeave: 0,
    // Title attribute for overlay when using displayBox
    closeMessage: 'Click to close',
    // use fadeOut effect when unblocking (can be overridden on unblock call)
    fadeOut:  1,
    // fadeOut transition time in millis
    fadeTime: 400
};

// the gory details
$.blockUI.impl = {
    box: null,
    boxCallback: null,
    pageBlock: null,
    pageBlockEls: [],
    op8: window.opera && window.opera.version() < 9,
    ie6: $.browser.msie && /MSIE 6.0/.test(navigator.userAgent),
    install: function(el, msg, css, opts) {
        opts = opts || {};
        this.boxCallback = typeof opts.displayMode == 'function' ? opts.displayMode : null;
        this.box = opts.displayMode ? msg : null;
        var full = (el == window);

        // use logical settings for opacity support based on browser but allow overrides via opts arg
        var noalpha = this.op8 || $.browser.mozilla && /Linux/.test(navigator.platform);
        if (typeof opts.alphaOverride != 'undefined')
            noalpha = opts.alphaOverride == 0 ? 1 : 0;

        if (full && this.pageBlock) this.remove(window, {fadeOut:0});
        // check to see if we were only passed the css object (a literal)
        if (msg && typeof msg == 'object' && !msg.jquery && !msg.nodeType) {
            css = msg;
            msg = null;
        }
        msg = msg ? (msg.nodeType ? $(msg) : msg) : full ? $.blockUI.defaults.pageMessage : $.blockUI.defaults.elementMessage;
        if (opts.displayMode)
            var basecss = jQuery.extend({}, $.blockUI.defaults.displayBoxCSS);
        else
            var basecss = jQuery.extend({}, full ? $.blockUI.defaults.pageMessageCSS : $.blockUI.defaults.elementMessageCSS);
        css = jQuery.extend(basecss, css || {});
        var f = ($.browser.msie) ? $('<iframe class="blockUI" style="z-index:1000;border:none;margin:0;padding:0;position:absolute;width:100%;height:100%;top:0;left:0" src="javascript:false;"></iframe>')
                                 : $('<div class="blockUI" style="display:none"></div>');
        var w = $('<div class="blockUI" style="z-index:1001;cursor:wait;border:none;margin:0;padding:0;width:100%;height:100%;top:0;left:0"></div>');
        var m = full ? $('<div class="blockUI blockMsg" style="z-index:1002;cursor:wait;padding:0;position:fixed"></div>')
                     : $('<div class="blockUI" style="display:none;z-index:1002;cursor:wait;position:absolute"></div>');
        w.css('position', full ? 'fixed' : 'absolute');
        if (msg) m.css(css);
        if (!noalpha) w.css($.blockUI.defaults.overlayCSS);
        if (this.op8) w.css({ width:''+el.clientWidth,height:''+el.clientHeight }); // lame
        if ($.browser.msie) f.css('opacity','0.0');

        $([f[0],w[0],m[0]]).appendTo(full ? 'body' : el);

        // ie7 must use absolute positioning in quirks mode and to account for activex issues (when scrolling)
        var expr = $.browser.msie && (!$.boxModel || $('object,embed', full ? null : el).length > 0);
        if (this.ie6 || expr) {
            // stretch content area if it's short
            if (full && $.blockUI.defaults.ie6Stretch && $.boxModel)
                $('html,body').css('height','100%');

            // fix ie6 problem when blocked element has a border width
            if ((this.ie6 || !$.boxModel) && !full) {
                var t = this.sz(el,'borderTopWidth'), l = this.sz(el,'borderLeftWidth');
                var fixT = t ? '(0 - '+t+')' : 0;
                var fixL = l ? '(0 - '+l+')' : 0;
            }

            // simulate fixed position
            $.each([f,w,m], function(i,o) {
                var s = o[0].style;
                s.position = 'absolute';
                if (i < 2) {
                    full ? s.setExpression('height','document.body.scrollHeight > document.body.offsetHeight ? document.body.scrollHeight : document.body.offsetHeight + "px"')
                         : s.setExpression('height','this.parentNode.offsetHeight + "px"');
                    full ? s.setExpression('width','jQuery.boxModel && document.documentElement.clientWidth || document.body.clientWidth + "px"')
                         : s.setExpression('width','this.parentNode.offsetWidth + "px"');
                    if (fixL) s.setExpression('left', fixL);
                    if (fixT) s.setExpression('top', fixT);
                }
                else {
                    if (full) s.setExpression('top','(document.documentElement.clientHeight || document.body.clientHeight) / 2 - (this.offsetHeight / 2) + (blah = document.documentElement.scrollTop ? document.documentElement.scrollTop : document.body.scrollTop) + "px"');
                    s.marginTop = 0;
                }
            });
        }
        if (opts.displayMode) {
            w.css('cursor','default').attr('title', $.blockUI.defaults.closeMessage);
            m.css('cursor','default');
            $([f[0],w[0],m[0]]).removeClass('blockUI').addClass('displayBox');
            $().click($.blockUI.impl.boxHandler).bind('keypress', $.blockUI.impl.boxHandler);
        }
        else
            this.bind(1, el);
        m.append(msg).show();
        if (msg.jquery) msg.show();
        if (opts.displayMode) return;
        if (full) {
            this.pageBlock = m[0];
            this.pageBlockEls = $(':input:enabled:visible',this.pageBlock);
            setTimeout(this.focus, 20);
        }
        else this.center(m[0]);
    },
    remove: function(el, opts) {
        var o = $.extend({}, $.blockUI.defaults, opts);
        this.bind(0, el);
        var full = el == window;
        var els = full ? $('body').children().filter('.blockUI') : $('.blockUI', el);
        if (full) this.pageBlock = this.pageBlockEls = null;

        if (o.fadeOut) {
            els.fadeOut(o.fadeTime, function() {
                if (this.parentNode) this.parentNode.removeChild(this);
            });
        }
        else els.remove();
    },
    boxRemove: function(el) {
        $().unbind('click',$.blockUI.impl.boxHandler).unbind('keypress', $.blockUI.impl.boxHandler);
        if (this.boxCallback)
            this.boxCallback(this.box);
        $('body .displayBox').hide().remove();
    },
    // event handler to suppress keyboard/mouse events when blocking
    handler: function(e) {
        if (e.keyCode && e.keyCode == 9) {
            if ($.blockUI.impl.pageBlock && !$.blockUI.defaults.allowTabToLeave) {
                var els = $.blockUI.impl.pageBlockEls;
                var fwd = !e.shiftKey && e.target == els[els.length-1];
                var back = e.shiftKey && e.target == els[0];
                if (fwd || back) {
                    setTimeout(function(){$.blockUI.impl.focus(back)},10);
                    return false;
                }
            }
        }
        if ($(e.target).parents('div.blockMsg').length > 0)
            return true;
        return $(e.target).parents().children().filter('div.blockUI').length == 0;
    },
    boxHandler: function(e) {
        if ((e.keyCode && e.keyCode == 27) || (e.type == 'click' && $(e.target).parents('div.blockMsg').length == 0))
            $.blockUI.impl.boxRemove();
        return true;
    },
    // bind/unbind the handler
    bind: function(b, el) {
        var full = el == window;
        // don't bother unbinding if there is nothing to unbind
        if (!b && (full && !this.pageBlock || !full && !el.$blocked)) return;
        if (!full) el.$blocked = b;
        var $e = $(el).find('a,:input');
        $.each(['mousedown','mouseup','keydown','keypress','click'], function(i,o) {
            $e[b?'bind':'unbind'](o, $.blockUI.impl.handler);
        });
    },
    focus: function(back) {
        if (!$.blockUI.impl.pageBlockEls) return;
        var e = $.blockUI.impl.pageBlockEls[back===true ? $.blockUI.impl.pageBlockEls.length-1 : 0];
        if (e) e.focus();
    },
    center: function(el) {
		var p = el.parentNode, s = el.style;
        var l = ((p.offsetWidth - el.offsetWidth)/2) - this.sz(p,'borderLeftWidth');
        var t = ((p.offsetHeight - el.offsetHeight)/2) - this.sz(p,'borderTopWidth');
        s.left = l > 0 ? (l+'px') : '0';
        s.top  = t > 0 ? (t+'px') : '0';
    },
    sz: function(el, p) { return parseInt($.css(el,p))||0; }
};

})(jQuery);



```
