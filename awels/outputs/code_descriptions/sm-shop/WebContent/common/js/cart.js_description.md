# cart.js

## Review

## 1. Summary  

The script implements a **client‑side shopping cart** that is tightly coupled to a legacy HTML form.  
Key responsibilities:

| Component | Responsibility |
|-----------|----------------|
| **Cart state** (`cartLineCount`, `lock`, `quantityInCart`) | Track the number of items, a form‑submission lock, and the total quantity. |
| **DOM manipulation** | Dynamically add/remove rows, update prices, costs, and totals. |
| **AJAX integration** | Calls to a global `AddProduct` object (`addProduct`, `calculate`, `removeProduct`, `removeAttributes`, etc.) to perform server‑side calculations. |
| **UI helpers** | `setErrorMessage`, `stripe`, `setShippingModule`, `addKeyPress` – all update the UI after server responses or user input. |
| **Event binding** | `addBindings` registers click/keypress events on the cart table. |
| **Page init** | `$(document).ready` sets up the cart, calculates totals, and binds the recalculate/post actions. |

The code is a classic jQuery 1.x style script, with inline HTML strings, direct DOM queries and global variable usage. It is designed for an existing server‑side infrastructure that exposes a JavaScript API (`AddProduct`) for cart operations.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

1. **Document Ready**  
   * Resets global flags, stripes the table, hides the “recalculate” button, adds an empty header column, calculates existing lines (`cartLineCount`), adds bindings, calculates totals, and attaches handlers for recalculate, form submit, and post‑items.  

2. **User Interaction**  
   * *Add product* → `addCartLine` → `AddProduct.addProduct` → `setCartLine` (UI).  
   * *Remove product* → `removeFromCart` → `AddProduct.removeProduct` → `calculate`.  
   * *Change quantity / price* → `addKeyPress` → `calculate`.  
   * *Set attributes* → `removeAttribute` → `resetProduct` → `calculate`.  
   * *Shipping selection* → `setShippingModule` → `calculate`.  

3. **Server Round‑Trip**  
   * Every change triggers an `AddProduct.calculate` call with the current cart items and optional shipping.  
   * Server response is handled by `setCalculate`, which updates the UI (prices, costs, subtotal, tax, shipping, credits, etc.) and triggers a final `stripe` call for alternating row styles.  

4. **Form Submission**  
   * The “post items” link sets `lock = 1` and submits the form once the cart has a non‑zero quantity.

5. **Cleanup**  
   * When the cart becomes empty, `calculate` clears the `sku…` cookie and redirects to `emptyCartUrl`.  

### 2.2 Dependencies & Constraints  

* **jQuery** – used for all DOM manipulation and AJAX calls.  
* **jQuery Cookie plugin** – for clearing the `sku…` cookie.  
* **`AddProduct` global object** – defined elsewhere; it performs server‑side cart logic.  
* **Hard‑coded string values** – `shippingText`, `subtotalText`, `taxText`, `totalText`, `customerRequiredText`, etc., are expected to exist in the global scope.  
* **HTML structure** – The script expects a table with specific ids (`#cart`, `#recalculate`, `#shippingMethodId`, etc.) and hidden inputs (`.item input`, `.quantity input`, `.price input`).  
* **Browser assumptions** – The code contains a comment about IE6, indicating support for very old browsers.  

### 2.3 Architecture & Design Choices  

* **Imperative UI updates** – The script manually builds HTML strings and appends them.  
* **Global state** – All cart counters and flags live in global variables.  
* **Event delegation** – Event handlers are attached per row during `addBindings`; no use of jQuery’s event delegation (`on`).  
* **Mix of AJAX & DOM** – Calls to `AddProduct` are asynchronous, but the script often performs DOM updates immediately after.  
* **Legacy “thin server” model** – The server supplies the logic, while the client renders and performs small validations.  

---

## 3. Functions / Methods  

