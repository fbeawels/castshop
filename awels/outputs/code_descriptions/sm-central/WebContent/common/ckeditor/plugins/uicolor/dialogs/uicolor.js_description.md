# uicolor.js

## Review

## 1. Summary

The file implements a **CKEditor dialog plugin** called **`uicolor`**.  
It provides a color picker UI that lets users pick or enter a UI color for the editor.  
Key components:

| Component | Role |
|-----------|------|
| **CKEDITOR.dialog.add** | Registers the dialog with the CKEditor core. |
| **YUI ColorPicker** | The heavy‑weight color picker widget rendered inside the dialog. |
| **Dialog elements** (`tab1`, `predefined`, `livePeview`, `configBox`) | UI controls for selecting, previewing, and applying colors. |

Design-wise, the code follows CKEditor's plugin architecture, using the dialog factory pattern. It mixes inline JavaScript with hard‑coded HTML strings and a large block of procedural code inside the dialog definition.

---

## 2. Detailed Description

### Initialization

1. **`CKEDITOR.dialog.add('uicolor', function(a){ … });`**  
   Registers the dialog with the name `uicolor`. The function receives the **dialog definition** (`a`) as argument.

2. Inside the factory a number of local variables are declared:
   - `b`: the dialog instance (assigned later in `onLoad`).
   - `c`: the YUI ColorPicker instance.
   - `d`: the definition for the HTML element that will host the YUI picker.
   - `e`: current UI color obtained from the editor (`a.getUiColor()`).

3. Two helper functions are defined:
   - **`f(i)`** – loads a color string into the picker and refreshes the widget. It handles both hex (`#RRGGBB`) and RGB objects.
   - **`g(i, j)`** – writes the selected color back to the editor and updates the preview/preview‑box.

4. **`d`** holds a minimal `html` dialog element that contains the container for the YUI picker. The `onLoad` callback creates the picker instance and wires the `rgbChange` event.

5. The dialog definition (`return { … }`) specifies:
   - Basic properties (`title`, `minWidth`, `minHeight`).
   - An `onLoad` hook that stores the dialog instance in `b` and calls `setupContent()`.
   - `contents`: a single tab (`tab1`) containing the YUI host, a checkbox for live preview, a predefined colour selector, a preview box, and a configuration string.

6. The dialog ends with the OK button that will close the dialog and apply changes (the actual commit logic is handled by `g` in response to events).

### Runtime Behaviour

| Event | Effect |
|-------|--------|
| **Dialog load** | YUI picker is instantiated, optionally pre‑set with the current UI color. |
| **Picker `rgbChange`** | Clears the predefined selector and updates the preview. |
| **Checkbox `livePeview`** | If live preview is on, the editor’s UI color is updated instantly. |
| **Predefined `select` change** | Sets the picker and preview to the selected colour. |
| **`configBox` show** | Displays the config string that the user can copy or edit. |

Cleanup is implicit – the dialog instance is discarded when closed. The YUI widget will be garbage‑collected because it has no external references after dialog removal.

### Assumptions & Constraints

* The code assumes **CKEditor 3.x** (use of `CKEDITOR.env.ie7Compat`, the `add` API, etc.).  
* It relies on **YUI 2** (`window.YAHOO.util.Color`, `window.YAHOO.widget.ColorPicker`) being available.  
* The plugin expects the editor to expose `getUiColor()` and `setUiColor()` methods.  
* All UI strings are fetched via `a.lang.uicolor.*`, requiring localisation files to be loaded.

### Architecture

The plugin follows the classic *dialog factory* pattern used by CKEditor: a closure that returns a dialog definition. UI components are mixed with plain HTML for the preview. The use of YUI as a third‑party widget demonstrates a common approach in CKEditor 3 where external UI libraries were often embedded.

---

## 3. Functions / Methods

| Function | Parameters | Purpose | Side‑Effects |
|----------|------------|---------|--------------|
| **`f(i)`** | `i` (color string or RGB object) | Normalises a colour value, sets it in the YUI picker, and refreshes the UI. | Modifies picker `c` state. |
| **`g(i, j)`** | `i` (colour string, hex without `#`), `j` (boolean flag) | Writes the colour to the editor (`setUiColor`) and updates the config preview. | Calls `a.setUiColor`, updates DOM (`configBox`). |
| **`d.onLoad(i)`** | `i` (unused) | Instantiates the YUI picker inside the dialog container. | Creates `c`; attaches `rgbChange` event. |
| **`onChange` handlers** in dialog elements | – | Respond to user interactions (preview toggle, predefined selector change). | Update picker, editor, preview. |

**Reusable utilities**

* `f` and `g` are generic enough to be reused for other colour‑picker dialogs.  
* The YUI colour conversion (`hex2rgb`) is invoked only inside `f`, encapsulating conversion logic.

---

## 4. Dependencies

| Dependency | Type | Notes |
|-------------|------|-------|
| **CKEditor core** | Third‑party (but mandatory) | Provides `CKEDITOR.dialog`, `CKEDITOR.env`, DOM utilities. |
| **YUI 2** | Third‑party | Required for `YAHOO.widget.ColorPicker`, `YAHOO.util.Color`. |
| **YUI assets** (`picker_thumb.png`, `hue_thumb.png`) | Static files | Path derived via `CKEDITOR.getUrl('plugins/uicolor/yui/')`. |
| **Localization files** (`uicolor.lang.*`) | Third‑party | Must exist in the plugin’s `lang/` folder. |

No platform‑specific code beyond IE7 compatibility checks is present.

---

## 5. Additional Notes

### Strengths

* **Modular** – All dialog logic is encapsulated inside a single closure.  
* **Internationalisation** – Uses `a.lang.*` for all visible strings.  
* **Live preview** – Gives users instant visual feedback.

