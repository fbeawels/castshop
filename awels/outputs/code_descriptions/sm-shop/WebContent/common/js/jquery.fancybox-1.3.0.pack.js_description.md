# jquery.fancybox-1.3.0.pack.js

## Review

## 1. Summary  

**Purpose** – The script implements the original *FancyBox* jQuery lightbox plugin (v1.3.0). It creates modal overlays for images, AJAX, inline HTML, iframes, Flash objects and more. The plugin is entirely client‑side and relies on jQuery for DOM manipulation, event handling, and Ajax.

**Key Components**  
| Component | Role |
|-----------|------|
| **`ba()`** | One‑time DOM construction (wrap, overlay, loading indicator, navigation). |
| **`B()`** | Entry point for opening a new item. Determines type (`image`, `ajax`, `inline`, etc.) and configures the overlay. |
| **`J()`, `G()`, `U()`** | Layout / size calculations and animation helpers for showing the content. |
| **`H()`, `Q()`, `B()`** | Clean‑up, error handling and cancel logic. |
| **`fancybox` namespace** – `b.fancybox`, `b.fancybox.close`, `next`, `prev`, etc. | Public API exposed to users. |
| **`defaults`** – `b.fn.fancybox.defaults` | Global configuration options. |

**Design patterns**  
- *Module pattern* – all internal variables are scoped in the IIFE.  
- *Callback hooks* – `onStart`, `onCancel`, `onComplete`, `onCleanup`, `onClosed`.  
- *State machine* – The plugin keeps an array `j` of all items, an index `n`, and a flag `h` for “currently running”.

## 2. Detailed Description  

### Initialization  
- `b(document).ready(function(){ba()})` – creates all hidden helper DOM elements (`#fancybox-wrap`, `#fancybox-overlay`, etc.) the first time the document loads.

### Binding the plugin  
- `b.fn.fancybox = function(a) { … }` – attaches the plugin to a jQuery collection.  
  * If an element is clicked, `d` (rel attribute or full selector) is used to build an array `q` of all linked items.  
  * The `rel` value may reference a group (`rel="gallery"`), or a single element (`rel=""`).  
  * `s` keeps the index of the clicked element inside that group.

- `b.fancybox = function(a, d)` – allows programmatic invocation (array or object). It populates `q` with one or more items and calls `B()`.

### Opening an item – `B()`  
1. **Cleanup** – calls `H()` (hides loading spinner, aborts previous XHR, empties content).  
2. **Configuration** – merges user options (`a`) with defaults (`b.fn.fancybox.defaults`).  
3. **Content type detection** –  
   * `image`, `ajax`, `inline`, `iframe`, `swf` or `html`.  
   * Uses heuristics: file extensions, CSS classes (`iframe`), anchor href patterns, etc.  
4. **Hooks** – `onStart` may cancel the request.  
5. **Styling** – sets padding, dimensions, overlay properties, etc.  
6. **Content loading** – switch statement handles each type:
   * **`image`** – creates `Image`, sets `onload`/`onerror` callbacks.  
   * **`swf`** – builds an `<object>`/`embed`.  
   * **`ajax`** – jQuery AJAX call; `success` inserts response.  
   * **`iframe`** – creates an `<iframe>` element.  
   * **`inline`** – clones the target element into the modal.  
   * **`html`** – directly inserts `e.content`.  

7. **Animation & layout** – `J()` triggers the appropriate animation (`elastic`, `fade`, etc.) and calls `G()` to set final dimensions.

### Animations  
- **Elastic** – uses `M()` to animate `width`, `height`, `top`, `left` with easing (`easingChange` or `easingOut`).  
- **Fade** – simple jQuery fade in/out.  
- **Show/hide** – `v` (loading) and `y` (overlay) are shown/hidden appropriately.

### Navigation  
- Arrow elements (`#fancybox-left`, `#fancybox-right`) bind to `prev()`/`next()`.  
- Keyboard shortcuts: ESC (close), left/right arrows (prev/next).  
- Mouse wheel navigation if `b.fn.mousewheel` is available.

### Cleanup  
- `H()` removes all content, aborts XHR (`F`), resets state.  
- `close()` animates out, removes elements, fires cleanup hooks.  
- `cancel()` cancels the current request and triggers `onCancel`.

