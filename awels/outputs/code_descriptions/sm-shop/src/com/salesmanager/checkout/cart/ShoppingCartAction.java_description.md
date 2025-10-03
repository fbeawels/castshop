# ShoppingCartAction.java

## Review

## 1. Summary  

`ShoppingCartAction` is a Struts 2 action that drives the shopping‑cart workflow for a multi‑merchant e‑commerce platform.  
It is responsible for:

| Responsibility | Key component | Notes |
|----------------|---------------|-------|
| **Input validation** | `validateAddItem()` | Checks that the product/merchant identifiers are present, the product is available, in stock, and that the user‑language context is valid. |
| **Store & session handling** | `prepareStore()`, `cleanShoppingCart()` | Loads the current `MerchantStore` into the session and clears the cart if a new merchant is detected or the transaction has already been committed. |
| **Cart assembly** | `assembleShoppingCartItems()`, `assembleItems()` | Converts user selections (via `CheckoutParams`) or raw `ShoppingCartProduct` objects into `OrderProduct` objects, calculates prices, applies attribute pricing, and stores them in the session. |
| **Business‑logic orchestration** | `addToCart()`, `displayCart()`, `emptyCart()` | Entry points for adding a product, showing the cart, and emptying it. These methods coordinate validation, store loading, cart assembly and set up the data needed by the JSPs. |
| **Order totals** | `super.updateOrderTotal()` | Delegated to `CheckoutBaseAction`; it returns an `OrderTotalSummary` that is kept in the action for rendering. |

The class relies heavily on the **MVC** pattern that Struts 2 implements: the action gathers data, populates the model (`CheckoutParams`), and forwards to a view (JSP) with the `OrderTotalSummary`. It also uses **utility classes** (`SessionUtil`, `CheckoutUtil`, `CurrencyUtil`, `LabelUtil`) and a few **service layers** (`CatalogService`, `MerchantService`, `CommonService`) for persistence and business rules.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

| Phase | Method | What happens |
|-------|--------|--------------|
| **Request entry** | `addToCart()` | 1. Sets a session token. <br>2. Cleans any stale cart data. <br>3. Validates input via `validateAddItem()`. <br>4. Loads the `MerchantStore`. <br>5. Builds a list of `CheckoutParams` (just the one from the request). <br>6. Calls `assembleItems()` to convert to `OrderProduct`s and calculate totals. |
| **Cart display** | `displayCart()` | Calls `preparePayments()` (inherited from `CheckoutBaseAction`) to populate payment options. |
| **Emptying the cart** | `emptyCart()` | Sets the return URL from the store, shows a message, and returns to the view. |
| **Session handling** | `prepareStore()`, `cleanShoppingCart()` | Keeps the `MerchantStore` in the session, ensures that a cart belongs to the same merchant, and removes the cart if a previous transaction was already committed. |
| **Cart assembly** | `assembleItems()` | Iterates over `CheckoutParams`, checks for duplicate products (without attributes) to update quantity, otherwise creates a new `OrderProduct`. Handles attribute pricing, text attributes, and stores the product in the session. Finally, calculates order totals and stores them in the action. |
| **Alternative assembly** | `assembleShoppingCartItems()` | Very similar to `assembleItems()` but works with a collection of `ShoppingCartProduct` (used when items are posted from a catalog page). |
| **Error handling** | `validateAddItem()` | Performs extensive checks; logs errors via `LogMerchantUtil`; sends emails for low‑stock or out‑of‑stock situations; and populates the action’s error messages for the view. |

### 2.2 Assumptions & Constraints  

| Item | Assumption | Potential issue |
|------|------------|-----------------|
| Merchant identification | `value.getMerchantId()` is always provided | If missing, `validateAddItem()` fails; but the action may still attempt to load a store. |
| Language context | `value.getLangId()` derived from request locale or merchant defaults | The code uses a fallback to the merchant’s first `MerchantUserInformation`; this is fragile if the list is empty. |
| Session state | `SessionUtil` stores the cart, order, and merchant store | No explicit synchronization – not thread‑safe if the same session is accessed concurrently. |
| Email templates | Hard‑coded template names (`email_template_outofstock.ftl`) | If a template is missing, the email will fail silently. |
| Pricing | Product prices come from `CheckoutUtil.createOrderProduct()` which uses the store’s currency | No support for dynamic currency conversion or tax calculation in this class. |
| Attribute handling | Each attribute is validated by checking product IDs | If a product has multiple options with the same attribute ID, only the first one will be processed. |

