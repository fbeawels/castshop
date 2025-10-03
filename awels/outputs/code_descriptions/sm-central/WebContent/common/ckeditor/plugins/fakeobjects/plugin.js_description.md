# plugin.js

## Review

## 1. Summary  

* **Purpose** – The snippet is a CKEditor 4 plugin called **`fakeobjects`**.  
  It lets the editor represent non‑visual or “real” elements (e.g. embedded media, widgets, widgets that are not supported natively) as lightweight image placeholders while the document is edited.  
  The real element is stored in a custom attribute (`_cke_realelement`) so that the original markup can be restored when the editor content is saved or when the placeholder is replaced.

* **Key Components**  
  * **`CKEDITOR.plugins.add('fakeobjects')`** – registers the plugin, declares a dependency on `htmlwriter`, and injects custom filtering rules into the editor’s `dataProcessor.htmlFilter`.  
  * **Custom filter rule** – uses a `$` rule to intercept `<img>` tags that have a `_cke_realelement` attribute, parse the original HTML fragment and, when possible, pull out width/height from the `style` attribute.  
  * **Three prototype helpers** –  
    * `createFakeElement` – builds a DOM `<img>` element from an existing CKEditor element.  
    * `createFakeParserElement` – builds a *parser* representation (for the data‑processor) from an element.  
    * `restoreRealElement` – turns a fake `<img>` back into the original CKEditor element.

* **Notable Patterns / Libraries**  
  * **Encapsulation via `CKEDITOR.plugins.add`** – standard CKEditor plugin architecture.  
  * **Filter rule as a “rule object”** – follows CKEditor’s HTML filtering conventions.  
  * **Custom attributes** – uses non‑standard attributes prefixed with `_cke_` to store metadata.  
  * **HTML encoding/decoding** – relies on `encodeURIComponent`/`decodeURIComponent` to serialize/deserialize the original markup.

---

## 2. Detailed Description  

### Core Flow  

| Phase | What Happens | Where |
|-------|--------------|-------|
| **Initialization** | The plugin is added, declaring a requirement for `htmlwriter`. In `afterInit`, it grabs the editor’s `dataProcessor.htmlFilter` (if available) and injects the custom rule that recognises fake objects. | `CKEDITOR.plugins.add('fakeobjects', …)` |
| **Creating a Fake Element** | When an element that cannot be rendered natively is to be inserted into the editor, `createFakeElement` is called. It: 1) builds a JSON attribute bag, 2) stores the outer HTML of the original element under `_cke_realelement`, 3) adds optional `class`, `src` (spacer.gif), `alt`, `_cke_real_element_type`, `_cke_resizable`. | `editor.prototype.createFakeElement` |
| **Parser‑side Fake** | `createFakeParserElement` is similar but creates a *parser* element (used by the data‑processor). It writes the element’s HTML to a writer, then uses the output as the `_cke_realelement` value. | `editor.prototype.createFakeParserElement` |
| **Restoring** | When the editor needs to output the real HTML (e.g. during `save` or `toDataFormat`), `restoreRealElement` decodes the `_cke_realelement` attribute and rebuilds the original element via `CKEDITOR.dom.element.createFromHtml`. | `editor.prototype.restoreRealElement` |
| **Filter Rule** | The custom `$` rule is applied when the editor parses HTML coming from the server or user. It matches `<img>` elements containing `_cke_realelement`. If a `style` attribute has numeric `width`/`height`, those are extracted and applied to the decoded real element’s attributes. The parser then replaces the fake image with the real element. | `afterInit` → `dataProcessor.htmlFilter` |

### Design Choices  

* **Use of a tiny image placeholder** – Keeps the editor’s UI lightweight and ensures the placeholder behaves like an inline image.  
* **Attribute‑based storage** – No external storage; the real markup is embedded directly in the fake element.  
* **Encoding** – `encodeURIComponent` guarantees that the stored string is a valid attribute value, though it can become large for complex elements.  
* **Resizability flag** – `_cke_resizable` lets the editor treat the placeholder as a resizable object, which is useful for widgets that can change size.  
* **Filter integration** – By hooking into the existing `htmlFilter`, the plugin works seamlessly with other filters and data processors.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return Value | Side‑Effects |
|--------|---------|------------|--------------|--------------|
| `CKEDITOR.plugins.add('fakeobjects', {...})` | Registers the plugin and injects the filter rule after the editor is initialized. | *none* | *none* | Adds rule to the editor’s dataProcessor.htmlFilter. |
| `createFakeElement(element, className, type, resizable)` | Builds a DOM `<img>` fake element representing *element*. | `element` – CKEditor element to fake. <br> `className` – CSS class for styling the placeholder. <br> `type` – optional string indicating the real element type. <br> `resizable` – boolean flag that marks the placeholder as resizable. | `<img>` CKEditor DOM element | None. |
| `createFakeParserElement(element, className, type, resizable)` | Same as above, but returns a parser element (for dataProcessor). | Same as above. | `CKEDITOR.htmlParser.element` (an `<img>` node). | None. |
| `restoreRealElement(fakeElement)` | Decodes the `_cke_realelement` attribute and creates the original CKEditor element. | `fakeElement` – the fake `<img>` element to restore. | `CKEDITOR.dom.element` – the real element. | None. |
| **Filter rule (`$` function)** – internal, called during parsing. | Detects `<img>` tags with `_cke_realelement`, parses the real HTML, copies width/height from inline `style` if present, and returns the real element to replace the fake. | `img` – the `<img>` CKEditor element being parsed. | `realElement` – the decoded element to inject into the DOM. | None. |

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `CKEDITOR` (core) | Standard | Provides the editor instance, element API, `getUrl`, and utilities. |
| `htmlwriter` | Required plugin | Needed for the filter rule (uses `CKEDITOR.htmlParser.basicWriter`). |
| `dataProcessor` | Editor internal | Source of `htmlFilter` into which the rule is injected. |
| `htmlParser` | CKEditor API | Parser elements and writer used in the rule and `createFakeParserElement`. |

