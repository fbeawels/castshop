# hoverIntent.js

## Review

## 1. Summary  
The code defines a **jQuery plugin** called `hoverIntent` that enhances the native `mouseover`/`mouseout` events with a “delayed‑hover” behavior.  
Instead of firing the hover callbacks immediately, it tracks mouse movement and only triggers the *over* callback when the cursor has moved less than a configurable sensitivity threshold over a short interval. The *out* callback is optionally delayed.  

Key components:  
- **Configuration handling** (`sensitivity`, `interval`, `timeout`).  
- **Mouse position tracking** (`track`) and **movement comparison** (`compare`).  
- **Hover state management** via properties (`hoverIntent_t`, `hoverIntent_s`) attached to the element.  
- **Event binding** in `handleHover` that decides whether to start or stop tracking and to schedule callbacks.  

The design follows the **plugin pattern** commonly used in jQuery: an Immediately‑Invoked Function Expression (IIFE) receives `jQuery` as `$` and extends `$.fn`. No external libraries are required beyond jQuery itself.

---

## 2. Detailed Description  
### 2.1 Initialization  
When the plugin is called, e.g. `$('.menu').hoverIntent(overFn, outFn)`, the following steps occur:

1. **Configuration**  
   - Default config (`sensitivity`, `interval`, `timeout`) is merged with any user‑supplied object or functions.  
2. **Variable setup**  
   - `cX, cY` track the current mouse coordinates.  
   - `pX, pY` hold the last “checked” coordinates.

### 2.2 Runtime Flow  
For each element in the jQuery collection:

1. **`handleHover`** is attached to both `mouseover` and `mouseout`.  
   - It normalizes the event source, preventing the handler from firing when the mouse moves between child elements.  
   - On `mouseover`:  
     - Sets `pX, pY` to the entry point.  
     - Binds a `mousemove` handler (`track`) that updates `cX, cY`.  
     - Starts a self‑recursive timer (`compare`) after the first interval if the element isn’t already “hovered”.  
   - On `mouseout`:  
     - Unbinds the expensive `mousemove` handler.  
     - If the element was considered hovered (`hoverIntent_s == 1`), schedules the `delay` callback after `timeout`.

2. **`compare`** (called by the timer) checks whether the mouse movement is below the sensitivity threshold.  
   - If the cursor is stationary enough, it clears the timer, sets the element to hovered state, and calls the *over* callback.  
   - Otherwise, it updates `pX, pY` and reschedules itself after `interval`.

3. **`delay`** (called after a mouseout if the element was hovered) clears timers, resets state, and executes the *out* callback.

### 2.3 Cleanup  
The plugin automatically removes the `mousemove` listener on mouseout, preventing memory leaks. The timers are cleared whenever a new hover state is entered or exited.

### 2.4 Assumptions & Constraints  
- Requires **jQuery** (v1.2+ compatible).  
- Works only in browsers that support `pageX/pageY` and `relatedTarget`.  
- Relies on the element’s own properties (`hoverIntent_t`, `hoverIntent_s`) for state, which may clash if the page uses the same names.  
- Uses `setTimeout` for timing; precision is limited to the browser’s timer resolution (~4 ms).  
- The plugin treats the first `mouseover` as a potential hover; it never triggers the *over* callback if the cursor is moving fast.

---

## 3. Functions / Methods  

| Function | Purpose | Parameters | Returns / Side‑Effects |
|----------|---------|------------|------------------------|
| **`$.fn.hoverIntent`** | Main plugin entry point. | `f` (over callback or config object), `g` (out callback) | Binds `handleHover` to each matched element; returns the jQuery collection (chainable). |
| **`track(ev)`** | Updates current mouse coordinates. | `ev` (jQuery event) | Sets global `cX`, `cY`. |
| **`compare(ev, ob)`** | Checks if movement stayed below sensitivity. | `ev` (event), `ob` (DOM element) | May trigger *over* callback or reschedule itself. |
| **`delay(ev, ob)`** | Delays execution of the *out* callback. | `ev`, `ob` | Calls *out* callback after `timeout`. |
| **`handleHover(e)`** | Core event handler for `mouseover`/`mouseout`. | `e` (event) | Binds/unbinds `mousemove`, starts/stops timers, orchestrates `compare`/`delay`. |

Reusable utility methods:  
- `$.extend` for config merging.  
- `$.fn.mouseover/.mouseout` for event binding.  
- `jQuery.extend` (aliased to `jQuery.extend`) for copying event objects.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| **jQuery** | Third‑party | Must be loaded before this script. The plugin uses core jQuery methods (`fn`, `extend`, `mouseover`, `mouseout`, `bind`, `unbind`). |
| **DOM Events** (`mouseover`, `mouseout`, `mousemove`) | Standard | Browser‑native events. |
| **`setTimeout` / `clearTimeout`** | Standard | Native timer API. |
| **`pageX`, `pageY`, `relatedTarget`** | Browser API | Provides mouse coordinates and event relationships. |

No other external dependencies are required. Platform‑specific assumptions are minimal but the code assumes a typical desktop browser environment (mouse support).

---

## 5. Additional Notes  

### 5.1 Strengths  
- **Lightweight**: No external dependencies beyond jQuery.  
- **Encapsulation**: All state lives on the DOM element; no global variables.  
- **Robustness**: Handles rapid mouse movements and prevents accidental *over* triggers.  

