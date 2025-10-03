# picker.js

## Review

## 1. Summary  

The code implements the **Tigra Color Picker**, a lightweight JavaScript colour picker widget that can be dropped into web pages. It is split into two files (`picker.js` and `picker.html`) and works in very old browsers (Netscape 4, IE 5).  

* **Purpose** – Provide a pop‑up dialog that allows the user to select a colour from several pre‑defined palettes (safe, Windows, Macintosh, grey‑scale, etc.).  
* **Key components**  
  * **`TColorPicker`** – Constructor that wires together all sub‑functions (drawing, event handlers, palette builders).  
  * **`TCDraw`** – Draws the colour palettes into a popup window.  
  * **Palette generators** – `TCGenerateSafe`, `TCGenerateWind`, `TCGenerateMac`, `TCGenerateGray` create HTML tables of colour swatches.  
  * **Utility helpers** – `TCBuildCell`, `TCDec2Hex`, `TCChgMode`, `TCSelect`, `TCPaint`.  
* **Notable design choices** – The code uses the old “layers” model for Netscape 4 compatibility (`document.layers`). It relies on global functions and string concatenation to generate markup. No modern frameworks are used; everything is hand‑rolled.  

---

## 2. Detailed Description  

### Flow of execution  

1. **Global instance**  
   ```js
   var TCP = new TColorPicker();
   ```  
   An instance is created once and reused by all pop‑ups.

2. **Invoking the picker**  
   ```js
   function TCPopup(field, palette) { … }
   ```  
   * Called by the page (typically from an `onclick` on a colour field).  
   * Opens `picker.html` in a new window, centers it, and passes the opener reference.

3. **Popup window** (`picker.html`)  
   * On load it calls `TCP.draw(this, this.document);` – the global `TCP` instance draws itself into the newly opened window.  
   * `TCDraw` writes the palette tables into the document, hooks up the sample display, and shows the initial palette (`this.C(this.initPalette)`).

4. **Palette interaction**  
   * Each colour swatch is an `<a>` that calls `P.S('xxxxxx')` on click and `P.P('xxxxxx')` on mouse‑over.  
   * `TCSelect` writes the chosen colour into the input field and closes the window.  
   * `TCPaint` updates the sample element to show the current colour.

5. **Cleanup** – When the window is closed the global `TCP` instance remains in the parent page but is otherwise idle.

### Core components & interaction  

| Component | Responsibility | Interaction |
|-----------|----------------|-------------|
| `TColorPicker` | Holds references to all helper functions, the palette builders, and the visibility handlers (`show/hide`). | Exposes `popup`, `draw`, `S`, `P`, `C`, etc. |
| `TCDraw` | Writes the markup into the popup window; sets up event handlers by assigning `this.win`/`this.doc`. | Called by the popup to build UI. |
| Palette generators (`TCGenerate*`) | Return string representation of `<tr>` rows containing colour cells. | Used by `TCDraw` when constructing each palette. |
| `TCBuildCell` | Builds a single colour cell `<td>` with an `<a>` pointing back to the parent picker. | Invoked by palette generators. |
| `TCDec2Hex` | Hex conversion helper. | Used by all colour calculations. |
| `TCChgMode`, `TCSelect`, `TCPaint` | UI event handlers for switching palettes, selecting a colour, and updating the preview. | Wired into the global picker instance. |

### Assumptions & constraints  

* **Browser support** – The code checks `document.layers` and `document.all`. It is intended for Netscape 4 and Internet Explorer 5–6. Modern browsers will work but the code is fragile (uses deprecated features).  
* **Global variables** – `TCP` is a global singleton. Functions are also defined in the global scope.  
* **Popup blocking** – Relies on a user action to open the window; modern browsers may block the popup unless invoked by a click.  
* **No CSS** – Styling is done inline and via table attributes; no external stylesheet.  

---

## 3. Functions / Methods  

| Name | Purpose | Parameters | Return | Side effects |
|------|---------|------------|--------|---------------|
| `TCPopup(field, palette)` | Opens the colour picker popup. | `field` – input element to receive colour.<br>`palette` – optional initial palette index. | None | Opens a new window, sets `opener`. |
| `TCBuildCell(R, G, B, w, h)` | Builds an HTML `<td>` containing a colour swatch. | `R, G, B` – colour components.<br>`w, h` – cell width/height. | String of `<td>` markup | None |
| `TCSelect(c)` | Sets the selected colour in the input and closes picker. | `c` – colour hex string (no `#`). | None | `field.value` updated, window closed. |
| `TCPaint(c, b_noPref)` | Updates the preview sample. | `c` – colour string.<br>`b_noPref` – flag to skip `#`. | None | Changes innerHTML / background color of sample. |
| `TCGenerateSafe()` | Generates the “Safe” colour palette. | None | String of `<tr>` rows | None |
| `TCGenerateWind()` | Generates the Windows palette. | None | String | None |
| `TCGenerateMac()` | Generates the Macintosh palette (including greys). | None | String | None |
| `TCGenerateGray()` | Generates a 256‑grey palette. | None | String | None |
| `TCDec2Hex(v)` | Converts integer to 6‑digit hex. | `v` – integer. | Hex string | None |
| `TCChgMode(v)` | Switches visible palette. | `v` – palette index. | None | Calls `hide` on all divs, then `show`. |
| `TColorPicker(field)` | Constructor for the picker instance. | `field` – input element. | Object | Binds helpers, sets up `divs`. |
| `TCDraw(o_win, o_doc)` | Draws the picker UI into a window. | `o_win` – window object.<br>`o_doc` – document object. | None | Writes markup, assigns `sample` reference. |