### Resizing & Centering  
- `resize()` recalculates the content area when the content height changes.  
- `center()` updates the overlay position on window resize or scroll events.

## 3. Functions/Methods  

| Function | Purpose | Inputs | Outputs | Side‑Effects |
|----------|---------|--------|---------|--------------|
| `H()` | Hide loading, abort XHR, clear content | – | – | Hides overlay, aborts AJAX, empties `l` |
| `Q()` | Show error message in overlay | – | – | Inserts error HTML |
| `B()` | Main entry for opening an item | – | – | Sets up content, starts load |
| `U()` | Called after image load | – | – | Sets image dimensions, inserts `<img>` |
| `G()` | Size layout after content load | – | – | Sets wrapper size |
| `J()` | Show modal, perform animation | – | – | Triggers overlay fade/elastic |
| `M(a)` | Animation step for elastic | `a` progress | – | Animates dimensions |
| `L()` | Finalize layout after animation | – | – | Sets overflow, attaches event handlers |
| `V()` | Compute viewport and modal dimensions | – | object | – |
| `S()` | Compute original element dimensions (for elastic) | – | object | – |
| `X()` | Keyboard/mouse navigation binding | – | – | Binds events |
| `Y()` | Pre‑loads adjacent images | – | – | Creates new `Image` objects |
| `$()` | Animation of loading spinner | – | – | Moves spinner |
| `N()` | Return viewport data | – | array `[width, height, scrollLeft, scrollTop]` | – |
| `Z(a)` | Compute offset & size of a jQuery element | `a` element | object | – |
| `W()` | Build title bar | – | – | Inserts title DOM |
| `aa(a)` | Format title based on position | `a` string | formatted string | – |
| `ba()` | Build all hidden elements (once) | – | – | Creates `#fancybox-wrap` etc. |
| `b.fn.fixPNG()` | IE PNG transparency fix | – | jQuery collection | Applies filter styles |
| `b.fn.fancybox()` | Public plugin method for elements | `a` options | `this` chainable | Binds click handlers |
| `b.fancybox()` | Public static method (programmatic) | `a` items, `d` defaults | – | Opens items |
| `b.fancybox.showActivity()` | Show loading spinner | – | – | Shows spinner |
| `b.fancybox.hideActivity()` | Hide spinner | – | – | Hides spinner |
| `b.fancybox.next()` | Show next item | – | – | Calls `pos()` |
| `b.fancybox.prev()` | Show previous item | – | – | Calls `pos()` |
| `b.fancybox.pos()` | Move to arbitrary index | `a` index | – | Updates `s`, calls `B()` |
| `b.fancybox.cancel()` | Cancel loading | – | – | Triggers `onCancel` |
| `b.fancybox.close()` | Close modal | – | – | Triggers cleanup hooks |
| `b.fancybox.resize()` | Resize modal to fit content | – | – | Adjusts container height |
| `b.fancybox.center()` | Re‑center on viewport | – | – | Sets `left`/`top` |

## 4. Dependencies  

| Library | Role | Notes |
|---------|------|-------|
| **jQuery (≥1.3)** | Core DOM manipulation, event handling, AJAX, animation | Required. No other external libs are used. |
| **jQuery Mousewheel plugin (optional)** | Mouse wheel navigation | If present, attaches wheel handlers. |
| **IE PNG fix** | For old IE versions | Uses CSS filters; wrapped in `b.fn.fixPNG`. |

All other functionality is self‑contained.

## 5. Additional Notes  

### Strengths  
- **Feature‑rich** – supports multiple content types and custom transitions.  
- **Extensible** – public API and hook callbacks make it easy to customize.  
- **Graceful degradation** – fallback to simple lightbox if some features are missing (e.g., no `mousewheel`).  

### Weaknesses / Edge Cases  
- **Legacy code style** – many variable names (`a`, `b`, `c`, etc.) reduce readability.  
- **Hard‑coded dimensions** – `t=20` padding and magic numbers inside layout logic make it brittle.  
- **No modern Promise/A+ support** – AJAX handling relies on callbacks; no ability to cancel with `AbortController`.  
- **IE6/7 PNG fix** – uses proprietary filter syntax and expressions that no longer work in modern browsers.  
- **Memory leaks** – repeated calls to `b.fancybox` may accumulate unbound event handlers if `onCleanup` returns `false`.  
- **Missing accessibility** – no ARIA roles, no focus management when modal opens.  
- **No responsive layout** – size calculations are static; on mobile it may overflow.  

