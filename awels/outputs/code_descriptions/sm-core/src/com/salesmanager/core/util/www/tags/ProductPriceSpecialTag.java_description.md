# ProductPriceSpecialTag.java

## Review

## 1. Summary

`ProductPriceSpecialTag` is a **JSP custom tag** that displays the *special* (discounted) price of a product.  
- It pulls the current `Locale` and `MerchantStore` from the request/session.  
- Using `ProductUtil`, it checks whether the product has a discount and calculates the discounted price.  
- `CurrencyUtil` formats that price either with or without the currency symbol based on the `displayCurrency` flag.  
- The formatted price is written to the page output.

Key components:
- **TagSupport** – base class for tag handling.
- **ProductUtil / CurrencyUtil** – domain utilities for price determination and formatting.
- **Log4j** – basic logging of unexpected exceptions.

The tag is straightforward, but a few design and robustness issues are worth noting.

---

## 2. Detailed Description

### Core Flow

| Step | What Happens | Why |
|------|--------------|-----|
| **Initialization** | Tag attributes (`product`, `displayCurrency`) are set by the JSP engine before `doStartTag()` is called. | These attributes drive the tag’s logic. |
| **Locale & Store Retrieval** | `Locale` fetched from request attribute `"LOCALE"`; `MerchantStore` fetched from session attribute `"STORE"`. | Needed to calculate the correct localized price. |
| **Discount Check** | `ProductUtil.hasDiscount(product)` determines if a special price should be shown. | Avoids formatting a normal price when no discount is present. |
| **Price Calculation** | `ProductUtil.determinePrice(product, locale, store.getCurrency())` returns a `BigDecimal` representing the discounted price. | Centralizes business logic for pricing. |
| **Formatting** | Depending on `displayCurrency`, `CurrencyUtil` formats the price with or without the currency symbol. | Provides a clean UI representation. |
| **Output** | The formatted price string is printed to the JSP output (`pageContext.getOut()`). | Renders the value into the page. |
| **Error Handling** | Any exception is logged (`log.error(ex)`), but the tag silently returns `SKIP_BODY`. | Avoids breaking the page but may hide underlying issues. |

### Design Choices & Assumptions

- **Thread‑Safety**: Tag instances are reused by the JSP container; stateful attributes (`product`, `displayCurrency`) should be cleared in `release()`. The current implementation does not reset them, risking stale data across requests.
- **Error Handling**: Swallowing all exceptions makes debugging harder. Ideally, re‑throw a `JspException` or at least surface a clear error message in the output.
- **Locale Source**: The tag reads the locale from a request attribute `"LOCALE"`. Many applications use `request.getLocale()` or a dedicated locale resolver; the hard‑coded attribute may break if the application changes locale handling.
- **Null Checks**: No validation for `product`, `locale`, or `store`. A `NullPointerException` could easily arise if any attribute is missing.
- **Tag Body**: The tag always returns `SKIP_BODY`. This is correct since the tag does not process a body, but implementing `doEndTag()` (or leaving it to `TagSupport`) clarifies intent.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `doStartTag()` | Entry point for the tag execution. Handles all logic described above. | None | `int` (`SKIP_BODY`) | Writes to `pageContext.getOut()`, logs exceptions. |
| `getProduct()` | Getter for the `product` attribute. | None | `Product` | None. |
| `setProduct(Product product)` | Setter invoked by the JSP engine to supply the product. | `Product` | None. |
| `isDisplayCurrency()` | Determines whether to show the currency symbol. | None | `boolean` | None. |
| `setDisplayCurrency(boolean displayCurrency)` | Setter for the currency display flag. | `boolean` | None. |
| `release()` | (Not overridden) The superclass resets tag state. | None | `void` | None. |

### Reusable/Utility Methods

The tag relies heavily on two utility classes:
- **`ProductUtil`**: Contains business logic for discount detection and price calculation.
- **`CurrencyUtil`**: Formats monetary values for display.

Both are designed to be reused across the application wherever pricing logic is required.

---

## 4. Dependencies

