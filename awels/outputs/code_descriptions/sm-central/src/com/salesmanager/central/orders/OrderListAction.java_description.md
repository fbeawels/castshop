# OrderListAction.java

## Review

## 1. Summary  
**Purpose** – `OrderListAction` is a Struts‑style action that handles listing, searching, and reporting of orders for a merchant in the SalesManager central module.  
**Key Components**  

| Component | Role |
|-----------|------|
| `buildCriteria()` | Parses request parameters (dates, customer name, order id) into a `SearchOrdersCriteria` used by the `OrderService`. |
| `searchByCriteria()` | Executes a paginated search and populates the action’s `orders` collection. |
| `getOrdersList()` | Entry point for the list page; supports navigation dates and pagination. |
| `createReportByCriteria()` | Builds a printable order report, writes it to a `ByteArrayOutputStream`, and exposes it via `InputStream`. |
| `PageBaseAction` | Inherited paging utilities (size, start index, page title, etc.). |
| `OrderService`, `MerchantService` | Service layer beans accessed through `ServiceFactory`. |
| `LabelUtil`, `LocaleUtil` | Localization helpers. |
| `FileUtil`, `PropertiesUtil` | Utility helpers for file paths and config. |

**Design patterns & frameworks**  

* **Service Locator** – `ServiceFactory.getService(...)` supplies services.  
* **Data Transfer Objects (DTOs)** – `SearchOrderResponse`, `SearchOrdersCriteria`, `OrderReport`, `ReportHeader`.  
* **Struts‑like MVC** – the class acts as a controller, populating request attributes for a JSP view.  
* **JavaBeans** – getters/setters expose properties for the view layer.

## 2. Detailed Description  
### Core Flow  

1. **Initialization** – `ordersize` is loaded from a static configuration during class loading.  
2. **Criteria Building** – `buildCriteria()` pulls request parameters, converts them into a `DateUtil` instance, sets dates and optional customer name, and assigns the current merchant id.  
3. **Action Execution**  
   * **Listing (`displayOrderList*` & `getOrdersList`)** – sets paging attributes, calls `OrderService.searchOrders`, updates the page model (`orders`, `listingCount`, `realCount`).  
   * **Searching (`searchByCriteria`)** – similar to listing but resets the session counter and uses the current criteria.  
   * **Reporting (`createReportByCriteria`)** – after building criteria, optionally narrows to a single order id, prepares a `OrderReport`, obtains a branded logo path, constructs a descriptive header, and delegates to `orderService.prepareOrderListReport`. The resulting byte array is wrapped in a `ByteArrayInputStream` for streaming to the client.  
4. **Cleanup** – No explicit cleanup; resources are managed by the servlet container (e.g., the `InputStream` is consumed by the view layer).  

### Assumptions & Constraints  

* **Per‑request action instance** – relies on the Struts/Servlet lifecycle; thread‑safe by design.  
* **Locale** – derived from the session; assumed to be set before any action execution.  
* **Merchant context** – `getContext().getMerchantid()` is expected to be populated by the base class or a filter.  
* **Config Availability** – `PropertiesHelper.getConfiguration()` must be initialized before any request; static block may throw `NullPointerException` if config fails.  

### Architectural Notes  

