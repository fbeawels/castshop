# OrderProduct.java

## Review

## 1. Summary
**Purpose & Functionality**  
`OrderProduct` represents a single line item in an order. It stores product details (ID, model, name, price, tax, quantity), pricing calculations, product attributes, downloadable content, and various flags such as “free”, “virtual”, “subscription”, and “shipping”. The class also contains transient fields used for rendering in a shopping‑cart UI (e.g., formatted prices, image paths, error messages).

**Key Components**  
| Component | Role |
|-----------|------|
| **Fields** | Persisted attributes mapped to a database table (via Hibernate) and transient UI fields. |
| **Constructors** | Default constructor (initializes numeric fields) and a full‑argument constructor for persistence. |
| **Getters/Setters** | Standard bean accessors for all fields. |
| **Utility Methods** | `getPrice()`, `getUnitPriceNoCurrency()`, `getProductPriceFormated()` – format price strings. |
| **Equals/HashCode** | Overridden for value comparison based on business key fields. |
| **Transient helpers** | `getSmallImagePath()` (via `FileUtil`), `setLocale()` overloads. |

**Notable Design Patterns / Frameworks**  
* JavaBean / POJO pattern – used for Hibernate entity mapping.  
* `I18NEntity` interface indicates support for internationalization (though only locale is stored).  
* Utility classes `CurrencyUtil` and `FileUtil` are used for formatting and path generation.  

---

## 2. Detailed Description
### Data Model
The class maps directly to a relational table (`order_product`). The primary key is `orderProductId`. Hibernate annotations are absent (likely XML‑based mapping), but the naming convention and types align with Hibernate’s default conventions.

### Runtime Flow
1. **Creation**  
   * When an order is created, `OrderProduct` instances are populated via the full‑argument constructor or via setters (e.g., by a DAO).  
   * The default constructor sets `productTax` and `onetimeCharge` to zero.

2. **Price Calculation**  
   * `productPrice` holds the unit price (with attribute adjustments).  
   * `finalPrice` is the total for the line item (unit price × quantity).  
   * `onetimeCharge` is an optional extra fee.  
   * `productSpecialNewPrice` + dates allow for time‑bound specials.

3. **Attributes & Downloads**  
   * `orderattributes` holds a set of `OrderProductAttribute` objects.  
   * `downloads` holds a set of `OrderProductDownload` objects.

4. **Transient UI Fields**  
   * `priceFormated`, `priceText`, `costText`, etc. are filled by the presentation layer before rendering.  
   * `setLocale()` / `setLocale(Locale, String)` allow locale/currency context to be passed to formatting methods.

5. **Persistence**  
   * Standard getters/setters allow Hibernate to read/write fields.  
   * `equals`/`hashCode` enable usage in collections and cache keys.

6. **Cleanup**  
   * No explicit resource handling is required.  

### Assumptions & Constraints
* **Hibernate**: Relies on default field‑to‑column mapping; missing annotations may lead to mapping errors if schema changes.  
* **Locale/Currency**: The class assumes the caller supplies a correct currency code when formatting prices.  
* **Thread‑Safety**: The object is mutable; it is intended for single‑thread usage per order line.  
* **Decimal Precision**: Uses `BigDecimal` everywhere for monetary values; arithmetic is not shown but likely performed elsewhere.

### Architecture & Design Choices
* **Separation of Concerns**: The class mixes persistence fields with UI helpers (formatted strings, image paths). This is convenient but deviates from a strict domain‑model approach.  
* **Transient Flags**: Booleans like `attributes`, `recursive`, `shipping` are ambiguous; they are likely used by the view layer but their semantics are not fully documented.  
* **Internationalization**: Implements `I18NEntity`, but only holds a `Locale`; no language‑specific fields or methods.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `initialize()` | Sets default values for `productTax` and `onetimeCharge`. | none | none | modifies `productTax`, `onetimeCharge` |
| `getPrice()` | Returns formatted price (unit × qty) with currency if available. | none | `String` | none |
| `getUnitPriceNoCurrency()` | Returns unit price without currency formatting. | none | `String` | none |
| `getProductPriceFormated()` | Returns formatted unit price with currency. | none | `String` | none |
| `equals(Object)` | Checks equality based on business key fields. | `Object` | `boolean` | none |
| `hashCode()` | Generates hash based on business key fields. | none | `int` | none |
| `getSmallImagePath()` | Builds relative path to a small product image. | none | `String` | uses `FileUtil` |
| `setLocale(Locale)` / `setLocale(Locale, String)` | Store locale and optionally currency. | `Locale`, optional `String` | none |
| Getters/Setters | Standard bean accessors for all fields. | – | – | – |

