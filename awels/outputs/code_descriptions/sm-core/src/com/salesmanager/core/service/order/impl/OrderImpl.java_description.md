# OrderImpl.java

## Review

## 1. Summary  

**Purpose**  
`OrderImpl` is a service‑layer implementation that generates and sends a variety of e‑mail notifications to customers and merchants. The notifications cover:

* Order confirmation  
* Order problems (e.g., payment failures)  
* Download link reset  
* Order status updates  

The class is responsible for:

1. Pulling merchant, customer, and order data from various services (`MerchantService`, `OrderService`, `CustomerService`).  
2. Building rich HTML email bodies (tables for products, totals, addresses, etc.).  
3. Invoking `CommonService.sendHtmlEmail()` to dispatch the message.

**Key components**

| Component | Role |
|-----------|------|
| `MerchantService` | Retrieves store and merchant user information |
| `OrderService` | Fetches order details (by id) |
| `CustomerService` | Fetches customer records |
| `CommonService` | Sends the final HTML e‑mail |
| `LabelUtil` | Internationalised text lookup |
| `DateUtil`, `FileUtil`, `PropertiesUtil` | Helper utilities for dates, file URLs, config |
| `RefCache` | Caches lookup tables (e.g., order status in different languages) |

**Notable design patterns / libraries**

* **Service Locator / Factory** – `ServiceFactory.getService()` is used to obtain the required services.  
* **Template engine** – The final email content is rendered via FreeMarker templates (e.g., `email_template_checkout.ftl`).  
* **Third‑party libs** – Apache Commons (`Configuration`, `StringUtils`), Log4j.  
* **Java Collections** – The code uses raw `Set`, `Map` types and `Iterator` instead of generics and streams.

---

## 2. Detailed Description  

### Execution Flow

1. **Preparation**  
   * A service instance (merchant, order, or customer) is fetched from the factory.  
   * The `Order` object is enriched with the merchant’s currency.  

2. **Content Building**  
   * Multiple `StringBuffer`s (now `StringBuilder`s) are constructed to hold HTML fragments: product list, totals, comments, addresses, and the download link.  
   * Labels are fetched via `LabelUtil` using the customer’s language.  
   * All fragments are inserted into a `Map<String,Object>` called *context*.  

3. **Sending**  
   * `CommonService.sendHtmlEmail()` is called with the recipient address, subject, store, context, template name, and language code.  

4. **Error handling**  
   * Minimal; most methods declare `throws Exception`.  
   * Logging is performed only in a couple of places (e.g., missing merchant user info).  

### Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| `Order` contains *complete* information (addresses, totals, products). | If any of these collections are `null` or missing, the method may produce NPEs or incomplete emails. |
| All language keys exist in the label repository. | Missing keys lead to `null` labels in the email. |
| FreeMarker templates are present in the classpath. | Absence of a template will throw an exception. |
| Merchant and customer services are thread‑safe. | The implementation is stateless, so this is safe. |

### Architectural Choices  

* **Monolithic Service** – All email‑related logic lives in one class.  
* **Raw Collections & Manual String Building** – No use of modern Java features (generics, streams, `StringBuilder`, `StringJoiner`).  
* **Hard‑coded HTML** – Email body is constructed inline instead of using a dedicated view‑model or builder.  
* **Exception Strategy** – Declaring `throws Exception` instead of more specific checked exceptions or custom runtime types.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side Effects |
|--------|---------|--------|------------------------|
| `sendOrderProblemEmail(int merchantid, Order order, Customer customer, MerchantStore store)` | Sends a “problem” e‑mail to the merchant for an order with issues. | `merchantid`, `order`, `customer`, `store` | Builds email content and sends via `CommonService`. No return value. |
| `sendOrderConfirmationEmail(int merchantid, Order order, Customer customer)` | Sends order confirmation to the customer. | `merchantid`, `order`, `customer` | Builds email content and sends via `CommonService`. |
| `sendOrderConfirmationEmail(int merchantid, long orderid, long customerid)` | Convenience wrapper: fetches `Order` and `Customer` by id, then delegates to the previous method. | `merchantid`, `orderid`, `customerid` | Same as above. |
| `sendResetDownloadCountrsEmail(int merchantid, Order order, Customer customer)` | Sends a download‑link reset e‑mail after a customer requests a re‑download. | `merchantid`, `order`, `customer` | Builds a URL and sends it. |
| `sendOrderStatusEmail(Order order, String comment, Customer customer)` | Sends an order‑status‑change e‑mail. | `order`, `comment`, `customer` | Builds status info and sends it. |

