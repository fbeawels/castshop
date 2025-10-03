# thickbox-noconflict.js

## Review

## 1. Summary  

**Purpose**  
The script implements a lightweight modal “thickbox” overlay for images, inline content, or iframes. It is intended to replace the native `<a rel="lightbox">` approach with a jQuery‑driven solution that works across browsers (including legacy IE6).

**Key Components**

| Component | Role |
|-----------|------|
| `tb_init` | Binds click events to elements with a `.thickbox` class, triggering the overlay. |
| `tb_show` | Main dispatcher – decides whether to show an image, iframe, or AJAX/inline content. |
| `tb_remove` | Closes the overlay and cleans up DOM/handlers. |
| `tb_position` | Centers the overlay. |
| Helper functions (`tb_parseQuery`, `tb_getPageSize`, `tb_detectMacXFF`, `tb_showIframe`) | Provide auxiliary logic for query string parsing, sizing, and specific browser work‑arounds. |

**Design Patterns & Libraries**

* **Module‑like pattern** – All logic is exposed via global functions; the plugin does not use an IIFE or closure for encapsulation, making many variables global.  
* **Event‑based** – Uses jQuery’s `click` and `keydown` events to interact with the overlay.  
* **DOM manipulation** – Elements (`#TB_overlay`, `#TB_window`, etc.) are created on demand and removed on close.  
* **Conditional feature detection** – Uses `document.body.style.maxHeight` for IE6 detection and `navigator.userAgent` for a mac‑Firefox quirk.  

The code relies on **jQuery** (any 1.x version, pre‑1.9) and the deprecated `$.browser` property for certain browser checks.

---

## 2. Detailed Description  

### Initialization  

```js
jQuery(document).ready(function(){
    tb_init('a.thickbox, area.thickbox, input.thickbox');
    imgLoader = new Image();
    imgLoader.src = tb_pathToImage;
});
```

* `tb_init` is called once the DOM is ready, binding the thickbox behaviour to all anchor/area/input elements that have the `.thickbox` class.  
* An image pre‑loader (`imgLoader`) is created to cache the loading animation.

### `tb_init(domChunk)`  

Iterates over the matched elements and attaches a click handler that:
1. Extracts title/name, href/alt, and rel attributes.  
2. Calls `tb_show(caption, url, imageGroup)`.  
3. Calls `this.blur()` and prevents default navigation (`return false`).

### `tb_show(caption, url, imageGroup)`  

1. **Browser compatibility** –  
   * IE6: injects an iframe `#TB_HideSelect` to hide `<select>` elements.  
   * Others: injects only `#TB_overlay` and `#TB_window`.  
2. **Overlay style** – Adds classes `TB_overlayMacFFBGHack` or `TB_overlayBG` based on a mac/Firefox detection.  
3. **Loading indicator** – Appends `#TB_load` with the preloaded GIF.  
4. **URL parsing** – Separates query string to determine width/height parameters.  
5. **Image detection** – Uses a regex to decide if the URL ends with an image extension.  

#### Image Branch

* Builds navigation data (prev/next) if `imageGroup` is supplied.  
* Preloads the target image (`imgPreloader`). On load:
  * Resizes large images so they fit within the page (`pagesize`).  
  * Sets `TB_WIDTH` / `TB_HEIGHT`.  
  * Inserts the image and caption into `#TB_window`.  
  * Wire up Prev/Next links (`#TB_prev`, `#TB_next`).  
  * Registers `keydown` handler for Escape, comma, and period navigation.  
  * Positions the overlay, removes the loading indicator, and shows the window.

#### Non‑Image Branch (iframe / AJAX / inline)

* Parses query parameters (`params`) to determine dimensions and modal behaviour.  
* Handles three sub‑cases:  
  * **iframe** – Appends an `<iframe>` inside `#TB_window`.  
  * **inline** – Moves a DOM node into the overlay and restores it on unload.  
  * **AJAX** – Uses `jQuery.load` to fetch content.  

* Adds a Close button that triggers `tb_remove`.  
* Sets a `keyup` handler for the Escape key (if not modal).

