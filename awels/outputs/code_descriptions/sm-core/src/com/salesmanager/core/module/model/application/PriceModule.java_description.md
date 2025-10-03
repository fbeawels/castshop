# PriceModule.java

## Review

## 1. Summary  

**Purpose**  
`PriceModule` is a contract that defines the public API for any component that is responsible for calculating prices and formatting them for display in the SalesManager core.  It abstracts away the details of currency conversion, tax application, and order‑level total aggregation so that various implementations (e.g. in‑store, e‑commerce, or third‑party integrations) can be swapped without affecting the rest of the system.

**Key Components**  

| Interface | Role |
|-----------|------|
| `getPrice` | Returns the numeric price of a single product in a specific currency. |
| `getPriceText`, `getPricePrefixText`, `getPriceSuffixText` | Provide locale‑aware textual representations (e.g. “$”, “EUR”, “USD”) that can be used by UI layers. |
| `calculateOrderPrice` (two overloads) | Aggregates the price of a product within an order, applies tax and discounts, and updates an `OrderTotalSummary`. |
| `isTaxApplicable` | Indicates whether the pricing logic should apply tax. |
| `getHtmlPriceFormated` | Returns a fully formatted HTML snippet that can be rendered directly in web views. |

**Design Patterns & Libraries**  
- **Strategy Pattern** – Different concrete implementations can be plugged in (e.g., a simple price module vs. a multi‑currency module).  
- **Factory Pattern** – Often used elsewhere in the SalesManager core to create a `PriceModule` based on configuration.  
- **Java Standard Library** – `java.math.BigDecimal` for precise monetary calculations, `java.util.Locale` for internationalisation, and `String` for formatting.  
- **Domain Entities** – The interface tightly couples to domain objects (`ProductPrice`, `Order`, etc.) defined in `com.salesmanager.core.entity`.  

---

## 2. Detailed Description  

### Core Flow  

1. **Product Pricing** –  
   *A client calls `getPrice(productPrice, currency)` to obtain the raw price in the requested currency.*  

2. **Textual Formatting** –  
   *The UI layer may then ask for prefix/suffix/text to build a user‑friendly string. These methods rely on the locale to insert the correct currency symbol or formatting.*  

3. **Order Total Calculation** –  
   *When an order is being processed, `calculateOrderPrice(...)` is invoked. The method takes the current order, the running `OrderTotalSummary`, a specific `OrderProduct` (which contains quantity, options, etc.), and the corresponding `OrderProductPrice` (price of a single unit). The implementation must:*
   - Convert or adjust the unit price to the order’s currency.  
   - Apply tax if `isTaxApplicable()` returns true.  
   - Multiply by quantity and add to the `OrderTotalSummary`.  
   - Possibly apply product‑level discounts or taxes (e.g., VAT, excise).  

4. **HTML Formatting** –  
   *For web front‑ends, `getHtmlPriceFormated` returns an HTML snippet that already includes the prefix and suffix, ready for rendering.*  

### Assumptions & Constraints  

| Item | Explanation |
|------|-------------|
| **Currency** | Assumes a simple 3‑letter ISO code (e.g., “USD”, “EUR”). No mention of exchange rates – the implementation must handle this. |
| **Locale** | Used only for formatting; the underlying numeric value is always `BigDecimal`. |
| **Tax** | The interface offers a boolean flag but no tax rate. The concrete implementation is expected to fetch tax rates from configuration or external services. |
| **Null Safety** | The interface itself does not document null‑safety. Implementations should either throw `NullPointerException` or return defaults. |
| **Thread‑Safety** | No contract about state; implementations may be stateless or thread‑safe. |
| **Performance** | Repeated calls to formatting methods may be expensive; caching strategies are advisable. |

### Architectural Choices  

- **Separation of Concerns** – The interface cleanly separates *price retrieval* from *order aggregation* and *display formatting*.  
- **Domain‑Driven Design** – It relies on domain entities rather than primitive data types, encouraging rich domain models.  
- **Extensibility** – The overload of `calculateOrderPrice` that accepts a `Locale` allows future implementations to adjust tax or discounts based on geographic information.  

---

## 3. Functions/Methods  

