# ShippingAction.java

## Review

## 1. Summary  

**Purpose**  
`ShippingAction` is a Struts‑2 action that drives the shipping‑selection stage of a checkout flow. It:
- Generates a list of shipping options (`displayShipping`) based on the current cart, customer and store.
- Persists those options in the HTTP session for later retrieval.
- Accepts the user’s chosen shipping option (`selectShipping`), updates the order totals, and stores the final shipping data back in the session.

**Key components**
| Component | Role |
|-----------|------|
| `ShippingInformation` | Holds overall quote information (handling cost, available methods, selected option). |
| `ShippingMethod` & `ShippingOption` | Domain objects representing a shipping provider and its individual rate options. |
| `SessionUtil` | Convenience wrapper around `HttpSession` for storing/retrieving order/cart data. |
| `ShippingService` | Service that communicates with the back‑end (or external provider) to obtain shipping quotes. |
| `CheckoutBaseAction` | Base class that supplies common utilities (e.g., `updateOrderTotal`, `addFieldError`, `setTechnicalMessage`). |

The code follows a typical **Model‑View‑Controller** pattern used by Struts‑2: the action manipulates model objects, stores them in the session, and returns navigation results (`SUCCESS`, `INPUT`, `"GENERICERROR"`).

## 2. Detailed Description  

### Execution Flow

| Step | Method | What Happens |
|------|--------|--------------|
| **1. Display** | `displayShipping()` | 1. Sets a request attribute `STEP = 2`. 2. Pulls the cart (`products`), customer and order from the session. 3. Builds a `List` of `OrderProduct` objects (`prodArray`). 4. Calls `ShippingService.getShippingQuote` to receive a `ShippingInformation` instance. 5. Extracts the `ShippingMethod` collection, and for each method & option: <ul><li>Sets the module name on the option (so it can be rendered later).</li><li>Stores the option in a `Map` keyed by `optionId`.</li><li>If the method’s priority is `0`, remembers that option as the default (`shippingOption`).</li></ul> 6. Persists the map and the full `ShippingInformation` back into the session. 7. Returns `SUCCESS` (Struts will forward to the shipping JSP). |
| **2. Selection** | `selectShipping()` | 1. Validates that a shipping option was submitted (`shippingOption` and its `optionId`). 2. Retrieves the stored map of options from the session. 3. Looks up the chosen option, verifies it exists. 4. Builds a `Shipping` domain object populated with handling cost, cost and description. 5. Updates the `ShippingInformation` with the selected option and stores it again. 6. Re‑fetches the order, store and customer, rebuilds a list of `OrderProduct`s, and calls `updateOrderTotal` (inherited from `CheckoutBaseAction`) to recalculate totals with the new shipping cost. 7. Returns `SUCCESS`. |

### Assumptions & Constraints  

- The session always contains a valid order, customer, and merchant store.  
- `ShippingService.getShippingQuote` is synchronous and returns a fully populated `ShippingInformation`.  
- `updateOrderTotal` correctly handles tax/discount logic.  
- The front‑end sends back a `ShippingOption` instance (or at least its `optionId`).  
- No multi‑threaded access to the session is considered; each request is isolated.

### Architecture  

The class is a **stateless service** except for the small amount of data it keeps as fields that are populated per request.  
It relies on **dependency injection** via the static `ServiceFactory`, which hides the creation of `ShippingService`.  
All business logic remains in domain services (`ShippingService`, `CheckoutBaseAction.updateOrderTotal`); the action merely orchestrates data flow.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `displayShipping()` | Prepares shipping options for the user | N/A (reads from session) | `SUCCESS` string or `"GENERICERROR"` | Sets request attribute `STEP`. Stores shipping methods and full `ShippingInformation` in session. |
| `selectShipping()` | Handles the user’s choice of shipping method | `this.shippingOption` (populated by Struts form binding) | `SUCCESS` or `"GENERICERROR"` | Updates session with selected shipping option, recalculates order totals. |
| `getShippingInformation()` / `setShippingInformation()` | Accessor for `shippingInformation` | N/A | `ShippingInformation` | None |
| `getShippingMethods()` / `setShippingMethods()` | Accessor for available methods | N/A | `Collection<ShippingMethod>` | None |
| `getShippingOption()` / `setShippingOption()` | Accessor for the user‑selected option | N/A | `ShippingOption` | None |

