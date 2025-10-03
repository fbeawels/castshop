# jqueryFileTree.js

## Review

## 1. Summary  

The snippet is a classic **jQuery file‑tree plug‑in** (version 1.01) that renders an expandable directory tree in the browser.  
It sends an AJAX request (`POST`) to a server‑side script (default `jqueryFileTree.php`) and expects an HTML fragment (a nested `<ul>/<li>` list) in return.  

**Key components**

| Component | Role |
|-----------|------|
| `$.fn.fileTree` | Public method that users call to initialise the plug‑in on a DOM element. |
| `showTree` | Recursive loader that fetches sub‑folders/files and injects them into the tree. |
| `bindTree` | Attaches click (or other configured event) handlers to `<a>` elements to handle expand/collapse or file selection. |
| `escape` (global) | Used to URL‑encode directory paths sent to the server. |

The plug‑in follows a **module‑pattern**: it extends `$.fn`, uses closure to encapsulate private helpers, and exposes a single public API. It is *not* built with modern ES6 syntax but is perfectly functional for the era it was written.

---

## 2. Detailed Description  

### Initialisation
```js
$('.fileTreeDemo').fileTree( options, callback )
```
* `options` – object of configurable properties (`root`, `script`, `expandSpeed`, …).  
* `callback` – function executed when the user clicks a *file* `<a>`.

Defaults are set via a series of `if (o.xxx === undefined) o.xxx = …;`. The code also gracefully handles the case where `options` is omitted.

### Core Flow

1. **Iteration** – `$(this).each(function(){ … });` allows multiple trees on the same page.
2. **Show Loading UI** – `$(this).html('<ul class="jqueryFileTree start"><li class="wait">…');` displays a “Loading…’’ message.
3. **Initial AJAX Load** – `showTree($(this), escape(o.root));`
4. **`showTree`**  
   * Adds the `wait` CSS class while the request is pending.  
   * Calls `$.post(o.script, {dir: t}, callback)`.  
   * On success it injects the returned `<ul>` into the tree, removes the waiting state, and triggers `bindTree`.
5. **`bindTree`**  
   * Binds the configured `folderEvent` (default `click`) to every `<a>` within the tree.  
   * If the `<li>` is a `directory` and is `collapsed`, it expands by recursively calling `showTree`.  
   * If already expanded, it collapses the child `<ul>` with a slide animation.  
   * For files, it calls the user‑supplied callback `h` with the file path.  
   * Handles `multiFolder` mode by collapsing sibling directories.  
   * Optionally runs a user supplied `o.loadedCallBack` via `eval` (dangerous).  
   * Adds a guard to prevent default link behaviour on non‑`click` events.

### Dependencies & Assumptions

| Dependency | Type | Notes |
|------------|------|-------|
| `jQuery` | Third‑party | Required; the code is wrapped in `if (jQuery) { … }`. |
| `escape` | Global (native `escape` function) | Used to encode directory strings; `escape` is deprecated in modern JS. |
| Server script (`jqueryFileTree.php` by default) | External | Must return well‑formed `<ul>`/`<li>` markup and honor `dir` POST parameter. |
| CSS classes (`jqueryFileTree`, `wait`, `directory`, `collapsed`, `expanded`) | Implicit | Must exist for visual styling and animation. |

The plug‑in assumes a relatively flat and sane server response and does not validate the returned markup.

### Architecture & Design Choices

* **Recursive Loading** – Instead of building the entire tree at once, sub‑folders are loaded on‑demand, which reduces initial payload but requires careful event rebinding.
* **Animation Parameters** – Speed and easing options allow customizing the expand/collapse feel.
* **Multi‑Folder Control** – A simple boolean that toggles whether only one branch can be open at a time.
* **Callback Model** – Two callbacks: `h` for file selection, and an optional `loadedCallBack` that is executed via `eval` after a directory is rendered.

---

## 3. Functions/Methods  

