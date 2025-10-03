# EditDiscountAction.java

## Review

## 1. Summary

**Purpose**  
`EditDiscountAction` is a Struts‑2 style action that lets an administrator view, create, update or delete a *discount* (a `Special` entity) for a particular `Product`. It populates the form with existing data, validates user input, persists changes through a `CatalogService`, and shows feedback messages.

**Key Components**

| Component | Role |
|-----------|------|
| `product` / `special` | Domain objects that are read/written. |
| `showDiscount()` | Prepares data for the JSP (product price, name, current discount). |
| `saveDiscount()` | Validates submitted fields, updates/creates a `Special`, and persists it. |
| `deleteDiscount()` | Removes the discount from persistence. |
| `Context`, `ProfileConstants` | Session helpers that provide the current currency and language. |
| `CurrencyUtil`, `DateUtil`, `LabelUtil`, `LanguageUtil` | Utility helpers for formatting, parsing and localisation. |
| `CatalogService` | DAO layer used to fetch and persist domain entities. |

**Design Patterns / Frameworks**

* **Service Locator** – `ServiceFactory.getService(...)` is used to obtain the `CatalogService`.  
* **Utility / Helper** – `CurrencyUtil`, `DateUtil`, etc. encapsulate common logic.  
* **Action‑based MVC** – Struts‑2 action with form backing fields and result strings (`SUCCESS`, `unauthorized`).  
* **i18n** – Messages and labels are pulled from resource bundles via `MessageUtil` and `LabelUtil`.  

---

## 2. Detailed Description

### Execution Flow

1. **`showDiscount()`**  
   * Checks that a product is supplied (`product` not null & has an id).  
   * Loads the full `Product` (with price, descriptions, and existing `Special`) via `CatalogService`.  
   * Formats the current price and discount dates into strings suitable for the JSP.  
   * Stores the data in the action’s fields (`productName`, `productPrice`, `productNewPrice`, `sdate`, `edate`).  
   * Returns `SUCCESS` so the view can render the form.

2. **`saveDiscount()`**  
   * Sets the page title (for the view).  
   * Reads the submitted `productNewPrice`, `dstartdate`, and `denddate`.  
   * **BUG** – `showDiscount()` is called *before* the value is validated, which overwrites the user‑supplied price.  
   * Parses and validates the new price (`CurrencyUtil.validateCurrency`).  
   * Parses the start / end dates (`yyyy-MM-dd`). Failing to parse falls back to the current date.  
   * If no existing `Special`, creates one; otherwise updates timestamps.  
   * Persists the `Special` via `CatalogService`.  
   * Adds a confirmation message and returns `SUCCESS`.

3. **`deleteDiscount()`**  
   * Calls `showDiscount()` to load the current discount.  
   * If a discount exists, deletes it via `CatalogService`.  
   * Adds a confirmation message and returns `SUCCESS`.

### Dependencies and Constraints

* **No input validation framework** – field errors are added manually.  
* **Java Date API** – uses legacy `java.util.Date` and `SimpleDateFormat`.  
* **Service Locator** – not ideal for unit testing or dependency injection.  
* **Null Checks** – performed at the beginning of each action but may be insufficient for all edge cases.  
* **Security** – no explicit permission checks; relies on surrounding Struts filters.  

### Architecture & Design Choices

* The action is tightly coupled to the persistence layer via a Service Locator.  
* Utility classes are heavily used, which keeps business logic small but can hide important behavior (e.g., `CurrencyUtil.validateCurrency`).  
* The class uses raw collections (`Set`) instead of generics, which can lead to `ClassCastException` if mis‑used.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `showDiscount()` | Loads product & discount data for display. | None | `String` (`SUCCESS` or `"unauthorized"`) | Sets action fields (`productName`, `productPrice`, …). |
| `saveDiscount()` | Validates user input, updates/creates a `Special`, persists it. | None | `String` (`SUCCESS`) | Modifies `special`, writes to DB, sets page title, adds messages/field errors. |
| `deleteDiscount()` | Deletes the current discount. | None | `String` (`SUCCESS`) | Calls `CatalogService.deleteSpecial`, adds message. |
| Getters/Setters | Standard JavaBean accessors for action properties (`product`, `special`, date fields, etc.). | None | Depends on accessor | None (except setters update fields). |

**Reusable/Utility Methods**

* `CurrencyUtil.validateCurrency(String, Currency)` – parses & validates price strings.  
* `DateUtil.formatDate(Date)` – formats dates for display.  
* `LanguageUtil.getLanguageNumberCode(String)` – maps language string to numeric code.  
* `LabelUtil.getInstance().getText(String)` – i18n label lookup.  

