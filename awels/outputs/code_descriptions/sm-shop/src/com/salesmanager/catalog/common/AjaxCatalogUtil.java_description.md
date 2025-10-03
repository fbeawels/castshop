# AjaxCatalogUtil.java

## Review

## 1. Summary  
**Purpose & Functionality**  
`AjaxCatalogUtil` is a stateless helper that exposes common AJAX‑driven actions for a storefront mini‑shopping‑cart:  

* **Add** a product (with or without attributes) to the cart.  
* **Remove** a product from the cart.  
* **Calculate** a product’s price based on selected attributes.  
* Persist the cart in an HTTP cookie (currently JSON‑serialized) and in the user session.

**Key Components**  

| Component | Role |
|-----------|------|
| `ShoppingCart` | Domain model for the mini‑cart held in the session and cookie. |
| `CatalogService` | Repository layer to fetch products & attributes. |
| `MiniShoppingCartUtil` | Helper to compute totals & formatting. |
| `MiniShoppingCartSerializationUtil` | JSON (de)serialization of the cart. |
| `SessionUtil` | Session attribute helpers for cart persistence. |

**Design Patterns & Libraries**  

* **DAO / Service Layer** – `CatalogService` abstracts data access.  
* **Strategy‑like Pricing** – `ProductUtil.determinePriceWithAttributes` encapsulates price logic.  
* **Facade** – The class serves as a thin façade over multiple services.  
* **Third‑Party** – `org.apache.commons.lang.StringUtils`, `org.apache.log4j.Logger`, `uk.ltd.getahead.dwr.WebContextFactory`, and `com.salesmanager.core.util.*`.  
* **Cookie Handling** – Uses plain Java EE `javax.servlet.http.Cookie`.  

---

## 2. Detailed Description  

### Execution Flow  

| Stage | What Happens | Notes |
|-------|--------------|-------|
| **Initialization** | `AjaxCatalogUtil` is instantiated by a servlet or DWR script; no stateful members. | All operations use the current HTTP request/response via `WebContextFactory`. |
| **Add / Remove** | Invoked via AJAX endpoints → the utility obtains the current session, cart, and store, then updates the cart accordingly. | Each operation recalculates the total, stores the cart back in the session, and writes a JSON cookie. |
| **Price Calculation** | `setPrice()` loads the base product, then any selected attributes, formats the price string. | Returns an empty string if product or attributes cannot be resolved. |
| **Persistence** | `SessionUtil.setMiniShoppingCart()` stores the cart object in the session; `setMiniCartCookie()` serializes to JSON and writes a cookie. | Cookie persistence is optional; cookie name is `CatalogConstants.CART_COOKIE_NAME`. |

### Assumptions & Constraints  

* The **session** is expected to contain `MerchantStore` and locale (`WW_TRANS_I18N_LOCALE`).  
* Cart is represented both in session *and* cookie; cookie is the “source of truth” for persistence across browsers.  
* **No thread‑safety** concerns – each request uses a fresh instance.  
* **Error handling** is limited to setting an error message on the cart and logging exceptions.  
* **Quantity limits** are enforced by product’s `productQuantityOrderMax`.  
* The utility expects that `CatalogService` and `MiniShoppingCartUtil` are available via `ServiceFactory` (Spring/Java‑EE injection not shown).  

### Architecture & Design Choices  

* The class is a **utility façade** rather than a Spring component; it obtains services lazily from `ServiceFactory`.  
* The heavy use of `WebContextFactory` tightly couples the utility to the DWR framework, making unit testing more difficult.  
* The cookie serialization logic is partially commented out; currently only the JSON string is set on the cart object.  
* Attribute handling uses a `Map<Long, ProductAttribute>` keyed by attribute IDs, which is fine but could be simplified with Java 8 streams.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `removeProduct(long)` | Deletes a product from the cart. | `productId` | Updated `ShoppingCart` | Sets error message to `null`, recalculates totals, updates session & cookie. |
| `setPrice(ProductAttribute[], String)` | Calculates formatted price for a product with given attributes. | `attributes`, `productId` | `String` price HTML | None (side‑effect: logs). |
| `addProductWithAttributes(long, int, ProductAttribute[])` | Adds a product with attributes. | `productId`, `quantity`, `attributes` | `ShoppingCart` | Delegates to `addProductToCart`. |
| `addProductNoAttributes(long, int)` | Adds a product without attributes. | `productId`, `quantity` | `ShoppingCart` | Delegates to `addProductToCart`. |
| `addProductToCart(HttpServletRequest, HttpServletResponse, long, int, ProductAttribute[])` | Core method that adds a product (attributes optional). | `req`, `resp`, `productId`, `quantity`, `attributes` | `ShoppingCart` | Creates/updates cart, calculates totals, persists session/cookie. |
| `setMiniCartCookie(HttpServletRequest, HttpServletResponse, ShoppingCart)` | Serializes cart to JSON and writes to cookie. | `req`, `resp`, `cart` | None | Writes/updates cookie, sets `jsonShoppingCart` field. |

