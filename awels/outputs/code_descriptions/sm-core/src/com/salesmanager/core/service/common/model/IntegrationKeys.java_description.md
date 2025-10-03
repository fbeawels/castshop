# IntegrationKeys.java

## Review

## 1. Summary  
`IntegrationKeys` is a lightweight DTO that aggregates configuration keys used for external integrations (shipping, payment, etc.).  
* **Key fields** – `userid`, `password`, `transactionKey`, `key1`‑`key4`.  
* **Custom keys** – a generic `Map` (`customkeys`) that can hold any additional key/value pairs.  
* Implements the `ParsableConfiguration` interface (method body is currently empty).  
The class is deliberately simple, but several implementation details could be improved for correctness, type safety, and maintainability.

## 2. Detailed Description  
The class is essentially a container:

1. **State** – All fields are `String` (or a `Map`) with default `null`.  
2. **Getters** – Straight‑forward, simply return the field value.  
3. **Setters** – Attempt to trim whitespace but fail to assign the trimmed value back to the field, so the original value is stored unchanged.  
4. **Custom keys** – Exposed as a raw `Map` with getter/setter that replace the entire map.  
5. **`parse` method** – Intended to populate the object from a `MerchantConfiguration` instance, but the implementation is missing.  
6. **No additional logic** – No validation, `equals`, `hashCode`, or `toString` methods.

The class relies only on standard Java collections and `org.apache.commons.lang.StringUtils` for blank checks.

### Assumptions & Constraints  
* The caller guarantees that the incoming strings are not `null` before passing them to setters (otherwise the `trim` call would throw a `NullPointerException`, although `isBlank` guards against this).  
* `customkeys` is assumed to be a map of `String` → `String` (though the type is not enforced).  
* The `parse` method will eventually be implemented to convert a `MerchantConfiguration` into this DTO.

### Architecture & Design Choices  
* The class is intentionally *anemic* – no business logic is included.  
* It is designed for simplicity, but the lack of generics and proper trimming breaks encapsulation.  
* Using a plain `Map` instead of a typed collection makes the API less safe.  

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects | Notes |
|--------|---------|------------|---------|--------------|-------|
| `getCustomkeys()` | Retrieve the custom key map | None | `Map` | None | Returns raw type, may expose internal state |
| `setCustomkeys(Map)` | Replace the custom key map | `customkeys` | None | Mutates internal map reference | Should validate map type |
| `getKey1()` … `getKey4()` | Getters for optional keys | None | `String` | None | |
| `setKey1(String)` … `setKey4(String)` | Setters that attempt to trim input | key value | None | Mutates field (but does not store trimmed value) | **Bug** – `trim()` result not used |
| `getPassword()` | Getter | None | `String` | None | |
| `setPassword(String)` | Setter with trim attempt | password | None | Mutates field | **Bug** – same trimming issue |
| `getUserid()` | Getter | None | `String` | None | |
| `setUserid(String)` | Setter with trim attempt | userid | None | Mutates field | **Bug** – same trimming issue |
| `getTransactionKey()` | Getter | None | `String` | None | |
| `setTransactionKey(String)` | Setter with trim attempt | transactionKey | None | Mutates field | **Bug** – same trimming issue |
| `parse(MerchantConfiguration)` | Intended to populate fields from an entity | `MerchantConfiguration` | None | Should set all fields | Currently empty – no functionality |

All setters share the same flaw: they call `trim()` without assigning the result back. This means any whitespace will remain in the stored value.

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Provides `isBlank()` to guard against null/empty strings. |
| Java Standard Library | Standard | `Map`, `HashMap`, `String`. |

No framework‑specific code is present, so the class is framework‑agnostic.

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Whitespace Preservation** – The current setter implementation keeps whitespace, which may cause authentication failures or mis‑configuration.  
2. **Null Values** – While `isBlank` protects against `null` during trimming, passing `null` to a setter will leave the field `null`, which may be acceptable but should be documented.  
3. **Raw Map** – Using a raw `Map` can lead to unchecked warnings and runtime `ClassCastException`s if non‑String keys or values are inserted.  
4. **Immutability** – The class is fully mutable; if used in concurrent contexts this could lead to race conditions.  
5. **Missing `parse` logic** – Without implementation, callers cannot rely on the integration keys being populated from the entity.  

### Suggested Improvements  
* **Correct Trimming** – Replace each setter’s body with:
  ```java
  this.key1 = StringUtils.isBlank(key1) ? null : key1.trim();
  ```
  or use a helper method to avoid repetition.  
* **Typed Map** – Change `customkeys` to `Map<String, String>` (or a more specific type) to provide compile‑time safety.  
* **Immutable DTO** – Consider making the class immutable: private final fields, no setters, and a constructor that receives all values.  
* **Builder Pattern** – For large numbers of optional keys, a builder can simplify object creation.  
* **Implement `parse`** – Map the relevant fields from `MerchantConfiguration` to this DTO.  
* **Add `equals`, `hashCode`, `toString`** – Useful for logging, unit tests, and collections.  
* **Documentation** – Add JavaDoc for each field and method explaining expected values (e.g., whether they can be `null`).  

Overall, the class fulfills its role as a simple configuration holder but requires several fixes to ensure correctness, type safety, and future maintainability.

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
package com.salesmanager.core.service.common.model;

import java.util.HashMap;
import java.util.Map;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.merchant.MerchantConfiguration;

/**
 * Used for containing the shipping / payment and other integration system keys
 * 
 * @author Carl Samson
 * 
 */
public class IntegrationKeys implements ParsableConfiguration {

	private String userid;
	private String password;
	private String transactionKey;
	private String key1;
	private String key2;
	private String key3;
	private String key4;

	private Map customkeys = new HashMap();

	public Map getCustomkeys() {
		return customkeys;
	}

	public void setCustomkeys(Map customkeys) {
		this.customkeys = customkeys;
	}

	public String getKey1() {
		return key1;
	}

	public void setKey1(String key1) {
		if (!StringUtils.isBlank(key1)) {
			key1.trim();
		}
		this.key1 = key1;
	}

	public String getKey2() {
		return key2;
	}

	public void setKey2(String key2) {
		if (!StringUtils.isBlank(key2)) {
			key2.trim();
		}
		this.key2 = key2;
	}

	public String getKey3() {
		return key3;
	}

	public void setKey3(String key3) {
		if (!StringUtils.isBlank(key3)) {
			key3.trim();
		}
		this.key3 = key3;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		if (!StringUtils.isBlank(password)) {
			password.trim();
		}
		this.password = password;
	}

	public String getUserid() {
		return userid;
	}

	public void setUserid(String userid) {
		if (!StringUtils.isBlank(userid)) {
			userid.trim();
		}
		this.userid = userid;
	}

	public void parse(MerchantConfiguration conf) {

	}

	public String getTransactionKey() {
		return transactionKey;
	}

	public void setTransactionKey(String transactionKey) {
		if (!StringUtils.isBlank(transactionKey)) {
			transactionKey.trim();
		}
		this.transactionKey = transactionKey;
	}

	public String getKey4() {
		return key4;
	}

	public void setKey4(String key4) {
		if (!StringUtils.isBlank(key4)) {
			key4.trim();
		}
		this.key4 = key4;
	}

}



```
