# InvoiceDetailsAction.java

## Review

## 1. Summary

**Purpose & Functionality**  
`InvoiceDetailsAction` is a Struts 2 action that powers the invoice module of the SalesManager application. It handles the full lifecycle of an invoice:
- Creating a new invoice (generating a provisional `Order` object)
- Editing invoice details (customer, dates, products, shipping)
- Persisting the invoice (`saveInvoice`)
- Sending invoice e‑mail (`sendInvoiceEmail`)
- Receiving and processing payment (`receiveInvoicePayment`, `processInvoicePayment`)
- Deleting an invoice
- Displaying an existing invoice for review or editing

**Key Components**
- **Domain Entities** – `Order`, `OrderProduct`, `OrderTotal`, `Customer`, `Shipping`, `PaymentMethod`, `OrderStatusHistory`.
- **Services** – `OrderService`, `CustomerService`, `MerchantService`, `PaymentService`, `SystemService`, `CatalogService`, `ServiceFactory`.
- **Utility classes** – `CurrencyUtil`, `DateUtil`, `FileUtil`, `LabelUtil`, `MessageUtil`, `SessionUtil`, `LocaleUtil`, `PaymentUtil`, `ProductUtil`.
- **Frameworks / Libraries** – Struts 2 (`Action`, `Preparable`), Apache Commons Lang (`StringUtils`), Log4j, Spring‑style `ServiceFactory` dependency resolution.

**Notable Design Patterns**
- **Factory / Service Locator** – `ServiceFactory` is used to obtain service instances instead of dependency injection.
- **Command Pattern** – each public method is a command/action that can be invoked by a Struts dispatcher.
- **DTO/Value Object** – `OrderTotalSummary` aggregates the order totals for the view layer.
- **Session‑Scoped Model** – `SessionUtil` is used heavily to store the current order and shipping information in the HTTP session.

---

## 2. Detailed Description

### Initialization
- `prepare()` sets the page title; nothing else is initialized here.
- `InvoiceDetailsAction` extends `BaseAction` which likely contains common helpers (`getContext()`, `setMessage()`, `setTechnicalMessage()`, `authorize()`, etc.).

### Runtime Flow

1. **Creating an Invoice**
   - `createInvoice()` obtains a new order ID from `SystemService`, clears the cart, creates a new `Order` object, stores it in the session, and returns `SUCCESS`.

2. **Displaying Invoice Creation Page**
   - `displayInvoiceCreate()` resets the cart, pre‑populates references (companies/customers), loads the order (if existing), and formats dates.

3. **Saving an Invoice**
   - `saveInvoice()` is the most complex method:
     - Retrieves cart line IDs (`ids`) and product data from request parameters.
     - Validates quantity & price per line.
     - Builds a list of `OrderProduct` objects (`processProducts`).
     - Calls `OrderService.calculateTotal()` to compute `OrderTotalSummary`.
     - Persists the invoice via `OrderService.saveInvoice()`.
     - Generates an invoice URL using `FileUtil.getInvoiceUrl()` and attaches it to the response.
     - Handles analytics config if present.

4. **Sending Invoice Email**
   - `sendInvoiceEmail()` simply forwards the order to `OrderService.sendInvoiceEmail()`.

5. **Receiving & Processing Payment**
   - `receiveInvoicePayment()` loads the order, authorises it, determines applicable payment methods by filtering core modules, and populates `applicablePayments`.
   - `processInvoicePayment()` first re‑invokes `receiveInvoicePayment()` for safety, validates `paymentModule`, updates the order with the chosen payment module, updates status, records a status history, and prepares shipping totals for the analytics tag.

6. **Deleting an Invoice**
   - `deleteInvoice()` loads the order, authorises, deletes via `OrderService.deleteOrder()` and signals success.

7. **Displaying Existing Invoice**
   - `displayInvoiceDetails()` pulls the order from persistence, attaches it to the session, formats all product lines, loads shipping info, populates totals, and builds the invoice URL for the front‑end.

### Cleanup
- No explicit resource cleanup; all interactions are stateless services. Session objects are removed only in specific branches (e.g., shipping removal).

### Error Handling
- Uses `try/catch` around every public method. Exceptions are logged and converted to user‑visible messages via `BaseAction` helpers.
- `AuthorizationException` is caught separately, returning `"AUTHORIZATIONEXCEPTION"`.

---

## 3. Strengths

| Aspect | Strength |
|--------|----------|
| **Clear action methods** | Each method corresponds to a distinct user action and returns a standard Struts result. |
| **Extensive use of utilities** | `CurrencyUtil`, `DateUtil`, and `LabelUtil` centralise formatting logic, making the code DRY. |
| **Session handling** | `SessionUtil` ensures the current order is preserved across multi‑step flows. |
| **Logging** | Log4j is used consistently; critical errors are logged before setting technical messages. |
| **Service abstraction** | Domain logic is encapsulated in services (`OrderService`, `CustomerService`, etc.), keeping the action focused on orchestration. |
| **Extensibility** | Payment module filtering is dynamic; new modules can be added without modifying the action. |

---

## 3. Weaknesses & Improvement Opportunities

