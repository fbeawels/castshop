# ckeditor_basic.js

## Review

## 1. Summary  

The snippet is the bootstrap code for **CKEditor 3.0.1** – the first “tiny” core that is shipped with the editor.  
Its responsibilities are:

| Component | What it does |
|-----------|--------------|
| **`CKEDITOR` namespace** | Holds global constants, paths, environment data and the event system. |
| **Path resolution** | Computes `CKEDITOR.basePath` by inspecting the current script tag or the `CKEDITOR_BASEPATH` variable. |
| **`CKEDITOR.getUrl()`** | Normalises URLs and appends a cache‑buster (`?t=...`). |
| **Event system** | Lightweight publish/subscribe implementation (`on`, `fire`, `fireOnce`, `removeListener`, `hasListeners`). |
| **Environment sniffing (`CKEDITOR.env`)** | Detects browser type, version, quirks mode, and compatibility. |
| **Editor constructors** | `CKEDITOR.editor` + helpers `replace` / `appendTo`. |
| **Full core loader** | `CKEDITOR.loadFullCore()` loads `ckeditor.js` asynchronously if the environment is compatible, otherwise the editor is unusable. |
| **Automatic replacement** | If `replaceByClassEnabled` is true, the script looks for all `<textarea>` elements with the class `ckeditor` and replaces them on `window.onload`. |

The code is intentionally compact (single‑file, IIFE) and uses only native browser APIs (no third‑party libraries). It is a classic “polyfill‑heavy” approach that targets IE 6–8, Gecko 1.8, WebKit 522+, Opera 9.5+ and Adobe AIR.

---

## 2. Detailed Description  

### 2.1 Initialization flow

1. **Global IIFE** – Wraps the entire file, protecting the global scope and creating the `CKEDITOR` object if it does not already exist.
2. **Base path discovery** – Uses `CKEDITOR_BASEPATH` if defined; otherwise iterates over `<script>` tags to find the one ending in `ckeditor.js` (or variants). If the path is relative, it is converted to an absolute URL based on the current document location.
3. **Cache‑buster** – `CKEDITOR.getUrl()` appends a timestamp (`CKEDITOR.timestamp`) to every non‑absolute URL that doesn’t already end with `/`.
4. **Event mix‑in** – The lightweight event system is attached to the `CKEDITOR` namespace and later mixed into `CKEDITOR.editor` prototypes.
5. **Environment detection** – `CKEDITOR.env` contains booleans for IE, Gecko, Opera, WebKit, AIR, quirks mode, as well as numeric version and a compatibility flag.
6. **Basic ready / loaded states** – `CKEDITOR.status` toggles between `'unloaded'`, `'basic_ready'`, `'basic_loaded'`, etc. The basic core is considered ready when the environment check passes.
7. **Full core loader** – If the browser is compatible (`env.isCompatible`), `CKEDITOR.loadFullCore()` is invoked to asynchronously load the full `ckeditor.js` file. A timeout can be set to trigger this later.
8. **Automatic replacement** – On `window.onload` (or `DOMContentLoaded`‑style handling) the script scans the document for all `<textarea>` elements with the class `ckeditor` (unless `replaceByClassEnabled` is disabled) and calls `CKEDITOR.replace()` on each.

### 2.2 Editor life‑cycle

- **`CKEDITOR.editor` constructor**  
  Creates an editor instance attached to a DOM element (`element`). The constructor sets up instance data (`_instanceConfig`), the `elementMode` (none / replace / append), and calls `_init()` which pushes the instance into a pending list.

- **`replace` / `appendTo` helpers**  
  These are thin wrappers that locate the target element (by id or name) and call the constructor with the appropriate `elementMode`. `replace` hides the original `<textarea>` (`visibility:hidden`) before instantiation.

- **Pending queue**  
  Editor instances are queued until the full core (`ckeditor.js`) is loaded. Once loaded, the editor factory processes this queue to instantiate the real editor UI.

