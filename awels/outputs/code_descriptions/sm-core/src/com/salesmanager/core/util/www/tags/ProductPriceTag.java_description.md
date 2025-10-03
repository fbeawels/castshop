# ProductPriceTag.java

## Review

## 1. Summary  

The `ProductPriceTag` is a JSP custom tag that renders the price of a `Product` in a web storefront.  
- **Purpose** – Given a `Product` instance, the tag formats the product’s price (optionally with a currency symbol) and writes the result to the JSP output stream.  
- **Key components**  
  - **Fields**: `product`, `displayCurrency`, `log`.  
  - **`doStartTag()`**: Core rendering logic – obtains locale and store information, calculates the price via `ProductUtil`, formats it with `CurrencyUtil`, and writes it out.  
  - **Getters/Setters** for the two configurable properties.  
- **Design patterns / libraries** – Implements the classic *Tag Support* pattern (extends `TagSupport`) and relies on Log4j for logging. No external frameworks beyond the standard JSP/Servlet stack and a handful of domain utilities.

## 2. Detailed Description  

### Core flow  
1. **Initialization** – When the tag is invoked in a JSP, `doStartTag()` is called.  
2. **Context retrieval** –  
   - `HttpServletRequest` and `HttpSession` are fetched from `pageContext`.  
   - `Locale` is extracted from the request attribute `"LOCALE"` (not the request’s locale).  
   - `MerchantStore` is taken from the session attribute `"STORE"`.  
3. **Price calculation** – `ProductUtil.determinePriceNoDiscount()` is invoked with the product, locale, and store currency.  
4. **Formatting** –  
   - If `displayCurrency` is `true`, `CurrencyUtil.displayFormatedAmountWithCurrency()` is used.  
   - Otherwise, `CurrencyUtil.displayFormatedAmountNoCurrency()` is called.  
5. **Output** – The formatted string is printed to the JSP output.  
6. **Return value** – The tag returns `SKIP_BODY`, so any body content inside the tag is ignored.

### Assumptions & constraints  
- The request must contain `"LOCALE"` and the session must contain `"STORE"` attributes; otherwise a `NullPointerException` will be thrown.  
- The `product` property is mandatory; a null product also leads to a runtime exception.  
- The tag does not provide any body processing or attributes beyond the two public properties.  
- All calculations assume the existence of the product’s attributes and that `ProductUtil` can handle them.

### Architecture / design choices  
- **TagSupport vs SimpleTagSupport** – The implementation uses the older `TagSupport` which forces the tag to be stateful and to handle `doStartTag()`/`doEndTag()` explicitly. For a read‑only tag, `SimpleTagSupport` would reduce boilerplate.  
- **Logging** – Log4j is used for error reporting; however, the logger is instantiated per instance rather than as a static final field.  
- **Exception handling** – A broad `catch (Exception)` swallows all failures, logs them, and still returns `SKIP_BODY`. In a JSP page, a silent failure may be confusing to the developer or user.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `int doStartTag()` | Primary rendering logic. Calculates, formats, and writes the product price. | None (uses instance fields and request/session context). | `SKIP_BODY` (int constant). | Writes to `pageContext.getOut()`, logs errors. |
| `Product getProduct()` | Getter for the product to be rendered. | None | `Product` | None |
| `void setProduct(Product product)` | Setter for the product. | `Product` | None | Sets instance field. |
| `boolean isDisplayCurrency()` | Getter for the flag controlling currency symbol display. | None | `boolean` | None |
| `void setDisplayCurrency(boolean displayCurrency)` | Setter for the currency flag. | `boolean` | None | Sets instance field. |

