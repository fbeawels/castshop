# IntegrationKeysAction.java

## Review

## 1. Summary  

`IntegrationKeysAction` is a Struts‑style action class that manages the configuration of third‑party integration keys (currently Google Analytics and, historically, Facebook).  
* **Purpose** – Allow a merchant to view and edit its Google Analytics integration credentials (API key and tracking ID).  
* **Key components**  
  * `displayPage()` – loads existing configuration and populates the UI.  
  * `editConfigurationKeys()` – validates and persists changes (or deletes when both fields are blank).  
  * Getter/setters for the two configuration fields.  
* **Design patterns / frameworks**  
  * **Service Factory** – retrieves a `MerchantService` instance.  
  * **Configuration Request/Response** – a simple request/response DTO pair for configuration lookup.  
  * **Apache Commons Lang** – `StringUtils.isBlank()` for null/empty checks.  
  * **Log4j** – basic logging (unused in the current code).  
  * Inherited `BaseAction` likely provides Struts helper methods (`setPageTitle`, `getContext`, `setSuccessMessage`).  

---

## 2. Detailed Description  

### Flow of Execution  

| Phase | Method | Key Actions |
|-------|--------|-------------|
| **Initialization** | – | `IntegrationKeysAction` is instantiated by the Struts framework with the merchant’s session context. |
| **Display** | `displayPage()` | 1. Set page title. <br> 2. Retrieve `MerchantService`. <br> 3. Build a `ConfigurationRequest` for `G_API`. <br> 4. Call `getConfiguration()` to fetch existing values. <br> 5. If found, store the Google API key (`analytics`) and tracking ID (`googleapi`). |
| **Edit** | `editConfigurationKeys()` | 1. Retrieve context and service. <br> 2. Fetch existing `G_API` configuration. <br> 3. **Create / Update** – if either field is non‑blank, create a new `MerchantConfiguration` or reuse the existing one, set values, and persist via `saveOrUpdateMerchantConfiguration`. <br> 4. **Delete / Clean‑up** – if both fields are blank, delete the configuration record. <br> 5. If only one field is blank, clear that value but keep the other. <br> 6. Update the merchant context (`ctx.setGcode`) when the Google tracking ID is present. <br> 7. Set a success message and return `SUCCESS`. |
| **Cleanup** | – | No explicit cleanup; Struts/Servlet container handles the request lifecycle. |

### Assumptions & Constraints  

* The merchant’s `Context` is already populated and contains a valid `merchantid`.  
* The `MerchantService` implementation correctly persists `MerchantConfiguration` entities.  
* Only Google Analytics integration is active; Facebook fields are commented out.  
* No input validation beyond `StringUtils.isBlank` – no format checks (e.g., valid API key).  

### Architectural Choices  

* **Separation of concerns** – the action delegates persistence to `MerchantService`.  
* **Explicit null/blank handling** – ensures that partial updates are possible without deleting the whole configuration.  
* **Redundant code** – the commented Facebook block shows legacy code that could be refactored into a reusable helper method.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `displayPage()` | Loads existing Google configuration and prepares the UI. | None | `String` (`SUCCESS` or error) | Sets page title, populates `analytics` & `googleapi`. |
| `editConfigurationKeys()` | Validates and persists updates to Google configuration. | None (uses instance fields) | `String` (`SUCCESS` or error) | Creates/updates/deletes `MerchantConfiguration`, updates `Context`, sets success message. |
| `getGoogleapi()` | Getter for the Google tracking ID. | None | `String` | None |
| `setGoogleapi(String)` | Setter for the Google tracking ID. | `googleapi` | void | Sets instance field. |
| `getAnalytics()` | Getter for the Google API key. | None | `String` | None |
| `setAnalytics(String)` | Setter for the Google API key. | `analytics` | void | Sets instance field. |

