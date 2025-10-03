# livevalidation_standalone.compressed.js

## Review

## 1. Summary  

**Purpose**  
The code is a lightweight, client‑side form validation library written in plain JavaScript (ES3‑style). It allows developers to attach a variety of built‑in validation rules to individual form fields and, optionally, to an entire form. When validation fails, the library displays contextual messages and applies CSS classes to highlight invalid fields.

**Key Components**  

| Component | Role |
|-----------|------|
| `LiveValidation` | Represents a single form element (input, select, textarea). Handles event wiring, validation execution, and DOM messaging. |
| `LiveValidationForm` | Maintains a collection of `LiveValidation` instances belonging to a single form. Intercepts the form’s `onsubmit` to run all validations before allowing submission. |
| `Validate` | A namespace containing static validator functions (Presence, Numericality, Format, Email, Length, Inclusion, Exclusion, Confirmation, Acceptance, Custom). Each validator throws a `Validate.Error` on failure. |
| `LiveValidation.massValidate` | Static helper that runs `validate()` on an array of fields (used internally by the form). |

**Design Patterns / Libraries**  
- *Singleton* – `LiveValidationForm.getInstance` ensures only one form manager per DOM form.  
- *Strategy* – Each validator function can be passed as a strategy to a field.  
- *Observer* – The library attaches to native DOM events (`focus`, `blur`, `keyup`, `click`, `change`) to reactively validate fields.  
- No external dependencies; it relies solely on native DOM APIs and JavaScript primitives.

---

## 2. Detailed Description  

### Initialization Flow  

1. **`LiveValidation(element, options)`**  
   - Receives a DOM element reference or ID.  
   - Determines element type (`TEXTAREA`, `TEXT`, `PASSWORD`, etc.).  
   - Stores references to the element, its parent form, and any user‑supplied options (`validMessage`, `onValid`, `onInvalid`, `onlyOnBlur`, `wait`, `onlyOnSubmit`, `insertAfterWhatNode`).  
   - Hooks event handlers on the element (focus, blur, keyup, click/change).  
   - If a form is found, registers itself with `LiveValidationForm.getInstance(form)`.

2. **`LiveValidationForm`**  
   - Keeps an array of fields belonging to the form.  
   - Replaces the form’s `onsubmit` with a wrapper that first calls `LiveValidation.massValidate(fields)`.  
   - The wrapper returns the original `onsubmit` result only if all validations pass.

### Runtime Behavior  

- **Event‑Driven Validation**  
  - `onfocus` → clears previous message and marks field as “focused”.  
  - `onblur` (or click/change for checkboxes/selects) → triggers validation immediately.  
  - `onkeyup` (unless `onlyOnBlur` is true) → schedules a deferred validation after `wait` milliseconds.  

- **Validation Execution**  
  - `doValidations()` iterates through all validators attached to the field.  
  - Each validator receives the field’s current value (or checked state for checkboxes).  
  - Validators throw `Validate.Error` on failure; `doValidations()` catches it, sets `validationFailed`, and stores the error message.  
  - After all validators, the field calls either `onValid()` or `onInvalid()`.

- **DOM Feedback**  
  - `createMessageSpan()` builds a `<span>` containing the message.  
  - `insertMessage()` inserts it after a user‑defined node (defaults to the field itself).  
  - `addFieldClass()` and `removeFieldClass()` toggle `LV_valid_field` / `LV_invalid_field`.  
  - `removeMessage()` removes any existing message node.

- **Cleanup**  
  - `destroy()` detaches event handlers, removes message and CSS classes, and unregisters the field from its form.

### Assumptions & Constraints  

- Works with legacy browsers (ES3). No modern JavaScript features.  
- Expects the form’s `onsubmit` to return `true` or `false` (no async submission).  
- Relies on the DOM to expose `nodeName`, `type`, `options`, etc.  
- Validation messages and classes are hard‑coded but can be overridden via options.  

### Architecture & Design Choices  

- **Modular, Self‑Contained** – The entire library is wrapped in one IIFE‑style object; no global namespace pollution beyond `LiveValidation` and `Validate`.  
- **Event‑Based vs Polling** – Uses event handlers for instant feedback, minimizing polling overhead.  
- **Synchronous Validation** – All checks run synchronously; no AJAX or async validation is provided.  
- **Error Handling** – Validators throw a custom error to signal failure; this pattern centralizes error handling in `doValidations()`.