* Tight coupling to `ServiceFactory` (service locator) – makes unit testing harder.  
* The action mixes UI concerns (setting request attributes) with business logic (report generation).  
* Hard‑coded date format (`yyyy-MM-dd`) – would break if the UI sends a different format.  
* Use of raw `Collection` rather than generics – loses compile‑time type safety.  

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `buildCriteria()` | Reads request parameters and builds `SearchOrdersCriteria`. | `HttpServletRequest` (via `super.getServletRequest()`) | Sets `criteria` field | Sets request attributes (`sdate`, `edate`, `customername`). |
| `createReportByCriteria()` | Generates a printable order report based on current criteria. | None (uses `criteria`, `orderId`) | Returns `"SUCCESS"` or `"INPUT"` | Populates `inputStream` with report bytes; writes error messages to request. |
| `searchByCriteria()` | Executes a paginated order search. | None (uses request params and `orderId`) | Returns `"SUCCESS"` | Populates `orders`, updates paging info; writes error messages. |
| `getOrdersList()` | Internal helper for list page rendering. | None | Returns `"SUCCESS"` | Sets page title, prepares criteria, executes search, updates pagination. |
| `displayOrderList()` | Entry point for order list view. | None | Delegates to `getOrdersList()` | Removes `ITEMCOUNT` from session. |
| `displayOrderListPage()` | Same as above but used when the page number is provided. | None | Delegates to `getOrdersList()` | |
| Getters/Setters (`getCardtext`, `setCardtext`, `getOrders`, `getCriteria`, `setCriteria`, `getInputStream`, `setInputStream`, `getOrderId`, `setOrderId`) | Standard JavaBean accessors. | – | – | – |
| `setSize(int)` / `setPageStartNumber()` / `setListingCount()`, etc. – inherited from `PageBaseAction` – used for paging. | – | – | – | – |

**Reusable / Utility Methods**  
* `LabelUtil.getInstance()`, `LocaleUtil.setLocaleToEntityCollection()` – used across actions for i18n.  
* `DateUtil` – centralizes date parsing and formatting.  

## 4. Dependencies  

