# jquery.pagination.js

## Review

## 1. Summary

The file implements a **jQuery plugin** named `pagination`. Its purpose is to render a paginated navigation UI inside any jQuery‑selected element.  
The plugin accepts the total number of items (`maxentries`) and an optional configuration object. It exposes three imperative methods (`selectPage`, `prevPage`, `nextPage`) on the underlying DOM element so that client code can programmatically change the page.  

Key components:
- **Configuration defaults** (`items_per_page`, `num_display_entries`, etc.).
- **Internal helpers**: `numPages`, `getInterval`, `pageSelected`, `drawLinks`.
- **DOM generation logic** that builds `<a>` and `<span>` tags for page links, ellipses, and previous/next controls.
- **Callback hook** to notify the caller whenever the page changes.

Design patterns:
- *Module pattern* via an immediately invoked function that returns the jQuery object.
- *Callback pattern* to decouple the UI from business logic.

## 2. Detailed Description

### 2.1 Flow of Execution

1. **Invocation**  
   `$(selector).pagination(maxentries, opts);` is called.  
   `opts` is merged with defaults using `jQuery.extend`.

2. **Iteration over matched elements**  
   `this.each(function() { … });` ensures each matched element gets its own independent pagination instance.

3. **Initialization**  
   - `current_page` is read from `opts.current_page`.
   - `maxentries` and `opts.items_per_page` are clamped to a minimum of `1`.
   - `panel` holds the jQuery object for the container.
   - Three public methods (`selectPage`, `prevPage`, `nextPage`) are attached to the raw DOM element for later use.

4. **Rendering**  
   `drawLinks()` is called immediately to populate the pagination UI.  
   After rendering, the initial callback is invoked with `current_page` and the element reference.

5. **User Interaction**  
   Clicking any link triggers the `pageSelected` handler which:
   - Updates `current_page`.
   - Re-renders links via `drawLinks()`.
   - Executes the user‑supplied callback, allowing the caller to prevent the default navigation by returning `false`.

### 2.2 Core Components

| Component | Responsibility |
|-----------|----------------|
| `numPages()` | Calculates the total number of pages based on `maxentries` and `items_per_page`. |
| `getInterval()` | Determines which page numbers should be displayed around the current page, respecting `num_display_entries`. |
| `pageSelected(page_id, evt)` | Handles page change requests, updates state, redraws UI, and triggers callback. |
| `drawLinks()` | Builds the entire pagination markup (prev, next, page numbers, ellipses). |
| Public methods (`selectPage`, `prevPage`, `nextPage`) | Expose imperative navigation controls to consumer code. |

### 2.3 Assumptions & Constraints

- `maxentries` and `items_per_page` must be positive integers; negative or zero values are coerced to `1`.
- `opts.current_page` defaults to `0`. Caller may supply a value > 0.
- The callback signature is `function(page_id, element)` and may return `false` to cancel navigation.
- The plugin uses jQuery 1.x+ APIs; no ES6+ features.
- It manipulates the DOM directly; no virtual DOM or React/Vue abstractions.

## 3. Functions/Methods

### 3.1 `numPages()`

- **Input**: None (uses outer `maxentries` and `opts.items_per_page`).  
- **Output**: Integer number of pages (`ceil(maxentries / items_per_page)`).

### 3.2 `getInterval()`

- **Input**: None (depends on `current_page`, `numPages()`, `opts`).  
- **Output**: Array `[start, end]` where `start` inclusive, `end` exclusive, defining the page range to show.

### 3.3 `pageSelected(page_id, evt)`

- **Inputs**:  
  - `page_id` – desired page index.  
  - `evt` – event object from the click handler.  
- **Side‑effects**:  
  - Updates `current_page`.  
  - Calls `drawLinks()` to refresh UI.  
  - Executes `opts.callback(page_id, panel)`.  
  - If callback returns falsy, stops event propagation.  
- **Returns**: Boolean indicating whether navigation should continue.

### 3.4 `drawLinks()`

- **Input**: None.  
- **Side‑effects**:  
  - Clears `panel`.  
  - Generates “Prev” link, edge pages, ellipses, current interval pages, edge pages at the end, and “Next” link.  
  - Uses helper `appendItem(page_id, appendopts)` to create `<a>` or `<span>` elements.

### 3.5 Public Methods on DOM element

| Method | Purpose | Parameters | Returns |
|--------|---------|------------|---------|
| `selectPage(page_id)` | Programmatically switch to a page. | Integer | `undefined` |
| `prevPage()` | Go to the previous page if possible. | None | Boolean (true if moved) |
| `nextPage()` | Go to the next page if possible. | None | Boolean (true if moved) |

### 3.6 Utilities

- **`getClickHandler(page_id)`** – returns a closure that calls `pageSelected` with the correct `page_id`.
- **`appendItem(page_id, appendopts)`** – normalizes `page_id`, creates either a `<span>` for the current page or an `<a>` with a click handler and href.

## 4. Dependencies

| Library / API | Nature | Notes |
|---------------|--------|-------|
| **jQuery** | Third‑party | Relies on `$`, `jQuery.extend`, `each`, DOM manipulation methods (`empty`, `append`, `bind`, `attr`, `addClass`). No external plugins. |
| **DOM Level 0/1** | Browser API | Uses `evt.stopPropagation` and `evt.cancelBubble` for event handling. |
| **ES5** | Language feature | Uses `function` declarations and closures; compatible with all browsers that support jQuery. |

No other external dependencies exist.

## 5. Additional Notes

### 5.1 Edge Cases & Potential Issues

- **Invalid `maxentries`**: If `maxentries` is `0`, the plugin coerces it to `1`, potentially showing an empty pagination bar. A clearer error or silent no‑op might be preferable.
- **Negative `items_per_page`**: Similarly coerced to `1`; a warning could aid debugging.
- **`opts.callback` Returning `true`/`false`**: The plugin treats any falsy value as a cancel signal. If a callback returns `0` or an empty string, navigation will be canceled unintentionally.
- **`opts.link_to`**: Uses `replace(/__id__/, page_id)`; if the string contains multiple `__id__` placeholders, only the first is replaced. A more robust templating approach might be needed.
- **Concurrency**: Re‑entering `pageSelected` while the UI is being redrawn could lead to race conditions in very rapid clicking scenarios, though unlikely.

### 5.2 Potential Enhancements

1. **Option Validation** – Add a validation step that throws descriptive errors for non‑numeric or out‑of‑range options.
2. **Accessibility** – Add ARIA roles (`role="navigation"`), labels for prev/next, and keyboard navigation support.
3. **Customization** – Allow the user to supply a custom rendering function for page items instead of the default `<a>`/`<span>`.
4. **Touch Support** – Ensure click events are replaced or supplemented with `touchstart`/`touchend` for mobile devices.
5. **Modernization** – Rewrite as a jQuery plugin in ES6, or expose a plain JavaScript module that can be used with other frameworks.

### 5.3 Overall Assessment

The code is **compact, readable, and functional**. It follows common jQuery plugin conventions, making it easy for developers familiar with jQuery to integrate. However, modern best practices (accessibility, error handling, separation of concerns) are not fully addressed. For legacy projects still using jQuery, this plugin is a solid foundation; for new projects, consider migrating to a framework‑agnostic solution or enhancing the plugin with the improvements above.

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
