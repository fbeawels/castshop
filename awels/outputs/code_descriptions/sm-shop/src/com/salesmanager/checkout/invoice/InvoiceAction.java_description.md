# InvoiceAction.java

## Review

## 1. Summary

**Purpose**  
`InvoiceAction` is a Struts‑2 action that orchestrates the generation and display of an order invoice. The action supports three distinct entry points:

1. **`printInvoice`** – Pulls a prepared HTML invoice from a catalog service, renders it to PDF via XhtmlRenderer/IText, and streams the PDF back to the client.  
2. **`displayInvoice`** – Renders a printable invoice view (typically a JSP) with order, customer, and status‑history information.  
3. **`prepareInvoice`** – Entry point used when an email link is clicked. It validates the `fileId` token, restores the order, customer and shipping data into the session, sets the appropriate locale, and prepares the request for the invoice view.

**Key Components**  
| Component | Role |
|-----------|------|
| `OrderService`, `CustomerService`, `MerchantService` | Retrieve persisted entities. |
| `FileUtil` | Parses the `fileId` token to extract the order ID. |
| `HttpCallUtil` | Invokes the catalog service that returns an HTML invoice. |
| `ITextRenderer` | Converts the HTML to PDF. |
| `SessionUtil` | Stores/retrieves order, customer, merchant store and other data in the HTTP session. |
| `LabelUtil`, `MessageUtil` | Handles i18n message rendering. |
| `LocaleUtil`, `Locale` | Manages locale switching based on the customer’s preferred language. |

The code uses **Apache Commons Configuration**, **Apache Log4j**, **Apache Xalan/XHTMLRenderer** (for PDF generation), and **iText** (via `ITextRenderer`). The overall design follows a fairly classic Struts‑2 action pattern, with dependency lookups via a custom `ServiceFactory`.

## 2. Detailed Description

### Initialization & Dependencies
- The action extends `CheckoutBaseAction`, inheriting helper methods such as `preparePayments()` and `setTechnicalMessage()`.
- On each request, the action pulls the current `MerchantStore` or `Order` from the session via `SessionUtil`.  
- Services are retrieved using a static `ServiceFactory.getService()` call, which implies a manual dependency injection mechanism rather than a DI framework.

### Flow of Execution

#### `printInvoice`
1. Retrieve the current `Order` from the session.  
2. Build an absolute URL to the catalog’s `prepareSimpleInvoice.action` using the merchant domain and static catalog paths (`core.salesmanager.catalog.url` & `core.salesmanager.cart.uri`).  
3. Append the `FILEID` stored in the session.  
4. Perform a GET request to that URL, obtaining the invoice as HTML (`content`).  
5. Feed the HTML into `ITextRenderer`, lay it out, and write the PDF into a `ByteArrayOutputStream`.  
6. Convert the stream to a `ByteArrayInputStream` and expose it via `invoiceInputStream` for the response.  
7. If any exception occurs, log it and return `"displayInvoice"` (a fallback view) after setting a technical error.

#### `displayInvoice`
1. Load `MerchantStore`, customer, and order from the session.  
2. Set the order’s currency and locale.  
3. Extract the latest order status history entry where `customerNotified == 0` and expose its comments.  
4. If the order is not yet invoiced (`STATUSINVOICED`), add an error message via `MessageUtil`.  
5. Expose the `CUSTOMER` and `ORDER` objects to the request for rendering.  
6. Return `SUCCESS` to show the invoice JSP.

#### `prepareInvoice`
1. Validate the `fileId` query parameter.  
2. Parse it with `FileUtil.getInvoiceTokens()`, pulling out the `order.orderId`.  
3. Load the `Order`, customer, merchant store, and shipping information from the database.  
4. Store these objects back into the session via `SessionUtil`.  
5. Resolve a suitable locale:
   - Prefer the customer’s language if it exists in the store’s supported languages.
   - Fallback to the store’s default language, then to a global default.  
   - Set the locale in the `ActionContext` for Struts’ i18n.  
6. Re‑initialize each `OrderProduct` with the correct locale and currency, storing them back in the session.  
7. Persist the `fileId` in the session for later use.  
8. Return `SUCCESS`.

