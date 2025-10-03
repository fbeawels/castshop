# CustomerListAction.java

## Review

## 1. Summary  

**Purpose** – `CustomerListAction` is a Struts‑style action that prepares a paginated list of customers for display on the “customer list” page of a central administration UI.  

**Key components**  

| Class / Interface | Role |
|--------------------|------|
| `CustomerListAction` | The action that fetches data and populates page‑related properties (title, pagination, customer collection). |
| `CustomerService` | Business layer that actually retrieves customer data from the database. |
| `SearchCustomerCriteria` | DTO that encapsulates pagination and filtering parameters. |
| `SearchCustomerResponse` | DTO that carries the list of customers and the total count back to the action. |
| `PageBaseAction` (superclass) | Provides page‑related helpers (`setPageTitle`, `setPageStartNumber`, etc.) used for pagination. |
| `ServiceFactory` | Factory/DI mechanism used to obtain an instance of `CustomerService`. |
| `PropertiesHelper` / `Configuration` | Loads the max‑size setting for the list from the central config file. |

**Notable patterns & libraries**  

* **Factory pattern** – `ServiceFactory.getService(ServiceFactory.CustomerService)` is a classic factory call.  
* **DTOs** – `SearchCustomerCriteria` & `SearchCustomerResponse` are simple data carriers.  
* **Apache Commons Configuration** – For reading the `central.custormerlist.maxsize` property.  
* **Log4j** – For error logging.  

---

## 2. Detailed Description  

### Execution Flow  

1. **Page title** – `setPageTitle("label.customer.customerlist.title")` configures the UI title.  
2. **Merchant context** – The current `Context` object is pulled from the HTTP session (`ProfileConstants.context`).  
3. **Service acquisition** – `CustomerService` is obtained from `ServiceFactory`.  
4. **Criteria preparation**  
   * If `customerSearchCriteria` is null, a new instance is created.  
   * The merchant ID, page size (`customersize`), and start index (`super.getPageStartIndex()`) are set on the criteria.  
5. **Search** – `cservice.searchCustomers(...)` returns a `SearchCustomerResponse`.  
6. **Populate state** –  
   * `setCustomers(...)` stores the list of customers for the view.  
   * Page metadata (`setListingCount`, `setRealCount`, `setPageElements`) is filled in using the response data.  
7. **Error handling** – Any exception is logged, a technical message is set, and `"ERROR"` is returned.  
8. **Success** – On normal completion, `SUCCESS` (defined in `PageBaseAction`) is returned.

### Dependencies & Constraints  

* The action assumes a valid `Context` in the session. A missing or null context will trigger a `NullPointerException`.  
* Pagination is hard‑wired to a static maximum size (`customersize`).  
* Thread safety is *not* guaranteed: `customersize` and `config` are static but immutable after initialization.  
* The action relies on the `PageBaseAction` contract for pagination helpers; if those methods are changed, this class may break.  

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `displayCustomerList()` | Main action entry point; prepares the customer list for rendering. | None | `String` – “SUCCESS” or “ERROR” | Populates pagination properties, fetches customers, sets `customers` collection, logs errors. |
| `getStartIndex()` | Getter for the `startIndex` field. | None | `int` | None |
| `setStartIndex(int)` | Setter for the `startIndex` field. | `int` | None | None |
| `getCustomers()` | Getter for the `customers` collection. | None | `Collection` | None |
| `setCustomers(Collection)` | Setter for the `customers` collection. | `Collection` | None | None |
| `getCustomerSearchCriteria()` | Getter for the search criteria DTO. | None | `SearchCustomerCriteria` | None |
| `setCustomerSearchCriteria(SearchCustomerCriteria)` | Setter for the search criteria DTO. | `SearchCustomerCriteria` | None | None |

*All getters/setters are straightforward JavaBeans conventions used by the view layer.*

---

## 4. Dependencies  

| External Library / API | Usage | Notes |
|------------------------|-------|-------|
| **Apache Commons Configuration** (`org.apache.commons.configuration.Configuration`) | Loads `central.custormerlist.maxsize`. | Standard third‑party lib. |
| **Log4j** (`org.apache.log4j.Logger`) | Error logging. | Standard third‑party lib. |
| **SalesManager Core** | `CustomerService`, `SearchCustomerCriteria`, `SearchCustomerResponse`, `ServiceFactory`. | Domain‑specific, proprietary code. |
| **SalesManager Central** | `PageBaseAction`, `Context`, `ProfileConstants`, `PropertiesHelper`. | Domain‑specific, likely part of the same product. |

