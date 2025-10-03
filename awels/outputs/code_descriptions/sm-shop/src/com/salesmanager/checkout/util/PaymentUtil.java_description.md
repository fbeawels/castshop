# PaymentUtil.java

## Review

## 1. Summary
`com.salesmanager.checkout.util.PaymentUtil` is a thin façade that adapts the payment‑method data returned by the core library (`com.salesmanager.core.util.PaymentUtil`) for use by the checkout module.  
Its single responsibility is to fetch a map of `PaymentMethod` objects for a given merchant and locale, then replace the key for any credit‑card method (`type == 1`) with the literal `"GATEWAY"`.  

Key components  
- **`getPaymentMethods`** – static utility that delegates to the core payment util, rewrites keys, and returns the transformed map.  
- **`Logger`** – declared but never used.  

The class relies on the **Sales Manager core** package and Apache Log4j (unused). No design patterns beyond the façade/adapter pattern are evident.

---

## 2. Detailed Description
1. **Invocation**  
   The method is invoked with a merchant ID and a `Locale`.  
2. **Delegation**  
   It calls the core util `com.salesmanager.core.util.PaymentUtil.getPaymentMethods(merchantId, locale)`, which is expected to return a `Map` of payment method identifiers to `PaymentMethod` objects.  
3. **Transformation**  
   * The returned map is iterated over its keys.  
   * For each entry, the key is kept unless the `PaymentMethod` has `type == 1` (credit card).  
   * In that case the key is overwritten with `"GATEWAY"` – effectively grouping all credit‑card methods under a single key.  
   * The entry is inserted into a new map (`returnMap`).  
4. **Return**  
   The transformed map is returned to the caller.  

No cleanup or resource management is required.

**Assumptions & Constraints**  
- The core util never returns `null` and the map keys are `String`.  
- Only a single credit‑card entry is expected; otherwise subsequent `"GATEWAY"` keys will overwrite previous ones.  
- The method signature throws a generic `Exception`, implying callers must handle any checked exception from the core util.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `public static Map<String, PaymentMethod> getPaymentMethods(int merchantId, Locale locale)` | Fetches payment methods from the core util and rewrites credit‑card keys to `"GATEWAY"`. | `merchantId` – the merchant’s identifier.<br>`locale` – the language/country context. | `Map<String, PaymentMethod>` – transformed map. | None (apart from potential logging, which is currently unused). |

*Reusable/Utility Methods*:  
None beyond the single public method. The class could be expanded to expose additional helper functions for key normalization or filtering.

---

## 4. Dependencies

| Dependency | Type | Purpose |
|-------------|------|---------|
| `com.salesmanager.core.util.PaymentUtil` | Third‑party (Sales Manager core library) | Provides the original payment‑method map. |
| `com.salesmanager.core.entity.payment.PaymentMethod` | Core domain entity | Represents individual payment methods. |
| `org.apache.log4j.Logger` | Third‑party (Log4j) | Declared for potential logging (currently unused). |
| Standard JDK classes (`java.util.*`, `java.util.Locale`) | Standard | Basic collections and locale handling. |

No platform‑specific or environment assumptions are evident beyond the availability of the Sales Manager core library and Log4j.

---

## 5. Additional Notes

### Edge Cases & Potential Issues
1. **Raw Types** – The method uses non‑generic `Map` and `HashMap`. This results in unchecked cast warnings and reduces type safety. It should be declared as `Map<String, PaymentMethod>` throughout.
2. **Key Collision** – Replacing any credit‑card key with `"GATEWAY"` will overwrite previous entries. If the core util returns multiple credit‑card methods, only the last one survives.
3. **Null Handling** – No null‑checks on the returned map or individual `PaymentMethod` objects. A `NullPointerException` would occur if `null` values are present.
4. **Exception Handling** – Throwing a generic `Exception` forces callers to catch all checked exceptions. A more specific exception hierarchy would improve clarity.
5. **Unused Logger** – The `log` field is never used. Either remove it or add meaningful log statements (e.g., for debugging missing methods or key collisions).
6. **Iteration Style** – The code uses a classic `Iterator`. Java 5+ for‑each loops or streams would be cleaner.

### Suggested Enhancements
- **Generics**: Replace raw types with generic collections to enforce compile‑time safety.
- **Key Normalization**: Provide a helper method to decide the key for each payment method, making it reusable for other contexts.
- **Collision Warning**: Log a warning when a `"GATEWAY"` key already exists to aid debugging.
- **Null‑safe API**: Return an empty map instead of propagating `null`, or throw a custom exception if the core util fails.
- **Unit Tests**: Add tests covering normal operation, multiple credit‑card methods, null values, and exception propagation.
- **Logging**: Use the `log` instance to record important events (e.g., number of methods retrieved, any key collisions).

By addressing these points the class would become safer, more maintainable, and easier to understand for future developers.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.checkout.util;

import java.util.HashMap;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.payment.PaymentMethod;

public class PaymentUtil {

	private static Logger log = Logger.getLogger(PaymentUtil.class);

	public static Map<String, PaymentMethod> getPaymentMethods(int merchantId,
			Locale locale) throws Exception {

		Map payments = com.salesmanager.core.util.PaymentUtil
				.getPaymentMethods(merchantId, locale);

		Map returnMap = new HashMap();
		// now, change credit card payment
		Iterator i = payments.keySet().iterator();
		while (i.hasNext()) {
			String key = (String) i.next();
			PaymentMethod pm = (PaymentMethod) payments.get(key);
			if (pm.getType() == 1) {
				key = "GATEWAY";
			}
			returnMap.put(key, pm);
		}

		return returnMap;

	}

}



```
