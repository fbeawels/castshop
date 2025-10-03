# livevalidation_standalone.compressed.js

## Review

## 1. Summary  

**Purpose**  
The script implements a lightweight client‑side validation library (LiveValidation 1.3 “stand‑alone”).  
It attaches validation rules to individual form fields, displays inline messages, and can coordinate a whole form’s validation on submit.

**Key components**

| Component | Role |
|-----------|------|
| `LiveValidation` | Per‑field validator. Handles DOM binding, rule storage, message rendering, and optional delayed validation. |
| `LiveValidationForm` | Keeps track of all fields belonging to a form, hooks into the form’s `onsubmit` to run `LiveValidation.massValidate`. |
| `Validate` | Namespace that contains the actual validation logic (Presence, Numericality, etc.). Provides `Validate.fail` for error propagation. |
| `LiveValidation.massValidate` | Static helper that iterates over a list of `LiveValidation` instances and aggregates the result. |

**Notable patterns / libraries**

* **Module pattern** – the code is wrapped in an IIFE‑style (not strictly an IIFE, but the functions are namespaced).  
* **Event wrapper** – old handlers are stored and called to avoid breaking user code.  
* **Singleton pattern** – `LiveValidationForm.getInstance` guarantees one form controller per form element.  
* **Error object** – `Validate.Error` is used to differentiate validation errors from other exceptions.  

No external libraries are required; all logic is vanilla JavaScript, compatible with modern browsers and older IE (pre‑9) thanks to its legacy syntax.

---

## 2. Detailed Description  

### 2.1 Initialization  

1. **Field level** – `new LiveValidation(element, options)`  
   * Resolves `element` either from a DOM node or an id.  
   * Detects element type (text, textarea, checkbox, etc.) via `getElementType`.  
   * Stores callbacks (`onValid`, `onInvalid`) and configuration flags (`onlyOnBlur`, `wait`, `onlyOnSubmit`).  
   * Binds event listeners for focus, blur, change, keyup, click (as appropriate).  
   * If part of a form, obtains or creates a `LiveValidationForm` instance and registers itself.

2. **Form level** – `LiveValidationForm.getInstance(form)`  
   * Creates a controller per form (or retrieves existing).  
   * Stores the original `onsubmit`.  
   * Overrides `onsubmit` to run `massValidate` on all registered fields and, only if valid, calls the original handler.

### 2.2 Runtime Flow  

| Step | What happens |
|------|--------------|
| **Typing / Interaction** | For most fields, keyup triggers `deferValidation()` (unless `onlyOnSubmit`). If `wait` is 0, validation runs immediately; otherwise it’s debounced via `setTimeout`. |
| **Blur** | Calls `doOnBlur` → `validate()` (unless `onlyOnSubmit`). |
| **Click / Change** | For checkboxes/select/file, `onclick`/`onchange` runs `validate()` immediately. |
| **Form submit** | `LiveValidationForm` calls `massValidate()`; if all fields pass, original `onsubmit` proceeds. |

`validate()` performs:

1. Skips if the element is disabled.  
2. Runs `doValidations()` which iterates over each rule in `this.validations`.  
3. For each rule it calls the appropriate `Validate` function inside a try/catch that traps `Validate.Error`.  
4. On success it calls `onValid()` (default inserts message + class). On failure it calls `onInvalid()`.

### 2.3 Cleanup  

* `destroy()` removes all event listeners, detaches from the form controller, clears validation arrays, and removes any UI elements or classes.  
* `LiveValidationForm.destroy()` restores the original `onsubmit` and removes its instance from the cache.

### 2.4 Dependencies & Constraints  

* **Pure JavaScript** – no jQuery, no modern ES6 features (e.g., `const`, `let`).  
* **DOM API** – relies on `document.getElementById`, `element.form`, `element.options`, etc.  
* **Browser support** – written for IE6+; however, modern browsers still run it fine.  
* **Assumptions** – fields are either `<input>`, `<textarea>`, or `<select>`. Other input types (radio, date, number) are unsupported unless wrapped in a `<div>` and manually handled.

### 2.5 Architecture Decisions  

* **Field‑centric vs. Form‑centric** – Each field owns its rules and UI, while the form controller aggregates only for the submit event.  
* **Event Delegation** – Directly attaches handlers to each element; could be replaced with a single delegated handler in a future version.  
* **Error Propagation** – Using a custom `Validate.Error` allows distinguishing validation failures from other runtime errors.

