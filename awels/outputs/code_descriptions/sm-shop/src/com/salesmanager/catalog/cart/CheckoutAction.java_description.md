# CheckoutAction.java

## Review

## 1. Summary

**Purpose**  
`CheckoutAction` is a Struts‑2 style action that prepares a remote checkout request.  
It examines the shopping cart stored in the user session, builds a collection of
`NameValuePair` objects (product IDs, quantities, attributes, etc.), and
provides a target URL (`postUrl`) for an external payment/checkout service.

**Key components**

| Component | Role |
|-----------|------|
| `checkout()` | Main action method – constructs the NVP list and decides whether to use local or remote checkout. |
| `nvps` | Collection of `NameValuePair` objects that will be posted to the remote checkout. |
| `postUrl` | The URL of the remote checkout endpoint, derived from the merchant store. |
| `SessionUtil` | Helper that fetches `MerchantStore` and `ShoppingCart` objects from the session. |
| `ReferenceUtil.buildCheckoutToCartUrl()` | Generates the remote checkout URL. |
| `PropertiesUtil` | Reads configuration (`core.catalog.checkout.type`) to decide the checkout strategy. |

**Design patterns & frameworks**

* **Action pattern** – extends `SalesManagerBaseAction` (likely a Struts‑2 action).  
* **Command / Data Transfer Object** – `NameValuePair` acts as a simple DTO for the NVP list.  
* **Factory / Utility** – `SessionUtil`, `ReferenceUtil`, and `PropertiesUtil` are stateless helpers.

No sophisticated frameworks are used beyond the standard Java EE stack and Apache Commons.

---

## 2. Detailed Description

### Flow of execution

1. **Checkout type decision**  
   The method first checks a configuration property `core.catalog.checkout.type`.  
   If the value equals `CatalogConstants.LOCAL_CART`, it immediately returns `"checkoutLocal"` – no remote processing is performed.

2. **Session data retrieval**  
   - `MerchantStore store` is obtained from the HTTP session.  
   - `ShoppingCart cart` is fetched from the session as well.  
   If the cart is `null`, an action error is set and the method returns `"landing"`.

3. **Post URL construction**  
   `ReferenceUtil.buildCheckoutToCartUrl(store)` is called to produce the remote checkout URL, stored in `postUrl`.

4. **NVP construction**  
   * A `NameValuePair` containing the merchant ID is added first.  
   * The method then iterates over each `ShoppingCartProduct` in the cart:  
     * Adds the product ID as `productId_<id>`.  
     * If quantity > 1, adds `qty_<id>`.  
     * If the product has attributes, iterates over them:  
       * Adds `attributeId_<productId>` for each attribute.  
       * If a textual value is present, adds `attributeValue_<attributeId>`.  

5. **Exception handling**  
   Any exception is logged, a technical message is set via the base action, and `"GENERICERROR"` is returned.

6. **Return**  
   If everything succeeds, `"checkoutRemote"` is returned.

### Assumptions & constraints

| Assumption | Impact |
|------------|--------|
| `SessionUtil.getMerchantStore()` never returns `null`. | If it does, a NPE will occur when accessing `store.getMerchantId()` or `store.getMerchantId()`. |
| `cart.getProducts()` never returns `null`. | A `NullPointerException` would surface. |
| All `NameValuePair` keys are stringified using simple concatenation. | No validation that keys are unique or follow any schema. |
| The code runs in a single-threaded request context. | No thread‑safety concerns, but the use of raw types is unsafe. |
| No localization for messages beyond the one `getText("message.cart.emptycart")`. | Only one localized message is supported. |

### Architecture & design choices