**Reusable / utility methods**

* `TCDec2Hex` – generic number → hex conversion.  
* `TCBuildCell` – could be reused in any colour‑grid rendering.  

---

## 4. Dependencies  

| Library / API | Nature | Notes |
|---------------|--------|-------|
| **`window.open`** | Standard JS | Used to create popup. |
| **`document.layers` / `document.all`** | Browser feature detection (legacy) | Enables Netscape 4 and IE 5 support. |
| **`document.write`** | Standard JS | Employed by `TCDraw` to build UI. |
| **No external libraries** | – | Entire implementation is hand‑rolled. |
| **`pixel.gif`** | Image resource | Small transparent GIF used to size swatches. |

All dependencies are either standard browser APIs or a single image asset. The code is fully self‑contained aside from the external `picker.html` file.

---

## 5. Additional Notes  

### Edge cases / limitations  

1. **Popup blockers** – Modern browsers may block `window.open` unless triggered by a user gesture.  
2. **Cross‑domain restrictions** – If `picker.html` is loaded from a different origin than the parent page, accessing `opener` may be blocked.  
3. **Deprecated APIs** – `document.layers` is obsolete; `document.all` is only for older IE. The code will still work in most browsers, but the visibility toggling logic may not behave as intended on modern browsers.  
4. **No validation** – The input field is set to a hex string regardless of whether it is a valid colour; there is no check for length or format.  
5. **Accessibility** – The palette is purely visual; no keyboard navigation or ARIA attributes.  

### Potential Enhancements  

| Idea | Benefit |
|------|---------|
| **Modernize** – Rewrite using standard DOM APIs, `addEventListener`, CSS classes, and `<canvas>` for drawing. | Better maintainability, improved performance, easier styling. |
| **Accessibility** – Add focus management, keyboard support, and ARIA labels. | Makes the picker usable for screen readers. |
| **Configurable palettes** – Allow users to supply custom palette arrays. | Extensibility. |
| **Promise / async API** – Return a Promise that resolves with the selected colour. | Cleaner integration with modern JS code. |
| **Unit tests** – Write tests for palette generators and colour conversion. | Ensures correctness during refactoring. |
| **Use of CSS** – Move styling out of inline attributes. | Easier theme changes. |

### Final remarks  

The script is a classic example of early 2000s JavaScript: global state, string‑based markup, and heavy reliance on legacy browser features. While it still functions in many browsers today, it is brittle in the face of modern web standards. Refactoring to a component‑based, event‑driven implementation would greatly improve readability, testability, and accessibility. Nonetheless, the code is well‑structured for its time, with clear separation between palette generation, UI rendering, and event handling.

## Code Critique



## Code Preview