All dependencies are **CKEditor 4** internal or standard plugin modules; no third‑party libraries are referenced.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Encoding Size** – `encodeURIComponent` can produce a very long string for complex elements (tables, widgets with nested markup). This bloats the placeholder attribute and may hit browser limits for attribute lengths.  
2. **Security** – The decoded markup is inserted directly into the editor DOM without sanitisation. If the source data is user‑controlled, this could be a vector for XSS unless the editor’s built‑in sanitisation is active.  
3. **Style Parsing** – The rule only pulls numeric `width`/`height` values from `style`. It ignores percent values, ems, or other CSS units, and ignores any other inline styles that might be needed.  
4. **Resizability** – The `_cke_resizable` flag is simply stored; actual resizing behaviour is handled elsewhere in CKEditor. If that code changes, this plugin may need updates.  
5. **Browser Support** – The use of `decodeURIComponent`/`encodeURIComponent` is well‑supported, but the code assumes that the original HTML can be safely parsed back into a CKEditor element; malformed HTML could cause `createFromHtml` to fail.

### Possible Enhancements  

* **Chunked Storage** – For large elements, store a reference (e.g., an ID) instead of raw markup, and retrieve the real element from a hidden element store or the server.  
* **Robust CSS Extraction** – Extend the style parsing to support more units and copy other relevant inline styles (e.g., `display`, `float`).  
* **Sanitisation Hook** – Allow an optional sanitiser callback that cleans the decoded HTML before it is re‑inserted.  
* **Explicit Resizable UI** – Provide configuration for how resizable placeholders should be displayed and resized.  
* **Debugging Helpers** – Expose methods to list all fake objects in a document or to replace them programmatically.

---

### Bottom Line  

The `fakeobjects` plugin is a compact, well‑integrated solution for representing non‑visual elements inside CKEditor. It leverages the editor’s filtering system to swap placeholders for real markup transparently, and offers a clean API for creating and restoring these elements. While functional, developers should be aware of the encoding size, security, and limited style handling, and may want to augment the plugin for more robust use cases.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){var a={elements:{$:function(b){var c=b.attributes._cke_realelement,d=c&&new CKEDITOR.htmlParser.fragment.fromHtml(decodeURIComponent(c)),e=d&&d.children[0];if(e){var f=b.attributes.style;if(f){var g=/(?:^|\s)width\s*:\s*(\d+)/.exec(f),h=g&&g[1];g=/(?:^|\s)height\s*:\s*(\d+)/.exec(f);var i=g&&g[1];if(h)e.attributes.width=h;if(i)e.attributes.height=i;}}return e;}}};CKEDITOR.plugins.add('fakeobjects',{requires:['htmlwriter'],afterInit:function(b){var c=b.dataProcessor,d=c&&c.htmlFilter;if(d)d.addRules(a);}});})();CKEDITOR.editor.prototype.createFakeElement=function(a,b,c,d){var e=this.lang.fakeobjects,f={'class':b,src:CKEDITOR.getUrl('images/spacer.gif'),_cke_realelement:encodeURIComponent(a.getOuterHtml()),alt:e[c]||e.unknown};if(c)f._cke_real_element_type=c;if(d)f._cke_resizable=d;return this.document.createElement('img',{attributes:f});};CKEDITOR.editor.prototype.createFakeParserElement=function(a,b,c,d){var e=new CKEDITOR.htmlParser.basicWriter();a.writeHtml(e);var f=e.getHtml(),g=this.lang.fakeobjects,h={'class':b,src:CKEDITOR.getUrl('images/spacer.gif'),_cke_realelement:encodeURIComponent(f),alt:g[c]||g.unknown};if(c)h._cke_real_element_type=c;if(d)h._cke_resizable=d;return new CKEDITOR.htmlParser.element('img',h);};CKEDITOR.editor.prototype.restoreRealElement=function(a){var b=decodeURIComponent(a.getAttribute('_cke_realelement'));return CKEDITOR.dom.element.createFromHtml(b,this.document);};



```
