# EditProductPriceDiscountAction.java

## Review

## 1. Summary

`EditProductPriceDiscountAction` is a Struts‑style action that lets an administrator view, edit, or delete a **special (discounted) price** for a product in the catalog.  
Key responsibilities:

| Component | Role |
|-----------|------|
| `displayProductPriceDiscount()` | Prepare all data needed for the “price‑discount” page and forward to the success view. |
| `editProductPriceDiscount()` | Validate user input, create or update a `ProductPriceSpecial`, persist it, and set a success message. |
| `deleteProductPriceDiscount()` | Remove an existing special price for the given product. |
| `prepareProductPriceDiscountDetails()` | Central helper that loads the product, price, special price, and product name, and populates UI helpers such as the duration selector (`days`). |

The action relies on a `CatalogService` to fetch and persist catalog entities, and on various helper utilities (`CurrencyUtil`, `DateUtil`, `LabelUtil`, `LanguageUtil`) for formatting and i18n. Logging is performed with Log4j.

---

## 2. Detailed Description

### Execution Flow

1. **User request → Action mapping**  
   The web framework dispatches a request to one of the three public methods based on the action name (`display`, `edit`, or `delete`).

2. **Authorization**  
   Each method begins by calling `super.setPageTitle(...)` and then, inside `prepareProductPriceDiscountDetails()`, verifies that the current user has permission to view or modify the requested product.

3. **Data Preparation**  
   `prepareProductPriceDiscountDetails()` loads:
   * The `Product` and its `ProductPrice`.
   * The optional `ProductPriceSpecial`.
   * The product name in the current locale.
   * A map of duration options (`days`) for the UI drop‑down.

4. **Display**  
   `displayProductPriceDiscount()` simply calls the preparer and returns `SUCCESS` so the view can render all the prepared fields.

5. **Edit**  
   `editProductPriceDiscount()` performs the following steps:
   * Re‑loads the current product/price objects for safety.
   * Validates the new discount amount using `CurrencyUtil.validateCurrency()`. On failure, a field error is added and the form is redisplayed.
   * Parses the start/end dates. If parsing fails, defaults to the current date.
   * Sets the start/end dates and duration on the `ProductPriceSpecial` entity.
   * Persists the entity with `CatalogService.saveOrUpdateProductPriceSpecial()`.
   * Sets a success message.

6. **Delete**  
   `deleteProductPriceDiscount()` retrieves the `ProductPriceSpecial` for the price and deletes it via the service, then sets a success message.

7. **Error Handling**  
   All three public methods catch `AuthorizationException` and generic `Exception`. The former causes a dedicated `AUTHORIZATIONEXCEPTION` result; the latter logs the error and returns the generic `ERROR` result.

### Design Choices

* **Separation of Concerns** – All business logic is delegated to `CatalogService`; the action merely orchestrates data flow and UI feedback.
* **Use of Helper Utilities** – `CurrencyUtil`, `DateUtil`, etc., keep formatting and parsing logic in one place.
* **Explicit Locale Handling** – Product names and labels are pulled according to the user’s language via `LanguageUtil` and `LabelUtil`.

### Assumptions & Constraints

* The request parameters (`price`, `product`, `productNewPrice`, etc.) are correctly populated by the framework (typical Struts 2 model driven approach).
* The `CatalogService` implementation is thread‑safe and transactional.
* The action does not perform any cross‑field validation (e.g., ensuring the end date is after the start date).

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `displayProductPriceDiscount()` | Show the discount editing page. | N/A (framework populates fields) | Sets page title, loads data via `prepareProductPriceDiscountDetails()`, returns `SUCCESS`. |
| `editProductPriceDiscount()` | Validate and persist a new or updated discount. | N/A | Parses/validates `productNewPrice`, dates, duration; creates/updates `ProductPriceSpecial`; sets success or error messages; returns `SUCCESS`/`ERROR`. |
| `deleteProductPriceDiscount()` | Remove an existing special price. | N/A | Calls service to delete; sets success message; returns `SUCCESS`/`ERROR`. |
| `prepareProductPriceDiscountDetails()` | Central helper that loads product, price, special price, product name, and duration options. | N/A | Sets fields (`price`, `product`, `special`, `productPrice`, `productNewPrice`, `sdate`, `edate`, `day`, `productName`, `days`) for the view. |
| Getters/Setters | Standard JavaBean accessors for the action’s fields. | Various | Return or set corresponding property. |

