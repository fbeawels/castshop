# ProductPreviewAction.java

## Review

## 1. Summary

**Purpose**  
`ProductPreviewAction` is a Struts‑style action that builds the data necessary to render a *product preview* page. It retrieves a product, its associated options/attributes, and calculates the price that will be displayed to the user.

**Key Components**

| Component | Role |
|-----------|------|
| `Product` | The domain object representing the item to preview. |
| `ProductOptionDescriptor` | A helper that groups options/attributes by option type and holds the values that belong to that option. |
| `CatalogService`, `MerchantService`, `ReferenceService` | Service layer beans that supply product data, store information, and configuration values. |
| `ProductUtil` | Utility that formats the product price (with or without selected attributes). |
| `specifications`, `options` | Two collections of `ProductOptionDescriptor` objects – *specifications* are read‑only attributes, *options* are priced options. |

**Design Patterns & Libraries**

* **Service Locator** – `ServiceFactory.getService(...)` is used to obtain service instances.
* **Struts Action** – The class extends `PageBaseAction` and implements the `display()` method that returns a result string (`SUCCESS` or `unauthorized`).
* **Logging** – `org.apache.log4j.Logger` is used for error reporting.

---

## 2. Detailed Description

1. **Initialization**  
   *The Struts framework populates `product` (via a setter) before the action is invoked.*

2. **`display()` Flow**

| Step | Description |
|------|-------------|
| 1 | `setPageTitle` sets the page title. |
| 2 | If `product` is `null`, the action immediately returns `"unauthorized"` – preventing any further processing. |
| 3 | **Merchant & Store** – Retrieve the `MerchantStore` instance for the current context using `MerchantService`. |
| 4 | **Product Load** – `CatalogService.getProduct()` fetches the full product details (including locale information). |
| 5 | **Attributes Retrieval** – `CatalogService.getProductAttributes()` returns all attribute/value pairs for the product in the current language. |
| 6 | **Attribute & Option Processing** – The method iterates through the attributes and groups them into two collections: `specifications` (display‑only) and `options` (priced). For each attribute it: <br>• Creates/updates a `ProductOptionDescriptor` <br>• Adds the attribute (`pa`) as a value <br>• Marks defaults and records default options. |
| 7 | **Configuration** – `ReferenceService.getModuleConfigurationsKeyValue(...)` fetches store‑specific configuration map. |
| 8 | **Price Calculation** – If any default options exist, the price is calculated using `formatHTMLProductPriceWithAttributes`; otherwise a simple price format is used. |
| 9 | **Exception Handling** – Any exception logs an error and sets a generic technical message on the action. |
|10 | Returns `SUCCESS` to dispatch to the preview view. |

3. **Assumptions & Constraints**

* The action is **request‑scoped** – all mutable fields are safe from concurrent access.  
* The service locator returns thread‑safe singletons.  
* Attribute lists are expected to be small; the current O(n) loop is acceptable but could be optimized.  
* The `ProductOptionDescriptor` class exposes `addValue()`, `setOptionId()`, etc., but the API contract is not validated in this code.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `public String display()` | Main action method that prepares the preview model and returns the result string. | None | `String` (`SUCCESS` or `"unauthorized"`) | Modifies instance fields (`product`, `productPrice`, `storeConfiguration`, `specifications`, `options`). |
| `public Product getProduct()` / `public void setProduct(Product product)` | Bean property for the product. | `Product` | `Product` | No side‑effects. |
| `public String getProductPrice()` / `public void setProductPrice(String productPrice)` | Price string shown on the page. | `String` | `String` | No side‑effects. |
| `public Map getStoreConfiguration()` / `public void setStoreConfiguration(Map storeConfiguration)` | Store‑level configuration map. | `Map` | `Map` | No side‑effects. |
| `public Collection<ProductOptionDescriptor> getSpecifications()` / `public void setSpecifications(Collection<ProductOptionDescriptor> specifications)` | Read‑only attributes. | `Collection` | `Collection` | No side‑effects. |
| `public Collection<ProductOptionDescriptor> getOptions()` / `public void setOptions(Collection<ProductOptionDescriptor> options)` | Priced options. | `Collection` | `Collection` | No side‑effects. |

**Reusable / Utility Methods** – None defined in this class; all logic is embedded in `display()`.

---

## 4. Dependencies

