# ShippingUtil.java

## Review

## 1. Summary

`ShippingUtil` is a small helper class that parses semicolon‑delimited strings into two domain objects:

| Method | Purpose |
|--------|---------|
| `getKeys(String)` | Builds an `IntegrationKeys` instance from a string formatted as `<userid>;<password>;<key1>;<key2>…` |
| `getProperties(String)` | Builds an `IntegrationProperties` instance from a string formatted as `<p1>;<p2>;<p3>;<p4>` |

The class resides in the `com.salesmanager.core.module.impl.integration.shipping` package and relies only on the custom models `IntegrationKeys` and `IntegrationProperties`. No external libraries or frameworks are used.

---

## 2. Detailed Description

### Core Flow

1. **Input** – The caller supplies a raw string (`keyline` or `propsline`) that is semicolon‑separated.
2. **Tokenization** – A `StringTokenizer` splits the string on `;`.  
   * The tokenizer is *legacy* but functional; it treats consecutive delimiters as separate tokens (empty strings will be returned).
3. **Iteration** – The code iterates over the tokens, using a simple counter (`i`) to decide which field to populate.  
   * For `getKeys`: first two tokens go to `userid` and `password`; the rest are mapped to `key1…key4` via a secondary counter (`j`).  
   * For `getProperties`: first four tokens are assigned to `properties1…properties4`. Any additional tokens are silently ignored.
4. **Return** – A fully populated instance of the target class is returned.

### Assumptions & Constraints

| Aspect | Detail |
|--------|--------|
| **Token Count** | The methods assume at least the required number of tokens. If fewer tokens are present, the corresponding fields remain unset. |
| **Order** | The fields are strictly position‑based; no key/value pairs are accepted. |
| **Maximum Size** | `getKeys` only stores up to four keys; `getProperties` only stores four properties. |
| **Null Input** | Passing `null` throws a `NullPointerException` from `StringTokenizer`. |
| **Whitespace** | Tokens are not trimmed; surrounding spaces will be preserved. |
| **Thread‑Safety** | The methods are static but use only local variables; they are thread‑safe as long as the returned domain objects are not shared concurrently. |

---

## 3. Functions/Methods

| Method | Signature | Inputs | Outputs | Side‑Effects | Notes |
|--------|-----------|--------|---------|--------------|-------|
| `public static IntegrationKeys getKeys(String keyline)` | `String` → `IntegrationKeys` | `keyline` – semicolon‑delimited credentials and keys | `IntegrationKeys` populated according to token position | None | *Uses `StringTokenizer`* |
| `public static IntegrationProperties getProperties(String propsline)` | `String` → `IntegrationProperties` | `propsline` – semicolon‑delimited properties | `IntegrationProperties` populated according to token position | None | *Uses `StringTokenizer`* |

Both methods are pure in the sense that they create a new instance and do not modify any global state. However, they expose the internal mutability of the returned objects; callers should treat the returned instances as *write‑once* unless intentional.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.service.common.model.IntegrationKeys` | Custom domain object | Holds user credentials and up to four keys. |
| `com.salesmanager.core.service.common.model.IntegrationProperties` | Custom domain object | Holds up to four properties. |
| `java.util.StringTokenizer` | JDK standard | Legacy class; could be replaced by `String.split` or `java.util.regex.Pattern`. |

No external libraries or frameworks are involved.

---

## 5. Additional Notes & Recommendations

### Strengths
- **Simplicity** – The class is small, focused, and easy to understand.
- **No external baggage** – Pure Java, no runtime dependencies.

### Weaknesses / Edge Cases
1. **Null Handling** – A `null` argument causes an immediate `NullPointerException`. Consider defensive checks or documenting the contract.
2. **Whitespace & Trimming** – Tokens like `"  user  "` will be stored verbatim. Trimming could prevent subtle bugs.
3. **Empty Tokens** – Consecutive semicolons produce empty strings, which are accepted and stored. Depending on business rules, this might be undesirable.
4. **Fixed Capacity** – `getKeys` silently discards any key beyond the fourth. If the integration expands to support more keys, the code must be rewritten.
5. **Legacy API** – `StringTokenizer` is considered legacy. Using `String.split(";")` (or a `Pattern`) is clearer and more flexible.
6. **Lack of Validation** – No checks that the first two tokens are non‑empty, or that `propsline` has at least four tokens. The method will silently return an object with null fields if the input is malformed.
7. **No Error Reporting** – If the input string is malformed, the caller receives a partially populated object without any indication of failure.

### Suggested Enhancements
| Idea | Benefit |
|------|---------|
| **Null & Blank Checks** – Return `Optional<IntegrationKeys>` or throw a custom `IllegalArgumentException` when input is null or invalid. | Prevents silent failures. |
| **Trimming Tokens** – `value.trim()` before assignment. | Reduces whitespace bugs. |
| **Dynamic Key Mapping** – Use a `Map<String, String>` inside `IntegrationKeys`/`IntegrationProperties` for arbitrary numbers of keys/properties. | Future‑proofs the integration. |
| **Refactor to Split** – Replace `StringTokenizer` with `String[] tokens = keyline.split(";")`. | Simplifies code, improves readability. |
| **Unit Tests** – Add tests covering normal, edge, and error cases. | Ensures reliability and documents expected behaviour. |
| **Builder Pattern** – Provide builders for the domain objects to make construction clearer. | Encapsulates field mapping logic. |

### Final Verdict
`ShippingUtil` serves a narrow, well‑defined purpose but would benefit from defensive programming and modernization. By addressing the edge cases and refactoring to modern Java idioms, the utility will become more robust and maintainable, especially if the integration evolves to handle more keys or properties.

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
package com.salesmanager.core.module.impl.integration.shipping;

import java.util.StringTokenizer;

import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;

public class ShippingUtil {

	public static IntegrationKeys getKeys(String keyline) {

		// 1->userid 2->password i->key
		StringTokenizer st = new StringTokenizer(keyline, ";");
		int i = 1;
		int j = 1;
		IntegrationKeys keys = new IntegrationKeys();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			if (i == 1) {
				keys.setUserid(value);
			} else if (i == 2) {
				keys.setPassword(value);
			} else {
				if (j == 1) {
					keys.setKey1(value);
				} else if (j == 2) {
					keys.setKey2(value);
				} else if (j == 3) {
					keys.setKey3(value);
				} else if (j == 4) {
					keys.setKey4(value);
				}
				j++;
			}
			i++;
		}
		return keys;

	}

	public static IntegrationProperties getProperties(String propsline) {

		// 1->userid 2->password i->key
		StringTokenizer st = new StringTokenizer(propsline, ";");
		int i = 1;
		int j = 1;
		IntegrationProperties props = new IntegrationProperties();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			if (i == 1) {
				props.setProperties1(value);
			} else if (i == 2) {
				props.setProperties2(value);
			} else if (i == 3) {
				props.setProperties3(value);
			} else if (i == 4) {
				props.setProperties4(value);
			}

			i++;
		}
		return props;

	}

}



```