### Assumptions & Constraints
- The catalog service is always reachable and returns valid HTML. No retry/back‑off logic.  
- The `fileId` token is trusted; there is no cryptographic validation.  
- The code assumes the presence of certain configuration keys (`core.salesmanager.catalog.url`, etc.).  
- Session attributes like `FILEID` are set by `prepareInvoice` and read by `printInvoice`.  
- Locale selection relies on a pre‑populated `Map` of languages from the store; if none match, the default logic applies.

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `printInvoice()` | Generates PDF from the catalog’s HTML invoice and streams it. | None | `String` (Struts result name) | Sets `invoiceInputStream`; writes logs. |
| `displayInvoice()` | Prepares data for the printable invoice view. | None | `String` | Sets request attributes `CUSTOMER`, `ORDER`, `HISTORY`; adds error messages. |
| `prepareInvoice()` | Entry point for email‑sent invoice links. Restores session state and locale. | None | `String` | Sets session attributes (`FILEID`, `ORDER`, `CUSTOMER`, etc.), updates `ActionContext` locale. |
| Getters/Setters (`getFileId`, `setFileId`, …) | Plain property accessors for the action properties. | - | - | None. |
| `setInvoiceInputStream(InputStream)` | Allows the framework to use the PDF stream. | `InputStream` | - | None. |

### Utility Methods (inherited or used)
- `SessionUtil.*` – manages session storage of domain objects.  
- `HttpCallUtil.invokeGetUrl()` – HTTP GET wrapper.  
- `LabelUtil.getInstance()` / `MessageUtil.addErrorMessage()` – i18n message handling.  
- `LocaleUtil.setLocaleForRequest()` – sets request locale for view rendering.

## 4. Dependencies

| External | Type | Usage |
|----------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads application properties. |
| `org.apache.commons.lang.StringUtils` | Third‑party | String checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `org.xhtmlrenderer.pdf.ITextRenderer` | Third‑party | HTML→PDF conversion. |
| `com.lowagie.text.pdf.BaseFont` | Third‑party | (Unused import; may be a leftover). |
| `com.opensymphony.xwork2.ActionContext` | Third‑party | Struts2 context. |
| Custom `ServiceFactory`, `*Service` classes | Internal | Domain services. |
| Custom utilities (`FileUtil`, `HttpCallUtil`, `LabelUtil`, etc.) | Internal | Business‑logic helpers. |
| `javax.servlet.http.*` (inherited via base action) | Standard | Servlet API. |

No platform‑specific code; the action should work on any servlet container.

## 5. Additional Notes & Recommendations

### Strengths
- **Clear separation of concerns**: Each public method handles a specific step in the invoice workflow.  
- **Locale handling**: The action respects the customer’s preferred language and falls back gracefully.  
- **Session cleanup**: `prepareInvoice` explicitly clears the cart to avoid stale data.

### Issues & Risks
1. **Hard‑coded `BASE_FONT` import**  
   `com.lowagie.text.pdf.BaseFont` is imported but never used. This unused import should be removed to avoid confusion.

2. **String manipulation & potential NPEs**  
   `super.getServletRequest().getSession().getAttribute("FILEID")` is cast to `(String)` without null‑check. If `FILEID` is missing, a `ClassCastException` could occur.

3. **Error handling in `printInvoice`**  
   The catch block logs the exception, sets a technical message, and returns `"displayInvoice"` – which may show a generic error page. It would be clearer to return a dedicated error result or propagate the error to the caller.

4. **No validation of the `fileId` token**  
   The `prepareInvoice` method assumes the token is valid and contains an order ID. If the token is malformed or tampered with, the service call may fail silently or return null. Consider adding signature validation or expiry checks.

5. **Configuration key lookup**  
   The code retrieves configuration keys like `"core.salesmanager.catalog.url"` and `"core.salesmanager.cart.uri"` but never checks for missing keys, potentially leading to `NullPointerException` if the config is mis‑configured.

6. **Hard‑coded request attribute names**  
   Strings like `"CUSTOMER"`, `"ORDER"`, `"HISTORY"` are used directly. Using constants would reduce typos and improve maintainability.

7. **Locale fallback logic**  
   The nested loops for language lookup are O(n) each time. If the store’s language list is large, caching the result or using a map lookup by code would be more efficient.

8. **Concurrency**  
   Session attributes are mutated across multiple methods. If the action is invoked concurrently in the same session (unlikely but possible), race conditions could occur. Consider using thread‑safe session handling or synchronized blocks if needed.

9. **Magic numbers**  
   `history.getCustomerNotified() == 0` and `OrderConstants.STATUSINVOICED` are compared with literals. Use descriptive constants or enums.