| External | Type | Notes |
|----------|------|-------|
| `org.apache.log4j.Logger` | Logging | Standard, widely used. |
| `com.salesmanager.central.PageBaseAction` | Base Struts action | Provides `getContext()`, `setTechnicalMessage()`, `SUCCESS`, etc. |
| `com.salesmanager.core.entity.catalog.*` | Domain models | `Product`, `ProductAttribute`, `ProductOption`, `ProductOptionDescriptor`, `ProductOptionValue`. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Domain model | Store metadata. |
| `com.salesmanager.core.service.ServiceFactory` | Service locator | Retrieves services by key. |
| `com.salesmanager.core.service.catalog.CatalogService` | Service | Handles product data. |
| `com.salesmanager.core.service.merchant.MerchantService` | Service | Retrieves merchant store. |
| `com.salesmanager.core.service.reference.ReferenceService` | Service | Retrieves configuration values. |
| `com.salesmanager.core.util.ProductUtil` | Utility | Formats product prices. |

All dependencies are **third‑party** or project‑internal. No platform‑specific APIs are used.

---

## 5. Additional Notes & Recommendations

### 5.1 Code Quality & Maintainability

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw Collections** – `specifications`, `options`, `storeConfiguration` use raw types. | Compile‑time warnings; potential `ClassCastException`. | Parameterize generics: `Collection<ProductOptionDescriptor> specifications = new ArrayList<>();` etc. |
| **Hard‑coded String Literals** – `"unauthorized"` and `"label.product.preview"`. | Difficult to localise; risk of typos. | Use constants or resource bundles. |
| **No `@Override` Annotation** on `display()`. | Missed compiler check. | Add `@Override`. |
| **Exception Path Returns `SUCCESS`** – Even if an exception occurs, the method still returns `SUCCESS`. | User may see a partially built page with missing data. | Return an error result or re‑throw the exception after setting the technical message. |
| **Logging** – `log.error(e)` prints only the stack trace implicitly; explicit `e.printStackTrace()` is not used. | Sufficient for most cases, but consider `log.error("Error displaying product preview", e);`. | Add a descriptive message. |
| **Thread‑Safety** – The action is request‑scoped, but the use of instance fields means the class is not thread‑safe if Struts ever changes its lifecycle. | Minor; generally safe in current environment. | Document scope explicitly. |
| **Large Method** – `display()` contains more than 200 lines of logic. | Hard to read & test. | Extract sub‑methods (`loadProduct()`, `processAttributes()`, `calculatePrice()`), or move business logic to a service layer. |
| **Iterator & Enhanced For Loop** – Uses legacy `Iterator`. | Verbose; harder to read. | Replace with enhanced `for` loops. |
| **Unnecessary Casts** – `(MerchantService) ServiceFactory.getService(...)`. | Compile‑time unchecked cast. | Use generic factory or dependency injection. |

### 5.2 Functional Bug

**Duplicate/Incorrect Specification Handling**

```java
if (pa.isAttributeDisplayOnly()) {
    if (lastSpecificationOptionId == -1) {
        // ...
    } else {
        if (pa.getOptionId() != lastOptionId) {
            // <--- bug: should compare to lastSpecificationOptionId
        }
    }
}
```

*The condition incorrectly uses `lastOptionId` instead of `lastSpecificationOptionId`, causing either duplicate `ProductOptionDescriptor` objects or missing ones. The fix is to compare with `lastSpecificationOptionId`.*

### 5.3 Logical Simplification

* The current logic groups attributes by iterating over a flat list and tracking `lastOptionId`/`lastSpecificationOptionId`.  
* A more robust approach: create a `Map<Long, ProductOptionDescriptor>` keyed by the option id, then add each attribute to the corresponding descriptor. This eliminates the need for manual id tracking and reduces error‑prone conditions.

### 5.4 Performance

* The attribute collection is iterated twice (once to build descriptors, once for defaults).  
* A single pass using the map approach above would reduce the complexity to O(n).

### 5.5 Security

* The action checks for `product == null` and returns `"unauthorized"`.  
* No explicit check for user permissions or ownership. If products can be accessed by only certain users, consider adding a permission check.

### 5.6 Testing

* No unit tests are evident.  
* Suggest creating tests for:
  - Attribute grouping logic (specifications vs options).
  - Default option handling and price calculation.
  - Error handling when services throw exceptions.

### 5.7 Future Enhancements

1. **Dependency Injection** – Replace the ServiceFactory locator with constructor injection (Spring, Guice, etc.) to make the class more testable.  
2. **DTO Layer** – Create a view‑model DTO that contains only the data needed by the view, instead of exposing the full domain objects.  
3. **Internationalisation** – Move hard‑coded labels into resource bundles.  
4. **API Refactor** – Expose a dedicated `ProductPreviewService` that encapsulates all business logic, leaving the action as a thin controller.  
5. **Error Result** – Define a dedicated error result (`error`) and return it when an exception occurs, rather than silently displaying `SUCCESS`.  

