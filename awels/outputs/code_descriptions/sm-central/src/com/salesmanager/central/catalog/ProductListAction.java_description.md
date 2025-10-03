# ProductListAction.java

## Review

## 1. Summary

`ProductListAction` is a Struts‑style action class that lives in the **Central** catalog module of the SalesManager system.  
Its purpose is two‑fold:

1. **`updateavailability`** – Batch‑updates the “visible” flag of a set of products, based on the checkboxes that a merchant has selected in the UI.  
2. **`show`** – Renders the product‑listing page, applying filters such as category, product name, availability, and stock status, and populating the request with a paginated list of `Product` objects.

Key components:
- **`PageBaseAction`** – Provides common action helpers (paging, messaging, etc.).  
- **`CatalogService`** – The service layer that actually performs the database operations.  
- **`SearchProductCriteria` / `SearchProductResponse`** – DTOs used to pass search parameters and results.  
- **`Context` / `ProfileConstants`** – Session‑scoped data used for multi‑tenancy (merchant id, language, locale).  
- **`LanguageUtil` / `LocaleUtil`** – Helpers for language/locale handling.  

The code follows a fairly conventional MVC/Struts architecture and uses Apache Log4j, Hibernate (via `HibernateException`), and Apache Commons Configuration.

---

## 2. Detailed Description

### Flow of execution

| Step | Method | What happens |
|------|--------|--------------|
| **Static init** | Class loading | `listsize` is read from `core.productlist.maxsize` in the Central configuration (default 20). |
| **`show`** | GET request | 1. Retrieves request parameters (`startindex`, `categ`, `productname`, `availability`, `status`).<br>2. Sets default filter values on the request.<br>3. Builds a `SearchProductCriteria` instance, mapping UI values to internal constants.<br>4. Calls `CatalogService.findProducts` with the criteria.<br>5. Populates request attributes (`PRRODUCTSLIST`, counts) and localises entities.<br>6. Returns `SUCCESS` for the Struts result. |
| **`updateavailability`** | POST request | 1. Reads selected (`prodavailability`) and all (`prodlist`) product IDs from the request.<br>2. Builds a list of IDs that should be marked visible.<br>3. Calls `CatalogService.updateProductListAvailability(true, …)` for those IDs.<br>4. Builds a list of IDs that should be marked invisible (those in `prodlist` but not in `prodavailability`).<br>5. Calls `CatalogService.updateProductListAvailability(false, …)` for those IDs.<br>6. Sets a success message and returns `SUCCESS`. |

### Assumptions & Constraints

- **Merchant Context** – The `Context` bean is always present in the HTTP session. The action does not guard against a missing context, which would result in a `NullPointerException`.  
- **Parameter presence** – Some parameters (e.g., `startindex`) are assumed non‑null; a null value would cause a NPE when calling `super.getPageStartNumber()` or `criteria.setStartindex(...)`.  
- **Legacy API** – The code uses raw collections (`Map`, `List`, `Collection`) and the old Struts 1 style (`SUCCESS` string). It relies on `CatalogService` methods that accept `List<Long>` of product IDs.  
- **Locale** – The final localisation step assumes that all entities returned from `findProducts` have a language‑specific field that `LocaleUtil.setLocaleToEntityCollection` can populate.  

### Design Choices

- **Single Action** – Both listing and update actions are grouped into one class, a typical Struts 1 pattern.  
- **Direct Service Calls** – The action hands raw parameters straight to the service layer without an intermediate DTO; the service is responsible for all validation.  
- **Configuration‑Driven List Size** – `listsize` is externalised, allowing administrators to tweak page size without code changes.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `public String updateavailability() throws Exception` | Batch‑updating product visibility. | None (reads request parameters). | `SUCCESS` or throws exception. | Sets success message, modifies database via `CatalogService`. |
| `public String show() throws Exception` | Prepares product listing page. | None (reads request parameters). | `SUCCESS` or throws exception. | Populates request attributes, sets pagination state. |
| **Private helpers** (inherited from `PageBaseAction`) | `setPageTitle(String)` – sets page title.<br>`setSuccessMessage()` – sets a generic success message.<br>`setListingCount(long)` – records total number of items.<br>`setRealCount(int)` – records items actually returned.<br>`setPageElements()` – calculates paging UI elements. | Various. | None. | UI state changes. |

**Reusable methods**: The action’s logic for mapping UI parameters to internal constants (`ProductSearchFilterCriteria.*`) is duplicated in both `show` and could be extracted into a helper method for clarity.

---

