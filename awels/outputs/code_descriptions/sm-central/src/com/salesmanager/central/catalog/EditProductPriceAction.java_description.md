# EditProductPriceAction.java

## Review

## 1. Summary

| Aspect | Description |
|--------|-------------|
| **Purpose** | Handles CRUD operations on a product’s pricing information (add, modify, delete) and prepares data for display. |
| **Key Components** | *`EditProductPriceAction`* – Struts‑like action that communicates with business services. <br>*`CatalogService`*, *`CommonService`* – DAO‑style services used to fetch and persist domain objects. <br>*`ProductPrice`*, *`ProductPriceDescription`* – JPA‑style entities representing pricing and its translations. |
| **Design Patterns** | *Service Locator* (`ServiceFactory`) to obtain services.<br>*DAO/Repository* pattern via `CatalogService`/`CommonService`. <br>*Action* (command) pattern – the class encapsulates the user request. <br>*Localization* support using `LocaleUtil`, `LanguageUtil`, `CountryUtil`. |
| **Frameworks / Libraries** | Apache Commons Lang (`StringUtils`), Log4j, JPA/Hibernate (inferred from entity IDs), Struts‑2 (by naming convention: `SUCCESS`, `INPUT`, `ERROR` return strings). |

---

## 2. Detailed Description

### High‑level flow

1. **User Request**  
   The web UI submits a form to `editProductPrice()` or `displayProductPrice()`.

2. **Initialization**  
   - `prepareLanguages()` sets up language maps for internationalization.  
   - `setPageTitle()` sets the page title via resource bundles.

3. **Determine Action**  
   The `action` field indicates:
   - `-1` → add a new price
   - `0`  → modify an existing price
   - `1`  → delete an existing price

4. **Process the Request**  
   - **Add**:  
     * Retrieves core module information (price type).  
     * Validates the input amount, populates `price` entity, and builds descriptions for each language.  
   - **Modify**:  
     * Copies module/type data from the existing price (fetched by ID).  
     * Validates the amount and updates the entity.  
   - **Delete**:  
     * Calls `CatalogService.deleteProductPrice()` and returns early.

5. **Persist**  
   `CatalogService.saveOrUpdateProductPrice(price)` writes the price (and its descriptions) to the database.

6. **Maintain Default Flag**  
   If the new/modified price is flagged as default, all other prices for that product are updated to non‑default.

7. **Prepare UI Data**  
   `preparePriceDetails()` loads:
   * All price modules for the current country.  
   * The target product and its prices.  
   * Product descriptions in the current language.

8. **Return**  
   The action returns one of the Struts result strings (`SUCCESS`, `INPUT`, `ERROR`, `AUTHORIZATIONEXCEPTION`).

### Assumptions & Constraints

| Item | Assumption / Constraint |
|------|------------------------|
| `reflanguages` | Provided by `BaseAction` – maps language indices to language IDs. |
| `productPriceAmount` | Must be a valid currency string in the current locale. |
| `priceNames` | One entry per language index; must be supplied for new prices. |
| `product` | Must be non‑null and authorized for the current user. |
| **Locale/Country** | Derived from `Context` (`ctx.getCountryid()`, `ctx.getCurrency()`, `ctx.getLang()`). |
| **Concurrency** | No optimistic locking or versioning shown – potential race conditions when multiple users edit the same price. |
| **Null‑safety** | Raw collections (`Set`, `Collection`) and unchecked casts are used throughout. |

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `public String editProductPrice()` | Handles add/modify/delete of a product price; prepares UI attributes. | none | `SUCCESS`, `INPUT`, or error codes | Sets page title, adds field errors, modifies database via services. |
| `private void preparePriceDetails() throws Exception` | Loads price modules, product data, and translations; populates request attributes. | none | void | Writes to `HttpServletRequest` attributes; sets `productName`. |
| `public String displayProductPrice()` | Loads price data for view; returns success/error. | none | `SUCCESS`, `AUTHORIZATIONEXCEPTION`, `ERROR` | Calls `preparePriceDetails()`. |
| Getters/Setters | Standard property accessors for `price`, `product`, `prices`, `pricesModules`, `productPriceAmount`, `action`, `productName`, `priceNames`. | none | property values | No side effects. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.service.ServiceFactory` | Third‑party / internal | Service locator. |
| `com.salesmanager.core.service.catalog.CatalogService` | Third‑party | DAO for catalog entities. |
| `com.salesmanager.core.service.common.CommonService` | Third‑party | DAO for core modules, localization. |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utilities. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.util.*` | Third‑party | Currency, country, label, language, locale utilities. |
| `javax.servlet.http.HttpServletRequest` | Platform | Provided by servlet container. |
| `java.math.BigDecimal` | JDK | Currency value. |
| `java.util.*` | JDK | Collections. |
| `com.salesmanager.core.entity.catalog.*` | Domain | JPA entities. |
| `com.salesmanager.core.entity.reference.CoreModuleService` | Domain | Core module representation. |
| `com.salesmanager.core.constants.ProductConstants` | Domain | Constant definitions. |

