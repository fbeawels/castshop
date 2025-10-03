# plugin.js

## Review

## 1. Summary  

**Purpose** –  
The file implements the *floatpanel* plugin for CKEditor.  
A floatpanel is a floating UI element that can display content (blocks, lists, etc.) next to a trigger element (typically a toolbar button).  It is used throughout CKEditor for context‑sensitive panels such as the color picker, font size selector, image dialog, etc.

**Key Components**  

| Component | Role |
|-----------|------|
| `CKEDITOR.plugins.add('floatpanel', …)` | Declares the plugin and its dependency on the core `panel` plugin. |
| `c(d,e,f,g,h)` | Helper that creates or re‑uses a `CKEDITOR.ui.panel` instance based on a composite key (skin, language, color, css, custom hash). |
| `CKEDITOR.ui.floatPanel` | Main UI class exposed to CKEditor.  It wraps a `panel` instance, manages positioning, focus/blur, visibility, and child panels. |
| Prototype methods (`addBlock`, `addListBlock`, `getBlock`, `showBlock`, `hide`, `allowBlur`, `showAsChild`, `hideChild`) | Public API used by editor components to interact with the panel. |

**Notable Patterns & Libraries**  

* Uses CKEditor’s `CKEDITOR.tools.createClass` to build a constructor‑based class with prototype methods.  
* Employs *event capture* (`CKEDITOR.event.useCapture`) for focus/blur handling inside the panel’s iframe.  
* Leverages CKEditor’s DOM abstraction (`CKEDITOR.dom.element`, `CKEDITOR.dom.window`) for cross‑browser DOM manipulation.  
* Compatibility code for older Internet Explorer (IE6‑8 quirks/compat modes) is sprinkled throughout.

---

## 2. Detailed Description  

### Initialization Flow  

1. **Plugin Registration** – `CKEDITOR.plugins.add('floatpanel', { requires: ['panel'] })` registers the plugin; CKEditor will load the core `panel` plugin first.  
2. **Helper Cache** – A global object `a` holds cached panel instances keyed by a string built in `c`.  
3. **`c` Function** –  
   * Builds a unique key from skin name, language direction, UI color, CSS class, and an optional hash.  
   * If a panel with that key already exists, it is reused; otherwise a new `CKEDITOR.ui.panel` is instantiated, rendered, and appended to the document.  
   * The returned panel is stored in the cache and its root element is styled `display:none; position:absolute`.  

4. **`floatPanel` Constructor** –  
   * Parameters: `definition`, `parentElement`, `editor`, `config`.  
   * Calls `c` to get (or create) the underlying panel.  
   * Stores references to the panel, parent element, definition, editor document, iframe (panel’s content window), and an array for child panels.  
   * Adds the panel’s root element to the editor’s `panels` array.

### Runtime Behaviour  

* **Block Management** – `addBlock`, `addListBlock`, `getBlock` delegate directly to the underlying panel.  
* **Displaying a Block** (`showBlock`)  
  * Retrieves the target block, tells the panel to show it, and prepares for positioning.  
  * Calculates initial coordinates relative to the trigger element (`e`).  
  * Handles RTL layout adjustments for horizontal positioning.  
  * Temporarily hides the panel (`visibility:hidden; opacity:0`) and positions it off‑screen (`left:-3000px`).  
  * Sets up **blur/focus event listeners** on the iframe’s window (or the content window in IE) to hide the panel when clicking outside.  
  * Handles the *escape* key by binding a handler that calls `onEscape`.  
  * After a short timeout, slides the panel into view with a smooth opacity transition.  
  * If `autoSize` is requested, it waits for the panel to load before sizing the block to match its content height.  
  * For accessibility: adds `role="region"` and a title in Gecko engines.  
  * Sets focus to the panel’s iframe for keyboard interaction and marks `allowBlur` accordingly.  

* **Hiding** (`hide`) – sets the element to `display:none` and clears the `visible` flag unless a custom `onHide` handler returns `true`.  
* **Focus Handling** – `allowBlur` controls whether the panel can be dismissed on focus loss.  
* **Child Panels** – `showAsChild` and `hideChild` allow a panel to host another panel (e.g., an image dialog inside a toolbar button).  
  * `showAsChild` hides any existing child, attaches event handlers, and displays the new child panel relative to its trigger.  
  * `hideChild` simply calls `hide` on the active child.

### Cleanup  

The plugin does not expose a dedicated cleanup method; panels are removed when the editor instance is destroyed.  Event listeners are attached directly to the iframe’s window, so they will be garbage‑collected when the editor is destroyed.

---

## 3. Functions/Methods  