* **Use of raw types** – The code predates generics (or simply ignores them). This leads to unchecked warnings and potential `ClassCastException`s.  
* **Flat iteration** – Simple `Iterator` loops are used rather than enhanced for‑loops, again reflecting older Java idioms.  
* **Mutable state** – The action keeps `nvps` and `postUrl` as instance fields, which is typical for Struts actions but may be confusing for unit testing.  
* **Hardcoded keys** – Key names are constructed via string concatenation rather than constants or a builder pattern.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `String checkout()` | Main action logic; prepares the NVP list and decides checkout mode. | None | `"checkoutLocal"`, `"checkoutRemote"`, `"landing"`, or `"GENERICERROR"` | Sets `postUrl`, populates `nvps`, adds action errors, logs exceptions. |
| `String getPostUrl()` | Getter for the remote checkout URL. | None | URL string | None |
| `void setPostUrl(String)` | Setter for the remote checkout URL. | URL string | None | Sets internal field. |
| `Collection getNvps()` | Getter for the NVP collection. | None | Collection of `NameValuePair` | None |
| `void setNvps(Collection)` | Setter for the NVP collection. | Collection | None | Sets internal field. |

> **Reusable utilities** – The code directly uses `NameValuePair` objects; a small helper method like `addNVP(String key, String value)` would reduce duplication.

---

## 4. Dependencies

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Used for blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Standard logging. |
| `com.salesmanager.common.SalesManagerBaseAction` | Project | Base Struts action. |
| `com.salesmanager.core.constants.CatalogConstants` | Project | Holds `LOCAL_CART`. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Project | Domain entity. |
| `com.salesmanager.core.entity.orders.ShoppingCart` | Project | Domain entity. |
| `com.salesmanager.core.entity.orders.ShoppingCartProduct` | Project | Domain entity. |
| `com.salesmanager.core.entity.orders.ShoppingCartProductAttribute` | Project | Domain entity. |
| `com.salesmanager.core.util.NameValuePair` | Project | Simple DTO. |
| `com.salesmanager.core.util.PropertiesUtil` | Project | Reads config. |
| `com.salesmanager.core.util.ReferenceUtil` | Project | Builds URLs. |
| `com.salesmanager.core.util.www.SessionUtil` | Project | Session helpers. |

All dependencies are internal to the `salesmanager` project or well‑known third‑party libraries. No platform‑specific APIs are used.

---

## 5. Additional Notes

### Edge cases & limitations

1. **Null handling** – The code assumes `store` and `cart` are non‑null after retrieval. Defensive checks (e.g., `if (store == null) { /* handle error */ }`) would make it more robust.
2. **Empty product list** – If `cart.getProducts()` is empty but not null, the method will still return `"checkoutRemote"` with only the merchant NVP. The downstream service might reject an empty order.
3. **Duplicate keys** – The NVP keys (`attributeId_<productId>`) may collide if the same attribute ID appears for multiple products. Using unique identifiers or namespacing would avoid ambiguity.
4. **Locale support** – Only one error message is localized. Consider injecting a message source for all user‑facing text.
5. **Generics** – Switching to `List<NameValuePair>` and enhanced for‑loops would eliminate unchecked warnings and simplify the code.
6. **Performance** – Building large NVP lists via `ArrayList` is fine, but if the cart is large, consider streaming or batching.
7. **Testability** – The use of static `SessionUtil` calls makes unit testing harder. Dependency injection or a wrapper could help.

### Suggested improvements

| Category | Recommendation |
|----------|----------------|
| **Type safety** | Replace raw `Collection` and `Iterator` with generics (`List<NameValuePair>`, `for-each` loops). |
| **Encapsulation** | Move NVP construction into a helper method (e.g., `buildNVPs(ShoppingCart cart)`) to keep `checkout()` focused. |
| **Null checks** | Guard against `null` `store`, `cart`, and `cart.getProducts()`. |
| **Constants** | Extract string key prefixes (`productId_`, `qty_`, `attributeId_`, `attributeValue_`) into static final constants. |
| **Error handling** | Provide more descriptive error messages (e.g., `message.cart.empty`) and consider returning a dedicated error view. |
| **Testing** | Refactor to inject dependencies (e.g., `SessionUtil`, `ReferenceUtil`) for easier mocking. |
| **Logging** | Log the constructed NVP list at debug level for easier debugging of remote checkout calls. |
| **Performance** | If the cart can contain thousands of items, consider using a `LinkedHashMap` or a `Map<String, String>` for NVPs to avoid duplicate keys and enable easier lookups. |