| Category | Issue | Impact | Suggested Fix |
|----------|-------|--------|---------------|
| **Code Duplication** | `processInvoicePayment()` calls `receiveInvoicePayment()` even though it contains almost identical logic. | Unnecessary request handling; double‑authorization. | Remove the call and duplicate the minimal logic needed to load applicable payments. |
| **Mixed Concerns** | `saveInvoice()` mixes request parsing, validation, business‑logic orchestration, and view‑related URL generation. | Hard to test, hard to maintain, violates SRP. | Extract helpers: create a `CartLineProcessor` for parsing/validation; a `SummaryGenerator` for totals; a `InvoicePersister` for persistence. |
| **Parameter Handling** | All request parameters are accessed via `HttpServletRequest.getParameter()` or via `SessionUtil` and not validated against a typed form bean. | Security risk (tampering) and harder to unit test. | Introduce a dedicated form bean (`InvoiceForm`) with typed fields, bind it via Struts parameters. |
| **Hard‑coded Strings & Magic Numbers** | Module names, core module sub‑types, status IDs (`1`, `0`, etc.) are hard‑coded. | Fragile if DB changes or internationalisation expands. | Use enums or constants (`OrderConstants`, `ConfigurationConstants`, `CoreModuleServiceSubtype`) for all magic numbers. |
| **Exception Handling** | Many `catch (Exception e)` blocks swallow all exceptions and only log. | Masking of specific failures, hard to debug. | Narrow catch blocks, propagate custom checked exceptions or wrap them in `RuntimeException`. |
| **Session Management** | Heavy reliance on `SessionUtil` makes the action heavily session‑dependent. | Hard to unit‑test and to run in stateless contexts. | Where possible, pass objects through the request or use Struts 2 `Session` interface. |
| **Thread Safety** | Instance fields (`order`, `customer`, `applicablePayments`, etc.) are shared across requests (Struts 2 actions are request‑scoped by default but custom scopes may exist). | If the action is ever used as a singleton, concurrency issues will arise. | Ensure the action is request‑scoped (documented in web.xml) or avoid instance fields and use local variables. |
| **Validation** | Validation is performed manually; no Struts 2 validation framework is used. | Duplicate logic across actions; potential missed validation rules. | Move validation to `validate()` or use a separate validator bean. |
| **Performance** | Repeated DB lookups (`CatalogService.getProduct`) inside loops. | Potential N+1 query problem. | Batch fetch all needed products before the loop, or use a cache. |
| **Null Handling** | `order.getOrderTotal()` and other collections are sometimes accessed without null checks. | Possible `NullPointerException`. | Guard against null or use `Optional` in newer Java. |
| **Internationalisation** | `customerText` is built with `<br>` tags in some methods and plain text in others. | Inconsistent display in UI. | Use a single formatting utility for all customer address rendering. |
| **Analytics** | Analytics string is retrieved via a separate config call in `processInvoicePayment()` but not in `displayInvoiceDetails()` or `saveInvoice()`. | Inconsistent presence of analytics tag. | Centralise analytics retrieval in a helper method. |
| **Code Style** | Mixed use of generics (`Collection`, `List`) with raw types (`List getIds()`). | Compile‑time warnings, possible `ClassCastException`. | Use full generics everywhere (`List<Integer>`). |
| **Readability** | Long methods (e.g., `saveInvoice`, `displayInvoiceDetails`, `processInvoicePayment`) make it difficult to grasp intent. | Hard to spot bugs. | Refactor into smaller private helper methods. |

---

## 3. Suggested Refactor / Enhancements

| Refactor | Rationale | Implementation Tips |
|----------|-----------|---------------------|
| **Use Dependency Injection** | Reduce coupling to `ServiceFactory`. | Switch to Spring or Guice; annotate fields with `@Inject` or constructor injection. |
| **Extract Validation** | Centralise quantity/price checks. | Create `InvoiceLineValidator` that returns a list of errors. |
| **Move Business Logic to Service Layer** | Keep the action thin. | `OrderService.saveInvoice()` could accept an `InvoiceCommand` object that contains all required data. |
| **Replace SessionUtil with Struts 2 Session Map** | Easier unit testing. | Store the order in the session map via `ActionContext.getContext().getSession()`. |
| **Introduce Struts 2 Validation** | Reduce duplicated error handling. | Implement `validate()` and use `@Valid` annotations or `<action-validator>` in XML. |
| **Eliminate Magic Strings** | Make code self‑documenting. | Use constants (`OrderConstants.OtShipping`, `ConfigurationConstants.G_API`, `PaymentConstants.ModuleSubtype.PAYMENT`) and enum types. |
| **Centralise URL Generation** | Avoid duplicated `<a href>` construction. | Provide `InvoiceUrlProvider` bean. |
| **Add Unit Tests** | Validate each command independently. | Use Mockito to mock services, test `saveInvoice()` validation logic, and ensure correct totals. |
| **Improve Logging** | Contextual logs (orderId, user). | Use structured logging (`log.info("Processing payment for order {}", orderId)`).

---

## 3. Specific Code Issues & Recommendations

### 3.1 `saveInvoice()`

| Problem | Recommendation |
|---------|-----------------|
| Accessing request parameters via `String.format` and `request.getParameter()` in a loop. | Wrap parameter extraction in a typed form bean or a helper method. |
| Manual validation of quantity & price with duplicated error message logic. | Use a validator object that returns a map of errors per line. |
| Conversion of `OrderTotalSummary` to array and then back to list for the view. | Keep a single list representation; avoid conversion to arrays unless necessary. |
| The invoice URL is built manually with string concatenation. | Use a `UrlBuilder` or a dedicated bean. |
| The analytics configuration is retrieved from `ConfigurationRequest` but the logic is repeated in multiple places. | Create a `AnalyticsHelper.getAnalyticsConfig(MerchantStore)` method. |
| `order.setPaymentMethod(moduleName)` where `moduleName` is derived from a label key. | Store the module code only; use label resolution at view time. |

### 3.2 Payment Handling (`receiveInvoicePayment`, `processInvoicePayment`)

