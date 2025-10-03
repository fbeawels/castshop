# URLInterceptor.java

## Review

## 1. Summary  
The **`URLInterceptor`** is a Struts 2‐style interceptor that processes SEO‑friendly URLs of the form:

```
/category/<category-slug>
/product/<product-slug>
/pagelink/<page-slug>
```

Its sole responsibility is to extract the slug (the “requested entity id”) from the request URI and set it on the current action (`SalesManagerBaseAction`) via `setRequestedEntityId`.  The class extends a custom `SalesManagerInterceptor` and overrides `baseIntercept`, which is called by the framework during request processing.

The implementation is straightforward, using only standard Java EE (`HttpServletRequest/Response`), Apache Commons Lang (`StringUtils`), and the Struts 2 `ActionInvocation` API. No additional frameworks or design patterns beyond the typical interceptor pattern are employed.

---

## 2. Detailed Description  
1. **URI Normalisation**  
   - `req.getRequestURI()` returns the full request path (e.g. `/shop/category/electronics`).  
   - `StringUtils.removeStart(uri, contextPath)` strips the context path (`/shop`) leaving `/category/electronics`.  
   - `path.substring(1)` removes the leading slash, yielding `category/electronics`.

2. **Path Tokenisation**  
   - The path is split on `/`, producing an array where `parameters[0]` is the resource type (`category`, `product`, or `pagelink`) and `parameters[1]` is the slug (which may still contain a query string or `.action` suffix).

3. **Validation & Error Handling**  
   - If the split array is `null` or has fewer than two elements, an `Exception` is thrown – signalling a malformed URL.

4. **Slug Cleaning**  
   - Any trailing `.action` fragment is removed. (Note: query parameters are **not** stripped.)

5. **Action Hook**  
   - The current action (`invoke.getAction()`) is cast to `SalesManagerBaseAction`.  
   - The extracted slug is passed to `action.setRequestedEntityId(id)`.

6. **Return Value**  
   - The method returns `null`, allowing the normal Struts2 execution flow to continue.

The interceptor makes no use of the `HttpServletResponse` object and performs no cleanup. It purely transforms the request URI into a form consumable by the underlying action.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `baseIntercept(ActionInvocation invoke, HttpServletRequest req, HttpServletResponse resp)` | Intercepts SEO URLs, extracts the entity slug, and stores it on the action. | `invoke` – current action invocation; `req` – the HTTP request; `resp` – the HTTP response (unused). | `String` – `null` to continue processing. | Sets `requestedEntityId` on the action; may throw `Exception` on malformed URL. |