| Name | Purpose | Inputs | Outputs | Side‑Effects |
|------|---------|--------|---------|--------------|
| **c** *(d, e, f, g, h)* | Creates/reuses a panel UI. | `definition` (panel definition), `editor` doc, `element` to attach to, optional `css` string, optional `hash` string. | `CKEDITOR.ui.panel` instance. | Adds root element to the document, caches panel. |
| **constructor** (`$`) | Instantiates a `floatPanel`. | `definition`, `parentElement`, `editor`, optional config. | `this` (floatPanel instance). | Stores references, registers panel element in editor. |
| **addBlock** | Adds a block to the panel. | `type`, `content`. | Panel block reference. | Delegates to underlying panel. |
| **addListBlock** | Adds a list block. | `type`, `list`. | Panel block reference. | Delegates. |
| **getBlock** | Retrieves a block by id. | `id`. | Block reference. | Delegates. |
| **showBlock** | Displays a block relative to a trigger element. | `blockId`, `triggerElement`, `align`, `hAlign`, `vAlign`, optional `childHash`. | None. | Positions panel, handles focus/blur, starts fade‑in. |
| **hide** | Hides the panel. | None. | None. | Sets `display:none`. |
| **allowBlur** | Gets/sets whether the panel can be hidden on blur. | Optional `bool`. | Current allowBlur value. | Sets property on underlying panel. |
| **showAsChild** | Shows a child panel (another `floatPanel`). | `childPanel`, `triggerElement`, `align`, `hAlign`, `vAlign`, optional `childHash`. | None. | Hides previous child, sets up event handlers, displays child. |
| **hideChild** | Hides the active child panel. | None. | None. | Calls `hide` on child. |

*Utility* – `CKEDITOR.tools.setTimeout`, `CKEDITOR.tools.bind`, `CKEDITOR.tools.addClass`, etc. are used internally but are part of CKEditor’s core utilities.

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| **CKEditor core** (`CKEDITOR`, `CKEDITOR.tools`, `CKEDITOR.event`, `CKEDITOR.dom`) | Standard | Provides DOM abstraction, event system, and utilities. |
| **panel plugin** (`panel`) | Third‑party CKEditor plugin | Required to render the panel UI; defines `CKEDITOR.ui.panel`. |
| **Browser APIs** (`window`, `document`) | Platform | Accessed through CKEditor wrappers; no external libs. |
| **IE Compatibility** | Platform | Uses conditional checks for `CKEDITOR.env.ie`, `quirks`, `compatMode`. |

No other external libraries are referenced.

---

## 5. Additional Notes  

### Strengths  

* **Encapsulation** – The floatpanel API is clean and wraps the underlying panel implementation, making it reusable across different editor components.  
* **Cross‑Browser Handling** – Extensive fallback logic for older IE versions keeps the plugin functional on legacy browsers.  
* **Accessibility** – Adds ARIA roles and titles for Gecko engines; the focus logic supports keyboard interaction.  

### Potential Issues / Edge Cases  

1. **Global State** – The cache `a` and flag `b` are global variables within the IIFE.  If multiple editor instances coexist on the same page, they share the same cache, potentially causing panels to be reused incorrectly across editors.  
2. **Memory Leaks** – Event listeners attached to iframe windows (`blur`, `focus`) are not explicitly removed.  In complex pages with many editors, this could lead to memory retention.  
3. **Positioning Calculations** – The logic for RTL adjustments and `left:-3000px` hack may fail on very large or transformed layouts (e.g., zoom, scrolling parents).  
4. **IE7/8 Compatibility Code** – The manual CSS injection and delayed style resets are brittle and rely on timing (`setTimeout`).  Unexpected layout changes might break the panel positioning.  
5. **Accessibility on Non‑Gecko** – Only Gecko receives `role="region"` and title attributes.  Other browsers may not announce the panel to screen readers.  

### Future Enhancements  

* **Instance‑Scoped Cache** – Move the panel cache inside the plugin instance to avoid cross‑editor contamination.  
* **Explicit Event Cleanup** – Provide a `destroy` method that removes all event listeners and clears references.  
* **Responsive Positioning** – Replace hard‑coded offsets with a more robust layout engine (e.g., use `getBoundingClientRect` and scroll parents).  
* **Better Accessibility** – Add ARIA attributes across all browsers and expose `aria-labelledby` hooks for custom panels.  
* **Unit Tests** – Write automated tests for the positioning logic and event handling, especially for the edge cases above.  

--- 

