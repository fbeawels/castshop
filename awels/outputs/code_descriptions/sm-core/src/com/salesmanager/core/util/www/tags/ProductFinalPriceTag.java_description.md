# ProductFinalPriceTag.java

## Review

## 1. Summary
**Purpose**  
`ProductFinalPriceTag` is a JSP custom tag that renders the final price of a `Product` object. It formats the price according to the current locale and the merchant store’s currency, optionally prepending the currency symbol.

**Key components**  
| Component | Role |
|-----------|------|
| `product` | The `Product` whose price will be displayed |
| `displayCurrency` | Boolean flag that controls whether the currency symbol is rendered |
| `doStartTag()` | Entry point of the tag – fetches request/session data, calculates the price, formats it, and writes it to the JSP output |
| `ProductUtil.determinePrice()` | Business‑logic helper that calculates the product’s price based on locale and currency |
| `CurrencyUtil.*` | Static formatting helpers that produce a string representation of the price |

**Notable patterns & frameworks**  
* TagSupport is the classic JSP tag‑library pattern (pre‑Tag‑Model API).  
* The tag relies on the servlet/JSP runtime (`HttpServletRequest`, `HttpSession`, `pageContext`) for context.  
* Uses standard Java EE APIs (Servlet, JSP) and no modern frameworks.

---

## 2. Detailed Description
### Execution Flow
1. **Tag initialization** – The tag handler is instantiated by the JSP container.  
2. **`doStartTag()`** – Called by the container when the start tag is encountered.  
   * Retrieves the current `HttpServletRequest` and `HttpSession`.  
   * Looks up the `Locale` (stored as request attribute `"LOCALE"`) and the `MerchantStore` (stored as session attribute `"STORE"`).  
   * Calls `ProductUtil.determinePrice()` to get the raw `BigDecimal` price.  
   * Formats the price twice (duplicate logic – a bug) using `CurrencyUtil` based on `displayCurrency`.  
   * Writes the formatted string to the page output.  
3. **Return value** – `SKIP_BODY` indicates that the tag has no body content to process.  

### Assumptions & Constraints
| Assumption | Impact |
|------------|--------|
| `request.getAttribute("LOCALE")` exists | Tag will throw `ClassCastException` or `NullPointerException` if missing. |
| `session.getAttribute("STORE")` exists | Same risk of `NullPointerException`. |
| `product` is set before tag execution | If null, `ProductUtil.determinePrice()` may throw an exception. |
| `Locale` and `MerchantStore` are not null | Critical for formatting. |
| No error handling – commented out `try/catch` | Runtime errors propagate to JSP and may expose stack traces. |

### Architecture & Design Choices
* **Separation of concerns** – Delegates price calculation and formatting to helper classes (`ProductUtil`, `CurrencyUtil`).  
* **Tag parameters** – Simple getter/setter style, enabling attributes in the JSP.  
* **Hard‑coded attribute names** – Tightly coupled to the application’s session/request attribute naming convention.  
* **No caching** – Each tag invocation recomputes the price and formatting.  

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `doStartTag()` | Core logic of the tag. Calculates, formats, and writes the price. | None | `int` (`SKIP_BODY`/`EVAL_BODY_INCLUDE`) | Writes to `pageContext.getOut()`. Reads from request/session. |
| `getProduct()` | Getter for the `product` field. | None | `Product` | None |
| `setProduct(Product product)` | Setter for the `product` field. | `Product product` | None | Sets internal state |
| `isDisplayCurrency()` | Getter for `displayCurrency`. | None | `boolean` | None |
| `setDisplayCurrency(boolean displayCurrency)` | Setter for `displayCurrency`. | `boolean displayCurrency` | None | Sets internal state |

*Reusable/Utility Methods*:  
`ProductUtil.determinePrice()`, `CurrencyUtil.displayFormatedAmountWithCurrency()`, and `CurrencyUtil.displayFormatedAmountNoCurrency()` are the only external helpers invoked; they encapsulate business logic and formatting.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard | Servlet API |
| `javax.servlet.http.HttpSession` | Standard | Servlet API |
| `javax.servlet.jsp.JspException` | Standard | JSP API |
| `javax.servlet.jsp.tagext.TagSupport` | Standard | JSP API |
| `java.math.BigDecimal` | Standard | Java SE |
| `java.util.Locale` | Standard | Java SE |
| `java.util.Map`, `HashMap`, `Set` | Standard | Java SE (unused in the current code) |
| `com.salesmanager.core.entity.catalog.Product` | Third‑party | Domain entity |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Third‑party | Domain entity |
| `com.salesmanager.core.util.CurrencyUtil` | Third‑party | Utility class |
| `com.salesmanager.core.util.ProductUtil` | Third‑party | Utility class |

No platform‑specific or exotic dependencies are present, but the code is tightly coupled to the *SalesManager* domain model and its session/request attribute conventions.