### Utility / Reusable Pieces  
- None beyond the standard string manipulation performed in `baseIntercept`.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest/HttpServletResponse` | Standard | Servlet API |
| `org.apache.commons.lang.StringUtils` | Third‑party | Provides `removeStart`. Uses Apache Commons Lang 2.x (`org.apache.commons.lang`), which is now deprecated in favour of `org.apache.commons.lang3`. |
| `com.opensymphony.xwork2.ActionInvocation` | Third‑party | Struts 2 core API. |
| `com.salesmanager.common.SalesManagerBaseAction` | Internal | Base action class that exposes `setRequestedEntityId`. |
| `com.salesmanager.core.util.www.SalesManagerInterceptor` | Internal | Custom interceptor base class. |

No platform‑specific dependencies beyond the servlet container and Struts 2.

---

## 5. Additional Notes  

### Strengths  
- **Simplicity** – The interceptor does a single, well‑defined job.  
- **Reusability** – By delegating to `SalesManagerBaseAction`, any action can benefit from this URL parsing logic.  
- **Framework Integration** – Extends the custom `SalesManagerInterceptor`, so it can be wired into the existing Struts 2 filter chain.

### Potential Issues & Edge Cases  
1. **Query String Leakage**  
   - The current implementation does **not** strip query parameters (`?foo=bar`).  
   - If a request is `/product/123?color=red`, `id` will be `"123?color=red"`, which is likely unintended.  
   - Suggested fix: split on `"?"` and take the first part before setting the entity id.

2. **Trailing Slash / Empty Segment**  
   - A request such as `/product/` or `/product` will produce `parameters[1]` as an empty string, causing the validator to throw an exception.  
   - Consider normalising the path (e.g., trim trailing slashes) before splitting.

3. **Non‑Numeric / Non‑Slug IDs**  
   - The code assumes the slug uniquely identifies the entity. If the slug can contain slashes or special characters, the split logic may fail.  
   - A more robust parser could use a regex or a dedicated URL router.

4. **`StringUtils.removeStart` Deprecation**  
   - The older `org.apache.commons.lang.StringUtils` is deprecated. Switching to `org.apache.commons.lang3.StringUtils` would future‑proof the code.

5. **`ActionInvocation` Casting**  
   - The cast to `SalesManagerBaseAction` assumes all actions in this interceptor’s scope extend that class. If a non‑conforming action is invoked, a `ClassCastException` will be thrown.  
   - Defensive coding (instanceof check) would improve robustness.

6. **Exception Handling**  
   - The method throws a generic `Exception` for malformed URLs.  
   - It would be better to throw a more specific exception (e.g., `IllegalArgumentException`) or return a Struts2 error result to provide a user‑friendly error page.

7. **Missing Use of `HttpServletResponse`**  
   - The parameter is unused; consider removing it to clean up the signature unless the base class requires it.

### Future Enhancements  
- **Robust URL Parser** – Replace manual string manipulation with a URL router that can handle various patterns, optional query parameters, and locale prefixes.  
- **Configuration‑Driven** – Allow the interceptor to read the supported URL prefixes (`/category`, `/product`, `/pagelink`) from a config file rather than hardcoding them.  
- **Logging** – Add diagnostic logging for malformed URLs to aid debugging.  
- **Unit Tests** – Provide a comprehensive test suite covering typical, edge, and invalid URLs.  
- **Internationalisation Support** – If slugs can contain non‑ASCII characters, ensure proper URL decoding.

Overall, the interceptor fulfills its intended purpose but would benefit from defensive coding and a few enhancements to handle real‑world URL patterns more gracefully.

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
package com.salesmanager.catalog;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang.StringUtils;

import com.opensymphony.xwork2.ActionInvocation;
import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.util.www.SalesManagerInterceptor;

public class URLInterceptor extends SalesManagerInterceptor {

	@Override
	protected String baseIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception {
		// TODO Auto-generated method stub

		/**
		 * This interceptor is used for urls invoked using SEO type urls The
		 * system is configured to support /category/category-name
		 * /product/product-name /pagelink/page-name
		 */

		// get the product from the path
		String path = StringUtils.removeStart(req.getRequestURI(), req
				.getContextPath());

		/**
		 * should have left /category/<PRODUCT-URL>?request parameters or should
		 * have left /pagelink/<PAGE-URL>?request parameters or should have left
		 * /product/<PRODUCT-URL>?request parameters keep after second / and
		 * before ?
		 */

		// remove /product or /category or /page
		String pathnocontext = path.substring(1);// remove /

		// should now have product/<PRODUCT-ID>?request parameters or
		// category/<CATEGORY-ID>?request parameters or pagelink/<PAGE-ID>
		String[] parameters = pathnocontext.split("/");

		// should now have
		// parameters[0] = product or category or pagelink
		// parameters[1] = <PRODUCT-ID>?parameters

		if (parameters == null || parameters.length == 1) {
			throw new Exception("Invalid parameters in request " + path);
		}

		String id = parameters[1];
		
		if(id.contains(".action")) {
			id = id.substring(0,id.indexOf(".action"));
		}

		SalesManagerBaseAction action = ((SalesManagerBaseAction) invoke
				.getAction());

		action.setRequestedEntityId(id);

		return null;

	}

}



```