| Function | Purpose | Inputs | Outputs | Side‑Effects |
|----------|---------|--------|---------|--------------|
| `setErrorMessage(message)` | Shows an error banner. | `message` (string) | None | Updates `#ajaxMessage` content & visibility |
| `initCart()` | Clears UI elements & shows checkout button. | None | None | Hides messages, removes rows, shows checkout |
| `removeFromCart(lineId)` | Calls server to remove a line and refreshes cart. | `lineId` | None | Calls `AddProduct.removeProduct`, triggers `calculate()` |
| `removeAttribute(productId,lineId)` | Calls server to remove selected attributes. | `productId`, `lineId` | None | Calls `AddProduct.removeAttributes` |
| `resetProduct(data)` | Restores the price of a line after attribute change. | `data` (object) | None | Sets price field, calls `calculate()` |
| `resetShipping()` | Clears shipping selection UI. | None | None | Resets shipping id, toggles UI |
| `addKeyPress()` | Attaches `keypress`/`blur` events to quantity/price inputs to allow only digits. | None | None | Adds event handlers |
| `addBindings()` | Adds delete, remove‑options handlers to each cart row. | None | None | Creates delete button, binds remove options |
| `calculate()` | Gathers cart data, validates, and calls server. | None | None | Triggers `AddProduct.calculate` |
| `setCalculate(data)` | Callback for `AddProduct.calculate`. Updates the UI with server results. | `data` (object) | None | Inserts totals, tax, shipping, credits, etc. |
| `stripe()` | Alternates row styles for readability. | None | None | Adds `odd/even` classes |
| `setShippingModule(moduleId)` | Sets selected shipping module and recalculates. | `moduleId` | None | Updates hidden input, UI, triggers `calculate()` |
| `setAttributes(data)` | Callback for adding attributes. Updates line UI. | `data` (object) | None | Modifies row, updates price, calls `calculate()` |
| `addCartLine(productId,productName)` | Adds a product to the cart; checks for duplicates. | `productId`, `productName` | None | Calls `AddProduct.addProduct` if not duplicate |
| `setCartLine(data)` | Callback for adding a product. Builds the row. | `data` (object) | None | Creates table row, attaches events, calls `calculate()` |
| `$(document).ready(...)` | Bootstrap the cart on page load. | None | None | Sets up event handlers, calculates totals, initializes UI |

**Reusable / Utility Methods**

* `stripe()` – pure UI helper.
* `setErrorMessage()` – UI helper for messaging.
* `addKeyPress()` – input sanitisation helper.

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| **jQuery (1.x)** | Third‑party | All DOM logic, event handling, AJAX. |
| **jQuery Cookie** | Third‑party | Only used to clear a cookie. |
| **AddProduct** | Custom global API | Provides server‑side cart operations. |
| **thickbox** | Third‑party | Used for modal attribute selection. |
| **TB_iframe** | Custom parameter | For Thickbox iframe integration. |

No native APIs or frameworks are required beyond jQuery.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality Issues  

1. **Global namespace pollution** – Variables (`cartLineCount`, `lock`, `quantityInCart`) and functions are declared in the global scope. This risks naming conflicts and makes unit testing hard.  
2. **Hard‑coded HTML** – The script concatenates large strings of HTML with embedded variable data. This is error‑prone, difficult to read, and potentially vulnerable to XSS if any data is untrusted.  
3. **Duplicate logic** – `addKeyPress` is called multiple times; events may be attached more than once.  
4. **IE6 comment** – The comment suggests IE6 support. If the target browsers are modern, the code can be simplified dramatically.  
5. **No error handling for AJAX failures** – `AddProduct.calculate` etc. may fail; no fallback or user notification.  
6. **Event handling** – Handlers are attached directly to each row. Using event delegation (`$(document).on('click', '.remove', handler)`) would be cleaner and safer.  
7. **Synchronous UI updates** – After calling `AddProduct` the code immediately updates UI assuming success; server validation errors are handled but there is no progress indicator during AJAX calls.  
8. **Missing `use strict`** – Using strict mode would catch accidental globals and other bugs.  
9. **Magic numbers / strings** – Many string literals (e.g., `"td:nth-child(2)"`) and hard‑coded IDs are repeated.  

### 5.2 Security Considerations  

* **XSS** – Server responses are inserted directly into the DOM (`jQuery('#price-'+ data.lineId).val(data.priceText)` etc.). If the server cannot guarantee that these values are safe, an attacker could inject scripts. Escape all user‑supplied data before insertion.  
* **CSRF** – The cart modifications are performed via form submission (`AddProduct` probably uses POST). Ensure that the server side validates CSRF tokens.  