**Reusable / Utility Methods**  
* `CurrencyUtil.displayFormatedAmountWithCurrency(...)` and `displayFormatedAmountNoCurrency(...)` – formatting helper.  
* `FileUtil.getSmallProductImagePath(...)` – image path generator.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | JDK | Monetary precision. |
| `java.util.Date` | JDK | For special price validity periods. |
| `java.util.Locale` | JDK | Locale handling. |
| `java.util.Set` | JDK | Collections for attributes/downloads/prices. |
| `com.salesmanager.core.util.CurrencyUtil` | Internal | Formats prices. |
| `com.salesmanager.core.util.FileUtil` | Internal | Builds image paths. |
| `com.salesmanager.core.entity.common.I18NEntity` | Internal | Marker interface for i18n support. |
| `com.salesmanager.core.entity.orders.OrderProductAttribute`, `OrderProductDownload`, `OrderProductPrice` | Internal | Related entities. |

All dependencies are standard Java or project‑specific utilities; no external libraries are referenced.

---

## 5. Additional Notes & Recommendations

### Strengths
* **Comprehensive data model** – covers product details, pricing, attributes, downloads, and special pricing.
* **Clear separation of persisted vs transient fields** (via comments).
* **Use of `BigDecimal`** prevents floating‑point precision errors in monetary calculations.

### Issues & Suggested Improvements
| Issue | Impact | Recommendation |
|-------|--------|----------------|
| **Mix of domain and view concerns** – formatted price strings, image paths, and UI flags clutter the entity. | Violates Single Responsibility Principle; makes unit testing harder. | Extract a separate DTO or view‑model class for presentation logic. |
| **Ambiguous field names** – `attributes`, `recursive`, `shipping`, `priceFormated` (typo). | Potential confusion for developers. | Rename to `hasAttributes`, `isRecursive`, `isShippingEnabled`, `formattedPrice`. |
| **Unimplemented `I18NEntity` methods** – only `setLocale` is present; `getLocale` is missing. | Incomplete interface implementation. | Add `getLocale()` and consider a `getCurrency()` method. |
| **Equals/HashCode** – includes many mutable fields.  If a field changes after insertion into a hash‑based collection, the contract is broken. | Bugs in collections (e.g., `Set<OrderProduct>`). | Use immutable key fields (e.g., `orderProductId`) for equality or enforce immutability. |
| **Null handling in formatting methods** – `getPrice()` and others return empty strings on null price; this may hide errors. | Silent failures. | Throw `IllegalStateException` or log a warning if critical fields are null. |
| **Lack of Javadoc** – methods lack documentation. | Harder maintenance. | Add Javadoc comments describing each method’s purpose. |
| **No validation** – setters do not validate input (e.g., negative price or quantity). | Data integrity issues. | Add validation logic or use Bean Validation annotations (`@NotNull`, `@Positive`). |
| **Date usage** – raw `Date` is used for special price windows; `java.time` API would be safer. | Time‑zone issues. | Migrate to `java.time.LocalDateTime` or `ZonedDateTime`. |
| **Thread‑safety** – mutable state could be problematic if shared across threads. | Concurrency bugs. | Document that the object is not thread‑safe or make it immutable. |
| **`getDescription()`** returns `null` – placeholder method. | Unused API. | Implement or remove. |

