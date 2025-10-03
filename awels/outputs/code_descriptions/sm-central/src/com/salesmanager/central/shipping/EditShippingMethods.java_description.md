# EditShippingMethods.java

## Review

## 1. Summary  
The **`EditShippingMethods`** action class is part of a larger e‑commerce application that calculates shipping options for a customer’s order.  
* **Purpose:**  
  * Retrieve the current order, customer and cart information from the HTTP session.  
  * Query the shipping service for a quote based on the cart contents and the customer’s location.  
  * Store the resulting shipping methods and options back into the session for subsequent use in the checkout flow.  
* **Key components:**  
  * `SessionUtil` – helper for session‑level persistence of orders, customers, and shipping data.  
  * `ShippingService` – domain service that performs the actual shipping quote calculation.  
  * `ShippingInformation`, `ShippingMethod`, `ShippingOption` – value objects that model the quote.  
* **Design pattern / framework usage:**  
  * The class extends `BaseAction`, indicating it is a Struts‑2 action (though the code itself is fairly framework‑agnostic).  
  * Dependency resolution via a generic `ServiceFactory`.  
  * Logging with Log4j.

## 2. Detailed Description  

### Flow of execution
1. **Context Acquisition**  
   * `Context ctx = super.getContext();` retrieves merchant‑specific settings (merchant id, currency).  
2. **Session Data Retrieval**  
   * `products`, `customer`, and `Order` are fetched from the HTTP session using `SessionUtil`.  
   * If the customer is not already present, it is loaded from the database via `CustomerService`.  
3. **Pre‑condition**  
   * The code expects a non‑null `customer` – the comment hints at an error return if it is missing, but the actual implementation silently continues.  
4. **Quote Calculation**  
   * `ShippingService.getShippingQuote()` is called with the product list, customer, merchant id, locale, and currency.  
   * The resulting `ShippingInformation` contains a collection of `ShippingMethod` objects, each with a set of `ShippingOption` objects.  
5. **Session Storage**  
   * The code iterates over all methods and options, adds the method module to each option, and builds a map of `optionId → ShippingOption`.  
   * Both the map and the full `ShippingInformation` are stored back into the session via `SessionUtil`.  
6. **Return**  
   * The method always returns `SUCCESS` (a constant from `BaseAction`), regardless of any errors that may have occurred.  

### Assumptions & Constraints  
* The session must already contain a valid `Order` and cart products.  
* `ShippingService` and `CustomerService` are available through `ServiceFactory`.  
* All objects are serializable or otherwise safe to store in an HTTP session.  
* The application uses a single-threaded action per request, so no additional synchronization is required.

### Architecture & Design Choices  
* The action tightly couples session handling with business logic – common in legacy Struts 2 applications but less testable.  
* Services are obtained via a static factory, which hides configuration details but introduces global state and makes mocking difficult.  
* Error handling is minimal – a single `catch (Exception e)` logs the problem but still returns success, potentially masking failures from the UI.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `displayShippingMethods()` | Main action entry point. Calculates and persists shipping options. | None (relies on session data). | Sets instance variables `shippingInformation` and `shippingMethods`. Stores data in session. Returns Struts result string `SUCCESS`. |
| `getShippingMethods()` | Getter for the `shippingMethods` collection. | None | Returns `Collection<ShippingMethod>` |
| `setShippingMethods(Collection<ShippingMethod>)` | Setter for `shippingMethods`. | `Collection<ShippingMethod>` | Sets instance variable |
| `getShippingInformation()` | Getter for the `shippingInformation` object. | None | Returns `ShippingInformation` |
| `setShippingInformation(ShippingInformation)` | Setter for `shippingInformation`. | `ShippingInformation` | Sets instance variable |

### Reusable / Utility Methods  
* `displayShippingMethods()` does most of the heavy lifting; no dedicated helper methods are defined, so refactoring to extract logic would improve readability.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party (Log4j) | Classic logging framework. |
| `com.salesmanager.central.BaseAction` | Framework | Likely a Struts 2 base action providing context/session helpers. |
| `com.salesmanager.central.profile.Context` | Framework | Holds merchant‑level configuration. |
| `com.salesmanager.core.entity.*` | Domain | POJOs for Customer, Order, Shipping entities. |
| `com.salesmanager.core.service.*` | Domain | Service interfaces for customer and shipping operations. |
| `com.salesmanager.core.util.www.SessionUtil` | Utility | Session‑level persistence helper. |
| `ServiceFactory` | Custom factory | Static method to obtain services; hides DI framework. |