---

## 4. Dependencies

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.central.BaseAction` | Project | Struts2 action base class. |
| `com.salesmanager.central.profile.Context`, `ProfileConstants` | Project | Session utilities. |
| `com.salesmanager.core.entity.catalog.*` | Project | JPA/Hibernate entities. |
| `com.salesmanager.core.service.ServiceFactory`, `CatalogService` | Project | Service locator and DAO layer. |
| `com.salesmanager.core.util.*` (`CurrencyUtil`, `DateUtil`, `LabelUtil`, `LanguageUtil`, `MessageUtil`) | Project | Helper utilities. |
| Java SE APIs (`java.util`, `java.math`, `java.text`) | Standard | Date handling, BigDecimal, formatting. |

No external frameworks beyond Log4j and the custom `salesmanager` packages.

---

## 5. Additional Notes & Recommendations

### 5.1  Critical Bugs / Issues

1. **Overwriting User Input**  
   * `saveDiscount()` calls `showDiscount()` before validating `productNewPrice`. The call re‑populates `productNewPrice` from the database, erasing the submitted value and causing validation to always succeed.  
   * **Fix:** Remove the call or move it after validation if necessary for view rendering.

2. **Date Parsing Fallback**  
   * When parsing `dstartdate` / `denddate` fails, the code silently defaults to the current date.  
   * **Fix:** Add a clear validation error and return the form to the user instead of silently applying the current date.

3. **Raw Types**  
   * `Set descriptionset = product.getDescriptions();` uses a raw `Set`.  
   * **Fix:** Use generics (`Set<ProductDescription>`). This will catch type‑mismatch at compile time.

4. **Legacy Date API**  
   * The code uses `java.util.Date` and `SimpleDateFormat`.  
   * **Fix:** Adopt `java.time` (`LocalDate`, `LocalDateTime`, `DateTimeFormatter`) for better type safety and time‑zone handling.

5. **No Permission Checks**  
   * The action assumes that the caller is authorized to edit discounts.  
   * **Fix:** Add role‑based checks (e.g., via Struts2 interceptor or Spring Security) to prevent unauthorized modifications.

6. **Transaction Management**  
   * Persistence operations are executed directly without explicit transaction boundaries.  
   * **Fix:** Wrap database calls in a transactional context (e.g., using Spring’s `@Transactional` or a dedicated transaction manager).

### 5.2  Improvements

| Area | Recommendation |
|------|----------------|
| **Dependency Injection** | Replace `ServiceFactory.getService()` with constructor/setter injection (Spring or CDI). Easier to mock in unit tests. |
| **Validation** | Use Struts2 `Validator` framework or Bean Validation (JSR‑380) to declaratively enforce constraints (price format, date ranges, non‑empty fields). |
| **Internationalisation** | Externalise all hard‑coded messages (`error.message.price.format`, `"message.confirmation.success"`) into resource bundles. |
| **Logging** | Use `log.info()` for successful operations, `log.warn()` for recoverable errors, and `log.error()` for exceptions. |
| **Error Handling** | Return distinct result strings (`error`, `validationError`) instead of always returning `SUCCESS`. |
| **Code Duplication** | Extract product loading logic into a private helper (`loadProduct()`), used by both `showDiscount()` and `saveDiscount()`. |
| **Unit Tests** | Add tests for each action method, mocking `CatalogService` and `Context`. Validate that date parsing, price validation, and special creation work as expected. |
| **Security** | Ensure that the `product` is loaded from the session or request in a way that prevents ID manipulation (e.g., by checking the current user’s permissions on that product). |

### 5.3  Edge Cases Not Handled

* **Start date after end date** – the code accepts it silently, leading to an inconsistent discount period.  
* **Negative or zero prices** – no check to prevent nonsensical discounts.  
* **Large discounts** – no upper bound validation.  
* **Duplicate discounts** – multiple specials for the same product could coexist.  
* **Concurrent edits** – race conditions could overwrite a discount.  

Addressing these would improve robustness.

---

### Bottom Line

`EditDiscountAction` fulfills its basic functional requirements but contains several design and implementation problems that could lead to incorrect data being persisted or to security issues. By refactoring for dependency injection, modern date/time APIs, proper validation, and transaction handling, the code would become safer, easier to maintain, and better suited for unit testing.

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

import java.math.BigDecimal;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Iterator;
import java.util.Set;

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.Special;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;

public class EditDiscountAction extends BaseAction {

	private Product product;
	private Special special;

	private String productName;
	private String productPrice;
	private String productNewPrice;// also submited
	private String sdate;
	private String edate;

	private String dstartdate;// submited
	private String denddate;// submited

	private static Logger log = Logger.getLogger(EditDiscountAction.class);

	public String showDiscount() {

		try {
			
			super.setPageTitle("label.product.discount");

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			// get the product

			if (product == null || product.getProductId() == 0) {
				this.addActionError(getText("errors.technical"));
				return "unauthorized";
			}

			CatalogService catalogservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			product = catalogservice.getProduct(product.getProductId());

			if (product == null) {
				this.addActionError(getText("errors.technical"));
				return "unauthorized";
			}

			productPrice = CurrencyUtil.displayFormatedAmountNoCurrency(product
					.getProductPrice(), ctx.getCurrency());

			Set descriptionset = product.getDescriptions();
			int lang = LanguageUtil.getLanguageNumberCode(ctx.getLang());
			if (descriptionset != null) {
				Iterator i = descriptionset.iterator();
				while (i.hasNext()) {
					ProductDescription desc = (ProductDescription) i.next();
					if (desc.getId().getLanguageId() == lang) {
						productName = desc.getProductName();
						break;
					}
				}
			}

			// get a discount
			special = catalogservice.getSpecial(product.getProductId());

			if (special != null) {
				productNewPrice = CurrencyUtil.displayFormatedAmountNoCurrency(
						special.getSpecialNewProductPrice(), ctx.getCurrency());
				sdate = DateUtil.formatDate(special.getSpecialDateAvailable());
				edate = DateUtil.formatDate(special.getExpiresDate());
			}

		} catch (Exception e) {
			log.error(e);
			super.addActionError(getText("error.technical"));
		}

		return SUCCESS;
	}

	public String saveDiscount() {
		
		super.setPageTitle("label.product.discount");

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);

		try {
			String newPrice = this.getProductNewPrice();
			this.showDiscount();

			BigDecimal bdNewPrice = null;

			Date dt = new Date();

			try {
				bdNewPrice = CurrencyUtil.validateCurrency(newPrice, ctx
						.getCurrency());
				this.setProductNewPrice(newPrice);
			} catch (Exception e) {
				super.addFieldError("productPrice",
						getText("error.message.price.format"));
				return SUCCESS;
			}

			DateFormat myDateFormat = new SimpleDateFormat("yyyy-MM-dd");
			Date sDate = null;
			Date eDate = null;
			try {
				sDate = myDateFormat.parse(this.getDstartdate());
				eDate = myDateFormat.parse(this.getDenddate());
			} catch (Exception e) {
				log.error(e);
				sDate = new Date(dt.getTime());
				eDate = new Date(dt.getTime());
			}

			if (this.getSpecial() == null) {
				special = new Special();
				special.setStatus(1);
				special.setProductId(product.getProductId());
			} else {
				special.setSpecialLastModified(dt);
				special.setDateStatusChange(dt);

			}
			special.setSpecialNewProductPrice(bdNewPrice);
			special.setSpecialDateAvailable(sDate);
			special.setExpiresDate(eDate);

			CatalogService catalogservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			catalogservice.saveOrUpdateSpecial(special);

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			super.addActionError(getText("error.technical"));
		}

		return SUCCESS;
	}

	public String deleteDiscount() {
		
		try {
			this.showDiscount();
			if (this.getSpecial() != null) {
				CatalogService catalogservice = (CatalogService) ServiceFactory
						.getService(ServiceFactory.CatalogService);
				catalogservice.deleteSpecial(special);
			}
			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			super.addActionError(getText("error.technical"));
		}

		return SUCCESS;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public String getDenddate() {
		return denddate;
	}

	public void setDenddate(String denddate) {
		this.denddate = denddate;
	}

	public String getDstartdate() {
		return dstartdate;
	}

	public void setDstartdate(String dstartdate) {
		this.dstartdate = dstartdate;
	}

	public String getEdate() {
		return edate;
	}

	public void setEdate(String edate) {
		this.edate = edate;
	}

	public String getProductName() {
		return productName;
	}

	public void setProductName(String productName) {
		this.productName = productName;
	}

	public String getProductNewPrice() {
		return productNewPrice;
	}

	public void setProductNewPrice(String productNewPrice) {
		this.productNewPrice = productNewPrice;
	}

	public String getProductPrice() {
		return productPrice;
	}

	public void setProductPrice(String productPrice) {
		this.productPrice = productPrice;
	}

	public String getSdate() {
		return sdate;
	}

	public void setSdate(String sdate) {
		this.sdate = sdate;
	}

	public Special getSpecial() {
		return special;
	}

	public void setSpecial(Special special) {
		this.special = special;
	}

}



```