--- 

**Overall Assessment**  
The class fulfills its functional goal but suffers from a handful of code‑quality issues, a subtle bug in attribute grouping, and opportunities for refactoring. Addressing these points will improve maintainability, testability, and correctness.

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
import java.util.Iterator;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.central.PageBaseAction;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescriptor;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ProductUtil;

public class ProductPreviewAction extends PageBaseAction {

	private static Logger log = Logger.getLogger(ProductPreviewAction.class);

	private Product product;
	private Collection<ProductOptionDescriptor> specifications = new ArrayList();// read
																					// only
																					// attributes
	private Collection<ProductOptionDescriptor> options = new ArrayList();// priced
																			// options

	private String productPrice;
	private Map storeConfiguration;

	public String display() {
		
		super.setPageTitle("label.product.preview");

		try {

			if (this.getProduct() == null) {
				return "unauthorized";
			}

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(super.getContext()
					.getMerchantid());

			// product details
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			product = cservice.getProduct(this.getProduct().getProductId());
			product.setLocale(super.getLocale());

			// options - attributes
			Collection attributes = cservice.getProductAttributes(product
					.getProductId(), super.getLocale().getLanguage());

			Collection defaultOptions = new ArrayList();

			if (attributes != null && attributes.size() > 0) {

				// extract read only
				Iterator i = attributes.iterator();

				long lastOptionId = -1;
				long lastSpecificationOptionId = -1;
				ProductOptionDescriptor pod = null;

				while (i.hasNext()) {

					ProductAttribute pa = (ProductAttribute) i.next();

					ProductOption po = pa.getProductOption();
					ProductOptionValue pov = pa.getProductOptionValue();
					if (po != null) {

						if (pa.isAttributeDisplayOnly()) {

							if (lastSpecificationOptionId == -1) {
								lastSpecificationOptionId = po
										.getProductOptionId();
								pod = new ProductOptionDescriptor();
								pod.setOptionType(po.getProductOptionType());
								pod.setName(po.getName());
								specifications.add(pod);
							} else {
								if (pa.getOptionId() != lastOptionId) {
									lastSpecificationOptionId = po
											.getProductOptionId();
									pod = new ProductOptionDescriptor();
									pod
											.setOptionType(po
													.getProductOptionType());
									pod.setName(po.getName());
									specifications.add(pod);
								}
							}

						} else {// option

							if (lastOptionId == -1) {
								lastOptionId = po.getProductOptionId();
								pod = new ProductOptionDescriptor();
								pod.setOptionType(po.getProductOptionType());
								pod.setName(po.getName());
								options.add(pod);
								if (pa.isAttributeDefault()) {
									defaultOptions.add(pa);
								}

							} else {
								if (pa.getOptionId() != lastOptionId) {
									lastOptionId = po.getProductOptionId();
									pod = new ProductOptionDescriptor();
									pod
											.setOptionType(po
													.getProductOptionType());
									pod.setName(po.getName());
									options.add(pod);
									if (pa.isAttributeDefault()) {
										defaultOptions.add(pa);
									}
								}
							}
						}

						pod.addValue(pa);
						pod.setOptionId(pa.getOptionId());
						if (pa.isAttributeDefault()) {
							pod.setDefaultOption(pa.getProductAttributeId());
						}
					}
				}

			}

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			storeConfiguration = rservice.getModuleConfigurationsKeyValue(store
					.getTemplateModule(), store.getCountry());

			if (defaultOptions != null && defaultOptions.size() > 0) {
				this.setProductPrice(ProductUtil
						.formatHTMLProductPriceWithAttributes(
								super.getLocale(), store.getCurrency(), this
										.getProduct(), defaultOptions, true));
			} else {
				this.setProductPrice(ProductUtil.formatHTMLProductPrice(super
						.getLocale(), store.getCurrency(), this.getProduct(),
						true, false));
			}

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

	public String getProductPrice() {
		return productPrice;
	}

	public void setProductPrice(String productPrice) {
		this.productPrice = productPrice;
	}

	public Map getStoreConfiguration() {
		return storeConfiguration;
	}

	public void setStoreConfiguration(Map storeConfiguration) {
		this.storeConfiguration = storeConfiguration;
	}

	public Collection<ProductOptionDescriptor> getSpecifications() {
		return specifications;
	}

	public void setSpecifications(
			Collection<ProductOptionDescriptor> specifications) {
		this.specifications = specifications;
	}

	public Collection<ProductOptionDescriptor> getOptions() {
		return options;
	}

	public void setOptions(Collection<ProductOptionDescriptor> options) {
		this.options = options;
	}

}



```
