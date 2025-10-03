# MerchantConfigurationUtil.java

## Review

## 1. Summary

`MerchantConfigurationUtil` is a pure‑utility class that serialises and deserialises merchant configuration values.  
Its responsibilities are:

| Method | Role |
|--------|------|
| `getConfigurationList` | Split a delimited string into a collection of tokens. |
| `buildConfigurationLine` | Join a collection of strings into a delimited string. |
| `getIntegrationProperties` | Parse a delimited string into an `IntegrationProperties` DTO. |
| `getIntegrationKeys` | Parse a delimited string into an `IntegrationKeys` DTO (user id, password, transaction key and up to three additional keys). |
| `getConfigurationValue` (overloads) | Build a delimited string from either a collection or an `IntegrationProperties` object, skipping empty values. |

The class is stateless and exposes all methods as `static`. It relies on `StringTokenizer` and Apache Commons Lang’s `StringUtils`. No design pattern beyond the “utility” pattern is evident.

---

## 2. Detailed Description

### Core Components

| Component | Purpose |
|-----------|---------|
| `getConfigurationList` | Tokenises the input line by the supplied delimiter. |
| `buildConfigurationLine` | Iterates over a collection of strings, concatenating them with the delimiter. |
| `getIntegrationProperties` | Maps the first five tokens of a delimited string to `properties1…5` in an `IntegrationProperties` object. |
| `getIntegrationKeys` | Maps the first three tokens to user credentials, then any remaining tokens to `key1…3` in an `IntegrationKeys` object. |
| `getConfigurationValue` | Two overloads: one concatenates a collection; the other concatenates non‑blank properties from an `IntegrationProperties` instance. |

### Execution Flow

1. **Deserialization** (`getConfigurationList`, `getIntegrationProperties`, `getIntegrationKeys`)  
   - The methods start by validating the input (null checks).  
   - They use `StringTokenizer` to break the string into tokens.  
   - A `while (st.hasMoreTokens())` loop assigns each token to the appropriate DTO field, incrementing counters (`i`, `j`) to keep track of position.  
   - The populated DTO is returned.

2. **Serialization** (`buildConfigurationLine`, `getConfigurationValue`)  
   - Methods iterate over the input collection or DTO fields, appending each element to a `StringBuffer` and inserting the delimiter only when required.  
   - The final string is returned.

3. **Cleanup** – No explicit resources to close; all operations are memory‑based.

### Assumptions & Constraints