*The class currently does not expose any utility methods; all logic is embedded in the two main actions.*

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.central.BaseAction` | Base class | Likely a Struts `ActionSupport` subclass providing `setPageTitle`, `getContext`, `setSuccessMessage`. |
| `com.salesmanager.central.profile.Context` | Merchant context | Holds merchant ID and cached values (`setGcode`). |
| `com.salesmanager.core.service.merchant.MerchantService` | Service layer | Persists `MerchantConfiguration`. |
| `com.salesmanager.core.service.merchant.ConfigurationRequest/Response` | DTOs | Simple request/response for config lookup. |
| `com.salesmanager.core.constants.ConfigurationConstants` | Constants | Contains keys like `G_API`. |
| `org.apache.commons.lang.StringUtils` | 3rd‑party | Null‑safe string checks. |
| `org.apache.log4j.Logger` | Logging | Instantiated but not used. |
| `java.util.Date` | Standard | For timestamp fields. |
| `java.util` (other classes) | Standard | – |

*All dependencies are either standard JDK classes or part of the SalesManager core library. No external frameworks (e.g., Spring) are required.*

---

## 5. Additional Notes  

### Strengths  
* Clear separation between read and write logic.  
* Handles partial updates gracefully (e.g., only API key present).  
* Uses DTOs and service layer to keep persistence logic out of the action.  

### Weaknesses & Edge Cases  
1. **Redundant / dead code** – the commented Facebook block and unused logger suggest code that could be removed or refactored.  
2. **No input validation** – malformed keys (e.g., empty strings that look like placeholders) could be persisted.  
3. **Concurrency** – simultaneous edits by different sessions could lead to lost updates; no locking or versioning is used.  
4. **Exception handling** – all methods declare `throws Exception` but never catch or log exceptions; failures may surface as generic server errors.  
5. **Hard‑coded constant** – `G_API` is the only key handled; adding another integration would require duplicate code.  
6. **Context caching** – `ctx.setGcode(googleapi)` is called only when a value is present; if the value becomes blank later, it might remain cached.  

### Suggested Enhancements  
* **Refactor to a helper** – create a private method `processConfiguration(String key, String value1, String value2)` to avoid duplicated logic.  
* **Validation** – add regex checks or a validator bean to enforce correct API key formats.  
* **Logging** – enable the logger to trace key changes and errors.  
* **Exception handling** – wrap service calls in try/catch blocks and set meaningful error messages.  
* **Unit tests** – write tests for both `displayPage` and `editConfigurationKeys` covering all branches (create, update, delete).  
* **Internationalization** – consider using message keys for user feedback instead of hard‑coded strings.  
* **Removal of legacy code** – delete the commented Facebook block or move it to a separate, optional module.  

Overall, the class fulfills its core requirement but would benefit from cleanup, stronger validation, and better error handling to improve robustness and maintainability.

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
package com.salesmanager.central.integration;

import java.util.Date;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.invoice.InvoiceDetailsAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;





public class IntegrationKeysAction extends BaseAction {
	
	
	private String googleapi;
	private String analytics;
/*	private String fbkey;
	private String fbsecret;*/
	
	private Logger log = Logger.getLogger(IntegrationKeysAction.class);
	
	
	public String displayPage() throws Exception {
		
		super.setPageTitle("label.menu.function.INTKEYO01");
		
		
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		Context ctx = super.getContext();
		
		// get analytics
		ConfigurationRequest req = new ConfigurationRequest(
				ctx.getMerchantid(), ConfigurationConstants.G_API);
		ConfigurationResponse resp = mservice.getConfiguration(req);

		MerchantConfiguration googleCode = resp
				.getMerchantConfiguration(ConfigurationConstants.G_API);
		
		if (googleCode != null) {
			analytics = googleCode.getConfigurationValue();
			googleapi = googleCode.getConfigurationValue1();
		}
		
/*		req = new ConfigurationRequest(
				ctx.getMerchantid(), ConfigurationConstants.FB_API);
		resp = mservice.getConfiguration(req);
		
		MerchantConfiguration fbCode = resp
		.getMerchantConfiguration(ConfigurationConstants.FB_API);
		
		if (fbCode != null) {
			fbkey = fbCode.getConfigurationValue();
			fbsecret = fbCode.getConfigurationValue1();
		}*/
		
		
		return SUCCESS;
		
	}
	
	public String editConfigurationKeys() throws Exception {
		
		super.setPageTitle("label.menu.function.INTKEYO01");
		
		Context ctx = super.getContext();
		
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		// get analytics
		ConfigurationRequest req = new ConfigurationRequest(
				ctx.getMerchantid(), ConfigurationConstants.G_API);
		ConfigurationResponse resp = mservice.getConfiguration(req);

		MerchantConfiguration googleCode = resp
				.getMerchantConfiguration(ConfigurationConstants.G_API);
		
		// google
		if (!StringUtils.isBlank(analytics) || !StringUtils.isBlank(googleapi)) {
			if (googleCode == null) {
				googleCode = new MerchantConfiguration();
				googleCode
						.setConfigurationKey(ConfigurationConstants.G_API);
				googleCode.setMerchantId(ctx.getMerchantid());
				googleCode.setDateAdded(new Date());
				googleCode.setLastModified(new Date());

			}
			if (!StringUtils.isBlank(analytics)) {
				googleCode.setConfigurationValue(analytics);
			} else {
				googleCode.setConfigurationValue("");
			}

			if (!StringUtils.isBlank(googleapi)) {
				googleCode.setConfigurationValue1(googleapi);
				ctx.setGcode(googleapi);
			} else {
				googleCode.setConfigurationValue1("");
			}

			mservice.saveOrUpdateMerchantConfiguration(googleCode);
		} else if (StringUtils.isBlank(analytics)
				|| StringUtils.isBlank(googleapi)) {

			
			if(StringUtils.isBlank(analytics) && StringUtils.isBlank(googleapi)) {
				
				if (googleCode != null) {
					ctx.setGcode(null);
					mservice.deleteMerchantConfiguration(googleCode);
				}
				
			} else {
				
				if (googleCode != null) {
					if(StringUtils.isBlank(googleapi)) {
						ctx.setGcode(null);
						googleCode.setConfigurationValue1("");
					}
					if(StringUtils.isBlank(analytics)) {
						ctx.setGcode(null);
						googleCode.setConfigurationValue("");
					}
					googleCode.setLastModified(new Date());
					mservice.saveOrUpdateMerchantConfiguration(googleCode);
				}
				
			}
			

		}
		
/*		
		req = new ConfigurationRequest(
				ctx.getMerchantid(), ConfigurationConstants.FB_API);
		resp = mservice.getConfiguration(req);
		
		MerchantConfiguration fbCode = resp
		.getMerchantConfiguration(ConfigurationConstants.FB_API);
		
		
		// fb
		if (!StringUtils.isBlank(fbkey) || !StringUtils.isBlank(fbsecret)) {
			if (fbCode == null) {
				fbCode = new MerchantConfiguration();
				fbCode
						.setConfigurationKey(ConfigurationConstants.FB_API);
				fbCode.setMerchantId(ctx.getMerchantid());
				fbCode.setDateAdded(new Date());
				fbCode.setLastModified(new Date());

			}
			if (!StringUtils.isBlank(fbkey)) {
				fbCode.setConfigurationValue(fbkey);
			} else {
				fbCode.setConfigurationValue("");
			}

			if (!StringUtils.isBlank(fbsecret)) {
				fbCode.setConfigurationValue1(fbsecret);
			} else {
				fbCode.setConfigurationValue1("");
			}
			mservice.saveOrUpdateMerchantConfiguration(fbCode);
		} else if (StringUtils.isBlank(fbkey)
				|| StringUtils.isBlank(fbsecret)) {
			
			
			if(StringUtils.isBlank(fbkey) && StringUtils.isBlank(fbsecret)) {
				
				if (fbCode != null) {
					mservice.deleteMerchantConfiguration(fbCode);
				}
				
			} else {
				
				if (fbCode != null) {
					if(StringUtils.isBlank(fbkey)) {
						googleCode.setConfigurationValue("");
					}
					if(StringUtils.isBlank(fbsecret)) {
						googleCode.setConfigurationValue("");
					}
					fbCode.setLastModified(new Date());
					mservice.saveOrUpdateMerchantConfiguration(fbCode);
				}
				
			}

		}*/
		
		super.setSuccessMessage();
		
		return SUCCESS;
	}


	public String getGoogleapi() {
		return googleapi;
	}


	public void setGoogleapi(String googleapi) {
		this.googleapi = googleapi;
	}


	public String getAnalytics() {
		return analytics;
	}


	public void setAnalytics(String analytics) {
		this.analytics = analytics;
	}

/*
	public String getFbkey() {
		return fbkey;
	}


	public void setFbkey(String fbkey) {
		this.fbkey = fbkey;
	}


	public String getFbsecret() {
		return fbsecret;
	}


	public void setFbsecret(String fbsecret) {
		this.fbsecret = fbsecret;
	}*/

}



```