### Future Enhancements
1. **DTO/VO Layer** – create an `OrderProductView` or `OrderProductDTO` for UI rendering.  
2. **Builder Pattern** – simplify construction of instances with many fields.  
3. **Currency & Locale Centralization** – move formatting logic to a dedicated service.  
4. **Unit Tests** – add comprehensive tests for price calculations and equals/hashCode behavior.  
5. **Persistence Optimizations** – consider lazy fetching for collections (`downloads`, `prices`) to avoid N+1 problems.  

Overall, the class fulfills its role as a persistence entity but could benefit from a cleaner separation of concerns, improved validation, and modern Java practices.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.entity.orders;

// Generated Oct 1, 2008 9:31:43 AM by Hibernate Tools 3.2.0.beta8

import java.math.BigDecimal;
import java.util.Date;
import java.util.Locale;
import java.util.Set;

import com.salesmanager.core.entity.common.I18NEntity;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.FileUtil;

/**
 * OrdersProducts generated by hbm2java
 */
public class OrderProduct implements java.io.Serializable, I18NEntity {

	// Fields

	static final long serialVersionUID = 176131742783954627L;

	private long orderProductId;

	private long orderId;

	private long productId;

	private String productModel;

	private String productName;

	private BigDecimal productPrice;// unit price + [attributes price if any
									// selected]

	private BigDecimal finalPrice;// (unit price + attributes) * quantity

	private BigDecimal productTax;

	private int productQuantity;

	private BigDecimal onetimeCharge;//

	private boolean productIsFree;

	private BigDecimal productSpecialNewPrice;
	private Date productSpecialDateAvailable;
	private Date productSpecialDateExpire;

	// one to one
	// private com.salesmanager.core.entity.orders.OrderProductDownload
	// download;
	private Set<OrderProductDownload> downloads;// all product prices

	// collections
	private java.util.Set<com.salesmanager.core.entity.orders.OrderProductAttribute> orderattributes;

	private Set<OrderProductPrice> prices;// all product prices

	/**
	 * transiant attributes not saved in the database
	 */

	// product specific attributes for processing the order
	private long taxClassId;
	private int productType;
	private boolean productVirtual;
	private boolean productSubscribtion;
	private BigDecimal productWeight;
	private BigDecimal productLength;
	private BigDecimal productWidth;
	private BigDecimal productHeight;
	private int productQuantityOrderMax = 1;

	private BigDecimal originalProductPrice;
	private BigDecimal soldPrice;

	// Shopping cart transient lines
	private String currency;
	private String errorMessage;
	private String priceErrorMessage;
	private String quantityErrorMessage;
	private int lineId;
	private boolean attributes;
	private boolean shipping;
	private boolean recursive;
	private String productDescription;
	private String priceFormated;// final price formated
	private String productImage;
	private String priceText;// unit price + attributes displayed in the cart &
								// price submited from JSP
	private String costText;// price + attributes * quantity
	private String attributesLine;
	private String quantityText;
	private BigDecimal applicableCreditOneTimeCharge = new BigDecimal("0");
	private BigDecimal attributeAdditionalCost;
	private int merchantId;

	private Locale locale;

	// Constructors

	public BigDecimal getAttributeAdditionalCost() {
		return attributeAdditionalCost;
	}

	public void setAttributeAdditionalCost(BigDecimal attributeAdditionalCost) {
		this.attributeAdditionalCost = attributeAdditionalCost;
	}

	public OrderProduct(long orderProductId, long orderId, long productId,
			String productModel, String productName, BigDecimal productPrice,
			BigDecimal finalPrice, BigDecimal productTax, int productQuantity,
			BigDecimal onetimeCharge, boolean productIsFree) {
		super();
		this.orderProductId = orderProductId;
		this.orderId = orderId;
		this.productId = productId;
		this.productModel = productModel;
		this.productName = productName;
		this.productPrice = productPrice;
		this.finalPrice = finalPrice;
		this.productTax = productTax;
		this.productQuantity = productQuantity;
		this.onetimeCharge = onetimeCharge;
		this.productIsFree = productIsFree;

	}