### 2.3 Architectural Notes  

* The class follows a **thin controller** model – most business logic is delegated to utility classes or services.  
* It mixes **validation**, **business rule enforcement**, **email notification** and **session management** in a single method (`validateAddItem()`), which makes the code hard to unit‑test and maintain.  
* The action uses **session‑backed state** (`Order`, `OrderProduct`s) rather than passing data through the request; this is typical for e‑commerce carts but increases coupling to the servlet container.  
* There is duplicated logic for out‑of‑stock and low‑stock handling; extracting this into a helper would reduce code duplication.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getModel()` | Struts 2 `ModelDriven` interface | – | `CheckoutParams` instance | – |
| `validateAddItem()` | Validates request data before adding an item; sends email alerts for stock issues. | – | `boolean` success flag | Adds action errors, logs via `LogMerchantUtil`, sends emails via `CommonService`. |
| `prepareStore(int)` | Loads `MerchantStore` into the session or from service. | `merchantId` | – | Sets `store` field, puts it into the session. |
| `cleanShoppingCart()` | Prevents duplicate submissions; clears cart when merchant changes. | – | – | Calls `SessionUtil.cleanCart()` / `SessionUtil.setOrder()`. |
| `assembleShoppingCartItems(Collection)` | Builds an `Order` from raw `ShoppingCartProduct` objects (used when posting from catalog). | `items` | – | Creates `Order`, `OrderProduct`s, updates session, calculates totals. |
| `assembleItems(List)` | Builds an `Order` from `CheckoutParams` (used in `addToCart`). | `params` | – | Same side‑effects as above. |
| `addToCart()` | Entry point for adding a single item to the cart. | – | `String` result code | Sets token, validates, prepares store, calls `assembleItems()`. |
| `displayCart()` | Prepares data to render the cart page. | – | `String` success | Calls `preparePayments()` from base class. |
| `emptyCart()` | Clears the cart and returns to merchant home. | – | `String` success | Sets a user message, updates returnUrl. |
| `setMerchantStore()` (commented out) | Legacy helper for setting store in request/session. | – | – | – |
| Getters / setters (`getSummary()`, `setSummary()`, `getReturnUrl()`, `setReturnUrl()`) | Accessors for view. | – | – | – |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | For reading system properties. |
| `org.apache.commons.lang.StringUtils` | Third‑party | String helper. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.opensymphony.xwork2.*` | Struts 2 | ActionContext, ModelDriven. |
| `com.salesmanager.checkout.*` | Internal | BaseAction, Params, Constants. |
| `com.salesmanager.core.entity.*` | Internal | Domain objects (`Product`, `Order`, etc.). |
| `com.salesmanager.core.service.*` | Internal | Catalog, Merchant, Common services. |
| `com.salesmanager.core.util.*` | Internal | CheckoutUtil, CurrencyUtil, LabelUtil, LanguageUtil, SessionUtil, LogMerchantUtil, PropertiesUtil. |
| `java.util.*` | JDK | Collections, BigDecimal, Date, etc. |
| `java.math.BigDecimal` | JDK | For price calculations. |

All dependencies are standard for a Java EE web application, except for the custom `com.salesmanager` packages.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Smells & Potential Bugs  

