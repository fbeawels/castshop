# GetCustomer.java

## Review

## 1. Summary
The `GetCustomer` class is a simple service‑like component that retrieves a `Customer` entity from the persistence layer, enriches it with locale‑aware reference data (countries and zones), and ties it to the current HTTP session.  
Key responsibilities:

1. **Customer retrieval** – Fetches a customer by ID using `CustomerService`.
2. **Reference enrichment** – Looks up country and zone names via `RefCache` and injects them into the customer DTO.
3. **Session coordination** – Stores the enriched customer back into the HTTP session and, if an `Order` is already present, associates the customer with that order.
4. **Locale handling** – Sets the customer’s locale to match the request locale.

The code relies on the *DWR* (`WebContextFactory`) to access the servlet request, and uses a number of internal services (`ServiceFactory`, `RefCache`, `LanguageUtil`) that are part of the SalesManager core library.

## 2. Detailed Description
### Flow of Execution
1. **Request acquisition** – `WebContextFactory.get().getHttpServletRequest()` pulls the current `HttpServletRequest`.
2. **Context retrieval** – From the session the `Context` bean (which holds language information) is fetched.
3. **Service lookup** – `CustomerService` is obtained from a factory, then used to load the customer by parsed `customerId`.
4. **Reference data lookup**  
   * `RefCache.getAllcountriesmap()` and `getAllZonesmap()` provide language‑specific maps of `Country` and `Zone`.  
   * The maps are queried using the customer’s stored country and zone IDs.
5. **DTO enrichment** – If the country or zone is found, the customer’s name fields are populated.
6. **Session side‑effects** –  
   * The customer object is stored under the key `"CUSTOMER"`.  
   * If an `Order` object exists in the session, its `customerId` field is updated.
7. **Locale assignment** – The customer’s locale is set to the request locale.
8. **Return** – The enriched customer (or a new blank one on error) is returned.

### Assumptions & Constraints
- The HTTP session must contain a `Context` bean with a language setting; otherwise a `NullPointerException` will occur.
- `customerId` must be a string representation of a long; invalid format throws `NumberFormatException` (caught generically).
- `RefCache` must be pre‑populated with the required reference data; missing entries result in null names but no exception.
- Only the billing country/zone fields are populated; shipping or other addresses are ignored.

### Design Choices
- **Imperative style** – All logic is performed in a single method; no layering or DTO conversion utilities.
- **Session coupling** – The method mutates the session, which may make unit testing harder and introduces hidden side‑effects.
- **Error handling** – A broad `catch (Exception e)` swallows all exceptions, logs them, and returns an empty customer; callers cannot distinguish between a “not found” and an internal error.

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `Customer getCustomerByCustomerId(String customerId)` | Loads a customer by ID, enriches it, stores it in the session, associates it with an order if present, and sets the locale. | `customerId` – string representation of the customer’s numeric ID. | Returns the populated `Customer` instance; if anything fails, returns a new empty `Customer`. | • Writes the customer to the session under `"CUSTOMER"`. <br>• Updates an existing `Order` in the session by setting `customerId`. <br>• Sets the locale on the customer. <br>• Logs any exception. |

### Utility methods
The class does not expose any reusable utility methods; all work is done in the single public method.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard J2EE | Servlet API |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `uk.ltd.getahead.dwr.WebContextFactory` | Third‑party | DWR (Direct Web Remoting) to obtain the current request |
| `com.salesmanager.central.profile.Context`, `ProfileConstants` | Internal | Holds session context (language) |
| `com.salesmanager.core.entity.customer.Customer` | Internal | Domain model |
| `com.salesmanager.core.entity.orders.Order` | Internal | Domain model |
| `com.salesmanager.core.entity.reference.Country`, `Zone` | Internal | Reference data models |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Service locator/factory |
| `com.salesmanager.core.service.cache.RefCache` | Internal | Cache of reference data |
| `com.salesmanager.core.service.customer.CustomerService` | Internal | Business service |
| `com.salesmanager.core.util.LanguageUtil` | Internal | Utility for language codes |

No platform‑specific or external APIs beyond DWR and Log4J are used. The code assumes the existence of a fully initialized Spring‑style context for `ServiceFactory` and that the session contains the expected attributes.

## 5. Additional Notes
### Edge Cases & Limitations
- **Null session attributes** – If the session lacks `"ORDER"` or `ProfileConstants.context`, a `NullPointerException` will propagate before the catch block, resulting in an empty customer being returned (silent failure).
- **Thread‑safety** – The method is stateless except for session manipulation, but the use of `RefCache` assumes thread‑safe reads; this is fine if `RefCache` is immutable or read‑only after initialization.
- **Internationalization** – Only the billing country/zone names are localized; other address fields may remain in the default language.
- **Error granularity** – Swallowing all exceptions hides the root cause from callers; a more fine‑grained exception strategy could be beneficial.

### Potential Enhancements
1. **Separate concerns** – Extract reference enrichment and order association into dedicated methods or services to reduce method size and improve testability.
2. **Session handling abstraction** – Pass a `SessionFacade` or use dependency injection to avoid direct `HttpSession` manipulation, facilitating unit tests.
3. **Improved error handling** – Throw custom exceptions (`CustomerNotFoundException`, `ReferenceDataException`) and let higher layers decide on fallback behavior.
4. **DTO mapping** – Use a mapper (e.g., MapStruct) to copy properties and enrich data, keeping business logic out of the service method.
5. **Caching improvements** – Consider using a read‑through cache or a dedicated service for country/zone lookups to encapsulate caching logic.

Overall, the class accomplishes its core task but could benefit from better separation of concerns, clearer error handling, and improved testability.

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
package com.salesmanager.central.customer;

import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.LanguageUtil;

public class GetCustomer {

	private Logger log = Logger.getLogger(GetCustomer.class);


	public Customer getCustomerByCustomerId(String customerId) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		Context ctx = (Context) req.getSession().getAttribute(
				ProfileConstants.context);

		try {

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);

			Customer c = cservice.getCustomer(Long.parseLong(customerId));

			if (c == null) {
				c = new Customer();
			}

			Map countries = RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));
			Map zones = RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));
			Country country = (Country) countries.get(c
					.getCustomerBillingCountryId());
			Zone zone = (Zone) zones.get(c.getCustomerBillingZoneId());
			if (country != null) {
				c.setCountryName(country.getCountryName());
				c.setCustomerBillingCountryName(country.getCountryName());
			}
			if (zone != null) {
				c.setCustomerBillingState(zone.getZoneName());
				c.setStateProvinceName(zone.getZoneName());
				c.setCustomerState(zone.getZoneName());
			}

			req.getSession().setAttribute("CUSTOMER", c);

			// associate the customer to the current order
			Order o = (Order) req.getSession().getAttribute("ORDER");
			if (o != null) {
				o.setCustomerId(c.getCustomerId());
			}

			c.setLocale(req.getLocale());
			return c;
		} catch (Exception e) {
			log.error(e);
			return new Customer();
		}

	}

}


```
