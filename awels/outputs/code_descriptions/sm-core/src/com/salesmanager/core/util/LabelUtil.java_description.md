# LabelUtil.java

## Review

## 1. Summary  
**Purpose** – `LabelUtil` is a thin façade for fetching localized text messages.  
It abstracts two back‑ends:  

1. **Struts2** – when a Struts2 `ActionContext` is available it delegates to the framework’s `TextProviderFactory`.  
2. **Custom** – when Struts is not available it loads a list of `ResourceBundle` files (configured via `struts.custom.i18n.resources`) and performs `MessageFormat` substitution.

**Key components**

| Class | Responsibility |
|-------|----------------|
| `LabelUtil` | Public API; holds a mutable `Locale`, resolves the appropriate `SmTextProvider`, and delegates all lookup calls. |
| `SmTextProvider` | Interface shared by both providers. |
| `StrutsLabelUtil` | Delegates to Struts2’s text resolution API. |
| `CustomLabelUtil` | Loads resource bundles from configuration and performs simple string replacement with `MessageFormat`. |

**Design patterns / libraries**

* Factory‑like behaviour (`getTextProvider`) chooses an implementation at runtime.  
* Uses **Struts2** (`ActionContext`, `TextProviderFactory`) when available.  
* Leverages Java’s built‑in `ResourceBundle` and `MessageFormat`.  
* Minor use of **Apache Commons** (`StringUtils`).

---

## 2. Detailed Description  

### Flow of execution

1. **Initialization** – No global initialisation occurs.  
   * `LabelUtil` is created via `LabelUtil.getInstance()` (which actually constructs a *new* instance every call).  
   * The static block in `CustomLabelUtil` pulls a list of bundle names from the application configuration (`struts.custom.i18n.resources`).  
2. **Runtime behaviour** – The caller picks one of the overloaded `getText(..)` methods.  
   * All overloads eventually call `getTextProvider()` which checks `ActionContext.getContext()`.  
   * If a context is present, a `StrutsLabelUtil` instance is returned; otherwise a `CustomLabelUtil`.  
   * The chosen provider receives the `Locale` (either from the instance field or the passed parameter) and the key / parameters.  
3. **Cleanup** – None. All resources are short‑lived (strings, `ResourceBundle` objects cached by the JVM).

### Assumptions / constraints

| Assumption | Effect |
|------------|--------|
| Only the languages “en” and “fr” are handled explicitly; all others are constructed via `new Locale(lang)` | Limited support for complex locales (e.g., “en_US”). |
| `LabelUtil` instance is *thread‑local* – each request constructs its own instance | Safe but costly; no caching of the provider. |
| `CustomLabelUtil` reads bundle names from a configuration property `struts.custom.i18n.resources` | Mis‑configuration results in an empty bundle list and thus missing messages. |
| All parameter lists contain `String` objects | If the caller passes non‑String objects, casting will fail. |

### Architectural notes

* The façade is intentionally lightweight but at the cost of **performance**: a new `TextProviderFactory` is created on each call when Struts is used.  
* The provider selection is **global** – it uses `ActionContext` to decide, but if the application uses both Struts and non‑Struts code paths, the choice can flip unexpectedly.  
* The locale is stored in the `LabelUtil` instance; however many overloads ignore the passed `Locale` parameter and rely on the instance’s current locale, leading to inconsistent behaviour.

---

## 3. Functions/Methods  

| Class | Method | Purpose | Inputs | Outputs | Side‑Effects |
|-------|--------|---------|--------|---------|--------------|
| `LabelUtil` | `getInstance()` | Factory for `LabelUtil` (not a true singleton). | None | New `LabelUtil` instance | None |
| | `setLocale(Locale)` | Stores locale in instance. | `Locale` | None | Updates instance field |
| | `getLocale()` | Retrieves stored locale. | None | `Locale` | None |
| | `getText(String)` | Retrieves text for current locale. | Key | `String` | Uses provider |
| | `getText(HttpServletRequest, String)` | Uses request locale. | `HttpServletRequest`, key | `String` | Sets locale from request |
| | `getText(Locale, String)` | Uses passed locale. | `Locale`, key | `String` | Sets locale |
| | `getText(String lang, String key, String parameter)` | Locale by language code + single parameter. | Language code, key, param | `String` | Sets locale via `getLocale(lang)` |
| | `getText(Locale, String, List)` | Locale + multiple params. | `Locale`, key, params | `String` | **Bug** – does not use passed locale; uses instance locale |
| | `getText(String lang, String key)` | Locale by language code. | Language code, key | `String` | Sets locale |
| | `getLocale(String)` | Helper to resolve `Locale` from language string. | Language code | None | Sets instance locale |
| | `getTextProvider()` | Chooses provider implementation. | None | `SmTextProvider` | Instantiates `CustomLabelUtil` or `StrutsLabelUtil` |

**Interface `SmTextProvider`**