---

## 3. Functions/Methods  

### LiveValidation  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `initialize(D, C)` | Core constructor logic; wires events. | `D`: element or id.<br>`C`: options object. | `this` | Sets event handlers; registers with form. |
| `destroy()` | Removes event handlers, cleans up DOM. | – | – | Restores original handlers, removes message and classes. |
| `add(A, B)` | Attaches a validator. | `A`: validator type (e.g., `Validate.Presence`).<br>`B`: validator parameters. | `this` | Pushes to `this.validations`. |
| `remove(B, D)` | Detaches a validator. | `B`: validator type.<br>`D`: parameters. | `this` | Removes matching validator from array. |
| `deferValidation(B)` | Schedules delayed validation. | `B`: unused. | – | Calls `validate()` after `wait` ms. |
| `doOnBlur(A)` | Focus flag reset + immediate validation. | `A`: event. | – | `validate()` invoked. |
| `doOnFocus(A)` | Clears message, marks as focused. | `A`: event. | – | `removeMessageAndFieldClass()` called. |
| `getElementType()` | Returns element type constant. | – | Integer | – |
| `doValidations()` | Runs all validators; sets message & failure flag. | – | Boolean | Populates `this.message`. |
| `validateElement(A, C)` | Invokes single validator, handles errors. | `A`: validator type.<br>`C`: params. | Boolean | Throws on failure. |
| `validate()` | Entry point for validation; triggers callbacks. | – | Boolean | Calls `onValid`/`onInvalid`. |
| `enable()` / `disable()` | Toggle `disabled` state; clean up if disabled. | – | `this` | Removes message and classes on disable. |
| `createMessageSpan()` | Builds span element with message. | – | DOM node | – |
| `insertMessage(B)` | Inserts message span after target node. | `B`: span element. | – | DOM mutation. |
| `addFieldClass()` | Applies valid/invalid CSS class to field. | – | – | Modifies `className`. |
| `removeMessage()` | Removes any existing message node. | – | – | DOM mutation. |
| `removeFieldClass()` | Strips validation CSS classes from field. | – | – | Modifies `className`. |
| `removeMessageAndFieldClass()` | Convenience to clean up. | – | – | Calls above two. |

### LiveValidationForm  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `initialize(B)` | Stores form element, hooks `onsubmit`. | `B`: form element. | – | Overrides `form.onsubmit`. |
| `addField(A)` | Adds a field to the form. | `A`: `LiveValidation` instance. | – | Pushes to `this.fields`. |
| `removeField(C)` | Removes a field from the form. | `C`: field instance. | – | Filters array. |
| `destroy(A)` | Restores original `onsubmit` if no fields left. | `A`: boolean to force? | Boolean | Resets `onsubmit`. |

### Validate  

| Function | Purpose | Parameters | Return | Side‑Effects |
|----------|---------|------------|--------|--------------|
| `Presence(B, C)` | Checks non‑empty value. | `B`: value.<br>`C`: options. | `true` | Throws `Validate.Error` on fail. |
| `Numericality(J, E)` | Checks numeric range, integer, etc. | `J`: value.<br>`E`: options. | `true` | Throws on fail. |
| `Format(C, E)` | RegExp pattern match. | `C`: value.<br>`E`: options. | `true` | Throws on fail. |
| `Email(B, C)` | Validates email format via `Format`. | – | – | – |
| `Length(F, G)` | Checks string length constraints. | – | – | – |
| `Inclusion(H, F)` | Checks value is in a list. | – | – | – |
| `Exclusion(A, B)` | Opposite of Inclusion. | – | – | – |
| `Confirmation(C, D)` | Matches another field’s value. | – | – | – |
| `Acceptance(B, C)` | Requires a checkbox to be checked. | – | – | – |
| `Custom(D, E)` | User‑supplied predicate. | – | – | – |
| `now(A, D, C)` | Executes a validator synchronously. | – | Boolean | – |
| `fail(A)` | Throws a `Validate.Error`. | `A`: message. | – | – |
| `Error(A)` | Constructor for validation error. | – | – | – |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `document.getElementById`, `document.createElement`, `document.createTextNode` | Standard DOM APIs | No polyfills required. |
| `Math.random`, `Date.valueOf` | Native | Used for generating unique form IDs. |
| None other | | Entire library is self‑contained; no external frameworks or third‑party scripts. |

