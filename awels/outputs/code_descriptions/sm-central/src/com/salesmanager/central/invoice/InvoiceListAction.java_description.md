# InvoiceListAction.java

## Review

## 1. Summary
`InvoiceListAction` is a Struts‑2 action that powers the “invoice list” page in the Central module of the SalesManager application.  
- **Purpose** – Load a paginated list of invoices for the currently logged‑in merchant, optionally filtered by date range or invoice number.  
- **Key components**  
  - **`PageBaseAction`** – Provides paging helpers (`setPageStartNumber`, `setListingCount`, etc.).  
  - **`OrderService`** – Service layer responsible for querying the database (`searchInvoices`).  
  - **`SearchOrdersCriteria`** – Value object that encapsulates paging, filtering, and merchant‑specific search parameters.  
  - **`Context`** – Holds session‑level data such as the logged‑in merchant ID.  
- **Design patterns / frameworks**  
  - **MVC** – Struts‑2 framework.  
  - **Service Locator** – `ServiceFactory.getService()` is used to obtain `OrderService`.  
  - **Command/Criteria** – `SearchOrdersCriteria` is passed to the service as a command object.  
  - **Locale handling** – `LocaleUtil.setLocaleToEntityCollection` localises entity fields after retrieval.

---

## 2. Detailed Description
### Flow of execution
1. **Action creation** – The Struts container instantiates `InvoiceListAction` and calls `prepare()` (because it implements `Preparable`).  
   - `prepare()` sets the page title via `PageBaseAction`.
2. **User triggers “display”** –  
   - The framework invokes `displayInvoiceList()`.  
   - Request parameters (`startindex`, `navstartdate`, `navenddate`, `invoiceId`) are parsed.  
   - `DateUtil` processes any posted start/end dates and populates the request attributes `sdate` / `edate` for the view.  
3. **Build search criteria** –  
   - `criteria` is instantiated if null.  
   - Date filters, invoice number, merchant ID, quantity, and paging start index are set on `criteria`.  
4. **Query the service layer** –  
   - `OrderService.searchInvoices(criteria)` is called.  
   - The returned `SearchOrderResponse` contains a collection of invoices and the total count.  
5. **Populate the action and view** –  
   - The `invoices` collection and paging information (`listingCount`, `realCount`) are stored on the action via helper methods from `PageBaseAction`.  
   - The collection is localised for the current UI locale.  
6. **Return result** – The action returns `SUCCESS`, which maps to the JSP that renders the list.

### Assumptions & constraints
- **Session must contain a `Context`** – otherwise a `NullPointerException` will occur.  
- **Pagination** is controlled by `PageBaseAction`, which expects `setSize()` to be called with a fixed page size read from the config.  
- **Dates** are optional; if not provided, all invoices for the merchant are returned.  
- **Thread‑safety** – `InvoiceListAction` is request‑scoped by Struts, so instance fields are safe.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `resetInvoiceList()` | Clears current filter criteria and request date attributes; then shows a fresh list. | None | `String` – result code (`SUCCESS`) | Resets `criteria`; removes `sdate`/`edate` from request |
| `displayInvoiceList()` | Main entry point that populates the invoice list based on filters and paging. | None (reads request parameters) | `String` – result code (`SUCCESS`) | Sets `invoices`; updates paging attributes; logs errors |
| `getInvoices()` / `setInvoices()` | Bean getter/setter for the invoices collection. | – / `Collection` | `Collection` | – |
| `getInvoiceId()` / `setInvoiceId()` | Getter/setter for the optional invoice number filter. | – / `String` | `String` | – |
| `getCriteria()` / `setCriteria()` | Getter/setter for the `SearchOrdersCriteria` object. | – / `SearchOrdersCriteria` | `SearchOrdersCriteria` | – |
| `prepare()` | Struts‑2 preparatory hook; sets the page title. | – | – | Calls `setPageTitle()` on `PageBaseAction` |