### Potential Issues & Edge Cases

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded YUI assets** | If the plugin is moved or the YUI version changes, asset URLs break. | Use `CKEDITOR.getUrl` consistently or expose a configuration parameter. |
| **Missing cleanup** | The YUI picker is never destroyed; memory leaks may occur if dialogs are opened/closed repeatedly. | Call `c.destroy()` in the dialog's `onHide` or `onCancel` callbacks. |
| **`b._.contents.tab1` usage** | Directly accessing internal dialog structure is fragile; it relies on internal API that may change. | Use dialog API (`b.getContentElement('tab1', 'livePeview').getValue()`). |
| **`g` overwrites the editor’s UI color even when preview is off** | Might be confusing; the editor is updated but not shown. | Only call `setUiColor` when preview is active or when OK is pressed. |
| **No validation of colour input** | If a user enters an invalid hex string in `predefined`’s preview or `configBox`, the picker may error. | Add a regex check before applying the colour. |
| **IE7 compatibility code is obsolete** | In modern CKEditor 4+ contexts this check is unnecessary. | Remove or guard with a version check. |
| **Global `CKEDITOR` usage** | Could clash with other scripts if `CKEDITOR` is re‑loaded. | Use module imports or encapsulate in a namespace. |

### Future Enhancements

1. **CKEditor 4+ migration** – Re‑implement using the newer dialog API (YUI is no longer bundled).  
2. **Enhanced colour validation** – Provide real‑time feedback for manually entered hex values.  
3. **Persisting selections** – Store last selected colour in `localStorage` for quicker access.  
4. **Accessibility** – Add ARIA attributes and keyboard navigation for the colour picker.  
5. **Unit tests** – Use CKEditor’s test harness or Jest to validate dialog behaviour.

--- 

**Verdict:**  
The plugin achieves its purpose in the context of CKEditor 3.x and YUI 2. It is functional but tightly coupled to specific internal APIs and asset paths. Refactoring to decouple from YUI, adding cleanup, and modernizing the dialog API would greatly improve maintainability and future‑proofing.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('uicolor',function(a){var b,c,d,e=a.getUiColor();function f(i){if(/^#/.test(i))i=window.YAHOO.util.Color.hex2rgb(i.substr(1));c.setValue(i,true);c.refresh('cke_uicolor_picker');};function g(i,j){if(j||b._.contents.tab1.livePeview.getValue())a.setUiColor(i);b._.contents.tab1.configBox.setValue('config.uiColor = "#'+c.get('hex')+'"');};d={id:'yuiColorPicker',type:'html',html:"<div id='cke_uicolor_picker' style='width: 360px; height: 200px; position: relative;'></div>",onLoad:function(i){var j=CKEDITOR.getUrl('plugins/uicolor/yui/');c=new window.YAHOO.widget.ColorPicker('cke_uicolor_picker',{showhsvcontrols:true,showhexcontrols:true,images:{PICKER_THUMB:j+'assets/picker_thumb.png',HUE_THUMB:j+'assets/hue_thumb.png'}});if(e)f(e);c.on('rgbChange',function(){b._.contents.tab1.predefined.setValue('');g('#'+c.get('hex'));});var k=new CKEDITOR.dom.nodeList(c.getElementsByTagName('input'));for(var l=0;l<k.count();l++)k.getItem(l).addClass('cke_dialog_ui_input_text');}};var h=true;return{title:a.lang.uicolor.title,minWidth:360,minHeight:320,onLoad:function(){b=this;this.setupContent();if(CKEDITOR.env.ie7Compat)b.parts.contents.setStyle('overflow','hidden');},contents:[{id:'tab1',label:'',title:'',expand:true,padding:0,elements:[d,{id:'tab1',type:'vbox',children:[{id:'livePeview',type:'checkbox',label:a.lang.uicolor.preview,'default':1,onLoad:function(){h=true;},onChange:function(){if(h)return;var i=this.getValue(),j=i?'#'+c.get('hex'):e;g(j,true);}},{type:'hbox',children:[{id:'predefined',type:'select','default':'',label:a.lang.uicolor.predefined,items:[[''],['Light blue','#9AB8F3'],['Sand','#D2B48C'],['Metallic','#949AAA'],['Purple','#C2A3C7'],['Olive','#A2C980'],['Happy green','#9BD446'],['Jezebel Blue','#14B8C4'],['Burn','#FF893A'],['Easy red','#FF6969'],['Pisces 3','#48B4F2'],['Aquarius 5','#487ED4'],['Absinthe','#A8CF76'],['Scrambled Egg','#C7A622'],['Hello monday','#8E8D80'],['Lovely sunshine','#F1E8B1'],['Recycled air','#B3C593'],['Down','#BCBCA4'],['Mark Twain','#CFE91D'],['Specks of dust','#D1B596'],['Lollipop','#F6CE23']],onChange:function(){var i=this.getValue();if(i){f(i);g(i);CKEDITOR.document.getById('predefinedPreview').setStyle('background',i);}else CKEDITOR.document.getById('predefinedPreview').setStyle('background','');},onShow:function(){var i=a.getUiColor();if(i)this.setValue(i);}},{id:'predefinedPreview',type:'html',html:'<div id="cke_uicolor_preview" style="border: 1px solid black; padding: 3px; width: 30px;"><div id="predefinedPreview" style="width: 30px; height: 30px;">&nbsp;</div></div>'}]},{id:'configBox',type:'text',label:a.lang.uicolor.config,onShow:function(){var i=a.getUiColor();
if(i)this.setValue('config.uiColor = "'+i+'"');}}]}]}],buttons:[CKEDITOR.dialog.okButton]};});



```