	private void initialize() {
		this.productTax = new BigDecimal("0");
		this.onetimeCharge = new BigDecimal("0");
	}

	/** default constructor */
	public OrderProduct() {
		initialize();
	}

	public BigDecimal getFinalPrice() {
		return finalPrice;
	}

	public void setFinalPrice(BigDecimal finalPrice) {
		this.finalPrice = finalPrice;
	}

	public BigDecimal getOnetimeCharge() {
		return onetimeCharge;
	}

	public void setOnetimeCharge(BigDecimal onetimeCharge) {
		this.onetimeCharge = onetimeCharge;
	}

	public long getOrderId() {
		return orderId;
	}

	public void setOrderId(long orderId) {
		this.orderId = orderId;
	}

	public long getOrderProductId() {
		return orderProductId;
	}

	public void setOrderProductId(long orderProductId) {
		this.orderProductId = orderProductId;
	}

	public long getProductId() {
		return productId;
	}

	public void setProductId(long productId) {
		this.productId = productId;
	}

	public boolean isProductIsFree() {
		return productIsFree;
	}

	public void setProductIsFree(boolean productIsFree) {
		this.productIsFree = productIsFree;
	}

	public String getProductModel() {
		return productModel;
	}

	public void setProductModel(String productModel) {
		this.productModel = productModel;
	}

	public String getProductName() {
		return productName;
	}

	public void setProductName(String productName) {
		this.productName = productName;
	}

	public BigDecimal getProductPrice() {
		return productPrice;
	}

	public void setProductPrice(BigDecimal productPrice) {
		this.productPrice = productPrice;
	}

	public int getProductQuantity() {
		return productQuantity;
	}

	public void setProductQuantity(int productQuantity) {
		this.productQuantity = productQuantity;
	}

	public BigDecimal getProductTax() {
		return productTax;
	}

	public void setProductTax(BigDecimal productTax) {
		this.productTax = productTax;
	}

	public String getCurrency() {
		return currency;
	}

	public void setCurrency(String currency) {
		this.currency = currency;
	}

	public java.util.Set<com.salesmanager.core.entity.orders.OrderProductAttribute> getOrderattributes() {
		return orderattributes;
	}

	public void setOrderattributes(
			java.util.Set<com.salesmanager.core.entity.orders.OrderProductAttribute> orderattributes) {
		this.orderattributes = orderattributes;
	}

	/**
	 * Price Formated With Currency productPrice (price + attributes) * quantity
	 * 
	 * @return
	 */
	public String getPrice() {
		if (this.currency != null) {
			return CurrencyUtil.displayFormatedAmountWithCurrency(
					this.getProductPrice()
							.multiply(
									new java.math.BigDecimal(this
											.getProductQuantity())),
					this.currency);
		} else {
			if (this.getProductPrice() != null) {
				return this.getProductPrice().multiply(
						new java.math.BigDecimal(this.getProductQuantity()))
						.toString();
			} else {
				return "";
			}
		}
	}

	/**
	 * Product price No Currency productPrice (price + attributes)
	 * 
	 * @return
	 */
	public String getUnitPriceNoCurrency() {
		if (this.currency != null) {
			return CurrencyUtil.displayFormatedAmountNoCurrency(this
					.getProductPrice(), this.currency);
		} else {
			if (this.getProductPrice() != null) {
				return this.getProductPrice().toString();
			} else {
				return "";
			}
		}
	}

