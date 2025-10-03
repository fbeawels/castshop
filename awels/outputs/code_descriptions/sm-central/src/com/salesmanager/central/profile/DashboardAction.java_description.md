# DashboardAction.java

## Review

## 1. Summary  

**Purpose**  
`DashboardAction` is a Struts‑style action (extends `BaseAction`) that is the entry point for the central system’s user dashboard. It is responsible for:

* Authenticating the merchant user after login.  
* Loading the merchant store and user profile.  
* Preparing data for the dashboard view – recent orders, order geography, configuration prompts, and security questions.  
* Providing helper data such as API keys and locale‑specific messages.

**Key components**

| Component | Role |
|-----------|------|
| `MerchantStore` | Holds store configuration (country, id, etc.). |
| `MerchantUserInformation` | Holds user status (NEW, OK, etc.) |
| `OrderService`, `ShippingService`, `CatalogService`, `PaymentService` | Service layer beans used to query orders, shipping, product, and payment data. |
| `MenuFactory`, `CentralFunction` | Build URLs for configuration steps. |
| `ZoneMap` | Lightweight DTO used to aggregate sales per zone/city/country. |
| `SecurityQuestionsModule` | Loads locale‑specific security questions. |
| `Configuration` (via `PropertiesHelper`) | Static config for order summary size. |
| `EncryptionUtil` | Generates API key for the merchant. |

**Design patterns / frameworks**

* **Service Locator** – `ServiceFactory.getService()` is used to obtain various services.  
* **MVC** – This is an Action class that prepares a model for a JSP (or other view).  
* **Dependency Injection (Spring)** – `SpringUtil.getBean()` is used to obtain the security questions module.  
* **Singletons** – `LabelUtil`, `MessageUtil`, `PropertiesUtil`, `RefCache`, `LanguageUtil` are singletons.  
* **Logging** – `log4j` is used for debug and error output.

---

## 2. Detailed Description  

### Execution Flow  

1. **`display()`** – called after successful login.  
   * Retrieves the current `Context` from the session.  
   * Loads `MerchantStore` and `MerchantUserInformation` if not already cached.  
   * Sets the country code, generates an encrypted API key.  
   * Retrieves the central G‑code configuration (if present).  
   * Calls `showOrderSummary()` to load recent orders and build zone statistics.  
   * Depending on the user status (`NEW`, `REVOCATED`, etc.) it calls `checkConfiguration()` to generate configuration prompts.  
   * Sets the page title and returns `"SUCCESS"` for the view.

2. **`showOrderSummary()`** – queries the latest `ORDERSQUANTITY` orders for the merchant.  
   * Determines whether shipping is international.  
   * If national (CA or US) builds a *zone map* per city; otherwise per country.  
   * Populates `orders`, `zoneMaps`, and `zoneMapCount`.

3. **`checkConfiguration()`** – verifies that essential configuration elements exist.  
   * Checks for the existence of a store profile.  
   * Checks that products, payment methods, and shipping modules are configured.  
   * Builds a list of HTML‑linked messages for missing items and sets `configurationMessages`.  
   * If everything is configured, marks the merchant user status as OK.

4. **`checkSales()`** – currently unused in the presented code but mirrors the logic of `showOrderSummary()` to populate `zoneMaps` based on *all* sales rather than a limited subset.

### Assumptions & Constraints  

* The application uses a session attribute `ProfileConstants.context` to hold a `Context` bean.  
* All services are expected to be registered in the `ServiceFactory`.  
* `MerchantStore` and `MerchantUserInformation` are mandatory for a working dashboard.  
* The code assumes a web container that supports J2EE security principals (`super.getPrincipal()`).

### Architecture  