---

## 3. Functions/Methods  

### 3.1 LiveValidation  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `initialize(B, A)` | Constructor logic. | `element` (node or id), `options` | Sets up the object | Binds events, registers with form |
| `destroy()` | Remove validation and clean UI. | – | – | Restores original handlers, removes messages, classes |
| `add(type, params)` | Register a new rule. | Rule constructor (`Validate.xxx`), options | `this` | Pushes into `this.validations` |
| `remove(type, params)` | Unregister a rule. | Same as `add` | `this` | Removes from array |
| `deferValidation()` | Debounce validation call. | – | – | Sets/clears timeout |
| `doOnBlur(evt)` | Called on blur. | Event | – | Calls `validate()` |
| `doOnFocus(evt)` | Called on focus. | Event | – | Clears UI |
| `getElementType()` | Detects element kind. | – | One of constants | – |
| `doValidations()` | Execute all rules. | – | `true/false` | Sets `this.message` and `this.validationFailed` |
| `validateElement(type, params)` | Execute single rule. | Rule, options | `true/false` | Throws `Validate.Error` on failure |
| `validate()` | Public API to validate the field. | – | `true/false` | Triggers callbacks |
| `enable()` / `disable()` | Toggle `disabled` attribute. | – | `this` | Modifies UI |
| `createMessageSpan()` | Helper: build `<span>` with message text. | – | `<span>` | – |
| `insertMessage(span)` | Insert message after configured node. | `<span>` | – | Adds classes, DOM insertion |
| `addFieldClass()` / `removeFieldClass()` | Manage CSS classes on the field. | – | – | Adds `LV_valid_field` / `LV_invalid_field` |
| `removeMessage()` / `removeMessageAndFieldClass()` | Remove UI artifacts. | – | – | Deletes message `<span>` and classes |

### 3.2 LiveValidation.massValidate(array)  

* Static helper that loops through an array of `LiveValidation` objects, stops on first failure.  
* Returns `true` if all pass, otherwise `false`.

### 3.3 LiveValidationForm  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `initialize(form)` | Bind to form, override submit. | `<form>` | – | Sets `this.fields` array |
| `addField(field)` | Register a `LiveValidation` instance. | `LiveValidation` | – | Pushes into array |
| `removeField(field)` | Deregister. | – | – | Removes from array |
| `destroy()` | Restore original `onsubmit`, clear instance. | – | `true/false` | Detaches handlers, clears cache |
| `getInstance(form)` (static) | Singleton factory. | `<form>` | `LiveValidationForm` | Creates or reuses instance |

### 3.4 Validate  

| Function | Purpose | Parameters | Returns | Notes |
|----------|---------|------------|---------|-------|
| `Presence(value, opts)` | Non‑empty check. | Value, options | `true` | Throws `Validate.Error` on fail |
| `Numericality(value, opts)` | Number checks (min, max, integer). | Value, options | `true` | Handles `is`, `onlyInteger` |
| `Format(value, opts)` | Regex pattern match. | Value, options | `true` | `negate` flips logic |
| `Email(value, opts)` | Email regex wrapper. | Value, options | `true` |
| `Length(value, opts)` | String length checks. | Value, options | `true` |
| `Inclusion(value, opts)` | Membership in list. | Value, options | `true` |
| `Exclusion(value, opts)` | Opposite of Inclusion. | Value, options | `true` |
| `Confirmation(value, opts)` | Match another field’s value. | Value, options | `true` |
| `Acceptance(value, opts)` | Checkbox must be checked. | Value, options | `true` |
| `Custom(value, opts)` | User‑supplied validator. | Value, options | `true` |
| `now(fn, value, opts)` | Safe wrapper that returns boolean. | Function, value, options | `true/false` |
| `fail(message)` | Throw `Validate.Error`. | String | throws | |
| `Error(message)` | Error constructor. | String | `Error` object | |

All validator functions throw `Validate.Error` on failure, allowing `LiveValidation` to catch and display messages without propagating the exception.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `document.getElementById`, `document.createElement`, `element.form`, `element.options` | DOM API | Native |
| `Math.random`, `Date` | JS standard | Used for unique form IDs |
| `window.event` | IE legacy | Accessed only in old browsers for the submit handler |
| `LiveValidationForm.instances` | In‑memory map | Singleton registry |

