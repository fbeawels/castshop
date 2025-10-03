# ContactAction.java

## Review

## 1. Summary  

`ContactAction` is a Struts‑style action that prepares the data required to render a *Contact Us* page for a merchant’s storefront.  

* **Key responsibilities**
  * Load the current `MerchantStore` from the session.
  * Retrieve configuration settings that control whether a physical address and/or a Google Map are shown.
  * Fetch a localized description (dynamic label) that is displayed on the page.
  * Expose three properties (`displayAddress`, `displayMap`, `description`) to the view layer via getters/setters.

* **Notable design patterns / frameworks**
  * **Action/Command pattern** – the `display()` method acts as the command that prepares the view.
  * **Factory pattern** – `ServiceFactory` is used to obtain service instances (`MerchantService`, `ReferenceService`).
  * **Singleton pattern** (implied by the service factory) – services are probably shared across the application.
  * **MVC** – The action populates request attributes and forwards to a view (JSP/Velocity/etc.) using the standard Struts flow.

* **Libraries**
  * Apache `log4j` for logging.
  * Custom `ServiceFactory`, `MerchantService`, `ReferenceService`, etc., from the SalesManager core layer.

---

## 2. Detailed Description  

### Execution Flow  
1. **MerchantStore resolution**  
   ```java
   MerchantStore store = SessionUtil.getMerchantStore(super.getServletRequest());
   ```
   The helper pulls the merchant context from the current HTTP session.

2. **Populate request attribute**  
   `paageId` (likely a typo for `pageId`) is set to `"contact"`.  This is presumably used by the view to select the correct navigation or layout fragment.

3. **Configuration loading**  
   * A `MerchantService` instance is fetched from the `ServiceFactory`.
   * A `ConfigurationRequest` is built for the current merchant ID and the `CONTACTUS` configuration key.
   * The response is examined for a `MerchantConfiguration` object.  
     * If the `configurationValue1` flag is `"true"`, a Google Map will be displayed (`displayMap = true`).  
     * If the `configurationValue` flag is `"false"`, the address block will be hidden (`displayAddress = false`).

4. **Description loading**  
   * A `ReferenceService` instance is obtained.
   * The `LabelConstants.STORE_FRONT_CONTACT_US` dynamic label is fetched for the current locale.
   * If present, the label text is stored in the `description` field.

5. **Error handling** – Any exception is caught and logged; no further error propagation occurs. The method always returns `SUCCESS`, signalling the framework to forward to the default success view.

### Dependencies & Constraints  
* **Assumptions**  
  * The `SessionUtil.getMerchantStore()` call must never return `null`; otherwise a `NullPointerException` would be thrown.
  * The configuration values are expected to be either `"true"`/`"false"` strings; any other value is ignored.
  * The `ReferenceService` will return a non‑`null` `DynamicLabelDescription` only if the label has been configured; otherwise `description` remains `null`.

* **Lifecycle**  
  * The action is short‑lived (one request).  All state is stored in instance fields that are reset on each new action invocation.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `display()` | Struts entry point; populates page data | None (uses HTTP request/session) | `String` – `"SUCCESS"` | Sets request attributes, updates instance fields, logs errors |
| `isDisplayAddress()` | Getter for `displayAddress` | None | `boolean` | None |
| `setDisplayAddress(boolean)` | Setter for `displayAddress` | `boolean` | None | Updates field |
| `isDisplayMap()` | Getter for `displayMap` | None | `boolean` | None |
| `setDisplayMap(boolean)` | Setter for `displayMap` | `boolean` | None | Updates field |
| `getDescription()` | Getter for `description` | None | `String` | None |
| `setDescription(String)` | Setter for `description` | `String` | None | Updates field |

The only non‑trivial method is `display()`. All other methods are standard JavaBean accessors used by the view layer.

---

## 4. Dependencies  

| Library/Component | Type | Notes |
|-------------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Standard logging library. |
| `com.salesmanager.common.SalesManagerBaseAction` | Internal | Base Struts action providing utility methods (`getServletRequest()`, `getLocale()`). |
| `com.salesmanager.core.constants.ConfigurationConstants` | Internal | Holds static keys for merchant configuration. |
| `com.salesmanager.core.constants.LabelConstants` | Internal | Holds static keys for dynamic labels. |
| `com.salesmanager.core.entity.merchant.*` | Internal | JPA/Hibernate entities for merchant configuration and store. |
| `com.salesmanager.core.service.*` | Internal | Service layer (MerchantService, ReferenceService). |
| `com.salesmanager.core.util.www.SessionUtil` | Internal | Helper to retrieve merchant store from the session. |
| `ServiceFactory` | Internal | Factory for obtaining service instances (likely a Spring bean factory or custom DI). |

