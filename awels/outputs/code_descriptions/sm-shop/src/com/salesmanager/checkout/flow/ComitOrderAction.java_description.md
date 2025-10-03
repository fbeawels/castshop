# ComitOrderAction.java

## Review

## 1. Summary  

**Purpose**  
`ComitOrderAction` (likely a typo for *Commit*) is a Struts‑style action that drives the final checkout steps of an e‑commerce platform.  
* `displayOrder()` – renders the “Thank you” page after an order has been placed.  
* `comitOrder()` – persists the order, runs the payment/validation workflow, and clears the shopping cart.

**Key components**  
| Component | Role |
|-----------|------|
| `OrderService`, `SystemService`, `CatalogService`, `MerchantService` | DAO‑style services accessed via `ServiceFactory`. |
| `WorkflowProcessor` | Executes the order workflow using a `ProcessorContext`. |
| `SessionUtil` | Thread‑safe helper for storing/retrieving objects in the HTTP session. |
| `SpringUtil` | Retrieves Spring‑managed beans (e.g., the order workflow). |
| `Logger` | Log4J for diagnostics. |

**Design patterns**  
* **Service Locator** – `ServiceFactory.getService(...)` is used to obtain business services.  
* **Command/Context** – `ProcessorContext` is populated with all required objects before the workflow runs.  
* **Template Method** – The action’s `execute`‑style methods (`displayOrder`, `comitOrder`) follow a fixed sequence: fetch data → prepare objects → perform operation → clean up.  

---

## 2. Detailed Description  

### 2.1 Flow of execution  

#### `displayOrder()`  
1. **Retrieve order & context** – Order and store are pulled from the session.  
2. **Load full order** – `OrderService.getOrder(...)` pulls the persisted order.  
3. **Attach product images** – Each `OrderProduct` is enriched with its `Product`’s image from `CatalogService`.  
4. **Gather order history** – The latest `OrderStatusHistory` is saved to the session and exposed as the `HISTORY` request attribute.  
5. **Prepare downloadable files** – If the order contains downloadable items, the file names are resolved against the product names.  
6. **Format totals** – Shipping totals are extracted and formatted for analytics.  
7. **Clean up cart** – Session cart objects are reset (except the order, store and customer).  
8. **Mark commit flag** – `orderCommited` is set to `true`.  

#### `comitOrder()`  
1. **Duplicate‑transaction guard** – Checks the session for `TRANSACTIONCOMITED`; if present an error is shown.  
2. **Create new order ID** – Calls `SystemService.getNextOrderIdSequence()` and sets it on the order.  
3. **Re‑assemble order details** – Shipping info, customer, payment method, products, and totals are collected from the session.  
4. **Context population** – All objects are placed in a `ProcessorContext` that the workflow will consume.  
5. **Run workflow** – `WorkflowProcessor.doWorkflow(context)` performs the business logic (payment, email, etc.).  
6. **Mark transaction as committed** – Session attribute `TRANSACTIONCOMITED` is set.  
7. **Optional status history** – If comments exist, they are persisted.  
8. **Error handling** – Specific exceptions (`TransactionException`, `OrderException`) are mapped to user‑friendly messages; other exceptions trigger a generic error and an email alert.  
9. **Cleanup** – The cart cookie is deleted.  

### 2.2 Dependencies & constraints  

* **Assumes** a correctly configured Spring application context (bean `orderWorkflow`).  
* **Requires** a servlet container that supports session persistence.  
* **Uses** the legacy `ServiceFactory` pattern instead of constructor/field injection.  
* **Assumes** that the order is already in the session before `displayOrder()`/`comitOrder()` are called.  

### 2.3 Architectural notes  

* The action mixes *controller* logic with *business* logic (e.g., fetching product images in `displayOrder()`).  
* All state is kept in the session; there is no thread‑safe caching mechanism.  
* The class uses raw `Collection` and `Map` types, sacrificing type safety.  

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `String displayOrder()` | Render thank‑you page after order completion. | none | `SUCCESS` or `GENERICEROR` | Sets request attributes, cleans up cart session, marks order committed |
| `String comitOrder()` | Persist order and run payment workflow. | none | `SUCCESS`, `PAYMENTERROR`, or `GENERICERROR` | Creates new order ID, runs workflow, sets session flag, deletes cart cookie |
| `OrderStatusHistory getOrderHistory()` | Getter for the latest order status history. | none | `OrderStatusHistory` | none |
| `void setOrderHistory(OrderStatusHistory)` | Setter. | `OrderStatusHistory` | none |
| `Order getOrder()` / `setOrder(Order)` | Accessors for the order. | `Order` | `Order` | none |
| `Collection getDownloadFiles()` / `setDownloadFiles(Collection)` | Accessors for downloadable files. | `Collection` | `Collection` | none |
| `String getFileMessage()` / `setFileMessage(String)` | Accessors for file messages. | `String` | `String` | none |
| `Collection getOrderProductList()` / `setOrderProductList(Collection)` | Accessors for the list of order products. | `Collection` | `Collection` | none |
| `Collection getTotals()` / `setTotals(Collection)` | Accessors for order totals. | `Collection` | `Collection` | none |
| `boolean isOrderCommited()` / `setOrderCommited(boolean)` | Accessors for commit flag. | `boolean` | `boolean` | none |
| `String getShippingTotal()` / `setShippingTotal(String)` | Accessors for formatted shipping total. | `String` | `String` | none |
| `MerchantStore getStore()` / `setStore(MerchantStore)` | Accessors for the store. | `MerchantStore` | `MerchantStore` | none |

