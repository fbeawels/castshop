# ReferenceUtil.java

## Review

## 1. Summary
`ReferenceUtil` is a utility class that centralises the construction of URLs used throughout the SalesManager application.  
Its primary responsibilities are:

| Function | Purpose |
|----------|---------|
| `getSecureDomain` / `getUnSecureDomain` | Resolve the HTTPS or HTTP base domain for a given `MerchantStore` (or fall back to a default). |
| `build*` methods | Compose full URLs for specific application resources (checkout, cart, login, invoice, central, bin, catalog, etc.). |

**Key components**

* **Configuration** – Uses `org.apache.commons.configuration.Configuration` loaded once at class‑load time via `PropertiesUtil.getConfiguration()`.  
* **StringUtils** – From Apache Commons Lang to guard against blank domain names.  
* **MerchantStore** – Domain information holder that the URL builders use.

The design is straightforward and functional; it simply concatenates strings to build URLs. No advanced design patterns are used, but the class follows a static‑utility style common in Java.

---

## 2. Detailed Description
### Initialization
```java
private static Configuration conf = PropertiesUtil.getConfiguration();
```
The configuration is loaded once when the class is first referenced. All subsequent URL resolutions read from this shared instance.

### Runtime Flow
When a URL method is invoked, the flow is:

1. **Domain resolution** – `getSecureDomain`/`getUnSecureDomain` fetch the domain from the `MerchantStore`. If the store is `null` or the domain is blank, a hard‑coded default (`"localhost"` or the server config value) is used.
2. **URL assembly** – A `StringBuffer` (or `StringBuilder` would be preferable) appends:
   * The protocol prefix (`http://` or `https://`) from the config,
   * The resolved domain,
   * A catalog or service path retrieved from the configuration,
   * An optional action/endpoint specific to the method.
3. **Return** – The concatenated string is returned.

The class has no stateful cleanup – all methods are pure functions that depend only on input parameters and the static configuration.

### Assumptions & Constraints
* `PropertiesUtil.getConfiguration()` returns a fully initialised `Configuration`; errors are not caught here.
* All configuration keys used must exist; otherwise `null` is appended, producing invalid URLs.
* The domain string is assumed to be a valid hostname; no validation or URL encoding is performed.
* The class is not thread‑safe regarding configuration changes (but it is immutable after load).