10. **Exception handling**  
    The catch blocks swallow all `Exception`s. It would be safer to catch more specific exceptions (e.g., `IOException`, `ServiceException`) and handle them accordingly.

### Suggested Enhancements
- **Inject services via a DI framework** (Spring, CDI) rather than a static factory. This will improve testability.
- **Externalize the invoice URL construction** into a utility or configuration bean, making it easier to adapt to changes.
- **Add unit tests** for each public method, mocking dependencies.
- **Improve PDF generation**: set a proper PDF header/footer, handle font embedding, and manage memory usage for large invoices.
- **Add a clean‑up method** for session attributes after the invoice is displayed to avoid memory leaks.
- **Refactor locale handling** into a dedicated helper to simplify `prepareInvoice`.
- **Document the expected lifecycle** of `FILEID` and explain its purpose to new developers.

Overall, the action accomplishes its goal but would benefit from modernizing dependency injection, tightening error handling, and cleaning up a few code smells.

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
package com.salesmanager.checkout.invoice;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.DataOutputStream;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.xhtmlrenderer.pdf.ITextRenderer;

import com.lowagie.text.pdf.BaseFont;
import com.opensymphony.xwork2.ActionContext;
import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.HttpCallUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.ProductUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.www.SessionUtil;

/**
 * Displays an invoice from a url
 * 
 * @author Carl Samson
 * 
 */
public class InvoiceAction extends CheckoutBaseAction {

	private Logger log = Logger.getLogger(InvoiceAction.class);

	private String fileId;// invoked url
	private Order order;
	private Customer customer;

	private InputStream invoiceInputStream;

	public String printInvoice() {

		try {

			// get order
			order = SessionUtil.getOrder(super.getServletRequest());

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			if (order == null) {
				LabelUtil label = LabelUtil.getInstance();
				label.setLocale(super.getLocale());
				MessageUtil.addErrorMessage(getServletRequest(), label
						.getText("error.sessionexpired"));
				return "GENERICERROR";
			}

			MerchantStore store = mservice.getMerchantStore(order
					.getMerchantId());
			super.getServletRequest().setAttribute("STORE", store);

			Configuration conf = PropertiesUtil.getConfiguration();

			StringBuffer invUrl = new StringBuffer();
			invUrl.append(ReferenceUtil.getUnSecureDomain(store)).append(
					(String) conf.getString("core.salesmanager.catalog.url"))
					.append("/").append(
							(String) conf
									.getString("core.salesmanager.cart.uri"))
					.append("/prepareSimpleInvoice.action");
			;
			invUrl.append("?fileId=").append(
					(String) super.getServletRequest().getSession()
							.getAttribute("FILEID"));

			String content = HttpCallUtil.invokeGetUrl(invUrl.toString());

			ByteArrayOutputStream stream = new ByteArrayOutputStream();

			/**
			 * Known issue with UTF-8 or any accents !!!!
			 * Don't put accents...
			 */
			ITextRenderer renderer = new ITextRenderer();
			renderer.setDocumentFromString(content);
			renderer.layout();
			renderer.createPDF(stream);

			InputStream inputStream = new ByteArrayInputStream(stream
					.toByteArray());
			this.setInvoiceInputStream(inputStream);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "displayInvoice";
		} 

		return SUCCESS;

	}