### 5.3 Performance & UX  

* **Repeated DOM queries** – The code uses `jQuery('.item input', this)` etc. inside loops. Cache selectors where possible.  
* **Synchronous `alert`** – Validation errors use `alert`, which is disruptive. Prefer inline error messages.  
* **Blocking UI** – While AJAX is pending, the UI remains responsive; but there is no spinner or loading indicator.  
* **Recalc frequency** – Every keypress triggers a full `calculate`. This may be too aggressive; consider debouncing or only recalc on `blur`.  

### 5.4 Maintainability Enhancements  

1. **Modularize** – Wrap all cart logic in an IIFE or ES6 module to avoid global pollution.  
2. **Template Engine** – Use a lightweight templating system (e.g., Handlebars) for building table rows instead of string concatenation.  
3. **Data Binding** – Adopt a small MVVM approach (e.g., Knockout.js or Vue) to keep the UI in sync with the cart model.  
4. **Configuration Object** – Store constants (`emptyCartUrl`, `shippingUrl`, `attributesUrl`, etc.) in a single config object.  
5. **Unit Tests** – Extract pure functions (e.g., price calculation, row rendering) so they can be unit tested.  

### 5.5 Edge Cases Not Handled  

* **Duplicate product + attributes** – The current check for existing product ignores attribute variations; the same product with different attributes will create duplicate rows.  
* **Negative quantity** – Only numeric validation is present; negative numbers are allowed unless the server rejects them.  
* **Large quantities** – No check on maximum quantity limits.  
* **Shipping removal** – If a user removes shipping after selection, the UI may not correctly reflect the new state.  

### 5.6 Suggested Future Enhancements  

| Feature | Rationale |
|---------|-----------|
| **LocalStorage cart persistence** | Preserve cart across page reloads without server round‑trips. |
| **Optimistic UI** | Update UI immediately on user actions, rollback on server failure. |
| **Accessibility** | Add ARIA roles, proper focus handling for modal dialogs. |
| **Responsive design** | Adapt the table layout for mobile devices. |
| **Internationalization** | Move all text constants into a locale JSON file. |
| **Progressive enhancement** | Use native `<input type="number">` and `pattern` for validation where supported. |

---

### Bottom Line  

The script accomplishes its core purpose—managing a shopping cart with live server‑side calculations—but it is heavily tied to an old jQuery‑centric pattern, uses global state, and contains several maintainability, performance, and security shortcomings. Refactoring to a more modern, modular approach (ES6 modules, template engine, event delegation) would dramatically improve the codebase and make it easier to test, extend, and secure.

## Code Critique



## Code Preview