No external libraries or build tools are required; the code is self‑contained.

---

## 5. Additional Notes & Recommendations  

### 5.1 Edge Cases & Limitations  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Unsupported input types** (radio, date, number) | Validation cannot be applied | Extend `getElementType` or provide a wrapper object |
| **Multiple submit buttons** | Only the first registered `onsubmit` runs | Hook into each button’s click to trigger validation or use form `submit` event with `e.preventDefault()` |
| **Dynamic DOM changes** | Adding/removing fields after initialization won’t auto‑register | Provide a method to re‑scan the form or use event delegation |
| **Message removal on blur** | `removeMessage()` removes *any* sibling element with the message class, even if it belongs to another field | Store a reference to the exact message element or use a data attribute |
| **CSS class pollution** | Class names are appended naïvely, may leave duplicate spaces | Use `classList` (modern) or regex to clean spaces |
| **Memory leaks** | Event handlers stored on the element reference might keep the object alive | Use `detach` in `destroy()` and null out references |
| **Accessibility** | Messages are inserted as plain `<span>`; no ARIA attributes | Add `aria-live="polite"` or associate with `aria-describedby` |
| **Custom error handling** | `Validate.now` swallows all non‑`Validate.Error` exceptions, potentially hiding bugs | Log unexpected errors or provide a callback for debugging |

### 5.2 Future Enhancements  

1. **ES6+ Migration** – rewrite using `class` syntax, arrow functions, `let/const`, and `module` pattern for better readability.  
2. **Event Delegation** – attach a single handler on the form or document to manage all fields, simplifying cleanup.  
3. **Promise / Async support** – allow asynchronous validators (e.g., remote username check) returning promises.  
4. **Built‑in localization** – expose i18n for default messages.  
5. **Custom UI** – expose hooks to render messages in tooltips, modal dialogs, or inline lists.  
6. **Validation Groups** – support “required on this step” for multi‑step forms.  
7. **Better API** – expose a fluent interface: `new LiveValidation(el).presence().length({min:3})` etc.  

### 5.3 Security Considerations  

* The library does **not** sanitize input values; it merely tests them.  
* All messages are inserted as text nodes, so XSS risk is minimal unless the `message` property is overwritten with HTML.  
* For highly sensitive data (e.g., passwords), consider hashing or server‑side validation in addition to client‑side.

---

**Verdict**  
LiveValidation 1.3 is a well‑structured, self‑contained library that covers most common form‑validation needs. It is lightweight, works on legacy browsers, and is easy to drop into a project. Modern JavaScript practices and additional features (async support, better UI hooks, accessibility) would make it more robust for current web applications.

## Code Critique



## Code Preview

