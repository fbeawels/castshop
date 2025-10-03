# plugin.js

## Review

## 1. Summary

This snippet implements the **`font`** plugin for CKEditor (v3.x).  
It adds two dropdowns to the toolbar:

1. **Font Family** – lets the user pick a typeface from a predefined list.  
2. **Font Size** – lets the user pick a point size from a predefined list.

Both dropdowns are built using CKEditor’s `RichCombo` UI widget and rely on the `styles` plugin to apply or remove a `span` element with the corresponding CSS (`font-family` or `font-size`). The plugin also configures default values (`config.font_names`, `config.fontSize_sizes`, etc.) and exposes a `font_style` and `fontSize_style` configuration object that can be overridden by the user.

Design patterns:
- **Factory** – `a()` creates and registers a RichCombo for a given style type.
- **Event‑Driven** – the UI listens to `selectionChange` to sync its state with the current cursor position.

## 2. Detailed Description

### Core Flow

1. **Initialization (`CKEDITOR.plugins.add`)**  
   The plugin registers two RichCombo UI elements: one for font families and one for sizes. The `init` function of the plugin calls `a()` twice, passing in the configuration data (labels, options, default values, and style definitions).

2. **`a()` – Combo Creation**  
   - Parses the `config.font_names` or `config.fontSize_sizes` string (semicolon‑separated) into an array of options.  
   - For each option, it creates a `CKEDITOR.style` instance (`l[p] = new CKEDITOR.style(h, n)`) and stores it in a map `l`.  
   - It registers a RichCombo (`b.ui.addRichCombo`) with the label, panel title, and CSS from the editor’s skin and contents.  
   - The `init` method populates the combo with items, showing a preview `<span>` that uses the same style that will be applied.  
   - `onClick` handles the user’s choice: it toggles the style on the current selection.  
   - `onRender` hooks into the editor’s `selectionChange` event to keep the combo’s value in sync with the active style.

3. **Style Application**  
   - `l[q]` holds the `CKEDITOR.style` for the selected value.  
   - If the combo already shows that value, `remove()` is called; otherwise `apply()` is invoked.  
   - Snapshot events (`saveSnapshot`) are fired before and after changes to support undo/redo.

4. **Cleanup**  
   No explicit cleanup is needed because CKEditor automatically disposes of UI components when the editor instance is destroyed.

### Assumptions & Constraints

- **String‑based Configuration** – The plugin expects `font_names` and `fontSize_sizes` to be a `;`‑separated string where each entry may optionally include an alias before `/` (e.g., `"Arial/Arial, Helvetica"`).  
- **`CKEDITOR.style`** – Assumes that `CKEDITOR.styles` plugin is available and that the editor can apply/strip inline styles via `span`.  
- **Browser Compatibility** – Relies on CKEditor’s abstraction; no raw DOM hacks that would break older browsers.  
- **No External Dependencies** – Entirely within CKEditor’s own libraries.

## 3. Functions / Methods

| Function / Method | Purpose | Parameters | Returns | Side Effects |
|-------------------|---------|------------|---------|--------------|
| `a(b, c, d, e, f, g, h)` | Factory for creating a RichCombo for a style type (font or size). | `b`: CKEditor instance<br> `c`: combo name (e.g., `"Font"`) <br> `d`: style key (`"family"` or `"size"`) <br> `e`: language object for labels <br> `f`: semicolon‑separated options string <br> `g`: default label string <br> `h`: style definition object | `undefined` | Adds UI component to editor, sets up event handlers |
| `b.ui.addRichCombo` | CKEditor API – registers a RichCombo widget. | `name`, `configObject` | `undefined` | Creates UI element, registers event callbacks |
| `CKEDITOR.style` | Wraps a CSS style for easy application/removal. | `definition` (element + styles + overrides) | `CKEDITOR.style` instance | N/A |
| `init` (inside RichCombo config) | Populates the combo list. | N/A | `undefined` | Adds items with preview spans |
| `onClick` (inside RichCombo config) | Handles user selection. | `q`: selected value | `undefined` | Applies or removes style, fires snapshots |
| `onRender` (inside RichCombo config) | Syncs combo value with editor selection. | N/A | `undefined` | Registers `selectionChange` listener |
| `b.on('selectionChange', ...)` | Event listener to keep UI state current. | `q`: event data | `undefined` | Calls `setValue` on combo |
| `b.fire('saveSnapshot')` | Creates undo snapshot. | N/A | `undefined` | Adds undo state |

### Reusable / Utility Methods

- **`a()`** – While tightly coupled to the `font` plugin, its logic could be extracted to a generic helper for any style‑based RichCombo.
- **`CKEDITOR.style`** – A CKEditor built‑in utility; reusable across plugins for any inline style.

## 4. Dependencies

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| **CKEditor (v3.x)** | Core | Provides `CKEDITOR.plugins`, `CKEDITOR.ui`, `CKEDITOR.style`, event system, and configuration. |
| **`richcombo` plugin** | CKEditor plugin | Required for the dropdown UI. |
| **`styles` plugin** | CKEditor plugin | Required for `CKEDITOR.style`. |
| **`CKEDITOR.getUrl`** | CKEditor helper | Generates URLs for skin assets. |
| **No external JS libraries** | – | The code is fully contained within CKEditor’s ecosystem. |