**Platform‑specific assumptions**  
- Works in browsers that support DOM Level 1 and basic ECMAScript 3.  
- Relies on the `className` string for adding/removing CSS classes; may conflict with other libraries that manipulate `className` in a non‑standard way.  

---

## 5. Additional Notes  

### Strengths  

- **Zero Dependencies** – Great for legacy projects or minimal pages.  
- **Clear API** – Simple chaining (`new LiveValidation('field').add(Validate.Presence)`).  
- **Extensible** – New validators can be added by extending the `Validate` namespace.  
- **Graceful Fallback** – If the element or form is missing, descriptive errors are thrown.

### Potential Weaknesses / Edge Cases  

1. **Class Name Collision** – The library uses simple string manipulation (`split/join`) to remove CSS classes, which can leave extra spaces or fail if the same class appears elsewhere in the class list.  
2. **Event Overwrite** – It replaces the element’s `onsubmit` entirely, which could conflict with other libraries that also override `onsubmit`. It only restores the original handler on `destroy()`.  
3. **`onlyOnSubmit` Option** – If set, validation will run only on form submit. The library still attaches event handlers for focus/blur; this may produce misleading UI if the user never submits.  
4. **No Async Validation** – Cannot handle server‑side checks (e.g., uniqueness) without custom logic.  
5. **Browser Compatibility** – The code predates `addEventListener`; older browsers (IE<9) may work, but modern browsers can still run it but might consider it outdated.  
6. **Error Handling** – Validators throw a custom `Validate.Error`. If any non‑`Validate.Error` is thrown (e.g., programmer mistake), it propagates uncaught, potentially breaking the form.  
7. **`insertAfterWhatNode`** – If the reference node changes (e.g., moved in DOM), the message may end up in the wrong place.  

### Suggested Enhancements  

| Area | Recommendation |
|------|----------------|
| **Class handling** | Use `classList` (when available) or a small helper that preserves spacing and supports removal of multiple classes robustly. |
| **Event attachment** | Switch to `addEventListener`/`attachEvent` abstraction to avoid overriding existing handlers. |
| **Async support** | Provide a `Validate.async` wrapper that accepts a promise/callback and defers form submission until resolved. |
| **Accessibility** | Add `aria-invalid` and `aria-describedby` attributes to communicate validation state to screen readers. |
| **Configuration** | Allow global defaults via `LiveValidation.defaults = { … }`. |
| **Modular build** | Split the code into ES6 modules or UMD wrapper so it can be imported via npm or a bundler. |
| **Testing** | Add unit tests (e.g., Jest) to cover edge cases like empty string, zero values, and class name collisions. |

### Final Verdict  

For projects that require a lightweight, synchronous, client‑side validator and can tolerate the quirks of older DOM APIs, this library remains a solid choice. Its API is intuitive, and the code is well‑organized despite being in a single file. However, for modern web applications, consider migrating to a more recent validation framework (e.g., **Yup**, **validator.js**, or native HTML5 constraint validation) that offers async support, better browser compatibility, and a more maintainable codebase.

## Code Critique



## Code Preview