| Problem | Recommendation |
|---------|-----------------|
| `receiveInvoicePayment()` does not return any status; it just populates `applicablePayments`. | Have it return a boolean or throw an exception on failure. |
| Duplicate call to `receiveInvoicePayment()` inside `processInvoicePayment()` unnecessarily re‑processes payment options. | Remove the call; the method already has the order loaded. |
| Payment module name resolution uses `label.getText("module." + paymentModule)`. | Use a dedicated enum or config lookup to map module codes to human‑readable names. |
| Date parsing uses `SimpleDateFormat` without setting locale or timezone. | Use `DateUtil.parse()` or Java 8 `LocalDate` for safer parsing. |
| Shipping total is only extracted if the module equals `"ot_shipping"`. | Extract shipping total as part of `OrderTotalSummary` or a dedicated method. |
| Analytics retrieval is performed only after payment processing; it might be required earlier. | Centralise analytics config retrieval in a helper invoked by any action that needs it. |

### 3.3 `deleteInvoice()`

| Problem | Recommendation |
|---------|-----------------|
| No check for whether the order status allows deletion. | Verify status before calling `deleteOrder`. |
| Returns `SUCCESS` even when an exception occurs. | Return `ERROR` on exception. |

### 3.4 `displayInvoiceDetails()`

| Problem | Recommendation |
|---------|-----------------|
| Mixed building of customer text with `<br>` tags and other methods that use `<br>` or plain text. | Consolidate address rendering in a single utility that accepts a locale. |
| Direct manipulation of `Session` attributes (`"PRODUCTLOADED"`, `"PRODUCTLOADED"`) that may be confusing. | Use a constant or an enum for session keys. |
| Repeated code for building the invoice URL. | Extract into a private helper. |
| The method is > 700 lines; difficult to maintain. | Split into private methods (`populateOrderProducts()`, `populateShippingInfo()`, `populateTotals()`). |

### 3.5 General Style & Conventions

| Issue | Recommendation |
|-------|----------------|
| Raw collections (`Collection`, `List`) without generics. | Use full generics (`Collection<OrderProduct>`, `List<PaymentMethod>`). |
| Public fields accessed via getters/setters but not documented in `BaseAction`. | Add Javadoc comments to explain each property’s role in the view. |
| Inconsistent null checks (sometimes `order.getOrderTotal()` is assumed non‑null). | Always guard against null; use optional patterns or utility methods. |
| Hard‑coded HTML fragments inside the action (`<br>`, `<a>`). | Keep UI markup in JSP/FreeMarker; expose only data in the action. |
| Duplicate `OrderService` acquisition in many methods. | Store the service in a field with lazy initialisation or use a helper method. |

---

## 4. Dependencies & External Libraries

- **`ServiceFactory`**: Acts as a Service Locator. While functional, it hides dependencies and makes unit testing harder. Consider replacing it with constructor injection (e.g., using Spring or Guice).
- **`CurrencyUtil` / `DateUtil`**: Utility classes that wrap JDK formatting/locale logic. Good to centralise, but be careful with thread‑safety if any static state is used.
- **`SessionUtil`**: Stores and retrieves domain objects in the HTTP session. Ensure session cleanup occurs to avoid stale data (the code does `resetCart` and removes `"PRODUCTLOADED"` where appropriate).

---

## 5. Testability

### What Works
- Methods are public actions returning standard strings, making them easy to call from unit tests.
- Domain objects are simple POJOs; they can be mocked or instantiated directly.

### What Hinders Testing
- **Heavy reliance on static service locators** (`ServiceFactory`). Mocking is possible but cumbersome.
- **Direct `HttpServletRequest` / `HttpSession` usage** via `SessionUtil`. Requires a servlet container or a mocking framework such as `MockHttpServletRequest`.
- **Parameter extraction** done inside the action (`request.getParameter(...)`). Hard to simulate without a full request setup.
- **Exception handling** swallows specific failure modes (`catch (Exception e)`), making it difficult to assert on exact error conditions.

### Suggested Tests
| Test Focus | What to Assert |
|------------|----------------|
| `createInvoice()` | New order ID > 0, cart cleared, order stored in session. |
| `displayInvoiceCreate()` | Company & customer lists populated, dates formatted correctly. |
| `saveInvoice()` – valid input | OrderTotalSummary contains expected totals; `OrderService.saveInvoice()` called with correct parameters. |
| `saveInvoice()` – invalid price/quantity | Action errors added; order not persisted. |
| `processInvoicePayment()` – successful payment | Order status updated, history added, shipping total extracted. |
| `deleteInvoice()` – unauthorized | Authorization message returned. |
| `receiveInvoicePayment()` | Applicable payments filtered correctly based on merchant config. |

Use Mockito for service mocks and Spring’s `MockHttpServletRequest` / `MockHttpSession` for session simulation.

---

## 6. Security Review

| Area | Concern | Mitigation |
|------|---------|------------|
| **Authorization** | Calls `super.authorize(o)` to verify user rights. | Ensure `authorize` throws a well‑defined `AuthorizationException`; confirm that all public methods guard against missing order IDs. |
| **Parameter Injection** | Product data (`quantity`, `price`) is passed directly from request parameters. | Validate ranges and numeric formats; sanitize before persisting. |
| **Session Fixation** | Session data (`Order`, `Shipping`) is stored without invalidating the session ID. | Consider `session.invalidate()` after critical operations or use `HttpServletRequest#changeSessionId()`. |
| **Cross‑Site Scripting** | Customer address is rendered with `<br>` tags; potential HTML injection. | Sanitize address fields or encode HTML before display. |
| **Analytics Exposure** | Analytics script is placed in a request attribute (`"ANALYTICS"`). | Ensure that the value is verified and encoded; avoid leaking sensitive data. |

