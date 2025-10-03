# hoverIntent.js

## Review

## 1. Summary  

**Purpose**  
The snippet implements the classic *hoverIntent* jQuery plugin, originally authored by Brian Cherne.  
Its job is to distinguish a deliberate “hover” from a brief mouse‑over by measuring mouse movement velocity and delay before invoking the supplied “over” and “out” callbacks.

**Key components**  
| Component | Role |
|-----------|------|
| `cfg` | Default options (`sensitivity`, `interval`, `timeout`) merged with user options |
| `track` | Updates the current mouse coordinates (`cX`, `cY`) on `mousemove` |
| `compare` | Periodically compares current vs. previous mouse position; triggers the `over` callback when movement is below the sensitivity threshold |
| `delay` | Executes the `out` callback after an optional timeout |
| `handleHover` | Handles `mouseover`/`mouseout` events, sets up/clears timers and binds the `mousemove` listener |
| `$.fn.hoverIntent` | Public jQuery method that binds `handleHover` to the selected elements |

**Design patterns & libraries**  
* Implements the **jQuery Plugin** pattern by extending `$.fn`.  
* Relies solely on **jQuery** (no external dependencies).  
* Uses closures to encapsulate state and timers.

---

## 2. Detailed Description  

### Execution Flow  
1. **Plugin invocation** – `$(selector).hoverIntent(overFn, outFn)` (or an options object).  
2. **Option handling** – Defaults are merged with user options via `$.extend`.  
3. **Event binding** – `handleHover` is attached to `mouseover` and `mouseout` on each element in the jQuery set.  
4. **Mouseover** –  
   * Records the initial mouse coordinates (`pX`, `pY`).  
   * Binds `mousemove` to `track`.  
   * Starts a timer (`compare`) that runs after `interval` ms.  
5. **mousemove** – `track` keeps `cX`, `cY` up‑to‑date.  
6. **compare** – Every `interval` ms:  
   * If the mouse has moved less than the sensitivity threshold, `overFn` is called.  
   * Otherwise, previous coordinates are updated and `compare` is re‑queued.  
7. **Mouseout** –  
   * Unbinds `mousemove`.  
   * If a hover was detected (`hoverIntent_s == 1`), a delayed `outFn` call is scheduled (`delay`).  
   * If the mouse moves away before the delay, timers are cleared.

### State & Assumptions  
* **State is stored on the element** (`hoverIntent_t`, `hoverIntent_s`) to avoid global pollution.  
* Uses the deprecated DOM properties `e.fromElement`, `e.toElement`, and `e.relatedTarget` for compatibility with very old browsers.  
* Assumes a mouse‑based interaction model; does not support touch or pointer events.  
* The code expects the jQuery namespace (`$`) to be globally available.

### Architecture & Design Choices  
* **Timer‑based polling**: `compare` is a self‑calling `setTimeout`, ensuring intervals are spaced even if previous executions are delayed.  
* **Event unbinding**: `mousemove` is removed as soon as it’s no longer needed, keeping the page performant.  
* **No external state**: All configuration and timer references are local to the plugin instance, making it safe to chain.

---

## 3. Functions / Methods  

| Function | Purpose | Inputs | Outputs / Side‑effects |
|----------|---------|--------|------------------------|
| `$.fn.hoverIntent = function(f,g)` | jQuery plugin entry point. Sets up configuration and binds events. | *`f`* – either an object of options or the `over` callback.<br>*`g`* – optional `out` callback. | Binds `handleHover` to each element; returns the original jQuery set (chainable). |
| `track(ev)` | Updates current mouse coordinates (`cX`, `cY`). | *`ev`* – jQuery mouse event. | Sets global variables `cX`, `cY`. |
| `compare(ev, ob)` | Compares current vs. previous coordinates; triggers `over` if below threshold. | *`ev`* – event object.<br>*`ob`* – DOM element (`this`). | Clears or sets timers; may call `overFn`; updates `hoverIntent_s`. |
| `delay(ev, ob)` | Executes `out` callback after `timeout`. | *`ev`* – event object.<br>*`ob`* – DOM element. | Calls `outFn`; clears timers; resets `hoverIntent_s`. |
| `handleHover(e)` | Central handler for `mouseover` and `mouseout`. | *`e`* – event object. | Binds/unbinds `mousemove`; manages timers; initiates `compare` or `delay`. |

**Reusable utilities** – `track` and `compare` are the core logic that could be extracted for unit‑testing or ported to other event systems.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery** | Third‑party | Required for `$`, `.fn`, `.extend`, `.bind`, `.unbind`, `.mousemove`, etc. |
| None else | — | The plugin is self‑contained and does not rely on other libraries. |

*Platform assumptions* – Works in desktop browsers with mouse input. Not designed for touch devices or IE8+ quirks beyond the old DOM properties used.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Shared State Across Elements**  
   `cX`, `cY`, `pX`, `pY` are defined once per plugin invocation. If `hoverIntent` is called on multiple elements from the same jQuery set, these variables are shared across *all* elements, leading to cross‑interference. Each element should maintain its own state; the current implementation can produce bugs when hovering over several elements rapidly.

2. **Deprecated Event Properties**  
   `e.fromElement`, `e.toElement` are legacy and may be undefined in modern browsers; the fallback to `e.relatedTarget` mitigates this, but the code still references them.

3. **No Support for Touch/Pointer Events**  
   On touch devices the plugin will never trigger because there is no mouse. A modern rewrite could expose a `touch` option or use Pointer Events.

4. **Memory Leaks**  
   Timers (`hoverIntent_t`) are stored on the DOM element and cleared appropriately, but if a page is heavily interactive and elements are frequently created/destroyed, ensuring the timers are cleared on removal is crucial.

### Future Enhancements  
| Enhancement | Why | Suggested Approach |
|-------------|-----|---------------------|
| Element‑level state isolation | Prevent cross‑element interference | Store coordinates and timers in a Map keyed by element or attach them as data via `$.data` |
| Modern event handling | Compatibility with touch & pointer events | Accept an options flag (`touch: true`) that binds `touchstart`/`touchend` and uses `touchmove` coordinates |
| Use `requestAnimationFrame` | More accurate timing, less CPU usage | Replace `setTimeout` loop with RAF‑based polling |
| Remove global `$` assumption | Support ES modules / bundlers | Expose as a UMD module that accepts jQuery as a parameter |
| Better error handling | Robustness | Validate options, provide default callbacks if missing |
| Unit tests | Confidence in refactoring | Write Jest/Enzyme tests that simulate mouse events and timers |

### Overall Assessment  
The plugin faithfully implements the original hoverIntent logic and is well‑structured for its era. However, modern usage would benefit from a few refactors: isolating per‑element state, dropping deprecated properties, and adding support for contemporary input paradigms. The code remains concise and readable, making it a solid base for such enhancements.

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