### Utility / helper methods
- `DateUtil.processPostedDates()` – Parses raw request parameters into `Date` objects.  
- `LocaleUtil.setLocaleToEntityCollection()` – Localises all string fields in the entity collection for the current locale.

---

## 4. Dependencies
| Library / Framework | Type | Notes |
|---------------------|------|-------|
| **Struts‑2** (`com.opensymphony.xwork2`) | Framework | Base `Action` class and `Preparable` interface. |
| **Apache Commons Configuration** (`org.apache.commons.configuration.Configuration`) | Configuration | Reads `central.invoicelist.maxsize` from `properties` files. |
| **Apache Commons Lang** (`org.apache.commons.lang.StringUtils`) | Utility | Used for string blank checks. |
| **Log4j** (`org.apache.log4j.Logger`) | Logging | Legacy logging framework. |
| **SalesManager Core** (`com.salesmanager.core.*`) | Service & entity layer | Provides `OrderService`, `SearchOrdersCriteria`, `SearchOrderResponse`. |
| **SalesManager Central** (`com.salesmanager.central.*`) | Web layer | `PageBaseAction`, `Context`, `ProfileConstants`. |
| **Java SE** (`java.util.*`, `java.text.*`) | Standard | Dates, collections, logging. |

All dependencies are standard to the SalesManager application except for the external Apache Commons libraries and Log4j, which are third‑party.

---

## 5. Additional Notes & Recommendations

### Strengths
- **Clear separation** between view (JSP), controller (action), and service layers.  
- **Re‑usability** of `SearchOrdersCriteria` across other actions.  
- **Configurable page size** via external properties.  
- **Locale‑aware** rendering of entity fields.

### Potential Issues / Edge Cases
1. **Null `Context`** – If the session is missing the `Context` object, the code throws a `NullPointerException`. A defensive check (`if (ctx == null) { ... }`) with an appropriate error message would improve robustness.
2. **Raw types** – `Collection invoices` and `Collection` parameters/returns should be generics (`List<Invoice>`) to avoid unchecked casts.
3. **Logging level** – Errors are logged at `log.error`, but the user is not notified of data‑format problems (e.g., unparsable dates). Consider adding user‑visible error messages for better UX.
4. **Exception handling** – Broad `catch (Exception e)` blocks swallow specific problems; finer‑grained exception handling would aid debugging.
5. **Service Locator** – `ServiceFactory.getService()` is used directly; dependency injection (e.g., via Spring) would make unit testing easier and reduce coupling.
6. **Thread‑local variables** – All instance fields are request‑scoped, but if the action is ever reused across threads, the static `invoicesize` could pose concurrency problems; this is unlikely but worth noting.
7. **Pagination logic** – `setPageStartIndex()` and related helpers are inherited from `PageBaseAction`. Ensure that they correctly reset the start index when filters change; otherwise, the user may see stale pages.
8. **Security** – The action does not validate that the requested `invoiceId` belongs to the logged‑in merchant. Adding an ownership check would prevent unauthorized access.

### Future Enhancements
- **Inject services** (OrderService, Configuration) via constructor/setter to support unit tests and better separation of concerns.  
- **Use Java 8+ streams** for collection manipulation, e.g., localising entities.  
- **Replace Log4j** with SLF4J + Logback for modern logging practices.  
- **Add input validation** (via Struts 2 validation framework) for date ranges and invoice ID.  
- **Introduce a DTO** for the response that contains only the fields needed by the view, reducing payload size and coupling.  
- **Implement caching** for common invoice lists (e.g., by merchant) to reduce database load.  

Overall, the action is functional and follows the established patterns of the SalesManager codebase. Addressing the points above would improve maintainability, robustness, and security.

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

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Collection;
import java.util.Date;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.PageBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.entity.orders.SearchOrderResponse;
import com.salesmanager.core.entity.orders.SearchOrdersCriteria;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LocaleUtil;