### 5.2 Potential Issues & Edge Cases  
1. **Property Collision** – Using `hoverIntent_t` and `hoverIntent_s` as custom properties could overwrite existing properties if a developer defines them. Using a unique namespace (e.g., `__hoverIntent`) or a WeakMap would mitigate this.  
2. **Touch Devices** – The plugin relies on mouse events; it won’t work on touch screens unless polyfilled.  
3. **Performance on Many Elements** – Binding `mousemove` per element can become costly if many elements use `hoverIntent`. A global mousemove handler with a throttled lookup could be more efficient.  
4. **IE Compatibility** – The code uses `e.fromElement`/`e.toElement` for legacy IE; modern browsers prefer `relatedTarget`. The fallback logic works but could be simplified with `e.relatedTarget`.  
5. **Timer Accuracy** – The plugin uses `setTimeout` in a recursive manner; if the browser throttles timers, the “intent” detection might be delayed.

### 5.3 Suggested Enhancements  
- **Namespace the custom properties** to avoid clashes.  
- **Add support for touch events** (e.g., `touchstart`, `touchend`) or provide an option to disable the plugin on touch devices.  
- **Expose the plugin’s state** (e.g., `isHoverIntent`) as a public API for debugging.  
- **Parameter validation**: Ensure numeric config values are positive and sensible.  
- **Modernize event handling**: Use `e.isPropagationStopped()` to avoid redundant work, and consider `PointerEvents` for future‑proofing.  
- **Unit tests**: Write Jasmine/Karma tests to validate the timing logic across browsers.

--- 

**Verdict:**  
The `hoverIntent` plugin is a well‑structured, classic jQuery implementation that successfully adds a “intentional hover” delay. It follows established patterns, is easy to understand, and is efficient for its intended use cases. Minor refactoring to avoid property collisions and to support modern event APIs would make it future‑proof and more robust.

## Code Critique



## Code Preview

```javascript
(function($){
	/* hoverIntent by Brian Cherne */
	$.fn.hoverIntent = function(f,g) {
		// default configuration options
		var cfg = {
			sensitivity: 7,
			interval: 100,
			timeout: 0
		};
		// override configuration options with user supplied object
		cfg = $.extend(cfg, g ? { over: f, out: g } : f );

		// instantiate variables
		// cX, cY = current X and Y position of mouse, updated by mousemove event
		// pX, pY = previous X and Y position of mouse, set by mouseover and polling interval
		var cX, cY, pX, pY;

		// A private function for getting mouse position
		var track = function(ev) {
			cX = ev.pageX;
			cY = ev.pageY;
		};

		// A private function for comparing current and previous mouse position
		var compare = function(ev,ob) {
			ob.hoverIntent_t = clearTimeout(ob.hoverIntent_t);
			// compare mouse positions to see if they've crossed the threshold
			if ( ( Math.abs(pX-cX) + Math.abs(pY-cY) ) < cfg.sensitivity ) {
				$(ob).unbind("mousemove",track);
				// set hoverIntent state to true (so mouseOut can be called)
				ob.hoverIntent_s = 1;
				return cfg.over.apply(ob,[ev]);
			} else {
				// set previous coordinates for next time
				pX = cX; pY = cY;
				// use self-calling timeout, guarantees intervals are spaced out properly (avoids JavaScript timer bugs)
				ob.hoverIntent_t = setTimeout( function(){compare(ev, ob);} , cfg.interval );
			}
		};

		// A private function for delaying the mouseOut function
		var delay = function(ev,ob) {
			ob.hoverIntent_t = clearTimeout(ob.hoverIntent_t);
			ob.hoverIntent_s = 0;
			return cfg.out.apply(ob,[ev]);
		};

		// A private function for handling mouse 'hovering'
		var handleHover = function(e) {
			// next three lines copied from jQuery.hover, ignore children onMouseOver/onMouseOut
			var p = (e.type == "mouseover" ? e.fromElement : e.toElement) || e.relatedTarget;
			while ( p && p != this ) { try { p = p.parentNode; } catch(e) { p = this; } }
			if ( p == this ) { return false; }

			// copy objects to be passed into t (required for event object to be passed in IE)
			var ev = jQuery.extend({},e);
			var ob = this;

			// cancel hoverIntent timer if it exists
			if (ob.hoverIntent_t) { ob.hoverIntent_t = clearTimeout(ob.hoverIntent_t); }

			// else e.type == "onmouseover"
			if (e.type == "mouseover") {
				// set "previous" X and Y position based on initial entry point
				pX = ev.pageX; pY = ev.pageY;
				// update "current" X and Y position based on mousemove
				$(ob).bind("mousemove",track);
				// start polling interval (self-calling timeout) to compare mouse coordinates over time
				if (ob.hoverIntent_s != 1) { ob.hoverIntent_t = setTimeout( function(){compare(ev,ob);} , cfg.interval );}

			// else e.type == "onmouseout"
			} else {
				// unbind expensive mousemove event
				$(ob).unbind("mousemove",track);
				// if hoverIntent state is true, then call the mouseOut function after the specified delay
				if (ob.hoverIntent_s == 1) { ob.hoverIntent_t = setTimeout( function(){delay(ev,ob);} , cfg.timeout );}
			}
		};

		// bind the function to the two event listeners
		return this.mouseover(handleHover).mouseout(handleHover);
	};
	
})(jQuery);


```
