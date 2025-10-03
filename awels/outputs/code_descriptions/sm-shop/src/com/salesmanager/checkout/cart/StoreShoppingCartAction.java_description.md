# StoreShoppingCartAction.java

## Review

## 1. Summary

**Purpose**  
`StoreShoppingCartAction` is a Struts‑style action class that handles two different checkout entry points for an e‑commerce platform:

| Method | Origin | Main Responsibility |
|--------|--------|---------------------|
| `checkoutLocal()` | The “shop/catalog” web‑app (same application) | Initializes a shopping cart from the session, validates it and forwards to the standard checkout workflow. |
| `checkoutRemote()` | A “remote” shop (`sm‑shop`) that sends its cart data via query string parameters | Parses URL parameters (`merchantId`, locale, product IDs, quantities, attributes), builds a list of `CheckoutParams`, and forwards the cart to the common checkout logic. |

**Key Components**

* **`ShoppingCartAction`** – superclass providing common checkout utilities (`prepareStore`, `assembleShoppingCartItems`, etc.).
* **`SessionUtil`** – helper class for manipulating session‑bound objects (cart, store, token).
* **`CheckoutParams`** – DTO that holds product ID, quantity, attribute IDs, and attribute values.
* **`MerchantStore` & `ShoppingCart`** – domain entities representing the store and the current cart.
* **`Constants`** – holds request‑parameter names.

The class does not use any modern frameworks (no Spring, no CDI); it relies on legacy Struts‑1‑style patterns.

---

## 2. Detailed Description

### Flow of Execution

#### `checkoutLocal()`
1. **Cart Reset** – clears any previous cart from the session (`SessionUtil.cleanCart`).
2. **Session Token** – sets a token to validate the session.
3. **Retrieve Cart** – obtains the current mini‑shopping cart. If missing, an action error is added and the “landing” view is returned.
4. **Store Setup** – fetches the `MerchantStore` and calls `prepareStore(merchantId)` to set up store‑specific configuration.
5. **Comitted Check** – if a prior checkout was already committed, the cart is cleaned again.
6. **Product Collection** – pulls the products from the cart and delegates the heavy lifting to `assembleShoppingCartItems(productsCollection)`.
7. **Return** – on success returns `SUCCESS`.

#### `checkoutRemote()`
1. **Session Token** – similar to local checkout.
2. **Cleanup** – clears any existing cart (`cleanShoppingCart()`).
3. **Parameter Parsing** – loops through the request parameters, looking for:
   * `merchantId` – to load the correct store.
   * `productId_<idx>` – each product, along with optional `quantity_<idx>` and `attributeId_<idx>`, `attributeValue_<attrId>`.
4. **Parameter Validation** – if required parameters are missing, an error is recorded and the method returns `GENERICERROR`.
5. **Build CheckoutParams** – for each product, a `CheckoutParams` instance is created and populated with its ID, quantity, attributes, and attribute values.
6. **Prepare Store** – `prepareStore(merchantId)` is called once all parameters are parsed.
7. **Assemble Items** – the list of `CheckoutParams` is passed to `assembleItems(prds)` which presumably creates a real `ShoppingCart`.
8. **Return** – `SUCCESS` on success, otherwise `GENERICERROR`.

### Assumptions & Constraints

* The request must contain at least one `productId_*` parameter.
* All numeric parameters are sent as strings that can be parsed into `int`/`long`.
* The request parameters are named according to constants defined in `Constants`.
* The action is executed in a servlet environment where session attributes are available.
* The action does not provide thread‑safety guarantees beyond the container’s request/response lifecycle.

### Design Choices

