# functions.js

## Review

## 1. Summary

The code is a small collection of vanilla JavaScript functions that:

1. **Persist shopping‑cart state in a cookie** (`setShoppingCartCookie`).
2. **Calculate and display a product price** based on selected options (`setPrice`, `setPriceCallback`).
3. **Collect user input from a form** (`getAttributes`).
4. **Submit a form programmatically** (`submitFORM`).
5. **Toggle tabbed UI sections** (`toggleTAB`).
6. **Detect the browser** (`whichBrs`) and apply IE‑specific CSS.

The functions are tightly coupled to the DOM and to external services such as `jQuery.cookie` and an undefined global `Catalog` object. No modern build tools, modules, or type safety are used.

---

## 2. Detailed Description

### Core Flow

1. **Cart Persistence**  
   - `setShoppingCartCookie(key, data)` checks whether a `data` object contains a `jsonShoppingCart` property.  
   - If it exists, the function stores it in a cookie that expires in 365 days.  
   - If the property is absent, it removes the cookie if it exists.

2. **Price Calculation**  
   - `setPrice()` grabs the form named `attributes`, collects its values with `getAttributes`, then forwards them to `Catalog.setPrice`.  
   - The result (expected to be HTML or a string) is rendered into the element with id `price` by `setPriceCallback`.

3. **Attribute Collection**  
   - `getAttributes(theForm)` iterates over all form elements, skipping buttons.  
   - For each input type (text/textarea, checkbox, select-one, radio) it creates an object with a name/value pair and stores it in an array.  
   - The array is returned to `setPrice`.

4. **UI Interaction**  
   - `submitFORM(id)` submits a form with the supplied id.  
   - `toggleTAB(num)` switches between tabs by manipulating class names and `display` styles.  
   - `whichBrs()` returns a string describing the browser based on the user‑agent, then conditionally injects CSS for Firefox.

### Assumptions & Dependencies

| Assumption | Reason |
|------------|--------|
| Global `Catalog` object | Used to calculate price; not defined in the snippet. |
| jQuery and jQuery.cookie plugin | Required for cookie handling. |
| Form named `attributes` | Hard‑coded selector in `setPrice`. |
| `document.getElementById("product.productId")` | Expects an element with that id to exist. |
| `document.getElementById('price')` | Destination for price string. |
| Global `tabCount` | Used in `toggleTAB`. |
| Browser detection via user‑agent | Used only to inject a few CSS rules for Firefox. |

---

## 3. Functions/Methods

| Function | Purpose | Inputs | Outputs | Side Effects |
|----------|---------|--------|---------|--------------|
| `setShoppingCartCookie(key, data)` | Persists or removes cart cookie | `key` (string), `data` (object) | None | Calls `jQuery.cookie`, manipulates document cookie |
| `setPrice()` | Initiates price calculation based on form data | None | None | Reads form, triggers `Catalog.setPrice` |
| `getAttributes(theForm)` | Builds an array of selected attribute objects | `theForm` (HTMLFormElement) | Array of attribute objects | None |
| `setPriceCallback(data)` | Updates the price display | `data` (string) | None | Modifies innerHTML of `#price` |
| `submitFORM(id)` | Submits a form programmatically | `id` (string) | None | Calls `.submit()` on the form element |
| `toggleTAB(num)` | Switches visible tab | `num` (int) | None | Changes class names and style.display of tab elements |
| `whichBrs()` | Browser sniffing based on user‑agent | None | Browser name (string) | None |
| **Inline CSS injection** | Inserts a `<style>` tag for Firefox | None | None | Adds style rules to the document head |

### Utility / Reusable Methods

- `getAttributes` is generic enough to be reused for any form, but the hard‑coded element creation logic (`attr.name = theForm.elements[i].value`) is not correct; it should use the element’s `name` property.
- `toggleTAB` relies on global `tabCount`; it would be safer to pass an array of tab IDs.

---

## 4. Dependencies

