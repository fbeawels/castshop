# UrlTag.java

## Review

## 1. Summary  

**Purpose**  
`UrlTag` is a custom Struts‑2 tag that extends `org.apache.struts2.views.jsp.URLTag`. Its primary goal is to inject the current merchant’s identifier (`merchantId`) into every generated URL when the application configuration allows it. It also adds a small amount of logic around the scheme selection.

**Key Components**  
| Component | Role |
|-----------|------|
| `UrlTag` | The tag implementation that overrides `populateParams()` to enrich URL parameters. |
| `URL` (Struts component) | The underlying Struts2 URL component that actually builds the URL string. |
| `PropertiesUtil` / `Configuration` | Reads application configuration (`core.url.usemerchantid`, `core.domain.http.secure`). |
| `SessionUtil` / `HttpServletRequest` | Retrieves the `MerchantStore` object stored in the request scope. |

**Design Patterns & Frameworks**  
- **Decorator / Extension Pattern** – The tag extends the framework class to alter its behaviour.  
- **Factory/Utility Pattern** – `PropertiesUtil.getConfiguration()` provides a singleton configuration instance.  
- **MVC / Tag Library** – Built on top of Struts‑2’s tag library, part of a typical JSP MVC stack.  

---

## 2. Detailed Description  

### Flow of Execution  
1. **Tag Invocation** – When the JSP engine encounters `<sm:url>` (or the tag’s alias), it creates an instance of `UrlTag`.  
2. **Component Creation** – `URLTag` creates a `URL` component during its initialization (handled by the superclass).  
3. **Parameter Population** – `populateParams()` is invoked automatically by the Struts tag framework.  
4. **Custom Logic**  
   - The method calls `super.populateParams()` first, so the base behaviour (setting scheme, action, etc.) remains intact.  
   - It retrieves the current `HttpServletRequest` from the tag’s `pageContext`.  
   - From the request, it pulls the `MerchantStore` attribute (expected to be set earlier in the request lifecycle).  
   - If the configuration flag `core.url.usemerchantid` is true, it inserts the `merchantId` into the URL’s parameter map.  
   - It also normalises the scheme when `scheme` is explicitly set to `"https"`, mapping it to the configured secure domain.  
5. **Component Finalisation** – All remaining attributes (includeParams, encode, etc.) are forwarded to the underlying `URL` component.  
6. **URL Generation** – The component then produces the final URL string, which is written to the JSP output.

### Assumptions & Constraints  
- The tag expects a `MerchantStore` instance to be present in the request with key `"STORE"`.  
- Configuration is read once and cached statically.  
- No null‑safety is provided for `pageContext.getRequest()` or the request attributes beyond a simple null check on `store`.  
- The tag is only safe in a servlet/JSP environment (no support for other containers).  
- The tag relies on Struts‑2 internals (e.g., `URL` component) and `org.apache.commons.configuration` for config reading.  

### Architectural Observations  
- The tag keeps the original Struts‑2 URL generation logic intact, only augmenting it. This is a lightweight, maintainable approach.  
- However, the code mixes **business logic** (retrieving `MerchantStore`) with **view logic** (tag rendering), violating separation of concerns.  
- The static `Configuration` reference could become a memory leak if the tag is re‑instantiated with a different configuration in a multi‑tenant environment.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `populateParams()` | Overridden Struts method that injects `merchantId` into the URL and normalises scheme. | None (uses tag fields) | None (sets properties on `URL` component) | Modifies the URL component’s internal state, writes to request attribute cache. |
| (Inherited) `URLTag` constructor | Instantiates the underlying `URL` component. | None | None | Creates component instance. |
| (Inherited) `doStartTag()` / `doEndTag()` | Executes the tag during page rendering. | None | `SKIP_BODY` or `EVAL_BODY_INCLUDE` | Invokes `populateParams()`, writes URL to output. |

> **Note** – No other public methods are defined in this class.

---

## 4. Dependencies  