### Reusable / Utility Methods  
- **`SessionUtil`**: Static helpers for session storage – used repeatedly for cart, order, shipping info, etc.  
- **`StringUtils.isBlank()`**: Used for validating `optionId`.

## 4. Dependencies  

| Dependency | Type | Notes |
|-------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | For safe string checks. |
| `org.apache.log4j.Logger` | Third‑party (Log4j) | For logging errors. |
| `com.salesmanager.core.service.ServiceFactory` | Third‑party | Static factory for service instances. |
| `com.salesmanager.core.service.shipping.ShippingService` | Third‑party | Core business logic for shipping quotes. |
| `com.salesmanager.core.util.www.SessionUtil` | Third‑party | Session convenience wrapper. |
| `com.salesmanager.core.entity.*` | Domain entities | Customer, MerchantStore, Order, Shipping, ShippingInformation, ShippingMethod, ShippingOption. |
| `com.salesmanager.checkout.CheckoutBaseAction` | Local | Base class providing common utilities. |

All dependencies are either part of the SalesManager core library or well‑known open‑source components. No platform‑specific code is visible.

## 5. Additional Notes  

### Strengths  
- Clear separation of concerns: action orchestrates, services perform business logic.  
- Uses Struts‑2 conventions (`SUCCESS`, `INPUT`, `"GENERICERROR"`).  
- Session data is centralized via `SessionUtil`, making it easier to maintain.

### Weaknesses & Risks  
1. **Raw types** – `Map`, `List`, `Collection` are used without generics (e.g., `Map products = SessionUtil.getOrderProducts(...)`). This leads to unchecked casts and potential `ClassCastException`.  
2. **Error handling** – `displayShipping()` catches `Exception` and silently returns `"GENERICERROR"`. This swallows runtime errors and makes debugging harder. Likewise `selectShipping()` declares `throws Exception` but never actually throws one.  
3. **Concurrent session usage** – The action assumes a single request per session. If a user opens multiple tabs, the stored shipping map may be overwritten or read incorrectly.  
4. **Hard‑coded default logic** – `shippingOption` is set only when `sm.getPriority() == 0`. This logic is embedded in the action and could become fragile if the business rule changes.  
5. **Duplicate code** – Rebuilding the `List<OrderProduct>` from the `Map` occurs in both methods. Refactor into a helper.  
6. **No JavaDoc** – Methods lack documentation; future contributors may struggle to understand intent.  

### Edge Cases  
- No shipping methods returned → `shippingMethods` stays null → UI may not render options.  
- `shippingOption` is null or has an empty `optionId` → user receives `INPUT` but the underlying error isn’t logged.  
- If the shipping quote returns a `null` `ShippingInformation`, the action will throw a `NullPointerException` when trying to access `getShippingMethods()`.

### Suggested Enhancements  
1. **Add Generics** – Update all collections to use typed generics.  
2. **Improve Logging** – Log the exception stack trace and context details when a failure occurs.  
3. **Encapsulate Session Logic** – Move session storage/retrieval into dedicated methods or a SessionDTO.  
4. **Separate Validation** – Create a validator method for shipping option selection to keep `selectShipping()` focused.  
5. **Unit Tests** – Add tests for `displayShipping()` and `selectShipping()` using a mocked `HttpServletRequest`/session.  
6. **Internationalization** – Extract all hard‑coded strings into resource bundles; some are already done (`getText`).  
7. **Error Result Mapping** – Use a dedicated result (`"error"` or `"shippingError"`) instead of `"GENERICERROR"` for better navigation handling.

---  

Overall, `ShippingAction` provides the necessary plumbing for shipping selection but would benefit from type safety, clearer error handling, and refactoring to reduce duplication and improve maintainability.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.entity.shipping.ShippingMethod;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.www.SessionUtil;

public class ShippingAction extends CheckoutBaseAction {

	private Logger log = Logger.getLogger(ShippingAction.class);

	private ShippingInformation shippingInformation;
	private Collection<ShippingMethod> shippingMethods;

	private ShippingOption shippingOption;