The class follows a *thin controller* approach: it delegates all business logic to services and prepares only the data needed for the view. All view‑related logic (e.g. building URLs, localization) is handled in this action, making the JSPs simpler. The code mixes data access with UI preparation (e.g., building HTML strings in `checkConfiguration()`), which could be refactored into a view‑model or helper class for cleaner separation.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| **`display()`** | Main entry after login. Loads store/user, API key, config, orders, and configuration prompts. | None (uses request/session). | Sets various instance fields, returns `"SUCCESS"` or `"ERROR"`. |
| **`showOrderSummary()`** | Loads recent orders (max `ORDERSQUANTITY`) and builds zone statistics. | None. | Populates `orders`, `zoneMaps`, `zoneMapCount`. |
| **`checkConfiguration()`** | Validates essential configuration (store, products, payments, shipping). Builds HTML messages for missing items. | None. | Sets `configurationMessages`, `configurationMessageTitle`, `statusMessageCode`. |
| **`checkSales()`** | (Unreferenced) Similar to `showOrderSummary()` but for all sales. | None. | Populates `zoneMaps`, `zoneMapCount`. |
| **`getOrders()`, `getCartCounters()`, ...** | Standard getters/setters for the view model. | - | Return or set field values. |
| **`setStore()`, `setMerchantProfile()`** | Store loaded objects for reuse. | MerchantStore / MerchantUserInformation. | Set internal fields. |
| **`getApiKey()`** | Returns encrypted API key. | - | String. |

**Reusable/utility methods**

* `EncryptionUtil.generatekey(...)` & `EncryptionUtil.encrypt(...)` – used to create an API key per store.
* `PropertiesHelper.getConfiguration()` – static config access.
* `LabelUtil.getInstance().getText(...)` – i18n.
* `MessageUtil.addErrorMessage(...)` – error handling.

---

## 4. Dependencies  