### Architecture & Design Choices
* **Static utility** – Simplifies usage (`ReferenceUtil.buildCheckoutUri(store)`) but makes unit testing harder (no easy injection of configuration).
* **String concatenation** – Uses `StringBuffer` which is thread‑safe but unnecessary; `StringBuilder` would be more efficient.
* **Hard‑coded defaults** – Fallbacks to `"localhost"` or config values; these could be externalised or parameterised.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getSecureDomain(MerchantStore)` | Builds the HTTPS domain string. | `store` | `String` | None |
| `getUnSecureDomain(MerchantStore)` | Builds the HTTP domain string. | `store` | `String` | None |
| `buildCheckoutUri(MerchantStore)` | Full checkout URL (secure). | `store` | `String` | None |
| `buildCartUri(MerchantStore)` | Full cart URL (unsecure). | `store` | `String` | None |
| `buildRemoteLogonUrl(MerchantStore)` | URL for remote login via Ajax. | `store` | `String` | None |
| `buildCheckoutShowCartUrl(MerchantStore)` | Show cart from checkout (unsecure). | `store` | `String` | None |
| `buildDisplayInvoiceUrl(MerchantStore)` | Display invoice URL. | `store` | `String` | None |
| `buildCentralUri(MerchantStore)` | Central service URL. | `store` | `String` | None |
| `buildBinUri(MerchantStore)` | Media bin URL. | `store` | `String` | None |
| `buildCatalogUri(MerchantStore)` | Catalog service URL. | `store` | `String` | None |
| `buildCheckoutToCartUrl(MerchantStore)` | Checkout‑to‑cart action URL. | `store` | `String` | None |

All methods are **public static**, making the class a pure helper with no state beyond the static configuration. No other reusable utilities are exposed; the only shared logic is in the two domain resolution methods.

---

## 4. Dependencies
| Library | Purpose | Notes |
|---------|---------|-------|
| `org.apache.commons.configuration.Configuration` | Configuration key/value access. | Third‑party (Apache Commons Configuration). |
| `org.apache.commons.lang.StringUtils` | Null/blank string checking. | Third‑party (Apache Commons Lang). |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Holds store metadata (domain name). | Internal domain model. |
| `com.salesmanager.core.util.PropertiesUtil` | Loads configuration. | Internal utility. |

No other frameworks or platform‑specific APIs are used. The class relies on proper configuration files being available and correctly formatted.

---

## 5. Additional Notes & Recommendations
### Strengths
* **Simplicity** – Easy to read and use; all URL patterns are centralised.  
* **Extensibility** – New URL builders can be added with minimal code.

### Weaknesses / Edge Cases
1. **Missing Config Keys** – If any key is absent, `conf.getString(...)` returns `null`, resulting in URLs containing the literal string `"null"` – potentially breaking the application. Consider adding validation or fallback defaults.
2. **StringBuffer vs. StringBuilder** – `StringBuffer` is synchronized; using `StringBuilder` would improve performance with negligible overhead.
3. **Thread‑Safety** – The static `conf` is immutable after load; however, if the configuration can be reloaded at runtime, this class would become inconsistent.
4. **URL Encoding** – Domain or path components are not URL‑encoded; if they contain special characters, the resulting URL could be malformed.
5. **Hard‑coded `"localhost"`** – In production, the default should probably be derived from configuration rather than a literal.

### Potential Enhancements
* **Injectable Configuration** – Refactor to accept a `Configuration` instance (or a dedicated `UrlConfiguration` DTO) via constructor or method argument to improve testability.
* **Centralised Path Builder** – Create a small helper (`UrlBuilder`) that handles protocol, domain, and path concatenation with validation and encoding.
* **Unit Tests** – Add tests covering:
  * Null `MerchantStore`,
  * Blank domain name,
  * Missing config keys,
  * Correct URL formation.
* **Exception Handling** – Wrap configuration access in a try/catch block or perform explicit checks to provide meaningful errors.
* **Performance** – Replace `StringBuffer` with `StringBuilder` and use `String.format` or `URIBuilder` for clarity.

### Conclusion
`ReferenceUtil` is a pragmatic, if somewhat rudimentary, utility for URL construction in the SalesManager application. Its straightforward approach works for the current use‑case, but the code would benefit from minor refactors around configuration handling, thread‑safety, and performance to make it more robust and maintainable.

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

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.merchant.MerchantStore;

public class ReferenceUtil {

	private static Configuration conf = PropertiesUtil.getConfiguration();

	public static String getSecureDomain(MerchantStore store) {
		String domain = "localhost";
		if (store != null && !StringUtils.isBlank(store.getDomainName())) {
			domain = store.getDomainName();
		}
		StringBuffer url = new StringBuffer();

		return url.append((String) conf.getString("core.domain.http.secure"))
				.append("://").append(domain).toString();
	}

	public static String getUnSecureDomain(MerchantStore store) {

		String domain = conf.getString("core.domain.server");
		
		if (store != null && !StringUtils.isBlank(store.getDomainName())) {
			domain = store.getDomainName();
		}
		StringBuffer url = new StringBuffer();

		return url.append((String) conf.getString("core.domain.http.unsecure"))
				.append("://").append(domain).toString();
	}

	public static String buildCheckoutUri(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk.append(getSecureDomain(store)).append(
				(String) conf.getString("core.salesmanager.catalog.url"))
				.append("/").append(
						(String) conf
								.getString("core.salesmanager.checkout.uri"));
		return chk.toString();
	}

	public static String buildCartUri(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk.append(getUnSecureDomain(store)).append(
				(String) conf.getString("core.salesmanager.catalog.url"))
				.append("/").append(
						(String) conf.getString("core.salesmanager.cart.uri"));
		return chk.toString();
	}

	public static String buildRemoteLogonUrl(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk
				.append(getSecureDomain(store))
				.append(
						(String) conf
								.getString("core.salesmanager.catalog.url"))
				.append(
						(String) conf
								.getString("core.accountmanagement.loginAjaxAction"));
		return chk.toString();
	}

	public static String buildCheckoutShowCartUrl(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk
				.append(getUnSecureDomain(store))
				.append(
						(String) conf
								.getString("core.salesmanager.catalog.url"))
				.append("/")
				.append(
						(String) conf
								.getString("core.salesmanager.checkout.uri"))
				.append(
						(String) conf
								.getString("core.salesmanager.checkout.showCartAction"));
		return chk.toString();
	}
	
	public static String buildDisplayInvoiceUrl(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk
				.append(getUnSecureDomain(store))
				.append(
						(String) conf
								.getString("core.salesmanager.catalog.url"))
				.append("/")
				.append(
						(String) conf
								.getString("core.salesmanager.checkout.uri"))
				.append(
						(String) conf
								.getString("core.salesmanager.checkout.showInvoiceAction"));
		return chk.toString();
	}

	public static String buildCentralUri(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk.append(getSecureDomain(store)).append(
				(String) conf.getString("core.salesmanager.central.url"));
		return chk.toString();
	}

	public static String buildBinUri(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk.append(getSecureDomain(store)).append(
				(String) conf.getString("core.store.mediaurl")).append(
				(String) conf.getString("core.bin.uri"));
		return chk.toString();
	}

	public static String buildCatalogUri(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk.append(getUnSecureDomain(store)).append(
				(String) conf.getString("core.salesmanager.catalog.url"));
		return chk.toString();
	}

	public static String buildCheckoutToCartUrl(MerchantStore store) {
		StringBuffer chk = new StringBuffer();
		chk
				.append(getSecureDomain(store))
				.append(
						(String) conf
								.getString("core.salesmanager.catalog.url"))
				.append(
						(String) conf
								.getString("core.salesmanager.checkout.checkoutAction"));
		return chk.toString();
	}

}



```