	public String displayShipping() {

		try {

			super.getServletRequest().setAttribute("STEP", 2);

			// get shopping cart products
			Map products = SessionUtil.getOrderProducts(super
					.getServletRequest());

			Customer customer = SessionUtil.getCustomer(super
					.getServletRequest());
			Order o = SessionUtil.getOrder(super.getServletRequest());

			List prodArray = new ArrayList(products.values());

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			ShippingService sservice = (ShippingService) ServiceFactory
					.getService(ServiceFactory.ShippingService);

			ShippingInformation shippingInfo = sservice.getShippingQuote(
					prodArray, customer, store.getMerchantId(), super
							.getLocale(), store.getCurrency());

			shippingInformation = shippingInfo;
			shippingMethods = shippingInfo.getShippingMethods();

			// must retain shipping methods proposed

			if (shippingMethods != null) {
				// cached map
				Map methodMap = new HashMap();
				Iterator i = shippingMethods.iterator();
				while (i.hasNext()) {
					ShippingMethod sm = (ShippingMethod) i.next();
					String module = sm.getShippingModule();
					Collection options = sm.getOptions();
					Iterator opIter = options.iterator();

					while (opIter.hasNext()) {
						ShippingOption option = (ShippingOption) opIter.next();
						if (sm.getPriority() == 0) {
							shippingOption = new ShippingOption();
							shippingOption = option;
						}
						option.setModule(module);
						methodMap.put(option.getOptionId(), option);
					}
				}

				// shipping options available
				SessionUtil.setShippingMethods(methodMap, super
						.getServletRequest());
				// merchant shipping information stored in http session
				SessionUtil.setShippingInformation(shippingInfo, super
						.getServletRequest());

			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;

	}

	public String selectShipping() throws Exception {

		if (this.getShippingOption() == null) {
			super.addFieldError("shipping",
					getText("messages.required.shippingmethod"));
			return INPUT;
		}

		if (StringUtils.isBlank(this.getShippingOption().getOptionId())) {
			super.addFieldError("shipping",
					getText("messages.required.shippingmethod"));
			return INPUT;
		}

		Map shippingOptionsMap = SessionUtil
				.getShippingMethods(getServletRequest());

		if (shippingOptionsMap == null || shippingOptionsMap.size() == 0) {
			super.setTechnicalMessage();
			log.error("No shipping options Map to select");
			return "GENERICERROR";
		}

		ShippingInformation shippingInformation = SessionUtil
				.getShippingInformation(getServletRequest());
		shippingInformation.setShippingOptionSelected(this.getShippingOption());

		ShippingOption opt = (ShippingOption) shippingOptionsMap.get(this
				.getShippingOption().getOptionId());
		if (opt == null) {
			super.setTechnicalMessage();
			log.error("No shipping option to select for optionId "
					+ this.getShippingOption().getOptionId());
			return "GENERICERROR";
		}

		Shipping shipping = new Shipping();
		shipping.setHandlingCost(shippingInformation.getHandlingCost());
		shipping.setShippingCost(opt.getOptionPrice());
		shipping.setShippingModule(opt.getModule());
		shipping.setShippingDescription(opt.getDescription());

		shippingInformation.setShippingCost(opt.getOptionPrice());
		shippingInformation.setShippingOptionSelected(opt);

		SessionUtil.setShippingInformation(shippingInformation,
				getServletRequest());

		Order order = SessionUtil.getOrder(getServletRequest());
		MerchantStore store = SessionUtil.getMerchantStore(getServletRequest());
		Customer customer = SessionUtil.getCustomer(getServletRequest());

		Map orderProducts = SessionUtil.getOrderProducts(getServletRequest());
		List products = new ArrayList();
		if (orderProducts != null) {
			Iterator i = orderProducts.keySet().iterator();
			while (i.hasNext()) {
				String line = (String) i.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				products.add(op);
			}
		}

		// update order with tax if it applies
		super.updateOrderTotal(order, products, customer, shipping, store);

		return SUCCESS;

	}

	public ShippingInformation getShippingInformation() {
		return shippingInformation;
	}

	public void setShippingInformation(ShippingInformation shippingInformation) {
		this.shippingInformation = shippingInformation;
	}

	public Collection<ShippingMethod> getShippingMethods() {
		return shippingMethods;
	}

	public void setShippingMethods(Collection<ShippingMethod> shippingMethods) {
		this.shippingMethods = shippingMethods;
	}

	public ShippingOption getShippingOption() {
		return shippingOption;
	}

	public void setShippingOption(ShippingOption shippingOption) {
		this.shippingOption = shippingOption;
	}

}



```