### `tb_remove`  

* Unbinds click/close handlers, fades out `#TB_window`, then removes overlay elements.  
* Restores body height/overflow for IE6.  
* Clears global keyboard handlers.

### `tb_position`  

* Centers the overlay horizontally, vertically (unless IE < 7).

### Helpers  

* `tb_parseQuery` – Parses a query string into an object.  
* `tb_getPageSize` – Calculates the current viewport size.  
* `tb_detectMacXFF` – Detects macOS Firefox to apply a PNG overlay.  
* `tb_showIframe` – Called when an iframe finishes loading.

---

## 3. Functions/Methods  

| Function | Purpose | Inputs | Outputs / Side‑Effects |
|----------|---------|--------|------------------------|
| `tb_init(domChunk)` | Binds click events to `.thickbox` elements. | `domChunk` (selector string) | Adds `click` handler; sets up overlay on click. |
| `tb_show(caption, url, imageGroup)` | Shows overlay for image, iframe, AJAX, or inline content. | `caption` (string), `url` (string), `imageGroup` (string or false) | Creates DOM, loads content, registers keyboard events. |
| `tb_remove()` | Hides overlay, cleans up. | none | Removes overlay elements, unbinds events. |
| `tb_position()` | Centers `#TB_window`. | none | Sets CSS `marginLeft/Top` on overlay. |
| `tb_parseQuery(query)` | Parses `?foo=bar&baz=qux` into `{foo:'bar', baz:'qux'}`. | `query` (string) | Object of key/value pairs. |
| `tb_getPageSize()` | Returns `[width, height]` of viewport. | none | Array `[w,h]`. |
| `tb_detectMacXFF()` | Returns true if running on macOS Firefox. | none | Boolean. |
| `tb_showIframe()` | Called when iframe finishes loading. | none | Removes loading spinner, shows window. |

**Reusable Utilities** – `tb_parseQuery` and `tb_getPageSize` can be extracted for other scripts.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| **jQuery (≥ 1.4)** | Third‑party | Entire script relies on jQuery for DOM manipulation, events, and AJAX. |
| `$.browser` | Deprecated jQuery feature | Used for Safari detection (`$.browser.safari`). Not available in jQuery ≥1.9. |
| `window.innerWidth/innerHeight` | Browser API | Fallback to `document.documentElement` for older browsers. |
| `navigator.userAgent` | Browser API | Simple string match for mac/Firefox. |

No other external dependencies.

---

## 5. Additional Notes & Recommendations  

### Strengths  
* **Cross‑browser coverage** – Includes work‑arounds for IE6, Safari, and macOS Firefox.  
* **Modular click binding** – `tb_init` can target any selector, making it flexible.  
* **Graceful degradation** – Non‑image URLs fall back to AJAX or inline content.

### Issues & Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Global namespace pollution** – All functions and variables are global. | Can clash with other scripts; hard to maintain. | Wrap the entire plugin in an IIFE or module, expose only a single namespace (e.g., `Thickbox`). |
| **Deprecated `$.browser`** | Fails in jQuery 1.9+ where the property is removed. | Replace with feature detection or userAgent checks. |
| **Image type detection** – relies on regex matching file extensions only. | URLs that return images without extensions (e.g., query strings) will be treated as non‑image. | Inspect `Content-Type` header via AJAX HEAD request or allow explicit `type=image` parameter. |
| **No error handling** – All `try/catch` blocks swallow exceptions silently. | Bugs are hard to debug. | Log errors to console (`console.error`) or provide callback hooks. |
| **Hard‑coded sizes** – Uses magic numbers (`+30`, `+60`) for image padding. | Layout can break on very small screens or different DPI settings. | Compute padding from CSS or expose as configurable options. |
| **Keyboard navigation** – Only comma/period for next/prev; does not support arrow keys. | User experience limitation. | Add arrow key handling. |
| **IE6 hidden select fix** – Uses an iframe overlay (`#TB_HideSelect`). | Still required for old IE; may conflict with other iframes. | Modernize to CSS `pointer-events` or remove legacy support if not needed. |
| **Memory leaks** – `imgPreloader.onload` assigns a function that keeps a reference to the outer scope. | Could prevent GC in long sessions. | Use `imgPreloader.onload = null;` (already done) but ensure no lingering closures. |
| **Potential XSS** – URLs and captions are inserted via `innerHTML` without sanitization. | If URLs or titles come from user input, script injection is possible. | Escape HTML or use text nodes for captions; validate URLs. |
| **No API for programmatic control** – Cannot open/close from script other than by triggering a click. | Limits integration with dynamic UIs. | Expose `tb_open`, `tb_close`, `tb_setCaption`, etc. |