```javascript
// Title: Tigra Color Picker
// URL: http://www.softcomplex.com/products/tigra_color_picker/
// Version: 1.1
// Date: 06/26/2003 (mm/dd/yyyy)
// Note: Permission given to use this script in ANY kind of applications if
//    header lines are left unchanged.
// Note: Script consists of two files: picker.js and picker.html

var TCP = new TColorPicker();

function TCPopup(field, palette) {
	this.field = field;
	this.initPalette = !palette || palette > 3 ? 0 : palette;
	var w = 194, h = 240,
	move = screen ? 
		',left=' + ((screen.width - w) >> 1) + ',top=' + ((screen.height - h) >> 1) : '', 
	o_colWindow = window.open('picker.html', null, "help=no,status=no,scrollbars=no,resizable=no" + move + ",width=" + w + ",height=" + h + ",dependent=yes", true);
	o_colWindow.opener = window;
	o_colWindow.focus();
}

function TCBuildCell (R, G, B, w, h) {
	return '<td bgcolor="#' + this.dec2hex((R << 16) + (G << 8) + B) + '"><a href="javascript:P.S(\'' + this.dec2hex((R << 16) + (G << 8) + B) + '\')" onmouseover="P.P(\'' + this.dec2hex((R << 16) + (G << 8) + B) + '\')"><img src="pixel.gif" width="' + w + '" height="' + h + '" border="0"></a></td>';
}

function TCSelect(c) {
	this.field.value = '#' + c.toUpperCase();
	this.win.close();
}

function TCPaint(c, b_noPref) {
	c = (b_noPref ? '' : '#') + c.toUpperCase();
	if (this.o_samp) 
		this.o_samp.innerHTML = '<font face=Tahoma size=2>' + c +' <font color=white>' + c + '</font></font>'
	if(this.doc.layers)
		this.sample.bgColor = c;
	else { 
		if (this.sample.backgroundColor != null) this.sample.backgroundColor = c;
		else if (this.sample.background != null) this.sample.background = c;
	}
}

function TCGenerateSafe() {
	var s = '';
	for (j = 0; j < 12; j ++) {
		s += "<tr>";
		for (k = 0; k < 3; k ++)
			for (i = 0; i <= 5; i ++)
				s += this.bldCell(k * 51 + (j % 2) * 51 * 3, Math.floor(j / 2) * 51, i * 51, 8, 10);
		s += "</tr>";
	}
	return s;
}

function TCGenerateWind() {
	var s = '';
	for (j = 0; j < 12; j ++) {
		s += "<tr>";
		for (k = 0; k < 3; k ++)
			for (i = 0; i <= 5; i++)
				s += this.bldCell(i * 51, k * 51 + (j % 2) * 51 * 3, Math.floor(j / 2) * 51, 8, 10);
		s += "</tr>";
	}
	return s	
}
function TCGenerateMac() {
	var s = '';
	var c = 0,n = 1;
	var r,g,b;
	for (j = 0; j < 15; j ++) {
		s += "<tr>";
		for (k = 0; k < 3; k ++)
			for (i = 0; i <= 5; i++){
				if(j<12){
				s += this.bldCell( 255-(Math.floor(j / 2) * 51), 255-(k * 51 + (j % 2) * 51 * 3),255-(i * 51), 8, 10);
				}else{
					if(n<=14){
						r = 255-(n * 17);
						g=b=0;
					}else if(n>14 && n<=28){
						g = 255-((n-14) * 17);
						r=b=0;
					}else if(n>28 && n<=42){
						b = 255-((n-28) * 17);
						r=g=0;
					}else{
						r=g=b=255-((n-42) * 17);
					}
					s += this.bldCell( r, g,b, 8, 10);
					n++;
				}
			}
		s += "</tr>";
	}
	return s;
}

function TCGenerateGray() {
	var s = '';
	for (j = 0; j <= 15; j ++) {
		s += "<tr>";
		for (k = 0; k <= 15; k ++) {
			g = Math.floor((k + j * 16) % 256);
			s += this.bldCell(g, g, g, 9, 7);
		}
		s += '</tr>';
	}
	return s
}

function TCDec2Hex(v) {
	v = v.toString(16);
	for(; v.length < 6; v = '0' + v);
	return v;
}

function TCChgMode(v) {
	for (var k in this.divs) this.hide(k);
	this.show(v);
}

function TColorPicker(field) {
	this.build0 = TCGenerateSafe;
	this.build1 = TCGenerateWind;
	this.build2 = TCGenerateGray;
	this.build3 = TCGenerateMac;
	this.show = document.layers ? 
		function (div) { this.divs[div].visibility = 'show' } :
		function (div) { this.divs[div].visibility = 'visible' };
	this.hide = document.layers ? 
		function (div) { this.divs[div].visibility = 'hide' } :
		function (div) { this.divs[div].visibility = 'hidden' };
	// event handlers
	this.C       = TCChgMode;
	this.S       = TCSelect;
	this.P       = TCPaint;
	this.popup   = TCPopup;
	this.draw    = TCDraw;
	this.dec2hex = TCDec2Hex;
	this.bldCell = TCBuildCell;
	this.divs = [];
}

function TCDraw(o_win, o_doc) {
	this.win = o_win;
	this.doc = o_doc;
	var 
	s_tag_openT  = o_doc.layers ? 
		'layer visibility=hidden top=54 left=5 width=182' : 
		'div style=visibility:hidden;position:absolute;left:6px;top:54px;width:182px;height:0',
	s_tag_openS  = o_doc.layers ? 'layer top=32 left=6' : 'div',
	s_tag_close  = o_doc.layers ? 'layer' : 'div'
		
	this.doc.write('<' + s_tag_openS + ' id=sam name=sam><table cellpadding=0 cellspacing=0 border=1 width=181 align=center class=bd><tr><td align=center height=18><div id="samp"><font face=Tahoma size=2>sample <font color=white>sample</font></font></div></td></tr></table></' + s_tag_close + '>');
	this.sample = o_doc.layers ? o_doc.layers['sam'] : 
		o_doc.getElementById ? o_doc.getElementById('sam').style : o_doc.all['sam'].style

	for (var k = 0; k < 4; k ++) {
		this.doc.write('<' + s_tag_openT + ' id="p' + k + '" name="p' + k + '"><table cellpadding=0 cellspacing=0 border=1 align=center>' + this['build' + k]() + '</table></' + s_tag_close + '>');
		this.divs[k] = o_doc.layers 
			? o_doc.layers['p' + k] : o_doc.all 
				? o_doc.all['p' + k].style : o_doc.getElementById('p' + k).style
	}
	if (!o_doc.layers && o_doc.body.innerHTML) 
		this.o_samp = o_doc.all 
			? o_doc.all.samp : o_doc.getElementById('samp');
	this.C(this.initPalette);
	if (this.field.value) this.P(this.field.value, true)
}



```