### Common Sub‑tasks  

* **HTML Table Construction** – Several methods build tables for products and totals; code is duplicated across methods.  
* **Label Retrieval** – Repeated calls to `LabelUtil.getText()` for each key; could be extracted into a helper.  
* **Address Formatting** – Shipping and billing addresses are manually concatenated; could be extracted to a `formatAddress()` method.  
* **Currency Appending** – In `sendOrderConfirmationEmail`, the total text is appended with currency only for the `ot_total` module; the logic is fragile.  

---

## 4. Dependencies  

| Library / Framework | Version / Notes | Standard / Third‑Party |
|---------------------|-----------------|------------------------|
| **Apache Commons Configuration** (`org.apache.commons.configuration.Configuration`) | Reads properties (`PropertiesUtil`) | Third‑party |
| **Apache Commons Lang** (`org.apache.commons.lang.StringUtils`) | String utilities (`isBlank`) | Third‑party |
| **Log4j** (`org.apache.log4j.Logger`) | Logging | Third‑party |
| **SalesManager Core** (`com.salesmanager.*`) | Domain entities, services, utilities | Project‑specific |
| **FreeMarker** (used via `CommonService.sendHtmlEmail`) | Email template engine | Third‑party |
| **Java Collections** (`Set`, `Map`, `Iterator`) | Raw types used throughout | Standard |

*No external network or database dependencies are explicit; they are encapsulated inside the service layer.*

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Smells & Anti‑Patterns  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw Types** – `Set`, `Map`, `Iterator` are used without generics. | Runtime `ClassCastException`, loss of type safety. | Replace with generics (e.g., `Set<OrderProduct>`). |
| **Duplicate Logic** – Product list and total table construction appears in multiple methods. | Hard to maintain, increased risk of inconsistency. | Extract into reusable helper methods or builder classes. |
| **Manual String Concatenation** – Using `StringBuffer` (synchronized) for email body. | Inefficient; modern code uses `StringBuilder` or template engines. | Replace with `StringBuilder` or delegate to FreeMarker view models. |
| **Hard‑coded HTML** – Inline markup within Java. | Poor separation of concerns, difficult to style or localise. | Store HTML fragments in separate Freemarker templates or use a template engine for the entire body. |
| **Exception Handling** – Methods declare `throws Exception`. | Hides specific failure reasons, forces callers to catch generic exception. | Throw specific checked exceptions or use unchecked ones (`RuntimeException`) with clear messages. |
| **Logging** – Only a single error log. | Lack of visibility into failures (e.g., missing template, null fields). | Add more comprehensive logging at DEBUG/ERROR levels. |
| **Null Checks** – Rely on implicit checks (`order.getDeliveryStreetAddress() != null`) but no guard against missing fields. | Potential NPEs or blank emails. | Validate critical fields early and throw informative exceptions. |
| **Internationalisation** – Labels fetched one by one. | Repetitive code, risk of missing keys. | Centralise label fetching in a helper or use a key‑based approach (`labelMap.put(key, label)`). |
| **Date Formatting** – `DateUtil.formatDate` used directly; unclear locale handling. | Potential formatting mismatches. | Pass locale explicitly or use `java.time` formatting. |
| **Hard‑coded Currency Appending** – Only for `ot_total`. | Fragile if other modules need currency. | Move currency handling into the `OrderTotal` entity or a dedicated formatter. |

### 5.2 Security & Data Exposure  

* The email body is built from raw order fields; if any of those contain malicious content (e.g., XSS), it could be reflected in the email.  
* Consider sanitising input or escaping HTML before inserting into the context.  

### 5.3 Performance & Scalability  

* String concatenation and repeated `Iterator` loops are acceptable for typical email sizes.  
* If the system scales to thousands of emails per second, consider asynchronous dispatch (e.g., queueing the e‑mail task).  

### 5.4 Suggested Enhancements  

1. **Refactor to a Dedicated Email Service**  
   * Create an `EmailBuilder` that accepts a model (`Order`, `Customer`, `MerchantStore`) and returns a context map.  
   * Use FreeMarker to build the entire email body, including tables, instead of manual HTML.  