```javascript

var cartLineCount = 0;
var lock = 0;
var quantityInCart = 0;


// Manages error messages from adding items in the shopping cart
function setErrorMessage(message) {
	
    jQuery('#ajaxMessage').html(message);
    jQuery('#ajaxMessage').css('display', 'block'); 

}

function initCart() {


    jQuery('.ajaxMessage').html('');
    jQuery('#ajaxMessage').css('display', 'none'); 
    jQuery('.duenow').remove();
    jQuery('.subtotal').remove();
    jQuery('.tax').remove();
    jQuery('.shipping').remove();
    jQuery('.recursive').remove();
    jQuery('.total').remove();
    if(jQuery('.href-button-checkout')!=null) {
	jQuery('.href-button-checkout').css('display', 'block'); 
    }


}



function removeFromCart(lineId) {
	AddProduct.removeProduct(lineId);
	calculate();
      stripe();
	return false;
}

function removeAttribute(productId,lineId) {
	AddProduct.removeAttributes(productId,lineId,resetProduct);

}

function resetProduct(data) {


	jQuery('#price-'+ data.lineId).val(data.priceText);
	calculate();
    stripe();


}


function resetShipping() {



	jQuery('#shippingMethodId').val('');
	jQuery('#removeShipping').hide();
      jQuery('#addShipping').show();

}

function addKeyPress() {
		//ie 6 does not like 2 input fields in the same class div invoking the same event, so i switched to blur event instead of change
		jQuery('.quantity input').bind('keypress', function(event) {

    			if (event.charCode && (event.charCode < 48 || event.charCode > 57)) {
     			 	event.preventDefault();
    			}

  		}).blur(function() {
      		//jQuery('#shippingMethodId').val('');
      		return false;//prevent submission
		});

		jQuery('.price input').bind('keypress', function(event) {

    			if (event.charCode && (event.charCode < 48 || event.charCode > 57)) {
     			 	event.preventDefault();
    			}

  		}).change(function() {
			calculate();
			return false;//prevent submission
		});


}


function addBindings() {

  //var count = 0;

  jQuery('#cart tbody tr').each(function() {

   var lineId = jQuery('.item input',this).val();

    if(! jQuery("#remove_" + lineId).length > 0 ) {
		    	$deleteButton = jQuery('<img />').attr({
      	'width': '16',
      	'height': '16',
      	'src': '/shop/common/img/cross.png',
      	'alt': 'X',
      	'title': 'remove from cart',
		'id': 'remove_' + lineId,
      	'class': 'clickable'
    	}).click(function() {//REMOVE PRODUCT
            var id = jQuery(this).parents('tr').find('.item input').val();
      		var pid = '#productid-' + id;
			var productId = jQuery(pid).val();
			jQuery(this).parents('tr').remove();
			removeFromCart(id);
    	});

    	jQuery('<td></td>').insertAfter(jQuery('td:nth-child(2)', this)).append($deleteButton);

    }//end if


    jQuery('#removeOptions-'+lineId).bind('click',function() {//REMOVE OPTIONS

		//get line id
		var id = jQuery(this).parents('tr').find('.item input').val();
		var pid = '#productid-' + id;
		var pname = '#productname-' + id;
		var name = jQuery(pname).val();
		var productId = jQuery(pid).val();
		jQuery(this).parents('tr').find('#productText').html(name);
		jQuery(this).parents('tr').find('#addOptionsLink').show();
		jQuery(this).parents('tr').find('#removeOptionsLink').hide();
		removeAttribute(productId,id);
		return false;

      });


   });



}


//calculates the shopping cart
function calculate() {
    var totalQuantity = 0;
    var totalCost = 0;
    var count = 0;
    lock = 0;
    quantityInCart = totalQuantity;
    var items=new Array()
    jQuery('#cart tbody tr').each(function() {



      	var quantity = jQuery('.quantity input', this).val();
		if(isNaN(quantity)) {
			alert(invalidQuantity);
			jQuery('.quantity input', this).val(0);
		}
		quantity = isNaN(quantity) ? 0 : parseInt(quantity);
		totalQuantity = totalQuantity + quantity;
		var item = new Object();

		//item lineId
		var id=jQuery('.item input', this).val();
		jQuery('#pmessage-'+ id).html('');
		item.lineId = id;
		//item productId
		var pid = '#productid-' + id;
		item.productId = jQuery(pid, this).val();
		item.productQuantity = quantity;
		items[count] = item;
		count++;
    });

	quantityInCart = totalQuantity;
    if(count==0) {
		var cookieKey = 'sku' + jQuery('#merchantId').val();
		jQuery.cookie(cookieKey,null,{ path: '/'});
		window.location = emptyCartUrl;
    }


    //submit
    if(items.length>0) {
    	var shipping = new Object();
		var shippingMethodId = jQuery('#shippingMethodId').val();
		if(shippingMethodId) {
			shipping.shippingMethodId = shippingMethodId;
    	}
		AddProduct.calculate(items,shipping,setCalculate);
    } else {
    	initCart();
    }

  }

function setCalculate(data) {


	initCart();


	if(data.errorMessage) {
		setErrorMessage(data.errorMessage);
		return false;
	}

	var opa = data.orderProducts;
	//check if validation Error

		var errMsg = '';
		var hasError = false
		for (i=0; i<opa.length; i++) {

			if(opa[i].priceErrorMessage) {
				alert(opa[i].priceErrorMessage);
				lock = 2;
				var lineId = opa[i].lineId;


				//set error message and price to 0

				jQuery('#price-'+ opa[i].lineId).val(0);
				jQuery('#pmessage-'+ opa[i].lineId).html('<b><font color=\"red\">*</font></b>');

				var cost = opa[i].costText;
				jQuery('.cost-'+ opa[i].lineId).text(cost);

			}
			if(opa[i].errorMessage) {
				errMsg = errMsg + opa[i].errorMessage + '<br/>';
				hasError = true;

			}

    		}
		if(hasError) {
			setErrorMessage(errMsg);
			if(jQuery('.href-button-checkout')!=null) {
				jQuery('.href-button-checkout').css('display', 'none'); 
			}
		}



	    //display lines
	    for (i=0; i<opa.length; i++) {

			jQuery('#price-'+ opa[i].lineId).val(opa[i].priceText);
			jQuery('.cost-'+ opa[i].lineId).text(opa[i].costText);
    	}

		//due now
		var dueNow = data.otherDueNowAmounts;
		if(dueNow) {
			for (i=0; i<dueNow.length; i++) {
				var dueNowLine = '<tr class=\"duenow\"><td></td><td colspan=\"3\" class=\"desc\">'+dueNow[i].text+'</td><td class=\"cost\">'+dueNow[i].costFormated+'</td></tr>';
				jQuery(dueNowLine).insertBefore(".footerspace");
    			}

		}

		//due now credits
		var credits = data.dueNowCredits;
		if(credits) {
			for (i=0; i<credits.length; i++) {
				var creditLine = '<tr class=\"duenow\"><td></td><td colspan=\"3\" class=\"desc\">'+credits[i].text+'</td><td class=\"cost\"><font color=\"red\">('+credits[i].costFormated+')</font></td></tr>';
				jQuery(creditLine).insertBefore(".footerspace");
    			}

		}

		//subtotal
		var subTotal = '<tr class=\"subtotal\"><td>'+subtotalText+'</td><td colspan=\"3\" class=\"desc\"></td><td class=\"cost\">'+data.oneTimeSubTotalText+'</td></tr>';
		jQuery(subTotal).insertBefore(".footerspace");

		//shipping
		if(data.shipping) {
				var methodId = '';
				var method = '';
				var addShippingDivStyle = 'style=\"display:block\"';
				var removeShippingDivStyle = 'style=\"display:none\"';
				var shippingMessage = shippingText;

				if(data.shippingLine  && data.shippingLine.shippingMethod) {

					shippingMessage = '<b>'+data.shippingLine.shippingMethod+'</b>';
				}
				if(data.shippingLine && data.shippingLine.shippingMethodId) {

					//display remove shipping

					addShippingDivStyle = 'style=\"display:none\"';
					removeShippingDivStyle = 'style=\"display:block\"';
					methodId = data.shippingLine.shippingMethodId;

				} else {
					//display add shipping
					addShippingDivStyle = 'style=\"display:block\"';
					removeShippingDivStyle = 'style=\"display:none\"';
				}
				var addShippingUrl = '<div id=\"addShipping\" ' +addShippingDivStyle+ '><a href=\"' + shippingUrl + '?placeValuesBeforeTB_=savedValues&TB_iframe=true&height=300&width=400&modal=true\" title=\"'+ shippingText +'\" class=\"thickbox\">'+shippingText+'</a></div>';
				var shippingLine = '<tr class=\"shipping\"><td>'+shippingMessage+'</td><td colspan=\"3\" class=\"desc\"><input type=\"hidden\" id=\"shipping-method\" name=\"shipping-method\" value=\"'+method+'\"><input type=\"hidden\" id=\"shippingMethodId\" name=\"shippingMethodId\" value=\"'+methodId+'\"><input type=\"hidden\" name=\"shipping-cost\" value=\"0\"></td><td class=\"cost\"><div id=\"shipping-cost-text\">'+data.shippingTotalText+'</div></td></tr>';
				jQuery(shippingLine).insertBefore(".footerspace");

		}

		//tax
		var tax = data.taxAmounts;
		if(tax) {
			for (i=0; i<tax.length; i++) {
				var taxLine = '';
				if(i==0) {
					taxLine = '<tr class=\"tax\"><td>'+taxText+'</td><td colspan=\"3\" class=\"desc\">'+tax[i].text+'</td><td class=\"cost\">'+tax[i].costFormated+'</td></tr>';
				} else {
					taxLine = '<tr class=\"tax\"><td></td><td colspan=\"3\" class=\"desc\">'+tax[i].text+'</td><td class=\"cost\">'+tax[i].costFormated+'</td></tr>';
				}
				jQuery(taxLine).insertBefore(".footerspace");
    		}
		}

		//total
		var total = '<tr class=\"total\"><td>'+totalText+'</td><td colspan=\"3\" class=\"desc\"></td><td class=\"cost\" nowrap>'+data.totalText+'</td></tr>';
		jQuery(total).insertBefore(".footerspace");

		//recursive
		var recur = data.recursiveAmounts;
		if(recur) {
			var recursiveTitle = '<tr class=\"recursive\"><td></td><td colspan=\"3\" class=\"desc\"></td><td class=\"cost\"></td></tr><tr class=\"recursive\"><td>'+recursive+'</td><td colspan=\"3\" class=\"desc\"></td><td class=\"cost\"></td></tr>';
			jQuery(recursiveTitle).insertBefore(".footerspace");
			for (i=0; i<recur.length; i++) {

				var recursiveLine = '<tr class=\"recursive\"><td></td><td colspan=\"3\" class=\"desc\">'+recur[i].text+'</td><td class=\"cost\">'+recur[i].costFormated+'</td></tr>';

				jQuery(recursiveLine).insertBefore(".footerspace");
			}
		}

		//recursive credits
		var recurCredits = data.recursiveCredits;
		if(recurCredits ) {
			for (i=0; i<recurCredits.length; i++) {

				var recursiveCreditLine = '<tr class=\"recursive\"><td></td><td colspan=\"3\" class=\"desc\">'+recurCredits[i].text+'</td><td class=\"cost\"><font color=\"red\">('+recurCredits[i].costFormated+')</font></td></tr>';

				jQuery(recursiveCreditLine).insertBefore(".footerspace");
			}
		}




}

function stripe() {
    jQuery('#cart tbody tr:visible:even').removeClass('odd').addClass('even');
    jQuery('#cart tbody tr:visible:odd').removeClass('even').addClass('odd');
}

function setShippingModule(moduleId) {

	jQuery('#shippingMethodId').val(moduleId);
	jQuery('#removeShipping').show();
    	jQuery('#addShipping').hide();

	calculate();
}


function setAttributes(data) {




	//initCart();

	if(data.errorMessage) {
		setErrorMessage(data.errorMessage);
		return false;
	}

	var lineId = data.lineId;
	var search = "cartlineid-" + lineId;

	var pname = '#productname-' + lineId;
	var name = jQuery(pname).val();

	var attrs = '<input type=\"hidden\" id="attributes-'+ lineId + '\" name=\"attributes-\"'+ lineId  + '\" value=\"OPA-'+ lineId +'\">';
	var cartId = '#cartlineid-' + lineId;
	jQuery(cartId).parents('tr').find('#productText').html(name + " " + attrs + "<br>" + data.attributesLine);

	jQuery('#price-'+ lineId).val(data.priceText);
	var cost = data.priceText * data.productQuantity;
	jQuery('.cost-'+ data.lineId).text('$' + cost.toFixed(2));

	jQuery(cartId).parents('tr').find('#addOptionsLink').hide();
	jQuery(cartId).parents('tr').find('#removeOptionsLink').show();

	calculate();


}

function addCartLine(productId,productName) {


   //initCart();


   if(document.getElementById('customer.customerId').value==0) {
		setErrorMessage(customerRequiredText);
		return false;
   }



   var productexist=0;
   jQuery('#cart tbody tr').each(function() {


		//get the product id
		var id=jQuery('.item input', this).val();
		var pid = '#productid-' + id;
		var product = jQuery(pid, this).val();
		if(product==productId) {
			//if no attributes
			var attrid = '#attributes-' + id;
			var value = jQuery(attrid, this).val();
			if(value) {
				//do nothing for now

			} else {
				var quantity = parseInt(jQuery('.quantity input', this).val());
				document.getElementById('quantity-' + id).value=quantity+1;

				calculate();
				productexist=1;
				return;

			}
		}

	});

	if(productexist==0){
		var newLineCount = cartLineCount+1;
		AddProduct.addProduct(productId,newLineCount ,setCartLine);
	}

}

//add a line to the shopping cart
function setCartLine(data) {



	if(data.errorMessage) {
		//document.getElementById('ajaxMessage').innerHTML='<div class=\"icon-error\">'+data.errorMessage+'</div>';
		setErrorMessage(data.errorMessage);
		return false;
	}

	cartLineCount ++;


	//properties
	var prop = "";
	if(data.attributes) {
		var addPropUrl = "<div id=\"addOptionsLink\"><a href=\"" + attributesUrl + data.productId + "&lineId=" +data.lineId+ "&placeValuesBeforeTB_=savedValues&TB_iframe=true&height=300&width=400&modal=true\" title=\""+ attributesText +"\" class=\"thickbox\">"+attributesText+"</a></div>";
		var removePropUrl = "<div id=\"removeOptionsLink\" style=\"display:none\"><a href=\"\" id=\"removeOptions-"+data.lineId+"\">"+removeAttributesText+"</a></div>";
		prop="<br>" + addPropUrl + " " + removePropUrl;
	}
	opening = "<tr><td class=\"item\"><input type=\"hidden\" name =\"cartlineid-"+data.lineId+"\" id=\"cartlineid-" +data.lineId+ "\" value=\"" + data.lineId+ "\"><input type=\"hidden\" name=\"ids["+data.lineId+ "]\" value=\""+data.lineId+"\"> <input type=\"hidden\" name=\"productid-"+data.lineId+"\" id=\"productid-" +data.lineId+ "\" value=\"" + data.productId + "\"><input type=\"hidden\" name=\"productname-"+data.lineId+"\" id=\"productname-" +data.lineId+ "\" value=\"" + data.productName + "\"><div id=\"productText\">" + data.productName + " </div> " + prop + "</td><td class=\"quantity\">";
	tfield="<div id=\"qmessage-"+data.lineId+"\"></div><input type=\"text\" name=\"quantity-" +data.lineId+ "\" value=\"1\" id=\"quantity-" +data.lineId+ "\" maxlength=\"3\" />";
	closing = "</td><td class=\"price\"><div id=\"pmessage-"+data.lineId+"\"></div><input type=\"text\" name=\"price-" +data.lineId+ "\" value=\"" + data.priceText + "\" id=\"price-" +data.lineId+ "\" size=\"5\" maxlength=\"5\" /></td><td align=\"right\" class=\"cost-"+data.lineId+"\">" + data.priceFormated + "</td></tr>";
	line = opening + tfield + closing;
    	jQuery(line).appendTo( "#cart" );
	//alert(line);
	stripe();
	addKeyPress();
	addBindings();
	calculate();
	tb_init('a.thickbox,area.thickbox,input.thickbox');
	return false;
}



/***************************************
   init shopping cart
****************************************/

jQuery(document).ready(function() {

  lock = 0;
  cartLineCount = 0;

  stripe();

  jQuery('#recalculate').hide();


  jQuery('<th>&nbsp;</th>').insertAfter('#cart thead th:nth-child(2)');


    var calculateLines = function() {
    var count = 0;

  	jQuery('#cart tbody tr').each(function() {

    	count++;

   	});
	return count;
  };




  //invoked during display
  cartLineCount  = calculateLines();
  addBindings();
  calculate();

  addKeyPress();


 jQuery("a#recalculateCart")
	.click(function(){
		if(lock==0) {
			//lock=0;
			calculate();
			return false;
		} else {
			return true;
		}
	});//prevent form submission when recalculate button is pressed

 jQuery("form:first")
	.submit(function(){
		if(lock==0) {
			//lock=0;
			calculate();
			return false;
		} else {
			return true;
		}
	});//prevent form submission when enter key is pressed


   jQuery("a#postItems")
	.click(function(){
		if(lock==0) {//no error
			lock =1;//ready to submit
			if(quantityInCart>0) {
				jQuery("form:first").submit();
			}
		}
	});//prevent form submission when enter key is pressed or quantity is 0


  jQuery('<td>&nbsp;</td>').insertAfter('#cart tfoot td:nth-child(2)');






});


```