```javascript
// LiveValidation 1.3 (standalone version)
// Copyright (c) 2007-2008 Alec Hill (www.livevalidation.com)
// LiveValidation is licensed under the terms of the MIT License
var LiveValidation=function(B,A){this.initialize(B,A);};LiveValidation.VERSION="1.3 standalone";LiveValidation.TEXTAREA=1;LiveValidation.TEXT=2;LiveValidation.PASSWORD=3;LiveValidation.CHECKBOX=4;LiveValidation.SELECT=5;LiveValidation.FILE=6;LiveValidation.massValidate=function(C){var D=true;for(var B=0,A=C.length;B<A;++B){var E=C[B].validate();if(D){D=E;}}return D;};LiveValidation.prototype={validClass:"LV_valid",invalidClass:"LV_invalid",messageClass:"LV_validation_message",validFieldClass:"LV_valid_field",invalidFieldClass:"LV_invalid_field",initialize:function(D,C){var A=this;if(!D){throw new Error("LiveValidation::initialize - No element reference or element id has been provided!");}this.element=D.nodeName?D:document.getElementById(D);if(!this.element){throw new Error("LiveValidation::initialize - No element with reference or id of '"+D+"' exists!");}this.validations=[];this.elementType=this.getElementType();this.form=this.element.form;var B=C||{};this.validMessage=B.validMessage||"Thankyou!";var E=B.insertAfterWhatNode||this.element;this.insertAfterWhatNode=E.nodeType?E:document.getElementById(E);this.onValid=B.onValid||function(){this.insertMessage(this.createMessageSpan());this.addFieldClass();};this.onInvalid=B.onInvalid||function(){this.insertMessage(this.createMessageSpan());this.addFieldClass();};this.onlyOnBlur=B.onlyOnBlur||false;this.wait=B.wait||0;this.onlyOnSubmit=B.onlyOnSubmit||false;if(this.form){this.formObj=LiveValidationForm.getInstance(this.form);this.formObj.addField(this);}this.oldOnFocus=this.element.onfocus||function(){};this.oldOnBlur=this.element.onblur||function(){};this.oldOnClick=this.element.onclick||function(){};this.oldOnChange=this.element.onchange||function(){};this.oldOnKeyup=this.element.onkeyup||function(){};this.element.onfocus=function(F){A.doOnFocus(F);return A.oldOnFocus.call(this,F);};if(!this.onlyOnSubmit){switch(this.elementType){case LiveValidation.CHECKBOX:this.element.onclick=function(F){A.validate();return A.oldOnClick.call(this,F);};case LiveValidation.SELECT:case LiveValidation.FILE:this.element.onchange=function(F){A.validate();return A.oldOnChange.call(this,F);};break;default:if(!this.onlyOnBlur){this.element.onkeyup=function(F){A.deferValidation();return A.oldOnKeyup.call(this,F);};}this.element.onblur=function(F){A.doOnBlur(F);return A.oldOnBlur.call(this,F);};}}},destroy:function(){if(this.formObj){this.formObj.removeField(this);this.formObj.destroy();}this.element.onfocus=this.oldOnFocus;if(!this.onlyOnSubmit){switch(this.elementType){case LiveValidation.CHECKBOX:this.element.onclick=this.oldOnClick;case LiveValidation.SELECT:case LiveValidation.FILE:this.element.onchange=this.oldOnChange;break;default:if(!this.onlyOnBlur){this.element.onkeyup=this.oldOnKeyup;}this.element.onblur=this.oldOnBlur;}}this.validations=[];this.removeMessageAndFieldClass();},add:function(A,B){this.validations.push({type:A,params:B||{}});return this;},remove:function(B,D){var E=false;for(var C=0,A=this.validations.length;C<A;C++){if(this.validations[C].type==B){if(this.validations[C].params==D){E=true;break;}}}if(E){this.validations.splice(C,1);}return this;},deferValidation:function(B){if(this.wait>=300){this.removeMessageAndFieldClass();}var A=this;if(this.timeout){clearTimeout(A.timeout);}this.timeout=setTimeout(function(){A.validate();},A.wait);},doOnBlur:function(A){this.focused=false;this.validate(A);},doOnFocus:function(A){this.focused=true;this.removeMessageAndFieldClass();},getElementType:function(){switch(true){case (this.element.nodeName.toUpperCase()=="TEXTAREA"):return LiveValidation.TEXTAREA;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="TEXT"):return LiveValidation.TEXT;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="PASSWORD"):return LiveValidation.PASSWORD;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="CHECKBOX"):return LiveValidation.CHECKBOX;case (this.element.nodeName.toUpperCase()=="INPUT"&&this.element.type.toUpperCase()=="FILE"):return LiveValidation.FILE;case (this.element.nodeName.toUpperCase()=="SELECT"):return LiveValidation.SELECT;case (this.element.nodeName.toUpperCase()=="INPUT"):throw new Error("LiveValidation::getElementType - Cannot use LiveValidation on an "+this.element.type+" input!");default:throw new Error("LiveValidation::getElementType - Element must be an input, select, or textarea!");}},doValidations:function(){this.validationFailed=false;for(var C=0,A=this.validations.length;C<A;++C){var B=this.validations[C];switch(B.type){case Validate.Presence:case Validate.Confirmation:case Validate.Acceptance:this.displayMessageWhenEmpty=true;this.validationFailed=!this.validateElement(B.type,B.params);break;default:this.validationFailed=!this.validateElement(B.type,B.params);break;}if(this.validationFailed){return false;}}this.message=this.validMessage;return true;},validateElement:function(A,C){var D=(this.elementType==LiveValidation.SELECT)?this.element.options[this.element.selectedIndex].value:this.element.value;if(A==Validate.Acceptance){if(this.elementType!=LiveValidation.CHECKBOX){throw new Error("LiveValidation::validateElement - Element to validate acceptance must be a checkbox!");}D=this.element.checked;}var E=true;try{A(D,C);}catch(B){if(B instanceof Validate.Error){if(D!==""||(D===""&&this.displayMessageWhenEmpty)){this.validationFailed=true;this.message=B.message;E=false;}}else{throw B;}}finally{return E;}},validate:function(){if(!this.element.disabled){var A=this.doValidations();if(A){this.onValid();return true;}else{this.onInvalid();return false;}}else{return true;}},enable:function(){this.element.disabled=false;return this;},disable:function(){this.element.disabled=true;this.removeMessageAndFieldClass();return this;},createMessageSpan:function(){var A=document.createElement("span");var B=document.createTextNode(this.message);A.appendChild(B);return A;},insertMessage:function(B){this.removeMessage();if((this.displayMessageWhenEmpty&&(this.elementType==LiveValidation.CHECKBOX||this.element.value==""))||this.element.value!=""){var A=this.validationFailed?this.invalidClass:this.validClass;B.className+=" "+this.messageClass+" "+A;if(this.insertAfterWhatNode.nextSibling){this.insertAfterWhatNode.parentNode.insertBefore(B,this.insertAfterWhatNode.nextSibling);}else{this.insertAfterWhatNode.parentNode.appendChild(B);}}},addFieldClass:function(){this.removeFieldClass();if(!this.validationFailed){if(this.displayMessageWhenEmpty||this.element.value!=""){if(this.element.className.indexOf(this.validFieldClass)==-1){this.element.className+=" "+this.validFieldClass;}}}else{if(this.element.className.indexOf(this.invalidFieldClass)==-1){this.element.className+=" "+this.invalidFieldClass;}}},removeMessage:function(){var A;var B=this.insertAfterWhatNode;while(B.nextSibling){if(B.nextSibling.nodeType===1){A=B.nextSibling;break;}B=B.nextSibling;}if(A&&A.className.indexOf(this.messageClass)!=-1){this.insertAfterWhatNode.parentNode.removeChild(A);}},removeFieldClass:function(){if(this.element.className.indexOf(this.invalidFieldClass)!=-1){this.element.className=this.element.className.split(this.invalidFieldClass).join("");}if(this.element.className.indexOf(this.validFieldClass)!=-1){this.element.className=this.element.className.split(this.validFieldClass).join(" ");}},removeMessageAndFieldClass:function(){this.removeMessage();this.removeFieldClass();}};var LiveValidationForm=function(A){this.initialize(A);};LiveValidationForm.instances={};LiveValidationForm.getInstance=function(A){var B=Math.random()*Math.random();if(!A.id){A.id="formId_"+B.toString().replace(/\./,"")+new Date().valueOf();}if(!LiveValidationForm.instances[A.id]){LiveValidationForm.instances[A.id]=new LiveValidationForm(A);}return LiveValidationForm.instances[A.id];};LiveValidationForm.prototype={initialize:function(B){this.name=B.id;this.element=B;this.fields=[];this.oldOnSubmit=this.element.onsubmit||function(){};var A=this;this.element.onsubmit=function(C){return(LiveValidation.massValidate(A.fields))?A.oldOnSubmit.call(this,C||window.event)!==false:false;};},addField:function(A){this.fields.push(A);},removeField:function(C){var D=[];for(var B=0,A=this.fields.length;B<A;B++){if(this.fields[B]!==C){D.push(this.fields[B]);}}this.fields=D;},destroy:function(A){if(this.fields.length!=0&&!A){return false;}this.element.onsubmit=this.oldOnSubmit;LiveValidationForm.instances[this.name]=null;return true;}};var Validate={Presence:function(B,C){var C=C||{};var A=C.failureMessage||"Can't be empty!";if(B===""||B===null||B===undefined){Validate.fail(A);}return true;},Numericality:function(J,E){var A=J;var J=Number(J);var E=E||{};var F=((E.minimum)||(E.minimum==0))?E.minimum:null;var C=((E.maximum)||(E.maximum==0))?E.maximum:null;var D=((E.is)||(E.is==0))?E.is:null;var G=E.notANumberMessage||"Must be a number!";var H=E.notAnIntegerMessage||"Must be an integer!";var I=E.wrongNumberMessage||"Must be "+D+"!";var B=E.tooLowMessage||"Must not be less than "+F+"!";var K=E.tooHighMessage||"Must not be more than "+C+"!";if(!isFinite(J)){Validate.fail(G);}if(E.onlyInteger&&(/\.0+$|\.$/.test(String(A))||J!=parseInt(J))){Validate.fail(H);}switch(true){case (D!==null):if(J!=Number(D)){Validate.fail(I);}break;case (F!==null&&C!==null):Validate.Numericality(J,{tooLowMessage:B,minimum:F});Validate.Numericality(J,{tooHighMessage:K,maximum:C});break;case (F!==null):if(J<Number(F)){Validate.fail(B);}break;case (C!==null):if(J>Number(C)){Validate.fail(K);}break;}return true;},Format:function(C,E){var C=String(C);var E=E||{};var A=E.failureMessage||"Not valid!";var B=E.pattern||/./;var D=E.negate||false;if(!D&&!B.test(C)){Validate.fail(A);}if(D&&B.test(C)){Validate.fail(A);}return true;},Email:function(B,C){var C=C||{};var A=C.failureMessage||"Must be a valid email address!";Validate.Format(B,{failureMessage:A,pattern:/^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i});return true;},Length:function(F,G){var F=String(F);var G=G||{};var E=((G.minimum)||(G.minimum==0))?G.minimum:null;var H=((G.maximum)||(G.maximum==0))?G.maximum:null;var C=((G.is)||(G.is==0))?G.is:null;var A=G.wrongLengthMessage||"Must be "+C+" characters long!";var B=G.tooShortMessage||"Must not be less than "+E+" characters long!";var D=G.tooLongMessage||"Must not be more than "+H+" characters long!";switch(true){case (C!==null):if(F.length!=Number(C)){Validate.fail(A);}break;case (E!==null&&H!==null):Validate.Length(F,{tooShortMessage:B,minimum:E});Validate.Length(F,{tooLongMessage:D,maximum:H});break;case (E!==null):if(F.length<Number(E)){Validate.fail(B);}break;case (H!==null):if(F.length>Number(H)){Validate.fail(D);}break;default:throw new Error("Validate::Length - Length(s) to validate against must be provided!");}return true;},Inclusion:function(H,F){var F=F||{};var K=F.failureMessage||"Must be included in the list!";var G=(F.caseSensitive===false)?false:true;if(F.allowNull&&H==null){return true;}if(!F.allowNull&&H==null){Validate.fail(K);}var D=F.within||[];if(!G){var A=[];for(var C=0,B=D.length;C<B;++C){var I=D[C];if(typeof I=="string"){I=I.toLowerCase();}A.push(I);}D=A;if(typeof H=="string"){H=H.toLowerCase();}}var J=false;for(var E=0,B=D.length;E<B;++E){if(D[E]==H){J=true;}if(F.partialMatch){if(H.indexOf(D[E])!=-1){J=true;}}}if((!F.negate&&!J)||(F.negate&&J)){Validate.fail(K);}return true;},Exclusion:function(A,B){var B=B||{};B.failureMessage=B.failureMessage||"Must not be included in the list!";B.negate=true;Validate.Inclusion(A,B);return true;},Confirmation:function(C,D){if(!D.match){throw new Error("Validate::Confirmation - Error validating confirmation: Id of element to match must be provided!");}var D=D||{};var B=D.failureMessage||"Does not match!";var A=D.match.nodeName?D.match:document.getElementById(D.match);if(!A){throw new Error("Validate::Confirmation - There is no reference with name of, or element with id of '"+D.match+"'!");}if(C!=A.value){Validate.fail(B);}return true;},Acceptance:function(B,C){var C=C||{};var A=C.failureMessage||"Must be accepted!";if(!B){Validate.fail(A);}return true;},Custom:function(D,E){var E=E||{};var B=E.against||function(){return true;};var A=E.args||{};var C=E.failureMessage||"Not valid!";if(!B(D,A)){Validate.fail(C);}return true;},now:function(A,D,C){if(!A){throw new Error("Validate::now - Validation function must be provided!");}var E=true;try{A(D,C||{});}catch(B){if(B instanceof Validate.Error){E=false;}else{throw B;}}finally{return E;}},fail:function(A){throw new Validate.Error(A);},Error:function(A){this.message=A;this.name="ValidationError";}};



```