### Future Enhancements  

1. **Modernize to ES6+** – Use `const/let`, arrow functions, and modules.  
2. **Add configuration object** – Options for animation speed, overlay color, modal behaviour, etc.  
3. **Accessibility** – Add ARIA roles, focus trapping, and keyboard navigation.  
4. **Testing** – Unit tests for parsing, sizing, and DOM creation.  
5. **Performance** – Lazy‑load thumbnails and cache previous/next images.  
6. **Responsive design** – Adapt overlay size to viewport on resize events.  

Overall, the script achieves its goal of a lightweight lightbox but would benefit from modern JavaScript practices, better encapsulation, and improved security and UX features.

## Code Critique



## Code Preview

```javascript
/*
 * Thickbox 3.1 - One Box To Rule Them All.
 * By Cody Lindley (http://www.codylindley.com)
 * Copyright (c) 2007 cody lindley
 * Licensed under the MIT License: http://www.opensource.org/licenses/mit-license.php
*/

var tb_pathToImage = "../common/img/loadingAnimation.gif";
var tb_closeImage = "../common/img/delete-icon.png";
var closekey="Close";
var exckey = " or Esc Key";

/*!!!!!!!!!!!!!!!!! edit below this line at your own risk !!!!!!!!!!!!!!!!!!!!!!!*/

//on page load call tb_init
jQuery(document).ready(function(){
	tb_init('a.thickbox, area.thickbox, input.thickbox');//pass where to apply thickbox
	imgLoader = new Image();// preload image
	imgLoader.src = tb_pathToImage;
});

//add thickbox to href & area elements that have a class of .thickbox
function tb_init(domChunk){
	jQuery(domChunk).click(function(){
	var t = this.title || this.name || null;
	var a = this.href || this.alt;
	var g = this.rel || false;
	tb_show(t,a,g);
	this.blur();
	return false;
	});
}

function tb_show(caption, url, imageGroup) {//function called when the user clicks on a thickbox link

	try {
		if (typeof document.body.style.maxHeight === "undefined") {//if IE 6
			jQuery("body","html").css({height: "100%", width: "100%"});
			jQuery("html").css("overflow","hidden");
			if (document.getElementById("TB_HideSelect") === null) {//iframe to hide select elements in ie6
				jQuery("body").append("<iframe id='TB_HideSelect'></iframe><div id='TB_overlay'></div><div id='TB_window'></div>");
				jQuery("#TB_overlay").click(tb_remove);
			}
		}else{//all others
			if(document.getElementById("TB_overlay") === null){
				jQuery("body").append("<div id='TB_overlay'></div><div id='TB_window'></div>");
				jQuery("#TB_overlay").click(tb_remove);
			}
		}

		if(tb_detectMacXFF()){
			jQuery("#TB_overlay").addClass("TB_overlayMacFFBGHack");//use png overlay so hide flash
		}else{
			jQuery("#TB_overlay").addClass("TB_overlayBG");//use background and opacity
		}

		if(caption===null){caption="";}
		jQuery("body").append("<div id='TB_load'><img src='"+imgLoader.src+"' /></div>");//add loader to the page
		jQuery('#TB_load').show();//show loader

		var baseURL;
	   if(url.indexOf("?")!==-1){ //ff there is a query string involved
			baseURL = url.substr(0, url.indexOf("?"));
	   }else{
	   		baseURL = url;
	   }

	   var urlString = /\.jpg$|\.jpeg$|\.png$|\.gif$|\.bmp$/;
	   var urlType = baseURL.toLowerCase().match(urlString);

		if(urlType == '.jpg' || urlType == '.jpeg' || urlType == '.png' || urlType == '.gif' || urlType == '.bmp'){//code to show images

			TB_PrevCaption = "";
			TB_PrevURL = "";
			TB_PrevHTML = "";
			TB_NextCaption = "";
			TB_NextURL = "";
			TB_NextHTML = "";
			TB_imageCount = "";
			TB_FoundURL = false;
			if(imageGroup){
				TB_TempArray = jQuery("a[@rel="+imageGroup+"]").get();
				for (TB_Counter = 0; ((TB_Counter < TB_TempArray.length) && (TB_NextHTML === "")); TB_Counter++) {
					var urlTypeTemp = TB_TempArray[TB_Counter].href.toLowerCase().match(urlString);
						if (!(TB_TempArray[TB_Counter].href == url)) {
							if (TB_FoundURL) {
								TB_NextCaption = TB_TempArray[TB_Counter].title;
								TB_NextURL = TB_TempArray[TB_Counter].href;
								TB_NextHTML = "<span id='TB_next'>&nbsp;&nbsp;<a href='#'>Next &gt;</a></span>";
							} else {
								TB_PrevCaption = TB_TempArray[TB_Counter].title;
								TB_PrevURL = TB_TempArray[TB_Counter].href;
								TB_PrevHTML = "<span id='TB_prev'>&nbsp;&nbsp;<a href='#'>&lt; Prev</a></span>";
							}
						} else {
							TB_FoundURL = true;
							TB_imageCount = "Image " + (TB_Counter + 1) +" of "+ (TB_TempArray.length);
						}
				}
			}

			imgPreloader = new Image();
			imgPreloader.onload = function(){
			imgPreloader.onload = null;

			// Resizing large images - orginal by Christian Montoya edited by me.
			var pagesize = tb_getPageSize();
			var x = pagesize[0] - 150;
			var y = pagesize[1] - 150;
			var imageWidth = imgPreloader.width;
			var imageHeight = imgPreloader.height;
			if (imageWidth > x) {
				imageHeight = imageHeight * (x / imageWidth);
				imageWidth = x;
				if (imageHeight > y) {
					imageWidth = imageWidth * (y / imageHeight);
					imageHeight = y;
				}
			} else if (imageHeight > y) {
				imageWidth = imageWidth * (y / imageHeight);
				imageHeight = y;
				if (imageWidth > x) {
					imageHeight = imageHeight * (x / imageWidth);
					imageWidth = x;
				}
			}
			// End Resizing

			TB_WIDTH = imageWidth + 30;
			TB_HEIGHT = imageHeight + 60;
			//jQuery("#TB_window").append("<a href='' id='TB_ImageOff' title='Close'><img id='TB_Image' src='"+url+"' width='"+imageWidth+"' height='"+imageHeight+"' alt='"+caption+"'/></a>" + "<div id='TB_caption'>"+caption+"<div id='TB_secondLine'>" + TB_imageCount + TB_PrevHTML + TB_NextHTML + "</div></div><div id='TB_closeWindow'><a href='#' id='TB_closeWindowButton' title='Close'>close</a> or Esc Key</div>");
			jQuery("#TB_window").append("<a href='' id='TB_ImageOff' title='Close'><img id='TB_Image' src='"+url+"' width='"+imageWidth+"' height='"+imageHeight+"' alt='"+caption+"'/></a>" + "<div id='TB_caption'>"+caption+"<div id='TB_secondLine'>" + TB_imageCount + TB_PrevHTML + TB_NextHTML + "</div></div><div id='TB_closeWindow'><a href='#' id='TB_closeWindowButton' title='Close'>" + "<img src='" + tb_closeImage + "'></a>"+esckey+"</div>");

			jQuery("#TB_closeWindowButton").click(tb_remove);

			if (!(TB_PrevHTML === "")) {
				function goPrev(){
					if(jQuery(document).unbind("click",goPrev)){jQuery(document).unbind("click",goPrev);}
					jQuery("#TB_window").remove();
					jQuery("body").append("<div id='TB_window'></div>");
					tb_show(TB_PrevCaption, TB_PrevURL, imageGroup);
					return false;
				}
				jQuery("#TB_prev").click(goPrev);
			}

			if (!(TB_NextHTML === "")) {
				function goNext(){
					jQuery("#TB_window").remove();
					jQuery("body").append("<div id='TB_window'></div>");
					tb_show(TB_NextCaption, TB_NextURL, imageGroup);
					return false;
				}
				jQuery("#TB_next").click(goNext);

			}

			document.onkeydown = function(e){
				if (e == null) { // ie
					keycode = event.keyCode;
				} else { // mozilla
					keycode = e.which;
				}
				if(keycode == 27){ // close
					tb_remove();
				} else if(keycode == 190){ // display previous image
					if(!(TB_NextHTML == "")){
						document.onkeydown = "";
						goNext();
					}
				} else if(keycode == 188){ // display next image
					if(!(TB_PrevHTML == "")){
						document.onkeydown = "";
						goPrev();
					}
				}
			};

			tb_position();
			jQuery("#TB_load").remove();
			jQuery("#TB_ImageOff").click(tb_remove);
			jQuery("#TB_window").css({display:"block"}); //for safari using css instead of show
			};

			imgPreloader.src = url;
		}else{//code to show html

			var queryString = url.replace(/^[^\?]+\??/,'');
			var params = tb_parseQuery( queryString );

			TB_WIDTH = (params['width']*1) + 30 || 630; //defaults to 630 if no paramaters were added to URL
			TB_HEIGHT = (params['height']*1) + 40 || 440; //defaults to 440 if no paramaters were added to URL
			ajaxContentW = TB_WIDTH - 30;
			ajaxContentH = TB_HEIGHT - 45;

			if(url.indexOf('TB_iframe') != -1){// either iframe or ajax window
					urlNoQuery = url.split('TB_');
					jQuery("#TB_iframeContent").remove();
					if(params['modal'] != "true"){//iframe no modal
						jQuery("#TB_window").append("<div id='TB_title'><div id='TB_ajaxWindowTitle'>"+caption+"</div><div id='TB_closeAjaxWindow'><a href='#' id='TB_closeWindowButton' title='Close'>close</a> or Esc Key</div></div><iframe frameborder='0' hspace='0' src='"+urlNoQuery[0]+"' id='TB_iframeContent' name='TB_iframeContent"+Math.round(Math.random()*1000)+"' onload='tb_showIframe()' style='width:"+(ajaxContentW + 29)+"px;height:"+(ajaxContentH + 17)+"px;' > </iframe>");
					}else{//iframe modal
					jQuery("#TB_overlay").unbind();
						jQuery("#TB_window").append("<iframe frameborder='0' hspace='0' src='"+urlNoQuery[0]+"' id='TB_iframeContent' name='TB_iframeContent"+Math.round(Math.random()*1000)+"' onload='tb_showIframe()' style='width:"+(ajaxContentW + 29)+"px;height:"+(ajaxContentH + 17)+"px;'> </iframe>");
					}
			}else{// not an iframe, ajax
					if(jQuery("#TB_window").css("display") != "block"){
						if(params['modal'] != "true"){//ajax no modal
						jQuery("#TB_window").append("<div id='TB_title'><div id='TB_ajaxWindowTitle'>"+caption+"</div><div id='TB_closeAjaxWindow'><a href='#' id='TB_closeWindowButton'>close</a> or Esc Key</div></div><div id='TB_ajaxContent' style='width:"+ajaxContentW+"px;height:"+ajaxContentH+"px'></div>");
						}else{//ajax modal
						jQuery("#TB_overlay").unbind();
						jQuery("#TB_window").append("<div id='TB_ajaxContent' class='TB_modal' style='width:"+ajaxContentW+"px;height:"+ajaxContentH+"px;'></div>");
						}
					}else{//this means the window is already up, we are just loading new content via ajax
						jQuery("#TB_ajaxContent")[0].style.width = ajaxContentW +"px";
						jQuery("#TB_ajaxContent")[0].style.height = ajaxContentH +"px";
						jQuery("#TB_ajaxContent")[0].scrollTop = 0;
						jQuery("#TB_ajaxWindowTitle").html(caption);
					}
			}

			jQuery("#TB_closeWindowButton").click(tb_remove);

				if(url.indexOf('TB_inline') != -1){
					jQuery("#TB_ajaxContent").append(jQuery('#' + params['inlineId']).children());
					jQuery("#TB_window").unload(function () {
						jQuery('#' + params['inlineId']).append( jQuery("#TB_ajaxContent").children() ); // move elements back when you're finished
					});
					tb_position();
					jQuery("#TB_load").remove();
					jQuery("#TB_window").css({display:"block"});
				}else if(url.indexOf('TB_iframe') != -1){
					tb_position();
					if($.browser.safari){//safari needs help because it will not fire iframe onload
						jQuery("#TB_load").remove();
						jQuery("#TB_window").css({display:"block"});
					}
				}else{
					jQuery("#TB_ajaxContent").load(url += "&random=" + (new Date().getTime()),function(){//to do a post change this load method
						tb_position();
						jQuery("#TB_load").remove();
						tb_init("#TB_ajaxContent a.thickbox");
						jQuery("#TB_window").css({display:"block"});
					});
				}

		}

		if(!params['modal']){
			document.onkeyup = function(e){
				if (e == null) { // ie
					keycode = event.keyCode;
				} else { // mozilla
					keycode = e.which;
				}
				if(keycode == 27){ // close
					tb_remove();
				}
			};
		}

	} catch(e) {
		//nothing here
	}
}

//helper functions below
function tb_showIframe(){
	jQuery("#TB_load").remove();
	jQuery("#TB_window").css({display:"block"});
}

function tb_remove() {
 	jQuery("#TB_imageOff").unbind("click");
	jQuery("#TB_closeWindowButton").unbind("click");
	jQuery("#TB_window").fadeOut("fast",function(){jQuery('#TB_window,#TB_overlay,#TB_HideSelect').trigger("unload").unbind().remove();});
	jQuery("#TB_load").remove();
	if (typeof document.body.style.maxHeight == "undefined") {//if IE 6
		jQuery("body","html").css({height: "auto", width: "auto"});
		jQuery("html").css("overflow","");
	}
	document.onkeydown = "";
	document.onkeyup = "";
	return false;
}

function tb_position() {
jQuery("#TB_window").css({marginLeft: '-' + parseInt((TB_WIDTH / 2),10) + 'px', width: TB_WIDTH + 'px'});
	if ( !(jQuery.browser.msie && jQuery.browser.version < 7)) { // take away IE6
		jQuery("#TB_window").css({marginTop: '-' + parseInt((TB_HEIGHT / 2),10) + 'px'});
	}
}

function tb_parseQuery ( query ) {
   var Params = {};
   if ( ! query ) {return Params;}// return empty object
   var Pairs = query.split(/[;&]/);
   for ( var i = 0; i < Pairs.length; i++ ) {
      var KeyVal = Pairs[i].split('=');
      if ( ! KeyVal || KeyVal.length != 2 ) {continue;}
      var key = unescape( KeyVal[0] );
      var val = unescape( KeyVal[1] );
      val = val.replace(/\+/g, ' ');
      Params[key] = val;
   }
   return Params;
}

function tb_getPageSize(){
	var de = document.documentElement;
	var w = window.innerWidth || self.innerWidth || (de&&de.clientWidth) || document.body.clientWidth;
	var h = window.innerHeight || self.innerHeight || (de&&de.clientHeight) || document.body.clientHeight;
	arrayPageSize = [w,h];
	return arrayPageSize;
}

function tb_detectMacXFF() {
  var userAgent = navigator.userAgent.toLowerCase();
  if (userAgent.indexOf('mac') != -1 && userAgent.indexOf('firefox')!=-1) {
    return true;
  }
}





```