| Assumption | Effect |
|------------|--------|
| Delimiter is a single character or string that does not appear in values. | If it appears in a value, tokenisation will split incorrectly. |
| `IntegrationProperties` and `IntegrationKeys` have exactly 5 and 6 fields respectively. | If the input string has more or fewer tokens, the remaining fields stay unset. |
| All values are already encrypted/decrypted. | The methods simply copy strings; decryption logic is omitted. |
| No concurrency concerns – the class is stateless. | Thread‑safety is guaranteed. |

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getConfigurationList` | `public static Collection getConfigurationList(String configurationLine, String delimiter)` | Splits a delimited string into a collection. | `configurationLine` (String), `delimiter` (String) | `Collection` of token strings (raw type). | None. |
| `buildConfigurationLine` | `public static String buildConfigurationLine(Collection<String> configs, String delimiter)` | Joins a collection into a delimited string. | `configs` (Collection<String>), `delimiter` (String) | `String` | None. |
| `getIntegrationProperties` | `public static IntegrationProperties getIntegrationProperties(String configurationValue, String delimiter)` | Parses string into `IntegrationProperties`. | `configurationValue` (String), `delimiter` (String) | `IntegrationProperties` object. | None. |
| `getIntegrationKeys` | `public static IntegrationKeys getIntegrationKeys(String configvalue, String delimiter) throws Exception` | Parses string into `IntegrationKeys`. | `configvalue` (String), `delimiter` (String) | `IntegrationKeys` object. | None. |
| `getConfigurationValue(Collection)` | `public static String getConfigurationValue(Collection<String> values, String delimiter)` | Builds delimited string from collection. | `values` (Collection<String>), `delimiter` (String) | `String` | None. |
| `getConfigurationValue(IntegrationProperties)` | `public static String getConfigurationValue(IntegrationProperties keys, String delimiter)` | Builds delimited string from non‑blank `IntegrationProperties`. | `keys` (IntegrationProperties), `delimiter` (String) | `String` | None. |

### Utility Notes

- **`StringTokenizer`** is legacy; modern code prefers `String.split()` or `String.join()`.  
- **`StringBuffer`** is synchronized; `StringBuilder` would be faster with no concurrency.  
- **Generics** are missing in the `Collection` return type of `getConfigurationList`.  
- **Error handling**: `getIntegrationKeys` declares `throws Exception` but never throws; the clause is unnecessary.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `java.util` (`ArrayList`, `Collection`, `Iterator`, `List`, `StringTokenizer`) | Standard | Core Java collections and tokeniser. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Used for blank‑string checks. |
| `com.salesmanager.core.service.common.model.IntegrationKeys` | Application | DTO representing credentials. |
| `com.salesmanager.core.service.common.model.IntegrationProperties` | Application | DTO representing generic properties. |

No platform‑specific APIs are used; the class is portable across any JVM.

---

## 5. Additional Notes

### Strengths

- **Simplicity** – Straightforward tokenisation and string building logic.  
- **Stateless** – No internal state; safe for concurrent use.  
- **Clear separation** – Deserialization and serialization responsibilities are split across dedicated methods.

### Weaknesses & Edge Cases

1. **Legacy APIs** – `StringTokenizer` and `StringBuffer` are outdated; they reduce readability and may lead to subtle bugs if the delimiter is a regex metacharacter.  
2. **Missing Generics** – `getConfigurationList` returns a raw `Collection`, which forces callers to cast.  
3. **Redundant Logic** – `buildConfigurationLine` and `getConfigurationValue(Collection)` perform identical work; one could be refactored into the other.  
4. **Delimiter Handling** – No escaping mechanism; a delimiter appearing in a value will corrupt the format.  
5. **Exception Signature** – `getIntegrationKeys` declares `throws Exception` but never throws; this may mislead callers.  
6. **Null/Empty Tokens** – Empty tokens are preserved; depending on the context, trimming might be desired.  
7. **Hard‑coded Limits** – `IntegrationProperties` accepts only five fields, and `IntegrationKeys` only three optional keys. If the data model changes, the code must be updated manually.

### Suggested Improvements

| Area | Recommendation |
|------|----------------|
| API Modernisation | Replace `StringTokenizer` with `String.split()` or `String.join()`; use `StringBuilder`. |
| Generics | Return `List<String>` (or `Collection<String>`) from `getConfigurationList`. |
| Method Consolidation | Merge `buildConfigurationLine` and `getConfigurationValue(Collection)` into a single utility. |
| Null‑Safety | Add defensive checks for `delimiter` being null or empty. |
| Documentation | Add Javadoc to each method, specifying pre‑conditions and post‑conditions. |
| Exception Handling | Remove `throws Exception` from `getIntegrationKeys` unless decryption logic is added. |
| Configuration Flexibility | Allow variable number of properties/keys by iterating over all tokens and assigning sequentially. |
| Unit Tests | Add comprehensive tests covering normal, empty, and malformed inputs. |

Implementing these changes will make the utility more robust, modern, and easier to maintain.

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
package com.salesmanager.core.util;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.StringTokenizer;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;

public class MerchantConfigurationUtil {

	/**
	 * Strips all delimiters
	 * 
	 * @param configurationLine
	 * @return
	 */
	public static Collection getConfigurationList(String configurationLine,
			String delimiter) {

		List returnlist = new ArrayList();
		StringTokenizer st = new StringTokenizer(configurationLine, delimiter);
		while (st.hasMoreTokens()) {
			String token = st.nextToken();
			returnlist.add(token);
		}
		return returnlist;
	}

	/**
	 * Build a line with delimiter
	 * 
	 * @param configs
	 * @param delimiter
	 * @return
	 */
	public static String buildConfigurationLine(Collection<String> configs,
			String delimiter) {
		StringBuffer keyLine = new StringBuffer();

		if (configs != null && configs.size() > 0) {
			int count = 1;
			Iterator i = configs.iterator();
			while (i.hasNext()) {
				String s = (String) i.next();
				keyLine.append(s);
				if (count < configs.size()) {
					keyLine.append(delimiter);
				}
				count++;
			}
		}

		return keyLine.toString();

	}

	public static IntegrationProperties getIntegrationProperties(
			String configurationValue, String delimiter) {
		if (configurationValue == null)
			return new IntegrationProperties();
		StringTokenizer st = new StringTokenizer(configurationValue, delimiter);
		int i = 1;
		IntegrationProperties keys = new IntegrationProperties();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			if (i == 1) {
				keys.setProperties1(value);
			} else if (i == 2) {
				keys.setProperties2(value);
			} else if (i == 3) {
				keys.setProperties3(value);
			} else if (i == 4) {
				keys.setProperties4(value);
			} else {
				keys.setProperties5(value);
			}

			i++;
		}
		return keys;
	}
	
	
	
	public static IntegrationKeys getIntegrationKeys(String configvalue, String delimiter)
	throws Exception {
		if (configvalue == null)
			return new IntegrationKeys();
		StringTokenizer st = new StringTokenizer(configvalue, delimiter);
		int i = 1;
		int j = 1;
		IntegrationKeys keys = new IntegrationKeys();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();

					if (i == 1) {
				// decrypt
				keys.setUserid(value);
			} else if (i == 2) {
				// decrypt
				keys.setPassword(value);
			} else if (i == 3) {
				// decrypt
				keys.setTransactionKey(value);
			} else {
				if (j == 1) {
					keys.setKey1(value);
				} else if (j == 2) {
					keys.setKey2(value);
				} else if (j == 3) {
					keys.setKey3(value);
				}
				j++;
			}
			i++;
		}
		return keys;
}
	

	public static String getConfigurationValue(Collection<String> values,
			String delimiter) {
		if (values == null || values.size() == 0) {
			return "";
		}
		int count = 1;
		Iterator i = values.iterator();
		StringBuffer b = new StringBuffer();
		while (i.hasNext()) {
			String value = (String) i.next();
			b.append(value);
			if (values.size() > count) {
				b.append(delimiter);
			}
			count++;
		}
		return b.toString();
	}

	public static String getConfigurationValue(IntegrationProperties keys,
			String delimiter) {

		if (StringUtils.isBlank(keys.getProperties1())
				&& StringUtils.isBlank(keys.getProperties2())
				&& StringUtils.isBlank(keys.getProperties3())
				&& StringUtils.isBlank(keys.getProperties4())
				&& StringUtils.isBlank(keys.getProperties5())) {
			return "";
		}
		StringBuffer b = new StringBuffer();
		b.append(keys.getProperties1());
		if (!StringUtils.isBlank(keys.getProperties2())) {
			b.append(delimiter);
			b.append(keys.getProperties2());
			if (!StringUtils.isBlank(keys.getProperties3())) {
				b.append(delimiter);
				b.append(keys.getProperties3());
				if (!StringUtils.isBlank(keys.getProperties4())) {
					b.append(delimiter);
					b.append(keys.getProperties4());
					if (!StringUtils.isBlank(keys.getProperties5())) {
						b.append(delimiter);
						b.append(keys.getProperties5());
					}
				}
			}
		}

		return b.toString();

	}
}



```