	@Override
	public int hashCode() {
		final int PRIME = 31;
		int result = 1;
		result = PRIME * result
				+ ((finalPrice == null) ? 0 : finalPrice.hashCode());
		result = PRIME * result
				+ ((onetimeCharge == null) ? 0 : onetimeCharge.hashCode());
		result = PRIME * result + (int) (orderId ^ (orderId >>> 32));
		result = PRIME * result
				+ (int) (orderProductId ^ (orderProductId >>> 32));
		result = PRIME * result + (int) (productId ^ (productId >>> 32));
		result = PRIME * result + (productIsFree ? 1231 : 1237);
		result = PRIME * result
				+ ((productModel == null) ? 0 : productModel.hashCode());
		result = PRIME * result
				+ ((productName == null) ? 0 : productName.hashCode());
		result = PRIME * result
				+ ((productPrice == null) ? 0 : productPrice.hashCode());
		result = PRIME * result + productQuantity;
		result = PRIME * result
				+ ((productTax == null) ? 0 : productTax.hashCode());
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		final OrderProduct other = (OrderProduct) obj;
		if (finalPrice == null) {
			if (other.finalPrice != null)
				return false;
		} else if (!finalPrice.equals(other.finalPrice))
			return false;
		if (onetimeCharge == null) {
			if (other.onetimeCharge != null)
				return false;
		} else if (!onetimeCharge.equals(other.onetimeCharge))
			return false;
		if (orderId != other.orderId)
			return false;
		if (orderProductId != other.orderProductId)
			return false;
		if (productId != other.productId)
			return false;
		if (productIsFree != other.productIsFree)
			return false;
		if (productModel == null) {
			if (other.productModel != null)
				return false;
		} else if (!productModel.equals(other.productModel))
			return false;
		if (productName == null) {
			if (other.productName != null)
				return false;
		} else if (!productName.equals(other.productName))
			return false;
		if (productPrice == null) {
			if (other.productPrice != null)
				return false;
		} else if (!productPrice.equals(other.productPrice))
			return false;
		if (productQuantity != other.productQuantity)
			return false;
		if (productTax == null) {
			if (other.productTax != null)
				return false;
		} else if (!productTax.equals(other.productTax))
			return false;
		return true;
	}

	public int getLineId() {
		return lineId;
	}

	public void setLineId(int lineId) {
		this.lineId = lineId;
	}

	public boolean isAttributes() {
		return attributes;
	}

	public void setAttributes(boolean attributes) {
		this.attributes = attributes;
	}

	public String getPriceFormated() {
		return priceFormated;
	}

	public void setPriceFormated(String priceFormated) {
		this.priceFormated = priceFormated;
	}

	public String getProductImage() {
		return productImage;
	}

	public void setProductImage(String productImage) {
		this.productImage = productImage;
	}

	public String getProductDescription() {
		return productDescription;
	}

	public void setProductDescription(String productDescription) {
		this.productDescription = productDescription;
	}

	public String getPriceText() {
		return priceText;
	}

	public void setPriceText(String priceText) {
		this.priceText = priceText;
	}

	public boolean isRecursive() {
		return recursive;
	}

	public void setRecursive(boolean recursive) {
		this.recursive = recursive;
	}

	public String getAttributesLine() {
		return attributesLine;
	}

	public void setAttributesLine(String attributesLine) {
		this.attributesLine = attributesLine;
	}

	public Set<OrderProductPrice> getPrices() {
		return prices;
	}

	public void setPrices(Set<OrderProductPrice> prices) {
		this.prices = prices;
	}

	public String getQuantityText() {
		return quantityText;
	}

	public void setQuantityText(String quantityText) {
		this.quantityText = quantityText;
	}

	public String getPriceErrorMessage() {
		return priceErrorMessage;
	}

	public void setPriceErrorMessage(String priceErrorMessage) {
		this.priceErrorMessage = priceErrorMessage;
	}

	public String getQuantityErrorMessage() {
		return quantityErrorMessage;
	}

	public void setQuantityErrorMessage(String quantityErrorMessage) {
		this.quantityErrorMessage = quantityErrorMessage;
	}

	public String getErrorMessage() {
		return errorMessage;
	}

	public void setErrorMessage(String errorMessage) {
		this.errorMessage = errorMessage;
	}

	public long getTaxClassId() {
		return taxClassId;
	}

	public void setTaxClassId(long taxClassId) {
		this.taxClassId = taxClassId;
	}

	public String getCostText() {
		return costText;
	}

	public void setCostText(String costText) {
		this.costText = costText;
	}

	public int getProductType() {
		return productType;
	}

	public void setProductType(int productType) {
		this.productType = productType;
	}

	public boolean isProductVirtual() {
		return productVirtual;
	}

