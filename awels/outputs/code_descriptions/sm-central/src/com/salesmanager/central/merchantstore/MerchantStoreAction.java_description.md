# MerchantStoreAction.java

## Review

## 1. Summary  

`MerchantStoreAction` is a Struts 2 action that handles CRUD operations for merchant‑store data.  
It relies on a `MerchantService` (obtained through a `ServiceFactory`) to perform business‑logic operations such as creating, updating, fetching, or deleting a store.  

Key responsibilities  

| Component | Responsibility |
|-----------|----------------|
| `MerchantStoreAction` | UI layer – receives HTTP requests, validates input, delegates to the service layer, and populates the request for the view. |
| `MerchantService` | Business layer – encapsulates persistence and validation of merchant data. |
| `ServiceFactory` | Factory/DI mechanism for obtaining service instances. |
| `CountrySelectBaseAction` | Common helper for country‑related selection logic and security checks (`isExistingStore`). |
| `Constants` | Holds static configuration values such as `MERCHANT_REG_DEF_CODES`. |

The class uses a mix of Struts 2 conventions (e.g., `addActionError`, `getText`) and a custom context (`Context`) that contains the current merchant/session data.  The code follows a classic three‑tier architecture: UI → Service → DAO.  No complex patterns are used beyond the ServiceFactory; the design is straightforward and easy to understand for developers familiar with Struts 2.

---

## 2. Detailed Description  

### Flow of Execution  

1. **Initialization** – Each action method begins by setting the page title (`setPageTitle`) and performing an `isExistingStore` security check.  
2. **Validation** – `saveMerchantStore()` runs `isValidMerchantInfo()` which uses Apache Commons `StringUtils` to ensure required fields are present.  
3. **Service Interaction** – Depending on the operation:
   * `saveMerchantStore()` obtains the current user’s `MerchantUserInformation`, loads the original `MerchantStore`, and calls `createNewOrSaveMerchant(...)`.
   * `fetchMerchantStore()` loads a specific merchant’s registration, user information, and store details.
   * `viewMerchantStores()` retrieves a list of all store headers.
   * `deleteMerchant()` calls `deleteMerchant(merchantId)`.
4. **Error Handling** – `ServiceException` and generic `Exception` are caught; errors are logged with `log.error(e)` and the user is shown a technical message.  
5. **Success Feedback** – If no action errors are present, `setSuccessMessage()` is called, and in the case of `saveMerchantStore()` request attributes `savedMerchantId` and `savedmerchantUserId` are populated.

### Assumptions & Constraints  

* The user must belong to an existing store (`isExistingStore`); otherwise, the action returns `"unauthorized"`.  
* `merchantId` is passed as a request parameter and must be a non‑zero integer for fetch/delete operations.  
* The `MerchantService` API is assumed to be thread‑safe; the action itself holds no mutable static state.  
* Internationalisation is handled via `getText()`; error messages are defined in property files.  
* The application uses `log4j` for logging; no SLF4J façade is present.

### Architecture  

* **Presentation Layer** – Struts 2 action with JSON annotations (`org.apache.struts2.json.annotations.JSON` used only as an import; not applied).  
* **Business Layer** – Encapsulated in `MerchantService`.  
* **Persistence** – Likely handled by DAO classes inside `MerchantService` (not shown).  
* **Utilities** – `StringUtils`, `ServiceFactory`, and `Constants`.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `saveMerchantStore()` | Persists or updates merchant store data. | None (uses form fields). | `"SUCCESS"`, `"ERROR"`, or `"INPUT"` | Sets success/error messages, request attributes. |
| `fetchMerchantStore()` | Loads merchant data for editing. | None (uses `merchantId`). | `"SUCCESS"` or `"ERROR"` | Sets request attributes, populates form objects. |
| `viewMerchantStores()` | Retrieves list of all store headers. | None | `"SUCCESS"` | Sets `merchantStoreHeaderList`. |
| `deleteMerchant()` | Deletes a merchant record. | None (uses `merchantId`). | `"SUCCESS"` | Sets success/error messages. |
| `isValidMerchantInfo()` | Validates mandatory fields of `merchantUserInfo`. | None | `boolean` | Calls `addActionError` for each missing field. |
| Getters/Setters | Standard JavaBean accessors for action properties (merchant info, registration, headers, etc.). | See field names. | Corresponding field values | No side effects. |

The helper methods (`setPageTitle`, `prepareSelections`, `setSuccessMessage`, `setTechnicalMessage`) are inherited from `CountrySelectBaseAction`.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For string validation. |
| `org.apache.log4j.Logger` | Third‑party | Classic logging. |
| `org.apache.struts2.json.annotations.JSON` | Third‑party | Imported but not used; could be removed. |
| `com.salesmanager.central.CountrySelectBaseAction` | Internal | Provides context, selection helper, and security checks. |
| `com.salesmanager.central.profile.Context` | Internal | Holds current user/merchant session data. |
| `com.salesmanager.central.web.Constants` | Internal | Static config values. |
| `com.salesmanager.core.constants.ErrorConstants` | Internal | Error codes for ServiceException. |
| `com.salesmanager.core.entity.*` | Internal | Domain entities (MerchantRegistration, MerchantStore, etc.). |
| `com.salesmanager.core.service.*` | Internal | Service layer, including `MerchantService`. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for obtaining services. |
| `com.salesmanager.core.service.ServiceException` | Internal | Custom checked exception. |
| `com.salesmanager.core.service.merchant.MerchantException` | Internal | Not directly used; likely thrown by service methods. |

