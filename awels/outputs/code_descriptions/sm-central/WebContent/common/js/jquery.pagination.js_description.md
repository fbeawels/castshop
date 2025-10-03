# jquery.pagination.js

## Review

## 1. Summary
The snippet is a classic **jQuery plugin** that renders a pagination control inside any selected element.  
* **Purpose** – Given a total number of entries and optional display parameters, it generates “Prev / Next” links, numbered page links, optional ellipses, and edge pages.  
* **Core components** –  
  * **`numPages`** – calculates the total page count.  
  * **`getInterval`** – decides which page numbers are shown based on the current page and the configured window of visible pages.  
  * **`pageSelected`** – the click‑handler that updates the current page, redraws the links, and fires the user callback.  
  * **`drawLinks`** – builds the DOM structure for the control.  
  * **Helper closures** – `getClickHandler` and `appendItem` that keep the main flow readable.  
* **Design patterns** – The plugin follows the *module pattern* by creating a closure over `maxentries`, `opts`, `current_page`, and `panel`.  It also exposes imperative methods (`selectPage`, `prevPage`, `nextPage`) on the DOM element, a pattern often seen in jQuery plugins.  
* **Dependencies** – Only the jQuery library (any version that supports the used API).

---

## 2. Detailed Description
### Execution Flow
1. **Initialization** – When called (`$(selector).pagination(maxentries, opts)`), jQuery’s `fn.pagination` receives the arguments, merges the user options with defaults, and iterates over each matched element via `this.each`.  
2. **Validation & Normalisation** –  
   * `maxentries` is forced to at least 1.  
   * `opts.items_per_page` is forced to at least 1.  
   * `current_page` is extracted from `opts`.  
3. **Private helpers** – The inner functions (`numPages`, `getInterval`, etc.) close over the local variables, giving them full access to the state (`current_page`, `maxentries`, `opts`).  
4. **Control methods** – The element receives three imperative methods (`selectPage`, `prevPage`, `nextPage`) that invoke `pageSelected`.  
5. **First render** – `drawLinks` is called to populate the container.  
6. **Callback** – Finally, the user’s callback is executed once with the initial page.  

### Runtime Behaviour
* Clicking any link triggers `pageSelected`, which updates `current_page`, redraws the control, and runs the callback.  
* The `prev/next` links respect the `prev_show_always` and `next_show_always` flags.  
* The `num_display_entries` window shrinks when near the ends, ensuring the current page is always visible.  
* Edge pages (`num_edge_entries`) are displayed separately, optionally separated by ellipses (`ellipse_text`).  

### Cleanup
Because `drawLinks` empties `panel` before inserting new links, all old DOM nodes (and their bound events) are removed automatically, so no explicit cleanup is required.

### Assumptions & Constraints
* The container element is a simple jQuery object; the plugin does **not** check for nested containers or special element types.  
* The callback is expected to return a boolean indicating whether the default action (event propagation) should be suppressed.  
* The `link_to` option must contain a placeholder `__id__` if real URLs are desired; otherwise, it is simply inserted as the `href` attribute.

---

## 3. Functions/Methods

| Name | Purpose | Inputs | Outputs | Side‑effects |
|------|---------|--------|---------|--------------|
| `numPages()` | Total page count | – | Integer | – |
| `getInterval()` | Visible page range | – | `[start, end]` | – |
| `pageSelected(page_id, evt)` | Handles page selection | `page_id` (int), `evt` (event) | Boolean (propagation flag) | Updates `current_page`, redraws, invokes callback |
| `drawLinks()` | Renders all pagination controls | – | – | Modifies `panel` DOM |
| `getClickHandler(page_id)` | Returns click‑handler closure | `page_id` | Function | – |
| `appendItem(page_id, appendopts)` | Adds a single link or span | `page_id`, optional options | – | Appends to `panel` |
| `this.selectPage(page_id)` | Imperative method exposed on the element | `page_id` | – | Calls `pageSelected` |
| `this.prevPage()` | Imperative method | – | Boolean (`true` if page changed) | Calls `pageSelected` |
| `this.nextPage()` | Imperative method | – | Boolean (`true` if page changed) | Calls `pageSelected` |

