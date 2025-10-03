# radio.js

## Review

## 1. Summary

The snippet is a **CKEditor dialog definition** that allows users to insert or edit an `<input type="radio">` element.  
* **Purpose** – Provide a UI for specifying the name, value, and checked state of a radio button.  
* **Key components**  
  * `CKEDITOR.dialog.add('radio', ...)` – registers a dialog named *radio*.  
  * `onShow` – prepares the dialog for editing an existing radio button or creating a new one.  
  * `onOk` – finalizes changes and inserts the element into the editor.  
  * `contents` – defines the tabbed interface (only one tab – *info*) and its form fields.  
* **Notable patterns** – Uses the **factory** pattern (`CKEDITOR.dialog.add`) and **callback** functions for setup/commit. Relies heavily on CKEditor’s DOM API and language localisation system (`a.lang`).  

---

## 2. Detailed Description

### Core Flow

| Step | Description |
|------|-------------|
| **Dialog registration** | `CKEDITOR.dialog.add('radio', function(a){ ... });` registers a dialog factory. `a` is the `CKEDITOR` instance. |
| **onShow** | Triggered when the dialog is opened.  
  * Deletes any previous reference to `c.radioButton`.  
  * Checks the current selection in the editor. If the selected element is `<input type="radio">`, it stores it in `c.radioButton` and calls `setupContent(b)` to pre‑populate the fields. |
| **onOk** | Triggered when the user clicks *OK*.  
  * Determines whether an existing element is being edited (`c`) or a new one must be created (`b`).  
  * If creating new, a `<input type="radio">` element is instantiated and inserted.  
  * Calls `commitContent` to write field values back to the element. |
| **Content Definition** | One tab (*info*) contains three fields: *name*, *value*, and *checked*. Each field defines `setup` and `commit` callbacks to sync with the DOM element. |
| **Cleanup** | No explicit cleanup – the dialog is discarded by CKEditor once closed. |

### Assumptions & Constraints

* Relies on **CKEditor 3.x** API (e.g., `CKEDITOR.env.ie`, `CKEDITOR.dom.element`).  
* Works only with `<input>` elements; no handling of `<label>` or group containers.  
* Assumes the editor has a language definition for the `checkboxAndRadio` namespace.  
* IE-specific handling for the *checked* attribute due to historic browser bugs.

### Architecture

The dialog is a **self‑contained component** that integrates with the editor’s dialog system. It encapsulates its own data flow (setup → user interaction → commit) and leverages CKEditor’s internal utilities for DOM manipulation and localisation. The design follows CKEditor’s plugin model, making it easy to extend or replace the dialog.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `CKEDITOR.dialog.add('radio', function(a){ ... })` | Factory that registers the dialog. | `a`: `CKEDITOR` instance | Dialog definition registered; dialog appears when invoked. |
| `onShow` | Initialise dialog fields based on current selection. | `this` (dialog instance) | Sets `this.radioButton`, populates form fields via `setupContent`. |
| `onOk` | Commit user changes, insert or replace element. | `this` (dialog instance) | Creates/replaces element; inserts into editor; calls `commitContent`. |
| `contents` (array) | Defines the UI layout and fields. | – | – |
| Individual field `setup(b)` | Pre‑fill a field from the element. | `b`: element being edited | Calls `setValue` with attribute value. |
| Individual field `commit(b)` | Write field value back to the element. | `b`: element being edited | Sets/clears attributes on `b.element`. |
| `CKEDITOR.dom.element.createFromHtml` (used in IE branch) | Generates a new DOM element from markup. | HTML string, document | Returns a new element. |
| `c.copyAttributes(f, {type:1,checked:1})` | Copies non‑specified attributes to a new element. | Source element `c`, target element `f`, whitelist object | Copies attributes except those in whitelist. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party | Core CKEditor library. |
| `CKEDITOR.dialog` | CKEditor API | Dialog system. |
| `CKEDITOR.env` | CKEditor API | Environment detection (e.g., IE). |
| `CKEDITOR.dom.element` | CKEditor API | Abstracted DOM manipulation. |
| `a.lang` | CKEditor API | Internationalisation. |
| `document` | Browser API | Standard DOM. |