### Reusable / Utility Methods  

* `MiniShoppingCartUtil.calculateTotal` – calculates cart totals.  
* `CurrencyUtil.displayFormatedAmountWithCurrency` – formats monetary amounts.  
* `ProductUtil.determinePriceWithAttributes` – calculates price based on attributes.  

---

## 4. Dependencies  

| Dependency | Type | Comments |
|------------|------|----------|
| `javax.servlet.http.*` | Java EE | Standard. |
| `org.apache.commons.lang.StringUtils` | Third‑party | String helpers. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `uk.ltd.getahead.dwr.WebContextFactory` | Third‑party | DWR framework integration. |
| `com.salesmanager.*` | Project | Core business entities, services, utilities. |
| `java.math.BigDecimal`, `java.util.*` | Standard | Collections & math. |

All dependencies are either part of Java EE or well‑known open‑source libraries; no native or platform‑specific code.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation** between cart manipulation and UI logic.  
* Uses **domain entities** (`Product`, `ShoppingCartProduct`) to keep data consistent.  
* Leverages existing services (`CatalogService`) for business logic, avoiding duplication.  

### Weaknesses & Edge Cases  

1. **Hard‑coded cookie name** – no configurability.  
2. **Cookie logic commented out** – currently only JSON string is set on the cart, but no actual cookie is written. If cookies are essential, this is a bug.  
3. **Null checks** – `setMiniCartCookie` sets `jsonShoppingCart` only if `cart != null`; but the method may be called with a null cart elsewhere.  
4. **Concurrency** – session state may be corrupted under heavy parallel requests; consider synchronizing cart modifications.  
5. **No validation of attribute combinations** – assumes that the combination is valid; missing checks for out‑of‑stock or incompatible attributes.  
6. **Exception handling** – only logs and sets a generic error message; clients may not know the precise issue.  
7. **Session fixation** – storing the entire cart in session may increase memory usage; consider a lightweight representation.  

### Potential Enhancements  

* **Dependency Injection** – convert to a Spring `@Component` and inject `CatalogService`, `MiniShoppingCartUtil`, etc., to improve testability.  
* **Configuration** – externalize cookie name, max age, and other constants.  
* **Proper Cookie Writing** – uncomment and test the cookie logic; ensure `Set-Cookie` header is sent.  
* **Attribute Validation** – check inventory levels for each attribute combination before adding.  
* **Unit Tests** – mock services and session objects to cover all code paths.  
* **API Layer** – expose these utilities as REST endpoints (e.g., Spring MVC) rather than relying on DWR.  
* **Async Pricing** – if price calculations become expensive, consider caching or async background jobs.  

---

### Summary of Recommendations  

| Area | Recommendation |
|------|-----------------|
| **Persistence** | Ensure cookies are actually written; externalize settings. |
| **Testing** | Refactor to DI for easier unit testing. |
| **Error handling** | Return specific error codes/messages for API clients. |
| **Concurrency** | Synchronize cart updates or use a concurrent map. |
| **Validation** | Add checks for attribute availability and inventory. |
| **Documentation** | Add JavaDoc to all public methods and clarify contract. |

Implementing these changes will make `AjaxCatalogUtil` more robust, maintainable, and easier to integrate with modern Java web frameworks.

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

package com.salesmanager.catalog.common;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.catalog.product.ProductAttribute;
import com.salesmanager.checkout.util.MiniShoppingCartUtil;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.common.I18NEntity;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.orders.ShoppingCartProductAttribute;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MiniShoppingCartSerializationUtil;
import com.salesmanager.core.util.ProductUtil;
import com.salesmanager.core.util.www.SessionUtil;

/**
 * Mini cart actions
 * 
 * @author Carl Samson
 * 
 */
public class AjaxCatalogUtil {

	private static Logger logger = Logger.getLogger(AjaxCatalogUtil.class);
	public static final int SECONDS_PER_YEAR = 60*60*24*365;

	public ShoppingCart removeProduct(long productId) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		
		HttpServletResponse resp = WebContextFactory.get()
		.getHttpServletResponse();

		HttpSession session = req.getSession();

		ShoppingCart cart = SessionUtil.getMiniShoppingCart(req);

		MerchantStore store = (MerchantStore) session.getAttribute("STORE");

		Locale locale = (Locale) session.getAttribute("WW_TRANS_I18N_LOCALE");

