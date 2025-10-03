# OrdersAction.java

## Review

## 1. Summary  

**Purpose**  
`OrdersAction` is a Struts‑style action class that retrieves a paginated list of orders for the currently logged‑in customer. It pulls configuration from a properties file, builds search criteria, calls an order service, and exposes the resulting collection to the view layer.  

**Key Components**  
- **PageBaseAction** – Base class providing paging helpers (`setSize`, `setPageStartNumber`, `setListingCount`, etc.).  
- **OrderService** – Service layer responsible for searching orders.  
- **SearchOrdersCriteria / SearchOrderResponse** – DTOs used to pass query parameters and receive results.  
- **SessionUtil** – Utility for retrieving the current `MerchantStore` and `Customer` from the HTTP session.  

**Design Patterns & Libraries**  
- **Service Locator** (`ServiceFactory.getService`) is used to obtain the `OrderService`.  
- **DTO pattern** – `SearchOrdersCriteria` and `SearchOrderResponse` encapsulate request/response data.  
- **Apache Commons Configuration** – for reading the `catalog.orderlist.maxsize` property.  
- **Apache Log4j** – for logging.  

## 2. Detailed Description  

1. **Static Initialization**  
   - The static block reads `catalog.orderlist.maxsize` from the configuration file, defaulting to `10`.  
   - The value is stored in the static field `size` and used as the page size for the order list.

2. **Action Flow (`displayOrders`)**  
   - `setSize(size)` sets the number of records per page.  
   - `setPageStartNumber()` calculates the current page’s start index (implementation in `PageBaseAction`).  
   - `getCriteria(pageStartIndex)` constructs a `SearchOrdersCriteria` object populated with:  
     - Date range: from 12 months ago to now (though the month calculation `calendar.set(Calendar.MONTH, -12)` is incorrect – it sets the month field to a negative value instead of subtracting 12 months).  
     - Language ID, merchant ID, customer ID, quantity (`size`), and start index.  
   - `ordersQuery(criteria)` invokes the `OrderService` to retrieve matching orders.  
   - If a response is obtained, it assigns the order collection, localizes the entities, and updates paging metadata (`setListingCount`, `setRealCount`, `setPageElements`).  
   - Exceptions are logged and a generic technical error message is set.

3. **Cleanup**  
   - No explicit cleanup; the action is stateless apart from the `orders` field.

**Assumptions & Constraints**  
- The request/response cycle is managed by the surrounding MVC framework (likely Struts).  
- The `SessionUtil` helpers reliably return non‑null `MerchantStore` and `Customer`.  
- `SearchOrdersCriteria` expects a `Calendar` month value to be manipulated directly – the current implementation is fragile.  
- Paging calculations rely on `PageBaseAction` utilities.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `displayOrders()` | Entry point for the action; prepares paging, builds criteria, queries orders. | None | `String` (SUCCESS) | Sets paging fields, populates `orders` collection, logs errors. |
| `ordersQuery(SearchOrdersCriteria)` | Delegates to `OrderService` to fetch orders. | `SearchOrdersCriteria` | None | Updates `orders`, localizes collection, updates paging metadata. |
| `getCriteria(int startIndex)` | Builds a `SearchOrdersCriteria` for the current customer and merchant. | `int` start index | `SearchOrdersCriteria` | None |
| `getOrders()` | Getter for the retrieved order collection. | None | `Collection` | None |
| `setOrders(Collection)` | Setter for the order collection (unused in normal flow). | `Collection` | None | Sets the field. |
| `getOrderId()` / `setOrderId(String)` | Accessors for a single order ID (unused in current logic). | `String` | `String` / None | Sets the field. |