**Conclusion** –  
The floatpanel plugin is a well‑structured, heavily cached UI component that handles a variety of browser quirks.  While functional, it could benefit from tighter encapsulation, more graceful cleanup, and modern positioning/ARIA practices to ensure long‑term maintainability.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.plugins.add('floatpanel',{requires:['panel']});(function(){var a={},b=false;function c(d,e,f,g,h){var i=e.getUniqueId()+'-'+f.getUniqueId()+'-'+d.skinName+'-'+d.lang.dir+(d.uiColor&&'-'+d.uiColor||'')+(g.css&&'-'+g.css||'')+(h&&'-'+h||''),j=a[i];if(!j){j=a[i]=new CKEDITOR.ui.panel(e,g);j.element=f.append(CKEDITOR.dom.element.createFromHtml(j.renderHtml(d),e));j.element.setStyles({display:'none',position:'absolute'});}return j;};CKEDITOR.ui.floatPanel=CKEDITOR.tools.createClass({$:function(d,e,f,g){f.forceIFrame=true;var h=e.getDocument(),i=c(d,h,e,f,g||0),j=i.element,k=j.getFirst().getFirst();this.element=j;d.panels?d.panels.push(j):d.panels=[j];this._={panel:i,parentElement:e,definition:f,document:h,iframe:k,children:[],dir:d.lang.dir};},proto:{addBlock:function(d,e){return this._.panel.addBlock(d,e);},addListBlock:function(d,e){return this._.panel.addListBlock(d,e);},getBlock:function(d){return this._.panel.getBlock(d);},showBlock:function(d,e,f,g,h){var i=this._.panel,j=i.showBlock(d);this.allowBlur(false);b=true;var k=this.element,l=this._.iframe,m=this._.definition,n=e.getDocumentPosition(k.getDocument()),o=this._.dir=='rtl',p=n.x+(g||0),q=n.y+(h||0);if(o&&(f==1||f==4))p+=e.$.offsetWidth;else if(!o&&(f==2||f==3))p+=e.$.offsetWidth-1;if(f==3||f==4)q+=e.$.offsetHeight-1;this._.panel._.offsetParentId=e.getId();k.setStyles({top:q+'px',left:'-3000px',visibility:'hidden',opacity:'0',display:''});if(!this._.blurSet){var r=CKEDITOR.env.ie?l:new CKEDITOR.dom.window(l.$.contentWindow);CKEDITOR.event.useCapture=true;r.on('blur',function(s){var v=this;if(CKEDITOR.env.ie&&!v.allowBlur())return;var t=s.data.getTarget(),u=t.getWindow&&t.getWindow();if(u&&u.equals(r))return;if(v.visible&&!v._.activeChild&&!b)v.hide();},this);r.on('focus',function(){this._.focused=true;this.hideChild();this.allowBlur(true);},this);CKEDITOR.event.useCapture=false;this._.blurSet=1;}i.onEscape=CKEDITOR.tools.bind(function(){this.onEscape&&this.onEscape();},this);CKEDITOR.tools.setTimeout(function(){if(o)p-=k.$.offsetWidth;k.setStyles({left:p+'px',visibility:'',opacity:'1'});if(j.autoSize){function s(){var t=k.getFirst(),u=j.element.$.scrollHeight;if(CKEDITOR.env.ie&&CKEDITOR.env.quirks&&u>0)u+=(t.$.offsetHeight||0)-(t.$.clientHeight||0);t.setStyle('height',u+'px');i._.currentBlock.element.setStyle('display','none').removeStyle('display');};if(i.isLoaded)s();else i.onLoad=s;}else k.getFirst().removeStyle('height');CKEDITOR.tools.setTimeout(function(){if(m.voiceLabel)if(CKEDITOR.env.gecko){var t=l.getParent();
t.setAttribute('role','region');t.setAttribute('title',m.voiceLabel);l.setAttribute('role','region');l.setAttribute('title',' ');}if(CKEDITOR.env.ie&&CKEDITOR.env.quirks)l.focus();else l.$.contentWindow.focus();if(CKEDITOR.env.ie&&!CKEDITOR.env.quirks)this.allowBlur(true);},0,this);},0,this);this.visible=1;if(this.onShow)this.onShow.call(this);b=false;},hide:function(){var d=this;if(d.visible&&(!d.onHide||d.onHide.call(d)!==true)){d.hideChild();d.element.setStyle('display','none');d.visible=0;}},allowBlur:function(d){var e=this._.panel;if(d!=undefined)e.allowBlur=d;return e.allowBlur;},showAsChild:function(d,e,f,g,h,i){if(this._.activeChild==d&&d._.panel._.offsetParentId==f.getId())return;this.hideChild();d.onHide=CKEDITOR.tools.bind(function(){CKEDITOR.tools.setTimeout(function(){if(!this._.focused)this.hide();},0,this);},this);this._.activeChild=d;this._.focused=false;d.showBlock(e,f,g,h,i);if(CKEDITOR.env.ie7Compat||CKEDITOR.env.ie8&&CKEDITOR.env.ie6Compat)setTimeout(function(){d.element.getChild(0).$.style.cssText+='';},100);},hideChild:function(){var d=this._.activeChild;if(d){delete d.onHide;delete this._.activeChild;d.hide();}}}});})();



```