| Library | Type | Purpose |
|---------|------|---------|
| **Struts‑2** (`org.apache.struts2.*`) | Third‑party | Provides base tag (`URLTag`) and URL component. |
| **Apache Commons Lang** (`org.apache.commons.lang.StringUtils`) | Third‑party | String utilities. |
| **Apache Commons Configuration** (`org.apache.commons.configuration.Configuration`) | Third‑party | Reads application properties. |
| **Apache Log4j** (`org.apache.log4j.Logger`) | Third‑party | Logging (currently unused). |
| **Spring** (`com.salesmanager.core.util.SpringUtil`) | Third‑party | Likely used elsewhere in the project; not directly used in this class. |
| **SalesManager Core** (`com.salesmanager.*`) | Project-specific | Holds constants, entities, services, utilities. |
| **Servlet API** (`javax.servlet.*`) | Standard | Request/response/session handling. |
| **JSP API** (`javax.servlet.jsp.*`) | Standard | Tag lifecycle management. |

> The class is tightly coupled to the servlet/JSP environment and Struts‑2 tag lifecycle; it is not platform‑agnostic.

---

## 5. Additional Notes & Recommendations  

### Strengths  
- **Simplicity** – Minimal code changes to a well‑known component.  
- **Reusability** – Once registered as a tag, it can be used everywhere the base URL tag is used.  
- **Configurability** – External flag controls whether the `merchantId` is added.

### Weaknesses / Risks  

1. **Raw Types & Generics**  
   - `Map parameters` is a raw type. Use `Map<String, Object>` to avoid unchecked warnings.  
   - The cast `(URL) component` assumes the component is always a `URL`; a safer check (`instanceof`) could guard against misuse.

2. **Null‑Safety**  
   - The request or the `STORE` attribute could be null in edge cases (e.g., AJAX calls without the attribute).  
   - The method silently ignores missing `store`, but subsequent code may rely on the presence of `merchantId`.  

3. **Hardcoded Attribute Key**  
   - The key `"STORE"` is a magic string. It would be clearer to use a constant (`Constants.REQUEST_ATTRIBUTE_STORE`).

4. **Unutilised Logger**  
   - `log` is defined but never used. Add debug/logging for missing store, configuration issues, or URL construction failures.

5. **Configuration Caching**  
   - The static `config` reference is read once at class load time. If the application supports hot‑reloading or multi‑tenant configs, this could cause stale data. Consider retrieving the configuration lazily or via dependency injection.

6. **Separation of Concerns**  
   - Business logic (obtaining `MerchantStore`) should ideally reside in a service or interceptor, not in a tag. The tag should purely be a view helper.

7. **Thread‑Safety**  
   - `pageContext` is per‑request, so the tag instance is not shared between threads. However, the static `config` is thread‑safe only if the underlying `Configuration` implementation is thread‑safe (Commons Configuration usually is, but worth documenting).

8. **Deprecated or Unused Code**  
   - The class imports `TagSupport` but never uses it. Remove unused imports.

9. **Testing & Maintainability**  
   - The logic is embedded in a tag, making unit testing harder. Extract the URL building into a plain Java helper that can be tested independently.

10. **Potential for URL Injection**  
    - Directly injecting `merchantId` into the URL might expose sensitive data if the URL is cached or logged. Ensure proper encoding and validation.

### Suggested Enhancements  

| Area | Suggested Change | Benefit |
|------|------------------|---------|
| **Generics** | Replace raw `Map` with `Map<String,Object>` | Removes unchecked warnings and clarifies key/value types. |
| **Constants** | Use a constant for the `"STORE"` attribute | Avoids typos and centralises change. |
| **Logging** | Add debug logs for missing store, config value, and final URL | Improves troubleshooting. |
| **Separation** | Move merchant retrieval to a `TagHandler` or interceptor; keep tag purely formatting | Cleaner architecture. |
| **Configuration** | Lazy load or inject configuration, or allow reloading on demand | Supports dynamic config changes. |
| **Null Handling** | Explicitly handle null request or store, possibly throwing an informative exception | Avoids silent failures. |
| **Clean‑up** | Remove unused imports, update documentation | Keeps code tidy. |