### Reusable / Utility Methods
`appendItem` and `getClickHandler` are internal utilities that keep `drawLinks` concise but are not exposed externally.

---

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| **jQuery** | Third‑party | The plugin uses the standard API (`fn`, `.extend`, `.each`, `.empty`, `.append`, `.bind`, `.attr`, `.addClass`). Any jQuery version ≥ 1.4 should work. |
| None | Standard | No other external dependencies. |

---

## 5. Additional Notes

### Strengths
* **Encapsulation** – The closure ensures no leakage into the global scope.  
* **Extensibility** – The callback hook lets callers react to page changes (e.g., AJAX load).  
* **Simplicity** – No heavy dependencies or complex data structures.  

### Potential Issues & Edge Cases
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **`link_to` placeholder** | If `opts.link_to` does not contain `__id__`, all links will point to the same URL. | Validate the placeholder or allow a function to generate the URL. |
| **Negative/Zero `maxentries`** | The plugin coerces to 1, potentially misleading callers that there is at least one item when there might be none. | Keep a distinct `numPages` that can be 0 and render a “no results” state. |
| **`current_page` out of bounds** | If the user sets `opts.current_page` ≥ numPages, subsequent logic may misbehave. | Clamp `current_page` to `[0, numPages-1]`. |
| **`num_display_entries` = 0** | Only Prev/Next appear. Might be acceptable but should be documented. | Add a safeguard or a warning in documentation. |
| **Event handling** | Uses `.bind`; on newer jQuery versions, `.on` is preferred and supports delegation. | Replace `.bind` with `.on`. |
| **Memory leak when re‑initialising** | If the same element is re‑initialised, the previous `selectPage/prevPage/nextPage` methods may persist and cause duplicate bindings. | Remove any existing handlers before re‑initialising or use a namespaced event. |

### Future Enhancements
1. **Keyboard navigation** – Allow arrow keys to move between pages.  
2. **Touch support** – Add swipe gestures for mobile.  
3. **Accessibility** – Add ARIA roles (`pagination`, `page`, `current`) and keyboard focus management.  
4. **Responsive design** – Collapse the pagination into a dropdown on small viewports.  
5. **Unit tests** – Wrap the logic in pure functions that can be unit‑tested without a DOM.  

---

## Code Critique



## Code Preview

