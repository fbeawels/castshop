# FbAnonymousContent.java

## Review

## 1. Summary  
**Purpose**  
`FbAnonymousContent` is a Spring‑managed component (`@Component("anonymouscontent")`) that implements the `PortletModule` interface.  Its job is to inject a label into a list of “anonymous content” titles whenever a Facebook visitor who has **not** liked the associated Facebook page accesses the page.

**Key Components**  
| Component | Role |
|-----------|------|
| `display(...)` | Executes during page rendering. Determines if the user is a fan, retrieves the appropriate field and appends it to the label list. |
| `requiresAuthorization()` | Declares that no user authentication is required for this module. |
| `submit(...)` | Placeholder for form‑submission logic (currently empty). |

**Frameworks / Libraries**  
* Spring (`@Component`) – for dependency injection.  
* Custom e‑commerce core classes (`MerchantStore`, `Field`, `PortletModule`, `PageExecutionContext`, `PageRequestAction`, `FacebookUser`).  
* Standard Java EE (`HttpServletRequest`, `Locale`, `List`, `Map`).

The code is intentionally lightweight; it only touches the `PageExecutionContext` to read/write data.

---

## 2. Detailed Description  
### Flow of Execution  
1. **Invocation** – When a page is rendered, the CMS or e‑commerce framework calls `display()` on the module.  
2. **Facebook User Retrieval** – The module pulls a `FacebookUser` instance from the execution context (`pageContext.getFromExecutionContext("facebookUser")`).  
3. **Fan Check** – If `user.isLikesPage()` returns `false`, the user is not a fan.  
4. **Field Retrieval** – The module fetches a `Map` named `"fields"` from the context, pulls the `Field` whose key is `"anonymousContentLabelTitle"`.  
5. **Label Augmentation** – It then obtains a `List` named `"labelTitles"` and appends the field’s value.  
6. **No Authorization Needed** – `requiresAuthorization()` returns `false`, so the framework bypasses any login requirement.  
7. **Submit Stub** – `submit()` is a no‑op; the module does not process form data.

### Assumptions & Constraints  
* The context keys `"facebookUser"`, `"fields"`, and `"labelTitles"` must be present; otherwise a `NullPointerException` will occur.  
* The `"fields"` map is expected to contain a `Field` object under the key `"anonymousContentLabelTitle"`.  
* The list of labels (`labelTitles`) is modifiable; the code assumes that adding to it is the desired behaviour.  
* No error handling or logging is performed – failures are silent.