---

## 5. Additional Notes

### Strengths
* **Separation of Concerns** – Uses services to interact with persistence.  
* **Localization** – Explicit handling of language and country specific data.  
* **Error Handling** – Distinct paths for validation errors (`INPUT`) vs. system errors.  

### Potential Issues & Edge Cases
| Area | Issue | Impact | Mitigation |
|------|-------|--------|------------|
| **Raw types** | `Set prices`, `Collection pricesModules` use raw `Set/Collection`. | Compile‑time warnings, possible `ClassCastException` at runtime. | Use generics (`Set<ProductPrice>`, `Collection<CoreModuleService>`). |
| **Unchecked casts** | E.g., `(Integer) i.next()`. | Runtime cast failures if the collection changes. | Store typed collections; remove casts. |
| **Hard‑coded action codes** | Magic numbers `-1`, `0`, `1`. | Hard to understand; risk of misuse. | Replace with `enum Action { ADD, MODIFY, DELETE }` or constants. |
| **Null checks** | `price.getProductPriceAmount()`, `price.getProductPriceModuleName()`. | Null pointer exceptions if the entity is incomplete. | Validate earlier or use defensive programming. |
| **Currency validation** | `CurrencyUtil.validateCurrency` may throw a generic `Exception`. | All errors treated the same; user may not know specific issue. | Catch specific exception types and provide clearer messages. |
| **Concurrency** | No locking; simultaneous edits may overwrite each other. | Data integrity issues. | Add optimistic locking (version field) or transaction isolation. |
| **Missing CSRF protection** | The action modifies state without visible token checks. | Vulnerable to CSRF attacks. | Add CSRF tokens or framework‑provided protection. |
| **Logging** | `log.error(e)` logs the stack trace but not the user‑friendly message. | Hard to debug in production. | Log exception message and stack trace; maybe use a logger helper. |
| **Internationalization** | Hard‑coded error keys (`error.message.price.format`). | Potential duplication of messages. | Centralize error codes or use `@ActionMessages`/`@FieldError`. |
| **Error paths** | `catch (Exception e)` swallows all errors and sets a generic technical message. | Users may not know why the operation failed. | Differentiate between validation and system errors. |

### Suggested Enhancements
1. **Use Generics & Type Safety** – Replace raw collections with generics throughout.  
2. **Define an `enum` for Actions** – Improves readability and type safety.  
3. **Centralize Validation** – Move price amount validation to a dedicated validator component.  
4. **Add Optimistic Locking** – Include a `version` field in `ProductPrice` and check before update.  
5. **CSRF & Session Validation** – Ensure actions are performed in a valid session.  
6. **Unit Tests** – Write tests for each CRUD path using mock services.  
7. **Logging & Metrics** – Add structured logs and metrics for performance monitoring.  
8. **Refactor Duplicate Code** – Extract common code (e.g., module lookup) into helper methods.  
9. **Use Optional** – For nullable service results (`pprice`, `price`) to avoid NPEs.  
10. **Internationalization** – Externalize all user messages into a resource bundle.  

---

### Final Verdict

The action performs its intended CRUD operations and prepares the necessary UI data. However, it relies heavily on raw types, magic numbers, and unchecked casts, which can compromise type safety and maintainability. By modernizing the code with generics, enums, and better exception handling, and by addressing concurrency, CSRF, and logging concerns, the component will become more robust, easier to test, and future‑proof.

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
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.core.constants.ProductConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.catalog.ProductPriceDescription;
import com.salesmanager.core.entity.catalog.ProductPriceDescriptionId;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MessageUtil;

public class EditProductPriceAction extends BaseAction {

	private static Logger log = Logger.getLogger(EditProductPriceAction.class);
	private Product product;
	private ProductPrice price;// submited
	private Set<ProductPrice> prices;// available prices for a given product
	private Collection pricesModules;// available prices modules
	private String productPriceAmount;// amount submited
	private String productName;

	private List<String> priceNames = new ArrayList<String>();

	private int action = -1;// actions -1 (add), 0 (modify) 1 (delete)