No platform‑specific libraries beyond the standard Java EE web stack (servlet API, Struts 2).

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Null Customer After Load** – If `cservice.getCustomer(customerId)` returns null, the code silently proceeds, potentially causing a `NullPointerException` when passing `customer` to `getShippingQuote`.  
2. **Exception Swallowing** – All exceptions are logged but the method still returns `SUCCESS`. The UI will receive a success response even if the shipping calculation failed.  
3. **Session Overwrite** – Existing session attributes for shipping methods are overwritten without validation; stale data could leak if the user changes cart items but the session is not refreshed.  
4. **Type Safety** – Raw collections (`Map`, `List`) are used without generics, leading to unchecked casts and potential `ClassCastException`.  
5. **Scalability** – Storing the entire `ShippingInformation` and a map of all options in the HTTP session could consume significant memory for large carts or many shipping options.

### Suggested Improvements  
* **Introduce Validation** – Return an error result if the customer or order is missing or if the shipping quote fails.  
* **Use Generics** – Replace raw `Map`, `List`, and `Collection` types with typed versions (`Map<Long, ShippingOption>`, `List<ShippingMethod>`, etc.).  
* **Dependency Injection** – Replace `ServiceFactory.getService` with constructor injection (e.g., via Spring) to improve testability.  
* **Refactor Logic** – Extract shipping calculation and session storage into separate service or helper classes to reduce coupling.  
* **Session Cleanup** – Add a method to clear old shipping data when the cart changes.  
* **Error Handling** – Throw a custom checked exception from the action and map it to an error page instead of always returning success.  

Overall, the class achieves its basic goal of retrieving and storing shipping options but would benefit from modernizing its design, improving robustness, and enhancing testability.

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
package com.salesmanager.central.shipping;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.entity.shipping.ShippingMethod;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.www.SessionUtil;

public class EditShippingMethods extends BaseAction {

	private Logger log = Logger.getLogger(EditShippingMethods.class);

	private ShippingInformation shippingInformation;

	private Collection<ShippingMethod> shippingMethods;

	/**
	 * Calculates packing Get RT shipping method when configured Get Custom
	 * shipping quote when configured
	 * 
	 * @return
	 */
	public String displayShippingMethods() {

		Context ctx = super.getContext();

		try {

			// get shopping cart products
			Map products = SessionUtil.getOrderProducts(super
					.getServletRequest());

			Customer customer = SessionUtil.getCustomer(super
					.getServletRequest());

			Order o = SessionUtil.getOrder(super.getServletRequest());
			if (customer == null) {

				long customerId = o.getCustomerId();
				if (customerId > 0) {
					CustomerService cservice = (CustomerService) ServiceFactory
							.getService(ServiceFactory.CustomerService);
					customer = cservice.getCustomer(customerId);
				}
			}

			// customer should not be null
			// return error

			BigDecimal total = o.getTotal();

			List prodArray = new ArrayList(products.values());

			ShippingService sservice = (ShippingService) ServiceFactory
					.getService(ServiceFactory.ShippingService);

			ShippingInformation shippingInfo = sservice.getShippingQuote(
					prodArray, customer, ctx.getMerchantid(),
					super.getLocale(), ctx.getCurrency());

			shippingInformation = shippingInfo;
			shippingMethods = shippingInfo.getShippingMethods();

			// must retain shipping methods proposed
			if (shippingMethods != null) {
				Map methodMap = new HashMap();
				Iterator i = shippingMethods.iterator();
				while (i.hasNext()) {
					ShippingMethod sm = (ShippingMethod) i.next();
					String module = sm.getShippingModule();
					Collection options = sm.getOptions();
					Iterator opIter = options.iterator();
					while (opIter.hasNext()) {
						ShippingOption option = (ShippingOption) opIter.next();
						option.setModule(module);
						methodMap.put(option.getOptionId(), option);
					}
				}

				// shipping options available
				SessionUtil.setShippingMethods(methodMap, super
						.getServletRequest());
				// merchant shipping information stored in http session
				SessionUtil.setShippingInformation(shippingInformation, super
						.getServletRequest());

			}

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public Collection<ShippingMethod> getShippingMethods() {
		return shippingMethods;
	}

	public void setShippingMethods(Collection<ShippingMethod> shippingMethods) {
		this.shippingMethods = shippingMethods;
	}

	public ShippingInformation getShippingInformation() {
		return shippingInformation;
	}

	public void setShippingInformation(ShippingInformation shippingInformation) {
		this.shippingInformation = shippingInformation;
	}

}



```
