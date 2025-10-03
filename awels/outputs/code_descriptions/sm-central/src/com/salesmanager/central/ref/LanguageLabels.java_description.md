# LanguageLabels.java

## Review

## 1. Summary

`LanguageLabels` is a utility façade that exposes a handful of static helper methods for building **localized key‑value maps** used throughout the application (e.g., for rendering “Yes/No” options, success/failure messages, or product type names).  
The class is intentionally very lightweight – it pulls translated strings from a `ResourceBundle` (`central-catalog`) or from a custom `LabelUtil` helper.  

**Key components**

| Component | Role |
|-----------|------|
| `LanguageLabels` | Public static factory for mapping values to UI labels. |
| `Types` (inner class) | Loads product‑type names from a `ResourceBundle` and returns them by ID. |
| `LabelUtil` | Third‑party (internal) helper that manages locale‑aware look‑ups. |
| `ResourceBundle` | Standard JDK mechanism for externalising strings. |

**Design patterns / libraries**

* The code follows a **simple factory** pattern: each public method returns a `Map` that can be injected into UI components.  
* Uses the standard Java **ResourceBundle** for i18n.  
* No external frameworks (Spring, CDI, etc.) – pure Java SE.

---

## 2. Detailed Description

### Execution Flow

1. **Initialization**  
   The class is *not* stateful; all public methods create and return new `Map` instances.  
   The `Types` helper is instantiated on demand in `getProductTypes`.

2. **Runtime behavior**  
   * For the “Yes/No” and “True/False” helpers, the method obtains a singleton `LabelUtil` instance, sets the desired `Locale`, and then asks it for the translated text.  
   * `buildSuccessFail` hard‑codes English and French strings.  
   * `getProductTypes` loads the product‑type labels from `central‑catalog` and substitutes them into the passed `ProductType` objects.  
   * `useCVV` simply returns a hard‑coded map for CVV‑usage options.

3. **Cleanup**  
   No resources are held after a method returns – the maps are transient.

### Assumptions & Constraints

| Item | Assumption | Potential issue |
|------|------------|-----------------|
| `LabelUtil` | Provides a thread‑safe singleton with `setLocale()` | If `setLocale()` mutates shared state, concurrent calls could corrupt other threads' look‑ups. |
| Resource files | Must contain keys `label.product.types.n` for every requested type | Missing keys throw `MissingResourceException` at runtime. |
| Languages | Only “en” and “fr” are supported | Adding another language requires editing the source. |
| Map types | Raw `Map` used everywhere | No compile‑time type safety; callers may cast incorrectly. |
| Locale vs language | Some methods accept `Locale`, others accept `String` | Inconsistent API surface. |

### Architecture & Design Choices

* **Static façade** – eliminates object construction but sacrifices dependency injection and testability.  
* **Inner helper class** – `Types` is tightly coupled to `LanguageLabels` and re‑loads the `ResourceBundle` on every call, which is wasteful.  
* **Hard‑coded values** – `buildSuccessFail` and `useCVV` embed strings directly in code instead of externalising them.  
* **Raw collections** – The use of `Map` without generics (`Map` → `Map<Object,Object>`) removes type safety and makes the code harder to maintain.

---

## 3. Functions/Methods

### `LanguageLabels`

| Method | Signature | Purpose | Inputs | Outputs | Side Effects |
|--------|-----------|---------|--------|---------|--------------|
| `buildYesNo(Locale locale)` | `public static Map buildYesNo(Locale)` | Builds a map `{1:"Yes",0:"No"}` localized to the supplied locale. | `Locale` | `Map<Integer,String>` | Calls `LabelUtil.getInstance()` and `setLocale()` (possible shared state mutation). |
| `buildSuccessFail(String lang)` | `public static Map buildSuccessFail(String)` | Hard‑coded success/failure strings for two languages. | `String` language code (`"en"`/`"fr"`) | `Map<Integer,String>` | None. |
| `buildTrueFalse(Locale locale)` | `public static Map buildTrueFalse(Locale)` | Same as `buildYesNo` (duplicate). | `Locale` | `Map<Integer,String>` | Same as `buildYesNo`. |
| `getProductTypes(String lang, Collection types)` | `public static Map getProductTypes(String,Collection)` | Replaces the `typeName` of each `ProductType` in the supplied collection with a localized label from the bundle. | `String` language code, `Collection<ProductType>` | `Map<Integer,ProductType>` (ID → updated `ProductType`) | Mutates `ProductType` objects in place; loads `ResourceBundle` each call. |
| `useCVV(String lang)` | `public static Map useCVV(String)` | Returns a map of CVV usage options in the requested language. | `String` language code | `Map<String,String>` | None. |

### `Types` (inner helper)

| Method | Signature | Purpose | Inputs | Outputs | Side Effects |
|--------|-----------|---------|--------|---------|--------------|
| `setLang(String lang)` | `public void setLang(String)` | Stores language code in an instance variable (unused). | `String` | None. |
| `load(int maxsize)` | `public void load(int)` | Loads product type labels from the `central-catalog` bundle into `content`. | `int` maximum number of types | None (mutates internal `content` map). |
| `getType(int i)` | `public String getType(int)` | Returns the type name for a given numeric ID. | `int` type ID | `String` type name or empty string if not found. | None. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `java.util.*` | Standard library | Collections, Locale, ResourceBundle |
| `com.salesmanager.core.util.LabelUtil` | Internal, third‑party | Handles locale‑aware text look‑ups |
| `com.salesmanager.core.constants.PaymentConstants` | Unused | Imported but never referenced |
| `com.salesmanager.core.entity.reference.ProductType` | Internal | Domain entity used in `getProductTypes` |
| `com.salesmanager.core.util.LabelUtil` | Third‑party | Singleton pattern; not thread‑safe if `setLocale()` mutates shared state |