	public String editProductPrice() {

		try {
			
			super.setPageTitle("label.product.productprices.title");

			Context ctx = super.getContext();

			super.prepareLanguages();

			ProductPrice productPrice = this.getPrice();

			Set prices = this.getProduct().getPrices();

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			ProductPrice pprice = cservice.getProductPrice(this.getPrice()
					.getProductPriceId());

			if (this.getAction() == -1) {// add
				// get core module service
				CommonService cs = (CommonService) ServiceFactory
						.getService(ServiceFactory.CommonService);
				CoreModuleService cms = cs.getModule(CountryUtil
						.getCountryIsoCodeById(ctx.getCountryid()), this
						.getPrice().getProductPriceModuleName());
				price.setProductPriceTypeId(cms.getCoreModuleServiceSubtype());// 1
																				// is
																				// one
																				// time,
																				// 2
																				// is
																				// recursive
			}

			if (this.getAction() == 1) {// delete
				cservice.deleteProductPrice(pprice);
				super.setSuccessMessage();
				return SUCCESS;
			}

			if (pprice != null && this.getAction() == 0) {// modify
				price.setProductPriceModuleName(pprice
						.getProductPriceModuleName());
				price.setProductPriceTypeId(pprice.getProductPriceTypeId());
			}

			boolean hasError = false;

			// validate submitedamount
			BigDecimal bdprice = null;
			try {
				bdprice = CurrencyUtil.validateCurrency(this
						.getProductPriceAmount(), ctx.getCurrency());
				price.setProductPriceAmount(bdprice);

			} catch (Exception e) {
				if (this.getAction() == -1) {
					super.addFieldError("productPriceAmount",
							getText("error.message.price.format"));
				} else {
					MessageUtil.addMessage(getServletRequest(), LabelUtil
							.getInstance()
							.getText("error.message.price.format"));
				}
				hasError = true;
			}

			if (this.getAction() == -1) {// add
				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String priceName = (String) this.getPriceNames().get(
							langcount);

					if (StringUtils.isBlank(priceName)) {
						super
								.addFieldError(
										"priceName[" + langcount + "]",
										getText("error.message.storefront.contentpagetitlerequired"));
						hasError = true;
					}

					int submitedlangid = (Integer) reflanguages.get(langcount);
					// create
					ProductPriceDescriptionId id = new ProductPriceDescriptionId();
					id.setLanguageId(submitedlangid);
					id.setProductPriceId(price.getProductPriceId());

					ProductPriceDescription pdescription = new ProductPriceDescription();
					pdescription.setId(id);
					pdescription.setProductPriceName(priceName);

					Set descs = price.getPriceDescriptions();
					if (descs == null) {
						descs = new HashSet();
					}

					descs.add(pdescription);

					price.setPriceDescriptions(descs);

				}

			}

			if (hasError) {
				return INPUT;
			}

			price.setProductId(product.getProductId());
			cservice.saveOrUpdateProductPrice(price);

			if (price.isDefaultPrice()) {
				List updatePrices = new ArrayList();
				// set all other one to false
				if (prices != null) {
					Iterator pricesIterator = prices.iterator();
					while (pricesIterator.hasNext()) {
						ProductPrice pp = (ProductPrice) pricesIterator.next();
						if (pp.getProductPriceId() != price.getProductPriceId()) {
							pp.setDefaultPrice(false);
							updatePrices.add(pp);
						}
					}
				}
				if (updatePrices.size() > 0) {
					cservice.saveOrUpdateProductPrices(updatePrices);
				}
			}

			this.preparePriceDetails();

			super.setSuccessMessage();
		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);

			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	private void preparePriceDetails() throws Exception {
		
		super.setPageTitle("label.product.productprices.title");

		Context ctx = super.getContext();

		super.prepareLanguages();

		// get all modules
		CommonService commonService = (CommonService) ServiceFactory
				.getService(ServiceFactory.CommonService);
		pricesModules = commonService.getModules(
				super.getLocale().getCountry(),
				ProductConstants.PRICE_MODULE_TYPE);

		Map modulesMap = new HashMap();
		if (pricesModules != null) {
			Iterator i = pricesModules.iterator();
			while (i.hasNext()) {
				CoreModuleService cms = (CoreModuleService) i.next();
				modulesMap.put(cms.getCoreModuleName(), cms.getDescription());
			}
		}

		super.getServletRequest().setAttribute("pricedescriptions", modulesMap);

		LocaleUtil
				.setLocaleToEntityCollection(pricesModules, super.getLocale());

		// get all ProductPrice for a Product
		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);
		product = cservice.getProduct(this.getProduct().getProductId());

		super.authorize(product);

		Set prices = product.getPrices();

		LocaleUtil.setLocaleToEntityCollection(prices, super.getLocale());

		super.getServletRequest().setAttribute("prices", prices);

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

	}

	public String displayProductPrice() {
		
		super.setPageTitle("label.product.productprices.title");

		try {

			this.preparePriceDetails();
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

	public Set<ProductPrice> getPrices() {
		return prices;
	}

	public void setPrices(Set<ProductPrice> prices) {
		this.prices = prices;
	}

	public Collection<CoreModuleService> getPricesModules() {
		return pricesModules;
	}

	public void setPricesModules(Collection<CoreModuleService> pricesModules) {
		this.pricesModules = pricesModules;
	}

	public String getProductPriceAmount() {
		return productPriceAmount;
	}

	public void setProductPriceAmount(String productPriceAmount) {
		this.productPriceAmount = productPriceAmount;
	}

	public int getAction() {
		return action;
	}

	public void setAction(int action) {
		this.action = action;
	}

	public String getProductName() {
		return productName;
	}

	public void setProductName(String productName) {
		this.productName = productName;
	}

	public List<String> getPriceNames() {
		return priceNames;
	}

	public void setPriceNames(List<String> priceNames) {
		this.priceNames = priceNames;
	}

}



```