| Issue | Impact | Suggested fix |
|-------|--------|---------------|
| **Raw types** (`Collection<ShoppingCartProduct>`, `List params = new ArrayList();`) | Compile‑time type safety lost; warnings. | Replace with generics everywhere (`List<CheckoutParams>`, `List<ShoppingCartProduct>`). |
| **Bitwise & vs. logical &&** (`if (items != null & items.size() > 0)`) | Unintentional use of bitwise AND; can lead to unexpected results. | Use `&&`. |
| **Unchecked casts** (`MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0);`) | Runtime `ClassCastException` if the list contains unexpected objects. | Iterate properly or use generics. |
| **Duplicate email‑sending code** | Hard to maintain, duplicated logic for low‑stock and out‑of‑stock. | Extract into a helper method (`sendStockAlert(...)`). |
| **Hard‑coded string literals** (`"GENERICERROR"`, `"TOKEN"`) | Fragile; risk of typos. | Define constants or use Struts result annotations. |
| **No `@SuppressWarnings` usage** | The compiler will complain for raw types; may hide other warnings. | Fix generics or add annotations if unavoidable. |
| **N‑arrowed exception handling** (`catch (Exception e) { log.error(e); }`) | Swallows specific causes; makes debugging harder. | Catch more specific exceptions or rethrow with context. |
| **Potential NPE** (`value.getAttributeValue()` could be null) | If the attribute map is missing an entry, a null key could cause NPE. | Defensive checks or use `Optional`. |
| **Session state manipulation in a non‑thread‑safe way** | Concurrent requests could corrupt the cart. | Use session‑level locks or store the cart in a thread‑safe structure. |
| **Email logic in controller** | Violates single‑responsibility; hard to test. | Move to a dedicated `StockAlertService`. |
| **Locale setup** | `prepareLocale()` is commented out; locale may not be set correctly. | Ensure locale is set early or use Struts 2’s `prepare()` method. |
| **Missing `serialVersionUID`** for Serializable domain classes (not shown here) | Warnings, but not critical. | Add if needed. |
| **No validation annotations** | Validation is manual and scattered. | Use Struts 2 validation framework (`validate()` method or XML). |

### 5.2 Performance & Scalability  

* The action recalculates totals on every request. If the cart grows large this becomes expensive. Consider caching the `OrderTotalSummary` or persisting it in a session attribute.  
* Currency conversion is done via `CurrencyUtil` inside the action; if many merchants use different currencies, a per‑merchant service that performs conversion once would reduce repeated calculations.  

### 5.3 Testability  

* **Unit‑testable** parts are mainly the `assembleItems()`/`assembleShoppingCartItems()` methods (they can be fed with mock `CheckoutParams`/`ShoppingCartProduct` and session mocks).  
* `validateAddItem()` is currently impossible to test in isolation because it mutates the action’s state, logs, and sends emails.  
* Extract the validation and email‑notification logic into a `CartValidator` or `StockAlertService`.  

### 5.4 Security  

* The action relies on a session token (`SessionUtil.setToken`) but does not guard against CSRF.  
* The `TOKEN` value is set to the merchant ID – this could be spoofed by a malicious user.  
* Suggest implementing a proper CSRF token that is tied to the session ID rather than the merchant ID.  

### 5.5 Documentation & Readability  

* The class is heavily commented, but many blocks of code lack JavaDoc.  
* Add JavaDoc to all public methods and explain the Struts result codes (`SUCCESS`, `INPUT`, `GENERICERROR`).  
* Provide a UML diagram that shows the relationships between `Order`, `OrderProduct`, `ProductAttribute`, and the session.  

### 5.6 Performance Improvements  

| Opportunity | How | Benefit |
|-------------|-----|---------|
| **Lazy loading of attributes** | Fetch attribute pricing only when needed. | Reduces service round‑trips. |
| **Batch email alerts** | Queue alerts instead of sending synchronously. | Improves response time. |
| **Cache product lookups** | Keep a small cache of product data per request. | Lowers database hit count. |

---

### 5.7 Suggested Refactoring Path  

1. **Introduce a `CartService`** that owns the cart lifecycle (add, remove, empty, calculate totals).  
2. **Move validation & stock‑alert logic** to the service layer.  
3. **Replace raw types** with generics.  
4. **Use Struts 2 `@Action` annotations** instead of hard‑coded string results where possible.  
5. **Externalize email‑template names** into a properties file or constants class.  
6. **Adopt a logging framework that supports MDC** (Mapped Diagnostic Context) to attach merchant ID automatically to log messages.  

