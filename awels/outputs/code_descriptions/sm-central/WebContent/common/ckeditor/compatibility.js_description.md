# compatibility.js

## Review

## 1. Summary  
The script is a lightweight helper that lives in the CKEditor “samples” directory. Its sole purpose is to inform users when they are accessing a CKEditor‑enabled page with a browser that the library does not officially support.  

- **Core functionality**  
  - Detects browser engine compatibility through `CKEDITOR.env`.  
  - Builds a user‑friendly message listing supported browsers and shows it in a DOM element with id `alerts`.  
  - Registers an `onload` handler to execute the check once the page has finished loading.  

- **Design / patterns**  
  - Immediately‑Invoked Function Expression (IIFE) to avoid polluting the global namespace.  
  - Defensive programming: the script only runs if `window.CKEDITOR` is defined, preventing errors on non‑CKEditor pages.  
  - Browser detection is performed via the `CKEDITOR.env` object that CKEditor exposes internally.  

- **Dependencies**  
  - Relies on CKEditor’s internal `CKEDITOR.env` API.  
  - Requires the presence of a DOM element with id `alerts`.  

## 2. Detailed Description  
### Execution flow  
1. **Console safeguard** – The script first checks whether the `console` object exists and, if so, calls `console.log()`. This is a workaround for old Firebug bugs where the console would crash if not pre‑initialized.  
2. **CKEditor presence check** – If `window.CKEDITOR` exists, an IIFE runs.  
3. **`showCompatibilityMsg`** –  
   - Reads `CKEDITOR.env` to determine which rendering engines are active (`gecko`, `ie`, `opera`, `webkit`).  
   - Builds an `<p>` element containing a warning that the browser is not compatible and a list of supported browsers.  
   - Handles pluralization and conjunction (e.g., “+ and Chrome”) using a regular expression.  
   - Inserts the resulting HTML into the element with id `alerts`.  
4. **`onload`** –  
   - Registered via `addEventListener` or `attachEvent`.  
   - Invoked when the window has fully loaded.  
   - Calls `showCompatibilityMsg()` only if `CKEDITOR.env.isCompatible` is `false`.  

### Assumptions & Constraints  
- The page includes a container `<div id="alerts"></div>` (or any element with that id) for the message.  
- The script is loaded after CKEditor’s core files so that `CKEDITOR.env` is available.  
- No CSS is provided; styling is up to the page developer.  
- The code expects older browsers (IE6, Firefox 2) as potential targets, yet it will still run on modern browsers; it simply does nothing if the environment is compatible.  

### Architecture  
This helper is deliberately minimalistic: it does not attempt to patch or emulate missing features; it merely informs the user. By encapsulating logic in a local IIFE, it avoids leaking variables into the global scope. The use of a dedicated `showCompatibilityMsg` function keeps the `onload` handler clean and makes the message construction testable in isolation.

## 3. Functions/Methods  

| Function | Purpose | Parameters | Return | Side Effects |
|----------|---------|------------|--------|--------------|
| `showCompatibilityMsg()` | Builds and injects an informative message about browser compatibility. | None | None | Sets `innerHTML` of `#alerts`. |
| `onload()` | Triggered on `window.onload`; decides whether to show the message. | None | None | Calls `showCompatibilityMsg()` if `!CKEDITOR.env.isCompatible`. |

Both functions are local to the IIFE and are not exported, ensuring they are only used internally.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` global | Third‑party (CKEditor library) | Requires CKEditor core to be loaded; uses `CKEDITOR.env` and `CKEDITOR.env.isCompatible`. |
| `window.addEventListener` / `attachEvent` | Standard browser API | Handles cross‑browser event registration. |
| `document.getElementById('alerts')` | DOM API | Expects a valid element; otherwise `innerHTML` assignment will throw. |
| `console` object | Standard, optional | Firebug workaround; harmless if `console` is undefined. |

No other external libraries or APIs are referenced.

## 5. Additional Notes  

### Edge Cases & Potential Issues  
- **Missing `#alerts` element** – If the page lacks this element, `document.getElementById('alerts')` returns `null`, leading to a runtime error when trying to set `innerHTML`. A defensive check could mitigate this.  
- **Internationalization** – The message is hard‑coded in English. For multi‑language sites, you’d want to externalize the text.  
- **Styling** – The message is plain HTML; if the host page applies conflicting CSS, the warning might not be prominent. Adding a dedicated CSS class or inline styles could help.  
- **Modern browser support** – While the script still works on modern browsers, the warning is suppressed if `CKEDITOR.env.isCompatible` is true. However, newer browsers may still have quirks not captured by the environment flags.  

### Suggested Enhancements  
1. **Graceful degradation** – Wrap the `innerHTML` assignment in a try/catch or check for the element’s existence.  
2. **Internationalization support** – Load a language file or use `CKEDITOR.lang` to retrieve localized strings.  
3. **Styling hook** – Add a CSS class (e.g., `ck-compat-alert`) to the `<p>` element for easier styling.  
4. **Refactor for unit testing** – Expose `showCompatibilityMsg` (perhaps via a global on `window.CKEDITOR.compat`) to allow automated tests to assert its output.  
5. **Remove console workaround** – Once Firebug or similar debugging tools are up to date, this snippet can be safely omitted, simplifying the script.  

Overall, the code is concise, well‑structured for its purpose, and follows good defensive practices. The only real improvement would be to guard against missing DOM elements and to consider internationalization if used in a global context.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

// This file is not required by CKEditor and may be safely ignored.
// It is just a helper file that displays a red message about browser compatibility
// at the top of the samples (if incompatible browser is detected).

// Firebug has been presented some bugs with console. It must be "initialized"
// before the page load to work.
// FIXME: Remove the following in the future, if Firebug gets fixed.
if ( typeof console != 'undefined' )
	console.log();


if ( window.CKEDITOR )
{
	(function()
	{
		var showCompatibilityMsg = function()
		{
			var env = CKEDITOR.env;

			var html = '<p><strong>Your browser is not compatible with CKEditor.</strong>';

			var browsers =
			{
				gecko : 'Firefox 2.0',
				ie : 'Internet Explorer 6.0',
				opera : 'Opera 9.5',
				webkit : 'Safari 3.0'
			};

			var alsoBrowsers = '';

			for ( var key in env )
			{
				if ( browsers[ key ] )
				{
					if ( env[key] )
						html += ' CKEditor is compatible with ' + browsers[ key ] + ' or higher.';
					else
						alsoBrowsers += browsers[ key ] + '+, ';
				}
			}

			alsoBrowsers = alsoBrowsers.replace( /\+,([^,]+), $/, '+ and $1' );

			html += ' It is also compatible with ' + alsoBrowsers + '.';

			html += '</p><p>With non compatible browsers, you should still be able to see and edit the contents (HTML) in a plain text field.</p>';

			document.getElementById( 'alerts' ).innerHTML = html;
		};

		var onload = function()
		{
			// Show a friendly compatibility message as soon as the page is loaded,
			// for those browsers that are not compatible with CKEditor.
			if ( !CKEDITOR.env.isCompatible )
				showCompatibilityMsg();
		};

		// Register the onload listener.
		if ( window.addEventListener )
			window.addEventListener( 'load', onload, false );
		else if ( window.attachEvent )
			window.attachEvent( 'onload', onload );
	})();
}



```