*All getters/setters are straightforward but rely on raw types.*

---

## 4. Dependencies  

| Library / Framework | Usage |
|---------------------|-------|
| **com.salesmanager.\*** | Core domain and service classes (`OrderService`, `CatalogService`, `WorkflowProcessor`, etc.). |
| **org.apache.commons.lang.StringUtils** | Utility for checking blank strings. |
| **org.apache.log4j.Logger** | Logging. |
| **javax.servlet.http.Cookie** | Deleting cart cookie. |
| **Spring (via SpringUtil)** | Bean lookup (`orderWorkflow`). |
| **Java EE Servlet API** | Request/response/session handling. |
| **java.util.\*** | Collections, maps, sets, iterators. |
| **java.math.BigDecimal** | Monetary amounts. |

*All dependencies are third‑party except for the Java standard library.*

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw collection types** | Potential `ClassCastException` and loss of type safety. | Use generics (e.g., `Collection<OrderProduct>`). |
| **Typos** (`ComitOrderAction`, `prroducts`) | Confusing code base; may cause bugs. | Rename class/methods and correct context keys. |
| **Duplicate commit guard** | Relies on a single session attribute; may not be thread‑safe across clustering. | Use a proper transaction id or a database flag. |
| **Null checks missing** | Possible NPEs when session objects are absent. | Validate all retrieved objects before use. |
| **Hard‑coded bean name** (`orderWorkflow`) | Tightly couples the action to the Spring configuration. | Inject via dependency injection or lookup by interface. |
| **Mixing UI and business logic** | Violates separation of concerns. | Move product image enrichment and history extraction to a service. |
| **Exception handling** | Catches generic `Exception` which may swallow runtime errors. | Narrow catch blocks and rethrow unchecked exceptions. |
| **Session state leakage** | Cart cookie is deleted, but other cart session attributes might remain. | Centralize cart cleanup logic in a dedicated service. |
| **Logging** | Only logs stack traces; user messages are generic. | Log more context (order id, user id). |
| **Hard‑coded constants** (`CatalogConstants.CART_COOKIE_NAME`) | May change without code changes. | Expose via configuration. |
| **Unnecessary local variables** (`nextOrderId`) | Not required if order ID can be set by persistence layer. | Let the DAO generate IDs. |

### 5.2 Potential Edge Cases  

1. **Concurrent checkout** – Two requests from the same user could produce duplicate orders if the session guard is bypassed.  
2. **Payment workflow failure** – The order may be partially persisted; currently no rollback is performed.  
3. **Missing shipping or payment** – Null pointers may surface if these objects are absent.  
4. **Large orders** – Enumerating all products to build a list may be expensive.  
5. **Internationalization** – The `getText` calls rely on Struts resource bundles; missing keys will cause fallback errors.

### 5.3 Suggested Enhancements  

| Enhancement | Benefit |
|-------------|---------|
| **Adopt Spring Dependency Injection** | Eliminates manual `ServiceFactory` calls and simplifies unit testing. |
| **Implement a Transactional boundary** | Guarantees atomicity of order creation and payment workflow. |
| **Move business logic to services** | Keeps the action thin; easier to test and maintain. |
| **Replace session guard with idempotent workflow** | Allows safe retries without duplicate orders. |
| **Use DTOs for request/response** | Avoids leaking domain objects into the view layer. |
| **Add unit tests** | Validate each method, especially the error paths. |
| **Refactor to use Generics** | Removes unchecked cast warnings. |
| **Externalize string keys** | Improves maintainability of i18n resources. |

---  