---

## 5. Additional Notes & Recommendations

### Duplicate Formatting Logic
The block

```java
pageContext.getOut().print(pprice);
if(this.displayCurrency) {
    pprice = CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency());
} else {
    pprice = CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());
}
pageContext.getOut().print(pprice);
```

writes the formatted price **twice**. This appears to be a copy‑paste error and will result in duplicate price output. The intended behavior is likely to print the formatted price **once**. Remove the first `print` or the redundant block.

### Missing Null‑Checks
```java
Locale locale = (Locale) request.getAttribute("LOCALE");
MerchantStore store = (MerchantStore)session.getAttribute("STORE");
```
Both values can be `null`. Add defensive checks and provide meaningful error messages or default fallbacks.

### Exception Handling
The commented‑out `try/catch` block swallows exceptions silently. In a production tag, you should either re‑throw a `JspException` or log the error and provide a fallback output. Example:

```java
try {
    // existing logic
} catch (Exception ex) {
    throw new JspException("Error rendering product price", ex);
}
```

### Thread‑Safety
`TagSupport` instances are reused by the container. The current code only uses instance fields for the tag attributes (`product`, `displayCurrency`) which are set per tag instance, so thread‑safety is maintained. However, avoid storing any request‑ or session‑specific data in instance fields.

### Logging
Add a logger (e.g., SLF4J) to capture runtime issues rather than silent failures.

### Decoupling
Hard‑coded attribute names (`"LOCALE"`, `"STORE"`) reduce portability. Consider injecting these via tag attributes or a context helper.

### Unused Imports
`HashMap`, `Set` are imported but never used. Clean them up.

### JavaDoc / Comments
Add Javadoc for the class and key methods to aid maintainability.

### Example Refactored `doStartTag()`
```java
@Override
public int doStartTag() throws JspException {
    try {
        HttpServletRequest request = (HttpServletRequest) pageContext.getRequest();
        HttpSession session = request.getSession(false);
        if (session == null) {
            throw new JspException("No HTTP session available");
        }

        Locale locale = (Locale) request.getAttribute("LOCALE");
        if (locale == null) {
            locale = request.getLocale(); // fallback
        }

        MerchantStore store = (MerchantStore) session.getAttribute("STORE");
        if (store == null) {
            throw new JspException("MerchantStore not found in session");
        }

        if (product == null) {
            throw new JspException("Product not set on tag");
        }

        BigDecimal price = ProductUtil.determinePrice(product, locale, store.getCurrency());

        String formattedPrice = displayCurrency
                ? CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency())
                : CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());

        pageContext.getOut().print(formattedPrice);
    } catch (Exception e) {
        throw new JspException("Error rendering product final price", e);
    }
    return SKIP_BODY;
}
```

### Future Enhancements
1. **Locale/Store Resolution Service** – Inject a service that abstracts retrieval of `Locale` and `MerchantStore` from request/session.  
2. **Unit Tests** – Implement tests for the tag logic using a mock JSP context.  
3. **Attribute Exposure** – Allow the tag to expose the computed `BigDecimal` price for downstream tags or scripting.  
4. **Internationalization** – Support more formatting options (e.g., currency symbols, locale‑specific decimal separators).  
5. **Switch to JSP‑Tag‑Model API** – Modernize the tag to use `SimpleTagSupport` or JSP‑Tag‑Model if the project allows.  

---

### Summary
`ProductFinalPriceTag` serves a clear purpose but contains a critical duplication bug, lacks defensive error handling, and relies on hard‑coded context attributes. By cleaning up the duplicate logic, adding null checks, proper exception handling, and logging, the tag will become more robust and maintainable. Further decoupling and modernizing the tag implementation will improve testability and future‑proof the component.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www.tags;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.tagext.TagSupport;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.ProductUtil;


public class ProductFinalPriceTag extends TagSupport {
	
	private Product product;
	private boolean displayCurrency;
	
	
	public int doStartTag() throws JspException {
		//try {



			HttpServletRequest request = (HttpServletRequest) pageContext
			.getRequest();
	
			HttpSession session = request.getSession();
			
			Locale locale = (Locale) request.getAttribute("LOCALE");
			
			MerchantStore store = (MerchantStore)session.getAttribute("STORE");

			BigDecimal price = ProductUtil.determinePrice(product, locale, store.getCurrency());

			
			String pprice = null;
			
			if(this.displayCurrency) {
				pprice = CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency());
			} else {
				pprice = CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());
			}
			

			pageContext.getOut().print(pprice);
	
	if(this.displayCurrency) {
		pprice = CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency());
	} else {
		pprice = CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());
	}
	
	
	
	pageContext.getOut().print(pprice);
			
		//} catch (Exception ex) {
			//log.error(ex);
		//}
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