All external dependencies are common, well‑maintained libraries; no platform‑specific code is present.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality & Readability  

* **Duplicated code** – `prepareSelections(...)` is called in multiple places; a helper method could centralize this.  
* **Null handling** – In `fetchMerchantStore()` the block that creates a new `MerchantUserInformation` sets the country code twice (`getZoneid()` and `getCountryid()`); only one should be used.  
* **Logging** – Use parameterised logging (`log.error("Can't delete merchant {}", getMerchantId(), e)`) to avoid unnecessary string concatenation.  
* **Error handling** – Catching `Exception` is too broad; narrow it to `ServiceException` or specific runtime exceptions.  
* **Return values** – The methods return `"SUCCESS"` even after an error is logged. It would be clearer to return `"ERROR"` or a custom result.  
* **Code comments** – Remove commented‑out blocks or add a comment explaining why they are disabled.

### 5.2 Security  

* **Authorization** – Relying solely on `isExistingStore()` may not be sufficient for all endpoints; consider role checks.  
* **Input sanitisation** – Besides checking for emptiness, validate email format, phone number, and postal code patterns.  
* **CSRF** – Struts 2 automatically handles CSRF tokens; ensure the JSP pages include the token.

### 5.3 Performance  

* **Service caching** – If `MerchantService` is stateless, it can be cached (e.g., using a dependency‑injection framework).  
* **Lazy loading** – Avoid loading full `MerchantStore` objects when only headers are needed (`viewMerchantStores()`).

### 5.4 Testing  

* Unit tests should mock `MerchantService` and `ServiceFactory`.  
* Integration tests should exercise each action method with realistic HTTP requests.  
* Validation logic (`isValidMerchantInfo()`) can be unit‑tested separately.

### 5.5 Future Enhancements  

1. **DTOs & JSON** – Introduce Data Transfer Objects and use Struts 2 JSON plugin for AJAX calls.  
2. **Asynchronous operations** – Use asynchronous Struts 2 actions for long‑running updates.  
3. **Refactor to Spring** – Replace `ServiceFactory` with Spring dependency injection for better testability.  
4. **Custom annotations** – Create a `@MerchantStoreAction` annotation that bundles common security and i18n checks.  
5. **Audit trail** – Log every change to merchant data (who, when, what).

---

**Overall Assessment**  
The action is functional and follows a clear, conventional pattern for a Struts 2 application.  The biggest improvements would be to tighten error handling, reduce duplicated code, and adopt modern best practices (e.g., dependency injection, parameterised logging).  With those adjustments, the code will be more maintainable, testable, and secure.

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
package com.salesmanager.central.merchantstore;

import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.json.annotations.JSON;

import com.salesmanager.central.CountrySelectBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.web.Constants;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.merchant.MerchantRegistration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantStoreHeader;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantException;
import com.salesmanager.core.service.merchant.MerchantService;

public class MerchantStoreAction extends CountrySelectBaseAction {

	private static final long serialVersionUID = 1L;
	private Logger log = Logger.getLogger(MerchantStoreAction.class);
	private MerchantUserInformation merchantUserInfo;
	private MerchantRegistration merchantRegistration;
	private MerchantStore merchantStore;
	private int merchantId;
	List<MerchantStoreHeader> merchantStoreHeaderList = null;
	


	private List<Integer> merchantRegistrationDefCodes = Constants.MERCHANT_REG_DEF_CODES;

	

	
	public String saveMerchantStore() {
		
		super.setPageTitle("label.menu.group.store");
		
		try {
		
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		MerchantUserInformation mu = mservice.getMerchantUserInformation(super.getPrincipal().getRemoteUser());
		
		prepareSelections(mu.getUsercountrycode());

		if (!super.getContext().isExistingStore()) {
			return "unauthorized";
		}
		if (!isValidMerchantInfo()) {
			return INPUT;
		}

		
		
		
		//check if email already exist
			//MerchantUserInformation user = mservice.getMerchantUserInformationByAdminEmail(merchantUserInfo.getAdminEmail());
			//if(user!=null) {
			//	super.setErrorMessage("messages.emailalreadyexist");
			//	return INPUT;
			//}

			//super.prepareSelections(merchantUserInfo.getUsercountrycode());


		

			// get original store
			Context ctx = super.getContext();
			MerchantStore originalStore = mservice.getMerchantStore(ctx
					.getMerchantid());

			mservice.createNewOrSaveMerchant(originalStore, merchantUserInfo,
					merchantRegistration);
		} catch (ServiceException e) {
			if (e.getReason() == ErrorConstants.EMAIL_ALREADY_EXISTS) {
				addActionError(getText("errors.merchant.email.already.exists"));
			} else {
				super.setTechnicalMessage();
				log.error(e);
				return ERROR;
			}
		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			return ERROR;
		}
		if (getActionErrors().size() == 0) {
			super.setSuccessMessage();
			getServletRequest().setAttribute("savedMerchantId",String.valueOf(merchantUserInfo.getMerchantId()));
			getServletRequest().setAttribute("savedmerchantUserId",
					merchantUserInfo.getMerchantUserId());
		}
		return SUCCESS;
	}