---

## 4. Dependencies

| Library / Framework | Role | Standard / 3rd‑Party |
|---------------------|------|---------------------|
| `org.apache.log4j.Logger` | Logging | 3rd‑party |
| `com.salesmanager.central.*` | BaseAction, Context, AuthorizationException | Project-specific |
| `com.salesmanager.core.*` | Domain entities (`Product`, `ProductPrice`, etc.) and services (`CatalogService`) | Project-specific |
| `com.salesmanager.core.util.*` (`CurrencyUtil`, `DateUtil`, `LabelUtil`, `LanguageUtil`) | Formatting, i18n, currency handling | Project-specific |
| `java.util.*` | Collections, dates | JDK standard |
| `java.math.BigDecimal` | Monetary values | JDK standard |
| `java.text.*` | Date formatting/parsing | JDK standard |

No external frameworks beyond the internal SalesManager core library and Log4j are used.

---

## 5. Additional Notes & Recommendations

### 5.1. Generic Types
The class uses raw types (`Map`, `Set`, `Iterator`). Modern Java (≥ 5) encourages generics to avoid unchecked warnings and potential `ClassCastException`s.

```java
private Map<String, String> days = new HashMap<>();
// ...
Set<ProductDescription> descriptionset = product.getDescriptions();
Iterator<ProductDescription> i = descriptionset.iterator();
```

### 5.2. Date Handling
* The date parsing logic silently falls back to the current date if parsing fails. This can hide user mistakes. Consider adding a field error or returning the user to the form with a clear message.
* The method does **not** verify that the end date is after the start date. A validation rule should be added.
* `SimpleDateFormat` is not thread‑safe; however, it is instantiated locally here, so it’s fine.

### 5.3. Numeric Conversion
`new Integer(this.getDay())` is deprecated; use `Integer.valueOf()` or `Integer.parseInt()`.

```java
ppp.setProductPriceSpecialDurationDays(Integer.valueOf(this.getDay()));
```

### 5.4. Field Error on Currency Validation
The field error uses the key `"productPrice"`, which may not match the input field name. Verify that the form field is named `productPrice` or adjust the key accordingly.

### 5.5. Null‑Pointer Risks
* If `price` or `product` is `null` when `editProductPriceDiscount()` or `deleteProductPriceDiscount()` is called, a NPE will occur. The action assumes that the framework always populates these fields.
* `productPrice` and `productNewPrice` are formatted strings; they can be `null` if the price does not exist, which may break the view.

### 5.6. Internationalization
The `days` map is built using `LabelUtil.getInstance().getText(...)`. This is acceptable, but the action could expose a typed list of `DayOption` objects instead of a `Map` to make the view logic clearer.

### 5.7. Logging
`log.error(e)` prints the stack trace, but consider logging the context (e.g., product ID) to aid debugging:

```java
log.error("Failed to edit discount for productId=" + this.getProduct().getProductId(), e);
```

### 5.8. Code Duplication
Both `displayProductPriceDiscount()` and `editProductPriceDiscount()` call `prepareProductPriceDiscountDetails()` and `super.setPageTitle(...)`. Extract a private helper method if this pattern grows.

### 5.9. Security
Authorization is performed inside `prepareProductPriceDiscountDetails()`. For `deleteProductPriceDiscount()` the authorization check is omitted; it relies on the same method indirectly via `getProductPriceSpecial()`. Explicitly authorizing the product before deletion is safer.

### 5.10. Future Enhancements
* **Cross‑field validation** – ensure that the new price is lower than the original, dates are valid, and duration matches the selected range.
* **Unit tests** – mock `CatalogService` and `Context` to test each branch of `editProductPriceDiscount()`.
* **Transactional boundaries** – rely on the service layer for transactions; ensure `CatalogService` is annotated with `@Transactional`.
* **Modern Java** – consider moving to Java 8+ streams for processing the description set.

---

### Bottom Line

The action implements its core responsibilities correctly and follows a clear structure. However, it would benefit from modernizing its use of generics, improving validation, and adding defensive coding against null values and parsing errors. Addressing the above recommendations will make the code more robust, maintainable, and easier to test.

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
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;

import org.apache.log4j.Logger;

import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.catalog.ProductPriceSpecial;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;

public class EditProductPriceDiscountAction extends BaseAction {

	private ProductPrice price;// parameter
	private Product product;// parameter
	private ProductPriceSpecial special;// used in the jsp
	private String productPrice;// display original price
	private Map days = new HashMap();