| Library / Framework | Role | Standard / Third‑party |
|---------------------|------|------------------------|
| `org.apache.commons.configuration.Configuration` | Configuration properties | Third‑party |
| `org.apache.commons.lang.StringUtils` | String utilities | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.central.*` | Core application classes (BaseAction, Context, etc.) | Internal |
| `com.salesmanager.core.*` | Core services, entities, constants | Internal |
| `com.salesmanager.core.service.*` | Service layer | Internal |
| `com.salesmanager.core.util.*` | Encryption, labeling, messaging, property utilities | Internal |
| `com.salesmanager.core.module.model.application.SecurityQuestionsModule` | Provides security questions | Internal |
| `com.salesmanager.core.service.cache.RefCache` | Cache of reference data (countries) | Internal |
| `org.apache.log4j` | Logging | Third‑party |
| `org.apache.commons` | Utility libraries | Third‑party |
| `SpringUtil` | Spring bean lookup | Internal (Spring) |
| `javax.servlet.http.*` (via BaseAction) | Request/session access | J2EE standard |

No platform‑specific dependencies beyond standard Java EE and the application's internal APIs.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation of concerns**: business logic lives in services; the action only prepares data.  
* **Internationalization support**: uses `LabelUtil` with locale.  
* **Extensible configuration**: `checkConfiguration()` builds dynamic messages and URLs.  
* **Use of static configuration** (`PropertiesHelper`) to control order summary size.  

### Weaknesses & Improvement Ideas  

1. **Hard‑coded HTML strings**  
   * `checkConfiguration()` concatenates HTML links in Java.  
   * **Fix**: move HTML generation to the view layer or use a templating helper.  

2. **Mix of business and presentation logic**  
   * Methods like `showOrderSummary()` set fields that the view uses directly.  
   * **Fix**: create a dedicated DTO (e.g., `DashboardViewModel`) and populate it, then expose that to the view.  

3. **Duplication**  
   * `checkSales()` duplicates logic in `showOrderSummary()`.  
   * **Fix**: extract common zone‑mapping logic into a private helper.  

4. **Unclear API key generation**  
   * The API key is created using `store.getMerchantId()` as both key and IV, which is not cryptographically secure.  
   * **Fix**: generate a random secret per store and store it in a secure place.  

5. **Error handling**  
   * The catch block in `display()` logs and adds a generic error but still returns `"SUCCESS"`.  
   * **Fix**: return an error string or set a failure flag so the view can react appropriately.  

6. **Thread‑safety / Statelessness**  
   * The class uses instance fields to hold request‑specific data. In a typical Struts environment this is acceptable because a new instance is created per request, but documentation should clarify this to avoid surprises in future frameworks.  

7. **Unused fields**  
   * `cartCounters` is defined but never populated.  
   * **Fix**: either implement it or remove it.  

8. **Internationalization**  
   * Some text is fetched with `super.getLocale().getLanguage()` while others use `super.getContext().getLang()`.  
   * **Fix**: use a consistent source of locale.  

9. **Code documentation**  
   * Many methods lack JavaDoc, making it harder for new developers to understand intent.  

10. **Magic numbers**  
    * `Constants.CA_COUNTRY_ID`, `Constants.US_COUNTRY_ID` are hard‑coded; consider constants or config.  

### Potential Enhancements  

* **Unit tests** – Mock services to test `display()` and `showOrderSummary()` independently.  
* **Asynchronous order loading** – For large stores, load orders via AJAX to avoid blocking the page.  
* **Better error messaging** – Use `MessageUtil` consistently for all errors, not just the technical one.  
* **Configuration wizard** – Replace the list of HTML links with a step‑by‑step wizard UI.  

---

### Final Verdict  

`DashboardAction` is functional and follows a standard MVC pattern. It efficiently leverages the service layer and internal utilities. However, it can be improved by separating view logic from business logic, removing duplicated code, enhancing error handling, and modernizing API key generation. With these refinements, the action would be more maintainable, testable, and secure.

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
package com.salesmanager.central.profile;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.util.LanguageHelper;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.central.web.MenuFactory;
import com.salesmanager.central.web.ZoneMap;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantRegistration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.merchant.MerchantUserRole;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.SearchOrderResponse;
import com.salesmanager.core.entity.orders.SearchOrdersCriteria;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.system.CentralFunction;
import com.salesmanager.core.module.model.application.SecurityQuestionsModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.SpringUtil;

/**
 * This is the entry point for the central system, manages logon/logout process
 * and dashboard display
 * 
 * @author Carl Samson
 * 
 */
public class DashboardAction extends BaseAction {

	private Logger log = Logger.getLogger(DashboardAction.class);

	static Configuration conf = PropertiesHelper.getConfiguration();

	private static int ORDERSQUANTITY = 10;

	static {
		ORDERSQUANTITY = conf.getInt("central.orderlistsummary.maxsize");
	}

	private MerchantStore store;
	private MerchantUserInformation merchantProfile;

	// Displays last <ORDERSQUANTITY> on the dashboard page
	private List orders = new ArrayList();

	// Displays Shopping Cart counters on the dashboard page
	private List cartCounters = new ArrayList();
	
	
	private String apiKey = null;

	private int countryCode = -1;

	public int getCountryCode() {
		return countryCode;
	}

	public void setCountryCode(int countryCode) {
		this.countryCode = countryCode;
	}

	private int statusMessageCode = -1;// displayed on the dashboard

	private String configurationMessageTitle;
	private Collection<String> configurationMessages;
	private String message;

	private Collection<ZoneMap> zoneMaps;// geo map
	private int zoneMapCount;// geo map
	private boolean zoneMapRegion = false;
	private String countryIsoCode;
	
	private Map securityQuestions;

	private void checkSales() throws Exception {

		// get sales

		SearchOrdersCriteria criteria = new SearchOrdersCriteria();
		criteria.setMerchantId(super.getContext().getMerchantid());
		criteria.setStartindex(0);
		criteria.setQuantity(-1);

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);
		SearchOrderResponse response = oservice.searchOrders(criteria);
		Collection sales = response.getOrders();

		ShippingService sservice = (ShippingService) ServiceFactory
				.getService(ServiceFactory.ShippingService);
		boolean international = sservice.isShippingInternational(super
				.getContext().getMerchantid());

		boolean nationalMode = false;
		if (sales != null && sales.size() > 0) {

			this.zoneMaps = new ArrayList();

			if (!international
					&& (this.getCountryCode() == Constants.CA_COUNTRY_ID || this
							.getCountryCode() == Constants.US_COUNTRY_ID)) {
				nationalMode = true;
				zoneMapRegion = true;
				Map countries = RefCache.getCountriesMap();
				Country c = (Country) countries.get(this.getCountryCode());
				if (c != null) {
					this.setCountryIsoCode(c.getCountryIsoCode2());
				}
			}

			Map zoneMap = new HashMap();
			Iterator i = sales.iterator();
			while (i.hasNext()) {
				Order order = (Order) i.next();
				String country = order.getBillingCountry();
				String city = order.getBillingCity();
				String finalZone = country;
				if (nationalMode) {
					finalZone = city;
				}
				ZoneMap zm = null;
				if (zoneMap.containsKey(finalZone)) {
					zm = (ZoneMap) zoneMap.get(finalZone);
				} else {
					zm = new ZoneMap();
					zm.setZone(finalZone);
					zoneMap.put(finalZone, zm);
					this.zoneMaps.add(zm);
				}
				int salesCount = zm.getSalesCount() + 1;
				zm.setSalesCount(salesCount);

			}
			this.zoneMapCount = this.zoneMaps.size();
		}

	}

	private void checkConfiguration() throws Exception {

		boolean minimumConfigured = true;

		LabelUtil label = LabelUtil.getInstance();

		List messages = new ArrayList();

		MenuFactory menuFactory = MenuFactory.getInstance();
		Map functions = menuFactory.getFunctionsByFunctionCode();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		// check if MerchantStore exists
		MerchantStore mstore = mservice.getMerchantStore(super.getContext()
				.getMerchantid());

		CentralFunction cfr = (CentralFunction) functions.get("ASTOR01");

		if (mstore == null) {
			messages
					.add("<a href=\""
							+ super.getServletRequest().getContextPath()
							+ cfr.getCentralFunctionUrl()
							+ "\">"
							+ label
									.getText(super.getContext().getLang(),
											"label.dashboard.configurationmessage.storeinformation")
							+ "</a>");
			minimumConfigured = false;
		} else {

			String txt = label.getText(super.getLocale().getLanguage(),
					"label.dashboard.configurationmessage.storeinformation");
			messages.add(txt);
		}

		if (super.getContext().isExistingStore()) {

			// check for any products
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			int count = cservice.getProductCount(super.getContext()
					.getMerchantid());
			cfr = (CentralFunction) functions.get("CPRD01");
			if (count == 0) {
				messages
						.add("<a href=\""
								+ super.getServletRequest().getContextPath()
								+ cfr.getCentralFunctionUrl()
								+ "\">"
								+ label
										.getText(super.getLocale()
												.getLanguage(),
												"label.dashboard.configurationmessage.products")
								+ "</a>");
				minimumConfigured = false;
			} else {
				messages.add(label.getText(super.getContext().getLang(),
						"label.dashboard.configurationmessage.products"));
			}

			// check for any payment method

			PaymentService pservice = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);

			Collection paymentMethods = pservice
					.getConfiguredPaymentMethods(super.getContext()
							.getMerchantid());
			cfr = (CentralFunction) functions.get("CHPAY01");
			if (paymentMethods == null || paymentMethods.size() == 0) {
				messages
						.add("<a href=\""
								+ super.getServletRequest().getContextPath()
								+ cfr.getCentralFunctionUrl()
								+ "\">"
								+ label
										.getText(super.getLocale()
												.getLanguage(),
												"label.dashboard.configurationmessage.payments")
								+ "</a>");
				minimumConfigured = false;
			} else {
				messages.add(label.getText(super.getLocale().getLanguage(),
						"label.dashboard.configurationmessage.payments"));
			}

			// check for any shipping configuration

			ShippingService sservice = (ShippingService) ServiceFactory
					.getService(ServiceFactory.ShippingService);
			Collection shippingMethods = sservice
					.getShippingModulesNamesConfigured(super.getContext()
							.getMerchantid());

			cfr = (CentralFunction) functions.get("CHSH01");
			if (shippingMethods == null || shippingMethods.size() == 0) {
				messages
						.add("<a href=\""
								+ super.getServletRequest().getContextPath()
								+ cfr.getCentralFunctionUrl()
								+ "\">"
								+ label
										.getText(super.getLocale()
												.getLanguage(),
												"label.dashboard.configurationmessage.shipping")
								+ "</a>");
				minimumConfigured = false;
			} else {
				messages.add(label.getText(super.getLocale().getLanguage(),
						"label.dashboard.configurationmessage.shipping"));
			}

		}

		if (messages != null && messages.size() > 0) {

			if (!minimumConfigured) {
				configurationMessages = new ArrayList();
				this.setConfigurationMessageTitle(getText(
						"label.dashboard.message.0",
						new String[] { PropertiesUtil.getConfiguration()
								.getString("core.systemname") }));
				configurationMessages.addAll(messages);
				this.setStatusMessageCode(0);
			}

			if (minimumConfigured) {
				MerchantUserInformation merchantUserInfo = mservice
						.getMerchantUserInformation(super.getPrincipal()
								.getRemoteUser());
				merchantUserInfo.setStatus(Constants.CENTRAL_STATUS_OK);
				mservice.saveOrUpdateMerchantUserInformation(merchantUserInfo);
			}

		}

	}

	public void showOrderSummary() throws Exception {

		SearchOrdersCriteria criteria = new SearchOrdersCriteria();
		criteria.setMerchantId(super.getContext().getMerchantid());
		criteria.setStartindex(0);
		criteria.setQuantity(ORDERSQUANTITY);

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);
		SearchOrderResponse response = oservice.searchOrders(criteria);
		Collection orderCollection = response.getOrders();

		ShippingService sservice = (ShippingService) ServiceFactory
				.getService(ServiceFactory.ShippingService);
		boolean international = sservice.isShippingInternational(super
				.getContext().getMerchantid());

		boolean nationalMode = false;
		if (orderCollection != null && orderCollection.size() > 0) {

			Map zoneMap = new HashMap();
			this.zoneMaps = new ArrayList();

			if (!international
					&& (this.getCountryCode() == Constants.CA_COUNTRY_ID || this
							.getCountryCode() == Constants.US_COUNTRY_ID)) {
				nationalMode = true;
				zoneMapRegion = true;
				Map countries = RefCache.getCountriesMap();
				Country c = (Country) countries.get(this.getCountryCode());
				if (c != null) {
					this.setCountryIsoCode(c.getCountryIsoCode2());
				}
			}

			this.orders = new ArrayList();
			Iterator i = orderCollection.iterator();
			int count = 1;
			while (i.hasNext()) {
				Order order = (Order) i.next();

				if (count < this.ORDERSQUANTITY) {
					this.orders.add(order);
				}

				String country = order.getBillingCountry();
				String city = order.getBillingCity();
				String finalZone = country;
				if (nationalMode) {
					finalZone = city;
				}
				ZoneMap zm = null;
				if (zoneMap.containsKey(finalZone)) {
					zm = (ZoneMap) zoneMap.get(finalZone);
				} else {
					zm = new ZoneMap();
					zm.setZone(finalZone);
					zoneMap.put(finalZone, zm);
					this.zoneMaps.add(zm);
				}
				int salesCount = zm.getSalesCount() + 1;
				zm.setSalesCount(salesCount);

			}
			this.zoneMapCount = this.zoneMaps.size();

		}

	}

	/**
	 * After logon is executed, this takes care of setting dashboard's objects
	 * 
	 * @return
	 * @throws Exception
	 */
	public String display() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		
		
		SecurityQuestionsModule module = (SecurityQuestionsModule)SpringUtil.getBean("securityQuestions");
		Map questions = module.getSecurityQuestions(super.getLocale());
		this.setSecurityQuestions(questions);

		try {

			if (this.getStore() == null) {

				if (ctx == null) {
					return "ERROR";
				}

				MerchantService service = (MerchantService) ServiceFactory
						.getService(ServiceFactory.MerchantService);
				store = service.getMerchantStore(ctx.getMerchantid());
				String userName = super.getPrincipal().getRemoteUser();
				merchantProfile = service.getMerchantUserInformation(userName);

			}

			int status = merchantProfile.getStatus();

			if (store != null) {
				this.setCountryCode(store.getCountry());
				
				String key = EncryptionUtil.generatekey(String.valueOf(store.getMerchantId()));
				apiKey = EncryptionUtil.encrypt(key, String.valueOf(store.getMerchantId()));
				
			} else {
				this.setCountryCode(ctx.getCountryid());
			}

			/** Get some f the configurations **/
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationRequest req = new ConfigurationRequest(ctx
					.getMerchantid(), ConfigurationConstants.G_API);
			ConfigurationResponse resp = mservice.getConfiguration(req);

			MerchantConfiguration gCode = resp
					.getMerchantConfiguration(ConfigurationConstants.G_API);
			if (gCode != null
					&& !StringUtils.isBlank(gCode.getConfigurationValue1())) {
				ctx.setGcode(gCode.getConfigurationValue1());
			}

			this.showOrderSummary();

			switch (status) {

			case Constants.CENTRAL_STATUS_NEW:// first access, store profile not
												// yet created
				this.checkConfiguration();
				break;

			case Constants.CENTRAL_STATUS_REVOCATED:// revocated not implemented
				this.setStatusMessageCode(99);
				break;

			}
			
			super.setPageTitle("label.dashboard.title");
			


		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));

		}

		return SUCCESS;
	}





	public List getOrders() {
		return orders;
	}

	public List getCartCounters() {
		return cartCounters;
	}

	public String getCountryIsoCode() {
		return countryIsoCode;
	}

	public void setCountryIsoCode(String countryIsoCode) {
		this.countryIsoCode = countryIsoCode;
	}

	public boolean isZoneMapRegion() {
		return zoneMapRegion;
	}

	public void setZoneMapRegion(boolean zoneMapRegion) {
		this.zoneMapRegion = zoneMapRegion;
	}

	public int getZoneMapCount() {
		return zoneMapCount;
	}

	public void setZoneMapCount(int zoneMapCount) {
		this.zoneMapCount = zoneMapCount;
	}

	public Collection<ZoneMap> getZoneMaps() {
		return zoneMaps;
	}

	public void setZoneMaps(Collection<ZoneMap> zoneMaps) {
		this.zoneMaps = zoneMaps;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}

	public String getConfigurationMessageTitle() {
		return configurationMessageTitle;
	}

	public void setConfigurationMessageTitle(String configurationMessageTitle) {
		this.configurationMessageTitle = configurationMessageTitle;
	}

	public Collection<String> getConfigurationMessages() {
		return configurationMessages;
	}

	public void setConfigurationMessages(
			Collection<String> configurationMessages) {
		this.configurationMessages = configurationMessages;
	}

	public int getStatusMessageCode() {
		return statusMessageCode;
	}

	public void setStatusMessageCode(int statusMessageCode) {
		this.statusMessageCode = statusMessageCode;
	}

	public MerchantStore getStore() {
		return store;
	}

	public void setStore(MerchantStore store) {
		this.store = store;
	}



	public String getApiKey() {
		return apiKey;
	}

	public void setApiKey(String apiKey) {
		this.apiKey = apiKey;
	}

	public Map getSecurityQuestions() {
		return securityQuestions;
	}

	public void setSecurityQuestions(Map securityQuestions) {
		this.securityQuestions = securityQuestions;
	}

	public MerchantUserInformation getMerchantProfile() {
		return merchantProfile;
	}

	public void setMerchantProfile(MerchantUserInformation merchantProfile) {
		this.merchantProfile = merchantProfile;
	}

}



```