	public void setProductVirtual(boolean productVirtual) {
		this.productVirtual = productVirtual;
	}

	// public void setRelatedPrices(List<ProductPrice> relatedPrices) {
	// this.relatedPrices = relatedPrices;
	// }

	public boolean isShipping() {
		return shipping;
	}

	public void setShipping(boolean shipping) {
		this.shipping = shipping;
	}

	public BigDecimal getProductHeight() {
		return productHeight;
	}

	public void setProductHeight(BigDecimal productHeight) {
		this.productHeight = productHeight;
	}

	public BigDecimal getProductLength() {
		return productLength;
	}

	public void setProductLength(BigDecimal productLength) {
		this.productLength = productLength;
	}

	public BigDecimal getProductWeight() {
		return productWeight;
	}

	public void setProductWeight(BigDecimal productWeight) {
		this.productWeight = productWeight;
	}

	public BigDecimal getProductWidth() {
		return productWidth;
	}

	public void setProductWidth(BigDecimal productWidth) {
		this.productWidth = productWidth;
	}

	public boolean isProductSubscribtion() {
		return productSubscribtion;
	}

	public void setProductSubscribtion(boolean productSubscribtion) {
		this.productSubscribtion = productSubscribtion;
	}

	public Set<OrderProductDownload> getDownloads() {
		return downloads;
	}

	public void setDownloads(Set<OrderProductDownload> downloads) {
		this.downloads = downloads;
	}

	public Date getProductSpecialDateAvailable() {
		return productSpecialDateAvailable;
	}

	public void setProductSpecialDateAvailable(Date productSpecialDateAvailable) {
		this.productSpecialDateAvailable = productSpecialDateAvailable;
	}

	public Date getProductSpecialDateExpire() {
		return productSpecialDateExpire;
	}

	public void setProductSpecialDateExpire(Date productSpecialDateExpire) {
		this.productSpecialDateExpire = productSpecialDateExpire;
	}

	public BigDecimal getProductSpecialNewPrice() {
		return productSpecialNewPrice;
	}

	public void setProductSpecialNewPrice(BigDecimal productSpecialNewPrice) {
		this.productSpecialNewPrice = productSpecialNewPrice;
	}

	public BigDecimal getApplicableCreditOneTimeCharge() {
		return applicableCreditOneTimeCharge;
	}

	public void setApplicableCreditOneTimeCharge(
			BigDecimal applicableCreditOneTimeCharge) {
		this.applicableCreditOneTimeCharge = applicableCreditOneTimeCharge;
	}

	public BigDecimal getOriginalProductPrice() {
		return originalProductPrice;
	}

	public void setOriginalProductPrice(BigDecimal originalProductPrice) {
		this.originalProductPrice = originalProductPrice;
	}

	public BigDecimal getSoldPrice() {
		return soldPrice;
	}

	public void setSoldPrice(BigDecimal soldPrice) {
		this.soldPrice = soldPrice;
	}

	public String getProductPriceFormated() {
		if (this.getCurrency() != null) {
			return CurrencyUtil.displayFormatedAmountWithCurrency(this
					.getProductPrice(), this.getCurrency());
		} else {
			if(this.getProductPrice()!=null) {
				return this.getProductPrice().toString();
			} else {
				return "";
			}
		}
	}

	public int getProductQuantityOrderMax() {
		return productQuantityOrderMax;
	}

	public void setProductQuantityOrderMax(int productQuantityOrderMax) {
		this.productQuantityOrderMax = productQuantityOrderMax;
	}

	public String getDescription() {
		// TODO Auto-generated method stub
		return null;
	}

	public void setLocale(Locale locale) {
		// TODO Auto-generated method stub
		this.locale = locale;
	}

	public void setLocale(Locale locale, String currency) {
		// TODO Auto-generated method stub
		this.locale = locale;
		this.currency = currency;
	}

	public String getSmallImagePath() {
		return FileUtil.getSmallProductImagePath(this.getMerchantId(), this
				.getProductImage());
	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

}



```
