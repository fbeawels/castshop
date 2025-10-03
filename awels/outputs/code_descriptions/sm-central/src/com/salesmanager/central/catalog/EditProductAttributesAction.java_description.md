# EditProductAttributesAction.java

## Review

## 1. Summary  

**Purpose** – `EditProductAttributesAction` is a Struts‑2 (or similar MVC) action that allows a merchant to create, edit or delete product attributes and option values in a multi‑language e‑commerce catalog.  

**Key responsibilities**  

| Component | Role |
|-----------|------|
| `prepare()` | Loads merchant store, supported languages, and populates the `reflanguages` map (index → languageId). |
| `saveProductOptionTextValues()` | Persists textual descriptions of an option value for all languages. |
| `editProductOptionTextValues()` | Loads the current textual descriptions of an option value for editing. |
| `editProductAttributes()` | Handles modification or deletion of a `ProductAttribute`. |
| `addProductAttributes()` | Creates a new `ProductAttribute` (and, if necessary, a new `ProductOptionValue`). |
| `displayProductAttributes()` | Prepares lists of product options and option values to be rendered in the UI. |

**Design patterns / libraries**  

* **Service locator** – `ServiceFactory.getService(...)` is used to obtain `CatalogService` and `MerchantService`.  
* **Entity‑DTO** – JPA/Hibernate entities (`Product`, `ProductAttribute`, `ProductOption`, …) are passed through the action.  
* **Internationalization** – `LabelUtil` and `MessageUtil` are used for i18n messages; `LanguageUtil` maps locale names to numeric codes.  
* **Apache Commons** – `StringUtils` for string checks.  
* **Log4j** – `Logger` for error logging.  

---

## 2. Detailed Description  

### Flow of Execution  

1. **Invocation** – A web request maps to one of the public methods (`addProductAttributes`, `editProductAttributes`, …).  
2. **Common preparation** – `prepare()` is called by most actions to load the current merchant, its supported languages and build the `reflanguages` map.  
3. **Business logic** – Each action performs validation, interacts with the catalog service, and manipulates entities.  
4. **Result** – A Struts result string (`SUCCESS`, `INPUT`, `ERROR`) is returned; the view layer renders accordingly.  

### Assumptions & Constraints  

* The action assumes a `ProfileConstants.context` attribute is present in the HTTP session.  
* Merchant and product IDs are passed in the HTTP request (not shown in the snippet).  
* All data is stored in a relational database through Hibernate entities.  
* The code uses raw collections (e.g., `Collection optionList = new ArrayList();`), which can lead to type‑safety issues.  
* `reflanguages` is a map from **index → languageId**; the order is critical for correctly matching the UI list of languages.  

### Architecture & Design Choices  

* **Monolithic Action** – The class contains all CRUD logic for product attributes and option values. While convenient, it mixes concerns (validation, persistence, i18n).  
* **Service Locator** – Direct calls to `ServiceFactory` make unit testing harder; dependency injection (e.g., via Spring) would be cleaner.  
* **Error Handling** – Errors are logged and a generic `ERROR` result is returned; user‑friendly messages are posted via `MessageUtil`.  

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `prepare()` | Initializes merchant, languages, and `reflanguages`. | N/A | Sets page title, populates `languages` and `reflanguages`. |
| `saveProductOptionTextValues()` | Persists a set of `ProductOptionValueDescription` objects for a text option. | `optionId`, `optionDescriptions` | Saves via `CatalogService`; sets success/technical messages. |
| `editProductOptionTextValues()` | Loads existing descriptions for a text option into `optionDescriptions`. | `optionId` | Fills `optionDescriptions`. |
| `editProductAttributes()` | Updates or deletes an existing `ProductAttribute`. | `productAttribute`, `action`, `optionValuePrice` | Calls `CatalogService` to save or delete. |
| `addProductAttributes()` | Creates a new `ProductAttribute` (and possibly a new `ProductOptionValue`). | `productAttribute`, `optionDescriptions`, `optionIdMode`, `optionValuePrice`, `productAttributeWeight` | Persists via `CatalogService`. |
| `displayProductAttributes()` | Prepares lists of product options/values and sets them in the request for rendering. | `product` | Populates `optionList`, `optionValueList`, request attribute `attributes`. |
| Getters/Setters | Standard JavaBean accessors for all fields. | N/A | Provide property access. |