| Method | Purpose | Inputs | Output |
|--------|---------|--------|--------|
| `getText(Locale, String)` | Simple lookup | `Locale`, key | `String` |
| `getText(Locale, String, List)` | Lookup + format with list | `Locale`, key, params | `String` |
| `getText(Locale, String, String)` | Lookup + format with single param | `Locale`, key, param | `String` |

**`StrutsLabelUtil`**

* Implements `LocaleProvider` and `SmTextProvider`.  
* Stores its own `Locale` (unused by `LocaleProvider` interface).  
* For each call, a new `TextProviderFactory` is created and used to obtain a `TextProvider`.  

**`CustomLabelUtil`**

* Static block loads bundle names from configuration.  
* Each lookup iterates over `bundleList`, tries `ResourceBundle.getBundle(bundle, locale)`, and returns the first key found.  
* For parameterized lookups it uses `MessageFormat` to substitute values.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party (Apache Commons Configuration) | Only used to read `struts.custom.i18n.resources` |
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | Used in `getLocale(String)` to check for blank |
| `javax.servlet.http.HttpServletRequest` | Standard (Java EE) | For locale extraction |
| `com.opensymphony.xwork2.*` | Struts2 framework | `ActionContext`, `LocaleProvider`, `TextProvider`, `TextProviderFactory` |
| Java SE (`Locale`, `ResourceBundle`, `MessageFormat`, `List`, `Iterator`) | Standard | Core functionality |

No native platform‑specific APIs are used; the code is portable across any servlet container that supports the listed libraries.

---

## 5. Additional Notes & Recommendations  

### 5.1. Thread Safety & Singleton
* `LabelUtil.getInstance()` returns a *new* instance every call, defeating the typical singleton pattern.  
* The instance holds mutable state (`locale`) and is therefore **not thread‑safe** if the same object is shared across requests.  
* **Fix** – Implement a thread‑local or request‑scoped bean, or make all methods static and remove mutable state.

### 5.2. Inconsistent Locale Handling
* Many overloads ignore the `Locale` passed to them (e.g., `getText(Locale, String, List)` still uses the instance locale).  
* This can lead to subtle bugs when callers expect the passed locale to take precedence.  
* **Fix** – Either remove the instance locale altogether and always use the parameter, or enforce that callers use the instance correctly.

### 5.3. Performance
* `StrutsLabelUtil` creates a new `TextProviderFactory` and `TextProvider` on each call – costly for high‑traffic applications.  
* `CustomLabelUtil` iterates over the entire bundle list for every lookup.  
* **Fix** – Cache the provider instances and/or the `ResourceBundle` lookups; consider using a dedicated `ResourceBundle.Control` with caching disabled only when necessary.

### 5.4. Error Handling
* Missing keys return an empty string silently; no fallback to a default message.  
* Exceptions thrown while loading bundles are swallowed, making debugging hard.  
* **Fix** – Log missing keys or configuration issues; optionally throw a custom unchecked exception.

### 5.5. Parameter Type Safety
* `MessageFormat` is used with `String[]` for list parameters, but the interface accepts `List`.  
* If callers pass non‑String objects, a `ClassCastException` will be thrown.  
* **Fix** – Either change the API to accept `Object...` or document that parameters must be Strings.

### 5.6. Locale Resolution
* `getLocale(String)` only handles “en” and “fr”; all other codes produce `new Locale(lang)`.  
* Locale strings such as “en_US” or “fr_CA” are not supported.  
* **Fix** – Use `Locale.forLanguageTag(lang)` or split language and country if present.

### 5.7. Configuration Dependency
* `CustomLabelUtil` expects the property `struts.custom.i18n.resources` to be a list.  
* If the property is missing or incorrectly typed, `bundleList` remains empty, causing all lookups to fail.  
* **Fix** – Validate configuration during startup and provide a clear error message.

### 5.8. Future Enhancements
* Support **parameter maps** (key/value) in addition to simple lists.  
* Add **asynchronous/async** message resolution for expensive lookups.  
* Expose a **message interpolation** API that can handle placeholders beyond `MessageFormat`.  
* Provide a **fallback chain**: first try Struts provider, then custom bundles, finally a hard‑coded default.

---

### Bottom line  
The code provides a functional, but **fragile** implementation of i18n support that mixes Struts2 and custom resource bundles. It works for simple use cases but will suffer from concurrency issues, inconsistent locale handling, and performance bottlenecks in production. Refactoring to a cleaner, thread‑safe API with proper caching and error handling would make it robust and easier to maintain.

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

import java.text.MessageFormat;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.ResourceBundle;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;

import com.opensymphony.xwork2.ActionContext;
import com.opensymphony.xwork2.LocaleProvider;
import com.opensymphony.xwork2.TextProvider;
import com.opensymphony.xwork2.TextProviderFactory;

public class LabelUtil {

	private Locale locale;

	private LabelUtil() {

	}

	public static LabelUtil getInstance() {

		return new LabelUtil();
	}

	public void setLocale(Locale locale) {
		this.locale = locale;
	}

	public Locale getLocale() {
		return locale;
	}

