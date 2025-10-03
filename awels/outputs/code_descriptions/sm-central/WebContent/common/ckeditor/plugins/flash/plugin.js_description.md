# plugin.js

## Review

## 1. Summary  

The file is a **CKEditor 3.x plug‑in** that adds support for embedding Flash objects.  
It registers the *flash* command, UI button, context‑menu item and dialog, and it uses the
`fakeobjects` plugin to represent the Flash element in the WYSIWYG editor as a placeholder image (`img.cke_flash`).  

Key components  

| Component | Role |
|-----------|------|
| **Regular expressions (`a`, `b`)** | Detect Flash source URLs and numeric values. |
| **Helper functions (`c`, `d`, `e`)** | 1) Append `px` to numeric dimensions, 2) Identify Flash `<object>`/`<embed>`, 3) Create the fake object placeholder. |
| **`CKEDITOR.plugins.add('flash')`** | Main plug‑in definition – `init` and `afterInit` hooks. |
| **Data‑filter rules** | Convert real Flash tags into the fake placeholder when content is loaded into the editor. |
| **`CKEDITOR.config` extension** | Provide plug‑in configuration flags (`flashEmbedTagOnly`, `flashAddEmbedTag`, `flashConvertOnEdit`). |

The plug‑in uses only the CKEditor core and the third‑party **`fakeobjects`** plug‑in, no external libraries.

---

## 2. Detailed Description  

### 2.1. High‑level Flow  

1. **Plugin registration** – the anonymous IIFE (`(function(){ … })()`) registers the *flash* plug‑in via `CKEDITOR.plugins.add`.  
2. **Initialization** (`init`)  
   * Adds the command (`flash`) that opens the dialog.  
   * Adds the toolbar button and optional menu item.  
   * Loads the dialog definition from `dialogs/flash.js`.  
   * Injects placeholder styling for `img.cke_flash`.  
   * Registers a context‑menu listener that only activates on the fake flash element.  
3. **After initialization** (`afterInit`)  
   * Adds data‑filter rules that catch `<object>` and `<embed>` tags that represent Flash and replace them with a fake image via `e()`.  
4. **User interaction** – the dialog collects properties (src, width, height, etc.) and inserts a `<embed>`/`<object>` element into the editor.  
   * When the content is later loaded (e.g., via `setData`), the data‑filter ensures that real Flash tags are rendered as the placeholder image.  

### 2.2. Key Design Choices  

| Choice | Rationale |
|--------|-----------|
| **`fakeobjects`** | Allows a complex element (Flash) to be represented by a simple `<img>` while still being editable. |
| **Regular expressions** | Quick detection of Flash URLs (`.swf`) and numeric values. |
| **`addRules`** | A lightweight approach to converting DOM elements during data filtering rather than a full XML parser. |
| **`createFakeParserElement`** | Provides a ready‑made helper for fake objects, reducing boilerplate. |
| **Config extension** | Gives developers fine‑grained control over how the plug‑in behaves in different scenarios (e.g., converting only on edit). |

---

## 3. Functions / Methods  

| Function | Purpose | Inputs | Output | Side‑effects |
|----------|---------|--------|--------|--------------|
| **`c(f)`** | Normalizes a dimension value. If `f` is numeric (`/^\d+(\.\d+)?$/`), returns `f+'px'`; otherwise returns `f` unchanged. | `f`: string | Dimension string | None |
| **`d(f)`** | Detects if element `f` is a Flash element. Returns `true` if `<embed type="application/x-shockwave-flash">` or src ends with `.swf`. | `f`: DOM element | Boolean | None |
| **`e(f,g)`** | Creates the fake object image used by the editor. <br>* `f`: CKEditor instance. <br>* `g`: Original `<object>` or `<embed>` element. | `f`, `g` | Fake `<img>` element | Adds width/height style attributes based on the original element. |
| **`CKEDITOR.plugins.add('flash', {...})`** | Main plug‑in definition. Handles registration, UI, data filtering, etc. | None | Plug‑in object | Registers commands, UI, dialog, CSS, data‑filter rules. |
| **`afterInit(f)`** (inner) | Hook executed after the editor’s data processor is ready. Adds data‑filter rules for `<object>`/`<embed>` tags. | `f`: CKEditor instance | None | Adds conversion rules to `dataFilter`. |

### Reusable/Utility Methods  