No external libraries beyond CKEditor itself are required. The code is **platform‑agnostic** except for the IE-specific branch.

---

## 5. Additional Notes

### Strengths

* **Clear separation** between UI definition (`contents`) and behaviour (`onShow`, `onOk`).  
* Utilises CKEditor’s localisation system (`a.lang`).  
* Handles the IE quirk for the `checked` attribute gracefully.  
* Re‑uses existing element when editing, avoiding unnecessary DOM churn.

### Potential Weaknesses & Edge Cases

1. **Non‑IE checked handling** – The code uses `c.setAttribute('checked','checked')` for non‑IE browsers. Modern browsers accept this, but some older ones might require boolean attributes.  
2. **Name Attribute Handling** – The `name` field stores the value in `_cke_saved_name` to preserve the original attribute if it was removed. However, the commit logic removes the attribute only if the field is empty; it never restores the original attribute when the user re‑enters a value.  
3. **Attribute Sanitisation** – No validation is performed on `name`, `value`, or `checked` inputs; potentially unsafe values could be inserted.  
4. **Group Context** – Radio buttons are typically grouped by `name`. The dialog does not provide a way to manage the group (e.g., ensure unique names, display other radios).  
5. **Accessibility** – The dialog does not expose ARIA attributes; however, CKEditor’s dialog system typically handles this.

### Suggested Enhancements

* **Validation** – Add simple validation (e.g., non‑empty name, valid characters) before commit.  
* **Group Management** – Provide a lookup of existing radio groups in the document and suggest a name.  
* **Modernisation** – Replace deprecated `CKEDITOR.dom.element.createFromHtml` with standard DOM `document.createElement`.  
* **Testing** – Include unit tests for `onShow`, `onOk`, and field commit logic, ensuring IE branch behaviour.  
* **Documentation** – Inline comments describing the purpose of each field and the rationale behind the IE workaround would aid maintainability.

Overall, the code is concise, follows CKEditor’s conventions, and achieves its intended purpose with minimal overhead. The main improvements revolve around robustness, validation, and future‑proofing.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('radio',function(a){return{title:a.lang.checkboxAndRadio.radioTitle,minWidth:350,minHeight:140,onShow:function(){var c=this;delete c.radioButton;var b=c.getParentEditor().getSelection().getSelectedElement();if(b&&b.getName()=='input'&&b.getAttribute('type')=='radio'){c.radioButton=b;c.setupContent(b);}},onOk:function(){var b,c=this.radioButton,d=!c;if(d){b=this.getParentEditor();c=b.document.createElement('input');c.setAttribute('type','radio');}if(d)b.insertElement(c);this.commitContent({element:c});},contents:[{id:'info',label:a.lang.checkboxAndRadio.radioTitle,title:a.lang.checkboxAndRadio.radioTitle,elements:[{id:'name',type:'text',label:a.lang.common.name,'default':'',accessKey:'N',setup:function(b){this.setValue(b.getAttribute('_cke_saved_name')||b.getAttribute('name')||'');},commit:function(b){var c=b.element;if(this.getValue())c.setAttribute('_cke_saved_name',this.getValue());else{c.removeAttribute('_cke_saved_name');c.removeAttribute('name');}}},{id:'value',type:'text',label:a.lang.checkboxAndRadio.value,'default':'',accessKey:'V',setup:function(b){this.setValue(b.getAttribute('value')||'');},commit:function(b){var c=b.element;if(this.getValue())c.setAttribute('value',this.getValue());else c.removeAttribute('value');}},{id:'checked',type:'checkbox',label:a.lang.checkboxAndRadio.selected,'default':'',accessKey:'S',value:'checked',setup:function(b){this.setValue(b.getAttribute('checked'));},commit:function(b){var c=b.element;if(!CKEDITOR.env.ie){if(this.getValue())c.setAttribute('checked','checked');else c.removeAttribute('checked');}else{var d=c.getAttribute('checked'),e=!!this.getValue();if(d!=e){var f=CKEDITOR.dom.element.createFromHtml('<input type="radio"'+(e?' checked="checked"':'')+'></input>',a.document);c.copyAttributes(f,{type:1,checked:1});f.replace(c);a.getSelection().selectElement(f);b.element=f;}}}}]}]};});



```