| Library/Framework | Purpose | Standard / Third‑Party |
|-------------------|---------|------------------------|
| `javax.servlet.jsp.tagext.TagSupport` | Base class for custom JSP tags. | Java EE / Jakarta EE |
| `javax.servlet.http.HttpServletRequest` / `HttpSession` | Access request/session data. | Java EE / Jakarta EE |
| `org.apache.log4j.Logger` | Logging of errors. | Third‑Party (Log4j) |
| `java.math.BigDecimal`, `java.util.Locale` | Core Java types for price and locale. | Standard |
| `com.salesmanager.core.entity.catalog.Product` | Domain entity for product data. | Application‑specific |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Domain entity for store/currency context. | Application‑specific |
| `com.salesmanager.core.util.CurrencyUtil` | Formatting currency values. | Application‑specific |
| `com.salesmanager.core.util.ProductUtil` | Price calculation and discount logic. | Application‑specific |

No external web frameworks (e.g., Spring MVC) are referenced directly; the tag relies purely on the servlet/JSP environment and the internal utilities.

---

## 5. Additional Notes

### Edge Cases & Missing Checks
- **Missing `product`**: If the tag is used without setting a product, `ProductUtil.hasDiscount(this.getProduct())` will throw a `NullPointerException`. A guard clause (`if (product == null) return SKIP_BODY;`) would prevent this.
- **Missing `locale` or `store`**: If the attributes `"LOCALE"` or `"STORE"` are absent, the code will crash. Defaulting to `request.getLocale()` and handling a missing store gracefully would improve resilience.
- **Null Price**: `ProductUtil.determinePrice()` might return `null`. The tag should handle this scenario gracefully.
- **Concurrent Use**: Tag instances may be reused across requests; not resetting attributes can cause cross‑request contamination. Overriding `release()` to clear `product` and `displayCurrency` is recommended.

### Potential Enhancements
1. **Better Error Reporting**: Instead of silently logging, write an informative error message to the JSP or re‑throw a `JspException`.
2. **Locale Retrieval**: Replace the hard‑coded request attribute with `request.getLocale()` or a configurable locale resolver.
3. **Internationalization**: Add a message key for “no discount” or “special price” instead of an empty string.
4. **Unit Tests**: Write tests for `doStartTag()` covering cases: product with/without discount, displayCurrency true/false, missing store/locale, null product.
5. **Code Separation**: Extract formatting logic into a separate helper method (`formatPrice(BigDecimal price)`) for readability.
6. **Release Override**: Reset state to avoid stale data.
7. **Thread‑Safe Logging**: Use a static `Logger` to avoid per‑instance logger creation.

Overall, the tag performs its intended function but would benefit from added robustness, clearer error handling, and adherence to JSP tag best practices.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www.tags;

import java.math.BigDecimal;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.tagext.TagSupport;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.ProductUtil;

public class ProductPriceSpecialTag extends TagSupport {
	private Product product;
	private boolean displayCurrency;
	
	private Logger log = Logger.getLogger(ProductPriceSpecialTag.class);
	
	
	public int doStartTag() throws JspException {
		try {



			HttpServletRequest request = (HttpServletRequest) pageContext
					.getRequest();
			
			HttpSession session = request.getSession();
			
			Locale locale = (Locale) request.getAttribute("LOCALE");
			MerchantStore store = (MerchantStore)session.getAttribute("STORE");
			
			String pprice = "";

			if(ProductUtil.hasDiscount(this.getProduct())) {
				
				BigDecimal price = ProductUtil.determinePrice(product, locale, store.getCurrency());
				if(this.displayCurrency) {
					pprice = CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency());
				} else {
					pprice = CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());
				}
			}

			pageContext.getOut().print(pprice);


			
		} catch (Exception ex) {
			log.error(ex);
		}
		return SKIP_BODY;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public boolean isDisplayCurrency() {
		return displayCurrency;
	}

	public void setDisplayCurrency(boolean displayCurrency) {
		this.displayCurrency = displayCurrency;
	}
}



```