| Library / API | Type | Notes |
|---------------|------|-------|
| jQuery | Third‑party | Required for `$.cookie` call. |
| jQuery.cookie plugin | Third‑party | Provides the `cookie()` method. |
| `Catalog` global | Unknown | Must provide a `setPrice` method. |
| DOM APIs | Standard | `document`, `getElementById`, form submission. |
| User‑Agent string | Standard | Used for browser detection. |

No module system, no ES6 syntax, and no type safety are present.

---

## 5. Additional Notes

### Issues & Edge Cases

| Area | Problem | Impact | Suggested Fix |
|------|---------|--------|---------------|
| **Attribute collection** | `attr.name = theForm.elements[i].value` uses the *value* instead of the element’s *name* | Wrong key in the payload | Use `element.name` |
| **Text/textarea handling** | `attr.textValue = value;` references an undefined variable `value` | Reference error, script stops | Remove or replace with actual element value |
| **Browser detection** | Relying on user‑agent is brittle; many modern browsers hide or change UA strings | Potential false positives | Prefer feature detection or use modern polyfills |
| **Cookie expiration** | `expires: 365` is interpreted as days by jQuery.cookie but can be mis‑understood | Fine for long‑term storage | Document that it means 365 days |
| **Global variables** | `tabCount`, `browser` etc. clutter the namespace | Risk of collision | Wrap in an IIFE or module |
| **CSS injection** | Inline `<style>` tags can cause duplicate rules or CSS injection issues | Minor, but better to use a stylesheet | Move styles to an external CSS file |
| **Form ID hard‑coding** | `document.getElementById("product.productId")` uses a dot in the id, which is valid but unusual | Potential confusion | Use a hyphen or underscore |
| **Missing error handling** | `Catalog.setPrice` could fail silently | User sees no price update | Add error callback or fallback |
| **Accessibility** | No ARIA roles for tabs | Poor accessibility | Add `role="tab"` / `role="tabpanel"` |
| **Code style** | Mixed indentation and inconsistent spacing | Hard to read | Adopt a consistent style guide |

### Potential Enhancements

1. **Modularization** – Wrap all functions in an object or ES6 module to avoid polluting the global namespace.
2. **Type safety** – Use TypeScript or JSDoc annotations to clarify input and output types.
3. **Feature detection** – Replace UA sniffing with checks like `navigator.userAgentData` or CSS media queries.
4. **Error handling** – Add callbacks for success/failure when communicating with `Catalog`.
5. **Accessibility** – Ensure the tab UI is keyboard navigable and uses ARIA attributes.
6. **Testing** – Write unit tests (e.g., with Jest) for `getAttributes` and `setPrice` logic.
7. **Modern APIs** – Replace `document.getElementById` with `document.querySelector` and avoid global form references.

---

## 6. Conclusion

The code provides basic cart and UI utilities but suffers from a number of maintainability and correctness issues:

- Hard‑coded assumptions about DOM structure and element names.
- Reliance on global variables and outdated browser detection.
- Minor bugs (undefined `value` variable, wrong attribute names).
- Lack of modularity and type safety.

Refactoring the code into a small, self‑contained module, fixing the attribute extraction logic, and removing the brittle browser sniffing would make the implementation more robust, testable, and future‑proof.

## Code Critique



## Code Preview