### Future extensions

* **Payment gateway integration** – The class could evolve to support multiple remote checkout providers (e.g., PayPal, Stripe) by abstracting the URL and NVP construction into provider-specific strategies.
* **Cart validation** – Before sending data to the remote gateway, validate stock, pricing, and promotions server‑side.
* **Asynchronous processing** – Offload remote checkout calls to a background job, improving responsiveness for the user.
* **Security** – Sign or encrypt the NVP payload to prevent tampering during transmission.

---

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

package com.salesmanager.catalog.cart;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.orders.ShoppingCartProductAttribute;
import com.salesmanager.core.util.NameValuePair;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class CheckoutAction extends SalesManagerBaseAction {

	private static Logger logger = Logger.getLogger(CheckoutAction.class);

	private Collection nvps = null;

	private String postUrl = null;

	/**
	 * When invoking shopping cart using url parameters
	 * 
	 * @return
	 */
	public String checkout() {

		try {

			// if the system uses remote or local checkout
			String cartType = PropertiesUtil.getConfiguration().getString(
					"core.catalog.checkout.type");
			if (cartType != null
					&& cartType.equalsIgnoreCase(CatalogConstants.LOCAL_CART)) {
				return "checkoutLocal";
			}

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			nvps = new ArrayList();

			ShoppingCart cart = SessionUtil.getMiniShoppingCart(super
					.getServletRequest());

			if (cart == null) {
				addActionError(getText("message.cart.emptycart"));
				return "landing";
			}

			postUrl = ReferenceUtil.buildCheckoutToCartUrl(store);

			NameValuePair merchantNvp = new NameValuePair();
			merchantNvp.setKey("merchantId");
			merchantNvp.setValue(String.valueOf(store.getMerchantId()));

			nvps.add(merchantNvp);

			Collection products = cart.getProducts();
			Iterator i = products.iterator();
			NameValuePair nvp = null;
			while (i.hasNext()) {
				ShoppingCartProduct product = (ShoppingCartProduct) i.next();
				nvp = new NameValuePair();
				nvp.setKey("productId_" + product.getProductId());
				nvp.setValue(String.valueOf(product.getProductId()));
				nvps.add(nvp);
				if (product.getQuantity() > 1) {
					nvp = new NameValuePair();
					nvp.setKey("qty_" + product.getProductId());
					nvp.setValue(String.valueOf(product.getQuantity()));
					nvps.add(nvp);
				}
				if (product.getAttributes() != null
						&& product.getAttributes().size() > 0) {
					List attrs = product.getAttributes();
					Iterator it = attrs.iterator();
					while (it.hasNext()) {
						ShoppingCartProductAttribute scpa = (ShoppingCartProductAttribute) it
								.next();
						nvp = new NameValuePair();
						nvp.setKey("attributeId_" + product.getProductId());
						nvp.setValue(String.valueOf(scpa.getAttributeId()));
						nvps.add(nvp);
						if (!StringUtils.isBlank(scpa.getTextValue())) {
							nvp = new NameValuePair();
							nvp.setKey("attributeValue_"
									+ scpa.getAttributeId());
							nvp.setValue(scpa.getAttributeValue());
							nvps.add(nvp);
						}
					}
				}
			}

		} catch (Exception e) {
			logger.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return "checkoutRemote";

	}

	public String getPostUrl() {
		return postUrl;
	}

	public void setPostUrl(String postUrl) {
		this.postUrl = postUrl;
	}

	public Collection getNvps() {
		return nvps;
	}

	public void setNvps(Collection nvps) {
		this.nvps = nvps;
	}

}



```