**Reusable/Utility Methods**  
- `getCriteria()` could be extracted into a helper service as it performs session lookups and date calculations that may be reused.  

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads configuration properties. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.common.PageBaseAction` | Project | Base action providing paging utilities. |
| `com.salesmanager.common.util.PropertiesHelper` | Project | Provides the `Configuration` instance. |
| `com.salesmanager.core.entity.customer.Customer` | Project | Domain entity. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Project | Domain entity. |
| `com.salesmanager.core.entity.orders.SearchOrderResponse` / `SearchOrdersCriteria` | Project | DTOs for order search. |
| `com.salesmanager.core.service.ServiceFactory` | Project | Service locator. |
| `com.salesmanager.core.service.order.OrderService` | Project | Business service for orders. |
| `com.salesmanager.core.util.LanguageUtil` | Project | Language ID mapping. |
| `com.salesmanager.core.util.LocaleUtil` | Project | Localizes entity collections. |
| `com.salesmanager.core.util.www.SessionUtil` | Project | Retrieves session objects. |

All dependencies are internal to the `salesmanager` project except for Commons Configuration, Log4j, and Java standard library classes.  

## 5. Additional Notes  

### Edge Cases & Issues  
1. **Month Calculation** – `calendar.set(Calendar.MONTH, -12);` does **not** subtract 12 months; it sets the month field to `-12`, which results in an invalid date. The correct approach would be `calendar.add(Calendar.MONTH, -12);`.  
2. **Null Handling** – The code assumes that `SessionUtil.getCustomer` and `SessionUtil.getMerchantStore` never return null. If they do, a `NullPointerException` will surface. Defensive checks or error handling would improve robustness.  
3. **Thread Safety** – `orders` is an instance field; Struts actions are typically reused per request, so this is safe. However, if the action is ever pooled, care must be taken to clear state.  
4. **Logging Granularity** – `log.error(e)` prints the stack trace but does not include contextual information (e.g., customer ID). Adding such context can aid debugging.  
5. **Unused Fields** – `orderId` and its accessors are present but never used in the displayed code. If not required, they could be removed to simplify the class.  

### Potential Enhancements  
- **Service Injection** – Replace the `ServiceFactory` lookup with dependency injection (e.g., Spring) for easier testing.  
- **Error Messages** – Provide more user‑friendly error handling rather than a generic technical message.  
- **Pagination Logic** – Move paging calculations to a dedicated utility/service to avoid code duplication across actions.  
- **Unit Tests** – Add tests for `getCriteria` to verify correct date range computation.  
- **Logging Context** – Include customer ID, merchant ID, and request parameters in logs.  

Overall, the action fulfills its core responsibility but would benefit from minor refactoring, especially around date handling and defensive programming.

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
package com.salesmanager.customer.profile;

import java.util.Calendar;
import java.util.Collection;
import java.util.Date;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.common.PageBaseAction;
import com.salesmanager.common.util.PropertiesHelper;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.SearchOrderResponse;
import com.salesmanager.core.entity.orders.SearchOrdersCriteria;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class OrdersAction extends PageBaseAction {

	private Logger log = Logger.getLogger(OrdersAction.class);
	private static Configuration config = PropertiesHelper.getConfiguration();

	private Collection orders;

	private String orderId;

	private static int size = 20;

	static {

		size = config.getInt("catalog.orderlist.maxsize", 10);

	}

	public String displayOrders() {

		try {

			super.setSize(size);// defined in configuration according to
								// template
			super.setPageStartNumber();

			SearchOrdersCriteria crit = getCriteria(super.getPageStartIndex());
			ordersQuery(crit);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	private void ordersQuery(SearchOrdersCriteria criteria) throws Exception {

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		SearchOrderResponse resp = oservice.searchOrdersByCustomer(criteria);
		if (resp != null) {
			orders = resp.getOrders();

			LocaleUtil.setLocaleToEntityCollection(orders, super.getLocale());

			super.setListingCount(resp.getCount());
			super.setRealCount(orders.size());
			super.setPageElements();

		}

	}

	private SearchOrdersCriteria getCriteria(int startIndex) {

		MerchantStore store = SessionUtil.getMerchantStore(super
				.getServletRequest());

		SearchOrdersCriteria criteria = new SearchOrdersCriteria();

		Customer customer = SessionUtil.getCustomer(super.getServletRequest());

		// 12 months
		Calendar calendar = Calendar.getInstance();
		calendar.setTime(new Date());
		calendar.set(Calendar.MONTH, -12);

		criteria.setSdate(calendar.getTime());
		criteria.setEdate(new Date());

		criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(super
				.getLocale().getLanguage()));
		criteria.setMerchantId(store.getMerchantId());
		criteria.setCustomerId(customer.getCustomerId());
		criteria.setQuantity(size);
		criteria.setStartindex(startIndex);

		return criteria;

	}

	public Collection getOrders() {
		return orders;
	}

	public void setOrders(Collection orders) {
		this.orders = orders;
	}

	public String getOrderId() {
		return orderId;
	}

	public void setOrderId(String orderId) {
		this.orderId = orderId;
	}

}



```