*No OS or platform‑specific dependencies are visible.*

---

## 5. Additional Notes  

### Strengths  
* Clear separation between UI (action) and business logic (service).  
* Uses DTOs for request/response, keeping the service contract explicit.  
* Pagination helpers are abstracted in `PageBaseAction`.  

### Weaknesses & Edge Cases  
1. **Null Session Context** – If `ProfileConstants.context` is absent, a `NullPointerException` will occur.  
2. **Hard‑coded Pagination Size** – The static `customersize` makes it impossible to adapt page size per user or per request.  
3. **Raw `Collection`** – Lacks generics; callers have to cast to the actual type, risking `ClassCastException`.  
4. **Unutilised `startIndex` Field** – It is never read; likely a vestigial artifact.  
5. **Static Configuration Loading** – The static block reads from `PropertiesHelper.getConfiguration()` without exception handling; if the property is missing or malformed, class initialization may fail.  
6. **Typo in Config Key** – `"central.custormerlist.maxsize"` – this misspelling could make the default 20 ineffective if the correct key is used elsewhere.  
7. **Exception Handling** – Swallowing all exceptions and only logging them hides the root cause from the user; the technical message may be too generic.  
8. **Thread Safety** – While `customersize` is immutable, the instance fields (`customers`, `customerSearchCriteria`) are mutable and not thread‑safe. In a servlet container, each request creates a new action instance, so this is generally safe, but documenting the lifecycle would help.  

### Recommendations  
* **Inject the `CustomerService`** (e.g., via Spring) instead of using a static factory; this makes unit testing easier.  
* **Use generics**: `private Collection<Customer> customers;` and return a `List<Customer>`.  
* **Remove or repurpose `startIndex`**; if pagination is required, expose it through the criteria DTO instead.  
* **Make page size configurable per request** (e.g., a `pageSize` request parameter) or expose a setter.  
* **Add validation** to ensure `ctx` is not null and that the merchant ID is valid.  
* **Correct the config key typo** and provide a fallback or a configuration validator.  
* **Handle specific exceptions** (e.g., `ServiceException`) rather than catching `Exception`.  
* **Consider using a logging framework that supports MDC** to attach request‑specific context (merchant ID, user ID).  

Overall, the action serves its purpose but could be modernised and hardened for robustness and maintainability.

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

import java.util.Collection;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.central.PageBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.entity.customer.SearchCustomerCriteria;
import com.salesmanager.core.entity.customer.SearchCustomerResponse;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;

public class CustomerListAction extends PageBaseAction {

	private Logger log = Logger.getLogger(CustomerListAction.class);

	static Configuration config = PropertiesHelper.getConfiguration();
	private Collection customers;

	private SearchCustomerCriteria customerSearchCriteria;

	private static int customersize = 1;

	private int startIndex = 0;

	static {

		customersize = config.getInt("central.custormerlist.maxsize", 20);

	}

	public String displayCustomerList() {
		
		super.setPageTitle("label.customer.customerlist.title");
		
		try {
			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			// Collection coll = cservice.getCustomerList(merchantid);

			if (this.getCustomerSearchCriteria() == null) {
				customerSearchCriteria = new SearchCustomerCriteria();
			}

			customerSearchCriteria.setMerchantId(super.getContext()
					.getMerchantid());
			customerSearchCriteria.setQuantity(customersize);

			this.setSize(customersize);
			super.setPageStartNumber();

			customerSearchCriteria.setStartindex(super.getPageStartIndex());
			SearchCustomerResponse response = cservice.searchCustomers(this
					.getCustomerSearchCriteria());

			this.setCustomers(response.getCustomers());

			super.setListingCount(response.getCount());
			super.setRealCount(response.getCustomers().size());
			super.setPageElements();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "ERROR";
		}

		return SUCCESS;

	}

	public int getStartIndex() {
		return startIndex;
	}

	public void setStartIndex(int startIndex) {
		this.startIndex = startIndex;
	}

	public Collection getCustomers() {
		return customers;
	}

	public void setCustomers(Collection customers) {
		this.customers = customers;
	}

	public SearchCustomerCriteria getCustomerSearchCriteria() {
		return customerSearchCriteria;
	}

	public void setCustomerSearchCriteria(
			SearchCustomerCriteria customerSearchCriteria) {
		this.customerSearchCriteria = customerSearchCriteria;
	}

}



```