---

### Quick Code Fix Example (Generics & Constants)

```java
public class UrlTag extends URLTag {

    private static final Logger log = Logger.getLogger(UrlTag.class);
    private static final String STORE_ATTRIBUTE = "STORE";

    protected void populateParams() {
        super.populateParams();

        HttpServletRequest req = (HttpServletRequest) pageContext.getRequest();
        MerchantStore store = (MerchantStore) req.getAttribute(STORE_ATTRIBUTE);

        URL url = (URL) component;
        Map<String, Object> parameters = url.getParameters();
        if (parameters == null) {
            parameters = new LinkedHashMap<>();
        }

        if (store != null) {
            boolean useMerchantId = config.getBoolean("core.url.usemerchantid", false);
            if (useMerchantId) {
                parameters.put("merchantId", store.getMerchantId());
            }
        } else {
            log.warn("MerchantStore attribute [" + STORE_ATTRIBUTE + "] missing in request");
        }

        // … rest of the logic …
    }
}
```

---  

**Overall Verdict**  
The tag achieves its goal with minimal code, but several small issues (raw types, unused logger, tight coupling) reduce maintainability and clarity. Addressing the recommendations above would make the component safer, more testable, and easier to evolve as the application grows.

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
package com.salesmanager.core.util.www.tags;

import java.io.IOException;
import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.PageContext;
import javax.servlet.jsp.tagext.SimpleTagSupport;
import javax.servlet.jsp.tagext.TagSupport;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.components.Component;
import org.apache.struts2.components.URL;
import org.apache.struts2.views.jsp.URLTag;

import com.opensymphony.xwork2.util.ValueStack;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class UrlTag extends URLTag {

	private Logger log = Logger.getLogger(UrlTag.class);

	private static org.apache.commons.configuration.Configuration config = PropertiesUtil
			.getConfiguration();

	/**
	 * Overwrites default struts 2 tag in order to add merchantId in url
	 * parameters if configured in the system
	 */
	protected void populateParams() {

		super.populateParams();

		HttpServletRequest req = (HttpServletRequest) pageContext.getRequest();

		MerchantStore store = (MerchantStore) req.getAttribute("STORE");

		URL url = (URL) component;

		Map parameters = url.getParameters();
		if (parameters == null) {
			parameters = new LinkedHashMap();
		}

		if (store != null) {

			// check if merchantId has to be set in the url
			boolean useMerchantId = config.getBoolean("core.url.usemerchantid",
					false);
			if (useMerchantId) {
				parameters.put("merchantId", store.getMerchantId());
			}

		}

		if (!StringUtils.isBlank(scheme)) {
			if (("https").equalsIgnoreCase(scheme)) {
				scheme = (String) config.getString("core.domain.http.secure",
						"https");
			}
		}

		url.setIncludeParams(includeParams);
		url.setScheme(scheme);
		url.setValue(value);
		url.setMethod(method);
		url.setNamespace(namespace);
		url.setAction(action);
		url.setPortletMode(portletMode);
		url.setPortletUrlType(portletUrlType);
		url.setWindowState(windowState);
		url.setAnchor(anchor);

		if (encode != null) {
			url.setEncode(Boolean.valueOf(encode).booleanValue());
		}
		if (includeContext != null) {
			url.setIncludeContext(Boolean.valueOf(includeContext)
					.booleanValue());
		}
		if (escapeAmp != null) {
			url.setEscapeAmp(Boolean.valueOf(escapeAmp).booleanValue());
		}
		if (forceAddSchemeHostAndPort != null) {
			url.setForceAddSchemeHostAndPort(Boolean.valueOf(
					forceAddSchemeHostAndPort).booleanValue());
		}

	}

}



```