- **Event handling**  
  Each editor instance inherits the event system, enabling plugins and user code to listen to editor events (`on`, `fire`, etc.).

### 2.3 Dependencies & Assumptions

- **Browser support** – The code assumes a fairly old JavaScript engine: IE6+ (uses `document.documentMode`), older Opera, Gecko 1.8, WebKit 522+.  
- **No external libraries** – The core is entirely self‑contained.  
- **Global variable** – The editor uses the global `CKEDITOR` object; no module system.  
- **Document structure** – Relies on `<script>` tags and `<textarea>` elements being present in the DOM when the script executes.  
- **Network** – The full core is loaded via a `<script>` tag insertion; no fallback if the network fails.

---

## 3. Functions / Methods  

| Function / Method | Purpose | Inputs | Outputs | Side‑effects |
|-------------------|---------|--------|---------|--------------|
| **`CKEDITOR` (object)** | Namespace holder | – | – | Sets up `timestamp`, `version`, `revision`, etc. |
| **`getUrl(d)`** | Normalise relative URLs and add cache‑buster | `d` – string URL | Normalised URL string | None |
| **`event` constructor** | Creates a lightweight event object | – | Event instance | Adds prototype methods (`on`, `fire`, …) |
| **`event.prototype.on(name, fn, data, priority)`** | Register event listener | `name` – string, `fn` – callback, optional `data`, `priority` | – | Adds listener to internal list |
| **`event.prototype.fire(name, data)`** | Trigger listeners | `name`, `data` | Boolean (whether default prevented) | Calls all listeners in priority order |
| **`event.prototype.fireOnce(name, data)`** | Trigger once, then remove listeners | – | – | Removes listeners after first trigger |
| **`event.prototype.removeListener(name, fn)`** | Remove a specific listener | – | – | Mutates listener list |
| **`event.prototype.hasListeners(name)`** | Check for listeners | – | Boolean | – |
| **`CKEDITOR.env`** | Browser detection object | – | – | Populates booleans (`ie`, `gecko`, `opera`, `webkit`, `air`, `quirks`) and numeric `version` |
| **`CKEDITOR.editor` constructor** | Create an editor instance | `config`, `element`, `mode` | `this` | Adds instance to pending queue |
| **`CKEDITOR.editor.prototype._init()`** | Push instance to pending list | – | – | Mutates `_pending` |
| **`CKEDITOR.editor.prototype.fire(name, data)`** | Delegate to event system | – | – | – |
| **`CKEDITOR.editor.prototype.fireOnce(name, data)`** | Delegate to event system | – | – | – |
| **`CKEDITOR.replace(elementOrId, config)`** | Replace textarea with editor | `elementOrId` – string or element, `config` – object | Editor instance | Hides original element |
| **`CKEDITOR.appendTo(elementOrId, config)`** | Append editor to container | `elementOrId` – string or element, `config` – object | Editor instance | – |
| **`CKEDITOR.add(editorInstance)`** | Queue an editor instance for later processing | `editorInstance` | – | Adds to `_pending` |
| **`CKEDITOR.replaceAll([selector])`** | Replace all textareas with the given class or filter function | Optional selector string or filter function | – | Calls `replace()` for each qualifying textarea |
| **`CKEDITOR.loadFullCore()`** | Asynchronously load `ckeditor.js` if not already loaded | – | – | Injects `<script>` tag into `<head>` |
| **`CKEDITOR.getUrl(d)` (internal helper)** | Resolve relative URLs for script injection | – | – | – |

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| **`document` / `window`** | Native | Standard DOM, no external libs. |
| **`navigator.userAgent`** | Native | Used for browser sniffing. |
| **`setTimeout`** | Native | Delays load of full core. |
| **`addEventListener` / `attachEvent`** | Native | Cross‑browser event binding for `load`. |
| **`document.createElement('script')`** | Native | Dynamic script loading. |
| **`document.getElementsByTagName` / `getElementsByName`** | Native | Element selection. |
| **`Array.prototype.slice`** | Native | Used for event listener cloning. |
| **`String.prototype.match` / `replace`** | Native | URL and class name parsing. |