By following these steps the action will become **smaller, easier to unit‑test, and more maintainable** while preserving the existing user experience.

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
package com.salesmanager.checkout.cart;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ActionContext;
import com.opensymphony.xwork2.ModelDriven;
import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.checkout.CheckoutParams;
import com.salesmanager.checkout.web.Constants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.orders.ShoppingCartProductAttribute;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CheckoutUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LogMerchantUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class ShoppingCartAction extends CheckoutBaseAction implements
		ModelDriven<CheckoutParams>, Constants {

	private Logger log = Logger.getLogger(ShoppingCartAction.class);

	private CheckoutParams value = new CheckoutParams();

	private CatalogService cservice = (CatalogService) ServiceFactory
			.getService(ServiceFactory.CatalogService);
	private MerchantService mservice = (MerchantService) ServiceFactory
			.getService(ServiceFactory.MerchantService);

	private String returnUrl;

	MerchantStore store = null;

	private OrderTotalSummary summary;

	public CheckoutParams getModel() {
		return value;
	}

	/**
	 * Validates input parameters for a new item added in the cart
	 * 
	 * @return
	 */
	private boolean validateAddItem() {

		boolean success = true;
		if (value.getMerchantId() == 0) {
			addActionError(getText("error.validation.parameter.missing",
					new String[] { MERCHANT_ID_PARAM }));
			success = false;
		}
		if (value.getProductId() == 0) {
			addActionError(getText("error.validation.parameter.missing",
					new String[] { PRODUCT_ID_PARAM }));
			success = false;
		}

		if (success) {
			try {

				store = mservice.getMerchantStore(value.getMerchantId());
				Collection<MerchantUserInformation> minfo = mservice
						.getMerchantUserInfo(value.getMerchantId());

				if (store == null) {
					addActionError(getText("error.merchant.unavailable",
							new String[] { String
									.valueOf(value.getMerchantId()) }));
					return false;
				}

				// maybe this has to be done
				value.setCountryId(store.getCountry());

				Product product = cservice.getProduct(value.getProductId());
				if (product == null
						|| product.getMerchantId() != value.getMerchantId()) {
					LogMerchantUtil.log(value.getMerchantId(), getText(
							"error.validation.merchant.product.ids",
							new String[] {
									String.valueOf(value.getProductId()),
									String.valueOf(value.getMerchantId()) }));
					addActionError(getText(
							"error.validation.merchant.product.ids",
							new String[] {
									String.valueOf(value.getProductId()),
									String.valueOf(value.getMerchantId()) }));
					success = false;
				} else {
					if (product.getProductDateAvailable().after(new Date())) {
						LogMerchantUtil.log(value.getMerchantId(), getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						addActionError(getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						success = false;
					}
					if (product.getProductQuantity() == OUT_OF_STOCK_PRODUCT_QUANTITY) {
						LogMerchantUtil.log(value.getMerchantId(), getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						addActionError(getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));

						Configuration config = PropertiesUtil
								.getConfiguration();

						// MerchantProfile profile =
						// mservice.getMerchantProfile(value.getMerchantId());

						String l = config.getString(
								"core.system.defaultlanguage", "en");

						if (minfo == null) {
							log
									.error("MerchantUserInformation is null for merchantId "
											+ value.getMerchantId());
							addActionError(getText(
									"error.product.unavailable.purchase",
									new String[] { String.valueOf(value
											.getProductId()) }));
							// goto global error
							throw new Exception(
									"Invalid MerchantId,Unable to find MerchantProfile");
						}
						
						MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0);

						if (!StringUtils.isBlank(user.getUserlang())) {
							l = user.getUserlang();
						}

						String description = "";

						Collection descriptionslist = product.getDescriptions();
						if (descriptionslist != null) {
							Iterator i = descriptionslist.iterator();
							while (i.hasNext()) {
								Object o = i.next();
								if (o instanceof ProductDescription) {
									ProductDescription desc = (ProductDescription) o;
									description = desc.getProductName();
									if (desc.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(l)) {
										description = desc.getProductName();
										break;
									}
								}
							}
						}

						List params = new ArrayList();
						params.add(description);
						params.add(product.getProductId());

						LabelUtil lhelper = LabelUtil.getInstance();
						String subject = lhelper.getText(l,
								"label.email.store.outofstock.subject");
						String productId = lhelper.getText(super.getLocale(),
								"label.email.store.outofstock.product", params);

						Map emailctx = new HashMap();
						emailctx.put("EMAIL_STORE_NAME", store.getStorename());
						emailctx.put("EMAIL_PRODUCT_TEXT", productId);

						CommonService cservice = new CommonService();
						cservice.sendHtmlEmail(store.getStoreemailaddress(),
								subject, store, emailctx,
								"email_template_outofstock.ftl", store
										.getDefaultLang());

						success = false;

					} else if (product.getProductQuantity() < LOW_STOCK_PRODUCT_QUANTITY) {

						Configuration config = PropertiesUtil
								.getConfiguration();

						// MerchantProfile profile =
						// mservice.getMerchantProfile(value.getMerchantId());

						String l = config.getString(
								"core.system.defaultlanguage", "en");

						if (minfo == null) {
							log
									.error("MerchantUserInformation is null for merchantId "
											+ value.getMerchantId());
							addActionError(getText(
									"error.product.unavailable.purchase",
									new String[] { String.valueOf(value
											.getProductId()) }));
							// goto global error
							throw new Exception(
									"Invalid MerchantId,Unable to find MerchantProfile");
						}

						MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0);

						if (!StringUtils.isBlank(user.getUserlang())) {
							l = user.getUserlang();
						}

						String description = "";

						Collection descriptionslist = product.getDescriptions();
						if (descriptionslist != null) {
							Iterator i = descriptionslist.iterator();
							while (i.hasNext()) {
								Object o = i.next();
								if (o instanceof ProductDescription) {
									ProductDescription desc = (ProductDescription) o;
									description = desc.getProductName();
									if (desc.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(l)) {
										description = desc.getProductName();
										break;
									}
								}
							}
						}

						List params = new ArrayList();
						params.add(description);
						params.add(product.getProductId());

						LabelUtil lhelper = LabelUtil.getInstance();
						String subject = lhelper.getText(l,
								"label.email.store.lowinventory.subject");
						String productId = lhelper.getText(super.getLocale(),
								"label.email.store.lowinventory.product",
								params);

						Map emailctx = new HashMap();
						emailctx.put("EMAIL_STORE_NAME", store.getStorename());
						emailctx.put("EMAIL_PRODUCT_TEXT", productId);

						CommonService cservice = new CommonService();
						cservice.sendHtmlEmail(store.getStoreemailaddress(),
								subject, store, emailctx,
								"email_template_lowstock.ftl", store
										.getDefaultLang());

					}

				}

			} catch (Exception e) {
				log.error("Exception occurred while getting product by Id", e);
				addActionError(getText("errors.technical"));
			}
		}

		return success;
	}

	protected void prepareStore(int merchantId) throws Exception {
		MerchantStore mstore = SessionUtil
				.getMerchantStore(getServletRequest());
		store = mstore;
		if (store == null) {
			store = mservice.getMerchantStore(merchantId);
		}
		SessionUtil.setMerchantStore(store, getServletRequest());
	}

	/*
	 * Avoid duplicate submission
	 */
	protected void cleanShoppingCart() throws Exception {
		if (SessionUtil.isComited(getServletRequest())) {
			SessionUtil.cleanCart(getServletRequest());
		}

		MerchantStore mStore = SessionUtil
				.getMerchantStore(getServletRequest());

		if (mStore != null && mStore.getMerchantId() != value.getMerchantId()) {
			SessionUtil.cleanCart(getServletRequest());
		}
	}

	protected void assembleShoppingCartItems(
			Collection<ShoppingCartProduct> items) throws Exception {
		/** Initial order **/
		// create an order with merchantId and all dates
		// will need to create a new order id when submited
		Order order = new Order();
		order.setMerchantId(store.getMerchantId());
		order.setCurrency(store.getCurrency());
		order.setDatePurchased(new Date());
		SessionUtil.setOrder(order, getServletRequest());
		/******/

		if (items != null & items.size() > 0) {

			Iterator i = items.iterator();
			while (i.hasNext()) {

				ShoppingCartProduct v = (ShoppingCartProduct) i.next();

				// Prepare order
				OrderProduct orderProduct = CheckoutUtil.createOrderProduct(v
						.getProductId(), getLocale(), store.getCurrency());
				if (orderProduct.getProductQuantityOrderMax() > 1) {
					orderProduct.setProductQuantity(v.getQuantity());
				} else {
					orderProduct.setProductQuantity(1);
				}

				orderProduct.setProductId(v.getProductId());

				List<OrderProductAttribute> attributes = new ArrayList<OrderProductAttribute>();
				if (v.getAttributes() != null && v.getAttributes().size() > 0) {
					for (ShoppingCartProductAttribute attribute : v
							.getAttributes()) {

						ProductAttribute pAttr = cservice.getProductAttribute(
								attribute.getAttributeId(), super.getLocale()
										.getLanguage());
						if (pAttr != null
								&& pAttr.getProductId() != orderProduct
										.getProductId()) {
							LogMerchantUtil
									.log(
											store.getMerchantId(),
											getText(
													"error.validation.product.attributes.ids",
													new String[] {
															String
																	.valueOf(attribute
																			.getAttributeId()),
															String
																	.valueOf(v
																			.getProductId()) }));
							continue;
						}

						if (pAttr != null
								&& pAttr.getProductId() == v.getProductId()) {
							OrderProductAttribute orderAttr = new OrderProductAttribute();
							orderAttr.setProductOptionValueId(pAttr
									.getOptionValueId());
							attributes.add(orderAttr);

							// get order product value
							ProductOptionValue pov = pAttr
									.getProductOptionValue();
							if (pov != null) {
								orderAttr.setProductOptionValue(pov.getName());
							}

							BigDecimal attrPrice = pAttr.getOptionValuePrice();

							BigDecimal price = orderProduct.getProductPrice();
							if (attrPrice != null && attrPrice.longValue() > 0) {
								price = price.add(attrPrice);
							}

							// string values
							if (!StringUtils.isBlank(attribute.getTextValue())) {
								orderAttr.setProductOptionValue(attribute
										.getTextValue());
							}
						} else {
							LogMerchantUtil
									.log(
											store.getMerchantId(),
											getText(
													"error.validation.product.attributes.ids",
													new String[] {
															String
																	.valueOf(attribute
																			.getAttributeId()),
															String
																	.valueOf(v
																			.getProductId()) }));
						}
					}
				}

				BigDecimal price = orderProduct.getProductPrice();
				orderProduct.setFinalPrice(price);
				orderProduct.setProductPrice(price);
				orderProduct.setPriceText(CurrencyUtil
						.displayFormatedAmountNoCurrency(price, store
								.getCurrency()));
				orderProduct.setPriceFormated(CurrencyUtil
						.displayFormatedAmountWithCurrency(price, store
								.getCurrency()));

				// original price
				orderProduct.setOriginalProductPrice(price);

				if (!attributes.isEmpty()) {
					CheckoutUtil.addAttributesToProduct(attributes,
							orderProduct, store.getCurrency(), getLocale());
				}

				Set attributesSet = new HashSet(attributes);
				orderProduct.setOrderattributes(attributesSet);

				SessionUtil.addOrderProduct(orderProduct, getServletRequest());

			}// end for

		}// end if

		// because this is a submission, cannot continue browsing, so that's it
		// for the OrderProduct
		Map orderProducts = SessionUtil.getOrderProducts(super
				.getServletRequest());

		// transform to a list
		List products = new ArrayList();
		if (orderProducts != null) {
			Iterator ii = orderProducts.keySet().iterator();
			while (ii.hasNext()) {
				String line = (String) ii.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				products.add(op);
			}
		}

		OrderTotalSummary summary = super.updateOrderTotal(order, products,
				store);

		this.setSummary(summary);

	}

	protected void assembleItems(List<CheckoutParams> params) throws Exception {

		// create an order with merchantId and all dates
		// will need to create a new order id when submited
		Order order = SessionUtil.getOrder(getServletRequest());
		if (order == null) {
			order = new Order();
			order.setMerchantId(store.getMerchantId());
			order.setDatePurchased(new Date());
			order.setCurrency(store.getCurrency());
		}

		SessionUtil.setOrder(order, getServletRequest());

		if (params != null & params.size() > 0) {

			Iterator i = params.iterator();
			while (i.hasNext()) {

				CheckoutParams v = (CheckoutParams) i.next();

				boolean quantityUpdated = false;

				// check if order product already exist. If that orderproduct
				// already exist
				// and has no ptoperties, so just update the quantity
				if (v.getAttributeId() == null
						|| (v.getAttributeId() != null && v.getAttributeId()
								.size() == 0)) {
					Map savedProducts = SessionUtil
							.getOrderProducts(getServletRequest());
					if (savedProducts != null) {
						Iterator it = savedProducts.keySet().iterator();
						while (it.hasNext()) {
							String line = (String) it.next();
							OrderProduct op = (OrderProduct) savedProducts
									.get(line);
							if (op.getProductId() == v.getProductId()) {
								Set attrs = op.getOrderattributes();
								if (attrs.size() == 0) {
									int qty = op.getProductQuantity();
									qty = qty + v.getQty();
									op.setProductQuantity(qty);
									quantityUpdated = true;
									break;
								}
							}
						}
					}
				}

				if (!quantityUpdated) {// new submission

					// Prepare order
					OrderProduct orderProduct = CheckoutUtil
							.createOrderProduct(v.getProductId(), getLocale(),
									store.getCurrency());
					if (orderProduct.getProductQuantityOrderMax() > 1) {
						orderProduct.setProductQuantity(v.getQty());
					} else {
						orderProduct.setProductQuantity(1);
					}

					orderProduct.setProductId(v.getProductId());

					List<OrderProductAttribute> attributes = new ArrayList<OrderProductAttribute>();
					if (v.getAttributeId() != null
							&& v.getAttributeId().size() > 0) {
						for (Long attrId : v.getAttributeId()) {
							if (attrId != null && attrId != 0) {
								ProductAttribute pAttr = cservice
										.getProductAttribute(attrId, super
												.getLocale().getLanguage());
								if (pAttr != null
										&& pAttr.getProductId() != orderProduct
												.getProductId()) {
									LogMerchantUtil
											.log(
													v.getMerchantId(),
													getText(
															"error.validation.product.attributes.ids",
															new String[] {
																	String
																			.valueOf(attrId),
																	String
																			.valueOf(v
																					.getProductId()) }));
									continue;
								}

								if (pAttr != null
										&& pAttr.getProductId() == v
												.getProductId()) {
									OrderProductAttribute orderAttr = new OrderProductAttribute();
									orderAttr.setProductOptionValueId(pAttr
											.getOptionValueId());
									attributes.add(orderAttr);

									// get order product value
									ProductOptionValue pov = pAttr
											.getProductOptionValue();
									if (pov != null) {
										orderAttr.setProductOptionValue(pov
												.getName());
									}

									BigDecimal attrPrice = pAttr
											.getOptionValuePrice();

									BigDecimal price = orderProduct
											.getProductPrice();
									if (attrPrice != null
											&& attrPrice.longValue() > 0) {
										price = price.add(attrPrice);
									}

									// string values
									if (v.getAttributeValue() != null) {

										Map attrValues = v.getAttributeValue();
										String sValue = (String) attrValues
												.get(attrId);
										if (!StringUtils.isBlank(sValue)) {
											orderAttr
													.setProductOptionValue(sValue);
										}
									}
								} else {
									LogMerchantUtil
											.log(
													v.getMerchantId(),
													getText(
															"error.validation.product.attributes.ids",
															new String[] {
																	String
																			.valueOf(attrId),
																	String
																			.valueOf(v
																					.getProductId()) }));
								}
							}
						}

						BigDecimal price = orderProduct.getProductPrice();
						orderProduct.setFinalPrice(price);
						orderProduct.setProductPrice(price);
						orderProduct.setPriceText(CurrencyUtil
								.displayFormatedAmountNoCurrency(price, store
										.getCurrency()));
						orderProduct.setPriceFormated(CurrencyUtil
								.displayFormatedAmountWithCurrency(price, store
										.getCurrency()));

						// original price
						orderProduct.setOriginalProductPrice(price);

					}

					if (!attributes.isEmpty()) {
						CheckoutUtil.addAttributesToProduct(attributes,
								orderProduct, store.getCurrency(), getLocale());
					}

					Set attributesSet = new HashSet(attributes);
					orderProduct.setOrderattributes(attributesSet);

					SessionUtil.addOrderProduct(orderProduct,
							getServletRequest());

				}

			}

		}

		// because this is a submission, cannot continue browsing, so that's it
		// for the OrderProduct
		Map orderProducts = SessionUtil.getOrderProducts(super
				.getServletRequest());

		// transform to a list
		List products = new ArrayList();
		if (orderProducts != null) {
			Iterator ii = orderProducts.keySet().iterator();
			while (ii.hasNext()) {
				String line = (String) ii.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				products.add(op);
			}
		}

		/**
		 * Order total calculation
		 */
		OrderTotalSummary summary = super.updateOrderTotal(order, products,
				store);

		this.setSummary(summary);
		ActionContext.getContext().getSession().put("TOKEN",
				String.valueOf(store.getMerchantId()));// set session token

	}

	/**
	 * Main shopping cart entry point when invoked remotely from any website
	 * 
	 * @return
	 */
	public String addToCart() {

		try {

			// prepareLocale();

			SessionUtil.setToken(super.getServletRequest());// need this to
															// check a valid
															// session

			cleanShoppingCart();

			if (!validateAddItem()) {
				return INPUT;
			}

			prepareStore(value.getMerchantId());

			value.setLangId(LanguageUtil.getLanguageNumberCode(super
					.getLocale().getDisplayLanguage()));

			if (!StringUtils.isBlank(value.getReturnUrl())) {
				// Return to merchant site Url is set from store.
				store.setContinueshoppingurl(value.getReturnUrl());
			}

			List params = new ArrayList();
			params.add(value);
			this.assembleItems(params);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;
	}

	/**
	 * Entry method for displaying the main shopping cart. This is usualy
	 * invoked after adding an object to the cart or when the user clicks on
	 * checkout from shop/catalog web application
	 * 
	 * @return
	 */
	public String displayCart() {
		try {
			preparePayments();
		} catch (Exception e) {
			log.error(e);
		}
		return SUCCESS;
	}

	public String emptyCart() {

		// set return url
		MerchantStore store = SessionUtil.getMerchantStore(super
				.getServletRequest());
		this.setReturnUrl(store.getContinueshoppingurl());

		List msg = new ArrayList();
		msg.add(getText("message.cart.emptycart"));
		super.setActionMessages(msg);

		return SUCCESS;
	}

	public OrderTotalSummary getSummary() {
		return summary;
	}

	public void setSummary(OrderTotalSummary summary) {
		this.summary = summary;
	}

	public String getReturnUrl() {
		return returnUrl;
	}

	public void setReturnUrl(String returnUrl) {
		this.returnUrl = returnUrl;
	}

	/*
	 * private void setMerchantStore(HttpServletRequest req, HttpServletResponse
	 * resp, String merchantId) throws Exception {
	 * 
	 * //different merchantId int iMerchantId = 1;
	 * 
	 * try { iMerchantId = Integer.parseInt(merchantId); } catch (Exception e) {
	 * log.error("Cannot parse merchantId to Integer " + merchantId); }
	 * 
	 * 
	 * //get MerchantStore MerchantService mservice =
	 * (MerchantService)ServiceFactory
	 * .getService(ServiceFactory.MerchantService); MerchantStore mStore =
	 * mservice.getMerchantStore(iMerchantId);
	 * 
	 * if(mStore==null) { //forward to error page
	 * log.error("MerchantStore does not exist for merchantId " + merchantId);
	 * resp.sendRedirect(req.getContextPath()+"/error.jsp"); }
	 * 
	 * req.getSession().setAttribute("STORE", mStore);
	 * 
	 * 
	 * 
	 * Cookie c = new Cookie("STORE",merchantId); c.setPath("/");
	 * c.setMaxAge(0); resp.addCookie(c);
	 * 
	 * }
	 */

}



```