### Potential Enhancements  
1. **Refactor** – rename variables to meaningful names, split logic into smaller modules.  
2. **Modernize** – use ES6 modules, arrow functions, `const/let`, and Promises.  
3. **Accessibility** – trap focus inside modal, add ARIA labels, handle screen readers.  
4. **Responsive design** – adapt width/height to viewport percentages or CSS media queries.  
5. **Improved error handling** – provide more context on AJAX errors, allow custom error callbacks.  
6. **Unit tests** – add tests for each content type and transition path.  
7. **Remove outdated IE hacks** – drop the PNG filter code or provide a modern fallback.  
8. **Documentation** – auto‑generate docs from code comments or use JSDoc.  

Overall, the code is a solid implementation of a classic lightbox plugin, but it is heavily legacy‑styled and would benefit from a modernization effort if used in new projects.

## Code Critique



## Code Preview

```javascript
/*
 * FancyBox - jQuery Plugin
 * Simple and fancy lightbox alternative
 *
 * Copyright (c) 20010 Janis Skarnelis
 * Examples and documentation at: http://fancybox.net
 *
 * Version: 1.3.0 (02/02/2010)
 * Requires: jQuery v1.3+
 *
 * Dual licensed under the MIT and GPL licenses:
 *   http://www.opensource.org/licenses/mit-license.php
 *   http://www.gnu.org/licenses/gpl.html
 */

;(function(b){function H(){v.hide();r.onerror=r.onload=null;F&&F.abort();l.empty()}function Q(){b.fancybox('<p id="fancybox_error">The requested content cannot be loaded.<br />Please try again later.</p>',{scrolling:"no",padding:20,transitionIn:"none",transitionOut:"none"})}function B(){H();var a=q[s];e=b.extend({},b.fn.fancybox.defaults,typeof b(a).data("fancybox")=="undefined"?e:b(a).data("fancybox"));var d,f,o=a.title||b(a).title||e.title||"";if(a.nodeName&&!e.orig)e.orig=b(a).children("img:first").length?
b(a).children("img:first"):b(a);if(o==""&&e.orig)o=e.orig.attr("alt");d=a.nodeName&&/^(?:javascript|#)/i.test(a.href)?e.href||null:e.href||a.href||null;if(e.type){f=e.type;if(!d)d=e.content}else if(e.content)f="html";else if(d)if(d.match(I))f="image";else if(d.match(T))f="swf";else if(b(a).hasClass("iframe"))f="iframe";else if(d.match(/#/)){a=d.substr(d.indexOf("#"));f=b(a).length>0?"inline":"ajax"}else f="ajax";else f="inline";e.type=f;e.href=d;e.title=o;if(e.autoDimensions&&e.type!=="iframe"&&e.type!==
"swf"){e.width="auto";e.height="auto"}if(e.modal){e.overlayShow=true;e.hideOnOverlayClick=false;e.hideOnContentClick=false;e.enableEscapeButton=false;e.showCloseButton=false}if(b.isFunction(e.onStart))if(e.onStart(q,s,e)===false){h=false;return}l.css("padding",t+e.padding+e.margin);b(".fancybox-inline-tmp").unbind("fancybox-cancel").bind("fancybox-change",function(){b(this).replaceWith(i.children())});switch(f){case "html":l.html(e.content);G();break;case "inline":b('<div class="fancybox-inline-tmp" />').hide().insertBefore(b(a)).bind("fancybox-cleanup",
function(){b(this).replaceWith(i.children())}).bind("fancybox-cancel",function(){b(this).replaceWith(l.children())});b(a).appendTo(l);G();break;case "image":h=false;b.fancybox.showActivity();r=new Image;r.onerror=function(){Q()};r.onload=function(){r.onerror=null;r.onload=null;U()};r.src=d;break;case "swf":var u="",w="";u+='<object classid="clsid:D27CDB6E-AE6D-11cf-96B8-444553540000" width="'+e.width+'" height="'+e.height+'"><param name="movie" value="'+d+'"></param>';b.each(e.swf,function(p,R){u+=
'<param name="'+p+'" value="'+R+'"></param>';w+=" "+p+'="'+R+'"'});u+='<embed src="'+d+'" type="application/x-shockwave-flash" width="'+e.width+'" height="'+e.height+'"'+w+"></embed></object>";l.html(u);G();break;case "ajax":a=d.split("#",2);f=e.ajax.data||{};if(a.length>1){d=a[0];typeof f=="string"?(f+="&selector="+a[1]):(f.selector=a[1])}h=false;b.fancybox.showActivity();F=b.ajax(b.extend(e.ajax,{url:d,data:f,error:Q,success:function(p){if(F.status==200){l.html(p);G()}}}));break;case "iframe":b('<iframe id="fancybox-frame" name="fancybox-frame'+
(new Date).getTime()+'" frameborder="0" hspace="0" scrolling="'+e.scrolling+'" src="'+e.href+'"></iframe>').appendTo(l);J();break}}function U(){h=true;e.width=r.width;e.height=r.height;b("<img />").attr({id:"fancybox-img",src:r.src,alt:e.title}).appendTo(l);J()}function G(){l.width(e.width);l.height(e.height);if(e.width=="auto")e.width=l.width();if(e.height=="auto")e.height=l.height();J()}function J(){v.hide();if(g.is(":visible")&&b.isFunction(c.onCleanup))if(c.onCleanup(j,n,c)===false){b.event.trigger("fancybox-cancel");
h=false;return}j=q;n=s;c=e;i.get(0).scrollTop=0;i.get(0).scrollLeft=0;if(c.overlayShow){K&&b("select:not(#fancybox-tmp select)").filter(function(){return this.style.visibility!=="hidden"}).css({visibility:"hidden"}).one("fancybox-cleanup",function(){this.style.visibility="inherit"});y.css({"background-color":c.overlayColor,opacity:c.overlayOpacity}).unbind().show()}m=V();W();if(g.is(":visible")){b(z.add(C).add(D)).hide();var a=g.position();k={top:a.top,left:a.left,width:g.width(),height:g.height()};
var d=k.width==m.width&&k.height==m.height;i.fadeOut(c.changeFade,function(){function f(){i.html(l.contents()).fadeIn(c.changeFade,L)}b.event.trigger("fancybox-change");i.css({top:c.padding,left:c.padding,width:Math.max(k.width-c.padding*2,1),height:Math.max(k.height-c.padding*2,1)}).empty().css("overflow","hidden");A.prop=0;b(A).animate({prop:1},{duration:d?0:c.changeSpeed,easing:c.easingChange,step:M,complete:f})})}else{g.css("opacity",1);if(c.transitionIn=="elastic"){k=S();i.css({top:c.padding,
left:c.padding,width:Math.max(k.width-c.padding*2,1),height:Math.max(k.height-c.padding*2,1)}).html(l.contents());g.css(k).show();if(c.opacity)m.opacity=0;A.prop=0;b(A).animate({prop:1},{duration:c.speedIn,easing:c.easingIn,step:M,complete:L})}else{i.css({top:c.padding,left:c.padding,width:Math.max(m.width-c.padding*2,1),height:Math.max(m.height-c.padding*2-x,1)}).html(l.contents());g.css(m).fadeIn(c.transitionIn=="none"?0:c.speedIn,L)}}}function M(a){var d=Math.round(k.width+(m.width-k.width)*a),
f=Math.round(k.height+(m.height-k.height)*a),o=Math.round(k.top+(m.top-k.top)*a),u=Math.round(k.left+(m.left-k.left)*a);g.css({width:d+"px",height:f+"px",top:o+"px",left:u+"px"});d=Math.max(d-c.padding*2,0);f=Math.max(f-(c.padding*2+x*a),0);i.css({width:d+"px",height:f+"px"});if(typeof m.opacity!=="undefined")g.css("opacity",a<0.5?0.5:a)}function L(){i.css("overflow",overflow=c.scrolling=="auto"?c.type=="image"||c.type=="iframe"||c.type=="swf"?"hidden":"auto":c.scrolling=="yes"?"auto":"visible");
if(!b.support.opacity){i.get(0).style.removeAttribute("filter");g.get(0).style.removeAttribute("filter")}b("#fancybox-title").show();c.hideOnContentClick&&i.one("click",b.fancybox.close);c.hideOnOverlayClick&&y.one("click",b.fancybox.close);c.showCloseButton&&z.show();X();b(window).bind("resize.fb",b.fancybox.center);c.centerOnScroll?b(window).bind("scroll.fb",b.fancybox.center):b(window).unbind("scroll.fb");b.isFunction(c.onComplete)&&c.onComplete(j,n,c);h=false;Y()}function V(){var a=N(),d={},f=
c.margin,o=c.autoScale,u=(t+f)*2,w=(t+f)*2,p=c.padding*2;if(c.width.toString().indexOf("%")>-1){d.width=a[0]*parseFloat(c.width)/100-t*2;o=false}else d.width=c.width+p;if(c.height.toString().indexOf("%")>-1){d.height=a[1]*parseFloat(c.height)/100-t*2;o=false}else d.height=c.height+p;if(o&&(d.width>a[0]-u||d.height>a[1]-w))if(e.type=="image"||e.type=="swf"){u+=p;w+=p;o=Math.min(Math.min(a[0]-u,c.width)/c.width,Math.min(a[1]-w,c.height)/c.height);d.width=Math.round(o*(d.width-p))+p;d.height=Math.round(o*
(d.height-p))+p}else{d.width=Math.min(d.width,a[0]-u);d.height=Math.min(d.height,a[1]-w)}d.top=a[3]+(a[1]-(d.height+t*2))*0.5;d.left=a[2]+(a[0]-(d.width+t*2))*0.5;if(c.autoScale==false){d.top=Math.max(a[3]+f,d.top);d.left=Math.max(a[2]+f,d.left)}return d}function S(){var a=e.orig?b(e.orig):false,d={};if(a&&a.length){a=Z(a);d={width:a.width+c.padding*2,height:a.height+c.padding*2,top:a.top-c.padding-t,left:a.left-c.padding-t}}else{a=N();d={width:1,height:1,top:a[3]+a[1]*0.5,left:a[2]+a[0]*0.5}}return d}
function X(){b(document).unbind("keydown.fb").bind("keydown.fb",function(a){if(a.keyCode==27&&c.enableEscapeButton){a.preventDefault();b.fancybox.close()}else if(a.keyCode==37){a.preventDefault();b.fancybox.prev()}else if(a.keyCode==39){a.preventDefault();b.fancybox.next()}});if(b.fn.mousewheel){g.unbind("mousewheel.fb");j.length>1&&g.bind("mousewheel.fb",function(a,d){a.preventDefault();h||d==0||(d>0?b.fancybox.prev():b.fancybox.next())})}if(c.showNavArrows){if(c.cyclic&&j.length>1||n!=0)C.show();
if(c.cyclic&&j.length>1||n!=j.length-1)D.show()}}function Y(){if(j.length-1>n){var a=j[n+1].href;if(typeof a!=="undefined"&&a.match(I)){var d=new Image;d.src=a}}if(n>0){a=j[n-1].href;if(typeof a!=="undefined"&&a.match(I)){d=new Image;d.src=a}}}function $(){if(v.is(":visible")){b("div",v).css("top",O*-40+"px");O=(O+1)%12}else clearInterval(P)}function N(){return[b(window).width(),b(window).height(),b(document).scrollLeft(),b(document).scrollTop()]}function Z(a){var d=a.offset();d.top+=parseFloat(a.css("paddingTop"))||
0;d.left+=parseFloat(a.css("paddingLeft"))||0;d.top+=parseFloat(a.css("border-top-width"))||0;d.left+=parseFloat(a.css("border-left-width"))||0;d.width=a.width();d.height=a.height();return d}function W(){b("#fancybox-title").remove();x=0;if(c.titleShow!=false){var a=c.title;a=b.isFunction(c.titleFormat)?c.titleFormat(a,j,n,c):aa(a);if(!(!a||a=="")){var d=m.width-c.padding*2;b('<div id="fancybox-title" class="'+("fancybox-title-"+c.titlePosition)+'" />').css({width:d,paddingLeft:c.padding,paddingRight:c.padding}).html(a).appendTo("body");
switch(c.titlePosition){case "inside":x=b("#fancybox-title").outerHeight(true)-c.padding;m.height+=x;break;case "over":b("#fancybox-title").css("bottom",c.padding);break;default:b("#fancybox-title").css("bottom",b("#fancybox-title").outerHeight(true)*-1);break}b("#fancybox-title").appendTo(E).hide();K&&b("#fancybox-title span").fixPNG()}}}function aa(a){if(a&&a.length)switch(c.titlePosition){case "inside":return a;case "over":return'<span id="fancybox-title-over">'+a+"</span>";default:return'<span id="fancybox-title-wrap"><span id="fancybox-title-left"></span><span id="fancybox-title-main">'+
a+'</span><span id="fancybox-title-right"></span></span>'}return false}function ba(){if(!b("#fancybox-wrap").length){b("body").append(l=b('<div id="fancybox-tmp"></div>'),v=b('<div id="fancybox-loading"><div></div></div>'),y=b('<div id="fancybox-overlay"></div>'),g=b('<div id="fancybox-wrap"></div>'));E=b('<div id="fancybox-outer"></div>').append('<div class="fancy-bg" id="fancy-bg-n"></div><div class="fancy-bg" id="fancy-bg-ne"></div><div class="fancy-bg" id="fancy-bg-e"></div><div class="fancy-bg" id="fancy-bg-se"></div><div class="fancy-bg" id="fancy-bg-s"></div><div class="fancy-bg" id="fancy-bg-sw"></div><div class="fancy-bg" id="fancy-bg-w"></div><div class="fancy-bg" id="fancy-bg-nw"></div>').appendTo(g);
E.append(i=b('<div id="fancybox-inner"></div>'),z=b('<a id="fancybox-close"></a>'),C=b('<a href="javascript:;" id="fancybox-left"><span class="fancy-ico" id="fancybox-left-ico"></span></a>'),D=b('<a href="javascript:;" id="fancybox-right"><span class="fancy-ico" id="fancybox-right-ico"></span></a>'));z.click(b.fancybox.close);v.click(b.fancybox.cancel);C.click(function(a){a.preventDefault();b.fancybox.prev()});D.click(function(a){a.preventDefault();b.fancybox.next()});b.support.opacity||E.find(".fancy-bg").fixPNG();
if(K){b(z.add(".fancy-ico").add("div",v)).fixPNG();y.get(0).style.setExpression("height","document.body.scrollHeight > document.body.offsetHeight ? document.body.scrollHeight : document.body.offsetHeight + 'px'");v.get(0).style.setExpression("top","(-20 + (document.documentElement.clientHeight ? document.documentElement.clientHeight/2 : document.body.clientHeight/2 ) + ( ignoreMe = document.documentElement.scrollTop ? document.documentElement.scrollTop : document.body.scrollTop )) + 'px'");E.prepend('<iframe id="fancybox-hide-sel-frame" src="javascript:\'\';" scrolling="no" frameborder="0" ></iframe>')}}}
var l,v,y,g,E,i,z,C,D,s=0,e={},q=[],n=0,c={},j=[],F=null,r=new Image,I=/\.(jpg|gif|png|bmp|jpeg)(.*)?$/i,T=/[^\.]\.(swf)\s*$/i,P,O=1,k,m,h=false,t=20,A=b.extend(b("<div/>")[0],{prop:0}),x=0,K=!b.support.opacity&&!window.XMLHttpRequest;b.fn.fixPNG=function(){return this.each(function(){var a=b(this).css("backgroundImage");if(a.match(/^url\(["']?(.*\.png)["']?\)$/i)){a=RegExp.$1;b(this).css({backgroundImage:"none",filter:"progid:DXImageTransform.Microsoft.AlphaImageLoader(enabled=true, sizingMethod="+
(b(this).css("backgroundRepeat")=="no-repeat"?"crop":"scale")+", src='"+a+"')"}).each(function(){var d=b(this).css("position");d!="absolute"&&d!="relative"&&b(this).css("position","relative")}).css("zoom",1)}})};b.fn.fancybox=function(a){b(this).data("fancybox",b.extend({},a));b(this).unbind("click.fb").bind("click.fb",function(d){d.preventDefault();if(!h){h=true;b(this).blur();q=[];s=0;d=b(this).attr("rel")||"";if(!d||d==""||d==="nofollow")q.push(this);else{q=b("a[rel="+d+"], area[rel="+d+"]");s=
q.index(this)}B();return false}});return this};b.fancybox=function(a,d){if(!h){h=true;q=[];s=0;if(b.isArray(a)){for(var f=0,o=a.length;f<o;f++)if(typeof a[f]=="object")b(a[f]).data("fancybox",b.extend({},d,a[f]));else a[f]=b({}).data("fancybox",b.extend({content:a[f]},d));q=jQuery.merge(q,a)}else{if(typeof a=="object")b(a).data("fancybox",b.extend({},d,a));else a=b({}).data("fancybox",b.extend({content:a},d));q.push(a)}B()}};b.fancybox.showActivity=function(){clearInterval(P);v.show();P=setInterval($,
66)};b.fancybox.hideActivity=function(){v.hide()};b.fancybox.next=function(){return b.fancybox.pos(n+1)};b.fancybox.prev=function(){return b.fancybox.pos(n-1)};b.fancybox.pos=function(a){if(!h){a=parseInt(a);if(a>-1&&j.length>a){s=a;B()}if(c.cyclic&&j.length>1&&a<0){s=j.length-1;B()}if(c.cyclic&&j.length>1&&a>=j.length){s=0;B()}}};b.fancybox.cancel=function(){if(!h){h=true;b.event.trigger("fancybox-cancel");H();e&&b.isFunction(e.onCancel)&&e.onCancel(q,s,e);h=false}};b.fancybox.close=function(){function a(){y.fadeOut("fast");
g.hide();b.event.trigger("fancybox-cleanup");i.empty();b.isFunction(c.onClosed)&&c.onClosed(j,n,c);j=e=[];n=s=0;c=e={};h=false}if(!(h||g.is(":hidden"))){h=true;if(c&&b.isFunction(c.onCleanup))if(c.onCleanup(j,n,c)===false){h=false;return}H();b(z.add(C).add(D)).hide();b("#fancybox-title").remove();g.add(i).add(y).unbind();b(window).unbind("resize.fb scroll.fb");b(document).unbind("keydown.fb");i.css("overflow","hidden");if(c.transitionOut=="elastic"){k=S();var d=g.position();m={top:d.top,left:d.left,
width:g.width(),height:g.height()};if(c.opacity)m.opacity=1;A.prop=1;b(A).animate({prop:0},{duration:c.speedOut,easing:c.easingOut,step:M,complete:a})}else g.fadeOut(c.transitionOut=="none"?0:c.speedOut,a)}};b.fancybox.resize=function(){if(!(h||g.is(":hidden"))){h=true;var a=i.wrapInner("<div style='overflow:auto'></div>").children(),d=a.height();g.css({height:d+c.padding*2+x});i.css({height:d});a.replaceWith(a.children());b.fancybox.center()}};b.fancybox.center=function(){h=true;var a=N(),d=c.margin,
f={};f.top=a[3]+(a[1]-(g.height()-x+t*2))*0.5;f.left=a[2]+(a[0]-(g.width()+t*2))*0.5;f.top=Math.max(a[3]+d,f.top);f.left=Math.max(a[2]+d,f.left);g.css(f);h=false};b.fn.fancybox.defaults={padding:10,margin:20,opacity:false,modal:false,cyclic:false,scrolling:"auto",width:560,height:340,autoScale:true,autoDimensions:true,centerOnScroll:false,ajax:{},swf:{wmode:"transparent"},hideOnOverlayClick:true,hideOnContentClick:false,overlayShow:true,overlayOpacity:0.3,overlayColor:"#666",titleShow:true,titlePosition:"outside",
titleFormat:null,transitionIn:"fade",transitionOut:"fade",speedIn:300,speedOut:300,changeSpeed:300,changeFade:"fast",easingIn:"swing",easingOut:"swing",showCloseButton:true,showNavArrows:true,enableEscapeButton:true,onStart:null,onCancel:null,onComplete:null,onCleanup:null,onClosed:null};b(document).ready(function(){ba()})})(jQuery);


```