No external, platform‑specific dependencies are present beyond the standard Java EE container.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation of concerns** – The action only orchestrates service calls; all business logic remains in the service layer.  
* **Use of constants** – Reduces magic strings and potential typos.  
* **Graceful error handling** – Exceptions are logged, preventing the application from crashing.  

### Areas for Improvement  

1. **Error Handling**  
   * Swallowing all exceptions and returning `SUCCESS` may mask real problems (e.g., missing configuration). Consider returning an error view or propagating a checked exception.

2. **Typo in request attribute**  
   * `"paageId"` should probably be `"pageId"`.  A typo may cause the view logic to fail silently.

3. **Hard‑coded string comparison**  
   * `equalsIgnoreCase("true")`/`"false"` could be replaced with a dedicated enum or boolean helper to avoid repeated string literals and improve readability.

4. **Null‑safety**  
   * The code assumes `store`, `conf`, and `label` are non‑null. Defensive checks or null‑safety utilities would make the action more robust.

5. **Method name**  
   * The action method is named `display()`; Struts typically expects `execute()` or a name derived from the action configuration. Verify that the mapping in `struts-config.xml` (or annotations) points to this method.

6. **Internationalization**  
   * `description` is loaded via `getLocale()`, but the action does not provide a fallback if the label is missing. A default message could improve user experience.

7. **Testing**  
   * The action is stateful (fields), which can make unit testing more cumbersome. A stateless design or explicit request-scoped beans could simplify testing.

### Future Enhancements  

* **Caching** – The configuration and label values are fetched on every request; caching per merchant could reduce load on the service layer.  
* **Parameterization** – Expose the configuration keys as parameters to the action so that the same action could be reused for different page types.  
* **Security** – Validate that the user has permission to view the contact page before loading configuration.

Overall, `ContactAction` is a straightforward, maintainable component that follows common enterprise Java patterns, with room for minor refinements in robustness and clarity.

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
package com.salesmanager.catalog.store;

import org.apache.log4j.Logger;

import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.www.SessionUtil;

public class ContactAction extends SalesManagerBaseAction {

	private Logger log = Logger.getLogger(ContactAction.class);

	private boolean displayAddress = true;
	private boolean displayMap = false;
	private String description = null;

	public String display() {

		MerchantStore store = SessionUtil.getMerchantStore(super
				.getServletRequest());

		try {

			super.getServletRequest().setAttribute("paageId", "contact");
			// get contact us properties & map
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationRequest request = new ConfigurationRequest(store
					.getMerchantId(), ConfigurationConstants.CONTACTUS);
			ConfigurationResponse response = mservice.getConfiguration(request);

			MerchantConfiguration conf = response
					.getMerchantConfiguration(ConfigurationConstants.CONTACTUS);

			if (conf != null) {

				// display google map
				String mapConf = conf.getConfigurationValue1();
				if (mapConf != null && mapConf.equalsIgnoreCase("true")) {
					this.setDisplayMap(true);
				}

				// display custom address
				String basicConf = conf.getConfigurationValue();
				if (basicConf != null && basicConf.equalsIgnoreCase("false")) {
					this.setDisplayAddress(false);
				}

			}

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			DynamicLabelDescription label = rservice
					.getDynamicLabelDescription(store.getMerchantId(),
							LabelConstants.STORE_FRONT_CONTACT_US, super
									.getLocale());
			if (label != null) {
				description = label.getDynamicLabelDescription();
			}

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public boolean isDisplayAddress() {
		return displayAddress;
	}

	public void setDisplayAddress(boolean displayAddress) {
		this.displayAddress = displayAddress;
	}

	public boolean isDisplayMap() {
		return displayMap;
	}

	public void setDisplayMap(boolean displayMap) {
		this.displayMap = displayMap;
	}

	public String getDescription() {
		return description;
	}

	public void setDescription(String description) {
		this.description = description;
	}

}



```