public class InvoiceListAction extends PageBaseAction implements Preparable {

	private Logger log = Logger.getLogger(InvoiceListAction.class);

	private static Configuration config = PropertiesHelper.getConfiguration();
	private Collection invoices;

	private SearchOrdersCriteria criteria;
	private String invoiceId;

	private static int invoicesize = 20;

	static {

		invoicesize = config.getInt("central.invoicelist.maxsize", 20);

	}

	public String resetInvoiceList() {
		this.criteria = null;
		super.getServletRequest().removeAttribute("sdate");
		super.getServletRequest().removeAttribute("edate");
		return displayInvoiceList();

	}

	public String displayInvoiceList() {

		// for page navigation
		String sstartindex = super.getServletRequest().getParameter(
				"startindex");

		try {

			DateUtil dh = new DateUtil();
			dh.processPostedDates(super.getServletRequest());

			if (criteria == null) {
				criteria = new SearchOrdersCriteria();
			}

			criteria.setSdate(dh.getStartDate());
			criteria.setEdate(dh.getEndDate());

			if (dh.getStartDate() != null) {
				super.getServletRequest().setAttribute("sdate",
						DateUtil.formatDate(dh.getStartDate()));
			}
			if (dh.getEndDate() != null) {
				super.getServletRequest().setAttribute("edate",
						DateUtil.formatDate(dh.getEndDate()));
			}

			DateFormat myDateFormat = new SimpleDateFormat("yyyy-MM-dd");
			Date sDate = null;
			Date eDate = null;
			try {
				if (super.getServletRequest().getParameter("navstartdate") != null) {
					if (criteria.getSdate() == null) {
						sDate = myDateFormat.parse(super.getServletRequest()
								.getParameter("navstartdate"));
					}
				}
				if (super.getServletRequest().getParameter("navenddate") != null) {
					if (criteria.getEdate() == null) {
						eDate = myDateFormat.parse(super.getServletRequest()
								.getParameter("navenddate"));
					}
				}
				criteria.setSdate(sDate);
				criteria.setEdate(eDate);
			} catch (Exception e) {
				log.error(e);
			}

			if (!StringUtils.isBlank(this.getInvoiceId())) {

				try {
					long invId = Long.parseLong(this.getInvoiceId());
					criteria.setOrderId(invId);
				} catch (Exception e) {
					log.error("Cannot parse invoiceId " + this.getInvoiceId());
				}

			}

			int startindex = 0;
			if (sstartindex != null) {
				try {
					startindex = Integer.parseInt(sstartindex);
				} catch (Exception e) {
					log
							.error("Did not received the index for iterator, will reset to 0");
				}
			}

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();


			this.setSize(invoicesize);
			this.getCriteria().setQuantity(invoicesize);
			this.getCriteria().setMerchantId(ctx.getMerchantid());
			this.getCriteria().setStartindex(this.getPageStartIndex());
			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			super.setPageStartNumber();
			SearchOrderResponse resp = oservice.searchInvoices(this
					.getCriteria());

			if (resp != null) {
				invoices = resp.getOrders();
				super.setListingCount(resp.getCount());
				super.setRealCount(resp.getOrders().size());
				super.setPageElements();
			}

			LocaleUtil.setLocaleToEntityCollection(invoices, super.getLocale());

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public Collection getInvoices() {
		return invoices;
	}

	public void setInvoices(Collection invoices) {
		this.invoices = invoices;
	}

	public String getInvoiceId() {
		return invoiceId;
	}

	public void setInvoiceId(String invoiceId) {
		this.invoiceId = invoiceId;
	}

	public SearchOrdersCriteria getCriteria() {
		return criteria;
	}

	public void setCriteria(SearchOrdersCriteria criteria) {
		this.criteria = criteria;
	}

	public void prepare() throws Exception {
		super.setPageTitle("label.invoice.invoicelist.title");
		
	}

}



```