* **Legacy Struts Style** – Action methods return `String` view names (`SUCCESS`, `"GENERICERROR"`, `"landing"`). No annotations or dependency injection.
* **Raw Types** – Collections are declared without generics (`Map products = new HashMap();`), making the code vulnerable to `ClassCastException` and reducing readability.
* **Manual Parameter Parsing** – The `checkoutRemote` method manually iterates over parameter names and uses string manipulation to identify indices, which is brittle and hard to maintain.
* **Centralized Checkout Logic** – The heavy lifting is delegated to superclass methods (`assembleShoppingCartItems`, `assembleItems`), keeping this class thin but still coupling it tightly to the superclass’s implementation.

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `checkoutLocal()` | Handles in‑app checkout: clears cart, loads session cart, sets up store, assembles cart items. | None | `String` view name (`SUCCESS` or `"landing"`) | Modifies session attributes (cart, token); logs errors. |
| `checkoutRemote()` | Handles remote checkout via query‑string parameters: parses parameters, builds `CheckoutParams`, assembles cart items. | None | `String` view name (`SUCCESS` or `"GENERICERROR"`) | Modifies session attributes (cart, token); logs errors; may set technical error message. |

### Helper Methods (inherited or used)

* `SessionUtil.cleanCart(HttpServletRequest)` – removes the cart from session.
* `SessionUtil.setToken(HttpServletRequest)` – sets a session token for validation.
* `SessionUtil.getMiniShoppingCart(HttpServletRequest)` – retrieves the current cart.
* `SessionUtil.getMerchantStore(HttpServletRequest)` – retrieves the current store.
* `prepareStore(int merchantId)` – loads store configuration (inherited).
* `assembleShoppingCartItems(Collection products)` – builds cart items for local checkout (inherited).
* `assembleItems(List<CheckoutParams> prds)` – builds cart items for remote checkout (inherited).

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For `StringUtils.isBlank`. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.checkout.CheckoutParams` | Application | DTO for product selections. |
| `com.salesmanager.checkout.web.Constants` | Application | Holds parameter names. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Application | Domain entity. |
| `com.salesmanager.core.entity.orders.ShoppingCart` | Application | Domain entity. |
| `com.salesmanager.core.util.www.SessionUtil` | Application | Session helper. |
| `ShoppingCartAction` (superclass) | Application | Provides shared checkout logic. |

All dependencies are library‑style; no framework‑specific annotations are used.

---

## 5. Additional Notes & Recommendations

### 5.1 Code Quality

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw collections** (`Map products = new HashMap();`) | Type‑safety, readability | Use generics (`Map<String, CheckoutParams> products = new HashMap<>();`). |
| **Missing `@Override` annotations** | May lead to accidental method hiding | Add `@Override` to `checkoutLocal` and `checkoutRemote`. |
| **Hard‑coded string constants** (`"GENERICERROR"`, `"landing"`) | Magic strings | Declare them as `private static final String`. |
| **No input validation beyond parsing** | Potential `NumberFormatException`, `NullPointerException` | Validate each parameter and provide user‑friendly error messages. |
| **Exception handling** – catching generic `Exception` and only logging | Swallows all errors, no recovery | Narrow exception types; add user‑friendly error messages. |
| **Repeated `SessionUtil.setToken`** – unclear purpose | Ambiguous semantics | Clarify token handling or remove if unused. |
| **Use of `StringUtils.isBlank`** – only on quantity; other fields not checked | Missing checks for other fields | Validate all required parameters. |
| **`prepareStore` called twice** (once after merchantId parse, again at the end) | Duplicate work | Remove the first call or guard it. |
| **Potential race condition** – reading request parameters into a map while iterating over it could cause concurrent modification if the underlying map is mutated (unlikely in servlet environment but still) | Minor | Use `parameters.keySet()` directly or `entrySet()` to avoid modification. |
| **Hard‑coded default merchant ID** (`Constants.DEFAULT_MERCHANT_ID`) | Magic number | Move to config or property file. |
| **Logging `e` without context** – `log.error(e)` only logs stack trace | Hard to debug | Include context message. |
| **`cleanShoppingCart()`** is not defined in the snippet – ensure it exists and cleans session correctly. | Undefined method | Verify implementation. |
| **No separation of concerns** – parameter parsing, validation, and cart assembly all in one method | Hard to test | Extract parsing into a dedicated helper class/method. |

### 5.2 Security & Robustness

* **CSRF Protection** – The `setToken` call suggests an attempt to validate session, but there is no mention of CSRF tokens for the remote checkout. Consider adding a token in the request that the action validates.
* **Input Sanitization** – While numeric parsing protects against injection for IDs, attribute values are stored as raw strings; ensure downstream code sanitizes them.
* **Session Hijacking** – The action should validate that the session is associated with the correct user (e.g., a logged‑in customer). If anonymous checkout is allowed, ensure that the session cannot be reused maliciously.
* **Timeouts & Cleanup** – `SessionUtil.cleanCart` is called, but if the cart contains large objects, consider serializing or persisting it elsewhere to avoid memory bloat.

### 5.3 Performance

* **Iterating over request parameters** – For a small number of products this is fine, but for large carts the nested loops may become costly. A more structured request format (e.g., JSON body) could simplify parsing.
* **Multiple Map lookups** – Each attribute value requires a new `Map` lookup; consider pre‑building a structure per product.

### 5.4 Future Enhancements

1. **Refactor to a modern framework** – Move to Spring MVC or a JAX‑RS resource. This would allow dependency injection, better request mapping, and clearer validation annotations.
2. **Unit‑testing** – Extract the parameter parsing into pure functions that can be unit‑tested with mock data.
3. **DTO Validation** – Use bean validation (`@NotNull`, `@Min`, etc.) for `CheckoutParams`.
4. **Internationalization** – Extract all message keys and provide localized error messages.
5. **Logging improvements** – Use MDC to log request IDs and user IDs for traceability.
6. **Error handling** – Return a structured JSON error response in the remote case rather than a generic view name.

---

**Verdict**  
The class fulfills its basic purpose and is likely functional within its legacy Struts environment. However, it suffers from several code‑quality and maintainability issues: raw types, lack of validation, duplicated logic, and minimal error handling. Refactoring to use generics, separating concerns, and improving exception handling would greatly enhance readability, robustness, and future extensibility.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.checkout.cart;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutParams;
import com.salesmanager.checkout.web.Constants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.util.www.SessionUtil;

public class StoreShoppingCartAction extends ShoppingCartAction {

	private Logger log = Logger.getLogger(StoreShoppingCartAction.class);

	/**
	 * Invoked from shop/catalog local web application, need to run in the same
	 * web application
	 * 
	 * @return
	 */
	public String checkoutLocal() {

		try {

			// cleanup actual shopping cart
			SessionUtil.cleanCart(super.getServletRequest());

			SessionUtil.setToken(super.getServletRequest());// need this to
															// check a valid
															// session

			ShoppingCart cart = SessionUtil.getMiniShoppingCart(super
					.getServletRequest());
			if (cart == null) {
				addActionError(getText("message.cart.emptycart"));
				return "landing";
			}

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());
			super.prepareStore(store.getMerchantId());

			// prepareLocale();

			if (SessionUtil.isComited(getServletRequest())) {
				SessionUtil.cleanCart(getServletRequest());
			}

			Map products = new HashMap();

			Collection productsCollection = cart.getProducts();

			super.assembleShoppingCartItems(productsCollection);

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	/**
	 * When invoked from sm-shop using url parameters Valid parameters are: -
	 * merchantId - locale - productId_<COUNT> - attributeId_<COUNT> -
	 * quantity_<COUNT>
	 * 
	 * @return
	 */
	public String checkoutRemote() {

		try {

			SessionUtil.setToken(super.getServletRequest());// need this to
															// check a valid
															// session

			// prepareLocale();

			cleanShoppingCart();

			Locale locale = null;
			int merchantId = -1;
			String requestLocale = null;

			Map products = new HashMap();

			Map parameters = super.getServletRequest().getParameterMap();

			if (parameters == null && parameters.size() == 0) {
				addActionError(getText("error.validation.parameters.missing"));
				return "GENERICERROR";
			}

			Iterator i = parameters.keySet().iterator();

			MerchantStore mStore = null;

			while (i.hasNext()) {

				String parameterName = (String) i.next();
				// handle merchant, locale and productId
				if (parameterName.equalsIgnoreCase(Constants.MERCHANT_ID_PARAM)) {
					try {
						String[] sMerchant = (String[]) parameters
								.get(parameterName);
						merchantId = Integer.parseInt(sMerchant[0]);
						prepareStore(merchantId);
					} catch (Exception e) {
						log.error("Cannot parse merchantId " + parameterName);
						addActionError(getText("error.merchant.unavailable",
								new String[] { parameterName }));
						return "GENERICERROR";
					}
				} else if (parameterName.startsWith(Constants.PRODUCT_ID_PARAM)) {
					if (parameterName.contains("_")) {
						int idx = parameterName.indexOf("_");
						String keyId = parameterName.substring(idx + 1,
								parameterName.length());
						CheckoutParams p = (CheckoutParams) products.get(keyId);
						if (p == null) {
							p = new CheckoutParams();
							// String parameter =
							// parameterName.substring(0,idx);
							String[] parameter = (String[]) parameters
									.get(parameterName);
							long productId = -1;
							try {
								productId = Long.parseLong(parameter[0]);
							} catch (Exception e) {
								log
										.error("Cannot parse productId "
												+ parameter);
								continue;
							}
							products.put(keyId, p);
							p.setProductId(productId);
						}

						// get quantity
						String[] sQuantity = (String[]) parameters
								.get(Constants.QUANTITY_PARAM + "_" + keyId);
						if (sQuantity != null
								&& !StringUtils.isBlank(sQuantity[0])) {
							try {
								int quantity = Integer.parseInt(sQuantity[0]);
								p.setQty(quantity);
							} catch (Exception e) {
								log.error("Cannot parse quantity " + sQuantity);
							}
						}

						// get attributes
						String[] attributesObject = (String[]) parameters
								.get(Constants.ATTRIBUTE_PARAM + "_" + keyId);
						if (attributesObject != null
								&& attributesObject.length > 0) {
							List attrs = p.getAttributeId();
							if (attrs == null) {
								attrs = new ArrayList();
								p.setAttributeId(attrs);
							}

							for (int attrCount = 0; attrCount < attributesObject.length; attrCount++) {

								String attributeId = attributesObject[attrCount];

								try {
									Long attr = Long.valueOf(attributeId);
									attrs.add(attr);

								} catch (Exception e) {
									log.error("Attribute " + attributeId
											+ " can't be parsed to a Long");
									continue;
								}

								String[] attributesValuesObject = (String[]) parameters
										.get(Constants.ATTRIBUTE_VALUE_PARAM
												+ "_" + attributeId);

								if (attributesValuesObject != null
										&& attributesValuesObject.length > 0) {

									Map attrValues = p.getAttributeValue();
									if (attrValues == null) {
										attrValues = new HashMap();
										p.setAttributeValue(attrValues);
									}
									// store any string value contained in the
									// query string
									for (int attrValCount = 0; attrValCount < attributesValuesObject.length; attrValCount++) {

										String attributeValue = attributesValuesObject[attrValCount];
										attrValues.put(new Long(attributeId),
												attributeValue);

									}

								}
							}
						}
					}
				}

			}

			if (merchantId == -1) {
				merchantId = com.salesmanager.core.constants.Constants.DEFAULT_MERCHANT_ID;
			}

			prepareStore(merchantId);

			if (products.size() == 0) {
				addActionError(getText("error.validation.parameter.missing",
						new String[] { PRODUCT_ID_PARAM }));
				return "GENERICERROR";
			}

			List prds = new ArrayList();

			Iterator ii = products.keySet().iterator();
			while (ii.hasNext()) {
				String key = (String) ii.next();
				CheckoutParams p = (CheckoutParams) products.get(key);
				prds.add(p);
			}

			super.assembleItems(prds);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;

	}

}



```