* `c()` – can be reused for other plugins that need dimension normalization.  
* `d()` – useful for generic detection of Flash in other plug‑ins.  
* `e()` – a wrapper around `createFakeParserElement` that could be extracted to a shared utility library.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` core | Standard | Provides the plugin API, dialog system, data filter, etc. |
| `CKEDITOR.plugins.fakeobjects` | Third‑party (built‑in CKEditor plug‑in) | Required for `createFakeParserElement`. |
| `CKEDITOR.getUrl` | Standard | Used to locate the placeholder image. |
| `CKEDITOR.dialog` | Standard | Loads the dialog from `dialogs/flash.js`. |

**No other external libraries** are used. The plug‑in is entirely self‑contained aside from CKEditor itself.

---

## 5. Additional Notes  

### 5.1. Edge Cases & Limitations  

| Issue | Impact | Mitigation |
|-------|--------|------------|
| **Flash is deprecated** | Modern browsers block Flash; embedding may not work for end users. | Provide an alternative plugin (e.g., HTML5 video) or fallback options. |
| **Numeric check (`b`)** | Only accepts integers or decimals; fails for dimensions specified in units other than `px` (e.g., `%`, `em`). | Extend regex or handle units explicitly. |
| **Attribute extraction** | `e()` assumes that width/height are direct attributes; ignores nested `<param>` tags that may specify size. | Parse `<param>` values or use the `width`/`height` from the `<object>` element’s attributes. |
| **Context menu activation** | Currently only checks `_cke_real_element_type=='flash'`. If the fake element is not set correctly, menu will not appear. | Ensure that `createFakeParserElement` consistently sets the required attribute. |
| **Config flags** | The meaning of `flashEmbedTagOnly` / `flashAddEmbedTag` / `flashConvertOnEdit` is not documented here; may be confusing. | Add inline comments or external documentation. |

### 5.2. Future Enhancements  

1. **Support for newer media** – replace or supplement Flash with HTML5 `<video>`/`<audio>` plug‑ins.  
2. **Better dimension handling** – accept any CSS unit and preserve the original value.  
3. **Internationalization** – ensure all strings (button label, menu labels) are retrieved from the language files.  
4. **Accessibility** – add `title`/`alt` attributes to the fake image for screen readers.  
5. **Unit tests** – write automated tests for `c()`, `d()`, and `e()` to catch regressions.  

### 5.3. Code Style & Maintainability  

* The code is heavily minified, making manual review harder.  
* Splitting the plug‑in into multiple files (`plugin.js`, `dialog.js`, `lang/*.js`) would improve readability.  
* Adding JSDoc comments to the helper functions would aid future contributors.  

---

**Overall**, the plug‑in is a compact, well‑structured CKEditor feature that follows standard practices for fake object handling. It serves its purpose but would benefit from modernization and documentation due to the deprecation of Flash technology.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){var a=/\.swf(?:$|\?)/i,b=/^\d+(?:\.\d+)?$/;function c(f){if(b.test(f))return f+'px';return f;};function d(f){var g=f.attributes;return g.type=='application/x-shockwave-flash'||a.test(g.src||'');};function e(f,g){var h=f.createFakeParserElement(g,'cke_flash','flash',true),i=h.attributes.style||'',j=g.attributes.width,k=g.attributes.height;if(typeof j!='undefined')i=h.attributes.style=i+'width:'+c(j)+';';if(typeof k!='undefined')i=h.attributes.style=i+'height:'+c(k)+';';return h;};CKEDITOR.plugins.add('flash',{init:function(f){f.addCommand('flash',new CKEDITOR.dialogCommand('flash'));f.ui.addButton('Flash',{label:f.lang.common.flash,command:'flash'});CKEDITOR.dialog.add('flash',this.path+'dialogs/flash.js');f.addCss('img.cke_flash{background-image: url('+CKEDITOR.getUrl(this.path+'images/placeholder.png')+');'+'background-position: center center;'+'background-repeat: no-repeat;'+'border: 1px solid #a9a9a9;'+'width: 80px;'+'height: 80px;'+'}');if(f.addMenuItems)f.addMenuItems({flash:{label:f.lang.flash.properties,command:'flash',group:'flash'}});if(f.contextMenu)f.contextMenu.addListener(function(g,h){if(g&&g.is('img')&&g.getAttribute('_cke_real_element_type')=='flash')return{flash:CKEDITOR.TRISTATE_OFF};});},afterInit:function(f){var g=f.dataProcessor,h=g&&g.dataFilter;if(h)h.addRules({elements:{'cke:object':function(i){var j=i.attributes,k=j.classid&&String(j.classid).toLowerCase();if(!k){for(var l=0;l<i.children.length;l++)if(i.children[l].name=='embed'){if(!d(i.children[l]))return null;return e(f,i);}return null;}return e(f,i);},'cke:embed':function(i){if(!d(i))return null;return e(f,i);}}},5);},requires:['fakeobjects']});})();CKEDITOR.tools.extend(CKEDITOR.config,{flashEmbedTagOnly:false,flashAddEmbedTag:true,flashConvertOnEdit:false});



```