```javascript
/** synchronize cart cookie **/

function setShoppingCartCookie(key, data) {

			if(data != null && data.jsonShoppingCart!=null) {
				jQuery.cookie(key,data.jsonShoppingCart, { expires: 365,path: '/'});
			} else {
				if(jQuery.cookie(key)!=null) {
					jQuery.cookie(key,null,{ path: '/'});
				}
			}
}

/** product price **/

function setPrice() {

	var theForm = document.attributes;

	var options = getAttributes(theForm);

	if(options && options.length>0) {

		var id = document.getElementById("product.productId").value;
		Catalog.setPrice(options,id,setPriceCallback);

	}

}

/** options / attributes **/

function getAttributes(theForm) {
	//gather all options
	
	var products = new Array();

	var count = 0;

	for(i=0; i<theForm.elements.length; i++){

		if(theForm.elements[i].type == "button") {
			continue;
		}
		if(theForm.elements[i].type == "text" || theForm.elements[i].type == "textarea"){
			
			if(theForm.elements[i].value!='') {
				var attr = new Object();
				attr.name = theForm.elements[i].value;
				attr.value = theForm.elements[i].value;
				attr.stringValue = true;
				attr.textValue = value;
				products[count] = attr;
				count++;
			}
						
		}else if(theForm.elements[i].type == "checkbox"){
			if(theForm.elements[i].checked) {
				var attr = new Object();
				attr.name = theForm.elements[i].value;
				attr.value = theForm.elements[i].value;
				products[count] = attr;
				count++;
			}

		}else if(theForm.elements[i].type == "select-one"){
			var attr = new Object();
			attr.name = theForm.elements[i].value;
			attr.value = theForm.elements[i].options[theForm.elements[i].selectedIndex].value;
			products[count] = attr;
			count++;

		} else if(theForm.elements[i].type == "radio") {
			var radios = document.getElementsByName(theForm.elements[i].name);

			if(theForm.elements[i].checked) {
				var attr = new Object();
				attr.name = theForm.elements[i].value;
				attr.value = theForm.elements[i].value;
				products[count] = attr;
				count++;
			}
		}
	}

	return products;

}

function setPriceCallback(data) {
	if(data!=null && data!='') {
		document.getElementById('price').innerHTML=data;
	}

}



function submitFORM(id) {
	var formObj = document.getElementById(id);
	formObj.submit();
}

function toggleTAB(num) {

	for ( var i=0; i<tabCount; i++ ) {
		var tabObj = document.getElementById("tab_"+i);
		if(tabObj) {
			var contentObj = document.getElementById("content_"+i);
			if ( i == num ) {
				tabObj.className = "tab-box tab-selected";
				contentObj.style.display = "block";
			} else {
				tabObj.className = "tab-box";
				contentObj.style.display = "none";
			}
		}
	}
}



function whichBrs() {
var agt=navigator.userAgent.toLowerCase();
if (agt.indexOf("opera") != -1) return 'Opera';
if (agt.indexOf("staroffice") != -1) return 'Star Office';
if (agt.indexOf("webtv") != -1) return 'WebTV';
if (agt.indexOf("beonex") != -1) return 'Beonex';
if (agt.indexOf("chimera") != -1) return 'Chimera';
if (agt.indexOf("netpositive") != -1) return 'NetPositive';
if (agt.indexOf("phoenix") != -1) return 'Phoenix';
if (agt.indexOf("firefox") != -1) return 'Firefox';
if (agt.indexOf("safari") != -1) return 'Safari';
if (agt.indexOf("skipstone") != -1) return 'SkipStone';
if (agt.indexOf("msie") != -1) return 'Internet Explorer';
if (agt.indexOf("netscape") != -1) return 'Netscape';
if (agt.indexOf("mozilla/5.0") != -1) return 'Mozilla';
if (agt.indexOf('\/') != -1) {
if (agt.substr(0,agt.indexOf('\/')) != 'mozilla') {
return navigator.userAgent.substr(0,agt.indexOf('\/'));}
else return 'Netscape';} else if (agt.indexOf(' ') != -1)
return navigator.userAgent.substr(0,agt.indexOf(' '));
else return navigator.userAgent;
}

var browser = whichBrs();

if ( browser == "Firefox" ) {
	document.write("<style>");
	document.write(".input-box2 { padding-top: 6px; height: 20px; }");
	document.write(".login-box2 { padding-top: 3px; height: 17px; }");
	document.write(".qty-box2 { padding-top: 3px; height: 17px; }");
	document.write("</style>");
}















```