**Utility / Reusable** – None; all logic is tightly coupled to the action.

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.lang.StringUtils` | 3rd‑party | For string checks. |
| `org.apache.log4j.Logger` | 3rd‑party | Logging. |
| `com.salesmanager.*` | 3rd‑party (own codebase) | Entities, services, utilities. |
| Java EE / Servlet API | Standard | For `HttpServletRequest` handling. |

*All dependencies are either standard Java EE or project‑specific libraries.*

---

## 5. Additional Notes  

### 5.1 Code‑Quality Issues  

| Issue | Impact | Recommendation |
|-------|--------|----------------|
| **Raw types** (`Collection optionList = new ArrayList();`) | No compile‑time type safety, potential `ClassCastException`. | Replace with generics (`Collection<ProductOptionDisplay> optionList = new ArrayList<>();`). |
| **Magic numbers / hard‑coded strings** (`action == 0`, `"errors.technical"`, `-1`) | Reduces readability, hard to change. | Define constants/enums (`ACTION_MODIFY = 0`, `ACTION_DELETE = 1`). |
| **Repeated code** (`prepare()` called manually in every action) | Maintenance risk. | Use `@Before` interceptor or a base action that automatically calls `prepare()`. |
| **Exception handling** – catches generic `Exception`, logs and sets technical message. | Swallows specific errors, makes debugging harder. | Catch specific exceptions or propagate them and let global error handler deal with them. |
| **Logging** – only `log.error(e)` in catch blocks. | No context (method name, parameters). | Log with a message: `log.error("Failed to save option values", e);`. |
| **String comparisons** – `StringUtils.isBlank(name)` is good, but `name = ""` fallback may be unnecessary. | Minor but unnecessary. | Just use the trimmed value; if empty, keep it. |
| **Null checks** – e.g., `if (value.getDescriptions() != null ...)` are safe but can be simplified using Java 8 streams or optional. | Code verbosity. | Use Java 8 (`Optional`) for readability. |
| **Hardcoded locale mapping** – `LanguageUtil.getLanguageNumberCode(ctx.getLang())`. | Coupling to internal number system. | Prefer language ISO codes or a mapping table. |
| **No unit tests** – The action is procedural and difficult to test. | Hard to guarantee correctness. | Extract business logic into service classes that can be unit‑tested. |
| **UI / Model coupling** – The action builds DTOs (`ProductOptionDisplay`, `ProductOptionValueDisplay`) manually. | Violates separation of concerns. | Use a dedicated view‑model factory or a mapper. |

### 5.2 Security & Validation  

* **Merchant validation** – Checks `value.getMerchantId() == super.getContext().getMerchantid()` are good, but could be centralized.  
* **Input validation** – Price and weight are validated with `CurrencyUtil.validateCurrency/Measure`, but no check for null or empty strings before conversion.  
* **Cross‑Site Scripting (XSS)** – The UI might display raw `optionDescriptions` or `optionValueName`. Ensure output is escaped in the JSP.  

### 5.3 Performance  

* The `displayProductAttributes()` method loads all options and values and iterates over them twice (once to build `optionList`, once to build `optionValueList`). For catalogs with many options, this could be costly.  
* Consider fetching only the options and values that belong to the current product, or use pagination.

### 5.4 Future Enhancements  

1. **Dependency Injection** – Replace `ServiceFactory.getService(...)` with Spring or CDI injection.  
2. **Command Pattern** – Encapsulate each CRUD operation in its own command object.  
3. **DTO Layer** – Move UI‑specific view models (`ProductOptionDisplay`, `ProductOptionValueDisplay`) to a dedicated DTO layer.  
4. **Validation Framework** – Use Bean Validation (JSR‑380) for price, weight, and other fields.  
5. **Unit & Integration Tests** – Write tests for service layer and mock the action using Struts' test harness.  
6. **Error Reporting** – Return detailed error objects (e.g., `ValidationError` list) instead of generic messages.  
7. **Internationalization** – Store language codes instead of numeric IDs to avoid fragile mapping.  

---

### 5.5 Quick‑Fix Checklist  

| Problem | Quick Fix |
|---------|-----------|
| Raw collections | `Collection<ProductOptionDisplay> optionList = new ArrayList<>();` |
| Magic `action` values | `enum Action { MODIFY, DELETE }` |
| Duplicate `prepare()` | Abstract base action with `@Before` interceptor |
| Catching generic `Exception` | Catch specific ones or rethrow |
| Logging context | `log.error("Error in saveProductOptionTextValues", e);` |
| Null checks before conversion | `if (StringUtils.isNotBlank(optionValuePrice)) { … }` |

---

## 6. Conclusion  

`EditProductAttributesAction` fulfills its functional requirements but is a classic example of a “fat action” that mixes validation, persistence, and UI preparation in a single class. The code would benefit from:

* **Type safety** (generics),  
* **Separation of concerns** (service layer + DTOs),  
* **Consistent error handling** and logging,  
* **Cleaner configuration** (dependency injection, constants/enums).  

Refactoring along these lines will improve maintainability, testability, and overall code quality.

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

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescription;
import com.salesmanager.core.entity.catalog.ProductOptionDescriptionId;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.catalog.ProductOptionValueDescription;
import com.salesmanager.core.entity.catalog.ProductOptionValueDescriptionId;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;

public class EditProductAttributesAction extends BaseAction {

	private Logger log = Logger.getLogger(EditProductAttributesAction.class);

	private Map<Integer, Integer> reflanguages = new HashMap();// reference
																// count -
																// languageId

	private List<String> optionDescriptions = new ArrayList<String>();

	private String productName = "";
	private int optionIdMode = 0;// 0 = select / checkbox ... 1= text

	public int getOptionIdMode() {
		return optionIdMode;
	}

	public void setOptionIdMode(int optionIdMode) {
		this.optionIdMode = optionIdMode;
	}

	public String getProductName() {
		return productName;
	}

	public void setProductName(String productName) {
		this.productName = productName;
	}

	private long optionId = -1;// for editing text option values

	public long getOptionId() {
		return optionId;
	}

	public void setOptionId(long optionId) {
		this.optionId = optionId;
	}

	public List<String> getOptionDescriptions() {
		return optionDescriptions;
	}

	public void setOptionDescriptions(List<String> optionDescriptions) {
		this.optionDescriptions = optionDescriptions;
	}

	private Collection<Language> languages;// used in the page as an index

	private Product product;

	private int lineId;// from shopping cart

	private Collection optionList = new ArrayList();
	private Collection optionValueList = new ArrayList();

	private ProductAttribute productAttribute;

	private String optionValuePrice;
	private String productAttributeWeight;

	private int action = -1;

	public void prepare() throws Exception {
		
		super.setPageTitle("label.product.productproperties.title");

		MerchantService service = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		MerchantStore mstore = service.getMerchantStore(merchantid);

		if (mstore == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
		}

		Map languagesMap = mstore.getGetSupportedLanguages();

		languages = languagesMap.values();// collection reverse the map

		int count = 0;
		Iterator langit = languagesMap.keySet().iterator();
		while (langit.hasNext()) {
			Integer langid = (Integer) langit.next();
			Language lang = (Language) languagesMap.get(langid);
			reflanguages.put(count, langid);
			count++;
		}

	}

	/**
	 * For Text ProductOptionValueDscription
	 * 
	 * @return
	 */
	public String saveProductOptionTextValues() {

		try {

			prepare();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			if (this.reflanguages.size() == 0) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"errors.profile.storenotcreated"));
				return INPUT;
			}

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			ProductOptionValue value = cservice.getProductOptionValue(this
					.getOptionId());

			if (value == null) {
				return INPUT;
			}

			if (value.getMerchantId() != super.getContext().getMerchantid()) {
				return INPUT;
			}

			Set valuesDescriptions = value.getDescriptions();

			Map iddescmap = new HashMap();

			if (valuesDescriptions != null && valuesDescriptions.size() > 0) {
				Iterator i = valuesDescriptions.iterator();
				while (i.hasNext()) {
					ProductOptionValueDescription o = (ProductOptionValueDescription) i
							.next();
					iddescmap.put(o.getId().getLanguageId(), o);
				}
			}

			if (this.getOptionDescriptions().size() > 0) {

				// text
				Iterator i = reflanguages.keySet().iterator();
				Set hs = new HashSet();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String name = (String) this.getOptionDescriptions().get(
							langcount);
					if (StringUtils.isBlank(name)) {
						name = "";
					}
					int submitedlangid = (Integer) reflanguages.get(langcount);
					ProductOptionValueDescription povd = (ProductOptionValueDescription) iddescmap
							.get(submitedlangid);
					ProductOptionValueDescriptionId id = null;
					if (povd == null) {
						povd = new ProductOptionValueDescription();
						id = new ProductOptionValueDescriptionId();
						id.setProductOptionValueId(this.getOptionId());
					}
					id = povd.getId();
					id.setLanguageId(submitedlangid);
					povd.setId(id);
					povd.setProductOptionValueName(name);
					hs.add(povd);
				}

				cservice.saveOrUpdateOptionValueDescriptions(hs);

			}

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

		return SUCCESS;

	}

	/**
	 * For Text ProductOptionValueDscription
	 * 
	 * @return
	 */
	public String editProductOptionTextValues() {

		try {

			prepare();
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			ProductOptionValue option = cservice.getProductOptionValue(this
					.getOptionId());

			int merchantId = option.getMerchantId();
			if (merchantId != super.getContext().getMerchantid()) {
				return "";
			}

			Map iddescmap = new HashMap();

			if (option != null && option.getDescriptions() != null
					&& option.getDescriptions().size() > 0) {
				Iterator i = option.getDescriptions().iterator();
				while (i.hasNext()) {
					ProductOptionValueDescription o = (ProductOptionValueDescription) i
							.next();
					iddescmap.put(o.getId().getLanguageId(), o);
				}
			}

			// iterate through languages for appropriate order
			for (int count = 0; count < reflanguages.size(); count++) {
				int langid = (Integer) reflanguages.get(count);
				ProductOptionValueDescription desc = (ProductOptionValueDescription) iddescmap
						.get(langid);
				if (desc != null) {
					optionDescriptions.add(desc.getProductOptionValueName());
				}
			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String editProductAttributes() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			prepare();

			ProductAttribute attribute = this.getProductAttribute();
			if (attribute == null || this.getProduct() == null) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
				return SUCCESS;
			}

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			if (this.getAction() == 0) {// modify

				// validate price
				if (this.getOptionValuePrice() == null) {
					super.addFieldError("optionValuePrice",
							getText("error.message.price.format"));
					return SUCCESS;
				}

				BigDecimal price = null;
				try {
					price = CurrencyUtil.validateCurrency(this
							.getOptionValuePrice(), ctx.getCurrency());
					attribute.setOptionValuePrice(price);

				} catch (Exception e) {
					super.addFieldError("optionValuePrice",
							getText("error.message.price.format"));
					return SUCCESS;
				}

				// get the product attribute

				ProductAttribute editableAttribute = cservice
						.getProductAttribute(attribute.getProductAttributeId());

				editableAttribute.setAttributeDefault(attribute
						.isAttributeDefault());
				editableAttribute.setAttributeRequired(attribute
						.isAttributeRequired());
				editableAttribute.setProductAttributeIsFree(attribute
						.isProductAttributeIsFree());
				editableAttribute.setProductOptionSortOrder(attribute
						.getProductOptionSortOrder());
				editableAttribute.setOptionValuePrice(attribute
						.getOptionValuePrice());
				editableAttribute.setProductAttributeWeight(attribute
						.getProductAttributeWeight());
				editableAttribute.setAttributeDisplayOnly(attribute.isAttributeDisplayOnly());

				cservice.saveOrUpdateProductAttribute(editableAttribute);

			} else {

				cservice.deleteProductAttribute(this.getProductAttribute()
						.getProductAttributeId());

			}

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String addProductAttributes() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();
			prepare();

			ProductAttribute attribute = this.getProductAttribute();

			if (attribute == null || this.getProduct() == null) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
				return SUCCESS;
			}

			// validate price
			if (this.getOptionValuePrice() == null) {
				// super.addFieldError("optionValuePrice",getText("error.message.price.format"));
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"error.message.price.format"));
				return SUCCESS;
			}

			BigDecimal price = null;
			try {
				price = CurrencyUtil.validateCurrency(this
						.getOptionValuePrice(), ctx.getCurrency());
				attribute.setOptionValuePrice(price);

			} catch (Exception e) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"error.message.price.format"));
				// super.addFieldError("optionValuePrice",getText("error.message.price.format"));
				return SUCCESS;
			}

			// weight validation
			BigDecimal weight = null;
			try {
				String w = String.valueOf(this.getProductAttributeWeight());
				weight = CurrencyUtil.validateMeasure(w, ctx.getCurrency());
				attribute.setProductAttributeWeight(weight);

			} catch (Exception e) {
				// super.addFieldError("productAttributeWeight",getText("invalid.fieldvalue.product.productWeight"));
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"invalid.fieldvalue.product.productWeight"));
				return SUCCESS;

			}

			attribute.setProductId(this.getProduct().getProductId());

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			if (this.getOptionDescriptions().size() > 0
					&& this.getOptionIdMode() == 1) {

				ProductOptionValue pov = new ProductOptionValue();
				pov.setMerchantId(super.getContext().getMerchantid());
				pov.setProductOptionValueSortOrder(0);

				// text
				Iterator i = reflanguages.keySet().iterator();
				Set hs = new HashSet();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String name = (String) this.getOptionDescriptions().get(
							langcount);
					if (StringUtils.isBlank(name)) {
						name = "";
					}
					int submitedlangid = (Integer) reflanguages.get(langcount);
					ProductOptionValueDescription povd = new ProductOptionValueDescription();
					ProductOptionValueDescriptionId id = new ProductOptionValueDescriptionId();
					id.setProductOptionValueId(pov.getProductOptionValueId());
					id.setLanguageId(submitedlangid);
					povd.setId(id);
					povd.setProductOptionValueName(name);
					hs.add(povd);
				}
				if (hs.size() > 0) {
					pov.setDescriptions(hs);
				}

				cservice.saveOrUpdateProductOptionValue(pov);
				attribute.setOptionValueId(pov.getProductOptionValueId());

			}

			cservice.saveOrUpdateProductAttribute(attribute);

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String displayProductAttributes() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			prepare();

			if (this.getProduct() == null
					|| this.getProduct().getProductId() == 0) {
				log.error("Should have received a productId");
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
				return SUCCESS;
			}

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			Collection options = cservice
					.getProductOptions(ctx.getMerchantid());

			if (options != null) {
				ProductOption option = (ProductOption) options.toArray()[0];
				ProductOption optionWithValues = cservice
						.getProductOptionWithValues(option.getProductOptionId());

				Iterator i = options.iterator();
				while (i.hasNext()) {
					ProductOption o = (ProductOption) i.next();
					ProductOptionDisplay pod = new ProductOptionDisplay();
					pod.setProductOptionId(o.getProductOptionId());
					pod.setProductOptionName(String.valueOf(o
							.getProductOptionId()));
					Set descs = o.getDescriptions();
					if (descs != null) {
						Iterator descsiter = descs.iterator();
						while (descsiter.hasNext()) {
							ProductOptionDescription podesc = (ProductOptionDescription) descsiter
									.next();
							ProductOptionDescriptionId id = podesc.getId();
							if (id.getLanguageId() == LanguageUtil
									.getLanguageNumberCode(ctx.getLang())) {
								pod.setProductOptionName(podesc
										.getProductOptionName());
								break;
							}
						}
					}
					optionList.add(pod);

				}

				if (optionWithValues != null) {

					Set values = optionWithValues.getValues();
					if (values != null) {
						Iterator viter = values.iterator();
						while (viter.hasNext()) {
							ProductOptionValue pov = (ProductOptionValue) viter
									.next();
							ProductOptionValueDisplay povd = new ProductOptionValueDisplay();
							povd.setProductOptionValueId(pov
									.getProductOptionValueId());
							povd.setProductOptionValueName(String.valueOf(pov
									.getProductOptionValueId()));
							Set descs = pov.getDescriptions();
							if (descs != null) {
								Iterator descsiter = descs.iterator();
								while (descsiter.hasNext()) {
									ProductOptionValueDescription povdesc = (ProductOptionValueDescription) descsiter
											.next();
									ProductOptionValueDescriptionId id = povdesc
											.getId();
									if (id.getLanguageId() == LanguageUtil
											.getLanguageNumberCode(ctx
													.getLang())) {
										povd.setProductOptionValueName(povdesc
												.getProductOptionValueName());
										break;
									}
								}
							}
							optionValueList.add(povd);
						}
					}

				}
			}

			Collection attributes = cservice.getProductAttributes(this
					.getProduct().getProductId(), super.getLocale()
					.getLanguage());

			super.getServletRequest().setAttribute("attributes", attributes);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public Collection getOptionList() {
		return optionList;
	}

	public void setOptionList(Collection optionList) {
		this.optionList = optionList;
	}

	public Collection getOptionValueList() {
		return optionValueList;
	}

	public void setOptionValueList(Collection optionValueList) {
		this.optionValueList = optionValueList;
	}

	public ProductAttribute getProductAttribute() {
		return productAttribute;
	}

	public void setProductAttribute(ProductAttribute productAttribute) {
		this.productAttribute = productAttribute;
	}

	public String getOptionValuePrice() {
		return optionValuePrice;
	}

	public void setOptionValuePrice(String optionValuePrice) {
		this.optionValuePrice = optionValuePrice;
	}

	public int getAction() {
		return action;
	}

	public void setAction(int action) {
		this.action = action;
	}

	public int getLineId() {
		return lineId;
	}

	public void setLineId(int lineId) {
		this.lineId = lineId;
	}

	public String getProductAttributeWeight() {
		return productAttributeWeight;
	}

	public void setProductAttributeWeight(String productAttributeWeight) {
		this.productAttributeWeight = productAttributeWeight;
	}

	public Map<Integer, Integer> getReflanguages() {
		return reflanguages;
	}

	public void setReflanguages(Map<Integer, Integer> reflanguages) {
		this.reflanguages = reflanguages;
	}

	public Collection<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(Collection<Language> languages) {
		this.languages = languages;
	}

}



```