| Method | Parameters | Returns | Side‑Effects | Notes |
|--------|------------|---------|--------------|-------|
| `BigDecimal getPrice(ProductPrice productPrice, String currency)` | *productPrice* – price object<br>*currency* – ISO code | Price in the requested currency | None | Should perform currency conversion if needed. |
| `String getPriceText(String currency, Locale locale)` | *currency* – ISO code<br>*locale* – formatting locale | Currency symbol/text suitable for UI | None | Example: “$” for USD in US locale. |
| `String getPricePrefixText(String currency, Locale locale)` | Same as above | Text placed before the numeric amount | None | Often the currency symbol. |
| `String getPriceSuffixText(String currency, Locale locale)` | Same as above | Text placed after the numeric amount | None | Useful for currencies that append the symbol (e.g., “50 €”). |
| `OrderTotalSummary calculateOrderPrice(Order order, OrderTotalSummary orderSummary, OrderProduct product, OrderProductPrice productPrice, String currency)` | Order context, running totals, product details, unit price, currency | Updated `OrderTotalSummary` | Modifies `orderSummary` (typically adds line totals) | No locale support. |
| `OrderTotalSummary calculateOrderPrice(Order order, OrderTotalSummary orderSummary, OrderProduct product, OrderProductPrice productPrice, String currency, Locale locale)` | Same as above + locale | Updated `OrderTotalSummary` | Same as above | Allows locale‑specific tax or discount rules. |
| `boolean isTaxApplicable()` | None | true/false | None | Determines whether the implementation applies tax. |
| `String getHtmlPriceFormated(String prefix, ProductPrice productPrice, Locale locale, String currency)` | *prefix* – optional text before amount<br>*productPrice* – price object<br>*locale* – formatting locale<br>*currency* – ISO code | Fully formatted HTML string | None | Could embed CSS classes or inline styles. |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | Standard Java | Precise monetary calculations. |
| `java.util.Locale` | Standard Java | Internationalisation support. |
| Domain entities (`ProductPrice`, `Order`, `OrderProduct`, `OrderProductPrice`, `OrderTotalSummary`) | Project‑specific | Defined in `com.salesmanager.core.entity`. |
| `com.salesmanager.core.entity.catalog.ProductPrice` | Project‑specific | Holds price per product and currency. |
| `com.salesmanager.core.entity.orders.*` | Project‑specific | Order model objects. |

No external third‑party libraries are referenced directly in the interface. However, concrete implementations will almost certainly use:

- `java.text.NumberFormat` for locale‑aware formatting.  
- A currency exchange service (REST/WS).  
- A tax engine or configuration service.  

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  

| Scenario | Possible Problem | Mitigation |
|----------|------------------|------------|
| **Null parameters** | NullPointerException if implementation does not guard. | Add defensive checks or document that nulls are prohibited. |
| **Unsupported currency** | Currency conversion may fail or return `null`. | Validate against a whitelist or throw a custom exception. |
| **Large quantities** | `BigDecimal` multiplication may lead to very large numbers; rounding concerns. | Use `MathContext` or scale appropriately. |
| **Locale mismatch** | Currency symbol may not match locale (e.g., “$” used in a non‑USD locale). | Use `Currency.getInstance(currency).getSymbol(locale)` for consistency. |
| **Thread‑safety** | If an implementation caches formatting objects (e.g., `NumberFormat`), it must be thread‑safe. | Use `ThreadLocal` or immutable formatters. |
| **Tax changes** | Tax rules may vary by date or jurisdiction. | Provide a `Locale` or `Date` parameter in the calculation methods. |

### Future Enhancements  

1. **Default Methods** – Introduce Java 8 default implementations for common formatting logic (e.g., prefix/suffix using `NumberFormat`).  
2. **Extensibility Hooks** – Add hooks for discount engines, promotion rules, or loyalty points that modify the `OrderTotalSummary`.  
3. **Currency Conversion Service** – Inject a dedicated service (e.g., `CurrencyExchangeService`) to keep the interface clean.  
4. **Internationalised Formatting** – Return `FormattedPrice` objects containing amount, currency, and locale instead of raw strings.  
5. **Immutable Summary** – Return a new `OrderTotalSummary` instead of mutating the existing one to avoid side‑effects.  
6. **Logging & Auditing** – Provide optional callbacks to record price calculations for compliance.  

### Design Recommendations  

- **Keep the interface minimal but expressive** – The current set of methods is adequate but consider merging the two `calculateOrderPrice` overloads if locale can be inferred from the order.  
- **Document contracts clearly** – Include Javadoc explaining expectations around nulls, currency support, and thread‑safety.  
- **Separate formatting from calculation** – While the interface mixes formatting and calculation, future versions might split into `PriceCalculator` and `PriceFormatter` interfaces for cleaner separation of concerns.  

Overall, `PriceModule` defines a solid abstraction that captures the core responsibilities of pricing within the SalesManager system. The next step is to implement concrete classes that adhere to these contracts while addressing the edge cases and enhancement ideas outlined above.

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
package com.salesmanager.core.module.model.application;

import java.math.BigDecimal;
import java.util.Locale;

import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductPrice;
import com.salesmanager.core.entity.orders.OrderTotalSummary;

public interface PriceModule {

	public BigDecimal getPrice(ProductPrice productPrice, String currency);

	public String getPriceText(String currency, Locale locale);

	public String getPricePrefixText(String currency, Locale locale);

	public String getPriceSuffixText(String currency, Locale locale);

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct product,
			OrderProductPrice productPrice, String currency);

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct product,
			OrderProductPrice productPrice, String currency, Locale locale);

	public boolean isTaxApplicable();

	public String getHtmlPriceFormated(String prefix,
			ProductPrice productPrice, Locale locale, String currency);

}



```