### Architecture & Design Choices  
* **Statelessness** – The module does not maintain any state; it operates purely on context objects.  
* **Dependency Injection** – Declared as a Spring bean, making it easy to wire into a larger application.  
* **Interface‑Driven** – By implementing `PortletModule`, the class conforms to a contract that the CMS expects for modular content blocks.  
* **Raw Types** – The code uses raw `Map` and `List` types, which is discouraged in modern Java (use generics).  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `display(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Renders module content. If the Facebook user is not a fan, adds a label to the page context. | `store` – merchant context<br>`request` – HTTP request (unused)<br>`locale` – user locale (unused)<br>`action` – request action (unused)<br>`pageContext` – execution context | `void` | Mutates the `labelTitles` list inside `pageContext`. |
| `requiresAuthorization()` | Declares that the module does **not** require an authenticated user. | None | `boolean` (always `false`) | None |
| `submit(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Placeholder for form submission logic (currently no operation). | Same as `display` | `void` | None |

**Reusable / Utility Methods** – None; the class is intentionally small.

---

## 4. Dependencies  

| External / Third‑Party | Package / Class | Comments |
|------------------------|-----------------|----------|
| Spring | `org.springframework.stereotype.Component` | Annotation for component scanning. |
| Java EE | `javax.servlet.http.HttpServletRequest` | Standard servlet request. |
| Java Core | `java.util.*`, `java.util.Locale` | Basic collections and locale. |
| SalesManager Core | `com.salesmanager.core.entity.merchant.MerchantStore`<br>`com.salesmanager.core.entity.system.Field`<br>`com.salesmanager.core.module.model.integration.PortletModule`<br>`com.salesmanager.core.util.www.PageExecutionContext`<br>`com.salesmanager.core.util.www.PageRequestAction`<br>`com.salesmanager.core.util.www.integration.fb.FacebookUser` | Domain‑specific classes that provide context, field data, and Facebook integration. |

All dependencies are part of the **SalesManager** e‑commerce platform, with Spring as the only external framework.

---

## 5. Additional Notes  

### Strengths  
* **Simplicity** – The module performs a single, well‑defined task.  
* **Framework‑compliant** – Implements the expected `PortletModule` interface.  
* **No security overhead** – Explicitly declares that no authorization is required.  

### Weaknesses / Edge Cases  
1. **Null‑Pointer Risks** – If any context key is missing, the module will throw a `NullPointerException`.  
2. **Raw Types** – Using raw `Map` and `List` bypasses compile‑time type safety.  
3. **Silent Failure** – There is no logging or exception handling; errors are swallowed silently.  
4. **Concurrency** – If the `labelTitles` list is shared across threads, concurrent modifications could lead to `ConcurrentModificationException`.  
5. **Incomplete Implementation** – The `submit()` method is empty; if the module is ever used in a form submission scenario, it will silently ignore data.  

### Recommendations  
* **Add Null Checks & Logging** – Guard against missing context entries and log useful diagnostics.  
* **Use Generics** – Replace raw types with `Map<String, Field>` and `List<String>` for safety.  
* **Document Context Keys** – Provide a Javadoc or constants for the context key names to avoid typos.  
* **Consider Thread‑Safety** – If the context is shared, use thread‑safe collections or synchronize appropriately.  
* **Implement `submit()` or Throw `UnsupportedOperationException`** – Explicitly state that submission is not supported if that is the intention.  
* **Unit Tests** – Write tests that mock `PageExecutionContext`, supply a dummy `FacebookUser`, and verify that the label list is updated only when appropriate.  

### Future Enhancements  
* **Dynamic Field Key** – Allow the field key (`"anonymousContentLabelTitle"`) to be configurable via the context or a property file.  
* **Internationalization** – The module currently ignores `locale`; consider fetching localized labels from the `Field` object.  
* **Cache or Session Management** – Cache the fan status per user to reduce repeated checks.  
* **Expose API** – Provide a public method that encapsulates the logic for easier reuse in other modules.  

Overall, the class is functional but would benefit from defensive programming, type safety, and better documentation to make it robust in a production environment.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Jan 12, 2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.integration.modules;

import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.springframework.stereotype.Component;

import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.module.model.integration.PortletModule;
import com.salesmanager.core.util.www.PageExecutionContext;
import com.salesmanager.core.util.www.PageRequestAction;
import com.salesmanager.core.util.www.integration.fb.FacebookUser;

/**
 * Content will be displayed only to non fan visitors of the page
 * @author Carl Samson
 *
 */
@Component("anonymouscontent")
public class FbAnonymousContent implements PortletModule {

	public void display(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		// TODO Auto-generated method stub
		
		//get facebook user
		FacebookUser user = (FacebookUser)pageContext.getFromExecutionContext("facebookUser");
		if(!user.isLikesPage()) {
			
			//get label id
			Map fields = (Map)pageContext.getFromExecutionContext("fields");
			
			//get field by fieldName
			Field field = (Field)fields.get("anonymousContentLabelTitle");
			
			//get list of labels
			List labels = (List)pageContext.getFromExecutionContext("labelTitles");
			labels.add(field.getFieldValue());
			
		}

	}

	public boolean requiresAuthorization() {
		// TODO Auto-generated method stub
		return false;
	}

	public void submit(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		// TODO Auto-generated method stub

	}

}



```