	public String fetchMerchantStore() {
		
		super.setPageTitle("label.menu.group.store");
		
		if (!super.getContext().isExistingStore()) {
			return "unauthorized";
		}

		if (getMerchantId() != 0) {
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			try {
				merchantRegistration = mservice
						.getMerchantRegistration(getMerchantId());
				
				
				String user = super.getPrincipal().getRemoteUser();
				merchantUserInfo = mservice.getMerchantUserInformation(user);
				
				//merchantUserInfo = mservice
				//		.getMerchantUserInfo(getMerchantId());
				merchantStore = mservice.getMerchantStore(getMerchantId());

				getServletRequest().setAttribute("savedMerchantId",
						String.valueOf(merchantUserInfo.getMerchantId()));
				getServletRequest().setAttribute("savedmerchantUserId",
						merchantUserInfo.getMerchantUserId().toString());
			} catch (Exception e) {
				log.error(e);
				super.setTechnicalMessage();
				return ERROR;
			}

		} else {
			this.merchantUserInfo = new MerchantUserInformation();
			this.merchantUserInfo.setUsercountrycode(super.getContext()
					.getZoneid());
			this.merchantUserInfo.setUsercountrycode(super.getContext()
					.getCountryid());

		}

		super.prepareSelections(merchantUserInfo.getUsercountrycode());

		return SUCCESS;
	}

	public String viewMerchantStores() {
		
		super.setPageTitle("label.menu.group.store");
		
		if (!super.getContext().isExistingStore()) {
			return "unauthorized";
		}
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		merchantStoreHeaderList = mservice.getAllMerchantStores();
		return SUCCESS;
	}

	public String deleteMerchant() {
		
		super.setPageTitle("label.menu.group.store");
		
		if (!super.getContext().isExistingStore()) {
			return "unauthorized";
		}

		try {
			if (getMerchantId() != 0) {
				MerchantService mservice = (MerchantService) ServiceFactory
						.getService(ServiceFactory.MerchantService);
				mservice.deleteMerchant(getMerchantId());
				// addActionMessage(getText("message.merchant.delete.success"));
				super.setSuccessMessage();
			} else {
				addActionError(getText("errors.invalid.merchant.id"));
			}

		} catch (Exception e) {
			log.error("Can't delete merchant " + getMerchantId(), e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	protected boolean isValidMerchantInfo() {
		if (StringUtils.isBlank(merchantUserInfo.getAdminName())) {
			addActionError(getText("messages.required.merchantname"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getAdminEmail())) {
			addActionError(getText("messages.required.adminEmail"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserfname())) {
			addActionError(getText("messages.required.merchantfirstname"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserlname())) {
			addActionError(getText("messages.required.merchantlastname"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserphone())) {
			addActionError(getText("messages.required.userphone"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUseraddress())) {
			addActionError(getText("messages.required.merchantaddress"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUsercity())) {
			addActionError(getText("messages.required.merchantaddress"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserpostalcode())) {
			addActionError(getText("messages.required.userpostalcode"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserstate())) {
			addActionError(getText("messages.required.userstate"));
		}
		if (merchantUserInfo.getUsercountrycode() == 0) {
			addActionError(getText("messages.required.usercountrycode"));
		}
		if (StringUtils.isBlank(merchantUserInfo.getUserlang())) {
			addActionError(getText("messages.required.language"));
		}
		return (getActionErrors().size() == 0);
	}

	public MerchantUserInformation getMerchantUserInfo() {
		return merchantUserInfo;
	}

	public void setMerchantUserInfo(MerchantUserInformation merchantUserInfo) {
		this.merchantUserInfo = merchantUserInfo;
	}

	public MerchantRegistration getMerchantRegistration() {
		return merchantRegistration;
	}

	public void setMerchantRegistration(
			MerchantRegistration merchantRegistration) {
		this.merchantRegistration = merchantRegistration;
	}

	public List<Integer> getMerchantRegistrationDefCodes() {
		return merchantRegistrationDefCodes;
	}

	public void setMerchantRegistrationDefCodes(
			List<Integer> merchantRegistrationDefCodes) {
		this.merchantRegistrationDefCodes = merchantRegistrationDefCodes;
	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

	public List<MerchantStoreHeader> getMerchantStoreHeaderList() {
		return merchantStoreHeaderList;
	}

	public void setMerchantStoreHeaderList(
			List<MerchantStoreHeader> merchantStoreHeaderList) {
		this.merchantStoreHeaderList = merchantStoreHeaderList;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}



}



```