2. **Move HTML to Freemarker Templates**  
   * Define templates for product lists, totals, addresses.  
   * Pass data as POJOs; FreeMarker can iterate and format automatically.  

3. **Adopt Generics and Streams**  
   * Example:  
     ```java
     for (OrderProduct product : order.getProducts()) {
         // build using product.getProductTitle(), etc.
     }
     ```  

3. **Implement Input Validation**  
   * Add a `validateOrder()` helper that ensures all necessary fields are present.  
   * Throw a custom `EmailPreparationException` if validation fails.  

4. **Improve Logging**  
   * Log start/end of each e‑mail preparation, errors for missing templates, and any NPEs.  

5. **Unit Tests**  
   * Mock all services (`MerchantService`, `OrderService`, etc.).  
   * Verify that the context map contains expected keys and values for a given order and language.  

6. **Dependency Injection**  
   * Instead of `ServiceFactory.getService()`, inject services via constructors or a DI framework (Spring).  
   * Makes the class easier to test and decouples it from the factory.  

7. **Use `java.time` API**  
   * Replace `DateUtil` with `java.time.format.DateTimeFormatter` and `ZonedDateTime`.  

8. **Encourage Asynchronous Email Delivery**  
   * Wrap the e‑mail send call in a background job or use a messaging system (Kafka, JMS).  

---

### 5.5 Quick‑Win Refactor

```java
private Map<String,Object> buildEmailContext(Order order, Customer customer, MerchantStore store, Locale locale) {
    Map<String,Object> ctx = new HashMap<>();
    // 1. Labels
    Map<String,String> labels = labelUtil.getLabels(customer.getCustomerLang(), /*list of keys*/);
    // 2. Product table
    ctx.put("productTable", buildProductTable(order.getProducts(), labels));
    // 3. Totals table
    ctx.put("totalsTable", buildTotalsTable(order.getTotals(), store.getCurrency(), labels));
    // 4. Addresses
    ctx.put("addressDelivery", formatAddress(order.getDeliveryAddress()));
    ctx.put("addressBilling", formatAddress(order.getBillingAddress()));
    // 5. Dates
    ctx.put("dateOrdered", DateUtil.formatDate(order.getDatePurchased(), locale));
    // …
    return ctx;
}
```

All public `sendXxxEmail` methods would then simply call `buildEmailContext()` and hand the resulting map to `CommonService`.

---

### 5.6 Final Verdict  

`OrderImpl` works correctly for its primary use cases, but the implementation is **error‑prone, hard to maintain, and not aligned with modern Java best practices**. The most valuable short‑term improvement is to eliminate duplicated code, replace raw collections with generics, and shift the HTML construction into FreeMarker templates. Once those changes are in place, the class can be split into smaller, testable units (e.g., an `EmailTemplateEngine`, `AddressFormatter`, and `OrderEmailContextBuilder`). This will drastically reduce bugs, improve localisation support, and make future feature additions (e.g., new email types or alternative templates) much easier.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.service.order.impl;