No platform‑specific APIs are used; the plugin works in any browser supported by CKEditor 3.x.

## 5. Additional Notes

### Strengths

- **Minimal footprint** – Leverages CKEditor’s built‑in widgets and styling system, avoiding duplication of UI logic.
- **Extensible configuration** – Users can override `font_names`, `fontSize_sizes`, default labels, and style definitions in their `config.js`.
- **Responsive UI** – The combo shows a preview of each font/size, improving usability.

### Edge Cases & Limitations

- **Malformed option strings** – If `font_names` or `fontSize_sizes` contain unexpected characters or missing slashes, the parser may fail silently or create invalid styles.
- **Non‑span styles** – The plugin is hard‑coded to apply styles to `span` elements. If a user wants to apply fonts to other elements (e.g., `p`, `div`), the plugin would need modification.
- **Overlap with existing `font` tags** – The `overrides` section removes `<font>` tags, but if the content contains nested `<font>` tags or other inline formatting, the logic may not remove them cleanly.
- **Accessibility** – The combo relies on `voiceLabel` and `panelVoiceLabel` for screen readers, but the plugin does not expose a way to localize these beyond the provided language objects.

### Potential Enhancements

1. **Array‑Based Configuration** – Accept an array of objects for `font_names` and `fontSize_sizes` instead of a semicolon string; this would simplify parsing and allow richer metadata (e.g., font weight, preview image).
2. **Multi‑Element Support** – Extend the style definition to allow applying fonts to block elements when appropriate.
3. **Improved State Synchronization** – Cache the last applied style to reduce the work done on each `selectionChange`.
4. **Dynamic Loading of Font Files** – Provide a mechanism to load custom web fonts (e.g., via @font-face) and expose them in the dropdown.
5. **Unit Tests** – Add automated tests (e.g., with QUnit) to verify that styles are applied/removed correctly under various editor states.

Overall, the code is concise, follows CKEditor conventions, and provides a useful UI for font selection. It could benefit from minor refactoring for configurability and robustness, but it serves its purpose well within the CKEditor ecosystem.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){function a(b,c,d,e,f,g,h){var i=b.config,j=f.split(';'),k=[],l={};for(var m=0;m<j.length;m++){var n={},o=j[m].split('/'),p=j[m]=o[0];n[d]=k[m]=o[1]||p;l[p]=new CKEDITOR.style(h,n);}b.ui.addRichCombo(c,{label:e.label,title:e.panelTitle,voiceLabel:e.voiceLabel,className:'cke_'+(d=='size'?'fontSize':'font'),multiSelect:false,panel:{css:[CKEDITOR.getUrl(b.skinPath+'editor.css')].concat(i.contentsCss),voiceLabel:e.panelVoiceLabel},init:function(){this.startGroup(e.panelTitle);for(var q=0;q<j.length;q++){var r=j[q];this.add(r,'<span style="font-'+d+':'+k[q]+'">'+r+'</span>',r);}},onClick:function(q){b.focus();b.fire('saveSnapshot');var r=l[q];if(this.getValue()==q)r.remove(b.document);else r.apply(b.document);b.fire('saveSnapshot');},onRender:function(){b.on('selectionChange',function(q){var r=this.getValue(),s=q.data.path,t=s.elements;for(var u=0,v;u<t.length;u++){v=t[u];for(var w in l)if(l[w].checkElementRemovable(v,true)){if(w!=r)this.setValue(w);return;}}this.setValue('',g);},this);}});};CKEDITOR.plugins.add('font',{requires:['richcombo','styles'],init:function(b){var c=b.config;a(b,'Font','family',b.lang.font,c.font_names,c.font_defaultLabel,c.font_style);a(b,'FontSize','size',b.lang.fontSize,c.fontSize_sizes,c.fontSize_defaultLabel,c.fontSize_style);}});})();CKEDITOR.config.font_names='Arial/Arial, Helvetica, sans-serif;Comic Sans MS/Comic Sans MS, cursive;Courier New/Courier New, Courier, monospace;Georgia/Georgia, serif;Lucida Sans Unicode/Lucida Sans Unicode, Lucida Grande, sans-serif;Tahoma/Tahoma, Geneva, sans-serif;Times New Roman/Times New Roman, Times, serif;Trebuchet MS/Trebuchet MS, Helvetica, sans-serif;Verdana/Verdana, Geneva, sans-serif';CKEDITOR.config.font_defaultLabel='';CKEDITOR.config.font_style={element:'span',styles:{'font-family':'#(family)'},overrides:[{element:'font',attributes:{face:null}}]};CKEDITOR.config.fontSize_sizes='8/8px;9/9px;10/10px;11/11px;12/12px;14/14px;16/16px;18/18px;20/20px;22/22px;24/24px;26/26px;28/28px;36/36px;48/48px;72/72px';CKEDITOR.config.fontSize_defaultLabel='';CKEDITOR.config.fontSize_style={element:'span',styles:{'font-size':'#(size)'},overrides:[{element:'font',attributes:{size:null}}]};



```