No third‑party libraries are required; all features are built on top of the standard browser API.

---

## 5. Additional Notes  

### 5.1 Strengths  
* **Self‑contained bootstrap** – No external dependencies or build tools needed.  
* **Graceful degradation** – The core checks for browser compatibility before attempting to load the full editor, preventing crashes in unsupported browsers.  
* **Lazy loading** – The full editor is loaded asynchronously only when needed, keeping the initial payload small.  
* **Extensible event system** – Provides a minimal but functional pub/sub layer that plugins can hook into.  

### 5.2 Weaknesses / Edge Cases  
1. **Browser sniffing** – Uses hard‑coded regexes and legacy IE properties (`documentMode`). Modern browsers or updated engines may not match correctly, leading to false negatives (e.g., IE11 in edge mode).  
2. **No error handling for script load failure** – If `ckeditor.js` fails to load, the editor never gets instantiated and the user sees no error message.  
3. **Global namespace pollution** – The entire library sits on the global `CKEDITOR` object; potential conflicts with other scripts that define the same name.  
4. **Memory leaks** – Event listeners are never cleaned up on editor destruction; if an editor is removed from the DOM, its listeners may still reference it.  
5. **Deprecated APIs** – `document.compatMode` and `document.documentMode` are legacy; the code still checks for quirks mode but does not adapt to modern standards mode.  
6. **Synchronous script tag lookup** – `CKEDITOR.basePath` relies on the order of `<script>` tags; if the CKEditor script is not the last script in the page, the path may be mis‑calculated.  

### 5.3 Potential Enhancements  
* **Modularization** – Wrap the bootstrap in an AMD/ES6 module or expose it via a UMD wrapper, enabling tree‑shaking and better integration with modern build systems.  
* **Promise‑based loader** – Replace the ad‑hoc `<script>` insertion with a Promise that resolves when the script is loaded, allowing better control flow.  
* **Better browser detection** – Use feature detection instead of user‑agent sniffing (e.g., `navigator.userAgentData` in modern browsers).  
* **Error handling** – Hook into `script.onerror` to notify users or fallback to a minimal editor mode.  
* **Cleaner cleanup** – Provide an `destroy` method on editor instances that removes all event listeners and DOM nodes.  
* **Internationalization** – Expose a locale‑aware message system instead of hard‑coding strings.  