import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class OrderImpl {

	private Logger log = Logger.getLogger(OrderImpl.class);

	public void sendOrderProblemEmail(int merchantid, Order order,
			Customer customer, MerchantStore store) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		//Collection<MerchantUserInformation> minfo = mservice.getMerchantUserInfo(customer
		//		.getMerchantId());
		
		//if(minfo==null) {
		//	log.error("No merchant user information for merchantId " + merchantid);
		//	return;
		//}
		
		
		
		//MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0); 

		order.setCurrency(store.getCurrency());

		Set products = order.getOrderProducts();

		Set histories = order.getOrderHistory();

		StringBuffer productlinesbuffer = new StringBuffer();
		productlinesbuffer
				.append("<table class=\"product-details\" border=\"0\" width=\"100%\" cellspacing=\"0\" cellpadding=\"2\">");

		Iterator productsiterator = products.iterator();

		while (productsiterator.hasNext()) {

			OrderProduct op = (OrderProduct) productsiterator.next();
			op.setCurrency(store.getCurrency());
			productlinesbuffer
					.append("<tr><td class=\"product-details\" align=\"right\" valign=\"top\" width=\"30\">");
			productlinesbuffer.append((int) op.getProductQuantity()).append(
					"&nbsp;x</td>");
			productlinesbuffer
					.append("<td class=\"product-details\" valign=\"top\">");
			productlinesbuffer.append(op.getProductName());
			productlinesbuffer.append("<nobr><small><em>");

			Set productsattributes = op.getOrderattributes();
			if (productsattributes != null) {
				Iterator productsattributesit = productsattributes.iterator();
				while (productsattributesit.hasNext()) {
					OrderProductAttribute opa = (OrderProductAttribute) productsattributesit
							.next();
					productlinesbuffer.append("<br>").append(
							opa.getProductOption()).append(" ").append(
							opa.getAttributeValue());
				}
			}

			productlinesbuffer.append("</em></small></nobr></td>");
			productlinesbuffer
					.append("<td class=\"product-details-num\" valign=\"top\" align=\"right\">");
			productlinesbuffer.append(op.getPrice());
			productlinesbuffer.append("</td></tr>");

		}

		productlinesbuffer.append("</table>");

		Set totals = order.getOrderTotal();

		StringBuffer totalsbuffer = new StringBuffer();
		totalsbuffer
				.append("<table border=\"0\" width=\"100%\" cellspacing=\"0\" cellpadding=\"2\">");

		if (totals != null) {
			Iterator totalsiterator = totals.iterator();

			while (totalsiterator.hasNext()) {
				OrderTotal ot = (OrderTotal) totalsiterator.next();
				totalsbuffer
						.append("<tr><td class=\"order-totals-text\" align=\"right\" width=\"100%\">");
				totalsbuffer
						.append("<b>")
						.append(ot.getTitle())
						.append("</b>")
						.append(
								"</td><td class=\"order-totals-num\" align=\"right\" nowrap=\"nowrap\">");
				totalsbuffer.append(ot.getText()).append("</td></tr>");

			}
		}
		totalsbuffer.append("</table>");

		StringBuffer historybuffer = new StringBuffer();

		if (histories != null) {
			Iterator histit = histories.iterator();
			while (histit.hasNext()) {
				OrderStatusHistory hist = (OrderStatusHistory) histit.next();
				historybuffer.append(hist.getComments()).append("<br>");
			}
		}


		LabelUtil lhelper = LabelUtil.getInstance();
		String ordernumber = lhelper.getText(customer.getCustomerLang(),
				"label.order.ordernumber");
		String dateordered = lhelper.getText(customer.getCustomerLang(),
				"label.order.dateordered");
		String orderproblem = lhelper.getText(customer.getCustomerLang(),
				"label.order.prderproblem");
		String shoppingtext = lhelper.getText(customer.getCustomerLang(),
				"email.shopping.message");
		String ordertext = lhelper.getText(customer.getCustomerLang(),
				"label.order.ordermessage");
		String shpaddress = lhelper.getText(customer.getCustomerLang(),
				"label.order.shippingaddress");
		String billaddress = lhelper.getText(customer.getCustomerLang(),
				"label.order.billingaddress");
		String shpmethod = lhelper.getText(customer.getCustomerLang(),
				"label.order.shippingmethod");
		String billmethod = lhelper.getText(customer.getCustomerLang(),
				"label.order.paymentmethod");

		String addressinfo = lhelper.getText(customer.getCustomerLang(),
				"label.generic.addressinformation");

		String productstext = lhelper.getText(customer.getCustomerLang(),
				"label.productstitle");

		Map context = new HashMap();
		context.put("ORDER_CONFIRMATION_TITLE", orderproblem);
		context.put("EMAIL_CUSTOMERS_NAME", order.getCustomerName());

		context.put("EMAIL_THANKS_FOR_SHOPPING", shoppingtext);
		context.put("EMAIL_DETAILS_FOLLOW", ordertext);

		context.put("INTRO_ORDER_NUM_TITLE", ordernumber + ": ");
		context.put("INTRO_ORDER_NUMBER", order.getOrderId());

		context.put("EMAIL_TEXT_DATE_ORDERED", dateordered + ": "
				+ DateUtil.formatDate(order.getDatePurchased()));

		context.put("PRODUCTS_TITLE", productstext);
		context.put("PRODUCTS_DETAIL", productlinesbuffer.toString());
		context.put("ORDER_TOTALS", totalsbuffer.toString());

		context.put("ORDER_COMMENTS", historybuffer.toString());

		context.put("HEADING_ADDRESS_INFORMATION", addressinfo);
		context.put("ADDRESS_DELIVERY_TITLE", shpaddress);
		context.put("ADDRESS_BILLING_TITLE", billaddress);

		context.put("SHIPPING_METHOD_TITLE", shpmethod);
		context.put("PAYMENT_METHOD_TITLE", billmethod);

		StringBuffer shippingbuffer = new StringBuffer();

		if (order.getDeliveryStreetAddress() != null
				&& !order.getDeliveryStreetAddress().trim().equals("")
				&& order.getDeliveryCity() != null
				&& !order.getDeliveryCity().trim().equals("")
				&& order.getDeliveryPostcode() != null
				&& !order.getDeliveryPostcode().trim().equals("")
				&& order.getDeliveryState() != null
				&& !order.getDeliveryState().trim().equals("")
				&& order.getDeliveryCountry() != null
				&& !order.getDeliveryCountry().trim().equals("")) {

			// shippingbuffer.append("<b>");
			shippingbuffer.append(order.getDeliveryName()).append("<BR>");
			shippingbuffer.append(order.getDeliveryStreetAddress()).append(
					"<BR>");
			shippingbuffer.append(order.getDeliveryCity()).append(", ").append(
					order.getDeliveryPostcode()).append("<BR>");
			shippingbuffer.append(order.getDeliveryState()).append(", ")
					.append(order.getDeliveryCountry());
		}

		context.put("ADDRESS_DELIVERY_DETAIL", shippingbuffer.toString());

		StringBuffer billingbuffer = new StringBuffer();
		billingbuffer.append(order.getBillingName()).append("<BR>");
		billingbuffer.append(order.getBillingStreetAddress()).append("<BR>");
		billingbuffer.append(order.getBillingCity()).append(", ").append(
				order.getBillingPostcode()).append("<BR>");
		billingbuffer.append(order.getBillingState()).append(", ").append(
				order.getBillingCountry());

		context.put("ADDRESS_BILLING_DETAIL", billingbuffer.toString());

		context.put("SHIPPING_METHOD_DETAIL", order.getShippingMethod());
		context.put("PAYMENT_METHOD_DETAIL", order.getPaymentMethod());

		String cardType = order.getCardType();
		if (cardType == null) {
			cardType = "";
		}

		context.put("PAYMENT_METHOD_FOOTER", cardType);

		context.put("INTRO_DATE_TITLE", dateordered + ": ");
		context.put("INTRO_DATE_ORDERED", DateUtil.formatDate(order
				.getDatePurchased()));
		context.put("EMAIL_DOWNLOAD_URL_TEXT", "");

		CommonService cservice = new CommonService();
		cservice.sendHtmlEmail(store.getStoreemailaddress(), orderproblem
				+ " No: " + order.getOrderId(), store, context,
				"email_template_checkout.ftl", customer.getCustomerLang());

	}

	public void sendOrderConfirmationEmail(int merchantid, Order order,
			Customer customer) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(merchantid);
		Collection<MerchantUserInformation> minfo = mservice
				.getMerchantUserInfo(merchantid);
		
		if(minfo==null) {
			log.error("No merchant user information for merchantId " + merchantid);
			return;
		}
		

		MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0); 

		order.setCurrency(store.getCurrency());

		Set products = order.getOrderProducts();

		Set histories = order.getOrderHistory();

		StringBuffer productlinesbuffer = new StringBuffer();
		productlinesbuffer
				.append("<table class=\"product-details\" border=\"0\" width=\"100%\" cellspacing=\"0\" cellpadding=\"2\">");

		Iterator productsiterator = products.iterator();

		while (productsiterator.hasNext()) {

			OrderProduct op = (OrderProduct) productsiterator.next();
			op.setCurrency(store.getCurrency());
			productlinesbuffer
					.append("<tr><td class=\"product-details\" align=\"right\" valign=\"top\" width=\"30\">");
			productlinesbuffer.append((int) op.getProductQuantity()).append(
					"&nbsp;x</td>");
			productlinesbuffer
					.append("<td class=\"product-details\" valign=\"top\">");
			productlinesbuffer.append(op.getProductName());
			productlinesbuffer.append("<nobr><small><em>");

			Set productsattributes = op.getOrderattributes();
			if (productsattributes != null) {
				Iterator productsattributesit = productsattributes.iterator();
				while (productsattributesit.hasNext()) {
					OrderProductAttribute opa = (OrderProductAttribute) productsattributesit
							.next();
					productlinesbuffer.append("<br>").append(
							opa.getProductOption()).append(" ").append(
							opa.getAttributeValue());
				}
			}

			productlinesbuffer.append("</em></small></nobr></td>");
			productlinesbuffer
					.append("<td class=\"product-details-num\" valign=\"top\" align=\"right\">");
			productlinesbuffer.append(op.getPrice());
			productlinesbuffer.append("</td></tr>");

		}

		productlinesbuffer.append("</table>");

		// Get ordertotal through a querry

		Set totals = order.getOrderTotal();

		// List totals =
		// (List)session.createQuery("from OrderTotal o where o.orderId=:p order by o.sortOrder")
		// .setParameter("p",order.getOrderId()).list();

		StringBuffer totalsbuffer = new StringBuffer();
		totalsbuffer
				.append("<table border=\"0\" width=\"100%\" cellspacing=\"0\" cellpadding=\"2\">");

		if (totals != null) {
			Iterator totalsiterator = totals.iterator();

			while (totalsiterator.hasNext()) {
				OrderTotal ot = (OrderTotal) totalsiterator.next();
				totalsbuffer
						.append("<tr><td class=\"order-totals-text\" align=\"right\" width=\"100%\">");
				totalsbuffer
						.append("<b>")
						.append(ot.getTitle())
						.append("</b>");
						if(ot.getModule().equals("ot_total")) {
							
							totalsbuffer.append(
							"</td><td class=\"order-totals-num\" align=\"right\" nowrap=\"nowrap\">");
							totalsbuffer.append(ot.getText()).append(" ").append(order.getCurrency()).append("</td></tr>");
							
							
						} else {
							
							totalsbuffer.append(
								"</td><td class=\"order-totals-num\" align=\"right\" nowrap=\"nowrap\">");
							totalsbuffer.append(ot.getText()).append("</td></tr>");
				
						}

			}
		}
		totalsbuffer.append("</table>");

		StringBuffer historybuffer = new StringBuffer();

		if (histories != null) {
			Iterator histit = histories.iterator();
			while (histit.hasNext()) {
				OrderStatusHistory hist = (OrderStatusHistory) histit.next();
				if (!StringUtils.isBlank(hist.getComments())) {
					historybuffer.append(hist.getComments()).append("<br>");
				}
			}
		}

		LabelUtil lhelper = LabelUtil.getInstance();
		String ordernumber = lhelper.getText(customer.getCustomerLang(),
				"label.order.ordernumber");
		String dateordered = lhelper.getText(customer.getCustomerLang(),
				"label.order.dateordered");
		String orderconf = lhelper.getText(customer.getCustomerLang(),
				"label.order.orderconfirmation");
		String shoppingtext = lhelper.getText(customer.getCustomerLang(),
				"email.shopping.message");
		String ordertext = lhelper.getText(customer.getCustomerLang(),
				"label.order.ordermessage");
		String shpaddress = lhelper.getText(customer.getCustomerLang(),
				"label.order.shippingaddress");
		String billaddress = lhelper.getText(customer.getCustomerLang(),
				"label.order.billingaddress");
		String shpmethod = lhelper.getText(customer.getCustomerLang(),
				"label.order.shippingmethod");
		String billmethod = lhelper.getText(customer.getCustomerLang(),
				"label.order.paymentmethod");

		String addressinfo = lhelper.getText(customer.getCustomerLang(),
				"label.generic.addressinformation");

		String productstext = lhelper.getText(customer.getCustomerLang(),
				"label.productstitle");

		Map context = new HashMap();
		context.put("ORDER_CONFIRMATION_TITLE", orderconf);
		context.put("EMAIL_CUSTOMERS_NAME", order.getCustomerName());

		context.put("EMAIL_THANKS_FOR_SHOPPING", shoppingtext);
		context.put("EMAIL_DETAILS_FOLLOW", ordertext);

		context.put("INTRO_ORDER_NUM_TITLE", ordernumber + ": ");
		context.put("INTRO_ORDER_NUMBER", order.getOrderId());

		context.put("EMAIL_TEXT_DATE_ORDERED", dateordered + ": "
				+ DateUtil.formatDate(order.getDatePurchased()));

		context.put("PRODUCTS_TITLE", productstext);
		context.put("PRODUCTS_DETAIL", productlinesbuffer.toString());
		context.put("ORDER_TOTALS", totalsbuffer.toString());

		context.put("ORDER_COMMENTS", historybuffer.toString());

		context.put("HEADING_ADDRESS_INFORMATION", addressinfo);
		context.put("ADDRESS_DELIVERY_TITLE", shpaddress);
		context.put("ADDRESS_BILLING_TITLE", billaddress);

		context.put("SHIPPING_METHOD_TITLE", shpmethod);
		context.put("PAYMENT_METHOD_TITLE", billmethod);

		StringBuffer shippingbuffer = new StringBuffer();

		if (order.getDeliveryStreetAddress() != null
				&& !order.getDeliveryStreetAddress().trim().equals("")
				&& order.getDeliveryCity() != null
				&& !order.getDeliveryCity().trim().equals("")
				&& order.getDeliveryPostcode() != null
				&& !order.getDeliveryPostcode().trim().equals("")
				&& order.getDeliveryState() != null
				&& !order.getDeliveryState().trim().equals("")
				&& order.getDeliveryCountry() != null
				&& !order.getDeliveryCountry().trim().equals("")) {

			// shippingbuffer.append("<b>");
			shippingbuffer.append(order.getDeliveryName()).append("<BR>");
			shippingbuffer.append(order.getDeliveryStreetAddress()).append(
					"<BR>");
			shippingbuffer.append(order.getDeliveryCity()).append(", ").append(
					order.getDeliveryPostcode()).append("<BR>");
			shippingbuffer.append(order.getDeliveryState()).append(", ")
					.append(order.getDeliveryCountry());
			// shippingbuffer.append("</b>");
		}

		context.put("ADDRESS_DELIVERY_DETAIL", shippingbuffer.toString());

		StringBuffer billingbuffer = new StringBuffer();
		billingbuffer.append(order.getBillingName()).append("<BR>");
		billingbuffer.append(order.getBillingStreetAddress()).append("<BR>");
		billingbuffer.append(order.getBillingCity()).append(", ").append(
				order.getBillingPostcode()).append("<BR>");
		billingbuffer.append(order.getBillingState()).append(", ").append(
				order.getBillingCountry());

		context.put("ADDRESS_BILLING_DETAIL", billingbuffer.toString());

		context.put("SHIPPING_METHOD_DETAIL", order.getShippingMethod());
		if (StringUtils.isBlank(order.getShippingMethod())) {
			context.put("SHIPPING_METHOD_DETAIL", " ");
		}

		context.put("PAYMENT_METHOD_DETAIL", order.getPaymentMethod());

		String cardType = order.getCardType();
		if (cardType == null) {
			cardType = "";
		}

		context.put("PAYMENT_METHOD_FOOTER", cardType);

		context.put("INTRO_DATE_TITLE", dateordered + ": ");
		context.put("INTRO_DATE_ORDERED", DateUtil.formatDate(order
				.getDatePurchased()));
		context.put("EMAIL_DOWNLOAD_URL_TEXT", "");

		// Locale locale = LocaleUtil.getLocale(customer.getCustomerLang());

		/*	   	*//**
		 * Invoice report
		 */
		/*
		 * ByteArrayOutputStream output = new ByteArrayOutputStream();
		 * OrderService oservice =
		 * (OrderService)ServiceFactory.getService(ServiceFactory.OrderService);
		 * oservice.prepareInvoiceReport(order, customer, locale, output);
		 * 
		 * 
		 * DataSource source = new ByteArrayDataSource(invoice,
		 * "application/pdf", output.toByteArray() );
		 */

		CommonService cservice = new CommonService();
		cservice.sendHtmlEmail(order.getCustomerEmailAddress(), orderconf
				+ " No: " + order.getOrderId(), store, context,
				"email_template_checkout.ftl", customer.getCustomerLang());

	}

	public void sendOrderConfirmationEmail(int merchantid, long orderid,
			long customerid) throws Exception {

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		CustomerService cservice = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);

		Order order = oservice.getOrder(orderid);

		if (order == null) {
			throw new Exception("Order id " + orderid + " does not exist");
		}

		Customer customer = cservice.getCustomer(customerid);

		this.sendOrderConfirmationEmail(merchantid, order, customer);

	}

	public void sendResetDownloadCountrsEmail(int merchantid, Order order,
			Customer customer) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(customer
				.getMerchantId());
		//MerchantUserInformation minfo = mservice.getMerchantUserInfo(customer
		//		.getMerchantId());

		Configuration conf = PropertiesUtil.getConfiguration();

		String downloadurl = FileUtil.getOrderDownloadFileUrl(order, customer);

		StringBuffer url = new StringBuffer();
		url.append("<a href=\"").append(downloadurl).append("\">").append(
				downloadurl).append("</a>");

		LabelUtil lhelper = LabelUtil.getInstance();
		String ordernumber = lhelper.getText(customer.getCustomerLang(),
				"label.order.ordernumber");
		String dateordered = lhelper.getText(customer.getCustomerLang(),
				"label.order.dateordered");
		String subj1 = lhelper.getText(customer.getCustomerLang(),
				"label.email.download.subject");
		String subj2 = lhelper.getText(customer.getCustomerLang(),
				"label.generic.number");
		String emailtext = lhelper.getText(customer.getCustomerLang(),
				"label.order.download.availability");

		Map context = new HashMap();
		context.put("EMAIL_DOWNLOAD_URL", url.toString());
		context.put("INTRO_ORDER_NUM_TITLE", ordernumber + ": ");
		context.put("INTRO_ORDER_NUMBER", order.getOrderId());
		context.put("INTRO_DATE_TITLE", dateordered + ": ");
		context.put("INTRO_DATE_ORDERED", DateUtil.formatDate(order
				.getDatePurchased()));
		context.put("EMAIL_DOWNLOAD_URL_TEXT", emailtext);

		CommonService cservice = new CommonService();
		cservice.sendHtmlEmail(order.getCustomerEmailAddress(), subj1 + " "
				+ subj2 + " " + order.getOrderId(), store, context,
				"email_template_checkout_download.ftl", customer
						.getCustomerLang());

	}

	public void sendOrderStatusEmail(Order order, String comment,
			Customer customer) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		MerchantStore store = mservice.getMerchantStore(order.getMerchantId());
		//MerchantUserInformation minfo = mservice.getMerchantUserInfo(order
		//		.getMerchantId());

		//if (minfo == null) {
		//	throw new Exception("Profile is null for merchantid "
		//			+ order.getMerchantId());
		//}

		LabelUtil lhelper = LabelUtil.getInstance();

		String lang = LanguageUtil.getDefaultLanguage();
		if (customer != null) {
			lang = customer.getCustomerLang();
		}

		String subject = lhelper.getText(lang, "email.status.subject");
		String ordernumber = lhelper.getText(lang, "label.order.ordernumber");
		String dateordered = lhelper.getText(lang, "label.order.dateordered");
		String dateupdated = lhelper.getText(lang, "label.order.dateupdated");
		String status = lhelper.getText(lang, "label.order.orderstatus");

		StringBuffer customerbuffer = new StringBuffer();
		customerbuffer.append(
				LabelUtil.getInstance().getText(customer.getCustomerLang(),
						"label.generic.dear")).append(" ");
		customerbuffer.append(order.getCustomerName());

		Map context = new HashMap();
		context.put("EMAIL_CUSTOMERS_NAME", customerbuffer.toString());
		context.put("EMAIL_TEXT_ORDER_NUMBER", ordernumber + ": "
				+ order.getOrderId());
		context.put("EMAIL_TEXT_DATE_ORDERED", dateordered + ": "
				+ DateUtil.formatDate(order.getDatePurchased()));
		context.put("EMAIL_TEXT_STATUS_UPDATED", dateupdated + ": "
				+ DateUtil.getPresentDate());

		Map statusmap = RefCache.getOrderstatuswithlang(LanguageUtil
				.getLanguageNumberCode(customer.getCustomerLang()));
		if (statusmap.containsKey(order.getOrderStatus())) {
			OrderStatus os = (OrderStatus) statusmap
					.get(order.getOrderStatus());
			context.put("EMAIL_TEXT_STATUS_LABEL", status + ": "
					+ os.getOrderStatusName());
		} else {
			context.put("EMAIL_TEXT_STATUS_LABEL", status + ": "
					+ order.getOrderId());
		}

		if (comment != null && !comment.trim().equals("")) {
			context.put("EMAIL_TEXT_STATUS_COMMENTS", comment.trim());
		} else {
			context.put("EMAIL_TEXT_STATUS_COMMENTS", "");
		}

		CommonService cservice = new CommonService();
		cservice.sendHtmlEmail(order.getCustomerEmailAddress(), subject,
				store, context, "email_template_order_status.ftl", customer
						.getCustomerLang());

	}

}



```