---

## 7. Performance & Scalability

- **N+1 Query Risk** – In `displayInvoiceDetails()`, each `OrderProduct` triggers a `catalogService.getProduct(op.getProductId())`. If an invoice contains many products, this can become expensive. Batch fetching or caching would mitigate.
- **Session Growth** – Entire `Order` and shipping objects are kept in the session. For high‑traffic scenarios, this can increase memory usage. Consider storing only IDs in the session and fetching from the DB as needed.
- **Hard‑coded Limits** – Shipping cost is assumed to be in a total with module `"ot_shipping"`. If multiple shipping modules exist, this assumption fails. Use a dedicated shipping total type.

---

## 8. Recommendations Summary

| Category | Recommendation |
|----------|----------------|
| **Architecture** | Migrate to dependency injection; replace `ServiceFactory` with constructor injection. |
| **Code Quality** | Refactor long methods into smaller helpers; add Javadoc; use generics everywhere. |
| **Validation** | Move validation to Struts 2 validators or custom validator classes. |
| **Security** | Validate all input; sanitize HTML; ensure session handling is robust. |
| **Performance** | Batch load products; cache shipping totals; reduce session footprint. |
| **Testing** | Create unit tests for each action method; mock services; test validation logic. |
| **Internationalisation** | Centralise address rendering; use a single utility for all customer addresses. |
| **Analytics** | Create a reusable helper for retrieving analytics configuration; avoid repeated code. |

By applying these changes, `InvoiceDetailsAction` will become **smaller, easier to understand, more secure, and easier to maintain** while keeping the rich functionality that the SalesManager invoice module requires.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.invoice;