| Function | Purpose | Inputs | Outputs / Side‑Effects |
|----------|---------|--------|------------------------|
| `$.fn.fileTree(o, h)` | Public API to initialise the file tree. | `o` (options object), `h` (file click callback). | Creates the tree markup, starts initial AJAX load, attaches event handlers. |
| `showTree(c, t)` | Loads the contents of directory `t` and injects into element `c`. | `c` – jQuery element (the tree container).<br> `t` – escaped directory path. | Updates DOM with new `<ul>`; triggers `bindTree` on the updated list. |
| `bindTree(t)` | Binds event handlers to all `<a>` inside the tree `t`. | `t` – jQuery element (the tree container). | Registers click/other events; expands/collapses directories; calls file callback. |
| `eval(o.loadedCallBack)` | Executes optional user supplied JS after a folder is loaded. | `o.loadedCallBack` – string. | Executes arbitrary code – **security risk**. |
| `escape` (global) | Encodes directory strings for safe transmission. | String | Encoded string. |

The plugin uses **closure** to keep `showTree` and `bindTree` private while exposing only `fileTree`.

---

## 4. Dependencies  

| Library / API | Category | Remarks |
|--------------|----------|---------|
| **jQuery** | Third‑party | The plug‑in is entirely dependent on jQuery. No namespace conflicts as it checks `if(jQuery)` and uses an IIFE. |
| **`escape`** | Native (global) | Deprecated in modern JS; should be replaced by `encodeURIComponent`. |
| **CSS classes** | Implicit | Must be defined in the page’s stylesheet for proper layout and animation. |
| **Server‑side script** (`jqueryFileTree.php`) | External | Handles the `dir` POST param and returns markup; the plug‑in has no fallback for malformed data. |

No other external libraries are referenced.

---

## 5. Additional Notes  

### Strengths
* **Lightweight** – ~1.5 kB gzipped; no heavy dependencies beyond jQuery.
* **Lazy Loading** – Only requested directories are fetched, keeping initial payload low.
* **Customizable** – Speed, easing, multi‑folder, and callbacks are all exposed.

### Weaknesses & Edge Cases
1. **Use of `eval`** – `o.loadedCallBack` is executed via `eval`, opening the door to arbitrary code injection if the option is set from an untrusted source.
2. **Deprecated `escape`** – This will be removed in future browsers. Replace with `encodeURIComponent` or a custom encoder.
3. **Event Re‑binding** – Every time a folder is loaded, `bindTree` re‑binds events on *all* `<a>` tags, potentially leading to multiple handlers if the plugin is invoked multiple times or used in a dynamic context.
4. **No error handling** – If the AJAX call fails, the user is left with a hanging “Loading…” state. A failure callback or timeout would improve UX.
5. **Accessibility** – The plug‑in uses `<a>` tags but does not add ARIA attributes or keyboard navigation support.
6. **Non‑click events** – The code tries to prevent default on non‑click events but uses `toLowerCase` without parentheses (`o.folderEvent.toLowerCase`), which will always return `false` because it’s a function reference, not the string. This is a bug that may silently break the guard.
7. **Memory leaks** – Calling `$(this).parent().find('UL').remove();` deletes the `<ul>` but does not detach event handlers that were attached to it, potentially leaving orphaned listeners.

### Suggested Enhancements
* **Replace `eval`** – Use a callback function instead of a string, or wrap the string in a safe sandbox.
* **Use `.on()`** – Delegate events from the root `<ul>` to avoid re‑binding.
* **Error handling** – Add a `.fail()` callback to `$.post` and display a user‑friendly message.
* **Accessibility** – Add keyboard navigation, `role="tree"`, and `aria-expanded` attributes.
* **Modernise** – Switch to `encodeURIComponent` and consider ES6 syntax if the codebase is updated.
* **Cache results** – Store loaded folders in memory to avoid redundant AJAX requests when a user collapses and re‑expands a folder.
* **Fix the `toLowerCase` bug** – Change to `o.folderEvent.toLowerCase() !== 'click'`.

Overall, the plug‑in is a solid, proven solution for the time it was written, but it would benefit from modernisation and a few security fixes before being used in a contemporary codebase.

## Code Critique



## Code Preview