### 5.4 Usage Context  
This snippet is the minimal bootstrap that ships with CKEditor 3.x. In a production environment it is usually followed by the full `ckeditor.js` file (the “full core”) and a collection of plugins. The bootstrap’s job is simply to detect the environment, expose the `CKEDITOR` namespace, and load the rest of the library on demand. The code is heavily commented, which aids maintenance, but the style is a mixture of ES3‑style and ES5 features; refactoring to a more modern style (const/let, arrow functions, classes) would improve readability and future‑proof the library.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){if(!window.CKEDITOR)window.CKEDITOR=(function(){var a={timestamp:'99GE',version:'3.0.1',revision:'4391',_:{},status:'unloaded',basePath:(function(){var d=window.CKEDITOR_BASEPATH||'';if(!d){var e=document.getElementsByTagName('script');for(var f=0;f<e.length;f++){var g=e[f].src.match(/(^|.*[\\\/])ckeditor(?:_basic)?(?:_source)?.js(?:\?.*)?$/i);if(g){d=g[1];break;}}}if(d.indexOf('://')==-1)if(d.indexOf('/')===0)d=location.href.match(/^.*?:\/\/[^\/]*/)[0]+d;else d=location.href.match(/^[^\?]*\/(?:)/)[0]+d;return d;})(),getUrl:function(d){if(d.indexOf('://')==-1&&d.indexOf('/')!==0)d=this.basePath+d;if(this.timestamp&&d.charAt(d.length-1)!='/')d+=(d.indexOf('?')>=0?'&':'?')+('t=')+this.timestamp;return d;}},b=window.CKEDITOR_GETURL;if(b){var c=a.getUrl;a.getUrl=function(d){return b.call(a,d)||c.call(a,d);};}return a;})();var a=CKEDITOR;if(!a.event){a.event=function(){};a.event.implementOn=function(b,c){var d=a.event.prototype;for(var e in d)if(b[e]==undefined)b[e]=d[e];};a.event.prototype=(function(){var b=function(d){var e=d.getPrivate&&d.getPrivate()||d._||(d._={});return e.events||(e.events={});},c=function(d){this.name=d;this.listeners=[];};c.prototype={getListenerIndex:function(d){for(var e=0,f=this.listeners;e<f.length;e++)if(f[e].fn==d)return e;return-1;}};return{on:function(d,e,f,g,h){var i=b(this),j=i[d]||(i[d]=new c(d));if(j.getListenerIndex(e)<0){var k=j.listeners;if(!f)f=this;if(isNaN(h))h=10;var l=this,m=function(o,p,q,r){var s={name:d,sender:this,editor:o,data:p,listenerData:g,stop:q,cancel:r,removeListener:function(){l.removeListener(d,e);}};e.call(f,s);return s.data;};m.fn=e;m.priority=h;for(var n=k.length-1;n>=0;n--)if(k[n].priority<=h){k.splice(n+1,0,m);return;}k.unshift(m);}},fire:(function(){var d=false,e=function(){d=true;},f=false,g=function(){f=true;};return function(h,i,j){var k=b(this)[h],l=d,m=f;d=f=false;if(k){var n=k.listeners;if(n.length){n=n.slice(0);for(var o=0;o<n.length;o++){var p=n[o].call(this,j,i,e,g);if(typeof p!='undefined')i=p;if(d||f)break;}}}var q=f||(typeof i=='undefined'?false:i);d=l;f=m;return q;};})(),fireOnce:function(d,e,f){var g=this.fire(d,e,f);delete b(this)[d];return g;},removeListener:function(d,e){var f=b(this)[d];if(f){var g=f.getListenerIndex(e);if(g>=0)f.listeners.splice(g,1);}},hasListeners:function(d){var e=b(this)[d];return e&&e.listeners.length>0;}};})();}if(!a.editor){a.ELEMENT_MODE_NONE=0;a.ELEMENT_MODE_REPLACE=1;a.ELEMENT_MODE_APPENDTO=2;a.editor=function(b,c,d){var e=this;e._={instanceConfig:b,element:c};
e.elementMode=d||0;a.event.call(e);e._init();};a.editor.replace=function(b,c){var d=b;if(typeof d!='object'){d=document.getElementById(b);if(!d){var e=0,f=document.getElementsByName(b);while((d=f[e++])&&(d.tagName.toLowerCase()!='textarea')){}}if(!d)throw '[CKEDITOR.editor.replace] The element with id or name "'+b+'" was not found.';}d.style.visibility='hidden';return new a.editor(c,d,1);};a.editor.appendTo=function(b,c){if(typeof b!='object'){b=document.getElementById(b);if(!b)throw '[CKEDITOR.editor.appendTo] The element with id "'+b+'" was not found.';}return new a.editor(c,b,2);};a.editor.prototype={_init:function(){var b=a.editor._pending||(a.editor._pending=[]);b.push(this);},fire:function(b,c){return a.event.prototype.fire.call(this,b,c,this);},fireOnce:function(b,c){return a.event.prototype.fireOnce.call(this,b,c,this);}};a.event.implementOn(a.editor.prototype,true);}if(!a.env)a.env=(function(){var b=navigator.userAgent.toLowerCase(),c=window.opera,d={ie:/*@cc_on!@*/false,opera:!!c&&c.version,webkit:b.indexOf(' applewebkit/')>-1,air:b.indexOf(' adobeair/')>-1,mac:b.indexOf('macintosh')>-1,quirks:document.compatMode=='BackCompat',isCustomDomain:function(){return this.ie&&document.domain!=window.location.hostname;}};d.gecko=navigator.product=='Gecko'&&!d.webkit&&!d.opera;var e=0;if(d.ie){e=parseFloat(b.match(/msie (\d+)/)[1]);d.ie8=!!document.documentMode;d.ie8Compat=document.documentMode==8;d.ie7Compat=e==7&&!document.documentMode||document.documentMode==7;d.ie6Compat=e<7||d.quirks;}if(d.gecko){var f=b.match(/rv:([\d\.]+)/);if(f){f=f[1].split('.');e=f[0]*10000+(f[1]||0)*(100)+ +(f[2]||0);}}if(d.opera)e=parseFloat(c.version());if(d.air)e=parseFloat(b.match(/ adobeair\/(\d+)/)[1]);if(d.webkit)e=parseFloat(b.match(/ applewebkit\/(\d+)/)[1]);d.version=e;d.isCompatible=d.ie&&e>=6||d.gecko&&e>=10801||d.opera&&e>=9.5||d.air&&e>=1||d.webkit&&e>=522||false;d.cssClass='cke_browser_'+(d.ie?'ie':d.gecko?'gecko':d.opera?'opera':d.air?'air':d.webkit?'webkit':'unknown');if(d.quirks)d.cssClass+=' cke_browser_quirks';if(d.ie){d.cssClass+=' cke_browser_ie'+(d.version<7?'6':d.version>=8?'8':'7');if(d.quirks)d.cssClass+=' cke_browser_iequirks';}if(d.gecko&&e<10900)d.cssClass+=' cke_browser_gecko18';return d;})();var b=a.env;var c=b.ie;if(a.status=='unloaded')(function(){a.event.implementOn(a);a.loadFullCore=function(){if(a.status!='basic_ready'){a.loadFullCore._load=true;return;}delete a.loadFullCore;var e=document.createElement('script');e.type='text/javascript';
e.src=a.basePath+'ckeditor.js';document.getElementsByTagName('head')[0].appendChild(e);};a.loadFullCoreTimeout=0;a.replaceClass='ckeditor';a.replaceByClassEnabled=true;var d=function(e,f,g){if(b.isCompatible){if(a.loadFullCore)a.loadFullCore();var h=g(e,f);a.add(h);return h;}return null;};a.replace=function(e,f){return d(e,f,a.editor.replace);};a.appendTo=function(e,f){return d(e,f,a.editor.appendTo);};a.add=function(e){var f=this._.pending||(this._.pending=[]);f.push(e);};a.replaceAll=function(){var e=document.getElementsByTagName('textarea');for(var f=0;f<e.length;f++){var g=null,h=e[f],i=h.name;if(!h.name&&!h.id)continue;if(typeof arguments[0]=='string'){var j=new RegExp('(?:^| )'+arguments[0]+'(?:$| )');if(!j.test(h.className))continue;}else if(typeof arguments[0]=='function'){g={};if(arguments[0](h,g)===false)continue;}this.replace(h,g);}};(function(){var e=function(){var f=a.loadFullCore,g=a.loadFullCoreTimeout;if(a.replaceByClassEnabled)a.replaceAll(a.replaceClass);a.status='basic_ready';if(f&&f._load)f();else if(g)setTimeout(function(){if(a.loadFullCore)a.loadFullCore();},g*1000);};if(window.addEventListener)window.addEventListener('load',e,false);else if(window.attachEvent)window.attachEvent('onload',e);})();a.status='basic_loaded';})();})();



```