import java.math.BigDecimal;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.system.SystemService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PaymentUtil;
import com.salesmanager.core.util.ProductUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class InvoiceDetailsAction extends BaseAction implements Preparable {

	private Logger log = Logger.getLogger(InvoiceDetailsAction.class);

	private Order order = null;
	private Customer customer;// submited from the invoicedetails.jsp file
	private String comments;// submited from the jsp page
	private List<OrderStatusHistory> statusHistory;

	private List<Integer> ids = new ArrayList<Integer>();// shopping cart lines

	private Collection applicablePayments = new ArrayList();
	private String paymentModule;// submited when doing a payment
	private String invoiceDate;

	private OrderTotalSummary summary;

	private Collection orderproducts;
	private Collection ordertotals;

	private Collection companyList = new ArrayList();// reference
	private Collection customerList = new ArrayList();// reference

	private String customerText;// existing invoice
	private String sdate;// existing invoice
	private String edate;// existing invoice
	private String shippingMethodId;// shipping method used

	private MerchantStore store = null;
	private String shippingTotal = null;// for analytics
	
	private String invoiceUrl = null;

	public String getShippingTotal() {
		return shippingTotal;
	}

	public void setShippingTotal(String shippingTotal) {
		this.shippingTotal = shippingTotal;
	}

	public MerchantStore getStore() {
		return store;
	}

	public void setStore(MerchantStore store) {
		this.store = store;
	}

	private List productList = new ArrayList();// for modal box

	private String formatCustomer(Customer customer) {
		StringBuffer customerTextBuffer = new StringBuffer();

		if (!StringUtils.isBlank(customer.getCustomerCompany())) {
			customerTextBuffer.append("\r\n").append(
					customer.getCustomerCompany());
		}
		customerTextBuffer.append("\r\n").append(
				customer.getCustomerFirstname()).append(" ").append(
				customer.getCustomerLastname()).append("\r\n").append(
				customer.getCustomerBillingStreetAddress()).append("\r\n")
				.append(customer.getCustomerBillingPostalCode()).append("\r\n")
				.append(customer.getCustomerBillingCity()).append("\r\n")
				.append(customer.getBillingState()).append("\r\n").append(
						customer.getBillingCountry());
		return customerTextBuffer.toString();
	}

	/**
	 * Sends an invoice email to invoice Customer
	 * 
	 * @return
	 */
	public String sendInvoiceEmail() {

		try {

			this.prepareInvoiceReferences();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(super.getContext()
					.getMerchantid());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			order = oservice.getOrder(this.getOrder().getOrderId());

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			Customer customer = cservice.getCustomer(order.getCustomerId());

			oservice
					.sendEmailInvoice(store, order, customer, super.getLocale());
			

			super.setMessage("message.order.invoice.emailsent");
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}

		return SUCCESS;

	}

	/**
	 * Gather all invoice lines and persist to the appropriate order tables
	 * 
	 * @return
	 */
	public String saveInvoice() {

		try {

			this.prepareInvoiceReferences();

			// gather cart lines
			/**
			 * <tr>
			 * <td class="item">
			 * <input type="hidden" name ="cartlineid-+data.lineId+"
			 * id="cartlineid- +data.lineId+ " value=" + data.lineId+ "> <input
			 * type="hidden"
			 * name="productid-"+data.lineId+" id="productid-" +data.lineId+ "
			 * value=" + data.productId + "> <input type="hidden" name="ids[]"
			 * value="+data.lineId+"> <input type="hidden"
			 * name="productname-"+data
			 * .lineId+" id="productname-" +data.lineId+ "
			 * value=" + data.productName + "> <div
			 * id="productText">" + data.productName + " </div> " + prop + "</td>
			 * <td class="quantity">"; <div
			 * id="qmessage-"+data.lineId+"\"></div> <input type="text"
			 * name="quantity-" +data.lineId+
			 * " value="1" id="quantity-" +data.lineId+ " maxlength="3" />";</td>
			 * <td class="price"><div
			 * id="pmessage-"+data.lineId+"></div><input type="
			 * text" name="price-" +data.lineId+ " value=" + data.priceText + "
			 * id="price-" +data.lineId+ " size="5" maxlength="5" /></td>
			 * </tr>
			 */

			Context ctx = super.getContext();

			// check that customer.customerId is there

			// order
			Order savedOrder = SessionUtil.getOrder(super.getServletRequest());

			// check sdate edate
			savedOrder.setDatePurchased(DateUtil.getDate(sdate));
			savedOrder.setOrderDateFinished(DateUtil.getDate(edate));
			savedOrder.setOrderId(this.getOrder().getOrderId());
			savedOrder.setDisplayInvoicePayments(this.getOrder().isDisplayInvoicePayments());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			Customer customer = SessionUtil.getCustomer(super
					.getServletRequest());
			customer.setLocale(super.getLocale());

			this.setCustomerText(formatCustomer(customer));

			// get latest shipping information
			ShippingInformation shippingInformation = SessionUtil
					.getShippingInformation(super.getServletRequest());

			Shipping shipping = null;

			if (shippingInformation != null
					&& shippingInformation.getShippingMethodId() != null) {
				shipping = new Shipping();
				shipping.setHandlingCost(shippingInformation.getHandlingCost());
				shipping.setShippingCost(shippingInformation.getShippingCost());
				shipping.setShippingDescription(shippingInformation
						.getShippingMethod());

				shipping.setShippingModule(shippingInformation
						.getShippingModule());
			}

			int cartLines = 0;

			List ids = this.getIds();

			List processProducts = new ArrayList();

			Map products = SessionUtil.getOrderProducts(super
					.getServletRequest());

			if (ids == null || ids.size()==0) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"error.cart.recalculate"));
				return ERROR;
			}

			Iterator idIterator = ids.iterator();
			boolean hasError = false;

			OrderProduct op = null;

			while (idIterator.hasNext()) {
				Object o = idIterator.next();

				int iKey = -1;
				try {
					iKey = (Integer) o;
				} catch (Exception ignore) {
					continue;
				}
				try {
					cartLines = cartLines + iKey;
					// now get the productid, quantity and price
					String sProductId = super.getServletRequest().getParameter(
							"productid-" + iKey);
					String sQuantity = super.getServletRequest().getParameter(
							"quantity-" + iKey);
					String sPrice = super.getServletRequest().getParameter(
							"price-" + iKey);
					// get orderproduct
					op = (OrderProduct) products.get(String.valueOf(iKey));
					if (op == null) {
						// throw an exception
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(
										super.getLocale(),
										"error.cart.recalculate"));
						return ERROR;
					}

					op.setPriceText(sPrice);
					op.setQuantityText(sQuantity);

					processProducts.add(op);

					// validate quantity and price

					long productId = Long.parseLong(sProductId);

					int quantity = 0;
					try {
						quantity = Integer.parseInt(sQuantity);
					} catch (Exception e) {
						// TODO: handle exception
						hasError = true;
						if (op != null) {
							op.setErrorMessage(LabelUtil.getInstance().getText(
									super.getLocale(),
									"errors.quantity.invalid"));
						}
					}

					BigDecimal price = new BigDecimal("0");
					try {
						price = CurrencyUtil.validateCurrency(sPrice, ctx
								.getCurrency());
					} catch (Exception e) {
						// TODO: handle exception
						hasError = true;
						if (op != null) {
							op.setPriceErrorMessage(LabelUtil.getInstance()
									.getText(super.getLocale(),
											"messages.price.invalid"));
						}
					}

					// set the submited data
					op.setProductQuantity(quantity);
					op.setProductPrice(price);

					op.setPriceFormated(CurrencyUtil
							.displayFormatedAmountWithCurrency(price, ctx
									.getCurrency()));

					double finalPrice = price.doubleValue();
					BigDecimal bdFinalPrice = new BigDecimal(finalPrice);
					op.setCostText(CurrencyUtil
							.displayFormatedAmountWithCurrency(bdFinalPrice,
									ctx.getCurrency()));
					op.setPriceText(CurrencyUtil
							.displayFormatedAmountNoCurrency(price, ctx
									.getCurrency()));
					
					BigDecimal bdFinalPriceQty = bdFinalPrice.multiply(new BigDecimal(quantity));
					
					//op.setFinalPrice(bdFinalPrice);
					op.setFinalPrice(bdFinalPriceQty);

				} catch (Exception e) {
					log.error(e);
					super.setTechnicalMessage();
					hasError = true;
				}

			}

			summary = oservice.calculateTotal(savedOrder, processProducts,
					customer, shipping, ctx.getCurrency(), super.getLocale());
			OrderProduct[] opArray = new OrderProduct[processProducts.size()];
			OrderProduct[] objects = (OrderProduct[]) processProducts
					.toArray(opArray);
			summary.setOrderProducts(objects);

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice
					.getMerchantStore(ctx.getMerchantid());

			super.getServletRequest()
					.setAttribute("ORDERTOTALSUMMARY", summary);

			if (hasError) {
				return ERROR;
			}

			oservice.saveInvoice(ctx.getMerchantid(), savedOrder.getOrderId(),
					savedOrder.getDatePurchased(), savedOrder
							.getOrderDateFinished(), this.getComments(), savedOrder.isDisplayInvoicePayments(),
					processProducts, customer, shipping, store, super
							.getLocale());
			
			// url
			LabelUtil lhelper = LabelUtil.getInstance();
			lhelper.setLocale(super.getLocale());
			StringBuffer url = new StringBuffer().append("<a href='").append(
					FileUtil.getInvoiceUrl(savedOrder, customer)).append(
					"&request_locale=").append(customer.getCustomerLang()).append("_")
					.append(super.getLocale().getCountry()).append("' target='_blank'>");
			url.append(
					lhelper.getText(customer.getCustomerLang(),
							"label.email.invoice.viewinvoice")).append("</a>");
			
			this.setInvoiceUrl(url.toString());


			super.setSuccessMessage();

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

	}

	public String selectProduct() {

		try {

			// nothing
		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public String createInvoice() {

		try {
			// create an order
			SystemService sservice = (SystemService) ServiceFactory
					.getService(ServiceFactory.SystemService);
			long nextOrderId = sservice.getNextOrderIdSequence();

			SessionUtil.resetCart(super.getServletRequest());

			Order o = new Order();
			o.setChannel(OrderConstants.INVOICE_CHANNEL);
			o.setDatePurchased(new Date(new Date().getTime()));
			o.setOrderId(nextOrderId);
			o.setMerchantId(super.getContext().getMerchantid());
			super.getServletRequest().getSession().setAttribute("ORDER", o);
			this.setOrder(o);
			


			return SUCCESS;
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

	}

	private void prepareInvoiceReferences() throws Exception {

		Context ctx = super.getContext();

		List celist = new ArrayList();
		List crlist = new ArrayList();

		/**
		 * For functionalities not going through struts (AJAX) and requiring
		 * Labels which internally invokes ActionContext
		 */

		String selectDefaultCompany = LabelUtil.getInstance().getText(
				super.getLocale(), "label.customer.selectcompany");
		Customer co = new Customer();
		co.setCustomerCompany("-- " + selectDefaultCompany + " --");
		celist.add(co);

		String selectDefaultCustomer = LabelUtil.getInstance().getText(
				super.getLocale(), "label.customer.selectcustomer");
		Customer c = new Customer();
		c.setName("-- " + selectDefaultCustomer + " --");
		crlist.add(c);

		CustomerService cservice = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);
		Collection coll = cservice.getCustomersHavingCompanies(ctx
				.getMerchantid());
		celist.addAll(coll);
		this.setCompanyList(celist);

		Collection collcust = cservice.getCustomerList(ctx.getMerchantid());
		crlist.addAll(collcust);

		this.setCustomerList(crlist);

	}

	public String displayInvoiceCreate() {

		try {
			SessionUtil.resetCart(super.getServletRequest());
			this.prepareInvoiceReferences();
			customer = new Customer();

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			Order currentOrder = oservice
					.getOrder(this.getOrder().getOrderId());

			if (currentOrder == null) {// creation of an invoice
				// log.error("Cannot retreive order " +
				// this.getOrder().getOrderId());
				currentOrder = this.getOrder();
				currentOrder.setDatePurchased(new Date(new Date().getTime()));
			}

			this.setOrder(currentOrder);

			DateFormat myDateFormat = new SimpleDateFormat("yyyy-MM-dd");
			String invoiceDate = myDateFormat.format(currentOrder
					.getDatePurchased());
			this.setSdate(invoiceDate);

			if (currentOrder.getOrderDateFinished() != null) {
				String dueDate = myDateFormat.format(currentOrder
						.getOrderDateFinished());
				this.setEdate(dueDate);
			} else {
				this.setEdate(myDateFormat
						.format(new Date(new Date().getTime())));
			}
			// set customerText
			if (!StringUtils.isBlank(currentOrder.getBillingStreetAddress())
					&& !StringUtils.isBlank(currentOrder.getBillingCity())
					&& !StringUtils.isBlank(currentOrder.getBillingPostcode())
					&& !StringUtils.isBlank(currentOrder.getBillingState())
					&& !StringUtils.isBlank(currentOrder.getBillingCountry())) {

				StringBuffer customerInformation = new StringBuffer();
				if (!StringUtils.isBlank(currentOrder.getBillingCompany())) {
					customerInformation
							.append(currentOrder.getBillingCompany());
				} else {
					customerInformation.append(currentOrder.getBillingName());
				}
				customerInformation.append("<br>");
				customerInformation.append(
						currentOrder.getBillingStreetAddress()).append("<br>");
				customerInformation.append(currentOrder.getBillingCity())
						.append("<br>");
				customerInformation.append(currentOrder.getBillingPostcode())
						.append("<br>");
				customerInformation.append(currentOrder.getBillingState())
						.append("<br>");
				customerInformation.append(currentOrder.getBillingCountry());

				this.setCustomerText(customerInformation.toString());

			}

			return SUCCESS;
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

	}

	public String receiveInvoicePayment() {

		try {

			if (order == null || order.getOrderId() == 0) {
				log.error("Missing orderId in request parameters");
				super.setTechnicalMessage();
				return SUCCESS;
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			Order o = oservice.getOrder(order.getOrderId());
			super.authorize(o);
			this.setOrder(o);
			// get payment options configured
			Map payments = PaymentUtil.getPaymentMethods(o.getMerchantId(),
					super.getLocale());

			// get all payment methods available
			PaymentService pservice = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);
			List services = pservice.getPaymentMethodsList(super.getLocale()
					.getCountry());

			Iterator servicesIterator = services.iterator();
			while (servicesIterator.hasNext()) {
				CoreModuleService cms = (CoreModuleService) servicesIterator
						.next();
				String module = cms.getCoreModuleName();
				// filter sub-type to 0
				if (cms.getCoreModuleServiceSubtype() == 0
						&& payments.containsKey(module)) {
					applicablePayments
							.add((PaymentMethod) payments.get(module));
				}
			}

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String processInvoicePayment() {

		try {

			this.receiveInvoicePayment();

			if (order == null || order.getOrderId() == 0) {
				log.error("Missing orderId in request parameters");
				super.setTechnicalMessage();
				return SUCCESS;
			}

			if (StringUtils.isBlank(this.getPaymentModule())) {
				String msg = super.getText("error.cart.nopaymentmodule");
				super.addActionError(msg);
				return ERROR;
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			Order o = oservice.getOrder(order.getOrderId());
			super.authorize(o);
			this.setOrder(o);
			
			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(super.getLocale());

			String moduleName = label.getText(
					"module." + this.getPaymentModule());
			o.setPaymentMethod(moduleName);
			o.setPaymentModuleCode(this.getPaymentModule());
			o.setOrderStatus(OrderConstants.STATUSUPDATE);
			o.setChannel(OrderConstants.ONLINE_CHANNEL);
			

			DateFormat myDateFormat = new SimpleDateFormat("yyyy-MM-dd");
			Date dt = myDateFormat.parse(this.getInvoiceDate());
			o.setDatePurchased(dt);

			oservice.updateOrderPayment(o);
			
			OrderStatusHistory history = new OrderStatusHistory();
			history.setCustomerNotified(1);
			history.setDateAdded(new Date());
			history.setOrderId(order.getOrderId());
			history.setOrderStatusId(OrderConstants.STATUSINVOICEPAID);
			history.setComments(label.getText("invoice.status.paid"));
			
			oservice.addOrderStatusHistory(history);

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			store = mservice.getMerchantStore(o.getMerchantId());

			order = o;

			Set orderTotalSet = order.getOrderTotal();

			// transform totals to a list
			List totalsList = new ArrayList();
			if (orderTotalSet != null && orderTotalSet.size() > 0) {
				Iterator totalsIterator = orderTotalSet.iterator();
				while (totalsIterator.hasNext()) {
					OrderTotal total = (OrderTotal) totalsIterator.next();
					totalsList.add(total);
					if (total.getModule().equals("ot_shipping")) {
						shippingTotal = CurrencyUtil
								.displayFormatedAmountNoCurrency(total
										.getValue(), order.getCurrency());
					}
				}
			}

			ConfigurationRequest request = new ConfigurationRequest(store
					.getMerchantId(), ConfigurationConstants.G_API);// get all
																	// configurations
			ConfigurationResponse vo = mservice.getConfiguration(request);
			if (vo != null) {
				MerchantConfiguration config = vo
						.getMerchantConfiguration(ConfigurationConstants.G_API);
				if(config!=null) {
					String analytics = config.getConfigurationValue();
					if (!StringUtils.isBlank(analytics)) {
						super.getServletRequest().setAttribute("ANALYTICS",
								analytics);
					}
				}
			}



		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}
		super.setSuccessMessage();
		return SUCCESS;

	}

	public String deleteInvoice() {

		try {
			if (order == null || order.getOrderId() == 0) {
				log.error("Missing orderId in request parameters");
				super.setTechnicalMessage();
				return SUCCESS;
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			Order o = oservice.getOrder(order.getOrderId());
			super.authorize(o);
			oservice.deleteOrder(o);
			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}
		return SUCCESS;

	}

	public String displayInvoiceDetails() {

		try {

			Context ctx = super.getContext();

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			this.prepareInvoiceReferences();
			SessionUtil.resetCart(super.getServletRequest());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			Order order = oservice.getOrder(this.getOrder().getOrderId());

			order.setLocale(super.getLocale(), order.getCurrency());
			Set prs = order.getOrderProducts();
			LocaleUtil.setLocaleToEntityCollection(prs, super.getLocale());

			SessionUtil.setOrder(order, super.getServletRequest());

			this.setOrder(order);

			this.setSdate(DateUtil.formatDate(order.getDatePurchased()));
			this.setEdate(DateUtil.formatDate(order.getOrderDateFinished()));

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			Customer customer = cservice.getCustomer(order.getCustomerId());

			SessionUtil.setCustomer(customer, super.getServletRequest());

			this.setCustomer(customer);
			this.setCustomerText(formatCustomer(customer));

			CatalogService catalogService = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			// Comments
			Set historySet = order.getOrderHistory();
			if (historySet != null) {
				//history where customer notified = 0
				Iterator historySetIterator = historySet.iterator();
				while (historySetIterator.hasNext()) {// get the last entry
					OrderStatusHistory history = (OrderStatusHistory) historySetIterator
							.next();
					if(history.getCustomerNotified()==0) {
						this.setComments(history.getComments());
					} else {
						if(statusHistory==null) {//sent invoice
							statusHistory = new ArrayList();
						}
						statusHistory.add(history);
					}
				}
			}

			OrderTotalSummary summary = new OrderTotalSummary(order.getCurrency());
			Set orderProductsSet = order.getOrderProducts();
			if (orderProductsSet != null) {

				Map orderAccountProducts = null;

				Iterator orderProductsSetIterator = orderProductsSet.iterator();
				int lineId = 1;
				while (orderProductsSetIterator.hasNext()) {
					OrderProduct op = (OrderProduct) orderProductsSetIterator
							.next();

					// set basic attributes
					op = ProductUtil.initOrderProduct(op, ctx.getCurrency());

					Product p = catalogService.getProduct(op.getProductId());

					if (StringUtils.isBlank(op.getAttributesLine())) {
						// Collection attrs =
						// catalogService.getProductAttributes(p.getProductId());
						Collection attrs = catalogService.getProductAttributes(
								p.getProductId(), super.getLocale()
										.getLanguage());
						if (attrs != null && attrs.size() > 0) {
							op.setAttributes(true);
						}
					}

					op.setProductImage(p.getProductImage());
					op.setProductType(p.getProductType());
					op.setProductVirtual(p.isProductVirtual());
					op.setProductWidth(p.getProductWidth());
					op.setProductWeight(p.getProductWeight());
					op.setProductHeight(p.getProductHeight());
					op.setProductLength(p.getProductLength());
					op.setTaxClassId(p.getProductTaxClassId());

					if (!p.isProductVirtual()) {
						op.setShipping(true);
					}

					op.setLineId(lineId);

					SessionUtil.addOrderTotalLine(op, String.valueOf(lineId),
							super.getServletRequest());

					lineId++;

				}

				OrderProduct[] opArray = new OrderProduct[orderProductsSet
						.size()];
				OrderProduct[] objects = (OrderProduct[]) orderProductsSet
						.toArray(opArray);
				summary.setOrderProducts(objects);

			}

			if (!StringUtils.isBlank(order.getShippingMethod())) {
				super.getServletRequest().getSession().removeAttribute(
						"PRODUCTLOADED");
				// shipping information
				ShippingInformation shippingInformation = new ShippingInformation();
				shippingInformation.setShippingModule(order
						.getShippingModuleCode());
				shippingInformation
						.setShippingMethod(order.getShippingMethod());
				shippingInformation.setShippingMethodId("1");// default to 1

				// get OrderTotalHistory
				Set orderHistory = order.getOrderTotal();
				if (orderHistory != null) {
					Iterator orderHistoryIterator = orderHistory.iterator();
					while (orderHistoryIterator.hasNext()) {
						OrderTotal total = (OrderTotal) orderHistoryIterator
								.next();
						if (total.getModule().equals("ot_shipping")) {
							shippingInformation.setShippingCost(total
									.getValue());
							shippingInformation
									.setShippingCostText(CurrencyUtil
											.displayFormatedAmountNoCurrency(
													total.getValue(), ctx
															.getCurrency()));
							break;
						}
					}
					SessionUtil.setShippingInformation(shippingInformation,
							super.getServletRequest());
					super.getServletRequest().getSession().setAttribute(
							"PRODUCTLOADED", "true");
				}
			}

			this.setSummary(summary);
			super.getServletRequest()
					.setAttribute("ORDERTOTALSUMMARY", summary);
			
			// url
			LabelUtil lhelper = LabelUtil.getInstance();
			lhelper.setLocale(super.getLocale());
			StringBuffer url = new StringBuffer().append("<a href='").append(
					FileUtil.getInvoiceUrl(order, customer)).append(
					"&request_locale=").append(customer.getCustomerLang()).append("_")
					.append(super.getLocale().getCountry()).append(" ' target='_blank'>");
			url.append(
					lhelper.getText(customer.getCustomerLang(),
							"label.email.invoice.viewinvoice")).append("</a>");
			
			this.setInvoiceUrl(url.toString());

		} catch (Exception e) {
			log.error(e);
			return ERROR;
		}

		return SUCCESS;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public Collection getOrderproducts() {
		return orderproducts;
	}

	public void setOrderproducts(Collection orderproducts) {
		this.orderproducts = orderproducts;
	}

	public Collection getOrdertotals() {
		return ordertotals;
	}

	public void setOrdertotals(Collection ordertotals) {
		this.ordertotals = ordertotals;
	}

	public Collection getCompanyList() {
		return companyList;
	}

	public void setCompanyList(Collection companyList) {
		this.companyList = companyList;
	}

	public Collection getCustomerList() {
		return customerList;
	}

	public void setCustomerList(Collection customerList) {
		this.customerList = customerList;
	}

	public String getCustomerText() {
		return customerText;
	}

	public void setCustomerText(String customerText) {
		this.customerText = customerText;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public String getEdate() {
		return edate;
	}

	public void setEdate(String edate) {
		this.edate = edate;
	}

	public String getSdate() {
		return sdate;
	}

	public void setSdate(String sdate) {
		this.sdate = sdate;
	}

	public List getProductList() {
		return productList;
	}

	public void setProductList(List productList) {
		this.productList = productList;
	}

	public String getComments() {
		return comments;
	}

	public void setComments(String comments) {
		this.comments = comments;
	}

	public List<Integer> getIds() {
		return ids;
	}

	public void setIds(List<Integer> ids) {
		this.ids = ids;
	}

	public String getShippingMethodId() {
		return shippingMethodId;
	}

	public void setShippingMethodId(String shippingMethodId) {
		this.shippingMethodId = shippingMethodId;
	}

	public OrderTotalSummary getSummary() {
		return summary;
	}

	public void setSummary(OrderTotalSummary summary) {
		this.summary = summary;
	}

	public Collection getApplicablePayments() {
		return applicablePayments;
	}

	public void setApplicablePayments(Collection applicablePayments) {
		this.applicablePayments = applicablePayments;
	}

	public String getPaymentModule() {
		return paymentModule;
	}

	public void setPaymentModule(String paymentModule) {
		this.paymentModule = paymentModule;
	}

	public String getInvoiceDate() {
		return invoiceDate;
	}

	public void setInvoiceDate(String invoiceDate) {
		this.invoiceDate = invoiceDate;
	}

	public List<OrderStatusHistory> getStatusHistory() {
		return statusHistory;
	}

	public void setStatusHistory(List<OrderStatusHistory> statusHistory) {
		this.statusHistory = statusHistory;
	}

	public void prepare() throws Exception {
		super.setPageTitle("label.invoice.invoicedetails");
		
	}

	public String getInvoiceUrl() {
		return invoiceUrl;
	}

	public void setInvoiceUrl(String invoiceUrl) {
		this.invoiceUrl = invoiceUrl;
	}

}



```