```javascript
/**
 * This jQuery plugin displays pagination links inside the selected elements.
 *
 * @author Gabriel Birke (birke *at* d-scribe *dot* de)
 * @version 1.2
 * @param {int} maxentries Number of entries to paginate
 * @param {Object} opts Several options (see README for documentation)
 * @return {Object} jQuery Object
 */
jQuery.fn.pagination = function(maxentries, opts){
	opts = jQuery.extend({
		items_per_page:10,
		num_display_entries:10,
		current_page:0,
		num_edge_entries:0,
		link_to:"#",
		prev_text:"Prev",
		next_text:"Next",
		ellipse_text:"...",
		prev_show_always:true,
		next_show_always:true,
		callback:function(){return false;}
	},opts||{});
	
	return this.each(function() {
		/**
		 * Calculate the maximum number of pages
		 */
		function numPages() {
			return Math.ceil(maxentries/opts.items_per_page);
		}
		
		/**
		 * Calculate start and end point of pagination links depending on 
		 * current_page and num_display_entries.
		 * @return {Array}
		 */
		function getInterval()  {
			var ne_half = Math.ceil(opts.num_display_entries/2);
			var np = numPages();
			var upper_limit = np-opts.num_display_entries;
			var start = current_page>ne_half?Math.max(Math.min(current_page-ne_half, upper_limit), 0):0;
			var end = current_page>ne_half?Math.min(current_page+ne_half, np):Math.min(opts.num_display_entries, np);
			return [start,end];
		}
		
		/**
		 * This is the event handling function for the pagination links. 
		 * @param {int} page_id The new page number
		 */
		function pageSelected(page_id, evt){
			current_page = page_id;
			drawLinks();
			var continuePropagation = opts.callback(page_id, panel);
			if (!continuePropagation) {
				if (evt.stopPropagation) {
					evt.stopPropagation();
				}
				else {
					evt.cancelBubble = true;
				}
			}
			return continuePropagation;
		}
		
		/**
		 * This function inserts the pagination links into the container element
		 */
		function drawLinks() {
			panel.empty();
			var interval = getInterval();
			var np = numPages();
			// This helper function returns a handler function that calls pageSelected with the right page_id
			var getClickHandler = function(page_id) {
				return function(evt){ return pageSelected(page_id,evt); }
			}
			// Helper function for generating a single link (or a span tag if it's the current page)
			var appendItem = function(page_id, appendopts){
				page_id = page_id<0?0:(page_id<np?page_id:np-1); // Normalize page id to sane value
				appendopts = jQuery.extend({text:page_id+1, classes:""}, appendopts||{});
				if(page_id == current_page){
					var lnk = jQuery("<span class='current'>"+(appendopts.text)+"</span>");
				}
				else
				{
					var lnk = jQuery("<a>"+(appendopts.text)+"</a>")
						.bind("click", getClickHandler(page_id))
						.attr('href', opts.link_to.replace(/__id__/,page_id));
						
						
				}
				if(appendopts.classes){lnk.addClass(appendopts.classes);}
				panel.append(lnk);
			}
			// Generate "Previous"-Link
			if(opts.prev_text && (current_page > 0 || opts.prev_show_always)){
				appendItem(current_page-1,{text:opts.prev_text, classes:"prev"});
			}
			// Generate starting points
			if (interval[0] > 0 && opts.num_edge_entries > 0)
			{
				var end = Math.min(opts.num_edge_entries, interval[0]);
				for(var i=0; i<end; i++) {
					appendItem(i);
				}
				if(opts.num_edge_entries < interval[0] && opts.ellipse_text)
				{
					jQuery("<span>"+opts.ellipse_text+"</span>").appendTo(panel);
				}
			}
			// Generate interval links
			for(var i=interval[0]; i<interval[1]; i++) {
				appendItem(i);
			}
			// Generate ending points
			if (interval[1] < np && opts.num_edge_entries > 0)
			{
				if(np-opts.num_edge_entries > interval[1]&& opts.ellipse_text)
				{
					jQuery("<span>"+opts.ellipse_text+"</span>").appendTo(panel);
				}
				var begin = Math.max(np-opts.num_edge_entries, interval[1]);
				for(var i=begin; i<np; i++) {
					appendItem(i);
				}
				
			}
			// Generate "Next"-Link
			if(opts.next_text && (current_page < np-1 || opts.next_show_always)){
				appendItem(current_page+1,{text:opts.next_text, classes:"next"});
			}
		}
		
		// Extract current_page from options
		var current_page = opts.current_page;
		// Create a sane value for maxentries and items_per_page
		maxentries = (!maxentries || maxentries < 0)?1:maxentries;
		opts.items_per_page = (!opts.items_per_page || opts.items_per_page < 0)?1:opts.items_per_page;
		// Store DOM element for easy access from all inner functions
		var panel = jQuery(this);
		// Attach control functions to the DOM element 
		this.selectPage = function(page_id){ pageSelected(page_id);}
		this.prevPage = function(){ 
			if (current_page > 0) {
				pageSelected(current_page - 1);
				return true;
			}
			else {
				return false;
			}
		}
		this.nextPage = function(){ 
			if(current_page < numPages()-1) {
				pageSelected(current_page+1);
				return true;
			}
			else {
				return false;
			}
		}
		// When all initialisation is done, draw the links
		drawLinks();
        // call callback function
        opts.callback(current_page, this);
	});
}





```