**Overall verdict:** The class performs its intended job but suffers from legacy patterns (ServiceLocator, raw types, manual session handling). Modernizing the code to leverage dependency injection, generics, and transactional guarantees would greatly improve reliability, readability, and testability.

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
package com.salesmanager.checkout.flow;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import javax.servlet.http.Cookie;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductDownload;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderException;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.service.system.SystemService;
import com.salesmanager.core.service.workflow.ProcessorContext;
import com.salesmanager.core.service.workflow.WorkflowProcessor;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class ComitOrderAction extends CheckoutBaseAction {

	private static final long serialVersionUID = -7846147110972657310L;
	private OrderStatusHistory orderHistory = null;
	private Order order = null;
	private boolean orderCommited = false;

	private Logger log = Logger.getLogger(ComitOrderAction.class);

	private String fileMessage;
	private Collection orderProductList;
	private Collection totals;

	private String shippingTotal = null;// for analytics
	private MerchantStore store = null;

	Collection downloadFiles;

	/**
	 * Thank you page
	 * 
	 * @return
	 */
	public String displayOrder() {

		try {
			// retreive the order
			Order savedOrder = SessionUtil.getOrder(getServletRequest());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			store = SessionUtil.getMerchantStore(getServletRequest());

			super.getServletRequest().setAttribute("STORE", store);

			order = oservice.getOrder(savedOrder.getOrderId());

			order.setLocale(super.getLocale());

			super.getServletRequest().setAttribute("ORDER", order);

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			Customer customer = SessionUtil.getCustomer(getServletRequest());
			super.getServletRequest().setAttribute("CUSTOMER", customer);

			if (order == null) {
				log.error("Order id " + savedOrder.getOrderId()
						+ " is null in the database");
				addActionError(getText("message.error.comitorder.error",
						new String[] { String.valueOf(order.getOrderId()),
								store.getStoreemailaddress() }));
				return "GENERICEROR";
			}
			
			Set savedProducts = savedOrder.getOrderProducts();
			Map productsMap = new HashMap();
			List opList = new ArrayList();
			for(Object o : savedProducts) {
				
				System.out.println(o.getClass().getName());
				
				OrderProduct op = (OrderProduct)o;
				
				Product p = cservice.getProduct(op.getProductId());

				if (p != null) {
					op.setProductImage(p.getProductImage());
				}
				
				productsMap.put(op.getOrderProductId(), op);
				opList.add(op);
				
			}
			


			Set historySet = order.getOrderHistory();

			if (historySet != null) {
				Iterator it = historySet.iterator();
				while (it.hasNext()) {
					orderHistory = (OrderStatusHistory) it.next();
					SessionUtil.setOrderStatusHistory(orderHistory,
							getServletRequest());
					super.getServletRequest().setAttribute("HISTORY",
							orderHistory.getComments());
				}
			}

			getServletRequest().setAttribute("ADDRESSTYPE", "BILLING");

			ShippingInformation shippingInformation = SessionUtil
					.getShippingInformation(getServletRequest());

			if (shippingInformation != null) {
				getServletRequest().setAttribute("ADDRESSTYPE", "BOTH");
			}

			if (this.getFileMessage() == null) {

				// downloadable files
				downloadFiles = oservice.getOrderProductDownloads(this
						.getOrder().getOrderId());
				if (downloadFiles != null && downloadFiles.size() > 0) {
					Iterator dfIterator = downloadFiles.iterator();
					while (dfIterator.hasNext()) {
						OrderProductDownload opd = (OrderProductDownload) dfIterator
								.next();
						OrderProduct op = (OrderProduct) productsMap.get(opd
								.getOrderProductId());
						if (op != null) {
							opd.setProductName(op.getProductName());
						} else {
							opd.setProductName(opd.getOrderProductFilename());
						}
					}
				}
			}

			this.setOrderProductList(opList);
			
			//List orderProductList = new ArrayList(savedOrder.getOrderProducts());
			
			//this.setOrderProductList(orderProductList);

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

			this.setTotals(totalsList);

			// cleanup cart session objects except Order, MerchantStore and
			// Customer
			SessionUtil.resetCart(getServletRequest());
			SessionUtil.setComited(getServletRequest());

		} catch (Exception e) {
			log.error(e);
		}

		this.setOrderCommited(true);

		return SUCCESS;

	}

	/**
	 * Process Payment Save Order entity
	 * 
	 * @return
	 */
	public String comitOrder() {

		// Get all entities

		Order order = SessionUtil.getOrder(getServletRequest());
		MerchantStore store = SessionUtil.getMerchantStore(getServletRequest());

		PaymentMethod payment = SessionUtil
				.getPaymentMethod(getServletRequest());

		ShippingInformation shippingInformation = SessionUtil
				.getShippingInformation(getServletRequest());
		Customer customer = SessionUtil.getCustomer(getServletRequest());

		if (super.getServletRequest().getSession().getAttribute(
				"TRANSACTIONCOMITED") != null) {
			addActionError(getText("error.transaction.duplicate", new String[] {
					String.valueOf(order.getOrderId()),
					store.getStoreemailaddress() }));
			return "GENERICERROR";
		}

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		try {

			SystemService sservice = (SystemService) ServiceFactory
					.getService(ServiceFactory.SystemService);
			long nextOrderId = sservice.getNextOrderIdSequence();
			order.setOrderId(nextOrderId);

			OrderTotalSummary summary = SessionUtil
					.getOrderTotalSummary(getServletRequest());

			Shipping shipping = null;
			if (shippingInformation != null) {
				shipping = new Shipping();
				shipping.setHandlingCost(shippingInformation.getHandlingCost());
				shipping.setShippingCost(shippingInformation
						.getShippingOptionSelected().getOptionPrice());
				shipping.setShippingModule(shippingInformation
						.getShippingOptionSelected().getModule());
				shipping.setShippingDescription(shippingInformation
						.getShippingOptionSelected().getDescription());
			}

			Map orderProducts = SessionUtil
					.getOrderProducts(getServletRequest());
			
			Set s = new HashSet();
			
			for(Object o: orderProducts.values()) {
				
				OrderProduct op = (OrderProduct)o;
				s.add(op);
			}

			order.setOrderProducts(s);

			// ajust order object
			order.setCustomerEmailAddress(customer.getCustomerEmailAddress());

			String comments = null;
			if (this.getOrderHistory() != null) {
				comments = this.getOrderHistory().getComments();
			}

			// Order, PaymentMethod,
			ProcessorContext context = new ProcessorContext();

			Collection files = oservice.getOrderProductDownloads(order
					.getOrderId());
			if (files != null && files.size() > 0) {
				context.addObject("files", files);

			}

			context.addObject("Order", order);
			context.addObject("Customer", customer);
			context.addObject("MerchantStore", store);
			context.addObject("PaymentMethod", payment);
			context.addObject("Shipping", shipping);
			context.addObject("Locale", super.getLocale());
			context.addObject("OrderTotalSummary", summary);
			context.addObject("comments", comments);
			context.addObject("prroducts", orderProducts.values());

			WorkflowProcessor wp = (WorkflowProcessor) SpringUtil
					.getBean("orderWorkflow");
			wp.doWorkflow(context);

			// set an indicator in HTTPSession to prevent duplicates
			super.getServletRequest().getSession().setAttribute(
					"TRANSACTIONCOMITED", "true");

			if (!StringUtils.isBlank(comments)) {
				SessionUtil.setOrderStatusHistory(this.getOrderHistory(),
						getServletRequest());
			}

		} catch (Exception e) {
			if (e instanceof TransactionException) {
				super.addErrorMessage("error.payment.paymenterror");
				return "PAYMENTERROR";
			}

			if (e instanceof OrderException) {
				try {
					oservice.sendOrderProblemEmail(order.getMerchantId(),
							order, customer, store);
				} catch (Exception ee) {
					log.error(ee);
				}
			}

			addActionError(getText("message.error.comitorder.error",
					new String[] { String.valueOf(order.getOrderId()),
							store.getStoreemailaddress() }));
			log.error(e);
			return "GENERICERROR";
		}
		//cleanup
		
		//delete shopping cart cookie
		Cookie c = new Cookie(CatalogConstants.CART_COOKIE_NAME,"");
		c.setMaxAge(0);
		super.getServletResponse().addCookie(c);

		return SUCCESS;

	}

	public OrderStatusHistory getOrderHistory() {
		return orderHistory;
	}

	public void setOrderHistory(OrderStatusHistory orderHistory) {
		this.orderHistory = orderHistory;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public Collection getDownloadFiles() {
		return downloadFiles;
	}

	public void setDownloadFiles(Collection downloadFiles) {
		this.downloadFiles = downloadFiles;
	}

	public String getFileMessage() {
		return fileMessage;
	}

	public void setFileMessage(String fileMessage) {
		this.fileMessage = fileMessage;
	}

	public Collection getOrderProductList() {
		return orderProductList;
	}

	public void setOrderProductList(Collection orderProductList) {
		this.orderProductList = orderProductList;
	}

	public Collection getTotals() {
		return totals;
	}

	public void setTotals(Collection totals) {
		this.totals = totals;
	}

	public boolean isOrderCommited() {
		return orderCommited;
	}

	public void setOrderCommited(boolean orderCommited) {
		this.orderCommited = orderCommited;
	}

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

}



```