	public String getText(String aTextName) {

		if (locale == null) {
			setLocale(LocaleUtil.getDefaultLocale());
		}

		return getTextProvider().getText(getLocale(), aTextName);
	}

	public String getText(HttpServletRequest request, String aTextName) {

		setLocale(LocaleUtil.getLocale(request));
		return getTextProvider().getText(getLocale(), aTextName);
	}

	public String getText(Locale locale, String aTextName) {

		setLocale(locale);
		return getTextProvider().getText(getLocale(), aTextName);
	}

	public String getText(String lang, String aTextName, String parameter) {
		getLocale(lang);
		return getTextProvider().getText(getLocale(), aTextName, parameter);
	}

	public String getText(Locale locale, String aTextName, List params) {

		return getTextProvider().getText(getLocale(), aTextName, params);
	}

	public String getText(String lang, String aTextName) {

		getLocale(lang);
		return getTextProvider().getText(getLocale(), aTextName);
	}

	private void getLocale(String lang) {
		
		if(StringUtils.isBlank(lang)) {
			lang = LanguageUtil.getDefaultLanguage();
		}
		
		if (lang.equals("en")) {
			setLocale(Locale.ENGLISH);
		} else if (lang.equals("fr")) {
			setLocale(Locale.FRENCH);
		} else {
			setLocale(new Locale(lang));
		}
	}

	private SmTextProvider getTextProvider() {
		ActionContext ctx = ActionContext.getContext();
		if (ctx == null) {// use the other method
			return new CustomLabelUtil();
		} else {// use struts method
			return new StrutsLabelUtil();
		}
	}

}

interface SmTextProvider {
	String getText(Locale locale, String key);

	String getText(Locale locale, String key, List parameters);

	String getText(Locale locale, String key, String parameter);
}

class StrutsLabelUtil implements LocaleProvider, SmTextProvider {

	private Locale locale;

	private void setLocale(Locale locale) {
		this.locale = locale;
	}

	public Locale getLocale() {
		return locale;
	}

	public String getText(Locale locale, String key) {
		this.setLocale(locale);
		TextProvider textProvider = (new TextProviderFactory()).createInstance(
				StrutsLabelUtil.class, this);
		return textProvider.getText(key);
	}

	public String getText(Locale locale, String key, List parameters) {
		this.setLocale(locale);
		TextProvider textProvider = (new TextProviderFactory()).createInstance(
				StrutsLabelUtil.class, this);
		return textProvider.getText(key, parameters);
	}

	public String getText(Locale locale, String key, String parameter) {
		this.setLocale(locale);
		TextProvider textProvider = (new TextProviderFactory()).createInstance(
				StrutsLabelUtil.class, this);
		return textProvider.getText(key, new String[] { parameter });
	}
}

class CustomLabelUtil implements SmTextProvider {

	private static List bundleList = new ArrayList();

	static {// load config files

		Configuration props = PropertiesUtil.getConfiguration();
		List lst = (List) props.getProperty("struts.custom.i18n.resources");
		if (lst != null) {
			bundleList = lst;
		}

	}

	public String getText(Locale locale, String key) {

		Iterator bundleListIterator = bundleList.iterator();
		ResourceBundle myResources = null;
		String label = "";
		while (bundleListIterator.hasNext()) {
			String bundle = (String) bundleListIterator.next();
			//try {
				myResources = ResourceBundle.getBundle(bundle, locale);
				if (myResources != null) {
					String l = myResources.getString(key);
					if (l != null) {
						label = l;
						break;
					}
				}

			//} catch (Exception e) {
				// TODO: handle exception
			//}

		}
		return label;
	}

	public String getText(Locale locale, String key, List parameters) {

		Iterator bundleListIterator = bundleList.iterator();
		ResourceBundle myResources = null;
		String label = "";
		while (bundleListIterator.hasNext()) {
			String bundle = (String) bundleListIterator.next();

			try {

				myResources = ResourceBundle.getBundle(bundle, locale);
				if (myResources != null) {
					String l = myResources.getString(key);
					if (l != null) {
						MessageFormat mFormat = new MessageFormat(l);
						String[] params = new String[parameters.size()];
						params = (String[]) parameters.toArray(params);
						l = mFormat.format(params);
						label = l;
						break;
					}
				}

			} catch (Exception e) {
				// Handle exception
			}

		}
		return label;
	}

	public String getText(Locale locale, String key, String parameter) {
		Iterator bundleListIterator = bundleList.iterator();
		ResourceBundle myResources = null;
		String label = "";
		while (bundleListIterator.hasNext()) {
			String bundle = (String) bundleListIterator.next();

			try {

				myResources = ResourceBundle.getBundle(bundle, locale);
				if (myResources != null) {
					String l = myResources.getString(key);
					if (l != null) {
						MessageFormat mFormat = new MessageFormat(l);
						l = mFormat.format(parameter);
						label = l;
						break;
					}
				}

			} catch (Exception e) {
				// TODO: handle exception
			}

		}
		return label;
	}
}



```