*Utility methods* – The tag delegates price calculation to `ProductUtil.determinePriceNoDiscount()` and formatting to `CurrencyUtil`. Those are reusable across the application but are not part of the tag itself.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest`, `HttpSession` | Servlet API | Standard |
| `javax.servlet.jsp.JspException`, `javax.servlet.jsp.tagext.TagSupport` | JSP API | Standard |
| `org.apache.log4j.Logger` | Logging | Third‑party (Log4j 1.x) |
| `com.salesmanager.core.entity.catalog.Product` | Domain entity | Internal |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Domain entity | Internal |
| `com.salesmanager.core.util.CurrencyUtil` | Utility | Internal |
| `com.salesmanager.core.util.ProductUtil` | Utility | Internal |
| `java.math.BigDecimal`, `java.util.Locale`, `java.util.Set` | Java SE | Standard |

No native platform dependencies beyond the servlet/JSP stack. The code is fully portable across any Java EE container that supports JSP 2.x.

## 5. Additional Notes  

### Edge Cases / Potential Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Null product / store / locale** | `NullPointerException` will be thrown and swallowed by the catch block, resulting in blank output. | Validate inputs; throw `JspException` if required attributes are missing. |
| **Unnecessary `attributes` set** | `product.getAttributes()` is called but never used. | Remove or incorporate logic that utilizes attributes if needed. |
| **Generics warning** | `Set attributes = product.getAttributes();` uses raw type. | Use generics: `Set<Attribute> attributes = product.getAttributes();` or remove entirely. |
| **Logging** | Logger is instantiated per tag instance; not efficient. | Declare `private static final Logger log = Logger.getLogger(ProductPriceTag.class);`. |
| **Exception handling** | Swallowing all exceptions hides problems during development. | Catch specific exceptions or rethrow as `JspException`. |
| **Tag lifecycle** | `TagSupport` implies stateful instances; if reused across requests, stale state could leak. | Use `SimpleTagSupport` or reset state in `release()`. |
| **Locale retrieval** | Depends on a request attribute `"LOCALE"`; may not be set. | Default to `request.getLocale()` or pass locale as an attribute. |
| **Currency formatting** | If `CurrencyUtil` returns null, `pageContext.getOut().print(pprice)` will throw. | Ensure non‑null formatting or fallback to a default string. |

### Potential Enhancements  

1. **Use `SimpleTagSupport`** – simplifies lifecycle, eliminates the need for `doStartTag()`/`doEndTag()` and allows body content if needed.  
2. **Parameter Validation** – validate `product`, `locale`, and `store` before use; throw a meaningful `JspException` to aid debugging.  
3. **Internationalization** – allow the tag to accept a locale as an attribute instead of hard‑coding request lookup.  
4. **Body Support** – expose the calculated price as an attribute in `pageContext` for further manipulation by the JSP.  
5. **Performance** – cache formatted strings for repeated rendering of the same product/currency pair during a request.  
6. **Unit Tests** – write tests for `doStartTag()` by mocking `PageContext`, `HttpServletRequest`, and the utility classes to verify correct formatting and error handling.  

Overall, the tag achieves its core goal of displaying a product price, but would benefit from stricter input validation, cleaner exception handling, and adherence to modern JSP tag practices.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www.tags;

import java.math.BigDecimal;
import java.util.Locale;
import java.util.Set;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.tagext.TagSupport;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.ProductUtil;

public class ProductPriceTag extends TagSupport {
	private Product product;
	private boolean displayCurrency;
	
	private Logger log = Logger.getLogger(ProductPriceTag.class);
	
	
	public int doStartTag() throws JspException {
		try {



			HttpServletRequest request = (HttpServletRequest) pageContext
					.getRequest();
			
			HttpSession session = request.getSession();
			
			Locale locale = (Locale) request.getAttribute("LOCALE");
			
			MerchantStore store = (MerchantStore)session.getAttribute("STORE");
			
			Set attributes = product.getAttributes();
							
			BigDecimal price = ProductUtil.determinePriceNoDiscount(product, locale, store.getCurrency());
			
			String pprice = null;
			
			if(this.displayCurrency) {
				pprice = CurrencyUtil.displayFormatedAmountWithCurrency(price, store.getCurrency());
			} else {
				pprice = CurrencyUtil.displayFormatedAmountNoCurrency(price, store.getCurrency());
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