	private String sdate;
	private String edate;
	private String productName;

	private String day;// submited duration
	private String dstartdate;// submited
	private String denddate;// submited
	private String productNewPrice;// submited price

	private static Logger log = Logger
			.getLogger(EditProductPriceDiscountAction.class);

	public String displayProductPriceDiscount() {
		
		
		super.setPageTitle("label.product.productprices.discount");
		
		try {

			this.prepareProductPriceDiscountDetails();
			return SUCCESS;

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			return ERROR;
		}
	}

	public String editProductPriceDiscount() {
		
		super.setPageTitle("label.product.productprices.discount");

		try {

			Context ctx = super.getContext();

			this.prepareProductPriceDiscountDetails();

			BigDecimal discountPrice;

			ProductPrice pp = this.getPrice();
			ProductPriceSpecial ppp;
			if (pp.getSpecial() != null) {
				ppp = pp.getSpecial();
			} else {
				ppp = new ProductPriceSpecial();
				ppp.setProductPriceId(pp.getProductPriceId());
			}

			// validate amount
			try {
				discountPrice = CurrencyUtil.validateCurrency(this
						.getProductNewPrice(), ctx.getCurrency());
				ppp.setProductPriceSpecialAmount(discountPrice);
			} catch (Exception e) {
				super.addFieldError("productPrice",
						getText("error.message.price.format"));
				return SUCCESS;
			}

			Date dt = new Date();
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

			ppp.setProductPriceSpecialStartDate(sDate);
			ppp.setProductPriceSpecialEndDate(eDate);

			ppp.setProductPriceSpecialDurationDays(new Integer(this.getDay()));

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			cservice.saveOrUpdateProductPriceSpecial(ppp);

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			return ERROR;
		}

		return SUCCESS;
	}

	public String deleteProductPriceDiscount() {
		
		super.setPageTitle("label.product.productprices.discount");

		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			ProductPriceSpecial ppp = cservice.getProductPriceSpecial(this
					.getPrice().getProductPriceId());


			if (ppp != null) {
				cservice.deleteProductPriceSpecial(ppp);
			}

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			return ERROR;
		}

		return SUCCESS;
	}

	private void prepareProductPriceDiscountDetails() throws Exception {
		
		super.setPageTitle("label.product.productprices.discount");

		Context ctx = super.getContext();

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		product = cservice.getProduct(this.getProduct().getProductId());

		super.authorize(product);

		// get price
		price = cservice.getProductPrice(this.getPrice().getProductPriceId());

		if (price == null || price.getProductId() != product.getProductId()) {
			throw new AuthorizationException("No price defined");
		}

		productPrice = CurrencyUtil.displayFormatedAmountNoCurrency(price
				.getProductPriceAmount(), ctx.getCurrency());

		// get discount
		special = price.getSpecial();

		if (special != null) {
			productNewPrice = CurrencyUtil.displayFormatedAmountNoCurrency(
					special.getProductPriceSpecialAmount(), ctx.getCurrency());
			sdate = DateUtil.formatDate(special
					.getProductPriceSpecialStartDate());
			edate = DateUtil
					.formatDate(special.getProductPriceSpecialEndDate());
			day = String.valueOf(special.getProductPriceSpecialDurationDays());
		}

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

		days = new HashMap();
		days.put("-1", LabelUtil.getInstance().getText(super.getLocale(),
				"label.product.productprices.discount.selectduration"));
		days.put("30", LabelUtil.getInstance().getText(super.getLocale(),
				"label.product.productprices.discount.30days"));
		days.put("60", LabelUtil.getInstance().getText(super.getLocale(),
				"label.product.productprices.discount.60days"));
		days.put("90", LabelUtil.getInstance().getText(super.getLocale(),
				"label.product.productprices.discount.90days"));
		days.put("120", LabelUtil.getInstance().getText(super.getLocale(),
				"label.product.productprices.discount.120days"));

	}

	public ProductPrice getPrice() {
		return price;
	}

	public void setPrice(ProductPrice price) {
		this.price = price;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public ProductPriceSpecial getSpecial() {
		return special;
	}

	public void setSpecial(ProductPriceSpecial special) {
		this.special = special;
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

	public Map getDays() {
		return days;
	}

	public void setDays(Map days) {
		this.days = days;
	}

	public String getDay() {
		return day;
	}

	public void setDay(String day) {
		this.day = day;
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

}



```