	public String displayInvoice() {

		try {

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			super.preparePayments();

			super.getServletRequest().getSession().setAttribute("TOKEN",
					"TOKEN");

			customer = SessionUtil.getCustomer(super.getServletRequest());

			order = SessionUtil.getOrder(super.getServletRequest());

			order.setCurrency(store.getCurrency());
			order.setLocale(super.getLocale());

			// Comments
			Set historySet = order.getOrderHistory();
			if (historySet != null) {
				// only history where customer notified = 0
				Iterator historySetIterator = historySet.iterator();
				while (historySetIterator.hasNext()) {// get the last entry
					OrderStatusHistory history = (OrderStatusHistory) historySetIterator
							.next();
					if(history.getCustomerNotified()==0) {
						super.getServletRequest().setAttribute("HISTORY",
							history.getComments());
					}
				}
			}

			// check if invoice is already paid
			if (order.getOrderStatus() != OrderConstants.STATUSINVOICED) {
				LabelUtil label = LabelUtil.getInstance();
				label.setLocale(super.getLocale());
				MessageUtil.addErrorMessage(getServletRequest(), label
						.getText("messages.invoice.cantbepaid"));
			}

			/**
			 * Set objects in the HttpRequest
			 */

			// Customer
			super.getServletRequest().setAttribute("CUSTOMER", customer);

			// Order
			super.getServletRequest().setAttribute("ORDER", order);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;

	}

	/**
	 * Entry point from email url
	 * 
	 * @return
	 */
	public String prepareInvoice() {

		try {

			if (StringUtils.isBlank(this.getFileId())) {
				log.warn("fileId is null");
				return "GENERICERROR";
			}

			// parse url information
			Map tokens = FileUtil.getInvoiceTokens(this.getFileId());
			String orderId = (String) tokens.get("order.orderId");

			// get order
			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			order = oservice.getOrder(Long.parseLong(orderId));

			if (order == null) {
				log.warn("order is null for order id " + orderId);
				return "GENERICERROR";
			}

			SessionUtil.cleanCart(super.getServletRequest());
			SessionUtil.setToken(super.getServletRequest());// need this to
															// check a valid
															// session

			Set orderProducts = order.getOrderProducts();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			MerchantStore store = mservice.getMerchantStore(order
					.getMerchantId());

			SessionUtil.setMerchantStore(store, super.getServletRequest());

			order.setLocale(super.getLocale(), store.getCurrency());
			SessionUtil.setOrder(order, super.getServletRequest());

			if (order == null) {
				log.warn("Order is null for orderId " + orderId);
				return "GENERICERROR";
			}

			// get customer
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			customer = cservice.getCustomer(order.getCustomerId());
			SessionUtil.setCustomer(customer, super.getServletRequest());

			if (customer == null) {
				log.warn("Customer is null for customerId "
						+ order.getCustomerId());
				return "GENERICERROR";
			}

			// restore shipping information
			ShippingInformation shipping = null;
			if (!StringUtils.isBlank(order.getShippingMethod())) {
				shipping = new ShippingInformation();
				shipping.setShippingMethod(order.getShippingMethod());
				shipping.setShippingModule(order.getShippingModuleCode());

				Set orderHistory = order.getOrderTotal();
				if (orderHistory != null) {
					Iterator orderHistoryIterator = orderHistory.iterator();
					while (orderHistoryIterator.hasNext()) {
						OrderTotal total = (OrderTotal) orderHistoryIterator
								.next();
						if (total.getModule().equals("ot_shipping")) {
							shipping.setShippingCost(total.getValue());

						}
					}
				}

				SessionUtil.setShippingInformation(shipping, super
						.getServletRequest());
			}

			/** Creates a locale corresponding to customer language **/
			String customerLang = customer.getCustomerLang();

			// check if language is still supported

			String c = CountryUtil.getCountryIsoCodeById(store.getCountry());

			String newLang = null;
			Map languages = store.getGetSupportedLanguages();
			if (languages != null && languages.size() > 0) {
				Iterator i = languages.keySet().iterator();
				while (i.hasNext()) {
					Integer langKey = (Integer) i.next();
					Language lang = (Language) languages.get(langKey);
					if (lang.getCode().equals(customerLang)) {
						newLang = customerLang;
						break;
					}
				}
			}

			if (newLang == null) {
				newLang = store.getDefaultLang();
			}

			if (newLang == null) {
				newLang = LanguageUtil.getDefaultLanguage();
			}

			Locale l = new Locale(newLang, c);

			ActionContext ctx = ActionContext.getContext();
			ctx.getSession().put("WW_TRANS_I18N_LOCALE", l);

			Set products = order.getOrderProducts();

			// Init order products
			if (products != null) {

				for (Object o : products) {

					OrderProduct op = (OrderProduct) o;

					op.setLocale(super.getLocale());
					op = ProductUtil.initOrderProduct(op, store.getCurrency());
					/** Required for checkout **/
					SessionUtil.addOrderProduct(op, super.getServletRequest());
				}
			}

			LocaleUtil.setLocaleForRequest(super.getServletRequest(), super
					.getServletResponse(), ctx, store);

			// put file id in HttpSession
			super.getServletRequest().getSession().setAttribute("FILEID",
					this.getFileId());

		} catch (Exception e) {
			log.error(e);
			return "GENERICERROR";
		}

		return SUCCESS;

	}

	public String getFileId() {
		return fileId;
	}

	public void setFileId(String fileId) {
		this.fileId = fileId;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public InputStream getInvoiceInputStream() {
		return invoiceInputStream;
	}

	public void setInvoiceInputStream(InputStream invoiceInputStream) {
		this.invoiceInputStream = invoiceInputStream;
	}

}



```