## 4. Dependencies

| Category | Dependency | Notes |
|----------|------------|-------|
| **Logging** | `org.apache.log4j.Logger` | Standard Log4j 1.x (outdated but still common). |
| **Configuration** | `org.apache.commons.configuration.Configuration` + `PropertiesHelper` | Reads `core.productlist.maxsize` from a central properties file. |
| **ORM** | `org.hibernate.HibernateException` | Indicates the action expects Hibernate errors. |
| **Framework** | `PageBaseAction` (custom) | Base Struts‑like action providing paging and messaging. |
| **Profile** | `Context`, `ProfileConstants` | Session management for merchant/account context. |
| **Catalog** | `SearchProductCriteria`, `SearchProductResponse`, `CatalogService` | Domain entities and service layer. |
| **Utils** | `LanguageUtil`, `LocaleUtil` | Language code mapping and entity localisation. |
| **Java Collections** | `java.util.*` | Raw types used throughout. |

All dependencies are **third‑party** libraries except the internal SalesManager packages. No platform‑specific (e.g., Android) dependencies.

---

## 5. Additional Notes

### Strengths
- **Simplicity** – Clear separation of concerns: the action only prepares data; the service layer performs business logic.  
- **Configurability** – List size is externally configurable.  
- **Use of Locale** – Entities are localized before being sent to the view.  

### Issues & Edge Cases
1. **Raw types** – The use of raw `Map`, `List`, `Collection` leads to unchecked casts and potential `ClassCastException`. Modern Java would use generics (`Map<String, String>`, `List<Long>`, etc.).  
2. **Null‑Pointer Risks** – `original` and `check` parameters are used without null checks before accessing `length`. If either is missing, the action will throw NPE.  
3. **Hard‑coded constants** – `ProductSearchFilterCriteria` constants are referenced but not imported; if the class is missing or renamed, compilation will fail.  
4. **Duplicate code** – The mapping from UI values (`availability`, `status`) to internal constants is repeated and could be abstracted.  
5. **No validation** – Input parameters (especially numeric IDs) are parsed directly with `Integer.parseInt`/`Long.valueOf` without handling `NumberFormatException`.  
6. **Logging** – Exceptions are logged and then re‑thrown. While this preserves stack traces, it could result in double logging if the caller also logs.  
7. **Internationalisation** – The default request attributes for filters are hard‑coded (e.g., `availability` set to `"2"`). If the UI changes the default, the action must be updated.  

### Possible Enhancements
- **Generics & Java 8+** – Replace raw types with generics and use streams/Optional to simplify logic.  
- **Parameter Validation** – Introduce a request‑validation layer (e.g., Struts 2 `Validator` or a custom filter) to guard against malformed input.  
- **Error Handling** – Return user‑friendly error messages rather than propagating raw exceptions.  
- **Pagination API** – Abstract paging logic into a reusable component rather than relying on inherited methods.  
- **Unit Tests** – The action logic is currently hard to test due to its tight coupling with `HttpServletRequest` and session attributes. Consider refactoring into a service class that can be unit‑tested independently.  
- **Logging Improvement** – Use structured logging and avoid re‑throwing unless necessary; wrap Hibernate exceptions into a domain‑specific exception.  
- **Localization** – Centralise the localisation logic to avoid calling `LocaleUtil.setLocaleToEntityCollection` manually in every action.  

### Closing Remarks
`ProductListAction` fulfills its basic responsibilities in a straightforward, albeit somewhat dated, fashion. Refactoring to modern Java idioms (generics, streams, improved error handling) and strengthening input validation would make the code more robust, maintainable, and testable.

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
package com.salesmanager.central.catalog;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;
import org.hibernate.HibernateException;

import com.salesmanager.central.PageBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.entity.catalog.SearchProductCriteria;
import com.salesmanager.core.entity.catalog.SearchProductResponse;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;

public class ProductListAction extends PageBaseAction {

	private static Logger log = Logger.getLogger(ProductListAction.class);

	private static Configuration config = PropertiesHelper.getConfiguration();
	private static int listsize = 20;

	static {

		listsize = config.getInt("core.productlist.maxsize", 20);
	}