		if (cart != null) {
			
			cart.setErrorMessage(null);

			Collection productsCollection = cart.getProducts();

			Collection newProductsCollection = new ArrayList();

			if (productsCollection != null) {
				Iterator i = productsCollection.iterator();
				while (i.hasNext()) {
					ShoppingCartProduct scp = (ShoppingCartProduct) i.next();
					if (scp.getProductId() == productId) {
						continue;
					}
					newProductsCollection.add(scp);
				}
			}
			cart.setProducts(newProductsCollection);
		}

		MiniShoppingCartUtil.calculateTotal(cart, store);
		
		//save the cart in the cookie
		setMiniCartCookie(req,resp,cart);



		return cart;
	}

	public String setPrice(ProductAttribute[] attributes, String productId) {

		if (StringUtils.isBlank(productId)) {
			return "";
		}

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		HttpSession session = req.getSession();

		MerchantStore store = (MerchantStore) session.getAttribute("STORE");

		Locale locale = (Locale) session.getAttribute("WW_TRANS_I18N_LOCALE");

		String price = "";

		// get original product price
		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Product p = cservice.getProduct(Long.valueOf(productId));
			if (p == null) {
				return "";
			}

			if (attributes != null && attributes.length > 0) {
				List ids = new ArrayList();
				for (int i = 0; i < attributes.length; i++) {
					if (!attributes[i].isStringValue()) {
						ids.add(new Long(attributes[i].getValue()));
					}
				}
				Collection attrs = cservice.getProductAttributes(ids, locale
						.getLanguage());

				if (attrs != null && attrs.size() > 0) {
					price = ProductUtil.formatHTMLProductPriceWithAttributes(
							locale, store.getCurrency(), p, attrs, false);
				}
			}

		} catch (Exception e) {
			logger.error(e);
		}

		return price;

	}

	public ShoppingCart addProductWithAttributes(long productId, int quantity,
			ProductAttribute[] attributes) {
		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		HttpServletResponse res = WebContextFactory.get()
		.getHttpServletResponse();
		return addProductToCart(req, res, productId, quantity, attributes);
	}

	public ShoppingCart addProductNoAttributes(long productId, int quantity) {
		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		HttpServletResponse res = WebContextFactory.get()
		.getHttpServletResponse();
		return addProductToCart(req, res, productId, quantity, null);
	}

	/**
	 * Add product to shopping cart
	 * 
	 * @param productId
	 * @param quantity
	 * @param attributes
	 * @return
	 */
	public ShoppingCart addProductToCart(HttpServletRequest req,HttpServletResponse resp,
			long productId, int quantity, ProductAttribute[] attributes) {

		HttpSession session = req.getSession();

		ShoppingCart cart = SessionUtil.getMiniShoppingCart(req);
		


		MerchantStore store = (MerchantStore) session.getAttribute("STORE");

		if (store == null) {
			cart = new ShoppingCart();
			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(req.getLocale());
			String msg = label.getText("error.sessionexpired");
			cart.setErrorMessage(msg);
			return cart;
		}

		Locale locale = (Locale) session.getAttribute("WW_TRANS_I18N_LOCALE");

		if (cart == null) {
			cart = new ShoppingCart();
		}
		cart.setErrorMessage(null);

		try {

			// get products
			Collection productsCollection = cart.getProducts();

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Product p = cservice.getProduct(productId);

			if (p == null || store == null) {
				String message = LabelUtil.getInstance().getText(locale,
						"errors.addtocart");
				cart.setErrorMessage(message);
				return cart;
			}
			
			
			if(quantity>p.getProductQuantityOrderMax()) {
				String message = LabelUtil.getInstance().getText(locale,
				"messages.invalid.quantity");
				cart.setErrorMessage(message);
				return cart;
			}

			((I18NEntity) p).setLocale(locale, store.getCurrency());

			if (p.getMerchantId() != store.getMerchantId())
				return cart;

			boolean productFound = false;
			if (productsCollection != null && (attributes == null || attributes.length==0)) {
				Iterator i = productsCollection.iterator();
				while (i.hasNext()) {
					ShoppingCartProduct scp = (ShoppingCartProduct) i.next();
					if(scp.getAttributes()!=null && scp.getAttributes().size()>0) {
						continue;
					}
					if (scp.getProductId() == productId) {
						int qty = scp.getQuantity();
						
						if(qty + quantity>p.getProductQuantityOrderMax()) {
							String message = LabelUtil.getInstance().getText(locale,
							"messages.invalid.quantity");
							cart.setErrorMessage(message);
							return cart;
						}
						
						scp.setQuantity(qty + quantity);
						productFound = true;
						break;
					}
				}
			}

			if (!productFound) {

				ShoppingCartProduct scp = new ShoppingCartProduct();
				scp.setProductId(p.getProductId());
				scp.setQuantity(quantity);
				if (!StringUtils.isBlank(p.getSmallImagePath())) {
					scp.setImage(p.getSmallImagePath());
				} else if (!StringUtils.isBlank(p.getLargeImagePath())) {
					scp.setImage(p.getLargeImagePath());
				} else {
					// nothing for now
				}

				if (attributes != null && attributes.length > 0) {
					Map ids = new HashMap();
					for (int i = 0; i < attributes.length; i++) {
						ids.put(new Long(attributes[i].getName()),
								attributes[i]);
					}
					Collection attrs = cservice.getProductAttributes(
							new ArrayList(ids.keySet()), locale.getLanguage());

					if (attrs != null && attrs.size() > 0) {
						BigDecimal priceWithAttributes = ProductUtil
								.determinePriceWithAttributes(p, attrs, locale,
										store.getCurrency());
						scp.setPrice(priceWithAttributes);
						scp.setPriceText(CurrencyUtil
								.displayFormatedAmountWithCurrency(
										priceWithAttributes, store
												.getCurrency()));

						Iterator attrIt = attrs.iterator();
						List attrList = new ArrayList();
						while (attrIt.hasNext()) {
							//com.salesmanager.core.entity.catalog.ProductAttribute prodAttr = (com.salesmanager.core.entity.catalog.ProductAttribute) attrIt
							//		.next();
							com.salesmanager.core.entity.catalog.ProductAttribute productAttribute = (com.salesmanager.core.entity.catalog.ProductAttribute)attrIt.next();
							ShoppingCartProductAttribute scpa = new ShoppingCartProductAttribute();
							scpa.setAttributeId(productAttribute.getProductAttributeId());
							ProductAttribute pa = (ProductAttribute) ids
									.get(new Long(productAttribute
											.getProductAttributeId()));
							if (pa != null) {
								scpa.setAttributeValue(pa.getValue());
								if (pa.isStringValue()) {
									scpa.setTextValue(pa.getTextValue());
								}
								attrList.add(scpa);
							}
						}
						scp.setAttributes(attrList);

					} else {
						scp.setPrice(ProductUtil.determinePrice(p, locale,
								store.getCurrency()));
						BigDecimal price = ProductUtil.determinePrice(p,
								locale, store.getCurrency());
						scp.setPriceText(CurrencyUtil
								.displayFormatedAmountWithCurrency(price, store
										.getCurrency()));
					}
				} else {
					scp.setPrice(ProductUtil.determinePrice(p, locale, store
							.getCurrency()));
					BigDecimal price = ProductUtil.determinePrice(p, locale,
							store.getCurrency());
					scp.setPriceText(CurrencyUtil
							.displayFormatedAmountWithCurrency(price, store
									.getCurrency()));
				}
				scp.setProductName(p.getName());

				Collection products = cart.getProducts();
				if (products == null) {
					products = new ArrayList();
					cart.setProducts(products);
				}
				products.add(scp);
			}

			MiniShoppingCartUtil.calculateTotal(cart, store);

			SessionUtil.setMiniShoppingCart(cart, req);
			
			//save the cart in the cookie
			setMiniCartCookie(req,resp,cart);


			return cart;

		} catch (Exception e) {
			logger.error(e);
			cart.setErrorMessage(LabelUtil.getInstance().getText(locale,
					"errors.technical"));
			return cart;
		}

	}
	
	private void setMiniCartCookie(HttpServletRequest req, HttpServletResponse resp, ShoppingCart cart) {
		
		
		try {
			
			String serializedCart = MiniShoppingCartSerializationUtil.serializeToJSON(cart);
			if(cart!=null) {
				cart.setJsonShoppingCart(serializedCart);
			}
			/*String cookieValue = serializedCart;
			int maxAge = SECONDS_PER_YEAR;
			String cookieName = CatalogConstants.CART_COOKIE_NAME;
			if(cookieValue==null) {
				cookieValue = "";
				maxAge = 0;
			}
			
			Cookie c = null;
			Cookie[] cookies = req.getCookies();
			if (cookies != null) {
				for (int i = 0; i < cookies.length; i++) {
					Cookie cookie = cookies[i];
					if(cookie.getName().equals(CatalogConstants.SKU_COOKIE)) {
						c = cookie;
						c.setValue(cookieValue);
					}
				}
			}
			
			if(c==null) {
				c = new Cookie(cookieName,cookieValue);
			}
			c.setMaxAge(maxAge);
			resp.addCookie(c);*/
			
		} catch (Exception e) {
			logger.error("Cannot serialize cart in the cookie",e);
		}
		
	}

}



```