---

## 5. Additional Notes

### Edge Cases & Potential Failures

| Area | Edge Case | Current Behaviour | Suggested Remedy |
|------|-----------|-------------------|------------------|
| `LabelUtil` | Concurrent calls with different locales | Potential race condition if `LabelUtil` mutates shared state | Make locale a method parameter or use a thread‑local wrapper |
| `ResourceBundle` | Missing key (`label.product.types.n`) | `MissingResourceException` propagates | Wrap load in try/catch; provide fallback (e.g., “Unknown”) |
| `buildSuccessFail` | Unsupported language | Returns empty map | Default to English or throw `IllegalArgumentException` |
| `useCVV` | Unsupported language | Returns empty map | Similar fallback strategy |
| `Types.load` | Bundle not found | `bundle` remains null → loop skipped silently | Throw informative exception or log warning |
| Raw collections | Type safety | Compile‑time warnings; runtime cast errors | Use generics (`Map<Integer, String>`) |

### Design Improvements

1. **Remove raw types** – switch to `Map<Integer, String>` or `Map<String, String>` throughout.  
2. **Centralise i18n** – Move all string constants into the `central-catalog` bundle; eliminate hard‑coded strings.  
3. **Thread‑safe locale handling** – Instead of mutating a singleton, pass the locale directly to `LabelUtil` or use a per‑thread instance.  
4. **Cache bundle data** – `Types` should load once (static initializer) or use a `Map<Integer,String>` cache to avoid repeated `ResourceBundle` look‑ups.  
5. **Expose type safety** – The `getProductTypes` method could return `List<ProductType>` or `Map<Integer,ProductType>` with generics, and avoid mutating input objects.  
6. **Rename or remove deprecated methods** – `buildTrueFalse` duplicates `buildYesNo`; consider consolidating.  
7. **Remove unused imports** – `PaymentConstants` is never used.  
8. **Unit‑testability** – Refactor to instance methods and inject dependencies (`LabelUtil`, `ResourceBundle`) to allow mocking.  
9. **Consistent API** – Use `Locale` consistently for all language parameters instead of a mix of `Locale` and `String`.  

### Future Enhancements

* **Internationalisation framework** – Adopt Spring's `MessageSource` or Java’s `ResourceBundleMessageSource` for more flexible i18n.  
* **Enum‑based type IDs** – Replace numeric keys with an enum for product types, improving readability and safety.  
* **Immutability** – Return unmodifiable maps to avoid accidental modification by callers.  
* **Error handling** – Provide clear fallback mechanisms for missing translations.  

--- 

**Conclusion** – The class fulfills its basic goal of providing localized label maps but suffers from several design shortcomings: raw collections, hard‑coded strings, potential thread‑safety issues, and repeated bundle loading. Refactoring to a more modern, type‑safe, and testable design would greatly improve maintainability and robustness.

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
package com.salesmanager.central.ref;

import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;
import java.util.ResourceBundle;

import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.reference.ProductType;
import com.salesmanager.core.util.LabelUtil;

public class LanguageLabels {


	/** slowly deprecating all this ! **/

	public static Map buildYesNo(Locale locale) {

		
		
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(locale);
		
		Map env = new HashMap();

		env.put(new Integer(1).intValue(), label.getText("label.generic.yes"));
		env.put(new Integer(0).intValue(), label.getText("label.generic.no"));

		return env;
	}

	public static Map buildSuccessFail(String lang) {

		Map env = new HashMap();
		if (lang.equals("en")) {
			env.put(new Integer(1).intValue(), "Success");
			env.put(new Integer(0).intValue(), "Fail");
		}
		if (lang.equals("fr")) {
			env.put(new Integer(1).intValue(), "Succès");
			env.put(new Integer(0).intValue(), "Échec");
		}
		return env;
	}

	public static Map buildTrueFalse(Locale locale) {

		
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(locale);
		Map env = new HashMap();


		env.put(new Integer(1).intValue(), label.getText("label.generic.yes"));
		env.put(new Integer(0).intValue(), label.getText("label.generic.no"));


		return env;
	}

	public static Map getProductTypes(String lang, Collection types) {
		Map typesmap = new HashMap();

		Types t = new Types();
		if (lang.equals("en")) {
			t.setLang("en");
		}
		if (lang.equals("fr")) {
			t.setLang("en");
		}
		t.load(types.size());
		Iterator kit = types.iterator();
		while (kit.hasNext()) {
			ProductType pt = (ProductType) kit.next();
			pt.setTypeName(t.getType(pt.getTypeId()));
			typesmap.put(pt.getTypeId(), pt);
		}

		return typesmap;

	}

	public static Map useCVV(String lang) {
		Map cardactions = new HashMap();

		if (lang.equals("en")) {
			cardactions.put("1", "Do not use");
			cardactions.put("2", "Use");
		}
		if (lang.equals("fr")) {
			cardactions.put("1", "Ne pas utiliser");
			cardactions.put("2", "Utiliser");
		}
		return cardactions;

	}

}

class Types {

	private String localleng;
	private Map content = new HashMap();

	public void setLang(String lang) {
		localleng = lang;
	}

	public void load(int maxsize) {
		ResourceBundle bundle = null;
		bundle = ResourceBundle.getBundle("central-catalog");
		if (bundle != null) {
			for (int i = 1; i <= maxsize; i++) {
				String type = bundle.getString("label.product.types." + i);
				content.put(i, type);
			}
		}
	}

	public String getType(int i) {
		if (content.containsKey(new Integer(i))) {
			return (String) content.get(new Integer(i));
		} else {
			return "";
		}
	}
}



```