	/**
	 * For updating the availability
	 * 
	 * @return
	 * @throws Exception
	 */
	public String updateavailability() throws Exception {

		super.setPageTitle("label.prodlist.title");
		
		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			String check[] = super.getServletRequest().getParameterValues(
					"prodavailability");
			String original[] = super.getServletRequest().getParameterValues(
					"prodlist");

			Map checked = new HashMap();

			// Set the availability for checked entries
			if (check != null && check.length > 0) {

				List cids = new ArrayList();
				for (int i = 0; i < check.length; i++) {
					// retrieve product id
					String id = check[i];
					cids.add(Long.valueOf(id));

					checked.put(id, id);
				}

				CatalogService cservice = (CatalogService) ServiceFactory
						.getService(ServiceFactory.CatalogService);

				cservice.updateProductListAvailability(true, super.getContext()
						.getMerchantid(), cids);

			}

			// unset the availability for unchecked entries
			if (original.length != checked.size()
					&& original.length >= checked.size()) {
				int ifound = original.length - checked.size();
				if (original != null && original.length > 0) {

					List ids = new ArrayList();
					int curfound = 0;
					for (int i = 0; i < original.length; i++) {
						String id = original[i];
						if (!checked.containsKey(id)) {
							ids.add(Long.valueOf(id));
							curfound++;
						}
					}

					CatalogService cservice = (CatalogService) ServiceFactory
							.getService(ServiceFactory.CatalogService);

					cservice.updateProductListAvailability(false, super
							.getContext().getMerchantid(), ids);

				}
			}

			super.setSuccessMessage();
			return SUCCESS;
		} catch (HibernateException e) {
			log.error(e);
			throw e;
		}

	}

	/**
	 * For displaying the listing page
	 * 
	 * @return
	 * @throws Exception
	 */
	public String show() throws Exception {

		super.setPageTitle("label.prodlist.title");
		
		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			// this property is specific to central

			String sstartindex = super.getServletRequest().getParameter(
					"startindex");
			String categ = super.getServletRequest().getParameter("categ");
			String productname = super.getServletRequest().getParameter(
					"productname");
			String availability = super.getServletRequest().getParameter(
					"availability");
			String status = super.getServletRequest().getParameter("status");

			super.getServletRequest().setAttribute("categ", -1);
			super.getServletRequest().setAttribute("productname", "");
			super.getServletRequest().setAttribute("availability", "2");
			super.getServletRequest().setAttribute("status", "2");



			SearchProductCriteria criteria = new SearchProductCriteria();

			// include the requested category in the query
			if (categ != null && !categ.equals("") && !categ.equals("-1")) {
				try {
					int categid = Integer.parseInt(categ);
					criteria.setCategoryid(categid);
					super.getServletRequest().setAttribute("categoryfilter",
							categ);
				} catch (Exception e) {
					log
							.error("Cannot parse String " + categ
									+ " to categoryid");
				}
			}
			if (productname != null && !productname.equals("")) {
				criteria.setDescription(productname);
				super.getServletRequest().setAttribute("productname",
						productname);
			}

			if (availability != null && !availability.equals("")) {// availability
				if (availability.equals("1")) {
					criteria
							.setVisible(ProductSearchFilterCriteria.VISIBLETRUE);
				}
				if (availability.equals("2")) {
					criteria.setVisible(ProductSearchFilterCriteria.VISIBLEALL);
				}
				if (availability.equals("0")) {
					criteria
							.setVisible(ProductSearchFilterCriteria.VISIBLEFALSE);
				}
				super.getServletRequest().setAttribute("availability",
						availability);
			}

			if (status != null && !status.equals("")) {// visibility

				if (status.equals("1")) {
					criteria
							.setStatus(ProductSearchFilterCriteria.STATUSINSTOCK);
				}
				if (status.equals("2")) {
					criteria.setStatus(ProductSearchFilterCriteria.STATUSALL);
				}
				if (status.equals("0")) {
					criteria
							.setStatus(ProductSearchFilterCriteria.STATUSOUTSTOCK);
				}
				super.getServletRequest().setAttribute("status", status);
			}

			setSize(listsize);
			super.setPageStartNumber();

			criteria.setMerchantId(super.getContext().getMerchantid());
			criteria.setQuantity(this.getSize());
			criteria.setStartindex(this.getPageStartIndex());
			criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(super
					.getContext().getLang()));

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			SearchProductResponse response = cservice.findProducts(criteria);

			super.getServletRequest().setAttribute("PRRODUCTSLIST",
					response.getProducts());
			super.setListingCount(response.getCount());
			super.setRealCount(response.getProducts().size());
			super.setPageElements();

			Collection prds = response.getProducts();
			LocaleUtil.setLocaleToEntityCollection(prds, super.getLocale());

			return SUCCESS;
		} catch (HibernateException e) {
			log.error(e);
			throw e;
		}
	}

}



```