| Library / Class | Role | Standard / Third‑party |
|-----------------|------|------------------------|
| `org.apache.commons.configuration.Configuration` | Holds config properties | Third‑party |
| `org.apache.commons.lang.StringUtils` | String utilities | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.central.PageBaseAction` | Base action with paging helpers | Project |
| `com.salesmanager.central.util.PropertiesHelper` | Loads config | Project |
| `com.salesmanager.core.entity.*` (e.g., `ReportHeader`, `MerchantStore`, `OrderReport`) | Domain entities / DTOs | Core module |
| `com.salesmanager.core.service.*` (`OrderService`, `MerchantService`) | Business services | Core module |
| `com.salesmanager.core.util.*` (`DateUtil`, `FileUtil`, `LabelUtil`, `LocaleUtil`, `MessageUtil`, `PropertiesUtil`) | Utilities | Core module |
| `java.io.*`, `java.text.*`, `java.util.*` | Standard Java APIs | Standard |

**Platform Specific** – None obvious; code is platform agnostic except for servlet context via `PageBaseAction`.

## 5. Additional Notes  

### Edge Cases & Potential Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Static `ordersize` loaded once** | If config changes at runtime, the action will keep the old value. | Make `ordersize` a non‑static field or reload on each request if necessary. |
| **Raw `Collection`** | No compile‑time type safety; risk of `ClassCastException`. | Use `Collection<Order>` or `List<Order>` with generics. |
| **No null checks on `super.getContext()`** | Could throw NPE if context not set. | Validate context before use. |
| **Repeated order ID parsing** | Duplicate code in `createReportByCriteria` and `searchByCriteria`. | Extract into a helper method. |
| **Hard‑coded date format** | Will fail if UI sends different format. | Use `DateUtil` for all parsing or accept format from config. |
| **Large reports in memory** | `ByteArrayOutputStream` may blow heap for huge reports. | Stream directly to response or use a temp file. |
| **Security** | Order ID is parsed without validation; potential injection if service layer doesn’t validate. | Add input validation and sanitize. |
| **Exception handling** | Only logs and shows generic error; stack trace lost. | Log stack trace, consider rethrowing or mapping to user‑friendly messages. |
| **Service locator** | Hard to mock in unit tests. | Use dependency injection (Spring, CDI) instead. |

### Potential Enhancements  

1. **Refactor to Service Layer** – Move report‑generation logic into a dedicated `OrderReportService` to keep the action thin.  
2. **Pagination & Navigation** – Centralize pagination logic; avoid manual `setPageStartNumber()` calls scattered across methods.  
3. **i18n** – Load all labels via `LabelUtil` consistently; avoid string concatenation with raw labels.  
4. **Unit Tests** – With generics and DI, write tests for `buildCriteria`, `searchByCriteria`, etc.  
5. **Error Reporting** – Use a custom exception hierarchy to provide richer context to the UI.  
6. **Logging** – Add correlation IDs, request IDs, or session IDs to logs for easier debugging.  
7. **Input Validation** – Use Struts/JSR‑303 validators for request parameters.  

---

Overall, the class fulfills its role of coordinating order list retrieval and report generation. The code is functional but could benefit from modern Java practices (generics, DI, better error handling) and a cleaner separation of concerns to improve maintainability and testability.

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
package com.salesmanager.central.orders;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.PageBaseAction;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.entity.common.ReportHeader;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.OrderReport;
import com.salesmanager.core.entity.orders.SearchOrderResponse;
import com.salesmanager.core.entity.orders.SearchOrdersCriteria;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class OrderListAction extends PageBaseAction {

	private Logger log = Logger.getLogger(OrderListAction.class);
	private String cardtext;

	private Collection orders = new ArrayList();

	static Configuration config = PropertiesHelper.getConfiguration();

	private static int ordersize = 20;

	private SearchOrdersCriteria criteria = null;
	
	private InputStream inputStream;
	
	private String orderId = "";

	static {

		ordersize = config.getInt("central.orderlist.maxsize", 20);

	}

	private void buildCriteria() throws Exception {

		criteria = new SearchOrdersCriteria();

		DateUtil dh = new DateUtil();
		dh.processPostedDates(super.getServletRequest());

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

		String customername = super.getServletRequest().getParameter(
				"customername");

		if (customername != null && !customername.trim().equals("")) {
			criteria.setCustomerName(customername);
			super.getServletRequest()
					.setAttribute("customername", customername);
		}

		criteria.setMerchantId(super.getContext().getMerchantid());

	}
	
	/**
	 * Creates report
	 * @return
	 */
	public String createReportByCriteria() {
		
		// START DATE - END DATE

		try {

			
			this.buildCriteria();
			
			
			if (!StringUtils.isBlank(this.getOrderId())) {
				long oid = -1;
				try {
					oid = Long.parseLong(this.getOrderId());

				} catch (NumberFormatException nfe) {
					log.error(nfe);
				}
				criteria.resetCriteria();
				criteria.setOrderId(oid);
			}

			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(super.getLocale());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			SearchOrderResponse response = oservice.searchOrders(criteria);

			orders = response.getOrders();
			
			LocaleUtil.setLocaleToEntityCollection(orders, super.getLocale());
			
			
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(super.getContext().getMerchantid());
			store.setLocale(super.getLocale());
			
			
			ReportHeader reportHeader = new ReportHeader();
			reportHeader.setStore(store);
			
			OrderReport report = new OrderReport();
			report.setOrders(orders);
			
			if (!StringUtils.isEmpty(store.getStorelogo())) {
				//String path = PropertiesUtil.getConfiguration().getString(
				//		"core.branding.cart.filefolder");
				String path = FileUtil.getBrandingFilePath();
				path = path + "/" + store.getMerchantId() + "/header/"
						+ store.getStorelogo();
				reportHeader.setMerchantStoreLogo(path);
			}
						
			//Build search string
			/**
			 * startDate (may be null)
			 * endDate (may be null)
			 * customerName (may be null)
			 */
			StringBuffer reportCriteriaText = new StringBuffer();
			reportCriteriaText.append(label.getText("label.order.searchreport.title"));
			
			
			

			if (!StringUtils.isBlank(this.getOrderId())) {
				long oid = -1;
				try {
					oid = Long.parseLong(this.getOrderId());

				} catch (NumberFormatException nfe) {
					log.error(nfe);
				}
				criteria.resetCriteria();
				criteria.setOrderId(oid);
				reportCriteriaText.append(" ").append(label.getText("label.order.orderid")).append(" ").append(oid);
			} else {
			
				if(criteria.getSdate()!=null) {
					reportCriteriaText.append(" ").append(label.getText("label.generic.startdate")).append(" ").append(criteria.getStartDateString());
				}
				if(criteria.getEdate()!=null) {
					reportCriteriaText.append(" ").append(label.getText("label.generic.enddate")).append(" ").append(criteria.getEndDateString());
				}
				if(!StringUtils.isBlank(criteria.getCustomerName())) {
					reportCriteriaText.append(" ").append(label.getText("label.customer.name")).append(" ").append(criteria.getCustomerName());
				}
			
			}
			
			reportHeader .setSearchReportCriteria(reportCriteriaText.toString());
			
			report.setReportHeader(reportHeader);
			
			OrderService orderService = (OrderService)ServiceFactory.getService(ServiceFactory.OrderService);
			
			ByteArrayOutputStream os = new ByteArrayOutputStream();
			
			orderService.prepareOrderListReport(report, super.getLocale(), os);
			
			inputStream = new ByteArrayInputStream(os.toByteArray());
			
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			
			return INPUT;
		}
		
	}

	/**
	 * Search header functions
	 * 
	 * @return
	 * @throws Exception
	 */
	public String searchByCriteria() throws Exception {

		// START DATE - END DATE

		try {

			super.getServletRequest().getSession().removeAttribute("ITEMCOUNT");
			this.buildCriteria();

			//String orderid = super.getServletRequest().getParameter("orderid");

			if (!StringUtils.isBlank(this.getOrderId())) {
				long oid = -1;
				try {
					oid = Long.parseLong(this.getOrderId());

				} catch (NumberFormatException nfe) {
					log.error(nfe);
				}
				criteria.resetCriteria();
				criteria.setOrderId(oid);

			}

			this.setSize(ordersize);
			super.setPageStartNumber();

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			SearchOrderResponse response = oservice.searchOrders(criteria);

			orders = response.getOrders();
			super.setListingCount(response.getCount());
			super.setRealCount(response.getOrders().size());
			super.setPageElements();

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}
		return SUCCESS;
	}

	/**
	 * Page entry point
	 * 
	 * @return
	 * @throws Exception
	 */

	private String getOrdersList() throws Exception {
		try {
			
			super.setPageTitle("label.order.orderlist.title");
			this.buildCriteria();

			// override start date & end date with page navigation criteria submission
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

			this.setSize(ordersize);
			super.setPageStartNumber();

			criteria.setQuantity(ordersize);
			criteria.setStartindex(super.getPageStartIndex());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			SearchOrderResponse response = oservice.searchOrders(criteria);

			orders = response.getOrders();

			super.setListingCount(response.getCount());
			super.setRealCount(response.getOrders().size());
			super.setPageElements();

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;

	}

	public String displayOrderList() throws Exception {

		super.getServletRequest().getSession().removeAttribute("ITEMCOUNT");
		return this.getOrdersList();

	}

	public String displayOrderListPage() throws Exception {
		return this.getOrdersList();
	}

	public String getCardtext() {
		return cardtext;
	}

	public void setCardtext(String cardtext) {
		this.cardtext = cardtext;
	}

	public Collection getOrders() {
		return orders;
	}

	public SearchOrdersCriteria getCriteria() {
		return criteria;
	}

	public void setCriteria(SearchOrdersCriteria criteria) {
		this.criteria = criteria;
	}

	public InputStream getInputStream() {
		return inputStream;
	}

	public void setInputStream(InputStream inputStream) {
		this.inputStream = inputStream;
	}

	public String getOrderId() {
		return orderId;
	}

	public void setOrderId(String orderId) {
		this.orderId = orderId;
	}

}



```