```javascript
// LiveValidation 1.3 (standalone version)
// Copyright (c) 2007-2008 Alec Hill (www.livevalidation.com)
// LiveValidation is licensed under the terms of the MIT License
var LiveValidation=function(B,A){this.initialize(B,A);};LiveValidation.VERSION="1.3 standalone";LiveValidation.TEXTAREA=1;LiveValidation.TEXT=2;LiveValidation.PASSWORD=3;LiveValidation.CHECKBOX=4;LiveValidation.SELECT=5;LiveValidation.FILE=6;LiveValidation.massValidate=function(C){var D=true;for(var B=0,A=C.length;B<A;++B){var E=C[B].validate();if(D){D=E;}}return D;};LiveValidation.prototype={validClass:"LV_valid",invalidClass:"LV_invalid",messageClass:"LV_validation_message",validFieldClass:"LV_valid_field",invalidFieldClass:"LV_invalid_field",initialize:function(D,C){var A=this;if(!D){throw new Error("LiveValidation::initialize - No element reference or element id has been provided!");}this.element=D.nodeName?D:document.getElementById(D);if(!this.element){throw new Error("LiveValidation::initialize - No element with reference or id of '"+D+"' exists!");}this.validations=[];this.elementType=this.getElementType();this.form=this.element.form;var B=C||{};this.validMessage=B.validMessage||"Thankyou!";var E=B.insertAfterWhatNode||this.element;this.insertAfterWhatNode=E.nodeType?E:document.getElementById(E);this.onValid=B.onValid||function(){this.insertMessage(this.createMessageSpan());this.addFieldClass();};this.onInvalid=B.onInvalid||function(){this.insertMessage(this.createMessageSpan());this.addFieldClass();};this.onlyOnBlur=B.onlyOnBlur||false;this.wait=B.wait||0;this.onlyOnSubmit=B.onlyOnSubmit||false;if(this.form){this.formObj=LiveValidationForm.getInstance(this.form);this.formObj.addField(this);}this.oldOnFocus=this.element.onfocus||function(){};this.oldOnBlur=this.element.onblur||function(){};this.oldOnClick=this.element.onclick||function(){};this.oldOnChange=this.element.onchange||function(){};this.oldOnKeyup=this.element.onkeyup||function(){};this.element.onfocus=function(F){A.doOnFocus(F);return A.oldOnFocus.call(this,F);};if(!this.onlyOnSubmit){switch(this.elementType){case LiveValidation.CHECKBOX:this.element.onclick=function(F){A.validate();return A.oldOnClick.call(this,F);};case LiveValidation.SELECT:case LiveValidation.FILE:this.element.onchange=function(F){A.validate();return A.oldOnChange.call(this,F);};break;default:if(!this.onlyOnBlur){this.element.onkeyup=function(F){A.deferValidation();return A.oldOnKeyup.call(this,F);};}this.element.onblur=function(F){A.doOnBlur(F);return A.oldOnBlur.call(this,F);};}}},destroy:function(){if(this.formObj){this.formObj.removeField(this);this.formObj.destroy();}this.element.onfocus=this.oldOnFocus;if(!this.onlyOnSubmit){switch(this.elementType){case LiveValidation.CHECKBOX:this.element.onclick=this.oldOnClick;case LiveValidation.SELECT:case LiveValidation.FILE:this.element.onchange=this.oldOnChange;break;default:if(!this.onlyOnBlur){this.element.onkeyup=this.oldOnKeyup;}this.element.onblur=this.oldOnBlur;}}this.validations=[];this.removeMessageAndFieldClass();},add:function(A,B){this.validations.push({type:A,params:B||{}});return this;},remove:function(B,D){var E=false;for(var C=0,A=this.validations.length;C<A;C++){if(this.validations[C].type==B){if(this.validations[C].params==D){E=true;break;}}}if(E){this.validations.splice(C,1);}return this;},deferValidation:function(B){if(this.wait>=300){this.removeMessageAndFieldClass();}var A=this;if(this.timeout){clearTimeout(A.timeout);}this.timeout=setTimeout(function(){A.validate();},A.wait);},doOnBlur:function(A){this.focused=false;this.validate(A);},doOnFocus:function(A){this.focused=true;this.removeMessageAndFieldClass();},getElementType:function(){switch(true){case (this.element.nodeName.toUpperCase()=="TEXTAREA"):return LiveValidation.TEXTAREA;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="TEXT"):return LiveValidation.TEXT;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="PASSWORD"):return LiveValidation.PASSWORD;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="CHECKBOX"):return LiveValidation.CHECKBOX;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="FILE"):return LiveValidation.FILE;case (this.element.nodeName.toUpperCase()=="SELECT"):return LiveValidation.SELECT;case (this.element.nodeName.toUpperCase()=="INPUT"):throw new Error("LiveValidation::getElementType - Cannot use LiveValidation on an "+this.element.type+" input!");default:throw new Error("LiveValidation::getElementType - Element must be an input, select, or textarea!");}},doValidations:function(){this.validationFailed=false;for(var C=0,A=this.validations.length;C<A;++C){var B=this.validations[C];switch(B.type){case Validate.Presence:case Validate.Confirmation:case Validate.Acceptance:this.displayMessageWhenEmpty=true;this.validationFailed=!this.validateElement(B.type,B.params);break;default:this.validationFailed=!this.validateElement(B.type,B.params);break;}if(this.validationFailed){return false;}}this.message=this.validMessage;return true;},validateElement:function(A,C){var D=(this.elementType==LiveValidation.SELECT)?this.element.options[this.element.selectedIndex].value:this.element.value;if(A==Validate.Acceptance){if(this.elementType!=LiveValidation.CHECKBOX){throw new Error("LiveValidation::validateElement - Element to validate acceptance must be a checkbox!");}D=this.element.checked;}var E=true;try{A(D,C);}catch(B){if(B instanceof Validate.Error){if(D!==""||(D===""&&this.displayMessageWhenEmpty)){this.validationFailed=true;this.message=B.message;E=false;}}else{throw B;}}finally{return E;}},validate:function(){if(!this.element.disabled){var A=this.doValidations();if(A){this.onValid();return true;}else{this.onInvalid();return false;}}else{return true;}},enable:function(){this.element.disabled=false;return this;},disable:function(){this.element.disabled=true;this.removeMessageAndFieldClass();return this;},createMessageSpan:function(){var A=document.createElement("span");var B=document.createTextNode(this.message);A.appendChild(B);return A;},insertMessage:function(B){this.removeMessage();if((this.displayMessageWhenEmpty&&(this.elementType==LiveValidation.CHECKBOX||this.element.value==""))||this.element.value!=""){var A=this.validationFailed?this.invalidClass:this.validClass;B.className+=" "+this.messageClass+" "+A;if(this.insertAfterWhatNode.nextSibling){this.insertAfterWhatNode.parentNode.insertBefore(B,this.insertAfterWhatNode.nextSibling);}else{this.insertAfterWhatNode.parentNode.appendChild(B);}}},addFieldClass:function(){this.removeFieldClass();if(!this.validationFailed){if(this.displayMessageWhenEmpty||this.element.value!=""){if(this.element.className.indexOf(this.validFieldClass)==-1){this.element.className+=" "+this.validFieldClass;}}}else{if(this.element.className.indexOf(this.invalidFieldClass)==-1){this.element.className+=" "+this.invalidFieldClass;}}},removeMessage:function(){var A;var B=this.insertAfterWhatNode;while(B.nextSibling){if(B.nextSibling.nodeType===1){A=B.nextSibling;break;}B=B.nextSibling;}if(A&&A.className.indexOf(this.messageClass)!=-1){this.insertAfterWhatNode.parentNode.removeChild(A);}},removeFieldClass:function(){if(this.element.className.indexOf(this.invalidFieldClass)!=-1){this.element.className=this.element.className.split(this.invalidFieldClass).join("");}if(this.element.className.indexOf(this.validFieldClass)!=-1){this.element.className=this.element.className.split(this.validFieldClass).join(" ");}},removeMessageAndFieldClass:function(){this.removeMessage();this.removeFieldClass();}};var LiveValidationForm=function(A){this.initialize(A);};LiveValidationForm.instances={};LiveValidationForm.getInstance=function(A){var B=Math.random()*Math.random();if(!A.id){A.id="formId_"+B.toString().replace(/\./,"")+new Date().valueOf();}if(!LiveValidationForm.instances[A.id]){LiveValidationForm.instances[A.id]=new LiveValidationForm(A);}return LiveValidationForm.instances[A.id];};LiveValidationForm.prototype={initialize:function(B){this.name=B.id;this.element=B;this.fields=[];this.oldOnSubmit=this.element.onsubmit||function(){};var A=this;this.element.onsubmit=function(C){return(LiveValidation.massValidate(A.fields))?A.oldOnSubmit.call(this,C||window.event)!==false:false;};},addField:function(A){this.fields.push(A);},removeField:function(C){var D=[];for(var B=0,A=this.fields.length;B<A;B++){if(this.fields[B]!==C){D.push(this.fields[B]);}}this.fields=D;},destroy:function(A){if(this.fields.length!=0&&!A){return false;}this.element.onsubmit=this.oldOnSubmit;LiveValidationForm.instances[this.name]=null;return true;}};var Validate={Presence:function(B,C){var C=C||{};var A=C.failureMessage||"Can't be empty!";if(B===""||B===null||B===undefined){Validate.fail(A);}return true;},Numericality:function(J,E){var A=J;var J=Number(J);var E=E||{};var F=((E.minimum)||(E.minimum==0))?E.minimum:null;var C=((E.maximum)||(E.maximum==0))?E.maximum:null;var D=((E.is)||(E.is==0))?E.is:null;var G=E.notANumberMessage||"Must be a number!";var H=E.notAnIntegerMessage||"Must be an integer!";var I=E.wrongNumberMessage||"Must be "+D+"!";var B=E.tooLowMessage||"Must not be less than "+F+"!";var K=E.tooHighMessage||"Must not be more than "+C+"!";if(!isFinite(J)){Validate.fail(G);}if(E.onlyInteger&&(/\.0+$|\.$/.test(String(A))||J!=parseInt(J))){Validate.fail(H);}switch(true){case (D!==null):if(J!=Number(D)){Validate.fail(I);}break;case (F!==null&&C!==null):Validate.Numericality(J,{tooLowMessage:B,minimum:F});Validate.Numericality(J,{tooHighMessage:K,maximum:C});break;case (F!==null):if(J<Number(F)){Validate.fail(B);}break;case (C!==null):if(J>Number(C)){Validate.fail(K);}break;}return true;},Format:function(C,E){var C=String(C);var E=E||{};var A=E.failureMessage||"Not valid!";var B=E.pattern||/./;var D=E.negate||false;if(!D&&!B.test(C)){Validate.fail(A);}if(D&&B.test(C)){Validate.fail(A);}return true;},Email:function(B,C){var C=C||{};var A=C.failureMessage||"Must be a valid email address!";Validate.Format(B,{failureMessage:A,pattern:/^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i});return true;},Length:function(F,G){var F=String(F);var G=G||{};var E=((G.minimum)||(G.minimum==0))?G.minimum:null;var H=((G.maximum)||(G.maximum==0))?G.maximum:null;var C=((G.is)||(G.is==0))?G.is:null;var A=G.wrongLengthMessage||"Must be "+C+" characters long!";var B=G.tooShortMessage||"Must not be less than "+E+" characters long!";var D=G.tooLongMessage||"Must not be more than "+H+" characters long!";switch(true){case (C!==null):if(F.length!=Number(C)){Validate.fail(A);}break;case (E!==null&&H!==null):Validate.Length(F,{tooShortMessage:B,minimum:E});Validate.Length(F,{tooLongMessage:D,maximum:H});break;case (E!==null):if(F.length<Number(E)){Validate.fail(B);}break;case (H!==null):if(F.length>Number(H)){Validate.fail(D);}break;default:throw new Error("Validate::Length - Length(s) to validate against must be provided!");}return true;},Inclusion:function(H,F){var F=F||{};var K=F.failureMessage||"Must be included in the list!";var G=(F.caseSensitive===false)?false:true;if(F.allowNull&&H==null){return true;}if(!F.allowNull&&H==null){Validate.fail(K);}var D=F.within||[];if(!G){var A=[];for(var C=0,B=D.length;C<B;++C){var I=D[C];if(typeof I=="string"){I=I.toLowerCase();}A.push(I);}D=A;if(typeof H=="string"){H=H.toLowerCase();}}var J=false;for(var E=0,B=D.length;E<B;++E){if(D[E]==H){J=true;}if(F.partialMatch){if(H.indexOf(D[E])!=-1){J=true;}}}if((!F.negate&&!J)||(F.negate&&J)){Validate.fail(K);}return true;},Exclusion:function(A,B){var B=B||{};B.failureMessage=B.failureMessage||"Must not be included in the list!";B.negate=true;Validate.Inclusion(A,B);return true;},Confirmation:function(C,D){if(!D.match){throw new Error("Validate::Confirmation - Error validating confirmation: Id of element to match must be provided!");}var D=D||{};var B=D.failureMessage||"Does not match!";var A=D.match.nodeName?D.match:document.getElementById(D.match);if(!A){throw new Error("Validate::Confirmation - There is no reference with name of, or element with id of '"+D.match+"'!");}if(C!=A.value){Validate.fail(B);}return true;},Acceptance:function(B,C){var C=C||{};var A=C.failureMessage||"Must be accepted!";if(!B){Validate.fail(A);}return true;},Custom:function(D,E){var E=E||{};var B=E.against||function(){return true;};var A=E.args||{};var C=E.failureMessage||"Not valid!";if(!B(D,A)){Validate.fail(C);}return true;},now:function(A,D,C){if(!A){throw new Error("Validate::now - Validation function must be provided!");}var E=true;try{A(D,C||{});}catch(B){if(B instanceof Validate.Error){E=false;}else{throw B;}}finally{return E;}},fail:function(A){throw new Validate.Error(A);},Error:function(A){this.message=A;this.name="ValidationError";}};



```