```javascript
// jQuery File Tree Plugin
//
// Version 1.01
//
// Cory S.N. LaViska
// A Beautiful Site (http://abeautifulsite.net/)
// 24 March 2008
//
// Visit http://abeautifulsite.net/notebook.php?article=58 for more information
//
// Usage: $('.fileTreeDemo').fileTree( options, callback )
//
// Options:  root           - root folder to display; default = /
//           script         - location of the serverside AJAX file to use; default = jqueryFileTree.php
//           folderEvent    - event to trigger expand/collapse; default = click
//           expandSpeed    - default = 500 (ms); use -1 for no animation
//           collapseSpeed  - default = 500 (ms); use -1 for no animation
//           expandEasing   - easing function to use on expand (optional)
//           collapseEasing - easing function to use on collapse (optional)
//           multiFolder    - whether or not to limit the browser to one subfolder at a time
//           loadMessage    - Message to display while initial tree loads (can be HTML)
//
// History:
//
// 1.01 - updated to work with foreign characters in directory/file names (12 April 2008)
// 1.00 - released (24 March 2008)
//
// TERMS OF USE
// 
// This plugin is dual-licensed under the GNU General Public License and the MIT License and
// is copyright 2008 A Beautiful Site, LLC. 
//
if(jQuery) (function($){
	
	$.extend($.fn, {
		fileTree: function(o, h) {
			// Defaults
			if( !o ) var o = {};
			if( o.root == undefined ) o.root = '/';
			if( o.script == undefined ) o.script = 'jqueryFileTree.php';
			if( o.folderEvent == undefined ) o.folderEvent = 'click';
			if( o.expandSpeed == undefined ) o.expandSpeed= 500;
			if( o.collapseSpeed == undefined ) o.collapseSpeed= 500;
			if( o.expandEasing == undefined ) o.expandEasing = null;
			if( o.collapseEasing == undefined ) o.collapseEasing = null;
			if( o.multiFolder == undefined ) o.multiFolder = true;
			if( o.loadMessage == undefined ) o.loadMessage = 'Loading...';
			
			$(this).each( function() {
				
				function showTree(c, t) {
					$(c).addClass('wait');
					$(".jqueryFileTree.start").remove();
					$.post(o.script, { dir: t }, function(data) {
						$(c).find('.start').html('');
						$(c).removeClass('wait').append(data);
						if( o.root == t ) $(c).find('UL:hidden').show(); else $(c).find('UL:hidden').slideDown({ duration: o.expandSpeed, easing: o.expandEasing });
						bindTree(c);
					});
				}
				
				function bindTree(t) {
					$(t).find('LI A').bind(o.folderEvent, function() {
						if( $(this).parent().hasClass('directory') ) {
							if( $(this).parent().hasClass('collapsed') ) {
								// Expand
								if( !o.multiFolder ) {
									$(this).parent().parent().find('UL').slideUp({ duration: o.collapseSpeed, easing: o.collapseEasing });
									$(this).parent().parent().find('LI.directory').removeClass('expanded').addClass('collapsed');
								}
								$(this).parent().find('UL').remove(); // cleanup
								showTree( $(this).parent(), escape($(this).attr('rel').match( /.*\// )) );
								$(this).parent().removeClass('collapsed').addClass('expanded');
							} else {
								// Collapse
								$(this).parent().find('UL').slideUp({ duration: o.collapseSpeed, easing: o.collapseEasing });
								$(this).parent().removeClass('expanded').addClass('collapsed');
							}
						} else {
							h($(this).attr('rel'));
						}
						return false;
					});
					
					if (o.loadedCallBack!=null) eval(o.loadedCallBack); 
      
					// Prevent A from triggering the # on non-click events
					if( o.folderEvent.toLowerCase != 'click' ) $(t).find('LI A').bind('click', function() { return false; });
				}
				// Loading message
				$(this).html('<ul class="jqueryFileTree start"><li class="wait">' + o.loadMessage + '<li></ul>');
				// Get the initial file list
				showTree( $(this), escape(o.root) );
			});
		}
	});
	
})(jQuery);


```
